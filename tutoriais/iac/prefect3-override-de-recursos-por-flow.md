# Override de Recursos por Flow (CPU / Memória)
**Data:** 2026-06-18

---

## Contexto

O work pool `basedosdados` tinha `resources` hardcoded no template do job manifest:

```json
"resources": {
  "limits":   {"cpu": "2",    "memory": "4Gi"},
  "requests": {"cpu": "500m", "memory": "1Gi"}
}
```

Isso impedia sobrescrever recursos por deployment — qualquer valor em `job_variables` era ignorado pelo worker. Flows como `br_anatel_telefonia_movel__*` morriam com OOMKilled mesmo setando `memory_limit` no deployment via API.

---

## Mudança aplicada

Substituímos os valores hardcoded por variáveis Jinja com defaults, nos dois work pools (`basedosdados` e `basedosdados-dev`).

### Antes
```json
"resources": {
  "limits":   {"cpu": "2",    "memory": "4Gi"},
  "requests": {"cpu": "500m", "memory": "1Gi"}
}
```

### Depois
```json
"resources": {
  "limits":   {"cpu": "{{ cpu_limit }}", "memory": "{{ memory_limit }}"},
  "requests": {"cpu": "{{ cpu_request }}", "memory": "{{ memory_request }}"}
}
```

As variáveis foram adicionadas ao schema com os mesmos valores como defaults:

| Variável | Default |
|---|---|
| `memory_limit` | `4Gi` |
| `memory_request` | `1Gi` |
| `cpu_limit` | `2` |
| `cpu_request` | `500m` |

O comportamento para todos os flows existentes é **idêntico ao anterior** — quem não define `job_variables` usa os defaults.

---

## Comandos utilizados

```python
import json, urllib.request

API = "https://prefect3.basedosdados.org/api"
KEY = "<PREFECT_API_KEY>"
headers = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def request(method, path, data=None):
    req = urllib.request.Request(f"{API}{path}", headers=headers, method=method,
                                  data=json.dumps(data).encode() if data else None)
    with urllib.request.urlopen(req) as r:
        return json.load(r) if r.length != 0 else {}

for pool in ["basedosdados", "basedosdados-dev"]:
    d = request("GET", f"/work_pools/{pool}")
    tmpl = d["base_job_template"]

    containers = tmpl["job_configuration"]["job_manifest"]["spec"]["template"]["spec"]["containers"]
    containers[0]["resources"] = {
        "limits":   {"cpu": "{{ cpu_limit }}",   "memory": "{{ memory_limit }}"},
        "requests": {"cpu": "{{ cpu_request }}", "memory": "{{ memory_request }}"}
    }

    props = tmpl["variables"]["properties"]
    props["memory_limit"]   = {"type": "string", "title": "Memory Limit",   "default": "4Gi",  "description": "Maximum memory for the job pod (e.g. '8Gi')."}
    props["memory_request"] = {"type": "string", "title": "Memory Request", "default": "1Gi",  "description": "Memory requested for scheduling."}
    props["cpu_limit"]      = {"type": "string", "title": "CPU Limit",      "default": "2",    "description": "Maximum CPU for the job pod."}
    props["cpu_request"]    = {"type": "string", "title": "CPU Request",    "default": "500m", "description": "CPU requested for scheduling."}

    request("PATCH", f"/work_pools/{pool}", {"base_job_template": tmpl})
    print(f"✅ {pool} atualizado")
```

> **Atenção:** `| default('valor')` (sintaxe Jinja2) **não funciona** no template do worker — o Kubernetes recebe a string literal. O default deve estar apenas no schema (`properties`), não no template.

---

## Como definir no flows.py

Flows que precisam de mais memória que o padrão de 4Gi devem declarar `job_variables` no deploy:

```python
from prefect import flow
from prefect.runner.storage import GitRepository

source = GitRepository(url="https://github.com/basedosdados/pipelines.git")

if __name__ == "__main__":
    br_anatel_telefonia_movel__densidade_brasil.deploy(
        name="br_anatel_telefonia_movel__densidade_brasil",
        work_pool_name="basedosdados",
        job_variables={
            "memory_limit": "8Gi",
            "memory_request": "2Gi",
        },
        ...
    )
```

Apenas `memory_limit` e `memory_request` precisam ser declarados — `cpu_limit` e `cpu_request` ficam no default.

### Regra geral

| Situação | Ação |
|---|---|
| Flow normal (< 4Gi) | Não declarar `job_variables` — usa o default do pool |
| Flow pesado (> 4Gi) | Declarar `job_variables: {memory_limit: "8Gi"}` no `flows.py` |
| Flow muito pesado | Ajustar conforme necessidade — nós têm ~12.6Gi alocável cada |

---

## Verificar a configuração atual do work pool

```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados" \
  -H "Authorization: Bearer $PREFECT_API_KEY" | python3 -c "
import json, sys
d = json.load(sys.stdin)
containers = d['base_job_template']['job_configuration']['job_manifest']['spec']['template']['spec']['containers']
print(json.dumps(containers[0].get('resources'), indent=2))
"
```

---

## Flows com OOM identificados

Flows que precisam de `job_variables` no pipelines para não OOMKillar:

| Flow | Memory limit sugerido |
|---|---|
| `br_anatel_telefonia_movel__densidade_brasil` | `8Gi` |
| `br_anatel_telefonia_movel__densidade_municipio` | `8Gi` |
| `br_anatel_telefonia_movel__densidade_uf` | `8Gi` |
| `br_anatel_telefonia_movel__microdados` | `8Gi` |
| `br_me_rais__*` | a confirmar |

Testado e confirmado: `densidade_brasil` com `memory_limit: "8Gi"` completou com sucesso.
