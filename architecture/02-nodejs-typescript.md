---
name: nodejs-typescript
description: "Read before writing or modifying any Node.js + TypeScript code."
---

# Node.js + TypeScript Architecture

## Filosofia

Estas decisões guiam todas as escolhas de arquitetura no projeto:

**Produção primeiro.** Toda decisão é tomada pensando em produção. Conforto de desenvolvimento é secundário.

**Consistência elimina decisão.** Quando o custo de decidir caso a caso é maior do que aplicar uniformemente, a regra vira absoluta. Logs em toda função é um exemplo: decidir onde logar e onde não logar custa mais do que logar em tudo.

**Explícito sobre implícito.** Tipos declarados, interfaces nomeadas, config validada no startup. Nada depende de inferência silenciosa ou convenção não escrita.

**Real sobre mockado.** Testes usam infraestrutura real. Mock só quando impossível subir o serviço.

**Nível mais alto possível.** Testa no ponto mais próximo da fronteira do sistema. Um teste de handler cobre handler, service e repository ao mesmo tempo.

**Reutilizar antes de criar.** Antes de escrever qualquer coisa, verificar se já existe no projeto ou em dependência instalada.

**Separação de responsabilidade.** Cada camada tem uma responsabilidade e não ultrapassa sua fronteira. O handler adapta HTTP para domínio. O service não sabe que existe HTTP. O repository não tem lógica de negócio.

---

## ES Modules

Always use ES Modules. Never use CommonJS.

- Set `"type": "module"` in `package.json`
- Use `import`/`export` syntax in all files
- Use `.js` extension in all local imports — required by NodeNext module resolution
- Never use `require()`, `module.exports`, or `exports`

```typescript
import { findById } from './tag.service.js'
import type { Tag } from './tag.type.js'
```

---

## tsconfig.json

Required compiler options for all projects:

```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true
  }
}
```

For `shared-libs` packages that are published to npm, also add:

```json
{
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true
  }
}
```

Never use `"moduleResolution": "node"` — it does not support ESM correctly.

---

## Path aliases

Use `@src` as an alias for the `src/` directory. This eliminates deep relative imports.

```typescript
// wrong
import { config } from '../../../../shared/config.js'

// correct
import { config } from '@src/shared/config.js'
```

### tsconfig.json

Add `baseUrl` and `paths` to `compilerOptions`:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@src/*": ["src/*"]
    }
  }
}
```

### Runtime resolution

TypeScript compiles path aliases but does not rewrite them in the emitted JavaScript. Use `tsc-alias` to rewrite `@src/*` to relative paths after compilation:

```
npm install -D tsc-alias
```

Update the `build` script in `package.json`:

```json
{
  "scripts": {
    "build": "tsc && tsc-alias"
  }
}
```

The `dev` script requires no change — `tsx` resolves path aliases from `tsconfig.json` natively.

### vitest.config.ts

Add `resolve.alias` so Vitest resolves the alias during tests:

```typescript
import { defineConfig } from 'vitest/config'
import { resolve } from 'node:path'

export default defineConfig({
  resolve: {
    alias: {
      '@src': resolve(import.meta.dirname, 'src'),
    },
  },
  test: {
    include: ['test/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      include: ['src/**/*.ts'],
    },
  },
})
```

### ESLint

Add `pathGroups` to `import/order` so the alias is treated as internal:

```javascript
'import/order': ['error', {
  'groups': ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
  'pathGroups': [{ pattern: '@src/**', group: 'internal' }],
  'newlines-between': 'always',
}],
```

---

## package.json scripts

Required scripts in every project:

```json
{
  "type": "module",
  "scripts": {
    "dev":        "concurrently \"tsx watch src/index.ts\" \"tsc --noEmit --watch\" \"esw --watch src\"",
    "build":      "tsc",
    "start":      "node dist/index.js",
    "lint":       "eslint src",
    "lint:fix":   "eslint src --fix",
    "test":       "vitest run",
    "test:watch": "vitest",
    "ci":         "npm run lint && tsc --noEmit && npm run test && npm run build"
  }
}
```

- `dev` runs three processes in parallel: app with auto-reload, TypeScript type checker, and ESLint watcher
- `build` compiles to native JavaScript for production
- `start` runs the compiled output — never runs TypeScript in production
- `ci` runs the full pipeline in order — lint, typecheck, tests, build. Build only runs if everything before it passes
- `lint` runs ESLint once — use in CI
- `lint:fix` runs ESLint and auto-fixes what it can

Install `concurrently` and `eslint-watch` as dev dependencies:

```
npm install -D concurrently eslint-watch
```

`eslint-watch` provides the `eslint --watch` command that ESLint does not include natively.

---

## ESLint

Every TypeScript project must have ESLint configured. Use flat config (v9+) with `typescript-eslint`:

```javascript
// eslint.config.js
import tseslint from 'typescript-eslint'
import importPlugin from 'eslint-plugin-import'

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    plugins: {
      import: importPlugin,
    },
    rules: {
      '@typescript-eslint/consistent-type-imports':          ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-import-type-side-effects':      'error',
      '@typescript-eslint/explicit-function-return-type':    ['error', { allowExpressions: true }],
      '@typescript-eslint/explicit-module-boundary-types':   'error',
      'no-unused-vars':                                       'off',
      '@typescript-eslint/no-unused-vars': ['error', {
        vars: 'all',
        args: 'all',
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      'import/no-duplicates':                                'error',
      'import/order': ['error', {
        'groups': ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
      }],
    },
  },
  { ignores: ['dist/', 'test/'] }
)
```

Install required plugins:

```
npm install -D typescript-eslint eslint-plugin-import
```

Key rules enforced:

- `consistent-type-imports` — forces `import type` for type-only imports, keeping runtime imports explicit
- `explicit-function-return-type` — all functions must declare their return type
- `explicit-module-boundary-types` — all exported functions must have typed parameters and return type
- `no-unused-vars` — variáveis e parâmetros não utilizados são erro. Prefixar com `_` para indicar intencionalmente ignorado (ex: `_req`)
- `import/order` — imports grouped and ordered: Node built-ins → external packages → internal modules

Never use the legacy `.eslintrc` format.

---

## Testing

Use Vitest as the test runner. It understands TypeScript and ESM natively.

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    include: ['test/**/*.test.ts'],
    coverage: {
      provider: 'v8',
      include: ['src/**/*.ts'],
    },
  },
})
```

**Filosofia de testes:**

- Use real infrastructure — never mock databases, queues, or external services unless it is impossible to run them
- Test at the highest level possible — a handler test covers handler, service, and repository together
- `service.test.ts` and `repository.test.ts` only exist to cover what the handler test cannot reach
- Each test file creates its own isolated infrastructure in `beforeAll` and destroys it in `afterAll`

```typescript
// test/helper/database.ts
export async function createTestDatabase() {
  const name = `test_${generateUUIDv7()}`
  const db = await client.createDatabase(name)
  return { db, cleanup: () => client.dropDatabase(name) }
}
```

**Test directory structure:**

`test/` mirrors `src/` — same folder names, same file names with `.test.ts` suffix appended:

```
test/
├── domain/
│   ├── tag/
│   │   ├── tag.handler.test.ts     ← always exists
│   │   ├── tag.service.test.ts     ← only if handler cannot cover it
│   │   └── tag.repository.test.ts  ← only if handler cannot cover it
│   └── <domain>/
│       └── <domain>.handler.test.ts
├── queue/
│   └── <group>/
│       └── <worker>.handler.test.ts
└── helper/
    └── database.ts   ← shared test utilities, not test files
```

Never place test files inside `src/`. Never use a flat `test/` directory — always mirror the source structure.

---

## Monorepo and package resolution

Monorepos are allowed. Never use the `workspaces` field in `package.json`.

Resolve local packages via `file:` references:

```json
{
  "dependencies": {
    "@pauloferreira25/commons-errors": "file:../../shared-libs/commons-errors"
  }
}
```

Each package remains independent with its own `node_modules`.

---

## shared-libs

Shared libraries live in `shared-libs/`. Each one is an independent npm-publishable package.

```
shared-libs/
└── commons-errors/
    ├── src/
    │   └── index.ts
    ├── package.json   ← name, version, exports, types
    └── tsconfig.json  ← declaration: true
```

Requirements for every shared-lib:

- Must have a proper `name` (scoped), `version`, and `exports` field in `package.json`
- Must compile to `dist/` with `declaration: true` — consumers depend on the compiled output
- Must be referenceable via `file:` in development and publishable to npm without changes

Never create shared logic (error classes, utilities, types) directly in a project. Extract to a shared-lib.

### Scoping convention

The npm scope signals the audience of the library:

**`@pauloferreira25/*`** — generic, reusable across any project. No business logic, no domain knowledge.

```
@pauloferreira25/commons-types   ← IdParams, PaginationParams, PaginationResponse
@pauloferreira25/commons-errors  ← base error classes, error envelope
```

**`@<project-scope>/*`** — project-specific or business domain libs. Scoped to the project that owns them. Each project defines its own scope.

---

## npm packages

When installing a dependency:

- Only use packages that are open source and maintained by a company or large community
- Always install the latest version — run `npm install <package>` without specifying a version
- Check whether an installed package already solves the problem before adding a new one

---

## Reuse before writing

Before implementing anything, verify it does not already exist:

- Search the codebase for existing utilities, types, and functions
- Check installed packages before adding a new dependency
- Never duplicate logic that exists elsewhere in the project

---

## Pagination

Prefer cursor-based pagination. Use offset/page only when there is a specific justification (fixed datasets, admin reports, export features).

Cursor-based is the default because it handles large volumes correctly and does not break when data changes between pages.

Standard response envelope — `PaginationResponse<T>` comes from `@pauloferreira25/commons-types`:

```typescript
import type { PaginationResponse } from '@pauloferreira25/commons-types'

// PaginationResponse<T> shape:
// {
//   data: T[]
//   pagination: {
//     pageSize:   number
//     nextCursor: string | null
//     prevCursor: string | null
//     hasNext:    boolean
//     hasPrev:    boolean
//   }
// }
```

Standard input — `PaginationParams` also from `@pauloferreira25/commons-types`:

```typescript
import type { PaginationParams } from '@pauloferreira25/commons-types'

// PaginationParams shape:
// {
//   cursor?:   string
//   pageSize?: number
// }
```

Never return a raw array from a list endpoint. Always wrap in the standard envelope.

---

## Identifiers

Use UUIDv7 for all entity identifiers. Never use auto-increment integers, UUIDv4, or any other format.

UUIDv7 is time-ordered, universally unique, and works across distributed systems without coordination.

---

## Configuration

Validate all environment variables with Zod at startup in a single file: `src/shared/config.ts`.

```typescript
// src/shared/config.ts
import { z } from 'zod'

export const config = z.object({
  DATABASE_URL: z.string().url(),
  PORT:         z.coerce.number().default(3000),
  LOG_LEVEL:    z.enum(['debug', 'info', 'warn', 'error']).default('info'),
}).parse(process.env)
```

Never access `process.env` outside of `src/shared/config.ts`. The rest of the codebase imports `config`.

The service fails fast on startup if any required variable is missing or invalid.

---

## Error handling

### Who throws

- `repository` — never throws domain errors. Returns `null` when an entity is not found
- `service` — checks the result and throws when the business rule is violated
- `handler` — never has `try/catch`. Errors propagate to the Fastify error middleware

```typescript
// repository — returns null, never throws NotFoundError
async function findById(params: IdParams): Promise<Tag | null> {
  return db.get(params.id) ?? null
}

// service — understands the domain, throws when needed
async function findById(params: IdParams): Promise<Tag> {
  const tag = await tagRepository.findById(params)
  if (!tag) throw new NotFoundError('TAG_NOT_FOUND', `Tag ${params.id} not found`)
  return tag
}

// handler — clean, no try/catch, no null checks
async function getTag(params: IdParams): Promise<Tag> {
  return tagService.findById(params)
}
```

### AppError base class

`@pauloferreira25/commons-errors` exports `AppError` as the base class for all errors. The Fastify error middleware handles any `AppError` subclass automatically — no registration required.

```typescript
// @pauloferreira25/commons-errors
export class AppError extends Error {
  constructor(
    public readonly statusCode: number,
    public readonly code:       string,
    message:                    string
  ) { super(message) }
}

export class NotFoundError extends AppError {
  constructor(code: string, message: string) { super(404, code, message) }
}

export class ValidationError extends AppError {
  constructor(code: string, message: string) { super(422, code, message) }
}

export class ConflictError extends AppError {
  constructor(code: string, message: string) { super(409, code, message) }
}
```

Projects define precise business errors by extending `AppError` — no changes to commons needed:

```typescript
// project-specific error
import { AppError } from '@pauloferreira25/commons-errors'

export class DataIntegrationError extends AppError {
  constructor(message: string) {
    super(422, 'DATA_INTEGRATION_ERROR', message)
  }
}
```

### HTTP response envelope

```json
{ "error": "TAG_NOT_FOUND", "message": "Tag abc-123 not found" }
```

- `error` — `UPPER_SNAKE_CASE` code, stable and programmatically comparable
- `message` — human-readable text, may change
- Unhandled errors (not `AppError` subclasses) return `500` with a generic message — never expose stack traces

---

## Folder structure (DDD)

Organize source code under `src/` using domain-driven design:

```
src/
├── index.ts         ← entry point, composes modules — no business logic
│
├── infra/           ← connections to external systems and bootstrap
│   ├── api.ts       ← HTTP server bootstrap (optional)
│   ├── queue.ts     ← message queue consumer bootstrap (optional)
│   ├── database.ts  ← database client (optional)
│   └── rabbitmq.ts  ← message queue client (optional)
│
├── domain/
│   └── <domain>/    ← one folder per bounded context
│
├── queue/           ← optional — present only when the project has workers
│   └── <worker>/    ← one folder per worker, grouped by business responsibility
│
└── shared/          ← domain-agnostic utilities with no external dependencies
    ├── config.ts    ← env var validation (Zod)
    ├── logger.ts    ← pino instance
    └── ...          ← pure helpers, no business logic, no infrastructure clients
```

`index.ts` only composes and starts. It never contains business logic.

`infra/` holds everything that connects to an external system. Files in `shared/` must be domain-agnostic and infrastructure-agnostic — if a file in `shared/` imports a database client or knows a domain rule, it is in the wrong place.

---

## Layer separation

Each domain follows a fixed file structure:

```
domain/<name>/
├── index.ts              ← exports routes for infra/api.ts
├── <name>.route.ts       ← route definitions and schema references
├── <name>.handler.ts     ← receives request, calls service, returns response
├── <name>.service.ts     ← business logic, orchestration, event publishing
├── <name>.repository.ts  ← database queries only — no business logic
├── <name>.schema.ts      ← input/output validation schemas
└── <name>.type.ts        ← TypeScript interfaces and types
```

Layer rules:

- `handler` translates the HTTP request into plain domain types before calling the service
- `handler` never queries the database directly — always goes through service
- `service` never imports types from the HTTP framework
- `repository` never emits events and never contains business logic
- `schema` never imports from `service` or `repository`

Workers follow the same pattern — replace `route` and `schema` with `consumer`:

```
queue/<group>/<worker>/
├── index.ts
├── <worker>.consumer.ts  ← subscribes to queue, calls handler
├── <worker>.handler.ts   ← processes the message
├── <worker>.service.ts   ← business logic
└── <worker>.type.ts      ← message types
```

---

## Function signatures

All functions receive a single object parameter. No positional parameters, no exceptions.

This ensures callers never break when the function evolves — adding a new field to the input type does not require updating call sites.

```typescript
// correct
findById(params: IdParams): Promise<Tag>
find(params: FindTagParams): Promise<Tag[]>
update(params: UpdateTagParams): Promise<Tag>
delete(params: IdParams): Promise<void>

// wrong
findById(id: string): Promise<Tag>
update(id: string, name: string, description: string): Promise<Tag>
```

`IdParams` comes from the shared-lib `@pauloferreira25/commons-types` — never inline `{ id: string }`:

```typescript
import type { IdParams } from '@pauloferreira25/commons-types'

findById(params: IdParams): Promise<Tag>
```

Always declare explicit types on function parameters. Always declare return types on exported functions.

Input types must be explicit interfaces or type aliases — never inline object types in signatures:

```typescript
// correct
interface FindTagParams {
  pagination?: { cursor?: string; pageSize?: number }
  filter?:     { isActive?: boolean }
}

// wrong
find(params: { pagination?: { cursor?: string }; filter?: { isActive?: boolean } }): Promise<Tag[]>
```

---

## Logging

Use `pino` as the logger in all projects. In projects with Fastify, use the built-in logger (`fastify({ logger: true })`). In projects without Fastify, instantiate directly:

```typescript
// src/shared/logger.ts
import pino from 'pino'
import { config } from './config.js'

export const log = pino({ level: config.LOG_LEVEL })
```

First line of every function must be a debug log with the function name as the second argument. No exceptions.

```typescript
async function findById(params: IdParams): Promise<Tag> {
  log.debug({ params }, 'findById')
  // ...
}

async function find(params: FindTagParams): Promise<Tag[]> {
  log.debug({ params }, 'find')
  // ...
}
```

The second argument is always the exact function name — this makes `grep` on logs reliable.

When parameters contain sensitive data (passwords, tokens, personal data), log an explicit object omitting the sensitive fields. Never suppress the log — only omit the fields:

```typescript
async function createUser(params: CreateUserParams): Promise<User> {
  log.debug({ email: params.email }, 'createUser') // omits password
  // ...
}

async function authenticate(params: AuthParams): Promise<Session> {
  log.debug({ username: params.username }, 'authenticate') // omits password
  // ...
}
```

Never log the full `params` object when it contains sensitive fields.

---

## Naming conventions

**Code identifiers:**

- `camelCase` — variables, functions, parameters, object properties
- `PascalCase` — interfaces, type aliases, classes, enums
- `UPPER_SNAKE_CASE` — environment variable names and error codes

```typescript
// camelCase
const tagId = params.id
async function findById(params: IdParams): Promise<Tag> { ... }

// PascalCase
interface FindTagParams { ... }
type TagStatus = 'draft' | 'valid' | 'invalid'
enum ValidationState { Draft, Valid, Invalid }

// UPPER_SNAKE_CASE
const DATABASE_URL = config.DATABASE_URL
throw new NotFoundError('TAG_NOT_FOUND', ...)
```

Never use abbreviations. Names must be descriptive and self-explanatory (`user`, not `usr`; `request`, not `req`; `manager`, not `mgr`).

**Files and directories:**

Use `camelCase` for all file and directory names, without exception:

```
src/
├── domain/
│   └── orderItem/
│       ├── orderItem.route.ts
│       ├── orderItem.handler.ts
│       ├── orderItem.service.ts
│       ├── orderItem.repository.ts
│       ├── orderItem.schema.ts
│       └── orderItem.type.ts
└── shared/
    ├── config.ts
    └── logger.ts
```

Exception: tooling config files in the project root follow each tool's own convention and are never renamed (`tsconfig.json`, `eslint.config.js`, `package.json`, `vitest.config.ts`).

Never use `kebab-case`, `snake_case`, or `PascalCase` for file or directory names.

---

## Language

All code, identifiers, field names, log messages, error codes, queue names, and event types must be in English. No exceptions.

```typescript
// correct
throw new NotFoundError('TAG_NOT_FOUND', `Tag ${id} not found`)
log.info({ tagId }, 'tagNotFound')

// wrong
throw new NotFoundError('TAG_NAO_ENCONTRADA', `Tag ${id} não encontrada`)
```
