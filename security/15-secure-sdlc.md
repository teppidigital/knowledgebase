# Secure Software Development Lifecycle (SSDLC)

## Category
Security, SSDLC, Secure Design, Security Requirements, Threat Review, Security Testing, Release Gates

## Context

**Secure Software Development Lifecycle (SSDLC)** embeds security activities into each phase of the software development lifecycle — from requirements to deployment — rather than bolting security on at the end. It is the process-level framework that makes security engineering a first-class concern alongside feature delivery.

SSDLC differs from DevSecOps tooling: DevSecOps automates security controls in the pipeline; SSDLC defines the **human processes, artefacts, and gates** that govern secure design and sign-off.

### SSDLC phases

| Phase | Security activities | Artefacts |
|-------|--------------------|-----------| 
| **Requirements** | Threat-aware requirements; abuse cases; regulatory mapping (GDPR, PCI) | Security requirements; STRIDE abuse cases |
| **Design** | Threat modelling (STRIDE, Attack Trees); security architecture review; DFD | Threat model; risk register; architecture decision records (ADRs) |
| **Implementation** | Secure coding standards; mandatory peer review checklist; SAST in IDE | Secure coding guide; code review checklist |
| **Testing** | SAST + SCA in CI; DAST in staging; unit tests for security controls | Pipeline scan reports; DAST results |
| **Release** | Security sign-off gate; SBOM generated; secrets scanned; pentest clearance | Release security checklist; pentest sign-off |
| **Operations** | Patch management; CVE monitoring; incident response readiness | Vulnerability SLA; runbooks |
| **Decommission** | Secure data erasure; credential revocation; infra teardown | Decommission checklist |

### Security requirements patterns

Good security requirements are **testable** and **threat-driven**:

| Bad (vague) | Good (testable + threat-driven) |
|------------|----------------------------------|
| "The system shall be secure" | "All API endpoints require a valid JWT issued by the corporate IdP" |
| "User data shall be protected" | "User PII shall be encrypted using AES-256-GCM with a CMK; field-level encryption for email and phone" |
| "The system shall handle errors gracefully" | "Error responses shall never include stack traces, database names, or internal IP addresses" |
| "Authentication shall be strong" | "All admin users shall be required to complete MFA using TOTP or FIDO2; passwords shall be hashed with Argon2id" |

### Security story acceptance criteria template

```gherkin
Feature: JWT Authentication
  As a security architect
  I want to ensure all API calls are authenticated with a valid JWT
  So that unauthenticated users cannot access protected resources

  Scenario: Missing JWT is rejected
    Given a request to GET /api/orders
    When no Authorization header is present
    Then the response status is 401
    And the response body contains { "error": "Authentication required" }
    And the response body does NOT contain internal system details

  Scenario: Expired JWT is rejected
    Given a JWT that expired 1 second ago
    When the client sends the expired token
    Then the response status is 401
    And the audit log records { action: "AUTH_FAILURE", reason: "token_expired" }
```

### Security sign-off gate (pre-production checklist)

A lightweight gate that must be passed before a release reaches production:

| Check | Owner | Pass criterion |
|-------|-------|---------------|
| SAST scan | CI/CD | Zero Critical/High unresolved findings |
| SCA scan | CI/CD | Zero Critical CVEs in direct dependencies |
| Secrets scan | CI/CD | No detected secrets in committed code |
| Security story coverage | Developer | All security ACs have passing automated tests |
| Architecture change review | Security Architect | No significant security-relevant changes without review |
| DAST results | QA / Security | No Critical/High findings in staging |
| Pentest clearance | Security | Latest pentest findings addressed or risk-accepted |

---

## Pros

- **Prevents vulnerabilities before they are built in**: Threat modelling at design phase is 10× cheaper to fix than in production.
- **Creates shared security ownership**: Developers, architects, and PMs all participate in security — not just a security team.
- **Measurable improvement**: Security KPIs (mean time to fix, vulnerability density by team) drive continuous improvement.
- **Evidence for auditors**: SSDLC artefacts (threat models, checklists, sign-offs) provide clear evidence for SOC 2, ISO 27001, PCI-DSS audits.
- **Regulatory alignment**: GDPR Article 25 (Privacy by Design) and 32 (appropriate technical measures) are satisfied by documented SSDLC.

---

## Cons

- **Adds friction to delivery**: Security gates and reviews add time — requires careful implementation to avoid becoming a bottleneck.
- **Requires security expertise at scale**: Embedded security champions in each team are needed — hard to build this capacity quickly.
- **Threat modelling is hard to automate**: It requires human expertise and contextual knowledge — cannot be fully replaced by tools.
- **Risk of checkbox culture**: Teams can go through the motions without genuine engagement — requires cultural investment.
- **Gates without teeth are ignored**: A release gate that is routinely bypassed or rubber-stamped provides false assurance.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph Sprint 0 — Design
        A[Security Requirements\nAbuse cases, regulatory mapping] --> B[Threat Modelling\nSTRIDE on DFD]
        B --> C[Security Architecture Review\nADRs, risk register]
    end

    subgraph Sprint N — Dev
        D[Security Story ACs\nGherkin + automated tests]
        E[Secure Code Review\nChecklist: injection, auth, secrets]
        F[SAST in IDE\nSonarQube / Semgrep]
    end

    subgraph CI Pipeline
        G[SAST scan\nSemgrep / CodeQL]
        H[SCA scan\nSnyk / Dependabot]
        I[Secrets scan\ngit-leaks / detect-secrets]
        J{All gates pass?}
        G & H & I --> J
    end

    subgraph Staging
        K[DAST — ZAP / nuclei]
        L[Integration security tests]
    end

    subgraph Release Gate
        M[Security sign-off\nchecklist]
        N{Approved?}
        M --> N
        N -->|Yes| O[Production deploy]
        N -->|No| P[Fix + re-review]
    end

    C --> D --> E --> F --> G
    J -->|Pass| K
    K --> L --> M
```

---

## Code Sample

### TypeScript — Security Acceptance Tests (Jest)

```typescript
// tests/security/authentication.security.test.ts
// Automated security acceptance criteria — run in CI against staging

import supertest from 'supertest';
import { generateExpiredJwt, generateValidJwt, generateJwtWithoutScope } from './helpers.js';

const api = supertest(process.env.API_BASE_URL ?? 'http://localhost:3000');

describe('Security AC: Authentication & Authorisation (JWT)', () => {

  describe('Missing JWT', () => {
    it('returns 401 when no Authorization header is sent', async () => {
      const res = await api.get('/api/orders');
      expect(res.status).toBe(401);
      expect(res.body.error).toBe('Authentication required');
    });

    it('does not leak internal details in 401 response', async () => {
      const res = await api.get('/api/orders');
      const body = JSON.stringify(res.body);
      // Must not reveal stack traces, DB info, internal IPs
      expect(body).not.toMatch(/at .+\.js:\d+/);         // Stack trace
      expect(body).not.toMatch(/postgresql|mongodb|mysql/i); // DB name
      expect(body).not.toMatch(/10\.\d+\.\d+\.\d+/);     // Internal IP
    });
  });

  describe('Expired JWT', () => {
    it('returns 401 for expired token', async () => {
      const expiredToken = await generateExpiredJwt();
      const res = await api.get('/api/orders').set('Authorization', `Bearer ${expiredToken}`);
      expect(res.status).toBe(401);
    });
  });

  describe('Insufficient scope', () => {
    it('returns 403 when JWT lacks required scope', async () => {
      const tokenWithoutScope = await generateJwtWithoutScope('orders:read');
      const res = await api.get('/api/orders').set('Authorization', `Bearer ${tokenWithoutScope}`);
      expect(res.status).toBe(403);
    });
  });

  describe('IDOR / BOLA prevention', () => {
    it('returns 404 (not 403) when accessing another user\'s order', async () => {
      // Token for user A, but requesting user B's order ID
      const userAToken = await generateValidJwt({ sub: 'user-A', scope: 'orders:read' });
      const userBOrderId = 'order-owned-by-user-B';

      const res = await api
        .get(`/api/orders/${userBOrderId}`)
        .set('Authorization', `Bearer ${userAToken}`);

      // 404 — do not leak that the resource exists (prevents enumeration)
      expect(res.status).toBe(404);
      // 403 would reveal the resource exists — that's an IDOR information leak
    });
  });
});

describe('Security AC: Transport Security', () => {
  it('includes HSTS header with min 1-year max-age', async () => {
    const res = await api.get('/');
    const hsts = res.headers['strict-transport-security'];
    expect(hsts).toBeDefined();
    const maxAge = parseInt(hsts.match(/max-age=(\d+)/)?.[1] ?? '0', 10);
    expect(maxAge).toBeGreaterThanOrEqual(31_536_000);
  });

  it('includes Content-Security-Policy header', async () => {
    const res = await api.get('/');
    expect(res.headers['content-security-policy']).toBeDefined();
  });

  it('does not include X-Powered-By header', async () => {
    const res = await api.get('/');
    expect(res.headers['x-powered-by']).toBeUndefined();
  });
});

describe('Security AC: Error Handling', () => {
  it('returns generic error message for server errors — no stack trace', async () => {
    // Trigger a 500 by hitting a known error-inducing endpoint (dev-only)
    const token = await generateValidJwt({ sub: 'user-A', scope: 'admin' });
    const res = await api.post('/api/internal/trigger-error').set('Authorization', `Bearer ${token}`);
    if (res.status === 500) {
      expect(res.body).not.toHaveProperty('stack');
      expect(JSON.stringify(res.body)).not.toMatch(/at .+\.js:\d+/);
    }
  });
});
```

### Markdown — Security Review Checklist (ADR Gate)

```markdown
# Security Review Checklist — Architecture Decision Record

Use this checklist when raising an ADR that involves security-relevant changes.

## Authentication & Authorisation
- [ ] How are users/services authenticated? (JWT, mTLS, API key, session cookie)
- [ ] How is authorisation enforced? (RBAC, ABAC, policy-as-code)
- [ ] Are all endpoints protected by default? (deny-all, explicit allow)
- [ ] Does the design prevent IDOR/BOLA? (ownership check server-side)

## Data Handling
- [ ] What data classifications does this feature process? (Public / Internal / Confidential / Restricted)
- [ ] Is PII encrypted at rest? (field-level, volume-level)
- [ ] Is PII encrypted in transit? (TLS 1.2+)
- [ ] Is PII logged? (if yes, is it masked?)
- [ ] Does the design include a right-to-erasure path?

## Secrets & Credentials
- [ ] Where are secrets stored? (Vault / Key Vault — not env vars or code)
- [ ] Are credentials short-lived and automatically rotated?
- [ ] Is any secret hardcoded or committed to source? (answer must be No)

## Network
- [ ] Is the service exposed externally? If so — is there a WAF?
- [ ] Are network policies / NSGs configured (deny by default)?
- [ ] Is mTLS used for service-to-service communication?

## Third-Party Dependencies
- [ ] Are new third-party libraries vetted for known CVEs?
- [ ] Are new sub-processors GDPR-compliant? (DPA in place?)

## Threat Model
- [ ] Has a STRIDE threat model been updated for this change?
- [ ] Are all identified threats mitigated or risk-accepted?

## Approval
- [ ] Security architect sign-off: __________________
- [ ] Privacy officer (if PII-involved): __________________
- [ ] Sign-off date: __________________
```

### YAML — Semgrep SSDLC Rules (Custom secure coding checks)

```yaml
# .semgrep/ssdlc-rules.yaml
# Custom Semgrep rules enforcing SSDLC secure coding standards

rules:
  - id: no-console-log-sensitive-data
    patterns:
      - pattern: console.log(..., $X, ...)
      - metavariable-regex:
          metavariable: $X
          regex: .*(password|secret|token|key|credential|ssn|card).*
    message: "Potential sensitive data in console.log — use structured audit logger"
    languages: [typescript, javascript]
    severity: ERROR

  - id: no-string-format-sql
    patterns:
      - pattern: |
          `... ${...} ...`
      - pattern-inside: |
          $DB.query($SQL, ...)
    message: "SQL query uses template literal — use parameterised query with $1, $2 placeholders"
    languages: [typescript, javascript]
    severity: ERROR

  - id: no-math-random-security
    pattern: Math.random()
    message: "Math.random() is not cryptographically secure — use crypto.randomBytes() for security tokens"
    languages: [typescript, javascript]
    severity: WARNING

  - id: no-md5-sha1-password
    patterns:
      - pattern: crypto.createHash('md5')
      - pattern: crypto.createHash('sha1')
    message: "MD5 and SHA-1 are cryptographically broken — use SHA-256 or stronger; for passwords use Argon2id"
    languages: [typescript, javascript]
    severity: ERROR

  - id: no-eval
    pattern: eval(...)
    message: "eval() enables code injection — remove or replace with a safe alternative"
    languages: [typescript, javascript]
    severity: ERROR
```
