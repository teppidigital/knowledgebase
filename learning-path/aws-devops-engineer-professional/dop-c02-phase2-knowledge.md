# DOP-C02 Phase 2 — Deep Knowledge Reference
## Domain 2: Configuration Management and IaC (Weeks 5–6)

> Detailed technical knowledge for every bullet in Phase 2 of the DOP-C02 study plan.
> Cross-reference: [`aws-devops-engineer-professional.md`](./aws-devops-engineer-professional.md)

---

## Week 5 — CloudFormation Mastery

### Template Anatomy

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Concise description of this template"

Metadata:
  AWS::CloudFormation::Interface:    # Groups parameters in console
    ParameterGroups:
      - Label: {default: "Database Config"}
        Parameters: [DBInstanceClass, DBName]

Parameters:
  Environment:
    Type: String
    AllowedValues: [prod, staging, dev]
    Default: dev
  DBInstanceClass:
    Type: String
    Default: db.t3.micro

Mappings:
  EnvironmentConfig:
    prod:
      InstanceType: m5.large
      MultiAZ: true
    dev:
      InstanceType: t3.micro
      MultiAZ: false

Conditions:
  IsProduction: !Equals [!Ref Environment, prod]
  CreateReplica: !And
    - !Condition IsProduction
    - !Not [!Equals [!Ref DBInstanceClass, db.t3.micro]]

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain      # Retain | Delete | Snapshot
    UpdateReplacePolicy: Retain
    Properties:
      BucketName: !Sub "${Environment}-my-bucket-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled

Outputs:
  BucketName:
    Description: "Name of the created bucket"
    Value: !Ref MyBucket
    Export:
      Name: !Sub "${AWS::StackName}-BucketName"   # cross-stack reference
```

---

### Intrinsic Functions — Reference

| Function | Short form | Usage |
|----------|-----------|-------|
| `Fn::Ref` | `!Ref` | Reference a parameter (returns value) or resource (returns its ID/ARN) |
| `Fn::GetAtt` | `!GetAtt` | Get an attribute of a resource: `!GetAtt MyBucket.Arn` |
| `Fn::Sub` | `!Sub` | String substitution: `!Sub "arn:aws:s3:::${BucketName}/*"` |
| `Fn::FindInMap` | `!FindInMap` | Look up value in Mappings section |
| `Fn::If` | `!If` | Conditional value: `!If [IsProduction, m5.large, t3.micro]` |
| `Fn::Select` | `!Select` | Select from list by index: `!Select [0, !GetAZs ""]` |
| `Fn::Split` | `!Split` | Split string by delimiter |
| `Fn::Join` | `!Join` | Join list into string: `!Join [",", [a, b, c]]` |
| `Fn::ImportValue` | `!ImportValue` | Import an Exported output from another stack |
| `Fn::Base64` | `!Base64` | Encode string to Base64 (used for UserData) |
| `Fn::Cidr` | `!Cidr` | Generate CIDR blocks from a parent CIDR |
| `Fn::Transform` | N/A | Apply a macro (e.g., `AWS::Include`, `AWS::Serverless::Transform`) |

**`!Sub` with variable map:**
```yaml
# Simple substitution
- !Sub "arn:aws:s3:::${BucketName}/*"

# With explicit variable map
- !Sub
  - "arn:${Partition}:s3:::${BucketName}/*"
  - Partition: !Ref AWS::Partition
    BucketName: !Ref MyBucket
```

---

### Change Sets

A Change Set previews what CloudFormation will DO before it does it. Always create a change set for production stacks before executing an update.

**Change types in a change set:**

| Type | Meaning |
|------|---------|
| `Add` | New resource being created |
| `Modify` | Existing resource being updated |
| `Remove` | Existing resource being deleted |

**Replacement column:**
- `True` — resource will be **deleted and re-created** (e.g., changing an RDS `DBName`)
- `False` — resource is updated in-place
- `Conditional` — depends on whether the new value differs from the old

**Exam trap:** If a Change Set shows `Replacement: True` for an RDS or EC2 instance, this means **data loss**. Use `DeletionPolicy: Retain` or a Stack Policy to prevent accidental execution.

**Import existing resources:**
```bash
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name import-bucket \
  --change-set-type IMPORT \
  --resources-to-import '[{"ResourceType":"AWS::S3::Bucket","LogicalResourceId":"MyBucket","ResourceIdentifier":{"BucketName":"my-existing-bucket"}}]' \
  --template-body file://template.yaml
```

---

### Stack Policies

Stack policies prevent accidental updates or replacements of critical resources. They are **not** IAM policies — they only apply during CloudFormation stack update operations.

```json
{
  "Statement": [{
    "Effect": "Deny",
    "Action": "Update:*",
    "Principal": "*",
    "Resource": "LogicalResourceId/ProductionDatabase"
  }, {
    "Effect": "Allow",
    "Action": "Update:*",
    "Principal": "*",
    "Resource": "*"
  }]
}
```

**Overriding a stack policy:** An administrator can pass a temporary override policy during a specific `update-stack` call. The override applies only for that update.

---

### Drift Detection

Drift occurs when someone modifies a CloudFormation-managed resource directly (e.g., via console, CLI, or SDK) — outside of CloudFormation.

**Drift detection:**
- Initiated manually (or via automation on a schedule).
- Compares actual resource configuration to the template.
- Results: `IN_SYNC`, `DRIFTED`, `NOT_CHECKED`, `DELETED`.

**Remediation options:**
1. Import the drifted resource back (for small changes to tracked properties).
2. Update the template to match the actual state and update the stack.
3. Revert the direct change manually, then mark as resolved.

**Exam pattern:** "Prevent drift" → Use SCP to deny direct modification of stack-managed resources, or use AWS Config rule `cloudformation-stack-drift-detection-check`.

---

### Nested Stacks vs StackSets vs Cross-Stack References

| Approach | What it does | When to use |
|----------|------------|-------------|
| **Nested stacks** | Child stacks created by a parent stack's template | DRY: reuse VPC, ECS, RDS patterns across environments in ONE account/region |
| **StackSets** | Deploy a single template to multiple accounts and/or regions simultaneously | Multi-account governance: deploy Config rules, IAM roles, CloudTrail org-wide |
| **Cross-stack references** | Export Outputs from one stack; Import in another | Share resources (VPC ID, Subnet IDs) between independent stacks in same account/region |

**StackSets — `SERVICE_MANAGED` vs `SELF_MANAGED`:**

| Mode | Admin account | Target accounts | Org integration |
|------|--------------|----------------|----------------|
| `SELF_MANAGED` | Manually designate | Manually list accounts | No |
| `SERVICE_MANAGED` | Management account or delegated admin | OUs or entire org | Yes — new accounts auto-enrolled |

**`SERVICE_MANAGED` advantages:** New accounts joining the OU automatically get the StackSet deployed.  Recommended for org-wide compliance (Config rules, IAM roles for SSO, CloudTrail).

**StackSet failure tolerance:**
- `MaxConcurrentAccounts`: how many accounts update simultaneously.
- `FailureToleranceCount` / `FailureTolerancePercentage`: how many accounts can fail before the operation is marked failed.

---

### Custom Resources

A Custom Resource runs a Lambda function (or SNS topic) as part of a CloudFormation stack create, update, or delete operation. Needed when CloudFormation doesn't natively support a resource or action.

**Flow:**
```
CloudFormation → sends event to Lambda (or SNS)
  Lambda runs custom logic
  Lambda calls cfn-response (sends SUCCESS or FAILED + data back to CloudFormation)
CloudFormation continues or fails the stack operation
```

**Lambda-backed custom resource:**
```python
import cfnresponse
import boto3

def handler(event, context):
    try:
        if event['RequestType'] == 'Create':
            # Do creation logic
            physical_id = "my-custom-resource-id"
            data = {"OutputKey": "OutputValue"}
            cfnresponse.send(event, context, cfnresponse.SUCCESS, data, physical_id)
        elif event['RequestType'] == 'Delete':
            # Do cleanup logic
            cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
        elif event['RequestType'] == 'Update':
            cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
    except Exception as e:
        cfnresponse.send(event, context, cfnresponse.FAILED, {"Error": str(e)})
```

**In the template:**
```yaml
MyCustomResource:
  Type: Custom::MyThing
  Properties:
    ServiceToken: !GetAtt CustomResourceLambda.Arn
    InputParam: !Ref SomeParameter
```

**`cfn-response` module:** Automatically imported when the Lambda runtime handles a CloudFormation custom resource event. Sends the pre-signed S3 URL response that CloudFormation is waiting for.

---

### `cfn-init` and `cfn-signal` — EC2 Bootstrap

**`cfn-init`** reads `AWS::CloudFormation::Init` metadata from the template and applies it to the EC2 instance:
- Install packages
- Create files
- Run commands
- Enable services

**`cfn-signal`** tells CloudFormation that the EC2 instance has finished bootstrapping (success or failure).

**Template pattern:**
```yaml
EC2Instance:
  Type: AWS::EC2::Instance
  CreationPolicy:
    ResourceSignal:
      Count: 1
      Timeout: PT10M     # ISO 8601: wait up to 10 minutes for signal
  Metadata:
    AWS::CloudFormation::Init:
      config:
        packages:
          yum:
            httpd: []
        services:
          sysvinit:
            httpd:
              enabled: true
              ensureRunning: true
  Properties:
    UserData:
      Fn::Base64: !Sub |
        #!/bin/bash
        yum install -y aws-cfn-bootstrap
        /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
        /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
```

**WAIT_CONDITION vs CreationPolicy:**
- `CreationPolicy` on the resource itself — preferred for EC2 and Auto Scaling groups.
- `WaitCondition` + `WaitConditionHandle` — legacy pattern; use for custom ordering across resources.

---

### CloudFormation Hooks (Proactive Compliance)

Hooks intercept CloudFormation create/update/delete operations BEFORE the resource is provisioned, allowing proactive validation.

**Use case:** Enforce that every S3 bucket has encryption enabled / every EC2 uses an approved AMI / every Lambda has a DLQ.

**Hook types:**
- `PRE_CREATE` — before resource creation
- `PRE_UPDATE` — before resource update
- `PRE_DELETE` — before resource deletion

**Hooks are registered at the CloudFormation registry** and apply to all stacks in the account/region (or configured target scope). They can be written in Python, Java, Go, or TypeScript.

---

### AWS CDK — Key Concepts

CDK generates CloudFormation templates programmatically using TypeScript, Python, Java, Go, or .NET.

**Construct levels:**
| Level | Description | Example |
|-------|-------------|---------|
| **L1 (Cfn)** | Direct CloudFormation resource mapping | `CfnBucket` (same props as CF template) |
| **L2** | Higher-level construct with sensible defaults | `s3.Bucket` (handles versioning, policies, grants) |
| **L3 (Patterns)** | Opinionated patterns combining multiple resources | `ecsPatterns.ApplicationLoadBalancedFargateService` |

**CDK workflow:**
```bash
cdk init app --language typescript    # create new CDK app
cdk synth                              # synthesise to CloudFormation JSON/YAML
cdk diff                               # compare deployed vs local
cdk deploy                             # deploy (calls CloudFormation)
cdk destroy                            # destroy the stack
```

**`cdk.json`:** Configuration file for the CDK app. Defines the app entry point and feature flags.

**CDK Aspects:** Apply a rule across the entire CDK app (like a middleware on all constructs). Use for tagging all resources, or enforcing that all S3 buckets have versioning.

---

## Week 6 — Systems Manager (SSM) Mastery

### SSM Agent and Managed Instances

The SSM Agent must be installed and running on EC2 instances. It is pre-installed on Amazon Linux 2, Amazon Linux 2023, Windows Server 2016+, Ubuntu 20.04+.

**Connectivity requirements (no SSM endpoint → outbound HTTPS):**
- Instance must be able to reach `ssm.{region}.amazonaws.com`, `ec2messages.{region}.amazonaws.com`, `ssmmessages.{region}.amazonaws.com`.
- For private instances: use VPC interface endpoints for SSM (`ssm`, `ssmmessages`, `ec2messages`).
- No port 22 or 3389 required — SSM is API-over-HTTPS.

**Instance profile requirements:**
- Attach `AmazonSSMManagedInstanceCore` managed policy to the instance IAM role.

**Hybrid activations:** Register on-premises servers or VMs in SSM as "managed instances". Creates an activation code + ID pair that the SSM Agent uses to register.

---

### Session Manager

| Feature | Detail |
|---------|--------|
| No open ports | Replaces port 22/3389 entirely |
| Session logging | All session commands and output logged to S3 and/or CloudWatch Logs |
| Port forwarding | Forward a remote port (e.g., RDS:5432) to localhost via SSM tunnel |
| Document sessions | Start a session using a specific SSM Document for structured access |
| IAM + tag conditions | Restrict session access to specific instances via tags or resource ARNs |

**Session Manager via AWS CLI:**
```bash
aws ssm start-session --target i-0123456789abcdef0

# Port forwarding (RDS example)
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.cluster.eu-west-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

---

### Run Command

Run Command executes a command or script on one or more managed instances WITHOUT SSH.

**Key document:** `AWS-RunShellScript` (Linux), `AWS-RunPowerShellScript` (Windows).

**Targeting:**
- Instance IDs: `--instance-ids i-abc i-def`
- Tags: `--targets Key=tag:Env,Values=prod`
- Resource groups: `--targets Key=resource-groups:Name,Values=ProdGroup`

**Rate controls:**
- `MaxConcurrency`: how many instances run at once (count or %)
- `MaxErrors`: stop execution after N failures (count or %)

**Output:**
- Console (limited to 2,500 characters)
- Full output to S3 bucket
- CloudWatch Logs

---

### Patch Manager

Patch Manager automates OS patching across a fleet with compliance reporting.

**Patch baseline:**
- Defines which patches are approved (auto-approved or manual approval).
- Criteria: severity (Critical, High, Medium, Low), classification (Security, Bugfix), age (auto-approve after N days).
- AWS-managed baselines exist (per OS); you can create custom ones.

**Patch groups:** Tag instances with `Patch Group = <group-name>`. Each patch group can be associated with a different patch baseline.

**Maintenance windows:**
- Define schedule (cron), duration, and allowed tasks.
- Register target (instance IDs or tags) and task (`AWS-RunPatchBaseline`).
- Maintenance window tasks run via Run Command or Automation.

**Patch compliance reporting:**
```
Patch Manager scans instances → sends compliance data to SSM Compliance
  → Config aggregates compliance data
  → Security Hub ingests Config findings
  → Dashboard: % of instances compliant per patch group
```

**`AWS-RunPatchBaseline` parameters:**
- `Operation: Scan` — assess what patches are missing (no install)
- `Operation: Install` — apply approved patches

---

### SSM Automation

Automation runs **multi-step operational runbooks** across AWS resources. Unlike Run Command (executes on instances), Automation can orchestrate AWS API calls.

**Document structure:**
```yaml
schemaVersion: "0.3"
description: "Stop and restart an EC2 instance"
assumeRole: "{{ AutomationAssumeRole }}"
parameters:
  InstanceId:
    type: String
  AutomationAssumeRole:
    type: String
mainSteps:
  - name: StopInstance
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: StopInstances
      InstanceIds:
        - "{{ InstanceId }}"
  - name: WaitForStopped
    action: aws:waitForAwsResourceProperty
    inputs:
      Service: ec2
      Api: DescribeInstanceStatus
      InstanceIds:
        - "{{ InstanceId }}"
      PropertySelector: "$.InstanceStatuses[0].InstanceState.Name"
      DesiredValues:
        - stopped
  - name: StartInstance
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: StartInstances
      InstanceIds:
        - "{{ InstanceId }}"
```

**Key action types:**

| Action | Description |
|--------|-------------|
| `aws:executeAwsApi` | Call any AWS API directly |
| `aws:executeScript` | Run Python or PowerShell script inline |
| `aws:invokeLambdaFunction` | Invoke a Lambda function |
| `aws:waitForAwsResourceProperty` | Poll a resource property until desired value |
| `aws:runCommand` | Run a Run Command document on instances |
| `aws:branch` | Conditional branching |
| `aws:sleep` | Pause for a duration |

**Cross-account automation:** The automation document runs in the "automation account" but can target resources in other accounts by assuming a cross-account role.

---

### SSM State Manager

State Manager **maintains a desired configuration** on instances over time (declarative, drift-correcting).

**Association:**
- Links an SSM Document to targets (instances/resource groups) with a schedule.
- CloudWatch Logs and S3 output for each execution.
- Compliance reporting: instances that failed the association are non-compliant.

**Common managed associations:**
- `AWS-GatherSoftwareInventory` — collect inventory on a schedule
- `AWS-ApplyAnsiblePlaybooks` — run Ansible playbooks
- `AWS-ConfigureCloudWatch` — install/configure CloudWatch Agent
- `AWS-UpdateSSMAgent` — keep the SSM agent updated

**Run Command vs State Manager:**

| | Run Command | State Manager |
|--|------------|--------------|
| Trigger | On-demand (manual or API) | Scheduled, recurring, and on resource creation |
| Purpose | Execute once | Maintain ongoing desired state |
| Drift correction | No | Yes (re-applies if config drifts) |
| Compliance tracking | Limited | Yes (association compliance) |

---

### SSM Parameter Store — Hierarchy and Advanced Features

**Hierarchy:** Parameters are organised by path:
```
/myapp/prod/db/password
/myapp/prod/db/host
/myapp/staging/db/password
```

```python
# Get all parameters for an environment
ssm.get_parameters_by_path(Path='/myapp/prod/', Recursive=True, WithDecryption=True)
```

**Change notification:** EventBridge fires an event when a parameter is created, updated, or deleted. Use to trigger Lambda for downstream config refresh.

**Advanced parameter policies (Advanced tier only):**
- **Expiration:** Parameter automatically deleted after a date — forces rotation.
- **ExpirationNotification:** SNS/EventBridge event N days before expiry.
- **NoChangeNotification:** Alert if parameter hasn't changed in N days (stale credential detection).

---

### OpsCenter

OpsCenter creates **OpsItems** — structured work items for operational issues.

**OpsItem sources:**
- CloudWatch Alarm (auto-create OpsItem action)
- EventBridge rule → SNS → OpsCenter
- Security Hub finding
- Config finding

**OpsItem structure:**
- Title, description, severity, status (Open, In Progress, Resolved)
- Related AWS resources (the affected resource ARNs)
- Associated runbooks (SSM Automation documents)
- Notifications (SNS)

**Deduplication:** OpsItems have a `dedupleLstring` — if an OpsItem with the same dedupe string already exists (and is Open), a new one is not created.

---

## Phase 2 — Key Decision Framework

### "CloudFormation: which construct for this scenario?"

```
Reuse same VPC/ECS pattern across environments in one account?
  → Nested stacks (parent calls child nested stack templates)

Deploy same Config rules/IAM roles across 50 accounts?
  → StackSets (SERVICE_MANAGED with OUs)

Two stacks need to share a VPC ID in the same account?
  → Cross-stack references (Export/ImportValue)

Need to provision a 3rd-party resource or call an external API?
  → Custom Resource (Lambda-backed)

EC2 needs to signal readiness before stack continues?
  → CreationPolicy + cfn-signal in UserData
```

### "Which SSM tool?"

```
Run a one-time script on 100 instances right now?
  → Run Command

Ensure instances always have CloudWatch Agent configured (self-healing)?
  → State Manager association with schedule

Run a multi-step AWS API workflow (stop DB, snapshot, resize, start)?
  → SSM Automation

Give a developer shell access to an instance without SSH?
  → Session Manager

Patch EC2 fleet every Sunday at 2am with compliance reporting?
  → Patch Manager + Maintenance Window

Store config value consumed by applications?
  → Parameter Store (Standard: free; SecureString for secrets)

Store a DB password that must auto-rotate?
  → Secrets Manager (not Parameter Store)
```

---

## Phase 2 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "CloudFormation update replaces RDS — how to prevent?" | Stack Policy with `Deny Update:Replace` on the RDS logical resource ID |
| "Nested stack vs StackSet" | Nested = same account, DRY reuse. StackSet = multi-account/region deployment |
| "CFN change set Replacement: True — what does it mean?" | Resource will be DELETED and RECREATED (risk of data loss) |
| "SSM State Manager vs Run Command" | State Manager = recurring schedule / drift correction. Run Command = one-time execution |
| "cfn-init fails on EC2 — what happens to stack?" | cfn-signal sends failure exit code → CloudFormation rolls back (if CreationPolicy is used) |
| "SSM Parameter Store SecureString — which tier needed for policies?" | Advanced tier (standard SecureString has no parameter policies) |
| "StackSets SERVICE_MANAGED — new account joining OU" | Automatically gets StackSet deployed — no manual action needed |
| "AppConfig vs Parameter Store for feature flags" | AppConfig: gradual rollout, automatic rollback, validators. Parameter Store: no deployment strategy |
| "Custom Resource Lambda timeout — what happens?" | If Lambda doesn't call cfn-response within execution timeout, CloudFormation waits up to 1 hour then fails |
| "CDK L1 vs L2 construct" | L1 = direct CFN mapping (same verbosity). L2 = higher-level defaults (recommended for most use) |
