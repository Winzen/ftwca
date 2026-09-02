
---

# 🧩 Problema: `Uncaught ReferenceError: abrirModal is not defined`

## 📌 Contexto

Ao usar JavaScript com:

```html
<script type="module" src="..."></script>
```

E chamar uma função no HTML:

```html
<button onclick="abrirModal()">Abrir</button>
```

O navegador retorna:

```
Uncaught ReferenceError: abrirModal is not defined
```

---

## 🔎 Causa do Problema

Quando usamos:

```html
<script type="module">
```

O JavaScript passa a funcionar como **ES Module**.

Isso muda o comportamento padrão:

- O arquivo passa a ter **escopo isolado**
    
- Funções e variáveis **não ficam disponíveis no objeto `window`**
    
- Nada é automaticamente global
    

Ou seja, isso:

```js
function abrirModal() {
  modal.style.display = 'flex';
}
```

❌ **não cria** `window.abrirModal`

Logo, o HTML não consegue encontrar a função.

---

# ✅ Solução 1 — Expor Manualmente no `window`

Adicionar ao final do arquivo JS:

```js
window.abrirModal = abrirModal;
window.fecharModal = fecharModal;
```

Exemplo completo:

```js
function abrirModal() {
  modal.style.display = 'flex';
}

function fecharModal() {
  modal.style.display = 'none';
}

window.abrirModal = abrirModal;
window.fecharModal = fecharModal;
```

Agora funciona com:

```html
<button onclick="abrirModal()">Abrir</button>
```

✔️ Simples  
✔️ Rápido  
✔️ Compatível com código existente

---

# ✅ Solução 2 — Forma Moderna (Recomendada)

Remover `onclick` do HTML:

```html
<button id="btnAbrirModal">Abrir</button>
```

E usar `addEventListener` no JS:

```js
document
  .getElementById('btnAbrirModal')
  .addEventListener('click', abrirModal);
```

Vantagens:

- Não mistura JS com HTML
    
- Mantém encapsulamento do módulo
    
- Código mais organizado
    
- Melhor manutenção
    
- Padrão moderno
    

---

# 🧠 Conceito Importante

Ao usar `type="module"`:

|JavaScript normal|JavaScript Module|
|---|---|
|Escopo global|Escopo isolado|
|Funções ficam no `window`|Não ficam no `window`|
|Pode usar `onclick` direto|Precisa expor ou usar listener|
|Não suporta `import/export`|Suporta `import/export`|

---

# 🎯 Conclusão

O erro não é na função.

O erro é consequência do uso de **ES Modules**, que:

- Ativam escopo isolado
    
- Impedem poluição global
    
- Tornam o código mais seguro e moderno
    

A solução depende da arquitetura desejada:

- 🔧 Projeto simples → usar `window.funcao = funcao`
    
- 🚀 Projeto organizado → usar `addEventListener`
    

---

Se quiser, posso montar também uma anotação explicando:

- Diferença entre Script clássico vs Module
    
- Como o navegador carrega módulos
    
- Como funciona o escopo global (`window`)