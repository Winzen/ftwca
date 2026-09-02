

---

## 📌 Contexto

Ao editar o `base.html` (ou qualquer template Django), após salvar o arquivo, o VS Code alterava automaticamente a estrutura da linha, quebrando atributos HTML que continham tags Django.

Exemplo original:

```html
<a class="dropdown-item" href="{{ child.url }}" {% if link.new_window %}target="_blank" {% endif %}>{{ child.name }}</a>
```

Após salvar, o VS Code transformava em:

```html
<a class="dropdown-item" href="{{ child.url }}" {% if link.new_window %}target="_blank"
    {% endif %}>{{ child.name }}</a>
```

Isso causava erro no Django:

```
Invalid block tag on line XX: 'endfor', expected 'elif', 'else' or 'endif'
```

---

## 🎯 Causa do problema

O VS Code estava formatando automaticamente o arquivo ao salvar.

Mesmo que o código estivesse correto, o formatter de HTML:

- Quebrava atributos longos
    
- Reorganizava linhas
    
- Alterava a estrutura visual dos blocos `{% %}`
    

Como o Django Template Engine é sensível à estrutura dos blocos (`{% if %}`, `{% for %}`, `{% endif %}`, `{% endfor %}`), pequenas alterações automáticas podiam quebrar a hierarquia dos blocos.

O problema não era o Django, nem o Jazzmin — era o **auto format on save** do VS Code.

---

## ✅ Solução

Desativar a formatação automática ao salvar.

### Passo a passo

1. Abrir o VS Code
    
2. Pressionar:
    

```
Ctrl + Shift + P
```

3. Digitar:
    

```
Open User Settings (JSON)
```

4. Adicionar:
    

```json
"editor.formatOnSave": false,
```

5. Salvar.
    

---

## 🧠 Conclusão

Templates Django não são HTML puro.

Formatadores automáticos (principalmente ao salvar) podem:

- Quebrar atributos que contêm `{% if %}`
    
- Separar blocos `{% %}`
    
- Gerar erros difíceis de identificar
    

Para projetos com:

- Django Admin
    
- Jazzmin
    
- Templates complexos
    
- Muitos blocos condicionais
    

É recomendado manter:

```json
"editor.formatOnSave": false
```

---

## 🏷️ Tags sugeridas para Obsidian

```
#django
#vscode
#debug
#templates
#jazzmin
#problemas-reais
```

---
