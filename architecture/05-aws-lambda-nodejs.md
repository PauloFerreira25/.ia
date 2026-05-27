---
name: aws-lambda-nodejs
description: "Read before writing or modifying any AWS Lambda + Node.js code."
---

# AWS Lambda + Node.js Architecture

> This document is specific to this project's Lambda implementation patterns. It covers conventions, rules, and pitfalls that apply to every Lambda function in this repository.

## npm packages

When installing a dependency:

- Always install the latest version — run `npm install <package>` without specifying a version
- Never manually edit `package.json` to add a dependency with a pinned version — always let npm resolve and record the version after installation
- Specifying a version (`npm install <package>@x.y.z`) is only allowed when a human explicitly requests a specific version, or when another installed dependency already constrains that version as a peer dependency

Pinning a specific version installs an outdated package instead of the current latest, accumulating security vulnerabilities over time. Every AI agent that hard-codes a version into `package.json` without running `npm install` introduces this risk.

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
