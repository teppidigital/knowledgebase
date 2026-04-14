# Methods & Tools Playbook

## Category

Continuous Architecture — Practice

## Context

This playbook distils the concrete **methods, tools, and templates** from the core continuous architecture literature into step-by-step instructions that can be applied directly to a real project. Each entry follows the same structure: what the method is, when to use it, how to run it, and a reusable template.

**Source books:**
- *Continuous Architecture in Practice* — Erder, Pureur, Woods (2021)
- *Building Evolutionary Architectures* — Ford, Parsons, Kua (2nd ed. 2023)
- *Software Architecture in Practice* — Bass, Clements, Kazman (4th ed. 2022)
- *Team Topologies* — Skelton, Pais (2019)
- *Accelerate* — Forsgren, Humble, Kim (2018)

---

## Method Index

| # | Method / Tool | Category | Time Investment |
|---|--------------|----------|----------------|
| M1 | [Quality Attribute Workshop (QAW)](#m1-quality-attribute-workshop-qaw) | Discovery | 4–8 h (half-day to full day) |
| M2 | [Utility Tree](#m2-utility-tree) | Discovery | 2–4 h |
| M3 | [QA Scenario Template](#m3-qa-scenario-template) | Specification | 30 min per scenario |
| M4 | [Lightweight ATAM](#m4-lightweight-atam) | Evaluation | 4–8 h |
| M5 | [Risk Storming](#m5-risk-storming) | Evaluation | 2–4 h |
| M6 | [Architecture Decision Record (MADR)](#m6-architecture-decision-record-madr) | Documentation | 30–60 min per decision |
| M7 | [Technical Debt Register](#m7-technical-debt-register) | Sustainability | Ongoing |
| M8 | [Fitness Function Definition](#m8-fitness-function-definition) | Governance | 1–2 h per function |
| M9 | [Architecture Runway Planning](#m9-architecture-runway-planning) | Process | Per PI/quarter |
| M10 | [Technology Radar](#m10-technology-radar) | Governance | 2–4 h per cycle |
| M11 | [C4 Model + Structurizr](#m11-c4-model--structurizr) | Documentation | 2–4 h initial; 30 min per update |
| M12 | [Team Cognitive Load Assessment](#m12-team-cognitive-load-assessment) | Organisation | 1–2 h |
| M13 | [Architecture Health Dashboard](#m13-architecture-health-dashboard) | Measurement | 1–2 days setup; weekly review |

---

## M1. Quality Attribute Workshop (QAW)

### What it is
A facilitated session to elicit, prioritise, and document the quality attributes that will drive architectural decisions. Developed by the SEI (Software Engineering Institute) and central to *Continuous Architecture in Practice* (Chapter 3).

### When to use it
- Start of a new product or major version
- Before significant architectural restructuring
- When technical debt is impacting delivery and you need to reprioritise QAs
- Onboarding a new architect to an existing system

### Participants
| Role | Responsibility |
|------|---------------|
| Facilitator (architect) | Guides discussion, prevents scope creep, captures QAs |
| Business stakeholder | Provides business context and priorities |
| Product manager | Prioritises QA importance vs cost to implement |
| Engineering leads | Assess feasibility and implementation difficulty |
| Operations / SRE | Represents operational QAs (availability, observability) |

### Steps

**Step 1 — Present the system (15 min)**
Architect presents a 1-page system overview: purpose, major components, deployment context, users. Use a C4 System Context diagram.

**Step 2 — Business driver elicitation (30 min)**
Each stakeholder presents their top 3 business drivers. Facilitator maps each driver to QAs:
> "We need to onboard 10× more customers next year" → Scalability, Performance
> "We are subject to PSD2 audit" → Security, Auditability, Modifiability

**Step 3 — QA brainstorm (30 min)**
Participants call out all QAs they care about. Record on sticky notes / Miro board. Do not prioritise yet. Use the ISO 25010 taxonomy as a prompt (performance, reliability, security, maintainability, usability, portability, compatibility, operability).

**Step 4 — Scenario generation (60 min)**
For each top QA, generate concrete scenarios using the stimulus-response structure (see M3). At least 1 scenario per QA; aim for 2–3 for each High-priority QA.

**Step 5 — Prioritisation (30 min)**
Vote using dot voting or the priority matrix:
- Axis 1: Business importance (H/M/L)
- Axis 2: Implementation difficulty (H/M/L)
High/High QAs drive the architecture. Low/Low QAs can be addressed at sprint level.

**Step 6 — Document as Utility Tree (see M2)**

### Output
- Prioritised QA list
- 10–20 QA scenarios
- Utility tree
- Inputs for architecture trade-off analysis

### Facilitation tips
- Never let a QA stay as a vague adjective ("the system must be fast") — always push to a scenario with a measurable response
- "Available" and "Reliable" mean different things — distinguish uptime from correctness from data consistency
- Stakeholders routinely ask for every QA at H/H — insist on realistic prioritisation with explicit trade-offs

---

## M2. Utility Tree

### What it is
A hierarchical decomposition of quality attributes into sub-attributes and concrete prioritised scenarios. From Bass, Clements, Kazman — widely adopted in *Continuous Architecture in Practice*.

### Template

```
Utility (top)
├── Performance
│   ├── Latency
│   │   ├── [H, H] QA-001: Search results within 200 ms (p95) under 500 concurrent users
│   │   └── [H, M] QA-002: Loan application submission response within 3 s
│   └── Throughput
│       └── [M, M] QA-003: System processes 10,000 transactions/hour without degradation
├── Availability
│   ├── Uptime
│   │   └── [H, H] QA-004: Core APIs available 99.9% in any rolling 30-day window
│   └── Recoverability
│       └── [H, M] QA-005: Automatic failover within 30 s of primary DB failure
├── Security
│   ├── Authentication
│   │   └── [H, H] QA-006: All API requests authenticated; unauthenticated rejected 401 in < 50 ms
│   └── Auditability
│       └── [H, M] QA-007: All state-changing operations logged with user, timestamp, delta
├── Modifiability
│   ├── Evolvability
│   │   └── [M, H] QA-008: New payment provider integrated in ≤ 5 dev days
│   └── Deployability
│       └── [H, M] QA-009: Any service deployed independently without downtime
└── Observability
    └── [M, M] QA-010: Any production error diagnosed to root cause within 15 min
```

**Priority notation:** `[Business importance, Technical difficulty]` — each rated H/M/L.

### Markdown template (copy-paste ready)

```markdown
## Utility Tree — [System Name] — [Date]

### Performance
- **Latency**
  - `[H, H]` QA-001: _Source / Stimulus / Environment / Response / Measure_
- **Throughput**
  - `[M, M]` QA-002: …

### Availability
- **Uptime**
  - `[H, H]` QA-003: …
- **Recoverability**
  - `[H, M]` QA-004: …

### Security
- **Confidentiality**
  - `[H, H]` QA-005: …
- **Auditability**
  - `[H, M]` QA-006: …

### Modifiability
- **Evolvability**
  - `[M, H]` QA-007: …
- **Deployability**
  - `[H, M]` QA-008: …

### Observability
- `[M, M]` QA-009: …
```

---

## M3. QA Scenario Template

### What it is
A structured six-part specification of a quality attribute requirement. From the SEI attribute-based architecture design (AADM/ATAM). Each scenario is concrete, measurable, and testable as a fitness function.

### Six-Part Structure

| Part | Description | Example |
|------|-------------|---------|
| **Source** | Who or what generates the stimulus | External user, internal batch job, attacker |
| **Stimulus** | The event that triggers the QA concern | 500 concurrent users submit loan applications |
| **Environment** | The system's operating state when it happens | Normal operation; peak business hours |
| **Artefact** | Which part of the system is affected | Loan application service |
| **Response** | How the system responds | Requests queued and processed; no errors returned |
| **Measure** | Quantified proof that the response was adequate | p95 latency ≤ 3 s; error rate < 0.1%; queue depth < 1000 |

### Full Template

```markdown
## QA Scenario: [Short Title]

| Field       | Value |
|-------------|-------|
| **ID**      | QA-XXX |
| **Quality Attribute** | Performance / Availability / Security / Modifiability / … |
| **Priority** | H/M/L (Business) × H/M/L (Technical) |
| **Source**  | [Who triggers the stimulus] |
| **Stimulus**| [The event] |
| **Environment** | [System state when event occurs] |
| **Artefact**| [Part of system affected] |
| **Response**| [What the system does] |
| **Measure** | [Quantified threshold for pass/fail] |
| **Fitness Function** | [How this will be verified automatically] |
| **Status**  | Draft / Agreed / Implemented / Verified |
| **Last Verified** | YYYY-MM-DD |
```

### Completed Example

```markdown
## QA Scenario: Core API Availability Under DB Failure

| Field       | Value |
|-------------|-------|
| **ID**      | QA-004 |
| **Quality Attribute** | Availability — Recoverability |
| **Priority** | H (Business) × M (Technical) |
| **Source**  | Infrastructure fault (AWS AZ failure) |
| **Stimulus**| Primary RDS instance becomes unavailable |
| **Environment** | Normal production traffic, business hours |
| **Artefact**| Loan management API |
| **Response**| System promotes read replica to primary; requests briefly retry; no data loss |
| **Measure** | Failover completes within 30 s; zero 5xx errors after 35 s; RPO = 0 transactions |
| **Fitness Function** | Monthly chaos engineering test: terminate primary RDS; assert failover within SLA |
| **Status**  | Agreed |
| **Last Verified** | 2026-03-15 |
```

---

## M4. Lightweight ATAM

### What it is
A condensed version of the SEI Architecture Trade-off Analysis Method. The full ATAM is a 2-day workshop for complex safety-critical systems. The lightweight version fits in a half-day and is the version recommended in *Continuous Architecture in Practice* for iterative use.

### When to use it
- Evaluating a proposed architecture for a new system or major capability
- Before committing to a significant technology choice
- Annually as an architecture health review of a live system

### Steps (half-day format, 4 h)

**Step 1 — Architecture presentation (45 min)**
The architect presents the architecture using C4 diagrams (context, container, component level). Do not skip component level — architecture risks hide in the component interactions.

**Step 2 — QA scenarios (30 min)**
Present the top 10 prioritised scenarios from the Utility Tree. Participants add any missing scenarios (5 min sticky note exercise).

**Step 3 — Architecture approaches (30 min)**
For each top-ranked QA scenario, architect explains which structural decision satisfies it: "QA-004 (availability) is addressed by the active-passive database cluster + health check + retry in the API gateway."

**Step 4 — Sensitivity and trade-off analysis (60 min)**
**Sensitivity point:** A change in one component changes a QA measure significantly.
**Trade-off point:** A decision affects two or more QAs in opposite directions.

Capture findings in this table:

```markdown
| Decision | QA Helped | QA Hurt | Sensitivity? | Trade-off? |
|----------|-----------|---------|-------------|-----------|
| Sync REST between services | Simplicity ↑ | Resilience ↓ | Yes | Yes |
| Event-driven via Kafka | Resilience ↑ | Observability ↓ | No | Yes |
| Single shared DB | Deployability ↓ | Consistency ↑ | Yes | Yes |
| Read replicas | Performance ↑ | Consistency ↓ (stale reads) | No | Yes |
```

**Step 5 — Risk identification (30 min)**
Participants use Risk Storming (see M5) to identify architectural risks from the sensitivity/trade-off findings.

**Step 6 — Results summary (15 min)**

Output document:
- Updated utility tree with any new scenarios
- Sensitivity/trade-off analysis table
- Risk register additions (see M7)
- ADRs for any decisions reached (see M6)

---

## M5. Risk Storming

### What it is
A collaborative visual technique to identify architectural risk, from *Building Evolutionary Architectures* (Ford, Parsons, Kua, Chapter 7). Participants independently annotate an architecture diagram with risks, then discuss disagreements.

### When to use it
- Before committing to a new architecture
- When a new team member reviews an existing architecture
- During a post-incident review to find structural causes
- Quarterly architecture health review

### Steps (2 h)

**Preparation (30 min before session)**
Print or display a C4 Container + Component diagram on a shared board (Miro, FigJam, or physical whiteboard). Each component/service should be individually identifiable.

**Step 1 — Individual risk identification (20 min)**
Each participant independently places coloured risk indicators on components/connections. No discussion yet.

| Colour | Risk level | Meaning |
|--------|-----------|---------|
| 🟢 Green | Low | Comfortable; no significant risk |
| 🟡 Yellow | Medium | Some concern; monitor |
| 🔴 Red | High | Significant risk; must address |

**Step 2 — Consensus areas (10 min)**
Areas where everyone placed the same colour require no discussion. Note them quickly.

**Step 3 — Disagreement discussion (60 min)**
Focus all discussion on disagreements. For each disagreement:
- Each participant states their risk reasoning
- Group agrees on final risk rating
- Facilitator captures the risk description

**Step 4 — Risk mitigation brainstorm (30 min)**
For each Red risk, brainstorm mitigation options. Map to one of:
- **Mitigate** — architectural change that reduces the risk
- **Accept** — acknowledge and monitor with a fitness function
- **Transfer** — insurance, vendor SLA, third-party responsibility
- **Avoid** — remove the risky component or capability

### Risk Register Template

```markdown
## Risk Register Entry

| Field         | Value |
|---------------|-------|
| **Risk ID**   | RISK-XXX |
| **Component** | [Component or connection where risk was identified] |
| **Description** | [What could go wrong] |
| **QA Affected** | [Which QA scenario is at risk] |
| **Likelihood** | H / M / L |
| **Impact**    | H / M / L |
| **Risk Score**| (Likelihood × Impact) |
| **Strategy**  | Mitigate / Accept / Transfer / Avoid |
| **Mitigation Action** | [Specific action + owner + due date] |
| **Fitness Function** | [How this risk is continuously monitored] |
| **Status**    | Open / In Progress / Resolved |
| **Identified** | YYYY-MM-DD |
| **Review Date** | YYYY-MM-DD |
```

---

## M6. Architecture Decision Record (MADR)

### What it is
Markdown Architectural Decision Records (MADR) is the most widely adopted ADR format. Structured, concise, and version-controllable. Referenced throughout *Continuous Architecture in Practice*.

### When to write one
- Technology selection (language, framework, database, messaging system)
- Structural pattern decision (monolith vs microservices, sync vs async, shared vs separate DB)
- Cross-team API contract decision
- Security control decision
- Deployment strategy decision

### MADR Template (full)

```markdown
# [ADR-NNN] [Short decision title — imperative verb]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNN
**Deciders:** [Names or roles of those making the decision]
**Consulted:** [Names or roles of those whose input was sought]
**Informed:** [Names or roles who are notified of the decision]

---

## Context and Problem Statement

[2–4 sentences: What is the architectural force or problem this decision addresses?
What makes this decision significant and hard to reverse?]

## Decision Drivers

- [QA scenario or business requirement driving this decision, e.g. QA-004 availability]
- [Technical constraint, e.g. team has no Go expertise]
- [Strategic driver, e.g. vendor lock-in risk]

## Considered Options

1. [Option A name]
2. [Option B name]
3. [Option C name]

## Decision Outcome

**Chosen option:** [Option X]

**Rationale:** [1–3 sentences: Why this option over the others? Which decision drivers does it best satisfy?]

**Consequences:**
- ✅ [Positive: what becomes easier or better]
- ✅ [Positive: …]
- ⚠️ [Negative / trade-off: what becomes harder or is compromised]
- ⚠️ [Negative: …]

---

## Options Analysis

### Option A: [Name]

**Description:** [1–3 sentences]

| Quality Attribute | Impact | Notes |
|------------------|--------|-------|
| Performance | Positive / Neutral / Negative | |
| Availability | … | |
| Modifiability | … | |
| Security | … | |
| Deployability | … | |

**Pros:**
- …

**Cons:**
- …

### Option B: [Name]

[Same structure as Option A]

### Option C: [Name]

[Same structure as A]

---

## Links

- [Supersedes ADR-NNN] (if applicable)
- [Related to QA-XXX]
- [RFC / Spike ticket link]
- [References: article, book chapter, RFC number]
```

### Minimal MADR (for simple decisions)

```markdown
# [ADR-NNN] [Title]

**Date:** YYYY-MM-DD | **Status:** Accepted
**Deciders:** [Names]

## Context
[What forced this decision?]

## Decision
[What was decided?]

## Consequences
- ✅ [Positive]
- ⚠️ [Trade-off]
```

### ADR Workflow

```
1. Engineer identifies a significant decision
2. Draft ADR written (Proposed status)
3. PR opened — reviewed by architect + affected team leads
4. Discussion in PR comments (max 5 business days)
5. ADR merged as Accepted (or rejected with reason documented)
6. ADR number added to related tickets and architecture diagrams
```

### File naming convention
```
docs/decisions/
  ADR-0001-use-postgresql-as-primary-datastore.md
  ADR-0002-event-driven-integration-via-kafka.md
  ADR-0003-adopt-hexagonal-architecture.md
```

---

## M7. Technical Debt Register

### What it is
A managed backlog of known architectural shortcuts, aging dependencies, and structural problems that impair the system's quality attributes. From *Continuous Architecture in Practice* (Chapter 5).

### Debt Taxonomy

| Type | Description | Example |
|------|-------------|---------|
| **Code debt** | Poorly structured code within a component | God class, duplicate logic |
| **Design debt** | Wrong architectural pattern for current needs | Synchronous calls where async needed |
| **Test debt** | Missing or low-quality test coverage | No integration tests for payment flow |
| **Documentation debt** | Outdated or missing architecture docs | No ADRs for key decisions |
| **Dependency debt** | Outdated or vulnerable libraries | Spring Boot 2.x (EOL), CVE-bearing transitive dep |
| **Infrastructure debt** | Manual processes, fragile infrastructure | Manual deployments, no IaC |
| **Knowledge debt** | Only 1 person understands a critical component | Bus factor = 1 on payments service |

### Debt Register Template

```markdown
## Technical Debt Register — [System Name]

Last updated: YYYY-MM-DD | Owner: [Architect name]

| ID | Type | Component | Description | QA Impact | Effort (T-shirt) | Risk if Unresolved | Priority | Status | Target Sprint |
|----|------|-----------|-------------|-----------|------------------|--------------------|----------|--------|---------------|
| TD-001 | Design | Payment Service | Sync HTTP to Fraud Service — blocking call causes P99 latency spike on fraud check delays | Performance QA-001 | M | High — latency SLO breach | H | In Backlog | Q3 2026 |
| TD-002 | Dependency | All services | Spring Boot 2.7 (EOL Jan 2024) — no more security patches | Security QA-006 | L | High — CVE exposure | H | In Progress | Sprint 42 |
| TD-003 | Test | Loan Service | No integration tests for refinancing flow — 3 regressions in last 6 months | Modifiability QA-008 | M | Medium — regression rate | M | In Backlog | Q4 2026 |
| TD-004 | Knowledge | Notification Service | Single engineer understands the Twilio integration — all changes go through them | Deployability QA-009 | S | High if that engineer leaves | M | In Backlog | Q3 2026 |
```

### Debt Scoring Heuristic

```
Risk if Unresolved:
  High   = likely to cause production incident or SLO breach within 6 months
  Medium = will slow delivery by 20%+ within 12 months
  Low    = nuisance; unlikely to become critical

Priority = f(Risk, Effort):
  High Risk + Any Effort = High Priority
  Medium Risk + Small Effort = High Priority (quick win)
  Medium Risk + Large Effort = Medium Priority
  Low Risk = Low Priority unless it blocks other work
```

### Debt Paydown Planning
- Reserve **20% of engineering capacity per sprint** for debt paydown (the "20% tech health rule", *Continuous Architecture in Practice*, p.142)
- Surface debt items as regular backlog stories — no separate "tech debt sprints"
- Every new debt item must have a named owner and a target quarter

---

## M8. Fitness Function Definition

### What it is
A template for specifying, implementing, and maintaining an architecture fitness function. From *Building Evolutionary Architectures* (Ford, Parsons, Kua).

### Fitness Function Specification Template

```markdown
## Fitness Function: [Short Name]

| Field | Value |
|-------|-------|
| **ID** | FF-XXX |
| **QA Scenario** | QA-XXX — [Scenario title] |
| **Type** | Atomic / Holistic |
| **Trigger** | Triggered (CI pipeline) / Continual (production monitoring) |
| **Nature** | Automated / Manual |
| **Temporality** | Static (code analysis) / Dynamic (runtime test) / Temporal (scheduled) |
| **Tool** | ArchUnit / dependency-cruiser / k6 / OPA / custom script |
| **Pass Condition** | [Exact measurable threshold] |
| **Fail Action** | Block PR merge / Block deployment / Alert only |
| **Owner** | [Team or person responsible] |
| **Implementation Status** | Not started / In progress / Active |
| **Last Run** | YYYY-MM-DD |
| **Pass Rate (last 30 days)** | XX% |
```

### Fitness Function Examples by Category

**Structural (ArchUnit — Java)**
```java
// FF-001: No service imports directly from another service's package
@Test
void services_must_not_depend_on_each_other_directly() {
    JavaClasses classes = new ClassFileImporter().importPackages("com.example");
    ArchRule rule = noClasses()
        .that().resideInAPackage("..loans..")
        .should().dependOnClassesThat()
        .resideInAPackage("..accounts..");
    rule.check(classes);
}

// FF-002: All public API controllers must use the @Validated annotation
@Test
void controllers_must_validate_input() {
    JavaClasses classes = new ClassFileImporter().importPackages("com.example");
    ArchRule rule = classes()
        .that().haveSimpleNameEndingWith("Controller")
        .should().beAnnotatedWith(Validated.class);
    rule.check(classes);
}
```

**Structural (dependency-cruiser — TypeScript / Node.js)**
```json
{
  "forbidden": [
    {
      "name": "no-cross-domain-imports",
      "comment": "FF-003: Loan domain must not import from account domain",
      "from": { "path": "src/loans/" },
      "to":   { "path": "src/accounts/" },
      "severity": "error"
    },
    {
      "name": "no-circular-dependencies",
      "comment": "FF-004: No circular module dependencies",
      "from": {},
      "to": { "circular": true },
      "severity": "error"
    }
  ]
}
```

**Performance (k6 — load test gate)**
```javascript
// FF-005: Loan search p95 latency ≤ 200 ms under 100 concurrent users
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 100,
  duration: '60s',
  thresholds: {
    'http_req_duration': ['p(95)<200'],   // p95 ≤ 200 ms
    'http_req_failed': ['rate<0.001'],    // error rate < 0.1%
  },
};

export default function () {
  const res = http.get('http://api.example.com/loans?status=active');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

**Temporal (scheduled — dependency freshness)**
```yaml
# FF-006: No dependency older than 18 months with a known CVE
# Run weekly in CI as a scheduled job
name: Dependency CVE Check
on:
  schedule:
    - cron: '0 8 * * 1'  # every Monday 08:00
jobs:
  cve-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - run: npx @cyclonedx/cyclonedx-npm --output sbom.json && npx @appthreat/cdxgen audit sbom.json
```

---

## M9. Architecture Runway Planning

### What it is
The process of identifying, writing, and sequencing **architecture enabler stories** — the work that keeps the architecture ahead of the feature delivery teams. From *Continuous Architecture in Practice* (Chapter 8) and SAFe.

### Runway Concepts

| Term | Definition |
|------|-----------|
| **Architecture runway** | The amount of existing architecture that supports feature implementation without immediate architectural rework |
| **Enabler story** | A user story for architectural work: infrastructure, exploration, technical preparation |
| **Runway length** | Typically 1–2 PI (programme increments) = 2–4 sprints ahead of feature teams |
| **Spike** | Time-boxed exploration of an unknown (prototype, proof of concept, benchmark) |

### Enabler Story Types

| Type | When to use | Example |
|------|------------|---------|
| **Infrastructure enabler** | Build the foundation needed for a future feature | "Set up Kafka cluster with schema registry for event-driven integration" |
| **Architecture exploration** | Evaluate technology or approach before committing | "Spike: evaluate GraphQL Federation vs BFF for mobile API — 3 days, decision document output" |
| **Technical debt paydown** | Eliminate existing structural shortcoming | "Migrate payment-service from synchronous to async Kafka-based credit check" |
| **Fitness function implementation** | Automate an architectural governance check | "Implement ArchUnit rule to enforce no cross-domain imports" |

### Runway Planning Template (per PI/Quarter)

```markdown
## Architecture Runway Plan — [Quarter / PI Name]

**Runway Target:** Features planned for Q[N+1] require: [list capabilities]

### Enabler Stories

| ID | Type | Title | Linked Feature | Sprint Target | Owner | Done Criteria |
|----|------|-------|---------------|---------------|-------|---------------|
| EN-001 | Infrastructure | Deploy Kafka cluster + Schema Registry to staging | Event-driven notifications (Q3) | Sprint 41 | Platform team | Cluster running; schema registry accessible; producer/consumer smoke test green |
| EN-002 | Exploration | Spike: GraphQL Federation vs REST aggregation gateway | Mobile app v2 (Q3) | Sprint 41 | API team | Decision document + prototype; ADR drafted |
| EN-003 | Debt | Replace sync credit check with async Queueable pattern | Loan application SLA (Q3) | Sprint 42 | Loans team | All credit checks async; P95 latency QA-001 green |
| EN-004 | FF | ArchUnit rule: no cross-service DB access | Platform stability (ongoing) | Sprint 42 | Architecture guild | Rule in CI; 0 violations in main |

### Runway Health Check

After each sprint, assess:
- Is the runway still 1–2 iterations ahead? ✅ / ⚠️ / ❌
- Are any features at risk of landing on architectural debt? List here
- Are spikes generating ADRs and decisions? List unresolved spikes
```

---

## M10. Technology Radar

### What it is
A visual tool for tracking the status of technologies, frameworks, tools, and platforms across four rings. Originated by ThoughtWorks; adopted in *Continuous Architecture in Practice* as the standard governance tool for technology direction.

### Quadrants & Rings

| Quadrant | What it covers |
|---------|---------------|
| **Techniques** | Process, architecture patterns, practices (CQRS, DDD, GitOps) |
| **Platforms** | Things you build software on (Kubernetes, Kafka, AWS, Salesforce) |
| **Tools** | Supporting software (SonarQube, Backstage, Datadog) |
| **Languages & Frameworks** | Programming languages, major frameworks (TypeScript, Spring Boot) |

| Ring | Meaning | Action |
|------|---------|--------|
| **Adopt** | Proven, recommended for use | Default choice for new work |
| **Trial** | Worth pursuing in a project; watch carefully | Pilot on a non-critical project first |
| **Assess** | Worth exploring; needs further investigation | Research and prototype — not yet production |
| **Hold** | Proceed with caution; not recommended for new work | Existing use acceptable; no new adoption |

### Running a Technology Radar Session (2–4 h, quarterly)

**Preparation (1 week before)**
Collect nominations from all engineers via a simple form:
- "Technology / Tool / Pattern name"
- "Proposed ring (Adopt/Trial/Assess/Hold)"
- "Reason in 2 sentences"

**Session Steps:**
1. **Group nominations by quadrant** (15 min)
2. **For each nomination, short discussion** — is the ring right? Any disagreements? (5 min per item; skip uncontested ones)
3. **Identify movements** — what has moved from Trial → Adopt? What should move to Hold? (20 min)
4. **Finalize and publish** — update the radar (use Thoughtworks Build Your Own Radar or Backstage TechRadar plugin)

### Radar Entry Template

```markdown
## Radar Entry: [Technology Name]

| Field | Value |
|-------|-------|
| **Quadrant** | Techniques / Platforms / Tools / Languages & Frameworks |
| **Ring** | Adopt / Trial / Assess / Hold |
| **Nominated by** | [Name / Team] |
| **Date** | YYYY-MM-DD |
| **Movement** | New entry / Moved from [ring] |

### Summary
[2–3 sentences: What is it? Why does it belong in this ring?]

### Experience at [Company]
[What has been tried internally? What was the outcome?]

### Recommendation
[What should teams do with this entry?]
```

### Governance Rules
- Radar is reviewed and published **quarterly**
- A technology cannot move from Assess to Adopt without a real production use case at the company
- **Hold** does not mean "remove immediately" — existing systems can continue; no new projects should adopt
- The radar is owned by the **Architecture Guild** (representatives from all stream-aligned teams + platform team)

---

## M11. C4 Model + Structurizr

### What it is
A hierarchical system of four diagram levels (Context, Container, Component, Code) that provides a consistent, learnable notation for architecture documentation. Created by Simon Brown; extensively referenced in continuous architecture practice.

### Four Levels

| Level | Audience | Shows | When to update |
|-------|---------|-------|---------------|
| **L1: System Context** | Business, leadership, new joiners | Your system + external systems + actors | When system boundaries change |
| **L2: Container** | Architects, engineers | Major deployable units (services, DBs, queues) | When services are added/removed |
| **L3: Component** | Engineers | Components inside a container | When internal structure changes significantly |
| **L4: Code** | Engineers | Classes, modules (auto-generated from code) | Rarely — use code browser instead |

### Structurizr DSL Template (L1 + L2)

```javascript
workspace "Lending Platform" {

    model {
        // Actors
        borrower    = person "Borrower" "A customer applying for or managing a loan"
        loanOfficer = person "Loan Officer" "Internal staff approving loans"
        ops         = person "Operations Team" "Monitors system health"

        // External systems
        creditBureau = softwareSystem "Credit Bureau" "Provides credit score for applicants" {
            tags "External"
        }
        emailProvider = softwareSystem "Email Service" "Sends notifications (SendGrid)" {
            tags "External"
        }
        idProvider = softwareSystem "Identity Provider" "Auth0 — authentication and token issuance" {
            tags "External"
        }

        // Your system
        lendingPlatform = softwareSystem "Lending Platform" "Core loan origination and management system" {

            // L2: Containers
            webApp = container "Web Application" "React SPA for borrowers and loan officers" "TypeScript / React"
            apiGateway = container "API Gateway" "Routes, authenticates, rate-limits all API requests" "Kong"
            loanService = container "Loan Service" "Loan origination, lifecycle management" "Node.js / Express" {
                // L3: Components
                loanController = component "Loan Controller" "REST endpoints for loan operations" "Express Router"
                loanRepository = component "Loan Repository" "DB access layer" "Prisma / TypeScript"
                loanDomainService = component "Loan Domain Service" "Business rules, state machine" "TypeScript"
                creditCheckAdapter = component "Credit Check Adapter" "Calls external credit bureau API" "TypeScript"
            }
            notificationService = container "Notification Service" "Sends emails and in-app notifications" "Node.js"
            loanDb = container "Loan Database" "Primary data store for loan records" "PostgreSQL" {
                tags "Database"
            }
            eventBus = container "Event Bus" "Async messaging between services" "Apache Kafka" {
                tags "Message Broker"
            }

            // Relationships — L2
            webApp          -> apiGateway "HTTPS API calls" "REST / JSON"
            apiGateway      -> loanService "Routes loan requests" "REST / JSON"
            loanService     -> loanDb "Reads and writes loan data" "TCP / Prisma ORM"
            loanService     -> eventBus "Publishes loan lifecycle events" "Kafka producer"
            notificationService -> eventBus "Consumes loan events" "Kafka consumer"
            notificationService -> emailProvider "Sends email notifications" "HTTPS / SendGrid API"
            loanService     -> creditBureau "Fetches credit score" "HTTPS / REST"
            apiGateway      -> idProvider "Validates JWT tokens" "HTTPS / JWKS"

            // Relationships — L3 (within loanService)
            loanController  -> loanDomainService "Calls domain logic"
            loanDomainService -> loanRepository "Reads / writes via repository"
            loanDomainService -> creditCheckAdapter "Triggers credit check"
            creditCheckAdapter -> creditBureau "External callout"
        }

        // Actor → system relationships
        borrower    -> webApp "Applies for loans, checks status" "HTTPS"
        loanOfficer -> webApp "Reviews and approves applications" "HTTPS"
        ops         -> apiGateway "Monitors via Datadog dashboards" "HTTPS"
    }

    views {
        systemContext lendingPlatform "SystemContext" {
            include *
            autoLayout
        }
        container lendingPlatform "Containers" {
            include *
            autoLayout
        }
        component loanService "LoanServiceComponents" {
            include *
            autoLayout
        }

        styles {
            element "Person"         { shape Person; background #1168bd; color #ffffff }
            element "Software System" { background #1168bd; color #ffffff }
            element "External"       { background #999999; color #ffffff }
            element "Container"      { background #438dd5; color #ffffff }
            element "Database"       { shape Cylinder }
            element "Message Broker" { shape Pipe }
            element "Component"      { background #85bbf0; color #000000 }
        }
    }
}
```

### C4 Documentation Standard
- L1 and L2 diagrams are **mandatory** for every service in production
- L3 is written for services where the internal structure is a common source of confusion
- Diagrams live in the same git repo as the service code — reviewed alongside code changes
- Use **Structurizr Lite** (free, self-hosted) for local rendering; push to Structurizr Cloud or embed in Confluence

---

## M12. Team Cognitive Load Assessment

### What it is
A structured exercise to assess whether a team's scope (services, domains, responsibilities) exceeds its sustainable cognitive load. From *Team Topologies* (Skelton, Pais, Chapter 4).

### Cognitive Load Types

| Type | Description | Sign it's exceeded |
|------|-------------|-------------------|
| **Intrinsic** | Inherent complexity of the domain being built | "Nobody fully understands the full domain" |
| **Extraneous** | Complexity caused by tools, process, environment | "Most of our time is spent fighting the platform" |
| **Germane** | Useful cognitive effort that builds expertise | "We are learning and improving — this is healthy" |

### Assessment Questionnaire (per team)

Rate each statement 1 (strongly agree) to 5 (strongly disagree):

**Domain ownership**
1. Every team member understands the full scope of our service boundaries
2. We can comfortably explain what our services do to a new joiner in < 30 minutes
3. We have clear ownership of each component — no "shared ownership" grey areas

**Change complexity**
4. We can make a change to any part of our codebase without needing another team's help in > 80% of cases
5. Understanding a new ticket takes < 2 hours on average before we can start coding
6. We rarely need to read code in other teams' repositories to complete our work

**Operations**
7. We are fully on-call responsible for all services in our scope
8. Our on-call burden is sustainable — fewer than 2 pages per week on average
9. We can diagnose most production issues in < 30 minutes

**Tooling and platform**
10. Our build and deploy tooling is largely self-service — we don't wait for platform team approval
11. We spend < 20% of our time on infrastructure / environment management

**Scoring:**
- 44–55: Low cognitive load — team may be able to absorb more scope
- 33–43: Acceptable cognitive load — monitor
- 22–32: Elevated cognitive load — consider splitting scope or adding platform support
- <22: Critical cognitive load — team is overwhelmed; immediate action required

### Action Templates

**If extraneous load is high (tooling / environment):**
→ Request a Platform Engineering team to build a self-service golden path
→ Create an enabling team engagement (Platform team embeds for 1–2 sprints)

**If intrinsic load is high (domain complexity):**
→ Split the team along domain seam (DDD bounded context boundary)
→ Create an enabling team to transfer capability on the complex subdomain

**If germane load is high (good learning):**
→ Document what is being learned as ADRs and team wikis to reduce future intrinsic load

---

## M13. Architecture Health Dashboard

### What it is
A consolidated view of the architecture's health across structural, delivery, operational, and governance dimensions. Referenced throughout *Continuous Architecture in Practice* and *Accelerate*.

### Dashboard Metrics Schema

```markdown
## Architecture Health Dashboard — [System Name]

**Period:** [Week / Month / Quarter]  **Updated:** YYYY-MM-DD  **Owner:** [Architect]

---

### 🚀 Delivery (DORA)

| Metric | This Period | Trend | Elite Threshold |
|--------|------------|-------|-----------------|
| Deployment Frequency | X deploys/day | ↑ / ↓ / → | Multiple per day |
| Lead Time for Changes | X hours/days | ↑ / ↓ / → | < 1 hour |
| Change Failure Rate | X% | ↑ / ↓ / → | < 5% |
| MTTR (Mean Time to Restore) | X minutes | ↑ / ↓ / → | < 1 hour |

---

### 🏗️ Structural

| Metric | This Period | Trend | Threshold |
|--------|------------|-------|-----------|
| Avg Instability (I = Ce / (Ca + Ce)) | 0.XX | ↑ / ↓ / → | < 0.5 |
| Avg Abstractness (A) | 0.XX | ↑ / ↓ / → | Topic-specific |
| Distance from Main Sequence (D) | 0.XX | ↑ / ↓ / → | < 0.2 |
| Cyclomatic complexity (avg per module) | XX | ↑ / ↓ / → | < 10 |
| Test coverage (unit) | XX% | ↑ / ↓ / → | > 80% |
| Duplicate code blocks | XX | ↑ / ↓ / → | 0 High severity |

---

### 📋 Technical Debt

| Metric | This Period | Trend |
|--------|------------|-------|
| Open debt items | XX | ↑ / ↓ / → |
| High-priority debt items | XX | ↑ / ↓ / → |
| Debt items resolved this period | XX | |
| Debt items added this period | XX | |
| Oldest unresolved debt item age | XX days | |

---

### ✅ Fitness Functions

| Function | Pass Rate | Status | Trend |
|----------|-----------|--------|-------|
| FF-001: No cross-domain imports | 100% | ✅ Green | → |
| FF-002: P95 latency ≤ 200 ms | 98.2% | ⚠️ Amber | ↓ |
| FF-003: No CVE-bearing dependencies | 100% | ✅ Green | → |
| FF-004: Test coverage ≥ 80% | 87% | ✅ Green | ↑ |

---

### ⚙️ Operational

| Metric | This Period | SLO Target | Status |
|--------|------------|------------|--------|
| API availability | 99.97% | 99.9% | ✅ |
| P95 latency (loan submission) | 185 ms | ≤ 200 ms | ✅ |
| P99 latency (loan submission) | 420 ms | ≤ 500 ms | ✅ |
| Error rate | 0.03% | < 0.1% | ✅ |
| SLO burn rate (30d) | 0.12× | < 1× | ✅ |

---

### 📌 Actions This Period

| Action | Owner | Due | Status |
|--------|-------|-----|--------|
| Investigate FF-002 amber trend (P95 regression) | Team Lead, Loans | YYYY-MM-DD | In Progress |
| Close TD-002: Spring Boot upgrade | Platform team | YYYY-MM-DD | Planned |
```

### Tooling Stack (recommended)

| Dimension | Tool |
|-----------|------|
| DORA metrics | DORA dashboard plugin (Backstage), LinearB, or custom from CI events |
| Structural metrics | SonarQube (complexity, duplication, coupling via DSM) |
| Debt register | Linear / Jira with custom field "Debt Type"; or a tracked markdown file in git |
| Fitness functions | ArchUnit (Java), dependency-cruiser (JS/TS), k6 (perf), OPA (policy) |
| Operational metrics | Prometheus + Grafana; Datadog APM |
| Dashboard aggregation | Backstage (with DORA + SonarQube + Grafana plugins) |

---

## Quick Reference

| Method | Input | Output | Frequency |
|--------|-------|--------|-----------|
| QAW (M1) | Business goals | Prioritised QA list | At start of major initiative |
| Utility Tree (M2) | QA list | Hierarchical priority map | After QAW; update annually |
| QA Scenario (M3) | Single QA | Testable specification | 1 per top-priority QA |
| Lightweight ATAM (M4) | Proposed architecture + QA scenarios | Risks, trade-offs, ADRs | Before major commitment |
| Risk Storming (M5) | Architecture diagram | Risk register additions | Quarterly; after incidents |
| ADR / MADR (M6) | Significant decision | Immutable decision record | Every significant decision |
| Debt Register (M7) | Known shortcuts | Managed debt backlog | Continuously |
| Fitness Function (M8) | QA scenario | Automated CI gate | Per QA scenario |
| Runway Planning (M9) | Feature roadmap + QA scenarios | Enabler story backlog | Per PI / quarter |
| Technology Radar (M10) | Team nominations | Published radar | Quarterly |
| C4 + Structurizr (M11) | Architecture knowledge | Living system diagrams | Per structural change |
| Cognitive Load (M12) | Team structure + scope | Action on team design | Bi-annually or on stress signal |
| Health Dashboard (M13) | All above | Architecture health report | Weekly / monthly |

## References

- Erder, Pureur, Woods — *Continuous Architecture in Practice* (Addison-Wesley, 2021)
- Ford, Parsons, Kua — *Building Evolutionary Architectures* 2nd ed (O'Reilly, 2023)
- Bass, Clements, Kazman — *Software Architecture in Practice* 4th ed (Addison-Wesley, 2022)
- Skelton, Pais — *Team Topologies* (IT Revolution, 2019)
- Forsgren, Humble, Kim — *Accelerate* (IT Revolution, 2018)
- Brown, Simon — [C4 Model](https://c4model.com/)
- [MADR — Markdown Architectural Decision Records](https://adr.github.io/madr/)
- [ThoughtWorks Technology Radar](https://www.thoughtworks.com/radar)
- [DORA Research](https://dora.dev/)
