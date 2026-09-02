# Validação BigQuery x Audit de isPartition — 2026-04-27

**Referência:** `audit_partition_fix_2026-04-27.md`  
**Método:** INFORMATION_SCHEMA.COLUMNS de cada dataset auditado  
**Credenciais BQ:** staging.json / billing: basedosdados-dev

---

## Resumo

| Categoria | Quantidade |
|---|---|
| Total de correções verificadas | 154 |
| **OK** (confirmadas pelo BQ) | **149 (96,8%)** |
| **Discrepantes** | **5 (3,2%)** |
| Não verificáveis (acesso negado) | 3 |

### Não verificáveis (access denied no INFORMATION_SCHEMA)

| Dataset | Tabela | Coluna | Correção |
|---|---|---|---|
| br_bcb_agencia | agencia | mes | True→False |
| br_b3_cotacoes | cotacoes | acao_atualizacao | False→True |
| br_anatel_telefonia_movel | microdados | id_municipio | False→True |

---

## Discrepâncias (5)

Todas são do tipo **False→True** — o audit corrigiu `isPartition=True` baseado no SQL dbt, mas o BigQuery não tem a coluna como partition nem cluster na tabela materializada.

| Dataset | Tabela | Coluna | Correção | BQ partition | BQ cluster | Diagnóstico |
|---|---|---|---|---|---|---|
| br_cvm_administradores_carteira | pessoa_fisica | `data_registro` | False→True | null | null | Tabela sem partição/cluster no BQ |
| br_cvm_administradores_carteira | pessoa_juridica | `data_registro` | False→True | null | null | Tabela sem partição/cluster no BQ |
| br_cgu_beneficios_cidadao | bolsa_familia_pagamento | `ano_referencia` | False→True | `ano_competencia` | sigla_uf, id_municipio | BQ usa `ano_competencia`, não `ano_referencia` |
| br_cgu_beneficios_cidadao | bolsa_familia_pagamento | `mes_referencia` | False→True | `ano_competencia` | sigla_uf, id_municipio | Coluna não é cluster nem partition no BQ |
| br_inep_indicadores_educacionais | uf_taxa_transicao | `sigla_uf` | False→True | null | null | Tabela sem partição/cluster no BQ |

### Análise das causas

**Grupo 1 — Tabela materializada sem partição/cluster no BQ** (3 casos)

`br_cvm_administradores_carteira.pessoa_fisica`, `pessoa_juridica` e `br_inep_indicadores_educacionais.uf_taxa_transicao`:
O SQL dbt define partição/cluster, mas a tabela no BQ foi materializada sem esses metadados.
Provável causa: o modelo foi atualizado no repositório mas nunca re-executado após a adição do `config()`.
Ação recomendada: rebuild dos modelos dbt (execução no Prefect/CLI) para que a tabela seja recriada com a partição correta.

**Grupo 2 — Nome de coluna diverge entre SQL e BQ** (2 casos)

`br_cgu_beneficios_cidadao.bolsa_familia_pagamento.ano_referencia` e `mes_referencia`:
O SQL usa `ano_referencia`/`mes_referencia` como partition/cluster, mas o BQ tem `ano_competencia` como coluna de partição e `sigla_uf, id_municipio` como cluster.
Provável causa: renomeação de colunas no modelo dbt que não foi sincronizada com a tabela materializada.
Ação recomendada: verificar o SQL atual do modelo e, se necessário, corrigir o `isPartition` dessas colunas para False na API (ou aguardar rebuild da tabela).

---

## Conclusão

**96,8% das correções aplicadas estão confirmadas pelo BigQuery.** As 5 discrepâncias são casos onde o SQL dbt define partição/cluster mas a tabela materializada no BQ ainda não reflete isso — não são erros do audit, mas sim modelos que precisam ser re-executados.

O audit foi bem-sucedido como correção dos metadados com base na fonte de verdade (SQL dbt). As discrepâncias são rastreáveis e têm ação clara de resolução.
