# TOGAF Phase 1 Deep Knowledge: Foundations and Core Concepts

> Covers Weeks 2–3 of the TOGAF learning path.  
> OGEA-101 (Part 1): ~40% of questions come from this material.

---

## The Four Architecture Domains (BDAT)

TOGAF organises all architectural work into four architecture domains. In practice, they are developed in sequence (B → D → A → T) but iterate against each other.

| Domain | Concerns | Typical Stakeholders | Example Artifacts |
|---|---|---|---|
| **Business** | Strategy, goals, capabilities, processes, org structure, governance | CEO, CFO, business owners, process owners | Business Capability Map, Value Stream, Organisation Chart, Business Process Diagram |
| **Data** | Logical and physical data assets, data flows, data management, ownership, quality | CDO, data owners, privacy officer | Data Entity Catalog, Logical Data Model, Data Flow Diagram, Data Lifecycle |
| **Application** | Application portfolio, interactions between applications, relationships to business processes | CIO, application owners, solution architects | Application Portfolio Catalog, Application Communication Diagram, Interface Catalog |
| **Technology** | Hardware, software platforms, middleware, infrastructure, network, standards | CTO, infrastructure architects, network engineers | Technology Standards Catalog, Network Diagram, Platform Decomposition |

**Key principle:** Business Architecture is always the context for all other domains. Technology Architecture is the most constrained because it must support Application, which must support Data, which must support Business. Work from the outside in.

**Exam trap:** "Technology Architecture" does NOT mean "IT strategy" — strategy belongs to Business Architecture. Technology Architecture describes the platforms and infrastructure components.

---

## Building Blocks: ABB vs SBB

The most commonly tested concept in Part 1. Memorise the distinction cold.

| Property | Architecture Building Block (ABB) | Solution Building Block (SBB) |
|---|---|---|
| Nature | Conceptual, technology-agnostic | Specific, concrete, product-aware |
| Defines | What capability is required | How the capability is implemented |
| Example (identity) | "Authentication Service" | Microsoft Azure AD B2C, AWS Cognito |
| Example (messaging) | "Asynchronous Messaging Platform" | Apache Kafka on EKS |
| Example (compute) | "Container Orchestration Platform" | Kubernetes 1.29 on AWS EKS |
| Phase produced | Primarily Phases B, C, D | Primarily Phase E, F |
| Reuse scope | Cross-organisation, generic | Organisation-specific, often vendor-specific |
| Standard reference | Architecture Continuum (generic end) | Solutions Continuum (specific end) |

**Decision rule:** If you can name a vendor product alongside it, it is probably an SBB. If it describes only what functionality is needed with no implementation commitment, it is an ABB.

**ABB → SBB mapping example:**
```
ABB: "Event-Driven Integration Bus"
  ↓ realised by
SBB: "Amazon EventBridge (custom bus) + AWS Lambda consumers"
```

---

## Architecture Views and Viewpoints

| Term | TOGAF Definition | Analogy |
|---|---|---|
| **Viewpoint** | A *specification* (template) that establishes conventions for constructing, interpreting, and using a view | A type of camera lens (wide-angle, macro, zoom) |
| **View** | A *representation* of the architecture from the perspective of a specific set of stakeholders, using a given viewpoint | The actual photograph taken through that lens |

**Critical exam point:** A view is *always* created from a viewpoint. If no suitable viewpoint exists for a stakeholder's concerns, the TOGAF approach is to **create a new viewpoint** — not to skip the view or force the concern into an unsuitable template.

**Common viewpoints (TOGAF 10th Edition):**

| Viewpoint | Primary Stakeholders | Shows |
|---|---|---|
| Functional | Business analysts, application architects | What the system does — functions and interactions |
| Operational | Operations, service management | How it runs in production — processes and events |
| Information | Data architects, privacy officers | What information is created, used, and stored |
| Physical | Infrastructure architects | Actual hardware, locations, networks |
| Availability | SRE, operations | Uptime, redundancy, failover |

**Relationship diagram:**
```
Stakeholder → has Concerns
Concern → is addressed by → View
View → is constructed using → Viewpoint
Viewpoint → may reference → Model Kind (diagram, table, matrix)
```

---

## Deliverables, Artifacts, and Building Blocks — Disambiguation

This triplet is a guaranteed Part 1 question. The distinctions matter precisely.

| Term | Nature | Formality | Signed Off? | Example |
|---|---|---|---|---|
| **Deliverable** | A contractual work product, formally agreed | Highest — it is a contract output | Yes — approved by sponsor | Statement of Architecture Work, Architecture Contract |
| **Artifact** | An architectural work product describing an aspect of the architecture | Medium — part of a deliverable | Not independently — bundled in deliverables | Application Communication Diagram, Technology Standards Catalog |
| **Building Block** | A reusable component of capability (ABB or SBB) | Captured in artifacts | No — it is a component, not a document | "Authentication Service" (ABB), "AWS Cognito" (SBB) |

**Hierarchy:** Deliverables contain Artifacts. Artifacts describe or reference Building Blocks.

**Memory hook:**
- You *sign off* a **Deliverable** (like a contract)
- You *draw* an **Artifact** (like a diagram or catalog)
- You *implement* a **Building Block** (like a service or component)

---

## Architecture Principles

Architecture Principles are high-level, stable statements that guide architectural decision-making. They are created in the **Preliminary Phase** (not Phase A — a common exam trap).

### Principle Structure (four mandatory components)

| Component | Definition | Example |
|---|---|---|
| **Name** | Short, memorable label | *Technology Independence* |
| **Statement** | Concise declaration of what the principle says | *Applications shall be designed to be independent of any single technology platform* |
| **Rationale** | Why this principle is important to the organisation | *Reduces vendor lock-in, protects the technology investment, allows future platform changes without rewriting applications* |
| **Implications** | What following this principle means in practice (costs, restrictions, actions) | *Development teams must use abstraction layers. Existing tightly-coupled applications are non-compliant and must be remediated in the next cycle. Adds short-term development cost.* |

**Why "Implications" matters on the exam:** TOGAF says principles must acknowledge the real-world impact of following them. An "Implications-free" principle is considered poorly formed.

### Principle Conflict Resolution

When two principles conflict, TOGAF does NOT prescribe a hierarchy — instead, it says:
1. Identify the conflict explicitly.
2. Document both sides in the principle definitions.
3. Escalate to the Architecture Board or appropriate governance body for adjudication.
4. Record the decision and rationale in the Governance Log.

**Never resolve a principle conflict silently** — doing so without governance is itself a governance failure.

### Example Set of Architecture Principles

| Name | Typical Category |
|---|---|
| Single Source of Truth | Data Architecture |
| Business Continuity | Cross-cutting |
| Technology Independence | Application Architecture |
| Common Use Applications | Application Architecture |
| Data is an Asset | Data Architecture |
| Security by Design | Cross-cutting |
| Build to Manage | Technology Architecture |
| Control Technical Diversity | Technology Architecture |

---

## Enterprise Continuum and Architecture Repository

### Enterprise Continuum

The Enterprise Continuum is a **classification mechanism** — it does NOT hold artefacts itself (the Architecture Repository does). It organises them by generality.

```
MOST GENERIC ←──────────────────────────────────────→ MOST SPECIFIC

Architecture Continuum:
Foundation → Common Systems → Industry-Specific → Organisation-Specific

Solutions Continuum:
Foundation → Common Systems → Industry-Specific → Organisation-Specific
```

| Level | Architecture Continuum Example | Solutions Continuum Example |
|---|---|---|
| Foundation | TOGAF TRM (Technical Reference Model) | Open-source Linux, Apache HTTP Server |
| Common Systems | Network Reference Architecture | Cisco MPLS network blueprint |
| Industry-Specific | Retail banking data model | BIAN (Banking Industry Architecture Network) |
| Organisation-Specific | Your company's enterprise architecture | Your company's actual deployed systems |

**How you use it:** When solving an architecture problem, start at the generic end of the continuum (Foundation → Common Systems) before reinventing. Only customise (move right) where the generic pattern genuinely does not fit.

**Architecture Continuum vs Solutions Continuum:**
- Architecture Continuum: the abstract designs, patterns, and frameworks (the "what")
- Solutions Continuum: the concrete implementations of those patterns (the "how")

---

## Technical Reference Model (TRM)

The TRM is a **Foundation Architecture** provided by TOGAF to support boundaryless information flow. It gives a standard taxonomy of technology service categories.

```
┌─────────────────────────────────────────────────────────┐
│              Application Software                       │
├─────────────────────────────────────────────────────────┤
│         Application Platform Interface (API)            │
├──────────────┬──────────────┬───────────────────────────┤
│  Data Exch.  │ Data Mgmt    │  Transaction Processing   │
│  Services    │  Services    │     Services              │
│──────────────┼──────────────┼───────────────────────────│
│  Security    │  User Interface │  Data Interchange      │
│  Services    │  Services       │  Services              │
│──────────────┼─────────────────┼──────────────────────-─│
│              Operating System Services                  │
│──────────────────────────────────────────────────────── │
│              Network Services                           │
└─────────────────────────────────────────────────────────┘
```

**TRM exam use:** The TRM gives a vocabulary for describing technology components without committing to a specific product. Reference it when you need to place a technology capability in context.

---

## TOGAF Standard Structure (10th Edition)

Understanding the structure is essential for open-book navigation under time pressure.

| Section | Contents | When You Turn To It |
|---|---|---|
| Introduction and Core Concepts | Definitions, BDAT, building blocks, views, principles | Part 1 foundation questions |
| Architecture Development Method (ADM) | All phases: Preliminary, A–H, Requirements Management | Every Part 2 scenario |
| ADM Guidelines and Techniques | Techniques for applying the ADM (stakeholder management, gap analysis, scenarios) | Part 2 techniques questions |
| Architecture Content Framework | Content metamodel, artifact types, deliverables | Deliverable/artifact questions |
| Enterprise Continuum and Architecture Repository | Continuum levels, repository components | Repository questions |
| Architecture Capability Framework | Setting up the practice, governance, maturity | Phase 5 — governance questions |

---

## TOGAF Definitions You Must Know Cold

These appear verbatim in exam questions — know the exact TOGAF wording, not paraphrases.

| Term | TOGAF Definition |
|---|---|
| Architecture | The fundamental concepts or properties of a system in its environment, embodied in its elements, relationships, and in the principles of its design and evolution |
| Enterprise Architecture | A description of the structure and interaction of an enterprise's strategy, its business functions, and its technology capabilities — and the principles governing their design and evolution |
| Building Block | Represents a (potentially re-usable) component of enterprise capability that can be combined with other building blocks to deliver architectures and solutions |
| Architecture Principle | A general rule and guideline intended to be enduring and seldom amended, that informs and supports the way in which an organisation sets about fulfilling its mission |
| Stakeholder | An individual, team, organisation, or class thereof having an interest in a system |
| Concern | An interest in a system relevant to one or more stakeholders |
| View | A representation of a system from the perspective of a related set of concerns |
| Viewpoint | A specification of the conventions for constructing and using a view |
| Deliverable | An architectural work product that is contractually specified and in turn formally reviewed, agreed, and signed off by the stakeholders |
| Artifact | An architectural work product that describes an aspect of the architecture — contained within a deliverable |

---

## Common Part 1 Traps — Phase 1 Material

| Trap | Wrong Assumption | Correct Answer |
|---|---|---|
| Architecture Principles are created in Phase A | Phase A is where you agree the vision | Principles are created in the **Preliminary Phase** |
| A View and a Viewpoint are the same thing | Vague terminology confusion | A viewpoint is the *template*; a view is the *instance* produced using it |
| An SBB is always better than an ABB | More detail = better | At early phases, ABBs are preferred to stay technology-agnostic |
| The Architecture Repository is the Enterprise Continuum | Often conflated | The Enterprise Continuum is a *classification mechanism*; the Repository is the *storage system* |
| The Solutions Continuum holds technical architectures | Confusing the two continua | The Architecture Continuum holds architectures; the Solutions Continuum holds implementations |
| A concern is the same as a requirement | Used interchangeably in practice | A concern is a stakeholder's *interest*; a requirement is a *formal statement of need* arising from a concern |
| Building Blocks = code components | Technical framing | Building Blocks can be business, data, application, or technology capabilities |
| The TRM is mandatory to use | It sounds official | The TRM is a *reference model* — you are expected to adapt or replace it |
