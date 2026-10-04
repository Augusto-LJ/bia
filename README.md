# 🚀 Projeto BIA — Formação AWS

Projeto educacional criado por [Henrylle Maia](https://github.com/henrylle/bia) para demonstrar, na prática, como construir e evoluir uma infraestrutura cloud na AWS — partindo de fundamentos até uma arquitetura com alta disponibilidade, load balancer e pipeline de CI/CD totalmente automatizado.

> **Stack:** Node.js · React · PostgreSQL · Docker · AWS ECS · ECR · RDS · ALB · CodePipeline · CodeBuild · SSM

---

## ☁️ Arquitetura AWS

Toda a infraestrutura foi provisionada na região **`us-east-1` (N. Virginia)** usando **Amazon ECS com instâncias EC2** (sem Fargate).

```
Internet
    │
    ▼
Application Load Balancer — bia-alb
(us-east-1a  +  us-east-1b)
    │
    ├──────────────────────────┐
    ▼                          ▼
EC2 t3.micro               EC2 t3.micro
(us-east-1a)               (us-east-1b)
ECS cluster-bia-alb        ECS cluster-bia-alb
    │                          │
    └──────────┬───────────────┘
               ▼
        RDS PostgreSQL
        (db.t3.micro)
```

O tráfego HTTP (porta 80) é automaticamente redirecionado para HTTPS (porta 443) pelo ALB. A comunicação entre o ALB e as instâncias ECS usa **portas aleatórias** — por isso o modo de rede é `bridge` e o Security Group do EC2 libera `All TCP` vindo do ALB.

---

## 🔧 Recursos Provisionados

| Recurso | Nome / Identificador | Configuração |
|---|---|---|
| ECS Cluster | `cluster-bia-alb` | 2 container instances, 2 tasks rodando |
| Task Definition | `task-def-bia-alb:7` | bridge · 1 vCPU · 410 MB memória reservada |
| ECS Service | `service-bia-alb` | Rolling Update · desired count: 2 |
| EC2 (zona A) | `ECS Instance - cluster-bia-alb` | `t3.micro` — us-east-1a |
| EC2 (zona B) | `ECS Instance - cluster-bia-alb` | `t3.micro` — us-east-1b |
| Application Load Balancer | `bia-alb` | internet-facing · HTTP→HTTPS redirect · HTTPS com TLS 1.3 |
| Target Group | `tg-bia` | tipo: instance · health check: `/` · deregistration delay: 30s |
| Certificado SSL | AWS ACM | `arn:aws:acm:us-east-1:<account-id>:certificate/...` _(não divulgado)_ |
| RDS PostgreSQL | `bia` | `db.t3.micro` · PostgreSQL 18 · 20 GB · sem Multi-AZ |
| ECR | `bia` | 11 imagens · tags: latest + hash do commit |
| EC2 dev | `bia-dev` | `t3.micro` — us-east-1a · acesso somente via SSM |

> 📸 **[Coloque aqui um print de: Amazon ECS → Clusters → cluster-bia-alb → Services → service-bia-alb, mostrando o running count igual a 2 e o status ACTIVE]**

> 📸 **[Coloque aqui um print de: Amazon ECS → Clusters → cluster-bia-alb → Tasks, mostrando as 2 tasks com status RUNNING, cada uma em uma zona de disponibilidade diferente]**

---

## 🔀 Application Load Balancer

| Configuração | Valor |
|---|---|
| Nome | `bia-alb` |
| DNS | _(não divulgado — dado sensível)_ |
| Tipo | Application Load Balancer (internet-facing) |
| Zonas | us-east-1a + us-east-1b |
| Listener HTTP (80) | Redirect 301 → HTTPS |
| Listener HTTPS (443) | Forward → `tg-bia` |
| Política SSL | ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09 |
| Certificado | AWS Certificate Manager (ACM) |

Ambas as instâncias EC2 estão registradas no Target Group `tg-bia` com status **healthy**:

| Instância | Zona | Porta dinâmica | Status |
|---|---|---|---|
| _(não divulgado)_ | us-east-1a | _(porta aleatória)_ | ✅ healthy |
| _(não divulgado)_ | us-east-1b | _(porta aleatória)_ | ✅ healthy |

> 📸 **[Coloque aqui um print de: EC2 → Load Balancers → bia-alb, mostrando o estado Active, o DNS name e as duas zonas de disponibilidade configuradas]**

> 📸 **[Coloque aqui um print de: EC2 → Target Groups → tg-bia → Targets, mostrando as duas instâncias com status healthy e suas portas dinâmicas]**

---

## 🔒 Security Groups

Todos os Security Groups referenciam **outros Security Groups como source** — nunca IPs hardcodados entre serviços internos.

### `bia-alb` — Application Load Balancer
| Porta | Protocolo | Source | Descrição |
|---|---|---|---|
| 80 | TCP | `0.0.0.0/0` | acesso público HTTP |
| 443 | TCP | `0.0.0.0/0` | acesso público HTTPS |

### `bia-ec2` — Instâncias ECS
| Porta | Protocolo | Source | Descrição |
|---|---|---|---|
| 0–65535 | All TCP | `bia-alb` | acesso do bia-alb |

> Portas aleatórias do ECS no modo `bridge` exigem que o `bia-ec2` libere All TCP vindo do ALB.

### `bia-db` — RDS PostgreSQL
| Porta | Protocolo | Source | Descrição |
|---|---|---|---|
| 5432 | TCP | `bia-web` | acesso do bia-web |
| 5432 | TCP | `bia-ec2` | acesso do bia-ec2 |
| 5432 | TCP | `bia-dev` | acesso vindo de bia-dev |

### `bia-dev` — EC2 de Desenvolvimento
| Porta | Protocolo | Source | Descrição |
|---|---|---|---|
| 3001 | TCP | `0.0.0.0/0` | acesso público para testes locais |

> 📸 **[Coloque aqui um print de: EC2 → Security Groups, mostrando os grupos `bia-alb`, `bia-ec2` e `bia-db` com suas inbound rules]**

---

## ⚙️ Task Definition — `task-def-bia-alb`

| Parâmetro | Valor |
|---|---|
| Família | `task-def-bia-alb` |
| Revisão atual | `:7` |
| Network mode | `bridge` |
| Container | `bia` |
| Imagem atual | `<account-id>.dkr.ecr.us-east-1.amazonaws.com/bia:a87a00a` _(account-id não divulgado)_ |
| CPU | 1024 units (1 vCPU) |
| Memory reservation (soft limit) | 410 MB |
| Porta do container | 8080 |
| Porta do host | aleatória (0) |

**Estratégia de deploy (Rolling Update):**

| Parâmetro | Valor |
|---|---|
| Minimum healthy percent | 50% |
| Maximum percent | 100% |
| AZ Rebalancing | Desativado |
| Placement strategy | spread por AZ + spread por instância |

> 📸 **[Coloque aqui um print de: Amazon ECS → Task Definitions → task-def-bia-alb, mostrando a revisão atual, o network mode bridge e as configurações de CPU/memória do container]**

---

## 🗄️ Banco de Dados — Amazon RDS

| Configuração | Valor |
|---|---|
| Identificador | `bia` |
| Engine | PostgreSQL 18 |
| Tipo | `db.t3.micro` |
| Storage | 20 GB (gp2) |
| Endpoint | _(não divulgado — dado sensível)_ |
| Acesso público | ❌ Desabilitado |
| Multi-AZ | ❌ Não (configuração educacional) |
| Security Group | `bia-db` |

A conexão é feita via variáveis de ambiente (`DB_HOST`, `DB_USER`, `DB_PWD`, `DB_PORT`). O código já inclui suporte a **AWS Secrets Manager** — basta definir `DB_SECRET_NAME` e `DB_REGION` para ativá-lo, sem necessidade de alterar o código.

> 📸 **[Coloque aqui um print de: Amazon RDS → Databases → bia, mostrando o status Available, o endpoint, o tipo da instância e o Security Group `bia-db` associado]**

---

## 📦 Container e Amazon ECR

A imagem é construída em **single-stage** a partir da imagem oficial do Node.js hospedada no ECR Público:

```
public.ecr.aws/docker/library/node:24.18.0-slim
```

O build inclui o frontend React (Vite) compilado e servido diretamente pelo Express — gerando **um único artefato deployável**. Cada build gera duas tags:

- `latest` — aponta sempre para o build mais recente
- `<commit-hash>` — 7 caracteres do hash do commit para rastreabilidade (ex: `a87a00a`)

**Repositório ECR:** `<account-id>.dkr.ecr.us-east-1.amazonaws.com/bia` _(account-id não divulgado)_  
**Total de imagens armazenadas:** 11

> 📸 **[Coloque aqui um print de: Amazon ECR → Repositories → bia → Images, mostrando a lista de imagens com as tags latest e os hashes de commit, tamanhos (~207 MB) e datas de push]**

---

## 🔄 Pipeline de CI/CD

Deploy totalmente automatizado com **AWS CodePipeline** + **AWS CodeBuild**. A cada push na branch principal, o pipeline é acionado via webhook e executa o ciclo completo sem intervenção manual.

```
GitHub (push na branch principal)
    │
    ▼
CodePipeline
    ├── Stage 1: Source ──► Checkout via webhook
    ├── Stage 2: Build  ──► CodeBuild executa buildspec.yml
    │                           ├── Login no ECR
    │                           ├── docker build
    │                           ├── docker push :latest + :<commit-hash>
    │                           └── Gera imagedefinitions.json
    └── Stage 3: Deploy ──► ECS Rolling Update com a nova imagem
```

O artefato `imagedefinitions.json` gerado pelo CodeBuild instrui o ECS sobre qual imagem usar no deploy:

```json
[{"name":"bia","imageUri":"<account-id>.dkr.ecr.us-east-1.amazonaws.com/bia:<commit-hash>"}]
```

**Variáveis de ambiente configuradas no CodeBuild:**

| Variável | Valor |
|---|---|
| `ECR_REGISTRY` | `<account-id>.dkr.ecr.us-east-1.amazonaws.com` _(não divulgado)_ |
| `ECR_REPO` | `bia` |

Além do pipeline automatizado, o projeto inclui o script `deploy-com-rollback.sh` para operações manuais: deploy, listagem de revisões da task definition e rollback para qualquer revisão anterior.

> 📸 **[Coloque aqui um print de: AWS CodePipeline → Pipelines, mostrando o pipeline com as 3 etapas (Source, Build, Deploy) com status Succeeded na última execução]**

> 📸 **[Coloque aqui um print de: AWS CodeBuild → Build projects → bia → Build history, mostrando o histórico de builds com status Succeeded, duração e datas]**

---

## 🖥️ EC2 de Desenvolvimento — `bia-dev`

Instância dedicada para builds manuais, execução de scripts de infraestrutura e acesso administrativo ao banco. O acesso é feito **exclusivamente via AWS Systems Manager (SSM)**, sem SSH e sem porta 22 aberta.

| Configuração | Valor |
|---|---|
| Instance ID | _(não divulgado — dado sensível)_ |
| Tipo | `t3.micro` |
| AMI | Amazon Linux 2023 |
| Zona | `us-east-1a` |
| IP Público | _(não divulgado — dado sensível)_ |
| Security Group | `bia-dev` |
| Acesso | SSM Session Manager — sem SSH |
| IAM Instance Profile | `role-acesso-ssm` (policy: `AmazonSSMManagedInstanceCore`) |

**Ferramentas instaladas via User Data:**
Docker · Docker Compose v2.23.3 · AWS CLI v2 · Node.js 24.x · Git · jq · Python 3.11 · uv · Swap 4 GB

```bash
# Iniciar sessão na instância
aws ssm start-session --target <instance-id> --region us-east-1
```

> 📸 **[Coloque aqui um print de: AWS Systems Manager → Fleet Manager → Managed nodes, mostrando a instância `bia-dev` com status Online, plataforma Amazon Linux 2023 e a IAM role `role-acesso-ssm` associada]**

---

## 🔐 IAM e Segurança

- **`role-acesso-ssm`** — IAM Role com a policy `AmazonSSMManagedInstanceCore`, associada à `bia-dev`. Permite acesso via SSM sem key pair nem porta 22 aberta, eliminando gerenciamento de credenciais SSH
- **Security Groups** referenciam apenas outros Security Groups como source — nunca IPs hardcodados entre serviços internos
- Banco de dados **sem acesso público** — acessível somente de dentro da VPC pelos Security Groups autorizados
- Acesso à instância de desenvolvimento **sem SSH** — somente via SSM
- ALB com **HTTPS obrigatório** e política TLS 1.3 (`ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09`)

> 📸 **[Coloque aqui um print de: IAM → Roles → role-acesso-ssm → Permissions, mostrando a policy `AmazonSSMManagedInstanceCore` anexada]**

---

## 🏃 Rodando Localmente

**Pré-requisitos:** Docker e Docker Compose.

```bash
git clone https://github.com/henrylle/bia.git
cd bia

# Subir app + banco local + Redis local
docker compose up -d

# Rodar as migrations
docker compose exec server bash -c 'npx sequelize db:migrate'

# Testar
curl http://localhost:3001/api/versao
```

**Variáveis de ambiente relevantes:**

| Variável | Descrição | Padrão local |
|---|---|---|
| `DB_HOST` | Host do PostgreSQL | `database` |
| `DB_USER` | Usuário do banco | `postgres` |
| `DB_PWD` | Senha do banco | `postgres` |
| `DB_PORT` | Porta do banco | `5432` |
| `CACHE_ENDPOINT` | Host do Redis | _(desativado se vazio)_ |
| `CACHE_TTL` | TTL do cache em segundos | `60` |
| `DB_SECRET_NAME` | Nome do secret no Secrets Manager | _(opcional)_ |
| `DB_REGION` | Região AWS para Secrets Manager | _(opcional)_ |

---

## 📁 Estrutura do Projeto

```
bia/
├── api/
│   ├── controllers/        # Lógica de negócio (tarefas, versão, cache-config)
│   ├── models/             # Models Sequelize
│   └── routes/             # Definição das rotas Express
├── client/                 # Frontend React 17 + Vite
├── config/
│   ├── database.js         # Sequelize (com suporte a Secrets Manager)
│   └── express.js          # Configuração do Express
├── database/migrations/    # Migrations Sequelize
├── docs/architecture/      # Diagramas HTML da arquitetura AWS
├── lib/cache.js            # Módulo de cache Redis (ioredis) com fallback
├── scripts/
│   ├── ecs/                # Scripts de build e deploy ECS
│   └── user_data_ec2_zona_a.sh  # User Data da instância bia-dev
├── Dockerfile              # Imagem single-stage (Node 24 + React build embutido)
├── buildspec.yml           # Spec do AWS CodeBuild (build + push ECR)
├── compose.yml             # Docker Compose para desenvolvimento local
├── deploy-com-rollback.sh  # Deploy manual com suporte a rollback por revisão
└── server.js               # Entrypoint — Express na porta 8080
```

---

## 📚 Documentação de Arquitetura

- [Diagrama ECS + EC2](./docs/architecture/aws-ecs-diagram.html)
- [Diagrama detalhado](./docs/architecture/bia-diagrama-linkedin.html)

---

*Projeto educacional — Formação AWS com Henrylle Maia*
