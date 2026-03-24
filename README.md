# Knowledge Portal

A comprehensive engineering knowledge base covering system design, distributed systems, DevOps, DevSecOps, security, and cloud-native patterns on AWS and Azure. Every entry shares a consistent structure — making it fast to compare, evaluate, and apply patterns in real projects.

---

## Document Format

Every pattern file follows the same structure:

| Section            | Contents                                          |
| ------------------ | ------------------------------------------------- |
| **Category**       | Domain tags (e.g., DevOps, Security, Cloud)       |
| **Context**        | Problem statement, comparison tables, when to use |
| **Pros**           | Advantages and benefits                           |
| **Cons**           | Trade-offs and failure modes                      |
| **Design Diagram** | Mermaid architecture or sequence diagram          |
| **Code Sample**    | Working TypeScript, YAML, HCL, Bash, or SQL       |

---

## Sections

### [System Design](system-design/README.md)

> 34 patterns · Architecture fundamentals through advanced distributed system design

Core architectural patterns for building scalable, resilient systems. Covers service decomposition, data management strategies, communication patterns, and cross-cutting concerns.

**Key topics:** Microservices, Monolith, Event-Driven Architecture, CQRS, Event Sourcing, Saga, API Gateway, Strangler Fig, BFF, Load Balancing, Circuit Breaker, Rate Limiting, Caching, Sharding, Replication, CAP Theorem, Service Mesh, Observability, Multi-tenancy, GraphQL, gRPC, Webhooks, Idempotency, Bulkhead, Retry & Backoff, and more.

---

### [Distributed Design Patterns](distributed-design-pattern/README.md)

> 19 patterns · Low-level distributed systems primitives

Deep-dive patterns for building correct distributed systems under partial failure, network partitions, and concurrent access.

**Key topics:** Consistent Hashing, Leader Election, Gossip Protocol, Quorum, Distributed Locking, Service Discovery, Write-Ahead Log, Raft Consensus, Vector Clocks, CAP/BASE, Two-Phase Commit, CRDT, Bloom Filter, Time-Series Storage, Log Compaction, Merkle Trees, Backpressure, Distributed Tracing, Conflict-Free Data Types.

---

### [DevOps](devops/README.md)

> 15 patterns · Full software delivery lifecycle and operational excellence

Operational patterns for building high-performing engineering teams — from CI/CD mechanics through to DORA metric measurement.

**Key topics:** CI/CD Pipeline Design, GitOps (ArgoCD/Flux), Infrastructure as Code (Terraform), Deployment Strategies (Blue-Green/Canary/Rolling), Feature Flags (OpenFeature), Observability & OpenTelemetry, SRE & SLOs, Chaos Engineering (LitmusChaos), Database DevOps (Flyway/Atlas), Alerting & On-Call (Alertmanager/PagerDuty), Platform Engineering (Backstage/Crossplane), Artifact Management (Cosign/SBOM), Configuration Management (Helm/Ansible/ESO), Disaster Recovery (Velero/cross-region), DORA Metrics.

---

### [DevSecOps](devsecops/README.md)

> 15 patterns · Security controls embedded in the delivery pipeline

Security practices integrated into CI/CD pipelines from the first commit through to production runtime — "shift left" security culture.

**Key topics:** Shift-Left Security, SAST, SCA & Dependency Scanning, DAST, Secret Management, IaC Security, Container Security, Supply Chain & SBOM, Zero Trust, Policy-as-Code, Runtime Security, Threat Modelling, Compliance-as-Code, Security in CI/CD Pipelines, Vulnerability Management.

---

### [Security](security/README.md)

> 15 patterns · Application and infrastructure security patterns

Application-level security patterns for building systems that are secure by design, covering identity, data protection, network security, and compliance.

**Key topics:** OAuth 2.0 / OIDC / JWT, API Security, Encryption Patterns, Network Security, Identity Federation & SSO, Data Privacy & PII, Security Logging & SIEM, DDoS & Rate Limiting, Web App Security (OWASP Top 10), Secrets Management, mTLS, Zero Trust Architecture, Secure SDLC, Audit Trails, Incident Response.

---

### [Cloud Native — AWS](cloud-native/aws/README.md)

> 15 patterns · AWS-specific cloud-native architecture

Patterns for building production-grade workloads on AWS using managed services, with Infrastructure-as-Code examples (Terraform/CDK) throughout.

**Key topics:** Serverless (Lambda), ECS Fargate, EKS Kubernetes, VPC Networking, IAM Least Privilege, API Gateway, RDS/Aurora, DynamoDB, SQS/SNS/EventBridge, S3 Storage, CloudFront CDN, ElastiCache, Step Functions, AWS WAF, Cost Optimisation.

---

### [Cloud Native — Azure](cloud-native/azure/README.md)

> 15 patterns · Azure-specific cloud-native architecture

Patterns for building production-grade workloads on Azure using managed services, with Bicep and Terraform examples.

**Key topics:** Azure Functions, Container Apps, AKS Kubernetes, VNet Networking, Microsoft Entra ID & RBAC, API Management (APIM), Azure SQL & Cosmos DB, Service Bus & Event Grid, Blob Storage, Azure Monitor, Azure Front Door, Redis Cache, Logic Apps, Azure Firewall, FinOps & Cost Management.

---

### [Data Solutions](data-solutions/README.md)

> 15 patterns · Data engineering, analytics, governance, privacy, and AI data infrastructure

End-to-end data architecture patterns from raw ingestion through to real-time analytics, machine learning feature stores, data governance, and organisational models (Data Mesh).

**Key topics:** Batch Ingestion & ETL (Airflow/dbt), Change Data Capture (Debezium/Kafka), Real-time Stream Processing (Flink/Kafka Streams), Data Lakehouse (Iceberg/Delta), Data Modelling (Star Schema/SCD2), Search & Full-Text (Elasticsearch/OpenSearch), Caching (Redis/Cache-Aside), Time-Series (TimescaleDB), Graph Databases (Neo4j), OLAP Engines (ClickHouse/Trino), ML Feature Store (Feast), Data Governance & Catalogue (DataHub), Data Privacy & GDPR Compliance, Data Mesh, Real-time Analytics & Reverse ETL.

---

### [AI & LLM Integration](ai-llm/README.md)

> 15 patterns · Production-ready patterns for building reliable, safe, and cost-efficient LLM systems

Patterns for integrating large language models into production applications — from retrieval-augmented generation and agent architectures through to cost control, safety guardrails, and automated quality evaluation.

**Key topics:** Retrieval-Augmented Generation (RAG), LLM Gateway & Provider Abstraction, Prompt Engineering & Management, AI Agents & Tool Use (ReAct/LangGraph), Vector Databases (pgvector/Pinecone/Weaviate), Embedding Pipelines, Semantic Caching, AI Observability (LangSmith/Phoenix), Guardrails & Content Safety, Fine-tuning vs RAG, Multimodal Pipelines (vision/audio), AI Cost Optimisation (model routing/Batch API), LLM Evaluation & Evals (RAGAS), Function Calling & Structured Outputs, AI-Assisted Code Generation.

---

### [API Design & Integration](api-design/README.md)

> 15 patterns · REST, GraphQL, gRPC, AsyncAPI, webhooks, and API lifecycle management

Contract-first API design patterns and integration techniques for building APIs that are reliable, discoverable, versioned, and monetisable — with a focus on developer experience and operational lifecycle.

**Key topics:** REST API Design (OpenAPI 3.1, RFC 7807, ETags), GraphQL Schema Design (DataLoader, Relay pagination), gRPC & Protobuf (streaming, auth interceptor), AsyncAPI & Event-Driven (Kafka, DLQ), API Versioning, Consumer-Driven Contracts (Pact V3), Webhook Design & Reliability (HMAC-SHA256, retry), API Gateway (Kong, AWS API Gateway), SDK Generation (openapi-generator, CI publish), Rate Limiting (token bucket, sliding window, Redis), API Monetisation (API keys, usage plans, Stripe), Hypermedia & HATEOAS (HAL, JSON:API), API Deprecation (RFC 8594, Sunset header), API Mocking (MSW, WireMock, Prism), GraphQL Federation (Apollo Federation v2, Router).

---

### [Frontend Architecture](frontend/README.md)

> 15 patterns · Component composition, state, performance, security, and delivery for modern React applications

Frontend architecture patterns for building production-grade React applications — covering micro-frontends, state management, accessibility, internationalisation, Progressive Web Apps, and the complete delivery pipeline from bundling to CDN edge delivery.

**Key topics:** Micro-Frontends (Vite Module Federation, host/remote, event bus), State Management (TanStack Query v5, Zustand, React Hook Form + Zod), Performance Optimisation (Core Web Vitals, code splitting, virtual lists, Web Workers, bundle budgets), Design System (Radix UI, Style Dictionary, Storybook), Server-Side Rendering (Next.js App Router, RSC, ISR, streaming Suspense), Progressive Web App (Workbox caching strategies, Push API, VAPID), Frontend Testing (Vitest, Testing Library, MSW, Playwright), OIDC Authentication (PKCE, oidc-client-ts, in-memory tokens), Feature Flags (OpenFeature, LaunchDarkly), Error Handling (ErrorBoundary, Sentry PII scrubbing), i18n & Localisation (i18next ICU, RTL, Intl.*), Accessibility (WCAG 2.1 AA, ARIA, axe-core, focus management), Bundling & Build Tools (Vite + SWC, manualChunks, CI budget), CDN & Edge Delivery (Cloudflare Workers, Next.js Middleware, CloudFront Terraform), Frontend Security (CSP nonces, Trusted Types, open redirect prevention, SBOM + Grype).

---

## At a Glance

| Section                                                             | Patterns | Primary Languages           |
| ------------------------------------------------------------------- | -------- | --------------------------- |
| [System Design](system-design/README.md)                            | 34       | TypeScript, YAML                   |
| [Distributed Design Patterns](distributed-design-pattern/README.md) | 19       | TypeScript, Go, YAML               |
| [DevOps](devops/README.md)                                          | 15       | TypeScript, YAML, HCL, Bash        |
| [DevSecOps](devsecops/README.md)                                    | 15       | TypeScript, YAML, Bash             |
| [Security](security/README.md)                                      | 15       | TypeScript, YAML                   |
| [Cloud Native — AWS](cloud-native/aws/README.md)                    | 15       | TypeScript, HCL, Bash              |
| [Cloud Native — Azure](cloud-native/azure/README.md)                | 15       | TypeScript, Bicep, HCL             |
| [Data Solutions](data-solutions/README.md)                          | 15       | Python, TypeScript, SQL, YAML      |
| [AI & LLM Integration](ai-llm/README.md)                           | 15       | TypeScript, Python, YAML           |
| [API Design & Integration](api-design/README.md)                    | 15       | TypeScript, YAML, Protobuf         |
| [Frontend Architecture](frontend/README.md)                         | 15       | TypeScript, YAML, HCL              |
| [Backend Architecture](backend/README.md)                           | 15       | TypeScript, Python, YAML, HCL      |
| [FinOps — Cloud Cost Discipline](finops/README.md)                  | 15       | Python, HCL, YAML, SQL             |
| **Total**                                                           | **218**  |                                    |

---

## How to Use This Portal

**Exploring a domain** — Browse the section README for a categorised index and tool ecosystem overview.

**Finding a specific pattern** — Each pattern file is numbered and named descriptively (e.g., `04-deployment-strategies.md`). Use your editor's file search or the section README tables.

**Evaluating trade-offs** — Every entry includes explicit Pros and Cons sections for quick decision support.

**Applying to a project** — Code samples are production-oriented — they use real libraries, real YAML structure, and avoid placeholder anti-patterns (no hardcoded credentials, no `any` types where avoidable, OIDC over API keys).

---

## Related Sections Cross-Reference

| If you need…                   | See…                                                                                         |
| ------------------------------ | -------------------------------------------------------------------------------------------- |
| Build and deploy automation    | [DevOps — CI/CD & GitOps](devops/README.md)                                                  |
| Security controls in pipelines | [DevSecOps](devsecops/README.md)                                                             |
| Application-level security     | [Security](security/README.md)                                                               |
| Splitting a monolith           | [System Design — Strangler Fig](system-design/README.md)                                     |
| Handling distributed failures  | [Distributed Patterns — Circuit Breaker, Raft, Quorum](distributed-design-pattern/README.md) |
| AWS infrastructure             | [Cloud Native — AWS](cloud-native/aws/README.md)                                             |
| Azure infrastructure           | [Cloud Native — Azure](cloud-native/azure/README.md)                                         |
| Measuring team performance     | [DevOps — DORA Metrics](devops/15-devops-metrics.md)                                         |
| Zero-downtime schema changes   | [DevOps — Database DevOps](devops/09-database-devops.md)                                     |
| Incident response              | [DevOps — Alerting & On-Call](devops/10-alerting-oncall.md)                                  |
| Data pipelines & ETL           | [Data Solutions — Batch Ingestion](data-solutions/01-batch-ingestion-etl.md)                 |
| Real-time event processing     | [Data Solutions — Stream Processing](data-solutions/03-stream-processing.md)                  |
| Analytical queries at scale    | [Data Solutions — OLAP Engines](data-solutions/10-olap-query-engines.md)                     |
| GDPR right-to-erasure          | [Data Solutions — Privacy & Compliance](data-solutions/13-data-privacy-compliance.md)         |
| ML features for real-time AI   | [Data Solutions — Feature Store](data-solutions/11-feature-store-ml.md)                      |
| Scaling data across many teams | [Data Solutions — Data Mesh](data-solutions/14-data-mesh.md)                                 |
| RAG / LLM in production        | [AI & LLM — RAG + Guardrails](ai-llm/README.md)                                              |
| LLM cost control               | [AI & LLM — Cost Optimisation](ai-llm/12-ai-cost-optimisation.md)                            |
| LLM quality regression testing | [AI & LLM — Evals](ai-llm/13-llm-evaluation.md)                                              |
| AI agent / tool use            | [AI & LLM — Agents](ai-llm/04-ai-agents-tool-use.md)                                         |
| Structured LLM outputs         | [AI & LLM — Function Calling](ai-llm/14-function-calling-structured-outputs.md)               |
