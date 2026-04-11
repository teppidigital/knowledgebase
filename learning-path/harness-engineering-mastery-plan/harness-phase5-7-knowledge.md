# Harness Engineering Phase 5–7 Deep Knowledge
## Feedback Loops, Entropy Management & Autonomous Delivery

> Covers Weeks 12–20 of the Harness Engineering Mastery Plan.  
> Hands-on implementation: pre-PR check script, agent review rubric, escalation criteria, QUALITY_SCORE.md, background cleanup agents, doc freshness linting, maturity scorecard, metrics.

---

## Phase 5 — The Self-Review Loop

### bin/pre-pr-check.sh — Full Implementation

This is the single script the agent runs before opening a PR. It must be a tool call the agent can invoke, and it must output structured, parseable results.

```bash
#!/usr/bin/env bash
# bin/pre-pr-check.sh
# Run all pre-PR checks. Outputs JSON summary to stdout.
# Exit 0 = all checks passed. Exit 1 = one or more checks failed.
# Usage: bin/pre-pr-check.sh [--skip-e2e]

set -euo pipefail

SKIP_E2E=false
[[ "${1:-}" == "--skip-e2e" ]] && SKIP_E2E=true

RESULTS=()
FAILED=false
START=$(date +%s)

run_check() {
  local name="$1"
  local cmd="$2"
  local output
  local exit_code=0

  output=$(eval "$cmd" 2>&1) || exit_code=$?

  if [ $exit_code -eq 0 ]; then
    RESULTS+=("{\"check\":\"$name\",\"status\":\"pass\",\"output\":\"\"}")
  else
    # Escape output for JSON embedding
    local escaped
    escaped=$(echo "$output" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))')
    RESULTS+=("{\"check\":\"$name\",\"status\":\"fail\",\"output\":$escaped}")
    FAILED=true
  fi
}

# 1. Lint (structured JSON output)
run_check "lint" "pnpm lint --format json --output-file /tmp/lint-results.json && \
                  jq -e '[.[] | .messages[] | select(.severity==2)] | length == 0' /tmp/lint-results.json"

# 2. TypeScript type check
run_check "typecheck" "pnpm tsc --noEmit"

# 3. Layer dependency check
run_check "layer-deps" "pnpm lint:deps"

# 4. Unit tests (affected packages only)
run_check "unit-tests" "pnpm test:affected --reporter=json --outputFile=/tmp/test-results.json"

# 5. Build (ensure no broken imports)
run_check "build" "pnpm build"

# 6. E2E smoke tests (UI journeys via CDP)
if [ "$SKIP_E2E" = false ]; then
  BRANCH=$(git branch --show-current)
  run_check "e2e-smoke" "pnpm test:e2e --smoke-only --branch=$BRANCH"
fi

# 7. Doc freshness check
run_check "doc-freshness" "bin/check-doc-freshness.sh"

END=$(date +%s)
ELAPSED=$((END - START))

# Build JSON output
RESULTS_JSON=$(IFS=','; echo "${RESULTS[*]}")
STATUS=$( [ "$FAILED" = true ] && echo "fail" || echo "pass" )

cat <<EOF
{
  "status": "$STATUS",
  "duration_seconds": $ELAPSED,
  "checks": [$RESULTS_JSON],
  "lint_details": $(cat /tmp/lint-results.json 2>/dev/null || echo "[]"),
  "next_steps": $([ "$FAILED" = true ] && echo '"Fix the failing checks above before opening a PR. Read each check output for remediation instructions."' || echo '"All checks passed. Open a PR with: gh pr create --draft --title \"<type>: <summary>\""')
}
EOF

[ "$FAILED" = false ]
```

### Agent Review Rubric Document

This document lives at `docs/references/agent-review-rubric.md`. A reviewer agent reads it when evaluating a PR.

```markdown
# Agent PR Review Rubric

<!-- Used by reviewer agents to evaluate PRs systematically -->
<!-- Last verified: YYYY-MM-DD -->

## How to use this rubric

Review the PR diff against each criterion below. For each item:
- PASS: the criterion is met
- FAIL: the criterion is violated — add a PR comment with the specific violation and its fix
- N/A: the criterion does not apply to this PR type

Score: If any criterion is FAIL, the PR must not be merged. Respond to the PR with your review.

## 1. Architecture Conformance

| Criterion | Check |
|---|---|
| Layer ordering | No backwards layer imports in the diff — verify with `pnpm lint:deps` |
| Domain boundary | No direct cross-domain imports (only through public `index.ts`) |
| Providers pattern | Cross-cutting concerns (logger, flags, telemetry) enter only through providers/ |
| No new raw console.log | Lint passes with no-console rule |

## 2. Code Quality

| Criterion | Check |
|---|---|
| File size | No single file exceeds 300 lines |
| Parse-don't-validate | External input (req.body, API responses, env vars) goes through Zod schema |
| Structured errors | Error objects include `code`, `message`, `context` — not raw Error strings |
| No magic strings | Enum or const is used for repeated string literals |

## 3. Test Coverage

| Criterion | Check |
|---|---|
| New service logic has unit tests | Service-layer functions have corresponding *.test.ts |
| Happy path + at least one error path | Tests cover success and at least one failure mode |
| No commented-out tests | No `xit`, `xdescribe`, `it.skip` in new code |

## 4. Observability

| Criterion | Check |
|---|---|
| New paths are instrumented | New API routes/functions have logger.info on entry and logger.error on failure |
| Performance-sensitive paths traced | Slow operations use trace() from providers/telemetry |

## 5. Documentation

| Criterion | Check |
|---|---|
| Design doc updated | If architecture changed, relevant design-doc's "last verified" date is updated |
| Execution plan archived | If this PR completes an execution plan, the plan is moved to docs/exec-plans/completed/ |
| No stale references | No references to removed files or renamed functions in docs/ |

## 6. PR Hygiene

| Criterion | Check |
|---|---|
| PR title follows convention | `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:` prefix |
| PR description includes evidence | Acceptance criteria status, test output summary, or screenshot attached |
| Draft only if WIP | PR is not draft unless the work is explicitly incomplete |

## Review output format

When submitting your review, output:

```json
{
  "verdict": "approve | request_changes",
  "summary": "<1-2 sentence summary of the overall quality>",
  "violations": [
    {
      "criterion": "<criterion name>",
      "file": "<file:line>",
      "description": "<what is wrong>",
      "fix": "<specific action to take>"
    }
  ]
}
```
```

---

### Escalation Criteria — AGENTS.md Section Template

Add this section to your `AGENTS.md` (summarised) and expand it in `docs/references/escalation-criteria.md`.

```markdown
## When to stop and escalate

Create a GitHub issue with label `agent-escalation` for ANY of the following:

### Product decision required
- The task requires answering "what should the product do here?" — not "how should the code do it?"
- The acceptance criteria in the spec are contradictory or missing for the specific case you've reached
- The feature would change user-visible behaviour in a way not covered by the spec

### Architecture boundary
- The task requires changing the layer ordering in ARCHITECTURE.md
- The task requires adding a new package or domain not described in ARCHITECTURE.md
- Cross-domain communication requires a new interface not yet documented

### Security decision
- The task touches authentication or authorisation logic
- The task adds a new external service integration involving credentials
- The task changes what data is logged (PII risk)

### Unknown territory
- The task requires using a library or API that does not have docs in docs/references/
- More than 3 attempts to get tests passing have failed and the failure mode is not in known errors

### Escalation issue template
Title: `[AGENT-ESCALATION] <short task description>`
Body:
- Task: <the original task prompt>
- Stuck at: <specific decision point>  
- Context read: <list of files read before escalating>
- Options considered: <what you evaluated>
- Question: <the specific question for the human>
```

---

## Phase 6 — Entropy Management

### QUALITY_SCORE.md — Full Schema

```markdown
# QUALITY_SCORE.md

<!-- Auto-updated by background cleanup agent on schedule -->
<!-- Manual override: update by running bin/compute-quality-scores.sh -->
<!-- Last updated: YYYY-MM-DD HH:MM UTC | Agent: codex/cleanup-task-<id> -->

## Summary

| Domain | Score | Trend | Last Clean |
|--------|-------|-------|------------|
| auth | A | → | 2026-04-10 |
| billing | B+ | ↑ | 2026-04-08 |
| users | C | ↓ | 2026-03-28 |
| notifications | B | → | 2026-04-05 |
| shared/utils | A | → | 2026-04-11 |

**Score definitions:**
- A: All golden principles satisfied; no files over size limit; no lint warnings
- B: 1–3 minor violations; all major invariants enforced
- C: 4–10 violations or 1 major invariant failing
- D: >10 violations or multiple major invariants failing
- F: CI does not pass for this domain

## Per-domain detail

### auth — A

| Check | Status |
|---|---|
| No cross-layer imports | ✓ |
| All external input validated | ✓ |
| Structured logging only | ✓ |
| Files under 300 lines | ✓ |
| Unit test coverage ≥ 80% | ✓ (84%) |
| Design doc last verified within 30 days | ✓ (2026-04-10) |

### billing — B+

| Check | Status |
|---|---|
| No cross-layer imports | ✓ |
| All external input validated | ✓ |
| Structured logging only | ✓ |
| Files under 300 lines | ⚠ 1 file at 312 lines (`billing/src/service/subscription.ts`) |
| Unit test coverage ≥ 80% | ✓ (81%) |
| Design doc last verified within 30 days | ✓ |

**Active cleanup PR:** #1042 (split subscription.ts — auto-mergeable)

### users — C

| Check | Status |
|---|---|
| No cross-layer imports | ✓ |
| All external input validated | ✗ 2 routes missing Zod schema (`users/src/runtime/admin.ts:48,92`) |
| Structured logging only | ✗ 3 console.log usages |
| Files under 300 lines | ✗ 2 files |
| Unit test coverage ≥ 80% | ✗ 71% |
| Design doc last verified within 30 days | ✗ (last: 2026-03-01) |

**Open cleanup PR:** None — background agent will open one on next run

## Golden Principles compliance

See `docs/references/golden-principles.md` for definitions.

| Principle | Compliant Domains | Violating Domains |
|---|---|---|
| Parse at the boundary | auth, billing, notifications | users |
| No cross-layer imports | all | — |
| Structured logging only | auth, billing, shared | users, notifications (2 files) |
| Shared utilities, not duplicates | auth, shared | billing (1 duplicated util) |
| Design docs within 30-day freshness | auth, billing, notifications | users |
```

---

### Golden Principles Document

```markdown
# docs/references/golden-principles.md

<!-- Last verified: YYYY-MM-DD -->

These are the 6–8 non-negotiable rules that define good code in this codebase.
They are enforced by linters, CI checks, and background cleanup agents.
Each principle has a measurable check — "good" is binary, not subjective.

## 1. Parse at the boundary

**Rule:** All external input (HTTP request body/params/query, environment variables, 
database results, external API responses) must be validated through a Zod schema at 
the point it enters the system.

**Measurable check:** No `req.body.<field>` access without a `.parse()` ancestor in the 
call chain (lint rule: no-unvalidated-input).

**Why:** Unvalidated input is the most common source of runtime errors and security 
vulnerabilities. Agents that generate code without validation produce inconsistent output 
because the failure mode is non-deterministic.

## 2. Structured logging only

**Rule:** No `console.log`, `console.warn`, `console.error` — only `logger` from 
`@app/providers/logger`. All log calls include a structured object as the first argument.

**Measurable check:** ESLint no-console rule passes across all source files.

**Why:** Agents query logs via LogQL. Unstructured logs are queryable only by text 
matching — unreliable for automated feedback loops.

## 3. Layers go left to right only

**Rule:** Within each domain, imports follow `types → config → repo → service → runtime → ui`. 
No backwards imports.

**Measurable check:** `pnpm lint:deps` passes with zero violations.

## 4. Shared utility, not duplicated

**Rule:** If the same function appears in two or more domain packages, it belongs in 
`packages/shared/utils/`. Duplication is a violation.

**Measurable check:** Automated duplicate detection scan (run by cleanup agent) finds 
zero function bodies duplicated across domain boundaries.

## 5. Files under 300 lines

**Rule:** No source file exceeds 300 lines. If a file is growing, it has multiple 
responsibilities and should be split.

**Measurable check:** ESLint max-lines rule with limit 300 passes across all source files.

## 6. Design docs within 30 days

**Rule:** Every domain's design doc must have its `last verified` date updated within 
the past 30 days. Stale docs are treated as missing docs.

**Measurable check:** CI doc-freshness lint check passes.

## 7. External calls only in Repo layer

**Rule:** HTTP calls, database queries, and file system access only occur in `*/src/repo/**`. 
Service and Runtime layers never make I/O calls directly.

**Measurable check:** Lint rule no-direct-io passes in service/ and runtime/ directories.
```

---

### Background Cleanup Agent Task Definition

```markdown
# docs/exec-plans/active/background-cleanup.md

## Status: RECURRING (runs weekly on Monday 09:00 UTC)

## Purpose

Scan the codebase for deviations from golden principles, update QUALITY_SCORE.md,
and open targeted refactoring PRs for violations that can be fixed programmatically.

## Scope constraints

- PRs opened by this task must be < 50 lines changed
- PRs must be auto-mergeable (all CI checks pass) — do NOT open a PR that needs human review
- Do NOT change business logic — only enforce structural/style invariants
- One PR per violation type per domain (do not bundle unrelated fixes)

## Steps

### Step 1: Compute quality scores
Run `bin/compute-quality-scores.sh` and capture the output.
Compare against current QUALITY_SCORE.md.

### Step 2: Update QUALITY_SCORE.md
If any scores have changed:
- Update the relevant rows in the Summary table
- Update the per-domain detail section
- Update the `Last updated` timestamp
- Open a PR: `chore: update quality scores (<date>)` — this PR is always auto-mergeable

### Step 3: Detect fixable violations
For each violation found:
- Is the fix < 50 lines? → proceed
- Is the fix purely structural (no business logic change)? → proceed
- Does the fix pass all CI checks? → proceed
Otherwise: add to the backlog in `docs/exec-plans/tech-debt/` and stop here for that violation.

### Step 4: Open one PR per fixable violation
PR title: `chore(<domain>): fix <violation-type> violation (<count> occurrences)`
PR body must include:
- Which golden principle is violated
- List of files changed
- Confirmation that all CI checks pass
- `closes #<issue-number>` if an issue exists

### Step 5: Detect stale design docs
Check each docs/design-docs/*.md for last-verified date older than 30 days.
For each stale doc, open a PR that:
- Adds a ⚠ STALE warning comment at the top of the file
- Adds an issue reference in the PR body requesting human review of the doc's substance
do NOT edit the content of the doc — only add the stale marker.

## When to escalate

If any cleanup step would require changing more than 50 lines,
create an issue labelled `tech-debt` with title:
`[TECH-DEBT] <domain>: <description of violation> — needs targeted plan`
and move on without opening a PR.
```

### Doc Freshness Lint Rule

```bash
#!/usr/bin/env bash
# bin/check-doc-freshness.sh
# Fails if any design doc has a last-verified date older than MAX_AGE_DAYS.
# Output: JSON array of stale documents.

MAX_AGE_DAYS="${MAX_AGE_DAYS:-30}"
STALE=()
NOW=$(date +%s)

for doc in docs/design-docs/*.md; do
  # Extract "Last verified: YYYY-MM-DD" from the doc
  LAST_VERIFIED=$(grep -oP 'Last verified:\s*\K\d{4}-\d{2}-\d{2}' "$doc" || echo "")

  if [ -z "$LAST_VERIFIED" ]; then
    STALE+=("{\"file\":\"$doc\",\"reason\":\"missing last-verified date\"}")
    continue
  fi

  DOC_TS=$(date -d "$LAST_VERIFIED" +%s 2>/dev/null || date -j -f "%Y-%m-%d" "$LAST_VERIFIED" +%s)
  AGE_DAYS=$(( (NOW - DOC_TS) / 86400 ))

  if [ "$AGE_DAYS" -gt "$MAX_AGE_DAYS" ]; then
    STALE+=("{\"file\":\"$doc\",\"reason\":\"last verified ${AGE_DAYS} days ago (max ${MAX_AGE_DAYS})\",\"last_verified\":\"$LAST_VERIFIED\"}")
  fi
done

if [ ${#STALE[@]} -gt 0 ]; then
  STALE_JSON=$(IFS=','; echo "${STALE[*]}")
  cat <<EOF
{
  "status": "fail",
  "stale_docs": [$STALE_JSON],
  "fix": "Update the 'Last verified' date in each stale doc after reviewing its accuracy. Run: date +%Y-%m-%d"
}
EOF
  exit 1
else
  echo '{"status":"pass","stale_docs":[]}'
fi
```

---

## Phase 7 — Autonomous Delivery Metrics

### Harness Maturity Scorecard — Populated Template

```markdown
# Harness Maturity Scorecard

<!-- Updated: YYYY-MM-DD | Assessed by: <agent or human> -->

## Scoring: 0 = absent, 1 = partial, 2 = fully implemented, 3 = optimised

| Dimension | Score | Evidence | Gaps |
|---|---|---|---|
| **Repo knowledge layer** | ? | AGENTS.md exists, <N> design docs | Missing: exec plans, product specs templated |
| **Tooling (observability)** | ? | LogQL/PromQL/TraceQL queryable | Missing: TraceQL integration |
| **Tooling (env isolation)** | ? | bin/boot.sh exists | Missing: automated teardown after merge |
| **Enforcement (linters)** | ? | <N> custom rules | Missing: complexity limits |
| **Enforcement (layer deps)** | ? | depcruise running in CI | — |
| **CI speed** | ? | First signal in <N> min | Target: < 5 min |
| **Feedback (pre-PR check)** | ? | bin/check.sh exists and passes | — |
| **Feedback (agent review)** | ? | Review rubric doc exists | Missing: automated reviewer agent wired to CI |
| **Escalation criteria** | ? | AGENTS.md section exists | Missing: issue template |
| **Quality scoring** | ? | QUALITY_SCORE.md exists | Missing: auto-update agent |
| **Golden principles** | ? | <N> principles documented | Missing: measurable checks for all |
| **Entropy management** | ? | Cleanup task defined | Missing: scheduled run wired |
| **Doc freshness** | ? | check-doc-freshness.sh exists | Missing: wired to CI |

## Maturity level

**0–10 points:** Harness Beginner — manual intervention required for most tasks  
**11–20 points:** Harness Operational — agents can do scoped tasks reliably  
**21–30 points:** Harness Mature — agents complete full features autonomously  
**31–36 points:** Harness Expert — background agents maintain quality continuously

**Current level:** <compute score × 3 above>
```

### Key Harness Metrics

Track these weekly to measure harness improvement objectively:

| Metric | Formula | Target | Measurement |
|---|---|---|---|
| **Human interventions per PR** | Count of human PR comments requiring code change / total PRs merged | < 0.2 (1 in 5 PRs) | GitHub PR analytics |
| **PR throughput** | Count of agent-generated PRs merged per week | > 20/week (mature), > 50/week (expert) | GitHub metrics |
| **Time to first CI signal** | Median time from PR open to first CI check result | < 5 minutes | GitHub Actions timing |
| **Flake rate** | % of CI runs with a flaky (non-deterministic) failure | < 2% | CI dashboard |
| **Entropy delta** | Change in QUALITY_SCORE.md aggregate grade week-over-week | Non-negative (stable or improving) | QUALITY_SCORE.md history |
| **Stale doc count** | Docs flagged as stale by check-doc-freshness.sh | 0 | CI check history |
| **Cost per feature** | Model API cost per completed feature PR (USD) | Trending down over harness improvements | API cost dashboard + PR count |
| **Gap classification rate** | % of agent failures classified by root cause within 24h | > 90% | Gap analysis log |

### Gap Analysis Framework

After every full autonomous delivery exercise, classify every human intervention:

```markdown
# Gap Analysis: <Feature Name> — <Date>

## Summary
- Total human interventions: N
- PRs opened: N
- Completion: FULL / PARTIAL (explain)

## Interventions by category

| # | Intervention | Category | Harness Fix |
|---|---|---|---|
| 1 | Added Zod schema that agent missed | Missing doc | Add to design doc that all external input needs Zod |
| 2 | Corrected layer violation not caught by linter | Missing lint rule | Add lint rule for this import pattern |
| 3 | Product decision about error message wording | Genuine ambiguity | Accept (not fixable by harness) |
| 4 | Agent couldn't query DB state mid-test | Missing tool | Add query_db(sql) tool to tool catalogue |

## Category breakdown
- Missing tool: N
- Missing doc: N
- Missing lint rule / enforcement: N
- Genuine ambiguity (product): N
- Model limitation: N

## Harness improvements open from this exercise
- [ ] PR #<N>: add Zod requirement to users design doc
- [ ] PR #<N>: add layer import lint rule for X pattern
- [ ] PR #<N>: add query_db tool to tool catalogue

## Target for next run
Reduce total interventions from N to N × 0.5 (50% reduction).
```

---

## Phase 5–7 Validation Checklist

```
Feedback loops
  [ ] bin/pre-pr-check.sh exists, exits 0/1, outputs valid JSON
  [ ] pre-pr-check.sh covers: lint, typecheck, layer-deps, unit tests, build, E2E smoke, doc freshness
  [ ] Agent review rubric document exists in docs/references/
  [ ] Review rubric covers: architecture, code quality, tests, observability, documentation, PR hygiene
  [ ] Escalation criteria exist in AGENTS.md (summarised) and docs/references/ (full)
  [ ] Escalation criteria cover: product decisions, architecture boundaries, security, unknown territory

Entropy management
  [ ] QUALITY_SCORE.md exists with per-domain scores and trend indicators
  [ ] docs/references/golden-principles.md exists with 5+ principles
  [ ] Each golden principle has a measurable check (tool or lint rule)
  [ ] Background cleanup agent task definition exists in docs/exec-plans/active/
  [ ] Cleanup task constraints: < 50 lines per PR, no business logic changes
  [ ] bin/check-doc-freshness.sh exists and is wired into bin/pre-pr-check.sh
  [ ] Doc freshness rule runs in CI

Autonomous delivery
  [ ] Harness maturity scorecard populated with evidence
  [ ] At least one full autonomous delivery run documented with gap analysis
  [ ] Each gap classified (missing tool / missing doc / missing lint / genuine ambiguity)
  [ ] Harness improvement PR opened for every non-genuine-ambiguity gap
  [ ] Key metrics tracked weekly: interventions/PR, throughput, entropy delta
```
