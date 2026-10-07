# Ideias — como testar os datasets migrados sem rodar tudo por completo

Discussão em aberto, 2026-09-13. **Nada decidido/implementado ainda** — registro das ideias levantadas pra retomar depois. Contexto: 11 datasets migrados (`br_ibge_ipca` + 10 da leva, ver [[migracao-lote-10-datasets]]), 34 pares tabela/pipeline, nenhum testado contra fonte/backend real ainda.

## O problema

Rodar o teste de ponta a ponta (`check_update` → `download` → `mat_test`) completo, contra dado real, pras 34 tabelas seria caro: alguns datasets baixam dado pesado (`br_me_cnpj`: dump completo de CNPJ; `us_cfpb_hmda`: reconstrói o histórico inteiro toda run; `br_ms_cnes`: FTP/DBC real), e materializar (dbt run/test em dev+prod) tem custo de BigQuery real. Fazer isso pras 34 de uma vez é desnecessário pra validar o que realmente importa nesse momento: **a fiação** (nomes, dispatch, resolução de deployment), não necessariamente o dado em si.

## Ideia 1 — separar "testar a fiação" de "testar o dado de verdade"

Duas camadas de custo bem diferente:

1. **Camada barata, cobre as 34**: disparar só `check_update` de cada tabela. Confirma que resolve `has_new_data` certo e aponta pro `download` certo — seguro mesmo em produção real, porque se não houver dado novo, nada cascateia. Cobre a maior parte do que pode ter quebrado na leva de refatoração (nome errado, colisão, import quebrado, `.fn.__name__` divergente).
2. **Camada cara, só numa amostra representativa**: escolher 1 dataset por "formato" de teste diferente (período único pequeno — `br_ibge_ipca`; catch-up de múltiplos meses — `br_me_caged`; FTP/DBC real — `br_ms_cnes`; dump pesado — `br_me_cnpj`) e rodar de ponta a ponta de verdade só nesses, aceitando o custo real uma vez por formato, não por dataset.

## Ideia 2 — baixar só "um pedaço" do dado, pra testar mais barato

Investigado se dá pra baixar só ~1000 linhas em vez do arquivo inteiro.

**Truncar na fonte (HTTP Range request)** — só funciona quando a fonte serve CSV puro, sem compressão, e o servidor aceita `Range`. Auditoria rápida dos 11 datasets:

| Dataset | Formato de download |
|---|---|
| `us_cfpb_hmda` | CSV puro via `curl` direto na URL — **único confirmado como candidato real** (falta testar se o servidor aceita `Range`) |
| `br_me_comex_stat` | `ZipFile` |
| `br_me_caged` | `.7z` via FTP |
| `br_ms_cnes` | `.dbc` (formato proprietário compactado do DATASUS) |
| `br_denatran_frota` | `.xlsx` (é um ZIP por dentro) |
| `br_sfb_sicar` | ZIP de shapefile |
| outros (`br_ibge_ipca`, `br_ans_beneficiario`, `br_inmet_bdmep`, `br_me_cnpj`) | não auditados a fundo ainda |

Pra ZIP/7z/DBC/XLSX, um `Range` request devolve bytes truncados de um container compactado — geralmente ilegível, não dá pra descomprimir parcialmente sem uma lógica bem mais elaborada (e mesmo assim, um membro específico do ZIP pode ser grande sozinho).

**Alternativa que funciona pra qualquer formato**: baixar o arquivo real inteiro (sem jeito de evitar isso pros comprimidos), mas truncar pra `N` linhas **na hora de escrever** o CSV/parquet final que sobe pro staging (`df.head(1000)` antes de `to_partitions`/`to_csv`, dentro de cada `download_data`). Não economiza banda/tempo de download da fonte, mas economiza o lado caro daqui pra frente (BigQuery, dbt run/test). É trabalho real por dataset (um `sample`/`limit` a mais em cada `download_data`), não uma regra genérica de graça.

## Ideia 3 — um flow de teste genérico por `dataset_id`/`table_id`

Proposta do usuário: um único flow parametrizado que baixa uma amostra, sobe, materializa, testa — todo o caminho, inclusive alterando uma tabela no backend referente ao teste.

**Parte que não existe hoje, precisa ser criada**: um jeito de descobrir dinamicamente qual `tasks.py` usar pra um `dataset_id`/`table_id` dado (ex. `importlib.import_module(f"pipelines.datasets.{dataset_id}.tasks")` + `make_pipeline(table_id)`). Direto de fazer, já que todo dataset migrado segue a convenção de expor `make_pipeline` (ver [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]]).

**Ponto em aberto — como lidar com "alterar uma tabela no backend"**, duas abordagens bem diferentes:

- **(A) Tabela de teste dedicada**: criar um `table_id` novo tipo `<table_id>__smoke_test` no backend, Coverage/Update próprios, isolado do real. Mais seguro/isolado, mas dá trabalho — precisaria de um modelo dbt próprio também (`run_dbt` casa `dataset_id.table_id` com o nome do modelo dbt), ou seja, na prática testaria uma cópia do modelo, não o modelo real usado em produção.
- **(B) Usar a tabela real, só em `dev`**: deixar a materialização e o commit de coverage acontecerem contra o backend/tabela de verdade, com `targets=["dev"]` (nunca promove pra prod). Sobrescreve o dev atual, mas isso já é esperado — dev é área descartável, qualquer run real de dev já sobrescreve a anterior, e a coverage/dado real de prod nunca é tocado. Mesmo padrão já usado nesta sessão (forçar dado novo rolando a coverage de `test_dataset`/`br_ibge_ipca` pra trás, deixando uma run real de verdade corrigir depois).

(A) isola completamente mas exige duplicar modelo/tabela por dataset. (B) testa o caminho real de verdade (inclusive o modelo dbt real em uso), sem esse overhead, mas mexe — ainda que só em dev, de forma reversível/descartável — num registro real do backend.

**Inclinação até agora**: (B), restrito a `dev` — nunca deixar esse flow genérico promover uma amostra de teste pra prod (`targets=["dev"]` sempre, nunca `["dev","prod"]`), já que isso escreveria dado de teste na tabela de produção de verdade.

## Status

Conversa pausada aqui — nenhuma decisão fechada, nada implementado. Retomar decidindo, nesta ordem:
1. Confirmar a abordagem (B) vs (A) pro backend (ou uma terceira opção não cogitada ainda).
2. Decidir se vale implementar o truncamento por linhas (`sample`/`limit` em `download_data`) em todos os 11 datasets, ou só nos escolhidos pra "camada cara" da Ideia 1.
3. Implementar o flow genérico de teste (`dataset_id`/`table_id` → dispatch dinâmico via `tasks.py`).
