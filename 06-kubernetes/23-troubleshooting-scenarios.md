# Chapter 23: Kubernetes Troubleshooting Scenarios

> **Prerequisites**: Chapter 12 (Containers & Kubernetes), Chapter 17 (Monitoring & Observability), Chapter 18 (Debugging Deep Dive), Chapter 19 (Networking)

This chapter presents real-world Kubernetes troubleshooting scenarios organized by category. Each scenario follows a structured format: **Symptoms → Investigation → Root Cause → Resolution → Prevention**. Use these as runbooks during incidents and as study material for building diagnostic intuition.

---

## 22.1 Troubleshooting Methodology for Kubernetes

### Layer Model

```
┌─────────────────────────────────────────────────────┐
│ Layer 7: Application          (logs, errors, traces)│
├─────────────────────────────────────────────────────┤
│ Layer 6: Service Mesh/Ingress (routing, TLS, policy)│
├─────────────────────────────────────────────────────┤
│ Layer 5: Kubernetes Services  (endpoints, DNS, LB)  │
├─────────────────────────────────────────────────────┤
│ Layer 4: Pod & Container      (scheduling, runtime) │
├─────────────────────────────────────────────────────┤
│ Layer 3: Cluster Network      (CNI, policies, kube- │
│                                proxy)               │
├─────────────────────────────────────────────────────┤
│ Layer 2: Node                 (kubelet, OS, disk)   │
├─────────────────────────────────────────────────────┤
│ Layer 1: Infrastructure       (cloud, hardware, net)│
└─────────────────────────────────────────────────────┘

Debug top-down (application first) for user-reported issues.
Debug bottom-up (infrastructure first) for cluster-wide issues.
```

### First-Response Triage Commands

```bash
# 1. Cluster health overview
kubectl get nodes
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl top nodes

# 2. Recent events (often reveals the root cause quickly)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# 3. Component health
kubectl get componentstatuses 2>/dev/null
kubectl get --raw /healthz

# 4. Namespace-specific triage
kubectl get all -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# 5. Resource pressure
kubectl describe nodes | grep -E "Conditions:|MemoryPressure|DiskPressure|PIDPressure" -A 1
```

### The Five Whys in Kubernetes

```
Symptom: Users report 500 errors
  Why? → Backend pods are returning errors
  Why? → Pods can't connect to database
  Why? → Database pod was evicted
  Why? → Node ran out of memory
  Why? → A memory leak in another pod consumed all node memory

Root cause: Missing memory limits on a noisy neighbor pod
Fix: Set resource limits, add ResourceQuota to namespace
```

---

## 22.2 Workload Scenarios

### Scenario: Stuck Deployment Rollout

**Symptoms**: `kubectl rollout status` hangs. New pods are Pending or CrashLoopBackOff. Old pods still running.

```bash
# Investigation
kubectl rollout status deployment/<name> -n <namespace>
kubectl get replicaset -n <namespace> -l app=<name>
kubectl describe deployment <name> -n <namespace>

# Check new ReplicaSet's pods
kubectl get pods -n <namespace> -l app=<name> --sort-by='.metadata.creationTimestamp'
kubectl describe pod <newest-pod> -n <namespace>
```

**Root Cause A: New image fails to start**
```bash
# Pods in CrashLoopBackOff or ImagePullBackOff
kubectl logs <new-pod> --previous
# Fix: Roll back
kubectl rollout undo deployment/<name>
# Then fix the image and redeploy
```

**Root Cause B: Insufficient resources for new pods**
```bash
# Pods stuck in Pending
kubectl describe pod <pending-pod> | grep -A 5 Events
# "0/N nodes are available: insufficient cpu/memory"

# Fix: Scale down, increase node pool, or reduce resource requests
kubectl scale deployment <name> --replicas=<fewer>
```

**Root Cause C: maxUnavailable=0 and readiness probe failing**
```bash
# Old pods can't be removed, new pods never become Ready
kubectl get pods -l app=<name> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Fix: Check readiness probe configuration
kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'
```

**Prevention**: Set `progressDeadlineSeconds` on deployments. Configure proper readiness probes. Use `kubectl rollout status --timeout` in CI/CD.

---

### Scenario: Pod Eviction Storm

**Symptoms**: Multiple pods terminated simultaneously. Events show "Evicted" status. Application availability drops.

```bash
# Investigation
kubectl get pods -A --field-selector=status.phase=Failed -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.reason}{"\n"}{end}' | grep Evicted

# Check which node triggered evictions
kubectl get events -A --field-selector reason=Evicted --sort-by='.lastTimestamp'

# Check node conditions
kubectl describe node <node> | grep -A 10 Conditions
```

**Root Cause: Node under memory or disk pressure**
```bash
# Check node resources
kubectl top node <node>
kubectl describe node <node> | grep -A 5 "Allocated resources"

# Find the memory-hungry pods
kubectl top pods -A --sort-by=memory | head -20
```

**Resolution**:
```bash
# 1. Identify and fix the resource hog
kubectl top pods -n <namespace> --sort-by=memory
# Set proper memory limits on the offending workload

# 2. Clean up evicted pods
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted | awk '{print $1, $2}' | xargs -L1 kubectl delete pod -n

# 3. If disk pressure: clean up images and logs
crictl rmi --prune
journalctl --vacuum-size=500M
```

**Prevention**: Set resource limits on all pods. Use LimitRange for namespace defaults. Monitor node resource utilization and alert at 80%.

---

### Scenario: CronJob Failures

**Symptoms**: CronJob not running on schedule, or jobs failing silently.

```bash
# Investigation
kubectl get cronjob <name> -n <namespace>
kubectl describe cronjob <name> -n <namespace>

# Check recent job history
kubectl get jobs -n <namespace> -l job-name=<cronjob-name> --sort-by='.status.startTime'

# Check failed jobs
kubectl get jobs -n <namespace> --field-selector=status.successful=0

# Check pod logs from latest job
kubectl logs job/<job-name> -n <namespace>
```

**Root Cause A: CronJob suspended or schedule wrong**
```bash
# Check if suspended
kubectl get cronjob <name> -o jsonpath='{.spec.suspend}'

# Check schedule syntax
kubectl get cronjob <name> -o jsonpath='{.spec.schedule}'
# Reminder: Kubernetes cron uses UTC
```

**Root Cause B: concurrencyPolicy blocking new jobs**
```bash
# If concurrencyPolicy=Forbid and previous job still running:
kubectl get jobs -l job-name=<cronjob> | grep Running
# The CronJob won't create new jobs until the running one finishes

# Fix: Set activeDeadlineSeconds on the Job spec to prevent runaway jobs
```

**Root Cause C: startingDeadlineSeconds missed**
```bash
# If the controller was down and missed the schedule window:
# CronJob won't run if more than startingDeadlineSeconds have passed
# Default: no deadline (will run on next reconciliation)
# If set too low: jobs are skipped when controller has brief downtime
```

**Prevention**: Set `activeDeadlineSeconds` on jobs. Set `backoffLimit` appropriately. Monitor CronJob last successful run time with Prometheus.

---

### Scenario: StatefulSet Pod Stuck Terminating

**Symptoms**: A StatefulSet pod is stuck in Terminating state for extended period. Replacement pod cannot start.

```bash
# Investigation
kubectl get pods -n <namespace> -l app=<statefulset>
kubectl describe pod <pod-name> -n <namespace>

# Check finalizers
kubectl get pod <pod-name> -o jsonpath='{.metadata.finalizers}'

# Check if the node is reachable
kubectl get node <node-name>
```

**Root Cause A: Node is unreachable**
```bash
# Pod is on a node that's NotReady
# Kubernetes won't force-delete the pod (data safety for StatefulSets)
kubectl get pod <pod> -o jsonpath='{.spec.nodeName}'
kubectl get node <node-name>

# If node will not recover, force delete:
kubectl delete pod <pod-name> --grace-period=0 --force
# WARNING: This may cause data corruption if pod is still running on the unreachable node
```

**Root Cause B: Finalizer blocking deletion**
```bash
# Remove the finalizer
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":null}}'
```

**Root Cause C: preStop hook hanging**
```bash
# Check if preStop hook is taking too long
kubectl describe pod <pod> | grep -A 5 "preStop"
# The pod will be SIGKILL'd after terminationGracePeriodSeconds (default: 30s)
# If still stuck, the node may have issues
```

**Prevention**: Set reasonable `terminationGracePeriodSeconds`. Ensure preStop hooks have timeouts. Monitor for pods stuck in Terminating state.

---

### Scenario: DaemonSet Not Scheduling on All Nodes

**Symptoms**: DaemonSet pod count is less than number of nodes.

```bash
# Investigation
kubectl get daemonset <name> -n <namespace>
# Compare DESIRED vs CURRENT vs READY

kubectl get pods -n <namespace> -l app=<daemonset> -o wide
# Check which nodes are missing pods

# Check for scheduling failures
kubectl describe daemonset <name> -n <namespace> | grep -A 20 Events
```

**Root Cause A: Node taints not tolerated**
```bash
# Check taints on the missing nodes
kubectl describe node <missing-node> | grep Taints

# Check DaemonSet tolerations
kubectl get daemonset <name> -o jsonpath='{.spec.template.spec.tolerations}'

# Fix: Add toleration to DaemonSet
```

**Root Cause B: nodeSelector or affinity excluding nodes**
```bash
kubectl get daemonset <name> -o jsonpath='{.spec.template.spec.nodeSelector}'
kubectl get daemonset <name> -o jsonpath='{.spec.template.spec.affinity}'

# Fix: Adjust selector/affinity or add labels to nodes
```

**Root Cause C: Insufficient resources on specific nodes**
```bash
kubectl describe node <missing-node> | grep -A 10 "Allocated resources"
# DaemonSet pods compete for resources like any other pod
```

**Prevention**: Document required tolerations for cluster-wide DaemonSets. Use `kubectl get daemonset` monitoring to alert on desired != ready.

---

## 22.3 Networking Scenarios

### Scenario: Intermittent Connection Refused

**Symptoms**: Services intermittently return "connection refused" errors. Works sometimes, fails sometimes.

```bash
# Investigation
# Check if all endpoints are healthy
kubectl get endpoints <service> -n <namespace>

# Check readiness of all pods
kubectl get pods -l app=<name> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Check for pods cycling (restarting)
kubectl get pods -l app=<name> -w   # Watch for restarts
```

**Root Cause A: Pod restarting — briefly in endpoints while not ready**
```bash
# A pod passes readiness probe, gets added to endpoints, then crashes
# Between restart and next probe: traffic is sent to non-ready pod

# Fix: Tune readiness probe
# - Increase initialDelaySeconds
# - Decrease periodSeconds for faster removal
# - Add failureThreshold: 1 for quick removal on failure
```

**Root Cause B: Application listening on wrong address**
```bash
# App binds to 127.0.0.1 instead of 0.0.0.0
kubectl exec <pod> -- ss -tlnp
# Check LISTEN address — must be 0.0.0.0 or :: for pod networking

# Fix: Configure application to bind to 0.0.0.0
```

**Root Cause C: kube-proxy iptables rules stale**
```bash
# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep -i error

# Restart kube-proxy to force rule refresh
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

**Prevention**: Configure readiness probes accurately. Use `publishNotReadyAddresses: false` on services. Monitor endpoint count.

---

### Scenario: Cross-Namespace Communication Broken

**Symptoms**: Pods in namespace A cannot reach services in namespace B.

```bash
# Investigation
# Test DNS resolution
kubectl exec -n ns-a <pod> -- nslookup <service>.ns-b.svc.cluster.local

# Test connectivity
kubectl exec -n ns-a <pod> -- curl -v --connect-timeout 5 http://<service>.ns-b.svc.cluster.local:<port>

# Check if it's a DNS issue or connectivity issue
kubectl exec -n ns-a <pod> -- curl -v --connect-timeout 5 http://<service-cluster-ip>:<port>
```

**Root Cause A: Network Policy blocking cross-namespace traffic**
```bash
# Check network policies in the target namespace
kubectl get networkpolicy -n ns-b
kubectl describe networkpolicy -n ns-b

# Common issue: default-deny policy without explicit cross-namespace allow
# Fix: Add namespace selector to ingress rule
```

```yaml
# Allow traffic from ns-a to ns-b backend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ns-a
  namespace: ns-b
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ns-a
    ports:
    - port: 8080
```

**Root Cause B: Service not using correct namespace in FQDN**
```bash
# Must use: <service>.<namespace>.svc.cluster.local
# NOT just: <service> (only works within same namespace)
```

**Prevention**: Label namespaces consistently. Document cross-namespace dependencies. Test network policies in staging.

---

### Scenario: Ingress Returns 503

**Symptoms**: All or some paths behind an Ingress return HTTP 503.

```bash
# Investigation
kubectl get ingress <name> -n <namespace>
kubectl describe ingress <name> -n <namespace>

# Check backend service
kubectl get endpoints <backend-service> -n <namespace>

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller | grep "503\|error\|upstream"
```

**Root Cause A: No healthy endpoints**
```bash
# Endpoints list is empty
kubectl get endpoints <service>
# NAME      ENDPOINTS
# myapp     <none>

# Check: Are pods running?
kubectl get pods -l app=<name>
# Check: Do pod labels match service selector?
kubectl get svc <service> -o jsonpath='{.spec.selector}'

# Fix: Ensure pods exist, are Running, and labels match service selector
```

**Root Cause B: Readiness probe failing**
```bash
# Pods are Running but not Ready — excluded from endpoints
kubectl get pods -l app=<name>
# STATUS: Running but READY: 0/1

# Check readiness probe
kubectl describe pod <pod> | grep -A 10 "Readiness"
kubectl logs <pod>  # Look for startup errors

# Fix: Fix the readiness probe or the application health endpoint
```

**Root Cause C: Backend service port mismatch**
```bash
# Ingress references port 80, but service is on port 8080
kubectl get ingress <name> -o jsonpath='{.spec.rules[0].http.paths[0].backend}'
kubectl get svc <service> -o jsonpath='{.spec.ports}'

# Fix: Align ingress backend port with service port
```

**Prevention**: Monitor endpoint count per service. Alert when endpoints drop to zero. Use proper readiness probes.

---

### Scenario: Slow DNS Resolution

**Symptoms**: Application requests have high latency. DNS lookups take hundreds of milliseconds or timeout.

```bash
# Investigation
# Measure DNS resolution time
kubectl exec <pod> -- time nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl top pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "error\|timeout\|refused"
```

**Root Cause A: CoreDNS overwhelmed**
```bash
# High CPU/memory on CoreDNS pods
kubectl top pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS metrics
kubectl exec -n kube-system <coredns-pod> -- wget -qO- http://localhost:9153/metrics | grep coredns_dns_request_duration_seconds

# Fix: Scale CoreDNS
kubectl scale deployment coredns -n kube-system --replicas=4
# Or use NodeLocal DNSCache
```

**Root Cause B: ndots:5 causing excessive queries**
```bash
# Every external hostname generates 4-5 extra DNS queries
kubectl exec <pod> -- cat /etc/resolv.conf
# options ndots:5

# Fix: Reduce ndots or use FQDNs with trailing dot in application config
```

**Root Cause C: Conntrack table exhaustion on nodes**
```bash
# DNS uses UDP, conntrack entries pile up
# On the node:
# conntrack -C
# cat /proc/sys/net/netfilter/nf_conntrack_max

# Fix: Increase conntrack max or deploy NodeLocal DNSCache
# (NodeLocal DNS uses TCP to upstream, reducing conntrack pressure)
```

**Prevention**: Deploy NodeLocal DNSCache. Scale CoreDNS based on cluster size. Reduce ndots for external-heavy workloads. Monitor DNS latency.

---

## 22.4 Storage Scenarios

### Scenario: PVC Stuck in Pending

**Symptoms**: PVC remains in Pending state. Pods using the PVC also stay Pending.

```bash
# Investigation
kubectl describe pvc <name> -n <namespace>
# Look at Events section for specific error

kubectl get storageclass
kubectl get pv
```

**Root Cause A: StorageClass not found**
```bash
# PVC references non-existent StorageClass
kubectl get pvc <name> -o jsonpath='{.spec.storageClassName}'
kubectl get storageclass

# Fix: Create the StorageClass or update PVC to use an existing one
```

**Root Cause B: WaitForFirstConsumer (expected behavior)**
```bash
# StorageClass has volumeBindingMode: WaitForFirstConsumer
kubectl get storageclass <sc> -o jsonpath='{.volumeBindingMode}'
# PVC intentionally waits until a pod using it is scheduled
# This is NORMAL — PVC will bind when a pod is created that references it
```

**Root Cause C: Cloud provider quota exceeded**
```bash
# CSI driver can't provision the volume
kubectl describe pvc <name> | grep -A 5 Events
# Look for: "failed to provision volume with StorageClass"
# Check cloud provider quotas for disk resources
```

**Root Cause D: No matching PV (static provisioning)**
```bash
# For static provisioning, a PV must exist with matching:
# - storageClassName
# - access modes
# - capacity >= request
kubectl get pv -o wide
```

**Prevention**: Use dynamic provisioning. Monitor PVC provisioning time. Set default StorageClass.

---

### Scenario: "Multi-Attach Error" on Volume

**Symptoms**: Pod stuck in ContainerCreating with "Multi-Attach error for volume".

```bash
# Investigation
kubectl describe pod <pod> | grep -A 5 "Multi-Attach"
# "Multi-Attach error for volume 'pvc-xxx': Volume is already
#  attached to node 'node-a' and can't be attached to node 'node-b'"
```

**Root Cause: ReadWriteOnce volume cannot attach to multiple nodes**
```bash
# Check access mode
kubectl get pv <pv-name> -o jsonpath='{.spec.accessModes}'
# ReadWriteOnce — can only mount on one node

# This happens when:
# 1. Pod was rescheduled to a different node
# 2. Old pod hasn't fully terminated
# 3. Volume didn't detach from old node

# Check volume attachment
kubectl get volumeattachment | grep <pv-name>
```

**Resolution**:
```bash
# Option 1: Wait for old pod to fully terminate (may take minutes)
kubectl get pods -o wide | grep <old-pod>

# Option 2: If old pod is gone but attachment lingers
kubectl delete volumeattachment <va-name>
# WARNING: Only if you're sure the old pod is truly gone

# Option 3: Force pod to schedule on same node
# Add nodeAffinity or nodeSelector to match the volume's node
```

**Prevention**: Use ReadWriteMany volumes for multi-node access (NFS, CephFS). Set appropriate `terminationGracePeriodSeconds`. For StatefulSets, ensure orderly pod replacement.

---

### Scenario: Data Loss After StatefulSet Scale Down

**Symptoms**: After scaling a StatefulSet down and back up, data is missing.

```bash
# Investigation
kubectl get pvc -l app=<statefulset>
# PVCs from scaled-down pods still exist!

kubectl get pv | grep <statefulset>
# PVs are Released or Available

# Check reclaim policy
kubectl get pv <pv-name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

**Root Cause: PV reclaim policy set to Delete**
```bash
# When PVC is deleted, PV with Delete policy is also deleted
# Data is permanently lost

# StorageClass default reclaimPolicy:
kubectl get storageclass <sc> -o jsonpath='{.reclaimPolicy}'
# "Delete" is the default for most dynamic provisioners
```

**Resolution**: Data recovery depends on cloud provider. Check for volume snapshots or cloud disk snapshots.

**Prevention**:
```bash
# 1. Set reclaim policy to Retain for important data
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# 2. Create VolumeSnapshots before scaling down
# 3. Use backup solution (Velero) for StatefulSet data
# 4. Document that scaling down StatefulSets does NOT delete PVCs
#    but manually deleting PVCs WILL trigger PV deletion
```

---

## 22.5 Control Plane Scenarios

### Scenario: kubectl Commands Timing Out

**Symptoms**: All kubectl commands hang or return "connection refused" or timeout errors.

```bash
# Investigation
# Can you reach the API server?
curl -k https://<api-server>:6443/healthz

# Check kubeconfig
kubectl config current-context
kubectl config view --minify

# Is it a network issue?
nc -zv <api-server-ip> 6443
```

**Root Cause A: API server not running**
```bash
# On control plane node:
crictl ps | grep kube-apiserver
# If not running, check static pod manifest:
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Check kubelet logs for why it's not starting:
journalctl -u kubelet | grep apiserver
```

**Root Cause B: etcd unavailable**
```bash
# API server depends on etcd
etcdctl endpoint health
# If etcd is down, API server can't serve requests

# Check etcd pods/process
crictl ps | grep etcd
journalctl -u etcd
```

**Root Cause C: Certificate expired**
```bash
# Check API server certificate
openssl s_client -connect <api-server>:6443 </dev/null 2>/dev/null | openssl x509 -noout -dates
# If expired: renew with kubeadm
kubeadm certs renew all
```

**Root Cause D: Network/firewall issue**
```bash
# Control plane port blocked
nc -zv <api-server-ip> 6443
# Check cloud provider security groups/firewall rules
```

**Prevention**: Monitor API server health endpoint. Alert on certificate expiry (30 days before). Ensure etcd health monitoring.

---

### Scenario: Scheduling Failures After Cluster Upgrade

**Symptoms**: After upgrading the cluster, new pods stay Pending with scheduling errors.

```bash
# Investigation
kubectl get pods --field-selector=status.phase=Pending -A
kubectl describe pod <pending-pod>

# Check scheduler
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl logs -n kube-system -l component=kube-scheduler
```

**Root Cause A: New version removed a deprecated scheduling feature**
```bash
# Example: PodSecurityPolicy removal in 1.25
# If PSP admission controller was blocking scheduling
# Or: API changes in affinity/topology spread syntax
kubectl get pod <pod> -o yaml | grep -A 20 affinity
```

**Root Cause B: Scheduler leader election failed after upgrade**
```bash
# Check scheduler lease
kubectl get lease -n kube-system kube-scheduler-leader
kubectl logs -n kube-system -l component=kube-scheduler | grep -i "leader\|election"
# Fix: Restart scheduler pods
```

**Root Cause C: Node version skew too large**
```bash
# After control plane upgrade, nodes may be too old
kubectl get nodes -o wide
# kubelet must be within ±1 minor version of API server
# Fix: Upgrade node kubelet versions
```

**Prevention**: Review release notes before upgrading. Test upgrades in staging. Follow version skew policy strictly.

---

### Scenario: etcd Alarm — No Space

**Symptoms**: API server returns errors. etcd logs show "database space exceeded" or "no space" alarm.

```bash
# Investigation
etcdctl alarm list
# alarm:NOSPACE

etcdctl endpoint status --write-out=table
# Check DB SIZE — should be < quota (default 2GB)
```

**Resolution**:
```bash
# 1. Compact old revisions
rev=$(etcdctl endpoint status --write-out=json | jq '.[0].Status.header.revision')
etcdctl compact $rev

# 2. Defragment to reclaim space
etcdctl defrag

# 3. Disarm the alarm
etcdctl alarm disarm

# 4. Verify
etcdctl endpoint status --write-out=table
etcdctl alarm list
```

**Root Cause**: Too many Kubernetes objects, excessive event history, or auto-compaction not configured.

**Prevention**: Enable auto-compaction (`--auto-compaction-retention=5m`). Monitor etcd DB size. Alert at 80% of quota. Regularly clean up old resources (completed jobs, evicted pods, stale events).

---

## 22.6 Security Scenarios

### Scenario: RBAC Forbidden Errors

**Symptoms**: Service account or user gets "forbidden" error when accessing Kubernetes API.

```bash
# Investigation
# Error: "pods is forbidden: User "system:serviceaccount:default:myapp"
#         cannot list resource "pods" in API group "" in the namespace "production""

# Check what the SA can do
kubectl auth can-i --list --as=system:serviceaccount:default:myapp -n production

# Check bindings
kubectl get rolebinding,clusterrolebinding -A -o json | \
  jq '.items[] | select(.subjects[]?.name=="myapp") | {name: .metadata.name, namespace: .metadata.namespace, role: .roleRef.name}'
```

**Root Cause A: Missing RoleBinding**
```bash
# Role exists but no binding to the service account
kubectl get role -n production
kubectl get rolebinding -n production

# Fix: Create RoleBinding
kubectl create rolebinding myapp-pod-reader \
  --role=pod-reader \
  --serviceaccount=default:myapp \
  -n production
```

**Root Cause B: RoleBinding in wrong namespace**
```bash
# RoleBinding is in namespace "default" but pod is trying to access "production"
# RoleBindings are namespace-scoped — must be in the target namespace

# Fix: Create RoleBinding in the correct namespace
```

**Root Cause C: Using Role instead of ClusterRole for cluster-scoped resources**
```bash
# Trying to access nodes, namespaces, or other cluster-scoped resources
# These require ClusterRole + ClusterRoleBinding

# Fix: Use ClusterRole and ClusterRoleBinding
kubectl create clusterrolebinding myapp-node-reader \
  --clusterrole=node-reader \
  --serviceaccount=default:myapp
```

**Prevention**: Use least-privilege principle. Document required permissions per service account. Audit RBAC configurations regularly.

---

### Scenario: Webhook Rejecting All Pod Creation

**Symptoms**: No new pods can be created. Error: "admission webhook denied the request."

```bash
# Investigation
# Try creating a simple pod
kubectl run test --image=nginx 2>&1
# "Error: admission webhook "validate.webhook.example.com" denied the request"

# List webhooks
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# Check webhook service
kubectl describe validatingwebhookconfiguration <name>
# Look at: service, caBundle, failurePolicy, namespaceSelector
```

**Root Cause A: Webhook service is down**
```bash
# The webhook service pods are not running
kubectl get endpoints -n <webhook-namespace> <webhook-service>
# No endpoints = webhook can't respond = depends on failurePolicy

# If failurePolicy: Fail → ALL matching requests are rejected
# If failurePolicy: Ignore → requests proceed without webhook check
```

**Resolution (Emergency)**:
```bash
# Option 1: Delete the webhook configuration (fastest)
kubectl delete validatingwebhookconfiguration <name>
# This removes the webhook entirely — re-apply when service is fixed

# Option 2: Change failurePolicy to Ignore
kubectl patch validatingwebhookconfiguration <name> \
  --type='json' -p='[{"op": "replace", "path": "/webhooks/0/failurePolicy", "value": "Ignore"}]'

# Option 3: Fix the webhook service
kubectl get pods -n <webhook-namespace> -l app=<webhook>
kubectl describe pod <webhook-pod> -n <webhook-namespace>
```

**Prevention**: Set `failurePolicy: Ignore` for non-critical webhooks. Exclude `kube-system` namespace from webhooks. Monitor webhook availability and latency.

---

### Scenario: Pod Security Standards Blocking Deployment

**Symptoms**: Pods rejected with "violates PodSecurity" error when deploying to an enforced namespace.

```bash
# Investigation
kubectl get namespace <ns> --show-labels
# pod-security.kubernetes.io/enforce: restricted

# The error message tells you exactly what's wrong:
# "violates PodSecurity 'restricted:latest': allowPrivilegeEscalation != false,
#  unrestricted capabilities, runAsNonRoot != true"
```

**Resolution**:
```yaml
# Fix the pod security context to comply with restricted standard
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: app
        image: myapp:latest
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          readOnlyRootFilesystem: true
```

**Prevention**: Test deployments in warn mode before switching to enforce. Use `--dry-run=server` to check if pods will be admitted.

---

## 22.7 Scaling Scenarios

### Scenario: HPA Not Scaling Up

**Symptoms**: Application under load but HPA shows `<unknown>` for current metrics or doesn't scale.

```bash
# Investigation
kubectl get hpa <name> -n <namespace>
kubectl describe hpa <name> -n <namespace>

# Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
kubectl top pods -n <namespace>
```

**Root Cause A: Metrics not available**
```bash
# HPA shows TARGETS: <unknown>/80%
# Metrics-server not running or pods don't have resource requests

# Check: Do target pods have resource requests defined?
kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[0].resources.requests}'
# If empty, HPA can't calculate percentage utilization

# Fix: Add resource requests
kubectl set resources deployment <name> --requests=cpu=100m,memory=128Mi
```

**Root Cause B: Custom metrics not configured**
```bash
# HPA using custom metrics but custom metrics adapter not installed
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
# If 404: custom metrics API not available

# Fix: Install Prometheus adapter or KEDA
```

**Root Cause C: HPA at maxReplicas**
```bash
kubectl get hpa <name>
# REPLICAS: 10/10 (already at max)

# Fix: Increase maxReplicas or investigate why the application
# needs that many replicas (may indicate a performance problem)
```

**Prevention**: Always set resource requests on containers. Monitor HPA status and scaling events. Test autoscaling behavior before going to production.

---

### Scenario: Cluster Autoscaler Not Adding Nodes

**Symptoms**: Pods stuck Pending. HPA created new replicas but no nodes available to schedule them.

```bash
# Investigation
kubectl get pods --field-selector=status.phase=Pending
kubectl describe pod <pending-pod> | grep -A 5 Events

# Check autoscaler status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
# Or check logs:
kubectl logs -n kube-system -l app=cluster-autoscaler | tail -50
```

**Root Cause A: Max node count reached**
```bash
# Node pool is at maximum size
# Check cloud provider node pool configuration
# Fix: Increase max node count in node pool settings
```

**Root Cause B: Pod doesn't fit any node type**
```bash
# Pod requests more resources than any single node can provide
kubectl describe pod <pending-pod> | grep -A 10 "Resources"
# Fix: Use larger instance types or split workload
```

**Root Cause C: NodeSelector/affinity restricts eligible node pools**
```bash
# Pod can only run on nodes with specific labels
kubectl get pod <pending-pod> -o jsonpath='{.spec.nodeSelector}'
# Autoscaler may not know to scale the correct node pool
# Fix: Ensure autoscaler can scale the node pool that matches labels
```

**Root Cause D: Scale-down preventing new nodes**
```bash
# Autoscaler in cooldown period after a recent scale-down
kubectl logs -n kube-system -l app=cluster-autoscaler | grep "scale-down\|cooldown"
```

**Prevention**: Set appropriate max node counts. Monitor Pending pod queue. Ensure node pool instance types can accommodate largest pod requests.

---

### Scenario: Underutilized Nodes Not Scaling Down

**Symptoms**: Nodes are nearly empty but cluster autoscaler doesn't remove them.

```bash
# Investigation
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler | grep "scale-down"
```

**Root Cause A: Pods preventing scale-down**
```bash
# Autoscaler won't remove a node if it has:
# - Pods with local storage (emptyDir with data)
# - Pods not managed by a controller (bare pods)
# - Pods with PDBs that would be violated
# - Kube-system pods without PDB
# - Pods with restrictive affinity rules

# Check which pods are blocking:
kubectl logs -n kube-system -l app=cluster-autoscaler | grep "cannot remove"
```

**Root Cause B: Scale-down disabled or threshold not met**
```bash
# Default: node must be below 50% utilization for 10 minutes
# Check autoscaler parameters for custom thresholds
kubectl get configmap cluster-autoscaler -n kube-system -o yaml | grep scale-down
```

**Prevention**: Set PDBs on kube-system DaemonSet pods. Avoid bare pods. Use `cluster-autoscaler.kubernetes.io/safe-to-evict: "true"` annotation on pods that are OK to evict.

---

## 22.8 Incident Response Playbook

### Triage Script

```bash
#!/bin/bash
# k8s-triage.sh - Quick cluster health assessment
# Usage: ./k8s-triage.sh [namespace]

NS="${1:---all-namespaces}"
NS_FLAG=$([ "$NS" = "--all-namespaces" ] && echo "-A" || echo "-n $NS")

echo "=== Cluster Info ==="
kubectl cluster-info

echo -e "\n=== Node Status ==="
kubectl get nodes -o wide

echo -e "\n=== Node Resource Usage ==="
kubectl top nodes 2>/dev/null || echo "metrics-server not available"

echo -e "\n=== Unhealthy Pods ==="
kubectl get pods $NS_FLAG --field-selector=status.phase!=Running,status.phase!=Succeeded \
  -o wide 2>/dev/null | head -30

echo -e "\n=== Pods with High Restarts ==="
kubectl get pods $NS_FLAG -o json | \
  jq -r '.items[] | select(.status.containerStatuses[]?.restartCount > 5) |
  "\(.metadata.namespace)/\(.metadata.name) restarts: \(.status.containerStatuses[0].restartCount)"' \
  2>/dev/null | head -20

echo -e "\n=== Recent Events (Warnings) ==="
kubectl get events $NS_FLAG --field-selector=type=Warning \
  --sort-by='.lastTimestamp' 2>/dev/null | tail -20

echo -e "\n=== PVCs Not Bound ==="
kubectl get pvc $NS_FLAG --field-selector=status.phase!=Bound 2>/dev/null

echo -e "\n=== Services Without Endpoints ==="
kubectl get endpoints $NS_FLAG -o json | \
  jq -r '.items[] | select((.subsets // []) | length == 0) |
  "\(.metadata.namespace)/\(.metadata.name)"' 2>/dev/null

echo -e "\n=== Resource Quotas ==="
kubectl describe resourcequota $NS_FLAG 2>/dev/null | grep -E "Name:|Used|Hard" | head -30
```

### Communication Template

```
INCIDENT REPORT (Initial)
========================
Time Detected: [YYYY-MM-DD HH:MM UTC]
Severity: [P1/P2/P3/P4]
Impact: [description of user impact]
Affected Services: [service names]
Current Status: [investigating/identified/mitigating/resolved]

Initial Findings:
- [bullet point observations]

Actions Taken:
- [bullet point actions]

Next Steps:
- [what you plan to do next]

Incident Commander: [name]
```

### Post-Incident Evidence Preservation

```bash
# Capture cluster state for post-mortem BEFORE fixing things

INCIDENT_DIR="incident-$(date +%Y%m%d-%H%M)"
mkdir -p $INCIDENT_DIR

# 1. Cluster state
kubectl get all -A -o yaml > $INCIDENT_DIR/all-resources.yaml
kubectl get events -A --sort-by='.lastTimestamp' > $INCIDENT_DIR/events.txt
kubectl get nodes -o yaml > $INCIDENT_DIR/nodes.yaml

# 2. Pod details for affected workloads
for pod in $(kubectl get pods -n <namespace> -o name); do
  name=$(basename $pod)
  kubectl describe $pod -n <namespace> > $INCIDENT_DIR/describe-$name.txt
  kubectl logs $pod -n <namespace> --all-containers --previous > $INCIDENT_DIR/logs-$name.txt 2>/dev/null
done

# 3. Resource usage
kubectl top nodes > $INCIDENT_DIR/node-usage.txt 2>/dev/null
kubectl top pods -A > $INCIDENT_DIR/pod-usage.txt 2>/dev/null

# 4. Network state (if relevant)
kubectl get endpoints -A > $INCIDENT_DIR/endpoints.txt
kubectl get networkpolicy -A -o yaml > $INCIDENT_DIR/network-policies.yaml

# 5. Archive
tar -czf $INCIDENT_DIR.tar.gz $INCIDENT_DIR/
echo "Evidence preserved in $INCIDENT_DIR.tar.gz"
```

### Post-Mortem Template

```
POST-MORTEM: [Incident Title]
============================

Date: [date]
Duration: [start to resolution]
Severity: [P1/P2/P3/P4]
Authors: [names]

SUMMARY
-------
[2-3 sentence summary of what happened and impact]

TIMELINE (UTC)
--------------
HH:MM - [event]
HH:MM - [event]
HH:MM - [detection]
HH:MM - [first response action]
HH:MM - [root cause identified]
HH:MM - [mitigation applied]
HH:MM - [resolution confirmed]

ROOT CAUSE
----------
[detailed technical explanation]

IMPACT
------
- Users affected: [count/percentage]
- Duration: [minutes/hours]
- Data loss: [yes/no, details]
- SLO impact: [which SLOs were breached]

WHAT WENT WELL
--------------
- [things that helped during the incident]

WHAT WENT WRONG
---------------
- [things that made the incident worse or slower to resolve]

ACTION ITEMS
------------
| Action | Owner | Priority | Due Date |
|--------|-------|----------|----------|
| [action] | [name] | P1/P2/P3 | [date] |

LESSONS LEARNED
---------------
- [key takeaways]
```

---

## Review Questions

1. A deployment rollout is stuck at 50% progress. New pods are Running but not Ready. Old pods are still serving traffic. What systematic steps would you follow to diagnose and resolve this?

2. Multiple pods across different namespaces are being evicted simultaneously. What is the most likely cause, and how would you investigate?

3. An ingress returns 503 for one specific path but works for others. The backend pods are running. Walk through the debugging process.

4. After a cluster upgrade from 1.27 to 1.28, some CronJobs stop running. What are the likely causes and how would you investigate?

5. A ValidatingWebhookConfiguration with `failurePolicy: Fail` is blocking all deployments because the webhook service is down. What are your options for immediate remediation, and which carries the least risk?

6. The HPA shows `<unknown>` for current CPU utilization. Metrics Server is running and `kubectl top pods` works. What could cause this?

7. Describe how you would preserve evidence during a Kubernetes incident before applying fixes. Why is this important?

---

## Hands-On Exercises

### Exercise 22.1: Workload Troubleshooting Lab
1. Create a deployment with a deliberate error (wrong image, failing health check, or insufficient resources)
2. Use the triage commands from Section 22.1 to identify the issue
3. Follow the appropriate scenario's resolution steps
4. Document your investigation process

### Exercise 22.2: Network Troubleshooting Lab
1. Create two namespaces with deployments that need to communicate
2. Apply a default-deny network policy that breaks communication
3. Debug the connectivity issue using the approaches from Section 22.3
4. Create the correct network policy to restore communication while maintaining least-privilege

### Exercise 22.3: Incident Response Simulation
1. Run the triage script from Section 22.8 against your cluster
2. Intentionally break something (scale down a critical deployment, apply a bad network policy)
3. Practice the incident response workflow: detect, triage, investigate, resolve
4. Capture evidence and write a mini post-mortem

---

## Key Takeaways

1. **Use the layer model for systematic debugging** — Start at the application layer for user-reported issues, infrastructure layer for cluster-wide problems. Avoid random troubleshooting.

2. **Events tell the story** — `kubectl get events --sort-by='.lastTimestamp'` is often the fastest path to root cause. Always check events early in your investigation.

3. **Scenarios follow patterns** — Most Kubernetes failures fall into repeatable categories: scheduling, networking, storage, RBAC, and configuration. Recognize the pattern and apply the corresponding runbook.

4. **Every scenario has prevention** — The most impactful troubleshooting happens before incidents. Resource limits, readiness probes, network policies, and monitoring prevent the majority of production issues.

5. **Preserve evidence before fixing** — Capture logs, events, resource state, and metrics before applying fixes. You can't investigate a root cause after the evidence is destroyed by a restart.

6. **Automate triage** — Build triage scripts that capture cluster health quickly. Under incident pressure, having pre-written commands reduces time to diagnosis.

7. **Post-mortems drive improvement** — Each incident is a learning opportunity. Track action items to completion to prevent recurrence.

---

[Back to Contents →](../README.md)
