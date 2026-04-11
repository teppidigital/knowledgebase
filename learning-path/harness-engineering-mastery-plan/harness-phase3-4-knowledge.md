# Harness Engineering Phase 3–4 Deep Knowledge
## Agent Legibility, Tooling, Enforcement & CI Pipeline

> Covers Weeks 6–11 of the Harness Engineering Mastery Plan.  
> Hands-on implementation: observability, git worktrees, tool catalogues, custom linters, dependency enforcement, CI pipeline design.

---

## Phase 3 — Observability Stack for Agents

### Why Agents Need Real Observability

An agent without observability access has only two feedback signals: "it compiled" and "the test passed." This is catastrophic for anything involving:
- Performance (startup time, latency, memory)
- Integration correctness (downstream service behaviour)
- Regression detection (metric drift after a code change)

The goal: an agent must be able to run a LogQL, PromQL, or TraceQL query from a tool call and get a structured numeric or log result back.

### Local Observability Stack — Reference Setup

```
┌─────────────────────────────────────────────────────────┐
│  AGENT TOOL CALLS                                       │
│  query_logs(logql)  query_metrics(promql)  query_traces │
└──────────┬─────────────────┬──────────────────┬────────┘
           │                 │                  │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │    Loki     │   │  Prometheus │   │    Tempo    │
    │  (logs)     │   │  (metrics)  │   │  (traces)   │
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           │                 │                  │
    ┌──────▼──────────────────▼──────────────────▼──────┐
    │              Grafana (port 3001)                   │
    │         Dashboards + API query endpoint            │
    └───────────────────────────────────────────────────┘
           ↑                 ↑                  ↑
    ┌──────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
    │  Promtail   │   │  OTEL       │   │  OTEL       │
    │ (log ship)  │   │  Collector  │   │  Collector  │
    └──────┬──────┘   └──────┬──────┘   └─────────────┘
           │                 │
    ┌──────▼─────────────────▼──────────────────────────┐
    │              Your Application (per worktree)       │
    │  Structured JSON logs → stdout (Promtail scrapes)  │
    │  OTEL SDK → traces and metrics → Collector         │
    └───────────────────────────────────────────────────┘
```

### Docker Compose for Local Observability

```yaml
# docker-compose.observability.yml
version: "3.9"
services:
  loki:
    image: grafana/loki:2.9.0
    ports: ["3100:3100"]
    volumes:
      - ./infra/loki/config.yaml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log
      - ./infra/promtail/config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml

  prometheus:
    image: prom/prometheus:v2.47.0
    ports: ["9090:9090"]
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  tempo:
    image: grafana/tempo:2.2.0
    ports: ["3200:3200", "4317:4317"]  # HTTP + OTLP gRPC
    volumes:
      - ./infra/tempo/config.yaml:/etc/tempo/config.yaml

  grafana:
    image: grafana/grafana:10.1.0
    ports: ["3001:3000"]
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
```

### Agent-Callable LogQL Queries

Agents invoke these via an MCP tool or a local API wrapper:

```bash
# Query Loki via HTTP API
curl -s 'http://localhost:3100/loki/api/v1/query_range' \
  --data-urlencode 'query={service="myapp"} |= "ERROR"' \
  --data-urlencode 'start=1h ago' \
  | jq '.data.result[].values[][1]'
```

**Standard queries to expose as tools:**

```
# Service startup time
{service="myapp"} |= "application started" | json | line_format "{{.duration_ms}}ms"

# Error rate in last 5 minutes
count_over_time({service="myapp"} |= "ERROR" [5m])

# Structured error log with stack trace
{service="myapp"} | json | severity="ERROR" | line_format "{{.message}}\n{{.stack}}"

# Slow database queries
{service="myapp"} | json | db_duration_ms > 100 | line_format "{{.query}} took {{.db_duration_ms}}ms"
```

**PromQL queries:**

```
# Startup duration (histogram p95)
histogram_quantile(0.95, rate(app_startup_duration_seconds_bucket[5m]))

# HTTP error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Memory usage above threshold
process_resident_memory_bytes / 1024 / 1024 > 512
```

**TraceQL (Tempo):**

```
# Traces slower than 2 seconds
{ .http.method = "GET" && duration > 2s }

# Failed traces with specific span
{ status = error && name = "db.query" }

# Traces for a specific user journey
{ resource.service.name = "api" && .user.id = "test-user-1" }
```

### Structured Logging Setup (Application Side)

Agents can only query logs they can parse. JSON structured logging is mandatory.

```typescript
// packages/providers/logger/index.ts
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: {
    level: (label) => ({ severity: label.toUpperCase() }),
  },
  base: {
    service: process.env.SERVICE_NAME ?? 'unknown',
    version: process.env.APP_VERSION ?? 'unknown',
  },
})

// Usage — enforced by custom lint rule:
logger.info({ userId, duration_ms: elapsed }, 'user login successful')
// NOT: console.log(`User ${userId} logged in`)
```

---

## Git Worktree Pattern — One App Per Branch

### Why Worktrees Matter

If the agent has only one running app instance, every branch's code runs against the same state. When two concurrent agent tasks are running, they collide. Worktrees give each agent task its own isolated environment.

```
main branch          → port 4000, DB: app_main
feature/login-flow   → port 4001, DB: app_feature_login_flow
fix/billing-crash    → port 4002, DB: app_fix_billing_crash
```

### Boot Script — Full Template

```bash
#!/usr/bin/env bash
# bin/boot.sh <branch-name>
# Boots a fully isolated app instance for the given branch.
# Usage: bin/boot.sh feature/my-feature

set -euo pipefail

BRANCH="${1:-$(git branch --show-current)}"
SAFE_BRANCH="${BRANCH//\//_}"  # replace / with _ for dir names
WORKTREE_DIR=".worktrees/${SAFE_BRANCH}"

# Port allocation: hash branch name to a port in range 4001-4999
PORT=$((4001 + $(echo -n "$BRANCH" | cksum | awk '{print $1}') % 998))
DB_NAME="app_${SAFE_BRANCH}"

echo "→ Branch: $BRANCH"
echo "→ Port:   $PORT"
echo "→ DB:     $DB_NAME"

# 1. Create worktree if it doesn't exist
if [ ! -d "$WORKTREE_DIR" ]; then
  git worktree add "$WORKTREE_DIR" "$BRANCH"
fi

# 2. Install deps if needed
if [ ! -d "$WORKTREE_DIR/node_modules" ]; then
  (cd "$WORKTREE_DIR" && pnpm install --frozen-lockfile)
fi

# 3. Create isolated database
psql postgres -tc "SELECT 1 FROM pg_database WHERE datname = '$DB_NAME'" | grep -q 1 \
  || psql postgres -c "CREATE DATABASE $DB_NAME"

# 4. Run migrations against isolated DB
(cd "$WORKTREE_DIR" && DATABASE_URL="postgresql://localhost/$DB_NAME" pnpm db:migrate)

# 5. Start the app with isolated config
(cd "$WORKTREE_DIR" && \
  PORT="$PORT" \
  DATABASE_URL="postgresql://localhost/$DB_NAME" \
  LOG_LEVEL="debug" \
  SERVICE_NAME="app-${SAFE_BRANCH}" \
  pnpm start &)

# 6. Wait for health check
echo "→ Waiting for startup..."
for i in {1..30}; do
  curl -sf "http://localhost:$PORT/health" > /dev/null 2>&1 && break
  sleep 1
done

echo "✓ App running at http://localhost:$PORT"
echo "✓ Logs streaming in Loki under {service=\"app-${SAFE_BRANCH}\"}"
echo "✓ Metrics at http://localhost:9090 — label: branch=\"${SAFE_BRANCH}\""
```

### Teardown Script

```bash
#!/usr/bin/env bash
# bin/teardown.sh <branch-name>
BRANCH="${1:?Usage: bin/teardown.sh <branch>}"
SAFE_BRANCH="${BRANCH//\//_}"

# Kill the process
pkill -f "PORT=.*$SAFE_BRANCH" || true

# Remove worktree
git worktree remove ".worktrees/${SAFE_BRANCH}" --force || true

# Drop DB (only if ephemeral)
psql postgres -c "DROP DATABASE IF EXISTS app_${SAFE_BRANCH}"
echo "✓ Teardown complete for $BRANCH"
```

---

## Chrome DevTools Protocol — Agent Browser Control

CDP lets an agent drive a browser programmatically. Key agent use cases:

| Use Case | CDP Operation |
|---|---|
| Verify UI renders correctly | `Page.captureScreenshot` |
| Interact with a form | `Runtime.evaluate` with document.querySelector + click |
| Check for JS console errors | Subscribe to `Runtime.consoleAPICalled` |
| Validate navigation | `Page.navigate` + wait for `Page.loadEventFired` |
| Take a DOM snapshot | `DOM.getDocument` + `DOM.getOuterHTML` |

### CDP Wrapper for Tool Use

```typescript
// tools/browser.ts — MCP-compatible tool definition
import CDP from 'chrome-remote-interface'

export async function screenshot(url: string): Promise<string> {
  const client = await CDP({ port: 9222 })
  const { Page } = client
  await Page.enable()
  await Page.navigate({ url })
  await Page.loadEventFired()
  const { data } = await Page.captureScreenshot({ format: 'png' })
  await client.close()
  return data  // base64 PNG
}

export async function checkForConsoleErrors(url: string): Promise<string[]> {
  const errors: string[] = []
  const client = await CDP({ port: 9222 })
  const { Runtime, Page } = client
  await Runtime.enable()
  Runtime.consoleAPICalled(({ type, args }) => {
    if (type === 'error') errors.push(args.map(a => a.value).join(' '))
  })
  await Page.navigate({ url })
  await Page.loadEventFired()
  await client.close()
  return errors
}
```

Launch Chrome with CDP enabled:
```bash
google-chrome --remote-debugging-port=9222 --headless --no-sandbox &
```

---

## Tool Catalogue Document Template

Every tool available to the agent must be documented. This document lives at `docs/references/tool-catalogue.md`.

```markdown
# Agent Tool Catalogue

<!-- Last verified: YYYY-MM-DD -->

## Shell tools

### bin/check.sh
**Purpose:** Run all pre-PR checks  
**Inputs:** None  
**Outputs:** Exit 0 (pass) or non-zero with structured JSON error list on stdout  
**When to use:** Always before opening a PR  
**Failure modes:** Exits 1 with JSON; parse `.errors` array for lint violations

### bin/boot.sh <branch>
**Purpose:** Boot isolated app instance for a branch  
**Inputs:** Branch name (string)  
**Outputs:** App URL printed on stdout; background process running  
**When to use:** Before UI validation or integration testing  
**Failure modes:** Exits non-zero if port conflict; check `.port` collision

## Observability tools

### query_logs(logql: string, window: string)
**Purpose:** Query Loki for structured log data  
**Inputs:** LogQL query string, time window (e.g. "5m", "1h")  
**Outputs:** JSON array of log lines with timestamps  
**Example:** `query_logs('{service="api"} |= "ERROR"', '5m')`

### query_metrics(promql: string)
**Purpose:** Query Prometheus for metric values  
**Inputs:** PromQL expression  
**Outputs:** JSON with metric name, labels, value  
**Example:** `query_metrics('rate(http_requests_total[5m])')`

## Git tools

### gh pr create --draft --title "<title>" --body "<body>"
**Purpose:** Open a draft PR  
**When to use:** After bin/check.sh passes; include evidence in body  
**Failure modes:** Fails if branch has no upstream; push first with `git push -u origin HEAD`
```

---

## Phase 4 — Custom Linter Design

### The Lint Error Message IS the Harness

The lint rule and its error message are two distinct systems. The **rule** enforces the invariant. The **message** tells the agent how to fix it. Writing a lint rule without a clear, actionable error message is a harness failure.

**Template for every lint error message:**

```
[RULE-NAME] <Plain-English description of what is wrong>.
→ Fix: <Specific action to take>
→ See: <Link to design doc or reference>
→ Example:
  BAD:  <bad code snippet>
  GOOD: <good code snippet>
```

**Bad message (generic):**
```
Unexpected console statement. (no-console)
```

**Good message (harness-quality):**
```
[NO-CONSOLE] Direct console.log is not structured and cannot be queried by agents.
→ Fix: Use the structured logger from @app/providers/logger
→ See: docs/design-docs/logging.md
→ Example:
  BAD:  console.log(`User ${id} logged in`)
  GOOD: logger.info({ userId: id }, 'user login successful')
```

### ESLint Custom Plugin — Anatomy

```javascript
// eslint-plugin-harness/rules/no-unvalidated-input.js
/**
 * Rule: no-unvalidated-input
 * Enforces parse-don't-validate at layer boundaries.
 * Input from HTTP requests, CLI args, or external APIs must pass through a Zod schema.
 */
module.exports = {
  meta: {
    type: 'problem',
    docs: {
      description: 'External input must be validated with a Zod schema at the boundary',
    },
    messages: {
      noRawRequest: [
        '[NO-UNVALIDATED-INPUT] req.body is used without Zod validation.',
        '→ Fix: Parse req.body through a Zod schema in the runtime layer.',
        '→ See: docs/design-docs/validation.md',
        '→ Example:',
        '  BAD:  const { userId } = req.body',
        '  GOOD: const { userId } = CreateUserSchema.parse(req.body)',
      ].join('\n'),
    },
    schema: [],
  },

  create(context) {
    return {
      MemberExpression(node) {
        // Detect req.body, req.params, req.query without .parse() ancestor
        if (
          node.object.name === 'req' &&
          ['body', 'params', 'query'].includes(node.property.name)
        ) {
          const parent = node.parent
          // Allow if it's the argument to a .parse() call: Schema.parse(req.body)
          const isBeingParsed =
            parent.type === 'CallExpression' &&
            parent.callee.type === 'MemberExpression' &&
            parent.callee.property.name === 'parse'

          if (!isBeingParsed) {
            context.report({ node, messageId: 'noRawRequest' })
          }
        }
      },
    }
  },
}
```

### Three Essential Custom Lint Rules — Specification

#### Rule 1: Structured Logging Only
```
Name: no-console
File: eslint-plugin-harness/rules/no-console.js
Trigger: Any call to console.log, console.warn, console.error, console.debug
Message: Use logger from @app/providers/logger — explains why + shows example
Exceptions: Allow in test files (*.test.ts, *.spec.ts)
```

#### Rule 2: File Size Limit
```
Name: max-file-lines
File: eslint-plugin-harness/rules/max-file-lines.js
Trigger: Any file exceeding MAX_LINES (default 300) lines
Message: File exceeds 300 lines — split by responsibility; see ARCHITECTURE.md layering
Implementation: Use Program:exit to count total lines in context.getSourceCode().lines
```

#### Rule 3: Cross-Layer Import Prohibition
```
Name: no-cross-layer-import (OR use dependency-cruiser — see below)
Trigger: Import from a "righter" layer within the same domain
Message: Service layer cannot import from Runtime — this reverses the dependency direction.
         See ARCHITECTURE.md for layer ordering.
         Move the shared logic to the Service layer or a Provider.
```

### Ruff Plugin (Python equivalent)

```python
# ruff_plugins/no_direct_db.py
"""
Custom Ruff rule: NDB001 — No direct database access outside repository layer.
"""
import ast
from ruff import ASTVisitor

class NoDirectDBAccess(ASTVisitor):
    message = (
        "NDB001: Direct database call outside repository layer.\n"
        "→ Fix: Move database access to the repository module.\n"
        "→ See: docs/design-docs/data-access.md"
    )

    def visit_Call(self, node: ast.Call) -> None:
        # Detect sqlalchemy calls (session.execute, session.query) outside *_repo.py files
        if hasattr(node.func, 'attr') and node.func.attr in ('execute', 'query'):
            if 'repo' not in self.filename:
                self.report(node, self.message)
```

---

## Layer Dependency Enforcement

### dependency-cruiser (JavaScript/TypeScript)

```javascript
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: 'no-service-to-runtime',
      comment: 'Service layer must not import from Runtime layer — layers go left to right only',
      severity: 'error',
      from: { path: '/src/service/' },
      to: { path: '/src/runtime/' },
    },
    {
      name: 'no-repo-to-service',
      comment: 'Repository layer must not import from Service layer',
      severity: 'error',
      from: { path: '/src/repo/' },
      to: { path: '/src/service/' },
    },
    {
      name: 'no-cross-domain',
      comment: 'Domains may not import directly from each other — use the public index.ts only',
      severity: 'error',
      from: { path: '^packages/([^/]+)/' },
      to: {
        path: '^packages/([^/]+)/src/(?!index\\.ts)',
        // The captured group ensures same-domain imports are allowed
        pathNot: '^packages/$1/',
      },
    },
    {
      name: 'no-providers-inward',
      comment: 'Providers package must not import from domain packages',
      severity: 'error',
      from: { path: '^packages/providers/' },
      to: { path: '^packages/(?!providers|shared)' },
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    reporterOptions: {
      text: {
        highlightViolations: true,
      },
    },
  },
}
```

Run in CI: `npx depcruise --config .dependency-cruiser.js packages/*/src`

### ArchUnit (JVM — Kotlin/Java)

```kotlin
// src/test/kotlin/ArchitectureTest.kt
import com.tngtech.archunit.core.importer.ClassFileImporter
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses
import org.junit.Test

class LayerArchitectureTest {
    private val classes = ClassFileImporter().importPackages("com.example")

    @Test
    fun `service layer must not depend on runtime layer`() {
        noClasses()
            .that().resideInAPackage("..service..")
            .should().dependOnClassesThat().resideInAPackage("..runtime..")
            .`as`("Service layer cannot import Runtime — see ARCHITECTURE.md")
            .check(classes)
    }

    @Test
    fun `repository layer must not depend on service layer`() {
        noClasses()
            .that().resideInAPackage("..repo..")
            .should().dependOnClassesThat().resideInAPackage("..service..")
            .`as`("Repo layer cannot import Service — see ARCHITECTURE.md")
            .check(classes)
    }
}
```

### import-linter (Python)

```ini
# setup.cfg
[importlinter]
root_package = myapp
include_external_packages = True

[importlinter:contract:service-cannot-import-repo]
name = Service layer cannot import Repository layer directly
type = forbidden
source_modules =
    myapp.service
forbidden_modules =
    myapp.adapters.db
    myapp.adapters.http
```

---

## CI Pipeline — Agent-Optimised Design

### Design Goals for Agent Workflows

| Goal | Why It Matters | How to Achieve |
|---|---|---|
| First signal < 5 minutes | Over 5 min, agents start second-guessing and re-running | Run lint + unit tests in parallel; skip slow integration tests on draft PRs |
| Lint errors as structured JSON | Agents parse JSON; prose error pages require HTML parsing | `eslint --format json`, `ruff --output-format json` |
| Flake detection and quarantine | Flaky tests create false negatives; agents retry unnecessarily | Track test history; auto-label flaky tests; skip quarantined tests |
| Non-blocking merge for clean PRs | Human approval gates kill throughput when all checks pass | Require checks pass; make human review optional (not required) for non-security PRs |

### GitHub Actions Pipeline Template

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    types: [opened, synchronize, ready_for_review]

jobs:
  fast-checks:
    name: Lint + Unit Tests (< 3 min)
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v3
        with: { version: 9 }

      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }

      - run: pnpm install --frozen-lockfile

      # Lint — output JSON for agent parseability
      - name: Lint
        run: |
          pnpm lint --format json --output-file lint-results.json || true
          # Exit non-zero if any errors (not warnings)
          jq -e '[.[] | .messages[] | select(.severity == 2)] | length == 0' lint-results.json

      - name: Upload lint results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: lint-results.json

      # Dependency layers
      - name: Check layer dependencies
        run: pnpm lint:deps

      # Unit tests — affected packages only on PRs
      - name: Unit tests
        run: pnpm test:affected --reporter=json --outputFile=test-results.json

      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.json

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    timeout-minutes: 15
    # Only run on non-draft PRs (skip for WIP agent PRs)
    if: github.event.pull_request.draft == false
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:migrate
        env:
          DATABASE_URL: postgresql://postgres:test@localhost/test
      - name: Integration tests
        run: pnpm test:integration --reporter=json
        env:
          DATABASE_URL: postgresql://postgres:test@localhost/test

  # Flake detection job — runs in background, does not block merge
  flake-report:
    name: Flake detection
    runs-on: ubuntu-latest
    if: always()
    needs: [fast-checks, integration-tests]
    steps:
      - name: Report flaky tests
        run: |
          # Compare test results against historical baseline
          # Flag tests that failed this run but passed in last 5 runs
          echo "Flake analysis: see docs/references/flake-quarantine.md"
```

### Lint Output Format for Agent Consumption

ESLint JSON output that agents can parse:

```json
[
  {
    "filePath": "/app/packages/users/src/service.ts",
    "messages": [
      {
        "ruleId": "no-console",
        "severity": 2,
        "message": "[NO-CONSOLE] Direct console.log is not structured...\n→ Fix: Use logger from @app/providers/logger",
        "line": 42,
        "column": 5,
        "source": "console.log(`User ${id} logged in`)"
      }
    ],
    "errorCount": 1,
    "warningCount": 0
  }
]
```

The agent reads `messages[].message` and `messages[].source` to understand exactly what to fix and where.

---

## Phase 3–4 Validation Checklist

```
Observability
  [ ] Docker Compose observability stack starts with one command
  [ ] Agent can query logs via LogQL tool call and get structured result
  [ ] Agent can query metrics via PromQL and get numeric result
  [ ] Startup time can be verified with a single LogQL query
  [ ] Application emits structured JSON logs (no console.log in source)

Tooling
  [ ] bin/boot.sh spins up isolated app instance per branch
  [ ] Each worktree has its own database and port
  [ ] bin/check.sh runs all pre-PR checks in < 5 minutes
  [ ] CDP available for browser control on a known port
  [ ] Tool catalogue document exists in docs/references/

Enforcement
  [ ] At least 3 custom lint rules exist with remediation messages
  [ ] Every lint error message: explains the problem, gives the fix, shows an example
  [ ] Layer dependency enforcement is running in CI (depcruise / archunit / import-linter)
  [ ] Lint output format is structured JSON (not plain text)

CI Pipeline
  [ ] First CI signal arrives in < 5 minutes
  [ ] Draft PRs skip slow integration tests
  [ ] Lint results uploaded as JSON artifact on failure
  [ ] No single flaky test can block agent-generated PRs indefinitely
```
