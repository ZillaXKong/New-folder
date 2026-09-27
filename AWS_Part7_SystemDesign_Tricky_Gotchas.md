# AWS Interview Prep — Part 7: System Design, Tricky Gotchas & Cost Optimization

> Covers: 10 Full System Designs, 40+ Tricky Gotcha Questions, Cost Optimization Deep Dive, AI/ML on AWS, Advanced HA/DR Patterns

---

## Section 1: System Design Questions

### Q1. DESIGN: Design a URL shortener service (like bit.ly) on AWS — must handle 100K writes/sec and 1M reads/sec globally.

**Answer:**

```
Write Path (100K/sec):
Client → Route 53 (latency routing) → CloudFront → ALB → Lambda (URL generator)
                                                              │
                                              Generate short code (base62, 7 chars)
                                              ← 62^7 = 3.5 trillion unique codes
                                                              │
                                              DynamoDB (write: shortCode → longUrl)
                                              ├── Partition key: shortCode (high cardinality)
                                              ├── TTL attribute (optional expiry)
                                              └── Global Tables (multi-region writes)

Read Path (1M/sec, must be fast):
Client → Route 53 → CloudFront (cache redirect responses!)
├── Cache HIT (TTL 24h): CloudFront returns 301 redirect directly
│   └── 99%+ of reads served from edge cache (saves cost + latency)
└── Cache MISS: CloudFront → Lambda → DynamoDB → 301 redirect
                                └── Response cached at CloudFront edge

Global architecture:
├── DynamoDB Global Tables (us-east-1 + eu-west-1 + ap-southeast-1)
├── CloudFront with 400+ edge locations caches popular URLs
├── Lambda@Edge for custom logic (link expiry check, analytics)
└── Route 53 latency routing → nearest ALB region

Analytics (non-critical path):
Lambda → Kinesis Data Firehose → S3 → Athena
(click tracking, geographic data, referrer info)

Scale math:
├── 1M reads/sec, CloudFront hit ratio 99% → 10K reads/sec hit DynamoDB
├── DynamoDB: On-Demand mode handles burst easily
├── Lambda: ~10K concurrent (well within limits with provisioned concurrency)
└── Storage: 100K writes/sec × 100 bytes × 86400s × 365 = ~315 TB/year
             → S3 archival for old URLs (lifecycle to Glacier after 1 year)
```

---

### Q2. DESIGN: Design a ride-sharing backend (Uber/Lyft clone) on AWS — real-time driver location, matching, surge pricing.

**Answer:**

```
Driver App                          Rider App
    │                                   │
    ▼ (location update every 4s)        ▼ (request ride)
API Gateway WebSocket               API Gateway REST
    │                                   │
    ▼                                   │
Location Service (Lambda)               │
├── Write driver location →             │
│   ElastiCache Redis (Geospatial)      │
│   GEOADD drivers:locations lat lng    │
│   driver_id                           │
│   (auto-expires stale locations)      │
│                                       │
│                                Matching Service (Lambda)
│                                ├── GEORADIUS query: drivers within 2km
│                                ├── Filter: available status
│                                ├── Score: ETA estimate (OSRM or HERE API)
│                                └── Select best match
│                                       │
└─────────────────────────────────────►│
                                       ▼
                                 Trip Service
                                 ├── Create trip record → DynamoDB
                                 ├── Notify driver → WebSocket push
                                 ├── Notify rider → WebSocket push
                                 └── Start trip tracking

Surge Pricing Engine:
├── Kinesis Data Streams (supply/demand events)
├── Kinesis Analytics: compute supply/demand ratio per geo-hex (H3 index)
├── Lambda: update surge multiplier → ElastiCache
└── Pricing Lambda reads multiplier in real-time

Trip tracking:
├── Driver sends location updates → SNS → Lambda → WebSocket push to rider
└── Rider sees live driver position

Data persistence:
├── DynamoDB: trips, drivers, riders (real-time operational data)
├── Aurora: complex relational queries (billing, reports)
├── Redshift: analytics (earnings, patterns, route optimization)
└── S3: raw event archive → ML training for ETA models

ML components:
├── SageMaker: ETA prediction model (real-time endpoint)
├── SageMaker: Surge pricing prediction
└── Feature Store: historical trip data features
```

---

### Q3. DESIGN: Design a distributed rate limiter for an API that handles 500K requests/second across multiple regions.

**Answer:**

**Problem:** Rate limiting at 500K req/s needs to be fast (<5ms overhead), globally consistent, and resilient.

**Approach: Token Bucket algorithm with ElastiCache Redis**

```
Global Rate Limiting Architecture:

Region: us-east-1               Region: eu-west-1
┌─────────────────┐             ┌─────────────────┐
│ API Gateway     │             │ API Gateway     │
│ Lambda Authorizer│            │ Lambda Authorizer│
│ (rate check)    │             │ (rate check)    │
└────────┬────────┘             └────────┬────────┘
         │                               │
         ▼                               ▼
ElastiCache Redis         ElastiCache Redis
(Global Datastore)  ←─────────────────►(Global Datastore)
Primary             async replication   Secondary
         │                               │
         └──── ~100ms replication lag ───┘

Per-request flow (< 5ms):
1. Lambda extracts client_id + endpoint from request
2. Redis EVAL Lua script (atomic):
   - INCR key: "rate:client_id:minute:1710000000"
   - SET EXPIRE: 60 seconds (sliding window)
   - If count > limit: return 429
   - Else: return allow
3. Lua script = atomic (no race conditions)

Window strategies:
├── Fixed window: simple, but burst at window boundary
├── Sliding log: accurate, high memory (store each request timestamp)
└── Sliding window counter: best balance:
    current_requests = window_count × ((60 - elapsed) / 60) + current_count

For 500K req/s:
├── Lambda concurrency: ~5K concurrent (100ms per invocation = 50K req/s per function)
│   → Need multiple functions or Lambda provisioned concurrency
├── Redis: cluster mode, 10 shards × 50K req/s = 500K req/s
├── Rate limit keys partitioned by client_id hash → even distribution
└── Local in-memory cache in Lambda: skip Redis for non-rate-limited endpoints
    (cache "this client is well under limit" for 1 second)
```

---

### Q4. DESIGN: Design a real-time collaborative document editing system (Google Docs clone) on AWS.

**Answer:**

```
Client (Browser)
    │
    ▼ WebSocket connection
API Gateway WebSocket API
    │
    ├── $connect → Lambda: authenticate, store connectionId in DynamoDB
    ├── $disconnect → Lambda: remove connectionId
    └── sendEdit → Lambda: process operation
                        │
                        ▼
              Operational Transformation (OT) or CRDT engine
              (Lambda processes edits, resolves conflicts)
                        │
                        ├── Write edit to DynamoDB:
                        │   ├── PK: doc_id
                        │   ├── SK: timestamp#userId#seq
                        │   └── operation: {type, position, content}
                        │
                        ├── Publish to SNS topic: doc_{doc_id}
                        │         │
                        │         ▼
                        │   Lambda: broadcast to all other
                        │   connected clients via WebSocket
                        │   (using connectionIds from DynamoDB)
                        │
                        └── Periodically: compact operations → snapshot
                            (Step Functions scheduled job)
                            Snapshot → S3 (doc versions)

Presence (who's editing, cursor positions):
├── ElastiCache Redis Pub/Sub: publish cursor positions
├── Lambda subscribes and fans out to WebSocket connections
└── TTL on presence keys (user appears offline if no heartbeat)

Document storage:
├── DynamoDB: real-time operation log (last 1000 ops per doc)
├── S3: document snapshots (full content at point in time)
├── S3: binary attachments (images, tables)
└── CloudFront: serve document snapshots + static assets

Access control:
├── Cognito: user authentication
├── IAM conditions on API Gateway: only doc members can connect
└── Document ACL in DynamoDB: owner, editors, viewers
```

---

### Q5. DESIGN: Design a leaderboard system for a mobile game — 50 million users, real-time global rankings, weekly resets.

**Answer:**

```
Game Server (Lambda / EKS)
    │
    ▼ Score update (user completes level)
    │
    ├──► DynamoDB: users table
    │    ├── PK: user_id
    │    ├── weekly_score (updated atomically with ADD)
    │    └── global_score
    │
    └──► ElastiCache Redis (Sorted Set — the leaderboard engine)
         │
         ZADD leaderboard:weekly user_id score
         (O(log N) insert, O(log N) rank lookup)
         │
         ├── ZRANK leaderboard:weekly user_id → global rank
         ├── ZRANGE leaderboard:weekly 0 99 WITHSCORES → top 100
         ├── ZRANGEBYSCORE → users in a score range
         └── Friend leaderboard:
             ZINTERSTORE friend_leaderboard:user_id 2
             leaderboard:weekly friends:user_id
             (intersection of weekly leaderboard + user's friend IDs)

Scale for 50M users:
├── Redis Sorted Set: O(log N) ops, handles 50M entries well
├── Redis Cluster Mode: shard by leaderboard type
│   ├── Shard 0: global leaderboard
│   ├── Shards 1-N: regional leaderboards (use hash tags)
│   └── {leaderboard}:na, {leaderboard}:eu, {leaderboard}:asia
│       (same hash tag = same shard for regional groupings)
├── Read replicas: top-100 reads hit replicas, writes to primary
└── ElastiCache Global Datastore: replicate to all regions

Weekly reset:
EventBridge scheduled rule → Step Functions
├── Snapshot current top 1000 → DynamoDB (historical)
├── Publish weekly results → SNS → push notifications
├── Award prizes → SQS → fulfillment service
└── RENAME leaderboard:weekly leaderboard:weekly:prev
    DEL leaderboard:weekly (atomic reset)
```

---

### Q6. DESIGN: Design a healthcare data platform that ingests patient data from hospitals, enables research queries, and complies with HIPAA.

**Answer:**

```
HIPAA Compliance Architecture:

Data Ingestion:
Hospitals → AWS HealthLake (FHIR API)
         → Transfer Family (SFTP for legacy HL7/FHIR files → S3)
         → API Gateway → Lambda → data validation → S3

Data Lake (HIPAA-eligible services only):
├── S3 (HIPAA eligible with BAA)
│   ├── PHI bucket: KMS CMK encrypted, versioning ON, Object Lock
│   ├── De-identified bucket: for research (safe harbor / expert determination)
│   └── S3 Replication to secondary region (DR)
│
├── AWS Glue: ETL pipeline
│   ├── De-identification: remove 18 HIPAA identifiers
│   │   (names, SSN, DOB → replaced with synthetic IDs)
│   └── Write to de-identified bucket for research
│
└── AWS Macie: scan for PHI in S3 (alert if PHI found in wrong bucket)

Research Access:
De-identified S3 → Athena → researchers (no direct PHI access)
PHI access: only authorized clinical systems via PrivateLink

Clinical access:
Lambda (in VPC) → PrivateLink → RDS Aurora (PHI database)
├── IAM authentication (no passwords)
├── Encrypted in transit (TLS) + at rest (KMS)
└── Audit: every query logged to CloudTrail + Aurora audit log

Governance:
├── AWS Organizations: PHI accounts in dedicated OU
├── SCPs: deny all services not on HIPAA eligible list
├── CloudTrail: all PHI access logged, integrity validation
├── Config: continuous compliance monitoring
├── GuardDuty: threat detection (unusual access patterns)
├── Macie: PHI discovery and classification
└── Business Associate Agreement (BAA) with AWS
```

---

### Q7. DESIGN: Design a distributed tracing and observability platform for 1000 microservices.

**Answer:**

```
1000 Microservices (EKS / ECS / Lambda)
    │
    ├── Instrument with AWS Distro for OpenTelemetry (ADOT)
    │   ├── Auto-instrumentation via init container (EKS)
    │   ├── Traces → AWS X-Ray (via ADOT collector)
    │   ├── Metrics → CloudWatch Metrics (EMF format)
    │   └── Logs → CloudWatch Logs (structured JSON)
    │
    └── ADOT Collector (DaemonSet on EKS)
        ├── Receives OTLP traces/metrics/logs
        ├── Batches and sends to:
        │   ├── X-Ray (traces)
        │   ├── CloudWatch (metrics)
        │   └── CloudWatch Logs (logs) → Kinesis Firehose → S3
        └── Sends to Managed Grafana / Managed Prometheus (optional)

Trace Storage & Query:
├── X-Ray: 30 days trace retention, service map, root cause analysis
├── Sampling: 100% for errors/5xx, 5% for successful requests
└── X-Ray Groups: separate sampling per service tier

Metrics:
├── CloudWatch Metrics: per-service, per-endpoint metrics
├── CloudWatch Contributor Insights: top slow endpoints, top error users
└── Managed Prometheus → Managed Grafana (Prometheus-native dashboards)

Logs:
├── CloudWatch Logs Insights: ad-hoc queries (no pre-schema needed)
├── Kinesis Firehose → S3 → Athena: long-term log queries
└── OpenSearch: full-text log search + Dashboards

Alerting:
├── CloudWatch Alarms (metric-based)
├── CloudWatch Composite Alarms (reduce noise: "alert if P99 high AND error rate high")
├── X-Ray Insights: automatic anomaly detection on trace data
└── SNS → PagerDuty / Slack / OpsGenie

SLO Monitoring:
├── CloudWatch Metrics Math: calculate error budget burn rate
├── SLO dashboard: requests served within SLA target
└── Alert when error budget burn rate > 2x threshold (30-day window)
```

---

### Q8. DESIGN: Design a data warehouse migration from on-premises Teradata to Amazon Redshift with zero downtime.

**Answer:**

```
Phase 1: Assessment (weeks 1-2)
├── AWS Schema Conversion Tool (SCT): scan Teradata schemas
├── Identify: incompatible functions, macros, stored procedures
├── Estimate conversion effort: green/amber/red assessment
└── Map Teradata distribution keys → Redshift DISTKEY/SORTKEY

Phase 2: Schema Migration (weeks 3-6)
├── SCT converts DDL: CREATE TABLE, views, procedures
├── Manual conversion of SCT-flagged items
├── Load test tables in Redshift dev environment
└── Query performance baseline tests

Phase 3: Initial Load (weeks 7-10)
├── AWS DMS: Full load from Teradata → Redshift
│   ├── DMS replication instance in VPC
│   ├── Teradata as source (over Direct Connect)
│   └── Redshift as target
├── Load in parallel: N DMS tasks per schema
├── Post-load: validate row counts, checksums
└── Redshift ANALYZE: update statistics for query optimizer

Phase 4: CDC (Change Data Capture) — weeks 11-12
├── DMS: enable CDC from Teradata logs
├── Incremental changes replicated to Redshift continuously
│   (lag typically <1 minute)
├── Validate: compare latest data in Teradata vs Redshift
└── Application teams validate queries produce same results

Phase 5: Cutover (zero downtime)
├── Freeze writes to Teradata (maintenance window — minutes)
├── Wait for DMS CDC to catch up to lag=0
├── Validate final data consistency
├── Update application connection strings → Redshift
├── Run smoke tests on Redshift
└── Teradata goes to read-only (available for fallback for 30 days)

Post-migration:
├── Redshift Advisor: recommendations for sort keys, distribution
├── Enable Redshift Serverless for bursty workloads
└── Cost comparison: on-prem Teradata licensing vs Redshift
```

---

## Section 2: Tricky Gotcha Questions

### Q9. TRICKY: You enable S3 versioning on a bucket. Your team accidentally deletes an object. How do you recover it?

**Answer:**

When you delete a versioned object WITHOUT specifying a version ID, S3 adds a **delete marker** — it does NOT permanently delete the data. The object appears deleted but all previous versions still exist.

**Recovery:**
1. List object versions: `aws s3api list-object-versions --bucket my-bucket --prefix my-key`
2. The most recent version will show type "DeleteMarker".
3. Delete the delete marker: `aws s3api delete-object --bucket my-bucket --key my-key --version-id <delete-marker-version-id>`
4. Object is now visible and accessible again.

**Permanent delete:** Only happens when you specify a version ID. This cannot be recovered unless you have a backup or S3 replication.

**TRICKY gotcha:** If `MFA Delete` is enabled on the bucket, deleting version IDs or delete markers requires an MFA token in the API call. This adds protection against accidental/malicious permanent deletes.

---

### Q10. TRICKY: An EC2 instance in a private subnet cannot reach the internet. The subnet has a route `0.0.0.0/0 → nat-xxxxxxxx`. What could be wrong?

**Answer:**

Many possible causes — work through systematically:

1. **NAT Gateway in wrong subnet:** NAT Gateway must be in a PUBLIC subnet. If you put it in a private subnet, it has no internet access itself.

2. **NAT Gateway subnet has no IGW route:** The public subnet containing the NAT Gateway needs `0.0.0.0/0 → igw-xxxxxxxx`. If it's missing, NAT Gateway can't reach internet.

3. **Route table not associated:** The private subnet's route table might not be the one with the NAT route. Check which route table the private subnet is ACTUALLY associated with (not just which table has the route).

4. **NAT Gateway in wrong AZ:** NAT Gateways are AZ-scoped. A NAT Gateway in AZ-a cannot serve instances in AZ-b without cross-AZ traffic charges. More importantly, routing tables must correctly point each AZ's private subnet to that AZ's NAT Gateway.

5. **Security Group denying outbound:** Check the instance's security group outbound rules.

6. **NACL denying traffic:** NACLs are stateless. Check both inbound AND outbound rules. Outbound on ephemeral ports (1024-65535) must be allowed.

7. **NAT Gateway is in FAILED state:** Check NAT Gateway status in VPC console.

8. **Elastic IP not associated with NAT Gateway:** NAT Gateway needs an EIP. If EIP was released, NAT Gateway fails.

---

### Q11. TRICKY: Your Lambda function suddenly starts timing out. It was working fine yesterday. No code changes were deployed. What do you check?

**Answer:**

No code change = external dependency issue. Check in this order:

1. **Downstream service degradation:** Is the database, API, or service the Lambda calls experiencing issues? Check CloudWatch metrics for RDS, DynamoDB, ElastiCache — look for increased latency, connection pool exhaustion.

2. **Lambda in VPC — ENI issues:** If Lambda is VPC-attached, check if the subnet is running out of IPs. Lambda creates Hyperplane ENIs — if the subnet CIDR is exhausted, Lambda can't create new environments → timeouts.

3. **Concurrency throttling:** Check `Throttles` metric. If Lambda is hitting reserved concurrency limit or account limit, some invocations queue and eventually timeout.

4. **DynamoDB/RDS connection exhaustion:** If you're not reusing connections (creating new connection per invocation), a burst in traffic creates too many connections → DB rejects new connections → timeout.

5. **External API rate limiting:** Third-party API you're calling may have started rate-limiting your Lambda (HTTP 429 responses that your code retries instead of failing fast).

6. **Memory pressure:** If Lambda memory was previously adequate but data sizes grew, Lambda may be spending more time in GC → effectively slower → timeout.

7. **Lambda timeout value changed:** Someone might have lowered the timeout in configuration even without a code deploy.

8. **Layer or dependency changes:** A Lambda Layer used by the function may have been updated.

---

### Q12. TRICKY: What happens if two Lambda functions simultaneously try to write the same item to DynamoDB? Is there a race condition?

**Answer:**

**Yes, there is a race condition by default.** DynamoDB `PutItem` is a last-writer-wins operation at the item level. If two Lambdas write to the same partition key + sort key simultaneously, the last write wins and the other is silently overwritten.

**Solutions for different scenarios:**

**1. Conditional writes (optimistic locking):**
```python
# Lambda A and Lambda B both try to update a user's balance
response = dynamodb.update_item(
    Key={'user_id': '123'},
    UpdateExpression='SET balance = :new',
    ConditionExpression='balance = :expected',  # only update if current value matches
    ExpressionAttributeValues={
        ':new': Decimal('150'),
        ':expected': Decimal('100')  # what we read before
    }
)
# If balance was changed by Lambda B between our read and write:
# → ConditionalCheckFailedException
# → We retry with the new value
```

**2. Atomic counters (no race condition for increments):**
```python
# Safe atomic increment — no read-modify-write needed
dynamodb.update_item(
    Key={'item_id': '123'},
    UpdateExpression='ADD quantity :delta',
    ExpressionAttributeValues={':delta': Decimal('1')}
)
```

**3. DynamoDB Transactions (ACID across multiple items):**
```python
dynamodb.transact_write_items(Items=[
    {'Update': {'Key': {'account': 'A'}, 'UpdateExpression': 'SET balance = balance - :amount'}},
    {'Update': {'Key': {'account': 'B'}, 'UpdateExpression': 'SET balance = balance + :amount'}}
])
# Both succeed or both fail — atomic
```

---

### Q13. TRICKY: CloudFormation stack update fails and rolls back. But the rollback also fails. What do you do?

**Answer:**

This leaves the stack in `UPDATE_ROLLBACK_FAILED` state — a nasty situation.

**Why rollback can fail:**
- A resource that CloudFormation is trying to restore was manually deleted outside CloudFormation.
- A resource limit was hit during rollback.
- An external dependency changed (e.g., a Lambda function it's trying to re-create references a resource that no longer exists).

**Resolution steps:**

1. **`ContinueUpdateRollback` API:**
   ```
   aws cloudformation continue-update-rollback --stack-name MyStack
   ```
   CloudFormation retries the rollback. If the same issue recurs, it fails again.

2. **`ContinueUpdateRollback` with `ResourcesToSkip`:**
   ```
   aws cloudformation continue-update-rollback \
     --stack-name MyStack \
     --resources-to-skip LogicalResourceId1
   ```
   Skip the problematic resource. CloudFormation completes rollback ignoring that resource. The skipped resource remains in whatever state it's in, and the stack can be updated manually.

3. **Delete and recreate** (last resort): If the stack is unrecoverable, delete it (which may also fail for resource dependencies) and rebuild from scratch.

**Prevention:** Always use change sets, test in staging first, never manually modify CloudFormation-managed resources.

---

### Q14. TRICKY: Your DynamoDB table is throttling writes even though you have plenty of WCU provisioned. CloudWatch shows utilization at only 40%. Why?

**Answer:**

This is a **hot partition** problem. DynamoDB distributes WCU evenly across all partitions. If you have 10,000 WCU and 100 partitions, each partition gets 100 WCU. But if 80% of your writes go to one partition, that partition gets throttled at 100 WCU even though the table-level average is only 40%.

**The 40% average is misleading** — it's a table-level metric, not a per-partition metric.

**Diagnosis:**
- CloudWatch Metric: `ThrottledRequests` on DynamoDB.
- DynamoDB Contributor Insights: shows which partition keys are "hot" (most accessed).
- Enable Contributor Insights: it shows you top partition keys by request count, throttles.

**Solutions:**
1. **Redesign partition key** to have higher cardinality (user_id + timestamp vs just user_id).
2. **Write sharding:** append random suffix (0–N) to hot partition key. Read by querying all N shards.
3. **DynamoDB On-Demand mode:** automatically scales per-partition without pre-provisioning (costs more per request but no throttling).
4. **DAX in front:** cache popular reads to reduce read pressure. Write-through DAX for write aggregation.
5. **Adaptive Capacity:** DynamoDB already tries to redistribute capacity to hot partitions automatically — but this only helps within a partition's physical shard group.

---

### Q15. TRICKY: An ECS task is in STOPPED state with exit code 137. What does this mean?

**Answer:**

Exit code **137 = 128 + 9 (SIGKILL)**. The process was killed by the OS — not a graceful shutdown.

**Most common causes:**

1. **OOM (Out of Memory) Kill:** The container exceeded its memory limit. The Linux kernel OOM killer sent SIGKILL. Check CloudWatch Container Insights for memory utilization. Fix: increase task memory limit, or find and fix the memory leak.

2. **ECS task memory limit exceeded:** The ECS agent killed the container because the task exceeded its soft or hard memory limit. Different from Linux OOM — ECS enforces the limit defined in the task definition.

3. **Instance-level memory pressure:** If the EC2 instance (for ECS on EC2) is running out of memory, the OS may kill containers.

**Differentiate OOM vs other kills:**
- Check ECS task stopped reason in the console.
- CloudWatch Container Insights: `MemoryUtilized` metric near or at `MemoryReserved`.
- `docker inspect` on the container (if on EC2, SSH to the host): `OOMKilled: true` in the State field.

**Exit code reference:**
- 0: success
- 1: general error
- 137: SIGKILL (usually OOM)
- 139: segmentation fault (SIGSEGV)
- 143: SIGTERM (graceful stop — ECS sends this before SIGKILL)

---

### Q16. TRICKY: You're using Aurora MySQL. A developer runs a long-running query on the read replica. Does it affect the primary writer?

**Answer:**

**Yes, it can.** Here's why:

**Replication lag:** Aurora uses shared storage so replication is near-zero (milliseconds). BUT if a read replica is handling a very long-running query, Aurora may delay applying redo log records to avoid conflicting with the ongoing query's snapshot. This increases **replica lag**.

**Impact on primary:** If you're reading from the writer endpoint and your application's logic depends on reading data that was JUST written, the lag won't matter. But if your app specifically uses read replicas for "fresh" data, the lag matters.

**More direct impact:** With RDS (non-Aurora) MySQL, long queries on the replica hold a read lock that can prevent the SQL thread from applying binlog updates → replica falls far behind. Aurora's storage-level replication is more resilient to this, but it's still not immune to lag under extreme read pressure.

**Connection pool exhaustion:** A long-running query holds a database connection. If a developer is running a multi-hour analytical query on a production read replica connection pool, other application connections may starve.

**Best practice:** Separate read replicas: one pool for application reads (fast, short queries), one replica for analytical/ad-hoc queries (can absorb long-running queries without affecting app).

---

### Q17. TRICKY: What is the difference between an ALB target group with `target type = instance` vs `target type = ip`?

**Answer:**

**Instance target type:**
- Traffic is routed to the instance's primary network interface.
- Must be in the same VPC as the ALB.
- Instance must have a Security Group allowing traffic from ALB.
- Cannot target specific ports on containers on the same EC2 host with different port mappings.

**IP target type:**
- Traffic is routed to any IP address in the VPC CIDR or VPC peering/Direct Connect ranges.
- Supports: specific container ports on ECS tasks (dynamic port mapping), on-premises servers via Direct Connect/VPN, Lambda (via Lambda target), IPs across peered VPCs.
- Required for **ECS Fargate** (Fargate tasks have IP addresses, not instance IDs).
- Required for **Lambda** as a target.

**Security Group difference (TRICKY):**
- Instance target type: the instance's SG must allow the ALB SG as a source.
- IP target type: the instance's SG must allow the specific IP addresses. But for Fargate: the task's SG must allow traffic from the ALB's SG (since Fargate has its own SG on the task's ENI).

---

### Q18. TRICKY: What is the maximum size of an SQS message and how do you handle payloads larger than this?

**Answer:**

SQS maximum message size is **256 KB**.

**Extended Client Library pattern (S3-backed large messages):**
```
Producer:
├── If message < 256 KB → send directly to SQS
└── If message > 256 KB:
    ├── Upload payload to S3: s3://my-bucket/sqs-payloads/uuid
    ├── Put reference in SQS message: {"s3Key": "sqs-payloads/uuid", "size": 1234567}
    └── Send small reference message to SQS

Consumer:
├── Receive SQS message
├── Check if message is an S3 reference
├── If yes: download payload from S3
├── Process full payload
└── Delete S3 object + SQS message

AWS provides the Extended Client Library for Java/Python that handles this transparently.
```

**S3 lifecycle on the payloads bucket:**
- Set lifecycle rule to delete objects after 14 days (matching SQS max retention).
- Prevents orphaned S3 objects if consumer fails to clean up.

---

### Q19. TRICKY: How does CloudFront handle requests when all origins in an origin group are unhealthy?

**Answer:**

CloudFront origin groups have a primary and failover origin. Here's the exact behavior:

1. CloudFront sends request to **primary origin**.
2. If primary returns 5xx (or connection fails) AND the error code is in the **failover criteria** list → CloudFront retries with **failover origin**.
3. If failover origin also fails → CloudFront returns the error to the client (whichever response it got from the failover attempt).

**CloudFront does NOT maintain a "both origins unhealthy" state like a load balancer does.** It always tries primary first, then failover if configured failover criteria are met. There's no circuit breaker that says "skip primary because it was unhealthy yesterday."

**Tricky detail:** Failover only triggers for specific HTTP error codes you configure (default: 500, 502, 503, 504). A 404 from your origin does NOT trigger failover — CloudFront serves the 404 to the client. This is intentional: a missing resource (404) should not fail over to another origin that may return a different result.

**If you need true health-based failover:** Use Route 53 health checks with failover routing policy in front of CloudFront, routing between two CloudFront distributions.

---

### Q20. TRICKY: A developer says "I added an explicit Allow in my IAM policy but the action is still being denied." Walk through how you would diagnose this.

**Answer:**

Systematic IAM policy evaluation debugging:

```
Step 1: Check for Explicit Deny
└── aws iam simulate-principal-policy --action-names s3:PutObject
    → This shows you exactly why access is denied

Step 2: Check SCPs (if AWS Organizations)
└── Are there SCPs on the account or OU that deny the action?
    Even AdministratorAccess is blocked if SCP denies it.

Step 3: Check Permission Boundaries
└── Does the role/user have a Permission Boundary attached?
    aws iam get-role --role-name MyRole → look for PermissionsBoundary
    The action must be allowed by BOTH the boundary AND the policy.

Step 4: Check Resource-Based Policies
└── Does the target resource (S3 bucket, SQS queue, KMS key) have
    a resource policy that denies the principal?

Step 5: Check Session Policies
└── Was the role assumed with --policy (session policy)?
    Session policies are an intersection with the role's policies.

Step 6: Check the specific resource ARN
└── Policy might say: "Resource": "arn:aws:s3:::my-bucket/*"
    But you're trying to: s3:ListBucket on "arn:aws:s3:::my-bucket"
    (without trailing /*) → not covered by the policy

Step 7: Check condition keys
└── Policy might have: "Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}
    If you assumed the role without MFA, condition fails → deny.

Step 8: Use IAM Policy Simulator
└── AWS Console: IAM → Policy Simulator
    Enter the exact principal, action, and resource
    → Shows which policy statement is causing the deny
```

---

## Section 3: AI/ML on AWS

### Q21. SageMaker — explain the difference between training jobs, endpoints, and batch transform.

**Answer:**

**Training Jobs:**
- Spins up EC2 instances (you choose type/count), runs your training script, stores model artifacts in S3.
- Pay only for training duration.
- Supports distributed training (multi-node with parameter server or Horovod).
- Spot training: 70% cost saving, handles interruptions via checkpointing.
- After job completes: instances terminated, you have model artifacts in S3.

**Real-time Endpoints:**
- Deploy model to always-on inference endpoint (EC2 backed).
- Auto-scaling based on invocations or custom metrics.
- Pay per instance-hour while endpoint is up.
- Latency: typically <100ms for well-optimized models.
- Multi-model endpoints: one endpoint, multiple models in memory (cost efficient for many small models).
- Serverless inference: auto-scales to zero when no requests (pay per invocation, cold starts possible).

**Batch Transform:**
- One-time bulk inference job on a dataset in S3.
- No persistent endpoint.
- Pay only for job duration.
- Processes large datasets in parallel.
- Output written to S3.
- Use for: offline scoring, feature generation, A/B testing model outputs.

**Async Inference:**
- Queue-based: send request → get back a URL to poll for result.
- No timeout (unlike real-time 60s limit).
- Ideal for large inputs (video, high-res images, long documents).
- Scales to zero when queue is empty.

---

### Q22. DESIGN: Design an ML pipeline for model training, evaluation, and deployment with CI/CD.

**Answer:**

```
Data Pipeline:
S3 (raw data) → Glue ETL → S3 (processed) → SageMaker Feature Store
                                                    │
                                              Training features
                                              (online + offline store)

ML Pipeline (SageMaker Pipelines):
┌─────────────────────────────────────────────────────┐
│  SageMaker Pipeline                                  │
│                                                      │
│  1. Processing Step: feature engineering            │
│     (SageMaker Processing Job — scikit-learn)       │
│                                                      │
│  2. Training Step                                    │
│     ├── SageMaker Training Job (XGBoost / PyTorch)  │
│     ├── Spot instances + checkpointing              │
│     └── Output: model artifacts → S3               │
│                                                      │
│  3. Evaluation Step                                  │
│     ├── SageMaker Processing Job                    │
│     ├── Compute: AUC, F1, precision, recall        │
│     └── Output: evaluation.json → S3               │
│                                                      │
│  4. Condition Step                                   │
│     └── AUC > 0.85? YES → continue, NO → fail      │
│                                                      │
│  5. Register Model (SageMaker Model Registry)       │
│     ├── Status: PendingManualApproval               │
│     └── Metadata: metrics, training data version   │
│                                                      │
│  6. Human Approval (Model Registry approval)        │
│     └── Data scientist approves in SageMaker Studio│
└─────────────────────────────────────────────────────┘

CD Pipeline (EventBridge + CodePipeline):
Model approved in Registry
        │
        ▼
EventBridge rule triggers CodePipeline
        │
        ▼
Deploy to Staging endpoint → run integration tests
        │
        ▼
Manual approval gate
        │
        ▼
Deploy to Production endpoint (Blue/Green traffic shifting)
├── Start at 10% traffic to new model
├── CloudWatch monitors model quality metrics
└── Full traffic shift if metrics stable

Model monitoring (SageMaker Model Monitor):
├── Data quality: detect data drift from training distribution
├── Model quality: monitor prediction accuracy with ground truth labels
├── Bias drift: Clarify for fairness monitoring
└── Alert if drift detected → trigger retraining pipeline
```

---

### Q23. What is Amazon Bedrock and how does it differ from SageMaker?

**Answer:**

**Amazon Bedrock:**
- Fully managed service providing access to **foundation models (FMs)** from multiple providers: Anthropic (Claude), Meta (Llama), Mistral, Cohere, Amazon (Titan), Stability AI (Stable Diffusion).
- No training infrastructure — you call an API.
- Supports: text generation, summarization, Q&A, embeddings, image generation, code generation.
- **Fine-tuning:** You can fine-tune supported models on your own data (stored in S3) without managing training infrastructure.
- **RAG (Retrieval Augmented Generation):** Bedrock Knowledge Bases — connect FM to your documents (S3), automatically chunks, embeds, and stores in a vector store (OpenSearch Serverless), handles retrieval at inference time.
- **Agents:** Bedrock Agents orchestrate multi-step tasks — break down user request, call APIs/Lambda functions, return synthesized result.

**SageMaker:**
- Full ML platform: build, train, deploy ANY model (not just FMs).
- You bring your own training code, data, model architecture.
- More control, more complexity.
- Better for: custom ML models, proprietary training data, research, fine-tuning open-source models deeply.

**Simple rule:**
- Need to USE a foundation model → Bedrock.
- Need to BUILD and train a custom ML model → SageMaker.
- Need to fine-tune a foundation model extensively → SageMaker (JumpStart) or Bedrock (light fine-tuning).

---

## Section 4: Advanced Cost Optimization

### Q24. You're spending $500K/month on AWS. The CEO asks you to cut 30% without impacting production. How do you approach this?

**Answer:**

**Systematic approach (not guessing):**

**Step 1: Get visibility (week 1)**
```
├── AWS Cost Explorer: top 5 cost drivers
├── Cost and Usage Report (CUR) → Athena: detailed analysis
├── AWS Compute Optimizer: right-sizing recommendations
└── Trusted Advisor: cost optimization checks
```

**Step 2: Quick wins (weeks 2-4, 15–20% savings)**
```
├── Right-size EC2 instances (Compute Optimizer shows ~$50K/month waste)
│   └── Downsize over-provisioned instances
├── Purchase Savings Plans for steady-state EC2/Lambda/Fargate
│   └── 66% discount on flexible compute → major savings
├── Identify and delete unused resources:
│   ├── Unattached EBS volumes ($20K+ often)
│   ├── Idle NAT Gateways
│   ├── Unused Elastic IPs ($0.005/hr each × thousands = significant)
│   └── Stale snapshots (old EBS snapshots never deleted)
├── S3 Intelligent Tiering: auto-moves to cheaper tiers
└── CloudFront: reduce origin data transfer costs
```

**Step 3: Architecture improvements (months 2-3, 10–15% additional savings)**
```
├── Migrate appropriate Lambda + Fargate to Compute Savings Plans
├── Use Spot Instances for:
│   ├── Batch processing (70-90% savings)
│   ├── Dev/test environments (schedule: on at 8am, off at 8pm)
│   └── CI/CD build jobs (CodeBuild uses spot automatically)
├── Data transfer optimization:
│   ├── Same-AZ communication (cross-AZ data transfer is expensive)
│   ├── S3 VPC Gateway Endpoints (eliminate NAT Gateway costs for S3)
│   └── CloudFront compression (reduce data transfer out)
├── Database right-sizing:
│   ├── Aurora Serverless for dev/staging (scales to zero)
│   ├── Reserved Instances for production Aurora
│   └── ElastiCache right-sizing
└── DynamoDB: provisioned + Auto Scaling vs on-demand for predictable workloads
```

**Typical breakdown for $500K/month:**
- EC2 right-sizing + Savings Plans: saves ~$80K
- S3 lifecycle + Intelligent Tiering: saves ~$20K
- Unused resources cleanup: saves ~$30K
- Dev environment scheduling: saves ~$20K
- **Total: ~$150K (30% savings achieved)**

---

### Q25. What is AWS Compute Optimizer and how does it work?

**Answer:**

Compute Optimizer uses **ML models** to analyze CloudWatch utilization metrics for your EC2 instances, Lambda functions, ECS services, and Auto Scaling Groups, then recommends optimal resource sizes.

**How it works:**
- Analyzes last 14 days of CPU, memory (requires CloudWatch agent), network, disk utilization.
- Compares actual utilization against the resources you're provisioned for.
- Recommends up-sizing (if you're consistently at 90%+ CPU) or down-sizing (if you're at 5% CPU).
- Provides estimated monthly savings.

**What it can analyze:**
- EC2 instances: CPU, memory, network, EBS.
- EC2 Auto Scaling Groups: entire ASG configuration.
- Lambda: memory size vs average memory used.
- ECS on Fargate: CPU and memory.
- EBS volumes: IOPS utilization (gp2 → gp3 recommendations often save 20%).

**TRICKY:** Compute Optimizer doesn't have memory data by default for EC2. You MUST have the **CloudWatch Agent** installed and sending memory metrics. Without memory data, recommendations may suggest downsizing RAM when you actually need it.

**Savings Plans purchase recommendations** are separate from Compute Optimizer — found in Cost Explorer → Savings Plans → Recommendations.

---

## Section 5: Missing Services Quick-Fire

### Q26. What is AWS Backup and how does it differ from service-native backups?

**Answer:**

**AWS Backup** is a centralized backup service that manages backups across multiple AWS services from a single console/API.

**Supports:** EBS, RDS, Aurora, DynamoDB, EFS, FSx, EC2, S3, VMware on AWS.

**vs Native backups:**
- RDS automated backups: managed by RDS, stored up to 35 days, automatic.
- AWS Backup: orchestrates backups across all services with unified policy, cross-account/cross-region copy, compliance reporting.

**Key features:**
- **Backup Plans:** Define backup frequency, retention, transition to cold storage.
- **Backup Vault:** Encrypted container for backup data. Vault Lock (WORM) for compliance.
- **Cross-account backup:** Copy backups to a separate AWS account (air-gapped recovery).
- **Cross-region backup:** Automatically copy backups to DR region.
- **AWS Backup Audit Manager:** Generate compliance reports (show that all production resources have daily backups).

---

### Q27. What is AWS DataSync vs Storage Gateway vs AWS Transfer Family?

**Answer:**

| Service | Use case | Protocol | Direction |
|---|---|---|---|
| **DataSync** | One-time or scheduled migration/sync between on-prem and AWS | NFS, SMB, S3, HDFS | On-prem ↔ AWS |
| **Storage Gateway** | Hybrid storage — on-prem apps access AWS storage transparently | NFS, SMB, iSCSI | On-prem ↔ AWS (ongoing) |
| **Transfer Family** | Managed SFTP/FTP/FTPS/AS2 server backed by S3 or EFS | SFTP, FTP, FTPS, AS2 | Clients ↔ AWS |

**Storage Gateway types:**
- **File Gateway:** On-prem NFS/SMB interface, data stored in S3. Local cache for frequently accessed data.
- **Volume Gateway:** iSCSI block storage backed by S3. Stored volumes (full data on-prem + S3 backup) or Cached volumes (primary in S3, cache on-prem).
- **Tape Gateway:** Virtual tape library backed by S3/Glacier. Replace physical tape with cloud.

**DataSync vs Snowball:**
- DataSync: online migration over Direct Connect or internet.
- Snowball: offline migration — ship physical device. Use when you have >10TB and network transfer would take too long.

---

### Q28. What is AWS AppFlow and when would you use it?

**Answer:**

AppFlow is a fully managed integration service that transfers data between SaaS applications and AWS services — without writing integration code.

**Supports:** Salesforce, ServiceNow, Slack, Zendesk, Google Analytics, SAP → S3, Redshift, EventBridge, Snowflake.

**Use cases:**
- Sync Salesforce CRM data to Redshift for analytics — no custom ETL code.
- Pull support tickets from Zendesk to S3 for ML training.
- Transfer Google Analytics data to Redshift for BI.
- Near-real-time (event-triggered) or scheduled transfers.
- Built-in data transformation: filtering, masking, mapping field names.

**vs Glue:** Glue requires you to write transformation code. AppFlow is no-code for SaaS-to-AWS transfers. For custom transformations or non-SaaS sources, use Glue.

---

### Q29. What is AWS EventBridge Scheduler and how is it different from EventBridge rules with schedules?

**Answer:**

Both can trigger targets on a schedule, but they have different use cases:

**EventBridge Rules (schedule-based):**
- Cron or rate expressions.
- ONE target definition per rule.
- Fires at the scheduled time, regardless of whether previous execution finished.
- No per-execution state or tracking.
- Best for: simple recurring events (hourly, daily).

**EventBridge Scheduler (newer service):**
- One-time schedules (fire once at a specific time) OR recurring.
- Up to 50 million scheduled invocations per second (scales far beyond rules).
- **Per-schedule state:** Each schedule is a distinct resource, can be created/deleted programmatically.
- **Flexible time windows:** Allow invocation within a time window (reduces thundering herd).
- **Dead-letter queue:** Failed invocations can be sent to DLQ.
- **IAM role per schedule:** Each schedule assumes a role to invoke the target.
- Best for: creating millions of per-user reminders, per-order timeouts, scheduled per-customer reports.

**Example:** An e-commerce site creates a "abandon cart reminder" schedule 1 hour after cart abandonment for each customer. EventBridge Scheduler handles millions of individual one-time schedules easily. EventBridge rules would require a different architecture.

---

### Q30. What is Amazon SES, SNS for email/SMS, and Pinpoint — when do you use each?

**Answer:**

| Service | Purpose | Volume | Use case |
|---|---|---|---|
| **SES (Simple Email Service)** | Transactional + bulk email | Very high (millions/day) | Receipts, password reset, marketing campaigns |
| **SNS (email/SMS subscriptions)** | Simple notifications | Lower | Alert emails to ops team, SMS for alarms |
| **Pinpoint** | Multi-channel customer engagement | High | Marketing campaigns, user journeys, analytics |

**SES deep dive:**
- SMTP interface OR AWS API (SES API v2).
- Sandbox mode (default): can only send to verified addresses. Request production access to send to anyone.
- Dedicated IPs (optional): ensure your IP reputation isn't affected by other SES senders.
- Configuration sets: track opens, clicks, bounces, complaints via SNS/Kinesis Firehose.
- Suppress list: automatically suppresses emails to addresses that bounced (protects reputation).

**Pinpoint:**
- Think of it as a customer engagement platform.
- Define **user segments** (users who haven't logged in for 30 days).
- Design **journeys** (multi-step campaigns: send email → wait 3 days → if no open, send SMS).
- Built-in analytics: open rates, click rates, conversion tracking.
- A/B testing for messages.

---

*End of Part 7 — 30 Questions covering System Design, Tricky Gotchas, Cost Optimization, AI/ML, and Missing Services*

---

## Master Quick Reference: All Service Limits

| Service | Key Limit | Notes |
|---|---|---|
| VPC | 5/region | Raiseable |
| Subnets | 200/VPC | |
| Security Groups | 2,500/VPC; 60 rules/SG | |
| NACLs | 200/VPC; 20 rules each inbound/outbound | |
| VPC Endpoints | 255/VPC | |
| TGW attachments | 5,000/TGW | |
| EC2 On-Demand | vCPU-based limit per family | Instance limits are vCPU-based now |
| Lambda concurrent | 1,000/region default | Raiseable to 10K+ |
| Lambda memory | 128 MB – 10,240 MB | |
| Lambda timeout | 15 minutes hard max | |
| Lambda package size | 50 MB zip / 250 MB unzipped / 10 GB container | |
| Lambda layers | 5 per function | |
| API Gateway timeout | 29 seconds hard max | |
| API Gateway payload | 10 MB REST; 128 KB WebSocket | |
| SQS message | 256 KB | Use extended client for larger |
| SQS retention | 1 min – 14 days | |
| SQS visibility timeout | 0 – 12 hours | |
| SNS message | 256 KB | |
| SNS subscriptions | 12.5 million/topic | |
| Kinesis shard | 1 MB/s write, 2 MB/s read, 1000 rec/s write | |
| DynamoDB item | 400 KB | |
| DynamoDB partition | 3,000 RCU + 1,000 WCU + 10 GB | |
| DynamoDB transaction | 100 items or 4 MB | |
| S3 object | 5 TB max; 5 GB single PUT | |
| S3 prefix requests | 5,500 GET/s per prefix | |
| S3 bucket | 100/account default | Raiseable to 1,000 |
| RDS storage | 64 TB (Aurora) | |
| Aurora replicas | 15 read replicas | |
| ElastiCache Redis cluster | 500 shards | |
| Redshift nodes | 128/cluster | |
| ECS tasks | 500/service (Fargate soft) | |
| EKS nodes | 450/cluster (managed) | |
| CloudFormation resources | 500/stack | Use nested stacks |
| CloudFormation stacks | 2,000/account | |
| IAM roles | 1,000/account | Raiseable |
| IAM policies per entity | 10 managed + inline | |
| Route 53 hosted zones | 500/account | |
| Route 53 records | 10,000/hosted zone | |
| CloudFront distributions | 200/account | |
| CloudFront origins per dist | 25 | |
| ACM certificates | 2,500/account | |
| KMS keys | 100,000/account | |
| Secrets Manager secrets | 500,000/account | |
| Step Functions executions | 1,000,000 open/account | |
| EventBridge rules | 300/event bus | |
| CodePipeline pipelines | 1,000/region | |
