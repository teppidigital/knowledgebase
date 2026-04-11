# DOP-C02 Phase 5–6 Deep Knowledge: Incident Response & Security/Compliance

> Covers Weeks 11–14 of the DOP-C02 learning path.  
> Domain 5: Incident and Event Response | Domain 6: Security and Compliance

---

## Week 11 — EventBridge & Automated Remediation

### EventBridge Event Buses

| Bus Type | Description | Cross-Account | Use Case |
|---|---|---|---|
| Default | Per-account, catches AWS service events | No (resource-based policy needed) | Auto-remediation from AWS Health, Config |
| Custom | Your own events via `PutEvents` | Yes (with resource policy) | Microservice events, decoupled notifications |
| Partner | SaaS events (Datadog, PagerDuty, Zendesk) | Pre-configured | Third-party pipeline triggers |

**Resource policy to allow cross-account send:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
    "Action": "events:PutEvents",
    "Resource": "arn:aws:events:us-east-1:444455556666:event-bus/central-bus"
  }]
}
```

---

### EventBridge Rule Event Pattern Matching

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated", "stopped"],
    "instance-id": [{ "prefix": "i-" }]
  }
}
```

| Pattern Operator | Example | Match |
|---|---|---|
| Exact match | `"state": ["running"]` | `state == "running"` |
| Prefix | `{ "prefix": "prod-" }` | starts with `prod-` |
| `anything-but` | `{ "anything-but": ["FAILED"] }` | any value except FAILED |
| `exists` | `{ "exists": true }` | field must be present |
| `exists: false` | `{ "exists": false }` | field must be absent |
| Numeric range | `{ "numeric": [">=", 100, "<", 200] }` | 100 ≤ value < 200 |
| IP prefix | `{ "cidr": "10.0.0.0/8" }` | IP in CIDR block |

**All EventBridge Rule Target Types (exam common):**
- Lambda, Step Functions, SQS, SNS, Kinesis Firehose/Streams
- CodeBuild, CodePipeline, ECS Task
- API Gateway, API Destination (external HTTP endpoint with auth)
- SSM Automation, SSM Run Command
- EC2 `RebootInstances`, `StopInstances`, `TerminateInstances`, `CreateSnapshot`

---

### EventBridge Pipes

Fully managed point-to-point integrations with optional filtering and enrichment.

```
Source → Filter → Enrichment (optional) → Target
```

| Stage | Options |
|---|---|
| Source | SQS, Kinesis, DynamoDB Streams, MSK, MQ, Kafka |
| Filter | JSON pattern (same syntax as EventBridge rules) |
| Enrichment | Lambda, Step Functions, API Gateway, API Destination |
| Target | Same as EventBridge targets |

**Exam use case:** Replace a complex Lambda fan-out function with Pipes when the pattern is source → transform → single target.

---

### EventBridge Scheduler

Replaces CloudWatch Events scheduled rules + cron expressions.

| Feature | EventBridge Scheduler | CloudWatch Events (legacy) |
|---|---|---|
| One-time schedules | Yes (`at()`) | No |
| Recurring (cron) | Yes (`cron()` / `rate()`) | Yes |
| Flexible time window | Yes (± N minutes) | No |
| Timezone support | Yes (60+ zones) | No (UTC only) |
| Scale | Millions of schedules | Limited |

```
cron(0 9 ? * MON-FRI *)   # 9 AM UTC Mon-Fri
rate(1 hour)               # every hour
at(2025-12-31T23:59:00)   # one-time
```

**Flexible time window:** allows schedule to fire within a ± window to spread load (e.g., ±15 min so not all tasks hit at exactly midnight).

---

### AWS Health Events & Automated Response

AWS Health publishes two event categories to the default event bus:

| Category | Example | How to Use |
|---|---|---|
| Service events | `AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED` | Auto-replace instances via ASG |
| Account events | `AWS_SUPPORT_CASE_CREATED` | Notify on-call |
| Operational notifications | Maintenance windows | Suppress alarms, pre-warm |

**Standard auto-replacement pattern for retiring instances:**

```
AWS Health event → EventBridge rule (detail-type: "AWS Health Event")
  → Lambda → TerminateInstances(instance-id from event)
  → ASG launches replacement (if in ASG)
```

Alternatively:
```
EventBridge → SSM Automation (AWS-RestartEC2Instance)
```

**Event pattern for retirement:**
```json
{
  "source": ["aws.health"],
  "detail-type": ["AWS Health Event"],
  "detail": {
    "service": ["EC2"],
    "eventTypeCode": ["AWS_EC2_INSTANCE_RETIREMENT_SCHEDULED"]
  }
}
```

---

### Automated Remediation Patterns

#### Pattern 1: Root IAM Activity Auto-Alert
```
CloudTrail → EventBridge (source: aws.iam, userIdentity.type: Root)
  → SNS → PagerDuty + Lambda → disable root access keys
```

#### Pattern 2: Security Group Port-22 Auto-Revert
```
Config Rule (restricted-ssh) fires NON_COMPLIANT
  → SSM Automation (AWS-DisablePublicAccessForSecurityGroup)
    → Reverts SG to compliant state
    → Creates OpsItem with evidence
```

Full Config + SSM Automation remediation setup:
```yaml
# config-remediation.yaml (CloudFormation)
RemediationConfig:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: restricted-ssh
    TargetType: SSM_DOCUMENT
    TargetId: AWS-DisablePublicAccessForSecurityGroup
    Parameters:
      GroupId:
        ResourceValue:
          Value: RESOURCE_ID
    Automatic: true
    MaximumAutomaticAttempts: 3
    RetryAttemptSeconds: 60
```

#### Pattern 3: EC2 CPU Spike Diagnostic + Remediation
```
CloudWatch Alarm (CPU > 90% for 5 min)
  → SNS → Lambda:
      1. ssm:SendCommand (collect diagnostics: ps aux, netstat)
      2. ssm:PutOpsItem (create incident with diagnostic output)
      3. Optional: ASG scale-out trigger via PutMetricData
```

#### Pattern 4: GuardDuty Finding → Pipeline Block
```
GuardDuty CryptoCurrency:EC2/BitcoinTool.B finding
  → EventBridge
  → Lambda:
      1. codepipeline:StopPipelineExecution (any running pipelines using that instance)
      2. ec2:CreateSnapshot (forensic copy)
      3. ec2:StopInstances (isolate)
      4. sns:Publish (security team notification)
```

---

### Step Functions Orchestrated Incident Remediation

Use Step Functions when remediation requires human approval or multi-step coordination.

**WaitForTaskToken pattern (human approval gate):**
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789/approval-queue",
    "MessageBody": {
      "TaskToken.$": "$$.Task.Token",
      "IncidentId.$": "$.incidentId",
      "ProposedAction.$": "$.action"
    }
  },
  "TimeoutSeconds": 86400,
  "HeartbeatSeconds": 3600
}
```

The approver sends the token back via:
```bash
aws stepfunctions send-task-success \
  --task-token "$TOKEN" \
  --task-output '{"approved": true}'
```

**Catch and Retry pattern for transient failures:**
```json
{
  "Type": "Task",
  "Resource": "arn:aws:lambda:us-east-1:123:function:remediate",
  "Retry": [{
    "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException"],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2
  }],
  "Catch": [{
    "ErrorEquals": ["States.ALL"],
    "Next": "NotifyFailure",
    "ResultPath": "$.error"
  }]
}
```

---

### OpsCenter Integration

| Source | How OpsItems Are Created |
|---|---|
| EventBridge rule | target: `aws:ssm:createOpsItem` |
| Config remediation | automatically creates on NON_COMPLIANT |
| Systems Manager Explorer | aggregates across accounts |
| CloudWatch Alarm | SNS → Lambda → CreateOpsItem |

**Deduplication:** Use `OperationalData` key `/aws/dedup` with a JSON fingerprint — SSM deduplicates OpsItems with the same source and dedup string within 24 hours.

**OpsItem to runbook linkage:**
- Associate `RelatedOpsItems` with runbook automation document ARN
- Use `Associations` to link SSM Automation or CloudFormation stacks as runbooks

---

## Week 12 — Fault Injection Service & SRE/SLOs

### AWS Fault Injection Service (FIS)

FIS is the managed chaos engineering service. Key concepts:

| Concept | Description |
|---|---|
| Experiment Template | Defines actions, targets, stop conditions, and IAM role |
| Action | A specific fault injected (e.g., stop EC2 instances) |
| Target | Resources to affect (by tag, ARN, or resource type filter) |
| Stop Condition | CloudWatch Alarm that auto-stops the experiment if triggered |
| Safety Guards | Stop conditions + experiment duration caps |

**Common FIS fault actions:**

| Action | Target | What It Does |
|---|---|---|
| `aws:ec2:stop-instances` | EC2 instances | Stops instances (tests ASG replacement) |
| `aws:ec2:terminate-instances` | EC2 instances | Terminates (harsher than stop) |
| `aws:ec2:cpu-stress` | EC2 instances | Injects CPU load via SSM |
| `aws:ec2:network-latency` | EC2 instances | Adds network latency via TC |
| `aws:ec2:network-packet-loss` | EC2 instances | Drops packets |
| `aws:ecs:drain-container-instances` | ECS container instances | Drains, tests ECS rescheduling |
| `aws:ecs:stop-task` | ECS tasks | Stops tasks, tests restart |
| `aws:rds:failover-db-cluster` | Aurora cluster | Forces failover, tests client reconnect |
| `aws:fis:inject-api-internal-error` | AWS APIs | Simulates throttling or 5xx on service calls |

**Experiment template (CloudFormation snippet):**
```yaml
Resources:
  FISExperiment:
    Type: AWS::FIS::ExperimentTemplate
    Properties:
      Description: Stop 30% of prod EC2 instances
      Targets:
        targetEC2:
          ResourceType: aws:ec2:instance
          ResourceTags:
            Environment: production
          SelectionMode: PERCENT(30)
      Actions:
        stopInstances:
          ActionId: aws:ec2:stop-instances
          Targets:
            Instances: targetEC2
          StartAfter: []
      StopConditions:
        - Source: aws:cloudwatch:alarm
          Value: !Sub "arn:aws:cloudwatch:${AWS::Region}:${AWS::AccountId}:alarm:FIS-Stop-Alarm"
      RoleArn: !GetAtt FISRole.Arn
      Tags:
        Experiment: ec2-stop-test
```

---

### Chaos Engineering Process Mapped to FIS

```
1. Define Steady State (CloudWatch dashboard baseline — latency p99, error rate, throughput)
2. Hypothesise (e.g., "stopping 30% of instances will not raise error rate above 1%")
3. Inject Fault (FIS experiment with stop condition = alarm at 1% error rate)
4. Observe (CloudWatch metrics, X-Ray traces, Synthetics canary)
5. Conclude (did the system stay within bounds? Document gap vs hypothesis)
6. Fix and Repeat
```

**Key FIS exam points:**
- Stop conditions are CloudWatch Alarms — set to ALARM state means STOP experiment
- IAM role for FIS needs `ec2:StopInstances`, `ssm:SendCommand`, etc. per action type
- `SelectionMode: ALL` vs `PERCENT(N)` vs `COUNT(N)` controls blast radius
- Always start with non-production; use tags to isolate targets

---

### SLI / SLO / SLA / Error Budget

| Term | Definition | Example |
|---|---|---|
| SLI | Service Level Indicator — what you measure | 99th percentile latency of /api/checkout |
| SLO | Service Level Objective — your target for the SLI | Latency < 300ms 99.9% of requests over 30 days |
| SLA | Service Level Agreement — contractual obligation, usually lower than SLO | 99.5% availability per contract |
| Error Budget | `1 - SLO target` expressed as allowed bad events | 0.1% of requests can fail = ~43 min downtime/month |

**Error budget burn rate alarm pattern (Google SRE model):**

Two alarms needed:
1. **Fast burn** (page immediately): 14× burn rate over 1 hour AND 5 min
2. **Slow burn** (ticket): 6× burn rate over 6 hours AND 30 min

```
Fast burn:
  CW Alarm: metric = error_rate / SLO_error_rate
  Threshold: > 14 (i.e., burning budget 14x faster than allowed)
  EvaluationPeriods: 2 
  DatapointsToAlarm: 2
  Period: 3600 (1 hour window)
  
  + short window confirmation:
  Period: 300 (5 min window), same threshold
  
  Both must be ALARM → Composite Alarm → PagerDuty

Slow burn:
  Period: 21600 (6 hour window), threshold > 6
  Period: 1800 (30 min window), same threshold
  Both → Composite Alarm → Jira ticket via Lambda
```

**CloudWatch metric math for error rate:**
```
error_rate = errors / (errors + successes)
burn_rate  = error_rate / (1 - SLO_target)
```

---

## Week 13 — Secrets Manager, KMS & Pipeline Security

### Secrets Manager Rotation Deep Dive

**Two rotation strategies:**

| Strategy | When to Use | Mechanism |
|---|---|---|
| Single-user | Dev/test, non-prod, service accounts | Rotate password, update secret, test |
| Alternating-user | Prod databases — no downtime | Two users alternate; old user kept valid during rotation |

**Lambda rotation function — 4 mandatory steps:**

```python
def lambda_handler(event, context):
    step = event['Step']
    secret_id = event['SecretId']
    token = event['ClientRequestToken']
    
    if step == 'createSecret':
        # Generate new credentials, store as AWSPENDING version
        create_new_credentials(secret_id, token)
    
    elif step == 'setSecret':
        # Apply new credentials on the target (e.g., ALTER USER in RDS)
        set_credentials_on_target(secret_id, token)
    
    elif step == 'testSecret':
        # Verify AWSPENDING version can authenticate to target
        test_credentials(secret_id, token)
    
    elif step == 'finishSecret':
        # Move AWSPENDING to AWSCURRENT, deprecate old version
        finalize_rotation(secret_id, token)
```

**Version staging labels:**
- `AWSCURRENT` — active version used by applications
- `AWSPENDING` — new version being rotated in
- `AWSPREVIOUS` — last AWSCURRENT (kept for rollback)

**Access pattern in applications — always use staging label, not version ID:**
```python
response = secrets_client.get_secret_value(
    SecretId='prod/db/password',
    VersionStage='AWSCURRENT'   # fetch current, not pinned version
)
```

**Rotation schedule:**
```json
{
  "RotationRules": {
    "AutomaticallyAfterDays": 30,
    "Duration": "2h",
    "ScheduleExpression": "cron(0 2 1 * ? *)"
  }
}
```

---

### KMS Deep Dive

**Key type comparison:**

| Type | Who Manages Key Material | Cross-Region? | Rotation | Use Case |
|---|---|---|---|---|
| AWS managed (`aws/s3`) | AWS | No (regional) | Auto (3 yr) | Default S3/RDS encryption |
| Customer managed CMK | Customer in KMS | No (per-region) | Optional (1 yr) | Custom control, cross-account |
| Customer managed with imported material | Customer (you provide) | Manually (reimport) | No (you manage) | Regulatory — bring your own |
| Multi-Region key | Customer | Yes (replicate) | Optional | DynamoDB global, cross-region decrypt |

**Key policy vs IAM policy precedence:**
- Both must allow access — neither alone is sufficient for cross-account
- Key policy with `"Principal": {"AWS": "*"}` + `"Condition": {"StringEquals": {"kms:CallerAccount": "123456789"}}` delegates all control to IAM policies in that account
- Without the `kms:CallerAccount` delegation statement, IAM policies alone cannot grant KMS access

**`kms:ViaService` condition key — exam favourite:**
```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789:role/AppRole" },
  "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "s3.us-east-1.amazonaws.com"
    }
  }
}
```
This allows the role to use the key **only** when the API call comes from S3, not directly. Prevents direct decryption by the application principal.

**Key rotation:**
- Customer managed keys: optional annual auto-rotation (KMS retains old backing material)
- Rotating does NOT change the key ID — alias and all references remain valid
- Imported key material: you control rotation by reimporting new material
- Deletion: 7–30 day waiting period (`ScheduleKeyDeletion`)
- Use `CancelKeyDeletion` within window to abort

**Envelope encryption (CloudWatch → KMS):**
```
1. KMS generates DEK (data encryption key) via GenerateDataKey
2. Plaintext DEK encrypts your data (in-memory)
3. Encrypted DEK stored alongside ciphertext
4. Plaintext DEK immediately discarded
5. To decrypt: KMS Decrypt(encrypted DEK) → plaintext DEK → decrypt data
```

---

### Pipeline Security Gates

#### SAST in CodeBuild
```yaml
phases:
  build:
    commands:
      # Semgrep SAST scan
      - semgrep --config=auto --json ./src > sast-results.json
      # Fail build on high/critical findings
      - |
        HIGH=$(jq '[.results[] | select(.extra.severity == "ERROR")] | length' sast-results.json)
        if [ "$HIGH" -gt 0 ]; then
          echo "SAST found $HIGH high/critical findings. Failing build."
          exit 1
        fi
```

#### SCA (Software Composition Analysis) in CodeBuild
```yaml
  post_build:
    commands:
      # OWASP Dependency Check
      - dependency-check --project MyApp --scan ./src --format JSON --out reports/
      - python3 check-cvss.py --threshold 7.0   # fail on CVSS >= 7.0
```

#### ECR Inspector Gate
```yaml
  post_build:
    commands:
      - ECR_URI=123456789.dkr.ecr.us-east-1.amazonaws.com/myapp
      - docker push $ECR_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      # Wait for Inspector scan (triggered automatically on push)
      - sleep 30
      - |
        CRITICAL=$(aws ecr describe-image-scan-findings \
          --repository-name myapp \
          --image-id imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION \
          --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
          --output text)
        if [ "$CRITICAL" != "None" ] && [ "$CRITICAL" -gt 0 ]; then
          echo "Found $CRITICAL critical vulnerabilities. Failing pipeline."
          exit 1
        fi
```

#### CodeBuild Environment Security Best Practices

| Requirement | Implementation |
|---|---|
| No secrets in buildspec | Use Parameter Store `{{resolve:ssm:/path/to/secret}}` or Secrets Manager env var |
| Least-privilege CodeBuild role | Separate IAM role per project; no `*` actions |
| VPC builds | Required to access RDS, internal endpoints (add NAT GW for internet) |
| Ephemeral credentials | Use IAM role (not access keys) in environment |
| Audit | CloudTrail + EventBridge rule on `codebuild:StartBuild` |

---

### GuardDuty Finding → Pipeline Remediation

```
EventBridge rule:
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{ "numeric": [">=", 7] }],
    "type": [{ "prefix": "CryptoCurrency:" }]
  }
}

→ Step Functions (orchestrated response):
   1. CreateSnapshot(instance-id)           # forensic copy
   2. StopInstances(instance-id)            # isolate
   3. RevokeSecurityGroupIngress(sg-id)     # cut network
   4. WaitForTaskToken → SNS approval       # human review
   5. On approval: terminate + audit report
   6. On rejection: restore + post-mortem
```

---

## Week 14 — Config, Security Hub, Compliance & IAM Hardening

### AWS Config Rules — Managed Rule Reference

| Rule Name | What It Checks | Auto-Remediation? |
|---|---|---|
| `encrypted-volumes` | EBS volumes encrypted | Yes (SSM) |
| `s3-bucket-public-read-prohibited` | S3 public ACLs blocked | Yes |
| `restricted-ssh` | SG port 22 open to 0.0.0.0/0 | Yes |
| `iam-password-policy` | Password policy meets requirements | No |
| `mfa-enabled-for-iam-console-access` | MFA on console users | No |
| `access-keys-rotated` | Access keys older than N days | No |
| `cloudtrail-enabled` | CloudTrail active in region | Yes (Lambda) |
| `rds-instance-public-access-check` | RDS not publicly accessible | No |
| `ec2-instances-in-vpc` | All EC2 in VPC | No |
| `lambda-function-public-access-prohibited` | Lambda resource policy not public | Yes |

**Custom Config Rule (Lambda-based):**
```python
def handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    config = boto3.client('config')
    
    # Get configuration item
    config_item = invoking_event['configurationItem']
    resource_type = config_item['resourceType']
    
    compliance = evaluate_compliance(config_item)
    
    config.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': resource_type,
            'ComplianceResourceId': config_item['resourceId'],
            'ComplianceType': compliance,  # 'COMPLIANT' or 'NON_COMPLIANT'
            'OrderingTimestamp': datetime.now()
        }],
        ResultToken=event['resultToken']
    )
```

**Config rule trigger types:**
- `Configuration changes` — fires when resource config changes
- `Periodic` — fires every 1/3/6/12/24 hours regardless of changes
- `Hybrid` — both

---

### Config Conformance Packs

Pre-packaged sets of Config rules deployed together.

| Conformance Pack | Standards Covered |
|---|---|
| `AWS-Control-Tower-Detective-Guardrails` | Landing Zone compliance |
| `Operational-Best-Practices-for-PCI-DSS` | PCI-DSS 3.2.1 |
| `Operational-Best-Practices-for-HIPAA-Security` | HIPAA Security Rule |
| `Operational-Best-Practices-for-CIS` | CIS AWS Foundations Benchmark |
| `Operational-Best-Practices-for-NIST-800-53` | NIST 800-53 |

**Org-wide deployment via StackSets:**
```bash
aws configservice put-organization-conformance-pack \
  --organization-conformance-pack-name "org-pci-compliance" \
  --template-s3-uri "s3://my-conformance-packs/pci-dss.yaml" \
  --excluded-accounts "111122223333"  # exclude management account
```

---

### CloudFormation Guard (`cfn-guard`)

Policy-as-code tool for validating CloudFormation templates before deployment.

**Guard rule syntax (`.guard` file):**
```
# s3-encryption.guard
rule s3_bucket_encrypted {
  AWS::S3::Bucket {
    Properties {
      BucketEncryption exists
      BucketEncryption.ServerSideEncryptionConfiguration[*].ServerSideEncryptionByDefault.SSEAlgorithm == "aws:kms"
    }
  }
}

rule no_public_s3 {
  AWS::S3::Bucket {
    Properties.PublicAccessBlockConfiguration {
      BlockPublicAcls == true
      BlockPublicPolicy == true
      IgnorePublicAcls == true
      RestrictPublicBuckets == true
    }
  }
}
```

**Validate in CodeBuild:**
```yaml
phases:
  install:
    commands:
      - pip install cloudformation-guard
  build:
    commands:
      - cfn-guard validate --rules ./guard-rules/ --data ./templates/ --show-summary all
      - |
        if [ $? -ne 0 ]; then
          echo "CFN Guard policy violations found. Failing pipeline."
          exit 1
        fi
```

**CloudFormation Hooks (proactive compliance):**
```yaml
# hook-handler.yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  MyHook:
    Type: AWS::CloudFormation::HookVersion
    Properties:
      TypeName: MyOrg::Security::S3Encryption
      SchemaHandlerPackage: s3://my-bucket/hook.zip

  HookDefault:
    Type: AWS::CloudFormation::HookDefaultVersion
    Properties:
      TypeName: MyOrg::Security::S3Encryption

  HookTypeConfig:
    Type: AWS::CloudFormation::HookTypeConfig
    Properties:
      TypeName: MyOrg::Security::S3Encryption
      Configuration: '{"CloudFormationConfiguration": {"HookConfiguration": {"TargetStacks": "ALL", "FailureMode": "FAIL", "Properties": {}}}}'
```

Hooks fire `PRE_CREATE`, `PRE_UPDATE`, `PRE_DELETE` before CloudFormation applies changes. `FailureMode: FAIL` blocks the stack operation.

---

### Security Hub

**Architecture:**
```
AWS Config Rules      →
GuardDuty Findings    →  Security Hub  →  Findings (ASFF format)
Inspector findings    →  (aggregates)     → EventBridge → Lambda → Remediation
IAM Access Analyzer   →
3P SIEM integrations  →                  → S3 (via Firehose) → Athena
```

**ASFF (Amazon Security Finding Format) — key fields:**
- `Severity.Label`: INFORMATIONAL / LOW / MEDIUM / HIGH / CRITICAL
- `WorkflowStatus`: NEW / NOTIFIED / RESOLVED / SUPPRESSED
- `Compliance.Status`: PASSED / FAILED / WARNING / NOT_AVAILABLE
- `ProductArn`: source service ARN

**Custom action → EventBridge → remediation:**
```json
EventBridge rule:
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Custom Action"],
  "detail": {
    "actionName": ["Remediate-S3-Public-Access"]
  }
}
→ Lambda:
  1. Extract S3 bucket name from finding ProductFields
  2. Put public access block on bucket
  3. Update finding WorkflowStatus to RESOLVED
```

**Cross-account Security Hub:**
- Designate delegated admin account (via AWS Organizations)
- Member accounts auto-enroll (or manually invite)
- All findings centralised in admin account
- Admin can suppress/archive from central view

**CSPM (Cloud Security Posture Management):** Security Hub's Foundational Security Best Practices standard acts as a continuously evaluated CSPM across your accounts.

---

### Audit Manager

Collects evidence for compliance audits (SOC 2, PCI-DSS, GDPR, FedRAMP).

| Component | Description |
|---|---|
| Framework | Collection of control sets (maps to compliance standard) |
| Control Set | Group of related controls (e.g., "Access Control") |
| Control | Individual requirement (e.g., "MFA must be enabled") |
| Evidence | Actual proof — Config snapshots, CloudTrail events, manual |
| Assessment | Running evaluation of a framework against your AWS account |

**Evidence types:**
- **Automated (Config-based):** Config rule compliance snapshots collected daily
- **Automated (CloudTrail-based):** API calls matching control actions
- **Manual:** PDF attachments uploaded by auditors

**Exam point:** Audit Manager does NOT remediate — it collects evidence. Remediation is done by Config + SSM or Lambda.

---

### SCPs as Preventive Controls

**SCP: Deny disabling CloudTrail**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyCloudTrailDisable",
    "Effect": "Deny",
    "Action": [
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "cloudtrail:UpdateTrail",
      "cloudtrail:PutEventSelectors"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalArn": "arn:aws:iam::*:role/SecurityAuditRole"
      }
    }
  }]
}
```

**SCP: Deny leaving the organisation**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyLeaveOrg",
    "Effect": "Deny",
    "Action": "organizations:LeaveOrganization",
    "Resource": "*"
  }]
}
```

**SCP: Restrict to approved regions**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonApprovedRegions",
    "Effect": "Deny",
    "NotAction": [
      "iam:*", "organizations:*", "support:*",
      "sts:*", "cloudfront:*", "route53:*", "budgets:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
      }
    }
  }]
}
```
Note: `NotAction` (not `Action: Deny`) allows global services (IAM, Route 53, CloudFront) to work regardless of region.

**SCP: Require IMDSv2 at launch**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireIMDSv2",
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "StringNotEquals": {
        "ec2:MetadataHttpTokens": "required"
      }
    }
  }]
}
```

---

### IAM Access Analyzer

| Feature | Description |
|---|---|
| External access analysis | Finds resources (S3, KMS, Lambda, IAM roles) accessible outside the account/org |
| Unused access analysis | Finds unused roles, access keys, permissions (90-day window) |
| Custom policy checks | Validates new policies against your standards before attaching |
| Path analysis | Shows which policy chains allow access to a resource |

**External access finding example:**
```json
{
  "resourceType": "AWS::S3::Bucket",
  "resource": "arn:aws:s3:::my-bucket",
  "principal": {"AWS": "*"},
  "action": ["s3:GetObject"],
  "condition": {},
  "status": "ACTIVE",
  "findingType": "ExternalAccess"
}
```

**Pipeline integration — validate policy before attaching:**
```bash
# In CodeBuild — validate IAM policy before applying
aws accessanalyzer validate-policy \
  --policy-document file://new-policy.json \
  --policy-type IDENTITY_POLICY \
  --query 'findings[?findingType==`ERROR` || findingType==`SECURITY_WARNING`]' \
  --output text | grep -q . && exit 1 || echo "Policy is valid"
```

**Unused access analysis — enable in CodePipeline audit stage:**
```bash
aws accessanalyzer list-findings \
  --analyzer-name org-analyzer \
  --filter '{"findingType": {"eq": ["UnusedPermission", "UnusedIAMRole"]}}' \
  --query 'findings[?status==`ACTIVE`]'
```

---

### Inspector v2 — Pipeline Enforcement

```
ECR image push → Inspector v2 automatic scan
  → EventBridge finding event
  → Lambda: evaluate severity
       CRITICAL/HIGH > threshold → 
         1. codepipeline:StopPipelineExecution
         2. ecr:BatchDeleteImage (prevent deployment)
         3. sns:Publish (notify team)
       ELSE → continue pipeline
```

**Inspector v2 key differences from v1:**

| Feature | Inspector v1 | Inspector v2 |
|---|---|---|
| EC2 scanning | Agent-based, scheduled | Agentless (SSM) + continuous |
| ECR scanning | No | Yes (on push + continuously) |
| Lambda scanning | No | Yes (function and layer packages) |
| SBOM export | No | Yes (CycloneDX/SPDX via API) |
| Network path | Limited | Package vulnerabilities only |
| Finding integration | Inspector only | Security Hub auto-aggregation |

---

## Decision Frameworks

### Which Remediation Driver to Use?

```
Is the trigger a Config rule violation?
  YES → Config Remediation (SSM Automation) — automatic, tracks compliance
  
Is the trigger a security finding (GuardDuty/Security Hub)?
  YES → EventBridge → Lambda/Step Functions (more flexible logic)
  
Does remediation require human approval?
  YES → Step Functions with WaitForTaskToken → API Gateway callback or SQS
  
Is it a scheduled compliance check?
  YES → EventBridge Scheduler → Lambda → generate compliance report
  
Is it a real-time infrastructure event (Health/State change)?
  YES → EventBridge rule on default bus → targeted action

Is it chaos/resilience testing?
  YES → FIS experiment template with CloudWatch stop condition
```

### Rotation vs Expiry vs Revoke

| Scenario | Action | Service |
|---|---|---|
| Rotate DB password every 30 days | Automated rotation Lambda | Secrets Manager |
| Rotate API key on demand | Manual rotation + update refs | Secrets Manager |
| IAM access key > 90 days | Detect via Config rule → notify/disable | Config + Lambda |
| KMS key annually | Enable auto-rotation on CMK | KMS |
| SSL/TLS cert expiry | ACM auto-renews (DNS validation) | ACM |
| Compromised secret | Disable current version → trigger rotation | Secrets Manager |

### KMS Key Policy Decision

```
Need to grant another account access to KMS key?
  → Add cross-account principal to key policy + attach IAM policy in target account

Need to restrict CMK usage to specific AWS service only?
  → Use kms:ViaService condition in key policy

Need to allow key usage without explicit key policy entry?
  → Add kms:CallerAccount delegation to key policy (enables IAM-only control)

Need compliance evidence of who decrypted data?
  → Enable CloudTrail + filter on kms:Decrypt events for that KeyId
```

---

## Exam Trap Summary Table

| Trap | Incorrect Assumption | Correct Answer |
|---|---|---|
| EventBridge rule max targets | One per rule | Up to 5 targets per rule |
| EventBridge rule → Lambda directly | Always direct | Yes, Lambda is a direct target |
| Config rules auto-remediate by default | Yes | Must explicitly configure `RemediationConfiguration` with `Automatic: true` |
| FIS stop condition fires automatically | No manual trigger needed | Stop condition fires when CW Alarm enters ALARM state (not you) |
| Secrets Manager always rotates in place | Single-user only | Alternating-user creates 2nd account on DB |
| KMS rotation changes key ID | Yes | No — same key ID, new backing material; old material retained |
| KMS imported key can be auto-rotated | Yes | No — you must reimport new material manually |
| `kms:ViaService` allows any caller | Yes | Only API calls originating from the named service |
| cfn-guard runs at deploy time | No — it's manual | Via Hooks it runs at deploy; standalone it's a pre-deploy gate |
| Security Hub remediates findings | Yes | No — it aggregates; remediation via EventBridge + Lambda/SSM |
| Audit Manager remediates | Yes | No — evidence collection only; no auto-remediation |
| SCPs apply to management account | Yes | No — SCPs never restrict the management account |
| SCP `NotAction` denies those actions | Yes (confusing) | `NotAction` means "Deny everything EXCEPT these actions" |
| IAM Access Analyzer finds internal misconfigs | Yes | External access only (unless using unused access analyzer) |
| FIS experiment runs indefinitely | Yes | Always time-limited; stop conditions also halt it |
| GuardDuty finding automatically blocks traffic | Yes | Findings are informational only; you must act on them |
| Config rule runs immediately on creation | Yes if Change-triggered | Periodic rules only run on their schedule even after creation |
| Inspector v2 requires agent on EC2 | Yes (like v1) | Uses AWS Systems Manager (agentless) |
| EventBridge can directly invoke DynamoDB | Yes | No — use Lambda or Pipes as intermediary |
