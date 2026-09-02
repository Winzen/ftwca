# Prefect 3 — Guia de Migração de Flows

> Handoff para quem for continuar a migração. Tudo que precisou ser mudado para os primeiros flows funcionarem, e o template para os próximos.

---

## 1. O que mudou nos utils compartilhados

### 1.1 Divisão de módulos (`pipelines/utils/`)

O `utils.py` original (1033 linhas, Prefect 0) foi quebrado em módulos com responsabilidade única:

| Arquivo novo | Conteúdo |
|---|---|
| `utils/utils.py` | `log()`, `is_running_in_prod()`, `query_to_line()`, `to_partitions()` |
| `utils/vault.py` | `get_vault_client`, `get_vault_secret`, `get_credentials_from_secret` |
| `utils/gcs.py` | `dump_header`, `DBTArtifactUploader`, `get_credentials_from_env` |
| `utils/discord.py` | `send_discord_message`, `notify_discord_on_failure`, `notify_discord` |
| `utils/tasks.py` | tasks Prefect 3: `rename_flow_run_dataset_table`, `upload_to_gcs`, `run_dbt`, `download_data_to_gcs` |

**`__init__.py` limpo** — o original tinha `from pipelines.datasets.*.flows import *` que tentava importar flows Prefect 0 e quebrava qualquer import do módulo `utils`. Removido.

### 1.2 Mudanças em `utils/utils.py`

**`is_running_in_prod()`** — antes checava env var `PREFECT__CLOUD__AUTH_TOKEN`. Agora usa `prefect.runtime`:
```python
from prefect.runtime import flow_run

def is_running_in_prod() -> bool:
    pool = flow_run.work_pool_name or ""
    return pool == "basedosdados"
```

**`to_partitions()` adicionada de volta** — existia no `utils.py` original mas ficou faltando após o split. Salva um DataFrame em esquema de partições Hive:
```python
def to_partitions(data: pd.DataFrame, partition_columns: list[str], savepath: str, file_type: str = "csv") -> None:
    """Salva data em Hive partition schema: savepath/col=val/data.csv"""
    savepath = Path(savepath)
    unique_combinations = data[partition_columns].drop_duplicates().to_dict(orient="records")
    for combo in unique_combinations:
        partition_path = savepath / "/".join(f"{k}={v}" for k, v in combo.items())
        partition_path.mkdir(parents=True, exist_ok=True)
        df_slice = (data.loc[data[list(combo.keys())].isin(list(combo.values())).all(axis=1)]
                       .drop(columns=partition_columns))
        if file_type == "csv":
            out = partition_path / "data.csv"
            df_slice.to_csv(out, sep=",", encoding="utf-8", na_rep="", index=False,
                            mode="a", header=not out.exists())
        elif file_type == "parquet":
            (partition_path / "data.parquet").write_bytes(
                df_slice.to_parquet(index=False, compression="gzip"))
```

**`log()`** — usa `get_run_logger()` dentro de tasks/flows; fallback para `logging.getLogger` fora de contexto Prefect.

### 1.3 Mudanças em `utils/tasks.py`

Decorator Prefect 3 em todas as tasks:
```python
# Prefect 0
@task(max_retries=3, retry_delay=timedelta(seconds=30))

# Prefect 3
@task(retries=3, retry_delay_seconds=30)
```

**`upload_to_gcs`** — antes chamava `create_table_dev_*` / `create_table_prod_*` (funções que hardcodavam o bucket). Agora recebe `bucket_name` explicitamente:
```python
@task(retries=2, retry_delay_seconds=30)
def upload_to_gcs(data_path: str, dataset_id: str, table_id: str,
                  bucket_name: str, dump_mode: str = "append") -> None:
```

Chamar com:
- `bucket_name="basedosdados-dev"` para dev/staging
- `bucket_name="basedosdados"` para prod

**`download_data_to_gcs` adicionada** — task que faz BQ → GCS em CSV.gz com regras por tamanho da tabela:

```python
@task(retries=2, retry_delay_seconds=30)
def download_data_to_gcs(
    dataset_id: str,
    table_id: str,
    project_id: str = "basedosdados",
    bd_project_mode: str = "prod",
    billing_project_id: str | None = None,
    location: str = "US",
) -> None:
```

Regras de tamanho:
- `> 1 GB`: pula (tabela grande demais)
- `100 MB – 1 GB`: só BDPro (sem dado aberto)
- `< 100 MB`: open + BDPro (gerencia `row_access_policy` para o BDPro)

### 1.4 Mudanças em `utils/metadata/tasks.py`

Port completo para Prefect 3 (commit `c7ecd442`).

**Decorator atualizado:**
```python
# Antes
@task(max_retries=3, retry_delay=timedelta(seconds=30), checkpoint=False)

# Depois
@task(retries=3, retry_delay_seconds=30)
```

**`_get_redis_client` → `__get_redis_client`** — função auxiliar com double-underscore (evita exposição via `from ... import *`). Os call sites dentro do módulo usam `__get_redis_client(...)`.

**Imports corrigidos:**
```python
# Antes (imports do utils.py monolítico)
from pipelines.utils.utils import get_credentials_from_secret, notify_discord

# Depois (imports dos módulos corretos)
from pipelines.utils.vault import get_credentials_from_secret
from pipelines.utils.discord import notify_discord
```

### 1.5 Mudanças em `utils/metadata/utils.py`

Mesmos imports corrigidos (mesmo problema de origem):
```python
from pipelines.utils.vault import get_credentials_from_secret
from pipelines.utils.discord import notify_discord
```

---

## 2. Autenticação com o backend Django

A task `update_django_metadata` faz mutations GraphQL na API da BD. Exige autenticação via JWT.

### Fluxo de autenticação

```
Pod de flow
  │
  ├─ VAULT_ADDRESS + VAULT_TOKEN  (via secret K8s vault-credentials)
  │
  └─ get_headers(backend)  [metadata/utils.py ~linha 719]
       │
       ├─ get_credentials_from_secret("api_user_prod")
       │    └─ Vault KV: api_user_prod → {"email": "...", "password": "..."}
       │
       ├─ mutation tokenAuth(email, password) → JWT token
       │
       └─ {"Authorization": "Bearer <token>"}  ← passado nas mutations
```

**O SDK `bd.Backend` não tem auth embutida** — o cliente gql é criado sem headers. Auth é feita em `get_headers()` nos pipelines.

```python
# metadata/utils.py — get_headers()
def get_headers(backend: bd.Backend) -> dict:
    api_mode = "staging" if "staging" in backend.graphql_url else "prod"
    credentials = get_credentials_from_secret(secret_path=f"api_user_{api_mode}")
    mutation = """
        mutation ($email: String!, $password: String!) {
            tokenAuth(email: $email, password: $password) { token }
        }
    """
    response = backend._execute_query(
        query=mutation,
        variables={"email": credentials["email"], "password": credentials["password"]},
    )
    return {"Authorization": f"Bearer {response['tokenAuth']['token']}"}
```

| API mode | Vault path |
|---|---|
| prod | `api_user_prod` |
| staging | `api_user_staging` |

**Pré-requisito:** pods precisam de `VAULT_ADDRESS` + `VAULT_TOKEN`. Chegam via sealed secret `vault-credentials` nos dois namespaces de worker (ver Prefect 3 Pendencias.md seção 3.3). **Confirmado funcionando em dev e prod (2026-05-21).**

---

## 3. Conversão Prefect 0 → Prefect 3

| Prefect 0 | Prefect 3 | Observação |
|---|---|---|
| `with Flow(name=...) as flow:` | `@flow(name=...) def flow(...):` | Flow vira função Python |
| `Parameter("x", default=v)` | argumento com default `x: str = v` | — |
| `with case(cond, True):` | `if cond:` | Python puro |
| `upstream_tasks=[t]` | chamada sequencial ou `t.submit()` | Sem grafos explícitos |
| `@task(max_retries=N, retry_delay=timedelta(...))` | `@task(retries=N, retry_delay_seconds=N)` | — |
| `task.run(...)` | `task.fn(...)` | `.run()` não existe no Prefect 3 |
| `create_table_dev_and_upload_to_gcs(...)` | `upload_to_gcs(..., bucket_name="basedosdados-dev")` | — |
| `create_table_prod_gcs_and_run_dbt(...)` | `upload_to_gcs(..., bucket_name="basedosdados")` + `run_dbt(target=target)` | — |
| `flow.storage = GCS(...)` | removido | Código vem do GitHub em runtime |
| `flow.run_config = KubernetesRun(...)` | removido | Worker cuida disso |
| `deepcopy(flow_base)` + `.schedule = ...` | factory `_flow(table_id, cron)` retornando `@flow` wrappers | Ver `br_ibge_ipca` como referência |
| `CronClock(cron=..., parameter_defaults={...})` | `{"cron": "...", "timezone": "..."}` em `deploy_schedules` | — |

---

## 4. Template canônico de um flow migrado

```python
"""
Flow br_dataset__tabela — Prefect 3.
"""
from functools import partial
from prefect import flow

from pipelines.utils.discord import notify_discord_on_failure
from pipelines.utils.metadata.tasks import check_if_data_is_outdated, update_django_metadata
from pipelines.utils.tasks import rename_flow_run_dataset_table, run_dbt, upload_to_gcs


@flow(
    name="br_dataset__tabela",
    log_prints=True,
    on_failure=[partial(notify_discord_on_failure, secret_path="discord_webhook")],
)
def br_dataset__tabela(
    dataset_id: str = "br_dataset",
    table_id: str = "tabela",
    materialize_after_dump: bool = True,
    dbt_alias: bool = True,
    update_metadata: bool = True,
    target: str = "prod",
) -> None:
    rename_flow_run_dataset_table(prefix="Dump: ", dataset_id=dataset_id, table_id=table_id)

    # [C'] checar atualização (opcional)
    max_date = check_source_max_date(...)
    is_outdated = check_if_data_is_outdated(
        dataset_id=dataset_id, table_id=table_id,
        data_source_max_date=max_date, date_format="%Y-%m-%d",
    )
    if not is_outdated:
        return

    # [D] extração + tratamento — tasks específicas do dataset
    filepath = extract_and_treat(...)

    # [E] upload dev
    upload_to_gcs(data_path=filepath, dataset_id=dataset_id, table_id=table_id,
                  bucket_name="basedosdados-dev", dump_mode="append")
    # [F] dbt dev
    run_dbt(dataset_id=dataset_id, table_id=table_id,
            dbt_command="run/test", dbt_alias=dbt_alias, target="dev")

    if not materialize_after_dump:
        return

    # [E'] upload prod
    upload_to_gcs(data_path=filepath, dataset_id=dataset_id, table_id=table_id,
                  bucket_name="basedosdados", dump_mode="append")
    # [F'] dbt prod
    run_dbt(dataset_id=dataset_id, table_id=table_id,
            dbt_command="run/test", dbt_alias=dbt_alias, target=target)

    # [V] metadados
    if update_metadata:
        update_django_metadata(
            dataset_id=dataset_id, table_id=table_id,
            date_column_name={"year": "ano", "month": "mes"},
            date_format="%Y-%m", coverage_type="part_bdpro",
            time_delta={"months": 6}, bq_project="basedosdados",
        )


br_dataset__tabela.deploy_schedules = [
    {"cron": "0 14 * * *", "timezone": "America/Sao_Paulo"}
]
```

### Pattern para múltiplas tabelas com schedule diferente

Quando um mesmo dataset tem várias tabelas (como IPCA/INPC), usar factory em vez de `deepcopy`:

```python
# pipelines/datasets/br_ibge_ipca/flows.py
from prefect import flow
from pipelines.crawler.ibge_inflacao.flows import _run_ibge_inflacao


def _ipca_flow(table_id: str, cron: str):
    @flow(name=f"br_ibge_ipca__{table_id}", log_prints=True)
    def _flow(
        dataset_id: str = "br_ibge_ipca",
        table_id: str = table_id,
        periodo: str | None = None,
        materialize_after_dump: bool = True,
        dbt_alias: bool = True,
        update_metadata: bool = True,
        target: str = "prod",
    ) -> None:
        _run_ibge_inflacao(
            dataset_id=dataset_id, table_id=table_id, periodo=periodo,
            materialize_after_dump=materialize_after_dump, dbt_alias=dbt_alias,
            update_metadata=update_metadata, target=target,
        )

    _flow.deploy_schedules = [{"cron": cron, "timezone": "America/Sao_Paulo"}]
    return _flow


br_ibge_ipca__mes_brasil           = _ipca_flow("mes_brasil",           "40 14 8,9,10,11,12,13 * *")
br_ibge_ipca__mes_categoria_brasil = _ipca_flow("mes_categoria_brasil", "30 14 8,9,10,11,12,13 * *")
br_ibge_ipca__mes_categoria_rm     = _ipca_flow("mes_categoria_rm",     "20 14 8,9,10,11,12,13 * *")
br_ibge_ipca__mes_categoria_municipio = _ipca_flow("mes_categoria_municipio", "50 14 8,9,10,11,12,13 * *")
```

**Por que factory e não `deepcopy`:**
O `deploy_flows.py` registra um flow só se `obj.fn.__code__.co_filename == str(path.resolve())`. Flows importados de outro arquivo têm o `co_filename` do arquivo original — o deploy script não os registra. O factory cria `@flow` funções **definidas no arquivo correto**.

---

## 5. Como deployar e testar

### Deploy

```bash
cd /mnt/d/repositorios/bd/pipelines
source .env

# Dev (testar antes de ir para prod)
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch feat/prefect3 \
  --files pipelines/datasets/SEU_DATASET/flows.py

# Prod (só após validar em dev)
uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados \
  --branch feat/prefect3 \
  --files pipelines/datasets/SEU_DATASET/flows.py
```

### Disparar via API (sem parâmetros de produção)

```bash
source .env
# Pegar o deployment ID na UI: https://prefect3.basedosdados.org
curl -s -X POST "$PREFECT_API_URL/deployments/<DEPLOYMENT_ID>/create_flow_run" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"target": "dev", "materialize_after_dump": false, "update_metadata": false}}'
```

---

## 6. Armadilhas conhecidas

### 6.1 `task.run()` → `task.fn()`

Qualquer `utils.py` de crawler que chame uma task diretamente (fora de um flow) usa a API Prefect 0 `.run()`. No Prefect 3 isso não existe — usar `.fn()`:

```python
# Prefect 0
result = task_get_api_most_recent_date.run(dataset_id=..., table_id=...)

# Prefect 3
result = task_get_api_most_recent_date.fn(dataset_id=..., table_id=...)
```

**Já corrigido em:** `crawler/ibge_inflacao/utils.py` (funções `get_date_api` e `next_date_update`).
**Onde pode ainda existir:** qualquer `utils.py` de crawler que chame tasks dos utils diretamente.

### 6.2 `datasets/__init__.py` quebra imports

`pipelines/datasets/__init__.py` tem:
```python
from pipelines.datasets.botdosdados.flows import *   # Prefect 0 → ImportError
```

Nunca fazer `from pipelines.datasets import ...` ou `import pipelines.datasets` diretamente.
Usar import direto: `from pipelines.datasets.br_ibge_ipca.flows import br_ibge_ipca__mes_brasil`

### 6.3 Artefatos dbt — 403 não-bloqueante

A SA de dev não tem `serviceusage.services.use` no bucket de artefatos. Vai aparecer nos logs como aviso mas **não bloqueia o flow** (capturado em `try/except` no `run_dbt`). Fix pendente: conceder permissão na SA dev.

### 6.4 `to_partitions` estava faltando

Função utilitária que estava no `utils.py` original mas não foi migrada para `utils/utils.py` no split. Adicionada em 2026-05-21. Se algum flow usar `to_partitions` e falhar na import, verificar que está no `utils/utils.py` atual.

### 6.5 `deploy_flows.py` só registra flows **definidos** no arquivo

O script compara `obj.fn.__code__.co_filename` com o caminho do arquivo sendo processado. Flows apenas **importados** de outro módulo são ignorados. Flows precisam ser **instanciados** no arquivo do dataset.

### 6.6 Pod travado por falta de `BASEDOSDADOS_CONFIG`

Se um pod ficar rodando por horas/dias sem produzir output, pode ser o wizard interativo do `basedosdados` lib tentando fazer setup. Causa: `BASEDOSDADOS_CONFIG` não presente no pod. Todos os pods devem receber essa env var via `gcp-credentials` secret.

---

## 7. Flows migrados e pendentes

### Validados end-to-end

| Flow | Status | Data | Observações |
|---|---|---|---|
| `br_camara_dados_abertos__deputado` | ✅ prod | 2026-05-14 | Referência de migração simples |
| `br_ibge_ipca__mes_brasil` | ✅ dev | 2026-05-21 | Referência para flows com schedule + check_outdated |
| `br_ibge_ipca__mes_categoria_brasil` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_ipca__mes_categoria_rm` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_ipca__mes_categoria_municipio` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_inpc__mes_brasil` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_inpc__mes_categoria_brasil` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_inpc__mes_categoria_rm` | deployado | 2026-05-21 | Não testado end-to-end ainda |
| `br_ibge_inpc__mes_categoria_municipio` | deployado | 2026-05-21 | Não testado end-to-end ainda |

### Pendentes / com problemas

| Flow | Estado | O que falta |
|---|---|---|
| `br_bcb_taxa_selic` | ⚠️ HTTP 406 | Adicionar `User-Agent` header na task `get_selic_data` da API do BCB |
| `br_fgv_igp` | ❌ não migrado | 580 linhas, requer reescrita completa |

### Próximos a migrar (sugestão de prioridade)

| Flow | Complexidade | Motivo |
|---|---|---|
| `br_bcb_taxa_selic` | Baixa | Só falta o fix do `User-Agent` |
| `br_ibge_inpc` 4 tabelas | Baixa | Código pronto, só fazer deploy e testar |
| `br_ibge_ipca` 3 tabelas restantes | Baixa | Já em dev, promover para prod |
| outros crawlers IBGE | Média | Verificar se usam `task.run()` nos utils |
| `br_fgv_igp` | Alta | Reescrita completa necessária |

---

## 8. Aumentar memória / CPU dos pods

Existem dois tipos de pod com limites independentes.

### 8.1 Pod do worker (processo de polling)

Configurado no `values.yaml` do Helm chart. Esse pod só faz polling no Prefect server — raramente precisa de mais recursos.

**Arquivos:**
- `k8s/prefect_workers/basedosdados-dev/chart/values.yaml`
- `k8s/prefect_workers/basedosdados/chart/values.yaml`

Limites atuais (ambos os workers):
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

Para alterar: editar o `values.yaml` e fazer helm upgrade:
```bash
helm upgrade prefect-worker-basedosdados-dev \
  -n prefect-worker-basedosdados-dev \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados-dev/chart/values.yaml

# ou para o worker prod:
helm upgrade prefect-worker-basedosdados \
  -n prefect-worker-basedosdados \
  prefecthq/prefect-worker \
  -f k8s/prefect_workers/basedosdados/chart/values.yaml
```

### 8.2 Pods de flow (jobs criados a cada run) — o que importa para OOM

Os limites dos pods que **executam os flows** ficam no `base_job_template` do work pool, configurado via API do Prefect (não no Helm). É esse que precisa ser aumentado quando um flow sofre OOM.

**Ver limites atuais:**
```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados-dev" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  | python3 -m json.tool | grep -A5 -i "memory\|cpu\|limit\|request"
```

**Alterar via PATCH:**
```bash
source .env
curl -s -X PATCH "$PREFECT_API_URL/work_pools/basedosdados-dev" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "base_job_template": {
      "variables": {
        "properties": {
          "finished_job_ttl": {"default": 60},
          "resources": {
            "default": {
              "requests": {"cpu": "500m", "memory": "1Gi"},
              "limits":   {"cpu": "2",    "memory": "4Gi"}
            }
          }
        }
      }
    }
  }'
```

> O campo `resources` dentro de `base_job_template.variables.properties` é passado diretamente para o `spec.containers[0].resources` do Job Kubernetes. Ajustar conforme a necessidade do flow.

**Pools disponíveis:**
| Pool | Namespace | Quando usar |
|---|---|---|
| `basedosdados-dev` | `prefect-worker-basedosdados-dev` | flows de teste / dev |
| `basedosdados` | `prefect-worker-basedosdados` | flows de prod |

---

## 9. Imports de referência

```python
from pipelines.utils.tasks import (
    rename_flow_run_dataset_table,  # renomear o flow run na UI
    upload_to_gcs,                  # fazer upload para GCS
    run_dbt,                        # rodar dbt run/test
    download_data_to_gcs,           # BQ → GCS para dado público
)
from pipelines.utils.metadata.tasks import (
    check_if_data_is_outdated,      # checar se a fonte tem dados novos
    update_django_metadata,         # atualizar cobertura temporal no backend
)
from pipelines.utils.utils import log, to_partitions
from pipelines.utils.vault import get_credentials_from_secret
from pipelines.utils.discord import notify_discord_on_failure
```
