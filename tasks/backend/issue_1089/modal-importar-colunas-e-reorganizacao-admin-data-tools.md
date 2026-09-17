# Modal de Importar Colunas com labels em branco + erros silenciosos no upload

**Repositório**: `backend`. **Issue**: [basedosdados/backend#1089](https://github.com/basedosdados/backend/issues/1089). **PR**: [basedosdados/backend#1090](https://github.com/basedosdados/backend/pull/1090). **Data**: 2026-09-17.

## O pedido original

O botão "Importar Colunas" na página de admin de uma `Table` abre uma modal (`dadosModal`) que, para alguns usuários, carregava com os nomes dos campos (labels) em branco. O ponto de partida indicado foi `backend/templates/admin/change_form.html`.

## Diagnóstico

| Sintoma | Causa raiz |
|---|---|
| Labels em branco pra alguns usuários | `</div>` sobrando no bloco do `dadosModal` em `change_form.html`, fechando uma `<div>` da página base do admin/Jazzmin (o bloco vive dentro de `{% block content %}{{ block.super }}...`) e quebrando a árvore DOM do resto da página. Navegadores recuperam de HTML malformado de formas diferentes — explica o problema depender do usuário/navegador |
| `SyntaxError: JSON.parse: unexpected character...` no navegador ao importar colunas | `upload_columns` não tinha tratamento de exceção — qualquer erro (`KeyError`, `Table.DoesNotExist`, planilha malformada) virava uma página HTML de traceback (500), que o `response.json()` do front não conseguia interpretar |
| Colunas de uma tabela sumindo depois de uma importação que falhou | `selected_table.columns.all().delete()` rodava **antes** de processar a planilha, sem transação — uma falha no meio do processamento deixava a tabela sem nenhuma coluna, sem recriar nada |
| Coluna salva com nome inválido pro BigQuery sem nenhum aviso | Nenhuma validação de identificador — um espaço, ponto a mais ou caractere inválido em `name`/`directory_column` era salvo silenciosamente e só quebraria bem mais tarde, na sincronização com o BigQuery |

**Ambiente de teste**: como não há suíte de testes automatizados rodável neste ambiente (sem infra de banco/Django test configurada), a validação foi feita subindo o `docker compose` local de verdade (banco real, servidor Django, mesma planilha real do usuário) e testando via `curl`/`manage.py shell`. Detalhe de ambiente: o Docker Desktop não tinha integração WSL habilitada para esta distro — contornado chamando `docker.exe` (binário Windows) diretamente do WSL, que enxerga o mesmo engine e resolve paths `/mnt/d/...` sem problema.

**Erro real usado como caso de teste**: a planilha de arquitetura tinha, em duas linhas referentes a `br_bd_diretorios_mundo.pais`, o `directory_column` com o nome da coluna duplicado — `br_bd_diretorios_mundo.pais.sigla_pais_iso3:sigla_pais_iso3` em vez de `br_bd_diretorios_mundo.pais:sigla_pais_iso3` (o padrão correto, `dataset.tabela:coluna`, usado nas outras linhas da mesma planilha).

## O que foi corrigido

### Modal (`change_form.html` / `ferramentas.js` / `main.css`)

- `</div>` sobrando removido; `class` duplicado no botão "Processar" removido.
- Botão de fechar ganhou `id` único (`dadosCloseButton`) — antes usava `document.querySelector('.close-button')`, que dependia da ordem no DOM (a mesma classe existe nos outros dois modais da página).
- `alert()` trocado por uma área de resultado inline (`#dadosResultado`), no mesmo estilo visual já usado pelo "Checar Metadados" (badge + nome da coluna em `<code>` + detalhe).
- Modal alargada (`modal-content--wide`) e `overflow-wrap: break-word` adicionado nas classes de mensagem — uma mensagem de erro longa e sem espaços estourava a largura fixa da modal.

### `upload_columns` — erro estruturado e proteção de dados

- Respostas sempre em JSON (nunca mais deixa vazar traceback HTML).
- Leitura da planilha e reconstrução das colunas dentro de `transaction.atomic()` — falha no meio não perde mais nada.
- Validação de identificador BigQuery (`_validar_identificador_bq`, regex `^[A-Za-z_][A-Za-z0-9_]*$`) aplicada ao nome da coluna e a cada pedaço do `directory_column`.
- Erro estruturado: `{"status": "erro", "erro": "<mensagem>", "coluna": "<nome>"}`, permitindo a UI destacar qual coluna da planilha causou o problema.

### Migração + reorganização

`upload_columns` (função solta em `api/v1/views.py`, rota `/upload_columns/`) virou `UploadColumnsView` (classe, rota `/admin-tools/upload-columns/`), junto das outras ferramentas da página de admin de `Table` (`CheckMetadadosView`, `SyncUpdateLatestView`). De quebra, `admin_data_tools/views.py` (690 linhas, 3 assuntos sem relação direta) foi dividido:

| Arquivo | Linhas | Conteúdo |
|---|---|---|
| `flow_monitoring.py` | 299 | `SyncDeploymentsView`, `FlowFailedWebhookView` e helpers de monitoramento/auto-disable de flow no Prefect |
| `bigquery_sync.py` | 190 | `CheckMetadadosView`, `SyncUpdateLatestView` e helpers de comparação/sincronização de `Table` contra o BigQuery |
| `column_import.py` | 217 | `UploadColumnsView`, `ColumnImportError` e helpers de leitura/validação da planilha de arquitetura |

Mesmo padrão já usado por `_prefect3_client.py` nesse app (arquivos irmãos por domínio, não um pacote `views/`).

## Testes realizados

| Cenário | Resultado |
|---|---|
| Nome de coluna com espaço (`'country id'`) | Bloqueado antes de gravar |
| `directory_column` sem `:` / com segmento extra (o erro real) / tabela de diretório inexistente / tipo BigQuery desconhecido | Bloqueados, mensagem aponta o campo exato |
| Importação válida (regressão) | Coluna criada e associada corretamente |
| Qualquer erro acima | Colunas existentes preservadas (transação) |
| Rota antiga `/upload_columns/` | 404 (removida) |
| `CheckMetadadosView` / `SyncUpdateLatestView` (só movidas de arquivo) | Testadas de ponta a ponta contra o BigQuery real após a divisão — sem regressão |
| `SyncDeploymentsView` / `FlowFailedWebhookView` (só movidas) | Guard de bearer token (401 sem token) confirmado; não re-executadas com token real por escreverem em deployments reais do Prefect, fora do escopo |
| Labels da modal | Confirmado visualmente no navegador |

## Pendência à parte (não resolvida aqui)

A view `upload_columns`/`UploadColumnsView` ainda tem um detalhe de leitura da URL da planilha (`read_architecture_table`/`_read_architecture_table`) que devolve um erro técnico cru (`list index out of range`) se a URL não tiver o formato esperado do Google Sheets — não chega a quebrar nada (JSON válido, sem perda de dados), mas a mensagem não é tão clara quanto as outras. Não mexido por não ter sido pedido; mapeado aqui caso valha a pena revisitar.
