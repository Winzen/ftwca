# Análise de Prioridade dos Erros — Prefect 3
**Data:** 2026-06-15 (atualizado 2026-07-02)

---

## ✅ Resolvidos

**`br_anatel_telefonia_movel` × 4** — OOMKilled resolvido via `job_variables: {memory_limit: "8Gi"}`. Schedules distribuídos por dias da semana para evitar concorrência. PR pipelines#1581 mergeado em 2026-06-19.

**`br_cgu_pessoal_executivo_federal__terceirizados`** — `UnicodeDecodeError` resolvido adicionando `encoding='latin-1'` no `pd.read_csv`. PR pipelines#1592 mergeado em 2026-06-22.

**`br_ibge_inpc` × 4** — `flows.py` migrado de Prefect 0 para Prefect 3 seguindo o padrão do `br_ibge_ipca`. Todos os 4 flows testados manualmente. PR pipelines#1599 mergeado em 2026-06-25.

**`br_ibge_ipca` × 4** — crawler parado por duas causas combinadas: `IndexError` em bloco vazio da API do IBGE + `Update` de metadado gravado antes da materialização (causava trava permanente). Fix: parser resiliente + separação de poll/commit. PR pipelines#1600 mergeado em 2026-06-25.

**`br_anatel_telefonia_movel__densidade_brasil/municipio/uf`** — dois bugs combinados: poll eagerness (Update gravado antes da materialização travava runs seguintes) + data máxima calculada pelo arquivo inteiro em vez de por geografia. Fix: poll deferido + `get_max_date_in_table_microdados` por geografia + filtro de geografia robusto a acento/caixa. PR pipelines#1628 mergeado em 2026-07-01. Testado e validado em 2026-07-02.

**`br_ibge_pnadc__microdados`** — `CERTIFICATE_VERIFY_FAILED` ao conectar no FTP do IBGE. `certifi` estava pinado em `==2021.10.8`, sem a raiz Sectigo R46 (cadeia nova do ftp.ibge.gov.br). Fix: bump para `certifi>=2024.2.2`. PR pipelines#1623 mergeado em 2026-07-01. Requer rebuild da imagem Docker — já feito. Testado e validado em 2026-07-02.

---

## Mais fácil (< 1h cada)

~~**`br_bd_indicadores__website_user`**~~ — movido para "Opcional/Baixa prioridade" (ver abaixo).

~~**`br_cgu_pessoal_executivo_federal__terceirizados`**~~ — movido para Resolvidos.

**`br_bcb_taxa_cambio__taxa_cambio`** — `TypeError: '>=' not supported between instances of 'datetime.date' and 'float'`. Comparação de data com float no código — falhou em 3 dias consecutivos (20-22/jun). Pausado em 2026-06-23. Runs: [1](https://prefect3.basedosdados.org/runs/flow-run/019edf8a-6c14-7955-8cc5-08b6e182c707) [2](https://prefect3.basedosdados.org/runs/flow-run/019edc79-cc4d-7047-b49f-75e916c09012) [3](https://prefect3.basedosdados.org/runs/flow-run/019edc79-cc4d-7ace-89b6-27cdd525c96a)

**`br_ms_cnes__regra_contratual`** — `ParserError: Expected 31 fields, saw 32`. Uma linha: adicionar `on_bad_lines='skip'` ou `engine='python'` no `pd.read_csv`.

**`br_bcb_sicor__saldo`** — não é bug de código, é estado. O BCB corrigiu um arquivo retroativamente e o arquivo ficou menor. Precisa resetar o valor registrado no metadado para o tamanho atual. Nenhum código a mudar.

**APIs transitórias (1: cnj)** — pode já funcionar. Só reativar e disparar um run de teste. Se passar, pronto. Se não, investigar. *(pnadc resolvido via PR #1623 — ver Resolvidos)*

~~taxa_selic, poder360, stf, mp_pep~~ — movidos para "Opcional/Baixa prioridade" (ver abaixo).

---

## Médio (algumas horas por flow)

**OOMKilled (1: me_rais)** — `br_me_rais__microdados_vinculos`. Mesmo padrão dos anatel — adicionar `job_variables: {memory_limit: "8Gi"}` no `flows.py`. Descobrir quanto de memória precisa.

**Schema da fonte mudou — FGV IGP (5) e ISP armas (1)** — atualizar o mapeamento de colunas no crawler para o schema atual da fonte. Precisa bater na API/site e mapear as novas colunas.

**`br_cgu_emendas_parlamentares__microdados`** — arquivo não baixado. A URL provavelmente mudou. Precisa encontrar a nova URL no site do Portal da Transparência.

**dbt tests falhando (13 flows)** — precisa investigar flow por flow quais testes específicos estão quebrando e se é schema da fonte ou dado inválido. Pode ser rápido (renomear coluna) ou demorado (fonte com problema estrutural).

| Flow                                                     | Adicionado |
| -------------------------------------------------------- | ---------- |
| `br_camara_dados_abertos__licitacao_contrato`            | 2026-06-14 |
| `br_rf_cno__areas`                                       | 2026-06-12 |
| `br_rf_cno__microdados`                                  | 2026-06-15 |
| `br_rf_cno__vinculos`                                    | 2026-06-15 |
| `br_rj_isp_estatisticas_seguranca__evolucao_mensal_cisp` | 2026-06-14 |
| `br_ms_sinan__microdados_dengue`                         | 2026-05-25 |
| `br_bcb_estban__municipio`                               | 2026-06-15 |
| `br_bcb_sicor__operacao`                                 | 2026-06-15 |
| `br_bcb_sicor__recurso_publico_propriedade`              | 2026-06-19 |
| `br_bcb_sicor__operacoes_desclassificadas`               | 2026-06-19 |
| `br_bcb_sicor__recurso_publico_mutuario`                 | 2026-06-19 |
| `br_bcb_sicor__recurso_publico_cooperado`                | 2026-06-19 |
| `br_ms_cnes__dados_complementares`                       | 2026-06-18 |

**A investigar (6 flows)** — crashed sem erro claro. Reativar isolado e observar.

| Flow                                  | Última pista                                     |
| ------------------------------------- | ------------------------------------------------ |
| `br_ms_cnes__equipamento`             | Morreu durante wrangling de `PA/equipamento.csv` |
| `br_ms_cnes__leito`                   | Morreu durante wrangling de `MT/leito.csv`       |
| `br_ms_sia__psicossocial`             | Provável falha no dbt run                        |
| `br_sfb_sicar__area_imovel`           | Leitura de data do estado AL                     |
| `br_tse_eleicoes__despesas_candidato` | Pod nunca iniciou                                |
| `br_ms_sinan__microdados_dengue`      | Falha não investigada — pausado                  |

---

## Mais difícil / mais incerto

~~**`br_ibge_inpc` × 4 — `flows.py` Prefect 0**~~ — movido para Resolvidos.

**Tabelas BQ inexistentes (tse_eleicoes × 2, fundacao_lemann)** — não é só código. Precisa entender se as tabelas devem existir, por que sumiram, e se os flows ainda fazem sentido. Pode envolver decisão de produto.

**Twitter API deprecada (2 flows)** — o X mudou o modelo de acesso à API v2. Reativar provavelmente exige novo plano pago ou repensar a coleta por completo. Decisão estratégica antes de qualquer código.

---

## Resumo por ROI

| Grupo                                 | Flows | Esforço           | Status      |
| ------------------------------------- | ----- | ----------------- | ----------- |
| ~~anatel × 4~~                        | ~~4~~ | ~~meio dia~~      | ✅ Resolvido |
| ~~cgu_pessoal (encoding)~~            | ~~1~~ | ~~30 min~~        | ✅ Resolvido |
| ~~ibge_inpc flows.py~~                | ~~4~~ | ~~meio dia~~      | ✅ Resolvido |
| ~~ibge_ipca × 4~~                     | ~~4~~ | ~~algumas horas~~ | ✅ Resolvido |
| ~~anatel densidades × 3~~             | ~~3~~ | ~~algumas horas~~ | ✅ Resolvido |
| ~~ibge_pnadc__microdados~~            | ~~1~~ | ~~30 min~~        | ✅ Resolvido |
| Encoding/parser (ms_cnes__regra)      | 1     | 30 min            | 🔴 Pendente |
| Reset estado (sicor__saldo)           | 1     | 30 min            | 🔴 Pendente |
| TypeError date/float (taxa_cambio)    | 1     | 30 min            | 🔴 Pendente |
| APIs transitórias (1: cnj)            | até 1 | 1h (testar)       | 🔴 Pendente |
| OOM me_rais                           | 2     | 1-2h              | 🔴 Pendente |
| Schema fonte (fgv_igp × 5, isp armas) | 6     | 1-2 dias          | 🔴 Pendente |
| cgu_emendas (URL mudou)               | 1     | 1-2h              | 🔴 Pendente |
| dbt tests                             | 13    | variável          | 🔴 Pendente |
| A investigar                          | 6     | variável          | 🔴 Pendente |
| Twitter / BQ inexistente              | 5     | indefinido        | 🔴 Pendente |

Os mais baratos de resolver com maior retorno imediato continuam sendo os de encoding/parser — dois flows desbloqueados com menos de 30 min de trabalho total.

---

## Opcional / Baixa prioridade — não necessário no repositório

| Flow                                     | Motivo                                                     |
| ---------------------------------------- | ---------------------------------------------------------- |
| `br_bd_indicadores__website_user`        | Flow não é necessário no repositório. Pode ser depreciado. |
| `br_bd_indicadores__twitter_metrics`     | Twitter API deprecada — decisão estratégica pendente.      |
| `br_bd_indicadores__twitter_metrics_agg` | Idem acima.                                                |
| `taxa_selic`                             | Baixa prioridade — movido de APIs transitórias.            |
| `poder360`                               | Baixa prioridade — movido de APIs transitórias.            |
| `stf`                                    | Baixa prioridade — movido de APIs transitórias.            |
| `mp_pep`                                 | Baixa prioridade — movido de APIs transitórias.            |
