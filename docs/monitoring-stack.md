# 📊 Monitoring Stack

> Complete observability implementation for wetfish web-services using FishVision monitoring model.

---

## 🎯 Monitoring Architecture Overview

### **FishVision-Based Observability Stack**
```
┌─────────────────────────────────────────────────────────────────┐
│                    wetfish-monitoring                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Grafana    │  │ Prometheus   │  │  AlertManager│     │
│  │ Visualization│  │   Metrics    │  │   Alerts     │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                  │             │
│  ┌──────▼──────┐  ┌───────▼───────┐  ┌──────▼───────┐     │
│  │    Loki     │  │    Tempo     │  │   IRC Relay  │     │
│  │    Logs     │  │   Traces     │  │ Webhook      │     │
│  └─────────────┘  └─────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### **Data Flow**
```
Applications → Metrics → Prometheus → Grafana Dashboards
            ↓           ↓           ↓
          Logs → Loki → Log Queries → Grafana
            ↓           ↓           ↓
        Traces → Tempo → Trace Analysis → Grafana
            ↓                       ↓
        Prometheus → AlertManager → IRC/Webhook → wetfish
```

---

## 🛠️ Component Configuration

### **1. Prometheus Setup**

#### **Configuration Structure**
```yaml
monitoring/prometheus/
├── prometheus.yaml          # Main configuration
├── alertmanager.yaml       # Alert routing rules
├── rules/                  # Alert rule definitions
│   ├── cluster.yaml
│   ├── applications.yaml
│   └── database.yaml
└── servicemonitors/        # Kubernetes service monitors
    ├── wiki.yaml
    ├── forum.yaml
    └── infrastructure.yaml
```

#### **Prometheus Configuration**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'wetfish-dev'
    environment: 'development'

rule_files:
  - "/etc/prometheus/rules/*.yaml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

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

#### **ServiceMonitors**
```yaml
# monitoring/servicemonitors/wiki.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: wiki-metrics
  namespace: wetfish-dev
  labels:
    app: wiki
    monitoring: prometheus
spec:
  selector:
    matchLabels:
      app: wiki
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
      honorLabels: true
```

### **2. Grafana Configuration**

#### **Dashboard Structure**
```yaml
monitoring/grafana/
├── dashboards/
│   ├── cluster-overview.json
│   ├── wiki-service.json
│   ├── database-performance.json
│   ├── resource-usage.json
│   └── alert-status.json
├── provisioning/
│   ├── datasources/
│   │   ├── prometheus.yaml
│   │   ├── loki.yaml
│   │   └── tempo.yaml
│   └── dashboards/
│       └── dashboard.yml
└── grafana.ini
```

#### **Key Dashboards**

**1. Cluster Overview Dashboard**
```json
{
  "dashboard": {
    "title": "Wetfish Cluster Overview",
    "panels": [
      {
        "title": "Node Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"kubernetes-nodes\"}",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total[5m]) * 100",
            "legendFormat": "{{pod}}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "container_memory_usage_bytes / 1024 / 1024",
            "legendFormat": "{{pod}}"
          }
        ]
      }
    ]
  }
}
```

**2. Wiki Service Dashboard**
```json
{
  "dashboard": {
    "title": "Wiki Service Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Database Connections",
        "type": "stat",
        "targets": [
          {
            "expr": "mysql_global_status_threads_connected",
            "legendFormat": "Active Connections"
          }
        ]
      }
    ]
  }
}
```

### **3. Loki Log Aggregation**

#### **Configuration**
```yaml
# monitoring/loki/loki.yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h
  max_chunk_age: 1h
  chunk_target_size: 1048576
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
```

#### **Fluent Bit Configuration**
```yaml
# monitoring/fluent-bit/fluent-bit.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: wetfish-monitoring
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-loki.conf

  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

  output-loki.conf: |
    [OUTPUT]
        Name  loki
        Match *
        Url   http://loki:3100/loki/api/v1/push
        Labels job=fluent-bit
```

### **4. Tempo Distributed Tracing**

#### **Configuration**
```yaml
# monitoring/tempo/tempo.yaml
server:
  http_listen_port: 3100

distributor:
  receivers:
    otlp:
      protocols:
        http:
          endpoint: "0.0.0.0:4318"
        grpc:
          endpoint: "0.0.0.0:4317"

ingester:
  max_block_bytes: 1_000_000
  max_block_duration: 5m

compactor:
  compaction:
    compaction_window: 1h

metrics_generator:
  processors:
    - service-graphs
    - span-metrics
  storage:
    path: /tmp/tempo/generator
    remote_write:
      - url: http://prometheus:9090/api/v1/write

storage:
  trace:
    backend: local
    block:
      v2:
        bloom_filter_false_positive: .05
        index_downsample_bytes: 1000
        size_downsample_bytes: 1000
    local:
      path: /tmp/tempo/blocks
```

---

## 🚨 Alert Management

### **Alert Rules**

#### **Cluster Alerts**
```yaml
# monitoring/rules/cluster.yaml
groups:
  - name: cluster.rules
    rules:
      - alert: NodeDown
        expr: up{job="kubernetes-nodes"} == 0
        for: 2m
        labels:
          severity: critical
          service: cluster
        annotations:
          summary: "Node {{ $labels.instance }} is down"
          description: "Node {{ $labels.instance }} has been down for more than 2 minutes"

      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total[5m]) * 100 > 80
        for: 5m
        labels:
          severity: warning
          service: cluster
        annotations:
          summary: "High CPU usage on {{ $labels.pod }}"
          description: "CPU usage is above 80% for pod {{ $labels.pod }}"

      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes / container_spec_memory_limit_bytes * 100 > 80
        for: 5m
        labels:
          severity: warning
          service: cluster
        annotations:
          summary: "High memory usage on {{ $labels.pod }}"
          description: "Memory usage is above 80% for pod {{ $labels.pod }}"
```

#### **Application Alerts**
```yaml
# monitoring/rules/applications.yaml
groups:
  - name: application.rules
    rules:
      - alert: WikiServiceDown
        expr: up{job="wiki"} == 0
        for: 1m
        labels:
          severity: critical
          service: wiki
        annotations:
          summary: "Wiki service is down"
          description: "Wiki service has been down for more than 1 minute"

      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 3m
        labels:
          severity: critical
          service: application
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 10% for the last 3 minutes"

      - alert: DatabaseConnectionFailure
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
          service: database
        annotations:
          summary: "Database connection failure"
          description: "Cannot connect to MySQL database"
```

### **AlertManager Configuration**
```yaml
# monitoring/alertmanager.yaml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@wetfish.net'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://irc-relay:8080/webhook'
        send_resolved: true

  - name: 'critical-alerts'
    webhook_configs:
      - url: 'http://irc-relay:8080/critical'
        send_resolved: true

  - name: 'warning-alerts'
    webhook_configs:
      - url: 'http://irc-relay:8080/warning'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

---

## 📊 Data Collection

### **Application Metrics**

#### **Wiki Service Metrics**
```go
// Go example for wiki service metrics
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "wiki_http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "wiki_http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
        },
        []string{"method", "path"},
    )

    wikiPageViews = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "wiki_page_views_total",
            Help: "Total number of page views",
        },
        []string{"page"},
    )

    mysqlConnections = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "wiki_mysql_connections_active",
            Help: "Number of active MySQL connections",
        },
        []string{"host"},
    )
)

func RecordHTTPRequest(method, status string, duration float64) {
    httpRequestsTotal.WithLabelValues(method, status).Inc()
    httpRequestDuration.WithLabelValues(method, "").Observe(duration)
}

func RecordPageView(page string) {
    wikiPageViews.WithLabelValues(page).Inc()
}
```

#### **PHP Application Metrics**
```php
<?php
// PHP metrics endpoint
require 'vendor/autoload.php';

use Prometheus\CollectorRegistry;
use Prometheus\RenderTextFormat;

$registry = new CollectorRegistry();

// Request counter
$requestCounter = $registry->getOrRegisterCounter(
    'wiki_requests_total',
    'Total wiki requests',
    ['method', 'status']
);

// Request duration histogram
$durationHistogram = $registry->getOrRegisterHistogram(
    'wiki_request_duration_seconds',
    'Request duration in seconds',
    ['method', 'endpoint']
);

// Record metrics
$requestCounter->incBy(1, [$_SERVER['REQUEST_METHOD'], http_response_code()]);
$durationHistogram->observe($duration, [$_SERVER['REQUEST_METHOD'], $_SERVER['PATH_INFO']]);

// Expose metrics endpoint
header('Content-Type: ' . RenderTextFormat::MIME_TYPE);
echo $registry->getMetricFamilySamples();
?>
```

### **Database Metrics**

#### **MySQL Exporter Configuration**
```yaml
# monitoring/mysql-exporter.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-exporter
  namespace: wetfish-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-exporter
  template:
    metadata:
      labels:
        app: mysql-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9104"
    spec:
      containers:
        - name: mysql-exporter
          image: prom/mysqld-exporter:latest
          env:
            - name: DATA_SOURCE_NAME
              value: "user:password@(wiki-db-service:3306)/"
          ports:
            - containerPort: 9104
```

---

## 🔧 Deployment Instructions

### **1. Install Monitoring Stack**
```bash
#!/bin/bash
# scripts/setup-monitoring.sh

set -euo pipefail

NAMESPACE="wetfish-monitoring"

echo "Setting up monitoring stack..."

# Create namespace
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Deploy Prometheus Stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace $NAMESPACE \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=admin \
  --create-namespace

# Deploy Loki
helm install loki grafana/loki \
  --namespace $NAMESPACE \
  --set loki.auth_enabled=false

# Deploy Tempo
helm install tempo grafana/tempo \
  --namespace $NAMESPACE

echo "Monitoring stack deployed successfully!"
```

### **2. Configure Dashboards**
```bash
#!/bin/bash
# scripts/import-dashboards.sh

NAMESPACE="wetfish-monitoring"

echo "Importing Grafana dashboards..."

# Import dashboards
kubectl apply -f monitoring/grafana/dashboards/

# Configure datasources
kubectl apply -f monitoring/grafana/provisioning/datasources/

# Restart Grafana to pick up changes
kubectl rollout restart deployment/grafana -n $NAMESPACE

echo "Dashboards imported successfully!"
```

### **3. Setup Alerting**
```bash
#!/bin/bash
# scripts/setup-alerting.sh

NAMESPACE="wetfish-monitoring"

echo "Setting up alerting..."

# Deploy alert rules
kubectl apply -f monitoring/rules/

# Deploy AlertManager configuration
kubectl create configmap alertmanager-config \
  --from-file=monitoring/alertmanager.yaml \
  --namespace $NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart Prometheus to pick up alert rules
kubectl rollout restart deployment/prometheus -n $NAMESPACE

echo "Alerting configured successfully!"
```

---

## 📈 Performance Optimization

### **Resource Allocation**
```yaml
# Recommended resource limits for monitoring stack
monitoring:
  prometheus:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  
  grafana:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
  
  loki:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
  
  tempo:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
```

### **Retention Policies**
```yaml
# Data retention configuration
prometheus:
  retention: 15d
  storageSpec:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 50Gi

loki:
  retention:
    enabled: true
    days: 7

tempo:
  retention:
    enabled: true
    days: 7
```

---

## 🧪 Testing and Validation

### **Monitoring Health Checks**
```bash
#!/bin/bash
# scripts/test-monitoring.sh

NAMESPACE="wetfish-monitoring"

echo "Testing monitoring stack..."

# Check Prometheus
curl -f http://localhost:9090/api/v1/targets || echo "Prometheus not accessible"

# Check Grafana
curl -f http://localhost:3000/api/health || echo "Grafana not accessible"

# Check Loki
curl -f http://localhost:3100/ready || echo "Loki not accessible"

# Check Tempo
curl -f http://localhost:3100/ready || echo "Tempo not accessible"

echo "Health checks completed!"
```

### **Alert Testing**
```bash
#!/bin/bash
# scripts/test-alerts.sh

echo "Testing alert rules..."

# Trigger a test alert
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning",
      "service": "monitoring"
    },
    "annotations": {
      "description": "This is a test alert"
    }
  }]'

echo "Test alert sent!"
```

---

## 🔍 Troubleshooting

### **Common Issues**

#### **Prometheus Not Scraping**
```bash
# Check ServiceMonitor configuration
kubectl get servicemonitors -n wetfish-dev

# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Check pod labels
kubectl get pods -n wetfish-dev --show-labels
```

#### **Grafana Not Showing Data**
```bash
# Check data source configuration
curl http://localhost:3000/api/datasources

# Check dashboard import
curl http://localhost:3000/api/dashboards

# Check Prometheus connection
curl http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up
```

#### **Logs Not Appearing in Loki**
```bash
# Check Fluent Bit logs
kubectl logs -l app=fluent-bit -n wetfish-monitoring

# Check Loki ingestion
curl http://localhost:3100/loki/api/v1/labels

# Test log query
curl -G -s http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={job="fluent-bit"}' \
  --data-urlencode 'start=2024-01-01T00:00:00Z' \
  --data-urlencode 'end=2024-01-01T01:00:00Z'
```

---

*Monitoring Stack v1.0 - Last Updated: $(date)*