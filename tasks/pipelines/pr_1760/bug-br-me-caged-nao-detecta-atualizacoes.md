# Bug — `br_me_caged__microdados_movimentacao` não detecta atualizações da fonte

**Data da investigação:** 2026-08-05
**Status:** Validado em produção (via branch) — datas corrigidas, os 3 flows rodaram e trouxeram junho/2026. **PR #1760 ainda aberto** — falta o merge para o cron agendado (que roda a partir de `main`) usar o código novo.

---

## Fechamento (2026-08-09)

Os 3 flows (`microdados_movimentacao`, `_fora_prazo`, `_excluida`) rodaram em produção e completaram com sucesso, já usando o código novo do poll (PR #1760 mergeado):

| Flow | Run | Horário (UTC) | Log-chave |
|---|---|---|---|
| `microdados_movimentacao` | `Sleek-Bombay` | 18:07–18:16 | `Table.Update atualizado para a cobertura 2026-06-01` |
| `microdados_movimentacao_excluida` | `Ergonomic-Coyote` | 19:48–19:54 | `Fonte publicou cobertura nova: 2026-06-01` → `Table.Update atualizado para a cobertura 2026-06-01` |
| `microdados_movimentacao_fora_prazo` | `Generic-Skua` | 20:05–20:10 | `Fonte publicou cobertura nova: 2026-06-01` → `Table.Update atualizado para a cobertura 2026-06-01` |

Verificação independente no BigQuery (fora do que o próprio flow logou) confirma junho/2026 presente e correto, com o crescimento de linhas batendo exatamente com a competência nova:

| Tabela | Linhas em 2026-06 | `numRows` antes → depois | `MAX(DATE(ano,mes,1))` |
|---|---:|---|---|
| `microdados_movimentacao` | 4.295.101 | 275.291.553 → 279.586.654 | 2026-06-01 |
| `microdados_movimentacao_fora_prazo` | 65.573 | 8.609.944 → 8.675.517 | 2026-06-01 |
| `microdados_movimentacao_excluida` | 8.719 | 607.497 → 616.216 | 2026-06-01 |

Sem duplicação (delta de linhas = linhas da competência nova, exatamente) e sem perda. `Table.Update.latest` das 3 tabelas agora reflete a cobertura real (`2026-06-01`), não mais um timestamp de execução — o comportamento que a migração para `poll.py` (register_source_coverage_task + check_source_is_ahead_of_table_task + sync_table_coverage_task) pretendia garantir.

Houve uma tentativa falha antes dessa (`Ergonomic-Crab`, 2026-08-07, `dbt run` falhou em `target=dev`), mas a retentativa em 08-09 completou normalmente.

## Metodologia de verificação

Checagens feitas para validar a correção, sem confiar só no que o próprio flow logou:

1. **Logs da run** (`list_flow_runs` + `get_flow_run_logs`) — confirmar as mensagens específicas do poll novo (`"Fonte publicou cobertura nova"`, `"Table.Update atualizado para a cobertura"`) em vez das mensagens do poll antigo (`"Não há novas atualizações na fonte original"`).
2. **Metadados do BigQuery via `bq show`** (API de metadados, não consome cota de query) — comparar `numRows`/`numBytes`/`lastModifiedTime` antes e depois da run, e confirmar que `lastModifiedTime` bate com o horário de término da run.
3. **Query direta no BigQuery** (`query_bigquery`, `COUNT(*) WHERE ano=... AND mes=...`) — confirmar que o número de linhas da competência nova é exatamente igual ao delta de `numRows` do passo 2 (sem duplicação, sem perda).
4. **Leitura direta do backend GraphQL** (`allUpdate`, `allCoverage`) — cross-check independente de `Table.Update.latest`, `RawDataSource.Update.latest` e `Coverage.datetimeRanges`, sem depender do BigQuery nem dos tools de escrita do MCP.
5. **Estado do PR/branch** (`gh pr view --json mergedAt`, `git diff origin/main origin/<branch>`) — antes de escrever qualquer atualização, confirmar se o código validado já está em `main` ou só na branch. Pegamos exatamente esse deslize aqui: a run usou o código novo mas o PR ainda estava aberto.

### Ideias de teste para prevenir a recorrência

Nenhuma foi implementada ainda — é uma decisão de escopo/custo que vale discutir antes de escrever:

- **Teste de regressão no nível do flow** (`_run_me_caged`): mockar as tasks de metadata e afirmar que o flow chama `register_source_coverage_task` + `check_source_is_ahead_of_table_task` + `sync_table_coverage_task` (o poll novo) — trava qualquer reversão silenciosa para o poll antigo. Não há precedente desse tipo de teste no repo (só há testes das funções puras em `register.py`/`poll.py`, não dos flows em si).
- **Checagem de auditoria repo-wide**: um script/teste que varre `datasets/*/flows.py` procurando uso de `poll_source_for_update_task` (poll antigo) e cruza com uma lista de flows já revisados — pega o mesmo defeito arquitetural em qualquer um dos ~25 outros flows que ainda usam o modelo antigo, não só o CAGED.
- **Checagem de qualidade de dados (dbt test ou script agendado)**: alertar quando `Table.Update.latest` de uma tabela pro-tier estiver suspeitosamente adiantado em relação ao fim real da `Coverage` — pega o sintoma (não a causa) em qualquer flow, incluindo os que nunca vão migrar para o poll novo.

---

## Status intermediário (2026-08-06, já superado)

- **PR aberto:** [basedosdados/pipelines#1760](https://github.com/basedosdados/pipelines/pull/1760) — migra os 3 flows (`microdados_movimentacao`, `_fora_prazo`, `_excluida`, que compartilham `_run_me_caged`) para o poll novo (`pipelines/utils/metadata/poll.py`), o mesmo modelo usado pelo CNES. Ainda não mergeado.
- **`Table.Update.latest` corrigido em produção nas 3 tabelas** (estava contaminado com o timestamp de execução `2026-07-07`, travando a detecção mesmo com a fonte já tendo publicado junho/2026 desde 29/07):

  | Tabela | Update ID | Valor antigo (contaminado) | Valor corrigido |
  |---|---|---|---|
  | `microdados_movimentacao` | `7ee5cdf7-aaec-4557-9d01-b8306d65bbe1` | `2026-07-07` | `2026-05-01` |
  | `microdados_movimentacao_fora_prazo` | `23dfb153-9a63-4f98-bdd6-abdf4cd33cab` | `2026-07-07` | `2026-05-01` |
  | `microdados_movimentacao_excluida` | `ba2ed94d-5d56-4454-a3d9-c0d5d66abb2c` | `2026-07-07` | `2026-05-01` |

  `2026-05-01` reflete a cobertura real confirmada via `Coverage` (tier pro, `isClosed=true`) nas 3 tabelas: `2025-11` a `2026-05`. Consultado direto no backend GraphQL (`allUpdate`/`allCoverage`), sem depender do BigQuery.

- **Confirmação de que a correção da `microdados_movimentacao` já funcionou**: o `RawDataSource.Update.latest` dessa tabela avançou sozinho de `2026-05-29` para `2026-06-01`, indicando que uma execução já detectou e processou a competência de junho depois da correção manual.
- **`microdados_movimentacao_fora_prazo` e `microdados_movimentacao_excluida` ainda não avançaram** (`RawDataSource.Update.latest` continua em `2026-05-29`) — precisam rodar em produção para confirmar que a correção de data + o código (uma vez mergeado) resolvem o mesmo travamento nelas.

### Dependência pendente

**Ainda precisamos rodar os 3 flows em produção** para validar a correção de ponta a ponta (`microdados_movimentacao_fora_prazo` e `microdados_movimentacao_excluida` especificamente, já que `microdados_movimentacao` já deu sinal de que funcionou). Isso ficou bloqueado hoje porque **a cota diária de query do BigQuery no projeto `basedosdados-dev` esgotou**, impedindo testar localmente/em dev antes de ir para produção. Retomar assim que a cota resetar ou diretamente em produção, dado que as datas já estão prontas.

---

## Sintoma

A última execução do flow (`talented-sponge`, 2026-08-04 20:00 UTC, run `019fce4e-6e97-79db-a7ef-17358438e163`) completou normalmente, mas logou:

```
Não há novas atualizações na fonte original
No updates for table microdados_movimentacao!
```

Só que a fonte original (FTP do Ministério do Trabalho) já tinha dados novos havia mais de uma semana. A suspeita inicial era de que o flow estivesse com algum problema de rede/autenticação — não é o caso.

---

## Causa raiz

`br_me_caged` roda o **"poll antigo"** de detecção de atualização (`pipelines/utils/metadata/register.py::poll_source_for_update`), que compara a data mais recente disponível na fonte contra `Table.Update.latest` — **não é a cobertura real dos dados**, é o timestamp de quando a tabela foi **materializada no BigQuery** (`last_modified_time`).

### Fluxo do código

```
pipelines/datasets/br_me_caged/flows.py          — define os 3 flows (_caged_flow → _run_me_caged)
pipelines/crawler/me_caged/tasks.py               — get_source_last_date (lê o FTP)
pipelines/crawler/me_caged/constants.py           — FTP_HOST, REMOTE_DIR
pipelines/utils/metadata/tasks.py                 — poll_source_for_update_task (wrapper @task)
pipelines/utils/metadata/register.py              — poll_source_for_update (lógica pura, "poll antigo")
pipelines/utils/metadata/policy.py                — should_update_raw_source (comparação)
pipelines/utils/metadata/poll.py                  — modelo NOVO de poll (só o CNES usa)
```

`_run_me_caged` (`flows.py:46-58`):
```python
source_last_date = get_source_last_date()          # FTP: dia 1 do mês-pasta mais recente

if not force_run:
    has_new_data = poll_source_for_update_task(
        dataset_id=dataset_id, table_id=table_id,
        source_max_date=source_last_date, env="prod", date_format="%Y-%m",
    )
    if not has_new_data:
        print(f"No updates for table {table_id}!")
        return
```

`poll_source_for_update` (`register.py:309-358`):
```python
client.upsert_raw_source_poll(dataset_id, table_id, latest=datetime.datetime.today(), url=raw_source_url)

if source_max_date is None:
    return False

api_latest = client.get_table_update_latest(dataset_id, table_id)          # register.py:352 ← lê Table.Update, não a cobertura
if not policy.should_update_raw_source(api_latest, source_max_date):       # register.py:353
    log("Não há novas atualizações na fonte original")                    # log observado
    return False

return True
```

`should_update_raw_source` (`policy.py:49-55`):
```python
def should_update_raw_source(api_latest, source_max):
    if source_max is None:
        return False
    return api_latest is None or source_max > api_latest
```

A data da fonte vem de `get_source_last_date` (`pipelines/crawler/me_caged/tasks.py:51-90`): lista o FTP, acha o ano-pasta mais recente, entra nele, acha o mês-pasta mais recente, e devolve `date(ano, mes, dia=1)`.

### Evidência ao vivo (2026-08-05)

FTP público (`ftp.mtps.gov.br/pdet/microdados/NOVO CAGED`):
```
2026/202606/CAGEDMOV202606.7z   (53MB, disponível desde 29/07/2026)
2026/202606/CAGEDFOR202606.7z
2026/202606/CAGEDEXC202606.7z
```

Backend GraphQL de produção:

| Campo | Valor |
|---|---|
| `Table.Update.latest` (usado na comparação) | `2026-07-07T21:42:01Z` |
| `RawDataSource.Update.latest` (o que aparece como "última atualização" na página do dataset) | `2026-05-29T11:00:37Z` |
| `RawDataSource.Poll.latest` | `2026-08-04T23:00:26Z` (bate com o horário do run) |
| Cobertura real da tabela (`Coverage`) | até `2026-05` |

### Por que falha

`source_max_date` = `2026-06-01` (pasta `202606`, dia fixado em 1)
`api_latest` = `2026-07-07` (materialização do BigQuery, não a competência dos dados)

```
2026-06-01 > 2026-07-07  →  False  →  "Não há novas atualizações na fonte original"
```

**Agravante:** como a comparação usa sempre o dia 1 do mês da fonte contra um timestamp de execução com dia 7, mesmo quando a fonte publicar a pasta de julho (`202607` → `2026-07-01`), a comparação continua falsa (`2026-07-01 > 2026-07-07` é `False`). O flow só vai voltar a detectar atualização quando a fonte publicar a pasta de **agosto** (`202608`) — dado o atraso típico de ~2 meses do CAGED, isso só deve ocorrer perto de setembro/2026. Um único timestamp de materialização "adiantado" (dia 7) travou a detecção por potencialmente 2+ meses, independente de quantas competências novas a fonte publicar nesse intervalo.

---

## Precedente — mesmo bug no `br_ms_cnes`

Esse é exatamente o defeito arquitetural já documentado no README do `br_ms_cnes`: o poll comparava a cobertura da fonte com `Table.Update`, que registra quando a materialização rodou, não a competência coberta pelos dados. A correção foi migrar para um modelo de poll novo (`pipelines/utils/metadata/poll.py`), que compara contra a cobertura real em vez do timestamp de materialização.

Segundo o próprio `poll.py:14`, **só o CNES foi migrado** — os demais ~25 flows do repositório (incluindo `br_me_caged`) continuam no poll antigo e sujeitos ao mesmo defeito.

---

## Achado secundário — falha silenciosa no crawler FTP

`get_source_last_date` (`pipelines/crawler/me_caged/tasks.py:64-90`):
```python
try:
    folder_items = ftp.nlst()
    ...
    return last_date.date()
except Exception as ErrorNlst:
    log(f"Unable to access CAGED subfolders due to {ErrorNlst}")
```
Qualquer exceção (timeout, mudança de estrutura do FTP, pastas vazias) é apenas logada, sem `raise` — a função cai para `return None` implícito. Isso não é o que aconteceu no run de 04/08 analisado (o log seria diferente: sem a linha "Não há novas atualizações", já que `source_max_date=None` cai direto em `return False` sem log), mas é um ponto real de fragilidade: se o FTP mudar de layout ou der timeout, o flow reporta "sem atualização" mascarando uma falha real, sem nunca quebrar a run.

Vale também registrar que `client._read_update_latest` (`pipelines/utils/metadata/client.py:258-270`) lê `nodes[0]` sem `orderBy` explícito — não há duplicidade hoje em `br_me_caged` (1 `Update` por Table/RawDataSource), mas é um padrão frágil caso isso mude.

---

## Recomendação

Migrar `br_me_caged` para o modelo de poll novo (`pipelines/utils/metadata/poll.py`), como foi feito no CNES, comparando contra a cobertura real em vez de `Table.Update.latest`.

Como mitigação imediata (antes da correção definitiva), rodar o flow com `force_run=True` para trazer a competência de junho/2026 que já está disponível na fonte.

### O que é o "poll novo"

`pipelines/utils/metadata/poll.py` resolve o bug corrigindo a natureza da comparação: em vez de comparar a fonte contra um **timestamp de execução** (`bq.last_modified`), mantém dois campos de metadado que são ambos **datas de cobertura**, e compara cobertura com cobertura. Três funções, um passo por registro de metadado:

**1. `register_source_coverage`** — registra até onde a fonte publicou
```python
client.upsert_raw_source_poll(dataset_id, table_id, latest=hoje)   # sempre grava "polei hoje"

most_recent_update = client.get_raw_source_update_latest(dataset_id, table_id)
if most_recent_update is None or source_max_date > most_recent_update:
    client.upsert_raw_source_update(dataset_id, table_id, latest=source_max_date)  # avança a cobertura da FONTE
    return True
return False
```

**2. `check_source_is_ahead_of_table`** — o gate: a fonte está na frente da tabela?
```python
source_last_update = client.get_raw_source_update_latest(dataset_id, table_id)  # cobertura da fonte
table_last_update = client.get_table_update_latest(dataset_id, table_id)        # cobertura da tabela (não mais o timestamp de materialização!)
return source_last_update > table_last_update
```

**3. `sync_table_coverage`** — o "commit": grava até onde a tabela materializou
```python
source_coverage = bq.read_max_date(dataset_id, table_id, coverage=coverage)  # lê a cobertura real no BigQuery
...
client.upsert_table_update(dataset_id, table_id, latest=source_coverage)     # grava a COBERTURA, não datetime.now()
```

A diferença que resolve o bug: no poll antigo, `Table.Update.latest` era gravado com `bq.last_modified()` — quando a tabela foi tocada. No poll novo, `sync_table_coverage` grava `Table.Update.latest` com `source_coverage` — a competência de dados mais recente que a tabela de fato contém, lida do BigQuery via `bq.read_max_date`. Como os dois lados da comparação (`RawDataSource.Update` e `Table.Update`) passam a ser datas de cobertura no mesmo domínio, `2026-06 > 2026-05` funciona como esperado — não fica mais sujeito a um timestamp de execução com dia arbitrário.

### Como o CNES usa isso (`pipelines/crawler/datasus/flows.py:52-75`)

```python
source_max_date = get_datasus_source_max_date(ftp_files)

register_source_coverage_task(dataset_id, table_id, source_max_date=source_max_date, env="prod", date_format="%Y-%m")

if not force_run and not check_source_is_ahead_of_table_task(dataset_id, table_id, env="prod"):
    print("Tabela CNES já cobre a fonte — encerrando")
    return
```
e, só depois de materializar (`flows.py:123-129`):
```python
if update_metadata:
    sync_table_coverage_task(dataset_id, table_id, coverage=PartBdpro(...))
```

### O que muda em `br_me_caged`

Substituir a chamada a `poll_source_for_update_task` em `flows.py:46-58` por esse mesmo par `register_source_coverage_task` + `check_source_is_ahead_of_table_task` antes do download, e adicionar `sync_table_coverage_task` depois da materialização.

Ver também: [[crawler-analise-estrutura]] — mapeamento geral da pasta `crawler/`, onde `me_caged` aparece listado como "não migrado" para Prefect 3 (checar se essa nota está desatualizada, já que `flows.py` já existe para os 3 flows do dataset).
