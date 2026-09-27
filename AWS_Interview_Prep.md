# AWS Senior Architect & Developer Interview Preparation Guide

> Covers: Fundamentals → Design → Complex Distributed Systems  
> Special emphasis on **Networking Internals**, all major AWS services, and architectural trade-offs.
> **300+ Questions | All AWS Services | Tricky Gotchas | System Design**

---

## Table of Contents

1. [Networking & VPC Internals — Basics](#1-networking--vpc-internals)
2. [Networking — Advanced & Tricky](#2-networking--advanced--tricky)
3. [Compute — EC2, ECS, EKS, Lambda — Basics](#3-compute--ec2-ecs-eks-lambda)
4. [Compute — Advanced & Auto Scaling](#4-compute--advanced--auto-scaling)
5. [Load Balancing — ALB, NLB, GWLB, CLB](#5-load-balancing)
6. [Storage — S3, EBS, EFS, FSx — Basics](#6-storage--s3-ebs-efs-fsx)
7. [Storage — Advanced & Tricky](#7-storage--advanced--tricky)
8. [Databases — RDS, Aurora, DynamoDB, ElastiCache, Redshift](#8-databases)
9. [Databases — Advanced & Tricky](#9-databases--advanced--tricky)
10. [Messaging & Streaming — SQS, SNS, EventBridge, Kinesis, MSK](#10-messaging--streaming)
11. [Serverless — Lambda, API Gateway, Step Functions, AppSync](#11-serverless--api-gateway--step-functions--appsync)
12. [Security — IAM, KMS, Secrets Manager, WAF, Shield, GuardDuty](#12-security)
13. [Security — Advanced & Tricky](#13-security--advanced--tricky)
14. [CDN & Edge — CloudFront, Global Accelerator, Lambda@Edge](#14-cdn--edge)
15. [Analytics & Big Data — Athena, Glue, EMR, Redshift, QuickSight, OpenSearch](#15-analytics--big-data)
16. [AI & ML — SageMaker, Bedrock, Rekognition, Comprehend](#16-ai--ml)
17. [Infrastructure as Code — CloudFormation, CDK, Terraform on AWS](#17-infrastructure-as-code)
18. [AWS Organizations, Control Tower, Landing Zone](#18-aws-organizations--control-tower--landing-zone)
19. [Observability — CloudWatch, X-Ray, CloudTrail, Config](#19-observability)
20. [High Availability & Disaster Recovery](#20-high-availability--disaster-recovery)
21. [Cost Optimization](#21-cost-optimization)
22. [Migration & Modernization](#22-migration--modernization)
23. [Containers Deep Dive — EKS Networking, Service Mesh, GitOps](#23-containers-deep-dive)
24. [Developer Tools — CodePipeline, CodeBuild, CodeDeploy, CodeArtifact](#24-developer-tools--cicd)
25. [System Design Questions (End-to-End)](#25-system-design-questions-end-to-end)
26. [Tricky Gotcha Questions](#26-tricky-gotcha-questions)
27. [Behavioral / Architecture Decision Questions](#27-behavioral--architecture-decision-questions)
28. [Quick Reference: Key Limits](#28-quick-reference-key-limits)

---

## 1. Networking & VPC Internals — Basics

### Q1. Walk me through the internals of a VPC. What actually happens when you create one?

**Answer:**  
A VPC is a logically isolated section of the AWS cloud backed by a software-defined network (SDN). Under the hood:

- AWS allocates a CIDR block (e.g., `10.0.0.0/16`) to a virtual router implemented in the **Nitro Hypervisor** or the underlying hardware.
- Each VPC gets a **implicit/main route table**, a **default security group**, and a **default NACL**.
- AWS reserves **5 IPs per subnet** (first 4 + last): network address, VPC router, DNS, future use, broadcast.
- The VPC router is a highly available distributed component — you never manage it, it lives at `x.x.x.1` of every subnet.
- Subnets are Availability Zone–scoped constructs; a subnet cannot span AZs.
- Traffic between subnets in the same VPC is routed by the implicit VPC router without any gateway needed.

```
VPC: 10.0.0.0/16
├── Public Subnet  10.0.1.0/24  (AZ-a)  → Route: 0.0.0.0/0 → IGW
├── Private Subnet 10.0.2.0/24  (AZ-a)  → Route: 0.0.0.0/0 → NAT GW
├── Public Subnet  10.0.3.0/24  (AZ-b)  → Route: 0.0.0.0/0 → IGW
└── Private Subnet 10.0.4.0/24  (AZ-b)  → Route: 0.0.0.0/0 → NAT GW
```

---

### Q2. What is the difference between a Security Group and a NACL? Which one takes precedence?

**Answer:**

| Feature | Security Group | NACL |
|---|---|---|
| Stateful? | Yes (return traffic auto-allowed) | No (must explicitly allow inbound + outbound) |
| Applied to | ENI (instance level) | Subnet boundary |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest number first) |
| Default | Deny all inbound, allow all outbound | Allow all |

**Order of evaluation:** Traffic hits the NACL **first** when entering/leaving a subnet, then the Security Group on the ENI. Neither "takes precedence" — both must allow the traffic.

**Key gotcha:** NACLs are stateless. If you allow inbound port 443, you must also allow outbound **ephemeral ports** (1024–65535) for return traffic.

---

### Q3. Explain VPC Peering vs Transit Gateway. When would you choose one over the other?

**Answer:**

**VPC Peering:**
- Direct 1:1 connection between two VPCs (same or cross-account, cross-region).
- Traffic stays on AWS backbone — low latency, no bandwidth bottleneck.
- **Non-transitive**: if A peers B and B peers C, A cannot reach C through B.
- No additional per-hour cost beyond data transfer.
- Scales poorly: N*(N-1)/2 peering connections for N VPCs (20 VPCs = 190 connections).

**Transit Gateway (TGW):**
- Hub-and-spoke model: all VPCs attach to TGW, which routes between them.
- Supports thousands of VPC attachments.
- Supports route table segmentation (isolate prod from dev).
- Supports VPN and Direct Connect attachments.
- Per-attachment hourly cost + per-GB data processing fee.
- Supports **multicast** (unique feature).

```
VPC Peering (small scale):        Transit Gateway (large scale):

  VPC-A ←→ VPC-B                    VPC-A ──┐
  VPC-A ←→ VPC-C                    VPC-B ──┤
  VPC-B ←→ VPC-C                    VPC-C ──┼── TGW ── On-Prem (VPN/DX)
  (3 connections)                   VPC-D ──┘
```

**Choose Peering** for: ≤ 5 VPCs, simple connectivity, lowest cost.  
**Choose TGW** for: many VPCs, centralized inspection, hybrid connectivity, network segmentation.

---

### Q4. How does DNS work inside a VPC? What is Route 53 Resolver?

**Answer:**  
- Every VPC has an **Amazon-provided DNS server** at the base of the CIDR + 2 (e.g., `10.0.0.2`).
- This resolver handles: AWS internal names (EC2 hostnames, RDS endpoints, S3), and forwards public DNS queries to Route 53 public resolvers.
- `enableDnsSupport` must be `true` for instances to use the VPC DNS server.
- `enableDnsHostnames` must be `true` for instances to get public DNS hostnames.

**Route 53 Resolver (formerly Route 53 DNS Resolver):**
- **Inbound endpoints**: Allow on-premises DNS servers to forward queries into your VPC (resolves AWS private hosted zone names from on-prem).
- **Outbound endpoints**: Allow your VPC to forward specific DNS queries to on-premises DNS servers.
- **Resolver Rules**: Forward rules (send domain X to on-prem resolver) and system rules.

```
On-Premises                     AWS VPC
DNS Server                      Route 53 Resolver
   │                                 │
   │── query: db.corp.internal ──→   │ Inbound Endpoint (ENI in VPC)
   │                                 │── forwards to on-prem DNS
   │                            Route 53 Private Hosted Zone
```

---

### Q5. Explain how AWS Direct Connect works at a technical level. How does BGP factor in?

**Answer:**  
Direct Connect (DX) is a dedicated network connection from your data center to AWS through a colocation facility.

**Physical path:**
1. Your router → colocation facility (cross-connect) → AWS DX router → AWS backbone.
2. A **Virtual Interface (VIF)** is a logical BGP session over the physical link:
   - **Private VIF**: connects to a VPC via VGW (Virtual Private Gateway) or TGW.
   - **Public VIF**: connects to AWS public services (S3, DynamoDB) without traversing internet.
   - **Transit VIF**: connects to Transit Gateway for multi-VPC access.

**BGP role:**
- DX uses **BGP (Border Gateway Protocol)** to exchange routes between your on-prem ASN and AWS's ASN (7224 for private VIFs).
- AWS advertises VPC CIDR ranges to your router.
- You advertise your on-prem CIDRs to AWS.
- You can influence routing with **BGP communities** and **AS path prepending**.
- For **Active/Passive failover** across two DX connections: prepend AS path on the passive connection so AWS prefers the active one.

**Resiliency options:**
- Single DX → single point of failure.
- Two DX connections to same location → resiliency against link failure.
- DX + VPN as backup → cost-effective but different performance.
- Two DX locations → maximum resiliency (recommended for production).

---

### Q6. What is AWS PrivateLink and how does it differ from VPC Peering?

**Answer:**  
PrivateLink allows you to expose a service (backed by an NLB) from one VPC to consumers in other VPCs **without peering, IGW, NAT, or VPN**.

**How it works:**
1. Provider: Creates an **NLB** in front of their service, then creates a **VPC Endpoint Service**.
2. Consumer: Creates an **Interface VPC Endpoint** — this provisions one or more **ENIs** with private IPs in the consumer's VPC.
3. Traffic flows: Consumer app → ENI (private IP in consumer VPC) → PrivateLink fabric (AWS backbone) → NLB → service.

**vs VPC Peering:**

| | PrivateLink | VPC Peering |
|---|---|---|
| Directionality | One-way (consumer → provider) | Bidirectional |
| CIDR overlap | Allowed | Not allowed |
| Transitive | N/A | No |
| Exposes | Single service/endpoint | Entire VPC CIDR |
| Use case | SaaS, cross-account service sharing | Network-level connectivity |

**Use PrivateLink when:** you want to expose a specific service without giving network-level access to your VPC.

---

### Q7. Design a multi-tier network architecture for a regulated financial application requiring network segmentation, inspection, and hybrid connectivity.

**Answer:**

```
                        ┌─────────────────────────────────────────────┐
                        │           AWS Account (Production)           │
                        │                                              │
  On-Premises DC        │  ┌──────────────────────────────────────┐   │
  ┌──────────┐          │  │         Transit Gateway               │   │
  │ Corp DC  │──DX──────┼──│  Route Table: Prod / Inspect / Mgmt  │   │
  │ Firewall │          │  └──┬──────┬──────┬──────┬──────────────┘   │
  └──────────┘          │     │      │      │      │                  │
                        │   [VPC]  [VPC]  [VPC]  [VPC]               │
                        │  Ingress Inspect  App    Data               │
                        │                                              │
  Internet ─────────────┼── ALB (WAF) ──► Ingress VPC                │
                        │                    │                         │
                        │           PrivateLink / TGW                 │
                        │                    ▼                         │
                        │             Inspection VPC                   │
                        │          (AWS Network Firewall)              │
                        │                    │                         │
                        │                    ▼                         │
                        │              App VPC                         │
                        │         ┌────────────────┐                  │
                        │         │  Private Subnet │                  │
                        │         │  ECS/EKS Tasks  │                  │
                        │         └────────┬────────┘                  │
                        │                  │                            │
                        │                  ▼                            │
                        │              Data VPC                         │
                        │         ┌────────────────┐                  │
                        │         │ RDS Aurora      │                  │
                        │         │ ElastiCache     │                  │
                        │         └────────────────┘                  │
                        └─────────────────────────────────────────────┘
```

**Key design decisions:**
- **Inspection VPC** with AWS Network Firewall for east-west and north-south inspection.
- **TGW route tables** enforce traffic flows: App VPC → Inspection VPC → Data VPC (never direct).
- **Ingress VPC** centralizes public-facing load balancers, isolating blast radius.
- **PrivateLink** for AWS service access (no public internet for S3, KMS, etc.).
- **Direct Connect** with two locations for max resiliency; VPN as tertiary backup.
- **VPC Flow Logs** on all VPCs → S3 → Athena for compliance queries.
- **Security Groups** are least-privilege; NACLs provide subnet-level deny rules for compliance.

---

### Q8. What is Anycast and how does AWS use it in Route 53 and CloudFront?

**Answer:**  
Anycast is a routing scheme where the same IP address is announced from multiple geographic locations, and the network routes traffic to the topologically nearest instance.

**Route 53:**  
- Route 53 name servers use anycast. The 4 name servers assigned to your hosted zone (`ns-xxx.awsdns-xx.com`) are announced from all AWS edge locations globally.
- DNS queries are automatically routed to the nearest Route 53 PoP, reducing latency.

**CloudFront:**  
- CloudFront distributions get a single domain (`xxxx.cloudfront.net`) resolved to edge PoP IPs via anycast.
- User queries are answered by the nearest edge location for DNS and TLS termination.
- Origin requests use **AWS backbone** (not public internet) to reach the origin — this is why CloudFront improves performance even for dynamic content.

**AWS Global Accelerator:**  
- Explicitly exposes **static anycast IPs** (two per accelerator).
- Traffic enters the AWS backbone at the nearest edge PoP and stays on it all the way to the regional endpoint.
- Better for TCP/UDP workloads, gaming, IoT, VoIP — anything where consistent low latency matters more than caching.

---

## 2. Networking — Advanced & Tricky

### Q9-A. TRICKY: You have VPC-A peered with VPC-B, and VPC-B peered with VPC-C. Can traffic flow from VPC-A to VPC-C? Why or why not, and how do you fix it?

**Answer:**  
**No.** VPC peering is non-transitive. Even though VPC-A → VPC-B → VPC-C connections exist, AWS does not route traffic transitively through VPC-B. The VPC router in VPC-B will not forward packets from VPC-A destined for VPC-C — it has no route for VPC-C in VPC-A's route table, and even if you tried to add one, the peering connection doesn't allow this.

**Why at a network level:** Peering is implemented as a private route — the VPC router only knows about directly peered VPC CIDRs. It has no concept of "forward through another VPC."

**Fixes:**
1. **Create a direct peering** between VPC-A and VPC-C.
2. **Use Transit Gateway** — attach all three VPCs, TGW routes transitively.
3. **Use a transit VPC** (older pattern) with EC2 routers acting as intermediaries — replaced by TGW.

**The trap:** Many candidates say "just add a route in VPC-B's route table pointing VPC-C traffic to the VPC-A peering connection." This does NOT work — AWS enforces non-transitive routing at the peering level.

---

### Q9-B. TRICKY: Can two VPCs with overlapping CIDR blocks be peered?

**Answer:**  
**No.** VPC peering requires non-overlapping CIDR blocks. If VPC-A is `10.0.0.0/16` and VPC-B is `10.0.0.0/16`, peering is rejected.

**Why:** The route table would have ambiguous routes — the VPC router couldn't determine whether `10.0.1.5` is local or in the peer.

**Workarounds:**
1. Re-IP one VPC (painful but correct).
2. Use **AWS PrivateLink** — exposes specific services, not entire CIDRs, so overlapping IPs are not an issue.
3. Use **NAT on the connecting layer** — translate source/destination IPs at a transit VPC (complex).

---

### Q9-C. TRICKY: What happens if you delete a VPC endpoint while a Lambda inside the VPC is actively using it?

**Answer:**  
In-flight connections will be interrupted (TCP RST). Lambda invocations that were mid-request to, say, S3 via the Gateway Endpoint will fail. New invocations will attempt to route S3 traffic via the NAT Gateway (if one exists) or fail with a connection error (if no internet path exists).

**Gateway vs Interface endpoint behavior:**
- **Gateway Endpoint** (S3, DynamoDB): Routes at the routing table level. Deleting it removes the route prefix list entry. Traffic immediately falls back to the internet route.
- **Interface Endpoint** (PrivateLink): Backed by ENIs. Deleting removes the ENI and DNS override. Existing TCP connections die; DNS resolves back to public IP.

---

### Q9-D. DESIGN: Your company has 50 AWS accounts and needs them all to share internet egress through a single set of NAT Gateways for compliance (all outbound traffic must be inspected and logged centrally). How do you architect this?

**Answer:**

```
┌──────────────────────────────────────────────────────────┐
│                  AWS Organizations                         │
│                                                            │
│  ┌──────────────────────────────────┐                     │
│  │    Shared Services Account        │                     │
│  │                                   │                     │
│  │  Egress VPC                        │                   │
│  │  ├── NAT Gateways (per AZ)         │                   │
│  │  ├── AWS Network Firewall           │                   │
│  │  │   (Inspect + log all egress)     │                   │
│  │  └── Internet Gateway              │                   │
│  │                                    │                   │
│  │  Transit Gateway (shared via       │                   │
│  │  RAM - Resource Access Manager)    │                   │
│  └──────────────────────────────────┘                     │
│           │                                                │
│     ┌─────┴──────┬──────────┬──────────┐                  │
│     │            │          │          │                   │
│  Acct A       Acct B     Acct C    Acct D                 │
│  (VPCs)       (VPCs)     (VPCs)    (VPCs)                 │
│  All have 0.0.0.0/0 → TGW attachment                     │
└──────────────────────────────────────────────────────────┘
```

**Key implementation steps:**
1. Create a **Transit Gateway** in the Shared Services account, share it via **AWS RAM** to all 50 accounts.
2. Each account's VPCs attach to TGW. Their private subnets have `0.0.0.0/0 → TGW`.
3. TGW routes `0.0.0.0/0` to the **Egress VPC** attachment.
4. Egress VPC has: Network Firewall → NAT Gateway → IGW.
5. TGW has **separate route tables** to ensure spoke VPCs cannot communicate with each other (only egress path).
6. VPC Flow Logs on the Egress VPC capture all outbound traffic for compliance.
7. Network Firewall rule groups enforce domain filtering (allow only approved domains).

**Tricky detail:** Blackhole routes in TGW route tables can prevent VPC-to-VPC traffic while allowing VPC-to-egress.

---

### Q9-E. TRICKY: What is the difference between an Internet Gateway and a NAT Gateway at the packet level?

**Answer:**

**Internet Gateway (IGW):**
- Performs **1:1 NAT** between the instance's private IP and its associated **Elastic IP or public IP**.
- The instance must have a public IP assigned.
- Inbound traffic can initiate connections (bidirectional).
- The IGW is stateless at the IP level (state is maintained by Security Groups).
- It does NOT change the source IP for outbound traffic from the instance's perspective — the instance uses its private IP in the packet, and the IGW swaps it for the public IP en route.

**NAT Gateway:**
- Performs **many-to-one NAT (MASQUERADE)** — multiple private IPs map to the NAT Gateway's single Elastic IP using ephemeral port tracking (similar to home router NAT).
- Only outbound-initiated connections are allowed. Inbound connections cannot be established through a NAT Gateway.
- Has a 55,000 simultaneous connection limit per destination IP/port combination.
- Each NAT Gateway supports up to 100 Gbps and auto-scales — but is **AZ-scoped**.
- **Critical gotcha:** A NAT Gateway in AZ-A cannot serve instances in AZ-B without cross-AZ data transfer charges. Always deploy one NAT Gateway per AZ.

---

### Q9-F. DESIGN: Design a hybrid DNS architecture where on-premises applications can resolve AWS private hosted zone records, and AWS workloads can resolve on-premises DNS names.

**Answer:**

```
On-Premises (192.168.0.0/16)              AWS VPC (10.0.0.0/16)
┌──────────────────────────┐              ┌──────────────────────────┐
│  Corporate DNS Server     │              │  Route 53 Resolver        │
│  (e.g., 192.168.1.53)    │              │                           │
│                           │              │  Inbound Endpoint         │
│  Conditional Forwarder:   │◄─DX/VPN─────│  ENI: 10.0.1.5 (AZ-a)    │
│  *.aws.internal → R53     │              │  ENI: 10.0.2.5 (AZ-b)    │
│  Inbound EP               │              │                           │
│                           │              │  Outbound Endpoint        │
│  Handles:                 │              │  ENI: 10.0.1.6 (AZ-a)    │
│  *.corp.internal natively │◄─────────────│  ENI: 10.0.2.6 (AZ-b)    │
│                           │              │                           │
└──────────────────────────┘              │  Resolver Rules:           │
                                          │  *.corp.internal →         │
                                          │  Forward to 192.168.1.53   │
                                          │                           │
                                          │  Route 53 Private Zones:   │
                                          │  *.aws.internal            │
                                          │  db.aws.internal → RDS EP  │
                                          └──────────────────────────┘
```

**Flow for on-prem → AWS DNS:**
1. On-prem app queries `db.aws.internal`.
2. Corporate DNS has a **conditional forwarder**: `aws.internal → 10.0.1.5` (R53 Inbound Endpoint).
3. Query traverses DX/VPN into VPC, hits the Inbound Endpoint ENI.
4. Route 53 Resolver resolves against Private Hosted Zone `aws.internal`.
5. Returns the private IP of the RDS endpoint.

**Flow for AWS → on-prem DNS:**
1. Lambda/EC2 in VPC queries `app.corp.internal`.
2. VPC DNS (`10.0.0.2`) checks R53 Resolver rules.
3. Outbound Endpoint forwards query to `192.168.1.53` over DX/VPN.
4. Corporate DNS resolves and returns answer.

**Security:** Outbound endpoint rules can be scoped to specific domains. Inbound endpoints require Security Group allowing UDP/TCP 53 from on-premises CIDR.

---

### Q9-G. TRICKY: Explain AWS Network Firewall vs Security Groups vs NACLs vs WAF — when does each apply?

**Answer:**

| | Security Group | NACL | Network Firewall | WAF |
|---|---|---|---|---|
| Layer | L4 (transport) | L4 (transport) | L3–L7 | L7 (HTTP/S) |
| Scope | ENI (instance) | Subnet | VPC/TGW | CloudFront/ALB/APIGW |
| Stateful | Yes | No | Yes (flow-based) | Yes |
| Protocol | TCP/UDP/ICMP | TCP/UDP/ICMP | TCP/UDP/ICMP + L7 | HTTP/HTTPS |
| L7 inspection | No | No | Yes (IDS/IPS rules, domain filtering) | Yes (OWASP, regex) |
| Direction | Inbound/Outbound | Inbound/Outbound | North-South + East-West | North-South only |
| Managed rules | No | No | Suricata-compatible | AWS managed |
| Use case | Instance isolation | Subnet-level deny | Centralized inspection, IPS, domain filtering | Web app protection |

**The answer they want:** These are complementary layers, not alternatives. A well-secured architecture uses ALL of them:
- **WAF** at the edge (CloudFront/ALB) for HTTP threats.
- **Network Firewall** at the VPC boundary for deep packet inspection.
- **NACLs** for stateless subnet-level rules (block known bad CIDR ranges).
- **Security Groups** for micro-segmentation at the instance level.

---

### Q9-H. What is AWS Global Accelerator and how does it differ from CloudFront? When would you pick one over the other?

**Answer:**

| | CloudFront | Global Accelerator |
|---|---|---|
| Primary purpose | Content caching + delivery | Network path optimization |
| Caches content | Yes (HTTP objects, static) | No |
| Protocol | HTTP/HTTPS | TCP, UDP, HTTP/HTTPS |
| IP addresses | Dynamic (domain-based) | Static anycast IPs (2 per accelerator) |
| Health checks | Origin-based | Endpoint health checks |
| Failover | Cache TTL dependent | Near-instant (<30s) |
| Use case | Static/dynamic web, CDN | Gaming, VoIP, IoT, APIs needing sticky IPs |

**Key insight:** Global Accelerator excels when:
1. You need **static IPs** (whitelisted by corporate firewalls).
2. You have **non-HTTP workloads** (UDP for gaming, custom TCP).
3. You need **instant regional failover** without DNS TTL delays.
4. Your app is **not cacheable** but users are geographically distributed and TCP connection establishment latency is the problem.

**CloudFront excels when:** You have cacheable content, need Lambda@Edge, need WAF integration at the CDN layer.

---

## 3. Compute — EC2, ECS, EKS, Lambda — Basics

### Q9. Explain the AWS Nitro System. What problems does it solve?

**Answer:**  
Nitro is AWS's purpose-built hypervisor and hardware offload system that replaced the older Xen-based hypervisor.

**Components:**
- **Nitro Hypervisor**: Lightweight hypervisor focused purely on CPU and memory virtualization. Moves I/O offload to dedicated hardware.
- **Nitro Cards**: Hardware cards that handle VPC networking (Nitro ENI), EBS I/O (Nitro Storage), and security monitoring.
- **Nitro Security Chip**: Verifies hardware components and firmware integrity.
- **Nitro Enclaves**: Isolated compute environments for sensitive data processing (no persistent storage, no external networking, no SSH).

**Problems solved:**
1. **Performance**: By offloading networking and storage to dedicated hardware, guest VMs get near bare-metal performance. C5/M5/R5 instances can deliver 25 Gbps networking and 19 Gbps EBS throughput.
2. **Security**: Reduced attack surface — hypervisor has no operator access. Nitro enclaves provide cryptographic attestation.
3. **Overhead**: Traditional hypervisors consumed 20-30% of host resources. Nitro reclaims this for customer workloads.
4. **Bare metal**: `*.metal` instances run directly on the Nitro hardware, no hypervisor at all.

---

### Q10. What are EC2 placement groups and when do you use each type?

**Answer:**

| Type | What it does | Use case | Trade-off |
|---|---|---|---|
| **Cluster** | Packs instances on same rack/physical host | HPC, low-latency (10 Gbps enhanced networking) | Single AZ only, higher failure blast radius |
| **Spread** | Places instances on distinct hardware (max 7 per AZ) | Critical instances requiring isolation | Limited to 7 per AZ |
| **Partition** | Divides into partitions (separate racks), up to 7 partitions/AZ | Hadoop, Kafka, Cassandra | Partition failure = multiple instances affected |

**Cluster group internals:** Instances are placed within the same network fabric zone, enabling sub-millisecond latency. Uses enhanced networking (SR-IOV / ENA) for up to 100 Gbps on latest instance types.

---

### Q11. ECS vs EKS — when do you choose one over the other? Explain Fargate in both contexts.

**Answer:**

**ECS (Elastic Container Service):**
- AWS-native, simpler operational model.
- Task definitions, services, clusters are AWS-specific abstractions.
- Deep integration with ALB, IAM (task roles), CloudWatch, Service Discovery.
- No Kubernetes API — less portable but lower operational overhead.
- Good for teams that want containers without Kubernetes complexity.

**EKS (Elastic Kubernetes Service):**
- Managed Kubernetes control plane.
- Portable: same workloads can run on-prem or other clouds.
- Richer ecosystem: Helm, service meshes (Istio/Linkerd), Argo CD.
- More complex: you manage node groups (or Fargate profiles), RBAC, networking (VPC CNI).
- Good for large engineering orgs with Kubernetes expertise or multi-cloud strategies.

**Fargate:**
- Serverless compute layer for both ECS and EKS — no EC2 instances to manage.
- On ECS: You define a task with CPU/memory, Fargate provisions micro-VMs (Nitro-isolated).
- On EKS: Fargate profiles map pod selectors to Fargate. Each pod runs in its own micro-VM.
- Trade-off: No GPU support, no privileged containers (EKS Fargate), higher per-unit cost vs EC2, cold start latency.

**Choose ECS** when: AWS-only shop, simpler operations, small-medium container workloads.  
**Choose EKS** when: Multi-cloud, large scale, existing K8s tooling, advanced scheduling needs.

---

### Q12. Explain Lambda cold starts — what causes them and how do you mitigate them?

**Answer:**

**What happens during a cold start:**
1. Lambda service provisions a new execution environment (micro-VM on Firecracker).
2. Downloads and extracts the deployment package.
3. Starts the language runtime (JVM, Node.js, Python interpreter).
4. Runs your initialization code (outside the handler).
5. Finally calls your handler.

This adds 100ms–10+ seconds depending on runtime and package size.

**Cold start duration by runtime (approximate):**
- Python/Node.js: 100–500ms
- Java (with JVM): 1–10s
- .NET: 500ms–2s
- Go: 50–200ms

**Mitigation strategies:**

1. **Provisioned Concurrency**: Pre-warms N execution environments. They stay initialized, eliminating cold starts for those invocations. Costs money even when idle.

2. **Lambda SnapStart** (Java only): Takes a snapshot of the initialized execution environment. New instances restore from snapshot rather than initializing fresh. Reduces Java cold start from ~8s to ~200ms.

3. **Minimize package size**: Smaller zip = faster extraction. Use Lambda layers for shared deps.

4. **Choose lighter runtimes**: Python/Node.js over Java for latency-sensitive functions.

5. **Keep functions warm**: Scheduled EventBridge rule pings functions every 5 minutes. Hacky but works for low-concurrency use cases.

6. **VPC-attached Lambdas**: Previously added 10s cold starts due to ENI provisioning. With Hyperplane ENI (2019+), this is now sub-second.

---

### Q13. How does Lambda scale? What are reserved vs provisioned concurrency?

**Answer:**

**Scaling behavior:**
- Each concurrent Lambda invocation runs in a separate execution environment.
- Default account limit: **1,000 concurrent executions per region** (can be raised).
- Burst scaling: 3,000 initial burst, then +500/minute until limit (varies by region).
- No concurrency = throttling → `429 TooManyRequestsException`.

**Reserved Concurrency:**
- Sets a **maximum** number of concurrent executions for a function.
- Guarantees that this function can always reach its limit (reserves from account pool).
- Also acts as a **throttle ceiling** — useful for protecting downstream systems.
- Free to configure.

**Provisioned Concurrency:**
- Pre-initializes execution environments.
- Eliminates cold starts for predictable traffic.
- Costs money (per GB-second of provisioned concurrency + per invocation).
- Can be auto-scaled using Application Auto Scaling.

```
Account Limit: 1000
├── Function A: Reserved = 300 (max 300 concurrent)
├── Function B: Reserved = 200 (max 200 concurrent)
└── Unreserved pool: 500 (shared by all other functions)

Function A also has Provisioned = 50
└── 50 environments always warm, no cold start for first 50 concurrent requests
```

---

---

## 4. Compute — Advanced & Auto Scaling

### Q13-A. Explain EC2 Auto Scaling — what are the different scaling policy types?

**Answer:**

**Target Tracking Scaling (recommended for most cases):**
- You specify a target value for a metric (e.g., CPU = 50%).
- ASG automatically calculates and adjusts capacity to maintain the target.
- Example: ALBRequestCountPerTarget = 1000 → ASG keeps adding/removing instances to maintain 1000 req/instance.
- Cooldown: default 300s after scale-out, 300s after scale-in.

**Step Scaling:**
- Define steps: if CPU > 50% add 2, if CPU > 80% add 4, if CPU > 90% add 8.
- More granular than target tracking.
- Triggered by CloudWatch Alarms.
- Allows scaling while a previous scaling action is in progress (no cooldown between steps).

**Simple Scaling:**
- Legacy. Wait for cooldown, then evaluate one step.
- One CloudWatch alarm → one scaling action. Superseded by step scaling.

**Scheduled Scaling:**
- Set desired/min/max at specific times.
- Use for predictable patterns: "every weekday at 8am, scale to 20 instances."

**Predictive Scaling:**
- Uses ML to forecast load based on historical CloudWatch metrics.
- Proactively adds capacity before the demand hits.
- Works alongside reactive policies.

```
Scale-Out Decision Flow:
CloudWatch Alarm (CPU > 70%)
    │
    ▼
ASG evaluates step policy
    ├── 70-80%: Add 2 instances
    ├── 80-90%: Add 4 instances  
    └── >90%:   Add 6 instances
    │
    ▼
Launch instances (from Launch Template)
    │
    ├── Instance warm-up period (avoid premature scale-in)
    └── Register with ALB Target Group
```

---

### Q13-B. TRICKY: What is the difference between a Launch Configuration and a Launch Template? Why should you use Launch Templates?

**Answer:**  
Launch Configurations are the legacy way. Launch Templates are the modern replacement and support:

| Feature | Launch Configuration | Launch Template |
|---|---|---|
| Versioning | No | Yes (multiple versions, default version) |
| Mixed instance types | No | Yes (multiple instance types in one ASG) |
| Spot + On-Demand mix | No | Yes |
| T2/T3 unlimited | No | Yes |
| Dedicated hosts | No | Yes |
| Capacity Reservations | No | Yes |
| Instance metadata v2 | Limited | Full support |
| EBS-optimized, Placement Groups | Basic | Advanced |

**The critical gotcha:** Launch Configurations cannot be modified after creation — you create a new one. Launch Templates support versioning. Always use Launch Templates.

**Mixed instances policy** with Launch Templates allows ASG to use multiple instance types and purchase options in a single ASG — critical for Spot Instance resilience (if one instance type is unavailable, use another).

---

### Q13-C. What is EC2 Instance Metadata Service (IMDS)? What is IMDSv2 and why is it more secure?

**Answer:**

**IMDS (v1):**
- HTTP endpoint at `169.254.169.254` accessible from any process on the EC2 instance.
- Returns instance metadata: instance ID, region, IAM role credentials, public IP, etc.
- No authentication — any process (including malicious code via SSRF vulnerabilities) can query it.

**SSRF Attack:** A web app with Server-Side Request Forgery vulnerability could be exploited to make HTTP requests to `169.254.169.254/latest/meta-data/iam/security-credentials/MyRole` and steal the IAM role credentials.

**IMDSv2 (session-oriented):**
1. First, obtain a session token via PUT request with TTL: `PUT http://169.254.169.254/latest/api/token` with `X-aws-ec2-metadata-token-ttl-seconds: 21600`.
2. Use token in subsequent GET requests: `GET http://169.254.169.254/latest/meta-data/` with `X-aws-ec2-metadata-token: <token>`.

**Why it blocks SSRF:** SSRF typically only sends GET requests. The required PUT to get a token uses a different HTTP verb — most SSRF vulnerabilities only allow GET/POST. Also, the PUT has a `X-Forwarded-For` check — if a proxy adds this header, the request is rejected (preventing SSRF through a proxy).

**Enforcement:** Set `HttpTokens: required` in Launch Template or via instance modification. Use AWS Config rule `ec2-imdsv2-check` to detect non-compliant instances.

---

### Q13-D. DESIGN: You need to run a large batch ML training job that uses 500 GPU instances for 6 hours. Cost is a priority. Design the compute strategy.

**Answer:**

```
Strategy: EC2 Spot with Capacity-Optimized + Checkpointing

┌─────────────────────────────────────────────┐
│  Step 1: Request Spot Fleet                  │
│  ├── Instance types: p3.8xl, p3.16xl,        │
│  │   p4d.24xl (diverse pool = fewer          │
│  │   interruptions)                          │
│  ├── Allocation: capacity-optimized          │
│  ├── Target: 500 units of GPU compute        │
│  └── Placement Group: Cluster (for EFA/NCCL) │
│                                              │
│  Step 2: Checkpointing                       │
│  ├── Training job writes checkpoint every    │
│  │   30 minutes to S3/EFS                    │
│  ├── On Spot interruption (2-min warning):   │
│  │   EventBridge → Lambda → save checkpoint  │
│  └── Job restarts from last checkpoint       │
│                                              │
│  Step 3: Orchestration                       │
│  ├── AWS Batch (managed) OR                  │
│  └── SageMaker Training Jobs                 │
│      (handles Spot + checkpointing natively) │
└─────────────────────────────────────────────┘
```

**Cost calculation:**
- p3.8xl On-Demand: ~$12.24/hr × 500 × 6h = **$36,720**
- p3.8xl Spot: ~$3–4/hr × 500 × 6h ≈ **$9,000–12,000** (70%+ savings)

**EFA (Elastic Fabric Adapter):** For distributed training across instances, use EFA for low-latency, high-bandwidth inter-node communication. Required for NCCL-based frameworks (PyTorch DDP, TensorFlow Horovod).

**SageMaker Managed Spot Training:** If using SageMaker, enable `UseSpotInstances: true` + `CheckpointConfig`. SageMaker handles all the spot interruption logic automatically.

---

### Q13-E. Explain EC2 Enhanced Networking — SR-IOV and ENA. What problem do they solve?

**Answer:**

**Traditional virtualized networking problem:**
- Network packets must travel: VM → Hypervisor → Physical NIC → Network.
- Hypervisor adds CPU overhead for every packet (software interrupt handling).
- High packet rates (millions PPS) consume significant host CPU.

**SR-IOV (Single Root I/O Virtualization):**
- The physical NIC is divided into multiple Virtual Functions (VFs).
- Each VF is assigned directly to a VM, bypassing the hypervisor for data path.
- The hypervisor retains control of the Physical Function (PF) for management.
- Reduces latency (no hypervisor round-trip), reduces jitter.

**ENA (Elastic Network Adapter):**
- AWS's custom SR-IOV driver for EC2.
- Supports up to 100 Gbps bandwidth on latest instances (C5n, P4d, etc.).
- Lower CPU overhead — NIC handles checksums, segmentation, etc. in hardware.
- Prerequisite: ENA module must be installed in the instance OS.

**EFA (Elastic Fabric Adapter):**
- Goes further than ENA — adds OS-bypass capability.
- Applications can communicate with the NIC directly (bypassing the OS network stack).
- Provides MPI (Message Passing Interface) semantics for HPC/ML workloads.
- Sub-microsecond latency, needed for tightly coupled parallel applications.

---

## 5. Load Balancing — ALB, NLB, GWLB, CLB

### Q14-A. Compare ALB vs NLB vs GWLB — when do you use each?

**Answer:**

| Feature | ALB | NLB | GWLB | CLB |
|---|---|---|---|---|
| OSI Layer | L7 (HTTP/HTTPS/gRPC/WebSocket) | L4 (TCP/UDP/TLS) | L3+L4 (IP) | L4/L7 (legacy) |
| Static IP | No (DNS only) | Yes (1 per AZ) | N/A | No |
| Preserve client IP | Via X-Forwarded-For | Yes (natively) | Yes | No |
| TLS termination | Yes | Yes (pass-through option) | No | Yes |
| Path/host routing | Yes | No | No | No |
| Header-based routing | Yes | No | No | No |
| Target types | Instance, IP, Lambda | Instance, IP, ALB | Instance, IP | Instance |
| Latency | ~400ms | ~100ms (ultra-low) | N/A | ~400ms |
| Use case | Web apps, APIs, microservices | Extreme performance, TCP, static IPs | Inline security appliances | Legacy (avoid) |

**GWLB (Gateway Load Balancer)** is the most misunderstood:
- It creates a **transparent bump in the wire** for sending traffic through third-party security appliances (firewalls, IDS/IPS, deep packet inspection tools).
- Uses **GENEVE protocol** to encapsulate original packets.
- Appliances see the original traffic without the client knowing the route.
- Route traffic through GWLB endpoints in your VPC, inspect, then forward to destination.

```
GWLB Architecture:
Internet → IGW → GWLB Endpoint (in your VPC) → GWLB → 
Firewall Appliance Fleet (inspects) → GWLB → 
GWLB Endpoint → ALB/EC2 Target
```

---

### Q14-B. TRICKY: What is the difference between ALB and NLB when it comes to client IP preservation?

**Answer:**

**ALB:**
- Terminates the TCP connection at the ALB.
- Establishes a new TCP connection to the backend.
- The backend sees the **ALB node's IP** as the source IP, NOT the client's IP.
- Client's original IP is passed in the `X-Forwarded-For` HTTP header.
- For IP-based rate limiting at the backend, you must parse the `X-Forwarded-For` header.
- The `X-Forwarded-Proto` header tells you if the original request was HTTP or HTTPS.

**NLB:**
- **Does NOT terminate the TCP connection** (in TCP pass-through mode).
- The backend instance sees the **actual client IP** as the source IP.
- No HTTP headers needed.
- This is why NLB is used for non-HTTP protocols where headers aren't possible.
- Requires the instance's security group to allow traffic from the client IP ranges (not just the NLB IPs).

**Tricky gotcha:** With ALB, backend Security Groups only need to allow traffic from ALB security group — not from the internet. With NLB (in TCP mode), the backend security group must allow the client CIDR ranges directly (or use a broad allow rule) because NLB doesn't have a security group of its own.

---

### Q14-C. TRICKY: An ALB shows healthy targets but users still get 502 errors. What are the possible causes?

**Answer:**  
This is a classic production troubleshooting scenario. The ALB health check passes but the actual request fails:

1. **Health check path vs actual endpoint:** Health check targets `/health` (returns 200) but the actual endpoint `/api/data` is broken. Different code paths.

2. **Response timeout mismatch:** The ALB's idle timeout (default 60s) is shorter than the backend processing time. Long-running requests get a 502 from ALB (connection reset after timeout).

3. **Backend closes connection prematurely:** The backend sends a TCP FIN before the ALB receives the full response. Node.js/HTTP2 keep-alive settings mismatch.

4. **HTTP version mismatch:** ALB sends HTTP/1.1 but backend only speaks HTTP/1.0.

5. **Response size too large:** ALB has limits; backend returning huge headers.

6. **Lambda target timeout:** Lambda function times out (15 min max but ALB has a 5 minute limit for Lambda integrations). Lambda returns 429 or function error.

7. **TLS backend errors:** ALB to backend HTTPS: certificate mismatch, expired cert on backend.

**Debugging steps:**
- Check ALB access logs (enable if not already) — they show backend status codes.
- Check ALB `HTTP_5XX` metrics broken down by target group.
- Enable target group detailed health check response (check the `ResponseCode` reason).
- Check backend application logs for the request timeline.

---

### Q14-D. DESIGN: You need to run a fleet of third-party firewall appliances in AWS that must inspect ALL traffic entering and leaving your VPCs. Design the architecture.

**Answer:**

```
Internet
  │
  ▼
┌─────────────────────────────────────────────┐
│  Inspection VPC                              │
│                                              │
│  Internet Gateway                            │
│       │                                      │
│  GWLB Endpoint (Ingress)                     │
│       │                                      │
│  Gateway Load Balancer ──────────────────┐   │
│       │                                  │   │
│  Firewall Appliance Fleet                │   │
│  (EC2 Auto Scaling Group)                │   │
│  - Palo Alto / Fortinet / CheckPoint     │   │
│  - Inspects GENEVE-encapsulated traffic  │   │
│       │                                  │   │
│  (traffic returned to GWLB) ◄────────────┘   │
│       │                                      │
│  GWLB Endpoint (Egress to app)               │
└─────────────────────────────────────────────┘
  │
  ▼ (via TGW or VPC route)
┌─────────────────┐
│  App VPC        │
│  ALB → EC2/EKS  │
└─────────────────┘

Route Table in App VPC:
0.0.0.0/0 → TGW
(TGW routes to Inspection VPC)

Route Table in Inspection VPC:
10.0.0.0/8 → GWLB Endpoint (return to apps)
0.0.0.0/0 → IGW (after inspection)
```

**Key GWLB properties:**
- GENEVE encapsulation preserves original 5-tuple (src/dst IP, port, protocol).
- Appliances see the original packets and can make policy decisions.
- GWLB provides high availability — if one appliance fails, GWLB routes to healthy ones.
- Flow stickiness: ensures the same flow always goes to the same appliance (stateful firewall requirement).

---

## 6. Storage — S3, EBS, EFS, FSx — Basics

### Q14. Explain S3 consistency model and how it works internally.

**Answer:**  
As of **December 2020**, S3 provides **strong read-after-write consistency** for all operations (PUT, DELETE) on all object operations — this replaced the previous eventual consistency model.

**Internals:**
- S3 objects are stored across multiple AZs (at least 3 for Standard storage class).
- The S3 API layer uses a **distributed metadata service** that tracks object version state.
- Strong consistency is achieved by ensuring all read requests go through the same consistent metadata path.
- No extra cost, no performance degradation.

**What this means practically:**
- After a successful PUT, any subsequent GET/LIST will return the new object.
- After a DELETE, subsequent GETs return 404 immediately.
- This eliminates the need for "read-your-writes" workarounds that were common pre-2020.

**S3 still does NOT provide:**
- Atomic transactions across multiple objects.
- Conditional writes based on object state (use S3 Object Lock for write-once semantics).

---

### Q15. What is S3 Transfer Acceleration and how does it work?

**Answer:**  
S3 Transfer Acceleration uses **CloudFront edge locations** to accelerate uploads to S3.

**How it works:**
1. You use a special endpoint: `bucket.s3-accelerate.amazonaws.com`.
2. The upload travels over public internet to the **nearest CloudFront edge location** (short distance, optimized TCP stack).
3. From the edge location, the data travels over **AWS's private global backbone** to the S3 bucket's region.
4. The backbone uses optimized routing (better than BGP over public internet).

**When it helps:** Transfers from geographically distant clients (e.g., uploading from Europe to `us-east-1`). Less useful for clients in the same region as the bucket.

**vs Multipart Upload:** These are complementary. Use multipart for large files (>100MB) AND Transfer Acceleration for geographically distant uploads.

---

### Q16. Compare EBS volume types — when would you use gp3 vs io2 Block Express?

**Answer:**

| Feature | gp3 | io2 Block Express |
|---|---|---|
| Max IOPS | 16,000 | 256,000 |
| Max throughput | 1,000 MB/s | 4,000 MB/s |
| Max size | 16 TiB | 64 TiB |
| IOPS:GB ratio | Decoupled | Decoupled |
| Durability | 99.8-99.9% | 99.999% |
| Multi-attach | No | Yes (Nitro, same AZ) |
| Use case | General workloads, cost-optimal | SAP HANA, Oracle, high-perf DBs |

**Key gp3 advantage:** IOPS and throughput are independently configurable (you don't pay for extra IOPS baked into storage size like gp2). 3,000 IOPS and 125 MB/s baseline free, can burst to 16K IOPS.

**EBS Multi-Attach (io2):** Multiple EC2 instances in the same AZ can attach the same volume. Requires applications to manage concurrent writes (e.g., cluster-aware file system like GFS2).

---

### Q17. EFS vs FSx — what are the differences and when to use each?

**Answer:**

| | EFS | FSx for Windows | FSx for Lustre | FSx for NetApp ONTAP |
|---|---|---|---|---|
| Protocol | NFS v4 | SMB/CIFS | Lustre (POSIX) | NFS, SMB, iSCSI |
| OS | Linux | Windows | Linux | Both |
| Performance | Burstable/provisioned | SSD-backed | Sub-ms, GB/s | Flexible |
| Use case | Shared Linux storage | Windows apps, AD | HPC, ML training | Enterprise NAS migration |
| Linking to S3 | No | No | Yes (data repo) | No |

**FSx for Lustre** is unique — it can mount an **S3 bucket as a data repository**. Files are lazily loaded from S3 on first access, and outputs can be written back. Critical for ML training jobs that need high-throughput access to datasets stored in S3.

---

---

## 7. Storage — Advanced & Tricky

### Q17-A. TRICKY: S3 has a 5,500 GET/s per prefix limit. Your application is hitting this and getting 503 SlowDown errors. How do you fix it?

**Answer:**  
S3 automatically scales to high request rates, but the limits are **per prefix** (prefix = everything in the key before the last `/` separator):

- 3,500 PUT/COPY/POST/DELETE requests/second per prefix.
- 5,500 GET/HEAD requests/second per prefix.

**If all objects share the same prefix** (e.g., `images/photo1.jpg`, `images/photo2.jpg`), the `images/` prefix becomes the bottleneck.

**Solutions:**

1. **Use multiple prefixes (randomized key names):**
   - Instead of `images/user123/photo.jpg`, use `a3f/images/user123/photo.jpg` where `a3f` is a hash prefix.
   - With 4 random hex chars as prefix, you get millions of distinct prefixes, each with its own 5,500 req/s limit.

2. **CloudFront in front of S3:**
   - CloudFront caches objects at edge. Only cache misses hit S3.
   - For highly requested objects (popular images, JS/CSS files), CloudFront dramatically reduces S3 request rate.
   - Cost: CloudFront data transfer vs S3 request pricing — often cheaper AND faster.

3. **S3 Intelligent Tiering** doesn't help here — this is a request rate issue, not a storage cost issue.

**Note:** AWS has improved S3 scalability — the old advice to randomize prefixes is less critical now since S3 can partition hot prefixes automatically. But the 5,500/s per-prefix limit still exists theoretically, and randomization is still best practice for extreme-scale workloads.

---

### Q17-B. Explain S3 lifecycle policies — what transitions are allowed and what are the minimum durations?

**Answer:**

**S3 Storage Classes (coldest to hottest):**
```
S3 Standard → S3 Standard-IA → S3 Intelligent-Tiering → S3 One Zone-IA → S3 Glacier Instant Retrieval → S3 Glacier Flexible Retrieval → S3 Glacier Deep Archive
```

**Lifecycle Transitions (allowed directions — only downward to colder):**

| From | To | Min days |
|---|---|---|
| Standard | Standard-IA or One Zone-IA | 30 days |
| Standard | Glacier Instant Retrieval | 0 days |
| Standard | Glacier Flexible Retrieval | 0 days |
| Standard | Glacier Deep Archive | 0 days |
| Standard-IA | One Zone-IA | 30 days |
| Standard-IA | Glacier classes | 0 days |

**Retrieval options:**
- **Glacier Instant Retrieval**: milliseconds (replaces Glacier with S3-like access)
- **Glacier Flexible Retrieval**: Expedited (1-5 min), Standard (3-5 hr), Bulk (5-12 hr)
- **Glacier Deep Archive**: Standard (12 hr), Bulk (48 hr) — cheapest storage, slowest retrieval

**Tricky:** You cannot transition FROM Glacier back to Standard via lifecycle. You must restore a temporary copy, then re-upload to Standard.

**Minimum storage duration charges:**
- Standard-IA: 30 days minimum (charged even if deleted earlier)
- Glacier Flexible: 90 days minimum
- Glacier Deep Archive: 180 days minimum

---

### Q17-C. TRICKY: What is S3 Object Lock and how is it different from bucket versioning?

**Answer:**

**Versioning:** Keeps multiple versions of an object. Objects can still be deleted (just creates a delete marker). Admins with sufficient permissions can permanently delete versions.

**S3 Object Lock:** Implements **WORM (Write Once, Read Many)** storage. Objects cannot be modified or deleted for a specified period — even by root accounts (if in Compliance mode).

**Retention modes:**
- **Governance mode**: Users with `s3:BypassGovernanceRetention` permission can override. Good for internal audit requirements.
- **Compliance mode**: NO one can delete or modify the object during the retention period — not even the root account, not even AWS Support. Designed for regulatory compliance (SEC 17a-4, FINRA, HIPAA).

**Legal Hold:** Independent of retention period. Prevents deletion indefinitely until the hold is removed. Doesn't expire automatically.

**Requirement:** Object Lock requires versioning to be enabled (it stores lock metadata on the object version).

**Use case:** Healthcare records, financial audit logs, SEC compliance. If your compliance auditor says "prove that logs cannot be tampered with," Object Lock in Compliance mode is the answer.

---

### Q17-D. DESIGN: Design a cost-optimized S3 storage strategy for a media company that uploads 10TB of video daily, keeps content hot for 30 days, warm for 1 year, and then archives permanently.

**Answer:**

```
Day 0: Upload to S3 Standard
├── Active production use
├── Frequent reads, low latency needed
└── Cost: ~$0.023/GB/month

Day 30 → Lifecycle transition to S3 Standard-IA
├── Occasional access (re-processing, re-delivery)
├── Cost: ~$0.0125/GB/month + retrieval fee
└── Saves ~45% vs Standard

Day 365 → Lifecycle transition to S3 Glacier Instant Retrieval
├── Rare access but must be available quickly if needed
├── Cost: ~$0.004/GB/month
└── Saves ~83% vs Standard

Day 2+ years → Lifecycle transition to S3 Glacier Deep Archive
├── Permanent archive, compliance copy
├── Cost: ~$0.00099/GB/month
└── Retrieval in 12h — acceptable for archival

Additional optimizations:
├── S3 Intelligent-Tiering for unpredictable access patterns
│   (auto-moves objects based on actual access)
├── S3 Replication (CRR) to secondary region for DR
│   (replicate to Glacier in secondary = cheap DR)
└── S3 Inventory + Storage Lens for cost visibility
```

**Monthly cost for 10TB/day × 30 days = 300TB active:**
- Standard (0-30 days): 300TB × $0.023 = **$6,900/month**
- After 30 days → IA: 300TB × $0.0125 = **$3,750/month**
- Savings: ~$3,150/month just from the first transition.

---

## 8. Databases

### Q18. Aurora vs RDS — what are the architectural differences?

**Answer:**

**Traditional RDS:**
- Single EC2 instance + EBS volume.
- Multi-AZ: synchronous replication to a standby (separate EBS in different AZ). Failover: ~60-120 seconds.
- Read replicas: asynchronous replication; separate storage.

**Aurora Architecture:**
- Separates compute from storage. Storage is a **shared distributed storage layer** spanning 3 AZs (6 copies across 6 storage nodes).
- Write quorum: 4/6 nodes must acknowledge. Read quorum: 3/6.
- Storage auto-scales in 10GB increments up to 128 TiB.
- **No binlog replication between replicas** — all replicas read from the same shared storage. Replication lag is near-zero.
- Failover: ~30 seconds (no storage replication needed, just DNS endpoint switch).

```
Traditional RDS Multi-AZ:          Aurora:
                                    
  Primary (AZ-a)                   Writer Instance (AZ-a)
  └── EBS ──sync──► Standby         Reader Instance (AZ-b)
                    (AZ-b)          Reader Instance (AZ-c)
  Read Replica                            │
  └── EBS (async replication)      Shared Storage Layer
                                   AZ-a: ██████  (2 copies)
                                   AZ-b: ██████  (2 copies)
                                   AZ-c: ██████  (2 copies)
```

**Aurora Global Database:**
- One primary region, up to 5 secondary regions.
- Replication at the storage layer (not application), typically **< 1 second lag**.
- Secondary can be promoted to primary in < 1 minute for DR.
- Supports **write forwarding**: write to secondary, Aurora forwards to primary.

---

### Q19. DynamoDB internals — how does partitioning work and what causes hot partitions?

**Answer:**

**Partitioning:**
- DynamoDB distributes data across partitions based on the **partition key hash**.
- Each partition holds up to **10 GB of data** and can serve up to **3,000 RCU and 1,000 WCU**.
- When a table grows beyond these limits, DynamoDB splits partitions automatically.

**Hash function:** DynamoDB uses an internal consistent hashing mechanism. You don't control which physical partition a key maps to.

**Hot partitions:**  
If your partition key has low cardinality (e.g., `status = "active"`) or a few keys receive disproportionate traffic, those partitions get overloaded → `ProvisionedThroughputExceededException`.

**Solutions:**
1. **High-cardinality keys**: Use UUIDs, user IDs, composite keys.
2. **Write sharding**: Append a random suffix (1–N) to the partition key. Query all shards and aggregate.
3. **Adaptive Capacity**: DynamoDB automatically redistributes capacity to hot partitions within a table (but this is a reactive measure, not a fix).
4. **DAX (DynamoDB Accelerator)**: In-memory cache in front of DynamoDB. Reduces read load from hot partitions. Sub-millisecond latency.

**Global Secondary Indexes (GSI):**
- GSIs are separate partitioned tables maintained asynchronously.
- GSI writes consume WCU from the base table.
- If a GSI has a hot partition, it throttles writes to the base table (GSI backpressure).

---

### Q20. Explain DynamoDB consistency models and when to use each.

**Answer:**

**Eventually Consistent Reads (default):**
- Reads may return stale data (up to ~1 second old).
- Uses 1 RCU per 4 KB.
- Reads from any storage node (may not have the latest write yet).

**Strongly Consistent Reads:**
- Always returns the most up-to-date data.
- Uses 2 RCU per 4 KB (double the cost).
- Reads from the leader storage node.
- Not available on GSIs.

**Transactional Reads/Writes:**
- ACID transactions across multiple items/tables.
- `TransactGetItems` / `TransactWriteItems`.
- Uses 2x RCU/WCU (reads are strongly consistent, writes go through a 2-phase commit).
- Cannot span more than 100 items or 4 MB.

**When to use strong consistency:** Financial balances, inventory counts, any scenario where stale reads would cause incorrect behavior.

---

### Q21. How does ElastiCache Redis clustering work? Compare cluster mode enabled vs disabled.

**Answer:**

**Cluster Mode Disabled (Replication Group):**
- One primary node, up to 5 read replicas.
- All nodes contain the full dataset.
- Single keyspace — all data lives together.
- Simple failover: if primary fails, a replica is promoted.
- Scales reads horizontally but not writes.

**Cluster Mode Enabled:**
- Data is **sharded** across multiple node groups (shards).
- Each shard has one primary + up to 5 replicas.
- Up to 500 node groups = 500x write/storage scalability.
- Clients use **Redis Cluster protocol** (MOVED/ASK redirects) or use a cluster-aware client.
- Cannot use multi-key operations across different shards (unless keys use hash tags to force same shard).

```
Cluster Mode Enabled (3 shards):

Shard 0 (slots 0-5460):      Shard 1 (slots 5461-10922):   Shard 2 (slots 10923-16383):
  Primary ──► Replica          Primary ──► Replica             Primary ──► Replica
```

**Use cluster mode enabled** when: dataset exceeds single node memory, or write throughput exceeds single primary capacity.

---

## 10. Messaging & Streaming

### Q22. SQS vs SNS vs EventBridge — when do you use each?

**Answer:**

| | SQS | SNS | EventBridge |
|---|---|---|---|
| Pattern | Queue (pull) | Pub/Sub (push) | Event Bus (push) |
| Consumers | One consumer per message | Multiple subscribers | Multiple rules/targets |
| Persistence | Up to 14 days | No persistence | No persistence (optional archive) |
| Filtering | No | Attribute-based | Rich event pattern matching |
| Schema | None | None | Schema Registry |
| Targets | Any poller | SQS, Lambda, HTTP, email | 20+ AWS services, custom |
| Ordering | FIFO queues | No | No (EventBridge Pipes can help) |

**Common pattern — Fan-out:**
```
SNS Topic
├── SQS Queue A (order-processing)
├── SQS Queue B (inventory-update)
└── SQS Queue C (audit-log)
```

SNS pushes to all SQS queues simultaneously. Each service processes independently at its own pace.

**EventBridge** is the evolution: use it when you need content-based routing, integration with 3rd-party SaaS (Shopify, Datadog, etc.), or fine-grained filtering without writing Lambda glue code.

**SQS FIFO:** Guarantees exactly-once processing and ordering within a message group. Throughput limited to 3,000 msg/s with batching (vs Standard's nearly unlimited). Use for financial transactions, order management.

---

### Q23. Kinesis Data Streams vs Kafka (MSK) — deep dive comparison.

**Answer:**

**Kinesis Data Streams:**
- Managed by AWS, serverless (on-demand mode) or provisioned shards.
- **Shard**: unit of capacity — 1 MB/s write, 2 MB/s read, 1,000 records/s write.
- Retention: 24h default, up to 365 days.
- Ordering: guaranteed within a shard (partition key determines shard).
- Enhanced Fan-Out: dedicated 2 MB/s read pipe per consumer (vs shared 2 MB/s across all consumers in standard mode).
- Consumer types: GetRecords polling (shared) or SubscribeToShard (enhanced fan-out, push).

**MSK (Managed Streaming for Kafka):**
- Managed Apache Kafka — AWS manages brokers, ZooKeeper/KRaft, OS patches.
- You still choose broker instance types, storage, replication factor.
- Full Kafka API compatibility — Kafka Streams, ksqlDB, Kafka Connect.
- MSK Serverless: auto-scales, no shard management.
- Better for: large ecosystems, on-prem Kafka migration, complex stream processing.

| | Kinesis | MSK |
|---|---|---|
| Operational overhead | Low (fully managed) | Medium (manage brokers) |
| API | AWS SDK | Kafka API |
| Throughput scaling | Add shards | Add brokers/partitions |
| Ecosystem | AWS-native | Kafka ecosystem |
| Consumer groups | Shard iterator / EFO | Consumer groups |
| Replay | Up to 365 days | Configurable retention |

---

### Q24. How do you implement exactly-once processing with SQS?

**Answer:**  
SQS Standard guarantees **at-least-once** delivery. True exactly-once processing requires a combination:

1. **SQS FIFO queues**: Content-based deduplication using SHA-256 hash of message body (or explicit deduplication ID). Within a 5-minute window, duplicate messages are rejected.

2. **Idempotent consumers**: Design your processing logic so receiving the same message twice produces the same result. Use a deduplication store (DynamoDB item with message ID + TTL).

```
Consumer logic:
1. Receive message from SQS
2. Check DynamoDB: has message_id been processed?
   - If YES: delete message from SQS, return
   - If NO: process message
3. Write message_id to DynamoDB (with TTL)
4. Delete message from SQS
```

3. **DynamoDB Transactions**: Combine the processing effect + deduplication record update in a single atomic transaction.

**Visibility timeout:** If processing fails before deleting the message, it becomes visible again after the visibility timeout. Set this to > max processing time to avoid duplicate processing during normal operation.

---

## 12. Security

### Q25. Explain IAM internals — how does AWS evaluate a policy?

**Answer:**

**Policy evaluation logic (in order):**

1. **Explicit Deny** in any policy → **DENY** (overrides everything)
2. **SCP (Service Control Policy)** — if in AWS Organizations, SCP must allow the action
3. **Resource-based policy** — if the principal is in the same account and RBP allows, can be ALLOW without identity policy
4. **IAM permission boundary** — if set, must allow the action
5. **Session policy** — if assuming a role with a session policy, must allow
6. **Identity-based policy (IAM policy)** — must explicitly allow
7. Default: **IMPLICIT DENY**

```
Request comes in
    │
    ▼
Explicit Deny? ──YES──► DENY
    │NO
    ▼
SCP allows? ──NO──► DENY
    │YES
    ▼
RBP allows (same account)? ──YES──► ALLOW (if no deny elsewhere)
    │NO
    ▼
Permission Boundary set? ──YES──► Must allow within boundary
    │NO (or boundary allows)
    ▼
Session Policy set? ──YES──► Must allow within session
    │NO (or session allows)
    ▼
Identity Policy allows? ──YES──► ALLOW
    │NO
    ▼
IMPLICIT DENY
```

**Cross-account access:** Both the identity policy (in account A) AND the resource policy (in account B) must allow the action.

---

### Q26. Explain KMS key hierarchy and envelope encryption.

**Answer:**

**Envelope Encryption:**
Rather than encrypting data directly with the KMS key (CMK), you:
1. Generate a **Data Encryption Key (DEK)** from KMS (`GenerateDataKey` API).
2. Use the DEK (plaintext) to encrypt your data locally (AES-256).
3. Store the **encrypted DEK** alongside your ciphertext.
4. Discard the plaintext DEK.
5. To decrypt: call KMS to decrypt the encrypted DEK → use plaintext DEK to decrypt data.

**Why?**
- KMS has a 4KB payload limit.
- Network round-trip for every encrypt/decrypt is expensive at scale.
- DEK stays in memory only during encryption/decryption.

**KMS Key Hierarchy:**
```
AWS KMS Hardware Security Module (HSM)
└── Customer Master Key (CMK) [never leaves HSM in plaintext]
    └── Encrypts Data Encryption Key (DEK)
        └── Encrypts your actual data
```

**Key types:**
- **AWS Managed Keys** (aws/s3, aws/rds): Created/managed by AWS services, rotated annually, no direct access.
- **Customer Managed Keys (CMK)**: You create and manage, can enable key rotation, set key policies, CloudTrail all usage.
- **AWS CloudHSM**: You own the HSM hardware (single-tenant), FIPS 140-2 Level 3, you manage keys.

---

### Q27. How does AWS WAF work? Explain rules, rule groups, and WebACL.

**Answer:**

**Web ACL:** The top-level construct. Attached to CloudFront, ALB, API Gateway, or AppSync. Evaluates requests against an ordered list of rules.

**Rules:** Each rule has:
- A **match condition** (IP, geo, string match, regex, SQL injection, XSS detection, rate-based).
- An **action**: Allow, Block, Count, CAPTCHA, or Challenge.

**Rule Groups:** Reusable collections of rules. Types:
- **AWS Managed Rule Groups**: Maintained by AWS (Core Rule Set, Known Bad Inputs, Bot Control, Fraud Control).
- **AWS Marketplace Rule Groups**: From security vendors (F5, Imperva).
- **Custom Rule Groups**: Your own.

**Evaluation order:** Rules are evaluated by priority (lowest number first). The first matching rule's action applies. If no rule matches, the default action (Allow/Block) applies.

**Rate-based rules:** Aggregate requests by IP (or custom key) over a 5-minute window. If threshold exceeded → block. Useful for DDoS mitigation and brute-force protection.

**WAF vs Shield:**
- WAF: Layer 7 (HTTP) — protects against web exploits, OWASP Top 10.
- Shield Standard: Layer 3/4 — automatically included for all AWS customers, protects against SYN floods, UDP reflection attacks.
- Shield Advanced: Layer 3/4/7 — DDoS Response Team (DRT) access, cost protection, enhanced detection.

---

## 19. Observability

### Q28. Design a comprehensive observability strategy for a microservices architecture on AWS.

**Answer:**

**Three pillars: Metrics, Logs, Traces**

```
Microservices Architecture Observability:

Application Layer:
  ECS/EKS Services
  ├── App Logs ──────────────► CloudWatch Logs / S3
  ├── Metrics (EMF) ─────────► CloudWatch Metrics
  └── Traces (X-Ray SDK) ────► X-Ray / AWS Distro for OpenTelemetry

Infrastructure Layer:
  ├── EC2/ECS/EKS metrics ───► CloudWatch Container Insights
  ├── RDS Performance Insights► RDS metrics
  └── VPC Flow Logs ─────────► S3 → Athena (or CW Logs)

Alerting:
  CloudWatch Alarms ──────────► SNS → PagerDuty / Slack
  CloudWatch Composite Alarms ► Reduces alert noise

Dashboards:
  CloudWatch Dashboards (operational)
  Grafana (rich visualization, uses CloudWatch as datasource)

Security Observability:
  CloudTrail ─────────────────► S3 + CloudWatch Logs
  GuardDuty ──────────────────► Security Hub → EventBridge → Lambda
  Config Rules ───────────────► Compliance dashboard
```

**Key recommendations:**
- Use **Structured logging (JSON)** so CloudWatch Insights can query fields.
- Use **CloudWatch Embedded Metric Format (EMF)** to emit custom metrics from Lambda/ECS without extra API calls.
- Use **X-Ray Groups and Sampling Rules** to trace 100% of error requests but sample normal traffic.
- Implement **Synthetic Monitoring** (CloudWatch Synthetics) for proactive endpoint monitoring.
- Use **CloudWatch Contributor Insights** to identify top contributors to high-latency or error patterns (e.g., which user ID is generating most 5xx errors).

---

## 20. High Availability & Disaster Recovery

### Q29. Explain the four DR strategies — what are the RTO/RPO trade-offs?

**Answer:**

```
                    Cost & Complexity
Low ◄─────────────────────────────────────── High
│                                              │
Backup &    Pilot     Warm        Multi-Site
Restore     Light     Standby     Active/Active
│                                              │
High ────────────────────────────────────► Low
                    RTO/RPO
```

| Strategy | RTO | RPO | Cost | Description |
|---|---|---|---|---|
| **Backup & Restore** | Hours | Hours | $ | Backups in S3, restore to new infra on DR event |
| **Pilot Light** | 10-60 min | Minutes | $$ | Core infra always on (DB replication), compute off |
| **Warm Standby** | Minutes | Seconds | $$$ | Scaled-down running copy, scale up on failover |
| **Multi-Site Active/Active** | Near-zero | Near-zero | $$$$ | Full production in both regions, traffic split |

**Pilot Light in practice:**
- RDS Aurora Global Database replicating to DR region.
- AMIs/container images ready in DR region.
- Route 53 health checks monitor primary.
- On failover: promote Aurora Global secondary, launch EC2/ECS in DR region, update Route 53.

**Active/Active in practice:**
- Route 53 latency-based or weighted routing between regions.
- DynamoDB Global Tables (multi-region, multi-master writes).
- Aurora Global Database with write forwarding.
- Requires conflict resolution strategy for concurrent writes.

---

### Q30. How does Route 53 health checking and failover work? What are the limits?

**Answer:**

**Health check types:**
- **Endpoint checks**: HTTP/HTTPS/TCP checks against IP or domain. Checks every 10s or 30s from multiple Route 53 health checker locations (15+ globally).
- **CloudWatch Alarm checks**: Fails if a CW alarm is in ALARM state. Enables checking internal metrics (DB latency, queue depth) that can't be accessed externally.
- **Calculated checks**: Combine multiple health checks with AND/OR/NOT logic.

**Failover routing:**
- Primary record + Secondary record, both with the same name.
- If primary health check fails, Route 53 stops returning primary IP → returns secondary.
- With short TTLs (60s), clients see failover within ~60 seconds.

**Important limits:**
- Health checkers are in `us-east-1`, `us-west-2`, `eu-west-1`, etc. — they check from outside your VPC.
- For private resources (no public IP), you must use CloudWatch Alarm-based health checks.
- DNS TTL determines actual failover time — Route 53 changes the DNS response, but cached responses persist until TTL expires.

---

### Q31. Design a multi-region active-active architecture for a global e-commerce platform.

**Answer:**

```
                         Route 53 (Latency-based routing)
                              │                │
                    ┌─────────┘                └─────────┐
                    │                                     │
              us-east-1                            eu-west-1
              ┌──────────────┐                  ┌──────────────┐
              │  CloudFront  │                  │  CloudFront  │
              └──────┬───────┘                  └──────┬───────┘
                     │                                  │
              ┌──────▼───────┐                  ┌──────▼───────┐
              │  WAF + ALB   │                  │  WAF + ALB   │
              └──────┬───────┘                  └──────┬───────┘
                     │                                  │
              ┌──────▼───────┐                  ┌──────▼───────┐
              │  EKS Cluster │                  │  EKS Cluster │
              │  (Microsvcs) │                  │  (Microsvcs) │
              └──────┬───────┘                  └──────┬───────┘
                     │                                  │
              ┌──────▼───────┐    ←─ replication ─►   ┌──────▼───────┐
              │ DynamoDB GT  │                          │ DynamoDB GT  │
              │ Aurora Global│                          │ Aurora Global│
              │   (Primary)  │                          │  (Secondary) │
              └──────────────┘                          └──────────────┘
                     │                                        │
              ┌──────▼────────────────────────────────────────▼───┐
              │                  S3 CRR (Cross-Region Replication)  │
              └────────────────────────────────────────────────────┘
```

**Key components:**
- **DynamoDB Global Tables**: Multi-master writes in both regions. Last-writer-wins conflict resolution (or custom Lambda-based).
- **Aurora Global Database**: Primary writes in us-east-1, replicated to eu-west-1 (<1s lag). Read traffic served locally. Write forwarding enabled for local writes in EU.
- **S3 Cross-Region Replication (CRR)**: Product images, static assets replicated.
- **ElastiCache Global Datastore**: Redis replication across regions for session data.
- **Route 53 Latency Routing**: Routes users to lowest-latency region. Health checks trigger failover.
- **CloudFront with Origin Groups**: Primary + failover origin for CDN resilience.

---

## 21. Cost Optimization

### Q32. What are the different EC2 purchasing options and when do you use each?

**Answer:**

| Option | Discount vs On-Demand | Commitment | Best For |
|---|---|---|---|
| **On-Demand** | Baseline | None | Dev/test, unpredictable, short-term |
| **Reserved (1yr)** | ~40% | 1 year | Steady-state production workloads |
| **Reserved (3yr)** | ~60% | 3 years | Long-running, predictable workloads |
| **Savings Plans (Compute)** | Up to 66% | 1 or 3 yr | Flexible — applies across EC2, Fargate, Lambda |
| **Savings Plans (EC2)** | Up to 72% | 1 or 3 yr | Specific instance family, any size/OS |
| **Spot** | Up to 90% | None (can be interrupted) | Batch, ML training, stateless microservices |
| **Dedicated Host** | Variable | 1 or 3 yr | License compliance (BYOL), regulatory |

**Spot best practices:**
- Use **Spot Instance Advisor** to find low-interruption-rate instance types.
- Use **EC2 Auto Scaling with mixed instance policy**: combine On-Demand base + Spot instances.
- Handle **interruption notice** (2-minute warning via instance metadata + EventBridge) — checkpoint work, drain gracefully.
- Use **capacity-optimized allocation strategy** (not cheapest first) to reduce interruptions.

**Savings Plans > Reserved Instances** for most customers because Compute Savings Plans apply automatically across instance families, regions, OS, and even Fargate/Lambda.

---

## 22. Migration & Modernization

### Q33. Explain the 7 Rs of cloud migration. Give examples of when to apply each.

**Answer:**

| Strategy | Description | Example |
|---|---|---|
| **Retire** | Decommission unused applications | Legacy reporting tool no one uses |
| **Retain** | Keep on-premises (for now) | Mainframe app requiring full rewrite |
| **Rehost** (Lift & Shift) | Move to cloud as-is | VM → EC2, no code changes |
| **Relocate** | Move container platforms | VMware on-prem → VMware Cloud on AWS |
| **Replatform** (Lift & Reshape) | Minor cloud optimizations | MySQL on EC2 → RDS MySQL |
| **Repurchase** | Move to SaaS | Custom CRM → Salesforce |
| **Refactor** (Re-architect) | Redesign for cloud-native | Monolith → microservices on EKS |

**AWS Migration Services:**
- **AWS Application Migration Service (MGN)**: Continuous block-level replication from source to AWS. Cutover when ready. Minimal downtime lift-and-shift.
- **AWS Database Migration Service (DMS)**: Heterogeneous DB migration (Oracle → Aurora). Full load + CDC for minimal downtime.
- **AWS Schema Conversion Tool (SCT)**: Converts DB schema + stored procedures from Oracle/SQL Server to Aurora/PostgreSQL.

---

## 25. System Design Questions (End-to-End)

### Q34. Design a serverless data processing pipeline that ingests 1 million events/second from IoT devices.

**Answer:**

```
IoT Devices (1M events/sec)
         │
         ▼
   AWS IoT Core
   (MQTT broker, device registry, rules engine)
         │
    IoT Rule Actions
    ├──► Kinesis Data Streams  (real-time processing)
    └──► S3 (raw data archive)
         │
         ▼
Kinesis Data Streams
(100 shards × 10K records/s = 1M records/s)
         │
    ┌────┴─────────────────────────────┐
    │                                  │
    ▼                                  ▼
Kinesis Data           Lambda (Enhanced Fan-Out)
Analytics (Flink)      ├── Alert processing
- Windowed aggregations └── Real-time notifications → SNS
- Anomaly detection
    │
    ▼
Kinesis Data Firehose
├── S3 (Parquet, partitioned) → Glue Catalog → Athena
└── OpenSearch (dashboards, search)

Batch Analytics:
S3 raw data → Glue ETL Jobs → S3 processed → Redshift Spectrum
                                             └── QuickSight
```

**Scale math:**
- 1M events/sec at 1KB each = 1 GB/s ingestion.
- Kinesis: 1 shard = 1 MB/s write. Need 1,000 shards OR use Kinesis On-Demand (auto-scales).
- IoT Core → Kinesis via IoT Rule (basic ingest rule, no compute needed).

**Handling backpressure:**
- Kinesis buffers data for up to 7 days (with extended retention).
- Lambda consumers scale automatically per-shard.
- Kinesis Data Firehose buffers and batches writes to S3 (1-minute or 1MB threshold).

---

### Q35. Design a multi-tenant SaaS platform on AWS. How do you handle tenant isolation?

**Answer:**

**Tenant isolation models:**

```
Silo Model (max isolation):       Bridge Model:             Pool Model (max efficiency):
One AWS account per tenant        One VPC per tenant        Shared everything
├── Separate VPC                  ├── Shared account        ├── Row-level isolation in DB
├── Separate DB                   ├── Separate DB schema    ├── Tenant ID in every table
└── Hard billing boundary         └── Shared compute        └── App-level isolation only
High cost, high isolation         Medium                    Low cost, lower isolation
```

**Recommended: Bridge Model with resource-based isolation:**

```
┌─────────────────────── Management Account ───────────────────────┐
│  Control Plane (shared):                                          │
│  ├── Tenant Registry (DynamoDB)                                   │
│  ├── Identity Service (Cognito User Pools per tenant)             │
│  └── Billing/Metering (Usage Plans on API Gateway)               │
└───────────────────────────────────────────────────────────────────┘

┌─────────────────────── Tenant Accounts ──────────────────────────┐
│  Tenant A Account          Tenant B Account                       │
│  ├── VPC                   ├── VPC                                │
│  ├── Aurora Serverless     ├── Aurora Serverless                  │
│  └── EKS Namespace         └── EKS Namespace                     │
└───────────────────────────────────────────────────────────────────┘
```

**Key isolation mechanisms:**
- **IAM Role per tenant**: Lambda/ECS assumes tenant-specific role. AWS resource policies enforce boundaries.
- **DynamoDB partition key = tenant_id**: Row-level isolation enforced at app layer. Add explicit condition on every query.
- **Cognito User Pool per tenant** OR custom claims in JWT with tenant_id claim.
- **API Gateway Usage Plans**: Rate limits and quotas per tenant API key.
- **S3 bucket + prefix per tenant** with bucket policy denying cross-tenant access.
- **AWS Organizations + SCPs**: If silo model, each tenant in own account, SCPs prevent data exfiltration.

---

### Q36. Design a real-time fraud detection system processing 50,000 transactions per second.

**Answer:**

```
Payment Systems → API Gateway (WebSocket / REST)
                       │
                       ▼
              Kinesis Data Streams
              (50 shards × 1K TPS)
                       │
          ┌────────────┴──────────────┐
          │                           │
          ▼                           ▼
   Lambda (pre-check)         Kinesis Analytics
   - Block list check         (Flink, sliding windows)
   - Velocity rules           - Tx count per card/min
   - Card BIN validation      - Geographic anomalies
          │                   - Amount deviation
          ▼                           │
   SageMaker Endpoint                 │
   (Real-time ML model)               │
   - Feature store (low-latency)      │
   - <10ms inference                  │
          │                           │
          └────────────┬──────────────┘
                       │
                       ▼
                  Decision Engine
                  ├── Score < threshold: APPROVE → SQS → payment processing
                  ├── Score > high threshold: DECLINE → response
                  └── Middle range: STEP-UP (MFA challenge)
                       │
                       ▼
                 Results stored in
                 DynamoDB (real-time status)
                 + S3/Redshift (analytics)
                       │
                       ▼
                 Feedback Loop:
                 Labeled outcomes → SageMaker
                 training pipeline (model retraining)
```

**SageMaker Feature Store:**  
- Online store (low-latency reads, ~1ms) for real-time inference.
- Offline store (S3, Parquet) for batch training.
- Features: transaction history, device fingerprint, behavioral patterns.

**Latency budget (50ms total):**
- Kinesis ingestion: ~5ms
- Pre-check Lambda: ~5ms
- Feature retrieval (Feature Store online): ~1ms
- SageMaker inference: ~10ms
- Decision + response: ~5ms
- Total: ~26ms (well within budget)

---

### Q37. Design a global content delivery system for a video streaming platform (Netflix-scale).

**Answer:**

```
Content Ingest:
  Studios → S3 Upload (Transfer Acceleration)
  → AWS Elemental MediaConvert (transcoding to HLS/DASH)
  → Multiple renditions (240p, 480p, 720p, 1080p, 4K)
  → S3 Origin Bucket (per region)

Delivery Architecture:
                    CloudFront Distribution
                    ├── Edge Caches (400+ PoPs globally)
                    │   └── Cache HLS segments (2-10s chunks)
                    ├── Origin Shield (regional cache layer)
                    │   └── Reduces origin load by 95%+
                    └── S3 Origin (or custom origin)

                    Signed URLs/Cookies for content protection
                    ├── Lambda@Edge validates JWT token
                    └── CloudFront enforces signed URL expiry

CDN Cache Strategy:
  ├── Manifest files (.m3u8): Short TTL (30s) — updated for live
  ├── Video segments (.ts): Long TTL (24h+) — immutable
  └── Static assets: Long TTL (1 year) with cache-busting

Adaptive Bitrate:
  Player ──► manifest ──► segment requests
  Player measures bandwidth → switches renditions
  CloudFront serves from cache regardless of rendition

Backend:
  ├── Catalog/Metadata: DynamoDB (global tables) + ElastiCache
  ├── User Sessions: ElastiCache Redis (Global Datastore)
  ├── Recommendations: SageMaker real-time endpoints
  └── Analytics: Kinesis → Redshift → QuickSight
```

---

### Q38. How would you design a highly available, secure API platform for a financial services company?

**Answer:**

```
External Clients
      │
      ▼
Route 53 (health-check based failover, latency routing)
      │
      ▼
CloudFront + AWS Shield Advanced + WAF
(DDoS protection, OWASP rules, Bot Control, TLS 1.2+ enforcement)
      │
      ▼
API Gateway (Regional, Multi-Region)
├── Throttling (per client, per method)
├── API Keys + Usage Plans (rate limiting)
├── Cognito Authorizer or Lambda Authorizer (JWT/OAuth2)
├── Request validation (schema enforcement)
└── Private integration (NLB or VPC Link → EKS/ECS)
      │
      ▼
Internal Microservices (Private VPC)
├── Service Mesh (AWS App Mesh or Istio) — mTLS between services
├── EKS (with IRSA — IAM Roles for Service Accounts)
├── No outbound internet access (NAT only for controlled egress)
└── PrivateLink for AWS service access (no public endpoints)
      │
      ▼
Data Layer
├── Aurora PostgreSQL (Multi-AZ, encrypted, IAM auth)
├── ElastiCache (encryption in-transit + at-rest)
├── KMS CMKs for all data encryption (automatic rotation)
└── Secrets Manager (credential rotation, no hardcoded secrets)

Compliance & Audit:
├── CloudTrail (all API calls, integrity validation enabled)
├── AWS Config (compliance rules, drift detection)
├── Security Hub (aggregated findings)
├── Macie (S3 PII detection)
└── GuardDuty (threat detection, enabled in all accounts)
```

---

## 27. Behavioral / Architecture Decision Questions

### Q39. How do you approach choosing between SQS, Kinesis, and MSK for a new workload?

**Answer (decision framework):**

```
Start here:
Is the data stream ordered and does ordering matter?
├── No ordering needed → SQS Standard (simplest, cheapest)
├── Ordering within a group needed + exactly-once → SQS FIFO
└── Ordered streams with multiple consumers:
    │
    Is throughput > 10K msg/sec?
    ├── No → Kinesis (simple, managed, no ops)
    └── Yes:
        │
        Do you need Kafka ecosystem (Kafka Streams, Connect, ksqlDB)?
        ├── Yes → MSK
        └── No → Kinesis (On-Demand for auto-scaling)

Additional factors:
- Need to replay old data? → Kinesis or MSK (SQS deletes after consumption)
- 3rd party Kafka producers/consumers already built? → MSK
- Want serverless + no ops? → Kinesis On-Demand
- Throughput in GB/s range? → MSK (more cost-effective at extreme scale)
```

---

### Q40. How do you handle secrets rotation without downtime in a production system?

**Answer:**

**AWS Secrets Manager rotation:**
1. Secrets Manager calls a **rotation Lambda** (you provide the logic).
2. Lambda creates a **new version** of the secret with `AWSPENDING` staging label.
3. Lambda updates the credential at the database (adds new user or updates password).
4. Lambda tests the new credential.
5. Secrets Manager moves `AWSCURRENT` label to new version, `AWSPREVIOUS` to old.

**Zero-downtime pattern:**
```
Application always fetches secret via Secrets Manager SDK
├── SDK caches secret in memory (TTL: 5 min default)
├── On rotation: AWSCURRENT is updated
├── New connections use new credential
└── Existing connections use old credential until they reconnect
    (DB continues to accept old password during overlap window)
```

**Multi-user rotation strategy (RDS):**
- Maintain two DB users (user_a, user_b).
- Alternate which user is "active" each rotation.
- During rotation: update passive user's password, test, flip active.
- Both credentials valid during transition → no connection drops.

**Best practices:**
- Never hardcode secrets; inject via environment variable that reads from Secrets Manager.
- Use **Secrets Manager VPC endpoint** so rotation Lambda doesn't need internet.
- Enable **automatic rotation** (every 30 days for compliance).
- Use **IAM condition keys** on KMS policy to restrict which services can decrypt which secrets.

---

### Bonus: Architecture Anti-Patterns to Discuss

**Q: What are common AWS architecture mistakes you've seen or would flag in a review?**

1. **Single-AZ deployments**: Any RDS, ElastiCache, or NAT Gateway in a single AZ is a single point of failure.

2. **Over-relying on large EC2 instances**: Vertical scaling has limits. Prefer horizontal scaling with smaller instances + Auto Scaling Groups.

3. **Not using VPC Endpoints**: Applications accessing S3/DynamoDB over the internet incur NAT Gateway costs and expose traffic to the internet when PrivateLink/Gateway Endpoints are free.

4. **Synchronous chains in Lambda**: Lambda calling Lambda synchronously creates tightly coupled, hard-to-debug chains. Use SQS/SNS between Lambda functions.

5. **IAM wildcard permissions**: `"Action": "*", "Resource": "*"` in IAM policies is a security disaster. Always use least-privilege.

6. **Ignoring CloudTrail in non-primary regions**: By default, CloudTrail must be enabled per region. Multi-region trail should be configured.

7. **DynamoDB scan operations in production**: Full table scans consume all provisioned RCU. Design access patterns to use Query with proper key design.

8. **Not enabling Delete Protection on critical resources**: RDS, DynamoDB, ALBs can be accidentally deleted without this.

9. **Treating Availability Zones as equivalent to fault domains**: AZs are isolated but if your application doesn't handle AZ failures (health checks, cross-AZ routing), you still have AZ-level SPOFs.

10. **CloudFront origin not protected**: If your ALB origin is public and not restricted to CloudFront IPs, attackers can bypass WAF by hitting the ALB directly. Use CloudFront managed prefix list on the ALB security group.

---

## 28. Quick Reference: Key Limits

| Service | Limit | Notes |
|---|---|---|
| VPC | 5 per region (default) | Raise via support |
| Subnets | 200 per VPC | |
| Security Groups | 2,500 per VPC; 60 rules per SG | |
| Lambda | 1,000 concurrent (default), 10 GB memory, 15 min timeout | |
| SQS message | 256 KB max | Use S3 + pointer for larger |
| SNS message | 256 KB max | |
| Kinesis shard | 1 MB/s write, 2 MB/s read | |
| DynamoDB item | 400 KB max | |
| DynamoDB partition | 3,000 RCU + 1,000 WCU + 10 GB | |
| S3 object | 5 TB max, 5 GB single PUT | Use multipart for > 100 MB |
| API Gateway | 29s integration timeout | Hard limit |
| ALB rules | 100 per listener | |
| Route 53 health checks | 200 per account (default) | |
| DX bandwidth | 1, 10, 100 Gbps | |

---

*Good luck in your interview! Focus on the "why" behind architectural decisions — trade-offs, not just "what service to use."*
