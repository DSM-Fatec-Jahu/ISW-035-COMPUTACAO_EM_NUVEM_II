# ☁️ Laboratório Prático: Geração de Boletins em PDF com Node.js (Cloud Functions Gen 2)

## 1. Descrição da Funcionalidade e Motivo do Uso

Neste laboratório, desenvolveremos uma Cloud Function responsável por receber as notas de um aluno, calcular a média, gerar um boletim em formato PDF e salvá-lo no Google Cloud Storage (GCS). A função retornará um link temporário e seguro (URL assinada) para o download do documento.

**Por que usar Cloud Functions (Serverless) para isso?**

* **Processamento Pesado:** A geração de arquivos PDF consome muita memória e CPU. Se essa tarefa fosse executada na API principal do sistema acadêmico, poderia causar lentidão para todos os outros usuários conectados.
* **Desacoplamento:** O serviço de armazenamento (Cloud Storage) e o processamento do arquivo ficam isolados. Se o módulo de PDFs falhar, o sistema de matrículas e lançamento de notas continua funcionando perfeitamente.
* **Escalabilidade Automática:** No fechamento do semestre, milhares de alunos acessam o boletim simultaneamente. A infraestrutura serverless escala instâncias automaticamente para dar conta da demanda sazonal e, depois, reduz a zero, economizando custos financeiros.

---

## 2. Recursos Necessários

Para realizar este laboratório, você precisará de:

* Conta no **Google Cloud Platform (GCP)** com um projeto ativo.
* APIs habilitadas no projeto: `Cloud Functions API`, `Cloud Build API`, `Cloud Run API` e `Cloud Storage API` (a habilitação está detalhada no item 4).
* Ambiente de execução: utilizaremos o **Google Cloud Shell** (terminal nativo na nuvem do Google).
* Cliente de API: **Bruno** instalado no seu computador com Windows, para realização de testes visuais.
* Um **bucket** previamente criado no Cloud Storage (a criação está detalhada no item 5).

---

## 3. Definição das Variáveis do Laboratório

Para evitar que você precise editar comandos e procurar identificadores a cada passo, definiremos no início algumas **variáveis de ambiente**. Depois disso, todos os comandos deste roteiro usarão essas variáveis automaticamente, bastando copiar e colar.

Abra o **Cloud Shell** e execute os comandos abaixo, um de cada vez:

```bash
# ID do projeto ativo (identificador textual, ex.: meu-projeto-2026)
export PROJETO_ID=$(gcloud config get-value project)

# Número do projeto (valor numérico usado no e-mail das contas de serviço)
export PROJETO_NUM=$(gcloud projects describe $PROJETO_ID --format="value(projectNumber)")

# Nome do bucket onde os boletins serão armazenados
export BUCKET=boletins-fatec-jahu

# Conta de serviço padrão usada pela função (build e execução)
export SA_COMPUTE=$PROJETO_NUM-compute@developer.gserviceaccount.com

# E-mail da sua conta Google (TROQUE pelo seu e-mail real, apenas uma vez)
export MEU_EMAIL=seu.email@exemplo.com
```

Confira se as variáveis foram preenchidas corretamente:

```bash
echo "Projeto ID:   $PROJETO_ID"
echo "Projeto NUM:  $PROJETO_NUM"
echo "Bucket:       $BUCKET"
echo "Conta SA:     $SA_COMPUTE"
echo "Meu e-mail:   $MEU_EMAIL"
```

> **Atenção a dois pontos:**
>
> * A única linha que você precisa **editar** é a do `MEU_EMAIL`: troque `seu.email@exemplo.com` pelo e-mail da sua conta Google educacional. As demais variáveis são preenchidas automaticamente.
> * Caso queira usar outro nome de bucket (lembre-se de que o nome é único globalmente em todo o GCP), altere apenas o valor de `BUCKET` nesta etapa. Todos os comandos seguintes se ajustam sozinhos.

> **Importante sobre a duração das variáveis:** essas variáveis valem apenas enquanto a sessão do Cloud Shell estiver aberta. Se você fechar o navegador, a sessão expirar ou o Cloud Shell reiniciar, será necessário executar novamente os comandos `export` desta seção antes de continuar de onde parou.

---

## 4. Habilitação das APIs Necessárias

Antes de criar qualquer recurso, é preciso habilitar no projeto as APIs dos serviços que serão utilizados. Sem essa etapa, os comandos de criação do bucket e de deploy da função falham.

No terminal do Cloud Shell, execute o comando abaixo, que habilita todas as APIs necessárias de uma só vez:

```bash
gcloud services enable \
  cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com \
  storage.googleapis.com \
  artifactregistry.googleapis.com \
  logging.googleapis.com \
  iamcredentials.googleapis.com
```

> **Entendendo cada API:**
>
> * `cloudfunctions.googleapis.com` (**Cloud Functions API**): permite criar e gerenciar as funções serverless.
> * `cloudbuild.googleapis.com` (**Cloud Build API**): responsável por empacotar o código e construir a imagem da função durante o deploy.
> * `run.googleapis.com` (**Cloud Run API**): necessária porque as funções de 2ª geração (`--gen2`) rodam sobre a infraestrutura do Cloud Run.
> * `storage.googleapis.com` (**Cloud Storage API**): permite criar buckets e armazenar os arquivos PDF.
> * `artifactregistry.googleapis.com` (**Artifact Registry API**): armazena a imagem de contêiner gerada no build da função de 2ª geração.
> * `logging.googleapis.com` (**Cloud Logging API**): registra os logs de execução da função, úteis para depurar erros.
> * `iamcredentials.googleapis.com` (**IAM Service Account Credentials API**): necessária para que a função gere as URLs assinadas (Signed URLs) de download dos boletins.

> **Observação:** a habilitação das APIs pode levar alguns instantes para se propagar. Ao executar o primeiro deploy, o `gcloud` ainda pode perguntar se deseja habilitar alguma API pendente; basta confirmar com `Y`. Habilitar tudo explicitamente desde o início, como neste passo, evita falhas no meio do laboratório.

---

## 5. Criação do Bucket no Cloud Storage

Antes de programar a função, precisamos do "depósito" onde os PDFs serão armazenados. Você pode criar o bucket de duas formas: pelo painel gráfico (Opção A) ou diretamente pelo terminal (Opção B). Escolha a que preferir.

> **Sobre o acesso aos arquivos:** este laboratório **não torna o bucket público**. Em contas vinculadas a uma organização (como contas educacionais), uma política de segurança chamada **Domain Restricted Sharing** costuma bloquear a concessão de acesso ao principal `allUsers`, resultando no erro `HTTPError 412: One or more users named in the policy do not belong to a permitted customer`. Para funcionar em qualquer cenário (conta pessoal ou organizacional) e ainda seguir uma prática mais segura, o bucket permanece **privado** e a função gera **URLs assinadas (Signed URLs)**, links que carregam a própria chave de acesso e permitem o download sem login Google. Esse mecanismo está detalhado na Seção 6.

### Opção A: Criação via Painel Gráfico (Console)

1. No painel do GCP, acesse o menu de navegação e procure por **Cloud Storage** e depois **Buckets**.
2. Clique em **Criar (Create)**.
3. Defina um nome único globalmente para o bucket. Use o mesmo nome definido na variável `BUCKET` da Seção 3 (`boletins-fatec-jahu`, ou o nome alternativo que você escolheu).
4. Em **Local do tipo (Location type)**, escolha **Region** e selecione `us-central1`. Essa região costuma ter o menor custo e disponibilidade de recursos gratuitos (free tier).
5. Mantenha as demais opções padrão, **sem alterar as configurações de acesso público**, e finalize a criação.
6. Ainda no bucket, acesse a aba **Ciclo de vida (Lifecycle)**, clique em **Adicionar uma regra (Add a rule)**, escolha a ação **Excluir objeto (Delete object)** e defina a condição **Idade (Age)** com o valor `7` dias. Salve a regra.

### Opção B: Criação via Terminal (Cloud Shell)

No terminal do Cloud Shell, execute os comandos abaixo. Eles usam a variável `BUCKET` definida na Seção 3, portanto não há nada para editar.

1. Crie o bucket na região `us-central1`:

   ```bash
   gcloud storage buckets create gs://$BUCKET --location=us-central1
   ```

2. Crie um arquivo de configuração do ciclo de vida e aplique-o ao bucket. O comando abaixo gera um arquivo `lifecycle.json` definindo a exclusão automática de objetos com mais de 7 dias:

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
   gcloud storage buckets update gs://$BUCKET --lifecycle-file=lifecycle.json
   ```

> **Entendendo os comandos:**
>
> * `gcloud storage buckets create`: cria o bucket. O prefixo `gs://` identifica o protocolo do Cloud Storage e `--location` define a região física.
> * `gcloud storage buckets update --lifecycle-file`: aplica regras de ciclo de vida ao bucket. A regra definida exclui automaticamente qualquer objeto que atinja a idade (`age`) de 7 dias, evitando o acúmulo de boletins antigos e o consumo desnecessário de armazenamento.

> **Por que não há comando para liberar acesso público?** Em versões anteriores deste roteiro, o bucket era tornado público com `add-iam-policy-binding` concedendo o papel `roles/storage.objectViewer` ao principal `allUsers`. Esse comando falha em contas educacionais por causa da política **Domain Restricted Sharing**, retornando o erro `HTTPError 412`. Por isso o bucket permanece privado, e o acesso aos PDFs é feito por URLs assinadas geradas pela função (ver Seção 6).

> **Como o ciclo de vida interage com a sobrescrita:** cada novo boletim gerado para o mesmo aluno e matéria substitui o arquivo anterior e reinicia a contagem de idade do objeto (a "idade" passa a contar a partir da última gravação). Na prática, isso significa que a regra de 7 dias raramente apagará um boletim atualizado com frequência, e atuará principalmente como faxina automática de boletins de alunos cujas notas não foram lançadas novamente naquele período.

---

## 6. Criação do Projeto e Código no Cloud Shell

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
// O nome do bucket vem da variável de ambiente BUCKET, informada no deploy.
// Caso a variável não exista, usa 'boletins-fatec-jahu' como valor padrão.
const bucketName = process.env.BUCKET || 'boletins-fatec-jahu';

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
    writeStream.on('finish', async () => {
        try {
            // Gera uma URL assinada (Signed URL) com validade temporária.
            // O bucket permanece PRIVADO; este link concede acesso por tempo limitado.
            // 7 dias é o prazo máximo permitido para URLs assinadas v4.
            const seteDiasEmMs = 7 * 24 * 60 * 60 * 1000;
            const [signedUrl] = await file.getSignedUrl({
                version: 'v4',
                action: 'read',
                expires: Date.now() + seteDiasEmMs // Válido por 7 dias
            });

            res.status(200).json({
                mensagem: 'Boletim gerado e salvo com sucesso!',
                link_download: signedUrl,
                validade: '7 dias'
            });
        } catch (err) {
            console.error('Erro ao gerar URL assinada:', err);
            res.status(500).json({ erro: 'Arquivo salvo, mas falhou ao gerar o link de download.' });
        }
    });

    writeStream.on('error', (err) => {
        console.error('Erro no Storage:', err);
        res.status(500).json({ erro: 'Falha ao salvar o arquivo no Storage.' });
    });
});
```

> **Nota sobre o nome do bucket no código:** repare que o código não tem o nome do bucket escrito diretamente. Ele lê `process.env.BUCKET`, ou seja, a variável de ambiente que será enviada à função no momento do deploy (parâmetro `--set-env-vars`, ver Seção 7). Assim, se você usar outro nome de bucket, basta ajustar a variável `BUCKET` na Seção 3, sem nunca precisar editar o `index.js`. O valor `'boletins-fatec-jahu'` após o operador `||` é apenas um padrão de segurança, usado caso a variável não seja informada.

> **Nota sobre a URL assinada (Signed URL):** o método `getSignedUrl` gera um link criptograficamente assinado que funciona como uma "chave de acesso" embutida na própria URL. Quem tiver o link consegue baixar o arquivo diretamente pelo navegador, sem precisar fazer login em uma conta Google. Aqui o link vale por 7 dias (`expires`), que é o prazo máximo permitido para URLs assinadas v4. Após esse prazo, o link para de funcionar e é necessário gerar o boletim novamente. Essa abordagem é mais segura que um bucket público e funciona em qualquer tipo de conta, inclusive em contas educacionais com a política Domain Restricted Sharing ativa.

> **Permissões necessárias para a conta de serviço:** a conta de serviço que executa a função precisa de dois papéis. O papel **Administrador de objetos do Storage (`roles/storage.objectAdmin`)** permite gravar e ler os PDFs no bucket. O papel **Criador de tokens de conta de serviço (`roles/iam.serviceAccountTokenCreator`)** permite gerar as URLs assinadas. Sem o primeiro, a função falha ao salvar o arquivo (`storage.objects.create access denied`); sem o segundo, salva o arquivo mas falha ao gerar o link. Os comandos para conceder ambos estão descritos na Seção 7, logo após o deploy.

---

## 7. Implementação (Deploy - Gen 2)

### Pré-requisito: permissão da conta de serviço de build

Antes do primeiro deploy, é preciso garantir que a conta de serviço usada para **construir** a função tenha a permissão correta. Desde 2024, o Google deixou de conceder esse papel automaticamente, então em projetos novos (especialmente sob contas educacionais) o passo abaixo é obrigatório. Sem ele, o build falha com a mensagem `missing permission on the build service account`.

Conceda o papel **Builder do Cloud Build** à conta de serviço padrão do Compute Engine. O comando usa as variáveis `PROJETO_ID` e `SA_COMPUTE` definidas na Seção 3, portanto não há nada para editar:

```bash
gcloud projects add-iam-policy-binding $PROJETO_ID \
  --member="serviceAccount:$SA_COMPUTE" \
  --role="roles/cloudbuild.builds.builder"
```

> O papel `roles/cloudbuild.builds.builder` autoriza a conta de serviço a executar o processo de build (empacotar o código e gerar a imagem de contêiner da função). Caso o `gcloud` exiba essa permissão como faltante durante o deploy e ofereça corrigir, você também pode aceitar a correção automática. Executá-la antecipadamente, como neste passo, evita um build falho.

### Deploy da função

Volte para o terminal do Cloud Shell (certifique-se de estar dentro da pasta `lab-boletim-node`). Execute o comando abaixo para implantar seu código na infraestrutura do Google. Ele usa a variável `BUCKET` definida na Seção 3 para informar à função qual bucket utilizar, sem precisar editar o `index.js`:

```bash
gcloud functions deploy boletim-node \
  --gen2 \
  --runtime=nodejs22 \
  --region=us-central1 \
  --source=. \
  --entry-point=gerarBoletim \
  --trigger-http \
  --no-allow-unauthenticated \
  --set-env-vars=BUCKET=$BUCKET
```

### Entendendo os parâmetros escolhidos

* `boletim-node`: é o nome que identificará sua função no painel do GCP.
* `--gen2`: **crucial.** A 2ª geração roda sobre o *Cloud Run*. Isso permite maior tempo de execução (até 60 minutos), instâncias maiores, e o mais importante: **concorrência**. Uma única instância pode processar múltiplas requisições ao mesmo tempo, reduzindo drasticamente o problema de *cold start* (tempo de inicialização a frio).
* `--runtime=nodejs22`: especifica o ambiente de execução. A versão 22 é uma das mais recentes suportadas pelo GCP, trazendo melhorias de desempenho e as últimas correções de segurança.
* `--region=us-central1`: região física dos servidores. A região `us-central1` costuma ter o menor custo e maior disponibilidade de recursos gratuitos (free tier).
* `--source=.`: indica que os arquivos necessários (`index.js` e `package.json`) estão na pasta atual.
* `--entry-point=gerarBoletim`: diz ao Google qual método exportado no código deve ser executado quando a requisição chegar. Tem que ser o nome exato definido no `functions.http(...)`.
* `--trigger-http`: define que a função será acionada por uma requisição web padrão (HTTP).
* `--no-allow-unauthenticated`: a função será **privada**, exigindo autenticação para ser chamada. Em contas educacionais e organizacionais, a política de segurança da instituição **bloqueia** funções públicas (`--allow-unauthenticated`), portanto a função autenticada é a única opção viável nesse cenário. Os testes serão feitos enviando um token de identidade na requisição, conforme detalhado na Seção 8.
* `--set-env-vars=BUCKET=$BUCKET`: envia o nome do bucket para a função como variável de ambiente. É por isso que o `index.js` lê `process.env.BUCKET` e não precisa ter o nome do bucket escrito diretamente no código.

> **Por que a função é privada, mas o download do PDF não exige login?** São duas camadas distintas. A **chamada da função** (enviar as notas e disparar a geração do boletim) exige autenticação, pois a função é privada. Já o **download do PDF** usa a URL assinada, que carrega a própria chave de acesso na URL e funciona sem login Google. Em resumo: gerar o boletim exige token; baixar o boletim gerado, não.

Após cerca de 2 minutos, o terminal exibirá o status `OK` e fornecerá a URL da sua função (algo como `https://boletim-node-...-uc.a.run.app`). Copie essa URL.

> **Dica:** se o Cloud Shell solicitar autorização durante o deploy, confirme com `Y`. O processo de build é gerenciado automaticamente pelo Cloud Build. Caso o `gcloud` ainda aponte a permissão de build como faltante (mesmo após o passo de pré-requisito), aceite a correção oferecida e aguarde a propagação antes de tentar novamente.

### Concedendo as permissões da conta de serviço

Após o deploy, a conta de serviço que executa a função precisa de dois papéis: um para **gravar e ler arquivos no bucket** e outro para **gerar as URLs assinadas**. Por padrão, funções de 2ª geração usam a conta de serviço padrão do Compute Engine, já armazenada na variável `SA_COMPUTE` da Seção 3.

1. Conceda à conta de serviço acesso de escrita e leitura ao bucket:

   ```bash
   gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
     --member="serviceAccount:$SA_COMPUTE" \
     --role="roles/storage.objectAdmin"
   ```

2. Conceda à conta de serviço a permissão de assinar tokens, necessária para gerar as URLs assinadas:

   ```bash
   gcloud iam service-accounts add-iam-policy-binding $SA_COMPUTE \
     --member="serviceAccount:$SA_COMPUTE" \
     --role="roles/iam.serviceAccountTokenCreator"
   ```

> **Entendendo as permissões:**
>
> * `roles/storage.objectAdmin`: concede à conta de serviço o direito de criar, ler e sobrescrever objetos no bucket. Sem ele, a função retorna o erro `storage.objects.create access denied` ao tentar salvar o PDF.
> * `roles/iam.serviceAccountTokenCreator`: autoriza a conta de serviço a assinar tokens em nome de si mesma, requisito para o método `getSignedUrl` funcionar. Sem ele, a função salva o PDF, mas falha ao gerar o link de download.
>
> Observe que ambos os comandos usam o prefixo `serviceAccount:` no membro, e não `allUsers`. Como contas de serviço pertencem ao domínio do projeto, esses comandos funcionam normalmente mesmo em contas educacionais com a política Domain Restricted Sharing ativa.

---

## 8. Testando a Função Implementada

Como a função foi implantada de forma **privada** (`--no-allow-unauthenticated`), toda requisição precisa enviar um **token de identidade** no cabeçalho `Authorization`. Esse token comprova que quem está chamando a função é um usuário autenticado e autorizado.

> **Importante:** o token de identidade tem validade de aproximadamente **1 hora**. Se a aula for longa, pode ser necessário gerar um novo token durante os testes. Isso não afeta o link de download do PDF, que continua válido por 7 dias.

### Concedendo permissão de invocação ao seu usuário

Antes de testar, sua conta precisa do papel **Invocador do Cloud Run (`roles/run.invoker`)** na função. O comando abaixo usa a variável `MEU_EMAIL` definida na Seção 3:

```bash
gcloud functions add-invoker-policy-binding boletim-node \
  --region=us-central1 \
  --member="user:$MEU_EMAIL"
```

> Esse comando autoriza especificamente a sua conta a chamar a função. O membro usa o prefixo `user:`, que pertence ao domínio da instituição, portanto não esbarra na política Domain Restricted Sharing. Se você não definiu corretamente a variável `MEU_EMAIL` na Seção 3, volte e ajuste antes de prosseguir.

### Opção A: Teste Rápido via Terminal (cURL)

No próprio Cloud Shell, execute o comando abaixo. Ele gera o token de identidade automaticamente com `gcloud auth print-identity-token` e o envia no cabeçalho. Substitua a URL pela gerada no seu deploy:

```bash
curl -X POST https://SUA_URL_GERADA_AQUI \
  -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d '{"ra": "12345", "materia_id": "BDR", "t1": 8, "t2": 7, "p1": 6.5, "p2": 8}'
```

Se tudo der certo, você receberá um JSON no terminal contendo o `link_download` do PDF. Esse link pode ser aberto diretamente no navegador, sem necessidade de login.

> **Dica:** se quiser inspecionar o token ou reutilizá-lo, gere-o separadamente e guarde em uma variável:
>
> ```bash
> TOKEN=$(gcloud auth print-identity-token)
> echo $TOKEN
> ```
>
> Depois é só usar `-H "Authorization: Bearer $TOKEN"` nas requisições.

### Opção B: Teste Visual Profissional via Bruno (Windows)

O Bruno é um cliente de API open-source e muito leve, excelente alternativa ao Postman para testes estruturados.

1. Primeiro, gere o token de identidade. No Cloud Shell, execute:

   ```bash
   gcloud auth print-identity-token
   ```

   Copie o texto longo retornado (é o token). Ele será colado no Bruno no passo 7.
2. Abra o Bruno no seu Windows.
3. Clique em **Create Collection** (Criar Coleção) e dê o nome de `Laboratorio Nuvem`.
4. Dentro da coleção, clique em **New Request** (Nova Requisição):
   * **Name:** Gerar Boletim Node
   * **Type:** HTTP
   * **Method:** mude de `GET` para `POST`.
   * **URL:** cole a URL gerada no final do comando de deploy.
5. Abaixo da barra de URL, clique na aba **Auth**.
6. No tipo de autenticação, selecione **Bearer Token**.
7. No campo **Token**, cole o token de identidade copiado no passo 1.
8. Clique na aba **Body**, selecione a opção **JSON** e cole o seguinte payload (dados de teste):

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

9. Clique no botão **Send** (Enviar) no canto superior direito.
10. Observe o painel de **Response** (Resposta) no lado direito da tela. Você verá o status `200 OK` e o JSON com a mensagem de sucesso.
11. Clique no link gerado na resposta (`link_download`) para baixar o PDF diretamente no navegador. Esse link é uma URL assinada, válida por 7 dias, e não exige login em conta Google.

> **Se o teste retornar `401 Unauthorized` ou `403 Forbidden`:** o token pode ter expirado (validade de cerca de 1 hora). Gere um novo token com `gcloud auth print-identity-token` e atualize o campo no Bruno. Confirme também que o papel `roles/run.invoker` foi concedido à sua conta.

---

## 9. Atividade Proposta

Para fixar os conceitos, realize as seguintes tarefas:

1. **Teste de aprovação x reprovação:** envie uma requisição usando o aluno de RA `67890` (Heloana) com notas altas e observe o status final. Explique por que o resultado pode ser `REPROVADO` mesmo com boa média.
2. **Tratamento de erro:** envie uma requisição com um `materia_id` inexistente (ex.: `XYZ`) e verifique a resposta retornada pela função.
3. **Validação de entrada:** envie uma requisição omitindo a nota `p2` e analise o comportamento da função após a inclusão da validação.
4. **Evolução do código:** acrescente um novo aluno e uma nova matéria aos arrays `alunosDB` e `materiasDB`, faça um novo deploy e gere o boletim correspondente.
5. **Análise do link assinado:** gere um boletim e examine a URL retornada no campo `link_download`. Identifique os parâmetros de assinatura presentes na URL (como `X-Goog-Expires` e `X-Goog-Signature`) e explique, com suas palavras, como esse mecanismo permite acesso ao arquivo sem login Google e por que ele é mais seguro que um bucket público. Considere que o link expira em 7 dias.
6. **Reflexão escrita:** descreva, em um parágrafo, uma situação real na qual a arquitetura serverless traria vantagem de custo frente a um servidor sempre ligado.

---

## 10. Encerramento e Limpeza de Recursos

Para evitar cobranças indevidas após o laboratório, lembre-se de remover os recursos criados:

* Exclua a Cloud Function pelo painel ou via comando:

  ```bash
  gcloud functions delete boletim-node --gen2 --region=us-central1
  ```

* Exclua o bucket e todos os seus arquivos pelo painel do Cloud Storage ou via terminal, caso não vá reutilizá-lo:

  ```bash
  gcloud storage rm --recursive gs://$BUCKET
  ```

> Manter recursos ociosos dentro do free tier geralmente não gera custo, mas a limpeza é uma boa prática profissional de governança em nuvem. Caso a sessão do Cloud Shell tenha sido reiniciada, lembre-se de redefinir as variáveis da Seção 3 antes de executar estes comandos.
