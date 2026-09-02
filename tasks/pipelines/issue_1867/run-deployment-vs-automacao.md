---
repo: basedosdados/pipelines
relacionado: issue #1867 (ver issue-1867-pipeline-eventos.md), automacoes-em-massa.md
status: DECIDIDO e IMPLEMENTADO (2026-09-01) — run_deployment() substituiu Automation/emit_event. Testado de ponta a ponta com sucesso na primeira tentativa (flow_download -> mat_test via run_deployment, subflow lineage real, dbt dev+prod, coverage atualizada). As duas automações antigas do piloto foram deletadas do workspace real do Prefect
atualizado: 2026-09-01
---

# `run_deployment()` vs. Automação — mecanismo de disparo entre os mesmos 3 deployments

Retomada da dúvida registrada em
[automacoes-em-massa.md](./automacoes-em-massa.md#dúvida-levantada-pelo-usuário-não-decidida-talvez-run_deployment-seja-mais-simples):
ao ver que a automação **também** precisaria de um mecanismo de descoberta
por tag pra escalar pros 82 datasets, será que vale a pena usar
`run_deployment()` chamado direto do código em vez de manter Automações
como objetos separados no Prefect?

**Importante — isto não é a mesma discussão de
[automacao-vs-subflow.md](./automacao-vs-subflow.md).** Aquele documento
comparava **quantos deployments/pods** existem (propunha fundir
`check_update`+`flow_download` num só, decidido que não vale a pena por
isolamento de memória). Este aqui mantém os **3 deployments separados
exatamente como estão hoje** — a única coisa em discussão é **como um
deployment dispara o próximo**: evento+Automação, ou uma chamada de
função (`run_deployment()`) dentro do próprio código do flow upstream.

## O que `run_deployment()` realmente é

Investigado no código-fonte do Prefect 3.5.0 instalado
(`prefect.deployments.run_deployment`):

```python
@sync_compatible
@inject_client
async def run_deployment(
    name: Union[str, UUID],
    parameters: Optional[dict[str, Any]] = None,
    timeout: Optional[float] = None,
    as_subflow: Optional[bool] = True,
    idempotency_key: Optional[str] = None,
    ...
) -> "FlowRun":
```

Pontos que mudam a análise em relação ao que se assumia antes:

- **`@sync_compatible`** — diferente do bug achado em
  `rename_flow_run_dataset_table` (issue #1940: `@task` async chamada sem
  `await` de um flow síncrono nunca executa), `run_deployment()` é seguro
  de chamar direto de um `@flow` síncrono, sem `run_coro_as_sync` nem
  `await` — o próprio decorator detecta o contexto e resolve a coroutine
  internamente.
- **`timeout=0` = não bloqueia.** Só cria o flow run e retorna
  imediatamente, sem esperar ele terminar. Ou seja: chamar
  `run_deployment(..., timeout=0)` de dentro do `check_update_flow`
  **não** faz o pod do `check_update` ficar de pé esperando o
  `flow_download` terminar — o comportamento de isolamento de recurso
  entre os 3 deployments (o motivo que ganhou a discussão anterior) **não
  muda**. `run_deployment()` ainda cria um flow run novo, que um worker
  pega e agenda num pod novo, exatamente como a ação `RunDeployment` de
  uma automação faz hoje — mesmo overhead de agendamento de pod
  (~70-90s), porque é o mesmo mecanismo por baixo (criar um flow run via
  API de orquestração). **Isto não reabre a discussão de isolamento de
  memória** — não muda em nada quantos pods existem nem quando.
- **`as_subflow=True` (padrão)** — o flow run criado aparece **linkado
  como filho** do flow run que chamou, na árvore de execução do Prefect
  UI. Automações **não têm esse link** hoje — um run disparado pela
  Automação 1 aparece como um flow run solto, correlacionado ao
  `check_update` só por tag/tempo, não por uma relação de verdade no
  grafo do Prefect. Isso seria uma **melhora** de observabilidade em
  relação ao que temos.
- **`parameters` é um dict nativo, não uma string Jinja.** Diferente de
  `RunDeployment.parameters` (que sempre renderiza como string, mesmo pra
  campos que deveriam ser dict — motivo de existir
  `encode_params`/`decode_params`, ver
  `pipelines/utils/automations.py`), `run_deployment()` passa o dict de
  parâmetros direto pra API de criação de flow run, que valida contra o
  schema tipado do flow (o mesmo mecanismo que fazia `coverage:
  CoverageSpec` ser validado de graça em `update_temporal_coverage` antes
  da correção do Bug 1 na parte 14 do documento principal). Ou seja: com
  `run_deployment()`, dá pra tipar `mat_test_params` como `dict` (ou
  decompor em campos tipados de verdade) e a validação do Pydantic
  acontece sozinha — o `TypeAdapter` manual que tivemos que adicionar como
  correção deixaria de ser necessário pra qualquer flow novo escrito
  assim (não muda o `mat_test_flow` já existente, que continua
  funcionando do jeito que está).

## Comparação

|                                                                                                  | Automação (atual)                                                                                                                                                     | `run_deployment()` direto                                                                                                                                                                                                          |
| ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Deployments/pods                                                                                 | 3, sem mudança                                                                                                                                                        | 3, sem mudança                                                                                                                                                                                                                     |
| Overhead de agendar pod no hop                                                                   | ~70-90s (medido, ver `automacao-vs-subflow.md`)                                                                                                                       | ~70-90s — **mesmo mecanismo**, não muda                                                                                                                                                                                            |
| Onde mora a lógica "quem dispara quem"                                                           | Objeto `Automation` separado no Prefect (fora do código Python)                                                                                                       | Dentro do código do flow upstream (`check_update_flow` chama `run_deployment(...)` explicitamente)                                                                                                                                 |
| Passagem de parâmetro                                                                            | JSON via `encode_params`/`decode_params` (Jinja nunca preserva tipo nativo)                                                                                           | Dict nativo, validado pelo schema tipado do flow — sem workaround                                                                                                                                                                  |
| Automação em massa pros 82 datasets                                                              | Precisa de mecanismo de descoberta por tag + upsert (`automacoes-em-massa.md`, não implementado)                                                                      | **Não existe o problema** — a lógica de disparo já mora no código de cada dataset, que já sabe seu próprio `dataset_id`/nome de deployment via `constants.py`                                                                      |
| Lineage no Prefect UI                                                                            | Flow run disparado aparece solto, correlacionado só por tag/tempo                                                                                                     | `as_subflow=True` linka como filho de verdade na árvore de execução                                                                                                                                                                |
| Auditoria de disparo                                                                             | `prefect.automation.triggered`/`.action.executed`/`.action.failed`, consultável via `/api/automations/filter`, visível numa aba dedicada do Prefect UI sem ler código | Só o log do próprio flow upstream (`log(f"disparando flow_download run {fr.id}")`) — não existe visão agregada de "todas as cadeias" fora do código                                                                                |
| Semântica de falha no disparo                                                                    | Desacoplada — se a ação falhar (`InvalidJinja`, API blip), o `check_update_flow` já tinha terminado `Completed` antes, sem saber                                      | Acoplada — uma falha ao criar o flow run downstream levanta exceção **dentro** do `check_update_flow`, marcando ele como `Failed` (a menos que envolva em `try/except`)                                                            |
| Gatilho externo (algo fora desta cadeia disparando uma etapa)                                    | Automação permite qualquer evento externo casar com o mesmo `match`/`match_related`                                                                                   | Só dispara de onde o código chama — quem quiser disparar de outro lugar precisa chamar explicitamente                                                                                                                              |
| Trigger patterns mais ricos (threshold, composto, reagir a estado de flow run, posture proativa) | Suportado nativamente pelo Prefect, sem mudar código                                                                                                                  | Precisaria ser escrito à mão em Python se algum dia for necessário                                                                                                                                                                 |
| Testável localmente sem subir pod                                                                | Não — precisa deploy + disparo real + esperar pod                                                                                                                     | Parcialmente melhor: a chamada em si (`run_deployment(...)`) ainda cria um flow run remoto (não roda local), mas a lógica de **qual** deployment/parâmetros chamar é só código Python comum, testável com mock de `run_deployment` |
| Isolamento entre etapas (tags, `match_related`)                                                  | Necessário, construído nesta sessão (parte 9 do documento principal)                                                                                                  | Não existe essa categoria de bug — não há evento pra casar errado                                                                                                                                                                  |

## O que sobra e o que morre em `pipelines/utils/automations.py`

Levantamento concreto do que cada função vira, caso a migração aconteça:

| Função hoje                                                                                                    | Destino                                                                                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `dataset_resource_id()`                                                                                        | Morre — só existia pra `emit_event`/`match` de automação                                                                                                                                                                       |
| `event_name()`                                                                                                 | Morre — idem                                                                                                                                                                                                                   |
| `etapa_tag()`                                                                                                  | Morre como função, mas a convenção `etapa:<etapa>` continua útil como tag de deploy (organização/descoberta manual no Prefect UI), sem uso funcional                                                                           |
| `deploy_tags()`                                                                                                | Mantém — ainda útil pra achar deployments relacionados no Prefect UI/CI, mesmo sem automação dependendo dela                                                                                                                   |
| `payload_parameters()`                                                                                         | Morre — só existia pra montar o dict Jinja `{{ event.payload.x }}`                                                                                                                                                             |
| `build_chained_automation()`                                                                                   | Morre — não há mais `Automation` pra construir                                                                                                                                                                                 |
| `encode_params()`/`decode_params()`                                                                            | Morrem **se** os flows forem migrados pra receber parâmetros tipados de verdade (dict nativo ou campos soltos) em vez de uma string JSON — é uma mudança de assinatura de flow, não só do disparo, então pode ficar pra depois |
| `check_update_and_emit()`                                                                                      | Vira `check_update_and_dispatch()` — mesmo poll/commit, mas troca `emit_event(...)` por `run_deployment(name=..., parameters=..., timeout=0)` no final                                                                         |
| Imports `Automation`, `EventTrigger`, `Posture`, `ResourceSpecification`, `RunDeployment` (ação), `emit_event` | Morrem todos                                                                                                                                                                                                                   |

Novo helper que entraria no lugar: algo como
`deployment_name(dataset_id: str, etapa: str) -> str`, resolvendo o nome
completo `"<flow name>: <etapa>/<etapa>_flow"` por convenção — usado tanto
pelo disparo quanto por qualquer tooling de CI que ainda queira achar
deployments por dataset/etapa sem depender de automação.

## Exemplo concreto — `check_update_and_emit` vira `check_update_and_dispatch`

Antes (`pipelines/utils/automations.py`, hoje):

```python
download_params = {"reference_date": reference_date.isoformat(), **(extra_download_params or {})}
emit_event(
    event=event_name(upstream_etapa),
    resource={"prefect.resource.id": dataset_resource_id(resource_dataset_id)},
    payload={"download_params": encode_params(download_params)},
)
```

Depois (proposta):

```python
from prefect.deployments import run_deployment

download_params = {"reference_date": reference_date.isoformat(), **(extra_download_params or {})}
run_deployment(
    name=deployment_name(resource_dataset_id, "flow_download"),
    parameters={"download_params": download_params},  # dict nativo, sem encode_params
    timeout=0,       # não espera o flow_download terminar
    as_subflow=True, # aparece como filho na árvore do Prefect UI
)
```

`flow_download_flow` receberia `download_params: dict` (tipo real) em vez
de `download_params: str` — ou, melhor ainda, `reference_date: str`
direto como parâmetro solto, já que não existe mais a limitação do Jinja
que motivou empacotar tudo num JSON.

O mesmo padrão se aplica ao segundo hop
(`emit_flow_download_completed` → chamada direta de `run_deployment()`
pro `mat_test_flow`, com `dataset_id`/`table_id`/`mat_test_params` como
argumentos nativos de novo).

## O que isso elimina de graça

- **`scripts/pilot_event_automations.py`** inteiro — não precisa mais
  existir. Nenhuma automação pra criar/manter em sincronia com
  deployments.
- **Todo o design de `automacoes-em-massa.md`** fica obsoleto — o
  problema que ele resolve (como criar automações pros 82 datasets sem
  script manual) simplesmente não existe mais: a lógica de disparo já é
  código do próprio dataset (`check_update_flow` já sabe seu
  `DATASET_ID`), não um objeto externo que precisa ser mantido em
  sincronia.
- Automações reais já criadas no workspace (Automação 1 e 2 do piloto)
  precisariam ser **deletadas** (`Automation.delete()`) pra não ficarem
  órfãs/inertes no Prefect (não causam disparo duplo — sem `emit_event`
  não há evento pra casar —, mas ficam como lixo confuso na aba de
  Automações).

## O que se perde

- **Visão agregada sem ler código.** Hoje dá pra abrir a aba de
  Automações do Prefect e ver todas as cadeias configuradas, sem abrir o
  repositório. Com `run_deployment()`, a única forma de saber "o que
  dispara o quê" é ler o código de cada `check_update_flow`/
  `flow_download_flow`.
- **Gatilho por qualquer evento externo.** Pro `mat_test_flow` genérico
  em particular, automação permite que **qualquer coisa** (um botão
  manual, outro pipeline, um evento de terceiro) dispare materialização
  só emitindo o evento certo — `run_deployment()` só dispara de onde o
  código efetivamente chama.
- **Trigger patterns compostos/proativos** (threshold, "se X não
  acontecer em N minutos", reagir a estado de flow run) — não usados
  hoje, mas ficam disponíveis de graça com automação, exigiriam código
  novo com `run_deployment()`.
- **Desacoplamento de falha.** Hoje, se o disparo do próximo estágio
  falhar por um blip da API, o flow que já tinha rodado com sucesso
  continua `Completed` — só a automação fica com uma falha registrada,
  sem afetar o histórico/alerta do flow que já terminou certo. Com
  `run_deployment()` inline, essa falha vira uma exceção dentro do
  próprio flow upstream, marcando ele como `Failed` mesmo que o trabalho
  dele (ex. o `poll`/`commit` do `check_update`) tenha sido concluído
  corretamente antes — pode disparar alerta/retry desnecessário do
  `check_update` por causa de um problema que é só do disparo.

## Status

Investigação registrada, sem decisão. Os dois argumentos que motivaram
manter automação em `automacao-vs-subflow.md` (overhead de pod
irrelevante pra flow agendado, isolamento de memória) **não se aplicam
aqui** — nenhum dos dois muda com esta proposta, porque os 3 deployments
continuam separados do jeito que estão. A pergunta real, caso isto seja
revisitado, é só: **o ganho de eliminar a complexidade da automação em
massa + o workaround de Jinja compensa perder a visão agregada de
automações e o desacoplamento de falha?** Fica pra quando o rollout real
for planejado.

## Recomendação (opinião registrada, não é decisão)

Perguntado diretamente qual caminho seguir — a favor de `run_deployment()`,
por três motivos, em ordem de peso:

1. **Evita construir infraestrutura que ainda não existe.** O mecanismo
   de descoberta em massa (`automacoes-em-massa.md`) é um sistema novo
   inteiro a ser desenhado/implementado/mantido, só pra resolver um
   problema que `run_deployment()` não tem — a lógica de disparo já mora
   no código de cada dataset, que já sabe seu próprio `dataset_id`. Zero
   datasets reais migrados ainda torna este o momento mais barato pra
   evitar essa dívida, antes dela virar peça que 82 datasets dependem.
2. **Resolve o Bug 1 (parte 14) na raiz, não com patch.** Parâmetro
   tipado de verdade restaura a validação automática do Pydantic — o
   `TypeAdapter` manual deixaria de ser necessário pra qualquer flow novo
   escrito assim.
3. **Lineage de verdade no Prefect UI** (`as_subflow=True`) — ganho
   tangível de observabilidade; boa parte do trabalho desta sessão foi
   correlacionar flow runs manualmente por tag/tempo entre deployments
   separados.

O que se perde (gatilho por evento externo, trigger patterns compostos,
visão agregada sem ler código) é **capacidade não usada por nada no
design atual** — otimizar por uma flexibilidade especulativa que a
pipeline não pede hoje. O risco real é o acoplamento de falha (uma
`RunDeployment` de automação falhando não derruba o flow upstream já
`Completed`; um `run_deployment()` falhando, sim) — mas é uma falha
estreita (só criar o flow run, não esperar ele terminar) e mitigável com
`try/except` + log, sem reintroduzir complexidade grande.

Decisão final ainda cabe ao usuário — não implementado.

## Implementado (2026-09-01)

Usuário decidiu seguir a recomendação: "Vamos com run_deployment então,
implementa." Antes de mexer no código, backup da branch com a
implementação de automação:
`backup/event-pipeline-automations-poc-com-automacao` (local + pushada
pro remoto), apontando pro commit `baeadddc` — se algum dia precisar
comparar ou reverter, o estado "com automação" está preservado ali,
intocado.

Mudanças (commit `5c6412ac`, branch `feat/event-pipeline-automations-poc`):

- **`pipelines/utils/automations.py`** — `check_update_and_emit` virou
  `check_update_and_dispatch` (chama `run_deployment(timeout=0,
  as_subflow=True)` em vez de `emit_event`). Removidos:
  `dataset_resource_id`, `event_name`, `payload_parameters`,
  `build_chained_automation`, e os imports de `Automation`/`EventTrigger`/
  `ResourceSpecification`/`emit_event`/`Posture`. Removidos também
  `encode_params`/`decode_params` — não fazem mais falta, `parameters` do
  `run_deployment()` já é dict nativo. Nova função `deployment_name(dataset_id,
  etapa)` resolve o identificador `"<flow name>/<deployment name>"` por
  convenção (`mat_test` tem nome fixo, genérico; as outras etapas seguem
  `"<dataset_id>: <etapa>/<etapa>_flow"`).
- **`pipelines/datasets/test_event_pipeline/tasks.py`** —
  `emit_flow_download_completed` virou `dispatch_mat_test`, chamando
  `run_deployment()` com `coverage`/`env`/`bq_project`/`prefect_mode`/
  `targets` como kwargs nativos (não mais um `mat_test_params` serializado).
- **`pipelines/datasets/test_event_pipeline/flows.py`** —
  `flow_download_flow(download_params: dict)`, não mais `str` +
  `decode_params`.
- **`pipelines/utils/metadata/flows.py`** — `mat_test_flow` ganhou
  parâmetros tipados de verdade (`coverage: CoverageSpec`, `env`,
  `bq_project`, `prefect_mode`, `targets`, `partition_folders`,
  `download_billing_project`) no lugar de `mat_test_params: str`. O
  `TypeAdapter`/`_coverage_adapter` manual da parte 14 **não existe mais**
  — a validação do Pydantic volta a acontecer automaticamente, porque
  `coverage` é de novo um parâmetro de flow tipado de verdade (mesmo
  mecanismo que já funcionava em `update_temporal_coverage`).
- **`scripts/pilot_event_automations.py`** — deletado, não faz mais
  sentido existir.

**Teste real, primeira tentativa, sucesso completo**: redeployado
`test_event_pipeline` (pool `basedosdados-dev`) e `mat_test_flow`/
`update_temporal_coverage` (pool `basedosdados`, prod). Disparado
`flow_download_flow` direto (`download_params` passado como objeto JSON
de verdade, não mais uma string `encode_params`) — flow run
`zealous-bullfrog` (`55ad1b1b-...`), `Completed`. Ele disparou
`mat_test_flow` via `run_deployment()`: flow run `piquant-cuscus`
(`f6d8b960-...`), e o log confirmou **"Beginning subflow run
'piquant-cuscus' for flow 'mat_test'"** — a promessa do `as_subflow=True`
se confirmou na prática, é um link de verdade na árvore de execução, não
uma correlação por tag/tempo como as automações davam. Sequência completa
passou: rename ✅, `dbt run/test` dev ✅, `transfer_files_to_prod_flow`
(subflow) ✅, `dbt run/test` prod ✅, `register_table_materialization_task`
✅ (coverage atualizada pra `2026-09-01` de novo). `Completed` limpo, sem
nenhum workaround de JSON/Jinja em nenhum ponto da cadeia.

**Automações antigas deletadas** do workspace real do Prefect, só depois
de confirmar o caminho novo funcionando (pedido explícito do usuário: "Vamos
deletar depois que confirmar que o caminho novo está funcionando"):
`test-event-pipeline: check_update -> flow_download` (id
`6c46e74d-501c-48e5-8697-b076adbd0497`) e `test-event-pipeline:
flow_download -> mat_test` (id `be11fd83-5f0c-495a-ba6b-3c5ead7084e7`),
via `Automation.read(name=...).delete()`, confirmado que ambas não existem
mais (`Automation.read` levanta `ValueError`).

**`deploy_tags`/`etapa_tag` mantidos** — pergunta do usuário ("teoricamente
não precisamos mais das tags, né?") respondida: não são mais
funcionalmente necessários (dispatch usa `deployment_name()` por
convenção, não tag), mas continuam com valor de descoberta/organização no
Prefect UI (filtrar por `dataset:X`/`etapa:Y` sem abrir código), então
ficaram no código sem mudança.

**`automacoes-em-massa.md` está obsoleto** — o problema que ele resolvia
(automação em massa pros 82 datasets sem script manual) não existe mais
com `run_deployment()`.
