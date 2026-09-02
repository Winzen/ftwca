# Issue 1511 — Execução e Resultado

**Data:** 2026-04-27  
**Repositório:** basedosdados/pipelines  
**Issue:** https://github.com/basedosdados/pipelines/issues/1511

---

## O que foi feito

### Problema
Os metadados de partição e clusterização das tabelas no BigQuery estavam desatualizados na API da Base dos Dados. O campo `isPartition` nas colunas não refletia a realidade dos modelos dbt no repositório de pipelines.

### Solução
Construímos e iteramos sobre a tool `audit_partition_metadata` no MCP server em duas etapas:

**Etapa 1 — Audit e correção (2-way: SQL ↔ API)**

1. Lê os arquivos `.sql` dos modelos dbt em `basedosdados/pipelines/models/`
2. Extrai `partition_by.field` e `cluster_by` de cada `config()` — ambos tratados como `isPartition=True`
3. Compara com o estado atual da API via GraphQL
4. Corrige as discrepâncias usando `update_column`
5. Gera um relatório `.md` com todas as mudanças agrupadas por dataset/tabela

**Etapa 2 — Validação BigQuery (3-way: SQL · BQ · API)**

Após aplicar as correções, adicionamos validação cruzada com o BigQuery para confirmar que os metadados da API estão alinhados com a tabela materializada real:

1. Para cada tabela auditada, consulta `INFORMATION_SCHEMA.COLUMNS` do BQ
2. Verifica `is_partitioning_column` e `clustering_ordinal_position`
3. Compara 3 fontes: SQL dbt config · BQ materializado · API metadado
4. Reporta divergências SQL↔BQ e BQ↔API separadamente

### Mudanças no MCP server (`server.py`)

| O que | Onde | Descrição |
|---|---|---|
| `isPartition` nas queries de leitura | `get_dataset` | Passou a retornar `is_partition` em cada coluna |
| `_get_tables_for_audit()` | função interna | Busca tabelas via `allTable(cloudTables_GcpDatasetId:)` — sem dependência de slug |
| `_parse_sql_partitions()` | função interna | Parseia todos os `.sql` e retorna mapa de colunas partition/cluster por tabela BQ |
| `_write_partition_report()` | função interna | Gera relatório `.md` com tabela de mudanças e seção de validação BQ |
| `_bq_get_partition_map()` | função interna | Consulta `INFORMATION_SCHEMA.COLUMNS` no BQ para múltiplos datasets |
| `audit_partition_metadata` | tool MCP | Orquestra tudo: audit + fix + relatório + validação BQ |

#### Refactoring da query de busca de tabelas

A query original usava `allDataset(slug: $slug)` (match exato por slug). Isso causava falha silenciosa quando slugs parciais eram passados (ex: `"frota"` não encontrava `"br_denatran_frota"`).

A nova query usa `allTable(cloudTables_GcpDatasetId: $gcpDatasetId)`, que:
- Busca diretamente pelo GCP dataset ID (a mesma chave já presente no SQL)
- Não depende do slug da API
- Funciona corretamente com qualquer dataset que tenha cloud tables cadastradas

### Parâmetros da tool `audit_partition_metadata`

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `pipelines_path` | `str` | Caminho local para o repo de pipelines |
| `gcp_dataset_id` | `str \| list[str] \| None` | GCP dataset ID (ex: `"br_denatran_frota"`), lista, ou None para todos |
| `fix` | `bool` | Se True, aplica as correções (requer auth) |
| `env` | `str` | `"dev"` ou `"prod"` |
| `report_path` | `str \| None` | Caminho para gerar o relatório `.md` |
| `validate_bq` | `bool` | Se True, faz validação 3-way com BigQuery INFORMATION_SCHEMA |
| `billing_project` | `str \| None` | Projeto GCP para billing (necessário quando `validate_bq=True`) |

> **Nota:** `billing_project` serve também como projeto de dados para consultar INFORMATION_SCHEMA. Use `"basedosdados-dev"` com a service account de staging para acesso mais amplo.

---

## Resultado da execução

### Etapa 1 — Correção de metadados

**Ambiente:** prod  
**Data:** 2026-04-27

| Métrica | Valor |
|---|---|
| Datasets consultados | 1.181 |
| Tabelas com SQL correspondente | 240 |
| Colunas verificadas | 11.828 |
| **Correções aplicadas** | **157** |
| Erros | 0 |

- **False positives (15):** API tinha `isPartition=True` mas coluna não é partition/cluster no SQL → corrigido para `False`
- **Missing (142):** Coluna era partition/cluster no SQL mas API tinha `isPartition=False` → corrigido para `True`

Relatório detalhado: `D:\Vaults\Begin PC\MCP\audit_partition_fix_2026-04-27.md`

### Etapa 2 — Validação BigQuery (3-way)

**Ambiente:** prod  
**Data:** 2026-04-27  
**Credenciais BQ:** `staging.json` / billing: `basedosdados-dev`

| Métrica | Valor |
|---|---|
| Tabelas verificadas | 240 |
| **Tabelas OK (3 fontes alinhadas)** | **201** |
| Divergências SQL ↔ BQ | 40 |
| Divergências BQ ↔ API | 40 |
| Datasets inacessíveis | 0 |

As 40 divergências SQL↔BQ são todas do padrão "SQL define partition/cluster mas BQ não reflete" — modelos que precisam ser re-executados (rebuild no Prefect/CLI). Não são erros dos metadados.

Dominante: `br_ibge_censo_demografico` — 30+ tabelas de setor censitário 2010 com `sigla_uf` como cluster no SQL mas não materializado no BQ.

Relatório completo: `D:\Vaults\Begin PC\MCP\audit_3way_2026-04-27.md`

---

## Como re-executar no futuro

```python
# Dry-run completo (só relatório, sem alterar)
audit_partition_metadata(
    pipelines_path="/mnt/d/repositorios/bd/pipelines",
    fix=False,
    env="prod",
    report_path="/mnt/d/Vaults/Begin PC/MCP/audit_partition_YYYY-MM-DD.md"
)

# Aplicar correções
audit_partition_metadata(
    pipelines_path="/mnt/d/repositorios/bd/pipelines",
    fix=True,
    env="prod",
    report_path="/mnt/d/Vaults/Begin PC/MCP/audit_partition_fix_YYYY-MM-DD.md"
)

# Validação 3-way completa (sem fix)
audit_partition_metadata(
    pipelines_path="/mnt/d/repositorios/bd/pipelines",
    fix=False,
    env="prod",
    validate_bq=True,
    billing_project="basedosdados-dev",
    report_path="/mnt/d/Vaults/Begin PC/MCP/audit_3way_YYYY-MM-DD.md"
)

# Dataset específico
audit_partition_metadata(
    pipelines_path="/mnt/d/repositorios/bd/pipelines",
    gcp_dataset_id="br_denatran_frota",
    validate_bq=True,
    billing_project="basedosdados-dev",
    env="prod"
)
```

Credenciais backend: `~/.basedosdados/backend_credentials.json` → chave `prod` com `email` e `password`.  
Credenciais BQ: `~/.basedosdados/credentials/staging.json` (ADC via `GOOGLE_APPLICATION_CREDENTIALS`).
