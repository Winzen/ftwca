# MCP — Refatoração: Oficializar Estrutura do Repositório

**Status:** Branch pushada pro `origin` (`origin/refactor/oficializar-estrutura`) — atualizado em 2026-08-25.

- `255ee82` — reorganização do `server.py` em `src/databasis_mcp/`
- `6cde98c` — `pyproject.toml`/`uv.lock` commitados (antes só existiam localmente, não versionados); `requirements.txt` removido; `README.md` atualizado pra `uv sync`; limpeza de arquivos soltos (`.venv-pip/`, scaffold do `uv init`)
- `6434ca7` — porta `set_deployment_schedule_active` pro pacote novo (tinha chegado só no `server.py` da raiz via PR #9)
- `a377284` — porta `run_deployment` e o helper `_prefect_get` pro pacote novo (lacuna mais antiga, desde o merge do PR #8: essa tool tinha entrado direto em `main` antes da branch do refactor ser cortada)

PR aberto: [basedosdados/mcp#10](https://github.com/basedosdados/mcp/pull/10). Com isso, `server.py` e `src/databasis_mcp/` batem nas 45 tools.

As 4 pendências abaixo continuam pendentes (nenhuma delas foi feita ainda: sem `[project.scripts]` no `pyproject.toml`, sem `.github/workflows/lint.yaml`, sem `CONTRIBUTING.md`, `settings.json` do Claude Code ainda aponta pro `server.py` da raiz).
**Branch:** `refactor/oficializar-estrutura` em `basedosdados/mcp`
**Issue pipelines:** [#1670](https://github.com/basedosdados/pipelines/issues/1670)
**Issue MCP:** [#6](https://github.com/basedosdados/mcp/issues/6)

---

## O que foi feito

O `server.py` original (3527 linhas, 43 tools) foi dividido em módulos por domínio dentro de `src/databasis_mcp/`:

```
src/databasis_mcp/
  __init__.py
  _app.py          # instância FastMCP + constante URLS
  auth.py          # _cache, _IDS_TTL, _get_credentials, _get_token
  gql.py           # _gql, _mut, _strip_id, _lookup_directory_column, _fetch_all
  server.py        # entry point: importa módulos de tools, chama mcp.run()
  tools/
    __init__.py
    metadata.py    # auth, discover_ids, lookup_id, list_datasets, get_dataset,
                   # get_authenticated_account, search_datasets (7 tools)
    write.py       # reorder_*, create_update_*, upload_columns,
                   # bulk_upsert_columns, update_column, delete_* (25 tools)
    bigquery.py    # preview_table, query_bigquery, audit_partition_metadata (3 tools)
    prefect.py     # list_flow_runs, get_flow_run_logs,
                   # get_failed_flow_runs, trigger_dbt_model (4 tools)
```

**Verificado:** `asyncio.run(mcp.list_tools())` retornou as 43 tools — nenhuma perdida.

### Arquitetura de imports (sem circular)

```
tools/* → gql → auth → _app (FastMCP)
             ↗
tools/* → auth
```

`_app.py` exporta `mcp`. Cada tool module faz `from .._app import mcp` e decora as funções com `@mcp.tool()`. `server.py` só importa os módulos para forçar o registro e chama `mcp.run()`.

---

## Pendências (próximos passos do PR)

### 1. `pyproject.toml` — entry point

Adicionar em `[project.scripts]` para que seja possível rodar com `uv run databasis-mcp`:

```toml
[project.scripts]
databasis-mcp = "databasis_mcp.server:main"
```

Isso requer adicionar uma função `main()` em `src/databasis_mcp/server.py`:
```python
def main():
    mcp.run()
```

E ajustar o `pyproject.toml` para reconhecer o pacote em `src/`:
```toml
[tool.setuptools.packages.find]
where = ["src"]
```

### 2. CI — GitHub Actions com ruff

Criar `.github/workflows/lint.yaml`:
```yaml
name: lint
on: [push, pull_request]
jobs:
  ruff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/ruff-action@v1
        with:
          args: check src/
```

### 3. `CONTRIBUTING.md`

Cobrir:
- Setup com `uv` (`uv sync`, `uv run python src/databasis_mcp/server.py`)
- Onde adicionar uma nova tool (qual arquivo de `tools/`, como importar `mcp` e os helpers)
- Como testar localmente (configurar `~/.basedosdados/credentials.json`, conectar ao Claude Code via `settings.json`)
- Estrutura de módulos e responsabilidades de cada um

### 4. Atualizar `settings.json` do Claude Code

Após o entry point estar configurado, a entrada no `~/.claude/settings.json` poderá ser simplificada de:
```json
{
  "command": "/path/to/mcp/.venv/bin/python",
  "args": ["/path/to/mcp/server.py"]
}
```
para:
```json
{
  "command": "uv",
  "args": ["--directory", "/path/to/mcp", "run", "databasis-mcp"]
}
```
