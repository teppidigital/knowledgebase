# TOGAF Phase 2 Deep Knowledge: ADM Phases In Depth

> Covers Weeks 4–5 of the TOGAF learning path.  
> Core of OGEA-102 (Part 2) and ~40% of OGEA-101 (Part 1).

---

## The ADM Mental Model

The ADM is an **iterative, cyclical process** — not a one-time waterfall. Every question about "what should happen next" has a phase-anchored answer.

```
           ┌──────────────────────────────────┐
           │         PRELIMINARY              │  ← Sets up the capability
           └──────────────┬───────────────────┘       (runs once, or on major restructure)
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE A                  │  ← Architecture Vision
           │     (Get buy-in on direction)    │
           └──────────────┬───────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
┌────────▼────────┐   REQUIREMENTS   ┌─────▼────────────┐
│    PHASE B      │   MANAGEMENT     │    PHASE C       │
│ Business Arch.  │   (continuous)   │ IS Architecture  │
└────────┬────────┘       │          └─────┬────────────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE D                  │  ← Technology Architecture
           └──────────────┬───────────────────┘
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE E                  │  ← Opportunities & Solutions
           └──────────────┬───────────────────┘
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE F                  │  ← Migration Planning
           └──────────────┬───────────────────┘
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE G                  │  ← Implementation Governance
           └──────────────┬───────────────────┘
                          │
           ┌──────────────▼───────────────────┐
           │         PHASE H                  │  ← Architecture Change Management
           └──────────────┬───────────────────┘
                          │
              (back to A or Preliminary)
```

**Three types of ADM iteration:**
1. **Cycle iteration** — completing the full cycle and starting a new architecture engagement
2. **Phase iteration** — re-visiting a previous phase because new information requires rework (e.g., Phase D reveals a constraint that invalidates a Phase C decision)
3. **ADM at different levels** — running simultaneous enterprise-level and segment-level ADM cycles with formal handshake points

---

## Preliminary Phase

**Purpose:** Establish the architecture capability *before* doing any architecture work. This is NOT part of the iterative cycle — it is a pre-condition.

### Objectives
- Define the enterprise context and scope
- Identify key stakeholders and sponsors
- Establish the architecture governance structures
- Tailor the TOGAF framework to the organisation's needs
- Define the Architecture Principles

### Key Activities (6-step sequence)

| Step | Activity |
|---|---|
| 1 | Determine the scope of organisations impacted |
| 2 | Confirm governance and support frameworks |
| 3 | Define and establish the Architecture Team |
| 4 | Identify and establish Architecture Principles |
| 5 | Tailor the TOGAF framework and select tools |
| 6 | Implement architecture tools |

### Key Outputs
- **Organisational Model for EA** — scope, boundaries, budget, stakeholders
- **Tailored ADM** — customised TOGAF framework for this organisation
- **Architecture Principles** — the rules that govern all architecture work
- **Initial Architecture Repository** — set up and ready for use
- **Architecture Governance Framework** — processes and bodies established

**Exam trap:** Architecture Principles are *defined* in the Preliminary Phase. They are *referenced* and *applied* throughout subsequent phases — especially Phase A, B, C, D.

---

## Phase A — Architecture Vision

**Purpose:** Establish the foundation for the architecture work — get senior stakeholder buy-in on the direction before investing in detailed architecture.

### Objectives
- Develop a high-level aspirational view of the target state
- Identify stakeholders and their concerns
- Confirm business goals and strategic drivers
- Obtain approval to proceed (Statement of Architecture Work)

### Trigger
Typically triggered by a **Request for Architecture Work** from the sponsoring organisation or by a business event (merger, regulatory change, strategic programme).

### Key Inputs
- Architecture Principles (from Preliminary)
- Organisation strategy and business goals
- Architecture Repository (existing landscapes)
- Constraints: time, budget, geography, legal

### Key Steps
1. Establish the architecture project scope
2. Identify stakeholders and their concerns
3. Confirm business goals, objectives, and constraints
4. Assess business capabilities (gap analysis at high level)
5. Assess readiness for change
6. Define the Architecture Vision
7. Define the Target Architecture Value Propositions
8. Identify architecture risks
9. Develop the Statement of Architecture Work
10. Obtain approval

### Key Outputs

| Output | Description |
|---|---|
| **Statement of Architecture Work** | The formal committed scope — this is a *deliverable* (signed off). It is NOT the architecture itself. |
| **Architecture Vision** | High-level view of Business, Data, Application, and Technology architectures for the target state |
| **Communications Plan** | Which stakeholders receive which communications, at which frequency, in which format |
| **Refined Stakeholder Map** | Updated with concerns, classifications (power/interest), and communication preferences |

### Statement of Architecture Work — Contents

The Statement of Architecture Work is the *mandate* for the architecture engagement. It must contain:
- Scope and objectives of the architecture work
- Schedule and constraints
- Acceptance criteria (how success is measured)
- Risks and issues acknowledged
- Stakeholder roles and responsibilities
- Authority to proceed with the architecture engagement

**Critical exam point:** The Statement of Architecture Work is a **deliverable** (contractually agreed and signed off), not an artifact. It is produced in Phase A.

---

## Phase B — Business Architecture

**Purpose:** Develop the Business Architecture to the degree needed to support the Architecture Vision.

### Objectives
- Describe the Baseline ("as-is") Business Architecture
- Develop the Target ("to-be") Business Architecture
- Analyse the gaps between Baseline and Target
- Identify candidate components of an Architecture Roadmap

### Key Techniques Used in Phase B

| Technique | What It Produces | When to Use |
|---|---|---|
| **Business Capability Modelling** | A map of what the organisation needs to be able to do | Defining scope; identifying capability gaps |
| **Value Stream Mapping** | End-to-end view of value delivery from trigger to outcome | Identifying process waste; alignment with business value |
| **Business Scenario Technique** | Narrative of a problem and its business context | Discovering and validating requirements |
| **Organisation Map** | Roles, units, and reporting relationships | Stakeholder analysis; identifying change impact |
| **Gap Analysis** | Matrix comparing Baseline vs Target | Identifying what must change |

### Key Outputs
- **Business Architecture Document** — Baseline + Target + gap analysis
- Updated **Architecture Roadmap** (candidate components only at this stage)
- **Gap Analysis** for Business domain
- Updated **Architecture Repository** (business catalog entries)

**Why Phase B matters to every other phase:** All subsequent architectures must be *justified by business need*. If an application or technology component cannot be traced back to a business capability or process, it should be challenged.

---

## Phase C — Information Systems Architecture

**Purpose:** Develop the Data and Application architectures needed to support the Business Architecture.

Phase C has TWO sub-phases, and the order matters:

```
Phase C: Information Systems Architecture
   ├── Data Architecture  (preferred first)
   └── Application Architecture
```

**Data Architecture first rationale:** Data entities and their ownership define constraints that applications must respect. Application design should follow data design, not precede it.

### Data Architecture

| Activity | Output |
|---|---|
| Identify data entities relevant to the business | Data Entity Catalog |
| Map data entities to business functions and processes | Business/Data Matrix |
| Define data ownership and stewardship | Data Stewardship Register |
| Identify data flows between systems | Data Flow Diagram |
| Define data lifecycle (create, use, archive, delete) | Data Lifecycle Diagram |

### Application Architecture

| Activity | Output |
|---|---|
| Define the application portfolio | Application Portfolio Catalog |
| Map applications to business functions | Application/Business Function Matrix |
| Define interactions between applications | Application Communication Diagram |
| Map applications to data entities | Application/Data Matrix |
| Identify application standards and patterns | Application Standards Catalog |

### Gap Analysis in Phase C
Gap analysis for Data and Application is performed separately, then combined for the Architecture Roadmap.

**Common gap categories:**
- **Eliminated:** component in Baseline that is not in Target (deprecated/retired)
- **New:** component in Target that does not exist in Baseline (must be built/procured)
- **Modified:** component exists in both but requires changes to support Target
- **Retained:** component exists in both and carries over unchanged

---

## Phase D — Technology Architecture

**Purpose:** Define the technology components (software platforms, hardware, networks, middleware) needed to deploy and support the Application and Data architectures.

### Objectives
- Describe the Baseline Technology Architecture
- Develop the Target Technology Architecture
- Identify gaps and candidate solution components
- Map ABBs to SBBs (first time product/vendor consideration enters the cycle)

### Key Activities
1. Review request for architecture work
2. Identify Technology Architecture reference models
3. Develop Baseline Technology Architecture description
4. Develop Target Technology Architecture description
5. Perform gap analysis
6. Define candidate requirements

### Key Outputs
- **Technology Architecture Document** (Baseline + Target + gap analysis)
- Updated **Architecture Roadmap** with technology components
- Technology Standards Catalog

**Technology Architecture input sources:**
- Application components from Phase C → need hosting platforms
- Data flows from Phase C → need network and storage
- Business requirements for availability and performance → need redundancy, scalability

**Phase D vs Phase C boundary:** Phase C defines *logical* application and data components. Phase D defines *physical* implementation platforms. An Application Architecture says "we need a message broker"; Technology Architecture says "we need Kafka on dedicated brokers with specific network configuration."

---

## Phase E — Opportunities and Solutions

**Purpose:** Bridge from architecture definition to implementation planning. Identify how the Target Architecture will actually be delivered.

### Objectives
- Identify major implementation projects/work packages
- Identify Transition Architectures if the change is too large for a single step
- Group work packages into logical programmes
- Produce the initial Architecture Roadmap with delivery vehicles

### Transition Architectures — Critical Concept

A **Transition Architecture** is an intermediate state of the enterprise architecture between the Baseline and the Target. It is used when:
- The target state is too large to achieve in one programme
- Business operations cannot tolerate a single big-bang cutover
- Dependencies between work packages require a staged sequence

```
Baseline ──→ Transition Architecture 1 ──→ Transition Architecture 2 ──→ Target

Example:
Current monolith ──→ Strangled monolith + new microservices ──→ Full microservices
```

**Transition Architecture defined in Phase E, planned in Phase F.** This distinction is a common exam trap.

### Work Packages
Work packages are discrete, bounded pieces of work that collectively deliver a Transition or Target Architecture. They must be:
- Identifiable and scoped
- Assignable to a delivery programme
- Sequenced based on dependencies and business value

### Key Outputs
- **Architecture Roadmap** (initial, with work packages)
- **Transition Architecture(s)** documents
- **Project Portfolio** — candidate projects that need to be commissioned
- **Implementation Factor Assessment and Deduction Matrix** — captures factors (positive/negative) that affect implementation

---

## Phase F — Migration Planning

**Purpose:** Develop a detailed Implementation and Migration Plan from the Architecture Roadmap and Project Portfolio defined in Phase E.

### Objectives
- Finalise the Architecture Roadmap
- Prioritise projects based on value, cost, and dependencies
- Assess resources and costs
- Create the Implementation and Migration Plan

### Key Techniques in Phase F

| Technique | Purpose |
|---|---|
| **Consolidated Gaps, Solutions, and Dependencies Matrix** | Maps all gaps (from B, C, D gap analyses) to solutions and their dependencies |
| **Architecture Definition Increments Table** | Shows which components are delivered in which increment (Transition Architecture to Transition Architecture) |
| **Cost/benefit analysis** | Justifies project prioritisation |
| **Risk Assessment** | Identifies and documents implementation risks |

### Migration Planning Approach
1. Confirm management framework for implementation
2. Assign business value to each work package
3. Estimate resource requirements and availability
4. Prioritise projects: high-value + low-cost + low-risk first
5. Confirm interactions with ongoing programmes
6. Create the Implementation and Migration Plan

### Key Outputs
- **Implementation and Migration Plan** — the primary output of Phase F; a detailed plan for moving from Baseline to Target via Transition Architectures
- **Finalised Architecture Roadmap** — sequenced and resourced
- **Updated Architecture Definition Document** — refined based on migration constraints

---

## Phase G — Implementation Governance

**Purpose:** Ensure that implementation projects conform to the agreed architecture.

**Critical mindset:** The architect in Phase G is a **governance authority**, not an implementation manager. The architect does NOT run the projects — the project manager does. The architect oversees, reviews, and enforces architectural conformance.

### Objectives
- Confirm scope and priorities with the portfolio manager
- Identify deployment resources and skills
- Guide development of solutions deployment
- Perform enterprise architecture compliance reviews
- Implement business and IT operations

### Architecture Contracts

The **Architecture Contract** is the primary governance mechanism in Phase G. It is a **bilateral agreement** between:
- **The development organisation** (the project team delivering the solution)
- **The sponsoring organisation / architecture board** (who approved the architecture)

**Architecture Contract must contain:**
- Scope of work
- Measures of effectiveness (how conformance is assessed)
- Acceptance criteria
- Risks and constraints
- Responsibilities and authorities
- Signature of all parties

### Compliance Assessment Outcomes

| Outcome | Meaning |
|---|---|
| **Fully Conformant** | Implementation matches the architecture in all respects |
| **Conformant** | Implementation substantially matches; minor deviations are acceptable |
| **Compliant with Dispensation** | Implementation deviates from architecture with formal approval |
| **Non-Conformant** | Implementation deviates without approval — action required |

**Dispensation:** A formal, time-limited approval to deviate from the architecture. Must be:
- Requested by the project team
- Reviewed by the architect
- Approved by the Architecture Board
- Recorded in the Governance Log with justification and expiry date

---

## Phase H — Architecture Change Management

**Purpose:** Manage the ongoing maintenance and evolution of the architecture after implementation is complete.

### Objectives
- Ensure that the Architecture Lifecycle is maintained
- Ensure that the Architecture Board and other governance bodies operate to coordinate change
- Ensure that the enterprise architecture continues to deliver business value

### Change Triggers

| Trigger Type | Examples |
|---|---|
| Business triggers | Merger/acquisition, regulatory change, strategic pivot, market disruption |
| Technology triggers | New technology opportunity, vendor EOL, security vulnerability |
| Lessons learned | Issues found in Phase G, post-implementation review findings |
| External triggers | Legislative change, competitor disruption, industry standard published |

### Change Classification — The Key Decision

The central question in Phase H: **Is this change simple or does it require a new ADM cycle?**

| Change Type | TOGAF Term | Response |
|---|---|---|
| Minor change | Simplification change | Handle within existing governance — no new ADM cycle |
| Significant change, planned | Incremental change / planned change | Re-enter the ADM at the appropriate phase (often Phase A) |
| Major change, unplanned | Re-architecting change | Trigger a new full ADM cycle from Phase A or even Preliminary |

**Decision criteria for a new ADM cycle:**
- Does the change affect the business strategy or goals?
- Does it affect multiple architecture domains simultaneously?
- Does it break the current Architecture Vision or Target Architecture?
- Is the change cost above a defined threshold?

If YES to any of the above → trigger a new ADM cycle.

### Key Outputs
- **Architecture Change Requests** — formal records of change requests and decisions
- **Updated Architecture Roadmap** — reflecting change impact
- **Updated Architecture Repository** — with new baseline reflecting implemented changes

---

## Requirements Management (Centre of the ADM)

Requirements Management is NOT a phase. It is a **continuous process** that runs throughout the entire ADM cycle.

### Purpose
- Store, manage, and govern all architecturally significant requirements
- Ensure requirements are tracked as they evolve across phases
- Prevent requirement loss or inconsistency between phases

### How It Works

```
Business Goal/Constraint
    ↓
Architecturally Significant Requirement
    ↓ (stored in)
Architecture Requirements Specification
    ↓ (feeds into)
Each ADM Phase
    ↓ (updates back)
Requirements Management Process
```

### Architecture Requirements Specification
This is the formal deliverable of Requirements Management. It contains:
- Quantitative criteria against which the architecture must be assessed
- Performance criteria, availability criteria, security criteria
- Constraints that must not be violated
- It is the *measurable* counterpart to the *descriptive* Architecture Definition Document

**Distinction:**
- **Architecture Definition Document** — qualitative description of the architecture ("what it looks like")
- **Architecture Requirements Specification** — quantitative criteria ("how we know it works")

---

## Phase-to-Deliverable Master Table

Memorise this. It is tested directly and indirectly in every scenario.

| Phase | Primary Deliverables Produced |
|---|---|
| **Preliminary** | Architecture Principles, Organisational Model for EA, Tailored ADM, Architecture Governance Framework |
| **A — Vision** | Statement of Architecture Work ✱, Architecture Vision, Communications Plan, Refined Stakeholder Map |
| **B — Business** | Business Architecture document (Baseline + Target), Gap Analysis (Business), updated Roadmap |
| **C — IS (Data + App)** | Data Architecture document, Application Architecture document, Gap Analyses (Data + App) |
| **D — Technology** | Technology Architecture document (Baseline + Target), Gap Analysis (Technology) |
| **E — Opps & Solutions** | Architecture Roadmap (initial), Transition Architecture(s), Project Portfolio |
| **F — Migration** | Implementation and Migration Plan ✱, Finalised Architecture Roadmap |
| **G — Governance** | Architecture Contract ✱, Compliance Assessments, Dispensation Records |
| **H — Change** | Architecture Change Requests, Updated Roadmap, Updated Repository |
| **Req. Management** | Architecture Requirements Specification |

✱ = **Deliverable** (signed off). All others are deliverables too, but these three are the most frequently tested as "contractual outputs."

---

## ADM Iteration Patterns — Exam Scenarios

### Scenario: "A new merger has been announced mid-cycle"
- **Phase:** Currently in Phase D (Technology Architecture)
- **TOGAF response:** Phase H is triggered. Assess the change classification:
  - If merger fundamentally changes business goals → New ADM cycle from Phase A
  - If merger only adds a new business unit → Phase iteration; revisit Phase B scoping
- **Never:** Just carry on from Phase D and ignore the business change

### Scenario: "A work package in Phase G has significantly deviated from the approved architecture"
- **Phase:** Phase G (Implementation Governance)
- **TOGAF response:**
  1. Raise compliance concern via Architecture Contract governance
  2. Issue a compliance warning to the project team
  3. Escalate to Architecture Board
  4. If deviation was commercially justified: architect recommends dispensation
  5. If deviation is technically wrong: mandate conformance or escalate to programme board
- **Never:** Simply approve the deviation without governance

### Scenario: "Phase C Data Architecture has revealed a constraint that makes a Phase B decision invalid"
- **Phase:** Phase C (IS Architecture)
- **TOGAF response:** Phase iteration — invoke Requirements Management, update the Architecture Requirements Specification, loop back to Phase B to revise the impacted Business Architecture
- **This is expected and healthy** — the ADM explicitly supports phase iteration

### Scenario: "The target state cannot be reached in a single programme"
- **Phase:** Phase E (Opportunities and Solutions)
- **TOGAF response:** Define one or more Transition Architectures — intermediate states that deliver business value progressively while building toward the full Target Architecture

---

## Decision Tree: Which ADM Phase Is This?

Use this decision tree when a Part 2 scenario describes an activity and asks what phase the architect is in.

```
Is the organisation setting up its EA practice for the first time?
  YES → Preliminary Phase

Has the architecture engagement just been formally approved and scoped?
  YES → Phase A (Vision) just completed OR just starting Phase B

Is the architect defining business capabilities, processes, or value streams?
  YES → Phase B (Business Architecture)

Is the architect mapping data entities, applications, and their interactions?
  YES → Phase C (IS Architecture)

Is the architect defining platforms, infrastructure, and middleware?
  YES → Phase D (Technology Architecture)

Is the architect identifying projects, work packages, and transition states?
  YES → Phase E (Opportunities and Solutions)

Is the architect creating a detailed delivery plan with costs and priorities?
  YES → Phase F (Migration Planning)

Is the architect reviewing a project's conformance to the approved architecture?
  YES → Phase G (Implementation Governance)

Has a significant business or technology change been signalled after implementation?
  YES → Phase H (Architecture Change Management)
```

---

## ADM Exam Trap Summary

| Trap | Wrong | Correct |
|---|---|---|
| Architecture Vision = Target Architecture | Vision is just a high-level aspiration | Architecture Vision is a high-level view to gain approval; the full Target is developed across B, C, D |
| Statement of Architecture Work is an artifact | It sounds like a document | It is a **deliverable** — contractually agreed and signed off |
| Transition Architectures are created in Phase F | Phase F is mentioned with them | Defined in Phase **E**; planned and resourced in Phase **F** |
| The architect manages implementation in Phase G | Seems logical | The architect *governs* (advises and reviews); the project manager manages |
| Requirements Management is "Phase 0" | It is always numbered | Requirements Management is NOT a phase — it is a continuous process |
| Phase B is done once, then Phase C starts | Waterfall thinking | Phases iterate; Phase C can loop back to Phase B if new constraints emerge |
| Architecture Contract is produced in Phase A | It is about commitment | Architecture Contract is a Phase **G** deliverable |
| Gap analysis only happens in Phase E | That is where solutions appear | Gap analysis is performed in every domain phase: B, C, and D |
