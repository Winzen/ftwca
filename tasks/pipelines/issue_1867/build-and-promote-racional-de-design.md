# `build_and_promote` — racional de design

Conteúdo que morava na docstring de `build_and_promote` (`pipelines/utils/metadata/flows.py`)
antes da conversão pra Google style (2026-09-29) — resumo curto + `Args:`,
sem prosa explicando o porquê das decisões de arquitetura. Esse "porquê"
foi movido pra cá, não descartado.

Nomes atualizados pra terminologia atual (`build_and_promote`,
`extract_and_load`, `CheckThenExtractLoadPipeline`) — a função nasceu como
`mat_test_flow`, ver [[revisao-laura-pr-1932]] categoria 1 pra o histórico
completo de renomeações.

## Por que materializar + testar + promover + registrar ficam num `@flow` só

Não são um estágio separado tipo `check_update`/`extract_and_load` porque
essas quatro coisas (`run_dbt` dev, `transfer_files_to_prod_flow`,
`register_table_materialization_task`) sempre rodam juntas, em sequência
estrita, sem nenhum ponto de decisão real entre elas — ao contrário de
`check_update` → `extract_and_load`, que só dispara o próximo estágio *se*
houver dado novo. Separar custaria um pod novo (com o overhead de baixar e
compilar o projeto dbt de novo) só pra imediatamente rodar a próxima etapa
sem ganhar nenhuma capacidade de disparo independente.

## Por que `run_dbt(target="dev")` sempre roda primeiro

Se falhar, a exceção propaga e aborta o flow antes de qualquer coisa tocar
prod — mesma garantia que o padrão de flow monolítico antigo tinha via
`if not materialize_after_dump`. Só promove pra prod
(`transfer_files_to_prod_flow`: baixa do staging de dev, sobe no staging
de prod, roda `run_dbt(target="prod")`, registra a materialização) quando
`"prod"` está em `targets` (default `["dev", "prod"]`).

## `register_table_materialization_task` só roda se promoveu pra prod

Até 2026-09-29 essa chamada ficava duplicada: uma vez (opt-in, desligada
por padrão) dentro de `transfer_files_to_prod_flow`, e de novo, sempre,
fora dele, direto em `build_and_promote`. Consolidado: agora só existe
dentro de `transfer_files_to_prod_flow`, que passou a atualizar metadado
sempre (não é mais opt-in). Efeito: se um dataset passar `targets=["dev"]`
(nenhum passa isso hoje), a coverage/Update no backend não é tocada,
porque o dado nunca chegou de fato na tabela pública — antes, era tocada
mesmo assim, o que não fazia sentido.

## `dataset_id`/`table_id` como parâmetros soltos

Não ficam dentro de um dict maior de propósito — aparecem visíveis direto
na lista de runs do Prefect, e dão pra nomear o flow run com eles
(`rename_flow_run_dataset_table`).

## O bug do `rename_flow_run_dataset_table` (issue #1940)

`rename_flow_run_dataset_table` é uma `@task` async. Chamada sem `await`
de dentro de um flow síncrono, ela só cria uma coroutine e descarta — o
rename nunca acontece, sem erro nem log. Esse bug afeta vários outros
flows do repositório que chamam essa mesma task da forma errada, não só
este. A correção aqui é `run_coro_as_sync(...)` em volta da chamada, que
efetivamente roda a coroutine e espera o resultado — deixado como
comentário inline no código, não na docstring, por ser detalhe de
implementação (workaround de bug), não parte da interface pública do flow.

## `partition_folders`

Cobertura mais completa já está na docstring de `ExtractAndLoad`
(`pipelines/utils/stage_dispatch.py`) — aqui só repassado direto pra
`transfer_files_to_prod_flow`, pra promover só a fatia nova, não o
staging inteiro.
