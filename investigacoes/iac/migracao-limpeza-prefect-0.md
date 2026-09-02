# Limpeza do Prefect 0 — 2026-06-15

Auditoria e desativação de todos os agendamentos e runs pendentes remanescentes no Prefect 0 (`https://prefect.basedosdados.org`).

---

## Contexto

Durante a migração para o Prefect 3, o Prefect 0 não foi desligado — os workers continuavam ativos e flows com `is_schedule_active: true` continuavam executando. Além disso, flows que já estavam com schedule desativado (`is_schedule_active: false`) ainda tinham runs pré-criados no estado `Scheduled` na fila, que poderiam ser pegos pelos workers a qualquer momento.

---

## O que foi feito

### 1. Desativação dos 3 flows com schedule ativo

Três flows ainda estavam com `is_schedule_active: true` e executaram com sucesso no próprio dia 2026-06-15:

| Flow | Schedule | Labels |
|---|---|---|
| `Limpeza de histórico de runs` | a cada 86400s (diário) | basedosdados-dev |
| `br_bndes_operacoes_contratadas__operacoes_nao_automaticas` | `00 6,18 * * 1` (segunda-feira) | basedosdados |
| `cross_update.update_metadata_table` | `0 8 * * *` (diário) | basedosdados |

Ação: chamada `set_schedule_inactive` via GraphQL para cada um.

```graphql
mutation {
  set_schedule_inactive(input: { flow_id: "<id>" }) {
    success
  }
}
```

---

### 2. Limpeza dos runs `Scheduled` pendentes

12 flows inativos tinham runs pré-criados no estado `Scheduled` que ainda poderiam ser executados pelos workers:

| Flow | Runs pendentes |
|---|---|
| `br_camara_dados_abertos.licitacao_contrato` | 44 |
| `br_bd_indicadores.contabilidade` | 30 |
| `br_me_cnpj.estabelecimentos` | 28 |
| `br_bcb_taxa_cambio.taxa_cambio` | 29 |
| `br_cvm_administradores_carteira.responsavel` | 20 |
| `br_me_cnpj.socios` | 19 |
| `br_cvm_administradores_carteira.pessoa_fisica` | 7 |
| `br_bd_indicadores.receitas_planejadas` | 9 |
| `br_cvm_oferta_publica_distribuicao.dia` | 4 |
| `br_cvm_administradores_carteira.pessoa_juridica` | 5 |
| `br_cgu_emendas_parlamentares.microdados` | 3 |
| `br_ibge_pnadc.microdados` | 2 |

**Método:** toggle de schedule (ativar → aguardar 5s → desativar) limpa os runs pré-criados automaticamente. Para os runs que resistiram ao toggle (`br_cvm_*`), foi usado `cancel_flow_run` diretamente.

```graphql
mutation {
  cancel_flow_run(input: { flow_run_id: "<id>" }) {
    state
  }
}
```

---

## Estado final

| Métrica | Antes | Depois |
|---|---|---|
| Flows com `is_schedule_active: true` | 3 | **0** |
| Runs no estado `Scheduled` | ~200 | **0** |

Nenhum flow executa mais no Prefect 0.

---

## Referências

- API GraphQL Prefect 0: `https://prefect.basedosdados.org/api/graphql`
- Auth: `~/.prefect/auth.toml` → `api_key` para `prefect.basedosdados.org`
- Tenant ID: `a0191a60-3d87-4aa1-b587-b45968cd6e53`
