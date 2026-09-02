# Status dos Deployments — Prefect 3
**Data:** 2026-06-15

---

## ✅ Ativos — 92 deployments (schedule ON)

| Dataset | Tabelas |
|---|---|
| `br_anatel_banda_larga_fixa` | densidade_brasil, densidade_municipio, densidade_uf, microdados |
| `br_anp_precos_combustiveis` | microdados |
| `br_ans_beneficiario` | informacao_consolidada |
| `br_bcb_agencia` | agencia |
| `br_bcb_taxa_cambio` | taxa_cambio |
| `br_bd_indicadores` | contabilidade, equipes, page_views, pessoas, receitas_planejadas |
| `br_bd_siga_o_dinheiro` | — |
| `br_camara_dados_abertos` | deputado, deputado_ocupacao, deputado_profissao, despesa, evento, evento_orgao, evento_presenca_deputado, evento_requerimento, frente, frente_deputado, funcionario, licitacao, licitacao_item, licitacao_pedido, licitacao_proposta, orgao, orgao_deputado, proposicao_autor, proposicao_microdados, proposicao_tema, votacao, votacao_objeto, votacao_orientacao_bancada, votacao_parlamentar, votacao_proposicao |
| `br_cvm_administradores_carteira` | pessoa_fisica, pessoa_juridica, responsavel |
| `br_cvm_fi` | documentos_balancete, documentos_carteiras_fundos_investimento, documentos_extratos_informacoes, documentos_informacao_cadastral, documentos_informe_diario, documentos_perfil_mensal |
| `br_cvm_oferta_publica_distribuicao` | dia |
| `br_denatran_frota` | municipio_tipo, uf_tipo |
| `br_fgv_igp` | igp_di_ano, igp_m_ano |
| `br_ibge_ipca15` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm |
| `br_inmet_bdmep` | microdados |
| `br_me_caged` | microdados_movimentacao, microdados_movimentacao_excluida, microdados_movimentacao_fora_prazo |
| `br_me_cnpj` | empresas, estabelecimentos, simples, socios |
| `br_me_comex_stat` | municipio_exportacao, municipio_importacao, ncm_exportacao, ncm_importacao |
| `br_me_rais` | microdados_estabelecimentos |
| `br_ms_cnes` | dados_complementares, equipe, estabelecimento, estabelecimento_ensino, estabelecimento_filantropico, gestao_metas, habilitacao, incentivos, profissional, servico_especializado |
| `br_ms_sia` | producao_ambulatorial |
| `br_ms_sih` | aihs_reduzidas, servicos_profissionais |
| `br_rf_cafir` | imoveis_rurais |
| `br_rf_cno` | cnaes |
| `br_rj_isp_estatisticas_seguranca` | evolucao_mensal_municipio, evolucao_mensal_uf, evolucao_policial_morto_servico_mensal, feminicidio_mensal_cisp |
| **Utilitários** | ibge_inflacao_flow, run_dbt_model_flow, update_temporal_coverage |

---

## ⏸️ Pausados — 81 deployments

### Grupo A — Aguardando merge do PR [#1578](https://github.com/basedosdados/pipelines/pull/1578) (33 flows)

Ao mergear o PR, o CI re-registra automaticamente todos esses flows em `main`. Após o merge, reativar schedules manualmente no Prefect.

| Dataset | Tabelas | Causa |
|---|---|---|
| `br_bcb_estban` | agencia, municipio | `clean_dataframe` removida do `utils.py` na migração P3 |
| `br_bcb_sicor` | dicionario, empreendimento, liberacao, operacao, operacoes_desclassificadas, recurso_publico_complemento_operacao, recurso_publico_cooperado, recurso_publico_gleba, recurso_publico_mutuario, recurso_publico_propriedade, saldo | `@root_validator` Pydantic v1 incompatível com v2 |
| `br_cgu_beneficios_cidadao` | bpc, garantia_safra, novo_bolsa_familia | `download_and_unzip_file` removida do `utils.py` na migração P3 |
| `br_cgu_cartao_pagamento` | microdados_compras_centralizadas, microdados_defesa_civil, microdados_governo_federal | Mesmo |
| `br_cgu_licitacao_contrato` | contrato_compra, contrato_item, contrato_termo_aditivo, licitacao, licitacao_empenho, licitacao_item, licitacao_participante | Mesmo |
| `br_cgu_servidores_executivo_federal` | afastamentos, cadastro_aposentados, cadastro_pensionistas, cadastro_reserva_reforma_militares, cadastro_servidores, observacoes, remuneracao | Mesmo |

---

### Grupo B — Precisam de migração manual (4 flows)

| Dataset | Tabelas | O que falta |
|---|---|---|
| `br_ibge_inpc` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm | `flows.py` em `main` ainda é Prefect 0. Necessário criar `flows.py` P3 seguindo o padrão do `br_ibge_ipca`. |

---

### Grupo C — Pausados por falha no último run (44 flows)

Precisam de diagnóstico individual antes de reativar.

| Dataset | Tabelas | Última falha |
|---|---|---|
| `br_anatel_telefonia_movel` | densidade_brasil, densidade_municipio, densidade_uf, microdados | Crashed — hoje |
| `br_bcb_taxa_selic` | taxa_selic | Failed — hoje |
| `br_bd_indicadores` | twitter_metrics, twitter_metrics_agg, website_user | Failed — 2026-05-25 |
| `br_camara_dados_abertos` | licitacao_contrato | — |
| `br_cgu_emendas_parlamentares` | microdados | — |
| `br_cgu_pessoal_executivo_federal` | terceirizados | — |
| `br_cnj_improbidade_administrativa` | condenacao | — |
| `br_fgv_igp` | igp_10_mes, igp_di_mes, igp_m_mes, igp_og_ano, igp_og_mes | — |
| `br_ibge_ipca` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm | Failed — 2026-06-14 |
| `br_ibge_pnadc` | microdados | — |
| `br_me_rais` | microdados_vinculos | Failed — 2026-05-25 |
| `br_mp_pep` | cargos_funcoes | — |
| `br_ms_cnes` | equipamento, leito, regra_contratual | — |
| `br_ms_sia` | psicossocial | — |
| `br_ms_sinan` | microdados_dengue | Failed — 2026-05-25 |
| `br_poder360_pesquisas` | microdados | — |
| `br_rf_cno` | areas, microdados, vinculos | — |
| `br_rj_isp_estatisticas_seguranca` | armas_apreendidas_mensal, evolucao_mensal_cisp | — |
| `br_sfb_sicar` | area_imovel | — |
| `br_stf_corte_aberta` | decisoes | — |
| `br_tse_eleicoes` | bens_candidato, candidatos, despesas_candidato, receitas_candidato | Failed — 2026-05-25 |
| `fundacao_lemann` | ano_escola_serie_educacao_aprendizagem_adequada | Failed — 2026-05-25 |
| **Testes** | hello_prefect3, hello-prefect3-prod | — |
