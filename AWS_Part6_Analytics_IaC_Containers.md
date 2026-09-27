# AWS Interview Prep — Part 6: Analytics, IaC, Containers Deep Dive & CI/CD

> Covers: Athena, Glue, EMR, Redshift, OpenSearch, QuickSight, CloudFormation, CDK, Terraform, EKS Networking, Service Mesh, GitOps, CodePipeline, CodeBuild, CodeDeploy

---

## Section 1: Analytics & Big Data

### Q1. Explain the AWS analytics stack — how do Glue, Athena, EMR, Redshift, and QuickSight fit together?

**Answer:**

```
Data Lake Architecture:
                                          BI / Reporting
Raw Data Sources                             │
    │                                  QuickSight
    ├── IoT Core → Kinesis Firehose    ├── Direct Query via SPICE
    ├── RDS → DMS → S3                 └── Connected to Redshift/Athena
    ├── APIs → Lambda → S3
    └── Logs → CloudWatch → Firehose
              │
              ▼
         S3 Data Lake
         ├── raw/           ← untouched source data
         ├── processed/     ← cleaned, transformed
         └── curated/       ← aggregated, query-optimized

         AWS Glue Data Catalog
         (Central metadata: databases, tables, schemas, partitions)
              │
    ┌─────────┼──────────────┐
    │         │              │
    ▼         ▼              ▼
 Athena     EMR         Redshift Spectrum
(Ad-hoc    (Complex      (Federate Redshift
 SQL on     Spark/Hadoop  queries to S3)
 S3)        workloads)

                    Redshift
                    (Data Warehouse — structured,
                     aggregated, fast dashboards)
```

**When to use each:**
- **Glue:** ETL jobs to transform raw → processed → curated. Crawlers auto-discover schema. Catalog shared with Athena, EMR, Redshift Spectrum.
- **Athena:** Ad-hoc SQL queries directly on S3. No infrastructure. Pay per query. Best for exploration.
- **EMR:** Complex batch processing (Spark, Hive, Presto, HBase). When Glue/Athena aren't enough — large-scale ML feature engineering, graph processing.
- **Redshift:** Structured analytical queries at scale with consistent performance. OLAP workloads, BI dashboards, data warehouse with defined schema.
- **QuickSight:** Business intelligence layer. Drag-and-drop dashboards, sharing with non-technical users, ML insights.

---

### Q2. Athena deep dive — what is partition projection and why does it matter?

**Answer:**

**Standard Athena partitioning:**
- Data in S3 structured as: `s3://bucket/logs/year=2024/month=01/day=15/`
- Glue Crawler discovers partitions and writes them to the Glue Catalog.
- Athena reads partition metadata from catalog to prune which S3 paths to scan.
- Problem: With thousands of partitions (daily data for years), crawler runs take a long time. `MSCK REPAIR TABLE` can be slow.

**Partition Projection:**
- Instead of reading partition metadata from Glue Catalog, Athena **calculates** the partition values from a configuration in the table definition.
- Athena knows: partitions follow pattern `year=YYYY/month=MM/day=DD` with a defined date range.
- No catalog lookup needed — Athena directly constructs S3 paths to read.
- Dramatically faster for time-series data with many partitions.

```sql
-- Table with partition projection
CREATE EXTERNAL TABLE logs (
    level string,
    message string
)
PARTITIONED BY (year string, month string, day string)
LOCATION 's3://my-bucket/logs/'
TBLPROPERTIES (
    'projection.enabled' = 'true',
    'projection.year.type' = 'integer',
    'projection.year.range' = '2022,2030',
    'projection.month.type' = 'integer',
    'projection.month.range' = '1,12',
    'projection.month.digits' = '2',
    'projection.day.type' = 'integer',
    'projection.day.range' = '1,31',
    'projection.day.digits' = '2',
    'storage.location.template' = 
        's3://my-bucket/logs/year=${year}/month=${month}/day=${day}/'
);
```

**Result:** Queries with `WHERE year=2024 AND month=01` go directly to the exact S3 prefix — no catalog round-trip, no slow partition discovery.

---

### Q3. Redshift — explain distribution styles, sort keys, and when to use each.

**Answer:**

**Distribution Styles (how rows are distributed across nodes):**

| Style | How | Use case |
|---|---|---|
| **AUTO** | Redshift chooses based on table size | Default — let Redshift decide |
| **EVEN** | Round-robin across all nodes | No clear join key, even distribution needed |
| **KEY** | All rows with same distkey value on same node | Tables frequently joined on a column (co-located joins) |
| **ALL** | Full copy of table on every node | Small dimension tables joined frequently |

**Join performance:** If two large tables are joined on a column, use `DISTSTYLE KEY` with the same distribution key on both tables. Rows with matching keys are on the same node → no network redistribution during join.

```sql
-- Orders: distribute by customer_id
CREATE TABLE orders (
    order_id bigint,
    customer_id bigint DISTKEY,
    amount decimal
) SORTKEY (order_date);

-- Customers: distribute by customer_id (same key = co-located join)
CREATE TABLE customers (
    customer_id bigint DISTKEY,
    name varchar(100)
);

-- This join is efficient — no data movement between nodes
SELECT c.name, SUM(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.name;
```

**Sort Keys:**
- **Compound sort key:** Most effective when queries filter/sort on all or leading columns. `SORTKEY(year, month, day)` — queries filtering on year are efficient; filtering on day only is not.
- **Interleaved sort key:** Treats each column equally. Effective when queries filter on any column independently. Higher maintenance cost (VACUUM more intensive).

---

### Q4. TRICKY: What is Redshift Spectrum vs Athena? When does one have an advantage?

**Answer:**

Both query S3 data using SQL. The key differences:

| | Redshift Spectrum | Athena |
|---|---|---|
| Requirement | Redshift cluster required | No cluster needed |
| Use case | Join S3 data with Redshift tables | Pure S3 querying |
| Performance | Spectrum compute layer (separate from cluster) | Presto-based distributed query |
| Cost | Spectrum: $5/TB scanned + cluster cost | $5/TB scanned, no cluster |
| Formats | Parquet, ORC, JSON, CSV, Avro | Same + Delta Lake, Iceberg, Hudi |
| Best for | Federated queries mixing hot (Redshift) + cold (S3) data | Ad-hoc exploration, no existing Redshift investment |

**Redshift Spectrum advantage:** When you have a Redshift data warehouse and want to extend queries to historical data in S3 without loading it. Single SQL query joins last 3 months (in Redshift) with last 3 years (in S3 via Spectrum).

**Athena advantage:** No cluster cost. True serverless. Better for exploration, data engineering pipelines that don't need Redshift.

---

### Q5. DESIGN: Design a real-time analytics pipeline — clickstream data from a website analyzed within 60 seconds and displayed on a live dashboard.

**Answer:**

```
Browser (JavaScript)
    │
    ▼ Events (clicks, page views, sessions)
API Gateway → Kinesis Data Streams
(low-latency ingestion, 1M+ events/sec)
    │
    ├──────────────────────────────────┐
    │                                  │
    ▼                                  ▼
Kinesis Data Analytics          Kinesis Firehose
(Apache Flink)                  (buffered delivery)
├── Sliding window (60s):        │
│   - Top 10 pages               ├── S3 (raw, Parquet)
│   - Conversion funnel          │   └── Athena / Glue queries
│   - Active users               │
│   - Error rates                └── OpenSearch
│         │                          └── Kibana live dashboard
│         ▼
│   Results every 30s
│         │
│         ├── DynamoDB (real-time counters)
│         │   └── API → QuickSight SPICE refresh
│         │
│         └── ElastiCache Redis (live leaderboard)
│             └── WebSocket API → Browser dashboard
│
└── Alert: if error rate > 5% → SNS → PagerDuty

60-second SLA:
├── Kinesis ingestion: <1s
├── Flink processing window: 30s sliding
├── DynamoDB write + API read: <5s
└── Dashboard refresh: 5s polling
Total: ~40s end-to-end ✓
```

---

### Q6. What is AWS Glue — explain crawlers, ETL jobs, and the Glue Data Catalog in detail.

**Answer:**

**Glue Data Catalog:**
- Central metadata repository for data lake.
- Contains: databases, tables, schemas, partition info, connection info.
- Shared with Athena, EMR, Redshift Spectrum — one catalog, multiple engines.
- Replacement for Apache Hive Metastore.

**Glue Crawlers:**
- Automatically scan data sources (S3, RDS, DynamoDB, JDBC) and infer schemas.
- Create/update table definitions in the Catalog.
- Run on-demand or scheduled.
- Support Parquet, ORC, JSON, CSV, XML, Avro.
- Detect schema changes and update partitions.

**Glue ETL Jobs:**
- Apache Spark under the hood (you write Python or Scala).
- Glue DynamicFrames: like Spark DataFrames but handle schema inconsistencies (missing fields, mixed types) gracefully.
- Can run on Glue-managed Spark cluster (pay per DPU-hour) or on your own EMR cluster.
- Glue Studio: visual ETL authoring (drag-and-drop).
- Glue DataBrew: data preparation without code (profile, clean, normalize).

**Glue Streaming ETL:** Reads from Kinesis Data Streams or Kafka, applies transformations, writes to S3/Redshift/etc. in micro-batches.

---

## Section 2: Infrastructure as Code

### Q7. CloudFormation basics — explain stacks, change sets, and stack policies.

**Answer:**

**Stack:** A collection of AWS resources managed as a single unit. Create/update/delete together. Defined by a CloudFormation template (YAML/JSON).

**Stack lifecycle:**
```
Template (YAML) → CloudFormation → Stack → AWS Resources
                                   │
                          Change Set (preview changes)
                                   │
                          Execute Change Set → Resources updated
```

**Change Sets:**
- Preview what changes CloudFormation will make BEFORE executing.
- Shows: resources added, modified, replaced, deleted.
- Critical: **Replacement** means the old resource is deleted and a new one created — downtime risk.
- Use change sets for ALL production updates.

**Stack Policies:**
- JSON policy that controls which resources can be updated during stack update.
- Protects critical resources (production databases) from accidental replacement.
```json
{
  "Statement": [{
    "Effect": "Deny",
    "Action": "Update:Replace",
    "Principal": "*",
    "Resource": "LogicalResourceId/ProductionDatabase"
  }]
}
```

**Nested Stacks:** Stack that creates other stacks (parent/child). Used to break large templates into reusable modules. Parent stack passes parameters to child stacks.

**Stack Sets:** Deploy the same stack across multiple accounts and regions from a single operation. Uses AWS Organizations for automated deployment.

---

### Q8. TRICKY: What is CloudFormation drift detection and what are its limitations?

**Answer:**

**Drift detection** identifies whether actual resource configurations differ from the CloudFormation template (i.e., someone manually changed something in the console or via CLI).

**What it detects:**
- Property changes: EC2 instance type changed manually.
- Tag added or removed.
- Security group rule added.

**Limitations:**
1. **Not all resources support drift detection.** Many resource types don't have drift detection support.
2. **Drift detection is point-in-time.** It doesn't continuously monitor — you must trigger it manually or schedule it.
3. **Doesn't auto-remediate.** Detecting drift doesn't fix it. You must decide to stack update (overwrite manual change) or import the change into the template.
4. **Doesn't detect deleted resources.** If someone deleted a CloudFormation-managed resource, drift detection may not catch it correctly.
5. **Performance:** Drift detection can be slow for large stacks (parallel detection with API throttling).

**Better approach for continuous compliance:** Use AWS Config rules to continuously detect configuration drift and trigger alerts or auto-remediation.

---

### Q9. AWS CDK vs CloudFormation vs Terraform — what are the trade-offs?

**Answer:**

| | CloudFormation | CDK | Terraform |
|---|---|---|---|
| Language | YAML/JSON | TypeScript, Python, Java, Go, C# | HCL |
| Learning curve | Low (declarative) | Medium (programming) | Medium |
| Abstraction | Low (resource-level) | High (constructs/patterns) | Medium |
| AWS integration | Native (no dependency) | Compiles to CloudFormation | AWS provider (API calls) |
| Multi-cloud | No | No | Yes |
| State management | AWS managed | AWS managed (via CFN) | State file (local or remote) |
| Drift detection | Built-in | Via CloudFormation | `terraform plan` shows drift |
| Community modules | CloudFormation modules | CDK construct library | Terraform Registry |

**CDK advantages:**
- Write IaC in a familiar programming language with IDE support, loops, conditionals.
- L1 constructs = raw CloudFormation. L2 constructs = sensible defaults. L3 constructs (patterns) = complete solutions in one line.
- Example: `new ApplicationLoadBalancedFargateService(stack, 'Service', {})` creates ALB + ECS Fargate + security groups + IAM roles in one construct.
- CDK compiles to CloudFormation templates — you get all CFN features (rollback, change sets, drift detection).

**Terraform advantages:**
- Multi-cloud (AWS + Azure + GCP + on-prem in one codebase).
- Large ecosystem of pre-built modules.
- `terraform plan` gives a clear diff before applying.
- Not dependent on CloudFormation service limits.

---

### Q10. TRICKY: What is the difference between CloudFormation `DependsOn`, `Ref`, and `GetAtt`? When can implicit dependencies fail?

**Answer:**

**`Ref`:** Returns the primary identifier of a resource (often the ID or name).
```yaml
SecurityGroupId: !Ref MySecurityGroup  # returns SG ID
```

**`GetAtt`:** Returns a specific attribute of a resource.
```yaml
LoadBalancerDNS: !GetAtt MyALB.DNSName  # returns DNS name
SubnetCidr: !GetAtt MySubnet.CidrBlock
```

**Implicit dependency:** When you use `Ref` or `GetAtt` to reference one resource in another, CloudFormation automatically knows to create the referenced resource first. This is an implicit dependency.

**`DependsOn`:** Explicit dependency declaration. Use when there's no `Ref`/`GetAtt` relationship but one resource must exist before another.

```yaml
# Example: EC2 instance needs internet access but doesn't directly reference IGW
MyEC2:
  Type: AWS::EC2::Instance
  DependsOn: VPCGatewayAttachment  # explicit — no Ref/GetAtt relationship
  Properties:
    ...
```

**When implicit dependencies FAIL (tricky!):**
- You reference a resource in `Metadata`, `DeletionPolicy`, or `UpdatePolicy` — CloudFormation does NOT track these as dependencies.
- You use a hardcoded ARN instead of `!GetAtt` — CloudFormation can't know there's a dependency.
- You reference a resource ID in a `UserData` script as a string — not tracked.

Always use `Ref`/`GetAtt` for cross-resource references. Use `DependsOn` as a supplement when implicit tracking isn't enough.

---

## Section 3: Containers Deep Dive

### Q11. Explain EKS networking with VPC CNI — how do pods get IPs?

**Answer:**

**AWS VPC CNI (Container Network Interface):**
- Each pod gets a **real VPC IP address** from the subnet — not a virtual overlay network IP.
- Pods are first-class VPC citizens: security groups, VPC flow logs, route tables all work on pod IPs.
- This is different from overlay networks (Flannel, Calico IPAM) where pod IPs are virtual and NAT'd at the node.

**How IP allocation works:**
```
Node: m5.xlarge (4 vCPUs)
├── Primary ENI: 10.0.1.4 (node's IP)
│   └── Can hold N secondary IPs (based on instance type)
├── Secondary ENI: 10.0.1.5 (attached by CNI)
│   ├── Pod-1: 10.0.1.20
│   ├── Pod-2: 10.0.1.21
│   └── Pod-3: 10.0.1.22
└── Secondary ENI: 10.0.1.6
    ├── Pod-4: 10.0.1.30
    └── Pod-5: 10.0.1.31
```

**Limits per instance type:** Each instance type supports a maximum number of ENIs and secondary IPs per ENI. `m5.xlarge`: 4 ENIs × 15 IPs = 60 max pods. This is a hard limit.

**Prefix delegation (addresses the scale limitation):**
- Instead of 1 IP per secondary assignment, assign a /28 prefix (16 IPs) per ENI slot.
- `m5.xlarge`: 4 ENIs × 2 prefixes × 16 IPs = 128 pods.
- Enables higher pod density without changing instance type.

**Pod-level Security Groups (via Security Groups for Pods):**
- Assign specific Security Groups to individual pods (not just nodes).
- Pod's traffic is routed through a dedicated branch ENI with the pod's SG.
- Requires Nitro instances.

---

### Q12. TRICKY: What is the difference between EKS managed node groups, self-managed node groups, and Fargate profiles?

**Answer:**

| | Managed Node Groups | Self-Managed | Fargate |
|---|---|---|---|
| Instance management | AWS manages launch, update, termination | You manage everything | No instances |
| OS patching | AWS provides updated AMIs, you apply | You patch | AWS manages |
| Draining on update | Automatic | Manual | N/A |
| Custom AMI | Limited (EKS-optimized base required) | Full control | No |
| GPU support | Yes | Yes | No |
| Windows nodes | Yes | Yes | No |
| Spot support | Yes | Yes | Yes (Fargate Spot) |
| Cost | EC2 instance pricing | EC2 instance pricing | Per vCPU/memory per second |
| Custom kernel | No | Yes | No |

**Managed Node Groups** are the recommended default — AWS handles the lifecycle. You still pay for EC2 instances.

**Self-managed** when you need: custom kernel parameters, custom OS (Bottlerocket, custom AMI), specific networking requirements.

**Fargate** when: you want zero node management, maximum isolation (each pod in its own micro-VM), or the workload is bursty and you don't want idle EC2 capacity.

**Key limitation of Fargate on EKS:** No DaemonSets support (DaemonSets need to run on every node — Fargate has no nodes). Monitoring agents, log collectors that use DaemonSets need alternative approaches (sidecar containers or node-level logging).

---

### Q13. Explain Kubernetes RBAC on EKS and how IAM integrates with it.

**Answer:**

**Kubernetes RBAC:**
- Kubernetes has its own RBAC system: Roles, ClusterRoles, RoleBindings, ClusterRoleBindings.
- Independent of IAM — a Kubernetes admin must explicitly grant permissions.

**EKS IAM-to-K8s integration via `aws-auth` ConfigMap:**
```yaml
# aws-auth ConfigMap in kube-system namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/NodeGroupRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789:role/DeveloperRole
      username: developer
      groups:
        - dev-team
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/admin-user
      username: admin
      groups:
        - system:masters
```

**Authentication flow:**
1. `kubectl` call → `aws eks get-token` generates a pre-signed STS URL as bearer token.
2. EKS API server validates the token → calls STS to identify the IAM identity.
3. EKS maps IAM identity to K8s username/groups via `aws-auth` ConfigMap.
4. K8s RBAC evaluates the K8s username/groups against RBAC rules.

**EKS Access Entries (new method, replaces aws-auth):** IAM-native way to manage EKS access without editing the ConfigMap. More secure, auditable via CloudTrail.

**IRSA (IAM Roles for Service Accounts):**
- Associates an IAM role with a Kubernetes Service Account.
- Pods using that Service Account automatically get IAM credentials via a projected volume (OIDC token).
- No more EC2 instance metadata credentials shared by all pods on the node.

---

### Q14. DESIGN: Design a GitOps pipeline for EKS using ArgoCD.

**Answer:**

```
Developer workflow:
Developer → Git push (feature branch)
                │
                ▼
        GitHub/CodeCommit PR
                │
                ▼
        CI Pipeline (CodePipeline / GitHub Actions)
        ├── Build Docker image
        ├── Run tests
        ├── Security scan (ECR image scanning / Snyk)
        ├── Push image to ECR: myapp:v1.2.3
        └── Update Helm chart values in GitOps repo:
            image.tag = v1.2.3

GitOps Repository (separate from app code):
├── /apps/myapp/dev/values.yaml    (image.tag: v1.2.3)
├── /apps/myapp/staging/values.yaml
└── /apps/myapp/prod/values.yaml

ArgoCD (running in EKS):
├── Watches GitOps repo every 3 minutes (or webhook)
├── Detects values.yaml changed (dev environment)
├── Compares desired state (Git) vs actual state (cluster)
├── Applies Helm chart with new image tag
└── Reports sync status

Promotion flow:
Dev auto-sync ──► manual PR to staging/values.yaml
                         │
                         ▼
               Staging sync (after PR merge)
                         │
                         ▼
               Manual approval + PR to prod/values.yaml
                         │
                         ▼
               Production sync

Benefits:
├── Git is the single source of truth
├── Full audit trail (git history = deployment history)
├── Easy rollback (revert commit = rollback deployment)
├── Consistent environments (no config drift)
└── Developers don't need kubectl access to production
```

---

## Section 4: CI/CD — Developer Tools

### Q15. Explain the AWS CI/CD toolchain — CodeCommit, CodeBuild, CodeDeploy, CodePipeline.

**Answer:**

| Service | Role | Equivalent |
|---|---|---|
| **CodeCommit** | Git repository hosting | GitHub/GitLab |
| **CodeBuild** | Managed build/test service | Jenkins / GitHub Actions runners |
| **CodeDeploy** | Deployment automation | Spinnaker, custom scripts |
| **CodePipeline** | CI/CD orchestration | Jenkins pipeline, GitHub Actions |
| **CodeArtifact** | Package repository (npm, Maven, PyPI) | Nexus, Artifactory |
| **ECR** | Docker container registry | Docker Hub, Quay |

**CodePipeline stages:**
```
CodeCommit (source)
    │
    ▼
CodeBuild (build + test)
├── buildspec.yml defines build steps
├── Build Docker image
├── Run unit tests
├── Static analysis (SonarQube integration)
└── Push to ECR
    │
    ▼
Manual Approval stage (optional)
    │
    ▼
CodeDeploy (deploy to target)
├── EC2/On-prem: rolling, blue/green
├── ECS: blue/green with ALB traffic shifting
└── Lambda: canary, linear, all-at-once
```

---

### Q16. CodeDeploy deployment strategies — explain each and when to use them.

**Answer:**

**For EC2/On-premises:**
- **In-place (Rolling):** Update instances in the current deployment group. Brief downtime per instance. Rollback = re-deploy old version.
- **Blue/Green:** New ASG with new instances + old ASG. Traffic shifted to new after health checks. Old kept for quick rollback, then terminated.

**For Lambda:**
- **AllAtOnce:** Instantly shift 100% traffic to new version. Fastest but highest risk.
- **Canary:** Shift X% of traffic to new version, wait N minutes, then shift remaining 100%. E.g., Canary10Percent5Minutes.
- **Linear:** Incrementally shift traffic by X% every N minutes. E.g., Linear10PercentEvery1Minute (10 minutes total for full shift).

**For ECS:**
- **Blue/Green with ALB:** New task set deployed alongside old. ALB test listener validates new. Traffic shifted (canary/linear/all-at-once). Old task set terminated.

```
ECS Blue/Green:

Before deployment:
ALB → Target Group Blue (v1 tasks) 100%

During deployment:
ALB → Target Group Blue (v1 tasks) 90%
ALB → Target Group Green (v2 tasks) 10%  (canary)

After validation:
ALB → Target Group Green (v2 tasks) 100%
Target Group Blue terminated
```

**Pre/Post traffic hooks:** Lambda functions run before and after traffic shifting. Use for smoke tests, DB migrations, cache warming. If hook fails, CodeDeploy automatically rolls back.

---

### Q17. TRICKY: CodeBuild buildspec.yml — explain phases, artifacts, caching, and environment variables.

**Answer:**

```yaml
version: 0.2

env:
  variables:
    APP_ENV: "production"
  parameter-store:                    # Fetch from SSM Parameter Store
    DB_PASSWORD: "/myapp/db/password"
  secrets-manager:                    # Fetch from Secrets Manager
    API_KEY: "myapp/api-key"

phases:
  install:                            # Install dependencies (cached)
    runtime-versions:
      nodejs: 18
    commands:
      - npm ci --cache .npm

  pre_build:                          # Before build (login, setup)
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)

  build:                              # Main build
    commands:
      - npm run test
      - npm run build
      - docker build -t $ECR_REGISTRY/myapp:$COMMIT_HASH .
      - docker push $ECR_REGISTRY/myapp:$COMMIT_HASH

  post_build:                         # After build (tag, notify)
    commands:
      - docker tag $ECR_REGISTRY/myapp:$COMMIT_HASH $ECR_REGISTRY/myapp:latest
      - docker push $ECR_REGISTRY/myapp:latest
      - printf '[{"name":"myapp","imageUri":"%s"}]' $ECR_REGISTRY/myapp:$COMMIT_HASH > imagedefinitions.json

artifacts:                            # What to pass to next pipeline stage
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json

cache:                                # Cache to S3 for faster subsequent builds
  paths:
    - '/root/.npm/**/*'
    - 'node_modules/**/*'
```

**Important CodeBuild limits:**
- Build timeout: 5 minutes to 8 hours (default 1 hour).
- Build environment: EC2 instances (various sizes). Lambda-based (smaller, faster startup).
- Concurrent builds: limited by service quota per account.
- Environment variables: sensitive values should come from SSM/Secrets Manager, NOT hardcoded.

---

### Q18. DESIGN: Design a complete blue/green deployment pipeline for a microservice on ECS with automated rollback.

**Answer:**

```
Source Stage:
GitHub push to main branch
        │
        ▼
CodePipeline triggers
        │
        ▼
CodeBuild Stage:
├── Run unit tests (fail → pipeline stops, no deployment)
├── Build Docker image
├── Run container security scan (ECR image scan)
│   └── fail if HIGH/CRITICAL CVEs found
├── Push to ECR with commit SHA tag
└── Generate: imagedefinitions.json, appspec.yml, taskdef.json
        │
        ▼
Manual Approval (optional for prod)
        │
        ▼
CodeDeploy Stage (ECS Blue/Green):
├── Deploy new task set (Green) alongside existing (Blue)
├── ALB test listener → validate Green on port 8080
│   └── Lambda hook: run smoke tests against test listener
│       ├── Success → proceed with traffic shift
│       └── Failure → CodeDeploy auto-rolls back (deletes Green)
├── Shift 10% traffic to Green (Canary10Percent5Minutes)
│   └── CloudWatch alarm monitors error rate
│       └── If errors > 1% → alarm triggers → CodeDeploy rollback
├── After 5 minutes → shift 100% to Green
└── Terminate Blue task set after 5 minute stability period
        │
        ▼
Post-deploy validation:
├── CloudWatch Synthetics canary runs E2E tests
├── If canary fails → SNS alert → manual rollback decision
└── Slack notification: deployment success/failure

Automatic rollback triggers:
├── Pre-traffic Lambda hook fails
├── Post-traffic Lambda hook fails
└── CloudWatch alarm in ALARM state during deployment window
```

---

## Section 5: OpenSearch & ElastiSearch

### Q19. What is Amazon OpenSearch Service and when do you use it over Athena?

**Answer:**

OpenSearch (formerly Elasticsearch Service) is a managed search and analytics engine based on OpenSearch/Elasticsearch.

**Use OpenSearch when:**
- You need **full-text search** (fuzzy matching, tokenization, relevance scoring).
- **Sub-second query response** on large datasets.
- **Log analytics** with Kibana/OpenSearch Dashboards for visualization.
- **Time-series data** with dense indexing.
- **Geospatial queries** (find all locations within X km).
- **Anomaly detection** (built-in ML-based anomaly detection).

**Use Athena when:**
- Ad-hoc SQL on structured data in S3.
- You don't need full-text search.
- Cost optimization is priority (no always-on cluster).
- Data is in Parquet/ORC format (efficient columnar scans).

**OpenSearch in a log analytics pipeline:**
```
CloudWatch Logs → Kinesis Firehose → OpenSearch
                                          │
                                    OpenSearch Dashboards
                                    (Kibana-compatible)
                                    ├── Log search
                                    ├── Error rate graphs
                                    ├── Alerting
                                    └── Anomaly detection
```

**Index management:** ISM (Index State Management) automatically moves indices through lifecycle:
- Hot (active writes, SSD): last 7 days
- Warm (read-only, less expensive): 7-90 days
- Cold (infrequent access): 90+ days
- Delete: after 1 year

---

### Q20. TRICKY: What is EMR Serverless vs EMR on EC2 vs EMR on EKS?

**Answer:**

| | EMR on EC2 | EMR on EKS | EMR Serverless |
|---|---|---|---|
| Infrastructure | You manage EC2 cluster | Spark runs as EKS pods | No cluster management |
| Scaling | Manual + auto-scaling | K8s scaling | Fully automatic |
| Cost | EC2 + EMR premium | EC2 + EKS + EMR | Per vCPU/GB-second |
| Pre-warming | Cluster startup ~10 min | Pod scheduling | Auto pre-initialized capacity |
| Customization | Full (bootstrap actions, custom AMI) | K8s configs | Limited |
| Idle cost | Yes (cluster up) | Partial | Zero (no jobs = no charge) |

**EMR Serverless** is the future direction — no cluster management, automatic scaling, pay only for actual job compute. Best for intermittent batch jobs.

**EMR on EKS** is for organizations already heavily invested in Kubernetes — run Spark jobs alongside other K8s workloads on the same cluster. Operational complexity: must manage the EKS cluster yourself.

**EMR on EC2** is still dominant for complex Spark workloads needing:
- Custom AMIs/bootstrap actions.
- Specific instance types (GPU, high memory).
- Persistent clusters for interactive analysis (Jupyter notebooks on EMR).

---

*End of Part 6 — 20 Questions covering Analytics, IaC, Containers, CI/CD*
