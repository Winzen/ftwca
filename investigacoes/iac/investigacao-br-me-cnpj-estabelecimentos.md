# Investigação — `br_me_cnpj.estabelecimentos`: dado em prod sem metadado atualizado

**Data:** 2026-07-15
**Tabela:** `basedosdados.br_me_cnpj.estabelecimentos`

---

## Gatilho

Comparação entre dado em prod e metadado da API revelou divergência de 2 dias:

| Fonte | Data mais recente |
|---|---|
| BigQuery prod (`basedosdados.br_me_cnpj.estabelecimentos`) | 2026-07-12 (partição `20260712`) |
| API de metadados (`allDatetimerange`) | 2026-07-10 |

53 fotografias distintas no BigQuery, atualização mensal. A cobertura mais recente na API terminava em 10/07 — 2 dias antes da última fotografia real.

---

## Investigação

### 1. Flow runs de julho 15 para `br_me_cnpj__estabelecimentos`

| Flow run | Pool | Estado | Target | `materialize_after_dump` | Duração |
|---|---|---|---|---|---|
| `meteoric-albatross` | basedosdados | Failed | — (vazio) | — | 88 min |
| `mu546-pernaia-anomaly` | basedosdados-dev | Failed | dev | false | 78 min |
| `xi-yalow-z` | basedosdados | Completed | prod | true | 0 s (poll) |
| **`daffy-raven`** | basedosdados | **Completed** | — | true | **15 min** |

Runs de 0 segundo são polls: o flow checou se havia dado novo e retornou sem fazer nada.

### 2. Logs do `meteoric-albatross` (flow que falhou)

Task `run_dbt-bb9` (`br_me_cnpj__estabelecimentos`, target=dev):

```
12:11:49  dbt run --select models/br_me_cnpj/br_me_cnpj__estabelecimentos.sql --target dev
12:16:32  dbt run OK                                ← dados gravados em dev
12:16:32  dbt test ...
12:17:24  Database Error: Unrecognized name: ano at [14:11]   ← teste falha
12:17:24  Database Error: Unrecognized name: mes at [14:11]   ← teste falha
12:17:28  dbt test falhou — Retry 1/1 will start in 10s      ← retry dispara
12:17:38  dbt run ... (segunda tentativa)
12:21:13  Custom quota exceeded: QueryUsagePerUserPerDay      ← cota estourada
12:21:18  dbt run falhou — Retries exhausted
```

**Causa raiz do `meteoric-albatross`:** dois problemas encadeados:
1. O `dbt test` falhou porque os testes de `relationships` em `schema.yml` referenciam as colunas `ano` e `mes` de forma incorreta no SQL compilado — `Unrecognized name` no BigQuery.
2. O `retries=1` (código antigo) disparou um segundo `dbt run` que estourou a cota diária de query do BigQuery. O PR [#1667](https://github.com/basedosdados/pipelines/pull/1667) remove o retry (`retries=0`), mas o bug do teste precisa ser corrigido separadamente no `schema.yml`.

### 3. Por que o dado chegou em prod mesmo com o flow falhando?

O flow `_run_me_cnpj` tem a arquitetura correta de dev-gating:

```python
# pipelines/crawler/me_cnpj/flows.py
run_dbt(target="dev", dbt_command="run/test")   # gate: se falhar, para aqui
if not materialize_after_dump:
    return
upload_to_gcs(bucket_name="basedosdados")       # só chega aqui se dev passar
run_dbt(target=target, dbt_command="run/test")  # prod
update_metadata()
```

O `meteoric-albatross` falhou no gate de dev — o prod **não foi** acionado por ele.

### 4. Quem materializou em prod?

O flow `daffy-raven` — identificado via busca por flow runs ativos entre 12:00 e 13:00 UTC.

```
2026-07-15T12:32:02 → 12:47:19  Completed  br_me_cnpj.estabelecimentos
id: 1fec331f-980d-45c4-9ea3-84c7a0c4a275
```

Parâmetros:
```json
{
  "folders": ["data=2026-07-12"],
  "source_bucket": "basedosdados-dev",
  "bq_project": "basedosdados",
  "materialize_after_dump": true,
  "update_metadata": true,
  "coverage_tier": "part_bdpro",
  "year_col": "ano",
  "month_col": "mes"
}
```

`daffy-raven` é o `transfer_files_to_prod_flow` — um **flow utilitário de disparo manual** definido em `pipelines/utils/materialize_prod/flows.py`. Ele:

```python
# Sem gate de dev. Vai direto para prod.
download_files_from_bucket_folders(folders=["data=2026-07-12"], ...)
upload_to_gcs(bucket_name="basedosdados")
run_dbt(target="prod", dbt_command="run")   # só run, sem test
register_table_materialization_task(...)    # atualiza metadado
```

Diferença crítica: `dbt_command="run"` — sem testes. Os problemas dos testes de `ano`/`mes` não bloqueiam este flow.

O `deploy_schedules = []` confirma que é utilitário manual — alguém o disparou explicitamente para forçar a fotografia de 07-12 em prod enquanto o flow regular estava falhando.

### 5. Por que o metadado ficou em 07-10 se `daffy-raven` completou com `update_metadata=true`?

O `register_table_materialization_task` lê o MAX da coluna de data no BigQuery para calcular a cobertura. A cobertura para `estabelecimentos` usa `date_format=YEAR_MONTH`. Tanto a fotografia de 07-10 quanto a de 07-12 pertencem ao mês `2026-07`.

A divergência de dias (10 vs 12) dentro do mesmo mês provavelmente não atualizou a API — o metadado já registrava `2026-07` como o último mês, e a task não detectou mudança.

> **Isso ainda precisa de investigação.** Pode ser comportamento esperado (granularidade mensal) ou um bug na comparação de datas da task.

---

## Estado atual

| Item | Status |
|---|---|
| Dado em prod (`estabelecimentos`) | ✅ correto, atualizado com fotografia 2026-07-12 |
| Metadado da API | ⚠️ mostra 2026-07-10 (mesmo mês, granularidade mensal) |
| Testes dbt (`ano`/`mes` relationships) | ❌ falham — `Unrecognized name` no SQL compilado |
| OOM do pod | ✅ resolvido via PR [#1664](https://github.com/basedosdados/pipelines/pull/1664) (`memory_limit=5Gi`) |
| Retry desnecessário no `run_dbt` | ✅ corrigido via PR [#1667](https://github.com/basedosdados/pipelines/pull/1667) (`retries=0`) |

---

## O que falta resolver

### Prioritário — Recriar as tabelas dev com o schema correto

**Causa raiz confirmada:** as tabelas dev de `empresas`, `estabelecimentos` e `socios` estão sem as colunas `ano` e `mes`. O modelo dbt adicionou essas colunas em algum momento, as tabelas prod foram atualizadas, mas as tabelas dev nunca foram recriadas com o novo schema. Não é um problema isolado da `estabelecimentos` — afeta 3 das 5 tabelas do dataset.

| Tabela | `ano`/`mes` em prod | `ano`/`mes` em dev | Diagnóstico |
|---|---|---|---|
| `estabelecimentos` | ✅ | ❌ | Dev desatualizado |
| `empresas` | ✅ | ❌ | Mesmo problema |
| `socios` | ✅ | ❌ | Mesmo problema |
| `simples` | — | — | Não tem essas colunas, OK |
| `dicionario` | — | — | Não tem essas colunas, OK |

> Também existe uma tabela `tmp_invalid_chars` só em dev — staging temporária, esperado, não é problema.

Como os testes de relationship rodam com `target=dev`, o BigQuery não encontra `ano`/`mes` nas tabelas dev — daí o `Unrecognized name`.

**Fix — rodar `--full-refresh` nas três tabelas antes de qualquer `dbt test`:**

```bash
dbt run --select models/br_me_cnpj/br_me_cnpj__empresas.sql \
        --target dev --full-refresh

dbt run --select models/br_me_cnpj/br_me_cnpj__estabelecimentos.sql \
        --target dev --full-refresh

dbt run --select models/br_me_cnpj/br_me_cnpj__socios.sql \
        --target dev --full-refresh
```

Ou em uma única chamada:

```bash
dbt run --select models/br_me_cnpj/ \
        --target dev --full-refresh \
        --exclude models/br_me_cnpj/br_me_cnpj__simples.sql
```

**Atenção:** rodar `dbt test --target dev` antes do `--full-refresh` vai reproduzir o mesmo erro de `Unrecognized name: ano/mes` nas três tabelas. Sempre recriar o schema dev primeiro.

Após o `--full-refresh`, o `dbt test --target dev` passa e o flow regular `_run_me_cnpj` volta a completar normalmente para as três tabelas, incluindo o `update_metadata`.

### Investigar granularidade do metadado

Confirmar se a divergência 07-10 vs 07-12 no metadado é comportamento esperado da granularidade mensal ou se há um bug na `register_table_materialization_task`.

---

## Arquitetura relevante

```
_run_me_cnpj (flow regular, cron diário)
  └─ run_dbt(target="dev", cmd="run/test")   ← gate de qualidade
  └─ [se dev OK] upload → run_dbt(target="prod", cmd="run/test") → update_metadata

transfer_files_to_prod_flow (utilitário manual)
  └─ download de pastas específicas do GCS
  └─ upload → run_dbt(target="prod", cmd="run")   ← sem test, sem gate de dev
  └─ [se update_metadata] register_table_materialization_task
```

O utilitário foi projetado para reprocessamento emergencial. Não tem proteção de qualidade por design — use com consciência de que os testes não rodam.
