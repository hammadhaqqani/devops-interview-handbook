# Scenario: Incident Response

## Problem Statement

Your production application is experiencing a critical outage. Users are reporting 500 errors, the application is slow, and monitoring shows high error rates. You need to respond quickly, diagnose the issue, and restore service.

## Environment

- **Application**: E-commerce platform
- **Infrastructure**: AWS (EC2, RDS, S3, CloudFront)
- **Traffic**: 10,000 requests/minute
- **Time**: Friday evening, peak shopping hours

## Initial Symptoms

1. **User Reports:**
   - "Website is down"
   - "Getting error messages"
   - "Can't complete checkout"

2. **Monitoring Alerts:**
   - Error rate: 45% (normal: <1%)
   - Response time: 15s (normal: <500ms)
   - Database connections: 95% (normal: <50%)
   - CPU utilization: 95% on application servers

3. **Dashboard Shows:**
   - Multiple application servers showing high CPU
   - Database connection pool exhausted
   - Increased 5xx errors
   - Decreased successful requests

## Incident Response Process

### Phase 1: Detection and Initial Response (0-5 minutes)

**Step 1: Acknowledge Incident**

```bash
# Create incident channel/thread
# Notify team
# Set severity level (P1 - Critical)
```

**Step 2: Gather Initial Information**

```bash
# Check monitoring dashboards
# Review recent alerts
# Check deployment history
# Review recent changes
```

**Key Questions:**
- When did it start?
- What changed recently?
- What's the scope? (All users? Specific region?)
- What's the impact? (Revenue loss? User impact?)

**Step 3: Initial Assessment**

```bash
# Check application logs
aws logs tail /aws/ec2/application --follow

# Check error rates
# Check database status
# Check infrastructure health
```

### Phase 2: Containment (5-15 minutes)

**Step 1: Stop the Bleeding**

**Option A: Scale Up (If Resource Constraint)**

```bash
# Increase Auto Scaling Group capacity
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name app-asg \
  --desired-capacity 20 \
  --honor-cooldown

# Or manually add instances
```

**Option B: Enable Maintenance Mode**

```bash
# Show maintenance page to users
# Route traffic away from affected services
# Use feature flags to disable features
```

**Option C: Rollback Recent Deployment**

```bash
# If recent deployment, rollback
kubectl rollout undo deployment/web-app
# or
aws ecs update-service --service web-app --force-new-deployment --task-definition previous-version
```

**Step 2: Isolate Affected Components**

```bash
# Remove unhealthy instances from load balancer
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-1234567890abcdef0

# Or in Kubernetes
kubectl delete pod <unhealthy-pod>
```

### Phase 3: Diagnosis (15-30 minutes)

**Step 1: Check Application Logs**

```bash
# CloudWatch Logs Insights query
fields @timestamp, @message
| filter @message like /ERROR/
| filter @timestamp > date_sub(now(), 15m)
| stats count() by @message

# Or using kubectl
kubectl logs -l app=web-app --tail=1000 | grep ERROR
```

**What to Look For:**
- Error patterns
- Stack traces
- Database connection errors
- Memory errors
- Timeout errors

**Step 2: Check Database**

```bash
# Check database connections
aws rds describe-db-instances --db-instance-identifier prod-db

# Check slow queries
# Connect to database and run:
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

# Check for locks
SELECT * FROM information_schema.innodb_locks;

# Check connection count
SHOW STATUS LIKE 'Threads_connected';
```

**Step 3: Check Infrastructure**

```bash
# Check EC2 instances
aws ec2 describe-instance-status --instance-ids i-1234567890

# Check CPU, memory, disk
# Use CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Average

# Check load balancer
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...
```

**Step 4: Check Recent Changes**

```bash
# Git history
git log --oneline -10

# Deployment history
kubectl rollout history deployment/web-app

# CloudTrail (recent API calls)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --max-results 10
```

### Phase 4: Root Cause Analysis

**Common Root Causes:**

**1. Database Connection Pool Exhausted**

**Symptoms:**
- High database connection count
- "Too many connections" errors
- Slow queries

**Investigation:**
```sql
-- Check active connections
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- Check for long-running queries
SELECT * FROM information_schema.processlist 
WHERE time > 30 
ORDER BY time DESC;

-- Check for locks
SHOW ENGINE INNODB STATUS;
```

**Resolution:**
- Increase connection pool size (temporary)
- Fix connection leaks (permanent)
- Kill long-running queries
- Increase database max_connections

**2. Memory Leak / Out of Memory**

**Symptoms:**
- High memory usage
- OOMKilled containers
- Application crashes

**Investigation:**
```bash
# Check memory usage
free -h
top
docker stats

# Check for memory leaks
# Application profiling
# Heap dumps
```

**Resolution:**
- Restart application (temporary)
- Increase memory limits (temporary)
- Fix memory leak (permanent)

**3. Slow Database Queries**

**Symptoms:**
- High database CPU
- Slow response times
- Timeout errors

**Investigation:**
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;

-- Check slow queries
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;

-- Check indexes
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```

**Resolution:**
- Add indexes
- Optimize queries
- Kill slow queries
- Scale database

**4. Recent Code Deployment**

**Symptoms:**
- Issue started after deployment
- New error patterns
- Performance degradation

**Investigation:**
```bash
# Check deployment time vs incident time
# Review code changes
git diff <previous-commit> <current-commit>

# Check if canary deployment affected
```

**Resolution:**
- Rollback deployment
- Fix code issue
- Redeploy

### Phase 5: Resolution

**Example: Database Connection Pool Issue**

**Immediate Fix:**
```bash
# 1. Increase connection pool (application config)
# Update application configuration
# Restart application

# 2. Kill idle connections
mysql -e "KILL <connection_id>;"

# 3. Increase database max_connections
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --apply-immediately \
  --db-parameter-group-name new-params
```

**Permanent Fix:**
```java
// Fix connection leak in code
// Ensure connections are closed
try (Connection conn = dataSource.getConnection()) {
    // use connection
} // automatically closed

// Or use connection pool properly
// Set appropriate pool size
// Monitor connection usage
```

**Verification:**
```bash
# Monitor error rate
# Should decrease

# Check database connections
SHOW STATUS LIKE 'Threads_connected';

# Check application health
curl https://api.example.com/health

# Monitor metrics
# Error rate should return to normal
```

### Phase 6: Recovery and Monitoring

**Step 1: Verify Resolution**

```bash
# Check error rates (should be <1%)
# Check response times (should be <500ms)
# Check user reports (should decrease)
# Test critical user flows
```

**Step 2: Gradual Traffic Restoration**

```bash
# If traffic was diverted, gradually restore
# Monitor closely
# Keep maintenance mode option ready
```

**Step 3: Enhanced Monitoring**

```bash
# Set up additional alerts
# Monitor key metrics closely
# Watch for recurrence
```

### Phase 7: Post-Incident Review

**Step 1: Document Incident**

**Incident Report Template:**

```markdown
# Incident Report: [Title]

## Summary
- **Date/Time**: [Start] - [End]
- **Duration**: [X hours Y minutes]
- **Severity**: P1
- **Impact**: [Users affected, revenue loss, etc.]

## Timeline
- [Time] - Incident detected
- [Time] - Team notified
- [Time] - Containment started
- [Time] - Root cause identified
- [Time] - Resolution implemented
- [Time] - Service restored

## Root Cause
[Detailed explanation]

## Resolution
[What was done to fix]

## Impact
- Users affected: [X]
- Revenue impact: [$X]
- Downtime: [X minutes]

## Lessons Learned
- What went well
- What could be improved
- Action items

## Action Items
- [ ] Fix connection leak
- [ ] Add monitoring
- [ ] Update runbooks
- [ ] Improve alerting
```

**Step 2: Action Items**

- Fix root cause permanently
- Improve monitoring/alerting
- Update runbooks
- Improve documentation
- Conduct post-mortem meeting

**Step 3: Follow-up**

- Implement permanent fixes
- Review and update processes
- Share learnings with team
- Update incident response playbook

## Prevention Strategies

### 1. Monitoring and Alerting

```yaml
# Set up alerts for:
- Error rate > 1%
- Response time > 1s
- Database connections > 80%
- CPU > 80%
- Memory > 80%
```

### 2. Runbooks

- Document common issues
- Step-by-step resolution procedures
- Escalation paths
- Contact information

### 3. Chaos Engineering

- Test failure scenarios
- Verify resilience
- Practice incident response

### 4. Capacity Planning

- Monitor trends
- Plan for growth
- Set up auto-scaling
- Regular capacity reviews

### 5. Code Quality

- Code reviews
- Testing (unit, integration, e2e)
- Gradual rollouts
- Feature flags

## Key Takeaways

- **Speed Matters**: Quick containment reduces impact
- **Communication**: Keep stakeholders informed
- **Documentation**: Document everything during incident
- **Post-Mortem**: Learn from every incident
- **Prevention**: Focus on preventing recurrence
- **Practice**: Regular incident response drills
- **Monitoring**: Good monitoring enables quick detection
- **Runbooks**: Pre-written procedures save time

## Follow-up Questions

1. How do you prioritize during an incident?
2. How do you communicate with stakeholders?
3. How do you balance speed vs thoroughness?
4. How do you prevent incident fatigue?
5. How do you handle incidents during off-hours?
