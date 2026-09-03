# Encapsular check_update + flow_download num helper genérico

Continuação da issue #1867, depois do POC (`feat/event-pipeline-automations-poc`, ver `issue-1867-pipeline-eventos.md` neste mesmo diretório pro histórico completo até aqui). Objetivo desta etapa: fechar bem a variante **padrão** (check_update e flow_download como estágios separados, ~51% dos datasets) antes de adaptar pro `check_and_download` (variante fundida, ~49%).

## Motivação

O piloto (`pipelines/datasets/test_event_pipeline/flows.py`) prova a mecânica, mas repete boilerplate que qualquer dataset novo nesse padrão teria que copiar: a chamada de `rename_flow_run_dataset_table` em cada estágio, e o dispatch pro próximo estágio via `run_deployment()`. Antes de migrar um dataset real ou generalizar pro `check_and_download`, queremos um helper genérico que receba só a lógica específica do dataset (como checar a fonte, como baixar) e cuide do resto.

## Restrição técnica descoberta

`deploy_flows.py` só reconhece um `Flow` como deployável se a função dele estiver definida **no próprio arquivo `flows.py` do dataset**:
```python
isinstance(obj, Flow) and obj.fn.__code__.co_filename == str(path.resolve())
```
Isso impede uma fábrica genérica gerar o objeto `@flow` inteiro escondida num módulo compartilhado — cada dataset precisa manter suas duas funções `@flow` finas no próprio `flows.py`. O que pode ser generalizado é tudo o que roda **dentro** delas.

## Desenho proposto

`CheckThenDownloadPipeline`, uma classe em `pipelines/utils/stage_dispatch.py`, com dois contratos tipados pra troca de dados entre os estágios:

```python
@dataclass
class CheckResult:
    reference_date: date
    extra_download_params: dict = field(default_factory=dict)

@dataclass
class DownloadResult:
    coverage: dict
    targets: list[str] = field(default_factory=lambda: ["dev", "prod"])
    bq_project: str = "basedosdados"
    prefect_mode: str = "prod"
    partition_folders: list[str] | None = None

class CheckThenDownloadPipeline:
    def __init__(self, *, dataset_id, table_id, check_fn: Callable[[], CheckResult],
                 download_fn: Callable[[dict], DownloadResult],
                 prefect_dataset_id: str | None = None, env: str = "prod"):
        ...

    def run_check_update(self) -> bool:
        """rename + check_fn() + poll/commit/dispatch (run_deployment do flow_download)."""

    def run_download(self, download_params: dict) -> None:
        """rename + download_fn(download_params) + dispatch pro mat_test (generaliza o
        dispatch_mat_test que hoje só existe dentro de test_event_pipeline/tasks.py)."""
```

`flows.py` de cada dataset fica reduzido a:
```python
_pipeline = CheckThenDownloadPipeline(
    dataset_id=DATASET_ID, table_id=TABLE_ID,
    check_fn=minha_logica_de_check,
    download_fn=minha_logica_de_download,
)

@flow(name=f"{PREFECT_DATASET_ID}: check_update", log_prints=True)
def check_update_flow() -> None:
    _pipeline.run_check_update()
check_update_flow.deploy_tags = deploy_tags(PREFECT_DATASET_ID, "check_update")

@flow(name=f"{PREFECT_DATASET_ID}: flow_download", log_prints=True)
def flow_download_flow(download_params: dict) -> None:
    _pipeline.run_download(download_params)
flow_download_flow.deploy_tags = deploy_tags(PREFECT_DATASET_ID, "flow_download")
```

**Por que classe, não funções soltas:** evita repetir `dataset_id`/`table_id`/`prefect_dataset_id`/`env` nas duas chamadas — instancia uma vez, os dois métodos já sabem quem são.

**Pensando à frente pro `check_and_download`:** os contratos tipados (`CheckResult`/`DownloadResult`) tornam a variante fundida natural depois — um terceiro método que chama `check_fn()` e, se houver dado novo, chama `download_fn()` na mesma execução (sem `run_deployment` no meio), reaproveitando o mesmo `DownloadResult` → dispatch pro `mat_test`.

## Ordem de execução

1. Escrever `CheckThenDownloadPipeline` em `stage_dispatch.py`, migrando `dispatch_mat_test` pra lá também (hoje só existe dentro do piloto, precisa generalizar).
2. Refatorar `test_event_pipeline` pra usar a classe, confirmar que o teste ponta a ponta (dev→prod) continua passando — mesmo comportamento, só reorganizado.
3. Só depois: aplicar num dataset real, ou generalizar pro `check_and_download`.

## Recursos de pod por etapa

Implementado junto (issue #1867 original já previa isso: "recursos idênticos pra cargas diferentes" era um dos problemas motivadores). `job_variables` por etapa, mesma convenção de `deploy_tags`:

- `check_update`/`flow_download`: variam por dataset (tamanho de download difere muito entre datasets reais), então ficam declarados no `constants.py` de cada um — não centralizado, pra não impor um tier "genérico" que não serve pra todo mundo.
- `mat_test`: é um deployment único compartilhado por todos os datasets, então o tier mora em `pipelines/utils/metadata/constants.py`, junto do resto.

Tiers usados no piloto: `check_update` 500m CPU / 512Mi limit (256Mi request — precisa ser explicitamente menor que o default do pool, 1Gi, senão o Kubernetes rejeita request > limit), `flow_download` 1 CPU / 2Gi, `mat_test` 2Gi (default do pool era 4Gi; baixado depois de medir ~1GB de uso real — ver seção de teste abaixo).

## `Etapa(StrEnum)`

`check_update`/`flow_download`/`mat_test` eram strings soltas repetidas em `deployment_name`/`deploy_tags`/`etapa_tag`. Virou `Etapa(StrEnum)` em `stage_dispatch.py` — protege contra typo silencioso (um `"check_updte"` digitado errado não gerava erro nenhum antes, só um nome de deployment quebrado que falhava tarde, no `run_deployment()`). `StrEnum` (não `Enum` puro) pra continuar funcionando direto em f-string/chave de dict sem precisar de `.value`.

## Teste ponta a ponta (2026-09-02/03)

Deploy de `check_update_flow`/`flow_download_flow`/`mat_test_flow` no pool `basedosdados` (prod), branch `feat/event-pipeline-automations-poc` (commit `2e9bc38f`). Disparado `check_update_flow` manualmente — cadeia completa:

`check_update_flow` (COMPLETED) → `flow_download_flow` (COMPLETED) → `mat_test_flow` (COMPLETED)

Confirma: a refatoração pra `CheckThenDownloadPipeline` não mudou o comportamento (mesmo resultado do teste anterior de 09-01), e o `memory_limit` de 2Gi do `mat_test` (metade do default de 4Gi do pool) rodou sem OOM — o dbt em dev + transferência + dbt em prod coube tranquilo nesse limite.

## Teste com dados particionados — `test_event_pipeline_partitioned` (2026-09-02/03)

O piloto original (`test_event_pipeline`) usa um único arquivo sem partição — nunca exercitou `DownloadResult.partition_folders` nem o caminho de `transfer_files_to_prod_flow(folders=...)` que promove só a fatia nova, não o staging inteiro. Criado um segundo piloto sintético, `pipelines/datasets/test_event_pipeline_partitioned/`, com dados em `ano=<ano>/mes=<mes>/dados.csv` (Hive, 2 níveis) — mesma tabela `test_dataset` do backend, tabela nova (`test_event_pipeline_partitioned`, colunas `ano`/`mes` marcadas `is_partition`).

Encontrados e corrigidos **4 problemas reais** até fechar o teste ponta a ponta:

### 1. Bug de fundo: `poll_source_for_update` quebra pra qualquer tabela sem `RawDataSource`

`client.upsert_raw_source_poll` resolve `rawDataSource_Id` via `_raw_source_id()`, que devolve `None` quando a tabela não tem `RawDataSource` vinculado. Esse `None` é repassado direto pra uma segunda query (`allPoll(rawDataSource_Id: null)`) — só que o filtro GraphQL **ignora por completo** um valor `null` (comportamento comum de filtro Django-filter: `None` = "não filtra", não "filtra onde é nulo"). Resultado: a query devolve *todos* os `Poll`s do backend, e a checagem de "no máximo 1 resultado" (`_query_id`) sempre estoura, porque a base de prod tem centenas deles.

**Isso não é uma condição rara** — acontece pra qualquer dataset sem `RawDataSource`, garantido, na primeira chamada. Corrigido pra nós criando um `RawDataSource` sintético e vinculando à tabela (`_raw_source_id` filtra certo por `tables_Id` quando existe um). O bug de fundo em `client.py` continua não corrigido — candidato a issue nova, mesmo padrão de #1940/#1943.

### 2. ⚠️ Achado mais estranho: faltava o ponteiro da tabela no BigQuery em `basedosdados-dev`

Esse foi o mais intrigante de diagnosticar. `upload_to_gcs`/`bd.Table.create()` sobe o arquivo certinho pro bucket `gs://basedosdados-dev/...`, mas **registra a tabela externa do BigQuery (o "ponteiro" que permite fazer `SELECT`) só dentro do projeto `basedosdados-staging`** — fixo, independente do `bucket_name`/`billing_project_id` passado pro `Table()`. Isso é definido em `~/.basedosdados/config.toml` (`gcloud-projects.staging.name`), não é algo controlável por parâmetro na chamada.

O problema: a macro `set_datalake_project` resolve o projeto de forma diferente por target do dbt —
```
target=dev  → projeto basedosdados-dev
target=prod → projeto basedosdados-staging
```
Ou seja, o `mat_test_flow` (que roda `target=dev` primeiro) espera achar a tabela em `basedosdados-dev`, mas o `bd.Table.create()` só a registra em `basedosdados-staging`. **O piloto original só funcionava porque alguém, em algum momento da sessão do notebook, criou manualmente uma segunda cópia da tabela externa em `basedosdados-dev`** — sem isso nunca ter virado parte do código genérico nem sido documentado. Qualquer dataset novo nesse fluxo bate na mesma parede.

Corrigido criando manualmente o segundo ponteiro via `google-cloud-bigquery` direto (mesmo schema/config da cópia real, incluindo `HivePartitioningOptions(mode="STRINGS")` pro caso particionado). Fica como pendência real de processo — o `upload_to_gcs` genérico deveria criar as duas cópias sozinho, ou o pipeline deveria assumir só uma delas. Candidato a issue nova.

### 3. Erro meu: `ano`/`mes` duplicados dentro do CSV

Escrevi o CSV com `ano,mes,reference_date` como colunas, além de codificar `ano=/mes=` no caminho da pasta — violando a convenção Hive (a coluna de partição só deveria existir no caminho). Isso só quebrou no lado com particionamento Hive de verdade (`basedosdados-staging`, criado via `bd.Table.create()`): o BigQuery descarta a duplicata do schema declarado que colide com o particionamento automático, sobrando só `reference_date` como coluna de arquivo esperada — `Database Error: Too many values in line. Found 3 column(s) when expecting 1.`. A cópia manual em `basedosdados-dev` (sem particionamento Hive, schema flat) não expôs o erro, por isso só apareceu no lado prod. Corrigido: `write_partitioned_csv` agora só escreve `reference_date` no arquivo.

### 4. Status `published` em vez de `under_review`

`assert_write_allowed` (`pipelines/utils/metadata/policy.py`) trava: gravar metadados em `env=prod` com `bq_project != "basedosdados"` só é permitido se a tabela estiver `under_review` — proteção pra não marcar dado de teste/dev como "publicado" de verdade. Cadastrei a tabela nova como `published` por engano; a tabela original sempre foi `under_review`. Datasets reais publicados não são afetados (sempre passam `bq_project="basedosdados"`, a condição nunca fica verdadeira).

**Resultado final:** cadeia completa `check_update_flow` → `flow_download_flow` → `mat_test_flow`, todas `COMPLETED`, dado certo nas duas tabelas (dev confirmado via query direta: `{ano: 2026, mes: 9, reference_date: 2026-09-03}`, tipos corretos via `safe_cast`).

## Consolidação em `test_dataset/` (2026-09-02/03)

Depois do teste particionado acima, ficou claro um problema de organização: `test_dataset/`, `test_event_pipeline/` e `test_event_pipeline_partitioned/` eram três pastas de topo diferentes, mas as duas últimas são só tabelas do **mesmo** `dataset_id` de sandbox (`test_dataset`) no backend — nenhuma razão pra ter `flows.py`/`tasks.py`/`constants.py` duplicados por piloto. A convenção real do repo (ver `pipelines/datasets/br_ms_sih/flows.py`: várias tabelas, um `flows.py` só) já resolvia isso.

Consolidado: `test_event_pipeline/` e `test_event_pipeline_partitioned/` foram apagadas, e os dois pilotos passaram a viver dentro de `pipelines/datasets/test_dataset/` — `constants.py`/`tasks.py`/`flows.py` únicos, cada piloto numa seção com banner de comentário separando do resto (inclusive dos testes de `download_data_to_gcs` que já moravam ali).

Isso expôs uma restrição nova: `deploy_flows.py` usa o **nome da variável do módulo** como nome literal do deployment (`for name, obj in vars(module).items()`), e o desenho original de `CheckThenDownloadPipeline`/`deployment_name()` assumia que esse nome era sempre `<etapa>_flow` — o que só funciona com um dataset por arquivo. Com dois pilotos no mesmo `flows.py`, as variáveis precisam de nomes próprios (`event_pipeline_check_update_flow`, `event_pipeline_partitioned_check_update_flow`, etc.), e o `check_update` de cada um precisa saber dispatchar pro `flow_download` certo — não mais o nome-padrão adivinhado.

Solução: novo parâmetro `flow_download_deployment: str | None` em `CheckThenDownloadPipeline`, setado como atributo **depois** que o flow existe, a partir do `.fn.__name__` da própria função:

```python
_event_pipeline.flow_download_deployment = (
    event_pipeline_flow_download_flow.fn.__name__
)
```

Deliberadamente não é uma string digitada solta no construtor — isso reintroduziria o mesmo risco que `Etapa(StrEnum)` foi criado pra eliminar (duas grafias do mesmo nome podendo divergir silenciosamente). Derivar de `.fn.__name__` (não `.__name__` direto no objeto `Flow` — o `pyrefly` acusa `missing-attribute`, já que `Flow` não declara esse atributo, mesmo funcionando em runtime) garante que o nome usado no dispatch é sempre o nome real da função.

Aproveitado pra também simplificar `deploy_tags`: a tag da etapa virou só o nome dela (`"check_update"`, `"flow_download"`), sem o prefixo `"etapa:"` que só poluía a tag sem agregar informação.

Commit `b9f0ac5f` (branch `feat/event-pipeline-automations-poc`). Depois do commit: os 9 flows do arquivo (5 de `download_data_to_gcs` + 4 dos dois pilotos de evento) foram redeployados, os 4 deployments órfãos das pastas antigas foram removidos, e a cadeia completa foi testada de ponta a ponta de novo — dessa vez forçando dado novo de propósito (coverage de ambas as tabelas rolada pra ontem no backend) pra validar o dispatch automático com os nomes novos, não só a lógica interna:

`check_update_flow` (ambos, `COMPLETED`, detectou `has_new_data=True`) → dispatch automático pro `flow_download` **pelo nome derivado de `.fn.__name__`** (`event_pipeline_flow_download_flow` / `event_pipeline_partitioned_flow_download_flow`, ambos `COMPLETED`) → dispatch automático pro `mat_test_flow` compartilhado (ambos `COMPLETED`).

Confirma que a correção funciona em execução real e não só em checagem estática/unitária.

## Status

Passos 1 e 2 do plano concluídos: `CheckThenDownloadPipeline` escrita, os dois pilotos (`event_pipeline`/`event_pipeline_partitioned`) consolidados dentro de `test_dataset/`, teste ponta a ponta revalidado com sucesso — inclusive dispatch automático com os nomes de deployment novos. 4 problemas reais encontrados e corrigidos no piloto particionado (2 deles são bugs/lacunas de processo genéricos, candidatos a issue separada — ver seções 1 e 2 acima), mais o ajuste de `flow_download_deployment` pra suportar múltiplos pilotos por arquivo. Próximo passo (passo 3): aplicar num dataset real, ou generalizar pro `check_and_download`.
