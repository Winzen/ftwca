
---

# Integração do Banco de Dados com Metabase e Dashboards

Anteriormente, havia a intenção de criar uma tabela de acompanhamento diretamente no **Admin do Django**. Entretanto, durante a análise, foi reconhecido que a abordagem mais adequada seria **conectar o banco PostgreSQL diretamente ao Metabase**, permitindo visualizações mais completas, seguras e dinâmicas, sem necessidade de criar telas adicionais na aplicação.
## Contexto

Foi realizada a integração do banco de dados PostgreSQL ao Metabase, permitindo a visualização e análise dos dados de flows e execuções. A partir dessa integração, foram construídas queries para suportar indicadores e visualizações nos dashboards.

---

## Conexão do Banco de Dados

A conexão entre o Metabase e o banco PostgreSQL foi realizada de forma segura utilizando o Cloud SQL Proxy como intermediário.

### Estratégia de Conexão

- O host configurado no Metabase foi o endpoint local exposto pelo Cloud SQL Proxy (`localhost`)
    
- O proxy estabelece um túnel seguro com a instância do banco no Google Cloud, evitando exposição pública
    
- A senha do banco foi obtida a partir de pods em execução no Kubernetes, já injetada via variáveis de ambiente ou secrets
    
- Essa abordagem evita criação manual de credenciais ou exposição de secrets
    

### Fluxo da Conexão

```text
Metabase → localhost (Cloud SQL Proxy) → Instância PostgreSQL (GCP)
```

### Boas Práticas Adotadas

- Usuário read-only dedicado ao Metabase
    
- Restrição de acesso por rede/IP
    
- Uso de proxy e SSL/TLS para segurança
    

---

## Tabelas Utilizadas

- `public.flow` — contém os metadados dos flows
    
- `public.flow_run` — armazena execuções dos flows
    
- `public.project` — informações de projetos relacionados aos flows
    

---

## Dashboards Criados

### Flows Ativos

**Objetivo:** Esta query tem como objetivo listar todos os flows com agendamento ativo pertencentes ao projeto `main`, permitindo um acompanhamento rápido do que está em execução programada.

**Query de exemplo:**

```sql
SELECT
  f.id,
  f.name,
  f.is_schedule_active,
  p.name AS project_name,
  'https://prefect.basedosdados.org/flow/' || f.id AS link
FROM public.flow f
LEFT JOIN public.project p ON f.project_id = p.id
WHERE f.is_schedule_active = TRUE
  AND p.name = 'main';
```

**Utilidade no Dashboard:**

- Monitorar quais flows estão ativos
    
- Facilitar navegação direta para o Prefect
    
- Auxiliar em auditoria e validação de agendamentos
    

---

### Flows Inativos

**Objetivo:** Esta query tem como objetivo listar os flows que não possuem agendamento ativo no projeto `main` e que não estão arquivados, permitindo acompanhamento de flows parados.

**Query de exemplo:**

```sql
SELECT
  f.id,
  f.name,
  f.is_schedule_active,
  p.name AS project_name,
  'https://prefect.basedosdados.org/flow/' || f.id AS link
FROM public.flow f
LEFT JOIN public.project p ON f.project_id = p.id
WHERE f.is_schedule_active = FALSE
  AND p.name = 'main'
  AND f.archived = FALSE
LIMIT 1048575;
```

**Utilidade no Dashboard:**

- Identificar flows desativados
    
- Apoiar auditoria e revisão operacional
    
- Detectar flows que podem ser reativados ou limpos
    

---

### Distribuição de Execuções por Horário

**Objetivo:** Esta query tem como objetivo analisar a distribuição dos horários de execução dos flows ativos, considerando o horário mais recente agendado para cada flow.

**Query de exemplo:**

```sql
SELECT
  hour,
  COUNT(*) AS total_flows
FROM (
  SELECT DISTINCT ON (f.name)
    f.name,
    TO_CHAR(fr.scheduled_start_time, 'HH24:MI') AS hour
  FROM public.flow f
  LEFT JOIN public.flow_run fr ON f.id = fr.flow_id
  WHERE f.archived = FALSE 
    AND f.is_schedule_active = TRUE
  ORDER BY f.name, fr.scheduled_start_time DESC
) t
GROUP BY hour
ORDER BY hour;
```

**Utilidade no Dashboard:**

- Visualizar concentração de execuções ao longo do dia
    
- Detectar horários com maior carga
    
- Apoiar balanceamento de schedules
    

---

### Visão Geral de Status dos Flows

**Objetivo:** Esta query tem como objetivo fornecer uma visão consolidada do status dos flows, calculando totais de flows ativos, desativados e percentual de ativos no projeto `main`.

**Query de exemplo:**

```sql
SELECT
  COUNT(*) FILTER (WHERE f.is_schedule_active = TRUE AND f.archived = FALSE) AS ativos,
  COUNT(*) FILTER (WHERE f.is_schedule_active = FALSE AND f.archived = FALSE) AS desativados,
  COUNT(*) FILTER (WHERE f.archived = FALSE) AS total,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE f.is_schedule_active = TRUE AND f.archived = FALSE)
    / NULLIF(COUNT(*) FILTER (WHERE f.archived = FALSE), 0),
    2
  ) AS percentual_ativo
FROM public.flow f
LEFT JOIN public.project p ON f.project_id = p.id
WHERE p.name = 'main';
```

**Utilidade no Dashboard:**

- KPI rápido de saúde operacional
    
- Monitorar proporção de flows ativos
    
- Auxiliar decisões de governança
    

---

### Flows com Mais Falhas (Últimos 7 dias)

**Objetivo:** Esta query tem como objetivo identificar os flows que mais apresentaram falhas nos últimos 7 dias, excluindo flows template irrelevantes.

**Query de exemplo:**

```sql
SELECT
    f.name AS flow_name,
    COUNT(*) AS count
FROM flow_run fr
JOIN flow f ON fr.flow_id = f.id
WHERE
    f.name IS NOT NULL
    AND (
        f.name <> 'BD template: Executa DBT model'
        OR f.name IS NULL
    )
    AND (
        f.name <> 'BD template: Ingerir tabela zipada para GCS'
        OR f.name IS NULL
    )
    AND fr.state = 'Failed'
    AND fr.start_time >= date_trunc(
        'day',
        (CURRENT_TIMESTAMP AT TIME ZONE 'America/Sao_Paulo') - INTERVAL '7 days'
    )
    AND fr.start_time < date_trunc(
        'day',
        CURRENT_TIMESTAMP AT TIME ZONE 'America/Sao_Paulo'
    )
GROUP BY f.name
ORDER BY count DESC, f.name ASC
LIMIT 15;
```

**Utilidade no Dashboard:**

- Identificar flows problemáticos
    
- Priorizar correções
    
- Monitorar estabilidade de pipelines
    

---

### Flows Arquivados com Schedule Ativo

**Objetivo:** Esta query tem como objetivo identificar a quantidade de flows arquivados que ainda possuem agendamento ativo, indicando possíveis inconsistências operacionais.

**Query de exemplo:**

```sql
SELECT count(*)
FROM flow
WHERE archived = true
AND is_schedule_active = true;
```

**Utilidade no Dashboard:**

- Detectar inconsistências de configuração
    
- Prevenir execuções indesejadas
    
- Apoiar ações de limpeza e governança
    

---

### Flows Arquivados com Execuções Agendadas

**Objetivo:** Esta query tem como objetivo identificar a quantidade de flows arquivados que ainda possuem execuções agendadas, indicando possíveis inconsistências operacionais.

**Query de exemplo:**

```sql
SELECT count(*)
FROM flow f
WHERE f.archived = true
AND EXISTS (
    SELECT 1
    FROM flow_run fr
    WHERE fr.flow_id = f.id and fr.state = 'Scheduled'
);
```

**Utilidade no Dashboard:**

- Detectar execuções pendentes de flows arquivados
    
- Garantir consistência entre estado de arquivamento e agendamento
    
- Apoiar ações de governança
    

---

## Próximos Passos Recomendados

- Refinar métricas e KPIs do dashboard
    
- Adicionar filtros dinâmicos (por projeto, status ou horário)
    
- Criar alertas automáticos com base em falhas ou execuções inconsistentes
    
- Automatizar verificação de flows arquivados com execuções ativas para manutenção proativa
    

---

Se você quiser, posso gerar **uma versão final formatada para GitHub**, com todos os blocos de código e títulos organizados, pronta para colar na issue sem ajustes.

Quer que eu faça isso agora?