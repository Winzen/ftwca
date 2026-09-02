# Padrão de implementação — check_update e check_and_download

**Referência:** [[Plano de ação — pipeline orientado a eventos]]
**Data:** 2026-08-28

---

## O que muda em relação ao flow atual

O flow monolítico atual termina o check com um `return` quando não há dado novo, ou continua para download + dbt quando há. Na nova arquitetura, o `check_update` faz exatamente o mesmo check — a única mudança é o que acontece quando há dado novo:

| Antes | Depois |
|---|---|
| `return` se não há dado | `return` se não há dado (igual) |
| continua para download/dbt se há dado | `emit_event(...)` se há dado, então termina |

O flow fica menor, não maior. Tudo que vem depois do check vai para os outros flows.

---

## Variante padrão — check_update

Para datasets onde o check é uma chamada leve (API, HEAD, listagem FTP) sem precisar baixar o arquivo.

```python
from prefect import flow
from prefect.events import emit_event

from pipelines.utils.metadata.tasks import poll_source_for_update_task
# + imports específicos do dataset para o check

@flow(name="check_update__br_ibge_ipca__mes_brasil", log_prints=True)
def check_update(
    dataset_id: str = "br_ibge_ipca",
    table_id: str = "mes_brasil",
) -> None:
    # 1. Descobre o max_date (ou URL, ou hash) na fonte — específico do dataset
    max_date = check_for_updates(dataset_id=dataset_id, table_id=table_id)

    # 2. Compara com o que o BD já tem
    has_new_data = poll_source_for_update_task(
        dataset_id=dataset_id,
        table_id=table_id,
        source_max_date=max_date,
        env="prod",
        date_format="%Y-%m",
        compare_against="coverage",
    )

    # 3. Emite o evento se há dado novo; automação dispara o flow_download
    #    Se não há dado novo, o helper retorna sem emitir — automação não dispara
    emit_pipeline_data_available(dataset_id, table_id, has_new_data, reference_date=max_date)
```

---

## Variante check_and_download

Para datasets que precisam baixar o arquivo para checar se há dado novo. O flow faz check + download + upload para GCS em sequência, e só emite o evento ao terminar com sucesso.

```python
from prefect import flow
from prefect.events import emit_event

from pipelines.utils.metadata.tasks import poll_source_for_update_task
from pipelines.utils.tasks import upload_to_gcs
# + imports específicos do dataset

@flow(name="check_and_download__br_anatel_banda_larga_fixa", log_prints=True)
def check_and_download(
    dataset_id: str = "br_anatel_banda_larga_fixa",
    table_id: str = "microdados",
) -> None:
    # 1. Baixa o arquivo (necessário para descobrir a data/hash)
    filepath, reference_date = download_and_extract_date(
        dataset_id=dataset_id, table_id=table_id
    )

    # 2. Compara com o que o BD já tem
    has_new_data = poll_source_for_update_task(
        dataset_id=dataset_id,
        table_id=table_id,
        source_max_date=reference_date,
        env="prod",
        date_format="%Y-%m",
        compare_against="coverage",
    )

    if has_new_data:
        # Faz upload para GCS staging (o mat_test vai ler daqui)
        upload_to_gcs(
            data_path=filepath,
            dataset_id=dataset_id,
            table_id=table_id,
            bucket_name="basedosdados-dev",
            dump_mode="append",
        )

    # Emite o evento se há dado novo; automação dispara o mat_test
    emit_pipeline_data_available(dataset_id, table_id, has_new_data, reference_date=reference_date)
```

Na variante check_and_download o `mat_test` não precisa de nada dinâmico além de `dataset_id` e `table_id` — o dado já está no GCS staging.

---

## O que vai no payload

O payload precisa conter o que o flow downstream precisa para funcionar. O mínimo obrigatório é `dataset_id` e `table_id`. O que mais entra depende do dataset:

| Dataset | Campo extra | Por quê o downstream precisa |
|---|---|---|
| br_ibge_ipca / inpc / ipca15 | `reference_date` | `collect_data_utils(periodo=periodo)` precisa saber qual período baixar |
| br_rf_cnpj / br_me_cnpj | `reference_date` | Pasta WebDAV muda por mês |
| br_anatel | — | URL é sempre a mesma; check_and_download, mat_test não precisa de nada extra |
| br_denatran_frota | `reference_date` | Data do arquivo a baixar |
| Datasets com URL fixa | — | Nada além de dataset/table |

Regra geral: se o `flow_download` precisaria de um parâmetro para saber *o quê* baixar, esse parâmetro vai no payload. Se o `flow_download` sempre baixa a mesma coisa (URL fixa, FTP com listagem dinâmica que ele faz sozinho), payload fica só com `dataset_id` e `table_id`.

---

## Como a automação usa o payload

A automação do Prefect 3 usa Jinja para interpolar `event.payload` nos parâmetros do `RunDeployment`. Exemplo para o `flow_download` do ibge_ipca:

```python
# automação — RunDeployment parameters
{
    "dataset_id":     "{{ event.payload.dataset_id }}",
    "table_id":       "{{ event.payload.table_id }}",
    "reference_date": "{{ event.payload.reference_date }}",
}
```

Para datasets onde o payload só tem `dataset_id` e `table_id`, a automação pode até usar valores estáticos (hardcoded na automação), sem Jinja.

---

## O que NÃO muda no check_update

- A lógica de check em si — `check_for_updates()`, `get_date_api()`, listagem FTP, HEAD request etc. — fica intacta
- `poll_source_for_update_task()` continua sendo chamado da mesma forma
- Nenhum comportamento novo para o caso "sem dado novo" — continua sendo um `return`

A única adição é o `emit_pipeline_data_available()` no caminho feliz.

---

## Helper compartilhado: emit_pipeline_data_available

O `emit_event` não vai direto em cada flow — tem um helper em `pipelines/utils/flows.py` que centraliza o nome do evento e o formato do resource. Se o nome do evento ou o formato mudar, muda em um lugar só.

```python
# pipelines/utils/flows.py

from prefect.events import emit_event

def emit_pipeline_data_available(
    dataset_id: str,
    table_id: str,
    has_new_data: bool,
    **extra_payload,
) -> None:
    if not has_new_data:
        return
    emit_event(
        event="bd.pipeline.data-available",
        resource={"prefect.resource.id": f"pipeline/{dataset_id}/{table_id}"},
        payload={"dataset_id": dataset_id, "table_id": table_id, **extra_payload},
    )
```

Uso em cada flow — sem nenhum `if` no flow:

```python
from pipelines.utils.flows import emit_pipeline_data_available

@flow(name="check_update__br_ibge_ipca__mes_brasil", log_prints=True)
def check_update(dataset_id: str = "br_ibge_ipca", table_id: str = "mes_brasil") -> None:
    max_date = check_for_updates(dataset_id=dataset_id, table_id=table_id)
    has_new_data = poll_source_for_update_task(
        dataset_id=dataset_id,
        table_id=table_id,
        source_max_date=max_date,
        env="prod",
        date_format="%Y-%m",
        compare_against="coverage",
    )
    emit_pipeline_data_available(dataset_id, table_id, has_new_data, reference_date=max_date)
```

**Por que não uma fábrica (`make_check_update_flow`)?** A repetição real entre todos os `check_update` é só o nome do evento e o formato do resource — 4 linhas. O helper resolve isso sem indireção. Uma fábrica valeria se houvesse mais lógica compartilhada (retry, tratamento de exceção específico), mas por enquanto não tem. O flow fica legível e direto sem precisar saber o que a fábrica faz internamente.
