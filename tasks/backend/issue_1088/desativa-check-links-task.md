# Desativa check_links_task (ruído no Discord, falsos positivos de anti-bot)

**Repositório**: `backend`. **Issue**: [basedosdados/backend#1088](https://github.com/basedosdados/backend/issues/1088). **PR**: [basedosdados/backend#1098](https://github.com/basedosdados/backend/pull/1098). **Data**: 2026-09-17.

## Contexto

`check_links_task` (`backend/apps/api/v1/tasks.py`) rodava toda segunda a sexta às 8h, fazendo um `GET` simples (`verify=False`, sem nenhum header) em toda `RawDataSource.url` cadastrada, reportando qualquer falha no Discord via `Messenger`. Exemplo real de um disparo:

```
Revise os seguintes links:
Portal da Transparência — Download de Dados (CGU Sanções) (Cadastros de Sanções da CGU): 405 Client Error: Not Allowed for url: https://portaldatransparencia.gov.br/download-de-dados
CEPR Uniform Data Extracts — programas de harmonização (Current Population Survey (CPS)): 406 Client Error: Not Acceptable for url: https://ceprdata.org/cps-uniform-data-extracts/
Dados (American National Election Studies (ANES)): 403 Client Error: Forbidden for url: https://electionstudies.org/data-center/
...
```

## Diagnóstico

Os erros (403 Forbidden, 405 Not Allowed, 406 Not Acceptable, conexão resetada) são sintomas clássicos de proteção anti-bot/anti-scraping do lado da fonte, não links quebrados de verdade. A task não manda nenhum header (`User-Agent`, `Accept`) — exatamente o tipo de requisição que esses mecanismos costumam rejeitar, mesmo quando o link funciona normalmente num navegador. Como isso não muda de um dia pro outro, a mesma lista de links repetia todo dia útil, virando ruído recorrente sem informação nova/acionável.

## Decisão e solução

Desativar por enquanto — sem reescrever a estratégia de request (headers/retry), que ficaria pra uma revisão futura se fizer sentido retomar. Removido só o `@db_periodic_task(crontab(...))` da função; o corpo continua no código, com `@production_task` e um comentário explicando o motivo + referência à issue, facilitando reativar depois.

## Teste

Reiniciei o ambiente local (`docker compose`) com a mudança e conferi o log de startup do huey (`> Running Huey` / "The following commands are available"): `check_links_task` não aparece mais na lista — antes aparecia junto das outras tasks periódicas (`update_search_index_task`, `update_table_metadata_task` etc.). Confirma que a desativação funciona em runtime, não só no código-fonte. `manage.py check` e `ruff` sem problemas.
