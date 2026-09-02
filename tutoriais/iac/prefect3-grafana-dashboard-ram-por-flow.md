# Grafana — Dashboard RAM por Flow

**Status:** Implementado em 2026-07-24
**Objetivo:** Painéis no Grafana mostrando o pico médio de RAM por flow run do Prefect 3, separados por ambiente (Produção e Desenvolvimento)
**Dashboard:** `Base dos Dados / Pipelines`

---

## Resultado

Dois painéis de bar gauge no dashboard de Pipelines:
- **RAM por flow — Produção** — agrupa por deployment, mostra o pico máximo observado nos últimos 30 dias
- **RAM por flow — Desenvolvimento** — mesma lógica, namespace separado

Thresholds visuais: verde < 512Mi, amarelo < 1Gi, vermelho ≥ 1Gi.

---

## Arquitetura da solução

O Prometheus coleta memória de containers via cAdvisor (`container_memory_working_set_bytes`), mas as métricas do cAdvisor **não incluem labels de pod** — só labels de container (namespace, pod name, container name). O nome do pod no Prefect é aleatório (`exuberant-frog-nwjsz-r7tfg`), então sem informação adicional não há como saber qual deployment rodou.

A solução usa três peças:

```
kube-state-metrics  →  kube_pod_labels (com prefect.io/deployment-name)
        ↓
PrometheusRule (recording rule, avalia a cada 1min)
  join: container_memory_working_set_bytes × kube_pod_labels
  grava: prefect_job_memory_bytes{label_prefect_io_deployment_name=...}
        ↓
Grafana (bar gauge)
  avg by (label_prefect_io_deployment_name) (
    max_over_time(prefect_job_memory_bytes[30d])
  )
```

**Por que recording rule e não join direto no Grafana?**
Quando o pod de flow run termina, `kube_pod_labels` para de reportar aquele pod. Um join feito em tempo de query no Grafana não encontraria mais o pod e perderia os dados históricos. A recording rule grava o dado com o label enquanto o pod ainda existe — e o dado gravado persiste no TSDB pelo período de retenção.

---

## Componentes e arquivos

### 1. PrometheusRule

**Arquivo:** `k8s/observability/prometheus/prometheusrule-prefect-memory.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: prefect-flow-memory
  namespace: observability
  labels:
    release: prometheus
spec:
  groups:
  - name: prefect.flow.memory
    interval: 1m
    rules:
    - record: prefect_job_memory_bytes
      expr: |
        container_memory_working_set_bytes{namespace=~"prefect-worker-basedosdados.*", container="prefect-job"}
        * on(pod, namespace) group_left(label_prefect_io_deployment_name)
        kube_pod_labels{namespace=~"prefect-worker-basedosdados.*"}
```

Aplicar:
```bash
kubectl apply -f k8s/observability/prometheus/prometheusrule-prefect-memory.yaml
```

### 2. kube-state-metrics — allowlist de labels

Por padrão o kube-state-metrics v2+ **não expõe labels de pod**. Sem isso `kube_pod_labels` retorna vazio e o join falha silenciosamente (recording rule avalia sem erro mas produz 0 series).

**Arquivo:** `k8s/observability/prometheus/values.yaml`

```yaml
kube-state-metrics:
  extraArgs:
    - --metric-labels-allowlist=pods=[prefect.io/deployment-name]
```

Aplicar sem Helm upgrade (patch direto):
```bash
kubectl patch deployment -n observability prometheus-kube-state-metrics \
  --type json \
  -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--metric-labels-allowlist=pods=[prefect.io/deployment-name]"}]'
```

> ⚠️ O patch é sobrescrito no próximo `helm upgrade`. O `extraArgs` no `values.yaml` é o que garante persistência.

### 3. Work pools — label no pod template

O Prefect worker injeta `prefect.io/deployment-name` no pod automaticamente via `_base_deployment_labels` + `_propagate_labels_to_pod`. Para que isso funcione, o pod template precisa ter `metadata` inicializado — adicionamos `"labels": "{{ labels }}"` no `spec.template.metadata` dos dois work pools via API.

Aplicado via API do Prefect (não é arquivo — fica salvo nos work pools `basedosdados` e `basedosdados-dev`):

```python
import json, subprocess, base64

PREFECT_KEY = "..."  # ver pipelines/.env
BASE_URL = "https://prefect3.basedosdados.org/api"

for pool in ["basedosdados", "basedosdados-dev"]:
    data = json.loads(subprocess.check_output([
        "curl", "-s", f"{BASE_URL}/work_pools/{pool}",
        "-H", f"Authorization: Bearer {PREFECT_KEY}",
    ], text=True))
    bjt = data["base_job_template"]
    pod_template = bjt["job_configuration"]["job_manifest"]["spec"]["template"]
    pod_template.setdefault("metadata", {})["labels"] = "{{ labels }}"

    subprocess.check_output([
        "curl", "-s", "-o", "/dev/null",
        "-X", "PATCH", f"{BASE_URL}/work_pools/{pool}",
        "-H", f"Authorization: Bearer {PREFECT_KEY}",
        "-H", "Content-Type: application/json",
        "-d", json.dumps({"base_job_template": bjt})
    ], text=True)
    print(f"{pool}: ok")
```

> **Por que não `{{ deployment.name }}`?** A API do Prefect valida que toda variável `{{ }}` no job manifest esteja declarada no schema de `variables` do work pool. `deployment.name` não é uma variável do schema — é uma variável de contexto injetada pelo worker em runtime. A API rejeita com 422. A solução foi usar `{{ labels }}`, que É declarada no schema, e deixar o worker preencher via `_propagate_labels_to_pod`.

### 4. Grafana — painéis

**Arquivo:** `k8s/observability/prometheus/configmap.yaml` — ConfigMap `grafana-dashboard-pipelines`

Dois painéis adicionados ao final do dashboard:
- ID 7: `RAM por flow — Produção`, `namespace="prefect-worker-basedosdados"`, `y=24`
- ID 8: `RAM por flow — Desenvolvimento`, `namespace="prefect-worker-basedosdados-dev"`, `y=36`

Query (igual para os dois, só muda o namespace):
```promql
avg by (label_prefect_io_deployment_name) (
  max_over_time(
    prefect_job_memory_bytes{namespace="prefect-worker-basedosdados"}
  [30d])
)
```

Aplicar:
```bash
kubectl apply -f k8s/observability/prometheus/configmap.yaml
```

---

## Diagnóstico — como verificar se está funcionando

```bash
# Port-forward pro Prometheus
kubectl port-forward -n observability svc/prometheus-kube-prometheus-prometheus 19090:9090 &

# 1. kube_pod_labels tem o label de deployment?
curl -s "http://localhost:19090/api/v1/query" \
  --data-urlencode 'query=kube_pod_labels{namespace=~"prefect-worker-basedosdados.*"}' \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin)['data']['result']:
    print(r['metric'].get('pod'), '→', r['metric'].get('label_prefect_io_deployment_name', 'AUSENTE'))
"

# 2. Recording rule está produzindo dados?
curl -s "http://localhost:19090/api/v1/query" \
  --data-urlencode 'query=prefect_job_memory_bytes' \
  | python3 -c "
import sys, json
for r in json.load(sys.stdin)['data']['result']:
    mb = int(float(r['value'][1])) // 1024 // 1024
    print(r['metric'].get('label_prefect_io_deployment_name'), f'{mb}Mi')
"
```

**Se `kube_pod_labels` estiver vazio:** o kube-state-metrics não tem o `--metric-labels-allowlist` configurado. Verificar se o patch do deployment ainda está ativo:
```bash
kubectl get deployment -n observability prometheus-kube-state-metrics \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n' | grep allowlist
```

---

## Limitações conhecidas

- **Dados históricos:** o painel só mostra flows que rodaram **após** o kube-state-metrics ser configurado com o allowlist. Flows anteriores não aparecem.
- **Retenção curta:** com `retention: 10d` e sem PVC, dados são perdidos no restart do Prometheus. A query usa `[30d]` mas na prática é limitada ao que o TSDB tem.
- **PVC pendente:** os passos 2 e 3 do plano original (PVC 30Gi + retenção 60d) ainda não foram feitos — ver doc `Grafana — Dashboard RAM por Flow (Pendente).md` para os detalhes. Quando feito, o `[30d]` na query passa a ser real.
