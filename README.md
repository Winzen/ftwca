# ftwca
Documentação técnica pra quem vier depois — notas, runbooks e guias que uso no dia a dia

## Sobre o nome

`ftwca` = **"For those who come after."** — em português, "por aqueles que virão depois" ou "para os que vierem depois".

É o lema recorrente de *Clair Obscur: Expedition 33*: cada expedição deixa algo pra ajudar as próximas, mesmo sabendo que provavelmente não vai sobreviver pra ver o resultado. Documentação técnica é isso — você escreve pra quem (às vezes você mesmo, no futuro) vai precisar entender ou continuar o que ficou pra trás.

## Estrutura

Os documentos ficam organizados em três categorias no topo, e dentro de cada uma por repositório/projeto (`backend`, `pipelines`, `iac`, `mcp`, ...):

```
tutoriais/<repo>/            guias e referência de "como fazer X" — não estão presos a uma tarefa específica
tasks/<repo>/issue_N/        documentação de um trabalho ligado a uma issue específica do GitHub
tasks/<repo>/pr_N/           mesma ideia, mas quando só existe uma PR e não há issue associada
investigacoes/<repo>/        auditorias, investigações e relatórios pontuais — não são passo-a-passo nem estão amarrados a uma issue/PR
```

Regra de nomenclatura em `tasks/`: prefira `issue_N` sempre que o trabalho tiver uma issue por trás (mesmo que tenha várias PRs); use `pr_N` só quando não existir issue formal.

Alguns documentos ainda estão pendentes de organização (revisão de conteúdo sensível antes de publicar, ou categoria ainda não decidida) — o repositório vai crescendo aos poucos.
