# Day 07 — Monitoring with Prometheus & Grafana 📊

## The Problem First

Your app is running in Kubernetes. But how do you know:
- Is CPU usage too high?
- Is a pod about to crash because it's running out of memory?
- How many requests per second is my app handling?
- Why did my app slow down at 3am?

**Without monitoring, you're flying blind.** Prometheus and Grafana fix this.

---

## The Big Picture — How It All Connects

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
│                                                              │
│   ┌──────────────┐     ┌──────────────┐   ┌─────────────┐  │
│   │  API Server  │     │   Kubelet    │   │  Your App   │  │
│   │  /metrics    │     │  /metrics    │   │  /metrics   │  │
│   └──────┬───────┘     └──────┬───────┘   └──────┬──────┘  │
│          │                    │                   │         │
│          └──────────┬─────────┘                   │         │
│                     │  SCRAPE (pull metrics)       │         │
│                     ▼          every 15 seconds    │         │
│            ┌────────────────┐◄────────────────────┘         │
│            │   PROMETHEUS   │                               │
│            │                │  stores metrics as            │
│            │  Time-series   │  time-series data             │
│            │  Database      │                               │
│            └────────┬───────┘                               │
│                     │  PromQL queries                       │
│                     ▼                                       │
│            ┌────────────────┐                               │
│            │    GRAFANA     │  → Beautiful dashboards 📊    │
│            │                │                               │
│            │  Visualizes    │                               │
│            │  the data      │                               │
│            └────────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

---

## How Prometheus Works — Step by Step

### Step 1: Every component exposes a `/metrics` endpoint

The API Server, Kubelet, and your own app all expose a URL like:
```
http://kube-apiserver:6443/metrics
http://node-ip:10255/metrics
http://your-app:8080/metrics
```

When you hit that URL, you get raw text data (not JSON!) like this:
```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", status="200"} 1234
http_requests_total{method="POST", status="500"} 7

# HELP node_cpu_seconds_total CPU time spent
node_cpu_seconds_total{cpu="0", mode="idle"} 5432.1
```

> 💡 This is called **Prometheus exposition format** — plain text, one metric per line.
> It's NOT JSON. It's simpler than JSON on purpose (lightweight and fast to parse).

### Step 2: Prometheus SCRAPES (pulls) these endpoints

Prometheus goes to each `/metrics` URL every **15 seconds** (configurable) and stores what it finds. This is called **scraping**.

```
Prometheus: "Hey API Server, give me your metrics"
API Server: "Here you go: 423 pods running, CPU at 34%..."
Prometheus: Stores with timestamp → 14:03:15 → cpu=34%
                                  → 14:03:30 → cpu=36%
                                  → 14:03:45 → cpu=35%
```

This is a **time-series database** — every metric has a value AND a timestamp.

### Step 3: Grafana queries Prometheus and shows dashboards

Grafana connects to Prometheus and asks questions using **PromQL** (Prometheus Query Language):

```promql
# Average CPU usage over last 5 minutes
rate(node_cpu_seconds_total[5m])

# Memory usage of all pods
container_memory_usage_bytes{namespace="default"}

# HTTP error rate
rate(http_requests_total{status=~"5.."}[5m])
```

Grafana turns these numbers into beautiful line charts, graphs, and alerts.

---

## Key Concepts

| Term | What it means |
|------|--------------|
| **Scraping** | Prometheus pulling metrics from an endpoint |
| **Exporter** | A service that exposes metrics in Prometheus format |
| **Time-series** | Data stored with timestamps (value at each point in time) |
| **PromQL** | Query language to ask questions about your metrics |
| **Alertmanager** | Sends alerts (Slack, email) when metrics cross a threshold |
| **ServiceMonitor** | A K8s resource that tells Prometheus what to scrape |

---

## What Metrics Look Like (Prometheus Format vs JSON)

### Prometheus Format (what `/metrics` returns):
```
http_requests_total{method="GET",status="200"} 1234 1700000000
```
Structure: `metric_name{labels} value timestamp`

### The same data in JSON (what Grafana gets back from PromQL API):
```json
{
  "metric": {
    "__name__": "http_requests_total",
    "method": "GET",
    "status": "200"
  },
  "values": [
    [1700000000, "1234"],
    [1700000015, "1289"],
    [1700000030, "1301"]
  ]
}
```

> Prometheus stores as its own format internally. When Grafana queries it via HTTP API, it gets JSON back. So the flow is:
> `/metrics` (plain text) → Prometheus stores it → Grafana queries via API (JSON) → Dashboard

---

## 📁 YAML Files

### `namespace.yml` — Separate namespace for monitoring tools
```yaml
# Keep monitoring tools separate from your app
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

### `prometheus-configmap.yml` — Tell Prometheus what to scrape
```yaml
# This is the "brain" of Prometheus — it tells it where to find metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s        # Pull metrics every 15 seconds
      evaluation_interval: 15s    # Check alert rules every 15 seconds

    scrape_configs:

      # Scrape Prometheus itself
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      # Scrape the Kubernetes API Server
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name]
            action: keep
            regex: default;kubernetes

      # Scrape all Kubernetes Nodes (kubelet metrics)
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

      # Scrape all pods that have the annotation:
      # prometheus.io/scrape: "true"
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
```

### `prometheus-deployment.yml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus   # Needs permission to access K8s API
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
          - "--config.file=/etc/prometheus/prometheus.yml"   # Use our ConfigMap
          - "--storage.tsdb.path=/prometheus"                # Where to store data
          - "--storage.tsdb.retention.time=7d"               # Keep 7 days of data
          - "--web.enable-lifecycle"                         # Allow hot reload
        ports:
        - containerPort: 9090           # Prometheus web UI port
        volumeMounts:
        - name: prometheus-config       # Mount the ConfigMap as a file
          mountPath: /etc/prometheus
        - name: prometheus-storage      # Storage for metrics data
          mountPath: /prometheus
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: prometheus-config
        configMap:
          name: prometheus-config       # The ConfigMap we created above
      - name: prometheus-storage
        emptyDir: {}                    # For learning; use PersistentVolume in prod
```

### `prometheus-service.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
spec:
  type: NodePort                        # Access from browser
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090                     # Access at <minikube-ip>:30090
```

### `prometheus-rbac.yml` — Give Prometheus permission to read K8s metrics
```yaml
# Prometheus needs to read K8s cluster info to discover what to scrape
# RBAC = Role Based Access Control
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]       # Read-only access
- apiGroups: ["extensions"]
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]                        # Access the /metrics endpoints
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

### `grafana-deployment.yml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000           # Grafana web UI port
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: "admin"                # Default login username
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin123"             # Change this in production!
        - name: GF_USERS_ALLOW_SIGN_UP
          value: "false"               # Don't allow random signups
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana   # Where Grafana stores dashboards
      volumes:
      - name: grafana-storage
        emptyDir: {}                    # For learning; use PersistentVolume in prod
```

### `grafana-service.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30030                     # Access at <minikube-ip>:30030
```

---

## 🏃 Hands-On — Deploy the Full Monitoring Stack

```bash
# Step 1: Apply everything in order
kubectl apply -f namespace.yml
kubectl apply -f prometheus-rbac.yml
kubectl apply -f prometheus-configmap.yml
kubectl apply -f prometheus-deployment.yml
kubectl apply -f prometheus-service.yml
kubectl apply -f grafana-deployment.yml
kubectl apply -f grafana-service.yml

# OR apply all at once:
kubectl apply -f .

# Step 2: Watch them start up
kubectl get pods -n monitoring -w
# Wait until both show Running

# Step 3: Get the URLs (with Minikube)
minikube service prometheus-service -n monitoring --url
minikube service grafana-service -n monitoring --url
```

### Access Prometheus UI
Open the Prometheus URL in your browser (port 30090).

```
Go to: Status → Targets
You should see all the things Prometheus is scraping ✅
```

Try a PromQL query in the search box:
```promql
# See all running containers
count(kube_pod_status_phase{phase="Running"})

# CPU usage
rate(node_cpu_seconds_total{mode="user"}[5m])
```

### Connect Grafana to Prometheus

1. Open Grafana URL (port 30030)
2. Login: `admin` / `admin123`
3. Go to **Configuration → Data Sources → Add data source**
4. Choose **Prometheus**
5. Set URL to: `http://prometheus-service.monitoring.svc.cluster.local:9090`
   - This is the internal DNS name of the Prometheus Service inside the cluster
6. Click **Save & Test** → should say "Data source is working" ✅

### Import a Pre-built Dashboard

Instead of building dashboards from scratch, use community ones:
1. Go to **Dashboards → Import**
2. Enter dashboard ID: `3119` (Kubernetes cluster overview)
3. Select your Prometheus data source
4. Click **Import**

You'll instantly get CPU, memory, pod count graphs! 🎉

---

## 🔍 Debugging Commands

```bash
# See pods in monitoring namespace
kubectl get pods -n monitoring

# Describe prometheus pod if it's not starting
kubectl describe pod -n monitoring -l app=prometheus

# See prometheus logs
kubectl logs -n monitoring -l app=prometheus

# See grafana logs
kubectl logs -n monitoring -l app=grafana

# Check if prometheus can reach the API server
kubectl exec -n monitoring -it <prometheus-pod> -- \
  wget -qO- http://localhost:9090/api/v1/targets | head -50

# Port-forward instead of NodePort (alternative access method)
kubectl port-forward -n monitoring svc/prometheus-service 9090:9090
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
# Then open http://localhost:9090 and http://localhost:3000
```

---

## Make Your Own App Scrapeable

Add this annotation to your pod/deployment and Prometheus will auto-discover it:

```yaml
# In your app's deployment template:
metadata:
  annotations:
    prometheus.io/scrape: "true"    # Tell Prometheus to scrape this pod
    prometheus.io/port: "8080"      # Port where /metrics is exposed
    prometheus.io/path: "/metrics"  # Path (default is /metrics)
```

Your app also needs to expose a `/metrics` endpoint. In Python:
```python
from prometheus_client import start_http_server, Counter

requests_total = Counter('http_requests_total', 'Total requests')

# In your request handler:
requests_total.inc()
```

---

## 📝 Summary

```
App / K8s Components
        ↓
   expose /metrics          (plain text format)
        ↓
    Prometheus              (scrapes every 15s, stores time-series)
        ↓
   PromQL queries           (Grafana asks questions)
        ↓
     Grafana                (shows dashboards and sends alerts)
```

| Tool | Job |
|------|-----|
| **Prometheus** | Collects and stores metrics |
| **Grafana** | Visualizes metrics as dashboards |
| **Alertmanager** | Sends alerts when something breaks |
| **Exporters** | Bridge between apps and Prometheus format |

**You've now completed the full 8-day Kubernetes + Monitoring journey! 🎉**

### Suggested YAML rename for your repo:
Rename the original `day-07-real-project` → `day-08-real-project` and add this as `day-07-monitoring`.
