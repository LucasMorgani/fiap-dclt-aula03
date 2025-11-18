# 🎬 Vídeo 3.3 - Estratégias de Deploy Avançadas (Blue/Green e Canary)

**Aula**: 3 - Docker e Kubernetes  
**Vídeo**: 3.3  
**Temas**: Blue/Green; Canary; Rolling Update; Rollback  

---

## 📋 Pré-requisitos

**⚠️ Importante: Cluster EKS da Aula 01 ou Vídeo 3.2**

Este vídeo **reutiliza** o cluster EKS criado anteriormente. Não precisa criar um novo!

**Opções:**

1. **Cluster já existe e está ativo (Aula 01 ou Vídeo 3.2):**
   - ✅ Use o mesmo cluster: `cicd-lab`
   - ✅ Verifique se está conectado: `kubectl get nodes`
   - ✅ Continue com este vídeo

2. **Cluster foi deletado:**
   - 📚 Consulte os comandos da **Aula 01**
   - 📂 Repositório: [fiap-dclt-aula01](https://github.com/josenetoo/fiap-dclt-aula01)
   - 🔄 Ou siga o **Vídeo 3.2** (Parte 2: Criar Cluster EKS)
   - Recrie o cluster usando os mesmos comandos

3. **Primeira vez (não fez Aula 01 nem Vídeo 3.2):**
   - 📚 Vá para **Vídeo 3.2** primeiro
   - Crie o cluster EKS
   - Depois volte para este vídeo

**Verificar se cluster existe:**
```bash
# Ver clusters disponíveis
aws eks list-clusters --region us-east-1

# Testar conexão
kubectl get nodes

# Reconfigurar kubectl (se necessário)
aws eks update-kubeconfig \
  --name cicd-lab \
  --region us-east-1
```

**Pré-requisitos adicionais:**
- ✅ **kubectl configurado** e conectado ao cluster
- ✅ **Aplicação deployada** com Kustomize (Vídeo 3.2)
- ✅ **Service LoadBalancer** funcionando

---

## 📚 Parte 1: Estratégias de Deploy

### Passo 1: Comparação de Estratégias

```mermaid
graph TB
    subgraph "Rolling Update"
        A1[V1: 100%] --> A2[V1: 80% + V2: 20%]
        A2 --> A3[V1: 50% + V2: 50%]
        A3 --> A4[V2: 100%]
    end
    
    subgraph "Blue/Green"
        B1[Blue: 100%] --> B2[Switch]
        B2 --> B3[Green: 100%]
    end
    
    subgraph "Canary"
        C1[V1: 90% + V2: 10%] --> C2[Monitor]
        C2 --> C3[V2: 100%]
    end
```

**Características:**
- **Rolling Update**: Gradual, sem downtime, padrão K8s
- **Blue/Green**: Instantâneo, fácil rollback, requer 2x recursos
- **Canary**: Teste com poucos usuários, baixo risco

---

## ☸️ Parte 2: Cluster EKS (Criado na Aula 01)

**⚠️ O cluster EKS já foi criado na Aula 01!**

Este vídeo **não cria** um novo cluster. Reutilizamos o cluster `cicd-lab` criado anteriormente.

### Passo 2: Verificar Cluster Existente

```bash
# Ver clusters disponíveis
aws eks list-clusters --region us-east-1

# Verificar nodes
kubectl get nodes

# Ver informações do cluster
kubectl cluster-info
```

**Se o cluster não existir:**
- 📚 Consulte a **Aula 01** para criar o cluster
- 📂 Repositório: [fiap-dclt-aula01](https://github.com/josenetoo/fiap-dclt-aula01)
- 🔄 Ou siga o **Vídeo 3.2** (Parte 2: Criar Cluster EKS)

### Passo 3: Reconfigurar kubectl (se necessário)

```bash
# Reconfigurar acesso ao cluster
aws eks update-kubeconfig \
  --name cicd-lab \
  --region us-east-1

# Testar conexão
kubectl get nodes

# Ver pods em execução
kubectl get pods --all-namespaces
```

**✅ Cluster pronto!** Agora vamos explorar estratégias de deploy avançadas.

---

## 🔵🟢 Parte 3: Blue/Green Deploy

### Passo 4: Arquitetura Blue/Green

```mermaid
graph TB
    A[LoadBalancer] --> B{Service}
    
    B -->|version: blue| C[Blue Deployment]
    B -.->|standby| D[Green Deployment]
    
    C --> E[Pod Blue 1]
    C --> F[Pod Blue 2]
    C --> G[Pod Blue 3]
    
    D --> H[Pod Green 1]
    D --> I[Pod Green 2]
    D --> J[Pod Green 3]
```

### Passo 5: Entender Manifests Blue/Green

**Os arquivos já estão criados no repositório em `k8s/blue-green/`**

**💡 Nota sobre Tags de Imagem:**
- Usamos `latest` para ambos (blue e green)
- A diferenciação é feita pela **variável de ambiente `VERSION`**
- Blue: `VERSION=v1.0-blue`
- Green: `VERSION=v2.0-green`
- Isso simula versões diferentes sem precisar criar tags no ECR
- Em produção real, você usaria tags específicas (`v1.0`, `v2.0`, etc.)

Vamos entender o que cada um faz:

**Deployment Blue (`deployment-blue.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fiap-todo-blue
  labels:
    app: fiap-todo-api
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fiap-todo-api
      version: blue
  template:
    metadata:
      labels:
        app: fiap-todo-api
        version: blue
    spec:
      containers:
      - name: api
        image: 777870534201.dkr.ecr.us-east-1.amazonaws.com/fiap-todo-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: VERSION
          value: "v1.0-blue"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

**Pontos-chave:**
- Label `version: blue` identifica esta versão
- Imagem: `v1.0` (versão atual em produção)

**Deployment Green (`deployment-green.yaml`):**
```yaml
# Similar ao Blue, mas com:
# - name: fiap-todo-green
# - version: green
# - image: v2.0 (nova versão)
```

**Service (`service.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fiap-todo-api
spec:
  type: LoadBalancer
  selector:
    app: fiap-todo-api
    version: blue  # ⬅️ Aponta para blue inicialmente
  ports:
  - port: 80
    targetPort: 3000
```

**🔑 Conceito-chave**: O Service usa o selector `version` para rotear tráfego. Mudando apenas essa label, fazemos o switch instantâneo!

### Passo 6: Deploy Blue/Green

```bash
# Deploy blue (versão atual)
kubectl apply -f k8s/blue-green/deployment-blue.yaml
kubectl apply -f k8s/blue-green/service.yaml

# Aguardar blue estar pronto
kubectl rollout status deployment/fiap-todo-blue

# Testar blue
LB_URL=$(kubectl get service fiap-todo-api -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$LB_URL/health

# Deploy green (nova versão)
kubectl apply -f k8s/blue-green/deployment-green.yaml

# Aguardar green estar pronto
kubectl rollout status deployment/fiap-todo-green

# Testar green diretamente (port-forward)
kubectl port-forward deployment/fiap-todo-green 8080:3000 &
curl http://localhost:8080/health
pkill -f "port-forward"
```

### Passo 7: Switch Blue → Green

```bash
# Verificar versão atual
CURRENT=$(kubectl get service fiap-todo-api -o jsonpath='{.spec.selector.version}')
echo "Current version: $CURRENT"

# Fazer o switch para green
kubectl patch service fiap-todo-api -p '{"spec":{"selector":{"version":"green"}}}'

echo "✅ Switched to green"
echo "Testing..."

sleep 5
LB_URL=$(kubectl get service fiap-todo-api -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -f http://$LB_URL/health && echo "✅ Health check passed!"
```

### Passo 9: Rollback Blue/Green

```bash
# Se green tiver problema, voltar para blue
kubectl patch service fiap-todo-api -p '{"spec":{"selector":{"version":"blue"}}}'

echo "✅ Rollback to blue completed!"
# Instantâneo! Sem downtime!
```

---

## 🐤 Parte 4: Canary Deploy

### Passo 10: Arquitetura Canary com Istio

```mermaid
graph TB
    A[Istio Gateway] --> B[VirtualService]
    
    B -->|90% weight| C[Stable v1.0]
    B -->|10% weight| D[Canary v2.0]
    
    C --> E[Pod Stable 1]
    C --> F[Pod Stable 2]
    
    D --> G[Pod Canary 1]
    D --> H[Pod Canary 2]
```

**Por que Istio?**
- ✅ Controle preciso de tráfego por peso (não depende de número de réplicas)
- ✅ Roteamento inteligente baseado em headers, cookies, etc
- ✅ Métricas e observabilidade integradas
- ✅ Rollback instantâneo
- ✅ Usado em produção por grandes empresas

### Passo 11: Entender Deployments e Services

**Os arquivos já estão criados em `k8s/canary-istio/`**

**💡 Nota sobre Tags de Imagem:**
- Usamos `latest` para ambos (stable e canary)
- A diferenciação é feita pela **variável de ambiente `VERSION`**
- Stable: `VERSION=v1.0`
- Canary: `VERSION=v2.0`
- Isso permite testar sem criar múltiplas tags no ECR
- Em produção, você usaria tags específicas para cada versão

Vamos entender cada um:

**Deployment Stable (`deployment-stable.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fiap-todo-stable
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fiap-todo-api
      version: v1
  template:
    metadata:
      labels:
        app: fiap-todo-api
        version: v1
    spec:
      containers:
      - name: api
        image: 777870534201.dkr.ecr.us-east-1.amazonaws.com/fiap-todo-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: VERSION
          value: "v1.0"
```

**Deployment Canary (`deployment-canary.yaml`):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fiap-todo-canary
spec:
  replicas: 2  # Mesmo número! Istio controla o tráfego
  selector:
    matchLabels:
      app: fiap-todo-api
      version: v2
  template:
    metadata:
      labels:
        app: fiap-todo-api
        version: v2
    spec:
      containers:
      - name: api
        image: 777870534201.dkr.ecr.us-east-1.amazonaws.com/fiap-todo-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: VERSION
          value: "v2.0"
```

**Service (`service.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: fiap-todo-api
spec:
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: fiap-todo-api  # Seleciona ambas versões
```

**🔑 Pontos importantes:**
- Ambos deployments têm **2 réplicas** (não importa o número!)
- Label `version: v1` e `version: v2` diferenciam as versões
- Service seleciona apenas `app: fiap-todo-api` (pega ambos)
- Estes são arquivos Kubernetes padrão - funcionam sem Istio!

### Passo 12: Instalar Istio

**Agora vamos adicionar o Istio por cima da infraestrutura:**

**Linux/Mac:**
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
```

**Windows (PowerShell):**
```powershell
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
$env:PATH += ";$PWD\bin"
```

**Ou baixar manualmente:**
- Acesse: https://github.com/istio/istio/releases
- Baixe a versão mais recente para seu OS
- Extraia e adicione `bin/` ao PATH

**Instalação (todos os sistemas):**
```bash
# Instalar Istio no cluster
istioctl install --set profile=demo -y

# Habilitar injeção automática de sidecar no namespace default
kubectl label namespace default istio-injection=enabled

# Verificar instalação
kubectl get pods -n istio-system

# Ver componentes instalados
kubectl get svc -n istio-system
```

**O que foi instalado:**
- ✅ `istiod`: Control plane (gerenciamento)
- ✅ `istio-ingressgateway`: Gateway de entrada
- ✅ `istio-egressgateway`: Gateway de saída (opcional)

### Passo 13: Entender Recursos Istio

**Os recursos Istio já estão em `k8s/canary-istio/`**

**Gateway (`gateway.yaml`)** - Expõe a aplicação externamente:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: fiap-todo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

**VirtualService (`virtualservice.yaml`)** - Controla o tráfego:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fiap-todo-api
spec:
  hosts:
  - "*"
  gateways:
  - fiap-todo-gateway
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"  # Header para testar canary
    route:
    - destination:
        host: fiap-todo-api
        subset: v2
      weight: 100
  - route:  # Tráfego normal
    - destination:
        host: fiap-todo-api
        subset: v1
      weight: 90  # 90% para stable
    - destination:
        host: fiap-todo-api
        subset: v2
      weight: 10  # 10% para canary
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: fiap-todo-api
spec:
  host: fiap-todo-api
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**🔑 Conceito-chave**: 
- **Gateway**: Expõe a aplicação externamente via Istio Ingress Gateway
- **VirtualService**: Controla a % de tráfego por peso (não depende de réplicas!)
- **DestinationRule**: Define os subsets (v1 e v2) baseados em labels
- **Header routing**: `x-canary: true` permite testar canary diretamente
- **Weights**: 90% stable + 10% canary = controle preciso

### Passo 14: Deploy e Testar Canary

**Deploy (todos os sistemas):**
```bash
# Deploy de tudo
kubectl apply -f k8s/canary-istio/

# Aguardar pods
kubectl rollout status deployment/fiap-todo-stable
kubectl rollout status deployment/fiap-todo-canary

# Ver pods (ambos com sidecar Istio)
kubectl get pods -l app=fiap-todo-api
# Cada pod terá 2 containers: app + istio-proxy
```

**Obter URL do Gateway:**

**Linux/Mac:**
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo "Gateway URL: $GATEWAY_URL"
```

**Windows (PowerShell):**
```powershell
$INGRESS_HOST = kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
$INGRESS_PORT = kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name==\"http2\")].port}'
$GATEWAY_URL = "$INGRESS_HOST:$INGRESS_PORT"
Write-Host "Gateway URL: $GATEWAY_URL"
```

**Testar distribuição de tráfego:**

**Linux/Mac:**
```bash
# 100 requisições
for i in {1..100}; do
  curl -s http://$GATEWAY_URL/health | jq -r '.version'
done | sort | uniq -c

# Resultado esperado:
# ~90 v1.0
# ~10 v2.0
```

**Windows (PowerShell):**
```powershell
# 100 requisições
1..100 | ForEach-Object {
  (curl -s "http://$GATEWAY_URL/health" | ConvertFrom-Json).version
} | Group-Object | Select-Object Name, Count

# Resultado esperado:
# v1.0: ~90
# v2.0: ~10
```

**Testar canary diretamente com header:**
```bash
# Linux/Mac/Windows (mesmo comando)
curl -H "x-canary: true" http://$GATEWAY_URL/health
# Sempre retorna v2.0
```

### Passo 15: Ajustar Peso do Canary

```bash
# Aumentar canary para 25%
kubectl patch virtualservice fiap-todo-api --type merge -p '
{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 75},
        {"destination": {"host": "fiap-todo-api", "subset": "v2"}, "weight": 25}
      ]
    }]
  }
}'

# Testar nova distribuição
for i in {1..100}; do
  curl -s http://$GATEWAY_URL/health | jq -r '.version'
done | sort | uniq -c
# Agora: ~75 v1.0, ~25 v2.0

# Aumentar para 50%
kubectl patch virtualservice fiap-todo-api --type merge -p '
{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 50},
        {"destination": {"host": "fiap-todo-api", "subset": "v2"}, "weight": 50}
      ]
    }]
  }
}'

# Promover para 100% (se tudo OK)
kubectl patch virtualservice fiap-todo-api --type merge -p '
{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "fiap-todo-api", "subset": "v2"}, "weight": 100}
      ]
    }]
  }
}'

echo "✅ Canary promovido para 100%!"
```

### Passo 16: Rollback Instantâneo

```bash
# Se detectar problema, voltar para v1 instantaneamente
kubectl patch virtualservice fiap-todo-api --type merge -p '
{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 100}
      ]
    }]
  }
}'

echo "✅ Rollback instantâneo para v1!"
# Sem restart de pods, sem downtime!
```

---

## 🚀 Parte 6: Pipeline com Estratégias

### Passo 17: Criar Workflow Canary Deploy

**Vamos criar um workflow que ajusta o tráfego Canary via Istio!**

**💡 Conceito**: A porcentagem do Canary é definida como **input manual** no GitHub Actions, permitindo ajustar o tráfego sem modificar código.

**Linux/Mac:**
```bash
mkdir -p .github/workflows

cat > .github/workflows/canary-deploy.yml << 'EOF'
name: 🐤 Canary Deploy with Istio

on:
  workflow_dispatch:
    inputs:
      canary-percentage:
        description: 'Canary percentage (0-100)'
        required: true
        default: '10'
        type: choice
        options:
          - '10'
          - '25'
          - '50'
          - '100'
      action:
        description: 'Action'
        required: true
        default: 'deploy'
        type: choice
        options:
          - 'deploy'
          - 'rollback'

jobs:
  canary-deploy:
    name: 🐤 Canary with Istio
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout
        uses: actions/checkout@v4
      
      - name: 🔑 Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: us-east-1
      
      - name: ☸️ Update kubeconfig
        run: |
          aws eks update-kubeconfig --name cicd-lab --region us-east-1
      
      - name: 🐤 Adjust Canary Traffic
        if: github.event.inputs.action == 'deploy'
        run: |
          CANARY_PCT=${{ github.event.inputs.canary-percentage }}
          STABLE_PCT=$((100 - CANARY_PCT))
          
          echo "🎯 Adjusting traffic: Stable $STABLE_PCT% | Canary $CANARY_PCT%"
          
          kubectl patch virtualservice fiap-todo-api --type merge -p "
          {
            \"spec\": {
              \"http\": [{
                \"route\": [
                  {\"destination\": {\"host\": \"fiap-todo-api\", \"subset\": \"v1\"}, \"weight\": $STABLE_PCT},
                  {\"destination\": {\"host\": \"fiap-todo-api\", \"subset\": \"v2\"}, \"weight\": $CANARY_PCT}
                ]
              }]
            }
          }"
          
          echo "✅ Traffic adjusted successfully!"
      
      - name: 🔙 Rollback to Stable
        if: github.event.inputs.action == 'rollback'
        run: |
          echo "🔙 Rolling back to 100% stable..."
          
          kubectl patch virtualservice fiap-todo-api --type merge -p '
          {
            "spec": {
              "http": [{
                "route": [
                  {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 100}
                ]
              }]
            }
          }'
          
          echo "✅ Rollback completed - 100% on stable!"
      
      - name: 📊 Deployment Summary
        run: |
          echo "## 🐤 Canary Deployment with Istio" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Action**: ${{ github.event.inputs.action }}" >> $GITHUB_STEP_SUMMARY
          
          if [ "${{ github.event.inputs.action }}" == "deploy" ]; then
            echo "**Canary Weight**: ${{ github.event.inputs.canary-percentage }}%" >> $GITHUB_STEP_SUMMARY
            echo "**Stable Weight**: $((100 - ${{ github.event.inputs.canary-percentage }}))%" >> $GITHUB_STEP_SUMMARY
          else
            echo "**Status**: Rolled back to 100% stable" >> $GITHUB_STEP_SUMMARY
          fi
          
          echo "" >> $GITHUB_STEP_SUMMARY
          kubectl get pods -l app=fiap-todo-api >> $GITHUB_STEP_SUMMARY
EOF

echo "✅ Workflow criado!"
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path .github\workflows

@'
name: 🐤 Canary Deploy with Istio

on:
  workflow_dispatch:
    inputs:
      canary-percentage:
        description: 'Canary percentage (0-100)'
        required: true
        default: '10'
        type: choice
        options:
          - '10'
          - '25'
          - '50'
          - '100'
      action:
        description: 'Action'
        required: true
        default: 'deploy'
        type: choice
        options:
          - 'deploy'
          - 'rollback'

jobs:
  canary-deploy:
    name: 🐤 Canary with Istio
    runs-on: ubuntu-latest
    
    steps:
      - name: 📥 Checkout
        uses: actions/checkout@v4
      
      - name: 🔑 Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: us-east-1
      
      - name: ☸️ Update kubeconfig
        run: |
          aws eks update-kubeconfig --name cicd-lab --region us-east-1
      
      - name: 🐤 Adjust Canary Traffic
        if: github.event.inputs.action == 'deploy'
        run: |
          CANARY_PCT=${{ github.event.inputs.canary-percentage }}
          STABLE_PCT=$((100 - CANARY_PCT))
          
          echo "🎯 Adjusting traffic: Stable $STABLE_PCT% | Canary $CANARY_PCT%"
          
          kubectl patch virtualservice fiap-todo-api --type merge -p "
          {
            \"spec\": {
              \"http\": [{
                \"route\": [
                  {\"destination\": {\"host\": \"fiap-todo-api\", \"subset\": \"v1\"}, \"weight\": $STABLE_PCT},
                  {\"destination\": {\"host\": \"fiap-todo-api\", \"subset\": \"v2\"}, \"weight\": $CANARY_PCT}
                ]
              }]
            }
          }"
          
          echo "✅ Traffic adjusted successfully!"
      
      - name: 🔙 Rollback to Stable
        if: github.event.inputs.action == 'rollback'
        run: |
          echo "🔙 Rolling back to 100% stable..."
          
          kubectl patch virtualservice fiap-todo-api --type merge -p '
          {
            "spec": {
              "http": [{
                "route": [
                  {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 100}
                ]
              }]
            }
          }'
          
          echo "✅ Rollback completed - 100% on stable!"
      
      - name: 📊 Deployment Summary
        run: |
          echo "## 🐤 Canary Deployment with Istio" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Action**: ${{ github.event.inputs.action }}" >> $GITHUB_STEP_SUMMARY
          
          if [ "${{ github.event.inputs.action }}" == "deploy" ]; then
            echo "**Canary Weight**: ${{ github.event.inputs.canary-percentage }}%" >> $GITHUB_STEP_SUMMARY
            echo "**Stable Weight**: $((100 - ${{ github.event.inputs.canary-percentage }}))%" >> $GITHUB_STEP_SUMMARY
          else
            echo "**Status**: Rolled back to 100% stable" >> $GITHUB_STEP_SUMMARY
          fi
          
          echo "" >> $GITHUB_STEP_SUMMARY
          kubectl get pods -l app=fiap-todo-api >> $GITHUB_STEP_SUMMARY
'@ | Out-File -FilePath .github\workflows\canary-deploy.yml -Encoding UTF8

Write-Host "✅ Workflow criado!"
```

### Passo 18: Como Funciona a Porcentagem do Canary

**🎯 Definição da Porcentagem:**

1. **Via GitHub Actions UI** (Manual):
   - Acesse: `Actions` → `Canary Deploy with Istio` → `Run workflow`
   - Escolha a porcentagem: `10%`, `25%`, `50%`, ou `100%`
   - Escolha a ação: `deploy` ou `rollback`

2. **O que acontece internamente**:
```bash
# Exemplo: Escolheu 25% canary
CANARY_PCT=25
STABLE_PCT=75  # Calculado automaticamente (100 - 25)

# Istio ajusta o VirtualService
kubectl patch virtualservice fiap-todo-api --type merge -p '{
  "spec": {
    "http": [{
      "route": [
        {"destination": {"host": "fiap-todo-api", "subset": "v1"}, "weight": 75},
        {"destination": {"host": "fiap-todo-api", "subset": "v2"}, "weight": 25}
      ]
    }]
  }
}'
```

3. **Resultado**:
   - 75% das requisições → Stable (v1.0)
   - 25% das requisições → Canary (v2.0)
   - **Sem restart de pods!**
   - **Sem mudança no número de réplicas!**

**🔑 Vantagens desta abordagem**:
- ✅ Ajuste de tráfego via Istio (não depende de réplicas)
- ✅ Opções pré-definidas (10%, 25%, 50%, 100%)
- ✅ Ação de rollback integrada
- ✅ Sem downtime ou restart de pods
- ✅ Controle fino de tráfego

### Passo 19: Testar Pipeline Canary

**Commit e push:**
```bash
git add .github/workflows/canary-deploy.yml
git commit -m "feat: add canary deploy workflow with Istio"
git push origin main
```

**Executar no GitHub:**
1. Acesse: `Actions` → `🐤 Canary Deploy with Istio`
2. Clique em `Run workflow`
3. Selecione:
   - **Canary percentage**: `25`
   - **Action**: `deploy`
4. Clique em `Run workflow`

**Verificar resultado:**
```bash
# Ver distribuição de tráfego
export GATEWAY_URL=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Testar 20 vezes
for i in {1..20}; do 
  curl -s http://$GATEWAY_URL/health | jq -r '.version'
done | sort | uniq -c

# Resultado esperado com 25% canary:
# 15 v1.0  (75%)
#  5 v2.0  (25%)
```

**Aumentar para 50%:**
1. Execute workflow novamente
2. Selecione: **Canary percentage**: `50`
3. Teste novamente - deve ter ~50/50

**Rollback:**
1. Execute workflow
2. Selecione: **Action**: `rollback`
3. Volta para 100% stable instantaneamente!

---

## 🎓 Parte 7: Conceitos Aprendidos

### Passo 20: Matriz de Decisão

```mermaid
graph TB
    A[Escolher Estratégia] --> B{Tipo de Mudança?}
    
    B -->|Pequena| C[Rolling Update]
    B -->|Grande| D{Tem 2x recursos?}
    B -->|Crítica| E[Canary]
    
    D -->|Sim| F[Blue/Green]
    D -->|Não| E
    
    C --> G[Deploy gradual]
    F --> H[Switch instantâneo]
    E --> I[Teste com poucos users]
```

**Estratégias de deploy:**
- ✅ **Rolling Update**: Gradual, padrão K8s, baixo risco
- ✅ **Blue/Green**: Switch instantâneo, fácil rollback, requer 2x recursos
- ✅ **Canary**: Teste com % pequeno, validação real, baixo risco

---

## 🧹 Parte 8: Limpeza

### Passo 21: Limpar Recursos

```bash
# Deletar deployments
kubectl delete deployment --all

# Deletar services
kubectl delete service fiap-todo-api

# Deletar cluster (se não for usar mais)
aws eks delete-nodegroup \
  --cluster-name cicd-lab \
  --nodegroup-name workers \
  --region us-east-1 \
  --profile fiapaws

aws eks delete-cluster \
  --name cicd-lab \
  --region us-east-1 \
  --profile fiapaws
```

---

**FIM DO VÍDEO 3.3** ✅

**FIM DA AULA 3** 🎓
