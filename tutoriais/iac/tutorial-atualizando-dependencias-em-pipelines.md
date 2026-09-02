# Tutorial — Atualizando dependências em pipelines

O repositório usa **uv** para gerenciar dependências Python e **dbt packages** para pacotes dbt. São dois fluxos separados.

---

## Dependência Python (`pyproject.toml`)

**1. Atualize a dependência**
```bash
# Atualiza para uma versão específica (edita o pyproject.toml E atualiza o uv.lock)
uv add "requests>=2.31.0"

# Ou, para atualizar para a versão mais recente compatível com os constraints existentes
uv lock --upgrade-package requests
```

O `uv add` é o ideal — ele edita o `pyproject.toml`, resolve conflitos e atualiza o `uv.lock` em um único comando. Se houver conflito com outra dependência, falha aqui com uma mensagem clara antes de qualquer instalação.

**2. Aplique no ambiente local**
```bash
uv sync
```

**3. Verifique que nada quebrou**
```bash
uv run pytest
# ou teste o flow específico que usa a dependência
```

**4. Commit dos dois arquivos juntos**
```bash
git add pyproject.toml uv.lock
git commit -m "chore: atualiza requests para 2.31.0"
```

> O `uv.lock` **sempre** deve ser commitado junto com o `pyproject.toml` — ele é o que garante reprodutibilidade no CI e no Docker build.

---

## Pacote dbt (`packages.yml`)

**1. Edite a versão no `packages.yml`**
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.4.1   # era 1.1.1
```

**2. Reinstale os pacotes**
```bash
uv run dbt deps
```

**3. Verifique que os modelos ainda compilam**
```bash
uv run dbt compile
```

**4. Commit dos arquivos alterados**
```bash
git add packages.yml package-lock.yml
git commit -m "chore: atualiza dbt_utils para 1.4.1"
```

---

## O que o CI valida automaticamente

O workflow de build Docker (`build-docker-prefect3.yaml`) roda `dbt deps` durante o build — se o `packages.yml` estiver inconsistente, o build falha antes de chegar em produção. Para a imagem de dev, basta adicionar a label `deploy-flow` no PR para disparar o build e validar antes do merge.
