# App Livros — GKE com Kubernetes

Aplicação de livros rodando no Google Kubernetes Engine (GKE). Usa frontend PHP com backend Redis em arquitetura primary/replica, com auto-scaling, network policies e alta disponibilidade.

## Arquitetura

```
                        ┌─────────────────────────────────┐
                        │         Namespace: livros        │
                        │                                  │
  Internet ──► LoadBalancer ──► Frontend (3 pods)         │
                        │           │                      │
                        │           ▼                      │
                        │    Redis Primary (1 pod)         │
                        │           │ replicação           │
                        │    Redis Replica (2 pods)        │
                        └─────────────────────────────────┘
```

## Pré-requisitos

- [Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Cluster GKE provisionado
- `kubectl` configurado para o cluster

## Deploy rápido (tudo de uma vez)

```bash
kubectl apply -f multi.yml
```

## Deploy por partes (ordem recomendada)

```bash
# 1. Namespace
kubectl apply -f namespace.yml

# 2. ConfigMap do Redis
kubectl apply -f configmap.yml

# 3. Redis Primary
kubectl apply -f redis-primary-deployment.yml
kubectl apply -f redis-primary-service.yml

# 4. Redis Replica
kubectl apply -f redis-replica-deployment.yml
kubectl apply -f redis-replica-service.yml

# 5. Frontend
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml

# 6. Auto-scaling
kubectl apply -f frontend-hpa.yml

# 7. Segurança de rede
kubectl apply -f network-policy.yml

# 8. Disponibilidade
kubectl apply -f pdb.yml

# 9. Ingress (opcional — requer nginx ingress controller)
kubectl apply -f nginx.yml
```

## Verificar status

```bash
# Listar todos os recursos no namespace
kubectl get all -n livros

# Verificar pods
kubectl get pods -n livros

# Verificar HPA
kubectl get hpa -n livros

# Obter IP externo do frontend
kubectl get service frontend -n livros --watch

# Ver logs do frontend
kubectl logs -l app=livros,tier=frontend -n livros

# Ver logs do Redis primary
kubectl logs -l app=livros,role=primary -n livros
```

## Estrutura dos arquivos

| Arquivo | Descrição |
|---------|-----------|
| `namespace.yml` | Namespace `livros` |
| `configmap.yml` | Configuração do Redis |
| `redis-primary-deployment.yml` | Deployment Redis primary (1 réplica) |
| `redis-primary-service.yml` | Service ClusterIP para Redis primary |
| `redis-replica-deployment.yml` | Deployment Redis replica (2 réplicas) |
| `redis-replica-service.yml` | Service ClusterIP para Redis replica |
| `frontend-deployment.yml` | Deployment frontend (3 réplicas) |
| `frontend-service.yml` | Service LoadBalancer — expõe porta 80 |
| `frontend-hpa.yml` | HorizontalPodAutoscaler (2–10 pods) |
| `network-policy.yml` | NetworkPolicies de isolamento |
| `pdb.yml` | PodDisruptionBudgets para alta disponibilidade |
| `nginx.yml` | Ingress NGINX (alternativa ao LoadBalancer) |
| `multi.yml` | Todos os manifests combinados em ordem |

## Recursos e limites

| Componente | CPU Request | CPU Limit | Mem Request | Mem Limit |
|-----------|-------------|-----------|-------------|-----------|
| Frontend | 100m | 250m | 100Mi | 256Mi |
| Redis Primary | 100m | 200m | 128Mi | 256Mi |
| Redis Replica | 100m | 200m | 128Mi | 256Mi |

## Auto-scaling (HPA)

O frontend escala automaticamente entre **2 e 10 pods** com base em:
- CPU acima de 70% de utilização média
- Memória acima de 80% de utilização média

## Ingress NGINX (opcional)

Se preferir usar Ingress ao invés de LoadBalancer, instale o controller primeiro:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
```

Depois edite `nginx.yml` e substitua `livros.example.com` pelo seu domínio, e aplique:

```bash
kubectl apply -f nginx.yml
```

## Remover tudo

```bash
kubectl delete namespace livros
```

## Criar cluster GKE (exemplo)

```bash
gcloud container clusters create app-livros \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=5

gcloud container clusters get-credentials app-livros --zone=us-central1-a
```

## Tecnologias

- **Kubernetes** 1.28+
- **Redis** 7.2 Alpine
- **Frontend** gcr.io/google-samples/gb-frontend:v5
- **Google Kubernetes Engine (GKE)**
