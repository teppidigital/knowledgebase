# TOGAF Phase 4 Deep Knowledge: Governance, Capability Framework, and Part 2 Scenarios

> Covers Weeks 10–13 of the TOGAF learning path.  
> Architecture Governance and Capability Framework feed the final 10–15% of OGEA-101.  
> Part 2 scenario patterns are the core of OGEA-102.

---

## Architecture Governance Deep Dive

### Four Levels of Governance

TOGAF positions Architecture Governance within a hierarchy of organisational governance. Understanding where it sits is itself an exam question.

| Level | Scope | Who Owns It | Typical Examples |
|---|---|---|---|
| **Corporate Governance** | Board-level, company-wide | Board of Directors | Sarbanes-Oxley compliance, risk appetite, fiduciary duty |
| **Technology Governance** | Ensures IT systems use technology appropriately | CTO, Technology Committee | Technology standards, vendor policy, cloud strategy |
| **IT Governance** | Ensures IT management has effective controls | CIO, IT Management | ITIL service management, COBIT framework, change management |
| **Architecture Governance** | Ensures architecture is developed, maintained, and complied with consistently | Chief Architect, Architecture Board | ADM compliance, dispensation management, standards enforcement |

**Key point:** Architecture Governance sits *within* and *must be consistent with* IT Governance, Technology Governance, and Corporate Governance. It does not replace them.

---

### Architecture Governance — Four Processes

Architecture Governance operates through four core processes:

| Process | What It Does | Output |
|---|---|---|
| **Policy Management** | Establishes and maintains architecture policies and standards | Policy documents, Standards Library updates |
| **Compliance Management** | Ensures projects conform to architecture decisions | Compliance assessments, Architecture Contracts |
| **Dispensation Management** | Formally approves justified deviations | Dispensation records in Governance Log |
| **Monitoring and Reporting** | Tracks architecture health and governance effectiveness | Governance dashboards, executive reports |

---

### Architecture Board

**Role:** The Architecture Board is the cross-functional governance body responsible for overseeing the enterprise architecture practice and ensuring architectural consistency across the organisation.

**Typical composition:**

| Role | Representation |
|---|---|
| Chief Architect (Chair) | Enterprise architecture |
| Business Architecture Representative | Business unit leads |
| Data Architecture Representative | CDO / data governance |
| Application Architecture Representative | CIO / application owners |
| Technology Architecture Representative | CTO / infrastructure |
| External Architect (optional) | Independent perspective |

**What the Architecture Board does:**

| Responsibility | Examples |
|---|---|
| Consistency and completeness | Review proposed architectures for gaps and conflicts |
| Compliance decisions | Approve Architecture Contracts for major projects |
| Dispensation approval | Approve time-limited deviations from the architecture |
| Conflict resolution | Adjudicate between competing architectural principles or stakeholder demands |
| Standards management | Approve additions to the Standards Library |
| Escalation handling | Receive and resolve architectural non-compliance escalated from Phase G |

**What the Architecture Board does NOT do:**
- Manage project budgets or timelines
- Make technology purchasing decisions (unless architecturally significant)
- Manage business strategy
- Run projects

**Exam trap:** The Architecture Board governs architecture conformance — not project management or IT service delivery.

---

### Architecture Contract — Full Content Requirement

The Architecture Contract is the bilateral governance agreement between the development team (implementing the solution) and the architecture function (approving the architecture). It is the primary mechanism by which Phase G governance operates.

**Mandatory contents:**

| Section | What It Defines |
|---|---|
| **Introduction and background** | Context, business objectives, strategic alignment |
| **Nature and scope of the engagements covered** | What this contract covers; what it excludes |
| **Objectives and business value** | What success looks like; business outcomes |
| **Architecture scope** | Which domains, which systems, which organisational units |
| **Strategic requirements** | Constraints and directions from the Architecture Vision and Phase A |
| **Conformance requirements** | Specific architectural standards and patterns that must be followed |
| **Architecture adoption requirements** | How and when architectural patterns must be followed |
| **Measures of effectiveness** | How conformance will be assessed (test criteria, metrics) |
| **Acceptance criteria** | The definition of "done" for architectural conformance |
| **Change request procedures** | How architectural changes during implementation are handled |
| **Risk and mitigation** | Known risks and agreed mitigation |
| **Responsibilities** | Who is accountable for what |
| **Signature of all parties** | Formal acceptance |

---

### Compliance Spectrum — Six Levels

TOGAF defines six levels of architecture compliance that a project may achieve:

| Level | Description | Action Required |
|---|---|---|
| **Irrelevant** | Architecture is not applicable to this project | No action — architecture simply does not apply |
| **Consistent** | The project's work is in the same style as the architecture | No action — good sign |
| **Compliant** | The project substantially conforms to the architecture | Minor issues may exist but no formal action needed |
| **Conformant** | The project fully conforms to all architectural requirements | Architecture Contract accepted |
| **Fully Conformant** | The project exceeds conformance expectations | Recognised and documented as exemplary |
| **Non-Conformant** | The project deviates from the architecture without approval | Formal governance action — escalate, mandate change, or approve dispensation |

**Practical exam note:** The distinction between "compliant" and "conformant" is subtle. TOGAF uses them intentionally — compliant implies general alignment; conformant implies specific formal fulfilment.

---

### Dispensation Process

A dispensation is a **time-limited, formally approved deviation** from the architecture. It is NOT a refusal to engage governance — it IS the appropriate governance response when a deviation is commercially or technically justified.

**Dispensation lifecycle:**

```
Project team identifies a need to deviate from architecture
    ↓
Project team submits a formal Dispensation Request to the architect/Architecture Board
    ↓
Architect reviews:
    - Is the deviation technically valid?
    - Is it commercially justified?
    - Can the architecture be amended to accommodate it permanently?
    - What is the risk if dispensation is granted?
    ↓
Architecture Board approves or rejects
    ↓ (if approved)
Dispensation record created in Governance Log:
    - What deviation is approved
    - For which project/system
    - Justification
    - Expiry date (dispensations are always time-limited)
    - Conditions attached
    ↓
At expiry: project must either bring back into conformance or renew dispensation
```

**Exam pattern:** When a scenario shows a project team that cannot meet an architecture requirement for a legitimate reason, the answer is NEVER "reject them" — it is "initiate the dispensation process."

---

### Governance Log Contents

The Governance Log is the audit trail of all architecture governance activity. It is a section of the Architecture Repository.

| Record Type | Contents |
|---|---|
| Architecture Contracts | Signed contracts with project teams |
| Compliance Assessments | Formal assessments of project conformance |
| Dispensation Records | All approved deviations, with justification and expiry |
| Architecture Decisions | Key architectural decisions and their rationale |
| Issues Register | Active architectural issues being managed |
| Risk Register | Architectural risks and their mitigation status |
| Architecture Board Minutes | Decisions and actions from board meetings |

---

## Architecture Capability Framework

### Components of the Architecture Capability

The Architecture Capability Framework defines how to set up and operate an effective EA practice. It has six components:

| Component | Description |
|---|---|
| **Architecture Board** | The governance body (covered above) |
| **Architecture Compliance** | The compliance processes and assessments |
| **Architecture Contracts** | The contract templates and management process |
| **Architecture Governance** | The governance framework and policies |
| **Architecture Maturity Models** | Models for assessing EA practice maturity |
| **Architecture Skills Framework** | Roles, skills, and competencies for EA practitioners |

---

### Architecture Maturity Models

Used to assess the current and target state of the EA practice's maturity. Applied primarily in the **Preliminary Phase** when setting up or improving the architecture capability.

**Typical maturity levels (TOGAF aligns to general maturity model scales):**

| Level | Name | Characteristic |
|---|---|---|
| 0 | None | No EA practice; ad hoc and reactive |
| 1 | Initial | EA exists informally; depends on individuals |
| 2 | Under Development | EA processes defined but not consistently followed |
| 3 | Defined | EA process is standardised, documented, and followed |
| 4 | Managed | EA outcomes are measured and tracked |
| 5 | Optimising | Continuous improvement; EA drives strategic decisions |

**Exam use:** If a scenario describes an organisation "establishing its first EA practice" → Preliminary Phase, aiming for maturity levels 2–3. If the scenario describes a mature organisation "improving its EA governance" → may still be in Preliminary or Phase H (continuous improvement).

---

### Architecture Skills Framework

Defines the roles and responsibilities of EA practitioners, and the skills required at each level.

**Key roles:**

| Role | Scope | Typical Skills |
|---|---|---|
| **Enterprise Architect** | Organisation-wide; all domains | Strategic thinking, business alignment, governance, all four architecture domains |
| **Business Architect** | Business domain | Business process analysis, capability modelling, value stream mapping, stakeholder management |
| **Solution Architect** | A specific solution or programme | Technical depth, integration design, pattern selection, trade-off analysis |
| **Domain Architect** | One architecture domain (Data, App, Tech) | Deep expertise in the domain + understanding of adjacent domains |

**Relevant phase:** Phase A and Phase B benefit most from Business Architect input. Phase D benefits from Technology/Domain Architect input. The Enterprise Architect spans all phases.

---

### Setting Up an Architecture Practice — Preliminary Phase Workflow

When a scenario describes an organisation "setting up architecture for the first time," the Preliminary Phase workflow applies:

```
Step 1: Determine the scope of organisations impacted
         (which business units, geographies, legal entities)
Step 2: Confirm governance and support frameworks
         (does a governance structure exist? who sponsors EA?)
Step 3: Define and establish the Architecture Team
         (skills audit, hiring, role definitions)
Step 4: Identify and establish Architecture Principles
         (consult with business and IT leaders; draft principles)
Step 5: Tailor the TOGAF framework and select tools
         (which ADM aspects to adopt, which to adapt, which repository tool)
Step 6: Implement architecture tools
         (set up repository, modelling tools, governance tools)
```

---

### Architecture Partitioning

**Definition:** Dividing the ADM into multiple simultaneous iterations at different levels of scope, so that a large organisation can run enterprise-level, segment-level, and capability-level ADM cycles in parallel.

**When needed:**
- The organisation is too large for a single monolithic EA cycle
- Different business divisions have significantly different architecture needs
- Different timescales or rates of change apply to different parts of the business

**How partitioned cycles interact:**

```
ENTERPRISE-LEVEL ADM CYCLE (Strategic Architecture)
    ↑ provides principles, standards, constraints
    ↓ receives feedback on feasibility, gaps
SEGMENT-LEVEL ADM CYCLE (e.g., Retail Banking)
    ↑ provides domain requirements and patterns
    ↓ receives implementation feedback
CAPABILITY-LEVEL ADM CYCLE (e.g., Identity Platform)
```

**Handshake points:** Each level feeds requirements down and constraints up. Changes at a lower level that affect higher-level decisions must be escalated through Phase H.

---

## OGEA-102 Part 2: Scenario Mastery

### Gradient Scoring Strategy

**Maximum score per question: 5 points (best answer)**

```
Best answer:    5 points
Second best:    3 points
Third best:     1 point
Fourth best:    0 points
Worst answer:   0 points
```

**Optimal strategy:**
1. Read the scenario fully before looking at options
2. Form your own answer based on TOGAF principles
3. Classify the scenario by pattern (see below)
4. Identify and eliminate the two worst answers (usually easily recognisable)
5. From the remaining 3, select the most TOGAF-aligned option
6. Never leave a question blank — 1 point for third-best beats 0

**The two worst answers are almost always:**
- "Do nothing / wait for the problem to resolve"
- "Bypass governance / skip a phase / take unilateral action"

---

### Scenario Pattern Recognition — Worked Examples

#### Pattern 1: Phase Identification

**Scenario signal:** "The architect has just completed [specific activity] and is now producing [specific output]"

**Example scenario:**
> "An organisation has completed its first enterprise-wide review of business capabilities, produced a Business Capability Map showing gaps, and is now preparing the Business Architecture document. What should the architect do next?"

**Approach:**
- The described activities are Phase B outputs → the architect is at the **end of Phase B**
- Next step: validate the gap analysis, then move into Phase C (IS Architecture)
- Best answer: "Proceed to Phase C to develop the Data and Application Architectures, using the Business Architecture gaps as input"
- Worst answer: "Jump directly to Phase E to identify implementation projects" (skips C and D)

---

#### Pattern 2: Stakeholder Conflict

**Scenario signal:** "Two stakeholders have opposing views / requirements that appear incompatible"

**Example scenario:**
> "The Finance Director wants all financial data to be stored in a single on-premises database for audit reasons. The Chief Digital Officer wants a cloud-first technology strategy. How should the architect handle this conflict?"

**Approach:**
- This is a **principle conflict / stakeholder concern conflict** — standard governance response
- Best answer: "Document both concerns, assess the impact of each on the architecture, and escalate to the Architecture Board for adjudication. Record the decision in the Governance Log."
- Good answer: "Create an Architecture Principle covering data residency and present it to both stakeholders"
- Poor answer: "Adopt the Finance Director's position as it is a compliance requirement" (unilateral decision)
- Worst answer: "Adopt the CDO's cloud-first position as the strategic direction is correct" (same problem)

---

#### Pattern 3: Governance Breach

**Scenario signal:** "A project team has deviated from / cannot meet / has ignored the approved architecture"

**Example scenario:**
> "During Phase G, the architecture review reveals that a project team has chosen a vendor database that is not on the approved technology standards list, citing a shorter time-to-market. What is the appropriate response?"

**Approach:**
- This is a Phase G compliance assessment scenario
- Best answer: "Issue a formal compliance warning to the project team. If the deviation is technically justified, initiate the dispensation process with the Architecture Board. Record all actions in the Governance Log."
- Good answer: "Meet with the project team to understand the justification and assess whether the Standards Library should be updated"
- Poor answer: "Mandate that the team switch to an approved database regardless of project impact" (ignores dispensation process)
- Worst answer: "Accept the deviation as the project is already committed" (no governance action)

---

#### Pattern 4: Repository / Artefact Confusion

**Scenario signal:** "The team cannot find / does not know where to look / is uncertain what exists"

**Example scenario:**
> "A new architect joins the team and needs to find: (a) the technology standards the organisation has adopted, (b) the approved variances from those standards, and (c) the current enterprise-level architecture. Where should they look?"

**Approach:**
- (a) Technology standards → **Standards Library** in the Architecture Repository
- (b) Approved variances (dispensations) → **Governance Log** in the Architecture Repository
- (c) Current enterprise architecture → **Architecture Landscape** in the Architecture Repository
- Best answer: "Direct the architect to the Architecture Repository, specifically the Standards Library for (a), the Governance Log for (b), and the Architecture Landscape for (c)"

---

#### Pattern 5: Change Request Triage (Phase H)

**Scenario signal:** "A significant business or technology change has been signalled"; "the board has approved a strategic restructure"

**Example scenario:**
> "Following a major acquisition, the organisation's business model has fundamentally changed. The current Target Architecture was developed for the pre-acquisition structure. What should the architect do?"

**Approach:**
- Major business model change → triggers Phase H assessment
- The change affects the **Architecture Vision** (Phase A output) → full ADM cycle restart
- Best answer: "Initiate Phase H assessment of the change impact. Given the fundamental change to business goals and strategy, recommend a new ADM cycle beginning at Phase A with a revised Statement of Architecture Work"
- Good answer: "Assess the impact of the change using Phase H change classification, and recommend re-entering the ADM at Phase A"
- Poor answer: "Continue the current Target Architecture with minor amendments" (underestimates impact)
- Worst answer: "The architect should advise the organisation not to proceed with the acquisition" (outside scope)

---

#### Pattern 6: Transition Architecture Need

**Scenario signal:** "The organisation cannot achieve the target state in one step"; "the change is too large / disruptive for a single programme"

**Example scenario:**
> "The organisation's Target Architecture replaces its 15-year-old monolithic core banking system with a cloud-native microservices platform. The business cannot tolerate a big-bang cutover. What should the architect recommend?"

**Approach:**
- Large-scale change requiring staged delivery → **Transition Architectures** in Phase E
- Best answer: "Define one or more Transition Architectures in Phase E that deliver business value progressively while building toward the Target Architecture. Each Transition Architecture represents a stable intermediate state from which business operations can continue."
- Good answer: "Develop a phased Architecture Roadmap showing how each Transition Architecture enables incremental migration while keeping the core banking platform operational"
- Poor answer: "Split the Target Architecture into two separate Target Architectures" (wrong concept — a Target is singular)
- Worst answer: "Recommend a big-bang cutover with a rollback plan" (ignores the stated constraint)

---

#### Pattern 7: Architecture Principle Conflict

**Scenario signal:** "An architecture principle prevents us from doing something commercially beneficial"; "two principles are in conflict"

**Example scenario:**
> "The Architecture Principle 'Control Technical Diversity' states that no more than two vendors should provide any single type of platform. A business sponsor wants to adopt a third cloud provider to achieve geographic coverage. What should the architect do?"

**Approach:**
- Conflict between a principle and a business requirement → standard principle conflict governance
- Best answer: "Document the conflict between the principle and the business requirement. Assess the implications of amending or creating an exception to the principle. Present the options (amend the principle, create a time-limited exception, find an alternative solution) to the Architecture Board for a governance decision. Record the decision in the Governance Log."
- Good answer: "Creating a new Architecture Principle that refines 'Control Technical Diversity' to allow a third provider in specific geographic circumstances, with Architecture Board approval"
- Poor answer: "Reject the business sponsor's request because it violates an architecture principle"
- Worst answer: "Approve the third cloud provider without governance action"

---

#### Pattern 8: Viewpoint Selection

**Scenario signal:** "A new stakeholder needs to see the architecture from a specific perspective"; "there is no existing view that addresses [concern]"

**Example scenario:**
> "A new data protection regulator requires evidence of how personal data flows between the organisation's systems, including which third parties receive data. No existing architecture view covers this requirement. What should the architect do?"

**Approach:**
- No existing viewpoint covers this concern → **create a new viewpoint**
- Best answer: "Define a new 'Data Residency and Third-Party Sharing' viewpoint that addresses the regulator's concern. Create the corresponding view using the new viewpoint. Add the viewpoint definition to the Architecture Repository for future use."
- Good answer: "Create a new viewpoint that captures personal data flows and third-party relationships, add it to the Standards Library"
- Poor answer: "Use the Data Architecture diagram and annotate it to show the required information" (misses the concept of viewpoint creation)

---

### What "Best Answer" Always Looks Like in Part 2

The best answer in TOGAF Part 2 scenarios consistently:

1. **Follows the ADM process** — does not skip phases or shortcuts
2. **Invokes governance** before making architectural decisions
3. **Involves stakeholders** at the appropriate phase and level
4. **Documents everything** — in the Repository, Governance Log, or relevant deliverable
5. **Uses TOGAF terminology** — gaps are analysed, not "worked around"; deviations are dispensed, not "approved informally"
6. **Does not go outside the architect's remit** — the architect governs, advises, and designs; the project manager delivers

The worst answer almost always:
- Skips governance
- Takes unilateral action
- Ignores stakeholder concerns
- Claims the architect manages implementation

---

## Architecture Governance — Relationship to Other Frameworks

| Framework | Relationship to TOGAF |
|---|---|
| **COBIT** | IT governance framework — Architecture Governance sits within the IT governance structures COBIT defines |
| **ITIL** | IT service management — complements TOGAF's Architecture Contract governance with operational service governance |
| **ISO 38500** | Corporate governance of IT — TOGAF aligns to this standard for board-level IT governance |
| **Zachman Framework** | Another enterprise architecture framework — TOGAF and Zachman address different concerns; they can be used together |
| **ArchiMate** | The standard modelling language for TOGAF artefacts — ArchiMate notation is used to express TOGAF architecture content |
| **PRINCE2 / PMI** | Project management methodologies — Phase G Architecture Contracts interface with project governance structures from these methodologies |

---

## Exam Trap Summary — Governance and Scenarios

| Trap | Wrong | Correct |
|---|---|---|
| "Architecture Board manages project budgets" | Sounds like an oversight body | Architecture Board governs architecture *compliance* only, not project management |
| "A dispensation means the architecture is wrong" | Treating deviation as a failure | A dispensation is a *legitimate governance mechanism* for justified deviation |
| "Compliant = Fully Conformant" | Near-enough thinking | TOGAF explicitly distinguishes the 6 levels; "compliant" is below "conformant" |
| "Jump to Phase E to fix an implementation problem" | Phase E produces solutions | Implementation problems are handled in Phase G via compliance assessment and dispute with Architecture Board |
| "Architecture Principles can be overridden by a project manager" | Projects take precedence | Principles can only be overridden by a formal Architecture Board governance decision |
| "Architecture Capability Framework is about IT capability" | Too narrow | It covers the *EA practice* capability: people, process, tools, governance |
| "The worst Part 2 answer is always technical" | Technical options can be correct | The worst answer is always the one that bypasses governance or takes unilateral action |
| "Phase H only triggers on large changes" | Only big events matter | Phase H should be triggered for *any* change that may affect the current architecture — size is assessed *as part of Phase H* |
| "Architecture maturity models are used in Phase G" | Phase G is compliance | Maturity models are used in the **Preliminary Phase** to establish the EA capability baseline |
| "A non-conformant project must be stopped" | Sounds like the right response | TOGAF says raise to Architecture Board — the board decides whether to mandate conformance, approve dispensation, or accept risk |
