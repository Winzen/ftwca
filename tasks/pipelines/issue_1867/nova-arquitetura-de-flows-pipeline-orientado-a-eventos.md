# Nova arquitetura de flows — pipeline orientado a eventos

**Status:** Em planejamento
**Data:** 2026-08-28

---

## Contexto

Os flows atuais no Prefect 3 são monolíticos: um único flow faz check de atualização, download, upload para GCS, materialização em dev, testes, materialização em prod e atualização de metadados — tudo em sequência dentro do mesmo flow run.

Isso gera quatro problemas principais:
1. **Desperdício de recurso no check** — todo flow sobe um pod de 4 GB para verificar se o dado precisa ser atualizado. Na maioria das execuções a resposta é "não" — o pod sobe, faz uma chamada de API e desce, sem justificar o custo. Com dezenas de flows agendados, isso representa a maior parte do consumo de recursos do cluster.
2. **Recursos idênticos para cargas diferentes** — check (leve), download (médio) e materialização dbt (pesado) rodam no mesmo pod com a mesma alocação, mesmo tendo necessidades muito distintas.
3. **Cota compartilhada** — o gate de dev consome a cota de `basedosdados-dev` junto com o desenvolvimento manual (issue #1767)
4. **Reruns custosos** — falhar em prod obriga a rerrodar tudo desde o check, mesmo que o dado já esteja no storage

---

## Proposta

Quebrar o flow monolítico em **quatro flows independentes**, conectados por **automações do Prefect 3** disparadas por evento de conclusão bem-sucedida.

### Pipeline

```
[check_update]
      │
      │ sucesso → automação dispara
      ▼
[flow_download]
      │
      │ sucesso → automação dispara
      ▼
[mat_test]  ← dbt run+test em dev → dbt run+test em prod → atualiza metadados
```

Cada seta é uma **automação do Prefect 3** — não uma chamada direta entre flows. O flow downstream só roda se o upstream concluiu com sucesso.

---

## Identificação por tags

Cada flow recebe duas tags no deploy:
- **Etapa:** `check_update`, `flow_download`, `mat_test`
- **Dataset:** ex: `dataset:br_tse_filiacao_partidaria`

As automações filtram pelo par `(etapa_downstream, dataset)` para disparar o flow certo. Exemplo:

```
Evento: flow com tag [check_update, dataset:br_tse_filiacao_partidaria] concluiu com sucesso
Ação:   disparar deployment com tag [flow_download, dataset:br_tse_filiacao_partidaria]
```

---

## Responsabilidade de cada flow

### `check_update`
- Consulta o backend para ver se há dado novo na fonte
- Se sim: conclui com sucesso (dispara a automação)
- Se não: conclui com um estado que não dispara downstream (ex: `Cancelled` ou estado customizado)

### `flow_download`
- Baixa os dados da fonte original
- Faz upload para o GCS (`basedosdados-dev` staging)
- Não sabe nada de dbt

### `mat_test`
- Executa `dbt run` + `dbt test` em `basedosdados-dev` (gate)
- Executa `dbt run` + `dbt test` em `basedosdados` (prod)
- Atualiza metadados no backend (cobertura temporal, datas de atualização)
- Roda num único pod — dev e prod são sequenciais dentro do mesmo flow

**Decisão:** `dev_mat_test` e `prod_mat_test` foram fundidos num único `mat_test`. Manter dois flows separados permitiria rerrodar só prod se prod falhar, mas esse caso é raro na prática. O overhead de um startup de pod extra em todo pipeline com dado novo não se justifica. A isolação de cota BQ entre dev e prod é controlada pelo `execution_project` do dbt target — não depende de pods separados.

---

## Passagem de parâmetros entre flows

O `check_update` pode descobrir informações em runtime (URL do arquivo, data de referência) que o `flow_download` precisa. **Solução verificada no código Prefect 3:**

O `RunDeployment` server-side usa Jinja com contexto `event` — o evento disparador com seu `payload`. Qualquer valor no `payload` do evento customizado emitido pelo `check_update` pode ser interpolado nos `parameters` da ação da automação.

**Padrão:** o `check_update` chama `emit_event()` com `payload` arbitrário ao encontrar dado novo. A automação interpola `{{ event.payload.chave }}` nos parâmetros passados ao `flow_download`. Se não há dado novo, o evento não é emitido — a automação não dispara.

```python
from prefect.events import emit_event

emit_event(
    event="bd.pipeline.data-available",
    resource={"prefect.resource.id": f"pipeline/{dataset_id}/{table_id}"},
    payload={
        "dataset_id": dataset_id,
        "table_id": table_id,
        "source_url": url,
        "reference_date": reference_date,
        # chaves extras por dataset são ignoradas pelo downstream que não as usa
    },
)
```

GCS não é necessário para handoff de parâmetros — o evento carrega os dados junto com o sinal.

---

## Perfil de recursos por etapa

Com flows separados, cada etapa pode ter uma alocação de pod adequada à sua carga real:

| Flow | CPU / Memória | Justificativa |
|---|---|---|
| `check_update` | Mínimo (ex: 0.5 CPU / 512 MB) | Só faz chamadas de API — sem processamento de dados |
| `flow_download` | Médio (ex: 1 CPU / 2 GB) | Download e parsing de arquivos, pode ser pesado dependendo do dataset |
| `mat_test` | Alto (ex: 2 CPU / 4 GB) | dbt run+test em dev, depois dbt run+test em prod + chamadas ao backend |

Na arquitetura atual, **todos** sobem com 4 GB — incluindo os checks que apenas fazem uma chamada de API. Com dezenas de flows agendados rodando checks diariamente, isso representa o maior desperdício de recurso do cluster.

A nova arquitetura usa **3 flows** (não 4): fundir `dev_mat_test` e `prod_mat_test` num único `mat_test` economiza um startup de pod em cada pipeline completo, sem perder a capacidade de isolação de cota BQ (controlada pelo dbt target, não pelo pod).

---

## Vantagens em relação à arquitetura atual

| | Atual (monolítico) | Proposta (orientado a eventos) |
|---|---|---|
| Custo do check | Pod de 4 GB para uma chamada de API | Pod mínimo, só sobe se houver dado novo |
| Recursos por etapa | Alocação única para tudo | Dimensionado para a carga real de cada etapa |
| Falha em prod | Rerroda tudo desde o check | Rerroda só `mat_test` |
| Cota dev | Compartilhada com desenvolvimento | Isolada por flow independente |
| Observabilidade | 1 flow run com 20+ tasks | 3 flow runs curtos e focados |
| Reúso | Lógica de download acoplada ao dbt | Download reutilizável sem materialização |
| Rerrun manual de etapa | Não possível | Qualquer etapa pode ser rerrodada isolada |
| Código dos flows | Boilerplate repetido em todo flow | Cada flow faz só o que é seu — sem repetição |

---

## Preocupação: mais startups de pod deixam o pipeline mais lento?

**Contexto:** na nova arquitetura, um pipeline completo (quando há dado novo) faz 4 startups de pod em sequência em vez de 1. A dúvida é se isso torna o processo de atualização mais lento.

### Custo real de um startup de pod no cluster

Um startup de pod no Kubernetes tem dois componentes:

1. **Agendamento e criação** — < 5 segundos em condições normais
2. **Pull da imagem Docker** — é o custo real. Se a imagem não está em cache no nó: 1–3 minutos. Se está em cache (caso comum após o primeiro run): segundos.

Como todos os flows de pipelines usam a mesma imagem base, na prática a imagem já está em cache no cluster. O overhead real por startup tende a ser **10–30 segundos**, não minutos. Além disso, a automação do Prefect 3 entre flows adiciona alguma latência de propagação de evento — pequena, mas existe.

### Balanço do overhead

| Cenário | Impacto na nova arquitetura |
|---|---|
| Dataset atualiza (pipeline completo) | +3 startups extras × ~20s = ~1 min de overhead |
| Dataset não atualiza (majoritário nas execuções diárias) | **−3 startups** — só o check pequeno sobe |

Para a maioria dos datasets (atualização diária, dado nem sempre novo), o balanço é positivo: as execuções sem dado novo ficam muito mais baratas, e as execuções com dado novo têm ~1 minuto de overhead adicional num pipeline que tipicamente leva 5–15 minutos.

### Quando o overhead importa

O overhead é relevante se:
- O pipeline precisa terminar o mais rápido possível após a publicação do dado (SLA apertado)
- O flow é de atualização quase certa toda vez que roda (ex: taxa de câmbio, séries econômicas diárias) — nesses casos os 4 startups são inevitáveis sem ganho compensatório

Para datasets com atualização esporádica (CGU, TSE, datasets AU/US), o ganho nos runs de "sem atualização" compensa amplamente.

### Mitigação possível

**Work pool com workers persistentes** — em vez de pods efêmeros, um worker fica rodando no cluster e pega flows de uma fila sem startup por execução. Elimina o overhead mas adiciona custo de workers ociosos. Faz mais sentido para `check_update` (leve, frequente) do que para `prod_mat_test` (pesado, raro). Não é necessário implementar agora — é uma otimização possível se o overhead se provar um problema real em produção.

---

## Flows mais limpos — eliminação de boilerplate

Hoje cada flow repete o mesmo conjunto de partes padronizadas, independentemente do que o dataset realmente precisa:

```python
# padrão atual — todo flow tem isso
poll_source_for_update()       # check
upload_to_gcs(bucket="dev")    # download + staging dev
run_dbt(target="dev")          # gate dev
run_dbt(target="prod")         # materialização prod
register_table_materialization() # metadata
commit_source_update()         # commit da atualização
```

Com a nova arquitetura, cada flow contém **apenas o que é seu**:

```python
# check_update — só isso
poll_source_for_update()

# flow_download — só isso
download_source()
upload_to_gcs()

# dev_mat_test — só isso
run_dbt(target="dev")
run_dbt_test(target="dev")

# prod_mat_test — só isso
run_dbt(target="prod")
run_dbt_test(target="prod")
register_table_materialization()
commit_source_update()
```

O código específico de cada dataset fica restrito ao `flow_download` — que é a única parte que realmente varia entre datasets. O `check_update`, `dev_mat_test` e `prod_mat_test` podem ser flows **genéricos e reutilizáveis**, parametrizados por `dataset_id` e `table_id`.

---

## Variantes do pipeline

A arquitetura não precisa ser idêntica para todos os datasets. O que importa é que cada etapa entrega o mesmo resultado esperado pelo downstream — o formato do pipeline pode variar.

### Variante padrão — check separado do download

Para datasets onde o check é só uma chamada de API (ex: comparar uma data ou hash):

```
[check_update] → [flow_download] → [mat_test]
```

### Variante check_and_download — quando baixar é necessário para checar

Alguns flows precisam baixar o arquivo da fonte para conseguir verificar se o dado é novo. Nesses casos, `check_update` e `flow_download` se fundem num único flow:

```
[check_and_download] → [mat_test]
```

O `check_and_download` baixa da fonte, verifica se é novo, faz as transformações necessárias e já faz upload para o GCS staging. Se o dado não for novo, o flow conclui sem disparar o downstream.

**Tradeoff documentado:** se o `check_and_download` for separado em dois flows (`check_update` + `flow_download`), o `flow_download` precisaria baixar o arquivo novamente — mas do GCS em vez da fonte externa. Isso ainda é vantajoso (GCS é mais rápido, confiável e sem rate limit), mas adiciona um step que não existia num flow monolítico. Por isso, para esses casos, fundir em `check_and_download` é mais limpo.

### Levantamento dos crawlers existentes (completo — 2026-08-28)

Universo: ~82 datasets reais (excluindo test_dataset, test_pipeline, cross_update, delete_flows, botdosdados).

| Variante | Qtd | % (dos ativos com check) |
|---|---|---|
| **Padrão** (check via API/metadata leve) | 28 | 51% |
| **check_and_download** (baixa para checar) | 27 | 49% |
| **Sem check** (sempre executa) | 13 | — |
| **Inativos** (sem flows.py) | 12 | — |

**Conclusão: as duas variantes são igualmente comuns.** Nenhuma é exceção — ambas precisam ser tratadas como cidadãs de primeira classe na nova arquitetura.

#### Variante padrão — técnicas usadas
| Técnica | Exemplos |
|---|---|
| HEAD/PROPFIND no arquivo remoto (Last-Modified) | br_rf_cno, br_rf_cafir, br_rf_cnpj, br_me_cnpj, br_mf_divida_ativa |
| Listagem FTP sem download | br_me_caged, br_ms_cnes/sia/sih/sinan |
| Scraping HTML leve (datas em links/tabelas) | br_ibge_pnadc, br_me_comex_stat, us_sec_edgar |
| API de metadata BCB/IBGE/BD | br_bcb_agencia/estban/ifdata/sicor, br_ibge_inpc/ipca/ipca15, br_ms_cnes |
| Selenium lê atributo HTML (sem download) | br_mp_pep |
| Streaming GET lê só primeiros KB | us_cfpb_hmda |

#### Variante check_and_download — padrão recorrente
A maioria tem a sequência: `download()` → `poll_source_for_update_task()`, ou seja, o download acontece antes ou durante o check. Casos notáveis:
- **br_stf_corte_aberta**: Selenium dispara download automático de arquivo no browser
- **br_rj_isp**: compara contagem de linhas com BigQuery — tem que baixar para contar
- **br_tse_eleicoes**: baixa ZIPs para extrair data máxima do conteúdo
- **us_bls_qcew**: faz HEAD primeiro, mas ainda baixa o singlefile para escanear coluna

#### Sem check (executam incondicionalmente)
br_me_siconfi (revisões retroativas), br_me_rais, br_senado_dados_abertos, br_poder360_pesquisas, br_bcb_taxa_cambio, br_bd_indicadores, br_bd_siga_o_dinheiro, br_camara_dados_abertos, br_cgu_pessoal_executivo_federal, br_cvm_administradores_carteira, br_cvm_oferta_publica_distribuicao, br_fgv_igp, fundacao_lemann

#### Inativos (sem flows.py)
br_mercadolivre_ofertas, br_mg_belohorizonte_smfa_iptu, br_mp_pep_cargos_funcoes, br_ons_avaliacao_operacao, br_ons_estimativa_custos, br_sp_saopaulo_dieese_icv, br_tse_filiacao_partidaria, mundo_transfermarkt_competicoes, mundo_transfermarkt_competicoes_internacionais, world_sofascore_competicoes_futebol, world_wil_wid, br_b3_cotacoes

---

## Pendências antes de implementar

- [x] ~~Confirmar como a automação passa parâmetros ao flow downstream~~ — Jinja em `RunDeployment.parameters` interpola `{{ event.payload.chave }}`; usar `emit_event()` com payload arbitrário
- [ ] Definir o estado de saída do `check_update` quando não há dado novo (para não disparar o downstream)
- [ ] Decidir a granularidade das tags: por dataset ou por tabela?
- [ ] Definir onde fica o código compartilhado (`dbt run`, `dbt test`) — utils ou task separada?
- [ ] Avaliar impacto no deploy CI: hoje um PR com label `deploy-flow` deploya o flow inteiro; com 4 flows por dataset, o que muda?
- [ ] Decidir se `mat_test` é um flow genérico (parameterizado por dataset/table) ou um por dataset
- [x] ~~Completar levantamento dos datasets~~ — concluído (2026-08-28)

---

## Relacionado

- Issue #1767 — isolamento de cota BQ entre desenvolvimento e gate de prod
- Issue #1705 — reorganização da pasta `crawler/`
- Issue #1768 — logs de `_upload_to_gcs` não indicam o ambiente
