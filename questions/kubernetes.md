# Kubernetes Interview Questions

## Table of Contents
- [Core Concepts](#core-concepts)
- [Pods & Containers](#pods--containers)
- [Services & Networking](#services--networking)
- [Deployments & ReplicaSets](#deployments--replicasets)
- [ConfigMaps & Secrets](#configmaps--secrets)
- [Storage & Volumes](#storage--volumes)
- [Troubleshooting](#troubleshooting)
- [Advanced Topics](#advanced-topics)

---

## Core Concepts

### Q1: What is Kubernetes and what problems does it solve?

**Difficulty:** Junior

**Answer:**

Kubernetes (K8s) is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

**Problems it Solves:**
- **Container Orchestration**: Manages many containers across many hosts
- **Service Discovery**: Containers find each other automatically
- **Load Balancing**: Distributes traffic across containers
- **Self-Healing**: Restarts failed containers, replaces unhealthy pods
- **Scaling**: Scale up/down based on demand
- **Rolling Updates**: Update applications with zero downtime
- **Resource Management**: Allocates CPU/memory efficiently
- **Storage Orchestration**: Mounts storage systems

**Key Concepts:**
- **Cluster**: Set of nodes (masters and workers)
- **Node**: Machine running containers
- **Pod**: Smallest deployable unit (one or more containers)
- **Service**: Stable network endpoint for pods

**Real-world Context:** Instead of manually managing 100 containers across 10 servers, Kubernetes automates deployment, scaling, and health checks.

**Follow-up:** What's the difference between Kubernetes and Docker Swarm? (K8s: more features, complex, industry standard. Swarm: simpler, Docker-native)

---

### Q2: Explain the Kubernetes architecture: Master and Worker nodes.

**Difficulty:** Mid

**Answer:**

**Master Node (Control Plane):**
- **API Server**: Entry point, validates requests, updates etcd
- **etcd**: Distributed key-value store (cluster state)
- **Scheduler**: Assigns pods to nodes based on resources/constraints
- **Controller Manager**: Runs controllers (replication, endpoints, etc.)
- **Cloud Controller Manager**: Cloud-specific controllers (optional)

**Worker Node:**
- **kubelet**: Agent that communicates with API server, manages pods
- **kube-proxy**: Network proxy, maintains network rules
- **Container Runtime**: Docker, containerd, CRI-O (runs containers)
- **Pods**: Running containers

**Communication Flow:**
1. User → kubectl → API Server
2. API Server → etcd (store state)
3. Scheduler → API Server (assign pod to node)
4. kubelet → API Server (get pod specs)
5. kubelet → Container Runtime (create containers)

**Real-world Context:** Master node manages cluster state. Worker nodes run your applications. If master fails, cluster management stops (but apps keep running).

**Follow-up:** What happens if the master node fails? (Cluster management stops, but worker nodes continue running existing pods. Need HA setup with multiple masters)

---

### Q3: What is a Pod and why is it the smallest deployable unit?

**Difficulty:** Mid

**Answer:**

A Pod is the smallest deployable unit in Kubernetes - a group of one or more containers that share storage and network.

**Pod Characteristics:**
- Containers in pod share:
  - Network namespace (same IP, can communicate via localhost)
  - Storage volumes
  - IPC namespace
- Containers are always co-located and co-scheduled
- Pods are ephemeral (created, destroyed, recreated)

**Why Pods, Not Containers?**
- Some applications need multiple containers working together
- Sidecar pattern: main container + helper (logging, proxy)
- Init containers: run before main containers

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: web
    image: nginx
  - name: log-collector
    image: fluentd
```

**Real-world Context:** Web server pod with nginx container and fluentd sidecar for log collection. They share network and can communicate via localhost.

**Follow-up:** Can you run multiple containers in a pod? (Yes, but usually one main container + sidecars. Don't put multiple apps in one pod)

---

### Q4: Explain Kubernetes namespaces and their use cases.

**Difficulty:** Mid

**Answer:**

Namespaces provide logical separation and resource isolation within a cluster.

**Default Namespaces:**
- `default`: User resources (if not specified)
- `kube-system`: System components
- `kube-public`: Publicly accessible resources
- `kube-node-lease`: Node heartbeat

**Use Cases:**
- **Environment Separation**: dev, staging, prod
- **Team Separation**: Different teams use different namespaces
- **Resource Quotas**: Limit resources per namespace
- **Access Control**: RBAC per namespace
- **Organization**: Group related resources

**Creating Namespace:**
```bash
kubectl create namespace production
kubectl apply -f app.yaml -n production
```

**Resource Quotas:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

**Real-world Context:** Separate dev and prod in same cluster. Dev team can't accidentally affect prod resources.

**Follow-up:** Can pods in different namespaces communicate? (Yes, using service DNS: `service-name.namespace.svc.cluster.local`)

---

## Pods & Containers

### Q5: What are the different pod lifecycle phases?

**Difficulty:** Mid

**Answer:**

Pod phases represent where a pod is in its lifecycle:

**Phases:**
- **Pending**: Pod accepted, but containers not created (scheduling, pulling images)
- **Running**: Pod bound to node, all containers created, at least one running
- **Succeeded**: All containers terminated successfully (Job pods)
- **Failed**: At least one container terminated with failure
- **Unknown**: Pod state cannot be determined (node communication issues)

**Checking Phase:**
```bash
kubectl get pods
kubectl describe pod <pod-name>
```

**Container States (within Pod):**
- **Waiting**: Container is starting (image pull, etc.)
- **Running**: Container is running
- **Terminated**: Container exited

**Real-world Context:** Pod stuck in Pending → check node resources, image pull issues, node selectors. Pod in Failed → check container logs.

**Follow-up:** What causes a pod to be in Pending state? (No available nodes, image pull errors, resource constraints, node selectors/affinity)

---

### Q6: Explain resource requests and limits in Kubernetes.

**Difficulty:** Mid

**Answer:**

**Requests**: Guaranteed resources (scheduler uses for placement)
**Limits**: Maximum resources (container cannot exceed)

**Example:**
```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

**CPU Units:**
- `1000m` = 1 CPU core
- `500m` = 0.5 cores
- `0.5` = 0.5 cores

**Memory Units:**
- `64Mi` = 64 mebibytes
- `1Gi` = 1 gibibyte

**QoS Classes:**
- **Guaranteed**: Requests = Limits (all containers)
- **Burstable**: Requests < Limits (at least one container)
- **BestEffort**: No requests/limits

**Scheduling:**
- Scheduler uses requests to find nodes with available resources
- Limits enforced by kubelet (cgroups)

**Real-world Context:** Web app requests 256Mi memory, limit 512Mi. Scheduler places on node with 256Mi free. If app uses 600Mi, it's killed (OOMKilled).

**Follow-up:** What happens if a container exceeds its memory limit? (Container is killed with OOMKilled status, pod may be restarted)

---

### Q7: What are Init Containers and when would you use them?

**Difficulty:** Mid

**Answer:**

Init containers run before main containers in a pod, must complete successfully before main containers start.

**Characteristics:**
- Run sequentially (one after another)
- Must complete successfully (exit code 0)
- If init container fails, pod restarts
- Can have different images than main containers

**Use Cases:**
- **Wait for Dependencies**: Wait for database to be ready
- **Setup/Configuration**: Download configs, setup directories
- **Security**: Run as different user, perform security checks
- **Data Migration**: Run migrations before app starts

**Example:**
```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
  - name: app
    image: myapp
```

**Real-world Context:** App depends on database. Init container waits for DB to be ready, then main app container starts.

**Follow-up:** What's the difference between init containers and sidecars? (Init: run before main, sequential. Sidecar: run alongside main, parallel)

---

### Q8: Explain liveness and readiness probes.

**Difficulty:** Mid

**Answer:**

**Liveness Probe:**
- Determines if container is alive
- If fails, container is restarted
- Use when: app can deadlock, needs restart

**Readiness Probe:**
- Determines if container is ready to serve traffic
- If fails, pod removed from Service endpoints
- Use when: app needs warm-up time, temporary unavailability

**Probe Types:**
- **HTTP**: HTTP GET request (most common)
- **TCP**: TCP connection check
- **Exec**: Execute command, check exit code

**Example:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**Real-world Context:** App takes 20s to start. Readiness probe waits 20s before adding to load balancer. Liveness probe restarts if app hangs.

**Follow-up:** What happens if liveness probe fails? (Container is killed and restarted. Pod phase may show CrashLoopBackOff)

---

## Services & Networking

### Q9: What is a Kubernetes Service and what are the different types?

**Difficulty:** Mid

**Answer:**

A Service provides stable network access to a set of pods, abstracting pod IPs which change.

**Service Types:**

**ClusterIP (default):**
- Internal IP, accessible within cluster
- Use for: Internal services, microservices communication

**NodePort:**
- Exposes service on each node's IP at static port (30000-32767)
- Use for: Development, testing, legacy apps

**LoadBalancer:**
- Cloud provider load balancer (AWS ELB, GCP LB)
- Use for: Production external access

**ExternalName:**
- Maps service to external DNS name
- Use for: External services, databases

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Real-world Context:** Frontend pods change IPs. Service provides stable endpoint. Frontend Service (ClusterIP) → Backend Service (ClusterIP) → Database (ExternalName).

**Follow-up:** How does Service select pods? (Using label selectors: `selector: { app: web }`)

---

### Q10: Explain how Kubernetes DNS works.

**Difficulty:** Mid

**Answer:**

Kubernetes has built-in DNS (CoreDNS) that provides service discovery.

**DNS Names:**
- **Service**: `service-name.namespace.svc.cluster.local`
- **Pod**: `pod-ip.namespace.pod.cluster.local`
- **Short names**: `service-name` (same namespace), `service-name.namespace` (different namespace)

**How it Works:**
- CoreDNS watches API server for Services/Endpoints
- Creates DNS records automatically
- Pods use these DNS names to find services

**Example:**
```bash
# From pod, access service:
curl http://web-service.default.svc.cluster.local
# Or short form (same namespace):
curl http://web-service
```

**Real-world Context:** Frontend pod needs to call backend API. Use DNS name `backend-service` instead of hardcoding IPs.

**Follow-up:** What's the difference between Service DNS and Pod DNS? (Service: stable, multiple pods. Pod: specific pod IP, changes)

---

### Q11: What is an Ingress and how does it differ from a Service?

**Difficulty:** Mid

**Answer:**

Ingress provides HTTP/HTTPS routing to services based on hostname/path, acting as a reverse proxy.

**Ingress vs Service:**
- **Service**: L4 (TCP/UDP) load balancing, single IP
- **Ingress**: L7 (HTTP/HTTPS) routing, multiple hosts/paths, SSL termination

**Ingress Controller:**
- Required to implement Ingress (nginx, traefik, AWS ALB)
- Runs as pods in cluster
- Watches Ingress resources, configures load balancer

**Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Real-world Context:** Multiple apps on same domain: `/app` → frontend service, `/api` → backend service. Ingress routes based on path.

**Follow-up:** Do you need an Ingress Controller? (Yes, Ingress is just a spec. Need controller like nginx-ingress or AWS ALB Ingress Controller)

---

## Deployments & ReplicaSets

### Q12: What is a Deployment and how does it manage ReplicaSets?

**Difficulty:** Mid

**Answer:**

Deployment manages ReplicaSets, which manage Pods. Provides declarative updates and rollback.

**Hierarchy:**
- Deployment → ReplicaSet → Pods
- Deployment creates ReplicaSets for each revision
- ReplicaSet ensures desired number of pods

**Deployment Features:**
- **Rolling Updates**: Update pods gradually (zero downtime)
- **Rollback**: Revert to previous version
- **Scaling**: Scale number of replicas
- **Pause/Resume**: Pause rollout for inspection

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.20
```

**Rolling Update:**
- Creates new ReplicaSet with new image
- Gradually scales up new, scales down old
- `maxSurge`: Max pods over desired (default 25%)
- `maxUnavailable`: Max pods unavailable (default 25%)

**Real-world Context:** Update app from v1 to v2. Deployment creates new ReplicaSet, gradually replaces pods. If issues, rollback to v1.

**Follow-up:** How do you rollback a deployment? (`kubectl rollout undo deployment/web-deployment`)

---

### Q13: Explain Deployment update strategies: RollingUpdate vs Recreate.

**Difficulty:** Mid

**Answer:**

**RollingUpdate (default):**
- Gradually replaces old pods with new
- Zero downtime
- Both versions run simultaneously
- Use for: Production, stateless apps

**Recreate:**
- Terminates all old pods, then creates new
- Brief downtime
- Only one version runs at a time
- Use for: Apps that can't run multiple versions, stateful apps

**Configuration:**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

**RollingUpdate Parameters:**
- `maxSurge`: Can exceed desired replicas during update
- `maxUnavailable`: Can be below desired replicas

**Real-world Context:** Stateless web app → RollingUpdate (zero downtime). Database migration → Recreate (can't run two versions).

**Follow-up:** What's the difference between maxSurge and maxUnavailable? (maxSurge: extra pods allowed, maxUnavailable: pods that can be down)

---

### Q14: What is a ReplicaSet and how does it differ from a Deployment?

**Difficulty:** Mid

**Answer:**

ReplicaSet ensures a specified number of pod replicas are running.

**ReplicaSet:**
- Manages pod replicas
- No update/rollback capabilities
- Used by Deployments internally
- Rarely created directly

**Deployment:**
- Manages ReplicaSets
- Provides updates, rollbacks, scaling
- Preferred for most use cases
- Creates ReplicaSets automatically

**When to Use ReplicaSet Directly:**
- Very simple use cases (rare)
- Usually use Deployment instead

**Example:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    # pod template
```

**Real-world Context:** Always use Deployment. ReplicaSet is lower-level component. Deployment = ReplicaSet + update capabilities.

**Follow-up:** Can you update a ReplicaSet? (Yes, but no rollback. Use Deployment for updates)

---

## ConfigMaps & Secrets

### Q15: What are ConfigMaps and Secrets, and when would you use each?

**Difficulty:** Mid

**Answer:**

**ConfigMaps:**
- Store non-sensitive configuration data
- Key-value pairs, files, or environment variables
- Use for: App configs, command-line arguments, config files

**Secrets:**
- Store sensitive data (passwords, tokens, keys)
- Base64 encoded (not encrypted by default)
- Use for: Passwords, API keys, TLS certificates

**Creating ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://db:5432/mydb"
  log_level: "info"
```

**Using in Pod:**
```yaml
env:
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database_url
```

**Creating Secret:**
```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123
```

**Real-world Context:** App config (API endpoint, log level) → ConfigMap. Database password → Secret. Mount as env vars or files.

**Follow-up:** Are Secrets encrypted? (Base64 encoded by default, not encrypted. Use external secret management or enable encryption at rest)

---

### Q16: How do you update ConfigMaps and Secrets without restarting pods?

**Difficulty:** Senior

**Answer:**

**Problem:** Changing ConfigMap/Secret doesn't automatically update pods using them.

**Solutions:**

**1. Restart Pods:**
```bash
kubectl rollout restart deployment/web-deployment
```

**2. Use Reloader (third-party):**
- Watches ConfigMap/Secret changes
- Automatically restarts deployments

**3. Volume Mounts with Reload:**
- Some apps watch mounted files and reload
- Update ConfigMap, app detects change

**4. Use Init Containers:**
- Copy ConfigMap to shared volume
- Main container reads from volume

**5. External Config Management:**
- Use service mesh (Istio) for config
- Use external config service

**Best Practice:**
- For env vars: Must restart pods
- For volume mounts: App must support file watching
- Consider using external config service for dynamic updates

**Real-world Context:** Update database URL in ConfigMap. Need to restart pods for changes to take effect. Use `kubectl rollout restart`.

**Follow-up:** What's the difference between env vars and volume mounts for ConfigMaps? (Env: set at startup, Volume: can be watched by app)

---

## Storage & Volumes

### Q17: Explain Kubernetes volumes and PersistentVolumes.

**Difficulty:** Mid

**Answer:**

**Volumes:**
- Storage attached to pods
- Lifecycle tied to pod (ephemeral)
- Types: emptyDir, hostPath, cloud volumes

**PersistentVolume (PV):**
- Cluster-wide storage resource
- Independent of pod lifecycle
- Provisioned by admin or dynamically

**PersistentVolumeClaim (PVC):**
- Request for storage by user
- Binds to PV
- Pods use PVCs

**Example:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Access Modes:**
- **ReadWriteOnce (RWO)**: Single node read-write
- **ReadOnlyMany (ROX)**: Multiple nodes read-only
- **ReadWriteMany (RWX)**: Multiple nodes read-write

**Real-world Context:** Database pod needs persistent storage. Create PVC, pod mounts it. If pod deleted, data persists. New pod can mount same PVC.

**Follow-up:** What's the difference between PV and PVC? (PV: storage resource, PVC: request for storage)

---

## Troubleshooting

### Q18: How do you troubleshoot a pod that won't start?

**Difficulty:** Mid

**Answer:**

**Check Pod Status:**
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Common Issues:**

**1. Pending:**
- No nodes available → Check node resources
- Image pull errors → Check image name, registry access
- Resource constraints → Check requests/limits
- Node selectors → Check affinity/selectors

**2. ImagePullBackOff:**
- Image doesn't exist → Verify image name/tag
- Registry authentication → Check imagePullSecrets
- Network issues → Check node network

**3. CrashLoopBackOff:**
- Container crashes → Check logs
- Application errors → Review application logs
- Resource limits → Check if OOMKilled
- Configuration errors → Check ConfigMaps/Secrets

**4. Error:**
- Init container failed → Check init container logs
- Startup command failed → Check command/args

**Debugging Steps:**
1. `kubectl describe pod` → Check events, status
2. `kubectl logs` → Check container logs
3. `kubectl exec` → Debug inside container
4. Check resource quotas, limits
5. Check node resources, taints

**Real-world Context:** Pod stuck in ImagePullBackOff. Check image name, verify registry access, check imagePullSecrets if private registry.

**Follow-up:** How do you debug a container that crashes immediately? (Check logs, describe pod for events, exec into container if possible, check resource limits)

---

### Q19: How do you debug networking issues in Kubernetes?

**Difficulty:** Senior

**Answer:**

**Common Networking Issues:**

**1. Pods can't communicate:**
- Check Service selectors match pod labels
- Verify Service endpoints: `kubectl get endpoints`
- Check NetworkPolicies blocking traffic
- Verify DNS resolution

**2. Service not accessible:**
- Check Service type (ClusterIP vs LoadBalancer)
- Verify Service selector matches pods
- Check pod readiness probes
- Verify ports match

**3. DNS not working:**
- Check CoreDNS pods running
- Verify DNS service exists
- Check pod DNS config
- Test DNS from pod: `nslookup service-name`

**Debugging Commands:**
```bash
# Check Service endpoints
kubectl get endpoints <service-name>

# Check DNS
kubectl run -it --rm debug --image=busybox -- nslookup <service-name>

# Test connectivity
kubectl exec <pod> -- curl <service-name>

# Check NetworkPolicies
kubectl get networkpolicies

# Check Service details
kubectl describe svc <service-name>
```

**Real-world Context:** Frontend can't reach backend. Check Service selector, verify endpoints, test DNS, check NetworkPolicies.

**Follow-up:** What's the difference between ClusterIP and NodePort? (ClusterIP: internal only, NodePort: exposed on node IP)

---

## Advanced Topics

### Q20: Explain Kubernetes RBAC and how to implement it.

**Difficulty:** Senior

**Answer:**

RBAC (Role-Based Access Control) controls who can do what in Kubernetes.

**Components:**

**Role/ClusterRole:**
- Defines permissions (verbs: get, list, create, delete)
- Role: namespace-scoped
- ClusterRole: cluster-scoped

**RoleBinding/ClusterRoleBinding:**
- Binds Role to users/groups/service accounts
- RoleBinding: namespace-scoped
- ClusterRoleBinding: cluster-scoped

**Example:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Best Practices:**
- Use least privilege
- Use service accounts for applications
- Review permissions regularly
- Use ClusterRoles for cluster admins

**Real-world Context:** Developer needs read-only access to pods in dev namespace. Create Role with get/list verbs, bind to user.

**Follow-up:** What's the difference between Role and ClusterRole? (Role: namespace-scoped, ClusterRole: cluster-scoped)

---

## Summary

Kubernetes is complex but powerful. Master these concepts: pods, services, deployments, ConfigMaps/Secrets, and troubleshooting. Practice with hands-on labs and understand the architecture.

**Next Steps:**
- Set up local Kubernetes (minikube, kind)
- Practice deploying applications
- Learn kubectl commands
- Study for CKA (Certified Kubernetes Administrator)
