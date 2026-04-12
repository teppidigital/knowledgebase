# Architecture Runway

## Category

Continuous Architecture — Process

## Context

Architecture runway is the pre-existing technical foundation that allows feature teams to implement upcoming business features without architectural blockers. It is the architecture equivalent of a road: the road must be built before traffic can flow; teams cannot ship features on infrastructure that doesn't exist yet.

In SAFe (Scaled Agile Framework), the runway is explicitly managed as a measure. In continuous architecture, it is a broader concept: any architectural foundation that enables feature delivery. When runway is depleted, feature teams are blocked, make short-term workarounds, or accumulate debt.

### Runway vs Technical Debt

| | Architecture Runway | Technical Debt |
|---|---|---|
| **What it is** | Foundation that *enables* future work | Shortcuts that make future work *harder* |
| **Trend** | Should be topped up continuously | Should trend downward |
| **Impact** | Teams can build on solid ground | Teams slow down; more bugs; more coupling |
| **Build by** | Enabler stories + IP iterations + platform team | Identified; added to debt register; paid down |
| **Visibility** | Runway buffer (sprints ahead) | Debt register; coupling metrics |

### Types of Runway

| Type | Description | Example |
|---|---|---|
| **Platform runway** | Infrastructure capabilities teams need but don't have | Kafka cluster not yet provisioned; secrets management not yet set up |
| **Architecture runway** | Cross-cutting structural foundations | No established service mesh; no API gateway; no standard health endpoint |
| **Domain runway** | Domain model pre-built ahead of feature work | Data model for the new product domain not yet defined |
| **Tooling runway** | Developer toolchain not yet in place | No CI/CD template for new service type; no local dev environment |

## Pros

- Feature teams never blocked on infrastructure they don't own
- Architectural decisions made properly (at LRM) before teams need to act on them
- Platform team has a backlog driven by stream-aligned team needs — not self-directed
- Reduces ad-hoc, in-sprint architectural shortcuts that become debt

## Cons

- Building too far ahead wastes effort on runway that requirements may never demand
- Runway work competes with feature work for capacity — requires disciplined product prioritisation
- Runway buffer is invisible to stakeholders — underfunding it is politically easy; consequences are lagged
- Too much runway in the wrong direction locks in premature decisions (violates Principle 3)

## Design Diagram

```mermaid
flowchart LR
    subgraph Now
        FT[Feature Teams\nSprints 1-2]
    end

    subgraph Runway — 2-3 Sprints ahead
        E1[Enabler: Kafka cluster\nSprint 3]
        E2[Enabler: API gateway\nauth middleware\nSprint 3]
        E3[Enabler: Secret\nmanagement for\nnew domains\nSprint 4]
    end

    subgraph IP / Platform Cadence
        IP[IP Iteration\nor Platform Sprint]
        ADR[Architecture Decisions\nADRs for runway]
        TRIAGE[Runway Triage\nWhat's needed in next PI?]
    end

    FT -- "blocked without" --> Runway
    IP --> E1 & E2 & E3
    ADR --> E1 & E2 & E3
    TRIAGE --> IP
```

## Code Sample

### Enabler story format

```markdown
## [ENABLER] Provision Kafka cluster for event-driven notification pipeline

**Type**: Architecture Enabler — Platform Runway
**Sprint**: Sprint 16 (3 sprints ahead of consumer — Notifications team needs in Sprint 19)
**Capacity**: 5 story points (Platform team)
**Drives**: ADR-038 (Kafka for notification fan-out)
**Enables features**: FEAT-204 (Email confirmation), FEAT-211 (SMS alerts), FEAT-219 (Push notifications)

**Definition of Done**:
- [ ] Confluent Cloud cluster provisioned in staging + production via Terraform
- [ ] `notifications.psp.requests` and `notifications.psp.results` topics created with correct retention
- [ ] Consumer group naming convention documented in platform wiki
- [ ] RBAC: each consuming service has a dedicated service account with least-privilege ACL
- [ ] Runbook: consumer lag alert + replay procedure documented
- [ ] Dead letter topic + alerting configured
- [ ] Platform team demo to Notifications team before Sprint 18 end
- [ ] Fitness function FF-020 (consumer lag < 30s) implemented in CI template

**Upstream dependency**: ADR-038 approved (✅ 2026-03-15)
**Consumer readiness**: Notifications team onboarding in Sprint 18 (1 sprint buffer)
```

### Runway buffer measurement

```typescript
// Architecture runway tracking model
export interface RunwayItem {
  id: string;
  title: string;
  type: 'platform' | 'architecture' | 'domain' | 'tooling';
  readySprint: number;         // sprint when enabler will be complete
  neededSprint: number;        // first sprint where a feature requires this
  buffer: number;              // neededSprint - readySprint (target: >= 2)
  status: 'not-started' | 'in-progress' | 'complete';
  consumingTeams: string[];
  drivesAdrs: string[];
}

export function detectRunwayGaps(items: RunwayItem[]): RunwayItem[] {
  return items.filter(item => {
    const buffer = item.neededSprint - item.readySprint;
    return buffer < 2 && item.status !== 'complete'; // < 2 sprints = at risk
  });
}

// Example usage
const runway: RunwayItem[] = [
  {
    id: 'RW-014',
    title: 'Kafka cluster provisioned',
    type: 'platform',
    readySprint: 16,
    neededSprint: 19,
    buffer: 3,           // safe
    status: 'in-progress',
    consumingTeams: ['notifications'],
    drivesAdrs: ['ADR-038'],
  },
  {
    id: 'RW-015',
    title: 'API Gateway auth middleware for new B2B product',
    type: 'architecture',
    readySprint: 18,
    neededSprint: 19,
    buffer: 1,           // AT RISK — escalate
    status: 'not-started',
    consumingTeams: ['b2b-portal'],
    drivesAdrs: ['ADR-044'],
  },
];

const gaps = detectRunwayGaps(runway);
console.log('Runway at risk:', gaps.map(g => g.title));
```

### Runway triage checklist (IP iteration)

```markdown
# Runway Triage — PI Planning / IP Sprint

## Step 1: Collect upcoming feature needs (from stream-aligned teams)
For each team: what architectural foundation will they need in the next 2–3 sprints that they don't currently have?

## Step 2: Check runway buffer
For each runway item:
- [ ] neededSprint - readySprint >= 2? (safe)
- [ ] neededSprint - readySprint < 2? (AT RISK — escalate or fast-track)
- [ ] readySprint > neededSprint? (BLOCKED — immediate action)

## Step 3: Prioritise enabler backlog
Stack rank runway items by: (urgency = gap) × (width = teams blocked)
Assign to platform team or architecture guild for next IP iteration.

## Step 4: Identify over-built runway
Any enabler ready > 5 sprints before it will be needed = potential premature investment.
Defer remaining work unless it will be needed sooner than expected.

## Step 5: Update ADRs and runway register
Ensure all runway items have an associated ADR (or that one is in progress).
```

## Key Patterns

### Runway in SAFe vs Continuous Architecture

In SAFe, runway is a quantitative measure (sprints of runway remaining). In continuous architecture, the concept is broader but the principle is the same: maintain a buffer of solved architectural problems ahead of feature delivery.

**Minimum viable runway practice** (non-SAFe teams):
1. Platform team maintains a "next needs" backlog, fed by stream-aligned teams in sprint planning/review
2. Platform lead reviews the backlog weekly against the feature roadmap
3. Escalate items with < 2-sprint buffer in the weekly architecture guild meeting
4. Enabler stories are visible in the programme board / roadmap alongside feature work

### Distinguishing Enablers from Features

| Story type | Who benefits | Value delivered |
|---|---|---|
| **Feature** | End user / business | Business outcome (revenue, user benefit) |
| **Enabler** | Engineering team | Architectural capability for future features |
| **Spike** | Architect / team | Knowledge to make a decision |
| **Tech debt paydown** | Engineering team | Reduced carrying cost; improved velocity |

Enablers are not non-functional requirements or defects. They are intentional investments in future delivery capacity.

## Related Patterns

- [05 — Architecture in Agile & DevOps](./05-architecture-agile.md) — Runway in the delivery cadence
- [09 — Team Topologies](./09-team-topologies.md) — Platform team builds and owns the runway
- [03 — Technical Debt](./03-technical-debt.md) — Depleted runway is often replaced by debt
- [06 — Fitness Functions](./06-fitness-functions.md) — FFs that verify runway items are correctly implemented
