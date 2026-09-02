# Situação da Migração — Prefect 3
**Data:** 2026-06-15 | Atualiza: [Situação da Migração (2026-06-12)](Situação%20da%20Migração%20(2026-06-12).md)

---

## Ações realizadas hoje

### PR #1578 — mergeado
Corrigiu 4 bugs identificados na comparação com o backup:

| Fix | Detalhe |
|---|---|
| `.run()` → `.fn()` em `ibge_inflacao/utils.py` | Corrigiu `br_ibge_ipca` × 4 |
| `billing_project_id` em `tasks.py` (`_upload_to_gcs`) | Corrigiu 403 em uploads GCS para buckets requester-pays |
| Credencial por `target` em `gcs.py` (`DBTArtifactUploader`) | Corrigiu 403 ao subir artefatos dbt no bucket dev |
| `schema_validator.py` migrado para Pydantic v2 nativo | Corrigiu `ImportError` no `br_bcb_sicor` |

### Prefect 0 desligado
- 3 schedules ativos desativados via API
- 200+ runs `Scheduled` pendentes cancelados (toggle + cancel direto)
- `br_cvm_administradores_carteira` e `br_cvm_oferta_publica_distribuicao`: runs cancelados manualmente

### Grupo A reativado
Schedules ativados manualmente no Prefect 3 após merge do PR:

| Dataset | Resultado |
|---|---|
| `br_bcb_estban__agencia` | ✅ Saudável (testado) |
| `br_bcb_estban__municipio` | ⛔ Pausado — erro de dbt (ver abaixo) |
| `br_bcb_sicor` × 6 | ✅ Saudáveis (empreendimento, liberacao, recurso_publico_complemento_operacao, recurso_publico_cooperado, recurso_publico_gleba, dicionario) |
| `br_bcb_sicor` × 5 | ⛔ Pausados — erros (ver abaixo) |
| `br_cgu_beneficios_cidadao` × 3 | ✅ Saudáveis (testados) |
| `br_cgu_cartao_pagamento` × 3 | ✅ Saudáveis (testados) |
| `br_cgu_licitacao_contrato` × 7 | ✅ Saudáveis (testados) |
| `br_cgu_servidores_executivo_federal` × 7 | ✅ Saudáveis |
| `br_cgu_emendas_parlamentares__microdados` | ⛔ Mantido pausado — `FileNotFoundError` |
| `br_cgu_pessoal_executivo_federal__terceirizados` | ⛔ Mantido pausado — `UnicodeDecodeError` |

### `br_ibge_ipca` × 4 — corrigidos e validados
Todos Completed após o merge. Fix do `.run()` → `.fn()` confirmado em produção.

### `br_ibge_inpc` × 4 — reativados
Flows estavam pausados desnecessariamente (último run era Completed). Reativados. Crashes de `git clone exit 128` observados durante os testes — falha transitória de rede, não bug de código.

---

## Novos erros identificados nos testes

### `br_bcb_estban__municipio`
**Erro:** `Unable to validate date: 2025-09-01 of type <class 'datetime.date'>` — dbt run falhou
**Estado:** Pausado
**Ação:** Investigar validação de data no modelo dbt ou no código do crawler

---

### `br_bcb_sicor__saldo`
**Erro:** `Tamanho na fonte (11932794) é MENOR que o último registrado (11933858)` — o BCB atualizou/corrigiu o arquivo retroativamente
**Estado:** Pausado
**Ação:** Resetar o valor registrado manualmente para permitir o próximo run, ou aguardar a fonte crescer novamente

---

### `br_bcb_sicor__operacao`
**Erro:** dbt run falhou (detalhes nos logs do run `divergent-hippo`)
**Estado:** Pausado
**Ação:** Investigar o modelo `models/br_bcb_sicor/br_bcb_sicor__operacao.sql`

---

### `br_bcb_sicor__operacoes_desclassificadas`, `__recurso_publico_mutuario`, `__recurso_publico_propriedade`
**Erro:** dbt test falhou (3 flows)
**Estado:** Pausados
**Ação:** Investigar quais testes estão falhando — provavelmente mudança de schema ou dados inválidos na fonte BCB

---

## Estado atual — flows pausados (44)

> Não inclui flows de teste (`hello-prefect3-prod`, `hello_prefect3`).

### OOMKilled — aumentar limite de memória (5)
| Flow |
|---|
| `br_anatel_telefonia_movel__densidade_brasil` |
| `br_anatel_telefonia_movel__densidade_municipio` |
| `br_anatel_telefonia_movel__densidade_uf` |
| `br_anatel_telefonia_movel__microdados` |
| `br_me_rais__microdados_vinculos` |

### dbt test falhando (9)
| Flow | Origem |
|---|---|
| `br_bcb_sicor__operacoes_desclassificadas` | Novo — identificado hoje |
| `br_bcb_sicor__recurso_publico_mutuario` | Novo — identificado hoje |
| `br_bcb_sicor__recurso_publico_propriedade` | Novo — identificado hoje |
| `br_camara_dados_abertos__licitacao_contrato` | Pré-existente |
| `br_rf_cno__areas` | Pré-existente |
| `br_rf_cno__microdados` | Pré-existente |
| `br_rf_cno__vinculos` | Pré-existente |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_cisp` | Pré-existente |
| `br_ms_sinan__microdados_dengue` | Pré-existente |

### dbt run falhando (2)
| Flow | Erro |
|---|---|
| `br_bcb_estban__municipio` | `Unable to validate date: 2025-09-01` — novo |
| `br_bcb_sicor__operacao` | dbt run falhou — novo |

### Validação de tamanho (1)
| Flow | Erro |
|---|---|
| `br_bcb_sicor__saldo` | Fonte menor que último registrado — BCB corrigiu arquivo retroativamente |

### Schema da fonte mudou (6)
| Flow |
|---|
| `br_fgv_igp__igp_10_mes` |
| `br_fgv_igp__igp_di_mes` |
| `br_fgv_igp__igp_m_mes` |
| `br_fgv_igp__igp_og_ano` |
| `br_fgv_igp__igp_og_mes` |
| `br_rj_isp_estatisticas_seguranca__armas_apreendidas_mensal` |

### API externa com problema (6)
| Flow | Erro |
|---|---|
| `br_bcb_taxa_selic__taxa_selic` | `HTTPError: 406 Not Acceptable` |
| `br_cnj_improbidade_administrativa__condenacao` | `httpx` transport error |
| `br_ibge_pnadc__microdados` | `urllib3` connection error |
| `br_poder360_pesquisas__microdados` | `urllib3` connection error |
| `br_stf_corte_aberta__decisoes` | `TimeoutException` (Selenium) |
| `br_mp_pep__cargos_funcoes` | `Failed to get element with date` |

### Arquivo / modelo ausente (2)
| Flow | Erro |
|---|---|
| `br_cgu_emendas_parlamentares__microdados` | `FileNotFoundError: EmendasParlamentares.csv` |
| `br_tse_eleicoes__receitas_candidato` | `FileNotFoundError: receitas_candidato.sql` |

### Tabela BQ inexistente (3)
| Flow |
|---|
| `br_tse_eleicoes__bens_candidato` |
| `br_tse_eleicoes__candidatos` |
| `fundacao_lemann__ano_escola_serie_educacao_aprendizagem_adequada` |

### Twitter API deprecada (2)
| Flow |
|---|
| `br_bd_indicadores__twitter_metrics` |
| `br_bd_indicadores__twitter_metrics_agg` |

### Env var ausente (1)
| Flow | Erro |
|---|---|
| `br_bd_indicadores__website_user` | `KeyError: 'GOOGLE_APPLICATION_CREDENTIALS'` |

### Parsing CSV (1)
| Flow | Erro |
|---|---|
| `br_ms_cnes__regra_contratual` | `ParserError: Expected 31 fields, saw 32` |

### Pod não iniciou / OOM eviction (2)
| Flow | Erro |
|---|---|
| `br_tse_eleicoes__despesas_candidato` | Pod evicted — node sem memória |
| `br_bcb_sicor__operacao` | Pod unschedulable + dbt run falhou |

### A investigar (4)
| Flow | Última informação |
|---|---|
| `br_ms_cnes__equipamento` | Morreu durante wrangling de `PA/equipamento.csv` |
| `br_ms_cnes__leito` | Morreu durante wrangling de `MT/leito.csv` |
| `br_ms_sia__psicossocial` | Executou dbt run (provável falha no dbt) |
| `br_sfb_sicar__area_imovel` | Falhou após leitura de dados do estado AL |

### Migração pendente (4)
| Flow | O que falta |
|---|---|
| `br_cgu_emendas_parlamentares__microdados` | URL de download mudou |
| `br_cgu_pessoal_executivo_federal__terceirizados` | `UnicodeDecodeError` ao ler CSV |
| `br_ibge_inpc` × 4 | `flows.py` ainda é Prefect 0 — reativados provisoriamente, git clone transitório |

---

## Próximos passos sugeridos

| Prioridade | Ação |
|---|---|
| P0 | Investigar dbt test em `br_bcb_sicor` × 3 (schema BCB mudou?) |
| P0 | Investigar `br_bcb_estban__municipio` — erro de validação de data no dbt |
| P0 | Criar `flows.py` Prefect 3 para `br_ibge_inpc` |
| P1 | Aumentar limite de memória para `br_anatel_telefonia_movel` × 4 e `br_me_rais` |
| P1 | Resetar valor registrado do `br_bcb_sicor__saldo` |
| P1 | Investigar APIs externas (taxa_selic 406, cnj, pnadc, poder360, stf, mp_pep) |
| P2 | Verificar schema atualizado das fontes FGV IGP e ISP |
| P2 | Criar modelo `receitas_candidato.sql` para TSE |
| P2 | Decidir o que fazer com Twitter API (deprecada) |
| P3 | Verificar tabelas BQ (tse_eleicoes × 2, fundacao_lemann) |
