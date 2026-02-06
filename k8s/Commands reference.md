# Kubernetes Command Reference Card

## 🔧 INSTALLATION & SETUP

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Minikube (local cluster)
curl -Lo minikube https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
chmod +x minikube
sudo mv minikube /usr/local/bin/

# Start Minikube
minikube start

# Check cluster info
kubectl cluster-info
kubectl get nodes
```

---

## 📦 CONTEXT & CONFIGURATION

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# View kubeconfig
kubectl config view

# Set default namespace
kubectl config set-context --current --namespace=<namespace>

# Create kubeconfig entry
kubectl config set-cluster <cluster> --server=<url> --certificate-authority=<ca>
kubectl config set-credentials <user> --client-certificate=<cert> --client-key=<key>
kubectl config set-context <context> --cluster=<cluster> --user=<user>
```

---

## 📋 RESOURCE MANAGEMENT

### Get Resources
```bash
# List resources
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods --all-namespaces
kubectl get pods -o wide
kubectl get pods -o json
kubectl get pods -o yaml

# Get specific resource
kubectl get pod <pod-name>
kubectl get pod <pod-name> -n <namespace>

# Get resources with labels
kubectl get pods --selector app=myapp
kubectl get pods -l app=myapp

# Get resources sorted
kubectl get pods --sort-by=.metadata.creationTimestamp

# Watch resources
kubectl get pods -w
kubectl get pods --watch
```

### Describe Resources
```bash
# Detailed info
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
kubectl describe service <service-name>
kubectl describe node <node-name>

# Get specific field
kubectl get pod <pod> -o jsonpath='{.status.phase}'
```

### Create Resources
```bash
# From YAML file
kubectl apply -f deployment.yaml
kubectl create -f deployment.yaml

# From URL
kubectl apply -f https://example.com/deployment.yaml

# From directory
kubectl apply -f ./manifests/

# Dry run (preview)
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server
```

### Edit Resources
```bash
# Edit in default editor
kubectl edit pod <pod-name>
kubectl edit deployment <deployment-name>

# Edit with specific editor
EDITOR=nano kubectl edit pod <pod-name>

# Apply changes
kubectl apply -f deployment.yaml --force
kubectl patch deployment <name> -p '{"spec":{"replicas":3}}'
```

### Delete Resources
```bash
# Delete resource
kubectl delete pod <pod-name>
kubectl delete deployment <deployment-name>
kubectl delete -f deployment.yaml

# Delete all pods in namespace
kubectl delete pod --all -n <namespace>

# Delete with cascading
kubectl delete deployment <name> --cascade=background
kubectl delete deployment <name> --cascade=orphan

# Force delete
kubectl delete pod <pod-name> --grace-period=0 --force
```

---

## 🚀 PODS & CONTAINERS

### Pod Operations
```bash
# Get pods
kubectl get pods
kubectl get pods -n <namespace>
kubectl get pods -o wide

# Describe pod
kubectl describe pod <pod-name>

# Create pod
kubectl run my-pod --image=nginx:latest

# Execute command
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container> -- ls /app

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container>
kubectl logs <pod-name> -f
kubectl logs <pod-name> --tail=100
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --previous

# Port forward
kubectl port-forward <pod-name> 8080:8080
kubectl port-forward svc/<service> 5432:5432

# Copy files
kubectl cp <pod>:/path/file.txt ./local-file.txt
kubectl cp ./local-file.txt <pod>:/path/file.txt

# Delete pod
kubectl delete pod <pod-name>
```

---

## 📦 DEPLOYMENTS

```bash
# Create deployment
kubectl create deployment my-app --image=myapp:1.0
kubectl apply -f deployment.yaml

# Get deployments
kubectl get deployments
kubectl describe deployment <name>

# Scale deployment
kubectl scale deployment <name> --replicas=5
kubectl scale deployment <name> --replicas=0

# Update image
kubectl set image deployment/<name> app=myapp:2.0
kubectl set image deployment/<name> app=myapp:2.0 --record

# Edit deployment
kubectl edit deployment <name>

# View deployment details
kubectl describe deployment <name>
kubectl get deployment <name> -o yaml

# Delete deployment
kubectl delete deployment <name>
```

---

## 🔄 ROLLOUTS & UPDATES

```bash
# Check rollout status
kubectl rollout status deployment/<name>
kubectl rollout status deployment/<name> -w

# View rollout history
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=2

# Rollback to previous
kubectl rollout undo deployment/<name>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=2

# Pause rollout
kubectl rollout pause deployment/<name>

# Resume rollout
kubectl rollout resume deployment/<name>

# Restart deployment
kubectl rollout restart deployment/<name>
```

---

## 🎁 REPLICASETS & STATEFULSETS

```bash
# ReplicaSet
kubectl get replicasets
kubectl describe replicaset <name>
kubectl scale replicaset <name> --replicas=3
kubectl delete replicaset <name>

# StatefulSet
kubectl get statefulsets
kubectl describe statefulset <name>
kubectl scale statefulset <name> --replicas=5
kubectl get pvc  # View persistent volume claims
kubectl delete statefulset <name>
```

---

## 📋 SERVICES

```bash
# Get services
kubectl get services
kubectl get svc
kubectl describe service <name>

# Create service
kubectl expose deployment <name> --type=ClusterIP --port=80
kubectl expose deployment <name> --type=NodePort --port=80
kubectl expose deployment <name> --type=LoadBalancer --port=80

# Port forward to service
kubectl port-forward svc/<service> 8080:8080

# Get service endpoints
kubectl get endpoints <service>

# Delete service
kubectl delete service <name>
```

---

## 🌐 INGRESS

```bash
# Get ingresses
kubectl get ingress
kubectl describe ingress <name>

# Apply ingress
kubectl apply -f ingress.yaml

# Create TLS secret
kubectl create secret tls <name> --cert=tls.crt --key=tls.key

# Edit ingress
kubectl edit ingress <name>

# Delete ingress
kubectl delete ingress <name>
```

---

## 📝 CONFIGMAPS & SECRETS

```bash
# ConfigMap
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=config.txt
kubectl get configmap <name>
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>
kubectl delete configmap <name>

# Secret
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret generic <name> --from-file=password.txt
kubectl get secret <name>
kubectl get secret <name> -o yaml
kubectl describe secret <name>
kubectl delete secret <name>

# Docker registry secret
kubectl create secret docker-registry <name> \
  --docker-server=docker.io \
  --docker-username=user \
  --docker-password=pass

# TLS secret
kubectl create secret tls <name> --cert=cert.crt --key=key.key
```

---

## 🏷️ NAMESPACES

```bash
# Create namespace
kubectl create namespace <name>
kubectl apply -f namespace.yaml

# Get namespaces
kubectl get namespaces
kubectl describe namespace <name>

# Switch namespace
kubectl config set-context --current --namespace=<name>

# List resources in namespace
kubectl get pods -n <namespace>
kubectl get all -n <namespace>

# Create resource in namespace
kubectl apply -f deployment.yaml -n <namespace>

# Delete namespace (deletes all resources)
kubectl delete namespace <name>
```

---

## 🎯 JOBS & CRONJOBS

```bash
# Create job
kubectl apply -f job.yaml
kubectl create job <name> --image=image:tag

# Get jobs
kubectl get jobs
kubectl describe job <name>

# View job logs
kubectl logs job/<name>

# Delete job
kubectl delete job <name>

# CronJob
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl create job --from=cronjob/<name> manual-<name>
kubectl patch cronjob <name> -p '{"spec":{"suspend":true}}'
```

---

## 💾 STORAGE

```bash
# Get persistent volumes
kubectl get pv
kubectl describe pv <name>

# Get persistent volume claims
kubectl get pvc
kubectl get pvc -n <namespace>
kubectl describe pvc <name>

# Get storage classes
kubectl get storageclasses
kubectl describe storageclass <name>

# Create PVC
kubectl apply -f pvc.yaml
```

---

## 📊 MONITORING & DEBUGGING

```bash
# Resource usage
kubectl top nodes
kubectl top pods
kubectl top pods -n <namespace>

# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl describe node <node>

# Events
kubectl get events
kubectl get events -n <namespace>
kubectl get events --sort-by='.lastTimestamp'

# Logs
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> -f
kubectl logs <pod> --previous

# Describe
kubectl describe pod <pod>
kubectl describe deployment <deployment>

# Debug pod
kubectl debug pod/<pod>
kubectl debug pod/<pod> -it --image=debug-image

# Port forward
kubectl port-forward pod/<pod> 8080:8080
kubectl port-forward svc/<service> 3000:3000
```

---

## 🔐 RBAC & SECURITY

```bash
# ServiceAccount
kubectl create serviceaccount <name>
kubectl get serviceaccounts
kubectl describe serviceaccount <name>

# Roles
kubectl get roles
kubectl describe role <name>
kubectl create role <name> --verb=get --resource=pods

# RoleBinding
kubectl get rolebindings
kubectl create rolebinding <name> --clusterrole=<role> --serviceaccount=<ns>:<sa>

# ClusterRole
kubectl get clusterroles
kubectl create clusterrole <name> --verb=get --resource=pods

# ClusterRoleBinding
kubectl get clusterrolebindings
kubectl create clusterrolebinding <name> --clusterrole=<role> --serviceaccount=<ns>:<sa>

# Check permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa
```

---

## 🎯 HELM COMMANDS

```bash
# Repository
helm repo add <name> <url>
helm repo list
helm repo update
helm repo remove <name>

# Search
helm search repo <chart>
helm search repo <chart> --version
helm show values <chart>
helm show chart <chart>

# Install
helm install <release> <chart>
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --set key=value

# Upgrade
helm upgrade <release> <chart>
helm upgrade <release> <chart> -f values.yaml
helm upgrade <release> <chart> --reuse-values

# Rollback
helm rollback <release>
helm rollback <release> <revision>

# List & Status
helm list
helm status <release>
helm history <release>

# Uninstall
helm uninstall <release>

# Chart
helm create <chart>
helm lint <chart>
helm template <release> <chart>
helm package <chart>
```

---

## 📋 USEFUL FLAGS

```bash
-n, --namespace          Specify namespace
-A, --all-namespaces     All namespaces
-l, --selector           Label selector
-o, --output             Output format (json, yaml, wide)
-w, --watch              Watch for changes
-f, --filename           YAML file
--dry-run               Preview changes
--sort-by               Sort results
--tail                  Last N lines
--since                 Logs since
-c, --container         Container name
-p, --previous          Previous container
-it                     Interactive terminal
--cascade               Delete cascading
--grace-period          Grace period for deletion
--force                 Force deletion
--record                Record changes
--reuse-values          Reuse values from previous
--atomic                Automatic rollback on failure
```

---

## 🔍 DEBUGGING TECHNIQUES

```bash
# Check pod status
kubectl describe pod <pod>

# View pod logs
kubectl logs <pod>
kubectl logs <pod> --previous

# Execute in pod
kubectl exec -it <pod> -- /bin/bash

# Check events
kubectl get events

# Port forward
kubectl port-forward <pod> 8080:8080

# Check resource usage
kubectl top pods

# Check node status
kubectl describe node <node>

# Check service connectivity
kubectl exec <pod> -- curl <service>:80

# DNS test
kubectl exec <pod> -- nslookup kubernetes.default
```

---

## 💡 COMMON PATTERNS

```bash
# Get all resources in namespace
kubectl get all -n <namespace>

# Delete all pods
kubectl delete pod --all

# Scale to zero
kubectl scale deployment <name> --replicas=0

# Force delete stuck pod
kubectl delete pod <pod> --grace-period=0 --force

# Restart pods
kubectl rollout restart deployment/<name>

# View resource YAML
kubectl get <resource> <name> -o yaml

# Edit resource
kubectl edit <resource> <name>

# Watch changes
kubectl get <resource> --watch

# Set resource requests
kubectl set resources deployment <name> --requests=cpu=250m,memory=256Mi
```

---

## 🚀 Useful Aliases

```bash
alias k=kubectl
alias kgs='kubectl get statefulsets'
alias kgd='kubectl get deployments'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias krm='kubectl delete'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kel='kubectl exec -it'
alias kl='kubectl logs'
alias kc='kubectl config'
```

Add to `~/.bashrc` or `~/.zshrc`

---

## 📞 Quick Help

```bash
# Get help
kubectl help
kubectl <command> --help
kubectl explain pod
kubectl explain pod.spec

# API resources
kubectl api-resources
kubectl api-versions

# Get version
kubectl version
```

---

**Remember: Practice these commands regularly to build muscle memory!**
