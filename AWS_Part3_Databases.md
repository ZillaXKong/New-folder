# AWS Interview Prep — Part 3: Databases
> RDS, Aurora, DynamoDB, ElastiCache, Redshift, Neptune, QLDB, DocumentDB
> All levels: Basic → Advanced → Tricky Gotchas → Design Questions

---

## RDS & AURORA

### Q1. Aurora vs RDS — what are the architectural differences?

**Traditional RDS:**
- EC2 instance + attached EBS volume.
- Multi-AZ: synchronous block replication to standby EBS in a different AZ.
- Failover: DNS switch + ~60–120 seconds downtime.
- Read replicas: separate EC2 + separate EBS, async binlog replication. Lag can be seconds to minutes.

**Aurora Architecture:**
- **Decouples compute from storage entirely.**
- Storage layer: **6 copies across 3 AZs** (2 per AZ) — a custom distributed storage service.
- **Write quorum: 4/6 nodes** must acknowledge. **Read quorum: 3/6**.
- Storage auto-scales in 10GB increments up to **128 TiB**.
- All read replicas read from the **same shared storage** — zero lag for reads (no replication).
- Failover: **~30 seconds** — no storage replication needed, just flip the writer endpoint.

```
Traditional RDS Multi-AZ:          Aurora:
  Primary (AZ-a)                   Writer (AZ-a)
  └── EBS ──sync──► Standby        Reader (AZ-b)
                    (AZ-b)         Reader (AZ-c)
  Read Replica (async)             Reader (AZ-a)
  └── Separate EBS                       │
      replication lag: variable    Shared Distributed Storage
                                   AZ-a: ██ ██  (2 of 6 copies)
                                   AZ-b: ██ ██  (2 of 6 copies)
                                   AZ-c: ██ ██  (2 of 6 copies)
```

**Aurora Global Database:**
- Primary region + up to 5 secondary regions.
- Replication at the **storage layer** (not SQL/binlog) → typically <1 second lag globally.
- Secondary promoted to primary in <1 minute for DR.
- **Write forwarding:** applications in secondary region can write — Aurora forwards to primary transparently.

---

### Q2. TRICKY — Aurora has 6 copies across 3 AZs. Can it survive an AZ failure?

**Yes.** With 6 copies and quorum requirements:
- **Lose 1 AZ (2 copies):** 4 copies remain. Write quorum (4/6) = still met. Read quorum (3/6) = still met. **No impact.**
- **Lose 2 AZs (4 copies):** 2 copies remain. Write quorum (4/6) = NOT met. Aurora goes into **read-only mode** automatically (it can still serve reads with 2 copies if it has the most recent data). **Writes fail.**

**Additional resilience:**
- Aurora can tolerate up to 2 copy failures for reads.
- Storage automatically repairs itself — if a copy is lost, Aurora replaces it in the background without human intervention.
- No need for manual backup/restore for AZ failures.

---

### Q3. TRICKY — What is the difference between Aurora Serverless v1 and v2?

| | Aurora Serverless v1 | Aurora Serverless v2 |
|---|---|---|
| Scaling granularity | In ACU steps (can take 15-30s) | Smooth, near-instant (fractions of ACU) |
| Scale-to-zero | Yes (cold start ~25s) | No (minimum 0.5 ACU) |
| Multi-AZ | No (single AZ) | Yes |
| Read replicas | No | Yes |
| Global Database | No | Yes |
| VPC | Required | Required |
| Use case | Dev/test, unpredictable, infrequent | Production with variable load |

**v1 gotcha:** Scale-to-zero is appealing but the cold start (25+ seconds) makes it unsuitable for production APIs. Connections are dropped during scaling events.

**v2** is the recommended option for most new deployments. It scales from 0.5 to 128 ACUs (Aurora Capacity Units) nearly instantly, supports all Aurora features, and has no cold start disruption.

---

### Q4. RDS Proxy — what is it and when do you need it?

**Problem it solves:** Applications (especially Lambda functions) that open and close many short-lived database connections overwhelm the DB's connection limit. PostgreSQL/MySQL can handle hundreds to low thousands of connections before performance degrades.

**RDS Proxy:**
- Sits between your application and RDS/Aurora.
- **Connection pooling:** maintains a pool of long-lived connections to the DB. App connections are multiplexed over the pool.
- Lambda → RDS Proxy (thousands of connections from Lambda) → DB (few hundred pooled connections).
- **Automatic failover:** RDS Proxy re-routes connections to the new primary during failover, reducing downtime from 60-120s to 5-10s.
- **IAM authentication:** Proxy can enforce IAM auth, eliminating hardcoded DB passwords.
- Fully managed, highly available, deployed across AZs.

**When you need it:**
- Lambda functions connecting to RDS/Aurora (Lambda can open thousands of concurrent connections).
- Apps with connection spikes exceeding DB limits.
- Need faster failover than standard RDS/Aurora provides.

---

### Q5. TRICKY — You have an RDS Multi-AZ instance. Does a failover affect Read Replicas?

**Yes, but not directly.**

In RDS Multi-AZ:
- Primary (AZ-a) + Synchronous Standby (AZ-b).
- Read Replicas replicate ASYNCHRONOUSLY from the primary.

**During failover:**
1. Primary fails → standby promoted to primary (new DNS resolves to new primary IP within 60-120s).
2. Read Replicas were replicating from OLD primary. After failover, they need to point to the NEW primary.
3. AWS **automatically re-establishes replication** from new primary to read replicas. This may cause:
   - Brief replication lag increase.
   - Read replicas temporarily falling behind.
   - For Aurora: No impact on readers — they all share the same storage layer, no replication to re-establish.

**Aurora advantage here:** Because all replicas share storage, failover is transparent to readers.

---

### Q6. Explain DynamoDB partitioning — how does it work and what causes hot partitions?

**Partitioning:**
- DynamoDB distributes data across partitions using a **consistent hash of the partition key**.
- Each partition: max 10GB data, max 3,000 RCU + 1,000 WCU.
- When a partition exceeds these limits, DynamoDB **splits** it automatically.
- You never control which partition a key maps to.

**Hot partition problem:**
Low-cardinality partition key (e.g., `country = "US"`) → all US users hit one partition → overwhelm its 3,000 RCU / 1,000 WCU → `ProvisionedThroughputExceededException`.

**Solutions:**
1. **High-cardinality partition keys:** Use `user_id` (UUID), not `status` or `country`.
2. **Write sharding:** Append random suffix `_1` to `_N` to partition key. Distribute writes across N partitions. Query all N shards and aggregate.
   ```
   Key: user_events_1, user_events_2, ... user_events_10
   Write: randomly pick suffix → distribute load
   Read: query all 10 partition key variants, merge results
   ```
3. **DAX (DynamoDB Accelerator):** In-memory cache for reads. Hot items served from cache, not DynamoDB. Sub-millisecond latency. Reduces DynamoDB read load dramatically.
4. **Adaptive Capacity:** DynamoDB automatically redistributes capacity within a table to hot partitions. This is reactive and not a substitute for good key design.

---

### Q7. DynamoDB GSI vs LSI — differences and trade-offs?

| | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
|---|---|---|
| Partition key | Same as base table | Different (your choice) |
| Sort key | Different from base table | Your choice |
| AZs / Storage | Shares base table storage | Separate partition space |
| Consistency | Strongly consistent reads supported | Eventually consistent only |
| WCU consumption | Base table WCU | Separate WCU (or on-demand) |
| Creation | At table creation ONLY | Anytime |
| Max per table | 5 | 20 |
| Throttling | Base table throttle = LSI throttle | GSI can independently throttle |

**GSI hot partition backpressure (critical gotcha):**
If your GSI has a hot partition (too many items with the same GSI partition key), the GSI throttles → this throttles WRITES to the base table too. The write must be propagated to the GSI asynchronously, and if the GSI is throttled, base table writes are rejected.

**Fix:** Ensure GSI partition key also has high cardinality.

---

### Q8. TRICKY — DynamoDB table has on-demand capacity. Customer suddenly sends 100x normal traffic. What happens?

**On-demand capacity mode:**
- DynamoDB automatically scales to ANY level of traffic.
- No capacity planning needed. Pay per request.
- **Instantaneous scaling up** — but there IS a limit: previous peak × 2 per table per day.

**The gotcha:** If a table has NEVER seen high traffic and suddenly gets 100× burst, DynamoDB has a "previous peak × 2" warmup. For brand new tables or tables with very low previous peak, there can be throttling during extreme sudden bursts.

**Mitigation:**
- Warm the table: gradually ramp traffic before the big event.
- Contact AWS Support to pre-warm capacity.
- Use exponential backoff in your SDK (built into AWS SDKs by default).

**On-demand vs Provisioned with Auto Scaling:**
- On-demand: higher per-request cost, no planning, handles spikes automatically.
- Provisioned + Auto Scaling: lower cost for steady workloads, ~1-3 minute scale-up lag.
- **Best practice:** Use on-demand for new tables or unpredictable workloads, switch to provisioned once traffic is predictable.

---

### Q9. How does DynamoDB Streams work? Compare to Kinesis Data Streams for DynamoDB?

**DynamoDB Streams:**
- Captures a time-ordered sequence of item-level changes (INSERT, MODIFY, REMOVE).
- Stream records available for **24 hours**.
- Each shard in the stream maps to a DynamoDB partition.
- View types: `KEYS_ONLY`, `NEW_IMAGE`, `OLD_IMAGE`, `NEW_AND_OLD_IMAGES`.
- Lambda trigger built-in — AWS manages the polling.

**Kinesis Data Streams for DynamoDB:**
- Newer option — streams DynamoDB changes to a Kinesis Data Stream you own.
- Retention: up to **1 year** (vs 24h for DynamoDB Streams).
- Multiple consumers (Enhanced Fan-Out).
- Can replay events.
- More expensive (Kinesis costs added).

| | DynamoDB Streams | KDS for DynamoDB |
|---|---|---|
| Retention | 24 hours | Up to 1 year |
| Consumers | 2 simultaneous | Unlimited via EFO |
| Replay | No | Yes |
| Cost | Included | Kinesis charges |
| Setup | Native | Need KDS configuration |

**Use DynamoDB Streams:** Simple Lambda triggers, change data capture for immediate processing.
**Use KDS for DynamoDB:** Long retention for audit, multiple consumers, replay capability.

---

### Q10. DESIGN — Design a DynamoDB table for a ride-sharing app. Users can look up rides by: user_id, by city+date, by driver_id.

**Access patterns:**
1. Get all rides for a user → query by `user_id`
2. Get rides in a city on a date → query by `city` + `date`
3. Get all rides for a driver → query by `driver_id`

**Table design:**

```
Base Table:
  PK: user_id (e.g., "user_abc123")
  SK: ride_id (e.g., "ride_20240115_xyz")

  Attributes: driver_id, city, date, status, fare, timestamp

  → Access Pattern 1: Query PK=user_id → gets all rides for user

GSI-1 (city-date-index):
  PK: city (e.g., "NYC")
  SK: date#ride_id (e.g., "2024-01-15#ride_xyz")

  → Access Pattern 2: Query PK=NYC, SK begins_with "2024-01-15"

GSI-2 (driver-index):
  PK: driver_id (e.g., "driver_def456")
  SK: timestamp (for ordering)

  → Access Pattern 3: Query PK=driver_id → all rides for driver
```

**Key design decisions:**
- Composite SK `date#ride_id` in GSI-1 allows range queries: "all NYC rides on 2024-01-15."
- High-cardinality PK in base table: `user_id` (UUID) — distributes writes evenly.
- GSI-1 PK is `city` — if "NYC" has enormous traffic, this GSI has a hot partition. Fix: shard with `city#shard_n` pattern or use `city#date` as PK.

---

### Q11. ElastiCache Redis vs Memcached — when do you use each?

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Strings, Lists, Sets, Sorted Sets, Hashes, Streams, Bitmaps | Strings only |
| Persistence | RDB snapshots + AOF | None |
| Replication | Yes (primary + replicas) | No |
| Cluster (sharding) | Yes (cluster mode) | Yes (native) |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Sorted sets (leaderboards) | Yes | No |
| Multi-threading | Single-threaded (per node) | Multi-threaded |
| Use case | Sessions, leaderboards, pub/sub, rate limiting, geospatial | Simple caching, highest throughput, multi-threaded |

**Choose Redis when:** You need persistence, replication, complex data structures, pub/sub, or Lua scripting.
**Choose Memcached when:** Pure caching, multi-threaded performance is needed, no persistence required, dead-simple setup.

---

### Q12. ElastiCache Redis — Cluster Mode Enabled vs Disabled?

**Cluster Mode Disabled:**
- Single primary + up to 5 replicas.
- All nodes hold the FULL dataset.
- Simple: one endpoint for writes, reader endpoint for reads.
- Scales reads but NOT writes or storage.

**Cluster Mode Enabled:**
- Dataset sharded across up to **500 node groups (shards)**.
- Each shard: 1 primary + up to 5 replicas.
- Redis hash slots: 16,384 slots distributed across shards.
- 500 shards × 16,384 slots → massive horizontal scale.
- Clients need cluster-aware libraries (MOVED/ASK redirects).
- Multi-key commands only work within the same shard (or use hash tags `{user123}:session`).

```
Cluster Mode Enabled (3 shards):
Shard 0 [slots 0–5460]:     Shard 1 [slots 5461–10922]:  Shard 2 [slots 10923–16383]:
  Primary + 2 Replicas        Primary + 2 Replicas          Primary + 2 Replicas
```

**When to use Cluster Mode:** Dataset > single node memory, write throughput > single primary capacity, need horizontal write scaling.

---

### Q13. What is Amazon Redshift and how does it differ from RDS and Athena?

**Redshift** is a fully managed **columnar data warehouse** for analytics workloads.

**Architecture:**
- **Leader node:** Parses queries, builds execution plans, coordinates.
- **Compute nodes:** Execute query fragments in parallel, store data in columnar format.
- **Columnar storage:** Only reads the columns needed for a query — dramatically reduces I/O vs row-based storage.
- **Massively Parallel Processing (MPP):** Query distributed across all compute nodes.

| | RDS/Aurora | Redshift | Athena |
|---|---|---|---|
| Type | Transactional (OLTP) | Analytical (OLAP) | Serverless query |
| Storage | Row-based | Columnar | S3 (external) |
| Scale | Vertical + read replicas | Horizontal (add nodes) | Serverless (unlimited) |
| Query pattern | Short, high-frequency, CRUD | Complex aggregations, billions of rows | Ad-hoc queries on S3 |
| Cost model | Per hour (instance) | Per hour (cluster) | Per TB scanned |
| Joins | Cross-table | Star/snowflake schema optimized | SQL on S3 (Parquet optimal) |

**Redshift Serverless:** No cluster management. Auto-scales capacity. Pay per compute unit used. Best for intermittent analytics workloads.

**Redshift Spectrum:** Query S3 data directly from Redshift SQL without loading into Redshift. Hot data in Redshift, cold data in S3 — query both with the same SQL.

---

### Q14. TRICKY — What is the N+1 query problem and how do you solve it on AWS?

**The problem:**
```
GET /orders
  for each order:
    GET /order/{id}/user  ← N additional DB queries
    
= 1 + N queries (100 orders = 101 DB queries)
```

**Solutions:**

1. **Application-level fix:** Batch fetch with SQL JOIN or `WHERE id IN (...)`.

2. **Caching with ElastiCache:** Cache user profiles. First order resolves user from DB, subsequent orders hit cache.

3. **DynamoDB single-table design:** Store related items in the same table with composite keys. Query with `begins_with` on sort key to fetch parent + all children in one request.

4. **AppSync + BatchInvoke:** GraphQL resolvers can be batched — AWS AppSync supports batch Lambda resolvers (DataLoader pattern).

5. **Aurora with Data API:** Batch queries via the HTTP-based Data API for serverless architectures.

---

### Q15. DESIGN — Multi-region active-active database strategy for a global user-facing app?

```
Region: us-east-1 (Primary writes)    Region: eu-west-1 (Secondary)
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ Application                 │        │ Application                 │
│ ├── User profiles: DynamoDB ├──sync──► DynamoDB Global Tables      │
│ │   Global Tables            │        │ (multi-master, <1s repl.)   │
│ │                            │        │                             │
│ ├── Orders/Financial: Aurora├──repl──► Aurora Global DB            │
│ │   Global Database          │        │ (storage replication <1s)   │
│ │   (primary writes here)    │        │ Read from local, writes     │
│ │                            │        │ forwarded to primary        │
│ ├── Sessions: ElastiCache   ├──repl──► ElastiCache Global          │
│ │   Global Datastore         │        │ Datastore (Redis)           │
│ │                            │        │                             │
│ └── Media/Assets: S3 CRR   ├──repl──► S3 (CRR to eu-west-1)       │
└─────────────────────────────┘        └─────────────────────────────┘
         │                                         │
         └───────── Route 53 Latency Routing ──────┘
              → EU users go to eu-west-1
              → US users go to us-east-1
```

**Conflict resolution for DynamoDB Global Tables:**
- Default: last-writer-wins (timestamp-based).
- Custom: Lambda-based conflict detection using conditional writes + version numbers.
- For financial data: avoid multi-region writes to the same item. Use Aurora primary region for all writes, Global Table only for read-heavy user profiles.

**Aurora write forwarding caveat:** Write forwarding adds latency (round-trip to primary). For write-heavy EU workloads, consider Aurora multi-master (limited use) or route all writes explicitly to primary region.

---

### Q16. What is Amazon Neptune and when would you use it over DynamoDB or RDS?

**Neptune** is AWS's managed **graph database** supporting:
- **Property Graph** with Gremlin query language (Apache TinkerPop).
- **RDF graphs** with SPARQL query language.

**When graph databases win:**
- Data has **highly connected relationships** with arbitrary depth traversal.
- Query: "Find all friends of friends of John who also bought Product X" — trivial in Gremlin, nightmarish in SQL joins.
- Use cases: social networks, fraud detection graphs, knowledge graphs, recommendation engines, network topology analysis.

| | RDS | DynamoDB | Neptune |
|---|---|---|---|
| Data model | Relational | Key-value / document | Graph |
| Relationships | Foreign keys + JOINs | Denormalized or GSI | Native edges, traversal |
| Multi-hop queries | Expensive recursive CTEs | Multiple requests | Single graph traversal |
| Use case | OLTP, structured data | High-scale key-access | Connected data, traversals |

---

### Q17. What is Amazon QLDB and when would you use it?

**QLDB (Quantum Ledger Database):**
- Fully managed, serverless, **immutable ledger** database.
- Every change is cryptographically verifiable — you can prove the entire history of a record.
- **Journal-based:** writes are appended to an immutable journal. Current state derived from journal.
- **PartiQL** (SQL-compatible) query language.
- Not a blockchain — no decentralized consensus. AWS controls it but provides cryptographic proof of integrity.

**Use cases:** Banking transaction history, supply chain audit trail, healthcare record audit, insurance claims history.

**vs DynamoDB + DynamoDB Streams:** Streams capture changes but can be modified by someone with DB admin access. QLDB's journal cannot be altered by anyone — including the DB owner. That's the key difference.

---

### Q18. TRICKY — What is connection pooling and why does it matter for RDS in a Lambda-heavy architecture?

**The problem:**
- Lambda functions are stateless. Each cold start opens a new DB connection. Each warm invocation may reuse or open new connections.
- At 1,000 concurrent Lambda invocations → potentially 1,000 DB connections.
- RDS PostgreSQL max connections: ~860 for db.r5.large. Exceeding this causes `FATAL: sorry, too many clients already`.

**Solutions:**

1. **RDS Proxy (AWS managed):** Connection pooling layer. Lambda → RDS Proxy (thousands of connections) → RDS (hundreds of pooled connections). RDS Proxy maintains persistent connections to the DB and multiplexes Lambda connections over them.

2. **PgBouncer on EC2/ECS (self-managed):** Open-source connection pooler. More control over pooling modes (session, transaction, statement pooling). Transaction pooling = highest efficiency (connection returned to pool after each transaction, not session).

3. **Architecture change:** Move from Lambda → RDS to Lambda → SQS → ECS service → RDS. The ECS service maintains a stable connection pool.

**Connection pooling modes:**
- **Session pooling:** One DB connection per client session. Minimal benefit.
- **Transaction pooling:** Connection returned after each transaction. Best for Lambda (short transactions).
- **Statement pooling:** Connection returned after each SQL statement. Restricts multi-statement transactions.
