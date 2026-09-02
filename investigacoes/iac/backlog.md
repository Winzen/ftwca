# Backlog — Melhorias e Ideias

Registro de ideias, melhorias e iniciativas identificadas mas ainda não priorizadas.

---

## Pipelines

### Validação de tabelas antes do `table-approve`

**Contexto:**
Hoje o `table-approve` é disparado no PR sem garantia de que as tabelas ou dados que serão testados já existem no BigQuery. Quando a tabela ainda não foi criada ou o dado ainda não foi carregado, o workflow falha de forma pouco clara.

**Ideia:**
Adicionar um step de validação no CI do PR — antes do `table-approve` rodar — que confirme se as tabelas dos modelos dbt que serão testados existem no BigQuery (ou pelo menos se o dataset existe). Se não existirem, o step falha com uma mensagem explicativa antes de chegar no `table-approve`.

**Possível implementação:**
- Step no GitHub Actions que lista as tabelas referenciadas no PR (via `dbt ls` ou parsing do `manifest.json`)
- Consulta à API do BigQuery confirmando existência de cada tabela
- Falha antecipada com output claro: `Tabela basedosdados.br_ibge_inpc.mes não encontrada — rode o flow antes de abrir o PR`

**Impacto:** Reduz falhas confusas no `table-approve` e deixa claro o que precisa ser feito antes do merge.

---

## manual-equipe-dados

### Migrar documentações do Vault para o manual

Documentos do Vault que ainda não estão no [`manual-equipe-dados`](https://github.com/basedosdados/manual-equipe-dados) e deveriam estar:

**PR aberto:** [manual-equipe-dados#4](https://github.com/basedosdados/manual-equipe-dados/pull/4)

| Arquivo no Vault                                                          | Destino no manual                                                    | Status |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------- | ------ |
| `Prefect 3/Referência/Gestão de Schedules — Guia da Equipe.md`            | `docs/infraestrutura/prefect/referencia/gestao-de-schedules.md`      | ✅ |
| `Prefect 3/Referência/Gestão de Schedules — Operação e Monitoramento.md`  | `docs/infraestrutura/prefect/runbooks/gestao-de-schedules.md`        | ✅ |
| `Prefect 3/Referência/Override de Recursos por Flow.md`                   | `docs/infraestrutura/prefect/referencia/override-recursos.md`        | ✅ |
| `Prefect 3/Tutoriais/Tutorial — Atualizando dependências em pipelines.md` | `docs/pipelines/como-fazer/atualizar-dependencias.md`                | ✅ |
| `Prefect 3/Tutoriais/Tutorial — Automação Flow Failed Webhook.md`         | `docs/infraestrutura/prefect/como-fazer/flow-failed-webhook.md`      | ✅ |
| `Kubernetes/Comandos K8s.md`                                              | `docs/infraestrutura/kubernetes/referencia/comandos.md`              | ✅ |
| `Kubernetes/Helm/Comandos Helm.md`                                        | `docs/infraestrutura/kubernetes/referencia/helm.md`                  | ✅ |
| `Kubernetes/Criar Pod Debug.md`                                           | `docs/infraestrutura/kubernetes/como-fazer/pod-debug.md`             | ⏳ (notas brutas, pendente) |
| `Kubernetes/Loki/Runbook Loki PVC.md`                                     | `docs/infraestrutura/observabilidade/runbook/loki-pvc.md`            | ⏳ (contém JSON bruto, pendente) |

**Não migrar:** Sessões, snapshots de situação, retrospectivas, Backlog, Formulário, CONTEXTO_CLAUDE.

---

## MCP

### Oficializar repositório `basedosdados/mcp`

**Contexto:**
O repositório existe mas tem toda a lógica em um único `server.py` de ~2700 linhas com 40+ tools. O objetivo é reorganizar para facilitar contribuição interna da equipe.

**Estrutura proposta:**
```
src/databasis_mcp/
  __init__.py
  server.py       # instância FastMCP + run
  auth.py         # _get_credentials, _get_token
  gql.py          # _gql, _mut, helpers GraphQL
  tools/
    metadata.py   # search_datasets, list_datasets, get_dataset, discover_ids, lookup_id
    write.py      # create_update_*, reorder_*, upload_columns, delete_*
    bigquery.py   # query_bigquery, preview_table, audit_partition_metadata
    prefect.py    # list_flow_runs, get_flow_run_logs, trigger_dbt_model
```

**Demais itens:**
- `pyproject.toml`: adicionar entry point `[project.scripts] databasis-mcp = "databasis_mcp.server:main"` para rodar com `uv run databasis-mcp`
- CI: GitHub Actions com `ruff` para lint
- `CONTRIBUTING.md`: setup com `uv`, como adicionar uma tool, como testar localmente

---

## Backend + MCP

### Migrar domínio `prefect.basedosdados.org` para o Prefect 3

**Contexto:**
O Prefect 3 está acessível em `prefect3.basedosdados.org` e o domínio antigo `prefect.basedosdados.org` ainda aponta para o servidor do Prefect 0, que continua rodando com 5 deployments ativos (`prefect-apollo`, `prefect-graphql`, `prefect-hasura`, `prefect-towel`, `prefect-ui`) — apenas os agents estão em 0 réplicas.

**Opções:**
1. **Desligar o Prefect 0 e redirecionar o domínio** — escalar os deployments do Prefect 0 para 0, atualizar o ingress `prefect-ingress` no namespace `prefect` para apontar para o `prefect-server` no namespace `prefect3`, e remover o ingress `prefect3-ingress`.
2. **Só desligar o Prefect 0** — escalar os deployments para 0 e remover o ingress antigo, mantendo `prefect3.basedosdados.org` como domínio definitivo.

**Atenção:** Verificar se algum serviço externo ainda referencia `prefect.basedosdados.org` antes de desligar ou redirecionar.

---

### Conectar Prefect 3 ao Prometheus e Grafana

**Contexto:**
O stack de observabilidade (Prometheus + Grafana + Loki) já está no cluster no namespace `observability`. O Prometheus usa `serviceMonitorNamespaceSelector: {}` — ou seja, já monitora todos os namespaces, desde que o ServiceMonitor tenha o label `release: prometheus`. O Grafana já tem dashboards para backend, frontend, metabase e pipelines adicionados como ConfigMaps.

O problema: o Prefect Server não expõe métricas Prometheus por padrão. Testado — `/metrics` e `/api/metrics` retornam 404/HTML. O Helm chart tem `telemetryEnabled: true`, mas isso é telemetria interna do Prefect, não scraping via Prometheus.

**Opções:**

1. **Métricas nativas do Prefect Server** (preferida se suportada pela versão atual)
   Prefect 3 adicionou suporte experimental a métricas Prometheus via variável de ambiente. Verificar se a versão instalada suporta e habilitar no `values.yaml` do Helm chart.

2. **Exporter da comunidade**
   Usar um exporter que consulta a API do Prefect e expõe as métricas no formato Prometheus. Requer deploy de um pod extra no namespace `prefect3` + ServiceMonitor apontando para ele.

3. **Dashboard via Loki** (sem setup adicional)
   Os logs dos workers e do backend já chegam ao Loki. É possível criar painéis no Grafana usando LogQL sobre esses logs para mostrar flow runs, falhas e desativações — sem precisar de métricas Prometheus. Mais simples, mas menos granular.

**O que falta fazer:**
- Verificar a versão exata do Prefect Server instalada e checar se suporta métricas nativas
- Escolher abordagem e implementar: ServiceMonitor no namespace `prefect3` + ConfigMap de dashboard no namespace `observability`

---

### Subir Jaeger no cluster (observabilidade de traces)

**Branch:** `feat/opentelemetry_jaeger` no repositório backend (1 commit à frente de main)

**O que já existe:**
A branch configura o stack completo para dev local via `compose.yaml`:
- Django instrumentado com OpenTelemetry (Django, Redis, requests, psycopg, logging)
- OTel Collector como intermediário (recebe traces via OTLP/gRPC na porta 4317)
- Jaeger all-in-one para visualização (UI na porta 16686)

O Django exporta traces para o OTel Collector via:
```
OTEL_SERVICE_NAME=django-api
OTEL_EXPORTER_OTLP_ENDPOINT=otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

**O que falta para o cluster:**
- Deploy do Jaeger e do OTel Collector no cluster (namespace `observability`, ao lado do Loki/Prometheus)
- Sealed secrets ou ConfigMap com as variáveis `OTEL_*` para os pods de staging e prod
- Ingress para a UI do Jaeger (ou acesso via port-forward)
- Merge da branch no backend e atualização do deployment

**Impacto:** Traces distribuídos em staging e prod — visibilidade de latência por endpoint, queries lentas no banco e chamadas externas (BigQuery, Prefect 3, Elasticsearch).

---

### Fluxo de promoção de dados entre ambientes (dev → staging → prod)

**Contexto:**
Atualmente o MCP e outros serviços conseguem escrever diretamente em qualquer ambiente (dev, staging, prod) passando o parâmetro `env`. Não há um caminho de promoção obrigatório — uma mudança feita via MCP pode ir direto para prod sem passar por validação em outros ambientes.

**Objetivo:**
Estabelecer um fluxo controlado onde mudanças feitas por serviços externos (MCP, integrações) sigam o caminho dev → staging → prod, com a promoção entre ambientes sendo uma ação explícita e rastreável.

**Direções possíveis:**

1. **Restrição de acesso por ambiente no backend**
   Tokens de API com escopo de ambiente — um token de dev não consegue escrever em prod. A credencial de prod ficaria restrita a operações manuais ou pipelines autorizados.

2. **Endpoint de promoção**
   Um endpoint no backend (ex: `POST /admin-tools/promote/`) que copia entidades de staging para prod de forma controlada. A promoção seria a única forma de algo chegar em prod via automação.

3. **Ferramenta de promoção no MCP**
   Tool `promote_to_prod(dataset_slug, env="staging")` que chama o endpoint acima — tornando a promoção explícita e auditável dentro do fluxo do agente.

**Impacto:** Reduz risco de mudanças acidentais em prod e cria um histórico claro de o que foi promovido, quando e por quem.

---

## Observabilidade de recursos

### Monitorar consumo de memória por flow (Prefect 3)

**Contexto:**
Hoje o limite de memória dos job pods é definido manualmente via `job_variables` no `flows.py`, sem visibilidade de quanto cada flow realmente consome. Descobrimos os limites necessários só quando o pod OOMKilla em produção (caso `br_me_cnpj__estabelecimentos`, `br_anatel_telefonia_movel__*`).

**Objetivo:**
Criar uma forma de acompanhar o pico de memória de cada flow para embasar melhor os limites padrão e os overrides por flow.

**Direções possíveis:**
1. **Métricas do Kubernetes** — coletar `container_memory_working_set_bytes` dos job pods via Prometheus e exibir no Grafana por `deployment_name`. Requer label do flow no pod ou mapeamento por nome do job.
2. **Logs do worker** — o worker já loga o nome do flow run; adicionar coleta de uso de memória no encerramento do pod via `kubectl top` ou metrics-server.
3. **Prefect + Prometheus** — se habilitarmos as métricas nativas do Prefect 3 (ver item "Conectar Prefect 3 ao Prometheus"), os flow runs já viriam com labels de deployment.

**Impacto:** Permite definir `memory_limit` com base em dados reais em vez de tentativa e erro em produção.

---

### Executar etapas isoladas de um flow (ex: só o dbt run)

**Contexto:**
Hoje para testar uma etapa específica de um flow — como o `dbt run` do `br_me_cnpj__estabelecimentos` — é necessário rodar o flow inteiro, passando pelo download de 10 arquivos zip (~3GB cada) antes de chegar na etapa desejada. Isso torna o ciclo de debug lento e caro em recursos.

**Objetivo:**
Criar uma forma de disparar etapas isoladas de um flow sem executar as anteriores, assumindo que os dados já estão disponíveis (ex: no GCS ou no `/tmp/output`).

**Direções possíveis:**
1. **Parâmetro `start_from`** — adicionar um parâmetro ao flow que pula tasks anteriores e começa a partir de uma etapa específica (ex: `start_from="dbt"`).
2. **Flows auxiliares de debug** — criar flows separados e leves que apenas executam o `dbt run/test` para um dado `dataset_id` e `table_id`, sem dependência de download.
3. **Task isolada via script local** — documentar como invocar a task `run_dbt` diretamente via Python no ambiente local, sem passar pelo Prefect.

**Impacto:** Reduz o tempo de debug de horas para minutos em flows com etapas de download pesadas.

---

## Backend

### Refatorar `update_table_metadata_task` para event-driven

**Contexto:**
A task roda de segunda a sexta às 06:00 e faz N chamadas ao BigQuery em série (uma por tabela), sendo lenta e rodando mesmo quando nenhum dado mudou. O erro imediato (`BadRequest: Requester Pays`) foi corrigido no PR #1042, mas a arquitetura ainda é problemática.

**Ideia:**
Trocar o cron diário por event-driven: ao fim de cada flow do Prefect 3, o próprio flow chama um endpoint da API do backend passando `dataset_id`, `table_id` e os metadados atualizados (`number_rows`, `number_columns`, `uncompressed_file_size`). O backend persiste diretamente no model `Table`, sem precisar consultar o BigQuery.

**Possível implementação:**
1. Endpoint no backend — `POST /api/v1/tables/<pk>/metadata/` que recebe os campos e salva no model `Table`
2. Utilitário nos pipelines — função chamada ao fim de cada flow, provavelmente num módulo compartilhado (`pipelines/utils/metadata.py`)
3. Manter a task periódica existente como fallback — mas restrita a tabelas não atualizadas nos últimos X dias (muito mais barata)

**Arquivo afetado:** `backend/backend/apps/api/v1/tasks.py`, função `update_table_metadata_task` (linha 33)

**Impacto:** Metadados atualizados em tempo real após cada flow, reduz carga desnecessária no BigQuery.

---

### Revisar e atualizar testes do backend

**Contexto:**
Os testes estão desatualizados e parte deles quebrada. O estado atual por app:

| App | Arquivo | Situação |
|---|---|---|
| `api/v1` | `test_models.py`, `test_column_coverage.py`, `test_table_coverage.py` | 446 linhas — os mais completos, mas podem estar desatualizados com mudanças recentes |
| `account` | `tests.py` | 97 linhas — testa criação de conta, signals, confirmação |
| `account_payment` | `tests.py` | 92 linhas — existente mas sem revisão recente |
| `admin_data_tools` | `tests.py` | **Placeholder vazio** — sem nenhum teste; contém `FlowFailedWebhookView`, `SyncDeploymentsView`, `DisabledFlowSchedule` — toda a lógica de gestão de schedules do Prefect 3 |
| `account_auth` | `tests.py` | Placeholder vazio |
| `user_notifications` | `tests.py` | Placeholder vazio — app criada esse semestre |
| `core` | `tests.py` | Placeholder vazio |

**Prioridade:**
O app `admin_data_tools` é o mais crítico — tem toda a lógica nova construída esse semestre e zero cobertura de testes. Um bom ponto de entrada seria testar `FlowFailedWebhookView` (critérios de desativação por falhas consecutivas e dbt) e `SyncDeploymentsView`.

---

### Pod `api-development` quebrado — volume referencia `CHATBOT_SA` removido

**Contexto:**
O pod `api-development` no namespace `website` está preso em `ContainerCreating` há 12 dias com o erro:
```
MountVolume.SetUp failed for volume "gcp-sa": references non-existent secret key: CHATBOT_SA
```

O volume `gcp-sa` no deployment referencia duas keys do secret `api-development-secrets`:
- `GCP_SA` → `gcp-sa.json` ✅
- `CHATBOT_SA` → `chatbot-sa.json` ❌ (removida no commit `9fb0fcd` — "remove chatbot secrets from Django secrets")

**Fix:**
Remover o item `CHATBOT_SA` da spec do volume no deployment `api-development`. O deployment é gerenciado no repositório do backend (não no IAC).

---
