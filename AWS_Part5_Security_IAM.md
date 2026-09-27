# AWS Interview Prep — Part 5: Security, IAM, Compliance & Governance

> Covers: IAM deep dive, KMS, ACM, Secrets Manager, WAF, Shield, GuardDuty, Macie, Security Hub, AWS Organizations, SCPs, Control Tower, Detective, Inspector, Network Firewall

---

## Section 1: IAM Deep Dive

### Q1. What are the different types of IAM policies and how do they differ?

**Answer:**

| Policy Type | Attached To | Use Case |
|---|---|---|
| **Identity-based** | IAM User, Group, Role | Grant permissions to an identity |
| **Resource-based** | AWS resource (S3, SQS, Lambda, etc.) | Grant cross-account access or allow services to be invoked |
| **Permission Boundary** | IAM User or Role | Max permissions an identity can have (ceiling, not grant) |
| **SCP (Service Control Policy)** | AWS Account or OU | Max permissions for entire account (Organizations) |
| **Session Policy** | Temporary session (AssumeRole) | Restrict permissions for a specific session |
| **ACL (Access Control List)** | S3, VPC | Legacy cross-account, not IAM JSON format |

**Permission Boundary example:**
```
Developer has IAM policy: AdministratorAccess (can do anything)
Developer has Permission Boundary: AllowedServicesOnly (only EC2, S3, Lambda)

Result: Developer can only use EC2, S3, Lambda — even though their policy says AdministratorAccess
The boundary is the ceiling, the policy is the request — both must allow.
```

**Practical use for Permission Boundaries:** Allow developers to create IAM roles for their Lambda functions, BUT prevent them from creating roles with more permissions than they themselves have (stops privilege escalation).

---

### Q2. TRICKY: What is the difference between `aws:SourceIp`, `aws:VpcSourceIp`, and `aws:SourceVpc` condition keys?

**Answer:**

| Condition Key | Matches | Works with |
|---|---|---|
| `aws:SourceIp` | Client's public IP address | Requests from outside AWS (internet) |
| `aws:VpcSourceIp` | Client's private IP inside VPC | Requests through VPC endpoints |
| `aws:SourceVpc` | VPC ID | Restricts to traffic from a specific VPC |
| `aws:SourceVpce` | VPC Endpoint ID | Restricts to a specific VPC endpoint |

**The trap:**  
If an EC2 instance inside a VPC accesses S3 via a **VPC Gateway Endpoint**, the request does NOT have a public IP — `aws:SourceIp` will NOT match. You must use `aws:VpcSourceIp` or `aws:SourceVpc`.

```json
// WRONG — won't work for VPC endpoint traffic
{
  "Condition": {
    "IpAddress": {"aws:SourceIp": "10.0.0.0/8"}
  }
}

// CORRECT — for private VPC traffic through VPC endpoint
{
  "Condition": {
    "StringEquals": {"aws:SourceVpc": "vpc-12345678"}
  }
}

// MOST RESTRICTIVE — requires specific endpoint
{
  "Condition": {
    "StringEquals": {"aws:SourceVpce": "vpce-12345678"}
  }
}
```

---

### Q3. TRICKY: A Lambda function has an execution role with S3 full access. The S3 bucket policy denies all access except from a specific VPC endpoint. The Lambda is in a VPC with that endpoint. Will it work?

**Answer:**

**Yes, BUT only if the Lambda is in the VPC that has the endpoint AND the bucket policy references the correct endpoint ID or VPC ID.**

Evaluation:
1. Lambda identity policy: allows S3 (identity-based policy allows)
2. Bucket resource policy: denies unless `aws:SourceVpce = vpce-xxxxx`
3. Lambda in the VPC uses the VPC Gateway Endpoint for S3 traffic
4. The request arrives at S3 with the VPC endpoint context → condition matches → ALLOW

**If Lambda is NOT in a VPC:** Traffic goes via public internet → `aws:SourceVpce` condition has no value → bucket policy denies → **403 AccessDenied**.

**If Lambda is in a VPC but subnet has no VPC endpoint route:** Traffic goes via NAT Gateway → public internet → same 403 result.

**Key lesson:** Resource policies with VPC endpoint conditions require the exact network path to go through that endpoint.

---

### Q4. Explain IAM Roles Anywhere and how it extends IAM to workloads outside AWS.

**Answer:**

IAM Roles Anywhere allows **on-premises servers, containers, or applications** to assume IAM roles using **X.509 certificates** instead of long-term access keys.

**How it works:**
1. You create a **Trust Anchor** in IAM Roles Anywhere referencing your Certificate Authority (CA) — either AWS Private CA or your own CA.
2. You create a **Profile** that maps certificate attributes to IAM roles.
3. On-premises workload: uses the **IAM Roles Anywhere SDK** (or AWS signing helper) to request temporary credentials.
4. The tool signs the request with the workload's private key + sends the certificate chain.
5. IAM Roles Anywhere validates the certificate against the trust anchor.
6. Returns temporary STS credentials (AccessKeyId, SecretAccessKey, SessionToken).
7. Credentials are short-lived (default 1 hour, max 12 hours).

**Why this matters:** Eliminates long-lived IAM access keys on servers. No more rotating keys in configuration files. Certificates are managed by your PKI infrastructure.

---

### Q5. What is ABAC (Attribute-Based Access Control) in IAM? How does it differ from RBAC?

**Answer:**

**RBAC (Role-Based Access Control) — traditional:**
- Create a separate IAM role for each team/permission set.
- `developer-team-a-role`, `developer-team-b-role`, etc.
- Scales poorly: 100 teams = 100 roles.
- Policy changes require updating multiple roles.

**ABAC (Attribute-Based Access Control):**
- Use **tags** on both IAM principals AND resources.
- Single policy that uses `aws:RequestedRegion`, `aws:PrincipalTag`, `aws:ResourceTag` condition keys.
- Access granted when tags match — no need for separate roles.

```json
// ABAC policy: developers can only access resources tagged with their team
{
  "Effect": "Allow",
  "Action": ["ec2:*", "rds:*"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}"
    }
  }
}
```

With this policy:
- Developer with tag `Team=payments` can only access EC2/RDS tagged `Team=payments`.
- Developer with tag `Team=orders` can only access resources tagged `Team=orders`.
- ONE policy, unlimited teams — just tag resources and principals correctly.

**ABAC scales to thousands of teams without policy explosion.**

---

### Q6. DESIGN: Design a cross-account access pattern where Account A's application needs to read from Account B's S3 bucket and write to Account B's DynamoDB table.

**Answer:**

```
Account A                                Account B
─────────────────────────────────────────────────────────
Lambda Function                         S3 Bucket
├── Execution Role: LambdaExecRole       ├── Bucket Policy: Allow Account A role
│   └── inline policy:                  │
│       sts:AssumeRole on               DynamoDB Table
│       arn:aws:iam::B:role/CrossAcctRole├── Resource Policy: (DynamoDB doesn't
│                                       │   have resource-based policies —
│                                       │   handled via role policy in B)
│
│ Runtime:
│   1. Lambda calls STS AssumeRole
│      → arn:aws:iam::B:role/CrossAcctRole
│
│   2. Gets temporary credentials for Account B
│
│   3. Uses those credentials to:
│      ├── s3:GetObject on bucket in Account B
│      └── dynamodb:PutItem on table in Account B

Account B: CrossAcctRole
├── Trust Policy: Allow Account A (arn:aws:iam::A:role/LambdaExecRole)
│   to assume this role
└── Permission Policy:
    ├── s3:GetObject on arn:aws:s3:::bucket-in-b/*
    └── dynamodb:PutItem on arn:aws:dynamodb:region:B:table/TableName
```

**Two things that must BOTH be true:**
1. Account A's role must have permission to call `sts:AssumeRole` on Account B's role.
2. Account B's role trust policy must allow Account A's role as a principal.

**For S3 (optional third layer):** Account B's bucket policy can also explicitly allow Account A's role for defense-in-depth. This is required if Account B uses SCPs that would otherwise block cross-account access.

---

## Section 2: KMS & Encryption

### Q7. What is the difference between AWS KMS, AWS CloudHSM, and AWS Private CA?

**Answer:**

| | KMS | CloudHSM | AWS Private CA |
|---|---|---|---|
| Purpose | Key management + encryption | Key storage in dedicated HSM | Private certificate authority |
| Key storage | Multi-tenant HSM (FIPS 140-2 L2) | Single-tenant HSM (FIPS 140-2 L3) | N/A (issues certs, uses KMS/HSM for private key) |
| You manage HSM | No | Yes (initialization, partition) | No |
| Key type | Symmetric + Asymmetric | All types | RSA/ECDSA for certs |
| Price | Per API call + per key | ~$1.45/hr per HSM + key fees | Per cert issued |
| Use case | App-level encryption, service integrations | Regulatory (PCI, strict FIPS L3), TLS offload | Internal PKI, mTLS, IoT certificates |

**KMS key types:**
- **Symmetric (AES-256-GCM):** Most common. Used by all AWS services for encryption at rest. Key never leaves KMS.
- **Asymmetric (RSA, ECC):** Public key downloadable, private key in KMS. Used for digital signing, certificate authorities.
- **HMAC:** For message authentication codes.

**CloudHSM use cases:** Oracle TDE (Transparent Data Encryption) with customer-managed keys, strict compliance requiring L3 FIPS, financial signing operations requiring physical key control.

---

### Q8. TRICKY: What happens when you delete a KMS key? Can data be recovered?

**Answer:**

**KMS key deletion is a 2-phase process:**
1. You schedule deletion with a **waiting period** (7–30 days, you choose). During this period:
   - Key is disabled (cannot be used for encryption/decryption).
   - All attempts to use the key return `KMSInvalidStateException`.
   - You can CANCEL the deletion during this window.
2. After the waiting period: **key is permanently deleted**. This is irreversible.

**After deletion:**
- Any data encrypted with that key is **permanently inaccessible**.
- EBS volumes encrypted with that key cannot be mounted.
- S3 objects encrypted with SSE-KMS using that key cannot be decrypted.
- RDS instances encrypted with that key cannot be restored from backups.

**Best practices:**
- Never delete CMKs unless you are CERTAIN no data encrypted with them exists.
- Before deleting: search CloudTrail for recent `Decrypt` calls using the key.
- Use KMS key policies to restrict who can schedule deletion.
- Set a CloudWatch alarm for key deletion events.
- Disable the key first (fully reversible) and wait 30+ days to ensure nothing breaks.

---

### Q9. Explain S3 server-side encryption options — SSE-S3, SSE-KMS, SSE-C, and DSSE-KMS.

**Answer:**

| Option | Key managed by | Key stored | Audit | Per-object key | Use case |
|---|---|---|---|---|---|
| **SSE-S3** | AWS | S3 (opaque) | No CloudTrail | No | Default, no compliance requirement |
| **SSE-KMS** | AWS or Customer | KMS | Yes (CloudTrail) | Optional (DEK per object) | Compliance, audit trail, access control |
| **SSE-C** | Customer | Client provides per-request | No | Yes | You control keys, never give to AWS |
| **DSSE-KMS** | AWS or Customer | KMS | Yes | Yes | Dual-layer (two independent encryptions, NIST guidelines) |

**SSE-KMS details:**
- S3 calls `GenerateDataKey` on your KMS CMK for each object (or reuses DEK).
- The encrypted DEK is stored with the object.
- On download: S3 calls KMS `Decrypt` to get the plaintext DEK, uses it to decrypt the object.
- You can use KMS key policies to restrict who can access objects regardless of S3 bucket policy.
- Adds KMS API call overhead and cost — at scale (millions of objects), use SSE-S3 unless you need the audit trail.

**SSE-C:** You must supply the encryption key with every PUT and GET request. AWS uses it, then discards it. If you lose the key, the data is inaccessible.

---

## Section 3: Network Security

### Q10. Explain AWS Shield Standard vs Shield Advanced in detail.

**Answer:**

**Shield Standard (free for all AWS customers):**
- Automatic protection against most common DDoS attacks.
- Defends against: SYN floods, UDP reflection, volumetric attacks at Layer 3/4.
- Always-on detection and automatic mitigation.
- Protects: EC2, ELB, CloudFront, Route 53.
- No visibility into attacks, no support.

**Shield Advanced ($3,000/month per organization + data transfer fees):**
- Enhanced Layer 3/4/7 protection.
- **DRT (DDoS Response Team):** 24/7 access to AWS experts during active attacks.
- **Cost protection:** AWS credits compute and data transfer costs incurred during DDoS attacks.
- **Advanced attack visibility:** Real-time metrics in CloudWatch, attack forensics in Shield console.
- **Proactive engagement:** DRT contacts you when they detect anomalies targeting your resources.
- **Protects:** EC2, ELB, CloudFront, Route 53, Global Accelerator, Elastic IP.
- Works with WAF for Layer 7 protection — Shield Advanced includes WAF charges for protected resources.

**Shield Advanced architectural requirement:** Resources must be explicitly added to Shield Advanced protection. It doesn't automatically protect all your resources — you choose which to protect.

---

### Q11. GuardDuty vs Macie vs Inspector vs Detective vs Security Hub — what does each do?

**Answer:**

| Service | What it does | Data sources | Output |
|---|---|---|---|
| **GuardDuty** | Threat detection — finds malicious activity | VPC Flow Logs, CloudTrail, DNS logs, S3 logs, EKS audit logs | Findings (severity, type, affected resource) |
| **Macie** | PII/sensitive data discovery in S3 | S3 object content | Data classification findings, PII alerts |
| **Inspector** | Vulnerability scanning of workloads | EC2 OS packages, Lambda code, ECR images | CVE findings with CVSS scores and fix guidance |
| **Detective** | Security investigation + root cause analysis | GuardDuty findings, CloudTrail, VPC Flow Logs | Visual investigation graphs, IP behavior, entity timelines |
| **Security Hub** | Aggregates findings from all security services | GuardDuty, Macie, Inspector, Config, IAM Access Analyzer, 3rd party | Normalized findings, compliance scores, cross-account aggregation |

**Typical workflow:**
```
1. GuardDuty detects: EC2 instance communicating with known C&C server
2. Finding goes to Security Hub (aggregated + prioritized)
3. Security Hub → EventBridge → Lambda (auto-remediation: isolate instance)
4. Security analyst opens Detective to investigate:
   - What other instances communicated with this IP?
   - What IAM role did the instance use?
   - Timeline of API calls from this instance
5. Inspector confirms: EC2 had unpatched CVE that enabled the compromise
6. Macie scans to determine if S3 data was exfiltrated
```

---

### Q12. TRICKY: What is IAM Access Analyzer and how does it find unintended access?

**Answer:**

IAM Access Analyzer uses **automated reasoning** (formal mathematical proofs) to analyze resource-based policies and identify resources that are accessible from outside your zone of trust.

**What it scans:** S3 buckets, IAM roles (external trust), KMS keys, Lambda functions, SQS queues, Secrets Manager secrets, SNS topics.

**Zone of trust:** You define it as your AWS account or AWS Organization. Access Analyzer flags any resource accessible from OUTSIDE this zone.

**Types of findings:**
- **External access:** S3 bucket publicly accessible, IAM role trusted by external account.
- **Unused access (new feature):** IAM roles/users that haven't used certain permissions in 90 days — right-size permissions.
- **Policy validation:** Checks policies for errors, security warnings, suggestions.

**TRICKY scenario:** You have an S3 bucket policy that says "allow access from arn:aws:iam::999999999999:root" — that's an external account. Access Analyzer flags this as external access finding even if intentional. You must acknowledge/archive findings that are intentional.

**Policy generation:** Access Analyzer can generate least-privilege policies by analyzing CloudTrail activity for an IAM entity — shows you exactly what actions were used and generates a tight policy.

---

## Section 4: AWS Organizations & Governance

### Q13. AWS Organizations — explain the structure and what you can control with SCPs.

**Answer:**

**Organization Structure:**
```
Management Account (root)
└── Root OU
    ├── OU: Security (audit account, log archive account)
    ├── OU: Infrastructure (shared services, network hub account)
    ├── OU: Workloads
    │   ├── OU: Production
    │   │   ├── Account: App-Prod
    │   │   └── Account: DB-Prod
    │   └── OU: Development
    │       └── Account: App-Dev
    └── OU: Sandbox
        └── Account: Experiments
```

**SCPs (Service Control Policies):**
- Applied to OUs or individual accounts.
- Act as a **permission ceiling** — they don't grant permissions, they define the max allowed.
- Even the account root user is restricted by SCPs.
- SCPs do NOT affect the management account.
- Inherited: SCPs on a parent OU apply to all child OUs and accounts.

**Common SCP examples:**

```json
// Prevent disabling CloudTrail
{
  "Effect": "Deny",
  "Action": ["cloudtrail:DeleteTrail", "cloudtrail:StopLogging"],
  "Resource": "*"
}

// Restrict to specific regions only
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
    }
  }
}

// Prevent leaving the organization
{
  "Effect": "Deny",
  "Action": "organizations:LeaveOrganization",
  "Resource": "*"
}
```

---

### Q14. Explain AWS Control Tower and Landing Zone. What does it automate?

**Answer:**

**Landing Zone:** A well-architected, multi-account AWS environment following AWS best practices. It's a concept/architecture pattern, not a single service.

**AWS Control Tower:** AWS's managed service that automates the setup and governance of a multi-account Landing Zone.

**What Control Tower sets up automatically:**
1. **Management account** with billing access.
2. **Log Archive account**: Centralized S3 bucket for all CloudTrail and Config logs from all accounts.
3. **Audit account**: Security team access, aggregated GuardDuty/Security Hub findings.
4. **SSO (AWS IAM Identity Center)**: Single sign-on for all accounts with permission sets.
5. **Guardrails (preventive + detective)**: Pre-built SCPs (preventive) and Config rules (detective).
6. **Account Factory**: Self-service account vending machine. Teams request accounts → Control Tower provisions them with baseline configuration.

**Guardrail types:**
- **Preventive (SCP-based):** Block disabling CloudTrail, block leaving org, region restrictions.
- **Detective (Config rule-based):** Alert if S3 bucket is public, if MFA is not enabled on root, if EBS volumes are unencrypted.
- **Mandatory:** Always enabled (you can't turn them off).
- **Strongly Recommended:** Best practice, you can enable/disable.
- **Elective:** Optional.

---

### Q15. TRICKY: What is AWS RAM (Resource Access Manager) and what resources can be shared?

**Answer:**

AWS RAM allows you to share **AWS resources** with other AWS accounts or OUs within your Organization **without creating copies**.

**Shareable resources include:**
- Transit Gateway (share TGW to spoke accounts — central network model)
- Subnets (share VPC subnets to other accounts — centralized VPC model)
- Route 53 Resolver Rules
- License Manager configurations
- AWS Glue Data Catalog
- CodeBuild projects
- Resource Groups
- EC2 Capacity Reservations

**Centralized VPC sharing pattern (very common for architects):**
```
Network Account (owns VPC)
├── VPC: 10.0.0.0/16
│   ├── Shared Private Subnet 10.0.1.0/24 → shared to App Account A
│   ├── Shared Private Subnet 10.0.2.0/24 → shared to App Account B
│   └── Shared Private Subnet 10.0.3.0/24 → shared to App Account C

App Account A:
├── Launches EC2 instances INTO the shared subnet
├── Can see only its own resources in the subnet
├── Cannot see Account B's resources in the adjacent subnet
└── Network account controls all routing, NACLs, gateways

Benefits:
├── One VPC to manage (vs one VPC per account)
├── Centralized Direct Connect / TGW attachment
├── Consistent network policies
└── No VPC peering needed between accounts
```

---

## Section 5: Advanced Security Patterns

### Q16. DESIGN: Design a security architecture for a PCI-DSS compliant payment processing system on AWS.

**Answer:**

```
Cardholder Data Environment (CDE) — Isolated Account
┌─────────────────────────────────────────────────────┐
│                                                       │
│  Network Layer:                                       │
│  ├── Dedicated VPC (no shared subnets)                │
│  ├── Private subnets only (no public IPs)            │
│  ├── AWS Network Firewall (IPS signatures)           │
│  ├── VPC Flow Logs → S3 (1-year retention)           │
│  └── No internet access (Direct Connect to PCI zone) │
│                                                       │
│  Compute:                                             │
│  ├── EKS on dedicated instances (no Fargate for PCI) │
│  ├── EC2 instances: IMDSv2 required                  │
│  ├── No persistent storage on instances              │
│  └── Inspector: continuous CVE scanning              │
│                                                       │
│  Data:                                                │
│  ├── Aurora PostgreSQL: encrypted (KMS CMK)          │
│  │   └── Only stores tokenized card data             │
│  ├── Tokenization service: actual PANs → tokens      │
│  │   (uses Nitro Enclaves for PAN processing)        │
│  ├── KMS: Customer-managed keys, CloudHSM backed     │
│  └── No card data in logs (scrubbed at Lambda layer) │
│                                                       │
│  Access Control:                                      │
│  ├── Zero standing access: all access via            │
│  │   AWS SSO with time-limited sessions              │
│  ├── MFA required for all human access               │
│  ├── Separate role for each microservice (IRSA)      │
│  └── Secrets Manager: DB creds, API keys             │
│                                                       │
│  Monitoring:                                          │
│  ├── CloudTrail: all API calls, integrity validation │
│  ├── GuardDuty: threat detection enabled             │
│  ├── Config: all resource changes tracked            │
│  └── Security Hub: compliance score (PCI standard)  │
│                                                       │
└─────────────────────────────────────────────────────┘

Audit & Compliance:
├── AWS Artifact: SOC/PCI compliance reports
├── Config conformance packs: PCI-DSS pack
├── Log Archive account: immutable logs (S3 Object Lock)
└── Audit account: read-only access for auditors
```

---

### Q17. TRICKY: What is the difference between encryption at rest and encryption in transit? What AWS services encrypt by default?

**Answer:**

**Encryption in Transit:**
- Protects data moving between clients and services, or between services.
- Implemented via TLS/SSL.
- AWS **enforces** TLS on most service endpoints (S3, DynamoDB, API Gateway, etc.).
- For internal VPC traffic (EC2 to EC2): NOT encrypted by default unless your application uses TLS. AWS backbone encrypts some traffic transparently, but don't rely on this.
- Services like MSK, ElastiCache, EFS allow you to enforce TLS in transit.

**Encryption at Rest:**
- Protects data stored on disk.
- Most AWS services support encryption at rest, but it may not be ON by default.

**Encrypted by DEFAULT (no action needed):**
- DynamoDB (always encrypted at rest — cannot be disabled)
- AWS Lambda environment variables
- CloudTrail logs (SSE-S3)
- SQS (server-side encryption available, SSE-SQS is default for new queues in some regions)
- SNS (SSE available)
- Secrets Manager (always encrypted with KMS)

**Encryption available but NOT default (you must enable):**
- EBS volumes (default depends on account settings — can set account-level default)
- RDS/Aurora (you must enable when creating)
- S3 (SSE-S3 is now default for new objects as of 2023)
- ElastiCache (must enable at cluster creation)
- Redshift (must enable)

**Account-level EBS encryption default:** You can enable "EBS encryption by default" per region so all new EBS volumes are automatically encrypted with a default KMS key.

---

### Q18. What is AWS Nitro Enclaves and what security problem does it solve?

**Answer:**

Nitro Enclaves are **isolated compute environments** carved out of an EC2 instance that provide:
- **No persistent storage** (no disk).
- **No external networking** (no internet, no VPC).
- **No SSH/console access** (no operator access, not even from the parent EC2 instance).
- **Cryptographic attestation** — the enclave can prove its identity and code state to external services.

**Problem it solves:** Processing highly sensitive data (cryptographic keys, cardholder data, PII) where you need to ensure the processing environment cannot be accessed, inspected, or tampered with — even by the instance owner.

**Example use case — PAN (card number) processing:**
```
Payment service (on EC2 parent)
    │
    └── vsock (virtual socket — only communication channel)
            │
            ▼
    Nitro Enclave
    ├── Receives encrypted PAN + KMS data key
    ├── Calls KMS to decrypt data key (using attestation doc as condition)
    │   └── KMS verifies enclave's cryptographic identity before decrypting
    ├── Decrypts PAN
    ├── Processes PAN (tokenize, charge, etc.)
    ├── Returns result via vsock
    └── PAN never leaves enclave in plaintext
```

**KMS key policy condition:** You can add an `aws:PrincipalArn` + enclave attestation condition so KMS will ONLY decrypt for verified enclaves with specific code hash.

---

### Q19. TRICKY: You get a GuardDuty finding: "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration." What does this mean and what do you do?

**Answer:**

**What the finding means:**
EC2 instance credentials (temporary credentials from the instance's IAM role via IMDS) were used from an IP address OUTSIDE of AWS — meaning someone exfiltrated the credentials from your instance and is using them from elsewhere (their own machine, another cloud, etc.).

**This is a critical severity finding indicating active credential theft.**

**Immediate response:**

```
Step 1: ISOLATE
├── Modify the compromised instance's security group → deny all inbound/outbound
├── Detach the IAM role from the instance
└── Create a deny-all SCP if entire account appears compromised

Step 2: REVOKE CREDENTIALS
├── Add a deny policy to the IAM role with condition:
│   "DateLessThan": {"aws:TokenIssueTime": "<current time>"}
│   This invalidates ALL existing sessions for this role
└── Do NOT delete the role yet — you need it for forensics

Step 3: INVESTIGATE
├── CloudTrail: What API calls were made with the stolen credentials?
│   ├── Was any data read/exfiltrated?
│   ├── Were new IAM users/roles created?
│   └── Were any resources modified?
├── GuardDuty → Detective: visualize the attack timeline
├── VPC Flow Logs: What connections did the instance make?
└── Inspector: Was there a CVE that enabled the SSRF/compromise?

Step 4: REMEDIATE
├── Force IMDSv2 on all instances (blocks most SSRF attacks)
├── Patch the vulnerability that allowed credential access
├── Review and tighten IAM role permissions (least privilege)
└── Post-incident review + update runbooks
```

---

### Q20. TRICKY: What is the confused deputy problem in AWS and how do cross-account roles prevent it?

**Answer:**

**The Confused Deputy Problem:** A service (the deputy) has permissions to access a resource. A malicious actor tricks the service into using those permissions on behalf of the attacker, even though the attacker themselves doesn't have those permissions.

**AWS example:** Assume Service X has a resource-based policy: "Any AWS principal can invoke me." Service X has an IAM role that can write to your S3 bucket. An attacker calls Service X from their account, and Service X uses its powerful role to write malicious data to YOUR bucket.

**The fix — `aws:SourceArn` and `aws:SourceAccount` condition keys:**

```json
// Service role trust policy — without protection (VULNERABLE)
{
  "Effect": "Allow",
  "Principal": {"Service": "lambda.amazonaws.com"},
  "Action": "sts:AssumeRole"
}

// Service role trust policy — with confused deputy protection (SAFE)
{
  "Effect": "Allow",
  "Principal": {"Service": "lambda.amazonaws.com"},
  "Action": "sts:AssumeRole",
  "Condition": {
    "ArnLike": {
      "aws:SourceArn": "arn:aws:lambda:us-east-1:123456789:function:MyFunction"
    },
    "StringEquals": {
      "aws:SourceAccount": "123456789"
    }
  }
}
```

Now the Lambda service can only assume this role when acting on behalf of YOUR specific Lambda function in YOUR account — not on behalf of an attacker's function.

**Always add `aws:SourceArn`/`aws:SourceAccount` conditions** when service principals (lambda.amazonaws.com, firehose.amazonaws.com, etc.) need to assume roles.

---

*End of Part 5 — 20 Questions covering Security, IAM, KMS, WAF, Shield, GuardDuty, Organizations, Control Tower*
