# Auditoria `date_type` — `poll_source_for_update` (Prefect 0 vs Prefect 3)

**Data:** 2026-08-10
**Relacionado:** [basedosdados/pipelines#1781](https://github.com/basedosdados/pipelines/issues/1781), [[Bug — br_me_caged não detecta atualizações do CAGED]]

---

## Contexto

A issue [#1781](https://github.com/basedosdados/pipelines/issues/1781) pede, em resumo: "verificar se a `max_date` é a cobertura ou a data de atualização", "incluir parâmetro de cobertura ou `data_atualização` na `poll_source_for_update_task`", "comparar bananas com bananas e laranjas com laranjas". Isso descreve exatamente o mesmo defeito arquitetural já diagnosticado e corrigido em `br_me_caged` (PR #1760) e `br_ans_beneficiario` (PR #1779): o poll atual (`pipelines/utils/metadata/register.py::poll_source_for_update`) sempre compara a data da fonte contra `Table.Update.latest` (via `client.get_table_update_latest()`) — um campo que pode ser um timestamp de execução (`bq.last_modified`), não a cobertura real dos dados.

## O achado: o parâmetro já existiu

Criamos um worktree do repositório `pipelines` no commit imediatamente anterior à migração do CAGED para Prefect 3 (`1dff7cfc`, via `git worktree add /mnt/d/repositorios/bd/pipelines_no_migration 1dff7cfcd47e21c418c281b54e1fab6c2ebcec59`) para inspecionar o sistema de poll do Prefect 0.

A função antiga, `check_if_data_is_outdated` (`pipelines/utils/metadata/tasks.py:202-261` nesse worktree), **já tinha exatamente o parâmetro que a issue pede**:

```python
def check_if_data_is_outdated(
    dataset_id, table_id, data_source_max_date,
    date_type: str = "data_max_date",
    date_format: str = "%Y-%m-%d",
) -> bool:
    if date_type == "data_max_date":
        data_api = get_api_most_recent_date(...)   # Coverage.DateTimeRange — cobertura real
    if date_type == "last_update_date":
        data_api = get_api_last_update_date(...)   # allUpdate(table_Id).latest — Table.Update.latest
    ...
    if data_source_max_date > data_api:
        ...
```

`get_api_last_update_date` (`pipelines/utils/metadata/utils.py:752-787`) faz literalmente a mesma query GraphQL que o bug atual usa (`allUpdate(table_Id).latest`) — ou seja, o antigo `"last_update_date"` é exatamente o `Table.Update.latest` contaminado que já corrigimos duas vezes.

**Durante a migração para Prefect 3, esse parâmetro se perdeu.** `poll_source_for_update_task` não tem escolha nenhuma — sempre compara contra `Table.Update.latest`, o equivalente do antigo `"last_update_date"`. Isso não foi uma decisão consciente: é uma regressão silenciosa da migração, que empurrou todo mundo (inclusive quem antes usava corretamente `"data_max_date"`) para o comportamento que causa o bug.

## Comparação detalhada: fluxo completo antigo vs atual

Além do parâmetro em si, vale registrar o fluxo completo de leitura/escrita dos dois sistemas — inclui uma nuance que muda a interpretação do bug.

### Fluxo antigo (`check_if_data_is_outdated`, Prefect 0)

```
check_if_data_is_outdated(dataset_id, table_id, source_max_date, date_type):
  1. SEMPRE grava RawDataSource.Poll.latest = agora
  2. Lê o alvo de comparação conforme date_type:
     - "data_max_date" (default) → get_api_most_recent_date() → Coverage.DateTimeRange (cobertura REAL)
     - "last_update_date"        → get_api_last_update_date()  → Table.Update.latest (allUpdate(table_Id))
  3. Compara source_max_date > alvo
  4. Se True: JÁ GRAVA RawDataSource.Update.latest (antes de baixar/materializar!):
     - modo "last_update_date": grava source_max_date (a data real da fonte)
     - modo "data_max_date" (maioria): grava datetime.today() — "agora", não a data da fonte
  5. retorna True/False

Se True → baixa, sobe, roda dbt, e no fim:
  update_django_metadata(...):
    - atualiza Coverage.DateTimeRange com a cobertura real lida do BQ
    - atualiza Table.Update.latest = bq.last_modified() (timestamp de execução)
```

Evidência: `check_if_data_is_outdated` (`pipelines/utils/metadata/tasks.py:202-261`), `update_data_source_update_date` (`pipelines/utils/metadata/utils.py:789-834` — o `if date_type == "last_update_date": ... else: latest = datetime.datetime.today()` está nas linhas 811-816), `update_django_metadata` (`pipelines/utils/metadata/tasks.py:49-195`, a escrita de `Table.Update` via `update_date_from_bq_metadata` está nas linhas 171-195) — todos no worktree pré-migração.

### Fluxo atual (`poll_source_for_update_task`, Prefect 3)

```
poll_source_for_update_task(dataset_id, table_id, source_max_date):
  1. SEMPRE grava RawDataSource.Poll.latest = agora (igual ao antigo)
  2. Lê SEMPRE Table.Update.latest (client.get_table_update_latest()) — sem escolha
  3. Compara source_max_date > Table.Update.latest
  4. Não grava nada aqui (a escrita de RawDataSource.Update fica pra depois)

Se True → baixa, sobe, roda dbt, e no fim:
  register_table_materialization_task: Coverage.DateTimeRange (igual ao antigo) + Table.Update.latest = bq.last_modified() (idêntico ao antigo)
  commit_source_update_task: RawDataSource.Update.latest = source_max_date (a data real da fonte)
```

Evidência: `poll_source_for_update` (`pipelines/utils/metadata/register.py:309-358`), `register_table_materialization` (`register.py:255-304`), `commit_source_update` (`register.py:361-393`) — repositório atual.

### O insight

`Table.Update.latest` **sempre** foi `bq.last_modified()` (timestamp de execução) nos dois sistemas — isso nunca mudou entre Prefect 0 e Prefect 3. A diferença é que o sistema antigo, no modo padrão (`data_max_date`, usado por 19 dos ~23 datasets auditados), **nunca comparava contra esse campo** — comparava contra `Coverage.DateTimeRange`, um terceiro campo, sempre fiel à cobertura real, nunca contaminado por timestamp de execução.

A migração colapsou os dois modos num só, e esse único modo escolhido foi justamente o que lia `Table.Update.latest` — o campo que sempre teve natureza de timestamp de execução. Não é que a migração "quebrou" o `Table.Update`; é que a comparação passou a usar o campo errado, que sempre teve essa natureza, só que antes nunca era consultado no caminho padrão.

**Curiosidade lateral:** o `RawDataSource.Update.latest` do sistema antigo, no modo padrão, também não era totalmente confiável — gravava "hoje" (quando detectamos a novidade) em vez da data real da fonte. O `commit_source_update_task` atual corrige isso "por acidente", gravando `source_max_date` de verdade — mais preciso que o antigo, mesmo dentro do sistema com o bug.

**O `poll.py` novo (usado na correção do CAGED e do ANS) não faz o que o antigo fazia** — não volta a ler `Coverage.DateTimeRange` diretamente. Em vez disso, torna o próprio `Table.Update.latest` confiável, trocando sua escrita de `bq.last_modified()` para `bq.read_max_date()` (em `sync_table_coverage`). Mecanismo estruturalmente diferente do antigo, mesmo objetivo: comparar cobertura com cobertura.

## Auditoria completa

Levantamento de todos os usos de `check_if_data_is_outdated(...)` no worktree pré-migração (`grep -rn "check_if_data_is_outdated" pipelines/`), com o `date_type` extraído de cada chamada (parsing de parênteses balanceados — default `"data_max_date"` quando o parâmetro não aparece na chamada).

### Usavam `data_max_date` (cobertura real — o comportamento correto, perdido na migração)

| Dataset | Tabela(s) | Evidência (worktree pré-migração) |
|---|---|---|
| `br_anatel_banda_larga_fixa` | banda_larga_fixa | `crawler/anatel/banda_larga_fixa/flows.py:41` |
| `br_anatel_telefonia_movel` | telefonia_movel | `crawler/anatel/telefonia_movel/flows.py:47` |
| `br_cgu_cartao_pagamento` | — | `crawler/cgu/flows.py:103` |
| `br_cgu_servidores_executivo_federal` | — | `crawler/cgu/flows.py:151` |
| `br_cgu_licitacao_contrato` | — | `crawler/cgu/flows.py:192` |
| `br_cgu_beneficios_cidadao` | — | `crawler/cgu/flows.py:233` |
| `br_ms_sia` / `br_ms_sih` / `br_ms_sinan` | — | `crawler/datasus/flows.py:353` (CNES já migrado à parte para o poll novo) |
| `br_ibge_inpc` / `br_ibge_ipca` / `br_ibge_ipca15` | — | `crawler/ibge_inflacao/flows.py:46` (função `_run_ibge_inflacao` compartilhada pelos 3) |
| `br_me_cnpj` | empresas, socios, estabelecimentos, simples | `datasets/br_me_cnpj/flows.py:57,140,225,335` |
| `br_rf_cno` | — | `crawler/rf/flows.py:53` |
| `br_anp_precos_combustiveis` | — | `datasets/br_anp_precos_combustiveis/flows.py:49` |
| `br_ans_beneficiario` | informacao_consolidada | `datasets/br_ans_beneficiario/flows.py:55` — **já corrigido, PR #1779** |
| `br_bcb_agencia` | — | `datasets/br_bcb_agencia/flows.py:59` |
| `br_bcb_estban` | — | `datasets/br_bcb_estban/flows.py:52` |
| `br_denatran_frota` | — | `datasets/br_denatran_frota/flows.py:57` |
| `br_inmet_bdmep` | — | `datasets/br_inmet_bdmep/flows.py:48` |
| `br_me_caged` | microdados_movimentacao, _fora_prazo, _excluida | `datasets/br_me_caged/flows.py:63,187,300` — **já corrigido, PR #1760** |
| `br_rf_cafir` | — | `datasets/br_rf_cafir/flows.py:64` |
| `br_stf_corte_aberta` | — | `datasets/br_stf_corte_aberta/flows.py:53` |

19 datasets nessa situação — candidatos a sofrer o mesmo bug do CAGED/ANS (2 já corrigidos, 17 pendentes).

### Usavam `last_update_date` explicitamente (escolha deliberada — não é bug)

| Dataset | Evidência |
|---|---|
| `br_cvm_fi` | `crawler/cvm/flows.py:53-56` |
| `br_tse_eleicoes` | `crawler/tse_eleicoes/flows.py:66-69` |
| `br_cgu_emendas_parlamentares` | `datasets/br_cgu_emendas_parlamentares/flows.py:48-52` |
| `br_ibge_pnadc` | `datasets/br_ibge_pnadc/flows.py:54-58` |

Esses 4 já escolhiam comparar contra `Table.Update.latest` antes da migração — não sofreram regressão, presumindo que a escolha original fazia sentido para a natureza da fonte de cada um (vale uma checagem pontual se ainda faz sentido, mas não é o mesmo bug).

### Sem histórico Prefect 0 (datasets novos, ou sem gate de poll)

| Dataset | Situação |
|---|---|
| `br_bndes_operacoes_contratadas` | Não existia no repo pré-migração |
| `br_bcb_taxa_selic` | Existia, mas o flow Prefect 0 nunca teve gate de poll — sempre rodava e redumpava a série inteira |
| `br_me_siconfi` | Não existia |
| `br_mf_divida_ativa` | Não existia |
| `au_abs_cpi`, `au_abs_labour_force`, `us_bls_cpi`, `us_bls_qcew`, `world_cricsheet` | Onboarding recente, criados direto em Prefect 3 |

Esses 9 precisam de uma decisão nova (não uma restauração), baseada na natureza de cada fonte.

**À parte:** `br_bcb_sicor` (`crawler/bcb/flows.py:49`) nunca usou comparação de data — sempre foi por tamanho em bytes (`check_if_data_is_outdated_by_size`), um terceiro modelo que não se aplica a este `date_type`.

## Metodologia

1. `git log --oneline -- pipelines/datasets/br_me_caged` no repo atual para achar o commit exato de migração do CAGED (`65ba1716`).
2. `git worktree add /mnt/d/repositorios/bd/pipelines_no_migration 1dff7cfc` (pai desse commit) — cópia completa e navegável do código pré-migração, sem afetar a branch de trabalho atual.
3. `grep -rn "check_if_data_is_outdated" --include="*.py" pipelines/` no worktree antigo para listar todas as chamadas.
4. Script Python com parsing de parênteses balanceados (não regex de linha única, já que as chamadas são multi-linha) para extrair o argumento `date_type` de cada chamada, tratando a ausência do parâmetro como o default `"data_max_date"`.
5. Cruzamento manual dos casos com múltiplas ocorrências no mesmo arquivo (CGU, CNPJ, IBGE inflação) contra os nomes das funções/flows para mapear cada ocorrência ao dataset correto.

## Decisão: parâmetro restaurado

Escolhemos a primeira abordagem: restaurar a escolha no `poll_source_for_update` atual, em vez de migrar cada um dos 17 datasets pendentes para o modelo `poll.py` novo (CAGED/ANS).

**PR:** [basedosdados/pipelines#1783](https://github.com/basedosdados/pipelines/pull/1783) (branch `feat/poll_compare_against`, a partir de `origin/main`).

Implementação:
- `MetadataClient.get_coverage_max_date(dataset_id, table_id)` (`client.py`) — novo método de leitura, consulta `Coverage.DateTimeRange` (`allCoverage(table_Id)` com `datetimeRanges` aninhado) e devolve o maior `end` entre todas as faixas (free + pro). Mês/dia ausentes (cobertura anual) default para `1`, para bater com a convenção já usada no resto do código (`"%Y-%m"` sempre normaliza pra dia 1).
- `poll_source_for_update(..., compare_against="coverage")` (`register.py`) — parâmetro novo. `"coverage"` usa `get_coverage_max_date`; `"table_update"` usa `get_table_update_latest` (o comportamento antigo).
- `poll_source_for_update_task(..., compare_against="coverage")` (`tasks.py`) — mesmo parâmetro exposto no wrapper.
- Nomes escolhidos deliberadamente diferentes do Prefect 0 (`compare_against: "coverage"|"table_update"` em vez de `date_type: "data_max_date"|"last_update_date"`) — os nomes antigos eram ambíguos (`"last_update_date"` não deixava óbvio que lia `Table.Update`, não uma "data de atualização" no sentido comum).
- Testes: `test_client.py` (3 casos do método novo) e `test_register.py` (dois modos de comparação, incluindo um que reproduz o cenário exato do bug — `Table.Update` "adiantado" bloqueando, `Coverage` real corretamente detectando a novidade).

**Default trocado de `"table_update"` para `"coverage"` depois da auditoria completa** (ver seção seguinte) — inicialmente o default era `"table_update"` (preservava o comportamento de todo mundo enquanto nada tinha sido revisado). Uma vez que todos os 32 flows passaram a declarar `compare_against` explicitamente (#1784), não havia mais nenhum caller dependendo do valor implícito — só então trocamos o default para `"coverage"` (27 dos 32 flows usam esse valor), sem mudar o comportamento de nenhum flow existente. Corrigidos 2 testes de `register_source_poll` (função sem `compare_against` exposto, sem uso em nenhum flow hoje) que dependiam do default antigo via fixture `table_update_latest` — trocados para `coverage_max_date`.

## Aplicado nos flows

**PR:** [basedosdados/pipelines#1784](https://github.com/basedosdados/pipelines/pull/1784) (branch `feat/poll_compare_against_apply_flows`, empilhado sobre `feat/poll_compare_against` — só pode ser mergeado depois do #1783).

`compare_against` setado explicitamente em **todos** os 32 flows que ainda usam `poll_source_for_update_task` (script com parsing de bloco balanceado, não regex de linha única, já que as chamadas são multi-linha — mesma técnica usada na auditoria), em 2 commits:

**1º commit — os 19 com equivalente no Prefect 0** (regressão confirmada, usavam `data_max_date` por padrão):
- `"coverage"`: anatel×2, cgu×4, datasus/SIA+SIH+SINAN×2, ibge_inflacao×1 função compartilhada, me_cnpj×1 função compartilhada, rf, br_anp_precos_combustiveis, br_bcb_agencia, br_bcb_estban, br_denatran_frota, br_inmet_bdmep, br_rf_cafir, br_stf_corte_aberta
- `"table_update"` explícito nos 4 que já usavam `last_update_date` de propósito (cvm_fi, tse_eleicoes, br_cgu_emendas_parlamentares, br_ibge_pnadc) — sem mudança de comportamento, só deixa a escolha visível em vez de implícita no default.

**2º commit — os 9 sem histórico Prefect 0** (datasets novos, decisão caso a caso pela natureza da fonte, não por convenção):
- `"coverage"` (8): `br_bcb_taxa_selic` (max date da série diária do BCB), `br_me_siconfi` e `br_mf_divida_ativa` (max ano/trimestre disponível, poll não-gating nos dois mas comparação correta), `au_abs_cpi`, `au_abs_labour_force`, `us_bls_cpi`, `us_bls_qcew`, `world_cricsheet` (todos com max ano-mês/trimestre/data publicado pela fonte, poll gating)
- `"table_update"` (`br_bndes_operacoes_contratadas`, 2 flows) — exceção deliberada, não bug: `source_max_date` é o `last_modified` do recurso no CKAN (timestamp de publicação, dia/hora), enquanto a `Coverage` da tabela é anual (coluna `ano`) — comparar contra `Coverage` misturaria granularidades desconexas. O próprio código já documentava essa distinção antes desta mudança.

`br_me_caged` e `br_ans_beneficiario` também usavam `data_max_date` por padrão no Prefect 0 (mesma regressão dos 19 do 1º commit) — ficam de fora deste PR porque já foram corrigidos por outro caminho: migração completa para `poll.py` (#1760/#1779), que não usa mais `poll_source_for_update_task`.

**Pendência:** os datasets com `compare_against="coverage"` ainda podem ter `Table.Update.latest` contaminado (igual ao que achamos no CAGED/ANS) — esses PRs só trocam o alvo da comparação. Depois do merge, vale checar um por um se algum já está com `Table.Update.latest` "adiantado" o suficiente para precisar da mesma correção manual de metadados que fizemos no CAGED e no ANS.

## Tabela final — estado atual de cada flow

Levantada direto do código (`git grep -n "compare_against=" -- '*/flows.py'` na branch `feat/poll_compare_against_apply_flows`), não de memória — 32 pontos de chamada em 29 arquivos, cobrindo 34 datasets/tabelas (alguns arquivos servem mais de um dataset via função compartilhada).

### `compare_against="coverage"` (26 pontos de chamada)

| Dataset | Tabela(s) | Motivo | Arquivo:linha |
|---|---|---|---|
| `br_anatel_banda_larga_fixa` | — | Prefect 0 usava `data_max_date` | `crawler/anatel/banda_larga_fixa/flows.py:54` |
| `br_anatel_telefonia_movel` | — | Prefect 0 usava `data_max_date` | `crawler/anatel/telefonia_movel/flows.py:59` |
| `br_cgu_cartao_pagamento` | — | Prefect 0 usava `data_max_date` | `crawler/cgu/flows.py:130` |
| `br_cgu_servidores_executivo_federal` | — | Prefect 0 usava `data_max_date` | `crawler/cgu/flows.py:182` |
| `br_cgu_licitacao_contrato` | — | Prefect 0 usava `data_max_date` | `crawler/cgu/flows.py:227` |
| `br_cgu_beneficios_cidadao` | — | Prefect 0 usava `data_max_date` | `crawler/cgu/flows.py:272` |
| `br_ms_sia`, `br_ms_sih` | — | Prefect 0 usava `data_max_date` (função `_run_dbf_to_parquet` compartilhada) | `crawler/datasus/flows.py:170` |
| `br_ms_sinan` | — | Prefect 0 usava `data_max_date` | `crawler/datasus/flows.py:278` |
| `br_ibge_inpc`, `br_ibge_ipca`, `br_ibge_ipca15` | — | Prefect 0 usava `data_max_date` (função `_run_ibge_inflacao` compartilhada) | `crawler/ibge_inflacao/flows.py:54` |
| `br_me_cnpj` | empresas, socios, estabelecimentos, simples | Prefect 0 usava `data_max_date` (função `_run_me_cnpj` compartilhada) | `crawler/me_cnpj/flows.py:61` |
| `br_rf_cno` | — | Prefect 0 usava `data_max_date` | `crawler/rf/flows.py:53` |
| `br_anp_precos_combustiveis` | microdados | Prefect 0 usava `data_max_date` | `datasets/br_anp_precos_combustiveis/flows.py:57` |
| `br_bcb_agencia` | agencia | Prefect 0 usava `data_max_date` | `datasets/br_bcb_agencia/flows.py:66` |
| `br_bcb_estban` | — | Prefect 0 usava `data_max_date` | `datasets/br_bcb_estban/flows.py:62` |
| `br_denatran_frota` | — | Prefect 0 usava `data_max_date` | `datasets/br_denatran_frota/flows.py:67` |
| `br_inmet_bdmep` | microdados | Prefect 0 usava `data_max_date` | `datasets/br_inmet_bdmep/flows.py:55` |
| `br_rf_cafir` | imoveis_rurais | Prefect 0 usava `data_max_date` | `datasets/br_rf_cafir/flows.py:68` |
| `br_stf_corte_aberta` | decisoes | Prefect 0 usava `data_max_date` | `datasets/br_stf_corte_aberta/flows.py:57` |
| `br_bcb_taxa_selic` | taxa_selic | Sem histórico Prefect 0 — `source_max_date` é o max date real da série diária buscada na API do BCB | `datasets/br_bcb_taxa_selic/flows.py:96` |
| `br_me_siconfi` | siconfi | Sem histórico Prefect 0 — max ano disponível na fonte (poll não-gating, mas comparação correta) | `datasets/br_me_siconfi/flows.py:147` |
| `br_mf_divida_ativa` | — | Sem histórico Prefect 0 — max trimestre disponível na fonte (poll não-gating, mas comparação correta) | `datasets/br_mf_divida_ativa/flows.py:116` |
| `au_abs_cpi` | monthly | Sem histórico Prefect 0 — max ano-mês publicado pela fonte, poll gating | `datasets/au_abs_cpi/flows.py:109` |
| `au_abs_labour_force` | — | Sem histórico Prefect 0 — max ano-mês publicado pela fonte, poll gating | `datasets/au_abs_labour_force/flows.py:124` |
| `us_bls_cpi` | monthly | Sem histórico Prefect 0 — max ano-mês publicado pela fonte, poll gating | `datasets/us_bls_cpi/flows.py:111` |
| `us_bls_qcew` | naics | Sem histórico Prefect 0 — max trimestre publicado pela fonte, poll gating | `datasets/us_bls_qcew/flows.py:120` |
| `world_cricsheet` | cricsheet | Sem histórico Prefect 0 — max data de partida publicada pela fonte, poll gating | `datasets/world_cricsheet/flows.py:124` |

### `compare_against="table_update"` (6 pontos de chamada)

| Dataset | Tabela(s) | Motivo | Arquivo:linha |
|---|---|---|---|
| `br_cvm_fi` | — | Prefect 0 já usava `last_update_date` de propósito | `crawler/cvm/flows.py:54` |
| `br_tse_eleicoes` | — | Prefect 0 já usava `last_update_date` de propósito | `crawler/tse_eleicoes/flows.py:51` |
| `br_cgu_emendas_parlamentares` | — | Prefect 0 já usava `last_update_date` de propósito | `datasets/br_cgu_emendas_parlamentares/flows.py:56` |
| `br_ibge_pnadc` | — | Prefect 0 já usava `last_update_date` de propósito | `datasets/br_ibge_pnadc/flows.py:58` |
| `br_bndes_operacoes_contratadas` | operações (indiretas automáticas) | Exceção deliberada: `source_max_date` é `last_modified` do CKAN (timestamp), `Coverage` é anual — granularidades desconexas | `crawler/bndes/flows.py:75` |
| `br_bndes_operacoes_contratadas` | administração pública | Mesma exceção do flow acima | `crawler/bndes/flows.py:180` |

### Não usam `compare_against` — migrados para o `poll.py` novo

| Dataset | Tabela(s) | Situação | PR |
|---|---|---|---|
| `br_me_caged` | microdados_movimentacao, _fora_prazo, _excluida | Usava `data_max_date` no Prefect 0 (mesma regressão dos acima); corrigido via `register_source_coverage_task`/`check_source_is_ahead_of_table_task`/`sync_table_coverage_task` em vez de `poll_source_for_update_task` | #1760 |
| `br_ans_beneficiario` | informacao_consolidada | Idem | #1779 |
