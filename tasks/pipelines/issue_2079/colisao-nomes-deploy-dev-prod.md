# Deploy de PR (label `deploy-flow`) roubava deployments de produção pro pool dev

**Repositório**: `pipelines`. **Issue**: [basedosdados/pipelines#2079](https://github.com/basedosdados/pipelines/issues/2079) (órfãos: [#2080](https://github.com/basedosdados/pipelines/issues/2080)). **PR**: [basedosdados/pipelines#2081](https://github.com/basedosdados/pipelines/pull/2081). **Data**: 2026-09-17.

## Como foi descoberto

Investigando testes na `SyncDeploymentsView` (backend, issue #1088/#1092 — ver [`tasks/backend/`](../../backend/)), o usuário notou que o work pool de produção (`basedosdados`) estava com só ~22 deployments — deveria ter centenas.

## Causa raiz

`.github/scripts/deploy_flows.py`, função `deploy_flow()`, registrava o deployment sempre com `name=flow_name` — **idêntico** entre:

- o caminho de prod: `deploy_flows.py --pool basedosdados --branch main --all`;
- o caminho de PR/dev: `deploy_flows.py --pool basedosdados-dev --branch <branch>`, disparado pela label `deploy-flow` (workflow `cd-prefect3-staging.yaml`).

O Prefect identifica um deployment pela combinação `<flow>/<name>`, **não** pelo work pool — `work_pool_name` é só um campo mutável do mesmo registro. Como os dois caminhos usavam o mesmo `name`, todo PR com a label `deploy-flow` que tocasse um flow já deployado em produção não criava um deployment novo em dev: **atualizava o mesmo registro de produção**, movendo-o pro pool `basedosdados-dev` e zerando o schedule (o caminho dev sempre passa `schedules=None`).

### Evidência concreta, coletada via API real do Prefect3 (leitura)

| Verificação | Resultado |
|---|---|
| Contagem `basedosdados` (prod) | 22 |
| Contagem `basedosdados-dev` | 326 |
| `au_abs_population_flow` (tem `deploy_schedules` no código-fonte) | Só existia no pool dev, sem schedule, ausente de prod |

## Mitigação imediata

Antes de qualquer correção de código, pra estancar o sangramento:

```
gh api -X DELETE repos/basedosdados/pipelines/issues/<n>/labels/deploy-flow
```

Aplicado nos 10 PRs abertos que tinham a label: #2074, #2073, #2056, #2012, #2006, #1995, #1939, #1906, #1872, #1632. Isso impede que um novo commit (`synchronize`) nesses PRs dispare de novo o deploy pro pool dev com o script ainda não corrigido.

## Recuperação de produção

1. **Deploy completo de restauração**: `deploy_flows.py --pool basedosdados --branch main --all` — reconstrói cada deployment de prod do zero, com o schedule correto, na pool certa (não importa o que aconteceu com o registro em dev, já que o `.deploy()` faz upsert por `<flow>/<name>`, recriando/atualizando o registro de prod independentemente de onde ele esteja hoje).

   Rodado com a versão **já paralelizada** do script (ver abaixo): 563 arquivos escaneados, **276 registrados, 438 pulados, 0 erros**, em **~10min33s** — contra **~36min** da mesma operação em série (comparação direta com a run [35147233729](https://github.com/basedosdados/pipelines/actions/runs/35147233729), que tem exatamente os mesmos números de registrados/pulados/erros). Pool prod: 22 → 298.

2. **`sync-deployments`** (backend, `POST /admin-tools/sync-deployments/`, chamado via `kubectl exec` no pod real de produção usando a env var já configurada, nunca expondo o token) — reconcilia o estado ativo/pausado de cada deployment recém-restaurado contra o `DisabledFlowSchedule` do backend: 208 ativados, 140 pausados, 0 erros.

3. Verificação pontual: `au_abs_population_flow` de volta em `basedosdados`, com `schedules` restaurado (`cron: "48 16 14,17,20,23,26 * *"`) e `pull_steps` apontando pra `branch: "main"`.

## Correção definitiva

Nome do deployment passa a ser diferente por ambiente:

- prod: `name=flow_name` (sem mudar — `sync-deployments`/`set_deployment_schedule_active` do backend dependem desse nome exato).
- dev: `name=f"dev-{flow_name}"` (prefixo, não sufixo — feedback direto do usuário depois de uma primeira versão com sufixo `-dev`).

Cada ambiente passa a ter seu próprio registro, nunca competindo pelo mesmo pool.

**Sobre a preocupação de "poluir" o pool dev com mais registros**: não procede. O Prefect faz upsert pelo nome — rodar a mesma PR de novo só atualiza o `dev-<flow_name>` que já existe, não cria um terceiro. O modelo com nome separado tem exatamente a mesma quantidade de registros por ambiente que o modelo anterior *deveria* ter (um por ambiente) — só que de verdade, em vez de compartilhado por acidente.

Também atualizado `.claude/skills/pipeline-doctor/references/fix-and-ship.md`, que documentava disparar um dev run com o nome puro do flow — esse fluxo só funcionava *por causa* do bug (nome duplicado = registro único, sempre encontrável por nome sozinho). Corrigir a colisão sem atualizar essa doc teria criado uma janela real de erro: alguém tentando `<flow_name>/<flow_name>` esperando o dev poderia acertar sem querer o deployment de **produção**.

### Teste da correção

| Verificação | Resultado |
|---|---|
| Deploy de `au_abs_population_flow` no pool dev, após a correção | Cria `dev-au_abs_population_flow` (ID novo, separado) |
| Deployment de prod (`au_abs_population_flow`) | Intacto — mesmo ID, `paused: False`, schedule preservado |

## Melhoria aproveitada: deploy paralelizado

Como a recuperação exigia rodar `--all` em produção (potencialmente ~36min em série), aproveitado pra paralelizar `deploy_flow()` via `ThreadPoolExecutor` (novo `--workers`, default 8):

- `deploy_flow()` deixou de imprimir diretamente e passou a devolver `(sucesso, mensagem)`, evitando que a saída de flows diferentes se misturasse ao rodar em várias threads.
- `main()` primeiro carrega todos os flows dos arquivos (rápido, sem rede), só depois registra em paralelo.

Resultado real medido: **~3.4x mais rápido** (36min → ~10min33s) na mesma operação de restauração completa.

## Pendência aberta

[Issue #2080](https://github.com/basedosdados/pipelines/issues/2080): ~50 deployments no pool dev ainda com o nome antigo (sem prefixo `dev-`), sobra de antes da correção. Sem risco de colisão (produção já foi restaurada), só ruído fora do padrão novo — decisão de limpeza adiada, não urgente.
