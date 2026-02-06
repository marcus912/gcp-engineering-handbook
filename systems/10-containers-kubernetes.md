# Chapter 10: Containers & Kubernetes

## 10.1 Container Fundamentals

### What Makes a Container

```
┌─────────────────────────────────────────────────────────────────┐
│                         Host OS                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Container Runtime                         ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         ││
│  │  │ Container A │  │ Container B │  │ Container C │         ││
│  │  │   ┌─────┐   │  │   ┌─────┐   │  │   ┌─────┐   │         ││
│  │  │   │ App │   │  │   │ App │   │  │   │ App │   │         ││
│  │  │   ├─────┤   │  │   ├─────┤   │  │   ├─────┤   │         ││
│  │  │   │Libs │   │  │   │Libs │   │  │   │Libs │   │         ││
│  │  │   └─────┘   │  │   └─────┘   │  │   └─────┘   │         ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘         ││
│  │                                                              ││
│  │  Isolation via: Namespaces (PID, Net, Mount, User, etc.)   ││
│  │  Resource limits via: cgroups                               ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Linux Namespaces

| Namespace | Isolates |
|-----------|----------|
| **PID** | Process IDs |
| **Network** | Network interfaces, IP addresses |
| **Mount** | Filesystem mount points |
| **UTS** | Hostname |
| **IPC** | Inter-process communication |
| **User** | User and group IDs |

### cgroups (Control Groups)

Limit and account for resource usage:
- CPU time
- Memory
- I/O bandwidth
- Network bandwidth

---

## 10.2 Docker Commands

### Image Management

```bash
# Pull image
docker pull nginx:1.21

# List images
docker images

# Build image
docker build -t myapp:1.0 .

# Push to registry
docker push gcr.io/project/myapp:1.0

# Remove image
docker rmi nginx:1.21
```

### Container Management

```bash
# Run container
docker run -d --name web -p 8080:80 nginx

# List containers
docker ps                 # Running
docker ps -a              # All

# Stop/start
docker stop web
docker start web
docker restart web

# Remove container
docker rm web
docker rm -f web          # Force (if running)

# Execute command in container
docker exec -it web bash
docker exec web ls /var/log

# View logs
docker logs web
docker logs -f web        # Follow
docker logs --tail 100 web

# Inspect container
docker inspect web
```

### Dockerfile Best Practices

```dockerfile
# Use specific version, not latest
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Copy dependency files first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Don't run as root
RUN useradd -m appuser
USER appuser

# Expose port (documentation)
EXPOSE 8000

# Use exec form for proper signal handling
CMD ["python", "app.py"]
```

### Multi-stage Build

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server

# Production stage
FROM alpine:3.18
COPY --from=builder /app/server /server
CMD ["/server"]
```

---

## 10.3 Container Debugging

```bash
# Check container processes
docker top container_name

# Resource usage
docker stats
docker stats container_name

# Inspect networking
docker network ls
docker network inspect bridge

# View container filesystem
docker diff container_name

# Copy files in/out
docker cp container_name:/app/logs ./logs
docker cp ./config container_name:/app/config

# Check why container exited
docker inspect container_name --format='{{.State.ExitCode}}'
docker inspect container_name --format='{{.State.Error}}'
```

---

## 10.4 Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Control Plane                           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌───────────┐  │  │
│  │  │  API     │ │  etcd    │ │ Scheduler  │ │Controller │  │  │
│  │  │  Server  │ │          │ │            │ │  Manager  │  │  │
│  │  └──────────┘ └──────────┘ └────────────┘ └───────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐     │
│  │   Node 1    │      │   Node 2    │      │   Node 3    │     │
│  │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │     │
│  │ │ kubelet │ │      │ │ kubelet │ │      │ │ kubelet │ │     │
│  │ ├─────────┤ │      │ ├─────────┤ │      │ ├─────────┤ │     │
│  │ │kube-prxy│ │      │ │kube-prxy│ │      │ │kube-prxy│ │     │
│  │ ├─────────┤ │      │ ├─────────┤ │      │ ├─────────┤ │     │
│  │ │Container│ │      │ │Container│ │      │ │Container│ │     │
│  │ │ Runtime │ │      │ │ Runtime │ │      │ │ Runtime │ │     │
│  │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │     │
│  │  [Pod][Pod] │      │  [Pod][Pod] │      │  [Pod][Pod] │     │
│  └─────────────┘      └─────────────┘      └─────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Function |
|-----------|----------|
| **API Server** | REST API, authentication, authorization |
| **etcd** | Distributed key-value store (cluster state) |
| **Scheduler** | Assigns pods to nodes |
| **Controller Manager** | Runs controllers (ReplicaSet, Deployment, etc.) |

### Node Components

| Component | Function |
|-----------|----------|
| **kubelet** | Agent that runs pods on the node |
| **kube-proxy** | Network proxy, implements Services |
| **Container Runtime** | Runs containers (containerd, CRI-O) |

---

## 10.5 Kubernetes Objects

### Pod

Smallest deployable unit (one or more containers):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Deployment

Manages ReplicaSets and provides declarative updates:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Service

Exposes pods to network:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer  # or ClusterIP, NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

### Service Types

| Type | Access | Use Case |
|------|--------|----------|
| **ClusterIP** | Internal only | Inter-service communication |
| **NodePort** | Node IP + port | Development, testing |
| **LoadBalancer** | External LB | Production (cloud) |
| **ExternalName** | DNS alias | External service access |

### ConfigMap and Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.example.com"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded
```

Using in Pod:

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DATABASE_PASSWORD
```

---

## 10.6 kubectl Commands

### Basic Operations

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# View resources
kubectl get pods
kubectl get pods -A                  # All namespaces
kubectl get pods -o wide             # More info
kubectl get deployment,svc,pods      # Multiple types

# Describe (detailed info)
kubectl describe pod nginx-pod
kubectl describe node node-1

# Create/apply
kubectl apply -f manifest.yaml
kubectl create deployment nginx --image=nginx

# Delete
kubectl delete pod nginx-pod
kubectl delete -f manifest.yaml

# Edit
kubectl edit deployment nginx-deployment
```

### Debugging

```bash
# Pod logs
kubectl logs pod-name
kubectl logs pod-name -c container   # Specific container
kubectl logs -f pod-name             # Follow
kubectl logs --previous pod-name     # Previous instance

# Execute in container
kubectl exec -it pod-name -- /bin/sh
kubectl exec pod-name -- ls /app

# Port forward
kubectl port-forward pod-name 8080:80
kubectl port-forward svc/service-name 8080:80

# Debug container
kubectl debug pod-name -it --image=busybox

# Events
kubectl get events --sort-by='.lastTimestamp'

# Resource usage
kubectl top pods
kubectl top nodes
```

### Troubleshooting Pods

```bash
# Pod won't start
kubectl describe pod pod-name
# Look at Events section

# Common states:
# - Pending: Waiting for scheduling (resources? node selector?)
# - ContainerCreating: Pulling image, mounting volumes
# - CrashLoopBackOff: Container crashes repeatedly
# - ImagePullBackOff: Can't pull image
# - ErrImagePull: Image doesn't exist
```

---

## 10.7 Common Kubernetes Issues

### Pod Stuck in Pending

```bash
kubectl describe pod pod-name

# Check for:
# - Insufficient resources (CPU, memory)
# - Node selector doesn't match any node
# - Taints and tolerations
# - PVC not bound
```

### Pod in CrashLoopBackOff

```bash
# Check logs
kubectl logs pod-name --previous

# Common causes:
# - Application error
# - Missing config/secrets
# - Liveness probe failing
# - OOMKilled
```

### Service Not Accessible

```bash
# Check endpoints
kubectl get endpoints service-name

# If empty, selector doesn't match pod labels
kubectl get pods --show-labels

# Check service config
kubectl describe svc service-name

# Test from within cluster
kubectl run debug --rm -it --image=busybox -- wget -qO- service-name
```

### DNS Issues

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS from pod
kubectl exec -it pod-name -- nslookup kubernetes
kubectl exec -it pod-name -- nslookup service-name.namespace.svc.cluster.local
```

---

## 10.8 GKE-Specific Features

### Workload Identity

```bash
# Annotate K8s service account
kubectl annotate serviceaccount app-sa \
  iam.gke.io/gcp-service-account=gsa@project.iam.gserviceaccount.com
```

### GKE Autopilot

```bash
# Create autopilot cluster
gcloud container clusters create-auto my-cluster --region=us-central1

# No node management
# Pay per pod resources
# Security best practices by default
```

### Node Auto-Provisioning

```bash
# Enable NAP
gcloud container clusters update my-cluster \
  --enable-autoprovisioning \
  --min-cpu 1 --max-cpu 100 \
  --min-memory 1 --max-memory 400
```

---

## 10.9 Advanced Container Performance Analysis

### Understanding cgroups Deep Dive

Containers use cgroups (control groups) to limit resources. Understanding cgroups is essential for performance analysis.

**cgroups v1 vs v2**:
```
cgroups v1:                          cgroups v2:
/sys/fs/cgroup/                      /sys/fs/cgroup/
├── cpu/                             └── unified hierarchy
├── memory/                              ├── cgroup.controllers
├── blkio/                               ├── cgroup.subtree_control
└── (separate hierarchies)               └── (single hierarchy)
```

### Finding Container cgroups

```bash
# Find container's cgroup (Docker)
docker inspect container_name --format '{{.State.Pid}}'
# Then check /proc/PID/cgroup

# Find cgroup path
cat /proc/<PID>/cgroup

# Example output (v1):
# 12:memory:/docker/abc123...
# 11:cpu,cpuacct:/docker/abc123...

# For cgroups v2:
# 0::/docker/abc123...
```

### CPU Throttling Analysis

One of the most common container performance issues is CPU throttling.

```bash
# Check CPU throttling stats (cgroups v1)
cat /sys/fs/cgroup/cpu/docker/<container_id>/cpu.stat
# nr_periods: Number of enforcement intervals
# nr_throttled: Number of times throttled
# throttled_time: Total time throttled (nanoseconds)

# cgroups v2
cat /sys/fs/cgroup/docker/<container_id>/cpu.stat
# throttled_usec: Total throttle time in microseconds

# Calculate throttle percentage
# throttle_pct = (nr_throttled / nr_periods) * 100
# If > 5%, container is CPU constrained
```

**Interpreting CPU throttling**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU Throttling Timeline                       │
│                                                                  │
│  Period (100ms default)                                         │
│  ├────────────────────────────────────────────────┤             │
│                                                                  │
│  cpu.cfs_quota_us = 50000 (50ms)                               │
│  ├──────────────────────┤                                       │
│  │   Container can run  │  Throttled (can't run)               │
│  │      for 50ms        │     remaining 50ms                    │
│  └──────────────────────┴─────────────────────────┘             │
│                                                                  │
│  If workload needs 80ms of CPU time:                            │
│  → Container runs 50ms, waits 50ms, runs 30ms next period      │
│  → Latency increases, throughput decreases                      │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Pressure Analysis

```bash
# Memory stats (cgroups v1)
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.stat
# Key metrics:
# - cache: Page cache memory
# - rss: Resident set size (actual memory)
# - pgfault: Page faults
# - pgmajfault: Major page faults (disk reads)

# Memory limit and usage
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.usage_in_bytes

# OOM events
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.oom_control
# oom_kill_disable: 0/1
# under_oom: Currently under OOM? 0/1
# oom_kill: Number of OOM kills

# Memory pressure (cgroups v2)
cat /sys/fs/cgroup/docker/<container_id>/memory.pressure
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**Detecting OOM kills**:
```bash
# Check dmesg for OOM kills
dmesg | grep -i "oom\|killed"

# Check container exit code (137 = OOM killed)
docker inspect container_name --format='{{.State.ExitCode}}'
# 137 = 128 + 9 (SIGKILL from OOM)

# Kubernetes OOM detection
kubectl describe pod pod-name | grep -A5 "Last State"
# Reason: OOMKilled
```

### I/O Performance Analysis

```bash
# Block I/O stats (cgroups v1)
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.io_serviced
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.io_service_bytes

# cgroups v2
cat /sys/fs/cgroup/docker/<container_id>/io.stat
# rbytes: Bytes read
# wbytes: Bytes written
# rios: Read I/O operations
# wios: Write I/O operations
```

---

## 10.10 Kernel-Level Container Debugging

### Using nsenter to Access Container Namespaces

`nsenter` allows you to enter a container's namespaces from the host - essential when the container lacks debugging tools.

```bash
# Get container PID
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# Enter all namespaces of the container
nsenter -t $CONTAINER_PID -m -u -i -n -p

# Enter specific namespaces:
nsenter -t $CONTAINER_PID -n                    # Network only
nsenter -t $CONTAINER_PID -m                    # Mount only
nsenter -t $CONTAINER_PID -p -r                 # PID + root

# Run command in container's network namespace
nsenter -t $CONTAINER_PID -n ip addr
nsenter -t $CONTAINER_PID -n netstat -tuln
nsenter -t $CONTAINER_PID -n ss -tuln

# Run tcpdump in container's namespace
nsenter -t $CONTAINER_PID -n tcpdump -i eth0

# Check container's processes from host
nsenter -t $CONTAINER_PID -p -r ps aux
```

### Using /proc for Container Analysis

```bash
# Container's main process info
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# Process command line
cat /proc/$CONTAINER_PID/cmdline | tr '\0' ' '

# Process environment (may contain secrets - be careful)
cat /proc/$CONTAINER_PID/environ | tr '\0' '\n'

# Open file descriptors
ls -la /proc/$CONTAINER_PID/fd/

# Memory maps
cat /proc/$CONTAINER_PID/maps

# Process status
cat /proc/$CONTAINER_PID/status
# Key fields: VmRSS, VmSize, Threads, voluntary_ctxt_switches

# Network connections
cat /proc/$CONTAINER_PID/net/tcp
cat /proc/$CONTAINER_PID/net/tcp6

# CPU time
cat /proc/$CONTAINER_PID/stat
# Field 14: utime (user mode ticks)
# Field 15: stime (kernel mode ticks)
```

### Using strace for System Call Tracing

```bash
# Trace container process from host
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# Basic strace
strace -p $CONTAINER_PID

# Trace with timestamps
strace -tt -p $CONTAINER_PID

# Trace specific syscalls
strace -e open,read,write -p $CONTAINER_PID
strace -e network -p $CONTAINER_PID          # Network syscalls
strace -e file -p $CONTAINER_PID             # File syscalls

# Follow child processes
strace -f -p $CONTAINER_PID

# Count syscalls (summary)
strace -c -p $CONTAINER_PID
# Press Ctrl+C to see summary

# Trace with timing info
strace -T -p $CONTAINER_PID                  # Time spent in syscall
strace -r -p $CONTAINER_PID                  # Relative timestamps

# Save to file
strace -o /tmp/trace.log -p $CONTAINER_PID
```

**Interpreting strace output**:
```
# Example: Slow I/O
open("/data/file.txt", O_RDONLY) = 3          # File opened, fd=3
read(3, "..."..., 4096) = 4096                 # Read 4KB
<... read resumed>) = 4096  <0.523456>        # Took 0.5 seconds!
                                              # ^ This is slow disk I/O

# Example: Connection timeout
connect(3, {sa_family=AF_INET, sin_port=htons(3306),
        sin_addr=inet_addr("10.0.0.5")}, 16) = -1 ETIMEDOUT
                                              # ^ Database connection timeout

# Example: Permission denied
open("/etc/secrets", O_RDONLY) = -1 EACCES (Permission denied)
```

### Using perf for Container Performance Profiling

```bash
# Install perf (if not present)
apt-get install linux-tools-$(uname -r)

# Get container PID
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)

# CPU profiling for container
perf record -p $CONTAINER_PID -g -- sleep 30
perf report

# Record all processes in container's cgroup
perf record -e cycles -g --cgroup=docker/$CONTAINER_ID -- sleep 30

# Real-time top for container
perf top -p $CONTAINER_PID

# Count specific events
perf stat -p $CONTAINER_PID -- sleep 10
# Shows: cycles, instructions, cache-misses, etc.

# Trace context switches
perf record -e context-switches -p $CONTAINER_PID -- sleep 10

# Flame graph generation
perf record -F 99 -p $CONTAINER_PID -g -- sleep 30
perf script > out.perf
# Use flamegraph.pl to generate SVG
```

**Interpreting perf output**:
```
# High % in these functions indicates:
# - malloc/free: Memory allocation pressure
# - __GI___libc_read: I/O bound
# - pthread_mutex_lock: Lock contention
# - copy_user_generic: Kernel copying data to/from userspace
```

---

## 10.11 eBPF and Modern Observability

### What is eBPF?

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Space                               │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │  bpftrace / bcc tools / custom eBPF programs            │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
├─────────────────────────────────────────────────────────────────┤
│                        Kernel Space                              │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │                    eBPF Verifier                         │  │
│    │    (ensures safety: no loops, bounded execution)         │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │              eBPF Programs attached to:                  │  │
│    │  • Tracepoints (syscalls, scheduler, network)           │  │
│    │  • Kprobes (any kernel function)                        │  │
│    │  • Uprobes (any userspace function)                     │  │
│    │  • XDP (network packet processing)                      │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### bpftrace for Container Debugging

```bash
# Install bpftrace
apt-get install bpftrace

# Trace syscalls for a container process
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)
bpftrace -e 'tracepoint:syscalls:sys_enter_* /pid == '$CONTAINER_PID'/ { @[probe] = count(); }'

# Trace file opens in container
bpftrace -e 'tracepoint:syscalls:sys_enter_openat /pid == '$CONTAINER_PID'/ { printf("%s %s\n", comm, str(args->filename)); }'

# Trace TCP connections
bpftrace -e 'kprobe:tcp_connect /pid == '$CONTAINER_PID'/ { printf("connect: %s\n", comm); }'

# Latency histogram for read() syscall
bpftrace -e 'tracepoint:syscalls:sys_enter_read /pid == '$CONTAINER_PID'/ { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read /pid == '$CONTAINER_PID' && @start[tid]/ { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# Memory allocations
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == '$CONTAINER_PID'/ { @bytes = hist(arg0); }'
```

### BCC Tools for Containers

```bash
# Install BCC tools
apt-get install bpfcc-tools

# Trace exec() calls (see what's being launched)
execsnoop-bpfcc -x  # Failed execs only
execsnoop-bpfcc --cgroupmap=/sys/fs/cgroup/docker/<container_id>

# Trace open() calls
opensnoop-bpfcc -p $CONTAINER_PID

# TCP connection tracer
tcpconnect-bpfcc -p $CONTAINER_PID
tcplife-bpfcc -p $CONTAINER_PID          # Connection lifespans

# I/O latency
biolatency-bpfcc                          # Block I/O histogram
ext4slower-bpfcc                          # Slow ext4 operations

# CPU scheduling
runqlat-bpfcc                             # Run queue latency
cpudist-bpfcc -p $CONTAINER_PID          # CPU time distribution

# Memory
memleak-bpfcc -p $CONTAINER_PID          # Detect memory leaks
oomkill-bpfcc                             # Trace OOM kills
```

### Common eBPF Debugging Scenarios

**Scenario 1: High Latency Investigation**
```bash
# 1. Check if it's CPU scheduling
runqlat-bpfcc -p $CONTAINER_PID

# 2. Check if it's syscall latency
syscount-bpfcc -p $CONTAINER_PID -L      # Syscalls with latency

# 3. Check if it's I/O
biolatency-bpfcc -D                       # Per-disk latency

# 4. Check if it's network
tcpconnlat-bpfcc -p $CONTAINER_PID       # TCP connection latency
```

**Scenario 2: Container Using Too Much CPU**
```bash
# Profile CPU usage
profile-bpfcc -F 99 -p $CONTAINER_PID 30

# Check for lock contention
offcputime-bpfcc -p $CONTAINER_PID 10    # Time spent off-CPU

# Check for excessive syscalls
syscount-bpfcc -p $CONTAINER_PID -i 1    # Syscalls per second
```

---

## 10.12 GKE Advanced Debugging

### Ephemeral Debug Containers

Debug running pods without modifying them:

```bash
# Add a debug container to a running pod
kubectl debug -it pod-name --image=busybox --target=container-name

# Debug with more tools
kubectl debug -it pod-name --image=nicolaka/netshoot --target=app

# Debug with privileged access (for kernel tools)
kubectl debug -it pod-name --image=ubuntu --target=app -- bash
apt update && apt install -y strace perf linux-tools-generic

# Create a copy of pod for debugging
kubectl debug pod-name -it --copy-to=debug-pod --container=debug --image=busybox

# Debug node
kubectl debug node/node-name -it --image=ubuntu
# This gives you access to the node's filesystem at /host
chroot /host
```

### Node Problem Detector

GKE uses node-problem-detector to surface node issues:

```bash
# Check node conditions
kubectl describe node node-name | grep -A10 Conditions

# Common conditions:
# - KernelDeadlock: Kernel deadlock detected
# - ReadonlyFilesystem: Filesystem is read-only
# - CorruptDockerOverlay2: Docker overlay2 corruption
# - FrequentKubeletRestart: Kubelet restarting frequently
# - FrequentContainerdRestart: Containerd issues

# View node problem detector logs
kubectl logs -n kube-system -l app=node-problem-detector

# Check events for node issues
kubectl get events --field-selector involvedObject.kind=Node
```

### Analyzing GKE Node Logs

```bash
# SSH to GKE node
gcloud compute ssh node-name --zone=zone

# Key log locations on GKE nodes:
# Kubelet logs
journalctl -u kubelet -f

# Container runtime (containerd)
journalctl -u containerd -f

# System logs
journalctl -k                            # Kernel messages
dmesg | tail -100                        # Recent kernel messages

# Audit logs for security
journalctl -u kubelet | grep -i audit
```

### GKE-Specific Performance Analysis

```bash
# Check for CPU throttling in GKE
kubectl exec -it pod-name -- cat /sys/fs/cgroup/cpu/cpu.stat
# Or for cgroups v2:
kubectl exec -it pod-name -- cat /sys/fs/cgroup/cpu.stat

# Check memory pressure
kubectl exec -it pod-name -- cat /sys/fs/cgroup/memory/memory.stat

# Analyze pod with debug container
kubectl debug -it pod-name --image=busybox:1.35 --target=main -- sh
# Inside debug container:
cat /proc/1/status                       # Main process status
cat /proc/1/io                           # I/O stats

# GKE Autopilot restrictions
# Note: Some debugging tools may not work in Autopilot due to security restrictions
# Use kubectl debug or Cloud Logging instead
```

### Using Cloud Logging for Container Debugging

```bash
# Query container logs in Cloud Logging
gcloud logging read 'resource.type="k8s_container" AND resource.labels.pod_name="my-pod"' --limit=100

# Query for OOM events
gcloud logging read 'resource.type="k8s_node" AND textPayload:"OOM"' --limit=50

# Query for container restarts
gcloud logging read 'resource.type="k8s_container" AND "Back-off restarting"' --limit=50

# Export logs for analysis
gcloud logging read 'resource.type="k8s_container" AND resource.labels.namespace_name="production"' \
  --format=json > logs.json
```

---

## 10.13 Performance Debugging Scenarios

### Scenario 1: Container Slow to Respond

```bash
# Step 1: Check resource usage
kubectl top pod pod-name
docker stats container_name

# Step 2: Check for CPU throttling
kubectl exec -it pod-name -- cat /sys/fs/cgroup/cpu/cpu.stat
# Look for: nr_throttled > 0

# Step 3: Check for memory pressure
kubectl exec -it pod-name -- cat /proc/meminfo
# Look for: MemAvailable vs total

# Step 4: Profile the process
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)
perf record -p $CONTAINER_PID -g -- sleep 30
perf report

# Step 5: Trace syscalls if needed
strace -c -p $CONTAINER_PID
# Look for syscalls taking long time
```

### Scenario 2: Container OOM Killed Repeatedly

```bash
# Step 1: Confirm OOM
kubectl describe pod pod-name | grep -i oom
dmesg | grep -i "oom\|killed"

# Step 2: Check memory usage pattern
kubectl exec -it pod-name -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
kubectl exec -it pod-name -- cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# Step 3: Check what's using memory
kubectl exec -it pod-name -- ps aux --sort=-%mem | head -10

# Step 4: Look for memory leaks
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)
# Use memleak-bpfcc if available
memleak-bpfcc -p $CONTAINER_PID

# Step 5: Decision
# - Increase memory limit if legitimate usage
# - Fix memory leak if found
# - Add memory profiling to application
```

### Scenario 3: Network Performance Issues

```bash
# Step 1: Enter container network namespace
CONTAINER_PID=$(docker inspect -f '{{.State.Pid}}' container_name)
nsenter -t $CONTAINER_PID -n

# Step 2: Check network config
ip addr
ip route
cat /etc/resolv.conf

# Step 3: Check for packet drops
netstat -s | grep -i drop
cat /proc/net/dev  # Look for drop column

# Step 4: Check TCP connections
ss -tunaop | head -20

# Step 5: Capture traffic
tcpdump -i eth0 -w /tmp/capture.pcap -c 1000

# Step 6: Check for DNS issues
time nslookup kubernetes.default.svc.cluster.local
# Should be < 10ms
```

---

## 10.14 Multi-Node Distributed Debugging

### Distributed Debugging Challenges

```
┌─────────────────────────────────────────────────────────────────┐
│            Multi-Node Debugging Challenges                       │
│                                                                  │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │ Node 1 │  │ Node 2 │  │ Node 3 │  │  ...   │  │Node 100│   │
│  │  Pods  │  │  Pods  │  │  Pods  │  │        │  │  Pods  │   │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘   │
│      │           │           │           │           │         │
│      └───────────┴───────────┴───────────┴───────────┘         │
│                              │                                  │
│                    Which node has the issue?                    │
│                    Is it one node or all nodes?                 │
│                    How do we correlate events?                  │
│                    Where did the request fail?                  │
└─────────────────────────────────────────────────────────────────┘
```

### Distributed Tracing with Cloud Trace

```python
# Enable tracing in your application
from opentelemetry import trace
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Set up Cloud Trace exporter
trace.set_tracer_provider(TracerProvider())
cloud_trace_exporter = CloudTraceSpanExporter()
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(cloud_trace_exporter)
)

tracer = trace.get_tracer(__name__)

# Instrument your code
@tracer.start_as_current_span("process_request")
def process_request(request_id):
    with tracer.start_as_current_span("database_query") as span:
        span.set_attribute("request.id", request_id)
        result = query_database()

    with tracer.start_as_current_span("external_api_call"):
        response = call_external_api()

    return result, response
```

```bash
# Query traces via gcloud
gcloud trace traces list --project=PROJECT_ID \
  --filter="span:process_request" \
  --limit=10

# Get specific trace
gcloud trace traces describe TRACE_ID --project=PROJECT_ID
```

### Correlating Logs Across Nodes

```bash
# Add trace context to logs
# In Cloud Logging, use trace and spanId fields

# Query logs by trace ID
gcloud logging read 'trace="projects/PROJECT/traces/TRACE_ID"' \
  --project=PROJECT_ID \
  --format=json

# Query logs across multiple pods for same request
gcloud logging read 'labels.request_id="abc123"' \
  --project=PROJECT_ID \
  --order-by=timestamp

# Find logs from all pods of a deployment
gcloud logging read 'resource.type="k8s_container" AND resource.labels.pod_name:"my-deployment"' \
  --project=PROJECT_ID
```

### Cluster-Wide Debugging Commands

```bash
# Check all pods across all nodes
kubectl get pods -A -o wide --sort-by='.status.startTime'

# Find pods with issues across cluster
kubectl get pods -A | grep -E "Error|CrashLoop|Pending|Unknown"

# Get events cluster-wide, sorted by time
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

# Check resource usage across all nodes
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20
kubectl top pods -A --sort-by=cpu | head -20

# Find pods scheduled on a specific node
kubectl get pods -A -o wide --field-selector spec.nodeName=node-1

# Check pod distribution across nodes
kubectl get pods -A -o wide | awk '{print $8}' | sort | uniq -c | sort -rn

# Describe all nodes to find issues
kubectl describe nodes | grep -A5 "Conditions:"
```

### Multi-Node Log Aggregation

```bash
# Stream logs from all pods of a deployment
kubectl logs -l app=my-app --all-containers -f --prefix

# Get logs from all pods, last hour
for pod in $(kubectl get pods -l app=my-app -o name); do
  echo "=== $pod ==="
  kubectl logs $pod --since=1h
done

# Parallel log collection (faster for many pods)
kubectl get pods -l app=my-app -o name | \
  xargs -P 10 -I {} kubectl logs {} --since=1h > all-logs.txt

# Using stern for multi-pod log streaming (if installed)
stern "my-app-.*" --since 1h
```

### Identifying Cluster-Wide vs Node-Specific Issues

```bash
# Step 1: Is it affecting all nodes or specific nodes?
kubectl get pods -A -o wide | grep -v Running | awk '{print $8}' | sort | uniq -c

# Step 2: Check node health
kubectl get nodes
kubectl describe node NODE_NAME | grep -A 20 "Conditions:"

# Step 3: Check for node-level resource pressure
for node in $(kubectl get nodes -o name); do
  echo "=== $node ==="
  kubectl describe $node | grep -A 3 "Allocated resources:"
done

# Step 4: Check for network partition
# From each node, verify connectivity to API server
kubectl get pods -A -o wide | awk 'NR>1 {print $8}' | sort -u | while read node; do
  echo "Checking $node..."
  kubectl debug node/$node -it --image=busybox -- \
    wget -qO- --timeout=5 https://kubernetes.default.svc/healthz
done
```

### Pattern: Cascading Failure Investigation

```
Symptom: Multiple services failing simultaneously

Investigation Flow:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. Timeline: When did each service start failing?             │
│     kubectl get events -A --sort-by='.lastTimestamp'           │
│                                                                  │
│  2. Dependency Map: What do failing services have in common?   │
│     - Shared database?                                         │
│     - Common network path?                                     │
│     - Same nodes?                                              │
│                                                                  │
│  3. Root Cause Categories:                                     │
│     ┌──────────────┬──────────────┬──────────────┐            │
│     │   Network    │   Resource   │  Dependency  │            │
│     │   Partition  │  Exhaustion  │   Failure    │            │
│     └──────────────┴──────────────┴──────────────┘            │
│                                                                  │
│  4. Verification: Fix one thing at a time, verify improvement  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Debugging Database Connection Issues at Scale

```bash
# Common issue: Connection pool exhaustion across pods

# Check total connections to database
kubectl exec -it db-pod -- psql -c "SELECT count(*) FROM pg_stat_activity;"

# Check connections per client IP (each pod has different IP)
kubectl exec -it db-pod -- psql -c \
  "SELECT client_addr, count(*) FROM pg_stat_activity GROUP BY client_addr ORDER BY count DESC;"

# Find pods with connection errors
kubectl logs -l app=my-app --all-containers | grep -i "connection\|timeout\|refused" | \
  awk '{print $1}' | sort | uniq -c | sort -rn

# Check if specific pods are holding connections
for pod in $(kubectl get pods -l app=my-app -o name); do
  echo "=== $pod ==="
  kubectl exec $pod -- netstat -an | grep 5432 | wc -l
done
```

### Using Ephemeral Debug Pods for Cluster Investigation

```bash
# Deploy a debug pod with networking tools
kubectl run debug-pod --rm -it --image=nicolaka/netshoot -- bash

# Inside debug pod, investigate cluster networking
# Check DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup my-service.my-namespace.svc.cluster.local

# Check connectivity to services
curl -v http://my-service.my-namespace:8080/health

# Check connectivity to external services
curl -v https://external-api.example.com

# Trace network path
mtr my-service.my-namespace.svc.cluster.local

# Check network policies effect
# (this will show if traffic is being blocked)
nc -zv my-service.my-namespace 8080
```

### Large-Scale Debugging Scenarios

**Scenario: 5% of requests failing across cluster**

```bash
# Step 1: Find pattern in failures
# Are failures from specific pods? nodes? paths?

# Check error distribution by pod
kubectl logs -l app=frontend --all-containers | \
  grep "ERROR\|500\|timeout" | \
  awk '{print $1}' | sort | uniq -c | sort -rn | head -20

# Step 2: Check if it's a specific node
# Map error pods to nodes
for pod in pod-with-errors-1 pod-with-errors-2; do
  echo "$pod: $(kubectl get pod $pod -o jsonpath='{.spec.nodeName}')"
done

# Step 3: If node-specific, investigate that node
kubectl describe node problem-node
kubectl get pods -A --field-selector spec.nodeName=problem-node

# Step 4: Check for resource contention
kubectl top pod -l app=frontend --sort-by=cpu
kubectl exec -it problem-pod -- cat /sys/fs/cgroup/cpu/cpu.stat  # throttling?

# Step 5: Check network path from problem node
kubectl debug node/problem-node -it --image=nicolaka/netshoot -- \
  mtr backend-service.default.svc.cluster.local
```

**Scenario: Intermittent latency spikes across services**

```bash
# Step 1: Identify timing pattern
gcloud logging read 'resource.type="k8s_container" AND jsonPayload.latency_ms>1000' \
  --format='value(timestamp,resource.labels.pod_name,jsonPayload.latency_ms)'

# Step 2: Correlate with cluster events
kubectl get events -A --sort-by='.lastTimestamp' | \
  grep -E "$(date +%H:%M|sed 's/:/../')"  # events around same time

# Step 3: Check for noisy neighbors
kubectl top pods -A --sort-by=cpu | head -20

# Step 4: Check for GC pauses or throttling
kubectl logs problem-pod | grep -i "gc\|pause\|throttl"

# Step 5: Use distributed tracing
# Look at slow traces in Cloud Trace
gcloud trace traces list \
  --filter="latency>1s" \
  --project=PROJECT_ID
```

---

## Chapter 10 Review Questions

### Basic Kubernetes
1. What's the difference between a Deployment and a ReplicaSet?

2. A pod is stuck in CrashLoopBackOff. How do you debug?

3. Explain the difference between ClusterIP, NodePort, and LoadBalancer services.

4. How does Kubernetes DNS work?

5. What's Workload Identity and why is it preferred over service account keys?

6. A service has no endpoints. What do you check?

### Advanced Debugging
7. How do you detect if a container is being CPU throttled? What metrics would you look at?

8. Explain how you would use `nsenter` to debug a container that has no debugging tools installed.

9. A container is repeatedly getting OOM killed. Walk through your debugging process.

10. What is eBPF and how does it help with container observability?

11. How would you use `perf` to profile a containerized application?

12. What are cgroups v1 vs v2 and why does it matter for debugging?

### Multi-Node Distributed Debugging
13. How do you correlate logs across 100+ pods to trace a single request?

14. 5% of requests are failing cluster-wide. How do you identify if it's node-specific?

15. Describe how to use distributed tracing to debug latency issues across microservices.

16. How do you debug a cascading failure affecting multiple services?

---

## Chapter 10 Hands-On Exercises

### Exercise 10.1: Basic Container Debugging
1. Run a container and inspect its cgroup
2. Find CPU throttling statistics
3. Check memory usage in cgroup

### Exercise 10.2: Namespace Debugging
1. Get container PID from Docker
2. Use `nsenter` to enter network namespace
3. Run network diagnostics from the host

### Exercise 10.3: Performance Analysis
1. Use `strace` to trace a container's syscalls
2. Identify slow syscalls
3. Profile with `perf` and generate a flame graph

---

## Key Takeaways

### Fundamentals
1. **Containers = namespaces + cgroups** - Isolation and limits
2. **Pods are the unit** - Not containers
3. **Deployments manage pods** - Use for stateless apps
4. **Services expose pods** - Abstract away pod IPs
5. **kubectl describe** - First stop for debugging
6. **Check events** - K8s tells you what's wrong

### Advanced
7. **CPU throttling is common** - Check `nr_throttled` in cgroup stats
8. **nsenter is essential** - Debug containers without tools from host
9. **strace shows syscalls** - Find slow I/O, connection issues, permission errors
10. **perf profiles CPU** - Find hot paths and bottlenecks
11. **eBPF is the future** - Low-overhead kernel observability
12. **Exit code 137 = OOM** - Remember 128 + signal number

### Distributed Debugging
13. **Distributed tracing** - Essential for microservices debugging
14. **Correlate by trace ID** - Link logs across services
15. **Node vs cluster issues** - First determine scope
16. **Check all the layers** - Node, network, pod, container, app

---

[Next Chapter: Troubleshooting Methodology →](./11-troubleshooting.md)
