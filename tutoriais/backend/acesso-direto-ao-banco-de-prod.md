# Acesso direto ao banco de prod (backend)

O banco de produção do backend roda no Cloud SQL e é acessível via `psql` dentro do pod `api-prod`, que já tem o cloud-sql-proxy como sidecar e a senha do banco como variável de ambiente.

---

## Passo a passo

### 0. Conta certa no gcloud (pré-requisito do `kubectl`)

`gcloud auth list` costuma mostrar a conta ativa (`*`) como `terraform@<projeto>.iam.gserviceaccount.com` — uma service account de infra, não a conta pessoal. Trocar antes de usar `kubectl`:

```bash
gcloud config set account <seu-email>
```

Se o token dela estiver expirado, o primeiro `kubectl` falha com `Reauthentication failed. cannot prompt during non-interactive execution.` — precisa de login interativo, não dá pra fazer numa sessão não-interativa:

```bash
gcloud auth login
```

**O cluster é compartilhado entre dev e prod.** Só existe um contexto de kubeconfig (`gke_<projeto>_<região>_<cluster>`) — apesar do nome, ele hospeda tanto dev quanto prod, isolados por **namespace** (`<namespace>` tem `api-prod-*`, `api-staging-*` e `api-development-*` juntos), não por cluster separado. Não precisa trocar de contexto.

### 1. Pegar o nome atual do pod

```bash
kubectl get pods -n <namespace> | grep api-prod
```

### 2. Abrir um psql interativo

```bash
kubectl exec -n <namespace> <nome-do-pod> -- sh -c \
  'psql "host=cloud-sql-proxy port=5432 user=api dbname=api sslmode=disable password=$DB_PASSWORD"'
```

### 3. Rodar um comando direto (sem abrir o shell interativo)

```bash
kubectl exec -n <namespace> <nome-do-pod> -- sh -c \
  'psql "host=cloud-sql-proxy port=5432 user=api dbname=api sslmode=disable password=$DB_PASSWORD" \
  -c "SEU SQL AQUI"'
```

> O `$DB_PASSWORD` já existe como variável de ambiente dentro do pod — não é necessário passar a senha manualmente.

---

## Exemplo: atualizar metadados de um dataset

Os metadados de datasets (datas de atualização, cobertura temporal) ficam em três tabelas:

| Tabela | O que armazena |
|---|---|
| `update` | Última atualização na BD e na fonte original |
| `poll` | Última verificação da fonte original |
| `datetime_range` | Cobertura temporal (start/end year/month/day) |

### Como encontrar os IDs via GraphQL

```bash
curl -s -X POST "https://backend.basedosdados.org/graphql" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  --data-raw '{"query":"{ allDataset(slug_Icontains: \"<slug>\") { edges { node { id slug tables { edges { node { id name updates { edges { node { id latest } } } coverages { edges { node { id datetimeRanges { edges { node { id startYear startMonth startDay endYear endMonth endDay } } } } } } } } } rawDataSources { edges { node { id name polls { edges { node { id latest } } } updates { edges { node { id latest } } } } } } } } } }"}' \
  | python3 -m json.tool
```

> O `slug` é o nome do dataset no backend (ex: `filiacao_partidaria`, não `br_tse_filiacao_partidaria`).

### SQL para atualizar as datas

```sql
-- Última atualização na BD ou na fonte original
UPDATE "update" SET latest = '2026-08-04 10:09:51+00' WHERE id = '<uuid>';

-- Última verificação da fonte original
UPDATE poll SET latest = '2026-08-04 23:45:01+00' WHERE id = '<uuid>';

-- Cobertura temporal (end date)
UPDATE datetime_range SET end_year = 2026, end_month = 8, end_day = 4 WHERE id = '<uuid>';
```

---

## Exemplo real: br_tse_filiacao_partidaria (2026-08-04)

```bash
kubectl exec -n <namespace> <nome-do-pod> -- sh -c \
  'psql "host=cloud-sql-proxy port=5432 user=api dbname=api sslmode=disable password=$DB_PASSWORD" -c "
    UPDATE \"update\" SET latest = '\''2026-08-04 10:09:51+00'\'' WHERE id = '\''0d1d2547-e606-4149-b01e-3686979f5302'\'';
    UPDATE \"update\" SET latest = '\''2026-08-04 23:45:01+00'\'' WHERE id = '\''92fb3c7d-e851-449e-9160-4521bbbeb94f'\'';
    UPDATE poll SET latest = '\''2026-08-04 23:45:01+00'\'' WHERE id = '\''a84686c6-9b0b-4206-a375-2c703f6f8226'\'';
    UPDATE datetime_range SET end_year = 2026, end_month = 8, end_day = 4 WHERE id = '\''d1bb1ae5-eaa1-4370-b820-cccb1efc5e27'\'';
  "'
```

---

## Atualização em lote (centenas de linhas de uma vez, via JOIN)

O exemplo acima resolve o `id` de cada `update`/`poll`/`datetime_range` via GraphQL antes de gravar — ótimo pra 1-10 registros, inviável pra uma correção em lote (ex.: chore #1861, 238 tabelas). Pra lote, não precisa resolver UUID nenhum: dá pra casar direto por `gcp_dataset_id`/`gcp_table_id` num `UPDATE ... FROM ... JOIN (VALUES ...)`, numa única chamada de `psql`.

Tabelas relevantes e como se ligam (`backend/apps/api/v1/models.py`):

| Model Django | `db_table` | Liga em |
|---|---|---|
| `Table` | `table` | — |
| `CloudTable` | `cloud_table` | `cloud_table.table_id → table.id`, tem `gcp_dataset_id`/`gcp_table_id` |
| `Update` | `update` | `update.table_id → table.id` |

```sql
UPDATE "update" u
SET latest = v.new_latest
FROM cloud_table ct
JOIN (VALUES
  ('br_ibge_ipca', 'mes_brasil', '2026-08-13T17:46:55.306+00:00'::timestamptz),
  ('br_stf_corte_aberta', 'decisoes', '2025-03-03T12:03:19.220+00:00'::timestamptz)
  -- ... uma linha por tabela a corrigir
) AS v(gcp_dataset_id, gcp_table_id, new_latest)
  ON ct.gcp_dataset_id = v.gcp_dataset_id AND ct.gcp_table_id = v.gcp_table_id
WHERE u.table_id = ct.table_id;
```

Isso vira uma única chamada de `kubectl exec -n <namespace> <nome-do-pod> -- sh -c 'psql "..." -c "UPDATE ..."'` (o SQL inteiro, com todas as linhas do `VALUES`, dentro do `-c`).

**Antes de rodar a versão que grava, sempre confira a resolução com um `SELECT`/`COUNT` equivalente:**

```sql
SELECT v.gcp_dataset_id, v.gcp_table_id, count(u.id)
FROM cloud_table ct
JOIN (VALUES ('br_ibge_ipca','mes_brasil') /* ... */) AS v(gcp_dataset_id, gcp_table_id)
  ON ct.gcp_dataset_id = v.gcp_dataset_id AND ct.gcp_table_id = v.gcp_table_id
LEFT JOIN "update" u ON u.table_id = ct.table_id
GROUP BY 1, 2
HAVING count(u.id) != 1;
```

Qualquer linha que aparecer nesse resultado tem 0 ou >1 `update` — **não inclua essas no `UPDATE` em lote**, eram pra ser tratadas à parte. (Achado real da chore #1861: de 238 tabelas, 236 resolveram certo e 2 deram `count == 2` — `br_inep_saeb.aluno_em_34ano` e `br_mg_belohorizonte_smfa_iptu.iptu`, provavelmente `update` duplicado pra mesma `table`, precisa investigar separado antes de corrigir.)

**Alternativa via Django ORM** (se preferir Python a SQL puro — mesma ideia, mais legível, mas precisa de `poetry run python manage.py shell` em vez de `psql` direto, já que o `python` puro do container não tem Django no `PYTHONPATH`):

```python
from django.db import transaction
from backend.apps.api.v1.models import Update

DATA = [("br_ibge_ipca", "mes_brasil", "2026-08-13T17:46:55.306000+00:00"), ...]

for dataset_id, table_id, new_latest in DATA:
    qs = Update.objects.filter(
        table__cloud_tables__gcp_dataset_id=dataset_id,
        table__cloud_tables__gcp_table_id=table_id,
    )
    if qs.count() != 1:
        continue  # ambíguo, pular — não adivinhar
    with transaction.atomic():
        obj = qs.first()
        obj.latest = new_latest
        obj.save(update_fields=["latest"])
```

Rodar um script assim via stdin (não digitar linha a linha):

```bash
kubectl exec -i -n <namespace> <nome-do-pod> -- sh -c "cd /app && poetry run python manage.py shell" < script.py
```

O `-i` é obrigatório pro shell receber o conteúdo do arquivo.

---

## Cuidados

- Sempre faça um `SELECT` antes do `UPDATE` para confirmar que está editando o registro certo
- O pod `api-prod` muda de nome a cada deploy — verifique com `kubectl get pods -n <namespace> | grep api-prod` antes de usar
- O banco de prod não tem rollback fácil — confira os IDs via GraphQL antes de executar
