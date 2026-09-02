# download_data_to_gcs — Integração no run_dbt

**Status:** Implementado e testado em prod — PR mergeado em 2026-08-03
**PR:** [pipelines#1715](https://github.com/basedosdados/pipelines/pull/1715)
**Issue:** [pipelines#1684](https://github.com/basedosdados/pipelines/issues/1684)

---

## Problema

Na refatoração do Prefect 0 para Prefect 3, a task `download_data_to_gcs` foi reimplementada em `pipelines/utils/tasks.py` mas não foi integrada ao fluxo principal. Ela só era chamada em alguns flows que tinham a chamada explícita — os crawlers que usam `run_dbt` de `tasks.py` rodavam em prod sem fazer nenhum download para GCS.

---

## Solução

Adicionadas 4 linhas no `run_dbt` (`pipelines/utils/tasks.py`), após todos os comandos dbt terminarem com sucesso:

```python
if target == "prod" and table_id is not None and "run" in dbt_command:
    print(f"Exportando {dataset_id}.{table_id} para GCS")
    download_data_to_gcs.fn(dataset_id=dataset_id, table_id=table_id)
```

O `.fn` desencapsula o `@task` para permitir a chamada de dentro de outra task sem aninhamento de contexto Prefect — padrão idiomático do Prefect 3 para esse caso.

---

## Regras do download (função sem mudanças)

| Tamanho da tabela | Ação |
|---|---|
| > 1 GB | Sem download |
| 100 MB – 1 GB | Apenas BDPro |
| < 100 MB | Open + BDPro (se houver `bdpro_filter`) |

Para tabelas com row access policy `bdpro_filter`, a função:
1. Remove a policy temporariamente
2. Exporta os dados abertos
3. Restaura a policy
4. Exporta os dados BDPro

---

## O que não foi afetado

O flow `BD template: Executa DBT model` (`pipelines/utils/execute_dbt_model/flows.py`) usa seu próprio `run_dbt_task` local e já tinha download próprio via `download_data_to_gcs_task` + parâmetro `download_csv_file`. Não é afetado por essa mudança.

---

## Flows beneficiados

Todos os crawlers que usam `run_dbt` de `tasks.py` diretamente e não tinham chamada explícita ao download. Exemplos:
- `crawler/bcb`
- `crawler/camara_dados_abertos`
- `crawler/cgu`
- `crawler/cvm`
- `crawler/datasus`
- `crawler/tse_eleicoes`
- e outros

---

## Testes realizados

Ver [[test_dataset — Flows de Teste para download_data_to_gcs]].
