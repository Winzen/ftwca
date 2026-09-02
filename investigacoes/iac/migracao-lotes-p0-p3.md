# Migração P0 → P3: Ativação dos 111 flows 🟢 Dev OK

> Flows validados em dev, ativados em prod (Prefect 3) e desativados no Prefect 0.
> Concluído em **2026-06-10**.

---

**Status dos lotes:**

| Lote | Flows | Datasets | Status |
|---|---|---|---|
| 1 | 26 | `br_camara_dados_abertos` | ✅ Concluído (2026-06-10) |
| 2 | 15 | `br_ibge_ipca` + `br_ibge_ipca15` (antecipado) | ✅ Concluído (2026-06-10) |
| 3 | 15 | `br_ibge_inpc`, `br_bcb_estban`, `br_me_comex_stat`, `br_bcb_agencia`, `br_anatel_banda_larga_fixa` | ✅ Concluído (2026-06-10) |
| 4 | 33 | `br_anp_precos_combustiveis`, `br_cgu_beneficios_cidadao`, `br_cgu_cartao_pagamento`, `br_cgu_licitacao_contrato`, `br_cgu_servidores_executivo_federal`, `br_denatran_frota`, `br_cvm_administradores_carteira`, `br_cvm_fi`, `br_cvm_oferta_publica_distribuicao`, `br_ibge_ipca15` | ✅ Concluído (2026-06-10) |
| 5 | 33 | `br_me_cnpj`, `br_ms_cnes`, `br_ms_sia`, `br_ms_sih`, `br_ms_sinan`, `br_rf_cafir`, `br_rf_cno`, `br_rj_isp_estatisticas_seguranca` | ✅ Concluído (2026-06-10) |

---

## Lote 1 — `br_camara_dados_abertos` (26 flows)

**Status:** ✅ Concluído (2026-06-10)

| Flow | Schedule P3 | P3 Ativo | P0 Desativado |
|---|---|---|---|
| `br_camara_dados_abertos__votacao` | `0 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__votacao_objeto` | `10 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__votacao_orientacao_bancada` | `20 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__votacao_parlamentar` | `30 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__votacao_proposicao` | `40 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__deputado` | `50 6 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__deputado_ocupacao` | `0 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__deputado_profissao` | `10 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__proposicao_microdados` | `20 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__proposicao_autor` | `30 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__proposicao_tema` | `40 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__orgao` | `50 7 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__orgao_deputado` | `0 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__evento` | `10 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__evento_orgao` | `20 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__evento_presenca_deputado` | `30 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__evento_requerimento` | `40 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__funcionario` | `50 8 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__frente` | `0 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__frente_deputado` | `10 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__licitacao` | `15 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__licitacao_proposta` | `20 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__licitacao_contrato` | `30 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__licitacao_item` | `40 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__licitacao_pedido` | `50 9 * * *` | ✅ | ✅ |
| `br_camara_dados_abertos__despesa` | `0 10 * * *` | ✅ | ✅ |

---

## Lote 2 — `br_ibge_ipca` + `br_ibge_ipca15` (15 flows)

**Status:** ✅ Concluído (2026-06-10)

> `br_ibge_ipca15` foi antecipado aqui pois o P0 desativou ambos com o mesmo prefixo de query.

| Flow | Schedule P3 | P3 Ativo | P0 Desativado |
|---|---|---|---|
| `br_ibge_ipca__mes_brasil` | `40 14 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_ipca__mes_categoria_brasil` | `30 14 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_ipca__mes_categoria_rm` | `20 14 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_ipca__mes_categoria_municipio` | `50 14 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_ipca15__mes_brasil` | `15 13 23,24,25,26,27 * *` | ✅ | ✅ |
| `br_ibge_ipca15__mes_categoria_brasil` | `30 13 23,24,25,26,27 * *` | ✅ | ✅ |
| `br_ibge_ipca15__mes_categoria_rm` | `20 13 23,24,25,26,27 * *` | ✅ | ✅ |
| `br_ibge_ipca15__mes_categoria_municipio` | `10 13 23,24,25,26,27 * *` | ✅ | ✅ |

---

## Lote 3 — IBGE INPC, BCB, Comex, Anatel (15 flows)

**Status:** ✅ Concluído (2026-06-10)

| Flow | Schedule P3 | P3 Ativo | P0 Desativado |
|---|---|---|---|
| `br_ibge_inpc__mes_brasil` | `20 15 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_inpc__mes_categoria_brasil` | `50 15 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_inpc__mes_categoria_rm` | `40 15 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_ibge_inpc__mes_categoria_municipio` | `30 15 8,9,10,11,12,13 * *` | ✅ | ✅ |
| `br_bcb_estban__agencia` | `0 22 25-31 * *` | ✅ | ✅ |
| `br_bcb_estban__municipio` | `30 22 25-31 * *` | ✅ | ✅ |
| `br_me_comex_stat__municipio_exportacao` | `0 21 * * 1-5` | ✅ | ✅ |
| `br_me_comex_stat__municipio_importacao` | `0 20 * * 1-5` | ✅ | ✅ |
| `br_me_comex_stat__ncm_exportacao` | `0 8,17 * * 1-5` | ✅ | ✅ |
| `br_me_comex_stat__ncm_importacao` | `0 8,17 * * 1-5` | ✅ | ✅ |
| `br_bcb_agencia__agencia` | `0 22 25-31 * *` | ✅ | ✅ |
| `br_anatel_banda_larga_fixa__microdados` | `0 15 * * *` | ✅ | ✅ |
| `br_anatel_banda_larga_fixa__densidade_municipio` | `0 16 * * *` | ✅ | ✅ |
| `br_anatel_banda_larga_fixa__densidade_brasil` | `0 17 * * *` | ✅ | ✅ |
| `br_anatel_banda_larga_fixa__densidade_uf` | `0 18 * * *` | ✅ | ✅ |

---

## Lote 4 — ANP, CGU, Denatran, CVM (33 flows)

**Status:** ✅ Concluído (2026-06-10)

| Flow | Schedule P3 | P3 Ativo | P0 Desativado |
|---|---|---|---|
| `br_anp_precos_combustiveis__microdados` | `0 10 * * *` | ✅ | ✅ |
| `br_cgu_beneficios_cidadao__novo_bolsa_familia` | `0 19 * * *` | ✅ | ✅ |
| `br_cgu_beneficios_cidadao__garantia_safra` | `15 19 * * *` | ✅ | ✅ |
| `br_cgu_beneficios_cidadao__bpc` | `30 19 * * *` | ✅ | ✅ |
| `br_cgu_cartao_pagamento__microdados_governo_federal` | `0 20 * * *` | ✅ | ✅ |
| `br_cgu_cartao_pagamento__microdados_defesa_civil` | `30 20 * * *` | ✅ | ✅ |
| `br_cgu_cartao_pagamento__microdados_compras_centralizadas` | `0 21 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__contrato_compra` | `0 21 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__contrato_item` | `15 20 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__contrato_termo_aditivo` | `30 20 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__licitacao` | `45 20 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__licitacao_empenho` | sem schedule | ✅ | ✅ |
| `br_cgu_licitacao_contrato__licitacao_item` | `20 20 * * *` | ✅ | ✅ |
| `br_cgu_licitacao_contrato__licitacao_participante` | `35 20 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__cadastro_aposentados` | `0 6 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__cadastro_pensionistas` | `15 6 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__cadastro_servidores` | `30 6 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__cadastro_reserva_reforma_militares` | `45 6 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__remuneracao` | `0 7 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__afastamentos` | `15 7 * * *` | ✅ | ✅ |
| `br_cgu_servidores_executivo_federal__observacoes` | `30 7 * * *` | ✅ | ✅ |
| `br_denatran_frota__uf_tipo` | `0 21 10-30 * *` | ✅ | ✅ |
| `br_denatran_frota__municipio_tipo` | `20 21 10-30 * *` | ✅ | ✅ |
| `br_cvm_administradores_carteira__responsavel` | `12 6 * * 1-5` | ✅ | ✅ |
| `br_cvm_administradores_carteira__pessoa_fisica` | `50 6 * * 1-5` | ✅ | ✅ |
| `br_cvm_administradores_carteira__pessoa_juridica` | `0 6 * * 1-5` | ✅ | ✅ |
| `br_cvm_fi__documentos_informe_diario` | `0 17 * * *` | ✅ | ✅ |
| `br_cvm_fi__documentos_carteiras_fundos_investimento` | `10 17 * * *` | ✅ | ✅ |
| `br_cvm_fi__documentos_extratos_informacoes` | `20 17 * * *` | ✅ | ✅ |
| `br_cvm_fi__documentos_informacao_cadastral` | `30 17 * * *` | ✅ | ✅ |
| `br_cvm_oferta_publica_distribuicao__dia` | `45 6 * * 1-5` | ✅ | ✅ |

---

## Lote 5 — CNPJ, MS, RF, ISP (33 flows)

**Status:** ✅ Concluído (2026-06-10)

| Flow | Schedule P3 | P3 Ativo | P0 Desativado |
|---|---|---|---|
| `br_me_cnpj__empresas` | `0 6 * * *` | ✅ | ✅ |
| `br_me_cnpj__socios` | `0 7 * * *` | ✅ | ✅ |
| `br_me_cnpj__simples` | `0 8 * * *` | ✅ | ✅ |
| `br_me_cnpj__estabelecimentos` | `0 9 * * *` | ✅ | ✅ |
| `br_ms_cnes__profissional` | `30 6 * * *` | ✅ | ✅ |
| `br_ms_cnes__estabelecimento` | `0 9 * * *` | ✅ | ✅ |
| `br_ms_cnes__equipe` | `30 9 * * *` | ✅ | ✅ |
| `br_ms_cnes__leito` | `0 10 * * *` | ✅ | ✅ |
| `br_ms_cnes__equipamento` | `30 10 * * *` | ✅ | ✅ |
| `br_ms_cnes__estabelecimento_ensino` | sem schedule | ✅ | ✅ |
| `br_ms_cnes__dados_complementares` | `0 11 * * *` | ✅ | ✅ |
| `br_ms_cnes__estabelecimento_filantropico` | `15 11 * * *` | ✅ | ✅ |
| `br_ms_cnes__gestao_metas` | `30 11 * * *` | ✅ | ✅ |
| `br_ms_cnes__habilitacao` | `45 11 * * *` | ✅ | ✅ |
| `br_ms_cnes__incentivos` | `50 11 * * *` | ✅ | ✅ |
| `br_ms_cnes__servico_especializado` | `30 12 * * *` | ✅ | ✅ |
| `br_ms_sia__psicossocial` | `0 7 * * *` | ✅ | ✅ |
| `br_ms_sia__producao_ambulatorial` | `0 21 * * *` | ✅ | ✅ |
| `br_ms_sih__servicos_profissionais` | `30 3 * * *` | ✅ | ✅ |
| `br_ms_sih__aihs_reduzidas` | `30 6 * * *` | ✅ | ✅ |
| `br_ms_sinan__microdados_dengue` | sem schedule | ✅ | ✅ |
| `br_rf_cafir__imoveis_rurais` | `0 0 * * *` | ✅ | ✅ |
| `br_rf_cno__microdados` | `5 4 * * 1-5` | ✅ | ✅ |
| `br_rf_cno__vinculos` | `15 4 * * 1-5` | ✅ | ✅ |
| `br_rf_cno__areas` | `25 4 * * 1-5` | ✅ | ✅ |
| `br_rf_cno__cnaes` | `35 4 * * 1-5` | ✅ | ✅ |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_cisp` | `5 10 * * *` | ✅ | ✅ |
| `br_rj_isp_estatisticas_seguranca__evolucao_policial_morto_servico_mensal` | `10 10 * * *` | ✅ | ✅ |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_municipio` | `20 10 * * 5` | ✅ | ✅ |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_uf` | `25 10 * * *` | ✅ | ✅ |
| `br_rj_isp_estatisticas_seguranca__feminicidio_mensal_cisp` | `40 10 * * *` | ✅ | ✅ |
