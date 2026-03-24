# Micro-Frontends

## Category

Frontend Architecture — Composition & Autonomy

## Context

Micro-frontends apply microservice thinking to the browser: a large frontend application is decomposed into independently developed, tested, and deployed UI fragments owned by separate teams. Module Federation (Webpack 5 / Rspack / Vite) is the dominant runtime technique, allowing a host shell to dynamically load remote modules without rebuilding the shell.

### Micro-Frontend Composition Strategies

| Strategy | Runtime | Coupling | Isolation | Best for |
|----------|---------|---------|-----------|---------|
| **Module Federation** | Browser | Low | Shared scope | SPA + multi-team |
| **iFrame** | Browser | None | Full | Low-trust third-party |
| **Web Components** | Browser | Low | Shadow DOM | Framework-agnostic |
| **Build-time composition** | CI/CD | High | None | Small, aligned team |
| **Server-side composition** | Server | Medium | Process | SSR + edge rendering |
| **Edge-side includes (ESI)** | CDN / Nginx | Medium | None | CDN-cached fragments |

### Module Federation Key Concepts

| Concept | Description |
|---------|-------------|
| **Host (shell)** | The container application — loads and renders remote modules |
| **Remote** | A separately built and deployed application exposing modules |
| **Shared** | Libraries shared between host and remotes to avoid duplication (React, ReactDOM) |
| **Exposed** | A specific module that a remote makes available to hosts |
| **Dynamic remote** | Remote URL resolved at runtime — enables independent deployment |

## Pros

- Teams deploy independently — no release coordination for routine features
- Technology heterogeneity possible at the cost of shared-library discipline
- Smaller bundles per team — faster initial load for specific routes
- Fault isolation: a failing remote does not crash the entire shell
- Clear codebase ownership aligns with product team topology

## Cons

- Shared library version mismatches cause runtime errors (React singleton violation)
- Developer experience overhead: need local orchestration to run all remotes
- Bundle duplication risk if `shared` configuration is misconfigured
- Cross-remote communication patterns (event bus, custom events) become bespoke conventions
- Performance testing must cover the composed app — individual bundle budgets mislead

## Design Diagram

```mermaid
flowchart LR
    Browser([Browser]) --> Shell[Shell Application\n:3000 — Host]

    Shell -->|dynamic import| AccountRemote[Account Remote\n:3001]
    Shell -->|dynamic import| PaymentRemote[Payment Remote\n:3002]
    Shell -->|dynamic import| NotifyRemote[Notification Remote\n:3003]

    subgraph Shared Scope
        Shell --- React[react@18 singleton]
        AccountRemote --- React
        PaymentRemote --- React
        NotifyRemote --- React
    end

    AccountRemote --> AccountDB[(Account service)]
    PaymentRemote --> PaymentDB[(Payment service)]

    subgraph CI/CD
        AccountPipeline[Account CI] -->|deploy| AccountCDN[Account bundle\nCDN]
        PaymentPipeline[Payment CI] -->|deploy| PaymentCDN[Payment bundle\nCDN]
    end
```

## Code Sample

### TypeScript — Vite Module Federation config (host shell)

```typescript
// vite.config.ts — shell (host)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import federation from '@originjs/vite-plugin-federation';

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'shell',
      remotes: {
        // URLs resolved at runtime — supports dynamic remote switching
        accountRemote: process.env.ACCOUNT_REMOTE_URL ?? 'http://localhost:3001/assets/remoteEntry.js',
        paymentRemote: process.env.PAYMENT_REMOTE_URL ?? 'http://localhost:3002/assets/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
        'react-router-dom': { singleton: true, requiredVersion: '^6.0.0' },
      },
    }),
  ],
  build: {
    target: 'esnext',
    minify: false, // remotes must not minify for correct module names
    cssCodeSplit: false,
  },
});
```

### TypeScript — Vite Module Federation config (payment remote)

```typescript
// vite.config.ts — payment remote
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import federation from '@originjs/vite-plugin-federation';

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: 'paymentRemote',
      filename: 'remoteEntry.js',
      exposes: {
        './PaymentDashboard': './src/PaymentDashboard',
        './CreatePaymentForm': './src/CreatePaymentForm',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
        'react-router-dom': { singleton: true, requiredVersion: '^6.0.0' },
      },
    }),
  ],
  server: { port: 3002, cors: true },
  preview: { port: 3002, cors: true },
  build: { target: 'esnext', minify: false, cssCodeSplit: false },
});
```

### TypeScript — Shell lazy-loading a remote with error boundary

```tsx
// src/RemoteLoader.tsx — shell-side lazy remote wrapper
import React, { Suspense, lazy, ComponentType } from 'react';

interface RemoteLoaderProps {
  module: () => Promise<{ default: ComponentType }>;
  fallback?: React.ReactNode;
}

class RemoteErrorBoundary extends React.Component<
  React.PropsWithChildren<{ name: string }>,
  { hasError: boolean; error?: Error }
> {
  state = { hasError: false, error: undefined };

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error) {
    console.error(`[micro-frontend] Remote "${this.props.name}" failed to load:`, error);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert" aria-live="assertive">
          <p>This section is temporarily unavailable. Please refresh the page.</p>
        </div>
      );
    }
    return this.props.children;
  }
}

export function RemoteLoader({ module, fallback }: RemoteLoaderProps) {
  const Component = lazy(module);
  return (
    <RemoteErrorBoundary name="remote">
      <Suspense fallback={fallback ?? <div aria-busy="true">Loading…</div>}>
        <Component />
      </Suspense>
    </RemoteErrorBoundary>
  );
}

// Usage in shell routing:
// const PaymentDashboard = () => (
//   <RemoteLoader module={() => import('paymentRemote/PaymentDashboard')} />
// );
```

### TypeScript — Cross-remote event bus (decoupled communication)

```typescript
// src/eventBus.ts — vanilla custom events for cross-remote communication
export type AppEventMap = {
  'payment:created': { paymentId: string; amount: number; currency: string };
  'account:selected': { accountId: string; iban: string };
  'auth:token-refreshed': { accessToken: string };
  'navigation:requested': { path: string };
};

type AppEventName = keyof AppEventMap;

export const eventBus = {
  emit<K extends AppEventName>(name: K, detail: AppEventMap[K]): void {
    window.dispatchEvent(new CustomEvent(`mfe:${name}`, { detail, bubbles: false }));
  },

  on<K extends AppEventName>(
    name: K,
    handler: (detail: AppEventMap[K]) => void,
  ): () => void {
    const listener = (e: Event) => {
      handler((e as CustomEvent<AppEventMap[K]>).detail);
    };
    window.addEventListener(`mfe:${name}`, listener);
    return () => window.removeEventListener(`mfe:${name}`, listener);
  },
};

// React hook wrapper for remotes:
import { useEffect } from 'react';

export function useAppEvent<K extends AppEventName>(
  name: K,
  handler: (detail: AppEventMap[K]) => void,
): void {
  useEffect(() => eventBus.on(name, handler), [name, handler]);
}
```
