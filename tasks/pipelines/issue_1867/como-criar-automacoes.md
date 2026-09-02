---
repo: basedosdados/pipelines
relacionado: issue #1867 (ver issue-1867-pipeline-eventos.md)
atualizado: 2026-08-31
---

# Como criar uma automação de encadeamento de flows (Prefect 3)

> **Obsoleto pra #1867.** A cadeia passou a usar `run_deployment()` em vez
> de `Automation` (ver
> [run-deployment-vs-automacao.md](./run-deployment-vs-automacao.md),
> decidido e implementado em 2026-09-01) — as duas automações descritas
> aqui foram deletadas do workspace real. Este guia continua válido como
> referência de **como criar uma `Automation` no Prefect 3** caso um caso
> de uso legítimo apareça no futuro (ex. algo que precise de gatilho por
> evento externo, fora do controle de um flow específico — automação
> permite isso, `run_deployment()` não) — só não é mais o padrão usado
> pela #1867.

Guia prático — passo a passo do que foi efetivamente usado para criar a
Automação 1 (`check_update` → `flow_download`) do piloto `test_event_pipeline`.
Serve de referência pra criar as próximas (Automação 2, e as reais dos
datasets quando a #1867 for implementada).

**Validado de ponta a ponta em 2026-08-31**: disparo real via deployment no
worker do pool `basedosdados-dev` — `check_update_flow` calculou
`reference_date`, emitiu o evento, a Automação 1 disparou sozinha o
`flow_download_flow`, que recebeu e decodificou os parâmetros corretos.
Detalhes em `issue-1867-pipeline-eventos.md`, "Progresso — parte 5".

## Pré-requisito conceitual

Uma automação de encadeamento liga dois flows por evento: o flow upstream
termina e emite um evento custom (`emit_event`); uma `Automation` reage a
esse evento e dispara o deployment do flow downstream, passando parâmetros
computados em runtime via template Jinja no payload do evento.

Toda a convenção de nomes e o template Jinja ficam centralizados em
`pipelines/utils/automations.py` — **novas automações devem usar esse
módulo, não recriar `EventTrigger`/`RunDeployment` na mão**.

## Passo 1 — upstream: task que emite o evento

No `tasks.py` do dataset, uma task que chama `emit_event`, usando as
convenções do módulo compartilhado (não strings soltas). Os parâmetros
computados vão **todos dentro de um dict** (`download_params`), serializado
com `encode_params` — não um campo por parâmetro (ver "por que um dict e não
campos soltos" logo abaixo):

```python
from prefect.events.utilities import emit_event
from pipelines.utils.automations import dataset_resource_id, encode_params, event_name

@task
def emit_check_update_completed(has_new_data: bool, download_params: dict) -> None:
    if not has_new_data:
        print("Nenhum dado novo — não emite evento, downstream não dispara.")
        return
    emit_event(
        event=event_name("check_update"),
        resource={"prefect.resource.id": dataset_resource_id(DATASET_ID)},
        payload={"download_params": encode_params(download_params)},
    )
```

O flow upstream monta o dict — e **calcula** o que precisar ser calculado,
em vez de só repassar parâmetro de entrada (senão não prova nada sobre
propagar valor computado em runtime):

```python
@flow(name="test_event_pipeline: check_update", log_prints=True)
def check_update_flow(has_new_data: bool = True, source_url: str = "...") -> None:
    reference_date = datetime.now(timezone.utc).date().isoformat()  # computado aqui
    emit_check_update_completed(
        has_new_data=has_new_data,
        download_params={"reference_date": reference_date, "source_url": source_url},
    )
```

Pontos que importam:
- A decisão de emitir ou não fica **na task**, não no corpo do `@flow` — o
  flow só chama a task. Assim datasets com o mesmo padrão de check_update
  não repetem esse `if`.
- O `payload` deve conter todo valor que o downstream vai precisar via
  Jinja — só parâmetros de entrada do flow **não bastam** pra passar algo
  computado em runtime (ex. uma `reference_date` calculada durante a
  execução). É por isso que se usa `emit_event` em vez de confiar só no
  evento nativo `prefect.flow-run.Completed`. Se o valor só está sendo
  repassado sem cálculo nenhum, isso não prova nada — computar de verdade.

**Por que um dict serializado (`download_params`) e não um campo por
parâmetro:** o Jinja das automações do Prefect **sempre renderiza como
string** (confirmado em
`prefect/server/utilities/user_templates.py::render_user_template` — não
existe modo "tipo nativo"). Um `RunDeployment.parameters` como
`{"campo": "{{ event.payload.um_dict }}"}` chegaria no flow downstream como
a `repr()` em Python do dict, não um dict de verdade — quebra se o
parâmetro for tipado. `encode_params`/`decode_params` (JSON) contornam isso
e, de quebra, deixam o conjunto de campos **livre**: cada dataset pode
precisar de um número diferente de parâmetros sem precisar redesenhar a
automação — só o nome do campo `download_params` é fixo, o conteúdo não.

Exemplo completo: `pipelines/datasets/test_event_pipeline/tasks.py` e
`flows.py`.

## Passo 2 — downstream: flow que recebe os parâmetros

O flow que vai ser disparado pela automação recebe a string JSON e faz
`decode_params`:

```python
from pipelines.utils.automations import decode_params

@flow(name="test_event_pipeline: flow_download", log_prints=True)
def flow_download_flow(download_params: str) -> None:
    simulate_download(download_params=download_params)

@task
def simulate_download(download_params: str) -> str:
    params = decode_params(download_params)
    reference_date = params["reference_date"]
    source_url = params["source_url"]
    ...
```

Ele também deve continuar chamável **diretamente** (sem passar pela
automação) — é assim que um rerun manual/debug funciona, montando o JSON à
mão com `encode_params({...})`.

## Passo 2.5 — marcar os flows com `deploy_tags`

Antes de deployar, marcar cada flow com a etapa que ele representa —
permite que uma automação em massa (pros datasets reais) descubra o
deployment certo por tag, em vez de nome hardcoded:

```python
from pipelines.utils.automations import deploy_tags

check_update_flow.deploy_tags = deploy_tags(DATASET_ID, "check_update")
flow_download_flow.deploy_tags = deploy_tags(DATASET_ID, "flow_download")
```

Isso vira, no deploy, `tags=["automated-deploy", "etapa:<etapa>", "dataset:<id>"]`
(`.github/scripts/deploy_flows.py` lê `flow.deploy_tags` — atributo
opcional, aditivo, não quebra flows que não usam isso). Granularidade hoje
é **por dataset**, não por tabela — datasets reais que precisem de
automação por tabela vão exigir revisitar essa convenção.

## Passo 3 — deploy dos dois flows (upstream e downstream)

Uma automação referencia um `deployment_id`, que só existe depois que o
flow for registrado como Deployment no Prefect — não basta a função Python
existir no repo. O deploy é feito com o script de CI, mas pode ser rodado
manualmente, apontando pra uma branch já pushada:

```bash
set -a && source .env && set +a   # carrega PREFECT_API_URL/PREFECT_API_KEY do .env
                                    # (não são lidos automaticamente por um
                                    # processo Python puro, só pelo CLI do
                                    # prefect)

uv run python .github/scripts/deploy_flows.py \
  --pool basedosdados-dev \
  --branch <nome-da-branch-pushada> \
  --files pipelines/datasets/<dataset>/flows.py
```

Isso registra **todos** os flows encontrados no arquivo. Anote (ou apenas
lembre) o nome de cada deployment — segue o padrão
`<flow.name>/<nome-da-variável-python-do-flow>`, ex.:
`test_event_pipeline: flow_download/flow_download_flow`.

⚠️ Isso cria deployment real no workspace Prefect compartilhado (mesmo
escopado ao pool dev) — confirmar antes de rodar.

## Passo 4 — criar (ou atualizar) a automação via código

Script standalone, não deployado, rodado uma vez manualmente
(`scripts/pilot_event_automations.py` é o exemplo pra Automação 1). Como o
mesmo script pode rodar de novo depois de mudar o flow, ele faz
update-or-create: lê por nome primeiro, só cria se não existir.

`build_chained_automation` casa por **duas coisas**, não só o nome do
evento: `match` (resource id do dataset) e `match_related` (a tag
`etapa:<upstream_etapa>`, anexada automaticamente pelo Prefect a partir do
`deploy_tags` do deployment upstream — ver Passo 2.5). Isso evita que a
automação dispare só porque o nome do evento bateu, sem confirmar que veio
do deployment certo — testado disparando a cadeia inteira e conferindo
`prefect.automation.triggered` que cada automação só reage ao seu próprio
evento, nunca ao do vizinho.

```python
from prefect.automations import Automation
from prefect.client.orchestration import get_client
from pipelines.utils.automations import build_chained_automation

AUTOMATION_NAME = "test-event-pipeline: check_update -> flow_download"

with get_client(sync_client=True) as client:
    deployment = client.read_deployment_by_name(
        "test_event_pipeline: flow_download/flow_download_flow"
    )

desired = build_chained_automation(
    name=AUTOMATION_NAME,
    dataset_id="test_event_pipeline",
    upstream_etapa="check_update",
    downstream_deployment_id=str(deployment.id),
    payload_fields=["download_params"],  # sempre esse campo único — ver passo 1
)

try:
    existing = Automation.read(name=AUTOMATION_NAME)
except ValueError:
    existing = None

if existing is None:
    created = desired.create()
    print(f"Automação criada: {created.name} (id={created.id})")
else:
    existing.trigger = desired.trigger
    existing.actions = desired.actions
    existing.update()
    print(f"Automação atualizada: {existing.name} (id={existing.id})")
```

Rodar com as mesmas credenciais do passo 3 (`source .env` antes).

`build_chained_automation` (em `pipelines/utils/automations.py`) monta por
baixo dos panos:

```python
Automation(
    name=name,
    trigger=EventTrigger(
        expect={f"{upstream_etapa}.completed"},
        match=ResourceSpecification({"prefect.resource.id": [f"dataset.{dataset_id}"]}),
        posture="Reactive",
        threshold=1,
        within=0,
    ),
    actions=[RunDeployment(
        deployment_id=downstream_deployment_id,
        parameters={f: f"{{{{ event.payload.{f} }}}}" for f in payload_fields},
    )],
)
```

Convenção de nome da automação: prefixo `test-event-pipeline:` (ou o nome
do dataset) — facilita achar e limpar depois no workspace.

## Passo 5 — verificar

Formas de confirmar que a automação foi criada e está com o trigger certo:
- O `print` do script já devolve o `id`.
- Consultar a API REST direto (não precisa do websocket, que é só pro envio
  de eventos custom):
  ```bash
  curl -s -X POST "$PREFECT_API_URL/automations/filter" \
    -H "Authorization: Bearer $PREFECT_API_KEY" -H "Content-Type: application/json" \
    -d '{}' | python3 -m json.tool
  ```
- UI do Prefect: `https://prefect3.basedosdados.org/automations`.

## Passo 6 — testar o disparo de ponta a ponta

Disparar o flow upstream (via deployment, não chamada local ad-hoc — só
assim passa pelo `emit_event` de verdade) e verificar se o downstream roda
sozinho:
- `mcp__databasis__run_deployment` (ou `prefect deployment run`) no
  deployment do upstream.
- `mcp__databasis__list_flow_runs` / `get_flow_run_logs` pra ver se o
  deployment downstream foi disparado e recebeu os parâmetros certos.

⚠️ Isso sobe execução real no worker do pool (pod real, se o pool for
Kubernetes) — confirmar antes de rodar, mesmo em dev.

Formas de acompanhar o disparo, além do `run_deployment`/`list_flow_runs`:
consultar `POST /api/events/filter` (nome do evento) pra ver se ele chegou,
e `POST /api/events/filter` com `{"event": {"prefix": ["prefect.automation"]}}`
pra ver se a automação processou (`prefect.automation.triggered`,
`prefect.automation.action.triggered`, e — se algo deu errado —
`prefect.automation.action.failed`, cujo `payload.reason` traz o motivo,
ex. `InvalidJinja()` quando o `RunDeployment.parameters` referencia um
campo que não existe mais na assinatura do flow).

## Armadilhas conhecidas (encontradas criando a Automação 1)

- **Sempre commitar e pushar antes de testar** — `deploy_flows.py` usa
  `flow.from_source(GitRepository(...))`, que busca o código **do branch
  remoto**, tanto pra rodar o flow quanto pra introspectar a assinatura de
  parâmetros no momento do deploy. Rodar `deploy_flows.py` localmente com
  mudanças só no disco (não commitadas/pushadas) **não tem efeito** — ele
  redeploya sobre o código antigo do GitHub. Isso já causou um
  `InvalidJinja()` neste projeto: a automação foi criada apontando pro
  parâmetro novo (`download_params`), mas o deployment ainda tinha a
  assinatura antiga porque o código não tinha sido pushado. Sintoma: o
  evento chega, a automação dispara (`prefect.automation.triggered`), mas a
  ação falha (`prefect.automation.action.failed`).
- **`.env` não carrega sozinho** em processo Python puro — precisa
  `source .env` (ou equivalente) antes de rodar qualquer script que use a
  API do Prefect. O CLI (`prefect config view`) lê o `.env` automaticamente,
  mas isso não se estende a scripts Python soltos. **Sintoma traiçoeiro**:
  sem `PREFECT_API_URL`, o Prefect não dá erro — sobe silenciosamente um
  servidor efêmero local em memória (`uvicorn ...
  prefect.server.api.server:create_app`) e roda `deploy_flows.py` contra
  ele. Parece só "travado" (processo roda minutos sem output nem erro,
  nenhum processo Python visível além do próprio uvicorn efêmero) — não é
  lentidão de rede, é a variável faltando. Especialmente fácil de esquecer
  num shell/sessão em background novo, que não herda o `source .env` já
  feito no shell principal.
- **`emit_event` pode não chegar ao servidor rodando localmente** — o
  cliente usa um websocket (`wss://.../api/events/in`) separado da API REST
  usada pra tudo mais. Nesta máquina (WSL), esse handshake é redirecionado
  pra uma página de login em vez de completar — o evento se perde
  silenciosamente (a task termina em `Completed`, sem indicar falha).
  **Rodando de dentro do worker (pod no cluster), isso não acontece** —
  confirmado disparando de verdade via deployment. Ou seja, é uma limitação
  do ambiente de rede local pra testes ad-hoc, não do mecanismo em si. Pra
  confirmar que um evento chegou (local ou não), consultar
  `POST /api/events/filter` com o nome do evento — não basta ver a task
  como bem-sucedida.
- **`git push` pode falhar em WSL** se o `credential.helper` global do Git
  apontar pro `git-credential-wincred.exe` do Git for Windows (não existe
  no WSL). Resolve com `gh auth setup-git` (usa o token do `gh` já
  autenticado, sem mexer no helper global de outros hosts).
- **Jinja das automações não preserva tipo** — sempre renderiza como
  string, mesmo quando o valor original é um dict/número/bool. Um
  `RunDeployment.parameters` que tenta passar um dict inteiro
  (`{{ event.payload.um_dict }}`) sem passar por JSON vira `repr()` de
  Python no flow downstream, não um dict de verdade. Ver "por que um dict
  serializado" no Passo 1.
