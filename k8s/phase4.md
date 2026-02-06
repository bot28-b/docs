# Kubernetes Phase 4: Resource Management & Health Checks

## 💾 RESOURCE REQUESTS & LIMITS

### What are Resource Requests and Limits?
- **Requests**: Minimum resources guaranteed to container
- **Limits**: Maximum resources container can use

### Why Set Resource Requests & Limits?
✅ Scheduler places pods efficiently
✅ Prevents resource starvation
✅ Prevents one pod from consuming all resources
✅ Enables better cluster utilization
✅ Supports autoscaling

### Resource Types
- **CPU**: Measured in millicores (m). 1000m = 1 vCPU
- **Memory**: Measured in Mi (mebibytes) or Gi (gibibytes)

### Define Requests and Limits
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: "100m"        # Guaranteed 100 millicores
        memory: "128Mi"    # Guaranteed 128 mebibytes
      limits:
        cpu: "500m"        # Max 500 millicores
        memory: "512Mi"    # Max 512 mebibytes
```

### Requests and Limits in Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
```

### View Resource Usage
```bash
# View pod resource requests
kubectl describe pod my-pod

# View node resource allocation
kubectl describe node node-1

# View resource usage (requires metrics server)
kubectl top nodes

# View pod resource usage
kubectl top pods

# View pod resource usage in namespace
kubectl top pods -n production

# View detailed resource usage
kubectl describe node node-1 | grep -A 5 "Allocated resources"
```

### Resource Units

| Unit | Meaning |
|------|---------|
| `m` (milli) | 1 CPU = 1000m |
| `1` | 1 full CPU core |
| `Mi` | 1 Mebibyte ≈ 1.05 MB |
| `Gi` | 1 Gibibyte ≈ 1.07 GB |

### QoS (Quality of Service) Classes
Kubernetes assigns QoS based on requests and limits.

```yaml
# Guaranteed (highest priority)
requests:
  cpu: "500m"
  memory: "512Mi"
limits:
  cpu: "500m"
  memory: "512Mi"
---
# Burstable (medium priority)
requests:
  cpu: "250m"
  memory: "256Mi"
limits:
  cpu: "1000m"
  memory: "1Gi"
---
# BestEffort (lowest priority - no requests/limits)
# Container evicted first under resource pressure
```

---

## ❤️ LIVENESS PROBES

### What is a Liveness Probe?
Checks if container is alive. Restarts container if probe fails.

### When to Use Liveness Probe
✅ Detect deadlocks
✅ Restart stuck applications
✅ Self-healing for hung processes
✅ Applications that need restart to recover

### Types of Liveness Probes

#### 1. HTTP Probe
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10  # Wait before first check
      periodSeconds: 5         # Check every 5 seconds
      timeoutSeconds: 2        # Timeout after 2 seconds
      failureThreshold: 3      # Fail after 3 failed checks
```

#### 2. TCP Probe
```yaml
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 15
  periodSeconds: 10
```

#### 3. Exec Probe (Command)
```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - pg_isready -U postgres
  initialDelaySeconds: 10
  periodSeconds: 10
```

### Liveness Probe Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Delay before first probe |
| `periodSeconds` | 10 | Interval between probes |
| `timeoutSeconds` | 1 | Timeout for probe response |
| `failureThreshold` | 3 | Failed attempts before restart |
| `successThreshold` | 1 | Successful attempts to mark healthy |

---

## 🚀 READINESS PROBES

### What is a Readiness Probe?
Checks if container is ready to accept traffic. Removes from service if probe fails.

### When to Use Readiness Probe
✅ Container not ready immediately on start
✅ Warm-up time needed
✅ Database connections pending
✅ Cache initialization
✅ External dependencies not ready

### Readiness Probe Examples

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3
      failureThreshold: 2
```

### Difference: Liveness vs Readiness

| Aspect | Liveness | Readiness |
|--------|----------|-----------|
| **Purpose** | Is pod alive? | Is pod ready for traffic? |
| **Action** | Restart pod | Remove from service endpoints |
| **Use Case** | Deadlock detection | Warm-up period |
| **Failure** | Container is restarted | Pod removed from load balancer |
| **Typical timeout** | Longer (avoid false restarts) | Shorter |

---

## ⏱️ STARTUP PROBES

### What is a Startup Probe?
Checks if application has started. Disables liveness/readiness until startup succeeds.

### When to Use Startup Probe
✅ Slow-starting applications
✅ Legacy apps with long boot time
✅ Complex initialization

### Startup Probe Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - name: app
    image: slow-app:latest
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      failureThreshold: 30      # Allow 30 failed attempts
      periodSeconds: 10         # Check every 10 seconds
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
```

### Probe Execution Flow
```
Start Pod
    ↓
Run Startup Probe
    ↓ (passes)
Disable startup probe
Enable liveness & readiness probes
    ↓
Container running + probes active
```

---

## 📈 HORIZONTAL POD AUTOSCALER (HPA)

### What is HPA?
Automatically scales number of pod replicas based on metrics.

### When to Use HPA
✅ Variable traffic patterns
✅ Peak hours need more replicas
✅ Cost optimization
✅ Automatic capacity management

### Prerequisites
- Metrics Server installed in cluster
- Requests defined for containers
- Deployment/StatefulSet with replicas

### HPA Based on CPU
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2         # Minimum pods
  maxReplicas: 10        # Maximum pods
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale at 70% CPU
```

### HPA Based on Custom Metrics
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### HPA Commands
```bash
# Create HPA
kubectl apply -f hpa.yaml

# Get HPAs
kubectl get hpa

# Watch HPA status
kubectl get hpa -w

# Describe HPA
kubectl describe hpa app-hpa

# View HPA detailed metrics
kubectl get hpa app-hpa -o yaml

# Delete HPA
kubectl delete hpa app-hpa
```

### Autoscaling Behavior
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
```

---

## 📊 VERTICAL POD AUTOSCALER (VPA)

### What is VPA?
Automatically adjusts CPU/memory requests based on actual usage.

### VPA Use Cases
✅ Right-sizing resource requests
✅ Reducing over-provisioned resources
✅ Cost optimization
✅ Improving cluster efficiency

### VPA Installation
```bash
# Install VPA (requires separate installation)
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

### VPA Example
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"  # Auto, Recreate, Off
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
```

---

## 🔄 RESOURCE QUOTAS (Namespace Level)

### Limit resources per namespace
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "100"
    persistentvolumeclaims: "10"
    requests.storage: "500Gi"
```

### View Quota Usage
```bash
kubectl get resourcequota -n production

kubectl describe resourcequota compute-quota -n production
```

---

## 📋 Quick Reference

```bash
# View resource requests/limits
kubectl describe pod <n>

# View node resources
kubectl describe node <node>

# View resource usage
kubectl top nodes
kubectl top pods

# Create HPA
kubectl autoscale deployment <n> --min=2 --max=10 --cpu-percent=70

# Get HPA
kubectl get hpa

# Watch HPA scaling
kubectl get hpa -w

# View resource quota
kubectl get resourcequota -n <namespace>
```

---

## 🎯 Best Practices

✅ Always set resource requests and limits
✅ Use readiness probes for graceful startup
✅ Use liveness probes for crash detection
✅ Set appropriate probe timeouts
✅ Use HPA for variable workloads
✅ Monitor actual usage and adjust
✅ Set resource quotas per namespace
✅ Use startup probes for slow applications

---

## 🎯 Key Takeaways

- **Requests** = Guaranteed resources (for scheduling)
- **Limits** = Maximum resources (for containment)
- **Liveness** = Container alive check (restart on failure)
- **Readiness** = Traffic ready check (remove from service on failure)
- **HPA** = Auto-scale pods based on metrics
- **Resource Quota** = Limit namespace resource usage

**Next Phase**: Learn about Ingress, Network Policies, and RBAC!
