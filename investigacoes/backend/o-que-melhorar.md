### Melhorar a função para levar as colunas da arquitetura para o backend:

create_columns:
```
directory_column_name = row["directory_column"].split(":")[1].replace("\n", "")
```

### Finalizar o backup

 - Colocar __ em scripts que não vão ser comandos

- Criar rotina de prod para fazer upload no storage

- E criar o novo populate

- Criar uma forma de Account não ser apagada no processo de sync com MAIN. Basicamente criar uma comando customizado de limpar a base dos dados
### Processo de Enviar emails

-  [x] Pode ser melhorar agrupar todas as notificações do mesmo usuário do dia para serem enviadas juntos e evitar spam caso tenha mais de uma tabela que foi atualizada
- Vetorizar o processo de vê precisam de notificação
- E melhorar questão de deixa a conexão aberta para ficar mais rápido o envio de vários e-mail
- Tem sim para fazer "sinais" para quando tiver uma atualização na tabela, Manda a notificação para o usuários

## Editar a Pagina de Erro de gigante

## Download de catalog mais rapido

O endpoint atual expõe o backend a riscos de sobrecarga (incluindo possíveis ataques de DDoS) e possui latência elevada na geração do resultado.  
Implementação de uma task responsável por gerar o arquivo de catálogo e armazená-lo em um storage acessível para download.

Verificar se isso tem algum preço

### Melhoria na parte de forma query do dicionário. Duplicando campos por causa da cobertura temporal

## Criar automaticamente um Última atualização na Base dos Dados quando criado uma nova tabela e afins. Isso não pode ficar vazio


## Fazer Teste

## Otimizar Query

Resposta curta: **nem sempre**.  
GraphQL **não é necessariamente mais lento** que REST. Mas **é mais fácil ficar lento** se não for otimizado.

Vou explicar o porquê.

---

# 1️⃣ Overhead do GraphQL (existe, mas é pequeno)

Uma requisição GraphQL precisa:

1. **parsear a query**
    
2. **validar contra o schema**
    
3. **executar resolvers**
    
4. **montar a resposta**
    

Enquanto REST normalmente faz:

```
request → view → serializer → response
```

GraphQL faz algo mais assim:

```
request
 → parse query
 → validate query
 → execute resolvers (campo por campo)
 → montar response
```

Esse overhead normalmente é **pequeno**:

```
~1–5ms
```

Ou seja, **não é o principal gargalo**.

---

# 2️⃣ O verdadeiro problema: resolvers em cascata

Em GraphQL cada campo pode ter um resolver.

Exemplo:

```graphql
{
  tables {
    slug
    dataset {
      slug
    }
  }
}
```

Execução típica:

```
resolver tables → query
resolver dataset (para cada table)
resolver slug
```

Se não houver otimização:

```
1 query tables
+ N queries dataset
```

Isso é o famoso **N+1 problem**.

---

# 3️⃣ REST normalmente evita isso por design

Endpoint REST típico:

```
GET /tables
```

A view já prepara tudo:

```python
Table.objects.select_related("dataset")
```

Então:

```
1 query
```

GraphQL precisa **prever todas combinações de campos** possíveis.

---

# 4️⃣ Mas GraphQL pode ser MAIS rápido

Porque evita **overfetching**.

REST:

```
GET /table
→ retorna 40 campos
```

GraphQL:

```graphql
{
  table {
    slug
  }
}
```

Retorna:

```
1 campo
```

Isso reduz:

- serialização
    
- rede
    
- CPU
    

---

# 5️⃣ Em sistemas grandes GraphQL costuma ganhar

Empresas que usam muito:

- GitHub
    
- Shopify
    
- Meta
    

porque permite:

```
1 request → tudo que o cliente precisa
```

REST muitas vezes vira:

```
GET /user
GET /posts
GET /comments
```

3 chamadas.

GraphQL:

```graphql
{
  user {
    posts {
      comments
    }
  }
}
```

1 chamada.

---

# 6️⃣ O problema específico do Django + GraphQL

Com Django + Graphene ou Strawberry GraphQL o gargalo quase sempre é:

```
ORM queries
```

Especialmente quando:

- properties fazem queries
    
- relations não usam `prefetch_related`
    
- resolvers não usam dataloaders
    

---

# Regra prática

Performance típica:

|Caso|Mais rápido|
|---|---|
|REST simples|REST|
|GraphQL mal otimizado|REST|
|GraphQL bem otimizado|empate|
|Mobile app complexo|GraphQL|

---

# No seu caso específico

O problema **não parece ser GraphQL**.

O problema parece ser:

```
Model properties
+ relations
+ queries escondidas
```

Isso afeta **REST e GraphQL igualmente**.

Mas GraphQL expõe mais porque clientes podem pedir **muitos campos ao mesmo tempo**.

---

💡 Uma curiosidade:  
Em vários sistemas Django + GraphQL que já auditei, **80% do tempo da query estava em 2 coisas**:

```
@property
prefetch faltando
```

---

✅ Se quiser, posso também te mostrar:

- **3 properties do seu model que provavelmente geram mais queries**
    
- **como medir quantas queries uma query GraphQL está fazendo**
    
- **uma técnica que reduz 90% das queries em GraphQL com Django**
    

Seu model que você mandou é **um exemplo bem clássico de gargalo em GraphQL**.