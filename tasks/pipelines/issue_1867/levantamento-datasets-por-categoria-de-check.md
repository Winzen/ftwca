# Levantamento de datasets por categoria de check (issue #1867)

Recontagem feita em 2026-09-03, pra substituir o levantamento original de 2026-08-28 (issue #1867, seção "Levantamento dos crawlers existentes"), que só registrou as contagens (28/27/13/12) sem a lista nominal. Motivo da recontagem: precisávamos saber exatamente quais são os datasets da categoria "Padrão" pra escolher candidatos a migrar pro pipeline orientado a eventos (`check_update` → `flow_download` → `mat_test`, ver [[encapsular-check-update-e-download]]).

**Discrepância com o levantamento original** (28/27/13/12 → 33/26/10/12 nesta recontagem): provavelmente por (1) datasets `au_*` adicionados ao repo depois de 28/08, fora do universo original, e (2) algumas funções com nome sugerindo check leve (ex. `get_data_source_anp_max_date`) na verdade baixam o arquivo inteiro antes de extrair a data — parecem ter sido classificadas como "Padrão" no levantamento original sem inspeção linha a linha. Esta lista é a mais confiável até agora.

Metodologia: inspeção de `tasks.py`/`flows.py`/`constants.py` de cada pasta em `pipelines/datasets/` (e `crawler/` pros datasets ainda não migrados dessa reorganização — issue #1705), olhando como cada flow decide se há dado novo antes de baixar.

## Padrão — check leve, sem baixar o arquivo completo (33)

Candidatos primários de migração pro novo pipeline — o check já é barato, só falta encapsular em `check_update`/`flow_download` via `CheckThenDownloadPipeline`.

| Dataset | Técnica de check | Onde |
|---|---|---|
| `au_ato_abr` | HTTP HEAD (`source_last_modified`) | `tasks.py:15` |
| `au_geoscape_gnaf` | API CKAN (`resolve_source`) | `tasks.py:15` |
| `br_ans_beneficiario` | scrape leve de listagem HTML/FTP (`extract_links_and_dates`) | `crawler/ans_beneficiario/tasks.py:26` |
| `br_bcb_agencia` | API de metadados BCB (`get_documents_metadata`) | `crawler/bcb_agencia/tasks.py:38` |
| `br_bcb_estban` | API de metadados BCB (`get_documents_metadata`) | `crawler/bcb_estban/tasks.py:35` |
| `br_bcb_ifdata` | API de índice de competências (`source_max_period`/`fetch_index`) | `datasets/br_bcb_ifdata/utils.py:79` |
| `br_bcb_sicor` | listagem de links da fonte antes do download (`search_sicor_links`) | `crawler/bcb/flows.py::_run_bcb_sicor` |
| `br_bndes_operacoes_contratadas` | API/metadado (`get_source_max_date`) | `crawler/bndes/flows.py:68` |
| `br_camara_dados_abertos` | checagem de URL (`check_if_url_is_valid`) | `crawler/camara_dados_abertos/flows.py:35` |
| `br_cnj_improbidade_administrativa` | contagem BQ + scrape leve de página (`is_up_to_date`) | `crawler/cnj_improbidade_administrativa/tasks.py:238` |
| `br_cvm_fi` | scrape de listagem (`extract_links_and_dates`) | `crawler/cvm/flows.py:43` |
| `br_denatran_frota` | API do próprio backend BD (`get_api_most_recent_date`) | `tasks.py:325` |
| `br_ibge_inpc` | API IBGE (`get_date_api`) | `crawler/ibge_inflacao/tasks.py:21` |
| `br_ibge_ipca` | API IBGE (`get_date_api`) | `crawler/ibge_inflacao/tasks.py:21` |
| `br_ibge_ipca15` | API IBGE (`get_date_api`) | `crawler/ibge_inflacao/tasks.py:21` |
| `br_inmet_bdmep` | listagem/API leve (`extract_last_date_from_source`) | `flows.py` |
| `br_me_caged` | API/metadado leve (`get_source_last_date`) | `flows.py` |
| `br_me_cnpj` | leitura de índice da API, sem baixar arquivos (`data_url`) | `crawler/me_cnpj/tasks.py:29` |
| `br_me_comex_stat` | scrape leve de página (`parse_last_date`) | `crawler/me_comex_stat/tasks.py:27` |
| `br_mf_divida_ativa` | probe leve da fonte (`latest_available_quarter`) | `tasks.py:21` |
| `br_mp_pep` | checagem de página via Selenium, sem baixar dado (`is_up_to_date`) | `crawler/mp_pep/tasks.py:431` |
| `br_ms_cnes` | listagem FTP (nomes de arquivo, sem baixar conteúdo) | `crawler/datasus/tasks.py:114` |
| `br_ms_sia` | listagem FTP (nomes de arquivo, sem baixar conteúdo) | `crawler/datasus/tasks.py:114` |
| `br_ms_sih` | listagem FTP (nomes de arquivo, sem baixar conteúdo) | `crawler/datasus/tasks.py:114` |
| `br_ms_sinan` | listagem FTP (nomes de arquivo, sem baixar conteúdo) | `crawler/datasus/tasks.py:114` |
| `br_rf_cafir` | API de metadados (`task_parse_api_metadata`/`task_get_last_update_date`) | `flows.py` |
| `br_rf_cno` | check leve (`check_need_for_update`) | `crawler/rf/flows.py:43` |
| `br_rf_cnpj` | leitura de índice da API (`data_url`) | `crawler/rf_cnpj/tasks.py:32` |
| `br_sfb_sicar` | 1 page fetch — comentário explícito no código ("Cheap... before downloading gigabytes") | `flows.py` |
| `us_bls_oes` | leitura de página HTML (`resolve_latest_year`) | `tasks.py:17` |
| `us_cfpb_hmda` | leve (`resolve_years`/`latest_source_year`) | `tasks.py:12` |
| `us_sec_edgar` | API/listagem leve (`resolve_latest_quarter`) | `flows.py` |

## check_and_download — precisa baixar o arquivo pra checar (26)

### Critério de reclassificação: limiar de 5 GB

Decisão do usuário (2026-09-03): dentre os `check_and_download`, qualquer um cujo download de check fica **abaixo de 5 GB** pode ser tratado como candidato de migração tão leve quanto a categoria "Padrão" — baixar o arquivo e conferir o que aconteceu nele é barato o suficiente pra não justificar tratamento especial. Só os que **de fato** passam de 5 GB (ou onde não dá pra confirmar que ficam abaixo disso) continuam como "pesado de verdade" — esses sim precisam de um desenho mais cuidadoso (ex. `check_and_download` fundido, streaming parcial, ou checagem por HEAD/ETag na origem antes de baixar).

Coluna **Reclassificação (< 5 GB?)** abaixo aplica esse critério evidência por evidência — sem baixar nada de verdade, só a partir do que o código/comentários/README indicam. Onde a evidência é textual ("multi-GB", "several hundred MB") sem número exato, marcado como "verificar" em vez de assumir.

### Confirmados no levantamento original (2026-08-28)

| Dataset | Técnica de check | Tamanho (evidência) | Reclassificação (< 5 GB?) |
|---|---|---|---|
| `br_anatel_banda_larga_fixa` | unzip pra checar data | ~1 GB (README) | ✅ sim → tratar como check leve |
| `br_anatel_telefonia_movel` | unzip pra checar data | `memory_limit: 8Gi` no job (mesma família do anterior, ~1 GB provável) | ⚠️ verificar — memória alta sugere processamento pesado, tamanho de download não confirmado |
| `br_cgu_beneficios_cidadao` | baixa antes do poll (ZIP assíncrono, Portal da Transparência) | não documentado no código | ⚠️ verificar |
| `br_cgu_cartao_pagamento` | baixa antes do poll (mesma família de portal) | não documentado no código | ⚠️ verificar |
| `br_cgu_licitacao_contrato` | baixa antes do poll (mesma família de portal) | não documentado no código | ⚠️ verificar |
| `br_cgu_sancoes` | baixa antes do poll, snapshot cumulativo (histórico completo + ativo) | ZIP inteiro em memória, sem tamanho documentado | ⚠️ verificar |
| `br_tse_eleicoes` | baixa ZIPs pra extrair data máxima | ZIPs nacionais por tipo de dado (candidatos, bens, prestação de contas), escopo Brasil inteiro | ⚠️ verificar — risco de passar de 5 GB pelo escopo nacional/histórico |
| `br_stf_corte_aberta` | Selenium dispara download automático | não documentado no código | ⚠️ verificar |
| `us_bls_qcew` | baixa ZIP trimestral | "multi-GB CSVs" no total; ZIP trimestral recente já passa de 200 MB | ⚠️ verificar — provável que o download por check (1 trimestre) fique < 5 GB, mas não confirmado |
| `us_fec_campaign_finance` | baixa arquivo de contribuições | atual ~2 GB comprimido; ciclo completo (`indiv20`) 5.6 GB | ❌ não — no limite/acima de 5 GB dependendo do arquivo baixado |
| `us_bea` | API JSON por série | não é bundle, é por série individual | ✅ sim → tratar como check leve |
| `us_fed_fred` | API JSON por série (`file_type=json`) | mesmo padrão do `us_bea` | ✅ sim → tratar como check leve |
| `world_cricsheet` | baixa bundle compactado | download ~114 MB (extração em disco chega a "several GB", mas isso é pós-download, não o próprio download) | ✅ sim → tratar como check leve (o download em si é pequeno) |
| `world_wb_wdi` | baixa arquivo | ~270 MB | ✅ sim → tratar como check leve |

### Novos achados nesta recontagem (2026-09-03)

Nome/função sugeria check leve, mas na prática baixa dado real antes de decidir se há novidade.

| Dataset | Técnica de check | Tamanho (evidência) | Reclassificação (< 5 GB?) |
|---|---|---|---|
| `au_abs_cpi` | baixa `.xlsx` por tabela | não documentado no código | ⚠️ verificar (provável pequeno — planilha por tabela) |
| `au_abs_labour_force` | baixa SDMX + Excel | ~38 MB | ✅ sim → tratar como check leve |
| `au_ato_taxation_statistics` | baixa dado real antes de decidir | ~92 MB | ✅ sim → tratar como check leve |
| `au_rba_statistical_tables` | baixa múltiplos CSVs individuais | não documentado, mas CSVs individuais | ✅ sim (provável) → tratar como check leve |
| `br_anp_precos_combustiveis` | baixa recorte de 4 semanas (`ultimas-4-semanas-*.csv`) | recorte pequeno, não histórico | ✅ sim → tratar como check leve |
| `br_bcb_taxa_cambio` | API JSON por intervalo de datas | pequeno | ✅ sim → tratar como check leve |
| `br_bcb_taxa_selic` | mesmo padrão de API JSON/CSV do `taxa_cambio` | pequeno | ✅ sim → tratar como check leve |
| `br_cgu_emendas_parlamentares` | baixa `emendas_parlamentares.zip` | não documentado no código | ⚠️ verificar |
| `br_cgu_servidores_executivo_federal` | ZIP assíncrono por subsistema/mês (README confirma) | não documentado | ⚠️ verificar |
| `br_sedec_desastres` | concatenação de 27 downloads (1 export CSV por UF) | individualmente pequeno | ✅ sim → tratar como check leve |
| `mx_sesnsp_incidencia_delictiva` | baixa dado real antes de decidir | "several hundred MB" | ✅ sim → tratar como check leve |
| `us_bls_cpi` | baixa dado real antes de decidir | "several hundred MB" | ✅ sim → tratar como check leve |

### Resumo pós-reclassificação

Aplicando o limiar de 5 GB: **18 dos 26** ficam com evidência de estarem abaixo do limite (13 confirmados + 5 prováveis) e passam a ser tratáveis como candidatos leves de migração, tão prioritários quanto a categoria "Padrão". Restam **8 realmente incertos ou genuinamente pesados**, que exigem confirmação (teste real ou inspeção manual do portal) antes de decidir a estratégia de migração: `br_anatel_telefonia_movel`, `br_cgu_beneficios_cidadao`, `br_cgu_cartao_pagamento`, `br_cgu_licitacao_contrato`, `br_cgu_sancoes`, `br_tse_eleicoes`, `br_stf_corte_aberta`, `us_fec_campaign_finance` (este último é o único com evidência concreta de passar dos 5 GB, não apenas incerto).

## Sem check — sempre roda, sem gate de novidade (10)

`br_bd_indicadores`, `br_bd_siga_o_dinheiro`, `br_cgu_pessoal_executivo_federal`, `br_cvm_administradores_carteira`, `br_cvm_oferta_publica_distribuicao`, `br_me_rais`, `br_me_siconfi` (poll não-bloqueante, sempre reconstrói), `br_poder360_pesquisas`, `br_senado_dados_abertos`, `fundacao_lemann`.

## Indeterminado (1)

`br_rj_isp_estatisticas_seguranca` — usa `get_count_lines`; não ficou claro se conta linhas de um arquivo já local ou baixa pra contar. Precisa inspeção manual antes de classificar.

## Inativos — sem `flows.py` (13)

`br_b3_cotacoes`, `br_mercadolivre_ofertas`, `br_mg_belohorizonte_smfa_iptu`, `br_mp_pep_cargos_funcoes`, `br_ons_avaliacao_operacao`, `br_ons_estimativa_custos`, `br_senado_dados_abertos_administrativos`, `br_sp_saopaulo_dieese_icv`, `br_tse_filiacao_partidaria`, `mundo_transfermarkt_competicoes`, `mundo_transfermarkt_competicoes_internacionais`, `world_sofascore_competicoes_futebol`, `world_wil_wid`.

## Candidatos sugeridos pra primeira migração

Baixo risco pra começar: dataset único (sem multi-tabela), check simples, sem particionamento.

- `br_ibge_ipca` / `br_ibge_ipca15` / `br_ibge_inpc` — os três usam a mesma API IBGE, então o `check_fn` é quase copiar-colar entre eles.
- `br_denatran_frota` — check via API do próprio backend BD, reaproveita bastante do padrão já usado no piloto.
- `us_sec_edgar` — check via API/listagem leve.

Depois de aplicar o limiar de 5 GB (ver seção acima), o universo de candidatos leves cresce bastante — os 18 `check_and_download` reclassificados também viram elegíveis, não só os 33 "Padrão" originais. Não competem entre si por complexidade de código (um `check_and_download` fundido é uma variante ainda não implementada, ver issue #1867), mas competem por prioridade de qual migrar primeiro.
