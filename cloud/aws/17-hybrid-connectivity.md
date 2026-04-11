# AWS Hybrid Connectivity Patterns

## Category
Cloud Native, Networking, AWS Direct Connect, Transit Gateway, VPN, Hybrid Cloud

## Context

**Hybrid connectivity** refers to secure, reliable network paths between on-premises datacentres (or other clouds) and AWS. The three primary technologies are Direct Connect, Site-to-Site VPN, and Transit Gateway. They are frequently combined to achieve high availability, cost efficiency, and routing flexibility.

---

## Direct Connect (DX)

### Connection Types

| Type | Bandwidth options | How provisioned |
|------|-----------------|----------------|
| **Dedicated** | 1, 10, 100 Gbps | AWS issues a Letter of Authorisation (LOA); customer/partner completes cross-connect at DX location |
| **Hosted** | 50 Mbps – 10 Gbps | Ordered via AWS Direct Connect Partner; sub-1 Gbps available |

Hosted connections share the partner's port. Dedicated connections have a physical port exclusively allocated to you. **MACsec** (Layer 2 encryption) is available only on dedicated connections (not hosted).

### Virtual Interfaces (VIFs)

| VIF Type | What it connects to | Access |
|----------|-------------------|--------|
| **Private VIF** | Virtual Private Gateway (VGW) in a specific VPC | Private IPs in that VPC |
| **Public VIF** | AWS public endpoint IPs | S3, DynamoDB, SNS, SQS, EC2 public IPs — via AWS backbone, not internet |
| **Transit VIF** | Direct Connect Gateway → Transit Gateway | Multiple VPCs across regions via TGW |

**Rule:** One DX connection can have one VIF of each type simultaneously.

### Direct Connect Gateway (DXGW)

DXGW is a **global, account-level resource** that acts as an intermediary between your DX connection and AWS networks.

```
On-premises router
  → DX dedicated port
    → Transit VIF
      → Direct Connect Gateway (global)
        ├── Transit Gateway (eu-west-1) — all VPCs in EU
        ├── Transit Gateway (us-east-1) — all VPCs in US
        └── Virtual Private Gateway (ap-southeast-1) — single VPC in APAC
```

**Key limits:**
- 1 DXGW can attach to up to **20 Transit Gateways** (1 TGW per attachment).
- 1 DXGW can attach to up to **10 Virtual Private Gateways**.
- Multiple DX connections from different locations can share one DXGW for HA.

### High Availability Patterns

| Pattern | What it survives | Cost |
|---------|----------------|------|
| Single DX port | Nothing (single point of failure) | Lowest |
| 2 ports, same DX location | Port failure | Low |
| 2 ports, different DX locations | Location failure | Medium |
| 2 DX locations + VPN backup | DX outage + VPN as emergency path | Medium |
| 2 DX locations, dual routers per side | Full redundancy per AWS recommendation | High |

**AWS maximum resiliency:** 4 connections total — 2 DX locations × 2 dedicated connections per location.

### BGP and Routing

Direct Connect uses **BGP (Border Gateway Protocol)** for dynamic routing.

- You advertise your on-premises CIDRs to AWS via BGP.
- AWS advertises VPC CIDRs (private VIF) or AWS public IP ranges (public VIF) back to you.
- **BGP AS Path Prepending:** Artificially lengthen the AS path for one connection → traffic prefers the shorter-path connection. Used for active/passive routing.
- **Local Preference:** Set on your on-premises router to prefer one DX connection over another.

---

## Site-to-Site VPN

### Properties

| Property | Value |
|----------|-------|
| Tunnels per VPN connection | Always 2 (active/active or active/passive) |
| Max bandwidth per tunnel | ~1.25 Gbps (theoretical) |
| Protocol | IPsec (IKEv1 or IKEv2) |
| Routing | Static or BGP (dynamic) |
| Attachment point | VGW (per-VPC) or Transit Gateway (shared) |

**Customer Gateway (CGW):** Represents your on-premises VPN device in AWS. Defined by the public IP of your device and its BGP ASN (if using BGP).

### ECMP for Throughput Scaling

When attached to **Transit Gateway** (not VGW) with BGP routing, Equal-Cost Multi-Path (ECMP) distributes traffic across multiple tunnels from multiple VPN connections.

```
4 VPN connections × 2 tunnels each × 1.25 Gbps = 10 Gbps aggregate throughput
(All tunnels advertising equal-cost routes to TGW)
```

ECMP is not supported with VGW (only TGW attachment supports it).

### Accelerated VPN

Routes VPN traffic via AWS Global Accelerator anycast edge nodes — ingress onto the AWS global backbone at the nearest edge rather than traversing the public internet to the VPN endpoint.

**When to use:** VPN endpoints geographically distant from the AWS region (intercontinental); reduces latency and packet loss.

---

## AWS Transit Gateway (TGW)

### Core Architecture

```
          [vpc-prod-a]   [vpc-prod-b]   [vpc-staging]   [vpc-shared-services]
               |               |              |                  |
               └───────────────┴──────────────┴──────────────────┘
                                      │
                              [Transit Gateway]
                                      │
                    ┌─────────────────┼──────────────────┐
               [VPN attachment]  [DX attachment]   [TGW Peering]
                    │                │                    │
              [On-premises]    [DX Gateway]       [TGW us-east-1]
```

### Attachments

| Type | Notes |
|------|-------|
| VPC | Creates ENIs in each AZ of the VPC; route table updates needed in VPC |
| Site-to-Site VPN | Terminates VPN on TGW; supports ECMP with BGP |
| Direct Connect (Transit VIF → DXGW) | Connect DXGW to TGW |
| TGW Peering | Connect TGWs in different regions; static routes only (no BGP) |
| AWS Cloud WAN | Managed global network using TGW as edge locations |

### Route Tables and Segmentation

Every attachment associates with exactly one TGW route table. Route tables control which attachments can communicate.

**Isolated VPCs with shared services:**
```
Route Table: "isolated"
  → Associate: vpc-prod-a, vpc-prod-b, vpc-staging
  → Routes: 10.1.0.0/16 → shared-services-vpc
  → NO routes between prod and staging (isolated from each other)

Route Table: "shared-services"
  → Associate: vpc-shared-services
  → Routes: 10.0.0.0/8 → propagated from all VPCs (can reach all)
```

**Blackhole routes:** Drop traffic to specific prefixes. Use to prevent VPCs from communicating even if they share a route table.

### VPC Peering vs Transit Gateway

| | VPC Peering | Transit Gateway |
|--|------------|----------------|
| Transitive routing | NO | YES |
| Full mesh connections | N*(N-1)/2 | N connections |
| Route propagation | Manual | Automatic via propagation |
| Cross-region | Yes | Yes (TGW peering, static routes) |
| Cost | Free same-region data | $0.05/GB + $0.05/hour/attachment |
| Bandwidth | No limit | 50 Gbps per VPC attachment |
| Support for on-prem traffic | Via VGW per VPC | Centralised (one DX/VPN connection) |

---

## Route 53 Resolver

Provides **bi-directional DNS resolution** between on-premises and VPC private hosted zones.

### Inbound Endpoints

```
On-premises DNS server
  → Conditional forwarder rule: *.internal.aws.example.com → [Inbound endpoint IPs]
  → Route 53 Resolver resolves against private hosted zones in the VPC
  → Returns private IP of the AWS resource
```

Up to 2 ENIs created across AZs for HA. Each ENI has an IP that DNS queries are sent to.

### Outbound Endpoints

```
EC2 instance in VPC
  → DNS query: db01.corp.example.com
  → Route 53 Resolver checks forwarding rules
  → Matches rule: corp.example.com → forward to 10.0.1.53 (on-premises DNS)
  → On-premises DNS resolves and returns the IP
```

### Sharing Rules via RAM

Resolver rules can be shared org-wide via RAM so all VPCs automatically have the same forwarding rules without per-VPC configuration.

---

## AWS PrivateLink

### Interface Endpoints vs Gateway Endpoints

| | Interface Endpoint | Gateway Endpoint |
|--|-------------------|-----------------|
| Services | All AWS services + PrivateLink services | S3 and DynamoDB only |
| Implementation | ENI in your subnet (private IP) | Route table entry (no ENI) |
| Cost | $0.01/hour/AZ + $0.01/GB | Free |
| Accessible from on-premises | YES (via DX/VPN) | NO |
| Accessible via TGW | YES | NO |
| Cross-region | No (endpoint is in one region) | No |

### Common Interface Endpoints (exam-relevant)

- `com.amazonaws.region.ec2` — EC2 API (for instance in private subnet to call AWS APIs)
- `com.amazonaws.region.ecr.dkr` + `ecr.api` — Docker image pulls without NAT gateway
- `com.amazonaws.region.s3` — S3 access without NAT (interface variant; gateway is free)
- `com.amazonaws.region.sts` — STS calls without NAT
- `com.amazonaws.region.ssm` + `ssmmessages` — Session Manager without NAT

### PrivateLink Endpoint Services

Expose your own NLB-backed service to consumers in other VPCs or accounts without peering or internet. Consumers see an interface endpoint — your service's private IP structure is never exposed.

```
Your service → NLB → Endpoint Service
Consumer VPC → Interface Endpoint → requests forwarded to your NLB → your service
```

Access control: Endpoint service accepts connections from specific AWS account IDs (allowlist). You can require manual approval of connection requests.

---

## Pros

- **Direct Connect:** Consistent network performance (no internet congestion); up to 100 Gbps; private connectivity; public VIF avoids internet egress for AWS service calls.
- **Site-to-Site VPN:** Lower cost than DX; instant provisioning; Internet path (variable latency but acceptable for many use cases); ECMP for horizontal throughput scaling.
- **Transit Gateway:** Eliminates full-mesh VPC peering complexity; centralised routing and policy; enables transitive routing; supports hybrid connectivity at scale.
- **PrivateLink:** Service exposure without VPC peering; no IP overlap concerns; works across accounts and organisations.

---

## Cons

- **Direct Connect:** Lead time for provisioning (weeks to months); requires partner or colocation; higher cost than VPN.
- **Site-to-Site VPN:** Limited to 1.25 Gbps per tunnel; public internet path means variable latency; not suitable for latency-sensitive workloads.
- **Transit Gateway:** Per-attachment cost at scale; cross-region TGW peering is static routes only; BGP required for DX/VPN attachments.
- **PrivateLink (Gateway Endpoints):** Not accessible from on-premises via DX or VPN — use interface endpoints for shared access.

---

## Design Patterns

### Hub-and-Spoke with Centralised Inspection

```
[spoke-vpc-a]  [spoke-vpc-b]  [spoke-vpc-c]
      |               |              |
      └───────────────┴──────────────┘
                      │
             [Transit Gateway]
                      │
             [Inspection VPC]
           (AWS Network Firewall)
                      │
              [Internet Gateway]
```

All egress traffic routes through the Inspection VPC where Network Firewall applies stateful rules before traffic reaches the internet or on-premises.

### DX + VPN Active/Passive HA

```
On-premises
├── DX connection (primary path — low latency, high bandwidth)
│     → DX Gateway → Transit Gateway
└── Site-to-Site VPN (backup path — internet, higher latency)
      → Transit Gateway

BGP: DX routes have shorter AS path → preferred by default
On DX failure: BGP convergence routes traffic via VPN automatically (< 60 seconds)
```

---

## When to Use

| Scenario | Choice |
|----------|--------|
| < 10 VPCs, no on-prem | VPC Peering or TGW (VPC Peering is cheaper for few VPCs) |
| 10+ VPCs, complex routing | Transit Gateway |
| On-premises connectivity, < 1 Gbps, quick setup | Site-to-Site VPN |
| On-premises connectivity, consistent latency, > 1 Gbps | Direct Connect |
| Both on-prem and multi-VPC | TGW + DX (Transit VIF → DXGW → TGW) |
| Access S3/DynamoDB from private subnet | Gateway Endpoint (free) |
| Access other AWS services from private subnet | Interface Endpoint |
| Access on-premises from S3? | Not possible via endpoint; use DX public VIF instead |
| Expose your service to other accounts privately | PrivateLink Endpoint Service |

---

## Common Mistakes

- Using Private VIF when you need to reach multiple VPCs or regions (use Transit VIF → DXGW → TGW instead).
- Expecting Gateway Endpoints (S3/DynamoDB) to be accessible from on-premises — they are route table entries only, not reachable via DX/VPN.
- Forgetting that TGW peering (cross-region) requires static routes — BGP is not supported.
- Using VPC Peering expecting transitive routing — VPC Peering is never transitive.
- Forgetting that each VPN connection has 2 tunnels; ECMP across 4 connections = 8 tunnels = ~10 Gbps (but TGW required for ECMP).
