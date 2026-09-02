---
title: "Resumo para gestão — decisões de compare_against sem precedente no Prefect 0"
data: 2026-08-12
relacionado:
  - "[[Auditoria date_type — poll_source_for_update (Prefect 0 vs Prefect 3)]]"
  - "basedosdados/pipelines#1781"
  - "basedosdados/pipelines#1783"
---

# Resumo para gestão — decisões de `compare_against` sem precedente no Prefect 0

## Contexto em uma frase

O mecanismo que decide "a fonte tem dado novo?" em ~34 tabelas comparava a data errada desde a migração para o Prefect 3 — restauramos a comparação correta em todos os casos com precedente, e tomamos decisão nova (documentada) nos casos sem precedente.

## O bug, resumido

O `poll_source_for_update_task` (função que cada flow chama pra decidir se vale a pena rodar) sempre comparava a data da fonte contra `Table.Update.latest` — um timestamp de **quando o flow rodou por último**, não contra a cobertura real dos dados (`Coverage.DateTimeRange`). Isso já tinha causado bugs silenciosos em produção (`br_me_caged`, `br_ans_beneficiario`: paravam de detectar dados novos depois que esse timestamp "adiantava" demais).

Investigando, achamos que essa escolha (`data_max_date` vs `last_update_date`) **já existia no Prefect 0** e se perdeu de forma não intencional na migração. Reconstruímos a escolha como parâmetro novo, `compare_against`, com dois valores:
- **`"coverage"`** — compara contra a cobertura real dos dados (correto na maioria dos casos)
- **`"table_update"`** — compara contra o timestamp de execução (correto só quando a própria fonte publica por timestamp, não por competência/data)

PR: [basedosdados/pipelines#1783](https://github.com/basedosdados/pipelines/pull/1783)

## Onde a decisão foi mecânica vs onde exigiu julgamento

Auditamos as 32 chamadas do poll hoje em produção. Para a maioria (23 de 32), a decisão foi **restaurar** o que o Prefect 0 já fazia — não uma escolha nova, apenas recuperar um comportamento perdido. Duas categorias, porém, exigiram decisão própria:

### 1. Datasets sem histórico no Prefect 0 (9 flows)

Datasets criados depois da migração, ou que nunca tiveram esse tipo de controle antes — não havia comportamento anterior para restaurar. Decidimos caso a caso, pela natureza de cada fonte:

| Dataset | Decisão | Motivo |
|---|---|---|
| `br_bcb_taxa_selic` | `coverage` | A fonte (BCB) publica por data real da série diária |
| `br_me_siconfi` | `coverage` | A fonte publica por ano de competência |
| `br_mf_divida_ativa` | `coverage` | A fonte publica por trimestre de competência |
| `au_abs_cpi` | `coverage` | A fonte publica por mês de competência |
| `au_abs_labour_force` | `coverage` | A fonte publica por mês de competência |
| `us_bls_cpi` | `coverage` | A fonte publica por mês de competência |
| `us_bls_qcew` | `coverage` | A fonte publica por trimestre de competência |
| `world_cricsheet` | `coverage` | A fonte publica por data real de partida |
| `br_bndes_operacoes_contratadas` (2 tabelas) | `table_update` | **Exceção deliberada** — a fonte (CKAN) só informa quando o arquivo foi publicado, não a que período os dados se referem; comparar isso contra a cobertura anual da tabela misturaria duas medidas incompatíveis |

### 2. Correção sobre a própria auditoria (2 datasets, achados por code review)

Depois do PR principal, um comentário de revisão identificou que `br_me_cnpj/simples` usa um tipo de cobertura (`NonHistorical`) que **nunca grava** `Coverage.DateTimeRange` no Prefect 3 — só grava o timestamp de execução. A auditoria original tinha marcado esse caso como "restaurar coverage" (porque era isso que o Prefect 0 fazia), mas essa restauração, aplicada ao pé da letra, quebraria o poll: sem cobertura pra comparar, ele sempre reportaria "há dado novo", rodando o flow todo dia sem necessidade.

Achamos o mesmo padrão em `br_rf_cnpj` (tabelas `simples` e `dicionario`).

| Dataset | Tabela(s) | Decisão final | Por quê |
|---|---|---|---|
| `br_me_cnpj` | `simples` | `table_update` | Sem cobertura real disponível (tier `NonHistorical`); o timestamp de execução é, na prática, o mesmo valor que o Prefect 0 comparava (ambos vêm do metadado `last_modified` do BigQuery) |
| `br_rf_cnpj` | `simples`, `dicionario` | `table_update` | Mesmo caso |

Nos dois, as demais tabelas do dataset (que têm coluna de data real) continuam em `coverage`.

## Estado atual

- PR #1783 aberto, com todos os 32 pontos de comparação corrigidos e testados (133 testes automatizados passando).
- Testado ao vivo em produção (`br_me_caged`, sem risco de duplicar dados — confirmado que o modelo dbt se protege contra reprocessamento).
- Pendência conhecida, fora do escopo deste PR: alguns dos 32 flows podem já ter o `Table.Update.latest` "contaminado" (adiantado demais), do mesmo jeito que achamos no CAGED e no ANS — precisa de checagem individual depois do merge.
