**Status:** Pendente — diagnosticado em 2026-08-26, ainda não corrigido nesta máquina nem comunicado pro time.

---

## O problema

O MCP `databasis` só aparece ativo quando o Claude Code é aberto dentro do repo `/mnt/d/repositorios/bd/mcp` (ou `test_mcp`). Em qualquer outro repositório (`pipelines`, `backend`, `iac`, etc.) ele não conecta.

## Causa

`mcpServers` dentro de `settings.json`/`settings.local.json` **não é o que o Claude Code realmente usa** pra decidir se um MCP está ativo num diretório — isso é lido de `~/.claude.json`, que tem dois lugares possíveis:

- bloco `mcpServers` no **nível raiz** do arquivo → escopo **user**, global, aplica em qualquer diretório
- bloco `projects["<caminho>"].mcpServers` → escopo **local**, só vale naquele repositório específico

Nesta máquina, o `databasis` só está registrado no escopo **local**, dentro de `projects["/mnt/d/repositorios/bd/mcp"]`. O bloco raiz (user) está vazio. Por isso ele nunca aparece fora dali.

Tem ainda uma bagunça extra: existem **três definições divergentes** do mesmo servidor `databasis` espalhadas por três arquivos diferentes:

1. `~/.claude/settings.json` (global) — `uv --directory /mnt/d/repositorios/bd/mcp run server.py`
2. `~/.claude.json` → `projects["/mnt/d/repositorios/bd/mcp"].mcpServers` (o registro que realmente está ativo) — `uv --directory ... run server.py`, praticamente igual à (1)
3. `mcp/.claude/settings.local.json` (arquivo local do repo, não versionado) — aponta direto pro Python de uma venv (`/home/winzen/.local/share/venvs/databasis-mcp/bin/python`), sem passar pelo `uv`

Nenhuma dessas três é a fonte "oficial" sozinha — (1) e (3) nem são o mecanismo que o Claude Code usa pra ativar por projeto; só (2) é.

## Precedência real (quando há mais de uma definição do mesmo nome)

local (`~/.claude.json` → `projects[path].mcpServers`) > projeto (`.mcp.json` versionado no repo) > user (`~/.claude.json`, bloco raiz) > plugin > conectores remotos.

É override total por nome do servidor, não merge de campos.

## Diagnóstico rápido

```bash
claude mcp get databasis
```
Mostra em qual escopo a definição ativa está.

## Correção (ainda não aplicada)

Registrar em escopo **user**, que é o que de fato se propaga pra qualquer repositório:

```bash
claude mcp add --scope user --command uv --args "--directory /mnt/d/repositorios/bd/mcp run server.py" databasis
```

E limpar as definições redundantes/divergentes:
- remover `mcpServers.databasis` de `~/.claude/settings.json` (não é lido pra esse fim, só confunde)
- remover `mcpServers.databasis` de `mcp/.claude/settings.local.json` (a versão sem `uv`, mais desatualizada das três)

## Recomendação pro time

Não faz sentido orientar todo mundo a abrir o Claude Code sempre dentro do repo `mcp` achando que isso "concentra as ferramentas" — não é assim que funciona. O caminho certo:

- **Cada pessoa registra o MCP uma vez, em escopo user** (comando acima, ou via fluxo `/mcp` dentro do Claude Code) — depois disso funciona em qualquer repositório que ela abrir, sem precisar reconfigurar nem começar pelo `mcp`.
- Alternativa pra compartilhar via git (escopo *project*, funciona só naquele repo): commitar um `.mcp.json` na raiz de cada repositório onde o time for usar o MCP. Não existe hoje nenhum `.mcp.json` em `mcp`, `pipelines` ou `backend`.

Ver [[Configuração Inicial]] pro resto do setup (credenciais, `uv sync`).
