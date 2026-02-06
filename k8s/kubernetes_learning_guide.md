# Kubernetes Learning Guide

## 📚 BASICS

### What is Kubernetes?
Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications across clusters of machines.

### Why Kubernetes when Docker already exists?
Docker is a container runtime that packages applications. Kubernetes manages those containers across multiple machines:
- **Docker**: Runs containers on a single machine
- **Kubernetes**: Manages, scales, and orchestrates containers across multiple machines

### Difference between Cluster and Node
- **Cluster**: A group of machines (nodes) working together to run containerized applications
- **Node**: A single machine (physical or virtual) that is part of a cluster and runs containers

### Control Plane vs Worker Node
| Aspect | Control Plane | Worker Node |
|--------|---------------|-------------|
| Purpose | Manages the cluster | Runs application containers |
| Role | Decision-making & scheduling | Executes workloads |
| Components | API Server, Scheduler, etcd, Controller Manager | Kubelet, Container Runtime, kube-proxy |

### What Problems Kubernetes Solves
✅ **Scaling**: Automatically scales applications up/down based on demand
✅ **Self-Healing**: Restarts failed containers, replaces and reschedules them
✅ **Management**: Handles rolling updates, rollbacks, and configuration management
✅ **Resource Optimization**: Efficiently distributes workloads across available resources
✅ **Load Balancing**: Distributes traffic across multiple instances
✅ **Storage Management**: Persists data across container restarts

---

## 🐳 PODS

### What is a Pod?
The smallest deployable unit in Kubernetes. A pod is a wrapper around one or more containers (usually Docker containers) that share networking and storage.

### Why Containers Run Inside Pods?
- **Networking**: Containers in a pod share the same IP address and network namespace
- **Storage Sharing**: Containers can share storage volumes
- **Co-location**: Allows tightly coupled sidecar containers to run together
- **Kubernetes Abstraction**: Pods provide Kubernetes' basic unit for management and scaling

### Single-Container vs Multi-Container Pod
| Aspect | Single-Container | Multi-Container |
|--------|------------------|-----------------|
| Use Case | Most common, standard applications | Sidecar patterns, logging, monitoring |
| Communication | Not applicable | Share localhost and volumes |
| Example | Web server | Web server + log shipper |

### Pod Lifecycle
```
Pending → Running → Succeeded/Failed
```
- **Pending**: Pod created but waiting for resources or containers to start
- **Running**: All containers are running
- **Succeeded**: All containers completed successfully (batch jobs)
- **Failed**: One or more containers failed

### Practice Commands
```bash
# Run a single pod
kubectl run my-pod --image=nginx

# Get all pods
kubectl get pods

# Get detailed information about a pod
kubectl describe pod my-pod

# Get pods with more details
kubectl get pods -o wide

# Delete a pod
kubectl delete pod my-pod
```

---

## ⚙️ WORKLOADS

### ReplicaSet
A Kubernetes resource that ensures a specified number of identical pods are running at all times.

**Key Features:**
- Maintains desired number of replicas
- Automatically replaces failed pods
- Rarely used directly (use Deployment instead)

```bash
# Create a ReplicaSet
kubectl apply -f replicaset.yaml

# Scale ReplicaSet to 5 replicas
kubectl scale rs my-replicaset --replicas=5

# Get ReplicaSets
kubectl get rs
```

### Deployment
The recommended way to manage ReplicaSets in real projects. Provides additional features like rolling updates and rollbacks.

**Key Features:**
- Manages ReplicaSets automatically
- Enables rolling updates
- Supports rollbacks
- More flexible than ReplicaSet

```bash
# Create a Deployment
kubectl apply -f deployment.yaml

# Get Deployments
kubectl get deployments

# Get deployment details
kubectl describe deployment my-deployment

# Update deployment image
kubectl set image deployment/my-deployment my-container=nginx:1.21

# Rollout status
kubectl rollout status deployment/my-deployment
```

### Rolling Updates
Gradually replace old pods with new ones without downtime.

```bash
# Check rollout history
kubectl rollout history deployment/my-deployment

# View specific revision details
kubectl rollout history deployment/my-deployment --revision=2

# Update deployment (triggers rolling update)
kubectl set image deployment/my-deployment app=app:v2.0
```

### Rollbacks
Revert to a previous version of a deployment.

```bash
# Rollback to previous version
kubectl rollout undo deployment/my-deployment

# Rollback to specific revision
kubectl rollout undo deployment/my-deployment --to-revision=1

# Pause rollout (useful for debugging)
kubectl rollout pause deployment/my-deployment

# Resume rollout
kubectl rollout resume deployment/my-deployment
```

---

## 🌐 SERVICES

### ClusterIP
Exposes service on a cluster-internal IP. Service is accessible only from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**Use Case**: Internal communication between microservices

### NodePort
Exposes the service on each node's IP at a static port (30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
  selector:
    app: my-app
```

**Use Case**: External access without load balancer

### LoadBalancer
Provisions an external load balancer (cloud-specific) to expose the service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**Use Case**: Production external exposure with cloud load balancers

### Headless Service
Service without ClusterIP. Returns pod IPs directly for custom service discovery.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  clusterIP: None  # Makes it headless
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**Use Case**: StatefulSets, custom DNS discovery patterns

---

## 💾 STORAGE

### Volumes
A storage abstraction that allows containers to access data that persists beyond container lifetime.

**Types:**
- **emptyDir**: Temporary storage deleted when pod terminates
- **hostPath**: Mounts a path from the host node
- **ConfigMap/Secret**: Mounts configuration or sensitive data

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

### PersistentVolume (PV)
A cluster-level resource representing physical storage (network storage, cloud storage, local disk).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-server.example.com
    path: "/exports"
```

**Access Modes:**
- **ReadWriteOnce**: Mounted as read-write by a single node
- **ReadOnlyMany**: Mounted as read-only by many nodes
- **ReadWriteMany**: Mounted as read-write by many nodes

### PersistentVolumeClaim (PVC)
A request for storage by a user. Binds to available PersistentVolumes.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

### StorageClass
Defines how PersistentVolumes are provisioned dynamically.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  iops: "1000"
```

### Dynamic Provisioning
Automatically creates PersistentVolumes when a PersistentVolumeClaim is created (no manual PV creation needed).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: fast-storage  # References StorageClass
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

## 📋 Quick Reference Commands

```bash
# Pod Management
kubectl run pod-name --image=image-name
kubectl get pods
kubectl get pods -o wide
kubectl describe pod pod-name
kubectl logs pod-name
kubectl exec -it pod-name -- /bin/bash
kubectl delete pod pod-name

# Deployment Management
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl describe deployment deploy-name
kubectl scale deployment deploy-name --replicas=3
kubectl set image deployment/deploy-name container-name=new-image:tag

# Rollout Management
kubectl rollout status deployment/deploy-name
kubectl rollout history deployment/deploy-name
kubectl rollout undo deployment/deploy-name
kubectl rollout undo deployment/deploy-name --to-revision=2

# Service Management
kubectl get services
kubectl describe service service-name
kubectl expose deployment deploy-name --type=NodePort --port=80

# Storage Management
kubectl get pv
kubectl get pvc
kubectl describe pv pv-name
kubectl describe pvc pvc-name

# Cluster Information
kubectl cluster-info
kubectl get nodes
kubectl describe node node-name
kubectl top nodes
kubectl top pods
```

---

## 🎯 Learning Path

1. **Start with**: Basics + Pods + Practice commands
2. **Then learn**: Deployments + Rolling updates
3. **Progress to**: Services and exposing applications
4. **Advanced**: Storage, StatefulSets, ConfigMaps, Secrets

---

**Happy Learning! 🚀**
