# Chapter 22: Kubernetes Reliability Engineering

**Prerequisites:** Before reading this chapter, ensure you understand Containers & Kubernetes (Chapter 12), Troubleshooting (Chapter 13), and Monitoring & Observability (Chapter 17).

Reliability engineering in Kubernetes goes beyond simply keeping applications running. It encompasses designing systems that gracefully handle failures, automatically recover from issues, and maintain service quality even under adverse conditions. This chapter explores the principles, patterns, and practices that enable you to build highly reliable Kubernetes workloads.

---

## 21.1 Reliability Principles

### The Reliability Pyramid

Kubernetes reliability is built on layers, each depending on the stability of the layer below:

```
                    /\
                   /  \
                  /APP \          Application Layer
                 /______\         - Circuit breakers
                /        \        - Retry logic
               / PLATFORM \       - Rate limiting
              /____________\
             /              \     Platform Layer
            /   KUBERNETES   \    - Pod scheduling
           /     CONTROL      \   - Service discovery
          /      PLANE         \  - Auto-healing
         /____________________\
        /                      \  Infrastructure Layer
       /    INFRASTRUCTURE      \ - Node availability
      /     (GKE/Compute)        \- Network reliability
     /__________________________\- Storage durability
```

Each layer must be reliable for the layers above to function properly. You cannot have reliable applications on an unreliable platform.

### Blast Radius Containment

The blast radius is the scope of impact when a failure occurs. Key strategies to minimize it:

**Namespace Isolation**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: team-a
spec:
  limits:
  - max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
    type: Container
```

**Network Policies for Failure Containment**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
```

### Service Level Objectives (SLOs)

SLOs define the target reliability for your services. Common SLIs (Service Level Indicators) for Kubernetes workloads:

**Availability SLO**
```yaml
# Define availability target: 99.9% (43.2 minutes downtime/month)
apiVersion: v1
kind: ConfigMap
metadata:
  name: slo-config
  namespace: monitoring
data:
  availability_slo: "0.999"  # 99.9%
  latency_slo_p50: "100ms"
  latency_slo_p99: "500ms"
  error_rate_slo: "0.001"    # 0.1%
```

**SLO Monitoring with Prometheus**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
  namespace: monitoring
spec:
  groups:
  - name: slo
    interval: 30s
    rules:
    - alert: AvailabilitySLOBreach
      expr: |
        (
          sum(rate(http_requests_total{status=~"2.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
        ) < 0.999
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Availability SLO breached"
        description: "Service availability is {{ $value | humanizePercentage }}"

    - alert: LatencySLOBreach
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
        ) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Latency SLO breached"
        description: "P99 latency is {{ $value }}s"
```

### Error Budgets

An error budget is the maximum amount of unreliability your SLO allows.

**Error Budget Calculation**
```
SLO: 99.9% availability over 30 days
Total time: 30 days × 24 hours × 60 minutes = 43,200 minutes
Allowed downtime: 43,200 × 0.001 = 43.2 minutes per month
```

**Error Budget Policy Example**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: error-budget-policy
  namespace: sre-team
data:
  policy.yaml: |
    slo:
      availability: 99.9%
      period: 30d

    error_budget_policy:
      - budget_remaining: 100-75%
        actions:
          - Continue normal development velocity
          - Deploy during business hours

      - budget_remaining: 75-50%
        actions:
          - Review recent changes
          - Increase testing rigor
          - Deploy during off-peak hours

      - budget_remaining: 50-25%
        actions:
          - Slow down feature releases
          - Focus on reliability improvements
          - Require SRE approval for deploys

      - budget_remaining: 25-0%
        actions:
          - Feature freeze
          - All hands on reliability
          - Emergency incident review
```

### Kubernetes Reliability Maturity Model

| Level | Description | Characteristics | Example Practices |
|-------|-------------|-----------------|-------------------|
| **1 - Reactive** | Manual intervention required | No automation, frequent outages, no monitoring | Manual pod restarts, no health checks |
| **2 - Proactive** | Basic automation in place | Health checks, basic monitoring, manual scaling | Liveness/readiness probes, log aggregation |
| **3 - Predictive** | Automated responses to issues | Auto-scaling, self-healing, incident alerts | HPA, PDB, automated rollbacks |
| **4 - Preventive** | Issues detected before impact | Chaos testing, canary deployments, SLOs | Progressive delivery, synthetic monitoring |
| **5 - Optimized** | Continuous improvement culture | Full observability, error budgets, game days | Chaos engineering, capacity planning, blameless postmortems |

---

## 21.2 High Availability Patterns

### Pod Anti-Affinity

Ensure pods are distributed across nodes and zones to prevent single points of failure.

**Node-Level Anti-Affinity**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          # Hard requirement: never schedule on same node
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-server
            topologyKey: kubernetes.io/hostname

          # Soft preference: prefer different zones
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: topology.kubernetes.io/zone

      containers:
      - name: web-server
        image: nginx:1.21
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### Topology Spread Constraints

More flexible and powerful than anti-affinity for controlling pod distribution.

**Even Distribution Across Zones**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      topologySpreadConstraints:
      # Spread across zones with max skew of 1
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api-server

      # Spread across nodes with max skew of 2
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: api-server

      containers:
      - name: api-server
        image: api-server:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
```

**Explanation of Parameters:**
- `maxSkew: 1` - Maximum difference in pod count between any two topology domains
- `whenUnsatisfiable: DoNotSchedule` - Hard constraint (fail if can't satisfy)
- `whenUnsatisfiable: ScheduleAnyway` - Soft constraint (best effort)
- `topologyKey` - Node label that defines the topology domain

### Pod Disruption Budgets (PDBs)

PDBs ensure a minimum number of pods remain available during voluntary disruptions (node drains, upgrades).

**minAvailable vs maxUnavailable Comparison**

| Strategy | Use Case | Example | Effect |
|----------|----------|---------|--------|
| `minAvailable` | Ensure minimum capacity | `minAvailable: 2` | At least 2 pods must be running |
| `minAvailable` (%) | Percentage-based capacity | `minAvailable: 75%` | At least 75% of pods must be running |
| `maxUnavailable` | Control disruption rate | `maxUnavailable: 1` | Only 1 pod can be down at a time |
| `maxUnavailable` (%) | Percentage-based disruption | `maxUnavailable: 25%` | Up to 25% of pods can be down |

**PDB Examples**
```yaml
# Example 1: Critical service - always keep majority running
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
spec:
  minAvailable: 75%
  selector:
    matchLabels:
      app: critical-service
      tier: production
---
# Example 2: Database - only disrupt one at a time
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: postgres
---
# Example 3: Stateless API - ensure minimum count
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: api-server
```

### Graceful Shutdown

Handle termination signals properly to avoid dropping in-flight requests.

**Complete Graceful Shutdown Configuration**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: graceful-app
  template:
    metadata:
      labels:
        app: graceful-app
    spec:
      terminationGracePeriodSeconds: 60  # Allow up to 60s for shutdown

      containers:
      - name: app
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
          name: http

        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "/app/prestop.sh"]

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 1  # Remove from service quickly

        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 512Mi
```

**PreStop Hook Script** (`/app/prestop.sh`)
```bash
#!/bin/bash
# Graceful shutdown script

echo "PreStop hook triggered at $(date)"

# 1. Stop accepting new connections
echo "Stopping health check endpoint..."
touch /tmp/shutdown

# 2. Wait for existing connections to drain
echo "Waiting for connections to drain..."
sleep 5

# 3. Notify load balancers
echo "Deregistering from service mesh..."
curl -X POST http://localhost:15000/healthcheck/fail || true

# 4. Wait for removal from endpoints
sleep 10

# 5. Graceful application shutdown
echo "Shutting down application..."
kill -TERM 1

echo "PreStop hook completed at $(date)"
```

### Health Check Comparison

| Probe Type | Purpose | When to Use | Typical Configuration |
|------------|---------|-------------|----------------------|
| **Startup** | Allow slow-starting apps | Legacy apps, large initialization | Long period, high threshold |
| **Liveness** | Detect deadlocks/crashes | All containers | Conservative timeouts |
| **Readiness** | Control traffic routing | All services receiving traffic | Aggressive failure threshold |

**Comprehensive Health Check Configuration**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complex-app
  template:
    metadata:
      labels:
        app: complex-app
    spec:
      containers:
      - name: app
        image: complex-app:v2.1.0
        ports:
        - containerPort: 8080

        # Startup probe - for slow-starting applications
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 30  # 30 * 10 = 300s (5 min) to start
          successThreshold: 1

        # Liveness probe - detect if container needs restart
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            httpHeaders:
            - name: X-Probe-Type
              value: Liveness
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3    # Restart after 30s of failures
          successThreshold: 1

        # Readiness probe - control traffic routing
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
            httpHeaders:
            - name: X-Probe-Type
              value: Readiness
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 1    # Remove from service immediately
          successThreshold: 2    # Require 2 successes before routing traffic

        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2
            memory: 2Gi
```

**Health Check Endpoint Implementation** (Go example)
```go
package main

import (
    "context"
    "net/http"
    "sync/atomic"
    "time"
)

var (
    isReady   atomic.Bool
    isHealthy atomic.Bool
)

func healthzHandler(w http.ResponseWriter, r *http.Request) {
    if !isHealthy.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte("unhealthy"))
        return
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("healthy"))
}

func readyHandler(w http.ResponseWriter, r *http.Request) {
    if !isReady.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte("not ready"))
        return
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ready"))
}

func main() {
    // Start unhealthy and not ready
    isHealthy.Store(false)
    isReady.Store(false)

    http.HandleFunc("/healthz", healthzHandler)
    http.HandleFunc("/ready", readyHandler)

    // Simulate startup process
    go func() {
        time.Sleep(10 * time.Second)
        isHealthy.Store(true)

        // Initialize dependencies
        time.Sleep(5 * time.Second)
        isReady.Store(true)
    }()

    http.ListenAndServe(":8080", nil)
}
```

### Multi-AZ Deployment Pattern

**Three-Zone High Availability**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-az-app
spec:
  replicas: 9  # 3 per zone
  selector:
    matchLabels:
      app: multi-az-app
  template:
    metadata:
      labels:
        app: multi-az-app
    spec:
      # Spread evenly across zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: multi-az-app

      # Prefer different nodes within zone
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - multi-az-app
              topologyKey: kubernetes.io/hostname

      containers:
      - name: app
        image: app:v1.0.0
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: multi-az-app-pdb
spec:
  minAvailable: 66%  # Always keep 2 zones worth
  selector:
    matchLabels:
      app: multi-az-app
```

---

## 21.3 Auto-Healing & Self-Recovery

### Controller Reconciliation Loop

Kubernetes controllers continuously reconcile desired state with actual state.

**How Controllers Heal Failures**
```
┌─────────────────────────────────────────────────────────────┐
│                  CONTROLLER RECONCILIATION                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │   DESIRED    │         │    ACTUAL    │                 │
│  │    STATE     │         │    STATE     │                 │
│  │              │         │              │                 │
│  │ replicas: 3  │         │ replicas: 2  │                 │
│  └──────┬───────┘         └──────┬───────┘                 │
│         │                        │                          │
│         │    ┌──────────────┐    │                          │
│         └───>│  CONTROLLER  │<───┘                          │
│              │    LOOP      │                               │
│              └──────┬───────┘                               │
│                     │                                        │
│                     │ Detect Drift                           │
│                     │ (3 desired - 2 actual = 1 missing)    │
│                     v                                        │
│              ┌──────────────┐                               │
│              │  RECONCILE   │                               │
│              │   ACTIONS    │                               │
│              └──────┬───────┘                               │
│                     │                                        │
│                     │ • Create new pod                       │
│                     │ • Wait for pod ready                   │
│                     │ • Update status                        │
│                     v                                        │
│              ┌──────────────┐                               │
│              │  CONVERGED   │                               │
│              │   (3 pods)   │                               │
│              └──────────────┘                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Deployment Auto-Healing & Rollback

**Automatic Rollback on Failed Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auto-rollback-app
spec:
  replicas: 5
  revisionHistoryLimit: 10  # Keep last 10 revisions

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployments

  # Progressive deadline - rollout must complete in 10 minutes
  progressDeadlineSeconds: 600

  selector:
    matchLabels:
      app: auto-rollback-app

  template:
    metadata:
      labels:
        app: auto-rollback-app
        version: v2.0.0
    spec:
      containers:
      - name: app
        image: app:v2.0.0
        ports:
        - containerPort: 8080

        # Readiness probe prevents bad pods from receiving traffic
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 1

        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1
            memory: 512Mi
```

**Manual Rollback Commands**
```bash
# Check rollout status
kubectl rollout status deployment/auto-rollback-app

# View rollout history
kubectl rollout history deployment/auto-rollback-app

# View specific revision details
kubectl rollout history deployment/auto-rollback-app --revision=3

# Rollback to previous version
kubectl rollout undo deployment/auto-rollback-app

# Rollback to specific revision
kubectl rollout undo deployment/auto-rollback-app --to-revision=2

# Pause a problematic rollout
kubectl rollout pause deployment/auto-rollback-app

# Resume after investigation
kubectl rollout resume deployment/auto-rollback-app
```

### StatefulSet Self-Recovery

StatefulSets maintain pod identity and recover pods with the same name and storage.

**StatefulSet with Recovery Configuration**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3

  # Ordered updates - wait for each pod to be ready
  podManagementPolicy: OrderedReady

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update all pods (set to N to update only pods >= N)

  selector:
    matchLabels:
      app: database

  template:
    metadata:
      labels:
        app: database
    spec:
      terminationGracePeriodSeconds: 60

      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
          name: postgres

        env:
        - name: POSTGRES_DB
          value: mydb
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata

        # Readiness check ensures pod is ready before proceeding
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3

        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: pd-ssd
      resources:
        requests:
          storage: 100Gi
```

**StatefulSet Recovery Behavior:**
1. Pod `database-1` fails
2. Controller detects pod is not running
3. Controller recreates pod with same name: `database-1`
4. Pod mounts same PVC: `data-database-1`
5. Application recovers with persistent data intact

### DaemonSet Self-Healing

DaemonSets ensure one pod runs on each matching node.

**DaemonSet with Node Recovery**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: monitoring-agent

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Update one node at a time

  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      # Tolerate all taints to run on every node
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists

      # Use host network for node-level monitoring
      hostNetwork: true
      hostPID: true

      containers:
      - name: agent
        image: monitoring-agent:v1.5.0

        securityContext:
          privileged: true

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

**DaemonSet Recovery Scenarios:**
- Node comes online → DaemonSet controller schedules pod
- Pod crashes → Controller recreates on same node
- Node cordoned → Pod remains (unless drained)
- Node drained → Pod deleted, recreated when uncordoned
- New node added → Pod automatically scheduled

### Node Auto-Repair

GKE can automatically repair unhealthy nodes.

**Node Auto-Repair Configuration**
```bash
# Enable auto-repair on existing node pool
gcloud container node-pools update default-pool \
  --cluster=production-cluster \
  --zone=us-central1-a \
  --enable-autorepair

# Create node pool with auto-repair enabled
gcloud container node-pools create reliable-pool \
  --cluster=production-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --enable-autorepair \
  --enable-autoupgrade \
  --machine-type=n2-standard-4
```

**Auto-Repair Triggers:**
1. Node becomes `NotReady` for > 10 minutes
2. Node boot disk is full
3. Node fails health checks repeatedly
4. Kubelet is not reporting metrics

**Auto-Repair Process:**
```
Node Unhealthy Detected
        ↓
Wait 10 minutes (grace period)
        ↓
Still unhealthy?
        ↓ Yes
Cordon node (prevent new pods)
        ↓
Drain node (evict existing pods)
        ↓
Delete node (VM instance)
        ↓
Create new node
        ↓
New node joins cluster
        ↓
Pods rescheduled
```

---

## 21.4 Progressive Delivery

Progressive delivery gradually rolls out changes to minimize risk.

### Deployment Strategies Comparison

| Strategy | Traffic Pattern | Rollback Speed | Resource Usage | Complexity |
|----------|----------------|----------------|----------------|------------|
| **Rolling Update** | Gradual shift | Medium | Low (no extra pods) | Low |
| **Blue/Green** | Instant switch | Instant | High (2x resources) | Medium |
| **Canary** | Percentage-based | Fast | Medium | High |
| **A/B Testing** | User-segment based | N/A | Medium | High |

### Argo Rollouts

Argo Rollouts extends Kubernetes with advanced deployment strategies.

**Installation**
```bash
# Install Argo Rollouts controller
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f \
  https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

**Canary Rollout with Analysis**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
spec:
  replicas: 10
  revisionHistoryLimit: 3

  selector:
    matchLabels:
      app: api-server

  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: api-server:v2.0.0
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: 200m
            memory: 256Mi

  strategy:
    canary:
      # Reference to stable and canary services
      stableService: api-server-stable
      canaryService: api-server-canary

      # Traffic routing (requires service mesh or ingress)
      trafficRouting:
        istio:
          virtualService:
            name: api-server-vsvc
            routes:
            - primary

      # Canary steps - gradual traffic increase
      steps:
      # Step 1: Deploy canary, send 10% traffic
      - setWeight: 10
      - pause: {duration: 5m}

      # Step 2: Run analysis
      - analysis:
          templates:
          - templateName: error-rate-check
          - templateName: latency-check

      # Step 3: Increase to 25%
      - setWeight: 25
      - pause: {duration: 5m}

      # Step 4: Run analysis again
      - analysis:
          templates:
          - templateName: error-rate-check
          - templateName: latency-check

      # Step 5: Increase to 50%
      - setWeight: 50
      - pause: {duration: 10m}

      # Step 6: Final analysis before full rollout
      - analysis:
          templates:
          - templateName: error-rate-check
          - templateName: latency-check

      # Step 7: Full rollout
      - setWeight: 100

      # Anti-affinity to prevent canary and stable on same node
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
---
# Stable service (baseline traffic)
apiVersion: v1
kind: Service
metadata:
  name: api-server-stable
spec:
  selector:
    app: api-server
  ports:
  - port: 80
    targetPort: 8080
---
# Canary service (test traffic)
apiVersion: v1
kind: Service
metadata:
  name: api-server-canary
spec:
  selector:
    app: api-server
  ports:
  - port: 80
    targetPort: 8080
```

**Analysis Templates**
```yaml
# Error rate analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
  - name: error-rate
    initialDelay: 1m
    interval: 1m
    count: 5
    successCondition: result < 0.01  # Less than 1% error rate
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
---
# Latency analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
spec:
  metrics:
  - name: p99-latency
    initialDelay: 1m
    interval: 1m
    count: 5
    successCondition: result < 0.5  # P99 latency under 500ms
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{app="api-server"}[5m])) by (le)
          )
```

**Argo Rollouts CLI Commands**
```bash
# Watch rollout progress
kubectl argo rollouts get rollout api-server --watch

# View rollout status
kubectl argo rollouts status api-server

# Promote canary to stable (skip remaining steps)
kubectl argo rollouts promote api-server

# Abort rollout and roll back
kubectl argo rollouts abort api-server
kubectl argo rollouts undo api-server

# Retry failed rollout
kubectl argo rollouts retry rollout api-server

# Set image for rollout
kubectl argo rollouts set image api-server api-server=api-server:v2.1.0
```

### Blue/Green Deployment

**Blue/Green with Argo Rollouts**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: blue-green-app
spec:
  replicas: 5

  selector:
    matchLabels:
      app: blue-green-app

  template:
    metadata:
      labels:
        app: blue-green-app
    spec:
      containers:
      - name: app
        image: app:v2.0.0
        ports:
        - containerPort: 8080

  strategy:
    blueGreen:
      # Service that receives production traffic
      activeService: blue-green-app-active

      # Service for preview/testing before promotion
      previewService: blue-green-app-preview

      # Automatically promote after successful tests
      autoPromotionEnabled: false

      # Keep old version for quick rollback
      scaleDownDelaySeconds: 300

      # Run smoke tests before promotion
      prePromotionAnalysis:
        templates:
        - templateName: smoke-tests

      # Monitor after promotion
      postPromotionAnalysis:
        templates:
        - templateName: error-rate-check
        - templateName: latency-check
---
apiVersion: v1
kind: Service
metadata:
  name: blue-green-app-active
spec:
  selector:
    app: blue-green-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: blue-green-app-preview
spec:
  selector:
    app: blue-green-app
  ports:
  - port: 80
    targetPort: 8080
```

### Flagger

Flagger automates progressive delivery with service meshes.

**Flagger Canary with Istio**
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api-server
  namespace: production
spec:
  # Target deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  # Autoscaling
  autoscalerRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: api-server

  # Service mesh provider
  provider: istio

  # Service configuration
  service:
    port: 80
    targetPort: 8080
    gateways:
    - public-gateway
    hosts:
    - api.example.com

  # Analysis configuration
  analysis:
    # Schedule for metrics evaluation
    interval: 1m

    # Number of successful checks before promotion
    threshold: 5

    # Max weight during canary phase
    maxWeight: 50

    # Weight increment step
    stepWeight: 10

    # Metrics to check
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m

    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m

    # Webhook tests
    webhooks:
    - name: load-test
      url: http://flagger-loadtester/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://api-server-canary/"
```

---

## 21.5 Chaos Engineering

Chaos engineering proactively tests system resilience by injecting failures.

### Chaos Engineering Principles

1. **Define Steady State** - Establish normal behavior metrics
2. **Hypothesize** - Predict how the system will respond
3. **Inject Failure** - Introduce real-world failures
4. **Observe** - Measure impact on steady state
5. **Learn & Improve** - Fix weaknesses and iterate

### Chaos Mesh

Chaos Mesh is a cloud-native chaos engineering platform for Kubernetes.

**Installation**
```bash
# Install Chaos Mesh
curl -sSL https://mirrors.chaos-mesh.org/v2.6.0/install.sh | bash

# Verify installation
kubectl get pods -n chaos-mesh

# Access dashboard
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

**Pod Chaos - Random Pod Deletion**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-experiment
  namespace: chaos-experiments
spec:
  action: pod-kill

  mode: fixed
  value: '2'  # Kill 2 pods

  # Alternative modes:
  # mode: one          # Kill one pod
  # mode: all          # Kill all matching pods
  # mode: fixed-percent
  # value: '50'        # Kill 50% of pods
  # mode: random-max-percent
  # value: '50'        # Kill up to 50% randomly

  selector:
    namespaces:
    - production
    labelSelectors:
      app: web-server

  scheduler:
    cron: '@every 10m'

  duration: '30s'
```

**Pod Chaos - Container Failure**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: container-kill
  namespace: chaos-experiments
spec:
  action: container-kill

  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: api-server

  containerNames:
  - api-server

  duration: '60s'

  scheduler:
    cron: '0 */2 * * *'  # Every 2 hours
```

**Network Chaos - Latency Injection**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
  namespace: chaos-experiments
spec:
  action: delay

  mode: all

  selector:
    namespaces:
    - production
    labelSelectors:
      app: payment-service

  delay:
    latency: '100ms'
    correlation: '100'
    jitter: '10ms'

  duration: '5m'

  direction: to  # Affect outgoing traffic

  # Target specific endpoints
  target:
    mode: all
    selector:
      namespaces:
      - production
      labelSelectors:
        app: database
```

**Network Chaos - Packet Loss**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-loss
  namespace: chaos-experiments
spec:
  action: loss

  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: microservice-a

  loss:
    loss: '10'        # 10% packet loss
    correlation: '25' # Correlation between losses

  duration: '3m'

  direction: to
```

**Network Chaos - Partition**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-partition
  namespace: chaos-experiments
spec:
  action: partition

  mode: all

  selector:
    namespaces:
    - production
    labelSelectors:
      app: frontend

  direction: both

  target:
    mode: all
    selector:
      namespaces:
      - production
      labelSelectors:
        app: backend

  duration: '2m'
```

**IO Chaos - Read/Write Failures**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-errno
  namespace: chaos-experiments
spec:
  action: fault

  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: database

  volumePath: /var/lib/postgresql/data

  path: '/var/lib/postgresql/data/**/*'

  errno: 5  # EIO (I/O error)

  percent: 50  # Affect 50% of operations

  duration: '1m'
```

**IO Chaos - Latency**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-latency
  namespace: chaos-experiments
spec:
  action: latency

  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: file-processor

  volumePath: /data

  delay: '500ms'

  percent: 100

  duration: '5m'
```

**Stress Chaos - CPU Stress**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress
  namespace: chaos-experiments
spec:
  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: compute-intensive

  stressors:
    cpu:
      workers: 2
      load: 80  # 80% CPU per worker

  duration: '5m'
```

**Stress Chaos - Memory Stress**
```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress
  namespace: chaos-experiments
spec:
  mode: one

  selector:
    namespaces:
    - production
    labelSelectors:
      app: cache-service

  stressors:
    memory:
      workers: 1
      size: '512MB'

  duration: '3m'
```

### Chaos Experiment Catalog

| Experiment Type | Scenario | Expected Outcome | Safety Level |
|----------------|----------|------------------|--------------|
| **Pod Kill** | Random pod deletion | Deployment creates new pods, no downtime | High |
| **Network Latency** | Slow network responses | Timeouts handled gracefully, retries work | Medium |
| **Network Partition** | Split-brain scenarios | Service mesh routes around failure | Medium |
| **IO Errors** | Disk failures | Applications handle read/write errors | Low |
| **CPU Stress** | Resource exhaustion | HPA scales, QoS enforcement works | Medium |
| **Memory Stress** | OOM scenarios | Container limits enforced, pods restart | Low |
| **DNS Failure** | Service discovery issues | Caching prevents total failure | Low |

### Game Day Planning

**Chaos Game Day Runbook Template**
```yaml
name: "Quarterly Reliability Game Day"
date: "2026-03-15"
duration: "4 hours"
participants:
  - SRE Team
  - Development Team
  - Product Owner
  - Incident Commander

objectives:
  - Validate disaster recovery procedures
  - Test monitoring and alerting
  - Practice incident response
  - Identify system weaknesses

pre-requisites:
  - All participants notified 2 weeks in advance
  - Customer support team on standby
  - Staging environment prepared
  - Monitoring dashboards ready
  - Rollback procedures documented

scenarios:
  - name: "Zone Failure"
    description: "Simulate complete failure of us-central1-a"
    duration: "30 minutes"
    expected_outcome: "Traffic automatically routes to other zones"

  - name: "Database Latency"
    description: "Inject 500ms latency to database connections"
    duration: "20 minutes"
    expected_outcome: "Application timeouts trigger circuit breakers"

  - name: "Pod Cascading Failures"
    description: "Delete 50% of frontend pods"
    duration: "15 minutes"
    expected_outcome: "Deployments restore pods, no user impact"

safety_measures:
  - Start in staging environment
  - Have rollback procedures ready
  - Set blast radius limits
  - Monitor customer impact metrics
  - Abort criteria defined

success_criteria:
  - No customer-facing downtime
  - All alerts triggered correctly
  - Incident response < 5 minutes
  - Automated recovery successful

post_game_day:
  - Debrief meeting within 24 hours
  - Document findings
  - Create improvement tickets
  - Schedule remediation work
```

### Safe Production Chaos

**Best Practices for Production Chaos**
```bash
# 1. Start small - single pod in non-critical service
kubectl apply -f experiments/pod-kill-non-critical.yaml

# 2. Use time windows - only during business hours with SRE coverage
# Set scheduler in chaos experiment

# 3. Monitor actively - watch dashboards during experiments
kubectl port-forward -n monitoring svc/grafana 3000:3000

# 4. Define abort conditions
cat <<EOF | kubectl apply -f -
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: controlled-chaos
spec:
  schedule: '0 14 * * 1-5'  # 2 PM on weekdays
  type: PodChaos
  podChaos:
    action: pod-kill
    mode: one
    selector:
      namespaces:
      - production
      labelSelectors:
        chaos: enabled
EOF

# 5. Use feature flags to enable/disable chaos
kubectl label namespace production chaos=enabled
kubectl label namespace production chaos=disabled  # Disable chaos
```

---

## 21.6 Load Testing & Performance

### k6 Load Testing

k6 is a developer-centric load testing tool.

**Installation**
```bash
# Install k6
curl https://github.com/grafana/k6/releases/download/v0.47.0/k6-v0.47.0-linux-amd64.tar.gz -L | tar xvz
sudo mv k6-v0.47.0-linux-amd64/k6 /usr/local/bin/

# Verify installation
k6 version
```

**Basic Load Test Script**
```javascript
// load-test-basic.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],  // Less than 1% errors
    'errors': ['rate<0.05'],
  },
};

export default function () {
  const res = http.get('http://api-server.production.svc.cluster.local/api/v1/data');

  const checkRes = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'response has data': (r) => r.json('data') !== undefined,
  });

  errorRate.add(!checkRes);

  sleep(1);
}
```

**Advanced Load Test with Multiple Scenarios**
```javascript
// load-test-advanced.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

// Custom metrics
const apiLatency = new Trend('api_latency');
const dbQueryCount = new Counter('db_queries');

export const options = {
  scenarios: {
    // Read-heavy scenario
    read_workload: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '2m', target: 0 },
      ],
      gracefulRampDown: '30s',
      exec: 'readScenario',
    },

    // Write-heavy scenario
    write_workload: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 20 },
        { duration: '5m', target: 20 },
        { duration: '2m', target: 0 },
      ],
      gracefulRampDown: '30s',
      exec: 'writeScenario',
    },

    // Spike test
    spike_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '10s', target: 100 },
        { duration: '1m', target: 100 },
        { duration: '10s', target: 0 },
      ],
      startTime: '10m',
      exec: 'spikeScenario',
    },
  },

  thresholds: {
    'http_req_duration': ['p(99)<1000'],
    'http_req_failed': ['rate<0.01'],
    'api_latency': ['p(95)<400'],
  },
};

const BASE_URL = 'http://api-server.production.svc.cluster.local';

export function readScenario() {
  group('Read Operations', function () {
    const res = http.get(`${BASE_URL}/api/v1/users/123`);

    check(res, {
      'read status is 200': (r) => r.status === 200,
    });

    apiLatency.add(res.timings.duration);
  });

  sleep(1);
}

export function writeScenario() {
  group('Write Operations', function () {
    const payload = JSON.stringify({
      name: 'Test User',
      email: `test-${Date.now()}@example.com`,
    });

    const params = {
      headers: { 'Content-Type': 'application/json' },
    };

    const res = http.post(`${BASE_URL}/api/v1/users`, payload, params);

    check(res, {
      'write status is 201': (r) => r.status === 201,
    });

    dbQueryCount.add(1);
  });

  sleep(2);
}

export function spikeScenario() {
  group('Spike Load', function () {
    const res = http.get(`${BASE_URL}/api/v1/data`);

    check(res, {
      'spike status is 200 or 503': (r) => r.status === 200 || r.status === 503,
    });
  });

  sleep(0.5);
}
```

**Running Load Tests**
```bash
# Run basic load test
k6 run load-test-basic.js

# Run with custom VUs and duration
k6 run --vus 100 --duration 30s load-test-basic.js

# Run with results output
k6 run --out json=results.json load-test-basic.js

# Run with InfluxDB output for Grafana
k6 run --out influxdb=http://influxdb:8086/k6 load-test-advanced.js

# Run in Kubernetes
kubectl run k6 --image=grafana/k6:latest --rm -it --restart=Never \
  --command -- run - <load-test-basic.js
```

### HPA Behavior Under Load

**Observing HPA Scaling Timeline**
```
Time    | CPU Usage | Pods | Status
--------|-----------|------|----------------------------------
00:00   | 30%       | 3    | Normal operation
00:05   | 80%       | 3    | Load increases
00:06   | 85%       | 3    | HPA detects high CPU
00:07   | 90%       | 3    | HPA calculates desired: 6 pods
00:08   | 85%       | 4    | New pod scheduling
00:09   | 80%       | 5    | Pods starting
00:10   | 70%       | 6    | All pods ready
00:11   | 60%       | 6    | Load distributed
00:15   | 55%       | 6    | Stable state
```

**HPA with Custom Metrics**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: performance-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server

  minReplicas: 3
  maxReplicas: 50

  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

### Bottleneck Identification

**Common Performance Bottlenecks**
```bash
# 1. CPU bottleneck
kubectl top pods -n production --sort-by=cpu

# 2. Memory bottleneck
kubectl top pods -n production --sort-by=memory

# 3. Network bottleneck - check pod network metrics
kubectl exec -it <pod> -- netstat -s

# 4. Disk I/O bottleneck
kubectl exec -it <pod> -- iostat -x 1

# 5. Check for throttled containers
kubectl describe pod <pod> | grep -A 5 "throttling"

# 6. Analyze slow requests with distributed tracing
# (Check Jaeger/Zipkin for trace spans)
```

### Key Performance Metrics

| Metric | Description | Good Target | Critical Threshold |
|--------|-------------|-------------|-------------------|
| **Request Latency (P50)** | Median response time | < 100ms | > 500ms |
| **Request Latency (P99)** | 99th percentile | < 500ms | > 2s |
| **Error Rate** | Failed requests / total | < 0.1% | > 1% |
| **Request Rate** | Requests per second | Baseline dependent | N/A |
| **CPU Utilization** | Average CPU usage | 50-70% | > 90% |
| **Memory Utilization** | Average memory usage | 60-80% | > 95% |
| **Pod Restart Rate** | Restarts per hour | 0 | > 1 |
| **Saturation** | Resource queue depth | 0 | > 10 |

---

## 21.7 Disaster Recovery & Business Continuity

### RPO and RTO

**Recovery Objectives**
```
RPO (Recovery Point Objective)
│
│   ◄──────────────────►
│   Maximum acceptable
│   data loss
│
├───────────────────────────────────────► Time
    ▲                   ▲
    │                   │
  Disaster            Recovery
  Occurs              Complete
                      │
                      ◄──────────►
                      RTO (Recovery
                      Time Objective)
```

**RPO/RTO Examples**

| Service Tier | RPO | RTO | Backup Strategy | Cost |
|--------------|-----|-----|----------------|------|
| **Critical** | 5 min | 15 min | Continuous replication | High |
| **High** | 1 hour | 4 hours | Hourly snapshots | Medium |
| **Medium** | 24 hours | 8 hours | Daily backups | Low |
| **Low** | 7 days | 24 hours | Weekly backups | Minimal |

### Active-Active Multi-Region

**Active-Active Architecture**
```
                    Global Load Balancer
                    (Cloud Load Balancing)
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │ Region 1  │     │ Region 2  │     │ Region 3  │
    │ us-east1  │     │ us-west1  │     │ eu-west1  │
    └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
          │                 │                 │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │ GKE       │     │ GKE       │     │ GKE       │
    │ Cluster   │◄────┤ Cluster   │─────┤ Cluster   │
    └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
          │                 │                 │
    ┌─────▼─────┐     ┌─────▼─────┐     ┌─────▼─────┐
    │ Cloud SQL │     │ Cloud SQL │     │ Cloud SQL │
    │ (Primary) │◄────┤ (Replica) │─────┤ (Replica) │
    └───────────┘     └───────────┘     └───────────┘
```

**Multi-Region Deployment**
```yaml
# Region 1 deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    region: us-east1
spec:
  replicas: 10
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
        region: us-east1
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values:
                - us-east1

      containers:
      - name: api-server
        image: api-server:v1.0.0
        env:
        - name: REGION
          value: "us-east1"
        - name: DATABASE_HOST
          value: "db-us-east1.example.com"
```

### Active-Passive Failover

**Active-Passive Configuration**
```yaml
# Primary cluster (active)
apiVersion: v1
kind: ConfigMap
metadata:
  name: failover-config
  namespace: production
data:
  mode: "active"
  primary_region: "us-central1"
  backup_region: "us-east1"

  # Health check configuration
  health_check_interval: "30s"
  failover_threshold: "3"

  # Automatic failover enabled
  auto_failover: "true"
---
# Backup cluster (passive) - minimal resources
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server-standby
  namespace: production
spec:
  replicas: 2  # Minimal replicas in standby
  selector:
    matchLabels:
      app: api-server-standby
  template:
    metadata:
      labels:
        app: api-server-standby
    spec:
      containers:
      - name: api-server
        image: api-server:v1.0.0
        env:
        - name: MODE
          value: "standby"
```

### Disaster Recovery Testing

**DR Test Runbook**
```bash
#!/bin/bash
# dr-test.sh - Disaster Recovery Test Script

set -e

echo "=== Disaster Recovery Test ==="
echo "Start time: $(date)"

# 1. Verify backup availability
echo "[1/6] Checking backups..."
gsutil ls gs://prod-backups/$(date +%Y-%m-%d)/ || {
  echo "ERROR: No backups found for today"
  exit 1
}

# 2. Create test namespace in DR cluster
echo "[2/6] Creating test namespace in DR cluster..."
kubectl --context=dr-cluster create namespace dr-test

# 3. Restore from backup
echo "[3/6] Restoring from backup..."
velero restore create dr-test-restore \
  --from-backup daily-backup-20260218 \
  --namespace-mappings production:dr-test \
  --context=dr-cluster

# 4. Wait for restore to complete
echo "[4/6] Waiting for restore..."
velero restore describe dr-test-restore --context=dr-cluster

# 5. Verify application health
echo "[5/6] Verifying application..."
kubectl --context=dr-cluster wait --for=condition=ready pod \
  -l app=api-server -n dr-test --timeout=300s

# 6. Run smoke tests
echo "[6/6] Running smoke tests..."
kubectl --context=dr-cluster run smoke-test \
  --image=curlimages/curl:latest \
  --rm -i --restart=Never \
  --namespace=dr-test \
  -- curl -f http://api-server.dr-test/healthz

echo "=== DR Test Completed Successfully ==="
echo "End time: $(date)"

# Cleanup
kubectl --context=dr-cluster delete namespace dr-test
```

**Automated DR Testing Schedule**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: dr-test
  namespace: sre-tools
spec:
  schedule: "0 2 * * 0"  # Every Sunday at 2 AM
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: dr-tester
          containers:
          - name: dr-test
            image: google/cloud-sdk:latest
            command:
            - /bin/bash
            - -c
            - |
              # Run DR test script
              /scripts/dr-test.sh

              # Send notification
              curl -X POST https://slack.com/api/chat.postMessage \
                -H "Authorization: Bearer ${SLACK_TOKEN}" \
                -d "channel=#sre" \
                -d "text=Weekly DR test completed successfully"

            volumeMounts:
            - name: scripts
              mountPath: /scripts

            env:
            - name: SLACK_TOKEN
              valueFrom:
                secretKeyRef:
                  name: slack-credentials
                  key: token

          volumes:
          - name: scripts
            configMap:
              name: dr-scripts
              defaultMode: 0755

          restartPolicy: OnFailure
```

### Disaster Recovery Runbook Template

**DR Runbook Structure**
```yaml
runbook:
  name: "Production Cluster DR - Region Failure"
  version: "2.1"
  last_updated: "2026-02-15"
  owner: "SRE Team"

  scenario: "Complete failure of us-central1 region"

  detection:
    - Alert: "ClusterUnreachable"
    - Monitor: "GCP Console shows region outage"
    - Verification: "kubectl commands timeout"

  decision_criteria:
    - Region outage > 15 minutes
    - RTO target: 30 minutes
    - RPO target: 5 minutes
    - Approval required: On-call SRE Lead

  steps:
    1_assess:
      - Verify scope of outage
      - Check GCP status dashboard
      - Confirm backup cluster health
      - Alert stakeholders

    2_failover:
      - Switch DNS to backup region
        command: "gcloud dns record-sets update api.example.com --rrdatas=34.56.78.90"
      - Update load balancer backends
        command: "gcloud compute backend-services update api-backend --region=us-east1"
      - Scale up standby deployments
        command: "kubectl scale deployment api-server --replicas=20 --context=dr-cluster"

    3_verify:
      - Test application endpoints
      - Check monitoring dashboards
      - Verify database connectivity
      - Run smoke tests

    4_communicate:
      - Update status page
      - Notify customer support
      - Post in #incidents channel
      - Email stakeholders

    5_monitor:
      - Watch error rates
      - Monitor latency
      - Check for cascading failures
      - Track customer reports

  rollback:
    - "If failover causes issues:"
    - "1. Switch DNS back to primary region"
    - "2. Scale down DR cluster"
    - "3. Investigate root cause"

  post_incident:
    - Schedule postmortem within 24 hours
    - Update runbook with lessons learned
    - Create improvement tickets
    - Test DR process improvements
```

### Cost vs Recovery Time Trade-off

| Strategy | Monthly Cost | RTO | RPO | Complexity | Use Case |
|----------|-------------|-----|-----|------------|----------|
| **Cold Standby** | Low ($) | 4-24 hours | 1-24 hours | Low | Non-critical systems |
| **Warm Standby** | Medium ($$) | 30-60 min | 5-15 min | Medium | Business applications |
| **Hot Standby** | High ($$$) | 1-5 min | < 1 min | High | Critical services |
| **Active-Active** | Very High ($$$$) | 0 (automatic) | 0 | Very High | Mission-critical |

---

## 21.8 Reliability Review Checklist

Use this comprehensive checklist before promoting workloads to production.

### Resource Configuration

- [ ] Resource requests and limits defined for all containers
- [ ] Requests match actual usage (not over/under provisioned)
- [ ] QoS class is appropriate (prefer Guaranteed for critical workloads)
- [ ] No containers requesting more than node capacity
- [ ] Resource quotas configured at namespace level
- [ ] LimitRanges prevent resource abuse

**Verification Commands:**
```bash
# Check resource configuration
kubectl describe pod <pod-name> | grep -A 10 "Requests:"

# Verify QoS class
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'

# Check namespace quotas
kubectl describe resourcequota -n <namespace>
```

### Health Checks

- [ ] Liveness probe configured for all containers
- [ ] Readiness probe configured for all services
- [ ] Startup probe configured for slow-starting apps
- [ ] Probe timeouts are reasonable (not too aggressive)
- [ ] Health check endpoints are lightweight
- [ ] Probes don't depend on external services

**Verification:**
```bash
# Check probe configuration
kubectl get pod <pod-name> -o yaml | grep -A 10 "Probe"
```

### High Availability

- [ ] Minimum 3 replicas for critical services
- [ ] Pod anti-affinity configured (different nodes)
- [ ] Topology spread constraints for multi-AZ
- [ ] PodDisruptionBudget defined
- [ ] Graceful shutdown configured (preStop hooks)
- [ ] terminationGracePeriodSeconds is adequate
- [ ] HPA configured for auto-scaling
- [ ] Multi-region deployment for critical services

**Verification:**
```bash
# Check replica count and distribution
kubectl get pods -l app=<app-name> -o wide

# Verify PDB
kubectl get pdb
```

### Observability

- [ ] Structured logging enabled
- [ ] Logs exported to centralized system
- [ ] Prometheus metrics exposed
- [ ] Key business metrics instrumented
- [ ] Distributed tracing configured
- [ ] Dashboard created in Grafana
- [ ] Alerts defined for critical conditions
- [ ] Runbooks linked to alerts

**Verification:**
```bash
# Check metrics endpoint
kubectl port-forward <pod-name> 9090:9090
curl localhost:9090/metrics

# Verify logging
kubectl logs <pod-name> --tail=10
```

### Security

- [ ] Container runs as non-root user
- [ ] Read-only root filesystem
- [ ] Security context configured
- [ ] Network policies restrict traffic
- [ ] Secrets stored in Secret Manager / Vault
- [ ] Service account follows least privilege
- [ ] Image scanning passed
- [ ] No privileged containers (unless absolutely necessary)
- [ ] Pod Security Standards enforced

**Verification:**
```bash
# Check security context
kubectl get pod <pod-name> -o jsonpath='{.spec.securityContext}'

# Verify network policies
kubectl get networkpolicy -n <namespace>

# Check service account permissions
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<sa-name>
```

### Scalability

- [ ] HPA configured with appropriate metrics
- [ ] VPA configured if using vertical scaling
- [ ] Node auto-scaling enabled
- [ ] Load tested at 2x expected peak
- [ ] Database connection pooling configured
- [ ] Rate limiting implemented
- [ ] Caching strategy in place
- [ ] No hardcoded dependencies on specific pod count

**Verification:**
```bash
# Check HPA status
kubectl get hpa

# Verify scaling behavior
kubectl describe hpa <hpa-name>
```

### Disaster Recovery

- [ ] Backups configured and tested
- [ ] Backup retention policy defined
- [ ] DR runbook documented
- [ ] DR tests scheduled quarterly
- [ ] Multi-region strategy defined
- [ ] RTO/RPO documented
- [ ] Failover procedures tested
- [ ] Data replication configured for stateful services

**Verification:**
```bash
# Check Velero backups
velero backup get

# Verify last successful backup
velero backup describe <backup-name>
```

### Operations

- [ ] Deployment strategy defined (rolling/canary/blue-green)
- [ ] Rollback procedure documented
- [ ] On-call runbook created
- [ ] Incident response process defined
- [ ] Change management process followed
- [ ] Service dependencies mapped
- [ ] SLOs defined and monitored
- [ ] Error budget tracking configured
- [ ] Capacity planning completed

**Final Sign-off:**
```yaml
reviewed_by:
  - name: "SRE Team Lead"
    date: "2026-02-18"
    approved: true

  - name: "Development Lead"
    date: "2026-02-18"
    approved: true

  - name: "Security Team"
    date: "2026-02-18"
    approved: true

production_ready: true
promotion_date: "2026-02-20"
```

---

## Chapter 22 Review Questions

1. **Explain the difference between error budgets and SLOs. How would you use a 99.9% availability SLO to calculate the monthly error budget? What actions would you take if the error budget is 75% consumed halfway through the month?**

2. **Compare Pod anti-affinity with topology spread constraints. In what scenarios would you prefer one over the other? Provide a practical example where topology spread constraints would be insufficient.**

3. **Describe the Kubernetes controller reconciliation loop and how it provides self-healing. What are the limitations of this approach? Give an example of a failure scenario that auto-healing cannot resolve.**

4. **What is the difference between `minAvailable` and `maxUnavailable` in PodDisruptionBudgets? If you have a Deployment with 10 replicas, what would be the practical difference between `minAvailable: 7` and `maxUnavailable: 3`?**

5. **Explain progressive delivery and compare canary deployments with blue/green deployments. What are the resource implications of each strategy? In what situations would you choose one over the other?**

---

## Chapter 22 Hands-On Exercises

**Exercise 1: Implement Complete High Availability**

Deploy a highly available application with all reliability best practices:
```bash
# 1. Create namespace with resource quotas
kubectl create namespace ha-exercise
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ha-quota
  namespace: ha-exercise
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
EOF

# 2. Create a Deployment with:
#    - 5 replicas
#    - Pod anti-affinity
#    - Topology spread across zones
#    - All three probe types
#    - Resource requests and limits
#    - Graceful shutdown

# 3. Add a PodDisruptionBudget ensuring 80% availability

# 4. Configure HPA with CPU and memory metrics

# 5. Test by:
#    - Deleting random pods (verify auto-healing)
#    - Draining a node (verify PDB prevents disruption)
#    - Generating load (verify HPA scales)

# 6. Validate all pods are spread across zones:
kubectl get pods -n ha-exercise -o wide
```

**Exercise 2: Chaos Engineering Experiment**

Run a comprehensive chaos engineering experiment:
```bash
# 1. Install Chaos Mesh
curl -sSL https://mirrors.chaos-mesh.org/v2.6.0/install.sh | bash

# 2. Deploy a test application with monitoring

# 3. Create a chaos experiment that:
#    - Kills 30% of pods every 5 minutes
#    - Adds 100ms network latency
#    - Causes 10% packet loss
#    - Runs for 30 minutes

# 4. Monitor during experiment:
#    - Error rates
#    - Latency (P50, P99)
#    - Pod restarts
#    - HPA scaling behavior

# 5. Document findings:
#    - Did the application remain available?
#    - Were alerts triggered correctly?
#    - What was the user-facing impact?
#    - What improvements are needed?
```

**Exercise 3: Disaster Recovery Drill**

Perform a complete DR test:
```bash
# 1. Deploy an application with Velero backups

# 2. Generate test data (database entries, uploaded files)

# 3. Create a full backup:
velero backup create dr-exercise --include-namespaces=production

# 4. Simulate disaster by deleting namespace:
kubectl delete namespace production

# 5. Restore from backup to new namespace:
velero restore create dr-restore \
  --from-backup dr-exercise \
  --namespace-mappings production:production-restored

# 6. Verify data integrity:
#    - All pods restored?
#    - Data intact?
#    - How long did recovery take?

# 7. Calculate actual RPO and RTO

# 8. Document lessons learned and process improvements
```

---

## Key Takeaways

1. **Reliability is layered** - Build from infrastructure up through platform to application. Each layer depends on the reliability of layers below. Focus on strengthening the foundation first.

2. **Define and measure SLOs** - Service Level Objectives provide clear reliability targets. Use error budgets to balance feature velocity with reliability. When error budget is exhausted, prioritize reliability work.

3. **Design for failure** - Use Pod anti-affinity, topology spread constraints, and PodDisruptionBudgets to ensure workloads survive node and zone failures. Configure graceful shutdown to prevent dropped requests during rolling updates.

4. **Kubernetes auto-heals by design** - Controllers continuously reconcile desired state with actual state. Leverage this with proper health checks, resource limits, and multiple replicas. Understand the limits of auto-healing.

5. **Progressive delivery reduces risk** - Canary and blue/green deployments allow you to test changes with production traffic before full rollout. Automate rollbacks based on metrics and analysis.

6. **Practice chaos engineering** - Proactively inject failures to identify weaknesses before customers do. Start small in non-production, then gradually increase scope. Make chaos experiments part of your regular testing cycle.

7. **Disaster recovery requires testing** - DR plans are worthless without regular testing. Automate DR tests, measure actual RPO/RTO, and continuously improve based on results. The best DR plan is the one you've successfully executed multiple times.

---

[Next Chapter: Kubernetes Troubleshooting Scenarios →](22-troubleshooting-scenarios.md)
