# Label prod/staging/dev automática ao abrir PR

**Repositório**: `backend`. **Issue**: [basedosdados/backend#1094](https://github.com/basedosdados/backend/issues/1094). **PR**: [basedosdados/backend#1095](https://github.com/basedosdados/backend/pull/1095). **Data**: 2026-09-17.

## O pedido original

O repositório já tinha as labels `prod`, `staging` e `dev`, cada uma com a descrição "Indica que o Pull Request está com a branch de destino (base) apontando para main/staging/dev" — mas nada aplicava elas sozinho, precisavam ser adicionadas à mão em cada PR.

## Solução

Novo workflow `.github/workflows/label-pr-by-target-branch.yaml`, disparado em `pull_request: types: [opened]`:

| Branch de destino (`base.ref`) | Label |
|---|---|
| `main` | `prod` |
| `staging` | `staging` |
| `dev` | `dev` |
| qualquer outra | nenhuma |

Usa `gh pr edit --add-label`, sem checkout — só lê `base.ref`/`number` do evento via variáveis de ambiente (evita interpolar `${{ }}` direto no script).

## Bug real encontrado e corrigido durante o teste

Primeira versão falhou na primeira execução real: `gh pr edit` sem um checkout no workspace não consegue inferir o repositório (usa `git remote` internamente pra isso) e falhava com `fatal: not a git repository`. Corrigido definindo `GH_REPO: ${{ github.repository }}` no `env:` do step — variável padrão que o `gh` CLI usa pra resolver o repositório sem precisar de contexto git.

## Testes

- Lógica do `case` (mapeamento branch → label) testada isoladamente pros 4 cenários.
- **Teste real #1**: abrir o PR #1095 (esse próprio, mirando `main`) revelou o bug do `GH_REPO` acima — execução real falhando com erro real, não hipotético.
- **Teste real #2** (após o fix): PR descartável, criado a partir de uma branch que continha o workflow já corrigido, mirando `dev` — a label `dev` foi aplicada sozinha com sucesso. PR e branch de teste deletados em seguida.
- `close`/`reopen` de um PR **não** dispara `types: [opened]` (só `reopened`) — comportamento correto do filtro, mas signfica que reabrir um PR não reaplica a label sozinho; descoberto ao tentar usar isso como atalho de teste e ver que a action simplesmente não rodava (confirmando que o filtro está fazendo exatamente o que deveria).
