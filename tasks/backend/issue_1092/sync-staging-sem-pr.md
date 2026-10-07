# Sync staging with main gerava falhas falsas de CI

**Repositório**: `backend`. **Issue**: [basedosdados/backend#1092](https://github.com/basedosdados/backend/issues/1092). **PR**: [basedosdados/backend#1093](https://github.com/basedosdados/backend/pull/1093). **Data**: 2026-09-17.

## O pedido original

A action "Sync staging with main" estava causando actions ficarem "bugadas" — exemplo apontado: [run 35188466182](https://github.com/basedosdados/backend/actions/runs/35188466182), onde `CI Python` falhou com a mensagem genérica "likely failed because of a workflow file issue" em cima de um commit que na verdade já tinha sido sincronizado com sucesso.

## Diagnóstico

`.github/workflows/sync-staging.yaml` (que automatiza o processo manual de trazer `main` pra `staging` via `chore/main_to_staging`) dispara em todo push pra `main`:

1. Cria a branch `chore/main_to_staging` a partir de `staging`.
2. `git merge origin/main -X ours` (prioriza `staging` em caso de conflito).
3. Abre um PR pra `staging`.
4. Mescla o PR e deleta a branch — **imediatamente**, sem esperar nenhum check.

O passo 4 sempre existiu assim (o workflow nunca mudou desde que foi criado, PR #1070). O que mudou foi o **tempo entre abrir e mesclar o PR**, que passou a levar só 3-5 segundos nas sincronizações recentes (desde 2026-09-02), contra minutos/horas nas anteriores (até 2026-08-20). Com o merge quase instantâneo, a branch `chore/main_to_staging` é deletada antes de qualquer workflow disparado por `pull_request` (`CI Python`, `CI Pull Request`) conseguir sequer começar a rodar — o GitHub não consegue agendar o job numa branch que já sumiu, e reporta a falha genérica e enganosa em cima de commits que já foram sincronizados corretamente.

Confirmado olhando o histórico de execuções (`gh api repos/.../actions/runs?branch=chore/main_to_staging`): todas as 5 sincronizações desde 2026-09-02 tiveram `CI Python`/`CI Pull Request` falhando ou em `action_required`; todas as anteriores tiveram sucesso.

**Impacto real**: cosmético/ruído. `staging` não tem proteção de branch, então a sincronização em si sempre funcionava — o problema era só os checks ❌ vermelhos permanentes em cima de commits já mesclados.

## A pergunta que destravou a solução certa

Em vez de só mascarar o sintoma (cogitado inicialmente: `branches-ignore: [chore/main_to_staging]` nos 4 workflows que reagem a `pull_request`), o usuário perguntou se a automação precisava mesmo abrir PR pra funcionar. Resposta: não.

- `staging` não tem proteção de branch — nada trava o merge esperando checks.
- O merge usa `-X ours`, que nunca deixa conflito real pra revisar — sempre resolve sozinho a favor de `staging`.
- O único valor real do PR era a lista "PRs trazidos de main" no título, preservável na mensagem do commit.

## Solução

Trocado o fluxo pra `git merge origin/main -X ours` + `git push origin HEAD:staging` (push normal, sem `--force`) direto em cima do checkout de `staging`, sem criar branch nem PR. A lista de PRs continua igual, agora na mensagem do commit de merge.

Sem branch efêmera, o evento `pull_request` nunca mais é disparado por essa sincronização — resolve a corrida na raiz, não só esconde o sintoma.

## Teste

Sem tocar no GitHub real: simulado o fluxo completo (`git init --bare`) num repositório local isolado, semeado com o estado real de `staging`/`main`.

| Cenário | Resultado |
|---|---|
| `has_changes` com `staging` já sincronizado | `false` |
| `has_changes` com commit novo simulado em `main` | `true` |
| Extração da lista de PRs (`feat: ... (#9999)`) | `PRS=#9999`, incorporado corretamente na mensagem final |
| Merge + push normal (sem `--force`) | Sucesso |
| Segunda rodada, sem mudanças | Idempotente, não faz nada |
| Merge `-X ours` com conflito de conteúdo real na mesma região do arquivo | Resolve sozinho, sem marcador de conflito sobrando |
