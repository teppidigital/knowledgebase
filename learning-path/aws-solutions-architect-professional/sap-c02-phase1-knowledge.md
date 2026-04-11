# SAP-C02 Phase 1 — Deep Knowledge Reference
## Domain 1: Design for Organisational Complexity (Weeks 2–4)

> This document provides the detailed technical knowledge behind every bullet in the Phase 1 study plan.
> Cross-reference: [`aws-solutions-architect-professional.md`](./aws-solutions-architect-professional.md)

---

## Week 2 — Multi-Account Governance

### AWS Organizations — Architecture

AWS Organizations is the backbone of enterprise AWS governance. Every account is a **member** of exactly one organisation. The **management account** (formerly master account) owns the organisation and cannot be moved.

```
Root
└── Organisational Unit (OU)
    ├── Organisational Unit (nested, up to 5 levels)
    │   └── AWS Account (leaf node)
    └── AWS Account
```

**Key distinctions for the exam:**

| Concept | Detail |
|---------|--------|
| Management account | Owns the org; **SCPs do NOT apply to it** |
| Delegated administrator | A member account can be granted admin rights for specific services (GuardDuty, Security Hub, Config, etc.) |
| OU depth | Maximum 5 levels of OUs under Root |
| Account move | Accounts can be moved between OUs without deleting them |
| Consolidated billing | All member account charges roll up to management account |

**Standard OU design for enterprises:**

```
Root
├── Management OU               ← ZERO workloads; billing and org control only
├── Security OU
│   ├── Log Archive Account     ← S3 immutable log bucket; read-only access
│   └── Security Tooling Account← GuardDuty delegated admin, Security Hub, Config aggregator
├── Infrastructure OU
│   ├── Network Account         ← Transit Gateway, Route 53 Resolver, shared VPCs via RAM
│   └── Shared Services Account ← ECR, internal NuGet/npm registries, Artifactory
├── Workloads OU
│   ├── Production OU
│   │   └── <service>-prod
│   └── Non-Production OU
│       ├── <service>-staging
│       └── <service>-dev
├── Policy Staging OU           ← Test new SCPs here before promoting to production OUs
└── Sandbox OU                  ← Developer experiments; strict SCPs; auto-expire 30 days
```

**Account vending machine pattern:**
1. Engineer submits a request (ServiceNow ticket or Terraform variable).
2. Control Tower Account Factory (or custom Lambda) provisions a new account.
3. Baseline applied automatically: SSO assignment, VPC structure, CloudTrail, Config, mandatory tags.
4. Account joins the correct OU → inherits SCPs automatically.

---

### Service Control Policies (SCPs) — Deep Dive

SCPs are **organisation-level permission guardrails**. They do not grant permissions — they set the maximum permission boundary for all principals in an account.

**Evaluation logic:**
```
Request allowed only if:
  1. No SCP anywhere in the OU hierarchy denies it, AND
  2. The IAM policy for the principal allows it

Management account: SCPs are NEVER evaluated — always full access
```

**SCP inheritance (additive scope, cumulative deny):**
```
Root SCP:       Deny leaving organisation
├── Security OU SCP:  Deny disabling GuardDuty
│   └── Security Tooling Account: Inherits BOTH above SCPs
└── Workloads OU SCP: Deny creating IAM users
    └── Production OU SCP: Deny deleting Config rules
        └── service-prod account: Inherits ALL four SCPs
```

**`Allow` vs `Deny` strategy:**

| Strategy | When to Use | Risk |
|----------|-------------|------|
| **Allow-list (whitelist)** | Restrict to specific approved services only | Blocks new services until SCP updated; high maintenance |
| **Deny-list (blacklist)** | Block specific dangerous actions; allow everything else | Simpler; used by most enterprises |

**Critical SCP patterns to know for the exam:**

```json
// Prevent leaving the organisation
{
  "Sid": "DenyLeaveOrg",
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}

// Prevent disabling security services
{
  "Sid": "DenySecurityDisable",
  "Effect": "Deny",
  "Action": [
    "guardduty:DeleteDetector",
    "guardduty:DisassociateFromMasterAccount",
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "config:DeleteConfigRule",
    "config:StopConfigurationRecorder"
  ],
  "Resource": "*"
}

// Region restriction — only allow eu-west-1 and us-east-1
{
  "Sid": "DenyOutsideApprovedRegions",
  "Effect": "Deny",
  "NotAction": [
    "iam:*",
    "sts:*",
    "support:*",
    "budgets:*",
    "s3:GetBucketLocation"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["eu-west-1", "us-east-1"]
    }
  }
}
```

**`NotAction` pattern** — used for region restriction because IAM, STS, and billing are global services that must be excluded from region-based deny.

**Exam trap:** An SCP `Deny` at the Root OU does NOT apply to the management account. Only member accounts are affected by SCPs.

---

### AWS Control Tower

Control Tower is AWS's **managed Landing Zone** service. It automates multi-account setup using Organizations, SSO, Config, CloudTrail, and S3.

**Components:**

| Component | Description |
|-----------|-------------|
| Landing Zone | The overall multi-account environment managed by Control Tower |
| Account Factory | Self-service account vending; uses Service Catalog and Organizations |
| Guardrails | Preventive (SCPs) or Detective (AWS Config rules) policy enforcement |
| Audit account | Dedicated account for aggregated Config findings and audit access |
| Log Archive account | Centralised, immutable S3 bucket for CloudTrail and Config logs |

**Guardrail types:**

| Type | Implementation | Example |
|------|---------------|---------|
| **Preventive** | SCP | Deny creating VPCs without flow logs enabled |
| **Detective** | AWS Config rule | Detect S3 buckets without server-side encryption |

**Guardrail guidance levels:**

| Guidance | Meaning |
|----------|---------|
| Mandatory | Always enabled; cannot be disabled |
| Strongly Recommended | Best practice; can be disabled |
| Elective | Optional; enable for specific compliance requirements |

**Account Factory for Terraform (AFT):** Control Tower integration that lets you define accounts as Terraform code. Account customisations (VPC, roles, tags) applied as Terraform pipelines post-vending.

---

### AWS Config — Cross-Account Compliance

Config records the **configuration history** of AWS resources and evaluates rules against them.

| Concept | Detail |
|---------|--------|
| Configuration recorder | Records resource state changes to S3 + SNS |
| Config rule | Custom or managed rule evaluated against resource config |
| Conformance pack | Collection of Config rules deployed as a unit (e.g., CIS benchmark, PCI-DSS) |
| Aggregator | Combines Config data from multiple accounts/regions into one view |
| Remediation | Automatic or manual SSM automation to fix non-compliant resources |

**Aggregator setup for org-wide compliance:**
1. Designate a **delegated administrator** account (Security Tooling) to be the aggregator.
2. Config aggregator pulls findings from all member accounts.
3. Centralised dashboard in Security Tooling account shows compliance across the entire org.

**Tagging enforcement pattern:**
```
Config rule: required-tags
  → Checks that EC2, RDS, S3, Lambda all have tags: Environment, Owner, CostCentre
  → Non-compliant → Remediation SSM document applies tags from account metadata
  → SCP: deny resource creation if aws:RequestTag/Environment is missing
```

---

### CloudTrail — Organisation Trail

| Trail type | Scope | Log destination |
|------------|-------|-----------------|
| Per-account trail | Single account | Any S3 bucket |
| Organisation trail | All current + future accounts | S3 in Log Archive account |

**Organisation trail facts:**
- Created from the management account only.
- Automatically applies to all member accounts (existing and new).
- Member accounts can see the trail exists but **cannot disable or modify it**.
- Logs go to a single S3 bucket in the Log Archive account with a bucket policy that denies `s3:DeleteObject`.

**Log structure:** `s3://bucket/AWSLogs/org-id/account-id/CloudTrail/region/YYYY/MM/DD/`

**Event types:**
| Type | What it captures | Default |
|------|-----------------|---------|
| Management events | API calls on AWS resources (CreateBucket, RunInstances) | ON |
| Data events | S3 object-level (GetObject, PutObject), Lambda invocations | OFF (extra cost) |
| Insights events | Anomalous API call rates or error rates | OFF (extra cost) |

---

## Week 3 — Hybrid Connectivity

### AWS Direct Connect

Direct Connect provides a **dedicated, private, physical network connection** from your on-premises datacentre to AWS. It does not traverse the public internet.

**Connection types:**

| Type | Speed | How obtained |
|------|-------|-------------|
| Dedicated connection | 1 Gbps, 10 Gbps, 100 Gbps | Ordered directly from AWS; physical cross-connect at DX location |
| Hosted connection | 50 Mbps – 10 Gbps | Ordered via AWS Direct Connect partner |

**Virtual Interfaces (VIFs) — critical exam topic:**

| VIF Type | Connects to | Use case |
|----------|-------------|---------|
| **Private VIF** | VPC via Virtual Private Gateway (VGW) | Access resources in a single VPC privately |
| **Public VIF** | AWS public endpoints | Access S3, DynamoDB, SNS, SQS without traversing internet |
| **Transit VIF** | Direct Connect Gateway → Transit Gateway | Access multiple VPCs/regions from one DX connection |

**Direct Connect Gateway (DXGW):**
- Global resource — not region-specific.
- A single DXGW can attach to VPCs in **multiple AWS regions** via Transit VIF.
- A DXGW attaches to **up to 20 Transit Gateways** (or up to 10 VGWs for private VIFs).
- Enables a single DX connection to provide connectivity to VPCs across regions.

**High availability patterns:**

| Pattern | Resilience | Cost |
|---------|-----------|------|
| Single DX connection | No HA | Low |
| Two DX connections (same DX location) | Survives DX port failure | Medium |
| Two DX connections (different DX locations) | Survives DX location failure | High |
| DX + Site-to-Site VPN backup | Failover to VPN if DX fails | Medium (VPN is cheaper backup) |

**MACsec:** Layer 2 encryption on the dedicated DX connection. Only available on dedicated connections (not hosted). Encrypts the physical link between your router and the AWS DX router.

**BGP:** Direct Connect always uses BGP for routing. You must configure BGP on your on-premises router. BGP attributes (AS path prepending, local preference) control traffic engineering.

---

### AWS Transit Gateway (TGW)

Transit Gateway is a **regional, hub-and-spoke** network transit hub. Every attachment connects to the TGW, and route tables on the TGW control which attachments can communicate.

**Attachment types:**

| Attachment | Description |
|------------|-------------|
| VPC | Attach a VPC to TGW via TGW attachment in the VPC |
| VPN | Site-to-Site VPN terminates on TGW (not VGW) |
| Direct Connect | Via Transit VIF → Direct Connect Gateway → TGW |
| TGW Peering | Connect TGWs across regions (static routes only; no BGP) |
| AWS Cloud WAN | Managed global network using TGWs as edge connections |

**Route tables:**

- TGW has its own route tables (separate from VPC route tables).
- An attachment can be associated with exactly **one** TGW route table.
- An attachment can propagate routes into **multiple** TGW route tables.
- Blackhole routes drop traffic to specific CIDRs — used to prevent unwanted VPC-to-VPC traffic.

**Segmentation pattern (exam favourite):**

```
TGW Route Table: Production
  → Associated: prod-vpc-A, prod-vpc-B
  → Propagated from: prod-vpc-A, prod-vpc-B
  → Cannot reach: dev VPCs (not in this route table)

TGW Route Table: Dev
  → Associated: dev-vpc-A, dev-vpc-B
  → Propagated from: dev-vpc-A, dev-vpc-B
  → Cannot reach: prod VPCs

TGW Route Table: Shared Services
  → Associated: shared-services-vpc
  → Propagated from: ALL VPCs (so shared services can reach all)
```

**VPC Peering vs Transit Gateway:**

| Criterion | VPC Peering | Transit Gateway |
|-----------|-------------|-----------------|
| Transitive routing | NO — must peer every pair | YES — hub-and-spoke |
| Number of connections | N*(N-1)/2 for full mesh | N connections to TGW |
| Cost | Free data transfer (same region) | $0.05/GB + $0.05/hour per attachment |
| Cross-region | Yes (inter-region peering) | Yes (TGW peering with static routes) |
| Bandwidth limit | No limit | 50 Gbps per VPC attachment |
| Route management | Manual per VPC | Centralised TGW route tables |

**Key rule:** VPC Peering is NOT transitive. If A peers with B, and B peers with C, A cannot reach C through B. You must peer A with C directly. Transit Gateway solves this.

**TGW Network Manager:** Global service that provides a central dashboard for your global network, including on-premises connections via SD-WAN. Supports Cloud WAN for managed global routing.

---

### Site-to-Site VPN

| Property | Detail |
|----------|--------|
| Bandwidth | Up to 1.25 Gbps per tunnel |
| Tunnels | Always 2 tunnels per VPN connection (HA) |
| Routing | Static or BGP (dynamic) |
| Attachment | VGW (per-VPC) or Transit Gateway (shared across VPCs) |

**Redundancy:** Each VPN connection has 2 tunnels in different AWS AZs. For full HA, create 2 VPN connections from 2 different customer gateway devices.

**Accelerated VPN:** Routes VPN traffic through AWS Global Accelerator anycast edge nodes — reduces latency for geographically distant endpoints.

**ECMP (Equal-Cost Multi-Path):** When attaching multiple VPN connections to a TGW with BGP, ECMP allows load balancing across all available tunnels — effectively multiplying throughput beyond 1.25 Gbps.

---

### Route 53 Resolver

Enables **bi-directional DNS resolution** between on-premises and AWS VPC private hosted zones.

| Endpoint type | Direction | Use case |
|--------------|-----------|---------|
| **Inbound endpoint** | On-premises → AWS | On-premises DNS server forwards queries for `*.internal.aws.example.com` to this endpoint |
| **Outbound endpoint** | AWS → On-premises | Route 53 Resolver forwards queries for `*.onprem.example.com` to on-premises DNS server |

**Setup:**
1. Create Inbound Resolver endpoint in VPC (2+ ENIs in different AZs).
2. On-premises DNS server adds a conditional forwarder: `aws.internal → <inbound_endpoint_IPs>`.
3. Create Outbound Resolver endpoint.
4. Create Resolver rule: `onprem.example.com → <on-premises_DNS_server_IP>`.
5. Share resolver rules via RAM with other VPCs in the org.

---

### AWS PrivateLink

PrivateLink exposes a service (your own or an AWS service) to consumers **privately via Interface Endpoints** — no internet gateway, no NAT, no public IP required.

| Component | Role |
|-----------|------|
| **Endpoint Service** | Your NLB-backed service exposed via PrivateLink |
| **Interface Endpoint** | ENI in consumer VPC that provides private IP to reach the service |
| **Gateway Endpoint** | S3 and DynamoDB only; route table entry (not ENI); free |

**Key distinctions:**
- Gateway endpoints (S3, DynamoDB) are routed via route table — no ENI, no IP, no cost per hour.
- Interface endpoints (all other services) create an ENI in your subnet — billed per hour + per GB.
- Interface endpoints can be accessed across VPC peering and Transit Gateway.
- Gateway endpoints **cannot** be accessed across VPC peering or Transit Gateway.

---

## Week 4 — Identity Federation and Cross-Account Access

### IAM Identity Center (SSO)

IAM Identity Center is the recommended approach for **human access** to AWS accounts. It replaces direct IAM users.

**Components:**

| Component | Description |
|-----------|-------------|
| Identity source | Where users and groups are defined (Identity Center built-in, Active Directory, external IdP) |
| Permission set | A bundle of IAM policies (managed + inline) that is assigned to a user/group in an account |
| Account assignment | Maps a user/group + permission set to a specific AWS account |

**SCIM provisioning:** When using an external IdP (Okta, Azure AD), SCIM syncs users and groups automatically to IAM Identity Center. Without SCIM, you must add users manually.

**Access flow:**
```
User → IdP login → SAML assertion to IAM Identity Center → 
  STS AssumeRoleWithSAML → Temporary credentials for target account role
```

**Permission set implementation:** Each permission set becomes an IAM role in each assigned account. The role name pattern: `AWSReservedSSO_<PermissionSetName>_<RandomSuffix>`.

---

### SAML 2.0 Federation

SAML 2.0 allows external Identity Providers (Okta, Azure AD, ADFS) to authenticate users and assert their identity to AWS.

**Flows:**

| Flow | Who initiates | Sequence |
|------|--------------|---------|
| **IdP-initiated** | IdP | User logs into IdP portal → selects AWS app → IdP sends SAML assertion to AWS SSO |
| **SP-initiated** | AWS | User navigates to AWS console URL → redirected to IdP → authenticates → redirected back |

**STS AssumeRoleWithSAML:**
- Called by the application or AWS console after receiving a SAML assertion.
- The SAML assertion must include `https://aws.amazon.com/SAML/Attributes/Role` attribute specifying the role ARN.
- SAML provider must be created in the AWS account and trusted in the IAM role's trust policy.

```json
// Trust policy for SAML federation
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::123456789012:saml-provider/MyIdP"},
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": {
      "StringEquals": {"SAML:aud": "https://signin.aws.amazon.com/saml"}
    }
  }]
}
```

---

### OIDC Federation

OIDC (OpenID Connect) is used for **workload identity** — allowing software to assume AWS roles without long-term credentials.

**Key use cases:**

| Use case | Service | How |
|----------|---------|-----|
| GitHub Actions CI/CD | GitHub OIDC | GitHub issues JWT; AWS validates against OIDC provider; STS AssumeRoleWithWebIdentity |
| EKS pod-level IAM | IRSA | EKS OIDC provider; Kubernetes ServiceAccount annotated with role ARN; pod gets temporary creds |
| Lambda | Not needed | Lambda execution role assigned directly — no OIDC required |

**IRSA (IAM Roles for Service Accounts) deep dive:**

```yaml
# ServiceAccount with IRSA annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyAppRole
```

```json
// IAM role trust policy for IRSA
{
  "Principal": {"Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E"},
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E:sub": "system:serviceaccount:default:my-app",
      "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E:aud": "sts.amazonaws.com"
    }
  }
}
```

**GitHub Actions OIDC:**
```yaml
permissions:
  id-token: write   # Required for OIDC token generation
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      aws-region: eu-west-1
```

---

### AWS STS — Assume Role Patterns

| API call | When to use |
|----------|-------------|
| `AssumeRole` | Cross-account access; role chaining; service-to-service |
| `AssumeRoleWithSAML` | Human SSO via SAML IdP |
| `AssumeRoleWithWebIdentity` | Workload identity via OIDC (GitHub Actions, EKS IRSA, mobile apps) |
| `GetSessionToken` | MFA enforcement for IAM users (legacy) |
| `GetFederationToken` | Custom federation broker (legacy; prefer IAM Identity Center) |

**`sts:ExternalId` — confused deputy prevention:**

Without ExternalId, a malicious third party could trick AWS into assuming your role by knowing your account ID and role name. ExternalId is a secret you share only with the trusted third party and require in the role's trust policy condition.

```json
{
  "Condition": {
    "StringEquals": {"sts:ExternalId": "your-unique-external-id"}
  }
}
```

**Role chaining:** Role A assumes Role B, which assumes Role C. Maximum session duration is capped at **1 hour** when chaining (cannot use the 12-hour maximum). Sessions cannot re-extend.

---

### Cross-Account Role Access

**Pattern:**

```
Account A (trusting) — has the resource
Account B (trusted) — has the principal (IAM user or role)

1. In Account A: Create role "CrossAccountRole"
   Trust policy: allow Account B principal to AssumeRole
   Permission policy: allow access to the resource

2. In Account B: IAM policy allows sts:AssumeRole on the Account A role ARN

3. Principal in B: calls sts:AssumeRole → gets temp creds → accesses resource in A
```

**Resource Access Manager (RAM):**

RAM allows sharing specific AWS resources across accounts **without cross-account role assumption**.

| Resource shareable via RAM | Sharing scope |
|---------------------------|--------------|
| VPC subnets | org, OU, or specific accounts |
| Transit Gateway | org or specific accounts |
| Route 53 Resolver rules | org or specific accounts |
| AWS License Manager configs | accounts |
| Capacity Reservations | org |

Key point: Shared subnets — participant accounts can launch resources into the subnet but **cannot modify the subnet or VPC**. All ENIs and security groups belong to the participant account.

---

### Amazon Cognito

Cognito provides **authentication and authorisation for end-user applications** (mobile, web). Separate from IAM Identity Center which is for developers/employees accessing AWS accounts.

**Two components:**

| Component | Purpose |
|-----------|---------|
| **User Pool** | User directory; handles sign-up, sign-in, MFA, OAuth 2.0 tokens (JWT) |
| **Identity Pool** | Maps authenticated (or guest) users to IAM roles → grants temporary AWS credentials |

**Federation into Identity Pools:**

```
User authenticates with:
  → User Pool (Cognito), or
  → External IdP: Google, Facebook, Apple, SAML, OIDC

→ Receives ID token

Identity Pool validates token →
  Calls STS AssumeRoleWithWebIdentity →
  Returns temporary AWS credentials

User can now call AWS APIs directly (e.g., upload to S3, query DynamoDB)
```

**Fine-grained DynamoDB access with Cognito:**

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
  "Resource": "arn:aws:dynamodb:eu-west-1:123456789012:table/UserData",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
    }
  }
}
```

This policy lets each user access only rows where the partition key equals their Cognito identity ID.

---

## Phase 1 — Key Decision Framework

### "Which connectivity approach?" Decision Tree

```
Need private access to a single VPC from on-premises?
  → Private VIF on Direct Connect → VGW in that VPC

Need private access to multiple VPCs or regions from on-premises?
  → Transit VIF → Direct Connect Gateway → Transit Gateway

Need internet-free access to AWS public services (S3, SQS) from on-premises?
  → Public VIF on Direct Connect

Need temporary/backup connectivity while DX is provisioned?
  → Site-to-Site VPN to VGW or Transit Gateway

Need to connect 10 VPCs together?
  → Transit Gateway (NOT VPC Peering full mesh — that's 45 connections)
```

### "Which identity pattern?" Decision Tree

```
Human access to AWS console/CLI for employees?
  → IAM Identity Center with external IdP (Okta/Azure AD)

Application in GitHub Actions needs AWS access?
  → GitHub OIDC → IAM role (no long-term credentials)

Pod in EKS needs AWS access?
  → IRSA (IAM Roles for Service Accounts)

Mobile/web app users need to call AWS APIs?
  → Cognito User Pool (auth) + Identity Pool (AWS credentials)

Third-party SaaS needs access to your AWS account?
  → Cross-account role with sts:ExternalId
```

---

## Phase 1 — Exam Trap Summary

| Trap | Correct Answer |
|------|---------------|
| "SCP deny on Root OU — does it apply to management account?" | NO. SCPs never apply to the management account |
| "Which VIF for accessing multiple VPCs across regions?" | Transit VIF → Direct Connect Gateway → TGW |
| "VPC Peering: can A reach C via B?" | NO. VPC Peering is not transitive |
| "Gateway endpoint accessible from on-prem?" | NO. Gateway endpoints (S3/DynamoDB) are not accessible via DX or VPN |
| "Interface endpoint accessible from on-prem?" | YES. Interface endpoints have ENIs with private IPs — accessible via DX |
| "RAM shared subnet — who owns the security group?" | Participant account owns the SG and ENIs |
| "Cognito for employee AWS console access?" | NO. Use IAM Identity Center. Cognito is for app end-users |
| "AssumeRole session duration when chaining roles?" | Capped at 1 hour regardless of role's MaxSessionDuration |
| "Which guardrail type uses SCPs?" | Preventive (Detective guardrails use AWS Config rules) |
