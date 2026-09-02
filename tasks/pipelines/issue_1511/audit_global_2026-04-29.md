# Audit de isPartition — 2026-04-29 (prod)

**Modo:** Encontradas (dry-run)  
**Tabelas verificadas:** 252  
**Colunas verificadas:** 12139  
**Alterações:** 2  
**Erros:** 0

## br_inmet_bdmep

### microdados (`br_inmet_bdmep.microdados`)

| Coluna | Antes | Depois | Motivo |
|--------|-------|--------|--------|
| ano | False | True | partition/cluster no SQL |
| id_estacao | False | True | partition/cluster no SQL |

---

# Validação BigQuery (4 fontes: SQL · BQ dev · BQ prod · API)

**Tabelas OK (4 fontes alinhadas):** 238  
**Divergências SQL ↔ BQ dev:** 9  
**Divergências SQL ↔ BQ prod:** 12  
**Divergências BQ dev ↔ BQ prod:** 13  
**Divergências BQ prod ↔ API:** 14  
**Datasets inacessíveis BQ dev:** 0  
**Datasets inacessíveis BQ prod:** 4

## Datasets inacessíveis no BQ prod
- `br_anatel_telefonia_movel`
- `br_b3_cotacoes`
- `br_bcb_agencia`
- `br_denatran_frota`

## Divergências SQL ↔ BQ dev
| Dataset.Tabela | Coluna | No SQL | No BQ dev | Nota |
|---|---|:---:|:---:|---|
| `br_geobr_mapas.setor_censitario_2010` | sigla_uf | ✓ | ✗ | SQL define mas BQ dev não tem |
| `br_ibge_censo_2022.cadastro_enderecos` | sigla_uf | ✓ | ✗ | SQL define mas BQ dev não tem |
| `br_inep_indicadores_educacionais.uf_taxa_transicao` | sigla_uf | ✓ | ✗ | SQL define mas BQ dev não tem |
| `mundo_transfermarkt_competicoes_internacionais.champions_league` | temporada | ✓ | ✗ | SQL define mas BQ dev não tem |
| `world_wb_mides.empenho` | mes | ✓ | ✗ | SQL define mas BQ dev não tem |
| `world_wb_mides.liquidacao` | mes | ✓ | ✗ | SQL define mas BQ dev não tem |
| `world_wb_mides.pagamento` | mes | ✓ | ✗ | SQL define mas BQ dev não tem |
| `world_wwf_hydrosheds.basins_atlas` | level | ✓ | ✗ | SQL define mas BQ dev não tem |
| `world_wwf_hydrosheds.lakes_atlas` | region | ✓ | ✗ | SQL define mas BQ dev não tem |

## Divergências SQL ↔ BQ prod
| Dataset.Tabela | Coluna | No SQL | No BQ prod | Nota |
|---|---|:---:|:---:|---|
| `br_anatel_telefonia_movel.microdados` | ano | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_anatel_telefonia_movel.microdados` | id_municipio | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_anatel_telefonia_movel.microdados` | mes | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_b3_cotacoes.cotacoes` | acao_atualizacao | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_b3_cotacoes.cotacoes` | data_referencia | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_bcb_agencia.agencia` | ano | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_denatran_frota.municipio_tipo` | ano | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_denatran_frota.municipio_tipo` | mes | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_geobr_mapas.setor_censitario_2010` | sigla_uf | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_ibge_censo_2022.cadastro_enderecos` | sigla_uf | ✓ | ✗ | SQL define mas BQ prod não tem |
| `br_inep_indicadores_educacionais.uf_taxa_transicao` | sigla_uf | ✓ | ✗ | SQL define mas BQ prod não tem |
| `mundo_transfermarkt_competicoes_internacionais.champions_league` | temporada | ✓ | ✗ | SQL define mas BQ prod não tem |

## Divergências BQ dev ↔ BQ prod
| Dataset.Tabela | Coluna | No BQ dev | No BQ prod | Nota |
|---|---|:---:|:---:|---|
| `br_anatel_telefonia_movel.microdados` | ano | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_anatel_telefonia_movel.microdados` | id_municipio | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_anatel_telefonia_movel.microdados` | mes | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_b3_cotacoes.cotacoes` | acao_atualizacao | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_b3_cotacoes.cotacoes` | data_referencia | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_bcb_agencia.agencia` | ano | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_denatran_frota.municipio_tipo` | ano | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `br_denatran_frota.municipio_tipo` | mes | ✓ | ✗ | BQ dev tem mas BQ prod não tem |
| `world_wb_mides.empenho` | mes | ✗ | ✓ | BQ prod tem mas BQ dev não tem |
| `world_wb_mides.liquidacao` | mes | ✗ | ✓ | BQ prod tem mas BQ dev não tem |
| `world_wb_mides.pagamento` | mes | ✗ | ✓ | BQ prod tem mas BQ dev não tem |
| `world_wwf_hydrosheds.basins_atlas` | level | ✗ | ✓ | BQ prod tem mas BQ dev não tem |
| `world_wwf_hydrosheds.lakes_atlas` | region | ✗ | ✓ | BQ prod tem mas BQ dev não tem |

## Divergências BQ prod ↔ API
| Dataset.Tabela | Coluna | No BQ prod | Na API | Nota |
|---|---|:---:|:---:|---|
| `br_anatel_telefonia_movel.microdados` | ano | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_anatel_telefonia_movel.microdados` | id_municipio | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_anatel_telefonia_movel.microdados` | mes | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_b3_cotacoes.cotacoes` | acao_atualizacao | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_b3_cotacoes.cotacoes` | data_referencia | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_bcb_agencia.agencia` | ano | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_denatran_frota.municipio_tipo` | ano | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_denatran_frota.municipio_tipo` | mes | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_geobr_mapas.setor_censitario_2010` | sigla_uf | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_ibge_censo_2022.cadastro_enderecos` | sigla_uf | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_inep_indicadores_educacionais.uf_taxa_transicao` | sigla_uf | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
| `br_inmet_bdmep.microdados` | ano | ✓ | ✗ | BQ prod tem partition/cluster mas API diz False |
| `br_inmet_bdmep.microdados` | id_estacao | ✓ | ✗ | BQ prod tem partition/cluster mas API diz False |
| `mundo_transfermarkt_competicoes_internacionais.champions_league` | temporada | ✗ | ✓ | API diz True mas BQ prod não tem partition/cluster |
