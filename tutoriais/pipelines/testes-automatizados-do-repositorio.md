# Testes automatizados do repositório `pipelines`

**Data do levantamento:** 2026-08-12
**Relacionado:** [[Auditoria date_type — poll_source_for_update (Prefect 0 vs Prefect 3)]]

---

## Onde ficam e como rodar

Toda a suíte real (pytest) vive em `pipelines/utils/tests/`. Não há testes automatizados por dataset/crawler — só da camada de metadata (`pipelines/utils/metadata/*`), que é código compartilhado por todos os flows.

```bash
uv run pytest pipelines/utils/tests/metadata/ -q
```

Rodar a pasta `metadata/` inteira, não `pipelines/utils/tests/` inteira — tem um arquivo quebrado fora dela que aborta a coleta (ver seção "Problemas encontrados").

**Filosofia:** tudo roda sem rede e sem BigQuery real — via doubles (`FakeMetadataClient`, `FakeBQ`, `RecordingBackend`) definidos em `pipelines/utils/tests/metadata/conftest.py`. `RecordingBackend` substitui `bd.Backend`, roteirizando respostas por substring da query GraphQL; `FakeMetadataClient`/`FakeBQ` são doubles em memória que gravam as chamadas em ordem, usados pelos testes dos orquestradores (`register.py`, `poll.py`) para assertar exatamente quais entidades foram escritas (e quais **não** foram).

## Inventário por arquivo (134 testes coletáveis)

| Arquivo | Testes | Cobre |
|---|---:|---|
| `test_client.py` | 19 | `MetadataClient` (`client.py`) — camada de transporte GraphQL: contrato das mutations `upsert_*`, tratamento de resposta (nó ausente → `None`, `errors[]` → exceção), cache de auth, seleção de fonte por URL (multi-fonte), e o `get_coverage_max_date` novo |
| `test_register.py` | 29 | Orquestradores (`register.py`): `register_source_poll`, `poll_source_for_update` (incluindo os 2 modos de `compare_against` e a validação de `ValueError`), `commit_source_update`, `register_table_materialization`, poll por tamanho (bytes) |
| `test_poll.py` | 14 | O poll novo (`poll.py`, usado por CNES/CAGED/ANS): `register_source_coverage`, `check_source_is_ahead_of_table`, `sync_table_coverage` — inclui o teste que prova a mudança-chave (`Table.Update` grava a cobertura, não o `last_modified`) |
| `test_register_tasks.py` | 13 | Wrappers `@task` (`tasks.py`) — que os flows realmente importam. Confirma que constroem `MetadataClient`/`BigQueryReader` a partir de `env`/`bq_project` e delegam certo pras funções puras; inclui a regressão de `_coerce_to_date` (bug de produção: `source_max_date` como string vs `date`) |
| `test_domain.py` | 18 | Camada Domain (`domain.py`) — validação Pydantic dos `CoverageSpec` (`AllFree`, `AllBdpro`, `PartBdpro`, `NonHistorical`) |
| `test_dto.py` | 18 | Camada DTO (`dto.py`) — o que sai pelo fio nas mutations (ranges numéricos, UUIDs, ISO 8601) |
| `test_policy.py` | 18 | Camada Policy (`policy.py`) — regras de negócio puras: cálculo de ranges de cobertura, topologia de coverage, `should_update_raw_source` |
| `test_bq.py` | 5 | Adapter `BigQueryReader` (`bq.py`) — tradução domain→dict legado e delegação pras funções de `utils.py`, sem BQ real |

## Problemas encontrados durante o levantamento

### Arquivo de teste quebrado (aborta a coleta)

`pipelines/utils/tests/test_check_if_data_is_outdated_by_size_task.py` (fora da pasta `metadata/`) dá `ImportError` na coleta:

```
ImportError: cannot import name 'check_if_data_is_outdated_by_size' from 'pipelines.utils.metadata.tasks'
```

Referencia o nome antigo (Prefect 0) da task que hoje se chama `poll_source_size_for_update_task`. Ficou órfão de um rename e nunca foi atualizado nem removido — **isso faz `pytest pipelines/utils/tests/` (sem escopo) abortar a coleta inteira** em vez de só falhar esse arquivo. Rodar sempre com o escopo `pipelines/utils/tests/metadata/` evita o problema, mas vale corrigir ou remover esse arquivo depois.

### Teste flaky pré-existente (não relacionado ao nosso trabalho)

`test_policy.py::test_compute_part_bdpro_monthly_syncs_and_lags` falha com uma data "hoje" hardcoded que já passou de mês:

```
assert (2026, 1) == (2025, 12)
```

Confirmado que já falha igual na branch antes de qualquer mudança nossa (testado via `git stash`). Provavelmente um teste de "janela rolante" (`free_lag`) que assume uma data de execução fixa.

### Scripts em `tmp/` não são testes automatizados

`tmp/*.py` (22 arquivos, ex.: `test_ibge_pnadc.py`, `test_tse.py`) parecem testes pelo nome, mas são scripts manuais de uma era anterior (chamam `run_local`/`run_cloud` diretamente, referenciam flows Prefect 0 que talvez nem existam mais). Não rodam via pytest, não fazem parte de nenhum CI — são scratch files esquecidos no repo.

## O que adicionamos/ajustamos nesta sprint (`compare_against`, #1783)

- `test_client.py`: +3 testes de `get_coverage_max_date` (sem cobertura → `None`; escolhe o maior `end` entre múltiplas faixas; mês/dia ausentes default pra `1`).
- `test_register.py`: testes de `poll_source_for_update` reescritos/adicionados — default agora testa `"coverage"`; `compare_against="table_update"` explícito ganhou teste próprio; teste que reproduz o cenário exato do bug (`Table.Update` "adiantado" bloqueando, `Coverage` real detectando certo); teste do `ValueError` pra valor inválido (garante que nenhuma escrita ocorre antes da validação). Também corrigimos 2 testes de `register_source_poll` que dependiam do default antigo via fixture errada.

Suíte completa hoje: **133 passando, 1 falha pré-existente e não relacionada** (`test_compute_part_bdpro_monthly_syncs_and_lags`).
