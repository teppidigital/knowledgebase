# GitOps

## Category

DevOps, GitOps, ArgoCD, Flux, Declarative Deployment, Pull-Based Delivery, Reconciliation

## Context

**GitOps** is an operational model where the entire desired state of a system — infrastructure, Kubernetes manifests, Helm chart values, and configuration — is stored in Git, and an automated operator continuously reconciles the live system to match that desired state.

### GitOps principles (OpenGitOps v1.0)

| Principle                   | Description                                                                |
| --------------------------- | -------------------------------------------------------------------------- |
| **Declarative**             | The system is described declaratively — _what_, not _how_                  |
| **Versioned and immutable** | Desired state stored in Git with full history; any version can be restored |
| **Pulled automatically**    | Software agents (not people or CI systems) pull and apply changes          |
| **Continuously reconciled** | Agents detect drift and correct it automatically                           |

### Push vs Pull deployment

| Model                     | Mechanism                                                   | Risks                                                       |
| ------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------- |
| **Push** (traditional CD) | CI pipeline authenticates to cluster and `kubectl apply`    | Pipeline credentials are powerful; drift is not detected    |
| **Pull** (GitOps)         | Operator inside the cluster watches Git and applies changes | No outbound cluster credentials in CI; drift auto-corrected |

### GitOps tools

| Tool             | Description                                               | Strengths                                                                        |
| ---------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **ArgoCD**       | Declarative GitOps controller for Kubernetes with full UI | Rich UI, multi-cluster, SSO, RBAC, webhook sync                                  |
| **Flux v2**      | CNCF GitOps toolkit — composable controllers              | GitRepository, Kustomization, HelmRelease CRDs; Flagger for progressive delivery |
| **Weave GitOps** | Flux-based with enterprise UI and policy                  | Built on Flux; COTS support                                                      |

### GitOps repository patterns

| Pattern                    | Structure                                              | When to use                                             |
| -------------------------- | ------------------------------------------------------ | ------------------------------------------------------- |
| **Monorepo**               | All apps + infra in one repo                           | Small org, tight coupling                               |
| **App repo + config repo** | App source in one repo; K8s manifests in separate repo | Most common; separates deployment artefacts from source |
| **Repo-per-environment**   | `env/staging` and `env/prod` are separate repos        | Max isolation; harder to promote                        |

### Promotion flow (app → staging → prod)

```
Dev pushes code → CI builds image → CI updates config repo (image tag bump)
     → ArgoCD detects config repo change → Syncs to staging
     → Tests pass → PR to promote to prod config
     → Merge PR → ArgoCD syncs to prod
```

---

## Pros

- **Single source of truth**: The Git history IS the deployment history — every change is auditable with `git log`.
- **Drift detection and auto-correction**: Operator detects manual changes to the cluster and reverts them.
- **Self-service deployments**: Developers deploy by opening a PR — no cluster access required.
- **Instant rollback**: `git revert` or point ArgoCD to a previous commit SHA → system rolls back.
- **Disaster recovery**: Re-bootstrap a cluster by pointing ArgoCD at the config repo — full state restored.

---

## Cons

- **Config repo drift discipline**: If engineers bypass Git (direct `kubectl apply`), GitOps breaks — requires enforcement.
- **Secret management complexity**: Secrets cannot be stored in Git in plaintext — requires Sealed Secrets, SOPS, or External Secrets Operator.
- **Slow feedback for complex changes**: Config promotion via PRs adds latency versus direct push deployments.
- **Multi-cluster complexity**: Managing many clusters with ArgoCD ApplicationSets and cluster-specific overrides requires careful structuring.
- **Learning curve**: Teams used to push-based CI/CD need to shift mental models around pull-based reconciliation.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph Developer workflow
        A[Feature branch] -->|CI passes| B[Merge to main]
        B -->|CI updates image tag| C[(Config repo\ngit commit)]
    end

    subgraph Config Repository
        C --> D[staging/api/values.yaml\nimage.tag: sha-abc1234]
        D -->|PR approved| E[production/api/values.yaml\nimage.tag: sha-abc1234]
    end

    subgraph Cluster — Staging
        F[ArgoCD controller] -->|watch + diff| D
        F -->|reconcile| G[Staging workloads]
    end

    subgraph Cluster — Production
        H[ArgoCD controller] -->|watch + diff| E
        H -->|reconcile| I[Production workloads]
    end

    subgraph Drift detection
        J[Someone runs\nkubectl edit directly]
        J -->|ArgoCD detects OutOfSync| F
        F -->|Auto-sync reverts| G
    end
```

---

## Code Sample

### YAML — ArgoCD Application (Helm-based)

```yaml
# gitops/argocd/apps/api-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-service-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io # Cascade delete managed resources
spec:
  project: staging

  source:
    repoURL: https://github.com/myorg/config-repo.git
    targetRevision: main
    path: staging/api-service # Path in config repo
    helm:
      valueFiles:
        - values.yaml
        - values-staging.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true # Remove resources deleted from Git
      selfHeal: true # Revert manual changes (drift correction)
      allowEmpty: false # Never sync an empty application
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    # Ignore fields that change at runtime without being a real drift
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas # HPA manages replica count
```

### YAML — ArgoCD ApplicationSet (multi-environment from one template)

```yaml
# gitops/argocd/appsets/api-service-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-service
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: staging
            cluster: https://staging-cluster.example.com
            namespace: api-staging
            autoSync: "true"
          - env: production
            cluster: https://prod-cluster.example.com
            namespace: api-production
            autoSync: "false" # Production requires manual sync trigger

  template:
    metadata:
      name: api-service-{{ .env }}
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/config-repo.git
        targetRevision: main
        path: "environments/{{ .env }}/api-service"
      destination:
        server: "{{ .cluster }}"
        namespace: "{{ .namespace }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: "{{ .autoSync }}"
```

### YAML — Flux HelmRelease with image automation

```yaml
# gitops/flux/api-service/helmrelease.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: api-service
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: api-service
      version: ">=1.0.0"
      sourceRef:
        kind: HelmRepository
        name: myorg-charts
        namespace: flux-system

  values:
    image:
      repository: ghcr.io/myorg/api
      tag: latest # Overridden by ImagePolicy below

  # Rollback on Helm upgrade failure
  rollback:
    timeout: 5m
    cleanupOnFail: true
    recreate: false

---
# Image policy: track semver patch releases only
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: api-service
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: api-service
  policy:
    semver:
      range: ">=1.0.0 <2.0.0" # Stay on 1.x, auto-upgrade patches

---
# Automation: update HelmRelease.values.image.tag in Git when new image detected
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: config-repo
  git:
    checkout:
      ref: { branch: main }
    commit:
      author:
        name: Flux Image Automation Bot
        email: flux@example.com
      messageTemplate: |
        chore(deps): update api-service image to {{ range .Updated.Images }}{{ .NewTag }}{{ end }}
    push:
      branch: main
  update:
    strategy: Setters
    path: ./gitops/flux
```

### Bash — Config Repo Image Tag Promotion Script

```bash
#!/usr/bin/env bash
# scripts/gitops/promote.sh
# Updates image tag in the config repo after a successful CI build
# Called by CI pipeline: ./scripts/gitops/promote.sh staging sha-abc1234

set -euo pipefail

ENV="${1:?Usage: $0 <env> <image-tag>}"
TAG="${2:?Usage: $0 <env> <image-tag>}"
CONFIG_REPO="${CONFIG_REPO_URL:?CONFIG_REPO_URL env var required}"
VALUES_FILE="environments/${ENV}/api-service/values.yaml"

# Clone config repo using deploy key
git clone "${CONFIG_REPO}" config-repo
cd config-repo

git config user.name  "CI Promotion Bot"
git config user.email "ci@example.com"

# Update image tag using yq (YAML processor)
yq -i ".image.tag = \"${TAG}\"" "${VALUES_FILE}"

git add "${VALUES_FILE}"
git commit -m "chore(${ENV}): promote api-service to ${TAG}

Automated promotion from CI build ${GITHUB_RUN_ID:-unknown}
Source SHA: ${GITHUB_SHA:-unknown}"

git push origin main
echo "Promoted ${TAG} to ${ENV}"
```
