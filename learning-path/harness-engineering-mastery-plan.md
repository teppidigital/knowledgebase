# Harness Engineering — Expert Mastery Plan

## Purpose

This document is your consolidated, structured roadmap to becoming an expert **harness engineer**: someone who can design, build, and continuously improve the environment, tooling, and feedback loops that enable AI coding agents to deliver high-quality software at scale.

> **Core mental model:** Your job is not to write code. Your job is to make agents write great code reliably.

---

## Prerequisites Self-Assessment

Before starting, you should be comfortable with:

| Skill | Minimum Level Required |
|-------|----------------------|
| Git and branching strategies | Intermediate |
| CI/CD pipelines (GitHub Actions or similar) | Intermediate |
| At least one backend language (TypeScript, Python, Go) | Intermediate |
| Basic shell scripting (bash/zsh) | Basic |
| Docker and containerisation | Basic |
| Reading and writing Markdown | Proficient |

If any of these gaps exist, close them first — agents will expose all weak spots in your environment design.

---

## Phase Overview

```
Phase 1 (Weeks 1–2)   → Foundational Mindset
Phase 2 (Weeks 3–5)   → Repository Architecture
Phase 3 (Weeks 6–8)   → Agent Legibility & Tooling
Phase 4 (Weeks 9–11)  → Enforcement & Taste Encoding
Phase 5 (Weeks 12–14) → Feedback Loops & Self-Review
Phase 6 (Weeks 15–17) → Entropy Management & Quality Signals
Phase 7 (Weeks 18–20) → Full Autonomous Delivery
Phase 8 (Ongoing)     → Expert Practice & Community
```

---

## Phase 1 — Foundational Mindset (Weeks 1–2)

### Goal
Internalise the paradigm shift. Understand *why* harness engineering exists, and how the job of the engineer fundamentally changes.

### Study Topics

**Week 1 — The Paradigm Shift**

- [ ] Read [`ai-llm/16-harness-engineering.md`](../ai-llm/16-harness-engineering.md) in full — twice. On the second pass, annotate every principle with a real example you can imagine from your work.
- [ ] Read [`ai-llm/01-rag.md`](../ai-llm/01-rag.md) — understand how agents consume and retrieve context.
- [ ] Read [`ai-llm/04-ai-agents-tool-use.md`](../ai-llm/04-ai-agents-tool-use.md) — understand what agents can and cannot do autonomously.
- [ ] Read the OpenAI harness engineering case study: [openai.com/index/harness-engineering](https://openai.com/index/harness-engineering/)
- [ ] Read the Codex App Server follow-up: [openai.com/index/unlocking-the-codex-harness](https://openai.com/index/unlocking-the-codex-harness/)

**Week 2 — Mental Model Building**

- [ ] Study the traditional vs. harness engineering comparison table in the knowledge doc — for each row, write one concrete consequence if you *don't* make the shift.
- [ ] Read the [AGENTS.md community standard](https://agents.md/) — understand the emerging convention.
- [ ] Read [ARCHITECTURE.md pattern by matklad](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html) — this is the structural model for your repo knowledge layer.
- [ ] Read [Parse, don't validate — Alexis King](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — one of the core taste invariants you will enforce.

### Week 1–2 Deliverable

Write a 1-page personal "Harness Engineering Manifesto" — your own distillation of the five principles you most need to encode into your future harnesses. Keep it in your notes. You'll revise it at Phase 8.

### Validation Questions

- Can you explain to a non-technical stakeholder why an engineer without any harness experience would produce slower results with an AI coding agent than with none?
- Can you map the five pillars to a concrete gap you've seen in a real past project?
- Can you describe the "Ralph Wiggum Loop" (self-review loop) from memory?

---

## Phase 2 — Repository Architecture (Weeks 3–5)

### Goal
Design and build the knowledge layer of a real harness: the `AGENTS.md`, `ARCHITECTURE.md`, and `docs/` structure that makes a repo self-describing for an agent.

### Study Topics

**Week 3 — The Knowledge Layer**

- [ ] Study Pillar 1 ("Repository as System of Record") in depth — review the artifact table.
- [ ] Read [`devops/02-gitops.md`](../devops/02-gitops.md) — the repo is the source of truth for both humans and agents equally.
- [ ] Study the "Map, not Manual" principle — work out the failure modes of BOTH extremes (too little context vs. monolithic AGENTS.md).

**Week 4 — Designing AGENTS.md**

- [ ] Write a `AGENTS.md` for a real (or fictional) project. Requirements:
  - Max 100 lines.
  - Must link to at least 3 sub-documents in `docs/`.
  - Must clearly state what the agent should do *first* when it receives a task.
  - Must state the architectural layer ordering for this repo.
- [ ] Peer-review (or self-review against) the checklist: Can a new agent navigate the entire repo using only this file as a starting point?

**Week 5 — Design Documents and Execution Plans**

- [ ] Read the [Codex Execution Plans cookbook](https://cookbook.openai.com/articles/codex_exec_plans).
- [ ] Write one execution plan for a mid-sized feature task (10–20 steps). Structure it as the agent would consume it — context-first, acceptance criteria explicit, steps granular.
- [ ] Write one design document using the `docs/design-docs/` template (problem, decision, verification status, core beliefs). Include a "verification" section that a CI check could eventually validate automatically.

### Week 3–5 Deliverable

A fully wired `AGENTS.md` + `ARCHITECTURE.md` + at least three `docs/` sub-documents for a real or practice project. Show this to someone unfamiliar with the project — if they can understand the structure in 10 minutes, you pass.

### Validation Questions

- Why is a fat `AGENTS.md` worse than no `AGENTS.md`?
- How does "progressive disclosure" in docs reduce context window pressure?
- What is the difference between a product spec and an execution plan in the docs layer?

---

## Phase 3 — Agent Legibility & Tooling (Weeks 6–8)

### Goal
Design and wire the tooling layer — the infrastructure that makes the running application, its logs, metrics, traces, and UI fully accessible to the agent without human mediation.

### Study Topics

**Week 6 — Observability for Agents**

- [ ] Read [`observability/`](../observability/) folder in full.
- [ ] Read [`devops/06-observability-opentelemetry.md`](../devops/06-observability-opentelemetry.md).
- [ ] Study [`ai-llm/08-ai-observability.md`](../ai-llm/08-ai-observability.md).
- [ ] Goal: You must be able to design a local observability stack (logs, metrics, traces) that an agent can query via LogQL, PromQL, and TraceQL from a tool call.

**Week 7 — Ephemeral Environments and Browser Tooling**

- [ ] Research Chrome DevTools Protocol (CDP) — understand how agents can drive a browser programmatically: DOM snapshots, screenshots, form interactions.
- [ ] Study git worktree mechanics: `git worktree add` — practice spinning up one isolated app instance per active change branch.
- [ ] Design a local boot script that, given a branch name, spins up a fully isolated environment with its own observability stack.

**Week 8 — Tool Catalogue Design**

- [ ] Read [`ai-llm/14-function-calling-structured-outputs.md`](../ai-llm/14-function-calling-structured-outputs.md) — how agents invoke tools and consume structured results.
- [ ] Read [`api-design/01-rest-api-design.md`](../api-design/01-rest-api-design.md) and [`api-design/03-grpc-protobuf.md`](../api-design/03-grpc-protobuf.md) — these inform how you design internal tooling APIs for agent consumption.
- [ ] Inventory and document the tools available to agents in your harness. For each tool, specify: input schema, output schema, failure modes, retry semantics.

### Week 6–8 Deliverable

A working (or fully designed) "agent-legible environment": git worktree boot script + local observability stack + tool catalogue document showing how an agent would query logs and drive the UI. Record a short demo or write a walkthrough.

### Validation Questions

- What query would an agent run to check if a service started in under 800 ms?
- Why do you need one app instance per git worktree rather than sharing a single running instance?
- What happens if the agent's only feedback mechanism is "it worked" vs. "it didn't"?

---

## Phase 4 — Enforcement & Taste Encoding (Weeks 9–11)

### Goal
Learn to codify human taste into mechanical rules — linters, CI checks, and architectural tests — so that agent-generated code automatically conforms to your standards without human review of every PR.

### Study Topics

**Week 9 — Custom Linters**

- [ ] Study [`devsecops/02-sast.md`](../devsecops/02-sast.md) — understand static analysis at the tool level.
- [ ] Pick your project's language and learn to write a custom lint rule (ESLint custom plugin, Ruff plugin, golangci-lint custom analyser, etc.).
- [ ] Write three custom lint rules for real invariants in your project:
  1. A naming convention rule.
  2. A "no raw `console.log`" or equivalent structured logging rule.
  3. A file size or complexity limit rule.
- [ ] Ensure every lint error message contains a remediation hint the agent can act on directly. Test this by reading the error message as if you are the agent.

**Week 10 — Architecture Tests and Dependency Enforcement**

- [ ] Research layer dependency enforcement tools for your language (e.g., `dependency-cruiser` for JS/TS, `archunit` for JVM, `import-linter` for Python).
- [ ] Implement the layered domain architecture pattern from Pillar 3 in a practice project — write tests that fail when a `UI` layer imports from `Repo` directly.
- [ ] Study [`distributed-design-pattern/`](../distributed-design-pattern/) — understand the architectural patterns that should become enforced invariants.

**Week 11 — CI Pipeline Hardening for Agent Workflows**

- [ ] Read [`devops/01-cicd-pipeline-design.md`](../devops/01-cicd-pipeline-design.md) in full.
- [ ] Read [`devops/04-deployment-strategies.md`](../devops/04-deployment-strategies.md).
- [ ] Design a CI pipeline that is agent-optimised:
  - Fast feedback (< 5 minutes to first signal).
  - Lint errors surfaced as structured JSON that agents can parse.
  - No blocking flaky tests — flake detection and auto-quarantine.
  - Merge gates that are non-blocking for agent-generated PRs that pass all checks.

### Week 9–11 Deliverable

A working custom linter with 3+ rules, a layer dependency enforcement setup, and a CI pipeline design document. Run a deliberately "bad" agent-generated PR through the system and verify that the lint errors alone guide the agent to the correct fix without any human comment.

### Validation Questions

- Why is the lint error message itself part of the harness, not just the rule?
- What is the failure mode if your CI pipeline takes 30 minutes?
- How does "enforce invariants, not implementations" differ from enforcing code style?

---

## Phase 5 — Feedback Loops & Self-Review (Weeks 12–14)

### Goal
Build and operate multi-stage self-review loops where agents review their own work and each other's, minimising the ratio of human review time to agent output.

### Study Topics

**Week 12 — The Self-Review Loop**

- [ ] Study the "Ralph Wiggum Loop" in detail from the harness engineering doc.
- [ ] Implement a local pre-PR checklist that the agent self-executes before opening a PR:
  - All lint checks pass.
  - All tests pass.
  - UI journeys validated via CDP.
  - Observability signals confirm no regression.
- [ ] Script this as a single `bin/pre-pr-check.sh` that agents can invoke as a tool call.

**Week 13 — Agent-to-Agent Review**

- [ ] Research multi-agent review patterns: one agent proposes, a second agent reviews using a structured rubric.
- [ ] Read [`ai-llm/13-llm-evaluation.md`](../ai-llm/13-llm-evaluation.md) — evaluation frameworks apply to agent output review.
- [ ] Design a review rubric document (as a repo artifact) that a reviewer agent uses to evaluate PRs against architecture, taste invariants, and functional correctness.
- [ ] Run a practice session: have one agent write a feature, have a second (or the same agent with a review prompt) critique it against the rubric. Iterate.

**Week 14 — Escalation and Human-in-the-Loop Design**

- [ ] Define the escalation criteria: what signals indicate a task requires human judgment?
  - Genuine product ambiguity (not resolvable from specs).
  - Security decisions outside documented boundaries.
  - Architecture changes that affect the layer ordering.
  - External system integration decisions not covered by reference docs.
- [ ] Document these escalation criteria in your `AGENTS.md` so agents know when to stop and surface a question rather than guess.
- [ ] Study [`ai-llm/09-guardrails-content-safety.md`](../ai-llm/09-guardrails-content-safety.md) — guardrails apply to agent behaviours as well as content.

### Week 12–14 Deliverable

A working `bin/pre-pr-check.sh` tool, a structured agent review rubric document, and an escalation criteria section added to your `AGENTS.md`. Run a full end-to-end exercise: task → agent executes → self-review → agent review → PR with evidence.

### Validation Questions

- At what point in the self-review loop should a PR be opened?
- What happens to agent throughput if human review is mandatory on every PR?
- How do you distinguish "agent needs a better tool" from "task is genuinely ambiguous"?

---

## Phase 6 — Entropy Management & Quality Signals (Weeks 15–17)

### Goal
Design and operate the continuous background processes that keep the codebase legible and healthy over time as agent-generated code accumulates.

### Study Topics

**Week 15 — Golden Principles and Quality Scoring**

- [ ] Derive and document the "Golden Principles" for your codebase — the 5–10 opinionated rules that define good code in your domain.
- [ ] Design a `QUALITY_SCORE.md` schema: per-domain quality grades, how they're computed, last-updated timestamp, and agent that updates them.
- [ ] Read [`data-solutions/12-data-governance-catalogue.md`](../data-solutions/12-data-governance-catalogue.md) — governance patterns apply to codebase quality catalogues.

**Week 16 — Background Cleanup Agents**

- [ ] Design a background Codex task that runs on a scheduled cadence:
  - Scans for deviations from golden principles.
  - Updates quality grades in `QUALITY_SCORE.md`.
  - Opens targeted, small refactoring PRs (< 50 lines changed).
  - These PRs should be auto-mergeable in under 1 minute if all checks pass.
- [ ] Study [`devops/09-database-devops.md`](../devops/09-database-devops.md) — the pattern of incremental, reversible, automated changes is identical to what you're doing with code entropy.
- [ ] Study [`devops/15-devops-metrics.md`](../devops/15-devops-metrics.md) — define your harness health KPIs.

**Week 17 — Doc Freshness and Knowledge Gardening**

- [ ] Write a CI lint rule that checks whether referenced `docs/` documents have been updated within a configurable time window.
- [ ] Design a "doc gardening" agent task that:
  - Detects stale documentation (untouched files where linked code has changed).
  - Opens a PR with updated documentation, prompting a human to review the substance.
- [ ] Apply this to your own knowledge portal in this workspace — practice on real content.

### Week 15–17 Deliverable

A `QUALITY_SCORE.md` for your practice project, a background cleanup agent task definition, and a doc freshness lint rule. Demonstrate one full cycle of: background task detects drift → opens PR → auto-merges.

### Validation Questions

- Why does AI-generated code accumulate entropy faster than human-written code?
- What is the difference between a "Golden Principle" and a lint rule?
- How do you prevent cleanup agents from themselves introducing inconsistency?

---

## Phase 7 — Full Autonomous Delivery (Weeks 18–20)

### Goal
Operate the complete harness end-to-end: design a feature, write a prompt, observe the agent execute the full delivery loop, and measure outcomes.

### Study Topics

**Week 18 — Full Feature Delivery Exercise**

- [ ] Pick a real, medium-complexity feature (2–3 days of manual effort).
- [ ] Write a product spec for it in `docs/product-specs/`.
- [ ] Write an execution plan for it in `docs/exec-plans/`.
- [ ] Submit a single prompt to Codex (or your chosen agent).
- [ ] Do not intervene. Observe. Note every place the agent gets stuck — these are harness gaps.
- [ ] After completion: classify each gap as missing tool, missing doc, missing linter, or missing test.

**Week 19 — Gap Analysis and Harness Improvement**

- [ ] For each gap identified in Week 18, add the missing element to the harness (via the agent itself wherever possible).
- [ ] Re-run the exercise on a comparable feature.
- [ ] Measure: how many human interventions were required this time?
- [ ] Target: 50% reduction in interventions between first and second run.

**Week 20 — Metrics and Reporting**

- [ ] Read [`ai-llm/12-ai-cost-optimisation.md`](../ai-llm/12-ai-cost-optimisation.md) — track model cost per feature, not just time.
- [ ] Read [`finops/`](../finops/) — connect agent economics to business economics.
- [ ] Define your harness maturity scorecard:

| Dimension | Immature | Mature | Expert |
|-----------|----------|--------|--------|
| Repo knowledge | No AGENTS.md | AGENTS.md + 3 docs | Full indexed doc graph |
| Tooling | Manual app boot | Scripted worktree | Fully automated ephemeral env |
| Enforcement | Zero linters | 3+ custom rules | All taste invariants encoded |
| Feedback loops | Human reviews all | Agent pre-check | Agent-to-agent + selective human |
| Entropy mgmt | None | Manual cleanup | Continuous background agents |
| Delivery speed | ~10 PRs/week | ~50 PRs/week | ~100+ PRs/week |

### Week 18–20 Deliverable

Two full-cycle delivery runs with a gap analysis document between them. A populated harness maturity scorecard. Evidence (PR history, metrics) of measurable improvement.

---

## Phase 8 — Expert Practice & Ongoing Mastery

### Sustaining Expert Status

Harness engineering is a rapidly evolving discipline. Expert maintenance requires:

**Monthly habits:**
- [ ] Review and update your `AGENTS.md` and core docs — do they still reflect reality?
- [ ] Review quality scores — is entropy increasing or decreasing?
- [ ] Run one "adversarial" agent task: give the agent a deliberately under-specified task and watch where it fails. Add one new harness element based on the failure.

**Quarterly habits:**
- [ ] Review the [OpenAI Cookbook](https://cookbook.openai.com/) and release notes for new agent capabilities — some will make existing harness elements obsolete.
- [ ] Review model benchmarks ([`ai-llm/13-llm-evaluation.md`](../ai-llm/13-llm-evaluation.md)) — your task routing and model selection should evolve.
- [ ] Revise your personal "Harness Engineering Manifesto" from Phase 1. Mark what has changed.

**Expertise signals (you're expert when):**
- You can diagnose an agent failure mode in under 5 minutes and trace it to a specific harness gap.
- Your harness produces fewer than 2 human interventions per 10 agent-generated PRs.
- You can onboard a new AI agent to your codebase with a single `AGENTS.md` pointer and zero verbal explanation.
- New engineers learn your codebase's architecture faster by reading the agent docs than by asking you.
- Your background cleanup agents run weekly and open 0–2 PRs per cycle (debt is under control).

---

## Cross-Reference: Knowledge Portal Topics

The following knowledge portal documents are directly relevant to harness engineering mastery. Study them in the order recommended within each phase above.

| Phase | Documents |
|-------|-----------|
| 1 | `ai-llm/16-harness-engineering.md`, `ai-llm/01-rag.md`, `ai-llm/04-ai-agents-tool-use.md` |
| 2 | `devops/02-gitops.md` |
| 3 | `observability/`, `devops/06-observability-opentelemetry.md`, `ai-llm/08-ai-observability.md`, `ai-llm/14-function-calling-structured-outputs.md`, `api-design/01-rest-api-design.md` |
| 4 | `devsecops/02-sast.md`, `distributed-design-pattern/`, `devops/01-cicd-pipeline-design.md`, `devops/04-deployment-strategies.md` |
| 5 | `ai-llm/13-llm-evaluation.md`, `ai-llm/09-guardrails-content-safety.md` |
| 6 | `data-solutions/12-data-governance-catalogue.md`, `devops/09-database-devops.md`, `devops/15-devops-metrics.md` |
| 7 | `ai-llm/12-ai-cost-optimisation.md`, `finops/` |

---

## Quick Reference: The Five Pillars Checklist

At any point in your journey, use this checklist to assess how mature your current harness is:

```
Pillar 1 — Repository as System of Record
  [ ] AGENTS.md exists and is ≤ 100 lines
  [ ] ARCHITECTURE.md maps all domains and layer ordering
  [ ] docs/ contains design docs, exec plans, product specs
  [ ] All docs have a "last verified" date
  [ ] QUALITY_SCORE.md exists and is auto-updated

Pillar 2 — Agent Legibility
  [ ] Codebase uses stable, well-documented dependencies
  [ ] Local observability stack queryable via LogQL/PromQL/TraceQL
  [ ] UI accessible via Chrome DevTools Protocol
  [ ] One app instance per git worktree (boot script exists)
  [ ] All agent-facing tool schemas are documented

Pillar 3 — Architecture Enforcement
  [ ] Layer dependency rules are mechanically enforced
  [ ] All custom lint rules have remediation messages
  [ ] File size and complexity limits are enforced in CI
  [ ] Naming conventions are lint-enforced, not comment-enforced

Pillar 4 — Feedback Loops
  [ ] bin/pre-pr-check.sh script exists and passes before every PR
  [ ] Agent review rubric document is in repo
  [ ] Escalation criteria documented in AGENTS.md
  [ ] CI gives first signal in < 5 minutes

Pillar 5 — Entropy Management
  [ ] Golden Principles document exists (5–10 rules)
  [ ] Background cleanup agent task is scheduled
  [ ] Doc freshness lint rule is active in CI
  [ ] Refactoring PRs are < 50 lines and auto-mergeable
```

---

## Resources Summary

| Resource | URL | Phase |
|----------|-----|-------|
| Harness Engineering (OpenAI) | [openai.com/index/harness-engineering](https://openai.com/index/harness-engineering/) | 1 |
| Unlocking the Codex Harness | [openai.com/index/unlocking-the-codex-harness](https://openai.com/index/unlocking-the-codex-harness/) | 1 |
| AGENTS.md standard | [agents.md](https://agents.md/) | 2 |
| ARCHITECTURE.md pattern | [matklad.github.io](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html) | 2 |
| Parse, don't validate | [lexi-lambda.github.io](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) | 2 |
| Codex Execution Plans | [cookbook.openai.com/articles/codex_exec_plans](https://cookbook.openai.com/articles/codex_exec_plans) | 2 |
| Chrome DevTools Protocol | [chromedevtools.github.io/devtools-protocol](https://chromedevtools.github.io/devtools-protocol/) | 3 |
| OpenAI Cookbook | [cookbook.openai.com](https://cookbook.openai.com/) | 8 |

---

*Last updated: April 2026 — revise quarterly as the discipline evolves.*
