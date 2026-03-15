# Brokerone Fechamento Dispatcher — Ambiente Local

Este guia descreve como executar o **dispatcher Kafka de fechamento** em ambiente local utilizando:

* **Docker** → build da imagem da aplicação Laravel
* **Minikube** → cluster Kubernetes local
* **Argo CD** → gerenciamento de deploy via GitOps

Arquitetura do ambiente local:

```
Docker → build da imagem
      ↓
Minikube (Kubernetes cluster local)
      ↓
Argo CD (deploy e sincronização)
      ↓
Dispatcher Kafka (Deployment Kubernetes)
      ↓
Criação dinâmica de Jobs/Pods
```

---

# 1. Pré-requisitos

Instale:

* Docker
* kubectl
* Minikube
* Git
* (opcional) CLI do Argo CD

Verifique:

```bash
docker --version
kubectl version --client
minikube version
```

---

# 2. Estrutura do projeto Kubernetes

Dentro da pasta `brokerone-fechamento-dispatcher`:

```
brokerone-fechamento-dispatcher/
│
├─ dispatcher-deployment.yaml
├─ dispatcher-rbac.yaml
├─ dispatcher-sa.yaml
├─ namespace.yaml
├─ kustomization.yaml
├─ application.yaml
└─ .env
```

Descrição:

| Arquivo                    | Função                         |
| -------------------------- | ------------------------------ |
| dispatcher-deployment.yaml | Deployment do dispatcher       |
| dispatcher-sa.yaml         | ServiceAccount                 |
| dispatcher-rbac.yaml       | Role + RoleBinding             |
| namespace.yaml             | Cria namespace `local`         |
| kustomization.yaml         | Agrupa os manifests            |
| application.yaml           | Aplicação gerenciada pelo Argo |
| .env                       | Variáveis da aplicação         |

---

# 3. Subir o cluster Minikube

```bash
minikube start
```

Verifique:

```bash
kubectl get nodes
```

Resultado esperado:

```
minikube   Ready
```

---

# 4. Usar Docker interno do Minikube

Isso permite buildar imagens diretamente no cluster local.

PowerShell:

```powershell
minikube -p minikube docker-env --shell powershell | Invoke-Expression
```

---

# 5. Build da imagem da aplicação

```powershell
docker build -f C:\Projects\broker-app\Dockerfile-kafka-dispatcher-dev -t broker-app-worker-base:local C:\Projects\broker-app
```

Imagem criada:

```
broker-app-worker-base:local
```

---

# 6. Criar namespace da aplicação

```bash
kubectl create namespace local
```

Caso já exista:

```bash
kubectl get namespace local
```

---

# 7. Criar Secret com variáveis de ambiente

O dispatcher utiliza variáveis de ambiente provenientes do `.env`.

```powershell
kubectl create secret generic brokerone-fechamento-dispatcher-keys-local \
--from-env-file=C:\Users\brendon.carvalho\Downloads\brokerone-fechamento-dispatcher\.env \
-n local
```

Caso precise recriar:

```powershell
kubectl delete secret brokerone-fechamento-dispatcher-keys-local -n local
kubectl create secret generic brokerone-fechamento-dispatcher-keys-local \
--from-env-file=.env \
-n local
```

---

# 8. Deploy manual (sem Argo)

Para deploy tradicional:

```powershell
kubectl apply -f dispatcher-sa.yaml
kubectl apply -f dispatcher-rbac.yaml
kubectl apply -f dispatcher-deployment.yaml
```

Verificar:

```bash
kubectl get pods -n local
```

---

# 9. Instalar Argo CD no Minikube

Criar namespace:

```bash
kubectl create namespace argocd
```

Instalar Argo:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verificar:

```bash
kubectl get pods -n argocd
```

Todos os pods devem ficar:

```
Running
```

---

# 10. Acessar interface do Argo CD

Executar:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir navegador:

```
https://localhost:8080
```

Login:

```
User: admin
Password: <senha inicial>
```

Senha inicial pode ser obtida com:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

---

# 11. Configuração GitOps

O Argo CD gerencia deploys a partir de um repositório Git.

Arquivo `application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: brokerone-fechamento-dispatcher-local
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/SEU-USUARIO/SEU-REPO.git
    targetRevision: main
    path: brokerone-fechamento-dispatcher
  destination:
    server: https://kubernetes.default.svc
    namespace: local
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Aplicar:

```bash
kubectl apply -f application.yaml -n argocd
```

---

# 12. Estrutura Kustomize

Arquivo `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: local

resources:
  - namespace.yaml
  - dispatcher-sa.yaml
  - dispatcher-rbac.yaml
  - dispatcher-deployment.yaml
```

Isso permite ao Argo aplicar todos os manifests juntos.

---

# 13. Fluxo completo de desenvolvimento

### Build da aplicação

```
minikube start
minikube docker-env
docker build
```

### Preparação do ambiente

```
kubectl create namespace local
kubectl create secret from .env
```

### Deploy

Via Argo:

```
kubectl apply application.yaml
```

ou manual:

```
kubectl apply deployment
```

---

# 14. Verificar pods

Dispatcher:

```bash
kubectl get pods -n local
```

Argo:

```bash
kubectl get pods -n argocd
```

---

# 15. Atualizar imagem

Sempre que mudar código:

```
docker build
kubectl rollout restart deployment brokerone-fechamento-dispatcher-local -n local
```

Ou atualizar tag da imagem no Git para Argo sincronizar.

---

# 16. Arquitetura final

```
┌──────────────┐
│    Docker    │
│ build image  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Minikube   │
│ Kubernetes   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Argo CD    │
│ GitOps sync  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Dispatcher  │
│ Deployment   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Kafka Worker │
│ Jobs/Pods    │
└──────────────┘
```

---

# 17. Observações

* Secrets continuam sendo criados manualmente a partir do `.env`.
* Em produção recomenda-se usar:

  * External Secrets
  * Vault
  * Sealed Secrets
* Argo CD gerencia apenas os manifests versionados no Git.

---

# 18. Comandos úteis

Ver logs:

```bash
kubectl logs -n local <pod>
```

Ver eventos:

```bash
kubectl get events -n local
```

Reiniciar deployment:

```bash
kubectl rollout restart deployment brokerone-fechamento-dispatcher-local -n local
```

Abrir dashboard do Minikube:

```bash
minikube dashboard
```

---

# 19. Resultado esperado

Ao final:

* Minikube rodando cluster Kubernetes
* Argo CD instalado e acessível
* Dispatcher rodando no namespace `local`
* Deploy controlado via GitOps

---
