
---
## 🎯 Objetivo

Sobrescrever o template `admin/base.html` do Jazzmin de forma segura, permitindo:

- Customizar sidebar
    
- Customizar navbar
    
- Adicionar categorias
    
- Adicionar botões
    
- Alterar layout
    

Sem modificar diretamente os arquivos do pacote instalado (evitando perder alterações em updates).

---

# 🧱 1. Entendendo como o Django encontra templates

O Django resolve templates na seguinte ordem:

1. Pastas declaradas em `TEMPLATES['DIRS']`
    
2. Templates dentro dos apps instalados
    
3. Pacotes de terceiros (como Jazzmin)
    

Isso significa que, se você criar:

```
templates/admin/base.html
```

Esse arquivo terá prioridade sobre o `base.html` do Jazzmin.

Esse processo se chama **Template Override**.

---

# 📂 2. Como encontrar o template original do Jazzmin

Para entender o que você está sobrescrevendo, é importante localizar o arquivo original.

## 🔎 Descobrindo onde está a virtualenv no Poetry

Dentro do projeto, execute:

```bash
poetry env info --path
```

Isso retorna algo como:

```
/home/usuario/.cache/pypoetry/virtualenvs/meuprojeto-abc123-py3.11
```

Esse é o caminho da sua virtualenv.

---

## 📁 Encontrando o Jazzmin dentro da env

Depois, navegue até:

```
<virtualenv>/lib/pythonX.X/site-packages/jazzmin/templates/admin/base.html
```

Exemplo:

```
/home/usuario/.cache/pypoetry/virtualenvs/meuprojeto-abc123-py3.11/lib/python3.11/site-packages/jazzmin/templates/admin/base.html
```

Você também pode localizar automaticamente com:

```bash
poetry run python -c "import jazzmin; print(jazzmin.__file__)"
```

Isso mostrará o caminho da pasta do pacote.

---

# 🛠 3. Criando o override corretamente

## 📁 Estrutura necessária no projeto

Certifique-se de ter:

```
project_root/
│
├── templates/
│   └── admin/
│       └── base.html
```

E no `settings.py`:

```python
TEMPLATES = [
    {
        "DIRS": [BASE_DIR / "templates"],
        ...
    }
]
```

---

## 🧾 Conteúdo básico do novo `base.html`

```django
{% extends "admin/base.html" %}
{% load jazzmin_tags %}

{% block sidebar %}
    {{ block.super }}
{% endblock %}
```

⚠️ Nunca copie o arquivo inteiro sem necessidade.  
Sempre use `{{ block.super }}` quando possível.

---

# 🧠 4. Estratégias de customização

## ✅ Estratégia segura (recomendada)

Use `{{ block.super }}` e apenas insira seu conteúdo antes ou depois.

Exemplo: adicionar categoria acima do Dashboard.

```django
{% extends "admin/base.html" %}
{% load jazzmin_tags %}

{% block sidebar %}
    {% if jazzmin_settings.show_sidebar %}
        {% get_side_menu as side_menu_list %}

        <aside class="main-sidebar elevation-4 {{ jazzmin_ui.sidebar_classes }}" id="jazzy-sidebar">

            <div class="sidebar">
                <nav class="mt-2">
                    <ul class="nav nav-pills nav-sidebar flex-column {{ jazzmin_ui.sidebar_list_classes }}">

                        <!-- Categoria custom -->
                        <li class="nav-header">Favoritos</li>
                        <li class="nav-item">
                            <a href="/minha-rota/" class="nav-link">
                                <i class="nav-icon fas fa-star"></i>
                                <p>Meu Link</p>
                            </a>
                        </li>

                        <!-- Mantém estrutura original -->
                        {{ block.super }}

                    </ul>
                </nav>
            </div>
        </aside>
    {% endif %}
{% endblock %}
```

---

## ⚠️ Estratégia arriscada (não recomendada)

Copiar todo o `base.html` do Jazzmin e editar diretamente.

Problema:

- Atualizações futuras do Jazzmin podem quebrar seu layout
    
- Você perde compatibilidade
    
- Difícil manutenção
    

---

# 🔍 5. Como inspecionar blocos disponíveis

Abra o `base.html` original do Jazzmin e procure por:

```django
{% block ... %}
```

Exemplos comuns:

- `sidebar`
    
- `navbar`
    
- `content`
    
- `footer`
    
- `bodyclass`
    

Esses são os pontos que você pode sobrescrever.

---

# 📌 6. Dicas importantes

- Sempre use `{% load jazzmin_tags %}` se for usar tags como:
    
    - `get_side_menu`
        
    - `jazzy_admin_url`
        
- Evite modificar arquivos dentro de `site-packages`
    
- Teste sempre limpando cache do navegador
    
- Se algo não aparecer, verifique se o template override está sendo carregado
    

Você pode testar colocando algo temporário:

```django
<h1>TESTE</h1>
```

Se aparecer, o override está funcionando.

---

# 🧩 7. Fluxo completo resumido

1. Descobrir caminho da env:
    
    ```
    poetry env info --path
    ```
    
2. Localizar template original:
    
    ```
    site-packages/jazzmin/templates/admin/base.html
    ```
    
3. Criar override em:
    
    ```
    templates/admin/base.html
    ```
    
4. Usar:
    
    ```
    {% extends "admin/base.html" %}
    ```
    
5. Sobrescrever apenas os blocos necessários.
    

---

# 🏁 Conclusão

O processo correto para customizar o Jazzmin é:

- Nunca editar `site-packages`
    
- Sempre usar template override
    
- Sempre reutilizar `block.super` quando possível
    
- Sempre verificar os blocos disponíveis no template original
    

---