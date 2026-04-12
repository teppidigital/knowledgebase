# Continuous Architecture in Practice

Key lessons from *Continuous Architecture in Practice* (Erder, Pureur, Woods — 2021).

A catalogue of 15 patterns and practices for keeping software architecture aligned with a continuously changing world — without the overhead of big-upfront design.

## Patterns

| # | Topic | Category | Key Concepts |
|---|-------|----------|--------------|
| 01 | [Six Principles of Continuous Architecture](./01-six-principles.md) | Foundations | Product-thinking, QA-first, delay decisions, small |
| 02 | [Quality Attributes](./02-quality-attributes.md) | Design | Utility tree, QA scenarios, tradeoff analysis |
| 03 | [Technical Debt](./03-technical-debt.md) | Sustainability | Debt taxonomy, debt register, paydown strategies |
| 04 | [The Architect's Role](./04-architect-role.md) | People | Enabler model, T-shaped, embedded vs consulting |
| 05 | [Architecture in Agile & DevOps](./05-architecture-agile.md) | Process | Architecture runway, spikes, just-in-time design |
| 06 | [Architecture Fitness Functions](./06-fitness-functions.md) | Governance | Automated/manual, ArchUnit, CI integration |
| 07 | [Architecture Decision Records](./07-adrs.md) | Documentation | MADR, Y-statements, decision workflow |
| 08 | [Modularity and Coupling](./08-modularity-coupling.md) | Design | Afferent/efferent coupling, cohesion, package principles |
| 09 | [Team Topologies & Conway's Law](./09-team-topologies.md) | Organisation | Stream-aligned, platform, enabling, inverse Conway |
| 10 | [Architecture Governance](./10-architecture-governance.md) | Governance | Lightweight review, architecture guild, radar |
| 11 | [Cloud-Native Architecture](./11-cloud-native.md) | Platform | 12-factor, microservices vs modular monolith, serverless |
| 12 | [Architecture Runway](./12-architecture-runway.md) | Process | Runway concept, enabler stories, IP iterations |
| 13 | [Evolutionary Architecture Patterns](./13-evolutionary-patterns.md) | Design | Strangler fig, anticorruption layer, expand-contract |
| 14 | [Architecture Documentation](./14-architecture-documentation.md) | Documentation | C4 model, arc42, lightweight diagrams |
| 15 | [Architecture Health Metrics](./15-architecture-metrics.md) | Measurement | DORA, coupling metrics, debt ratio, fitness results |

---

## Categories

### Foundations
- **Six Principles** — The 6 universal principles that define continuous architecture practice

### Design
- **Quality Attributes** — Identifying, prioritising, and testing the "ilities"
- **Modularity and Coupling** — Managing structure and dependencies for changeability
- **Evolutionary Patterns** — Patterns for changing a running production system safely

### Sustainability
- **Technical Debt** — Classifying, measuring, and reducing architectural debt

### People and Organisation
- **The Architect's Role** — How the architect's job evolves in agile, DevOps, and cloud organisations
- **Team Topologies & Conway's Law** — Aligning team structure with desired architecture

### Process
- **Architecture in Agile & DevOps** — Integrating architecture into sprints, pipelines, and delivery
- **Architecture Runway** — Staying ahead of the teams without big-upfront design

### Governance
- **Fitness Functions** — Automated and manual architecture verification
- **Architecture Governance** — Lightweight mechanisms for consistent architecture decisions

### Documentation
- **Architecture Decision Records** — Capturing and communicating decisions with minimal overhead
- **Architecture Documentation** — Right-sized documentation for living systems

### Platform
- **Cloud-Native Architecture** — Principles and patterns for modern cloud deployments

### Measurement
- **Architecture Health Metrics** — Quantifying architecture quality over time

---

## The Continuous Architecture Mental Model

```
Business Goals & Quality Attribute Priorities
          │
          ▼
Architecture Principles (6 rules that always apply)
          │
          ▼
Continuous Architecture Practice
  ├── Just-in-time design decisions (backed by ADRs)
  ├── Architecture runway (stay ahead of teams)
  ├── Fitness functions (automated governance)
  ├── Technical debt register (managed consciously)
  └── Lightweight documentation (C4 + ADRs)
          │
          ▼
Deployed, Observable, Evolvable Systems
```

## Source

Erder, M., Pureur, P., & Woods, E. (2021). *Continuous Architecture in Practice: Software Architecture in the Age of Agility and DevOps*. Addison-Wesley Professional.
