# Kubernetes Configuration & Secrets

## Category
Configuration, Security, Kubernetes

## Context

**ConfigMap** holds non-sensitive configuration as key-value pairs or files. **Secret** holds sensitive data (base64-encoded, optionally encrypted at rest with KMS). Both can be mounted as environment variables or volumes.

| Concern | ConfigMap | Secret | External Secrets Operator |
|---------|-----------|--------|--------------------------|
| Sensitive data | No | Yes (base64) | Yes (encrypted in external store) |
| Encryption at rest | No (unless KMS configured) | Optional (envelope encryption) | Yes (AWS SM, Vault, GCP SM) |
| Rotation | Manual redeploy | Manual redeploy | Automatic sync + optional pod reload |
| Audit trail | None | None | Full in the external provider |
| Kubernetes RBAC control | Yes | Yes | Yes + provider IAM |

**External Secrets Operator (ESO)** syncs secrets from external stores (AWS Secrets Manager, Vault, GCP Secret Manager, Azure Key Vault) into Kubernetes Secrets automatically.

**Sealed Secrets** (Bitnami) encrypts a Secret at rest in Git using the cluster's public key — only the cluster can decrypt it. Good for GitOps-native workflows.

---

## Pros

- **ConfigMap**: Decouples config from image; hot-reload via volume watch without redeploy.
- **Secret**: Native Kubernetes primitive; integrates with RBAC; easy to rotate by updating and rolling out.
- **ESO**: Secrets never stored in Git; single source of truth in a dedicated secrets manager; supports TTL-based auto-rotation.
- **Sealed Secrets**: Zero external dependencies; works fully offline; secrets can live in Git safely.
- **Volume-mounted**: Files appear as a mountPath; applications read them as regular files — no code changes needed.

---

## Cons

- **Secret**: Base64 is not encryption — anyone with `get secret` RBAC can read it in plain text.
- **ConfigMap size limit**: 1 MiB per object.
- **Volume mount latency**: Kubernetes syncs volume mounts every `syncPeriod` (default 1 min) — not instant on update.
- **ESO**: Adds an operator dependency and cloud IAM complexity; ExternalSecret objects must be reconciled.
- **Sealed Secrets**: Cluster-specific encryption — backups must include the sealing key; re-sealing needed per cluster.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph External["External Stores"]
        VAULT["HashiCorp Vault"]
        AWS_SM["AWS Secrets Manager"]
    end

    subgraph Cluster["Kubernetes Cluster"]
        ESO["External Secrets\nOperator"]
        ES["ExternalSecret CR"]
        SEC["Secret\n(synced)"]
        POD["Pod"]
        CM["ConfigMap\n(app config)"]
    end

    AWS_SM & VAULT -->|ESO reads| ESO
    ES -->|defines source| ESO
    ESO -->|creates/updates| SEC
    SEC -->|env or volume| POD
    CM  -->|env or volume| POD
```

---

## Code Sample

### ConfigMap — application config

```yaml
# configmap/order-service-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  LOG_LEVEL: "info"
  FEATURE_NEW_CHECKOUT: "true"
  DB_MAX_CONNECTIONS: "20"
  # Multi-line file as a config map entry
  app.properties: |
    server.port=8080
    metrics.enabled=true
    cache.ttl=300
```

### Secret — manually managed (development only)

```yaml
# secret/order-service-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
stringData:                          # stringData auto-encodes to base64
  DATABASE_URL: "postgresql://user:pass@postgres:5432/orders"
  JWT_SECRET: "super-secret-jwt-key"
  STRIPE_API_KEY: "sk_live_..."
```

### Using ConfigMap + Secret in a Deployment

```yaml
# Deployment container spec (snippet)
spec:
  containers:
    - name: order-service
      envFrom:
        - configMapRef:
            name: order-service-config      # All keys become env vars
        - secretRef:
            name: order-service-secrets     # All keys become env vars
      # Or mount as files
      volumeMounts:
        - name: app-config
          mountPath: /app/config
          readOnly: true
  volumes:
    - name: app-config
      configMap:
        name: order-service-config
```

### External Secrets Operator — sync from AWS Secrets Manager

```yaml
# eso/secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:                              # IRSA on EKS
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
---
# eso/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: production
spec:
  refreshInterval: 1h                    # Sync every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: order-service-secrets          # Creates/updates this Secret
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/order-service
        property: database_url
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/order-service
        property: jwt_secret
```

### Sealed Secret — encrypt for GitOps

```bash
# Install kubeseal CLI
brew install kubeseal

# Create a plain Secret, pipe to kubeseal for encryption
kubectl create secret generic db-secret \
  --from-literal=password='s3cr3t' \
  --dry-run=client -o yaml \
  | kubeseal --controller-name sealed-secrets \
             --controller-namespace kube-system \
             --format yaml > sealed-db-secret.yaml

# sealed-db-secret.yaml is safe to commit to Git
# The cluster decrypts it automatically via the SealedSecrets controller
```

---

## Related

- [01 — Workloads](./01-workloads.md) — Deployments consume ConfigMaps and Secrets
- [05 — RBAC & Security](./05-rbac-security.md) — RBAC controls who can read Secrets
- [08 — GitOps](./08-gitops.md) — Sealed Secrets or ESO enables secret management in GitOps workflows
- [14 — Supply Chain Security](./14-supply-chain-security.md) — OPA Gatekeeper can enforce secret hygiene policies
