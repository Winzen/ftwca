### Acessar POD
#### PROD

```
kubectl exec -it <nome-do-pod> -n <namespace> -- /bin/sh
```
#### Dev
```
kubectl exec -it <nome-do-pod> -n <namespace> -- /bin/sh
```
### Export banco de dados

```
kubectl port-forward svc/cloud-sql-proxy 15432:5432 -n cloud-sql-proxy
```
### Fazer backup de um APP especifico

```
python manage.py dumpdata v1 account --indent 2 -e contenttypes --natural-foreign --natural-primary --output=backup_prod.json
```
```
python manage.py dumpdata v1 account auth djstripe --indent 2 -e contenttypes  --output=backup_prod_v1.json
```
```
python manage.py dumpdata v1.entity --indent 2 --output=backup_prod_v1_entity.json
```

Comando final para funciona
```
python manage.py dumpdata v1 --indent 2 --output=backup_prod_v1.json
```
```
kubectl exec -it <nome-do-pod> -n <namespace> -- python manage.py dumpdata v1 --indent 2 --output=backup_prod_v1.json
```
Load:
```
python manage.py loaddata backup_prod_v1.json
```
### Copiar o arquivo do POD

```
kubectl cp -n <namespace> <nome-do-pod>:backup_prod_v1.json backup_prod_v1.json
```
```
kubectl cp -n <namespace> backend/apps/core/management/commands/__utils.py <nome-do-pod>:/app/backend/apps/core/management/commands/__utils.py
```
```
kubectl cp -n <namespace> <nome-do-pod>:/app/backend/apps/core/management/commands/__utils.py backend/apps/core/management/commands/__utils.py 
```

#### Manda task para prod

```
kubectl cp /mnt/d/repositorios/bd/backend/backend/apps/user_notifications/management/commands/send_notification.py -n <namespace> <nome-do-pod>:/app/backend/apps/user_notifications/management/commands/send_notification.py
```
```
kubectl cp <nome-do-pod>:/app/backend/apps/user_notifications/management/commands/send_notification.py -n <namespace> /mnt/d/repositorios/bd/backend/backend/apps/user_notifications/management/commands/send_notification.py 
```
### Copiar para o bucket

```
gsutil cp backup_prod_v1.json gs://<bucket>/backend_backup/backup_prod_v1.json
```
### Acessar Banco de Dados Django
```
python manage.py dbshell
```


Detalhes:

### Detalhes: Fazer backup de um APP especifico

Os parâmetros `--natural-foreign`, `--natural-primary` e `--pks` são opções do comando `dumpdata` do Django, que serve para exportar os dados de um banco de dados para o formato JSON. Vou explicar cada um desses parâmetros com mais detalhes:

### 1. `--natural-foreign`

Esse parâmetro instrui o comando `dumpdata` a exportar as chaves estrangeiras (`ForeignKey`) usando os valores naturais (ou seja, valores legíveis para o ser humano, como `nome` ou `slug` ao invés do ID numérico do objeto relacionado).

Por exemplo, se você tem uma tabela `Author` e uma tabela `Book` em que cada livro tem uma chave estrangeira apontando para o autor, o comportamento padrão do Django ao exportar os dados seria salvar o ID do autor. Com `--natural-foreign`, o comando vai tentar substituir esse ID pelo valor legível do campo que representa a chave primária natural do modelo `Author`, como o nome do autor.

#### Exemplo:

- Sem `--natural-foreign`:
    
    ```json
    {
        "model": "app.book",
        "pk": 1,
        "fields": {
            "title": "Some Book",
            "author": 5
        }
    }
    ```
    
- Com `--natural-foreign` (supondo que o autor tenha o nome "John Doe"):
    
    ```json
    {
        "model": "app.book",
        "pk": 1,
        "fields": {
            "title": "Some Book",
            "author": "John Doe"
        }
    }
    ```
    

### 2. `--natural-primary`

Esse parâmetro instrui o `dumpdata` a exportar as chaves primárias dos modelos usando valores naturais, quando possível. Ou seja, ao invés de exportar o ID numérico padrão como chave primária, o Django tentará usar outro campo único e legível para ser a chave primária no arquivo de exportação.

Por exemplo, se um modelo tem um campo `slug` que é único e usado como chave primária (em vez do ID), o `--natural-primary` faz com que o Django use esse `slug` no lugar do ID.

#### Exemplo:

- Sem `--natural-primary`:
    
    ```json
    {
        "model": "app.book",
        "pk": 1,
        "fields": {
            "slug": "some-book"
        }
    }
    ```
    
- Com `--natural-primary` (se o campo `slug` for usado como chave primária):
    
    ```json
    {
        "model": "app.book",
        "pk": "some-book",
        "fields": {
            "slug": "some-book"
        }
    }
    ```
    

### 3. `--pks PRIMARY_KEYS`

Esse parâmetro permite especificar quais chaves primárias devem ser exportadas. Ele é usado quando você deseja exportar apenas um subconjunto específico de objetos de um modelo, com base em suas chaves primárias. Você pode passar uma lista de chaves primárias ou um intervalo de chaves para filtrar quais registros serão exportados.

#### Exemplo:

- Suponha que você tenha um modelo `Book` com IDs 1, 2, 3, 4, e 5, e deseja exportar apenas os livros com IDs 1, 3 e 5:
    
    ```bash
    python manage.py dumpdata app.book --pks 1 3 5
    ```
    

Isso geraria a exportação apenas para os livros com as chaves primárias especificadas.

---

### Resumo:

- **`--natural-foreign`**: Exporte chaves estrangeiras com valores naturais (em vez de IDs numéricos).
    
- **`--natural-primary`**: Exporte chaves primárias com valores naturais (se o modelo tiver um campo natural configurado para ser a chave primária).
    
- **`--pks PRIMARY_KEYS`**: Exporte apenas os objetos com as chaves primárias especificadas.
    

Esses parâmetros são úteis principalmente quando você deseja ter uma exportação mais legível e/ou focada em um conjunto específico de dados.
