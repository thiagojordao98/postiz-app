# Design de Implantação do Postiz no Dokploy com Cloudflare R2

**Data**: 2026-09-09  
**Status**: Aprovado  
**Repositório**: [thiagojordao98/postiz-app](https://github.com/thiagojordao98/postiz-app)

---

## 1. Visão Geral

Este documento define a arquitetura, topologia de containers, configurações de ambiente e processo de deploy do **Postiz** em uma VPS gerenciada com **Dokploy**.

A solução adota a execução via Docker Compose com a imagem oficial pré-compilada do Postiz (`ghcr.io/gitroomhq/postiz-app:latest`), armazenamento de mídias externo no **Cloudflare R2** (compatível com S3) e integração nativa com o proxy reverso **Traefik** do Dokploy para emissão automática de certificados SSL (Let's Encrypt).

---

## 2. Arquitetura e Topologia de Containers

A stack será declarada no arquivo `docker-compose.dokploy.yaml` e composta pelos seguintes serviços:

### 2.1 Aplicação Principal
- **`postiz`**:
  - Imagem: `ghcr.io/gitroomhq/postiz-app:latest`
  - Porta interna da aplicação: `5000`
  - Redes:
    - `postiz-internal`: rede interna privada para comunicação com bancos e Temporal.
    - `dokploy-network` (externa): rede do Traefik/Dokploy para receber tráfego HTTPS.
  - Dependências: aguarda `postiz-postgres` e `postiz-redis` estarem saudáveis (`service_healthy`).
  - Volumes:
    - `postiz-config:/config/`
    - `postiz-uploads:/uploads/` (cache local de processamento)

### 2.2 Bancos de Dados e Filas
- **`postiz-postgres`**:
  - Imagem: `postgres:17-alpine`
  - Rede: `postiz-internal` (sem portas publicadas para a internet).
  - Healthcheck configurado (`pg_isready`).
  - Volume persistente: `postiz-postgres-data:/var/lib/postgresql/data`.
- **`postiz-redis`**:
  - Imagem: `redis:7.2`
  - Rede: `postiz-internal` (sem portas publicadas para a internet).
  - Healthcheck configurado (`redis-cli ping`).
  - Volume persistente: `postiz-redis-data:/data`.

### 2.3 Orquestrador de Tarefas (Temporal Stack)
- **`temporal`**:
  - Imagem: `temporalio/auto-setup:1.28.1`
  - Porta interna: `7233`
  - Rede: `postiz-internal`
  - Volume montado: `./dynamicconfig:/etc/temporal/config/dynamicconfig`
- **`temporal-postgresql`**:
  - Imagem: `postgres:16`
  - Rede: `postiz-internal`
  - Volume persistente: `temporal-postgres-data:/var/lib/postgresql/data`.
- **`temporal-elasticsearch`**:
  - Imagem: `elasticsearch:7.17.27`
  - Parâmetros JVM: `ES_JAVA_OPTS=-Xms256m -Xmx256m`
  - Rede: `postiz-internal`
  - Volume persistente: `temporal-es-data:/usr/share/elasticsearch/data`.
- **`temporal-admin-tools`** e **`temporal-ui`**:
  - Imagem UI: `temporalio/ui:2.34.0` (porta `8080` interna se necessário inspecionar workflows).

---

## 3. Configuração de Variáveis de Ambiente e Cloudflare R2

Será disponibilizado o arquivo `.env.dokploy.example` para preenchimento na aba **Environment** do Dokploy:

```env
# === Domínio e URLs ===
POSTIZ_DOMAIN=postiz.seudominio.com
MAIN_URL=https://postiz.seudominio.com
FRONTEND_URL=https://postiz.seudominio.com
NEXT_PUBLIC_BACKEND_URL=https://postiz.seudominio.com/api
BACKEND_INTERNAL_URL=http://localhost:3000

# === Segurança e Sessão ===
JWT_SECRET=substitua_por_uma_string_aleatoria_longa_e_segura_32_chars
IS_GENERAL=true
DISABLE_REGISTRATION=false

# === Banco de Dados Principal ===
POSTGRES_USER=postiz_user
POSTGRES_PASSWORD=gere_uma_senha_forte_aqui
POSTGRES_DB=postiz_production
DATABASE_URL=postgresql://postiz_user:gere_uma_senha_forte_aqui@postiz-postgres:5432/postiz_production
REDIS_URL=redis://postiz-redis:6379

# === Temporal ===
TEMPORAL_ADDRESS=temporal:7233
TEMPORAL_POSTGRES_PASSWORD=gere_outra_senha_forte_aqui

# === Armazenamento Cloudflare R2 (S3-compatible) ===
STORAGE_PROVIDER=cloudflare
CLOUDFLARE_ACCOUNT_ID=seu_account_id_cloudflare
CLOUDFLARE_ACCESS_KEY=sua_access_key_r2
CLOUDFLARE_SECRET_ACCESS_KEY=sua_secret_access_key_r2
CLOUDFLARE_BUCKETNAME=postiz-uploads
CLOUDFLARE_BUCKET_URL=https://pub-xxxxxx.r2.dev
CLOUDFLARE_REGION=auto
```

---

## 4. Integração com Dokploy e Roteamento

1. **Repositório Git**:
   - O repositório oficial da stack no GitHub (`thiagojordao98/postiz-app`) é conectado diretamente ao Dokploy.
   - Branch: `main`.
   - Tipo de Serviço: **Compose**.
   - Caminho do Compose: `docker-compose.dokploy.yaml`.

2. **Roteamento Traefik via Dokploy**:
   - Rede externa compartilhada: `dokploy-network`.
   - Configuração de Domínio na UI do Dokploy:
     - Host: `postiz.seudominio.com`
     - Service: `postiz`
     - Container Port: `5000`
     - HTTPS / Let's Encrypt: Ativado automaticamente.

---

## 5. Plano de Validação e Verificação

1. **Sintaxe do Compose**:
   - Validação local do arquivo com `docker compose -f docker-compose.dokploy.yaml config`.
2. **Commit e Push**:
   - Inclusão e commit de `docker-compose.dokploy.yaml`, `.env.dokploy.example` e documentação de apoio.
3. **Deploy no Dokploy**:
   - Acionar o deploy no Dokploy e checar logs dos serviços (`postiz-postgres`, `postiz-redis`, `temporal` e `postiz`).
4. **Verificação Funcional**:
   - Acesso HTTPS a `https://postiz.seudominio.com`.
   - Criação da primeira conta (Admin).
   - Teste de upload de mídia no Cloudflare R2.
