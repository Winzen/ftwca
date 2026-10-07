# Plano de testes reais — datasets migrados (issue #1867)

Continuação de [[migracao-lote-10-datasets]] e [[ideias-teste-em-lote-datasets-migrados]]: depois da migração preliminar dos 11 datasets, começamos a validar de verdade contra fonte/backend real, dataset por dataset. Este documento registra o que já foi provado e o peso de cada dataset restante, pra decidir a ordem dos próximos testes.

## O que já foi provado — `br_ibge_ipca`

### `mes_brasil` — ponta a ponta completo, com dado real

Primeiro teste real de toda a arquitetura (2026-09-13). Sequência completa `check_update → download → mat_test` disparada contra o backend/fonte de verdade:

1. Primeira tentativa: `check_update` rodou e concluiu `COMPLETED`, mas não disparou `download` — a produção real (cron próprio, independente desta branch de teste) já tinha avançado a coverage de 2026-06 pra 2026-08 nos dias entre uma checagem e outra. **Resultado correto, não um bug** (a lógica de "não há dado novo" funcionou como esperado).
2. Pra forçar a detecção de dado novo, a coverage real do backend foi rolada de volta pra julho (`startYear=2026, startMonth=3, endYear=2026, endMonth=8` → ajustada), decisão explícita do usuário.
3. Com a coverage rolada pra trás, a cadeia completa rodou: `check_update` → `download` → `mat_test` (`run_dbt(target=dev)` → `transfer_files_to_prod_flow` → `run_dbt(target=prod)` → `register_table_materialization_task`), todos os três flow runs terminando `COMPLETED`.
4. Dado real do IPCA (agosto/2026) foi materializado na tabela real `basedosdados.br_ibge_ipca.mes_brasil` em produção, e a coverage do backend se autocorrigiu de volta pro novo máximo depois do commit.

**Esse é o primeiro dataset com o caminho completo validado ponta a ponta contra produção real.**

### As outras 3 tabelas do dataset (`mes_categoria_brasil`, `mes_categoria_rm`, `mes_categoria_municipio`)

Mesmo `tasks.py`/`make_pipeline` do `mes_brasil` (ver [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]] — as 4 tabelas compartilham 100% da lógica, só variando `table_id`). Testadas em 2026-09-13 **sem mexer em coverage** (pedido explícito do usuário: "não precisa mudar no backend") — só disparo do `check_update` real:

| Tabela | `check_update` | `download` disparado? |
|---|---|---|
| `mes_categoria_brasil` | `COMPLETED` | não (sem dado novo — coverage já em dia) |
| `mes_categoria_rm` | `COMPLETED` | não (sem dado novo — coverage já em dia) |
| `mes_categoria_municipio` | `COMPLETED` | não (sem dado novo — coverage já em dia) |

Valida o dispatch (nomes de deployment corretos, zero colisão) e a comparação contra coverage; não prova `download`/`mat_test` nessas 3 especificamente — mas como o código é idêntico ao `mes_brasil` (já provado ponta a ponta), o risco residual é baixo.

## Peso e homogeneidade dos outros 10 datasets

Levantamento feito em 2026-09-13, lendo `flows.py`/`tasks.py` de cada um, pra decidir quantos testes (não quantas tabelas) cada dataset realmente precisa.

| Dataset | Nº tabelas | Mesma lógica em todas? | O que 1 teste cobre |
|---|---|---|---|
| `br_ms_cnes` | 13 | **sim** — código idêntico, só varia `table_id` | 1 tabela generaliza pras 13 |
| `br_ans_beneficiario` | 1 | — | única, testa tudo |
| `br_inmet_bdmep` | 1 | — | única, testa tudo |
| `us_cfpb_hmda` | 1 | — | única, testa tudo |
| `br_me_cnpj` | 4 | quase — `empresas`/`socios`/`estabelecimentos` iguais; `simples` diferente (`NonHistorical`, `compare_against="table_update"`) | 1 tabela cobre as 3 iguais; `simples` precisa de teste à parte |
| `br_me_caged` | 3 | `check_for_update` é função única compartilhada; `download_data` difere por tipo de arquivo (CAGEDMOV/FOR/EXC) | 1 teste valida o check comum, não o parsing das outras 2 |
| `br_me_comex_stat` | 4 | `check_for_update` é função única compartilhada; `download_data` difere (município×NCM, export×import) | idem — check generaliza, parsing não |
| `br_denatran_frota` | 2 | **não** — parsing genuinamente distinto por tabela (`uf_tipo` vs `municipio_tipo`) | **migração revertida (2026-09-14)** — dataset renomeado pra `br_senatran_estatisticas` em produção, ver seção "Rodada de testes 2026-09-14" |
| `br_sfb_sicar` | 9 (materializadas juntas) | não se aplica — é uma unidade só, teste dbt cruzado entre as 9 | precisa revisão humana antes de qualquer teste real (ver [[migracao-lote-10-datasets]]) |
| `br_bcb_sicor` | 11 | — | **bloqueado**, ainda na arquitetura antiga, nem migrado |

## Rodada de testes 2026-09-14 — 9 tabelas representativas

Deploy seletivo (script à parte, reaproveitando `load_flows_from_file`/`deploy_flow` de `deploy_flows.py`, mas só pras funções de 1 tabela por dataset — não o arquivo inteiro) da tabela mais leve de cada um dos 8 datasets restantes, escolhida por contagem real de linhas no BigQuery (não estimativa): `br_ms_cnes.estabelecimento_ensino`, `br_me_caged.microdados_movimentacao_excluida`, `br_me_comex_stat.municipio_exportacao`, `br_denatran_frota.uf_tipo`, `br_me_cnpj.socios` + `br_me_cnpj.simples` (2 tabelas, caminhos de `compare_against` diferentes), `br_ans_beneficiario`, `br_inmet_bdmep`, `us_cfpb_hmda` (únicas). 18 flows registrados no pool `basedosdados-dev`, `paused=True`, 0 erros.

Disparados os 9 `check_update` reais:

| Dataset | Resultado | Observação |
|---|---|---|
| `br_ms_cnes` | ✅ COMPLETED | — |
| `br_ans_beneficiario` | ✅ COMPLETED | — |
| `br_inmet_bdmep` | ✅ COMPLETED | — |
| `br_me_caged` | ✅ COMPLETED | — |
| `br_me_comex_stat` | ✅ COMPLETED | — |
| `us_cfpb_hmda` | ✅ COMPLETED | — |
| `br_me_cnpj` (`socios`, `simples`) | ❌ FAILED (4x cada, 2 rodadas ~15min de diferença) | `ConnectionError`/`RemoteDisconnected` no PROPFIND contra o servidor da Receita Federal, reincidente nas 4 tentativas. **Origem: a fonte original** (servidor WebDAV da Receita Federal), não o cluster de dev nem o código — confirmado pelo usuário. O teste direto que respondeu `HTTP 207` daqui não foi suficiente pra descartar a fonte (rede/IP/timing diferentes o bastante pra não ser conclusivo). **Retirado dos testes desta rodada por enquanto** — retomar quando a fonte estabilizar, sem exigir mudança de código ou de infra da nossa parte. |
| `br_denatran_frota` | ❌ FAILED | Ver seção seguinte — **não é bug, é rename real em produção**. |

### `br_denatran_frota` — achado real, migração revertida

`IndexError` em `backend._get_table_id_from_name` (`allCloudtable` vazio pra `gcp_dataset_id="br_denatran_frota"`). Investigando o PR #1932 (`gh pr view 1932` mostrava `mergeable: CONFLICTING`) e rodando `git merge-tree` contra `origin/main`, descobrimos a causa raiz: **o dataset foi renomeado em produção, em paralelo a este trabalho**, via dois PRs mesclados no `main` que esta branch não tinha:

- `497c3101` — "refactor(br_senatran_estatisticas): renomeia br_denatran_frota e corrige pre_hook de row access policies" (#1934)
- `a973cd75` — "feat(br_senatran_estatisticas): adiciona municipio_combustivel e o parâmetro backfill_start" (#1936)

Confirmado também via BigQuery: `br_senatran_estatisticas.uf_tipo` tem 159.327 linhas contra 158.760 em `br_denatran_frota.uf_tipo` — a tabela nova é a que está recebendo dado real agora.

O `br_senatran_estatisticas` em `main` **continua no padrão monolítico antigo** (não foi migrado pro pipeline orientado a eventos) e ganhou coisas que nossa migração não previa: uma 3ª tabela (`municipio_combustivel`) e parâmetros de override manual (`force_run`/`backfill_start` — resolve o mesmo gap nº 2 documentado em [[migracao-lote-10-datasets]], só que de um jeito diferente do que cogitamos pro `us_cfpb_hmda`).

**Decisão do usuário (2026-09-14): ignorar `br_denatran_frota` por enquanto e desfazer a migração.** Revertido:
- `constants.py`/`flows.py`/`tasks.py` de `pipelines/datasets/br_denatran_frota/` restaurados ao estado anterior à migração (commit `e54a55b6`, pai de `90cfd080`) — de volta ao flow monolítico único, sem `check_update`/`download` separados.
- Os 2 deployments criados nesta rodada (`br_denatran_frota_uf_tipo_check_update_flow`, `..._download_flow`) foram **deletados** do Prefect (pool dev) — não deixar deployment órfã pra trás.
- Os deployments antigos reais (`br_denatran_frota__uf_tipo`, `br_denatran_frota__municipio_tipo`, provavelmente já órfãos desde o rename em produção) **não foram tocados** — fora do escopo desta sessão.
- `ruff`/`pyrefly` conferidos limpos depois da reversão.

Retomar depois decidindo: migrar direto pra cima do `br_senatran_estatisticas` já existente em `main` (reconciliando as 3 tabelas + `force_run`/`backfill_start`), ou esperar o PR #1932 ser rebasado primeiro.

**Código da migração revertida não foi perdido** — continua recuperável a qualquer momento nos commits `90cfd080` (versão original) e `a1895b24` (depois do refactor `pipeline_factory`), ambos pushados em `feat/event-pipeline-automations-poc`:

```bash
git show a1895b24:pipelines/datasets/br_denatran_frota/flows.py
# ou pra restaurar de vez, quando for a hora:
git checkout a1895b24 -- pipelines/datasets/br_denatran_frota/
```

## Plano — ordem sugerida

1. **`br_ms_cnes`** — maior peso (13 tabelas) resolvido por 1 teste só, mesmo perfil de "código idêntico" que já funcionou no `br_ibge_ipca`. Prioridade alta por custo-benefício.
2. **`br_ans_beneficiario`, `br_inmet_bdmep`, `us_cfpb_hmda`** — datasets de 1 tabela só, teste direto, sem decisão extra sobre "qual tabela escolher".
3. ~~`br_me_cnpj`~~ — **pausado por enquanto**, fonte original (Receita Federal) instável nas últimas tentativas (4 falhas de conexão em 2 rodadas). Retomar quando a fonte normalizar: 2 testes, 1 tabela entre `empresas`/`socios`/`estabelecimentos` (generaliza as 3) + `simples` isolado (valida o `compare_against="table_update"`).
4. **`br_me_caged`, `br_me_comex_stat`** — testar 1 tabela valida o check compartilhado; as demais tabelas de cada um ficam com risco residual (parsing não exercitado) até serem testadas individualmente depois.
5. ~~`br_denatran_frota`~~ — **fora do plano**, migração revertida (dataset renomeado pra `br_senatran_estatisticas` em produção, ver acima). Retomar só depois de decidir como reconciliar com o que já existe em `main`.
6. **`br_sfb_sicar`** — só depois da revisão humana pendente (foge do padrão, split em 2 pods nunca exercitado).
7. **`br_bcb_sicor`** — fora do plano até a cápsula ganhar suporte a comparação por tamanho (gap documentado em [[migracao-lote-10-datasets]]).

## Status

Nada além do `br_ibge_ipca` (4/4 tabelas, 1 com caminho completo provado) foi testado contra produção real até agora. Este documento define a ordem; a decisão de rolar coverage pra forçar dado novo (como feito no `mes_brasil`) é caso a caso — nem sempre necessária nem sempre desejável (nas 3 tabelas restantes do `br_ibge_ipca` optou-se por não mexer em coverage).
