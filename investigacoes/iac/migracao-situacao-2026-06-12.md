# Situação da Migração — Prefect 3
**Data:** 2026-06-12

---

## ✅ Funcionando — 134 flows ativos em `main`

Todos rodando com `--branch main`, schedule ativo e infraestrutura Prefect 3.

| Dataset | Tabelas |
|---|---|
| `br_anatel_banda_larga_fixa` | densidade_brasil, densidade_municipio, densidade_uf, microdados |
| `br_anatel_telefonia_movel` | densidade_brasil, densidade_municipio, densidade_uf, microdados |
| `br_anp_precos_combustiveis` | microdados |
| `br_ans_beneficiario` | informacao_consolidada |
| `br_bcb_agencia` | agencia |
| `br_bcb_taxa_cambio` | taxa_cambio |
| `br_bcb_taxa_selic` | taxa_selic |
| `br_bd_indicadores` | contabilidade, equipes, page_views, pessoas, receitas_planejadas, twitter_metrics, twitter_metrics_agg, website_user |
| `br_bd_siga_o_dinheiro` | — |
| `br_camara_dados_abertos` | deputado, deputado_ocupacao, deputado_profissao, despesa, evento, evento_orgao, evento_presenca_deputado, evento_requerimento, frente, frente_deputado, funcionario, licitacao, licitacao_contrato, licitacao_item, licitacao_pedido, licitacao_proposta, orgao, orgao_deputado, proposicao_autor, proposicao_microdados, proposicao_tema, votacao, votacao_objeto, votacao_orientacao_bancada, votacao_parlamentar, votacao_proposicao |
| `br_cgu_emendas_parlamentares` | microdados |
| `br_cgu_pessoal_executivo_federal` | terceirizados |
| `br_cnj_improbidade_administrativa` | condenacao |
| `br_cvm_administradores_carteira` | pessoa_fisica, pessoa_juridica, responsavel |
| `br_cvm_fi` | documentos_balancete, documentos_carteiras_fundos_investimento, documentos_extratos_informacoes, documentos_informacao_cadastral, documentos_informe_diario, documentos_perfil_mensal |
| `br_cvm_oferta_publica_distribuicao` | dia |
| `br_denatran_frota` | municipio_tipo, uf_tipo |
| `br_fgv_igp` | igp_10_mes, igp_di_ano, igp_di_mes, igp_m_ano, igp_m_mes, igp_og_ano, igp_og_mes |
| `br_ibge_ipca` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm |
| `br_ibge_ipca15` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm |
| `br_ibge_pnadc` | microdados |
| `br_inmet_bdmep` | microdados |
| `br_me_caged` | microdados_movimentacao, microdados_movimentacao_excluida, microdados_movimentacao_fora_prazo |
| `br_me_cnpj` | empresas, estabelecimentos, simples, socios |
| `br_me_comex_stat` | municipio_exportacao, municipio_importacao, ncm_exportacao, ncm_importacao |
| `br_me_rais` | microdados_estabelecimentos, microdados_vinculos |
| `br_mp_pep` | cargos_funcoes |
| `br_ms_cnes` | dados_complementares, equipamento, equipe, estabelecimento, estabelecimento_ensino, estabelecimento_filantropico, gestao_metas, habilitacao, incentivos, leito, profissional, regra_contratual, servico_especializado |
| `br_ms_sia` | producao_ambulatorial, psicossocial |
| `br_ms_sih` | aihs_reduzidas, servicos_profissionais |
| `br_ms_sinan` | microdados_dengue |
| `br_poder360_pesquisas` | microdados |
| `br_rf_cafir` | imoveis_rurais |
| `br_rf_cno` | areas, cnaes, microdados, vinculos |
| `br_rj_isp_estatisticas_seguranca` | armas_apreendidas_mensal, evolucao_mensal_cisp, evolucao_mensal_municipio, evolucao_mensal_uf, evolucao_policial_morto_servico_mensal, feminicidio_mensal_cisp |
| `br_sfb_sicar` | area_imovel |
| `br_stf_corte_aberta` | decisoes |
| `br_tse_eleicoes` | bens_candidato, candidatos, despesas_candidato, receitas_candidato |
| `fundacao_lemann` | ano_escola_serie_educacao_aprendizagem_adequada |

---

## ⏸️ Pausados — 37 flows

Pausados para evitar crashes contínuos causados pela deleção da branch `feat/prefect3-flows-migration`. Divididos em dois grupos conforme o esforço de reativação.

---

### Grupo A — Reativados automaticamente após merge do PR [#1578](https://github.com/basedosdados/pipelines/pull/1578)

Ao mergear o PR, o CI re-registra todos esses flows em `main` com o código corrigido. Os schedules precisam ser reativados manualmente no Prefect após o merge.

| Dataset | Tabelas | Causa |
|---|---|---|
| `br_bcb_estban` | agencia, municipio | Função `clean_dataframe` removida do `utils.py` na migração P3 |
| `br_bcb_sicor` | dicionario, empreendimento, liberacao, operacao, operacoes_desclassificadas, recurso_publico_complemento_operacao, recurso_publico_cooperado, recurso_publico_gleba, recurso_publico_mutuario, recurso_publico_propriedade, saldo | `@root_validator` do Pydantic v1 incompatível com Pydantic v2 |
| `br_cgu_beneficios_cidadao` | bpc, garantia_safra, novo_bolsa_familia | Função `download_and_unzip_file` removida do `utils.py` na migração P3 |
| `br_cgu_cartao_pagamento` | microdados_compras_centralizadas, microdados_defesa_civil, microdados_governo_federal | Mesmo |
| `br_cgu_licitacao_contrato` | contrato_compra, contrato_item, contrato_termo_aditivo, licitacao, licitacao_empenho, licitacao_item, licitacao_participante | Mesmo |
| `br_cgu_servidores_executivo_federal` | afastamentos, cadastro_aposentados, cadastro_pensionistas, cadastro_reserva_reforma_militares, cadastro_servidores, observacoes, remuneracao | Mesmo |

---

### Grupo B — Precisam de migração manual

| Dataset | Tabelas | O que falta |
|---|---|---|
| `br_ibge_inpc` | mes_brasil, mes_categoria_brasil, mes_categoria_municipio, mes_categoria_rm | O `flows.py` em `main` ainda é Prefect 0. O deploy antigo era feito diretamente da branch `feat/prefect3-flows-migration` (já deletada). É necessário criar um `flows.py` Prefect 3 seguindo o padrão dos outros datasets IBGE (ex: `br_ibge_ipca`). |

---

## 🔧 Dívidas técnicas identificadas na revisão do backup (2026-06-15)

Comparação entre `feat/prefect3-flows-migration-backup` e `main` revelou código que ficou para trás na migração. Itens em ordem de prioridade.

---

### P0 — Quebram em runtime

| # | Arquivo | Problema | Solução |
|---|---|---|---|
| 1 | `pipelines/crawler/ibge_inflacao/utils.py` | Duas chamadas usam `.run(...)` — método do Prefect 0 que não existe no P3. Causa `AttributeError` nos flows de IBGE Inflação (IPCA, IPCA-15, INPC). | Substituir `.run(...)` por `.fn(...)` nos dois call sites (linhas 378 e 387). |
| 2 | `pipelines/utils/tasks.py` — `_upload_to_gcs` | `bd.Table` e `bd.Storage` instanciados sem `billing_project_id`. A lib usa `basedosdados-staging` como padrão, mas a SA do pod só tem `serviceusage.services.use` no próprio projeto — causa 403 em `blob.exists()`. | Adicionar `billing_project_id = bucket_name` e passar para `bd.Table(...)` e `bd.Storage(...)`. |
| 3 | `pipelines/utils/utils.py` | `clean_dataframe` e `download_and_unzip_file` removidas na migração P3. Causa `ImportError` em `bcb_estban` e todos os crawlers `cgu`. | **Em andamento** — PR [#1578](https://github.com/basedosdados/pipelines/pull/1578). |
| 4 | `pipelines/utils/schema_validator.py` | `@root_validator(pre=False)` sem `skip_on_failure=True` — incompatível com Pydantic v2. Causa erro de importação no `br_bcb_sicor`. | **Em andamento** — PR [#1578](https://github.com/basedosdados/pipelines/pull/1578). |
| 5 | `pipelines/datasets/br_ibge_inpc/flows.py` | Arquivo ainda é Prefect 0 (`KubernetesRun`, `prefect.run_configs`, `prefect.storage`). Não pode ser registrado no P3. | Criar `flows.py` Prefect 3 (ver Grupo B acima). |

---

### P1 — Problema de infraestrutura (403 em artefatos dbt)

| # | Arquivo | Problema | Solução |
|---|---|---|---|
| 6 | `pipelines/utils/gcs.py` — `DBTArtifactUploader` | Main usa ADC do pod para criar o `storage.Client`, que pega a SA prod (`dbt-rpc@basedosdados`). Quando o flow escreve no bucket dev (`basedosdados-dev`), o GCS tenta cobrar cotas no projeto `basedosdados-dev`, mas a SA prod não tem `serviceusage.services.use` lá — 403. O backup selecionava a credencial correta por `target` (SA dev para bucket dev). | Opções: (a) portar seleção de credencial por `target` do `gcs.py` do backup; (b) conceder `serviceusage.services.use` de `dbt-rpc@basedosdados` no projeto `basedosdados-dev`; (c) configurar Workload Identity por namespace. |
