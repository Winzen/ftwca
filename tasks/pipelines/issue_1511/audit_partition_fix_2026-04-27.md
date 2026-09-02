# Audit de isPartition — 2026-04-27 (prod)

**Modo:** Aplicadas  
**Tabelas verificadas:** 240  
**Colunas verificadas:** 11828  
**Alterações:** 157  
**Erros:** 0

## administradores_carteira

### pessoa_fisica (`br_cvm_administradores_carteira.pessoa_fisica`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| data_registro | False | True | partition/cluster no SQL |

### pessoa_juridica (`br_cvm_administradores_carteira.pessoa_juridica`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| data_registro | False | True | partition/cluster no SQL |

## agencia

### agencia (`br_bcb_agencia.agencia`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | True | False | não está no SQL |

## arrecadacao

### itr (`br_rf_arrecadacao.itr`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### uf (`br_rf_arrecadacao.uf`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |

## avaliacao_operacao

### energia_armazenada_reservatorio (`br_ons_avaliacao_operacao.energia_armazenada_reservatorio`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |

### energia_natural_afluente (`br_ons_avaliacao_operacao.energia_natural_afluente`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |

### geracao_termica_motivo_despacho (`br_ons_avaliacao_operacao.geracao_termica_motivo_despacho`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |

## banda_larga_fixa

### microdados (`br_anatel_banda_larga_fixa.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |
| sigla_uf | True | False | não está no SQL |

## beneficiario

### informacao_consolidada (`br_ans_beneficiario.informacao_consolidada`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## beneficios_cidadao

### auxilio_brasil (`br_cgu_beneficios_cidadao.auxilio_brasil`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |

### auxilio_emergencial (`br_cgu_beneficios_cidadao.auxilio_emergencial`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### bolsa_familia_pagamento (`br_cgu_beneficios_cidadao.bolsa_familia_pagamento`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano_referencia | False | True | partition/cluster no SQL |
| id_municipio | False | True | partition/cluster no SQL |
| mes_referencia | False | True | partition/cluster no SQL |

### bpc (`br_cgu_beneficios_cidadao.bpc`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes_competencia | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### garantia_safra (`br_cgu_beneficios_cidadao.garantia_safra`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes_referencia | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### novo_bolsa_familia (`br_cgu_beneficios_cidadao.novo_bolsa_familia`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes_competencia | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## br_inmet_bdmep

### microdados (`br_inmet_bdmep.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_estacao | False | True | partition/cluster no SQL |

## caged

### microdados_movimentacao_fora_prazo (`br_me_caged.microdados_movimentacao_fora_prazo`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | True | False | não está no SQL |
| sigla_uf | True | False | não está no SQL |

## censo_2022

### setor_censitario (`br_ibge_censo_2022.setor_censitario`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_uf | False | True | partition/cluster no SQL |

## censo_educacao_superior

### curso (`br_inep_censo_educacao_superior.curso`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### ies (`br_inep_censo_educacao_superior.ies`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## cnes

### estabelecimento (`br_ms_cnes.estabelecimento`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | True | False | não está no SQL |
| sigla_uf | True | False | não está no SQL |

### profissional (`br_ms_cnes.profissional`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | True | False | não está no SQL |
| sigla_uf | True | False | não está no SQL |

## cnpj

### empresas (`br_me_cnpj.empresas`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### estabelecimentos (`br_me_cnpj.estabelecimentos`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |
| sigla_uf | True | False | não está no SQL |

### socios (`br_me_cnpj.socios`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

## comex_stat

### ncm_importacao (`br_me_comex_stat.ncm_importacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_pais_iso3 | True | False | não está no SQL |
| sigla_uf_ncm | False | True | partition/cluster no SQL |

## cotacoes

### cotacoes (`br_b3_cotacoes.cotacoes`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| acao_atualizacao | False | True | partition/cluster no SQL |

## dados_abertos

### proposicao_microdados (`br_camara_dados_abertos.proposicao_microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### proposicao_tema (`br_camara_dados_abertos.proposicao_tema`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

## diretorios_brasil

### empresa (`br_bd_diretorios_brasil.empresa`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## educacao_especial

### etapa_ensino (`br_inep_educacao_especial.etapa_ensino`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### faixa_etaria (`br_inep_educacao_especial.faixa_etaria`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### localizacao (`br_inep_educacao_especial.localizacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### sexo_raca_cor (`br_inep_educacao_especial.sexo_raca_cor`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### tempo_ensino (`br_inep_educacao_especial.tempo_ensino`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### tipo_deficiencia (`br_inep_educacao_especial.tipo_deficiencia`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## eleicoes

### partidos (`br_tse_eleicoes.partidos`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | False | True | partition/cluster no SQL |

### receitas_comite (`br_tse_eleicoes.receitas_comite`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### receitas_orgao_partidario (`br_tse_eleicoes.receitas_orgao_partidario`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

## emissoes

### municipio (`br_seeg_emissoes.municipio`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### uf (`br_seeg_emissoes.uf`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## estimativa_custos

### balanco_energia_subsistemas_dessem (`br_ons_estimativa_custos.balanco_energia_subsistemas_dessem`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### custo_variavel_unitario_usinas_termicas (`br_ons_estimativa_custos.custo_variavel_unitario_usinas_termicas`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

## fi

### documentos_balancete (`br_cvm_fi.documentos_balancete`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| data_competencia | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### documentos_carteiras_fundos_investimento (`br_cvm_fi.documentos_carteiras_fundos_investimento`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_fundo | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### documentos_extratos_informacoes (`br_cvm_fi.documentos_extratos_informacoes`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| data_competencia | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### documentos_informacao_cadastral (`br_cvm_fi.documentos_informacao_cadastral`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_fundo | False | True | partition/cluster no SQL |

### documentos_informe_diario (`br_cvm_fi.documentos_informe_diario`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_fundo | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

### documentos_perfil_mensal (`br_cvm_fi.documentos_perfil_mensal`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| data_competencia | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

## filiacao_partidaria

### microdados_antigos (`br_tse_filiacao_partidaria.microdados_antigos`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_partido | True | False | não está no SQL |

## indicadores_educacionais

### municipio_taxa_transicao (`br_inep_indicadores_educacionais.municipio_taxa_transicao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### uf_taxa_transicao (`br_inep_indicadores_educacionais.uf_taxa_transicao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | False | True | partition/cluster no SQL |

## mides

### orgao_unidade_gestora (`world_wb_mides.orgao_unidade_gestora`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | False | True | partition/cluster no SQL |

## pam

### lavoura_permanente (`br_ibge_pam.lavoura_permanente`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| id_municipio | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### lavoura_temporaria (`br_ibge_pam.lavoura_temporaria`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| id_municipio | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## pep

### cargos_funcoes (`br_mp_pep.cargos_funcoes`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| mes | False | True | partition/cluster no SQL |

## pevs

### producao_extracao_vegetal (`br_ibge_pevs.producao_extracao_vegetal`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |

### producao_silvicultura (`br_ibge_pevs.producao_silvicultura`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |

## pisa

### student (`world_oecd_pisa.student`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| country_id_iso_3 | False | True | partition/cluster no SQL |

## pnad

### microdados_compatibilizados_domicilio (`br_ibge_pnad.microdados_compatibilizados_domicilio`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | True | False | não está no SQL |

### microdados_compatibilizados_pessoa (`br_ibge_pnad.microdados_compatibilizados_pessoa`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | True | False | não está no SQL |

## pnadc

### educacao (`br_ibge_pnadc.educacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| sigla_uf | False | True | partition/cluster no SQL |

### microdados (`br_ibge_pnadc.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### rendimentos_outras_fontes (`br_ibge_pnadc.rendimentos_outras_fontes`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## ppm

### efetivo_rebanhos (`br_ibge_ppm.efetivo_rebanhos`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### producao_aquicultura (`br_ibge_ppm.producao_aquicultura`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### producao_origem_animal (`br_ibge_ppm.producao_origem_animal`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### producao_pecuaria (`br_ibge_ppm.producao_pecuaria`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## precos_combustiveis

### microdados (`br_anp_precos_combustiveis.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## queimadas

### microdados (`br_inpe_queimadas.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |

## rais

### microdados_vinculos (`br_me_rais.microdados_vinculos`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |

## siconfi

### municipio_execucao_restos_pagar (`br_me_siconfi.municipio_execucao_restos_pagar`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### municipio_execucao_restos_pagar_funcao (`br_me_siconfi.municipio_execucao_restos_pagar_funcao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### municipio_variacoes_patrimoniais (`br_me_siconfi.municipio_variacoes_patrimoniais`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### uf_execucao_restos_pagar (`br_me_siconfi.uf_execucao_restos_pagar`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### uf_execucao_restos_pagar_funcao (`br_me_siconfi.uf_execucao_restos_pagar_funcao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

### uf_variacoes_patrimoniais (`br_me_siconfi.uf_variacoes_patrimoniais`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |

## sicor

### liberacao (`br_bcb_sicor.liberacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| ano_emissao | True | False | não está no SQL |
| mes | False | True | partition/cluster no SQL |

### operacao (`br_bcb_sicor.operacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano_safra_emissao | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### saldo (`br_bcb_sicor.saldo`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| ano_emissao | True | False | não está no SQL |
| mes | False | True | partition/cluster no SQL |

## sih

### aihs_reduzidas (`br_ms_sih.aihs_reduzidas`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| mes | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## sinopse_estatistica_educacao_basica

### docente_deficiencia (`br_inep_sinopse_estatistica_educacao_basica.docente_deficiencia`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### docente_escolaridade (`br_inep_sinopse_estatistica_educacao_basica.docente_escolaridade`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### docente_etapa_ensino (`br_inep_sinopse_estatistica_educacao_basica.docente_etapa_ensino`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### docente_faixa_etaria_sexo (`br_inep_sinopse_estatistica_educacao_basica.docente_faixa_etaria_sexo`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### docente_localizacao (`br_inep_sinopse_estatistica_educacao_basica.docente_localizacao`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### docente_regime_contrato (`br_inep_sinopse_estatistica_educacao_basica.docente_regime_contrato`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

### sexo_raca_cor (`br_inep_sinopse_estatistica_educacao_basica.sexo_raca_cor`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## sisam

### microdados (`br_inpe_sisam.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf | False | True | partition/cluster no SQL |

## sisu

### microdados (`br_mec_sisu.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| sigla_uf_candidato | False | True | partition/cluster no SQL |

## supply_chain

### soy_beans (`br_trase_supply_chain.soy_beans`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| year | False | True | partition/cluster no SQL |

## telefonia_movel

### microdados (`br_anatel_telefonia_movel.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| id_municipio | False | True | partition/cluster no SQL |
