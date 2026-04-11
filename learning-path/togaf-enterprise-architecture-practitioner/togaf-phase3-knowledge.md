# TOGAF Phase 3 Deep Knowledge: Content Framework, Repository, and ADM Techniques

> Covers Weeks 6–9 of the TOGAF learning path.  
> Phase 3 (Content Framework + Repository) feeds ~20% of OGEA-101 Part 1.  
> Phase 4 (ADM Techniques) is the primary content of OGEA-102 Part 2 scenarios.

---

## Architecture Content Framework

### The Three Artifact Types

Every artifact produced during architecture work is one of three types:

| Type | What It Is | TOGAF Examples |
|---|---|---|
| **Catalog** | A list or inventory of architecture elements | Application Portfolio Catalog, Technology Standards Catalog, Data Entity Catalog, Business Service/Function Catalog |
| **Matrix** | A cross-reference showing relationships between two or more elements | Business Interaction Matrix, Actor/Role Matrix, Application/Data Matrix, Technology/Application Matrix |
| **Diagram** | A visual representation of architectural concepts or relationships | Business Footprint Diagram, Value Chain Diagram, Application Communication Diagram, Network Zone Map, Platform Decomposition Diagram |

**Memory rule:** If it lists things → Catalog. If it cross-references two lists → Matrix. If it draws relationships → Diagram.

---

### Content Metamodel — Core Entities

The Content Metamodel defines the types of architectural elements (entities) and their relationships. You do not need to memorise every entity for the exam, but you must understand the structure.

**Core entity layers:**

```
MOTIVATION LAYER
  Driver ──influences──► Goal ──refined into──► Objective
                                   ↑
                              Principle

BUSINESS LAYER
  Actor ──assigned to──► Role
  Role ──performs──► Business Function / Process
  Business Function/Process ──delivers──► Business Service
  Business Service ──enabled by──► Application Service

DATA LAYER
  Business Function ──creates/uses──► Data Entity

APPLICATION LAYER
  Application Component ──realises──► Application Service
  Application Component ──processes──► Data Entity

TECHNOLOGY LAYER
  Technology Component ──realises──► Technology Service
  Technology Component ──hosts──► Application Component
```

**Key relationships to know:**
- **Association:** two elements have a relationship (e.g., Application Component is associated with a Business Service)
- **Realisation:** one element implements another (SBB realises ABB; Technology Component realises Application Component)
- **Assignment:** a role is assigned to a process or actor
- **Aggregation:** a composite element is made up of sub-elements

---

### Architecture Definition Document Anatomy

The Architecture Definition Document is the primary deliverable for Phases B, C, and D combined. It spans all architecture domains.

| Section | Content | Which Phase |
|---|---|---|
| Architecture Vision summary | High-level target recap | Phase A reference |
| Business Architecture (Baseline) | Current-state business | Phase B |
| Business Architecture (Target) | Target-state business | Phase B |
| Business Architecture (Gap) | What changes, what stays, what is retired | Phase B |
| Data Architecture (Baseline + Target + Gap) | Data entities, flows, ownership | Phase C |
| Application Architecture (Baseline + Target + Gap) | Application portfolio, interactions | Phase C |
| Technology Architecture (Baseline + Target + Gap) | Platforms, infrastructure, middleware | Phase D |

**Architecture Definition Document vs Architecture Requirements Specification:**

| Property | Architecture Definition Document | Architecture Requirements Specification |
|---|---|---|
| Nature | Qualitative — describes *what* the architecture looks like | Quantitative — defines *measurable criteria* for conformance |
| Content | Baseline, Target, Gap narratives and diagrams | Performance metrics, service levels, security constraints |
| Purpose | Communicate and agree the architecture | Assess and verify conformance during Phase G |
| Format | Text + diagrams + catalogs | Measurable statements, testable criteria |
| Example entry | "The Target Application Architecture consolidates 18 legacy apps into 4 capability platforms" | "Application response time shall be ≤ 200ms at p95 under 5,000 concurrent users" |

---

### Deliverable-to-Phase Master Mapping

A direct exam question pattern: "Which phase produces [deliverable]?"

| Deliverable | Phase Produced | Deliverable or Artifact? |
|---|---|---|
| Architecture Principles | Preliminary | Deliverable |
| Organisational Model for EA | Preliminary | Deliverable |
| Tailored ADM | Preliminary | Deliverable |
| Statement of Architecture Work | Phase A | Deliverable ★ |
| Architecture Vision | Phase A | Deliverable |
| Stakeholder Map / Communications Plan | Phase A | Deliverable |
| Business Architecture document | Phase B | Deliverable |
| Gap Analysis (Business) | Phase B | Artifact |
| Data Architecture document | Phase C | Deliverable |
| Application Architecture document | Phase C | Deliverable |
| Application/Data Matrix | Phase C | Artifact |
| Technology Architecture document | Phase D | Deliverable |
| Technology Standards Catalog | Phase D | Artifact |
| Architecture Definition Document | Phases B–D (spans all) | Deliverable |
| Architecture Requirements Specification | Requirements Management | Deliverable |
| Architecture Roadmap (draft) | Phase E | Deliverable |
| Transition Architecture(s) | Phase E | Deliverable |
| Project Portfolio | Phase E | Artifact |
| Implementation and Migration Plan | Phase F | Deliverable ★ |
| Finalised Architecture Roadmap | Phase F | Deliverable |
| Architecture Contract | Phase G | Deliverable ★ |
| Compliance Assessment | Phase G | Artifact |
| Dispensation Record | Phase G | Artifact → stored in Governance Log |
| Architecture Change Request | Phase H | Artifact |

★ = The three most-tested contractual deliverables

---

## Architecture Repository — Six Components

The Architecture Repository is the **storage system** for all architecture artefacts and governance records. It has exactly six components. Know all six.

| Component | What It Contains | Exam Trigger |
|---|---|---|
| **Architecture Metamodel** | The organisation-specific customisation of the TOGAF Content Metamodel — defines the entity types and relationships the organisation uses | "Where does the organisation define what architecture element types they use?" |
| **Architecture Landscape** | The actual architectures at all three levels: Strategic, Segment, Capability — Baseline, Transition, and Target | "Where is the current approved architecture stored?" |
| **Standards Library** | Standards the organisation has formally adopted — mandatory and optional | "Where does the architect check which technology standards are approved?" |
| **Reference Library** | External reference materials consulted — industry standards, vendor architectures, TOGAF Series Guides | "Where do external architecture patterns and best practices sit?" |
| **Governance Log** | Record of all governance activities: decisions, contracts, compliance assessments, dispensations, issues | "Where is the dispensation record kept?" |
| **Architecture Capability** | Documentation of the EA practice itself: the team, its skills, tools, processes, and capability maturity | "Where does the EA team's operating model reside?" |

**Distinction: Standards Library vs Reference Library:**
- Standards Library: things the organisation has *decided to adopt* as mandatory or recommended
- Reference Library: things the organisation *consults* as external input — not necessarily adopted

**Distinction: Architecture Metamodel vs Content Metamodel:**
- Content Metamodel: the generic TOGAF-defined set of entity types (part of the TOGAF Standard)
- Architecture Metamodel: the organisation's *customised* version — a subset and/or extension of the Content Metamodel, stored in the Repository

---

### Architecture Landscape Levels

| Level | Scope | Typical Owner | Example |
|---|---|---|---|
| **Strategic Architecture** | Enterprise-wide, long-term, broad strokes | Group CTO / EA Board | "Cloud-first, API-first platform strategy for the next 5 years" |
| **Segment Architecture** | A specific division, business domain, or product line | Domain Architect | "Retail banking digital channels architecture" |
| **Capability Architecture** | A specific capability to be deployed, regardless of business unit | Solution Architect | "Group-wide identity and access management platform architecture" |

**How they relate:**
A Strategic Architecture sets the direction and constraints. Segment Architectures must conform to it. Capability Architectures must conform to both the Strategic and their respective Segment Architecture.

---

## ADM Techniques (Week 8)

### Business Scenario Technique

A Business Scenario is a structured technique for **discovering and documenting business requirements** that motivate architecture work. It is NOT just a use case or user story — it is a narrative at the enterprise business level.

**When to use:** Phase A (to populate the Architecture Vision) and Phase B (to confirm Business Architecture requirements).

**Business Scenario structure:**

| Element | Description | Example |
|---|---|---|
| **Problem description** | What problem needs solving | "Customer on-boarding takes 14 days, losing 30% of applicants to competitors" |
| **Business and technical environment** | Context — what systems, processes, people are involved | "Three legacy systems, manual handoffs, no shared customer identity" |
| **Objectives and success measures** | What success looks like | "On-boarding in 2 days, <5% drop-off, 99.9% SLA on customer portal" |
| **Human actors** | People involved and their roles | Customer, Relationship Manager, Compliance Officer |
| **Computer actors** | Systems involved | CRM, KYC platform, Core Banking |
| **Role/responsibilities** | Who does what | Compliance Officer approves KYC; CRM triggers on-boarding workflow |
| **Desired outcome** | What the architecture must enable | "Integrated digital on-boarding with straight-through processing to Core Banking" |

---

### Gap Analysis Technique

Gap Analysis is performed in Phases B, C, and D. It identifies the difference between the Baseline ("as-is") and Target ("to-be") architecture.

**Gap Analysis Matrix Format:**

```
                   TARGET ARCHITECTURE
              ┌──────────┬──────────┬──────────┐
              │  Comp X  │  Comp Y  │  Comp Z  │
BASELINE  ────┼──────────┼──────────┼──────────┤
│ Comp A  │   │ Retained │ Modified │          │
│ Comp B  │   │          │          │  (Gap)   │
│ Comp C  │   │Eliminated│          │          │
│ (New)   │   │          │  New     │          │
          ────┘──────────┴──────────┴──────────┘
```

**Gap categories:**

| Category | Description | Typical Action |
|---|---|---|
| **Retained** | Exists in both Baseline and Target, unchanged | Document and carry forward |
| **Modified** | Exists in both, but must change to support Target | Define change requirements |
| **Eliminated** | In Baseline but not in Target — deprecated or retired | Decommission plan, data migration |
| **New** | In Target but not in Baseline — must be created or procured | Build/buy decision, procurement |
| *(Blank cell)* | Not relevant — not in Baseline, not in Target | Ignored |

---

### Stakeholder Management Technique

**Stakeholder identification** must happen in Phase A and be maintained throughout the cycle.

**Power/Interest Grid classification:**

```
HIGH POWER
│  Manage Closely  │  Keep Satisfied  │
│  (Key Decision   │   (Board level,  │
│   Makers, Exec   │   not day-to-day)│
├──────────────────┼──────────────────│
│  Keep Informed   │    Monitor       │
│  (Technical      │  (Low priority   │
│   teams, users)  │   stakeholders)  │
LOW POWER
LOW INTEREST ────────────────── HIGH INTEREST
```

**Concerns vs Requirements — critical distinction:**

| Term | Definition | Example |
|---|---|---|
| **Concern** | A stakeholder's *interest* in the architecture | "The CIO is concerned about vendor lock-in" |
| **Requirement** | A *formal, specific statement of need* derived from a concern | "All application components must be deployable without a proprietary hosting contract" |

**Architecture Communications Planning** — based on stakeholder classification:

| Stakeholder Class | Communication | Frequency | Format |
|---|---|---|---|
| Executive sponsors | Progress, risk, business value | Monthly | Executive dashboard, 1-pager |
| Architecture Board | Architecture decisions, compliance | Per governance cycle | Formal architecture review pack |
| Project teams | Architecture standards, patterns | As required | Technical briefs, pattern library |
| End users | Impact of change | Milestone-based | Plain-language change communications |

---

### Capability-Based Planning

**Definition:** A business-planning technique that focuses on the planning, engineering, and delivery of **strategic business capabilities** as the primary unit of planning, rather than projects or systems.

**When used in TOGAF:**
- Phase B: defining what business capabilities are needed
- Phase E: packaging delivery around capability increments rather than system deployments

**Capability vs Project:**

| Property | Capability | Project |
|---|---|---|
| Scope | A repeatable, sustained business ability | A time-bounded effort to deliver an output |
| Success criteria | "Can the business do X?" | "Was deliverable Y produced?" |
| Lifecycle | Ongoing — capabilities exist after projects finish | Temporary — closes on delivery |
| Example | "Real-time fraud detection" | "Implement ML fraud scoring engine in Q3" |

**How they relate:** A project *builds* towards a capability. Multiple projects may contribute to one capability. The capability exists permanently; the project does not.

---

### Business Transformation Readiness Assessment

**When:** Performed in Phase A or Phase E.  
**Purpose:** Assess the organisation's readiness and willingness to undergo the change implied by the Target Architecture.

**Assessment dimensions:**

| Dimension | What to Assess | Low Readiness Signal |
|---|---|---|
| Vision | Is there a clear, shared vision for change? | Conflicting exec narratives |
| Desire, Willingness And Resolve | Do leaders actively want this change? | Change driven by compliance only |
| Need | Is there a business case / urgency? | "Nice to have" perception |
| Business Case | Is there a quantified ROI? | No metrics defined |
| Funding | Is budget allocated? | Dependent on future approval |
| Sponsorship & Leadership | Is there an executive sponsor? | Multiple sponsors with conflicting priorities |
| Governance | Are decision-making structures clear? | No change ownership |
| Accountability | Who is accountable for change outcomes? | Diffused responsibility |
| Workable Solutions | Do viable solution options exist? | Unproven technology dependencies |
| Enterprise Capacity | Does the organisation have bandwidth for this change? | Multiple concurrent large programmes |
| Enterprise Ability | Does the organisation have the skills? | Skill gap with no training plan |

**Output:** A readiness rating (H/M/L) per dimension, and an overall transformation readiness score that feeds into Phase E migration planning.

---

### Migration Planning Techniques

#### Implementation Factor Assessment and Deduction Matrix

Captures factors (constraints, risks, opportunities) that will affect implementation decisions.

| Factor Type | Examples |
|---|---|
| **Positive factors** | Existing capability that can be reused, vendor relationships, existing contracts |
| **Negative factors** | Legacy dependencies, regulatory constraints, skills gaps, time-to-market pressure |
| **Constraints** | Budget ceiling, regulatory freeze periods, existing contractual obligations |

#### Consolidated Gaps, Solutions, and Dependencies Matrix

Combines gap analyses from Phases B, C, and D into a single view showing:
- Gap (what is missing)
- Solution (what will address the gap)
- Dependency (what the solution depends on)
- Sequencing (which solutions must precede others)

#### Architecture Definition Increments Table

Maps which architectural components are delivered in which Transition Architecture increment:

| Component | Baseline | Transition 1 | Transition 2 | Target |
|---|---|---|---|---|
| Customer Portal | v1 monolith | v1 monolith + API gateway | Decomposed services | Full microservices |
| Identity Platform | Legacy LDAP | LDAP + SSO federation | SSO + MFA mandatory | Cloud-native IAM |
| Data Platform | Operational DBs only | Data lake (raw) | Data lake + marts | Unified data mesh |

---

### Interoperability Requirements

In TOGAF, interoperability refers to the ability of systems and components to exchange and use information across organisational, technical, or geographic boundaries.

**The "Boundaryless Information Flow" concept** from TOGAF's TRM/III-RM:
- Information must be able to flow freely between applications, business units, and partner organisations without requiring bespoke integration for each connection

**Interoperability types:**

| Type | Description | Example |
|---|---|---|
| **Syntactic** | Systems can exchange data in a shared format | Both systems accept JSON REST APIs |
| **Semantic** | Systems interpret data the same way | "Customer" means the same entity in both systems |
| **Process** | Systems support compatible workflows | Order placed in CRM triggers picking in WMS without manual handoff |
| **Organisational** | Policies and governance allow data sharing | Data sharing agreement in place; privacy controls aligned |

**Phase relevance:** Interoperability requirements surface in Phase B (business process boundaries) and are realised in Phase C (application integration patterns) and Phase D (technology integration standards).

---

### Risk Management in the ADM

Risk is managed throughout the ADM cycle. The key TOGAF-specific concepts:

| Term | Definition |
|---|---|
| **Initial (Inherent) Risk** | The risk before any mitigation actions are taken |
| **Residual Risk** | The risk that remains after mitigation measures are applied |
| **Risk Categorisation** | H (High) / M (Medium) / L (Low) — based on probability × impact |
| **Risk Register** | Maintained in the Architecture Repository (Governance Log section) |

**Risk classification for architecture change (Phase H):**
- Low-risk change → handle as a simplification change (no new ADM cycle)
- High-risk change → trigger a new ADM cycle

---

## Views and Viewpoints — Technique Deep Dive

### Creating a New Viewpoint (when an existing one doesn't fit)

TOGAF explicitly states that when no standard viewpoint addresses a stakeholder's concern, the architect **must create a new viewpoint**. The process:

1. Identify the stakeholder and their concern
2. Determine what questions the view must answer
3. Define the model kind (type of representation: diagram, table, matrix)
4. Define the notation and conventions
5. Document the new viewpoint in the Architecture Repository (Standards Library or Architecture Metamodel)

**Exam pattern:** A scenario presents a new regulatory requirement (e.g., a data residency view). The architect must create a new viewpoint rather than forcing the concern into an existing viewpoint.

### Viewpoint Selection for Common Stakeholder Concerns

| Concern | Most Appropriate Viewpoint | Artifact Type |
|---|---|---|
| "How does the business work end-to-end?" | Business Footprint / Value Chain | Diagram |
| "Which applications support which business functions?" | Application to Business Function | Matrix |
| "Where does data flow between systems?" | Data Dissemination / Data Flow | Diagram |
| "What platforms host which applications?" | Technology / Platform Decomposition | Diagram |
| "What is the network security perimeter?" | Network Zone Map / Physical | Diagram |
| "How do our applications communicate?" | Application Communication | Diagram |
| "What are all the business processes and who owns them?" | Process Flow / Actor/Role Matrix | Diagram + Matrix |
| "Which data entities are most sensitive?" | Data Security Classification | Catalog + Matrix |

---

## Phase 3 Exam Trap Summary

| Trap | Wrong | Correct |
|---|---|---|
| Architecture Repository and Enterprise Continuum are the same | Often confused | Continuum classifies; Repository stores |
| Architecture Metamodel = Content Metamodel | Same term, different scope | Content Metamodel = TOGAF's generic model; Architecture Metamodel = your org's customisation |
| "Gap analysis only happens in Phase E" | Phase E produces the solutions | Gap analysis is performed in **B, C, and D** — Phase E addresses the gaps |
| "Standards Library holds external reference materials" | Sounds like a library | External materials go in the **Reference Library**; Standards Library holds internally adopted standards |
| "A concern is a requirement" | Often treated interchangeably | A concern is a stakeholder interest; a requirement is a formal, specific statement derived from a concern |
| "Architecture Definition Document lists requirements" | It describes the architecture | Requirements go in the **Architecture Requirements Specification** |
| "Governance Log is only for decisions" | Sounds like a decision log | Governance Log also holds Architecture Contracts, dispensation records, and compliance assessments |
| "Capability-based planning means IT capability" | Technical framing | Capability-based planning is a **business** planning technique focused on business capabilities |
| "The ADM produces diagrams" | Diagrams = deliverables | Diagrams are **artifacts** — they are part of deliverables, not deliverables themselves |
