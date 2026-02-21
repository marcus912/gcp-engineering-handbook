# Chapter 17: Kubernetes Monitoring & Observability

> **Prerequisites**: Chapter 12 (Containers & Kubernetes), Chapter 13 (Troubleshooting Methodology)

## 16.1 The Kubernetes Observability Stack

### Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                 Kubernetes Observability                          │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   METRICS   │  │    LOGS     │  │   TRACES    │             │
│  │             │  │             │  │             │             │
│  │ Prometheus  │  │ Fluent Bit  │  │ OpenTelemetry│            │
│  │ Grafana     │  │ Loki        │  │ Jaeger      │             │
│  │ kube-state  │  │ EFK Stack   │  │ Tempo       │             │
│  │ metrics     │  │             │  │             │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│              ┌───────────┴───────────┐                          │
│              │    Correlation Layer   │                          │
│              │  (TraceID, Labels,     │                          │
│              │   Timestamps)          │                          │
│              └───────────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

Why Kubernetes needs specialized monitoring:
- **Ephemeral pods**: Containers are created and destroyed constantly
- **Dynamic scheduling**: Workloads move between nodes
- **Multi-layer stack**: Application → container → pod → node → cluster
- **Service discovery**: Endpoints change as pods scale

### Observability Maturity Model

| Level | Metrics | Logs | Traces | Alerting |
|-------|---------|------|--------|----------|
| **L0** | kubectl top | kubectl logs | None | None |
| **L1** | Metrics Server | Node-level logs | None | Basic alerts |
| **L2** | Prometheus + Grafana | Centralized logging | Request tracing | Symptom-based |
| **L3** | Custom metrics + SLOs | Structured + correlated | Distributed tracing | SLO-based |
| **L4** | Full observability | ML-based anomaly detection | Auto-instrumentation | Predictive |

---

## 16.2 Metrics Server & Resource Metrics Pipeline

### Architecture

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  Node 1  │    │  Node 2  │    │  Node 3  │
│ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │
│ │kubelet│ │    │ │kubelet│ │    │ │kubelet│ │
│ │  +    │ │    │ │  +    │ │    │ │  +    │ │
│ │cAdvisor│ │   │ │cAdvisor│ │   │ │cAdvisor│ │
│ └───┬───┘ │    │ └───┬───┘ │    │ └───┬───┘ │
└─────┼─────┘    └─────┼─────┘    └─────┼─────┘
      │                │                │
      └────────────────┼────────────────┘
                       │
              ┌────────┴────────┐
              │  Metrics Server  │
              │  (in-memory)     │
              └────────┬────────┘
                       │
              ┌────────┴────────┐
              │  metrics.k8s.io  │
              │  API             │
              └─────────────────┘
```

### Using kubectl top

```bash
# Node resource usage
kubectl top nodes
# NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1       450m         22%    3200Mi          41%
# node-2       890m         44%    5100Mi          65%
# node-3       120m         6%     1800Mi          23%

# Pod resource usage
kubectl top pods -n production
kubectl top pods --sort-by=memory -A | head -20
kubectl top pods --sort-by=cpu -A | head -20

# Container-level usage
kubectl top pods --containers -n production
```

### Metrics Pipeline Comparison

| Feature | Metrics Server | cAdvisor | kube-state-metrics |
|---------|---------------|----------|-------------------|
| **Data source** | kubelet summary API | Container runtime | Kubernetes API |
| **Metrics type** | CPU/memory current | Container resource stats | Object state |
| **Historical data** | No (point-in-time) | No | No |
| **Used by** | kubectl top, HPA | Prometheus scraping | Prometheus scraping |
| **Install** | Required for HPA | Built into kubelet | Separate deployment |

---

## 16.3 Prometheus for Kubernetes

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Prometheus Stack                               │
│                                                                  │
│  ┌─────────────┐     ┌──────────────────┐     ┌──────────────┐ │
│  │ Exporters   │────▶│ Prometheus Server │────▶│   Grafana    │ │
│  │ - node      │     │ - Scraping       │     │ - Dashboards │ │
│  │ - kube-state│     │ - Storage (TSDB) │     │ - Alerts     │ │
│  │ - cAdvisor  │     │ - PromQL engine  │     └──────────────┘ │
│  │ - app       │     │ - Rules engine   │                      │
│  └─────────────┘     └────────┬─────────┘                      │
│                               │                                  │
│                      ┌────────┴─────────┐                       │
│                      │  Alertmanager    │                       │
│                      │  - Routing       │                       │
│                      │  - Silencing     │                       │
│                      │  - Notification  │                       │
│                      └──────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### Service Discovery

Prometheus discovers targets via Kubernetes API:

```yaml
# prometheus.yml - Kubernetes service discovery
scrape_configs:
  # Scrape pods with prometheus.io annotations
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
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
        action: replace
        regex: (.+);(.+)
        replacement: $2:$1
        target_label: __address__

  # Scrape nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

### Essential PromQL Queries for Kubernetes

```bash
# CPU usage by pod (as percentage of request)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
/
sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod, namespace)
* 100

# Memory usage by pod
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)

# Pod restart count (last hour)
increase(kube_pod_container_status_restarts_total[1h])

# Node CPU utilization
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# Pods not ready
kube_pod_status_ready{condition="false"}

# Deployment replicas mismatch
kube_deployment_spec_replicas != kube_deployment_status_available_replicas

# PVC usage percentage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100

# API server request latency (99th percentile)
histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])) by (le, verb))

# Container CPU throttling
rate(container_cpu_cfs_throttled_periods_total[5m])
/
rate(container_cpu_cfs_periods_total[5m]) * 100
```

### Long-Term Storage

| Backend | Architecture | Best For |
|---------|-------------|----------|
| **Thanos** | Sidecar + object storage | Multi-cluster, HA Prometheus |
| **Cortex** | Horizontally scalable | Large-scale multi-tenant |
| **Mimir** | Grafana's fork of Cortex | Grafana ecosystem users |
| **VictoriaMetrics** | Single binary or cluster | Simple setup, high performance |

### kube-prometheus-stack (Helm)

```bash
# Install the full monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=secure-password \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

---

## 16.4 kube-state-metrics

### Key Metrics Families

| Metric | Description | Use Case |
|--------|-------------|----------|
| `kube_pod_status_phase` | Pod phase (Pending/Running/Failed/Succeeded) | Pod health |
| `kube_pod_container_status_restarts_total` | Container restart count | Crash detection |
| `kube_pod_container_status_waiting_reason` | Why container is waiting | Debugging |
| `kube_deployment_status_replicas` | Current replica count | Scaling health |
| `kube_deployment_status_replicas_unavailable` | Unavailable replicas | Rollout issues |
| `kube_node_status_condition` | Node conditions (Ready, Pressure) | Node health |
| `kube_node_status_allocatable` | Allocatable resources | Capacity planning |
| `kube_resourcequota` | Resource quota usage | Quota monitoring |
| `kube_persistentvolumeclaim_status_phase` | PVC phase (Bound/Pending) | Storage health |
| `kube_horizontalpodautoscaler_status_current_replicas` | HPA current state | Autoscaling |
| `kube_job_status_failed` | Failed jobs | Job monitoring |
| `kube_cronjob_next_schedule_time` | Next CronJob run | Schedule monitoring |

### Example Queries

```bash
# Pods in non-running state
kube_pod_status_phase{phase!="Running",phase!="Succeeded"} == 1

# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# Nodes not ready
kube_node_status_condition{condition="Ready",status="true"} == 0

# Resource quota near limit (>80%)
kube_resourcequota{type="used"} / kube_resourcequota{type="hard"} > 0.8

# StatefulSet replicas mismatch
kube_statefulset_status_replicas != kube_statefulset_replicas
```

---

## 16.5 Grafana Dashboards

### USE Method (Infrastructure)

| Resource | Utilization | Saturation | Errors |
|----------|------------|------------|--------|
| **CPU** | CPU usage % | CPU throttling, load average | - |
| **Memory** | Memory usage % | OOMKill count, swap usage | OOM events |
| **Network** | Bandwidth usage | Packet drops, retransmits | Interface errors |
| **Disk** | Disk usage %, IOPS | I/O wait, queue depth | Read/write errors |

### RED Method (Services)

| Signal | Metric | PromQL Example |
|--------|--------|---------------|
| **Rate** | Requests/sec | `sum(rate(http_requests_total[5m])) by (service)` |
| **Errors** | Error rate % | `sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100` |
| **Duration** | Latency p99 | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` |

### Dashboard Hierarchy

```
Cluster Overview
├── Node count, pod count, CPU/memory utilization
├── Top resource consumers
└── Alerts firing

Namespace Dashboard
├── Pod status by namespace
├── Resource usage vs quota
└── Network traffic

Workload Dashboard
├── Deployment status, replicas
├── Pod restarts, errors
├── CPU/memory per pod
└── Network in/out

Pod Dashboard
├── Container resource usage
├── Log error rate
├── Network connections
└── Volume usage
```

---

## 16.6 Alerting Strategies

### Symptom-Based vs Cause-Based

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| **Symptom** | "Error rate > 1%" | Actionable, user-facing | May miss root cause |
| **Cause** | "Node CPU > 90%" | Specific root cause | Alert fatigue, may not affect users |

**Best practice**: Alert on symptoms, investigate causes.

### Critical Kubernetes Alerts

| Alert | Expression | Severity |
|-------|-----------|----------|
| KubePodCrashLooping | `increase(kube_pod_container_status_restarts_total[1h]) > 5` | Warning |
| KubePodNotReady | `kube_pod_status_ready{condition="false"} == 1` for 15m | Warning |
| KubeDeploymentReplicasMismatch | `kube_deployment_status_replicas_unavailable > 0` for 15m | Warning |
| NodeNotReady | `kube_node_status_condition{condition="Ready",status="true"} == 0` for 5m | Critical |
| NodeMemoryPressure | `kube_node_status_condition{condition="MemoryPressure",status="true"} == 1` | Warning |
| NodeDiskPressure | `kube_node_status_condition{condition="DiskPressure",status="true"} == 1` | Warning |
| KubePersistentVolumeFillingUp | `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85` | Warning |
| KubeQuotaExceeded | `kube_resourcequota{type="used"} / kube_resourcequota{type="hard"} > 0.9` | Warning |
| etcdHighCommitDuration | `histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25` | Warning |
| KubeAPIErrorsHigh | `sum(rate(apiserver_request_total{code=~"5.."}[5m])) / sum(rate(apiserver_request_total[5m])) > 0.03` | Critical |

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      group_wait: 10s
    - match:
        severity: warning
      receiver: 'slack'
    - match:
        alertname: Watchdog
      receiver: 'null'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<key>'
  - name: 'slack'
    slack_configs:
      - channel: '#k8s-alerts'
        send_resolved: true
  - name: 'null'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'namespace']
```

---

## 16.7 Custom Metrics & Autoscaling

### HPA with Custom Metrics

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
  maxReplicas: 20
  metrics:
    # CPU-based
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Custom metric from Prometheus
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
    # External metric (e.g., queue depth)
    - type: External
      external:
        metric:
          name: queue_messages_ready
          selector:
            matchLabels:
              queue: "worker-tasks"
        target:
          type: AverageValue
          averageValue: "30"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
```

### KEDA (Kubernetes Event-Driven Autoscaling)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaledobject
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: rabbitmq
      metadata:
        queueName: tasks
        host: amqp://rabbitmq.default:5672
        queueLength: "5"
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: http_requests_total
        query: sum(rate(http_requests_total{deployment="worker"}[2m]))
        threshold: "100"
```

### VPA (Vertical Pod Autoscaler)

| Mode | Behavior |
|------|----------|
| **Off** | Only produces recommendations, no changes |
| **Initial** | Sets resources on pod creation only |
| **Auto** | Updates resources on running pods (causes restarts) |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"  # Start with Off, review recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
```

```bash
# Check VPA recommendations
kubectl get vpa my-app-vpa -o jsonpath='{.status.recommendation}'
```

---

## 16.8 Log Aggregation

### Logging Architectures

```
Node-Level (DaemonSet):             Sidecar Pattern:
┌──────────────────┐                ┌──────────────────┐
│  Node             │               │  Pod              │
│  ┌──────┐         │               │  ┌──────┐ ┌────┐│
│  │Pod A │──┐      │               │  │ App  │→│Log ││
│  └──────┘  │      │               │  └──────┘ │Side││
│  ┌──────┐  ├─▶/var/log/           │            │car ││
│  │Pod B │──┘  containers/         │            └─┬──┘│
│  └──────┘     │   │               └──────────────┼───┘
│  ┌────────┐   │   │                              │
│  │FluentBit│◀─┘   │               ┌──────────────┘
│  │DaemonSet│      │               ▼
│  └───┬────┘       │          Log Backend
│      │            │
└──────┼────────────┘
       ▼
  Log Backend
```

### Fluent Bit DaemonSet Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     info
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            cri
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        K8S-Logging.Parser  On

    [OUTPUT]
        Name            loki
        Match           *
        Host            loki.logging.svc
        Port            3100
        Labels          job=fluent-bit
        Auto_Kubernetes_Labels on
```

### Loki LogQL Examples

```bash
# All logs from a namespace
{namespace="production"}

# Error logs from a specific deployment
{namespace="production", app="api-server"} |= "error"

# JSON parsed logs with filter
{namespace="production"} | json | status_code >= 500

# Log rate per service
sum(rate({namespace="production"} [5m])) by (app)

# Top 10 error messages
topk(10, sum(count_over_time({namespace="production"} |= "error" [1h])) by (app))
```

### Structured Logging Best Practices

```bash
# Good: structured JSON logs
{"timestamp":"2024-01-15T10:30:00Z","level":"error","message":"Connection refused","service":"api","pod":"api-7d9f8","trace_id":"abc123","duration_ms":1500}

# Bad: unstructured logs
ERROR 2024-01-15 10:30:00 Connection refused to database
```

Key fields to include:
- `timestamp` (ISO 8601)
- `level` (error, warn, info, debug)
- `message` (human-readable)
- `service` / `component`
- `trace_id` (for correlation)
- `pod` / `node` (Kubernetes context)
- Request-specific fields (user_id, request_id)

---

## Chapter 17 Review Questions

1. What are the three pillars of observability and how do they complement each other in Kubernetes?

2. Explain the difference between Metrics Server, cAdvisor, and kube-state-metrics. When would you use each?

3. Write a PromQL query to find all pods that have restarted more than 3 times in the last hour.

4. What is the difference between symptom-based and cause-based alerting? Which should you prioritize and why?

5. Compare the node-level DaemonSet and sidecar logging patterns. When would you choose each?

---

## Chapter 17 Hands-On Exercises

### Exercise 16.1: Deploy Monitoring Stack
1. Install kube-prometheus-stack via Helm in a test cluster
2. Access Grafana and explore the pre-built Kubernetes dashboards
3. Create a custom dashboard showing pod restart rates by namespace

### Exercise 16.2: Write Alerting Rules
1. Create a PrometheusRule that alerts when any deployment has unavailable replicas for more than 10 minutes
2. Configure Alertmanager to route critical alerts to one channel and warnings to another
3. Test the alert by scaling a deployment beyond available resources

### Exercise 16.3: Set Up Log Aggregation
1. Deploy Fluent Bit as a DaemonSet and Loki as the backend
2. Configure Grafana to query Loki
3. Write LogQL queries to find error patterns across namespaces

---

## Key Takeaways

1. **Three pillars work together** - Metrics for detection, logs for context, traces for request flow
2. **Metrics Server is limited** - Use Prometheus for historical data and custom metrics
3. **kube-state-metrics complements cAdvisor** - Object state vs container resources
4. **Alert on symptoms, not causes** - "Error rate high" is more actionable than "CPU high"
5. **Structured logging is essential** - JSON logs with trace IDs enable correlation
6. **Right-size your monitoring** - Match observability investment to workload criticality
7. **Autoscaling needs metrics** - HPA, VPA, and KEDA all depend on reliable metrics pipelines

---

[Next Chapter: Kubernetes Debugging Deep Dive →](17-debugging.md)
