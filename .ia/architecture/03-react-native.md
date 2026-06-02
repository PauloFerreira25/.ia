---
name: react-native
description: "Read before writing or modifying any React Native / Expo code."
---

# React Native + Expo Architecture

> Base document for React Native projects. Describes architectural decisions, conventions, and responsibilities of each layer. Examples and configurations use Expo as the toolchain — adapt where necessary if the project does not use Expo. Read before writing or modifying any screen, component, service, store, or navigation file.
>
> **This document prescribes how to do things — it does not describe what has already been done.** The rule is what matters, not the example. The current state of the project may differ; the goal is to converge toward these guidelines.
>
> **Domain values in examples are always illustrative.** Store names, logger namespaces, error codes, screen names, folder structures — when they appear in examples, they show the pattern to follow, not the required values. The project defines the concrete values; the document defines the structure.
>
> **On guidelines that seem project-specific:** some decisions in this document (such as the HTTP response envelope format or the behavior of the authentication client) are established best practices, not impositions from a specific project. When there is no project document that replaces or complements one of these guidelines, this is the current rule. Documents numbered `30+` may override or extend any part of this document.

---

## Index

| Section | Content | When to read |
| --- | --- | --- |
| **S1 — Philosophy and Patterns** | General principles, TypeScript, Atomic Design, type safety, naming conventions, linting, logs, performance, tests | Always — any task |
| **S2 — Infrastructure** | Dependencies, folder structure, layers (screen / service / HTTP / store / context), navigation, theme, config, types, hooks, utils | When creating or changing structure, new files, new layers |
| **S3 — Implementation** | `authStore`, domain stores (bootstrap/clear), critical errors and error screens | Only when implementing or maintaining: auth, session, bootstrap, login flow, error screens |
| **S4 — Interaction Patterns** | Loading (local + global overlay), errors (types, hooks, toast), empty states, forms (validation, action hooks) | When implementing any screen with loading, error, empty list, or form |

**Task → section mapping:**

- Create or edit any screen, component, hook, util → **S1 + S2 + S4**
- Unsure where to place a file or which layer to use → **S2**
- Implement login, logout, bootstrap, error screen → **S1 + S2 + S3**
- Maintain `authStore`, `AuthContext`, `errorStore` → **S3**
- Implement form, loading, error feedback, or empty state → **S4**

---

## S1 — Philosophy and Patterns

### Philosophy

The same decisions that guide the backend, adapted for the frontend:

**Production first.** Every decision is made with production behavior in mind. Development convenience (mocks, navigation shortcuts, fixed data) is temporary and must never reach the final version.

**Consistency eliminates decisions.** When the cost of deciding case by case is higher than applying uniformly, the rule becomes absolute. Where to fetch data, how to handle errors, where to store state, how to structure the UI — these answers are always the same.

**Explicit over implicit.** Types declared in named interfaces. No field inferred from `any`. Global state declared in stores with explicit interfaces.

**Mock is scaffolding, not architecture.** Mocks exist to let the frontend advance before the backend is ready. Every mock has an expiration date: the moment the real endpoint is delivered. Never add business logic to a mock.

**Reuse before creating.** Before writing anything, check whether a component, hook, util, service, or type already exists in the project — or in installed libraries — that solves the problem. The search starts in `utils/` and `hooks/` for pure and stateful logic, in the Atomic Design levels for UI, and in the APIs of installed libraries.

**Doesn't exist? Ask before creating.** If the needed component does not exist in the project or installed libraries, do not create it on your own. Ask the user: what should this component do, how should it behave, what variations does it need to support, where will it be used. Creating a component without these answers is prototyping — and this is not a prototyping project.

**Real project, not a prototype.** The shortest solution is rarely the correct one. The criterion is not "works now" — it is "works well, is consistent, and can be maintained". Shortcuts that save minutes create hours of rework.

**Subdirectories are preferable to accumulated files.** A directory with a few well-named files is more navigable than one with dozens of flat files. Organize by domain from the start — it is easier to expand an already-segmented structure than to refactor a chaotic directory later.

**Separation of concerns.** Each layer has one responsibility and does not cross its boundary. The screen does not make HTTP calls. The service does not know about navigation. The store does not know about the API. The component does not know about the store.

**UI built in layers.** The interface follows the Atomic Design model: atoms form molecules, molecules form organisms, organisms compose layouts, layouts sustain screens. No layer skips a level. This hierarchy ensures that changing an atom propagates the change to the entire system in a predictable way.

---

### TypeScript

The project uses the base Expo config (`expo/tsconfig.base`) with `strict: true`. Do not change compiler options without documented justification.

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

#### Path alias

Use `@/` as an alias for `src/`. Never use relative paths with `../` to import from outside the current directory.

```typescript
// wrong
import { theme } from "../../../theme";

// correct
import { theme } from "@/theme";
```

---

### Atomic Design

The project UI follows the **Atomic Design** model. Each piece of interface belongs to exactly one level, and that level defines its responsibility contract.

| Level | Location | Description |
| --- | --- | --- |
| **Atom** | `components/atom/` | Minimum and indivisible piece. No composition of other project components. Examples: `Button`, `Badge`, `Avatar`, `Input`. |
| **Molecule** | `components/molecule/` | Composition of atoms with a single purpose. Examples: `SearchBar` (input + button), `ListItem` (avatar + text + badge). |
| **Organism** | `components/organism/` | Complete and self-sufficient section, composed of atoms and molecules. Examples: `UserCard`, table with header and pagination. |
| **Layout** | `layouts/` | Structural skeleton of the screen — defines where the header, content, and footer are. Contains no data. Examples: `ListScreenLayout`, `FormScreenLayout`. |
| **Screen** | `screens/` | The screen itself. Uses a layout, composes organisms/molecules/atoms, fetches data via services, and handles navigation. |

#### Rules

- Each level can only compose elements from its own level or lower. An atom does not use molecules; a molecule does not use organisms.
- Within each level (`atom/`, `molecule/`, `organism/`), components are organized in subdirectories by functional category (e.g., `atom/buttons/`, `atom/badges/`).
- No component below Screen knows about navigation, stores, or services.
- Before creating any new component, check whether one already exists at the appropriate level that solves the problem.
- **Library components can be used at any level.** Atoms, molecules, and organisms can directly use components from installed libraries.
- **Only create a component if it is a specialization.** If the library already provides a `Button`, use it directly. Create an `atom/buttons/BtnConfirm` only if there is project-specific behavior or style that justifies the specialization. Purposeless wrappers are prohibited.

---

### Type safety

#### No `any`

Never use `any`. If the type is not known, use `unknown` and narrow before using the value.

```typescript
// wrong
const err = e as any;
console.log(err.code);

// correct
const err = e as Error & { code?: string };
console.log(err.code);
```

#### No `as` for domain type casting

Never use `as` to force a domain type on an unknown value. Use type guards when necessary.

```typescript
// wrong
const account = result as Account;

// correct
function isAccount(value: unknown): value is Account {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "email" in value
  );
}
```

The `as` operator is only allowed in one specific case: interop with libraries that lack proper types — documented with an inline comment explaining the reason. For React Navigation props, the correct type comes from the ParamList via `StackScreenProps` or `useRoute<RouteProp<...>>` — no `as` needed.

---

### File names and conventions

**Base rule: everything is `camelCase`.** The only exception is files that export React components (screens, atoms, molecules, organisms, layouts, navigators) — these use `PascalCase` because React requires components to start with a capital letter. Components starting with a lowercase letter are interpreted as HTML elements and do not work.

**Screen and stack names encapsulate the access hierarchy.** The name starts with the access level (`Public` or `Private`), followed by the domain and the action — forming a chain that is self-documenting and eliminates collisions between screens from different domains.

```
Public  + Auth  + Login        → PublicAuthLoginScreen
Public  + Error + Generic      → PublicErrorGenericScreen
Private + Admin + Users        → PrivateAdminUsersScreen
Private + Admin + Users + Add  → PrivateAdminUsersAddScreen
```

`PrivateAdminUsersAddScreen` is different from `PrivateUsersAddScreen` — the `Admin` domain is in the name, not just in the directory.

**Files:**

| Type | Convention | Example |
| --- | --- | --- |
| Screen | `PascalCase` + `Screen.tsx` following access hierarchy | `PrivateAdminUsersScreen.tsx` |
| Stack | `PascalCase` + `Stack.tsx` following access hierarchy | `PrivateAdminStack.tsx` |
| Atom | `PascalCase.tsx` inside `atom/<category>/` | `atom/buttons/BtnBack.tsx` |
| Molecule | `PascalCase.tsx` inside `molecule/<category>/` | `molecule/cards/EventCard.tsx` |
| Organism | `PascalCase.tsx` inside `organism/<category>/` | `organism/lists/UserList.tsx` |
| Layout | `PascalCase` + `Layout.tsx` inside `layouts/<category>/` | `layouts/forms/FormScreenLayout.tsx` |
| Navigator | `PascalCase.tsx` | `AppNavigator.tsx`, `PrivateMainTabs.tsx` |
| Context | `PascalCase` + `Context.tsx` | `AuthContext.tsx` |
| Service | `camelCase` + `Service.ts` | `userService.ts` |
| Mock | `camelCase` + `Service.mock.ts` | `userService.mock.ts` |
| Store | `camelCase` + `Store.ts` | `authStore.ts` |
| Type | `camelCase.ts` inside `types/<domain>/` | `types/user/user.model.ts` |
| Config | `camelCase.ts` | `config/index.ts` |
| Hook | `camelCase` with `use` prefix + `.ts` inside `hooks/<domain>/` | `hooks/commons/modal/useGenericModal.ts` |
| Util | `camelCase` describing the operation + `.ts` inside `utils/<domain>/` | `utils/dateTime/formatter.ts` |

**Code identifiers:**

- `camelCase` — variables, functions, parameters, object properties, service and store exports
- `PascalCase` — React components (React requirement), interfaces and type aliases
- Descriptive and self-explanatory names — avoid abbreviations. Only use those widely recognized by the community. Never create new abbreviations.

---

### Identifier language

**Best practice: all identifiers in English.** Variable names, functions, components, types, interfaces, files, stores, services, hooks, utils, and any other code element must be written in English. This promotes consistency, facilitates readability by tools and international integrations, and eliminates ambiguity in mixed teams.

**Exception: when the human explicitly requests it.** If the human deliberately requests names in another language, respect the decision. The default rule is English — any deviation must be explicitly requested.

---

### Linting

Every project must have a linter configured. Run before committing — the command depends on the project, check `package.json`.

Lint errors are not warnings — they are errors. CI rejects code with lint errors. Linter-specific configuration (rules, plugins, extends) is defined by the project.

**No eslint-disable.** Never use `eslint-disable`, `eslint-disable-next-line`, `eslint-disable-line`, or any variant of ESLint suppression comments in `.js`, `.ts`, or `.tsx` files. If an ESLint error appears, fix the code. If an existing `eslint-disable` is found, remove it and fix the underlying issue. If the issue cannot be resolved without suppressing the rule, stop and notify a human.

---

### Logs

All project logs go through the `logger` — never use `console.log`, `console.info`, or `console.warn` directly in the code. Direct console logs have no level control, no namespace, and cannot be silenced in production.

The library is `react-native-logs`. It supports levels, configurable transports, and namespace loggers.

#### Setup

`logger` is an object with mutable namespace properties — not a frozen constant. This is intentional: when the log level changes at runtime, `recreateLogger` replaces the internal instance and reassigns every namespace. Any call to `logger.service.debug(...)` always reads the current namespace at call time.

```typescript
// utils/logger/logger.ts
import { createLogger, consoleTransport } from 'react-native-logs'

export type LogLevel = 'debug' | 'info' | 'warn' | 'error'

export const DEFAULT_LEVEL: LogLevel = __DEV__ ? 'debug' : 'error'

function build(level: LogLevel) {
  return createLogger({
    levels: { debug: 0, info: 1, warn: 2, error: 3 },
    transport: consoleTransport,
    severity: level,
  })
}

let _log = build(DEFAULT_LEVEL)

export const logger = {
  auth:    _log.extend('AUTH'),
  service: _log.extend('SERVICE'),
  store:   _log.extend('STORE'),
  screen:  _log.extend('SCREEN'),
  hook:    _log.extend('HOOK'),
}

export function recreateLogger(level: LogLevel): void {
  _log = build(level)
  logger.auth    = _log.extend('AUTH')
  logger.service = _log.extend('SERVICE')
  logger.store   = _log.extend('STORE')
  logger.screen  = _log.extend('SCREEN')
  logger.hook    = _log.extend('HOOK')
}
```

Never destructure the logger at import time — always call `logger.service.debug(...)` directly, never `const { service } = logger`. Destructuring captures a reference to the old namespace and will not reflect level changes after `recreateLogger` is called.

#### loggerStore

The log level lives in a Zustand store. Changing the level calls `recreateLogger` and updates the store state. No persistence — the level resets to the build default on every app launch.

```typescript
// store/loggerStore.ts
import { create } from 'zustand'

import { recreateLogger, DEFAULT_LEVEL, type LogLevel } from '@/utils/logger/logger'

interface LoggerStore {
  level: LogLevel
  setLevel: (level: LogLevel) => void
}

export const useLoggerStore = create<LoggerStore>((set) => ({
  level: DEFAULT_LEVEL,
  setLevel: (level) => {
    recreateLogger(level)
    set({ level })
  },
}))
```

`DEFAULT_LEVEL` is imported from `logger.ts` — the single place where `__DEV__` is checked for logging purposes.

#### Settings screen

Any authenticated user can change the log level via a settings screen. The screen reads `level` from `useLoggerStore` and calls `setLevel`. No access control, no feature flag — this is local state on the user's device, not a server-side concern.

This solves the production debugging scenario: open the settings screen, switch to `debug`, reproduce the issue, read the logs in the browser console or device logger.

#### Rules

**First line of every function with side effects is a debug log — no exceptions.**

Deciding where to log and where not to log costs more than logging everywhere. A missing log is always a blind spot. The rule is absolute — no function with side effects is exempt.

Functions with side effects: service methods, store actions, custom hooks, screen handlers (`handleSave`, `handleDelete`, etc.). Pure React components that only render JSX are excluded — they re-render frequently and produce noise, not signal.

| Layer | Namespace |
|---|---|
| Service methods | `logger.service` |
| Store actions | `logger.store` |
| Custom hooks | `logger.hook` |
| Screen handlers | `logger.screen` |
| Auth flow | `logger.auth` |

```typescript
// service method
async update(params: UpdatePersonParams): Promise<void> {
  logger.service.debug({ params }, 'update')
  // ...
}

// store action
setAccount(account: Account): void {
  logger.store.debug({ account }, 'setAccount')
  // ...
}

// screen handler
const { id } = useRoute<RouteProp<...>>().params
const handleSave = runFormAction(async (data) => {
  logger.screen.debug({ id, data }, 'handleSave')
  // ...
})
```

**The second argument is always the exact function or method name — never a description, never a variation.** This makes `grep 'update'` on logs find exactly that function with no ambiguity.

**Log the full parameter object. Never pre-filter fields anticipating what might be useful later — that creates the blind spots the logging system exists to eliminate. The log level controls what is visible in each environment, not the choice of fields.**

The only exception is genuinely sensitive data (passwords, tokens). These fields are explicitly omitted — the log still exists, only the sensitive field is absent.

```typescript
async login(params: LoginParams): Promise<void> {
  logger.auth.debug({ email: params.email }, 'login') // omits password — sensitive field
}
```

Suppressing the log entirely is never acceptable as a workaround for sensitive data.

**Never use `console.log`, `console.warn`, or `console.error` directly.** The linter must block it. Every log goes through `logger`.

---

### Performance

#### ScrollView vs FlatList

This is the most common performance pitfall in React Native. The wrong choice freezes the app with large lists.

| Component | Renders | Use when |
| --- | --- | --- |
| `ScrollView` | **All children at once** | Fixed and small content — forms, detail screens, settings |
| `FlatList` | **Only what is visible** (virtualized) | Dynamic data lists — anything that comes from an API |
| `SectionList` | **Only what is visible**, with sections | Lists grouped by category |

**Rule:** if the content comes from an API or can grow, use `FlatList`. `ScrollView` with many items renders everything in memory and freezes the UI.

```typescript
// wrong — freezes with large lists
<ScrollView>
  {products.map(p => <ProductCard key={p.id} product={p} />)}
</ScrollView>

// correct — renders only what is visible
<FlatList
  data={products}
  keyExtractor={p => p.id}
  renderItem={({ item }) => <ProductCard product={item} />}
/>
```

#### Render optimizations (`React.memo`, `useCallback`, `useMemo`)

React re-renders child components when the parent re-renders. In some scenarios this is unnecessary and causes slowness — `React.memo`, `useCallback`, and `useMemo` exist to prevent this.

**Do not apply these optimizations by default.** They add complexity and in most cases React is fast enough without them. Used without real need, they complicate the code without benefit.

**If you identify a scenario where they seem necessary, do not decide alone — present the case to the user and validate before applying.** The criterion is: is there a measured or clearly visible performance problem? If not, do not use.

---

### Tests

A test that does not find real bugs has no value. Do not create tests to achieve coverage — create tests to guarantee behavior.

**Highest level possible.** The right test for a screen is an E2E test that simulates the user using the app — it covers the screen, store, and service all at once, just as the handler test covers handler, service, and repository in the backend. Isolated component tests rarely justify the maintenance cost.

| What to test | Level | Justification |
| --- | --- | --- |
| `utils/` | Unit | Pure functions, no UI, easy to test and high value |
| Screen flows | E2E | Covers everything at once, it is the "user using" |
| Isolated components | — | Avoid — E2E already covers it, and component tests tend to test implementation, not behavior |

Test tool configuration (Jest, Maestro, Detox) is defined by the project in the `30+` file.

---

### Project visual patterns (`XX-ui-patterns.md`)

This document covers architecture and principles — decisions that apply to any React Native project. Project-specific visual decisions belong in a separate document, numbered in the `30+` range, following the index in `00-index.md`.

**Every project that uses this base document must create its own visual patterns file.** Without it, each screen invents its own answer to recurring situations — and inconsistency is inevitable.

That file covers decisions that repeat across every screen but depend on the project's design. Examples of what needs to be defined there:

- How to display loading state
- How to display errors to the user
- How to display empty lists
- How to structure forms and display validation errors

The list is not closed. Whenever a recurring situation has no defined answer, the decision goes to that file — it does not stay implicit in the code.

---

## S2 — Infrastructure

### Dependencies and libraries

This is a real, long-term project — not a prototype. Every decision must consider maintainability, consistency, and scale.

**Read `package.json` before writing anything.** The installed libraries define the project's vocabulary. They already solve navigation, state, UI, persistence, and HTTP requests. The work is to compose these solutions, not reinvent them.

**Use the concepts the libraries already provide.** If the UI library has typography variants, use those variants — do not create manual `fontSize`. If the navigation library has a pattern for stacks, use that pattern — do not create custom navigation. Adopting the library's model is always preferable to creating a parallel one.

**Before installing a new dependency**, check whether an already installed library solves the problem. Adding a dependency should be the last option, not the first.

#### Package installation

Always install packages without specifying a version:

```
npm install <package>
```

- Never manually edit `package.json` to add a dependency with a pinned version
- Specifying a version (`npm install <package>@x.y.z`) is only allowed when a human explicitly requests it, or when another installed dependency already requires that version as a peer dependency

Pinning a specific version installs an outdated package instead of the current version, accumulating security vulnerabilities over time. Every AI agent that hard-codes a version into `package.json` without running `npm install` introduces this risk.

---

### Folder structure

```
src/
├── components/      ← UI pieces without business logic (Atomic Design)
│   ├── atom/        ← minimum and indivisible pieces
│   │   ├── buttons/
│   │   ├── badges/
│   │   └── text/
│   ├── molecule/    ← compositions of atoms with a purpose
│   │   ├── cards/
│   │   └── list-items/
│   └── organism/    ← complete and self-sufficient sections
│       └── ...
├── layouts/         ← structural screen templates
│   ├── forms/       ← FormScreenLayout, FormTabScreenLayout
│   └── list/        ← ListScreenLayout
├── config/          ← environment configurations — a single index.ts (see Config section)
├── contexts/        ← React Contexts created by the project (see rule below)
├── navigation/      ← navigation organized by access hierarchy
│   ├── AppNavigator.tsx
│   ├── public/
│   │   ├── PublicStack.tsx
│   │   ├── PublicAuthStack.tsx
│   │   └── PublicErrorStack.tsx
│   └── private/
│       ├── PrivateStack.tsx
│       ├── PrivateMainTabs.tsx
│       └── <domain>/     ← one subdirectory per functional area
│           └── Private<Domain>Stack.tsx
├── screens/         ← screens mirroring the navigation hierarchy
│   ├── public/
│   │   ├── auth/    ← PublicAuthLoginScreen, PublicAuthRegisterScreen
│   │   └── error/   ← PublicErrorGenericScreen, PublicErrorBootstrapScreen
│   └── private/
│       ├── <domain>/  ← Private<Domain><Action>Screen
│       └── ...        ← one subdirectory per functional area
├── services/        ← API calls (real + mock)
├── store/           ← Zustand stores per entity
├── theme/           ← global theme (react-native-paper)
├── types/           ← TypeScript interfaces and types, subdirectory per domain
│   └── navigation/  ← ParamList for each stack — one file per stack
├── hooks/           ← reusable custom hooks, subdirectory per domain
│   ├── commons/     ← generic hooks with no specific domain
│   └── <domain>/    ← domain-specific hooks
└── utils/           ← pure functions without hooks, subdirectory per domain
    ├── dateTime/    ← formatter.ts, validator.ts
    └── string/      ← formatter.ts, parser.ts
```

---

### Layers and responsibilities

#### Screen (`screens/`)

The screen is responsible for:

- Rendering the UI based on local state and stores
- Calling services to fetch or modify data
- Handling local loading and error states
- Navigating between screens

The screen **never**:

- Makes HTTP calls directly (does not use `fetch`, does not use `request`)
- Contains business logic or data transformation
- Knows the implementation details of the service

```typescript
// wrong — screen calling fetch directly
const data = await fetch("/v1/persons");

// correct — screen calling the service
const persons = await personService.list();
```

**Error handling in the screen.** Every call to a service or store must be inside a `try/catch`. The screen is the only layer that decides what to do with an error:

- Business error (validation failed, resource not found, no permission) → local `useState`, inline feedback to the user
- Critical error (unrecoverable failure, invalid state) → `useReportCriticalError()` — encapsulates `errorStore.setError(error)` and `navigate('PublicErrorGenericScreen')`

```typescript
// correct
const reportCriticalError = useReportCriticalError();

try {
  const persons = await personService.list();
  setPersons(persons);
} catch (err) {
  if (err instanceof ApiError && err.code === "NOT_FOUND") {
    setError("No person found."); // business error — inline feedback
  } else {
    reportCriticalError(err); // unexpected error — errorStore + navigate
  }
}
```

#### Service (`services/`)

The service is responsible for:

- Making HTTP calls via `request()` from `httpClient`
- Encapsulating the URL, method, and payload for each endpoint
- Returning typed data to the screen

The service **never**:

- Knows about navigation or UI state
- Knows about stores — does not import or access any store
- Contains domain business logic (business validations, transformations)
- Is called by another service

**Error handling in the service.** The service can and should handle errors that are its responsibility — retry on timeout, normalization of HTTP error codes, etc. What the service never does is silently discard an error outside its domain. Errors the service cannot resolve are always propagated to the caller.

The service pattern has two files, organized in subdirectories per domain. HTTP infrastructure lives in `services/api/`, separate from domain services:

```
services/
├── api/                          ← HTTP infrastructure — one file per backend
│   ├── backend.httpClient.ts      ← main backend
│   ├── erp.httpClient.ts         ← ERP backend
│   └── facebook.httpClient.ts    ← external API
├── auth/
│   ├── authService.ts
│   └── authService.mock.ts
├── event/
│   ├── eventService.ts
│   └── eventService.mock.ts
└── user/
    ├── userService.ts
    └── userService.mock.ts
```

Each domain service imports the appropriate HTTP client from `@/services/api/`. If a service needs two backends, it imports two clients.

Screens and components import **only** the real service. The mock is an implementation detail of the service.

```typescript
// xyzService.ts — while the backend does not exist
import { xyzServiceMock } from "./xyzService.mock";

export const xyzService = {
  // TODO: backend not implemented — replace when endpoint is ready
  list: () => xyzServiceMock.list(),
};
```

When the endpoint is ready, the mock is removed and the `.mock.ts` file is deleted:

```typescript
// xyzService.ts — backend implemented, no mock
import { backendHttpClient } from "@/services/api/backend.httpClient";

export const xyzService = {
  list: () => backendHttpClient.request<Xyz[]>("/v1/xyz"),
};
```

**Mock rules:**

- Every mock usage has a `// TODO: backend not implemented` comment — makes it easy to locate what is still missing
- If the endpoint exists, there is no justification to keep the mock — using a mock with an available backend is technical debt
- When all endpoints of a service are implemented, the `.mock.ts` file is deleted
- The migration is done **only** in the real service — screens do not change

#### HTTP clients (`services/api/`)

Each file in `services/api/` is an HTTP client dedicated to one backend. A client:

- Injects the JWT in the `Authorization` header by reading the token directly from `authStore` (in memory) — never from `AsyncStorage`. `AsyncStorage` is only accessed in `authStore.bootstrap()` during app initialization.
- Unwraps the API response envelope and throws an error when the response indicates failure
- Calls the 401 handler registered by `AuthContext` (when applicable)

**Standard response envelope** — this is the base model; the project's `30+` document defines the actual structure if the backend differs:

```typescript
// success (list)
{ data: T[], pagination: { pageSize: number, nextCursor: string | null, hasNext: boolean, ... } }

// success (single item)
{ data: T }

// error
{ "error": "ENTITY_NOT_FOUND", "message": "Entity abc-123 not found" }
```

- `error` — `UPPER_SNAKE_CASE` code, stable and programmatically comparable. When coming from the backend, passed through `t()` to display to the user.
- `message` — human-readable text, may change — do not use for logic.

No file outside `services/` imports an HTTP client directly.

#### Store (`store/`)

Zustand stores hold session state of the authenticated user — data that needs to be available in any screen without prop drilling.

Stores live directly in `store/`, without subfolders. Examples of typical stores:

- `authStore` — JWT token and session identifiers
- `accountStore` — account data loaded after login
- `personStore` — person data loaded after login
- `errorStore` — current critical error, used to redirect the user to an error screen

**When to use a store:**

- Data shared between screens that has a single modification point (e.g., user name — any screen reads it, only login changes it)
- Session data that needs to be available anywhere in the app without prop drilling

**When not to use a store:**

- Screen-local data (loading, error, form values) → `useState`
- Data the screen fetches for itself → `useState` + service
- Data passed between screens via navigation → navigator params. Params identify and orient — they do not transport state. Passing `id`, `type`, or fields that define the behavior of the next screen is correct. Passing a complete domain object via params is prohibited: it disappears on reload and creates implicit coupling between screens.

**Cache stores:** when a screen identifies, by business rule, that fetching data on every access is too expensive, it may require a dedicated cache store. This store lives in `store/` like any other and is responsible for holding the data and controlling when to invalidate (by time or user action). Creating a cache store for every list is not a rule — it is a case-by-case decision based on real need.

General rules:

- Each store has an explicit interface with named getters and setters
- Stores do not call `httpClient` directly — all API communication is delegated to services
- `AsyncStorage` is accessed exclusively by stores — never by screens, services, or contexts. Each store is responsible for its own persistence keys.
- Stores may handle errors within their domain of responsibility — for example, validating data before storing it and throwing if invalid. What a store never does is silently discard an error outside its domain. Errors the store cannot resolve are always propagated to the caller. The final decision about what to do with an error belongs to the screen or `AuthContext`.

```typescript
interface AccountStore {
  account: Account | null;
  setAccount: (account: Account) => void;
  clearAccount: () => void;
}
```

#### AuthContext (`contexts/AuthContext.tsx`)

The `AuthContext` is the session guardian — it contains no business logic. Its only responsibility is to observe whether there is a valid session and act when there is not.

Flow on app initialization:

```
AuthContext calls authStore.getToken()
  → authStore checks memory → not found → checks AsyncStorage
    → not found → returns null → AppNavigator renders PublicStack
    → found     → saves in memory → returns token
                  → AppNavigator renders PrivateStack
                    → PrivatePostLoginLoadingScreen calls authStore.bootstrap()
                      → success → navigates to PrivateHomeScreen
                      → failure → reportCriticalError(err)
                                  → PublicErrorBootstrapScreen
```

Responsibilities of `AuthContext`:

- Call `authStore.getToken()` on initialization — the token is resolved from memory or `AsyncStorage` by the store itself, without `AuthContext` knowing the persistence details
- Register the 401 handler in `httpClient` — on receiving 401, `AuthContext` calls `authStore.clearAuth()` and navigates to `PublicAuthLoginScreen`. `clearAuth()` is responsible for clearing the token and triggering the `clear()` of each domain store internally.
- Observe the token in `authStore` — `AppNavigator` reacts and renders `PublicStack` or `PrivateStack` based on the state

**`authStore.clearAuth()` is called in two contexts:**

- **`AuthContext`** — in response to a 401 (session expired by the server)
- **`PublicAuthLogoutScreen`** — in response to an explicit logout by the user

No other service, hook, or screen calls `clearAuth()` directly.

The `AuthContext` **never**:

- Calls services directly
- Contains authentication business rules
- Manages tokens or user data — that is the responsibility of stores

#### Internationalization (i18n)

Support for multiple languages is implemented as a store — never as a Context. A `languageStore` holds the current language and the loaded translations. Any component reads from the store. Changing the language updates all subscribers via Zustand.

**Stack:**

- **`i18next`** — i18n engine, manages current language and translation loading
- **`react-i18next`** — `useTranslation()` hook for React Native components
- **`zod-i18n-map`** — integrates zod with i18next, translates validation messages automatically

**File structure:**

```
src/
└── locales/
    ├── pt-BR.json   ← default language
    └── es.json      ← add when needed
```

Key organization by domain:

```json
{
  "validation": {
    "required": "Required field.",
    "email": "Invalid email.",
    "min": "Minimum {{count}} characters."
  },
  "action": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "add": "Add"
  }
}
```

**Usage in components:**

```typescript
const { t } = useTranslation()

<TextInput label={t('entity.name')} />
<Button>{t('action.save')}</Button>
```

**Mandatory rule:** no user-visible text is hardcoded. Every string defined in the frontend goes through `t()`. This includes: labels, placeholders, screen titles, button text, empty state messages, feedback toasts.

**Content from the backend — case by case:**

- Backend sent an **error code** (e.g., `"USER_NOT_FOUND"`) → pass through `t()`, translating the code into a readable message
- Backend sent **domain data** (user name, product title, description) → display directly, without `t()`. These are user-inserted values, not system constants — it does not make sense to translate them.

**Integration with `languageStore`:**

```
authStore.bootstrap()
  → languageStore.bootstrap()  ← loads saved language + calls i18next.changeLanguage()
  → accountStore.bootstrap()
  → ...
```

**i18n is global state — goes in a store, not a Context.**

#### On Contexts in general

Context is the React mechanism for running reactive effects at the root of the component tree — things that need to run before any screen is rendered. **It is not a substitute for global state** — Zustand exists for that.

In practice, in a project with Zustand, you almost never need to create a second Context. Everything that seems to need a Context — language, theme, permissions, user data — is solved with a store.

Contexts that appear in the app tree but **are not written by the project** are library Providers (`PaperProvider`, `NavigationContainer`, etc.) — they do not count as project Contexts.

**Before creating a new Context, answer:** does this need reactive effects integrated into the React lifecycle, or is it just global state? If it is global state, it is a store.

---

### Navigation (`navigation/`)

The navigation tree uses a **single `NavigationContainer`**. The `AppNavigator` is the root navigator inside it and decides which area to render based on the token. The separation between `Public` and `Private` is done by conditional stacks — not by separate containers.

```
NavigationContainer
└── AppNavigator
    ├── PublicStack              ← no token (navigation/public/)
    │   ├── PublicAuthStack
    │   │   ├── PublicAuthLoginScreen
    │   │   ├── PublicAuthRegisterScreen
    │   │   ├── PublicAuthLogoutScreen
    │   │   └── ...
    │   └── PublicErrorStack     ← accessible without authentication
    │       ├── PublicErrorBootstrapScreen
    │       └── PublicErrorGenericScreen
    └── PrivateStack             ← with token (navigation/private/)
        ├── PrivatePostLoginLoadingScreen
        └── PrivateMainTabs
            ├── Private<Domain>Stack  ← one per functional area
            └── ...
```

**Error screens inside `PublicStack`.** Critical errors do not require authentication to be displayed — they live in `PublicErrorStack`, inside `PublicStack`. Any screen (public or private) navigates to them with `navigate('PublicErrorGenericScreen')`. React Navigation finds the screen by going up and down the hierarchy automatically. Placing them in `public/error/` forces the question: "can this data be exposed without authentication?"

**Unified navigation history.** The history is single and linear — `goBack()` works freely between different tab stacks. If the user was on `PrivateAdminUsersScreen`, navigated to Home, and pressed back, they return to `PrivateAdminUsersScreen`. This is the correct and expected behavior.

**Double-tap on tab resets the stack.** Tapping an already-active tab pops all screens of that stack and returns to the root screen. It is the user's explicit action to "start over" in that area.

```typescript
// in each Screen inside the tab navigator:
listeners={({ navigation }) => ({
  tabPress: () => {
    if (navigation.isFocused()) {
      navigation.dispatch(StackActions.popToTop())
    }
  },
})}
```

Rules:

- `AppNavigator` contains no logic — it only reads the token from `authStore` and renders `PublicStack` or `PrivateStack`
- Every new screen is declared in exactly one stack, mirroring its folder in `screens/`
- With token but stores still empty → `PrivatePostLoginLoadingScreen` runs the bootstrap before showing the app
- Each functional area has its own stack inside `PrivateMainTabs`
- Root screens of each tab never have a back button
- **Screen and stack names encapsulate the access hierarchy** — the name is self-documented and eliminates ambiguity between screens from different domains (see Conventions section)

---

### Theme

The central theme lives in `src/theme/` and is the **single source of truth** for colors, typography, and spacing of the design system. It extends the theme of the installed UI library — read `package.json` to identify which one it is.

**Never hardcode visual values in components or screens.** Colors, font sizes, and spacings always come from the theme.

```typescript
// wrong
color: "#6750A4";
backgroundColor: "#F5F5F5";
fontSize: 16;
padding: 12;

// correct
color: theme.colors.primary;
backgroundColor: theme.colors.background;
fontSize: theme.fonts.bodyMedium.fontSize; // or equivalent from installed lib
padding: Spacing.md;
```

When you need a value that does not exist in the theme, add it to the theme — not inline in the component.

---

### Config (`config/index.ts`)

Environment values that change between builds (URLs, timeouts, feature flags) live in a single file `src/config/index.ts`. No other file accesses environment variables directly.

```typescript
// src/config/index.ts
export const API_BASE_URL =
  process.env.EXPO_PUBLIC_API_URL ?? "http://localhost:3000";
export const API_TIMEOUT = 10_000;
```

```typescript
// usage
import { API_BASE_URL } from "@/config";
```

If the file grows too large, break it by domain (`config/api.ts`, `config/features.ts`). Start with a single file.

---

### Types (`types/`)

The standard is always to create types in `types/`, with a subdirectory per domain.

**Using `index.ts` as a barrel export is prohibited anywhere in `src/`.** This rule is global — applies to `types/`, `components/`, `hooks/`, `services/`, `utils/`, and other directories. The file path is semantic information: `import { User } from '@/types/user/user.model'` says it is the domain model. `import { User } from '@/types/user/user.api'` says it is an API contract type. An `index.ts` that re-exports everything erases this information, hinders tree-shaking, and increases the risk of circular imports.

```
types/
├── navigation/
│   ├── publicStack.ts        ← PublicStack ParamList
│   ├── privateStack.ts       ← PrivateStack ParamList
│   ├── mainTabs.ts           ← MainTabs ParamList
│   └── adminStack.ts         ← ParamList for each domain stack
├── user/
│   ├── user.model.ts         ← domain model (what the app uses internally)
│   └── user.api.ts           ← API request/response types
├── event/
│   ├── event.model.ts
│   └── event.api.ts
└── organization/
    └── organization.model.ts
```

Navigation types live in `types/navigation/`, one file per stack navigator. Never inside `navigation/` — types belong in `types/`.

If the domain is simple and has few types, a single file per domain is sufficient — break into files as the domain grows.

**Local type is the exception, not the rule.** Use local types only when truly internal to a single file and with no usefulness outside it — for example, an auxiliary type for parameter passing between two private functions within the same service.

```typescript
// local — correct, will never be used outside this file
type ParsedResponse = { id: string; raw: unknown }
function parse(raw: unknown): ParsedResponse { ... }
function transform(parsed: ParsedResponse) { ... }
```

If there is any chance the type will be used in another file — screen, store, service, component — it goes into `types/`.

---

### Hooks (`hooks/`)

Hooks are functions that use React hooks internally (`useState`, `useEffect`, `useRef`, etc.). They are the place for reusable stateful logic between screens or components.

```
hooks/
├── commons/          ← generic hooks with no specific domain
│   └── modal/
│       └── useGenericModal.ts
└── <domain>/         ← domain-specific hooks
    └── useEventsFilter.ts
```

**When to create a hook:** when the same stateful logic appears in more than one place. If the logic exists in only one screen, it stays in the screen.

**Hooks can receive parameters** — including stores — making them reusable adapters between state and component.

**Hooks are not `utils/`.** If the function does not use any React hook, it is a pure function and goes into `utils/`.

---

### Utils (`utils/`)

Utils are pure functions without React hooks — formatting, validation, parsing, calculations. They can be used in any context, including outside React.

```
utils/
├── dateTime/
│   ├── formatter.ts   ← functions that format dates
│   └── validator.ts   ← functions that validate dates
└── string/
    ├── formatter.ts   ← functions that format strings
    └── parser.ts      ← functions that parse strings
```

The subdirectory defines the domain. The file defines the operation (`formatter`, `validator`, `parser`, `calculator`). Together they are self-explanatory without repetition.

**If you need a hook inside, it is not a util — it is a hook.**

---

## S3 — Implementation

> Read this section only when implementing or maintaining: auth, session, bootstrap, login flow, critical error screens.

### authStore (`store/authStore.ts`)

The `authStore` is the session repository and the orchestrator of authentication and store initialization.

Responsibilities:

- Hold and expose the JWT token in memory
- Implement `getToken()`: checks memory first — if not found, fetches from `AsyncStorage`, saves in memory, and returns. If not found in either, returns `null`.
- Implement `login(credentials)`: calls `authService.login()`, persists the token in `AsyncStorage` and in memory, returns success or propagates the exception for the screen to handle
- Implement `bootstrap()`: calls `bootstrap()` on each store that needs to be initialized — called by `PrivatePostLoginLoadingScreen`
- Clear the token from `AsyncStorage` and memory, and trigger `clear()` on other stores on logout

**Login flow:**

```
PublicAuthLoginScreen — user submits form
  → authStore.login(credentials)
    → authService.login(credentials)
      → success → token saved in AsyncStorage and authStore
        → authStore returns success to PublicAuthLoginScreen
          → PublicAuthLoginScreen navigates to PrivatePostLoginLoadingScreen
            → PrivatePostLoginLoadingScreen calls authStore.bootstrap()
              → success → navigates to PrivateHomeScreen
              → UNAUTHORIZED failure → ignore — AuthContext 401 handler
                                       already cleared the session and navigated to
                                       PublicAuthLoginScreen
              → other failure → reportCriticalError(err) → PublicErrorBootstrapScreen
      → failure → exception propagated → PublicAuthLoginScreen handles (inline feedback)
```

---

### Domain stores

Each domain store (e.g., `accountStore`, `personStore`) is self-sufficient in its initialization.

Responsibilities:

- Hold and expose its domain data
- Implement `bootstrap()`: calls its own service to fetch and populate the data
- Implement `clear()`: clears the data on logout

```typescript
// standard interface for initializable stores
interface DomainStore {
  bootstrap(): Promise<void>;
  clear(): void;
}
```

`authStore.bootstrap()` knows which stores need to be initialized and calls the `bootstrap()` of each in sequence. Each store decides how to populate itself — `authStore` does not know the details.

```typescript
// example — the exact list depends on the project
async bootstrap() {
  await languageStore.bootstrap()
  await accountStore.bootstrap()
  await personStore.bootstrap()
}
```

**Rule:** every store that needs to be initialized at login **must** be explicitly added to `authStore.bootstrap()`. This list is the source of truth for app initialization dependencies. Creating a store with `bootstrap()` without adding it here is a bug — the store will start empty and the screen will break silently.

---

### Critical errors

The screen always has the error in hand — it captures the exception in `try/catch` and decides what to do with it:

| Type | When | Where to handle |
| --- | --- | --- |
| **Expected error** | The screen recognizes the error code and knows how to react — 404, 403, validation failure, empty list | `useState` on the screen itself, inline feedback. The user continues navigating. |
| **Unexpected error** | The screen does not recognize the error — runtime exception, invalid state, error with no known code | `useReportCriticalError()` → `errorStore.setError(error)` + `navigate('PublicErrorGenericScreen')` |

The list of error codes each screen handles on its own is a project decision — defined in the `30+` file. What this document establishes is the principle: recognized errors stay in the screen; unrecognized errors go to `errorStore`.

The project has predefined error screens in `screens/public/error/`, each with specific responsibility and recovery action. The project can add its own error screens in this directory as needed.

**`PublicErrorBootstrapScreen`** — exclusive for session bootstrap failures. Displays the error and offers a retry button that navigates to `PrivatePostLoginLoadingScreen` — without calling any store directly.

```
PrivatePostLoginLoadingScreen calls authStore.bootstrap()
  → failure → reportCriticalError(err)
              → errorStore.setError(err)
              → navigate('PublicErrorBootstrapScreen')
                → user triggers retry → navigate('PrivatePostLoginLoadingScreen')
                  → PrivatePostLoginLoadingScreen calls authStore.bootstrap() again
                    → success → navigates to PrivateHomeScreen
                    → failure → reportCriticalError(err) → PublicErrorBootstrapScreen again
```

**`PublicErrorGenericScreen`** — general error screen, used by any screen in the app for unexpected errors not covered by a specific screen. The recovery action is defined by context — typically go back to the start or try again.

The pattern for unexpected errors in screens is always the same:

```
screen captures unrecognized exception in catch
  → useReportCriticalError(err)
    → errorStore.setError(err)
    → navigation.navigate('PublicErrorGenericScreen')
      → screen reads the error from errorStore
      → displays message and recovery action
      → recovery action: errorStore.clearError() → navigates back
```

To avoid repetition, a hook encapsulates both calls:

```typescript
// hooks/commons/useReportCriticalError.ts
export function useReportCriticalError() {
  const navigation = useNavigation();
  return (err: unknown) => {
    errorStore.setError(err);
    navigation.navigate("PublicErrorGenericScreen");
  };
}
```

**`errorStore`** is the communication channel between whoever detects the error and the error screen. Do not pass the error via navigation params — that would violate the rule of params as identifiers.

**What every error screen must do:**

- Inform that the system encountered an error and guide the user (try again or contact support)
- Offer the recovery action appropriate to the context
- Display technical error details (code, service, timestamp) to support the support team
- **Never expose sensitive data** — JWT token, passwords, and personal data (name, email, ID) never appear

---

## S4 — Interaction Patterns

> Read this section when implementing any screen with loading, error, empty list, or form.

---

### Loading

The project has two loading contexts with distinct behaviors.

#### Local loading — initial data fetch

**Screens with form or single content** — displays a centered `ActivityIndicator` while waiting.

```typescript
return loading ? <ActivityIndicator style={{ flex: 1 }} /> : <content />
```

**Screens with list** — displays `SKELETON_COUNT` repetitions of `SkeletonCard` while waiting. The skeleton is a light gray card with a shimmer animation — a single component, independent of the list item type.

```typescript
return loading ? (
  <>
    {Array.from({ length: SKELETON_COUNT }).map((_, i) => (
      <SkeletonCard key={i} />
    ))}
  </>
) : <FlatList ... />
```

`SKELETON_COUNT` comes from `config`:

```typescript
export const SKELETON_COUNT = Number(process.env.EXPO_PUBLIC_SKELETON_COUNT ?? 5)
```

`SkeletonCard` lives in `atom/feedback/SkeletonCard.tsx`.

#### Global overlay — user actions

Used when the user executes an action that calls the API (save, subscribe, delete, login, logout, etc.). Blocks the entire screen via `loadingStore`, preventing double-click and accidental navigation during the operation.

**Rule:** every `async` operation triggered by the user — via `service` or via `async store action` — uses the global overlay. No exceptions.

Synchronous store operations (`setUser`, `clearError`, `setError`) **do not** use the overlay — they are instantaneous.

The `LoadingOverlay` is a transparent `Modal` with an `ActivityIndicator` rendered at the root of the app, above everything (tabs, header, navigation).

---

### Errors

The project has five error types with distinct presentations.

#### `showToast` — mandatory wrapper

Every call to toast goes through `showToast` in `utils/toast/toast.ts`. Never call the toast library directly. Defaults live in a single place.

```typescript
// utils/toast/toast.ts
import { TOAST_VISIBILITY_MS } from '@/config'

export function showToast(params: ToastShowParams) {
  Toast.show({
    position: 'top',
    visibilityTime: TOAST_VISIBILITY_MS,
    ...params,
  })
}
```

The display time is configurable in `src/config`:

```typescript
// src/config/index.ts
export const TOAST_VISIBILITY_MS = Number(
  process.env.EXPO_PUBLIC_TOAST_VISIBILITY_MS ?? 30_000
)
```

#### Default toast types

Three types are defined once in the `react-native-toast-message` configuration:

| Type | Icon | When to use |
|---|---|---|
| `error` | error/alert icon | API business error |
| `networkError` | network/wifi icon | No connection, timeout |
| `success` | check icon | Action completed successfully |

All follow the same behavior: top of the screen, `TOAST_VISIBILITY_MS` to disappear, swipe to close, tap pauses the counter.

#### Type 1 — Form validation

Appears inline, directly below the field with the problem.

- On leaving the field (`onBlur`) — validates only the field that lost focus
- An untouched field never displays an error, even if required
- On submit — validates all fields, including untouched ones

```typescript
<TextInput
  error={touched.name && !!errors.name}
  onBlur={() => setTouched(t => ({ ...t, name: true }))}
/>
{touched.name && errors.name && (
  <HelperText type="error">{errors.name}</HelperText>
)}
```

If the error can be mapped to a specific field (e.g., "email already registered" from the API), treat it as form validation — appears inline in the corresponding field.

#### Type 2 — API business error

The API processed the request and returned a business rule error that cannot be attributed to a specific field.

**How it appears:** floating banner at the top of the screen via toast, overlaid on the content without shifting the layout. Disappears automatically, but can be closed by swipe. Tapping pauses the counter.

```typescript
showToast({ type: 'error', text1: t('error.title'), text2: err.message })
```

#### Type 3 — No connection / timeout

The request did not reach the server. Uses toast with a network-specific type (defined in `30+`).

The `isNetworkError` function lives in `utils/error/classifier.ts` and is the only one responsible for distinguishing network errors from business errors.

#### Type 4 — Expired session (401)

Intercepted by `httpClient` and handled exclusively by `AuthContext` — no screen handles 401 directly.

```
httpClient receives 401 → AuthContext handler → showToast → authStore.clearAuth() → AppNavigator redirects to login
```

#### Type 5 — Critical / unexpected error

Last resort from `catch` — not a network error, not a business error, not anticipated. Goes to `errorStore` + `PublicErrorGenericScreen` via `useReportCriticalError()`.

**Decision hierarchy in `catch`:**

```typescript
} catch (err) {
  if (isNetworkError(err)) {
    showToast({ type: 'networkError', ... })
  } else if (err instanceof ApiError) {
    showToast({ type: 'error', text2: err.message, ... })
  } else {
    reportCriticalError(err)
  }
}
```

**Rule:** every screen `catch` follows this hierarchy. Never silence an error with an empty `catch (err) {}`.

#### Action hooks — `useErrorHandler` and `useAsyncAction`

To avoid repeating the same loading + error block in every screen, two hooks encapsulate these responsibilities.

**`useErrorHandler`** — classifies and presents the error automatically. Lives in `hooks/commons/useErrorHandler.ts`.

```typescript
export function useErrorHandler() {
  const reportCritical = useReportCriticalError()
  const { t } = useTranslation()

  return (err: unknown) => {
    if (isNetworkError(err)) {
      showToast({ type: 'networkError', text1: t('error.network.title'), text2: t('error.network.message') })
    } else if (err instanceof ApiError) {
      showToast({ type: 'error', text1: t('error.title'), text2: err.message })
    } else {
      reportCritical(err)
    }
  }
}
```

**`useAsyncAction`** — wraps any async action with a loading overlay and error handling. Lives in `hooks/commons/useAsyncAction.ts`.

```typescript
interface AsyncActionOptions {
  onSuccess?: () => void
  onError?: (err: unknown) => boolean | void
  overlay?: boolean  // default: true — false for batch action screens
}

export function useAsyncAction() {
  const handleError = useErrorHandler()

  return async (action: () => Promise<void>, options?: AsyncActionOptions) => {
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

`onError` returns `true` if it handled the error — the default handler is not called. Without a return, falls through to the handler automatically.

**Batch action with `overlay: false`** — for screens with multiple sequential actions (e.g., marking attendance), disable the overlay and manage loading locally per item:

```typescript
const [loadingId, setLoadingId] = useState<string | null>(null)

const handleMark = (id: string) => runAction(
  async () => { setLoadingId(id); await service.mark(id) },
  {
    overlay: false,
    onSuccess: () => { setLoadingId(null); scheduleRefresh() },
    onError: () => { setLoadingId(null); return false },
  }
)
```

**Debounce to reload list** — after batch actions, the list reloads when the user stops interacting. Configurable time in `src/config`:

```typescript
export const BATCH_ACTION_DEBOUNCE_MS = Number(
  process.env.EXPO_PUBLIC_BATCH_DEBOUNCE_MS ?? 5_000
)
```

```typescript
const refreshTimer = useRef<ReturnType<typeof setTimeout> | null>(null)

const scheduleRefresh = () => {
  if (refreshTimer.current) clearTimeout(refreshTimer.current)
  refreshTimer.current = setTimeout(loadList, BATCH_ACTION_DEBOUNCE_MS)
}
```

---

### Paginated lists

The pagination model is always cursor-based. Which of the two cases below to use is dictated by the business rule — not by technical preference.

---

#### Case 1 — Full scan: load everything, handle on the frontend

Used when the operation requires access to the complete data set — arbitrary sorting, cross-filters, multiple selection, export. The frontend fetches all records in batches via cursor until the end and applies sorting, filtering, and display locally.

Examples: administer all users, manage product catalog.

```typescript
const [items, setItems] = useState<Item[]>([])
const [isLoading, setIsLoading] = useState(false)

async function loadAll(): Promise<void> {
  setIsLoading(true)
  const all: Item[] = []
  let cursor: string | null = null

  try {
    do {
      const res = await itemService.list({ cursor: cursor ?? undefined, pageSize: PAGE_SIZE })
      all.push(...res.data)
      cursor = res.pagination.nextCursor
    } while (cursor)
    setItems(all)
  } finally {
    setIsLoading(false)
  }
}
```

Filter, sorting, and search are done on `items` with `useMemo` — without a new API call.

> The viable volume limit for Case 1 depends on the project, the screen, and the business rule. When the volume compromises frontend performance, the filter must migrate to the backend — a decision documented in the domain's `30+` file.

---

#### Case 2 — Continuation: load on demand

Used when the data has a natural order (most recent, by creation date) and the user advances as needed. The frontend fetches the first batch and displays a "Load more" button while there is a cursor. Infinite scroll is not used — the user explicitly controls when to fetch more items.

Examples: list the 50 most recent events, show products by creation order.

```typescript
const [items, setItems] = useState<Item[]>([])
const [nextCursor, setNextCursor] = useState<string | null>(null)
const [loadingMore, setLoadingMore] = useState(false)
const runAction = useAsyncAction()

const loadMore = () => runAction(
  async () => {
    setLoadingMore(true)
    const res = await itemService.list({ cursor: nextCursor ?? undefined, pageSize: PAGE_SIZE })
    setItems(prev => [...prev, ...res.data])
    setNextCursor(res.pagination.nextCursor)
  },
  { overlay: false, onSuccess: () => setLoadingMore(false), onError: () => { setLoadingMore(false); return false } }
)

const handleRefresh = () => {
  setItems([])
  setNextCursor(null)
  loadInitial()
}

return (
  <FlatList
    data={items}
    keyExtractor={item => item.id}
    renderItem={({ item }) => <ItemCard item={item} />}
    onRefresh={handleRefresh}
    refreshing={refreshing}
    ListFooterComponent={
      nextCursor ? (
        <Button loading={loadingMore} onPress={loadMore}>
          {t('action.loadMore')}
        </Button>
      ) : null
    }
    ListEmptyComponent={!isLoading ? <EmptyState ... /> : null}
  />
)
```

---

The batch size comes from `config`:

```typescript
// src/config/index.ts
export const PAGE_SIZE = Number(process.env.EXPO_PUBLIC_PAGE_SIZE ?? 20)
```

> This document does not cover WebSockets, SSE, or any real-time update mechanism. Projects that require these features must document them in a `30+` file.

---

### Empty States

When a list has no items, the `EmptyState` component replaces the content.

- Icon + contextual message defined by the screen — never generic (`"No items"`)
- Centered vertically and horizontally on the screen
- Discreet icon — small, above the message
- Optional action — the screen passes `action` only when it makes sense for the business

```typescript
// atom/feedback/EmptyState.tsx
interface EmptyStateProps {
  icon: string     // icon name (e.g., MaterialCommunityIcons)
  message: string  // contextual message — always via t()
  action?: {
    label: string
    onPress: () => void
  }
}
```

**FAB** is a separate pattern from `EmptyState`. It appears on screens that allow addition, regardless of whether there are items or not.

Rules:
- The icon varies according to the screen action — it is not always `plus`
- FAB and header button are both valid — the screen decides based on context
- The FAB never covers content: the `FlatList` receives `contentContainerStyle={{ paddingBottom: FAB_HEIGHT }}` so the last item has space above the button
- Number of FABs per screen: ask the human — specific decision for each screen

---

### Modals vs Navigation

**Rule: prefer navigation over modal.** Any interaction that would require a modal — form, detail, data confirmation — becomes a new screen with `navigate`.

A modal is only justified when it is impossible to resolve with navigation — when blocking the originating screen is an essential part of the experience, not a convenience. When in doubt, it is a new screen.

---

### Screen layouts

Every screen uses one of the layouts below as a structural skeleton. Never assemble the scroll, keyboard, and footer structure directly in the screen — the layout encapsulates this.

#### `ListScreenLayout`

For screens with data lists. Manages the `FlatList` with pull-to-refresh, skeleton, "load more", and optional FAB.

```
SafeAreaView
└── FlatList (pull-to-refresh, skeleton, ListFooterComponent)
FAB (absolute, bottom right corner — optional)
```

```typescript
// layouts/list/ListScreenLayout.tsx
interface ListScreenLayoutProps<T> {
  data: T[]
  loading: boolean
  nextCursor: string | null
  loadingMore: boolean
  onRefresh: () => void
  onLoadMore: () => void
  renderItem: (item: T) => React.ReactElement
  keyExtractor: (item: T) => string
  emptyState: React.ReactElement
  fab?: { icon: string; onPress: () => void }
}
```

#### `FormScreenLayout`

For screens with forms. Manages `KeyboardAvoidingView`, field scroll, and a fixed footer with the save button.

```
KeyboardAvoidingView
└── ScrollView (form fields)
Fixed footer (save button — outside scroll)
```

```typescript
// layouts/forms/FormScreenLayout.tsx
interface FormScreenLayoutProps {
  children: React.ReactNode        // form fields
  onSave: () => void
  saveLabel?: string               // default: t('action.save')
  saving?: boolean                 // loading on button
  saveDisabled?: boolean           // disables the button
}
```

```typescript
// usage in screen
return (
  <FormScreenLayout
    onSave={handleSave}
    saveDisabled={Object.keys(errors).length > 0}
  >
    <TextInput label={t('person.name')} ... />
    <TextInput label={t('person.email')} ... />
  </FormScreenLayout>
)
```

**`KeyboardAvoidingView`:** `behavior="padding"` on iOS, no behavior on Android — Android adjusts via `windowSoftInputMode` in `app.json`.

---

### Forms

#### Validation stack

- **`react-hook-form`** — manages form values, errors, and state
- **`zod`** — defines the validation schema with automatically generated TypeScript types
- **`zod-i18n-map`** — integrates zod with the project's i18n system to translate validation messages

#### Validation behavior

- **onBlur** — when leaving a field, validates that field. Untouched fields do not display errors.
- **On submit** — validates all fields, including untouched ones.
- **Submit button** — enabled in the initial state. Disabled when there are visible errors: `disabled={Object.keys(errors).length > 0}`.

#### `useFormAction`

Hook that encapsulates the complete form cycle: validation, dirty check, loading overlay, and error handling. Used in **all forms** — creation and editing.

```typescript
// hooks/commons/useFormAction.ts
export function useFormAction<T>(form: UseFormReturn<T>) {
  const runAction = useAsyncAction()
  const navigation = useNavigation()
  const { t } = useTranslation()
  const errorCount = useRef(0)
  const { isDirty } = form.formState  // lido no render para o RHF subscrever a propriedade

  return (action: (data: T) => Promise<void>, options?: AsyncActionOptions) => {
    const submitHandler = form.handleSubmit(
      async (data) => {
        errorCount.current = 0
        await runAction(() => action(data), options)
      },
      () => {
        errorCount.current += 1
        if (errorCount.current >= 3) {
          errorCount.current = 0
          showToast({ type: 'error', text1: t('form.hasErrors.title'), text2: t('form.hasErrors.message') })
        }
      }
    )
    return (e?: unknown) => {
      if (!isDirty) {
        navigation.goBack()  // nothing changed — go back without validating or calling the API
        return
      }
      return submitHandler(e as any)
    }
  }
}
```

**Internal flow:**
1. If nothing changed (`isDirty = false`) — navigates back immediately, **before** validation runs
2. Validates all fields — marks untouched ones with error if invalid
3. If invalid 3 times in a row — displays a toast guiding the user to check the fields (resolves fields outside the visible area on small screens)
4. Executes the action via `useAsyncAction` — overlay + error handling

**Why `isDirty` is checked before `handleSubmit`:** In edit mode, forms often have optional fields (e.g. `createAccount`, `email`, `password`) that are only relevant for create mode and are reset to `""` or `undefined` for editing. Running Zod validation on these before the dirty check would produce spurious errors. Checking `isDirty` first avoids validation entirely when nothing changed.

**`isDirty` subscription:** `{ isDirty }` must be destructured from `form.formState` in the hook body (render phase), not inside the callback. RHF uses a Proxy that only tracks properties read during render — reading inside a callback bypasses this and always returns `false`.

**`setValue` + `isDirty`:** `setValue` does not mark the form as dirty by default. Calls triggered by user interaction (e.g. selecting from a list, RadioButton) must pass `{ shouldDirty: true }`. Calls that load initial data (e.g. in a `useEffect` for edit mode) must NOT pass `shouldDirty: true`.

#### `useDestructiveAction`

Hook for irreversible actions. Displays a confirmation dialog, waits for response, executes only on OK.

**Default rule: every irreversible action uses `useDestructiveAction`.** The exception is when a human expresses that the dialog should be removed — in that case, document with an inline comment in the code.

```typescript
// hooks/commons/useDestructiveAction.ts
export function useDestructiveAction() {
  const runAction = useAsyncAction()
  const { t } = useTranslation()
  const [visible, setVisible] = useState(false)
  const [dialog, setDialog] = useState<{ title: string; message: string } | null>(null)
  const pendingAction = useRef<(() => void) | null>(null)

  const destructiveAction = (
    action: () => Promise<void>,
    options?: AsyncActionOptions & { title?: string; message?: string }
  ) => {
    setDialog({
      title: options?.title ?? t('dialog.destructive.title'),
      message: options?.message ?? t('dialog.destructive.message'),
    })
    pendingAction.current = () => runAction(action, options)
    setVisible(true)
  }

  const handleConfirm = () => { setVisible(false); pendingAction.current?.() }

  const ConfirmDialog = () => (
    <Dialog visible={visible} onDismiss={() => setVisible(false)}>
      <Dialog.Title>{dialog?.title}</Dialog.Title>
      <Dialog.Content><Text>{dialog?.message}</Text></Dialog.Content>
      <Dialog.Actions>
        <Button onPress={() => setVisible(false)}>{t('action.cancel')}</Button>
        <Button onPress={handleConfirm}>{t('action.confirm')}</Button>
      </Dialog.Actions>
    </Dialog>
  )

  return { destructiveAction, ConfirmDialog }
}
```

The dialog message is passed by the screen — contextual and specific, never generic:

```typescript
const { destructiveAction, ConfirmDialog } = useDestructiveAction()

const handleDelete = () => destructiveAction(
  async () => { await service.delete(id); showToast(...); navigation.goBack() },
  { message: t('entity.deleteConfirm', { name: entity.name }) }
)

return (
  <>
    <Button onPress={handleDelete}>{t('action.delete')}</Button>
    <ConfirmDialog />
  </>
)
```

#### Separation of handlers and JSX

Handlers are defined as `const` before the `return` — never as inline functions in JSX.

```typescript
export function SomeScreen() {
  const form = useForm<FormData>({ defaultValues: entity })
  const runFormAction = useFormAction(form)
  const runAction = useAsyncAction()

  const handleSave = runFormAction(async (data) => {
    await service.update(id, data)
    showToast({ type: 'success', ... })
    navigation.goBack()
  })

  const handleDelete = () => runAction(async () => {
    await service.delete(id)
    navigation.goBack()
  })

  return (
    <>
      <TextInput label={t('entity.name')} {...form.register('name')} />
      <Button onPress={handleSave}>{t('action.save')}</Button>
      <Button onPress={handleDelete}>{t('action.delete')}</Button>
    </>
  )
}
```

Inline functions in `onPress` are only allowed for trivial cases: `onPress={() => navigation.goBack()}`.

#### Success feedback

Every action that completes successfully displays a `success` toast — no exceptions. Even when the screen navigates back, the toast appears over the destination screen.

**Navigation rule after success:**
- Forms (creation and editing) → toast + `goBack()`
- Single-purpose actions (delete, confirm) → toast + `goBack()`
- Batch actions (mark attendance, list toggles) → toast per action + debounce to reload list

#### Required fields

Indicated with an asterisk after the label: `Name *`. A discreet legend at the top of the form explains the asterisk — once, not on every field.

```typescript
<Text variant="bodySmall">{t('form.requiredFieldsNote')}</Text>
<TextInput label={`${t('entity.name')} *`} ... />  // required
<TextInput label={t('entity.phone')} ... />          // optional
```

**When to use each hook:**

| Hook | When to use |
|---|---|
| `useFormAction` | Every form — creation and editing |
| `useAsyncAction` | Actions without form and without confirmation |
| `useDestructiveAction` | Irreversible actions that require confirmation |
