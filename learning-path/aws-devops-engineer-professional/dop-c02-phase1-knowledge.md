# DOP-C02 Phase 1 — Deep Knowledge Reference
## Domain 1: SDLC Automation (Weeks 2–4)

> Detailed technical knowledge for every bullet in Phase 1 of the DOP-C02 study plan.
> Cross-reference: [`aws-devops-engineer-professional.md`](./aws-devops-engineer-professional.md)

---

## Week 2 — Source Control and Build Automation

### CodeBuild — `buildspec.yml` Reference

CodeBuild executes your build using a `buildspec.yml` file at the root of the source repository (or inline in the project definition).

**Phase order (strictly sequential):**
```
install → pre_build → build → post_build
```

If any phase exits with a non-zero code, the build fails. `post_build` runs even if `build` fails (useful for uploading test reports even on failure).

**Full buildspec structure:**
```yaml
version: 0.2

env:
  variables:
    ENV_NAME: "production"           # plaintext env vars (visible in console)
  parameter-store:
    DB_PASSWORD: "/myapp/db/password"  # fetched from SSM Parameter Store
  secrets-manager:
    API_KEY: "myapp/api-key"          # fetched from Secrets Manager

phases:
  install:
    runtime-versions:
      java: corretto17
      docker: 20
    commands:
      - echo "Installing dependencies"
      - mvn dependency:resolve

  pre_build:
    commands:
      - echo "Logging into ECR"
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - mvn clean package -DskipTests=false
      - docker build -t $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_REGISTRY/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION

  post_build:
    commands:
      - docker push $ECR_REGISTRY/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo "Build complete"

artifacts:
  files:
    - target/*.jar
    - appspec.yml
    - scripts/**/*
  discard-paths: no

reports:
  JUnitReports:
    files:
      - "target/surefire-reports/**/*.xml"
    file-format: JUNITXML
  CoverageReports:
    files:
      - "target/site/jacoco/jacoco.xml"
    file-format: JACOCOXML

cache:
  paths:
    - "/root/.m2/**/*"       # local cache path
```

**Key built-in environment variables:**
| Variable | Value |
|----------|-------|
| `CODEBUILD_BUILD_ID` | Unique build ID |
| `CODEBUILD_RESOLVED_SOURCE_VERSION` | Commit SHA (git) |
| `CODEBUILD_SOURCE_REPO_URL` | Source repository URL |
| `AWS_DEFAULT_REGION` | Region of the build |
| `CODEBUILD_BUILD_SUCCEEDING` | "1" if build succeeding so far |

---

### CodeBuild Caching

| Type | What is cached | Best for |
|------|---------------|---------|
| **No cache** | Nothing | Fast builds with no reusable dependencies |
| **Local cache** | Files on the build agent (within a single build run) | Docker layer cache within same build |
| **S3 cache** | Tar archive uploaded/downloaded from S3 between builds | Maven ~/.m2, npm node_modules, pip packages |

**S3 cache configuration:**
```yaml
cache:
  paths:
    - "/root/.m2/**/*"
    - "/root/.npm/**/*"
```

**Cache key:** The cache is keyed per CodeBuild project. All builds of the same project share the same S3 cache object. Use local Docker layer cache (`DOCKER_BUILDKIT`, `--cache-from`) in combination with S3 for Docker builds.

**Exam trap:** S3 cache has an eventual consistency delay — a build may not see a cache uploaded by a concurrent build. Cache misses are handled gracefully (build continues without cache).

---

### CodeBuild Reports — Test and Coverage Gating

CodeBuild Reports let you publish test results and coverage into the console for visibility and pipeline gating.

**Supported formats:**
- JUnit XML (`JUNITXML`) — most common for Java, Node.js
- NUnit XML (`NUNITXML`)
- Visual Studio TRX (`VISUALSTUDIOTESTRESULTS`)
- Cucumber JSON (`CUCUMBERJSON`)
- TestNG XML (`TESTNGXML`)
- Clover XML (`CLOVERXML`) — coverage
- Cobertura XML (`COBERTURAXML`) — coverage
- JaCoCo XML (`JACOCOXML`) — coverage

**Gating on reports:**
CodeBuild does not automatically fail a build based on report content. To gate:
1. Run tests and export report file.
2. Parse the report in the `post_build` phase using a bash script or a test framework flag.
3. Exit with non-zero code if the threshold is not met.

```bash
# Example: fail if coverage < 80%
COVERAGE=$(python3 -c "import xml.etree.ElementTree as ET; t=ET.parse('coverage.xml').getroot(); print(float(t.get('line-rate','0'))*100)")
if (( $(echo "$COVERAGE < 80" | bc -l) )); then
  echo "Coverage $COVERAGE% is below 80% threshold"
  exit 1
fi
```

---

### CodeBuild — VPC Builds

By default, CodeBuild builds run on AWS-managed compute outside of your VPC. This means they cannot reach private resources (private RDS, private ECR endpoints, private NuGet feeds).

**VPC build configuration:**
- Specify VPC ID, subnet IDs (private subnets), and security group IDs in the CodeBuild project.
- The build runs on compute inside those subnets.
- Private resources become reachable.
- For internet access (to pull Docker base images, install packages): add a **NAT Gateway** in the VPC — CodeBuild in a VPC has no internet access by default.

**Exam pattern:** "CodeBuild must run integration tests against a private RDS database" → VPC builds.

---

### Docker-in-Docker Builds (Privileged Mode)

Building Docker images inside CodeBuild requires **privileged mode** enabled on the build project. Without it, the Docker daemon is not available.

```
Project settings → Enable "Privileged" checkbox
```

With privileged mode:
- `docker build`, `docker run`, `docker push` work inside the build.
- Docker-in-Docker is available via the host's Docker socket.

**Security note:** Privileged mode grants the container elevated Linux capabilities. Use a restrictive IAM role for the CodeBuild service role to limit what the build can do.

---

### CodeArtifact

CodeArtifact is a **managed artifact repository** supporting npm, Maven, PyPI, NuGet, Ruby gems, and Swift packages.

**Concepts:**

| Concept | Description |
|---------|-------------|
| **Domain** | Logical grouping of repositories; one AWS account owns the domain; other accounts can be included |
| **Repository** | Collection of packages matching a specific format (npm, Maven, etc.) |
| **Upstream repository** | A repository the current repo fetches from when a package is not found locally |
| **External connection** | Connects an upstream to a public registry (npmjs.com, Maven Central, PyPI) |
| **Pull-through cache** | Packages fetched from the upstream are cached locally in CodeArtifact |

**Repository chain example:**
```
Your project build
  → CodeArtifact "my-npm" repo (check local)
    → Upstream: CodeArtifact "shared-npm" repo (check shared internal packages)
      → External connection: npmjs.com (public packages)
```

**Cross-account sharing:**
1. Domain owner grants `codeartifact:ReadFromRepository` to the consumer account via resource policy.
2. Consumer account creates a repository with an upstream pointing to the cross-account repository.
3. Consumer CodeBuild uses the consumer account's endpoint URL.

**Package origin controls:** Configure whether a package in your repository can only come from upstream (blocks someone from publishing a malicious version of a public package name).

---

### Amazon ECR — Exam-Relevant Features

**Image scanning:**

| Mode | When scanned | Engine | What it finds |
|------|-------------|--------|--------------|
| **Basic scanning** | On push (once) | CoreOS Clair | OS package CVEs |
| **Enhanced scanning (Inspector v2)** | On push + continuously | Inspector v2 | OS packages + programming language packages; new CVEs re-evaluated |

**Lifecycle policies:** Automatically delete old/untagged images based on rules:
```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Delete untagged images older than 7 days",
    "selection": {
      "tagStatus": "untagged",
      "countType": "sinceImagePushed",
      "countUnit": "days",
      "countNumber": 7
    },
    "action": {"type": "expire"}
  }, {
    "rulePriority": 2,
    "description": "Keep only 10 tagged images per repo",
    "selection": {
      "tagStatus": "tagged",
      "tagPrefixList": ["release-"],
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": {"type": "expire"}
  }]
}
```

**Immutable tags:** Prevent overwriting an existing tag. Once `v1.2.3` is pushed, no future push can overwrite it. Enforces build traceability.

**Cross-account pull:**
1. ECR repository resource policy — allow `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage` from the consumer account.
2. Consumer EC2/ECS/CodeBuild role — allow `ecr:GetAuthorizationToken` (always needed).

**ECR pull-through cache:** Cache images from public registries (Docker Hub, Quay, ECR Public, GitHub Container Registry) into your private ECR — avoids rate limits and provides a local copy for reliability.

---

### CodeCommit — Cross-Account Access Pattern

```
Developer in Account A owns the repository
Pipeline in Account B needs to pull source

Account A: Create CodeCommit repository.
Account A: IAM role "CodeCommitReader" with codecommit:GitPull permissions.
           Trust policy: allow Account B to assume this role.

Account B: CodePipeline source action uses Account A role via cross-account configuration.
```

**Triggers:** CodeCommit can trigger on branch push/create/delete and tag events:
- SNS notification
- Lambda function invocation

**Approval rule templates:** Require a number of approvals from a specific approval pool (IAM users/roles) before a pull request can be merged. Approval rule templates apply automatically to all PRs in specified repositories.

---

## Week 3 — Deployment Automation

### CodeDeploy — Lifecycle Hooks

**EC2 / On-Premises lifecycle hooks:**
```
ApplicationStop          ← stop running app gracefully
  ↓
DownloadBundle           ← CodeDeploy agent fetches revision from S3/GitHub
  ↓
BeforeInstall            ← pre-installation scripts (backup files, decrypt)
  ↓
Install                  ← copy files to destination
  ↓
AfterInstall             ← post-installation scripts (configure, set permissions)
  ↓
ApplicationStart         ← start the application
  ↓
ValidateService          ← health checks (HTTP ping, smoke test)
```

**`appspec.yml` for EC2:**
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/myapp
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 30
      runas: root
  AfterInstall:
    - location: scripts/configure.sh
      timeout: 60
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 30
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 120
```

**ECS Blue/Green lifecycle hooks:**
```
BeforeAllowTraffic   ← run tests before routing any traffic to new task set
AfterAllowTraffic    ← run tests after traffic shift complete
```

**Lambda lifecycle hooks:**
```
BeforeAllowTraffic   ← run validation before traffic shift begins
AfterAllowTraffic    ← run validation after traffic shift completes
```

**Exam tip:** Draw this table from memory — hook names differ per compute type.

| Phase | EC2 | ECS Blue/Green | Lambda |
|-------|-----|---------------|--------|
| Before traffic | BeforeInstall | BeforeAllowTraffic | BeforeAllowTraffic |
| After traffic | ValidateService | AfterAllowTraffic | AfterAllowTraffic |

---

### CodeDeploy Deployment Configurations

**EC2/On-premises:**

| Config | Behaviour |
|--------|-----------|
| `CodeDeployDefault.AllAtOnce` | All instances at once; fastest; highest risk |
| `CodeDeployDefault.HalfAtATime` | Half at a time |
| `CodeDeployDefault.OneAtATime` | One at a time; safest |
| Custom | Define minimum healthy hosts (count or percentage) |

**Lambda and ECS traffic shifting:**

| Type | Pattern | Config example |
|------|---------|---------------|
| **Canary** | X% → wait N minutes → 100% | `CodeDeployDefault.LambdaCanary10Percent5Minutes` |
| **Linear** | X% every N minutes | `CodeDeployDefault.LambdaLinear10PercentEvery1Minute` |
| **All at once** | 100% immediately | `CodeDeployDefault.LambdaAllAtOnce` |

**Rollback:**
- **Automatic:** Triggered by a CloudWatch Alarm associated with the deployment group OR when a lifecycle hook fails.
- **Manual:** Operator triggers via console/CLI.
- Rollback = CodeDeploy re-deploys the LAST SUCCESSFUL revision (not reverting code — deploying a new deployment of the old revision).
- Redeploy ≠ rollback: a redeploy deploys the same revision again; rollback deploys the previous revision.

---

### CodePipeline — Architecture

**CodePipeline structure:**
```
Pipeline
├── Stage 1: Source
│   └── Action: CodeCommit (or S3, ECR, GitHub via CodeStar)
├── Stage 2: Build
│   └── Action: CodeBuild (build + unit tests)
├── Stage 3: Test
│   ├── Action: CodeBuild (integration tests) — parallel with ↓
│   └── Action: Lambda Invoke (custom test gate)
├── Stage 4: Approval
│   └── Action: Manual Approval (sends SNS → reviewer approves/rejects in console)
└── Stage 5: Deploy
    └── Action: CodeDeploy (blue/green to ECS)
```

**Parallel actions:** Actions in the same stage with the same `runOrder` execute in parallel. Different `runOrder` values within a stage are sequential.

**Cross-account CodePipeline deployment:**
```
Pipeline account (Account A):
  - Pipeline IAM role: can assume target account's deploy role
  - Artifact S3 bucket: KMS CMK (both accounts' roles need kms:Decrypt)
  - CodePipeline assumes cross-account role for each action in target account

Target account (Account B):
  - Deploy role: trusted by Account A pipeline role
  - Deploy role: has permissions for CodeDeploy/CloudFormation/ECS in Account B
```

**EventBridge triggers:**
- CodeCommit push → EventBridge rule → CodePipeline `StartPipelineExecution`
- ECR image push → EventBridge rule → CodePipeline
- S3 object upload (with CloudTrail enabled) → EventBridge → CodePipeline

**Exam trap:** CloudWatch Events and EventBridge are the same service. The exam may use either name.

---

### Elastic Beanstalk Deployment Modes

| Mode | Downtime | Speed | Rollback |
|------|---------|-------|---------|
| **All at once** | Yes (brief) | Fastest | Re-deploy old version |
| **Rolling** | No (some capacity reduced) | Slow | Re-deploy old version |
| **Rolling with additional batch** | No | Medium | Re-deploy old version |
| **Immutable** | No | Slowest | Terminate new fleet |
| **Blue/green** | No (DNS swap) | Slow (new environ.) | Swap URL back |

**Immutable deployment:** Launches a new Auto Scaling group alongside the existing one. Deploys to the new group. If successful, moves instances to the existing group. Safest for stateful changes.

**Blue/green in Beanstalk:** Create a second Beanstalk environment, deploy the new version, then swap CNAMEs via `eb swap`. Use Route 53 weighted routing for gradual traffic shift.

**`.ebextensions`:** YAML configuration files in the `.ebextensions/` directory of your application bundle that customise the Beanstalk environment:
```yaml
# .ebextensions/01-packages.config
packages:
  yum:
    git: []
    jq: []

commands:
  01_configure:
    command: "echo 'configured' > /tmp/setup.log"
```

---

### AWS AppConfig

AppConfig manages **feature flags and runtime configuration** separately from your deployment pipeline. Changes can be deployed gradually with automatic rollback.

**Concepts:**
| Concept | Description |
|---------|-------------|
| **Application** | Logical grouping (e.g., "payment-service") |
| **Environment** | Prod, staging, dev |
| **Configuration profile** | Where config lives: SSM Parameter Store, S3, or Secrets Manager |
| **Deployment strategy** | How to roll out config: linear, exponential, all-at-once |
| **Validator** | JSON Schema validator or Lambda validator — blocks deployment if config is invalid |

**Deployment strategies:**

| Strategy | Description |
|----------|-------------|
| All-at-once | Deploy to 100% immediately |
| Linear | Deploy X% every Y minutes (e.g., 10% every 1 minute) |
| Exponential | Deploy 1%, 2%, 4%, 8%… (slower start, faster later) |

**Automatic rollback:** Configure a CloudWatch Alarm on error rate metrics. If the alarm triggers during an AppConfig deployment, AppConfig automatically rolls back to the previous configuration version.

**SDK integration:** Applications poll AppConfig at a configurable interval (e.g., every 30 seconds) via the AppConfig SDK. The SDK caches the configuration locally — no per-request calls to AppConfig.

---

### EC2 Image Builder

Image Builder automates the creation, testing, and distribution of custom AMIs and container images.

**Pipeline components:**

| Component | Description |
|-----------|-------------|
| **Infrastructure configuration** | Instance type, IAM role, VPC/subnet, SNS notification for builds |
| **Recipe** | Base image (e.g., Amazon Linux 2023) + ordered list of build components |
| **Build components** | Shell scripts or YAML component documents (install packages, configure services) |
| **Test components** | Post-build tests (run smoke tests, verify service health) |
| **Distribution settings** | Target regions for AMI copy, target accounts, AMI name format |

**Component YAML:**
```yaml
name: InstallNginx
description: Installs and configures NGINX
schemaVersion: 1.0
phases:
  - name: build
    steps:
      - name: InstallNginx
        action: ExecuteBash
        inputs:
          commands:
            - yum install -y nginx
            - systemctl enable nginx
  - name: test
    steps:
      - name: TestNginx
        action: ExecuteBash
        inputs:
          commands:
            - systemctl is-active nginx || exit 1
```

**Cross-region distribution:** Image Builder can copy the resulting AMI to multiple regions and share it with other accounts automatically.

---

## Week 4 — Testing Automation and Release Gates

### CloudWatch Synthetics

Synthetics **canaries** are scripts that run on a schedule and simulate user interactions, validating endpoints even outside of deployments.

**Canary runtimes:** Node.js (Puppeteer) for UI tests, Python (Boto3) for API tests.

**Built-in blueprints:**
- `Heartbeat` — HTTP(S) GET, returns success/failure
- `API Canary` — runs a series of REST API calls, validates responses
- `Broken Link Checker` — crawls URLs, reports broken links
- `GUI Workflow Builder` — Puppeteer-based UI tests

**Integration with CodePipeline:**
A Lambda Invoke action in CodePipeline can trigger a Synthetics canary run and evaluate the result — serving as an automated post-deploy gate.

**CloudWatch Alarms on Synthetics:**
- `SuccessPercent` — fires if canary success rate drops below threshold
- `Duration` — fires if canary takes too long (degraded performance)

---

### Step Functions in CI/CD Context

Step Functions orchestrate multi-step pipelines with retry logic, timeouts, and conditional branching.

**In CI/CD, Step Functions are useful for:**
- **Parallel test execution:** Map state fans out integration tests across N environments simultaneously.
- **Multi-account deploy orchestration:** Fan-out deploys to 20 accounts with success/failure tracking.
- **Human approval with timeout:** WaitForTaskToken pattern — CodePipeline or Step Functions sends a token and waits; the approver HTTP calls back with the token to continue or reject.
- **Automated DR drills:** Spin up recovery environment → validate → tear down — with each step guarded by conditions.

**WaitForTaskToken pattern:**
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
  "Parameters": {
    "QueueUrl": "https://sqs.eu-west-1.amazonaws.com/123456789012/approval-queue",
    "MessageBody": {
      "taskToken.$": "$$.Task.Token",
      "input.$": "$"
    }
  }
}
```
The Step Function pauses. An external system (human approval, test system) calls `SendTaskSuccess` or `SendTaskFailure` with the token to resume or fail the execution.

---

### DORA Metrics — Pipeline Design Implications

DORA metrics measure software delivery performance. SAP-C02 covers them conceptually; DOP-C02 tests which AWS service measures each.

| Metric | Definition | Measured with |
|--------|------------|--------------|
| **Deployment Frequency** | How often you deploy to production | CodePipeline execution history, CloudWatch custom metric on `PipelineExecutionSucceeded` events |
| **Lead Time for Changes** | Time from commit to production | CodePipeline stage duration; CloudWatch metric from CodeCommit push event to pipeline completion |
| **Change Failure Rate** | % of deployments causing incidents | CodeDeploy rollback rate; CloudWatch Alarm firing rate post-deploy |
| **MTTR (Mean Time to Restore)** | Time to restore from incident | OpsCenter OpsItem open-to-closed time; CloudWatch Alarm duration |

**Trunk-based development → pipeline implications:**
- Every commit to `main` triggers the pipeline (no feature branches lingering).
- Feature flags (AppConfig) control feature visibility — code ships before the feature is enabled.
- Short-lived branches: PRs merged within hours, not days.
- Consequence: pipeline must be fast (< 10 minutes to production) and reliable.

**GitFlow → pipeline implications:**
- Release branches are the trigger, not every commit.
- Multiple environments (feature, develop, release, hotfix) — more complex pipeline topology.
- Higher batch sizes per deployment → higher change failure rate risk.

---

### CodePipeline — Pipeline Notifications and Audit

**CodeStar Notifications:**
Sends pipeline execution events (started, succeeded, failed, cancelled) to:
- SNS topic → Email, Lambda, Slack (via Lambda)
- AWS Chatbot → Slack or Teams channel

**CloudTrail logging:**
All CodePipeline API calls (StartPipelineExecution, PutJobSuccessResult, etc.) are logged to CloudTrail. Use for audit: who triggered a pipeline execution, who approved a manual approval, what changed.

**Pipeline execution history:**
- Console: navigate to pipeline → "View history" — shows all executions with start/end time, status, trigger source, and artifact links.
- API: `ListPipelineExecutions`, `GetPipelineState` — use for monitoring/dashboards.

---

## Phase 1 — Key Decision Framework

### "Which CodeDeploy configuration for this scenario?"

```
Need to shift 10% of Lambda traffic, wait 5 minutes, then shift 100%?
  → LambdaCanary10Percent5Minutes

Need to shift Lambda traffic by 10% every minute until 100%?
  → LambdaLinear10PercentEvery1Minute

Need EC2 in-place, only 1 instance at a time?
  → CodeDeployDefault.OneAtATime

Need ECS, test new task set before any traffic shifts?
  → BeforeAllowTraffic hook + blue/green
```

### "Which Beanstalk deployment mode?"

```
Fastest, downtime acceptable (dev/test)?
  → All at once

No downtime, fast rollback, stateless app?
  → Immutable

No downtime, gradual, cost-conscious?
  → Rolling with additional batch

Staged release with DNS-level control?
  → Blue/Green (separate environment + CNAME swap)
```

---

## Phase 1 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "CodeDeploy rollback vs redeploy" | Rollback = deploy PREVIOUS revision; Redeploy = deploy SAME revision again |
| "CodeBuild test report gate — does it auto-fail?" | NO. You must parse the report and `exit 1` in your buildspec to fail the build |
| "CodePipeline detects ECR image push — how?" | EventBridge rule on ECR push event → CodePipeline `StartPipelineExecution` |
| "CodeBuild in VPC — internet access?" | NO. Add NAT Gateway in the VPC for internet access |
| "CodeDeployDefault.LambdaCanary10Percent5Minutes — what does it do?" | Shifts 10% traffic immediately, waits 5 min (alarm window), then shifts remaining 90% |
| "AppConfig validator failing — does deployment proceed?" | NO. Deployment is blocked until a valid config is provided |
| "Cross-account CodePipeline — what KMS requirement?" | Artifact S3 bucket KMS CMK must have key policy allowing both source and target account roles |
| "S3 cache in CodeBuild — shared across projects?" | NO. Cache is per-project. Different projects have separate cache archives |
| "EC2 Image Builder distribution to another account" | Configure in distribution settings; target account does NOT need to be in same org |
| "Elastic Beanstalk `.ebextensions` vs platform hooks" | .ebextensions: YAML config for env resources. Platform hooks: scripts at specific EB lifecycle events |
