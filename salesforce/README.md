# Salesforce Engineering Patterns

A comprehensive reference of 15 production-ready patterns for architecting, developing, integrating, and operating Salesforce platforms — covering the full lifecycle from data model design through to DevOps, security, AI, and industry cloud deployments.

---

## Pattern Index

| # | Pattern | Key Topics |
|---|---------|-----------|
| 01 | [Platform Architecture & Data Model](01-platform-data-model.md) | Standard/custom objects, relationships, metadata API, schema design |
| 02 | [Apex Development & Governor Limits](02-apex-governor-limits.md) | Apex classes, triggers, bulkification, async patterns, governor limits |
| 03 | [SOQL & Data Access Patterns](03-soql-data-access.md) | SOQL, SOSL, relationship queries, aggregate queries, query optimisation |
| 04 | [Salesforce Security Model](04-security-model.md) | OWD, profiles, permission sets, sharing rules, FLS, field encryption |
| 05 | [Lightning Web Components (LWC)](05-lightning-web-components.md) | LWC framework, wire adapters, Lightning Data Service, component communication |
| 06 | [Flow & Declarative Automation](06-flow-automation.md) | Flow Builder, subflows, approval processes, scheduled flows, Flow testing |
| 07 | [Integration Patterns](07-integration-patterns.md) | REST/SOAP API, Connected Apps, OAuth, outbound messaging, named credentials |
| 08 | [Platform Events & Change Data Capture](08-platform-events-cdc.md) | Platform Events, CDC, event-driven architecture, Pub/Sub API, replay |
| 09 | [Salesforce DevOps & CI/CD](09-devops-cicd.md) | SFDX, scratch orgs, packaging, GitHub Actions pipelines, sandbox strategy |
| 10 | [Testing Strategies](10-testing-strategies.md) | Apex test classes, mock frameworks, LWC Jest, end-to-end testing, code coverage |
| 11 | [Bulk API & Large Data Volumes](11-bulk-api-large-data.md) | Bulk API 2.0, data archival, LDV design, chunking, Salesforce Data Loader |
| 12 | [CPQ & Revenue Cloud](12-cpq-revenue-cloud.md) | Salesforce CPQ, product catalogue, pricing rules, quote lifecycle, Revenue Cloud |
| 13 | [Einstein AI & CRM Analytics](13-einstein-crm-analytics.md) | Einstein Copilot, CRM Analytics, predictions, Agentforce, prompt templates |
| 14 | [Multi-Org Strategy & Architecture](14-multi-org-strategy.md) | Single-org vs multi-org, hub-and-spoke, Salesforce-to-Salesforce, org migration |
| 15 | [Industry Clouds & Financial Services](15-industry-clouds.md) | Financial Services Cloud, Health Cloud, data model extensions, action plans |
| 16 | [Design Patterns & Anti-Patterns](16-design-patterns-antipatterns.md) | 10 Apex/platform patterns, 10 anti-patterns, quick reference card |

---

## Decision Guide

### Automation — Code vs Clicks

| Requirement | Recommended Approach |
|-------------|---------------------|
| Simple field updates, branching logic | **Flow** |
| Cross-object complex logic, reuse, testability | **Apex** |
| Scheduled batch jobs | **Scheduled Flow** or **Apex Schedulable** |
| Complex UI interaction | **LWC + Apex controller** |
| Approval workflows | **Approval Process** or **Flow** |
| Real-time event-driven integration | **Platform Events** |

### Integration Pattern Selector

| Scenario | Pattern |
|---------|---------|
| External system reads Salesforce data | REST API (Connected App + OAuth) |
| Salesforce pushes changes to external | Platform Events / Outbound Messaging / Apex Callout |
| External pushes changes to Salesforce | REST API upsert / Bulk API 2.0 |
| Real-time CDC to data warehouse | Change Data Capture + Pub/Sub API |
| Enterprise middleware bus | MuleSoft / named credentials + external services |

### Deployment Approach Selector

| Team maturity | Org strategy | Recommended toolchain |
|--------------|-------------|----------------------|
| Solo / small | Single sandbox | SFDX + VS Code + change sets |
| Team, multiple environments | Scratch orgs | SFDX + 2GP unlocked packages + GitHub Actions |
| Multi-product platform | Multi-org | 2GP managed packages + hub org + DevOps Center |
| ISV / AppExchange | Packaging required | Managed packages + LMA + Partner Community |

---

## Categories

### Platform Foundations
- **Data Model** — Schema design, object relationships, metadata API, schema evolution
- **Security Model** — OWD, profiles, permission sets, sharing, field-level security

### Development
- **Apex** — Server-side logic, triggers, bulkification, governor limits, async patterns
- **SOQL** — Query language, relationship queries, performance optimisation
- **LWC** — Component-based UI, wire adapters, Lightning Data Service

### Automation
- **Flow** — Declarative automation, subflows, approval processes, testing

### Integration & Events
- **Integration Patterns** — REST/SOAP APIs, OAuth, named credentials, callouts
- **Platform Events & CDC** — Event-driven messaging, replay, Change Data Capture

### DevOps & Quality
- **DevOps & CI/CD** — SFDX, scratch orgs, unlocked packages, pipelines
- **Testing** — Apex tests, mocks, LWC Jest, coverage thresholds

### Data Operations
- **Bulk API & LDV** — Large data volume design, bulk load, archival

### Business Applications
- **CPQ & Revenue Cloud** — Quote lifecycle, pricing rules, revenue recognition
- **Einstein AI & Analytics** — Agentforce, CRM Analytics, predictions

### Architecture
- **Multi-Org Strategy** — Org topology, Salesforce-to-Salesforce, migrations
- **Industry Clouds** — Financial Services Cloud, Health Cloud, vertical data models
