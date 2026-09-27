# AWS Interview Prep — Part 4: Messaging, Streaming & Serverless

> Covers: SQS, SNS, EventBridge, Kinesis, MSK, Lambda Deep Dive, API Gateway, Step Functions, AppSync

---

## Section 1: Messaging & Streaming

### Q1. What is the difference between SQS Standard and SQS FIFO queues?

**Answer:**

| Feature | Standard | FIFO |
|---|---|---|
| Ordering | Best-effort (not guaranteed) | Strict FIFO within message group |
| Delivery | At-least-once (duplicates possible) | Exactly-once processing |
| Throughput | Nearly unlimited | 300 msg/s without batching, 3,000 with batching |
| Deduplication | No | Yes (5-min dedup window) |
| Message groups | No | Yes (MessageGroupId) |
| Use case | High throughput, order doesn't matter | Financial tx, order processing, sequenced workflows |

**Message Group ID:** Within a FIFO queue, messages with the same `MessageGroupId` are processed in strict order. Different group IDs can be processed in parallel. This allows partial ordering — e.g., all messages for `order_id=123` are ordered, but orders 123 and 456 process in parallel.

**Deduplication ID:** If you send two messages with the same `MessageDeduplicationId` within 5 minutes, SQS discards the second. You can also use content-based deduplication (SHA-256 hash of body).

---

### Q2. TRICKY: What happens to a message if your Lambda consumer throws an exception? Walk through the full flow including DLQ.

**Answer:**

```
Message arrives in SQS
        │
        ▼
Lambda polls and receives message
(message becomes invisible — VisibilityTimeout starts)
        │
        ▼
Lambda throws exception
        │
        ▼
Lambda returns error to SQS event source mapping
        │
        ▼
VisibilityTimeout expires → message becomes VISIBLE again
        │
        ▼
Lambda retries (up to maxReceiveCount, e.g., 3 times)
        │
        ▼
After maxReceiveCount retries → message moved to DLQ
(Dead Letter Queue — SQS or SNS)
        │
        ▼
Alert on DLQ message count → investigate + replay
```

**Key settings:**
- `VisibilityTimeout`: How long the message is hidden after being received. Must be LONGER than your Lambda timeout. If Lambda takes 5 minutes, set visibility timeout to at least 6 minutes.
- `maxReceiveCount` (on the Redrive Policy): How many times a message can be received before going to DLQ. Set based on transient vs permanent failure patterns.
- `MessageRetentionPeriod`: How long messages stay in the queue (1 min to 14 days). DLQ should have LONGER retention than the source queue.

**TRICKY gotcha:** Lambda's SQS event source mapping reports a batch as failed if ANY message in the batch fails. All other messages in the batch are retried too, even successful ones — unless you use `reportBatchItemFailures` in your Lambda response to report partial failures.

---

### Q3. Explain SNS message filtering. How does it work and what are its limits?

**Answer:**

SNS subscription filter policies allow each subscriber to receive only the messages it cares about, based on **message attributes**.

**How it works:**
- Publisher adds message attributes (key-value pairs) to the SNS publish call.
- Each subscription has a **filter policy** (JSON) defining which attribute values it wants.
- SNS evaluates the policy before delivering to the subscriber.
- Messages not matching the filter policy are NOT delivered (and NOT charged for delivery).

```
SNS Topic: orders
├── SQS Queue A (filter: {"event_type": ["order_placed"]})
├── SQS Queue B (filter: {"event_type": ["order_shipped", "order_delivered"]})
└── Lambda C   (filter: {"event_type": ["order_cancelled"], "priority": ["high"]})

Message published with attributes:
  event_type = "order_placed"
  priority = "normal"

Result:
  → Queue A receives it (matches event_type)
  → Queue B does NOT receive it
  → Lambda C does NOT receive it
```

**Filter policy limits:**
- Up to 5 attribute conditions per filter policy.
- String matching: exact match, prefix match, anything-but, exists/not-exists.
- Numeric matching: =, >, <, between.
- Complex conditions possible with nested OR logic.

**Important:** Filter policies apply to **message attributes**, not the message body. To filter on body content, use EventBridge (which supports rich content-based filtering including body fields).

---

### Q4. DESIGN: Design an event-driven order processing system using SNS fan-out pattern.

**Answer:**

```
Customer Places Order
        │
        ▼
API Gateway → Lambda (Order Service)
        │
        ▼ Publish to SNS Topic: "order-events"
        │  Message attributes: {event_type: "order_placed", tier: "premium"}
        │
        ├──────────────────────────────────────────────┐
        │                     │                         │
        ▼                     ▼                         ▼
SQS: inventory-svc     SQS: payment-svc         SQS: notification-svc
Filter: all orders     Filter: all orders        Filter: all orders
        │                     │                         │
        ▼                     ▼                         ▼
Lambda: reserve        Lambda: charge card       Lambda: email/SMS
inventory              │                         customer
        │               ▼
        │          SQS: fraud-check (premium only)
        │          Filter: {tier: ["premium"]}
        │               │
        ▼               ▼
   DynamoDB          Fraud Lambda → approve/decline
   (inventory)

Each SQS queue:
├── Has its own DLQ
├── Has its own retry policy
└── Scales independently
```

**Benefits:**
- Services are fully decoupled — order service doesn't know about downstream consumers.
- Each consumer scales independently based on its own queue depth.
- Adding a new consumer (e.g., analytics) = add new SNS subscription, zero changes to publisher.
- DLQ per service ensures one service's failures don't affect others.

---

### Q5. EventBridge deep dive — explain event buses, rules, and EventBridge Pipes.

**Answer:**

**Event Buses:**
- **Default bus**: Receives events from AWS services automatically (EC2 state changes, S3 events, etc.)
- **Custom bus**: Your own applications publish events here.
- **Partner bus**: SaaS integrations (Shopify, Zendesk, Stripe, etc.) publish to partner buses you subscribe to.

**Rules:**
- Match events on the bus and route them to targets.
- Event pattern matching: JSON path-based filtering on any field in the event envelope.
- Up to 5 targets per rule. Targets include: Lambda, SQS, SNS, Step Functions, ECS, Kinesis, API destinations (HTTP endpoints), another event bus.
- Rules support **input transformation** — reshape the event before delivering to target.

**EventBridge Pipes (2022):**
- Point-to-point integrations with built-in filtering, enrichment, and transformation.
- Source → Filter → Enrichment (Lambda/API GW) → Target.
- Sources: SQS, DynamoDB Streams, Kinesis, Kafka, RabbitMQ, MSK.
- Reduces boilerplate Lambda glue code.

```
EventBridge Pipe example:
DynamoDB Stream (source)
    │
    ▼ Filter: only INSERT events for new users
    │
    ▼ Enrichment: Lambda adds user details from external API
    │
    ▼ Target: EventBridge custom bus → rules → multiple targets
```

**vs SNS:** EventBridge has richer filtering (content-based on body, not just attributes), built-in schema registry, and 20+ AWS service targets. SNS is simpler and higher throughput for basic fan-out.

---

### Q6. TRICKY: What is the difference between Kinesis Data Streams, Kinesis Data Firehose, and Kinesis Data Analytics?

**Answer:**

| Service | Purpose | Consumers | Retention | Processing |
|---|---|---|---|---|
| **Data Streams** | Real-time ingestion & streaming | Custom (Lambda, KCL app, Enhanced Fan-Out) | 1–365 days | You write processing logic |
| **Data Firehose** | Load streaming data to destinations | Fully managed delivery | No retention (buffered delivery) | Basic transform via Lambda |
| **Data Analytics** | Real-time SQL/Flink on streams | Reads from Streams or Firehose | No retention (processes live) | Managed Flink / SQL |

**Typical pipeline:**
```
Producers → Kinesis Data Streams → Kinesis Data Analytics (Flink)
                    │                         │
                    │                    Processed results
                    ▼                         │
             Kinesis Firehose ◄───────────────┘
                    │
                    ├── S3 (Parquet, partitioned by date)
                    ├── Redshift (via S3 COPY)
                    └── OpenSearch
```

**Firehose buffering:** Firehose does NOT deliver records one-by-one. It buffers data and delivers in batches:
- Buffer size: 1 MB to 128 MB
- Buffer interval: 60 to 900 seconds
- Whichever threshold is hit first triggers delivery.

**TRICKY:** Firehose is NOT a queue — it does NOT support replay. Once data is delivered (or the retention window passes), it's gone. Use Kinesis Data Streams if you need replay.

---

### Q7. How does Kinesis Enhanced Fan-Out work and when do you need it?

**Answer:**

**Standard (Shared) Throughput:**
- All consumers share 2 MB/s read throughput per shard.
- If you have 5 consumers on a shard, each gets ~400 KB/s.
- Uses polling (`GetRecords` API), adds 200ms+ latency.
- Cost: included in shard hour pricing.

**Enhanced Fan-Out (EFO):**
- Each registered consumer gets a **dedicated 2 MB/s pipe** per shard.
- 5 consumers × 2 MB/s = 10 MB/s total read throughput per shard.
- Uses push-based delivery (`SubscribeToShard` HTTP/2 server-push), latency ~70ms.
- Cost: additional per-consumer-shard-hour + per-GB charge.

```
Standard:
Shard (2 MB/s read)
├── Consumer A ─┐
├── Consumer B ─┤──── shared 2 MB/s ──► all get less
└── Consumer C ─┘

Enhanced Fan-Out:
Shard
├── Consumer A ──── dedicated 2 MB/s
├── Consumer B ──── dedicated 2 MB/s
└── Consumer C ──── dedicated 2 MB/s
```

**Use EFO when:**
- You have multiple independent consumers on the same stream (fraud detection + analytics + archival all reading the same stream).
- Latency matters (sub-100ms vs 200ms+).
- Consumer throughput requirements would exceed shared limits.

---

### Q8. MSK (Managed Kafka) — what does AWS manage and what do you still manage?

**Answer:**

**AWS manages:**
- ZooKeeper nodes (or KRaft mode) — provisioned, patched, HA.
- Broker OS patching and security updates.
- Broker health monitoring and replacement.
- Multi-AZ broker placement.
- CloudWatch metrics integration.
- Storage auto-expansion (MSK Serverless).

**You still manage:**
- Kafka broker instance type and count.
- Kafka version upgrades (you trigger them, AWS performs them with rolling restarts).
- Topic creation, partition count, replication factor.
- Consumer group management.
- Kafka Connect connectors (you deploy workers on EC2/ECS).
- Schema Registry (Confluent or Glue Schema Registry).
- MSK cluster security (TLS, SASL/SCRAM, IAM auth configuration).
- Storage provisioning (for provisioned clusters).

**MSK Serverless:** AWS manages everything including scaling. You pay per partition-hour and per GB. No broker management at all. Good for unpredictable workloads.

---

## Section 2: Lambda Deep Dive

### Q9. Lambda execution model — explain the lifecycle of an execution environment.

**Answer:**

```
Phase 1: INIT (cold start only)
├── Provision micro-VM (Firecracker)
├── Download + extract deployment package
├── Start language runtime
├── Run initialization code (outside handler):
│   ├── Import statements
│   ├── SDK client instantiation
│   ├── DB connection pool creation
│   └── Config loading
└── Duration: 100ms – 10s+ depending on runtime/size

Phase 2: INVOKE (every invocation)
├── Run your handler function
├── Return response
└── Duration: your code's execution time

Phase 3: SHUTDOWN (after idle period)
├── Lambda service sends SIGTERM
├── Runtime has 2 seconds to clean up
└── Environment is frozen (not billed while frozen)
```

**Key optimization:** Move everything possible to the INIT phase (outside the handler). SDK clients, DB connections, and config loaded once in INIT are reused across warm invocations.

```python
# BAD — creates new client every invocation
def handler(event, context):
    dynamodb = boto3.resource('dynamodb')  # Cold path every time
    table = dynamodb.Table('orders')
    return table.get_item(Key={'id': event['id']})

# GOOD — client created once in INIT, reused across invocations
import boto3
dynamodb = boto3.resource('dynamodb')  # Runs once (INIT)
table = dynamodb.Table('orders')

def handler(event, context):
    return table.get_item(Key={'id': event['id']})
```

---

### Q10. TRICKY: Lambda has a 15-minute timeout. Your job needs 30 minutes. What are your options?

**Answer:**

Lambda's hard limit is **15 minutes**. You cannot change this. Options:

1. **Step Functions + Lambda (most common):** Break the job into steps. Each step is a Lambda invocation. Step Functions orchestrates the chain with state passing between steps. Total workflow can run hours/days.

2. **Lambda chaining with SQS:** Lambda A processes part of the work, writes a checkpoint to DynamoDB, publishes a message to SQS with the checkpoint state. Lambda B picks up and continues from checkpoint.

3. **ECS Fargate task:** For true long-running jobs (no 15-min limit), use ECS Fargate. Submit via Lambda or EventBridge → ECS `RunTask` API. Fargate task runs until completion.

4. **AWS Batch:** Purpose-built for batch jobs. No timeout. Manages job queues, compute environments, dependencies. Lambda submits the job; Batch runs it on EC2/Fargate.

5. **Refactor into chunks:** If the work is parallelizable (e.g., process 1M records), split into chunks of 10K records each, fan out to multiple Lambda invocations in parallel using Step Functions Map state.

```
Step Functions solution:

EventBridge (scheduled)
        │
        ▼
Step Functions State Machine
├── State 1: Lambda (process records 0-10K) → checkpoint
├── State 2: Lambda (process records 10K-20K) → checkpoint
├── State 3: Lambda (process records 20K-30K) → checkpoint
└── ... continues until complete
```

---

### Q11. TRICKY: What is Lambda Destinations and how is it different from DLQ?

**Answer:**

**DLQ (Dead Letter Queue):**
- Only captures **failed** async invocations.
- Receives the original event payload.
- Supports SQS or SNS as destinations.
- No success path.
- Does NOT work for synchronous invocations.

**Lambda Destinations:**
- Captures both **success AND failure** for async invocations.
- Receives enriched result: original event + function response/error + metadata (request ID, duration, etc.).
- Supports SQS, SNS, EventBridge, another Lambda as destinations.
- Separate destinations for success and failure.

```
Async Lambda invocation
        │
        ├── SUCCESS ──► EventBridge (trigger downstream workflow)
        │                OR SQS (queue successful results)
        │
        └── FAILURE ──► SQS DLQ (for retry/investigation)
                        OR SNS (alert ops team)
                        OR EventBridge (trigger compensating action)
```

**Use DLQ for:** Simple failure capture when you only need the original payload.  
**Use Destinations for:** When you need success notifications, enriched failure data, or want to route to EventBridge for complex routing.

---

### Q12. DESIGN: Design a serverless image processing pipeline — users upload images, generate thumbnails in 3 sizes, extract metadata, and store results.

**Answer:**

```
User
 │
 ▼
Pre-signed S3 URL (generated by API Gateway + Lambda)
 │ (direct upload — no Lambda in the upload path)
 ▼
S3 Bucket: raw-images/
 │
 ▼ S3 Event Notification (ObjectCreated)
 │
 ▼
EventBridge (or direct S3→Lambda trigger)
 │
 ▼
Step Functions State Machine
 │
 ├── Parallel State:
 │   ├── Lambda: Generate thumbnail 100x100 → S3: thumbs/small/
 │   ├── Lambda: Generate thumbnail 400x400 → S3: thumbs/medium/
 │   └── Lambda: Generate thumbnail 800x800 → S3: thumbs/large/
 │
 ├── Lambda: Extract EXIF metadata (Rekognition for labels)
 │   └── Store metadata → DynamoDB: image-metadata table
 │
 └── Lambda: Update image record status → DynamoDB: images table
     │
     └── SNS: notify user "processing complete"
         └── WebSocket API Gateway → push notification to browser

Storage:
├── S3: raw-images/ (original, versioned)
├── S3: thumbs/ (processed thumbnails)
└── DynamoDB: image metadata + processing status

Cost optimizations:
├── Lambda layers for sharp/PIL image library (shared across functions)
├── S3 Intelligent Tiering on raw-images (older originals rarely accessed)
└── Step Functions Express Workflows (cheaper for high-volume, short workflows)
```

---

## Section 3: API Gateway

### Q13. What are the three types of API Gateway APIs and when do you use each?

**Answer:**

| | REST API | HTTP API | WebSocket API |
|---|---|---|---|
| Protocol | HTTP/REST | HTTP/REST | WebSocket (persistent) |
| Features | Full (usage plans, API keys, request/response mapping, caching) | Lightweight subset | Bidirectional real-time |
| Performance | Good | ~2x faster than REST | Real-time |
| Cost | Higher | ~71% cheaper than REST | Per message + connection |
| Lambda Proxy | Yes | Yes | Yes |
| VPC Link | Yes | Yes (v2) | No |
| Caching | Yes (per stage) | No | No |
| Use case | Complex APIs with throttling, keys, transformations | Simple proxies, microservices, lower cost | Chat, live updates, gaming |

**Key differences:**
- **HTTP API** doesn't support request/response mapping templates, API keys, or caching. Use REST API if you need these.
- **REST API** supports edge-optimized (CloudFront-backed) and regional deployments.
- **HTTP API** supports only regional deployments.
- **WebSocket API**: clients connect once, maintain persistent connection. Server can push messages to clients without polling.

---

### Q14. TRICKY: What is the API Gateway 29-second timeout? How do you work around it for long-running operations?

**Answer:**

API Gateway has a **hard maximum integration timeout of 29 seconds** for all integration types. This cannot be increased.

**Work-arounds for long-running operations:**

**Pattern 1: Async + Polling**
```
Client → POST /jobs → Lambda (starts job, returns 202 + job_id immediately)
                              │
                              └─► SQS → Worker Lambda (does the work)
                                         │
                                         └─► DynamoDB: jobs table (status updates)

Client → GET /jobs/{job_id} → Lambda → DynamoDB (poll for status)
(client polls every 5s)
```

**Pattern 2: Async + WebSocket callback**
```
Client connects to WebSocket API (persistent connection)
Client → REST API POST /jobs → Lambda → SQS → Worker
Worker completes → posts result to WebSocket connection ID
Client receives result pushed via WebSocket
```

**Pattern 3: Step Functions Express Workflows**
- Start execution via API Gateway.
- Return execution ARN immediately (202).
- Client polls Step Functions `DescribeExecution` API (or use callback pattern with .waitForTaskToken).

---

### Q15. Explain API Gateway caching. What are the cache key options and invalidation strategies?

**Answer:**

**Cache configuration:**
- Cache is per-stage (not per-deployment).
- Cache capacity: 0.5 GB to 237 GB.
- TTL: 0 to 3600 seconds (default 300s).
- Encryption at rest optional.
- Costs per hour based on cache size.

**Cache key:** By default, the full request path + query string parameters form the cache key. You can:
- Include specific **query string parameters** in the key (or exclude them).
- Include specific **HTTP headers** in the key (e.g., `Authorization` for per-user caching).
- Include **stage variables** in the key.

**TRICKY:** If you include the `Authorization` header in the cache key, different users get different cached responses — correct behavior for user-specific data. If you DON'T include it, all users get the same cached response — only safe for public, non-personalized data.

**Invalidation:**
- TTL expiry (automatic).
- Manual invalidation: `Cache-Control: max-age=0` header in request (if you enable **per-client cache invalidation** and grant IAM permission `execute-api:InvalidateCache`).
- Flush entire cache from console/API for a stage.

---

## Section 4: Step Functions

### Q16. Step Functions Standard vs Express Workflows — when to use each?

**Answer:**

| Feature | Standard | Express |
|---|---|---|
| Max duration | 1 year | 5 minutes |
| Execution model | Exactly-once | At-least-once |
| Execution history | Full (in console + API) | CloudWatch Logs only |
| Execution rate | 2,000/s | 100,000/s |
| Pricing | Per state transition | Per duration + invocation |
| Use case | Long-running, auditable workflows | High-volume, short, event processing |

**Standard Workflow:** Use for business processes — order fulfillment, user onboarding, data pipelines where you need audit trail and exactly-once semantics.

**Express Workflow:** Use for high-throughput event processing pipelines — IoT event routing, streaming data transformation, microservice orchestration where volume is high and duration is short.

**Synchronous Express:** Can return result inline to the caller (API Gateway integration). The caller waits up to 5 minutes for the result.

---

### Q17. Explain Step Functions integration patterns — SDK Integration, Optimistic vs .waitForTaskToken.

**Answer:**

**Request-Response (default):**
- Step Functions calls the service and immediately moves to the next state.
- Does NOT wait for the job to complete.
- Example: Start a Glue job, immediately proceed without waiting for result.

**Sync Integration (.sync:2):**
- Step Functions calls the service AND waits for it to complete.
- Polls the service internally.
- Supported for: ECS RunTask, Glue StartJobRun, SageMaker training, Athena queries, Lambda, etc.
- Example: `arn:aws:states:::glue:startJobRun.sync:2` — starts Glue job and waits.

**.waitForTaskToken:**
- Step Functions passes a unique `taskToken` to the worker.
- Step Functions PAUSES (no cost while waiting).
- Worker does its work (minutes, hours, days).
- Worker calls `SendTaskSuccess` or `SendTaskFailure` with the token.
- Step Functions resumes from where it paused.
- Perfect for: human approval steps, third-party API callbacks, long external processes.

```
.waitForTaskToken pattern:

Step Functions → Lambda (receives taskToken)
                      │
                      └─► Email/Slack with approval link
                                    │
                     Manager clicks Approve/Reject
                                    │
                                    ▼
                          Lambda callback handler
                          SendTaskSuccess(taskToken, result)
                                    │
                                    ▼
                     Step Functions resumes next state
```

---

### Q18. DESIGN: Design an order fulfillment workflow using Step Functions that handles payment, inventory, shipping, and failures with compensation.

**Answer:**

```
┌─────────────────────────────────────────────┐
│  Step Functions Standard Workflow            │
│                                              │
│  START                                       │
│    │                                         │
│    ▼                                         │
│  Validate Order (Lambda)                     │
│    │ fail → END (validation error)           │
│    │ pass                                    │
│    ▼                                         │
│  Charge Payment (Lambda)                     │
│  [catches PaymentFailed → Cancel Order]      │
│    │ fail → Compensate → END                 │
│    │ pass                                    │
│    ▼                                         │
│  Reserve Inventory (Lambda)                  │
│  [catches OutOfStock → Refund Payment]       │
│    │ fail → Refund Payment → Cancel → END    │
│    │ pass                                    │
│    ▼                                         │
│  Parallel State:                             │
│  ├── Notify Customer (SNS)                   │
│  └── Update Order Status (DynamoDB)          │
│    │                                         │
│    ▼                                         │
│  Create Shipment (Lambda → Shipping API)     │
│  [waitForTaskToken — external carrier]       │
│    │                                         │
│    ▼ (hours/days later — carrier callback)   │
│  Shipment Confirmed                          │
│    │                                         │
│    ▼                                         │
│  Wait State: 7 days                          │
│    │                                         │
│    ▼                                         │
│  Request Delivery Confirmation               │
│    │                                         │
│    ▼                                         │
│  Close Order → END                           │
│                                              │
│  Compensation (Saga Pattern):                │
│  PaymentRefund Lambda                        │
│  InventoryRelease Lambda                     │
│  OrderCancellation Lambda                    │
└─────────────────────────────────────────────┘
```

**Saga pattern:** Each successful step has a compensating transaction. If step N fails, Step Functions executes compensating actions for steps 1..N-1 in reverse order. This maintains eventual consistency across distributed services without distributed transactions.

---

## Section 5: AppSync

### Q19. What is AWS AppSync and when would you use it over API Gateway?

**Answer:**

AppSync is a **managed GraphQL service**. It handles the GraphQL layer — schema definition, resolvers, subscriptions — so you don't build a GraphQL server yourself.

**When to use AppSync over API Gateway:**
- Your clients need **GraphQL** (flexible queries, nested types, subscriptions).
- Real-time data via **GraphQL subscriptions** (WebSocket-based, managed by AppSync).
- **Offline sync** for mobile apps (Amplify DataStore — AppSync handles conflict resolution).
- **Multiple data sources** behind one endpoint (DynamoDB, Lambda, RDS via HTTP resolver, OpenSearch, HTTP endpoints).

**Resolver pipeline:**
```
Client GraphQL Query
        │
        ▼
AppSync (validates against schema)
        │
        ▼
Resolver (Velocity Template Language or JavaScript)
├── Before mapping template (transform request)
├── Data source call (DynamoDB, Lambda, RDS, HTTP)
└── After mapping template (transform response)
        │
        ▼
Client receives strongly-typed GraphQL response
```

**Real-time subscriptions:**
```
Client subscribes to: onOrderUpdated(orderId: "123")
        │
        ▼
AppSync maintains WebSocket connection
        │
Order update event (via Lambda mutation or DynamoDB trigger)
        │
        ▼
AppSync pushes update to all subscribed clients instantly
```

**vs API Gateway:** API Gateway is REST/HTTP — you define endpoints. AppSync is GraphQL — clients define the shape of their query. Use AppSync for data-driven apps with complex relationships; API Gateway for standard REST/HTTP APIs.

---

### Q20. TRICKY: What are AppSync caching, batching, and the N+1 problem?

**Answer:**

**N+1 Problem in GraphQL:**
```
Query: { orders { id customer { name email } items { product price } } }

Without batching:
- 1 query to get 100 orders
- 100 queries to get each customer (N queries for N orders) ← N+1
- 100 queries to get each order's items
= 201 database queries for one GraphQL request
```

**AppSync solutions:**

1. **Pipeline Resolvers:** Chain multiple resolver functions. Fetch all orders, then batch-fetch all referenced customers in one query instead of N queries.

2. **DataLoader pattern (Lambda resolvers):** Lambda resolver uses batching to group individual ID lookups into batch DynamoDB queries (`BatchGetItem`).

3. **AppSync Server-Side Caching:**
   - Enable per-resolver caching with a TTL.
   - Cache key is based on resolver arguments.
   - Eliminates redundant data source calls for identical queries.
   - Cache capacity: 0.5 GB to 237 GB (same as API Gateway caching).

4. **DynamoDB batch resolvers:** AppSync has native DynamoDB resolvers that can use `BatchGetItem` and `TransactGetItems` directly without Lambda.

---

## Section 6: Advanced Messaging Patterns

### Q21. DESIGN: Design a CQRS (Command Query Responsibility Segregation) architecture on AWS.

**Answer:**

```
WRITE SIDE (Commands):
Client → API Gateway → Lambda (Command Handler)
                            │
                            ├── Validate command
                            ├── Apply business rules
                            ├── Write to DynamoDB (write model)
                            └── Publish event to EventBridge
                                        │
                                        ▼
                                EventBridge Rule
                                        │
                        ┌───────────────┼───────────────┐
                        ▼               ▼               ▼
                  Lambda           Lambda          Lambda
               (Update read     (Update search  (Update analytics
                model: Aurora)  index: OpenSearch) model: Redshift)

READ SIDE (Queries):
Client → API Gateway → Lambda (Query Handler)
                            │
                            ├── Simple lookups → DynamoDB (read model)
                            ├── Search queries → OpenSearch
                            ├── Reports       → Aurora read replica
                            └── Analytics     → Redshift

Key principles:
├── Write model optimized for writes (normalized, DynamoDB)
├── Read models optimized for specific query patterns
├── Eventual consistency between write and read models
└── Each read model is independently scalable and schema-optimized
```

**Trade-offs:**
- Complexity increases significantly.
- Read models are eventually consistent (small lag after write).
- Multiple models to maintain and monitor.
- Only justified when read and write patterns have very different scaling requirements.

---

### Q22. TRICKY: SQS message visibility timeout vs message retention period — what happens if visibility timeout expires before processing completes?

**Answer:**

**Visibility Timeout:** When a consumer receives a message, it becomes invisible for the visibility timeout duration. If the consumer deletes the message before timeout expires → message gone. If consumer fails or timeout expires → message becomes visible again for another consumer.

**What actually happens when it expires mid-processing:**
1. Consumer A receives message, starts processing (visibility timeout = 30s).
2. Processing takes 45s (longer than timeout).
3. At 30s: message becomes visible again.
4. Consumer B picks up the SAME message and starts processing it.
5. Consumer A finishes at 45s and tries to delete the message.
6. Result: **message processed TWICE** (by A and B concurrently).

**Fix options:**
1. Set visibility timeout to 6× the maximum expected processing time.
2. Use `ChangeMessageVisibility` API to extend the timeout programmatically during long processing.
3. Design idempotent consumers so double-processing has no harmful effect.

**Message Retention Period:** Completely separate. This is how long the message stays in the queue if NOBODY processes it. After retention period (max 14 days), message is permanently deleted — even if nobody ever consumed it.

**TRICKY scenario:** Visibility timeout = 5 min, retention = 1 hour. If a consumer takes 6 minutes but only extends visibility once at 3 minutes (to 8 minutes), the message is safe. But if the consumer crashes at 5 min 30s without extending — the message reappears, consumer retries start.

---

*End of Part 4 — 22 Questions covering Messaging, Streaming, Lambda, API Gateway, Step Functions, AppSync*
