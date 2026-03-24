# Frontend Authentication (OIDC / OAuth 2.0 PKCE)

## Category

Frontend Architecture — Identity & Security

## Context

SPAs are public clients — they cannot hold a client secret. The correct OAuth 2.0 flow for SPAs is **Authorization Code + PKCE** (Proof Key for Code Exchange). Silent token renewal uses a refresh token stored in an HttpOnly cookie (not localStorage). Third-party OIDC SDKs (`oidc-client-ts`) handle the full lifecycle: login redirect, callback handling, token refresh, and logout.

### Token Storage Comparison

| Storage | XSS risk | CSRF risk | Accessible by JS | Recommended |
|---------|----------|----------|-----------------|-------------|
| **localStorage** | ❌ High | ✅ None | ✅ | ❌ No |
| **sessionStorage** | ❌ High | ✅ None | ✅ | ❌ No |
| **In-memory (module variable)** | ✅ None | ✅ None | ✅ | ✅ Access token |
| **HttpOnly cookie** | ✅ None | ❌ Mitigated with `SameSite=Strict` | ❌ | ✅ Refresh token |
| **Web Worker** | ✅ None (isolated context) | ✅ None | ❌ | ✅ Advanced |

### PKCE Flow

```
1. Generate code_verifier (random 43-128 chars)
2. code_challenge = BASE64URL(SHA256(code_verifier))
3. Redirect → /authorize?response_type=code&code_challenge=...&code_challenge_method=S256
4. Auth server redirects → /callback?code=...
5. Exchange: POST /token { code, code_verifier, client_id, redirect_uri }
6. Receive access_token (short-lived) + refresh_token (long-lived, HttpOnly cookie)
7. Silent renew: POST /token { grant_type=refresh_token } via invisible iframe or back channel
```

## Pros

- PKCE prevents auth code interception attacks without requiring a client secret
- `oidc-client-ts` handles token expiry, silent renewal, and session timeout automatically
- Access token stored in memory is not accessible to XSS scripts in other modules
- Centralised `AuthProvider` makes auth state available everywhere without prop drilling
- Route guards with `PrivateRoute` prevent unauthenticated rendering of sensitive routes

## Cons

- In-memory access tokens are lost on page refresh — requires silent renew on startup
- Silent renew via hidden iframe requires third-party cookies — blocked on Safari ITP
- Back-channel logout requires the SPA to poll or use SSE — iframes blocked more broadly
- PKCE adds a round trip vs implicit flow (now deprecated) — negligible in practice
- Integrating with enterprise IdPs (ADFS, legacy Keycloak) requires careful scope/claim mapping

## Design Diagram

```mermaid
flowchart LR
    SPA([SPA]) -->|1. login redirect + PKCE| IdP[Identity Provider\nKeycloak / Auth0 / Azure AD]
    IdP -->|2. code| Callback[/callback route]
    Callback -->|3. exchange code + verifier| IdP
    IdP -->|4. access_token + refresh_token cookie| SPA

    SPA -->|5. Authorization: Bearer ...| API[API]

    SPA -->|token expiring| Renew[Silent Renew\nback-channel /token]
    Renew --> IdP
    IdP -->|new access_token| SPA

    subgraph Protected Routes
        SPA --> Guard[PrivateRoute\ncheck isAuthenticated]
        Guard -->|not authenticated| Login[Redirect to login]
        Guard -->|authenticated| Page[Protected Page]
    end
```

## Code Sample

### TypeScript — OIDC auth service using oidc-client-ts

```typescript
// src/auth/authService.ts
import { UserManager, WebStorageStateStore, type User } from 'oidc-client-ts';

const userManager = new UserManager({
  authority: import.meta.env.VITE_OIDC_AUTHORITY,      // e.g. https://auth.example.com/realms/acme
  client_id: import.meta.env.VITE_OIDC_CLIENT_ID,      // public client — no client_secret
  redirect_uri: `${window.location.origin}/auth/callback`,
  post_logout_redirect_uri: window.location.origin,
  response_type: 'code',                                // PKCE — always code flow
  scope: 'openid profile email payments:read payments:write',
  automaticSilentRenew: true,
  silent_redirect_uri: `${window.location.origin}/auth/silent-renew.html`,
  // Store user object in sessionStorage (not access_token in localStorage)
  userStore: new WebStorageStateStore({ store: window.sessionStorage }),
  // Store access token expiry metadata only — actual token in memory
  filterProtocolClaims: true,
  loadUserInfo: true,
});

// ── Events ─────────────────────────────────────────────────────────────────────
userManager.events.addSilentRenewError((err) => {
  console.error('[auth] Silent renew failed — redirecting to login', err);
  void authService.login();
});

userManager.events.addUserSignedOut(() => {
  console.warn('[auth] User signed out from IdP — clearing local session');
  void userManager.removeUser();
});

// ── Auth service ───────────────────────────────────────────────────────────────
export const authService = {
  login: (returnTo?: string) =>
    userManager.signinRedirect({ state: { returnTo: returnTo ?? window.location.pathname } }),

  handleCallback: () => userManager.signinRedirectCallback(),

  logout: () => userManager.signoutRedirect(),

  getUser: () => userManager.getUser(),

  getAccessToken: async (): Promise<string | null> => {
    const user = await userManager.getUser();
    if (!user || user.expired) return null;
    return user.access_token;
  },

  isAuthenticated: async (): Promise<boolean> => {
    const user = await userManager.getUser();
    return Boolean(user && !user.expired);
  },

  hasScope: async (scope: string): Promise<boolean> => {
    const user = await userManager.getUser();
    return Boolean(user?.scopes?.includes(scope));
  },
};
```

### TypeScript — React AuthContext and PrivateRoute

```tsx
// src/auth/AuthContext.tsx
import {
  createContext,
  useContext,
  useEffect,
  useState,
  type ReactNode,
} from 'react';
import { type User } from 'oidc-client-ts';
import { authService } from './authService';

interface AuthContextValue {
  user: User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  login: () => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    void (async () => {
      try {
        const currentUser = await authService.getUser();
        setUser(currentUser);
      } finally {
        setIsLoading(false);
      }
    })();
  }, []);

  return (
    <AuthContext.Provider
      value={{
        user,
        isLoading,
        isAuthenticated: Boolean(user && !user.expired),
        login: () => void authService.login(),
        logout: () => void authService.logout(),
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used inside AuthProvider');
  return ctx;
}

// ── PrivateRoute ───────────────────────────────────────────────────────────────
// src/auth/PrivateRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';

interface PrivateRouteProps {
  children: ReactNode;
  requiredScope?: string;
}

export function PrivateRoute({ children, requiredScope }: PrivateRouteProps) {
  const { isAuthenticated, isLoading, user } = useAuth();
  const location = useLocation();

  if (isLoading) return <div aria-busy="true" aria-label="Checking authentication…" />;

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (requiredScope && !user?.scopes?.includes(requiredScope)) {
    return <Navigate to="/unauthorised" replace />;
  }

  return <>{children}</>;
}
```

### TypeScript — Callback and silent renew route handlers

```tsx
// src/auth/CallbackPage.tsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { authService } from './authService';

export function CallbackPage() {
  const navigate = useNavigate();

  useEffect(() => {
    void (async () => {
      try {
        const user = await authService.handleCallback();
        const returnTo = (user.state as { returnTo?: string } | undefined)?.returnTo ?? '/';
        navigate(returnTo, { replace: true });
      } catch (err) {
        console.error('[auth] Callback error:', err);
        navigate('/login', { replace: true });
      }
    })();
  }, [navigate]);

  return <div aria-busy="true" aria-label="Completing sign-in…" />;
}
```

### TypeScript — Axios interceptor that attaches Bearer token

```typescript
// src/api/httpClient.ts
import axios from 'axios';
import { authService } from '../auth/authService';

export const httpClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

httpClient.interceptors.request.use(async (config) => {
  const token = await authService.getAccessToken();
  if (token) config.headers['Authorization'] = `Bearer ${token}`;
  return config;
});

httpClient.interceptors.response.use(
  (response) => response,
  async (error: unknown) => {
    if (axios.isAxiosError(error) && error.response?.status === 401) {
      // Token rejected by API — force re-login
      await authService.login(window.location.pathname);
    }
    return Promise.reject(error);
  },
);
```
