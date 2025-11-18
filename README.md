# 📚 Aula 03 - Docker e Kubernetes

## 🎯 Objetivos

- Criar pipelines para build e publicação de imagens Docker
- Publicar imagens em Container Registry (AWS ECR)
- Automatizar deploys no Kubernetes com Kustomize
- Implementar estratégias avançadas de deploy (Blue/Green e Canary)
- Configurar health checks e probes no Kubernetes

## 📹 Vídeos

| Vídeo | Título | Temas |
|-------|--------|-------|
| 3.1 | Build e Publicação de Imagens Docker | Multi-stage build; ECR; OIDC; Pipeline Docker |
| 3.2 | Deploy com Automação de Manifestos | EKS; Kustomize; Helm; Manifests; Overlays |
| 3.3 | Estratégias de Deploy Avançadas | Blue/Green; Canary; Rolling Update; Rollback |

## 🚀 Como Usar

### 1. Fork e Clone

```bash
git clone https://github.com/josenetoo/fiap-dclt-aula03.git
cd fiap-dclt-aula03
```

### 2. Seguir Vídeos em Ordem

- [VIDEO-3.1-PASSO-A-PASSO.md](VIDEO-3.1-PASSO-A-PASSO.md) - Docker Build e ECR
- [VIDEO-3.2-PASSO-A-PASSO.md](VIDEO-3.2-PASSO-A-PASSO.md) - Kubernetes e Kustomize
- [VIDEO-3.3-PASSO-A-PASSO.md](VIDEO-3.3-PASSO-A-PASSO.md) - Estratégias Avançadas

## 📁 Estrutura do Projeto

```
aula-03/
├── app/
│   ├── src/
│   │   ├── app.js              # API Express
│   │   └── server.js           # Entry point
│   └── package.json
├── k8s/
│   ├── base/                  # Manifests base
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── overlays/
│   │   └── production/        # Overlay de produção
│   └── blue-green/            # Manifests Blue/Green
├── Dockerfile                 # Multi-stage build
├── VIDEO-3.1-PASSO-A-PASSO.md
├── VIDEO-3.2-PASSO-A-PASSO.md
├── VIDEO-3.3-PASSO-A-PASSO.md
└── README.md
```

## ✅ Checklist de Aprendizado

### Vídeo 3.1
- [ ] Entender Dockerfile multi-stage
- [ ] Configurar AWS ECR
- [ ] Configurar GitHub Secrets com credenciais AWS
- [ ] Criar pipeline de build Docker
- [ ] Configurar scan de vulnerabilidades

### Vídeo 3.2
- [ ] Criar cluster EKS (cicd-lab) com AWS CLI
- [ ] Entender Deployments e Services
- [ ] Configurar Kustomize com overlays
- [ ] Implementar pipeline de deploy
- [ ] Configurar health checks

### Vídeo 3.3
- [ ] Implementar Blue/Green deploy
- [ ] Implementar Canary deploy
- [ ] Configurar Rolling Update
- [ ] Testar rollback de deploys
- [ ] Entender quando usar cada estratégia

## 🐛 Troubleshooting

### Erro: "ImagePullBackOff"
- **Causa**: Kubernetes não consegue baixar a imagem do ECR
- **Solução**: Verificar permissões IAM e URI da imagem

### Erro: "CrashLoopBackOff"
- **Causa**: Container inicia e falha repetidamente
- **Solução**: Verificar logs com `kubectl logs <pod-name>`

### Erro: "Insufficient CPU/Memory"
- **Causa**: Nós do cluster sem recursos suficientes
- **Solução**: Escalar cluster ou reduzir requests

### Erro: "OIDC Authentication Failed"
- **Causa**: Configuração incorreta do OIDC Provider
- **Solução**: Verificar trust policy e thumbprint

## 📚 Recursos Adicionais

- [Documentação Kubernetes](https://kubernetes.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

## ⚠️ Importante

- **AWS Learner Lab**: Usar sempre `--profile fiapaws` nos comandos AWS CLI
- **Cluster**: Nome do cluster: `cicd-lab`
- **Limitações**: Máximo 9 instâncias EC2 e 32 vCPU concorrentes
- **Instance Types**: Apenas nano, micro, small, medium, large
- **Regiões**: us-east-1 ou us-west-2
- **Credenciais**: Usar GitHub Secrets para armazenar AWS Access Keys
- **Sessão**: Renovar credenciais quando a sessão do Learner Lab expirar
- **Limpeza**: Sempre deletar recursos após a aula para preservar o budget
- **Secrets**: Nunca commitar credenciais no código
