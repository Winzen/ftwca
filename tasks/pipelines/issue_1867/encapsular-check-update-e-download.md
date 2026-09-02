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

## Status

Fase de desenho — ainda não implementado. Próxima sessão retoma no passo 1.
