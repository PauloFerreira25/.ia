---
name: new-monorepo
description: "Interview the human to define a new monorepo structure. Spec first, validate, then implement."
when_to_use: "When the human asks to create a new monorepo or set up a new multi-package repository."
---

# Playbook 01 — New Monorepo

> **Reference document.** This playbook cannot predict every project combination or every answer a human might give. Parts of it will need to be adapted to the situation at hand. What matters most: always present the final result to the human before creating anything, so they can make informed decisions. When in doubt, show — don't assume.

**This playbook is an interview.** Its purpose is to collect all decisions about the monorepo through conversation with the human, document every answer in `docs/monorepo-architecture.md`, and only then create anything.

The interview produces the document. The document drives the implementation. Nothing is created before both are confirmed.

Do not skip phases. Do not create files before receiving human confirmation.

**Language:** Conduct the interview in whatever language the human uses. All names (services, directories, packages) must be in English unless the human explicitly requests otherwise. If the human provides a name in another language, translate it to English before recording. Example: "backend maçã" becomes `backend/apple/`. Exception: if the human explicitly says "keep it in Portuguese" or similar, respect that choice.

**Flow over options.** Every technology name in this document is an example — it illustrates the kind of answer to expect, not a menu to present. Ask open-ended questions and record whatever the human says. Never turn examples into multiple-choice options. The flow is what matters, not the technologies listed.

**Scope boundary.** This playbook owns only its own flow. Every rule about how to structure a specific technology, tool, or environment lives in the referenced `.ia/architecture/` documents — not here. When a technology is identified during the interview, reference the relevant document. Do not copy, summarize, or re-derive its rules inside this playbook.

**Interview complement.** Use the [grill-me skill](https://raw.githubusercontent.com/mattpocock/skills/refs/heads/main/skills/productivity/grill-me/SKILL.md) as a companion during Phase 2. After each section, present on screen a clear summary of what has been defined and what still remains open — so the human always knows where they are in the process.

**This is a single-run playbook.** It runs once to lay the foundation — like terraforming. Once the structure is created, this playbook's job is done. If the project evolves and diverges from what this playbook created, that is expected and fine. Do not re-run this playbook to reconcile drift.

---

## Fundamental principle

This is a monorepo for convenience — not for coupling. Every service must be independently runnable, deployable, and containerizable (Docker). A service must never depend on another service's directory or files to run.

Consequences:
- Each service has its own `.env` file. Never create a `.env` at the monorepo root.
- `shared-libs/` are installed via `file:` references — each service pulls what it needs independently.
- A service extracted from the monorepo into its own repo must work without changes.
- **Do not use npm workspaces, Turborepo, Nx, or any monorepo tooling.** These tools hoist `node_modules` to the root and create tooling-level coupling, which directly contradicts the independence principle. The `file:` reference approach is the intentional alternative — each service installs shared-libs locally as if they were external packages.

---

## Important notes before starting

**`02-nodejs-typescript.md` is single-repo oriented.** Every rule in that document (folder structure, layer separation, shared-libs, etc.) applies *inside* each service's own directory — not at the monorepo root. Example: `queue/<worker>/` described there becomes `backend/<service>/src/queue/<worker>/` in a monorepo.

**`shared-libs/` is conditional.** Only create `backend/shared-libs/`, `frontend/shared-libs/`, `lambdas/nodejs/shared-libs/`, or `lambdas/python/shared-libs/` when there are multiple services/apps/lambdas in that block. A single service has no sibling to share with.

**Directory naming follows the language convention — and is defined per service during the interview.** Each language has its own naming standard. Do not impose a single convention across the monorepo — each service follows the convention of its own technology. Node.js may use `camelCase`, Python may use `snake_case`, some frameworks prefer `kebab-case`. This cannot be decided at the playbook level. During the interview, when a technology is chosen: follow what its `.ia/architecture/` file specifies. If no file exists, ask the human which convention to follow for that service.

**Architecture file lookup** — see [Architecture File Lookup](#architecture-file-lookup) (runs at the end of Phase 1).

---

## Phase 1 — Pre-requisites

**Step 1 — Check required directories:**

| Item | Source |
|---|---|
| `.devcontainer/` | [github.com/PauloFerreira25/base-devcontainer](https://github.com/PauloFerreira25/base-devcontainer) |
| `.ia/` | [github.com/PauloFerreira25/.ia](https://github.com/PauloFerreira25/.ia) |

- If either is missing → recommend the human clone and set up the missing directory from the repository above before continuing. Do not proceed until both are present.

**Step 2 — Check output files:**

Check whether any of the following files already exist:

- `README.md`
- `docs/index.md`
- `docs/monorepo-architecture.md`
- `task-spec/.gitkeep`

For each file that already exists, ask the human:

> `<file>` already exists. Overwrite or skip?

Wait for the answer before checking the next file. Respect the human's decision for each file individually.

**Step 3** — Run the [Architecture File Lookup](#architecture-file-lookup) below, then proceed to Phase 2.

---

## Architecture File Lookup

Run `ls .ia/architecture/` to get the current list of available architecture files. Do this once — reuse the result throughout the entire interview. Do not re-run `ls` for each section.

Every time Phase 2 references "see Architecture File Lookup", apply this process:

1. Match the chosen technology against the file list.
2. If a match exists → reference that file in `docs/monorepo-architecture.md`.
3. If no match exists → document the technology without a link (the absence of a link is the signal that no architecture file exists). Handle it in Phase 5 via the community standards interview. Do not create a "Gaps" section in the document.

---

## Phase 2 — Interview

Ask one question at a time. Wait for the answer before asking the next.

Every answer collected here becomes a section in `docs/monorepo-architecture.md`. Nothing answered in this phase is lost — all of it is documented before implementation begins.

> **Agent reminder:** All technology names in the sections below are illustrative — they show the kind of answer to expect, not a list to present to the human. Always ask open questions. Never offer options.

### 2.1 Repository purpose

> What is the purpose of this repository?

Used to write `README.md` and the opening section of `docs/monorepo-architecture.md`.

### 2.2 Backend

**Per-backend questions** (repeat for each backend):
1. Name
2. Language
3. Framework (if any)
4. See [Architecture File Lookup](#architecture-file-lookup) for language and framework
5. Will this backend have queue workers / consumers?
   - If yes: which queue technology? (example: `backend/<service>/src/queue/<worker>/`)
   - See [Architecture File Lookup](#architecture-file-lookup) for the queue technology

**Flow:**
> Will this monorepo have a backend?

If no → skip to 2.3.

If yes:
> How many backends?

Repeat the per-backend questions above for each backend.

---

### 2.3 Frontend

**Per-frontend questions** (repeat for each frontend):
1. Name
2. Technology
3. UI library (if any)
4. See [Architecture File Lookup](#architecture-file-lookup) for technology and UI library

**Flow:**
> Will this monorepo have a frontend?

If no → skip to 2.4.

If yes:
> How many frontends?

Repeat the per-frontend questions above for each frontend.

---

### 2.4 Lambdas

**Per-lambda questions** (repeat for each lambda):
1. Name
2. Language (e.g. Node.js, Python, Go)
3. See [Architecture File Lookup](#architecture-file-lookup) for the language

**Flow:**
> Will this monorepo have lambdas?

If no → skip to 2.5.

If yes:
> How many lambdas?

Repeat the per-lambda questions above for each lambda.

### 2.5 npm scope

Only ask this section if at least one Node.js technology was confirmed in 2.2, 2.3, or 2.4 (Node.js backend, frontend, or Node.js lambda). Skip if the project has no Node.js components.

> What is the npm scope for internal packages in this project? (e.g. `@acme`, `@myproject`)

Convention: use the company or project name in lowercase, no spaces. Example: `@pauloferreira25`.
Once defined, this scope must never change — renaming it breaks all `file:` references across every package.

Document the chosen scope in `docs/monorepo-architecture.md`.

### 2.6 Database

**Per-database questions** (repeat for each database):
1. Technology (e.g. DynamoDB, PostgreSQL, Redis, MongoDB)
2. See [Architecture File Lookup](#architecture-file-lookup) for the technology
3. Will it run locally during development?
   - If yes → read `.ia/architecture/01-devcontainer.md`.

**Flow:**
> Will this project use a database?

If no → skip to 2.7.

If yes:
> How many database technologies?

Repeat the per-database questions above for each database.

### 2.7 Infra

> Will this monorepo have infrastructure as code?

If yes:
- Which IaC tool? (most common: CDK or OpenTofu) — see [Architecture File Lookup](#architecture-file-lookup)
- Which cloud provider? (AWS / GCP / Azure / other)
- Environments — `dev` and `prd` are always included. Ask about additional environments (examples below — the human decides which apply):
  - Staging (e.g. `hml`)?
  - Integration (e.g. `itg`)?
  - Any other environments?
- Which regions will be used? (list all — format: `us-east-1`, `sa-east-1`, `us-central1`, etc.)
- Will `shared/global` be needed?
  - `shared/global` = resources created once per account/project (e.g. Route53 / Cloud DNS / Azure DNS, IAM policies)
- Will `shared/<region>` be needed? If yes, which regions?
  - `shared/<region>` = resources shared across all environments in that region (e.g. SES, shared VPCs)

> Note: `<env>/<region>` directories always exist for every environment × region combination. This is not a question — it is always included.

Sub-divisions inside `<env>/<region>` (e.g. `dev/us-east-1/lambda`) are not defined by this playbook. Left for the team to define.

---

## Phase 3 — Preview

After collecting all answers, build a new table with the actual values from the interview and present it to the human.

**Do not show the template below to the human.** The template defines which rows to include and when — use it as a reference to build the real table. The real table uses the actual names provided by the human, not placeholders like `<name>` or `<region>`.

**Template (agent reference only):**

| Path | Include when |
|---|---|
| `README.md` | Always — created by this playbook |
| `docs/index.md` | Always — created by this playbook |
| `docs/monorepo-architecture.md` | Always — created by this playbook |
| `task-spec/` | Always — created by this playbook |
| `backend/<name>/` | One row per backend name collected in 2.2 |
| `backend/shared-libs/` | Only if 2 or more backends |
| `frontend/<name>/` | One row per frontend name collected in 2.3 |
| `frontend/shared-libs/` | Only if 2 or more frontends |
| `lambdas/nodejs/<name>/` | One row per Node.js lambda name collected in 2.4 |
| `lambdas/nodejs/shared-libs/` | Only if 2 or more Node.js lambdas |
| `lambdas/python/<name>/` | One row per Python lambda name collected in 2.4 |
| `lambdas/python/shared-libs/` | Only if 2 or more Python lambdas |
| `infra/shared/global/` | Only if human confirmed shared/global in 2.7 |
| `infra/shared/<region>/` | One row per region confirmed for shared/<region> in 2.7 |
| `infra/<env>/<region>/` | One row per environment × region combination from 2.7 |

**Example of the real table the agent should present (using hypothetical interview answers):**

> Note: the example below intentionally shows mixed naming conventions (`adminApi` in camelCase, `itg_queues` in snake_case, `backoffice-api` in kebab-case). This is not an error — it reflects that each service follows the convention of its own language/framework. This cannot be defined at the playbook level; it is determined during the interview when each technology is chosen. The agent must apply the correct convention per service based on what its `.ia/architecture/` file specifies (or what the human confirms for gap technologies).

| Path | Notes |
|---|---|
| `README.md` | Created by this playbook |
| `docs/index.md` | Created by this playbook |
| `docs/monorepo-architecture.md` | Created by this playbook |
| `task-spec/` | Created by this playbook |
| `backend/adminApi/` | Node.js — Fastify (TypeScript — camelCase) |
| `backend/itg_queues/` | Python — FastAPI (snake_case) |
| `backend/backoffice-api/` | Go (kebab-case) |
| `backend/shared-libs/` | Shared between backends |
| `frontend/app/` | React Native — Tamagui |
| `infra/shared/global/` | Route53, IAM policies |
| `infra/dev/us-east-1/` | |
| `infra/prd/us-east-1/` | |

Present the real table and ask:

> Does this structure reflect what you need? Can I proceed to generate the documentation?

Do not continue until the human confirms. The confirmed table is what gets reproduced verbatim in `docs/monorepo-architecture.md`.

---

## Phase 4 — Generate architectural documentation

This playbook only creates the files listed in the Phase 3 table under "Created by this playbook". Nothing else is created — no service directories, no package.json, no initialization.

`docs/monorepo-architecture.md` is the written record of every decision made during the interview. It must be complete enough that any person or agent reading it can understand the full structure of the monorepo without revisiting the conversation.

For each technology identified during the interview, include a hardcoded link to its `.ia/architecture/` file. The links were already resolved during Phase 2 via Architecture File Lookup — at this point, just write them. Do not copy the content of those files — only link to them.

Create `docs/monorepo-architecture.md`. The document must cover two types of content:

**A — Definitions from this playbook** (carry these regardless of interview answers):

- Monorepo purpose: convenience, not coupling — every service must be independently runnable, deployable, and containerizable
- Each service has its own `.env` file. There is no `.env` at the monorepo root
- `shared-libs/` are installed via `file:` references — each service pulls what it needs independently
- A service extracted from the monorepo into its own repo must work without changes
- `02-nodejs-typescript.md` rules apply inside each service's own directory, not at the monorepo root (e.g. `queue/<worker>/` becomes `backend/<service>/src/queue/<worker>/`)
- `shared-libs/` only exists when there are multiple services in the same block
- CI/CD is not covered by this setup — must be defined separately by the team

**B — Answers collected in the interview** (one section per topic):

- **Repository purpose** — what this repository does (from 2.1)
- **Backend** — names, language, framework, queue technology if applicable; hardcoded link to `.ia/architecture/` file (from 2.2)
- **Frontend** — names, technology, UI library; hardcoded link to `.ia/architecture/` file (from 2.3)
- **Lambdas** — language(s), names; hardcoded link to `.ia/architecture/` file (from 2.4)
- **npm scope** — the chosen scope value; rule: never change after definition (from 2.5, if applicable)
- **Database** — technology, whether it runs locally; hardcoded link to `.ia/architecture/` file (from 2.6)
- **Infra** — tool, cloud provider, environments, regions, infra layer structure (from 2.7)
- **Directory structure** — the confirmed structure from Phase 3, reproduced in full

Create `docs/index.md` linking to `monorepo-architecture.md`.

Create `README.md` with a brief description of the repository (from 2.1).

Create `task-spec/.gitkeep`. This directory is the designated location for all AI-generated task specs in this project. Fixed documentation lives in `docs/` — specs that drive implementation live in `task-spec/`. Any playbook or agent that generates a task spec must place it here.

Present the generated files to the human for review.

---

## Phase 5 — Task spec (optional follow-up)

After presenting the files, ask:

> The documentation is ready. Do you want me to generate a task spec (`task-spec/task-monorepo-architecture.md`) with everything that needs to be built?

- **If no** → the playbook is done. Implementation is the team's responsibility.
- **If yes** → generate `task-spec/task-monorepo-architecture.md` by:
  1. Reading `docs/monorepo-architecture.md`
  2. For each technology that has a `.ia/architecture` file, reading that file and deriving the tasks from it
  3. Combining both into a list of tasks — one per service/app/lambda/infra block

**Task spec format:** This playbook does not define the format of the task spec — that is the responsibility of the skill or agent used to generate it. Each concern in its own place.

**When no `.ia/architecture` file exists for a technology:**
1. Inform the human: "There is no `.ia/architecture` document for [technology]."
2. Interview the human about the community standards they want to follow (directory structure, key config files, conventions).
3. Use those answers to write that technology's tasks in the spec.
4. Never skip silently. Never generate tasks for a technology without either a `.ia/architecture` file or human-confirmed community standards.

