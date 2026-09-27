# AWS Interview Prep — Part 2: Compute, Auto Scaling, Load Balancing & Storage
> All levels: Basic → Advanced → Tricky Gotchas → Design Questions

---

## COMPUTE — EC2, ECS, EKS, Lambda

### Q1. Explain the AWS Nitro System. What problems does it solve?

Nitro is AWS's purpose-built hypervisor and hardware offload system replacing the older Xen hypervisor.

**Components:**
- **Nitro Hypervisor:** Lightweight, focuses only on CPU/memory virtualization. All I/O is offloaded to dedicated hardware.
- **Nitro Cards:** Dedicated hardware for VPC networking (Nitro ENI), EBS I/O (Nitro Storage), and monitoring.
- **Nitro Security Chip:** Continuously verifies hardware and firmware integrity. No AWS operator can access customer instances.
- **Nitro Enclaves:** Isolated compute with no persistent storage, no external networking, no SSH. For sensitive data processing (PII, private keys). Uses cryptographic attestation to prove enclave identity.

**Problems solved:**
1. **Performance:** Offloading I/O to hardware means guest VMs get near-bare-metal performance. C5 instances: 25 Gbps networking, 19 Gbps EBS.
2. **Security:** Hypervisor has no operator access path. Reduced attack surface.
3. **Overhead:** Old Xen hypervisor consumed 20-30% of host resources. Nitro reclaims this for customers.
4. **Bare metal:** `*.metal` instances run directly on Nitro hardware — no hypervisor layer at all.

---

### Q2. EC2 Placement Groups — types, use cases, trade-offs?

| Type | What it does | Use case | Limit / Trade-off |
|---|---|---|---|
| **Cluster** | Same rack/network fabric | HPC, low-latency inter-node, up to 100 Gbps | Single AZ only, higher blast radius |
| **Spread** | Each instance on distinct hardware | Critical instances that must survive hardware failure | Max 7 instances per AZ |
| **Partition** | Groups of instances on separate racks (partitions), up to 7 per AZ | Distributed systems: Kafka, Hadoop, Cassandra | Partition failure affects multiple instances in that partition |

**Cluster placement internals:** All instances share the same network fabric zone. Combined with ENA/EFA, achieves sub-millisecond latency and 100 Gbps bandwidth between instances.

**Spread vs Partition:** Spread = max isolation (one instance per rack). Partition = group isolation (a Kafka broker rack = a partition, so no two brokers in same partition share hardware).

---

### Q3. ECS vs EKS vs Fargate — when do you choose each?

**ECS:**
- AWS-native container orchestration. No Kubernetes.
- Simple mental model: Task Definitions, Services, Clusters.
- Deep native integrations: ALB, IAM task roles, CloudWatch, Service Discovery, AppMesh.
- Lower operational overhead. Good for AWS-only shops or teams that don't need Kubernetes.

**EKS:**
- Managed Kubernetes control plane (AWS manages etcd, API server, upgrades).
- Full Kubernetes API — portable workloads, Helm, Argo CD, Istio, Karpenter.
- More complex: you manage node groups/Fargate profiles, VPC CNI networking, RBAC.
- Good for: multi-cloud strategy, large org with K8s expertise, complex scheduling needs.

**Fargate (serverless layer for both):**
- No EC2 instances to manage. You define CPU/memory, AWS provisions isolated micro-VMs.
- On ECS: task-level isolation. On EKS: each pod runs in its own micro-VM (Fargate profile).
- Trade-offs: no GPU, no privileged containers (EKS Fargate), ~10-15% higher cost than equivalent EC2, cold start latency.

**Decision:**
- ECS + Fargate: Simplest, lowest ops overhead, AWS-native.
- EKS + EC2: Maximum control, K8s ecosystem, large scale.
- EKS + Fargate: K8s portability without node management (limits apply).

---

### Q4. Lambda cold starts — causes, runtimes, mitigations?

**What happens during a cold start:**
1. Lambda provisions a new Firecracker micro-VM.
2. Downloads and extracts your deployment package.
3. Starts the language runtime (JVM, Node.js, Python interpreter).
4. Executes your initialization code (outside handler).
5. Finally calls your handler function.

**Duration by runtime:**
| Runtime | Typical cold start |
|---|---|
| Go | 50–200ms |
| Python | 100–400ms |
| Node.js | 100–500ms |
| .NET | 500ms–2s |
| Java (JVM) | 1–10s |

**Mitigations:**
1. **Provisioned Concurrency:** Pre-initializes N execution environments. They stay warm, eliminating cold starts for those N slots. Costs money even when idle. Auto-scalable via Application Auto Scaling.
2. **Lambda SnapStart (Java):** Snapshots the initialized JVM state. New environments restore from snapshot — reduces Java cold start from ~8s to ~200ms.
3. **Minimize package size:** Smaller zip = faster extraction. Use Lambda Layers for shared dependencies.
4. **Lighter runtimes:** Python/Node/Go over Java for latency-sensitive functions.
5. **VPC cold start:** Previously ~10s for ENI provisioning. Now sub-second with Hyperplane ENI (shared ENI model, 2019+).

---

### Q5. How does Lambda scale? Reserved vs Provisioned Concurrency?

**Scaling:**
- Each concurrent invocation = separate execution environment.
- Default limit: **1,000 concurrent executions per account per region** (soft limit, raiseable).
- Burst limit: up to 3,000 initial burst, then +500/minute (varies by region).
- Throttled invocations receive `429 TooManyRequests`.
- Synchronous callers (API Gateway) get 429. Async callers (SQS, EventBridge) retry automatically.

**Reserved Concurrency:**
- Sets the **maximum** concurrent executions for one function.
- Also **guarantees** capacity from the account pool (reserved capacity can't be stolen by other functions).
- Use as a throttle to protect downstream systems (e.g., "this function never uses more than 50 concurrent to not overwhelm the DB").
- Free to configure.

**Provisioned Concurrency:**
- Pre-initializes environments — eliminates cold starts.
- Costs money: charged per GB-second of provisioned + per invocation.
- Can be put on a specific function version or alias.
- Auto-scale with Application Auto Scaling based on schedule or metrics.

```
Account pool: 1000
├── Function A: Reserved = 300 (max 300, guaranteed 300 from pool)
├── Function B: Reserved = 200 (max 200)
└── Unreserved pool: 500 (shared by all other functions)

Function A also: Provisioned = 50
└── 50 environments always initialized → no cold start for first 50 concurrent
```

---

### Q6. TRICKY — What is IMDSv2 and why is it more secure than IMDSv1?

**IMDSv1 (insecure):**
- Any process on the instance can do: `GET http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole`
- No authentication. An **SSRF (Server-Side Request Forgery)** vulnerability in a web app can steal IAM role credentials.

**IMDSv2 (session-oriented):**
1. First, PUT request to get a token: `PUT http://169.254.169.254/latest/api/token` with TTL header.
2. Use token in all subsequent GETs: `X-aws-ec2-metadata-token: TOKEN`.

**Why SSRF can't exploit IMDSv2:**
- Most SSRF vulnerabilities only allow GET requests. The required PUT blocks basic SSRF.
- If request goes through a proxy, `X-Forwarded-For` is added → IMDSv2 rejects it.
- TTL limits token lifetime.

**Enforcement:** Set `HttpTokens: required` in Launch Template. Use AWS Config rule `ec2-imdsv2-check` to detect non-compliant instances. IMDSv2 should be mandatory for all new workloads.

---

### Q7. EC2 Auto Scaling — scaling policy types?

**Target Tracking (recommended):**
- Pick a metric target (CPU = 50%, ALB requests/target = 1000).
- ASG auto-calculates capacity adjustments to maintain the target.
- Simplest to configure, handles scale-in and scale-out.

**Step Scaling:**
- Define steps: CPU 60-70% → add 1, 70-80% → add 2, >80% → add 4.
- More granular control.
- Can scale while a previous scaling is in progress.

**Scheduled Scaling:**
- Pre-defined capacity at specific times.
- "Every weekday at 8am, set min=20. At 8pm, set min=5."
- Combine with target tracking for predictable + reactive behavior.

**Predictive Scaling:**
- ML-based forecasting using CloudWatch historical data.
- Proactively adds capacity BEFORE the load hits.
- Works as a layer on top of reactive policies.

**Instance Warm-Up:** Period after launch before a new instance is included in target tracking metrics. Prevents premature scale-in of newly launched instances that haven't fully loaded.

**Cooldown:** Default 300s after scale activity. Prevents rapid thrashing. Step scaling can bypass cooldown for urgent scale-outs.

---

### Q8. TRICKY — Launch Configuration vs Launch Template?

Launch Configurations are **legacy and deprecated for new features**. Always use Launch Templates.

| Feature | Launch Config | Launch Template |
|---|---|---|
| Versioning | No | Yes — multiple versions, set default |
| Mixed instance types | No | Yes — use multiple types in one ASG |
| Spot + On-Demand mix | No | Yes — mixed instances policy |
| T3 Unlimited CPU credits | No | Yes |
| Capacity Reservations | No | Yes |
| IMDSv2 enforcement | Limited | Full |
| Modify after create | No (create new) | Yes (new version) |

**Mixed instances policy** is the key feature: define a base of On-Demand instances + a pool of Spot instances using multiple instance types. If one Spot pool is unavailable, ASG uses another — critical for fault-tolerant Spot usage.

---

### Q9. DESIGN — 500 GPU instances for 6-hour ML training job. Cost is priority.

```
Strategy: EC2 Spot Fleet + Checkpointing + SageMaker

Option A (AWS Batch / custom):
  Spot Fleet Request:
  ├── Instance types: p3.8xlarge, p3.16xlarge, p4d.24xlarge
  │   (diversified pool reduces interruption risk)
  ├── Allocation: capacity-optimized (not lowest-price)
  │   → picks the pool with most available capacity = fewer interruptions
  ├── Placement Group: Cluster (required for EFA/NCCL distributed training)
  └── Target: 500 GPU units

  Checkpointing:
  ├── Write checkpoint to S3 / EFS every 30 minutes
  ├── EventBridge catches Spot interruption notice (2-min warning)
  ├── Lambda saves state, marks last checkpoint
  └── Job restarts from last checkpoint on new Spot instances

Option B (SageMaker - easiest):
  SageMaker Training Job:
  ├── UseSpotInstances: true
  ├── MaxWaitTimeInSeconds: 86400
  └── CheckpointConfig: S3Uri (SageMaker handles everything automatically)
```

**Cost comparison (p3.8xlarge):**
- On-Demand: ~$12.24/hr × 500 × 6h = **$36,720**
- Spot (~70% discount): ~$3.67/hr × 500 × 6h ≈ **$11,010**
- **Savings: ~$25,700 for one training run.**

---

### Q10. Explain EC2 Enhanced Networking — ENA vs EFA?

**Problem with traditional networking:** Every packet traverses hypervisor (software interrupt). At high PPS (packets per second), hypervisor becomes a bottleneck.

**SR-IOV (Single Root I/O Virtualization):**
- Physical NIC presents multiple Virtual Functions (VFs).
- Each VF is mapped directly to a VM, bypassing hypervisor for data path.
- Reduces CPU overhead, reduces latency, reduces jitter.

**ENA (Elastic Network Adapter):**
- AWS's SR-IOV implementation.
- Supports up to 100 Gbps on latest instances (c5n, p4d).
- Hardware offloads: checksums, segmentation, scatter/gather.

**EFA (Elastic Fabric Adapter):**
- Goes further: OS-bypass via libfabric. Applications communicate with NIC directly (kernel network stack bypassed entirely).
- MPI (Message Passing Interface) semantics — required for HPC/ML distributed training.
- Sub-microsecond latency for inter-node communication.
- Required for NCCL (NVIDIA Collective Communications Library) used by PyTorch DDP, TensorFlow Horovod.

---

## LOAD BALANCING — ALB, NLB, GWLB

### Q11. Compare ALB vs NLB vs GWLB?

| Feature | ALB | NLB | GWLB |
|---|---|---|---|
| OSI Layer | L7 (HTTP/HTTPS/gRPC/WebSocket) | L4 (TCP/UDP/TLS) | L3+L4 (GENEVE encapsulation) |
| Static IP | No | Yes (1 per AZ, Elastic IP) | N/A |
| Client IP to backend | X-Forwarded-For header | Direct (native) | Original packet preserved |
| TLS termination | Yes | Yes (or pass-through) | No |
| Content-based routing | Yes (path, header, host, query) | No | No |
| Target types | Instance, IP, Lambda, ALB | Instance, IP, ALB | Instance, IP |
| Latency | ~400ms overhead | ~100ms | N/A |
| Use case | Web apps, APIs, microservices | Extreme perf, static IP, non-HTTP | Security appliance inspection |

**GWLB** is the most misunderstood. It creates a transparent bump-in-the-wire for third-party security appliances (firewalls, IDS/IPS). Uses **GENEVE protocol** to encapsulate original packets. Appliance sees full original traffic, inspects, returns. Source/destination unchanged from application's perspective.

---

### Q12. TRICKY — ALB healthy targets but users get 502 errors. What are the causes?

A 502 from ALB means ALB got an **invalid response or no response** from the backend.

**Causes (memorize all of these):**
1. **Response timeout:** Backend takes longer than ALB's idle timeout (default 60s). Long DB queries, slow APIs. Fix: increase `Idle timeout` on ALB or optimize backend.
2. **Health check passes but real path fails:** Health check hits `/ping` (returns 200) but real endpoint `/api/data` throws a 500. Different code paths.
3. **Backend closes connection prematurely:** Node.js HTTP keep-alive mismatch. Backend sends FIN before ALB receives full response.
4. **HTTP version mismatch:** ALB sends HTTP/1.1, backend only speaks HTTP/1.0.
5. **Lambda target errors:** Function throws an unhandled error, returns a non-200 to ALB. ALB converts this to 502.
6. **Backend TLS cert expired:** ALB → backend HTTPS with invalid cert.
7. **Target deregistration delay:** Instance in draining state, connections being dropped.

**Debugging:** Enable ALB Access Logs → check `backend_status_code` and `target_status_code`. Check `HTTPCode_Target_5XX_Count` CloudWatch metric.

---

### Q13. TRICKY — NLB vs ALB for client IP preservation?

**ALB:**
- Terminates TCP, establishes NEW TCP connection to backend.
- Backend sees **ALB node's private IP** as source.
- Real client IP in `X-Forwarded-For` HTTP header.
- Backend SG only needs to allow ALB SG (not internet CIDRs).

**NLB:**
- In TCP pass-through mode: does NOT terminate TCP.
- Backend sees **actual client IP** directly as source IP.
- No HTTP headers needed — works for non-HTTP protocols.
- Backend SG must allow the **actual client CIDR ranges** (NLB has no security group).
- In TLS termination mode: NLB terminates TLS, NLB's IP is source to backend.

**The common gotcha:** Teams put an NLB in front of EC2, then wonder why EC2 security group blocking internet traffic breaks things. NLB doesn't have its own SG — it passes client IPs through, so you must allow client IPs in the EC2 security group.

---

### Q14. DESIGN — Inline security appliance fleet to inspect all VPC traffic?

```
Internet
  │
  ▼
Inspection VPC
  ┌─────────────────────────────────────────┐
  │  Internet Gateway                        │
  │       ↑↓                                │
  │  [GWLB Endpoint - ingress]              │
  │       ↑↓ (GENEVE encapsulated)          │
  │  Gateway Load Balancer                  │
  │       ↑↓                                │
  │  Firewall Fleet (Auto Scaling Group)    │
  │  - Palo Alto / Fortinet / CheckPoint    │
  │  - Each appliance: receive GENEVE,      │
  │    inspect original packet, return      │
  │       ↑↓ (after inspection)             │
  │  [GWLB Endpoint - egress]               │
  └─────────────────────────────────────────┘
  │
  ▼ (via TGW route)
App VPC → ALB → ECS/EKS

Routing:
App VPC subnet route: 0.0.0.0/0 → TGW
Inspection VPC: IGW route table → GWLB Endpoint
                Return route → GWLB Endpoint → TGW
```

**GWLB key properties:**
- **GENEVE encapsulation:** Preserves original 5-tuple. Appliance sees src/dst without modification.
- **Flow stickiness:** Same flow always goes to same appliance (required for stateful firewalls).
- **HA:** GWLB health-checks appliances, reroutes around failures.
- **Horizontal scale:** Add more appliances to the ASG behind GWLB — scales inspection throughput.

---

## STORAGE — S3, EBS, EFS, FSx

### Q15. S3 consistency model — what changed in 2020?

Before December 2020: Eventually consistent for overwrite PUTs and DELETEs.
After December 2020: **Strong read-after-write consistency for ALL operations.**

**What this means:**
- After a successful PUT → any subsequent GET/LIST immediately returns the new object.
- After a DELETE → subsequent GET returns 404 immediately.
- No extra cost, no performance penalty.

**How it works:** S3's distributed metadata service was updated to use a consistent read path for all requests.

**What S3 still does NOT provide:**
- Atomic multi-object transactions.
- Conditional writes (CAS — compare-and-swap). Use S3 Object Lock for immutability.

---

### Q16. TRICKY — S3 prefix throughput limits and how to avoid throttling?

S3 automatically scales per prefix:
- **5,500 GET/HEAD requests/second per prefix.**
- **3,500 PUT/COPY/POST/DELETE requests/second per prefix.**

A "prefix" is everything before the final object name (the path up to the last `/`).

**Problem:** All objects under `logs/2024/` share that prefix — you hit 5,500 GET/s for that prefix.

**Solutions:**
1. **Randomize key prefixes:** `a3f9/logs/2024/file.txt`, `b1c2/logs/2024/file.txt`. Random hash prefix = millions of distinct prefixes = effectively unlimited throughput.
2. **CloudFront in front of S3:** Caches objects at edge. Popular objects served from edge cache — S3 only sees cache misses.
3. **S3 on-demand auto-partitioning:** AWS now automatically partitions hot prefixes. Still randomize for extreme scale.

---

### Q17. Compare EBS volume types — gp2 vs gp3 vs io2 Block Express?

| | gp2 | gp3 | io2 Block Express |
|---|---|---|---|
| IOPS | 3 IOPS/GB, max 16K | Up to 16K (independent) | Up to 256K |
| Throughput | Up to 250 MB/s | Up to 1,000 MB/s | Up to 4,000 MB/s |
| IOPS coupled to size? | Yes (bigger disk = more IOPS) | No (decouple IOPS from size) | No |
| Durability | 99.8-99.9% | 99.8-99.9% | 99.999% |
| Multi-attach | No | No | Yes (Nitro instances, same AZ) |
| Max size | 16 TiB | 16 TiB | 64 TiB |
| Use case | General workloads | Cost-optimized general | SAP HANA, Oracle RAC, high-perf DBs |

**gp2 gotcha:** IOPS is coupled to size. To get 16,000 IOPS on gp2, you need a 5,333 GB disk — even if you only need 100 GB of space. With gp3, set 100 GB storage + 16,000 IOPS separately. Huge cost saving.

**io2 Multi-Attach:** Multiple EC2 instances (same AZ) can attach the same io2 volume simultaneously. Application must handle concurrent writes (cluster-aware filesystem like GFS2 or application-level coordination).

---

### Q18. EFS vs FSx family — which do you use when?

| | EFS | FSx for Windows | FSx for Lustre | FSx for NetApp ONTAP | FSx for OpenZFS |
|---|---|---|---|---|---|
| Protocol | NFS v4 | SMB/CIFS, DFS | Lustre (POSIX) | NFS, SMB, iSCSI, NVMe/TCP | NFS, SMB |
| OS | Linux | Windows (AD-integrated) | Linux | Linux + Windows | Linux + Windows |
| Performance mode | Burstable / Provisioned | SSD, up to 12 GB/s | Sub-ms, hundreds of GB/s | Flexible tiers | High IOPS |
| S3 integration | No | No | Yes (data repository) | No | No |
| Use case | Shared Linux apps, CMS, ECS volumes | Windows apps, Active Directory | HPC, ML training, genomics | Enterprise NAS lift-and-shift | ZFS workloads |

**FSx for Lustre + S3:** Files are lazily loaded from S3 on first access. Writes can be exported back to S3. This is the key pattern for ML training — massive S3 datasets accessed at Lustre speeds without copying everything up front.

---

### Q19. TRICKY — S3 Object Lock vs Versioning?

**Versioning:** Keeps all versions of every object. A DELETE creates a delete marker. Previous versions accessible. Admin with permissions can permanently delete versions. Not immutable.

**S3 Object Lock (WORM):**
- **Governance mode:** Object protected from deletion/modification. Users with `s3:BypassGovernanceRetention` permission can override. Good for internal retention policies.
- **Compliance mode:** Absolutely no one can delete or modify the object during the retention period — not root account, not AWS Support, not AWS itself. Meets SEC 17a-4, CFTC, FINRA requirements.
- **Legal Hold:** Independent of retention. Object locked indefinitely until the hold is explicitly removed. No expiry.
- Requires versioning to be enabled.

**Use case:** Healthcare records, financial audit trails, compliance-mandated immutable logs.

---

### Q20. S3 Transfer Acceleration vs Multipart Upload — explain both and when to combine.

**S3 Transfer Acceleration:**
- Uses CloudFront edge locations as fast entry points.
- Client uploads to `bucket.s3-accelerate.amazonaws.com` → nearest edge PoP.
- Data travels CloudFront edge → AWS backbone → S3 bucket region.
- AWS backbone is faster and more reliable than public internet BGP routing.
- When it helps: Large files uploaded from geographically distant clients.
- When it doesn't help: Client in same region as bucket (no edge acceleration benefit).

**Multipart Upload:**
- For objects > 100MB (required for > 5GB).
- Split object into parts (5MB–5GB each), upload in parallel.
- Benefits: parallel upload (faster), resume failed parts (only retry failed part).
- Each part is a separate PUT — if network drops, only the failed part retries.
- After all parts uploaded, send `CompleteMultipartUpload` to assemble.

**Combine both:** Upload a 50GB video from Australia to us-east-1:
- Use Multipart Upload (split into 100 × 500MB parts).
- Use Transfer Acceleration endpoint.
- Parts upload in parallel from AU to nearest CloudFront PoP, travel backbone to us-east-1.
- **Result:** Much faster than sequential direct upload.

---

### Q21. DESIGN — Cost-optimized S3 storage for 10TB/day video uploads, hot 30 days, warm 1 year, archive forever.

```
Day 0: S3 Standard
  Cost: $0.023/GB/month
  Access: frequent, low latency

Day 30 → Lifecycle: S3 Standard-IA
  Cost: $0.0125/GB/month (~46% cheaper)
  Min duration charge: 30 days
  Retrieval fee: $0.01/GB

Day 365 → Lifecycle: S3 Glacier Instant Retrieval
  Cost: $0.004/GB/month (~83% cheaper than Standard)
  Retrieval: milliseconds (same as S3 Standard access pattern)
  Min duration: 90 days

Day 730+ → Lifecycle: S3 Glacier Deep Archive
  Cost: $0.00099/GB/month (cheapest storage in AWS)
  Retrieval: 12 hours Standard, 48 hours Bulk
  Min duration: 180 days
  Used for: permanent compliance archive

Additional:
├── S3 Intelligent-Tiering on content with unpredictable access
│   (auto-moves between Standard, Standard-IA, Archive tiers)
├── S3 Inventory weekly report for audit/compliance
├── S3 Storage Lens for cross-account cost visibility
└── CRR (Cross-Region Replication) to secondary region for DR
    Replicate to Glacier in secondary = cheap DR copy
```

**Monthly cost for 300TB active storage:**
- All Standard: 300TB × $0.023 = $6,900
- After lifecycle optimization: ~$1,200–2,000/month
- **Annual savings: ~$57,000–69,600**

---

### Q22. TRICKY — What is S3 Replication and what are its limitations?

**Types:**
- **CRR (Cross-Region Replication):** Source and destination in different regions. Compliance, latency reduction, aggregation.
- **SRR (Same-Region Replication):** Same region, different buckets. Log aggregation, dev/test copies, data sovereignty.

**Requirements:**
- Versioning must be enabled on BOTH source and destination buckets.
- IAM role with permissions to read source and write destination.

**What is NOT replicated by default:**
- Objects that existed before replication was enabled (use S3 Batch Replication for existing objects).
- Delete markers (configurable — can enable replication of delete markers).
- Objects in Glacier/Glacier Deep Archive (must restore first).
- Objects replicated from another rule (no chaining — A→B→C doesn't auto-replicate A to C).
- Objects encrypted with customer-provided keys (SSE-C).

**Replication Time Control (RTC):** Guarantees 99.99% of objects replicated within 15 minutes. Comes with replication metrics. Use for compliance with replication time SLAs.
