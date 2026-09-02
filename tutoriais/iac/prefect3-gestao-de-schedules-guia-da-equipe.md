# Gestão de Schedules — Guia da Equipe

Sistema automático que monitora falhas de flows no Prefect 3 e desativa schedules de flows com problemas recorrentes.

---

## Visão geral

```
PR merged
   └─ CI deploys flows com paused=True
         └─ CI chama POST /admin-tools/sync-deployments/
               └─ Backend ativa/pausa cada flow conforme o banco

Flow falha no Prefect 3
   └─ Automação "flow-failed-prod-webhook" dispara
         └─ Chama POST /admin-tools/flow-failed/
               └─ Backend avalia se deve desativar
                     └─ Se sim: pausa no Prefect 3 + atualiza banco
```

O banco de dados é sempre a fonte de verdade. O Prefect 3 é ajustado para refletir o que está no banco.

---

## Fluxo de deploy

Quando um PR é mergeado no repositório `pipelines` e o CD (`cd-prefect3.yaml`) roda:

1. **Deploy dos flows** — `deploy_flows.py` faz deploy de todos os flows com `paused=True`. Isso garante que flows novos nunca entrem ativos por padrão.

2. **Sync com o backend** — o CI chama o endpoint de sync automaticamente.

3. **O sync aplica o estado do banco no Prefect 3:**
   - Flow já conhecido + ativo no banco → despausa no Prefect 3
   - Flow já conhecido + pausado no banco → mantém pausado
   - Flow novo (não existe no banco) → cria registro pausado, permanece pausado

> O sync **nunca altera** `is_schedule_active` no banco — só lê. Ele corrige o Prefect 3 para refletir o banco.

**Verificar no log do backend após deploy:**
```
Sync complete: {'created': 0, 'updated': 0, 'activated': 141, 'paused': 53, 'errors': 0}
```

---

## Endpoints

### `POST /admin-tools/sync-deployments/`

Sincroniza o Prefect 3 com o estado do banco. Chamado pelo CI após cada deploy.

**Resposta:**
```json
{"created": 0, "updated": 0, "activated": 141, "paused": 53, "errors": 0}
```

| Campo | Significado |
|---|---|
| `created` | Flows novos registrados no banco (ficam pausados) |
| `updated` | Flows com `deployment_id` atualizado (re-deploy) |
| `activated` | Flows que estavam pausados no Prefect 3 mas ativos no banco — despaused |
| `paused` | Flows pausados no banco, confirmados pausados no Prefect 3 |
| `errors` | Falhas ao processar um deployment específico |

---

### `POST /admin-tools/flow-failed/`

Recebe notificações de falha da automação do Prefect 3. Decide se o flow deve ser desativado.

**Lógica de desativação (condições OR):**

| Condição            | Critério                                                                   |
| ------------------- | -------------------------------------------------------------------------- |
| Falha no dbt        | Task `run_dbt` falhou com erro não ignorável após `reactivated_at`         |
| Falhas consecutivas | Últimos 2 runs terminais falharam e o mais recente é após `reactivated_at` |

**Respostas possíveis:**

| `action` | Significado |
|---|---|
| `disabled` | Flow desativado — pausado no Prefect 3 e banco atualizado |
| `no_action` | Condições não atingidas — flow permanece ativo |
| `already_paused` | Flow já estava inativo no banco |
| `ignored_unknown` | `deployment_id` não encontrado no banco |

---

## Automação Prefect 3

A automação `flow-failed-prod-webhook` dispara em todo evento `prefect.flow-run.Failed` e chama o endpoint `/admin-tools/flow-failed/`.

| Recurso | Valor |
|---|---|
| Automação | `flow-failed-prod-webhook` (id: `a6b04de5`) |
| Bloco | `flow-failed-prod` (id: `2488ee16`) |
| Trigger | `prefect.flow-run.Failed` |

---

## Django Admin

Acesse `https://backend.basedosdados.org/admin/admin_data_tools/disabledflowschedule/` para visualizar e gerenciar os flows.

### Campos do modelo

| Campo | Descrição |
|---|---|
| `flow_name` | Nome do deployment no Prefect 3 |
| `deployment_id` | UUID do deployment no Prefect 3 |
| `is_schedule_active` | `True` = ativo, `False` = pausado |
| `disabled_at` | Quando foi desativado (atualizado automaticamente pelo webhook) |
| `reactivated_at` | Quando foi reativado por um admin — falhas antes dessa data são ignoradas |

### Ativar um flow

1. Abra o flow no Django Admin
2. Marque `is_schedule_active = True`
3. Preencha `reactivated_at` com a data/hora atual — isso zera o histórico de falhas anteriores
4. Salve — o admin chama `set_paused(False)` no Prefect 3 automaticamente

### Desativar um flow

1. Abra o flow no Django Admin
2. Desmarque `is_schedule_active`
3. Salve — o admin chama `set_paused(True)` no Prefect 3 automaticamente

> `reactivated_at` é crítico: sem ele, uma falha anterior ao fix do flow pode reativar a desativação imediatamente após a reativação.

---

## Monitoramento

### Grafana — Logs da automação

**Webhook recebido e ações:**
```logql
{app="$environment"} |~ "webhook received|Disabled.*after failure|Unknown deployment|already_paused" | json | line_format "[{{ .record_time_repr | trunc 16 }}] {{ .record_message }}"
```

**Erros de autenticação no webhook:**
```logql
{app="$environment"} |= "Unauthorized" |= "flow-failed" | json | line_format "[{{ .record_time_repr | trunc 16 }}] {{ .record_message }}"
```

**Sync do CI após deploy:**
```logql
{app="$environment"} |= "Sync complete" | json | line_format "[{{ .record_time_repr | trunc 16 }}] {{ .record_message }}"
```

### Verificar sincronização banco ↔ Prefect 3

Rodar no shell do pod para detectar divergências:

```python
from backend.apps.admin_data_tools.models import DisabledFlowSchedule
from backend.apps.admin_data_tools._prefect3_client import Prefect3Client

client = Prefect3Client()
for dep in client.iter_deployments():
    try:
        record = DisabledFlowSchedule.objects.get(deployment_id=dep['id'])
        prefect_paused = dep.get('paused', False)
        db_active = record.is_schedule_active
        if prefect_paused == db_active:
            print(f'DIVERGÊNCIA: {record.flow_name} | DB active={db_active} | Prefect paused={prefect_paused}')
    except DisabledFlowSchedule.DoesNotExist:
        print(f'SEM REGISTRO: {dep["name"]}')
```

---

## Operações comuns

### Forçar desativação de flows com falhas acumuladas

Útil após um período em que a automação estava fora:

```python
from backend.apps.admin_data_tools.models import DisabledFlowSchedule
from backend.apps.admin_data_tools.views import _is_consecutive_failure, _is_dbt_failure
from backend.apps.admin_data_tools._prefect3_client import Prefect3Client
from datetime import datetime, timezone

client = Prefect3Client()
for record in DisabledFlowSchedule.objects.filter(is_schedule_active=True):
    runs = client.get_recent_completed_runs(record.deployment_id, limit=2)
    should_disable = _is_consecutive_failure(runs, record.reactivated_at)
    if not should_disable and runs:
        task_runs = client.get_failed_task_runs(runs[0]['id'])
        should_disable = _is_dbt_failure(task_runs, runs[0]['start_time'], record.reactivated_at)
    if should_disable:
        client.set_paused(record.deployment_id, paused=True)
        record.is_schedule_active = False
        record.reactivated_at = None
        record.disabled_at = datetime.now(tz=timezone.utc)
        record.save(update_fields=['is_schedule_active', 'reactivated_at', 'disabled_at'])
        print(f'Desativado: {record.flow_name}')
```

---

## Troubleshooting

| Sintoma | Causa provável | Ação |
|---|---|---|
| Flow falhou mas não foi desativado | Automação não disparou ou webhook retornou erro | Checar logs do Grafana — buscar `webhook received` no horário da falha |
| Sync do CI retorna 401 | Chave errada no secret do pod | Verificar sealed secret no IAC |
| Flow ativo no banco mas pausado no Prefect | Divergência pós-deploy ou intervenção manual | Rodar sync via Django Admin ou curl |
| `Sync complete` não aparece nos logs após deploy | Step do CI falhou silenciosamente | Verificar log da action no GitHub |

---

## Referência rápida

| Recurso | Valor |
|---|---|
| Endpoint sync | `POST https://backend.basedosdados.org/admin-tools/sync-deployments/` |
| Endpoint webhook | `POST https://backend.basedosdados.org/admin-tools/flow-failed/` |
| Django Admin | `https://backend.basedosdados.org/admin/admin_data_tools/disabledflowschedule/` |
| Automação prod | `flow-failed-prod-webhook` (id: `a6b04de5`) |
| Bloco prod | `flow-failed-prod` (id: `2488ee16`) |
