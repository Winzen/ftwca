# Loki — Redução de Cache e Configuração de Retenção
**Data:** 2026-06-18
**Branch IAC:** `feat/reduce-observability-resources`
**Arquivo alterado:** `k8s/observability/loki/values.yaml`

---

## Contexto

O Loki estava com dois problemas:

1. **Cache superdimensionado** — `loki-chunks-cache` e `loki-results-cache` são pods memcached cujos valores de `allocatedMemory` vinham dos defaults do chart (8192 MB e 1024 MB). O scheduler do K8s reserva esse espaço inteiro, mesmo com uso real mínimo.

2. **Sem retenção configurada** — sem `retention_period`, o Loki guarda logs indefinidamente. Com crescimento de ~10 GB/mês e PVC de 100 GB, o volume encheria em ~10 meses.

---

## Diagnóstico

### Uso real dos caches (7 dias via Prometheus)

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090 -n observability
```

| Cache | Mínimo | Média | Pico |
|---|---|---|---|
| `loki-chunks-cache` | 8Mi | 210Mi | 387Mi |
| `loki-results-cache` | 2Mi | 2Mi | 2Mi |

Query utilizada:
```promql
max_over_time(container_memory_working_set_bytes{namespace="observability",pod="loki-chunks-cache-0",container="memcached"}[7d])
max_over_time(container_memory_working_set_bytes{namespace="observability",pod="loki-results-cache-0",container="memcached"}[7d])
```

### Uso do PVC e taxa de crescimento

```bash
kubectl get pvc -n observability
kubectl exec -n observability loki-0 -- df -h /var/loki
kubectl exec -n observability loki-0 -- sh -c "du -sh /var/loki/*/"
```

| Métrica | Valor |
|---|---|
| Capacidade do PVC | 100 Gi |
| Uso atual | 35.9 GB (36%) |
| Idade do PVC | 111 dias |
| Taxa de crescimento | ~0.32 GB/dia (~10 GB/mês) |
| Quase tudo em | `/var/loki/chunks/` (35.9 GB) |

### Projeção de retenção

| Retenção | Storage em estado estável | Headroom |
|---|---|---|
| 3 meses | ~30 GB | 70 GB livre |
| **6 meses** | **~60 GB** | **40 GB livre** |
| 12 meses | ~120 GB | estoura o PVC |
| Sem retenção | — | enche em ~10 meses |

---

## Mudanças aplicadas

### `k8s/observability/loki/values.yaml`

```yaml
# Cache — valores baseados no pico real de 7 dias (chunks: 387Mi, results: 2Mi)
chunksCache:
  allocatedMemory: 512   # era 8192 (default do chart)
resultsCache:
  allocatedMemory: 64    # era 1024 (default do chart)

# Retenção — 6 meses baseado na taxa de crescimento de ~10 GB/mês
loki:
  limits_config:
    retention_period: 4320h  # 6 meses
  compactor:
    retention_enabled: true
    delete_request_store: filesystem
```

**Antes:** os valores de cache não estavam no `values.yaml` — vinham dos defaults do chart. Isso significa que um `helm upgrade` sem esses valores os redefiniria para os defaults novamente.

### Comando de aplicação

```bash
helm upgrade loki grafana/loki \
  -n observability \
  -f k8s/observability/loki/values.yaml
```

---

## Resultado

| Componente | Antes (reservado) | Depois (reservado) | Uso real |
|---|---|---|---|
| `loki-chunks-cache` | 8192 MB | **512 MB** | ~387Mi pico |
| `loki-results-cache` | 1024 MB | **64 MB** | ~2Mi constante |
| Retenção de logs | indefinida | **6 meses** | — |

**Memória liberada no scheduler:** ~8.5 GB de requests de memória devolvidos ao cluster.

---

## Verificação pós-upgrade

```bash
# Verificar pods
kubectl get pods -n observability

# Verificar uso real dos caches após estabilizar
kubectl top pods -n observability

# Verificar configuração aplicada
helm get values loki -n observability
```

---

## Próximos passos

- Monitorar o crescimento do PVC ao longo dos próximos meses para validar a estimativa de 10 GB/mês
- Se o uso do `loki-chunks-cache` consistentemente ficar abaixo de 300Mi, pode-se reduzir `allocatedMemory` para 384 MB
- Avaliar se o `resultsCache` pode ser desabilitado (`resultsCache.enabled: false`) dado o uso de apenas 2Mi
