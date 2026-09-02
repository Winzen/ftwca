# test-mcp — Servidor MCP de Teste

**Data:** 2026-04-07  
**Repositório:** `D:\repositorios\bd\test_mcp`

---

## O que é

Um servidor MCP (Model Context Protocol) criado com a biblioteca `FastMCP`, usado para testar a integração entre Claude Code e servidores MCP locais.

---

## Stack

| Item | Detalhe |
|---|---|
| Linguagem | Python ≥ 3.11 |
| Framework | `mcp[FastMCP]` ≥ 1.27.0 |
| Gerenciador de pacotes | `uv` |

---

## Ferramentas expostas

### `somar`
Soma dois números.

- **Parâmetros:** `a: float`, `b: float`
- **Retorno:** `float`

```python
somar(a=3, b=4)  # → 7.0
```

---

### `echo`
Retorna a mesma mensagem enviada.

- **Parâmetros:** `mensagem: str`
- **Retorno:** `str`

```python
echo(mensagem="Olá")  # → "Olá"
```

---

### `info_servidor`
Retorna informações sobre o servidor.

- **Parâmetros:** nenhum
- **Retorno:** `dict`

```python
info_servidor()
# → {
#     "nome": "test-mcp",
#     "versao": "0.1.0",
#     "ferramentas": ["somar", "echo", "info_servidor"],
#     "status": "ok"
# }
```

---

## Estrutura de arquivos

```
test_mcp/
├── server.py        # Definição das ferramentas MCP (FastMCP)
├── main.py          # Entry point simples
├── pyproject.toml   # Dependências e metadados do projeto
└── uv.lock          # Lock file do uv
```

---

## Instalação

### Pré-requisitos

- **Python 3.11+** instalado (o projeto usa `.python-version` com valor `3.11`)
- **uv** instalado — gerenciador de pacotes/ambientes Python moderno

```bash
# Instalar o uv (caso não tenha)
pip install uv
# ou via script oficial:
curl -Lsf https://astral.sh/uv/install.sh | sh
```

---

### 1. Criar o projeto com uv

```bash
uv init test_mcp
cd test_mcp
```

Isso gera a estrutura inicial com `pyproject.toml` e `main.py`.

---

### 2. Adicionar a dependência MCP

```bash
uv add "mcp>=1.27.0"
```

O `uv` cria automaticamente o `.venv` e o `uv.lock` com todas as dependências travadas.

---

### 3. Criar o servidor (`server.py`)

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("test-mcp")

@mcp.tool()
def somar(a: float, b: float) -> float:
    """Soma dois números."""
    return a + b

@mcp.tool()
def echo(mensagem: str) -> str:
    """Retorna a mesma mensagem enviada."""
    return mensagem

@mcp.tool()
def info_servidor() -> dict:
    """Retorna informações sobre este servidor MCP."""
    return {
        "nome": "test-mcp",
        "versao": "0.1.0",
        "ferramentas": ["somar", "echo", "info_servidor"],
        "status": "ok",
    }

if __name__ == "__main__":
    mcp.run()
```

---

### 4. Registrar o servidor no Claude Code

Editar o arquivo `~/.claude/settings.json` (Linux/WSL) e adicionar a entrada em `mcpServers`:

```json
{
  "mcpServers": {
    "test-mcp": {
      "type": "stdio",
      "command": "uv",
      "args": ["--directory", "/mnt/d/repositorios/bd/test_mcp", "run", "server.py"]
    }
  }
}
```

> **Nota WSL:** O caminho deve usar o formato Linux (`/mnt/d/...`). O `uv` é chamado diretamente — ele resolve o ambiente virtual automaticamente pelo `uv.lock`.

---

### 5. Verificar no Claude Code

Reinicie o Claude Code. As ferramentas do servidor aparecem disponíveis com o prefixo `mcp__test-mcp__`:

- `mcp__test-mcp__somar`
- `mcp__test-mcp__echo`
- `mcp__test-mcp__info_servidor`

---

## Como rodar manualmente

```bash
cd /mnt/d/repositorios/bd/test_mcp
uv run server.py
```

---

## O que foi testado no Claude Code

- Carregamento do servidor `test-mcp` via MCP no Claude Code
- Inspeção do schema da ferramenta `somar` com `ToolSearch`
- Chamada interativa da ferramenta `mcp__test-mcp__somar`
