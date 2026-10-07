# Pendências da revisão da Laura — PR #1932

Checklist original: [comentário da Laura](https://github.com/basedosdados/pipelines/pull/1932#issuecomment-5735990478),
no PR [basedosdados/pipelines#1932](https://github.com/basedosdados/pipelines/pull/1932)
(issue [#1867](https://github.com/basedosdados/pipelines/issues/1867)).

Este documento é o nosso rastreamento de progresso — **o comentário da Laura
no GitHub ainda não foi atualizado/riscado**, isso é feito à parte, depois de
revisar com ela. Trabalho feito **por partes**, por decisão do usuário.

Branch: `feat/event-pipeline-automations-poc`. **Commitado em 2026-09-29**
(commit `9aba70a55`, categorias 1-6 completas) — ainda **não pushado**.

## Merge com `origin/main` (2026-09-29) — 47 commits de atraso, resolvido

A branch estava 47 commits atrás do `main`. Ao mergear, apareceu um achado
importante: **7 dos 8 datasets migrados nesta branch** (`br_ans_beneficiario`,
`br_ibge_ipca`, `br_inmet_bdmep`, `br_me_caged`, `br_me_comex_stat`,
`br_ms_cnes`, `us_cfpb_hmda`) têm **conflito de conteúdo real** com o
`main`, não só textual — o `main` ainda tem a versão monolítica antiga
desses mesmos datasets, com `Cron`/`job_variables` de produção.
Investigação (via fork) confirmou que isso **não é trabalho paralelo de
outra pessoa colidindo** — os 7 flows monolíticos já existiam no
merge-base, antes desta branch nascer. O `main` só recebeu dois refactors
de infra que tocaram `flows.py` do repo inteiro mecanicamente
(`Flow`/`pipelines.utils.flow` customizado, `Cron` em vez de dict de
schedule) — não uma remigração. Substituir esses 7 monolíticos sempre foi
o objetivo da #1867/PR #1932; eles só seguiam no `main` porque a PR nunca
foi mergeada.

**Decisão do usuário**: manter a arquitetura nova nos 7, descartando o
monolítico do `main` — sem copiar os schedules de produção **de início**
(bate com o plano já documentado: validar cada dataset contra a fonte real
primeiro, depois agendar). Atualização (2026-09-30, depois da validação
completa em dev + prod real): schedules restaurados — ver seção "Schedules
restaurados" mais abaixo. `br_me_caged` fica de fora desta PR (revertido,
seção própria), sem schedule aqui.

**Mudanças adicionais exigidas pelo merge** (framework, não específicas
da #1867):
- `deploy_tags`/`job_variables`/`deploy_schedules` agora só funcionam via
  `Flow` customizado (`pipelines/utils/flow.py`) — `deploy_tags` não
  existia nesse arquivo (só `deploy_schedules`/`job_variables`, adicionados
  por outra pessoa), adicionado seguindo o mesmo padrão.
- Todo `flows.py` que essa branch toca precisou trocar
  `from prefect import flow` por `from pipelines.utils.flow import flow`
  — sem isso, `isinstance(obj, Flow)` em `deploy_flows.py` não reconhece
  os flows migrados e **nada seria deployado** (achado antes de virar
  incidente real, verificado com teste de carregamento de verdade contra
  `br_ms_cnes`: 26 flows, batendo com as 13 tabelas × 2 etapas).
- `rename_flow_run_dataset_table` virou síncrona no `main` (corrigiu um
  bug real: a versão `async` antiga nunca executava quando chamada de um
  flow síncrono, issue #1940). O padrão `run_coro_as_sync(rename_flow_run_dataset_table(...))`
  usado nesta branch (4 lugares: `stage_dispatch.py` ×2,
  `metadata/flows.py`, `materialize_prod/flows.py`) quebraria em runtime
  com a nova versão — corrigido pra chamada direta, sem o wrapper.
- `us_cfpb_hmda`: schedule (que já tinha sido copiado do monolítico antigo
  de propósito, numa sessão anterior) precisou virar `Cron(...)` de
  verdade — o `main` removeu a conversão automática de dict pra `Cron` em
  `deploy_flows.py`.
- Limpeza: 67 comentários `# pyrefly: ignore [missing-attribute]`, agora
  desnecessários (o `Flow` customizado tipa os três atributos), removidos
  nos 10 arquivos tocados pelo merge — mesmo tipo de limpeza que o `main`
  já tinha feito nos outros 60 `flows.py` do repo.

Validado com `ruff check`, `py_compile`, `pyrefly` (0 diagnósticos, 5
supressões pré-existentes não relacionadas) e um teste real de
carregamento via `deploy_flows.load_flows_from_file()` contra
`br_ms_cnes` — 26 flows, tags/job_variables corretos.

Merge commit: `571238f1c`. Branch agora 49 commits à frente do próprio
remoto — nada pushado ainda.

## Teste real dos 7 datasets migrados (2026-09-29/30)

Testando contra fonte real (não faz parte do checklist da Laura — achado
do próprio teste). Tabela de status por dataset e detalhe de cada rodada:
ver commits `571238f1c`..`HEAD` e [[bug-dispatch-dev-prod]] (bug sério de
dispatch achado no processo, já corrigido — dispatch entre etapas não
sabia se estava em dev ou prod). `build_and_promote` ganhou também
`update_metadata: bool = False`, pra testar a atualização de coverage sem
precisar promover pra prod de verdade (commit `671013c5c`).

Ainda investigando esse mesmo bug, ficou claro que `ExtractAndLoad.targets`
era redundante com a tag `env:dev`/`env:prod` recém-introduzida — e pior,
perigoso (decidia promoção pra prod a partir do que o dataset escreveu, não
de onde o flow realmente rodava). Removido por completo, substituído por
`promote_to_prod` calculado dentro de `dispatch_build_and_promote`. Ver
[[bug-dispatch-dev-prod]], seção "Remoção do `targets`".

## Revisão do CodeRabbit (2026-09-30) — 3 achados reais corrigidos

Conferido item por item contra o código atual (vários dos scripts de
investigação do bot partiam de premissas já desatualizadas). Corrigidos:
`br_ms_cnes.extract_load_data` não setava `partition_folders` (promovia
o staging inteiro pra prod toda vez, não só o mês novo);
`br_ans_beneficiario` podia perder um mês se a ANS atualizasse 2 de uma
vez (`partition_folders` vinha só do `reference_date` do check, não dos
arquivos realmente baixados); `check_for_updates`/`get_date_api`
(`crawler/ibge_inflacao`) tinham anotação de retorno errada, suprimida 4
vezes em cascata pela cadeia de chamada — corrigida na raiz.

No processo, achado que a reversão do `br_me_caged` (seção acima) tinha
usado uma referência de `main` desatualizada (sem `git fetch` antes) —
corrigido, re-sincronizado com o `origin/main` real.

Demais achados (schedule do `br_ms_cnes`, cuidado de `bq_project` no
`update_metadata`, convenção de `@task`, etc.) documentados na descrição
do PR como decisão deliberada ou falso positivo, sem mudança de código —
detalhe na seção "Revisão automática (CodeRabbit)" do corpo do PR.

## `partition_folders` automático — ✅ implementado nesta PR (2026-09-30)

Decisão revertida: implementado ainda dentro da #1932, não adiado —
usuário pediu pra fazer na hora depois de ver a proposta.

Ideia levantada revisando os 2 bugs do CodeRabbit acima: **derivar
`partition_folders` de `data_path` automaticamente**, em vez de cada
dataset computar na mão (fonte dos 2 bugs — cálculo manual errado). A
invariante já documentada em `ExtractAndLoad` (`stage_dispatch.py:67-79`)
já exige que `data_path` seja organizado exatamente na estrutura Hive
que `partition_folders` nomeia — então `partition_folders` é, por
definição, redundante: sempre dá pra derivar de `data_path` em vez de
declarar.

`br_me_comex_stat` já tem exatamente esse helper, só que local e não
reaproveitado: `_discover_partition_folders(base_path)`
(`pipelines/datasets/br_me_comex_stat/tasks.py:54`) — varre `data_path`
depois do download e acha as pastas-folha Hive (`ano=.../mes=...[/sigla_uf=...]`)
realmente escritas em disco, genérico pra 2 ou 3 níveis de partição, sem
precisar saber de antemão quais partições vieram no arquivo.

**Implementação**: `discover_partition_folders(data_path)` (nova função
pública em `stage_dispatch.py`, generalizada a partir do helper local do
`br_me_comex_stat`) roda dentro da cápsula
(`CheckThenExtractLoadPipeline.run_extract_and_load()`), logo depois de
`extract_load_data()` devolver. `ExtractAndLoad.partition_folders` virou
`field(init=False)` — não é mais parâmetro do construtor, nenhum dataset
declara isso na mão (quem tentar, quebra na hora com `TypeError`, em vez
de silenciosamente aceitar um valor errado). Os 6 datasets que calculavam
isso manualmente (`br_ibge_ipca`, `br_ans_beneficiario`, `br_inmet_bdmep`,
`br_me_comex_stat` — perdeu o `_discover_partition_folders` local,
agora redundante —, `br_ms_cnes`, `test_dataset`) tiveram essa lógica
removida.

Validado com testes unitários sintéticos (0/1/2/3 níveis de partição,
caminho inexistente) e depois contra Prefect real em dev: `br_ibge_ipca.mes_brasil`
→ `['ano=2026/mes=08']`, `test_event_pipeline_partitioned` →
`['ano=2026/mes=9']`, `test_event_pipeline` (sem partição) → `None` —
todos batendo exatamente com o valor que o cálculo manual antigo
produzia. `dbt run/test OK` em todos.

### Achado real: `br_me_comex_stat` promove muito mais partição que antes

`br_me_comex_stat.municipio_exportacao` devolveu **232 pastas** (8
meses × 29 UFs), não só `ano=2026/mes=08` como o cálculo manual
produzia. Investigado: a fonte do Comex Stat publica **1 arquivo por
ano, cumulativo** (contém janeiro–agosto inteiro) — `clean_br_me_comex_stat`
particiona esse arquivo inteiro em disco a cada download, então a
descoberta automática (fiel ao disco) pega tudo, não só o mês do
`reference_date`.

**Comparado contra o flow monolítico original (pré-#1867)**: ele nunca
teve o conceito de `partition_folders`/`folders` — fazia
`upload_to_gcs(data_path=filepath, ..., bucket_name="basedosdados")`
direto, sempre com o `filepath` inteiro (todos os meses acumulados),
toda execução. Ou seja, **o comportamento "só o mês atual" nunca foi o
de produção** — foi uma restrição que a própria migração #1867 introduziu
(o `partition_folders=[mês atual]` escrito nesta leva), sem perceber que
isso passava a ignorar silenciosamente qualquer correção da fonte a
meses anteriores do mesmo arquivo anual. A descoberta automática não
deixa esse dataset mais pesado do que sempre foi em produção — **restaura**
o comportamento original, que a migração tinha estreitado (e quebrado)
sem querer. Seguro de qualquer forma: o filtro incremental do dbt
(`WHERE data > max(data) já em prod`) garante zero duplicação nos meses
já cobertos.

Confirmado contra Prefect real: `dbt run/test OK` com as 232 partições
descobertas (só dev, `promote_to_prod=False` — o efeito de promover tudo
isso pra prod de verdade só se manifesta na próxima vez que este dataset
rodar com `promote_to_prod=True`).

## Schedules restaurados (2026-09-30)

Depois da validação completa (dev + prod real, seções acima), decisão
revisitada: restaurar os schedules de produção nos `check_update` dos 6
datasets desta PR, em vez de deixar pra depois do merge. Buscados do
`origin/main` real (`git fetch` antes — ver lição de
[[project_ftwca_branch_errada_pendente]]/[[feedback_git_fetch_antes_de_main]]),
direto dos flows monolíticos ainda vivos lá, e aplicados exatamente iguais
no `deploy_schedules` do `check_update` correspondente (nunca no
`extract_and_load` — mesmo lugar que o CodeRabbit já tinha sugerido pro
`br_ms_cnes`, agora generalizado pros outros 5).

| Dataset.tabela | Cron |
|---|---|
| `br_ans_beneficiario.informacao_consolidada` | `0 21 * * *` |
| `br_ibge_ipca.mes_brasil` | `40 14 8,9,10,11,12,13 * *` |
| `br_ibge_ipca.mes_categoria_brasil` | `30 14 8,9,10,11,12,13 * *` |
| `br_ibge_ipca.mes_categoria_rm` | `20 14 8,9,10,11,12,13 * *` |
| `br_ibge_ipca.mes_categoria_municipio` | `50 14 8,9,10,11,12,13 * *` |
| `br_inmet_bdmep.microdados` | `0 22 * * 1-5` |
| `br_me_comex_stat.municipio_exportacao` | `0 21 * * 1-5` |
| `br_me_comex_stat.municipio_importacao` | `0 20 * * 1-5` |
| `br_me_comex_stat.ncm_exportacao` | `0 8,17 * * 1-5` |
| `br_me_comex_stat.ncm_importacao` | `0 8,17 * * 1-5` |
| `br_ms_cnes.profissional` | `30 6 * * *` |
| `br_ms_cnes.estabelecimento` | `0 9 * * *` |
| `br_ms_cnes.equipe` | `30 9 * * *` |
| `br_ms_cnes.leito` | `0 10 * * *` |
| `br_ms_cnes.equipamento` | `30 10 * * *` |
| `br_ms_cnes.estabelecimento_ensino` | sem schedule (já era assim em `main`) |
| `br_ms_cnes.dados_complementares` | `0 11 * * *` |
| `br_ms_cnes.estabelecimento_filantropico` | `15 11 * * *` |
| `br_ms_cnes.gestao_metas` | `30 11 * * *` |
| `br_ms_cnes.habilitacao` | `45 11 * * *` |
| `br_ms_cnes.incentivos` | `50 11 * * *` |
| `br_ms_cnes.regra_contratual` | sem schedule (já era assim em `main`) |
| `br_ms_cnes.servico_especializado` | `30 12 * * *` |
| `us_cfpb_hmda.loan_application_register` | `25 16 8,9,10 3,4,5,6,7,8 *` (já estava, de uma sessão anterior) |

Validado com import de sanidade real de todos os 6 `flows.py` — cada
`check_update` mostrando exatamente o `Schedule(cron=...)` esperado (ou
`None` nos 2 casos que já eram sem schedule). `ruff`/`py_compile`/`pyrefly`
limpos.

Continua valendo o que já era verdade antes: o deploy registra sempre
`paused=True` (`deploy_flows.py`) — ter o schedule no código não ativa
nada sozinho, a ativação real continua manual/via sync do backend, como
já era pro resto do repo.

## 1. Renomeações — ✅ feito (localmente, não commitado)

- [x] `check_for_update` → `get_latest_update`
- [x] `CheckResult` → `SourceInspection`
- [x] `DownloadResult` → `ExtractAndLoad`
- [x] Flow "Download" → "Extract and Load" (`Etapa.EXTRACT_AND_LOAD`,
      `extract_and_load_flow_name`, prefixo em `rename_flow_run_dataset_table`)
- [x] `dispatch_mat_test` → `dispatch_build_and_promote`
- [x] `mat_test` (etapa/flow/deployment) → `build_and_promote` — flow
      **sem** sufixo `_flow` (`build_and_promote`, não `build_and_promote_flow`)
- [ ] `poll_source_for_update_task` → renomear pra refletir o que a função
      faz de fato — **decisão tomada, execução adiada pra depois da migração
      dos ~82 datasets** (ver categoria 7, nova issue)

Extra, não pedido pela Laura mas natural consequência da renomeação acima:

- `CheckThenDownloadPipeline.run_download()` → `run_extract_and_load()`
  (definição, docstrings de uso e todos os call sites nos 8 datasets já
  migrados pra essa arquitetura).
- Observação do usuário depois de ver o resto renomeado: `download_data`
  (parâmetro/atributo do callback) e `download_deployment` (atributo de
  override) ainda diziam "download", inconsistente com `extract_and_load`.
  Renomeados junto com a própria classe:
  - `CheckThenDownloadPipeline` → `CheckThenExtractLoadPipeline`
  - `download_data` → `extract_load_data` (parâmetro/atributo/callback)
  - `download_deployment` → `extract_load_deployment`
  - `pipeline_factory`: `download_data_factory` → `extract_load_data_factory`;
    nos 4 datasets que usam essa fábrica (`br_ibge_ipca`, `br_me_caged`,
    `br_me_comex_stat`, `br_ms_cnes`), `make_download_data` →
    `make_extract_load_data`
  - `us_cfpb_hmda/tasks.py`: a função bare `download_data` (sem prefixo de
    dataset) → `extract_load_data`, mesmo tratamento já dado ao
    `get_latest_update` bare desse dataset
  - Deliberadamente **não** tocado: nomes de função/flow específicos de
    cada dataset que descrevem a ação real de baixar dado
    (`br_ans_beneficiario_download`, `microdados_download`,
    `event_pipeline_download`, `br_ibge_ipca_mes_brasil_download`, etc.) —
    são escolha de nome por dataset, não fazem parte da API compartilhada
    do `stage_dispatch.py`.
  - Achado uma referência solta em `.github/scripts/deploy_flows.py`
    (comentário citando a classe antiga) — corrigida junto.

Todos os arquivos tocados (núcleo `pipelines/utils/stage_dispatch.py` +
`pipelines/utils/metadata/{flows,constants}.py` + os datasets
`br_ans_beneficiario`, `br_ibge_ipca`, `br_inmet_bdmep`, `br_me_caged`,
`br_me_comex_stat`, `br_ms_cnes`, `us_cfpb_hmda`, `test_dataset` +
`.github/scripts/deploy_flows.py`) validados com `ruff check`,
`python3 -m py_compile` e `pyrefly check` (0 diagnósticos) a cada rodada de
renomeação.

Achado no processo: `test_dataset/constants.py` tinha um dict
(`EVENT_PIPELINE_JOB_VARIABLES`) com chave string literal `"download"`,
lida via `Etapa.DOWNLOAD` — teria virado `KeyError` em runtime assim que o
valor do enum mudou pra `"extract_and_load"`. Corrigido junto.

## 2. Mudança de estrutura de dados — ✅ feito (localmente, não commitado)

- [x] Mover `compare_against` pra dentro de `SourceInspection` (ex-`CheckResult`)

Escopo confirmado com o usuário antes de implementar: só
`pipelines/utils/stage_dispatch.py` e os 8 datasets já migrados pra essa
arquitetura (nenhum deles passava `compare_against` explicitamente — todos
usavam o default). Os ~80 flows monolíticos legados que chamam
`poll_source_for_update_task(compare_against=...)` direto, fora do
`CheckThenExtractLoadPipeline`, ficam de fora — não fazem parte da #1867.

Os valores em si **não foram renomeados** — continuam `"coverage"`/
`"table_update"` (decisão do usuário: bate menos ao pé da letra com o texto
da Laura, "no formato `compare_against={latest_update, max_coverage}`", mas
evita uma mudança maior sem necessidade clara).

Mudança: `compare_against` deixou de ser parâmetro do construtor de
`CheckThenExtractLoadPipeline` (config fixa da pipeline inteira) e virou campo
de `SourceInspection`, com default `"coverage"` — quem decide o tipo de
comparação agora é o próprio `get_latest_update()` de cada tabela, não a
fábrica/construtor. `check_update_and_dispatch()` não mudou de assinatura.

## 3. Automatizar valores hoje manuais — ✅ feito (localmente, não commitado)

- [x] Remover `bq_project="basedosdados"` de `ExtractAndLoad` (deve ser
      resolvido automaticamente)

      Achado antes de implementar: `bq_project` e `prefect_mode` sempre
      andavam juntos em todo o repo (todo dataset real: `bq_project=
      "basedosdados"` com `prefect_mode` no default `"prod"`; `test_dataset`:
      `bq_project="basedosdados-dev"` com `prefect_mode="dev"`, sempre em
      par) — nunca um caso divergente. E já existe `MODE_PROJECT` (`pipelines/
      utils/metadata/constants.py`), o mesmo mapeamento `"dev"`→
      `"basedosdados-dev"`/`"prod"`→`"basedosdados"` já usado em
      `register_table_materialization_task` pra resolver o billing project.
      Reaproveitado: campo `bq_project` saiu de `ExtractAndLoad`;
      `dispatch_build_and_promote` agora resolve
      `metadata_constants.MODE_PROJECT.value[result.prefect_mode]` na hora de
      montar os parâmetros do `run_deployment`. Removida a linha
      `bq_project="..."` dos 9 call sites (7 datasets reais + 2 instâncias no
      `test_dataset`) — sobrou só `prefect_mode`, que agora faz o trabalho
      dos dois. ruff/py_compile/pyrefly (0 diagnósticos) limpos.
- [x] Automatizar quantos GB de RAM `get_latest_update`/download usam

      Decisão do usuário: abordagem 2 de [[pendencia-perfil-recursos-por-etapa]]
      — `deploy_flow()` (`.github/scripts/deploy_flows.py`) aplica
      `CHECK_UPDATE_JOB_VARIABLES = {"memory_limit": "1Gi", "memory_request":
      "1Gi"}` automaticamente pra qualquer flow com a tag `check_update`,
      **só quando o flow não seta `job_variables` explicitamente** — o
      mecanismo de override (`<flow>.job_variables = {...}`) já existia,
      só faltava o "senão" automático. Mudança de ~10 linhas.

      Ponto de atenção sinalizado ao usuário, não bloqueante: `br_ibge_ipca`
      é o único `check_update` migrado que **não** é um poll leve de
      verdade — `make_get_latest_update` chama `collect_data_utils` (baixa
      dado real da API do IBGE) dentro do próprio check. Ele vai herdar o
      novo default de 1Gi sem `job_variables` próprio hoje; se isso for
      pouco, precisa de um override explícito nesse dataset — não medido
      ainda, fica como próximo passo separado.

## 4. Documentação a acrescentar — ✅ feito (localmente, não commitado)

- [x] Documentar que `data_path` deve ser particionado, pra garantir
      transferência pro prod parcial
- [x] Deixar mais claro por que existem as partições

      Os dois num só lugar: `ExtractAndLoad.data_path`/`partition_folders`
      em `stage_dispatch.py`. Cadeia documentada: `data_path` (quando a
      tabela é particionada) precisa ser um diretório na estrutura Hive
      exata que `partition_folders` nomeia, porque `basedosdados.Storage.upload()`
      deriva o prefixo de partição no GCS a partir da estrutura de pastas
      em disco, não do campo `partition_folders` em si — se divergirem, o
      arquivo sobe pro lugar errado e `transfer_files_to_prod_flow` não
      acha nada pra promover. E a razão de existir partição: sem
      `partition_folders`, `download_files_from_bucket_folders` (chamada
      por `transfer_files_to_prod_flow`) baixa e reenvia
      `staging/{dataset_id}/{table_id}/` inteiro pra prod a cada
      `build_and_promote` — numa tabela grande/histórica isso reenviaria a
      série toda de novo, não só a fatia nova.
- [x] Documentar por que materialização + teste + publicação
      (`build_and_promote`) ficam no mesmo flow — reaproveitamento de recursos

      Em `build_and_promote` (`pipelines/utils/metadata/flows.py`): ao
      contrário de check_update → extract_and_load (que só dispara o
      próximo estágio *se* houver dado novo — ponto de decisão real),
      materializar/testar/promover/registrar sempre rodam juntos, em
      sequência estrita, sem nenhuma condição no meio — separar custaria
      um pod novo (com overhead de baixar e compilar o projeto dbt de
      novo) só pra rodar a próxima etapa sem ganhar capacidade de disparo
      independente nenhuma.

## 5. Correções pontuais — ✅ feito (localmente, não commitado)

- [x] `run_extract_and_load` (ex-`run_download`): renomear a variável local
      `result` pra `download_result` — só dentro desse método
      (`stage_dispatch.py`); `result` de `run_check_update` (guarda um
      `SourceInspection`, não um `ExtractAndLoad`) e o parâmetro `result`
      de `dispatch_build_and_promote` ficaram como estavam, fora do pedido
      literal da Laura.
- [x] `transfer_files_to_prod_flow`: remover o parâmetro `update_metadata`
      (deve sempre atualizar metadados); mover a chamada de
      `register_table_materialization_task` de fora pra dentro do flow

      Feito em `pipelines/utils/materialize_prod/flows.py` +
      `pipelines/utils/metadata/flows.py`. `update_metadata: bool = False`
      removido — metadado sempre atualiza ao final agora. Novo parâmetro
      `coverage: CoverageSpec | None = None` (quando quem chama já tem um
      `CoverageSpec` tipado, como `build_and_promote`, usa ele direto; senão
      cai no `_build_coverage(...)` a partir dos parâmetros primitivos, que
      ficam só pra disparo manual via UI). Novo parâmetro `prefect_mode`
      também — faltava antes, a chamada antiga nunca passava isso, então
      sempre resolvia billing como prod mesmo quando o dataset era dev.

      A chamada duplicada de `register_table_materialization_task`, que
      ficava fora, incondicional, em `build_and_promote`, foi removida —
      `build_and_promote` agora só passa `coverage`/`env`/`bq_project`/
      `prefect_mode` pra dentro de `transfer_files_to_prod_flow`.
      Consequência real (nenhum dataset atual é afetado, confirmado via
      grep que ninguém usa `targets=["dev"]` sozinho): se algum dataset só
      promover pra dev, a coverage/Update no backend não é mais tocada —
      mais correto que o comportamento antigo (registrava materialização
      mesmo sem o dado ter chegado em prod).

      Docstrings de `transfer_files_to_prod_flow` e `_build_coverage`
      adicionadas (não tinham nenhuma antes) — Google style enxuto direto,
      sem passar pela fase "diário" (já aplicando o padrão refinado da
      seção de docstrings abaixo).

## 6. Coverage — 🔶 parcial (localmente, não commitado)

- [x] Agora: mover `coverage` (`CoverageSpec`) pra dentro das constantes do
      dataset (`constants.py`), em vez de embutido ad hoc em `tasks.py`

      Achado antes de implementar: em nenhum dos 8 datasets a coverage
      varia por tabela — `br_ibge_ipca` (4 tabelas), `br_me_caged` (3),
      `br_me_comex_stat` (4) e `br_ms_cnes` (13) usam exatamente o mesmo
      `PartBdpro(date_column=YearMonth(year="ano", month="mes"),
      date_format=DateFormat.YEAR_MONTH)` pra todas as tabelas do dataset.
      Então um `COVERAGE` só por dataset (não por tabela) já cobre o caso
      real — nenhum tem tier ou date_column diferente entre tabelas.

      Cada `constants.py` ganhou um `COVERAGE = <CoverageSpec>(...)` (a
      instância Pydantic construída, não o dict já serializado);
      `tasks.py` passou a importar `COVERAGE` e chamar só
      `COVERAGE.model_dump()` no lugar da construção inline.

      **Por que a constante é a instância Pydantic, e `.model_dump()` só
      roda no ponto de uso — não o dict já pronto direto em
      `constants.py`** (dúvida levantada pelo usuário, resposta registrada
      2026-09-29):

      1. **Evita dict compartilhado por acidente.** `.model_dump()`
         devolve um `dict` novo a cada chamada. Se `constants.py` já
         guardasse o dict pronto (`COVERAGE = PartBdpro(...).model_dump()`),
         esse dict viraria um objeto único reaproveitado em toda tabela
         que referenciar `COVERAGE` — no `br_ibge_ipca`, as 4 tabelas
         receberiam literalmente o mesmo objeto em memória. Ninguém muta
         esse dict hoje, mas é o tipo de coisa que quebra silenciosamente
         no futuro: um `coverage["algo"] = x` feito antes de repassar
         adiante pareceria inofensivo (parece uma cópia local), mas
         mutaria a constante compartilhada por todas as tabelas do
         dataset, sem erro nenhum acusando. Gerar o dict na hora do uso
         elimina esse risco de aliasing por completo.
      2. **Mantém `COVERAGE` tipado enquanto possível.** Com a instância
         Pydantic na constante, dá pra inspecionar campos com
         autocomplete/validação (`COVERAGE.tier`, `COVERAGE.date_column`)
         se algum código um dia precisar disso antes de virar dict. Uma
         vez dumpado, essa garantia de tipo se perde — vira só um dict
         solto, sem checagem nenhuma até alguém tentar usar errado. Padrão
         comum em código Pydantic: manter o objeto tipado "em repouso" e
         só serializar na fronteira onde o `dict` é de fato exigido — aqui,
         o campo `ExtractAndLoad.coverage: dict`.

      Contraponto considerado: um objeto Pydantic não é, a rigor, um
      "literal" no sentido estrito da regra "constants.py só literais"
      (string/dict/enum) — um dict pré-dumpado bateria mais literalmente
      com essa convenção. Decisão: o ganho de segurança contra aliasing
      pesa mais que a pureza estilística aqui.

      **Correção depois de revisar**: em `us_cfpb_hmda`, que usa
      `class constants(Enum)` pros literais antigos (padrão pré-#1867),
      `COVERAGE` tinha sido colocado dentro do Enum por engano — obrigava
      `constants.COVERAGE.value.model_dump()`, um `.value` a mais que
      destoava dos outros 7 datasets. `COVERAGE` é código novo desta leva
      de migração, não literal antigo do dataset — não precisa herdar a
      convenção do Enum só por estar no mesmo arquivo. Movido pra fora,
      como constante de módulo solta — agora `COVERAGE.model_dump()` em
      `us_cfpb_hmda` também, igual todo mundo.

      Confirmado que `constants.py` importar de `pipelines.utils.metadata.domain`
      não viola a regra "constants.py só literais, nunca importa de
      tasks.py" (documentada em
      [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]]) — a regra é
      especificamente contra importar do `tasks.py` do próprio dataset
      (risco de ciclo); importar de um módulo de utils compartilhado é o
      mesmo padrão que `tasks.py` já usava.

      Validado com `ruff check --fix`, `py_compile` e `pyrefly` (0
      diagnósticos, 69 supressões pré-existentes) nos 24 arquivos tocados
      (`constants.py`/`tasks.py`/`flows.py` × 8 datasets), e com um `import`
      de sanidade real de cada `flows.py` — sem erro nem warning novo.
- [ ] Futuro (nova issue): mover o coverage pro backend em vez de manter
      como constante nas pipelines

## 7. Issues novas a abrir — não iniciado (não fazer nesta PR)

- [ ] `source_format="parquet"` automático (hoje default `"csv"` em
      `ExtractAndLoad`)
- [ ] Coverage vindo do backend (mesmo item do 6)
- [ ] Separar `poll_source_for_update_task` em duas tasks — decisão da
      equipe, resposta trazida pelo usuário em 2026-09-28: entre
      `register_poll_and_check_update` (renomear só, mantendo uma task só)
      e separar em `register_poll`/`detect_update` (duas tasks), a equipe
      prefere a separação ("não vejo necessidade dessas 2 coisas ficarem
      juntas"), mas reconhece que dá mais trabalho.

      **Achado que muda o escopo**: `poll_source_for_update_task` é
      chamada em **94 arquivos** — praticamente todo flow monolítico do
      repositório (crawlers + ~80 datasets ainda não migrados), não é
      código exclusivo da #1867. Separar em duas tasks exige reescrever
      cada um desses 94 call sites (uma chamada vira duas), não só
      renomear — ordens de grandeza maior que qualquer outra renomeação
      feita nesta revisão. E não desbloqueia nenhuma capacidade nova: hoje
      poll e check sempre são usados juntos, em toda chamada, sem exceção
      — é separação por organização, não por necessidade funcional
      observada.

      **Decisão**: registrar como issue nova, a ser feita **depois que a
      migração dos ~82 datasets pra arquitetura orientada a eventos
      terminar** — nesse ponto, os flows monolíticos que hoje chamam
      `poll_source_for_update_task` direto já não existirão mais (viram
      `get_latest_update` de cada dataset via `CheckThenExtractLoadPipeline`),
      reduzindo drasticamente o raio de impacto da separação. Fazer agora,
      no meio da migração, duplicaria o trabalho (call sites que ainda vão
      ser migrados/removidos de qualquer forma).

## Fora do checklist da Laura: docstrings em Google style — 🔶 em andamento

Pedido separado do usuário: as docstrings do código novo da migração
estavam em prosa corrida ("estilo diário", explicando raciocínio/histórico
da decisão) em vez do padrão Google style (resumo curto + `Args:`/
`Returns:`/`Attributes:`) usado no resto do repo — regra geral do usuário
é que funções novas/alteradas ganham `Args:`/`Returns:`, não só prosa
solta.

Escopo confirmado com o usuário: começar só por
`pipelines/utils/stage_dispatch.py` (arquivo núcleo, docstrings mais
longas e mais reaproveitadas).

**Padrão refinado em 2026-09-29** (2ª rodada, depois de revisar
`build_and_promote`): não basta ter `Args:`/`Returns:` — nada de prosa
explicando o *porquê* de decisões de arquitetura, referência a número de
issue, ou "ver arquivo X" apontando pra contexto de design. Isso tudo sai
da docstring por completo e vai pra um `.md` dedicado no ftwca — a
docstring vira só fato: o que a função/classe faz, `Args:`/`Returns:`/
`Attributes:` descrevendo cada parâmetro. Comentário inline (não
docstring) continua permitido pra workaround de bug não-óbvio (ex. o
`run_coro_as_sync`/issue #1940), critério já usado no resto do repo.

- [x] `pipelines/utils/stage_dispatch.py` — reescrito por completo, 2
      rodadas (Google style básico, depois o corte de raciocínio/issues/
      pointers). Racional completo preservado em
      [[stage-dispatch-racional-de-design]]. Verificado que **nenhuma
      linha de código executável mudou** nas duas rodadas — comparação de
      AST ignorando docstrings, antes/depois idênticos. ruff/py_compile/
      pyrefly (0 diagnósticos) limpos.
- [x] `pipelines/utils/metadata/flows.py` (`update_temporal_coverage` +
      `build_and_promote`) — mesmo tratamento, 2 rodadas. Racional
      completo preservado em [[build-and-promote-racional-de-design]]. A
      explicação do bug do `run_coro_as_sync`/`rename_flow_run_dataset_table`
      ficou como comentário inline (workaround de bug, não interface
      pública). ruff/py_compile/pyrefly limpos.
- [x] Os 8 datasets migrados (`tasks.py`/`flows.py`/`constants.py` de cada
      um) — feito em 2026-09-30, a pedido do usuário depois de ele notar
      prosa "diário" sobrevivendo no `flows.py` novo do `br_bcb_taxa_cambio`
      (parâmetro `anos`/pendência apontando pro ftwca). Removido: toda tag
      `(issue #1867)` (referência externa solta), e os 2 casos reais de
      "ver X.md no ftwca" que sobreviveram na migração original
      (`br_ibge_ipca`/`br_inmet_bdmep`, ambos sobre o critério de 5 GB —
      conteúdo já preservado em [[migracao-br-ibge-ipca]]/
      [[migracao-lote-10-datasets]], nada perdido). Mantido: comentário
      técnico/factual sem "porquê" de decisão (ex. "diferente de
      br_ibge_ipca, aqui o check é leve de verdade") e invariantes reais
      (ex. `memory_request` do `check_update` de `test_dataset` precisa
      ser < default do pool, senão o Kubernetes rejeita o pod). De quebra,
      achado que `br_inmet_bdmep` (flagado como "tamanho não medido, testar
      antes de produção") já tinha sido validado contra a fonte real nesta
      mesma sessão — tabela do [[migracao-lote-10-datasets]] atualizada.
      `ruff`/`py_compile`/`pyrefly` limpos, import de sanidade real (59
      flows descobertos nos 8 arquivos, nenhuma quebra).

## Estado ao pausar (2026-09-24)

Sessão pausada a pedido do usuário — retomar amanhã. Nada foi commitado
ainda em `pipelines` (branch `feat/event-pipeline-automations-poc`, só
mudanças no working tree). Ordem sugerida pra retomar:

1. Decidir se commita o que já está pronto (itens 1, 2 e a renomeação
   `CheckThenExtractLoadPipeline` desta seção, mais a docstring de
   `stage_dispatch.py`) antes de seguir, ou junta tudo num commit só.
2. Continuar a conversão de docstrings (`metadata/flows.py`, depois os 8
   datasets), se o usuário confirmar que quer seguir por aí.
3. Categorias 3-7 do checklist da Laura, ainda não iniciadas (ver acima).

---

Ver também: [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]] (guia de
nomenclatura — vai precisar de atualização própria depois que os nomes
acima estiverem commitados/pushados) e
[[relatorio-status-2026-09-09.md|relatório de status]] (ainda usa os nomes
antigos — desatualizado em relação a este documento).
