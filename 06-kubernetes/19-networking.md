# Chapter 19: Kubernetes Networking Deep Dive

**Prerequisites:** This chapter builds on Chapter 8 (TCP/IP Fundamentals), Chapter 9 (DNS & HTTP), and Chapter 12 (Containers & Kubernetes Basics). Familiarity with Linux networking concepts (iptables, routing, network namespaces) is strongly recommended.

Kubernetes networking is often cited as one of the most complex aspects of the platform. Understanding how pods communicate, how services provide stable endpoints, and how traffic flows through the cluster is essential for troubleshooting production issues and designing reliable applications.

---

## 18.1 The Kubernetes Networking Model

### Four Fundamental Requirements

Kubernetes imposes four fundamental requirements on any network implementation:

1. **Pods can communicate with all other pods** without NAT (on any node)
2. **Agents on a node** (kubelet, system daemons) can communicate with all pods on that node
3. **Pods in the host network** can communicate with all pods on all nodes without NAT
4. **The IP address a pod sees for itself** is the same IP that others see for it (no NAT between pods)

This "flat network" model simplifies application development—pods can communicate as if they're all on the same network segment, regardless of which node they're running on.

### IP Address Allocation

A Kubernetes cluster has three distinct IP address ranges:

```
┌─────────────────────────────────────────────────────────┐
│                     Cluster Network                     │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ Node CIDR    │  │ Pod CIDR     │  │ Service CIDR │ │
│  │ 10.0.0.0/16  │  │ 10.244.0.0/16│  │ 10.96.0.0/12 │ │
│  │              │  │              │  │              │ │
│  │ Node IPs for │  │ Pod IPs      │  │ Virtual IPs  │ │
│  │ physical     │  │ allocated to │  │ for Services │ │
│  │ machines     │  │ containers   │  │ (ClusterIP)  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────┘
```

- **Node CIDR**: IP addresses for cluster nodes (VMs or physical machines)
- **Pod CIDR**: Pool of IPs assigned to pods (each node gets a subnet)
- **Service CIDR**: Virtual IPs for Services (never actually assigned to interfaces)

### Pod Network Namespace and veth Pairs

Each pod has its own network namespace with a virtual ethernet pair (veth):

```
┌─────────────────────────────────────────────────────────────────┐
│ Node (root network namespace)                                   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Pod Network Namespace                                    │  │
│  │                                                          │  │
│  │  ┌────────┐                                             │  │
│  │  │ eth0   │ 10.244.1.5                                  │  │
│  │  │        │                                             │  │
│  │  └────┬───┘                                             │  │
│  │       │ (veth pair)                                     │  │
│  └───────┼──────────────────────────────────────────────────┘  │
│          │                                                      │
│  ┌───────┴───┐                                                 │
│  │ vethXXXXX │                                                 │
│  └───────┬───┘                                                 │
│          │                                                      │
│  ┌───────┴───────┐                                             │
│  │  cni0 bridge  │ 10.244.1.1                                  │
│  │  or routing   │                                             │
│  └───────┬───────┘                                             │
│          │                                                      │
│  ┌───────┴───┐                                                 │
│  │   eth0    │ 10.0.1.10 (node IP)                             │
│  └───────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
```

When a pod is created:
1. CNI plugin creates a new network namespace
2. Creates a veth pair (virtual ethernet cable with two ends)
3. One end (eth0) goes inside the pod namespace
4. Other end (vethXXX) stays in the root namespace
5. Pod end gets an IP from the pod CIDR
6. Traffic flows through the veth pair to the node's network

### Pod-to-Pod Communication: Same Node

```
┌──────────────────────────────────────────────────────┐
│ Node A (10.0.1.10)                                   │
│                                                      │
│  ┌────────────┐           ┌────────────┐            │
│  │ Pod A      │           │ Pod B      │            │
│  │ 10.244.1.5 │           │ 10.244.1.6 │            │
│  └─────┬──────┘           └──────┬─────┘            │
│        │ veth1                   │ veth2            │
│        │                         │                  │
│  ┌─────┴─────────────────────────┴─────┐            │
│  │         cni0 bridge                 │            │
│  │         10.244.1.1                  │            │
│  └─────────────────────────────────────┘            │
│                                                      │
│  Packet flow: Pod A → veth1 → cni0 → veth2 → Pod B  │
└──────────────────────────────────────────────────────┘
```

For pods on the same node, traffic goes through the local bridge without leaving the node.

### Pod-to-Pod Communication: Different Nodes

```
┌────────────────────────┐         ┌────────────────────────┐
│ Node A (10.0.1.10)     │         │ Node B (10.0.1.11)     │
│                        │         │                        │
│  ┌──────────────┐      │         │      ┌──────────────┐  │
│  │ Pod A        │      │         │      │ Pod B        │  │
│  │ 10.244.1.5   │      │         │      │ 10.244.2.8   │  │
│  └──────┬───────┘      │         │      └──────┬───────┘  │
│         │              │         │             │          │
│  ┌──────┴───────┐      │         │      ┌──────┴───────┐  │
│  │ cni0 bridge  │      │         │      │ cni0 bridge  │  │
│  │ 10.244.1.1   │      │         │      │ 10.244.2.1   │  │
│  └──────┬───────┘      │         │      └──────┬───────┘  │
│         │              │         │             │          │
│  ┌──────┴───────┐      │         │      ┌──────┴───────┐  │
│  │ eth0         │      │         │      │ eth0         │  │
│  │ 10.0.1.10    │──────┼────────►│      │ 10.0.1.11    │  │
│  └──────────────┘      │  Route  │      └──────────────┘  │
│                        │  or     │                        │
│  Route:                │  Tunnel │  Route:                │
│  10.244.2.0/24 →       │  (VXLAN/│  10.244.1.0/24 →       │
│    via 10.0.1.11       │  IPIP)  │    via 10.0.1.10       │
└────────────────────────┘         └────────────────────────┘
```

Steps for cross-node communication:
1. Pod A sends packet to 10.244.2.8
2. Packet goes to cni0 bridge
3. Node routing table directs to Node B (10.0.1.11)
4. Packet travels over node network (may be encapsulated in overlay)
5. Node B receives packet, routes to cni0 bridge
6. Bridge forwards to Pod B's veth

### Pod-to-Service Communication

Services provide stable virtual IPs that map to pod endpoints:

```
┌────────────────────────────────────────────────────────┐
│ Service: my-app (ClusterIP: 10.96.100.50:80)          │
│                                                        │
│  Client Pod sends to: 10.96.100.50:80                 │
│         │                                              │
│         ▼                                              │
│  ┌──────────────┐                                      │
│  │  kube-proxy  │ (iptables rules or IPVS)            │
│  │              │                                      │
│  │  DNAT rules: │                                      │
│  │  10.96.100.50:80 →                                 │
│  │    25% → 10.244.1.5:8080 (Pod A)                   │
│  │    25% → 10.244.1.6:8080 (Pod B)                   │
│  │    25% → 10.244.2.8:8080 (Pod C)                   │
│  │    25% → 10.244.2.9:8080 (Pod D)                   │
│  └──────┬───────┘                                      │
│         │                                              │
│         ▼                                              │
│  Selected pod receives traffic with:                   │
│    Source: 10.244.x.y (client pod IP)                 │
│    Dest: 10.244.x.z:8080 (selected pod IP)            │
└────────────────────────────────────────────────────────┘
```

Key insight: Service ClusterIPs are **never assigned to any interface**. They exist only in iptables/IPVS rules. kube-proxy watches the API server and updates rules when endpoints change.

### Checking IP Allocations

```bash
# View pod CIDR allocation per node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Output:
# node-1    10.244.0.0/24
# node-2    10.244.1.0/24
# node-3    10.244.2.0/24

# View service CIDR (from API server)
kubectl cluster-info dump | grep service-cluster-ip-range
# --service-cluster-ip-range=10.96.0.0/12

# View all pod IPs
kubectl get pods -A -o wide

# Inspect pod network namespace
NODE="node-1"
POD_NAME="my-app-xyz"
kubectl debug node/${NODE} -it --image=nicolaka/netshoot -- bash
# Inside debug pod:
CONTAINER_PID=$(crictl inspect --output go-template --template '{{.info.pid}}' ${CONTAINER_ID})
nsenter -t ${CONTAINER_PID} -n ip addr show
```

---

## 18.2 CNI Plugins

The **Container Network Interface (CNI)** is a specification for configuring network interfaces in Linux containers. Kubernetes uses CNI plugins to set up pod networking.

### Common CNI Plugins

#### Calico

**Architecture:**
- Uses BGP (Border Gateway Protocol) to distribute routes
- Can operate in pure L3 mode (no overlay) or with IPIP/VXLAN encapsulation
- Supports Network Policies

```bash
# Install Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Check Calico status
kubectl get pods -n kube-system -l k8s-app=calico-node

# View BGP peers
kubectl exec -n kube-system calico-node-xxxxx -- calicoctl node status

# Check IP pool configuration
kubectl get ippool -o yaml
```

**Calico IPIP Mode:**
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  ipipMode: CrossSubnet  # IPIP only when crossing subnets
  natOutgoing: true
  nodeSelector: all()
```

**Calico VXLAN Mode:**
```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  vxlanMode: CrossSubnet
  natOutgoing: true
```

**When to use:**
- Need Network Policy support
- Want native routing without overlays (BGP mode)
- Enterprise features (encryption, compliance reporting)

#### Cilium

**Architecture:**
- Uses eBPF (extended Berkeley Packet Filter) for packet processing
- Bypasses iptables for better performance
- Provides Layer 7 visibility and policies

```bash
# Install Cilium
curl -LO https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
cilium install

# Check status
cilium status

# View connectivity map
cilium connectivity test

# Monitor traffic
cilium monitor --type drop
cilium monitor --type trace --to-pod default/my-app-xyz
```

**Cilium Network Policy (Layer 7):**
```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: http-only
spec:
  endpointSelector:
    matchLabels:
      app: web
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/.*"
```

**When to use:**
- Need high performance (eBPF is faster than iptables)
- Want Layer 7 visibility and policies
- Modern kernel (4.19+) available

#### Flannel

**Architecture:**
- Simple overlay network using VXLAN (default) or other backends
- No built-in Network Policy support
- Easy to set up

```bash
# Install Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Check Flannel configuration
kubectl get cm -n kube-flannel kube-flannel-cfg -o yaml
```

**Flannel Configuration:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI": 1,
        "Port": 8472
      }
    }
```

**When to use:**
- Simple cluster setup
- Don't need Network Policies
- Just need basic pod networking

### CNI Comparison Table

| Feature | Calico | Cilium | Flannel |
|---------|--------|---------|---------|
| **Routing Mode** | BGP, IPIP, VXLAN | Native, Tunnel | VXLAN, Host-GW |
| **Network Policies** | Yes (L3/L4) | Yes (L3-L7) | No |
| **Performance** | Good | Excellent (eBPF) | Good |
| **Complexity** | Medium | High | Low |
| **Layer 7 Support** | No | Yes | No |
| **Encryption** | WireGuard/IPsec | WireGuard/IPsec | No |
| **Multi-cluster** | Yes | Yes | Limited |
| **Windows Support** | Yes | Limited | Yes |

### Overlay vs Native Routing

**Overlay Networks (VXLAN/IPIP):**
- Encapsulate pod packets inside node network packets
- Work in any environment (cloud, on-prem)
- Slight performance overhead (~5-10%)
- Hide pod IPs from underlying network

```
Original Packet:
┌─────────────────────────────────────┐
│ Src: 10.244.1.5 (Pod A)             │
│ Dst: 10.244.2.8 (Pod B)             │
│ Payload: HTTP request               │
└─────────────────────────────────────┘

VXLAN Encapsulated:
┌─────────────────────────────────────┐
│ Outer Header:                       │
│   Src: 10.0.1.10 (Node A)           │
│   Dst: 10.0.1.11 (Node B)           │
│   UDP Port 8472 (VXLAN)             │
├─────────────────────────────────────┤
│ VXLAN Header (VNI: 1)               │
├─────────────────────────────────────┤
│ Inner Header:                       │
│   Src: 10.244.1.5 (Pod A)           │
│   Dst: 10.244.2.8 (Pod B)           │
├─────────────────────────────────────┤
│ Payload: HTTP request               │
└─────────────────────────────────────┘
```

**Native Routing (BGP):**
- Pod IPs routed directly by node network
- Better performance (no encapsulation)
- Requires underlying network to support pod CIDR routes
- Cloud providers may not allow this (AWS VPC routing limits)

```bash
# Node A routing table with BGP
ip route show
# 10.244.0.0/24 via 10.0.1.9 dev eth0 proto bird
# 10.244.1.0/24 dev cni0 proto kernel scope link
# 10.244.2.0/24 via 10.0.1.11 dev eth0 proto bird
```

### CNI Troubleshooting

**Problem: Pods can't communicate**

```bash
# 1. Check CNI pods are running
kubectl get pods -n kube-system -l k8s-app=calico-node  # or cilium, flannel

# 2. Check CNI logs
kubectl logs -n kube-system calico-node-xxxxx -c calico-node

# 3. Verify pod has IP address
kubectl get pod my-app-xyz -o jsonpath='{.status.podIP}'

# 4. Check routing from pod perspective
kubectl exec my-app-xyz -- ip route

# 5. Test connectivity between pods
kubectl run test-pod --image=nicolaka/netshoot -it --rm -- bash
# Inside pod:
ping 10.244.2.8
curl http://10.244.2.8:8080

# 6. Check iptables rules (on node)
iptables -t nat -L KUBE-SERVICES -n -v
iptables -t filter -L FORWARD -n -v

# 7. Verify CNI configuration
cat /etc/cni/net.d/10-calico.conflist
```

**Problem: NetworkPolicy not working**

```bash
# Check if CNI supports NetworkPolicies
kubectl get crd networkpolicies.networking.k8s.io

# For Calico:
kubectl exec -n kube-system calico-node-xxxxx -- calico-node -felix-live

# Test with temporary allow-all policy
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - {}
EOF
```

**Problem: CNI binary missing**

```bash
# Check CNI binaries exist on node
ls -l /opt/cni/bin/
# Should contain: bridge, calico, cilium, flannel, host-local, loopback, etc.

# Check CNI config
ls -l /etc/cni/net.d/

# Reinstall CNI if necessary
kubectl delete -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## 18.3 kube-proxy & Service Implementation

**kube-proxy** is responsible for implementing Kubernetes Services. It runs on every node and programs iptables (or IPVS) rules to redirect Service traffic to pod endpoints.

### Service Types

#### ClusterIP (Default)

Exposes Service on a cluster-internal IP. Only reachable from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80        # Service port
      targetPort: 8080  # Container port
```

```bash
kubectl get svc my-app
# NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# my-app   ClusterIP   10.96.100.50    <none>        80/TCP    5m

# Test from within cluster
kubectl run test --image=curlimages/curl -it --rm -- curl http://my-app.default.svc.cluster.local
```

#### NodePort

Exposes Service on each node's IP at a static port (30000-32767 range).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080  # Optional: specify port, or auto-assigned
```

```bash
kubectl get svc my-app-nodeport
# NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# my-app-nodeport   NodePort   10.96.100.51    <none>        80:30080/TCP   5m

# Access from outside cluster
curl http://<any-node-ip>:30080
```

**Traffic flow:**
```
External Client
   │
   ▼
Node IP:30080 (NodePort)
   │
   ▼
iptables DNAT
   │
   ▼
ClusterIP:80
   │
   ▼
Pod IP:8080
```

#### LoadBalancer

Exposes Service externally using a cloud provider's load balancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```bash
kubectl get svc my-app-lb
# NAME        TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
# my-app-lb   LoadBalancer   10.96.100.52    35.123.45.67      80:31234/TCP   5m

# Cloud provider assigns external IP (GCP LB, AWS ELB, etc.)
curl http://35.123.45.67
```

#### ExternalName

Maps Service to a DNS name (CNAME record). No proxying.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

```bash
# Pods can reference external-db.default.svc.cluster.local
# DNS returns CNAME to db.example.com
kubectl run test --image=curlimages/curl -it --rm -- nslookup external-db
```

### iptables Mode Deep Dive

When a Service is created, kube-proxy creates iptables rules in the NAT table. Here's the chain walk-through:

```bash
# View Service chains
iptables -t nat -L KUBE-SERVICES -n

# Example output:
# Chain KUBE-SERVICES (2 references)
# target         prot  source     destination
# KUBE-SVC-XYZ   tcp   0.0.0.0/0  10.96.100.50  tcp dpt:80
```

**Complete iptables chain for a Service with 3 endpoints:**

```
PREROUTING / OUTPUT
    │
    ▼
KUBE-SERVICES
    │
    ├── Match 10.96.100.50:80? → KUBE-SVC-ABCDEF123456
    │                                  │
    │                                  ├── statistic mode random prob 0.33 → KUBE-SEP-111111
    │                                  │        └── DNAT to 10.244.1.5:8080
    │                                  │
    │                                  ├── statistic mode random prob 0.50 → KUBE-SEP-222222
    │                                  │        └── DNAT to 10.244.1.6:8080
    │                                  │
    │                                  └── (default) → KUBE-SEP-333333
    │                                           └── DNAT to 10.244.2.8:8080
    │
    └── ...other services...
```

**Actual iptables rules:**

```bash
# Service chain (entry point)
iptables -t nat -L KUBE-SVC-ABCDEF123456 -n -v

# Chain KUBE-SVC-ABCDEF123456 (1 references)
# -A KUBE-SVC-ABCDEF123456 -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-111111
# -A KUBE-SVC-ABCDEF123456 -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-222222
# -A KUBE-SVC-ABCDEF123456 -j KUBE-SEP-333333

# Endpoint chains (DNAT to pod IPs)
iptables -t nat -L KUBE-SEP-111111 -n -v

# Chain KUBE-SEP-111111 (1 references)
# -A KUBE-SEP-111111 -p tcp -m tcp -j DNAT --to-destination 10.244.1.5:8080
```

**Probability calculation** for N endpoints:
- 1st endpoint: 1/N = 0.33333 (for 3 endpoints)
- 2nd endpoint: 1/(N-1) = 0.50000 (from remaining 2)
- Last endpoint: 1.00000 (default, no rule needed)

### IPVS Mode

IPVS (IP Virtual Server) is a more efficient alternative to iptables for large clusters.

```bash
# Enable IPVS mode in kube-proxy
kubectl edit cm -n kube-system kube-proxy
# Set: mode: "ipvs"

# Restart kube-proxy pods
kubectl delete pod -n kube-system -l k8s-app=kube-proxy

# Verify IPVS mode
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using ipvs Proxier"

# View IPVS rules
ipvsadm -Ln

# Example output:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
# TCP  10.96.100.50:80 rr
#   -> 10.244.1.5:8080              Masq    1      0          0
#   -> 10.244.1.6:8080              Masq    1      0          0
#   -> 10.244.2.8:8080              Masq    1      0          0
```

**IPVS vs iptables:**

| Aspect | iptables | IPVS |
|--------|----------|------|
| **Algorithm** | Sequential rule matching | Hash table lookup |
| **Performance** | O(n) - degrades with rules | O(1) - constant time |
| **Scalability** | Poor (>1000 services) | Excellent (10,000+ services) |
| **Load Balancing** | Random | rr, lc, dh, sh, sed, nq |
| **Overhead** | High CPU in large clusters | Low CPU |

### Headless Services

A Service with `clusterIP: None` doesn't get a virtual IP. DNS returns pod IPs directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-headless
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

```bash
# DNS returns all pod IPs
kubectl run test --image=curlimages/curl -it --rm -- nslookup my-app-headless

# Output:
# Name:   my-app-headless.default.svc.cluster.local
# Address: 10.244.1.5
# Address: 10.244.1.6
# Address: 10.244.2.8
```

**Use cases:**
- StatefulSets (stable DNS per pod: `pod-0.my-app-headless.default.svc.cluster.local`)
- Custom load balancing logic
- Service discovery without kube-proxy

### Session Affinity

By default, Services use random load balancing. Session affinity routes requests from the same client to the same pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-sticky
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

**How it works:**
- kube-proxy tracks client IP in conntrack table
- Same client IP always routes to same pod (for timeout duration)
- Based on source IP, so won't work across NAT boundaries

### iptables Debugging

**Check if Service has endpoints:**

```bash
kubectl get endpoints my-app
# NAME     ENDPOINTS                                   AGE
# my-app   10.244.1.5:8080,10.244.1.6:8080,10.244.2.8:8080   5m

# No endpoints? Check pod selector
kubectl get pods -l app=my-app -o wide
```

**View iptables rules for specific Service:**

```bash
# Get Service CIDR
SERVICE_IP=$(kubectl get svc my-app -o jsonpath='{.spec.clusterIP}')
SERVICE_PORT=$(kubectl get svc my-app -o jsonpath='{.spec.ports[0].port}')

# Find chain name
iptables -t nat -L KUBE-SERVICES -n | grep ${SERVICE_IP}
# -A KUBE-SERVICES -d 10.96.100.50/32 -p tcp -m tcp --dport 80 -j KUBE-SVC-ABCDEF123456

# View Service chain
iptables -t nat -L KUBE-SVC-ABCDEF123456 -n -v

# View endpoint chain
iptables -t nat -L KUBE-SEP-111111 -n -v
```

**Trace packet flow:**

```bash
# Enable iptables trace
iptables -t raw -A PREROUTING -d 10.96.100.50 -j TRACE
iptables -t raw -A OUTPUT -d 10.96.100.50 -j TRACE

# Test connection
curl http://10.96.100.50

# View trace (check kernel logs)
journalctl -k -f | grep TRACE

# Disable trace
iptables -t raw -D PREROUTING -d 10.96.100.50 -j TRACE
iptables -t raw -D OUTPUT -d 10.96.100.50 -j TRACE
```

**Check conntrack entries:**

```bash
# View active connections tracked by netfilter
conntrack -L | grep 10.96.100.50

# Example:
# tcp  6 86399 ESTABLISHED src=10.244.1.5 dst=10.96.100.50 sport=54321 dport=80 \
#                          src=10.244.2.8 dst=10.244.1.5 sport=8080 dport=54321 [ASSURED]
```

---

## 18.4 DNS in Kubernetes

Kubernetes uses **CoreDNS** (as of v1.13+) to provide DNS resolution for Services and Pods.

### CoreDNS Architecture

```
┌───────────────────────────────────────────────────────────┐
│ Pod makes DNS query: my-app.default.svc.cluster.local    │
└────────────────────────────┬──────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ /etc/resolv.conf in Pod                                  │
│                                                          │
│   nameserver 10.96.0.10  # kube-dns Service ClusterIP   │
│   search default.svc.cluster.local svc.cluster.local    │
│   search cluster.local                                   │
│   options ndots:5                                        │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│ CoreDNS Pod (kube-dns Service)                           │
│                                                          │
│   ┌──────────────────────────────────────────┐          │
│   │ Corefile configuration                   │          │
│   │   cluster.local zone                     │          │
│   │   Watches Kubernetes API for Services    │          │
│   │   and Endpoints                          │          │
│   └──────────────────────────────────────────┘          │
│                                                          │
│   Returns: 10.96.100.50 (Service ClusterIP)             │
└──────────────────────────────────────────────────────────┘
```

### CoreDNS Configuration (Corefile)

```bash
kubectl get cm -n kube-system coredns -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

**Key plugins:**
- `kubernetes`: Answers queries for Kubernetes DNS (Services, Pods)
- `forward`: Forwards non-cluster queries to upstream DNS
- `cache`: Caches DNS responses
- `errors`: Logs DNS errors
- `health` / `ready`: Health check endpoints

### DNS Record Types

Kubernetes creates several types of DNS records:

#### Service DNS Records

```bash
# Standard Service
# Format: <service>.<namespace>.svc.cluster.local
my-app.default.svc.cluster.local → 10.96.100.50 (ClusterIP)

# Headless Service (clusterIP: None)
# Returns all pod IPs
my-app-headless.default.svc.cluster.local → 10.244.1.5, 10.244.1.6, 10.244.2.8

# SRV records for named ports
_http._tcp.my-app.default.svc.cluster.local
# Returns: 0 33 80 my-app.default.svc.cluster.local
```

#### Pod DNS Records

```bash
# Format: <pod-ip-with-dashes>.<namespace>.pod.cluster.local
# Example: Pod with IP 10.244.1.5
10-244-1-5.default.pod.cluster.local → 10.244.1.5

# StatefulSet pods get predictable DNS
# Format: <pod-name>.<service-name>.<namespace>.svc.cluster.local
my-app-0.my-app-headless.default.svc.cluster.local → 10.244.1.5
my-app-1.my-app-headless.default.svc.cluster.local → 10.244.1.6
```

### DNS Search Domains and ndots

The `/etc/resolv.conf` in pods contains search domains:

```bash
kubectl exec my-pod -- cat /etc/resolv.conf

# Output:
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

**ndots:5** means if a query has fewer than 5 dots, try all search domains first:

```
Query: my-app
       └── Tries:
           1. my-app.default.svc.cluster.local (5 dots, matches!)
           2. (stops, found match)

Query: my-app.default
       └── Tries:
           1. my-app.default.default.svc.cluster.local (fails)
           2. my-app.default.svc.cluster.local (4 dots, matches!)

Query: google.com (1 dot, < 5)
       └── Tries:
           1. google.com.default.svc.cluster.local (fails)
           2. google.com.svc.cluster.local (fails)
           3. google.com.cluster.local (fails)
           4. google.com (finally succeeds)
```

**Performance impact:** External DNS queries make 4 attempts before succeeding!

**Optimization:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-dns
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"  # Reduces unnecessary search attempts
  containers:
    - name: app
      image: myapp:latest
```

Or use FQDN with trailing dot to skip search:

```bash
# Instead of:
curl http://google.com  # 4 DNS queries

# Use:
curl http://google.com.  # 1 DNS query (trailing dot = absolute)
```

### DNS Resolution Flow

```
┌──────────────────────────────────────────────────────────┐
│ 1. Application calls getaddrinfo("my-app")               │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ 2. glibc checks /etc/resolv.conf                         │
│    - ndots:5 → query has 0 dots, use search domains      │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ 3. Query: my-app.default.svc.cluster.local               │
│    Sent to: 10.96.0.10:53 (kube-dns Service)             │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ 4. kube-proxy intercepts (ClusterIP)                     │
│    DNAT to CoreDNS pod: 10.244.0.5:53                    │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ 5. CoreDNS processes query                               │
│    - Checks cache (30s TTL)                              │
│    - Queries Kubernetes API for Service                  │
│    - Returns ClusterIP: 10.96.100.50                     │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ 6. Application receives IP, connects to Service          │
│    Connection goes through kube-proxy to pod             │
└──────────────────────────────────────────────────────────┘
```

### NodeLocal DNSCache

For high-traffic clusters, NodeLocal DNSCache runs a DNS cache on every node to reduce latency and CoreDNS load.

```yaml
# Deploy NodeLocal DNSCache
kubectl apply -f https://k8s.io/examples/admin/dns/nodelocaldns.yaml

# Architecture:
# ┌─────────────────────────────────────────┐
# │ Node                                    │
# │                                         │
# │  ┌───────────────┐                      │
# │  │ Pod           │                      │
# │  │ nameserver:   │                      │
# │  │ 169.254.20.10 │                      │
# │  └───────┬───────┘                      │
# │          │                              │
# │  ┌───────┴─────────────┐                │
# │  │ NodeLocal DNSCache  │ (DaemonSet)    │
# │  │ 169.254.20.10       │                │
# │  └───────┬─────────────┘                │
# │          │ (cache miss)                 │
# │          ▼                              │
# │  ┌─────────────────────┐                │
# │  │ CoreDNS Service     │                │
# │  │ 10.96.0.10          │                │
# │  └─────────────────────┘                │
# └─────────────────────────────────────────┘
```

**Benefits:**
- Reduces DNS query latency (local cache vs remote pod)
- Reduces CoreDNS load
- Improves reliability (node-local, no network dependency)

### Common DNS Issues

#### Problem: DNS queries timing out

```bash
# Test DNS from pod
kubectl exec -it my-pod -- nslookup kubernetes.default

# If timeout, check:
# 1. CoreDNS pods running?
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. kube-dns Service exists?
kubectl get svc -n kube-system kube-dns

# 3. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 4. Test CoreDNS directly
kubectl run -it --rm debug --image=nicolaka/netshoot -- bash
dig @10.96.0.10 kubernetes.default.svc.cluster.local
```

#### Problem: Intermittent DNS failures

Often caused by conntrack table exhaustion or kernel race condition:

```bash
# Check conntrack table size (on node)
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# If count approaches max, increase:
sysctl -w net.netfilter.nf_conntrack_max=1048576

# Persistent:
echo "net.netfilter.nf_conntrack_max = 1048576" >> /etc/sysctl.conf
sysctl -p

# Alternative: Use TCP for DNS instead of UDP
kubectl set env deployment/coredns -n kube-system GODEBUG=netdns=go
```

#### Problem: External DNS not resolving

```bash
# Check CoreDNS forward configuration
kubectl get cm -n kube-system coredns -o yaml | grep -A 5 forward

# Should have:
# forward . /etc/resolv.conf

# Test external DNS
kubectl exec -it my-pod -- nslookup google.com

# If fails, check node's DNS
kubectl debug node/my-node -it --image=nicolaka/netshoot -- bash
cat /etc/resolv.conf
nslookup google.com
```

#### Problem: Slow external DNS queries (ndots issue)

```bash
# Confirm multiple queries being made
kubectl exec -it my-pod -- tcpdump -i eth0 -n port 53

# Try querying google.com, will see 4 queries:
# 1. google.com.default.svc.cluster.local
# 2. google.com.svc.cluster.local
# 3. google.com.cluster.local
# 4. google.com

# Fix: Use FQDN with trailing dot or lower ndots
curl http://google.com.  # Trailing dot = absolute query
```

### DNS Debugging Commands

```bash
# From inside pod:
nslookup my-app
nslookup my-app.default.svc.cluster.local
dig my-app.default.svc.cluster.local
dig @10.96.0.10 my-app.default.svc.cluster.local  # Query CoreDNS directly

# Check SRV records
dig SRV _http._tcp.my-app.default.svc.cluster.local

# Check what DNS server is being used
cat /etc/resolv.conf

# Trace DNS queries
tcpdump -i eth0 -n port 53

# CoreDNS metrics
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
curl http://localhost:9153/metrics | grep coredns_dns_requests_total
```

---

## 18.5 Network Policies

By default, Kubernetes allows all traffic between pods. **NetworkPolicies** restrict traffic to specific pods based on labels.

### Default Behavior (No Policies)

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│ Pod A    │────────▶│ Pod B    │────────▶│ Pod C    │
│ frontend │         │ backend  │         │ database │
└──────────┘         └──────────┘         └──────────┘
     │                                         ▲
     └─────────────────────────────────────────┘
              All traffic allowed
```

### Ingress Policy (Incoming Traffic)

**Deny all ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
    - Ingress
  # No ingress rules = deny all ingress
```

```bash
kubectl apply -f deny-all-ingress.yaml

# Test: Traffic blocked
kubectl run frontend --image=nginx
kubectl run backend --image=nginx
kubectl exec frontend -- curl backend  # Timeout!
```

**Allow ingress from specific pods:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend  # Apply to backend pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend  # Allow from frontend pods
      ports:
        - protocol: TCP
          port: 80
```

**Allow ingress from specific namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring-ns
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring  # Namespace must have this label
      ports:
        - protocol: TCP
          port: 9090  # Prometheus scrape port
```

**Allow ingress from IP range (external):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-cidr
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 192.168.1.0/24
            except:
              - 192.168.1.5/32  # Block this specific IP
      ports:
        - protocol: TCP
          port: 443
```

### Egress Policy (Outgoing Traffic)

**Deny all egress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = deny all egress
```

**Allow egress to specific pods:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
```

**Allow egress to DNS and external:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-external
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # Allow external HTTPS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # Block metadata service
      ports:
        - protocol: TCP
          port: 443
```

### Combined Ingress + Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-isolation
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only allow from frontend
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Only allow to database and DNS
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

### Debugging Blocked Traffic

**Verify NetworkPolicy is applied:**

```bash
kubectl get networkpolicy
kubectl describe networkpolicy deny-all-ingress
```

**Test connectivity:**

```bash
# Source pod
kubectl run source --image=nicolaka/netshoot -it --rm -- bash

# Inside source pod, test connection
curl http://target-pod-ip:8080
# If timeout, traffic is blocked

# Check if blocked by NetworkPolicy or other issue
# Try connecting to pod on same node first
```

**Check which policies affect a pod:**

```bash
kubectl get networkpolicy -o yaml | grep -A 20 "app: backend"
```

**Temporarily allow all traffic for testing:**

```bash
kubectl delete networkpolicy --all  # Restore default allow-all behavior
```

**Verify CNI supports NetworkPolicies:**

```bash
# Not all CNI plugins support NetworkPolicies!
# Check CNI documentation:
# ✓ Calico, Cilium, Weave: Full support
# ✗ Flannel: No support (requires another CNI)

# With Calico, check policy rules:
kubectl exec -n kube-system calico-node-xxxxx -- iptables -L -n | grep cali
```

### CNI Support Matrix for NetworkPolicies

| CNI Plugin | NetworkPolicy Support | Notes |
|------------|----------------------|-------|
| Calico | Full (L3/L4) | Uses iptables or eBPF |
| Cilium | Full (L3-L7) | Uses eBPF, supports HTTP rules |
| Weave | Full (L3/L4) | Uses iptables |
| Flannel | None | No NetworkPolicy support |
| Canal | Full (L3/L4) | Flannel + Calico |
| Antrea | Full (L3/L4) | VMware CNI |

---

## 18.6 Ingress & Gateway API

**Ingress** provides HTTP/HTTPS routing to Services from outside the cluster. It consolidates routing rules and reduces the need for multiple LoadBalancer Services.

### Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress

# NAME         CLASS    HOSTS                ADDRESS         PORTS   AGE
# my-ingress   nginx    myapp.example.com    35.123.45.67    80      5m
```

**Traffic flow:**

```
Internet
   │
   ▼
Ingress Controller (nginx, 35.123.45.67)
   │
   ├─ Host: myapp.example.com, Path: / → my-app:80
   └─ Host: myapp.example.com, Path: /api → api-service:8080
```

### NGINX Ingress Controller

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx ingress-nginx-controller

# NAME                       TYPE           EXTERNAL-IP      PORT(S)
# ingress-nginx-controller   LoadBalancer   35.123.45.67     80:30080/TCP,443:30443/TCP
```

### Common Ingress Annotations (NGINX)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    # Rewrite paths
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Custom timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"

    # Backend protocol
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /app(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### TLS with cert-manager

**Install cert-manager:**

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
```

**Create ClusterIssuer (Let's Encrypt):**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

**Ingress with TLS:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress-tls
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls  # cert-manager creates this Secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-tls.yaml

# Check certificate
kubectl get certificate
# NAME        READY   SECRET       AGE
# myapp-tls   True    myapp-tls    2m

kubectl describe certificate myapp-tls
```

### Gateway API (Next-Gen Ingress)

Gateway API is the successor to Ingress, providing more flexibility and extensibility.

**Install Gateway API CRDs:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

**Gateway (infrastructure):**

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: nginx  # Or istio, gke-l7-global-external-managed, etc.
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: myapp-tls
```

**HTTPRoute (routing rules):**

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: default
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - myapp.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app
          port: 80
```

**Advanced: Traffic splitting (canary deployments):**

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - myapp.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-v1
          port: 80
          weight: 90  # 90% to stable
        - name: my-app-v2
          port: 80
          weight: 10  # 10% to canary
```

### Ingress vs Gateway API Comparison

| Aspect | Ingress | Gateway API |
|--------|---------|-------------|
| **API Maturity** | Stable (GA) | Beta (v1.0) |
| **Role Separation** | Single resource | Gateway (infra) + HTTPRoute (routing) |
| **Extensibility** | Annotations (vendor-specific) | Native fields |
| **Protocol Support** | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC |
| **Traffic Splitting** | Via annotations | Native weight field |
| **Header Manipulation** | Via annotations | Native rules |
| **Cross-namespace** | Limited | Full support (ReferenceGrant) |
| **Vendor Lock-in** | High (annotations) | Low (standard API) |

---

## 18.7 Service Mesh - Istio

A **service mesh** adds observability, security, and traffic management to microservices without changing application code.

### Istio Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Control Plane (istiod)                                      │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ istiod (single binary)                               │  │
│  │                                                      │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐    │  │
│  │  │ Pilot      │  │ Citadel    │  │ Galley     │    │  │
│  │  │ (config)   │  │ (certs)    │  │ (validation)│   │  │
│  │  └────────────┘  └────────────┘  └────────────┘    │  │
│  │                                                      │  │
│  │  Pushes config to Envoy sidecars via xDS protocol   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼ (xDS config)
┌─────────────────────────────────────────────────────────────┐
│ Data Plane (Envoy sidecars)                                 │
│                                                             │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  │
│  │ Pod A                   │  │ Pod B                   │  │
│  │  ┌─────────┐            │  │  ┌─────────┐            │  │
│  │  │  App    │            │  │  │  App    │            │  │
│  │  └────┬────┘            │  │  └────┬────┘            │  │
│  │       │                 │  │       │                 │  │
│  │  ┌────┴────┐            │  │  ┌────┴────┐            │  │
│  │  │ Envoy   │◄───────────┼──┼─►│ Envoy   │            │  │
│  │  │ Sidecar │  mTLS      │  │  │ Sidecar │            │  │
│  │  └─────────┘            │  │  └─────────┘            │  │
│  └─────────────────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Key components:**
- **istiod**: Control plane managing configuration and certificates
- **Envoy sidecar**: Proxy injected into each pod, handles all traffic
- **mTLS**: Automatic mutual TLS between services
- **Telemetry**: Metrics, logs, traces collected from Envoy

### Installing Istio

```bash
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio (demo profile)
istioctl install --set profile=demo -y

# Verify installation
kubectl get pods -n istio-system
# NAME                                    READY   STATUS
# istiod-xxxxx                            1/1     Running
# istio-ingressgateway-xxxxx              1/1     Running
# istio-egressgateway-xxxxx               1/1     Running

# Enable sidecar injection for namespace
kubectl label namespace default istio-injection=enabled

# Deploy sample app
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Check sidecars injected
kubectl get pods
# NAME                              READY   STATUS
# productpage-v1-xxxxx              2/2     Running  # 2/2 = app + sidecar
```

### VirtualService (Traffic Routing)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews  # Service name
  http:
    - match:
        - headers:
            user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2  # Route to v2 for user "jason"
    - route:
        - destination:
            host: reviews
            subset: v1  # Default route to v1
```

### DestinationRule (Service Subsets)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
```

### Traffic Splitting (Canary Deployment)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - route:
        - destination:
            host: reviews
            subset: v1
          weight: 80  # 80% to v1
        - destination:
            host: reviews
            subset: v2
          weight: 20  # 20% to v2 (canary)
```

### Fault Injection (Testing Resilience)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
    - ratings
  http:
    - fault:
        delay:
          percentage:
            value: 10  # 10% of requests
          fixedDelay: 5s  # 5 second delay
        abort:
          percentage:
            value: 5  # 5% of requests
          httpStatus: 500  # Return 500 error
      route:
        - destination:
            host: ratings
            subset: v1
```

### mTLS (Mutual TLS)

```yaml
# Require mTLS for entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # Only allow mTLS traffic

# Check mTLS status
istioctl authn tls-check productpage-xxxxx.default ratings.default.svc.cluster.local
```

### istioctl Debugging Commands

```bash
# Check proxy configuration
istioctl proxy-config cluster productpage-xxxxx
istioctl proxy-config route productpage-xxxxx
istioctl proxy-config listener productpage-xxxxx
istioctl proxy-config endpoint productpage-xxxxx

# Analyze mesh configuration
istioctl analyze

# View metrics
istioctl dashboard prometheus
istioctl dashboard grafana
istioctl dashboard kiali  # Service mesh visualization

# Check why pod has no sidecar
istioctl analyze pod/my-pod

# Capture traffic (like tcpdump)
istioctl dashboard envoy productpage-xxxxx
# Navigate to: /stats/prometheus
```

### When NOT to Use a Service Mesh

Service meshes add complexity. Avoid if:

- **Small cluster**: < 10 services, overhead not justified
- **Simple traffic patterns**: Basic load balancing sufficient
- **Performance critical**: Sidecar adds ~1-2ms latency
- **Team unfamiliar**: Steep learning curve
- **Already solved**: Existing ingress/load balancer sufficient

**Use service mesh when you need:**
- Advanced traffic management (canary, blue-green, circuit breaking)
- mTLS between services without code changes
- Centralized observability (metrics, traces, logs)
- Complex multi-cluster or hybrid cloud setups

---

## 18.8 Network Troubleshooting Playbook

### Systematic Troubleshooting Approach

```
Problem: Pod A cannot reach Pod B

Step 1: Verify basic connectivity
   │
   ├─► Can Pod A ping itself? (ip addr, ping 127.0.0.1)
   │       └─► No? Container networking broken
   │
   ├─► Can Pod A ping its gateway? (ip route, ping gateway)
   │       └─► No? veth pair or bridge issue
   │
   ├─► Can Pod A resolve DNS? (nslookup kubernetes.default)
   │       └─► No? DNS configuration issue (see 18.4)
   │
   ├─► Can Pod A reach Pod B by IP? (curl http://POD_B_IP:PORT)
   │       └─► No? Network policy or routing issue
   │
   └─► Can Pod A reach Pod B by Service name? (curl http://service:port)
           └─► No? kube-proxy or Service configuration issue
```

### tcpdump for Packet Analysis

```bash
# On pod
kubectl exec -it pod-a -- tcpdump -i eth0 -n

# On node (for pod traffic)
# Find pod's veth interface
POD_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
VETH=$(ip addr | grep -B 2 $POD_IP | head -n 1 | awk '{print $2}' | tr -d ':')
tcpdump -i $VETH -n

# Specific filters
tcpdump -i eth0 host 10.244.1.5  # Specific IP
tcpdump -i eth0 port 80  # Specific port
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'  # SYN packets only

# Write to file for analysis
tcpdump -i eth0 -w /tmp/capture.pcap
# Copy to local machine
kubectl cp pod-a:/tmp/capture.pcap ./capture.pcap
# Analyze with Wireshark
```

### MTU Issues

Packets larger than Maximum Transmission Unit (MTU) get fragmented or dropped.

```bash
# Check MTU (inside pod)
ip link show eth0
# mtu 1450 (common in overlay networks)

# Test MTU with ping
ping -M do -s 1400 10.244.2.8  # Success
ping -M do -s 1500 10.244.2.8  # May fail if MTU too small

# Check node MTU
ip link show eth0
# mtu 1500

# Common issue: Overlay network (VXLAN/IPIP) reduces MTU by 50 bytes
# Pod MTU should be: Node MTU - Overlay overhead
# VXLAN overhead: 50 bytes → Pod MTU = 1450
# IPIP overhead: 20 bytes → Pod MTU = 1480

# Fix: Set CNI MTU configuration
kubectl edit cm -n kube-system calico-config
# Set: veth_mtu: "1450"
```

### Conntrack Table Exhaustion

Connection tracking table can fill up in high-traffic clusters.

```bash
# Check conntrack usage (on node)
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# If count > 80% of max:
# 1. Increase max
sysctl -w net.netfilter.nf_conntrack_max=262144
echo "net.netfilter.nf_conntrack_max = 262144" >> /etc/sysctl.conf

# 2. Reduce timeout
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=600
echo "net.netfilter.nf_conntrack_tcp_timeout_established = 600" >> /etc/sysctl.conf

# 3. Check for connections in TIME_WAIT
conntrack -L | grep TIME_WAIT | wc -l

# 4. View top connection consumers
conntrack -L | awk '{print $5}' | sort | uniq -c | sort -rn | head -20
```

### Common Scenarios & Resolutions

#### Scenario 1: "Connection refused" when accessing Service

```bash
# Symptom
kubectl exec pod-a -- curl http://my-service
# curl: (7) Failed to connect to my-service port 80: Connection refused

# Diagnosis
# 1. Does Service exist?
kubectl get svc my-service

# 2. Does Service have endpoints?
kubectl get endpoints my-service
# If ENDPOINTS is <none>, no pods match selector

# 3. Check pod selector
kubectl get svc my-service -o yaml | grep selector -A 2
kubectl get pods -l app=my-app  # Should match selector

# 4. Are pods ready?
kubectl get pods -l app=my-app
# READY should be 1/1, STATUS should be Running

# Resolution
# Fix pod labels or Service selector
kubectl label pod my-app-xyz app=my-app  # Add missing label
# Or edit Service
kubectl edit svc my-service
```

#### Scenario 2: "Connection timeout" when accessing Service

```bash
# Symptom
kubectl exec pod-a -- curl http://my-service
# (hangs, then timeout after 60s)

# Diagnosis
# 1. Service has endpoints?
kubectl get endpoints my-service
# ENDPOINTS: 10.244.1.5:8080,10.244.1.6:8080

# 2. Can reach pod directly?
kubectl exec pod-a -- curl http://10.244.1.5:8080
# If this works, kube-proxy issue

# 3. Check iptables rules
kubectl get svc my-service -o jsonpath='{.spec.clusterIP}'
# 10.96.100.50
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.100.50

# 4. Check NetworkPolicy
kubectl get networkpolicy
kubectl describe networkpolicy deny-all-ingress

# Resolution A: NetworkPolicy blocking traffic
kubectl delete networkpolicy deny-all-ingress
# Or add explicit allow rule

# Resolution B: kube-proxy not running
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx
```

#### Scenario 3: DNS resolution fails

```bash
# Symptom
kubectl exec pod-a -- nslookup my-service
# ;; connection timed out; no servers could be reached

# Diagnosis & Resolution: See section 18.4 DNS debugging
```

#### Scenario 4: Ingress returns 502 Bad Gateway

```bash
# Symptom
curl http://myapp.example.com
# 502 Bad Gateway

# Diagnosis
# 1. Check Ingress controller logs
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx

# Common errors:
# "upstream connect error or disconnect/reset before headers"
#   → Backend Service not reachable

# 2. Check Ingress configuration
kubectl describe ingress my-ingress

# 3. Check backend Service and pods
kubectl get svc my-app
kubectl get endpoints my-app
kubectl get pods -l app=my-app

# 4. Test Service from Ingress controller pod
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxxx -- curl http://my-app.default.svc.cluster.local

# Resolution
# Fix backend Service or pod issues
```

#### Scenario 5: Intermittent connection failures

```bash
# Symptom
# Requests succeed ~90% of the time, fail ~10%

# Diagnosis
# 1. Check if one pod is failing
kubectl get pods -l app=my-app
# Look for restarts or non-ready pods

# 2. Check pod logs
kubectl logs my-app-xyz | grep -i error

# 3. Check pod readiness probe
kubectl describe pod my-app-xyz | grep Readiness
# If readiness fails, pod removed from endpoints temporarily

# 4. Check conntrack table
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# Resolution A: Fix failing pod
kubectl logs my-app-xyz  # Debug application issue

# Resolution B: Fix readiness probe
kubectl edit deployment my-app
# Adjust readiness probe (initialDelaySeconds, periodSeconds)

# Resolution C: Increase conntrack table
sysctl -w net.netfilter.nf_conntrack_max=262144
```

### Network Debugging Toolbox

Essential commands for network troubleshooting:

```bash
# Inside pod (use debug image if needed)
kubectl run netshoot --image=nicolaka/netshoot -it --rm -- bash

# Basic connectivity
ping 10.244.1.5
curl http://10.244.1.5:8080
telnet 10.244.1.5 8080

# DNS
nslookup my-service
dig my-service.default.svc.cluster.local
host my-service

# Network configuration
ip addr show
ip route show
ip link show
cat /etc/resolv.conf

# Port scanning
nc -zv 10.244.1.5 8080
nmap -p 8080 10.244.1.5

# Packet capture
tcpdump -i eth0 -n

# HTTP debugging
curl -v http://my-service
curl -I http://my-service  # Headers only

# On node
iptables -t nat -L -n -v
ipvsadm -Ln
conntrack -L
ss -tunap  # Active connections
netstat -antp
```

---

## Chapter 19 Review Questions

1. **Explain the Kubernetes networking model.** What are the four fundamental requirements? Why is the "flat network" model simpler than NAT-based approaches?

2. **Compare overlay networking (VXLAN/IPIP) vs native routing (BGP).** What are the tradeoffs? When would you choose each approach?

3. **How does kube-proxy implement Services in iptables mode?** Walk through the iptables chains for a Service with 3 endpoints. How does the probability calculation work for random load balancing?

4. **What causes slow external DNS resolution in Kubernetes?** Explain the ndots parameter and how search domains work. How can you optimize DNS performance?

5. **Describe the difference between Ingress and Gateway API.** What problems does Gateway API solve that Ingress cannot? When would you use each?

---

## Chapter 19 Hands-On Exercises

**Exercise 1: Diagnose Network Policy Issue**

Deploy a frontend and backend application. Apply a NetworkPolicy that denies all traffic. The frontend cannot reach the backend. Debug the issue, create an appropriate NetworkPolicy to allow only frontend-to-backend traffic, and verify connectivity.

```bash
# Deploy apps
kubectl create deployment frontend --image=nginx
kubectl create deployment backend --image=nginx
kubectl expose deployment backend --port=80

# Apply deny-all policy
kubectl apply -f deny-all-policy.yaml

# Your task: Create allow policy and verify
```

**Exercise 2: Debug DNS Resolution**

A pod cannot resolve external DNS (google.com) but can resolve internal Services (kubernetes.default). Investigate why, check CoreDNS logs, examine /etc/resolv.conf, and fix the issue.

```bash
kubectl run test --image=nicolaka/netshoot -it --rm -- bash
# Inside pod:
nslookup kubernetes.default  # Works
nslookup google.com  # Fails

# Your task: Diagnose and fix
```

**Exercise 3: Service Traffic Splitting with Istio**

Install Istio, deploy two versions of an application (v1 and v2), and configure a VirtualService to send 90% of traffic to v1 and 10% to v2. Generate traffic and verify the split ratio using metrics.

```bash
# Install Istio
istioctl install --set profile=demo -y

# Deploy app v1 and v2
kubectl apply -f app-v1.yaml
kubectl apply -f app-v2.yaml

# Your task: Configure VirtualService and verify traffic split
```

---

## Key Takeaways

- **Kubernetes networking model** requires flat networking (no NAT between pods), achieved through CNI plugins like Calico, Cilium, or Flannel with either overlay or native routing.

- **kube-proxy** implements Services using iptables (or IPVS) rules that DNAT ClusterIP traffic to pod endpoints; understanding iptables chains is essential for debugging Service connectivity.

- **CoreDNS** provides DNS resolution for Services (cluster.local domain) and pods; ndots and search domains can cause slow external DNS queries if not configured properly.

- **NetworkPolicies** default-deny all traffic when applied, requiring explicit ingress/egress rules; not all CNI plugins support NetworkPolicies (Flannel does not).

- **Ingress** consolidates HTTP/HTTPS routing rules and reduces LoadBalancer costs; Gateway API is the next-generation replacement with better role separation and extensibility.

- **Service meshes like Istio** provide advanced traffic management, mTLS, and observability but add complexity; only adopt when benefits justify the operational overhead.

- **Systematic troubleshooting** starts with basic connectivity (ping, DNS, direct pod IP) and progresses to Service/Ingress/NetworkPolicy debugging; tcpdump, iptables inspection, and conntrack monitoring are essential tools.

---

[Next Chapter: Kubernetes Security & Access Control →](19-security.md)
