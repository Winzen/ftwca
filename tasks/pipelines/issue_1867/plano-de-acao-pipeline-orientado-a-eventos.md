# Plano de ação — pipeline orientado a eventos

**Referência:** [[Nova arquitetura de flows — pipeline orientado a eventos]]
**Status:** Em elaboração — 2026-08-28

---

## Contexto rápido

Quebrar o flow monolítico em 3 flows independentes conectados por automações Prefect 3:

```
[check_update] → automação → [flow_download] → automação → [mat_test]
```

Variante check_and_download funde os dois primeiros:

```
[check_and_download] → automação → [mat_test]
```

---

## Decisão de arquitetura: handoff de parâmetros

### Jinja no RunDeployment — parâmetros totalmente dinâmicos

A ação `RunDeployment` server-side herda de `JinjaTemplateAction`. O contexto Jinja disponível em `parameters` inclui `event` — o evento disparador completo, com seu `payload`. Verificado no código Prefect 3 instalado (`prefect/server/events/actions.py`, `RunDeployment.render_parameters`).

**Qualquer valor no `payload` do evento disparador pode ser interpolado nos parâmetros da ação:**

```python
# parameters da automação (RunDeployment)
{
    "dataset_id":     "{{ event.payload.dataset_id }}",
    "table_id":       "{{ event.payload.table_id }}",
    "source_url":     "{{ event.payload.source_url }}",
    "reference_date": "{{ event.payload.reference_date }}",
}
```

### Padrão adotado: emit_event com payload arbitrário

Em vez de usar o evento padrão `prefect.flow-run.Completed` (cujo payload não inclui dados do flow), o `check_update` emite um **evento customizado** ao concluir com sucesso:

```python
from prefect.events import emit_event

# dentro do check_update, quando há dado novo:
emit_event(
    event="bd.pipeline.data-available",
    resource={"prefect.resource.id": f"pipeline/{dataset_id}/{table_id}"},
    payload={
        "dataset_id":     dataset_id,
        "table_id":       table_id,
        "source_url":     url,           # pode ser None se não aplicável
        "reference_date": "2026-08",     # qualquer chave variável que o downstream precise
        # datasets específicos podem adicionar chaves extras — a automação ignora o que não usa
    },
)
```

A automação dispara em `bd.pipeline.data-available` filtrado pelo `resource.id`, e passa o payload via Jinja ao `flow_download`. Se não há dado novo, o `check_update` simplesmente **não emite o evento** — sem evento, sem disparo, sem downstream.

**GCS não é necessário para handoff de parâmetros.** Sem latência de leitura, sem arquivo para gerenciar, sem risco de dado desatualizado. O evento carrega os dados junto com o sinal.

---

## Fases do plano

### Fase 0 — PoC com um dataset (antes de qualquer migração em escala)

**Objetivo:** validar as mecânicas de ponta a ponta antes de comprometer com a refatoração em escala.

Dataset piloto sugerido: **br_ibge_ipca** (variante padrão, check simples via API IBGE, dbt trivial).

Tarefas:
- [ ] Implementar `emit_pipeline_data_available()` em `pipelines/utils/flows.py`
- [ ] Criar o `check_update` do br_ibge_ipca usando o helper
- [ ] Criar o `flow_download` do br_ibge_ipca (lógica de download + upload GCS, sem dbt)
- [ ] Criar o `mat_test` genérico (dbt run+test dev → prod → metadata)
- [ ] Criar as 2 automações no Prefect 3:
  - `bd.pipeline.data-available` para `pipeline/br_ibge_ipca/mes_brasil` → dispara `flow_download`
  - `bd.pipeline.data-available` para `pipeline/br_ibge_ipca/mes_brasil` após download → dispara `mat_test`
  - ⚠️ Definir qual evento o `flow_download` emite ao terminar (mesmo padrão `emit_pipeline_data_available` ou evento diferente?)
- [ ] Validar que a automação interpola corretamente `{{ event.payload.reference_date }}` nos parâmetros do flow_download
- [ ] Validar que `mat_test` roda dbt run+test em dev e depois em prod no mesmo pod
- [ ] Rodar o pipeline completo de ponta a ponta em ambiente de dev

**Critério de saída:** pipeline br_ibge_ipca rodando pelos 3 flows com automação, sem intervenção manual entre etapas.

---

### Fase 1 — Infraestrutura base (paralela ao PoC ou logo após)

**Objetivo:** construir as peças compartilhadas que todos os datasets vão usar.

- [ ] **`emit_pipeline_data_available()`** em `pipelines/utils/flows.py` — centraliza nome do evento, formato do resource e guard `has_new_data`
- [ ] **`mat_test` genérico** — flow parametrizado por `dataset_id` e `table_id` que roda dbt run+test em dev → prod → atualiza metadados no backend
- [ ] **Script de criação de automações** — dado um `dataset_id`/`table_id`, cria as automações no Prefect 3 com filtro por `resource.id`; idempotente
- [ ] **Atualização do `deploy_flows.py`** — suportar deploy de 3 flows por dataset com os nomes corretos (`check_update__{id}`, `flow_download__{id}`, mais o `mat_test` genérico compartilhado)
- [ ] **Decidir granularidade das automações** — por tabela (mais automações, mais granular, certo para datasets com múltiplas tabelas independentes) ou por dataset

---

### Fase 2 — Migração gradual dos datasets

**Estratégia:** migrar em paralelo com os flows antigos. O flow antigo continua ativo; o novo pipeline só vai para produção após validação. Após validação, o flow antigo é desativado.

**Ordem sugerida:**

1. **Variante padrão simples** (check via API, sem download pesado) — br_ibge_inpc, br_ibge_ipca15, br_bcb_agencia, br_bcb_estban
2. **Variante padrão com FTP/WebDAV** — br_rf_cnpj, br_rf_cafir, br_me_caged, br_ms_cnes/sia/sih
3. **Variante check_and_download** — br_cgu_beneficios_cidadao, br_cgu_cartao_pagamento, br_cgu_sancoes, br_anatel
4. **Datasets pesados / casos especiais** — au_geoscape_gnaf (16 GB), br_me_cnpj, br_tse_eleicoes

Tarefas por dataset:
- [ ] Extrair a lógica de check para o `check_update` (ou `check_and_download`)
- [ ] Extrair a lógica de download para o `flow_download`
- [ ] Apontar o `mat_test` genérico com os parâmetros do dataset
- [ ] Criar as automações via script
- [ ] Validar pipeline completo em dev
- [ ] Ativar em prod e monitorar por 1 semana
- [ ] Desativar o flow monolítico antigo

---

### Fase 3 — Datasets sem check e datasets inativos

- **Sem check (sempre executam):** br_me_siconfi, br_me_rais, br_senado_dados_abertos etc. — nesses não há `check_update`; o pipeline começa direto no `flow_download` agendado
- **Inativos:** deixar como estão até houver plano de reativação

---

### Fase 4 — Reorganização de pastas (issue #1705)

Só após toda a migração estar concluída e estável. A nova estrutura de pastas faz sentido depois que a estrutura de flows está definida.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Automação dispara mesmo sem dado novo | `check_update` não emite o evento quando não há dado — sem evento, automação não dispara |
| Pod startup overhead torna o pipeline mais lento | Overhead de ~1 min em pipelines completos; irrelevante para pipelines diários. Monitorar nos primeiros datasets migrados |
| `mat_test` genérico não parametriza o dbt corretamente para todos os datasets | Validar no PoC com br_ibge_ipca; casos especiais mantêm um `mat_test` específico |
| Datasets com múltiplas tabelas (br_me_cnpj: empresas/socios/estabelecimentos) | Automações por tabela (resource.id inclui table_id) — cada tabela é um pipeline independente |
| Flow antigo e novo rodando em paralelo durante migração | Migrar um dataset por vez; desativar o schedule do flow antigo antes de ativar o novo |

---

## Estado atual — 2026-08-28

### O que já foi implementado

- [x] `pipelines/utils/flows.py` — helpers criados e importando corretamente:
  - `emit_pipeline_data_available(dataset_id, table_id, has_new_data, **extra)` — check_update → flow_download
  - `emit_pipeline_download_complete(dataset_id, table_id, **extra)` — flow_download → mat_test
- [x] `pipelines/datasets/test_pipeline_eventos/flows.py` — 3 flows de teste sem lógica real:
  - `check_update__test_pipeline_eventos` — `force_new_data=True` por padrão, emite evento
  - `flow_download__test_pipeline_eventos` — imprime params recebidos, emite próximo evento
  - `mat_test__test_pipeline_eventos` — imprime params recebidos, termina

### Próximo passo (retomar aqui)

1. **Deploy dos 3 flows de teste no Prefect 3** — fazer o deploy de `check_update`, `flow_download` e `mat_test` do test_pipeline_eventos
2. **Criar as 2 automações no Prefect 3:**
   - Evento `bd.pipeline.data-available` para resource `pipeline/test_pipeline_eventos/tabela_teste` → dispara `flow_download__test_pipeline_eventos` com `{{ event.payload.dataset_id }}`, `{{ event.payload.table_id }}`, `{{ event.payload.reference_date }}`
   - Evento `bd.pipeline.download-complete` para resource `pipeline/test_pipeline_eventos/tabela_teste` → dispara `mat_test__test_pipeline_eventos` com os mesmos campos
3. **Rodar `check_update` manualmente** e confirmar que os 3 flows rodam em sequência via automação
