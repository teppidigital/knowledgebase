# AWS Certified DevOps Engineer – Professional (DOP-C02)
## Expert Learning Path

---

## Exam at a Glance

| Property | Detail |
|----------|--------|
| Exam code | DOP-C02 |
| Duration | 180 minutes |
| Questions | 75 (multiple choice + multiple response) |
| Cost | USD 300 |
| Passing score | 750 / 1000 |
| Validity | 3 years |
| Recommended experience | 2+ years hands-on DevOps on AWS |

### Exam Domains and Weighting

| Domain | Title | Weight |
|--------|-------|--------|
| 1 | SDLC Automation | 22% |
| 2 | Configuration Management and IaC | 17% |
| 3 | Resilient Cloud Solutions | 15% |
| 4 | Monitoring and Logging | 15% |
| 5 | Incident and Event Response | 14% |
| 6 | Security and Compliance | 17% |

> **Total study time estimate:** 12–16 weeks at 8–10 hours/week. Faster if you hold SAA-C03 or SAP-C02.

---

## How DOP-C02 Differs from SAP-C02

If you have already completed the SAP-C02 path, note the key differences:

| Dimension | SAP-C02 | DOP-C02 |
|-----------|---------|---------|
| Breadth vs Depth | Very broad — all AWS domains | Deep in DevOps toolchain and automation |
| Architecture focus | Multi-account design, migration | Pipeline automation, IaC, release engineering |
| Audience | Solutions architects | DevOps engineers, platform/SRE engineers |
| Key AWS services | Organizations, Direct Connect, DMS | CodePipeline, CodeBuild, CloudFormation, Systems Manager |
| Scenario type | Design trade-offs | Automate this, detect this, respond to this |

**Shared foundation:** IAM, CloudWatch, CloudTrail, VPC, EC2, Lambda, ECS, EKS, RDS — all carry over. Do not restudy these from scratch.

---

## Prerequisites Self-Assessment

| Area | Topics | Minimum Level |
|------|--------|---------------|
| CI/CD concepts | Pipeline stages, artifact management, gating | Proficient |
| Infrastructure as Code | CloudFormation or Terraform basics | Intermediate |
| Linux / shell scripting | bash, systemd, cron | Intermediate |
| Containers | Docker build, push, run; ECS/EKS basics | Intermediate |
| AWS core | IAM, VPC, EC2, Lambda, S3, RDS | Solid |
| Git | Branching, PRs, hooks, tagging | Proficient |

---

## Phase Overview

```
Phase 0 (Week 1)        → Baseline and orientation
Phase 1 (Weeks 2–4)     → Domain 1: SDLC Automation
Phase 2 (Weeks 5–6)     → Domain 2: Configuration Management and IaC
Phase 3 (Weeks 7–8)     → Domain 3: Resilient Cloud Solutions
Phase 4 (Weeks 9–10)    → Domain 4: Monitoring and Logging
Phase 5 (Weeks 11–12)   → Domain 5: Incident and Event Response
Phase 6 (Weeks 13–14)   → Domain 6: Security and Compliance
Phase 7 (Weeks 15–16)   → Mock exams, gap remediation, exam-day prep
```

---

## Phase 0 — Baseline and Orientation (Week 1)

### Goal
Understand the exam's competency model, identify your starting gaps, and build your study system.

### Actions

- [ ] Download and read the official [DOP-C02 Exam Guide](https://aws.amazon.com/certification/certified-devops-engineer-professional/) fully — one sitting.
- [ ] Take the **AWS Certification Official Practice Question Set** (free on AWS Skill Builder). Record your score per domain.
- [ ] Map your score to the domain weightings table above — lowest % score relative to its weight = highest-priority gap.
- [ ] Book your exam now (12–14 weeks from today). Commitment creates accountability.
- [ ] Open your study log: date, topic, time spent, practice score.
- [ ] Confirm your hands-on environment: AWS account with billing alerts, a git repo you actively use, and a working CodePipeline or equivalent pipeline you can experiment with.

### Validation
You can name all six domains and explain in one sentence what each tests. You know which domain is your weakest.

---

## Phase 1 — Domain 1: SDLC Automation (Weeks 2–4)

**Exam weight: 22% — approximately 16–17 questions**

The most AWS-toolchain-specific domain. Covers the full CI/CD pipeline on AWS: source, build, test, deploy, and release automation. Expect heavy CodeSuite questions.

### Week 2 — Source Control and Build Automation

**Core services:** CodeCommit, CodeBuild, CodeArtifact, Amazon ECR, S3 (as artifact store)

**Topics to master:**

- [ ] **CodeBuild:** `buildspec.yml` structure (phases: install, pre_build, build, post_build), environment variables, build caching (S3 and local), Docker-in-Docker builds, VPC builds for private resources
- [ ] **CodeBuild reports:** test reports (JUnit XML), coverage reports — how to publish and gate on them
- [ ] **CodeArtifact:** repositories, domains, upstream repository chaining (npm, Maven, PyPI, NuGet), cross-account package sharing, pull-through cache for public repos
- [ ] **ECR:** image scanning (Basic vs Inspector v2 Enhanced), lifecycle policies, replication (cross-region, cross-account), immutable tags, ECR pull-through cache
- [ ] **CodeCommit:** triggers (SNS/Lambda on push), approval rule templates, notification rules, cross-account access via IAM roles
- [ ] **S3 as artifact store:** versioning, encryption (SSE-S3, SSE-KMS), lifecycle policies for old build artifacts
- [ ] Build badge, build notifications, build timeout strategies

**Knowledge portal cross-reference:**
- [`devops/01-cicd-pipeline-design.md`](../devops/01-cicd-pipeline-design.md) — full read
- [`devops/12-artifact-release-management.md`](../devops/12-artifact-release-management.md) — full read
- [`devsecops/14-security-cicd-pipeline.md`](../devsecops/14-security-cicd-pipeline.md)

**Hands-on lab:** Build a CodeBuild project that:
1. Builds a Docker image
2. Runs unit tests and publishes a JUnit test report to CodeBuild
3. Pushes the image to ECR with a commit SHA tag
4. Fails the build if test coverage < 80%

### Week 3 — Deployment Automation

**Core services:** CodeDeploy, CodePipeline, Elastic Beanstalk, AppConfig, EC2 Image Builder

**Topics to master:**

- [ ] **CodeDeploy:** EC2/on-premises (in-place vs blue/green), ECS (blue/green), Lambda (canary, linear, all-at-once), `appspec.yml` lifecycle hooks (BeforeInstall, AfterInstall, ApplicationStart, ValidateService)
- [ ] **CodeDeploy rollback:** automatic vs manual, CloudWatch alarm triggers, rollback vs redeploy distinction
- [ ] **CodePipeline:** stages, actions, action types (Source, Build, Test, Deploy, Approval, Invoke), parallel vs sequential actions, manual approval + SNS notification, cross-account pipeline
- [ ] **CodePipeline triggers:** EventBridge rules for CodeCommit/S3/ECR events — how pipelines detect source changes
- [ ] **Elastic Beanstalk:** deployment modes (all-at-once, rolling, rolling with additional batch, immutable, blue/green), `.ebextensions`, platform hooks, managed updates, worker tier
- [ ] **AppConfig:** configuration profiles, deployment strategies (linear, exponential, all-at-once), validators (JSON Schema + Lambda), rollback on CloudWatch alarm
- [ ] **EC2 Image Builder:** pipelines, recipes, components, image testing, distribution settings, cross-region AMI copy

**Knowledge portal cross-reference:**
- [`devops/04-deployment-strategies.md`](../devops/04-deployment-strategies.md) — full read; map every strategy to a CodeDeploy mode
- [`devops/05-feature-flags.md`](../devops/05-feature-flags.md) — AppConfig as a feature flag store
- [`devops/02-gitops.md`](../devops/02-gitops.md)

**Hands-on lab:** Build a CodePipeline that:
1. Triggers on CodeCommit push to `main`
2. CodeBuild: build + test
3. Manual approval action with SNS notification
4. CodeDeploy: blue/green to ECS with 10% canary weight, automatic rollback on 5xx alarm

### Week 4 — Testing Automation and Release Gates

**Core services:** CodeBuild (test reports), Device Farm, Lambda (synthetic tests), CloudWatch Alarms, Step Functions

**Topics to master:**

- [ ] Test types in a pipeline: unit (CodeBuild), integration (CodeBuild + test environment), load (Artillery/Gatling in CodeBuild), synthetic (CloudWatch Synthetics canaries)
- [ ] **CloudWatch Synthetics:** Canary scripts (Puppeteer/Node.js), canary scheduling, canary groups, success/failure metrics
- [ ] Gating on test results: CodeBuild report groups, Lambda Invoke action in CodePipeline as a test gate
- [ ] **Step Functions as orchestration:** Express vs Standard workflows in CI/CD context, fan-out test parallelism using Map state
- [ ] **Pipeline notifications and audit:** CodeStar Notifications, CloudTrail logging of pipeline events, pipeline execution history
- [ ] Semantic versioning in automated pipelines: tagging strategies, `buildId` vs `commitSha` vs `semver`
- [ ] **Trunk-based development vs GitFlow** — and how each affects pipeline design decisions

**Knowledge portal cross-reference:**
- [`devops/15-devops-metrics.md`](../devops/15-devops-metrics.md) — DORA metrics; these are exam scenarios
- [`devops/05-feature-flags.md`](../devops/05-feature-flags.md)

**Phase 1 Deliverable:** A fully documented pipeline design (draw.io, Mermaid, or written) for a microservices release. Include: source trigger, build, unit test gate, integration test gate, manual approval, blue/green deploy to ECS, and automatic rollback conditions. Justify every decision.

**Domain 1 Validation Questions:**
- What are CodeBuild's three caching options, and when would you choose each?
- A CodeDeploy deployment to Lambda must shift 10% of traffic immediately, then 100% after 30 minutes if no alarm fires — which deployment configuration type is this?
- How does CodePipeline detect a change in an ECR image and trigger a new execution?
- What is the difference between a CodeDeploy rollback and a CodeDeploy redeploy?
- How do you share packages from a CodeArtifact repository across two AWS accounts?

---

## Phase 2 — Domain 2: Configuration Management and IaC (Weeks 5–6)

**Exam weight: 17% — approximately 12–13 questions**

Deep AWS-native IaC. This domain is almost entirely CloudFormation + Systems Manager. Terraform knowledge is useful for understanding concepts but CloudFormation is the exam focus.

### Week 5 — CloudFormation Mastery

**Core services:** CloudFormation, CloudFormation StackSets, Service Catalog, CDK

**Topics to master:**

- [ ] **Template anatomy:** Parameters, Mappings, Conditions, Resources, Outputs, Metadata, Transform
- [ ] **Intrinsic functions:** `Fn::If`, `Fn::Select`, `Fn::FindInMap`, `Fn::Sub`, `Fn::GetAtt`, `Fn::ImportValue`, `!Ref`
- [ ] **Change Sets:** preview changes before executing, change set types (CREATE, UPDATE, IMPORT)
- [ ] **Stack policies:** prevent accidental updates to specific resources (e.g., RDS, EC2)
- [ ] **Drift detection:** when drift happens, what it detects, how to remediate
- [ ] **Nested stacks vs StackSets vs Stack References:** know which to use for each scenario
- [ ] **StackSets:** deployment targets (OU-based vs account-based), failure tolerance, concurrent accounts, `SELF_MANAGED` vs `SERVICE_MANAGED` mode
- [ ] **Custom Resources:** Lambda-backed, `cfn-response` module, creation policy, update policy
- [ ] **cfn-init and cfn-signal:** EC2 instance bootstrapping, `CreationPolicy` with `WaitCondition`
- [ ] **CloudFormation Hooks:** pre/post resource creation validation (proactive compliance)
- [ ] **Resource imports:** importing existing resources into CloudFormation management
- [ ] **AWS CDK:** construct levels (L1/L2/L3), `cdk synth`, `cdk deploy`, `cdk diff` — and how CDK output maps to CloudFormation

**Knowledge portal cross-reference:**
- [`devops/03-infrastructure-as-code.md`](../devops/03-infrastructure-as-code.md) — full read
- [`devsecops/06-iac-security.md`](../devsecops/06-iac-security.md) — CloudFormation Guard, cfn-nag
- [`devsecops/10-policy-as-code.md`](../devsecops/10-policy-as-code.md)

**Hands-on lab:** Write a CloudFormation template that:
1. Uses nested stacks (VPC stack, ECS stack, RDS stack)
2. Deploys across 3 accounts using StackSets with `SERVICE_MANAGED` mode
3. Includes a custom resource that registers the deployment in an external system
4. Has a stack policy preventing deletion of the RDS instance

### Week 6 — Systems Manager and Configuration Management

**Core services:** Systems Manager (SSM), OpsCenter, Patch Manager, Parameter Store, Session Manager, Run Command, State Manager, Automation, Inventory, Fleet Manager

**Topics to master:**

- [ ] **SSM Agent:** installation on EC2 and on-premises, managed instance registration, hybrid activations
- [ ] **Session Manager:** replace SSH/RDP (no open ports, no bastion), session logging to S3/CloudWatch, port forwarding, VPC endpoint for private instances
- [ ] **Run Command:** `AWS-RunShellScript`, output to S3, rate controls (concurrency + error threshold), targets (tags, resource groups, instance IDs)
- [ ] **Patch Manager:** patch baselines (custom vs AWS-managed), patch groups (tag-based), maintenance windows, compliance reporting, `AWS-RunPatchBaseline` document
- [ ] **State Manager:** association schedules, `AWS-ApplyAnsiblePlaybooks`, `AWS-ConfigureCloudWatch`
- [ ] **SSM Automation:** documents (runbooks), `aws:executeScript`, `aws:invokeLambdaFunction`, `aws:waitForAwsResourceProperty`, cross-account automation
- [ ] **Parameter Store:** Standard vs Advanced tiers, SecureString (KMS), hierarchies, `GetParametersByPath`, change notification via EventBridge
- [ ] **OpsCenter:** operational work items (OpsItems), integrations with Security Hub, Config, CloudWatch Alarms
- [ ] **Inventory:** OS/application inventory, resource data sync to S3, Athena queries on inventory
- [ ] **SSM Documents:** Command vs Automation vs Session vs Policy document types

**Knowledge portal cross-reference:**
- [`devops/13-configuration-management.md`](../devops/13-configuration-management.md) — full read
- [`devsecops/05-secret-management.md`](../devsecops/05-secret-management.md) — Parameter Store vs Secrets Manager decision

**Phase 2 Deliverable:** Design a "Day 2 operations" playbook using SSM Automation that: detects an instance with patch compliance < 100%, opens an OpsItem, applies missing patches via Patch Manager in a maintenance window, and closes the OpsItem. Document as an SSM Automation runbook outline.

**Domain 2 Validation Questions:**
- What is the difference between a CloudFormation nested stack and a StackSet?
- A CloudFormation update would delete and re-create an RDS instance — how do you prevent this accidental action?
- When would you use SSM Parameter Store over Secrets Manager?
- How does `cfn-signal` interact with `CreationPolicy`?
- What is the difference between SSM State Manager and SSM Run Command?

---

## Phase 3 — Domain 3: Resilient Cloud Solutions (Weeks 7–8)

**Exam weight: 15% — approximately 11–12 questions**

Significant overlap with SAP-C02 Domain 2. Focus here on the *automation* of resilience — Auto Scaling policies, self-healing architectures, and pipeline-driven DR — rather than static design.

### Week 7 — Auto Scaling and Self-Healing

**Core services:** EC2 Auto Scaling, Application Auto Scaling, ECS Auto Scaling, EKS (Karpenter/Cluster Autoscaler), Lambda (concurrency), ElastiCache Scaling

**Topics to master:**

- [ ] **EC2 Auto Scaling:** scaling policies (target tracking, step, scheduled), warmup periods, lifecycle hooks (pending:wait, terminating:wait), predictive scaling
- [ ] **Warm pools:** pre-initialised instances that reduce scale-out latency — when to use and cost implications
- [ ] **Launch Templates vs Launch Configurations:** why Launch Templates are required for modern features (Spot mixed instances, multiple instance types)
- [ ] **Mixed instances policy:** Spot + On-Demand blend, `SpotAllocationStrategy` (capacity-optimised, price-capacity-optimised, diversified)
- [ ] **Application Auto Scaling:** DynamoDB (RCU/WCU), Aurora replicas, ECS service tasks, Lambda provisioned concurrency — target tracking vs step policies
- [ ] **Health check types:** EC2 (status check) vs ELB (application-level) — impact on replacement behaviour
- [ ] **Instance refresh:** rolling replacement with min healthy percentage, skip matching instances (hash-based)
- [ ] **ECS Service Auto Scaling:** task-count target tracking on CPU/memory/SQS queue depth (custom ALB metric)
- [ ] **Self-healing patterns:** SSM Automation triggered by CloudWatch Alarm → auto-remediate → notify via SNS

**Knowledge portal cross-reference:**
- [`cloud/aws/02-ecs-fargate.md`](../cloud/aws/02-ecs-fargate.md) — auto scaling section
- [`cloud/aws/03-eks-kubernetes.md`](../cloud/aws/03-eks-kubernetes.md) — Karpenter section
- [`production-hardening/`](../production-hardening/)

### Week 8 — High Availability and Disaster Recovery Automation

**Core services:** Route 53, Aurora Global Database, DynamoDB Global Tables, S3, Backup, Elastic Disaster Recovery (DRS), CloudFormation

**Topics to master:**

- [ ] **Route 53 health checks and failover:** primary/secondary routing, health check types (HTTP, HTTPS, TCP, calculated), string matching, latency-based with failover
- [ ] **Aurora Global Database:** automated vs manual promotion, write forwarding, replication lag monitoring (`AuroraGlobalDBReplicationLag` metric)
- [ ] **AWS Backup:** backup plans, backup vaults, cross-region copy, cross-account copy, Backup Audit Manager
- [ ] **Elastic Disaster Recovery (DRS):** agent-based replication, continuous replication, point-in-time recovery, launch templates
- [ ] **RTO/RPO automation:** automating failover decisions using CloudWatch + EventBridge + Step Functions (human-approval gate for production)
- [ ] **Multi-AZ design for ECS:** task placement constraints and strategies (spread across AZ), ALB cross-zone load balancing
- [ ] **SQS as a resilience buffer:** decoupling components so downstream failure doesn't propagate upstream
- [ ] **Pipeline-driven DR testing:** running DR drills via CodePipeline + Lambda (spinning up recovery environment, validating, tearing down)

**Knowledge portal cross-reference:**
- [`cloud/aws/14-disaster-recovery.md`](../cloud/aws/14-disaster-recovery.md) — full read
- [`devops/14-disaster-recovery.md`](../devops/14-disaster-recovery.md) — full read
- [`devops/08-chaos-engineering.md`](../devops/08-chaos-engineering.md) — Fault Injection Service (FIS)

**Phase 3 Deliverable:** Design a self-healing architecture for a three-tier web application: a CloudWatch Alarm on 5xx rate triggers an SSM Automation runbook that (1) checks ALB targets, (2) replaces unhealthy instances via Auto Scaling, (3) runs a synthetic canary to validate recovery, and (4) notifies via SNS. Document the event flow end-to-end.

**Domain 3 Validation Questions:**
- What is the difference between EC2 Auto Scaling target tracking and step scaling? When does each outperform the other?
- How does Instance Refresh work, and how does it differ from a rolling deployment in CodeDeploy?
- A Route 53 health check is failing even though the endpoint responds to curl — what are three possible causes?
- What is a warm pool and what specific workload characteristic justifies its cost?
- How do you automate Aurora Global Database failover without human intervention, and what are the risks?

---

## Phase 4 — Domain 4: Monitoring and Logging (Weeks 9–10)

**Exam weight: 15% — approximately 11–12 questions**

This domain is heavily operational. Expect scenarios requiring you to choose the right CloudWatch feature, aggregate logs at scale, or build metric-driven auto-remediation. Observability-as-code is a key differentiator.

### Week 9 — CloudWatch Deep Dive

**Core services:** CloudWatch Metrics, Alarms, Logs, Log Insights, Dashboards, Synthetics, Evidently, Contributor Insights, Metric Streams, Embedded Metrics Format

**Topics to master:**

- [ ] **Metric namespaces and dimensions:** standard vs custom metrics, `PutMetricData` IAM permission, metric math
- [ ] **CloudWatch Alarms:** single metric, composite alarms, anomaly detection alarms — `ALARM`, `OK`, `INSUFFICIENT_DATA` states; alarm actions (Auto Scaling, SNS, EC2)
- [ ] **CloudWatch Logs:** log groups, log streams, retention policies, metric filters (extract custom metrics from logs), subscription filters (Lambda, Kinesis Firehose, Kinesis Streams)
- [ ] **CloudWatch Logs Insights:** query syntax (`fields`, `filter`, `stats`, `sort`, `limit`), cross-log-group queries, saved queries
- [ ] **Contributor Insights:** top-N contributors to a metric (e.g., which IP sends the most requests), uses log data — built-in rules vs custom rules
- [ ] **CloudWatch Synthetics:** canary scripts, canary run grouping, CloudWatch Alarm integration, Visual Monitoring (screenshot comparison)
- [ ] **Evidently:** feature flags + A/B experiment metrics (launches vs experiments), overrides
- [ ] **Embedded Metrics Format (EMF):** structured JSON logs that CloudWatch auto-extracts as metrics — key for Lambda observability at scale
- [ ] **Metric Streams:** near-real-time metric delivery to Kinesis Firehose → S3/Datadog/New Relic, filter by namespace
- [ ] **CloudWatch dashboards:** cross-account, cross-region dashboards; shared dashboards

**Knowledge portal cross-reference:**
- [`cloud/aws/11-observability-cloudwatch.md`](../cloud/aws/11-observability-cloudwatch.md) — full read
- [`observability/01-opentelemetry-instrumentation.md`](../observability/01-opentelemetry-instrumentation.md)
- [`observability/03-metrics-prometheus.md`](../observability/03-metrics-prometheus.md)
- [`observability/04-structured-logging.md`](../observability/04-structured-logging.md)
- [`observability/06-alerting-slo-burn-rate.md`](../observability/06-alerting-slo-burn-rate.md)

**Hands-on lab:** Build a CloudWatch-based observability stack for a Lambda function:
1. Use EMF to emit custom business metrics from Lambda code
2. Create a CloudWatch Alarm on the custom metric with anomaly detection
3. Build a Logs Insights query that counts error types per 5-minute window
4. Create a Contributor Insights rule on the ALB access logs to find the top-10 slowest paths

### Week 10 — Centralised Logging, Tracing, and Audit

**Core services:** CloudTrail, AWS Config, X-Ray, ADOT (AWS Distro for OpenTelemetry), Kinesis Firehose, OpenSearch Service, S3

**Topics to master:**

- [ ] **CloudTrail:** management events vs data events vs Insights events, organisation trail, log file integrity validation, CloudTrail Lake (SQL-based event analysis)
- [ ] **CloudTrail Insights:** unusual API activity detection (e.g., spike in `RunInstances`), how to query Insights events
- [ ] **AWS Config:** configuration items, configuration history, configuration snapshots, Config Rules (managed vs custom via Lambda/Guard), remediation actions (automatic vs manual), conformance packs
- [ ] **Config Aggregator:** cross-account and cross-region compliance view, aggregate dashboard
- [ ] **X-Ray:** sampling rules, groups, annotations vs metadata, subsegments, service map, X-Ray Analytics — Active tracing vs passive tracing
- [ ] **ADOT:** collector pipeline (receivers, processors, exporters), replacing X-Ray SDK with OTEL SDK, structured trace export to X-Ray + CloudWatch
- [ ] **Centralised log aggregation:** multi-account log → Kinesis Data Firehose → S3 (security account) pattern; cross-account subscription filters
- [ ] **Log encryption:** CloudWatch Logs encryption with KMS CMK, CloudTrail log encryption
- [ ] **Athena on CloudTrail/Config:** querying historical events using S3 + Athena, partition projection for performance

**Knowledge portal cross-reference:**
- [`observability/02-distributed-tracing.md`](../observability/02-distributed-tracing.md) — full read
- [`observability/12-log-correlation-tracing.md`](../observability/12-log-correlation-tracing.md)
- [`observability/13-otel-collector-pipeline.md`](../observability/13-otel-collector-pipeline.md)
- [`devops/06-observability-opentelemetry.md`](../devops/06-observability-opentelemetry.md)

**Phase 4 Deliverable:** Design a centralised observability architecture for a 5-account AWS Organisation:
- Logs: per-account CloudWatch → subscription filter → cross-account Kinesis Firehose → S3 in a dedicated logging account
- Metrics: CloudWatch Metric Streams → Kinesis Firehose → S3 / third-party tool
- Traces: ADOT → X-Ray per account, X-Ray Group dashboards in a central account
- Audit: CloudTrail organisation trail → S3 in logging account with integrity validation

**Domain 4 Validation Questions:**
- What is the difference between a CloudWatch metric filter and Contributor Insights?
- How does Embedded Metrics Format differ from `PutMetricData` for Lambda metrics at scale?
- A CloudTrail Insights event fires for `DescribeInstances` — what does this mean and how do you investigate?
- How do you create a cross-account CloudWatch dashboard showing metrics from 4 different accounts?
- What is the difference between AWS Config managed rules and custom Config rules?

---

## Phase 5 — Domain 5: Incident and Event Response (Weeks 11–12)

**Exam weight: 14% — approximately 10–11 questions**

This domain is about automated response to operational events. The key pattern is: detect via CloudWatch/EventBridge → respond via Lambda/SSM/Step Functions → notify via SNS/OpsCenter. Expect compound scenarios.

### Week 11 — EventBridge and Automated Remediation

**Core services:** EventBridge, Lambda, Step Functions, SNS, SQS, Systems Manager OpsCenter, Systems Manager Automation, AWS Health

**Topics to master:**

- [ ] **EventBridge Event Bus:** default bus (AWS service events), custom buses (application events), partner event sources
- [ ] **EventBridge Rules:** event pattern matching (exact match, prefix, suffix, `anything-but`, `exists`, numeric ranges), targets (Lambda, SQS, SNS, Step Functions, CodePipeline, EC2 Actions)
- [ ] **EventBridge Pipes:** point-to-point integration between source and target with optional filtering and enrichment (Lambda)
- [ ] **EventBridge Scheduler:** one-time vs recurring, flexible time window, target invocation (Lambda, Step Functions, CodePipeline)
- [ ] **AWS Health:** service health events vs account-specific events, `AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED` — automating EC2 replacement on retirement notice
- [ ] **Auto-remediation patterns:**
  - IAM policy attached to root account → CloudTrail → EventBridge → Lambda → detach + notify
  - Security Group opens port 22 to 0.0.0.0/0 → Config Rule → SSM Automation → revert + OpsItem
  - EC2 instance CPU > 90% for 5 min → CloudWatch Alarm → SNS → Lambda → Auto Scaling scale-out + SSM Run Command diagnostic
- [ ] **Step Functions for orchestrated remediation:** multi-step runbooks with wait states, retries, error handling (`Catch`, `Retry`), human approval via API Gateway callback
- [ ] **OpsCenter OpsItems:** creating from EventBridge rules, associating runbooks, deduplication, resolution criteria

**Knowledge portal cross-reference:**
- [`cloud/aws/09-sqs-sns-eventbridge.md`](../cloud/aws/09-sqs-sns-eventbridge.md) — EventBridge section
- [`devops/10-alerting-oncall.md`](../devops/10-alerting-oncall.md) — full read
- [`observability/06-alerting-slo-burn-rate.md`](../observability/06-alerting-slo-burn-rate.md)
- [`devops/07-sre-slos.md`](../devops/07-sre-slos.md) — full read

**Hands-on lab:** Build an automated remediation pipeline:
1. Config Rule detects Security Group with `0.0.0.0/0` ingress on port 22
2. Non-compliant → triggers SSM Automation runbook
3. Runbook: captures current rule, removes the offending ingress rule, opens an OpsItem
4. OpsItem links to the runbook execution history for audit

### Week 12 — Chaos Engineering, SRE Practices, and On-Call

**Core services:** AWS Fault Injection Service (FIS), CloudWatch SLO dashboards, Systems Manager, PagerDuty/OpsGenie integration patterns

**Topics to master:**

- [ ] **AWS Fault Injection Service (FIS):** experiment templates, fault actions (EC2 stop, CPU stress, network latency, ECS task termination, RDS failover), stop conditions (CloudWatch Alarm), safety guardrails
- [ ] **Chaos engineering process:** steady-state hypothesis → inject fault → observe → conclude — how FIS maps to each step
- [ ] **Error budgets and SLOs:** SLI (what you measure), SLO (target), error budget (tolerance consumed) — how CloudWatch Alarms encode SLOs
- [ ] **Burn rate alarms:** fast burn (high severity, small window — 1h burn rate > 14x) vs slow burn (lower severity, longer window — 6h burn rate > 6x) — the two-alarm pattern
- [ ] **Runbook automation:** SSM Automation documents as structured incident runbooks, linking OpsItems to runbook ARNs
- [ ] **Post-incident review:** using CloudTrail + CloudWatch Logs Insights to reconstruct timeline, correlating X-Ray traces to the incident window
- [ ] **Incident notification integrations:** SNS → Lambda → PagerDuty/Slack webhook (exam doesn't test third-party tools directly, but the Lambda integration pattern is tested)

**Knowledge portal cross-reference:**
- [`devops/08-chaos-engineering.md`](../devops/08-chaos-engineering.md) — full read
- [`devops/07-sre-slos.md`](../devops/07-sre-slos.md) — full read
- [`observability/05-sli-slo-error-budgets.md`](../observability/05-sli-slo-error-budgets.md) — full read

**Phase 5 Deliverable:** Design a full incident lifecycle for a production availability event:
- Detection: which CloudWatch metrics and alarms trip first
- Alert routing: CloudWatch Alarm → SNS → on-call notification
- Automated first response: SSM Automation runbook actions (within 0–5 minutes)
- Escalation: Step Functions human approval gate (after 15 minutes unresolved)
- Post-incident: Athena query on CloudTrail to reconstruct the timeline

**Domain 5 Validation Questions:**
- An EC2 instance is scheduled for retirement by AWS. How do you automate its replacement without manual intervention?
- What is the difference between EventBridge Rules and EventBridge Pipes?
- Describe the two-alarm burn rate pattern for SLO alerting. Why are two alarms needed instead of one?
- How does FIS define a "stop condition", and why is it critical for production chaos experiments?
- A Config Rule finds a non-compliant S3 bucket. What two mechanisms can automatically remediate it?

---

## Phase 6 — Domain 6: Security and Compliance (Weeks 13–14)

**Exam weight: 17% — approximately 12–13 questions**

Shares significant overlap with SAP-C02 Domain 3. DOP-C02 focuses on *automating* security enforcement - secrets rotation, automated compliance checks, and pipeline security gates — rather than security architecture design.

### Week 13 — Secret and Key Management + Pipeline Security

**Core services:** Secrets Manager, KMS, Parameter Store, IAM, CodePipeline, CodeBuild, GuardDuty, Inspector

**Topics to master:**

- [ ] **Secrets Manager rotation:** single-user vs alternating-user rotation strategies, Lambda rotation function (`createSecret`, `setSecret`, `testSecret`, `finishSecret` steps), rotation failure handling
- [ ] **KMS key policies:** key policy vs IAM policy (both required for cross-account access), `kms:ViaService` condition key, key rotation (automatic annual, on-demand), key deletion (7–30 day waiting period)
- [ ] **Pipeline security gates:** CodeBuild stage running SAST (Semgrep, Checkmarx), SCA (OWASP Dependency Check), container image scanning (ECR Inspector) — gate on HIGH/CRITICAL findings
- [ ] **CodeBuild environment security:** no secrets in buildspec (use Parameter Store/Secrets Manager), VPC CodeBuild for private resource access, build role least-privilege
- [ ] **GuardDuty findings in pipelines:** EventBridge rule on `GuardDuty Finding` → Lambda → CodePipeline `StopExecution` (for critical findings in production account)
- [ ] **Inspector v2:** EC2 + Lambda + ECR scanning, SBOM export, findings severity threshold enforcement
- [ ] **CodeArtifact + SCA:** pulling packages only from CodeArtifact (policy), blocking public package pull-through on vulnerability match

**Knowledge portal cross-reference:**
- [`devsecops/05-secret-management.md`](../devsecops/05-secret-management.md) — full read
- [`devsecops/02-sast.md`](../devsecops/02-sast.md) — full read
- [`devsecops/03-sca-dependency-scanning.md`](../devsecops/03-sca-dependency-scanning.md) — full read
- [`devsecops/07-container-security.md`](../devsecops/07-container-security.md)
- [`devsecops/08-supply-chain-sbom.md`](../devsecops/08-supply-chain-sbom.md)
- [`devsecops/14-security-cicd-pipeline.md`](../devsecops/14-security-cicd-pipeline.md) — full read

### Week 14 — Compliance Automation and Policy Enforcement

**Core services:** AWS Config, Security Hub, CloudFormation Guard, Service Control Policies, AWS Audit Manager

**Topics to master:**

- [ ] **AWS Config Rules as compliance:** managed rules (e.g., `encrypted-volumes`, `s3-bucket-public-read-prohibited`, `iam-password-policy`) vs custom Lambda rules vs Guard (proactive evaluation)
- [ ] **Config conformance packs:** bundled rules + remediation, AWS-managed packs (CIS, NIST, PCI-DSS, HIPAA), deploy via StackSets across organisation
- [ ] **CloudFormation Guard (`cfn-guard`):** policy-as-code rules in `.guard` syntax, `cfn-guard validate` in CodeBuild as a pre-deploy gate, proactive CloudFormation Hooks
- [ ] **Security Hub:** CSPM findings (aggregated from GuardDuty, Macie, Inspector, Config), custom action → EventBridge → remediation Lambda, cross-account aggregation (delegated admin)
- [ ] **Audit Manager:** evidence collection frameworks (GDPR, SOC2, PCI-DSS), automated evidence (Config snapshots, CloudTrail events), assessment reports
- [ ] **SCPs as preventive controls:** `Deny` on disabling CloudTrail, `Deny` on leaving AWS Organizations, `Deny` on non-compliant region usage — write and validate actual SCP JSON
- [ ] **IAM Access Analyzer:** resource-based policies with external access findings, path analysis (validates that a permission path exists), custom policy checks in pipeline

**Knowledge portal cross-reference:**
- [`devsecops/13-compliance-as-code.md`](../devsecops/13-compliance-as-code.md) — full read
- [`devsecops/10-policy-as-code.md`](../devsecops/10-policy-as-code.md) — full read
- [`devsecops/06-iac-security.md`](../devsecops/06-iac-security.md) — CFN Guard section
- [`devsecops/01-shift-left-security.md`](../devsecops/01-shift-left-security.md)

**Phase 6 Deliverable:** Design a "Security as Code" pipeline that enforces compliance on every CloudFormation deployment:
1. `cfn-guard` validates template against organisational policies (CodeBuild stage)
2. IAM Access Analyzer validates IAM roles in the template (CodeBuild stage, Lambda invoke)
3. Post-deploy: Config conformance pack checks all resources within 15 minutes
4. Security Hub custom action auto-raises a Jira/OpsItem for any HIGH findings
5. Document the SCPs that make all of the above non-bypassable

**Domain 6 Validation Questions:**
- What are the four steps in a Secrets Manager Lambda rotation function?
- How does `kms:ViaService` restrict which AWS service can use a KMS key?
- What is the difference between a Config Rule and a CloudFormation Hook?
- A developer commits AWS credentials to CodeCommit. What chain of events should automatically detect and respond to this?
- How does Security Hub aggregate findings from GuardDuty, Macie, and Config, and where can you take automated action?

---

## Phase 7 — Mock Exams, Gap Remediation, and Exam-Day Prep (Weeks 15–16)

### Week 15 — Timed Full Mock Exams

Complete in order under timed, full-exam conditions (180 minutes, no pause):

| Exam | Source | Target Score |
|------|--------|--------------|
| Mock 1 | [Tutorials Dojo DOP-C02](https://portal.tutorialsdojo.com/courses/aws-certified-devops-engineer-professional-practice-exams/) | 70%+ |
| Mock 2 | [Whizlabs DOP-C02](https://www.whizlabs.com/aws-devops-certification-training/) | 72%+ |
| Mock 3 | [AWS Skill Builder Official](https://skillbuilder.aws/exam-prep/devops-engineer-professional) | 75%+ |
| Mock 4 | Tutorials Dojo (timed section mode per domain) | 80%+ per domain |

After every mock exam:
1. Review every wrong answer — no exceptions.
2. Tag by domain and sub-topic in your study log.
3. For every wrong answer, open the official AWS documentation for the relevant service — read the source, not a blog post.

**High-failure-rate topics (historical exam reports):**
- CodeDeploy `appspec.yml` lifecycle hook order for EC2 vs Lambda vs ECS
- CloudFormation `DependsOn` vs `WaitCondition` vs `CreationPolicy` — when to use each
- SSM Automation vs Run Command vs State Manager — choosing the right tool
- EventBridge vs CloudWatch Events — they are the same service; new name = EventBridge
- CodePipeline cross-account deployment — which IAM roles are required in which accounts
- Config Rules: triggered by configuration changes vs triggered periodically — different query patterns

### Week 16 — Exam Strategy and Final Polish

**Exam technique:**

- [ ] **Read the question stem for the constraint.** DOP-C02 questions always have a constraint: "minimum operational overhead", "without modifying application code", "using only AWS native services", "most cost-effective". This constraint eliminates 2 of the 4 options immediately.
- [ ] **CodeSuite questions:** always check the lifecycle hook order for CodeDeploy (it changes between EC2, Lambda, ECS). Draw it from memory before the exam.
- [ ] **"Notify" vs "Auto-remediate":** if the question says "notify the team and automatically fix", the pattern is SNS (notify) + Lambda/SSM (remediate) both invoked from the same CloudWatch Alarm or EventBridge Rule.
- [ ] **CloudWatch Alarms fire actions:** to Auto Scaling, SNS, EC2 (stop/terminate/reboot), and Systems Manager OpsItem — not directly to Lambda. Lambda is triggered via SNS subscription.
- [ ] **Config Rules vs CloudTrail vs GuardDuty:** Config = resource configuration compliance; CloudTrail = API call audit; GuardDuty = threat intelligence (behaviour anomalies). Know which to use for which scenario.

**Known exam traps:**

| Trap | The Trick |
|------|-----------|
| "Automatically patch EC2 instances with minimum downtime" | Patch Manager + Maintenance Window (not Run Command alone) |
| "Zero-downtime Lambda deployment with gradual shift" | CodeDeploy Lambda Canary or Linear — NOT all-at-once |
| "Store database credentials securely, auto-rotate" | Secrets Manager — NOT Parameter Store SecureString |
| "Prevent human SSH access to EC2" | Session Manager — remove port 22 security group rule entirely |
| "CloudFormation stack fails on rollback — stack stuck" | `ROLLBACK_COMPLETE` state → must delete and redeploy; or use `--disable-rollback` flag |
| "Detect if S3 bucket is made public" | Config Rule (`s3-bucket-public-read-prohibited`) OR GuardDuty S3 protection — Config for compliance, GuardDuty for threat detection |
| "Detect unusual API activity" | CloudTrail Insights — not GuardDuty (GuardDuty uses CloudTrail data but detects behaviour patterns, not API volume anomalies) |
| "Cross-account CodePipeline deploy" | Pipeline role in source account needs `sts:AssumeRole` on deploy role in target account; both KMS key policies must allow cross-account access |

**Final two days:**
- [ ] Reread only your own wrong-answer notes — not new material.
- [ ] Write on scratch paper: six domain names + weights, CodeDeploy lifecycle hooks for all three compute types, CloudFormation intrinsic functions you always forget.
- [ ] Confirm booking, ID, and environment.

---

## Core Services — Mastery Checklist

### CI/CD and DevOps Toolchain
- [ ] CodeCommit (triggers, approval rules, repository notifications)
- [ ] CodeBuild (buildspec, caching, reports, VPC builds, Docker-in-Docker)
- [ ] CodeArtifact (domains, repos, upstream chaining, cross-account)
- [ ] CodeDeploy (EC2, Lambda, ECS; appspec, lifecycle hooks, rollback)
- [ ] CodePipeline (stages, actions, parallel, manual approval, cross-account)
- [ ] EC2 Image Builder (pipelines, recipes, distribution)
- [ ] Elastic Beanstalk (deployment modes, ebextensions, managed updates)
- [ ] AppConfig (deployment strategies, validators, rollback)

### Infrastructure as Code
- [ ] CloudFormation (all intrinsic functions, change sets, nested stacks, StackSets, drift, custom resources, hooks, cfn-init, cfn-signal)
- [ ] CDK (construct levels, synth, deploy, cdk.json)
- [ ] CloudFormation Guard (cfn-guard syntax, validation in CI)
- [ ] Service Catalog (portfolios, products, launch constraints)

### Compute and Auto Scaling
- [ ] EC2 Auto Scaling (all scaling policy types, lifecycle hooks, warm pools, instance refresh, mixed instances)
- [ ] Application Auto Scaling (DynamoDB, ECS, Lambda concurrency, Aurora)
- [ ] ECS Auto Scaling (service tasks, custom metrics via Lambda)
- [ ] Lambda (concurrency, SnapStart, destinations, EMF, layers)
- [ ] AWS Fault Injection Service (experiment templates, actions, stop conditions)

### Configuration Management
- [ ] SSM Session Manager, Run Command, Patch Manager, State Manager, Automation, Parameter Store, Inventory, OpsCenter, Fleet Manager
- [ ] SSM Documents (Command, Automation, Session, Policy types)
- [ ] SSM Hybrid activations (on-premises managed instances)

### Monitoring and Observability
- [ ] CloudWatch Metrics (namespaces, dimensions, metric math, EMF)
- [ ] CloudWatch Alarms (single metric, composite, anomaly detection)
- [ ] CloudWatch Logs (metric filters, subscription filters, Log Insights)
- [ ] CloudWatch Synthetics (canaries, visual monitoring)
- [ ] CloudWatch Contributor Insights, Evidently, Metric Streams
- [ ] X-Ray (sampling, groups, annotations, subsegments, service map)
- [ ] ADOT (collector pipeline, OTEL SDK integration)

### Events and Incident Response
- [ ] EventBridge (event buses, rules, pipes, scheduler)
- [ ] AWS Health (service events, account events, partner events)
- [ ] Step Functions (Standard vs Express, error handling, callback pattern)
- [ ] SNS (topics, subscriptions, message filtering, FIFO)
- [ ] SQS (Standard vs FIFO, visibility timeout, DLQ, batch)

### Security and Compliance
- [ ] Secrets Manager (rotation strategies, Lambda function, cross-account)
- [ ] KMS (key policies, grants, `kms:ViaService`, automatic rotation, cross-account)
- [ ] AWS Config (rules, conformance packs, aggregators, remediation)
- [ ] Security Hub (findings, custom actions, cross-account aggregation)
- [ ] GuardDuty (finding types, suppression, S3 protection, EKS protection)
- [ ] IAM Access Analyzer (external access findings, path analysis, policy validation)
- [ ] Audit Manager (evidence frameworks, assessment reports)
- [ ] Inspector v2 (EC2, ECR, Lambda scanning; SBOM export)

---

## Resources — Prioritised

### Free Resources
| Resource | URL | Priority |
|----------|-----|----------|
| AWS Skill Builder Exam Prep (DOP-C02) | [skillbuilder.aws](https://skillbuilder.aws/exam-prep/devops-engineer-professional) | ★★★★★ |
| Official AWS Documentation | [docs.aws.amazon.com](https://docs.aws.amazon.com/) | ★★★★★ |
| CloudFormation User Guide | [docs.aws.amazon.com/AWSCloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/) | ★★★★★ |
| SSM User Guide | [docs.aws.amazon.com/systems-manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/) | ★★★★★ |
| AWS DevOps Blog | [aws.amazon.com/blogs/devops](https://aws.amazon.com/blogs/devops/) | ★★★★☆ |
| AWS Well-Architected — Operational Excellence | [docs.aws.amazon.com/wellarchitected](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html) | ★★★★☆ |

### Paid Resources
| Resource | Type | Notes |
|----------|------|-------|
| [Adrian Cantrill DOP-C02](https://learn.cantrill.io/p/aws-certified-devops-engineer-professional) | Video course | Most thorough; recommended |
| [A Cloud Guru DOP-C02](https://www.pluralsight.com/cloud-guru) | Video course | Good for hands-on labs |
| [Tutorials Dojo DOP-C02](https://portal.tutorialsdojo.com/) | Practice exams | Essential — best question quality |
| [Whizlabs DOP-C02](https://www.whizlabs.com/) | Practice exams | Good volume; use as secondary source |

### This Knowledge Portal
| Domain | Docs |
|--------|------|
| 1 — SDLC Automation | `devops/01-cicd-pipeline-design.md`, `devops/04-deployment-strategies.md`, `devops/05-feature-flags.md`, `devops/12-artifact-release-management.md`, `devops/15-devops-metrics.md` |
| 2 — Config Mgmt & IaC | `devops/03-infrastructure-as-code.md`, `devops/13-configuration-management.md`, `devsecops/06-iac-security.md`, `devsecops/10-policy-as-code.md` |
| 3 — Resilience | `cloud/aws/14-disaster-recovery.md`, `devops/14-disaster-recovery.md`, `devops/08-chaos-engineering.md` |
| 4 — Monitoring | `cloud/aws/11-observability-cloudwatch.md`, `devops/06-observability-opentelemetry.md`, `observability/` (all) |
| 5 — Incident Response | `devops/10-alerting-oncall.md`, `devops/07-sre-slos.md`, `devops/08-chaos-engineering.md`, `cloud/aws/09-sqs-sns-eventbridge.md` |
| 6 — Security & Compliance | `devsecops/` (all), `devsecops/05-secret-management.md`, `devsecops/13-compliance-as-code.md`, `devsecops/14-security-cicd-pipeline.md` |

---

## Final Readiness Gate

Do not sit the exam until all boxes are checked:

```
Domains
  [ ] Domain 1 practice score consistently ≥ 75%
  [ ] Domain 2 practice score consistently ≥ 75%
  [ ] Domain 3 practice score consistently ≥ 75%
  [ ] Domain 4 practice score consistently ≥ 75%
  [ ] Domain 5 practice score consistently ≥ 75%
  [ ] Domain 6 practice score consistently ≥ 75%

Mock exams
  [ ] 3 full timed mock exams completed
  [ ] Scoring ≥ 78% on most recent mock
  [ ] All wrong answers reviewed and mapped to source docs

Service mastery
  [ ] Can draw the CodeDeploy lifecycle hook order for EC2, Lambda, and ECS from memory
  [ ] Can describe all four SSM Automation document action types without notes
  [ ] Can write a CloudFormation template with a Custom Resource and CreationPolicy
  [ ] Knows the difference between Config conformance packs and Security Hub standards
  [ ] Can describe the EventBridge → Lambda → SSM auto-remediation chain end-to-end
  [ ] Knows all CodeBuild caching options and their trade-offs

Hands-on
  [ ] Has built at least one end-to-end CodePipeline (source → build → test → deploy)
  [ ] Has deployed a CloudFormation StackSet across multiple accounts
  [ ] Has set up at least one CloudWatch Alarm with auto-remediation via SSM

Exam mechanics
  [ ] Exam booked via Pearson VUE
  [ ] ID prepared
  [ ] Testing environment validated
```

---

## Certification Roadmap Context

| Path | Cert | Synergy with DOP-C02 |
|------|------|-----------------------|
| Already have SAP-C02 | — | Significant overlap in security, resilience, and networking domains |
| Security depth | [AWS Security – Specialty](https://aws.amazon.com/certification/certified-security-specialty/) | Deep KMS, GuardDuty, IAM, Detective |
| Networking depth | [AWS Advanced Networking – Specialty](https://aws.amazon.com/certification/certified-advanced-networking-specialty/) | VPC, Direct Connect, Route 53 advanced |
| Data platform | [AWS Data Engineer – Associate](https://aws.amazon.com/certification/certified-data-engineer-associate/) | Kinesis, Glue, Redshift pipelines |

Holding both **SAP-C02 + DOP-C02** is widely regarded as the strongest AWS professional-level credential combination for senior cloud engineers.

---

*Last updated: April 2026 — review against the official exam guide every 6 months as DOP-C02 content is refreshed.*
