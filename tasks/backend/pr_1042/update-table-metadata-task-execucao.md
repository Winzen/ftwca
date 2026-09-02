# update_table_metadata_task — Execução de Teste (2026-07-22)

**Contexto:** Após merge do PR #1042 (fix do bucket requester pays), a task foi rodada manualmente para validar o fix.

---

## Como foi rodada

```bash
kubectl exec -it -n website api-prod-99df7f585-bn29q -- \
  /opt/pypoetry/virtualenvs/backend-9TtSrW0h-py3.11/bin/python /app/manage.py shell -c "
from backend.apps.api.v1.tasks import update_table_metadata_task
update_table_metadata_task.call_local()
"
```

A task ficou rodando em background. O output completo está em:

```
/tmp/claude-1000/-mnt-d-repositorios-bd-iac/07239d5d-56b0-4258-8ca4-e39062a9d203/tasks/bj03t4to3.output
```

> ⚠️ Esse caminho é da sessão do Claude Code na WSL. Se a sessão expirar, o arquivo pode não existir mais. Nesse caso, verificar diretamente nos logs do pod (ver abaixo).

---

## Como verificar o resultado

### 1. Contar quantas tabelas foram processadas

```bash
grep -c "tasks.py" /tmp/claude-1000/-mnt-d-repositorios-bd-iac/07239d5d-56b0-4258-8ca4-e39062a9d203/tasks/bj03t4to3.output
```

Esperado: ~858 (total de tabelas elegíveis em prod).

### 2. Verificar se houve erros

```bash
grep "ERROR\|BadRequest\|Exception" /tmp/claude-1000/-mnt-d-repositorios-bd-iac/07239d5d-56b0-4258-8ca4-e39062a9d203/tasks/bj03t4to3.output
```

Esperado: nenhuma linha.

### 3. Se o arquivo de output não existir mais — verificar nos logs do pod

```bash
kubectl get pods -n website | grep api-prod
kubectl logs -n website <pod> --since=12h | grep -E "update_table_metadata|ERROR|Task concluída"
```

### 4. Verificar no banco se os metadados foram preenchidos

Acessar o Metabase (db 133) e rodar:

```sql
SELECT
  COUNT(*) FILTER (WHERE uncompressed_file_size IS NULL AND number_rows IS NULL) AS ainda_null,
  COUNT(*) FILTER (WHERE uncompressed_file_size IS NOT NULL OR number_rows IS NOT NULL) AS preenchidos,
  COUNT(*) AS total
FROM public.api_table t
JOIN public.api_dataset d ON t.dataset_id = d.id
WHERE t.status_id NOT IN (
  SELECT id FROM public.api_status WHERE slug IN ('under_review', 'excluded')
)
AND d.status_id NOT IN (
  SELECT id FROM public.api_status WHERE slug IN ('under_review', 'excluded')
)
```

Antes do fix: 130 tabelas com ambos os campos NULL. Após: esperado próximo de 0.
