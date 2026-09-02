Perfeito — vou te montar uma documentação no estilo **Obsidian-friendly**, bem estruturada, direta e fácil de navegar.

---

# 📦 DBT Artifacts Upload (GCS)

## 🎯 Objetivo

Criar uma solução para capturar e armazenar os SQLs gerados pelo DBT (`compiled` e `run`) durante execuções em jobs do Prefect no Kubernetes (GCP).

Essa abordagem substitui o uso do Elementary para inspeção de queries e falhas.

---

## 🧠 Contexto

Anteriormente, utilizávamos o Elementary para:

- Visualizar materializações falhas
    
- Inspecionar queries geradas
    

### Problemas identificados:

- Alto consumo de CPU
    
- Aumento significativo no tempo de execução (`dbt run` / `dbt test`)
    
- Baixa acessibilidade dos resultados pela equipe
    
- Delay na atualização das informações
    

---

## 💡 Solução

Implementação da classe `DBTArtifactUploader`, responsável por:

- Ler a pasta `target/` gerada pelo DBT
    
- Filtrar apenas:
    
    - `compiled/`
        
    - `run/`
        
- Enviar os arquivos para um bucket no GCS
    
- Sobrescrever apenas o diretório do modelo correspondente
    

---

## 🏗️ Arquitetura

```text
DBT run/test
    ↓
gera target/
    ↓
DBTArtifactUploader (finally)
    ↓
upload → GCS (dbt-artifacts/)
```

---

## ⚙️ Comportamento

### Execução

|Ambiente|Executa upload|
|---|---|
|Kubernetes|✅ Sempre|
|Local|❌ Não (default)|
|Local + debug|✅ (`enable_upload=True`)|

---

### Estrutura no bucket

```text
dbt-artifacts/
  compiled/
    project/model.sql
  run/
    project/model.sql
```

---

### Estratégia de sobrescrita

- Antes do upload:
    
    - Deleta apenas o diretório do modelo
        
- Exemplo:
    

```text
run/basedosdados/models/br_me_cnpj/
```

👉 Apenas esse path é removido e reescrito

---

## 🧩 Integração com `run_dbt`

O uploader é executado via bloco `finally`:

```python
try:
    # execução DBT
finally:
    uploader.run()
```

### Garantia

- Executa sempre (sucesso ou erro)
    
- Não interfere no resultado do DBT
    
- Falhas no upload são apenas logadas
    

---

## 🧱 Estrutura da Classe

### Responsabilidades

- `_list_target_files` → leitura do filesystem
    
- `_get_model_prefix` → regra de negócio (modelo DBT)
    
- `_delete_prefix` → limpeza no GCS
    
- `_upload_file` → upload
    

---

### Método principal

```python
uploader.run()
```

---

## 🔐 Controle de execução

```python
def _should_run(self):
    if self._is_running_in_kubernetes():
        return True
    return self.enable_upload
```

---

## 🧪 Testes

### Cenários validados

- ✅ DBT com sucesso → upload realizado
    
- ✅ DBT com falha → upload realizado
    
- ✅ Execução em Kubernetes → upload automático
    
- ✅ Execução local sem flag → não executa
    
- ✅ Execução local com flag → executa
    

---

## 📊 Logs

Exemplo:

```text
Fazendo upload dos arquivos da pasta target gerada pelo DBT!
```

---

## 🚀 Benefícios

- Acesso direto aos SQLs gerados
    
- Redução de custo computacional
    
- Execuções DBT mais rápidas
    
- Independência do Elementary
    
- Debug mais simples
    

---

## ⚠️ Considerações

- Upload pode gerar muitos arquivos pequenos no GCS
    
- Sem versionamento (sobrescreve por modelo)
    
- Dependente de permissões GCP corretamente configuradas
    

---

## 🔮 Possíveis melhorias

- Versionamento por timestamp ou run_id
    
- Paralelização de uploads
    
- Upload apenas de modelos alterados
    
- Integração com `manifest.json`
    
- Compressão antes do upload
    

---

Se quiser, posso adaptar isso para um formato ainda mais “Obsidian power user”, com:

- links internos (`[[...]]`)
    
- tags (#dbt #gcp #prefect)
    
- ou separar em múltiplas notas menores (arquitetura / código / decisões)