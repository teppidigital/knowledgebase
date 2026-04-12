# Identity Federation

## Category

CIAM / IAM — B2B Federation, External IdP, Cross-Org, Trust Framework, SAML, OIDC Federation

## Context

**Identity Federation** allows users from one organisation (the partner/customer) to authenticate with their own corporate Identity Provider and gain access to your application — without creating new credentials in your system. You trust the external IdP to vouch for the user.

This is the foundation of B2B SaaS: a corporate customer's employees sign in with their Okta or Azure AD account to access your platform.

### Federation Models

| Model | Description | Use Case |
|-------|-------------|---------|
| **SP-Initiated SSO** | User starts at your app, redirected to their IdP | B2B SaaS login |
| **IdP-Initiated SSO** | User starts at corporate IdP portal | Enterprise intranet portals |
| **OIDC Federation** | External OIDC provider as trusted issuer | Modern B2B, developer portals |
| **Token Exchange (RFC 8693)** | Exchange external token for internal token | Cross-service, partner APIs |
| **Cross-Tenant** | Same platform, different tenant IdPs | Multi-tenant SaaS |

### Trust Configuration Per Tenant

Each B2B customer requires a federation configuration:

```
Tenant: Acme Corp
  Protocol: SAML 2.0
  IdP SSO URL: https://acme.okta.com/app/saml/sso
  IdP Certificate: <PEM>
  Attribute Mapping:
    email ← NameID
    groups ← http://schemas.microsoft.com/ws/2008/06/identity/claims/groups
    department ← department
```

---

## Pros

- Customers manage their own user lifecycle — offboarding an employee immediately revokes access.
- No password management for federated users — credential security is the IdP's responsibility.
- Single sign-on experience: employees access your app from their corporate dashboard without re-login.
- SCIM + Federation combination provides full automated lifecycle (join, move, leave).
- Reduces onboarding friction for enterprise deals — "it works with Okta/Azure AD" is a deal requirement.

---

## Cons

- Each tenant's IdP configuration must be managed and tested separately.
- Certificate rotation at the customer's IdP can break federation without warning.
- Attribute mapping inconsistencies across IdPs require flexible parsing logic.
- IdP outages become your outage for federated tenants — no fallback authentication.
- Token Exchange (RFC 8693) is complex to implement and not all IdPs support it.

---

## Design Diagram

```mermaid
flowchart LR
    CORP_USER([Employee\n@ Acme Corp])
    CORP_IDP["Acme Corp IdP\n(Okta / Azure AD)"]
    YOUR_APP["Your App\nSP (Multi-tenant)"]
    TENANT_CONFIG[("Federation Config\nper tenant")]
    USER_DB[("User Store\nexternal_id + tenant")]

    CORP_USER -->|access your app| YOUR_APP
    YOUR_APP -->|lookup tenant federation config| TENANT_CONFIG
    YOUR_APP -->|redirect to tenant IdP| CORP_IDP
    CORP_IDP -->|authenticates employee| CORP_USER
    CORP_IDP -->|SAML assertion / OIDC token| YOUR_APP
    YOUR_APP -->|JIT provision or lookup| USER_DB
```

---

## Code Sample

### TypeScript — multi-tenant federation router

```typescript
import { Router } from 'express';
import { SAML } from '@node-saml/node-saml';

const router = Router();

interface FederationConfig {
  tenantId: string;
  protocol: 'saml' | 'oidc';
  // SAML
  idpSsoUrl?: string;
  idpCert?: string;
  // OIDC
  discoveryUrl?: string;
  clientId?: string;
  clientSecret?: string;
  // Attribute mapping
  emailAttribute?: string;
  groupsAttribute?: string;
}

async function getFederationConfig(tenantId: string): Promise<FederationConfig | null> {
  return db.federationConfig.findUnique({ where: { tenantId } });
}

// Initiate SSO for a tenant — detects protocol automatically
router.get('/auth/sso/:tenantId', async (req, res) => {
  const config = await getFederationConfig(req.params.tenantId);

  if (!config) {
    return res.status(404).json({ error: 'No federation configured for this tenant' });
  }

  req.session.federationTenantId = config.tenantId;

  if (config.protocol === 'saml') {
    const saml = buildSamlInstance(config);
    const { context: url } = await saml.getAuthorizeUrlAsync('/', req.headers.host!, {});
    return res.redirect(url);
  }

  if (config.protocol === 'oidc') {
    // Redirect to OIDC discovery-based IdP
    const oidcDiscovery = await fetch(`${config.discoveryUrl}/.well-known/openid-configuration`).then(r => r.json());
    const state = crypto.randomUUID();
    req.session.oidcState = state;

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: config.clientId!,
      redirect_uri: `${process.env.BASE_URL}/auth/oidc/${config.tenantId}/callback`,
      scope: 'openid profile email',
      state,
    });
    return res.redirect(`${oidcDiscovery.authorization_endpoint}?${params}`);
  }
});

function buildSamlInstance(config: FederationConfig): SAML {
  return new SAML({
    callbackUrl: `${process.env.BASE_URL}/auth/saml/${config.tenantId}/callback`,
    issuer: `${process.env.BASE_URL}/sp/${config.tenantId}`,
    entryPoint: config.idpSsoUrl!,
    idpCert: config.idpCert!,
    wantAssertionsSigned: true,
    signatureAlgorithm: 'sha256',
  });
}
```

### TypeScript — JIT provisioning on federated login

```typescript
interface FederatedIdentity {
  externalSub: string;  // IdP's unique user identifier
  tenantId: string;
  email: string;
  name?: string;
  groups?: string[];
  department?: string;
}

async function jitProvision(identity: FederatedIdentity) {
  // Find existing user by externalSub + tenantId (most reliable) or email fallback
  let user = await db.user.findFirst({
    where: {
      OR: [
        { externalSub: identity.externalSub, tenantId: identity.tenantId },
        { email: identity.email, tenantId: identity.tenantId },
      ],
    },
  });

  if (!user) {
    // First login — provision the user
    user = await db.user.create({
      data: {
        email: identity.email,
        name: identity.name,
        tenantId: identity.tenantId,
        externalSub: identity.externalSub,
        emailVerified: true, // trusted from IdP
        source: 'federation',
      },
    });
  } else {
    // Subsequent login — sync attributes that may have changed in the IdP
    await db.user.update({
      where: { id: user.id },
      data: {
        name: identity.name,
        externalSub: identity.externalSub, // ensure externalSub is linked
        lastFederatedLoginAt: new Date(),
      },
    });
  }

  // Sync IdP groups to application roles
  if (identity.groups) {
    await syncFederatedGroups(user.id, identity.groups, identity.tenantId);
  }

  return user;
}

async function syncFederatedGroups(userId: string, idpGroups: string[], tenantId: string): Promise<void> {
  // Load mapping: Okta group → app role
  const mappings = await db.groupRoleMapping.findMany({ where: { tenantId } });

  const rolesToAssign = mappings
    .filter(m => idpGroups.includes(m.idpGroupName))
    .map(m => m.roleId);

  // Replace existing federated role assignments
  await db.$transaction([
    db.userRole.deleteMany({ where: { userId, tenantId, source: 'federation' } }),
    db.userRole.createMany({
      data: rolesToAssign.map(roleId => ({ userId, roleId, tenantId, source: 'federation' })),
      skipDuplicates: true,
    }),
  ]);
}
```

### TypeScript — OAuth 2.0 Token Exchange (RFC 8693)

```typescript
// Exchange an external partner token for an internal access token
router.post('/auth/token-exchange', async (req, res) => {
  const {
    grant_type,
    subject_token,
    subject_token_type,
    audience,
  } = req.body;

  if (grant_type !== 'urn:ietf:params:oauth:grant-type:token-exchange') {
    return res.status(400).json({ error: 'unsupported_grant_type' });
  }

  // Validate the incoming external token
  let externalClaims: Record<string, unknown>;
  try {
    if (subject_token_type === 'urn:ietf:params:oauth:token-type:jwt') {
      externalClaims = await validatePartnerToken(subject_token);
    } else {
      return res.status(400).json({ error: 'unsupported_token_type' });
    }
  } catch {
    return res.status(401).json({ error: 'invalid_subject_token' });
  }

  // Map external identity to internal user
  const user = await db.user.findFirst({
    where: { externalSub: externalClaims.sub as string },
  });

  if (!user) {
    return res.status(401).json({ error: 'unknown_subject' });
  }

  // Issue internal access token scoped to the requested audience
  const internalToken = await issueAccessToken(user.id, audience);

  res.json({
    access_token: internalToken,
    token_type: 'Bearer',
    expires_in: 900,
    issued_token_type: 'urn:ietf:params:oauth:token-type:access_token',
  });
});
```

---

## Related

- [03 — SAML & Enterprise SSO](./03-saml-enterprise-sso.md) — SAML is the most common B2B federation protocol
- [04 — SCIM User Provisioning](./04-scim-user-provisioning.md) — SCIM + federation for full automated lifecycle
- [07 — RBAC](./07-rbac.md) — IdP group → role mapping drives federated authorisation
- [13 — Identity Providers](./13-identity-providers.md) — Okta and Azure AD as enterprise federation counterparties
