Bom dia pessoal! 👋

Fizemos duas mudanças no cluster ontem para liberar recursos que estavam sendo reservados sem uso:

**1. Prefect 0 — scale down dos workers e agentes**
O Prefect 0 não tem mais schedules ativos desde a migração para o Prefect 3. Os pods continuavam rodando e reservando ~1.35 cores de CPU e ~1.7Gi de memória. Zeramos as réplicas dos agentes (4 namespaces) e reduzimos os requests dos componentes do servidor (hasura, graphql, apollo) para valores compatíveis com o uso real. Os deployments não foram deletados — podem ser reativados se necessário.
PR: https://github.com/basedosdados/iac/pull/139

**2. Loki — redução de cache e retenção de logs**
Os pods de cache do Loki (`chunks-cache` e `results-cache`) usavam os defaults do chart: 8192 MB e 1024 MB reservados. Medindo o uso real dos últimos 7 dias via Prometheus, o pico foi 387Mi e 2Mi respectivamente. Ajustamos para 512Mi e 64Mi. Também configuramos retenção de 6 meses — sem isso, o PVC de 100Gi encheria em ~10 meses.
PR: https://github.com/basedosdados/iac/pull/137

**Recursos liberados no total:**

| Recurso | Liberado |
|---|---|
| CPU (requests) | ~1.35 cores |
| Memória (requests scheduler) | ~1.7Gi |
| Memória (cache Loki) | ~8.5Gi |

Se observarem qualquer comportamento estranho no cluster — pods não subindo, métricas fora do normal, erros inesperados — me chamem aqui!
