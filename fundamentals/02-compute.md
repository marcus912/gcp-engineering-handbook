# Chapter 2: Compute Services on GCP

## 2.1 Compute Options Overview

GCP offers multiple compute options, each suited for different use cases:

```
                    More Control
                         ↑
                         │
    Compute Engine (VMs) │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                         │
         GKE (Kubernetes)│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                         │
           Cloud Run     │ ▓▓▓▓▓▓▓▓▓▓
                         │
      Cloud Functions    │ ▓▓▓▓▓▓
                         │
           App Engine    │ ▓▓▓▓
                         │
                         ↓
                    Less Control / More Managed
```

### When to Use Each

| Service | Use When | Avoid When |
|---------|----------|------------|
| **Compute Engine** | Need full OS control, lift-and-shift, specific hardware | Simple stateless apps |
| **GKE** | Containerized microservices, need orchestration | Simple apps, no container expertise |
| **Cloud Run** | Stateless containers, variable traffic, pay-per-request | Stateful apps, need GPUs |
| **Cloud Functions** | Event-driven, short-lived tasks | Long-running processes |
| **App Engine** | Simple web apps, rapid deployment | Need custom runtime, full control |

---

## 2.2 Compute Engine (Virtual Machines)

### Machine Types

**Predefined Machine Types**

| Family | Use Case | vCPU:Memory |
|--------|----------|-------------|
| **E2** | Cost-optimized, general purpose | Shared or dedicated |
| **N2/N2D** | Balanced, most workloads | 1:4 GB |
| **N1** | Legacy balanced | 1:3.75 GB |
| **C2/C2D** | Compute-intensive | 1:4 GB, highest per-core performance |
| **M2/M3** | Memory-optimized | Up to 1:24 GB |
| **A2** | GPU workloads | With NVIDIA A100 GPUs |

**Custom Machine Types**

Create VMs with exact vCPU and memory:

```bash
gcloud compute instances create my-vm \
  --custom-cpu=6 \
  --custom-memory=24GB \
  --zone=us-central1-a
```

### Preemptible and Spot VMs

**Preemptible VMs**:
- Up to 80% cheaper than regular
- Maximum 24-hour lifetime
- Can be preempted with 30-second warning
- Good for: batch processing, CI/CD, fault-tolerant workloads

**Spot VMs** (newer):
- Same pricing as preemptible
- No maximum lifetime
- Same preemption behavior
- Replaces preemptible VMs

```bash
# Create a spot VM
gcloud compute instances create my-spot-vm \
  --provisioning-model=SPOT \
  --instance-termination-action=DELETE \
  --zone=us-central1-a
```

### VM Lifecycle

```
PROVISIONING → STAGING → RUNNING ↔ SUSPENDED
                           ↓
                        STOPPING → TERMINATED
```

**States**:
- **PROVISIONING**: Resources being allocated
- **STAGING**: Resources acquired, preparing to launch
- **RUNNING**: VM is running
- **SUSPENDED**: VM is suspended (memory preserved)
- **STOPPING**: VM is shutting down
- **TERMINATED**: VM is stopped (can be restarted)

### Boot Disks and Images

**Image types**:
- **Public images**: Provided by Google (Debian, Ubuntu, CentOS, Windows)
- **Custom images**: Created by you from disks or other images
- **Image families**: Points to latest image in a group

```bash
# List public images
gcloud compute images list

# Create custom image from disk
gcloud compute images create my-image \
  --source-disk=my-disk \
  --source-disk-zone=us-central1-a
```

### Instance Templates and Groups

**Instance Templates**: Define VM configuration for reuse

```bash
gcloud compute instance-templates create my-template \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud
```

**Managed Instance Groups (MIGs)**: Groups of identical VMs

Features:
- Autoscaling based on metrics
- Auto-healing (recreates unhealthy VMs)
- Rolling updates
- Regional or zonal

```bash
# Create a managed instance group
gcloud compute instance-groups managed create my-mig \
  --template=my-template \
  --size=3 \
  --zone=us-central1-a

# Configure autoscaling
gcloud compute instance-groups managed set-autoscaling my-mig \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --zone=us-central1-a
```

### Sole-Tenant Nodes

Dedicated physical servers for compliance or licensing:

```bash
# Create a node template
gcloud compute sole-tenancy node-templates create my-template \
  --node-type=n1-node-96-624 \
  --region=us-central1

# Create a node group
gcloud compute sole-tenancy node-groups create my-group \
  --node-template=my-template \
  --target-size=1 \
  --zone=us-central1-a
```

### Shielded VMs

Enhanced security features:
- **Secure Boot**: Only run verified bootloader and kernel
- **vTPM**: Virtual Trusted Platform Module
- **Integrity Monitoring**: Verify VM hasn't been tampered with

### Confidential VMs

Memory encryption in use:
- Data encrypted in memory
- Keys not accessible to Google
- For highly sensitive workloads

---

## 2.3 Compute Engine Networking

### Network Interfaces

VMs have one or more network interfaces:

```bash
# Create VM with multiple network interfaces
gcloud compute instances create multi-nic-vm \
  --network-interface=network=vpc-a,subnet=subnet-a \
  --network-interface=network=vpc-b,subnet=subnet-b
```

### Internal vs External IPs

**Internal IP**:
- Always assigned
- Used for communication within VPC
- Private RFC 1918 address

**External IP**:
- Optional
- For internet access
- Ephemeral (changes on restart) or static (permanent)

```bash
# Reserve a static external IP
gcloud compute addresses create my-static-ip \
  --region=us-central1

# Assign to VM
gcloud compute instances add-access-config my-vm \
  --access-config-name="External NAT" \
  --address=my-static-ip
```

### Metadata and Startup Scripts

**Metadata**: Key-value pairs available to the VM

```bash
# Set metadata
gcloud compute instances add-metadata my-vm \
  --metadata=foo=bar

# Access from within VM
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/foo
```

**Startup Scripts**: Run on every boot

```bash
gcloud compute instances create my-vm \
  --metadata-from-file=startup-script=./startup.sh
```

---

## 2.4 Google Kubernetes Engine (GKE)

### GKE Modes

**Standard Mode**:
- You manage node configuration
- More control over infrastructure
- Pay for nodes whether used or not

**Autopilot Mode**:
- Google manages nodes
- Pay per pod resource requests
- Less operational overhead
- Best for most new deployments

### Cluster Architecture

```
┌─────────────────────────────────────────────────┐
│                  GKE Cluster                     │
│  ┌───────────────────────────────────────────┐  │
│  │           Control Plane (Master)          │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────────┐ │  │
│  │  │   API   │ │  etcd   │ │  Scheduler  │ │  │
│  │  │ Server  │ │         │ │ Controller  │ │  │
│  │  └─────────┘ └─────────┘ └─────────────┘ │  │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │   Node 1    │ │   Node 2    │ │   Node 3  │ │
│  │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌───────┐ │ │
│  │ │  Pod A  │ │ │ │  Pod C  │ │ │ │ Pod E │ │ │
│  │ ├─────────┤ │ │ ├─────────┤ │ │ ├───────┤ │ │
│  │ │  Pod B  │ │ │ │  Pod D  │ │ │ │ Pod F │ │ │
│  │ └─────────┘ │ │ └─────────┘ │ │ └───────┘ │ │
│  └─────────────┘ └─────────────┘ └───────────┘ │
└─────────────────────────────────────────────────┘
```

### Creating a GKE Cluster

```bash
# Create a standard cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-medium

# Create an autopilot cluster
gcloud container clusters create-auto my-autopilot-cluster \
  --region=us-central1

# Get credentials for kubectl
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a
```

### Node Pools

Groups of nodes with the same configuration:

```bash
# Add a node pool
gcloud container node-pools create high-memory-pool \
  --cluster=my-cluster \
  --machine-type=n1-highmem-4 \
  --num-nodes=2 \
  --zone=us-central1-a

# Enable autoscaling
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --zone=us-central1-a \
  --node-pool=default-pool
```

### GKE Networking

**VPC-Native Clusters** (recommended):
- Pods get IP addresses from VPC subnet
- Enables direct routing to pods
- Required for many features (Private Google Access, etc.)

**Alias IP Ranges**:
- Primary range: Node IPs
- Pod range: Pod IPs
- Service range: Kubernetes Service IPs

```bash
# Create VPC-native cluster
gcloud container clusters create my-cluster \
  --enable-ip-alias \
  --cluster-ipv4-cidr=/16 \
  --services-ipv4-cidr=/22
```

### Workload Identity

Best practice for GKE authentication to GCP services:

```bash
# Enable on cluster
gcloud container clusters update my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog \
  --zone=us-central1-a

# Create GCP service account
gcloud iam service-accounts create gke-workload-sa

# Grant permissions
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Allow K8s SA to impersonate GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  gke-workload-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/K8S_SA_NAME]"
```

### Private Clusters

Control plane and nodes don't have public IPs:

```bash
gcloud container clusters create private-cluster \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr=172.16.0.0/28 \
  --enable-ip-alias
```

---

## 2.5 Cloud Run

Fully managed serverless container platform.

### Key Concepts

- **Service**: A deployed container that handles requests
- **Revision**: Immutable snapshot of a service
- **Traffic splitting**: Route traffic between revisions

### Deploying to Cloud Run

```bash
# Deploy from container image
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-image \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated

# Deploy from source code (builds automatically)
gcloud run deploy my-service \
  --source=. \
  --region=us-central1
```

### Configuration Options

```bash
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-image \
  --memory=512Mi \
  --cpu=1 \
  --timeout=300 \
  --concurrency=80 \
  --min-instances=1 \
  --max-instances=100 \
  --set-env-vars=KEY=value \
  --service-account=my-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Cloud Run vs Cloud Functions vs App Engine

| Feature | Cloud Run | Cloud Functions | App Engine |
|---------|-----------|-----------------|------------|
| Unit of deployment | Container | Function | Application |
| Language | Any | Node, Python, Go, Java | Limited runtimes |
| Pricing | Per request + CPU/memory | Per invocation | Per instance |
| Cold start | Container pull | Function load | Instance spin-up |
| Concurrency | Multiple requests per instance | 1 request per instance | Multiple |
| Max timeout | 60 minutes | 9 minutes (Gen 1), 60 min (Gen 2) | Varies |

---

## 2.6 Cloud Functions

Event-driven serverless functions.

### Triggers

**HTTP Triggers**:
```bash
gcloud functions deploy http-function \
  --runtime=python39 \
  --trigger-http \
  --entry-point=main \
  --allow-unauthenticated
```

**Event Triggers**:
```bash
# Cloud Storage trigger
gcloud functions deploy storage-function \
  --runtime=python39 \
  --trigger-resource=my-bucket \
  --trigger-event=google.storage.object.finalize \
  --entry-point=process_file

# Pub/Sub trigger
gcloud functions deploy pubsub-function \
  --runtime=python39 \
  --trigger-topic=my-topic \
  --entry-point=process_message
```

### Gen 1 vs Gen 2

| Feature | Gen 1 | Gen 2 |
|---------|-------|-------|
| Infrastructure | Custom | Cloud Run |
| Max timeout | 9 minutes | 60 minutes |
| Instance size | 8GB / 2 vCPU | 16GB / 4 vCPU |
| Concurrency | 1 request/instance | Multiple |
| Min instances | No | Yes |
| Traffic splitting | No | Yes |

**Gen 2 is recommended for new functions.**

---

## 2.7 App Engine

Platform-as-a-Service for web applications.

### Standard vs Flexible

| Feature | Standard | Flexible |
|---------|----------|----------|
| Languages | Python, Java, Node.js, PHP, Ruby, Go | Any (Docker) |
| Scaling | Scale to zero | Min 1 instance |
| Instance startup | Seconds | Minutes |
| SSH access | No | Yes |
| Write to local disk | No | Yes (temporary) |
| Pricing | Per instance-hour + free tier | Per resource (vCPU, memory) |

### Deploying

```yaml
# app.yaml
runtime: python39
entrypoint: gunicorn -b :$PORT main:app

instance_class: F2
automatic_scaling:
  min_instances: 1
  max_instances: 10
  target_cpu_utilization: 0.6
```

```bash
gcloud app deploy app.yaml
```

---

## 2.8 Compute Troubleshooting

### Common Issues and Solutions

**VM won't start**:
1. Check quotas (vCPU, persistent disk)
2. Check zone capacity
3. Check boot disk (corrupted, wrong image)
4. Check IAM permissions

```bash
# Check project quotas
gcloud compute project-info describe --project=PROJECT_ID

# Check serial port output for boot issues
gcloud compute instances get-serial-port-output my-vm
```

**Can't SSH to VM**:
1. Firewall rule exists for SSH (port 22)
2. VM has external IP or you're using IAP
3. OS Login is configured correctly
4. Check VM is running

```bash
# Check firewall rules
gcloud compute firewall-rules list

# SSH troubleshooting
gcloud compute ssh my-vm --troubleshoot

# Use IAP for VMs without external IP
gcloud compute ssh my-vm --tunnel-through-iap
```

**VM is slow**:
1. Check CPU/memory usage
2. Check disk I/O
3. Check network throughput
4. Check for noisy neighbors (use sole-tenant nodes)

```bash
# View metrics in Cloud Monitoring
# Or use Cloud Console → Compute Engine → VM → Monitoring tab
```

**GKE pod not starting**:
1. Check pod events: `kubectl describe pod POD_NAME`
2. Check container logs: `kubectl logs POD_NAME`
3. Check node resources: `kubectl describe node NODE_NAME`
4. Check image pull: verify image exists and SA has access

**GKE connectivity issues**:
1. Check Services and Endpoints
2. Check Network Policies
3. Check firewall rules
4. Check DNS (CoreDNS)

```bash
# Debug DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes

# Check endpoints
kubectl get endpoints SERVICE_NAME
```

---

## Chapter 2 Review Questions

1. When would you choose Cloud Run over GKE?

2. What's the difference between preemptible and spot VMs?

3. A customer's managed instance group isn't scaling up. What would you check?

4. Explain Workload Identity and why it's preferred over service account keys.

5. What's the difference between GKE Standard and Autopilot modes?

6. A Cloud Function is timing out. What are possible causes and solutions?

---

## Chapter 2 Hands-On Exercises

### Exercise 2.1: Create and Connect to a VM
1. Create an e2-micro VM with Debian
2. SSH into the VM
3. View the metadata from inside the VM
4. Create a startup script that installs nginx

### Exercise 2.2: Deploy to Cloud Run
1. Create a simple Flask or Express app
2. Build a Docker image
3. Push to Container Registry or Artifact Registry
4. Deploy to Cloud Run
5. Test the endpoint

### Exercise 2.3: Create a GKE Cluster
1. Create an Autopilot cluster
2. Deploy a sample application
3. Expose it via a LoadBalancer service
4. Set up Workload Identity for a pod

---

## Key Takeaways

1. **Choose the right compute option** - Match service to workload requirements
2. **GKE Autopilot is preferred** for most containerized workloads
3. **Cloud Run is ideal** for stateless, request-driven containers
4. **Use managed instance groups** for scalable VM workloads
5. **Workload Identity > service account keys** for GKE auth
6. **Know the troubleshooting flow** for each compute type

---

[Next Chapter: Storage & Databases →](./03-storage-databases.md)
