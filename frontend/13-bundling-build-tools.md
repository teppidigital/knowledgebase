# Bundling & Build Tools

## Category

Frontend Architecture — Build Infrastructure

## Context

The build tool transforms source TypeScript/JSX into optimised bundles suitable for production delivery. Vite (dev) + Rollup (production build) is the dominant choice for SPAs; Turbopack is emerging for Next.js. The build pipeline affects developer velocity (HMR speed), bundle size (Tree shaking, code splitting), and deployment (asset hashing, source maps).

### Build Tool Comparison

| Tool | Dev speed | Prod bundler | Tree shaking | Code splitting | Use case |
|------|----------|------------|-------------|---------------|----------|
| **Vite** | ⚡ Instant (ESM) | Rollup | ✅ | ✅ | SPA, libraries |
| **Next.js** | 🟡 Fast (SWC) | Webpack / Turbopack | ✅ | ✅ Auto | SSR, SSG |
| **Rspack** | ⚡ Fast (Rust) | Rspack | ✅ | ✅ | Webpack migration |
| **Webpack 5** | 🔴 Slow | Webpack | ✅ | ✅ | Legacy large apps |
| **esbuild** | ⚡ Fastest | esbuild | ✅ | Limited | Scripting, CLIs |
| **Rollup** | N/A (no dev server) | Rollup | ✅ Best | ✅ | Libraries |

### Vite Config Optimisation Levers

| Config | Purpose |
|--------|---------|
| `build.rollupOptions.output.manualChunks` | Control chunk grouping |
| `build.chunkSizeWarningLimit` | Alert on oversized chunks |
| `build.sourcemap` | Sentry source maps for production debugging |
| `build.minify: 'esbuild'` | Faster minification (default) |
| `optimizeDeps.include` | Pre-bundle slow CJS dependencies |
| `resolve.alias` | Absolute import paths (no `../../`) |
| `preview.port` | Local preview of production build |

## Pros

- Vite's native ESM dev server compiles only modules needed for the current page — instant cold start
- Rollup tree-shaking removes unused exports at the symbol level — no dead code in production
- `manualChunks` prevents vendor churn — library chunks are cached across deploys
- Source maps uploaded to Sentry give production-quality stack traces without exposing source
- `@rollup/plugin-visualizer` renders an interactive bundle treemap in the browser

## Cons

- Vite/Rollup chunk splitting incorrectly can create waterfall loading (chunk A imports B imports C)
- Dynamic `import()` with variable specifiers defeats tree-shaking
- Source maps double the total build size — must be uploaded to Sentry, not served publicly
- `optimizeDeps` pre-bundling list must be maintained as dependencies change
- CommonJS packages require Vite's `vite-plugin-commonjs` or `@rollup/plugin-commonjs`

## Design Diagram

```mermaid
flowchart LR
    Src[Source TS/TSX] --> Vite[Vite dev server\nESM native]
    Vite -->|HMR < 50ms| Browser[Browser]

    Src --> Build[vite build\nRollup production]
    Build --> TS[esbuild\nTypeScript → JS]
    Build --> Tree[Rollup\ntree shaking]
    Build --> Split[Code splitting\nmanualChunks]
    Build --> Minify[esbuild minify\n+ CSS purge]

    Split --> Chunks[dist/assets/\nindex.[hash].js\nvendor.[hash].js\npayments.[hash].js]

    Chunks --> Sentry[Upload source maps\nsentry-vite-plugin]
    Chunks --> CDN[CDN deploy]
    Chunks --> Budget[Bundle size check\nCI gate]
```

## Code Sample

### TypeScript — Production-grade Vite config

```typescript
// vite.config.ts
import { defineConfig, splitVendorChunkPlugin } from 'vite';
import react from '@vitejs/plugin-react-swc';      // SWC for faster transpilation
import tsconfigPaths from 'vite-tsconfig-paths';   // resolve TypeScript path aliases
import { visualizer } from 'rollup-plugin-visualizer';
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    tsconfigPaths(),
    splitVendorChunkPlugin(),

    // Bundle visualiser — generates stats.html after build
    visualizer({
      filename: 'dist/stats.html',
      gzipSize: true,
      brotliSize: true,
      open: false,
    }),

    // Upload source maps to Sentry — only in production build
    ...(mode === 'production'
      ? [
          sentryVitePlugin({
            org: process.env.SENTRY_ORG,
            project: process.env.SENTRY_PROJECT,
            authToken: process.env.SENTRY_AUTH_TOKEN,
            release: { name: process.env.VITE_APP_VERSION },
            sourcemaps: {
              // Upload source maps, then delete them from the build output
              filesToDeleteAfterUpload: ['dist/assets/*.js.map'],
            },
          }),
        ]
      : []),
  ],

  resolve: {
    alias: {
      '@': '/src',           // resolves '@/components/...' to 'src/components/...'
    },
  },

  build: {
    target: 'es2022',
    sourcemap: true,             // needed for Sentry source map upload
    chunkSizeWarningLimit: 150,  // warn on chunks > 150 kB
    minify: 'esbuild',

    rollupOptions: {
      output: {
        // Manual chunk grouping — separates vendor cache from app code
        manualChunks: (id) => {
          // React ecosystem — large, stable, cache-friendly
          if (id.includes('node_modules/react') || id.includes('node_modules/react-dom')) {
            return 'react-vendor';
          }
          // TanStack (Query + Virtual) — changes with feature work
          if (id.includes('@tanstack')) {
            return 'tanstack';
          }
          // Radix UI primitives
          if (id.includes('@radix-ui')) {
            return 'radix';
          }
          // i18n runtime
          if (id.includes('i18next') || id.includes('react-i18next')) {
            return 'i18n';
          }
          // Sentry SDK
          if (id.includes('@sentry')) {
            return 'sentry';
          }
        },
        // Consistent file naming for CDN long-term caching
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash].[ext]',
      },
    },
  },

  optimizeDeps: {
    // Pre-bundle slow CJS deps during dev startup
    include: ['lodash-es', 'date-fns'],
  },

  server: {
    port: 3000,
    proxy: {
      '/api': { target: 'http://localhost:4000', changeOrigin: true },
    },
  },
}));
```

### TypeScript — tsconfig.json for strict, path-aliased TypeScript

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["vite/client"]
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### YAML — GitHub Actions CI with bundle size reporting

```yaml
# .github/workflows/build.yml
name: Build & Bundle

on:
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm

      - run: npm ci

      - name: TypeScript type check
        run: npx tsc --noEmit

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build
        env:
          VITE_ENV: ci
          VITE_APP_VERSION: ${{ github.sha }}

      - name: Check bundle size budgets
        run: node scripts/checkBundleSize.ts

      - name: Upload bundle stats
        uses: actions/upload-artifact@v4
        with:
          name: bundle-stats
          path: dist/stats.html

      - name: Comment PR with bundle sizes
        uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```
