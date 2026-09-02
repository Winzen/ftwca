# Planejamento — Gestão de Schedules via Backend (Prefect 3)
**Data:** 2026-07-06

---

## Contexto

O Prefect 0 tinha um comando Django (`disable_unhealthy_flow_schedules`) que:
1. Re-desativava flows que o Prefect reativava após re-registro
2. Detectava flows com falhas consecutivas e os desativava automaticamente
3. Enviava notificação no Discord

Com a migração para o Prefect 3, esse comando ficou obsoleto (usava a API GraphQL do Prefect 0). Além disso, o CI atual reseta `paused=False` em todo deploy, exigindo que um script manual seja rodado após cada merge.

---

## Objetivo

O backend passa a ser o único dono do estado de pausa dos flows. O CI nunca ativa schedules — apenas registra deployments pausados. O backend decide o que ativa ou mantém pausado, e é notificado automaticamente quando um flow falha.

---

## Arquitetura

```
CI (GitHub Actions)
  └── deploy flows com paused=True
  └── POST /api/admin/sync-deployments  ──→  Backend
                                               └── consulta Prefect 3 API
                                               └── sincroniza deployment_ids no banco
                                               └── ativa flows saudáveis
                                               └── mantém pausados os desativados

Prefect 3 (automação)
  └── evento: prefect.flow-run.Failed  ──→  Backend
                                               └── roda disable_unhealthy_flow_schedules
                                               └── pausa flows doentes via Prefect 3 API
                                               └── notifica Discord
```

---

## Blocos de trabalho

### 1. Backend — migration do modelo
**Dependência:** nenhuma (base de tudo)

O modelo `DisabledFlowSchedule` rastreia `flow_id` (UUID do Prefect 0). No Prefect 3, o identificador relevante é o `deployment_id`.

**O que fazer:**
- Adicionar campo `deployment_id = models.CharField(max_length=255, blank=True)` no `DisabledFlowSchedule`
- Criar migration Django
- `flow_id` pode ser mantido por ora (dados históricos) ou deprecated

**Arquivo:** `backend/apps/admin_data_tools/models.py`

---

### 2. Backend — management command `sync_prefect3_deployments`
**Dependência:** bloco 1

Chamado pelo CI após cada deploy. O backend consulta o Prefect 3, sincroniza os `deployment_id`s e aplica o estado correto de pausa.

**O que fazer:**
- Criar `management/commands/sync_prefect3_deployments.py`
- Lógica:
  1. Chamar `POST /deployments/filter` na API do Prefect 3 (paginado) — obter todos os deployments
  2. Para cada deployment:
     - Se não existe em `DisabledFlowSchedule`: criar registro + ativar no Prefect 3 (`PATCH /deployments/{id} {"paused": false}`)
     - Se existe e `is_schedule_active=False`: garantir que está pausado no Prefect 3
     - Se existe e `is_schedule_active=True` (reativado por admin): garantir que está ativo no Prefect 3
  3. Atualizar `deployment_id` no banco sempre (pode mudar após re-deploy)

**Variável de ambiente necessária:** `PREFECT3_API_KEY`, `PREFECT3_API_URL`

---

### 3. Backend — endpoint `POST /api/admin/sync-deployments`
**Dependência:** bloco 2

Endpoint HTTP que o CI chama após o deploy. Apenas dispara o management command `sync_prefect3_deployments`.

**O que fazer:**
- Criar view/endpoint no Django (autenticado — só o CI deve chamar)
- Recebe o POST, executa o sync de forma síncrona ou via task (Celery/thread)
- Retorna `200 OK` com resumo do sync

---

### 4. CI (pipelines) — deploy com `paused=True`
**Dependência:** bloco 3 funcionando

Hoje o CI deploya flows com schedule ativo. Precisa passar a deployar sempre pausado.

**O que fazer:**
- Identificar onde `flow.deploy()` é chamado no CI
- Adicionar `paused=True` no deploy
- Após o deploy, adicionar step que chama `POST /api/admin/sync-deployments` no backend
- Esse step substitui o script manual `tmp/prefect_pause_flows.py`

---

### 5. Backend — reescrita do `disable_unhealthy_flow_schedules` para Prefect 3
**Dependência:** bloco 1

Substituir todas as chamadas GraphQL (Prefect 0) por chamadas REST (Prefect 3).

**Mapeamento de mudanças:**

| Prefect 0 (GraphQL) | Prefect 3 (REST) |
|---|---|
| `query flow { is_schedule_active, flow_runs { state: Failed } }` | `POST /flow_runs/filter` com `{"flow_run_filter": {"state": {"name": {"any_": ["Failed"]}}, "start_time": {"after_": ...}}}` |
| `mutation set_schedule_inactive` | `PATCH /deployments/{deployment_id} {"paused": true}` |
| `query flow_run { task_runs { state: Failed } }` | `POST /task_runs/filter` com filtro por `flow_run_id` |
| `gql.Client` | `urllib.request` ou `httpx` (sem dependência extra) |

**O que fazer:**
- Reescrever `service.py` usando Prefect 3 REST API
- Substituir `flow_id` por `deployment_id` nas queries e mutations
- Adaptar `FlowDisable.validate()` — lógica de validação permanece igual (2 falhas consecutivas ou `run_dbt` falhando)
- Remover dependência do `gql` (GraphQL client)

---

### 6. Prefect 3 — automação de falha + endpoint webhook no backend
**Dependência:** bloco 5

Quando um flow falha no Prefect 3, o backend é notificado automaticamente.

**O que fazer no Prefect 3:**
- Criar automação via UI ou API:
  - Trigger: evento `prefect.flow-run.Failed`
  - Action: `Send a notification` → webhook `POST /api/admin/flow-failed`
  - Payload: `{"flow_run_id": "{{ flow_run.id }}", "deployment_id": "{{ deployment.id }}"}`

**O que fazer no backend:**
- Criar endpoint `POST /api/admin/flow-failed`
- Recebe o webhook, dispara `disable_unhealthy_flow_schedules` (bloco 5)
- Autenticar o webhook (token fixo ou HMAC)

---

## Ordem de execução

```
1 → 2 → 3 → 4    (linha principal: modelo → sync → endpoint → CI)
1 → 5 → 6         (linha paralela: modelo → disable reescrito → automação)
```

Os dois caminhos podem ser desenvolvidos em paralelo após o bloco 1.

---

## O que deixa de existir

| Hoje | Depois |
|---|---|
| Script manual `tmp/prefect_pause_flows.py` | Substituído pelo sync automático pós-deploy |
| `service.py` com GraphQL / Prefect 0 | Substituído pela versão REST / Prefect 3 |
| `constants.py` com queries GraphQL | Removido |
| `flow_id` no modelo | Substituído por `deployment_id` |

---

## Pendências a confirmar

- [ ] Autenticação do endpoint `/api/admin/sync-deployments` — qual mecanismo? (token fixo, secret do GitHub Actions)
- [ ] Autenticação do webhook `/api/admin/flow-failed` — HMAC ou token fixo?
- [ ] O sync deve ser síncrono no request ou assíncrono (Celery)?
- [ ] Manter `flow_id` no modelo para histórico ou remover na migration?
