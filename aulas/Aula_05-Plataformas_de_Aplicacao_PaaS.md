# Aula 05 — Plataformas de Aplicação — PaaS

> **Disciplina:** Computação em Nuvem II (ISW035)  
> **Professor:** Ronan Adriel Zenatti — FATEC Jahu / Centro Paula Souza  
> **Semestre:** 1º/2026  
> **Carga Horária:** 4h práticas  
> **Entregável:** Aplicação rodando no Azure App Service e Google App Engine com URLs públicas

---

## 1. Visão Geral e Contextualização

Nas aulas anteriores, provisionamos a infraestrutura de dados: storage (Aulas 02-03) e bancos de dados (Aula 04). Agora, chegamos ao momento de **hospedar a aplicação** em si. Nesta aula, exploramos o modelo **PaaS (Platform as a Service)**, onde o provedor gerencia completamente o sistema operacional, o runtime da linguagem, o load balancer e o escalonamento — você apenas faz deploy do código.

### IaaS vs. PaaS vs. Serverless — Onde Estamos?

```mermaid
graph TB
    subgraph IaaS["IaaS — Você gerencia quase tudo"]
        I1["✅ Aplicação"]
        I2["✅ Runtime / Middleware"]
        I3["✅ Sistema Operacional"]
        I4["☁️ Virtualização"]
        I5["☁️ Hardware"]
    end

    subgraph PaaS["PaaS — Foco no código ⭐ ESTA AULA"]
        P1["✅ Aplicação"]
        P2["☁️ Runtime / Middleware"]
        P3["☁️ Sistema Operacional"]
        P4["☁️ Virtualização"]
        P5["☁️ Hardware"]
    end

    subgraph Serverless["Serverless — Foco na função"]
        S1["✅ Função / Handler"]
        S2["☁️ Tudo gerenciado"]
    end

    style PaaS fill:#2a9d8f,color:#fff
```

O PaaS é o equilíbrio ideal para a maioria das aplicações web: você mantém controle total sobre o código e a configuração da aplicação, mas não precisa se preocupar com patching de SO, configuração de web servers ou gerenciamento de infraestrutura. Isso reduz drasticamente o tempo de deploy e a sobrecarga operacional.

### Mapa de Equivalência — PaaS

| Conceito | Microsoft Azure | Google Cloud |
|---|---|---|
| PaaS principal para web apps | **App Service** | **App Engine** |
| Ambientes | Windows / Linux | Standard / Flexible |
| Deploy via Git | Sim (deployment center) | Sim (`gcloud app deploy`) |
| Deploy via container | Sim (custom containers) | Sim (Flexible environment) |
| Variáveis de ambiente | Application Settings | `app.yaml` env_variables / Secret Manager |
| Deployment slots | Sim (Staging, Production, etc.) | Sim (Traffic Splitting por versão) |
| Escalonamento automático | Autoscale rules | Automático (Standard) / Configurável (Flexible) |
| Custom domains + SSL | Sim (gratuito com managed certificate) | Sim (gratuito com managed certificate) |
| CI/CD integrado | Azure DevOps / GitHub Actions | Cloud Build / GitHub integração |

---

## 2. Azure App Service

### 2.1 Conceitos Fundamentais

O **Azure App Service** é a plataforma PaaS principal do Azure para hospedagem de aplicações web, APIs REST e backends móveis. Ele suporta múltiplas linguagens e frameworks nativamente, além de containers personalizados.

**Linguagens e runtimes suportados nativamente:**

| Linguagem | Versões Suportadas | Framework Popular |
|---|---|---|
| .NET | .NET 6, 7, 8 | ASP.NET Core, Blazor |
| Java | 8, 11, 17, 21 | Spring Boot, Quarkus |
| Node.js | 16, 18, 20 | Express, NestJS |
| Python | 3.9, 3.10, 3.11, 3.12 | Flask, Django, FastAPI |
| PHP | 8.1, 8.2, 8.3 | Laravel |
| Ruby | 2.7, 3.0, 3.1 | Rails |
| Custom Container | Docker | Qualquer linguagem |

### 2.2 App Service Plans (Planos de Serviço)

O App Service Plan define os recursos computacionais (CPU, memória) alocados para suas aplicações. Múltiplas aplicações podem compartilhar o mesmo plano.

| Tier | Instâncias | CPU/RAM | Features | Preço (aprox.) |
|---|---|---|---|---|
| **Free (F1)** | 1 | Compartilhado / 1 GB | 60 min CPU/dia, sem custom domain | Grátis |
| **Basic (B1)** | Até 3 | 1 core / 1.75 GB | Custom domains, SSL, manual scale | ~$13/mês |
| **Standard (S1)** | Até 10 | 1 core / 1.75 GB | Autoscale, deployment slots (5), backups | ~$70/mês |
| **Premium v3 (P1v3)** | Até 30 | 2 cores / 8 GB | Slots (20), VNet integration, zone redundancy | ~$138/mês |
| **Isolated v2 (I1v2)** | Até 100 | 2 cores / 8 GB | App Service Environment (ASE), rede isolada | ~$302/mês |

### 2.3 Deployment Slots

Uma das funcionalidades mais poderosas do App Service (a partir do tier Standard) são os **deployment slots** — ambientes adicionais de deploy associados à mesma aplicação, com URLs próprias. Isso permite:

- Deploy em staging → testar → swap para produção com **zero downtime**
- Validação de novas versões antes de expô-las aos usuários
- Rollback instantâneo (swap novamente para a versão anterior)

```mermaid
graph LR
    DEV["👨‍💻 Developer"] -->|"git push / deploy"| STAGING["🔶 Slot: staging<br/>app-cnuvem2-staging.azurewebsites.net"]
    STAGING -->|"Testar e validar"| QA["✅ QA aprovado"]
    QA -->|"az webapp<br/>deployment slot swap"| PROD["🟢 Slot: production<br/>app-cnuvem2.azurewebsites.net"]
    PROD -->|"Rollback<br/>(swap reverso)"| STAGING
```

### 2.4 Deploy Prático — Azure App Service com Python/Flask

```bash
# 1. Criar App Service Plan (Linux, Basic)
az appservice plan create \
    --resource-group rg-cnuvem2 \
    --name plan-cnuvem2 \
    --is-linux \
    --sku B1

# 2. Criar Web App (Python 3.12)
az webapp create \
    --resource-group rg-cnuvem2 \
    --plan plan-cnuvem2 \
    --name app-cnuvem2-2026 \
    --runtime "PYTHON:3.12"

# 3. Configurar variáveis de ambiente
az webapp config appsettings set \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026 \
    --settings \
        DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require" \
        STORAGE_CONNECTION_STRING="DefaultEndpointsProtocol=https;..." \
        FLASK_ENV="production" \
        SECRET_KEY="chave-segura-aqui"

# 4. Deploy via Git local
az webapp deployment source config-local-git \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026

# Obter URL do Git remoto
az webapp deployment list-publishing-credentials \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026 \
    --query scmUri -o tsv

# No diretório do projeto:
git remote add azure <URL-do-SCM>
git push azure main

# 5. Verificar deploy
az webapp browse --resource-group rg-cnuvem2 --name app-cnuvem2-2026
# URL: https://app-cnuvem2-2026.azurewebsites.net
```

**Alternativa — Deploy via GitHub Actions:**

```bash
# Configurar deploy contínuo direto do GitHub
az webapp deployment github-actions add \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026 \
    --repo SEU_USUARIO/cnuvem2-projeto \
    --branch main \
    --runtime "PYTHON:3.12"
```

**Estrutura mínima da aplicação Flask:**

```
projeto/
├── app.py              # Aplicação Flask
├── requirements.txt    # Dependências Python
└── startup.txt         # Comando de inicialização (opcional)
```

```python
# app.py
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({
        "servico": "Computação em Nuvem II",
        "plataforma": "Azure App Service",
        "ambiente": os.environ.get("FLASK_ENV", "development")
    })

@app.route("/health")
def health():
    return jsonify({"status": "ok"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
```

```
# requirements.txt
flask==3.1.*
gunicorn==23.*
psycopg2-binary==2.9.*
azure-storage-blob==12.*
```

---

## 3. Google App Engine

### 3.1 Conceitos Fundamentais

O **Google App Engine** foi o primeiro serviço PaaS da Google Cloud (lançado em 2008) e continua sendo uma das formas mais simples de hospedar aplicações web no GCP. Ele oferece dois ambientes com características distintas.

### 3.2 Standard vs. Flexible Environment

| Característica | Standard Environment | Flexible Environment |
|---|---|---|
| **Modelo** | Sandbox gerenciado, altamente otimizado | Container Docker gerenciado |
| **Linguagens** | Python, Java, Node.js, Go, PHP, Ruby | Qualquer (via Dockerfile) |
| **Cold start** | Segundos (instâncias leves) | Minutos (VMs subjacentes) |
| **Scale to zero** | ✅ Sim (reduz custos quando inativo) | ❌ Não (mínimo 1 instância) |
| **Escalonamento** | Automático (milissegundos) | Automático (mais lento) |
| **Acesso à rede** | Limitado (sem sockets arbitrários) | Total (pode acessar VPC) |
| **Customização SO** | Nenhuma | Dockerfile completo |
| **SSH** | Não | Sim (`gcloud app instances ssh`) |
| **Preço** | Baseado em instance hours (pode ser $0) | Baseado em vCPU, memória, disco |
| **Free tier** | 28 instance hours/dia (F1) | Não |
| **Melhor para** | APIs leves, microserviços, protótipos | Apps complexas, dependências nativas, VPC |

> **Recomendação para a disciplina:** Use o **Standard Environment** para os exercícios desta aula (é gratuito no tier F1 e escala a zero). Reserve o Flexible para quando precisar de dependências específicas ou acesso à VPC (Aula 12).

### 3.3 Configuração com app.yaml

Toda aplicação App Engine é configurada por um arquivo `app.yaml` na raiz do projeto. Este arquivo define o runtime, variáveis de ambiente, escalonamento e handlers.

```yaml
# app.yaml — Configuração do App Engine (Standard, Python 3.12)
runtime: python312

# Variáveis de ambiente
env_variables:
  FLASK_ENV: "production"
  DATABASE_URL: "postgresql://user:pass@/db?host=/cloudsql/PROJECT:REGION:INSTANCE"

# Escalonamento automático
automatic_scaling:
  min_instances: 0        # Scale to zero (economia!)
  max_instances: 5
  target_cpu_utilization: 0.65
  min_pending_latency: 30ms
  max_pending_latency: automatic

# Handlers (rotas estáticas e dinâmicas)
handlers:
  - url: /static
    static_dir: static    # Servir arquivos estáticos diretamente
  - url: /.*
    script: auto          # Todas as outras rotas → aplicação
```

### 3.4 Deploy Prático — App Engine com Python/Flask

```bash
# 1. Garantir que o App Engine está habilitado no projeto
gcloud app create --region=southamerica-east1

# 2. Estrutura do projeto
# projeto/
# ├── app.yaml
# ├── main.py
# ├── requirements.txt
# └── static/
#     └── style.css

# 3. Deploy!
gcloud app deploy --quiet

# 4. Abrir no navegador
gcloud app browse
# URL: https://PROJECT_ID.rj.r.appspot.com
```

```python
# main.py (App Engine usa "main" como ponto de entrada padrão)
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({
        "servico": "Computação em Nuvem II",
        "plataforma": "Google App Engine",
        "ambiente": os.environ.get("FLASK_ENV", "development"),
        "versao": os.environ.get("GAE_VERSION", "local")
    })

@app.route("/health")
def health():
    return jsonify({"status": "ok"}), 200

# App Engine Standard roda via gunicorn automaticamente
# Não precisa de if __name__ == "__main__" em produção,
# mas é útil para desenvolvimento local:
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```
# requirements.txt
flask==3.1.*
gunicorn==23.*
psycopg2-binary==2.9.*
google-cloud-storage==2.*
cloud-sql-python-connector[pg8000]==1.*
```

### 3.5 Traffic Splitting (Equivalente a Deployment Slots)

Enquanto o Azure usa deployment slots para gerenciar múltiplas versões, o App Engine usa **versionamento com traffic splitting**. Cada deploy cria uma nova versão da aplicação, e você pode dividir o tráfego entre versões.

```bash
# Deploy cria versão automaticamente (ex.: v1, v2)
gcloud app deploy --version=v2 --no-promote
# --no-promote: NÃO direciona tráfego automaticamente

# Verificar versões disponíveis
gcloud app versions list

# Dividir tráfego (80% v1, 20% v2 — canary deployment)
gcloud app services set-traffic default \
    --splits=v1=0.8,v2=0.2

# Promover v2 para 100% do tráfego
gcloud app services set-traffic default \
    --splits=v2=1.0

# Rollback para v1
gcloud app services set-traffic default \
    --splits=v1=1.0

# Remover versão antiga
gcloud app versions delete v1
```

```mermaid
graph LR
    USERS["👥 Usuários"] -->|"100%"| LB["⚖️ Load Balancer<br/>App Engine"]
    LB -->|"80%"| V1["🟢 Versão 1<br/>(produção atual)"]
    LB -->|"20%"| V2["🔶 Versão 2<br/>(canary)"]
    
    V2 -->|"Testes OK?<br/>Promover para 100%"| V2P["🟢 Versão 2<br/>(nova produção)"]
```

---

## 4. Comparativo Detalhado — App Service vs. App Engine

| Aspecto | Azure App Service | Google App Engine |
|---|---|---|
| **Modelo de deploy** | Git, ZIP, Docker, GitHub Actions | `gcloud app deploy`, Git, Cloud Build |
| **Arquivo de config** | Portal / `appsettings.json` / CLI | `app.yaml` |
| **Variáveis de ambiente** | Application Settings (portal/CLI) | `app.yaml` env_variables + Secret Manager |
| **Deployment slots** | Slots nomeados (staging, canary) | Versões com traffic splitting |
| **Zero-downtime deploy** | Slot swap (instantâneo) | Traffic splitting (gradual) |
| **Scale to zero** | Não (mínimo 1 instância no plano) | Sim (Standard environment) |
| **Free tier** | F1 (60 min CPU/dia, sem SLA) | Standard F1 (28 instance-hours/dia) |
| **Custom domains** | Sim (managed certificate grátis) | Sim (managed certificate grátis) |
| **VNet integration** | Sim (Premium+) | Sim (Flexible environment) |
| **WebSockets** | Sim | Sim (Flexible) / Limitado (Standard) |
| **Startup command** | Configurável via portal/CLI | Definido por `entrypoint` no `app.yaml` |
| **Logs** | Log Stream + App Insights | Cloud Logging (Stackdriver) |
| **Max request timeout** | 230 segundos | 60 seg (Standard) / 60 min (Flexible) |
| **Storage efêmero** | Disco local (não persistente entre restarts) | Disco local (não persistente) |
| **URL padrão** | `<app>.azurewebsites.net` | `<project>.appspot.com` |

---

## 5. Conectando a Aplicação aos Serviços de Dados

A verdadeira potência do PaaS aparece quando você integra a aplicação hospedada com os serviços de dados provisionados nas aulas anteriores.

### 5.1 Padrão de Arquitetura

```mermaid
graph TB
    subgraph Frontend["🌐 Navegador / Mobile"]
        USER["👤 Usuário"]
    end

    subgraph PaaS["☁️ PaaS Layer"]
        AZ_APP["Azure App Service<br/>app-cnuvem2.azurewebsites.net"]
        GCP_APP["App Engine<br/>projeto.appspot.com"]
    end

    subgraph Data["🗄️ Data Layer"]
        AZ_DB["Azure DB for PostgreSQL"]
        GCP_DB["Cloud SQL PostgreSQL"]
        AZ_STOR["Azure Blob Storage"]
        GCP_STOR["Cloud Storage"]
        AZ_NOSQL["Cosmos DB"]
        GCP_NOSQL["Firestore"]
    end

    USER -->|"HTTPS"| AZ_APP
    USER -->|"HTTPS"| GCP_APP

    AZ_APP -->|"TLS"| AZ_DB
    AZ_APP -->|"SDK"| AZ_STOR
    AZ_APP -->|"SDK"| AZ_NOSQL

    GCP_APP -->|"Cloud SQL Proxy<br/>(Unix socket)"| GCP_DB
    GCP_APP -->|"SDK"| GCP_STOR
    GCP_APP -->|"SDK"| GCP_NOSQL
```

### 5.2 Connection Strings e Variáveis de Ambiente

A prática recomendada é **nunca hardcodar** credenciais no código. Use variáveis de ambiente, e em produção, integre com serviços de gerenciamento de segredos.

**Azure — Configurando variáveis de ambiente:**

```bash
az webapp config appsettings set \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026 \
    --settings \
        DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://kv-cnuvem2.vault.azure.net/secrets/db-url)" \
        AZURE_STORAGE_CONNECTION_STRING="@Microsoft.KeyVault(...)" \
        SECRET_KEY="@Microsoft.KeyVault(...)"
```

**GCP — Usando Secret Manager com App Engine:**

```yaml
# app.yaml — Referenciando secrets
env_variables:
  FLASK_ENV: "production"

# Para secrets sensíveis, use o Secret Manager via código:
# from google.cloud import secretmanager
# client = secretmanager.SecretManagerServiceClient()
# response = client.access_secret_version(name="projects/P/secrets/db-url/versions/latest")
# DATABASE_URL = response.payload.data.decode("UTF-8")
```

### 5.3 Conectando App Engine ao Cloud SQL

O App Engine se conecta ao Cloud SQL de forma especial: via **Unix socket** no Standard environment ou via **IP privado** no Flexible. Isso elimina a necessidade de configurar IPs autorizados.

```python
# Conexão App Engine → Cloud SQL (via Unix socket)
import os
import sqlalchemy

def get_db_connection():
    """Cria conexão com Cloud SQL usando Unix socket (App Engine Standard)."""
    
    db_user = os.environ.get("DB_USER", "app_user")
    db_pass = os.environ.get("DB_PASS")
    db_name = os.environ.get("DB_NAME", "app_projeto")
    
    # No App Engine Standard, o Cloud SQL é acessado via socket
    instance_connection_name = os.environ.get(
        "INSTANCE_CONNECTION_NAME",
        "projeto-id:southamerica-east1:pg-cnuvem2-2026"
    )
    
    # Caminho do Unix socket fornecido pelo App Engine
    unix_socket_path = f"/cloudsql/{instance_connection_name}"
    
    pool = sqlalchemy.create_engine(
        sqlalchemy.engine.url.URL.create(
            drivername="postgresql+pg8000",
            username=db_user,
            password=db_pass,
            database=db_name,
            query={"unix_sock": f"{unix_socket_path}/.s.PGSQL.5432"}
        ),
        pool_size=5,
        max_overflow=2,
        pool_timeout=30,
        pool_recycle=1800,
    )
    
    return pool
```

### 5.4 Exemplos Práticos de Integração

**Exemplo 1 — API de produtos com upload de imagens:** Uma API Flask recebe dados de produtos (JSON) e imagens (multipart/form-data). O metadata do produto vai para o PostgreSQL (Azure SQL / Cloud SQL), enquanto a imagem vai para o storage (Blob / GCS). A URL da imagem é armazenada no banco. No frontend, a imagem é servida via Signed URL/SAS gerada pelo backend.

**Exemplo 2 — Dashboard com dados em tempo real:** Uma aplicação Flask serve um dashboard HTML que consulta dados do PostgreSQL (totais, médias, contagens) e do Firestore/Cosmos DB (eventos em tempo real). O JavaScript do frontend faz polling a cada 30 segundos para dados SQL e usa WebSocket ou Firestore real-time listeners para dados ao vivo.

**Exemplo 3 — Aplicação multi-ambiente (staging → production):** A equipe desenvolve localmente com SQLite, faz deploy no slot de staging (Azure) / versão de canary (GCP) com banco de testes, e após validação, faz swap/promote para produção com banco de produção. As variáveis de ambiente (`DATABASE_URL`, `STORAGE_*`) são diferentes em cada ambiente, configuradas via Application Settings (Azure) ou `app.yaml` com versões diferentes (GCP).

---

## 6. Boas Práticas para PaaS

### 6.1 Configuração e Deploy

| Prática | Descrição |
|---|---|
| **12-Factor App** | Siga os princípios de apps cloud-native: config via env vars, stateless, logs para stdout |
| **Health checks** | Sempre exponha um endpoint `/health` para que o load balancer monitore a saúde da aplicação |
| **Startup commands** | Configure explicitamente o comando de inicialização (ex.: `gunicorn app:app`) |
| **Requirements lock** | Use versões fixas em `requirements.txt` para builds reproduzíveis |
| **Gitignore** | Nunca commite `.env`, `__pycache__`, arquivos de credenciais |

### 6.2 Performance e Custos

| Prática | Descrição |
|---|---|
| **Scale to zero** | Use App Engine Standard (GCP) ou Azure Functions para workloads intermitentes |
| **Connection pooling** | Use pools de conexão (SQLAlchemy pool, PgBouncer) para evitar esgotamento de conexões |
| **Arquivos estáticos** | Sirva via Storage + CDN, não pelo app server |
| **Cold start** | Minimize dependências e tempo de boot para reduzir latência de cold starts |
| **Alertas de custo** | Configure budget alerts em ambas as plataformas para evitar surpresas |

### 6.3 Segurança

| Prática | Descrição |
|---|---|
| **HTTPS always** | Redirecione HTTP → HTTPS (ambas as plataformas suportam nativamente) |
| **Secrets management** | Use Key Vault (Azure) / Secret Manager (GCP), nunca env vars com senhas visíveis |
| **CORS** | Configure CORS restritivo (apenas domínios autorizados) |
| **Headers de segurança** | Adicione `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security` |

---

## 7. Cenários de Integração com Aulas Futuras

### Cenário 1 — PaaS → Container (evolução natural)

```mermaid
graph LR
    V1["📦 v1: App Service /<br/>App Engine<br/>(PaaS simples)"] -->|"App cresce,<br/>precisa de mais<br/>controle"| V2["🐳 v2: Container Apps /<br/>Cloud Run<br/>(Container serverless)"]
    V2 -->|"Múltiplos serviços,<br/>orquestração"| V3["☸️ v3: AKS / GKE<br/>(Kubernetes)"]
```

> **Aula 06:** Containerização e Orquestração na Nuvem — quando migrar de PaaS puro para containers.

### Cenário 2 — Deploy Automatizado

```mermaid
graph LR
    GIT["📝 git push"] -->|"Webhook"| CI["🔨 CI/CD<br/>GitHub Actions /<br/>Azure DevOps /<br/>Cloud Build"]
    CI -->|"Build + Test"| BUILD["✅ Artefato"]
    BUILD -->|"Deploy staging"| STAGING["🔶 Staging"]
    STAGING -->|"Aprovação"| PROD["🟢 Produção"]
```

> **Aula 08:** CI/CD na Nuvem — automação completa do pipeline de deploy.

### Cenário 3 — PaaS + Serverless (arquitetura complementar)

```mermaid
graph TB
    USER["👤 Usuário"] -->|"HTTPS"| APP["🖥️ App Service /<br/>App Engine<br/>(interface web)"]
    APP -->|"Trigger"| FUNC["⚡ Azure Functions /<br/>Cloud Functions<br/>(processamento pesado)"]
    FUNC -->|"Resultado"| QUEUE["📨 Service Bus /<br/>Pub/Sub"]
    QUEUE -->|"Notificação"| APP
```

> **Aulas 14-15:** Serverless e Filas de Mensagens.

---

## 8. Resumo Comparativo Final

| Aspecto | Azure App Service | Google App Engine |
|---|---|---|
| **Tipo de serviço** | PaaS gerenciado | PaaS gerenciado |
| **Ambientes** | Windows / Linux | Standard (sandbox) / Flexible (Docker) |
| **Scale to zero** | ❌ | ✅ (Standard) |
| **Deployment slots** | Slots nomeados + swap | Versões + traffic splitting |
| **Config file** | Portal / CLI | `app.yaml` |
| **URL padrão** | `<app>.azurewebsites.net` | `<projeto>.appspot.com` |
| **Free tier** | F1 (limitado, sem SLA) | Standard F1 (28 hrs/dia) |
| **Conexão ao banco** | Connection string + TLS / Private Endpoint | Cloud SQL Auth Proxy / Unix socket |
| **Logs** | Log Stream + Application Insights | Cloud Logging + Error Reporting |
| **Custom containers** | ✅ (qualquer tier Linux) | ✅ (Flexible environment) |
| **Melhor para** | Ecossistema Microsoft, .NET, integração AD | Protótipos rápidos, microserviços, scale-to-zero |

---

## 9. Exercícios Propostos

1. **Exercício de Deploy Duplo:** Crie uma aplicação Flask mínima (3 rotas: `/`, `/health`, `/info`) e faça deploy em ambas as plataformas. A rota `/info` deve retornar em qual plataforma a aplicação está rodando (use variáveis de ambiente para detectar). Documente ambas as URLs públicas resultantes.

2. **Exercício de Variáveis de Ambiente:** Configure a mesma aplicação para conectar-se ao banco de dados provisionado na Aula 04. No Azure, use Application Settings; no GCP, use `app.yaml` + Secret Manager. Crie uma rota `/db-test` que executa um `SELECT 1` e retorna o resultado.

3. **Exercício de Traffic Splitting:** No App Engine, faça deploy de duas versões da aplicação (v1 retorna `{"version": 1}`, v2 retorna `{"version": 2}`). Configure traffic splitting 50/50 e faça 20 requests via `curl` para verificar a distribuição. No Azure, crie um deployment slot "staging" e faça swap.

4. **Exercício de Integração Completa:** Usando a camada de dados do T1 (Aula 04), crie uma API CRUD simples (criar, listar, editar, deletar itens) hospedada em PaaS. Deploy em ambas as plataformas com URLs funcionais. Este exercício é uma base para o projeto interdisciplinar.

---

## 10. Referências

**Azure:**
- [App Service — Documentação](https://learn.microsoft.com/azure/app-service/)
- [Deploy Python app to App Service](https://learn.microsoft.com/azure/app-service/quickstart-python)
- [Deployment Slots](https://learn.microsoft.com/azure/app-service/deploy-staging-slots)
- [App Service + Azure SQL](https://learn.microsoft.com/azure/app-service/tutorial-python-postgresql-app)

**GCP:**
- [App Engine — Documentação](https://cloud.google.com/appengine/docs)
- [Deploy Python app to App Engine](https://cloud.google.com/appengine/docs/standard/python3/building-app)
- [Traffic Splitting](https://cloud.google.com/appengine/docs/admin-api/migrating-splitting-traffic)
- [App Engine + Cloud SQL](https://cloud.google.com/sql/docs/postgres/connect-app-engine-standard)

**Conceitos:**
- [The Twelve-Factor App](https://12factor.net/) — Metodologia para apps cloud-native
- [Flask Documentation](https://flask.palletsprojects.com/) — Framework web Python

---

> **Aula Anterior:** [Aula 04 — Bancos de Dados Gerenciados](./Aula_04-Bancos_de_Dados_Gerenciados.md)  
> **Próxima Aula:** [Aula 06 — Containerização e Orquestração na Nuvem](./Aula_06-Containerizacao_e_Orquestracao_na_Nuvem.md)
