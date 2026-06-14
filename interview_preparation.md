# AWS Interview Preparation — Complete Guide
### For DevOps Engineer (4 Years Experience) — IBM / Amazon / Flipkart / Google-tier

---

# TABLE OF CONTENTS
1. IAM (Identity & Access Management)
2. EC2 & Compute
3. VPC & Networking
4. S3 & Storage (EBS, EFS, Glacier)
5. RDS & Databases
6. Load Balancing & Auto Scaling
7. Route 53 & DNS
8. CloudWatch, CloudTrail & Monitoring
9. CI/CD (CodePipeline, CodeBuild, CodeDeploy)
10. Lambda & Serverless
11. Cost Optimization
12. Security & Disaster Recovery
13. Troubleshooting Scenario Bank
14. Architecture/System Design Scenarios
15. Quick-Fire Round
16. Topic Weightage Map

---

# 1. IAM (IDENTITY & ACCESS MANAGEMENT)

### Q1. IAM User vs IAM Role vs IAM Group vs IAM Policy — explain each.
**Answer:**
- **IAM User** — represents a person or application with long-term credentials (password/access keys). Use for individuals needing console/CLI access.
- **IAM Role** — provides **temporary** credentials, assumed by users, applications, or AWS services (e.g., EC2 assumes a role to access S3). No long-term keys — more secure.
- **IAM Group** — collection of users; attach policies to the group instead of each user individually.
- **IAM Policy** — JSON document defining permissions (Allow/Deny on specific actions/resources). Attached to users, groups, or roles.

**Best practice:** Always prefer Roles over long-term access keys for applications/EC2/Lambda — eliminates credential leakage risk.

---

### Q2. Explain how IAM policy evaluation logic works — what happens when there's an explicit Deny and an Allow for the same action?
**Answer:** Evaluation order:
1. By default, everything is **implicitly denied**
2. AWS evaluates ALL applicable policies (identity-based, resource-based, SCPs, permission boundaries)
3. **An explicit Deny ALWAYS wins** — overrides any Allow, regardless of where it comes from
4. If no explicit Deny and at least one explicit Allow exists → action is permitted
5. Otherwise → implicit deny

**Interviewer Notes:** This is heavily tested. The phrase to remember: "explicit deny beats explicit allow, always" — relevant when debugging why a user with `AdministratorAccess` still can't perform an action (an SCP or permission boundary may have an explicit deny).

---

### Q3. What is a Permission Boundary vs an SCP (Service Control Policy)? When would you use each?
**Answer:**
- **Permission Boundary** — attached to an IAM user/role, sets the MAXIMUM permissions that user/role can ever have, regardless of what identity policies grant. Use case: allow developers to create IAM roles for their apps, but ensure those roles can never exceed a boundary (e.g., can't access billing or IAM itself).
- **SCP** — applied at the AWS Organizations level (account/OU), sets maximum permissions for ALL identities in that account, including the root user. Use case: company-wide guardrail — e.g., "no account in the dev OU can ever use regions outside us-east-1/ap-south-1" regardless of individual IAM policies.

---

### Q4. SCENARIO: A developer says their Lambda function gets "AccessDenied" when calling S3, but the IAM role attached has `s3:GetObject` with `Resource: "*"`. What else could be blocking it?
**Answer:** Checklist:
1. **S3 Bucket Policy** — an explicit Deny in the bucket policy overrides IAM Allow (e.g., bucket enforces `aws:SecureTransport: true` and the request is over HTTP, or restricts to a specific VPC endpoint)
2. **S3 Block Public Access settings** — if misconfigured, can block even legitimate cross-account access
3. **KMS key policy** — if the object is encrypted with a customer-managed KMS key, the Lambda role ALSO needs `kms:Decrypt` permission on that specific key's key policy, separate from the S3 permission
4. **VPC endpoint policy** — if Lambda runs in a VPC and uses an S3 VPC endpoint (Gateway endpoint), the endpoint policy might restrict which buckets are accessible
5. **Resource ARN typo/wildcard mismatch** — `Resource: "arn:aws:s3:::mybucket"` (bucket only) vs `"arn:aws:s3:::mybucket/*"` (objects) — `GetObject` needs the `/*` version

---

### Q5. What is the principle of least privilege, and how do you implement it practically in a team of 20 engineers?
**Answer:** Grant only the minimum permissions needed to perform a task — nothing more.

**Practical implementation:**
- Use IAM Roles per application/service (not shared users)
- Group-based policies by team/function (e.g., `developers`, `dba-readonly`, `platform-admins`) rather than per-user custom policies
- Use AWS managed policies as a baseline, customize with inline policies only for specific resource scoping
- Enable IAM Access Analyzer to find unused permissions and overly-permissive policies
- Use temporary credentials via AWS SSO/IAM Identity Center instead of long-lived access keys
- Periodically review with `last-accessed` data (IAM Access Advisor) and remove unused permissions

---

# 2. EC2 & COMPUTE

### Q6. Explain EC2 instance types — when would you pick each family (T, M, C, R, I)?
**Answer:**
- **T-series (T3, T4g)** — burstable, general purpose, cost-effective for variable/low-baseline workloads (dev/test, small web servers). Uses CPU credits — sustained high CPU can exhaust credits and throttle performance.
- **M-series** — balanced compute/memory/network, general-purpose production workloads (app servers).
- **C-series** — compute-optimized (high CPU:memory ratio), for CPU-intensive workloads (batch processing, gaming servers, ML inference).
- **R-series** — memory-optimized, for in-memory databases/caches (Redis, large JVM heaps, SAP HANA).
- **I-series** — storage-optimized with NVMe SSD, for high I/O workloads (NoSQL databases, data warehousing).

---

### Q7. What happens to data when you STOP vs TERMINATE vs REBOOT an EC2 instance?
**Answer:**
- **Reboot** — like an OS restart; all data persists (EBS and instance store both retained); public/private IP unchanged (unless using Elastic IP, which always stays).
- **Stop** — instance moves to a different host potentially; **EBS volumes persist**, but **instance store (ephemeral) volumes are LOST**; public IP is released (unless Elastic IP). You stop paying for compute, but still pay for EBS storage.
- **Terminate** — instance permanently deleted; EBS root volume deleted by default (unless `DeleteOnTermination=false`); instance store always lost; Elastic IP detached.

**Interviewer Notes:** The instance-store-data-loss-on-stop is the classic trap question.

---

### Q8. SCENARIO: An EC2 instance is unreachable via SSH. Walk through your troubleshooting.
**Answer:**
1. **Check instance status checks** in console — "System status check" (AWS infrastructure issue) vs "Instance status check" (OS-level issue, e.g., kernel panic, filesystem corruption)
2. **Security Group** — verify inbound rule allows port 22 from your IP/CIDR
3. **Network ACL** on the subnet — NACLs are stateless, check both inbound AND outbound rules allow port 22 and ephemeral return ports (1024-65535)
4. **Route table** — does the subnet have a route to an Internet Gateway (if public) or is your connection via VPN/Direct Connect/bastion for private subnets?
5. **Public IP/Elastic IP** — confirm instance has one if connecting from internet
6. **Key pair mismatch** — using wrong `.pem` file, or the AMI doesn't have the expected user (`ec2-user` vs `ubuntu` vs `admin`)
7. **If OS-level issue suspected** — use **EC2 Serial Console** or **Session Manager (SSM)** to access without SSH/network dependency — check `/var/log/` for boot issues, disk full (`df -h`), or SSH daemon crashed

---

### Q9. What is the difference between an AMI and a Snapshot?
**Answer:** A **Snapshot** is a point-in-time backup of a single EBS volume (block-level, incremental after the first). An **AMI (Amazon Machine Image)** is a template for launching EC2 instances — includes the root volume snapshot PLUS launch permissions, block device mappings, and (for instance-store backed AMIs) the actual image. You can create an AMI FROM a snapshot, but a snapshot alone can't launch an instance — it needs to be registered as part of an AMI first.

---

### Q10. SCENARIO: You need to upgrade the instance type of a production EC2 instance from `m5.large` to `m5.xlarge` with minimal downtime. Steps?
**Answer:**
1. Confirm the new instance type is supported in the current AZ (`m5.xlarge` availability can vary by AZ)
2. **Stop** the instance (brief downtime — EBS-backed instances retain data on stop)
3. Change instance type via console/CLI: `aws ec2 modify-instance-attribute --instance-id i-xxx --instance-type m5.xlarge`
4. **Start** the instance
5. Verify application health, check CloudWatch metrics post-change

**For true zero-downtime (HA setups):** if behind an ASG/ALB, launch new instances with the new instance type in the launch template, let ASG replace old instances one at a time (rolling), and the ALB drains connections gracefully — no single-instance downtime.

---

# 3. VPC & NETWORKING

### Q11. Explain VPC components: Subnet, Route Table, Internet Gateway, NAT Gateway, Security Group, NACL.
**Answer:**
- **Subnet** — a range of IPs within a VPC, tied to a single AZ. Public (route to IGW) or private (no direct route to IGW).
- **Route Table** — defines where network traffic is directed; each subnet associates with one route table.
- **Internet Gateway (IGW)** — allows resources in public subnets to communicate with the internet (bidirectional).
- **NAT Gateway** — allows private subnet resources to initiate OUTBOUND internet connections (e.g., for updates) WITHOUT being reachable from the internet inbound. Placed in a public subnet.
- **Security Group (SG)** — instance-level (ENI-level) virtual firewall, **stateful** (return traffic automatically allowed), supports only Allow rules.
- **Network ACL (NACL)** — subnet-level firewall, **stateless** (must explicitly allow both inbound AND outbound, including ephemeral ports for return traffic), supports both Allow and Deny rules, evaluated in rule-number order.

---

### Q12. SCENARIO: An EC2 instance in a public subnet has a public IP and an IGW attached to the VPC, but still can't reach the internet. What do you check?
**Answer:**
1. **Route table** — does the subnet's route table have a route `0.0.0.0/0 → igw-xxxx`? (Most common miss — IGW attached to VPC doesn't automatically mean the subnet routes to it)
2. **Security Group** — outbound rules allow the traffic (default SG allows all outbound, but custom SGs might not)
3. **NACL** — both inbound (return traffic) and outbound rules must allow the relevant ports — remember NACLs are stateless
4. **Public IP actually assigned** — `Auto-assign Public IP` must be enabled at launch, OR an Elastic IP attached; having a route to IGW doesn't help without a public IP
5. **OS-level firewall** (iptables/ufw) blocking outbound traffic
6. **DNS resolution** — VPC's `enableDnsSupport`/`enableDnsHostnames` settings, in case the issue is DNS not connectivity

---

### Q13. VPC Peering vs Transit Gateway vs PrivateLink — when to use each?
**Answer:**
- **VPC Peering** — 1:1 connection between two VPCs, private IP routing, NOT transitive (if A peers with B, and B peers with C, A cannot reach C through B). Good for small numbers of VPCs.
- **Transit Gateway** — hub-and-spoke model, connects many VPCs (and on-prem via VPN/Direct Connect) through a central hub, IS transitive, supports route tables for segmentation. Use for large multi-VPC/multi-account architectures.
- **PrivateLink (VPC Endpoint Services)** — exposes a SPECIFIC SERVICE (not the whole VPC) privately to other VPCs/accounts via an ENI — used for SaaS providers exposing services without traversing the internet, or accessing AWS services (S3, DynamoDB via Gateway endpoints, others via Interface endpoints) without an IGW/NAT.

---

### Q14. SCENARIO: You have a 3-tier architecture (web, app, db) across public and private subnets. The app tier can't connect to the RDS database in a private subnet. Debug steps?
**Answer:**
1. **Security Group on RDS** — must allow inbound on DB port (3306/5432) FROM the app tier's security group (not CIDR — SG-to-SG references are best practice)
2. **Subnet group** — RDS subnet group must include subnets in the same AZs as the app tier (or at least overlapping AZs for Multi-AZ RDS)
3. **NACLs** on both app and DB subnets — stateless rules for both directions
4. **Route tables** — app and DB subnets should be able to route to each other (usually automatic within same VPC via local route, but check if using separate route tables with overly restrictive entries)
5. **DB endpoint/connection string** — verify app is using the correct RDS endpoint (not a cached old IP — RDS endpoints can change IP on failover, always use the DNS endpoint, never the IP)
6. **Credentials/Secrets Manager** — if using IAM database authentication or Secrets Manager, verify the app's IAM role has `secretsmanager:GetSecretValue` and `rds-db:connect`

---

### Q15. What is the difference between a Gateway VPC Endpoint and an Interface VPC Endpoint?
**Answer:**
- **Gateway Endpoint** — only for S3 and DynamoDB. Works by adding a route table entry — traffic to S3/DynamoDB stays on AWS network, no extra cost.
- **Interface Endpoint** — for most other AWS services (e.g., SSM, Secrets Manager, EC2 API, ECR). Creates an ENI with a private IP in your subnet; charged per-hour + data processing. Requires the VPC to have DNS resolution enabled to resolve the service's private DNS name to this ENI.

---

# 4. S3 & STORAGE (EBS, EFS, GLACIER)

### Q16. S3 storage classes — explain and give use cases.
**Answer:**
- **S3 Standard** — frequently accessed data, low latency, high durability (11 nines)
- **S3 Intelligent-Tiering** — automatically moves objects between tiers based on access patterns; good when access patterns are unpredictable
- **S3 Standard-IA / One Zone-IA** — infrequent access but need millisecond retrieval; One Zone is cheaper but less resilient (single AZ)
- **S3 Glacier Instant/Flexible/Deep Archive** — archival, retrieval times from milliseconds (Instant) to 12+ hours (Deep Archive), lowest cost — for compliance/long-term backups

**Cost optimization use case:** Set up a Lifecycle Policy — logs move to Standard-IA after 30 days, Glacier after 90 days, deleted after 1 year (compliance-dependent).

---

### Q17. EBS vs EFS vs S3 — when to use each.
**Answer:**
- **EBS** — block storage, attached to a SINGLE EC2 instance (at a time, with exceptions for Multi-Attach io1/io2), persists independently of instance lifecycle, used for databases/OS volumes needing low-latency block access.
- **EFS** — file storage (NFS), can be mounted by MULTIPLE EC2 instances/AZs simultaneously, used for shared content (web server farms sharing static assets, shared application data).
- **S3** — object storage, accessed via API/HTTP (not mounted as a filesystem natively, though tools like s3fs exist), used for static assets, backups, data lakes, logs.

---

### Q18. SCENARIO: A user reports they can't delete an S3 bucket — "BucketNotEmpty" error even though the console shows it's empty.
**Answer:** Most likely cause: **Versioning is enabled** on the bucket. The console "empty" view might only show current versions, but **previous versions and delete markers still exist** and count as objects.

**Fix:**
```bash
# Delete all versions and delete markers
aws s3api list-object-versions --bucket mybucket \
  --output json --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' > versions.json
aws s3api delete-objects --bucket mybucket --delete file://versions.json

# Repeat for delete markers
aws s3api list-object-versions --bucket mybucket \
  --output json --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' > markers.json
aws s3api delete-objects --bucket mybucket --delete file://markers.json
```
Then delete the bucket. Also check for incomplete multipart uploads (`list-multipart-uploads`) — these can also block deletion in some edge cases.

---

### Q19. How would you upload a 5GB file to S3? What's different about handling large files?
**Answer:** Single `PutObject` has a 5GB limit. For files >100MB (AWS recommendation), use **Multipart Upload**:
- Splits the file into parts (5MB-5GB each, up to 10,000 parts), uploads in parallel
- If a part fails, only that part is retried (not the whole file)
- Must explicitly complete or abort the multipart upload — incomplete uploads consume storage and cost money (set a lifecycle rule to auto-abort incomplete multipart uploads after N days)

The AWS CLI `aws s3 cp` and SDKs handle this automatically above a size threshold — no manual multipart API calls needed in most cases.

---

### Q20. What is S3 Cross-Region Replication (CRR) and what's a common gotcha?
**Answer:** CRR automatically replicates objects from a source bucket to a destination bucket in a different region — used for compliance (data residency), lower-latency access for global users, or DR.

**Gotchas:**
1. Versioning MUST be enabled on BOTH source and destination buckets
2. CRR is NOT retroactive — only replicates objects uploaded AFTER CRR is enabled (existing objects need S3 Batch Replication separately)
3. Doesn't replicate objects encrypted with SSE-C by default, or objects from another replication (no replica-of-replica chains by default)
4. IAM role for replication needs explicit permissions on both buckets

---

# 5. RDS & DATABASES

### Q21. RDS Multi-AZ vs Read Replicas — explain the difference and use cases.
**Answer:**
- **Multi-AZ** — synchronous replication to a STANDBY instance in another AZ, for **HIGH AVAILABILITY**. Automatic failover during outages/maintenance. The standby is NOT readable — it's purely for failover (except Aurora, where replicas can be readable).
- **Read Replicas** — asynchronous replication to one or more replicas, for **READ SCALING**. Replicas ARE readable — offload read-heavy queries (reporting, analytics) from the primary. Can be cross-region. NOT automatically used for failover (though can be manually promoted).

**Combine both:** Multi-AZ primary for HA + Read Replicas for scaling read traffic — common production pattern.

---

### Q22. SCENARIO: Application reports "too many connections" errors to RDS during peak traffic. How do you diagnose and fix?
**Answer:**
1. **Check current connections vs `max_connections` parameter** — `SHOW STATUS LIKE 'Threads_connected'` (MySQL) or check CloudWatch `DatabaseConnections` metric
2. **`max_connections` is tied to instance class memory** — if undersized instance, connections are capped low
3. **Check for connection leaks** in application — connections not being closed/returned to pool (common with serverless/Lambda creating new connections per invocation without pooling)
4. **Immediate fixes:**
   - Use **RDS Proxy** — connection pooling layer that multiplexes many app connections into fewer DB connections, especially critical for Lambda
   - Increase instance size (more memory → higher `max_connections`)
   - Implement application-side connection pooling (e.g., PgBouncer for Postgres, HikariCP for Java)
5. **Long-term:** if read-heavy, offload to Read Replicas; review ORM connection pool settings (min/max pool size per app instance × number of instances must not exceed DB capacity)

---

### Q23. How does RDS automated backup work, and what's the difference between automated backups and manual snapshots?
**Answer:**
- **Automated backups** — daily full snapshot + transaction logs (continuous), enables **point-in-time recovery (PITR)** to any second within the retention period (1-35 days). Deleted automatically when retention expires, AND deleted if you delete the RDS instance (unless you opt to retain final snapshot).
- **Manual snapshots** — user-initiated, persist indefinitely until manually deleted, even after the RDS instance is deleted. Use for long-term retention/compliance archives or before risky changes (schema migrations).

**PITR restore** creates a NEW RDS instance (can't restore in-place) — important for planning downtime/cutover during recovery.

---

### Q24. SCENARIO: A scheduled RDS maintenance window caused unexpected downtime even though Multi-AZ is enabled. Why might this happen, and how do you minimize impact?
**Answer:** Multi-AZ failover during maintenance typically causes only ~60-120 seconds of downtime (DNS endpoint re-points to standby). Possible reasons for LONGER downtime:
1. **Some engine version upgrades require both primary AND standby to be upgraded** — can't fail over to an un-upgraded standby, causing longer outage
2. **Application not handling the DNS endpoint change gracefully** — connection pools holding stale DNS-resolved IPs (DNS TTL caching issue) — fix with shorter TTL or connection pool recycling on failover
3. **Certain parameter group changes require a reboot** of both instances (`apply immediately` + static parameters)

**Minimize impact:** Schedule maintenance windows during low-traffic periods, use RDS Proxy (handles failover transparently without app-level reconnection logic), test major version upgrades in staging first.

---

# 6. LOAD BALANCING & AUTO SCALING

### Q25. ALB vs NLB vs CLB — differences and use cases.
**Answer:**
- **ALB (Application LB)** — Layer 7 (HTTP/HTTPS), supports path-based/host-based routing, WebSockets, integrates with ECS/EKS/Lambda targets. Use for web applications/microservices.
- **NLB (Network LB)** — Layer 4 (TCP/UDP/TLS), ultra-low latency, handles millions of requests/sec, supports static IP/Elastic IP per AZ. Use for extreme performance needs, non-HTTP protocols, or when client IP preservation is critical.
- **CLB (Classic LB)** — legacy, Layer 4/7 mixed, mostly deprecated — avoid for new designs.

---

### Q26. SCENARIO: Your ALB target group shows targets as "Unhealthy" but the application is running fine when accessed directly via the instance's private IP. Debug steps?
**Answer:**
1. **Health check path/port mismatch** — target group health check might point to `/health` on port 8080, but app serves on a different port, or the health endpoint returns non-2xx/3xx
2. **Security Group** — target's SG must allow inbound from the ALB's security group on the health check port (common miss: SG only allows traffic from a CIDR, not the ALB's SG)
3. **Health check thresholds** — `HealthyThresholdCount`/`UnhealthyThresholdCount`/`Interval`/`Timeout` — app might be slow to respond under load, exceeding timeout
4. **Application binding to `127.0.0.1` instead of `0.0.0.0`** — app responds locally but not to ALB's request from a different IP
5. **Target registration** — instance might be registered in target group but in a different AZ than expected, or deregistering due to connection draining during a deploy

---

### Q27. How does an Auto Scaling Group decide when to scale, and what's the difference between Target Tracking, Step Scaling, and Scheduled Scaling?
**Answer:**
- **Target Tracking** — maintains a metric at a target value (e.g., "keep average CPU at 50%") — ASG automatically calculates how many instances needed; simplest, most commonly used
- **Step Scaling** — define specific scaling actions for different CloudWatch alarm breach magnitudes (e.g., CPU 70-80% → add 1 instance, CPU >80% → add 3 instances) — more granular control
- **Scheduled Scaling** — pre-set capacity changes at specific times (e.g., scale up every weekday at 8 AM before business hours, scale down at 8 PM) — for predictable traffic patterns

**Combine:** Scheduled scaling for baseline predictable patterns + Target Tracking for handling unexpected spikes on top.

---

### Q28. SCENARIO: ASG keeps launching and terminating instances repeatedly (flapping). What's likely wrong and how do you fix it?
**Answer:** Common causes:
1. **Scale-out and scale-in alarms have overlapping/conflicting thresholds** — e.g., scale-out at CPU>70%, scale-in at CPU<70% with no buffer — adding an instance drops average CPU below 70%, triggering immediate scale-in, repeat. **Fix:** add a gap (scale-out at 70%, scale-in at 30%) and use **cooldown periods**.
2. **Health check failing on new instances during startup** — ASG terminates instances that fail health checks before the app finishes initializing. **Fix:** increase `HealthCheckGracePeriod` to allow app startup time.
3. **Launch template/AMI issue** — new instances fail to bootstrap (user data script errors), marked unhealthy, terminated, ASG launches replacement, repeat. **Fix:** check `/var/log/cloud-init-output.log` on a manually-launched instance from the same template.

---

# 7. ROUTE 53 & DNS

### Q29. Explain Route 53 routing policies: Simple, Weighted, Latency, Failover, Geolocation.
**Answer:**
- **Simple** — single resource, no health checks, basic DNS resolution
- **Weighted** — distribute traffic by percentage across multiple resources — useful for canary/A-B testing (e.g., 90% to v1, 10% to v2)
- **Latency-based** — routes to the region with lowest latency for the user — for global apps optimizing user experience
- **Failover** — active-passive; routes to primary, switches to secondary if primary's health check fails — for DR setups
- **Geolocation** — routes based on user's geographic location — for compliance (data residency) or localized content

---

### Q30. SCENARIO: You updated a Route 53 record to point to a new ALB, but users still hit the old endpoint. Why?
**Answer:** **DNS TTL (Time To Live) caching** — the old record's TTL hasn't expired in resolvers/clients' caches. If TTL was set to 3600 (1 hour), some clients/ISPs cache the old IP for up to an hour after the change.

**Mitigation for future changes:** Lower the TTL well BEFORE a planned change (e.g., to 60 seconds, wait for old TTL to expire, then make the change) — this is a standard pre-migration step.

**Also check:** if using an Alias record to an ALB, Alias records resolve dynamically (no TTL issue at Route 53 level), but client-side OS/browser DNS caching can still apply.

---

# 8. CLOUDWATCH, CLOUDTRAIL & MONITORING

### Q31. CloudWatch vs CloudTrail — what's the difference and when do you use each?
**Answer:**
- **CloudWatch** — monitors PERFORMANCE/operational data: metrics (CPU, memory via agent, custom metrics), logs (application/system logs), alarms, dashboards. "What is happening with my resources?"
- **CloudTrail** — records API CALLS/actions taken on your account (who did what, when, from where) — for auditing, security investigation, compliance. "WHO did WHAT?"

**Example:** CloudWatch alarm fires because an S3 bucket's size suddenly dropped to zero (metric). CloudTrail tells you WHO ran the `DeleteObjects` API call that caused it.

---

### Q32. SCENARIO: A CloudWatch alarm for "EC2 CPU > 80%" isn't triggering even though the instance is clearly under high load (verified via SSH `top`). Why?
**Answer:**
1. **Default EC2 CPU metric is at the HYPERVISOR level**, reported every 5 minutes (basic monitoring) — if load spikes briefly and drops, it might not register in the 5-min average. Enable **detailed monitoring** (1-minute granularity)
2. **Alarm evaluation periods** — if alarm requires "3 consecutive periods above threshold" and load is intermittent, it won't breach
3. **Wrong metric/dimension** — alarm might be configured for a different instance ID, or using `CPUUtilization` vs a custom memory metric (memory isn't a default EC2 metric — requires CloudWatch Agent)
4. **SNS topic/notification misconfigured** — alarm state changed to ALARM but notification action failed (check alarm history in console, separate from notification delivery)

---

### Q33. What is the CloudWatch Agent and why is it needed beyond default EC2 metrics?
**Answer:** Default EC2 metrics (from the hypervisor) only include CPU, network, disk I/O (basic), and status checks — NOT memory usage, disk space usage, or custom application metrics, since AWS can't see inside the OS without an agent.

The **CloudWatch Agent** runs on the instance and pushes OS-level metrics (memory %, disk usage %, running processes) and log files to CloudWatch Logs. Essential for any real production monitoring — "my disk is full" or "my app is leaking memory" alarms are impossible without it.

---

# 9. CI/CD (CodePipeline, CodeBuild, CodeDeploy)

### Q34. Explain the AWS native CI/CD pipeline flow: CodeCommit → CodeBuild → CodeDeploy → CodePipeline.
**Answer:**
- **CodeCommit** — managed Git repository (source stage)
- **CodeBuild** — compiles code, runs tests, produces build artifacts (defined via `buildspec.yml`) — the "build" stage
- **CodeDeploy** — deploys the artifact to EC2/ECS/Lambda — supports deployment strategies (in-place, blue-green) via `appspec.yml`
- **CodePipeline** — orchestrates the whole flow — triggers on source change, runs Build stage, then Deploy stage, with optional manual approval gates between stages

---

### Q35. SCENARIO: A CodePipeline deployment to EC2 via CodeDeploy fails at the "Install" lifecycle event. How do you debug?
**Answer:**
1. **Check CodeDeploy agent is running** on the target instance: `sudo service codedeploy-agent status` — if not installed/running, deployments fail immediately
2. **Check deployment logs** on the instance: `/opt/codedeploy-agent/deployment-root/deployment-logs/codedeploy-agent-deployment-logs.log`
3. **Check `appspec.yml`** — file paths/permissions in the `files` section must match actual artifact structure; `hooks` scripts must have execute permissions and correct shebang
4. **IAM role** — the EC2 instance profile needs permission to access the S3 bucket/artifact location where CodePipeline stored the build output
5. **Check disk space** — "Install" often fails silently if `/opt` partition is full

---

### Q36. What is the difference between Blue-Green and In-Place (Rolling) deployment in CodeDeploy?
**Answer:**
- **In-Place** — CodeDeploy stops the app on existing instances, deploys new version, restarts — instances are reused. Downtime per instance during deployment (mitigated by deploying to subsets at a time via deployment config like `OneAtATime` or `HalfAtATime`).
- **Blue-Green** — CodeDeploy provisions NEW instances (green) with the new version, registers them with the load balancer, and only after they're healthy, deregisters/terminates the OLD instances (blue). Zero downtime, instant rollback (just re-point traffic to blue), but requires 2x capacity temporarily and only works with ASG/ELB-integrated deployments.

---

# 10. LAMBDA & SERVERLESS

### Q37. Lambda cold starts — what causes them and how do you mitigate?
**Answer:** A "cold start" happens when Lambda needs to initialize a new execution environment (container) — download code, start runtime, run init code outside the handler — before processing the first request. Subsequent requests reuse the warm environment (no cold start) until it's recycled (idle timeout, ~ 5-15 min, or scaling up needs more concurrent environments).

**Mitigation:**
- **Provisioned Concurrency** — pre-initializes a specified number of environments, eliminating cold starts for that capacity (cost: pay for idle provisioned capacity)
- Reduce package size (remove unused dependencies, use Lambda Layers for shared libs)
- Choose faster-starting runtimes (Go/Node generally faster than Java/.NET for cold start)
- Move heavy initialization (DB connections, SDK clients) OUTSIDE the handler (in global scope) so it's reused across warm invocations
- For VPC-attached Lambdas, ensure ENI creation isn't adding latency (improved significantly in recent years with Hyperplane ENIs, but still a factor)

---

### Q38. SCENARIO: A Lambda function intermittently times out when calling an RDS database. What are the likely causes?
**Answer:**
1. **No connection pooling** — each invocation creates a new DB connection; under concurrent load, hits `max_connections` limit on RDS, new connections hang/timeout. **Fix:** use RDS Proxy.
2. **VPC configuration** — Lambda in a VPC needs ENI setup to reach RDS in private subnet; if subnet has no available IPs or NAT for other calls (e.g., to Secrets Manager for credentials), can cause delays
3. **Lambda timeout setting too low** vs actual query time under load — increase timeout AND investigate slow queries
4. **Lambda concurrency limit reached** — if account/function concurrency limit is hit, invocations throttle (429) rather than timeout, but worth checking `Throttles` metric alongside `Duration`
5. **Security group on RDS not allowing the Lambda's security group**

---

### Q39. When would you choose Lambda vs ECS/EKS vs EC2 for a workload? (Senior trade-off question)
**Answer:**
- **Lambda** — event-driven, short-running (<15 min), spiky/unpredictable traffic, pay-per-execution makes sense for low/intermittent usage. Avoid for: long-running processes, workloads needing consistent low-latency (cold starts), heavy/complex multi-service apps (becomes unmanageable as "everything is a function").
- **ECS/EKS (containers)** — consistent moderate-to-high traffic, microservices needing more control over runtime/dependencies, long-running processes, need for fine-grained scaling per service.
- **EC2** — full OS control needed (custom kernels, licensing requirements, GPU workloads, legacy apps not container-friendly), or cost-optimization at very high sustained scale with Reserved/Spot pricing.

**Key senior framing:** "I'd start by asking: is this event-driven and short? Lambda. Is it a long-running service with predictable load? Containers. Do we need OS-level control or licensing constraints? EC2."

---

# 11. COST OPTIMIZATION

### Q40. Reserved Instances vs Savings Plans vs Spot Instances — explain trade-offs.
**Answer:**
- **Reserved Instances (RI)** — commit to specific instance type/region for 1-3 years for discount (up to ~72%); less flexible (tied to instance family/region, though Convertible RIs allow some changes)
- **Savings Plans** — commit to a $/hour spend for 1-3 years, FLEXIBLE across instance families/regions/even compute types (EC2, Fargate, Lambda) — generally more flexible than RIs for similar discounts
- **Spot Instances** — up to 90% discount, but AWS can reclaim with 2-minute warning — for fault-tolerant, interruptible workloads (batch processing, CI/CD runners, stateless web tier behind ASG with diverse instance types)

**Practical strategy:** Savings Plans for baseline steady-state load, Spot for batch/non-critical/interruptible workloads, On-Demand for unpredictable spikes on top.

---

### Q41. SCENARIO: Your monthly AWS bill suddenly doubled with no obvious change in traffic. How do you investigate?
**Answer:**
1. **AWS Cost Explorer** — filter by service, identify which service's cost increased, group by linked account/tag if multi-account
2. **Check for orphaned resources** — unattached EBS volumes (still billed after instance termination if `DeleteOnTermination=false`), unused Elastic IPs (charged when NOT attached to a running instance), old snapshots accumulating
3. **Data transfer costs** — cross-AZ or cross-region transfer is often the hidden culprit (e.g., a misconfigured app suddenly routing traffic cross-AZ)
4. **NAT Gateway data processing charges** — high outbound traffic through NAT (e.g., a misconfigured backup job pulling large data through NAT instead of via VPC endpoint)
5. **CloudTrail + Cost Anomaly Detection** — set up AWS Cost Anomaly Detection going forward to catch this automatically
6. **Check for a new/forgotten resource** — someone spun up a large instance type for testing and forgot to terminate it

---

# 12. SECURITY & DISASTER RECOVERY

### Q42. Explain the 4 DR strategies: Backup & Restore, Pilot Light, Warm Standby, Multi-Site Active-Active.
**Answer:** (Ordered by increasing cost and decreasing RTO/RPO)
- **Backup & Restore** — backups stored (e.g., S3, snapshots), restore infrastructure from scratch during disaster. Cheapest, highest RTO (hours).
- **Pilot Light** — core infrastructure (e.g., DB) replicated and kept running at minimal size in DR region; other components provisioned only when needed. Moderate RTO (10s of minutes).
- **Warm Standby** — scaled-down but FULLY FUNCTIONAL copy running in DR region at all times; scale up during failover. Lower RTO (minutes).
- **Multi-Site Active-Active** — full production capacity running in multiple regions simultaneously, traffic distributed (e.g., via Route 53). Near-zero RTO/RPO, highest cost.

**Choose based on:** business RTO/RPO requirements vs cost — most companies use Pilot Light or Warm Standby as the sweet spot.

---

### Q43. SCENARIO: Security team flags that an S3 bucket containing sensitive data was publicly accessible for 2 days. Walk through your incident response.
**Answer:**
1. **Immediate containment:** Apply S3 Block Public Access at the bucket (and account) level immediately
2. **Assess exposure:** Use CloudTrail to check `GetObject`/`ListBucket` calls during the exposure window — identify if/what was accessed and from which IPs (especially unfamiliar/external IPs)
3. **Root cause:** Check CloudTrail for the API call that made it public — was it a manual S3 console change, a Terraform/CloudFormation misconfiguration, or an IAM policy change?
4. **Remediate root cause:** Fix the IaC/policy that caused it; add **AWS Config rules** (e.g., `s3-bucket-public-read-prohibited`) to detect/auto-remediate future occurrences
5. **Notify stakeholders** per company incident process — if PII/regulated data was exposed, legal/compliance involvement may be required
6. **Post-incident:** Add automated guardrails (SCP denying public bucket ACLs at the Org level), review IAM permissions that allowed the change

---

### Q44. What is KMS and how does envelope encryption work?
**Answer:** KMS (Key Management Service) manages encryption keys. **Envelope encryption**: instead of encrypting large data directly with the KMS key (KMS has size limits and API call costs), AWS:
1. Generates a unique **Data Encryption Key (DEK)** per object
2. Encrypts the actual data with the DEK (fast, local operation)
3. Encrypts the DEK itself with the KMS **Customer Master Key (CMK)** — produces an "encrypted DEK"
4. Stores the encrypted DEK alongside the encrypted data

To decrypt: call KMS to decrypt the small encrypted DEK (using the CMK), then use the plaintext DEK to decrypt the actual data locally. This is how S3 SSE-KMS, EBS encryption, etc. work under the hood — efficient and scalable.

---

# 13. TROUBLESHOOTING SCENARIO BANK (Multi-Service)

### Q45. SCENARIO: Users report the website is down. `curl` to the ALB DNS name times out. Walk through your full diagnostic tree.
**Answer:**
1. **Check ALB itself** — console shows ALB state "active"? Any recent changes (security group, listener rules)?
2. **ALB Security Group** — allows inbound 80/443 from `0.0.0.0/0` (or expected source)?
3. **Target Group health** — are targets healthy? If ALL unhealthy → 503 from ALB (not a timeout) — so a full timeout suggests ALB itself isn't reachable
4. **Route 53** — does the DNS record actually point to this ALB? `dig`/`nslookup` the domain
5. **VPC/Subnet** — ALB's subnets — are they correctly configured with routes to IGW (for internet-facing ALB)?
6. **NACL** on ALB's subnets — stateless rules blocking traffic?
7. **If ALB is reachable but app is down** — check target group health checks, then `kubectl`/ECS task status, then app logs
8. **CloudTrail** — any recent infrastructure changes (someone modified SG/route table) around the time issue started?

**Interviewer Notes:** This is THE classic "walk me through your debugging" question — they want a systematic OUTER-TO-INNER (DNS → LB → Network → Compute → App) approach, not random guessing.

---

### Q46. SCENARIO: An application deployed via Terraform suddenly can't be updated — `terraform apply` fails with "resource already exists" for resources that ARE in the state file. What happened and how do you fix?
**Answer:** Likely **state drift** — someone manually created/modified the resource outside Terraform, OR the state file is out of sync with actual infrastructure (e.g., state file was corrupted/reverted to an older version, or a previous `apply` partially failed and state wasn't updated correctly).

**Fix:**
1. `terraform plan` — see exact diff causing the conflict
2. `terraform state list` / `terraform state show <resource>` — compare state vs actual AWS resource
3. If resource exists in AWS but not properly tracked: `terraform import <resource_address> <resource_id>` to bring it under management
4. If state file itself is corrupted: restore from S3 versioning (if using S3 backend with versioning enabled — **always enable this**)
5. **Prevention:** enable state locking (DynamoDB table with S3 backend) to prevent concurrent applies causing partial state corruption; restrict direct console access via SCPs for resources managed by IaC

---

### Q47. SCENARIO: A scheduled Lambda function (via EventBridge) stopped running 3 days ago with no error notifications. How do you investigate?
**Answer:**
1. **Check EventBridge rule status** — is it ENABLED? (Someone might have disabled it during testing/debugging and forgot to re-enable)
2. **Check Lambda's resource-based policy** — EventBridge needs `lambda:InvokeFunction` permission; if recently changed (e.g., via a security tightening sweep), invocation could fail silently
3. **CloudTrail** — search for `PutRule`/`DisableRule`/`RemovePermission` events around 3 days ago — likely human or automated change
4. **CloudWatch Logs for the Lambda** — if no new log streams in 3 days, the function isn't being invoked at all (rule/permission issue) vs if log streams exist but show errors, it's an execution issue
5. **Why no alert?** — likely no CloudWatch Alarm was configured on `Invocations`/`Errors` metrics for this function — **add one**: alarm if `Invocations == 0` over the expected schedule period (catches silent failures like this)

---

### Q48. SCENARIO: ECS Fargate tasks keep getting stopped with reason "OutOfMemoryError: Container killed due to memory usage". But CloudWatch shows memory utilization at only 60%. Why the discrepancy?
**Answer:** CloudWatch's reported memory utilization is often an AVERAGE or sampled at intervals — a brief SPIKE to 100% between sampling points can trigger an OOM kill without showing in the averaged metric.

Also check:
1. **Container-level vs Task-level memory** — in Fargate, you set memory at both task AND container level; if container limit is lower than task limit and the container alone spikes, it gets killed even if task-level utilization looks fine
2. **Memory reservation vs hard limit** — if `memory` (hard limit) is set lower than what the app needs under peak load, even if AVERAGE usage is 60%
3. Enable **Container Insights** for more granular, per-second memory metrics to catch spikes
4. Check application logs right before the kill timestamp for clues (large request processed, batch job, GC pause causing memory spike in JVM apps)

---

# 14. ARCHITECTURE / SYSTEM DESIGN SCENARIOS

### Q49. Design a highly available, scalable 3-tier web application architecture on AWS. Walk through your choices.
**Answer:**
"I'd design across 2+ AZs for HA:

**Web/Presentation tier:** Route 53 → CloudFront (CDN, caches static content, reduces origin load) → ALB (internet-facing, spans 2+ AZs) → ASG of EC2 (or ECS/EKS) running the web app, in public subnets

**App tier:** Internal ALB → ASG of app servers in private subnets across 2+ AZs, auto-scaled based on CPU/custom metrics

**Data tier:** RDS Multi-AZ (for HA) + Read Replicas (for read scaling) in private subnets (isolated, no direct internet route); ElastiCache (Redis) for session storage/caching to keep app tier stateless

**Cross-cutting:**
- S3 for static assets, user uploads
- Secrets Manager for DB credentials (rotated automatically)
- CloudWatch + CloudTrail for monitoring/audit
- WAF on CloudFront/ALB for security
- All resources tagged for cost allocation

I'd start with the constraint: what's the expected traffic and RTO/RPO requirement? That determines whether Multi-AZ alone suffices or we need multi-region."

---

### Q50. SCENARIO: Design a system to process 10,000 image uploads/minute, resize them, and store results — cost-effectively.
**Answer:**
"Event-driven serverless architecture fits well here given the bursty, parallelizable nature:

1. User uploads image directly to S3 (using presigned URLs — avoids routing large files through app servers)
2. S3 `ObjectCreated` event triggers Lambda (or SQS for buffering if Lambda concurrency limits are a concern at 10K/min)
3. Lambda resizes the image (using a layer with image processing libs like Sharp/Pillow) and writes the result to a separate output S3 bucket
4. SQS as a buffer between S3 events and Lambda if we need to control concurrency/avoid throttling downstream services, with a Dead Letter Queue for failed processing
5. DynamoDB to track processing status/metadata (fast, scales automatically)

**Cost consideration:** Lambda's pay-per-invocation model fits well for bursty/intermittent load — if this were CONSTANT 24/7 high volume, I'd reconsider Fargate/ECS for better cost-per-unit at sustained scale. I'd benchmark both based on actual sustained vs peak patterns before committing."

**Interviewer Notes:** Senior interviewers want the "it depends, here's my reasoning and what I'd validate" framing — not a single rigid answer.

---

# 15. QUICK-FIRE ROUND (Answer in under 20 seconds each)

| Q | A |
|---|---|
| Default VPC CIDR? | 172.31.0.0/16 (varies, but this is the AWS default) |
| What is an Elastic IP and what's the gotcha? | Static public IP; charged if allocated but NOT attached to a running instance |
| Difference: Security Group vs NACL stateful/stateless? | SG = stateful (return traffic auto-allowed); NACL = stateless (must allow both directions explicitly) |
| What does `DeleteOnTermination` control? | Whether the EBS volume is deleted when the EC2 instance is terminated |
| S3 consistency model? | Strong read-after-write consistency for all operations (since Dec 2020) |
| What is a Launch Template vs Launch Configuration? | Launch Template is the modern replacement — supports versioning, mixed instance types, Spot+On-Demand combos; Launch Configurations are legacy/immutable |
| What's the max Lambda execution timeout? | 15 minutes |
| CloudFront origin types? | S3, ALB/EC2, custom HTTP origin, MediaStore |
| What is an ENI? | Elastic Network Interface — virtual network card attachable to EC2 instances |
| Difference: `aws s3 sync` vs `aws s3 cp --recursive`? | `sync` only transfers changed/new files (delta); `cp --recursive` copies everything every time |

---

# 16. TOPIC WEIGHTAGE MAP — AWS for IBM/Amazon/Flipkart/Google DevOps Loops

| Topic | Frequency | Format | Why Tested |
|---|---|---|---|
| **IAM (policies, roles, troubleshooting access)** | Very High | Conceptual + scenario | Security-first culture at all 4 companies; IAM debugging is a daily DevOps task |
| **VPC/Networking (routing, SG/NACL, connectivity debugging)** | Very High | Scenario, whiteboard | Networking issues are the #1 real production fire |
| **EC2 fundamentals + troubleshooting** | High | Scenario | Baseline compute knowledge expected |
| **S3 (storage classes, versioning quirks, large file handling)** | High | Conceptual + scenario | Universal service, lots of edge cases |
| **RDS (Multi-AZ vs Read Replica, connection issues)** | High | Scenario | DB connectivity issues are extremely common in interviews and production |
| **Load Balancing & Auto Scaling (health checks, flapping)** | High | Scenario | Core to HA architecture questions |
| **CloudWatch/CloudTrail** | Medium-High | Conceptual + scenario | Observability is assumed baseline |
| **CI/CD (CodePipeline/CodeDeploy)** | Medium | Conceptual + scenario | More relevant if pipeline ownership is in scope |
| **Lambda/Serverless trade-offs** | Medium-High at Amazon | "When NOT to use X" conceptual | Tests architectural judgment, not just service knowledge |
| **Cost Optimization** | Medium-High, esp. senior | Scenario ("bill doubled") | Every company cares about cloud cost in 2026 |
| **DR/Security incident response** | Medium-High at Amazon/IBM | Scenario | Compliance-heavy orgs test incident-response thinking |
| **Architecture/System Design (3-tier, event-driven)** | High at senior level | Open-ended design | Tests ability to reason about trade-offs, not memorize |

### Interview flow pattern (consistent across all 4):
1. **IAM/security warm-up** (2-3 conceptual Qs) — gauges security awareness
2. **Networking deep-dive** — VPC connectivity scenario, often whiteboard/diagram
3. **Compute/storage/DB troubleshooting** — 1-2 "service is down" scenarios, testing debugging tree
4. **Cost or DR scenario** — open-ended, tests trade-off reasoning
5. **System design** (senior loops) — design a small system end-to-end, justify choices

**Highest-leverage prep:** Section 13 (Q45 especially — the outer-to-inner debugging tree applies to almost ANY "X is down" question) + Section 1 (IAM, especially Q2 and Q4) + Section 14 (architecture reasoning).

---

# PRACTICE INSTRUCTIONS (15-Day Track, parallel with K8s prep)

1. **Day 1-2:** IAM (Q1-Q5) — especially Q2 (policy evaluation) and Q4 (multi-layer AccessDenied debugging)
2. **Day 3-4:** EC2 + VPC (Q6-Q15) — focus on Q8, Q12, Q14 (connectivity debugging trees)
3. **Day 5-6:** S3/EBS/EFS + RDS (Q16-Q24) — focus on Q18 (versioning gotcha), Q22 (connection pooling)
4. **Day 7:** Load Balancing/ASG + Route 53 (Q25-Q30) — focus on Q26, Q28 (flapping ASG)
5. **Day 8:** CloudWatch/CloudTrail + CI/CD (Q31-Q36)
6. **Day 9:** Lambda + Cost Optimization (Q37-Q41) — especially Q39 (when NOT to use Lambda) and Q41 (bill doubled)
7. **Day 10:** Security/DR (Q42-Q44)
8. **Day 11-12:** Troubleshooting Bank (Q45-Q48) — drill Q45 (outer-to-inner tree) until automatic, it's a template for almost any "X is down" question
9. **Day 13:** Architecture/System Design (Q49-Q50) — practice talking through designs out loud, sketch on paper
10. **Day 14:** Quick-fire round + Topic Weightage Map — rehearse the 5-stage flow
11. **Day 15:** Full mock — random Qs from all 50, mix with K8s questions since interviewers often blend "how does this K8s app connect to that RDS" style cross-topic questions

**Self-check:** For every scenario answer, ask "am I going outer-to-inner (DNS→LB→Network→Compute→App) or do I have SOME systematic order?" Random guessing is the #1 thing that fails candidates with good theoretical knowledge.
