---
name: aws-lambda-nodejs
description: "Read before writing or modifying any AWS Lambda + Node.js code. Start by reading the first 25 lines — the index tells you which sections are relevant to your task."
---

# AWS Lambda + Node.js Architecture

> **DESCONTINUADO.** Este documento cobre padrões do modelo anterior (camada primeiro). Para novos projetos, use [09-lambda-domain-architecture.md](./09-lambda-domain-architecture.md), que define o modelo atual (domínio primeiro).
>
> Alterações em projetos existentes devem, no mínimo, migrar a estrutura de diretórios das Lambdas para o modelo do documento 09:
>
> ```
> lambdas/http/v1/products/get/       ← @<project>/lambda-http-v1-products-get
> lambdas/http/v1/products/post/      ← @<project>/lambda-http-v1-products-post
> lambdas/http/v1/products/_id/get/   ← @<project>/lambda-http-v1-products-id-get
> lambdas/http/v1/products/_id/delete/ ← @<project>/lambda-http-v1-products-id-delete
> ```
>
> O conteúdo abaixo continua válido como referência para projetos já existentes no modelo antigo.

---

> This document is specific to this project's Lambda implementation patterns. It covers conventions, rules, and pitfalls that apply to every Lambda function in this repository.
>
> For general Node.js + TypeScript rules, see [02-nodejs-typescript.md](./02-nodejs-typescript.md). This document only covers what is specific to Lambda.

---

## Index

| Section | When to consult |
|---|---|
| [Error handling](#error-handling) | When writing or modifying the Lambda handler wrapper |
| [Path Parameters](#path-parameters) | When defining or naming API Gateway path parameters in CDK routes or handlers |
| [npm packages](#npm-packages) | When installing dependencies |
| [ESLint](#eslint) | When resolving ESLint errors |
| [Language](#language) | Always — naming identifiers |

---

## Error handling

In Lambda there is no framework error handler — the handler function is the boundary. Any error that is not caught and logged before returning a response is lost forever. CloudWatch will only show the response envelope, not the original error.

Every Lambda handler must wrap its execution in a `try/catch` and delegate error handling to a shared `handleError` utility. Never write the error-to-response logic inline in the handler.

```typescript
export const handler = async (event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> => {
  log.debug({ ... }, 'handler')
  try {
    // ...
    return formatResponse(event, 200, { data: result })
  } catch (error) {
    return handleError(event, error)
  }
}
```

`handleError` is defined in the project's shared utilities package and is responsible for:
- Logging every unhandled error with `log.error({ error }, 'unhandledError')` before returning the response
- Mapping `AppError` subclasses to their HTTP status codes and codes
- Returning a generic 500 envelope for everything else — never exposing the original error message or stack trace

Each project specializes `handleError` in its shared utils package. The exact implementation is project-specific, but the contract is always the same: the caller passes `(event, error)` and receives a formatted API Gateway response.

Rules:
- Always use `handleError` in the catch block — never write the error mapping inline.
- Never return a 500 response without going through `handleError`. A 500 with no log is undebuggable.
- Never expose the original error message or stack trace in the response body.

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
