# Prefect 3 — Situação da Migração (2026-06-10)

## Resumo executivo

| Status                                  | Flows   |
| --------------------------------------- | ------- |
| ✅ Migrado (rodando em prod no P3)       | **160** |
| ⚠️ Bloqueado (bug no código ou memória) | **10**  |
| ⚠️ Schedule desativado (NÃO ATIVAR)     | **5**   |
| ❌ Não migrado (sem flows.py)            | **2**   |
| **Total rastreado**                     | **177** |

**Progresso: 160/177 — 90%**

O Prefect 0 foi desativado para todos os 160 flows migrados. O Prefect 3 está rodando em prod. Resta resolver os 15 flows com problemas conhecidos e confirmar o status dos 2 sem código.

---

## O que foi feito hoje (2026-06-10)

### Lotes migrados (111 flows)

Flows com status "🟢 Dev OK" no tracking, ativados em prod no P3 e desativados no P0:

| Lote | Dataset(s) | Flows |
|---|---|---|
| 1 | br_camara_dados_abertos | 26 |
| 2 | br_ibge_ipca, br_ibge_ipca15 | 8 |
| 3 | br_ibge_inpc, br_bcb_estban, br_me_comex_stat, br_bcb_agencia, br_anatel_banda_larga_fixa | 15 |
| 4 | br_anp_precos_combustiveis, br_cgu_beneficios_cidadao, br_cgu_emendas_parlamentares, br_cgu_licitacao_contrato, br_cgu_servidores_executivo_federal, br_denatran_frota, br_cvm_administradores_carteira, br_cvm_fi, br_cvm_oferta_publica_distribuicao | 31 |
| 5 | br_me_cnpj, br_ms_cnes, br_ms_sia, br_ms_sih, br_ms_sinan, br_rf_cafir, br_rf_cno, br_rj_isp | 31 |

### Flows ativados direto em prod (26 flows — sem passar pelo dev)

Estes estavam deployados no P3 com `paused=False` mas sem schedule (`crons=[]`). Foram confirmados como inativos no P0 (apenas 2 com agendamento ativo, ambos desativados). Ativados diretamente com o cron correto extraído do código `flows.py`:

- `br_fgv_igp` (7 flows)
- `br_bd_indicadores` (8 flows — sem schedule, `paused=False` já ativo)
- `br_tse_eleicoes` (4 flows — sem schedule)
- `br_ans_beneficiario__microdados`
- `br_ibge_pnadc__microdados`
- `br_me_rais__microdados` + `br_me_rais__vinculo` (sem schedule)
- `br_bd_siga_o_dinheiro__emendas_parlamentares` (sem schedule)
- `fundacao_lemann__ideb_municipio_e_uf__escola`
- `br_bcb_sicor__dicionario` (sem schedule)

### Já em prod antes de hoje (16 flows)

| Dataset | Flows |
|---|---|
| br_bcb_sicor (10 tabelas) | `divida_ativa`, `empreendimento`, `complemento_operacao`, `garantia`, `liberacao_recursos`, `listagem_pendente`, `operacao`, `propriedade`, `saldos_posicao`, `subtema` |
| br_inmet_bdmep | `microdados` |
| br_me_caged (3 tabelas) | `microdados_antigos`, `microdados_movimentacao`, `microdados_estabelecimento` |
| br_bcb_taxa_cambio | `diario` |
| br_sfb_sicar | `area_imovel` |

---

## Flows bloqueados — fix necessário

Estes flows estão deployados no P3 **sem schedule ativo**. O P0 já foi desativado. Precisam de fix no código antes de ativar o schedule.

### OOMKilled — aumentar memory limit

| Flow | Problema |
|---|---|
| `br_anatel_telefonia_movel__microdados` | Pod OOMKilled ao baixar ZIP da Anatel (>4Gi) |
| `br_anatel_telefonia_movel__densidade_municipio` | Mesmo problema |
| `br_anatel_telefonia_movel__densidade_uf` | Mesmo problema |
| `br_anatel_telefonia_movel__densidade_brasil` | Mesmo problema |
| `br_cvm_fi__documentos_perfil_mensal` | Pod evicted por MemoryPressure no node |
| `br_cvm_fi__documentos_balancete` | Pod OOMKilled (12 meses × CSVs grandes) |

**Fix:** adicionar `job_variables` no deployment com `memory: 8Gi` (ou mais). O trabalho de migração do código está correto — é só limite de recurso.

```python
# No flows.py, adicionar ao deploy:
flow.deploy_schedules = [...]
# E via API PATCH /api/deployments/{id}:
{"job_variables": {"memory": "8Gi"}}
```

Alternativamente, pode-se otimizar o código para processar em streaming sem carregar tudo na memória.

### Bug no código do crawler

| Flow | Erro | Fix |
|---|---|---|
| `br_cgu_pessoal_executivo_federal__terceirizados` | `UnicodeDecodeError: 'utf-8' codec can't decode byte 0xba` ao ler CSV | Adicionar `encoding='latin-1'` na leitura do CSV |
| `br_cgu_emendas_parlamentares__microdados` | `FileNotFoundError: /tmp/input/EmendasParlamentares.csv` — download/unzip falhou silenciosamente | Debug do `download_file_from_url` — verificar URL, user-agent, ou usar curl subprocess |
| `br_ms_cnes__regra_contratual` | `ParserError: Error tokenizing data` — CSV malformado ou separador errado | Inspecionar o arquivo fonte e ajustar o read_csv (sep, quoting, error_bad_lines) |
| `br_rj_isp_estatisticas_seguranca__armas_apreendidas_mensal` | `KeyError` — colunas removidas da fonte ISP | A fonte mudou estrutura; o código precisa ser atualizado para as colunas atuais |

---

## Flows com schedule desativado — NÃO ATIVAR até fix

Estes flows estão deployados no P3 com `paused=True`. O P0 já foi desativado. São flows que foram testados em prod e falharam repetidamente (6 falhas consecutivas, 03-08/06). **Não ativar o schedule antes de corrigir o bug.**

| Flow | Erro | Fix necessário |
|---|---|---|
| `br_bcb_taxa_selic` | HTTP 406 — API do BCB rejeita o User-Agent padrão do `requests` | Adicionar `headers={"User-Agent": "Mozilla/5.0 ..."}` na task `get_selic_data` |
| `br_cnj_improbidade_administrativa__condenacao` | `ConnectionError` intermitente | Investigar estabilidade da fonte; adicionar retry com backoff |
| `br_mp_pep__cargos_funcoes` | Selenium Timeout — webdriver expirando | Aumentar timeout ou adicionar retry no Selenium; verificar se o site mudou |
| `br_poder360_pesquisas__microdados` | `KeyError: 'BIG THEME'` + `ConnectionError` | A estrutura do dado da fonte mudou; atualizar o parser |
| `br_stf_corte_aberta__decisoes` | Selenium Timeout | Mesmo problema do `br_mp_pep` |

---

## Flows sem código — confirmar

| Pasta | Situação | Ação |
|---|---|---|
| `pipelines/datasets/br_jota/` | Pasta existe, sem `flows.py` | Confirmar se é pipeline ativo ou pode ser removido |
| `pipelines/datasets/br_mp_pep_cargos_funcoes/` | Pasta existe, sem `flows.py`; provavelmente duplicata de `br_mp_pep` | Confirmar e remover se for duplicata |

---

## Estado do Prefect 0

Todos os 160 flows migrados foram desativados no Prefect 0 via mutation `set_schedule_inactive`. Os flows bloqueados/desativados também foram desativados no P0 (eles já estavam inativos lá).

O Prefect 0 continua rodando no namespace `prefect` mas **sem flows com schedule ativo**.

---

## Commits pendentes

O arquivo `prefect3-flows-migrados.md` no repositório `pipelines` tem as marcações dos 42 flows mais recentes como ✅ Migrado **não commitadas** (os 111 anteriores foram commitados em `a25208e4`).

Quando for commitar:
```bash
cd /mnt/d/repositorios/bd/pipelines
git add prefect3-flows-migrados.md
git commit -m "docs: mark 42 more flows as migrated (2026-06-10)"
git push
```

---

## Próximos passos prioritários

1. **Fix br_bcb_taxa_selic** — user-agent de 1 linha, impacto alto (dado financeiro diário)
2. **Fix br_anatel_telefonia_movel** — 4 flows, só precisa de `job_variables` com mais memória
3. **Fix br_cgu_pessoal_executivo_federal__terceirizados** — encoding latin-1, 1 linha
4. **Fix br_cvm_fi** — 2 flows, mesmo fix de memória do anatel
5. **Investigar br_cgu_emendas_parlamentares** — download silencioso falhando
6. **Confirmar br_jota e br_mp_pep_cargos_funcoes** — limpar ou migrar
7. **Fix Selenium flows** (br_mp_pep, br_stf_corte_aberta) — mais complexo, menor urgência se não têm SLA diário

---

## Referências

- Tracking detalhado: `pipelines/prefect3-flows-migrados.md`
- Lotes migrados: `Vaults/IAC/Prefect 3 Migração Lotes P0→P3.md`
- Infraestrutura geral: `Vaults/IAC/Prefect 3 Pendencias.md`
- Recursos dos pods: `Vaults/IAC/Prefect 3 Recursos dos Pods.md`
