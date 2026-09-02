# BigQuery — Quota Diária no Grafana

## Problema

O erro `Your usage exceeded the custom quota for QueryUsagePerDay, which is set by your administrator` ocorre quando o total de bytes processados por queries BigQuery no dia ultrapassa o limite configurado. Atualmente não há visibilidade dessa métrica no Grafana — o estouro só é descoberto quando o erro aparece.

## Análise de opções

### Opção A — Cloud Monitoring datasource no Grafana
GCP exporta quota usage automaticamente para o Cloud Monitoring. A métrica seria `serviceruntime.googleapis.com/quota/rate/net_usage` filtrada por `quota_metric = "bigquery.googleapis.com/quota/query/usage"`.

**Problema:** requer instalar o plugin `grafana-googlecloudmonitoring-datasource` e adicionar um novo datasource — dependência de plugin externo não gerenciado hoje.

### Opção B — CronJob + Prometheus Pushgateway
CronJob que queries INFORMATION_SCHEMA e empurra resultado para um Pushgateway que o Prometheus raspa.

**Problema:** exige dois componentes novos (CronJob + Pushgateway) para uma métrica só.

### Opção C — Exporter Python como Deployment ✅ Recomendado
Um `Deployment` leve na namespace `observability` que expõe `/metrics` com o total de bytes processados no dia. O Prometheus já sabe fazer scrape por padrão. O exporter faz a query no `INFORMATION_SCHEMA.JOBS_BY_PROJECT` com cache de 15 minutos para não bater no BigQuery em toda raspagem.

**Vantagem:** usa infraestrutura existente (Prometheus + Grafana), sem plugins externos, sem Pushgateway.

## Implementação planejada

### Fonte de dados
```sql
SELECT SUM(total_bytes_processed)
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE DATE(creation_time, "America/Sao_Paulo") = CURRENT_DATE("America/Sao_Paulo")
  AND state = "DONE"
```

A métrica `total_bytes_processed` de `JOBS_BY_PROJECT` é exatamente o que o BigQuery contabiliza para a cota `QueryUsagePerDay`.

### Onde vive no IAC
```
k8s/observability/
  bq-quota-exporter/
    deployment.yaml      # exporter Python (prometheus_client)
    service.yaml         # expõe :8080/metrics para o Prometheus scrape
    serviceaccount.yaml  # SA com Workload Identity → roles/bigquery.resourceViewer
```

### Painel no Grafana
- Gauge: bytes processados hoje vs. limite da cota
- Série temporal: consumo ao longo do dia (acumulado)
- Linha de referência no limite da cota para referência visual

## Status

- [ ] Implementar o exporter (`k8s/observability/bq-quota-exporter/`)
- [ ] Configurar Workload Identity para o ServiceAccount ter acesso ao INFORMATION_SCHEMA
- [ ] Adicionar scrape config no Prometheus values.yaml
- [ ] Criar painel no Grafana (dashboard Pipelines ou novo dashboard BQ)

## Referências

- [INFORMATION\_SCHEMA.JOBS\_BY\_PROJECT](https://cloud.google.com/bigquery/docs/information-schema-jobs)
- [Cotas do BigQuery](https://cloud.google.com/bigquery/quotas)
- Dashboards existentes: `IAC/Observability/Dashboards.md`
