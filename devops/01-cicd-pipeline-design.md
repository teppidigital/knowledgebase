# CI/CD Pipeline Design

## Category

DevOps, Continuous Integration, Continuous Delivery, Pipeline Architecture, Artifact Management

## Context

**Continuous Integration (CI)** is the practice of automatically building and testing every commit. **Continuous Delivery (CD)** extends CI by automating the release process so every passing build can be deployed to production at any time. Together they form the CI/CD pipeline — the assembly line of modern software delivery.

### Pipeline stages

```
Code push → Build → Test → Package → Publish → Deploy staging → E2E/smoke → Deploy prod
```

| Stage                 | Activities                                 | Failure =        |
| --------------------- | ------------------------------------------ | ---------------- |
| **Build**             | Compile, transpile, lint                   | Block merge      |
| **Unit test**         | Fast isolated tests (<5 min)               | Block merge      |
| **Integration test**  | Service-level tests with real DB/MQ        | Block merge      |
| **Package**           | Docker image build, Helm chart package     | Block publish    |
| **Publish**           | Push to container registry, artifact store | Block deployment |
| **Deploy staging**    | Apply Helm/Terraform to staging cluster    | Block prod       |
| **E2E / smoke**       | User-journey tests against staging         | Block prod       |
| **Deploy production** | Progressive rollout (canary / blue-green)  | Rollback         |

### Branching strategy and pipeline triggers

| Strategy        | Main branches                         | Pipeline trigger                 | Suitable for                   |
| --------------- | ------------------------------------- | -------------------------------- | ------------------------------ |
| **Trunk-based** | `main` only                           | Every commit to main             | High-velocity, feature-flagged |
| **Git Flow**    | `main`, `develop`, `release/*`        | `develop` → CI; `release/*` → CD | Scheduled releases             |
| **GitHub Flow** | `main` + short-lived feature branches | PR → CI; merge → CD              | SaaS / continuous delivery     |

### Pipeline principles

| Principle                  | Description                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------ |
| **Fast feedback**          | Unit tests must complete in <5 minutes; total CI in <15 min                                |
| **Single source of truth** | Pipeline defined in code (`.github/workflows`, `.gitlab-ci.yml`) — no click-ops            |
| **Reproducible builds**    | Same source + same dependencies → identical artifact; use lockfiles and pinned base images |
| **Artifact promotion**     | Build once, promote the same artifact across environments — no rebuilds                    |
| **Immutable artifacts**    | Published images/charts are never overwritten; use semantic versioning + SHA tags          |
| **Parallelism**            | Run independent jobs (lint, test, security scan) in parallel to minimise wall time         |

### Artifact versioning

```
Image tag strategy:
  ghcr.io/myorg/api:1.4.2                   ← semver release tag
  ghcr.io/myorg/api:sha-abc1234             ← immutable commit SHA tag
  ghcr.io/myorg/api:main-20260324-abc1234   ← branch + date + SHA
```

---

## Pros

- **Rapid feedback loop**: Developers know within minutes if their change breaks the build or tests.
- **Reduced integration risk**: Small, frequent merges avoid the "big bang" merge problem.
- **Deployment confidence**: Every prod deploy uses a tested, versioned artifact — not ad-hoc scripts.
- **Audit trail**: Pipeline run history is a complete audit log of who deployed what, when, and with what result.
- **Rollback path**: Immutable artifacts make rollback fast — redeploy the previous tag.

---

## Cons

- **Slow pipelines kill adoption**: If CI takes >20 minutes, developers stop waiting and batch changes — negating the benefit.
- **Flaky tests erode trust**: Intermittently failing tests cause teams to re-run instead of fix, masking real failures.
- **Pipeline complexity**: Large monorepos with many services require complex pipeline orchestration (path filtering, monorepo tooling).
- **Secrets management in CI**: Injecting secrets into pipeline runners securely requires careful setup.
- **Cost**: Cloud CI minutes for large test suites and container builds can be significant.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph Developer
        A[git push / PR]
    end

    subgraph CI — Parallel jobs
        B[Lint + type check]
        C[Unit tests]
        D[SAST + secrets scan]
        B & C & D --> E{All pass?}
    end

    subgraph Build & Publish
        E -->|Yes| F[Docker build\nmulti-arch]
        F --> G[Push to registry\n:sha-abc1234]
        F --> H[Helm chart package\n+ push to OCI registry]
    end

    subgraph Deploy — Staging
        G & H --> I[Helm upgrade\nstaging cluster]
        I --> J[Integration tests]
        J --> K[E2E / smoke tests]
        K --> L{Pass?}
    end

    subgraph Deploy — Production
        L -->|Yes| M[ArgoCD sync\nproduction — canary 5%]
        M --> N[Metrics check\nerror rate < 0.1%]
        N --> O[Full rollout\n100%]
    end

    E -->|No| P[Block merge\nnotify author]
    L -->|No| Q[Block prod\nnotify on-call]
```

---

## Code Sample

### YAML — GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yaml
name: CI / CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Parallel CI jobs ──────────────────────────────────────────────────────
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: npm }
      - run: npm ci --ignore-scripts
      - run: npm run lint
      - run: npm run typecheck

  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22", cache: npm }
      - run: npm ci --ignore-scripts
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with: { name: coverage, path: coverage/ }

  # ── Build & Publish (main branch only) ───────────────────────────────────
  build-publish:
    name: Build & Push Image
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.push.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=ref,event=branch,suffix=-{{date 'YYYYMMDD'}}-{{sha}}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true # SLSA provenance attestation
          sbom: true # Attach SBOM to image

  # ── Deploy staging ────────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    needs: build-publish
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v4
        with: { version: v3.14.0 }

      - name: Configure kubectl
        uses: azure/k8s-set-context@v4
        with:
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Helm upgrade
        run: |
          helm upgrade --install api-service ./charts/api-service \
            --namespace staging \
            --set image.tag=sha-${{ github.sha }} \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --atomic \
            --timeout 5m \
            --wait

      - name: Run integration tests
        run: npm run test:integration
        env:
          API_URL: https://api-staging.example.com
```

### TypeScript — Pipeline Helper: Semantic Version from Conventional Commits

```typescript
// scripts/next-version.ts
// Determines the next semantic version from conventional commit messages

import { execSync } from "child_process";

type BumpType = "major" | "minor" | "patch";

function getCommitsSinceLastTag(): string[] {
  try {
    const lastTag = execSync("git describe --tags --abbrev=0", {
      encoding: "utf8",
    }).trim();
    return execSync(`git log ${lastTag}..HEAD --pretty=format:%s`, {
      encoding: "utf8",
    })
      .split("\n")
      .filter(Boolean);
  } catch {
    // No previous tag — return all commits
    return execSync("git log --pretty=format:%s", { encoding: "utf8" })
      .split("\n")
      .filter(Boolean);
  }
}

function determineBump(commits: string[]): BumpType {
  if (commits.some((c) => c.startsWith("BREAKING CHANGE") || c.includes("!")))
    return "major";
  if (commits.some((c) => c.startsWith("feat"))) return "minor";
  return "patch";
}

function bumpVersion(current: string, bump: BumpType): string {
  const [major, minor, patch] = current
    .replace(/^v/, "")
    .split(".")
    .map(Number);
  if (bump === "major") return `${major + 1}.0.0`;
  if (bump === "minor") return `${major}.${minor + 1}.0`;
  return `${major}.${minor}.${patch + 1}`;
}

const commits = getCommitsSinceLastTag();
const bump = determineBump(commits);
const current = execSync(
  "git describe --tags --abbrev=0 2>/dev/null || echo v0.0.0",
  { encoding: "utf8" },
).trim();
const next = bumpVersion(current, bump);

console.log(next); // Output consumed by CI pipeline: VERSION=$(npx ts-node scripts/next-version.ts)
```
