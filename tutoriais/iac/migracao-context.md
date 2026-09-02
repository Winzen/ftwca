# Contexto para Migração Prefect 0 → Prefect 3

Este documento dá contexto para ajudar na migração dos flows Python do Prefect 0.15.9 para o Prefect 3. A infraestrutura já está pronta — o trabalho é migrar os flows um a um.

---

## O que já está funcionando

- Prefect 3 server rodando em `https://prefect3.basedosdados.org`
- Dois workers Kubernetes ativos:
  - `basedosdados-dev` → namespace `prefect-worker-basedosdados-dev` (para testar)
  - `basedosdados` → namespace `prefect-worker-basedosdados` (prod)
- CI/CD configurado: push na branch `feat/prefect3` faz deploy automático de todos os flows para o pool `basedosdados` (prod)
- PR com label `deploy-flow` faz deploy dos flows alterados para o pool `basedosdados-dev` (dev)
- Flow de referência já migrado e validado end-to-end: `br_camara_dados_abertos.deputado`

---

## Como o Prefect 3 executa flows

O worker **não usa a imagem Docker para executar o código Python dos flows**. A imagem só contém as dependências. O código vem direto do GitHub em runtime:

```python
flow.from_source(
    source=GitRepository(url=REPO_URL, branch=branch_name),
    entrypoint="pipelines/datasets/meu_dataset/flows.py:nome_do_flow",
)
```

Isso significa:
- Mudar o código do flow → só precisa fazer push, não rebuild da imagem
- Mudar dependências (`pyproject.toml`) → precisa rebuild da imagem

---

## Como migrar um flow

### Estrutura mínima de um flow Prefect 3

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=10)
def minha_task():
    ...

@flow(log_prints=True)
def meu_dataset__minha_tabela(
    dataset_id: str = "meu_dataset",
    table_id: str = "minha_tabela",
    materialize_after_dump: bool = True,
    upload_to_prod: bool = False,
):
    minha_task()

# Obrigatório — lista de schedules (pode ser vazia)
meu_dataset__minha_tabela.deploy_schedules = []
```

### O que remover do código Prefect 0

| Prefect 0 | Prefect 3 |
|---|---|
| `from prefect import Parameter` | removido — use parâmetros normais Python no `@flow` |
| `Deployment.build_from_flow(...)` | substituído por `flow.from_source(...).deploy(...)` (feito pelo CI/CD) |
| `run_config = KubernetesRun(...)` | removido |
| `storage = GCS(...)` | removido |
| `executor = LocalDaskExecutor(...)` | removido |
| `with Flow("nome") as flow:` | substituído por `@flow` decorator |
| `.run()` | substituído por chamada direta à função do flow |

### Arquivo de referência já migrado

`pipelines/datasets/br_camara_dados_abertos/flows.py` — flow completo com download, upload GCS, dbt run e dbt test. Use como template.

---

## Como testar um flow em dev

### Opção 1 — Via CI/CD (recomendado)

1. Crie uma branch a partir de `feat/prefect3`
2. Migre o flow
3. Abra um PR de volta para `feat/prefect3`
4. Adicione a label `deploy-flow` no PR
5. O workflow `cd-prefect3-staging.yaml` detecta os arquivos alterados e registra os flows no pool `basedosdados-dev`
6. Acesse `https://prefect3.basedosdados.org` e acione o flow manualmente no pool `basedosdados-dev`

### Opção 2 — Deploy manual pelo terminal

```bash
cd /caminho/para/pipelines
source .env  # exporta PREFECT_API_URL e PREFECT_API_KEY

uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch nome-da-sua-branch \
  --files pipelines/datasets/meu_dataset/flows.py
```

O `.env` precisa ter:
```bash
export PREFECT_API_URL=https://prefect3.basedosdados.org/api
export PREFECT_API_KEY=<token>   # perguntar ao Luiz
```

---

## Credenciais GCP disponíveis nos pods

Cada pod de flow recebe automaticamente o secret `gcp-credentials` do namespace. O `entrypoint.sh` da imagem decodifica as credenciais antes de iniciar o flow:

| O que chega no pod | Como usar |
|---|---|
| `BASEDOSDADOS_CREDENTIALS_STAGING` (env var, base64) | lib `basedosdados` lê diretamente |
| `/credentials-dev/dev.json` (arquivo) | dbt `--target dev` via `profiles.yml` |
| `/credentials-prod/prod.json` (arquivo) | dbt `--target prod` via `profiles.yml` |

O `BASEDOSDADOS_CONFIG` (env var) configura automaticamente o `bucket_name` correto para cada worker:
- worker dev → `basedosdados-dev`
- worker prod → `basedosdados`

**Não é necessário configurar credenciais manualmente nos flows** — basta instanciar `bd.Table(...)` ou `bd.Storage(...)` normalmente.

---

## Upload para GCS com a lib `basedosdados`

```python
import basedosdados as bd

# Upload para bucket de dev (basedosdados-dev)
tb = bd.Table(dataset_id=dataset_id, table_id=table_id, bucket_name="basedosdados-dev")
st = bd.Storage(dataset_id=dataset_id, table_id=table_id, bucket_name="basedosdados-dev")

if not tb.table_exists(mode="staging"):
    tb.create(path=data_path, if_storage_data_exists="replace", if_table_exists="replace")
st.upload(path=data_path, mode="staging", if_exists="replace")
```

---

## dbt nos flows

```python
from dbt.cli.main import dbtRunner
from pathlib import Path

def run_dbt_model(dataset_id, table_id, target="dev", dbt_alias=True):
    model = f"{dataset_id}__{table_id}" if dbt_alias else table_id
    model_path = Path("models") / dataset_id / f"{model}.sql"

    if not model_path.exists():
        raise FileNotFoundError(f"Modelo dbt não encontrado: {model_path}")

    runner = dbtRunner()
    for command in ["run", "test"]:
        result = runner.invoke([command, "--select", model, "--target", target])
        if result.exception:
            raise Exception(f"dbt {command} exception: {result.exception}")
        if not result.success:
            for event in result.result or []:
                print(f"dbt | {getattr(event, 'message', event)}")
            raise Exception(f"dbt {command} falhou para {model}")
```

**Observações importantes:**
- `dbt_project.yml` já tem `packages-install-path: /app/dbt_packages` — os pacotes estão na imagem, não precisa rodar `dbt deps` em cada execução
- O `profiles.yml` já está configurado para ler os arquivos de credencial corretos em `/credentials-dev/dev.json` e `/credentials-prod/prod.json`

---

## Armadilhas comuns na migração

| Problema | Causa | Solução |
|---|---|---|
| Flow importa mas não aparece na UI | Falta `flow.deploy_schedules = []` no final do arquivo | Adicionar a linha |
| `ModuleNotFoundError: No module named 'prefect'` ao rodar deploy script | Executando sem o ambiente do projeto | Usar `uv run python ...` |
| `No such file or directory: '/credentials-dev/dev.json'` | `entrypoint.sh` só roda no pod; localmente precisa setar o caminho manualmente | Setar `BD_SERVICE_ACCOUNT_DEV` localmente, ou rodar sempre via worker |
| CSV com erro de parse no BigQuery | Arquivo original usa `;` como separador | Converter para `,` com `csv.reader`/`csv.writer` |
| dbt falha com `SubjectTokenSupplier` | `google-auth < 2.23.0` | Já corrigido na imagem atual |
| dbt falha com `dbt_packages not found` | `packages-install-path` não configurado | Já corrigido no `dbt_project.yml` |

---

## Estrutura de repositório relevante

```
pipelines/
├── .github/
│   ├── scripts/deploy_flows.py     # script de deploy
│   └── workflows/
│       ├── cd-prefect3.yaml              # push feat/prefect3 → deploy prod
│       ├── cd-prefect3-staging.yaml      # PR com label → deploy dev
│       └── build-docker-prefect3.yaml    # rebuild imagem quando deps mudam
├── Dockerfile.prefect3             # imagem com dependências
├── entrypoint.sh                   # decodifica credenciais antes do flow
├── pyproject.toml                  # dependências (uv)
├── dbt_project.yml                 # config dbt (packages-install-path aqui)
├── profiles.yml                    # targets dev e prod do dbt
└── pipelines/
    └── datasets/
        └── br_camara_dados_abertos/
            └── flows.py            # flow de referência já migrado
```

---

## Links

- Prefect 3 UI: `https://prefect3.basedosdados.org`
- Documentação oficial de migração: `https://docs.prefect.io/v3/resources/upgrade-prefect-2-to-3`
- Repositório de flows: `github.com/basedosdados/pipelines` — branch `feat/prefect3`
