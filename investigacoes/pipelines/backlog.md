# Backlog — Melhorias e Ideias (Pipeline Orientado a Eventos)

Registro de ideias, melhorias e pendências identificadas ao longo da migração pro pipeline orientado a eventos (`check_update` → `extract_and_load` → `build_and_promote`), ainda não priorizadas. Reconferido contra o código de `pipelines` (`origin/main`, commit `454c628a0`) em 2026-10-06 — status de cada item reflete esse momento.

---

## Arquitetura do dispatch

### Objeto único pros parâmetros repetidos da cadeia de promoção

**Contexto:** `dispatch_build_and_promote` → `build_and_promote` → `transfer_files_to_prod_flow` → `upload_to_gcs`/`register_table_materialization_task` repetem os mesmos parâmetros (`dataset_id`, `table_id`, `env`, `bq_project`, `prefect_mode`, `dump_mode`, `source_format`) em cada assinatura. Já causou um bug real: `source_format`/`dump_mode` nunca chegavam no `build_and_promote`, e qualquer dataset com `source_format="parquet"` (`br_ans_beneficiario`, `br_ms_cnes`, `us_cfpb_hmda`) falhava ao promover pra prod (`pipelines#2162`, fix já aplicado mantendo os parâmetros soltos — a correção estrutural fica pra depois).

**Proposta:** agrupar os campos idênticos e repassados sem mudança num modelo Pydantic único (sugestão: `MaterializationTarget`), passado adiante em vez de reescrito em cada assinatura. Precisa ser Pydantic (não dataclass puro) porque `build_and_promote` é um deployment Prefect real, disparado via `run_deployment(parameters={...})` — os parâmetros atravessam uma fronteira de processo, mesmo padrão já usado por `coverage: CoverageSpec`.

**Detalhe completo, incluindo quais campos ficam de fora e por quê:** `ftwca/tasks/pipelines/pr_2162/plano-objeto-parametros-promocao.md`.

**Status:** planejado, não implementado. Fazer como PR separado, depois do `pipelines#2162` mergeado.

---

### Variante `check_and_download`

**Contexto:** no levantamento original dos ~82 datasets candidatos à migração (2026-09-03), 26 foram classificados como `check_and_download` (o check em si já baixa o dado, não dá pra separar check de download como etapas independentes) — categoria distinta do padrão `check_update`/`extract_and_load` usado pelos 6 datasets já migrados.

**Status:** confirmado no código atual — `check_and_download` **não existe** em `stage_dispatch.py`, nunca foi prototipado. Pelo critério de 5GB decidido em 2026-09-03, 18 dos 26 datasets dessa categoria são leves o bastante pra tratar como padrão normal (baixar e conferir é barato); restam ~8 datasets que genuinamente precisam dessa variante (destaque: `us_fec_campaign_finance`, único com evidência concreta de passar de 5GB).

---

### Deployments órfãos não são removidos automaticamente quando um dataset migra

**Contexto:** quando um dataset migra pro pipeline orientado a eventos, o deployment do flow monolítico antigo (com schedule real, cron ativo) não é removido — fica rodando em paralelo com a cadeia nova até alguém notar e pausar manualmente. Isso já aconteceu de verdade no `pipelines#1932` (10 deployments órfãos, ligados em memória/na issue `basedosdados/iac#157`).

**Diferença importante:** isso é diferente do problema resolvido em `backend#1104` (que limpa registros de `DisabledFlowSchedule` **sem** schedule nenhum — etapas internas como `extract_and_load`/`build_and_promote`). O deployment monolítico órfão **tem** um schedule real — continua aparecendo e rodando até alguém decidir manualmente removê-lo ou pausá-lo.

**Status:** sem solução estrutural. Precisa de um runbook ou automação que, no momento da migração de um dataset, pause/arquive o deployment antigo automaticamente.

---

## Checklist da Laura (`pipelines#1932`) — itens adiados por dependerem da migração completa

### Rename de `poll_source_for_update_task`

**Contexto:** nome novo ainda não decidido na época (2026-08-21/09-29), adiado de propósito da rodada de renomeações feita nessa sessão.

**Status:** confirmado no código atual — `poll_source_for_update_task` (`pipelines/utils/metadata/tasks.py:273`) e `poll_source_for_update` (`pipelines/utils/metadata/register.py:320`) continuam com os nomes antigos. Ainda não retomado.

### `source_format`/`dump_mode` automáticos (inferidos, não configurados manualmente)

**Contexto:** hoje cada dataset declara `source_format`/`dump_mode` manualmente no `ExtractAndLoad` que `extract_load_data` devolve — depende de o autor do dataset lembrar de setar corretamente (e, como o `pipelines#2162` mostrou, de todo o resto do pipeline repassar esse valor sem perder). A ideia original (item 7 da checklist da Laura) era inferir isso automaticamente a partir do que `download_data` efetivamente escreve em disco, em vez de depender de configuração manual.

**Status:** não implementado — permanece configuração manual. Relacionado ao item do `MaterializationTarget` acima (resolve o "esquecer de repassar", não o "esquecer de configurar certo").

### Coverage vindo do backend (em vez de constante local por dataset)

**Contexto:** item 7 da checklist da Laura — hoje `coverage` é uma instância Pydantic declarada nas constantes de cada dataset; a ideia era buscar isso do backend em vez de manter duplicado no código.

**Status:** não implementado. Nenhuma issue própria aberta ainda.

---

## Bugs conhecidos, sem fix ainda

### `poll_source_for_update` quebra pra tabela sem `RawDataSource`

**Contexto:** achado em 2026-09-03 — o filtro GraphQL `rawDataSource_Id: null` é ignorado na query (`allPoll`/`allUpdate` em `pipelines/utils/metadata/client.py:290-341`), quebrando o poll pra qualquer tabela sem `RawDataSource` cadastrado.

**Status:** código no mesmo padrão de antes (`rawDataSource_Id` ainda usado como filtro direto) — sem confirmação de que foi corrigido. Ainda sem issue própria.

### `bd.Table.create()` só registra o ponteiro externo em `basedosdados-staging`, nunca em `basedosdados-dev`

**Status:** issue `pipelines#1967` **aberta**, não implementada.

### `br_me_caged` — `build_partitions` quebra sem arquivo de exclusão publicado

**Contexto:** `build_partitions` (`pipelines/crawler/me_caged/tasks.py:280`) falha com `FileNotFoundError` quando um período não tem nenhum arquivo de exclusão publicado (aconteceu de verdade, jul/2026). Dataset excluído da migração do `pipelines#1932` por causa disso.

**Status:** `br_me_caged` continua **não migrado** (`pipelines/datasets/br_me_caged/` só tem `flows.py`, sem `constants.py`/`tasks.py` — convenção antiga). Bug não confirmado como corrigido nem na versão antiga.

---

## Datasets bloqueados/pendentes de migração

### `br_bcb_sicor`

**Contexto:** comparação de atualização por tamanho de arquivo, não por data — gap não coberto pela cápsula genérica (`CheckThenExtractLoadPipeline`).

**Status:** confirmado não migrado — `pipelines/datasets/br_bcb_sicor/` só tem `flows.py` (monolítico).

### `br_sfb_sicar`

**Contexto:** foge do padrão comum o bastante pra precisar de revisão humana antes de tentar migrar.

**Status:** confirmado não migrado — mesma estrutura monolítica.

### `br_denatran_frota` / `br_senatran_estatisticas` — resolvido por si só

**Contexto:** em 2026-09-14, descobrimos que `br_denatran_frota` tinha sido renomeado em produção pra `br_senatran_estatisticas` por outros PRs mesclados em paralelo — a migração foi revertida e ficou pendente "decidir como reconciliar".

**Status:** **resolvido/obsoleto** — `pipelines/datasets/br_denatran_frota/` hoje só tem `__pycache__` (sem código-fonte, pasta morta); `pipelines/datasets/br_senatran_estatisticas/` é a implementação real atual (monolítica, não migrada pro pipeline orientado a eventos). Não há mais nada pra reconciliar — se `br_senatran_estatisticas` for migrado no futuro, é uma migração nova, não uma reconciliação de duas versões.

---

## Backend (`admin_data_tools`)

### Simplificar a sincronização Table ↔ Flow Schedule depois que os flows legados migrarem

**Contexto:** o backend (`SyncDeploymentsView`) liga automaticamente cada `Table` ao flow que a alimenta (`Table.flow_schedule`), resolvendo pela tag/nome `dataset_id__table_id` de cada deployment. Dois casos fogem dessa convenção simples e já exigiram lógica extra:

1. Flows monolíticos pré-migração que alimentam várias tabelas num só deployment, sem tag nenhuma (ex. `br_me_siconfi_flow`, 7 tabelas) — resolvido com um fallback que vincula todas as `CloudTable` do dataset.
2. Dentro de um desses datasets multi-tabela, uma tabela específica pode vir de um flow dedicado diferente — exigiu uma passada extra (`_dedicated_table_ids`) pra garantir que o fallback nunca "roube" essa tabela do dono certo, independente da ordem dos deployments.

**Pendência:** essa complexidade existe só por causa de flows que fogem do padrão — não é uma necessidade estrutural. Quem precisa mudar são os flows, não o código de sincronização.

1. Migrar os flows monolíticos multi-tabela (ex. `br_me_siconfi_flow`, e os ~33 datasets multi-tabela confirmados no levantamento de 2026-10-07) pro pipeline orientado a eventos — ou, no mínimo, dar a cada tabela sua própria tag `dataset_id__table_id`, mesmo sem migrar o resto do flow.
2. Depois disso, remover o fallback multi-tabela e `_dedicated_table_ids` de `_resolve_tables` (`backend/apps/admin_data_tools/flow_monitoring.py`), voltando a uma resolução simples de "uma tag, uma tabela" (`_resolve_single_table`).

**Status:** não implementado — registrado como dívida técnica intencional (complexidade aceita deliberadamente pra cobrir o estado real de hoje, não pra ficar permanente).

---

## Resolvidos desde que foram registrados (mantido aqui por rastreabilidade)

- **`rename_flow_run_dataset_table` chamada sem `await`** — `pipelines#1940`, **fechada**.
- **Deploy de prod deveria ser seletivo, não `--all`** — `pipelines#1943`, **fechada**.
- **Fragilidade do `sync-deployments` (sobrescrevia estado `paused` com registros desatualizados)** — resolvida por `basedosdados/backend#1060` (endpoint `set-schedule-active`, mergeado e em produção desde 2026-10-06).
- **Página "Flow Schedules" do admin listava deployments sem schedule nenhum** (etapas internas `extract_and_load`/`build_and_promote`) — resolvida por `basedosdados/backend#1104` (mergeado, deployado, `sync-deployments` já rodado em produção removendo 487 registros órfãos).
