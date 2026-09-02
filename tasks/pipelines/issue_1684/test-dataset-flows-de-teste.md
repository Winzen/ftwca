# test_dataset — Flows de Teste para download_data_to_gcs

**Testado em:** 2026-08-03
**PR:** [pipelines#1715](https://github.com/basedosdados/pipelines/pull/1715)
**Relacionado:** [[download_data_to_gcs — Integração no run_dbt]]

---

## Propósito

Verificar as 4 condições de tamanho da `download_data_to_gcs` sem depender de dados reais de internet. Os flows materializam tabelas sintéticas no BigQuery via dbt e disparam o export automaticamente pelo `run_dbt(target="prod")`.

---

## Localização no repositório

```
pipelines/datasets/test_dataset/
  flows.py     # 4 flows individuais + 1 suite
  tasks.py     # vazio

models/test_dataset/
  test_dataset__tabela_pequena.sql          # < 100 MB, sem row access policy
  test_dataset__tabela_pequena_bdpro.sql    # < 100 MB, com bdpro_filter (post-hook)
  test_dataset__tabela_media.sql            # ~200 MB — faixa 100 MB-1 GB
  test_dataset__tabela_grande.sql           # ~1,1 GB — acima de 1 GB
```

---

## Modelos dbt

Os dados são gerados com `FARM_FINGERPRINT` — valores INT64 pseudo-aleatórios resistentes à compressão do BigQuery. Tabelas médias e grandes usam `CROSS JOIN` de dois arrays para evitar `GENERATE_ARRAY` com dezenas de milhões de elementos (BigQuery pode falhar).

| Modelo | Tamanho gerado | Método |
|---|---|---|
| `tabela_pequena` | ~48 KB (1 000 linhas × 6 colunas INT64) | `GENERATE_ARRAY(1, 1000)` |
| `tabela_pequena_bdpro` | ~48 KB | igual + post-hooks adicionam `allusers_filter` e `bdpro_filter` |
| `tabela_media` | ~200 MB (2 121 × 2 121 linhas) | CROSS JOIN dois arrays de 2 121 |
| `tabela_grande` | ~1,1 GB (4 900 × 4 900 linhas) | CROSS JOIN dois arrays de 4 900 |

`tabela_pequena_bdpro` simula o estado real de tabelas BDPro: as duas row access policies precisam coexistir (`allusers_filter` + `bdpro_filter`). Remover só a `bdpro_filter` sem a `allusers_filter` presente causa erro no BigQuery ("dropping the last row access policy would make the table accessible to all users").

---

## Flows disponíveis

| Flow (nome Python) | Condição testada | Work pool |
|---|---|---|
| `test_download_data_to_gcs_open_only_flow` | < 100 MB, sem bdpro → exporta só open | `basedosdados` (prod) |
| `test_download_data_to_gcs_open_and_bdpro_flow` | < 100 MB, com bdpro → exporta open + BDPro | `basedosdados` (prod) |
| `test_download_data_to_gcs_bdpro_only_flow` | 100 MB-1 GB → exporta só BDPro | `basedosdados` (prod) |
| `test_download_data_to_gcs_skip_large_flow` | > 1 GB → sem export | `basedosdados` (prod) |
| `test_download_data_to_gcs_all_cases_flow` | todos os 4 casos em sequência | `basedosdados` (prod) |

A suite roda os 4 flows sequencialmente (não em paralelo) para não esgotar a RAM do pod.

---

## Resultados dos testes (2026-08-03)

Verificada presença **e ausência** dos arquivos no GCS após cada flow:

| Condição | Arquivo open (`/open/`) | Arquivo BDPro (`/closed/`) |
|---|---|---|
| < 100 MB sem bdpro | ✅ presente | ✅ ausente |
| < 100 MB com bdpro | ✅ presente | ✅ presente |
| 100 MB-1 GB | ✅ ausente | ✅ presente |
| > 1 GB | ✅ ausente | ✅ ausente |

Suite completa (`test_download_data_to_gcs_all_cases_flow`) também executada com sucesso:
https://prefect3.basedosdados.org/v2/runs/flow-run/3f277aa5-3f26-47ef-8419-31333ab671df?tab=Logs

---

## Como rodar novamente

Os modelos `tabela_media` e `tabela_grande` são `materialized="incremental"` — na segunda execução o dbt percebe que a tabela já existe e faz no-op (sem reprocessar tudo). Para forçar recriação usar `--full-refresh` via o parâmetro `flags` do `run_dbt`.

Para disparar manualmente via Prefect UI: acessar o deployment do flow desejado no work pool `basedosdados` e criar uma flow run.
