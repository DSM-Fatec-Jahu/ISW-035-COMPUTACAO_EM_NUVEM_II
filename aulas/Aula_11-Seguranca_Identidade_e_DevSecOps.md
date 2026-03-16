# Aula 11 — Segurança, Identidade e DevSecOps

> **Disciplina:** Computação em Nuvem II (ISW035)  
> **Professor:** Ronan Adriel Zenatti — FATEC Jahu / Centro Paula Souza  
> **Semestre:** 1º/2026  
> **Carga Horária:** 4h práticas

---

## 1. Visão Geral e Contextualização

Segurança em nuvem não é um recurso que se "habilita" — é uma disciplina transversal que permeia todas as decisões, desde a criação de uma conta de armazenamento até o pipeline de CI/CD. Nas aulas anteriores, já abordamos aspectos de segurança pontualmente: criptografia no storage (Aula 02), firewall de banco de dados (Aula 04), secrets no pipeline (Aula 08) e scanning de vulnerabilidades (Aula 08). Nesta aula, consolidamos tudo em um framework coerente, abordando os três eixos: **Identidade e Acesso (IAM)**, **Gestão de Segredos** e **DevSecOps integrado**.

### Modelo de Responsabilidade Compartilhada

```mermaid
graph TB
    subgraph Voce["🔐 Sua Responsabilidade"]
        V1["Dados e conteúdo"]
        V2["Identidade e acesso (IAM)"]
        V3["Configuração de aplicações"]
        V4["Configuração de rede (firewall, NSG)"]
        V5["Criptografia do lado do cliente"]
        V6["Segurança no código (DevSecOps)"]
    end

    subgraph Provedor["☁️ Responsabilidade do Provedor"]
        P1["Segurança física do datacenter"]
        P2["Hardware e rede global"]
        P3["Hypervisor e isolamento"]
        P4["Patches do SO (em PaaS/Serverless)"]
        P5["Criptografia em repouso (padrão)"]
        P6["Compliance (SOC, ISO, PCI)"]
    end

    style Voce fill:#e63946,color:#fff
    style Provedor fill:#2a9d8f,color:#fff
```

> **Ponto crítico:** A maioria dos incidentes de segurança em nuvem não é causada por falhas do provedor, mas por **configurações incorretas** feitas pelos clientes — permissões excessivas, secrets expostos, buckets públicos e firewalls abertos.

### Mapa de Equivalência — Segurança

| Conceito | Azure | GCP |
|---|---|---|
| Plataforma de identidade | Microsoft Entra ID (Azure AD) | Cloud IAM + Google Workspace |
| Roles pré-definidos | Azure RBAC Built-in Roles | IAM Predefined Roles |
| Roles customizados | Custom Roles (RBAC) | Custom Roles (IAM) |
| Identidade para aplicações | Managed Identity | Service Account |
| Gestão de segredos | Azure Key Vault | Secret Manager |
| Gestão de chaves de criptografia | Azure Key Vault (Keys) | Cloud KMS |
| Gestão de certificados | Azure Key Vault (Certificates) | Certificate Authority Service |
| Postura de segurança | Microsoft Defender for Cloud | Security Command Center |
| Conformidade | Azure Policy + Compliance Manager | Organization Policy + Security Health Analytics |
| Auditoria | Azure Activity Log + Audit Logs | Cloud Audit Logs |
| Proteção de rede | NSG + Azure Firewall | VPC Firewall Rules + Cloud Armor |

---

## 2. Identidade e Controle de Acesso (IAM)

### 2.1 Princípio do Menor Privilégio

O princípio mais importante da segurança em nuvem: **conceda apenas as permissões mínimas necessárias** para que uma pessoa, serviço ou aplicação realize sua tarefa. Nada mais.

```mermaid
graph LR
    subgraph Errado["❌ Anti-padrão"]
        DEV_BAD["Desenvolvedor"] -->|"Contributor<br/>(acesso total<br/>ao resource group)"| RG_BAD["Resource Group<br/>Produção"]
    end

    subgraph Correto["✅ Menor Privilégio"]
        DEV_GOOD["Desenvolvedor"] -->|"Reader<br/>(apenas leitura)"| RG_PROD["RG Produção"]
        DEV_GOOD -->|"Contributor<br/>(acesso total)"| RG_DEV["RG Desenvolvimento"]
        APP["Aplicação<br/>(Managed Identity)"] -->|"Storage Blob<br/>Data Contributor<br/>(apenas storage)"| STOR["Storage Account"]
    end

    style Errado fill:#e63946,color:#fff
    style Correto fill:#2a9d8f,color:#fff
```

### 2.2 Azure RBAC (Role-Based Access Control)

O Azure usa um modelo de RBAC hierárquico: permissões atribuídas em um escopo superior (Management Group → Subscription → Resource Group → Resource) são herdadas nos escopos inferiores.

**Roles mais comuns:**

| Role | Permissão | Quando Usar |
|---|---|---|
| **Owner** | Tudo + gerenciar permissões | Administradores da subscription (restrito) |
| **Contributor** | Criar/editar/deletar recursos, sem gerenciar permissões | Desenvolvedores em seus resource groups |
| **Reader** | Apenas visualizar | Auditores, stakeholders, ambientes de produção (para devs) |
| **Storage Blob Data Contributor** | Ler/escrever blobs | Aplicações que precisam acessar storage |
| **SQL DB Contributor** | Gerenciar bancos SQL | Administradores de banco |
| **AcrPush** | Push de imagens para ACR | Pipelines CI/CD |

```bash
# Atribuir role a um usuário
az role assignment create \
    --assignee user@fatecjahu.edu.br \
    --role "Reader" \
    --scope "/subscriptions/SUB_ID/resourceGroups/rg-cnuvem2"

# Listar atribuições em um resource group
az role assignment list \
    --resource-group rg-cnuvem2 \
    --output table
```

### 2.3 GCP IAM

O GCP IAM segue um modelo similar: `member` (quem) recebe um `role` (conjunto de permissões) em um `resource` (escopo). Os roles são compostos por `permissions` granulares.

**Tipos de membros no GCP:**

| Tipo | Formato | Exemplo |
|---|---|---|
| **Conta Google** | `user:email` | `user:aluno@gmail.com` |
| **Service Account** | `serviceAccount:email` | `serviceAccount:app-sa@project.iam.gserviceaccount.com` |
| **Grupo** | `group:email` | `group:devs@fatecjahu.edu.br` |
| **Domínio** | `domain:domain` | `domain:fatecjahu.edu.br` |

**Roles mais comuns:**

| Role | Permissão | Quando Usar |
|---|---|---|
| **roles/viewer** | Apenas visualizar | Auditores, stakeholders |
| **roles/editor** | Criar/editar/deletar, sem gerenciar IAM | Desenvolvedores |
| **roles/owner** | Tudo | Administradores do projeto (restrito) |
| **roles/storage.objectUser** | Ler/escrever objetos em buckets | Aplicações com acesso a storage |
| **roles/cloudsql.client** | Conectar a instâncias Cloud SQL | Aplicações que acessam banco |
| **roles/run.invoker** | Invocar serviços Cloud Run | APIs que chamam outros serviços |

```bash
# Atribuir role a um membro
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="user:aluno@gmail.com" \
    --role="roles/viewer"

# Listar bindings do projeto
gcloud projects get-iam-policy PROJECT_ID --format=yaml
```

### 2.4 Identidade para Aplicações — Sem Senhas!

O cenário ideal é que **nenhuma aplicação use senhas ou chaves de API** para se autenticar com serviços cloud. Em vez disso, usam-se identidades gerenciadas pelo provedor.

| Conceito | Azure | GCP |
|---|---|---|
| **Nome** | Managed Identity | Service Account |
| **Como funciona** | A plataforma injeta automaticamente um token na aplicação, sem credenciais no código | A plataforma fornece credenciais via metadata server, sem arquivo de chave |
| **Tipos** | System-assigned (vive com o recurso) + User-assigned (independente) | Default SA (automática) + Custom SA (criada pelo usuário) |
| **Uso em App Service** | `az webapp identity assign` → app obtém token via `DefaultAzureCredential` | Service Account vinculada ao serviço Cloud Run/App Engine |
| **Uso em Container Apps** | Managed Identity atribuída ao Container App | Workload Identity (GKE) ou SA associada (Cloud Run) |

```bash
# Azure: Atribuir Managed Identity a um App Service
az webapp identity assign \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026

# Conceder acesso ao Storage via Managed Identity (sem senha!)
az role assignment create \
    --assignee $(az webapp identity show --resource-group rg-cnuvem2 --name app-cnuvem2-2026 --query principalId -o tsv) \
    --role "Storage Blob Data Contributor" \
    --scope "/subscriptions/.../resourceGroups/rg-cnuvem2/providers/Microsoft.Storage/storageAccounts/stcnuvem2"
```

```bash
# GCP: Criar Service Account dedicada
gcloud iam service-accounts create app-sa \
    --display-name="SA da Aplicação CNII"

# Conceder acesso ao Storage e Cloud SQL
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/storage.objectUser"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"

# Vincular ao Cloud Run
gcloud run services update cnuvem2-app \
    --service-account=app-sa@PROJECT_ID.iam.gserviceaccount.com \
    --region=southamerica-east1
```

**No código Python — autenticação transparente (sem secrets):**

```python
# Azure: DefaultAzureCredential detecta Managed Identity automaticamente
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

credential = DefaultAzureCredential()  # Usa Managed Identity em cloud, Azure CLI local
blob_service = BlobServiceClient(
    account_url="https://stcnuvem2.blob.core.windows.net",
    credential=credential
)
# Sem connection string, sem chave de conta!

# GCP: google-auth detecta Service Account automaticamente
from google.cloud import storage

client = storage.Client()  # Usa SA vinculada ao Cloud Run em cloud, gcloud auth local
bucket = client.bucket("cnuvem2-arquivos")
# Sem arquivo JSON de chave, sem variável de ambiente!
```

---

## 3. Gestão de Segredos

### 3.1 Por que Vault de Segredos?

Variáveis de ambiente são melhores que senhas hardcoded no código, mas ainda têm problemas: são visíveis a qualquer pessoa com acesso ao portal/console, não têm rotação automática, não registram quem acessou e não são criptografadas em memória.

Vaults de segredos resolvem tudo isso: armazenam secrets criptografados, controlam acesso via IAM, registram auditoria de acesso e suportam rotação automática.

### 3.2 Azure Key Vault

```bash
# Criar Key Vault
az keyvault create \
    --resource-group rg-cnuvem2 \
    --name kv-cnuvem2-2026 \
    --location brazilsouth \
    --enable-soft-delete true \
    --retention-days 7

# Armazenar secret
az keyvault secret set \
    --vault-name kv-cnuvem2-2026 \
    --name "db-connection-string" \
    --value "postgresql://user:pass@host:5432/db?sslmode=require"

# Conceder acesso à Managed Identity da aplicação
az keyvault set-policy \
    --name kv-cnuvem2-2026 \
    --object-id $(az webapp identity show --resource-group rg-cnuvem2 --name app-cnuvem2-2026 --query principalId -o tsv) \
    --secret-permissions get list

# Na aplicação: referenciar secret (App Service)
az webapp config appsettings set \
    --resource-group rg-cnuvem2 \
    --name app-cnuvem2-2026 \
    --settings DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://kv-cnuvem2-2026.vault.azure.net/secrets/db-connection-string)"
```

### 3.3 GCP Secret Manager

```bash
# Criar secret
echo -n "postgresql://user:pass@host:5432/db" | \
    gcloud secrets create db-connection-string \
    --data-file=- \
    --replication-policy="automatic"

# Conceder acesso à Service Account
gcloud secrets add-iam-policy-binding db-connection-string \
    --member="serviceAccount:app-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# No Cloud Run: montar secret como variável de ambiente
gcloud run services update cnuvem2-app \
    --set-secrets="DATABASE_URL=db-connection-string:latest" \
    --region=southamerica-east1
```

**Lendo secret via Python (para qualquer ambiente):**

```python
# Azure
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

client = SecretClient(
    vault_url="https://kv-cnuvem2-2026.vault.azure.net",
    credential=DefaultAzureCredential()
)
db_url = client.get_secret("db-connection-string").value

# GCP
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
response = client.access_secret_version(
    name="projects/PROJECT_ID/secrets/db-connection-string/versions/latest"
)
db_url = response.payload.data.decode("UTF-8")
```

### 3.4 Comparativo — Gestão de Segredos

| Aspecto | Azure Key Vault | GCP Secret Manager |
|---|---|---|
| **Tipos de segredos** | Secrets, Keys (criptografia), Certificates | Secrets |
| **Chaves de criptografia** | Key Vault Keys (HSM ou software) | Cloud KMS (serviço separado) |
| **Certificados** | Key Vault Certificates (auto-renovação) | Certificate Authority Service (separado) |
| **Versionamento** | Automático (por version ID) | Automático (por version number) |
| **Rotação automática** | Via Event Grid + Azure Function | Via Pub/Sub + Cloud Function |
| **Soft delete** | Sim (padrão, 7-90 dias) | Sim (etag-based) |
| **Referência direta no App Service** | `@Microsoft.KeyVault(...)` (sem código) | `--set-secrets` no Cloud Run (sem código) |
| **Free tier** | 10.000 operações/mês (Standard) | 6 versões ativas grátis + 10.000 operações/mês |
| **Preço por secret** | $0.03/10.000 operações | $0.06/10.000 operações |
| **Auditoria** | Key Vault Audit Logs | Cloud Audit Logs |

---

## 4. DevSecOps Integrado — Consolidação

Na Aula 08, introduzimos scanning de segurança no pipeline. Agora, integramos esse conceito com o framework completo de segurança.

### 4.1 Segurança em Cada Fase do Ciclo de Vida

```mermaid
graph LR
    PLAN["📋 Plan<br/><br/>Threat modeling<br/>Security requirements"] --> CODE["💻 Code<br/><br/>SAST, Linting<br/>Secret detection<br/>Dependency lock"]
    CODE --> BUILD["🔨 Build<br/><br/>SCA (dependências)<br/>Container scanning<br/>IaC scanning"] 
    BUILD --> TEST["🧪 Test<br/><br/>DAST (testes dinâmicos)<br/>Penetration testing<br/>Compliance checks"]
    TEST --> DEPLOY["🚀 Deploy<br/><br/>Managed Identity<br/>Secret Manager<br/>Network isolation"]
    DEPLOY --> OPERATE["📊 Operate<br/><br/>Monitoring + Alertas<br/>Audit logs<br/>Incident response"]
    OPERATE --> PLAN
```

### 4.2 Checklist de Segurança para o Projeto

| Camada | Verificação | Como |
|---|---|---|
| **Código** | Sem secrets hardcoded | Gitleaks no pipeline (Aula 08) |
| **Código** | Dependências sem CVEs críticas | pip-audit / npm audit no pipeline |
| **Container** | Imagem base sem vulnerabilidades | Trivy scan no pipeline |
| **Container** | Usuário non-root no Dockerfile | `USER appuser` no Dockerfile (Aula 06) |
| **Identidade** | Aplicação usa Managed Identity / SA | Configurar no App Service / Cloud Run |
| **Identidade** | Menor privilégio nos roles IAM | Roles específicos, não Owner/Editor |
| **Secrets** | Senhas no Vault, não em env vars | Key Vault / Secret Manager |
| **Rede** | Banco sem IP público em produção | Private Endpoint / Private IP |
| **Rede** | HTTPS obrigatório | TLS enforced no App Service / Cloud Run |
| **Dados** | Criptografia em repouso | Automática (padrão em ambos) |
| **Dados** | Soft delete habilitado | Storage + Banco |
| **Auditoria** | Logs de acesso habilitados | Activity Log / Cloud Audit Logs |
| **Monitoramento** | Alertas configurados | Azure Alerts / GCP Alerting (Aula 10) |

---

## 5. Exemplos Práticos de Segurança

**Exemplo 1 — Eliminar credenciais do código:** Uma aplicação Flask acessa PostgreSQL e Cloud Storage. Em vez de connection string com senha no `.env`, usa-se: Managed Identity (Azure) ou Service Account (GCP) para o storage (zero credenciais); e Key Vault (Azure) ou Secret Manager (GCP) para a URL do banco. O código chama `DefaultAzureCredential()` ou `secretmanager.Client()` — nenhuma senha toca o código.

**Exemplo 2 — Auditoria de acesso a dados sensíveis:** Uma aplicação de saúde armazena prontuários. Todo acesso ao Key Vault / Secret Manager é registrado em audit logs. Se alguém acessar a connection string fora do horário comercial, um alerta dispara automaticamente. O Cloud Audit Log (GCP) ou Key Vault Diagnostic Settings (Azure) registra quem, quando e de qual IP.

**Exemplo 3 — Resposta a incidente de secret vazado:** Um desenvolvedor acidentalmente commita uma service account key em um repositório público. O pipeline detecta via Gitleaks e bloqueia o push. Simultaneamente, o GitHub Secret Scanning notifica o GCP, que revoga a chave automaticamente. O time cria uma nova service account key, atualiza o Secret Manager e re-deploya a aplicação — tudo em menos de 30 minutos.

---

## 6. Cenários de Integração

### Cenário 1 — IAM + IaC (Aulas 07 + 11)

> Toda configuração de IAM é definida como código no Terraform: roles, atribuições, service accounts, políticas. Mudanças em permissões passam por code review via Pull Request, garantindo auditoria e aprovação.

### Cenário 2 — Secrets + CI/CD (Aulas 08 + 11)

> O pipeline CI/CD usa Service Principal (Azure) ou Service Account (GCP) com permissões mínimas. Secrets do pipeline (credenciais de deploy) ficam em GitHub Encrypted Secrets. Secrets da aplicação ficam em Key Vault / Secret Manager e são injetados no deploy.

### Cenário 3 — Segurança + Alta Disponibilidade (Aulas 11 + 13)

> A replicação geo-redundante do Key Vault / Secret Manager garante que, em caso de falha regional, os secrets ainda estejam acessíveis na região secundária. Os backups de banco usam Customer-Managed Keys (CMK) armazenadas no Key Vault / Cloud KMS.

---

## 7. Resumo Comparativo Final

| Aspecto | Azure | GCP |
|---|---|---|
| **IAM** | Entra ID + Azure RBAC | Cloud IAM |
| **Identidade de aplicação** | Managed Identity | Service Account + Workload Identity |
| **Vault de segredos** | Key Vault (secrets + keys + certs) | Secret Manager (secrets) + Cloud KMS (keys) |
| **Referência no runtime** | `@Microsoft.KeyVault(...)` (declarativo) | `--set-secrets` no Cloud Run (declarativo) |
| **Auditoria** | Activity Log + Key Vault Diagnostics | Cloud Audit Logs |
| **Postura de segurança** | Defender for Cloud + Secure Score | Security Command Center + Health Analytics |
| **Políticas de compliance** | Azure Policy | Organization Policy |
| **Detecção de ameaças** | Defender for Containers / SQL / Storage | Security Command Center Premium |
| **Network security** | NSG + Azure Firewall + Private Link | VPC Firewall + Cloud Armor + Private Service Connect |

---

## 8. Exercícios Propostos

1. **Exercício IAM:** Crie um role assignment (Azure) ou IAM binding (GCP) que conceda à sua aplicação acesso de leitura e escrita ao storage, sem usar chaves de conta. Documente os comandos e verifique que a aplicação consegue acessar o storage.

2. **Exercício Managed Identity / Service Account:** Configure sua aplicação (App Service, Cloud Run ou Container Apps) para usar Managed Identity (Azure) ou Service Account (GCP) em vez de connection strings. Remova as credenciais de storage das variáveis de ambiente.

3. **Exercício Key Vault / Secret Manager:** Mova a connection string do banco de dados para o Key Vault ou Secret Manager. Configure a aplicação para ler o secret do vault, em vez de variável de ambiente. Documente a diferença de segurança.

4. **Exercício de Auditoria:** Habilite audit logs no Key Vault ou Secret Manager. Acesse um secret e verifique que o acesso foi registrado no log. Capture um screenshot do registro de auditoria.

---

## 9. Referências

**Azure:**
- [Azure RBAC — Documentação](https://learn.microsoft.com/azure/role-based-access-control/)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/)
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/)

**GCP:**
- [Cloud IAM — Documentação](https://cloud.google.com/iam/docs)
- [Service Accounts](https://cloud.google.com/iam/docs/service-accounts)
- [Secret Manager](https://cloud.google.com/secret-manager/docs)
- [Security Command Center](https://cloud.google.com/security-command-center/docs)

**Conceitos:**
- [OWASP Cloud-Native Security](https://owasp.org/www-project-cloud-native-application-security-top-10/)
- [CIS Benchmarks for Cloud](https://www.cisecurity.org/benchmark)

---

> **Aula Anterior:** [Aula 10 — Monitoramento: Nativo vs Open Source](./Aula_10-Monitoramento_Nativo_vs_Open_Source.md)  
> **Próxima Aula:** [Aula 12 — Redes Virtuais e Conectividade](./Aula_12-Redes_Virtuais_e_Conectividade.md)
