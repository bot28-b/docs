# Kubernetes Phase 7: Monitoring, Logging & Production

## 📊 MONITORING WITH PROMETHEUS

### What is Prometheus?
Open-source monitoring and alerting system. Collects metrics from Kubernetes and applications.

### Prometheus Architecture
```
Applications/Pods
        ↓
    Prometheus (scrapes metrics)
        ↓
   Time-Series Database
        ↓
    Alert Manager (sends alerts)
        ↓
    Grafana (visualizes data)
```

### Install Prometheus with Helm
```bash
# Add Prometheus community charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack (Prometheus + Grafana + AlertManager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

### Create ServiceMonitor (Prometheus discovers scrape targets)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
  namespace: dev
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
```

### Prometheus Scrape Config
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
spec:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
```

### Expose Metrics from Application
```bash
# Application must expose metrics on /metrics endpoint
# Example using Python (Flask):
from prometheus_client import Counter, Histogram, generate_latest

request_count = Counter('app_requests_total', 'Total requests')
request_duration = Histogram('app_request_duration_seconds', 'Request duration')

@app.route('/metrics')
def metrics():
    return generate_latest()
```

### Common Prometheus Queries (PromQL)
```promql
# CPU usage
container_cpu_usage_seconds_total

# Memory usage
container_memory_usage_bytes

# Requests per second
rate(http_requests_total[5m])

# Error rate
rate(http_errors_total[5m])

# Pod restarts
kube_pod_container_status_restarts_total

# Request latency (95th percentile)
histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

### Prometheus Alerting Rules
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
spec:
  groups:
  - name: app.rules
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: rate(http_errors_total[5m]) > 0.05
      for: 5m
      annotations:
        summary: "High error rate detected"
    
    - alert: PodRestartingTooOften
      expr: kube_pod_container_status_restarts_total > 5
      for: 10m
      annotations:
        summary: "Pod restarting frequently"
    
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
      for: 5m
      annotations:
        summary: "Pod using > 90% memory"
```

---

## 📈 GRAFANA DASHBOARDS

### Access Grafana
```bash
# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Port forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access at http://localhost:3000
# Username: admin
# Password: (from above)
```

### Add Prometheus Data Source
1. Settings → Data Sources → Add Prometheus
2. URL: `http://prometheus:9090`
3. Save & Test

### Create Dashboard
1. New Dashboard → New Panel
2. Select Prometheus as data source
3. Write PromQL query
4. Visualize metrics

### Popular Grafana Dashboards
- Kubernetes Cluster Monitoring (ID: 8588)
- Node Exporter (ID: 1860)
- Application Metrics (ID: 6670)

---

## 🔍 LOGGING WITH ELK STACK

### What is ELK?
- **Elasticsearch**: Searchable log storage
- **Logstash**: Log processing pipeline
- **Kibana**: Log visualization

### Install ELK with Helm
```bash
# Add Elastic charts
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch -n logging --create-namespace

# Install Kibana
helm install kibana elastic/kibana -n logging

# Install Logstash (or use Fluentd)
helm install logstash elastic/logstash -n logging
```

### Alternatively: Use Fluentd for Logs
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    <match kubernetes.**>
      @type elasticsearch
      @id output_elasticsearch
      @log_level info
      include_tag_key true
      host elasticsearch.logging.svc.cluster.local
      port 9200
      path_prefix logstash
      logstash_format true
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_interval 5s
        retry_timeout 72h
      </buffer>
    </match>
```

### Deploy Fluentd as DaemonSet
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
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluentd-config
```

### Access Kibana
```bash
# Port forward
kubectl port-forward -n logging svc/kibana-kibana 5601:5601

# Access at http://localhost:5601
```

### Create Kibana Index Pattern
1. Kibana → Stack Management → Index Patterns
2. Create Index Pattern: `logstash-*`
3. Select `@timestamp` as time field

### Query Logs in Kibana
```
# Search specific pod
kubernetes.pod_name: "my-app-abc123"

# Search specific container
kubernetes.container_name: "app"

# Search specific namespace
kubernetes.namespace_name: "production"

# Search log level
level: "ERROR"

# Combined query
kubernetes.namespace_name: "production" AND level: "ERROR"
```

---

## 📞 ALERTMANAGER

### Configure AlertManager
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    
    route:
      receiver: default
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      routes:
      - receiver: critical
        match:
          severity: critical
        repeat_interval: 1h
    
    receivers:
    - name: default
      slack_configs:
      - api_url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
        channel: '#alerts'
    
    - name: critical
      slack_configs:
      - api_url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
        channel: '#critical-alerts'
      email_configs:
      - to: oncall@example.com
        from: alerts@example.com
        smarthost: smtp.gmail.com:587
        auth_username: your-email@gmail.com
        auth_password: your-password
```

### Send Alerts to Slack
```yaml
slack_configs:
- api_url: YOUR_SLACK_WEBHOOK_URL
  channel: '#kubernetes-alerts'
  title: '{{ .GroupLabels.alertname }}'
  text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

### Send Alerts to PagerDuty
```yaml
pagerduty_configs:
- service_key: YOUR_PAGERDUTY_KEY
  description: '{{ .GroupLabels.alertname }}'
  details:
    firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
```

---

## 🔍 DEBUGGING & TROUBLESHOOTING

### View Pod Logs
```bash
# View pod logs
kubectl logs <pod-name>

# View logs with follow
kubectl logs <pod-name> -f

# View logs from all containers in pod
kubectl logs <pod-name> --all-containers=true

# View logs from specific container
kubectl logs <pod-name> -c <container-name>

# View logs from previous container (if restarted)
kubectl logs <pod-name> --previous

# View logs from last hour
kubectl logs <pod-name> --since=1h

# View last 100 lines
kubectl logs <pod-name> --tail=100
```

### Describe Resources
```bash
# Describe pod (events, resource usage, etc.)
kubectl describe pod <pod-name>

# Describe deployment
kubectl describe deployment <deployment-name>

# Describe service
kubectl describe service <service-name>

# Describe node
kubectl describe node <node-name>
```

### Exec into Pod
```bash
# Execute command
kubectl exec -it <pod-name> -- /bin/bash

# Execute command in specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Run single command
kubectl exec <pod-name> -- ls -la /app
```

### Port Forwarding
```bash
# Forward local port to pod
kubectl port-forward <pod-name> 8080:8080

# Forward to service
kubectl port-forward svc/<service-name> 5432:5432

# Forward with specific address
kubectl port-forward --address 0.0.0.0 <pod-name> 8080:8080
```

### Check Events
```bash
# View cluster events
kubectl get events

# View events in namespace
kubectl get events -n <namespace>

# Watch events
kubectl get events -w

# Describe events
kubectl describe event <event-name>
```

---

## 🏆 PRODUCTION BEST PRACTICES

### Deployment Checklist
✅ Set resource requests & limits
✅ Configure liveness & readiness probes
✅ Use readiness gates for graceful startup
✅ Enable pod disruption budgets
✅ Use network policies
✅ Configure RBAC properly
✅ Enable pod security policies
✅ Use secrets for sensitive data
✅ Implement monitoring & alerting
✅ Set up centralized logging
✅ Use Helm for deployment
✅ Version everything in Git
✅ Test rollbacks
✅ Document runbooks
✅ Implement autoscaling

### Pod Disruption Budget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

### Readiness Gates
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  readinessGates:
  - conditionType: "www.example.com/custom-condition"
  containers:
  - name: app
    image: myapp:latest
```

### Pod Security Policy
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  readOnlyRootFilesystem: true
  requiredDropCapabilities:
  - ALL
```

---

## 📋 Common Production Issues & Solutions

### Pod Stuck in Pending
```bash
# Check if node has resources
kubectl describe node <node>

# Check for insufficient memory/CPU
kubectl top nodes
```

### Pod CrashLoopBackOff
```bash
# View logs
kubectl logs <pod> --previous

# Check liveness probe config
kubectl describe pod <pod>
```

### Service Endpoints Not Ready
```bash
# Check service endpoints
kubectl get endpoints <service>

# Check pod readiness probe
kubectl get pods -o wide

# Check selector matches
kubectl describe service <service>
```

### DNS Issues
```bash
# Test DNS from pod
kubectl exec <pod> -- nslookup kubernetes.default

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## 📋 Quick Reference

```bash
# Monitoring
kubectl top nodes
kubectl top pods
kubectl get events

# Logging
kubectl logs <pod>
kubectl logs <pod> -f
kubectl logs <pod> --previous

# Debugging
kubectl describe pod <pod>
kubectl exec -it <pod> -- /bin/bash
kubectl port-forward <pod> 8080:8080

# Get resources
kubectl get all
kubectl get all -n <namespace>
kubectl get all --all-namespaces
```

---

## 🎯 Key Takeaways

- **Prometheus** = Metrics collection & alerting
- **Grafana** = Metrics visualization
- **ELK/Fluentd** = Log aggregation & analysis
- **AlertManager** = Alert routing & notifications
- Always monitor and log in production
- Implement alerting for critical issues
- Keep detailed runbooks for common issues
- Test backup & disaster recovery procedures

**Congratulations! You've completed all 7 phases of Kubernetes learning!**

---

## 🚀 Next Steps

1. **Hands-On Practice**: Deploy real applications to Kubernetes
2. **Advanced Topics**: Istio service mesh, Knative serverless
3. **Cloud Platforms**: EKS, AKS, GKE specific features
4. **GitOps**: ArgoCD for continuous deployment
5. **Cost Optimization**: Kubecost for cluster monitoring
6. **Security**: Falco for runtime security
7. **Infrastructure as Code**: Terraform for cluster provisioning

---

**Happy Kubernetes Journey! 🚀**
