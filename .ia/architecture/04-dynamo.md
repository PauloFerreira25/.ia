---
name: dynamo
description: "Read before writing or modifying any DynamoDB code. Start by reading the first 20 lines — the index tells you which sections are relevant to your task."
---

# DynamoDB — Decisões e Padrões

---

## Index

| Section | When to consult |
|---|---|
| [Nomenclatura de GSI](#nomenclatura-de-gsi) | When creating or naming a Global Secondary Index |
| [Criação de GSI](#criação-de-gsi) | Before adding a new GSI |
| [Language](#language) | Always — naming identifiers in English |

---

## Nomenclatura de GSI

Usar sempre nomes descritivos baseados no campo indexado, no padrão `{campo}-index`.

```
personId-index   ✅
email-index      ✅
GSI1             ❌
GSI2             ❌
```

O nome deve deixar claro o campo e a finalidade do índice sem precisar abrir o código.

---

## Criação de GSI

Só criar GSI quando houver código que o utilize. Não criar por antecipação.

O DynamoDB permite adicionar GSI em tabelas existentes a qualquer momento, sem downtime e sem perda de dados — não há custo em esperar.

---

## Language

**Best practice: all identifiers in English.** Table names, attribute names, GSI names, and any other code-level identifiers must be written in English. The only exception is when the human explicitly and deliberately requests otherwise.
