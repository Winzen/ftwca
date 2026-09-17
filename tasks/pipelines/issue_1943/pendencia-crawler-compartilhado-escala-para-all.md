# Pendência — `pipelines/crawler/` sempre escala pra `--all`? Talvez precise ser mais fino

Registrado em 2026-09-16, a partir de uma dúvida real do usuário depois do primeiro disparo em produção do deploy seletivo (issue #1943, PR [basedosdados/pipelines#2044](https://github.com/basedosdados/pipelines/pull/2044)) que caiu no caminho de escalada. **Nada decidido — só o mapeamento do espaço de opções**, pra retomar quando fizer sentido.

## O caso real que motivou isso

Run [35147233729](https://github.com/basedosdados/pipelines/actions/runs/35147233729) — push em `main` alterando `pipelines/crawler/datasus/{constants,flows,tasks,utils}.py` + `pipelines/datasets/br_ms_sia/flows.py` + `pipelines/datasets/br_ms_sih/flows.py`.

```
Último deploy de prod bem-sucedido: 67bede06...
Flows alterados: pipelines/crawler/datasus/{constants,flows,tasks,utils}.py
                 pipelines/datasets/br_ms_sia/flows.py
                 pipelines/datasets/br_ms_sih/flows.py
Arquivos  : 563   ← escalou pro all_python_files()
Resultado: 276 registrados, 438 pulados, 0 com erro
```

`pipelines/crawler/` é um dos dois `SHARED_PREFIXES` em `deploy_flows.py` (o outro é `pipelines/utils/`) — qualquer mudança ali faz `expand_changed_files()` devolver `None`, escalando pro `--all` (~36min neste caso, 0 erros — funcionou certo, só pagou o custo cheio).

## A dúvida

`pipelines/utils/` é genuinamente repo-wide — `CheckThenDownloadPipeline`, `run_dbt`, `upload_to_gcs` etc. são usados por **todo** dataset migrado, não tem como restringir o escopo do impacto. Mas `pipelines/crawler/<nome>/` é diferente: cada subpasta tem um conjunto **conhecido e limitado** de datasets consumidores — `datasus` alimenta `br_ms_sia`/`br_ms_sih`/`br_ms_cnes`/`br_ms_sinan` (4), mas `me_cnpj` alimenta só `br_me_cnpj` (1). Tratar os dois casos com a mesma escalada pro repo inteiro é seguro, mas caro — e fica mais caro conforme mais flows forem migrados (issue #1867).

## Opções mapeadas

1. **Manter como está.** Simples, sempre seguro, mas paga ~36min inteiros mesmo quando só 1 dataset depende do crawler tocado.

2. **Mapa manual `crawler → datasets consumidores`**, expandindo pra esses datasets em vez de escalar pra tudo (mesmo princípio de `pipelines/datasets/<dataset>/` já expande hoje). Risco real: é exatamente o tipo de coisa que `expand_changed_files()` foi desenhado pra evitar — um mapa desatualizado (alguém passa a importar de um crawler sem atualizar o mapa) falha **silenciosamente**, sem erro, só um flow que fica desatualizado sem ninguém notar. Precisaria de alguma trava (ex. um teste/lint que confira o mapa contra os imports reais) pra não reintroduzir esse risco.

3. **Escaneamento de import em vez de mapa manual** — grep/AST simples em `pipelines/datasets/*/tasks.py`/`flows.py` procurando `from pipelines.crawler.<nome>` na hora do deploy, montando a lista de consumidores dinamicamente, sem mapa pra manter atualizado à mão. Mais robusto que a opção 2 (não fica velho sozinho), mas é mecanismo novo, nunca implementado nem testado — imports dinâmicos ou com alias passariam despercebidos.

4. **Não mexer na precisão, atacar o custo do `--all` em si** — `deploy_flow()` já é sequencial, um `.deploy()` por vez (~12-13s/flow, o número original que abriu a issue #1943). Paralelizar isso reduziria o tempo de qualquer `--all` (seletivo ou não), tornando a pergunta "quando escalar" bem menos importante — mesmo pagando o caminho seguro com frequência, o custo real cai. Não resolve a "imprecisão" do prefixo, mas reduz o motivo de se importar com ela.

## Status

Só mapeado. Nenhuma das opções decidida — inclusive é razoável não fazer nada agora e só reavaliar se o `--all` de fallback começar a acontecer com frequência incômoda na prática (esse foi o primeiro disparo real em produção). Se/quando retomar, a ordem sugerida seria: medir quantas vezes isso realmente dispara ao longo de algumas semanas, decidir se vale o esforço, e só então escolher entre as opções 2-4 acima.
