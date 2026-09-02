# update_table_metadata_task — Investigação das 22 tabelas NULL

**Data:** 2026-07-22
**Contexto:** Após o sweep completo (`ce55a8ee`) que rodou depois do merge do PR #1042, 840 de 862 tabelas elegíveis foram preenchidas. 22 permaneceram com `number_rows`, `number_columns` e `uncompressed_file_size` em NULL.

---

## Método

A investigação foi feita direto no pod `api-prod-99df7f585-bn29q` (namespace `website`).

**Passo 1 — Confirmar que todas têm `cloud_tables` configurado**

```python
Table.objects.filter(...).values_list('cloud_tables__gcp_project_id', 'cloud_tables__gcp_dataset_id', 'cloud_tables__gcp_table_id')
```

Todas as 22 tinham `cloud_tables` com os três campos preenchidos. O `gbq_slug` não seria o problema.

**Passo 2 — Checar existência no BigQuery**

Rodamos `bq_client.get_table(table.gbq_slug)` para cada uma e capturamos o resultado:

```python
try:
    bq_table = bq_client.get_table(gbq_slug)
    print(f'[OK] type={bq_table.table_type} rows={bq_table.num_rows} bytes={bq_table.num_bytes}')
except GoogleAPICallError as exc:
    print(f'[BQ ERROR] {msg}')
```

**Passo 3 — Verificar se o `save()` foi chamado no sweep anterior**

Checamos `number_columns` e `updated_at` de cada tabela. Todas tinham `number_columns=None` e `updated_at` anterior a 2026-07-22, confirmando que o `table.save()` nunca foi chamado para nenhuma delas durante o `ce55a8ee`.

**Passo 4 — Re-rodar a task só para as 22**

```python
update_table_metadata_task.call_local(pks)
```

---

## Resultado da investigação

As 22 tabelas se dividem em dois grupos com causas distintas.

---

### Grupo 1 — 16 tabelas: válidas no BQ, não salvas no sweep

O BigQuery retornou dados completos para essas tabelas (`num_rows > 0`, `num_bytes > 0`). Após re-rodar a task com os PKs delas, todas foram salvas com sucesso.

**Causa provável:** O sweep `ce55a8ee` acumulou erros suficientes de outras tabelas para preencher o `Messenger` (soft limit: 1600 chars). Quando o `messenger.is_full` retorna `True`, o loop quebra e as tabelas restantes não são processadas. Como essas 16 ficaram na fila depois dos erros, nunca foram alcançadas.

Tabelas afetadas (agora corrigidas):

| Tabela |
|---|
| `br_inep_avaliacao_alfabetizacao.br_inep_avaliacao_alfabetizacao__alunos` |
| `br_mma_sinpatinhas.microdados` |
| `operacoes_contratadas.operacoes_indiretas_automaticas` |
| `operacoes_contratadas.operacoes_nao_automaticas` |
| `siconfi.brasil_despesas_funcao` |
| `siconfi.brasil_despesas_orcamentarias` |
| `siconfi.brasil_execucao_restos_pagar` |
| `siconfi.brasil_execucao_restos_pagar_funcao` |
| `siconfi.brasil_receitas_orcamentarias` |
| `siconfi.brasil_variacoes_patrimoniais` |
| `siconfi.municipio_execucao_restos_pagar` |
| `siconfi.municipio_execucao_restos_pagar_funcao` |
| `siconfi.municipio_variacoes_patrimoniais` |
| `siconfi.uf_execucao_restos_pagar` |
| `siconfi.uf_execucao_restos_pagar_funcao` |
| `siconfi.uf_variacoes_patrimoniais` |

---

### Grupo 2 — 6 tabelas: não existem no BigQuery

O BQ retornou `Not found` para todas. Não é um bug da task — são tabelas cadastradas no backend mas cujos dados ainda não foram publicados no BQ.

| Tabela no backend | Dataset/tabela BQ tentado |
|---|---|
| `sicor.operacoes_desclassificadas` | `basedosdados.br_bcb_sicor.operacoes_desclassificadas` |
| `sage.dicionario` | `basedosdados.world_dasanaike_sage.dicionario` |
| `sage.election_returns` | `basedosdados.world_dasanaike_sage.election_returns` |
| `sage.germany_erststimme` | `basedosdados.world_dasanaike_sage.germany_erststimme` |
| `sage.netherlands_preference_votes` | `basedosdados.world_dasanaike_sage.netherlands_preference_votes` |
| `sage.spatial_admin_crosswalk` | `basedosdados.world_dasanaike_sage.spatial_admin_crosswalk` |

Essas 6 vão continuar com NULL até que os dados sejam carregados no BigQuery. A task vai logar `WARNING: Not found` para elas a cada execução.

---

## Estado final

| Situação | Quantidade |
|---|---|
| Preenchidas pelo sweep `ce55a8ee` | 840 |
| Preenchidas pela re-execução manual (2026-07-22) | 16 |
| Ainda NULL — dados ausentes no BQ | 6 |
| **Total elegível** | **862** |

---

## Próximos passos sugeridos

- **Sage:** Verificar com o time de dados se o dataset `world_dasanaike_sage` será publicado ou se as tabelas devem ser marcadas como `under_review`/`excluded` no backend.
- **Sicor:** Mesmo questionamento para `br_bcb_sicor.operacoes_desclassificadas`.
- **Task:** O comportamento de `messenger.is_full` que quebra o loop pode fazer com que tabelas válidas fiquem sem atualização se houver muitos erros num sweep. A refatoração event-driven (backlog) resolveria isso estruturalmente.
