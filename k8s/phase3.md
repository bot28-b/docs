# Kubernetes Phase 3: Advanced Workloads

## 📊 STATEFULSETS

### What is a StatefulSet?
A workload resource for managing stateful applications. Pods have persistent, ordered identities and stable storage.

### When to Use StatefulSet
✅ Databases (MySQL, PostgreSQL, MongoDB)
✅ Message queues (RabbitMQ, Kafka)
✅ Distributed systems needing unique identities
✅ Applications requiring stable hostnames

❌ Don't use for stateless applications (use Deployment)

### Characteristics of StatefulSets
- **Ordered Pod Names**: `mysql-0`, `mysql-1`, `mysql-2` (predictable naming)
- **Stable Network Identity**: Each pod has stable hostname
- **Persistent Storage**: Each pod gets dedicated PVC
- **Ordered Deployment**: Pods created/deleted in order
- **Ordered Updates**: Rolling updates respect order

### Difference: Deployment vs StatefulSet

| Aspect | Deployment | StatefulSet |
|--------|-----------|------------|
| Pod Names | Random hashes | Ordered (pod-0, pod-1) |
| Network Identity | Generic | Stable hostnames |
| Storage | Shared | Per-pod dedicated storage |
| Use Case | Stateless apps | Stateful apps |
| Scaling | Fast (unordered) | Ordered, slower |

### StatefulSet YAML Example
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql  # Required for network identity
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  # Headless Service required
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

### Headless Service for StatefulSet
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless service
  selector:
    app: mysql
  ports:
  - port: 3306
```

### StatefulSet Commands
```bash
# Create StatefulSet
kubectl apply -f statefulset.yaml

# Get StatefulSets
kubectl get statefulsets

# Get pods (notice ordered names)
kubectl get pods
# Output: mysql-0, mysql-1, mysql-2

# Describe StatefulSet
kubectl describe statefulset mysql

# Scale StatefulSet
kubectl scale statefulset mysql --replicas=5

# Get persistent volume claims
kubectl get pvc

# Delete StatefulSet (keep data)
kubectl delete statefulset mysql

# Delete StatefulSet and PVCs
kubectl delete statefulset mysql --cascade=orphan
```

### Access StatefulSet Pods
```bash
# Pod hostname (accessible within cluster)
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local

# Exec into specific pod
kubectl exec -it mysql-0 -- /bin/bash
```

---

## 🔄 DAEMONSETS

### What is a DaemonSet?
Ensures that all (or some) nodes run a copy of a pod. As nodes are added/removed, pods are added/removed accordingly.

### When to Use DaemonSet
✅ Node monitoring (Prometheus, Datadog)
✅ Logging agents (Fluentd, Logstash)
✅ Container runtimes (kubelet alternatives)
✅ Node maintenance scripts
✅ Network plugins

### Characteristics of DaemonSets
- **One Pod Per Node**: Runs on every node automatically
- **Auto-Scaling**: Adds pod when new node joins
- **Auto-Cleanup**: Removes pod when node leaves
- **No Replicas Field**: Number equals number of nodes
- **Efficient**: Minimal resource overhead

### DaemonSet YAML Example
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: logs
          mountPath: /var/log
      volumes:
      - name: logs
        hostPath:
          path: /var/log
```

### Node Selector in DaemonSet
Run DaemonSet on specific nodes only.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  selector:
    matchLabels:
      app: gpu-monitor
  template:
    metadata:
      labels:
        app: gpu-monitor
    spec:
      nodeSelector:
        gpu: "true"  # Only nodes with gpu=true label
      containers:
      - name: gpu-monitor
        image: gpu-monitor:latest
```

### Tolerations for DaemonSet
Allow pods to run on nodes with taints.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: system-monitor
spec:
  selector:
    matchLabels:
      app: system-monitor
  template:
    metadata:
      labels:
        app: system-monitor
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: monitor
        image: monitor:latest
```

### DaemonSet Commands
```bash
# Create DaemonSet
kubectl apply -f daemonset.yaml

# Get DaemonSets
kubectl get daemonsets

# Get DaemonSet pods (one per node)
kubectl get pods -l app=fluentd

# Describe DaemonSet
kubectl describe daemonset fluentd

# Update DaemonSet (rolling update)
kubectl set image daemonset/fluentd fluentd=fluent/fluentd:v1.2

# Delete DaemonSet
kubectl delete daemonset fluentd
```

---

## 🎯 JOBS

### What is a Job?
Runs one or more pods to completion. Pod restarts on failure until successful completion or timeout.

### When to Use Job
✅ Batch processing
✅ Database migrations
✅ Backups
✅ Report generation
✅ One-time tasks
✅ Scheduled tasks (with CronJob)

### Characteristics of Jobs
- **Run to Completion**: Pod exits after task completes
- **Auto-Retry**: Retries failed pods
- **Parallelism**: Run multiple pods in parallel
- **Cleanup**: Pods cleaned after completion

### Job YAML Example
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-db
spec:
  backoffLimit: 3  # Retry 3 times on failure
  activeDeadlineSeconds: 300  # Timeout after 5 minutes
  completions: 1  # Number of successful pods needed
  parallelism: 1  # Number of pods running in parallel
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["backup-script.sh"]
        args: ["--database", "mydb"]
      restartPolicy: Never  # Don't restart on failure
```

### Job with Parallelism
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-process
spec:
  completions: 10  # Need 10 successful pods
  parallelism: 3   # Run 3 pods in parallel
  template:
    spec:
      containers:
      - name: processor
        image: image-processor:latest
      restartPolicy: Never
```

### Job Commands
```bash
# Create Job
kubectl apply -f job.yaml

# Get Jobs
kubectl get jobs

# Get job details
kubectl describe job backup-db

# Get pods created by job
kubectl get pods -l job-name=backup-db

# View job logs
kubectl logs -l job-name=backup-db

# Delete Job (keep pods)
kubectl delete job backup-db

# Delete Job and pods
kubectl delete job backup-db --cascade=background

# Wait for job completion
kubectl wait --for=condition=complete job/backup-db --timeout=300s
```

---

## ⏰ CRONJOBS

### What is a CronJob?
Creates Jobs on a schedule (cron syntax). Useful for recurring tasks.

### When to Use CronJob
✅ Daily database backups
✅ Hourly report generation
✅ Regular cleanup tasks
✅ Scheduled data synchronization
✅ Periodic health checks

### CronJob YAML Example
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM every day (UTC)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["backup-script.sh"]
            args: ["--database", "mydb"]
          restartPolicy: OnFailure
      backoffLimit: 3
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1      # Keep last 1 failed job
  startingDeadlineSeconds: 300   # Start within 5 mins of scheduled time
```

### Cron Schedule Syntax
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

### Common Cron Schedules
```
0 0 * * *      # Daily at midnight
0 */6 * * *    # Every 6 hours
0 2 * * 0      # Every Sunday at 2 AM
0 0 1 * *      # First day of month at midnight
0 0 * * 1-5    # Weekdays at midnight
*/5 * * * *    # Every 5 minutes
```

### CronJob Commands
```bash
# Create CronJob
kubectl apply -f cronjob.yaml

# Get CronJobs
kubectl get cronjobs

# Describe CronJob
kubectl describe cronjob daily-backup

# Get Jobs created by CronJob
kubectl get jobs -l cronjob=daily-backup

# View CronJob logs
kubectl logs -l cronjob=daily-backup

# Manually trigger CronJob
kubectl create job --from=cronjob/daily-backup manual-backup-$(date +%s)

# Suspend CronJob
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob daily-backup -p '{"spec":{"suspend":false}}'

# Delete CronJob
kubectl delete cronjob daily-backup
```

---

## 📊 Workload Comparison Table

| Aspect | Deployment | StatefulSet | DaemonSet | Job | CronJob |
|--------|-----------|------------|----------|-----|---------|
| **Use Case** | Stateless apps | Stateful apps | Node monitoring | One-time tasks | Scheduled tasks |
| **Pod Replicas** | User specified | User specified | One per node | One or many | Created by schedule |
| **Pod Identity** | Fungible | Ordered & stable | Unordered | Temporary | Temporary |
| **Storage** | Shared/none | Per-pod dedicated | Per-node | Temporary | Temporary |
| **Pod Restart** | Always | Always | Always | Never/OnFailure | Never/OnFailure |
| **Update Strategy** | RollingUpdate | RollingUpdate | RollingUpdate | N/A | N/A |

---

## 🎯 Quick Reference Commands

```bash
# StatefulSet
kubectl apply -f statefulset.yaml
kubectl get statefulsets
kubectl scale statefulset <n> --replicas=5
kubectl describe statefulset <n>

# DaemonSet
kubectl apply -f daemonset.yaml
kubectl get daemonsets
kubectl describe daemonset <n>
kubectl set image daemonset/<n> <container>=<image>

# Job
kubectl apply -f job.yaml
kubectl get jobs
kubectl describe job <n>
kubectl logs -l job-name=<n>
kubectl delete job <n> --cascade=background

# CronJob
kubectl apply -f cronjob.yaml
kubectl get cronjobs
kubectl create job --from=cronjob/<n> manual-<n>-$(date +%s)
kubectl patch cronjob <n> -p '{"spec":{"suspend":true}}'
```

---

## 🎯 Key Takeaways

- **StatefulSet** = Stateful apps needing identity & storage (databases)
- **DaemonSet** = Agents running on every node (logging, monitoring)
- **Job** = One-time batch tasks with guaranteed completion
- **CronJob** = Scheduled recurring jobs (backups, cleanups)
- Choose the right workload type for your use case!

**Next Phase**: Learn about Helm, Resource Management, and Probes!
