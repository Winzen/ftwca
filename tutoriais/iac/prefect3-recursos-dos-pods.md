# Prefect 3 — Recursos dos Pods (CPU / Memória)

---

## 1. Dois tipos de pod, dois lugares para configurar

| Tipo | O que faz | Onde configurar |
|---|---|---|
| **Worker pod** | Processo que faz polling no Prefect server | `values.yaml` do Helm chart |
| **Job pod** | Executa o flow (criado a cada run) | `base_job_template` do work pool via API |

O que importa para OOM é sempre o **job pod**.

---

## 2. Limites atuais dos job pods (configurado em 2026-06-08)

| Pool | Requests | Limits |
|---|---|---|
| `basedosdados` (prod) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |
| `basedosdados-dev` (dev) | 500m CPU / 1Gi RAM | 2 CPU / 4Gi RAM |

**Por que limitar é melhor que deixar sem limite:**
- Sem limite, o kernel Linux mata processos de forma imprevisível quando o nó fica sob pressão de memória — qualquer pod pode ser vítima
- Com limite, o Kubernetes mata **só o pod que excedeu**, com erro claro (`OOMKilled`, exit code 137), sem afetar outros flows no mesmo nó

**Nós do cluster:** 3 × 4 CPU / ~12.6 GB alocável cada.

---

## 3. Ver limites atuais via API

```bash
source .env
curl -s "$PREFECT_API_URL/work_pools/basedosdados" \
  -H "Authorization: Bearer $PREFECT_API_KEY" | python3 -c "
import json, sys
d = json.load(sys.stdin)
containers = d['base_job_template']['job_configuration']['job_manifest']['spec']['template']['spec']['containers']
print(json.dumps(containers[0].get('resources', 'não definido'), indent=2))
"
```

---

## 4. Alterar o limite global do pool

```python
import json, urllib.request, os

api = os.environ['PREFECT_API_URL']
key = os.environ['PREFECT_API_KEY']

resources = {
    "requests": {"cpu": "500m", "memory": "1Gi"},
    "limits":   {"cpu": "2",    "memory": "4Gi"}
}

for pool in ["basedosdados", "basedosdados-dev"]:
    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        headers={'Authorization': f'Bearer {key}'})
    with urllib.request.urlopen(req) as r:
        d = json.load(r)

    tmpl = d['base_job_template']
    containers = tmpl['job_configuration']['job_manifest']['spec']['template']['spec']['containers']
    containers[0]['resources'] = resources

    req = urllib.request.Request(f'{api}/work_pools/{pool}',
        data=json.dumps({"base_job_template": tmpl}).encode(),
        method='PATCH',
        headers={'Authorization': f'Bearer {key}', 'Content-Type': 'application/json'})
    with urllib.request.urlopen(req) as r:
        r.read()
    print(f'✅ {pool} atualizado')
```

---

## 5. Override de recursos por flow específico

Quando um flow precisa de mais memória que o padrão do pool, usar `job_variables` no `flows.py` — sobrescreve só para aquele deployment, sem afetar os demais:

```python
_flow.deploy_schedules = [{"cron": "0 3 * * *", "timezone": "America/Sao_Paulo"}]
_flow.job_variables = {"memory_limit": "8Gi", "memory_request": "2Gi"}
```

O `deploy_flows.py` lê o `job_variables` e passa para `.deploy()` automaticamente. Os outros flows continuam com o padrão do pool.

> **Regra:** manter o pool em 4Gi como baseline e usar `job_variables` nos flows que precisam de mais. Não inflar o baseline global para todos.

**Pré-requisito (configurado em 2026-06-18):** o template do work pool precisa ter os recursos como variáveis Jinja — `{{ memory_limit }}`, `{{ memory_request }}`, `{{ cpu_limit }}`, `{{ cpu_request }}` — com os defaults no schema. Antes dessa mudança, `resources` era hardcoded e qualquer `job_variables` era ignorado silenciosamente. Ver doc [[Override de Recursos por Flow]] para os comandos completos.

> **Armadilha:** `{{ memory_limit | default('4Gi') }}` **não funciona** — o Kubernetes recebe a string literal. O default deve estar apenas no schema do work pool, não no template.

---

## 6. Diagnosticar OOM

Quando um pod morre por OOM, verificar antes que seja deletado:

```bash
# Listar pods recentes no namespace
kubectl get pods -n prefect-worker-basedosdados --sort-by='.metadata.creationTimestamp' | tail -10

# Ver motivo da morte
kubectl describe pod <pod-name> -n prefect-worker-basedosdados | grep -A5 "Last State\|OOMKilled\|Exit Code\|Reason"
```

Se aparecer `OOMKilled` e `Exit Code: 137` → o pod excedeu o memory limit. Aumentar o limite via seção 4 ou 5.

---

## 7. Eviction por pressão de nó vs OOMKilled

São dois erros distintos com causas diferentes:

| Erro | Mensagem nos logs | Causa | Solução |
|---|---|---|---|
| **OOMKilled** | `Container was killed due to out-of-memory` / Exit Code 137 | O pod excedeu o seu próprio `memory limit` | Aumentar `memory_limit` via `job_variables` |
| **Eviction** | `Pod was evicted: The node was low on resource: memory` | O nó ficou sem memória por concorrência de pods | Distribuir schedules para não rodar ao mesmo tempo |

Eviction acontece quando vários flows pesados sobem no mesmo nó ao mesmo tempo. O kubelet escolhe uma vítima (o pod com maior consumo em relação ao request) e a mata antes que o nó inteiro trave.

**Exemplo real (2026-06-18):** 3 flows `br_anatel_telefonia_movel` com 8Gi cada rodaram simultâneos. Um nó ficou com apenas 37MB livres e o kubelet evictou um dos pods.

**Como evitar:** distribuir os schedules de flows pesados por dias diferentes:

```python
# Cada flow roda em um dia da semana — nunca competem pelo mesmo nó
br_anatel_telefonia_movel__microdados          = _flow(..., cron="30 16 * * 1")  # segunda
br_anatel_telefonia_movel__densidade_municipio = _flow(..., cron="30 16 * * 2")  # terça
br_anatel_telefonia_movel__densidade_uf        = _flow(..., cron="30 16 * * 3")  # quarta
br_anatel_telefonia_movel__densidade_brasil    = _flow(..., cron="30 16 * * 4")  # quinta
```

---

## 8. Flows com override de recursos configurado

| Flow | `memory_limit` | PR |
|---|---|---|
| `br_anatel_telefonia_movel__microdados` | `8Gi` | pipelines#1581 |
| `br_anatel_telefonia_movel__densidade_municipio` | `8Gi` | pipelines#1581 |
| `br_anatel_telefonia_movel__densidade_uf` | `8Gi` | pipelines#1581 |
| `br_anatel_telefonia_movel__densidade_brasil` | `8Gi` | pipelines#1581 |
