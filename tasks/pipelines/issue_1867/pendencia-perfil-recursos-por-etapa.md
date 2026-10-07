# Pendência — perfil de recursos do pod ainda não diferenciado por etapa

Registrado em 2026-09-14, a partir de uma observação do usuário sobre o PR [basedosdados/pipelines#1932](https://github.com/basedosdados/pipelines/pull/1932). **Nada decidido/implementado ainda** — só o desenho do problema, pra retomar depois.

## O problema

Os flows de `check_update` (ver [[pipeline-orientado-a-eventos-fluxo-e-nomenclatura]]) são propositalmente leves — fazem só um poll de metadado (data mais recente na fonte, ou uma checagem HTTP leve), sem baixar dado nenhum. Mas hoje eles herdam o **mesmo `job_variables`/tamanho de pod** que os flows de `download`/`mat_test`, que de fato baixam e materializam dado real — potencialmente pesado.

Isso vale pros **datasets já migrados** (os 9 da leva + `br_ibge_ipca`), não só pra migrações futuras — nenhum deles tem um `job_variables` diferenciado pro `check_update` hoje.

`deploy_flow()` (`deploy_flows.py`) resolve `job_variables` assim:

```python
job_variables = getattr(flow, "job_variables", None)
```

— ou seja, só usa um perfil de recursos diferente se o `flows.py` do dataset setar isso explicitamente no flow (como `mat_test_flow.job_variables = MAT_TEST_JOB_VARIABLES` já faz, ver `pipelines/utils/metadata/flows.py`). Nenhum dataset migrado faz isso hoje pro `check_update` — cai no default do work pool, dimensionado historicamente pro flow monolítico antigo (que fazia tudo num pod só).

## Duas abordagens possíveis, ainda não decididas

1. **Baixar o perfil padrão do work pool** — mudar o baseline de recursos que todo pod novo recebe, sem precisar tocar em código de dataset nenhum. Mais simples, mas afeta indiscriminadamente qualquer flow que não sobrescreva `job_variables` (não só `check_update`).
2. **Automatizar no `deploy_flows.py`** — se a tag `check_update` (já setada via `deploy_tags(dataset_id, Etapa.CHECK_UPDATE)`, ver `pipelines/utils/stage_dispatch.py`) estiver presente no flow, aplicar um `job_variables` de recursos mais baixos por padrão automaticamente, sem exigir que cada dataset configure isso manualmente — mais preciso (só afeta `check_update`), mas é lógica nova em `deploy_flow()`/`deploy_flows.py`.

## Status

Só desenhado. Retomar decidindo entre as duas abordagens (ou uma combinação — baseline mais baixo pro pool inteiro, com `download`/`mat_test` sobrescrevendo pra cima quando precisarem de mais recurso).
