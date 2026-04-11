# TOGAF Enterprise Architecture Practitioner (OGEA-102)
## Expert Learning Path

---

## Certification at a Glance

TOGAF (The Open Group Architecture Framework) is the world's most widely adopted enterprise architecture framework. The current certification path is based on **TOGAF Standard, 10th Edition** and has two levels:

| Certification | Exam Code | Level | Purpose |
|---------------|-----------|-------|---------|
| TOGAF EA Foundation | OGEA-101 | Part 1 | Validate knowledge of TOGAF terminology, structure, and core concepts |
| TOGAF EA Practitioner | OGEA-102 | Part 2 | Validate ability to *apply* TOGAF to real-world architecture scenarios |

> You must pass OGEA-101 before sitting OGEA-102. Both can also be taken in a single sitting as OGEA-103 (combined exam).

### Exam Format Comparison

| Property | OGEA-101 (Foundation) | OGEA-102 (Practitioner) |
|----------|----------------------|------------------------|
| Questions | 40 multiple choice | 8 scenario-based gradient questions |
| Duration | 60 minutes | 90 minutes |
| Pass mark | 55% (22/40) | 60% (24/40) |
| Open book? | Yes — TOGAF Standard allowed | Yes — TOGAF Standard allowed |
| Scoring | 1 mark per question | Gradient: best answer = 5 marks, worst = 0 |
| Provider | Pearson VUE (online or centre) | Pearson VUE (online or centre) |
| Cost | ~USD 320 (Part 1) | ~USD 320 (Part 2) |

### Important: What "Open Book" Really Means

Both exams allow you to use the TOGAF Standard. This sounds easy — it is not. The exam is designed assuming you have the standard; the advantage only exists if you can navigate it in seconds. Practice tabbing between ADM phases and specific sections under exam conditions.

> **Study goal:** Know the content well enough to need the book only for confirmation, not for discovery.

---

## TOGAF vs Other Certifications You Have Studied

| Dimension | AWS SAP-C02 / DOP-C02 | TOGAF |
|-----------|----------------------|-------|
| Focus | Technical implementation on AWS | Enterprise-wide architecture governance and strategy |
| Scope | Cloud services and automation | Business, Data, Application, Technology domains |
| Audience | Cloud / DevOps engineers | Enterprise architects, solution architects, CTO-adjacent roles |
| Value to you | Deep technical credibility | Strategic architecture credibility; bridges business ↔ technology |
| Synergy | TOGAF governs *what* to build; AWS skills define *how* to build it |

TOGAF makes you significantly more effective at the AWS certs: you will make better architectural decisions because you understand the business motivation behind them.

---

## Prerequisites Self-Assessment

| Area | Minimum Level |
|------|---------------|
| Working in IT programmes or architecture | 2+ years |
| Exposure to system or solution architecture | Basic |
| Understanding of business strategy concepts | Basic |
| Reading technical or business documents | Proficient |

TOGAF has no formal prerequisite exam, but the Part 2 scenario questions assume you can reason about organisational context and architectural trade-offs — not just technical ones.

---

## The TOGAF Mental Model (Read Before Starting)

Before any study, internalise this map. Everything else hangs off it.

```
TOGAF Standard, 10th Edition
│
├── TOGAF Fundamental Content
│   ├── Introduction and Core Concepts
│   ├── Architecture Development Method (ADM)        ← the process engine
│   ├── ADM Guidelines and Techniques                ← how to apply ADM
│   ├── Architecture Content Framework               ← what you produce
│   ├── Enterprise Continuum and Architecture Repository  ← where you store it
│   ├── Architecture Capability Framework            ← how you set up the practice
│   └── Architecture Governance                      ← how you control it
│
└── TOGAF Series Guides (configuration / extensions)
    ├── Business Architecture Guide
    ├── Agile EA Guide
    ├── Digital Transformation Guide
    └── ... (many others)
```

**The single most important concept:** TOGAF is a *method* for doing enterprise architecture, not a prescribed architecture. You configure and adapt it for your organisation. The ADM is the cycle that drives everything.

---

## Phase Overview

```
Phase 0 (Week 1)        → Orientation, mental model, exam strategy
Phase 1 (Weeks 2–3)     → TOGAF foundations and core concepts
Phase 2 (Weeks 4–5)     → ADM phases in depth (Preliminary through H)
Phase 3 (Weeks 6–7)     → Architecture Content Framework and Repository
Phase 4 (Weeks 8–9)     → ADM Techniques and Stakeholder Management
Phase 5 (Weeks 10–11)   → Architecture Governance and Capability Framework
Phase 6 (Weeks 12–13)   → Part 2 scenario practice and application
Phase 7 (Week 14)       → Mock exams, final polish, exam-day prep
```

> **Total study time estimate:** 14 weeks at 5–8 hours/week. Part 1 is achievable in 6–7 weeks; add 7 weeks for Part 2 mastery.

---

## Phase 0 — Orientation and Exam Strategy (Week 1)

### Goal
Understand what TOGAF is actually testing, navigate the standard confidently, and set your study cadence.

### Actions

- [ ] Download the TOGAF Standard, 10th Edition (free registration via [The Open Group](https://www.opengroup.org/togaf-licensed-downloads)) — this is your primary reference.
- [ ] Read the official exam guides:
  - [OGEA-101 datasheet](https://certification.opengroup.org/docs/datasheets/datasheet_togaf_ea_foundation.pdf)
  - [OGEA-102 datasheet](https://certification.opengroup.org/docs/datasheets/datasheet_togaf_ea_practitioner.pdf)
- [ ] Take at least 20 free TOGAF sample questions online to calibrate your starting point. (Sources: The Open Group sample papers, Whizlabs TOGAF free tier.)
- [ ] Build physical or digital tabs/bookmarks for the TOGAF Standard: one per ADM phase, one for each major framework section. You will navigate this under time pressure.
- [ ] Book OGEA-101 exam (6 weeks out). Book OGEA-102 (8 weeks after that).
- [ ] Register at [AWS Skill Builder](https://skillbuilder.aws) is irrelevant here — use [The Open Group Official Training Calendar](https://training.opengroup.org/calendar/) to find accredited courses.

### Key Insight for Part 2
The gradient scoring on Part 2 means **partial credit applies**. If you cannot identify the best answer with certainty, eliminating the worst answer still earns you points. Never leave a scenario unanswered.

### Validation
You can navigate to any ADM phase in the TOGAF Standard in under 30 seconds. You understand how Part 1 and Part 2 differ in what they test.

---

## Phase 1 — TOGAF Foundations and Core Concepts (Weeks 2–3)

**Tested in: OGEA-101 (Part 1) — approximately 40% of Part 1 questions**

### Week 2 — Introduction, Architecture Domains, and Building Blocks

**Topics to master:**

- [ ] **What is enterprise architecture?** The distinction between EA as a discipline, EA as a practice, and EA as deliverables
- [ ] **The Four Architecture Domains (BDAT):**
  - **Business Architecture:** strategy, governance, organisational goals, business capabilities, business processes
  - **Data Architecture:** logical and physical data assets, data management resources
  - **Application Architecture:** individual applications, interactions, relationships to core business processes
  - **Technology Architecture:** logical software and hardware capabilities, infrastructure
- [ ] **Building Blocks:** Architecture Building Blocks (ABBs) vs Solution Building Blocks (SBBs)
  - ABB: a re-usable component that defines required capability (what, not how)
  - SBB: a specific component that realises an ABB (a product, vendor solution, or custom implementation)
- [ ] **Architecture Views and Viewpoints:**
  - Viewpoint: a definition and convention for a particular type of view (template)
  - View: a representation of the architecture from a specific stakeholder's perspective
  - Architect's job: select the right viewpoints for the relevant stakeholders
- [ ] **Deliverables, Artifacts, and Building Blocks:**
  - Deliverable: a contractual output from a project (approved and signed-off)
  - Artifact: an architectural work product describing an aspect of the architecture
  - Building Block: a potentially reusable component of business, IT, or architectural capability
- [ ] **Architecture Principles:** definition, structure (name, statement, rationale, implications), governance role
- [ ] **The Open Group's definition of "architecture"**: The fundamental concepts or properties of a system in its environment embodied in its elements, relationships, and in the principles of its design and evolution.

### Week 3 — TOGAF Standard Structure and Enterprise Continuum

**Topics to master:**

- [ ] **TOGAF Standard structure:** Fundamental Content vs Series Guides — what each contains
- [ ] **Enterprise Continuum:** a categorisation mechanism for architecture and solution artefacts
  - **Architecture Continuum:** Foundation Architectures → Common Systems → Industry → Organisation-Specific
  - **Solutions Continuum:** the implemented realisation of the Architecture Continuum
  - The Continuum runs from Foundation (most generic) to Organisation-specific (most specific); customise by moving right
- [ ] **Architecture Patterns:** how patterns from the Architecture Continuum are re-used and adapted
- [ ] **Architecture Repository:** the storage mechanism for all EA artefacts (see Phase 3 for deep dive)
- [ ] **Architecture Capability:** the ability to perform enterprise architecture work — requires people, process, tools, and governance
- [ ] **TOGAF Library:** the extended reference material (Series Guides, white papers) that sits alongside the Standard
- [ ] **Integrated Information Infrastructure Reference Model (III-RM):** a Foundation Architecture for boundaryless information flow
- [ ] **Technical Reference Model (TRM):** a Foundation Architecture providing a standard taxonomy of technology components and their interfaces

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase1-knowledge.md`](togaf-phase1-knowledge.md) — BDAT domains with examples, ABB vs SBB decision table, views vs viewpoints, deliverable/artifact/building block disambiguation, Architecture Principles full structure, Enterprise Continuum, TRM, TOGAF definitions

**Phase 1 Deliverable:** Without looking at notes, draw the TOGAF Standard structure, the four architecture domains, and the Enterprise Continuum. Label every component. This is the map you will navigate for the next 13 weeks.

**Phase 1 Validation Questions:**
- What is the difference between an ABB and an SBB? Give one example of each.
- What is the difference between an architecture deliverable and an artifact?
- Where does an "organisation-specific architecture" sit on the Architecture Continuum?
- What are the four components of an Architecture Principle?
- A new stakeholder asks "why do we need a data architecture?" — how does TOGAF frame the answer?

---

## Phase 2 — The ADM Phases (Weeks 4–5)

**Tested in: OGEA-101 (Part 1) — approximately 40% of Part 1 questions; core of OGEA-102 (Part 2)**

The ADM is the core of TOGAF. Every scenario question in Part 2 is fundamentally asking: "Given this situation and these stakeholders, what does the ADM say you should do next, and why?"

### The ADM Overview

```
                    ┌─────────────────┐
                    │   Preliminary   │ ← Set up the capability
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Phase A:      │
                    │ Architecture    │ ← Define the vision
                    │    Vision       │
                    └────────┬────────┘
          ┌──────────────────┼──────────────────┐
          │                  │                  │
 ┌────────▼────────┐         │        ┌─────────▼────────┐
 │    Phase B:     │         │        │    Phase C:      │
 │    Business     │        REQ       │  IS Architecture │
 │  Architecture   │      MGMT        │ (Data + App)     │
 └────────┬────────┘    (centre)      └─────────┬────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                    ┌────────▼────────┐
                    │    Phase D:     │
                    │   Technology    │ ← Infrastructure layer
                    │  Architecture  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Phase E:     │
                    │ Opportunities & │ ← Identify projects
                    │   Solutions     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Phase F:     │
                    │   Migration     │ ← Plan the roadmap
                    │   Planning      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Phase G:     │
                    │Implementation   │ ← Oversee delivery
                    │  Governance     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │    Phase H:     │
                    │  Architecture   │ ← Manage change
                    │    Change       │
                    └────────┬────────┘
                             │
                    (back to Phase A or Preliminary)
```

### Week 4 — Preliminary through Phase D

**Topics to master for each phase:**
For every phase, know: **Objectives → Inputs → Steps → Outputs** (deliverables produced).

**Preliminary Phase:**
- [ ] Objectives: establish the architecture capability, define the scope and constraints, tailor the ADM if needed
- [ ] Key activities: define stakeholders, establish governance and support frameworks, select tools, define the Architecture Principles
- [ ] Key outputs: organisational model for EA, tailored ADM map, Architecture Principles, initial Architecture Repository
- [ ] This phase is NOT part of the cycle — it sets up the capability to do architecture

**Phase A — Architecture Vision:**
- [ ] Objectives: develop a high-level, aspirational view of the target state that all stakeholders agree on
- [ ] Trigger: request for architecture work (Statement of Architecture Work)
- [ ] Key inputs: Architecture Principles, organisation strategy, business goals
- [ ] Key outputs: **Statement of Architecture Work** (the mandate), Architecture Vision, Communications Plan, refined stakeholder map
- [ ] Critical concept: get buy-in at the vision level *before* doing the detailed architecture work

**Phase B — Business Architecture:**
- [ ] Objectives: develop the Business Architecture to the degree needed to support the Architecture Vision
- [ ] Techniques: business capability modelling, value stream mapping, business scenario technique, organisation maps
- [ ] Key outputs: Business Architecture document (Baseline + Target), gap analysis, updated Architecture Roadmap
- [ ] This phase produces the *context* in which the other architectures must operate

**Phase C — Information Systems Architecture:**
- [ ] Covers two sub-phases: **Data Architecture** and **Application Architecture**
- [ ] Data Architecture: covers logical and physical data assets; focus on where data lives, who owns it, how it flows
- [ ] Application Architecture: covers the application portfolio, interactions, and relationships to business processes
- [ ] Key outputs: Data and Application Architecture documents, gap analyses

**Phase D — Technology Architecture:**
- [ ] Objectives: map capabilities and components needed to support the logical application and data components
- [ ] Inputs: Application and Data Architecture decisions from Phase C
- [ ] Key outputs: Technology Architecture document (Baseline + Target), gap analysis
- [ ] Platforms, infrastructure, middleware, standards — all defined here

### Week 5 — Phase E through H and Requirements Management

**Phase E — Opportunities and Solutions:**
- [ ] Objectives: identify delivery vehicles (projects, programmes, work packages) that will deliver the target architecture
- [ ] Introduces: **Transition Architectures** — intermediate states between Baseline and Target
- [ ] Key outputs: Project Portfolio, initial Architecture Roadmap, Transition Architecture(s)
- [ ] Critical concept: work packages and their sequencing based on business value and dependencies

**Phase F — Migration Planning:**
- [ ] Objectives: develop a detailed Implementation and Migration Plan
- [ ] Activities: prioritise projects, assess resources and costs, create Migration Plan
- [ ] Key outputs: **Implementation and Migration Plan**, updated Architecture Roadmap, updated Architecture Definition Document
- [ ] Cost/benefit analysis, risk assessment, prioritisation by business value

**Phase G — Implementation Governance:**
- [ ] Objectives: ensure conformance of implementation projects to the architecture
- [ ] Activities: architecture oversight, compliance reviews, Architecture Contracts
- [ ] Key outputs: **Architecture Contract** (agreement between sponsor and implementation team), compliance assessments, updated Architecture Repository
- [ ] Key concept: the architect advises and governs — does not manage the implementation project

**Phase H — Architecture Change Management:**
- [ ] Objectives: establish a process for managing changes to the architecture over time
- [ ] Triggers: technology changes, business events, lessons learned from Phase G
- [ ] Determines: whether a change requires a new architecture cycle (back to Preliminary or A) or can be handled as a simple change request
- [ ] Key outputs: updated Architecture Roadmap, change requests, updated Architecture Repository

**Requirements Management (centre of ADM):**
- [ ] NOT a phase — it is a continuous process that runs throughout the ADM cycle
- [ ] Stores, manages, and governs architecturally significant requirements
- [ ] Requirements feed into each phase and are updated as architecture progresses
- [ ] Architecture Requirements Specification: the formal output documenting quantitative requirements for each architecture domain

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase2-knowledge.md`](togaf-phase2-knowledge.md) — full phase-by-phase breakdown (objectives/inputs/steps/outputs), Statement of Architecture Work contents, Architecture Contract contents, Transition Architecture concept, phase-to-deliverable master table, ADM iteration patterns, Phase H change classification, decision trees

**Phase 2 Deliverable:** Create a one-page ADM phase cheat sheet: for every phase, write 3 bullet points (objectives, key inputs, key outputs). Do this from memory first, then validate against the Standard. Keep this as your exam reference tab.

**Phase 2 Validation Questions:**
- What is the purpose of the Statement of Architecture Work and in which phase is it produced?
- What is the difference between a Transition Architecture and a Target Architecture?
- Why is Requirements Management in the centre of the ADM diagram rather than in a linear sequence?
- What does an Architecture Contract establish and between which parties?
- At the end of Phase H, what are the two possible outcomes for a change request, and what determines which applies?

---

## Phase 3 — Architecture Content Framework and Repository (Weeks 6–7)

**Tested in: OGEA-101 (Part 1) — approximately 20% of questions; context-setting for Part 2 scenarios**

### Week 6 — Architecture Content Framework

**Topics to master:**

- [ ] **Content Metamodel:** The formal model of TOGAF architectural content
  - Core entities: Actor, Role, Business Service, Function, Process, Data Entity, Application Component, Technology Component
  - Extensions for specific concerns: motivation, governance, services, etc.
  - Relationships between entities (association, realisation, assignment, aggregation)
- [ ] **Architecture Artifacts types:**
  - Catalogs: lists of things (e.g., application portfolio catalog, technology standards catalog)
  - Matrices: relationships between items (e.g., Business Interaction Matrix, Actor/Role Matrix)
  - Diagrams: visual representations (e.g., Business Footprint Diagram, Application Communication Diagram)
- [ ] **Deliverables produced per ADM phase:** match every deliverable to its phase (exam question pattern)
  - Architecture Vision → Phase A
  - Business/Data/Application/Technology Architecture → Phases B, C, D
  - Architecture Definition Document → spans B, C, D
  - Architecture Requirements Specification → Requirements Management
  - Architecture Roadmap → Phases E, F
  - Implementation and Migration Plan → Phase F
  - Architecture Contract → Phase G
- [ ] **Architecture Definition Document vs Architecture Requirements Specification:**
  - Architecture Definition Document: describes the *qualitative* aspects of the architecture (what it looks like)
  - Architecture Requirements Specification: describes the *quantitative* aspects (measurable criteria for conformance)
- [ ] **Gap Analysis:** technique for identifying the difference between Baseline and Target architecture — applied in every domain phase (B, C, D)

### Week 7 — Architecture Repository

**Topics to master:**

- [ ] **Architecture Repository components (six classes):**

  | Component | Contents |
  |-----------|---------|
  | Architecture Metamodel | Organisation-specific metamodel (customised from TOGAF Content Metamodel) |
  | Architecture Landscape | Baseline, Transition, and Target architectures at all levels (strategic, segment, capability) |
  | Standards Library | Standards the organisation has adopted (mandatory + optional) |
  | Reference Library | Reference materials from external sources (industry standards, vendor architectures) |
  | Governance Log | Record of governance activity: decisions, dispensations, compliance assessments |
  | Architecture Capability | Documentation of the EA team, processes, skills, and tools |

- [ ] **Architecture Landscape levels:**
  - **Strategic Architecture:** organisation-wide, long-term, high-level
  - **Segment Architecture:** a specific area of the business (e.g., a division, domain, or product line)
  - **Capability Architecture:** specific capability to be deployed (e.g., an identity platform, a data mesh)
- [ ] **The Governance Log:** records Architecture Contracts, dispensations (approved deviations), and compliance reviews — essential for audit and control
- [ ] **Standards Library vs Reference Library:** Standards = adopted by this organisation; Reference = external materials consulted

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase3-knowledge.md`](togaf-phase3-knowledge.md) — catalog/matrix/diagram classification, Content Metamodel entity layers, Architecture Definition Document anatomy, deliverable-to-phase master table, Architecture Repository six components, Architecture Landscape levels

**Phase 3 Deliverable:** Map every artifact from the Content Metamodel to the ADM phase where it is produced. Then map every section of the Architecture Repository to the phase where it is updated. This two-way mapping is a highly effective study method and directly answers many Part 1 questions.

**Phase 3 Validation Questions:**
- What are the six classes of the Architecture Repository?
- What is the difference between the Architecture Metamodel and the Content Metamodel?
- What is the difference between the Architecture Definition Document and the Architecture Requirements Specification?
- What is stored in the Governance Log and why is it important?
- At what level of the Architecture Landscape would you describe an organisation-wide data strategy?

---

## Phase 4 — ADM Techniques and Stakeholder Management (Weeks 8–9)

**This is the primary content of OGEA-102 (Part 2) scenarios.**

Part 2 scenarios always involve: a given organisational context, a set of stakeholders with competing concerns, and an architectural decision that must be made justifiably using TOGAF techniques.

### Week 8 — ADM Techniques

**Topics to master:**

- [ ] **Architecture Principles Management:** how principles are created, approved, amended; conflict resolution between principles; principles as drivers for architectural decisions
- [ ] **Stakeholder Management:** stakeholder identification, classification (power/interest grid), tailoring communication plans, managing concerns vs requirements
- [ ] **Business Scenarios:** technique for discovering and documenting business requirements that motivate architecture work — structured narrative that describes a problem and its business context
- [ ] **Gap Analysis (technique):** baseline → target comparison; identifying what exists, what needs to change, what can be retired. Gap matrix format.
- [ ] **Migration Planning Techniques:**
  - Implementation Factor Assessment and Deduction Matrix
  - Consolidated Gaps, Solutions, and Dependencies Matrix
  - Prioritisation of work packages using business value vs cost
- [ ] **Risk Management in the ADM:** initial (inherent) risk vs residual risk, risk categorisation (H/M/L), risk register, architecture risk in the Governance Log
- [ ] **Capability-Based Planning:** planning, engineering, and delivering strategic business capabilities — distinct from project delivery; used in Phases B and E
- [ ] **Business Transformation Readiness Assessment:** assessing the organisation's readiness and willingness to undergo change — performed in Phase A or E
- [ ] **Interoperability Requirements:** ensuring systems can work together across boundaries — a common Part 2 scenario topic

### Week 9 — Architecture Views, Viewpoints, and Governance Techniques

**Topics to master:**

- [ ] **Selecting Architecture Viewpoints:**
  - Match viewpoints to stakeholder concerns — not all stakeholders need all views
  - Key standard viewpoints: functional, operational, information, physical, availability
  - When a relevant viewpoint does not exist, the TOGAF approach says: create a new viewpoint
- [ ] **Architecture Communications Planning:** who receives which communications at which frequency and in which format — directly governed by stakeholder classification
- [ ] **Architecture Contracts:** content and purpose — commitment by both sponsor and implementing team; compliance measure; change management trigger
- [ ] **Compliance Assessment:** performed in Phase G — determines whether an implementation project conforms to the architecture; outputs: fully compliant, compliant with dispensation, non-compliant
- [ ] **Dispensation:** an approved deviation from the architecture — must be recorded in the Governance Log with justification and expiry
- [ ] **Architecture Governance Processes:**
  - Policy management
  - Compliance management
  - Dispensation management
  - Monitoring and reporting
  - Business control
- [ ] **Partition the ADM:** adapting the ADM to run at multiple levels simultaneously (enterprise-wide and segment-level cycles running in parallel with handshakes)

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase3-knowledge.md`](togaf-phase3-knowledge.md) — Business Scenario technique, Gap Analysis matrix format, Stakeholder management power/interest grid, Capability-Based Planning, Business Transformation Readiness Assessment, Migration Planning techniques, Risk management, interoperability types, Viewpoint selection table

**Phase 4 Deliverable:** Work through three published TOGAF Part 2 sample scenarios in full. For each:
1. Identify which ADM phase the scenario is in.
2. Identify the key stakeholders and their primary concerns.
3. Identify the TOGAF technique most relevant to the scenario.
4. Select and justify your answer.
5. After scoring, trace wrong answers back to the Standard.

**Phase 4 Validation Questions:**
- What is a Business Scenario in TOGAF and when is it used?
- A project team wants to deviate from the approved architecture for commercial reasons — what TOGAF process governs this?
- What is the difference between a stakeholder's "concern" and a stakeholder's "requirement"?
- An architecture principle conflicts with a business goal — how does TOGAF recommend resolving this?
- What is the difference between "fully compliant", "compliant with dispensation", and "non-compliant" in a Phase G compliance assessment?

---

## Phase 5 — Architecture Governance and Capability Framework (Weeks 10–11)

**Tested in: OGEA-101 (Part 1) — final 10–15% of questions; context in Part 2 scenarios**

### Week 10 — Architecture Governance

**Topics to master:**

- [ ] **Architecture Governance:** the practice and orientation by which enterprise architectures and other architectures are managed and controlled at an enterprise-wide level
- [ ] **Governance Levels:**
  - Corporate Governance: board-level oversight
  - Technology Governance: ensures IT systems and services use technology appropriately
  - IT Governance: ensures IT management has effective controls
  - Architecture Governance: ensures architecture is developed, maintained, and complied with consistently
- [ ] **Architecture Board:** who sits on it, what decisions it makes, how it relates to the project portfolio
  - Cross-organisational; typically includes senior stakeholders from business and IT
  - Responsibilities: consistency of architecture, compliance reviews, dispensations, escalated issue resolution
- [ ] **The Architecture Contract:** the primary governance mechanism between the architecture function and project teams — what it must contain (scope, measures of effectiveness, acceptance criteria, risks, constraints)
- [ ] **Architecture Compliance:** types of compliance (irrelevant, consistent, compliant, conformant, fully conformant, non-conformant) — know the spectrum
- [ ] **Architecture Governance Repository:** the Governance Log, Architecture Contracts, compliance assessments, dispensation records

### Week 11 — Architecture Capability Framework

**Topics to master:**

- [ ] **Architecture Capability Framework:** provides guidance on how to organise and establish an enterprise architecture practice
- [ ] **Components of the Architecture Capability:**
  - Architecture Board
  - Architecture Compliance
  - Architecture Contracts
  - Architecture Governance
  - Architecture Maturity Models
  - Architecture Skills Framework
- [ ] **Architecture Maturity Models:** used to assess the current state and target state of an EA capability — supports Preliminary Phase activities
- [ ] **Architecture Skills Framework:** defines roles (e.g., Enterprise Architect, Business Architect, Solution Architect, Domain Architect) and the skills/competencies expected at each level
- [ ] **Setting up an Architecture Practice (Preliminary Phase workflow):**
  1. Determine the scope of organisations impacted
  2. Confirm governance and support frameworks
  3. Define and establish the Architecture Team
  4. Identify and establish Architecture Principles
  5. Tailor the TOGAF framework
  6. Implement architecture tools
- [ ] **Architecture Partitioning:** how to scope the EA practice so large organisations can run multiple simultaneous ADM iterations (by geography, division, capability)

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase4-knowledge.md`](togaf-phase4-knowledge.md) — governance levels (Corporate/Technology/IT/Architecture), four governance processes, Architecture Board composition and responsibilities, Architecture Contract full contents, compliance spectrum (six levels), dispensation lifecycle, Governance Log contents

**Phase 5 Deliverable:** Write a one-page Architecture Governance Charter for a fictional organisation: define the Architecture Board membership and remit, the compliance process, the dispensation process, and the escalation path. Reference specific TOGAF concepts throughout.

**Phase 5 Validation Questions:**
- What is the difference between "consistent" and "conformant" architecture compliance?
- Who is typically on the Architecture Board and what is its mandate?
- What is architecture partitioning and when is it needed?
- In which phase of the ADM is the Architecture Skills Framework most relevant?
- What is the difference between "Corporate Governance" and "Architecture Governance" in TOGAF?

---

## Phase 6 — Part 2 Scenario Practice and Application (Weeks 12–13)

**This phase is 100% focused on preparing for OGEA-102.**

### Understanding Gradient Scoring

Each Part 2 scenario presents a situation and asks: "What is the BEST course of action?" with 5 options labelled (a)–(e). Scoring:

| Answer chosen | Marks awarded |
|--------------|---------------|
| Best answer | 5 |
| Second-best | 3 |
| Third-best | 1 |
| Fourth-best | 0 |
| Worst answer | 0 |

**Strategy:** Always identify and eliminate the two clearly wrong answers first. This guarantees you at least 1 mark per question even if you are uncertain between the remaining three.

### Scenario Pattern Recognition

Part 2 scenarios follow recognisable patterns. Learn to spot them:

| Pattern | Signals in the Scenario | Key TOGAF Concept to Apply |
|---------|------------------------|---------------------------|
| Phase identification | "The architect has just finished..."; "The team is about to begin..." | ADM phase objectives and outputs |
| Stakeholder conflict | "The Finance Director wants X but the CIO wants Y" | Stakeholder management, concerns mapping |
| Governance breach | "The project team has deviated from..." | Compliance assessment, dispensation, Architecture Contract |
| Repository confusion | "The team cannot find..." | Architecture Repository structure |
| Change request triage | "A major technology change has been signalled..." | Phase H, impact classification |
| Transition architecture need | "The organisation cannot achieve the target state in one step..." | Phases E and F, transition architectures |
| Architecture principle conflict | "Principle A says X but the vendor only offers Y..." | Principle management, trade-off analysis |
| Viewpoint selection | "The new regulator needs to see..." | Views and viewpoints, stakeholder concerns |

### Week 12 — Scenario Drill: ADM Application

For each of the following prompt types, write a structured TOGAF answer (2–3 sentences per response):

- [ ] "An organisation is starting its first EA programme. The CEO asks the architect what they should do first." → Preliminary Phase activities
- [ ] "A project manager asks what the architect should produce by the end of Phase B." → Business Architecture deliverables
- [ ] "A project's implementation has deviated significantly from the approved architecture. What should happen?" → Phase G compliance and dispensation
- [ ] "The board has approved a major merger. How does this affect the current architecture cycle?" → Phase H change classification, new ADM cycle trigger
- [ ] "The organisation wants to move from a Baseline Architecture to a Target Architecture but the change is too large for a single programme." → Transition Architectures in Phase E

**Knowledge portal cross-reference:**
- **Deep knowledge:** [`togaf-phase4-knowledge.md`](togaf-phase4-knowledge.md) — gradient scoring strategy, all 8 scenario pattern types with worked examples, what the "best answer" always looks like, Architecture Capability Framework, maturity models, Skills Framework roles, architecture partitioning

### Week 13 — Full Scenario Practice Sets

- [ ] Complete at least 4 full 8-question practice Part 2 exams under timed conditions (90 minutes each).
- [ ] Source: Tutorials Dojo TOGAF Part 2, official Open Group sample scenarios, accredited training provider materials.
- [ ] After each set: for every question with a sub-optimal answer, write out why the best answer was correct by citing the TOGAF Standard chapter and page.
- [ ] Score target before booking: **consistently 65%+ across all 8 questions.**

**Exam-day note:** The TOGAF Standard is available during the exam. Part 2 questions are scenario-based and designed so that lookup alone is insufficient — you need to reason about context. However, use the Standard to verify phase objectives and deliverable names when you are between two plausible answers.

---

## Phase 7 — Mock Exams, Final Polish, and Exam-Day Preparation (Week 14)

### Part 1 (OGEA-101) Mock Exams

Complete under timed conditions (60 minutes, 40 questions):

| Source | Target Score |
|--------|--------------|
| Open Group official sample paper | 75%+ |
| Tutorials Dojo TOGAF Foundation | 78%+ |
| Whizlabs TOGAF Foundation | 80%+ |

### Part 2 (OGEA-102) Mock Exams

Complete under timed conditions (90 minutes, 8 scenarios):

| Source | Target Score |
|--------|--------------|
| Open Group official sample scenarios | 65%+ |
| Tutorials Dojo TOGAF Practitioner | 68%+ |
| Accredited training provider practice paper | 70%+ |

### Exam-Day Technique

**For Part 1 (multiple choice):**
- [ ] If unsure, eliminate obviously wrong answers first — reduce to 2 and decide.
- [ ] Questions about which phase a deliverable comes from: anchor on the phase objectives, not the deliverable name.
- [ ] "Open book" does not mean browsing — go to the Standard only for confirmation of facts you already know.

**For Part 2 (scenarios):**
- [ ] Read the scenario fully before reading the options — form your own answer first, then match to the closest option.
- [ ] Classify scenarios by pattern (see table above) before reading options.
- [ ] Never leave a scenario unanswered — gradient scoring means even a 1-mark answer beats 0.
- [ ] Allocate 10–11 minutes per scenario; do not let one scenario consume more than 15 minutes.

**Known traps for Part 1:**

| Trap | The Trick |
|------|-----------|
| "Architecture principles are created in Phase A" | Wrong — they are created in the **Preliminary Phase** |
| "Requirements Management is Phase 0" | Wrong — it is NOT a phase; it is a continuous process |
| "The Architecture Vision is the Target Architecture" | Wrong — it is a high-level, aspirational view used to get approval, not the full detailed architecture |
| "The Statement of Architecture Work is an artifact" | Wrong — it is a **deliverable** (contractual output) |
| "Transition architectures are created in Phase F" | Partially wrong — they are DEFINED in Phase E, PLANNED in Phase F |
| "The Architecture Board makes decisions about project budgets" | Wrong — it governs architecture compliance, not project finance |
| "An ABB is a specific product" | Wrong — that is an SBB. An ABB defines what is needed, not how it is provided |

**Known traps for Part 2:**

| Trap | The Trick |
|------|-----------|
| "The architect should fix the implementation problem" | The architect governs and advises — the project manager implements. Architect issues compliance assessment and raises it to the Architecture Board. |
| "The stakeholder's requirement should be rejected" | TOGAF preference: document the concern, assess impact, explore dispensation before rejecting |
| "Jump straight to defining the Target Architecture" | Always check if there is an approved Architecture Vision first (Phase A output) |
| "The architect should recommend a specific product" | TOGAF stays technology-agnostic until Technology Architecture (Phase D) — earlier phases use ABBs |

**Two days before the exam:**
- [ ] Review your one-page ADM phase cheat sheet only.
- [ ] Re-read your Architecture Repository component table.
- [ ] Take one 20-question Part 1 warm-up set — no more.
- [ ] Confirm booking, browser/Pearson VUE setup, ID requirements.

---

## Master Cheat Sheet — ADM Phases Summary

| Phase | One-Line Objective | Key Output |
|-------|-------------------|------------|
| Preliminary | Set up the EA capability and tailor TOGAF | Architecture Principles, organisational EA model |
| A — Vision | Get stakeholder agreement on the direction | Statement of Architecture Work, Architecture Vision |
| B — Business | Describe the business architecture | Business Architecture document, gap analysis |
| C — Information Systems | Describe data and application architectures | Data and Application Architecture documents |
| D — Technology | Describe the technology architecture | Technology Architecture document |
| E — Opportunities & Solutions | Identify projects and transition states | Architecture Roadmap, Transition Architectures |
| F — Migration Planning | Plan the transition in detail | Implementation and Migration Plan |
| G — Implementation Governance | Oversee implementation conformance | Architecture Contract, compliance assessments |
| H — Change Management | Manage architecture changes over time | Change requests, updated Roadmap |
| Requirements Management | Continuous — manage arch requirements throughout | Architecture Requirements Specification |

---

## Resources — Prioritised

### Essential (free)
| Resource | URL | Priority |
|----------|-----|----------|
| TOGAF Standard, 10th Edition | [opengroup.org/togaf-licensed-downloads](https://www.opengroup.org/togaf-licensed-downloads) | ★★★★★ |
| Official sample exam papers | [opengroup.org/certifications](https://www.opengroup.org/certifications/togaf-certification-portfolio) | ★★★★★ |
| TOGAF Standard online (9.2) | [pubs.opengroup.org/architecture/togaf92-doc/arch](https://pubs.opengroup.org/architecture/togaf92-doc/arch/) | ★★★★☆ |
| Open Group YouTube channel | [youtube.com/user/theopengroup](https://www.youtube.com/user/theopengroup) | ★★★☆☆ |

### Paid Resources
| Resource | Type | Notes |
|----------|------|-------|
| [Udemy: TOGAF Foundation & Practitioner](https://www.udemy.com/course/togaf/) | Video course | Search for 10th Edition specifically; multiple providers available |
| [Tutorials Dojo TOGAF](https://portal.tutorialsdojo.com/) | Practice exams | Best scenario-based Part 2 questions available |
| [Whizlabs TOGAF](https://www.whizlabs.com/) | Practice exams | Good for Part 1 MCQ volume |
| [Accredited Training Course](https://training.opengroup.org/calendar/) | Instructor-led | 3–5 day intensive; includes Applied Practitioner badge eligibility |

### This Knowledge Portal
| Phase | Docs |
|-------|------|
| Business and domain context | `system-design/`, `distributed-design-pattern/` — understanding *what* architects are designing gives context to *how* TOGAF governs it |
| Technology Architecture (Phase D) | `cloud/aws/`, `devops/03-infrastructure-as-code.md` |
| Application Architecture (Phase C) | `api-design/`, `system-design/` |
| Governance and compliance patterns | `devsecops/13-compliance-as-code.md`, `devsecops/10-policy-as-code.md` |

---

## Final Readiness Gate

Do not sit OGEA-101 until:
```
  [ ] Can draw the full ADM cycle with all phases from memory in 5 minutes
  [ ] Can describe every ADM phase objective in one sentence without notes
  [ ] Can list all 6 Architecture Repository components from memory
  [ ] Can distinguish ABB from SBB, deliverable from artifact, view from viewpoint
  [ ] Part 1 mock exam score consistently ≥ 75%
  [ ] Can navigate to any ADM phase in the TOGAF Standard in < 30 seconds
```

Do not sit OGEA-102 until:
```
  [ ] Can identify the ADM phase of any scenario in < 60 seconds
  [ ] Can name the TOGAF technique most relevant to each scenario pattern type
  [ ] Can explain gradient scoring strategy: eliminate worst first, then reason
  [ ] Part 2 mock exam score consistently ≥ 65%
  [ ] Has completed at least 4 full 8-question timed practice sets
  [ ] Knows when dispensation applies vs when non-compliance must be escalated
```

---

## After TOGAF Practitioner — Progression Paths

| Path | Credential | Why |
|------|-----------|-----|
| Deepen architecture practice | [Applied Practitioner Badge](https://www.credly.com/org/the-open-group/badge/the-open-group-certified-applied-togaf-enterprise-architecture-practitioner) | Earn via accredited training course; shows hands-on application |
| Model architecture artefacts | [ArchiMate 3 Practitioner](https://www.opengroup.org/certifications/archimate) | The standard modelling language for TOGAF artefacts; pairs directly |
| Business architecture depth | [TOGAF Business Architecture Foundation](https://www.opengroup.org/certifications/togaf-certification-portfolio) | Targets business capability modelling, value streams, strategy alignment |
| EA leadership | [TOGAF EA Leader credential](https://www.credly.com/org/the-open-group/badge/the-open-group-certified-togaf-enterprise-architecture-leader) | Governance, communication, and EA programme leadership |
| Combine with AWS | TOGAF + SAP-C02 together | TOGAF defines the architecture; SAP-C02 implements it — this combination is rare and highly valued at senior levels |

---

*Last updated: April 2026 — the TOGAF Standard, 10th Edition exam guide is updated periodically; verify the current Conformance Requirements document (X2202) on The Open Group website before sitting.*
