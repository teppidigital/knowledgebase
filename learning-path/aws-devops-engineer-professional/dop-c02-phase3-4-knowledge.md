# DOP-C02 Phases 3 & 4 — Deep Knowledge Reference
## Domain 3: Resilient Cloud Solutions (Weeks 7–8)
## Domain 4: Monitoring and Logging (Weeks 9–10)

> Detailed technical knowledge for Phases 3 and 4 of the DOP-C02 study plan.
> Cross-reference: [`aws-devops-engineer-professional.md`](./aws-devops-engineer-professional.md)

---

# DOMAIN 3: RESILIENT CLOUD SOLUTIONS

## Week 7 — Auto Scaling and Self-Healing

### EC2 Auto Scaling Policy Types

| Type | Behaviour | Best for |
|------|-----------|---------|
| **Target tracking** | Scale to keep a metric at a target value (e.g., CPU = 60%) | Steady, proportional load |
| **Step scaling** | Scale by fixed amounts based on alarm breach magnitude | Irregular spikes requiring different capacities |
| **Scheduled scaling** | Scale at a specific time | Predictable patterns (open/close of business, batch windows) |
| **Predictive scaling** | ML-based forecast; pre-warms capacity before anticipated demand | Recurring daily/weekly patterns |

**Target tracking metrics (built-in):**
- `ASGAverageCPUUtilization`
- `ASGAverageNetworkIn`, `ASGAverageNetworkOut`
- `ALBRequestCountPerTarget` (requires ALB attachment)

**Custom metric for target tracking:** Supply a `CustomizedMetricSpecification` — any CloudWatch metric that is proportional to load (e.g., SQS `ApproximateNumberOfMessagesVisible` divided by running task count).

---

### Lifecycle Hooks

Lifecycle hooks pause the instance at specific state transitions to run custom actions.

**Scale-out hook (`autoscaling:EC2_INSTANCE_LAUNCHING`):**
```
Pending → Pending:Wait → [your script runs] → Pending:Proceed → InService
```

**Scale-in hook (`autoscaling:EC2_INSTANCE_TERMINATING`):**
```
Terminating → Terminating:Wait → [your script runs] → Terminating:Proceed → Terminated
```

**Common uses:**
- Launch hook: install agents, warm up application cache, register with service discovery.
- Terminate hook: drain connections, upload logs, deregister from service discovery, send final metrics.

**Notify the hook completion:**
```bash
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name MyHook \
  --auto-scaling-group-name MyASG \
  --lifecycle-action-result CONTINUE \   # or ABANDON (ASG terminates/abandons)
  --instance-id i-0123456789abcdef0
```

Default heartbeat timeout: 3600 seconds (1 hour). Use SSM Automation to complete the hook automatically.

---

### Warm Pools

A warm pool maintains a set of **pre-initialised, stopped EC2 instances** ready to replace cold launches.

**Without warm pool:** Scale-out event triggers a cold launch (full boot + application startup — may take 3–10 minutes).
**With warm pool:** Instances in pool are started from stopped state (30–60 seconds) and enter the ASG immediately.

**States in the warm pool:**
- `Stopped` (default): Pay only for EBS; no compute charge.
- `Running`: Pay full compute + EBS; instances are live but not taking traffic.
- `Hibernated`: Fast start (RAM state saved to EBS); applicable instance types only.

**Cost:** EBS volume charges accumulate for stopped instances. Justified when:
- Application startup time > 3 minutes.
- Traffic spikes are sudden with < 5 minutes acceptable scale-out latency.

**Reuse lifecycle hooks:** The `autoscaling:EC2_INSTANCE_LAUNCHING` hook fires when an instance moves FROM the warm pool INTO the ASG (not during warm pool initialisation — that uses a separate lifecycle `WarmPoolWarmedEntryStatus` hook).

---

### Launch Templates vs Launch Configurations

**Launch Configurations** are the legacy approach. AWS no longer adds new features to them.

**Launch Templates** support:
- Multiple instance types in a mixed instances policy (required for Spot diversification)
- Latest EC2 features (Nitro, Graviton, capacity reservations, dedicated hosts)
- Versioning (a new version of the template can be set as the default)
- Spot and On-Demand blend in the same ASG

**Mixed instances policy (Exam pattern):**
```json
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "price-capacity-optimized"
    },
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {"LaunchTemplateId": "lt-abc", "Version": "$Latest"},
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5a.large"},
        {"InstanceType": "m4.large"}
      ]
    }
  }
}
```

---

### Instance Refresh

Instance Refresh replaces instances in an ASG with a new Launch Template version (e.g., after patching the AMI).

| Property | Detail |
|----------|--------|
| `MinHealthyPercentage` | Minimum % of healthy instances that must remain during refresh |
| `MaxHealthyPercentage` | Maximum % above desired capacity (room for new instances before old are terminated) |
| `InstanceWarmupSeconds` | How long to wait after new instance is InService before replacing the next one |
| `SkipMatching` | Skip replacement of instances already running the new launch template version (hash-based) |

**Instance Refresh vs CodeDeploy rolling deployment:**

| | Instance Refresh | CodeDeploy |
|--|-----------------|------------|
| What changes | AMI/Launch Template version | Application code (without replacing OS) |
| Mechanism | Launches new instances, terminates old ones | Runs lifecycle hooks on existing instances |
| Rollback | Launch old LT version, trigger another refresh | CodeDeploy rollback to previous revision |
| Use case | OS patching, AMI updates | Application updates |

---

### ECS Auto Scaling

ECS service auto scaling uses **Application Auto Scaling** (not EC2 Auto Scaling).

**Target tracking for ECS:**
- `ECSServiceAverageCPUUtilization` — scale on average CPU across all tasks
- `ECSServiceAverageMemoryUtilization` — scale on average memory
- `ALBRequestCountPerTarget` — scale on ALB request count per target (requires ALB + target group)

**Custom SQS metric for ECS:**
```
SQS queue depth / number of running ECS tasks = backlog per task
Target tracking on this custom metric: each task processes N messages/minute
Scale to maintain N messages/task at all times
```

---

### Health Check Types and Replacement Behaviour

| Health check type | What it checks | Replacement triggered by |
|-------------------|---------------|-------------------------|
| **EC2** (default) | Instance status checks (hardware, hypervisor) | EC2 status check failure |
| **ELB** | ALB/NLB health check on the application | Application returning non-2xx or failing TCP |

**Exam trap:** EC2 health check only detects hardware failures. If your app crashes but the instance is healthy, **EC2 health check will NOT replace the instance**. Use ELB health checks for application-level replacement.

---

### Self-Healing Pattern — Auto-Remediation Architecture

```
1. CloudWatch Alarm fires (5xx error rate > threshold for 5 minutes)
2. Alarm action:
   a. SNS topic → notification to on-call
   b. SSM Automation runbook → start auto-remediation
3. SSM Automation steps:
   a. Describe ALB target group — identify unhealthy targets
   b. For each unhealthy target: terminate EC2 instance (ASG replaces it)
   c. Wait for replacement instance to be healthy (aws:waitForAwsResourceProperty)
   d. Run CloudWatch Synthetics canary to validate recovery
   e. Send summary to SNS (success or escalate)
```

---

## Week 8 — High Availability and DR Automation

### Route 53 Health Checks for Failover

**Health check types:**

| Type | How |
|------|-----|
| **Endpoint health check** | HTTP/HTTPS/TCP to an IP or hostname; checks status code and optional string match |
| **Calculated health check** | Composite: healthy if X of N child health checks are healthy |
| **CloudWatch Alarm health check** | Healthy if the specified alarm is in `OK` state |

**Failover routing for DR:**
```
Primary record (us-east-1 ALB) — associated with health check
Secondary record (eu-west-1 ALB) — no health check (passive standby)

If primary health check fails → Route 53 stops serving primary record → serves secondary
```

**Health check string matching:** Route 53 checks the first 5,120 bytes of the response body for a specific string. Use to detect degraded responses even when HTTP 200 is returned.

---

### AWS Backup

AWS Backup provides centralised, policy-driven backup across AWS services.

**Supported services:** EC2 (EBS snapshots + AMIs), RDS, Aurora, DynamoDB, EFS, FSx, Storage Gateway, DocumentDB, Neptune, S3, VMware via Backup Gateway.

**Backup plan:** Defines schedule (cron), backup window, lifecycle, and destination vault.

**Backup vault:**
- Encrypted S3-backed store for your backups.
- Vault Lock: WORM (Write Once Read Many) protection — backups cannot be deleted during the retention period (compliance).

**Cross-region copy:** Backup plan rules can include cross-region copy configuration. Backups are copied to a vault in another region automatically.

**Cross-account copy:** Copy backups to a different AWS account (e.g., to the security/backup account) — provides protection against accidental or malicious deletion in the production account.

**Backup Audit Manager:** Evidence collection for backup compliance — generates reports showing which resources are covered by backup plans and whether backups completed successfully.

---

### AWS Elastic Disaster Recovery (DRS)

DRS (formerly CloudEndure Disaster Recovery) provides **agent-based, continuous block-level replication** for any physical, virtual, or cloud servers.

**How it works:**
1. Install the DRS replication agent on the source server.
2. Continuous block-level replication to a lightweight staging area in the target AWS region.
3. On failover: launch recovery instances from the latest replicated state.
4. Launch templates define the instance type, VPC, subnet, and IAM role for recovery instances.

**Point-in-time recovery:** DRS retains block-level journal data for up to 90 days. You can failover to a point-in-time (before ransomware or corruption).

**DRS vs MGN:**
- DRS: disaster recovery — continuous replication, failover on failure.
- MGN: migration — replicate once, cut over when ready, archive source.
Both use the same agent technology.

---

### SQS as a Resilience Buffer

**Pattern:**
```
Service A → SQS Queue → Service B

If Service B is down, messages accumulate in SQS (up to 14 days retention).
Service B recovers → drains queue naturally.
Service A never knew Service B was down.
```

**Failure isolation:** Without SQS, if Service B is slow, Service A backs up and eventually fails. With SQS, Service A is always available (queue accepts messages instantly).

**DLQ for poison pills:** Messages that fail processing `maxReceiveCount` times are moved to the DLQ. Monitor DLQ depth with CloudWatch Alarm → alert operations. Do NOT let items silently accumulate in the DLQ without review.

---

### Pipeline-Driven DR Testing

```
CodePipeline (monthly schedule via EventBridge Scheduler)
  Stage 1: Lambda — spin up recovery environment (CloudFormation create-stack)
  Stage 2: Lambda — wait for stack creation, validate endpoints
  Stage 3: Lambda — run synthetic test against recovery environment
  Stage 4: Manual Approval — human validates the DR environment
  Stage 5: Lambda — tear down recovery environment (CloudFormation delete-stack)
  Stage 6: Lambda — publish DR test results to S3 and SNS
```

Value: DR is regularly tested; the pipeline is the runbook. Human error is eliminated.

---

# DOMAIN 4: MONITORING AND LOGGING

## Week 9 — CloudWatch Deep Dive

### CloudWatch Metrics Fundamentals

| Concept | Detail |
|---------|--------|
| **Namespace** | Logical grouping: `AWS/EC2`, `AWS/Lambda`, `MyApp/Payments` |
| **Metric** | A named data series: `CPUUtilization`, `ErrorCount` |
| **Dimension** | Key-value pair that scopes the metric: `InstanceId=i-abc`, `FunctionName=orders` |
| **Resolution** | Standard (60-second) or High-Resolution (1-second, extra cost) |
| **PutMetricData** | API to publish custom metrics (up to 20 metrics per call) |

**Metric math:**
```
Expression: m1 / m2 * 100
  m1 = AWS/ApplicationELB HTTPCode_ELB_5XX_Count
  m2 = AWS/ApplicationELB RequestCount
Result: 5xx error percentage — can alarm on this derived metric
```

---

### CloudWatch Alarms

**Alarm states:**
- `OK` — metric is within threshold
- `ALARM` — metric has breached threshold
- `INSUFFICIENT_DATA` — not enough data points to evaluate (new alarm, sparse metric)

**Evaluation:**
- Alarm evaluates N data points within M periods (e.g., 3 of 5 consecutive 1-minute periods)
- "M-of-N" evaluation prevents noisy alarms (transient spikes don't fire the alarm)

**Composite alarms:**
- Combine multiple alarms with `AND`/`OR`/`NOT` logic.
- Use to reduce noise: page only when BOTH high CPU AND high memory are breached simultaneously.
```json
{
  "AlarmRule": "ALARM(CPUAlarm) AND ALARM(MemoryAlarm)"
}
```

**Anomaly detection alarms:**
- CloudWatch creates a band of expected values based on historical data (ML model).
- Alarm fires when the metric falls outside the band.
- No threshold to define — the model learns seasonality (hourly, daily, weekly patterns).

**Alarm actions:**
- Auto Scaling → `ScalingPolicy`
- SNS topic → Email, Lambda, HTTP endpoint
- EC2 → stop, terminate, reboot, recover
- Systems Manager → create OpsItem
- **NOT directly to Lambda** — Lambda is triggered via SNS subscription.

---

### CloudWatch Logs

**Log groups → Log streams:**
- Log group: a collection of log streams with the same retention policy and permissions.
- Log stream: an individual source (one Lambda function execution, one EC2 instance).

**Metric filters:** Extract a numeric or count metric from log data without code changes:
```
Filter pattern: [timestamp, requestId, level="ERROR", message]
Metric name: ErrorCount
Namespace: MyApp/Payments
Value: 1 (count each match)
```

**Subscription filters** deliver log data to:
- Lambda (synchronous, per-batch processing)
- Kinesis Data Streams (real-time processing)
- Kinesis Firehose (delivery to S3, OpenSearch, Splunk)

**Cross-account subscription filter:**
```
Account A log group → subscription filter → cross-account Kinesis Firehose (in Account B)

Account B: Kinesis Firehose → S3 in logging account
```
Requires a resource policy on the Kinesis Firehose allowing Account A to put records.

---

### CloudWatch Logs Insights

SQL-like query language for interactive log analysis.

**Key syntax:**
```
# Count errors by error type
fields @timestamp, errorType, @message
| filter level = "ERROR"
| stats count() as errorCount by errorType
| sort errorCount desc
| limit 20

# Percentile latency
fields @timestamp, duration
| filter @message like /END RequestId/
| stats pct(duration, 99) as p99, pct(duration, 95) as p95, avg(duration) as avg

# Lambda cold starts
filter @type = "REPORT"
| fields @memorySize, @billedDuration, @initDuration
| filter ispresent(@initDuration)   # cold starts have initDuration
| stats avg(@initDuration) by bin(30m)
```

**Cross-log-group queries:** Query up to 50 log groups in one Insights query.

---

### CloudWatch Synthetics — Canaries

Canaries are **scheduled scripts** that simulate user interactions to continuously monitor endpoints.

**Built-in blueprints:**
| Blueprint | Use |
|-----------|-----|
| `Heartbeat` | Simple HTTP GET; check response code and optionally body content |
| `API Canary` | Series of API calls; validate request/response pairs |
| `Broken Link Checker` | Crawl a page and report broken links |
| `GUI Workflow Builder` | Puppeteer-based browser simulation |
| `Visual Monitoring` | Screenshot-based comparison (detects UI regressions) |

**Canary schedule:** Cron or rate expression. Minimum 1 minute intervals.

**Outputs:** S3 screenshots and HAR files per run. CloudWatch metrics: `SuccessPercent`, `Duration`.

**Canary run status CloudWatch metric:**
```
Alarm: SuccessPercent < 100% for 2 consecutive periods
  → SNS topic → On-call notification + automated rollback trigger
```

---

### Embedded Metrics Format (EMF)

EMF allows Lambda (and ECS/EC2) to publish CloudWatch metrics by writing structured JSON to stdout — no `PutMetricData` API call required, no additional IAM permissions, no extra cost for the API call.

**EMF log structure:**
```python
import json

def handler(event, context):
    # Business logic
    order_total = 150.00
    processing_time_ms = 45

    # Emit EMF metrics via stdout
    print(json.dumps({
        "_aws": {
            "Timestamp": 1712000000000,
            "CloudWatchMetrics": [{
                "Namespace": "MyApp/Orders",
                "Dimensions": [["FunctionName", "Environment"]],
                "Metrics": [
                    {"Name": "OrderTotal", "Unit": "None"},
                    {"Name": "ProcessingTime", "Unit": "Milliseconds"}
                ]
            }]
        },
        "FunctionName": context.function_name,
        "Environment": "production",
        "OrderTotal": order_total,
        "ProcessingTime": processing_time_ms
    }))
```

**Key advantage over `PutMetricData`:** At scale, millions of Lambda invocations would require millions of API calls. EMF writes to CloudWatch Logs (cheap) and CloudWatch automatically extracts metrics from the log data asynchronously.

---

### Contributor Insights

Contributor Insights identifies the **top-N contributors** to a metric by analysing log data.

**Example rules:**
- Top 10 IPs sending the most requests to an ALB (based on ALB access logs)
- Top 10 slowest API paths (based on API Gateway logs)
- Top 10 Lambda functions with the most cold starts

**Built-in rule:** `AWS/ApiGateway` endpoints, `AWS/DynamoDB` top partitions — AWS provides pre-built rules.

**Custom rule:**
```yaml
Schema:
  Name: MyRule
  Schema:
    CloudWatchLogRule:
      # JSON path into the log line to extract the contributor key
      KeyFields:
        - $.userId
        - $.httpMethod
      ContributorFields:
        - $.latency
LogGroupNames:
  - /aws/apigateway/accesslogs
```

---

### CloudWatch Evidently

Evidently provides **feature flags + A/B experiment metrics** as a managed AWS service.

| Concept | Description |
|---------|-------------|
| **Project** | Logical grouping of features and experiments |
| **Feature** | A code path toggle (enabled/disabled or variant selection) |
| **Launch** | Gradual rollout of a feature by traffic % — no metrics comparison |
| **Experiment** | A/B test: compare metric outcomes (e.g., checkout conversion) between variants |
| **Override** | Force a specific variant for a specific user (for testing) |

**SDK integration:** Applications call `evaluateFeature` to get the variant for a given entity (user ID, session ID). Results are deterministic per entity — same user always gets the same variant.

---

### Metric Streams

Deliver CloudWatch metrics **near-real-time** to Kinesis Firehose → S3 or third-party observability tools (Datadog, New Relic, Dynatrace, Splunk).

**Advantages over API polling:**
- Sub-minute delivery (vs 1-minute polling latency)
- Push-based (no API rate limits)
- Can stream ALL metrics in an account or filter by namespace

```
CloudWatch → Metric Stream → Kinesis Firehose → S3 (Parquet/JSON)
                                              → Datadog HTTP endpoint
```

---

## Week 10 — Centralised Logging, Tracing, and Audit

### CloudTrail Deep Dive

| Event type | What it records | Default | Cost |
|------------|----------------|---------|------|
| **Management events** (control plane) | CreateBucket, RunInstances, AuthorizeSecurityGroupIngress | ON (free for first copy) | Free |
| **Data events** | S3 object-level (GetObject, PutObject); Lambda invocations; DynamoDB PutItem | OFF | Per-event charge |
| **Insights events** | Unusual API call rate or error rate | OFF | Per-event charge |

**CloudTrail Lake:** SQL-based event analysis directly within CloudTrail. No need to export to S3 + Athena. Query using SQL across all trails in the organization.

```sql
SELECT eventSource, eventName, errorCode, COUNT(*) as count
FROM aws_cloudtrail_lake.my_event_data_store
WHERE eventTime > '2026-04-01 00:00:00'
  AND errorCode IS NOT NULL
GROUP BY eventSource, eventName, errorCode
ORDER BY count DESC
LIMIT 20;
```

**Insights events:** CloudTrail detects if the volume of an API call is statistically unusual compared to a 7-day baseline. Example: sudden spike in `RunInstances` calls. Delivery to S3; EventBridge events fire on new Insights findings.

**Log file integrity validation:** CloudTrail signs each log file with a SHA-256 hash. Use the `validate-logs` CLI command to verify no log files have been modified or deleted.

---

### AWS Config — Deep Dive

**Configuration item:** A snapshot of an AWS resource's configuration at a specific point in time. Stored when the resource changes. All items stored in S3 (configuration history).

**Config rule types:**

| Type | Triggering | Implementation |
|------|-----------|---------------|
| **Managed (configuration change)** | When resource changes | AWS-managed (no code) |
| **Managed (periodic)** | On a schedule (1h, 3h, 6h, 24h) | AWS-managed (no code) |
| **Custom Lambda** | On change or periodic | Lambda function you write |
| **Custom Guard** | On change (proactive: before deploy) | CloudFormation Guard policy |

**Remediation actions:**
- **Manual:** Tag the finding for a human to investigate.
- **Automatic:** Run an SSM Automation document automatically when a non-compliant resource is found.

**Config Conformance packs:** A collection of Config rules + remediation actions packaged together, deployable via CloudFormation or StackSets.

| Managed conformance pack | Standard it maps to |
|--------------------------|-------------------|
| `Operational-Best-Practices-for-CIS-AWS-v1.4-Level1` | CIS AWS Foundations Benchmark |
| `AWS-NIST-Cybersecurity-Framework` | NIST CSF |
| `Operational-Best-Practices-for-PCI-DSS` | PCI-DSS |
| `Operational-Best-Practices-for-HIPAA` | HIPAA |

---

### AWS X-Ray

X-Ray traces requests as they flow through distributed systems.

**Core concepts:**

| Concept | Description |
|---------|-------------|
| **Trace** | End-to-end request journey from entry to all downstream services |
| **Segment** | Work done by one service for a request |
| **Subsegment** | Child of a segment — DB call, HTTP call, Lambda downstream invoke |
| **Annotation** | Indexed key-value; used for filtering traces in Groups |
| **Metadata** | Non-indexed key-value; stored in the trace but not searchable |
| **Service map** | Visual graph of all services and their trace data |

**Sampling rules:**
- Default: 1 request/second + 5% of additional requests per service.
- Custom rules: per-service, per-resource, per-HTTP-method sampling rates.
- **Active tracing:** X-Ray SDK instrumented in code; sampling applies.
- **Passive tracing:** X-Ray reads traces from services that already emit them (ALB, API Gateway) — no SDK required.

**X-Ray Groups:** Filter traces by annotation or attribute, then view the service map for just that group. Use for per-team or per-service debugging isolation.

**X-Ray Analytics:** Compare trace groups by count, latency percentiles, error rates — helps identify which service in a complex call chain is responsible for increased latency.

---

### ADOT (AWS Distro for OpenTelemetry)

ADOT packages the OpenTelemetry Collector with AWS-specific components:

**Collector pipeline:**
```
Receivers → Processors → Exporters

Receivers: OTLP (gRPC/HTTP), Prometheus, StatsD, Jaeger
Processors: batch, memory_limiter, resource, transform
Exporters: AWS X-Ray, CloudWatch EMF, Prometheus Remote Write, OTLP
```

**Replace X-Ray SDK with OTEL SDK:**
- SDK is vendor-neutral (no AWS lock-in in application code)
- Same traces sent to X-Ray via the ADOT exporter
- Also send traces to other backends (Jaeger, Grafana Tempo) via same instrumentation

**Lambda ADOT layer:** Add the ADOT Lambda layer ARN to your function — instruments all Lambda invocations automatically without code changes.

---

### Centralised Log Aggregation — Multi-Account Pattern

```
Account A (payment-service):
  Lambda/ECS → CloudWatch Logs
  CloudWatch Logs → Subscription Filter → Kinesis Firehose (cross-account)

Account B (order-service):
  Lambda/ECS → CloudWatch Logs
  CloudWatch Logs → Subscription Filter → Kinesis Firehose (cross-account)

Logging Account:
  Kinesis Firehose → S3 (immutable bucket, Object Lock)
                  → OpenSearch (for real-time search via Kibana)
                  → Splunk/Datadog HTTP endpoint
```

**Setup for cross-account subscription filter:**
1. In the logging account: create Kinesis Firehose with a resource policy allowing source accounts to PutRecord.
2. In each source account: create a CloudWatch Logs subscription filter with the cross-account Kinesis Firehose ARN as the destination and an IAM role that Logs can assume to write to Firehose.

---

### Querying CloudTrail with Athena

Athena can query CloudTrail logs stored in S3 without ETL.

**Partition projection** (avoids scanning entire S3 path):
```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING, userIdentity STRUCT<...>, eventTime STRING,
  eventSource STRING, eventName STRING, awsRegion STRING,
  sourceIPAddress STRING, errorCode STRING, requestParameters STRING, ...
)
PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/123456789012/CloudTrail/'
TABLE PROPERTIES (
  'projection.enabled'='true',
  'projection.region.type'='enum',
  'projection.region.values'='eu-west-1,us-east-1',
  'projection.year.type'='date', ...
)
```

**Common investigation queries:**
```sql
-- Who called DeleteBucket in the last 7 days?
SELECT eventTime, userIdentity.arn, sourceIPAddress, requestParameters
FROM cloudtrail_logs
WHERE eventName = 'DeleteBucket'
  AND eventTime > date_format(now() - interval '7' day, '%Y-%m-%dT%H:%i:%sZ');

-- All root account API calls
SELECT eventTime, eventName, sourceIPAddress
FROM cloudtrail_logs
WHERE userIdentity.type = 'Root'
ORDER BY eventTime DESC;
```

---

## Phase 3+4 — Key Decision Framework

### "Auto Scaling — which policy?"

```
Metric is proportional to load (CPU, request count) and load is stable/predictable?
  → Target tracking (simplest; AWS manages scale-in/scale-out)

Load comes in unpredictable bursts that need different responses?
  → Step scaling (larger steps for larger breaches)

Load is predictable by time (8am-6pm weekdays)?
  → Scheduled scaling (pre-scale; combine with target tracking for residual variation)

Load follows weekly/daily patterns, need capacity ready before demand?
  → Predictive scaling
```

### "Monitoring — which CloudWatch feature?"

```
Need to extract a metric from a log entry count?
  → Metric filter on the log group

Need to find top-N contributors to a metric (worst IPs, slowest APIs)?
  → Contributor Insights

Need to alert on unexpected patterns without setting a threshold?
  → Anomaly detection alarm

Need to monitor a URL/API as if you're a user?
  → CloudWatch Synthetics canary

Need to publish business metrics from Lambda without PutMetricData API cost?
  → Embedded Metrics Format (EMF)

Need to stream metrics to Datadog/Splunk in near-real-time?
  → CloudWatch Metric Streams → Kinesis Firehose

Need to A/B test a feature with metrics comparison?
  → CloudWatch Evidently experiment
```

---

## Phase 3+4 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "EC2 health check won't replace a crashed app on a healthy instance" | Correct — use ELB health check for application-level replacement |
| "Instance Refresh vs CodeDeploy" | Instance Refresh = new AMI/LT version (OS-level). CodeDeploy = same OS, new app code |
| "Lifecycle hook default timeout" | 3600 seconds (1 hour). Set a shorter timeout for fast operations |
| "Warm pool — compute charge when stopped?" | NO charge for compute. YES for EBS volumes |
| "CloudWatch Alarm can invoke Lambda directly?" | NO. Alarm → SNS → Lambda subscription. Or Alarm → SSM OpsItem |
| "CloudTrail data events — default ON?" | NO. Must be enabled explicitly; costs per API event |
| "X-Ray passive vs active tracing" | Passive = read from services that emit traces (ALB, API GW). Active = SDK instrumented |
| "Contributor Insights uses Log or Metric data?" | Log data — it analyses log entries to find top contributors |
| "EMF — does it require PutMetricData permission?" | NO. Written to stdout/CloudWatch Logs; CloudWatch extracts metrics automatically |
| "Config rule periodic vs change-triggered" | Periodic = checked on schedule. Change-triggered = checked when resource configuration changes |
| "AWS Backup cross-account copy — what is it for?" | Protect backups against accidental or malicious deletion in the production account |
| "Can Route 53 health check detect app errors that return HTTP 200?" | YES — use string matching on the response body |
