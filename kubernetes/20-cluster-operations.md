# Chapter 20: Kubernetes Cluster Operations

**Prerequisites**: This chapter builds on cluster setup (Chapter 10), monitoring (Chapter 11), and security (Chapter 16).

Operating a production Kubernetes cluster requires deep understanding of control plane health, data management, upgrade strategies, and disaster recovery procedures. This chapter covers the operational aspects that keep clusters running reliably at scale.

---

## 20.1 Control Plane Health

The Kubernetes control plane consists of multiple components that must work together seamlessly. Understanding their architecture and health is critical for cluster operations.

### Control Plane Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Control Plane Node(s)                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐      ┌──────────────┐                     │
│  │  API Server  │◄────►│     etcd     │                     │
│  │   :6443      │      │    :2379     │                     │
│  └──────┬───────┘      └──────────────┘                     │
│         │                                                     │
│         ├──────────┬──────────────┬──────────────┐          │
│         ▼          ▼              ▼              ▼          │
│  ┌────────────┐ ┌──────────┐ ┌─────────┐ ┌──────────────┐  │
│  │ Scheduler  │ │Controller│ │  Cloud  │ │   CoreDNS    │  │
│  │            │ │ Manager  │ │Controller│ │              │  │
│  └────────────┘ └──────────┘ └─────────┘ └──────────────┘  │
│                                                               │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │    Worker Nodes       │
                    │    (kubelet)          │
                    └───────────────────────┘
```

### API Server Health Monitoring

The API server provides built-in health endpoints:

```bash
# Liveness check - is the server running?
curl -k https://localhost:6443/livez?verbose

# Readiness check - is the server ready to serve requests?
curl -k https://localhost:6443/readyz?verbose

# Overall health (deprecated but still useful)
curl -k https://localhost:6443/healthz?verbose
```

Detailed component health:

```bash
# Check individual health checks
curl -k https://localhost:6443/readyz/etcd
curl -k https://localhost:6443/readyz/etcd-readiness
curl -k https://localhost:6443/readyz/informer-sync
curl -k https://localhost:6443/livez/ping
curl -k https://localhost:6443/livez/log
curl -k https://localhost:6443/livez/poststarthook/generic-apiserver-start-informers
```

### Key Control Plane Metrics

| Metric | Description | Healthy Range | Alert Threshold |
|--------|-------------|---------------|-----------------|
| `apiserver_request_duration_seconds` | API request latency | p99 < 1s | p99 > 5s |
| `apiserver_request_total` | Request count by code | - | 5xx rate > 1% |
| `etcd_request_duration_seconds` | etcd operation latency | p99 < 100ms | p99 > 1s |
| `etcd_db_total_size_in_bytes` | etcd database size | < 2GB | > 4GB |
| `scheduler_scheduling_duration_seconds` | Scheduling latency | p99 < 1s | p99 > 10s |
| `workqueue_depth` | Controller work queue depth | < 100 | > 1000 |
| `rest_client_requests_total` | Client request rate | - | Sudden spikes |
| `process_resident_memory_bytes` | Component memory usage | - | > 80% limit |

### Debugging API Server Issues

Check API server logs:

```bash
# For kubeadm clusters
kubectl logs -n kube-system kube-apiserver-<node-name>

# For systemd-managed clusters
journalctl -u kube-apiserver -f

# Check for common issues
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i error
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i "etcd"
kubectl logs -n kube-system kube-apiserver-<node-name> | grep -i "timeout"
```

Verify API server connectivity:

```bash
# Test API server from control plane
curl -k https://localhost:6443/version

# Test from worker node
curl -k https://<api-server-ip>:6443/version \
  --cacert /etc/kubernetes/pki/ca.crt \
  --cert /var/lib/kubelet/pki/kubelet-client-current.pem \
  --key /var/lib/kubelet/pki/kubelet-client-current.pem
```

### Scheduler Health

Monitor scheduler operations:

```bash
# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-<node-name>

# Check for unschedulable pods
kubectl get pods --all-namespaces --field-selector status.phase=Pending

# Describe pending pod to see scheduling failures
kubectl describe pod <pending-pod> -n <namespace>

# Check scheduler metrics
kubectl get --raw /metrics | grep scheduler_
```

### Controller Manager Health

The controller manager runs multiple controllers:

```bash
# Check controller manager logs
kubectl logs -n kube-system kube-controller-manager-<node-name>

# Check for leader election
kubectl logs -n kube-system kube-controller-manager-<node-name> | grep "leader"

# Verify controllers are running
kubectl get --raw /metrics | grep workqueue_depth

# Check for failing controllers
kubectl logs -n kube-system kube-controller-manager-<node-name> | grep -i "error"
```

### Control Plane Monitoring Dashboard

PrometheusRule for control plane alerts:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: control-plane-alerts
  namespace: monitoring
spec:
  groups:
  - name: control-plane
    interval: 30s
    rules:
    - alert: APIServerDown
      expr: up{job="apiserver"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "API Server is down"
        description: "API Server {{ $labels.instance }} has been down for 5 minutes"

    - alert: APIServerHighLatency
      expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket{verb!="WATCH"}[5m])) > 5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "API Server high latency"
        description: "API Server p99 latency is {{ $value }}s"

    - alert: EtcdHighLatency
      expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.25
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd high commit latency"
        description: "etcd p99 commit latency is {{ $value }}s"

    - alert: SchedulerHighLatency
      expr: histogram_quantile(0.99, rate(scheduler_scheduling_duration_seconds_bucket[5m])) > 10
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Scheduler high latency"
        description: "Scheduler p99 latency is {{ $value }}s"
```

---

## 20.2 etcd Operations

etcd is the brain of Kubernetes, storing all cluster state. Proper etcd operations are critical for cluster health.

### Raft Consensus Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    etcd Cluster                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐ │
│  │  etcd-1  │◄───────►│  etcd-2  │◄───────►│  etcd-3  │ │
│  │ (Leader) │  Raft   │(Follower)│  Raft   │(Follower)│ │
│  └────┬─────┘         └──────────┘         └──────────┘ │
│       │                                                   │
│       │ Quorum = (n/2) + 1 = 2 for 3-node cluster       │
│       │                                                   │
└───────┼───────────────────────────────────────────────────┘
        │
        │ Client writes go to leader
        │ Reads can come from any member
        │
        ▼
  ┌────────────┐
  │ API Server │
  └────────────┘

Write Flow:
1. Client → Leader: Write request
2. Leader → Followers: Propose change
3. Followers → Leader: Ack (quorum reached)
4. Leader: Commit entry
5. Leader → Client: Success response
```

### Health Checks

Install etcdctl:

```bash
# Set etcdctl version
export ETCDCTL_API=3

# For TLS-enabled etcd (typical in Kubernetes)
alias etcdctl='etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'
```

Basic health checks:

```bash
# Endpoint health
etcdctl endpoint health --cluster
# https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.3ms
# https://10.0.1.11:2379 is healthy: successfully committed proposal: took = 3.1ms
# https://10.0.1.12:2379 is healthy: successfully committed proposal: took = 2.8ms

# Endpoint status
etcdctl endpoint status --cluster --write-out=table
# +---------------------------+------------------+---------+---------+-----------+
# |         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER |
# +---------------------------+------------------+---------+---------+-----------+
# | https://127.0.0.1:2379   | 8e9e05c52164694d |  3.5.9  |   45 MB |      true |
# | https://10.0.1.11:2379   | 91bc3c398fb3c146 |  3.5.9  |   45 MB |     false |
# | https://10.0.1.12:2379   | fd422379fda50e48 |  3.5.9  |   45 MB |     false |
# +---------------------------+------------------+---------+---------+-----------+

# Member list
etcdctl member list --write-out=table

# Check cluster health metrics
etcdctl endpoint health --cluster --write-out=json | jq
```

### Backup Operations

Critical: Regular backups prevent catastrophic data loss.

```bash
# Create snapshot
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify snapshot
etcdctl snapshot status /backup/etcd-snapshot-20260218-120000.db --write-out=table

# Automated backup script
#!/bin/bash
BACKUP_DIR="/backup/etcd"
RETENTION_DAYS=7

# Create backup
mkdir -p $BACKUP_DIR
etcdctl snapshot save $BACKUP_DIR/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify backup
LATEST_BACKUP=$(ls -t $BACKUP_DIR/etcd-snapshot-*.db | head -1)
etcdctl snapshot status $LATEST_BACKUP

# Remove old backups
find $BACKUP_DIR -name "etcd-snapshot-*.db" -mtime +$RETENTION_DAYS -delete

# Upload to cloud storage
aws s3 cp $LATEST_BACKUP s3://my-etcd-backups/$(basename $LATEST_BACKUP)
```

Kubernetes CronJob for automated backups:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: backup
            image: registry.k8s.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
              find /backup -name "etcd-snapshot-*.db" -mtime +7 -delete
            env:
            - name: ETCDCTL_API
              value: "3"
            - name: ETCDCTL_ENDPOINTS
              value: https://127.0.0.1:2379
            - name: ETCDCTL_CACERT
              value: /etc/kubernetes/pki/etcd/ca.crt
            - name: ETCDCTL_CERT
              value: /etc/kubernetes/pki/etcd/server.crt
            - name: ETCDCTL_KEY
              value: /etc/kubernetes/pki/etcd/server.key
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            hostPath:
              path: /var/backups/etcd
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - effect: NoSchedule
            key: node-role.kubernetes.io/control-plane
```

### Restore Operations

**WARNING**: Restoration destroys current cluster state. Only use in disaster recovery.

```bash
# Stop all API servers first
systemctl stop kube-apiserver

# On each etcd member, stop etcd
systemctl stop etcd

# Restore snapshot on each member
ETCD_NAME=etcd-1
ETCD_INITIAL_CLUSTER="etcd-1=https://10.0.1.10:2380,etcd-2=https://10.0.1.11:2380,etcd-3=https://10.0.1.12:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://10.0.1.10:2380

etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name $ETCD_NAME \
  --initial-cluster $ETCD_INITIAL_CLUSTER \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS \
  --data-dir /var/lib/etcd-restored

# Move old data dir
mv /var/lib/etcd /var/lib/etcd-old
mv /var/lib/etcd-restored /var/lib/etcd

# Start etcd
systemctl start etcd

# Verify cluster health
etcdctl endpoint health --cluster

# Start API servers
systemctl start kube-apiserver
```

### Compaction and Defragmentation

etcd keeps history of all changes. Regular maintenance prevents database bloat.

```bash
# Check current revision
etcdctl endpoint status --write-out=table

# Compact history (keep last 1000 revisions)
rev=$(etcdctl endpoint status --write-out="json" | jq -r '.[] | .Status.header.revision')
etcdctl compact $((rev - 1000))

# Defragment database (reduces physical size)
etcdctl defrag --cluster

# Check size after defrag
etcdctl endpoint status --write-out=table
```

Automated compaction and defrag:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-maintenance
  namespace: kube-system
spec:
  schedule: "0 2 * * 0"  # Weekly at 2 AM Sunday
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: maintenance
            image: registry.k8s.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              # Get current revision
              rev=$(etcdctl endpoint status --write-out="json" | jq -r '.[] | .Status.header.revision')
              echo "Current revision: $rev"

              # Compact (keep last 10000 revisions)
              compact_rev=$((rev - 10000))
              echo "Compacting to revision: $compact_rev"
              etcdctl compact $compact_rev

              # Defragment
              echo "Defragmenting..."
              etcdctl defrag --cluster

              # Show results
              etcdctl endpoint status --write-out=table
            env:
            - name: ETCDCTL_API
              value: "3"
            - name: ETCDCTL_ENDPOINTS
              value: https://127.0.0.1:2379
            - name: ETCDCTL_CACERT
              value: /etc/kubernetes/pki/etcd/ca.crt
            - name: ETCDCTL_CERT
              value: /etc/kubernetes/pki/etcd/server.crt
            - name: ETCDCTL_KEY
              value: /etc/kubernetes/pki/etcd/server.key
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - effect: NoSchedule
            key: node-role.kubernetes.io/control-plane
```

### Performance Tuning

etcd configuration options:

```yaml
# /etc/etcd/etcd.yaml
name: etcd-1
data-dir: /var/lib/etcd
listen-client-urls: https://0.0.0.0:2379
advertise-client-urls: https://10.0.1.10:2379
listen-peer-urls: https://0.0.0.0:2380
initial-advertise-peer-urls: https://10.0.1.10:2380

# Performance tuning
heartbeat-interval: 100           # Default: 100ms
election-timeout: 1000            # Default: 1000ms
snapshot-count: 10000             # Default: 100000 (snapshot after N transactions)
quota-backend-bytes: 8589934592   # 8GB quota (default: 2GB)
max-request-bytes: 1572864        # 1.5MB max request size
grpc-keepalive-min-time: 5s
grpc-keepalive-interval: 2h
grpc-keepalive-timeout: 20s

# Auto compaction
auto-compaction-mode: periodic
auto-compaction-retention: "1h"   # Compact hourly

# Logging
log-level: info
log-outputs:
- /var/log/etcd/etcd.log
```

### Common etcd Issues

**Issue: etcd database too large**

```bash
# Symptoms
etcdctl endpoint status shows DB SIZE > 8GB
API server logs show "etcdserver: mvcc: database space exceeded"

# Resolution
# 1. Increase quota temporarily
etcdctl --endpoints=https://127.0.0.1:2379 alarm disarm

# 2. Compact and defrag
rev=$(etcdctl endpoint status --write-out="json" | jq -r '.[] | .Status.header.revision')
etcdctl compact $((rev - 1000))
etcdctl defrag --cluster

# 3. Increase quota permanently
# Edit /etc/etcd/etcd.yaml: quota-backend-bytes: 8589934592
systemctl restart etcd
```

**Issue: etcd cluster losing quorum**

```bash
# Symptoms
etcdctl endpoint health shows unhealthy members
API server cannot connect to etcd

# Diagnosis
etcdctl member list
etcdctl endpoint status --cluster

# Resolution for failed member
# Remove failed member
etcdctl member remove <member-id>

# On new node, add member
etcdctl member add etcd-3 --peer-urls=https://10.0.1.12:2380

# Start etcd on new node with --initial-cluster-state=existing
```

**Issue: Slow disk I/O affecting etcd**

```bash
# Check disk latency
etcdctl check perf

# Expected output for healthy disk:
# 60 / 60 Boooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo! 100.00%1m0s
# PASS: Throughput is 150 writes/s
# PASS: Slowest request took 0.011s
# PASS: Stddev is 0.002s
# PASS

# Use faster disks (SSD/NVMe)
# Enable disk write caching
# Mount with noatime option
```

---

## 20.3 Cluster Upgrades

Kubernetes upgrades require careful planning to minimize downtime and prevent breaking changes.

### Upgrade Strategies

| Strategy | Downtime | Risk | Use Case |
|----------|----------|------|----------|
| **In-Place Rolling** | Minimal (per node) | Medium | Production, tight budget |
| **Blue-Green Cluster** | None (cutover) | Low | Critical production, can afford 2x resources |
| **Node Pool Rolling** | None | Low | Cloud-managed (GKE, EKS, AKS) |
| **Canary Upgrade** | None | Very Low | Large clusters, risk-averse |

### Pre-Upgrade Checklist

```bash
# 1. Backup etcd
etcdctl snapshot save /backup/pre-upgrade-$(date +%Y%m%d).db

# 2. Check cluster health
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get componentstatuses

# 3. Review API deprecations
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Use pluto to find deprecated APIs
pluto detect-files -d .
pluto detect-helm --helm-version 3

# 4. Check version skew policy
kubectl version --short
# Control plane: v1.28.5
# Nodes: v1.28.3 (must be within 2 minor versions)

# 5. Verify node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# 6. Document current state
kubectl get pods --all-namespaces -o yaml > pre-upgrade-pods.yaml
kubectl get deployments --all-namespaces -o yaml > pre-upgrade-deployments.yaml

# 7. Check for StatefulSets and DaemonSets
kubectl get statefulsets --all-namespaces
kubectl get daemonsets --all-namespaces

# 8. Review custom resources
kubectl api-resources --verbs=list -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found --all-namespaces

# 9. Check admission webhooks
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# 10. Verify monitoring and alerting
# Ensure you have visibility during upgrade
```

### API Deprecation Handling

Version skew policy:

| Component | Supported Version Skew |
|-----------|------------------------|
| kubelet | Control plane N-2 to N |
| kube-proxy | Control plane N-2 to N |
| kubectl | Control plane N-1 to N+1 |
| API Server | N (all same version recommended) |
| Controller Manager | API server N to N-1 |
| Scheduler | API server N to N-1 |
| Cloud Controller Manager | API server N to N-1 |

Common deprecated APIs:

```bash
# Kubernetes 1.25 removed PodSecurityPolicy
# Before upgrade: Migrate to Pod Security Admission or external policy engine

# Kubernetes 1.26 removed HorizontalPodAutoscaler v2beta2
# Find usage
kubectl get hpa --all-namespaces -o yaml | grep "apiVersion: autoscaling/v2beta2"

# Convert to v2
apiVersion: autoscaling/v2  # Changed from v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### In-Place Rolling Upgrade (kubeadm)

Control plane upgrade:

```bash
# On first control plane node
# 1. Check available versions
apt update
apt-cache madison kubeadm

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 3. Verify upgrade plan
kubeadm upgrade plan

# 4. Apply upgrade
kubeadm upgrade apply v1.29.0

# 5. Drain node
kubectl drain <control-plane-node> --ignore-daemonsets --delete-emptydir-data

# 6. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

# 7. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 8. Uncordon node
kubectl uncordon <control-plane-node>

# 9. Verify
kubectl get nodes
kubectl version

# For additional control plane nodes
# On each subsequent control plane node:
kubeadm upgrade node
# Then repeat steps 5-8
```

Worker node upgrade:

```bash
# On control plane, drain worker
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --force

# On worker node
# 1. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 2. Upgrade node
kubeadm upgrade node

# 3. Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

# 4. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# On control plane, uncordon
kubectl uncordon <worker-node>

# Verify
kubectl get nodes
```

### Blue-Green Cluster Upgrade

```bash
# 1. Create new cluster (v1.29) alongside existing (v1.28)
gcloud container clusters create prod-v129 \
  --cluster-version 1.29 \
  --region us-central1 \
  --num-nodes 3

# 2. Deploy applications to new cluster
kubectl --context=prod-v129 apply -f ./manifests/

# 3. Test new cluster
kubectl --context=prod-v129 get pods --all-namespaces
# Run smoke tests

# 4. Migrate persistent data
# For stateful apps, migrate data before cutover
velero backup create pre-cutover --include-namespaces=production
velero restore create --from-backup pre-cutover --namespace-mappings production:production

# 5. Switch traffic (DNS/Load Balancer)
# Update DNS to point to new cluster
# Or update LoadBalancer to new cluster

# 6. Monitor new cluster
# Watch metrics, logs, alerts

# 7. Keep old cluster for rollback window (24-48h)
# After successful cutover
gcloud container clusters delete prod-v128
```

### Rollback Procedures

If upgrade fails:

```bash
# For kubeadm clusters
# 1. Restore etcd from backup
systemctl stop kube-apiserver
systemctl stop etcd

etcdctl snapshot restore /backup/pre-upgrade-20260218.db \
  --data-dir=/var/lib/etcd-rollback

mv /var/lib/etcd /var/lib/etcd-failed
mv /var/lib/etcd-rollback /var/lib/etcd

systemctl start etcd
systemctl start kube-apiserver

# 2. Downgrade control plane components
apt-mark unhold kubeadm kubelet kubectl
apt-get install -y kubeadm=1.28.5-00 kubelet=1.28.5-00 kubectl=1.28.5-00
apt-mark hold kubeadm kubelet kubectl

systemctl restart kubelet

# 3. Downgrade worker nodes
# On each worker
apt-get install -y kubelet=1.28.5-00 kubectl=1.28.5-00
systemctl restart kubelet

# For cloud-managed clusters
# GKE - can't rollback, must restore from backup or recreate
# EKS - can't rollback, must create new cluster
# AKS - can't rollback, must create new cluster
```

For blue-green, rollback is simple:

```bash
# Switch traffic back to old cluster
# Update DNS or LoadBalancer
```

---

## 20.4 Node Management

Proper node management ensures workloads run reliably while allowing maintenance operations.

### Cordon, Drain, Uncordon Workflow

```
┌─────────────────────────────────────────────────────────┐
│              Node Maintenance Workflow                   │
└─────────────────────────────────────────────────────────┘

1. CORDON - Mark node unschedulable
   ┌──────────┐  kubectl cordon node-1  ┌──────────────────┐
   │  Node-1  │ ────────────────────────►│  Node-1 (NoSch) │
   │          │                           │  Existing Pods   │
   │ Ready    │                           │  New Pods ✗      │
   └──────────┘                           └──────────────────┘

2. DRAIN - Evict all pods gracefully
   ┌──────────────────┐  kubectl drain node-1  ┌────────────┐
   │  Node-1 (NoSch)  │ ───────────────────────►│  Node-1    │
   │  [Pod1][Pod2]    │  --ignore-daemonsets    │  Empty     │
   │  [Pod3][Pod4]    │  --delete-emptydir-data │  (DaemonS) │
   └──────────────────┘                          └────────────┘
            │
            │ Pods rescheduled to other nodes
            ▼
   ┌─────────────────┐
   │ Node-2   Node-3 │
   │ [Pod1]   [Pod3] │
   │ [Pod2]   [Pod4] │
   └─────────────────┘

3. MAINTENANCE - Perform node operations
   - OS patches
   - Kernel upgrades
   - Hardware maintenance
   - Kubelet upgrades

4. UNCORDON - Mark node schedulable again
   ┌────────────┐  kubectl uncordon node-1  ┌──────────┐
   │  Node-1    │ ─────────────────────────►│  Node-1  │
   │  Empty     │                            │  Ready   │
   │  (NoSch)   │                            │  Accept  │
   └────────────┘                            └──────────┘
```

Commands:

```bash
# Cordon - prevent new pods from scheduling
kubectl cordon node-1

# Check node status
kubectl get nodes
# NAME     STATUS                     ROLES    AGE   VERSION
# node-1   Ready,SchedulingDisabled   <none>   10d   v1.28.5

# Drain - evict all pods
kubectl drain node-1 \
  --ignore-daemonsets \           # DaemonSets can't be evicted
  --delete-emptydir-data \        # Delete pods using emptyDir
  --force \                        # Force delete pods not managed by controller
  --grace-period=300              # Allow 5 minutes for graceful shutdown

# Drain with PodDisruptionBudget respect
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --disable-eviction              # Use delete instead of eviction API

# Perform maintenance
# ... upgrade, patch, reboot ...

# Uncordon - allow scheduling again
kubectl uncordon node-1

# Verify
kubectl get nodes
# NAME     STATUS   ROLES    AGE   VERSION
# node-1   Ready    <none>   10d   v1.28.5
```

### PodDisruptionBudget Design

PodDisruptionBudgets (PDBs) protect application availability during voluntary disruptions:

```yaml
# Minimum available approach
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2              # At least 2 pods must remain available
  selector:
    matchLabels:
      app: myapp
---
# Maximum unavailable approach
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  maxUnavailable: 1            # At most 1 pod can be unavailable
  selector:
    matchLabels:
      app: myapp
---
# Percentage-based
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  maxUnavailable: 25%          # At most 25% of pods unavailable
  selector:
    matchLabels:
      app: myapp
---
# For single-pod deployments
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: single-pod-app-pdb
spec:
  maxUnavailable: 0            # Can't evict any pods
  selector:
    matchLabels:
      app: single-pod-app

# This blocks drain! Use only when absolutely necessary
# Alternative: minAvailable: 1
```

Best practices:

```yaml
# For stateless apps with 5 replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx:1.21
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 3              # 60% availability during disruption
  selector:
    matchLabels:
      app: web-app
---
# For critical services
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-service-pdb
spec:
  minAvailable: 4              # Only 1 pod can be disrupted at a time
  selector:
    matchLabels:
      app: critical-service
      tier: critical
```

Check PDB status:

```bash
# List PDBs
kubectl get pdb --all-namespaces

# Describe PDB
kubectl describe pdb myapp-pdb
# Status:
#   Current Number:    5
#   Desired Number:    3
#   Disruptions Allowed: 2
#   Expected Pods:     5
#   Observed Generation: 1

# Check if PDB is blocking drain
kubectl get pdb -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.disruptionsAllowed}{"\n"}{end}'
```

### Node Problem Detection

Node Problem Detector identifies hardware/kernel issues:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-problem-detector
        image: registry.k8s.io/node-problem-detector/node-problem-detector:v0.8.12
        securityContext:
          privileged: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: log
          mountPath: /var/log
          readOnly: true
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        - name: config
          mountPath: /config
          readOnly: true
        command:
        - /node-problem-detector
        - --logtostderr
        - --config.system-log-monitor=/config/kernel-monitor.json
        - --config.system-log-monitor=/config/docker-monitor.json
        - --config.custom-plugin-monitor=/config/custom-plugin-monitor.json
      volumes:
      - name: log
        hostPath:
          path: /var/log/
      - name: kmsg
        hostPath:
          path: /dev/kmsg
      - name: localtime
        hostPath:
          path: /etc/localtime
      - name: config
        configMap:
          name: node-problem-detector-config
      tolerations:
      - effect: NoSchedule
        operator: Exists
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-problem-detector-config
  namespace: kube-system
data:
  kernel-monitor.json: |
    {
      "plugin": "kmsg",
      "logPath": "/dev/kmsg",
      "lookback": "5m",
      "bufferSize": 10,
      "source": "kernel-monitor",
      "conditions": [
        {
          "type": "KernelDeadlock",
          "reason": "KernelHasNoDeadlock",
          "message": "kernel has no deadlock"
        },
        {
          "type": "ReadonlyFilesystem",
          "reason": "FilesystemIsReadOnly",
          "message": "Filesystem is read-only"
        }
      ],
      "rules": [
        {
          "type": "temporary",
          "reason": "OOMKilling",
          "pattern": "Kill process \\d+ (.+) score \\d+ or sacrifice child\\nKilled process \\d+ (.+) total-vm:\\d+kB, anon-rss:\\d+kB, file-rss:\\d+kB.*"
        },
        {
          "type": "temporary",
          "reason": "TaskHung",
          "pattern": "task \\S+:\\w+ blocked for more than \\w+ seconds\\."
        },
        {
          "type": "permanent",
          "condition": "ReadonlyFilesystem",
          "reason": "FilesystemIsReadOnly",
          "pattern": "Remounting filesystem read-only"
        }
      ]
    }
```

Check node conditions:

```bash
# View node conditions
kubectl describe node node-1 | grep -A 20 Conditions

# Node conditions added by NPD:
# KernelDeadlock     False
# ReadonlyFilesystem False
# DiskPressure       False
# MemoryPressure     False
# PIDPressure        False
```

### Cluster Autoscaler

Automatically adjusts cluster size based on pending pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - name: cluster-autoscaler
        image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=gce
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=mig:name=node-pool
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
        - --scale-down-enabled=true
        - --scale-down-delay-after-add=10m
        - --scale-down-unneeded-time=10m
        - --scale-down-utilization-threshold=0.5
        - --max-node-provision-time=15m
        env:
        - name: AWS_REGION
          value: us-west-2
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
- apiGroups: [""]
  resources: ["events", "endpoints"]
  verbs: ["create", "patch"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/status"]
  verbs: ["update"]
- apiGroups: [""]
  resources: ["endpoints"]
  resourceNames: ["cluster-autoscaler"]
  verbs: ["get", "update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch", "list", "get", "update"]
- apiGroups: [""]
  resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["extensions"]
  resources: ["replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["policy"]
  resources: ["poddisruptionbudgets"]
  verbs: ["watch", "list"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "replicasets", "daemonsets"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
  verbs: ["watch", "list", "get"]
- apiGroups: ["batch", "extensions"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "patch"]
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["create"]
- apiGroups: ["coordination.k8s.io"]
  resourceNames: ["cluster-autoscaler"]
  resources: ["leases"]
  verbs: ["get", "update"]
```

### Karpenter (Modern Alternative)

Karpenter provisions nodes more efficiently:

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["spot", "on-demand"]
  - key: kubernetes.io/arch
    operator: In
    values: ["amd64"]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30          # Scale down after 30s empty
  ttlSecondsUntilExpired: 604800    # Recycle nodes after 7 days
  consolidation:
    enabled: true
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: my-cluster
  securityGroupSelector:
    karpenter.sh/discovery: my-cluster
  instanceProfile: KarpenterNodeInstanceProfile
  amiFamily: AL2
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 100Gi
      volumeType: gp3
      encrypted: true
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh my-cluster
```

---

## 20.5 Capacity Planning & Resource Management

Right-sizing workloads prevents waste and improves cluster efficiency.

### Requests vs Limits

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:               # Minimum guaranteed resources
            cpu: 100m             # 0.1 CPU cores
            memory: 128Mi         # 128 MiB RAM
          limits:                 # Maximum allowed resources
            cpu: 500m             # 0.5 CPU cores
            memory: 512Mi         # 512 MiB RAM

# Quality of Service classes:
# 1. Guaranteed: requests == limits (both CPU and memory)
# 2. Burstable: requests < limits OR only requests set
# 3. BestEffort: No requests or limits set
```

QoS comparison:

| QoS Class | CPU Behavior | Memory Behavior | OOMKilled Priority | Use Case |
|-----------|--------------|-----------------|-------------------|----------|
| **Guaranteed** | Always gets requested CPU | Always gets memory | Last | Critical services |
| **Burstable** | Gets requested CPU, can burst to limit | Gets requested memory, can burst to limit | Middle | Most workloads |
| **BestEffort** | Uses leftover CPU | Uses leftover memory | First | Batch jobs, non-critical |

### Vertical Pod Autoscaler (VPA)

VPA recommends optimal resource requests:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"            # Auto, Initial, Recreate, or Off
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits  # Or RequestsOnly
```

VPA modes:

- **Off**: Only provide recommendations (no changes)
- **Initial**: Set requests on pod creation only
- **Recreate**: Update requests and restart pods
- **Auto**: Same as Recreate (no in-place updates yet)

Install VPA:

```bash
# Clone VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Check VPA recommendations
kubectl describe vpa myapp-vpa

# Status:
#   Recommendation:
#     Container Recommendations:
#       Container Name:  app
#       Lower Bound:
#         Cpu:     75m
#         Memory:  128Mi
#       Target:
#         Cpu:     150m
#         Memory:  256Mi
#       Upper Bound:
#         Cpu:     500m
#         Memory:  512Mi
```

### Goldilocks (VPA Analysis Tool)

```bash
# Install Goldilocks
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install goldilocks fairwinds-stable/goldilocks --namespace goldilocks --create-namespace

# Enable VPA for namespace
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Port-forward to dashboard
kubectl port-forward -n goldilocks svc/goldilocks-dashboard 8080:80

# Open http://localhost:8080 to see recommendations
```

### Namespace Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"              # Total CPU requests
    requests.memory: 20Gi           # Total memory requests
    limits.cpu: "20"                # Total CPU limits
    limits.memory: 40Gi             # Total memory limits
    persistentvolumeclaims: "10"    # Max PVCs
    requests.storage: 100Gi         # Total storage requests
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota
  namespace: development
spec:
  hard:
    pods: "50"                      # Max pods
    services: "10"                  # Max services
    services.loadbalancers: "2"     # Max LoadBalancer services
    services.nodeports: "5"         # Max NodePort services
    configmaps: "20"                # Max ConfigMaps
    secrets: "20"                   # Max Secrets
---
# Quota with scopes
apiVersion: v1
kind: ResourceQuota
metadata:
  name: best-effort-quota
  namespace: batch-jobs
spec:
  hard:
    pods: "10"                      # Max BestEffort pods
  scopes:
  - BestEffort
---
# Priority-based quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: high-priority-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
  scopeSelector:
    matchExpressions:
    - operator: In
      scopeName: PriorityClass
      values: ["high-priority"]
```

Check quota usage:

```bash
# View quota
kubectl get resourcequota -n development

# Describe quota with usage
kubectl describe resourcequota compute-quota -n development

# Name:            compute-quota
# Namespace:       development
# Resource         Used   Hard
# --------         ----   ----
# limits.cpu       8      20
# limits.memory    16Gi   40Gi
# requests.cpu     4      10
# requests.memory  8Gi    20Gi
```

### LimitRange

Set default and enforce min/max resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: development
spec:
  limits:
  # Container limits
  - type: Container
    default:                          # Default limits
      cpu: 500m
      memory: 512Mi
    defaultRequest:                   # Default requests
      cpu: 100m
      memory: 128Mi
    min:                              # Minimum allowed
      cpu: 50m
      memory: 64Mi
    max:                              # Maximum allowed
      cpu: 2
      memory: 2Gi
    maxLimitRequestRatio:             # Max limit/request ratio
      cpu: 4
      memory: 4

  # Pod limits
  - type: Pod
    max:
      cpu: 4
      memory: 8Gi

  # PVC limits
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 100Gi
```

### Cost Optimization Strategies

**1. Use Spot/Preemptible Nodes**

```yaml
apiVersion: v1
kind: Node
metadata:
  labels:
    node.kubernetes.io/lifecycle: spot
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 10
  template:
    spec:
      nodeSelector:
        node.kubernetes.io/lifecycle: spot
      tolerations:
      - key: "node.kubernetes.io/lifecycle"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      containers:
      - name: processor
        image: batch-processor:1.0
```

**2. Bin Packing with Priority**

```yaml
# High priority workload - gets scheduled first
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority workloads"
---
# Low priority workload - fills gaps
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority batch workloads"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: api
        image: api:1.0
---
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processing
spec:
  template:
    spec:
      priorityClassName: low-priority
      containers:
      - name: processor
        image: processor:1.0
```

**3. Node Affinity for Cost Zones**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cost-sensitive-app
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                - t3.medium        # Cheaper instance type
                - t3.large
          - weight: 50
            preference:
              matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-west-2a       # Cheaper availability zone
```

---

## 20.6 Backup & Disaster Recovery

Comprehensive backup strategy prevents data loss and enables quick recovery.

### Velero Architecture

```
┌───────────────────────────────────────────────────────────┐
│                  Velero Backup Flow                        │
└───────────────────────────────────────────────────────────┘

   kubectl                    ┌──────────────────┐
   velero backup create  ────►│  Velero Server   │
                               │  (Deployment)    │
                               └────────┬─────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
            │ Kubernetes   │    │  Restic     │    │   Backup    │
            │ API Server   │    │  Daemon     │    │   Storage   │
            │              │    │  (DaemonSet)│    │   (S3/GCS)  │
            └──────────────┘    └─────────────┘    └─────────────┘
                    │                   │                   │
                    │ 1. Query resources│                   │
                    │◄──────────────────┤                   │
                    │                   │                   │
                    │ 2. Return objects │                   │
                    ├──────────────────►│                   │
                    │                   │                   │
                    │                   │ 3. Backup volumes │
                    │                   ├──────────────────►│
                    │                   │                   │
                    │                   │ 4. Upload metadata│
                    │                   │◄──────────────────│
                    │                   │                   │
                    └───────────────────┴───────────────────┘

Backed up:
- Kubernetes objects (YAML manifests)
- Persistent volumes (via Restic or CSI snapshots)
- Resource relationships
- Namespace-scoped and cluster-scoped resources
```

### Velero Installation

```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# Install Velero in cluster (AWS example)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-west-2 \
  --snapshot-location-config region=us-west-2 \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --uploader-type restic

# For GCP
velero install \
  --provider gcp \
  --plugins velero/velero-plugin-for-gcp:v1.8.0 \
  --bucket my-velero-backups \
  --secret-file ./credentials-velero

# Verify installation
kubectl get pods -n velero
```

### Backup Procedures

```bash
# Full cluster backup
velero backup create full-backup --wait

# Backup specific namespaces
velero backup create prod-backup --include-namespaces production,database

# Backup with labels
velero backup create app-backup --selector app=myapp

# Exclude resources
velero backup create backup-no-secrets \
  --exclude-resources secrets,configmaps

# Include persistent volumes
velero backup create pv-backup \
  --default-volumes-to-restic \
  --include-namespaces production

# Scheduled backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h0m0s

# Weekly full backup, retain 4 weeks
velero schedule create weekly-full \
  --schedule="0 2 * * 0" \
  --ttl 672h

# Describe backup
velero backup describe full-backup

# View backup contents
velero backup describe full-backup --details

# Download backup locally
velero backup download full-backup

# Check backup logs
velero backup logs full-backup
```

### Restore Procedures

```bash
# List backups
velero backup get

# Restore full backup
velero restore create --from-backup full-backup --wait

# Restore to different namespace
velero restore create --from-backup prod-backup \
  --namespace-mappings production:production-restore

# Restore specific resources
velero restore create --from-backup full-backup \
  --include-resources deployments,services \
  --include-namespaces production

# Restore without persistent volumes
velero restore create --from-backup full-backup \
  --restore-volumes=false

# Partial restore with label selector
velero restore create --from-backup app-backup \
  --selector app=myapp

# Check restore status
velero restore describe <restore-name>

# View restore logs
velero restore logs <restore-name>
```

### etcd Snapshot Backup

```bash
# Manual snapshot
ETCDCTL_API=3 etcdctl snapshot save \
  /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-20260218.db --write-out=table

# Automated backup with restic
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE="/tmp/etcd-snapshot-${BACKUP_DATE}.db"

# Create snapshot
etcdctl snapshot save $SNAPSHOT_FILE

# Upload to restic repository
restic -r s3:s3.amazonaws.com/my-etcd-backups backup $SNAPSHOT_FILE

# Cleanup local snapshot
rm $SNAPSHOT_FILE

# Prune old backups (keep last 30 daily, 12 monthly)
restic -r s3:s3.amazonaws.com/my-etcd-backups forget \
  --keep-daily 30 \
  --keep-monthly 12 \
  --prune
```

### GitOps Recovery

With GitOps (ArgoCD/Flux), recovery is simplified:

```bash
# All manifests are in Git
# Disaster recovery steps:

# 1. Recreate cluster
gcloud container clusters create prod-recovery \
  --cluster-version 1.28 \
  --region us-central1

# 2. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Apply root app-of-apps
kubectl apply -f bootstrap/root-app.yaml

# 4. ArgoCD syncs all applications from Git
argocd app sync --all

# Result: Entire cluster state restored from Git
# Persistent data still needs Velero restore
```

### What to Backup

| Component | Method | Frequency | Retention |
|-----------|--------|-----------|-----------|
| **etcd** | etcdctl snapshot | Hourly | 24 hours |
| **Kubernetes objects** | Velero | Daily | 30 days |
| **Persistent volumes** | Velero + Restic/CSI | Daily | 30 days |
| **Namespace configs** | GitOps | Continuous | Indefinite (Git) |
| **Secrets (encrypted)** | Sealed Secrets in Git | Continuous | Indefinite |
| **Helm releases** | Helm chart source in Git | Continuous | Indefinite |
| **Custom resources** | Velero | Daily | 30 days |
| **Cluster configuration** | Terraform/IaC in Git | Continuous | Indefinite |

### Disaster Recovery Architecture

```
┌──────────────────────────────────────────────────────────────┐
│             Multi-Region DR Architecture                      │
└──────────────────────────────────────────────────────────────┘

Primary Region (us-west-2)              DR Region (us-east-1)
┌─────────────────────────┐            ┌─────────────────────────┐
│  Production Cluster     │            │  Standby Cluster        │
│                         │            │  (Cold/Warm)            │
│  ┌──────────────────┐   │            │  ┌──────────────────┐   │
│  │  Applications    │   │            │  │  Applications    │   │
│  │  (Active)        │   │            │  │  (Standby)       │   │
│  └──────────────────┘   │            │  └──────────────────┘   │
│                         │            │                         │
│  ┌──────────────────┐   │            │  ┌──────────────────┐   │
│  │  etcd            │   │  Backup    │  │  etcd            │   │
│  │  Data            │   │────────────┼─►│  (Restored)      │   │
│  └──────────────────┘   │            │  └──────────────────┘   │
└──────────┬──────────────┘            └──────────┬──────────────┘
           │                                      │
           │                                      │
           ▼                                      ▼
    ┌────────────────┐                    ┌────────────────┐
    │  Velero        │  Cross-Region      │  Velero        │
    │  Backups       │  Replication       │  Backups       │
    │  (S3 us-west)  │───────────────────►│  (S3 us-east)  │
    └────────────────┘                    └────────────────┘
           │
           │ Replicate
           ▼
    ┌────────────────┐                    ┌────────────────┐
    │  GitOps Repo   │  Git Mirror        │  GitOps Repo   │
    │  (GitHub)      │───────────────────►│  (Mirror)      │
    └────────────────┘                    └────────────────┘

RTO (Recovery Time Objective):
- Cold DR: 4-8 hours
- Warm DR: 1-2 hours
- Hot DR (active-active): Minutes

RPO (Recovery Point Objective):
- With hourly backups: ~1 hour data loss
- With continuous replication: ~minutes data loss
```

---

## 20.7 Multi-Cluster Operations

Managing multiple clusters requires centralized tooling and processes.

### Multi-Cluster Use Cases

1. **Environment Separation**: dev, staging, production
2. **Geographic Distribution**: Regional clusters for latency
3. **Compliance**: Separate clusters per regulatory region
4. **Tenant Isolation**: One cluster per customer/team
5. **Blue-Green Deployments**: Parallel cluster versions
6. **Cost Optimization**: Mix on-prem and cloud clusters

### Multi-Cluster Service Mesh

Istio multi-cluster setup:

```yaml
# Install Istio on both clusters with multi-cluster enabled

# Cluster 1 (us-west)
istioctl install --set values.global.multiCluster.clusterName=cluster-west

# Cluster 2 (us-east)
istioctl install --set values.global.multiCluster.clusterName=cluster-east

# Enable endpoint discovery between clusters
# On cluster-west, create remote secret for cluster-east
istioctl create-remote-secret \
  --context=cluster-east \
  --name=cluster-east | \
  kubectl apply -f - --context=cluster-west

# On cluster-east, create remote secret for cluster-west
istioctl create-remote-secret \
  --context=cluster-west \
  --name=cluster-west | \
  kubectl apply -f - --context=cluster-east
```

Service exposed across clusters:

```yaml
# Deployed in both clusters
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  ports:
  - port: 8080
    name: http
  selector:
    app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
---
# ServiceEntry for cross-cluster discovery
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: myapp-multicluster
  namespace: production
spec:
  hosts:
  - myapp.production.svc.cluster.local
  location: MESH_INTERNAL
  ports:
  - number: 8080
    name: http
    protocol: HTTP
  resolution: DNS
```

### Centralized Monitoring with Thanos

```yaml
# Prometheus with Thanos sidecar in each cluster
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      external_labels:
        cluster: cluster-west
        region: us-west-2

    remote_write:
    - url: http://thanos-receive:19291/api/v1/receive
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: prometheus
  namespace: monitoring
spec:
  serviceName: prometheus
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      # Prometheus container
      - name: prometheus
        image: prom/prometheus:v2.45.0
        args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        - --storage.tsdb.retention.time=6h
        - --web.enable-lifecycle
        - --storage.tsdb.min-block-duration=2h
        - --storage.tsdb.max-block-duration=2h
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: data
          mountPath: /prometheus

      # Thanos sidecar
      - name: thanos-sidecar
        image: quay.io/thanos/thanos:v0.32.0
        args:
        - sidecar
        - --tsdb.path=/prometheus
        - --prometheus.url=http://localhost:9090
        - --objstore.config-file=/etc/thanos/objstore.yml
        - --grpc-address=0.0.0.0:10901
        - --http-address=0.0.0.0:10902
        ports:
        - containerPort: 10901
          name: grpc
        - containerPort: 10902
          name: http
        volumeMounts:
        - name: data
          mountPath: /prometheus
        - name: thanos-objstore
          mountPath: /etc/thanos

      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: thanos-objstore
        secret:
          secretName: thanos-objstore-config

  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
---
# Thanos Query (central)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: quay.io/thanos/thanos:v0.32.0
        args:
        - query
        - --http-address=0.0.0.0:10902
        - --grpc-address=0.0.0.0:10901
        - --store=dnssrv+_grpc._tcp.thanos-sidecar-cluster-west.monitoring.svc.cluster.local
        - --store=dnssrv+_grpc._tcp.thanos-sidecar-cluster-east.monitoring.svc.cluster.local
        - --store=dnssrv+_grpc._tcp.thanos-store.monitoring.svc.cluster.local
        ports:
        - containerPort: 10902
          name: http
        - containerPort: 10901
          name: grpc
```

### GitOps Multi-Cluster (ArgoCD)

```yaml
# ApplicationSet for multi-cluster deployment
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-multicluster
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: cluster-west
        url: https://cluster-west.example.com
        region: us-west-2
      - cluster: cluster-east
        url: https://cluster-east.example.com
        region: us-east-1

  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: production
      source:
        repoURL: https://github.com/myorg/apps
        targetRevision: main
        path: apps/myapp
        helm:
          values: |
            cluster: {{cluster}}
            region: {{region}}
            replicas: 5
      destination:
        server: '{{url}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### Fleet Management Comparison

| Tool | Features | Best For | Complexity |
|------|----------|----------|------------|
| **kubectl contexts** | Native, simple | Small fleets (<5 clusters) | Low |
| **Rancher** | UI, RBAC, multi-tenant | Enterprise with diverse teams | Medium |
| **ArgoCD ApplicationSets** | GitOps, declarative | Cloud-native, K8s-savvy teams | Medium |
| **Flux** | GitOps, lightweight | Simple GitOps workflows | Low-Medium |
| **Crossplane** | Control plane, infrastructure | Platform teams, multi-cloud | High |
| **GKE Hub/EKS Anywhere** | Cloud-native | Same cloud provider | Low |

Multi-context kubectl:

```bash
# List contexts
kubectl config get-contexts

# Switch context
kubectl config use-context cluster-west

# Query all clusters
for ctx in cluster-west cluster-east cluster-central; do
  echo "=== $ctx ==="
  kubectl --context=$ctx get nodes
done

# kubectx for easier switching
kubectx cluster-west
kubectx cluster-east

# kubectl with multiple contexts
kubectl get pods --context=cluster-west -n production
kubectl get pods --context=cluster-east -n production
```

---

## 20.8 Production Runbooks

Standard operating procedures for common issues.

### Runbook: Node NotReady

**Symptoms:**
- `kubectl get nodes` shows node in NotReady state
- Pods not scheduling to affected node
- Alerts: NodeNotReady, NodeStatusUnknown

**Diagnosis:**

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq

# SSH to node and check kubelet
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 100

# Check disk space
df -h

# Check memory
free -h

# Check for OOM kills
dmesg | grep -i "out of memory"

# Check network connectivity to API server
curl -k https://<api-server>:6443/healthz

# Check CNI plugin
ls -la /etc/cni/net.d/
ls -la /opt/cni/bin/
```

**Resolution:**

```bash
# If disk full
# Clean up old images
docker system prune -a

# Clean up old logs
journalctl --vacuum-time=1d

# If kubelet stopped
systemctl restart kubelet

# If network issue
# Restart CNI plugin
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
kubectl delete pod -n kube-system -l k8s-app=calico-node

# If corrupted kubelet config
# Restore from backup or regenerate
kubeadm init phase kubelet-start

# If node needs reboot
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
ssh <node> 'sudo reboot'
# Wait for reboot
kubectl uncordon <node-name>
```

**Prevention:**
- Monitor disk space (alert at 80%)
- Use log rotation (`journald` vacuum)
- Enable kubelet auto-restart (`systemd` restart policy)
- Monitor node metrics continuously
- Use Node Problem Detector

### Runbook: etcd High Latency

**Symptoms:**
- API server slow to respond
- `kubectl` commands timeout
- Alerts: EtcdHighLatency, EtcdSlowRequests
- etcd logs show slow disk writes

**Diagnosis:**

```bash
# Check etcd latency metrics
etcdctl endpoint status --cluster --write-out=table

# Check disk performance
etcdctl check perf

# Check etcd logs
kubectl logs -n kube-system etcd-<control-plane-node>

# Check disk I/O
iostat -x 1 10
iotop

# Check etcd database size
etcdctl endpoint status --write-out=json | jq '.[] | .Status.dbSize'

# Check network latency between etcd members
for member in etcd-1 etcd-2 etcd-3; do
  ping -c 5 $member
done
```

**Resolution:**

```bash
# If database too large - compact and defrag
rev=$(etcdctl endpoint status --write-out="json" | jq -r '.[] | .Status.header.revision')
etcdctl compact $((rev - 1000))
etcdctl defrag --cluster

# If slow disk - migrate to faster disk
# 1. Backup etcd
etcdctl snapshot save /backup/etcd-pre-migration.db

# 2. Stop etcd
systemctl stop etcd

# 3. Move data to faster disk
rsync -av /var/lib/etcd/ /mnt/nvme/etcd/

# 4. Update etcd config to use new path
# Edit /etc/etcd/etcd.yaml: data-dir: /mnt/nvme/etcd

# 5. Start etcd
systemctl start etcd

# If high network latency - check network
# Verify etcd members can reach each other
etcdctl member list

# If split brain - restore from backup
# See section 20.2 for restore procedures
```

**Prevention:**
- Use SSD/NVMe for etcd storage
- Enable auto-compaction (--auto-compaction-retention=1h)
- Monitor etcd metrics continuously
- Set up alerts for p99 latency > 100ms
- Regular defragmentation (weekly)
- Ensure low network latency (<10ms) between etcd members

### Runbook: API Server Overloaded

**Symptoms:**
- `kubectl` commands slow or timeout
- High API server CPU/memory
- Alerts: APIServerHighLatency, APIServerRequestsHigh
- API server logs show rate limiting

**Diagnosis:**

```bash
# Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_total
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# Check top API clients
kubectl get --raw /metrics | grep apiserver_audit_requests_rejected_total

# Check API server resource usage
kubectl top pod -n kube-system kube-apiserver-*

# Check for slow requests
kubectl logs -n kube-system kube-apiserver-<node> | grep "Trace\[" | grep "ms"

# Check for large list requests
kubectl logs -n kube-system kube-apiserver-<node> | grep "LIST" | grep -v "resourceVersion=0"

# Identify noisy clients
kubectl get --raw /metrics | grep 'apiserver_request_total{client="[^"]*"' | sort -t= -k2 -nr | head -20
```

**Resolution:**

```bash
# If caused by watch storms
# Identify and fix misbehaving controllers
kubectl logs <controller-pod> | grep -i "watch"

# If caused by expensive LIST operations
# Enable API priority and fairness
# Edit API server manifest:
# --enable-priority-and-fairness=true

# If insufficient resources
# Increase API server replicas (for HA setup)
kubectl scale deployment -n kube-system kube-apiserver --replicas=3

# If too many objects
# Clean up unused resources
kubectl get all --all-namespaces | wc -l

# Archive old resources
velero backup create cleanup-backup
kubectl delete namespace <old-namespace>

# If authentication storm
# Enable authentication caching
# --authentication-token-webhook-cache-ttl=2m

# Enable audit log sampling (if audit logging enabled)
# --audit-log-maxage=7
```

**Prevention:**
- Set API request rate limits
- Use resource quotas per namespace
- Enable API priority and fairness
- Monitor API server resource usage
- Use efficient list operations (field selectors, label selectors)
- Implement client-side rate limiting in custom controllers
- Regular cleanup of unused resources

### Runbook: Certificate Expiry

**Symptoms:**
- API server authentication failures
- kubelet cannot connect to API server
- Alerts: CertificateExpiring
- TLS handshake errors in logs

**Diagnosis:**

```bash
# Check certificate expiration dates
kubeadm certs check-expiration

# Manual certificate check
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 2 Validity

# Check all certificates
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -dates
done

# Check kubelet certificates
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Check certificate SANs
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A 1 "Subject Alternative Name"
```

**Resolution:**

```bash
# Renew all kubeadm-managed certificates
kubeadm certs renew all

# Verify renewal
kubeadm certs check-expiration

# Restart control plane components
# For static pods
kubectl -n kube-system delete pod kube-apiserver-<node>
kubectl -n kube-system delete pod kube-controller-manager-<node>
kubectl -n kube-system delete pod kube-scheduler-<node>

# Or restart kubelet
systemctl restart kubelet

# Update kubeconfig with new certs
kubeadm init phase kubeconfig all

# Copy new admin.conf
cp /etc/kubernetes/admin.conf ~/.kube/config

# Renew specific certificate
kubeadm certs renew apiserver

# For kubelet client certificates
# kubelet auto-rotates certificates if configured
# Check /var/lib/kubelet/config.yaml:
# rotateCertificates: true
```

**Prevention:**
- Enable automatic certificate rotation (kubelet)
- Set up monitoring for certificate expiration (alert 30 days before)
- Use cert-manager for application certificates
- Document certificate renewal procedures
- Set calendar reminders for manual renewals
- Use longer certificate validity periods (kubeadm default: 1 year)

### Runbook: PVC Full

**Symptoms:**
- Application errors: "No space left on device"
- Pod status: CrashLoopBackOff
- Alerts: PersistentVolumeFillingUp
- Application cannot write to persistent volume

**Diagnosis:**

```bash
# Check PVC usage
kubectl get pvc --all-namespaces

# Check pod volume usage
kubectl exec <pod-name> -- df -h

# Describe PVC
kubectl describe pvc <pvc-name>

# Check for CSI driver support for volume expansion
kubectl get storageclass <storage-class> -o yaml | grep allowVolumeExpansion

# Check current PV size
kubectl get pv <pv-name> -o jsonpath='{.spec.capacity.storage}'

# Identify large files
kubectl exec <pod-name> -- du -sh /* | sort -h
```

**Resolution:**

```bash
# Option 1: Expand existing PVC (if storage class supports it)
# Edit PVC to increase size
kubectl edit pvc <pvc-name>
# Change spec.resources.requests.storage: 50Gi

# Restart pod to recognize new size
kubectl delete pod <pod-name>

# Verify expansion
kubectl get pvc <pvc-name>

# Option 2: Clean up space
# Delete old logs
kubectl exec <pod-name> -- find /data -name "*.log" -mtime +7 -delete

# Archive old data
kubectl exec <pod-name> -- tar -czf /data/archive-$(date +%Y%m%d).tar.gz /data/old/*
kubectl exec <pod-name> -- rm -rf /data/old/*

# Option 3: Create new larger PVC and migrate
# 1. Create new PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>-new
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: <storage-class>
EOF

# 2. Copy data using a temporary pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pvc-migrator
spec:
  containers:
  - name: migrator
    image: busybox
    command: ["sh", "-c", "cp -av /source/* /dest/ && echo Done"]
    volumeMounts:
    - name: source
      mountPath: /source
    - name: dest
      mountPath: /dest
  volumes:
  - name: source
    persistentVolumeClaim:
      claimName: <pvc-name>
  - name: dest
    persistentVolumeClaim:
      claimName: <pvc-name>-new
  restartPolicy: Never
EOF

# 3. Update deployment to use new PVC
kubectl patch deployment <deployment-name> -p '{"spec":{"template":{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"<pvc-name>-new"}}]}}}}'
```

**Prevention:**
- Set up PVC usage monitoring (alert at 80%)
- Use storage classes with allowVolumeExpansion: true
- Implement log rotation
- Regular data archival
- Set application log retention policies
- Use separate volumes for logs vs. data
- Consider using ephemeral volumes for logs

### Runbook: Namespace Stuck Terminating

**Symptoms:**
- `kubectl delete namespace <ns>` hangs
- Namespace shows "Terminating" for extended time
- Namespace cannot be recreated
- Resources in namespace not deletable

**Diagnosis:**

```bash
# Check namespace status
kubectl get namespace <namespace> -o yaml

# Check for finalizers
kubectl get namespace <namespace> -o jsonpath='{.spec.finalizers}'

# Check resources in namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace>

# Check for custom resources
kubectl get customresourcedefinitions

# Check for webhooks blocking deletion
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Check API server logs for errors
kubectl logs -n kube-system kube-apiserver-<node> | grep <namespace>
```

**Resolution:**

```bash
# Remove finalizers from namespace
kubectl get namespace <namespace> -o json > namespace.json

# Edit namespace.json - remove finalizers section:
# "spec": {
#   "finalizers": []
# }

# Update namespace
kubectl replace --raw "/api/v1/namespaces/<namespace>/finalize" -f namespace.json

# Force delete specific resources with finalizers
kubectl patch <resource-type> <resource-name> -n <namespace> \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# If caused by missing CRD
# Delete resources of deleted CRD
for ns in $(kubectl get namespace -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get <crd-resource> -n $ns -o name | xargs kubectl delete -n $ns
done

# If caused by webhook
# Delete or update webhook to not block namespace deletion
kubectl delete validatingwebhookconfiguration <webhook-name>

# Nuclear option: remove namespace from etcd (DANGEROUS!)
# Only use as last resort
etcdctl del /registry/namespaces/<namespace>
```

**Prevention:**
- Properly implement finalizers in custom controllers
- Ensure webhooks have timeouts and failure policies
- Document resource dependencies
- Use namespace lifecycle webhooks carefully
- Regular namespace cleanup procedures
- Test namespace deletion in dev before applying to prod
- Monitor for stuck namespaces

---

## Chapter 20 Review Questions

1. **What are the three health endpoints provided by the Kubernetes API server, and what is the difference between them?**

2. **Explain the etcd Raft consensus protocol. What is quorum, and why is it important? What happens if a 5-node etcd cluster loses 3 nodes?**

3. **Compare in-place rolling upgrades vs. blue-green cluster upgrades. When would you use each strategy?**

4. **What is a PodDisruptionBudget, and why is it important? Write a PDB that ensures at least 80% availability for a deployment with 10 replicas.**

5. **Describe the difference between Kubernetes resource requests and limits. What are the three QoS classes, and how are they determined?**

---

## Chapter 20 Hands-On Exercises

1. **etcd Backup and Restore Simulation**
   - Take an etcd snapshot
   - Deploy an application with persistent data
   - Simulate disaster by deleting the application
   - Restore from etcd snapshot
   - Verify application and data recovery

2. **Node Maintenance Workflow**
   - Deploy an application with 5 replicas and a PodDisruptionBudget
   - Cordon and drain a node
   - Verify application remains available (check pod distribution)
   - Perform simulated maintenance (wait 2 minutes)
   - Uncordon the node and verify scheduling resumes

3. **Multi-Cluster GitOps Setup**
   - Create two kind clusters (cluster-a, cluster-b)
   - Install ArgoCD in a management cluster
   - Create an ApplicationSet that deploys to both clusters
   - Modify application config in Git
   - Verify automatic sync to both clusters

---

## Key Takeaways

- **Control plane health** depends on API server, etcd, scheduler, and controller manager working together. Monitor health endpoints and key metrics continuously.

- **etcd operations** are critical for cluster reliability. Regular backups (hourly snapshots), compaction, and defragmentation prevent data loss and performance degradation.

- **Cluster upgrades** require careful planning. Back up etcd before upgrading, check API deprecations, follow version skew policies, and always have a rollback plan.

- **Node management** with cordon/drain/uncordon ensures safe maintenance. PodDisruptionBudgets protect application availability during voluntary disruptions.

- **Capacity planning** involves right-sizing workloads with proper requests/limits, using VPA for recommendations, and enforcing quotas at the namespace level.

- **Backup and disaster recovery** strategies should include etcd snapshots, Velero for Kubernetes objects and volumes, and GitOps for configuration management.

- **Production runbooks** provide standardized procedures for common issues, reducing MTTR (Mean Time To Recovery) and ensuring consistent incident response.

---

[Next Chapter: Kubernetes Reliability Engineering →](21-reliability.md)