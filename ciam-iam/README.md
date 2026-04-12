# CIAM & IAM

A comprehensive catalogue of 15 production-ready patterns for Customer Identity and Access Management (CIAM) and Identity and Access Management (IAM).

## Patterns

| # | Pattern | Category | Key Technologies |
|---|---------|----------|-----------------|
| 01 | [OAuth 2.0 & OpenID Connect](./01-oauth2-oidc.md) | Protocols | Auth Code + PKCE, token endpoint, scopes |
| 02 | [JWT & Token Management](./02-jwt-token-management.md) | Token Security | JWK sets, rotation, introspection, DPoP |
| 03 | [SAML & Enterprise SSO](./03-saml-enterprise-sso.md) | Enterprise Federation | SAML 2.0, IdP/SP-initiated, metadata exchange |
| 04 | [SCIM — User Provisioning](./04-scim-user-provisioning.md) | Lifecycle Management | SCIM 2.0, HR sync, just-in-time provisioning |
| 05 | [MFA & Step-Up Authentication](./05-mfa-step-up-auth.md) | Authentication | TOTP, SMS OTP, risk-based MFA, step-up flows |
| 06 | [Passkeys & WebAuthn](./06-passkeys-webauthn.md) | Passwordless | FIDO2, WebAuthn, passkey registration/assertion |
| 07 | [Role-Based Access Control (RBAC)](./07-rbac.md) | Authorisation | Roles, permissions, hierarchy, enforcement |
| 08 | [Attribute-Based Access Control (ABAC)](./08-abac-policy-engines.md) | Authorisation | OPA/Rego, Cedar, policy-as-code |
| 09 | [Customer Identity (CIAM)](./09-ciam-customer-identity.md) | Customer Identity | Social login, progressive profiling, consent |
| 10 | [Identity Federation](./10-identity-federation.md) | B2B / Cross-Org | External IdP, federation, B2B provisioning |
| 11 | [Fine-Grained Authorisation (ReBAC)](./11-fine-grained-authorization.md) | Authorisation | Zanzibar, OpenFGA, SpiceDB, relationship tuples |
| 12 | [Session Management](./12-session-management.md) | Auth Lifecycle | Session tokens, revocation, refresh, concurrent |
| 13 | [Identity Providers Comparison](./13-identity-providers.md) | Platform | Keycloak, Auth0, Okta, AWS Cognito, Azure AD B2C |
| 14 | [API Security with OAuth](./14-api-security-oauth.md) | API Protection | Bearer tokens, token exchange, mTLS, DPoP |
| 15 | [Identity Threat Detection](./15-identity-threat-detection.md) | Security | ATO, credential stuffing, anomaly detection |

---

## Decision Guides

### Which authorisation model?

| Scenario | Model | File |
|---------|-------|------|
| Simple app with a few roles (admin/user/viewer) | RBAC | [07](./07-rbac.md) |
| Complex policies based on resource attributes and context | ABAC / OPA | [08](./08-abac-policy-engines.md) |
| Social network style (user A can see user B's data) | ReBAC / Zanzibar | [11](./11-fine-grained-authorization.md) |
| Enterprise — honour org-level permissions | Hierarchical RBAC | [07](./07-rbac.md) |
| Multi-tenant SaaS with per-tenant permission sets | ReBAC | [11](./11-fine-grained-authorization.md) |

### Which IdP / auth platform?

| Scenario | Recommendation | File |
|---------|----------------|------|
| Open-source, self-hosted, full control | Keycloak | [13](./13-identity-providers.md) |
| Consumer-facing app, social login, low friction | Auth0 or Cognito | [13](./13-identity-providers.md) |
| Enterprise workforce SSO + AD sync | Okta or Azure AD | [13](./13-identity-providers.md) |
| AWS-native, serverless | AWS Cognito | [13](./13-identity-providers.md) |
| Passwordless-first | Any + WebAuthn passkeys | [06](./06-passkeys-webauthn.md) |

### CIAM vs IAM

| Dimension | IAM (workforce) | CIAM (customer) |
|-----------|----------------|----------------|
| Users | Employees, partners | End customers, consumers |
| Registration | Admin-provisioned | Self-service + social login |
| Scale | Thousands | Millions |
| UX priority | Security | Frictionless experience |
| Compliance | SOX, HIPAA (internal) | GDPR, CCPA (consumer) |
| MFA style | Enforced always | Risk-based, step-up |

---

## Categories

### Protocols & Standards
- **OAuth 2.0 & OIDC** — the foundation of modern delegated auth
- **SAML** — enterprise SSO for legacy and B2B integrations

### Token Security
- **JWT & Token Management** — signing, validation, rotation, revocation

### Authentication
- **MFA & Step-Up** — layered authentication for high-risk operations
- **Passkeys & WebAuthn** — phishing-resistant passwordless authentication

### Authorisation
- **RBAC** — role-based, simple and effective for most apps
- **ABAC** — attribute and policy-based for fine-grained control
- **Fine-Grained AuthZ (ReBAC)** — relationship-based, Google Zanzibar model

### Identity Lifecycle
- **SCIM** — automated user provisioning and de-provisioning
- **Session Management** — session persistence, refresh, and revocation

### Customer Identity
- **CIAM** — customer registration, social login, consent, progressive profiling
- **Identity Federation** — B2B federation, external IdP trust

### Platform & Security
- **Identity Providers** — platform comparison and selection
- **API Security with OAuth** — protecting APIs with tokens
- **Identity Threat Detection** — detecting and responding to identity attacks
