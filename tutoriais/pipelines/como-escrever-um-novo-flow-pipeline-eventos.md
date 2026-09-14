# Pipeline orientado a eventos — como escrever um novo flow

Passo a passo pra migrar um dataset (ou criar um novo) pro padrão `check_update → download → mat_test` da issue #1867. Complementa [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]] — aquele documento é a referência de nomenclatura/convenções (consulta rápida); este aqui é o walkthrough, com código real, pra quem tá escrevendo um flow pela primeira vez.

Exemplo principal: `br_ans_beneficiario` (1 tabela, caso mais comum). Depois, a variação pra dataset com várias tabelas (`br_ibge_ipca`).

## O que você escreve, e o que já existe pronto

| Você escreve | Já existe, só usa |
|---|---|
| `constants.py`: `DATASET_ID` + `*_TABLE_ID` | `CheckThenDownloadPipeline`/`pipeline_factory` (`pipelines/utils/stage_dispatch.py`) — todo o boilerplate de dispatch |
| `tasks.py`: `check_for_update` e `download_data` — a lógica específica da fonte | `mat_test_flow` (`pipelines/utils/metadata/flows.py`) — dbt run/test, promoção pra prod, atualização de coverage. Genérico, um deployment só, **nunca precisa ser escrito** |
| `flows.py`: a fiação (2 `@flow` por tabela) | `poll_source_for_update_task`/`commit_source_update_task`/`upload_to_gcs` — chamados por dentro da cápsula, nunca direto pelo seu código |

Ou seja: o trabalho real é escrever **duas funções Python puras** (`check_for_update`, `download_data`) que devolvem os tipos certos. O resto é fiação repetitiva, já resolvida pela cápsula.

## Antes de escrever qualquer código

Abrir o código real do dataset (`pipelines/datasets/<dataset>/` se já existir, e/ou `pipelines/crawler/<nome>/`) — **nunca confiar num levantamento de alto nível** pra saber quantas tabelas tem ou se o check é leve de verdade. Já erramos a contagem de tabelas 3 vezes nesta migração (`br_denatran_frota`, `br_ibge_ipca`, `br_sfb_sicar`) e classificamos 2 checks como "leves" quando na prática baixavam dado antes de checar (`br_ibge_ipca`, `br_inmet_bdmep` — ver [[migracao-lote-10-datasets]]).

Pergunta chave: **o check é leve de verdade e independente do download, ou "disfarçado"?**
- **Leve de verdade** (`br_ans_beneficiario`): a fonte expõe a data mais recente sem precisar baixar dado (HEAD request, HTML de listagem, FTP, endpoint de metadata).
- **Disfarçado** (`br_ibge_ipca`): a única forma de saber a data mais recente é baixar o período mais novo e inspecionar. Nesse caso, `download_data` acaba refazendo essa mesma busca — não dá pra passar dado entre pods, e geralmente o download é pequeno o bastante pra valer a pena baixar duas vezes (ver critério de 5 GB em `levantamento-datasets-por-categoria-de-check.md`).

## Passo 1 — `constants.py`

Só literais: `DATASET_ID` e um `*_TABLE_ID` por tabela.

```python
"""
Constant values for br_ans_beneficiario.
"""

DATASET_ID = "br_ans_beneficiario"

INFORMACAO_CONSOLIDADA_TABLE_ID = "informacao_consolidada"
INFORMACAO_CONSOLIDADA_URL = "https://dadosabertos.ans.gov.br/FTP/PDA/informacoes_consolidadas_de_beneficiarios-024/"
```

Nunca importa de `tasks.py` — evita import circular (`tasks.py` já importa `constants.py`).

## Passo 2 — `tasks.py`: `check_for_update`

Assinatura fixa: `() -> CheckResult`. Só precisa descobrir a data mais recente na fonte — a decisão "isso é dado novo?" (comparar contra a coverage do backend) é genérica, não é responsabilidade sua.

```python
from datetime import date

from pipelines.crawler.ans_beneficiario.tasks import extract_links_and_dates, get_file_max_date
from pipelines.datasets.br_ans_beneficiario.constants import INFORMACAO_CONSOLIDADA_URL
from pipelines.utils.stage_dispatch import CheckResult


def br_ans_beneficiario_check_for_update() -> CheckResult:
    links_and_dates = extract_links_and_dates(url=INFORMACAO_CONSOLIDADA_URL)
    file_last_date = get_file_max_date(df=links_and_dates)  # "YYYY-MM"
    reference_date = date.fromisoformat(f"{file_last_date}-01")
    return CheckResult(reference_date=reference_date)
```

`CheckResult` (`stage_dispatch.py`):

```python
@dataclass
class CheckResult:
    reference_date: datetime.date
    extra_download_params: dict = field(default_factory=dict)
```

`extra_download_params` só é necessário quando `check_for_update` descobre algo que `download_data` vai precisar e que não dá pra recalcular a partir da data (ex. uma URL exata que só aparece na checagem). Na maioria dos casos fica vazio — `download_data` recebe `reference_date` e refaz o que precisar.

## Passo 3 — `tasks.py`: `download_data`

Assinatura fixa: `(download_params: dict) -> DownloadResult`. `download_params` sempre tem `reference_date` (string ISO), mais o que `extra_download_params` tiver posto.

```python
from pipelines.crawler.ans_beneficiario.tasks import crawler_ans, files_to_download
from pipelines.utils.metadata.domain import DateFormat, PartBdpro, YearMonth
from pipelines.utils.stage_dispatch import DownloadResult


def br_ans_beneficiario_download(download_params: dict) -> DownloadResult:
    links_and_dates = extract_links_and_dates(url=INFORMACAO_CONSOLIDADA_URL)
    files = files_to_download(df=links_and_dates, year=None)
    output_filepath = crawler_ans(files=files)

    ref = date.fromisoformat(download_params["reference_date"])
    return DownloadResult(
        coverage=PartBdpro(
            date_column=YearMonth(year="ano", month="mes"),
            date_format=DateFormat.YEAR_MONTH,
        ).model_dump(),
        data_path=output_filepath,
        bq_project="basedosdados",
        source_format="parquet",
        partition_folders=[f"ano={ref.year}/mes={ref.month:02d}"],
    )
```

**Nunca chama `upload_to_gcs` diretamente** — a cápsula faz isso por você (`CheckThenDownloadPipeline.run_download`), a partir do `data_path` que você devolver.

`DownloadResult` (`stage_dispatch.py`):

```python
@dataclass
class DownloadResult:
    coverage: dict
    data_path: str
    targets: list[str] = field(default_factory=lambda: ["dev", "prod"])
    bq_project: str = "basedosdados"
    prefect_mode: str = "prod"
    partition_folders: list[str] | None = None
    dump_mode: str = "append"
    source_format: str = "csv"
```

- **`data_path`**: caminho local do arquivo (ou da pasta base, se particionado — ver `partition_folders`) que você escreveu.
- **`partition_folders`**: só preenche se o dado é particionado estilo Hive (`ano=2026/mes=08`) — faz o `mat_test` promover só a fatia nova pra prod, não o staging inteiro.
- **`dump_mode`**: `"append"` (a maioria) ou `"overwrite"` (reconstrói o histórico inteiro a cada run — raro, só quando a fonte não permite incremental).
- **`source_format`**: `"csv"` (default) ou `"parquet"` — depende do que seu crawler escreve.

### `coverage` — qual `CoverageSpec` usar

| Tipo | Quando usar |
|---|---|
| `AllFree` | Dado sempre público, sem tier BD pro |
| `AllBdpro` | Dado sempre fechado (BD pro) |
| `PartBdpro` | Série histórica com defasagem — dado recente é BD pro, mais antigo vira público depois de um tempo (`free_lag`, default 6 meses) — **caso mais comum** |
| `NonHistorical` | Tabela sem coluna de data confiável (data vem do `__TABLES__.last_modified_time` do BQ) — precisa de `compare_against="table_update"` no `check_for_update` também (ver "Casos especiais" abaixo) |

`date_column`/`date_format` (quando aplicável) precisam casar entre si — `DateOnly`↔`%Y-%m-%d`, `YearMonth`↔`%Y-%m`, `YearQuarter`↔`%Y-%m`, `YearOnly`↔`%Y`. `.model_dump()` sempre no final — `coverage` é um `dict`, não o objeto Pydantic.

## Passo 4 — `flows.py`: a fiação

```python
"""
Flows para br_ans_beneficiario — Prefect 3.
"""

from prefect import flow

from pipelines.datasets.br_ans_beneficiario.constants import (
    DATASET_ID,
    INFORMACAO_CONSOLIDADA_TABLE_ID,
)
from pipelines.datasets.br_ans_beneficiario.tasks import (
    br_ans_beneficiario_check_for_update,
    br_ans_beneficiario_download,
)
from pipelines.utils.stage_dispatch import (
    CheckThenDownloadPipeline,
    Etapa,
    deploy_tags,
)

_informacao_consolidada_pipeline = CheckThenDownloadPipeline(
    dataset_id=DATASET_ID,
    table_id=INFORMACAO_CONSOLIDADA_TABLE_ID,
    check_for_update=br_ans_beneficiario_check_for_update,
    download_data=br_ans_beneficiario_download,
    date_format="%Y-%m",  # granularidade da comparação de coverage
)


@flow(
    name=_informacao_consolidada_pipeline.check_update_flow_name,
    log_prints=True,
)
def br_ans_beneficiario_informacao_consolidada_check_update_flow() -> None:
    _informacao_consolidada_pipeline.run_check_update()


# pyrefly: ignore [missing-attribute]
br_ans_beneficiario_informacao_consolidada_check_update_flow.deploy_tags = (
    deploy_tags(DATASET_ID, Etapa.CHECK_UPDATE)
)


@flow(
    name=_informacao_consolidada_pipeline.download_flow_name,
    log_prints=True,
)
def br_ans_beneficiario_informacao_consolidada_download_flow(
    download_params: dict,
) -> None:
    _informacao_consolidada_pipeline.run_download(download_params)


# pyrefly: ignore [missing-attribute]
br_ans_beneficiario_informacao_consolidada_download_flow.deploy_tags = (
    deploy_tags(DATASET_ID, Etapa.DOWNLOAD)
)
_informacao_consolidada_pipeline.download_deployment = (
    br_ans_beneficiario_informacao_consolidada_download_flow.fn.__name__
)
```

Pontos que sempre se repetem, nessa ordem:
1. `_x_pipeline = CheckThenDownloadPipeline(...)` — passa as duas funções do Passo 2/3.
2. `@flow(name=_x_pipeline.check_update_flow_name, ...)` — nunca monta o nome à mão.
3. `.deploy_tags = deploy_tags(DATASET_ID, Etapa.CHECK_UPDATE)` logo depois do flow.
4. Repete 2-3 pro `download_flow`.
5. **Por último**, `_x_pipeline.download_deployment = <download_flow>.fn.__name__` — sempre depois que a função existir, nunca uma string digitada à mão (ver "Por quê" em [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]]).

## Quando o dataset tem mais de uma tabela — `pipeline_factory`

Se as tabelas compartilham a mesma lógica (só variando `table_id`), `CheckThenDownloadPipeline(...)` repetido por tabela ficaria idêntico. `pipeline_factory` resolve isso — fixa o que é comum, devolve uma função `table_id -> CheckThenDownloadPipeline`.

`tasks.py` (`br_ibge_ipca`, 4 tabelas):

```python
def make_check_for_update(table_id: str) -> Callable[[], CheckResult]:
    def check_for_update() -> CheckResult:
        ...
        return CheckResult(reference_date=reference_date)
    return check_for_update


def make_download_data(table_id: str) -> Callable[[dict], DownloadResult]:
    def download_data(download_params: dict) -> DownloadResult:
        ...
        return DownloadResult(...)
    return download_data


make_pipeline = pipeline_factory(
    DATASET_ID,
    make_check_for_update,
    make_download_data,
    date_format="%Y-%m",  # kwarg comum a todas as tabelas
)
```

`make_pipeline` é **sempre a última definição de `tasks.py`** — nunca em `flows.py` (misturaria fiação com composição de lógica) nem em `constants.py` (causaria import circular).

`flows.py` — cada tabela ganha sua própria seção, com banner de comentário mostrando os 2 nomes de flow completos:

```python
from pipelines.datasets.br_ibge_ipca.tasks import make_pipeline

# ──────────────────────────────────────────────────────────────────────────────
# mes_brasil
# check_update: br_ibge_ipca__mes_brasil
# download: br_ibge_ipca__mes_brasil
# ──────────────────────────────────────────────────────────────────────────────

_mes_brasil_pipeline = make_pipeline(MES_BRASIL_TABLE_ID)


@flow(name=_mes_brasil_pipeline.check_update_flow_name, log_prints=True)
def br_ibge_ipca_mes_brasil_check_update_flow() -> None:
    _mes_brasil_pipeline.run_check_update()

# ... deploy_tags, @flow de download, download_deployment — mesmo padrão do passo 4
```

**Nunca fábrica de `@flow` compartilhada** — cada tabela escreve os 2 `@flow` explicitamente. Só `tasks.py` (`make_check_for_update`/`make_download_data`/`make_pipeline`) usa fábrica. Motivo: `download_deployment` deriva de `.fn.__name__`, que seria o mesmo literal pra todas as instâncias se viesse de um template de closure compartilhado.

**Check compartilhado entre tabelas** (a fonte não distingue por tabela, ex. `br_me_caged`): passa `lambda _: minha_funcao_unica` no lugar de `make_check_for_update`.

## Casos especiais

- **Tabela `NonHistorical`** (sem coluna de data confiável, ex. `br_me_cnpj.simples`): `coverage=NonHistorical().model_dump()` + `compare_against="table_update"` no `CheckThenDownloadPipeline(...)` (ou como override na chamada de `make_pipeline(TABLE_ID, compare_against="table_update")`, se as outras tabelas do dataset usam o default).
- **Dado particionado**: `partition_folders` no `DownloadResult`, `data_path` aponta pra pasta base (não o arquivo), mantendo a estrutura `chave=valor/` — o BigQuery detecta partição Hive sozinho a partir disso.

## `job_variables` — dimensionar o pod

Opcional, por flow — sem isso, usa o default do work pool. Setado do mesmo jeito que `deploy_tags`:

```python
# Pico medido em produção: ~1.78Gi. ~1.7x de margem.
br_ans_beneficiario_informacao_consolidada_download_flow.job_variables = {
    "memory": "3Gi"
}
```

`check_update` costuma poder ser bem mais leve que `download` (é só um poll de metadado) — mas hoje nenhum dataset migrado diferencia isso ainda, é uma pendência em aberto (ver `pendencia-perfil-recursos-por-etapa.md`).

## Validar antes de considerar pronto

Checklist completo (ruff/pyrefly/import de sanidade + checagem de colisão de nome) já documentado em [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]], seção "Validação mínima" — não repetido aqui.

**Se está substituindo um flow monolítico antigo com deployments reais em produção**: remover o código antigo do arquivo por completo antes de qualquer deploy pela sua branch — `deploy_flows.py --files <arquivo>` redeploya todo `Flow` que encontrar no arquivo, usando a branch informada como fonte git. Deixar o flow antigo ali sobrescreveria silenciosamente a fonte do deployment real (que deveria continuar em `main`) pra apontar pra sua branch de trabalho.

## Testar de verdade, sem tocar em produção

1. Deploy só do seu `flows.py`, pool `basedosdados-dev`, sem schedule (deploy em dev nunca ativa schedule automaticamente).
2. Disparar o `check_update` manualmente (`run_deployment`, ou pela UI do Prefect).
3. Acompanhar os logs — se houver dado novo, o `check_update` deve disparar o `download` sozinho (via `run_deployment()`, aparece linkado como subflow na árvore do Prefect UI), que por sua vez dispara o `mat_test`.
4. Se a coverage já estiver em dia (sem dado novo pra detectar), isso **não é um bug** — é o comportamento esperado. Forçar um teste completo exige rolar a coverage do backend pra trás (decisão que precisa de aprovação explícita, mexe em dado real do backend).

Ver [[plano-de-testes-datasets-migrados]] pra exemplos reais dessa validação (incluindo o primeiro caminho completo provado, `br_ibge_ipca.mes_brasil`).

## Ver também

- [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]] — nomenclatura, convenções, checklist de validação.
- [[migracao-lote-10-datasets]] — 10 exemplos reais migrados, com as particularidades de cada um.
- [[plano-de-testes-datasets-migrados]] — como testar contra o Prefect/backend real.
