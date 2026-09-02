
Perfeito! Aqui está um `.md` pronto para você salvar no Obsidian, já com título, descrição, causa, diagnóstico e solução de forma clara e casual:

````markdown
# Problema Prefect Agent com template.yaml do GCS

## Data do Incidente
26/02/2026

## Contexto
O **Prefect Agent** estava subindo via Kubernetes com o comando:

```bash
prefect agent kubernetes start \
  --job-template https://storage.googleapis.com/basedosdados-dev/prefect_job_template/template.yaml
````

O job template é baixado diretamente do bucket do Google Cloud Storage (`basedosdados-dev`) quando o agent inicia.

No mesmo dia, houve uma alteração na política do bucket, que passou a ser **Requester Pays**.

---

## Sintomas

- O agent falhou ao iniciar.
    
- Erro no log:
    

```
AttributeError: 'str' object has no attribute 'setdefault'
```

- Nenhuma alteração foi feita no código do flow ou no Prefect Agent.
    

---

## Causa

A política **Requester Pays** do bucket passou a exigir que todo acesso informe um **projeto de billing**.  
Como o Prefect acessa o template via URL direta (`https://storage.googleapis.com/...`) sem informar projeto de faturamento, o download do YAML falhou.

Isso fez com que o conteúdo lido não fosse o YAML esperado, e o Prefect tentou processar uma string inválida como se fosse um dicionário, gerando o `AttributeError`.

**Resumo simples:**

> O problema não estava no Loki nem no Kubernetes. Foi uma mudança na política do bucket que quebrou o download do job template.

---

## Diagnóstico

1. Verificação do Prefect Agent:
    

```bash
kubectl describe pod prefect-agent-86b4f89bb8-b74v6 -n prefect
```

- Imagem do agent estava fixa (`prefecthq/prefect:0.15.9`)
    
- Sem reinícios recentes
    
- Configuração de ambiente correta
    

2. Confirmação do bucket:
    

```bash
gsutil ls -L -b gs://basedosdados-dev
```

- Mostrou `Requester Pays: Enabled`
    

3. Verificação do comando de startup do agent confirmou que o template era baixado diretamente do bucket.
    

---

## Solução / Ações

- Entender que o bucket é **Requester Pays**.
    
- Para contornar:
    
    - Usar um **projeto de billing** ao baixar via `gs://` com `gsutil` (não funciona via URL pública)
        
    - Ou baixar o template previamente e montar via **ConfigMap** ou volume no container
        
    - Ou desativar temporariamente Requester Pays, se permitido
        

---

## Observações

- O incidente coincidiu com uma manutenção no Loki, mas não teve relação.
    
- Foi um caso típico de **mudança de política em infra** impactando um workflow que depende de recursos externos.
    
- Esse registro serve para referência futura caso o mesmo bucket ou outros buckets Requester Pays impactem o Prefect.
    

---

## Referência do comando que quebrou

```bash
prefect agent kubernetes start \
  --job-template https://storage.googleapis.com/basedosdados-dev/prefect_job_template/template.yaml
```

---

```

Se você quiser, posso também criar uma versão **bem resumida estilo postmortem interno**, pronta para colar em Slack ou Notion, em 1 página só.  
Quer que eu faça?
```