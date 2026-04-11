# AWS Identity Federation Patterns

## Category
Cloud Native, Security, Identity, AWS IAM Identity Center, SAML, OIDC, Cognito

## Context

**Identity federation** allows external identity systems to authenticate users or workloads and grant them access to AWS resources using temporary credentials — without creating IAM users. It addresses two distinct use cases:

1. **Workforce identity:** Employees/developers accessing AWS accounts, console, or CLI.
2. **Workload identity:** Applications, CI/CD pipelines, and Kubernetes pods assuming AWS IAM roles without long-term credentials.

The choice of federation mechanism depends on who or what is being authenticated, the existing identity infrastructure, and the required access granularity.

---

## IAM Identity Center (SSO)

### Overview

IAM Identity Center (formerly AWS SSO) is the **recommended approach for workforce access** to AWS accounts. It provides centralised access management across all accounts in an AWS Organisation.

### Components

| Component | Description |
|-----------|-------------|
| **Identity source** | Where users and groups live: Identity Center built-in directory, AWS Managed Microsoft AD, or external IdP (Okta, Azure AD, Google Workspace) |
| **Permission set** | Bundle of IAM managed policies + inline policy + session duration. Becomes an IAM role in each assigned account |
| **Account assignment** | Maps user or group + permission set → specific AWS account. Many-to-many |
| **Identity Center application** | OAuth 2.0 / SAML 2.0 application assignments for non-AWS apps |

### Access Flow

```
Employee → IdP (Okta/Azure AD/ADFS) → SAML assertion → IAM Identity Center
  → Identity Center portal (choose account + permission set)
  → STS AssumeRoleWithSAML → Temporary credentials (15 min to 12 hours)
  → AWS Console or CLI access
```

### SCIM Provisioning

SCIM (System for Cross-domain Identity Management) automates user/group sync from an external IdP to IAM Identity Center:
- New employees added to Okta group → automatically provisioned in Identity Center.
- Terminated employees deactivated in Okta → automatically deprovisioned in Identity Center.
- Without SCIM: users must be manually added and removed.

### Permission Set Implementation

Each permission set creates an IAM role in every assigned account:
- Role name: `AWSReservedSSO_<PermissionSetName>_<RandomHex>`
- Trust policy: allows the `signin.aws.amazon.com` principal to assume the role via SAML.
- These roles should NOT be modified directly — manage via Identity Center.

### Multi-Account Permission Model

```
Permission Set: "DeveloperAccess"
  → Attached policies: AmazonEC2FullAccess, AmazonS3ReadOnlyAccess
  → Assigned to groups: ["developers"] → accounts: [dev-account, staging-account]

Permission Set: "ReadOnlyAccess"
  → Attached policies: ReadOnlyAccess
  → Assigned to groups: ["all-staff"] → accounts: [prod-account] (console read-only)
```

---

## SAML 2.0 Federation

### How SAML Works

SAML 2.0 is an XML-based open standard for exchanging authentication and authorisation data between an **Identity Provider (IdP)** and a **Service Provider (SP)**.

**Key parties:**
- **IdP (Identity Provider):** Okta, Azure AD, ADFS, OneLogin — authenticates the user and issues assertions.
- **SP (Service Provider):** AWS — consumes the assertion to grant access.
- **SAML Assertion:** Signed XML document containing user attributes and the roles they are authorised to assume.

### SP-Initiated vs IdP-Initiated

| | SP-Initiated | IdP-Initiated |
|--|-------------|-------------|
| Entry point | User navigates to AWS signin URL | User clicks app tile in IdP portal |
| Flow | AWS → redirect to IdP → authenticate → SAML assertion → back to AWS | IdP → SAML assertion → directly to AWS |
| More common | Yes (browser-based AWS Console) | Yes (applications in enterprise portals) |

### SP-Initiated SAML Flow

```
1. User visits: https://signin.aws.amazon.com/saml
2. AWS redirects to IdP login page (AuthnRequest)
3. User authenticates at IdP (password/MFA)
4. IdP issues SAML assertion (signed XML) with:
   - User attributes (email, groups)
   - AWS attribute: Role ARN + SAML Provider ARN
5. Browser POSTs assertion to: https://signin.aws.amazon.com/saml
6. AWS validates assertion signature against SAML provider certificate
7. STS AssumeRoleWithSAML → temporary credentials
8. User lands in AWS Console in the assumed role
```

### IAM Configuration for SAML

**Step 1 — Create SAML provider:**
```bash
aws iam create-saml-provider \
  --saml-metadata-document file://metadata.xml \
  --name MyCorpIdP
```

**Step 2 — Create IAM role with SAML trust:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:saml-provider/MyCorpIdP"
    },
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": {
      "StringEquals": {
        "SAML:aud": "https://signin.aws.amazon.com/saml"
      }
    }
  }]
}
```

**Step 3 — Configure IdP attribute mapping:**
The IdP must send AWS-specific SAML attributes in the assertion:
```xml
<Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
  <AttributeValue>
    arn:aws:iam::123456789012:role/MyRole,arn:aws:iam::123456789012:saml-provider/MyCorpIdP
  </AttributeValue>
</Attribute>
<Attribute Name="https://aws.amazon.com/SAML/Attributes/SessionDuration">
  <AttributeValue>28800</AttributeValue>  <!-- 8 hours, max 43200 (12h) -->
</Attribute>
```

---

## OIDC Federation

### OIDC vs SAML

| | SAML 2.0 | OIDC |
|--|---------|------|
| Format | XML assertion | JWT (JSON Web Token) |
| Primary use | Human SSO to web apps | Workload identity (machines, APIs) |
| Common use on AWS | Human console/CLI access (via Identity Center) | GitHub Actions, EKS IRSA, Lambda |
| Standard | OASIS | OpenID Foundation (built on OAuth 2.0) |

### GitHub Actions OIDC

GitHub Actions can assume IAM roles without storing any long-term AWS credentials in the repo.

**Setup:**
1. Create OIDC provider in the AWS account:
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <thumbprint>
```

2. Create IAM role with OIDC trust:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
      }
    }
  }]
}
```

3. GitHub Actions workflow:
```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubDeployRole
          aws-region: eu-west-1
      - run: aws s3 cp ./build s3://my-bucket/ --recursive
```

**Security note:** Use the `sub` condition to restrict which repository, branch, or environment can assume the role. Without it, any GitHub repository could assume the role by knowing the ARN.

---

### IRSA — IAM Roles for Service Accounts (EKS)

IRSA gives Kubernetes pods **pod-level IAM permissions** using OIDC — no node-level IAM roles needed, no credential sharing across pods.

**How it works:**
```
1. EKS cluster has an OIDC provider endpoint (unique per cluster)
2. IAM role's trust policy scopes access to a specific K8s namespace/ServiceAccount
3. Pod's ServiceAccount annotated with the role ARN
4. EKS injects projected token into pod as volume mount
5. AWS SDK in pod uses token to call STS AssumeRoleWithWebIdentity
6. Returns temporary credentials scoped to the role's permissions
```

**ServiceAccount setup:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: data-processing
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3ReaderRole
```

**IAM role trust policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLECLUSTERID"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLECLUSTERID:sub":
          "system:serviceaccount:data-processing:s3-reader",
        "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLECLUSTERID:aud": "sts.amazonaws.com"
      }
    }
  }]
}
```

**IRSA vs Node Instance Profile:**

| | IRSA | Node Instance Profile |
|--|------|----------------------|
| Scope | Per-pod | All pods on that node |
| Least privilege | Yes | No (all pods share same permissions) |
| Audit trail | Per-pod CloudTrail entries | All attributed to the node role |
| Recommendation | Always use IRSA | Only for DaemonSets or system components |

---

## AWS STS — Temporary Credentials

### STS API Reference

| API | Principal | Use case |
|-----|-----------|---------|
| `AssumeRole` | IAM user, IAM role, or AWS service | Cross-account access; role chaining; service-to-service |
| `AssumeRoleWithSAML` | Any (via IdP) | Human SSO via SAML 2.0 |
| `AssumeRoleWithWebIdentity` | Any (via OIDC IdP) | Workload identity via OIDC |
| `GetSessionToken` | IAM user | Enforce MFA (user provides MFA token) |
| `GetFederationToken` | IAM user or root | Custom federation broker (legacy) |

### Session Duration

| API | Minimum | Maximum |
|-----|---------|---------|
| `AssumeRole` (standard) | 15 min | 12 hours (or role's `MaxSessionDuration`) |
| `AssumeRole` (role chaining) | 15 min | **1 hour** (hard limit — cannot extend) |
| `AssumeRoleWithSAML` | 15 min | 12 hours |
| `AssumeRoleWithWebIdentity` | 15 min | 12 hours |
| `GetSessionToken` | 15 min | 36 hours |
| `GetFederationToken` | 15 min | 36 hours |

### ExternalId — Confused Deputy Prevention

Without ExternalId, a malicious SaaS vendor could use your account ID and role ARN to access your resources (if they have their own AWS account that your role trusts).

```json
// Role trust policy (in your account, the resource owner)
{
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "abc123-unique-per-customer-id"
    }
  }
}
```

The SaaS vendor generates a unique ExternalId for each customer. You include it in the trust policy. The vendor passes it when calling AssumeRole. A malicious actor cannot discover this value without you telling them.

---

## Amazon Cognito

### Cognito User Pools vs Identity Pools

| | User Pool | Identity Pool |
|--|-----------|--------------|
| Purpose | User directory and authentication | Maps authenticated identities to IAM roles |
| Output | JWT tokens (ID token, access token, refresh token) | Temporary AWS credentials (STS) |
| Who uses it | Your application (auth layer) | Your application (AWS API access) |
| Federated IdPs | Google, Facebook, Apple, SAML, OIDC | User Pool tokens, Google, Facebook, SAML, OIDC, developer authenticated |
| Use case | Login/signup; OAuth 2.0 flows; MFA; token vending | Let end users call AWS APIs (S3, DynamoDB) directly |

### Combined Pattern (most common):

```
Mobile app user → Cognito User Pool (sign in with email/password or social login)
  → Receives ID token (JWT)
    → Presents to Cognito Identity Pool
      → Identity Pool validates token → calls STS AssumeRoleWithWebIdentity
        → Returns temporary AWS credentials (15 min – 1 hour)
          → App calls S3/DynamoDB directly with credentials
```

### Fine-Grained DynamoDB Access per User

Use `${cognito-identity.amazonaws.com:sub}` as a policy variable to scope DynamoDB access to the authenticated user's identity ID:

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:eu-west-1:123456789012:table/UserProfiles",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
    }
  }
}
```

### Guest (Unauthenticated) Access

Cognito Identity Pools support guest access — unauthenticated users get a separate, restricted IAM role. Useful for: anonymous analytics, public content delivery, read-only access before sign-in.

---

## Cross-Account Role Assumption

### Pattern

```
Account B (Trusted — has the principal)           Account A (Trusting — has the resource)
┌──────────────────────────────────┐              ┌─────────────────────────────────────────┐
│  IAM Principal (user/role)       │              │  IAM Role: CrossAccountRole              │
│  Permission: sts:AssumeRole      │──────────────│  Trust policy: Principal = Account B     │
│  on CrossAccountRole ARN         │  AssumeRole  │  Permission: s3:GetObject on bucket      │
└──────────────────────────────────┘              └─────────────────────────────────────────┘
```

**Step 1 — Account A (resource owner):** Create IAM role with trust policy:
```json
{
  "Principal": {"AWS": "arn:aws:iam::B_ACCOUNT_ID:role/DeployRole"},
  "Action": "sts:AssumeRole"
}
```

**Step 2 — Account B (principal):** Grant the principal permission to call AssumeRole:
```json
{
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::A_ACCOUNT_ID:role/CrossAccountRole"
}
```

**Step 3 — Runtime:**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::A_ACCOUNT_ID:role/CrossAccountRole \
  --role-session-name "my-session"
```

---

## Resource Access Manager (RAM)

RAM shares specific AWS resource types across accounts without cross-account role assumption.

**Shareable resources (exam-relevant):**

| Resource | Share scope |
|----------|------------|
| VPC subnets | Org, OU, specific accounts |
| Transit Gateway | Org, OU, specific accounts |
| Route 53 Resolver rules | Org, OU, specific accounts |
| AWS License Manager configs | Accounts |
| Capacity Reservations | Accounts |
| Aurora global clusters | Org |

**VPC subnet sharing:** Participants launch resources (EC2, RDS, Lambda) into the shared subnet. They own their ENIs and security groups. They do NOT control the subnet/VPC itself. The owner account controls the VPC infrastructure.

**Key use case:** Centralise VPC networking (Transit Gateway, subnets routing) in a Network account. Workload accounts use shared subnets to launch their services — no VPC peering needed, and networking is managed centrally.

---

## Pros

- **IAM Identity Center:** Centralised access management across all accounts; no credential management; SSO user experience; CloudTrail logs per-account.
- **SAML federation:** Leverages existing enterprise IdP investment; no user sync needed (assertion-based).
- **OIDC/IRSA:** No long-term credentials anywhere; pod-level isolation in EKS; works natively with GitHub Actions.
- **Cognito:** Managed user directory and federation for customer-facing applications; scales to millions of users; handles token issuance, refresh, and revocation.

---

## Cons

- **SAML (managing roles per account):** Without Identity Center, you must manage a separate SAML provider and role in each account + update IdP attribute maps for each new account.
- **OIDC (trust policy management):** Every new repository/cluster requires trust policy updates; condition strings must be carefully scoped or any OIDC issuer could assume the role.
- **Cognito (advanced flows):** Hosted UI defaults are basic; custom UI requires significant front-end work; JWT validation libraries must be used correctly to avoid token spoofing.
- **Role chaining:** Maximum session duration is 1 hour regardless of role's MaxSessionDuration — avoid chaining more than 2 levels.

---

## Decision Matrix

| Scenario | Recommended Pattern |
|----------|-------------------|
| Employees need AWS Console/CLI access | IAM Identity Center + external IdP |
| GitHub Actions CI/CD needs AWS access | GitHub OIDC provider → IAM role |
| EKS pod needs to write to S3 | IRSA (IAM Roles for Service Accounts) |
| Lambda function needs DynamoDB access | Lambda execution role (no federation needed) |
| Mobile app user needs to upload to S3 | Cognito User Pool + Identity Pool |
| SaaS vendor needs read access to your S3 | Cross-account role with sts:ExternalId |
| Share VPC subnets across accounts | Resource Access Manager (RAM) |
| Third-party auth (Google login) for app | Cognito User Pool (social federation) |
| Custom SAML app (not AWS console) | IAM Identity Center Application assignment |

---

## Common Mistakes

- Using Cognito for **employee** AWS console access — Cognito is for **application end-users**. Use IAM Identity Center for workforce access.
- Forgetting ExternalId when permitting a third-party to assume a role — confused deputy risk.
- Not scoping the `sub` claim in OIDC trust policies — allows any repository/pod from that OIDC issuer to assume the role.
- Setting SAML session duration > 1 hour without updating the role's `MaxSessionDuration` — the role will reject the assertion.
- Creating IAM users for CI/CD pipelines — always use OIDC or instance profiles instead.
- Managing IAM users per developer in each account — use IAM Identity Center instead to centralise access management.
