# Rebuild das Divergências BQ — 2026-04-28

Continuação do trabalho iniciado em 2026-04-27 (ver `Issue 1511 - Execução e Resultado.md`).

**Objetivo:** materializar as 39 tabelas únicas com divergência SQL↔BQ identificadas na validação 3-way, para que as 3 fontes fiquem alinhadas.

---

## Resumo de status

| Categoria | Tabelas | Status |
|---|---|---|
| ✅ Materializadas em dev | 32 | Concluído |
| ❌ Falha em dev (staging ausente) | 5 | Pendente — ver abaixo |
| ⏳ Incrementais (sem full-refresh) | 2 | Deixado para depois |
| ⚠️ Caso especial (renomeação de coluna) | 1 | Deixado para depois |
| **Prod (Prefect)** | 0 | Não iniciado |

---

## Dev — Executado via dbt local

**Comando base:**
```bash
cd /mnt/d/repositorios/bd/pipelines
uv run dbt run --select models/<dataset>/<modelo>.sql ...
```

**Target:** `dev` | **Threads:** 2 (padrão) | **Duração:** ~6 min

### ✅ 32 tabelas OK

| Dataset | Tabela |
|---|---|
| `br_cgu_beneficios_cidadao` | `bolsa_familia_pagamento` |
| `br_geobr_mapas` | `setor_censitario_2010` |
| `br_ibge_censo_2022` | `cadastro_enderecos` |
| `br_ibge_censo_demografico` | `microdados_domicilio_1970` |
| `br_ibge_censo_demografico` | `microdados_domicilio_1980` |
| `br_ibge_censo_demografico` | `microdados_domicilio_1991` |
| `br_ibge_censo_demografico` | `microdados_domicilio_2000` |
| `br_ibge_censo_demografico` | `microdados_domicilio_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_alfabetizacao_homens_mulheres_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_alfabetizacao_total_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_basico_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_domicilio_caracteristicas_gerais_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_domicilio_moradores_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_domicilio_renda_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_entorno_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_idade_homens_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_idade_mulheres_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_idade_total_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_pessoa_renda_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_raca_alfabetizacao_idade_genero_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_raca_idade_0_4_genero_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_raca_idade_genero_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_registro_civil_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_relacao_parentesco_conjuges_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_relacao_parentesco_filhos_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_relacao_parentesco_filhos_enteados_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_relacao_parentesco_outros_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_responsavel_domicilios_homens_total_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_responsavel_domicilios_mulheres_2010` |
| `br_ibge_censo_demografico` | `setor_censitario_responsavel_renda_2010` |
| `br_inep_indicadores_educacionais` | `uf_taxa_transicao` |
| `mundo_transfermarkt_competicoes_internacionais` | `champions_league` |

---

## Casos pendentes — não cobertos agora

### ❌ 5 tabelas — staging ausente em dev

**Erro:** `404 Not found: basedosdados-dev:br_ibge_censo_demografico_staging.microdados_pessoa_*`

Os modelos `microdados_pessoa_*` dependem de tabelas de staging que não existem no projeto `basedosdados-dev`. Provavelmente o pipeline de ingestão de dados brutos precisa rodar antes do modelo dbt.

| Tabela |
|---|
| `br_ibge_censo_demografico.microdados_pessoa_1970` |
| `br_ibge_censo_demografico.microdados_pessoa_1980` |
| `br_ibge_censo_demografico.microdados_pessoa_1991` |
| `br_ibge_censo_demografico.microdados_pessoa_2000` |
| `br_ibge_censo_demografico.microdados_pessoa_2010` |

**Ação recomendada:** verificar se o pipeline de staging existe e rodar antes, ou materializar direto em prod via Prefect.

---

### ⏳ 2 tabelas — materialization incremental

Sem `--full-refresh`, o dbt apenas faz INSERT/MERGE e não recria a estrutura da tabela — logo, partition e cluster não são aplicados.

| Dataset | Tabela | Motivo |
|---|---|---|
| `br_cvm_administradores_carteira` | `pessoa_fisica` | `materialized="incremental"` |
| `br_cvm_administradores_carteira` | `pessoa_juridica` | `materialized="incremental"` |

**Ação recomendada:** rodar com `--full-refresh` especificamente para essas duas tabelas quando apropriado.

---

### ⚠️ 1 tabela — renomeação de coluna

| Dataset | Tabela | Detalhe |
|---|---|---|
| `br_cgu_beneficios_cidadao` | `bolsa_familia_pagamento` | SQL usa `ano_referencia`/`mes_referencia` como partition/cluster, mas BQ tem `ano_competencia` como partition. Indica renomeação de coluna no modelo que não foi sincronizada com a tabela materializada. |

> **Nota:** a tabela foi re-materializada em dev com sucesso (1,5 bilhão de linhas, 108 GiB). Mas o alinhamento real entre SQL e BQ precisa ser verificado — o nome da coluna pode ter mudado intencionalmente no modelo.

**Ação recomendada:** revisar o SQL atual do modelo vs. o schema real da tabela no BQ e confirmar se `ano_referencia` existe ou se houve renomeação.

---

## Prod — Pendente

Os 32 modelos materializados em dev ainda precisam ser disparados em prod via Prefect (`trigger_dbt_model` com `target="prod"`). Não iniciado nesta sessão.
