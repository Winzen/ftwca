

# Limpeza e correção de registros na tabela `flow` e `flow_run`

## Contexto

Foram identificados registros na tabela `flow` que:

- Estão **arquivados (`archived = true`)**
    
- Não possuem execuções associadas na tabela `flow_run`
    
- Podem estar com **schedule ativo indevidamente**
    

Além disso, foi identificado um volume de **execuções antigas na tabela `flow_run`** que não eram mais necessárias para o funcionamento atual dos workflows.

**Devido ao grande número de registros afetados, foi decidido executar queries diretas no banco de dados** em vez de chamadas individuais para a API do Prefect via GraphQL. Essa abordagem foi escolhida para reduzir significativamente o tempo de execução e evitar sobrecarga de requisições.

Esses casos podem gerar inconsistências ou manter configurações ativas para flows que não deveriam mais executar.

---

# Sumário

Resultado da execução das queries de limpeza:

- **196.553** flows arquivados removidos por **não possuírem nenhum `flow_run`**
    
- **392** registros da tabela **`flow_run`** removidos por terem **mais de 1 ano**
    
- **875** flows removidos por **não possuírem nenhum `flow_run` associado**
    

---

# Tarefas concluídas

- Removidas execuções (`flow_run`) com **mais de 1 ano**
    
- Removidos **flows arquivados sem execuções**
    
- Desativados **schedules ativos em flows arquivados**
    
- Removidos **flows que ficaram sem execuções após a limpeza**
    

> Antes da execução de cada `DELETE`, foi realizada uma verificação utilizando `COUNT` para validar a quantidade de registros que seriam afetados.

---

# 1. Remover execuções (`flow_run`) mais antigas que 1 ano

## Objetivo

Reduzir o volume da tabela `flow_run` removendo execuções muito antigas, que não são mais necessárias para auditoria ou operação.

## Query de verificação

Antes da remoção, foi executada uma query para verificar **quantos registros seriam removidos**:

```sql
SELECT count(*)
FROM flow_run
WHERE scheduled_start_time < NOW() - INTERVAL '1 year';
```

## Query de remoção

Após a validação do volume de dados, foi executada a remoção das execuções antigas:

```sql
DELETE FROM flow_run
WHERE scheduled_start_time < NOW() - INTERVAL '1 year';
```

---

# 2. Remover flows arquivados sem execução

## Objetivo

Excluir registros da tabela `flow` que:

- Estão marcados como `archived = true`
    
- **Não possuem registros relacionados** na tabela `flow_run`
    

Isso remove flows arquivados que nunca foram executados ou que ficaram sem execuções após a limpeza da tabela `flow_run`.

## Query de verificação

```sql
SELECT count(*)
FROM flow f
WHERE f.archived = true
AND NOT EXISTS (
    SELECT 1
    FROM flow_run fr
    WHERE fr.flow_id = f.id
);
```

## Query de remoção

```sql
DELETE FROM flow f
WHERE f.archived = true
AND NOT EXISTS (
    SELECT 1
    FROM flow_run fr
    WHERE fr.flow_id = f.id
);
```

---

# 3. Desativar schedule de flows arquivados

## Objetivo

Garantir que flows arquivados não tenham agendamentos ativos.

## Query de verificação

```sql
SELECT count(*)
FROM flow
WHERE archived = true
AND is_schedule_active = true;
```

## Query de atualização

```sql
UPDATE flow
SET is_schedule_active = false
WHERE archived = true
AND is_schedule_active = true;
```

---

# 4. Remover flows sem execuções

## Objetivo

Após a limpeza das execuções antigas (`flow_run`), foi realizada uma verificação para identificar **flows que ficaram sem nenhuma execução associada**.

Esses registros podem ser flows criados para testes ou que não possuem mais utilidade no sistema.

## Query de verificação

```sql
SELECT count(*)
FROM flow f
WHERE NOT EXISTS (
    SELECT 1
    FROM flow_run fr
    WHERE fr.flow_id = f.id
);
```

## Query de remoção

```sql
DELETE FROM flow f
WHERE NOT EXISTS (
    SELECT 1
    FROM flow_run fr
    WHERE fr.flow_id = f.id
);
```

---

