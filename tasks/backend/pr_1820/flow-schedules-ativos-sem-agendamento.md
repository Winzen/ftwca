# Levantamento: Flow Schedules "ativos" no backend sem agendamento real no Prefect 3

**Data**: 2026-08-22

## Método

1. `psql` direto em produção (`kubectl exec` no pod `api-prod`), tabela `admin_data_tools_disabledflowschedule`:
   ```sql
   SELECT flow_name, deployment_id FROM admin_data_tools_disabledflowschedule
   WHERE is_schedule_active = true ORDER BY flow_name;
   ```
   → **174 flows** marcados como ativos no backend.

2. Pra cada `deployment_id`, consulta direta ao Prefect 3 (`POST /deployments/filter`, batches de 100), olhando os campos `paused` e `schedules[].active` de cada deployment.

Lista completa por categoria em `flow_schedules_ativos_sem_agendamento_2026-08-22.csv` (mesma pasta).

## Resultado

| Categoria | Quantidade |
|---|---|
| **Pausado no Prefect** (`paused: true`), mesmo o backend achando que está ativo | **150** |
| Sem nenhuma schedule ativa (não pausado, mas 0 schedules com `active: true`) | 22 |
| Deployment não encontrado no Prefect (`deployment_id` órfão) | 1 |
| Genuinely ativo e agendado | 1 |

**52 datasets distintos** afetados pela categoria "pausado no Prefect" — não é um caso isolado.

### Os 150 "pausado no Prefect"

Esse é o achado principal: o backend (`DisabledFlowSchedule.is_schedule_active=True`) diz que o flow deveria estar rodando, mas o Prefect tem o deployment com `paused: true` — ou seja, **nenhuma dessas 150 tabelas está sendo atualizada automaticamente no momento**, apesar de aparecerem como "ativas" no admin.

**Causa raiz confirmada**: [run 32449984877](https://github.com/basedosdados/pipelines/actions/runs/32449984877/job/96676553380) do workflow `cd-prefect3 (production)` em `pipelines`, disparado pelo PR #1820 (`br_sfb_sicar`), em 2026-08-21T05:17. O job registra o catálogo inteiro de flows a cada push (não só os alterados pela PR). No meio do lote, `br_sfb_sicar_flow` falhou com `409 Conflict`:

```
Response: {'detail': 'Error creating deployment: <ValidationError: "{\'MALLOC_ARENA_MAX\': \'2\', \'MALLOC_TRIM_THRESHOLD_\': \'131072\'} is not of type \'array\'">'}
```

O script continua registrando os demais flows depois desse erro (179 flows registrados com sucesso no total), mas o **job inteiro termina com exit code 1** por causa dessa falha isolada — e o step seguinte, **"Sync deployments with backend"** (que chama `/admin-tools/sync-deployments/` e reativa cada deployment conforme o `is_schedule_active` do backend), **ficou marcado como pulado** e nunca rodou.

Confirmação numérica: dos 150 flows pausados, **137 (91%) foram exatamente os flows registrados nesse run** — batendo com a hipótese. Os outros 13 provavelmente vêm de drift acumulado de antes (esse mecanismo já existia e pode ter falhado silenciosamente outras vezes).

### Os 22 "sem schedule ativa"

Bem menos preocupante — a maioria são flows que plausivelmente não deveriam ter cron mesmo:
- `br_tse_eleicoes__*` (17 flows) — eleições, sem periodicidade regular fora de período eleitoral
- `hello_prefect3` / `hello-prefect3` / `hello-prefect3-prod` — flows de teste
- `bd-template-executa-dbt-model` — template

Vale confirmar cada um, mas não parecem indicar o mesmo problema sistêmico dos 150.

### O 1 órfão

`br_bndes_operacoes_contratadas__operacoes_contratadas_forma_indireta_automatica` (deployment `160c42d4-9805-4d87-8e1c-c85b8a47769a`) — o registro no backend aponta pra um `deployment_id` que não existe mais no Prefect. Provavelmente foi recriado com outro ID e o backend ficou com o antigo.

## Correção — executada em 2026-08-22

Rodado `POST /admin-tools/sync-deployments/` no backend de prod (`SyncDeploymentsView`), autenticado com `Authorization: Bearer $PREFECT3_API_KEY` executado de dentro do próprio pod `api-prod` (pra não expor o secret). Resultado:

```json
{"created": 1, "updated": 0, "activated": 173, "paused": 65, "errors": 0}
```

- **173 deployments reativados** (`paused: false`) — cobre os 150 identificados nesse levantamento mais outros que também tinham drift.
- **65 pausados** — deployments cujo `is_schedule_active` no backend é `False`; esperado, não é regressão.
- **0 erros**.

Reconferido direto no Prefect depois: dos 150 flows originalmente pausados, **0 continuam pausados**. Corrigido.

## Outros pontos

- Resolver o órfão (`br_bndes_operacoes_contratadas__operacoes_contratadas_forma_indireta_automatica`): o `sync-deployments` sozinho não corrige isso, porque o `deployment_id` antigo não existe mais no Prefect pra ser encontrado por `iter_deployments()`. Precisa achar o ID novo (mesmo `flow_name`) e atualizar o registro manualmente.
- Considerar se o workflow `cd-prefect3` deveria rodar o "Sync deployments with backend" mesmo quando o deploy de algum flow individual falha (`if: always()` em vez de depender do sucesso do step anterior) — evitaria que uma falha isolada like essa derrube a reativação de todo o resto do lote.
