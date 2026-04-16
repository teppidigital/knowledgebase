# Security Patterns

Comprehensive security architecture, protocol, and implementation patterns for modern cloud-native applications. This section covers the full security stack — from authentication protocols and encryption through to incident response and secure development practices.

> **Relationship to DevSecOps**: The [DevSecOps section](../devsecops/) covers security *pipeline controls* (SAST, SCA, DAST, IaC scanning, secret detection in CI/CD). This section covers security *architecture and implementation patterns* — the protocols, cryptographic constructs, network controls, and operational processes that protect systems at runtime.

---

## Pattern Index

| # | Pattern | Category | Standards / Frameworks |
|---|---------|----------|----------------------|
| [01](./01-oauth2-oidc-jwt.md) | OAuth 2.0, OIDC & JWT | Authentication & Token Security | OAuth 2.0 RFC 6749, OIDC Core, RFC 7519 |
| [02](./02-api-security.md) | API Security | API Protection | OWASP API Top 10 (2023) |
| [03](./03-encryption-patterns.md) | Encryption Patterns | Cryptography | NIST SP 800-57, AES-256-GCM, FIPS 140-2 |
| [04](./04-network-security.md) | Network Security & Defence in Depth | Network | Zero Trust (NIST SP 800-207), mTLS, SPIFFE |
| [05](./05-identity-federation-sso.md) | Identity Federation & SSO | Identity | SAML 2.0, OIDC, SCIM 2.0 |
| [06](./06-data-privacy-pii.md) | Data Privacy & PII | Data Protection | GDPR, CCPA, LGPD |
| [07](./07-security-logging-siem.md) | Security Logging & SIEM | Detection & Monitoring | OWASP Logging Cheat Sheet, SOC 2 CC7 |
| [08](./08-ddos-rate-limiting.md) | DDoS & Rate Limiting | Availability | OWASP API Security, RFC 6585 |
| [09](./09-web-app-security.md) | Web Application Security | Application | OWASP Top 10 (2021), CSP Level 3 |
| [10](./10-secrets-management.md) | Secrets Management & Rotation | Credential Security | NIST SP 800-57, HashiCorp Vault, Azure Key Vault |
| [11](./11-pki-certificate-management.md) | PKI & Certificate Management | Cryptography | RFC 5280, ACME RFC 8555, cert-manager |
| [12](./12-incident-response.md) | Security Incident Response | Operations | NIST SP 800-61, SOAR, GDPR Art. 33 |
| [13](./13-penetration-testing.md) | Penetration Testing & Attack Surface | Assurance | PTES, OWASP Testing Guide, CVSS v3.1 |
| [14](./14-data-loss-prevention.md) | Data Loss Prevention (DLP) | Data Protection | GDPR Art. 25, PCI-DSS Req. 3, Microsoft Purview |
| [15](./15-secure-sdlc.md) | Secure Software Development Lifecycle | Process | OWASP SAMM, NIST SSDF, Microsoft SDL |
| [16](./16-ai-llm-security.md) | AI & LLM Security | Application / AI | OWASP LLM Top 10 (2025), MITRE ATLAS, NIST AI RMF |
| [17](./17-mobile-security.md) | Mobile Security | Application / Mobile | OWASP MASVS v2, OWASP MASTG, Play Integrity API, Apple App Attest |
| [18](./18-third-party-vendor-risk.md) | Third-Party & Vendor Risk Management | Assurance / Supply Chain | GDPR Art. 28, PCI-DSS Req. 12.8, DORA Art. 28–30, NIST SP 800-161 |

---

## Patterns by Category

### Authentication & Identity
- [01 — OAuth 2.0, OIDC & JWT](./01-oauth2-oidc-jwt.md) — Token-based authentication, PKCE, M2M flows
- [05 — Identity Federation & SSO](./05-identity-federation-sso.md) — SAML 2.0, OIDC federation, SCIM provisioning

### Authorisation & Access Control
- [02 — API Security](./02-api-security.md) — BOLA/IDOR prevention, scope enforcement
- [09 — Web Application Security](./09-web-app-security.md) — CSRF, clickjacking, broken access control

### Cryptography
- [03 — Encryption Patterns](./03-encryption-patterns.md) — Envelope encryption, field-level encryption, CMK, cert-manager
- [10 — Secrets Management & Rotation](./10-secrets-management.md) — Vault dynamic secrets, Key Vault, auto-rotation
- [11 — PKI & Certificate Management](./11-pki-certificate-management.md) — CA hierarchy, ACME, cert-manager, Vault PKI

### Network Security
- [04 — Network Security & Defence in Depth](./04-network-security.md) — Zero trust, mTLS, SPIFFE, Istio, Cilium, NetworkPolicy
- [08 — DDoS & Rate Limiting](./08-ddos-rate-limiting.md) — L3/L4/L7 DDoS, WAF, sliding window rate limiting

### Application Security
- [02 — API Security](./02-api-security.md) — OWASP API Top 10 mitigations
- [09 — Web Application Security](./09-web-app-security.md) — XSS, SQLi, CSRF, CSP nonces, security headers, SRI
- [16 — AI & LLM Security](./16-ai-llm-security.md) — Prompt injection, output sanitisation, LLM trust boundaries, agent least-privilege

### Mobile Security
- [17 — Mobile Security](./17-mobile-security.md) — OWASP MASVS, Keychain/Keystore, certificate pinning, root detection, biometric auth

### Data Protection & Privacy
- [03 — Encryption Patterns](./03-encryption-patterns.md) — Encryption at rest, in transit, key management
- [06 — Data Privacy & PII](./06-data-privacy-pii.md) — GDPR rights, tokenisation, right to erasure, consent management
- [14 — Data Loss Prevention](./14-data-loss-prevention.md) — Data classification, DLP scanning, egress control

### Detection & Response
- [07 — Security Logging & SIEM](./07-security-logging-siem.md) — Audit logging, HMAC chaining, SIEM forwarding
- [12 — Security Incident Response](./12-incident-response.md) — IR lifecycle, SOAR automation, forensics

### Assurance & Process
- [13 — Penetration Testing & ASM](./13-penetration-testing.md) — Red/purple team, CVSS, bug bounty, automated scanning
- [15 — Secure SDLC](./15-secure-sdlc.md) — Security requirements, threat review gates, security ACs, Semgrep rules
- [18 — Third-Party & Vendor Risk](./18-third-party-vendor-risk.md) — Vendor tiers, DPA, SOC 2 review, SBOM, breach notification SLA

---

## Threat → Pattern Mapping

| Threat | Primary pattern | Supporting patterns |
|--------|----------------|-------------------|
| Credential theft / brute-force | [01 OAuth/JWT](./01-oauth2-oidc-jwt.md) + [08 Rate Limiting](./08-ddos-rate-limiting.md) | [07 Logging](./07-security-logging-siem.md), [12 IR](./12-incident-response.md) |
| API abuse / BOLA | [02 API Security](./02-api-security.md) | [08 Rate Limiting](./08-ddos-rate-limiting.md), [07 Logging](./07-security-logging-siem.md) |
| Data breach (at rest) | [03 Encryption](./03-encryption-patterns.md) | [06 Privacy](./06-data-privacy-pii.md), [14 DLP](./14-data-loss-prevention.md) |
| Man-in-the-middle | [04 Network Security](./04-network-security.md) | [11 PKI](./11-pki-certificate-management.md), [03 Encryption](./03-encryption-patterns.md) |
| Account takeover | [05 SSO/Federation](./05-identity-federation-sso.md) | [01 OAuth/JWT](./01-oauth2-oidc-jwt.md), [12 IR](./12-incident-response.md) |
| PII exfiltration | [06 Data Privacy](./06-data-privacy-pii.md) | [14 DLP](./14-data-loss-prevention.md), [07 Logging](./07-security-logging-siem.md) |
| DDoS / service disruption | [08 DDoS & Rate Limiting](./08-ddos-rate-limiting.md) | [12 IR](./12-incident-response.md) |
| XSS / injection | [09 Web App Security](./09-web-app-security.md) | [15 SSDLC](./15-secure-sdlc.md) |
| Secrets exposure | [10 Secrets Management](./10-secrets-management.md) | [15 SSDLC](./15-secure-sdlc.md) |
| Compromised service identity | [04 Network Security](./04-network-security.md) + [11 PKI](./11-pki-certificate-management.md) | [10 Secrets](./10-secrets-management.md) |
| Insider threat / data loss | [14 DLP](./14-data-loss-prevention.md) | [07 Logging](./07-security-logging-siem.md), [06 Privacy](./06-data-privacy-pii.md) |
| Vulnerability in production | [13 Pentest](./13-penetration-testing.md) | [15 SSDLC](./15-secure-sdlc.md), [12 IR](./12-incident-response.md) |
| Prompt injection / LLM abuse | [16 AI/LLM Security](./16-ai-llm-security.md) | [02 API Security](./02-api-security.md), [08 Rate Limiting](./08-ddos-rate-limiting.md) |
| Mobile credential theft | [17 Mobile Security](./17-mobile-security.md) | [01 OAuth/JWT](./01-oauth2-oidc-jwt.md), [12 IR](./12-incident-response.md) |
| Vendor / supply chain breach | [18 Vendor Risk](./18-third-party-vendor-risk.md) | [12 IR](./12-incident-response.md), [06 Privacy](./06-data-privacy-pii.md) |

---

## Regulatory Mapping

| Regulation | Key requirements | Relevant patterns |
|------------|-----------------|------------------|
| **GDPR** | Encryption (Art. 32), Privacy by Design (Art. 25), 72h breach notification (Art. 33), Right to Erasure (Art. 17) | [03](./03-encryption-patterns.md), [06](./06-data-privacy-pii.md), [12](./12-incident-response.md), [15](./15-secure-sdlc.md) |
| **PCI-DSS v4** | Req 3 (encrypt PAN), Req 7 (access control), Req 10 (logging), Req 11 (pentest), Req 12 (policy) | [03](./03-encryption-patterns.md), [02](./02-api-security.md), [07](./07-security-logging-siem.md), [13](./13-penetration-testing.md) |
| **ISO 27001** | A.8 (asset mgmt), A.9 (access control), A.10 (cryptography), A.12 (operations), A.16 (incident mgmt) | [06](./06-data-privacy-pii.md), [01](./01-oauth2-oidc-jwt.md), [03](./03-encryption-patterns.md), [12](./12-incident-response.md) |
| **SOC 2 Type II** | CC6 (access), CC7 (monitoring), CC8 (change mgmt), CC9 (risk) | [01](./01-oauth2-oidc-jwt.md), [07](./07-security-logging-siem.md), [15](./15-secure-sdlc.md), [13](./13-penetration-testing.md) |
| **NIST CSF** | Identify, Protect, Detect, Respond, Recover | [13](./13-penetration-testing.md), [04](./04-network-security.md), [07](./07-security-logging-siem.md), [12](./12-incident-response.md) |

---

## Key Security Principles Applied

| Principle | How it appears in these patterns |
|-----------|----------------------------------|
| **Defence in depth** | Multiple security layers: WAF → API gateway → application → data — [04](./04-network-security.md), [08](./08-ddos-rate-limiting.md), [09](./09-web-app-security.md) |
| **Least privilege** | RBAC, scope-restricted JWTs, minimal SCIM attributes — [01](./01-oauth2-oidc-jwt.md), [05](./05-identity-federation-sso.md) |
| **Fail secure / deny by default** | NetworkPolicy default-deny, 404 not 403 for IDOR, explicit CORS allowlist — [04](./04-network-security.md), [02](./02-api-security.md) |
| **Zero trust** | mTLS between services, SPIFFE identities, no implicit trust on network — [04](./04-network-security.md) |
| **Cryptographic agility** | Abstracted encryption APIs, key rotation policies — [03](./03-encryption-patterns.md), [11](./11-pki-certificate-management.md) |
| **Privacy by design** | Data minimisation, purpose limitation, erasure workflows — [06](./06-data-privacy-pii.md), [15](./15-secure-sdlc.md) |
| **Security left shift** | Security requirements, threat model, and gate reviews at design time — [15](./15-secure-sdlc.md) |

---

## Related Sections

- [DevSecOps](../devsecops/) — CI/CD pipeline security controls: SAST, SCA, DAST, container scanning, IaC security, secret scanning
- [Cloud Native — AWS](../cloud-native/aws/) — AWS-specific security services: IAM, GuardDuty, Security Hub, KMS, WAF
- [Cloud Native — Azure](../cloud-native/azure/) — Azure-specific security: Defender for Cloud, Entra ID, Key Vault, Policy, Sentinel
- [System Design](../system-design/) — Authentication patterns, rate limiting at scale, data residency
- [Distributed Design Patterns](../distributed-design-pattern/) — Service mesh, mTLS, saga pattern security considerations
