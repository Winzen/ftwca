# Limpeza de Recursos Kubernetes — Prefect 0
**Data:** 2026-06-18

## Contexto

O Prefect 0 continua rodando no cluster mas não tem mais trabalho (schedules desativados em 2026-06-15). Os pods reservam muito mais CPU/memória do que realmente usam, impedindo o scheduler do K8s de alocar pods do Prefect 3 (`Pod is unschedulable`).

## Estado antes da limpeza

| Pod | CPU request | CPU real | Mem request | Mem real |
|---|---|---|---|---|
| prefect-agent | 100m | 1m | 128Mi | 72Mi |
| prefect-apollo × 2 | 600m | 5m | 512Mi | 158Mi |
| prefect-graphql × 2 | 200m | 10m | 512Mi | 135Mi |
| prefect-hasura | 500m | 14m | 1Gi | 161Mi |
| dbt-rpc-dev | sem request | 1m | sem request | 243Mi |
| dbt-rpc-prod | sem request | 1m | sem request | 243Mi |
| **Total reservado** | **~1.4 cores** | ~35m | **~2.1Gi** | ~1.1Gi |

## Ações executadas

| Ação | Motivo |
|---|---|
| `prefect-agent` → 0 réplicas | Schedules desativados — sem trabalho |
| `prefect-apollo` 2 → 1 réplica | Redundância desnecessária com carga zero |
| `prefect-graphql` 2 → 1 réplica | Idem |
| `prefect-hasura` requests: 500m/1Gi → 50m/256Mi | Uso real: 14m/161Mi |
| `prefect-apollo` requests: 300m/256Mi → 50m/128Mi | Uso real: ~3m/79Mi |
| `prefect-graphql` requests: 100m/256Mi → 50m/128Mi | Uso real: ~5m/68Mi |
| `dbt-rpc-dev` → 0 réplicas | Não é mais utilizado |
| `dbt-rpc-prod` → 0 réplicas | Não é mais utilizado |
| Todos os pods restantes: requests reduzidos ao mínimo absoluto (10m CPU) | Scheduler reserva mínimo; sem limits, pods usam o que precisam |

## Ganho total

| Recurso | Antes | Depois | Liberado |
|---|---|---|---|
| CPU (requests) | ~1.4 cores | **50m** | **~1.35 cores** |
| Memória (requests) | ~2.1Gi | **416Mi** | **~1.7Gi** |

## Estado final do namespace `prefect`

| Pod | Réplicas | CPU request | CPU real | Mem request | Mem real |
|---|---|---|---|---|---|
| prefect-apollo | 1 | 10m | 159m | 64Mi | 89Mi |
| prefect-graphql | 1 | 10m | 99m | 64Mi | 55Mi |
| prefect-hasura | 1 | 10m | 67m | 192Mi | 175Mi |
| prefect-towel | 1 | 10m | 0m | 80Mi | 58Mi |
| prefect-ui | 1 | 10m | 4m | 16Mi | 6Mi |
| dbt-rpc-dev | **0** | — | — | — | — |
| dbt-rpc-prod | **0** | — | — | — | — |
| prefect-agent | **0** | — | — | — | — |
| **Total** | | **50m** | | **416Mi** | |

## Namespaces dos agentes Prefect 0

Três namespaces adicionais identificados com o mesmo padrão (`prefect-agent` + `dbt-rpc-dev` + `dbt-rpc-prod`). Todos escalados para 0.

### Estado antes

| Namespace | Pod | Mem real |
|---|---|---|
| prefect-agent-basedosdados | prefect-agent + dbt-rpc-dev + dbt-rpc-prod | ~574Mi |
| prefect-agent-basedosdados-projetos | prefect-agent + dbt-rpc-dev + dbt-rpc-prod | ~231Mi |
| prefect-agent-basedosdados-perguntas | prefect-agent + dbt-rpc-dev + dbt-rpc-prod | ~231Mi |
| **Total** | 9 pods | **~1Gi** |

### Ações executadas

Nos três namespaces: `prefect-agent`, `dbt-rpc-dev` e `dbt-rpc-prod` → **0 réplicas**

### Estado final

Todos os namespaces de agentes estão vazios — sem pods rodando.

---

## Ganho total (namespace `prefect` + namespaces de agentes)

| Recurso | Antes | Depois | Liberado |
|---|---|---|---|
| CPU (requests) | ~1.4 cores | ~50m | **~1.35 cores** |
| Memória real | ~2.1Gi | ~370Mi | **~1.75Gi** |

---

## O que NÃO foi alterado

- Nenhum deployment foi deletado — podem ser reativados se necessário
- Nenhum `limits` foi adicionado — pods usam recursos além do request sem restrição
- Próximo passo: avaliar deletar os deployments zerados quando confirmado que não serão mais necessários
