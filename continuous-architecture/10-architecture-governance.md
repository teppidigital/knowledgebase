# Architecture Governance

## Category

Continuous Architecture — Governance

## Context

Architecture governance is the set of mechanisms that ensure architectural decisions are made consistently, communicated effectively, and adhered to over time. The failure mode of traditional governance is being too heavy: gating delivery through approval committees that slow teams down, produce resentment, and are eventually bypassed.

Continuous architecture governance is *lightweight by design*: it shifts accountability from a review board to the teams themselves, uses automated fitness functions as first-line guardrails, and reserves human review for high-impact or cross-team decisions only.

### Heavy vs Lightweight Governance

| Dimension | Heavy governance | Lightweight governance |
|---|---|---|
| **Decision authority** | Central review board | Teams decide; architect guides |
| **Review trigger** | All significant changes | Only cross-team or high-risk changes |
| **Primary mechanism** | Approval gate | ADR + fitness function |
| **Latency** | Weeks (scheduling, committee) | Days (async ADR review) |
| **Accountability** | Committee approved it | Team owns it with architect endorsement |
| **Compliance evidence** | Meeting minutes | ADR in version control + FF results in CI |
| **Failure detection** | Retrospective audit | Continuous (fitness function in every build) |

## Pros

- Scales with organisation size without creating bottlenecks
- Accountability stays with the teams that build and operate the system
- Fitness functions catch violations faster and cheaper than reviews
- Architecture guild creates cross-team consistency without mandates
- ADRs provide an auditable governance trail at low cost

## Cons

- Requires cultural maturity: teams must own their architectural decisions responsibly
- Lightweight governance can miss risks if fitness functions have gaps
- Architecture guild effectiveness depends on consistent participation
- High-stakes decisions (compliance, security, enterprise standards) may still require formal approval

## Design Diagram

```mermaid
flowchart TD
    subgraph Automated Layer — Always on
        FF[Fitness Functions<br/>in CI/CD pipeline]
        LINT[Static Analysis<br/>SonarQube / depcruiser]
        SECSC[Security Scanning<br/>CVE / SAST / DAST]
    end

    subgraph Lightweight Review Layer
        ADR[ADR Review<br/>Async — 48h window<br/>Architect + relevant team leads]
        GUILD[Architecture Guild<br/>Weekly forum<br/>Cross-team patterns + decisions]
        RADAR[Technology Radar<br/>Quarterly update]
    end

    subgraph Formal Review — High-stakes only
        ARB[Architecture Review<br/>Cross-org impact<br/>Regulatory / compliance<br/>Major platform decisions]
    end

    FF --> GUILD
    ADR --> GUILD
    GUILD --> RADAR
    RADAR --> ARB
    ARB --> ADR
```

## Code Sample

### Architecture guild agenda template

```markdown
# Architecture Guild — 2026-04-09

**Facilitator**: @bob-jones
**Duration**: 45 minutes
**Format**: Async prep + sync discussion

---

## Standing items (10 min)
- [ ] Review fitness function failures from last week (dashboard link)
- [ ] ADRs submitted since last meeting — any needing broader input?
  - ADR-042: Adopt OpenTelemetry as the observability standard (Payments team)
  - ADR-043: GraphQL Federation for public API layer (API Platform team)

## Main topic (25 min): Tech Debt Triage — Q2 Review
- Debt register review: 3 high-risk items escalated from teams
- Prioritisation vote: which items get capacity in Q2?
- Owners assigned for top 3 paydown items

## Technology Radar update (5 min)
- Items proposed since last quarter:
  - ADOPT: Vitest (replaces Jest — faster, native ESM)
  - TRIAL: Temporal.io (workflow orchestration)
  - HOLD: MongoDB (no new adoption — existing uses maintained)

## Architecture health review (5 min)
- Coupling trend: 3 modules exceeding instability threshold (link to dashboard)
- Fitness function coverage: 2 new FFs added; 1 removed (FF-007 superseded by FF-019)

## AOB / open discussion
```

### Technology Radar — YAML format

```yaml
# technology-radar.yaml
version: "2026-Q2"
updated: "2026-04-01"
owner: "Architecture Guild"

quadrants:
  - name: Languages & Frameworks
    entries:
      - name: TypeScript
        ring: ADOPT
        description: "Default language for all new services"
        since: "2022-Q1"
      - name: Temporal.io
        ring: TRIAL
        description: "Evaluate for workflow orchestration; POC in 2026-Q2"
        since: "2026-Q1"
      - name: CoffeeScript
        ring: HOLD
        description: "No new usage; existing code migrated to TypeScript on touch"
        since: "2024-Q3"

  - name: Platforms & Infrastructure
    entries:
      - name: Kubernetes (EKS)
        ring: ADOPT
        description: "Standard container orchestration platform"
        since: "2021-Q3"
      - name: AWS Lambda
        ring: TRIAL
        description: "Evaluating for async background tasks; not yet standard"
        since: "2025-Q4"
      - name: EC2 manual provisioning
        ring: HOLD
        description: "No new EC2 instances; migrate to EKS on opportunity"
        since: "2023-Q1"

  - name: Techniques
    entries:
      - name: Architecture Fitness Functions
        ring: ADOPT
        description: "All new services must include fitness functions in CI"
        since: "2024-Q2"
      - name: Architecture Decision Records
        ring: ADOPT
        description: "Required for all cross-team or QA-level decisions"
        since: "2023-Q3"
      - name: Event Sourcing
        ring: TRIAL
        description: "Used in audit domain; evaluate broader applicability"
        since: "2025-Q2"
```

### Lightweight ADR review process

```typescript
// Automated ADR freshness and completeness check
// Runs as a CI step on any PR touching docs/architecture/decisions/

import { readFileSync, readdirSync } from 'fs';

interface AdrCheck {
  file: string;
  violations: string[];
}

const REQUIRED_FIELDS = ['Status', 'Date', 'Deciders', 'Decision Outcome'];
const VALID_STATUSES = ['Proposed', 'Under Review', 'Accepted', 'Rejected', 'Superseded'];

export function lintAdrs(adrDir: string): AdrCheck[] {
  const files = readdirSync(adrDir).filter(f => f.endsWith('.md') && f !== 'README.md');
  return files.map(file => {
    const content = readFileSync(`${adrDir}/${file}`, 'utf-8');
    const violations: string[] = [];

    for (const field of REQUIRED_FIELDS) {
      if (!content.includes(`## ${field}`) && !content.includes(`**${field}**`)) {
        violations.push(`Missing required field: ${field}`);
      }
    }

    const statusMatch = content.match(/\*\*Status\*\*:\s*(\w[\w ]*)/);
    if (statusMatch && !VALID_STATUSES.includes(statusMatch[1].trim())) {
      violations.push(`Invalid status: "${statusMatch[1].trim()}" — must be one of: ${VALID_STATUSES.join(', ')}`);
    }

    return { file, violations };
  });
}
```

## Key Patterns

### Governance Trigger Rules

Define explicitly what requires which level of review:

| Change | Governance level | Mechanism |
|---|---|---|
| Implementation change within one service | None | Code review only |
| Technology choice within one team | Lightweight | ADR posted to guild; 48h async review |
| Cross-team API contract change | Lightweight + notification | ADR + informed all consuming teams |
| New platform capability | Guild discussion | Agenda item; guild endorsement |
| Technology added to radar | Guild vote | Quarterly radar update |
| Enterprise standard change | Formal review | Architecture Review Board; compliance sign-off |
| Third-party vendor contract (≥£100k TCO) | Formal review | ARB + procurement + legal |

### Architecture Review Board (Lightweight Version)

If an ARB is needed, keep it minimal:
- **Composition**: 3–5 people maximum (lead architect, 1–2 domain architects, product/engineering director)
- **Cadence**: On-demand (not monthly standing meeting)
- **Input required**: ADR draft + fitness function plan + risk assessment
- **Decision in**: ≤ 1 week from submission
- **Output**: Decision recorded in the ADR; no separate meeting minutes needed

### Common Governance Anti-Patterns

| Anti-pattern | Symptom | Fix |
|---|---|---|
| **Review theatre** | Reviews take weeks; nothing changes; teams resent the process | Automate what can be automated; reserve human review for genuine cross-cutting risk |
| **Governance bypass** | Teams make decisions without ADRs; drift accumulates | Review fitness functions — they should catch structural drift |
| **Consistency mandating** | Central team mandates technology choices without team input | Technology radar with ring process; teams have input and a path to trial new options |
| **Guild as gossip** | Guild has no agenda; nothing decided; no follow-through | Structured agenda; assigned owners for every action |

## Related Patterns

- [06 — Fitness Functions](./06-fitness-functions.md) — Automated first-line governance
- [07 — ADRs](./07-adrs.md) — The primary governance artefact
- [04 — Architect's Role](./04-architect-role.md) — Architect's role in the governance system
- [09 — Team Topologies](./09-team-topologies.md) — Guild spans all team types
