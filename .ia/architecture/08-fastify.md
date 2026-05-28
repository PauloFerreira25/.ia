---
name: fastify
description: "Read before writing or modifying any Fastify backend code. Start by reading the first 50 lines — the index tells you which sections are relevant to your task."
---

# Fastify Architecture

> Based on Fastify v5 (latest: 5.x). API examples follow the v5 conventions — do not use v4 patterns.

> Read [02-nodejs-typescript.md](./02-nodejs-typescript.md) first — this document only covers what differs from or is specific to Fastify backends. All rules defined there (ESM, tsconfig, scripts, logging, naming, error classes, testing philosophy, type safety, identifiers, pagination) apply here without repetition.

---

## Index

| Category | Section | When to consult |
|---|---|---|
| **Domain construction** | [Layer Rules](#layer-rules) | When creating or modifying handlers, services, repositories |
| | [Dependency Injection](#dependency-injection) | When wiring layers with the `buildX` pattern |
| | [Validation](#validation) | When defining route schemas |
| | [DTOs](#dtos) | When defining request/response schemas |
| | [Route registration](#route-registration) | When adding routes to a domain |
| | [Authentication](#authentication) | When securing routes |
| **Standards and best practices** | [Logging](#logging) | When adding log statements to any layer |
| | [Error Handling](#error-handling) | When handling errors |
| | [Extending FastifyRequest](#extending-fastifyrequest) | When adding per-request data (user, traceId, etc.) |
| | [Testing](#testing) | When writing tests |
| **Fastify configuration** | [File structure](#file-structure) | Project file structure reference |
| | [Server setup](#server-setup) | Application bootstrap |
| | [Configuration](#configuration) | Environment variables |
| | [Security headers](#security-headers) | Helmet |
| | [CORS](#cors) | Cross-origin setup |
| | [Health check](#health-check) | Liveness / readiness |
| | [Swagger](#swagger) | API documentation |
| | [Compression](#compression) | Response compression |
| | [Rate limiting](#rate-limiting) | Request throttling |
| | [UUID](#uuid) | ID generation |
| | [Project-specific (30+) checklist](#project-specific-30-checklist) | What each project must document |

---

## Layer Rules

**Handler** — owns everything HTTP:
- Request is already validated by the route schema — no manual validation in the handler
- Maps request params to plain service input — no business logic, no conditions
- Calls the service
- Returns the service result directly — serialization is handled automatically by the route response schema

Because validation lives in the route schema and handlers are small by design, all handlers for a domain live in a single `<domain>.handlers.ts` file. They follow the same `buildX` pattern as service and repository.

```typescript
// src/domain/user/user.handlers.ts
import type { RouteHandler } from 'fastify'

import { userService }                           from './user.service.js'
import type { UserService }                      from './user.service.js'
import type { CreateUserBody, IdParams }         from './user.dto.js'

type Deps = { userService: UserService }

const create = ({ userService }: Deps): RouteHandler<{ Body: CreateUserBody }> =>
  async (request) => {
    return userService.create(request.body)
  }

const findById = ({ userService }: Deps): RouteHandler<{ Params: IdParams }> =>
  async (request) => {
    return userService.findById(request.params)
  }

export function buildUserHandlers(deps: Deps = { userService }) {
  return {
    create:   create(deps),
    findById: findById(deps),
  }
}

export type UserHandlers = ReturnType<typeof buildUserHandlers>
export const userHandlers = buildUserHandlers()
```

**Service** — owns all business logic. Returns what the operation produced — a Model, a composed type, or a primitive. Never returns a DTO and never knows about HTTP. May call multiple repositories or other services.

**Repository** — data access only. Returns `Model` types. No business logic.

---

## Dependency Injection

Use functional composition with default values. Each function is a standalone `const`. The `buildX` function assembles them and exports a default instance.

**Repository** imports its datasource directly and exports a default instance:

```typescript
// src/domain/user/user.repository.ts
import { db } from '@src/infra/database.js'

type Deps = { db: typeof db }

const save = ({ db }: Deps) =>
  async (params: CreateUserParams): Promise<UserModel> => {
    log.debug({ params }, 'save')
    // ...
  }

const findById = ({ db }: Deps) =>
  async (params: IdParams): Promise<UserModel | null> => {
    log.debug({ params }, 'findById')
    // ...
  }

export function buildUserRepository(deps: Deps = { db }) {
  return {
    save:     save(deps),
    findById: findById(deps),
  }
}

export type UserRepository = ReturnType<typeof buildUserRepository>
export const userRepository = buildUserRepository()
```

**Service** imports the default repository instance and exports its own default:

```typescript
// src/domain/user/user.service.ts
import { userRepository } from './user.repository.js'

type Deps = { userRepository: UserRepository }

const create = ({ userRepository }: Deps) =>
  async (params: CreateUserParams): Promise<UserModel> => {
    log.debug({ params }, 'create')
    if (await userRepository.existsByEmail({ email: params.email }))
      throw new ConflictError('USER_EMAIL_TAKEN', `Email ${params.email} already in use`)
    return userRepository.save(params)
  }

const findById = ({ userRepository }: Deps) =>
  async (params: IdParams): Promise<UserModel | null> => {
    log.debug({ params }, 'findById')
    return userRepository.findById(params)
  }

export function buildUserService(deps: Deps = { userRepository }) {
  return {
    create:   create(deps),
    findById: findById(deps),
  }
}

export type UserService = ReturnType<typeof buildUserService>
export const userService = buildUserService()
```

**Test overrides** — inject mocks only in lower layers (service, repository), never in the handler:
```typescript
const testRepo    = buildUserRepository({ db: testDb })
const testService = buildUserService({ userRepository: testRepo })
```

Handler tests always use `buildApp()` with real infrastructure via `app.inject()`. Never mock the service at the handler level — if you need to isolate business logic, test the service directly.

**Multi-datasource override** — inject the right database:
```typescript
const analyticsRepo = buildUserRepository({ db: analyticsDb })
```

---

## Validation

Use `fastify-type-provider-zod` to integrate Zod into Fastify's schema system. It validates request body, params, query, and headers automatically from the route schema — no manual validation in handlers.

```
npm install fastify-type-provider-zod
```

Registered once in `buildApp()`:

```typescript
app.setValidatorCompiler(validatorCompiler)
app.setSerializerCompiler(serializerCompiler)
```

Define schemas directly on the route:

```typescript
import { type ZodTypeProvider } from 'fastify-type-provider-zod'

app.withTypeProvider<ZodTypeProvider>().route({
  method:  'POST',
  url:     '/users',
  schema: {
    body:     CreateUserBodySchema,
    response: { 200: CreateUserResponseSchema },
  },
  handler: createUserHandler,
})
```

---

## DTOs

Follow the DTO pattern defined in [02-nodejs-typescript.md](./02-nodejs-typescript.md) — base schema in `<domain>.schema.ts`, per-endpoint DTOs in `<domain>.dto.ts` derived via `omit`, `extend`, or `pick`.

In Fastify, DTOs are plain Zod schemas — no class wrappers needed.

```typescript
// src/domain/user/user.dto.ts
import { UserBaseSchema } from './user.schema.js'

export const CreateUserBodySchema     = UserBaseSchema.omit({ role: true })
export const CreateUserResponseSchema = UserBaseSchema.extend({ id: z.string() })

export type CreateUserBody     = z.infer<typeof CreateUserBodySchema>
export type CreateUserResponse = z.infer<typeof CreateUserResponseSchema>
```

---

## Route registration

Routes live in `<domain>/<domain>.router.ts` and are registered as Fastify plugins. Each domain exports a plugin that registers its routes.

```typescript
// src/domain/user/user.router.ts
import type { FastifyPluginAsync } from 'fastify'
import type { ZodTypeProvider } from 'fastify-type-provider-zod'

import { userHandlers } from './user.handlers.js'
import { CreateUserBodySchema, CreateUserResponseSchema, FindUserByIdResponseSchema } from './user.dto.js'

export const userRoutes: FastifyPluginAsync = async (app) => {
  const typed = app.withTypeProvider<ZodTypeProvider>()

  typed.route({
    method:  'POST',
    url:     '/users',
    schema: { body: CreateUserBodySchema, response: { 200: CreateUserResponseSchema } },
    handler: userHandlers.create,
  })

  typed.route({
    method:  'GET',
    url:     '/users/:id',
    schema: { params: IdParamsSchema, response: { 200: FindUserByIdResponseSchema } },
    handler: userHandlers.findById,
  })
}
```

```typescript
// src/infra/routes.ts
import type { FastifyPluginAsync } from 'fastify'

import { userRoutes }  from '@src/domain/user/user.router.js'
import { orderRoutes } from '@src/domain/order/order.router.js'

export const routes: FastifyPluginAsync = async (app) => {
  await app.register(userRoutes)
  await app.register(orderRoutes)
}
```

---

## Authentication

Do not use Passport. Authentication and authorization are declared via `onRequest` on the route — never via a global hook with `config.public`.

Always use array syntax for `onRequest`, even with a single hook.

Each project must document its `authHook` and `requireRole` implementations in a file numbered `30+`.

```typescript
// src/domain/user/user.router.ts
export const userRoutes: FastifyPluginAsync = async (app) => {
  const typed = app.withTypeProvider<ZodTypeProvider>()

  typed.route({
    method:    'POST',
    url:       '/users',
    schema:    { body: CreateUserBodySchema, response: { 200: CreateUserResponseSchema } },
    handler:   userHandlers.create,
    // no onRequest — public route
  })

  typed.route({
    method:    'GET',
    url:       '/users/:id',
    onRequest: [authHook],
    schema:    { params: IdParamsSchema, response: { 200: FindUserByIdResponseSchema } },
    handler:   userHandlers.findById,
  })

  typed.route({
    method:    'DELETE',
    url:       '/users/:id',
    onRequest: [authHook, requireRole('admin')],
    schema:    { params: IdParamsSchema, response: { 200: DeleteUserResponseSchema } },
    handler:   userHandlers.delete,
  })
}
```

When all routes in a router share the same auth, register the hook at the plugin level instead of repeating it per route:

```typescript
export const userRoutes: FastifyPluginAsync = async (app) => {
  app.addHook('onRequest', authHook)

  // all routes below are authenticated
}
```

---

## Logging

Use `AsyncLocalStorage` to propagate `request.log` through all layers automatically. This ensures every log entry within a request carries the `requestId` — without passing the logger as a parameter.

```typescript
// src/shared/logger.ts
import { AsyncLocalStorage } from 'node:async_hooks'
import type { FastifyBaseLogger } from 'fastify'
import pino from 'pino'

const storage = new AsyncLocalStorage<FastifyBaseLogger>()

export const globalLogger = pino({ /* project-specific config — documented in 30+ */ })

export const log = new Proxy(globalLogger, {
  get: (_, prop) => {
    const store = storage.getStore()
    return (store ?? globalLogger)[prop as keyof FastifyBaseLogger]
  }
})

export function runWithLogger(logger: FastifyBaseLogger, fn: () => void): void {
  storage.run(logger, fn)
}
```

Import `log` directly in any layer — no logger parameter needed:

```typescript
import { log } from '@src/shared/logger.js'

const create = ({ userRepository }: Deps) =>
  async (params: CreateUserParams): Promise<UserModel> => {
    log.debug({ params }, 'create')  // requestId included automatically inside a request
    return userRepository.save(params)
  }
```

- Inside a request → `log` uses `request.log` (scoped, includes `requestId`)
- Outside a request (bootstrap, tests, background jobs) → `log` uses `globalLogger`

---

## Error Handling

`errorHandler` maps `AppError` subclasses to the HTTP envelope. Register in `buildApp()` via `app.setErrorHandler(errorHandler)`.

```typescript
// src/shared/errors/errorHandler.ts
import type { FastifyError, FastifyReply, FastifyRequest } from 'fastify'

import { AppError } from '@pauloferreira25/commons-errors'
import { log } from '@src/shared/logger.js'

export function errorHandler(
  error:   FastifyError | Error,
  request: FastifyRequest,
  reply:   FastifyReply,
): void {
  if (error instanceof AppError) {
    void reply.status(error.statusCode).send({ error: error.code, message: error.message })
    return
  }

  if ('validation' in error && error.validation) {
    void reply.status(400).send({ error: 'VALIDATION_ERROR', message: error.message })
    return
  }

  log.error({ error }, 'unhandled error')
  void reply.status(500).send({ error: 'INTERNAL_ERROR', message: 'Internal server error' })
}
```

Three cases covered:
- `AppError` subclass → status code + error code from the error
- Zod/Fastify validation error → 400 `VALIDATION_ERROR`
- Anything else → 500 `INTERNAL_ERROR`, details never exposed to the client

---

## Extending FastifyRequest

`FastifyRequest` is the single extension point for any per-request data — authenticated user, trace ID, tenant ID, etc.

Always follow three steps:

**1. Declare the decorator in `buildApp()`** — required for V8 object shape optimization:
```typescript
app.decorateRequest('user', null)
app.decorateRequest('traceId', '')
```

**2. Populate in a hook:**
```typescript
app.addHook('onRequest', async (request) => {
  request.traceId = request.headers['x-trace-id'] as string ?? generateUUIDv7()
})
```

**3. Extend the type** in `src/shared/fastify.d.ts`:
```typescript
declare module 'fastify' {
  interface FastifyRequest {
    user:    AuthenticatedUser | null
    traceId: string
  }
}
```

The `AuthenticatedUser` type and auth-specific decorators are documented in the project's `30+` file.

---

## Testing

Follow the philosophy in [02-nodejs-typescript.md](./02-nodejs-typescript.md) — real infrastructure, highest level possible, 100% coverage.

In Fastify, the highest level is the handler. Build the full app with `buildApp()` and use `app.inject()` for requests — no real HTTP port needed.

`bootstrap` is production orchestration — in tests, initialize infrastructure explicitly in `beforeAll` before calling `buildApp()`. The project's `30+` file documents which initializations are needed.

```typescript
// test/domain/user/user.post.create.handler.test.ts
import { buildApp } from '@src/infra/api.js'
// import infrastructure initializers from 30+ (e.g. initDatabase)

let app: Awaited<ReturnType<typeof buildApp>>

beforeAll(async () => {
  // initialize infrastructure in the same order as bootstrap — documented in 30+
  await initDatabase()
  app = await buildApp()
  await app.ready()
})

afterAll(async () => {
  await app.close()
  // teardown infrastructure — documented in 30+
  await closeDatabase()
})

test('POST /users creates a user', async () => {
  const response = await app.inject({
    method:  'POST',
    url:     '/users',
    payload: { name: 'Paulo', email: 'paulo@example.com' },
  })

  expect(response.statusCode).toBe(200)
  expect(response.json()).toMatchObject({ name: 'Paulo' })
})
```

Test file structure mirrors `src/`:

```
test/
└── domain/
    └── <domain>/
        ├── <domain>.post.create.handler.test.ts    ← always exists
        ├── <domain>.service.test.ts                ← only if handler cannot cover it
        └── <domain>.repository.test.ts             ← only if handler cannot cover it
```

---

## File structure

```
src/
├── index.ts                              ← bootstrap, gracefulShutdown, signal handlers
├── infra/
│   ├── api.ts                            ← buildApp: Fastify config, plugins, routes, error handler
│   └── routes.ts                         ← registers all domain routers
└── shared/
    ├── config.ts                         ← env var validation (Zod)
    ├── fastify.d.ts                      ← FastifyRequest augmentations (user, traceId, etc.)
    ├── logger.ts                         ← globalLogger, log proxy, runWithLogger
    ├── uuid.ts                           ← generateUUIDv7
    └── errors/
        └── errorHandler.ts              ← AppError → HTTP + validation errors + 500 fallback
```

Domain structure per domain:
```
src/domain/<domain>/
├── <domain>.router.ts                   ← Fastify plugin, route definitions, onRequest hooks
├── <domain>.handlers.ts                 ← HTTP layer: maps request → service → response
├── <domain>.service.ts                  ← business logic
├── <domain>.repository.ts               ← data access
├── <domain>.schema.ts                   ← base Zod schema
└── <domain>.dto.ts                      ← per-endpoint schemas and types
```

---

## Server setup

`src/infra/api.ts` is responsible only for Fastify configuration — plugins, hooks, routes, error handler. `src/index.ts` owns the startup sequence (`bootstrap`) and signal handling (`gracefulShutdown`). Project-specific implementations are documented in a file numbered `30+`.

```typescript
// src/infra/api.ts — Fastify configuration only
import Fastify from 'fastify'
import cors from '@fastify/cors'
import { serializerCompiler, validatorCompiler } from 'fastify-type-provider-zod'

import { config } from '@src/shared/config.js'
import { errorHandler } from '@src/shared/errors/errorHandler.js'
import { globalLogger, runWithLogger } from '@src/shared/logger.js'
import { generateUUIDv7 } from '@src/shared/uuid.js'
import { routes } from './routes.js'

export async function buildApp(): Promise<ReturnType<typeof Fastify>> {
  const app = Fastify({ logger: globalLogger, genReqId: () => generateUUIDv7() })

  app.setValidatorCompiler(validatorCompiler)
  app.setSerializerCompiler(serializerCompiler)

  await app.register(import('@fastify/helmet'))
  await app.register(import('@fastify/compress'))
  await app.register(import('@fastify/rate-limit'), { max: 100, timeWindow: '1 minute' })
  await app.register(cors, { origin: config.CORS_ORIGIN })

  await app.register(import('@fastify/swagger'), { openapi: { info: { title: config.APP_NAME, version: config.APP_VERSION } } })
  await app.register(import('@fastify/swagger-ui'), { routePrefix: config.SWAGGER_PREFIX })

  app.addHook('onRequest', (request, _reply, done) => {
    runWithLogger(request.log, done)
  })

  app.setErrorHandler(errorHandler)

  await app.register(routes)

  return app
}

// src/index.ts — initialization order and signal handling
import { buildApp } from './infra/api.js'
import { config } from './shared/config.js'

async function bootstrap(): Promise<ReturnType<typeof buildApp>> {
  // 1. logger, database, and other dependencies — documented in 30+
  const app = await buildApp()
  await app.listen({ port: config.PORT, host: '0.0.0.0' })
  return app
}

async function gracefulShutdown(app: Awaited<ReturnType<typeof buildApp>>): Promise<void> {
  // close database connections and other dependencies — documented in 30+
  await app.close()
  process.exit(0)
}

const app = await bootstrap()

process.on('SIGTERM', () => { void gracefulShutdown(app) })
process.on('SIGINT',  () => { void gracefulShutdown(app) })
```

---

## Configuration

Base environment variables every Fastify project must declare in `src/shared/config.ts`:

```typescript
// src/shared/config.ts
import { z } from 'zod'

export const config = z.object({
  PORT:           z.coerce.number().default(3000),
  CORS_ORIGIN:    z.string().url(),
  APP_NAME:       z.string(),
  APP_VERSION:    z.string().default('1.0.0'),
  SWAGGER_PREFIX: z.string().default('/docs'),
}).parse(process.env)
```

Project-specific variables (database URL, auth secrets, external service keys, etc.) are documented in `30+`.

---

## Security headers

Use `@fastify/helmet`. Register before all other plugins.

```
npm install @fastify/helmet
```

---

## CORS

Use `@fastify/cors`. Never use the `cors` npm package — it is Express-based.

```
npm install @fastify/cors
```

CORS policy (allowed origins, credentials, headers) is defined by the project in a file numbered `30+`.

---

## Health check

Two endpoints — register both in `src/infra/routes.ts` before domain routes:

- `GET /health` — liveness: always returns `200 { status: 'ok' }` if the process is running
- `GET /health/ready` — readiness: checks dependencies (database, external services); documented in `30+`

```typescript
// src/infra/routes.ts
export const routes: FastifyPluginAsync = async (app) => {
  app.get('/health', async () => ({ status: 'ok' }))
  // GET /health/ready — implemented in 30+

  await app.register(userRoutes)
  // ...
}
```

---

## Swagger

Use `@fastify/swagger` + `@fastify/swagger-ui`. With `fastify-type-provider-zod`, Swagger is generated automatically from the route schemas — no extra annotations needed.

```
npm install @fastify/swagger @fastify/swagger-ui
```

Swagger UI is available at the path defined by `config.SWAGGER_PREFIX` (default: `/docs`).

---

## Compression

Use `@fastify/compress`. Register after helmet and rate-limit, before routes.

```
npm install @fastify/compress
```

---

## Rate limiting

Use `@fastify/rate-limit`. The base default is `100 requests / 1 minute` per IP. Override per route or globally in `30+` when the project requires different limits.

```
npm install @fastify/rate-limit
```

```typescript
// base default — applied globally
await app.register(import('@fastify/rate-limit'), { max: 100, timeWindow: '1 minute' })

// per-route override (in router)
typed.route({
  config: { rateLimit: { max: 10, timeWindow: '1 minute' } },
  // ...
})
```

---

## UUID

Use `uuidv7` for all ID generation, including `requestId`:

```
npm install uuidv7
```

```typescript
// src/shared/uuid.ts
import { uuidv7 } from 'uuidv7'

export const generateUUIDv7 = (): string => uuidv7()
```

---

## Language

**Best practice: all identifiers in English.** This rule is inherited from [02-nodejs-typescript.md](./02-nodejs-typescript.md) and applies here without exception: function names, variable names, route paths, handler names, type names, file names, error codes, and log messages must be written in English. The only exception is when the human explicitly and deliberately requests otherwise.

---

## Project-specific (`30+`) checklist

Every Fastify project must document the following in a `30+` file:

- CORS policy — allowed origins, credentials, headers
- Authentication strategy — token type, validation logic, `authHook` and `requireRole` implementations
- `AuthenticatedUser` type and `decorateRequest` declarations
- Rate limit overrides — global or per-route limits that differ from the base default (100 req/min)
- `GET /health/ready` — readiness check: verify database connection and other critical dependencies
- `bootstrap` — initialization order: logger, database, other dependencies, then `buildApp()` + `listen`
- `gracefulShutdown` — teardown order: stop accepting requests (`app.close()`), close database, other cleanup
- Project-specific environment variables — extend `config.ts` with database URLs, auth secrets, external service keys, etc.
