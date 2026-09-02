# Hospedagem da documentação interna

Comparativo das opções para publicar o `manual-equipe-dados` online sem criar um novo registro DNS.

**Repositório:** https://github.com/basedosdados/manual-equipe-dados

---

## Resumo comparativo

| Opção | URL resultante | Esforço | Acesso restrito? | Infra nova? |
|---|---|---|---|---|
| GitHub Pages | `basedosdados.github.io/manual-equipe-dados` | Baixo | Sim — membros da org GitHub | Não |
| Subpath no ingress K8s | `basedosdados.org/manual/` | Médio | Não (público) | Pod nginx + regra no ingress |
| Subpath via Django | `basedosdados.org/manual/` | Médio | Sim — login do admin Django | Não |

---

## Opção 1 — GitHub Pages

Deploy estático via `mkdocs gh-deploy` no branch `gh-pages`.

### Passos para ativar

1. Criar `.github/workflows/deploy-docs.yml` com `mkdocs gh-deploy --force` disparado no push para `main`
2. No GitHub: **Settings → Pages → Source** → selecionar branch `gh-pages`
3. Fazer push de qualquer commit em `main` para disparar o primeiro deploy
4. Acessar `https://basedosdados.github.io/manual-equipe-dados`

### Vantagens

- Zero infraestrutura nova — tudo no GitHub
- Deploy automático a cada commit em `main`
- Acesso restrito a membros da org (repo privado)
- O `mkdocs.yml` já tem o `site_url` correto

### Limitações

- URL fora do domínio `basedosdados.org`
- Requer conta GitHub para acessar (se repo privado)

> **Pré-requisito:** repositório privado + GitHub Team/Pro para GitHub Pages em repos privados (ou tornar o repositório público).

---

## Opção 2 — Subpath no ingress K8s existente

Pod nginx com os arquivos estáticos do MkDocs, exposto por uma nova regra de path no ingress já existente.

### Passos para ativar

1. Adicionar step de `mkdocs build` no CI do repositório, gerando a pasta `site/`
2. Criar imagem Docker com nginx servindo a pasta `site/` e publicar no Container Registry
3. Adicionar Deployment + Service no cluster K8s (namespace a definir)
4. Adicionar regra no ingress existente: `path: /manual/` apontando para o Service criado
5. Configurar `base_url: /manual/` no `mkdocs.yml` para que os assets carreguem corretamente no subpath

### Vantagens

- URL no domínio próprio: `basedosdados.org/manual/`
- Sem novo DNS — só nova regra no ingress já existente
- Sem dependência do GitHub para acesso

### Limitações

- Conteúdo público — qualquer pessoa com a URL acessa
- Requer manter uma imagem Docker e um pod rodando
- Deploy mais complexo: CI → imagem → rollout no K8s

> **Pré-requisito:** acesso ao IAC para editar o ingress e criar os recursos K8s. Para restringir acesso, seria necessário adicionar um `oauth2-proxy` na frente do pod.

---

## Opção 3 — Subpath via Django (backend)

Arquivos estáticos do MkDocs servidos pelo Django, protegidos pelo login do admin.

### Passos para ativar

1. Rodar `mkdocs build` no CI e copiar a pasta `site/` para dentro do repositório do backend
2. Configurar o Django para servir os arquivos em `/manual/` usando `django.views.static.serve`
3. Proteger a rota `/manual/` com `@login_required` ou verificação de sessão Django
4. Configurar `base_url: /manual/` no `mkdocs.yml`
5. Adicionar script no CI do `manual-equipe-dados` que abre PR no backend com os novos arquivos estáticos a cada push em `main`

### Vantagens

- URL no domínio próprio: `basedosdados.org/manual/`
- Acesso restrito pelo login Django — sem infraestrutura extra
- Sem novo pod ou imagem Docker

### Limitações

- Arquivos estáticos versionados junto com o código do backend
- Atualizar a doc exige abrir PR no repositório errado
- O Django em produção delega static files para CDN/S3 — `django.views.static.serve` requer cuidado em produção

> **Pré-requisito:** acesso ao repositório backend para adicionar a rota e os arquivos estáticos.

---

## Recomendação

Comece com a **Opção 1 (GitHub Pages)**: zero infraestrutura nova, deploy automático configurado, e o controle de acesso via repositório privado é suficiente para documentação interna. Se a equipe precisar de uma URL em `basedosdados.org` ou de acesso sem conta GitHub, migre para a Opção 2.
