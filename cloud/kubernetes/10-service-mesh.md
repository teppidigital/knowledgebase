# Service Mesh — Istio, Linkerd & Cilium

## Category
Networking, Security, Kubernetes

## Context

A service mesh adds cross-cutting concerns (mTLS, traffic management, observability) to pod-to-pod communication without changing application code. Three leading options differ in architecture and scope.

| Concern | Istio | Linkerd | Cilium eBPF |
|---------|-------|---------|-------------|
| Data plane | Envoy sidecar | micro-proxy (Rust) | eBPF (no sidecar) |
| CPU overhead | High (~100 m/pod) | Low (~5 m/pod) | Near-zero |
| mTLS | ✅ (SPIFFE/SPIRE) | ✅ automatic | ✅ via Wireguard |
| Traffic shifting | ✅ VirtualService | ✅ TrafficSplit (SMI) | Partial |
| Circuit breaking | ✅ DestinationRule | Partial | ✗ |
| Ambient mesh | ✅ v1.21+ | ✗ | N/A |
| Web UI | Kiali | Buoyant Cloud | Hubble UI |
| Learning curve | High | Low | Medium |

**Istio Ambient Mesh** (sidecar-free mode) uses a per-node `ztunnel` for L4 and an optional `waypoint` proxy for L7 — reduces overhead significantly.

**Cilium** replaces kube-proxy entirely using eBPF, providing network policy at kernel level with the **Hubble** observability layer.

---

## Pros

- **mTLS everywhere**: all pod-to-pod traffic is encrypted and authenticated with zero app changes.
- **Istio VirtualService / DestinationRule**: precise traffic shaping — canary splits, AB testing, retries, timeouts all in YAML.
- **Circuit breaker and outlier detection** (Istio): automatically eject unhealthy pods from the load-balancing pool.
- **Linkerd** is lightweight and easy to install — ideal entry point for teams new to service meshes.
- **Cilium eBPF** eliminates the iptables tax and sidecars entirely — better throughput and lower latency at scale.
- **SPIFFE workload identity** ensures only the authorized service can talk to another — zero-trust networking.

---

## Cons

- **Sidecar injection** doubles pod count from a port/file-descriptor perspective; adds startup dependencies.
- **Istio complexity**: VirtualService, DestinationRule, Gateway, AuthorizationPolicy, PeerAuthentication — large CRD surface.
- **Debugging mTLS** is harder — plain-text TCP dumps no longer work; need `istioctl proxy-config` or Kiali.
- **Cilium** requires kernel ≥5.10; managed clusters (EKS, AKS) may require specific node image.
- Certificate rotation failures can silently break service-to-service communication.
- **Header-based routing** (Istio) doesn't work without a Gateway/VirtualService for TCP-level services.

---

## Design Diagram

```mermaid
flowchart LR
    subgraph PodA["Pod A"]
        APP_A["App Container"]
        PROXY_A["Envoy Sidecar<br/>(15001)"]
        APP_A <-->|iptables redirect| PROXY_A
    end

    subgraph PodB["Pod B"]
        APP_B["App Container"]
        PROXY_B["Envoy Sidecar<br/>(15001)"]
        APP_B <-->|iptables redirect| PROXY_B
    end

    PROXY_A <-->|mTLS (SPIFFE certs)| PROXY_B

    ISTIOD["istiod<br/>(Control Plane)<br/>cert management<br/>xDS config push"]
    ISTIOD -->|xDS| PROXY_A
    ISTIOD -->|xDS| PROXY_B
```

---

## Code Sample

### Istio — VirtualService for canary traffic split

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10
```

### Istio — DestinationRule (circuit breaking + outlier detection)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 50
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50   # Eject at most 50% of pods at once
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

### Istio — AuthorizationPolicy (deny-by-default + allow specific paths)

```yaml
# Deny all traffic in production namespace by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec: {}
---
# Allow order-service → payment-service on /api/v1/payments only
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/payments"]
```

### Linkerd — inject mesh into a namespace

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# Install control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check

# Enable automatic proxy injection for namespace
kubectl annotate namespace production \
  linkerd.io/inject=enabled

# Check mesh traffic stats
linkerd viz stat deployments -n production
linkerd viz routes deployment/order-service -n production
```

### Cilium — CiliumNetworkPolicy (L7 aware)

```yaml
# Block all ingress except GET /health and POST /orders from frontend namespace
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: order-service-l7
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: order-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: GET
                path: "/health"
              - method: POST
                path: "/orders"
```

---

## Related

- [02 — Networking & Services](./02-networking-services.md) — NetworkPolicy (Kubernetes native) vs CiliumNetworkPolicy
- [05 — RBAC & Security](./05-rbac-security.md) — AuthorizationPolicy complements RBAC at the network layer
- [09 — Observability](./09-observability.md) — Istio / Linkerd export metrics to Prometheus; Hubble complements Grafana
