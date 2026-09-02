# Troubleshooting: Schedule não desativa no Prefect 1.0

Este documento descreve os principais motivos pelos quais um schedule pode não ser desativado corretamente no Prefect 1.x e como investigar cada caso.

Plataforma: Prefect 1.0 (Server/Core)  
Sistema: Prefect — workflow orchestration platform

---

## 📌 Visão Geral

No Prefect 1.x, a desativação de um schedule envolve múltiplos componentes:

- API (GraphQL)
- Banco de dados
- Scheduler (loop de geração de ticks)
- Agents (execução de flow runs)

Por conta dessa arquitetura, existem cenários onde o schedule pode aparentar continuar ativo mesmo após a mutation retornar `success`.

---

# 1️⃣ Race condition com o Scheduler (Causa mais comum)

## O que acontece

Quando a mutation de desativação é executada enquanto o scheduler ainda está processando ticks:

1. A API marca `is_schedule_active = false`
2. O scheduler ainda cria um último batch de `flow_runs`
3. O flow parece continuar executando
4. A segunda tentativa de desativação funciona

## Sintomas

- Mutation retorna `success`
- Novas runs continuam sendo criadas logo após
- Segunda execução da mesma mutation resolve

## Mitigação

- Executar a mutation duas vezes de forma idempotente
- Cancelar `flow_runs` em estado `Scheduled`
- Confirmar via query que `is_schedule_active` está `false`

---

# 2️⃣ Runs já estavam agendadas

Desativar o schedule **não cancela runs já criadas**.

Se existirem `flow_runs` com estado `Scheduled`, elas continuarão executando.

## Como verificar

Consultar:

- flow_run com state = "Scheduled"

## Mitigação

Cancelar explicitamente as runs:

- update_flow_run
- state → "Cancelled"

---

# 3️⃣ Múltiplos Flows com mesmo nome

No Prefect 1.x:

- Nome NÃO é identificador único
- UUID é a identidade real

Pode existir:

- Mesmo nome
- IDs diferentes
- Schedules diferentes

Você pode estar desativando apenas um dos IDs.

## Mitigação

- Buscar todos os flows com aquele nome
- Iterar sobre todos os UUIDs
- Desativar cada um explicitamente

---

# 4️⃣ FlowGroup com schedule ativo

No Prefect 1.0, o schedule está vinculado ao FlowGroup.

Mesmo que o Flow esteja inativo, um FlowGroup pode conter schedule persistente.

## Mitigação

- Verificar `flow_group`
- Remover `schedule` do FlowGroup se necessário

---

# 5️⃣ Flow foi re-registrado

Se alguém rodar novamente:

```python
flow.register(...)