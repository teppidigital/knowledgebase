# DevSecOps Patterns

A comprehensive catalog of **DevSecOps patterns** for embedding security throughout the software development lifecycle — from developer workstations to production runtime monitoring.

---

## Pattern Index

| #                                    | Pattern                                     | Category   | SDLC Phase     |
| ------------------------------------ | ------------------------------------------- | ---------- | -------------- |
| [01](01-shift-left-security.md)      | Shift-Left Security                         | Prevention | Code           |
| [02](02-sast.md)                     | Static Application Security Testing (SAST)  | Prevention | Code / Build   |
| [03](03-sca-dependency-scanning.md)  | SCA & Dependency Scanning                   | Prevention | Code / Build   |
| [04](04-dast.md)                     | Dynamic Application Security Testing (DAST) | Detection  | Test / Deploy  |
| [05](05-secret-management.md)        | Secret Management                           | Prevention | All            |
| [06](06-iac-security.md)             | Infrastructure as Code Security             | Prevention | Build / Deploy |
| [07](07-container-security.md)       | Container Security                          | Prevention | Build / Deploy |
| [08](08-supply-chain-sbom.md)        | Supply Chain Security & SBOM                | Prevention | Build          |
| [09](09-zero-trust.md)               | Zero Trust Architecture                     | Prevention | Operate        |
| [10](10-policy-as-code.md)           | Policy as Code                              | Governance | All            |
| [11](11-runtime-security.md)         | Runtime Security Monitoring                 | Detection  | Operate        |
| [12](12-threat-modeling.md)          | Threat Modeling                             | Governance | Plan / Design  |
| [13](13-compliance-as-code.md)       | Compliance as Code                          | Governance | All            |
| [14](14-security-cicd-pipeline.md)   | Security CI/CD Pipeline                     | Automation | Build / Test   |
| [15](15-vulnerability-management.md) | Vulnerability Management                    | Response   | All            |
| [16](16-api-security.md)             | API Security (OWASP API Top 10)             | Prevention | Code / Test    |
| [17](17-penetration-testing-red-teaming.md) | Penetration Testing & Red Teaming    | Validation | Test / Operate |
| [18](18-incident-response.md)        | Incident Response                           | Response   | Operate        |
| [19](19-data-security-privacy.md)    | Data Security & Privacy Engineering         | Prevention | All            |

---

## Patterns by Category

### Prevention — Stop vulnerabilities before they ship

| Pattern                                          | Key Tools                                 |
| ------------------------------------------------ | ----------------------------------------- |
| [Shift-Left Security](01-shift-left-security.md) | detect-secrets, Gitleaks, pre-commit      |
| [SAST](02-sast.md)                               | Semgrep, CodeQL, SonarQube                |
| [SCA](03-sca-dependency-scanning.md)             | Snyk, OWASP Dependency-Check, Renovate    |
| [Secret Management](05-secret-management.md)     | HashiCorp Vault, AWS Secrets Manager, ESO |
| [IaC Security](06-iac-security.md)               | Checkov, tfsec, KICS                      |
| [Container Security](07-container-security.md)   | Trivy, Kyverno, Cosign                    |
| [Supply Chain & SBOM](08-supply-chain-sbom.md)   | Syft, Grype, SLSA, Cosign                 |
| [Zero Trust](09-zero-trust.md)                   | SPIFFE/SPIRE, Istio, OPA                  |
| [API Security](16-api-security.md)               | RESTler, CATS, Spectral, express-rate-limit, OWASP ZAP API mode |
| [Data Security & Privacy](19-data-security-privacy.md) | AES-256-GCM, Faker.js, Zod, tokenisation, GDPR Art. 17/20 |

### Detection — Identify vulnerabilities and threats at runtime

| Pattern                                                    | Key Tools                           |
| ---------------------------------------------------------- | ----------------------------------- |
| [DAST](04-dast.md)                                         | OWASP ZAP, Nuclei                   |
| [Runtime Security Monitoring](11-runtime-security.md)      | Falco, AWS GuardDuty, Tetragon      |
| [Vulnerability Management](15-vulnerability-management.md) | Trivy (registry), Grype, DefectDojo |

### Governance — Enforce standards and maintain compliance

| Pattern                                        | Key Tools                               |
| ---------------------------------------------- | --------------------------------------- |
| [Policy as Code](10-policy-as-code.md)         | OPA, Rego, Conftest                     |
| [Threat Modeling](12-threat-modeling.md)       | STRIDE, Threat Dragon                   |
| [Compliance as Code](13-compliance-as-code.md) | Prowler, AWS Config, kube-bench, InSpec |

### Automation — Orchestrate and streamline security workflows

| Pattern                                                 | Key Tools                        |
| ------------------------------------------------------- | -------------------------------- |
| [Security CI/CD Pipeline](14-security-cicd-pipeline.md) | GitHub Actions, all of the above |

### Validation — Confirm real-world exploitability and detection capability

| Pattern                                                             | Key Tools                                      |
| ------------------------------------------------------------------- | ---------------------------------------------- |
| [Penetration Testing & Red Teaming](17-penetration-testing-red-teaming.md) | Burp Suite, MITRE ATT&CK, TIBER-EU, CBEST, HackerOne |

### Response — Contain, eradicate, and recover from security incidents

| Pattern                               | Key Tools                                      |
| ------------------------------------- | ---------------------------------------------- |
| [Incident Response](18-incident-response.md) | SOAR (Sentinel/EventBridge), Falco, PagerDuty, Runbooks |

---

## SDLC Phase Mapping

```
PLAN          CODE           BUILD          TEST           DEPLOY         OPERATE
────────────────────────────────────────────────────────────────────────────────────
Threat        Shift-Left     IaC Security   DAST           Container      Runtime
Modeling      Security                                     Security       Security
              ─────────────────────────────────────────────────────────────────────
              SAST           Supply Chain   Vuln           Zero Trust     Zero Trust
              ─────────────────────────────────────────────────────────────────────
              SCA            SBOM           Management     Policy as      Policy as
                                                           Code           Code
              ─────────────────────────────────────────────────────────────────────
              Secret         Container                                    Compliance
              Management     Security                                     as Code
              ─────────────────────────────────────────────────────────────────────
                             Security                                     Vuln
                             Pipeline                                     Management
```

---

## Tool Reference Matrix

| Tool                                                                      | Category           | Language          | License                   |
| ------------------------------------------------------------------------- | ------------------ | ----------------- | ------------------------- |
| [Semgrep](https://semgrep.dev)                                            | SAST               | Any               | LGPL-2.1 / Commercial     |
| [CodeQL](https://codeql.github.com)                                       | SAST               | 10+               | Free for OSS / Commercial |
| [SonarQube](https://sonarqube.org)                                        | SAST + Quality     | 25+               | GPL / Commercial          |
| [Snyk](https://snyk.io)                                                   | SCA + Container    | Any               | Commercial                |
| [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) | SCA                | JVM, .NET, Node   | Apache 2.0                |
| [Renovate](https://docs.renovatebot.com)                                  | Dependency Update  | Any               | AGPL                      |
| [Gitleaks](https://gitleaks.io)                                           | Secret Scanning    | Any               | MIT                       |
| [detect-secrets](https://github.com/Yelp/detect-secrets)                  | Secret Scanning    | Any               | Apache 2.0                |
| [OWASP ZAP](https://zaproxy.org)                                          | DAST               | Any               | Apache 2.0                |
| [Nuclei](https://nuclei.projectdiscovery.io)                              | DAST               | Any               | MIT                       |
| [HashiCorp Vault](https://vaultproject.io)                                | Secret Management  | Any               | BSL / Enterprise          |
| [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)            | Secret Management  | Any               | Commercial                |
| [External Secrets Operator](https://external-secrets.io)                  | Secret Sync        | K8s               | Apache 2.0                |
| [Checkov](https://checkov.io)                                             | IaC Security       | TF, K8s, CFn      | Apache 2.0                |
| [tfsec](https://tfsec.dev)                                                | IaC Security       | Terraform         | MIT                       |
| [Trivy](https://trivy.dev)                                                | Container + IaC    | Any               | Apache 2.0                |
| [Grype](https://github.com/anchore/grype)                                 | Container SCA      | Any               | Apache 2.0                |
| [Syft](https://github.com/anchore/syft)                                   | SBOM Generation    | Any               | Apache 2.0                |
| [Cosign](https://sigstore.dev)                                            | Image Signing      | Any               | Apache 2.0                |
| [Kyverno](https://kyverno.io)                                             | Policy (K8s)       | Kubernetes        | Apache 2.0                |
| [OPA / Gatekeeper](https://openpolicyagent.org)                           | Policy as Code     | Any               | Apache 2.0                |
| [SPIFFE/SPIRE](https://spiffe.io)                                         | Workload Identity  | Any               | Apache 2.0                |
| [Istio](https://istio.io)                                                 | Service Mesh mTLS  | K8s               | Apache 2.0                |
| [Falco](https://falco.org)                                                | Runtime Security   | Linux / K8s       | Apache 2.0                |
| [AWS GuardDuty](https://aws.amazon.com/guardduty/)                        | Threat Detection   | AWS               | Commercial                |
| [Prowler](https://prowler.com)                                            | Compliance Scan    | AWS / Azure / GCP | Apache 2.0                |
| [kube-bench](https://github.com/aquasecurity/kube-bench)                  | CIS K8s Benchmark  | Kubernetes        | Apache 2.0                |
| [Chef InSpec](https://inspec.io)                                          | Compliance as Code | Any               | Apache 2.0                |

---

## Pattern Combination Guide

### "Secure Development Pipeline"

**Goal**: Catch 95%+ of vulnerabilities before they reach production.

```
Shift-Left (01) → SAST (02) → SCA (03) → Secret Management (05) → Security Pipeline (14)
```

### "Zero Trust Microservices"

**Goal**: Defense-in-depth for internal service communication.

```
Zero Trust (09) → Policy as Code (10) → Container Security (07) → Runtime Security (11)
```

### "Compliant Cloud Infrastructure"

**Goal**: Continuous regulatory compliance for cloud environments.

```
IaC Security (06) → Compliance as Code (13) → Policy as Code (10) → Vulnerability Management (15)
```

### "Software Supply Chain Hardening"

**Goal**: Ensure only verified, signed software is deployed.

```
SCA (03) → Supply Chain & SBOM (08) → Container Security (07) → Policy as Code (10)
```

### "Threat-Driven Security"

**Goal**: Start with threat modeling, then systematically address each threat.

```
Threat Modeling (12) → Shift-Left (01) → SAST (02) → DAST (04) → Runtime Security (11)
```

---

## Decision Guide: Which Pattern First?

```
Are you just starting?
├── YES → Start with Shift-Left Security (01) — lowest friction, immediate value
└── NO
    ├── Are you shipping containers?
    │   └── YES → Container Security (07) + Supply Chain (08)
    ├── Are you on Kubernetes?
    │   └── YES → Container Security (07) + Policy as Code (10) + Runtime Security (11)
    ├── Using cloud infrastructure (AWS/Azure/GCP)?
    │   └── YES → IaC Security (06) + Compliance as Code (13)
    ├── Have compliance requirements (SOC 2, PCI, ISO 27001)?
    │   └── YES → Compliance as Code (13) + Policy as Code (10)
    └── Building microservices?
        └── YES → Zero Trust (09) + Secret Management (05)
```

---

## Severity and SLA Reference

| CVSS Score | Severity      | Remediation SLA | Escalation     |
| ---------- | ------------- | --------------- | -------------- |
| 9.0–10.0   | Critical      | **24 hours**    | Immediate page |
| 7.0–8.9    | High          | **7 days**      | Slack alert    |
| 4.0–6.9    | Medium        | **30 days**     | Jira ticket    |
| 0.1–3.9    | Low           | **90 days**     | Backlog        |
| N/A        | Informational | Best effort     | —              |

---

## Related Pattern Libraries

- [System Design Patterns](../system-design/README.md)
- [Distributed Design Patterns](../distributed-design-pattern/README.md)
