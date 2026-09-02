# Prefect 3 — Pendências

## ✅ 1. Registro DNS para `prefect3.basedosdados.org` — RESOLVIDO

**Responsável:** quem tem acesso ao painel DNS da organização

**O que precisa ser feito:**
Adicionar um registro A apontando o subdomínio para o IP do cluster GKE:

| Tipo | Nome | Valor |
|---|---|---|
| `A` | `prefect3.basedosdados.org` | `35.225.94.47` |

**Por que é necessário:**
O Prefect 3 server já está rodando no cluster. O ingress e o cert-manager já estão configurados e aguardando o DNS. Assim que o registro for criado:
- O cert-manager completa o desafio HTTP01 com o Let's Encrypt automaticamente
- O certificado TLS é emitido e o secret `prefect3-tls` fica pronto
- A UI fica acessível em `https://prefect3.basedosdados.org`

**Como verificar se o DNS propagou:**
```bash
python3 -c "import socket; print(socket.gethostbyname('prefect3.basedosdados.org'))"
```
Deve retornar `35.225.94.47`.

**Como verificar se o certificado foi emitido após o DNS:**
```bash
kubectl get certificate -n prefect3
```
Deve mostrar `READY: True`.

**Contexto:**
Todos os outros subdomínios do cluster foram adicionados da mesma forma:
- `development.basedosdados.org` → `35.225.94.47`
- `staging.basedosdados.org` → `35.225.94.47`
- `prefect.basedosdados.org` → `35.225.94.47`
- `vault.basedosdados.org` → `35.225.94.47`

---

## ✅ 2. Workers do Prefect 3 — RESOLVIDO

**Responsável:** time de infraestrutura

**O que foi feito:**
Criados 2 workers Kubernetes consolidando os 4 agents antigos.

| Agent anterior | Worker Prefect 3 | Namespace K8s | Status |
|---|---|---|---|
| `basedosdados` | `basedosdados` (prod) | `prefect-worker-basedosdados` | ✅ Online |
| `basedosdados-perguntas` | consolidado em `basedosdados` | — | ✅ |
| `basedosdados-projetos` | consolidado em `basedosdados` | — | ✅ |
| `basedosdados-dev-gcp-vertex` | `basedosdados-dev` (dev) | `prefect-worker-basedosdados-dev` | ✅ Online |

**Observação:** a SA do Terraform precisou receber `roles/container.admin` para criar RBAC (Roles/RoleBindings) nos novos namespaces — mesma exigência dos agents antigos.

---

## ✅ 3. Imagem Docker com dependências dos flows — RESOLVIDO (com ressalvas)

**Responsável:** time de infraestrutura

**O que foi feito:**
- `Dockerfile.prefect3` criado no repositório `basedosdados/pipelines` (branch `feat/prefect3`)
- Workflow `.github/workflows/build-docker-prefect3.yaml` ativo — faz build e push automático para o GCR e atualiza o work pool via API
- Imagem `gcr.io/basedosdados-dev/pipelines:dev` buildada e disponível no GCR
- Flow de teste `hello_prefect3` rodou com sucesso usando a nova imagem

São necessárias **duas imagens separadas**:

| Imagem | Branch de origem | Work pool | Status |
|---|---|---|---|
| `gcr.io/basedosdados-dev/pipelines:dev` | `feat/prefect3` / feature branches | `basedosdados-dev` | ✅ Ativa |
| `gcr.io/basedosdados-dev/pipelines:latest` | `main` | `basedosdados` | ❌ Não existe — aguarda merge na main |

**Fix crítico no Dockerfile (2026-05-14):**
O `uv export` incluía o próprio projeto (`pipelines`) na lista de dependências, fazendo o `uv pip install` falhar com `ValueError: Unable to determine which files to ship`. Correção: adicionar `--no-emit-project`:
```bash
uv export --locked --no-dev --no-hashes --no-emit-project -o /tmp/requirements.txt
uv pip install --system --no-cache -r /tmp/requirements.txt
```
Os pacotes são instalados no Python do sistema (`/usr/local/lib/python3.12/site-packages`) para que o Prefect worker encontre tudo ao executar `/usr/local/bin/python -m prefect.engine`.

**Como o pod sabe qual imagem usar:**
O Kubernetes worker do Prefect cria jobs sem imagem definida por padrão — usa a variável `image` do work pool. Se não houver default, cai para `prefecthq/prefect:3-latest` (sem nossas dependências). A correção foi setar o default via API:
```bash
curl -s -X PATCH https://prefect3.basedosdados.org/api/work_pools/<pool> \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"base_job_template": {..., "variables": {"properties": {"image": {"default": "gcr.io/basedosdados-dev/pipelines:dev"}}}}}'
```
Ver seção de comandos para o script completo.

**Estado atual dos work pools (2026-05-14):**
- `basedosdados-dev` → `gcr.io/basedosdados-dev/pipelines:dev` ✅
- `basedosdados` → `gcr.io/basedosdados-dev/pipelines:dev` ⚠️ temporário até `:latest` existir (merge na main)

**Pacotes atualizados para compatibilidade com Python 3.12:**
- `PyYAML 6.0` → `>=6.0.1`
- `lxml 4.9.2` → `>=4.9.4`
- `ruamel-yaml-clib 0.2.6` → `>=0.2.8`
- `pymssql 2.2.5` → `>=2.3.0`
- `shapely 2.0.1` → `>=2.0.2`
- `grpcio 1.56.2` → `>=1.59.0`

**Por que duas imagens:**
Dependências novas ficam em teste no dev antes de irem para prod.

---

## ✅ 3.1 Credenciais GCP nos pods — RESOLVIDO

**Decisão:** um único secret Kubernetes (`gcp-credentials`) serve tanto a lib `basedosdados` quanto o dbt, sem duplicar credenciais em formatos diferentes.

**Como funciona:**

O secret armazena as variáveis como JSON **duplamente encodado em base64** (a lib basedosdados espera base64 e decodifica internamente; o Kubernetes adiciona mais uma camada ao armazenar o secret).

O pod recebe a variável com uma camada de base64. O `entrypoint.sh` (copiado na imagem pelo `Dockerfile.prefect3`) decodifica as variáveis e escreve os arquivos de credencial antes de iniciar o flow. Suporta dois modos:

```sh
#!/bin/sh
mkdir -p /credentials-dev /credentials-prod

if [ -n "$DBT_SERVICE_ACCOUNT" ]; then
    # Worker prod: usa SA dedicada para dbt em ambos os targets
    echo "$DBT_SERVICE_ACCOUNT" | base64 -d > /credentials-dev/dev.json
    echo "$DBT_SERVICE_ACCOUNT" | base64 -d > /credentials-prod/prod.json
else
    # Worker dev: reutiliza as SAs da lib basedosdados
    [ -n "$BASEDOSDADOS_CREDENTIALS_STAGING" ] && \
        echo "$BASEDOSDADOS_CREDENTIALS_STAGING" | base64 -d > /credentials-dev/dev.json
    [ -n "$BASEDOSDADOS_CREDENTIALS_PROD" ] && \
        echo "$BASEDOSDADOS_CREDENTIALS_PROD" | base64 -d > /credentials-prod/prod.json
fi

exec "$@"
```

| Consumidor | Como usa a credencial |
|---|---|
| lib `basedosdados` | lê `BASEDOSDADOS_CREDENTIALS_STAGING` ou `BASEDOSDADOS_CREDENTIALS_PROD` (base64) diretamente |
| dbt `--target dev` | lê `/credentials-dev/dev.json` via `profiles.yml` |
| dbt `--target prod` | lê `/credentials-prod/prod.json` via `profiles.yml` |

**Por que não montar dois secrets diferentes:**
Evitar ter a mesma credencial escrita em múltiplos lugares com formatos diferentes. Um secret, dois consumidores.

**`entrypoint.sh` adicionado ao trigger do build-docker:**
O workflow `build-docker-prefect3.yaml` foi atualizado para rebuildar a imagem quando `entrypoint.sh` mudar (além de `Dockerfile.prefect3`, `pyproject.toml`, etc.).

**Uso local:** o `profiles.yml` usa `env_var('BD_SERVICE_ACCOUNT_DEV', '/credentials-dev/dev.json')`. Localmente, o desenvolvedor aponta `BD_SERVICE_ACCOUNT_DEV` para o arquivo de credenciais local (ou coloca o arquivo em `/credentials-dev/dev.json`).

---

## ✅ 3.2 Credenciais do worker prod (`prefect-worker-basedosdados`) — RESOLVIDO

**O que foi feito (2026-05-20):**

Sealed secret `gcp-credentials` recriado como `secret-02_sealed.yaml` com 4 chaves (o `secret-01_sealed.yaml` tinha valores vazios por bug na geração):

| Variável | Service Account | Projeto GCP |
|---|---|---|
| `BASEDOSDADOS_CREDENTIALS_STAGING` | `prefect@basedosdados-dev` | `basedosdados-dev` ⚠️ workaround |
| `BASEDOSDADOS_CREDENTIALS_PROD` | `prefect@basedosdados` | `basedosdados` |
| `BASEDOSDADOS_CONFIG` | *(config.toml)* | `bucket_name=basedosdados` |
| `DBT_SERVICE_ACCOUNT` | `dbt-rpc@basedosdados` | `basedosdados` |

Job manifest do work pool `basedosdados` atualizado via API:
- `envFrom` com `gcp-credentials` → env vars chegam nos pods de flow
- `GOOGLE_APPLICATION_CREDENTIALS=/credentials-prod/prod.json` → ADC para libs Python que não usam credencial explícita

**Validação:**
- `dbt run --target dev` (flow `debonair-kagu`) ✅
- `dbt run --target prod` (flow `speedy-saluki`) ✅

**Workaround ativo:** `BASEDOSDADOS_CREDENTIALS_STAGING` usa `prefect@basedosdados-dev` (SA do projeto dev) para permitir upload ao bucket `basedosdados-dev` durante testes. Quando o ambiente de staging for separado, trocar para `prefect@basedosdados-staging`.

---

## ✅ 4. Script de deploy dos flows (CI/CD) — RESOLVIDO

**Responsável:** time de infraestrutura / pipelines

**O que foi feito:**
- `.github/scripts/deploy_flows.py` criado — importa flows dinamicamente, ignora flows Prefect 0.x automaticamente
- `.github/workflows/cd-prefect3.yaml` — push para `main` ou `feat/prefect3` → registra todos os flows no work pool `basedosdados`
- `.github/workflows/cd-prefect3-staging.yaml` — PR com label `deploy-flow` → registra flows alterados no work pool `basedosdados-dev`
- Secret `PREFECT3_AUTH_TOKEN` adicionado no GitHub repo

**Como o code do flow chega no worker:**
O worker não usa a imagem para executar o código Python dos flows. A imagem contém só as dependências. O código vem direto do GitHub em runtime via `GitRepository(branch=<branch>)`. Isso significa:
- Mudanças de código não precisam rebuildar a imagem
- A imagem só precisa ser recriada quando mudam dependências (`pyproject.toml`, `uv.lock`)

**Tratamento de flows Prefect 0.x:**
O script tenta importar cada arquivo. Arquivos com `from prefect import Parameter` (Prefect 0.x) falham com `ImportError` e são pulados automaticamente. Apenas flows Prefect 3 são registrados.

---

## ✅ 7. GitHub Secrets no repositório `basedosdados/pipelines` — RESOLVIDO

**Responsável:** time de infraestrutura

**O que foi feito:** secrets adicionados. `PREFECT3_API_URL` foi removido como secret e hardcodado diretamente nos workflows.

**O que precisava ser feito:**
Adicionar os seguintes secrets no repositório `basedosdados/pipelines` em Settings → Secrets and variables → Actions:

| Secret | Valor | Usado por |
|---|---|---|
| `GCP_PROJECT_ID` | `basedosdados-dev` | build-docker-prefect3.yaml |
| `GCP_SA_KEY` | JSON da SA com permissão no GCR | build-docker-prefect3.yaml |
| `PREFECT3_AUTH_TOKEN` | Token Django permanente do admin | build-docker-prefect3.yaml, cd-prefect3.yaml, cd-prefect3-staging.yaml |
| `PREFECT3_API_URL` | `https://prefect3.basedosdados.org/api` | build-docker-prefect3.yaml |

**Como criar o token Django:**
1. Acessar `https://backend.basedosdados.org/admin/authtoken/token/`
2. Criar token para o usuário de serviço (ou usar a conta admin)
3. Copiar o token gerado (ex: `998b4202-e62f-4473-b990-eb6c6a112afc`)

**Verificar se o token funciona:**
```bash
curl -s https://prefect3.basedosdados.org/api/work_pools/ \
  -H "Authorization: Bearer <token>" | python3 -m json.tool | head -20
```

**Após adicionar os secrets:** dar push da branch `feat/prefect3` para disparar o primeiro build da imagem Docker.

---

## ✅ 3.3 `vault-credentials` nos workers Prefect 3 — RESOLVIDO

**O que foi feito (2026-05-20):**

Sealed secrets criados a partir do secret existente no namespace `prefect-agent-basedosdados`:

| Namespace | Arquivo |
|---|---|
| `prefect-worker-basedosdados` | `secret-03_sealed.yaml` |
| `prefect-worker-basedosdados-dev` | `secret-02_sealed.yaml` |

Variáveis disponíveis nos pods:

| Variável | Valor |
|---|---|
| `VAULT_ADDRESS` | `http://vault.vault.svc.cluster.local:8200/` |
| `VAULT_TOKEN` | token de acesso |

`envFrom` atualizado via API nos dois work pools — pods agora recebem `VAULT_ADDRESS` e `VAULT_TOKEN` automaticamente junto com `gcp-credentials`.

---

## 3.4 Imagem `:latest` e estratégia de branch — DECISÃO PENDENTE

### Problema

Toda a migração vive na branch `feat/prefect3`, que nunca foi mergeada na `main`. Isso cria três consequências:

1. **Não existe `:latest`** — o workflow de build só roda em `feat/prefect3`, então a tag `:latest` nunca foi criada. O worker prod usa `:dev` como workaround.
2. **Push em `feat/prefect3` → deploy direto no pool `basedosdados` (prod)** — não há separação real entre "testando a migração" e "está em prod".
3. **Secrets do GitHub Actions estão no repo `pipelines`** — qualquer mudança de repo exige replicar todos eles.

### Opções

**A) Branch `prefect3/stable`**
Criar uma branch que age como "main" para o Prefect 3. CI/CD publica `:latest` quando há push nela.
- Prós: menos overhead, mesmo repo e histórico.
- Contras: continua sendo branch paralela; não resolve o acúmulo técnico de nunca mergear na `main` real.

**B) Repositório separado**
Repo novo, independente do `pipelines` atual. Secrets e CI/CD próprios.
- Prós: autonomia total, CI/CD limpo.
- Contras: flows ficam duplicados temporariamente enquanto o Prefect 0 ainda roda; overhead de manutenção de dois repos.

**C) Mergear `feat/prefect3` na `main` agora (recomendado)**
O merge na `main` é mais limpo e resolve o problema estruturalmente. Os riscos são menores do que parecem:
- O Prefect 0 agent **não lê código do GitHub** — ele usa a imagem e flows armazenados no GCS bucket. O merge não afeta o Prefect 0.
- O deploy script do Prefect 3 pula flows Prefect 0.x automaticamente (detecta `from prefect import Parameter` e ignora).
- Após o merge, `:latest` começa a ser buildado automaticamente, e o worker prod passa a usar a imagem correta.

### Próximos passos (opção C)

1. Fazer merge de `feat/prefect3` → `main`
2. O workflow `build-docker-prefect3.yaml` precisará ser ajustado para rodar também em `main` (além de `feat/prefect3`)
3. Atualizar o work pool `basedosdados` para usar `:latest` em vez de `:dev`
4. O work pool `basedosdados-dev` continua usando `:dev` (buildado de feature branches)

---

## ✅ 5.1 Utils compartilhados — Fase 1 — RESOLVIDO (2026-05-21)

**Estado dos blocos do template canônico:**

| Bloco | Status | Onde |
|---|---|---|
| A — `@flow` | ✅ | Prefect 3 nativo |
| B — parâmetros | ✅ | Args da função `@flow` |
| C — `rename_flow_run_dataset_table` | ✅ | `utils/tasks.py` |
| C' — `check_if_data_is_outdated` | ✅ ver 5.2 | `utils/tasks.py` |
| D — extração + tratamento | per-dataset | — |
| E — `upload_to_gcs` | ✅ | `utils/tasks.py` |
| F — `run_dbt` | ✅ | `utils/tasks.py` |
| V — `update_django_metadata` | ✅ ver 5.2 | `utils/tasks.py` |

---

## ✅ 5.2 Utils compartilhados — Fase 2 (metadata) — RESOLVIDO

**O que foi feito (2026-05-21):**

`pipelines/utils/` reescrito para Prefect 3 e dividido em módulos com responsabilidade única:

| Arquivo | Conteúdo | Linhas |
|---|---|---|
| `utils/utils.py` | `log()`, `is_running_in_prod()`, `query_to_line()` | ~30 |
| `utils/vault.py` | `get_vault_client`, `get_vault_secret`, `get_credentials_from_secret` | ~25 |
| `utils/gcs.py` | `dump_header`, `DBTArtifactUploader`, `get_credentials_from_env` | ~200 |
| `utils/discord.py` | `send_discord_message`, `notify_discord_on_failure`, `notify_discord` | ~80 |
| `utils/tasks.py` | `get_credentials`, `rename_flow_run_dataset_table`, `upload_to_gcs`, `run_dbt` | ~160 |

**Mudanças estruturais:**
- `__init__.py` limpo — removidos imports de flows Prefect 0 que quebravam o import no Prefect 3
- `is_running_in_prod()` adaptado: checa `flow_run.work_pool_name == "basedosdados"` via `prefect.runtime`
- `notify_discord_on_failure` tem assinatura Prefect 3: `(flow, flow_run, state, secret_path, code_owners)` — usar com `functools.partial`
- `upload_to_gcs` recebe `bucket_name` explicitamente (antes era hardcodado em `create_table_dev_*` e `create_table_prod_*`)
- `run_dbt` portado com suporte a `flags`, `_vars`, e upload de artefatos ao GCS via `DBTArtifactUploader`
- `log()` usa `get_run_logger()` do Prefect 3 dentro de tasks/flows; fallback para `logging` padrão fora

---

## 5.3 Autenticação com o backend Django (API da BD)

**Contexto:**
A task `update_django_metadata` usa `bd.Backend` (lib `basedosdados`) para fazer mutations GraphQL na API — criar/atualizar coberturas temporais, datas de update, etc. Essas mutations exigem autenticação.

**Como funciona o fluxo de autenticação:**

```
Pod de flow
  │
  ├─ VAULT_ADDRESS + VAULT_TOKEN  (via secret K8s vault-credentials)
  │
  └─ get_headers(backend)  [metadata/utils.py]
       │
       ├─ get_credentials_from_secret("api_user_prod")
       │    └─ Vault KV path: api_user_prod → {"email": "...", "password": "..."}
       │
       ├─ mutation tokenAuth(email, password) → JWT token
       │
       └─ header {"Authorization": "Bearer <token>"}
            └─ passado em todas as mutations GraphQL subsequentes
```

**O `bd.Backend` do SDK não tem autenticação embutida** — o cliente GraphQL (`gql`) é criado sem headers. A autenticação é responsabilidade do código dos pipelines via `get_headers()`.

**Paths no Vault:**

| API mode | Secret path |
|---|---|
| `prod` | `api_user_prod` |
| `staging` | `api_user_staging` |

A função `get_headers()` detecta o mode pelo URL: se `"staging"` estiver na URL do GraphQL, usa `api_user_staging`.

**Pré-requisito nos workers Prefect 3:**
Os pods precisam das env vars `VAULT_ADDRESS` + `VAULT_TOKEN` para conseguir buscar as credenciais Django em runtime. Essas variáveis chegam via sealed secret `vault-credentials` (ver seção 3.3).

**✅ Confirmado (2026-05-21):** `VAULT_ADDRESS` e `VAULT_TOKEN` chegam corretamente nos pods de flow de **dev** e **prod**, verificado via `printenv` dentro dos job pods durante execução real.

---

## ✅ 5.4 Guia de migração de flows — para quem continuar

> Esta seção é um handoff para o colega que vai continuar a migração dos flows.

### O que já está pronto na infraestrutura

Todos os blocos do template canônico estão portados. Basta usá-los:

```python
from pipelines.utils.tasks import (
    rename_flow_run_dataset_table,  # [C]
    upload_to_gcs,                  # [E]
    run_dbt,                        # [F]
    download_data_to_gcs,           # download CSV para o GCS público
)
from pipelines.utils.metadata.tasks import (
    check_if_data_is_outdated,      # [C']
    update_django_metadata,         # [V]
)
from pipelines.utils.utils import log, to_partitions
from pipelines.utils.vault import get_credentials_from_secret
from pipelines.utils.discord import notify_discord_on_failure
```

### Anatomia de um flow migrado (template Prefect 3)

```python
from functools import partial
from prefect import flow
from pipelines.utils.discord import notify_discord_on_failure

@flow(
    name="br_dataset__tabela",
    flow_run_name="br_dataset__tabela",
    log_prints=True,
    on_failure=[partial(notify_discord_on_failure, secret_path="discord_webhook")],
)
def br_dataset__tabela(
    dataset_id: str = "br_dataset",
    table_id: str = "tabela",
    periodo: str | None = None,
    materialize_after_dump: bool = True,
    dbt_alias: bool = True,
    update_metadata: bool = True,
    target: str = "prod",
) -> None:
    rename_flow_run_dataset_table(prefix="Dump: ", dataset_id=dataset_id, table_id=table_id)

    # [C'] checar se há atualização (opcional — pode pular se não aplicável)
    max_date = check_source_max_date(...)          # task específica do dataset
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

# Schedule Prefect 3 — cron sem timezone usa UTC, sempre especificar
br_dataset__tabela.deploy_schedules = [
    {"cron": "0 14 * * *", "timezone": "America/Sao_Paulo"}
]
```

### Mudanças obrigatórias ao migrar do Prefect 0

| Prefect 0 | Prefect 3 | Observação |
|---|---|---|
| `with Flow(name=...) as flow:` | `@flow(name=...) def flow(...):` | Flow vira função Python |
| `Parameter("x", default=v)` | argumento com default `x: str = v` | — |
| `with case(cond, True):` | `if cond:` | Lógica condicional Python puro |
| `upstream_tasks=[t]` | chamada sequencial ou `t.submit()` | Sem grafos de dependência explícitos |
| `@task(max_retries=N, retry_delay=timedelta(...))` | `@task(retries=N, retry_delay_seconds=N)` | — |
| `task.run(...)` | `task.fn(...)` | `.run()` não existe no Prefect 3 |
| `create_table_dev_and_upload_to_gcs(...)` | `upload_to_gcs(..., bucket_name="basedosdados-dev")` | — |
| `create_table_prod_gcs_and_run_dbt(...)` | `upload_to_gcs(..., bucket_name="basedosdados")` + `run_dbt(target=target)` | — |
| `flow.storage = GCS(...)` | removido | Código vem do GitHub em runtime |
| `flow.run_config = KubernetesRun(...)` | removido | Worker cuida disso |
| `deepcopy(flow_ibge)` + `.schedule = ...` | `@flow` wrapper por tabela + `.deploy_schedules` | Ver br_ibge_ipca como referência |
| `CronClock(cron=..., parameter_defaults={...})` | `{"cron": "...", "timezone": "..."}` em `deploy_schedules` | — |

### Como deployar um flow

```bash
# Dev (testar antes de ir para prod)
cd /mnt/d/repositorios/bd/pipelines
source .env
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

### Como disparar um teste via API

```bash
source .env
# Pegar o deployment ID na UI: https://prefect3.basedosdados.org
curl -s -X POST "$PREFECT_API_URL/deployments/<DEPLOYMENT_ID>/create_flow_run" \
  -H "Authorization: Bearer $PREFECT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"target": "dev", "materialize_after_dump": false, "update_metadata": false}}'
```

### Flows já validados end-to-end

| Flow | Status | Observações |
|---|---|---|
| `br_camara_dados_abertos__deputado` | ✅ prod (2026-05-14) | Referência de migração simples |
| `br_ibge_ipca__mes_brasil` | ✅ dev (2026-05-21) | Referência para flows com schedule + check_outdated |
| `br_bcb_taxa_selic` | ⚠️ parcial | HTTP 406 na API do BCB — adicionar `User-Agent` na task `get_selic_data` |

### Flows prioritários para migrar

| Flow | Complexidade | Observação |
|---|---|---|
| `br_ibge_inpc` (4 tabelas) | Baixa | Deploy apenas — usa mesmo código do IPCA |
| `br_ibge_ipca` (3 tabelas restantes) | Baixa | Já deployado `mes_brasil`, faltam as outras 3 em prod |
| `br_bcb_taxa_selic` | Baixa | Só falta fix do User-Agent |
| `br_fgv_igp` | Alta | 580 linhas, requer reescrita completa |

### Armadilhas conhecidas

1. **`task.run()` → `task.fn()`** — qualquer `utils.py` do crawler que chame uma task diretamente fora de um flow precisa usar `.fn()` em vez de `.run()`. Já corrigido em `ibge_inflacao/utils.py`.
2. **`datasets/__init__.py`** — contém `from pipelines.datasets.*.flows import *` que importa flows Prefect 0 e quebra. Nunca importar via `pipelines.datasets.*` diretamente — usar `importlib` ou import direto do módulo.
3. **Artefatos dbt (403)** — SA de dev não tem `serviceusage.services.use` no bucket de artefatos. Não bloqueia o flow (capturado em `try/except`) mas gera aviso nos logs.
4. **`to_partitions`** — função utilitária que estava faltando; foi adicionada a `utils/utils.py` em 2026-05-21.

---

## 5. Migração dos flows Python

**Responsável:** times que mantêm os flows

**O que precisa ser feito:**
Os flows escritos para Prefect 0.15.9 precisam ser migrados para Prefect 3. A branch `feat/prefect3` no repositório `basedosdados/pipelines` já tem o Prefect 3 configurado como dependência — os flows precisam ser atualizados um a um.

**O que muda na prática:**
- Deploy: substituir `Deployment.build_from_flow(...)` por `flow.from_source(...).deploy(...)`
- Remover `run_configs`, `storage`, `executor` das definições
- `@flow` e `@task` continuam iguais

Documentação oficial de migração:
- https://docs.prefect.io/v3/resources/upgrade-prefect-2-to-3

**Progresso (2026-05-14):**
- `br_bcb_taxa_selic` migrado e deployado no pool `basedosdados` (usando `:dev` temporariamente)
  - Flow executou com sucesso até a task `get_selic_data`, que falhou com **HTTP 406** da API do BCB (`api.bcb.gov.br`) — problema externo à infra, a API rejeita o User-Agent padrão do `requests`. A infra está funcionando corretamente.
  - Próximo passo: corrigir o header `User-Agent` na task `get_selic_data` e validar o flow end-to-end

- ✅ `br_camara_dados_abertos.deputado` — **validado end-to-end no Prefect 3** (2026-05-14)
  - check_url ✅ → download ✅ → upload GCS ✅ → dbt run ✅ → dbt test ✅
  - Fixes aplicados durante a migração:
    - `google-auth` despinado de `==2.22.0` para `>=2.23.0` (`SubjectTokenSupplier` ausente na versão antiga, exigido pelo dbt-bigquery)
    - `dbt_project.yml`: adicionado `packages-install-path: /app/dbt_packages` para que o dbt encontre os pacotes instalados na imagem, independente do diretório de execução clonado pelo Prefect
    - CSV da Câmara usa `;` como separador — convertido para `,` no task de download via `csv.reader`/`csv.writer`
    - Credenciais: `entrypoint.sh` decodifica `BASEDOSDADOS_CREDENTIALS_STAGING` → `/credentials-dev/dev.json` para o dbt (ver seção 3.1)

- ✅ `br_ibge_ipca__mes_brasil` — **validado end-to-end em dev no Prefect 3** (2026-05-21)
  - collect_data ✅ → check_outdated ✅ → upload GCS dev ✅ → dbt run ✅ → dbt test ✅ (`PASS=2`)
  - 4 flows deployados no pool `basedosdados-dev` com schedules (ver seção 5.4)
  - Fixes aplicados durante a migração:
    - `task.run()` → `task.fn()` em `crawler/ibge_inflacao/utils.py` (Prefect 0 API removida)
    - `to_partitions` adicionado a `utils/utils.py` (estava faltando após o split)
    - Pattern `deepcopy(flow)` substituído por `@flow` wrappers com `deploy_schedules`

- ⏳ `br_ibge_inpc` (4 tabelas) — código pronto, aguarda deploy (idêntico ao IPCA)

- ⚠️ `br_bcb_taxa_selic` — migrado, falha com HTTP 406 na API do BCB — adicionar `User-Agent` em `get_selic_data`

---

## 7. (Opcional) Migrar imagem Docker do GCR para GHCR

**Motivação:** O Prefect 0 já usa `ghcr.io/basedosdados/prefect-flows`. Manter consistência e eliminar o secret `GCP_SA_KEY` do GitHub Actions (substituído pelo `GITHUB_TOKEN` nativo). O custo do GCR é irrisório (~$2/ano), então a razão principal é simplificação operacional.

**O que mudar:**
- `build-docker-prefect3.yaml` — trocar push de `gcr.io/basedosdados-dev/pipelines` para `ghcr.io/basedosdados/pipelines`
- Autenticação via `GITHUB_TOKEN` nativo, sem precisar do secret `GCP_SA_KEY`
- Default de imagem nos work pools: `ghcr.io/basedosdados/pipelines:dev` e `:latest`

---

## 8. Observabilidade — Prometheus + Grafana nos pods do Prefect 3

Verificar se os pods do Prefect 3 estão sendo monitorados pelo Prometheus e visíveis no Grafana.

**O que checar:**
- Os workers (`prefect-worker-basedosdados` e `prefect-worker-basedosdados-dev`) estão expondo métricas para o Prometheus?
- O Prometheus está fazendo scrape dos namespaces `prefect-worker-basedosdados` e `prefect-worker-basedosdados-dev`?
- Há dashboards no Grafana cobrindo os job pods (CPU/memória)?
- Pods que morrem por OOM aparecem no Grafana com sinal claro?

**Referência:** `k8s/observability/prometheus/configmap.yaml`

---

## 6. Cutover final

Após imagens, CI/CD e flows migrados:

1. Testar todos os flows no Prefect 3 em dev
2. Validar em prod com flows reais
3. Trocar o ingress de `prefect.basedosdados.org` para apontar para o novo server
4. Desativar o namespace `prefect` antigo

---

## 9. Script de automação para manter flows pausados

**Contexto:**
Hoje toda vez que há um merge na `main` do pipelines, o CI redeploya e reativa todos os flows — incluindo os que estão pausados por erro. É necessário pausar manualmente os flows com problema logo após cada merge. Isso é frágil e manual.

**O que construir:**
Um script/flow no Prefect 3 que mantenha a lista de flows pausados automaticamente em dois momentos:

1. **Após deploy (trigger: merge na main)** — logo após o CI terminar o deploy, o script lê a lista `SHOULD_BE_PAUSED` e pausa novamente os flows que devem estar pausados.

2. **Após falha de flow (trigger: flow com estado `Failed` ou `Crashed`)** — quando um flow falha, pausar automaticamente para evitar runs repetidos com erro até que seja corrigido. Pode usar automações do Prefect 3 (`POST /api/automations`) ou um flow de monitoramento periódico.

**Referência:**
A lista `SHOULD_BE_PAUSED` e os scripts de pausa manual já existem — o objetivo é automatizar o que hoje é feito à mão após cada merge.
