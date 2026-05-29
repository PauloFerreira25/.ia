---
name: dynamo
description: "Read before writing or modifying any DynamoDB code. Start by reading the first 20 lines — the index tells you which sections are relevant to your task."
---

# DynamoDB — Decisions and Patterns

---

## Index

| Section | When to consult |
|---|---|
| [GSI Naming](#gsi-naming) | When creating or naming a Global Secondary Index |
| [GSI Creation](#gsi-creation) | Before adding a new GSI |
| [Language](#language) | Always — naming identifiers in English |

---

## GSI Naming

Always use descriptive names based on the indexed field, following the pattern `{field}-index`.

```
personId-index   ✅
email-index      ✅
GSI1             ❌
GSI2             ❌
```

The name must make the field and the purpose of the index clear without having to open the code.

---

## GSI Creation

Only create a GSI when there is code that uses it. Do not create in anticipation.

DynamoDB allows adding GSIs to existing tables at any time, with no downtime and no data loss — there is no cost in waiting.

---

## Language

**Best practice: all identifiers in English.** Table names, attribute names, GSI names, and any other code-level identifiers must be written in English. The only exception is when the human explicitly and deliberately requests otherwise.
