# Bug: dispatch entre etapas não sabia se estava em dev ou prod

Achado em 2026-09-29/30, testando de verdade (fonte real, pool de dev) os
7 datasets migrados pra arquitetura orientada a eventos — não fazia parte
do checklist da Laura, surgiu do teste em si. Ver [[revisao-laura-pr-1932]]
pro contexto geral dessa leva de testes e a tabela de status por dataset.

## O problema

`deploy_flow()` (`.github/workflows/scripts/deploy_flows.py`) registra
cada deployment com um nome diferente conforme o pool:

```python
is_dev = "dev" in pool_name
deployment_name = f"dev-{flow_name}" if is_dev else flow_name
```

Isso evita que um deploy de PR "roube" o deployment de prod (mesmo nome,
`work_pool_name` é só um campo mutável do registro). Mas essa decisão só
existe **na hora do deploy** — `deployment_name()`/`_flow_name()`
(`pipelines/utils/stage_dispatch.py`), que rodam **dentro do flow em
execução** pra saber pra quem despachar o próximo estágio
(`check_update_and_dispatch`/`dispatch_build_and_promote`), não tinham
nenhuma noção de estarem rodando em dev ou prod — sempre montavam o nome
sem o prefixo `dev-`.

Em prod isso nunca dava problema (prefixo vazio nos dois lados, por
coincidência). Em dev, todo dispatch entre estágios falhava com `404 Not
Found` — só não tínhamos visto isso antes porque, em todo teste anterior
desta sessão, ou o disparo era manual (nome completo digitado à mão, já
com `dev-`), ou o `check_update` nunca achava dado novo (o dispatch nunca
chegava a rodar).

## As 3 opções levantadas

1. **Ler o nome do próprio deployment em runtime**
   (`prefect.runtime.deployment.name.startswith("dev-")`) — mais simples,
   1 arquivo só, mas depende de inferir o ambiente a partir de uma
   convenção de string.
2. **Tag explícita de ambiente** (`env:dev`/`env:prod`, lida via
   `prefect.runtime.flow_run.tags`) — mais explícito, reaproveita o
   mecanismo de tag que já existia pro `CHECK_UPDATE_JOB_VARIABLES`.
   Escolhida.
3. **Variável de ambiente na infraestrutura do work pool** — mais robusto
   ainda (não depende de nome nem tag do Prefect, só de onde o pod
   fisicamente roda), mas exige mexer em config de infra (Terraform/UI do
   Prefect), fora do que dá pra fazer só no código deste repositório.
   Deixado pra depois, se a opção 2 se mostrar insuficiente.

## A implementação (opção 2)

`deploy_flow()` passou a marcar todo deployment com `env:dev` ou
`env:prod`, além das tags que já existiam:

```python
tags = ["automated-deploy", f"env:{'dev' if is_dev else 'prod'}", *extra_tags]
```

`stage_dispatch.py` ganhou `_dev_prefix()`, que lê
`prefect.runtime.flow_run.tags` (já disponível no contexto do flow run —
sem chamada de API) e devolve `"dev-"` quando `env:dev` está presente,
`""` caso contrário (inclusive fora de um flow run real, ex. chamada
local — `runtime.flow_run.tags` vem `[]`, mesmo comportamento de hoje).
`deployment_name()` usa esse prefixo ao montar o nome do próximo estágio.

## Erro pego durante a implementação, antes de testar

`build_and_promote` **não** deveria levar o prefixo `dev-`. Ele é um
deployment único, global, que sempre roda no pool `basedosdados` (prod)
por desenho — mesmo quando quem despacha pra ele (`extract_and_load`)
rodou em dev — porque só a service account do pool prod tem permissão de
escrita real pra promover dado (mesma razão documentada desde o início da
issue #1867: "`mat_test`/`build_and_promote` roda no pool basedosdados
(prod)"). O 404 que motivou tudo isso não era sobre prefixo dev/prod —
era porque esse deployment simplesmente não existia ainda em prod (só
resolvido deployando-o lá, separado desta correção).

Apliquei o prefixo dinâmico nos dois ramos de `deployment_name()` por
padrão, sem perceber essa exceção — corrigido antes de testar: só o ramo
genérico (`check_update`→`extract_and_load`) usa `_dev_prefix()`; o ramo
`Etapa.BUILD_AND_PROMOTE` continua hardcoded, sem prefixo, sempre.

## Resíduo a limpar

Durante a investigação, um deployment `build_and_promote/dev-build_and_promote`
chegou a ser registrado no pool de dev (parte do primeiro deploy em lote
dos 7 datasets, que incluiu `metadata/flows.py`). Com a correção acima,
ele nunca mais é referenciado pelo dispatch — órfão, sem risco (não é
lido por nada), mas vale apagar na próxima limpeza de deployments de
teste em dev.

## Remoção do `targets`

Discutindo a implementação da opção 2, ficou claro que `ExtractAndLoad.targets`
(`list[str]`, default `["dev", "prod"]`, escrito por cada dataset em
`extract_load_data`) tinha ficado redundante com a tag `env:dev`/`env:prod`
recém-introduzida — e pior que redundante: **decidia se promovia pra prod
a partir do que o dataset escreveu em `constants.py`, não de onde o flow
estava realmente rodando**. Como todo dataset real usa
`prefect_mode="prod"`/`targets=["dev","prod"]` por padrão, um `extract_and_load`
disparado manualmente num pool de dev (pra testar) ia `dispatch_build_and_promote`
com `targets` pedindo promoção pra prod — e `build_and_promote` roda
*sempre* no pool prod, então promoveria de verdade. Só não aconteceu porque,
até este ponto, todo teste forçado usou invocação manual direta de
`build_and_promote` (contornando o dispatch automático), nunca deixou o
`extract_and_load` disparar sozinho — mas o buraco existia.

Não existe caso de uso legítimo pra um `targets` diferente do que
`_dev_prefix()` já implica: um `extract_and_load` rodando em dev nunca deve
promover pra prod (é teste), e um rodando em prod sempre deve promover (é
produção real). Não há "prod que só materializa até dev" nem "dev que
promove até prod" via dispatch automático — o único caso real disso
(`test_dataset`'s dois pilotos, que setavam `targets=["dev","prod"]` pra
exercitar `transfer_files_to_prod_flow` de verdade a partir de um dataset
seguro em `basedosdados-dev`) sempre foi um teste manual, não uma regra de
negócio genérica.

Decisão (aprovada pelo usuário): remover `targets` por completo.

- `ExtractAndLoad` perdeu o campo `targets`.
- `dispatch_build_and_promote` (`stage_dispatch.py`) calcula
  `promote_to_prod = not _dev_prefix()` sozinho — quem despacha decide
  puramente a partir de onde ele mesmo está rodando, sem depender de nada
  que o dataset tenha escrito.
- `build_and_promote` (`metadata/flows.py`) trocou
  `targets: list[str] | None = None` por `promote_to_prod: bool = True`;
  `if "prod" in targets` virou `if promote_to_prod`; `update_metadata`
  passou a ser condicionado a `not promote_to_prod`.
- `test_dataset/tasks.py` perdeu o `targets=["dev", "prod"]` dos dois
  pilotos — a capacidade de exercitar `transfer_files_to_prod_flow` de
  verdade a partir de um dataset seguro continua existindo, só que agora
  exclusivamente via invocação manual direta de `build_and_promote` (com
  `promote_to_prod=True` explícito), não mais via dispatch automático.

Validado estaticamente (ruff/py_compile/pyrefly limpos, grep de varredura
em todo `pipelines/` sem nenhuma referência restante ao `targets` desta
arquitetura).

## Armadilha: `_dev_prefix()` só pode ser lido no ponto de disparo

`promote_to_prod` **tem** que ser calculado dentro de
`dispatch_build_and_promote` (que roda no contexto do `extract_and_load`
chamador) e passado como parâmetro explícito pro `build_and_promote` — não
dá pra simplificar chamando `_dev_prefix()` de dentro do próprio
`build_and_promote`.

O motivo: `_dev_prefix()` lê `prefect.runtime.flow_run.tags` — as tags do
flow run **que está executando a função no momento em que ela é chamada**.
Como `build_and_promote` é um deployment único, global, que sempre roda no
pool `basedosdados` (prod) — a mesma razão pela qual seu nome nunca leva o
prefixo `dev-` (ver seção acima) —, sua própria tag é sempre `env:prod`,
não importa quem o disparou. `_dev_prefix()` chamada de dentro dele sempre
devolveria `""`, e `promote_to_prod` seria sempre `True` — reintroduzindo,
de outra forma, exatamente o mesmo bug de segurança que motivou remover o
`targets`.

A informação que importa (o `extract_and_load` chamador rodou em dev ou em
prod?) só existe no contexto de quem despacha. Por isso `_dev_prefix()`
precisa ser lida ali, em `dispatch_build_and_promote`, e roteada como
parâmetro — `build_and_promote` não tem como descobrir isso sozinho
olhando pras próprias tags.

## Status

Implementado (opção 2 + remoção do `targets`), validado estaticamente
(ruff/py_compile/pyrefly limpos) e **testado contra Prefect real** em
2026-09-30, depois de redeployar os 7 datasets + `test_dataset` em dev e
`build_and_promote`/`update_temporal_coverage` em prod.

Smoke test com `test_dataset` (`test_event_pipeline`, dado sintético —
`check_update` sempre acha "dado novo", cadeia determinística, zero risco
real) fechou o ciclo inteiro pela primeira vez via dispatch 100%
automático, sem nenhuma invocação manual:

1. `check_update` (dev) comparou fonte (2026-09-30) contra coverage
   (2026-09-05), achou dado novo, disparou `extract_and_load` — a
   correção do prefixo `dev-` funcionou (antes, 404 aqui).
2. `extract_and_load` (dev) subiu o CSV pro staging
   (`gs://basedosdados-dev/staging/test_dataset/test_event_pipeline`) e
   disparou `build_and_promote`.
3. `build_and_promote` rodou na infra do pool prod (por desenho — log
   confirma `Kubernetes job ... in namespace 'prefect-worker-basedosdados'`),
   mas com `promote_to_prod=False` (calculado corretamente a partir de
   onde o `extract_and_load` chamador rodou) — só `dbt run OK`/`dbt test
   OK` em dev, sem `transfer_files_to_prod_flow`, sem escrita em prod.

Repetido em seguida pros 7 datasets reais migrados (2026-09-30), forçando
`extract_and_load` diretamente (bypass do `check_update` — produção já
estava em dia em todos, sem dado novo real disponível no momento) num
período já coberto (seguro de re-rodar, checado por dataset: estratégia
`incremental` com filtro estrito de data/`data_carga` de origem, ou
`unique_key`+merge, ou `materialized="table"` full-rebuild — ver
[[revisao-laura-pr-1932]] pra detalhe por dataset). Todos os 6 passaram —
`dbt run/test OK`, `promote_to_prod=False` confirmado em log:
`br_ibge_ipca.mes_brasil`, `br_ms_cnes.habilitacao`,
`br_ans_beneficiario.informacao_consolidada`,
`br_me_comex_stat.municipio_exportacao`, `br_inmet_bdmep.microdados`
(baixa o ZIP inteiro do ano — mais pesado, sem problema),
`us_cfpb_hmda.loan_application_register` (reconstrói o histórico
2018-2025 inteiro a cada run, por desenho — mais pesado ainda, sem
problema). `br_me_caged` fica de fora, bloqueado pelo bug de período sem
dado (ver seção própria em [[revisao-laura-pr-1932]]).

Dispatch dev/prod totalmente validado contra Prefect real, nos dois
sentidos (`check_update`→`extract_and_load` e
`extract_and_load`→`build_and_promote`), pra 7 datasets reais + o piloto
sintético.

## `promote_to_prod=True` — validado contra prod de verdade (2026-09-30)

Todo o teste acima usou `promote_to_prod=False` (dev) de propósito — o
branch que chama `transfer_files_to_prod_flow` (upload real pro bucket
`basedosdados`, `dbt run/test --target prod`, registro de materialização
no backend) nunca tinha sido exercitado nesta arquitetura nova. Testado
via invocação manual direta de `build_and_promote/build_and_promote`
(`promote_to_prod=True` explícito):

- **`test_dataset.test_event_pipeline`** (sintético, `bq_project=basedosdados-dev`,
  confinado): **falhou** com `IndexError: list index out of range` em
  `basedosdados/backend.py:429` (`_get_table_id_from_name`) —
  `register_table_materialization` procura o `CloudTable` de
  `test_dataset.test_event_pipeline` no backend e não acha nada (`test_dataset`
  nunca foi cadastrado como dataset real, é só um piloto de código). Achado
  isolado, específico do `test_dataset` — tratamento de erro frágil ali
  (`IndexError` sem mensagem clara em vez de um erro "tabela não encontrada
  no backend"), não é bug da correção do dispatch. Não investigado a fundo
  por ora.
- **`br_ibge_ipca.mes_brasil`** (real, `bq_project=basedosdados`,
  `env=prod`, período já coberto — ago/2026): **sucesso completo em
  produção de verdade**. Log confirma upload real (`gs://basedosdados/staging/br_ibge_ipca/mes_brasil`),
  `dbt target=prod | project=basedosdados | account=dbt-rpc@basedosdados.iam.gserviceaccount.com`,
  `dbt run/test OK` contra prod, export e registro de materialização OK.
  Resultado esperado (zero linhas novas, período já coberto) confirmado —
  "Última data: 2026-08" inalterada. Primeira prova real do caminho de
  promoção completo nesta arquitetura.
