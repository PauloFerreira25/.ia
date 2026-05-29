---
name: vuejs
description: "Read before writing or modifying any Vue.js code."
---

# Vue.js Architecture

> Base document for Vue.js projects. Describes architecture decisions, conventions, and responsibilities for each layer. All examples assume Vue 3 + Composition API + Vite + TypeScript. Read before writing or modifying any view, component, composable, store, or routing file.
>
> **This document prescribes how to do things — it does not describe what has already been done.** The rule is what matters, not the example. The current state of the project may differ; the goal is to converge toward these guidelines.
>
> **Domain values in examples are always illustrative.** Store names, logger namespaces, error codes, route names, folder structures — when they appear in examples, they show the pattern to follow, not the required values. The project defines the concrete values; the document defines the structure.
>
> **On guidelines that seem project-specific:** some decisions in this document (such as the HTTP envelope format or the behavior of the authentication client) are established best practices, not impositions from a specific project. When there is no project document that overrides or complements one of these guidelines, this is the current rule. Documents numbered `30+` may override or extend any part of this document.

---

## Index

| Section | Contents | When to read |
|---|---|---|
| **S1 — Philosophy and Patterns** | General principles, Composition API, Atomic Design, type safety, naming conventions, linting, logs, tests, project visual patterns | Always — any task |
| **S2 — Infrastructure** | Folder structure, layers (view / composable / service / store / httpClient), response envelope, routing, config, types, i18n, barrel exports | When creating or changing structure, new files, new layers |
| **S3 — Implementation** | `authStore`, domain stores (bootstrap/clear), route guards (`getToken()`), error views | Only when implementing or maintaining: auth, session, bootstrap, login flow, error views |
| **S4 — Interaction Patterns** | Loading (`useAsyncState` local + global overlay), errors (philosophy, types, `useErrorHandler`, `useAsyncAction`), pagination (full scan / on demand), batch actions, forms (`useFormAction`, `useDestructiveAction`), empty states | When implementing any view with loading, error, list, form, or action |

**Task → section mapping:**

- Create or edit any view, component, composable, util → **S1 + S2 + S4**
- Unsure where to put a file or which layer to use → **S2**
- Implement login, logout, bootstrap, error view → **S1 + S2 + S3**
- Maintain `authStore`, router guards → **S3**
- Implement a form → **S4** (`useFormAction`, `useDestructiveAction`)
- Implement a list with pagination or per-item actions → **S4** (Pagination, Batch Actions)
- Implement loading, error, or empty state → **S4** (`useAsyncState`, `useErrorHandler`, `useAsyncAction`)
- Implement a service mock → **S2** (Service + mock rules)

---

## S1 — Philosophy and Patterns

### Philosophy

The same decisions that guide the backend, adapted for the frontend:

**Production first.** Every decision is made with production behavior in mind. Development convenience (mocks, fixed data, shortcut routes) is temporary and must never reach the final version.

**Consistency eliminates decisions.** Where to fetch data, how to handle errors, where to store state, how to structure the UI — these answers are always the same.

**Explicit over implicit.** Types declared in named interfaces. No field inferred from `any`. Global state declared in stores with explicit types. Props typed with `defineProps<T>()`.

**Mock is scaffolding, not architecture.** Mocks exist to allow the frontend to move forward before the backend is ready. Every mock has an expiration date: the moment the real endpoint is delivered. Never add business logic to a mock.

**Reuse before creating.** Before writing anything, check whether there is already — in the project or in installed libs — a component, composable, util, service, or type that solves the problem.

**Does it not exist? Ask before creating.** If the needed component does not exist in the project or in installed libs, do not create it on your own. Ask the user: what should this component do, how should it behave, what variations does it need to support, where will it be used. Creating a component without these answers is prototyping — and this is not a prototyping project.

**Real project, not a prototype.** The shortest solution is rarely the correct one. The criterion is not "works now" — it is "works well, is consistent, and can be maintained". Shortcuts that save minutes create hours of rework.

**Separation of concerns.** Each layer has one responsibility and does not cross its boundary. The view does not make HTTP calls. The service does not know about routing. The store does not know the API. The component does not know the store.

**UI built in layers.** The interface follows the Atomic Design model: atoms form molecules, molecules form organisms, organisms compose layouts, layouts sustain views. No layer skips a level. This hierarchy ensures that changing an atom propagates the change throughout the system in a predictable way.

---

### Composition API

Always use the Composition API with `<script setup lang="ts">`. Never use the Options API.

```vue
<!-- wrong -->
<script lang="ts">
export default {
  data() { return { name: '' } },
  methods: { save() { /* ... */ } }
}
</script>

<!-- correct -->
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')

function save(): void { /* ... */ }
</script>
```

`<script setup>` is the canonical form of Vue 3. It offers better type inference, less boilerplate, and better tree-shaking.

**Reactivity rules:**

- Use `ref()` for primitives and `reactive()` for objects when the entire object needs reactivity
- Prefer `ref()` in most cases — it is more consistent and avoids the destructuring problem of `reactive()`
- Use `computed()` for derived values — never recalculate in the template
- Use `watch()` and `watchEffect()` sparingly — in most cases, `computed()` is sufficient

```typescript
// correct — ref for primitive, computed for derived value
const items  = ref<Item[]>([])
const isEmpty = computed(() => items.value.length === 0)
```

---

### TypeScript

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "jsx": "preserve",
    "lib": ["ESNext", "DOM"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.vue"]
}
```

#### Path alias

Use `@/` as an alias for `src/`. Never use relative paths with `../` to import from outside the current directory.

```typescript
// wrong
import { userService } from '../../../services/user/userService'

// correct
import { userService } from '@/services/user/userService'
```

Configure the alias in `vite.config.ts`:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'node:path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(import.meta.dirname, 'src'),
    },
  },
})
```

---

### Atomic Design

The project UI follows the **Atomic Design** model. Each piece of interface belongs to exactly one level, and that level defines its responsibility contract.

| Level        | Where                  | Description                                                                                                                                              |
| ------------ | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Atom**     | `components/atom/`     | Minimum and indivisible piece. No composition of other project components. Examples: `Button.vue`, `Badge.vue`, `Input.vue`.                             |
| **Molecule** | `components/molecule/` | Composition of atoms with a single purpose. Examples: `SearchBar.vue` (input + button), `ListItem.vue` (avatar + text + badge).                          |
| **Organism** | `components/organism/` | Complete and self-contained section, composed of atoms and molecules. Examples: `UserCard.vue`, table with header and pagination.                         |
| **Layout**   | `layouts/`             | Structural skeleton of the view — defines where header, content, and footer go. Contains no data. Examples: `ListLayout.vue`, `FormLayout.vue`.           |
| **View**     | `views/`               | The page itself. Uses a layout, composes organisms/molecules/atoms, fetches data via services, and handles routing.                                       |

#### Rules

- Each level can only compose elements from its own level or lower levels. An atom does not use molecules; a molecule does not use organisms.
- Within each level (`atom/`, `molecule/`, `organism/`), components are organized in subdirectories by functional category (e.g., `atom/buttons/`, `atom/badges/`).
- No component below View knows the router, stores, or services directly.
- Before creating any new component, check whether one already exists at the appropriate level that solves the problem.
- **Lib components can be used at any level.** Atoms, molecules, and organisms may use components from installed libs directly.
- **Only create a component if it is a specialization.** If the lib already offers a button, use it directly. Create an `atom/buttons/BtnConfirm.vue` only if there is project-specific behavior or styling that justifies the specialization. Wrappers without purpose are prohibited.

---

### Type safety

#### No `any`

Never use `any`. If the type is unknown, use `unknown` and narrow before using the value.

```typescript
// wrong
const data = response as any

// correct
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'id' in value
}
```

#### Typed props

Always type props with `defineProps<T>()`. Never use the array form or an object without types.

```typescript
// wrong
const props = defineProps(['name', 'age'])

// correct
interface Props {
  name:  string
  age?:  number
}
const props = defineProps<Props>()
```

#### Typed emits

Always type emits with `defineEmits<T>()`.

```typescript
// correct
const emit = defineEmits<{
  save:   [data: FormData]
  cancel: []
}>()
```

---

### File names and conventions

**Vue components use `PascalCase.vue`.** Vue requires that components starting with an uppercase letter be treated as custom components, not HTML elements.

**Views encapsulate the access hierarchy.** The name starts with the access level (`Public` or `Private`), followed by the domain and the action — forming a self-documenting chain that eliminates collisions between views from different domains.

```
Public  + Auth  + Login       → PublicAuthLoginView.vue
Public  + Error + Generic     → PublicErrorGenericView.vue
Private + Admin + Users       → PrivateAdminUsersView.vue
Private + Admin + Users + Add → PrivateAdminUsersAddView.vue
```

**Files:**

| Type        | Convention                                                               | Example                                        |
| ----------- | ------------------------------------------------------------------------ | ---------------------------------------------- |
| View        | `PascalCase` + `View.vue` following the access hierarchy                 | `PrivateAdminUsersView.vue`                    |
| Atom        | `PascalCase.vue` inside `atom/<category>/`                               | `atom/buttons/BtnBack.vue`                     |
| Molecule    | `PascalCase.vue` inside `molecule/<category>/`                           | `molecule/cards/EventCard.vue`                 |
| Organism    | `PascalCase.vue` inside `organism/<category>/`                           | `organism/lists/UserList.vue`                  |
| Layout      | `PascalCase` + `Layout.vue` inside `layouts/<category>/`                 | `layouts/forms/FormLayout.vue`                 |
| Store       | `camelCase` + `Store.ts`                                                 | `authStore.ts`                                 |
| Service     | `camelCase` + `Service.ts`                                               | `userService.ts`                               |
| Mock        | `camelCase` + `Service.mock.ts`                                          | `userService.mock.ts`                          |
| Composable  | `camelCase` with `use` prefix + `.ts` inside `composables/<domain>/`     | `composables/commons/useAsyncAction.ts`        |
| Util        | `camelCase` describing the operation + `.ts` inside `utils/<domain>/`    | `utils/dateTime/formatter.ts`                  |
| Type        | `camelCase.ts` inside `types/<domain>/`                                  | `types/user/user.model.ts`                     |
| Config      | `camelCase.ts`                                                           | `config/index.ts`                              |

**Identifiers in code:**

- `camelCase` — variables, functions, parameters, object properties, service and store exports
- `PascalCase` — Vue components (framework requirement), interfaces and type aliases
- Descriptive and self-explanatory names — avoid abbreviations

---

### Language of identifiers

**Best practice: all identifiers in English.** Variable names, functions, Vue components, types, interfaces, files, stores, services, composables, utils, and any other code element must be written in English. This promotes consistency, improves readability by tools and international integrations, and eliminates ambiguity in mixed teams.

**Exception: when the human explicitly requests it.** If the human deliberately requests names in another language, respect the decision. The default rule is English — any deviation must be explicitly requested.

---

### Linting

Every project must have ESLint configured with `typescript-eslint` and `eslint-plugin-vue`:

```javascript
// eslint.config.js
import tseslint from 'typescript-eslint'
import pluginVue from 'eslint-plugin-vue'
import importPlugin from 'eslint-plugin-import'

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  ...pluginVue.configs['flat/recommended'],
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
        extraFileExtensions: ['.vue'],
      },
    },
    plugins: { import: importPlugin },
    rules: {
      '@typescript-eslint/consistent-type-imports':         ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-import-type-side-effects':     'error',
      '@typescript-eslint/explicit-function-return-type':   ['error', { allowExpressions: true }],
      '@typescript-eslint/explicit-module-boundary-types':  'error',
      '@typescript-eslint/no-unused-vars': ['error', {
        vars: 'all',
        args: 'all',
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      'vue/component-api-style':      ['error', ['script-setup']],
      'vue/block-lang':               ['error', { script: { lang: 'ts' } }],
      'vue/define-props-declaration': ['error', 'type-based'],
      'vue/define-emits-declaration': ['error', 'type-based'],
      'import/no-duplicates':         'error',
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        pathGroups: [{ pattern: '@/**', group: 'internal' }],
        'newlines-between': 'always',
      }],
    },
  },
  { ignores: ['dist/'] }
)
```

Install required plugins:

```
npm install -D typescript-eslint eslint-plugin-vue eslint-plugin-import
```

Lint errors are not warnings — they are errors. CI rejects code with lint errors.

**No eslint-disable.** Never use `eslint-disable`, `eslint-disable-next-line`, `eslint-disable-line`, or any variant of ESLint suppression in `.js`, `.ts`, or `.vue` files. If a lint error appears, fix the code. If an existing `eslint-disable` is found, remove it and fix the underlying problem. If the problem cannot be resolved without suppressing the rule, stop and notify a human.

---

### Logs

All project logs go through `logger` — never use `console.log`, `console.info`, or `console.warn` directly in the code. Direct console logs have no level control, no namespace, and cannot be turned off in production.

```typescript
// src/utils/logger/logger.ts
const base = createLogger({ level: import.meta.env.PROD ? 'warn' : 'debug' })

export const logger = {
  auth:    base.extend('AUTH'),
  service: base.extend('SERVICE'),
  store:   base.extend('STORE'),
}
```

```typescript
// usage
import { logger } from '@/utils/logger/logger'

logger.auth.debug('bootstrap started')
logger.service.error('failed to fetch users', error)
```

The logging library is defined by the project. The example above shows the expected interface — namespaces per layer, configurable levels per environment.

Rules:

- Never commit `console.log` — the linter must block it
- `logger.error` and `logger.warn` are active in production (sent to monitoring service)
- `logger.debug` and `logger.info` are silenced in production by default

---

### Tests

Use Vitest as the test runner. It integrates natively with Vite and understands TypeScript and ESM without additional configuration.

**Testing philosophy:**

- Test at the highest level possible. The correct test for a view is an E2E test that simulates the user using the app — it covers view, store, and service all at once.
- Component tests in isolation rarely justify their maintenance cost.

| What to test          | Level | Justification                                                                 |
| --------------------- | ----- | ----------------------------------------------------------------------------- |
| `utils/`              | Unit  | Pure functions, easy to test and high value                                   |
| `composables/`        | Unit  | Reusable logic, testable without a full DOM                                   |
| View flows            | E2E   | Covers everything at once — it is the "user using the app"                    |
| Isolated components   | —     | Avoid — E2E already covers them, and component tests tend to test implementation |

The E2E tool (Playwright or Cypress) is defined by the project in the `30+` file.

---

### Project visual patterns (`XX-ui-patterns.md`)

This document covers architecture and principles — decisions that apply to any Vue.js project. Project-specific visual decisions belong to a separate document, numbered in the `30+` range, following the index in `00-index.md`.

**Every project that uses this base document must create its own visual patterns file.** Without it, each view invents its own answer to recurring situations — and inconsistency is inevitable.

That file covers decisions that repeat across every view but depend on the project's design. Examples of what needs to be defined there:

- How to display a loading state
- How to display errors to the user
- How to display empty lists
- How to structure forms and display validation errors
- Recurring visual components (`ConfirmDialog`, `EmptyState`, `SkeletonCard`)

The list is not exhaustive. Whenever a recurring situation has no defined answer, the decision goes into that file — it does not remain implicit in the code.

---

## S2 — Infrastructure

### Package installation

Always install packages without specifying a version:

```
npm install <package>
```

- Never manually edit `package.json` to add a dependency with a fixed version
- Specifying a version (`npm install <package>@x.y.z`) is only allowed when a human explicitly requests it, or when another already-installed dependency requires that version as a peer dependency

Pinning a version installs an outdated package instead of the current version, accumulating security vulnerabilities over time. Any AI agent that writes a version directly into `package.json` without running `npm install` introduces this risk.

---

### Folder structure

```
src/
├── components/       ← UI pieces without business logic (Atomic Design)
│   ├── atom/
│   │   ├── buttons/
│   │   ├── badges/
│   │   └── inputs/
│   ├── molecule/
│   │   ├── cards/
│   │   └── list-items/
│   └── organism/
│       └── ...
├── layouts/          ← structural view templates
│   ├── forms/        ← FormLayout.vue
│   └── list/         ← ListLayout.vue
├── views/            ← views mirroring the route hierarchy
│   ├── public/
│   │   ├── auth/     ← PublicAuthLoginView.vue
│   │   └── error/    ← PublicErrorGenericView.vue, PublicErrorBootstrapView.vue
│   └── private/
│       └── <domain>/ ← Private<Domain><Action>View.vue
├── router/           ← Vue Router organized by access hierarchy
│   ├── index.ts      ← createRouter, global guards
│   ├── public.ts     ← public routes
│   └── private.ts    ← private routes (meta: { requiresAuth: true })
├── stores/           ← Pinia stores per entity
├── services/         ← API calls (real + mock)
│   └── api/          ← HTTP clients
├── composables/      ← reusable stateful logic (equivalent to hooks)
│   └── commons/      ← generic composables without a specific domain
├── config/           ← environment variables — a single index.ts
├── types/            ← TypeScript interfaces and types, subdirectory per domain
│   └── router/       ← RouteMeta declaration
├── utils/            ← pure functions without Vue reactivity
│   ├── dateTime/
│   └── string/
└── locales/          ← i18n translation files
    ├── pt-BR.json
    └── en.json
```

---

### Layers and responsibilities

#### View (`views/`)

The view is responsible for:

- Rendering the UI based on local state and stores
- Calling services to fetch or modify data
- Handling local loading and error states
- Navigating between routes

The view **never**:

- Makes HTTP calls directly
- Contains business logic or data transformation
- Knows the implementation details of the service

```typescript
// wrong — view calling fetch directly
const data = await fetch('/v1/users')

// correct — view calling the service
const users = await userService.list()
```

**Error handling in the view.** Every service call must be inside a `try/catch`. The view decides what to do with the error:

- Business error (validation failed, resource not found) → local `ref`, inline feedback to the user
- Critical error (unrecoverable failure, invalid state) → `useReportCriticalError()` → `errorStore.setError(err)` + `router.push({ name: 'public-error-generic' })`

```typescript
const reportCriticalError = useReportCriticalError()

try {
  users.value = await userService.list()
} catch (err) {
  if (err instanceof ApiError && err.code === 'NOT_FOUND') {
    errorMessage.value = t('user.notFound')  // expected error — inline
  } else {
    reportCriticalError(err)  // unexpected error — errorStore + redirect
  }
}
```

#### Service (`services/`)

The service is responsible for:

- Making HTTP calls via `request()` from `httpClient`
- Encapsulating the URL, method, and payload for each endpoint
- Returning typed data to the view

The service **never**:

- Knows the router or UI state
- Knows stores — it does not import or access any store
- Contains domain business logic
- Is called by another service

The service pattern has two files, organized in subdirectories by domain:

```
services/
├── api/
│   ├── backend.httpClient.ts     ← main backend
│   └── erp.httpClient.ts         ← ERP backend (when needed)
├── auth/
│   ├── authService.ts
│   └── authService.mock.ts
└── user/
    ├── userService.ts
    └── userService.mock.ts
```

Views import **only** the real service. The mock is an implementation detail of the service:

```typescript
// userService.ts — while the backend does not exist
import { userServiceMock } from './userService.mock'

export const userService = {
  // TODO: backend not implemented — replace when endpoint is ready
  list: () => userServiceMock.list(),
}
```

When the endpoint is ready, the mock is removed and the `.mock.ts` file is deleted:

```typescript
// userService.ts — backend implemented
import { backendHttpClient } from '@/services/api/backend.httpClient'

export const userService = {
  list: () => backendHttpClient.request<User[]>('/v1/users'),
}
```

**Mock rules:**

- **Mandatory separation:** mock logic never goes into the real service file. The `.mock.ts` file is the only place where mocks exist. The real service only delegates to it temporarily via `import`.
- Every use of a mock has a `// TODO: backend not implemented` comment
- If the endpoint exists, there is no justification for keeping the mock — it is technical debt
- When all endpoints of a service are implemented, the `.mock.ts` file is deleted
- The migration is done **only** in the real service — views do not change

As a complement, the `VITE_USE_MOCKS=true` variable can be used to make mock usage explicit per environment — useful for forcing a runtime error when a service still delegates to the mock in an environment where it should not.

#### HTTP clients (`services/api/`)

Each file in `services/api/` is an HTTP client dedicated to a backend. A client:

- Injects the JWT in the `Authorization` header by reading the token directly from `authStore` (in memory) — never from `localStorage`. `localStorage` is accessed only in `authStore.bootstrap()`.
- Unwraps the API response envelope and throws an error when the response indicates failure
- Calls the 401 handler registered by the router guard (when applicable)

**Default response envelope** — this is the base model; the project's `30+` document defines the actual structure if the backend diverges:

```typescript
// success (list)
{ data: T[], pagination: { pageSize: number, nextCursor: string | null, hasNext: boolean, ... } }

// success (single item)
{ data: T }

// error
{ "error": "ENTITY_NOT_FOUND", "message": "Entity abc-123 not found" }
```

- `error` — `UPPER_SNAKE_CASE` code, stable and programmatically comparable. When coming from the backend, it is passed through `t()` to display to the user.
- `message` — human-readable text, may change — do not use for logic.

The `httpClient` reads `authStore` directly — this is a tolerable coupling without a viable alternative. Passing the token as a parameter in every call pollutes all services. The client is infrastructure, not a domain service, and this is the only exception to the rule that infrastructure layers do not know stores.

No file outside `services/` imports an HTTP client directly.

#### Store (`stores/`)

Pinia stores hold session state — data that needs to be available in any view without prop drilling.

Always use the **setup store** style (Composition API) in Pinia — never the options style:

```typescript
// stores/authStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null)

  function setToken(newToken: string): void {
    token.value = newToken
  }

  function clearToken(): void {
    token.value = null
  }

  return { token, setToken, clearToken }
})
```

Examples of typical stores:

- `authStore` — JWT token and session identifiers
- `accountStore` — account data loaded after login
- `errorStore` — current critical error, used to redirect the user to an error view
- `loadingStore` — state of the global loading overlay

> **`errorStore` and `loadingStore` are optional.** They exist for projects that adopt global error and loading handling — an overlay that blocks the entire interface, a centralized error view. Not every project needs this. The project's `30+` document must evaluate whether these stores are needed, how they are structured, and which components consume them. Without that documented decision, do not create them on your own.

**When to use a store:**

- Data shared between views with a single point of modification (e.g., username — any view reads it, only login changes it)
- Session data that needs to be available anywhere in the app

**When not to use a store:**

- View-local data (loading, error, form values) → local `ref`
- Data the view fetches for itself → `ref` + service
- Data passed between views via navigation → route params. Params identify and orient — they do not transport state. Passing `id`, `type`, or fields that define the behavior of the next view is correct. Passing a complete domain object via router state is prohibited: it disappears on F5 and creates implicit coupling between views.

General rules:

- Each store uses setup store with explicitly typed `ref`s
- Stores do not call `httpClient` directly — all API communication is delegated to services
- `localStorage` and `sessionStorage` are accessed exclusively by stores
- Errors that the store does not know how to resolve are always propagated to the caller

#### Composables (`composables/`)

Composables are functions that use Vue reactivity APIs internally (`ref`, `reactive`, `watch`, etc.). They are the place for reusable stateful logic between views or components.

```
composables/
├── commons/          ← generic composables without a specific domain
│   ├── useAsyncAction.ts
│   ├── useErrorHandler.ts
│   └── useReportCriticalError.ts
└── <domain>/         ← composables specific to a domain
    └── useUserFilter.ts
```

**When to create a composable:** when the same stateful logic appears in more than one place. If the logic only exists in one view, it stays in the view.

**Composables are not `utils/`.** If the function does not use any Vue reactivity API, it is a pure function and goes into `utils/`.

#### Utils (`utils/`)

Utils are pure functions without Vue reactivity — formatting, validation, parsing, calculations. They can be used in any context.

```
utils/
├── dateTime/
│   ├── formatter.ts
│   └── validator.ts
└── string/
    ├── formatter.ts
    └── parser.ts
```

The subdirectory defines the domain. The file defines the operation (`formatter`, `validator`, `parser`, `calculator`). Together they are self-explanatory without repetition.

**If you need Vue reactivity inside, it is not a util — it is a composable.**

---

### Routing (`router/`)

Use Vue Router. Organize routes in two separate files by access level:

```typescript
// router/public.ts
import type { RouteRecordRaw } from 'vue-router'

export const publicRoutes: RouteRecordRaw[] = [
  {
    path: '/login',
    name: 'public-auth-login',
    component: () => import('@/views/public/auth/PublicAuthLoginView.vue'),
  },
  {
    path: '/error',
    name: 'public-error-generic',
    component: () => import('@/views/public/error/PublicErrorGenericView.vue'),
  },
]
```

```typescript
// router/private.ts
import type { RouteRecordRaw } from 'vue-router'

export const privateRoutes: RouteRecordRaw[] = [
  {
    path: '/',
    meta: { requiresAuth: true },
    component: () => import('@/layouts/PrivateLayout.vue'),
    children: [
      {
        path: 'users',
        name: 'private-admin-users',
        component: () => import('@/views/private/admin/PrivateAdminUsersView.vue'),
      },
    ],
  },
]
```

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'
import { publicRoutes } from './public'
import { privateRoutes } from './private'

const router = createRouter({
  history: createWebHistory(),
  routes: [...publicRoutes, ...privateRoutes],
})

router.beforeEach((to) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.getToken()) {
    return { name: 'public-auth-login' }
  }
})

export default router
```

Rules:

- **Route names in `kebab-case`** — they encapsulate the access hierarchy: `private-admin-users`
- **Lazy loading is mandatory** — every view uses `() => import(...)`. Never use static imports in routes.
- **Error views inside public routes** — critical errors do not require authentication
- The global guard is the sole responsible for redirecting protected routes — never check authentication inside a view
- The guard uses `authStore.getToken()`, not `authStore.token`. On page reload, the token has not yet been loaded into memory — `getToken()` falls back to `localStorage`. Using `authStore.token` directly would cause an incorrect redirect to login before bootstrap finishes.
- Declare route meta in `types/router/meta.ts` so that `to.meta.requiresAuth` is typed

```typescript
// types/router/meta.ts
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
  }
}
```

---

### Config (`config/index.ts`)

Environment variables that change between builds are kept in a single file. No other file accesses `import.meta.env` directly.

```typescript
// src/config/index.ts
export const API_BASE_URL       = import.meta.env.VITE_API_URL ?? 'http://localhost:3000'
export const API_TIMEOUT        = 10_000
export const PAGE_SIZE          = Number(import.meta.env.VITE_PAGE_SIZE ?? 20)
export const SKELETON_COUNT     = Number(import.meta.env.VITE_SKELETON_COUNT ?? 5)
export const TOAST_VISIBILITY_MS = Number(import.meta.env.VITE_TOAST_VISIBILITY_MS ?? 30_000)
```

The `VITE_` prefix is mandatory for Vite to expose the variable to the client.

---

### Types (`types/`)

The standard is to always create types in `types/`, with a subdirectory per domain.

**Using `index.ts` as a barrel export anywhere in `src/` is prohibited.** This rule is global — it applies to `types/`, `components/`, `composables/`, `services/`, `utils/`, and other directories. The file path is semantic information: `import { User } from '@/types/user/user.model'` says it is the domain model. `import { User } from '@/types/user/user.api'` says it is an API contract type. An `index.ts` that re-exports everything erases that information, hinders tree-shaking, and increases the risk of circular imports.

```
types/
├── router/
│   └── meta.ts            ← RouteMeta declaration
├── user/
│   ├── user.model.ts      ← domain model (what the app uses internally)
│   └── user.api.ts        ← API request/response types
└── event/
    ├── event.model.ts
    └── event.api.ts
```

**Local type is the exception, not the rule.** Use a local type only when it is truly internal to a single file and has no use elsewhere. If there is any chance the type will be used in another file — view, store, service, component — it goes into `types/`.

---

### Internationalization (i18n)

Use `vue-i18n` for multi-language support. The current language is stored in a Pinia store — never managed only locally.

```
npm install vue-i18n
```

**i18n bootstrap:**

```
authStore.bootstrap()
  → languageStore.bootstrap()  ← loads saved language from localStorage + applies via i18n
  → accountStore.bootstrap()
  → ...
```

Key organization by domain:

```json
{
  "validation": {
    "required": "Campo obrigatório.",
    "email":    "Email inválido.",
    "min":      "Mínimo {{count}} caracteres."
  },
  "action": {
    "save":   "Salvar",
    "cancel": "Cancelar",
    "delete": "Excluir",
    "add":    "Adicionar"
  }
}
```

**Mandatory rule:** no text visible to the user is hardcoded. Every string defined in the frontend goes through `t()`. This includes: labels, placeholders, view titles, button text, empty state messages, feedback toasts.

**Content coming from the backend — case by case:**

- Backend sent an **error code** (e.g., `"USER_NOT_FOUND"`) → pass through `t()`, translating the code into a readable message
- Backend sent **domain data** (username, product title, description) → display directly, without `t()`. These are values entered by users, not system constants — translating them makes no sense.

```vue
<script setup lang="ts">
import { useI18n } from 'vue-i18n'
const { t } = useI18n()
</script>

<template>
  <input :placeholder="t('user.name')" />
  <button>{{ t('action.save') }}</button>
</template>
```

---

## S3 — Implementation

> Read this section only when implementing or maintaining: auth, session, bootstrap, login flow, critical error views.

### authStore (`stores/authStore.ts`)

`authStore` is the session repository and the orchestrator of authentication and store initialization.

Responsibilities:

- Store and expose the JWT token in memory
- Implement `getToken()`: checks memory first — if not found, fetches from `localStorage`, saves to memory and returns. If not found in either, returns `null`.
- Implement `login(credentials)`: calls `authService.login()`, persists the token in `localStorage` and in memory, returns success or propagates the exception for the view to handle
- Implement `bootstrap()`: calls `bootstrap()` on each store that needs to be initialized — called by `PrivatePostLoginLoadingView`
- Implement `clearAuth()`: clears the token from `localStorage` and memory, and triggers `clear()` on other stores

**Login flow:**

```
PublicAuthLoginView — user submits the form
  → authStore.login(credentials)
    → authService.login(credentials)
      → success → token saved in localStorage and authStore
        → authStore returns success to PublicAuthLoginView
          → router.push({ name: 'private-post-login-loading' })
            → PrivatePostLoginLoadingView calls authStore.bootstrap()
              → success → router.push({ name: 'private-home' })
              → UNAUTHORIZED failure → 401 route guard already cleared the session
                                       and redirected to public-auth-login
              → other failure → reportCriticalError(err)
                                 → PublicErrorBootstrapView
      → failure → exception propagated → PublicAuthLoginView handles it (inline feedback)
```

---

### Domain stores

Each domain store (e.g., `accountStore`, `personStore`) is self-contained in its initialization.

Responsibilities:

- Store and expose its domain data
- Implement `bootstrap()`: calls its own service to fetch and populate data
- Implement `clear()`: clears data on logout

```typescript
// standard interface for initializable stores
interface DomainStore {
  bootstrap(): Promise<void>
  clear(): void
}
```

`authStore.bootstrap()` knows which stores need to be initialized:

```typescript
async function bootstrap(): Promise<void> {
  await languageStore.bootstrap()
  await accountStore.bootstrap()
  await personStore.bootstrap()
}
```

**Rule:** every store with `bootstrap()` **must** be explicitly added to `authStore.bootstrap()`. This list is the source of truth for the app's initialization dependencies. Creating a store with `bootstrap()` without adding it here is a bug — the store will start empty and the view will break silently.

`authStore` knowing the domain stores is an intentional and defensive coupling — it makes initialization dependencies explicit and visible in a single place. The list is small by design: only the stores strictly necessary for the app to function are initialized in the bootstrap. Domain stores that are not critical at initial load do not belong here.

---

### Route guards and session

The global guard in `router/index.ts` is the sole responsible for redirecting protected routes.

**`authStore.clearAuth()` is called in two contexts:**

- **`httpClient`** — upon receiving a 401, the client calls `authStore.clearAuth()` and the guard redirects to login
- **Logout view** — in response to the user's explicit logout

No other service, composable, or view calls `clearAuth()` directly.

---

### Error views

**`PublicErrorBootstrapView`** — exclusively for session bootstrap failures. Displays the error and offers a retry button that navigates to `PrivatePostLoginLoadingView`.

```
PrivatePostLoginLoadingView calls authStore.bootstrap()
  → failure → reportCriticalError(err)
               → errorStore.setError(err)
               → router.push({ name: 'public-error-bootstrap' })
                 → user triggers retry → router.push({ name: 'private-post-login-loading' })
                   → PrivatePostLoginLoadingView calls authStore.bootstrap() again
```

**`PublicErrorGenericView`** — used by any view in the app for unexpected errors not covered by a specific view. Reads the error from `errorStore`.

**What every error view must do:**

- Inform that the system encountered an error and guide the user (try again or contact support)
- Offer the appropriate recovery action for the context
- Display technical error details (code, timestamp) to support troubleshooting
- **Never expose sensitive data** — JWT tokens, passwords, and personal data must never appear

---

## S4 — Interaction Patterns

> Read this section when implementing any view with loading, error, empty list, or form.

---

### Loading

The project has two loading contexts with distinct behaviors.

The project has two loading contexts with distinct tools.

#### Local loading — initial data fetch

Use `useAsyncState` from VueUse. It exposes reactive `isLoading`, `state`, and `error`, eliminates the boilerplate of `ref<boolean>` + try/catch, and is the standard for fetching data when the view mounts.

```
npm install @vueuse/core
```

**Views with a form or single content** — displays a centered spinner while waiting:

```vue
<script setup lang="ts">
import { useAsyncState } from '@vueuse/core'

const { state: user, isLoading } = useAsyncState(() => userService.get(id), null)
</script>

<template>
  <div v-if="isLoading" class="loading-center">
    <Spinner />
  </div>
  <div v-else><!-- content --></div>
</template>
```

**Views with a list** — displays `SKELETON_COUNT` repetitions of `SkeletonCard`:

```vue
<template>
  <template v-if="isLoading">
    <SkeletonCard v-for="i in SKELETON_COUNT" :key="i" />
  </template>
  <template v-else><!-- list --></template>
</template>
```

`SKELETON_COUNT` comes from `config/index.ts`. `SkeletonCard` lives in `components/atom/feedback/SkeletonCard.vue`.

#### Global overlay — user actions

Use `useAsyncAction` (internal project composable) when the user executes an action that calls the API (save, delete, login, logout, etc.). It blocks the entire interface via `loadingStore`, preventing double-clicks and accidental navigation during the operation.

`useAsyncState` and `useAsyncAction` **are not interchangeable:**

| | `useAsyncState` (VueUse) | `useAsyncAction` (internal) |
|---|---|---|
| When to use | Initial data fetch on mount | Action triggered by the user |
| Global overlay | No | Yes |
| Error classification | No | Yes (network / business / critical) |

**Rule:** every `async` operation triggered by the user uses `useAsyncAction`. No exceptions.

Synchronous store operations (setters, clearers) **do not** use the overlay — they are instantaneous.

---

### Errors

#### Philosophy

**The view is the one that decides what to do with each error.** Errors bubble up from the service layer to the store, and from the store to the view — no intermediate layer silences errors that are not its responsibility.

In the view, the decision follows a simple hierarchy:

1. **Recognized domain error** — the view knows the error code and knows exactly how to react: show an inline message on a field, display a specific toast, redirect to another route. The view handles it and signals that the error was handled.
2. **Generic error** — network, timeout, unknown error, API error without specific handling → `useErrorHandler` applies the default treatment (network toast or critical redirect).

**Never silence an error with an empty `catch (err) {}`.**

---

#### Error types and how they appear

| Type | Origin | Presentation |
|---|---|---|
| Form validation | Frontend (zod/vee-validate) | Inline below the field |
| API domain error | Backend (known `error` code) | Handled by the view (inline or specific toast) |
| Generic business error | Backend (unknown `error` code) | Toast via `useErrorHandler` |
| No connection / timeout | Network | Network toast via `useErrorHandler` |
| Expired session (401) | `httpClient` | Intercepted by the client — no view handles 401 directly |
| Critical / unexpected error | Any layer | `useReportCriticalError` → `errorStore` + redirect |

The 401 is intercepted by `httpClient`:
```
httpClient receives 401 → authStore.clearAuth() → router guard redirects to login
```

---

#### `showToast` — mandatory wrapper

Every toast call goes through `showToast` in `utils/toast/toast.ts`. Never call the toast lib directly.

```typescript
// utils/toast/toast.ts
import { TOAST_VISIBILITY_MS } from '@/config'

export function showToast(params: ToastParams): void {
  Toast.show({ duration: TOAST_VISIBILITY_MS, ...params })
}
```

---

#### `useReportCriticalError`

Saves the error to `errorStore` and redirects to the generic error view. Used as a last resort — when the view does not recognize the error.

```typescript
// composables/commons/useReportCriticalError.ts
export function useReportCriticalError() {
  const router     = useRouter()
  const errorStore = useErrorStore()

  return (err: unknown): void => {
    errorStore.setError(err)
    void router.push({ name: 'public-error-generic' })
  }
}
```

---

#### `useErrorHandler`

Classifies and presents generic errors. Lives in `composables/commons/useErrorHandler.ts`. It is the default handler for everything the view did not explicitly handle.

```typescript
// composables/commons/useErrorHandler.ts
export function useErrorHandler() {
  const reportCritical = useReportCriticalError()
  const { t } = useI18n()

  return (err: unknown): void => {
    if (isNetworkError(err)) {
      showToast({ type: 'networkError', text: t('error.network.message') })
    } else if (err instanceof ApiError) {
      showToast({ type: 'error', text: err.message })
    } else {
      reportCritical(err)
    }
  }
}
```

`isNetworkError` lives in `utils/error/classifier.ts` and is the sole responsible for distinguishing network errors from business errors.

---

#### `useAsyncAction`

Wraps user-triggered actions with a loading overlay and error handling. `onError` is where the view handles domain errors — returning `true` signals that the error was handled and the generic handler is not called.

```typescript
// composables/commons/useAsyncAction.ts
interface AsyncActionOptions {
  onSuccess?: () => void
  onError?:   (err: unknown) => boolean | void
  overlay?:   boolean  // default: true — false for batch actions
}

export function useAsyncAction() {
  const handleError  = useErrorHandler()
  const loadingStore = useLoadingStore()

  return async (action: () => Promise<void>, options?: AsyncActionOptions): Promise<void> => {
    const showOverlay = options?.overlay ?? true
    if (showOverlay) loadingStore.show()
    try {
      await action()
      options?.onSuccess?.()
    } catch (err) {
      const handled = options?.onError?.(err)
      if (!handled) handleError(err)
    } finally {
      if (showOverlay) loadingStore.hide()
    }
  }
}
```

**Example with domain error:**

```typescript
const runAction = useAsyncAction()

const onSubmit = handleSubmit(async (values) => {
  await runAction(() => userService.create(values), {
    onError: (err) => {
      if (err instanceof ApiError && err.code === 'EMAIL_IN_USE') {
        emailError.value = t('user.emailInUse') // inline on the field
        return true // handled — does not fall through to the generic handler
      }
      // no return → falls through to useErrorHandler automatically
    },
    onSuccess: () => { showToast({ type: 'success', ... }); router.back() }
  })
})
```

---

#### Errors on initial data fetch (`useAsyncState`)

For the initial data fetch, error handling goes in the `onError` callback of `useAsyncState`:

```typescript
const handleError = useErrorHandler()

const { state: users, isLoading } = useAsyncState(
  () => userService.list(),
  [],
  {
    onError: (err) => {
      if (err instanceof ApiError && err.code === 'FORBIDDEN') {
        noAccess.value = true // domain error — view handles directly
        return
      }
      handleError(err) // generic — delegates to the default handler
    }
  }
)
```

---

### Empty States

When a list has no items, the `EmptyState` component replaces the content.

```typescript
// components/atom/feedback/EmptyState.vue
interface Props {
  icon:     string
  message:  string
  action?:  { label: string; onClick: () => void }
}
```

- Icon + contextual message defined by the view — never generic (`"No items"`)
- Centered in the content area
- Optional action — passed only when it makes sense for the business

---

### Paginated lists

The pagination model is always cursor-based. Which of the two cases below to use is dictated by the business rule — not by technical preference.

---

#### Case 1 — Full scan: load everything, handle on the frontend

Used when the operation requires access to the complete dataset — arbitrary sorting, cross-filters, multiple selection, export. The frontend fetches all records in batches via cursor until the end and applies sorting, filtering, and display locally.

Examples: administering all users, managing a product catalog.

```typescript
const items     = ref<Item[]>([])
const isLoading = ref(false)

async function loadAll(): Promise<void> {
  isLoading.value = true
  items.value = []
  let cursor: string | null = null

  try {
    do {
      const res = await itemService.list({ cursor: cursor ?? undefined, pageSize: PAGE_SIZE })
      items.value = [...items.value, ...res.data]
      cursor = res.pagination.nextCursor
    } while (cursor)
  } finally {
    isLoading.value = false
  }
}
```

Filter, sorting, and search are done over `items` with `computed()` — without a new API call.

> The viable volume limit for Case 1 depends on the project, the page, and the business rule. When the volume degrades frontend performance, the filter must migrate to the backend — a decision documented in the domain's `30+` file.

---

#### Case 2 — Continuation: load on demand

Used when the data has a natural order (most recent, by creation date) and the user advances as needed. The frontend fetches the first batch and displays a "Load more" button while there is a cursor.

Examples: listing the 50 most recent events, showing products in order of creation.

```typescript
const items       = ref<Item[]>([])
const nextCursor  = ref<string | null>(null)
const loadingMore = ref(false)

async function loadInitial(): Promise<void> {
  const res = await itemService.list({ pageSize: PAGE_SIZE })
  items.value     = res.data
  nextCursor.value = res.pagination.nextCursor
}

async function loadMore(): Promise<void> {
  loadingMore.value = true
  try {
    const res = await itemService.list({ cursor: nextCursor.value ?? undefined, pageSize: PAGE_SIZE })
    items.value      = [...items.value, ...res.data]
    nextCursor.value = res.pagination.nextCursor
  } finally {
    loadingMore.value = false
  }
}
```

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ItemCard :item="item" />
  </div>
  <EmptyState v-if="!isLoading && items.length === 0" ... />
  <button
    v-if="nextCursor"
    :disabled="loadingMore"
    @click="loadMore"
  >
    {{ t('action.loadMore') }}
  </button>
</template>
```

Infinite scroll is not used — the user explicitly controls when to fetch more items.

---

The batch size comes from `config/index.ts` (`PAGE_SIZE`).

> This document does not cover WebSockets, SSE, or any real-time update mechanism. Projects that require these features must document them in a `30+` file.

---

### Forms

#### Validation stack

- **`vee-validate`** — manages values, errors, and form state
- **`zod`** — defines the validation schema with automatically generated TypeScript types
- **`@vee-validate/zod`** — integrates vee-validate with zod via `toTypedSchema`

```
npm install vee-validate zod @vee-validate/zod
```

#### Validation behavior

- **onBlur** — when leaving a field, validates that field. Untouched fields do not display errors.
- **On submit** — validates all fields, including untouched ones.
- **Submit button** — enabled in the initial state. Disabled when there are visible errors.

#### `useFormAction`

Composable that encapsulates the complete form cycle: validation, dirty check, loading overlay, and error handling. Receives the `useForm` instance and returns a function that creates the submit handler. Used in **all forms** — creation and editing.

```typescript
// composables/commons/useFormAction.ts
export function useFormAction<T extends Record<string, unknown>>(
  form: ReturnType<typeof useForm<T>>
) {
  const runAction  = useAsyncAction()
  const router     = useRouter()
  const { t }      = useI18n()
  const errorCount = ref(0)

  return (action: (data: T) => Promise<void>, options?: AsyncActionOptions) =>
    form.handleSubmit(
      async (data) => {
        errorCount.value = 0
        if (!form.meta.value.dirty) {
          void router.back()  // nothing changed — intentional behavior, no feedback
          return
        }
        await runAction(() => action(data), options)
      },
      () => {
        errorCount.value++
        if (errorCount.value >= 3) {
          errorCount.value = 0
          showToast({ type: 'error', text: t('form.hasErrors.message') })
        }
      }
    )
}
```

**Internal flow:**
1. Validates all fields on submit — marks untouched ones with an error if invalid
2. If nothing changed (`dirty = false`) — navigates back without calling the API. Intentional behavior: nothing to save, nothing to communicate.
3. If invalid 3 times in a row — shows a toast guiding the user to check the fields (resolves fields outside the visible area)
4. Executes the action via `useAsyncAction` — overlay + error handling

#### Creation form

```vue
<script setup lang="ts">
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

const schema = z.object({
  name:  z.string().min(1),
  email: z.string().email(),
})

const form = useForm({ validationSchema: toTypedSchema(schema) })

const [name,  nameAttrs]  = form.defineField('name',  { validateOnBlur: true })
const [email, emailAttrs] = form.defineField('email', { validateOnBlur: true })

const runFormAction = useFormAction(form)

const handleSave = runFormAction(async (values) => {
  await userService.create(values)
  showToast({ type: 'success', ... })
  router.back()
})
</script>

<template>
  <p class="required-note">{{ t('form.requiredFieldsNote') }}</p>
  <input v-model="name" v-bind="nameAttrs" :placeholder="`${t('user.name')} *`" />
  <span v-if="form.errors.value.name">{{ form.errors.value.name }}</span>
  <input v-model="email" v-bind="emailAttrs" :placeholder="t('user.email')" />
  <button :disabled="Object.keys(form.errors.value).length > 0" @click="handleSave">
    {{ t('action.save') }}
  </button>
</template>
```

#### Edit form — pre-population

In edit forms, data is fetched via `useAsyncState` and the form is populated with `resetForm` when the data arrives. vee-validate's `dirty` compares against the `initialValues` passed to `resetForm` — if the user returns to the original value, `dirty` goes back to `false`.

```vue
<script setup lang="ts">
import { watch } from 'vue'
import { useAsyncState } from '@vueuse/core'

const props = defineProps<{ id: string }>()

const schema = z.object({ name: z.string().min(1), email: z.string().email() })
const form   = useForm({ validationSchema: toTypedSchema(schema) })

const [name,  nameAttrs]  = form.defineField('name',  { validateOnBlur: true })
const [email, emailAttrs] = form.defineField('email', { validateOnBlur: true })

const handleError = useErrorHandler()

const { state: user, isLoading } = useAsyncState(
  () => userService.get(props.id),
  null,
  { onError: (err) => handleError(err) }
)

// Populates the form when data arrives — resetForm sets the initialValues
// that vee-validate uses to calculate dirty
watch(user, (val) => {
  if (val) form.resetForm({ values: { name: val.name, email: val.email } })
}, { immediate: true })

const runFormAction = useFormAction(form)

const handleSave = runFormAction(async (values) => {
  await userService.update(props.id, values)
  showToast({ type: 'success', ... })
  router.back()
})
</script>
```

#### `useAsyncAction` directly in the view

For simple user-triggered actions without a form and without confirmation — archive, activate/deactivate, resend email, sync. `onError` handles specific domain errors; the generic handler covers the rest.

#### `useDestructiveAction`

For irreversible actions. Displays a confirmation dialog, waits for the response, executes only on OK.

**Every irreversible action uses `useDestructiveAction`.** The exception is when a human explicitly states that the dialog should be removed — document with a comment in the code.

```typescript
// composables/commons/useDestructiveAction.ts
export function useDestructiveAction() {
  const runAction    = useAsyncAction()
  const { t }        = useI18n()
  const visible      = ref(false)
  const dialogConfig = ref({ title: '', message: '' })
  const pendingAction = ref<(() => void) | null>(null)

  function destructiveAction(
    action: () => Promise<void>,
    options?: AsyncActionOptions & { title?: string; message?: string }
  ): void {
    dialogConfig.value = {
      title:   options?.title   ?? t('dialog.destructive.title'),
      message: options?.message ?? t('dialog.destructive.message'),
    }
    pendingAction.value = () => runAction(action, options)
    visible.value = true
  }

  function confirm(): void { visible.value = false; pendingAction.value?.() }
  function cancel():  void { visible.value = false }

  return { destructiveAction, visible, dialogConfig, confirm, cancel }
}
```

```vue
<script setup lang="ts">
const { destructiveAction, visible, dialogConfig, confirm, cancel } = useDestructiveAction()

const handleDelete = (): void => destructiveAction(
  async () => { await userService.delete(id); router.back() },
  { message: t('user.deleteConfirm', { name: user.value.name }) }
)
</script>

<template>
  <button @click="handleDelete">{{ t('action.delete') }}</button>
  <ConfirmDialog
    :visible="visible"
    :title="dialogConfig.title"
    :message="dialogConfig.message"
    @confirm="confirm"
    @cancel="cancel"
  />
</template>
```

> The `ConfirmDialog` referenced here is a required project component. Its props/emits interface, placement in the Atomic Design hierarchy, and relationship with the UI lib must be detailed in a complementary document numbered `30+`.

#### Separation of handlers and template

Handlers are defined as `const` in `<script setup>` — never as inline functions in the template.

```typescript
// correct — handler defined before the template
const handleSave = runFormAction(async (data) => {
  await userService.update(id, data)
  showToast({ type: 'success', ... })
  router.back()
})
```

```html
<!-- correct -->
<button @click="handleSave">{{ t('action.save') }}</button>

<!-- wrong — inline logic in the template -->
<button @click="async () => { await service.update(id, data); router.back() }">Salvar</button>
```

Inline functions are allowed only for trivial cases: `@click="router.back()"`.

#### Success feedback

Every action that completes successfully displays a `success` toast — no exceptions.

**Navigation rule after success:**

- Forms (creation and editing) → toast + `router.back()`
- Single-purpose actions (delete, confirm) → toast + `router.back()`
- Batch actions (toggles in a list) → toast per action + debounce to reload the list

#### Per-item actions in a list

When each item in a list has its own action (e.g., a "change status" button), each button needs independent loading and disabled states — the global overlay is not used.

Per-item loading is tracked by a reactive `Set` of IDs:

```typescript
const loadingIds = ref(new Set<string>())
const runAction  = useAsyncAction()

function handleToggleStatus(item: Item): void {
  void runAction(
    async () => {
      loadingIds.value.add(item.id)
      const updated = await itemService.toggleStatus(item.id)
      const index = items.value.findIndex(i => i.id === item.id)
      if (index !== -1) items.value[index] = updated  // updates in-place
    },
    {
      overlay: false,
      onSuccess: () => { loadingIds.value.delete(item.id) },
      onError:   () => { loadingIds.value.delete(item.id); return false },
    }
  )
}
```

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ItemCard :item="item" />
    <button :disabled="loadingIds.has(item.id)" @click="handleToggleStatus(item)">
      <Spinner v-if="loadingIds.has(item.id)" />
      <span v-else>{{ t('action.toggleStatus') }}</span>
    </button>
  </div>
</template>
```

**Two strategies for updating the list after the action:**

- **In-place update** — the service returns the updated item and the view replaces it directly in the array. Smoother UX, no re-fetch.
- **Refresh with debounce** — after each action, schedules a list reload with `useDebounceFn` from VueUse. Simpler, guarantees consistency with the backend. Preferable when the server does not return the updated item.

The choice depends on what the endpoint returns and the business rule — defined in the domain's `30+` file.

#### When to use each action composable

| Composable | When to use |
|---|---|
| `useFormAction` | Every form — creation and editing |
| `useAsyncAction` | Actions without a form and without confirmation (archive, activate/deactivate, resend) |
| `useDestructiveAction` | Irreversible actions that need confirmation (delete, cancel) |

#### Required fields

Indicated with an asterisk after the label: `Name *`. A subtle note at the top of the form explains the asterisk — once, not on every field.

```vue
<p class="required-note">{{ t('form.requiredFieldsNote') }}</p>
<input :placeholder="`${t('user.name')} *`" v-bind="nameAttrs" v-model="name" />
<input :placeholder="t('user.phone')"        v-bind="phoneAttrs" v-model="phone" />
```
