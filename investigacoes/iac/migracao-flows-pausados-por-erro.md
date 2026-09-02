# Flows Pausados por Erro — Prefect 3
**Data:** 2026-06-15 (atualizado 2026-06-19) | **Total:** 43 flows

---

## 🔴 OOMKilled — aumentar limite de memória (1 flow)

| Flow                                             | Último erro |
| ------------------------------------------------ | ----------- |
| `br_me_rais__microdados_vinculos`                | 2026-05-25  |

**Causa:** Pod do Kubernetes morto pelo kernel OOM killer — container excedeu o limite de memória.
**Ação:** Aumentar `resources.limits.memory` no work pool ou no deployment.

> ✅ `br_anatel_telefonia_movel` (4 flows) — **resolvido em 2026-06-19** via PR pipelines#1581. `job_variables: {memory_limit: "8Gi"}` adicionado no `flows.py`. Schedules distribuídos por dias da semana para evitar concorrência.

---

## 🟠 `AttributeError: 'Task' object has no attribute 'run'` — bug conhecido (4 flows)

| Flow                                    | Último erro |
| --------------------------------------- | ----------- |
| `br_ibge_ipca__mes_brasil`              | 2026-06-13  |
| `br_ibge_ipca__mes_categoria_brasil`    | 2026-06-13  |
| `br_ibge_ipca__mes_categoria_municipio` | 2026-06-13  |
| `br_ibge_ipca__mes_categoria_rm`        | 2026-06-13  |

**Causa:** `pipelines/crawler/ibge_inflacao/utils.py` usa `.run(...)` em vez de `.fn(...)` — método do Prefect 0. Documentado como dívida técnica P0 #1.
**Ação:** Substituir `.run(...)` por `.fn(...)` nas linhas 378 e 387.

---

## 🟠 Mudança de schema na fonte (6 flows)

| Flow                                                         | Erro                                                                                                                      | Último erro |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------- | ----------- |
| `br_fgv_igp__igp_10_mes`                                     | `ValueError: reindex` — colunas não encontradas no DataFrame                                                              | 2026-05-25  |
| `br_fgv_igp__igp_di_mes`                                     | `ValueError: reindex` — colunas não encontradas no DataFrame                                                              | 2026-05-25  |
| `br_fgv_igp__igp_m_mes`                                      | `ValueError: reindex` — colunas não encontradas no DataFrame                                                              | 2026-05-25  |
| `br_fgv_igp__igp_og_ano`                                     | `KeyError: 'BIG THEME'` — coluna removida da fonte                                                                        | 2026-05-25  |
| `br_fgv_igp__igp_og_mes`                                     | `KeyError: 'BIG THEME'` — coluna removida da fonte                                                                        | 2026-05-25  |
| `br_rj_isp_estatisticas_seguranca__armas_apreendidas_mensal` | `KeyError: ['quantidade_artefato_explosivo_bomba_fabricacao_caseira', ...] not in index` — colunas removidas da fonte ISP | 2026-06-14  |

**Causa:** Fonte upstream mudou o schema (renomeou ou removeu colunas). O crawler não acompanhou.
**Ação:** Atualizar o crawler para o schema atual da fonte.

---

## 🟠 dbt tests falhando (10 flows)

| Flow | Erro | Último erro |
|---|---|---|
| `br_camara_dados_abertos__licitacao_contrato` | `dbt test falhou` | 2026-06-14 |
| `br_rf_cno__areas` | `dbt test falhou` | 2026-06-12 |
| `br_rf_cno__microdados` | `dbt test falhou` | 2026-06-15 |
| `br_rf_cno__vinculos` | `dbt test falhou` | 2026-06-15 |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_cisp` | `dbt test falhou` | 2026-06-14 |
| `br_ms_sinan__microdados_dengue` | `dbt TEST` — falha em `custom_dictionary_coverage` (múltiplos campos) | 2026-05-25 |
| `br_bcb_sicor__recurso_publico_propriedade` | `dbt test falhou` | 2026-06-19 |
| `br_bcb_sicor__operacoes_desclassificadas` | `dbt test falhou` | 2026-06-19 |
| `br_bcb_sicor__recurso_publico_mutuario` | `dbt test falhou` | 2026-06-19 |
| `br_bcb_sicor__recurso_publico_cooperado` | `dbt test falhou` | 2026-06-19 |

**Causa:** Dados baixados não passam nos testes de qualidade do dbt — provavelmente mudança de schema ou dados inválidos na fonte.
**Ação:** Investigar quais testes estão falhando e se é problema de fonte ou de modelo dbt.

---

## 🟡 API externa indisponível / erro de rede (6 flows)

| Flow                                            | Erro                                               | Último erro |
| ----------------------------------------------- | -------------------------------------------------- | ----------- |
| `br_bcb_taxa_selic__taxa_selic`                 | `HTTPError: 406 Not Acceptable` — `api.bcb.gov.br` | 2026-06-14  |
| `br_cnj_improbidade_administrativa__condenacao` | `httpx` transport error — falha de conexão         | 2026-06-08  |
| `br_ibge_pnadc__microdados`                     | `urllib3` connection error                         | 2026-05-25  |
| `br_poder360_pesquisas__microdados`             | `urllib3` connection error                         | 2026-06-15  |
| `br_stf_corte_aberta__decisoes`                 | `TimeoutException` (Selenium/browser)              | 2026-06-14  |
| `br_mp_pep__cargos_funcoes`                     | `Exception: Failed to get element with date`       | 2026-06-03  |

**Causa:** API da fonte retornou erro ou ficou inacessível no momento do run.
**Ação:** Reativar e monitorar — podem ser falhas transitórias. Se persistir, investigar mudança na API.

---

## 🟡 Arquivo/modelo não encontrado (2 flows)

| Flow | Erro | Último erro |
|---|---|---|
| `br_cgu_emendas_parlamentares__microdados` | `FileNotFoundError: /tmp/input/EmendasParlamentares.csv` — arquivo não baixado | 2026-06-14 |
| `br_tse_eleicoes__receitas_candidato` | `FileNotFoundError: models/br_tse_eleicoes/receitas_candidato.sql` — modelo dbt ausente | 2026-05-25 |

**Ação:**
- `emendas_parlamentares`: verificar se a URL de download mudou
- `tse_eleicoes__receitas_candidato`: criar/localizar o modelo dbt `receitas_candidato.sql`

---

## 🟡 Tabela/dataset não existe no BigQuery (3 flows)

| Flow                                                               | Erro                                             | Último erro |
| ------------------------------------------------------------------ | ------------------------------------------------ | ----------- |
| `br_tse_eleicoes__bens_candidato`                                  | `pandas_gbq` query error (tabela inexistente)    | 2026-05-25  |
| `br_tse_eleicoes__candidatos`                                      | `pandas_gbq` query error (tabela inexistente)    | 2026-05-25  |
| `fundacao_lemann__ano_escola_serie_educacao_aprendizagem_adequada` | `NotFound: Dataset basedosdados:fundacao_lemann` | 2026-05-25  |

**Ação:** Verificar se as tabelas/datasets existem no BQ e se o flow ainda é relevante.

---

## 🟡 Twitter API — deprecada (2 flows)

| Flow | Erro | Último erro |
|---|---|---|
| `br_bd_indicadores__twitter_metrics` | `403`: API v2 exige app vinculado a projeto Twitter Developer | 2026-05-25 |
| `br_bd_indicadores__twitter_metrics_agg` | `NotFound`: tabela `twitter_metrics_agg` não existe no BQ | 2026-05-25 |

**Causa:** Twitter/X mudou as regras de acesso à API v2.
**Ação:** Verificar se ainda faz sentido manter esses flows ou deprecar.

---

## 🟡 Variável de ambiente ausente (1 flow)

| Flow                              | Erro                                         | Último erro |
| --------------------------------- | -------------------------------------------- | ----------- |
| `br_bd_indicadores__website_user` | `KeyError: 'GOOGLE_APPLICATION_CREDENTIALS'` | 2026-05-25  |

**Ação:** Verificar se a env var está configurada no work pool / secret do K8s.

---

## 🟡 Erro de parsing na fonte (1 flow)

| Flow | Erro | Último erro |
|---|---|---|
| `br_ms_cnes__regra_contratual` | `ParserError: Expected 31 fields in line 213, saw 32` — CSV da fonte com campo extra | 2026-05-25 |

**Ação:** Fonte mudou o formato do CSV. Atualizar o parser para lidar com o campo extra.

---

## ⚪ Investigar (7 flows)

| Flow | Última informação nos logs | Último erro |
|---|---|---|
| `br_ms_cnes__equipamento` | Morreu durante wrangling de `PA/equipamento.csv` | 2026-06-14 |
| `br_ms_cnes__leito` | Morreu durante wrangling de `MT/leito.csv` | 2026-06-14 |
| `br_ms_cnes__dados_complementares` | `dbt run falhou para br_ms_cnes__dados_complementares.sql (target=dev)` | 2026-06-18 |
| `br_ms_sia__psicossocial` | Executou `dbt run` para psicossocial (provável falha no dbt) | 2026-06-14 |
| `br_sfb_sicar__area_imovel` | Última linha: leitura de data do estado AL (falhou após isso) | 2026-05-25 |
| `br_tse_eleicoes__despesas_candidato` | `No active or succeeded pods found` — pod nunca iniciou | 2026-05-25 |
| `br_ibge_pnadc__microdados` | `urllib3` connection error | 2026-05-25 |

**Ação:** Verificar manualmente no Prefect UI ou reativar para observar o próximo run.

---

## Resumo por categoria

| Categoria                  | Flows  | Ação                                                   |
| -------------------------- | ------ | ------------------------------------------------------ |
| OOMKilled                  | 1      | Aumentar limite de memória no K8s                      |
| `.run()` em vez de `.fn()` | 4      | Fix em `ibge_inflacao/utils.py` (dívida técnica P0 #1) |
| Schema da fonte mudou      | 6      | Atualizar crawlers                                     |
| dbt tests falhando         | 10     | Investigar testes + fonte                              |
| API externa com problema   | 6      | Reativar e monitorar                                   |
| Arquivo/modelo ausente     | 2      | Verificar URL / criar modelo dbt                       |
| Tabela BQ inexistente      | 3      | Verificar + decidir se depreca                         |
| Twitter API deprecada      | 2      | Deprecar ou reconfigurar                               |
| Env var ausente            | 1      | Configurar `GOOGLE_APPLICATION_CREDENTIALS`            |
| Parsing CSV                | 1      | Atualizar parser                                       |
| A investigar               | 7      | Verificar manualmente                                  |
| **Total**                  | **43** |                                                        |
