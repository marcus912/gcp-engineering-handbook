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

## Chapter 10 Review Questions

1. What's the difference between a Deployment and a ReplicaSet?

2. A pod is stuck in CrashLoopBackOff. How do you debug?

3. Explain the difference between ClusterIP, NodePort, and LoadBalancer services.

4. How does Kubernetes DNS work?

5. What's Workload Identity and why is it preferred over service account keys?

6. A service has no endpoints. What do you check?

---

## Key Takeaways

1. **Containers = namespaces + cgroups** - Isolation and limits
2. **Pods are the unit** - Not containers
3. **Deployments manage pods** - Use for stateless apps
4. **Services expose pods** - Abstract away pod IPs
5. **kubectl describe** - First stop for debugging
6. **Check events** - K8s tells you what's wrong

---

[Next Chapter: Troubleshooting Methodology →](./11-troubleshooting.md)
