
# Configuração Inicial do MCP (Base dos Dados)

Passos realizados para integrar o servidor MCP do Base dos Dados ao Claude Code.

**Atualizado em 2026-08-24** — conferido contra o estado real do repositório na branch `refactor/oficializar-estrutura` (já pushada pro `origin`). Ver [[refactor-oficializar-estrutura]] pro contexto da reorganização em `src/databasis_mcp/`.

---

## 1. Clonar o repositório

```bash
git clone https://github.com/basedosdados/mcp.git /caminho/para/repositorios/mcp
```

---

## 2. Criar as credenciais do backend

O servidor MCP lê credenciais em `~/.basedosdados/credentials.json` (nome preferido hoje — `backend_credentials.json` ainda é aceito como fallback legado, nessa ordem de prioridade).

Caso o diretório `~/.basedosdados` pertença ao root, é necessário criar o arquivo com sudo e transferir a propriedade:

```bash
sudo touch ~/.basedosdados/credentials.json
sudo chown $USER ~/.basedosdados/credentials.json
```

Conteúdo do arquivo — formato preferido, com Token de backend (`bdtoken_...`, gerado em Django admin → Account Auth → Tokens):

```json
{
  "dev": { "token": "bdtoken_..." },
  "staging": { "token": "bdtoken_..." },
  "prod": {
    "token": "bdtoken_...",
    "prefect3": "bdtoken_..."
  }
}
```

Formato legado (email/senha), ainda suportado como fallback:

```json
{
  "dev": { "email": "seu@email.com", "password": "<senha_dev>" },
  "staging": { "email": "seu@email.com", "password": "<senha_staging>" },
  "prod": { "email": "seu@email.com", "password": "<senha_prod>" }
}
```

**Atenção com a chave `prefect3`** (usada pelas tools de Prefect, `list_flow_runs`/`get_flow_run_logs`/etc.): não é uma "Prefect API key" — é um **Token de backend** emitido especificamente pro domínio `prefect3.basedosdados.org` (o nginx desse domínio autentica via `backend.basedosdados.org/auth/`, então o token precisa estar escopado pra ele). A chave antiga `"prefect"` (escopada pro host do Prefect 2) **não funciona** aqui — o próprio código rejeita com redirect pra tela de login se você usar o token errado.

---

## 3. Configurar ambiente Python com uv

**Resolvido em 2026-08-25** — `pyproject.toml`/`uv.lock` commitados na branch `refactor/oficializar-estrutura` (commit `6cde98c`, já pushado), `requirements.txt` removido (redundante e desatualizado). Pra um clone novo, só:

```bash
cd /caminho/para/repositorios/mcp
uv sync
```

Isso cria o `.venv` e instala tudo já fixado no `uv.lock` (`fastmcp>=2.0`, `requests>=2.31`, `google-cloud-bigquery>=3.41.0`).

> **Atenção:** o projeto foi renomeado de `mcp` para `databasis-mcp` no `pyproject.toml` para evitar conflito com a dependência interna `mcp` do pacote `fastmcp`.
>
> **`google-cloud-bigquery` é obrigatório** — sem ele, as tools `query_bigquery`, `preview_table` e `audit_partition_metadata` (em `src/databasis_mcp/tools/bigquery.py`) não funcionam. Já está no `pyproject.toml`.

Validado: `uv sync` limpo, `src/databasis_mcp` importa e registra as 45 tools (mesma contagem do `server.py` da raiz — ver [[refactor-oficializar-estrutura]] pro histórico de duas lacunas de sincronização já corrigidas).

---

## 4. Configurar o Claude Code

Adicionar o servidor MCP em `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "databasis": {
      "type": "stdio",
      "command": "/caminho/para/repositorios/mcp/.venv/bin/python",
      "args": ["/caminho/para/repositorios/mcp/server.py"]
    }
  }
}
```

Após salvar, reiniciar o Claude Code ou rodar `/mcp` para reconectar.

**Nota (2026-08-24):** essa configuração aponta pro `server.py` da raiz do repositório (o monolito original, 3527 linhas). Ele continua funcional e sincronizado com o código novo em `src/databasis_mcp/` (conferido: o fix mais recente, exposição do campo `observations`, está presente nos dois). É uma configuração **transitória** — o plano do refactor (ver [[refactor-oficializar-estrutura]], item 4) prevê registrar um entry point em `pyproject.toml` e simplificar essa entrada pra:

```json
{
  "command": "uv",
  "args": ["--directory", "/caminho/para/repositorios/mcp", "run", "databasis-mcp"]
}
```

Isso ainda não foi feito — atualizar esse doc quando o entry point for adicionado.

**Pendência (2026-08-26):** o registro do `databasis` nesta máquina está em escopo *local* (só ativo dentro do repo `mcp`), não em escopo *user* — por isso não aparece em outros repositórios (`pipelines`, `backend`, etc.). Ver [[Escopo do mcpServers entre projetos]] pro diagnóstico completo e a correção pendente.
