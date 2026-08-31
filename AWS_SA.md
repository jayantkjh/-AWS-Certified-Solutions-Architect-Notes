# AWS Solutions Architect — Exam & Interview Cheatsheet

*Condensed reference covering storage, compute, networking, databases/migration, messaging, caching, analytics/ML, and identity/security/governance — built from practice questions and interview-style notes.*

## 1. Storage

### EBS volume types

| Type | Backed by | Best for | Boot volume? | Focus |
|---|---|---|---|---|
| gp3 | SSD | General purpose, configurable IOPS/throughput | Yes | Balanced, cost-effective |
| gp2 | SSD | General purpose, occasional bursts | Yes | Balanced |
| io1/io2 | SSD | Mission-critical DBs, provisioned IOPS | Yes | **IOPS** (up to 64,000) |
| st1 | HDD | Big data, logs, Kafka, sequential throughput | **No** | **Throughput** |
| sc1 | HDD | Infrequent, large cold datasets | **No** | Lowest cost |
| Instance Store | Local disk | Temp/scratch cache, very high I/O | Yes | Not persistent — lost on HW failure |

**Rule of thumb:** Database + high IOPS → io1/io2 · General workload → gp3 · Big sequential/streaming data → st1 · Cold data → sc1.

### File systems — pick by client + protocol

| Requirement | Choose |
|---|---|
| Shared filesystem across many EC2/ECS tasks (Linux only, NFS) | **EFS** |
| High throughput despite small stored data | **EFS Provisioned Throughput** |
| Throughput that scales with amount of data stored | **EFS Bursting Throughput** (default) |
| Unpredictable workload, automatic throughput scaling | **EFS Elastic Throughput** |
| Massive parallelism / thousands of clients | **EFS Max I/O** performance mode (vs default General Purpose) |
| Windows + Linux, SMB, NTFS ACLs, AD integration | **FSx for Windows File Server** |
| HPC / parallel compute, transparent S3 tiering (hot/cold) | **FSx for Lustre** |
| Windows *and* Linux need SMB **and** NFS simultaneously | **FSx for NetApp ONTAP** |
| POSIX Linux/Mac/Windows-via-NFS, no SMB needed | **FSx for OpenZFS** |
| POSIX + Multi-AZ + cross-Region backup, RPO ≤ 8h | **EFS (Regional, Standard) + AWS Backup cross-Region copy** |
| Block storage for a single EC2 instance | **EBS** |
| Small structured NoSQL records | **DynamoDB** |
| Large object/file storage | **S3** |
| On-prem app needs SMB/NFS access to S3-backed data | **AWS Storage Gateway — File Gateway** |

EFS storage classes: **Standard** (frequent) → **IA** (infrequent) → **Archive** (rarely accessed); **Regional** (Multi-AZ, HA) vs **One Zone** (cheaper, single AZ, recreatable data).

### S3

- **Object ownership**: the *uploading account* owns the object even in someone else's bucket (classic cross-account Redshift `UNLOAD` gotcha) — fix with a cross-account IAM role (Bucket Role + Cluster Role trust), not a bucket policy. Doesn't apply if either side uses SSE-KMS.
- **Access Points**: fine-grained per-prefix access when many services share one bucket — scales better than per-object bucket policies or per-user inline IAM policies.
- **Protect against accidental deletion**: Versioning + **MFA Delete**.
- **Object Lock retention**: explicit (Retain-Until-Date) or bucket default (a duration); explicit always overrides the default; different object versions can carry different retention modes.
- **Encryption choice**:
 - AWS holds keys but you need an **audit trail of key usage** → **SSE-KMS**.
 - You want to **provide/manage your own keys** but let S3 do the encryption work → **SSE-C**.
 - You need a **proprietary/custom encryption algorithm** → **Client-Side Encryption**.
- **CloudFront in front of S3, repeated requests for the same objects** → CDN edge caching cuts latency, cuts S3 request load, and is usually cheaper than serving directly from S3 (no S3→CloudFront transfer fee).
- **Large objects, geographically dispersed *uploaders*** → **S3 Transfer Acceleration** (CloudFront/PUT-POST is only better for objects <1GB).
- **CloudFront + private S3 origin** → **Origin Access Control (OAC)** (avoids making the bucket public). CloudFront + custom HTTPS domain → ACM certificate **must be in us-east-1**. S3 bucket's own region doesn't drive the ACM region.
- **Signed URL vs Signed Cookie** (both give temporary, expiring access to private CloudFront content): Signed URL → one specific object (e.g. a single video); Signed Cookie → access to multiple objects (e.g. a subscriber area) without changing the URL.
- Capacity tiers: **EC2 Instance Store** → fastest temporary processing; **S3 Standard** → durable, large-scale (hundreds of TB); **S3 Glacier** → cheap long-term archival.

---

## 2. Compute, Scaling & Resilience

- **Long-running app** → **ECS Service + Fargate**. **Scheduled/one-time job** → **ECS RunTask + Fargate** (no ECS Service needed).
 - Simple scheduled container → EventBridge Scheduler → ECS Fargate RunTask.
 - Complex multi-step workflow → EventBridge Scheduler → Step Functions → ECS/Batch.
 - Complex batch processing → EventBridge/Step Functions → AWS Batch → Fargate/EC2.
- **EC2 automatic recovery from hardware failure** (standalone instance, no ASG): CloudWatch alarm on the **system status check** → EC2 Recover action. Requires **EBS-backed** instance (not instance store). Recovered instance keeps instance ID, private IP, Elastic IP, and attached EBS volumes — but **in-memory data is lost** (recovery involves a reboot).
- **Run exactly one instance, but recover automatically if it or its AZ fails** (can't horizontally scale the app):
 1. Bake an **AMI** of the app.
 2. Allocate an **Elastic IP** as the fixed address.
 3. Give the instance an **IAM role** with `ec2:AssociateAddress`.
 4. Build a **Launch Template** (AMI + instance type + IAM role + **User Data** script that re-associates the EIP on boot).
 5. Put it in an **Auto Scaling Group spanning 2+ AZs** with **Min=1, Desired=1, Max=1**.
 6. On failure, the ASG launches a replacement in another AZ; User Data reattaches the same EIP — application identity (its public IP) doesn't change.
- **Detailed EC2 monitoring**: default CloudWatch metric interval is 5 minutes; enabling **detailed monitoring** drops it to 1-minute granularity — simplest way to get finer-grained CPU/network/disk visibility, no extra tooling needed.
- **Patch EC2 instances behind an ALB without disrupting traffic** → the SSM automation document `AWSEC2-PatchLoadBalancerInstance` (deregister, patch/reboot, then re-register once healthy). Scheduling patch windows → **Systems Manager Maintenance Windows**. The patching engine itself → **Systems Manager Patch Manager**.

**Remote access to EC2 without managing SSH keys**

| Requirement | Best choice |
|---|---|
| Temporary SSH access, avoid permanent keys, public IP | **EC2 Instance Connect** |
| Private EC2 but still want SSH | **EC2 Instance Connect Endpoint** |
| Private EC2, no SSH/port 22 desired at all | **Systems Manager Session Manager** |
| Long-term traditional SSH | Static key pair |

- **CloudWatch Logs agent on EC2** → even after an instance terminates, its logs remain in CloudWatch Logs (log data survives instance lifecycle).
- **Lambda scales extremely fast — monitor for cost/downstream risk**: watch `ConcurrentExecutions`, `Invocations`, `Errors`, `Throttles`, `Duration`; alarm (e.g. `ConcurrentExecutions > threshold`) → SNS → engineering team.
- **RDS Multi-AZ failover still causes a brief interruption**, including for version upgrades — use **Blue/Green Deployments** for RDS to avoid downtime during upgrades.
- **Migrate on-prem servers/VMs to AWS with minimal change (lift-and-shift)** → **AWS Application Migration Service (MGN)**: continuous replication → test instance → cutover. For ongoing DR posture instead of a one-time migration → **AWS Elastic Disaster Recovery (DRS)**.

#### RDS vs Aurora resilience

| Feature | RDS Multi-AZ | Aurora |
|---|---|---|
| Standby/reader serves reads | No (standby doesn't serve reads) | Yes (Aurora Replicas do) |
| Automatic failover | Yes | Yes |
| Shared distributed storage | No | Yes |
| Cross-Region DR | Cross-Region Read Replica / backups | **Aurora Global Database** |

---

## 3. Networking

### Global Accelerator vs CloudFront vs ALB/NLB

| Application | Protocol | Best fit |
|---|---|---|
| Gaming, VoIP, IoT telemetry | UDP/TCP | **Global Accelerator** |
| Web app, static site, images/video | HTTP/HTTPS | **CloudFront** |
| API acceleration | HTTP/HTTPS | CloudFront or Global Accelerator depending on need for static IP/failover speed |

- **Global Accelerator for Blue/Green cutover**: shift traffic gradually or all-at-once between environments via traffic dials/endpoint weights, effective within seconds — avoids client/resolver **DNS caching** delays that plain DNS-based (Route 53 weighted) cutovers suffer from.
- **ALB**: DNS-name-based endpoint; backing IPs are AWS-managed and **can change** — never whitelist ALB IPs as fixed. **NLB**: supports **static IPs** (one Elastic IP per AZ) — the right answer whenever a partner/firewall needs to **whitelist a fixed IP**. Fixed **outbound** IP from a VPC to a third party → **NAT Gateway + Elastic IP**.

### Route 53

- **Active-passive failover**: Health Check (HTTPS, path, interval, failure threshold) on the primary → primary Alias record (`failover_routing_policy = PRIMARY`, `evaluate_target_health = true`) + secondary Alias record (`SECONDARY`) pointing at Region B's ALB.
- **CNAME vs Alias**: CNAME **cannot** be created at the zone apex (e.g. `example.com`); Alias **can**, and Alias is required/preferred for pointing at AWS resources (ALB, CloudFront, S3). CNAME *can* point to an ALB DNS name for a subdomain (e.g. `www.example.com`), and can point anywhere (even outside AWS); Route 53 charges for CNAME lookups, not for Alias lookups.

### Hybrid DNS (on-prem ↔ AWS)

- **Inbound Resolver endpoint**: on-prem DNS resolvers forward queries **into** the VPC to resolve AWS resource names.
- **Outbound Resolver endpoint**: Route 53 Resolver forwards queries **out** to on-prem DNS servers for on-prem domain names (via forwarding rules for domains like `corp.example.com`). Connectivity via Direct Connect or VPN. No "universal endpoint" exists.

### NAT instance vs NAT gateway

| Feature | NAT Instance | NAT Gateway |
|---|---|---|
| Managed by | You (OS, patching, HA) | AWS |
| Security Groups | Yes | No |
| Port forwarding | Yes | No |
| Bastion server use | Yes | No |
| Recommended default | Only for custom NAT needs | **One per AZ, for production HA** |

### Multi-VPC / multi-account connectivity

| Need | Best choice |
|---|---|
| Cheapest private connectivity across many accounts, same Region | **Shared VPC** — create subnets in one account, share via **AWS Resource Access Manager (RAM)** |
| Full mesh across many VPCs (e.g. 25) + Direct Connect, minimize private VIF sprawl | **Transit Gateway** (route propagation) + a single **Transit VIF** from Direct Connect to the TGW |
| Central resources (VPC endpoints, DNS resolver, directory services) shared by many spoke VPCs on a TGW | **Shared Services VPC** attached to the Transit Gateway |
| Simple, single-VPC VPN | **Virtual Private Gateway (VGW)** |
| Large/multi-VPC hybrid architecture | **Transit Gateway** (VGW doesn't centralize routing across many VPCs) |
| Private connectivity to a SaaS/partner service (one-way) | **AWS PrivateLink** |
| General inter-VPC/on-prem routing across many parties | **Transit Gateway** (PrivateLink is one-directional to a specific service, not general routing) |
| Standardize a multi-account landing zone with governance baked in | **AWS Control Tower** (creates/governs accounts with SCP guardrails, Config rules, CloudTrail) + centralized networking account + RAM-shared subnets |

### Security / edge protection

- **AWS WAF**: HTTP/HTTPS layer-7 filtering — Geo Match (block/allow by country), IP Set (allow/block specific IPs), SQLi/XSS rules, rate limiting. Attaches to CloudFront, ALB, or API Gateway.
- **AWS Shield (Advanced)**: DDoS protection, includes DRT engagement and CloudWatch/WAF-log based audit trail — minimal architecture change compared to introducing a new edge layer.
- **AWS Firewall Manager**: centrally manage WAF/Shield/firewall policies **across many accounts**.
- Typical edge stack: `Internet → CloudFront → Shield → WAF → ALB/API Gateway → EC2`.

### Data-plane access patterns

- **VPC → S3 without NAT / from a private subnet** → **S3 Gateway Endpoint** (free, avoids NAT data-processing charges).
- **Direct Connect + Public VIF** → reach AWS public-service IP prefixes (e.g. S3) without traversing the public internet.
- **AWS Transfer Family** data stores are limited to **S3 and EFS only**.
- SFTP migration pattern: existing on-prem SFTP app + private connectivity already exists + need immediate (not polled) processing + HA →
 **Internal Transfer Family endpoint (private) + Multi-AZ + S3 backing store + Managed Workflow invoking Lambda immediately on upload.**

---

## 4. Database & Migration

- **Heterogeneous migration** (different source/target engines, e.g. Oracle → Aurora PostgreSQL, need indexes/FKs/stored procs preserved):
 **AWS SCT** (schema/code conversion: structure) → **AWS DMS** (full load + CDC: data, keeps source/target in sync during migration) → **application cutover**.
 *"Basic Schema Copy" (a DMS built-in) does NOT migrate secondary indexes, FKs, or stored procedures — use full SCT for that.*
- **SQL Server (T-SQL) app → Aurora PostgreSQL with minimal app refactor** → **Babelfish for Aurora PostgreSQL** (understands T-SQL/SQL Server wire protocol) + SCT/DMS for schema & data.
- RDS uses AWS **service-linked roles** (e.g. `AWSServiceRoleForRDS`) to perform actions on your behalf — enhanced monitoring, automated management ops, integrations with other services.
- **DynamoDB Point-in-Time Recovery (PITR)**: continuous, per-second backups, restorable to any second in the preceding **35 days** — protects against accidental writes/deletes (e.g. a bad `DeleteItem` or a test script hitting prod).

---

## 5. Messaging & Streaming

### SQS vs Kinesis

| | SQS | Kinesis Data Streams |
|---|---|---|
| Purpose | Message queue, decoupling | Real-time streaming/analytics |
| Ordering | FIFO queue type available | Ordering within a shard |
| Replay | Limited by retention | Consumers can reread retained records |
| Multiple consumers | Possible | Designed for it |
| Typical use | Order processing | IoT/telemetry/financial streams |

### SQS specifics

- **Standard → FIFO migration**: cannot convert in place — create a **new** queue ending in **`.fifo`**, update producers/consumers, add `MessageGroupId` (and dedup config as needed). Throughput **with batching: 3,000 msg/s**; without: 300 msg/s.
- **Delay Queue** hides a message *before* a consumer ever sees it (postpone delivery). **Visibility Timeout** hides it *after* a consumer receives it (gives time to process). **Dead-Letter Queue** captures messages after repeated processing failures. **FIFO** ≈ ordering + dedup, not primarily a hiding mechanism. **Temporary Queue** = short-lived request/response messaging pattern.
- **SQS backlog growing** → scale EC2/consumers based on **queue depth** (dynamic scaling), not just CPU. Predictable load → scheduled scaling. Need event routing → EventBridge. High-throughput streaming → Kinesis instead of SQS.

### Kinesis family

- **Kinesis Data Streams** — "give me the streaming data" (ingest/buffer).
- **Kinesis Data Analytics** (Flink) — "analyze this streaming data right now" (continuous processing).
- **Kinesis Data Firehose** — "deliver the resulting data somewhere" (e.g. to S3).
- **High-rate individual record puts causing throttling** → batch with `PutRecords` first; use exponential backoff for transient throttling; increase shards only if genuinely more throughput capacity is needed; reduce retention only to cut cost, not to fix throughput.

### Lambda / SNS scaling limits

- Lambda concurrency ceiling breached via SNS fan-out at high request rates → **you can't "add servers"** (fully managed/serverless) — **contact AWS Support to raise the account concurrency quota**.

---

## 6. Caching

| Requirement | Answer |
|---|---|
| General RDS/DB read-heavy caching | **ElastiCache** |
| Geospatial caching, pub/sub, persistence, replication, advanced data structures | **Redis** |
| Simple, highly scalable, multi-threaded cache | **Memcached** |
| DynamoDB-specific caching | **DAX** |
| Global network acceleration (not caching) | Global Accelerator |

| Feature | Memcached | Redis |
|---|---|---|
| Multi-threading | Yes | No (traditionally single-threaded) |
| Persistence | No | Yes |
| Replication | No | Yes |
| Pub/Sub | No | Yes |
| Best when | Simple, scalable, multi-core caching | Advanced cache / data store / sessions |

**ElastiCache fits**: read-heavy or compute-intensive workloads (leaderboards, gaming, session store, recommendation engines). **Not** a fit for write-heavy workloads, ETL, or complex JOINs (use RDS/Aurora, Glue, or EMR for those).

---

## 7. Analytics, ML & BI

| Service | Main purpose |
|---|---|
| **Kinesis Data Streams** | Real-time streaming ingestion |
| **Kinesis Data Analytics (Flink)** | Continuous stream processing / real-time analytics |
| **Kinesis Data Firehose** | Deliver streaming data to destinations (S3, etc.) |
| **AWS Glue** | ETL / data integration |
| **AWS Glue DataBrew** | No-code visual data prep + built-in profiling + shareable "recipes" (lineage/audit) |
| **Amazon Athena** | SQL queries directly on S3 |
| **Amazon Redshift (Serverless)** | Data warehouse, MPP; **Redshift ML** = SQL-native model training |
| **Amazon OpenSearch** | Search + real-time analytics |
| **Amazon QuickSight** | BI dashboards / visualization (doesn't require AWS-native data, but S3/Athena/Redshift/RDS are common sources) |
| **Amazon Comprehend** | Pre-trained NLP — sentiment, entity detection (pairs naturally with Textract-extracted text; least ops overhead vs custom SageMaker models) |
| **Amazon DynamoDB** | NoSQL key-value/document store |

Pipeline shorthand: `S3 → Glue (ETL) → Redshift Serverless → Redshift ML` for serverless, SQL-only MPP analytics + ML. `Textract → Comprehend → S3` for document text + sentiment with least operational overhead.

---

## 8. Identity, Security & Governance

### Authentication for app users → AWS resource access

```
User → Cognito User Pool (sign-up/sign-in, issues JWT)
 → Cognito Identity Pool (exchanges JWT for temporary AWS credentials)
 → IAM Role (fine-grained permissions)
 → S3 (or other AWS resource)
```

### "Which security service answers this question?"

| Service | Answers | Think |
|---|---|---|
| **IAM Access Analyzer** | "Who can access this resource?" | Access |
| **IAM Access Advisor** | "What services has this principal actually used, and when?" | Last used |
| **AWS Config** | "Is my resource configured per my rules? What did it look like at time X?" | Compliance / history |
| **Amazon Inspector** | "Does my workload have software vulnerabilities?" | Vulnerabilities |
| **Amazon GuardDuty** | "Is something malicious happening?" (crypto-mining, compromised EC2, suspicious API activity) | Threat detection |
| **AWS WAF** | "Is bad HTTP traffic coming in?" (SQLi, XSS, bot traffic, rate limiting) | App-layer |
| **AWS Shield** | "Am I being DDoS'd?" | DDoS |
| **AWS Firewall Manager** | "How do I enforce WAF/Shield/firewall rules across every account centrally?" | Central policy |
| **CloudWatch** | Performance metrics, alarms | Monitor |
| **CloudTrail** | Who called which API, when | Audit |

### Governance at the Organization level

- **Service Control Policies (SCPs)**: cap max permissions across **all accounts including root**; an SCP deny/non-allow overrides any IAM allow; SCPs **do not** apply to service-linked roles.
- **Tag Policies** answer "what tags are required/valid?" while **SCPs** answer "what actions are allowed?" — e.g. Tag Policy enforces `dataClassification = confidential|public`; a paired SCP denies `RunInstances` without the required tag and denies `DeleteTags`.
- **Multi-account landing zone**: **AWS Control Tower** for account creation/governance (guardrails = SCPs + Config rules + CloudTrail) + a centralized networking account hosting a shared VPC, with subnets shared via **RAM** to workload accounts — centralizes NAT/internet egress and reduces duplication.
- **Standardize infra across many accounts/Regions**: a CloudFormation **Stack** = one deployment unit; a **StackSet** = deploy that same template across many accounts/Regions (can target specific OUs) with centralized update management.

---

## Exam & Interview Thumb-Rules

- **CloudWatch** → performance & alarms · **CloudTrail** → who/what API audit · **Config** → resource history & compliance.
- **Global Accelerator** → non-HTTP/TCP-UDP, static IP, fast failover, DNS-cache-proof cutovers · **CloudFront** → HTTP content caching/CDN.
- **ALB** → DNS-based, IPs can change, never whitelist · **NLB** → static IP support, whitelist-friendly · **NAT Gateway + EIP** → fixed *outbound* IP.
- **AD Connector** → login passthrough only · **Simple AD** → small shop, no trust relationships · **Managed Microsoft AD** → workloads + trust + SSO, >5,000 users.
- **st1/sc1** → never a boot volume · **gp2/gp3/io1/io2/instance store** → can be a boot volume.
- **SCT** → structure/schema · **DMS** → data + CDC · combine both for heterogeneous migrations.
- **GuardDuty** → is something bad happening · **WAF** → is bad web traffic coming in · **Shield** → am I being DDoS'd · **Firewall Manager** → enforce all of the above centrally, org-wide.
- Managed/serverless services (SNS, SQS, Lambda) scale by raising **quotas/limits** — you never "add servers" to them.
- SQS Standard **cannot** be converted to FIFO in place — always a new `.fifo`-suffixed queue.
- RDS Multi-AZ failover ≠ zero downtime for version upgrades — use **Blue/Green Deployments** for that.
