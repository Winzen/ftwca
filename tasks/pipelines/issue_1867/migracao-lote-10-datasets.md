# Migração em lote — 10 datasets reais (issue #1867, passo 3)

Continuação de [[migracao-br-ibge-ipca]]: depois de validar o padrão manualmente num dataset, o usuário pediu pra aplicar o mesmo padrão em mais 10 de uma vez — "não precisa testar, só fazer algo preliminar mesmo". Trabalho feito em 2026-09-09, branch `feat/event-pipeline-automations-poc`, via 10 agentes em paralelo (cada um migrando um dataset, seguindo a mesma diretiva). **Implementado e verificado estaticamente (ruff/pyrefly/import/colisão de nomes); nada deployado, testado contra API/backend real, commitado ou dado push.**

## Datasets migrados

| Dataset | Tabelas reais | Check | Observação |
|---|---|---|---|
| `br_ans_beneficiario` | 1 (`informacao_consolidada`) | leve de verdade | `download_data` repete a listagem HTML leve (não dá pra repassar entre pods) |
| `br_denatran_frota` | 2 (`uf_tipo`, `municipio_tipo`) | leve de verdade | encaixou limpo, sem truque nenhum |
| `br_ibge_ipca` | 4 | **disfarçado** (ver [[migracao-br-ibge-ipca]]) | já documentado antes |
| `br_inmet_bdmep` | 1 (`microdados`) | **disfarçado** | baixa o ZIP do ano inteiro (todas as ~600 estações) só pra ler os nomes de arquivo e achar a data — tamanho real não medido, precisa confirmar antes de produção |
| `br_me_caged` | 3 (`microdados_movimentacao`, `_fora_prazo`, `_excluida`) | leve de verdade | check único, compartilhado pelas 3 (mesma fonte FTP); preserva catch-up de múltiplos meses atrasados |
| `br_me_cnpj` | 4 (`empresas`, `socios`, `estabelecimentos`, `simples`) | leve de verdade (PROPFIND/WebDAV) | ver "Mudança na cápsula" abaixo; gap não replicado (ver "Pendências") |
| `br_me_comex_stat` | 4 (`municipio_exportacao/importacao`, `ncm_exportacao/importacao`) | leve de verdade | check único compartilhado pelas 4; particionamento varia entre 2 e 3 níveis, resolvido via descoberta das pastas escritas em disco depois do download — parte menos testada de tudo, conferir com atenção |
| `br_ms_cnes` | 13 | leve de verdade (listagem FTP) | maior leva, 26 flows; fallback pra quando o FTP não tem período novo (`None` → usa a coverage atual) |
| `br_sfb_sicar` | 9 (materializadas juntas) | leve de verdade | ver "Desvio deliberado" abaixo |
| `us_cfpb_hmda` | 1 (`loan_application_register`) | leve de verdade | `dump_mode="overwrite"` reconstrói o histórico inteiro (2018..ano mais recente) a cada run — já era assim no flow antigo, preservado |

Contando `br_ibge_ipca`, isso fecha **11 datasets migrados** — mais um confirmadamente bloqueado (`br_bcb_sicor`, ver abaixo). Todos os flows monolíticos antigos foram **removidos por completo** dos respectivos `flows.py` (não coexistem com os novos) — mesma mitigação de risco de deploy documentada em [[migracao-br-ibge-ipca]] (não deixar flow antigo no arquivo antes de qualquer deploy pela branch de trabalho).

## Padrão de erro que se confirmou de novo

Terceira e quarta vez nesta issue que a contagem de tabelas do levantamento original estava errada: `br_sfb_sicar` tem 9 tabelas (não 1), reforçando o que já valia pra `br_denatran_frota` (2, não 1) e `br_ibge_ipca` (4, não 1). Regra prática continua de pé: **sempre inspecionar o código de verdade antes de migrar**, nunca confiar só no levantamento de alto nível pra decidir escopo.

Em compensação, a classificação "check leve" (categoria Padrão) do levantamento **acertou** pra 8 dos 10 (só `br_inmet_bdmep` foi disfarçado nesta leva, contra `br_ibge_ipca` na leva anterior) — o padrão de erro é específico de contagem de tabelas, não de categoria de check.

## Mudança na infraestrutura compartilhada

`pipelines/utils/stage_dispatch.py` ganhou um parâmetro `compare_against: str = "coverage"` em `check_update_and_dispatch()`/`CheckThenDownloadPipeline` (default idêntico ao comportamento anterior, backward-compatible). Motivo: `br_me_cnpj.simples` é `NonHistorical` (sem `Coverage.DateTimeRange` confiável pra comparar) e precisa de `compare_against="table_update"` — valor que `poll_source_for_update_task` já aceitava, só não estava exposto pela cápsula. Revisado e confirmado correto; `test_dataset`/`br_ibge_ipca` continuam passando no `pyrefly` depois da mudança.

## Desvio deliberado — `br_sfb_sicar`

Não usa `CheckThenDownloadPipeline`/`dispatch_mat_test` pro estágio de download. As 9 tabelas de tema são materializadas numa execução só, com upload resumível por UF/tema em GCS (~30h de corrida no backfill nacional) e **testes dbt cruzados entre as 9 tabelas** depois que todas terminam de ser construídas — o `mat_test_flow` genérico testa 1 tabela por vez e quebraria essa dependência cruzada. Implementado como:
- `check_for_update`/`br_sfb_sicar_check_update_flow` usando `check_update_and_dispatch` diretamente (não a classe) — dispatch e nomenclatura seguem a convenção normal.
- `download_and_materialize`/`br_sfb_sicar_download_flow` — corpo quase idêntico ao flow monolítico antigo (download+clean+stage+dbt+test+metadata das 9 tabelas juntas), sem devolver `DownloadResult` nem disparar `mat_test`.

**Precisa de revisão humana antes de qualquer deploy real** — a divisão em 2 pods é segura em teoria (o `download` de 30h/resumível não muda de comportamento), mas nunca foi exercitada nesse novo split.

## Pendências reais — capacidades que a cápsula ainda não tem

Três gaps descobertos e **documentados, não implementados** (decisão consciente de não expandir escopo no meio de uma migração em lote preliminar):

1. **Comparação por tamanho, não por data** (`br_bcb_sicor`, bloqueado por completo — nenhum arquivo alterado). O dataset tem 11 tabelas (10 + `dicionario`, não 1), e a checagem real usa `poll_source_size_for_update_task`/`commit_source_size_update_task` (`byte_length`), não data. `CheckResult`/`check_update_and_dispatch` são hardcoded pra comparação por data (`reference_date`). Precisa estender `stage_dispatch.py` com um modo de comparação por tamanho antes de migrar datasets desse tipo — provavelmente existem outros no repo com o mesmo padrão.
2. **Overrides manuais de execução** (`us_cfpb_hmda`). O flow antigo aceitava `materialize_to_prod`/`update_metadata`/`force_run` como parâmetros pra rodar só em dev, pular a materialização, ou forçar uma run mesmo sem dado novo — usado pra teste/debug manual. `CheckThenDownloadPipeline` não tem equivalente hoje.
3. **Materialização/teste cruzado entre várias tabelas de um mesmo dataset** (`br_sfb_sicar`). `mat_test_flow` genérico assume 1 tabela por vez; não há como expressar "testa estas 9 juntas, só depois que todas terminarem de baixar".

## Status (2026-09-09)

11 datasets migrados (`br_ibge_ipca` + os 10 desta leva), 1 bloqueado (`br_bcb_sicor`, documentado, zero código alterado). `ruff`/`pyrefly` limpos em tudo; confirmado por script que os 68 flows novos (mais os 8 do `br_ibge_ipca`) não colidem entre si nem em nome de `@flow` nem em nome de deployment. **Nada deployado, testado contra rede/backend real, commitado ou dado push** — trabalho puramente preliminar, à espera de validação real dataset por dataset.

Próximos passos possíveis, não decididos ainda:
- Decidir a ordem de validação real (deploy sem schedule + disparo manual, como já foi feito pro `mes_brasil` do `br_ibge_ipca`).
- Decidir se/quando estender a cápsula pra cobrir os 3 gaps acima, ou se `br_bcb_sicor`/`br_sfb_sicar`/`us_cfpb_hmda` ficam como pendência separada até lá.
