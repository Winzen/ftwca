# Plano: objeto único pros parâmetros repetidos da cadeia de promoção

Contexto: `pipelines#2162` (ainda não mergeado no momento em que este plano foi escrito).

## Problema

`dispatch_build_and_promote` → `build_and_promote` → `transfer_files_to_prod_flow` → `upload_to_gcs`/`register_table_materialization_task` formam uma cadeia de 3-4 funções que repetem os mesmos parâmetros em cada assinatura: `dataset_id`, `table_id`, `env`, `bq_project`, `prefect_mode`, e agora também `dump_mode`/`source_format` (adicionados no `pipelines#2162`).

Isso já causou um bug real: `source_format`/`dump_mode` do `ExtractAndLoad` do dataset nunca chegavam no `build_and_promote` — a chamada final a `upload_to_gcs` caía nos defaults hardcoded (`"csv"`/`"append"`), e qualquer dataset com `source_format="parquet"` (`br_ans_beneficiario`, `br_ms_cnes`, `us_cfpb_hmda`) sempre falhava ao promover pra prod. O fix (`pipelines#2162`) resolve isso adicionando os 2 parâmetros em cada camada — mas não resolve o problema estrutural: é fácil esquecer de repassar um parâmetro novo em alguma das 3-4 camadas, e isso falha silenciosamente até alguém testar o caminho específico que usa aquele valor não-default.

## Proposta

Agrupar os parâmetros que são **idênticos e repassados sem mudança** através de toda a cadeia num único objeto, passado adiante em vez de reescrito em cada assinatura.

### Por que precisa ser um modelo Pydantic, não um dataclass puro

`build_and_promote` é um **deployment Prefect real**, disparado via `run_deployment(parameters={...})` (não uma chamada de função direta) — os parâmetros atravessam uma fronteira de processo e precisam ser serializáveis/validáveis como JSON. O código já prova que isso funciona bem com um modelo Pydantic: `coverage: CoverageSpec` (união discriminada `AllFree`/`AllBdpro`/`PartBdpro`/`NonHistorical`, em `pipelines.utils.metadata.domain`) já atravessa exatamente essa mesma fronteira hoje. O novo objeto deve seguir o mesmo padrão.

`transfer_files_to_prod_flow`, por outro lado, é chamado direto (não via `run_deployment`) de dentro de `build_and_promote` — não teria essa restrição sozinho, mas receber o mesmo objeto do `build_and_promote` mantém a cadeia consistente.

### Campos propostos — dentro do objeto

Só o que é idêntico e passa adiante sem mudar de significado em nenhuma camada:

- `dataset_id: str`
- `table_id: str`
- `env: str`
- `bq_project: str`
- `prefect_mode: str`
- `dump_mode: str`
- `source_format: str`

Nome sugerido (em aberto, discutir): `MaterializationTarget` — descreve "onde e como esta tabela deve ser escrita", que é exatamente o que esses 7 campos capturam juntos.

### Campos que ficam de fora (continuam individuais)

Parâmetros com semântica diferente em cada camada, ou que não vêm do `ExtractAndLoad`:

- `coverage` (`CoverageSpec`) — já é seu próprio objeto Pydantic, ortogonal ao destino de materialização.
- `partition_folders` — specífico do resultado do download (`ExtractAndLoad.partition_folders`), não do "destino".
- `promote_to_prod` — **não** vem do dataset: é derivado de `_dev_prefix()` dentro de `dispatch_build_and_promote` (quem despacha decide, não quem é despachado) — misturar isso no objeto criaria a falsa impressão de que é configurável pelo dataset.
- `download_billing_project`, `materialize_after_dump`, `update_metadata` — flags específicas de cada etapa, sem equivalente no `ExtractAndLoad`.
- Os parâmetros de fallback manual do `transfer_files_to_prod_flow` (`coverage_tier`, `date_column_kind`, `date_col`, `year_col`, `month_col`, `quarter_col`, `free_lag_unit`, `free_lag_value`) — só fazem sentido em invocação manual sem `coverage` pronto; não fazem parte do caminho automático de dispatch.

## Como a cadeia ficaria

```
dispatch_build_and_promote(dataset_id, table_id, result: ExtractAndLoad, env)
    │
    │  constrói o objeto a partir de result + env:
    │  target = MaterializationTarget(
    │      dataset_id=dataset_id, table_id=table_id, env=env,
    │      bq_project=..., prefect_mode=result.prefect_mode,
    │      dump_mode=result.dump_mode, source_format=result.source_format,
    │  )
    ▼
run_deployment(parameters={"target": target, "coverage": ..., "promote_to_prod": ..., "partition_folders": ...})
    │
    ▼
build_and_promote(target: MaterializationTarget, coverage, promote_to_prod, partition_folders, ...)
    │  repassa target inteiro, sem desmontar campo por campo
    ▼
transfer_files_to_prod_flow(target: MaterializationTarget, folders=partition_folders, coverage=coverage, ...)
    │
    ▼
upload_to_gcs(data_path=..., target=target)
register_table_materialization_task(target=target, coverage=coverage)
```

Um parâmetro novo que precise atravessar toda a cadeia (o próximo "`source_format` esquecido") passa a ser um campo a mais no objeto — aparece automaticamente em toda função que já recebe `target`, sem precisar editar 3-4 assinaturas.

## Riscos e pontos em aberto

- **Nome do objeto** — `MaterializationTarget` é só uma sugestão inicial, vale validar se bate com a nomenclatura já usada no resto do código.
- **UI do Prefect** — os parâmetros de um flow run aparecem como JSON na UI do Prefect. Hoje cada campo (`dataset_id`, `table_id`, `env`...) aparece solto; agrupados num objeto, aparecem aninhados sob uma chave (`target: {...}`). É mais organizado pra um grupo coeso, mas é uma mudança visual real pra quem inspeciona runs na UI — vale confirmar que não atrapalha nenhum hábito de debug já estabelecido.
- **Coordenação de deploy** — `build_and_promote` é um deployment único compartilhado por todos os datasets/tabelas (`pipeline único`, não um por dataset). Mudar sua assinatura de parâmetros exige redeploy coordenado (`deploy_flows.py`) — não é algo que "just works" incrementalmente como uma função interna; precisa ser testado contra o Prefect real antes de ir pra produção, do mesmo jeito que qualquer mudança de contrato de deployment.
- **Risco de regressão caso algo dispare `build_and_promote` com o shape antigo de parâmetros** durante a transição — não deve haver chamadas externas manuais hoje fora do próprio `dispatch_build_and_promote` (não encontradas no levantamento do `pipelines#2162`), mas vale confirmar de novo no momento de implementar.

## Sequenciamento decidido

Fazer como PR separado, **depois** que `pipelines#2162` (o fix urgente de `source_format`/`dump_mode`) for revisado e mergeado — não misturar um fix de produção já no ar com uma refatoração estrutural maior. `pipelines#2162` já resolve o sintoma imediato (os 3 datasets parquet pausados); esta refatoração é a correção estrutural pra não repetir o mesmo tipo de bug no futuro.
