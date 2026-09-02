---
title: Auditoria de lacunas de dados — priorização por risco de Table.Update contaminado
data: 2026-08-14
relacionado:
  - "[[Auditoria date_type — poll_source_for_update (Prefect 0 vs Prefect 3)]]"
  - "[[Bug — br_me_caged não detecta atualizações do CAGED]]"
  - "basedosdados/pipelines#1781"
  - "basedosdados/pipelines#1783"
---

# Auditoria de lacunas de dados — priorização por risco de `Table.Update` contaminado

## Por que isso é provável, não uma exceção rara

O bug corrigido no #1783 (`poll_source_for_update` comparando contra `Table.Update.latest` em vez de `Coverage.DateTimeRange`) não é um caso raro de borda — o mecanismo faz com que, uma vez disparado, ele tenda a **travar a detecção de vez**, não só ocasionalmente pular um mês.

`Table.Update.latest` é um timestamp de **relógio** (grava "agora" a cada materialização bem-sucedida, via `bq.last_modified()`). `source_max_date` é uma data de **competência** (ex.: `2026-06-01`, representando dados de junho). Como toda fonte tem defasagem de publicação (junho é publicado em julho/agosto, nunca no mesmo dia), "agora" é quase sempre cronologicamente **maior** que qualquer competência real que a fonte publique. Isso significa que, depois da **primeira** materialização bem-sucedida pós-migração pro Prefect 3 (que grava `Table.Update.latest = agora`), a comparação `source_max_date > table_update_latest` tende a ficar `False` **permanentemente** — até alguém rodar manualmente com `force_run=True`.

Isso já foi confirmado em produção duas vezes (`br_me_caged` #1760, `br_ans_beneficiario` #1779) — os dois só foram descobertos porque alguém percebeu "tem dado novo na fonte que a gente não tem". A pergunta em aberto: quantos dos outros ~24 flows que usavam o default `data_max_date`/coverage no Prefect 0 (e foram silenciosamente jogados pro modo `table_update` na migração, antes do #1783) estão na mesma situação, sem que ninguém tenha notado ainda?

## Priorização

Critério: cadência real de publicação da fonte, não a frequência do schedule do flow (a maioria roda diariamente — o que importa é de quanto em quanto tempo a fonte publica, porque é isso que determina quantos períodos um poll travado deixaria passar).

### Alta — fonte mensal ou mais frequente, calendário público e fácil de conferir

| Dataset / tabela | Cadência real da fonte | Por que é prioridade |
|---|---|---|
| `br_ibge_ipca`, `br_ibge_ipca15`, `br_ibge_inpc` | Mensal, dia fixo | Índice de inflação oficial — calendário de divulgação do IBGE é público e rígido; fácil confirmar se falta o mês mais recente |
| `br_me_cnpj` / `br_rf_cnpj` (empresas/socios/estabelecimentos) | Mensal | Dump da Receita Federal, alto volume de consulta — qualquer atraso de mês é visível |
| `br_bcb_estban` | Mensal | Dado bancário do BCB, calendário fixo |
| `br_denatran_frota` | Mensal | Frota de veículos, calendário fixo |
| `br_anp_precos_combustiveis` | Semanal | Alta frequência → um poll travado acumula lacuna rápido |
| `br_inmet_bdmep` | Quase diária | Alta frequência; uma lacuna de semanas seria grave |

### Média — provavelmente mensal, cadência não confirmada com a mesma certeza

| Dataset / tabela | Cadência estimada |
|---|---|
| `br_cgu_servidores_executivo_federal` | Mensal (folha de pagamento) |
| `br_cgu_cartao_pagamento` | Mensal |
| `br_ms_sia`, `br_ms_sih`, `br_ms_sinan` | Mensal (DATASUS) |
| `br_anatel_banda_larga_fixa`, `br_anatel_telefonia_movel` | Mensal |

### Baixa — cadência irregular ou incerta

| Dataset / tabela | Observação |
|---|---|
| `br_cgu_licitacao_contrato` | Contínuo, sem "fechamento de mês" claro |
| `br_cgu_beneficios_cidadao` | Cadência incerta |
| `br_rf_cno` | Cadência incerta |
| `br_rf_cafir` | Provavelmente baixa frequência |
| `br_bcb_agencia` | Registro que muda pouco (cadastro de agências) |
| `br_stf_corte_aberta` | Cadência incerta |

## Confiança e caveat

Essa priorização é uma **hipótese** baseada em conhecimento geral sobre cada fonte (calendários de divulgação institucionais conhecidos), cruzada com o schedule real de cada flow (confirmado via `grep` dos padrões cron em cada `flows.py` — a maioria roda diária ou quase-diariamente, o que não é o fator limitante aqui). Não é evidência checada — nenhuma tabela da lista foi verificada ainda contra o BigQuery ou contra a fonte real.

## Próximo passo — método de verificação proposto

Para cada tabela, nessa ordem de prioridade:
1. Ler o max real de cobertura no BigQuery (`SELECT MAX(...)` nas colunas de competência da tabela).
2. Comparar contra o calendário de divulgação oficial da fonte (quando existe um conhecido/público) ou contra o dado mais recente disponível diretamente na fonte.
3. Se houver divergência (BigQuery atrasado em relação ao que a fonte já publicou), documentar como um caso confirmado — mesmo padrão do CAGED/ANS — e decidir a correção manual de metadados necessária (igual ao que foi feito nesses dois casos).
4. Complementar checando `Table.Update.latest` de cada tabela via GraphQL, pra confirmar a assinatura de contaminação (hora/minuto/segundo não-zero, ou data adiantada em relação à cobertura real).

Auditoria começa pelos 6 datasets da lista de prioridade alta.
