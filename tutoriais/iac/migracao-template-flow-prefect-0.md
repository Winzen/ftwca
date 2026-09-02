# Template atual de flows — arquitetura Prefect 0.15 na Base dos Dados

> Objetivo: documentar a estrutura atual dos flows para orientar a migração Prefect 0.15 → 3.
> Status: **apenas descritivo** — sem passos de migração. Complementa `Prefect 3 Pendencias.md`.

## Contexto

Estamos migrando do Prefect 0.15 para o Prefect 3 (ver `Prefect 3 Pendencias.md`, seções 5 e 3.4). Antes de tocar nos flows de dataset um a um, precisamos de um mapa escrito do que é nosso template atual — o que é framework, o que é cola por dataset, o que é reutilizável, o que é dívida técnica. Este documento é esse mapa. Ele serve de entrada para dois artefatos futuros:

1. Um playbook de migração por dataset (próximo passo, não aqui).
2. Uma biblioteca `pipelines.utils` para Prefect 3 cobrindo tudo que está hoje em `pipelines/utils/`.

Amostra analisada: `pipelines/utils/{custom,decorators,tasks,utils}.py`, `pipelines/utils/{metadata,execute_dbt_model,materialize_prod,dump_to_gcs}/`, e 5 datasets (`br_bcb_taxa_selic`, `br_camara_dados_abertos`, `br_anatel_banda_larga_fixa`, `br_cgu_beneficios_cidadao`, `br_ibge_inpc`).

---

## 1. O template canônico

Todo flow de dataset hoje segue o mesmo esqueleto de blocos:

```
[A] Abre flow       — with Flow(name="<dataset_id>.<table_id>", code_owners=[...]) as flow:
[B] Parâmetros      — dataset_id, table_id, materialize_after_dump, dbt_alias, update_metadata, target
[C] Renomeia run    — rename_current_flow_run_dataset_table(prefix="Dump: ", ...)
[C'] Verifica atualização — check_if_data_is_outdated(...) — pode ser ignorado, não é obrigatório
[D] Extrai + trata  — tasks específicas do dataset que produzem um arquivo local
                       → Saída: caminho de arquivo. Teste deve confirmar que o arquivo existe.
    [E] Upload dev  — create_table_dev_and_upload_to_gcs(...)
    [F] Materializa — case(materialize_after_dump, True):
            [E'] Upload prod  — create_table_prod_gcs_and_run_dbt(...)
            [F'] dbt run/test
            [V] update_django_metadata — case(update_metadata, True): ...
```

**Obrigatórios:** A, B, E.
**Convenção forte:** C (presente em todo dataset).
**Por dataset:** D — extração e tratamento específicos de cada fonte.
**Estrutura compartilhada, ativada por parâmetro:** F, V.

### Exemplo mínimo real (simplificado de `br_bcb_taxa_selic`)

```python
with Flow("br_bcb_taxa_selic.taxa_selic", code_owners=["lauris"]) as flow:
    dataset_id = Parameter("dataset_id", default="br_bcb_taxa_selic")
    table_id = Parameter("table_id", default="taxa_selic")
    materialize_after_dump = Parameter("materialize_after_dump", default=True)
    dbt_alias = Parameter("dbt_alias", default=False)
    update_metadata = Parameter("update_metadata", default=False)

    rename = rename_current_flow_run_dataset_table(prefix="Dump: ", dataset_id=dataset_id, table_id=table_id, wait=table_id)
    raw_path = get_data_taxa_selic(table_id=table_id, upstream_tasks=[rename])
    file_info = treat_data_taxa_selic(table_id=table_id, upstream_tasks=[raw_path])

    wait_dev = create_table_dev_and_upload_to_gcs(
        data_path=file_info["save_output_path"], dataset_id=dataset_id, table_id=table_id,
        dump_mode="append", upstream_tasks=[file_info],
    )
    wait_mat = run_dbt(dataset_id=dataset_id, table_id=table_id, dbt_command="run/test",
                      dbt_alias=dbt_alias, upstream_tasks=[wait_dev])

    with case(materialize_after_dump, True):
        wait_prod = create_table_prod_gcs_and_run_dbt(
            data_path=file_info["save_output_path"], dataset_id=dataset_id, table_id=table_id,
            dump_mode="append", upstream_tasks=[wait_mat],
        )
        with case(update_metadata, True):
            update_django_metadata(dataset_id=dataset_id, table_id=table_id, upstream_tasks=[wait_prod])

flow.storage = GCS(constants.GCS_FLOWS_BUCKET.value)
flow.run_config = KubernetesRun(image=constants.DOCKER_IMAGE.value)
flow.schedule = schedule_every_weekday_taxa_selic
```

---

## 2. Blocos obrigatórios

| Bloco | O que faz | Por que é obrigatório |
|---|---|---|
| `Flow(name="<dataset>.<table>", code_owners=[...])` | Subclasse `CustomFlow` (`utils/custom.py`), aliasada como `Flow` em `utils/decorators.py` | Liga `on_failure=notify_discord_on_failure` e roteamento de ownership |
| `Parameter("dataset_id" / "table_id")` | Dois parâmetros string | Todos os tasks downstream dependem deles — propagam para prefixo GCS, seletor de modelo dbt, lookup de metadata no Django |
| `create_table_dev_and_upload_to_gcs(...)` | Escreve parquet/CSV no bucket `basedosdados-dev`, cria tabela staging no BQ na primeira execução | Caminho único de upload usado por todos os datasets |
| `flow.storage = GCS(constants.GCS_FLOWS_BUCKET.value)` | Armazena o flow serializado no bucket `basedosdados-dev-prefect` | Exigido pelo agent do Prefect 0.15 — puxado do GCS em runtime |
| `flow.run_config = KubernetesRun(image=constants.DOCKER_IMAGE.value)` | Spec do pod Kubernetes para o flow | Indica ao agent qual imagem e labels usar |

---

## 3. Blocos opcionais (usados pela maioria, mas controlados por parâmetro)

| Bloco | Padrão | Controle |
|---|---|---|
| Gate `materialize_after_dump` em torno de `create_table_prod_gcs_and_run_dbt` | `True` | `Parameter("materialize_after_dump")` |
| Gate `update_metadata` em torno de `update_django_metadata` | `False` | `Parameter("update_metadata")` |
| `dbt_alias` | varia | `Parameter("dbt_alias")` — seleciona `models/<dataset>/<dataset>__<table>.sql` vs `<table>.sql` |
| `target` | `"dev"` | `Parameter("target")` — sobrescrito para `"prod"` em runs agendados |
| `flow.schedule` | `None` para flows utilitários | Anexado por dataset, geralmente por tabela |

---

## 4. Inventário do framework reutilizável — o que está em `pipelines/utils/`

Estes são os blocos de construção que **todo** dataset importa. Devem ser portados **uma vez** para o Prefect 3.

### 4.1 `utils/custom.py` — `CustomFlow`
Subclasse de `prefect.Flow`. Adiciona `code_owners: list[str]` e conecta `on_failure=partial(notify_discord_on_failure, secret_path=..., code_owners=code_owners)`. Aliasada como `Flow` em todos os datasets.

### 4.2 `utils/decorators.py`
Apenas `from pipelines.utils.custom import CustomFlow as Flow`. Uma linha. Todos os datasets importam de cá para consistência.

### 4.3 `utils/tasks.py` — biblioteca de tasks compartilhadas

| Task | Propósito | Ponto de contato GCP |
|---|---|---|
| `log_task` | Task de logging | — |
| `rename_current_flow_run` / `rename_current_flow_run_dataset_table` | Renomeia o run na UI para incluir dataset.table | — |
| `get_current_flow_labels` | Lê `prefect.context.config.cloud.agent.labels` | — |
| `get_credentials(secret_path)` | Leitura no Vault, retorna `(user, password)`. Decorada com `max_retries=5, retry_delay=10s, nout=2` | Vault |
| `create_table_dev_and_upload_to_gcs` | `bd.Table` + `bd.Storage` contra o bucket `basedosdados-dev` | GCS + BQ staging |
| `create_table_prod_gcs_and_run_dbt` | Igual ao acima mas contra `basedosdados`, depois executa `run_dbt(target="prod")` e `download_data_to_gcs` | GCS + BQ + dbt + exportação CSV |
| `run_dbt` | Invoca `dbtRunner` para `run` e/ou `test`, faz parse do log, sobe artefatos via `DBTArtifactUploader` | dbt-bigquery |
| `download_data_to_gcs` | BQ `extract_table` para CSV no GCS, trata políticas de acesso BD Pro (splits aberto/fechado, limites de 100MB/5GB) | BQ + GCS |
| `get_temporal_coverage` | Calcula string de cobertura no estilo BD `INÍCIO(INTERVALO)FIM` | — |
| `dump_header` | Escreve arquivo de cabeçalho de uma linha para criação inicial de tabela | — |

### 4.4 `utils/utils.py` — helpers Python (não são tasks Prefect)
- `get_vault_client()`, `get_vault_secret()`, `get_credentials_from_secret()` — cliente hvac via env vars `VAULT_ADDRESS` / `VAULT_TOKEN`.
- `log(msg, level)` — formata e escreve em `prefect.context.logger`.
- `notify_discord_on_failure(flow, state, secret_path, code_owners)` — state handler; **noop a não ser que `is_running_in_prod()`**.
- `notify_discord(...)` — envio manual de webhook.
- `is_running_in_prod()` — detecção de ambiente.
- `set_default_parameters(flow, defaults)` — seta `Parameter.default` após a definição do flow.
- `send_discord_message(...)`.
- `DBTArtifactUploader` — sobe `target/run_results.json`, `manifest.json`, etc. para o GCS.

### 4.5 `utils/metadata/` — integração com o backend Django
- `tasks.py::update_django_metadata` — entrypoint principal. Args: `dataset_id, table_id, date_column_name={"year":"ano","month":"mes"}, date_format, coverage_type ∈ {"part_bdpro","all_bdpro","all_free"}, time_delta={"months":6}, prefect_mode, api_mode, bq_project, historical_database`.
- `utils.py::extract_last_date_from_bq` — consulta BQ para `MAX(coluna_data)`.
- `utils.py::get_coverage_parameters` — constrói janelas de cobertura gratuita e Pro.
- `utils.py::create_update` — mutação GraphQL genérica contra `basedosdados.Backend(graphql_url=...)`.
- `utils.py::check_if_data_is_outdated`, `update_row_access_policy`.
- URLs de API em `pipelines/constants.py::API_URL` (`staging` e `prod`).

### 4.6 `utils/execute_dbt_model/`
Flow standalone "BD template: Executa DBT model". Encapsula `run_dbt` para que outros flows possam disparar um dbt run via chamada à API do Prefect. Contém também a versão Prefect 3 em andamento (decorada com `@flow`).

### 4.7 `utils/materialize_prod/`
Flow standalone "BD Utils: Transfere arquivos do bucket basedosdados-dev para basedosdados" — baixa pastas do bucket dev, sobe para prod, executa dbt. Usado para backfills e promoção manual para prod.

### 4.8 `utils/dump_to_gcs/`
"BD template: Ingerir tabela zipada para GCS" + variantes da task `download_data_to_gcs`. Usado por datasets que apenas precisam empurrar arquivos brutos para o GCS sem passar pela `bd.Table`.

### 4.9 `utils/to_download/`
Helper menor para download de arquivos de dados fora da lib bd.

---

## 5. Cola por dataset — o que vive em `datasets/<slug>/`

Layout padrão:

```
datasets/<dataset_slug>/
├── __init__.py
├── flows.py        # blocos [A][B][C][D][E][F][V] + anexo de schedule + storage/run_config
├── tasks.py        # funções @task para [D] — extração e tratamento específicos do dataset
├── schedules.py    # um ou mais Schedule(clocks=[CronClock|IntervalClock], filters=[], adjustments=[])
├── constants.py    # URLs, mapeamentos de colunas, URL da planilha de arquitetura
└── utils.py        # helpers Python puro usados pelos tasks
```

Variações observadas:

- **Flows por tabela**: `br_bcb_taxa_selic` define um flow por tabela (`flow` chamado `br_bcb_taxa_selic.taxa_selic`).
- **Multiplex com `deepcopy`**: `br_anatel_banda_larga_fixa` e `br_ibge_inpc` definem um flow base e depois fazem `deepcopy()` 3–4 vezes, rebindando `.name`, `.code_owners`, `.storage`, `.run_config`, `.schedule` para cada variante de tabela.
- **Sub-flows compostos**: `br_camara_dados_abertos` encadeia grupos de tasks via `dataset_id` compartilhado sem cascata de `case()` — usa o mesmo task `[E]` de upload mas a extração é dividida em múltiplos `@task`s.
- **Sem branch de materialização**: minoria pequena pula `[F]` inteiramente (raro; principalmente flows utilitários).

---

## 6. Retries, timeouts, logging, secrets — padrões

- **Retries de task**: `max_retries=5, retry_delay=timedelta(seconds=10)` é o padrão do framework (`constants.TASK_MAX_RETRIES`, `constants.TASK_RETRY_DELAY`). Tasks de dataset geralmente sobrescrevem para `retries=3, retry_delay_seconds=3`.
- **Específico do dbt**: `RUN_DBT_MODEL_MAX_RETRIES=1`, `WAIT_FOR_MATERIALIZATION_RETRY_ATTEMPTS=3`, `WAIT_FOR_MATERIALIZATION_RETRY_INTERVAL=5`.
- **Timeout de task**: não usado no nível de task. Apenas `requests.get(timeout=30|60)` dentro de tasks de extração. O pod K8s limita o resto.
- **Logging**: sempre `from pipelines.utils.utils import log; log(msg, level=...)`. Acesso direto ao `prefect.context.logger` existe mas é desencorajado.
- **Secrets**: apenas Vault. Nunca Prefect Secret blocks, nunca env vars no código (apenas o bootstrap `VAULT_ADDRESS` / `VAULT_TOKEN`). Credenciais de prod também são montadas como arquivos em `/credentials-prod/prod.json` pelo job spec do K8s — `run_dbt(target="prod")` lê esse arquivo diretamente.
- **Falha**: tratada exclusivamente pelo `notify_discord_on_failure` conectado pelo `CustomFlow`. Sem Slack, sem email, sem Sentry.

---

## 7. Schedules

Definidos como `Schedule(clocks=[CronClock(...)|IntervalClock(...)], filters=[...], adjustments=[...])` do Prefect 0.15. Anexados como `flow.schedule = ...` após o bloco `with Flow():`.

Convenções:
- `labels=[constants.BASEDOSDADOS_PROD_AGENT_LABEL.value]` (ou `DEV`). Um label por clock.
- `parameter_defaults={ "dataset_id": ..., "table_id": ..., "target": "prod", "materialize_after_dump": True, "update_metadata": True, "dbt_alias": True }`.
- `filters=[filters.is_weekday]` + `adjustments=[adjustments.next_weekday]` são comuns.
- Um dataset com N tabelas tipicamente tem N schedules em `schedules.py` — um por atribuição `flow.schedule = ...`.

---

## 8. Dependências — como os tasks são conectados

Três idiomas:

1. **`upstream_tasks=[...]`** — passado posicionalmente em toda chamada de task. É o padrão dominante.
2. **Dataflow por valor de retorno** — `file_info = treat_...(...); use(file_info["save_output_path"])`. Retornos em dict na maioria; `nout=N` para retornos em tupla (`get_credentials` usa `nout=2`).
3. **`case(parameter, value):`** — branches condicionais, usados exclusivamente para `materialize_after_dump` e `update_metadata`. `case` aninhado é o padrão para "atualiza metadata só se também materializou".

`Task.map(...)` existe em `dump_to_gcs` mas está comentado na maioria dos casos — fan-out é raro na prática.

---

## 9. Anti-padrões / dívida a limpar na migração

| Problema | Onde | Observação |
|---|---|---|
| `deepcopy(flow)` para multiplexar variantes de tabela | `br_anatel_*`, `br_ibge_*` | Frágil: todo atributo do flow (`storage`, `run_config`, `code_owners`, `schedule`) precisa ser reatribuído manualmente em cada cópia. O padrão nativo do Prefect 3 é um `@flow` + múltiplos objetos `Deployment`. |
| Nomes de bucket hardcodados `"basedosdados"` / `"basedosdados-dev"` | `utils/tasks.py` | Deveria ser uma constante derivada de env. |
| Caminho hardcodado `/credentials-prod/prod.json` | `run_dbt`, outros | Já documentado em Pendencias §3.2. Manter por ora; o `entrypoint.sh` escreve esse arquivo. |
| Código Prefect 0.x e Prefect 3 `@flow` coexistindo nos mesmos arquivos | `utils/execute_dbt_model/`, `br_bcb_taxa_selic/flows.py` | Aceitável durante a migração, mas fácil de importar o errado. |
| Typos em logs visíveis ao usuário | `utils/tasks.py::_upload_to_gcs` | Cosmético. |
| Acesso a `prefect.context.config.cloud.agent.labels` | `notify_discord_on_failure` | Não existe no Prefect 3 — precisa de equivalente. |
| `flow.storage = GCS(...)` / `flow.run_config = KubernetesRun(...)` | todo dataset | Desaparece inteiramente no Prefect 3 (job template do work pool substitui ambos). |
| `Parameter(...)` | todo dataset | Viram argumentos normais da função `@flow`. |
| `with case(param, True):` | todo dataset | Vira `if param:` simples no Prefect 3 — sem mais execução diferida. |

---

## 10. O que precisa virar lib/utilitário compartilhado no Prefect 3

O limite da migração é nítido. Tudo em `pipelines/utils/` precisa ser reimplementado em Prefect 3 idiomático **uma vez**, e então cada flow de dataset reescrito para consumir a nova API.

**Deve ser portado (framework, porta-se uma vez):**
1. Wrapper `Flow` com `code_owners` + hook de falha no Discord → hook de erro no nível do flow do Prefect 3 ou argumento `on_failure=[...]`.
2. Helper `log()` → usar `get_run_logger()` diretamente; manter `log()` como wrapper fino para compatibilidade de fonte.
3. Helpers do Vault (`get_vault_client`, `get_vault_secret`, `get_credentials_from_secret`) → inalterados, são Python puro.
4. `notify_discord_on_failure` / `notify_discord` → lógica inalterada; reconectar o ponto de disparo.
5. `is_running_in_prod()` → adaptar ao contexto de runtime do Prefect 3 (nome do work pool `basedosdados` vs `basedosdados-dev`, conforme Pendencias §2).
6. `create_table_dev_and_upload_to_gcs`, `create_table_prod_gcs_and_run_dbt`, `dump_header`, `download_data_to_gcs` → portados como `@task`s do Prefect 3, mesmas assinaturas.
7. `run_dbt` → já tem stub Prefect 3 em `utils/execute_dbt_model/`; consolidar.
8. `update_django_metadata` + helpers GraphQL → portados como `@task`.
9. `get_temporal_coverage`, `rename_current_flow_run_dataset_table`, `get_credentials`, `log_task` → ports como `@task`.
10. `DBTArtifactUploader` → inalterado.
11. `set_default_parameters` → remover (Prefect 3 tem defaults de parâmetro nativamente).
12. Schedules → movidos para `flow.deploy(schedule=...)` ou registrados via script de deploy (`.github/scripts/deploy_flows.py`).

**Deve ser reescrito por dataset (caso a caso):**
- `datasets/<slug>/flows.py` — substituir `with Flow():` + Parameters + `case()` + `upstream_tasks` por `@flow` + args de função + `if` + dataflow por valor de retorno.
- `datasets/<slug>/tasks.py` — decorator `@task` sobrevive praticamente intacto, mas acessos ao `prefect.context` precisam migrar para `get_run_context()` / `runtime`.
- `datasets/<slug>/schedules.py` — traduzidos para objetos de schedule do Prefect 3, registrados via script de deploy e não como atributo do flow.
- `datasets/<slug>/constants.py`, `utils.py` — Python puro, sem alterações.

---

## 11. Verificação (como validar este documento)

Este é um documento descritivo, não código. Validar por:

1. **Verificação pontual**: abrir um dataset que não está na amostra (ex: `br_cvm_fi`, `br_anp_precos_combustiveis`) e confirmar que o esqueleto de blocos bate.
2. **Cobertura de utils**: `grep -r "from pipelines.utils" pipelines/datasets/ | sort -u | wc -l` — todo import distinto aqui precisa estar coberto na §4. Se algo é importado mas não listado, este documento está incompleto.
3. **Cross-check com o piloto Prefect 3 funcionando**: comparar com `br_camara_dados_abertos.deputado` (validado end-to-end em 2026-05-14 conforme `Prefect 3 Pendencias.md`) — a reescrita feita lá é a referência de facto do que "idiomático Prefect 3" significa neste repositório.

---

## 12. Ideias para o futuro (não implementadas)

### 12.1 Flows filhos por formato de origem

Ideia: datasets com múltiplas fontes de dados (ex: HTML e JSON) poderiam ser estruturados com um flow pai que orquestra flows filhos independentes por formato.

```
bola.casa                      ← flow pai (orquestrador)
├── bola.casa_html             ← flow filho: extrai e trata fonte HTML
└── bola.casa_json             ← flow filho: extrai e trata fonte JSON
```

Cada flow filho é autônomo: tem seu próprio [D] de extração + tratamento, produz seu próprio arquivo e faz o upload. O flow pai apenas dispara os filhos e aguarda.

**Por que faz sentido:**
- Fontes diferentes têm cadências e falhas independentes — separar evita que a falha do JSON bloqueie o HTML.
- Cada flow filho pode ter seu próprio schedule, retry e `code_owners`.
- Facilita testar e deployar uma fonte sem tocar na outra.

**Status:** ideia conceitual. Não há implementação atual. A avaliar na migração para Prefect 3 (onde sub-flows são cidadãos de primeira classe via `flow` chamando outro `flow`).

---

### 12.2 Extract + treat = tratar só com SQL (sem Python)

Ideia: em vez de fazer o tratamento de dados no bloco [D] com Python (pandas, csv, etc.), fazer o tratamento inteiramente via SQL/dbt.

Fluxo simplificado:
```
[D] Extrai apenas   — baixa o arquivo bruto, sem transformação Python
[E] Upload raw      — sobe o arquivo bruto para staging no GCS/BQ
[F] dbt run         — todo o tratamento acontece no modelo SQL
```

**Por que faz sentido:**
- Elimina a camada Python de tratamento — menos código para manter por dataset.
- O SQL fica versionado no dbt, com testes de qualidade nativos.
- Funciona bem para fontes tabulares simples onde a transformação é essencialmente renomear colunas, filtrar linhas e converter tipos.

**Limitações:**
- Não funciona para fontes complexas (XML aninhado, PDFs, APIs paginadas que exigem lógica Python).
- Exige que o dado bruto seja ingestível diretamente no BQ (CSV, JSON, Parquet).

**Status:** ideia conceitual. Não há implementação atual. Candidatos naturais: datasets cujo `tasks.py` atual faz apenas `pd.read_csv` + renomear colunas.
