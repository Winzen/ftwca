# Banco de Dados Prefect 3 — Conexão

## Onde fica

O Prefect 3 usa um banco PostgreSQL dedicado no Cloud SQL, separado do banco do Prefect 0.

| Campo | Valor |
|-------|-------|
| Host | `cloud-sql-proxy.cloud-sql-proxy.svc.cluster.local` |
| Port | `5432` |
| Database | `prefect3` |
| User | `prefect3` |
| Instância Cloud SQL | `basedosdados-dev:us-central1:basedosdados-dev-438172e0` |

> **Armadilha comum:** o banco `prefect` (mesmo host, mesma instância) é o banco do Prefect 0 e tem tabelas como `flow`, `flow_run`, `project`. O banco do Prefect 3 é o `prefect3` com tabelas como `deployment`, `deployment_schedule`, `work_pool`.

## Como obter as credenciais

As credenciais ficam no Kubernetes, namespace `prefect3`:

```bash
kubectl get secret prefect3-db-connection -n prefect3 \
  -o jsonpath='{.data.connection-string}' | base64 -d
```

Saída esperada (formato SQLAlchemy asyncpg):

```
postgresql+asyncpg://prefect3:<PASSWORD>@cloud-sql-proxy:5432/prefect3
```

## Tabelas principais do Prefect 3

| Tabela | Descrição |
|--------|-----------|
| `deployment` | Deployments registrados (nome, flow_id, paused, status) |
| `deployment_schedule` | Schedules vinculados a cada deployment (cron, active) |
| `flow` | Flows registrados |
| `flow_run` | Execuções de flows |
| `work_pool` | Work pools configurados |

O campo `schedule` em `deployment_schedule` é JSONB. Para extrair o cron:

```sql
schedule->>'cron'   -- ex: "0 6 * * *"
```

## Conectar ao Metabase

Use a API do Metabase (não a UI, para ter o `database_id` de volta):

```bash
TOKEN=$(curl -s -X POST "https://perguntas.basedosdados.org/api/session" \
  -H "Content-Type: application/json" \
  -d '{"username":"<email>","password":"<senha>"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

curl -s -X POST "https://perguntas.basedosdados.org/api/database" \
  -H "Content-Type: application/json" \
  -H "X-Metabase-Session: ${TOKEN}" \
  -d '{
    "engine": "postgres",
    "name": "Prefect 3",
    "details": {
      "host": "cloud-sql-proxy.cloud-sql-proxy.svc.cluster.local",
      "port": 5432,
      "dbname": "prefect3",
      "user": "prefect3",
      "password": "<PASSWORD>"
    }
  }'
```

Após criar, forçar sincronização do schema (o Metabase faz lazy-load das tabelas):

```bash
curl -s -X POST "https://perguntas.basedosdados.org/api/database/<DB_ID>/sync_schema" \
  -H "X-Metabase-Session: ${TOKEN}"
```

## Exemplo de query — flows agrupados por hora de schedule

Usada no card 1244 (`Flows com Schedule Agrupado por Hora`):

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

A expressão `SPLIT_PART(cron, ' ', 2)` extrai o segundo campo (hora UTC) do cron. O filtro `~ '^[0-9]+$'` exclui expressões complexas como `8,17` que não representam uma única hora.

## Contexto de descoberta

Identificado em 2026-07-15/16 ao consertar os cards Metabase que quebraram com a migração Prefect 0 → Prefect 3. O banco `prefect` na mesma instância Cloud SQL pertence ao Prefect 0 — conectar nele e ver tabelas como `flow_run` com `scheduled_start_time` é sinal de que você está no banco errado.
