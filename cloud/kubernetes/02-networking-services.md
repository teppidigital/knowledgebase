# Kubernetes Networking & Services

## Category
Networking, Kubernetes, Ingress

## Context

Kubernetes networking follows four rules: every pod gets a unique cluster-wide IP; pods can communicate with all other pods without NAT; nodes can communicate with all pods; and the IP a pod sees itself as is the same IP others see it as.

**Service types:**
| Type | Accessibility | Use case |
|------|-------------|----------|
| `ClusterIP` | Cluster-internal only | Default; service-to-service traffic |
| `NodePort` | Cluster nodes (`nodeIP:port`) | Dev/debug; non-cloud environments |
| `LoadBalancer` | External cloud LB (L4) | Production external exposure (TCP/UDP) |
| `ExternalName` | DNS CNAME alias | Proxy to external service by DNS name |
| `Headless` (`clusterIP: None`) | Pod IPs directly | StatefulSet DNS; service discovery |

**Ingress vs Gateway API:**
| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Maturity | Stable | GA (v1.2+) |
| HTTP routing | Path, host | Path, host, headers, methods, params |
| gRPC routing | Controller-specific | Native `GRPCRoute` |
| TLS | Annotation-based | Native `TLSRoute` |
| Traffic splitting | Controller-specific | Native weight-based split |
| Multi-tenancy | Single owner | Role separation (Infra / App teams) |

**NetworkPolicy** controls L3/L4 traffic between pods using label selectors. It is additive — no policy = allow all; any policy = deny everything not explicitly allowed.

---

## Pros

- **Services**: Stable virtual IP and DNS (`svc.namespace.svc.cluster.local`) even as pods restart.
- **Ingress**: Single load balancer entry point with TLS termination and path-based routing.
- **Gateway API**: Expressive routing rules, native traffic splitting, cleaner multi-tenancy model.
- **NetworkPolicy**: Zero-trust segmentation without changing application code.
- **Headless services**: Pod-level DNS for stateful workloads without a proxy layer.

---

## Cons

- **NodePort**: Limited to ports 30000–32767; not suitable for production.
- **Ingress**: No standard for traffic splitting or header routing — each controller uses annotations differently.
- **Gateway API**: Requires installing a compatible controller (Istio, Envoy Gateway, Nginx Gateway Fabric).
- **NetworkPolicy**: Requires a CNI that enforces it (Calico, Cilium, Weave); the default `kubenet` does not.
- **LoadBalancer**: Creates one cloud LB per service — expensive at scale. Use Ingress or Gateway API for HTTP.

---

## Important Production Details

### Networking decisions that matter early

| Decision | Why it matters | Recommendation |
|----------|----------------|----------------|
| **Ingress vs Gateway API** | Defines L7 routing model for years | Start with Gateway API for new platforms; keep Ingress for legacy |
| **kube-proxy mode** | Impacts scale and latency | Prefer IPVS (or eBPF dataplane with Cilium) for large clusters |
| **`externalTrafficPolicy`** | Preserves client IP vs load distribution | Use `Local` when source IP is needed; ensure enough pods per node |
| **NetworkPolicy baseline** | Prevents lateral movement | Default deny in every production namespace |
| **DNS dependency** | DNS outages break all service-to-service calls | Run CoreDNS with multiple replicas across zones |
| **Egress control model** | Data exfiltration and compliance | Explicit egress allowlist + egress gateway/NAT policy |

### Best practices

- Define a platform-wide convention for service exposure: internal (`ClusterIP`), north-south HTTP (Gateway/Ingress), and L4 public endpoints (`LoadBalancer`).
- Apply `default-deny` ingress and egress policies first, then allow only required flows.
- Reserve `NodePort` for troubleshooting and non-cloud edge cases only.
- Use separate ingress classes/gateways per trust boundary (public vs private).
- Treat DNS as critical platform infrastructure: anti-affinity, PDB, and latency SLOs.

### Common anti-patterns

- Exposing many services as `LoadBalancer` instead of using one ingress/gateway tier.
- Creating policies that allow all egress (`0.0.0.0/0`) for convenience.
- Forgetting DNS egress rules, causing hidden service resolution failures.
- Mixing multiple ingress controllers without clear ownership and class boundaries.
- Relying on pod IP allowlists while pods are ephemeral by design.

---

## Design Diagram

```mermaid
flowchart TD
    subgraph External["External Traffic"]
        USER["Client"]
    end

    subgraph Cluster["Kubernetes Cluster"]
        LB["LoadBalancer Service<br/>(Cloud LB / L4)"]
        ING["Ingress Controller<br/>(Nginx / Traefik)"]

        subgraph NS1["namespace: production"]
            SVC_A["Service: api-svc<br/>(ClusterIP)"]
            SVC_B["Service: orders-svc<br/>(ClusterIP)"]
            SVC_C["Service: pg-headless<br/>(Headless)"]

            POD_A1["api pod 1"]
            POD_A2["api pod 2"]
            POD_B["orders pod"]
            POD_DB0["postgres-0"]
            POD_DB1["postgres-1"]
        end
    end

    USER -->|HTTPS :443| LB
    LB   --> ING
    ING  -->|/api/*| SVC_A
    ING  -->|/orders/*| SVC_B
    SVC_A --> POD_A1 & POD_A2
    SVC_B --> POD_B
    SVC_C -.->|DNS: postgres-0.pg-headless| POD_DB0
    SVC_C -.->|DNS: postgres-1.pg-headless| POD_DB1
```

---

## Code Sample

### ClusterIP Service + Deployment

```yaml
# service/api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: production
spec:
  selector:
    app: api
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

### Ingress with TLS and path routing

```yaml
# ingress/api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls-cert         # cert-manager populates this
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders-svc
                port:
                  number: 80
```

### Gateway API — HTTPRoute with traffic splitting

```yaml
# gateway-api/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: production
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: api-tls-cert
---
# gateway-api/httproute-canary.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: production
spec:
  parentRefs:
    - name: prod-gateway
  hostnames:
    - api.myapp.com
  rules:
    - backendRefs:
        - name: api-svc-stable
          weight: 90
        - name: api-svc-canary
          weight: 10         # 10% canary split
```

### NetworkPolicy — deny all, then allow selectively

```yaml
# netpol/deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}         # Applies to all pods in namespace
  policyTypes:
    - Ingress
    - Egress
---
# netpol/allow-api-to-orders.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-orders
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: orders
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 8080
---
# netpol/allow-egress-dns.yaml
# Allow all pods to reach kube-dns
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### Service source IP preservation with `externalTrafficPolicy`

```yaml
# service/public-api.yaml
apiVersion: v1
kind: Service
metadata:
  name: public-api
  namespace: production
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # preserve real client source IP
  selector:
    app: public-api
  ports:
    - name: https
      port: 443
      targetPort: 8443
```

### NetworkPolicy — namespace-scoped and external egress allowlist

```yaml
# netpol/allow-api-egress-only-needed-destinations.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress-only-needed-destinations
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    # Allow egress to payments namespace over HTTPS
    - to:
        - namespaceSelector:
            matchLabels:
              name: payments
      ports:
        - protocol: TCP
          port: 443

    # Allow DNS to kube-dns
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # Allow egress to one external partner endpoint only
    - to:
        - ipBlock:
            cidr: 203.0.113.42/32
      ports:
        - protocol: TCP
          port: 443
```

---

## Troubleshooting Playbook

### Quick checks

```bash
# Service and endpoint wiring
kubectl -n production get svc,ep,endpointslices

# Ingress/Gateway resources and controller status
kubectl -n production get ingress,gateway,httproute
kubectl -n ingress-nginx get pods

# DNS checks from inside a pod
kubectl -n production run dns-debug --rm -it --image=busybox:1.36 -- nslookup orders-svc.production.svc.cluster.local

# NetworkPolicy visibility
kubectl -n production get networkpolicy

# Describe failed routing object
kubectl -n production describe ingress api-ingress
kubectl -n production describe httproute api-route
```

### Symptom to likely cause

| Symptom | Likely cause |
|---------|--------------|
| Service exists but no traffic reaches pods | Selector mismatch; no endpoints |
| Ingress returns 404/default backend | Host/path rule mismatch or wrong ingress class |
| Intermittent cross-namespace connectivity | Missing/overly strict NetworkPolicy rules |
| Everything fails after enabling default deny | DNS egress was not explicitly allowed |
| Source IP lost at app layer | `externalTrafficPolicy` left at `Cluster` |

---

## Related

- [01 — Workloads](./01-workloads.md) — Workload resources exposed through Services
- [05 — RBAC & Security](./05-rbac-security.md) — NetworkPolicy for zero-trust pod segmentation
- [10 — Service Mesh](./10-service-mesh.md) — L7 mTLS and traffic management beyond NetworkPolicy
- [13 — Pod Reliability](./13-pod-reliability.md) — Topology spread and affinity for availability
