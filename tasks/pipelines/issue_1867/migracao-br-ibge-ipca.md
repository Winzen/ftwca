# Migração de dataset real — br_ibge_ipca (issue #1867, passo 3)

Primeira aplicação do `CheckThenDownloadPipeline` (ver [[encapsular-check-update-e-download]]) num dataset real, não sintético. Trabalho iniciado em 2026-09-05, branch `feat/event-pipeline-automations-poc`, ainda **não deployado nem testado contra infra real** — só implementado e verificado estaticamente.

## Por que br_ibge_ipca

Escolhido a partir do levantamento de candidatos ([[levantamento-datasets-por-categoria-de-check]]) como o mais simples da categoria "Padrão": check via API IBGE, achado inicialmente como single-table. Duas correções de percurso já na escolha:

1. **`br_denatran_frota` (candidato alternativo) na verdade tem 2 tabelas** (`uf_tipo`, `municipio_tipo`), não 1 como presumido no levantamento original.
2. **`br_ibge_ipca` na verdade tem 4 tabelas** (`mes_brasil`, `mes_categoria_brasil`, `mes_categoria_rm`, `mes_categoria_municipio`), não 1.

Ambos os erros reforçam um padrão já visto no levantamento: classificações feitas sem abrir o código real erram detalhes importantes. Regra prática: **sempre inspecionar o código de verdade antes de migrar um dataset**, não confiar só no levantamento de alto nível.

## Achado arquitetural: br_ibge_ipca na verdade é `check_and_download`

Inspecionando `pipelines/crawler/ibge_inflacao/flows.py::_run_ibge_inflacao`, a ordem real é:

```python
collect_data_utils(...)          # baixa da API do IBGE (SIDRA) de verdade
max_date = check_for_updates(...)  # lê o JSON que acabou de ser baixado
has_new_data = poll_source_for_update_task(...)
if not has_new_data: return
...
```

`check_for_updates`/`get_date_api` não faz uma chamada de metadata independente — ele **lê o arquivo local** que `collect_data_utils` escreveu (`tmp/json/.../data.json`). Ou seja, esse dataset é `check_and_download` na prática (baixa pra descobrir se há dado novo), não a variante padrão — **o levantamento de datasets classificou isso errado** (mesmo padrão de erro já visto lá: função com nome de "check leve" que na verdade depende de download prévio).

**Verificação de tamanho antes de decidir como seguir**: `collect_data`/`build_url` mostra que cada chamada busca **um único período** (`next_date_update()` sempre pede 1 mês), 1 agregado × poucas variáveis (4), geo_level no máximo `N6[all]` (~5.570 municípios) + `classificacao=315[all]` (~20 categorias) — poucos milhares de pontos numéricos em JSON, bem abaixo do limiar de 5 GB (ver critério em [[levantamento-datasets-por-categoria-de-check]]).

**Decisão** (usuário, 2026-09-05): já que o download é barato, `check_for_update` (etapa 1, pod separado) faz a busca leve de sempre; `download_data` (etapa 2, pod separado) **busca o mesmo período de novo** em vez de tentar repassar o JSON baixado entre pods (que não compartilham disco). "Tudo bem baixar duas vezes o arquivo" — mais simples que desenhar um jeito de passar o payload baixado entre estágios, e o custo extra é desprezível.

## Risco real descoberto: deploy podia sobrescrever produção

Ao planejar remover os 4 flows antigos (`_ipca_flow`/`br_ibge_ipca__mes_brasil` etc., com cron ativo e materialização real em `basedosdados.br_ibge_ipca.*`) do arquivo, o usuário perguntou diretamente: *"o que poderíamos fazer para [o deploy] não sobrescrever a versão antiga?"*

Investigação em `.github/scripts/deploy_flows.py` confirmou que a pergunta identificou um risco real, não hipotético:

```python
flow.from_source(
    source=GitRepository(url=REPO_URL, branch=branch_name),
    entrypoint=entrypoint,
).deploy(name=flow_name, work_pool_name=pool_name, ...)
```

`deploy_flows.py --files <arquivo>` redeploya **todo** `Flow` que encontra no arquivo escaneado (`load_flows_from_file`, via `vars(module).items()`), usando `branch_name` (o que vier em `--branch`) como fonte git do deployment. Se os 4 flows antigos continuassem no arquivo e rodássemos `deploy_flows.py --branch feat/event-pipeline-automations-poc --files pipelines/datasets/br_ibge_ipca/flows.py`, isso **sobrescreveria silenciosamente a fonte git dos 4 deployments reais de produção** (que hoje apontam pra `main`) pra apontar pra nossa branch de trabalho — sem aviso, sem confirmação, na primeira vez que rodássemos o deploy.

**Mitigação**: apagar os 4 flows antigos do arquivo (nesta branch) antes de deployar. Como `deploy_flows.py` só age sobre o que *encontra* no arquivo, removê-los da fonte significa que o deploy simplesmente não os toca — os deployments reais em produção continuam existindo exatamente como estão (fonte `main`, cron ativo) até uma decisão explícita e futura de merge/corte. Isso não é só "mais organizado" — é a mitigação de risco necessária.

## Restrição técnica: sem fábrica compartilhada de `@flow`

Tentativa inicial de generalizar as 4 tabelas com uma fábrica de flows (mesmo padrão do `_ipca_flow` antigo) esbarrou num problema: `CheckThenDownloadPipeline.download_deployment` é derivado de `.fn.__name__` (ver [[encapsular-check-update-e-download]], seção de refinamentos). Se os 4 `@flow` de download viessem de um template de closure compartilhado (`def download_flow(...)` definido uma vez dentro da fábrica), as 4 instâncias teriam **o mesmo `.fn.__name__` literal** (`"download_flow"`), mesmo sendo atribuídas a 4 variáveis de módulo diferentes — porque uma função não sabe a que nome de variável foi atribuída no escopo de quem chamou a fábrica.

**Solução**: fábrica só pra lógica que não tem essa restrição (`check_for_update`/`download_data` em `tasks.py`, e a construção do `CheckThenDownloadPipeline` em `flows.py`) — os 8 `@flow` (4 tabelas × check_update/download) são escritos explicitamente, cada um com `def` próprio, exatamente como já era feito pros 2 pilotos de teste em `test_dataset/flows.py`.

## Estrutura final

Seguindo a convenção pedida pelo usuário — "não quero nada fora os flows no flows.py", e que isso vire padrão pra qualquer dataset migrado que ainda não tenha os 3 arquivos:

- **`pipelines/datasets/br_ibge_ipca/constants.py`**: `DATASET_ID` + as 4 `*_TABLE_ID`.
- **`pipelines/datasets/br_ibge_ipca/tasks.py`**: `make_check_for_update(table_id)`/`make_download_data(table_id)` — fábricas parametrizadas (sem a restrição de nome único, já que os callables nunca são introspectados por `__name__`, só guardados como atributo de instância).
- **`pipelines/datasets/br_ibge_ipca/flows.py`**: só `_make_pipeline(table_id)` (constrói `CheckThenDownloadPipeline`) + os 8 `@flow` explícitos + `deploy_tags`/`download_deployment`. Nenhuma lógica de negócio.

Os 4 flows antigos (`_ipca_flow`, `br_ibge_ipca__mes_brasil` etc.) foram **completamente removidos** deste arquivo nesta branch — não coexistem mais com os novos. `crawler/ibge_inflacao/flows.py::_run_ibge_inflacao` continua existindo (ainda usado por `br_ibge_ipca15`/`br_ibge_inpc`, não migrados).

Validação estática feita (sem deploy real ainda): os 8 flows são encontrados no arquivo, os 4 nomes antigos confirmadamente não existem mais nele, os 4 `download_deployment` são únicos, e o dispatch (`deployment_name()`) resolve exatamente pro `@flow(name=...)` real de cada tabela. `ruff`/`pyrefly` limpos.

## Verificação no BigQuery real (2026-09-05)

Antes de qualquer disparo real, conferido o estado atual de `basedosdados.br_ibge_ipca.*`:

| Tabela | `ano`/`mes` (schema final) | Cobertura mais recente | Linhas no último mês |
|---|---|---|---|
| `mes_brasil` | `INT64` (CSV intermediário usa string, cast já feito pelo dbt) | 2026-06 | 1 |
| `mes_categoria_brasil` | idem | 2026-06 | 457 |
| `mes_categoria_rm` | idem | **2026-07** | 4.570 |
| `mes_categoria_municipio` | idem | 2026-06 | 4.113 |

Achado: `mes_categoria_rm` está um mês à frente das outras 3 — inconsistência real de produção (cron de cada tabela roda em horário diferente, mesmo dia). Consequência pro teste real ainda não feito: pra `mes_brasil`/`mes_categoria_brasil`/`mes_categoria_municipio`, o próximo período seria julho/2026 — bem provável já publicado pelo IBGE (lag típico ~5-6 semanas), ou seja, **disparar o `check_update` dessas 3 tem chance real de cascatear até materializar em produção de verdade** (mesmo destino que o flow antigo já escreve todo mês — não é dado espúrio, só antecipa o que o cron faria de qualquer forma). Pra `mes_categoria_rm`, o próximo período (agosto/2026) provavelmente ainda não foi publicado.

## Status (fim do dia, 2026-09-05)

Implementado e validado estaticamente. **Nada deployado no Prefect ainda, nada commitado.** Próximos passos, retomar amanhã:

1. Decidir se testamos com deploy real (sem schedule, disparo manual) — dado o achado acima, ciente de que 3 das 4 tabelas provavelmente cascateiam até prod de verdade se disparadas.
2. Se/quando validado: commit + push (autorização explícita necessária, por regra já estabelecida na sessão).
3. Decisão futura, não urgente: quando/como fazer o corte de verdade (pausar os 4 deployments antigos reais, ou deixá-los coexistir até confirmação total).
