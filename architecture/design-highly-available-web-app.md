# Architecture Exercise: Design a Highly Available Web Application

## Problem Statement

Design a highly available web application that can handle 1 million daily active users with 99.99% uptime. The application should be resilient to failures, scalable, and cost-effective.

## Requirements

### Functional Requirements

1. **Web Application**
   - Serve web pages and API requests
   - Handle user authentication
   - Process transactions
   - Serve static assets (images, CSS, JS)

2. **Data Storage**
   - User data (profiles, preferences)
   - Transaction data
   - Session data
   - File uploads (images, documents)

3. **Features**
   - User registration and login
   - Real-time notifications
   - Search functionality
   - Analytics and reporting

### Non-Functional Requirements

1. **Availability**: 99.99% uptime (52.56 minutes downtime/year)
2. **Scalability**: Handle traffic spikes (10x normal load)
3. **Performance**: <200ms response time (p95)
4. **Durability**: Zero data loss
5. **Security**: Encrypt data at rest and in transit, compliance (GDPR, PCI DSS if handling payments)
6. **Cost**: Optimize for cost-effectiveness
7. **Disaster Recovery**: RTO < 1 hour, RPO < 15 minutes

## Constraints and Assumptions

### Constraints

- **Budget**: $50,000/month infrastructure budget
- **Compliance**: Must comply with GDPR
- **Geographic**: Primary users in US and EU
- **Technology**: Prefer managed services (AWS, GCP, Azure)

### Assumptions

- Average request size: 10KB
- Peak traffic: 3x average (evening hours)
- Read:Write ratio: 10:1
- User sessions: Average 30 minutes
- Database size: 500GB, growing 10GB/month
- Static assets: 100GB, CDN cached

## Reference Architecture

### High-Level Architecture

```
                    Internet
                       |
                  [CloudFront CDN]
                       |
            ┌──────────┴──────────┐
            |                     |
      [Route 53]            [WAF]
            |                     |
    ┌───────┴────────┐            |
    |                |            |
[ALB US-EAST]  [ALB EU-WEST]     |
    |                |            |
    └───────┬────────┘            |
            |                     |
    ┌───────┴────────┐            |
    |                |            |
[Auto Scaling]  [Auto Scaling]   |
    |                |            |
┌───┴───┐      ┌───┴───┐         |
| ECS   |      | ECS   |         |
|Tasks  |      |Tasks  |         |
└───┬───┘      └───┬───┘         |
    |              |              |
    └──────┬───────┘              |
           |                      |
    ┌──────┴──────┐               |
    |             |               |
[ElastiCache]  [RDS Multi-AZ]    |
[Redis]        [PostgreSQL]      |
    |             |               |
    └──────┬──────┘               |
           |                      |
    ┌──────┴──────┐               |
    |             |               |
[S3]          [S3]               |
[US]          [EU]               |
```

### Component Breakdown

#### 1. Content Delivery Network (CDN)

**Component**: CloudFront (AWS) / Cloud CDN (GCP)

**Purpose**:
- Cache static assets globally
- Reduce latency for users
- Offload origin servers

**Configuration**:
- Edge locations worldwide
- Cache TTL: 24 hours for static assets
- Compression enabled
- HTTPS only

**Benefits**:
- 80% of requests served from edge (cache hit)
- Reduced origin load
- Lower latency globally

#### 2. DNS and Load Balancing

**Component**: Route 53 (DNS) + Application Load Balancer (ALB)

**Purpose**:
- Route traffic to nearest region
- Health checks and failover
- SSL termination

**Configuration**:
- **Route 53**: 
  - Latency-based routing
  - Health checks every 30s
  - Failover to secondary region
- **ALB**:
  - Cross-zone load balancing
  - SSL certificates (ACM)
  - Path-based routing (/api → API servers, /static → S3)

**High Availability**:
- Multiple AZs per region
- Active-active setup (both regions active)
- Automatic failover

#### 3. Web Application Layer

**Component**: ECS Fargate / EKS (Kubernetes)

**Purpose**:
- Run application containers
- Handle business logic
- Process requests

**Configuration**:
- **Containerization**: Docker containers
- **Orchestration**: ECS Fargate (serverless) or EKS
- **Scaling**: 
  - Min: 4 tasks per region
  - Max: 50 tasks per region
  - Target: 70% CPU utilization
  - Scale up: CPU > 70% for 2 minutes
  - Scale down: CPU < 30% for 10 minutes

**High Availability**:
- Deploy across 3+ AZs
- Health checks (ALB)
- Auto-replace unhealthy tasks
- Rolling deployments

**Cost Optimization**:
- Use Spot instances for non-critical workloads (if EKS)
- Right-size containers
- Use Fargate Spot (60% savings)

#### 4. Caching Layer

**Component**: ElastiCache (Redis)

**Purpose**:
- Session storage
- Frequently accessed data
- Rate limiting
- Real-time features

**Configuration**:
- **Redis Cluster Mode**: 3 shards, 2 replicas each
- **Multi-AZ**: Enabled
- **TTL**: 30 minutes for sessions
- **Memory**: 32GB per node

**High Availability**:
- Automatic failover
- Multi-AZ deployment
- Backup and restore

#### 5. Database Layer

**Component**: RDS PostgreSQL (Multi-AZ)

**Purpose**:
- Primary data storage
- ACID transactions
- Data consistency

**Configuration**:
- **Instance**: db.r5.2xlarge (8 vCPU, 64GB RAM)
- **Storage**: 1TB GP3 SSD, auto-scaling
- **Multi-AZ**: Enabled (synchronous replication)
- **Backups**: 
  - Automated daily backups
  - 7-day retention
  - Point-in-time recovery (35 days)
- **Read Replicas**: 
  - 2 read replicas (1 per region)
  - For read-heavy workloads

**High Availability**:
- Automatic failover (<60s)
- Multi-AZ deployment
- Automated backups
- Read replicas for scaling reads

**Disaster Recovery**:
- Cross-region read replica (EU)
- Can be promoted to primary
- RPO: Near-zero (synchronous replication)
- RTO: <15 minutes (promote read replica)

#### 6. Object Storage

**Component**: S3 (Simple Storage Service)

**Purpose**:
- Static assets (images, documents)
- Backup storage
- Log archival

**Configuration**:
- **Buckets**: 
  - `app-assets-us` (US region)
  - `app-assets-eu` (EU region)
- **Storage Classes**:
  - Standard for frequently accessed
  - Standard-IA for older assets
- **Versioning**: Enabled
- **Replication**: Cross-region replication
- **Lifecycle**: Move to Glacier after 90 days

**High Availability**:
- 99.999999999% durability
- Cross-region replication
- Versioning

#### 7. Monitoring and Logging

**Component**: CloudWatch, X-Ray, ELK Stack

**Purpose**:
- Monitor application health
- Track performance metrics
- Debug issues
- Alert on anomalies

**Configuration**:
- **CloudWatch**:
  - Custom metrics (error rate, latency)
  - Logs aggregation
  - Alarms for critical metrics
- **X-Ray**: Distributed tracing
- **ELK**: Centralized logging (optional, for advanced)

**Alerts**:
- Error rate > 1%
- Response time > 500ms (p95)
- Database connections > 80%
- CPU > 80%
- Disk > 80%

## Discussion Points

### Trade-offs

**1. Multi-Region vs Single Region**

**Multi-Region (Chosen)**:
- ✅ Better latency for global users
- ✅ Disaster recovery
- ✅ Compliance (data residency)
- ❌ Higher cost (2x infrastructure)
- ❌ More complex to manage

**Single Region**:
- ✅ Lower cost
- ✅ Simpler architecture
- ❌ Higher latency for distant users
- ❌ Single point of failure

**Decision**: Multi-region for 99.99% availability requirement.

**2. ECS Fargate vs EKS**

**ECS Fargate (Chosen)**:
- ✅ Serverless (no node management)
- ✅ Simpler
- ✅ Good for most use cases
- ❌ Less control
- ❌ Higher cost at scale

**EKS**:
- ✅ More control
- ✅ Can use Spot instances (cost savings)
- ✅ More flexible
- ❌ More complex
- ❌ Node management required

**Decision**: ECS Fargate for simplicity, but EKS viable for cost optimization.

**3. Database: RDS vs Self-Managed**

**RDS (Chosen)**:
- ✅ Managed (backups, patching, monitoring)
- ✅ Multi-AZ failover
- ✅ Point-in-time recovery
- ❌ Higher cost
- ❌ Less control

**Self-Managed**:
- ✅ Lower cost
- ✅ Full control
- ❌ More operational overhead
- ❌ Need to manage backups, failover

**Decision**: RDS for reliability and reduced operational burden.

### Scalability Considerations

**Horizontal Scaling**:
- Application: Auto-scaling groups
- Database: Read replicas
- Cache: Redis cluster mode

**Vertical Scaling**:
- Database: Upgrade instance size
- Cache: Increase node size

**Caching Strategy**:
- CDN for static assets
- Redis for sessions and hot data
- Application-level caching

### Security Considerations

**Network Security**:
- VPC with private subnets
- Security groups (least privilege)
- WAF for DDoS protection
- VPN for admin access

**Data Security**:
- Encryption at rest (RDS, S3, EBS)
- Encryption in transit (TLS)
- Secrets management (AWS Secrets Manager)
- IAM roles (no hardcoded credentials)

**Application Security**:
- Input validation
- Rate limiting
- Authentication (OAuth 2.0, JWT)
- Regular security scans

**Compliance**:
- GDPR: Data residency (EU region), right to deletion
- PCI DSS: If handling payments, use compliant services
- SOC 2: Use compliant cloud services

### Cost Optimization

**Estimated Monthly Cost** (Rough):

- **Compute (ECS Fargate)**: $2,000 (average 20 tasks)
- **Load Balancers**: $50 (2 ALBs)
- **Database (RDS)**: $1,500 (Multi-AZ + replicas)
- **Cache (ElastiCache)**: $800
- **Storage (S3)**: $100
- **CDN (CloudFront)**: $300
- **Data Transfer**: $500
- **Monitoring**: $200
- **Total**: ~$5,450/month

**Optimization Strategies**:
- Use Spot instances (EKS) for non-critical workloads
- Reserved instances for database (1-year, 40% savings)
- S3 lifecycle policies (move to cheaper storage)
- Right-size instances
- Cache aggressively (reduce database load)

### Disaster Recovery Plan

**RPO (Recovery Point Objective)**: < 15 minutes
- RDS Multi-AZ: Synchronous replication
- S3: Cross-region replication
- Redis: Multi-AZ with automatic failover

**RTO (Recovery Time Objective)**: < 1 hour
- Route 53 failover: < 5 minutes
- Database failover: < 60 seconds
- Application scaling: < 5 minutes

**Recovery Steps**:
1. Detect failure (monitoring alerts)
2. Route 53 fails over to secondary region
3. Promote read replica to primary (if DB failure)
4. Scale up application in secondary region
5. Verify health checks
6. Monitor for stability

### Alternative Architectures

**Serverless Architecture**:
- API Gateway + Lambda + DynamoDB
- Lower cost at low scale
- Automatic scaling
- Less control, cold starts

**Kubernetes on EKS**:
- More control
- Can use Spot instances
- Better for complex workloads
- More operational overhead

**Microservices Architecture**:
- Separate services (user, order, payment)
- Independent scaling
- More complex
- Better for large teams

## Implementation Phases

### Phase 1: MVP (Minimum Viable Product)
- Single region (US)
- Basic infrastructure
- Single database instance
- Basic monitoring

### Phase 2: High Availability
- Multi-AZ deployment
- RDS Multi-AZ
- Auto-scaling
- Enhanced monitoring

### Phase 3: Multi-Region
- Second region (EU)
- Cross-region replication
- Route 53 failover
- Disaster recovery testing

### Phase 4: Optimization
- Cost optimization
- Performance tuning
- Advanced monitoring
- Chaos engineering

## Key Takeaways

1. **Redundancy**: Multiple layers (regions, AZs, instances)
2. **Automation**: Auto-scaling, auto-failover, auto-backups
3. **Monitoring**: Comprehensive observability
4. **Testing**: Regular DR drills, chaos engineering
5. **Documentation**: Runbooks, architecture diagrams
6. **Cost-Benefit**: Balance cost vs availability requirements

## Follow-up Questions

1. How would you handle database migrations in this architecture?
2. How do you ensure zero-downtime deployments?
3. How would you scale this to 10 million users?
4. How do you handle compliance requirements (GDPR, PCI DSS)?
5. What's your strategy for cost optimization?
