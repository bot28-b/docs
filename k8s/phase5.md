# Kubernetes Phase 5: Networking & Security

## 🔀 INGRESS

### What is Ingress?
Manages external HTTP/HTTPS access to services in a cluster. Routes traffic based on hostnames and paths.

### Why Ingress?
- **LoadBalancer Problem**: LoadBalancer service creates one load balancer per service (expensive)
- **Ingress Solution**: One load balancer routes traffic to multiple services
- **SSL/TLS**: Centralized certificate management
- **URL Routing**: Route based on path and hostname
- **Cost Effective**: Fewer external load balancers

### Service vs Ingress

| Aspect | Service (LoadBalancer) | Ingress |
|--------|----------------------|---------|
| **Load Balancers** | One per service | One for all |
| **Cost** | Expensive | Cheap |
| **SSL/TLS** | Per service | Centralized |
| **Routing** | Basic (L4) | Advanced (L7) |
| **Hostnames** | N/A | Multiple hostnames |

### Ingress Controller
An Ingress Controller (Nginx, HAProxy, Traefik) interprets Ingress rules and configures routing.

### Install Nginx Ingress Controller
```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Simple Ingress Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Ingress with Multiple Services
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

### Path Types
```yaml
pathType: Prefix      # /api matches /api, /api/v1, /api/users
pathType: Exact       # /api matches only /api exactly
pathType: ImplementationSpecific  # Controller-specific
```

### Ingress with TLS/SSL
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls  # TLS certificate secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Create TLS Secret
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=example.com"

# Create TLS secret
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key

# Verify secret
kubectl get secret example-tls -o yaml
```

### Ingress with Annotations
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "10"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

### Ingress Commands
```bash
# Create Ingress
kubectl apply -f ingress.yaml

# Get Ingresses
kubectl get ingress

# Get Ingress details
kubectl describe ingress simple-ingress

# View Ingress IP/hostname
kubectl get ingress -o wide

# Get Ingress YAML
kubectl get ingress simple-ingress -o yaml

# Edit Ingress
kubectl edit ingress simple-ingress

# Delete Ingress
kubectl delete ingress simple-ingress
```

---

## 🔒 NETWORK POLICIES

### What is a Network Policy?
Defines rules for pod-to-pod and pod-to-external communication. Provides network segmentation.

### When to Use Network Policy
✅ Restrict traffic between pods
✅ Isolate namespaces
✅ Compliance & security
✅ Zero-trust networking
✅ Prevent lateral movement

### Network Policy Basics
- **Default**: No restrictions (all pods can communicate)
- **Explicit Deny**: Define what's allowed, rest is denied
- **Selector-based**: Uses labels for pod selection

### Default Deny All Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress
```

### Allow Specific Pod Communication
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Communication Within Namespace
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
    - namespaceSelector:
        matchLabels:
          name: dev
```

### Allow Communication Across Namespaces
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cross-namespace
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Egress Network Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
spec:
  podSelector:
    matchLabels:
      app: restricted-app
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53  # DNS
    - protocol: UDP
      port: 53
```

### Network Policy Commands
```bash
# Create Network Policy
kubectl apply -f network-policy.yaml

# Get Network Policies
kubectl get networkpolicies

# Get Network Policies in namespace
kubectl get networkpolicies -n dev

# Describe Network Policy
kubectl describe networkpolicy allow-frontend-to-backend

# Delete Network Policy
kubectl delete networkpolicy allow-frontend-to-backend
```

---

## 🔐 RBAC (Role-Based Access Control)

### What is RBAC?
Defines who can do what in Kubernetes cluster. Controls API access using roles and role bindings.

### RBAC Components
1. **ServiceAccount**: User identity in Kubernetes
2. **Role**: Set of permissions
3. **RoleBinding**: Binds Role to ServiceAccount
4. **ClusterRole**: Cluster-wide role
5. **ClusterRoleBinding**: Cluster-wide role binding

### Create ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
```

### Create Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get"]
```

### Bind Role to ServiceAccount
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: dev
```

### ClusterRole Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin-custom
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### ClusterRoleBinding Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin-custom
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: dev
```

### Common Verbs
```
get          - Read single resource
list         - List resources
create       - Create new resources
update       - Update existing resources
patch        - Partial update
delete       - Delete resources
watch        - Stream events
exec         - Execute commands in pod
logs         - Read pod logs
port-forward - Forward ports
```

### API Groups
```
""                     - Core API (pods, services, etc.)
"apps"                 - Deployments, StatefulSets, etc.
"batch"                - Jobs, CronJobs
"storage.k8s.io"       - PersistentVolumes, StorageClasses
"rbac.authorization"   - Roles, RoleBindings
"networking.k8s.io"    - Network Policies, Ingress
"*"                    - All API groups
```

### RBAC Commands
```bash
# Create ServiceAccount
kubectl create serviceaccount app-sa -n dev

# Get ServiceAccounts
kubectl get serviceaccounts -n dev

# Describe ServiceAccount
kubectl describe serviceaccount app-sa -n dev

# Get Roles
kubectl get roles -n dev

# Describe Role
kubectl describe role pod-reader -n dev

# Get RoleBindings
kubectl get rolebindings -n dev

# Create RoleBinding
kubectl create rolebinding pod-reader-binding \
  --clusterrole=pod-reader \
  --serviceaccount=dev:app-sa \
  -n dev

# Get ClusterRoles
kubectl get clusterroles

# Get ClusterRoleBindings
kubectl get clusterrolebindings

# Check if user can perform action
kubectl auth can-i get pods --as=system:serviceaccount:dev:app-sa -n dev
```

### Example: Developer RBAC
```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: developer
  namespace: dev

---
# Role with limited permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "pods/logs"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create", "get"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
subjects:
- kind: ServiceAccount
  name: developer
  namespace: dev
```

---

## 📊 RBAC vs Network Policy

| Aspect | RBAC | Network Policy |
|--------|------|----------------|
| **Controls** | API access | Network traffic |
| **Authentication** | User/ServiceAccount | Pod labels |
| **Layer** | Kubernetes API | Network layer |
| **Who** | Who accesses API | What pods communicate |
| **How** | Roles & Bindings | Ingress/Egress rules |

---

## 🎯 Security Best Practices

✅ Use Network Policies for traffic control
✅ Implement RBAC with least privilege
✅ Use separate ServiceAccounts per app
✅ Enable Pod Security Policies
✅ Use Ingress for external access
✅ Enable TLS for all traffic
✅ Scan container images for vulnerabilities
✅ Use resource quotas and limits

---

## 📋 Quick Reference Commands

```bash
# Ingress
kubectl get ingress
kubectl describe ingress <n>
kubectl create secret tls <n> --cert=cert.crt --key=key.key

# Network Policy
kubectl get networkpolicies
kubectl describe networkpolicy <n>
kubectl apply -f network-policy.yaml

# RBAC
kubectl get serviceaccounts
kubectl get roles
kubectl get rolebindings
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl auth can-i get pods --as=<user>
```

---

## 🎯 Key Takeaways

- **Ingress** = External HTTP/HTTPS routing to services
- **Network Policy** = Pod-to-pod network segmentation
- **RBAC** = API access control for users & ServiceAccounts
- Combine all three for complete network security

**Next Phase**: Learn about Helm, Monitoring, and Logging!
