# Scenario: Debug Failing Deployment

## Problem Statement

A web application deployment to Kubernetes is failing. Pods are in `CrashLoopBackOff` state, and the application is not accessible. You need to diagnose and fix the issue.

## Environment

- **Platform**: Kubernetes cluster (EKS)
- **Application**: Node.js web application
- **Deployment Method**: Kubernetes Deployment
- **Service Type**: LoadBalancer
- **Namespace**: production

## Symptoms

1. Pods show status: `CrashLoopBackOff`
2. Application endpoint returns 503 Service Unavailable
3. `kubectl get pods` shows:
   ```
   NAME                           READY   STATUS             RESTARTS   AGE
   web-app-7d8f9b4c5-abc12        0/1     CrashLoopBackOff   5          10m
   web-app-7d8f9b4c5-def34        0/1     CrashLoopBackOff   5          10m
   ```

## Step-by-Step Debugging

### Step 1: Check Pod Status and Events

```bash
# Get detailed pod information
kubectl describe pod web-app-7d8f9b4c5-abc12 -n production

# Check events
kubectl get events -n production --sort-by='.lastTimestamp'
```

**What to Look For:**
- Container exit codes
- Error messages in events
- Resource constraints (OOMKilled, CPU throttling)
- Image pull errors

**Common Findings:**
- `ImagePullBackOff`: Cannot pull container image
- `OOMKilled`: Out of memory
- `Error`: Application crash

### Step 2: Check Container Logs

```bash
# Get logs from current container
kubectl logs web-app-7d8f9b4c5-abc12 -n production

# Get logs from previous container (if crashed)
kubectl logs web-app-7d8f9b4c5-abc12 -n production --previous

# Follow logs in real-time
kubectl logs -f web-app-7d8f9b4c5-abc12 -n production
```

**What to Look For:**
- Application errors
- Stack traces
- Configuration errors
- Database connection failures
- Missing environment variables

### Step 3: Check Deployment Configuration

```bash
# Get deployment YAML
kubectl get deployment web-app -n production -o yaml

# Check deployment status
kubectl rollout status deployment/web-app -n production

# Check deployment history
kubectl rollout history deployment/web-app -n production
```

**What to Look For:**
- Incorrect image tag
- Missing environment variables
- Wrong resource limits
- Incorrect command/args

### Step 4: Verify Configuration and Secrets

```bash
# Check ConfigMaps
kubectl get configmap -n production
kubectl describe configmap app-config -n production

# Check Secrets
kubectl get secrets -n production
kubectl describe secret db-credentials -n production

# Verify environment variables in pod
kubectl exec web-app-7d8f9b4c5-abc12 -n production -- env
```

**What to Look For:**
- Missing ConfigMap/Secret references
- Incorrect environment variable names
- Base64 encoding issues in secrets

### Step 5: Check Resource Constraints

```bash
# Check resource usage
kubectl top pod web-app-7d8f9b4c5-abc12 -n production

# Check node resources
kubectl top nodes

# Check resource quotas
kubectl describe quota -n production
```

**What to Look For:**
- Memory limits too low (OOMKilled)
- CPU limits causing throttling
- Resource quota exceeded

### Step 6: Test Container Locally (If Possible)

```bash
# Run container locally with same configuration
docker run -e DATABASE_URL=$DATABASE_URL \
  -e PORT=3000 \
  myregistry/web-app:latest

# Check if application starts
curl http://localhost:3000/health
```

## Root Cause Analysis

### Common Root Causes:

**1. Missing Environment Variables**
- Application requires `DATABASE_URL` but not set
- Solution: Add to deployment or ConfigMap

**2. Database Connection Failure**
- Database not accessible from cluster
- Wrong connection string
- Solution: Verify database endpoint, network policies, credentials

**3. Application Code Error**
- Recent code change introduced bug
- Solution: Review recent commits, rollback if needed

**4. Resource Limits Too Low**
- Memory limit causes OOMKilled
- Solution: Increase memory limits

**5. Wrong Image Tag**
- Deployment uses wrong/old image
- Solution: Update image tag

**6. Health Check Failures**
- Readiness probe failing
- Liveness probe killing healthy pods
- Solution: Adjust probe configuration

## Resolution Steps

### Example Fix: Missing Environment Variable

**Problem Identified:**
Logs show: `Error: DATABASE_URL is not defined`

**Solution:**

1. **Create or Update ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_URL: "postgresql://db:5432/mydb"
  NODE_ENV: "production"
  PORT: "3000"
```

2. **Update Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:v1.2.3
        envFrom:
        - configMapRef:
            name: app-config
```

3. **Apply Changes:**
```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml

# Wait for rollout
kubectl rollout status deployment/web-app -n production
```

4. **Verify Fix:**
```bash
# Check pod status
kubectl get pods -n production

# Check logs
kubectl logs -l app=web-app -n production --tail=50

# Test endpoint
curl https://web-app.example.com/health
```

## Prevention Strategies

### 1. Pre-Deployment Validation

```bash
# Validate YAML before applying
kubectl apply --dry-run=client -f deployment.yaml

# Use kubeval or similar tools
kubeval deployment.yaml
```

### 2. Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. Resource Monitoring

- Set up alerts for pod failures
- Monitor resource usage
- Use Prometheus/Grafana for observability

### 4. Staged Rollouts

- Deploy to staging first
- Use canary deployments
- Gradual rollout with monitoring

### 5. Configuration Management

- Use ConfigMaps/Secrets properly
- Validate configuration before deployment
- Version control all configurations

### 6. Testing

- Test containers locally before deployment
- Integration tests in CI/CD
- Load testing to verify resource limits

## Follow-up Questions

1. How would you debug if logs show no errors but pods still crash?
2. What if the issue only occurs in production, not staging?
3. How do you prevent this issue in future deployments?
4. What monitoring would you set up to detect this earlier?

## Key Takeaways

- Always check logs first (`kubectl logs`)
- Use `kubectl describe` for detailed information
- Verify configuration and environment variables
- Check resource constraints
- Test locally when possible
- Implement proper health checks
- Set up monitoring and alerts
