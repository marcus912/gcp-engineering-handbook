# Chapter 18: Kubernetes Debugging Deep Dive

**Prerequisites**: Chapter 12 (Containers & Kubernetes), Chapter 13 (Troubleshooting Methodology), Chapter 17 (Monitoring & Observability)

Debugging Kubernetes issues requires systematic methodology and deep understanding of cluster components. This chapter covers production debugging techniques, common failure patterns, and specialized tooling for resolving complex issues.

---

## 17.1 Systematic Pod Debugging

### Pod Lifecycle

Understanding the pod lifecycle is fundamental to debugging:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Pod Lifecycle States                        │
└─────────────────────────────────────────────────────────────────┘

    ┌──────────┐
    │ Pending  │  ← Pod accepted but not scheduled or pulling images
    └────┬─────┘
         │
         ├─→ ImagePullBackOff  (image issues)
         ├─→ ContainerCreating (volume mount issues)
         │
         ▼
    ┌──────────┐
    │ Running  │  ← At least one container running
    └────┬─────┘
         │
         ├─→ CrashLoopBackOff (container exits repeatedly)
         ├─→ Error (startup/liveness probe failures)
         │
         ▼
    ┌──────────┐                ┌──────────┐
    │Succeeded │                │  Failed  │
    └──────────┘                └──────────┘
         │                            │
         └────────────┬───────────────┘
                      ▼
                 ┌──────────┐
                 │Terminating│
                 └──────────┘
```

### Debugging Decision Tree

```
Pod Issue?
│
├─→ Pending
│   ├─→ Insufficient resources → Check node capacity
│   ├─→ PVC not bound → Check storage class, capacity
│   ├─→ Node selector/affinity → Check node labels
│   └─→ ImagePullBackOff → Check image name, registry auth
│
├─→ CrashLoopBackOff
│   ├─→ Check logs: kubectl logs <pod> --previous
│   ├─→ Check exit code in container status
│   ├─→ Check startup probes/liveness probes
│   └─→ Check command/args in pod spec
│
├─→ Error/ImagePullBackOff
│   ├─→ Check image pull secrets
│   ├─→ Verify registry connectivity
│   ├─→ Check image name/tag validity
│   └─→ Check registry credentials
│
├─→ Running but not working
│   ├─→ Check application logs
│   ├─→ Verify service/endpoint configuration
│   ├─→ Check readiness probes
│   └─→ Test network connectivity (kubectl exec)
│
└─→ Terminating (stuck)
    ├─→ Check finalizers
    ├─→ Force delete (last resort)
    └─→ Check for volume unmount issues
```

### Essential kubectl Commands

```bash
# Get pod status and recent events
kubectl get pod my-pod -o wide
kubectl describe pod my-pod

# Check container logs (current and previous)
kubectl logs my-pod
kubectl logs my-pod --previous
kubectl logs my-pod -c container-name  # multi-container pod
kubectl logs my-pod --tail=100 --follow

# Get all events in namespace (sorted by time)
kubectl get events --sort-by=.metadata.creationTimestamp

# Get pod events only
kubectl get events --field-selector involvedObject.name=my-pod

# Check pod spec and status
kubectl get pod my-pod -o yaml
kubectl get pod my-pod -o jsonpath='{.status.phase}'
kubectl get pod my-pod -o jsonpath='{.status.conditions[*].type}'

# Interactive debugging
kubectl exec -it my-pod -- /bin/sh
kubectl exec my-pod -- env
kubectl exec my-pod -- ps aux

# Port forwarding for local testing
kubectl port-forward my-pod 8080:8080

# Copy files for inspection
kubectl cp my-pod:/var/log/app.log ./app.log
```

### Container Status Interpretation

| Status | Reason | Common Causes | Debugging Steps |
|--------|--------|---------------|-----------------|
| **Waiting** | ContainerCreating | Volume mount issues, image pull | Check events, describe pod |
| **Waiting** | ImagePullBackOff | Image not found, auth failure | Verify image name, check imagePullSecrets |
| **Waiting** | CrashLoopBackOff | Application crashes on start | Check logs with `--previous`, verify command |
| **Waiting** | CreateContainerConfigError | ConfigMap/Secret missing | Verify configMapRef/secretRef exists |
| **Waiting** | InvalidImageName | Malformed image reference | Check image field in pod spec |
| **Terminated** | Error | Application exited with error | Check exit code, logs, and error message |
| **Terminated** | OOMKilled | Out of memory | Increase memory limits, check for leaks |
| **Terminated** | Completed | Container finished successfully | Normal for Jobs/CronJobs |
| **Running** | (none) | Container is running | Check application logs if not working |

### Detailed Status Inspection

```bash
# Get detailed container status
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*]}'

# Check restart count
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*].restartCount}'

# Get container state
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*].state}'

# Get last termination reason
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# Get exit code
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'
```

### Common Exit Codes

```bash
# Exit code meanings
0   - Success
1   - General application error
2   - Misuse of shell command
126 - Command cannot execute
127 - Command not found
130 - Container terminated by Ctrl+C
137 - Container received SIGKILL (OOMKilled = 137)
143 - Container received SIGTERM
255 - Exit status out of range
```

### Example: Debugging CrashLoopBackOff

```bash
# Step 1: Get pod status
kubectl get pod webapp-5c7d9f8b6-abc12
# NAME                      READY   STATUS             RESTARTS   AGE
# webapp-5c7d9f8b6-abc12    0/1     CrashLoopBackOff   5          3m

# Step 2: Check previous container logs
kubectl logs webapp-5c7d9f8b6-abc12 --previous
# Error: Cannot connect to database at db-service:5432

# Step 3: Check if service exists
kubectl get svc db-service
# Error from server (NotFound): services "db-service" not found

# Step 4: Fix - create missing service or update connection string
kubectl apply -f db-service.yaml

# Step 5: Verify pod recovers
kubectl get pod webapp-5c7d9f8b6-abc12 -w
```

---

## 17.2 Scheduling & Resource Debugging

### Scheduler Decision Process

```
┌────────────────────────────────────────────────────────────┐
│              Kubernetes Scheduler Flow                      │
└────────────────────────────────────────────────────────────┘

Pod Created
    │
    ▼
┌─────────────────────┐
│  Filtering Phase    │  ← Eliminate incompatible nodes
│                     │
│ • Node resources    │
│ • Node selector     │
│ • Affinity/Anti     │
│ • Taints/Tolerations│
│ • Volume zones      │
│ • Port conflicts    │
└──────────┬──────────┘
           │
           ▼
    All nodes filtered out? ──Yes──→ Pod stays Pending
           │
           No
           │
           ▼
┌─────────────────────┐
│   Scoring Phase     │  ← Rank remaining nodes
│                     │
│ • Resource balance  │
│ • Affinity weights  │
│ • Image locality    │
│ • Inter-pod affinity│
└──────────┬──────────┘
           │
           ▼
    Select highest score
           │
           ▼
┌─────────────────────┐
│   Binding Phase     │  ← Assign pod to node
└─────────────────────┘
```

### Common Scheduling Failures

```bash
# Check why pod is pending
kubectl describe pod my-pod | grep -A 10 Events

# Common messages:
# - "0/3 nodes are available: 3 Insufficient cpu."
# - "0/3 nodes are available: 3 node(s) didn't match Pod's node affinity."
# - "0/3 nodes are available: 3 node(s) had untolerated taint."
# - "persistentvolumeclaim not found"
```

**Insufficient Resources**

```bash
# Check node capacity and allocatable resources
kubectl describe node node-1 | grep -A 5 "Allocated resources"

# Example output:
# Allocated resources:
#   CPU Requests    CPU Limits    Memory Requests    Memory Limits
#   ------------    ----------    ---------------    -------------
#   1800m (90%)     3000m (150%)  2.5Gi (80%)       4Gi (128%)

# Check resource quotas in namespace
kubectl describe resourcequota -n my-namespace

# Check limit ranges
kubectl describe limitrange -n my-namespace

# Get resource requests/limits for all pods
kubectl get pods -n my-namespace -o custom-columns=\
NAME:.metadata.name,\
CPU_REQ:.spec.containers[*].resources.requests.cpu,\
CPU_LIM:.spec.containers[*].resources.limits.cpu,\
MEM_REQ:.spec.containers[*].resources.requests.memory,\
MEM_LIM:.spec.containers[*].resources.limits.memory
```

### ResourceQuota Example

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: development
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
    pods: "50"
```

```bash
# Debug quota issues
kubectl describe resourcequota compute-quota -n development

# Output shows:
# Name:                   compute-quota
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              19      20
# limits.memory           38Gi    40Gi
# pods                    48      50
# requests.cpu            9500m   10
# requests.memory         19Gi    20Gi
```

### Node Selector & Affinity Issues

```bash
# Check node labels
kubectl get nodes --show-labels

# Check if any nodes match pod's node selector
kubectl get nodes -l disktype=ssd

# Example pod with node selector
kubectl get pod my-pod -o yaml | grep -A 5 nodeSelector
# nodeSelector:
#   disktype: ssd

# Check node affinity
kubectl get pod my-pod -o yaml | grep -A 20 affinity
```

### Taints and Tolerations

```bash
# Check node taints
kubectl describe node node-1 | grep Taints
# Taints: dedicated=gpu:NoSchedule

# Check pod tolerations
kubectl get pod my-pod -o yaml | grep -A 5 tolerations

# Example: Pod without toleration cannot schedule
# Event: 0/3 nodes are available: 3 node(s) had untolerated taint {dedicated: gpu}.
```

**Toleration Example**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: app
    image: my-gpu-app
```

### Pod Priority and Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "High priority workloads"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: critical-app
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
```

```bash
# Check priority classes
kubectl get priorityclasses

# Check if pod is preempting others
kubectl describe pod critical-app | grep -i preempt

# View preemption events
kubectl get events --field-selector reason=Preempted
```

### Node Pressure and Eviction

```bash
# Check node conditions
kubectl describe node node-1 | grep -A 10 Conditions

# Pressure types:
# - MemoryPressure: node is running out of memory
# - DiskPressure: node is running out of disk space
# - PIDPressure: node is running out of PIDs

# Check node allocatable vs capacity
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
MEMORY_CAP:.status.capacity.memory,\
MEMORY_ALLOC:.status.allocatable.memory,\
DISK_CAP:.status.capacity.ephemeral-storage,\
DISK_ALLOC:.status.allocatable.ephemeral-storage

# Get eviction events
kubectl get events --all-namespaces --field-selector reason=Evicted

# Check pod eviction status
kubectl get pods --all-namespaces --field-selector status.phase=Failed \
  -o custom-columns=NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
REASON:.status.reason
```

### Eviction Thresholds (kubelet)

```bash
# Common eviction thresholds:
# memory.available < 100Mi
# nodefs.available < 10%
# nodefs.inodesFree < 5%
# imagefs.available < 15%

# Check kubelet configuration (on node)
sudo systemctl cat kubelet | grep eviction
```

### Debugging Node Resource Issues

```bash
# SSH to node and check actual resource usage
# (requires node access)

# Check memory
free -h
cat /proc/meminfo

# Check disk
df -h
du -sh /var/lib/kubelet/*
du -sh /var/lib/docker/*  # or /var/lib/containerd

# Check running containers
crictl ps
crictl stats

# Check kubelet logs
sudo journalctl -u kubelet -f

# Check system resource pressure
dmesg | grep -i "out of memory"
```

---

## 17.3 Storage & Volume Debugging

### PVC Lifecycle

```
┌────────────────────────────────────────────────────────────┐
│          PersistentVolumeClaim Lifecycle                    │
└────────────────────────────────────────────────────────────┘

PVC Created
    │
    ▼
┌─────────────┐
│   Pending   │  ← Waiting for PV or dynamic provisioning
└──────┬──────┘
       │
       ├─→ No matching PV → Check PV availability, labels
       ├─→ StorageClass not found → Verify storage class exists
       ├─→ Provisioner error → Check CSI driver logs
       │
       ▼
┌─────────────┐
│    Bound    │  ← PVC successfully bound to PV
└──────┬──────┘
       │
       ▼
   Pod mounts PVC
       │
       ├─→ Mount errors → Check permissions, fsGroup
       ├─→ Volume not attached → Check CSI node driver
       │
       ▼
┌─────────────┐
│  Released   │  ← PVC deleted, PV still contains data
└──────┬──────┘
       │
       ▼
Reclaim Policy:
├─→ Retain: Manual cleanup required
├─→ Delete: PV and storage automatically deleted
└─→ Recycle: Deprecated (use Delete)
```

### Common PVC Issues

```bash
# Check PVC status
kubectl get pvc
# NAME        STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-pvc    Pending                                       fast-ssd       5m

# Describe PVC for events
kubectl describe pvc data-pvc

# Common issues in events:
# - "waiting for a volume to be created"
# - "no persistent volumes available"
# - "storageclass.storage.k8s.io 'fast-ssd' not found"
```

### Storage Class Issues

```bash
# List available storage classes
kubectl get storageclass

# Check if storage class exists
kubectl get storageclass fast-ssd

# Describe storage class
kubectl describe storageclass fast-ssd

# Check default storage class
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

**Example PVC with Storage Class**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd  # Must exist
  resources:
    requests:
      storage: 10Gi
```

### Capacity Mismatch

```bash
# PVC requests 100Gi but only 50Gi PV available
kubectl get pv -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name

# Output:
# NAME      CAPACITY   STATUS      CLAIM
# pv-001    50Gi       Available   <none>
# pv-002    50Gi       Bound       other-pvc
```

### Access Mode Conflicts

| Access Mode | Abbreviation | Description | Use Case |
|-------------|--------------|-------------|----------|
| ReadWriteOnce | RWO | Volume mounted read-write by single node | Most common, block storage |
| ReadOnlyMany | ROX | Volume mounted read-only by many nodes | Shared configuration |
| ReadWriteMany | RWX | Volume mounted read-write by many nodes | Shared data (NFS, CephFS) |
| ReadWriteOncePod | RWOP | Volume mounted read-write by single pod | Kubernetes 1.22+ |

```bash
# PVC requests RWX but PV only supports RWO
kubectl describe pvc data-pvc
# Events:
#   Type     Reason              Age   Message
#   ----     ------              ----  -------
#   Warning  ProvisioningFailed  1m    no volume with matching access modes found
```

### CSI Driver Troubleshooting

```bash
# List CSI drivers
kubectl get csidrivers

# Check CSI driver pods (example: GCE PD CSI)
kubectl get pods -n kube-system -l app=gcp-compute-persistent-disk-csi-driver

# Check CSI controller logs
kubectl logs -n kube-system \
  -l app=gcp-compute-persistent-disk-csi-driver \
  -c csi-provisioner

# Check CSI node driver logs (runs on each node)
kubectl logs -n kube-system \
  -l app=gcp-compute-persistent-disk-csi-driver \
  -c csi-driver-registrar

# Describe CSI node
kubectl get csinode
kubectl describe csinode node-1
```

### Volume Attachment Issues

```bash
# Check VolumeAttachments
kubectl get volumeattachment

# Example:
# NAME                                   ATTACHER             PV         NODE     ATTACHED   AGE
# csi-a1b2c3...                          pd.csi.storage...    pv-001     node-1   true       5m

# Describe volume attachment
kubectl describe volumeattachment csi-a1b2c3...

# Common issues:
# - "failed to attach volume" → Check CSI driver, node connectivity
# - "multi-attach error" → RWO volume already attached to another node
```

### StatefulSet PVC Management

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
```

```bash
# StatefulSet creates PVC per pod: data-database-0, data-database-1, data-database-2
kubectl get pvc -l app=database

# Common StatefulSet PVC issues:
# 1. PVC stuck in Pending → Check storage class, capacity
# 2. Pod stuck in Pending → PVC not bound
# 3. Scaling down doesn't delete PVCs (by design)

# Manual PVC cleanup after scaling down
kubectl delete pvc data-database-2

# Danger: Deleting PVC may delete data depending on reclaim policy
```

### Volume Mount Errors

```bash
# Check pod events for mount errors
kubectl describe pod my-pod | grep -A 10 "Events:"

# Common mount errors:
# - "Unable to attach or mount volumes" → CSI issue, node connectivity
# - "MountVolume.SetUp failed" → Permissions, fsGroup, SELinux
# - "Volume is already exclusively attached" → RWO multi-attach attempt
```

**Permission Issues with fsGroup**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  securityContext:
    fsGroup: 2000  # Group ID for volume ownership
    runAsUser: 1000
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

```bash
# Verify volume permissions inside pod
kubectl exec app-pod -- ls -la /data
# drwxrwsr-x 2 root 2000 4096 Jan 1 00:00 .

# Without fsGroup, may see permission denied errors
# Error: open /data/file.txt: permission denied
```

### Debugging Volume Mount Failures

```bash
# Step 1: Check if PVC is bound
kubectl get pvc data-pvc
# STATUS should be "Bound"

# Step 2: Check if PV exists
kubectl get pv | grep data-pvc

# Step 3: Check pod describe for mount errors
kubectl describe pod my-pod

# Step 4: Check node-level CSI driver
kubectl get pods -n kube-system -l app=<csi-driver-name> -o wide
# Ensure CSI driver running on the node where pod is scheduled

# Step 5: Check VolumeAttachment
kubectl get volumeattachment -o yaml | grep -A 20 pv-name
```

---

## 17.4 RBAC Debugging

### Testing Permissions with kubectl auth can-i

```bash
# Test if current user can perform action
kubectl auth can-i create deployments
kubectl auth can-i get pods --namespace kube-system
kubectl auth can-i '*' '*'  # Check cluster-admin

# Test as specific user
kubectl auth can-i create deployments --as john@example.com
kubectl auth can-i delete pods --as system:serviceaccount:default:my-sa

# Test as user with group
kubectl auth can-i create deployments --as john@example.com \
  --as-group=system:authenticated

# List all permissions for current user
kubectl auth can-i --list

# List all permissions in specific namespace
kubectl auth can-i --list --namespace production
```

### Common RBAC Misconfigurations

| Issue | Symptom | Common Cause | Fix |
|-------|---------|--------------|-----|
| Forbidden (403) | API requests denied | Missing Role/RoleBinding | Create appropriate binding |
| ServiceAccount can't access | Pod operations fail | SA not bound to Role | Add RoleBinding for SA |
| Cross-namespace access denied | Can't access other namespace | Role instead of ClusterRole | Use ClusterRole + ClusterRoleBinding |
| Wildcard not working | Specific resources denied | Incorrect API group | Verify `apiGroups` matches resource |
| Verb mismatch | Can list but not watch | Missing verb in Role | Add all required verbs |
| Webhook failures | Admission denied | Webhook SA lacks permissions | Grant SA appropriate ClusterRole |

### Service Account Debugging

```bash
# Check pod's service account
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'

# Describe service account
kubectl describe sa my-service-account

# Check service account secrets (token)
kubectl get sa my-service-account -o jsonpath='{.secrets[*].name}'

# Get token for testing
TOKEN=$(kubectl create token my-service-account --duration=1h)
kubectl --token=$TOKEN get pods

# Check RoleBindings for service account
kubectl get rolebinding -A -o json | \
  jq -r '.items[] | select(.subjects[]?.name=="my-service-account") |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Check ClusterRoleBindings for service account
kubectl get clusterrolebinding -o json | \
  jq -r '.items[] | select(.subjects[]?.name=="my-service-account") |
  .metadata.name'
```

### ClusterRole vs Role

```yaml
# Role: Namespace-scoped permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ClusterRole: Cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Debugging RBAC Denials

```bash
# Enable audit logging to capture RBAC denials
# (requires API server configuration)

# Example audit policy for RBAC:
# apiVersion: audit.k8s.io/v1
# kind: Policy
# rules:
# - level: Metadata
#   omitStages:
#   - RequestReceived
#   verbs: ["create", "update", "patch", "delete"]
# - level: Request
#   resources:
#   - group: ""
#     resources: ["pods"]

# View audit logs (location varies by cluster)
# GKE: Cloud Logging
# EKS: CloudWatch
# Self-managed: /var/log/kubernetes/audit.log

# Search for RBAC denials in audit log
grep "Forbidden" /var/log/kubernetes/audit.log
```

### Testing RBAC with Impersonation

```bash
# Create test user and binding
kubectl create serviceaccount test-user
kubectl create rolebinding test-binding \
  --clusterrole=view \
  --serviceaccount=default:test-user \
  --namespace=default

# Test as service account
kubectl get pods --as=system:serviceaccount:default:test-user

# Should succeed (view role allows get pods)
kubectl get pods --as=system:serviceaccount:default:test-user

# Should fail (view role doesn't allow create)
kubectl run nginx --image=nginx \
  --as=system:serviceaccount:default:test-user
# Error from server (Forbidden): pods is forbidden...
```

### Advanced RBAC Debugging

```bash
# Find all Roles/ClusterRoles that grant access to specific resource
kubectl get clusterroles -o json | \
  jq -r '.items[] | select(.rules[]?.resources[]? == "secrets") |
  .metadata.name'

# Find all bindings for a specific user/SA
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq -r '.items[] | select(.subjects[]?.name=="jenkins") |
  "\(.kind) \(.metadata.namespace)/\(.metadata.name)"'

# Check if service account has ClusterAdmin
kubectl auth can-i '*' '*' \
  --as=system:serviceaccount:kube-system:my-sa

# View effective permissions (requires rbac-lookup tool)
kubectl rbac-lookup my-service-account --kind serviceaccount
```

---

## 17.5 Admission Controller & Webhook Debugging

### Admission Control Flow

```
┌────────────────────────────────────────────────────────────────┐
│              Admission Controller Pipeline                      │
└────────────────────────────────────────────────────────────────┘

API Request (kubectl apply)
    │
    ▼
┌──────────────────────┐
│  Authentication      │
│  Authorization(RBAC) │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│        Mutating Admission Webhooks           │
│                                              │
│  • Inject sidecars (Istio, Linkerd)         │
│  • Add labels/annotations                    │
│  • Set defaults (Kyverno mutate)            │
│  • Modify resource requests                  │
│                                              │
│  Multiple webhooks run in configured order   │
└──────────┬───────────────────────────────────┘
           │
           ▼
    Object Schema Validation
           │
           ▼
┌──────────────────────────────────────────────┐
│       Validating Admission Webhooks          │
│                                              │
│  • Policy enforcement (OPA Gatekeeper)       │
│  • Security policies (Pod Security)          │
│  • Custom validation (Kyverno validate)      │
│  • Quota checks                              │
│                                              │
│  All webhooks must approve (any deny fails)  │
└──────────┬───────────────────────────────────┘
           │
           ▼
    All webhooks approved?
           │
           ├─→ No → Return 403 Forbidden (denied by webhook)
           │
           ▼
    Persisted to etcd
```

### Debugging Webhook Failures

```bash
# Webhook failures appear immediately in kubectl output
kubectl apply -f pod.yaml
# Error from server (Forbidden): error when creating "pod.yaml":
# admission webhook "validate.kyverno.svc" denied the request:
# validation error: Image must be from approved registry

# Check active validating webhooks
kubectl get validatingwebhookconfigurations

# Check active mutating webhooks
kubectl get mutatingwebhookconfigurations

# Describe webhook configuration
kubectl describe validatingwebhookconfiguration policy-validation

# Check webhook service and endpoints
kubectl get svc -n kyverno kyverno-svc
kubectl get endpoints -n kyverno kyverno-svc

# Check webhook pod logs
kubectl logs -n kyverno -l app=kyverno --tail=50
```

### Webhook Configuration Example

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: policy-validation
webhooks:
- name: validate.policies.example.com
  clientConfig:
    service:
      name: policy-service
      namespace: policy-system
      path: "/validate"
    caBundle: LS0tLS1CRUdJTi...  # Base64 encoded CA cert
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  namespaceSelector:
    matchLabels:
      policy: enabled
  failurePolicy: Fail  # or Ignore
  sideEffects: None
  timeoutSeconds: 10
  admissionReviewVersions: ["v1", "v1beta1"]
```

### Webhook Failure Policies

```bash
# failurePolicy: Fail (default)
# - Webhook failure causes request rejection
# - Use for critical policies

# failurePolicy: Ignore
# - Webhook failure allows request through
# - Use for non-critical, advisory policies

# Check failure policy
kubectl get validatingwebhookconfiguration policy-validation \
  -o jsonpath='{.webhooks[*].failurePolicy}'
```

### Common Webhook Issues

```bash
# Issue 1: Webhook timeout (default 10s)
# Error: context deadline exceeded

# Check webhook timeout configuration
kubectl get validatingwebhookconfiguration policy-validation \
  -o jsonpath='{.webhooks[*].timeoutSeconds}'

# Increase timeout if needed
kubectl patch validatingwebhookconfiguration policy-validation \
  --type='json' -p='[{"op": "replace",
  "path": "/webhooks/0/timeoutSeconds", "value":30}]'

# Issue 2: Webhook service unreachable
# Error: failed calling webhook: Post "https://...": dial tcp: i/o timeout

# Check service exists and has endpoints
kubectl get svc -n policy-system policy-service
kubectl get endpoints -n policy-system policy-service

# Check webhook pod is running
kubectl get pods -n policy-system -l app=policy-service

# Issue 3: Certificate validation failure
# Error: x509: certificate signed by unknown authority

# Check webhook CA bundle
kubectl get validatingwebhookconfiguration policy-validation \
  -o jsonpath='{.webhooks[*].clientConfig.caBundle}' | base64 -d

# Update CA bundle if expired
kubectl patch validatingwebhookconfiguration policy-validation \
  --type='json' -p='[{"op": "replace",
  "path": "/webhooks/0/clientConfig/caBundle", "value":"'$(cat ca.crt | base64)'"}]'
```

### OPA Gatekeeper Debugging

```bash
# Check Gatekeeper installation
kubectl get pods -n gatekeeper-system

# List constraint templates
kubectl get constrainttemplates

# List active constraints
kubectl get constraints

# Example: Check required labels constraint
kubectl get k8srequiredlabels

# Describe constraint for violations
kubectl describe k8srequiredlabels require-team-label

# Check Gatekeeper audit logs for violations
kubectl logs -n gatekeeper-system \
  -l control-plane=audit-controller --tail=100

# Test constraint without applying
kubectl apply --dry-run=server -f pod.yaml
```

**Example Gatekeeper Constraint**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    labels:
    - key: "team"
      allowedRegex: "^[a-z]+$"
```

### Kyverno Debugging

```bash
# Check Kyverno installation
kubectl get pods -n kyverno

# List policies
kubectl get clusterpolicy
kubectl get policy -A

# Check policy status
kubectl describe clusterpolicy require-labels

# View policy violations (policy reports)
kubectl get policyreport -A
kubectl describe policyreport polr-ns-default -n default

# Check Kyverno webhook logs
kubectl logs -n kyverno -l app=kyverno --tail=100 -f

# Test policy in audit mode (don't enforce)
kubectl apply -f policy.yaml
# Change validationFailureAction: audit  (instead of enforce)
```

**Example Kyverno Policy**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-registry
spec:
  validationFailureAction: enforce  # or audit
  background: true
  rules:
  - name: check-registry
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Images must be from approved registry"
      pattern:
        spec:
          containers:
          - image: "registry.example.com/*"
```

### Pod Security Standards

```bash
# Pod Security Admission replaced PodSecurityPolicy in 1.25

# Check namespace pod security labels
kubectl get namespace production -o yaml | grep pod-security

# Example namespace labels:
# pod-security.kubernetes.io/enforce: restricted
# pod-security.kubernetes.io/audit: restricted
# pod-security.kubernetes.io/warn: baseline

# Test pod against policy
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

kubectl apply -f privileged-pod.yaml
# Warning: would violate PodSecurity "restricted:latest":
# privileged, runAsNonRoot != true, seccompProfile
```

### Bypassing Webhooks for Debugging

```bash
# Temporary disable webhook (dangerous!)
kubectl delete validatingwebhookconfiguration policy-validation

# Or modify webhook to Ignore failure policy
kubectl patch validatingwebhookconfiguration policy-validation \
  --type='json' -p='[{"op": "replace",
  "path": "/webhooks/0/failurePolicy", "value":"Ignore"}]'

# Exclude namespace from webhook
kubectl label namespace debugging policy=disabled

# Check webhook namespaceSelector
kubectl get validatingwebhookconfiguration policy-validation \
  -o jsonpath='{.webhooks[*].namespaceSelector}'
```

---

## 17.6 Helm & Release Debugging

### Helm Template Validation

```bash
# Render templates locally without installing
helm template my-release ./my-chart

# Render with specific values
helm template my-release ./my-chart -f custom-values.yaml

# Render and pipe to kubectl to validate
helm template my-release ./my-chart | kubectl apply --dry-run=client -f -

# Validate against API server (includes admission webhooks)
helm template my-release ./my-chart | kubectl apply --dry-run=server -f -

# Show only specific resource
helm template my-release ./my-chart --show-only templates/deployment.yaml

# Debug template rendering issues
helm template my-release ./my-chart --debug
```

### Helm Install/Upgrade Debugging

```bash
# Dry run before actual install
helm install my-release ./my-chart --dry-run --debug

# Dry run shows:
# 1. Rendered templates
# 2. Values used (including defaults)
# 3. Hooks that would execute

# Install with debug output
helm install my-release ./my-chart --debug --timeout 10m

# Upgrade with wait and atomic (auto rollback on failure)
helm upgrade my-release ./my-chart --wait --atomic --timeout 10m

# Upgrade with dry-run to see what would change
helm upgrade my-release ./my-chart --dry-run --debug
```

### Helm Release History

```bash
# View release history
helm history my-release

# Example output:
# REVISION  UPDATED                   STATUS      CHART           DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  myapp-1.0.0     Install complete
# 2         Mon Jan 1 11:00:00 2024   superseded  myapp-1.1.0     Upgrade complete
# 3         Mon Jan 1 12:00:00 2024   deployed    myapp-1.2.0     Upgrade complete

# Get release values for specific revision
helm get values my-release --revision 2

# Get manifest for specific revision
helm get manifest my-release --revision 2

# Get all information for release
helm get all my-release

# Rollback to previous revision
helm rollback my-release 2

# Rollback with wait
helm rollback my-release 2 --wait --timeout 5m
```

### Common Helm Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `cannot re-use a name that is still in use` | Release exists | Use `--replace` or uninstall first |
| `UPGRADE FAILED: timed out waiting` | Resources not ready | Increase `--timeout`, check pod status |
| `YAML parse error` | Invalid template syntax | Run `helm template` to debug |
| `unable to build kubernetes objects` | Invalid K8s resource | Validate with `kubectl apply --dry-run` |
| `has no deployed releases` | Release never succeeded | Check `helm history`, may need uninstall |
| `release name is invalid` | Name violates DNS rules | Use lowercase, hyphens only |
| `failed to download chart` | Repository/auth issue | Check `helm repo list`, update repo |

### Stuck Releases

```bash
# Check release status
helm list --all-namespaces

# Common stuck states:
# - pending-install: Initial install failed/interrupted
# - pending-upgrade: Upgrade failed/interrupted
# - pending-rollback: Rollback failed/interrupted

# View release details
helm get all my-release

# Attempt rollback
helm rollback my-release 0  # 0 = previous revision

# Force delete if rollback fails (dangerous!)
kubectl delete secret -n default \
  sh.helm.release.v1.my-release.v3

# Or uninstall with force
helm uninstall my-release --no-hooks
```

### Debugging Helm Hooks

```bash
# Helm hooks run at specific points:
# - pre-install, post-install
# - pre-upgrade, post-upgrade
# - pre-rollback, post-rollback
# - pre-delete, post-delete
# - test

# View hooks in chart
helm get hooks my-release

# Example hook annotation:
# annotations:
#   "helm.sh/hook": pre-upgrade
#   "helm.sh/hook-weight": "0"
#   "helm.sh/hook-delete-policy": hook-succeeded

# Check hook resources (Jobs/Pods)
kubectl get pods -l heritage=Helm,release=my-release

# Check hook job logs
kubectl logs job/my-release-pre-upgrade-hook

# Hooks that fail will cause install/upgrade to fail
# Skip hooks if problematic (not recommended)
helm upgrade my-release ./my-chart --no-hooks
```

### Helm Diff Plugin

```bash
# Install diff plugin
helm plugin install https://github.com/databus23/helm-diff

# Show what would change in upgrade
helm diff upgrade my-release ./my-chart

# Show changes with color
helm diff upgrade my-release ./my-chart --color

# Compare to specific revision
helm diff revision my-release 2 3
```

### Template Function Debugging

```yaml
# Common template issues

# Issue: Undefined variable
{{ .Values.nonexistent.field }}
# Solution: Use default or check existence
{{ .Values.nonexistent.field | default "fallback" }}
{{ if .Values.nonexistent }}{{ .Values.nonexistent.field }}{{ end }}

# Issue: Type mismatch (string vs int)
replicas: {{ .Values.replicas }}  # May render as "3" instead of 3
# Solution: Use int conversion
replicas: {{ .Values.replicas | int }}

# Issue: Invalid YAML indentation
{{ .Values.annotations | toYaml | indent 4 }}
# Solution: Use nindent (newline + indent)
{{ .Values.annotations | toYaml | nindent 4 }}

# Issue: Quote handling
name: {{ .Values.name }}  # Missing quotes if name has spaces
# Solution: Use quote function
name: {{ .Values.name | quote }}
```

---

## 17.7 CRD & Operator Debugging

### Reconciliation Loop

```
┌────────────────────────────────────────────────────────────┐
│            Operator Reconciliation Loop                     │
└────────────────────────────────────────────────────────────┘

Custom Resource Created/Updated
    │
    ▼
┌─────────────────────┐
│  Watch Trigger      │  ← Operator watches CR changes
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Reconcile Loop     │
│                     │
│  1. Get desired     │  ← Read CR spec
│  2. Get current     │  ← Read cluster state
│  3. Compare         │  ← Diff desired vs current
│  4. Take action     │  ← Create/update/delete resources
└──────────┬──────────┘
           │
           ├─→ Success → Update CR status
           │             Requeue after X seconds
           │
           └─→ Error → Update CR status
                        Exponential backoff retry

Finalizers prevent deletion until cleanup complete
```

### CRD Debugging

```bash
# List CRDs
kubectl get crd

# Describe CRD
kubectl describe crd certificates.cert-manager.io

# Check CRD versions
kubectl get crd certificates.cert-manager.io \
  -o jsonpath='{.spec.versions[*].name}'

# Get custom resources
kubectl get certificates -A

# Describe custom resource
kubectl describe certificate my-cert -n default

# Check status field (operator updates this)
kubectl get certificate my-cert -o jsonpath='{.status}'

# Check conditions
kubectl get certificate my-cert \
  -o jsonpath='{.status.conditions[*].type}'
```

### Finalizer Stuck Issues

```bash
# Custom resource stuck in Terminating
kubectl get certificate my-cert
# NAME      READY   SECRET     AGE
# my-cert   False              5m  (Terminating)

# Check for finalizers
kubectl get certificate my-cert -o jsonpath='{.metadata.finalizers}'
# ["cert-manager.io/certificate"]

# Operator not running or failing to cleanup
# Check operator logs
kubectl logs -n cert-manager deploy/cert-manager

# If operator is gone/broken, manually remove finalizer (dangerous!)
kubectl patch certificate my-cert -p '{"metadata":{"finalizers":[]}}' \
  --type=merge

# Better: Fix operator and let it cleanup properly
```

### cert-manager Debugging Chain

```
Certificate (user creates)
    │
    ▼
CertificateRequest (cert-manager creates)
    │
    ▼
Order (ACME issuer creates)
    │
    ▼
Challenge (Order creates)
    │
    ▼
Ingress/Service (Challenge creates for HTTP-01/DNS-01)
```

```bash
# Debug certificate issue
kubectl get certificate my-cert -o yaml

# Check status.conditions for errors
kubectl describe certificate my-cert

# Example error in conditions:
# Type:    Ready
# Status:  False
# Reason:  Failed
# Message: Certificate issuance failed: Order not valid

# Step 1: Check CertificateRequest
kubectl get certificaterequest
kubectl describe certificaterequest my-cert-xxxxx

# Step 2: Check Order (ACME only)
kubectl get order
kubectl describe order my-cert-xxxxx-yyyyy

# Step 3: Check Challenge
kubectl get challenge
kubectl describe challenge my-cert-xxxxx-yyyyy-zzzzz

# Common issues at each level:
# Certificate: Wrong issuer, invalid domain
# CertificateRequest: Issuer not ready
# Order: ACME authorization failed
# Challenge: DNS not propagated, HTTP-01 path not reachable
```

**Example: HTTP-01 Challenge Failure**

```bash
# Challenge stuck in Pending
kubectl get challenge
# NAME                           STATE     DOMAIN           AGE
# my-cert-xxxxx-yyyyy-zzzzz      pending   example.com      5m

kubectl describe challenge my-cert-xxxxx-yyyyy-zzzzz
# Events:
#   Presented challenge using HTTP-01
#   Waiting for HTTP-01 challenge propagation

# Verify HTTP-01 endpoint is reachable
curl http://example.com/.well-known/acme-challenge/<token>

# If unreachable:
# - Check Ingress configuration
# - Check Service/Pod for solver
# - Check NetworkPolicies blocking solver
```

### external-dns Debugging

```bash
# Check external-dns logs
kubectl logs -n kube-system -l app=external-dns

# Common log messages:
# "msg"="All records are already up to date" ← Working correctly
# "msg"="Failed to list Ingresses" ← RBAC issue
# "msg"="failed to create record" ← DNS provider API error

# Check which resources external-dns watches
# (Ingress, Service type LoadBalancer, etc.)

# Verify Ingress has hostname
kubectl get ingress my-ingress -o jsonpath='{.spec.rules[*].host}'

# Check external-dns annotations
kubectl get ingress my-ingress -o yaml | grep external-dns
# external-dns.alpha.kubernetes.io/hostname: example.com
# external-dns.alpha.kubernetes.io/ttl: "300"

# Check DNS provider records match
# Example for Route53:
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC

# Force sync (delete and recreate external-dns pod)
kubectl delete pod -n kube-system -l app=external-dns
```

### Operator Debugging

```bash
# Check operator pod
kubectl get pods -n operator-namespace -l app=my-operator

# Check operator logs
kubectl logs -n operator-namespace -l app=my-operator --tail=100 -f

# Look for reconciliation errors:
# "error"="failed to reconcile" resource="my-resource"
# "error"="timeout waiting for condition"

# Check operator RBAC
kubectl auth can-i create deployments \
  --as=system:serviceaccount:operator-namespace:operator-sa

# Check operator webhook (if applicable)
kubectl get validatingwebhookconfigurations | grep my-operator
kubectl get mutatingwebhookconfigurations | grep my-operator

# Increase operator log verbosity (if supported)
kubectl set env deployment/my-operator -n operator-namespace \
  LOG_LEVEL=debug
```

### Custom Resource Status Debugging

```yaml
# Well-designed CRs have informative status
apiVersion: example.com/v1
kind: MyApp
metadata:
  name: my-app
spec:
  replicas: 3
  image: my-image:v1
status:
  conditions:
  - type: Ready
    status: "False"
    reason: DeploymentNotReady
    message: "Waiting for deployment rollout"
    lastTransitionTime: "2024-01-01T12:00:00Z"
  - type: Deployed
    status: "True"
    reason: DeploymentCreated
    message: "Deployment created successfully"
  observedGeneration: 2  # Matches metadata.generation when synced
  replicas: 3
  readyReplicas: 2
```

```bash
# Check if operator has reconciled latest spec
kubectl get myapp my-app -o jsonpath='{.metadata.generation}'
# 3

kubectl get myapp my-app -o jsonpath='{.status.observedGeneration}'
# 2  ← Operator hasn't reconciled generation 3 yet

# Wait for reconciliation
kubectl wait --for=condition=Ready myapp/my-app --timeout=60s
```

---

## 17.8 Debugging Tools Reference

### kubectl debug with Ephemeral Containers

```bash
# Kubernetes 1.23+ supports ephemeral debug containers
# Add debugging container to running pod

kubectl debug my-pod -it --image=busybox --target=my-container

# Example: Debug distroless container (no shell)
kubectl debug distroless-pod -it --image=busybox \
  --target=app-container -- sh

# Inside debug container, can access target container's process namespace
ps aux  # See all processes including target container

# Debug with specific tools
kubectl debug my-pod -it --image=nicolaka/netshoot

# Inside netshoot container:
# - curl, wget, netcat
# - dig, nslookup, host
# - tcpdump, iftop
# - traceroute, mtr
# - iperf3

# Create copy of pod for debugging
kubectl debug my-pod --copy-to=my-pod-debug --container=debug \
  --image=busybox -- sh

# Create node debugger pod
kubectl debug node/worker-1 -it --image=ubuntu
# Mounts node's / at /host
chroot /host
```

### netshoot - Network Debugging

```bash
# Run netshoot pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Inside netshoot:

# Test DNS
nslookup kubernetes.default.svc.cluster.local
dig +short my-service.default.svc.cluster.local

# Test connectivity
curl http://my-service:8080/health
wget -O- http://my-service:8080

# TCP connection test
nc -zv my-service 8080

# Trace route
traceroute my-service
mtr my-service  # Combines ping + traceroute

# Capture packets
tcpdump -i eth0 port 8080

# HTTP load test
ab -n 1000 -c 10 http://my-service/

# Network performance test
iperf3 -c my-service
```

### k9s - Terminal UI

```bash
# Install k9s
brew install k9s  # macOS
# or download from https://github.com/derailed/k9s/releases

# Launch k9s
k9s

# Key commands:
# :pods          - View pods
# :deployments   - View deployments
# :services      - View services
# :contexts      - Switch context
# /              - Filter resources
# l              - View logs
# d              - Describe resource
# e              - Edit resource
# <ctrl-k>       - Delete resource
# s              - Shell into pod
# <shift-f>      - Port forward
# :xray          - View resource dependencies
# :pulse         - View cluster metrics
# ?              - Help

# View logs with context
# Select pod → press 'l' → view logs with timestamps

# Port forward
# Select pod/service → press <shift-f> → enter local:remote ports
```

### stern - Multi-Pod Log Streaming

```bash
# Install stern
brew install stern  # macOS

# Stream logs from all pods matching label
stern -l app=my-app

# Stream logs from multiple pods with regex
stern my-app  # Matches pod names containing "my-app"

# Include pod name and container
stern my-app --template '{{.PodName}}/{{.ContainerName}}: {{.Message}}{{"\n"}}'

# Stream from specific namespace
stern my-app -n production

# Stream from all namespaces
stern my-app --all-namespaces

# Exclude containers
stern my-app --exclude-container istio-proxy

# Stream with color coding
stern my-app --color always

# Follow new pods
stern my-app --since 1h  # Only logs from last hour

# Combine with grep
stern my-app | grep ERROR
```

### kubectl Plugins

| Plugin | Purpose | Installation | Usage Example |
|--------|---------|--------------|---------------|
| **kubectl-tree** | Show resource hierarchy | `kubectl krew install tree` | `kubectl tree deployment my-app` |
| **kubectl-slice** | Split multi-resource YAML | `kubectl krew install slice` | `kubectl slice -f all.yaml -o output/` |
| **kubectl-images** | List images in cluster | `kubectl krew install images` | `kubectl images -A` |
| **kubectl-who-can** | RBAC query tool | `kubectl krew install who-can` | `kubectl who-can create pods` |
| **kubectl-neat** | Clean kubectl output | `kubectl krew install neat` | `kubectl get pod -o yaml \| kubectl neat` |
| **kubectl-ctx** | Switch contexts | `kubectl krew install ctx` | `kubectl ctx staging` |
| **kubectl-ns** | Switch namespaces | `kubectl krew install ns` | `kubectl ns production` |
| **kubectl-df-pv** | Show PV disk usage | `kubectl krew install df-pv` | `kubectl df-pv` |
| **kubectl-resource-capacity** | Node resource summary | `kubectl krew install resource-capacity` | `kubectl resource-capacity` |
| **kubectl-cost** | Show pod costs | `kubectl krew install cost` | `kubectl cost -n production` |

```bash
# Install krew (kubectl plugin manager)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Add to PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Example: kubectl tree
kubectl krew install tree
kubectl tree deployment my-app

# Output shows resource hierarchy:
# NAMESPACE  NAME                                     READY  STATUS
# default    Deployment/my-app                        -
# default    ├─ReplicaSet/my-app-5c7d9f8b6           -
# default    │ ├─Pod/my-app-5c7d9f8b6-abc12          True
# default    │ ├─Pod/my-app-5c7d9f8b6-def34          True
# default    │ └─Pod/my-app-5c7d9f8b6-ghi56          True
```

### Tool Selection Guide

| Scenario | Recommended Tool | Command |
|----------|------------------|---------|
| Quick pod status check | kubectl | `kubectl get pods -o wide` |
| Detailed pod debugging | kubectl describe | `kubectl describe pod <name>` |
| Interactive cluster browsing | k9s | `k9s` |
| Multi-pod log streaming | stern | `stern -l app=myapp` |
| Single pod logs | kubectl logs | `kubectl logs -f <pod>` |
| Network debugging | kubectl debug + netshoot | `kubectl debug <pod> --image=nicolaka/netshoot` |
| RBAC permission check | kubectl auth can-i | `kubectl auth can-i create pods` |
| RBAC permission query | kubectl who-can | `kubectl who-can create deployments` |
| Resource hierarchy | kubectl tree | `kubectl tree deployment <name>` |
| Node resource usage | kubectl resource-capacity | `kubectl resource-capacity --util` |
| PV disk usage | kubectl df-pv | `kubectl df-pv` |
| Template validation | helm template | `helm template <release> <chart>` |
| Webhook debugging | kubectl describe | `kubectl describe validatingwebhookconfiguration` |
| Operator debugging | kubectl logs | `kubectl logs -n <ns> -l app=<operator>` |
| CRD status check | kubectl get + describe | `kubectl describe <crd-type> <name>` |

### Advanced Debugging Script

```bash
#!/bin/bash
# debug-pod.sh - Comprehensive pod debugging

POD_NAME=$1
NAMESPACE=${2:-default}

echo "=== Pod Status ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o wide

echo -e "\n=== Pod Events ==="
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD_NAME \
  --sort-by=.metadata.creationTimestamp

echo -e "\n=== Pod Describe ==="
kubectl describe pod $POD_NAME -n $NAMESPACE

echo -e "\n=== Container Status ==="
kubectl get pod $POD_NAME -n $NAMESPACE \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'

echo -e "\n=== Recent Logs ==="
kubectl logs $POD_NAME -n $NAMESPACE --tail=50

echo -e "\n=== Previous Logs (if restarted) ==="
kubectl logs $POD_NAME -n $NAMESPACE --previous --tail=50 2>/dev/null || echo "No previous logs"

echo -e "\n=== Resource Requests/Limits ==="
kubectl get pod $POD_NAME -n $NAMESPACE \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.resources}{"\n"}{end}'

echo -e "\n=== Node Resources ==="
NODE=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
kubectl describe node $NODE | grep -A 5 "Allocated resources"

echo -e "\n=== Service Endpoints ==="
kubectl get endpoints -n $NAMESPACE | grep $(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.metadata.labels.app}')
```

Usage:

```bash
chmod +x debug-pod.sh
./debug-pod.sh my-pod production
```

---

## Chapter 18 Review Questions

1. A pod is stuck in `CrashLoopBackOff` state. You run `kubectl logs my-pod` but see "Error from server (BadRequest): previous terminated container not found". What is the likely issue and how would you debug it?

2. Your application pods are stuck in `Pending` state with the event message "0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had untolerated taint". Describe the systematic approach to diagnose and resolve this issue. What kubectl commands would you use?

3. A PersistentVolumeClaim has been in `Pending` state for 10 minutes. List five potential causes and the specific kubectl commands you would use to diagnose each one.

4. You receive "Forbidden: error when creating pod" from a ValidatingWebhookConfiguration. The webhook service appears to be running. How would you debug this issue? Include specific commands to check webhook configuration, certificates, and service connectivity.

5. A cert-manager Certificate resource shows `Ready=False` with the message "Order not valid". Describe the complete debugging chain from Certificate to Challenge, including the kubectl commands to inspect each resource and common issues at each level.

---

## Chapter 18 Hands-On Exercises

### Exercise 1: Multi-Level Debugging

Create a deployment with intentional issues and debug systematically:

```bash
# Create problematic deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:nonexistent-tag
        resources:
          requests:
            memory: "100Gi"  # Intentionally too large
            cpu: "50"
        volumeMounts:
        - name: config
          mountPath: /etc/config
      volumes:
      - name: config
        configMap:
          name: missing-config  # Doesn't exist
EOF

# Tasks:
# 1. Identify all issues preventing pod from running
# 2. Document each issue with evidence from kubectl describe/logs
# 3. Fix each issue one by one
# 4. Verify deployment is healthy

# Expected issues:
# - Image pull error
# - Resource requests too large for nodes
# - ConfigMap not found
```

### Exercise 2: RBAC Debugging

Create a service account with insufficient permissions and debug:

```bash
# Create namespace and service account
kubectl create namespace rbac-test
kubectl create serviceaccount app-sa -n rbac-test

# Create pod using service account
kubectl run test-pod --image=nginx -n rbac-test \
  --serviceaccount=app-sa

# Exec into pod and try to access API
kubectl exec -it test-pod -n rbac-test -- sh
# Inside pod:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/rbac-test/pods
# Expected: Forbidden

# Tasks:
# 1. Use kubectl auth can-i to verify service account permissions
# 2. Create Role allowing list/get pods in rbac-test namespace
# 3. Create RoleBinding binding app-sa to Role
# 4. Verify service account can now list pods
# 5. Verify service account still cannot list pods in other namespaces
```

### Exercise 3: Operator & CRD Debugging

Install cert-manager and debug a certificate issue:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/cert-manager -n cert-manager

# Create ClusterIssuer (with intentional error)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: invalid-email  # Should be valid email
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx-nonexistent  # Ingress class doesn't exist
EOF

# Create certificate
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-cert
  namespace: default
spec:
  secretName: test-cert-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
  - example.invalid  # Invalid domain
EOF

# Tasks:
# 1. Check Certificate status and identify why it's not ready
# 2. Follow the debugging chain: Certificate → CertificateRequest → Order → Challenge
# 3. Identify issues in ClusterIssuer configuration
# 4. Check cert-manager logs for errors
# 5. Document each issue and the kubectl command that revealed it
```

---

## Key Takeaways

- **Systematic debugging starts with understanding pod lifecycle states** - Pending, Running, CrashLoopBackOff, and Terminating each indicate different issue categories. Use `kubectl describe pod` as the first debugging step to see events and status conditions.

- **Scheduler failures require checking three areas** - Node resources (CPU/memory capacity), node selectors/affinity rules, and taints/tolerations. Use `kubectl describe node` to see allocated resources and taints that prevent scheduling.

- **Storage debugging follows PVC lifecycle** - Verify storage class exists, capacity matches request, access modes are compatible, and CSI drivers are running. Check VolumeAttachment resources for attachment issues.

- **RBAC debugging uses "kubectl auth can-i" as primary tool** - Test permissions systematically for different users/service accounts. Check both Role/RoleBinding (namespace-scoped) and ClusterRole/ClusterRoleBinding (cluster-wide).

- **Admission webhooks fail fast with clear error messages** - Check webhook service endpoints, certificate validity, and timeout configuration. Use `failurePolicy: Ignore` temporarily for debugging but never in production for critical policies.

- **Helm debugging separates template rendering from cluster state** - Use `helm template --debug` to catch YAML/template errors, then `--dry-run=server` to catch admission webhook and validation errors before actual deployment.

- **Operator debugging requires understanding reconciliation loops** - Check CR status.conditions for operator-reported issues, examine operator pod logs for reconciliation errors, and verify finalizers aren't blocking deletion. Tools like `kubectl tree` show resource hierarchy created by operators.

[Next Chapter: Kubernetes Networking Deep Dive →](18-networking.md)
