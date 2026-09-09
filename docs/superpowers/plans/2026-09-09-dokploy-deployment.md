# Dokploy Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configurar o Postiz para execução em VPS com Dokploy utilizando a imagem oficial pré-construída, armazenamento Cloudflare R2, rede interna isolada com Traefik e documentação passo a passo de deploy.

**Architecture:** A stack conterá um arquivo `docker-compose.dokploy.yaml` integrando `postiz`, `postiz-postgres`, `postiz-redis` e a stack completa do `temporal` (Temporal Server, Elasticsearch, PostgreSQL do Temporal, UI e Admin Tools). A aplicação se conecta à rede `dokploy-network` para roteamento reverso e SSL automático via Traefik. Todas as variáveis de ambiente sensíveis e parâmetros de R2 serão abstraídos em `.env.dokploy.example`.

**Tech Stack:** Docker, Docker Compose, Dokploy, Traefik, PostgreSQL 17, Redis 7.2, Temporal 1.28, Elasticsearch 7.17, Cloudflare R2 (S3 API).

## Global Constraints

- Imagem oficial do Postiz: `ghcr.io/gitroomhq/postiz-app:latest`
- Armazenamento: Cloudflare R2 (S3-compatible) com `STORAGE_PROVIDER: 'cloudflare'`
- Porta interna da aplicação: `5000`
- Rede externa do Dokploy: `dokploy-network` (`external: true`)
- Nenhum segredo ou credencial real comitada no Git
- Todos os arquivos criados devem ser compatíveis com Git e Dokploy

---

### Task 1: Criar template `.env.dokploy.example` e atualizar `.gitignore`

**Files:**
- Create: `c:\www\postiz-app\.env.dokploy.example`
- Modify: `c:\www\postiz-app\.gitignore`

**Interfaces:**
- Produces: `.env.dokploy.example` contendo todas as variáveis necessárias para a stack Dokploy + Cloudflare R2.
- Consumes: Definições de variáveis do `.env.example` e do design spec.

- [ ] **Step 1: Atualizar `.gitignore` para ignorar `.env.dokploy` e arquivos locais**

Adicionar regras em `c:\www\postiz-app\.gitignore`:
```gitignore
# Dokploy environment files
.env.dokploy
.env.dokploy.local
```

- [ ] **Step 2: Criar o arquivo `.env.dokploy.example`**

Criar `c:\www\postiz-app\.env.dokploy.example` com o conteúdo:
```env
# =================================================================
# POSTIZ DOKPLOY CONFIGURATION TEMPLATE
# Preencha estas variáveis na aba "Environment" do seu Compose no Dokploy.
# =================================================================

# === Domínio e URLs Públicas ===
# Substitua pelo seu domínio/subdomínio apontado para a VPS
POSTIZ_DOMAIN=postiz.seudominio.com
MAIN_URL=https://postiz.seudominio.com
FRONTEND_URL=https://postiz.seudominio.com
NEXT_PUBLIC_BACKEND_URL=https://postiz.seudominio.com/api
BACKEND_INTERNAL_URL=http://localhost:3000

# === Segurança e Autenticação ===
# Gere uma chave aleatória de pelo menos 32 caracteres (ex: openssl rand -hex 32)
JWT_SECRET=substitua_por_uma_string_aleatoria_longa_e_segura_32_chars
IS_GENERAL=true
DISABLE_REGISTRATION=false

# === Banco de Dados Principal (PostgreSQL) ===
POSTGRES_USER=postiz_user
POSTGRES_PASSWORD=defina_uma_senha_forte_para_o_postgres
POSTGRES_DB=postiz_production
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postiz-postgres:5432/${POSTGRES_DB}

# === Cache e Filas (Redis) ===
REDIS_URL=redis://postiz-redis:6379

# === Temporal Workflow Engine ===
TEMPORAL_ADDRESS=temporal:7233
TEMPORAL_POSTGRES_USER=temporal
TEMPORAL_POSTGRES_PASSWORD=defina_uma_senha_forte_para_temporal_pg

# === Armazenamento de Mídia: Cloudflare R2 (S3 Compatible) ===
STORAGE_PROVIDER=cloudflare
CLOUDFLARE_ACCOUNT_ID=seu_account_id_cloudflare
CLOUDFLARE_ACCESS_KEY=sua_access_key_do_token_r2
CLOUDFLARE_SECRET_ACCESS_KEY=sua_secret_access_key_do_token_r2
CLOUDFLARE_BUCKETNAME=postiz-uploads
CLOUDFLARE_BUCKET_URL=https://pub-substitua-pelo-id.r2.dev
CLOUDFLARE_REGION=auto

# === Limites de API e Recursos ===
API_LIMIT=30
NX_ADD_PLUGINS=false
```

- [ ] **Step 3: Validar que `.env.dokploy` está protegido no Git**

Executar:
```powershell
git check-ignore -v .env.dokploy
```
Esperado: linha indicando correspondência em `.gitignore`.

- [ ] **Step 4: Commit das alterações do Task 1**

```powershell
git add .gitignore .env.dokploy.example; git commit -m "feat(dokploy): add env template and gitignore rules"
```

---

### Task 2: Criar o arquivo `docker-compose.dokploy.yaml` e validar a sintaxe

**Files:**
- Create: `c:\www\postiz-app\docker-compose.dokploy.yaml`

**Interfaces:**
- Produces: `docker-compose.dokploy.yaml` pronto para consumo pelo Dokploy.
- Consumes: Parâmetros definidos em `.env.dokploy.example` e imagem `ghcr.io/gitroomhq/postiz-app:latest`.

- [ ] **Step 1: Criar o arquivo `docker-compose.dokploy.yaml`**

Criar `c:\www\postiz-app\docker-compose.dokploy.yaml` contendo:
```yaml
services:
  postiz:
    image: ghcr.io/gitroomhq/postiz-app:latest
    container_name: postiz
    restart: unless-stopped
    environment:
      MAIN_URL: ${MAIN_URL}
      FRONTEND_URL: ${FRONTEND_URL}
      NEXT_PUBLIC_BACKEND_URL: ${NEXT_PUBLIC_BACKEND_URL}
      JWT_SECRET: ${JWT_SECRET}
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      BACKEND_INTERNAL_URL: ${BACKEND_INTERNAL_URL:-http://localhost:3000}
      TEMPORAL_ADDRESS: ${TEMPORAL_ADDRESS:-temporal:7233}
      IS_GENERAL: ${IS_GENERAL:-true}
      DISABLE_REGISTRATION: ${DISABLE_REGISTRATION:-false}
      STORAGE_PROVIDER: ${STORAGE_PROVIDER:-cloudflare}
      CLOUDFLARE_ACCOUNT_ID: ${CLOUDFLARE_ACCOUNT_ID}
      CLOUDFLARE_ACCESS_KEY: ${CLOUDFLARE_ACCESS_KEY}
      CLOUDFLARE_SECRET_ACCESS_KEY: ${CLOUDFLARE_SECRET_ACCESS_KEY}
      CLOUDFLARE_BUCKETNAME: ${CLOUDFLARE_BUCKETNAME}
      CLOUDFLARE_BUCKET_URL: ${CLOUDFLARE_BUCKET_URL}
      CLOUDFLARE_REGION: ${CLOUDFLARE_REGION:-auto}
      API_LIMIT: ${API_LIMIT:-30}
      NX_ADD_PLUGINS: "false"
    volumes:
      - postiz-config:/config/
      - postiz-uploads:/uploads/
    networks:
      - postiz-internal
      - dokploy-network
    depends_on:
      postiz-postgres:
        condition: service_healthy
      postiz-redis:
        condition: service_healthy

  postiz-postgres:
    image: postgres:17-alpine
    container_name: postiz-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postiz_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:-postiz_production}
    volumes:
      - postiz-postgres-data:/var/lib/postgresql/data
    networks:
      - postiz-internal
    healthcheck:
      test: pg_isready -U ${POSTGRES_USER:-postiz_user} -d ${POSTGRES_DB:-postiz_production}
      interval: 10s
      timeout: 3s
      retries: 5

  postiz-redis:
    image: redis:7.2
    container_name: postiz-redis
    restart: unless-stopped
    volumes:
      - postiz-redis-data:/data
    networks:
      - postiz-internal
    healthcheck:
      test: redis-cli ping
      interval: 10s
      timeout: 3s
      retries: 5

  temporal-elasticsearch:
    container_name: temporal-elasticsearch
    image: elasticsearch:7.17.27
    restart: unless-stopped
    environment:
      - cluster.routing.allocation.disk.threshold_enabled=true
      - cluster.routing.allocation.disk.watermark.low=512mb
      - cluster.routing.allocation.disk.watermark.high=256mb
      - cluster.routing.allocation.disk.watermark.flood_stage=128mb
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms256m -Xmx256m
      - xpack.security.enabled=false
    networks:
      - postiz-internal
    expose:
      - 9200
    volumes:
      - temporal-es-data:/usr/share/elasticsearch/data

  temporal-postgresql:
    container_name: temporal-postgresql
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${TEMPORAL_POSTGRES_USER:-temporal}
      POSTGRES_PASSWORD: ${TEMPORAL_POSTGRES_PASSWORD}
    networks:
      - postiz-internal
    expose:
      - 5432
    volumes:
      - temporal-postgres-data:/var/lib/postgresql/data

  temporal:
    container_name: temporal
    image: temporalio/auto-setup:1.28.1
    restart: unless-stopped
    depends_on:
      - temporal-postgresql
      - temporal-elasticsearch
    environment:
      - DB=postgres12
      - DB_PORT=5432
      - POSTGRES_USER=${TEMPORAL_POSTGRES_USER:-temporal}
      - POSTGRES_PWD=${TEMPORAL_POSTGRES_PASSWORD}
      - POSTGRES_SEEDS=temporal-postgresql
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
      - ENABLE_ES=true
      - ES_SEEDS=temporal-elasticsearch
      - ES_VERSION=v7
      - TEMPORAL_NAMESPACE=default
    networks:
      - postiz-internal
    volumes:
      - ./dynamicconfig:/etc/temporal/config/dynamicconfig

  temporal-admin-tools:
    container_name: temporal-admin-tools
    image: temporalio/admin-tools:1.28.1-tctl-1.18.4-cli-1.4.1
    restart: unless-stopped
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CLI_ADDRESS=temporal:7233
    networks:
      - postiz-internal
    depends_on:
      - temporal

  temporal-ui:
    container_name: temporal-ui
    image: temporalio/ui:2.34.0
    restart: unless-stopped
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://127.0.0.1:3000
    networks:
      - postiz-internal

volumes:
  postiz-postgres-data:
  postiz-redis-data:
  postiz-config:
  postiz-uploads:
  temporal-postgres-data:
  temporal-es-data:

networks:
  postiz-internal:
    driver: bridge
  dokploy-network:
    external: true
```

- [ ] **Step 2: Validar a sintaxe do arquivo Compose com docker compose config**

Executar:
```powershell
docker compose --env-file .env.dokploy.example -f docker-compose.dokploy.yaml config
```
Esperado: Validação com sucesso da estrutura sem erros de sintaxe ou referências inválidas.

- [ ] **Step 3: Commit das alterações do Task 2**

```powershell
git add docker-compose.dokploy.yaml; git commit -m "feat(dokploy): add production docker-compose for Dokploy"
```

---

### Task 3: Criar Guia Operacional Completo de Deploy no Dokploy

**Files:**
- Create: `c:\www\postiz-app\docs\deployment\dokploy-guide.md`

**Interfaces:**
- Produces: Manual prático para guiar a configuração no painel web do Dokploy, criação de bucket Cloudflare R2 e DNS.
- Consumes: Arquivos e variáveis criados nas Tasks 1 e 2.

- [ ] **Step 1: Criar o documento `docs/deployment/dokploy-guide.md`**

Criar `c:\www\postiz-app\docs\deployment\dokploy-guide.md` com instruções detalhadas cobrindo:
1. Pré-requisitos (VPS com Dokploy rodando, domínio apontado, bucket Cloudflare R2 criado com token API de leitura/escrita).
2. Como criar a aplicação do tipo **Compose** no Dokploy.
3. Como conectar o repositório Git (`https://github.com/thiagojordao98/postiz-app`, branch `main`, arquivo `docker-compose.dokploy.yaml`).
4. Como configurar as variáveis de ambiente copiadas de `.env.dokploy.example`.
5. Como configurar a aba **Domains** no Dokploy (apontando para service `postiz`, port `5000`, ativando Let's Encrypt).
6. Procedimento de primeiro acesso, registro do primeiro usuário e verificação dos logs.

- [ ] **Step 2: Validar formatação e links do guia**

Inspecionar o arquivo criado para garantir formatação markdown perfeita e sem placeholders.

- [ ] **Step 3: Commit das alterações do Task 3**

```powershell
git add docs/deployment/dokploy-guide.md; git commit -m "docs(dokploy): add complete deployment guide"
```

---

### Task 4: Validação Final e Sincronização com o Repositório GitHub

**Files:**
- Modify: `c:\www\postiz-app\README.md` (adicionar seção/badge de deploy Dokploy e link para o guia)

**Interfaces:**
- Produces: Repositório sincronizado no GitHub (`origin/main`) pronto para deploy no Dokploy.

- [ ] **Step 1: Adicionar referência ao guia Dokploy no `README.md`**

Inserir link e instruções rápidas para deploy via Dokploy no arquivo [README.md](file:///c:/www/postiz-app/README.md).

- [ ] **Step 2: Executar verificação git status e git diff**

Executar:
```powershell
git status
```
Esperado: árvore de trabalho limpa ou apenas alterações rastreadas.

- [ ] **Step 3: Enviar todos os commits para o GitHub origin/main**

Executar:
```powershell
git push origin main
```
Esperado: Envio bem-sucedido para `https://github.com/thiagojordao98/postiz-app.git`.
