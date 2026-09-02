## Por que isso é necessário

Feature branches às vezes são cortadas a partir de `main` em vez de `staging`/`dev`. Quando isso acontece, um PR dessa feature contra `staging` vem "poluído": o GitHub mostra, junto com o commit real da feature, todos os commits que já existem em `main` mas ainda não existem em `staging` — mesmo sem relação nenhuma com a feature.

Exemplo real (2026-08-13): o PR #1049 (`feat/email_end_trial`, backend) mostrava 3 commits — o commit real da feature (`feat: update email templates...`) mais dois commits (`fix: corrige update_table_metadata_task...` #1042 e `fix: remove Prefect 0...` #1043) que já estavam em `main` mas não em `staging`. Sincronizar `staging` com `main` (este processo) faz esses dois commits virarem ancestrais comuns — depois disso, o diff do #1049 limpa sozinho, sobrando só o commit da feature.

**Nota:** o repositório tem um `CLAUDE.md` (adicionado em 2026-08-12) documentando um fluxo de Git Flow diferente (cherry-pick por feature, nunca merge entre branches de ambiente). Por enquanto, **ignore essa política** — o repositório ainda não está pronto pra esse formato. Continue sincronizando via merge, como abaixo.

## Passo a passo

```
git checkout main
git pull
git checkout staging
git pull
```

Criar a branch de sincronização (nome padrão usado nos PRs anteriores: `chore/main_to_staging`):
```
git checkout -b chore/main_to_staging
```

Merge de `main`, priorizando o que já está em `staging` em caso de conflito real (`-X ours` só entra em ação se houver conflito de linha — se o merge for limpo, o resultado é idêntico a um merge normal):
```
git merge main -X ours
```

Push e abrir PR de `chore/main_to_staging` para `staging` (não fazer push direto em `staging`):
```
git push -u origin chore/main_to_staging
```

Título do PR, seguindo o padrão dos anteriores (#1027, #1041): `chore: sync staging and main - (PRs: #XXXX, #YYYY)`, listando os PRs/commits reais trazidos de `main` (não é preciso listar commits só de documentação se já convergem sem conflito).

Depois do merge do PR, deletar a branch de sincronização:
```
git branch -d chore/main_to_staging
git push origin --delete chore/main_to_staging
```

## Exemplo aplicado com sucesso

PR [basedosdados/backend#1055](https://github.com/basedosdados/backend/pull/1055) — sincronizou `staging` com `main` (3 commits: #1042, #1043, e o `CLAUDE.md` #1053), merge limpo sem conflitos, `-X ours` aplicado por padronização.

## Claude Ignore a baixo
djangorestframework = "^3.16.0"
djangorestframework-simplejwt = "^5.5.0"

### Usando JWT
```
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
}

SIMPLE_JWT = {
    "ALGORITHM": getenv("DJANGO_JWT_ALGORITHM"),
    "ACCESS_TOKEN_LIFETIME": timedelta(hours=1),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "USER_AUTHENTICATION_RULE": "backend.apps.chatbot.authentication.authentication_rule",
}
```
