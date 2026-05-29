---
name: aws-lambda-nodejs
description: "Read before writing or modifying any AWS Lambda + Node.js code. Start by reading the first 25 lines — the index tells you which sections are relevant to your task."
---

# AWS Lambda + Node.js Architecture

> This document is specific to this project's Lambda implementation patterns. It covers conventions, rules, and pitfalls that apply to every Lambda function in this repository.
>
> For general Node.js + TypeScript rules, see [02-nodejs-typescript.md](./02-nodejs-typescript.md). This document only covers what is specific to Lambda.

---

## Index

| Section | When to consult |
|---|---|
| [Path Parameters](#path-parameters) | When defining or naming API Gateway path parameters in CDK routes or handlers |
| [npm packages](#npm-packages) | When installing dependencies |
| [ESLint](#eslint) | When resolving ESLint errors |
| [Language](#language) | Always — naming identifiers |

---

## Path Parameters

Name path parameters after what they represent, never after their type.

```
/accounts/{accountId}     ✅
/events/{eventId}         ✅
/accounts/{id}            ❌ — id of what?
```

This matters because API Gateway injects path parameters using the exact name defined in the route. If the CDK route defines `{id}` but the handler calls `requirePathParam(event, 'accountId')`, the value is always `undefined`.

Nested routes make the value of this rule concrete:

```
/accounts/{accountId}/enrollments/{enrollmentId}   ✅ — unambiguous
/accounts/{id}/enrollments/{id}                    ❌ — which id is which?
```

The name used in `requirePathParam` must match exactly what is defined in the CDK route.

```typescript
// CDK route: /accounts/{accountId}
const accountId = requirePathParam(event, 'accountId')  // ✅

// CDK route: /accounts/{id}
const accountId = requirePathParam(event, 'accountId')  // ❌ — returns undefined
const id = requirePathParam(event, 'id')                // ✅ — but {id} is wrong to begin with
```

---

## npm packages

Follow the rules defined in [02-nodejs-typescript.md → npm packages](./02-nodejs-typescript.md#npm-packages).

---

## ESLint

Follow the rules defined in [02-nodejs-typescript.md → ESLint](./02-nodejs-typescript.md#eslint), including the no eslint-disable rule.

---

## Language

Follow the rules defined in [02-nodejs-typescript.md → Language](./02-nodejs-typescript.md#language).
