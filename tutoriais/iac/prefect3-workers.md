# Prefect 3 — Workers (substitutos dos Agents)

## Novo conceito: Work Pools

No Prefect 1.x o agent sabia diretamente onde executar os flows (via labels).
No Prefect 3 existe uma camada intermediária chamada **Work Pool**:

```
Prefect Server
     ↓
  Work Pool  (define a infraestrutura: Kubernetes, Vertex AI...)
     ↓
   Worker    (processo que fica "ouvindo" o pool e executa os flows)
     ↓
  Flow Run   (Job Kubernetes ou job no Vertex AI)
```

---

## Comparativo: Agents atuais → Workers

| Agent atual | Namespace atual | Tipo | Work Pool Prefect 3 |
|---|---|---|---|
| `basedosdados` | `prefect-agent-basedosdados` | Kubernetes | `basedosdados` ✅ |
| `basedosdados-perguntas` | `prefect-agent-basedosdados-perguntas` | Kubernetes | consolidado em `basedosdados` ✅ |
| `basedosdados-projetos` | `prefect-agent-basedosdados-projetos` | Kubernetes | consolidado em `basedosdados` ✅ |
| `basedosdados-dev-gcp-vertex` | `prefect-agent-basedosdados-dev-gcp-vertex` | Vertex AI | `basedosdados-dev` ✅ (Kubernetes por ora) |

### ✅ Decisão tomada: consolidar em 2 workers

Após alinhamento com o Pisa: os 3 Kubernetes agents foram consolidados num único worker de produção (`basedosdados`) e um worker de dev (`basedosdados-dev`). Vertex AI não será usado neste momento — os flows de dev rodam em Kubernetes também.

---

## Passos para configurar os Workers

### 1. Criar os Work Pools no servidor

Via UI (acesso local via port-forward enquanto DNS não está configurado):
```bash
kubectl --namespace prefect3 port-forward svc/prefect-server 4200:4200
```
Acessa em `http://localhost:4200` → Work Pools → Create.

Tipos disponíveis:
- `kubernetes` — para os 3 agents Kubernetes
- `vertex-ai` — para o agent Vertex AI

### 2. Criar RBAC para os workers

Os workers precisam de permissão para criar e gerenciar Jobs no cluster.
Criar em cada namespace de worker:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "watch", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prefect-worker
  namespace: <namespace-do-worker>
subjects:
  - kind: ServiceAccount
    name: prefect-worker
roleRef:
  kind: Role
  name: prefect-worker
  apiGroup: rbac.authorization.k8s.io
```

### 3. Criar sealed secrets com credenciais GCP

Diferente do server, os workers precisam das credenciais GCP para:
- Criar Jobs no Kubernetes com acesso ao GCP
- Interagir com Vertex AI, BigQuery, GCS etc.

Os secrets necessários (mesmos usados nos agents atuais):
- `gcp-credentials` — `BASEDOSDADOS_CONFIG`, `BASEDOSDADOS_CREDENTIALS_PROD/STAGING`
- `vault-credentials` — `VAULT_ADDRESS`, `VAULT_TOKEN`
- `gcp-sa` — service account key (`creds.json`)
- `credentials-dev` / `credentials-prod` — JSON credentials

Criar via kubeseal (mesma abordagem do `secret-00_sealed.yaml` do server):
```bash
kubectl create secret generic gcp-credentials \
  --namespace <namespace-do-worker> \
  --from-literal=BASEDOSDADOS_CONFIG=... \
  --dry-run=client -o yaml | \
  kubeseal --namespace <namespace-do-worker> --format yaml > secret-01_sealed.yaml
```

### 4. Criar Helm chart values para cada worker

Usando o chart oficial `prefecthq/prefect-worker`:

```yaml
# k8s/prefect3_workers/<nome>/chart/values.yaml

fullnameOverride: "prefect-worker-<nome>"

worker:
  apiConfig: "server"
  config:
    workPool: "<nome-do-work-pool>"  # criado no passo 1

  serverApiConfig:
    apiUrl: "http://prefect-server.prefect3.svc.cluster.local:4200/api"

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

  extraEnvVarsSecret: "gcp-credentials"

  extraVolumes:
    - name: gcp-sa
      secret:
        secretName: gcp-sa

  extraVolumeMounts:
    - name: gcp-sa
      mountPath: /mnt/
      readOnly: true

  env:
    - name: GOOGLE_APPLICATION_CREDENTIALS
      value: /mnt/creds.json
```

### 5. Instalar os workers via Helm

```bash
helm repo add prefecthq https://prefecthq.github.io/prefect-helm
helm repo update
helm upgrade --install prefect-worker-<nome> -n <namespace-do-worker> prefecthq/prefect-worker -f k8s/prefect3_workers/<nome>/chart/values.yaml
```

### 6. Verificar conexão

Na UI do Prefect 3 (`http://localhost:4200`):
- Work Pools → o pool deve mostrar o worker como **online**

Via kubectl:
```bash
kubectl get pods -n <namespace-do-worker>
kubectl logs -n <namespace-do-worker> -l app.kubernetes.io/name=prefect-worker --tail=30
```

---

## Diferenças importantes para os flows

Os flows precisam ser atualizados para usar os novos work pools ao fazer deploy:

```python
# Prefect 3
from prefect import flow
from prefect.deployments import Deployment

@flow
def meu_flow():
    pass

Deployment.build_from_flow(
    flow=meu_flow,
    name="meu-deploy",
    work_pool_name="<nome-do-work-pool>",  # novo — substituiu labels
)
```

---

## Referências

- [Prefect 3 — Workers docs](https://docs.prefect.io/latest/deploy/infrastructure-concepts/workers/)
- [Prefect Helm chart — prefect-worker](https://github.com/PrefectHQ/prefect-helm/tree/main/charts/prefect-worker)
- [Prefect 3 — Vertex AI Worker](https://docs.prefect.io/latest/deploy/infrastructure-examples/serverless/)
