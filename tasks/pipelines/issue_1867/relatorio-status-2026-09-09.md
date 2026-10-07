# Pipeline orientado a eventos — status

**Data:** 09 de setembro de 2026
**Issue:** [basedosdados/pipelines#1867](https://github.com/basedosdados/pipelines/issues/1867)
**Branch:** `feat/event-pipeline-automations-poc`

## Resumo

Estamos substituindo os flows monolíticos de coleta de dados — que hoje sobem um pod grande pra fazer tudo (checar, baixar, materializar) mesmo quando não há nada novo — por uma cadeia de três etapas independentes, cada uma no seu próprio pod, com o tamanho de recurso certo pra cada trabalho:

```
[check_update] --dispara-->  [download]  --dispara-->  [mat_test]
 pod leve, checa a fonte     baixa e envia o dado       compartilhado, roda dbt + testes
```

A última etapa (`mat_test`) é um único deployment genérico, compartilhado por todos os datasets — reduz o número de peças que precisamos manter conforme mais datasets migram. Esse mecanismo de disparo já está validado de ponta a ponta em infraestrutura real desde o piloto inicial.

## Números

| | |
|---|---|
| Datasets reais migrados | **10** |
| Tabelas cobertas | **42** |
| Flows novos registrados no código | **68** |
| Datasets bloqueados (pendência documentada) | **1** |

## Linha do tempo

- **28 ago** — Proposta e levantamento: desenho original da arquitetura; mapeados ~82 datasets reais e como cada um checa por dados novos.
- **01–03 set** — Prova de conceito validada: mecanismo de disparo testado de ponta a ponta com dado real em dev e produção, incluindo dados particionados. Boilerplate encapsulado numa classe reutilizável.
- **04–05 set** — Primeiro dataset real: `br_ibge_ipca` migrado (4 tabelas) — expôs e resolveu um risco real de deploy sobrescrever produção.
- **09 set** — Migração em lote: mais 10 datasets reais migrados de uma vez, cobrindo 42 tabelas. Um ficou de fora, bloqueado por uma limitação real da abstração.

## Datasets migrados

| Dataset | Tabelas | Situação |
|---|---|---|
| `br_ibge_ipca` | 4 | ✅ Migrado |
| `br_ans_beneficiario` | 1 | ✅ Migrado |
| `br_denatran_frota` | 2 | ✅ Migrado |
| `br_inmet_bdmep` | 1 | ✅ Migrado |
| `br_me_caged` | 3 | ✅ Migrado |
| `br_me_cnpj` | 4 | ✅ Migrado |
| `br_me_comex_stat` | 4 | ✅ Migrado |
| `br_ms_cnes` | 13 | ✅ Migrado |
| `br_sfb_sicar` | 9 | ⚠️ Migrado, precisa revisão |
| `us_cfpb_hmda` | 1 | ✅ Migrado |
| `br_bcb_sicor` | 11 | 🚫 Bloqueado |

"Migrado" = implementado e verificado estaticamente (sem erros de sintaxe/tipo, sem conflito de nomes). Nenhum dos 10 foi testado ainda contra a fonte de dados real nem colocado em produção — esse é o próximo passo, um de cada vez.

## Decisões e riscos administrados

**Risco de deploy real, mitigado antes de acontecer.** O processo de deploy publica qualquer flow que encontrar no arquivo de código. Se o flow antigo de um dataset (já em produção, agendado) continuasse no mesmo arquivo, um deploy de teste poderia sobrescrever silenciosamente a fonte de onde o Prefect busca esse código. Identificamos isso antes de rodar qualquer deploy e removemos o código antigo por completo antes de migrar.

**`br_sfb_sicar` foge do padrão de propósito.** Esse dataset materializa e testa 9 tabelas juntas, com testes cruzados entre elas — o mecanismo genérico de teste assume uma tabela por vez. A migração manteve essa etapa como estava, sem tentar forçar o encaixe. Precisa de revisão de alguém do time antes de qualquer deploy real.

**`br_bcb_sicor` ficou de fora.** Esse dataset decide se há dado novo comparando o tamanho do arquivo na fonte, não uma data — algo que a abstração atual não sabe fazer. Nada foi alterado nesse dataset; a lacuna está documentada para decidirmos se vale estender o mecanismo comum.

**Erro recorrente identificado.** Em 3 dos datasets migrados, a contagem de tabelas do levantamento inicial estava errada (uma tabela, quando na real eram 2, 4 ou até 9). Motivo: o levantamento olhou só metadados, não o código real. Toda migração agora inspeciona o código-fonte antes de começar.

## Próximos passos

- Validar cada dataset migrado contra a fonte de dados real, um de cada vez, antes de qualquer schedule automático entrar em produção.
- Decidir, com o time, se vale estender o mecanismo comum para cobrir os 3 casos que ficaram de fora (comparação por tamanho, ajustes manuais de execução, testes cruzados entre tabelas).
- Revisar `br_sfb_sicar` com alguém familiarizado com o dataset antes de migrá-lo de vez.
- Seguir migrando o restante dos ~70 datasets ainda no formato antigo.

## Documentação de referência

- [Issue #1867](https://github.com/basedosdados/pipelines/issues/1867) — proposta original, decisões de arquitetura e status atualizado
- [PR #1932](https://github.com/basedosdados/pipelines/pull/1932) — implementação do mecanismo, prova de conceito e detalhes técnicos
- [Registro cronológico completo](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tasks/pipelines/issue_1867/issue-1867-pipeline-eventos.md) — histórico detalhado de toda a implementação
- [Desenho do mecanismo reutilizável](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tasks/pipelines/issue_1867/encapsular-check-update-e-download.md) — como o encadeamento entre etapas foi generalizado
- [Referência de nomenclatura](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tutoriais/pipelines/pipeline-orientado-a-eventos-fluxo-e-nomenclatura.md) — guia rápido de como flows e deployments são nomeados
- [Levantamento dos ~82 datasets](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tasks/pipelines/issue_1867/levantamento-datasets-por-categoria-de-check.md) — classificação de cada dataset e critério de priorização
- [Migração do br_ibge_ipca](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tasks/pipelines/issue_1867/migracao-br-ibge-ipca.md) — primeiro dataset real, achados e o risco de deploy identificado
- [Migração em lote — 10 datasets](https://github.com/Winzen/ftwca/blob/docs/issue-1867-encapsulate-check-download/tasks/pipelines/issue_1867/migracao-lote-10-datasets.md) — detalhe dataset a dataset da leva mais recente

---
Issues relacionadas abertas a partir deste trabalho: [#1940](https://github.com/basedosdados/pipelines/issues/1940) (bug pré-existente, repositório inteiro), [#1943](https://github.com/basedosdados/pipelines/issues/1943) (tempo de deploy em produção), [#1967](https://github.com/basedosdados/pipelines/issues/1967) (lacuna de configuração do BigQuery).
