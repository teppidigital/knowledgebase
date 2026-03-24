# Frontend Architecture Patterns

A comprehensive reference of 15 frontend architecture patterns covering component composition, state management, performance, accessibility, security, and the full delivery pipeline for modern React applications.

---

## Pattern Index

| # | Pattern | Key Concepts |
|---|---------|-------------|
| 01 | [Micro-Frontends](01-micro-frontends.md) | Module Federation (Vite), host/remote, shared scope, event bus, error boundary |
| 02 | [State Management](02-state-management.md) | TanStack Query v5, Zustand, React Hook Form + Zod, server vs client state |
| 03 | [Performance Optimisation](03-performance-optimisation.md) | Core Web Vitals, code splitting, virtual lists, Web Workers, bundle budgets |
| 04 | [Design System](04-design-system.md) | Radix UI, design tokens (Style Dictionary), Storybook, accessibility-first components |
| 05 | [Server-Side Rendering](05-server-side-rendering.md) | Next.js App Router, RSC, SSG/ISR/SSR, streaming Suspense, revalidateTag |
| 06 | [Progressive Web App](06-progressive-web-app.md) | Workbox, caching strategies, Vite PWA plugin, Push API, VAPID, SW lifecycle |
| 07 | [Frontend Testing](07-frontend-testing.md) | Vitest, Testing Library (role queries), MSW integration tests, Playwright E2E |
| 08 | [Frontend Authentication](08-authentication-frontend.md) | OIDC PKCE, oidc-client-ts, in-memory token, AuthProvider, PrivateRoute, Axios interceptor |
| 09 | [Feature Flags](09-feature-flags-frontend.md) | OpenFeature SDK, LaunchDarkly provider, in-memory test provider, A/B string variants |
| 10 | [Error Handling](10-error-handling-frontend.md) | Error Boundary, Sentry (PII scrubbing), QueryErrorBoundary, global handlers |
| 11 | [i18n & Localisation](11-i18n-localisation.md) | i18next, ICU plurals, HTTP backend, Intl.* APIs, RTL support, relative time |
| 12 | [Accessibility (a11y)](12-accessibility-a11y.md) | WCAG 2.1 AA, ARIA roles, focus trap/restore, axe-core, skip nav, form patterns |
| 13 | [Bundling & Build Tools](13-bundling-build-tools.md) | Vite + SWC, Rollup manualChunks, source maps, Sentry plugin, bundle size CI |
| 14 | [CDN & Edge Delivery](14-cdn-edge-delivery.md) | Cloudflare Workers, A/B at edge, Next.js Middleware, CloudFront Terraform, SRI |
| 15 | [Frontend Security](15-frontend-security.md) | CSP nonce, Trusted Types, SameSite cookies, open redirect prevention, SBOM + Grype |

---

## Decision Guide

### Which state management approach should I use?

```
Is the state remote data from an API?
  └── Yes → TanStack Query (useQuery + useMutation + invalidateQueries)

Is it shared UI state (auth user, theme, feature flags)?
  └── Zustand store (flat, no provider)

Is it form state (field values, validation)?
  └── React Hook Form + Zod resolver

Is it URL-driven (filters, pagination, sort)?
  └── useSearchParams (React Router v6)

Is it local component state only?
  └── useState / useReducer
```

### Which rendering strategy should I use?

```
Page content is static at build time (marketing, docs)?
  └── Next.js SSG (generateStaticParams)

Content changes occasionally (product pages, blogs)?
  └── Next.js ISR with revalidate + revalidateTag

Content is personalised or auth-gated?
  └── Next.js SSR (dynamic) or RSC with auth check

App is complex SPA with frequent updates (dashboard)?
  └── Client-side rendering with TanStack Query
```

### When should I use Micro-Frontends?

```
Multiple teams, independent deploy cadence?
  └── Yes → Module Federation (Vite / Rspack)

Single team, single deploy?
  └── No → Monolith SPA with code splitting is simpler

Third-party/untrusted fragment?
  └── iFrame (full isolation)
```

### How do I secure the frontend?

```
Always:
  ├── CSP with per-request nonces (no unsafe-inline, no unsafe-eval)
  ├── SameSite=Strict cookies for session
  ├── Access token in memory (not localStorage)
  ├── npm audit + Dependabot in CI
  └── Runtime: OWASP ZAP scan

For external scripts:
  └── Subresource Integrity (integrity attribute)

For DOM manipulation:
  └── Trusted Types policy + DOMPurify
```

---

## Tool Ecosystem

### Build

| Tool | Purpose |
|------|---------|
| **Vite + @vitejs/plugin-react-swc** | Dev server + SWC transpiler |
| **Rollup** | Production bundler (via Vite) |
| **TypeScript 5** | Type safety across all layers |
| **rollup-plugin-visualizer** | Interactive bundle treemap |
| **sentry-vite-plugin** | Source map upload |

### State & Data

| Tool | Purpose |
|------|---------|
| **TanStack Query v5** | Server state, caching, mutations |
| **Zustand v4** | Global client state |
| **React Hook Form** | Form state and validation |
| **Zod** | Runtime schema validation |

### Testing

| Tool | Layer |
|------|-------|
| Vitest | Unit + integration |
| @testing-library/react | Component behaviour |
| MSW v2 | API mocking (browser + Node) |
| Playwright | E2E browser automation |
| axe-core / jest-axe | Accessibility testing |
| Chromatic | Visual regression |

### UI & Accessibility

| Tool | Purpose |
|------|---------|
| **Radix UI** | Accessible headless primitives |
| **Storybook 8** | Component docs + visual testing |
| **Style Dictionary** | Design token pipeline |
| **Tailwind CSS** | Utility-first styling |

### Authentication

| Tool | Purpose |
|------|---------|
| **oidc-client-ts** | OIDC PKCE flow for SPAs |
| **jose** | JWT verification in Edge Functions |

### Internationalisation

| Tool | Purpose |
|------|---------|
| **i18next** | Core i18n runtime |
| **react-i18next** | React hooks + Suspense integration |
| **i18next-icu** | ICU message format for plurals |
| **Intl.**** | Browser-native date, number, currency |

### Delivery & Security

| Tool | Purpose |
|------|---------|
| **Cloudflare Workers** | Edge compute (A/B, geo, auth) |
| **Vite PWA plugin** | Workbox integration |
| **DOMPurify** | HTML sanitisation |
| **Dependabot / Snyk** | Dependency vulnerability scanning |

---

## Key Conventions

- **TypeScript**: strict mode, `noUncheckedIndexedAccess`, path aliases via `@/*`
- **State**: server state → TanStack Query; client state → Zustand; form → RHF
- **Authentication**: PKCE + `oidc-client-ts`; access token in memory only; refresh token in HttpOnly cookie
- **Security**: CSP nonce per request; no `innerHTML` without Trusted Types + DOMPurify; `SameSite=Strict`
- **Accessibility**: WCAG 2.1 AA; `<button>` over `<div>`; `aria-live` for dynamic updates; focus management on modal close
- **Performance**: `React.lazy()` per route; `@tanstack/virtual` for lists >100 items; Web Worker for CPU tasks
- **Testing queries**: `getByRole` first; `getByLabelText` for forms; `getByTestId` only as last resort
- **Assets**: hashed bundles with `immutable` cache; `index.html` never cached; source maps to Sentry only
