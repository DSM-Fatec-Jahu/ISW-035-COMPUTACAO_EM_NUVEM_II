# Aula: Implantação no Azure — App Services + MySQL Flexible Server

**Disciplina:** Computação em Nuvem II (ISW035)  
**Projeto de referência:** TaskFlow — Backend (Next.js + Prisma) e Frontend (React + Vite)

---

## Sumário

1. [Visão geral da arquitetura](#1-visão-geral-da-arquitetura)
2. [Pré-requisitos](#2-pré-requisitos)
3. [Roteiro geral de implantação](#3-roteiro-geral-de-implantação)
4. [Etapa 1 — Criar o Grupo de Recursos](#4-etapa-1--criar-o-grupo-de-recursos)
5. [Etapa 2 — Criar o Banco de Dados MySQL Flexible Server](#5-etapa-2--criar-o-banco-de-dados-mysql-flexible-server)
6. [Etapa 3 — Configurar o Banco de Dados](#6-etapa-3--configurar-o-banco-de-dados)
7. [Etapa 4 — Criar o App Service (Backend Next.js)](#7-etapa-4--criar-o-app-service-backend-nextjs)
8. [Etapa 5 — Configurar Variáveis de Ambiente do Backend](#8-etapa-5--configurar-variáveis-de-ambiente-do-backend)
9. [Etapa 6 — Fazer Deploy do Backend via GitHub Actions](#9-etapa-6--fazer-deploy-do-backend-via-github-actions)
10. [Etapa 7 — Executar Migrações do Prisma](#10-etapa-7--executar-migrações-do-prisma)
11. [Etapa 8 — Criar o App Service (Frontend React)](#11-etapa-8--criar-o-app-service-frontend-react)
12. [Etapa 9 — Configurar Variáveis de Ambiente do Frontend](#12-etapa-9--configurar-variáveis-de-ambiente-do-frontend)
13. [Etapa 10 — Fazer Deploy do Frontend via GitHub Actions](#13-etapa-10--fazer-deploy-do-frontend-via-github-actions)
14. [Etapa 11 — Verificar CORS no Backend](#14-etapa-11--verificar-cors-no-backend)
15. [Referência rápida de preços](#15-referência-rápida-de-preços)
16. [Solução de problemas comuns](#16-solução-de-problemas-comuns)

---

## 1. Visão geral da arquitetura

O projeto TaskFlow é composto por três camadas implantadas dentro de um único **Grupo de Recursos** no Azure. O diagrama abaixo mostra como os serviços se comunicam em produção e quais variáveis de ambiente controlam cada conexão.

```mermaid
graph TD
    U(["👤 Usuário (Navegador)"])

    subgraph rg["☁️ rg-taskflow-prod — Brazil South"]
        direction TB

        subgraph fe_plan["📦 asp-taskflow-frontend (Plano B1 Linux)"]
            FE["🌐 App Service — Frontend\nfrontend-taskflow-prod\nReact + Vite\nNode 22 LTS"]
        end

        subgraph be_plan["📦 asp-taskflow-backend (Plano B1 Linux)"]
            BE["⚙️ App Service — Backend\nbackend-taskflow-prod\nNext.js + Prisma\nNode 22 LTS"]
        end

        subgraph db_plane["🗄️ MySQL Flexible Server"]
            DB[("mysql-taskflow-prod\nStandard_B1ms\nMySQL 8.0\nBanco: taskflow")]
        end
    end

    U -->|"HTTPS :443"| FE
    FE -->|"VITE_API_URL\nHTTPS /api/*"| BE
    BE -->|"DATABASE_URL\nTCP :3306 + SSL"| DB

    style rg fill:#e6f3ff,stroke:#0078d4,stroke-width:2px
    style fe_plan fill:#fff4e6,stroke:#ff8c00,stroke-width:1px
    style be_plan fill:#e6ffe6,stroke:#107c10,stroke-width:1px
    style db_plane fill:#ffe6e6,stroke:#d83b01,stroke-width:1px
    style FE fill:#fff4e6,stroke:#ff8c00
    style BE fill:#e6ffe6,stroke:#107c10
    style DB fill:#ffe6e6,stroke:#d83b01
```

> **Por que App Service e não Static Web Apps para o frontend?**  
> O Static Web Apps gratuito é ótimo para SPAs puras, mas o App Service oferece um ambiente unificado e mais fácil de configurar variáveis de ambiente em tempo de execução. Para este projeto didático, utilizaremos App Service para ambos os serviços.

---

## 2. Pré-requisitos

Antes de começar, garanta que você tem:

- Uma conta ativa no [portal.azure.com](https://portal.azure.com) (conta gratuita com crédito de US$ 200 é suficiente)
- O código do projeto em um repositório no **GitHub** (público ou privado)
- O `package.json` do backend contendo os scripts:
  ```json
  "scripts": {
    "build": "next build",
    "start": "next start -p $PORT"
  }
  ```
- O `package.json` do frontend contendo os scripts:
  ```json
  "scripts": {
    "build": "tsc -b && vite build",
    "preview": "vite preview --port $PORT --host"
  }
  ```

> **Dica:** O Azure App Service (Linux) injeta automaticamente a variável de ambiente `PORT`. É obrigatório que sua aplicação escute nessa porta, caso contrário o health check falhará e o deploy será considerado mal sucedido.

---

## 3. Roteiro geral de implantação

O diagrama abaixo mostra a **ordem correta** de criação de todos os recursos. Siga exatamente esta sequência para evitar dependências quebradas.

```mermaid
flowchart TD
    A([🚀 Início]) --> B

    B["1️⃣ Criar Grupo de Recursos\nrg-taskflow-prod"]
    B --> C

    C["2️⃣ Criar MySQL Flexible Server\nmysql-taskflow-prod"]
    C --> D

    D["3️⃣ Criar banco de dados\ntaskflow + charset utf8mb4"]
    D --> E

    E["4️⃣ Criar App Service — Backend\nbackend-taskflow-prod\nNode 22 / Linux / B1"]
    E --> F

    F["5️⃣ Configurar variáveis de ambiente\ndo Backend no portal Azure"]
    F --> G

    G["6️⃣ Ajustar workflow GitHub Actions\npara subpasta backend/"]
    G --> H

    H["7️⃣ git push → Deploy do Backend\nvia GitHub Actions"]
    H --> I

    I["8️⃣ Executar migrações Prisma\nnpx prisma migrate deploy"]
    I --> J

    J{Backend OK?\nGET /api/tasks → [ ]}
    J -- Não --> K["🔍 Verificar Log Stream\nno portal Azure"]
    K --> F
    J -- Sim --> L

    L["9️⃣ Criar App Service — Frontend\nfrontend-taskflow-prod\nNode 22 / Linux / B1"]
    L --> M

    M["🔟 Configurar variáveis de ambiente\ndo Frontend no portal Azure"]
    M --> N

    N["1️⃣1️⃣ Ajustar workflow GitHub Actions\npara subpasta frontend/"]
    N --> O

    O["1️⃣2️⃣ git push → Deploy do Frontend\nvia GitHub Actions"]
    O --> P

    P{Frontend OK?\nTasks aparecem na tela?}
    P -- Não --> Q["🔍 Verificar CORS\nFRONTEND_URL no backend"]
    Q --> F
    P -- Sim --> R

    R([✅ Deploy concluído!])

    style A fill:#107c10,color:#fff,stroke:none
    style R fill:#107c10,color:#fff,stroke:none
    style J fill:#fff4e6,stroke:#ff8c00
    style P fill:#fff4e6,stroke:#ff8c00
    style K fill:#ffe6e6,stroke:#d83b01
    style Q fill:#ffe6e6,stroke:#d83b01
```

---

## 4. Etapa 1 — Criar o Grupo de Recursos

O Grupo de Recursos é um contêiner lógico que agrupa todos os serviços do projeto. Isso facilita o gerenciamento e a exclusão em bloco ao final do projeto.

**Caminho no portal:** `Início > Grupos de recursos > + Criar`

| Campo | Valor recomendado |
|---|---|
| **Assinatura** | Sua assinatura ativa (ex.: "Azure for Students" ou "Pay-As-You-Go") |
| **Nome do grupo de recursos** | `rg-taskflow-prod` |
| **Região** | `Brazil South` (menor latência para usuários brasileiros) |

Clique em **Revisar + criar** → **Criar**.

---

## 5. Etapa 2 — Criar o Banco de Dados MySQL Flexible Server

O **Azure Database for MySQL - Flexible Server** é a solução gerenciada recomendada pela Microsoft (o Single Server foi aposentado em 2024). O tier **Burstable** é o mais barato e adequado para desenvolvimento e cargas de trabalho leves.

**Caminho no portal:** `Início > Criar um recurso > Bancos de dados > Azure Database for MySQL`

Na tela de opções, selecione **Servidor Flexível** e clique em **Criar**.

### Comparativo entre tiers de computação

```mermaid
quadrantChart
    title Tiers do MySQL Flexible Server — Custo vs Capacidade
    x-axis "Baixo Custo" --> "Alto Custo"
    y-axis "Baixa Capacidade" --> "Alta Capacidade"
    quadrant-1 Alto custo, Alta capacidade
    quadrant-2 Baixo custo, Alta capacidade
    quadrant-3 Baixo custo, Baixa capacidade
    quadrant-4 Alto custo, Baixa capacidade
    B1ms Burstable: [0.15, 0.18]
    B2ms Burstable: [0.28, 0.32]
    D2ds General Purpose: [0.52, 0.60]
    D4ds General Purpose: [0.72, 0.75]
    E2ds Memory Opt: [0.80, 0.85]
```

> Para este projeto, o **B1ms Burstable** está no quadrante ideal: custo mínimo com capacidade suficiente para desenvolvimento e produção leve.

### Aba "Básico"

| Campo | Valor recomendado | Observação |
|---|---|---|
| **Assinatura** | Sua assinatura | — |
| **Grupo de recursos** | `rg-taskflow-prod` | Criado na etapa anterior |
| **Nome do servidor** | `mysql-taskflow-prod` | Deve ser globalmente único no Azure |
| **Região** | `Brazil South` | Mesma região do App Service para menor latência |
| **Versão do MySQL** | `8.0` | Versão mais recente suportada |
| **Tipo de carga de trabalho** | `Para desenvolvimento ou projetos hobby` | Seleciona automaticamente o SKU menor |
| **Disponibilidade** | `Sem redundância (zona única)` | Sem alta disponibilidade — mais barato |
| **Autenticação** | `Somente autenticação MySQL` | |
| **Nome de usuário do administrador** | `adminuser` | Anote este valor |
| **Senha** | `SuaSenhaForte@123` | Anote este valor — não é recuperável depois |

### Aba "Computação + armazenamento"

Clique em **Configurar servidor** para ajustar o SKU:

| Campo | Valor recomendado | Observação |
|---|---|---|
| **Camada de computação** | `Com capacidade de intermitência (Burstable)` | Tier mais barato — ideal para dev e produção leve |
| **Tamanho da computação** | `Standard_B1ms` (1 vCore, 2 GB RAM) | ~US$ 12–15/mês na região Brazil South |
| **Armazenamento** | `20 GB` | Mínimo permitido |
| **IOPS** | `396` (padrão para 20 GB) | Suficiente para o projeto |
| **Backup Redundante** | `Localmente redundante` | Mais barato que geo-redundante |
| **Período de retenção de backup** | `7 dias` | Mínimo recomendado |

### Aba "Rede"

O diagrama abaixo ilustra a diferença entre as duas opções de conectividade disponíveis:

```mermaid
graph LR
    subgraph opcao1["✅ Opção desta aula — Acesso público com Firewall"]
        direction TB
        I1["🌐 Internet"] -->|"IP liberado no firewall"| F1["🔒 Firewall Rules\n• Seu IP local\n• Serviços do Azure"]
        F1 --> DB1[("MySQL\nFlexible Server")]
        APP1["⚙️ App Service"] -->|"Serviço Azure = liberado"| DB1
    end

    subgraph opcao2["🔐 Opção avançada — VNet Integration (produção real)"]
        direction TB
        I2["🌐 Internet"] -->|"❌ Bloqueado"| VN["🏰 Virtual Network\nPrivada"]
        VN --> DB2[("MySQL\nFlexible Server")]
        APP2["⚙️ App Service"] -->|"VNet peering\n✅ Permitido"| VN
    end

    style opcao1 fill:#e6ffe6,stroke:#107c10
    style opcao2 fill:#e6f3ff,stroke:#0078d4
    style I2 fill:#ffe6e6
```

| Campo | Valor recomendado | Observação |
|---|---|---|
| **Método de conectividade** | `Acesso público (endereços IP permitidos)` | Mais simples para este projeto didático |
| **Permitir acesso público** | ✅ Marcado | |
| **Firewall — Adicionar endereço IP atual** | ✅ Marcar | Permite acesso do seu computador local para executar migrações |
| **Permitir acesso público de qualquer serviço do Azure** | ✅ Marcado | Permite que o App Service se conecte ao banco |

> **Aviso de segurança:** Em produção real, prefira o uso de **Rede virtual privada (VNet Integration)** para isolar o banco de dados da internet pública. Para fins didáticos, o acesso público com firewall é aceitável.

### Aba "Tags" (opcional)

| Chave | Valor |
|---|---|
| `Projeto` | `TaskFlow` |
| `Ambiente` | `Producao` |
| `Disciplina` | `ISW035` |

Clique em **Revisar + criar** → **Criar**. A implantação leva cerca de 5 a 10 minutos.

---

## 6. Etapa 3 — Configurar o Banco de Dados

Após a criação do servidor, é necessário criar o banco de dados que a aplicação irá usar.

**Caminho:** `mysql-taskflow-prod > Bancos de dados > + Adicionar`

| Campo | Valor |
|---|---|
| **Nome do banco de dados** | `taskflow` |
| **Conjunto de caracteres** | `utf8mb4` |
| **Ordenação** | `utf8mb4_unicode_ci` |

Clique em **Salvar**.

### Estrutura criada pelo Prisma após as migrações

O schema do projeto (`backend/prisma/schema.prisma`) define a seguinte tabela, que será criada automaticamente na Etapa 7:

```mermaid
erDiagram
    TASK {
        int id PK "AUTO_INCREMENT"
        varchar_255 title "NOT NULL"
        text description "NULL"
        varchar_50 status "DEFAULT pending"
        datetime createdAt "DEFAULT NOW()"
        datetime updatedAt "ON UPDATE NOW()"
    }
```

### Anotar a string de conexão

Vá em `mysql-taskflow-prod > Visão geral` e copie o **Nome do servidor**. Ele terá o formato:

```
mysql-taskflow-prod.mysql.database.azure.com
```

Monte a `DATABASE_URL` no formato exigido pelo Prisma:

```
mysql://adminuser:SuaSenhaForte@123@mysql-taskflow-prod.mysql.database.azure.com:3306/taskflow?ssl=true
```

> **Importante:** O Azure Database for MySQL exige SSL por padrão. O parâmetro `?ssl=true` é obrigatório para conexões externas.

---

## 7. Etapa 4 — Criar o App Service (Backend Next.js)

**Caminho no portal:** `Início > Criar um recurso > Web > Aplicativo Web`

### Aba "Básico"

| Campo | Valor recomendado | Observação |
|---|---|---|
| **Assinatura** | Sua assinatura | — |
| **Grupo de recursos** | `rg-taskflow-prod` | — |
| **Nome** | `backend-taskflow-prod` | URL resultante: `backend-taskflow-prod.azurewebsites.net` |
| **Publicar** | `Código` | Não usaremos Docker nesta aula |
| **Pilha de runtime** | `Node 22 LTS` | Versão compatível com Next.js 16 e Prisma 7 |
| **Sistema operacional** | `Linux` | Mais barato que Windows e melhor suporte ao Node.js |
| **Região** | `Brazil South` | — |

### Comparativo entre planos do App Service

```mermaid
xychart-beta
    title "App Service Linux — Custo mensal (US$) por Plano"
    x-axis ["F1 Grátis", "B1 Básico ⭐", "B2 Básico", "S1 Standard", "P1v3 Premium"]
    y-axis "Custo/mês (US$)" 0 --> 200
    bar [0, 13, 26, 64, 170]
```

### Aba "Plano de Preços"

Clique em **Criar novo** em "Plano do Serviço de Aplicativo":

| Campo | Valor | Observação |
|---|---|---|
| **Nome do plano** | `asp-taskflow-backend` | — |
| **SKU e tamanho** | `B1 Básico` (1 vCore, 1,75 GB RAM, 10 GB) | ~US$ 13/mês no Linux |

> **Por que B1 e não F1 (Gratuito)?**  
> O plano F1 (gratuito) não suporta **domínios personalizados**, **sempre ativo (Always On)** e tem limite de 60 minutos de CPU por dia. Como o backend precisa estar responsivo o tempo todo, o B1 é o mínimo recomendado para produção.

### Aba "Implantação"

| Campo | Valor |
|---|---|
| **Implantação contínua** | `Habilitar` |
| **Conta do GitHub** | Faça login com sua conta |
| **Organização** | Sua organização/usuário GitHub |
| **Repositório** | Seu repositório do projeto |
| **Branch** | `main` |

O Azure criará automaticamente um arquivo de workflow no GitHub Actions (`.github/workflows/`).

> **Atenção:** O Azure configurará o workflow para implantar a raiz do repositório. Como o backend está na pasta `backend/`, você precisará ajustar o workflow após a criação — veja a Etapa 6.

### Aba "Monitoramento"

| Campo | Valor |
|---|---|
| **Habilitar o Application Insights** | `Não` (reduz custos para esta aula) |

Clique em **Revisar + criar** → **Criar**.

---

## 8. Etapa 5 — Configurar Variáveis de Ambiente do Backend

As variáveis de ambiente no App Service substituem o arquivo `.env` da aplicação. **Nunca suba o `.env` para o repositório.**

**Caminho:** `backend-taskflow-prod > Configuração > Configurações do aplicativo > + Nova configuração de aplicativo`

### Como as variáveis fluem entre os serviços

```mermaid
graph LR
    subgraph portal["🖥️ Portal Azure — App Service Backend"]
        V1["DATABASE_URL"]
        V2["FRONTEND_URL"]
        V3["NODE_ENV = production"]
        V4["SCM_DO_BUILD_DURING_DEPLOYMENT = true"]
        V5["NPM_CONFIG_PRODUCTION = false"]
    end

    subgraph runtime["⚙️ Next.js em Execução"]
        R1["process.env.DATABASE_URL"]
        R2["process.env.FRONTEND_URL"]
        R3["process.env.NODE_ENV"]
    end

    subgraph nextconfig["📄 next.config.ts"]
        C1["Access-Control-Allow-Origin\n= process.env.FRONTEND_URL"]
    end

    subgraph prisma["🔌 Prisma Client"]
        P1["Conexão SSL\ncom MySQL"]
    end

    subgraph mysql["🗄️ MySQL Flexible Server"]
        DB[("taskflow")]
    end

    V1 --> R1
    V2 --> R2
    V3 --> R3
    R2 --> C1
    R1 --> P1
    P1 -->|"TCP 3306 + SSL"| DB

    style portal fill:#e6f3ff,stroke:#0078d4
    style runtime fill:#e6ffe6,stroke:#107c10
    style nextconfig fill:#fff4e6,stroke:#ff8c00
    style prisma fill:#f0e6ff,stroke:#8764b8
    style mysql fill:#ffe6e6,stroke:#d83b01
```

Adicione **uma a uma** as seguintes variáveis:

| Nome da variável | Valor de exemplo | Obrigatório? |
|---|---|---|
| `DATABASE_URL` | `mysql://adminuser:Senha@mysql-taskflow-prod.mysql.database.azure.com:3306/taskflow?ssl=true` | ✅ Sim |
| `NODE_ENV` | `production` | ✅ Sim |
| `FRONTEND_URL` | `https://frontend-taskflow-prod.azurewebsites.net` | ✅ Sim (CORS) |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` | ✅ Sim (executa `npm run build`) |
| `NPM_CONFIG_PRODUCTION` | `false` | ✅ Sim (instala devDependencies necessárias para o build) |
| `WEBSITE_NODE_DEFAULT_VERSION` | `~22` | Recomendado |

Após adicionar todas as variáveis, clique em **Salvar** no topo da página. O App Service irá reiniciar automaticamente.

### Configurar o Comando de Inicialização

**Caminho:** `backend-taskflow-prod > Configuração > Configurações gerais`

| Campo | Valor |
|---|---|
| **Comando de inicialização** | `npm run start` |
| **Sempre Ativo** | `Ativado` (evita que o app "adormeça" quando sem tráfego) |

Clique em **Salvar**.

---

## 9. Etapa 6 — Fazer Deploy do Backend via GitHub Actions

O Azure já terá criado um arquivo de workflow no seu repositório. Você precisará editá-lo para apontar para a subpasta `backend/` do monorepo.

### Pipeline de CI/CD do Backend

```mermaid
sequenceDiagram
    actor Dev as 👨‍💻 Desenvolvedor
    participant GH as 🐙 GitHub
    participant GHA as ⚡ GitHub Actions
    participant AZ as ☁️ Azure App Service

    Dev->>GH: git push origin main
    GH->>GHA: Dispara workflow<br/>(paths: backend/**)

    rect rgb(230, 255, 230)
        Note over GHA: Job: build-and-deploy
        GHA->>GHA: actions/checkout@v4
        GHA->>GHA: setup-node (Node 22)
        GHA->>GHA: npm ci (instala dependências)
        GHA->>GHA: npm run build<br/>(next build + prisma generate)
    end

    rect rgb(230, 243, 255)
        Note over GHA,AZ: Deploy
        GHA->>AZ: azure/webapps-deploy@v3<br/>package: ./backend
        AZ->>AZ: npm run start<br/>(next start -p $PORT)
        AZ-->>GHA: ✅ Deploy concluído
    end

    GHA-->>Dev: ✅ Workflow concluído
    Dev->>AZ: GET /api/tasks → []
```

**Caminho no repositório:** `.github/workflows/backend-taskflow-prod.yml`

Edite o arquivo e substitua seu conteúdo pelo seguinte:

```yaml
name: Deploy Backend (Next.js) para Azure App Service

on:
  push:
    branches:
      - main
    paths:
      - 'backend/**'   # Só dispara quando arquivos do backend mudam
  workflow_dispatch:   # Permite disparo manual pelo portal do GitHub

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout do codigo
        uses: actions/checkout@v4

      - name: Configurar Node.js 22
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Instalar dependencias
        working-directory: ./backend
        run: npm ci

      - name: Build da aplicacao
        working-directory: ./backend
        run: npm run build
        env:
          # Necessario para o Prisma gerar o client durante o build
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      - name: Deploy para Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'backend-taskflow-prod'
          slot-name: 'production'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_BACKEND }}
          package: './backend'
```

### Obter o Publish Profile

**Caminho:** `backend-taskflow-prod > Visão geral > Obter perfil de publicação`

Baixe o arquivo `.PublishSettings`, abra no Notepad e copie **todo o conteúdo**.

No GitHub, vá em `Settings > Secrets and variables > Actions > New repository secret`:

| Nome do secret | Valor |
|---|---|
| `AZURE_WEBAPP_PUBLISH_PROFILE_BACKEND` | Conteúdo completo do arquivo `.PublishSettings` |
| `DATABASE_URL` | Sua string de conexão MySQL completa |

Faça um `git push` na branch `main` para disparar o primeiro deploy automático.

---

## 10. Etapa 7 — Executar Migrações do Prisma

O Prisma precisa criar as tabelas no banco de dados antes do backend funcionar. Execute as migrações uma única vez **do seu computador local** (você já adicionou seu IP ao firewall do MySQL na Etapa 2).

### Fluxo das migrações

```mermaid
sequenceDiagram
    actor Dev as 👨‍💻 Desenvolvedor (local)
    participant Prisma as 🔌 Prisma CLI
    participant FW as 🔒 Firewall MySQL
    participant MySQL as 🗄️ MySQL Flexible Server

    Dev->>Dev: Cria .env.production.local<br/>com DATABASE_URL de prod
    Dev->>Prisma: npx prisma migrate deploy

    Prisma->>FW: Tenta conexão TCP 3306
    FW->>FW: Verifica IP do Dev<br/>na lista de regras
    FW-->>Prisma: ✅ IP autorizado

    Prisma->>MySQL: Conecta com SSL
    MySQL-->>Prisma: Conexão estabelecida

    Prisma->>MySQL: Verifica tabela _prisma_migrations
    MySQL-->>Prisma: Tabela não existe (primeira vez)

    loop Para cada arquivo em prisma/migrations/
        Prisma->>MySQL: Executa SQL da migration
        MySQL-->>Prisma: ✅ Aplicada com sucesso
        Prisma->>MySQL: Registra em _prisma_migrations
    end

    Prisma-->>Dev: ✅ Todas as migrations aplicadas
    Dev->>MySQL: npx prisma db pull (verificação)
    MySQL-->>Dev: Schema confirmado ✅
```

No seu terminal, dentro da pasta `backend/`:

```bash
# Crie um arquivo temporario com a DATABASE_URL de producao
# NUNCA commite este arquivo!
echo 'DATABASE_URL="mysql://adminuser:SuaSenhaForte@123@mysql-taskflow-prod.mysql.database.azure.com:3306/taskflow?ssl=true"' > .env.production.local

# Executar as migracoes em producao
npx dotenv -e .env.production.local -- npx prisma migrate deploy

# Verificar se as tabelas foram criadas corretamente
npx dotenv -e .env.production.local -- npx prisma db pull
```

Após as migrações, acesse o backend no navegador:

```
https://backend-taskflow-prod.azurewebsites.net/api/tasks
```

Você deverá receber um JSON com array vazio: `[]`

> **Alternativa sem expor o banco localmente:** Você também pode usar o **Azure Cloud Shell** diretamente no portal para executar estes comandos, sem precisar adicionar seu IP ao firewall.

---

## 11. Etapa 8 — Criar o App Service (Frontend React)

**Caminho no portal:** `Início > Criar um recurso > Web > Aplicativo Web`

### Aba "Básico"

| Campo | Valor recomendado | Observação |
|---|---|---|
| **Assinatura** | Sua assinatura | — |
| **Grupo de recursos** | `rg-taskflow-prod` | — |
| **Nome** | `frontend-taskflow-prod` | URL: `frontend-taskflow-prod.azurewebsites.net` |
| **Publicar** | `Código` | — |
| **Pilha de runtime** | `Node 22 LTS` | Para servir os arquivos de build com `vite preview` |
| **Sistema operacional** | `Linux` | — |
| **Região** | `Brazil South` | — |

### Aba "Plano de Preços"

| Campo | Valor | Observação |
|---|---|---|
| **Plano do Serviço de Aplicativo** | `Criar novo: asp-taskflow-frontend` | Plano separado do backend |
| **SKU e tamanho** | `B1 Básico` | ~US$ 13/mês no Linux |

> **Dica de economia:** Como o frontend React/Vite gera arquivos estáticos após o build, o plano **F1 Gratuito** pode ser usado aqui para reduzir custos. O Always On não é crítico para servir arquivos estáticos.

### Aba "Implantação"

Configure da mesma forma que o backend, apontando para o mesmo repositório e branch `main`.

### Aba "Monitoramento"

| Campo | Valor |
|---|---|
| **Habilitar o Application Insights** | `Não` |

Clique em **Revisar + criar** → **Criar**.

---

## 12. Etapa 9 — Configurar Variáveis de Ambiente do Frontend

**Caminho:** `frontend-taskflow-prod > Configuração > Configurações do aplicativo`

### Como as variáveis VITE_ funcionam (ponto crítico!)

```mermaid
flowchart LR
    subgraph buildtime["🔨 Tempo de BUILD (GitHub Actions)"]
        S["🔐 GitHub Secret\nVITE_API_URL"]
        S -->|"env: VITE_API_URL"| VB["vite build"]
        VB -->|"embutido no JS"| BUNDLE["📦 dist/assets/index.js\n'https://backend.../api'"]
    end

    subgraph runtime["🌐 Tempo de EXECUÇÃO (navegador do usuário)"]
        BUNDLE -->|"baixado pelo browser"| BR["💻 Navegador\nimport.meta.env.VITE_API_URL\n= 'https://backend.../api'"]
        BR -->|"requisição HTTPS"| API["⚙️ Backend API"]
    end

    subgraph wrongway["❌ O que NÃO funciona"]
        AZ_VAR["Azure App Service\nVariável: VITE_API_URL"] -->|"runtime apenas"| SERVER["⚙️ Node.js Server\n(só process.env)"]
        SERVER -->|"NÃO chega ao"| BROWSER["💻 Navegador"]
    end

    style buildtime fill:#e6ffe6,stroke:#107c10
    style runtime fill:#e6f3ff,stroke:#0078d4
    style wrongway fill:#ffe6e6,stroke:#d83b01
    style AZ_VAR fill:#ffe6e6
    style SERVER fill:#ffe6e6
    style BROWSER fill:#ffe6e6
```

> **Conclusão:** Variáveis `VITE_` precisam estar nos **GitHub Secrets** e passadas no step de build do workflow — não apenas no App Service do Azure.

| Nome da variável | Valor | Observação |
|---|---|---|
| `VITE_API_URL` | `https://backend-taskflow-prod.azurewebsites.net/api` | Necessário em tempo de **build** |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` | — |
| `NPM_CONFIG_PRODUCTION` | `false` | — |

### Comando de Inicialização

**Caminho:** `frontend-taskflow-prod > Configuração > Configurações gerais`

| Campo | Valor |
|---|---|
| **Comando de inicialização** | `npx vite preview --port $PORT --host` |

Clique em **Salvar**.

---

## 13. Etapa 10 — Fazer Deploy do Frontend via GitHub Actions

### Pipeline de CI/CD do Frontend

```mermaid
sequenceDiagram
    actor Dev as 👨‍💻 Desenvolvedor
    participant GH as 🐙 GitHub
    participant GHA as ⚡ GitHub Actions
    participant AZ as ☁️ Azure App Service

    Dev->>GH: git push origin main
    GH->>GHA: Dispara workflow<br/>(paths: frontend/**)

    rect rgb(255, 244, 230)
        Note over GHA: Build com variáveis de ambiente
        GHA->>GHA: actions/checkout@v4
        GHA->>GHA: setup-node (Node 22)
        GHA->>GHA: npm ci
        GHA->>GHA: npm run build<br/>env: VITE_API_URL=secrets.VITE_API_URL<br/>↳ Embutida no bundle JS
    end

    rect rgb(230, 243, 255)
        Note over GHA,AZ: Deploy dos arquivos estáticos
        GHA->>AZ: azure/webapps-deploy@v3<br/>package: ./frontend
        AZ->>AZ: npx vite preview --port $PORT --host
        AZ-->>GHA: ✅ Deploy concluído
    end

    GHA-->>Dev: ✅ Workflow concluído
    Dev->>AZ: Acessa frontend-taskflow-prod.azurewebsites.net
```

Edite o arquivo de workflow gerado pelo Azure ou crie um novo em `.github/workflows/frontend-taskflow-prod.yml`:

```yaml
name: Deploy Frontend (React/Vite) para Azure App Service

on:
  push:
    branches:
      - main
    paths:
      - 'frontend/**'
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout do codigo
        uses: actions/checkout@v4

      - name: Configurar Node.js 22
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Instalar dependencias
        working-directory: ./frontend
        run: npm ci

      - name: Build da aplicacao
        working-directory: ./frontend
        run: npm run build
        env:
          # Variaveis VITE_ precisam estar aqui para serem embutidas no bundle
          VITE_API_URL: ${{ secrets.VITE_API_URL }}

      - name: Deploy para Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: 'frontend-taskflow-prod'
          slot-name: 'production'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_FRONTEND }}
          package: './frontend'
```

### Secrets no GitHub para o Frontend

No GitHub, em `Settings > Secrets and variables > Actions`, adicione:

| Nome do secret | Valor |
|---|---|
| `AZURE_WEBAPP_PUBLISH_PROFILE_FRONTEND` | Conteúdo do `.PublishSettings` do **frontend** |
| `VITE_API_URL` | `https://backend-taskflow-prod.azurewebsites.net/api` |

Faça um `git push` para disparar o deploy do frontend.

---

## 14. Etapa 11 — Verificar CORS no Backend

### Como o CORS funciona neste projeto

```mermaid
sequenceDiagram
    actor User as 👤 Usuário
    participant Browser as 💻 Navegador
    participant FE as 🌐 Frontend<br/>frontend-taskflow-prod.azurewebsites.net
    participant BE as ⚙️ Backend<br/>backend-taskflow-prod.azurewebsites.net

    User->>Browser: Acessa o frontend
    Browser->>FE: GET /
    FE-->>Browser: HTML + JS (bundle com VITE_API_URL)

    Note over Browser: Usuário clica em "Nova task"

    Browser->>BE: OPTIONS /api/tasks<br/>Origin: https://frontend-taskflow-prod...
    BE->>BE: Verifica FRONTEND_URL<br/>no process.env
    BE-->>Browser: 200 OK<br/>Access-Control-Allow-Origin: https://frontend-taskflow-prod...

    Browser->>BE: POST /api/tasks<br/>{ "title": "Minha task" }
    BE->>BE: Prisma.task.create()
    BE-->>Browser: 201 Created<br/>{ id: 1, title: "Minha task", ... }

    Browser-->>User: ✅ Task aparece na lista
```

O arquivo `backend/next.config.ts` do projeto já contém a configuração de CORS que usa a variável `FRONTEND_URL`:

```typescript
// backend/next.config.ts — configuracao ja existente no projeto
{ key: "Access-Control-Allow-Origin", 
  value: process.env.FRONTEND_URL ?? "http://localhost:5173" }
```

Esta configuração funciona automaticamente desde que a variável `FRONTEND_URL` esteja corretamente definida no App Service do backend (Etapa 5).

**Teste final:** Acesse o frontend pelo navegador e tente criar uma task. Se aparecer no painel, o deploy foi concluído com sucesso!

---

## 15. Referência rápida de preços

Todos os valores são aproximados em USD para a região **Brazil South**. Os valores em BRL variam conforme câmbio e IOF.

### Azure Database for MySQL — Flexible Server (Burstable)

| SKU | vCores | RAM | Armazenamento | Custo estimado/mês |
|---|---|---|---|---|
| **Standard_B1ms** ⭐ | 1 | 2 GB | 20 GB (incluso) | ~US$ 12–15 |
| Standard_B2ms | 2 | 4 GB | 20 GB (incluso) | ~US$ 24–28 |
| Standard_D2ds_v4 | 2 | 8 GB | 20 GB (incluso) | ~US$ 45–50 |

> ⭐ = opção mais barata recomendada para este projeto.  
> Armazenamento adicional custa ~US$ 0,115/GB/mês.

### Azure App Service — Linux

| Plano | vCPU | RAM | Armazenamento | Custo/mês |
|---|---|---|---|---|
| F1 Gratuito | Compartilhado | 1 GB | 1 GB | US$ 0 (60 min CPU/dia) |
| **B1 Básico** ⭐ | 1 | 1,75 GB | 10 GB | ~US$ 13 |
| B2 Básico | 2 | 3,5 GB | 10 GB | ~US$ 26 |
| S1 Standard | 1 | 1,75 GB | 50 GB | ~US$ 64 |

> O plano **F1 é suficiente para o frontend** em ambiente acadêmico.  
> O plano **B1 é o mínimo para o backend** com Next.js em produção.

### Custo total estimado do projeto

```mermaid
pie title Distribuição de custo mensal (~US$ 39/mês)
    "MySQL Flexible Server B1ms" : 13
    "App Service Backend B1" : 13
    "App Service Frontend B1" : 13
```

| Recurso | SKU escolhido | Custo/mês |
|---|---|---|
| MySQL Flexible Server | Standard_B1ms | ~US$ 13 |
| App Service — Backend | B1 Linux | ~US$ 13 |
| App Service — Frontend | B1 Linux (ou F1 Gratuito) | ~US$ 13 (ou US$ 0) |
| **Total** | | **~US$ 39/mês** (ou ~US$ 26 com F1 para frontend) |

> Com o crédito gratuito de **US$ 200** disponível em contas de estudante Azure, é possível rodar o projeto por aproximadamente **5 meses** sem custo adicional.

---

## 16. Solução de problemas comuns

```mermaid
flowchart TD
    START(["🐛 Problema identificado"]) --> Q1

    Q1{"Qual sintoma?"}

    Q1 -->|"Tela branca /\nApplication Error"| A1
    Q1 -->|"Erro de CORS\nno navegador"| A2
    Q1 -->|"P1001 / ETIMEDOUT\n(banco)"| A3
    Q1 -->|"VITE_API_URL\nundefined"| A4
    Q1 -->|"migrate deploy\nfalhando"| A5

    A1["📋 Verificar Log Stream\nno portal Azure"]
    A1 --> B1{"build OK?\nstart -p $PORT?"}
    B1 -- Não --> C1["Corrigir package.json\ne variáveis de build"]
    B1 -- Sim --> C1B["Verificar SCM_DO_BUILD\n= true"]

    A2["Verificar FRONTEND_URL\nno App Service do Backend"]
    A2 --> B2{"URL correta\nsem barra final?"}
    B2 -- Não --> C2["Corrigir e Salvar\nno portal Azure"]
    B2 -- Sim --> C2B["Reiniciar App Service\nmanualmente"]

    A3["Verificar Firewall\ndo MySQL"]
    A3 --> B3{"Serviços Azure\nliberados?"}
    B3 -- Não --> C3["Ativar em:\nmysql > Rede"]
    B3 -- Sim --> C3B["Verificar ?ssl=true\nna DATABASE_URL"]

    A4["VITE_ precisa estar\nno build, não runtime"]
    A4 --> B4{"Secret no\nGitHub Actions?"}
    B4 -- Não --> C4["Criar secret e\nadicionar no env: do step build"]
    B4 -- Sim --> C4B["Reexecutar workflow\npara rebuild"]

    A5["Verificar banco e\nmigrations"]
    A5 --> B5{"Banco 'taskflow'\nexiste?"}
    B5 -- Não --> C5["Criar em:\nmysql > Bancos de dados"]
    B5 -- Sim --> C5B["Verificar pasta\nprisma/migrations/"]

    style START fill:#d83b01,color:#fff,stroke:none
    style C1 fill:#e6ffe6,stroke:#107c10
    style C1B fill:#e6ffe6,stroke:#107c10
    style C2 fill:#e6ffe6,stroke:#107c10
    style C2B fill:#e6ffe6,stroke:#107c10
    style C3 fill:#e6ffe6,stroke:#107c10
    style C3B fill:#e6ffe6,stroke:#107c10
    style C4 fill:#e6ffe6,stroke:#107c10
    style C4B fill:#e6ffe6,stroke:#107c10
    style C5 fill:#e6ffe6,stroke:#107c10
    style C5B fill:#e6ffe6,stroke:#107c10
```

### Erro: "Application Error" ao acessar o backend

**Causa provável:** O comando de inicialização está errado ou o build falhou.

**Solução:**
1. Acesse `backend-taskflow-prod > Log stream` no portal
2. Verifique se `npm run build` foi executado com sucesso
3. Confirme que `SCM_DO_BUILD_DURING_DEPLOYMENT = true`
4. Verifique se o script `"start": "next start -p $PORT"` existe no `package.json` do backend

---

### Erro de CORS no frontend (bloqueado pelo navegador)

**Causa provável:** A variável `FRONTEND_URL` no backend está errada ou ausente.

**Solução:**
1. Acesse `backend-taskflow-prod > Configuração > Configurações do aplicativo`
2. Confirme que `FRONTEND_URL = https://frontend-taskflow-prod.azurewebsites.net` (sem barra no final)
3. Clique em **Salvar** — o App Service reiniciará automaticamente

---

### Erro de conexão com o banco de dados (P1001 ou ETIMEDOUT)

**Causa provável:** Firewall do MySQL bloqueando a conexão, `DATABASE_URL` incorreta ou SSL ausente.

**Solução:**
1. Acesse `mysql-taskflow-prod > Rede`
2. Confirme que **"Permitir acesso público de qualquer serviço do Azure"** está ativado
3. Verifique a `DATABASE_URL` — ela deve conter `?ssl=true` no final
4. Teste a conexão localmente com `npx prisma db pull`

---

### As variáveis `VITE_` aparecem como `undefined` no frontend

**Causa provável:** As variáveis `VITE_` precisam ser injetadas **durante o build**, não em runtime.

**Solução:**
1. Adicione `VITE_API_URL` como **secret no GitHub** (não apenas no App Service do Azure)
2. Passe o secret no step de build do workflow:
   ```yaml
   env:
     VITE_API_URL: ${{ secrets.VITE_API_URL }}
   ```
3. Refaça o deploy após adicionar o secret — o build precisa ser reexecutado

---

### O Prisma retorna erro ao fazer `migrate deploy`

**Causa provável:** As migrações não foram encontradas ou o banco de dados não foi criado.

**Solução:**
1. Confirme que o banco de dados `taskflow` existe em `mysql-taskflow-prod > Bancos de dados`
2. Verifique se a pasta `backend/prisma/migrations/` existe no repositório
3. Execute localmente com seu IP liberado no firewall:
   ```bash
   npx dotenv -e .env.production.local -- npx prisma migrate deploy
   ```

---

## Conclusão

Nesta aula você aprendeu a:

- Criar e configurar um **Grupo de Recursos** para organizar todos os serviços em um único lugar
- Provisionar um **Azure Database for MySQL Flexible Server** no tier mais econômico (Burstable B1ms)
- Criar dois **App Services** separados (backend e frontend) no plano Basic B1 para Linux
- Configurar **variáveis de ambiente** via portal sem expor segredos no código-fonte
- Automatizar o deploy com **GitHub Actions** para subpastas de um monorepo
- Executar **migrações do Prisma** em ambiente de produção com segurança
- Verificar a configuração de **CORS** entre os dois serviços

**Próximos passos sugeridos:**

- Configurar um **domínio personalizado** com certificado SSL gratuito
- Habilitar o **Application Insights** para monitoramento de erros e performance
- Explorar **Deployment Slots** para implementar estratégias de blue-green deployment
- Estudar **VNet Integration** para isolar o banco de dados da internet pública

---

*Documentação oficial de referência:*  
*[learn.microsoft.com/azure/app-service](https://learn.microsoft.com/azure/app-service) | [learn.microsoft.com/azure/mysql/flexible-server](https://learn.microsoft.com/azure/mysql/flexible-server/overview)*
