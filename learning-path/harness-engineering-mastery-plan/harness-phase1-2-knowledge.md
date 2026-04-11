# Harness Engineering Phase 1–2 Deep Knowledge
## Foundational Mindset + Repository Architecture

> Covers Weeks 1–5 of the Harness Engineering Mastery Plan.  
> This file is implementation-focused. For concepts and pillars, see [`ai-llm/16-harness-engineering.md`](../../ai-llm/16-harness-engineering.md).

---

## The Paradigm Shift — Concrete Consequences

The engineering job changes in four concrete ways. Every row in the table below has an action you must take.

| Traditional Engineering Work | Harness Engineering Equivalent | If You Don't Shift |
|---|---|---|
| Write `UserService.ts` | Write `docs/design-docs/user-service.md` with acceptance criteria the agent validates | Agent writes an architecturally inconsistent service that passes tests but diverges from the system |
| Comment "this is fragile, fix later" | Open a Codex task that fixes it and runs in the background this sprint | The comment is invisible to the agent; the fragility compounds |
| Explain architecture in a PR review | Write a lint rule that fails on the violation with a remediation message | You become the bottleneck; agent throughput is limited by your review capacity |
| Debug a failed PR by reading the diff | Identify which harness element was missing (tool? doc? linter?) | You fix the symptom, not the harness; the same failure recurs on the next similar task |
| Update Confluence after a refactor | Write a CI check that fails if the doc's linked code path no longer exists | Documentation rots; agents make decisions based on stale context |

**The one-sentence mindset test:** Before writing any code yourself, ask: "Is this task something I should encode into a rule, a doc, or a tool — and then let the agent execute?"

---

## AGENTS.md — Full Annotated Template

This is the ~100-line entry-point file. It is the **only file** the agent reads unconditionally. Everything else it reads on demand by following links here.

```markdown
# AGENTS.md

<!-- 
  This file is the agent's entry point.
  - Keep it under 100 lines.
  - Every section must have a link or a pointer.
  - Do NOT put content here that belongs in docs/.
  - Last verified: YYYY-MM-DD (update this monthly)
-->

## What this repo is

<One-sentence description of the system and its primary purpose>
<Who are the users and what do they do with it>

## Start here

When you receive a task:
1. Read `ARCHITECTURE.md` to understand the layer ordering and domain boundaries.
2. Read the relevant design doc in `docs/design-docs/` for the affected domain.
3. If a task touches the API layer, read `docs/references/api-contracts.md`.
4. Check `QUALITY_SCORE.md` for the current state of the domain you are modifying.
5. Run `bin/check.sh` before opening a PR.

## Domains

| Domain | Location | Design Doc |
|--------|----------|------------|
| <Domain A> | `packages/<domain-a>/` | `docs/design-docs/<domain-a>.md` |
| <Domain B> | `packages/<domain-b>/` | `docs/design-docs/<domain-b>.md` |
| Shared Utils | `packages/utils/` | `docs/design-docs/utils.md` |

## Architecture rules (enforced by linters)

- Layers: `types → config → repo → service → runtime → ui`
- No layer may import from a layer to its right (caught by `dependency-cruiser`)
- Cross-cutting concerns (auth, telemetry, feature flags) enter ONLY via `packages/providers/`
- See `ARCHITECTURE.md` for the full domain map

## Coding conventions

- No `console.log` — use the structured logger from `packages/logger/` (lint rule: `no-console`)
- All external input must be parsed at the boundary — use Zod schemas (see `docs/design-docs/validation.md`)
- File size limit: 300 lines. If you exceed this, split the file.
- Naming: see `docs/references/naming-conventions.md`

## Testing

- Unit tests: co-located with source (`*.test.ts` next to `*.ts`)
- Integration tests: `tests/integration/`
- E2E: `tests/e2e/` — runs against the local boot environment
- Run all: `pnpm test`; run affected only: `pnpm test:affected`

## Tooling available to you

- `bin/boot.sh <branch>` — boots an isolated app instance for the given branch
- `bin/check.sh` — runs all pre-PR checks (lint, tests, E2E smoke)
- `gh pr create --draft` — opens a draft PR
- Observability: Grafana at `http://localhost:3001` (LogQL, PromQL, TraceQL)
- Browser control: Chrome DevTools Protocol on port `9222`

## When to stop and ask

Stop and surface a question (do not guess) when:
- The task requires a product decision not answerable from specs in `docs/product-specs/`
- The task requires a change to the layer ordering in `ARCHITECTURE.md`
- The task touches a security boundary outside `docs/design-docs/security.md`
- The task introduces a new external dependency not on the approved vendor list

## Escalation

Create a GitHub issue labelled `agent-escalation` with:
- The task you were given
- The specific question you cannot resolve from context
- The files you read before escalating
```

### AGENTS.md Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails |
|---|---|
| Inline all rules instead of linking to docs | Exceeds 100 lines; context crowding; rots without the linked code |
| "Do your best to follow our style" | Vague guidance produces inconsistent output — every instruction must be verifiable |
| No "start here" sequence | Agent reads irrelevant files first, wastes context window |
| Missing "when to stop" section | Agent guesses on product decisions; produces technically complete but functionally wrong code |
| Last-verified date missing | You cannot mechanically check whether this file is stale |

---

## ARCHITECTURE.md — Full Annotated Template

This file describes the structural map of the codebase. It is NOT a code style guide — it is a map of what exists and how it fits together.

```markdown
# ARCHITECTURE.md

<!-- Last verified: YYYY-MM-DD -->

## System overview

<Two-paragraph description of what the system does, its primary actors, and its boundaries.>

## Domain map

The system is divided into the following domains. Each domain is independent; cross-domain
communication happens only through explicitly defined interfaces.

```
packages/
├── auth/           → Authentication and session management
├── users/          → User profile and preferences
├── billing/        → Subscription and payment flows
├── notifications/  → Email, push, webhook delivery
├── shared/
│   ├── types/      → Shared TypeScript types (no runtime code)
│   ├── utils/      → Pure utility functions
│   └── providers/  → Cross-cutting: logger, feature flags, telemetry, config
└── infra/          → Infrastructure config, CI tooling, migrations
```

## Layer ordering (enforced)

Within each domain, code is organised in layers. Dependencies ONLY go left to right:

```
types → config → repo → service → runtime → ui
```

- `types`: pure TypeScript types and Zod schemas — no imports from within the domain
- `config`: environment-specific configuration, validated at startup
- `repo`: data access (DB, external APIs) — returns typed domain objects
- `service`: business logic — no I/O, no framework imports
- `runtime`: framework adapters (Express handlers, Lambda handlers, Kafka consumers)
- `ui`: React components — imports from `runtime` only via props or context

Violations are caught by `dependency-cruiser`. Run `pnpm lint:deps` to check.

## Cross-cutting concerns

All cross-cutting concerns enter domains via `packages/providers/`:

| Concern | Package | How to Use |
|---------|---------|------------|
| Structured logging | `providers/logger` | `import { logger } from '@app/providers/logger'` |
| Telemetry / tracing | `providers/telemetry` | Wrap service functions with `trace()` |
| Feature flags | `providers/flags` | `flags.isEnabled('feature-name')` |
| Auth context | `providers/auth` | Injected via request context |

## Data flow

```
HTTP Request → Runtime layer (validates input with Zod)
             → Service layer (pure business logic)
             → Repo layer (DB/API call)
             → Response (typed domain object)
```

External calls always go through the Repo layer. Service functions never make I/O calls directly.

## Boundaries and interfaces

- Domain A talks to Domain B only through `packages/<domain-b>/src/index.ts` (the public API)
- Internal sub-modules are not cross-domain accessible
- If you find yourself importing from `packages/<domain>/src/internal/`, stop — it's a boundary violation

## Known tech debt

See `QUALITY_SCORE.md` for current state. See `docs/exec-plans/tech-debt-*.md` for active plans.
```

---

## docs/ Folder Structure — Reference Layout

```
docs/
├── design-docs/          # One file per domain; describes approach, decisions, verification
│   ├── auth.md
│   ├── billing.md
│   └── ...
├── exec-plans/           # Active, completed, and planned work items (Codex execution plans)
│   ├── active/
│   ├── completed/
│   └── tech-debt/
├── product-specs/        # Product requirements as structured, agent-readable specs
│   └── feature-xyz.md
├── references/           # External API docs, naming conventions, vendor docs (llms.txt)
│   ├── api-contracts.md
│   ├── naming-conventions.md
│   └── external-services/
│       └── stripe-reference.md  # vendor docs distilled for agent consumption
└── adr/                  # Architecture Decision Records (append-only log)
    ├── 001-monorepo-choice.md
    └── 002-zod-for-validation.md
```

### Design Doc Structure (per-domain)

```markdown
# Design Doc: <Domain Name>

<!-- Last verified: YYYY-MM-DD -->
<!-- Verification status: CURRENT | STALE | IN_PROGRESS -->

## Problem

<One paragraph: what business problem does this domain solve?>

## Decision

<What architectural decisions were made and why? Include the alternatives that were rejected.>
<Keep honest about trade-offs — agents use this to understand intent.>

## Core beliefs

The following assumptions are load-bearing for this design:
- <Assumption 1>
- <Assumption 2>

If any of these change, the design should be revisited.

## Verification

<How do you know this design is working? What can be checked automatically vs. manually?>

Automated checks:
- [ ] `pnpm test packages/<domain>` passes
- [ ] `pnpm lint:deps` shows no violations for this domain
- [ ] Integration test `tests/integration/<domain>` passes

Manual checks:
- [ ] `<describe a human-verifiable behaviour>`

## Change history

| Date | Change | Author |
|------|--------|--------|
| YYYY-MM-DD | Initial design | <agent/human> |
```

---

## Execution Plan Template (Codex-Optimised)

An execution plan is a **structured, stepwise task definition** that an agent can follow without ambiguity. It is NOT a product spec. It is NOT a high-level goal. It is an ordered sequence of discrete, verifiable actions.

```markdown
# Execution Plan: <Feature or Task Name>

## Status: ACTIVE | COMPLETED | ABANDONED

## Context (read this first)

- Related domain: `packages/<domain>/`
- Design doc: `docs/design-docs/<domain>.md`
- Product spec: `docs/product-specs/<spec>.md` (if applicable)

## Acceptance criteria

The plan is complete when ALL of the following are true:
- [ ] <Criterion 1 — user-visible behaviour>
- [ ] <Criterion 2 — measurable property, e.g. "response time < 200ms at p95">
- [ ] <Criterion 3 — test coverage>
- [ ] `bin/check.sh` passes with no new errors

## Steps

### Step 1: <Short label>
**Files to change:** `packages/<domain>/src/<file>.ts`  
**What to do:** <Precise description — no ambiguity>  
**Verification:** Run `pnpm test packages/<domain>` — all tests pass

### Step 2: <Short label>
**Files to change:** `packages/<domain>/src/<other-file>.ts`  
**What to do:** <Description>  
**Verification:** `<specific check command>`

### Step N: Open PR
**What to do:** Run `bin/check.sh`. When it passes, open a draft PR with title `[TYPE]: <short summary>`.  
Include in PR description:
- Link to this execution plan
- Evidence that all acceptance criteria are met
- Screenshots or LogQL query results if behaviour changed

## Known risks / blockers

- <Risk 1 and how to handle it>
- If you hit <condition>, stop and create a `agent-escalation` issue

## Definition of "done" for archives

Move this file to `docs/exec-plans/completed/` after the PR is merged.
```

### Execution Plan Quality Checklist

| Check | What to Look For |
|---|---|
| Context-first | Does the plan tell the agent what to read BEFORE starting? |
| Measurable acceptance criteria | Are all criteria verifiable with a specific command or observable output? |
| Step granularity | Is each step executable in one sitting without judgment calls? |
| Verification per step | Does every step have a "how do you know it worked?" check? |
| Escalation path | Does the plan say what to do when it gets stuck? |
| PR evidence | Does the final step specify what evidence to include in the PR? |

---

## Progressive Disclosure Architecture

The most common AGENTS.md failure mode is making it a monolithic reference manual. The correct architecture is a **graph of progressively deeper documents** — the agent traverses depth only when the task requires it.

```
AGENTS.md (100 lines — entry point)
    │
    ├── ARCHITECTURE.md (200-300 lines — structural map)
    │       └── references domain design docs
    │
    ├── docs/design-docs/<domain>.md (per domain — deep context)
    │       └── problem, decisions, core beliefs, verification
    │
    ├── docs/product-specs/<feature>.md (per feature — requirements)
    │
    ├── docs/exec-plans/<task>.md (per task — implementation steps)
    │
    └── docs/references/<topic>.md (per reference — vendor/API docs)
```

**Context window budget model:**

| Document | Lines | When Agent Reads It | Context Cost |
|---|---|---|---|
| AGENTS.md | ~100 | Every task | Low (always paid) |
| ARCHITECTURE.md | ~300 | Structural tasks | Medium |
| Design doc (1 domain) | ~150 | Domain-specific task | Low |
| Product spec | ~200 | Feature implementation | Medium |
| Exec plan | ~100 | When assigned | Low |
| Reference doc | ~500 | API integration tasks | High (but targeted) |

Design so that a typical task requires: AGENTS.md + ARCHITECTURE.md + 1 design doc = ~550 lines of context. This leaves the model's context window predominantly for code.

---

## Phase 1–2 Validation Checklist

```
Knowledge layer
  [ ] AGENTS.md exists, ≤ 100 lines, has "start here" sequence
  [ ] AGENTS.md has domain table linking to design docs
  [ ] AGENTS.md has "when to stop" escalation section
  [ ] AGENTS.md has last-verified date
  [ ] ARCHITECTURE.md exists and maps all domains and layer ordering
  [ ] ARCHITECTURE.md includes cross-cutting concerns table
  [ ] At least 3 design docs exist in docs/design-docs/
  [ ] Each design doc has verification section with runnable checks
  [ ] At least one execution plan exists in docs/exec-plans/
  [ ] Execution plan has explicit acceptance criteria with verification commands

Test: give the AGENTS.md + ARCHITECTURE.md to a colleague unfamiliar with the project.
      Can they answer: "What layer does business logic live in?" in < 2 minutes?
      If no → the knowledge layer is insufficient.
```
