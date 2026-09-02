Perfeito — segue a **descrição exata de PR no padrão GitHub**, pronta para copiar e colar 👇

---

# ✨ Nova funcionalidade: Disable Unhealthy Flow Schedules

## 📌 Contexto

Este PR adiciona o comando `disable_unhealthy_flow_schedules`, responsável por **desativar automaticamente schedules de flows não saudáveis no Prefect**, com base no histórico recente de execuções.

A solução foi construída com foco em **segurança**, **testabilidade** e **evitar falsos positivos**, permitindo também a execução em **modo dry-run** para validação sem efeitos colaterais.

---

## 🚀 O que este PR faz

- Identifica flows com execuções recentes problemáticas
    
- Avalia a saúde do flow a partir das **duas últimas execuções concluídas**
    
- Desativa automaticamente o schedule de flows considerados não saudáveis
    
- Permite execução em modo `--dry-run`, apenas logando os flows afetados
    

---

## 🧠 Regra de negócio (validação de flows)

A decisão de desativar um flow está encapsulada em `FlowDisable.validate()` e segue as regras abaixo:

### ✅ Um flow será considerado **não saudável** se **qualquer uma** das condições for verdadeira:

#### 1️⃣ Falha de task crítica após a criação do flow

- A última execução contém uma task cujo nome está configurado em `Constants.TASKS_NAME_DISABLE` (ex: `run_dbt`)
    
- A execução ocorreu **após a criação do flow**
    
- O estado da execução é `Failed`
    
- A mensagem de erro da task **não está** na lista de mensagens ignoradas (`STATE_MESSAGE_IGNORE`)
    

> Essa regra evita desativar flows por falhas conhecidas ou transitórias.

---

#### 2️⃣ Falhas consecutivas após a criação do flow

- As **duas últimas execuções** do flow estão no estado `Failed`
    
- Pelo menos uma dessas execuções ocorreu após a criação do flow
    

> Essa regra cobre cenários de instabilidade contínua do flow, mesmo sem envolvimento direto de uma task crítica.

---

## 🗂️ Organização de pastas

A lógica do comando foi organizada por feature, isolando responsabilidades e facilitando manutenção e testes:

```
management/
└── commands/
    ├── disable_unhealthy_flow_schedules.py
    └── _disable_unhealthy_flow_schedules/
        ├── __init__.py
        ├── constants.py        # Constantes e queries GraphQL
        ├── datetime_utils.py   # Helpers de datas
        ├── models.py           # Domínio (FlowDisable, FlowRun, TaskRun)
        └── service.py          # Integrações externas e orquestração
```

---

## 🔁 Fluxo de execução

```text
disable_unhealthy_flow_schedules.py (Command Django)
        │
        ▼
FlowService.disable_unhealthy_flow_schedules(dry_run)
        │
        ▼
Consulta GraphQL (flows com falhas recentes)
        │
        ▼
FlowDisable.get_runs()
        │
        ▼
FlowDisable.validate()
        │
        ▼
Desativa schedule (ou apenas loga, se dry-run)
```

---

## 🧪 Dry-run

O comando aceita o argumento `--dry-run`, que:

- **Não altera nenhum flow**
    
- Loga todos os flows avaliados
    
- Indica quais seriam desativados caso o comando fosse executado normalmente
    

---

## ✅ Benefícios desta implementação

- 🧩 Código **modular e testável**
    
- 🛡️ Regras de negócio **centralizadas e explícitas**
    
- 🎯 Menos falsos positivos (mensagens ignoradas configuráveis)
    
- 🔁 Detecção de falhas consecutivas
    
- 🧪 Execução segura via `--dry-run`
    

---

## 📝 Observações

- As regras de validação são facilmente ajustáveis via `constants.py`
    
- A arquitetura facilita a inclusão de novas regras de saúde no futuro
    

---

Se quiser, posso:

- Adaptar o texto para o **padrão da sua empresa**
    
- Criar uma seção de **How to test**
    
- Sugerir **checklist de revisão** pro PR 👌

Claro 👍  
Segue a **substituição completa da parte 4️⃣ do `models.py`** na descrição do PR, já **documentando a nova versão do `FlowDisable`** e alinhada com a lógica atual.

Você pode copiar e colar direto no PR.

---

## 4️⃣ `models.py` — Domínio e regra de negócio

O arquivo `models.py` contém as **classes de domínio** responsáveis por representar e avaliar a saúde dos flows:

- `Task`
    
- `TaskRun`
    
- `FlowRun`
    
- `FlowDisable`
    

A classe **`FlowDisable`** encapsula toda a **regra de negócio** para decidir se um flow deve ter seu schedule desativado.

---

### 🧠 `FlowDisable`

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

### ✅ Regra de validação — `validate()`

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

### 📋 Critérios de desativação

O flow será considerado **não saudável** se **qualquer uma** das condições abaixo for atendida:

#### 1️⃣ Falha de task crítica após a criação do flow

- A última execução contém uma task cujo nome está em `Constants.TASKS_NAME_DISABLE` (ex: `run_dbt`)
    
- A execução ocorreu **após a criação do flow**
    
- O estado da execução é `Failed`
    
- A mensagem de erro da task **não está** na lista de mensagens ignoradas (`STATE_MESSAGE_IGNORE`)
    

👉 Essa regra evita desativar flows por falhas conhecidas ou irrelevantes.

---

#### 2️⃣ Falhas consecutivas após a criação do flow

- As **duas últimas execuções** do flow estão no estado `Failed`
    
- Pelo menos uma dessas execuções ocorreu **após a criação do flow**
    

👉 Essa regra cobre cenários de instabilidade contínua do flow, mesmo sem uma task específica como causa.

---

### 🎯 Resultado

- `True` → o flow deve ter seu schedule desativado
    
- `False` → o flow permanece ativo
    

Toda a lógica fica **centralizada no domínio**, tornando o comportamento:

- Mais previsível
    
- Fácil de testar
    
- Simples de evoluir com novas regras no futuro
    

---

Se quiser, posso:

- Ajustar o texto para ficar **ainda mais sucinto**
    
- Criar um **exemplo prático** de flow que seria desativado
    
- Sugerir **test cases** para essa regra 👌

Boa — obrigado por corrigir 👍  
Segue então a **versão atualizada e correta da parte 5️⃣ `service.py`**, **substituindo a descrição anterior** e refletindo exatamente as **novas mudanças** (sem dry-run, sem logs intermediários).

Pronto para colar no PR 👇

---

### 5️⃣ `service.py` — Orquestração e desativação automática

O arquivo `service.py` é responsável por **orquestrar todo o processo de desativação**, integrando:

- Consulta de flows com falhas recentes
    
- Avaliação da saúde via domínio (`FlowDisable`)
    
- Desativação automática de schedules
    
- Notificação consolidada no Discord
    

A lógica principal está no método  
`disable_unhealthy_flow_schedules()`.

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

### 🧠 O que esse método faz

#### 1️⃣ Busca flows com falhas recentes

- Recupera via GraphQL os flows que apresentaram falhas na última semana
    
- Esses flows entram no conjunto inicial de análise
    

---

#### 2️⃣ Constrói os objetos de domínio

- Cada flow retornado é instanciado como `FlowDisable`
    
- As execuções recentes são carregadas e normalizadas em objetos de domínio (`FlowRun`, `TaskRun`)
    

---

#### 3️⃣ Avalia a saúde dos flows

- Cada flow executa `FlowDisable.validate()`
    
- Apenas flows considerados **não saudáveis** são selecionados para desativação
    

---

#### 4️⃣ Desativa schedules automaticamente

- Se houver flows não saudáveis:
    
    - O schedule de cada flow é desativado via `set_flow_schedule`
        
- A operação é **direta e automática**, sem modo dry-run nesta versão
    

---

#### 5️⃣ Envia notificação consolidada no Discord

Após a desativação:

- Uma única mensagem é enviada ao Discord contendo:
    
    - 🚨 **Flows em alerta** (todos os analisados)
        
    - ⛔ **Flows desativados**
        
- O grupo configurado em `Constants.DISCORD_ROLE_DADOS` é automaticamente mencionado
    

---

### 🆕 Mudanças em relação à versão anterior

- ❌ Remoção do modo `dry-run`
    
- ❌ Remoção de logs intermediários
    
- ✅ Foco em **ação direta + notificação**
    
- 🔔 Comunicação centralizada via Discord
    
- 🧩 Orquestração mais simples e objetiva
    

---

### 🎯 Benefícios da nova abordagem

- ⚡ **Resposta mais rápida** a flows instáveis
    
- 📣 **Visibilidade imediata** para o time via Discord
    
- 🧠 Regras de negócio continuam isoladas no domínio (`FlowDisable`)
    
- 🧼 Código mais enxuto e fácil de manter
    

---

Se quiser, posso:

- Ajustar o texto para um tom mais **executivo** ou mais **técnico**
    
- Criar uma seção **Riscos / Considerações**
    
- Sugerir um **rollback strategy** caso algum flow seja desativado indevidamente

Perfeito — segue uma **nova seção para o PR**, documentando a **task periódica**, no mesmo padrão das partes anteriores.

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

#### 🧠 O que essa task faz

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

---

### 🛡️ Considerações de segurança

- A task apenas **orquestra a execução** do comando
    
- Toda a regra de negócio permanece centralizada no domínio (`FlowDisable`)
    
- A execução recorrente garante **resposta rápida** a flows instáveis, reduzindo impacto operacional
    

---

### 🎯 Benefícios

- ⏱️ Monitoramento contínuo, sem ação manual
    
- ⚙️ Reutilização do comando Django (CLI + automação)
    
- 📣 Integração automática com alertas no Discord
    
- 🧩 Arquitetura consistente e fácil de evoluir
    

---

Se quiser, posso:

- Adicionar essa task ao **diagrama geral do fluxo**
    
- Criar uma seção **Observabilidade / métricas**
    
- Sugerir **feature flags** para pausar a task rapidamente em produção
Perfeito! Vou criar um **índice automático** baseado na estrutura do seu `.md`, usando links de ancoragem compatíveis com Markdown. Ficaria assim:

---

# Índice

1. [Nova funcionalidade: Disable Unhealthy Flow Schedules](#nova-funcionalidade-disable-unhealthy-flow-schedules)
    
2. [Contexto](#contexto)
    
3. [Organização de pastas](#organiza%C3%A7%C3%A3o-de-pastas)
    
4. [Organização de pastas e fluxo](#organiza%C3%A7%C3%A3o-de-pastas-e-fluxo)
    
5. [Diagrama visual do fluxo de execução](#diagrama-visual-do-fluxo-de-execu%C3%A7%C3%A3o)
    
6. [Notas visuais](#notas-visuais)
    
7. [Benefícios desta primeira versão](#benef%C3%ADcios-desta-primeira-vers%C3%A3o)
    
8. [Estrutura do código](#estrutura-do-c%C3%B3digo)
    
    1. [Command Django](#1-command-django)
        
    2. [`constants.py`](#2-constantspy)
        
    3. [`datetime_utils.py`](#3-datetime_utilspy)
        
    4. [`models.py` — Domínio e regra de negócio](#4-modelspy--dom%C3%ADnio-e-regra-de-neg%C3%B3cio)
        
        - [`FlowDisable`](#flowdisable)
            
        - [Regra de validação — `validate()`](#regra-de-valida%C3%A7%C3%A3o---validate)
            
        - [Critérios de desativação](#crit%C3%A9rios-de-desativa%C3%A7%C3%A3o)
            
        - [Resultado](#resultado)
            
        - [Observações](#observa%C3%A7%C3%B5es)
            
    5. [`service.py` — Orquestração e desativação automática](#5-servicepy--orquestra%C3%A7%C3%A3o-e-desativa%C3%A7%C3%A3o-autom%C3%A1tica)
        
        - [Fluxo principal — `disable_unhealthy_flow_schedules`](#fluxo-principal---disable_unhealthy_flow_schedules)
            
    6. [Task periódica — execução automática](#6-task-peri%C3%B3dica---execu%C3%A7%C3%A3o-autom%C3%A1tica)
        
        - [`disable_unhealthy_flow_schedules` (Periodic Task)](#disable_unhealthy_flow_schedules-periodic-task)
            
        - [Fluxo de execução](#fluxo-de-execu%C3%A7%C3%A3o)
            

---

Se você quiser, posso já **inserir esse índice no seu `.md` original**, com links funcionando de verdade, de modo que cada item leve direto para o título correspondente.

Quer que eu faça isso?