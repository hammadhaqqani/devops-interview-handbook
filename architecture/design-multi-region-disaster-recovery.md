# Architecture Exercise: Design Multi-Region Disaster Recovery

## Problem Statement

Design a disaster recovery solution for a critical financial application that processes $1 billion in transactions annually. The system must survive regional disasters, maintain data consistency, and meet strict RTO/RPO requirements.

## Requirements

### Functional Requirements

1. **Application Availability**
   - Web application (customer-facing)
   - API services
   - Background job processing
   - Real-time transaction processing

2. **Data Requirements**
   - Customer accounts
   - Transaction history
   - Financial records
   - Audit logs

3. **Compliance**
   - Financial regulations
   - Data retention (7 years)
   - Audit trails
   - Data sovereignty

### Non-Functional Requirements

1. **RPO (Recovery Point Objective)**: < 1 minute (near-zero data loss)
2. **RTO (Recovery Time Objective)**: < 15 minutes
3. **Availability**: 99.99% (52.56 minutes downtime/year)
4. **Data Consistency**: Strong consistency required
5. **Geographic Distribution**: Primary US, Secondary EU
6. **Compliance**: PCI DSS, SOX, GDPR

## Constraints and Assumptions

### Constraints

- **Budget**: $200,000/month for DR infrastructure
- **Regulations**: Must comply with financial regulations
- **Data Residency**: EU data must stay in EU
- **Latency**: < 100ms for transaction processing

### Assumptions

- Transaction volume: 10,000 transactions/minute peak
- Average transaction size: 5KB
- Database size: 2TB, growing 50GB/month
- Read:Write ratio: 5:1
- Network latency between regions: 80ms

## Reference Architecture

### Multi-Region Active-Active Architecture

```
                    Global Users
                         |
                  [Route 53 DNS]
              (Latency-Based Routing)
                         |
        ┌─────────────────┴─────────────────┐
        |                                   |
   [US Region]                        [EU Region]
   (Primary)                          (Secondary)
        |                                   |
   ┌────┴────┐                        ┌────┴────┐
   |         |                        |         |
[ALB]    [ALB]                    [ALB]    [ALB]
   |         |                        |         |
   └────┬────┘                        └────┬────┘
        |                                   |
   ┌────┴────┐                        ┌────┴────┐
   |         |                        |         |
[App]    [App]                    [App]    [App]
(AZ-1)  (AZ-2)                  (AZ-1)  (AZ-2)
   |         |                        |         |
   └────┬────┘                        └────┬────┘
        |                                   |
   ┌────┴────┐                        ┌────┴────┐
   |         |                        |         |
[RDS]    [RDS]                    [RDS]    [RDS]
Primary  Replica                Replica  Replica
        |                                   |
        └───────────┬───────────────────────┘
                    |
            [Global Database]
        (Cross-Region Replication)
                    |
        ┌────────────┴────────────┐
        |                         |
   [S3 US]                   [S3 EU]
   (Backup)                 (Backup)
        |                         |
        └────────────┬────────────┘
                     |
            [S3 Cross-Region Replication]
```

### Component Breakdown

#### 1. DNS and Traffic Routing

**Component**: Route 53 (AWS)

**Configuration**:
- **Latency-Based Routing**: Route to nearest region
- **Health Checks**: 
  - Check every 30 seconds
  - Failover if health check fails
  - Monitor application endpoints
- **Failover Routing**: 
  - Primary: US region
  - Secondary: EU region
  - Automatic failover on failure

**High Availability**:
- Multiple health check endpoints per region
- Failover in < 1 minute
- DNS TTL: 60 seconds (fast failover)

#### 2. Application Layer

**Component**: ECS/EKS with Auto Scaling

**US Region (Primary)**:
- **Capacity**: 
  - Normal: 20 tasks
  - Peak: 50 tasks
  - DR: Can scale to 100 tasks
- **Deployment**: 3 AZs
- **Load Balancing**: ALB with cross-zone load balancing

**EU Region (Secondary)**:
- **Capacity**:
  - Normal: 10 tasks (warm standby)
  - Can scale to 50 tasks on failover
- **Deployment**: 3 AZs
- **Traffic**: Can handle full load if needed

**Configuration**:
- Same application code in both regions
- Environment-specific configuration
- Feature flags for gradual rollout

#### 3. Database Layer

**Component**: RDS PostgreSQL with Cross-Region Replication

**US Region (Primary)**:
- **Instance**: db.r5.4xlarge (16 vCPU, 128GB RAM)
- **Storage**: 2TB GP3, auto-scaling
- **Multi-AZ**: Enabled (synchronous replication within region)
- **Backups**: 
  - Automated daily backups
  - Continuous backup (point-in-time recovery)
  - 35-day retention

**EU Region (Replica)**:
- **Instance**: db.r5.4xlarge (same specs)
- **Replication**: 
  - Cross-region read replica (asynchronous)
  - Can be promoted to primary
  - Lag: < 1 second typically

**Global Database (Aurora Global Database)**:
- **Alternative**: Use Aurora Global Database
- **Benefits**:
  - < 1 second replication lag
  - Automatic failover
  - Read replicas in multiple regions
  - Point-in-time recovery

**Configuration**:
```sql
-- US Primary
CREATE PUBLICATION global_publication FOR ALL TABLES;

-- EU Replica
CREATE SUBSCRIPTION global_subscription
  CONNECTION 'host=us-db.example.com dbname=mydb'
  PUBLICATION global_publication;
```

**RPO Achievement**:
- Synchronous replication within region (US)
- Asynchronous cross-region (< 1s lag)
- Continuous backups
- RPO: < 1 minute (near-zero data loss)

#### 4. Caching Layer

**Component**: ElastiCache Redis with Global Datastore

**Configuration**:
- **US Region**: 
  - Primary cluster (3 shards, 2 replicas)
  - Handles all writes
- **EU Region**: 
  - Secondary cluster (read replica)
  - Handles reads
  - Can be promoted to primary

**Alternative**: Redis with Replication
- Use Redis replication between regions
- Write to primary, read from replica
- Promote replica on failover

#### 5. Message Queue

**Component**: Amazon SQS / RabbitMQ with Federation

**Configuration**:
- **US Region**: Primary queue
- **EU Region**: Replica queue
- **Replication**: Queue federation or SQS cross-region
- **Failover**: Switch to EU queue on failure

**Alternative**: Kafka with MirrorMaker
- Kafka cluster in each region
- MirrorMaker replicates topics
- Consumers can switch regions

#### 6. Object Storage

**Component**: S3 with Cross-Region Replication

**Configuration**:
- **US Region**: 
  - Primary bucket
  - Versioning enabled
  - Lifecycle policies
- **EU Region**: 
  - Replica bucket
  - Cross-region replication
  - Same versioning and lifecycle

**Backup Strategy**:
- Daily backups to S3
- Cross-region replication
- Glacier for long-term retention (7 years)
- Encrypted at rest

#### 7. Monitoring and Alerting

**Component**: CloudWatch, X-Ray, Custom Dashboards

**Monitoring**:
- Application health (error rate, latency)
- Database replication lag
- Regional health
- Network connectivity
- Failover readiness

**Alerts**:
- Replication lag > 5 seconds
- Regional health check failures
- Database connection issues
- High error rates

#### 8. Disaster Recovery Automation

**Component**: Lambda Functions, EventBridge

**Failover Automation**:
```python
def handle_regional_failure(event):
    # 1. Detect failure (health checks)
    if us_region_health_check_failed():
        # 2. Promote EU replica to primary
        promote_eu_replica_to_primary()
        
        # 3. Update Route 53 to failover
        route53_failover_to_eu()
        
        # 4. Scale up EU region
        scale_up_eu_region()
        
        # 5. Notify team
        send_alert("Failover to EU region")
        
        # 6. Verify health
        verify_eu_region_health()
```

## Disaster Recovery Scenarios

### Scenario 1: Regional Outage (US Region)

**Detection**:
- Health checks fail
- Monitoring alerts triggered
- Manual or automatic failover

**Recovery Steps**:
1. **Detect Failure** (< 1 minute)
   - Health checks fail
   - Alert triggered

2. **Promote EU Database** (< 2 minutes)
   - Stop replication from US
   - Promote EU replica to primary
   - Verify data consistency

3. **Update DNS** (< 1 minute)
   - Route 53 failover to EU
   - DNS propagation

4. **Scale EU Region** (< 5 minutes)
   - Scale application to full capacity
   - Verify health

5. **Verify Service** (< 2 minutes)
   - Smoke tests
   - Monitor metrics

**Total RTO**: < 15 minutes

### Scenario 2: Database Failure (US Primary)

**Detection**:
- Database health check fails
- Application errors increase

**Recovery Steps**:
1. **Failover to US Replica** (< 1 minute)
   - RDS Multi-AZ automatic failover
   - Application reconnects automatically

2. **If US Replica Fails** (< 5 minutes)
   - Promote EU replica to primary
   - Update application connection strings
   - Update DNS if needed

**Total RTO**: < 5 minutes (US replica) or < 15 minutes (EU)

### Scenario 3: Network Partition

**Detection**:
- High latency between regions
- Replication lag increases

**Recovery Steps**:
1. **Continue Operating** (both regions)
   - US handles US traffic
   - EU handles EU traffic
   - Resolve conflicts when network restored

2. **Conflict Resolution**:
   - Use timestamps
   - Manual review if needed
   - Merge strategies

## Discussion Points

### Data Consistency Trade-offs

**Strong Consistency (Chosen)**:
- Synchronous replication within region
- Asynchronous cross-region (< 1s lag)
- **Pros**: Data consistency, no conflicts
- **Cons**: Higher latency, more complex

**Eventual Consistency**:
- Asynchronous replication everywhere
- **Pros**: Lower latency, simpler
- **Cons**: Potential conflicts, data inconsistency

**Decision**: Strong consistency for financial data (regulatory requirement)

### Active-Active vs Active-Passive

**Active-Active (Chosen)**:
- Both regions handle traffic
- **Pros**: Better resource utilization, lower latency
- **Cons**: More complex, potential conflicts

**Active-Passive**:
- Only primary handles traffic
- **Pros**: Simpler, no conflicts
- **Cons**: Wasted resources, higher cost

**Decision**: Active-Active for better performance and cost

### Replication Strategies

**Synchronous Replication**:
- Within region (US Multi-AZ)
- **Pros**: Zero data loss, strong consistency
- **Cons**: Higher latency

**Asynchronous Replication**:
- Cross-region (US → EU)
- **Pros**: Lower latency, better performance
- **Cons**: Potential data loss (< 1s)

**Decision**: Hybrid approach (sync within region, async cross-region)

### Cost Considerations

**Estimated Monthly Cost**:

- **US Region**: $50,000
  - Compute: $20,000
  - Database: $15,000
  - Storage: $5,000
  - Network: $10,000

- **EU Region**: $30,000
  - Compute (warm): $10,000
  - Database (replica): $10,000
  - Storage: $5,000
  - Network: $5,000

- **DR Infrastructure**: $20,000
  - Monitoring: $5,000
  - Backup storage: $10,000
  - Testing: $5,000

- **Total**: ~$100,000/month

**Optimization**:
- Use reserved instances (40% savings)
- Right-size instances
- Optimize data transfer
- Use S3 lifecycle policies

### Testing Strategy

**Regular DR Drills**:
- Monthly: Test failover procedures
- Quarterly: Full DR test
- Annually: Simulate complete regional failure

**Test Scenarios**:
1. Database failover
2. Regional failover
3. Network partition
4. Data corruption recovery

**Success Criteria**:
- RTO < 15 minutes
- RPO < 1 minute
- No data loss
- Application functional

## Implementation Phases

### Phase 1: Single Region Multi-AZ
- Multi-AZ deployment
- Automated backups
- Basic monitoring

### Phase 2: Cross-Region Replication
- Set up EU region
- Database replication
- S3 replication

### Phase 3: Active-Active
- Route traffic to both regions
- Load balancing
- Conflict resolution

### Phase 4: Automation
- Automated failover
- Self-healing
- Advanced monitoring

## Key Takeaways

1. **RPO/RTO**: Define clear objectives
2. **Replication**: Choose appropriate strategy
3. **Automation**: Automate failover when possible
4. **Testing**: Regular DR drills
5. **Monitoring**: Comprehensive observability
6. **Documentation**: Detailed runbooks
7. **Cost**: Balance cost vs requirements

## Follow-up Questions

1. How do you handle data conflicts in active-active?
2. How do you test DR without impacting production?
3. How do you ensure compliance during failover?
4. How do you handle partial failures?
5. How do you optimize costs while maintaining DR?
