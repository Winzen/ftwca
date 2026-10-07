# Incidente: scheduler do prefect-server parado (SQLAlchemy 2.1 + OOMKill crônico)

Referências: `basedosdados/iac` issue [#157](https://github.com/basedosdados/iac/issues/157), PR [#158](https://github.com/basedosdados/iac/pull/158).

## Resumo

O scheduler do Prefect 3 (`prefect3.basedosdados.org`) parou de gerar novos flow runs `Scheduled` a partir de 2026-10-02. O sintoma só ficou visível vários dias depois, quando o estoque de runs já agendadas antes do bug começar se esgotou. Causa raiz dupla: um bug upstream do Prefect com SQLAlchemy 2.1 (puxado por uma tag Docker flutuante), e um OOMKill crônico do pod que já vinha acontecendo havia pelo menos um mês, de forma independente, por memória insuficiente pro volume atual de deployments.

## Fluxograma da solução

```text
Scheduler parado (zero runs "Scheduled" no sistema inteiro)
            │
            ▼
Buffer de runs já geradas se esgota → sintoma só fica visível dias depois
            │
            ▼
kubectl: pod prefect-server com 3 restarts (OOMKilled), criado em 2026-10-02
            │
            ▼
Logs do pod: InvalidRequestError ("Can't evaluate bulk DML statement")
            │
            ├──────────────────────────────┬──────────────────────────────┐
            ▼                              ▼
  Causa 1 — SQLAlchemy 2.1         Causa 2 — memória insuficiente
  tag "3-latest" puxou                limite 256Mi, ~775 deployments
  SQLAlchemy 2.1.1 numa repull                 │
            │                                  ▼
            ▼                       Cloud Logging: histórico de restarts
  Bug confirmado                    do container reconstruído — já
  (PrefectHQ/prefect#23199,         reiniciava com cadência regular
   sem fix em release estável)      desde 2026-09-08, um mês antes do
            │                       bug do SQLAlchemy — crônico, não é
    ┌───────┴────────┐              sintoma do bug (causa independente)
    ▼                ▼                          │
Downgrade pra    EXTRA_PIP_PACKAGES              ▼
3.8.6-python3.12  em runtime             Aumentar memória do pod
    │                │                   256Mi → request 512Mi / limit 1Gi
    ▼                ▼                             │
CrashLoopBackOff  Read-only filesystem              │
(Alembic: banco    (os error 30 ao                  │
já migrado pro     tentar remover o                 │
schema da 3.8.7)   pacote antigo)                   │
    │                │                              │
    └────────┬───────┘                              │
             ▼                                      │
   Imagem Docker customizada (build-time):          │
   FROM prefecthq/prefect:3.8.7-python3.12          │
   RUN pip install --no-cache-dir "sqlalchemy<2.1"  │
             │                                      │
             ▼                                      │
   Tag fixa no values.yaml                          │
   (prefectTag deixa de ser "3-latest")              │
             │                                      │
             └──────────────────┬───────────────────┘
                                 ▼
                    helm upgrade --install (produção)
                                 │
                                 ▼
            Verificação: pod com SQLAlchemy 2.0.54, sem erro de
            bulk DML nos logs, runs "Scheduled" voltando a aparecer
            no sistema inteiro (inclusive os 6 datasets do pipelines#1932)
```

## Como o sintoma foi percebido

Pós-merge do `pipelines#1932` (migração de 6 datasets pro pipeline orientado a eventos), os 22 novos deployments `check_update` foram reativados (`paused=False`), mas nenhum gerava runs futuras. Olhando o sistema como um todo — não só os deployments novos — nenhum tinha runs `Scheduled`: sintoma generalizado, não ligado à migração.

## Diagnóstico

1. Comparação `created` vs `expected_start_time` das runs mais recentes confirmou que o scheduler tinha parado de rodar há dias — em operação normal, o Prefect mantém um buffer de vários DIAS de runs futuras já geradas, o que explica por que o problema ficou invisível por tanto tempo (o sistema "parecia" funcionar enquanto consumia esse estoque).
2. `kubectl` no cluster: pod do `prefect-server` com 3 restarts por `OOMKilled`, criado em 2026-10-02.
3. Logs do pod revelaram o erro real por trás de cada crash do scheduler:
   ```
   sqlalchemy.exc.InvalidRequestError: Can't evaluate bulk DML statement;
   please supply a bulk_dml decorated function
   ```

## Causa raiz 1: bug upstream do Prefect com SQLAlchemy 2.1

O pod nasceu com a tag Docker flutuante `prefectTag: "3-latest"`, que puxou SQLAlchemy 2.1.1 (lançado 2026-09-24) numa repull. O Prefect declara a dependência como `sqlalchemy[asyncio]>=2.0,<3.0.0` — uma faixa aberta que não protege contra 2.1.

O bulk insert de flow runs (`prefect/server/models/deployments.py:_insert_scheduled_flow_runs`) quebra com SQLAlchemy >= 2.1 porque um dict de insert com a chave `state` é resolvido pra hybrid property `FlowRun.state` em vez de ser tratado como coluna bruta. Issue upstream: [PrefectHQ/prefect#23199](https://github.com/PrefectHQ/prefect/issues/23199). Nenhuma release estável tem o fix ainda — [PrefectHQ/prefect#23200](https://github.com/PrefectHQ/prefect/pull/23200) (capar SQLAlchemy abaixo de 2.1) e [#23201](https://github.com/PrefectHQ/prefect/pull/23201) (compatibilizar o server com 2.1) seguem abertos.

### Por que não um downgrade simples

Tentativa de rodar `prefecthq/prefect:3.8.6-python3.12` (versão anterior) resultou em `CrashLoopBackOff`:
```
alembic.util.exc.CommandError: Can't locate revision identified by '<hash>'
```
O banco já tinha sido migrado pelo Alembic pro schema da 3.8.7 antes da tentativa — downgrade de versão do Prefect não é seguro depois que uma versão mais nova já rodou contra o banco.

Também não dá pra usar `EXTRA_PIP_PACKAGES=sqlalchemy<2.1` em runtime (mecanismo oficialmente suportado pela imagem do Prefect pra instalar pacotes extras na inicialização): o container roda com filesystem raiz somente leitura, e o `pip`/`uv pip install` falha tentando remover o pacote antigo (`Read-only file system (os error 30)`).

### Fix aplicado

Imagem Docker customizada (`k8s/prefect3/image/Dockerfile` no `iac`), baseada na versão já migrada no banco (`3.8.7-python3.12`), com SQLAlchemy fixado abaixo de 2.1 em **build-time** — não runtime:

```dockerfile
FROM prefecthq/prefect:3.8.7-python3.12
RUN pip install --no-cache-dir "sqlalchemy<2.1"
```

Publicada em `gcr.io/basedosdados-dev/prefect-server-sqlalchemy-fix:3.8.7-sqlalchemy2.0`. A tag Docker também deixou de ser flutuante (`values.yaml`: `prefectTag` fixo na versão exata) — a tag `"3-latest"` repullando numa versão incompatível foi o que originou o incidente.

**Reavaliar quando o fix for mergeado upstream**: assim que uma versão estável do Prefect >= 3.8.8 sair com [#23200](https://github.com/PrefectHQ/prefect/pull/23200)/[#23201](https://github.com/PrefectHQ/prefect/pull/23201) mergeados, voltar pra `prefecthq/prefect` oficial e remover esta imagem customizada.

## Causa raiz 2: OOMKill crônico, anterior e independente do bug acima

Depois do fix do SQLAlchemy aplicado, surgiu a dúvida: o OOMKill observado era um sintoma causado pelo próprio bug (ex.: algum loop de retry consumindo memória a cada falha de bulk insert), ou um problema de memória genuinamente insuficiente, só coincidindo em timing?

Reconstruindo o histórico de restarts do container via Cloud Logging (cada restart reimprime o banner de inicialização do Prefect, já que os eventos nativos do Kubernetes não ficam retidos por muito tempo):

| Pod | Reinícios observados |
|---|---|
| `...8mq8q` | 09-10 → 09-14 (4d) → 09-17 (3d) |
| `...shzch` | 09-18 → 09-22 (4d) → 09-23 (1d) → 09-28 (5d) → 09-30 (2d) |
| `...6cckh` (o que pegou o SQLAlchemy 2.1.1 em 10-02) | 10-02 → 10-03 (17h) → 10-04 (22h) → 10-04 (22h) |

É o mesmo pod reiniciando sozinho (nome do pod inalterado, `restartCount` incrementando — não é um rollout, que criaria um pod novo), com cadência regular desde pelo menos 2026-09-08, **um mês antes** do bug do SQLAlchemy entrar em cena. Não dá pra confirmar 100% que a causa exata desses reinícios de setembro era `OOMKilled` (os eventos do Kubernetes daquele período não ficaram retidos), mas a cadência é muito parecida com a do incidente confirmado em outubro, e bate com o uso de memória observado logo após o fix: **~199Mi em repouso**, contra um limite antigo de **256Mi** — margem de ~22% pro volume atual (~775 deployments), claramente apertada demais.

**Conclusão**: o aumento de memória (`256Mi` → request `512Mi` / limit `1Gi`) era necessário de forma independente do bug do SQLAlchemy — não foi um sintoma criado por ele, era um problema crônico que só ficou mais frequente (reiniciando quase todo dia, em vez de a cada poucos dias) depois que o bug entrou.

## Verificação em produção

- Pod rodando SQLAlchemy `2.0.54` (confirmado via exec direto no pod).
- Ausência do erro de bulk DML nos logs após o redeploy.
- Novas flow runs `Scheduled` voltaram a aparecer no sistema inteiro, incluindo os deployments `check_update` dos 6 datasets migrados no `pipelines#1932`.

O fix foi aplicado ao vivo via `helm upgrade` antes de ser commitado no `iac` — o trabalho de branch/commit/PR serviu só pra versionar o que já estava rodando em produção.

## Pendências

**Reavaliar e remover a imagem customizada quando o Prefect oficial corrigir o bug.** `prefect-server-sqlalchemy-fix` é um workaround, não a solução definitiva — o ideal é voltar pra `prefecthq/prefect` assim que uma versão estável >= 3.8.8 sair com [#23200](https://github.com/PrefectHQ/prefect/pull/23200)/[#23201](https://github.com/PrefectHQ/prefect/pull/23201) mergeados. Enquanto isso não acontece, qualquer rebuild da imagem customizada deve manter o pin do SQLAlchemy.

**Reconfirmar o estado final dos 22 deployments `check_update` reativados no `pipelines#1932`.** Durante o incidente, esses deployments foram reativados manualmente, depois um `sync-deployments` os reverteu pra `paused=True` (sobrescrevendo com registros desatualizados do banco), e foram reativados de novo. Essa sequência de idas e vindas nunca teve uma verificação final pós-fix — é preciso confirmar se os 22 realmente seguem com schedule ativo agora, e não assumir que a última ação tomada durante o incidente ainda vale.

**Decidir o destino dos 10 deployments órfãos dos flows monolíticos pré-migração.** Esses são os deployments dos flows antigos dos 6 datasets que migraram pro pipeline orientado a eventos no `pipelines#1932` — ficaram órfãos porque a migração não remove automaticamente a versão antiga. Foram pausados manualmente (decisão tomada durante o incidente), mas não deletados — falta decidir se valem a pena manter como histórico/rollback de emergência ou se devem ser removidos de vez.

**A fragilidade do `sync-deployments` segue sem solução definitiva.** O mecanismo sobrescreve o estado `paused` ao vivo no Prefect com registros desatualizados do banco do backend — foi exatamente isso que reverteu os 22 deployments reativados durante este incidente (pendência acima). A correção própria (endpoint `set-schedule-active`, que evita essa sobrescrita cega) está em `basedosdados/backend#1060`, ainda aberto/não mergeado.
