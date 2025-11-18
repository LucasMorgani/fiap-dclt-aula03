# 🎬 Vídeo 3.1 - Build e Publicação de Imagens Docker

**Aula**: 3 - Docker e Kubernetes  
**Vídeo**: 3.1  
**Temas**: Multi-stage build; ECR; OIDC; Pipeline Docker  

---

## 📚 Parte 1: Conceito AWS ECR

### Passo 1: O que é ECR?

**ECR - Elastic Container Registry:**

```mermaid
graph LR
    A[Docker Hub] -->|Público| B[Limitações]
    C[ECR AWS] -->|Privado| D[Ilimitado]
    C -->|Integrado| E[AWS Services]
```

**Vantagens do ECR:**
- ✅ Integração nativa com AWS
- ✅ Segurança (IAM, encryption)
- ✅ Scan de vulnerabilidades
- ✅ Sem limite de pull
- ✅ Lifecycle policies

---

## 🔧 Parte 2: Setup AWS

### Passo 2: Verificar Estrutura

```bash
# Ver estrutura do projeto
tree -L 2

# Estrutura:
# ├── app/              # Aplicação Todo API
# ├── Dockerfile        # Multi-stage build
# └── k8s/              # Manifests Kubernetes
```

### Passo 3: Criar ECR Repository

```bash
# Verificar credenciais AWS Learner Lab
aws sts get-caller-identity --profile fiapaws

# Criar repository ECR
aws ecr create-repository \
  --repository-name fiap-todo-api \
  --region us-east-1 \
  --profile fiapaws

# Salvar URI do repository
ECR_URI=$(aws ecr describe-repositories \
  --repository-names fiap-todo-api \
  --region us-east-1 \
  --profile fiapaws \
  --query 'repositories[0].repositoryUri' \
  --output text)

echo "ECR URI: $ECR_URI"
```

---

## 🐳 Parte 3: Dockerfile Otimizado

### Passo 4: Arquitetura Multi-stage

```mermaid
graph TB
    subgraph "Stage 1: Dependencies"
        A[node:18-alpine] --> B[npm ci]
        B --> C[node_modules]
    end
    
    subgraph "Stage 2: Builder"
        D[npm ci + build] --> E[Testes]
        E --> F[Artifacts]
    end
    
    subgraph "Stage 3: Runtime"
        G[Copiar deps] --> H[Copiar código]
        H --> I[Imagem final]
    end
    
    C --> G
    F --> H
```

### Passo 5: Ver Dockerfile Multi-stage

```bash
# Ver Dockerfile
cat Dockerfile
```

**Dockerfile:**
```dockerfile
# Stage 1: Dependencies
FROM node:18-alpine AS deps
WORKDIR /app
COPY app/package*.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:18-alpine AS builder
WORKDIR /app
COPY app/package*.json ./
RUN npm ci
COPY app/ ./
RUN npm run build --if-present
RUN npm test

# Stage 3: Runtime
FROM node:18-alpine
WORKDIR /app

# Criar usuário não-root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copiar apenas o necessário
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/src ./src

# Mudar para usuário não-root
USER nodejs

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "src/server.js"]
```

### Passo 6: Build Local

```bash
# Build da imagem
docker build -t fiap-todo-api:local .

# Ver tamanho
docker images fiap-todo-api:local

# Rodar container
docker run -d --name todo-test -p 3000:3000 fiap-todo-api:local

# Testar
curl http://localhost:3000/todos

# Limpar
docker stop todo-test && docker rm todo-test
```

---

## 🔐 Parte 4: Credenciais AWS para GitHub Actions

### Passo 7: Fluxo de Autenticação

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant Secrets as GitHub Secrets
    participant AWS as AWS
    participant ECR as AWS ECR
    
    GH->>Secrets: 1. Ler credenciais
    Secrets->>GH: 2. AWS_ACCESS_KEY_ID + SECRET
    GH->>AWS: 3. Autenticar
    AWS->>GH: 4. Sessão autenticada
    GH->>ECR: 5. Push image
    
    Note over GH,ECR: Credenciais seguras no GitHub Secrets
```

**⚠️ Importante - AWS Learner Lab:**
- Não podemos criar roles customizadas (sem OIDC)
- Usaremos AWS Access Keys do Learner Lab
- Credenciais armazenadas como GitHub Secrets

### Passo 8: Obter Credenciais AWS

```bash
# No AWS Learner Lab, clicar em "AWS Details"
# Copiar as credenciais:
# - aws_access_key_id
# - aws_secret_access_key
# - aws_session_token (se disponível)

# Verificar credenciais
aws sts get-caller-identity --profile fiapaws
```

### Passo 9: Configurar GitHub Secrets

**No GitHub Repository:**

1. Ir em **Settings** → **Secrets and variables** → **Actions**
2. Clicar em **New repository secret**
3. Adicionar os seguintes secrets:

```
Nome: AWS_ACCESS_KEY_ID
Valor: [copiar do AWS Details]

Nome: AWS_SECRET_ACCESS_KEY
Valor: [copiar do AWS Details]

Nome: AWS_SESSION_TOKEN (opcional, se disponível)
Valor: [copiar do AWS Details]

Nome: AWS_REGION
Valor: us-east-1
```

**🚨 Segurança:**
- Nunca commitar credenciais no código
- Usar apenas GitHub Secrets
- Renovar credenciais quando a sessão do Learner Lab expirar

---

## 🚀 Parte 5: Pipeline Docker

### Passo 10: Fluxo do Pipeline

```mermaid
graph LR
    A[Push Code] --> B[Checkout]
    B --> C[Setup Buildx]
    C --> D[AWS Auth]
    D --> E[Login ECR]
    E --> F[Build Image]
    F --> G[Push ECR]
    G --> H[Scan Vulnerabilities]
```

### Passo 11: Criar Workflow (Faremos juntos na aula)

**Workflow que criaremos:**
```yaml
name: 🐳 Docker Build and Push

on:
  push:
    branches: [ main ]
    paths: [ 'app/**', 'Dockerfile' ]
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: fiap-todo-api

jobs:
  build-and-push:
    name: 🐳 Build and Push to ECR
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout código
        uses: actions/checkout@v4
      
      - name: 🔧 Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: 🔑 Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: 🔑 Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: 🏷️ Generate image tags
        id: meta
        run: |
          echo "tags<<EOF" >> $GITHUB_OUTPUT
          echo "${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest" >> $GITHUB_OUTPUT
          echo "${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}" >> $GITHUB_OUTPUT
          echo "${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:v1.${{ github.run_number }}" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: 🐳 Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64
      
      - name: 🔍 Scan image
        run: |
          aws ecr start-image-scan \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=latest \
            --region ${{ env.AWS_REGION }}
      
      - name: 📊 Summary
        run: |
          echo "## 🐳 Docker Build Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Repository**: ${{ env.ECR_REPOSITORY }}" >> $GITHUB_STEP_SUMMARY
          echo "**Tags**:" >> $GITHUB_STEP_SUMMARY
          echo "- \`latest\`" >> $GITHUB_STEP_SUMMARY
          echo "- \`${{ github.sha }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- \`v1.${{ github.run_number }}\`" >> $GITHUB_STEP_SUMMARY
```

---

## 🧪 Parte 6: Testar Pipeline

### Passo 12: Verificar Secrets

**Verificar se os secrets foram configurados:**
- ✅ `AWS_ACCESS_KEY_ID`
- ✅ `AWS_SECRET_ACCESS_KEY`
- ✅ `AWS_SESSION_TOKEN` (opcional)
- ✅ `AWS_REGION`

### Passo 13: Commit e Push

```bash
# Adicionar workflow que criamos
git add .github/workflows/docker-build.yml

# Commit
git commit -m "feat: adicionar pipeline Docker com ECR"
git push origin main

# Ver no GitHub Actions
# ✅ Build and Push to ECR
```

### Passo 14: Verificar no ECR

```bash
# Listar imagens
aws ecr list-images \
  --repository-name fiap-todo-api \
  --region us-east-1 \
  --profile fiapaws

# Ver detalhes
aws ecr describe-images \
  --repository-name fiap-todo-api \
  --region us-east-1 \
  --profile fiapaws
```

---

## 🔍 Parte 7: Scan de Vulnerabilidades

### Passo 15: Ver Resultados do Scan

```bash
# Aguardar scan completar (1-2 min)
sleep 120

# Ver resultados
aws ecr describe-image-scan-findings \
  --repository-name fiap-todo-api \
  --image-id imageTag=latest \
  --region us-east-1 \
  --profile fiapaws
```

### Passo 16: Configurar Scan Automático

```bash
# Habilitar scan automático
aws ecr put-image-scanning-configuration \
  --repository-name fiap-todo-api \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1 \
  --profile fiapaws
```

---

## 🎓 Parte 8: Conceitos Aprendidos

### Passo 17: Fluxo Completo

```mermaid
graph LR
    A[Code] --> B[Build Multi-stage]
    B --> C[Push ECR]
    C --> D[Scan]
    D --> E[Deploy K8s]
```

**O que aprendemos:**
- ✅ Dockerfile multi-stage otimizado
- ✅ AWS ECR como registry privado
- ✅ GitHub Secrets para credenciais AWS (Learner Lab)
- ✅ Pipeline Docker automatizado
- ✅ Multiple tags (latest, sha, version)
- ✅ Scan de vulnerabilidades
- ✅ Cache de layers

---

**FIM DO VÍDEO 3.1** ✅
