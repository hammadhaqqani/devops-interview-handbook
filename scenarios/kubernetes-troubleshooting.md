# Scenario: Kubernetes Troubleshooting

## Problem Statement

A production Kubernetes cluster is experiencing issues: pods are not starting, services are unreachable, and nodes are reporting as NotReady. You need to systematically troubleshoot and resolve these issues.

## Environment

- **Platform**: Kubernetes 1.28 (EKS)
- **Cluster Size**: 10 worker nodes across 3 availability zones
- **Applications**: Multiple microservices deployed
- **Issue Started**: 2 hours ago after a cluster update

## Symptoms

1. **Node Issues:**
   - 3 nodes showing `NotReady` status
   - `kubectl get nodes` shows:
     ```
     NAME                          STATUS     ROLES    AGE   VERSION
     ip-10-0-1-10.ec2.internal     Ready      <none>   30d   v1.28.0
     ip-10-0-2-20.ec2.internal     NotReady   <none>   30d   v1.28.0
     ip-10-0-3-30.ec2.internal     NotReady   <none>   30d   v1.28.0
     ```

2. **Pod Issues:**
   - Multiple pods in `Pending` state
   - Some pods in `CrashLoopBackOff`
   - Pods cannot be scheduled

3. **Service Issues:**
   - Services returning 503 errors
   - Endpoints not found
   - DNS resolution failing

## Step-by-Step Troubleshooting

### Step 1: Check Node Status

```bash
# Get node status
kubectl get nodes

# Describe problematic nodes
kubectl describe node ip-10-0-2-20.ec2.internal

# Check node conditions
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, conditions: .status.conditions}'
```

**What to Look For:**
- **Ready condition**: Should be True
- **MemoryPressure**: Check if node out of memory
- **DiskPressure**: Check disk space
- **PIDPressure**: Check process limits
- **NetworkUnavailable**: Network issues
- **KubeletNotReady**: kubelet problems

**Common Issues:**
- Out of disk space
- Out of memory
- kubelet not running
- Network connectivity issues
- Container runtime issues

### Step 2: Check Node Resources

```bash
# Check node resource usage
kubectl top nodes

# Check node capacity and allocatable
kubectl describe node ip-10-0-2-20.ec2.internal | grep -A 5 "Allocated resources"

# Check for resource pressure
kubectl get nodes -o custom-columns=NAME:.metadata.name,\
  MEMORY-PRESSURE:.status.conditions[?(@.type=='MemoryPressure')].status,\
  DISK-PRESSURE:.status.conditions[?(@.type=='DiskPressure')].status
```

**Actions:**
- If disk full: Clean up unused images, increase disk size
- If memory full: Evict pods, add nodes, increase node size
- If CPU throttling: Check resource limits

### Step 3: Check kubelet Status

```bash
# SSH to node (if possible)
ssh ec2-user@ip-10-0-2-20.ec2.internal

# Check kubelet status
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet -n 100 --no-pager

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml
```

**Common kubelet Issues:**
- kubelet not running: `systemctl start kubelet`
- Certificate issues: Check kubelet certificates
- API server connectivity: Check network/firewall
- Container runtime issues: Check containerd/docker

### Step 4: Check Container Runtime

```bash
# On node, check container runtime
sudo systemctl status containerd
# or
sudo systemctl status docker

# Check runtime logs
sudo journalctl -u containerd -n 100

# Test container runtime
sudo crictl images
sudo crictl ps -a
```

**Common Issues:**
- Container runtime not running
- Image pull failures
- Runtime version mismatch

### Step 5: Check Pod Scheduling

```bash
# Check pending pods
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Describe pending pod
kubectl describe pod <pod-name> -n <namespace>

# Check events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20
```

**Common Scheduling Issues:**
- No available nodes (all nodes NotReady or resource constraints)
- Node selectors/affinity not matching
- Taints preventing scheduling
- Resource requests exceed available resources
- PVC cannot be bound

### Step 6: Check Network Issues

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS from pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check service endpoints
kubectl get endpoints -A

# Check if endpoints are empty
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods -l <selector> --show-labels
```

**Common Network Issues:**
- CoreDNS not running
- Service selectors don't match pods
- Network policies blocking traffic
- CNI plugin issues

### Step 7: Check API Server and Control Plane

```bash
# Check API server status (if you have access)
kubectl get componentstatuses
# or
kubectl get cs

# Check API server logs (if on master)
kubectl logs -n kube-system kube-apiserver-<node-name>

# Test API server connectivity
kubectl cluster-info
```

**Common Control Plane Issues:**
- API server down
- etcd issues
- Scheduler issues
- Controller manager issues

### Step 8: Check Storage Issues

```bash
# Check PVCs
kubectl get pvc -A

# Check PVs
kubectl get pv

# Check storage classes
kubectl get storageclass

# Describe pending PVC
kubectl describe pvc <pvc-name> -n <namespace>
```

**Common Storage Issues:**
- Storage class not found
- No available PVs
- Storage provisioner issues
- EBS volume attachment issues (AWS)

## Resolution Examples

### Example 1: Node NotReady - Disk Full

**Symptoms:**
- Node shows `NotReady`
- `kubectl describe node` shows `DiskPressure: True`

**Resolution:**

```bash
# SSH to node
ssh ec2-user@ip-10-0-2-20.ec2.internal

# Check disk usage
df -h

# Clean up Docker/containerd images
sudo docker system prune -a --volumes
# or for containerd
sudo crictl rmi --prune

# Clean up old logs
sudo journalctl --vacuum-time=7d

# If still full, increase disk size (AWS)
# 1. Create snapshot
# 2. Resize EBS volume
# 3. Extend filesystem
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1

# Restart kubelet
sudo systemctl restart kubelet

# Verify node status
kubectl get nodes
```

### Example 2: Pods Pending - No Available Nodes

**Symptoms:**
- Pods stuck in `Pending`
- Events show: `0/3 nodes are available`

**Resolution:**

```bash
# Check why pods can't be scheduled
kubectl describe pod <pod-name> | grep -A 10 "Events:"

# Common reasons:
# 1. Resource constraints
kubectl describe node | grep -A 5 "Allocated resources"

# 2. Node selectors
kubectl get pods -o jsonpath='{.spec.nodeSelector}'

# 3. Taints
kubectl describe node | grep Taint

# Solutions:
# Option 1: Add more nodes
# Option 2: Remove node selector or add label
kubectl label node <node-name> <key>=<value>

# Option 3: Remove taint (if appropriate)
kubectl taint nodes <node-name> <key>-<effect>-

# Option 4: Add toleration to pod
```

### Example 3: Service Unreachable - No Endpoints

**Symptoms:**
- Service exists but returns 503
- `kubectl get endpoints` shows no endpoints

**Resolution:**

```bash
# Check service selector
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'

# Check if pods match selector
kubectl get pods -l app=web --show-labels

# Common issue: Selector doesn't match pod labels
# Fix: Update service or pod labels

# Update service selector
kubectl patch svc <service-name> -p '{"spec":{"selector":{"app":"web"}}}'

# Or update pod labels
kubectl label pods <pod-name> app=web

# Verify endpoints
kubectl get endpoints <service-name>
```

### Example 4: DNS Not Working

**Symptoms:**
- Pods can't resolve service names
- `nslookup` fails

**Resolution:**

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# If not running, check why
kubectl describe pod -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Restart CoreDNS if needed
kubectl delete pod -n kube-system -l k8s-app=kube-dns

# Verify DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
```

## Prevention Strategies

### 1. Resource Management

```yaml
# Set resource requests and limits
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### 2. Node Health Monitoring

- Set up node monitoring (Prometheus)
- Alert on node NotReady
- Alert on resource pressure
- Regular node health checks

### 3. Cluster Autoscaling

```yaml
# Enable cluster autoscaler
# Automatically adds nodes when needed
```

### 4. Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web
```

### 5. Regular Maintenance

- Regular cluster updates
- Node rotation
- Image cleanup
- Log rotation

## Troubleshooting Checklist

- [ ] Check node status (`kubectl get nodes`)
- [ ] Check node resources (`kubectl top nodes`)
- [ ] Check kubelet status (on node)
- [ ] Check container runtime (on node)
- [ ] Check pod status (`kubectl get pods -A`)
- [ ] Check pod events (`kubectl describe pod`)
- [ ] Check service endpoints (`kubectl get endpoints`)
- [ ] Check DNS (`kubectl run debug -- nslookup`)
- [ ] Check network policies
- [ ] Check storage (PVCs, PVs)
- [ ] Check control plane components
- [ ] Review recent changes (deployments, updates)

## Key Takeaways

- Start with node status, then pods, then services
- Check resources (CPU, memory, disk) first
- Use `kubectl describe` for detailed information
- Check logs at multiple levels (kubelet, container, application)
- Verify network connectivity and DNS
- Document findings and resolutions
- Implement monitoring to detect issues early
