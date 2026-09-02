# Implementação — Gestão de Schedules via Backend (Prefect 3)
**Branch backend:** `feat/prefect3-schedule-management-main` (prod) / `feat/prefect3-schedule-management` (staging)
**Branch pipelines:** `feat/prefect3-deploy-paused`

## PRs

| PR    | Repo      | Descrição                                                | Status   |
| ----- | --------- | -------------------------------------------------------- | -------- |
| #1029 | backend   | feat principal (staging)                                 | ✅ merged |
| #1031 | backend   | fix: admin usa `Prefect3Client` + resposta 204 (staging) | ✅ merged |
| #1033 | backend   | fix: sufixo hash no nome da task `run_dbt` (staging)     | ✅ merged |
| #1034 | backend   | fix: `disabled_at` atualizado ao desativar (staging)     | ✅ merged |
| #1035 | backend   | feat principal (main)                                    | ✅ merged |
| #1037 | backend   | fix: migration 0003 `disabled_at` não commitada          | ✅ merged |
| #1631 | pipelines | deploy `paused=True` + sync no CI                        | ✅ merged |
| #140  | iac       | `PREFECT3_API_KEY` sealed secrets prod + staging         | ✅ merged |


Documento de acompanhamento da implementação. Para o planejamento completo, ver:
`Planejamento — Gestão de Schedules via Backend (Prefect 3).md`

---

## Bloco 1 — Migration do modelo ✅

**Arquivo:** `backend/apps/admin_data_tools/models.py`

Renomeado o campo `flow_id` para `deployment_id`. No Prefect 3 o identificador relevante é o deployment UUID, não o flow UUID como era no Prefect 0.

```python
# antes
flow_id = models.CharField(max_length=255)

# depois
deployment_id = models.CharField(max_length=255)
```

**Arquivo:** `backend/apps/admin_data_tools/admin.py`

Todas as referências a `flow_id` atualizadas para `deployment_id`.

**Migration:** gerada pelo Django via `makemigrations` — não criada manualmente.

**`backend/apps/admin_data_tools/admin.py`** — `save_model` migrado de `FlowService` (Prefect 0) para `Prefect3Client.set_paused()` (PR #1031). Ao ativar/desativar um flow pelo Django admin, chama `PATCH /deployments/{id}` no Prefect 3 diretamente e atualiza `reactivated_at` conforme o estado.

> **Pendente — rename do modelo:**
> O nome `DisabledFlowSchedule` não reflete mais a realidade: o modelo agora registra **todos** os flows (não apenas os desativados), já que o estado padrão é pausado e o admin decide o que ativar. Um rename para algo como `FlowDeployment` foi discutido mas adiado — será feito após confirmar que as migrations estão rodando corretamente.

---

## Bloco 2 — View de sync de deployments ✅

### Arquivos criados

**`backend/apps/admin_data_tools/_prefect3_client.py`**

Cliente HTTP reutilizável para a API REST do Prefect 3. Sem dependências externas — usa apenas `urllib.request`.

Métodos:
- `iter_deployments(page_size)` — percorre todos os deployments paginado (até 10.000)
- `get_recent_completed_runs(deployment_id, limit)` — últimos runs terminais de um deployment (estados: Failed, Crashed, Completed, Cancelled)
- `get_failed_task_runs(flow_run_id)` — task runs falhados de um flow run específico (usado pelo bloco 3 para detectar falha no dbt)
- `set_paused(deployment_id, paused)` — pausa ou ativa um deployment via `PATCH /deployments/{id}`

> **Fix (PR #1031):** `_request` retornava `JSONDecodeError` ao processar resposta do `PATCH /deployments/{id}`, que retorna 204 sem body. Corrigido para retornar `None` quando o body é vazio.

**`backend/apps/admin_data_tools/views.py`**

`SyncDeploymentsView` — `POST /admin-tools/sync-deployments/`

Lógica por deployment recebido da API do Prefect 3:
- **Novo** (não existe no banco): cria registro com `is_schedule_active=False` (permanece pausado)
- **Existente com `deployment_id` diferente**: atualiza o UUID (ocorre quando um flow é re-deployado com outro nome de deployment)
- **Existente**: aplica o estado correto de pausa/ativação no Prefect 3

**Otimização — `currently_paused`:** a API do Prefect 3 retorna o campo `paused` em cada deployment. O sync só chama `set_paused` quando o estado atual difere do desejado. Sem essa verificação, cada sync fazia ~194 PATCHs desnecessários → 504 Gateway Timeout.

Retorna JSON com contadores: `created`, `updated`, `activated`, `paused`, `errors`.

**`backend/apps/admin_data_tools/urls.py`** *(novo)*

```
POST /admin-tools/sync-deployments/
POST /admin-tools/flow-failed/
```

### Arquivos modificados

**`backend/urls.py`** — inclui as URLs de `admin_data_tools`

**`backend/settings/base.py`** — adicionadas as variáveis:
```python
PREFECT3_API_URL = getenv("PREFECT3_API_URL", "https://prefect3.basedosdados.org/api")
PREFECT3_API_KEY = getenv("PREFECT3_API_KEY")
```

---

## Decisão — Autenticação dos endpoints

**Contexto:** os endpoints `POST /admin-tools/sync-deployments/` e `POST /admin-tools/flow-failed/` precisam ser autenticados.

**Opção descartada:** criar um `BACKEND_SYNC_TOKEN` separado — seria mais correto do ponto de vista de separação de responsabilidades, mas exigiria gerenciar mais um secret.

**Decisão:** reutilizar a `PREFECT3_API_KEY` como bearer token nos dois endpoints.

- O CI já tem essa chave como GitHub secret (`PREFECT3_AUTH_TOKEN`)
- O backend já precisa dela para chamar o Prefect 3
- A automação do Prefect 3 também pode reutilizá-la no header do webhook
- Nenhum secret adicional precisa ser criado ou rotacionado

**Trade-off registrado:** se a `PREFECT3_API_KEY` for rotacionada, o CI e a automação do Prefect 3 precisarão ser atualizados. Se no futuro isso se tornar um problema, criar um `BACKEND_SYNC_TOKEN` dedicado é a solução.

**Onde encontrar o valor:** é o token do usuário `api_user@basedosdados.org` no Prefect 3 — Prefect 3 UI → Settings → API Tokens. No CI está como GitHub secret `PREFECT3_AUTH_TOKEN` no repositório `pipelines`. A `PREFECT_API_KEY` no `.env` local de pipelines é a chave pessoal do dev e **não** deve ser usada aqui.

---

## Bloco 3 — View webhook de falha ✅

**Arquivo:** `backend/apps/admin_data_tools/views.py`

`FlowFailedWebhookView` — `POST /admin-tools/flow-failed/`

Payload esperado (configurado na automação do Prefect 3):
```json
{
    "deployment_id": "{{ deployment.id }}",
    "flow_run_id": "{{ flow_run.id }}",
    "flow_run_name": "{{ flow_run.name }}"
}
```

Fluxo de execução:
1. Autentica bearer token
2. Busca o registro no banco pelo `deployment_id` — se não encontrar, retorna `ignored_unknown`
3. Se `is_schedule_active=False`, retorna `already_paused`
4. Busca os últimos 2 runs terminais do deployment + task runs falhados do run atual via Prefect 3 API
5. Avalia duas condições (`OR`) para decidir se desativa:
   - **Falha no dbt** — run atual falhou na task `run_dbt` com erro não ignorável, e ocorreu após `reactivated_at` (1 falha já basta)
   - **Falhas consecutivas** — últimos 2 runs terminais falharam e o mais recente é posterior a `reactivated_at`
6. Se qualquer condição for verdadeira: pausa no Prefect 3, atualiza `is_schedule_active=False`, `reactivated_at=None` e `disabled_at=now()` no banco

Retorna JSON com `action`: `disabled`, `no_action`, `already_paused` ou `ignored_unknown`.

### Funções auxiliares

**`_after_reactivation(start_time_iso, reactivated_at)`**

Retorna `True` se `start_time_iso` é estritamente posterior a `reactivated_at`. Quando `reactivated_at` é `None`, retorna `True` (sem baseline = qualquer falha conta).

**`_is_dbt_task(name)`**

Verifica se o nome de uma task corresponde a `run_dbt`. O Prefect 3 adiciona um sufixo hash curto ao nome das tasks (ex: `run_dbt-9da`), então o match usa `startswith` em vez de igualdade exata.

**`_is_dbt_failure(task_runs, run_start_time, reactivated_at)`**

Retorna `True` se algum task run correspondente a `run_dbt` (via `_is_dbt_task`) falhou com mensagem de estado não ignorável. A lista `_STATE_MESSAGES_IGNORE` exclui timeouts de heartbeat que não indicam falha real no dbt:
```python
_STATE_MESSAGES_IGNORE = {
    "No heartbeat detected from the remote task; marking the run as failed.",
}
```

**`_is_consecutive_failure(runs, reactivated_at)`**

Retorna `True` se os últimos 2 runs terminais falharam (estado `Failed` ou `Crashed`) e o mais recente ocorreu após `reactivated_at`.

**`_check_bearer_token(request)`**

Valida o header `Authorization` contra `PREFECT3_API_KEY`.

---

## Bloco 4 — CI deploy com `paused=True` ✅

### Arquivos modificados (repositório `pipelines`)

**`.github/scripts/deploy_flows.py`**

Adicionado `paused=True` no `.deploy()`. Todos os flows chegam pausados após o deploy — em prod o backend sync ativa os que devem estar ativos; em dev é feito manualmente. Manual runs continuam funcionando independente do estado de pausa.

```python
).deploy(
    ...
    paused=True,  # schedules activated by backend sync (prod) or manually (dev)
)
```

**`.github/workflows/cd-prefect3.yaml`**

Adicionado step após o deploy que notifica o backend para rodar o sync. O `|| echo` garante que o CI não quebra se o backend sync falhar.

```yaml
- name: Sync deployments with backend
  env:
    PREFECT3_AUTH_TOKEN: ${{ secrets.PREFECT3_AUTH_TOKEN }}
  run: |-
    curl -sf -X POST https://backend.basedosdados.org/admin-tools/sync-deployments/ \
      -H "Authorization: Bearer $PREFECT3_AUTH_TOKEN" \
      -H "Content-Type: application/json" \
      || echo "⚠️ Backend sync failed — activate flows manually via Django admin"
```

---

## Bloco 5 — Lógica de disable para Prefect 3 ✅

Implementado diretamente em `views.py` — sem módulo separado, pois a lógica ficou suficientemente simples com a nova arquitetura.

A lógica original do `disable_unhealthy_flow_schedules` foi simplificada:

| Prefect 0 | Prefect 3 |
|---|---|
| Varria todos os flows da semana | Recebe o deployment específico via webhook |
| Fase 1: re-pausar flows reativados pelo Prefect | Não necessário — sync pós-deploy cobre isso |
| Fase 2: detectar novos flows com falha | Automação do Prefect 3 chama o webhook |
| GraphQL API | REST API |
| `flow_id` | `deployment_id` |

**Duas condições para desativar (OR):**

1. **Falha no dbt** — basta 1 run com task `run_dbt` falhada após `reactivated_at`
2. **Falhas consecutivas** — 2 runs terminais falhados consecutivos, o mais recente após `reactivated_at`

A separação entre as condições deixa o código legível e facilita ajustar cada uma independentemente no futuro.

---

## Bloco 6 — Automação Prefect 3 + remoção do cron ✅

### Cron removido

**`backend/apps/admin_data_tools/tasks.py`** — removido o `@db_periodic_task` do `disable_unhealthy_flow_schedules`. O arquivo ficou vazio pois era o único task.

### Automação Prefect 3 — configuração manual

A automação é criada via API do Prefect 3 (não pela UI, pois o tipo `send-notification` com bloco `custom-webhook` requer um block document).

Ver tutorial completo: `Tutoriais/Tutorial — Automação Flow Failed Webhook (Prefect 3).md`

**Configuração do trigger:**

```json
{
  "type": "event",
  "match": {"prefect.resource.id": "prefect.flow-run.*"},
  "expect": ["prefect.flow-run.Failed"],
  "for_each": ["prefect.resource.id"],
  "posture": "Reactive",
  "threshold": 1,
  "within": 0
}
```

> **Nota:** o filtro `match_related` por work pool (`prefect.resource.name`) não funciona nesta versão do Prefect 3. Flows desconhecidos são ignorados pelo endpoint via `ignored_unknown`, então a ausência do filtro não causa problema.

**Template variables corretos no Prefect 3:**

| Campo no payload | Template | Observação |
|---|---|---|
| `deployment_id` | `{{ flow_run.deployment_id }}` | `{{ deployment.id }}` não existe no contexto de eventos |
| `flow_run_id` | `{{ flow_run.id }}` | |

**Block document (`custom-webhook`):**

```json
{
  "name": "flow-failed-prod",
  "url": "https://backend.basedosdados.org/admin-tools/flow-failed/",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer <PREFECT3_API_KEY>",
    "Content-Type": "application/json"
  },
  "json_data": {
    "deployment_id": "{{subject}}",
    "flow_run_id": "{{body}}",
    "flow_run_name": ""
  },
  "timeout": 10
}
```

> O campo `name` é obrigatório no `CustomWebhookNotificationBlock` — sem ele a action falha silenciosamente.

---

## Testes locais (Docker Compose)

Ambiente: `docker compose up` no repositório `backend` com `compose.override.yaml` substituindo o container Elasticsearch por um stub Alpine (incompatibilidade do Elasticsearch 7.17.3 com WSL2 cgroup v2).

### `POST /admin-tools/sync-deployments/`

| Cenário | Resultado |
|---|---|
| Token inválido | 401 `{"error": "Unauthorized"}` ✅ |
| Token válido, primeiro sync | 200 `{"created": 194, "updated": 0, "activated": 0, "paused": 194, "errors": 0}` ✅ |
| Token válido, sync repetido (sem mudanças) | 200 sem chamadas desnecessárias ao Prefect 3 (otimização `currently_paused`) ✅ |

### `POST /admin-tools/flow-failed/`

| Cenário | Resultado |
|---|---|
| Token inválido | 401 `{"error": "Unauthorized"}` ✅ |
| `deployment_id` desconhecido | 200 `{"action": "ignored_unknown"}` ✅ |
| Flow já pausado (`is_schedule_active=False`) | 200 `{"action": "already_paused"}` ✅ |
| Flow ativo, condições não atingidas | 200 `{"action": "no_action"}` ✅ |
| Flow ativo, falha no `run_dbt` | 200 `{"action": "disabled"}` ✅ |
| Flow ativo, 2 falhas consecutivas | 200 `{"action": "disabled"}` ✅ |

### End-to-end em staging

Flow de teste: `br_bcb_sicor__operacoes_desclassificadas` (falha em ~3-5 min com `run_dbt`)

Automação Prefect 3 → bloco `custom-webhook` → webhook recebido → `action: disabled` → deployment pausado no Prefect 3 → `disabled_at` atualizado no banco ✅

### End-to-end em produção

**Data:** 2026-07-07

Flow de teste: `br_bcb_sicor__operacoes_desclassificadas` (deployment_id: `04610df7`)

Mesmo fluxo do staging: `force_run=True` → falha no `run_dbt` → automação `flow-failed-prod-webhook` disparou → webhook recebido → `action: disabled` → deployment pausado no Prefect 3 → `disabled_at: 2026-07-07 20:22:00 UTC` atualizado no banco ✅

---

## Incidente de produção — sync pausou flows saudáveis

**Data:** 2026-07-06

**O que aconteceu:**

O segundo sync foi executado antes da otimização `currently_paused` ser implementada. O banco tinha 194 registros com `is_schedule_active=False` (todos os flows eram novos), mas o Prefect tinha os flows saudáveis ativos. O sync tentou pausar todos os 194 — após ~152 chamadas, o nginx retornou 504 Gateway Timeout e o sync foi interrompido.

**Resultado:** 152 flows saudáveis foram pausados no Prefect.

**Recuperação:**

Criado script `pipelines/tmp/reactivate_healthy_flows.py` (gitignored) que ativou todos os deployments **não** presentes na lista `SHOULD_BE_PAUSED` do `prefect_pause_flows.py`.

```
Ativados: 152 | Mantidos pausados: 42 | Erros: 0
```

**Correção aplicada:**

A view `SyncDeploymentsView._sync_deployment` passou a comparar `currently_paused` (campo retornado pela API do Prefect) com `should_be_paused` (derivado de `is_schedule_active`), e só chama `set_paused` quando diferem.

**Estado atual do banco vs. Prefect:**

Corrigido via script Django shell (2026-07-06): 152 flows atualizados para `is_schedule_active=True`, banco e Prefect em sincronia.

---

## Status final

| Bloco | Descrição | Status |
|---|---|---|
| 1 | Migration do modelo (`flow_id` → `deployment_id`) | ✅ |
| 2 | View de sync de deployments | ✅ |
| 3 | View webhook de falha | ✅ |
| 4 | CI deploy com `paused=True` + chamar sync | ✅ |
| 5 | Lógica de disable para Prefect 3 REST | ✅ |
| 6 | Automação Prefect 3 + remover cron | ✅ |

## Bugs encontrados durante testes em staging

| Bug | Causa | Fix |
|---|---|---|
| Automação não disparava | `match_related` com `prefect.resource.name` inválido nesta versão | Removido o filtro — flows desconhecidos ignorados via `ignored_unknown` |
| Action falhava silenciosamente | `CustomWebhookNotificationBlock` exige campo `name` nos dados do bloco | Adicionado campo `name` ao criar o block document |
| `{{ deployment.id }}` não existe no contexto de eventos | Variável errada no template da automação | Corrigido para `{{ flow_run.deployment_id }}` |
| Task `run_dbt` nunca detectada | Prefect 3 adiciona sufixo hash ao nome (`run_dbt-9da`) | Corrigido com `_is_dbt_task()` usando `startswith` — PR #1033 |
| `disabled_at` não atualizava | Campo com `auto_now_add=True` (somente na criação) | Alterado para `default=timezone.now` + view seta `disabled_at=now()` — PR #1034 |

---

## Deploy e validação em produção (2026-07-07)

### Configuração inicial

1. **Merge PR #1035** → deploy automático via CI
2. **Migration 0003** — a migration não estava commitada (gerada pelo entrypoint do container em staging e aplicada direto no banco). Criada manualmente na branch `fix/migration-0003-disabled-at` e mergeada via PR #1037
3. **Block de prod** criado: `flow-failed-prod` (id: `2488ee16`)
4. **Automação de prod** criada: `flow-failed-prod-webhook` (id: `a6b04de5`)
5. **Sync inicial em prod**: `{"created": 183, "updated": 0, "activated": 0, "paused": 11, "errors": 0}` — 183 flows novos + 11 existentes (intencionalmente desativados)
6. **Ativação dos 183 flows**: todos os flows criados pelo sync (que estavam saudáveis) ativados via shell Django — 0 erros
7. **Limpeza de orphans Prefect 0**: 64 registros do banco que não correspondiam a nenhum deployment do Prefect 3 (flows do Prefect 0 com `disabled_at` entre maio e junho 2026) deletados — banco ficou com 194 registros, igual ao Prefect 3 e ao staging
8. **Automação de staging** (`flow-failed-staging-webhook`, id: `71ee3136`) desativada — mantida para uso em desenvolvimento futuro

### Estado final em produção

| Ambiente | Flows no Prefect 3 | Flows no banco | Ativos | Pausados |
|---|---|---|---|---|
| Staging | 194 | 194 | 150 | 44 |
| Prod | 194 | 194 | 183 | 11 |

---

## Pendências

- [x] Corrigir DB vs. Prefect fora de sincronia (staging) — 2026-07-06
- [x] Adicionar `PREFECT3_API_KEY` nas env vars (IAC PR #140)
- [x] Migrar `save_model` do admin para `Prefect3Client` (PR #1031)
- [x] Validar end-to-end em staging ✅
- [x] Merge do PR #1035 e deploy em produção ✅
- [x] Rodar migrations em produção (`0002` via PR #1035, `0003` via PR #1037) ✅
- [x] Rodar sync inicial em produção ✅
- [x] Criar automação no Prefect 3 apontando para prod ✅
- [x] Validar end-to-end em produção ✅
- [ ] Deletar `tmp/prefect_pause_flows.py` no repositório `pipelines`
- [ ] Rename do modelo `DisabledFlowSchedule` → nome mais adequado
