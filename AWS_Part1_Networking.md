# AWS Interview Prep — Part 1: Networking & VPC (Deep Dive)
> All levels: Basic → Advanced → Tricky Gotchas → Design Questions

---

## Q1. Walk me through the internals of a VPC. What actually happens when you create one?

A VPC is a logically isolated section of AWS backed by a software-defined network (SDN).

- AWS allocates your CIDR block to a **virtual router built into the Nitro hypervisor**.
- You get an implicit/main route table, default security group, and default NACL automatically.
- AWS **reserves 5 IPs per subnet**: network address (.0), VPC router (.1), DNS (.2), future use (.3), broadcast (.255).
- The VPC router lives at `x.x.x.1` of every subnet — highly available, fully managed by AWS.
- Subnets are **AZ-scoped** — a single subnet cannot span AZs.
- Traffic between subnets in the same VPC is routed by the VPC router with no extra gateway needed.

```
VPC 10.0.0.0/16
├── Public Subnet  10.0.1.0/24 (AZ-a) → route 0.0.0.0/0 → IGW
├── Private Subnet 10.0.2.0/24 (AZ-a) → route 0.0.0.0/0 → NAT GW
├── Public Subnet  10.0.3.0/24 (AZ-b) → route 0.0.0.0/0 → IGW
└── Private Subnet 10.0.4.0/24 (AZ-b) → route 0.0.0.0/0 → NAT GW
```

---

## Q2. Security Group vs NACL — differences and evaluation order?

| Feature | Security Group | NACL |
|---|---|---|
| Stateful? | Yes — return traffic auto-allowed | No — must explicitly allow both directions |
| Applied to | ENI (instance level) | Subnet boundary |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated together | Rules evaluated lowest number first, first match wins |
| Default | Deny all inbound, allow all outbound | Allow all in both directions |

**Evaluation order:** NACL is evaluated FIRST at the subnet boundary. Security Group is evaluated SECOND at the ENI. Both must allow traffic — neither "wins."

**Key gotcha:** NACLs are stateless. Allowing inbound 443 requires also allowing outbound **ephemeral ports 1024–65535** for return traffic. Forgetting this is the #1 NACL mistake.

---

## Q3. VPC Peering vs Transit Gateway — internals and when to use each?

**VPC Peering:**
- Direct 1:1 private connection between two VPCs (same/cross-account, same/cross-region).
- Traffic stays entirely on AWS backbone — no bandwidth limit, no extra hop.
- **Non-transitive by design** — if A↔B and B↔C, A cannot reach C through B.
- Scales as N*(N-1)/2 — 20 VPCs = 190 peering connections.
- No per-hour cost, only data transfer charges.

**Transit Gateway:**
- Hub-and-spoke: all VPCs attach to TGW, TGW routes between them.
- Supports thousands of attachments (VPCs, VPNs, Direct Connect, SD-WAN).
- **Multiple route tables** for network segmentation (prod can't reach dev).
- Supports **multicast** — unique feature not available with peering.
- Per-attachment hourly fee + per-GB data processing fee.
- Inter-region peering of TGWs supported.

```
Peering (3 VPCs = 3 connections):    TGW (3 VPCs = 3 attachments):
  A ←→ B                               A ──┐
  A ←→ C                               B ──┼── TGW ── VPN/DX
  B ←→ C                               C ──┘
```

**Choose peering:** ≤5 VPCs, simple, lowest cost.
**Choose TGW:** Many VPCs, hybrid connectivity, centralized inspection, network segmentation.

---

## Q4. TRICKY — VPC-A peers VPC-B, VPC-B peers VPC-C. Can A reach C?

**No.** Peering is non-transitive. Even with both peering connections active, the VPC router in B will not forward packets from A destined for C — there is no route for C in A's route table, and even if you manually added one, AWS blocks transitive routing through peerings at the infrastructure level.

**Common wrong answer:** "Add a static route in VPC-B's route table pointing C's CIDR to the A-B peering." This does NOT work. The peering connection enforces non-transitivity regardless of routes.

**Fixes:**
1. Direct peering between A and C.
2. Transit Gateway — attach all three, TGW routes transitively.

---

## Q5. TRICKY — Can two VPCs with overlapping CIDRs be peered?

**No.** AWS rejects the peering request. Both VPCs must have non-overlapping CIDRs because the route table would be ambiguous.

**Workarounds:**
- Re-IP one VPC (best long-term fix, painful).
- Use **PrivateLink** — exposes a specific service endpoint, not a whole CIDR, so IP overlap is irrelevant.
- Transit VPC with NAT/proxy (complex, legacy pattern).

---

## Q6. How does DNS work inside a VPC? What is Route 53 Resolver?

Every VPC has an **Amazon-provided DNS resolver** at `base_CIDR + 2` (e.g., `10.0.0.2`).

- Handles resolution of: AWS internal names (EC2 hostnames, RDS/ElastiCache endpoints), private hosted zones, and public DNS (forwarded to Route 53 public resolvers).
- `enableDnsSupport = true` → instances use VPC DNS server.
- `enableDnsHostnames = true` → instances get public DNS hostnames like `ec2-x-x-x-x.compute-1.amazonaws.com`.

**Route 53 Resolver:**
- **Inbound Endpoint:** ENIs in your VPC that on-premises DNS servers forward queries to. Lets on-prem resolve your AWS private hosted zone names.
- **Outbound Endpoint:** ENIs in your VPC that forward specific DNS queries out to on-premises DNS servers. Lets AWS workloads resolve on-prem names.
- **Resolver Rules:** Forward rules (send domain X to IP Y) and system rules (AWS handles domain Z).

```
On-Premises DNS                    Route 53 Resolver
192.168.1.53                       Inbound EP: 10.0.1.5, 10.0.2.5
     │                                    │
     └──── query db.aws.internal ────────►│──► Private Hosted Zone
                                          │    returns 10.0.3.22

AWS Lambda                         Outbound EP: 10.0.1.6, 10.0.2.6
     │                                    │
     └──── query app.corp.local ─────────►│──► 192.168.1.53 (on-prem DNS)
```

---

## Q7. How does AWS Direct Connect work technically? How does BGP factor in?

Direct Connect (DX) is a **dedicated physical connection** from your data center to AWS through a colocation facility.

**Physical path:**
1. Your router → colo cross-connect → AWS DX router → AWS backbone.

**Virtual Interfaces (VIFs) — logical BGP sessions over the physical link:**
- **Private VIF** → connects to a VPC via Virtual Private Gateway (VGW) or TGW.
- **Public VIF** → connects to AWS public IPs (S3, DynamoDB) without internet.
- **Transit VIF** → connects to Transit Gateway for multi-VPC access.

**BGP:**
- DX uses eBGP. AWS's ASN for private VIFs is **7224**.
- AWS advertises your VPC CIDR(s) to your router.
- You advertise your on-premises CIDRs to AWS.
- **Route preference:** Use **BGP communities** or **AS path prepending** for active/passive failover across two DX connections.
- Longer AS path = less preferred. Prepend your passive connection's AS path so AWS prefers the active one.

**Resiliency tiers:**
```
Single DX connection          → Single point of failure (dev/test only)
Two DX connections, same colo → Handles link failure, not colo failure
Two DX connections, diff colo → Maximum resiliency (production standard)
DX + VPN backup               → Cost-effective, VPN as cold standby
```

**Link Aggregation Group (LAG):** Bundle multiple DX ports into one logical connection for higher bandwidth. All ports must be at the same colo, same speed.

---

## Q8. What is AWS PrivateLink and how is it different from VPC Peering?

PrivateLink exposes a specific service from one VPC to consumers **without peering, without IGW, without NAT, without exposing entire CIDRs**.

**How it works:**
1. **Provider** puts their service behind a **Network Load Balancer**, then creates a **VPC Endpoint Service**.
2. **Consumer** creates an **Interface VPC Endpoint** → AWS provisions one or more **ENIs** with private IPs in the consumer's subnets.
3. Consumer app talks to those ENI IPs → traffic flows over AWS backbone → hits provider's NLB → service.

```
Consumer VPC                          Provider VPC
  App → ENI (10.0.1.23) ──PrivateLink──► NLB → Service
         (Interface Endpoint)
```

| | PrivateLink | VPC Peering |
|---|---|---|
| Exposes | Single service | Entire VPC CIDR |
| Direction | One-way (consumer → provider) | Bidirectional |
| CIDR overlap | Allowed | Not allowed |
| Transitive | N/A | No |
| Use case | SaaS, cross-account service sharing | Full network connectivity |

---

## Q9. TRICKY — What happens when you delete a VPC endpoint mid-use?

**Gateway Endpoint (S3, DynamoDB):** Deleting removes the route table entry (prefix list route). In-flight requests fail immediately. New requests route via NAT Gateway or internet — if no internet path exists, they fail with connection timeout.

**Interface Endpoint (PrivateLink):** Deleting removes the ENI. Existing TCP connections get a TCP RST. DNS override reverts — the service FQDN resolves back to the public IP. If your security groups/NACLs block public internet, all traffic to that service fails.

---

## Q10. What is the difference between an IGW and a NAT Gateway at the packet level?

**Internet Gateway:**
- Performs **1:1 NAT** — maps instance's private IP to its assigned Elastic IP or public IP.
- The instance must have a public IP.
- **Bidirectional** — inbound connections from internet can be initiated.
- The instance kernel uses its private IP; IGW swaps to public IP in transit, and reverses for inbound.

**NAT Gateway:**
- Performs **many-to-one PAT (Port Address Translation)** — like a home router.
- Multiple private IPs → one Elastic IP, differentiated by ephemeral port tracking.
- **Unidirectional** — only outbound-initiated connections work. Internet cannot initiate a connection to a private instance through NAT GW.
- Supports up to 55,000 simultaneous connections per destination IP.
- **AZ-scoped** — a NAT GW in AZ-a cannot serve instances in AZ-b without cross-AZ charges.

**Critical design rule:** Deploy **one NAT Gateway per AZ** for HA and to avoid inter-AZ data transfer costs.

---

## Q11. DESIGN — 50 accounts need centralized internet egress with inspection and logging. How?

```
┌─── Shared Services Account ──────────────────────────────┐
│                                                           │
│  Egress VPC                                               │
│  ┌─────────────────────────────────┐                     │
│  │  Internet Gateway               │                     │
│  │         ↑                       │                     │
│  │  NAT Gateway (per AZ)           │                     │
│  │         ↑                       │                     │
│  │  AWS Network Firewall           │                     │
│  │  (domain allowlist, IDS rules)  │                     │
│  │         ↑                       │                     │
│  │  TGW Attachment                 │                     │
│  └─────────────────────────────────┘                     │
│                                                           │
│  Transit Gateway (shared via AWS RAM to all 50 accounts) │
└───────────────────────────────────────────────────────────┘
         │          │          │
    Account A   Account B   Account C  (× 50)
    VPCs with   Route:
    0.0.0.0/0 → TGW attachment
```

**Key details:**
- TGW shared via **AWS Resource Access Manager (RAM)** — no need to create per-account TGWs.
- **TGW Route Tables**: Spoke route table has `0.0.0.0/0 → Egress VPC`. Blackhole routes prevent spoke-to-spoke traffic.
- **Network Firewall** in Egress VPC inspects all outbound traffic — domain filtering, IDS/IPS rules (Suricata-compatible).
- **VPC Flow Logs** on Egress VPC → S3 → Athena for compliance queries.
- All 50 accounts' outbound traffic has a single NAT IP pool → easy to whitelist at external partners.

---

## Q12. DESIGN — Hybrid DNS: on-prem resolves AWS names, AWS resolves on-prem names.

```
On-Premises (192.168.0.0/16)                AWS VPC (10.0.0.0/16)
┌──────────────────────┐                  ┌───────────────────────────┐
│  Corporate DNS        │                  │  Route 53 Resolver         │
│  192.168.1.53         │                  │                           │
│                       │◄── DX/VPN ──────►│  Inbound Endpoints:       │
│  Conditional forwarder│                  │    10.0.1.5 (AZ-a)        │
│  *.internal.aws →     │                  │    10.0.2.5 (AZ-b)        │
│  10.0.1.5 (inbound EP)│                  │                           │
│                       │                  │  Outbound Endpoints:      │
│  Handles *.corp.local │◄─────────────────│    10.0.1.6 (AZ-a)        │
│  natively             │                  │    10.0.2.6 (AZ-b)        │
└──────────────────────┘                  │                           │
                                          │  Resolver Rule:           │
                                          │  *.corp.local →           │
                                          │  forward to 192.168.1.53  │
                                          │                           │
                                          │  Private Hosted Zone:     │
                                          │  *.internal.aws           │
                                          └───────────────────────────┘
```

**Flow A — On-prem resolves AWS name:**
1. App queries `db.internal.aws` → corporate DNS.
2. Conditional forwarder sends to R53 Inbound Endpoint `10.0.1.5` over DX.
3. R53 Resolver returns the RDS private IP.

**Flow B — AWS resolves on-prem name:**
1. Lambda queries `payroll.corp.local` → VPC DNS (`10.0.0.2`).
2. R53 Resolver checks rules → matches `*.corp.local` → forwards via Outbound Endpoint.
3. Corporate DNS `192.168.1.53` returns answer over DX.

---

## Q13. AWS Network Firewall vs Security Groups vs NACLs vs WAF — when does each apply?

| | SG | NACL | Network Firewall | WAF |
|---|---|---|---|---|
| OSI Layer | L4 | L4 | L3–L7 | L7 (HTTP/S only) |
| Scope | ENI | Subnet | VPC / TGW | CloudFront / ALB / APIGW |
| Stateful | Yes | No | Yes (flow tracking) | Yes |
| L7 inspection | No | No | Yes (domain, IDS/IPS) | Yes (OWASP, regex, rate) |
| East-West traffic | Yes | Yes | Yes | No |
| Managed threat rules | No | No | Suricata-compatible | AWS Managed Rules |

**These are complementary layers, not alternatives:**
- WAF at the CDN/load balancer edge for HTTP exploits.
- Network Firewall at VPC boundary for deep packet inspection and domain filtering.
- NACLs for stateless subnet-level deny rules (block known bad CIDRs).
- Security Groups for micro-segmentation at the ENI level.

---

## Q14. TRICKY — What is Anycast and how do Route 53, CloudFront, and Global Accelerator use it?

**Anycast:** Same IP address announced from multiple geographic locations. The network routes each user to the topologically nearest instance of that IP.

**Route 53:** Name server IPs are anycasted from all AWS edge PoPs globally. Your DNS query automatically hits the nearest PoP — reduces DNS lookup latency from hundreds of ms to single-digit ms.

**CloudFront:** Your distribution's IPs are anycasted. Users connect to the nearest edge PoP for TLS termination and caching. Even for dynamic (cache-miss) content, the response travels over AWS backbone from origin — faster than public internet routing.

**Global Accelerator:**
- Gives you **two static anycast IPs** (unlike CloudFront's domain-only model).
- Traffic enters AWS backbone at the nearest PoP and stays on it end-to-end.
- Better than CloudFront for: TCP/UDP non-HTTP workloads, gaming, VoIP, applications requiring static IP whitelisting.
- Instant failover (<30s) without DNS TTL delays — client keeps the same IP, Global Accelerator re-routes to healthy endpoint.

| | CloudFront | Global Accelerator |
|---|---|---|
| Caches content | Yes | No |
| Static IPs | No | Yes (2 anycast IPs) |
| Protocols | HTTP/HTTPS | TCP, UDP, HTTP |
| Best for | CDN, web apps | Non-HTTP, static IPs, instant failover |

---

## Q15. DESIGN — Multi-tier regulated financial app: network segmentation, inspection, hybrid connectivity.

```
On-Premises DC
┌──────────────┐
│ Corp Router  │──── Direct Connect (2 locations, LAG) ────┐
│ + VPN backup │                                           │
└──────────────┘                                           ▼
                                              ┌─── Transit Gateway ───┐
                                              │  Route Tables:        │
                                              │  - Prod RT            │
                                              │  - Inspection RT      │
                                              │  - Mgmt RT            │
                                              └──┬────┬────┬────┬────┘
                                                 │    │    │    │
                                 ┌───────────────┘    │    │    └──────────────┐
                                 ▼                    ▼    ▼                   ▼
                           Ingress VPC          Inspection VPC          Data VPC
                           ┌──────────┐         ┌────────────┐         ┌──────────┐
                           │ WAF+ALB  │         │ AWS Network│         │ Aurora   │
                           │ (public) │         │ Firewall   │         │ ElastiCache│
                           └────┬─────┘         └────────────┘         └──────────┘
                                │                                           ▲
                                └──────── App VPC ──────────────────────────┘
                                          ┌──────────┐
                                          │ EKS/ECS  │
                                          │ (private)│
                                          └──────────┘
```

**Traffic flow (north-south, internet → app):**
`Internet → WAF → ALB (Ingress VPC) → TGW → Inspection VPC (Network Firewall) → TGW → App VPC → TGW → Data VPC`

**TGW route table design:**
- App VPC route table: `0.0.0.0/0 → Inspection VPC`. No direct route to Data VPC.
- Inspection VPC route table: inspects, then forwards to App or Data.
- This enforces **all traffic is inspected** — east-west and north-south.

**Additional controls:**
- PrivateLink for all AWS service access (no public S3/KMS endpoints).
- VPC Flow Logs on all VPCs → S3 → Athena.
- Direct Connect with two physical locations for maximum resiliency.
- VPN as tertiary backup (different transport path).

---

## Q16. TRICKY — What is the difference between a Gateway VPC Endpoint and an Interface VPC Endpoint?

| | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|---|---|---|
| Services | S3 and DynamoDB only | 100+ AWS services + custom services |
| Implementation | Route table entry (prefix list) | ENI with private IP in your subnet |
| Cost | Free | Hourly per ENI + per-GB charge |
| DNS | No DNS override | Private DNS overrides service FQDN |
| Accessible from | Within the VPC only | VPN, DX, VPC peering (with DNS config) |
| Security | Route + resource policy | Security group on ENI + resource policy |

**Gateway Endpoint gotcha:** It only works within the VPC where it's created. If you have on-premises connectivity via DX and want on-prem servers to access S3 privately, you need an **Interface Endpoint for S3** (not a Gateway Endpoint), because the Gateway Endpoint is not routable from outside the VPC.

**Interface Endpoint DNS override:** When you enable private DNS, the service's public FQDN (e.g., `s3.us-east-1.amazonaws.com`) resolves to the endpoint ENI's private IP instead of the public S3 IP. Traffic stays private without any code change.

---

## Q17. TRICKY — Your Lambda in a VPC cannot reach the internet or AWS services. What are the possible causes?

This is a very common production issue. Checklist:

1. **No NAT Gateway:** Lambda in a private subnet needs a NAT Gateway for internet access. Lambdas in VPCs do NOT get public IPs. No NAT = no internet.

2. **NAT Gateway in wrong subnet:** NAT GW must be in a **public subnet** (one with a route to IGW). If you put NAT GW in a private subnet, it has no path to the internet itself.

3. **Route table missing:** Private subnet's route table must have `0.0.0.0/0 → NAT Gateway ID`. If this route is missing, traffic goes nowhere.

4. **Security Group too restrictive:** Lambda's SG must allow outbound to the target (port 443 for HTTPS). Default SG allows all outbound but custom SGs might not.

5. **NACL blocking:** NACL on Lambda's subnet must allow outbound 443 AND inbound ephemeral ports (1024–65535) for return traffic (NACLs are stateless).

6. **VPC Endpoint needed:** For AWS service access (S3, DynamoDB, KMS, SSM), use VPC Endpoints to avoid routing through NAT GW entirely. Cheaper and more secure.

7. **No VPC Endpoint + no NAT:** If using VPC Endpoints for some services but not all, the missing services need NAT or their own endpoint.

---

## Q18. What is VPC Lattice and when would you use it?

**VPC Lattice** (newer service) is a fully managed application networking layer that connects services across VPCs and accounts without VPC peering, PrivateLink, or managing load balancers per service.

**How it works:**
- Create a **Service Network** — a logical grouping.
- Associate **Services** (backed by EC2, ECS, Lambda, ALB, etc.) to the network.
- Associate **VPCs** to the Service Network.
- Any resource in an associated VPC can discover and call any service in the network.
- Auth policies control which services/clients can communicate.
- Built-in: request routing, health checks, access logging.

**vs PrivateLink:**
- PrivateLink: expose ONE service to consumers via NLB. Manual per-service setup.
- VPC Lattice: expose MANY services across MANY VPCs with a single network construct. Better for microservices at scale.

**Use case:** Large microservices platform across multiple accounts/VPCs where setting up individual PrivateLink endpoints for every service would be operationally unmanageable.

---

## Q19. TRICKY — Explain what happens during a VPC Flow Log capture. What is NOT captured?

**What IS captured:**
- Source/destination IP, source/destination port, protocol, packet count, byte count, start/end time, action (ACCEPT/REJECT), log status.
- Captures at the ENI level (or subnet/VPC level which aggregates ENIs).
- Both accepted and rejected traffic.

**What is NOT captured (important for exams and interviews):**
- Traffic to/from 169.254.169.254 (instance metadata service).
- Traffic to/from 169.254.169.123 (Amazon Time Sync).
- DHCP traffic.
- DNS traffic to the VPC DNS resolver (10.0.0.2).
- Traffic between a Windows instance and the AWS KMS activation server.
- Traffic between a VPC mirror target and source (mirrored traffic is not logged separately).
- Amazon-provided DNS server traffic.

**Gotcha:** Flow logs show connection-level data, not payload. They tell you a connection was made but not WHAT was in the packets. For payload inspection, use Traffic Mirroring or Network Firewall.

**Storage destinations:** CloudWatch Logs, S3, or Kinesis Data Firehose (newest — near real-time streaming to OpenSearch, Splunk, etc.).

---

## Q20. DESIGN — Design a Zero Trust network architecture on AWS.

Traditional perimeter security: trust everything inside the network. Zero Trust: **trust nothing, verify everything**.

```
Zero Trust Architecture on AWS:

User/Device                  AWS
┌─────────────┐             ┌──────────────────────────────────────┐
│ Employee    │             │                                       │
│ + Device    │──mTLS/OIDC─►│  AWS Verified Access                  │
│ Posture     │             │  (Identity + device posture check)   │
│ Check       │             │         │                            │
└─────────────┘             │         ▼                            │
                            │  Internal App (no public exposure)  │
                            │  ├── IAM auth (SigV4)                │
                            │  ├── mTLS between services           │
                            │  ├── App Mesh (Envoy sidecar)        │
                            │  └── IRSA (per-pod IAM roles)        │
                            │                                       │
                            │  Data Layer                          │
                            │  ├── KMS encryption at rest          │
                            │  ├── TLS in transit (enforced)       │
                            │  └── IAM DB auth (no passwords)      │
                            │                                       │
                            │  Audit                               │
                            │  ├── CloudTrail (all API calls)      │
                            │  ├── VPC Flow Logs                   │
                            │  └── GuardDuty (threat detection)    │
                            └──────────────────────────────────────┘
```

**Key Zero Trust principles on AWS:**
- **AWS Verified Access:** Grants access to internal apps based on identity (IAM Identity Center/OIDC) AND device posture (CrowdStrike, Jamf) — without a VPN.
- **IRSA (IAM Roles for Service Accounts):** Each Kubernetes pod gets a scoped IAM role — no shared credentials.
- **Service-to-service mTLS:** App Mesh or Istio enforces mutual TLS between every service. Even internal lateral movement requires valid certs.
- **Private endpoints everywhere:** No service has a public endpoint. All AWS service access via VPC Endpoints.
- **Micro-segmentation:** Security groups at ENI level, each service only allows its specific callers.
