# Guia de Implantação do Postiz no Dokploy com Cloudflare R2

Este guia orienta o passo a passo completo para implantar o **Postiz** em produção em uma VPS utilizando o **Dokploy** como gerenciador de containers e o **Cloudflare R2** para armazenamento de mídias.

---

## 1. Pré-Requisitos

Antes de iniciar no Dokploy, certifique-se de ter:

1. **VPS com Dokploy Instalado e Operante**:
   - Mínimo recomendado: 2 vCPU e 4 GB de RAM (devido ao Elasticsearch e Temporal).
   - Se sua VPS possuir 2 GB de RAM, crie um arquivo Swap de 2 GB a 4 GB para evitar que o Elasticsearch sofra OOM (Out Of Memory).
2. **Domínio ou Subdomínio Apontado**:
   - Crie um registro DNS do **Tipo A** no seu provedor de domínio (ex: `postiz.seudominio.com`) apontando para o IP público da sua VPS.
3. **Bucket no Cloudflare R2**:
   - No painel da Cloudflare, acesse **R2** > **Create bucket**.
   - Nome do bucket: ex. `postiz-uploads`.
   - Na aba **Settings** do bucket, em **Public Access**, habilite um **Custom Domain** (ex: `media.seudominio.com`) ou habilite o **R2.dev subdomain** (ex: `https://pub-xxxxxx.r2.dev`).
   - Crie um **R2 API Token** (em R2 > *Manage R2 API Tokens* > *Create API Token*) com permissões de **Object Read & Write**.
   - Guarde as credenciais geradas:
     - Account ID (visível na URL ou na página do R2)
     - Access Key ID
     - Secret Access Key

---

## 2. Passo a Passo no Dokploy

### Passo 2.1: Criar o Serviço Compose

1. Acesse o painel web do seu **Dokploy**.
2. Selecione o seu projeto (ou crie um novo projeto chamado `Postiz`).
3. Clique em **Create Service** e selecione **Compose**.
4. Defina um nome para o serviço (ex: `postiz-stack`).

### Passo 2.2: Conectar o Repositório Git

1. Na aba **Source / General** do Compose recém-criado:
   - **Source Type**: `Git`
   - **Repository URL**: `https://github.com/thiagojordao98/postiz-app`
   - **Branch**: `main`
   - **Compose Path**: `docker-compose.dokploy.yaml`
2. Clique em **Save**.

### Passo 2.3: Configurar as Variáveis de Ambiente

1. Acesse a aba **Environment** do serviço Compose no Dokploy.
2. Copie o conteúdo abaixo, preencha com seus dados reais e cole no campo:

```env
# === Domínio e URLs Públicas ===
POSTIZ_DOMAIN=postiz.seudominio.com
MAIN_URL=https://postiz.seudominio.com
FRONTEND_URL=https://postiz.seudominio.com
NEXT_PUBLIC_BACKEND_URL=https://postiz.seudominio.com/api
BACKEND_INTERNAL_URL=http://localhost:3000

# === Segurança e Sessão ===
# Gere uma string aleatória forte de pelo menos 32 caracteres:
JWT_SECRET=gere_uma_chave_super_secreta_com_pelo_menos_32_caracteres
IS_GENERAL=true
DISABLE_REGISTRATION=false

# === Banco de Dados Principal (PostgreSQL) ===
POSTGRES_USER=postiz_user
POSTGRES_PASSWORD=defina_uma_senha_forte_aqui
POSTGRES_DB=postiz_production
DATABASE_URL=postgresql://postiz_user:defina_uma_senha_forte_aqui@postiz-postgres:5432/postiz_production

# === Cache e Filas (Redis) ===
REDIS_URL=redis://postiz-redis:6379

# === Temporal Workflow Engine ===
TEMPORAL_ADDRESS=temporal:7233
TEMPORAL_POSTGRES_USER=temporal
TEMPORAL_POSTGRES_PASSWORD=defina_outra_senha_forte_para_o_temporal

# === Armazenamento Cloudflare R2 (S3-Compatible) ===
STORAGE_PROVIDER=cloudflare
CLOUDFLARE_ACCOUNT_ID=seu_cloudflare_account_id
CLOUDFLARE_ACCESS_KEY=sua_access_key_id_do_r2
CLOUDFLARE_SECRET_ACCESS_KEY=sua_secret_access_key_do_r2
CLOUDFLARE_BUCKETNAME=postiz-uploads
CLOUDFLARE_BUCKET_URL=https://pub-xxxxxx.r2.dev
CLOUDFLARE_REGION=auto

# === Limites de Requisições ===
API_LIMIT=30
NX_ADD_PLUGINS=false
```

3. Clique em **Save**.

### Passo 2.4: Configurar o Domínio e Roteamento SSL (Traefik)

1. Acesse a aba **Domains** do serviço Compose.
2. Clique em **Add Domain**:
   - **Host**: `postiz.seudominio.com` (o mesmo configurado em `POSTIZ_DOMAIN`)
   - **Service**: Selecione `postiz`
   - **Container Port**: `5000`
   - **Certificate**: Marque para habilitar Let's Encrypt / HTTPS.
3. Clique em **Save**.

### Passo 2.5: Realizar o Deploy

1. No topo da página do serviço no Dokploy, clique em **Deploy**.
2. Acompanhe a aba **Deployments** e a aba **Logs**.
3. Na primeira inicialização:
   - O PostgreSQL do Postiz e do Temporal serão criados e inicializados.
   - O Elasticsearch subirá e inicializará os índices.
   - O Temporal executará as migrações automáticas de schema.
   - O Postiz se conectará ao banco, executará as migrações do Prisma e iniciará o servidor na porta 5000.

---

## 3. Primeiro Acesso e Validação

1. Abra no navegador: `https://postiz.seudominio.com`.
2. Você deverá ver a tela inicial do Postiz protegida com certificado SSL válido (HTTPS).
3. Cadastre seu usuário inicial (ele será o administrador da sua instância).
4. No painel do Postiz, crie um post de teste e faça upload de uma imagem.
   - A imagem deverá ser salva e servida diretamente através do Cloudflare R2.
5. (Opcional) Para fechar sua instância e impedir que terceiros criem contas públicas:
   - Volte na aba **Environment** no Dokploy.
   - Altere `DISABLE_REGISTRATION=true`.
   - Clique em **Redeploy**.

---

## 4. Dicas de Otimização e Resolução de Problemas

### Criando Swap na VPS (Caso sua VPS tenha 2GB a 4GB de RAM)
Se os containers do Elasticsearch ou Postiz forem finalizados inesperadamente com erro `OOMKilled` (Exit code 137), adicione memória swap na sua VPS via terminal SSH:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Reiniciar ou Atualizar a Imagem do Postiz
Para atualizar o Postiz no futuro:
- No Dokploy, acesse o Compose `postiz-stack` e clique em **Redeploy**. A imagem `ghcr.io/gitroomhq/postiz-app:latest` será baixada novamente se houver nova versão.
