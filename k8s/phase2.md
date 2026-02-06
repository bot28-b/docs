# Kubernetes Phase 2: Configuration & Namespaces

## 📝 CONFIGMAPS

### What is a ConfigMap?
A Kubernetes object that stores non-sensitive configuration data as key-value pairs. ConfigMaps decouple configuration from application code.

### When to Use ConfigMap
✅ Configuration files
✅ Environment variables
✅ Command-line arguments
✅ Application settings (non-sensitive)

❌ Don't use for passwords, API keys, credentials (use Secrets instead)

### Create ConfigMap from Literal
```bash
# Create from literal values
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_ROLE=backend

# Get ConfigMaps
kubectl get configmaps

# Describe ConfigMap
kubectl describe configmap app-config
```

### Create ConfigMap from File
```bash
# From single file
kubectl create configmap app-config --from-file=config.txt

# From entire directory
kubectl create configmap app-config --from-file=/path/to/config/

# Specify key name
kubectl create configmap app-config --from-file=config.properties=./app.properties
```

### Define ConfigMap in YAML
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_ROLE: backend
  database.url: "postgresql://db:5432/myapp"
  app.properties: |
    server.port=8080
    server.servlet.context-path=/api
    logging.level=INFO
```

### Use ConfigMap in Pod - Environment Variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # From ConfigMap key-value pairs
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: APP_ROLE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ROLE
```

### Use ConfigMap in Pod - As Volume (Files)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### Use ConfigMap in Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
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
        envFrom:
        - configMapRef:
            name: app-config
```

### Edit ConfigMap
```bash
# Edit ConfigMap
kubectl edit configmap app-config

# Apply from new YAML file
kubectl apply -f new-config.yaml --force
```

---

## 🔐 SECRETS

### What is a Secret?
A Kubernetes object that stores sensitive data (passwords, API keys, tokens, certificates) in an encrypted or base64-encoded format.

### Types of Secrets

| Type | Use Case |
|------|----------|
| `Opaque` | Default type, arbitrary user-defined data |
| `kubernetes.io/service-account-token` | Service account token |
| `kubernetes.io/dockercfg` | Serialized Docker config |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/basic-auth` | Basic authentication |
| `kubernetes.io/ssh-auth` | SSH authentication |
| `kubernetes.io/tls` | TLS certificate and key |

### Create Secret - Literal
```bash
# Create from literals
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secretpass123

# Get Secrets
kubectl get secrets

# Describe Secret
kubectl describe secret db-secret

# View Secret (base64 encoded)
kubectl get secret db-secret -o yaml
```

### Create Secret from File
```bash
# From single file
kubectl create secret generic db-secret --from-file=username=./username.txt --from-file=password=./password.txt

# From directory
kubectl create secret generic certs --from-file=/path/to/certs/
```

### Define Secret in YAML (Opaque)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=  # base64 encoded 'admin'
  password: c2VjcmV0cGFzczEyMw==  # base64 encoded 'secretpass123'
```

### Encode/Decode Values
```bash
# Encode to base64
echo -n "admin" | base64
# Output: YWRtaW4=

# Decode from base64
echo "YWRtaW4=" | base64 -d
# Output: admin
```

### Use Secret - Environment Variables
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### Use Secret - As Volume (Files)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
```

### Use Secret - Docker Registry Credentials
```bash
# Create docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=myuser \
  --docker-password=mypass \
  --docker-email=user@example.com
```

### Use in Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: myregistry.com/myapp:latest
```

### Use Secret in Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
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
        envFrom:
        - secretRef:
            name: db-secret
```

### Best Practices for Secrets
✅ Use RBAC to limit secret access
✅ Enable encryption at rest in etcd
✅ Use tools like Sealed Secrets or External Secrets Operator
✅ Rotate secrets regularly
✅ Never commit secrets to version control
✅ Use `.gitignore` for sensitive files

---

## 🏷️ NAMESPACES

### What is a Namespace?
A way to divide cluster resources between multiple users or teams. Provides scope for names and resource isolation.

### Why Use Namespaces?
✅ Organize resources (dev, staging, prod)
✅ Team isolation and multi-tenancy
✅ Resource quotas per team
✅ RBAC policies scoped to namespaces
✅ Reduce naming conflicts

### Default Namespaces
```
default          - Default namespace if none specified
kube-system      - System components (DNS, controller manager)
kube-public      - Publicly readable resources
kube-node-lease  - Node heartbeat data
```

### Create Namespace
```bash
# Create namespace
kubectl create namespace dev

# Create via YAML
kubectl apply -f namespace.yaml

# List namespaces
kubectl get namespaces

# Get details
kubectl describe namespace dev
```

### Define Namespace in YAML
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
```

### Switch Namespace (Context)
```bash
# Set current namespace
kubectl config set-context --current --namespace=dev

# Verify current namespace
kubectl config current-context

# List all contexts
kubectl config get-contexts
```

### Create Resources in Specific Namespace
```bash
# Specify namespace with -n flag
kubectl apply -f deployment.yaml -n dev

# Run pod in namespace
kubectl run my-pod --image=nginx -n dev

# Get pods from specific namespace
kubectl get pods -n dev

# Get pods from all namespaces
kubectl get pods --all-namespaces

# Delete resource from namespace
kubectl delete pod my-pod -n dev
```

### Define Resource with Namespace in YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  namespace: dev  # Explicitly define namespace
spec:
  containers:
  - name: app
    image: nginx
```

### Resource Quotas
Limit resources (CPU, memory, storage) per namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "100"
    persistentvolumeclaims: "10"
```

### View Resource Quota
```bash
# Get resource quotas
kubectl get resourcequota -n dev

# Describe quota
kubectl describe resourcequota dev-quota -n dev
```

### Network Policies in Namespaces
Restrict network traffic between namespaces.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: dev
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
      namespaceSelector:
        matchLabels:
          name: dev
```

---

## 🔗 CONFIGMAP + SECRET + NAMESPACE EXAMPLE

```yaml
# 1. Create namespace
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
# 2. Create ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: production
  LOG_LEVEL: INFO
  DATABASE_HOST: postgres.production.svc.cluster.local

---
# 3. Create Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: production
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded

---
# 4. Create Deployment using ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
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
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
```

---

## 📋 Quick Reference Commands

```bash
# ConfigMap
kubectl create configmap <name> --from-literal=key=value
kubectl get configmaps
kubectl describe configmap <name>
kubectl delete configmap <name>
kubectl edit configmap <name>

# Secret
kubectl create secret generic <name> --from-literal=key=value
kubectl get secrets
kubectl describe secret <name>
kubectl delete secret <name>

# Namespace
kubectl create namespace <name>
kubectl get namespaces
kubectl config set-context --current --namespace=<name>
kubectl get pods -n <namespace>
kubectl delete namespace <name>

# Resource Quota
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>
```

---

## 🎯 Key Takeaways

- **ConfigMaps** = Non-sensitive config (files, variables, settings)
- **Secrets** = Sensitive data (passwords, tokens, credentials)
- **Namespaces** = Logical cluster division (teams, environments)
- Use all three together for organized, secure, multi-environment deployments

**Next Phase**: Learn about StatefulSets, DaemonSets, and Jobs!
