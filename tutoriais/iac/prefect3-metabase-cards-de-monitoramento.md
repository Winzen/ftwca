# Metabase — Cards de Monitoramento Prefect 3

**Coleção:** [Central de Acompanhamento - Equipe Dados](https://perguntas.basedosdados.org/collection/215)
**Última revisão:** 2026-07-16

---

## Cards ativos (Prefect 3)

Todos consultam `admin_data_tools_disabledflowschedule` no Django backend (db 133) ou o banco PostgreSQL do Prefect 3 (db 168).

### Visão geral de schedules

#### [1243 — Número de Flows com Schedule Geral](https://perguntas.basedosdados.org/question/1243)

**Tipo:** Escalar (4 métricas)  
**Fonte:** db 133 (`admin_data_tools_disabledflowschedule`)  
**O que mostra:** Contagens totais de deployments ativos, desativados, total e percentual ativo.

```sql
SELECT
  COUNT(*) FILTER (WHERE is_schedule_active = TRUE)  AS ativos,
  COUNT(*) FILTER (WHERE is_schedule_active = FALSE) AS desativados,
  COUNT(*) AS total,
  ROUND(100.0 * COUNT(*) FILTER (WHERE is_schedule_active = TRUE) / NULLIF(COUNT(*), 0), 2) AS percentual_ativo
FROM public.admin_data_tools_disabledflowschedule
```

---

#### [1244 — Flows com Schedule Agrupado por Hora](https://perguntas.basedosdados.org/question/1244)

**Tipo:** Tabela / Gráfico de barras  
**Fonte:** db 168 (`deployment_schedule` do Prefect 3)  
**O que mostra:** Quantos flows têm schedule ativo em cada hora UTC. Extrai a hora do campo `schedule->>'cron'` (formato `minuto hora * * *`). Exclui expressões com múltiplas horas (ex: `8,17`).

```sql
SELECT
  SPLIT_PART(schedule->>'cron', ' ', 2) AS hora,
  COUNT(*) AS total_flows
FROM deployment_schedule
WHERE active = TRUE
  AND schedule->>'cron' IS NOT NULL
  AND SPLIT_PART(schedule->>'cron', ' ', 2) ~ '^[0-9]+$'
GROUP BY 1
ORDER BY SPLIT_PART(schedule->>'cron', ' ', 2)::int
```

---

### Flows ativos

#### [1239 — Flows com Schedule Ativo](https://perguntas.basedosdados.org/question/1239)

**Tipo:** Tabela  
**Fonte:** db 133  
**O que mostra:** Lista completa de deployments com schedule ativo, ordenada por nome, com link direto para o deployment no Prefect 3.

```sql
SELECT id, flow_name AS name, is_schedule_active,
  'https://prefect3.basedosdados.org/deployments/' || deployment_id AS link
FROM public.admin_data_tools_disabledflowschedule
WHERE is_schedule_active = TRUE
ORDER BY flow_name
```

---

### Flows desativados

#### [1240 — Flows com Schedule Desativados](https://perguntas.basedosdados.org/question/1240)

**Tipo:** Tabela  
**Fonte:** db 133  
**O que mostra:** Lista de deployments com schedule desativado, ordenada do mais recente para o mais antigo.

```sql
SELECT id, flow_name AS name, is_schedule_active, disabled_at, reactivated_at,
  'https://prefect3.basedosdados.org/deployments/' || deployment_id AS link
FROM public.admin_data_tools_disabledflowschedule
WHERE is_schedule_active = FALSE
ORDER BY disabled_at DESC
LIMIT 1048575
```

---

#### [1248 — Flows Desativados Pela Task — Disabled Flow Schedules](https://perguntas.basedosdados.org/question/1248)

**Tipo:** Tabela  
**Fonte:** db 133  
**O que mostra:** Similar ao 1240. Inclui `reactivated_at` explicitamente. Pensado como view de auditoria da task `disable_flow_schedule` do backend.

```sql
SELECT
  f."id", f."flow_name" AS "name", f."is_schedule_active",
  'https://prefect3.basedosdados.org/deployments/' || f."deployment_id" AS link,
  f."disabled_at", f."reactivated_at"
FROM public.admin_data_tools_disabledflowschedule f
WHERE is_schedule_active = False
LIMIT 1048575
```

---

#### [1250 — Deployments pausados há mais de 30 dias](https://perguntas.basedosdados.org/question/1250)

**Tipo:** Tabela  
**Fonte:** db 133  
**O que mostra:** Alerta de flows "esquecidos" — desativados há mais de 30 dias e que nunca foram reativados. Ordenado do mais antigo para o mais recente.

```sql
SELECT
  flow_name AS name,
  disabled_at,
  DATE_PART('day', NOW() - disabled_at)::int AS dias_pausado,
  'https://prefect3.basedosdados.org/deployments/' || deployment_id AS link
FROM public.admin_data_tools_disabledflowschedule
WHERE is_schedule_active = FALSE
  AND disabled_at < NOW() - INTERVAL '30 days'
ORDER BY disabled_at ASC
```

---

### Tendências e saúde operacional

#### [1251 — Histórico de pausas e reativações por mês](https://perguntas.basedosdados.org/question/1251)

**Tipo:** Gráfico de barras (duas séries: `desativacoes` e `reativacoes`)  
**Fonte:** db 133  
**O que mostra:** Volume de desativações e reativações de schedules por mês. Útil para ver picos (ex: migração Prefect 0 → 3 em 2026-07 gerou 186 desativações).

```sql
SELECT mes, SUM(desativacoes) AS desativacoes, SUM(reativacoes) AS reativacoes
FROM (
  SELECT TO_CHAR(disabled_at, 'YYYY-MM') AS mes, COUNT(*) AS desativacoes, 0 AS reativacoes
  FROM public.admin_data_tools_disabledflowschedule WHERE disabled_at IS NOT NULL GROUP BY 1
  UNION ALL
  SELECT TO_CHAR(reactivated_at, 'YYYY-MM') AS mes, 0 AS desativacoes, COUNT(*) AS reativacoes
  FROM public.admin_data_tools_disabledflowschedule WHERE reactivated_at IS NOT NULL GROUP BY 1
) t
GROUP BY mes ORDER BY mes
```

---

#### [1252 — Runs mais longos por flow (últimos 30 dias)](https://perguntas.basedosdados.org/question/1252)

**Tipo:** Tabela  
**Fonte:** db 168 (Prefect 3 postgres)  
**O que mostra:** Ranking dos flows com maior duração média de execução nos últimos 30 dias. Só inclui deployments com ≥ 3 runs concluídos (filtra outliers de amostra única).

```sql
SELECT
  d.name,
  ROUND(AVG(EXTRACT(EPOCH FROM fr.total_run_time)/60)::numeric, 1) AS media_min,
  MAX(EXTRACT(EPOCH FROM fr.total_run_time)/60)::int AS max_min,
  COUNT(*) AS runs
FROM deployment d
JOIN flow_run fr ON fr.deployment_id = d.id
WHERE fr.state_name = 'Completed'
  AND fr.created > NOW() - INTERVAL '30 days'
GROUP BY d.name HAVING COUNT(*) >= 3
ORDER BY media_min DESC LIMIT 20
```

---

## Cards obsoletos (Prefect 0 — db 167)

Estes cards ainda existem na coleção mas consultam o banco Prefect 0 (`db 167`), cujos dados não são mais atualizados. Podem ser removidos.

| ID | Nome | Equivalente no Prefect 3 |
|----|------|--------------------------|
| [1241](https://perguntas.basedosdados.org/question/1241) | Numero de Flows com Schedule Ativo | → 1243 (campo `ativos`) |
| [1242](https://perguntas.basedosdados.org/question/1242) | Numero de Flows com Schedule Desativado | → 1243 (campo `desativados`) |
| [1236](https://perguntas.basedosdados.org/question/1236) | Contagem de flows quebrados por nome | sem equivalente atual |
| [1247](https://perguntas.basedosdados.org/question/1247) | Flows sem Schedule Ativo (últimos 5 meses) | sem equivalente atual |

---

## Fontes de dados

| DB ID | Nome no Metabase | O que contém |
|-------|-----------------|--------------|
| 133 | Metadados-PROD | Django backend — `admin_data_tools_disabledflowschedule` com histórico de pausas/reativações gerenciado pela task `disable_flow_schedule` |
| 168 | Prefect 3 | PostgreSQL do Prefect 3 — `deployment`, `deployment_schedule`, `flow_run`. Credenciais em `kubectl get secret prefect3-db-connection -n prefect3` |
| 167 | Metabase-Prefect *(obsoleto)* | PostgreSQL do Prefect 0 — dados não atualizados desde a migração |

Ver detalhes de conexão: [[Banco de Dados Prefect 3 — Conexão]]
