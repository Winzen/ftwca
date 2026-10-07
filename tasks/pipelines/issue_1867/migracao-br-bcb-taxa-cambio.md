# Migração `br_bcb_taxa_cambio` — prova de facilidade (2026-09-30)

Escolhido como exemplo de dataset **leve** pra medir o quão fácil é migrar um
dataset novo pro pipeline orientado a eventos (#1867), fora do lote da PR
#1932 — não faz parte dela, é só uma prova.

## Por que era um bom candidato

- Única tabela (`taxa_cambio`), sem `constants.py`/`tasks.py` próprios antes
  (tudo cabia no `flows.py` monolítico, 143 linhas).
- `get_source_max_date()` (`pipelines/crawler/bcb_taxa_cambio/utils.py`) já é
  um check leve e independente de verdade: consulta só o dólar, janela de 15
  dias — não baixa o ano inteiro pra checar.
- `treat_data_taxa_cambio` já escreve partição Hive de verdade
  (`to_partitions(..., partition_columns=["ano"], ...)`) — a automação de
  `partition_folders` (`discover_partition_folders`, ver
  [[revisao-laura-pr-1932]]) funciona aqui de graça, sem o dataset precisar
  declarar nada.
- Coverage já era um `PartBdpro` simples, sem particularidade.

## O que foi feito

- `constants.py` novo: `DATASET_ID`/`TAXA_CAMBIO_TABLE_ID`/`COVERAGE` (mesmo
  `PartBdpro` que já existia).
- `tasks.py` novo: `taxa_cambio_get_latest_update()` (chama
  `get_source_max_date()`) + `taxa_cambio_download()` (chama
  `get_data_taxa_cambio(ano=ref.year)` + `treat_data_taxa_cambio()`, devolve
  `ExtractAndLoad` sem `partition_folders` — automático).
- `flows.py` reescrito: 1 flow monolítico → 2 (`check_update`/`extract_and_load`)
  via `CheckThenExtractLoadPipeline`, mesmo padrão do `br_inmet_bdmep`.
- De quebra, corrigido `treat_data_taxa_cambio` (`pipelines/crawler/bcb_taxa_cambio/tasks.py`):
  anotação de retorno errada (`-> str`, devolvia `dict` de verdade),
  suprimida com `# pyrefly: ignore [bad-return]` — corrigida na raiz, mesmo
  padrão da revisão do CodeRabbit.

**Tempo real**: ~15-20 minutos, como estimado antes de começar — confirma a
hipótese de que datasets "Padrão" (check leve e independente, 1 tabela) são
rápidos de migrar.

Validado: `ruff`/`py_compile`/`pyrefly` limpos, import de sanidade real
(`load_flows_from_file` reconhece os 2 flows, tags corretas: `['check_update',
'br_bcb_taxa_cambio', 'br_bcb_taxa_cambio__taxa_cambio']` e a variante
`extract_and_load`). **Não deployado nem testado contra Prefect real** — só
código local, é uma prova de conceito, não faz parte de nenhuma PR aberta.

## Pendência: parâmetro `anos` (backfill de vários anos) não tem equivalente direto

O flow monolítico antigo aceitava `anos: list[int]` — recarregava vários anos
numa chamada só (conserto de partição incompleta/duplicada, sem consultar a
fonte). A arquitetura nova não tem esse conceito: `extract_and_load` é sempre
disparado pra 1 período por vez (via `download_params["reference_date"]`).

**Como isso se resolve hoje, sem código novo**: invocação manual direta do
deployment `extract_and_load`, uma vez por ano, passando
`download_params={"reference_date": f"{ano}-12-31"}` — mesmo padrão já usado
nesta sessão pra forçar testes em datasets já migrados (`br_ibge_ipca`,
`br_ms_cnes`, etc.). Funciona, só não é "1 chamada pra N anos" — pra esse
dataset especificamente (dado pequeno, poucos anos de histórico) o custo de
rodar N vezes é desprezível.

**Decisão pendente, registrar antes de migrar de verdade** (não decidida
ainda, fica como pendência pra quando este dataset for migrado pra valer,
fora desta prova): se vale a pena adicionar um parâmetro de backfill
genérico na arquitetura orientada a eventos (ex. `extract_and_load` aceitar
uma lista de `reference_date` opcionalmente, loopando internamente antes de
subir pro staging uma vez só) — ou se "disparo manual N vezes" é suficiente
pra todos os casos de uso reais de backfill que existem hoje nos ~82
datasets. Não investigado a fundo ainda.
