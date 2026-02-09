# AWS Interview Questions

## Table of Contents
- [EC2 & Compute](#ec2--compute)
- [S3 & Storage](#s3--storage)
- [VPC & Networking](#vpc--networking)
- [IAM & Security](#iam--security)
- [Lambda & Serverless](#lambda--serverless)
- [RDS & Databases](#rds--databases)
- [CloudWatch & Monitoring](#cloudwatch--monitoring)

---

## EC2 & Compute

### Q1: What is the difference between EC2 instance types and when would you choose one over another?

**Difficulty:** Mid

**Answer:**

EC2 instance types are optimized for different use cases:

- **General Purpose (t3, m5)**: Balanced compute, memory, and networking. Good for web servers, small databases, development environments.
- **Compute Optimized (c5, c6i)**: High-performance processors. Ideal for compute-intensive workloads like batch processing, gaming servers, HPC.
- **Memory Optimized (r5, x1e)**: High memory-to-vCPU ratio. Perfect for in-memory databases (Redis), real-time big data analytics, high-performance databases.
- **Storage Optimized (i3, d2)**: High sequential read/write performance. Best for NoSQL databases, data warehousing, log processing.
- **Accelerated Computing (p3, g4)**: GPUs and FPGAs. Used for machine learning, graphics rendering, video encoding.

**Real-world Context:** Choose based on workload characteristics. A web application might use t3.medium, while a Redis cache cluster would use r5.large.

**Follow-up:** How do you determine the right instance size? (Use CloudWatch metrics, load testing, start small and scale)

---

### Q2: Explain the difference between EBS volume types and their use cases.

**Difficulty:** Mid

**Answer:**

EBS volume types differ in performance characteristics:

- **gp3 (General Purpose SSD)**: Baseline 3,000 IOPS, up to 16,000 IOPS. Default choice for most workloads. Cost-effective with consistent performance.
- **gp2 (General Purpose SSD)**: Performance scales with size (3 IOPS/GB). Good for boot volumes and small databases.
- **io1/io2 (Provisioned IOPS SSD)**: Up to 64,000 IOPS. For I/O-intensive databases (Oracle, SQL Server) requiring consistent low latency.
- **st1 (Throughput Optimized HDD)**: 500 MB/s throughput. For big data, data warehouses, log processing. Cannot be boot volume.
- **sc1 (Cold HDD)**: Lowest cost. For infrequently accessed data, archives. Cannot be boot volume.

**Real-world Context:** Production database → io2. Web server boot volume → gp3. Log aggregation → st1.

**Follow-up:** How do you migrate from gp2 to gp3? (Create snapshot, restore to gp3, or modify volume type)

---

### Q3: What is an Auto Scaling Group and how does it work?

**Difficulty:** Mid

**Answer:**

An Auto Scaling Group (ASG) automatically adjusts the number of EC2 instances based on demand or schedules.

**Components:**
- **Launch Template/Configuration**: Defines what instances to launch (AMI, instance type, security groups)
- **Scaling Policies**: Rules for scaling (target tracking, step scaling, simple scaling)
- **Health Checks**: EC2 or ELB health checks to replace unhealthy instances
- **Cooldown Period**: Prevents rapid scaling actions

**How it works:**
1. CloudWatch alarms trigger scaling policies
2. ASG launches/terminates instances based on metrics (CPU, memory, custom)
3. New instances are registered with load balancer
4. Unhealthy instances are replaced automatically

**Real-world Context:** Web application behind ALB. Scale from 2 to 10 instances during peak hours based on CPU utilization.

**Follow-up:** What's the difference between target tracking and step scaling? (Target tracking maintains a metric at target value, step scaling uses multiple thresholds)

---

### Q4: Explain VPC, Subnets, Internet Gateway, and NAT Gateway.

**Difficulty:** Mid

**Answer:**

**VPC (Virtual Private Cloud)**: Isolated network environment in AWS. You control IP ranges, subnets, routing, security.

**Subnets**: Logical subdivision of VPC IP range. Can be public (route to IGW) or private (route to NAT).

**Internet Gateway (IGW)**: Allows communication between VPC and internet. One per VPC. Provides 1:1 NAT for public IPs.

**NAT Gateway**: Allows private subnets to access internet for updates/downloads while remaining inaccessible from internet. Managed service, highly available in one AZ (or use NAT Instance for multi-AZ).

**Architecture:**
- Public Subnet: Route table → 0.0.0.0/0 → IGW
- Private Subnet: Route table → 0.0.0.0/0 → NAT Gateway

**Real-world Context:** Web servers in public subnets, databases in private subnets. NAT Gateway allows DB instances to download patches.

**Follow-up:** What's the difference between NAT Gateway and NAT Instance? (NAT Gateway is managed, scales automatically, more expensive. NAT Instance is EC2-based, requires management, cheaper for low traffic)

---

### Q5: What is the difference between Security Groups and NACLs?

**Difficulty:** Mid

**Answer:**

**Security Groups:**
- Stateful firewall (return traffic automatically allowed)
- Applied at instance level
- Rules: allow only (default deny all)
- Evaluates all rules before deciding
- Can reference other security groups

**NACLs (Network ACLs):**
- Stateless firewall (must allow both inbound and outbound)
- Applied at subnet level
- Rules: allow and deny (evaluated in order, first match wins)
- Can block specific IPs
- Default: allows all traffic

**Use Cases:**
- Security Groups: Primary defense, instance-level protection
- NACLs: Subnet-level protection, compliance requirements, blocking specific IPs

**Real-world Context:** Security Groups protect instances. NACLs add extra layer - block known malicious IPs at subnet level.

**Follow-up:** If you block port 80 in NACL but allow it in Security Group, what happens? (Traffic is blocked - NACL is evaluated first)

---

## S3 & Storage

### Q6: Explain S3 storage classes and when to use each.

**Difficulty:** Mid

**Answer:**

**S3 Standard**: 99.99% availability, 99.999999999% durability. For frequently accessed data. Low latency, high throughput.

**S3 Intelligent-Tiering**: Automatically moves objects between access tiers. For unpredictable access patterns. Small monitoring fee.

**S3 Standard-IA (Infrequent Access)**: Lower cost for infrequently accessed data. 99.9% availability. Minimum 30-day storage, retrieval fee.

**S3 One Zone-IA**: Like Standard-IA but stored in single AZ. 99.5% availability. 20% cheaper. For non-critical, reproducible data.

**S3 Glacier Instant Retrieval**: Archive with millisecond retrieval. For rarely accessed data requiring immediate access. Minimum 90-day storage.

**S3 Glacier Flexible Retrieval**: 3 retrieval options (expedited 1-5 min, standard 3-5 hours, bulk 5-12 hours). For archives, backups.

**S3 Glacier Deep Archive**: Lowest cost. 12-hour retrieval. For long-term compliance archives.

**Real-world Context:** Active application data → Standard. Logs older than 30 days → Standard-IA. Compliance archives → Glacier Deep Archive.

**Follow-up:** How does lifecycle policy work? (Automatically transitions objects between classes based on age/prefix)

---

### Q7: What is S3 versioning and why is it important?

**Difficulty:** Junior

**Answer:**

S3 versioning keeps multiple versions of an object with the same key. Each version has a unique version ID.

**Benefits:**
- Protect against accidental deletion (can restore previous version)
- Recover from overwrites
- Maintain object history
- Compliance requirements

**How it works:**
- When versioning enabled, PUT creates new version (or new version ID)
- DELETE creates delete marker (object appears deleted but versions remain)
- Can restore by deleting delete marker or copying previous version

**Real-world Context:** Developer accidentally overwrites production config file. With versioning, restore previous version immediately.

**Follow-up:** How do you enable versioning on existing bucket? (Enable versioning, existing objects become version with null version ID)

---

### Q8: Explain S3 cross-region replication and its use cases.

**Difficulty:** Mid

**Answer:**

S3 Cross-Region Replication (CRR) automatically replicates objects to another region.

**Requirements:**
- Versioning enabled on source and destination buckets
- IAM permissions for replication
- Source and destination buckets in different regions

**Use Cases:**
- **Disaster Recovery**: Backup in another region
- **Compliance**: Data residency requirements
- **Low Latency**: Copy data closer to users
- **Data Migration**: Move data between regions

**How it works:**
- Configure replication rule (source prefix, destination bucket, IAM role)
- New objects matching rule are automatically replicated
- Existing objects not replicated (unless use S3 Batch Replication)

**Real-world Context:** GDPR requires EU data in EU region. Replicate from us-east-1 to eu-west-1.

**Follow-up:** What happens if you delete an object? (Delete marker is replicated, object appears deleted in both regions)

---

## VPC & Networking

### Q9: What is a VPC Peering connection and how does it work?

**Difficulty:** Mid

**Answer:**

VPC Peering connects two VPCs using private IP addresses, as if they're in the same network.

**Characteristics:**
- One-to-one connection (not transitive)
- Can peer VPCs in same or different accounts/regions
- No single point of failure
- No bandwidth bottleneck
- CIDR blocks must not overlap

**Setup:**
1. Request peering connection (initiator)
2. Accept peering connection (accepter)
3. Update route tables in both VPCs
4. Update security groups/NACLs to allow traffic

**Use Cases:**
- Connect VPCs in same region
- Share resources between accounts
- Hub-and-spoke architecture (each spoke peers with hub)

**Real-world Context:** Development VPC needs access to shared services VPC (databases, monitoring).

**Follow-up:** How do you connect 3 VPCs? (Need 3 peering connections - not transitive. Or use Transit Gateway)

---

### Q10: What is AWS Transit Gateway and when would you use it?

**Difficulty:** Senior

**Answer:**

Transit Gateway is a network transit hub that connects VPCs, VPNs, and Direct Connect.

**Benefits:**
- Centralized management
- Transitive routing (VPC A → TGW → VPC B)
- Supports up to 5,000 VPCs
- Route tables for segmentation
- Cross-region peering

**Use Cases:**
- Hub-and-spoke architecture
- Connecting many VPCs (simpler than VPC peering mesh)
- Centralized network security inspection
- Multi-account networking

**Architecture:**
- Create Transit Gateway
- Attach VPCs/VPNs
- Configure route tables
- Set up routing (propagate or static routes)

**Real-world Context:** Organization with 50 VPCs. Instead of 1,225 peering connections, use Transit Gateway with one attachment per VPC.

**Follow-up:** How do you implement network segmentation with Transit Gateway? (Use multiple route tables, attach VPCs to different route tables)

---

## IAM & Security

### Q11: Explain IAM roles vs IAM users and when to use each.

**Difficulty:** Mid

**Answer:**

**IAM Users:**
- Long-term credentials (access keys, passwords)
- For humans or applications that need long-term access
- Can have MFA enabled
- Best for: Developers, CI/CD systems, service accounts

**IAM Roles:**
- Temporary credentials (assumed via STS)
- No passwords or access keys
- Can be assumed by users, services, or other roles
- Best for: EC2 instances, Lambda functions, cross-account access

**Key Differences:**
- Roles provide temporary credentials (1 hour default, up to 12 hours)
- Roles can be assumed by multiple entities
- Roles support cross-account access
- Users have permanent credentials (until rotated)

**Best Practice:** Use roles for everything possible. Only use users when absolutely necessary.

**Real-world Context:** EC2 instance needs S3 access → IAM Role attached to instance. Developer needs console access → IAM User with MFA.

**Follow-up:** How do you assume a role? (Use AWS STS AssumeRole API, or attach role to service)

---

### Q12: What is the principle of least privilege in IAM?

**Difficulty:** Junior

**Answer:**

Principle of least privilege: Grant only the minimum permissions necessary to perform a task.

**Implementation:**
- Start with no permissions
- Add permissions only as needed
- Use specific actions, not wildcards (*)
- Scope to specific resources (ARNs), not all resources
- Use conditions (IP, time, tags) to further restrict

**Example:**
- ❌ Bad: `s3:*` on `*`
- ✅ Good: `s3:GetObject` on `arn:aws:s3:::my-bucket/data/*`

**Benefits:**
- Reduces attack surface
- Limits impact of compromised credentials
- Easier to audit and understand
- Compliance requirements

**Real-world Context:** Lambda function only needs to read from one S3 bucket. Grant `s3:GetObject` on that bucket only, not `s3:*`.

**Follow-up:** How do you audit IAM permissions? (Use IAM Access Analyzer, CloudTrail, or IAM policy simulator)

---

### Q13: Explain IAM policy evaluation logic.

**Difficulty:** Senior

**Answer:**

IAM evaluates policies in this order:

1. **Default Deny**: All requests denied by default
2. **Explicit Deny**: Any explicit Deny overrides everything
3. **Explicit Allow**: Grants permission
4. **Default Deny**: If no Allow, request denied

**Evaluation Process:**
- Check all applicable policies (identity-based, resource-based, boundary, service control)
- If any policy has explicit Deny → Deny
- If any policy has explicit Allow → Allow
- Otherwise → Deny

**Key Points:**
- Explicit Deny always wins
- Need at least one Allow to proceed
- Resource-based policies evaluated separately
- Permissions boundaries limit maximum permissions

**Real-world Context:** User has Allow in user policy, but Deny in group policy → Request denied (explicit Deny wins).

**Follow-up:** What's the difference between permissions boundary and policy? (Boundary sets maximum permissions, policy grants permissions within boundary)

---

## Lambda & Serverless

### Q14: What are Lambda cold starts and how do you minimize them?

**Difficulty:** Mid

**Answer:**

**Cold Start**: Time to initialize Lambda execution environment (download code, initialize runtime, run initialization code).

**Factors Affecting Cold Starts:**
- Runtime (Python/Node.js faster than Java/.NET)
- Package size (larger = slower)
- VPC configuration (adds ENI setup time)
- Provisioned Concurrency (eliminates cold starts)

**Minimization Strategies:**
1. **Optimize package size**: Remove unused dependencies, use layers for common code
2. **Choose faster runtime**: Node.js/Python over Java
3. **Avoid VPC if possible**: Adds 10+ seconds
4. **Use Provisioned Concurrency**: Pre-warms execution environments
5. **Optimize initialization**: Move heavy initialization outside handler
6. **Keep functions warm**: CloudWatch Events ping function periodically

**Real-world Context:** API Gateway → Lambda. Cold start adds 2-3 seconds. Use Provisioned Concurrency for production APIs.

**Follow-up:** When would you use Provisioned Concurrency? (When you need consistent low latency and can predict traffic)

---

### Q15: Explain Lambda concurrency and reserved concurrency.

**Difficulty:** Mid

**Answer:**

**Concurrency**: Number of execution environments running simultaneously.

**Default Limits:**
- Account-level: 1,000 concurrent executions (can increase)
- Per function: Unlimited (unless reserved concurrency set)

**Reserved Concurrency:**
- Guarantees minimum concurrency for a function
- Limits maximum concurrency for a function
- Other functions cannot use reserved concurrency
- Prevents one function from consuming all account concurrency

**Use Cases:**
- **Reserve concurrency**: Critical function needs guaranteed capacity
- **Limit concurrency**: Function that calls downstream API with rate limits
- **Cost control**: Limit expensive function execution

**Throttling:**
- When concurrency limit reached, requests throttled (429 error)
- Can configure DLQ for throttled events
- Use reserved concurrency to prevent throttling

**Real-world Context:** Lambda calls external API with 100 req/sec limit. Set reserved concurrency to 100 to prevent exceeding API limit.

**Follow-up:** What happens when account concurrency limit is reached? (All functions throttled unless reserved concurrency set)

---

### Q16: What is Lambda@Edge and what are its use cases?

**Difficulty:** Senior

**Answer:**

Lambda@Edge runs Lambda functions at CloudFront edge locations, closer to users.

**Use Cases:**
- **Request/Response Manipulation**: Modify headers, redirects, A/B testing
- **Authentication/Authorization**: Validate requests before origin
- **Custom Error Pages**: Generate custom error responses
- **Geographic Personalization**: Serve different content by location
- **Bot Detection**: Block malicious requests at edge

**Limitations:**
- Node.js and Python runtimes only
- Smaller package size (1 MB for viewer, 50 MB for origin)
- Shorter execution time (viewer: 5s, origin: 30s)
- Limited access to AWS services

**Execution Points:**
- Viewer Request: Before CloudFront forwards to origin
- Origin Request: Before CloudFront forwards to origin (can cache)
- Origin Response: After origin responds
- Viewer Response: Before CloudFront responds to viewer

**Real-world Context:** Redirect mobile users to mobile site, add security headers, implement geo-blocking.

**Follow-up:** What's the difference between Viewer Request and Origin Request? (Viewer Request runs on every request, Origin Request runs before cache check)

---

## RDS & Databases

### Q17: Explain RDS Multi-AZ and Read Replicas.

**Difficulty:** Mid

**Answer:**

**Multi-AZ:**
- Synchronous replication to standby in different AZ
- Automatic failover (60-120 seconds)
- High availability, not scalability
- Same endpoint (DNS switches on failover)
- Use for: Production databases requiring HA

**Read Replicas:**
- Asynchronous replication (eventual consistency)
- Can be in different region
- Read scaling (offload read traffic)
- Manual promotion to primary
- Use for: Read-heavy workloads, disaster recovery, cross-region replication

**Key Differences:**
- Multi-AZ: HA, synchronous, automatic failover
- Read Replicas: Scalability, asynchronous, manual promotion

**Can Combine:** Multi-AZ primary with Read Replicas for both HA and read scaling.

**Real-world Context:** E-commerce site. Multi-AZ for primary DB (HA), Read Replicas for reporting/analytics (read scaling).

**Follow-up:** What's the RTO/RPO for Multi-AZ? (RTO: 60-120s, RPO: 0 - no data loss)

---

### Q18: What is RDS Proxy and why would you use it?

**Difficulty:** Senior

**Answer:**

RDS Proxy is a fully managed database connection pooler and proxy.

**Benefits:**
- **Connection Pooling**: Reuses connections, reduces database load
- **Failover Handling**: Handles failovers without application changes
- **IAM Authentication**: Use IAM instead of passwords
- **Query Filtering**: Block or allow specific SQL statements
- **Enhanced Monitoring**: CloudWatch metrics for connections

**Use Cases:**
- Serverless applications (Lambda) that create many connections
- Applications with many concurrent connections
- Need IAM authentication
- Reduce connection overhead

**How it works:**
- Application connects to RDS Proxy endpoint
- Proxy manages connection pool to database
- Reuses connections across Lambda invocations
- Handles failover transparently

**Real-world Context:** Lambda functions connecting to RDS. Without proxy, each invocation creates new connection. With proxy, connections reused.

**Follow-up:** What's the difference between RDS Proxy and connection pooling in application? (RDS Proxy is managed, shared across applications, handles failover)

---

## CloudWatch & Monitoring

### Q19: Explain CloudWatch Logs, Metrics, and Alarms.

**Difficulty:** Mid

**Answer:**

**CloudWatch Logs:**
- Centralized log storage
- Log groups (application) and log streams (instance)
- Retention: 1 day to never
- Log Insights for querying
- Export to S3, stream to Elasticsearch

**CloudWatch Metrics:**
- Time-series data points
- Namespace (AWS/EC2, Custom/MyApp)
- Dimensions (InstanceId, AutoScalingGroupName)
- Standard resolution: 1 minute, High resolution: 1 second
- Custom metrics via PutMetricData API

**CloudWatch Alarms:**
- Monitor metrics and trigger actions
- States: OK, ALARM, INSUFFICIENT_DATA
- Actions: SNS, Auto Scaling, EC2 actions, Lambda
- Evaluation periods, datapoints to alarm

**Real-world Context:** Monitor CPU utilization → CloudWatch Metric. Alert when > 80% → CloudWatch Alarm → SNS → Email. Auto Scale based on alarm → ASG.

**Follow-up:** What's the difference between CloudWatch and CloudTrail? (CloudWatch: metrics/logs, CloudTrail: API calls/audit)

---

### Q20: What is CloudWatch Logs Insights and how do you use it?

**Difficulty:** Mid

**Answer:**

CloudWatch Logs Insights is a query language to search and analyze log data.

**Query Syntax:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
```

**Common Use Cases:**
- Search for errors: `filter @message like /ERROR/`
- Parse JSON logs: `parse @message "user=*," as user`
- Aggregate metrics: `stats count() by statusCode`
- Time-based analysis: `stats count() by bin(5m)`

**Fields:**
- `@timestamp`: Log timestamp
- `@message`: Log message
- `@logStream`: Log stream name
- Custom fields from parsed logs

**Real-world Context:** Application logs contain JSON. Query: `fields @timestamp, level, error | filter level = "ERROR" | stats count() by error`.

**Follow-up:** How do you create a dashboard from Logs Insights query? (Save query, add to dashboard, or export to CloudWatch Metrics)

---

## Summary

These questions cover fundamental AWS concepts across compute, storage, networking, security, serverless, databases, and monitoring. Practice explaining these concepts clearly and relate them to real-world scenarios you've encountered or designed.

**Next Steps:**
- Practice drawing architecture diagrams
- Work through AWS hands-on labs
- Review AWS Well-Architected Framework
- Study for AWS certifications (Solutions Architect, DevOps Engineer)
