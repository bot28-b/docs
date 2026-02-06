# Kubernetes Phase 6: Helm & Package Management

## 📦 HELM BASICS

### What is Helm?
Helm is a package manager for Kubernetes. Automates installation, updates, and management of Kubernetes applications.

### Why Helm?
- **Templates**: Reduces repetitive YAML (DRY principle)
- **Packages**: Package entire applications (Charts)
- **Versioning**: Version control for deployments
- **Dependency Management**: Manage app dependencies
- **Easy Rollback**: Revert to previous versions
- **Reusability**: Share and reuse charts

### Helm Concepts

| Concept | Description |
|---------|-------------|
| **Chart** | Helm package (app blueprint) |
| **Release** | Running instance of chart |
| **Repository** | Collection of charts |
| **Values** | Configuration variables for chart |

### Install Helm
```bash
# On macOS
brew install helm

# On Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

### Add Helm Repository
```bash
# Add popular repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# List repositories
helm repo list

# Update repositories
helm repo update
```

### Search Helm Charts
```bash
# Search for chart
helm search repo nginx

# Search with keywords
helm search repo database

# Show chart versions
helm search repo nginx --versions

# Get chart details
helm show chart bitnami/mysql

# Show chart values
helm show values bitnami/mysql
```

### Install Chart
```bash
# Basic install
helm install my-release bitnami/mysql

# Install with namespace
helm install my-release bitnami/mysql -n dev --create-namespace

# Install specific version
helm install my-release bitnami/mysql --version 9.3.4

# Install from local chart
helm install my-release ./my-chart

# Install from URL
helm install my-release https://example.com/charts/mysql-1.0.0.tgz
```

### Override Values
```bash
# Override single value
helm install my-release bitnami/mysql --set primary.persistence.size=20Gi

# Override multiple values
helm install my-release bitnami/mysql \
  --set auth.rootPassword=secret \
  --set primary.persistence.size=20Gi

# Use values file
helm install my-release bitnami/mysql -f values.yaml

# Combine values files
helm install my-release bitnami/mysql -f values.yaml -f custom-values.yaml
```

### Helm List Commands
```bash
# List releases
helm list

# List releases in namespace
helm list -n dev

# List all releases (all namespaces)
helm list --all-namespaces

# Get release status
helm status my-release

# Get release values
helm get values my-release
```

### Upgrade Release
```bash
# Upgrade release
helm upgrade my-release bitnami/mysql

# Upgrade with new values
helm upgrade my-release bitnami/mysql --set primary.persistence.size=30Gi

# Upgrade and reuse values
helm upgrade my-release bitnami/mysql --reuse-values

# Dry run (preview changes)
helm upgrade my-release bitnami/mysql --dry-run --debug
```

### Rollback Release
```bash
# Rollback to previous release
helm rollback my-release

# Rollback to specific revision
helm rollback my-release 1

# View release history
helm history my-release

# Get detailed release info
helm history my-release --output json
```

### Uninstall Release
```bash
# Uninstall release
helm uninstall my-release

# Uninstall but keep release history
helm uninstall my-release --keep-history

# Uninstall from namespace
helm uninstall my-release -n dev
```

---

## 📝 CREATE CUSTOM HELM CHART

### Chart Directory Structure
```
my-app/
├── Chart.yaml                 # Chart metadata
├── values.yaml               # Default values
├── values-dev.yaml           # Dev environment values
├── values-prod.yaml          # Prod environment values
├── templates/
│   ├── deployment.yaml       # Deployment template
│   ├── service.yaml          # Service template
│   ├── configmap.yaml        # ConfigMap template
│   ├── secret.yaml           # Secret template
│   ├── ingress.yaml          # Ingress template
│   ├── _helpers.tpl          # Template helpers
│   └── NOTES.txt             # Post-install notes
├── charts/                   # Dependent charts
└── README.md                 # Documentation
```

### Create Chart
```bash
# Create new chart
helm create my-app

# Validate chart syntax
helm lint my-app

# Dry run to preview
helm install my-release ./my-app --dry-run --debug
```

### Chart.yaml
```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
type: application
version: 1.0.0              # Chart version
appVersion: "1.0"           # Application version
maintainers:
- name: Your Name
  email: your@email.com
keywords:
  - my-app
  - kubernetes
home: https://github.com/you/my-app
sources:
  - https://github.com/you/my-app
dependencies:
- name: redis
  version: "17.0.0"
  repository: https://charts.bitnami.com/bitnami
```

### values.yaml
```yaml
replicaCount: 3

image:
  repository: myregistry/my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

ingress:
  enabled: true
  hostname: myapp.example.com
  path: /

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  LOG_LEVEL: INFO
  APP_ENV: production
```

### Deployment Template (templates/deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        env:
        {{- range $key, $val := .Values.config }}
        - name: {{ $key }}
          value: {{ $val | quote }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service Template (templates/service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### Helpers Template (_helpers.tpl)
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

---

## 🔗 HELM DEPENDENCIES

### Define Dependencies
```yaml
# Chart.yaml
dependencies:
- name: redis
  version: "17.0.0"
  repository: https://charts.bitnami.com/bitnami
  alias: cache
- name: postgresql
  version: "12.0.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
  tags:
    - database
```

### Manage Dependencies
```bash
# Download dependencies
helm dependency update ./my-app

# List dependencies
helm dependency list ./my-app
```

### Use Dependency Values
```yaml
# values.yaml
redis:
  enabled: true
  auth:
    password: secret123

postgresql:
  enabled: true
  auth:
    password: dbpass
```

---

## 🎯 HELM BEST PRACTICES

✅ Use semantic versioning for charts
✅ Document chart values
✅ Provide default secure values
✅ Use templates for DRY code
✅ Test charts with `helm lint`
✅ Use `--dry-run` before installation
✅ Keep release history
✅ Use separate values files per environment
✅ Version your charts independently

---

## 📋 Helm Quick Reference

```bash
# Add repository
helm repo add <name> <url>
helm repo update

# Search
helm search repo <chart>
helm show values <chart>

# Install
helm install <release> <chart>
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --set key=value

# Manage
helm list
helm status <release>
helm get values <release>
helm history <release>

# Update
helm upgrade <release> <chart>
helm upgrade <release> <chart> --reuse-values

# Rollback
helm rollback <release>
helm rollback <release> <revision>

# Uninstall
helm uninstall <release>

# Create & Validate
helm create <name>
helm lint <chart>
helm template <release> <chart>
helm install <release> <chart> --dry-run --debug

# Package
helm package <chart>
helm push <chart.tgz> <registry>
```

---

## 🎯 Common Helm Patterns

### Install with Custom Environment
```bash
# Development
helm install my-app ./my-app -f values-dev.yaml -n dev --create-namespace

# Staging
helm install my-app ./my-app -f values-staging.yaml -n staging --create-namespace

# Production
helm install my-app ./my-app -f values-prod.yaml -n production --create-namespace
```

### Upgrade with Zero-Downtime
```bash
helm upgrade my-app ./my-app \
  --values values.yaml \
  --values custom-values.yaml \
  --reuse-values \
  --wait \
  --timeout 5m
```

### Rollback on Failed Upgrade
```bash
# If upgrade fails, automatically rollback
helm upgrade my-app ./my-app \
  --atomic \
  --timeout 5m
```

---

## 🎯 Key Takeaways

- **Helm Charts** = Packaged, templated Kubernetes applications
- **Values** = Configuration variables for customization
- **Templates** = Dynamic YAML files using Go templating
- **Repositories** = Package registries for sharing charts
- **Releases** = Running instances of charts
- Use Helm for repeatable, versionable deployments

**Next Phase**: Learn about Monitoring, Logging, and Observability!
