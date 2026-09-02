# Grafana — Atualização do Dashboard Pipelines

**Data:** 2026-07-16  
**Dashboard:** `Base dos Dados / Pipelines` (uid: `cdlzb0owwly4gd`)  
**Motivo:** Migração Prefect 0 → Prefect 3 mudou a arquitetura de pods e o dbt deixou de ser um serviço separado.

---

## Como o dashboard é gerenciado

O Grafana usa um **sidecar de dashboards** configurado no helm chart do `kube-prometheus-stack`. Qualquer ConfigMap no namespace `observability` com a label `grafana_dashboard: "1"` é carregado automaticamente — sem precisar acessar a UI do Grafana.

O dashboard de Pipelines fica em:

```
k8s/observability/prometheus/configmap.yaml
```

O arquivo contém vários ConfigMaps concatenados (datasources, backend, frontend, pipelines, metabase). O dashboard de Pipelines é o `grafana-dashboard-pipelines`.

Para aplicar mudanças:

```bash
kubectl apply -f k8s/observability/prometheus/configmap.yaml
```

O sidecar detecta a mudança e recarrega o dashboard em segundos. Não é necessário reiniciar o Grafana.

---

## Diagnóstico — o que havia de errado

As queries do dashboard usavam labels do Prefect 0 que não existem mais no Prefect 3:

| Painel | Query antiga | Motivo da falha |
|--------|-------------|-----------------|
| Prefect Server | `{app="prefect-server"}` | Label existia mas misturava logs do Prefect 0 e 3. Prefect 3 tem `loggingLevel: WARNING` — praticamente sem logs |
| Prefect Job | `{app="prefect-job"}` | Não existe no Prefect 3. Cada flow run vira um pod com `app=<dataset__tabela>` |
| Prefect Agent | `{app="prefect-agent"}` | Renomeado para `prefect-worker` no Prefect 3 |
| dbt production | `{app="dbt-rpc-prod"}` | Pod dedicado de dbt foi removido. dbt roda dentro do pod do flow run |
| dbt development | `{app="dbt-rpc-dev"}` | Idem |

### Como identificar as labels corretas

Para descobrir o que o Loki tem disponível, use o port-forward para o gateway:

```bash
kubectl port-forward -n observability svc/loki-gateway 13100:80 &

# Lista todos os labels disponíveis
curl -s http://localhost:13100/loki/api/v1/labels

# Lista valores de um label específico
curl -s http://localhost:13100/loki/api/v1/label/namespace/values
curl -s http://localhost:13100/loki/api/v1/label/app/values
```

No Prefect 3, o Loki mostrou:
- `namespace="prefect3"` → pod do servidor Prefect 3
- `namespace="prefect-worker-basedosdados"` → worker + todos os flow runs de prod (incluindo logs do dbt)
- `namespace="prefect-worker-basedosdados-dev"` → idem para dev
- `app="prefect-worker"` → processo worker em si

### Por que o painel do servidor foi trocado para métricas

O Prefect 3 server está configurado com `loggingLevel: WARNING`, então não emite logs em operação normal. Verificado via:

```bash
kubectl logs -n prefect3 <pod> --since=24h | wc -l  # retornou: 16 linhas (só o banner de startup)
```

A solução foi trocar o painel de logs por métricas de CPU e memória via Prometheus, que já coleta dados de todos os pods via cadvisor.

Queries usadas:
```
# CPU
rate(container_cpu_usage_seconds_total{namespace="prefect3", container="prefect-server"}[5m])

# Memória atual
container_memory_working_set_bytes{namespace="prefect3", container="prefect-server"}

# Limite de memória (para referência visual)
kube_pod_container_resource_limits{namespace="prefect3", container="prefect-server", resource="memory"}
```

> ⚠️ Em 2026-07-16, a memória do servidor estava em ~244Mi de um limite de 256Mi — próximo do teto. Considerar aumentar `server.resources.limits.memory` no helm values do prefect-server.

---

## Resultado final — novo layout

```
┌──────────────────────────┬──────────────────────────┐
│  Prefect Server — CPU    │  Prefect Server — Memória │
│  (timeseries, Prometheus)│  (timeseries, Prometheus) │
│  w=12                    │  w=12, mostra limite      │
├──────────────────────────┼──────────────────────────┤
│  Flow Runs — Produção    │  Flow Runs — Desenvolvimento│
│  (logs, Loki)            │  (logs, Loki)             │
│  namespace=prefect-worker│  namespace=prefect-worker  │
│  -basedosdados           │  -basedosdados-dev        │
│  h=16, w=12              │  h=16, w=12               │
└──────────────────────────┴──────────────────────────┘
```

Worker + flows + dbt estão todos no mesmo namespace do worker — os painéis de logs mostram tudo junto, refletindo que no Prefect 3 não há mais separação de processos.

---

## UIDs das datasources

Definidos no `values.yaml` do kube-prometheus-stack e no ConfigMap `grafana-datasources`:

| Datasource | UID a usar no JSON |
|------------|-------------------|
| Prometheus | `prometheus` |
| Loki | `lokiuid` |

Usar UIDs errados resulta em painel sem data sem mensagem de erro clara.

---

## Procedimento de atualização

1. Editar `k8s/observability/prometheus/configmap.yaml`
2. Incrementar `"version"` do dashboard (evita conflito com cache do Grafana)
3. Aplicar: `kubectl apply -f k8s/observability/prometheus/configmap.yaml`
4. Verificar no Grafana — o sidecar recarrega em ~30 segundos

Não é necessário acessar a UI do Grafana para editar dashboards. Toda mudança deve passar pelo arquivo no repositório para garantir que o estado é reproduzível.
