# ☁️ Laboratório Prático: Geração de Boletins em PDF com Node.js (Cloud Functions Gen 2)

## 1. Descrição da Funcionalidade e Motivo do Uso

Neste laboratório, desenvolveremos uma Cloud Function responsável por receber as notas de um aluno, calcular a média, gerar um boletim em formato PDF e salvá-lo no Google Cloud Storage (GCS). A função retornará um link público para o download do documento.

**Por que usar Cloud Functions (Serverless) para isso?**

* **Processamento Pesado:** A geração de arquivos PDF consome muita memória e CPU. Se essa tarefa fosse executada na API principal do sistema acadêmico, poderia causar lentidão para todos os outros usuários conectados.
* **Desacoplamento:** O serviço de armazenamento (Cloud Storage) e o processamento do arquivo ficam isolados. Se o módulo de PDFs falhar, o sistema de matrículas e lançamento de notas continua funcionando perfeitamente.
* **Escalabilidade Automática:** No fechamento do semestre, milhares de alunos acessam o boletim simultaneamente. A infraestrutura serverless escala instâncias automaticamente para dar conta da demanda sazonal e, depois, reduz a zero, economizando custos financeiros.

---

## 2. Recursos Necessários

Para realizar este laboratório, você precisará de:

* Conta no **Google Cloud Platform (GCP)** com um projeto ativo.
* APIs habilitadas no projeto: `Cloud Functions API`, `Cloud Build API`, `Cloud Run API` e `Cloud Storage API`.
* Ambiente de execução: utilizaremos o **Google Cloud Shell** (terminal nativo na nuvem do Google).
* Cliente de API: **Bruno** instalado no seu computador com Windows, para realização de testes visuais.
* Um **bucket** previamente criado no Cloud Storage (a criação está detalhada no item 3).

---

## 3. Criação do Bucket no Cloud Storage

Antes de programar a função, precisamos do "depósito" onde os PDFs serão armazenados. Você pode criar o bucket de duas formas: pelo painel gráfico (Opção A) ou diretamente pelo terminal (Opção B). Escolha a que preferir.

### Opção A: Criação via Painel Gráfico (Console)

1. No painel do GCP, acesse o menu de navegação e procure por **Cloud Storage** e depois **Buckets**.
2. Clique em **Criar (Create)**.
3. Defina um nome único globalmente para o bucket. Neste roteiro usaremos `boletins-fatec-jahu` (caso o nome já esteja em uso, escolha outro e anote, pois ele será usado no código).
4. Em **Local do tipo (Location type)**, escolha **Region** e selecione `us-central1`. Essa região costuma ter o menor custo e disponibilidade de recursos gratuitos (free tier).
5. Mantenha as demais opções padrão e finalize a criação.
6. Após criar o bucket, vá até a aba **Permissões (Permissions)**, clique em **Conceder acesso (Grant access)**, adicione o principal `allUsers` e atribua o papel **Leitor de objetos do Storage (Storage Object Viewer)**. Confirme o aviso de que o bucket se tornará público.
7. Ainda no bucket, acesse a aba **Ciclo de vida (Lifecycle)**, clique em **Adicionar uma regra (Add a rule)**, escolha a ação **Excluir objeto (Delete object)** e defina a condição **Idade (Age)** com o valor `7` dias. Salve a regra.

### Opção B: Criação via Terminal (Cloud Shell)

No terminal do Cloud Shell, execute os comandos abaixo. Lembre-se de trocar `boletins-fatec-jahu` caso esse nome já esteja em uso.

1. Crie o bucket na região `us-central1`:

   ```bash
   gcloud storage buckets create gs://boletins-fatec-jahu --location=us-central1
   ```

2. Libere a leitura pública dos arquivos, concedendo o papel de leitor de objetos ao principal `allUsers`:

   ```bash
   gcloud storage buckets add-iam-policy-binding gs://boletins-fatec-jahu \
     --member=allUsers \
     --role=roles/storage.objectViewer
   ```

3. Crie um arquivo de configuração do ciclo de vida e aplique-o ao bucket. O comando abaixo gera um arquivo `lifecycle.json` definindo a exclusão automática de objetos com mais de 7 dias:

   ```bash
   cat > lifecycle.json << 'EOF'
   {
     "rule": [
       {
         "action": { "type": "Delete" },
         "condition": { "age": 7 }
       }
     ]
   }
   EOF
   ```

   Em seguida, aplique a configuração ao bucket:

   ```bash
   gcloud storage buckets update gs://boletins-fatec-jahu --lifecycle-file=lifecycle.json
   ```

> **Entendendo os comandos:**
>
> * `gcloud storage buckets create`: cria o bucket. O prefixo `gs://` identifica o protocolo do Cloud Storage e `--location` define a região física.
> * `gcloud storage buckets add-iam-policy-binding`: adiciona uma regra de permissão (IAM) ao bucket. O membro `allUsers` representa qualquer pessoa na internet, e o papel `roles/storage.objectViewer` concede apenas leitura dos objetos, sem permitir alteração ou exclusão.
> * `gcloud storage buckets update --lifecycle-file`: aplica regras de ciclo de vida ao bucket. A regra definida exclui automaticamente qualquer objeto que atinja a idade (`age`) de 7 dias, evitando o acúmulo de boletins antigos e o consumo desnecessário de armazenamento.

> **Como o ciclo de vida interage com a sobrescrita:** cada novo boletim gerado para o mesmo aluno e matéria substitui o arquivo anterior e reinicia a contagem de idade do objeto (a "idade" passa a contar a partir da última gravação). Na prática, isso significa que a regra de 7 dias raramente apagará um boletim atualizado com frequência, e atuará principalmente como faxina automática de boletins de alunos cujas notas não foram lançadas novamente naquele período.

> **Observação importante sobre o acesso público:** por padrão, buckets novos vêm com o **Acesso uniforme em nível de bucket (Uniform Bucket-Level Access)** ativado e bloqueio de acesso público. Como nosso laboratório retorna um link direto de download, precisamos liberar a leitura pública (passo executado tanto na Opção A quanto na Opção B). Em ambiente real de produção isso não seria recomendado; aqui é aceitável por se tratar de um laboratório controlado.

---

## 4. Criação do Projeto e Código no Cloud Shell

Para não precisarmos instalar nada localmente, desenvolveremos diretamente na nuvem.

1. Acesse o painel do Google Cloud Platform e clique no ícone do **Cloud Shell** (um símbolo de `>_` no canto superior direito).
2. No terminal do Cloud Shell, crie uma pasta para o projeto e entre nela:

   ```bash
   mkdir lab-boletim-node
   cd lab-boletim-node
   ```

3. Inicie um projeto Node.js e instale as dependências necessárias:

   ```bash
   npm init -y
   npm install @google-cloud/functions-framework pdfkit @google-cloud/storage
   ```

4. Clique no botão **"Abrir Editor" (Open Editor)** no menu superior do Cloud Shell. Isso abrirá uma interface visual semelhante ao VS Code diretamente no navegador.
5. No painel esquerdo, localize a pasta `lab-boletim-node`, crie um arquivo chamado `index.js` e cole o código abaixo.

### Código Fonte (`index.js`)

```javascript
// Importação dos módulos necessários
const functions = require('@google-cloud/functions-framework');
const PDFDocument = require('pdfkit');
const { Storage } = require('@google-cloud/storage');

// Inicialização do cliente do Cloud Storage
const storage = new Storage();
// ATENÇÃO: Substitua pelo nome do seu bucket criado previamente no GCP!
const bucketName = 'boletins-fatec-jahu';

// Simulação de um Banco de Dados em memória
const alunosDB = [
    { ra: '12345', nome: 'Américo Silva', frequencia: 85 },
    { ra: '67890', nome: 'Heloana', frequencia: 60 }
];

const materiasDB = [
    { id: 'BDR', nome: 'Banco de Dados Relacionais' },
    { id: 'CN2', nome: 'Computação em Nuvem II' }
];

// Declaração da função HTTP. O nome 'gerarBoletim' será usado no deploy.
functions.http('gerarBoletim', (req, res) => {
    // 1. Extração dos dados enviados no corpo da requisição (JSON)
    const { ra, materia_id, t1, t2, p1, p2 } = req.body;

    // 2. Validação dos dados recebidos
    const notas = [t1, t2, p1, p2];
    const notasValidas = notas.every(n => typeof n === 'number');
    if (!ra || !materia_id || !notasValidas) {
        return res.status(400).json({
            erro: 'Dados incompletos. Informe ra, materia_id e as quatro notas numéricas (t1, t2, p1, p2).'
        });
    }

    // 3. Busca dos dados no nosso "banco de dados"
    const aluno = alunosDB.find(a => a.ra === ra);
    const materia = materiasDB.find(m => m.id === materia_id);

    if (!aluno || !materia) {
        return res.status(404).json({ erro: 'Aluno ou Matéria não encontrados.' });
    }

    // 4. Regras de Negócio: Cálculo de Média e Aprovação
    const mediaFinal = (((t1 + t2) / 2) * 0.4) + (((p1 + p2) / 2) * 0.6);
    const status = (mediaFinal >= 6 && aluno.frequencia >= 75) ? 'APROVADO' : 'REPROVADO';

    // 5. Preparação do arquivo PDF
    const doc = new PDFDocument({ margin: 50 });
    const fileName = `${materia.id}_${aluno.ra}.pdf`; // Ex: BDR_12345.pdf

    // Conexão com o Bucket do Cloud Storage
    const bucket = storage.bucket(bucketName);
    const file = bucket.file(fileName);

    // Cria um fluxo de escrita direto para o bucket na nuvem
    const writeStream = file.createWriteStream({
        resumable: false,
        contentType: 'application/pdf',
        metadata: { cacheControl: 'no-cache' } // Evita que navegadores guardem versão antiga
    });

    // O gerador de PDF agora despeja os dados diretamente no Storage
    doc.pipe(writeStream);

    // 6. Desenhando o conteúdo do PDF
    doc.fontSize(20).text('Fatec Jahu', { align: 'center' }).moveDown();
    doc.fontSize(14).text('Boletim de Desempenho Acadêmico', { align: 'center' }).moveDown();

    doc.fontSize(12).text(`Disciplina: ${materia.nome}`);
    doc.text(`RA: ${aluno.ra}`);
    doc.text(`Aluno: ${aluno.nome}`);
    doc.text(`Frequência: ${aluno.frequencia}%`).moveDown();

    doc.text(`Notas dos Trabalhos: T1 = ${t1} | T2 = ${t2}`);
    doc.text(`Notas das Provas: P1 = ${p1} | P2 = ${p2}`).moveDown();

    doc.fontSize(14).text(`Média Final: ${mediaFinal.toFixed(2)}`);
    doc.text(`Status Final: ${status}`);

    doc.end(); // Finaliza a criação do PDF

    // 7. Tratamento de Sucesso ou Erro ao salvar o arquivo
    writeStream.on('finish', () => {
        // Gera a URL pública padrão do Google Cloud Storage
        const publicUrl = `https://storage.googleapis.com/${bucketName}/${fileName}`;
        res.status(200).json({
            mensagem: 'Boletim gerado e salvo com sucesso!',
            link_download: publicUrl
        });
    });

    writeStream.on('error', (err) => {
        console.error('Erro no Storage:', err);
        res.status(500).json({ erro: 'Falha ao salvar o arquivo no Storage.' });
    });
});
```

> **Nota sobre a URL pública:** o link no formato `https://storage.googleapis.com/bucket/arquivo` só funciona porque liberamos o bucket para leitura pública no item 3. Caso prefira tornar público apenas o objeto gerado (em vez do bucket inteiro), você pode adicionar `predefinedAcl: 'publicRead'` nas opções do `createWriteStream`, desde que o bucket esteja com o Acesso uniforme em nível de bucket desativado.

---

## 5. Implementação (Deploy - Gen 2)

Volte para o terminal do Cloud Shell (certifique-se de estar dentro da pasta `lab-boletim-node`). Execute o comando abaixo para implantar seu código na infraestrutura do Google:

```bash
gcloud functions deploy boletim-node \
  --gen2 \
  --runtime=nodejs22 \
  --region=us-central1 \
  --source=. \
  --entry-point=gerarBoletim \
  --trigger-http \
  --allow-unauthenticated
```

### Entendendo os parâmetros escolhidos

* `boletim-node`: é o nome que identificará sua função no painel do GCP.
* `--gen2`: **crucial.** A 2ª geração roda sobre o *Cloud Run*. Isso permite maior tempo de execução (até 60 minutos), instâncias maiores, e o mais importante: **concorrência**. Uma única instância pode processar múltiplas requisições ao mesmo tempo, reduzindo drasticamente o problema de *cold start* (tempo de inicialização a frio).
* `--runtime=nodejs22`: especifica o ambiente de execução. A versão 22 é uma das mais recentes suportadas pelo GCP, trazendo melhorias de desempenho e as últimas correções de segurança.
* `--region=us-central1`: região física dos servidores. A região `us-central1` costuma ter o menor custo e maior disponibilidade de recursos gratuitos (free tier).
* `--source=.`: indica que os arquivos necessários (`index.js` e `package.json`) estão na pasta atual.
* `--entry-point=gerarBoletim`: diz ao Google qual método exportado no código deve ser executado quando a requisição chegar. Tem que ser o nome exato definido no `functions.http(...)`.
* `--trigger-http`: define que a função será acionada por uma requisição web padrão (HTTP).
* `--allow-unauthenticated`: torna a URL pública. Como é um laboratório focado na funcionalidade, pulamos a camada de tokens do IAM para facilitar os testes via APIs externas.

Após cerca de 2 minutos, o terminal exibirá o status `OK` e fornecerá a URL pública da sua função (algo como `https://boletim-node-...-uc.a.run.app`). Copie essa URL.

> **Dica:** se o Cloud Shell solicitar autorização ou perguntar se deseja habilitar APIs durante o primeiro deploy, confirme com `Y`. O processo de build é gerenciado automaticamente pelo Cloud Build.

---

## 6. Testando a Função Implementada

### Opção A: Teste Rápido via Terminal (cURL)

No próprio Cloud Shell (ou no terminal do seu computador), você pode testar disparando o comando abaixo. Substitua a URL pela gerada no seu deploy:

```bash
curl -X POST https://SUA_URL_GERADA_AQUI \
  -H "Content-Type: application/json" \
  -d '{"ra": "12345", "materia_id": "BDR", "t1": 8, "t2": 7, "p1": 6.5, "p2": 8}'
```

Se tudo der certo, você receberá um JSON no terminal contendo o link de download do PDF.

### Opção B: Teste Visual Profissional via Bruno (Windows)

O Bruno é um cliente de API open-source e muito leve, excelente alternativa ao Postman para testes estruturados.

1. Abra o Bruno no seu Windows.
2. Clique em **Create Collection** (Criar Coleção) e dê o nome de `Laboratorio Nuvem`.
3. Dentro da coleção, clique em **New Request** (Nova Requisição):
   * **Name:** Gerar Boletim Node
   * **Type:** HTTP
   * **Method:** mude de `GET` para `POST`.
   * **URL:** cole a URL gerada no final do comando de deploy.
4. Abaixo da barra de URL, clique na aba **Body**.
5. Selecione a opção **JSON**.
6. Cole o seguinte payload (dados de teste) na caixa de texto:

   ```json
   {
     "ra": "12345",
     "materia_id": "BDR",
     "t1": 8,
     "t2": 7,
     "p1": 6.5,
     "p2": 8
   }
   ```

7. Clique no botão **Send** (Enviar) no canto superior direito.
8. Observe o painel de **Response** (Resposta) no lado direito da tela. Você verá o status `200 OK` e o JSON com a mensagem de sucesso.
9. Clique no link gerado na resposta para fazer o download do PDF criado diretamente no navegador.

---

## 7. Atividade Proposta

Para fixar os conceitos, realize as seguintes tarefas:

1. **Teste de aprovação x reprovação:** envie uma requisição usando o aluno de RA `67890` (Heloana) com notas altas e observe o status final. Explique por que o resultado pode ser `REPROVADO` mesmo com boa média.
2. **Tratamento de erro:** envie uma requisição com um `materia_id` inexistente (ex.: `XYZ`) e verifique a resposta retornada pela função.
3. **Validação de entrada:** envie uma requisição omitindo a nota `p2` e analise o comportamento da função após a inclusão da validação.
4. **Evolução do código:** acrescente um novo aluno e uma nova matéria aos arrays `alunosDB` e `materiasDB`, faça um novo deploy e gere o boletim correspondente.
5. **Reflexão escrita:** descreva, em um parágrafo, uma situação real na qual a arquitetura serverless traria vantagem de custo frente a um servidor sempre ligado.

---

## 8. Encerramento e Limpeza de Recursos

Para evitar cobranças indevidas após o laboratório, lembre-se de remover os recursos criados:

* Exclua a Cloud Function pelo painel ou via comando:

  ```bash
  gcloud functions delete boletim-node --gen2 --region=us-central1
  ```

* Exclua o bucket e todos os seus arquivos pelo painel do Cloud Storage ou via terminal, caso não vá reutilizá-lo:

  ```bash
  gcloud storage rm --recursive gs://boletins-fatec-jahu
  ```

> Manter recursos ociosos dentro do free tier geralmente não gera custo, mas a limpeza é uma boa prática profissional de governança em nuvem.
