# `stage_dispatch.py` — racional de design

Conteúdo que morava nas docstrings de `pipelines/utils/stage_dispatch.py`
antes de enxugar pra Google style de verdade (2026-09-29) — resumo curto +
`Args:`/`Returns:`/`Attributes:`, sem prosa explicando o porquê das
decisões. Esse "porquê" foi movido pra cá, não descartado. Complementa
[[build-and-promote-racional-de-design]] (mesmo tratamento, arquivo
diferente) e [[revisao-laura-pr-1932]] (onde `compare_against` e
`bq_project` já tinham sido documentados a fundo — não repetidos aqui).

## `Etapa(StrEnum)` — por que `StrEnum`, não `Enum` puro

Os membros já são `str` de verdade, então funcionam direto em
f-string/chave de dict sem precisar de `.value` em lugar nenhum — só ganha
a proteção extra de um typo virar `AttributeError` na hora
(`Etapa.CHECK_UPDTE`) em vez de gerar silenciosamente um nome de
deployment errado que só falha lá na frente, no `run_deployment()`. Mesmo
padrão de `DateFormat` em `pipelines.utils.metadata.domain`.

## `deploy_tags` — por que a tag da etapa não tem prefixo

`[str(etapa), f"dataset:{dataset_id}"]` — a tag da etapa é só o nome dela
(`check_update`/`extract_and_load`/`build_and_promote`), sem prefixo
`etapa:`. `dataset:X` já deixa claro que a outra tag é a etapa; o prefixo
só poluía sem agregar informação.

## `deploy_tags` — tag `event-pipeline`, dívida técnica reconhecida (2026-10-05)

`deploy_tags()` agora sempre inclui `"event-pipeline"` na lista retornada
— pedido do usuário pra achar no Prefect UI, de uma vez, todo flow que já
passou pela migração pro pipeline orientado a eventos (issue #1867),
diferenciando de flows ainda no formato monolítico antigo.

**Isso é um acoplamento errado, reconhecido conscientemente**: usar
`deploy_tags()` é uma convenção (cada `flows.py` chama ela manualmente,
numa linha separada depois da definição do `@flow`), não uma garantia —
nada impede um dataset migrado de setar `.deploy_tags` de outro jeito e
ficar sem a tag, nem um flow não migrado de chamar `deploy_tags()` por
engano e ganhar a tag sem ser. O lugar logicamente certo seria
`CheckThenExtractLoadPipeline` aplicar a tag sozinha nos flows que ela
ajuda a criar — mas ela não tem acesso ao objeto `Flow` (o `@flow` é
aplicado fora da classe, no `flows.py` de cada dataset; ver seção "por
que o `@flow` não é gerado pela classe" acima), então isso exigiria
trocar a API de uso da cápsula (ex. `_pipeline.deploy_tags(Etapa.X)` no
lugar da função solta) — mudança que toca as ~26 chamadas já existentes
nos 7 datasets migrados em #1932, considerada arriscada demais pra fazer
perto do merge só por causa de uma tag.

Decisão: manter como está por enquanto (serve pro propósito imediato —
achar os migrados no Prefect UI), revisar quando a API da cápsula for
mexida de novo por outro motivo. Mencionar essa dívida no PR #1932
quando ele for atualizado.

## `_flow_name` — por que existe como função separada

Um lugar só define o formato `"<etapa>: <dataset_id>"` — usado tanto por
`deployment_name()` (resolve o `run_deployment(name=...)`) quanto pelas
propriedades `check_update_flow_name`/`extract_and_load_flow_name` de
`CheckThenExtractLoadPipeline` (o que de fato vira o `@flow(name=...)`).
Evita que onde o flow é declarado e onde é referenciado no dispatch
divirjam silenciosamente.

## `deployment_name` — convenções por trás dos parâmetros

- `etapa`: as etapas que não são `build_and_promote` seguem
  `@flow(name="<etapa>: <dataset_id>")` numa função chamada literalmente
  `<etapa>`, **sem** sufixo `_flow` (ver `pipelines/datasets/br_ibge_ipca/flows.py`)
  — é o default de `deployment` quando não informado.
- `deployment`: sobrescreve a segunda metade (depois da barra) quando a
  variável do flow não se chama literalmente `<etapa>`. Necessário quando
  vários datasets/pilotos compartilham o mesmo arquivo `flows.py` —
  `deploy_flows.py` descobre flows pelo nome da variável no módulo
  (`vars(module)`), então duas pipelines no mesmo arquivo não podem ter as
  duas uma variável `check_update` (a segunda sobrescreveria a primeira no
  namespace do módulo, e `deploy_flows.py` nunca veria a primeira). Cada
  uma precisa de um nome de variável próprio, que precisa bater com o que
  foi de fato implantado — ver `pipelines/datasets/test_dataset/flows.py`.

## `check_update_and_dispatch` — `prefect_dataset_id` vs `dataset_id`/`table_id`

`dataset_id`/`table_id` são a identidade real no backend/BigQuery.
`prefect_dataset_id` é só a convenção de nome usada por `deployment_name()`
pra resolver qual deployment chamar em seguida — as duas coisas podem
divergir. Isso não é caso raro: é a convenção padrão do repo inteiro pra
qualquer dataset com mais de uma tabela (`f"{dataset_id}__{table_id}"`,
ver `br_bcb_agencia__agencia`/`br_denatran_frota__uf_tipo`/
`br_rf_cno__{table_id}` em `pipelines/datasets/`), usada mesmo quando há
só uma tabela. `CheckThenExtractLoadPipeline` deriva esse valor sozinho
(ver `__init__` abaixo). `poll_source_for_update_task`/
`commit_source_update_task` vêm de `pipelines.utils.metadata.tasks` — o
mesmo par usado pelos datasets reais (ver `br_bcb_estban/flows.py`).

## `pipeline_factory` — exemplo de `compare_against` divergente

`compare_against` não é kwarg da fábrica nem de
`CheckThenExtractLoadPipeline`: é campo de `SourceInspection`, devolvido
pelo `get_latest_update` de cada tabela — ex. `br_me_cnpj.simples`
devolveria `SourceInspection(..., compare_against="table_update")`
enquanto as demais tabelas do dataset usam o default `"coverage"` (ver
[[revisao-laura-pr-1932]] categoria 2 pro racional completo dessa decisão).

## `CheckThenExtractLoadPipeline` — por que o `@flow` não é gerado pela classe

O `@flow` em si continua tendo que ser definido no `flows.py` do próprio
dataset — não dá pra gerar via fábrica genérica, porque `deploy_flows.py`
só reconhece um `Flow` cuja função esteja definida no arquivo do dataset
(via `obj.fn.__code__.co_filename`). A classe só reduz o corpo de cada
`@flow` a uma chamada de método.

### `prefect_dataset_id` — convenção derivada no `__init__`

Sempre `f"{dataset_id}__{table_id}"` quando não informado — mesma
convenção usada em todo o repo (`br_bcb_agencia__agencia`,
`br_denatran_frota__uf_tipo`), usada mesmo quando há só uma tabela.
`prefect_dataset_id` continua aceito como override pra casos que fujam da
convenção.

### `extract_load_deployment` — por que só pode ser setado depois do flow existir

Quando vários pilotos/datasets dividem o mesmo `flows.py`
(`deploy_flows.py` descobre flows pelo nome da variável no módulo, então
duas pipelines no mesmo arquivo não podem ter as duas uma variável
`extract_and_load` — ver `pipelines/datasets/test_dataset/flows.py`), o
`extract_load_deployment` precisa ser setado **depois** que o flow existe,
a partir do `.fn.__name__` da própria função — nunca repetido como string
solta no construtor:

```python
_pipeline = CheckThenExtractLoadPipeline(...)

@flow(name=_pipeline.extract_and_load_flow_name, log_prints=True)
def meu_extract_and_load(download_params: dict) -> None:
    _pipeline.run_extract_and_load(download_params)
meu_extract_and_load.deploy_tags = deploy_tags(...)
_pipeline.extract_load_deployment = meu_extract_and_load.fn.__name__
```

**Não funciona com fábrica compartilhada de `@flow`**: se o `@flow` viesse
de uma closure reusada pra várias tabelas (`def extract_and_load(...)`
definida uma vez dentro de uma função-fábrica), todas as instâncias
teriam o **mesmo** `.fn.__name__` literal, mesmo atribuídas a variáveis de
módulo diferentes — uma função não sabe a que nome foi atribuída no
escopo de quem a criou. Por isso, quando um dataset tem várias tabelas
(ex. `br_ibge_ipca`, 4 tabelas), os `@flow` são sempre escritos
explicitamente, um por um, em `flows.py` — só a lógica em `tasks.py`
(`get_latest_update`/`extract_load_data`) usa fábrica, porque essas não
têm essa restrição. (Mesmo conteúdo já registrado em
[[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]], que usa nomes
antigos — pendente de atualização geral.)

Construir o nome como string solta no construtor não repete a mesma coisa
por acidente com duas grafias divergentes — mesmo risco que o
`Etapa(StrEnum)` evita.
