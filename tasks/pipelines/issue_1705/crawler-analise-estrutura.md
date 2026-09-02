# Análise da pasta `crawler/`

## Contexto

A pasta `pipelines/crawler/` concentra o código de ingestão de dados externos — downloads, scraping e parsing de fontes primárias. Ela surgiu no Prefect 0 com um padrão genérico chamado "dump from URL / dump to GCS" e cresceu sem uma convenção de nomenclatura uniforme.

Enquanto `pipelines/datasets/` usa o `dataset_id` exato do BigQuery como nome de pasta (ex: `br_me_caged/`), `crawler/` usa nomes arbitrários — às vezes o órgão (`cgu/`), às vezes a tabela (`bcb_taxa_cambio/`), às vezes um código diferente do dataset real (`bcb/` para `br_bcb_sicor`, `isp/` para `br_rj_isp_estatisticas_seguranca`).

---

## Problemas identificados

### 1. Nomenclatura inconsistente com `datasets/`

A maioria das pastas em `crawler/` não tem o mesmo nome que o dataset correspondente em `datasets/`. Alguns casos:

| Pasta em `crawler/` | Dataset real em `datasets/` | Problema |
|---|---|---|
| `bcb/` | `br_bcb_sicor` | Nome genérico (BCB como órgão, não como dataset) |
| `cvm/` | `br_cvm_fi` | Ambíguo — CVM tem 3 datasets distintos |
| `isp/` | `br_rj_isp_estatisticas_seguranca` | Nome completamente diferente — perde "RJ" e "segurança" |
| `bndes/` | `br_bndes_operacoes_contratadas` | Nome truncado |
| `rf/` | `br_rf_cno` | Genérico — Receita Federal tem múltiplos datasets |
| `datasus/` | `br_ms_cnes`, `br_ms_sia`, `br_ms_sih`, `br_ms_sinan` | Serve 4 datasets do Ministério da Saúde sob um nome interno |

### 2. Crawlers que servem múltiplos datasets

Três crawlers concentram código de múltiplos datasets usando `if dataset_id == ...` no meio das tasks — o que mistura lógicas distintas e dificulta manutenção:

**`crawler/cgu/`** → 4 datasets:
- `br_cgu_beneficios_cidadao`
- `br_cgu_cartao_pagamento`
- `br_cgu_licitacao_contrato`
- `br_cgu_servidores_executivo_federal`

**`crawler/datasus/`** → 4 datasets:
- `br_ms_cnes`
- `br_ms_sia`
- `br_ms_sih`
- `br_ms_sinan`

**`crawler/ibge_inflacao/`** → 3 datasets:
- `br_ibge_inpc`
- `br_ibge_ipca`
- `br_ibge_ipca15`

### 3. Estrutura interna única em `anatel/`

`crawler/anatel/` é o único que usa subpastas por tabela:

```
crawler/anatel/
  banda_larga_fixa/
  telefonia_movel/
```

Todos os outros usam estrutura plana. Isso é uma exceção não documentada.

### 4. Status de migração para Prefect 3

14 dos 36 crawlers têm `flows.py` (Prefect 3). Os demais ainda têm apenas `tasks.py` e `utils.py` do Prefect 0, sem flow ativo.

Com `flows.py` (migrados):
`bcb`, `bndes`, `camara_dados_abertos`, `cgu`, `cvm`, `cvm_administradores_carteira`, `datasus`, `fgv_igp`, `ibge_inflacao`, `isp`, `me_cnpj`, `me_rais`, `rf`, `tse_eleicoes`

Sem `flows.py` (não migrados — código legado Prefect 0):
`anatel`, `anp_precos_combustiveis`, `ans_beneficiario`, `bcb_agencia`, `bcb_estban`, `bcb_taxa_cambio`, `bd_indicadores`, `bd_siga_o_dinheiro`, `cgu_emendas_parlamentares`, `cgu_pessoal_executivo_federal`, `cnj_improbidade_administrativa`, `cvm_oferta_publica_distribuicao`, `denatran_frota`, `ibge_pnadc`, `inmet_bdmep`, `me_caged`, `me_comex_stat`, `mp_pep`, `poder360_pesquisas`, `rf_cafir`, `sfb_sicar`, `stf_corte_aberta`, `world_sofascore_competicoes_futebol`

### 5. O nome "crawler" não descreve bem o conteúdo

A maioria não faz scraping — são downloads diretos de portais de dados abertos (FTP, zip, CSV). "Crawler" vem do jargão do Prefect 0 e não é o termo mais preciso.

---

## Inventário completo

| Pasta em `crawler/` | Dataset(s) em `datasets/` | Flows Prefect 3 | Observação |
|---|---|---|---|
| `anatel/banda_larga_fixa/` | `br_anatel_banda_larga_fixa` | não | Subpasta — estrutura única |
| `anatel/telefonia_movel/` | `br_anatel_telefonia_movel` | não | Subpasta — estrutura única |
| `anp_precos_combustiveis/` | `br_anp_precos_combustiveis` | não | |
| `ans_beneficiario/` | `br_ans_beneficiario` | não | |
| `bcb/` | `br_bcb_sicor` | **sim** | Nome genérico — deveria ser `bcb_sicor` |
| `bcb_agencia/` | `br_bcb_agencia` | não | |
| `bcb_estban/` | `br_bcb_estban` | não | |
| `bcb_taxa_cambio/` | `br_bcb_taxa_cambio` | não | |
| `bd_indicadores/` | `br_bd_indicadores` | não | |
| `bd_siga_o_dinheiro/` | `br_bd_siga_o_dinheiro` | não | |
| `bndes/` | `br_bndes_operacoes_contratadas` | **sim** | Nome truncado |
| `camara_dados_abertos/` | `br_camara_dados_abertos` | **sim** | |
| `cgu/` | `br_cgu_beneficios_cidadao`, `br_cgu_cartao_pagamento`, `br_cgu_licitacao_contrato`, `br_cgu_servidores_executivo_federal` | **sim** | 4 datasets num único crawler |
| `cgu_emendas_parlamentares/` | `br_cgu_emendas_parlamentares` | não | |
| `cgu_pessoal_executivo_federal/` | `br_cgu_pessoal_executivo_federal` | não | |
| `cnj_improbidade_administrativa/` | `br_cnj_improbidade_administrativa` | não | |
| `cvm/` | `br_cvm_fi` | **sim** | Nome genérico — deveria ser `cvm_fi` |
| `cvm_administradores_carteira/` | `br_cvm_administradores_carteira` | **sim** | |
| `cvm_oferta_publica_distribuicao/` | `br_cvm_oferta_publica_distribuicao` | não | |
| `datasus/` | `br_ms_cnes`, `br_ms_sia`, `br_ms_sih`, `br_ms_sinan` | **sim** | 4 datasets MS, nome não reflete órgão real (MS) |
| `denatran_frota/` | `br_denatran_frota` | não | |
| `fgv_igp/` | `br_fgv_igp` | **sim** | |
| `ibge_inflacao/` | `br_ibge_inpc`, `br_ibge_ipca`, `br_ibge_ipca15` | **sim** | 3 datasets, lógica compartilhada legítima |
| `ibge_pnadc/` | `br_ibge_pnadc` | não | |
| `inmet_bdmep/` | `br_inmet_bdmep` | não | |
| `isp/` | `br_rj_isp_estatisticas_seguranca` | **sim** | Nome muito diferente do dataset |
| `me_caged/` | `br_me_caged` | não | |
| `me_cnpj/` | `br_me_cnpj` | **sim** | Também usa `br_bd_diretorios_brasil` como auxiliar |
| `me_comex_stat/` | `br_me_comex_stat` | não | |
| `me_rais/` | `br_me_rais` | **sim** | |
| `mp_pep/` | `br_mp_pep` | não | `br_mp_pep_cargos_funcoes` em `datasets/` sem crawler correspondente |
| `poder360_pesquisas/` | `br_poder360_pesquisas` | não | |
| `rf/` | `br_rf_cno` | **sim** | Nome genérico — deveria ser `rf_cno` |
| `rf_cafir/` | `br_rf_cafir` | não | |
| `sfb_sicar/` | `br_sfb_sicar` | não | |
| `stf_corte_aberta/` | `br_stf_corte_aberta` | não | |
| `tse_eleicoes/` | `br_tse_eleicoes` | **sim** | |
| `world_sofascore_competicoes_futebol/` | `world_sofascore_competicoes_futebol` | não | |

Datasets em `datasets/` sem nenhum crawler correspondente:
`br_b3_cotacoes`, `br_bcb_taxa_selic`, `br_mercadolivre_ofertas`, `br_mg_belohorizonte_smfa_iptu`, `br_mp_pep_cargos_funcoes`, `br_ms_sinan` (flow em datasus), `br_ons_avaliacao_operacao`, `br_ons_estimativa_custos`, `br_sp_saopaulo_dieese_icv`, `br_tse_filiacao_partidaria`

---

## Proposta de reorganização

### Direção geral

A migração para Prefect 3 é a oportunidade natural de resolver isso. A proposta é **eliminar a pasta `crawler/` progressivamente**, movendo cada crawler para dentro de `datasets/<dataset_id>/` conforme é migrado. O código de ingestão (download, parse) vai junto — não há razão para separá-lo do flow que o usa.

### Regras por tipo de crawler

**Caso 1 — 1 crawler : 1 dataset**
Mover tudo para `datasets/<dataset_id>/`. Exemplo:
```
crawler/bcb_taxa_cambio/  →  datasets/br_bcb_taxa_cambio/
crawler/tse_eleicoes/     →  datasets/br_tse_eleicoes/
```

**Caso 2 — 1 crawler : N datasets (cgu, datasus, ibge_inflacao)**
Duas opções:

a) Se a lógica de download é genuinamente compartilhada (mesmo endpoint, mesmos parâmetros), extrair as utils para `utils/<orgao>.py` e criar um `flows.py` separado em cada `datasets/<dataset_id>/`.

b) Se os datasets apenas coexistem historicamente sem lógica realmente compartilhada, separar cada um em sua própria pasta com código independente.

`ibge_inflacao/` é o candidato mais claro para a opção (a): os três índices (IPCA, INPC, IPCA15) usam a mesma API do IBGE com parâmetros diferentes.

**Caso 3 — `anatel/` com subpastas**
Aplanar para o padrão do restante:
```
crawler/anatel/banda_larga_fixa/  →  datasets/br_anatel_banda_larga_fixa/
crawler/anatel/telefonia_movel/   →  datasets/br_anatel_telefonia_movel/
```

### Sobre o nome "crawler"

Alternativas consideradas:
- `ingestao/` — mais preciso tecnicamente, mas mistura download com scraping
- `coleta/` — genérico demais
- Eliminar a categoria e usar só `datasets/` — a mais limpa, e já é o que está acontecendo gradualmente

A recomendação é não criar uma nova pasta intermediária: migrar direto para `datasets/` e deixar `crawler/` encolher até poder ser removida.
