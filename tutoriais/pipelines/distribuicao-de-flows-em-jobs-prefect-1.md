# Distribuição de Flows em Jobs no Prefect 1.0 com Kubernetes

## Visão Geral

No Prefect 1.0, quando utilizamos `KubernetesRun` como `run_config`, cada execução de um Flow (Flow Run) é transformada em um **Job do Kubernetes**, que por sua vez cria um **Pod** para executar o código.

A arquitetura segue o seguinte fluxo:

Prefect Server/Cloud  
        ↓  
Prefect Agent  
        ↓  
Kubernetes Job  
        ↓  
Kubernetes Pod  
        ↓  
Execução do Flow  

---

# 1. Conceitos Fundamentais

## Flow
Definição do pipeline (DAG de tasks).

## Flow Run
Uma execução específica de um Flow.  
Cada Flow Run possui um `flow_run_id` (UUID único).

## RunConfig (KubernetesRun)
Configuração que define onde e como o Flow será executado.

## Agent
Processo responsável por:
- Monitorar a fila de execuções
- Criar a infraestrutura necessária (Job no Kubernetes)

---

# 2. Como ocorre a distribuição dos Flows

## 2.1 Disparo do Flow

Quando um Flow é agendado ou executado manualmente:

- Um Flow Run é criado no backend (Server ou Cloud).
- O Flow Run entra em estado `Scheduled`.

---

## 2.2 Atuação do Agent

O Prefect Agent:

1. Monitora a fila de execução.
2. Identifica um Flow Run com `KubernetesRun`.
3. Cria um **Job** no Kubernetes.
4. O Job cria um **Pod**.
5. O Pod executa o comando:

   prefect execute flow-run <flow_run_id>

---

## 2.3 Execução no Pod

O Pod recebe variáveis de ambiente como:

- PREFECT__FLOW_RUN_ID
- PREFECT__API_URL
- PREFECT__API_KEY

O container então:
1. Conecta ao backend.
2. Busca a definição do Flow.
3. Executa as tasks.
4. Reporta status.
5. Finaliza.

---

# 3. Distribuição e Escalabilidade

## Modelo de Escala

Cada Flow Run = 1 Job = 1 Pod

Se 10 Flows forem disparados simultaneamente:
→ 10 Jobs serão criados  
→ 10 Pods serão executados  

A escalabilidade depende de:

- Capacidade do cluster Kubernetes
- Limites de CPU/memória definidos
- Políticas de autoscaling

---

# 4. Quem gerencia cada parte?

## Prefect Server / Cloud
Responsável por:
- Orquestração
- Estados do Flow
- Agendamento
- Armazenamento de metadados

## Prefect Agent
Responsável por:
- Observar a fila
- Criar Jobs no Kubernetes
- Intermediar execução

O Agent **não executa o Flow diretamente** quando usamos KubernetesRun.

## Kubernetes
Responsável por:
- Criar Jobs
- Criar Pods
- Gerenciar containers
- Reiniciar em caso de falha (se configurado)
- Gerenciar recursos (CPU/memória)

## DevOps / Plataforma
Normalmente gerencia:
- Cluster Kubernetes
- RBAC
- Namespaces
- Policies
- Autoscaling
- Observabilidade (logs, métricas)

---

# 5. Identificação do Pod que executou o Flow

O Prefect adiciona labels como:

flow_run_id=<uuid-completo>

Para localizar o Pod:

kubectl get pods -l flow_run_id=<flow_run_id>

Essa é a forma mais confiável de rastrear execuções.

---

# 6. Boas Práticas

- Definir requests/limits de CPU e memória.
- Configurar TTL para Jobs finalizados.
- Centralizar logs (ex: ELK, Loki).
- Usar namespaces dedicados para flows.
- Monitorar falhas de Agent.

---

# 7. Resumo Executivo

- O Prefect orquestra.
- O Agent cria a infraestrutura.
- O Kubernetes executa.
- Cada execução de Flow gera um Pod isolado.
- A distribuição é automática e baseada na fila monitorada pelo Agent.

Esse modelo garante isolamento, escalabilidade e execução distribuída.