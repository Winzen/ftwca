# Automação — Flow Failed Webhook (Prefect 3)

Documenta como criar a automação no Prefect 3 que chama o endpoint `/admin-tools/flow-failed/` do backend quando um flow falha.

A automação usa a API REST do Prefect 3 diretamente — sem acesso à UI.

---

## Pré-requisitos

- `PREFECT3_API_KEY` disponível
- `PREFECT3_API_URL` = `https://prefect3.basedosdados.org/api`
- Endpoint `/admin-tools/flow-failed/` disponível no backend alvo (staging ou prod)

Todas as chamadas abaixo usam:
```bash
API="https://prefect3.basedosdados.org/api"
KEY="<PREFECT3_API_KEY>"
```

---

## Contexto — limitações da API

A versão do Prefect 3 em uso **não suporta** o tipo de ação `call-webhook` diretamente nas automações. Os tipos de ação disponíveis são: `do-nothing`, `run-deployment`, `pause-deployment`, `resume-deployment`, `send-notification`, entre outros.

A solução é usar `send-notification` com um bloco do tipo **Custom Webhook** (`custom-webhook`). O bloco permite configurar URL, headers e body como JSON. As variáveis dinâmicas por notificação são injetadas via `{{subject}}` e `{{body}}` no `json_data` do bloco.

---

## Passo 1 — Obter o ID do block type `custom-webhook`

```bash
curl -sL -X POST "$API/block_types/filter" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -c "
import json, sys
for b in json.load(sys.stdin):
    if b.get('slug') == 'custom-webhook':
        print('block_type_id:', b['id'])
"
```

Resultado esperado:
```
block_type_id: eda6e257-2deb-4b38-a3b6-c194d1a961ea
```

---

## Passo 2 — Obter o ID do block schema (versão mais recente)

```bash
curl -sL -X POST "$API/block_schemas/filter" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{"block_schemas": {"block_type_id": {"any_": ["eda6e257-2deb-4b38-a3b6-c194d1a961ea"]}}}' | python3 -c "
import json, sys
schemas = json.load(sys.stdin)
# o primeiro é o mais recente
s = schemas[0]
print('block_schema_id:', s['id'], '| version:', s['version'])
"
```

Resultado esperado:
```
block_schema_id: 358ee644-d961-4c42-a98b-c3670e73fd00 | version: 3.7.6
```

---

## Passo 3 — Criar o block document (Custom Webhook)

O bloco define a URL, headers e o template do body. O `json_data` usa `{{subject}}` e `{{body}}` como placeholders — esses valores são preenchidos dinamicamente pela automação em cada notificação.

Mapeamento de variáveis:
| Placeholder no bloco | Valor enviado pela automação |
|---|---|
| `{{subject}}` | `{{ deployment.id }}` (Prefect event template) |
| `{{body}}` | `{{ flow_run.id }}` (Prefect event template) |

> `flow_run_name` é omitido pois é opcional no endpoint (usado apenas para logging).

```bash
curl -sL -X POST "$API/block_documents/" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "flow-failed-prod",
    "block_type_id": "eda6e257-2deb-4b38-a3b6-c194d1a961ea",
    "block_schema_id": "358ee644-d961-4c42-a98b-c3670e73fd00",
    "data": {
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
    },
    "is_anonymous": false
  }' | python3 -c "import json,sys; d=json.load(sys.stdin); print('block_document_id:', d['id'])"
```

Resultado esperado:
```
block_document_id: <uuid>
```

---

## Passo 4 — Criar a automação

Usando o `block_document_id` obtido no passo anterior:

```bash
curl -sL -X POST "$API/automations" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "flow-failed-webhook",
    "description": "Chama o backend ao detectar falha no work pool basedosdados.",
    "enabled": true,
    "trigger": {
      "type": "event",
      "match": {"prefect.resource.id": "prefect.flow-run.*"},
      "match_related": {
        "prefect.resource.name": "basedosdados",
        "prefect.resource.role": "work-pool"
      },
      "expect": ["prefect.flow-run.Failed"],
      "for_each": ["prefect.resource.id"],
      "posture": "Reactive",
      "threshold": 1,
      "within": 0
    },
    "actions": [
      {
        "type": "send-notification",
        "block_document_id": "<block_document_id do passo 3>",
        "subject": "{{ deployment.id }}",
        "body": "{{ flow_run.id }}"
      }
    ]
  }' | python3 -c "import json,sys; d=json.load(sys.stdin); print('automation_id:', d['id'])"
```

---

## Listar automações existentes

```bash
curl -sL -X POST "$API/automations/filter" \
  -H "Authorization: Bearer $KEY" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -c "
import json, sys
for a in json.load(sys.stdin):
    print(a['name'], '|', a['id'], '| enabled:', a['enabled'])
"
```

---

## Deletar uma automação

```bash
curl -sL -X DELETE "$API/automations/<automation_id>" \
  -H "Authorization: Bearer $KEY"
```

---

## O que foi criado (2026-07-06)

### Staging (temporário — para validação)

| Recurso | Nome | ID |
|---|---|---|
| Block document | `flow-failed-staging` | `650dcff7-8f1e-4b56-8282-178e927a1ac5` |
| Automação | `flow-failed-staging-webhook` | `7cde884e-da5b-4386-bf58-8d80e8d25ee0` |

URL do block: `https://staging.backend.basedosdados.org/admin-tools/flow-failed/`

> **Remover após validação** — desativar ou deletar a automação de staging antes de criar a de produção.

### Produção (a criar após validação em staging)

Repetir os passos 3 e 4 com:
- `"name": "flow-failed-prod"`
- `"url": "https://backend.basedosdados.org/admin-tools/flow-failed/"`
