---
name: nestjs-fastify
description: "Read before writing or modifying any NestJS + Fastify code."
---

# NestJS + Fastify Architecture

> ⚠️ **Deprecated — do not use.** NestJS does not support ES Modules, which is a hard requirement of this project (defined in `02-nodejs-typescript.md`). Use [08-fastify.md](./08-fastify.md) instead.

---

## Overrides from 02

The following rules from `02-nodejs-typescript.md` do not apply to NestJS projects. Each override is documented in its own section below.

| Rule in 02 | Overridden by |
|---|---|
| ES Modules (`"type": "module"`, `.js` extensions) | CommonJS — NestJS CLI does not support ESM reliably |
| `package.json` scripts (`tsx`, `tsc`, `concurrently`) | NestJS CLI scripts (`nest build`, `nest start`) |
| Handler file splitting convention | Not applicable — NestJS uses controller methods |

---

## ES Modules

NestJS projects use **CommonJS**. Do not add `"type": "module"` to `package.json` and do not use `.js` extensions in local imports. The NestJS CLI handles compilation and does not support ESM reliably.

```typescript
// correct in NestJS
import { UserService } from './user.service'

// wrong in NestJS
import { UserService } from './user.service.js'
```

---

## package.json scripts

NestJS CLI replaces the `tsx`/`tsc`/`concurrently` scripts from `02-nodejs-typescript.md`:

```json
{
  "scripts": {
    "dev":        "nest start --watch",
    "build":      "nest build",
    "start":      "node dist/main",
    "lint":       "eslint src",
    "lint:fix":   "eslint src --fix",
    "test":       "vitest run",
    "test:watch": "vitest",
    "coverage":   "vitest run --coverage",
    "ci":         "npm run lint && tsc --noEmit && npm run test && npm run build"
  }
}
```

Install the NestJS CLI as a dev dependency:

```
npm install -D @nestjs/cli
```

---

## tsconfig.json

Add to the base config from `02-nodejs-typescript.md`. NestJS decorators require two extra options — without `emitDecoratorMetadata`, dependency injection silently fails at runtime.

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

---

## Fastify Adapter

Always use the Fastify adapter. Never use the default Express adapter. Disable NestJS's built-in logger — Fastify's pino handles everything.

`main.ts` is structured around two explicit functions that every project must implement:

- `bootstrap` — runs before `listen`. Load dependencies, warm caches, validate external services.
- `gracefulShutdown` — runs on `SIGTERM`/`SIGINT`. Flush queues, close connections, release resources. Always ends with `app.close()`.

Project-specific implementations of both functions are documented in a file numbered `30+`.

Below is a complete example `main.ts`. Each section is detailed further in its own chapter — this is the assembly reference.

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core'
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify'
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger'
import { ZodValidationPipe } from 'nestjs-zod'
import { patchNestJsSwagger } from 'nestjs-zod'

import { AppModule } from './app.module.js'
import { AppErrorFilter } from './shared/filters/appError.filter.js'
import { CatchAllFilter } from './shared/filters/catchAll.filter.js'
import { config } from './shared/config.js'

// ← see: Bootstrap / Graceful Shutdown
async function bootstrap(app: NestFastifyApplication): Promise<void> {
  // project-specific pre-listen setup — documented in 30+
}

async function gracefulShutdown(app: NestFastifyApplication): Promise<void> {
  // project-specific teardown — documented in 30+
  await app.close()
  process.exit(0)
}

async function main(): Promise<void> {
  // ← see: Fastify Adapter
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }),
    { logger: false },
  )

  // ← see: Error Handling
  app.useGlobalFilters(new CatchAllFilter(), new AppErrorFilter())

  // ← see: Validation
  app.useGlobalPipes(new ZodValidationPipe())

  // ← see: CORS
  await app.register(import('@fastify/cors'), { origin: config.CORS_ORIGIN })

  // ← see: Swagger
  patchNestJsSwagger()
  const document = SwaggerModule.createDocument(app, new DocumentBuilder().setTitle('API').setVersion('1.0').build())
  SwaggerModule.setup('docs', app, document)

  await bootstrap(app)
  await app.listen(config.PORT, '0.0.0.0')

  process.on('SIGTERM', () => { void gracefulShutdown(app) })
  process.on('SIGINT',  () => { void gracefulShutdown(app) })
}

main()
```

---

## CORS

Use `@fastify/cors`. Never use `app.enableCors()` or the `cors` npm package — both are Express-based and do not work with the Fastify adapter.

```
npm install @fastify/cors
```

CORS policy (allowed origins, credentials, headers) is defined by the project in a file numbered `30+`.

---

## Validation

Never use `class-validator` or `class-transformer` — they are the NestJS default but not used here.

Use `nestjs-zod`'s `ZodValidationPipe` globally. It automatically validates any request body, param, or query that uses a DTO class created with `createZodDto`. Register it in `main.ts`:

```typescript
import { ZodValidationPipe } from 'nestjs-zod'

app.useGlobalPipes(new ZodValidationPipe())
```

No per-param pipe application needed — the global pipe handles all DTOs.

---

## DTOs

Follow the DTO pattern defined in [02-nodejs-typescript.md](./02-nodejs-typescript.md) — base schema in `<domain>.schema.ts`, per-endpoint DTOs in `<domain>.dto.ts` derived via `omit`, `extend`, or `pick`.

In NestJS, DTOs must be classes for Swagger to introspect them. Use `createZodDto` from `nestjs-zod` to create a class from each Zod schema. The class is used as the NestJS type; the Zod schema remains the source of truth.

```
npm install nestjs-zod
```

```typescript
// src/domain/user/user.dto.ts
import { createZodDto } from 'nestjs-zod'

import { UserBaseSchema } from './user.schema.js'

const CreateUserBodySchema     = UserBaseSchema.omit({ role: true })
const CreateUserResponseSchema = UserBaseSchema.extend({ id: z.string() })
const UpdateUserBodySchema     = UserBaseSchema.pick({ name: true })

export class CreateUserBodyDto     extends createZodDto(CreateUserBodySchema) {}
export class CreateUserResponseDto extends createZodDto(CreateUserResponseSchema) {}
export class UpdateUserBodyDto     extends createZodDto(UpdateUserBodySchema) {}
```

Use the class as the type in the controller — no separate `z.infer<>` needed:

```typescript
@Post()
create(@Body() body: CreateUserBodyDto): Promise<CreateUserResponseDto> {
  return this.userService.create({ body })
}
```

---

## Swagger

Use `@nestjs/swagger` with `nestjs-zod`. Call `patchNestJsSwagger()` before creating the Swagger document — it teaches `@nestjs/swagger` to read Zod schemas from `createZodDto` classes. Without this call, all DTOs appear as empty objects in the docs.

```
npm install @nestjs/swagger
```

```typescript
// src/main.ts
import { patchNestJsSwagger } from 'nestjs-zod'
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger'

patchNestJsSwagger()

// after app setup:
const document = SwaggerModule.createDocument(app, new DocumentBuilder().setTitle('API').setVersion('1.0').build())
SwaggerModule.setup('docs', app, document)
```

Swagger UI will be available at `/docs`. The path and title are project-specific — document them in a file numbered `30+`.

---

## Folder Structure

NestJS replaces `infra/api.ts` with `main.ts`. Each domain gets a `<domain>.module.ts`. The `infra/` folder still holds external system connections (database clients, queue clients).

```
src/
├── main.ts                   ← bootstrap (replaces infra/api.ts)
├── app.module.ts             ← root module
│
├── infra/
│   ├── database/
│   │   ├── database.module.ts
│   │   └── database.service.ts
│   └── ...
│
├── domain/
│   └── <domain>/
│       ├── <domain>.module.ts
│       ├── <domain>.controller.ts
│       ├── <domain>.service.ts
│       ├── <domain>.repository.ts  ← always present when the project has a database
│       ├── <domain>.schema.ts
│       ├── <domain>.dto.ts
│       └── <domain>.model.ts
│
└── shared/
    ├── config.ts
    ├── decorators/
    │   └── public.decorator.ts
    └── filters/
        ├── appError.filter.ts
        └── catchAll.filter.ts
```

---

## AppModule

`AppModule` is the root — it imports infrastructure modules and a single `DomainsModule` barrel. Never import domain modules directly into `AppModule`.

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common'
import { ConfigModule } from '@nestjs/config'

import { configSchema } from './shared/config.js'
import { DomainsModule } from './domain/domains.module.js'

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, validate: (env) => configSchema.parse(env) }),
    DomainsModule,
  ],
})
export class AppModule {}
```

`DomainsModule` is the barrel — it imports all domain modules:

```typescript
// src/domain/domains.module.ts
import { Module } from '@nestjs/common'

import { UserModule }  from './user/user.module.js'
import { OrderModule } from './order/order.module.js'

@Module({
  imports: [UserModule, OrderModule],
})
export class DomainsModule {}
```

Each domain module imports only the database module it needs. This allows projects with multiple databases — each domain declares its own dependency without a global lock:

```typescript
// src/domain/user/user.module.ts
import { Module } from '@nestjs/common'

import { DatabaseModule }  from '@src/infra/database/database.module.js'
import { UserController }  from './user.controller.js'
import { UserService }     from './user.service.js'
import { UserRepository }  from './user.repository.js'

@Module({
  imports:     [DatabaseModule],
  controllers: [UserController],
  providers:   [UserService, UserRepository],
  exports:     [UserService],
})
export class UserModule {}
```

---

## Module Structure

Every domain has a `<domain>.module.ts`. The rules below apply to all modules — the example uses `user` as illustration.

- `controllers` — the domain controller
- `providers` — service and repository
- `exports` — always export the service, so other modules can inject it
- `imports` — import another domain module only when the service needs to inject that module's service

```typescript
// src/domain/user/user.module.ts
import { Module } from '@nestjs/common'

import { UserController }  from './user.controller.js'
import { UserService }     from './user.service.js'
import { UserRepository }  from './user.repository.js'

@Module({
  controllers: [UserController],
  providers:   [UserService, UserRepository],
  exports:     [UserService],
})
export class UserModule {}
```

When `OrderService` needs to call `UserService`, import `UserModule` in `OrderModule`:

```typescript
// src/domain/order/order.module.ts
import { Module } from '@nestjs/common'

import { UserModule } from '@src/domain/user/user.module.js'
import { OrderController }  from './order.controller.js'
import { OrderService }     from './order.service.js'
import { OrderRepository }  from './order.repository.js'

@Module({
  imports:     [UserModule],
  controllers: [OrderController],
  providers:   [OrderService, OrderRepository],
  exports:     [OrderService],
})
export class OrderModule {}
```

Infrastructure modules (`DatabaseModule`, `ConfigModule`) are registered as global in `AppModule` — domain modules never import them directly.

---

## Layer Rules

**Controller** — owns everything HTTP:
- Defines routes
- Applies guards (authentication and authorization)
- Validates request via `ZodValidationPipe` (global)
- Maps request params to plain service input — no business logic, no `if`
- Calls the service
- Always serializes the response via `Schema.parse()` before returning
- Declares Swagger metadata

The controller never knows what the service does internally. If any condition or domain decision appears in the controller, it belongs in the service.

```typescript
@Post()
@ApiOperation({ summary: 'Create user' })
async create(@Body() body: CreateUserBodyDto): Promise<CreateUserResponseDto> {
  const user = await this.userService.create({ name: body.name, email: body.email })
  return CreateUserResponseSchema.parse(user)
}
```

**Service** — owns all business logic. Returns whatever the business operation produces — a Model, a composed type, a primitive. Never returns a DTO and never knows about HTTP.

A service may inject and call multiple repositories or other services. Cross-domain orchestration belongs in the service layer.

```typescript
// service returns what the business produced — not a DTO
async create(params: CreateUserParams): Promise<UserModel> {
  log.debug({ params }, 'create')
  if (await this.userRepository.existsByEmail({ email: params.email }))
    throw new ConflictError('USER_EMAIL_TAKEN', `Email ${params.email} already in use`)
  return this.userRepository.save(params)
}

// when the result is not a Model, the service defines its own output type
interface AgesSumResult { totalAge: number; userIds: string[] }

async sumAges(params: SumAgesParams): Promise<AgesSumResult> {
  log.debug({ params }, 'sumAges')
  const [user1, user2] = await Promise.all([
    this.userRepository.findById({ id: params.userId1 }),
    this.userRepository.findById({ id: params.userId2 }),
  ])
  return { totalAge: user1.age + user2.age, userIds: [user1.id, user2.id] }
}
```

**Repository** — data access only. Returns `Model` types. No business logic, no calls to other repositories or services.

---

## Interceptors vs Fastify Hooks

Always prefer NestJS interceptors for cross-cutting concerns. Fastify hooks bypass the NestJS lifecycle and should only be used when the same result cannot be achieved with an interceptor.

```typescript
// correct — NestJS interceptor
@Injectable()
export class RequestIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<FastifyRequest>()
    request.headers['x-request-id'] ??= randomUUID()
    return next.handle()
  }
}
```

```typescript
// avoid — Fastify hook. Only justified when an interceptor cannot solve it.
app.getHttpAdapter().getInstance().addHook('onRequest', (request, _reply, done) => {
  request.headers['x-request-id'] ??= randomUUID()
  done()
})
```

---

## Configuration

Use `@nestjs/config` with Zod validation. Register as global so all modules have access via `ConfigService`.

```typescript
// src/app.module.ts
import { ConfigModule } from '@nestjs/config'

import { configSchema } from './shared/config.js'

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validate: (env) => configSchema.parse(env),
    }),
  ],
})
export class AppModule {}
```

```typescript
// src/shared/config.ts
import { z } from 'zod'

export const configSchema = z.object({
  PORT:      z.coerce.number().default(3000),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
})

export type Config = z.infer<typeof configSchema>
export const config = configSchema.parse(process.env)
```

`config` is used in `main.ts` before the NestJS container initializes. `ConfigService` is used for DI within modules.

---

## Authentication

Do not use Passport or `@fastify/passport` — `@fastify/passport` is beta with low adoption and known incompatibilities.

### Guards

All access control is implemented as NestJS guards (`CanActivate`). Guards follow an onion model: global guards run first, then class-level, then method-level. Within each level, guards execute left-to-right. If any guard returns `false` or throws, the chain stops.

```
Request
  → auth guard (global)          ← validates token; respects @Public()
    → permission guard (class)   ← checks role or scope
      → ownership guard (method) ← checks if resource belongs to the requester
        → handler
```

Every project must have a `@Public()` decorator to mark open routes. The auth guard checks for it before validating the token:

```typescript
// src/shared/decorators/public.decorator.ts
export const IS_PUBLIC_KEY = 'isPublic'
export const Public = (): MethodDecorator => SetMetadata(IS_PUBLIC_KEY, true)
```

```typescript
// example of an auth guard that respects @Public()
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ])
    if (isPublic) return true
    // token validation logic — defined in 30+
  }
}
```

```typescript
// example of guard composition in a controller
@UseGuards(PermissionGuard)
@Controller('users')
export class UserController {

  @Public()
  @Post('register')
  register() {}             // no token required

  @Get('profile')
  profile() {}              // auth guard only

  @Roles('admin')
  @Get()
  listAll() {}              // auth guard + PermissionGuard

  @UseGuards(OwnerGuard)
  @Delete(':id')
  delete() {}               // auth guard + PermissionGuard + OwnerGuard
}
```

Guard names (`AuthGuard`, `PermissionGuard`, `OwnerGuard`, `@Roles`) are illustrative. Each project defines its own guards and decorators in a file numbered `30+`, including token validation logic, role sources, and ownership rules.

---

## Repository

Always create a repository layer whenever the project has a database. The repository is where custom queries, log patterns, and query object definitions live. Services never import the database client directly.

The example below uses Prisma, but the pattern applies to any database client.

```typescript
// src/domain/user/user.repository.ts
import { Injectable } from '@nestjs/common'

import { log } from '@src/shared/logger.js'
import { PrismaService } from '@src/infra/database/prisma.service.js'
import type { IdParams } from '@src/shared/commons/handler.types.js'
import type { User } from './user.model.js'

@Injectable()
export class UserRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(params: IdParams): Promise<User | null> {
    log.debug({ params }, 'findById')
    return this.prisma.user.findUnique({ where: { id: params.id } })
  }
}
```

---

## Response Serialization

The controller always serializes the response via `Schema.parse()` before returning. This ensures no unexpected or sensitive fields leak from the service output. The service never knows about DTOs — it returns what the business produced.

```typescript
// controller — always Schema.parse() on the way out
@Post()
async create(@Body() body: CreateUserBodyDto): Promise<CreateUserResponseDto> {
  const user = await this.userService.create({ name: body.name, email: body.email })
  return CreateUserResponseSchema.parse(user)
}

// service — returns what the business produced, never a DTO
async create(params: CreateUserParams): Promise<UserModel> {
  log.debug({ params }, 'create')
  return this.userRepository.save(params)
}
```

When the service result is not a Model (composed type, aggregate), the controller still does `Schema.parse()` — the pattern is always the same regardless of what the service returns.

---

## Error Handling

Two global filters handle all errors. Register both in `main.ts` — order matters: `CatchAllFilter` first, `AppErrorFilter` second. NestJS applies filters in reverse registration order, so `AppErrorFilter` runs first for `AppError` subclasses and `CatchAllFilter` handles everything else.

```typescript
// src/main.ts
app.useGlobalFilters(new CatchAllFilter(), new AppErrorFilter())
```

`AppErrorFilter` catches `AppError` subclasses and maps to the HTTP envelope:

```typescript
// src/shared/filters/appError.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common'
import type { FastifyReply } from 'fastify'

import { AppError } from '@pauloferreira25/commons-errors'

@Catch(AppError)
export class AppErrorFilter implements ExceptionFilter {
  catch(exception: AppError, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse<FastifyReply>()
    response.status(exception.statusCode).send({
      error:   exception.code,
      message: exception.message,
    })
  }
}
```

`CatchAllFilter` catches everything else and returns `500` — never exposes stack traces or internal details:

```typescript
// src/shared/filters/catchAll.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common'
import type { FastifyReply } from 'fastify'

import { log } from '@src/shared/logger.js'

@Catch()
export class CatchAllFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    log.error({ exception }, 'unhandled exception')
    const response = host.switchToHttp().getResponse<FastifyReply>()
    response.status(500).send({
      error:   'INTERNAL_ERROR',
      message: 'Internal server error',
    })
  }
}
```

NestJS's built-in `HttpException` is never used — all errors extend `AppError`.

---

## ESLint — No eslint-disable

Never use `eslint-disable`, `eslint-disable-next-line`, `eslint-disable-line`, or any variant of ESLint suppression comments in `.ts` files. If an ESLint error appears, fix the code. If an existing `eslint-disable` is found, remove it and fix the underlying issue. If the issue cannot be resolved without suppressing the rule, stop and notify a human.

---

## Language

**Best practice: all identifiers in English.** Class names, method names, variable names, file names, module names, decorator names, and any other code-level identifiers must be written in English. The only exception is when the human explicitly and deliberately requests otherwise.

---

## Testing

Follow the philosophy in [02-nodejs-typescript.md](./02-nodejs-typescript.md) — real infrastructure, highest level possible, 100% coverage.

In NestJS, the highest level is the controller. Set up the full application with `Test.createTestingModule` and the Fastify adapter. Call `.ready()` after `app.init()` — Fastify requires it before the instance can handle requests.

```typescript
// test/domain/user/user.controller.test.ts
import { Test } from '@nestjs/testing'
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify'

import { AppModule } from '@src/app.module.js'
import { AppErrorFilter } from '@src/shared/filters/appError.filter.js'

let app: NestFastifyApplication

beforeAll(async () => {
  const moduleRef = await Test.createTestingModule({
    imports: [AppModule],
  }).compile()

  app = moduleRef.createNestApplication<NestFastifyApplication>(new FastifyAdapter())
  app.useGlobalFilters(new AppErrorFilter())
  await app.init()
  await app.getHttpAdapter().getInstance().ready()
})

afterAll(async () => {
  await app.close()
})
```

Test file structure mirrors `src/`:

```
test/
└── domain/
    └── <domain>/
        ├── <domain>.controller.test.ts     ← always exists
        ├── <domain>.service.test.ts        ← only if controller cannot cover it
        └── <domain>.repository.test.ts     ← only if controller cannot cover it
```
