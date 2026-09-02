# Grafana — Dashboard de RAM por Flow (Pendente)

**Status:** Dashboard funcionando desde 2026-07-24 — ver doc completo em `Grafana — Dashboard RAM por Flow.md`. Este doc cobre apenas os passos pendentes de armazenamento (PVC + retenção).  
**Objetivo:** Painel no Grafana mostrando o consumo médio de RAM por flow run do Prefect 3  
**Custo estimado:** ~$1,20/mês (PVC 30Gi, pd-standard, us-central1)

---

## Por que não funciona agora

Dois bloqueadores identificados em 2026-07-16:

**1. Pods de flow run não têm label de deployment**  
O Prometheus coleta memória dos pods, mas sem nenhuma label que identifique qual flow rodou. O nome do pod é gerado aleatoriamente pelo Prefect (`native-jackal-x2pks-bcdth`). Sem label, é impossível agrupar memória por flow.

**2. Prometheus usa `emptyDir` sem PVC**  
Dados do TSDB vivem no disco efêmero do nó. Qualquer restart do pod apaga todo o histórico. A retenção atual é de 10 dias, insuficiente para flows mensais.

---

## Passo 1 — Adicionar label de deployment no job template ✅ (2026-07-24)

**Arquivo:** `k8s/prefect/job_template/job_template.yaml`

Adicionar `metadata.labels` no template do pod para que cada flow run pod herde o nome do deployment:

```yaml
# ANTES (estado atual):
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: flow
        ...

# DEPOIS:
apiVersion: batch/v1
kind: Job
spec:
  template:
    metadata:
      labels:
        deployment: "{{ '{{' }} deployment.name {{ '}}' }}"
    spec:
      containers:
      - name: flow
        ...
```

O Prefect 3 suporta variáveis Jinja2 no job template. `deployment.name` resolve para o nome do deployment (ex: `br_me_cnpj__estabelecimentos`).

**Arquivo editado em 2026-07-24.** Reaplicar o job template em cada work pool via Prefect 3 UI:
1. Acessar https://prefect3.basedosdados.org/work-pools
2. Para cada work pool (`basedosdados`, `basedosdados-dev`): editar → colar o job template atualizado → salvar

> ⚠️ O job template no repositório é a fonte de verdade, mas o Prefect 3 não lê o arquivo automaticamente — é necessário atualizar via UI ou API após cada mudança.

**Verificação:** Após o próximo flow run, confirmar no Prometheus:
```bash
kubectl port-forward -n observability svc/prometheus-kube-prometheus-prometheus 19090:9090 &
curl -s "http://localhost:19090/api/v1/query" \
  --data-urlencode 'query=container_memory_working_set_bytes{namespace="prefect-worker-basedosdados", container="prefect-job"}' \
  | python3 -c "import sys,json; [print(r['metric'].get('deployment','MISSING')) for r in json.load(sys.stdin)['data']['result']]"
```

---

## Passo 2 — Configurar PVC para o Prometheus

**Arquivo:** `k8s/observability/prometheus/values.yaml`, linha 3684

Substituir `storageSpec: {}` por:

```yaml
# ANTES:
storageSpec: {}

# DEPOIS:
storageSpec:
  volumeClaimTemplate:
    spec:
      storageClassName: standard
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 30Gi
```

> `standard` = pd-standard (HDD) em us-central1, $0.040/GB/mês → **$1,20/mês total**.  
> Se preferir SSD, usar `standard-rwo` (pd-balanced) = ~$0.10/GB/mês → $3,00/mês.

**Aplicar via helm:**
```bash
helm upgrade -i -n observability \
  -f k8s/observability/prometheus/values.yaml \
  prometheus-stack prometheus-community/kube-prometheus-stack
```

> ⚠️ O Prometheus StatefulSet vai ser recriado com o novo PVC. Haverá downtime breve (~1-2 min) e os dados do `emptyDir` atual serão perdidos (era efêmero de qualquer forma).

**Verificação:**
```bash
kubectl get pvc -n observability
# Deve aparecer um novo PVC: prometheus-prometheus-kube-prometheus-prometheus-db-*
```

---

## Passo 3 — Aumentar retenção

**Arquivo:** `k8s/observability/prometheus/values.yaml`, linha 3579

```yaml
# ANTES:
retention: 10d

# DEPOIS:
retention: 60d
```

Projeção de uso com 60 dias de retenção: ~22G (dentro do PVC de 30Gi com margem).

Aplicar junto com o Passo 2 no mesmo `helm upgrade`.

---

## PrometheusRule ✅ (2026-07-24)

**Arquivo:** `k8s/observability/prometheus/prometheusrule-prefect-memory.yaml`

Recording rule que faz o join `container_memory_working_set_bytes × kube_pod_labels` enquanto o pod está vivo, gravando `prefect_job_memory_bytes` com o label `label_prefect_io_deployment_name`. Os dados gravados persistem no TSDB mesmo após o pod ser deletado (dentro da janela de retenção).

```bash
kubectl apply -f k8s/observability/prometheus/prometheusrule-prefect-memory.yaml
```

---

## Passo 4 — Adicionar painel no Grafana ✅ (2026-07-24)

**Arquivo:** `k8s/observability/prometheus/configmap.yaml`

Painel adicionado ao dashboard `Base dos Dados / Pipelines` em 2026-07-24, antes do PVC estar configurado. O painel aparece no Grafana mas só mostra dados de flows rodados depois do passo 1 ser aplicado nos work pools. Query usada:

```promql
avg by (deployment) (
  max_over_time(
    container_memory_working_set_bytes{
      namespace="prefect-worker-basedosdados",
      container="prefect-job"
    }[30d]
  )
)
```

Configurações sugeridas para o painel:
- **Tipo:** `bar gauge` (facilita comparação entre flows)
- **Unidade:** `bytes` → Grafana converte para Mi/Gi automaticamente
- **Título:** `RAM média por flow (últimos 30 dias)`
- **Ordenação:** decrescente (flows mais pesados no topo)

Aplicar:
```bash
kubectl apply -f k8s/observability/prometheus/configmap.yaml
```

Configurações do painel adicionado:
- **Tipo:** `bargauge`, orientação horizontal
- **Unidade:** `bytes` (Grafana converte para Mi/Gi)
- **Cor:** gradiente azul-roxo, thresholds em 512Mi (amarelo) e 1Gi (vermelho)
- **Query:** instant query, `max` como reducer por série

---

## Ordem de execução recomendada

```
Passo 1 (job template + UI)       ← ✅ arquivo editado, falta aplicar nos work pools
    ↓
    aguardar 1-2 ciclos de flow runs para acumular dados com a nova label
    ↓
Passos 2 + 3 juntos (helm upgrade) ← pendente
    ↓
    aguardar acumular dados com PVC (idealmente 30+ dias)
    ↓
Passo 4 (painel Grafana)           ← ✅ painel adicionado, falta kubectl apply
```

---

## Referências de diagnóstico

```bash
# Verificar métricas disponíveis para flow run pods agora
kubectl port-forward -n observability svc/prometheus-kube-prometheus-prometheus 19090:9090 &
curl -s "http://localhost:19090/api/v1/query" \
  --data-urlencode 'query=max_over_time(container_memory_working_set_bytes{namespace="prefect-worker-basedosdados", container="prefect-job"}[1h])' \
  | python3 -c "
import sys,json
for r in json.load(sys.stdin)['data']['result']:
    m = r['metric']
    vals = r['values']
    peak = max(int(float(v[1])) for v in vals)
    print(f'deployment={m.get(\"deployment\",\"MISSING\")} peak={peak//1024//1024}Mi')
"
```

```bash
# Verificar retenção atual
kubectl get prometheus prometheus-kube-prometheus-prometheus -n observability \
  -o jsonpath='{.spec.retention}' && echo

# Verificar se PVC existe
kubectl get pvc -n observability
```
