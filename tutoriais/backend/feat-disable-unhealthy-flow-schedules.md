# Nova funcionalidade: Disable Unhealthy Flow Schedules

## Contexto

Este PR adiciona o comando `disable_unhealthy_flow_schedules`, responsável por **desativar automaticamente schedules de flows não saudáveis no Prefect**, com base no histórico recente de execuções.

Estamos adicionando o comando `disable_unhealthy_flow_schedules`, que:

* Identifica flows com execuções recentes problemáticas
* Avalia a saúde do flow a partir das **duas últimas execuções concluídas**  
* Desativa automaticamente o schedule de flows considerados não saudáveis
    
## Organização de pastas

A lógica do comando foi organizada em módulos por feature, dentro de `_disable_unhealthy_flow_schedules`, para separar responsabilidades e facilitar manutenção e testes:

## Organização de pastas e fluxo

```
management/
└── commands/
    ├── disable_unhealthy_flow_schedules.py   # comando Django mínimo
    └── _disable_unhealthy_flow_schedules/   # código auxiliar do comando
        ├── __init__.py
        ├── constants.py       # Constantes e queries GraphQL
        ├── datetime_utils.py  # Helpers para parsing de datas e cálculo de datas relativas
        ├── models.py          # Task, TaskRun, FlowRun, FlowDisable
        └── service.py         # MakeClient, FlowService e integrações externas
```

### Diagrama visual do fluxo de execução

```text
disable_unhealthy_flow_schedules.py (Command Django)
        │
        │  (chama)
        ▼
FlowService.disable_unhealthy_flow_schedules(dry_run)
        │
        │  (usa)
        ▼
   ┌─────────────┐
   │  FlowService │
   └─────────────┘
        │
        │  (consulta GraphQL)
        ▼
   MakeClient.query.execute(...)
        │
        │  (retorna dados)
        ▼
   FlowDisable.get_runs()
        │
        │  (instancia FlowRun e TaskRun)
        ▼
   FlowDisable.validate()
        │
        │  (decide se flow deve ser desativado)
        ▼
   FlowService.set_flow_schedule()  (se não dry-run)
        │
        ▼
   ✅ Flow desativado ou apenas logado
```

### Notas visuais

* `➡️` indica **chamada de função / fluxo de execução**
* `FlowService` é o núcleo que **orquestra a lógica**
* `FlowDisable` encapsula **regra de negócio / validação**
* `MakeClient` é responsável por **comunicação externa com GraphQL**

---

## Benefícios desta primeira versão

* **Modular e testável**: cada parte (models, service, utils, constants) isolada
* **Parsing de datas confiável**: comparações precisas com datetime
* **Organização clara por feature**: facilita manutenção e expansão futura

---

## Estrutura do código

### 1️⃣ Command Django

```python
# -*- coding: utf-8 -*-

from django.core.management.base import BaseCommand

  

from ._disable_unhealthy_flow_schedules.service import FlowService

  
  

class Command(BaseCommand):

    help = "Disable unhealthy flow schedules"

  

    def handle(self, *args, **options):

        FlowService().disable_unhealthy_flow_schedules()
```
---

### 2️⃣ `constants.py`

```python
from enum import Enum

class Querys(Enum):
    FLOWS_FAILED_LAST_WEEK = "..."  # GraphQL
    LAST_COMPLETED_RUNS_TASKS = "..."

RUN_DBT_TASK_NAME = "run_dbt"
FLOW_SUCCESS_STATE = "Success"
FLOW_FAILED_STATE = "Failed"
```

---

### 3️⃣ `datetime_utils.py`

```python
from datetime import datetime, timedelta
from django.utils import timezone

ISO_FORMAT = "%Y-%m-%dT%H:%M:%SZ"

def parse_datetime(value: str) -> datetime:
    return datetime.strptime(value, ISO_FORMAT)

def one_week_ago() -> str:
    return (timezone.now() - timedelta(days=7)).strftime(ISO_FORMAT)
```

---

### 4️⃣ `models.py` — Domínio e regra de negócio

O arquivo `models.py` contém as **classes de domínio** responsáveis por representar e avaliar a saúde dos flows:

- `Task`
    
- `TaskRun`
    
- `FlowRun`
    
- `FlowDisable`
    

A classe **`FlowDisable`** encapsula toda a **regra de negócio** para decidir se um flow deve ter seu schedule desativado.

---

#### 🧠 `FlowDisable`

Responsável por:

- Buscar as execuções recentes do flow
    
- Avaliar se o flow está em um estado **não saudável**
    
- Centralizar a lógica de validação, isolada de integrações externas
    

```python
class FlowDisable:
    def __init__(self, id: str, created: str, service: "FlowService"):
        self.id = id
        self.created = parse_datetime(created)
        self.service = service
        self.runs = self.get_runs()

    def get_runs(self):
        response = self.service.last_completed_runs_tasks(self.id)
        return [FlowRun(**run) for run in response["flow_run"]]
```

- O flow é inicializado com seu `id`, data de criação e o `FlowService`
    
- As **últimas execuções concluídas** são buscadas via GraphQL e transformadas em objetos de domínio (`FlowRun`)
    

---

#### ✅ Regra de validação — `validate()`

O método `validate()` decide se o flow deve ser desativado analisando **as duas últimas execuções concluídas**.

```python
def validate(self) -> bool:
    last_run = self.runs[0]
    next_last = self.runs[1] if len(self.runs) == 2 else None

    failed = Constants.FLOW_FAILED_STATE.value

    dbt_failed_after_created = (
        last_run.task_runs.task.name in Constants.TASKS_NAME_DISABLE.value
        and last_run.start_time >= self.created
        and last_run.state == failed
        and last_run.task_runs.state_message not in Constants.STATE_MESSAGE_IGNORE.value
    )

    consecutive_failed_after_created = (
        next_last
        and last_run.state == failed
        and next_last.state == failed
        and max(last_run.start_time, next_last.start_time) >= self.created
    )

    return bool(dbt_failed_after_created or consecutive_failed_after_created)
```

---

#### 📋 Critérios de desativação

O flow será considerado **não saudável** se **qualquer uma** das condições abaixo for atendida:

##### 1️⃣ Falha de task crítica após a criação do flow

- A última execução contém uma task cujo nome está em `Constants.TASKS_NAME_DISABLE` (ex: `run_dbt`)
    
- A execução ocorreu **após a criação do flow**
    
- O estado da execução é `Failed`
    
- A mensagem de erro da task **não está** na lista de mensagens ignoradas (`STATE_MESSAGE_IGNORE`)
    

👉 Essa regra evita desativar flows por falhas conhecidas ou irrelevantes.

---

##### 2️⃣ Falhas consecutivas após a criação do flow

- As **duas últimas execuções** do flow estão no estado `Failed`
    
- Pelo menos uma dessas execuções ocorreu **após a criação do flow**
    

👉 Essa regra cobre cenários de instabilidade contínua do flow, mesmo sem uma task específica como causa.

---

#### 🎯 Resultado

- `True` → o flow deve ter seu schedule desativado
    
- `False` → o flow permanece ativo
    

Toda a lógica fica **centralizada no domínio**, tornando o comportamento:

- Mais previsível
    
- Fácil de testar
    
- Simples de evoluir com novas regras no futuro

#### 📝 Observações

- As regras de validação são facilmente ajustáveis via `constants.py`
    
- A arquitetura facilita a inclusão de novas regras de saúde no futuro
    


---

### 5️⃣ `service.py` — Orquestração e desativação automática

- `MakeClient` → inicializa GraphQL client
- `FlowService` → funções de integração, queries e desativação de flows
- Função principal: `disable_unhealthy_flow_schedules(dry_run=False)`

O `disable_unhealthy_flow_schedules` é responsável por **orquestrar todo o processo de desativação**, integrando:

- Consulta de flows com falhas recentes
    
- Avaliação da saúde via domínio (`FlowDisable`)
    
- Desativação automática de schedules
    
- Notificação consolidada no Discord
    
---

### 🔁 Fluxo principal — `disable_unhealthy_flow_schedules`

```python
def disable_unhealthy_flow_schedules(self) -> None:
        
    flows_data = self.flows_failed_last_week()

    flows = [FlowDisable(**flow, service=self) for flow in flows_data]

    flows_to_disable = [flow for flow in flows if flow.validate()]

    if flows_to_disable:

        for flow in flows_to_disable:
            self.set_flow_schedule(flow_id=flow.id, active=False)

        message_parts = [
            self.format_flows("🚨 Flows em alerta", flows),
            self.format_flows(
                f"⛔ Flows desativados <@&{Constants.DISCORD_ROLE_DADOS.value}>",
                flows_to_disable,
            ),
        ]

        send_discord_message("\n\n".join(message_parts))
```

---

#### 🧠 O que esse método faz

##### 1️⃣ Busca flows com falhas recentes

- Recupera via GraphQL os flows que apresentaram falhas na última semana
    
- Esses flows entram no conjunto inicial de análise
    

---

##### 2️⃣ Constrói os objetos de domínio

- Cada flow retornado é instanciado como `FlowDisable`
    
- As execuções recentes são carregadas e normalizadas em objetos de domínio (`FlowRun`, `TaskRun`)
    

---

##### 3️⃣ Avalia a saúde dos flows

- Cada flow executa `FlowDisable.validate()`
    
- Apenas flows considerados **não saudáveis** são selecionados para desativação
    

---

##### 4️⃣ Desativa schedules automaticamente

- Se houver flows não saudáveis:
    
    - O schedule de cada flow é desativado via `set_flow_schedule`
        
- A operação é **direta e automática**, sem modo dry-run nesta versão
    

---

##### 5️⃣ Envia notificação consolidada no Discord

Após a desativação:

- Uma única mensagem é enviada ao Discord contendo:
    
    - 🚨 **Flows em alerta** (todos os analisados)
        
    - ⛔ **Flows desativados**
        
- O grupo configurado em `Constants.DISCORD_ROLE_DADOS` é automaticamente mencionado
    
---

### 6️⃣ Task periódica — execução automática

Além do comando Django, foi criada uma **task periódica** responsável por executar automaticamente a desativação de flows não saudáveis em produção.

---

#### ⏱️ `disable_unhealthy_flow_schedules` (Periodic Task)

```python
@db_periodic_task(crontab(minute="*/20"))
@production_task
def disable_unhealthy_flow_schedules():
    """Disable unhealthy flow schedules"""
    call_command("disable_unhealthy_flow_schedules")
```

---
- Executa o comando Django `disable_unhealthy_flow_schedules`
    
- Roda **a cada 20 minutos**
    
- Atua apenas em ambiente de **produção**
    

---

#### 🔁 Fluxo de execução

```text
Periodic Task (a cada 20 minutos)
        │
        ▼
call_command("disable_unhealthy_flow_schedules")
        │
        ▼
FlowService.disable_unhealthy_flow_schedules()
        │
        ▼
Avaliação e desativação automática de flows não saudáveis
```




