# Chapter 4: Networking on GCP

## 4.1 VPC Overview

Virtual Private Cloud (VPC) is a global, software-defined network.

### GCP VPC vs AWS VPC

| Feature | GCP VPC | AWS VPC |
|---------|---------|---------|
| Scope | Global | Regional |
| Subnets | Regional | Zonal (AZ) |
| Default VPC | Auto mode (one subnet per region) | One default per region |
| Firewall | Global firewall rules | Security Groups per resource |

### VPC Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        GCP Project                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     VPC Network (Global)                   │  │
│  │                                                            │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐       │  │
│  │  │  us-central1 (Region)│  │  asia-southeast1      │       │  │
│  │  │  ┌────────────────┐  │  │  ┌────────────────┐  │       │  │
│  │  │  │ Subnet A       │  │  │  │ Subnet C       │  │       │  │
│  │  │  │ 10.0.0.0/24    │  │  │  │ 10.2.0.0/24    │  │       │  │
│  │  │  │  ┌────┐ ┌────┐ │  │  │  │  ┌────┐       │  │       │  │
│  │  │  │  │VM1 │ │VM2 │ │  │  │  │  │VM4 │       │  │       │  │
│  │  │  │  └────┘ └────┘ │  │  │  │  └────┘       │  │       │  │
│  │  │  └────────────────┘  │  │  └────────────────┘  │       │  │
│  │  │  ┌────────────────┐  │  │                      │       │  │
│  │  │  │ Subnet B       │  │  └──────────────────────┘       │  │
│  │  │  │ 10.1.0.0/24    │  │                                 │  │
│  │  │  │  ┌────┐        │  │                                 │  │
│  │  │  │  │VM3 │        │  │                                 │  │
│  │  │  │  └────┘        │  │                                 │  │
│  │  │  └────────────────┘  │                                 │  │
│  │  └──────────────────────┘                                 │  │
│  │                                                            │  │
│  │  Firewall Rules (apply to entire VPC)                     │  │
│  │  Routes (apply to entire VPC)                             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Auto Mode vs Custom Mode

**Auto Mode**:
- One subnet auto-created in each region
- IP ranges are predetermined (10.128.0.0/9)
- Good for getting started
- Can convert to custom mode (one-way)

**Custom Mode**:
- You create subnets explicitly
- Full control over IP ranges
- Recommended for production
- Required for VPC peering with overlapping ranges

```bash
# Create custom mode VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom

# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24
```

### IP Addressing

**Internal IPs**:
- Assigned from subnet range
- Private RFC 1918 addresses
- Can be ephemeral or static

**External IPs**:
- Public internet-routable addresses
- Ephemeral (changes on restart) or static
- Required for direct internet access

**Alias IP Ranges**:
- Additional IP ranges for a VM
- Used for containers (GKE pods)

```bash
# Reserve static internal IP
gcloud compute addresses create my-internal-ip \
  --subnet=my-subnet \
  --region=us-central1 \
  --addresses=10.0.0.10

# Reserve static external IP
gcloud compute addresses create my-external-ip \
  --region=us-central1
```

### Private Google Access

Allow VMs without external IPs to reach Google APIs:

```bash
# Enable Private Google Access on subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access
```

### Private Service Connect

Private connectivity to Google APIs and services:

```bash
# Create endpoint for Google APIs
gcloud compute addresses create google-apis-endpoint \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.0.100 \
  --network=my-vpc

gcloud compute forwarding-rules create google-apis-rule \
  --global \
  --network=my-vpc \
  --address=google-apis-endpoint \
  --target-google-apis-bundle=all-apis
```

---

## 4.2 Firewall Rules

GCP firewalls are stateful and applied at the VPC level.

### Firewall Rule Components

```yaml
Rule:
  Name: allow-ssh
  Network: my-vpc
  Priority: 1000           # Lower = higher priority
  Direction: INGRESS       # or EGRESS
  Action: ALLOW            # or DENY
  Targets: all-instances   # or specific tags/service accounts
  Source: 0.0.0.0/0        # For ingress
  Destination: 10.0.0.0/8  # For egress
  Protocols: tcp:22
```

### Implied Rules

Every VPC has these implied rules (cannot be deleted):

1. **Implied allow egress** (priority 65535): Allow all outbound
2. **Implied deny ingress** (priority 65535): Deny all inbound

### Creating Rules

```bash
# Allow SSH from anywhere
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --allow=tcp:22 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=ssh-enabled

# Allow internal communication
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

# Allow health checks from Google
gcloud compute firewall-rules create allow-health-checks \
  --network=my-vpc \
  --allow=tcp \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=load-balanced
```

### Targeting

**Network tags**: Strings applied to VMs

```bash
# Apply tag to VM
gcloud compute instances add-tags my-vm \
  --tags=web-server,ssh-enabled \
  --zone=us-central1-a
```

**Service accounts**: More secure than tags

```bash
gcloud compute firewall-rules create allow-backend \
  --network=my-vpc \
  --allow=tcp:8080 \
  --source-service-accounts=frontend@project.iam.gserviceaccount.com \
  --target-service-accounts=backend@project.iam.gserviceaccount.com
```

### Firewall Policies (Hierarchical)

Organization-wide firewall rules:

```bash
# Create policy
gcloud compute firewall-policies create my-policy \
  --organization=ORG_ID

# Add rule
gcloud compute firewall-policies rules create 1000 \
  --firewall-policy=my-policy \
  --action=allow \
  --direction=INGRESS \
  --src-ip-ranges=0.0.0.0/0 \
  --layer4-configs=tcp:22

# Associate with folder or organization
gcloud compute firewall-policies associations create \
  --firewall-policy=my-policy \
  --organization=ORG_ID
```

---

## 4.3 Routes

Determine how traffic is directed within and out of the VPC.

### Route Types

**System-generated routes**:
- Default route to internet gateway (0.0.0.0/0)
- Subnet routes (auto-created for each subnet)

**Custom routes**:
- Static routes (manually created)
- Dynamic routes (from Cloud Router/BGP)

```bash
# Create static route
gcloud compute routes create route-to-vpn \
  --network=my-vpc \
  --destination-range=192.168.0.0/16 \
  --next-hop-vpn-tunnel=my-tunnel \
  --priority=1000
```

### Next Hop Types

| Next Hop | Use Case |
|----------|----------|
| Default internet gateway | Internet access |
| Instance | Route through a VM (NAT, firewall appliance) |
| Internal TCP/UDP Load Balancer | Route to ILB |
| VPN tunnel | Route to on-premises |
| Interconnect attachment | Route to on-premises (dedicated) |

---

## 4.4 Cloud NAT

Network Address Translation for outbound internet access without external IPs.

### How Cloud NAT Works

```
┌─────────────────────────────────────────────────────────────┐
│                         VPC                                  │
│  ┌─────────────────┐      ┌─────────────────┐               │
│  │ VM (no ext IP)  │      │ VM (no ext IP)  │               │
│  │   10.0.0.2      │      │   10.0.0.3      │               │
│  └────────┬────────┘      └────────┬────────┘               │
│           │                        │                         │
│           └────────────┬───────────┘                         │
│                        │                                     │
│                        ▼                                     │
│              ┌──────────────────┐                            │
│              │    Cloud NAT     │                            │
│              │ (translates IPs) │                            │
│              └────────┬─────────┘                            │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │
                        ▼
                   Internet
```

### Creating Cloud NAT

```bash
# First, create a Cloud Router
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# Create Cloud NAT
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

### NAT Configuration Options

**IP allocation**:
- Auto-allocate (Google manages IPs)
- Manual (you specify IPs - useful for whitelisting)

**Subnet scope**:
- All subnets in region
- Specific subnets only
- Primary ranges only (exclude secondary/alias ranges)

---

## 4.5 Cloud Load Balancing

GCP offers multiple load balancer types:

### Load Balancer Comparison

| Type | Scope | Layer | Use Case |
|------|-------|-------|----------|
| **External HTTP(S) LB** | Global | L7 | Web apps, CDN |
| **External TCP/UDP LB** | Regional | L4 | Non-HTTP TCP/UDP |
| **External TCP Proxy** | Global | L4 | TCP with SSL offload |
| **External SSL Proxy** | Global | L4 | SSL termination |
| **Internal HTTP(S) LB** | Regional | L7 | Internal microservices |
| **Internal TCP/UDP LB** | Regional | L4 | Internal non-HTTP |

### External HTTP(S) Load Balancer

```
                     ┌─────────────────────────────────┐
                     │   Global External HTTP(S) LB    │
                     │                                 │
                     │  ┌─────────────────────────┐   │
     Internet ─────▶│  │   Forwarding Rule       │   │
                     │  │   (Global IP)           │   │
                     │  └───────────┬─────────────┘   │
                     │              │                  │
                     │  ┌───────────▼─────────────┐   │
                     │  │    Target HTTP Proxy    │   │
                     │  └───────────┬─────────────┘   │
                     │              │                  │
                     │  ┌───────────▼─────────────┐   │
                     │  │        URL Map          │   │
                     │  │  (routing rules)        │   │
                     │  └───────────┬─────────────┘   │
                     │              │                  │
                     │  ┌───────────▼─────────────┐   │
                     │  │    Backend Service      │   │
                     │  │  ┌─────────────────┐   │   │
                     │  │  │ Backend (MIG)   │   │   │
                     │  │  │ Backend (NEG)   │   │   │
                     │  │  └─────────────────┘   │   │
                     │  └─────────────────────────┘   │
                     └─────────────────────────────────┘
```

### Creating HTTP(S) Load Balancer

```bash
# 1. Create instance group
gcloud compute instance-groups managed create web-servers \
  --template=web-template \
  --size=3 \
  --zone=us-central1-a

# 2. Create health check
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/health

# 3. Create backend service
gcloud compute backend-services create web-backend \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=web-health-check \
  --global

# 4. Add backend
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-servers \
  --instance-group-zone=us-central1-a \
  --global

# 5. Create URL map
gcloud compute url-maps create web-map \
  --default-service=web-backend

# 6. Create target proxy
gcloud compute target-http-proxies create web-proxy \
  --url-map=web-map

# 7. Create forwarding rule (gets global IP)
gcloud compute forwarding-rules create web-forwarding \
  --global \
  --target-http-proxy=web-proxy \
  --ports=80
```

### Backend Types

**Instance Groups**:
- Managed Instance Groups (MIGs)
- Unmanaged Instance Groups

**Network Endpoint Groups (NEGs)**:
- Zonal NEG: Individual VMs or containers
- Serverless NEG: Cloud Run, App Engine, Cloud Functions
- Internet NEG: External endpoints
- Hybrid NEG: On-premises endpoints

### Cloud CDN

Enable caching at Google's edge:

```bash
gcloud compute backend-services update web-backend \
  --enable-cdn \
  --global
```

### SSL/TLS Certificates

**Google-managed certificates**:
```bash
gcloud compute ssl-certificates create my-cert \
  --domains=example.com,www.example.com \
  --global
```

**Self-managed certificates**:
```bash
gcloud compute ssl-certificates create my-cert \
  --certificate=cert.pem \
  --private-key=key.pem \
  --global
```

---

## 4.6 VPC Peering

Connect VPCs for private communication.

### Peering Characteristics

- Traffic stays on Google's network
- No external IPs needed
- Non-transitive (A↔B and B↔C doesn't mean A↔C)
- IP ranges cannot overlap

```bash
# Create peering from VPC-A to VPC-B
gcloud compute networks peerings create peer-a-to-b \
  --network=vpc-a \
  --peer-project=project-b \
  --peer-network=vpc-b

# Create peering from VPC-B to VPC-A (required from both sides)
gcloud compute networks peerings create peer-b-to-a \
  --network=vpc-b \
  --peer-project=project-a \
  --peer-network=vpc-a
```

### Shared VPC

Centralized network management across projects:

```
┌─────────────────────────────────────────────────────────────┐
│                      Host Project                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   Shared VPC                           │  │
│  │  ┌─────────────────┐  ┌─────────────────┐             │  │
│  │  │  Subnet A       │  │  Subnet B       │             │  │
│  │  │  (for Proj 1)   │  │  (for Proj 2)   │             │  │
│  │  └─────────────────┘  └─────────────────┘             │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
        │                          │
        │                          │
┌───────▼───────┐          ┌───────▼───────┐
│  Service      │          │  Service      │
│  Project 1    │          │  Project 2    │
│               │          │               │
│  VMs use      │          │  VMs use      │
│  Subnet A     │          │  Subnet B     │
└───────────────┘          └───────────────┘
```

```bash
# Enable Shared VPC in host project
gcloud compute shared-vpc enable HOST_PROJECT

# Associate service project
gcloud compute shared-vpc associated-projects add SERVICE_PROJECT \
  --host-project=HOST_PROJECT
```

---

## 4.7 Hybrid Connectivity

Connect on-premises to GCP.

### Cloud VPN

IPsec VPN over internet:

**Classic VPN**:
- Single tunnel
- Up to 3 Gbps

**HA VPN**:
- Two tunnels for 99.99% SLA
- Up to 3 Gbps per tunnel

```bash
# Create HA VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

# Create external VPN gateway (represents on-prem)
gcloud compute external-vpn-gateways create on-prem-gateway \
  --interfaces 0=ON_PREM_IP_1,1=ON_PREM_IP_2

# Create VPN tunnels
gcloud compute vpn-tunnels create tunnel-0 \
  --vpn-gateway=my-vpn-gateway \
  --peer-external-gateway=on-prem-gateway \
  --peer-external-gateway-interface=0 \
  --shared-secret=SECRET \
  --router=my-router \
  --interface=0

# Create BGP sessions
gcloud compute routers add-interface my-router \
  --interface-name=bgp-0 \
  --vpn-tunnel=tunnel-0 \
  --ip-address=169.254.0.1 \
  --mask-length=30

gcloud compute routers add-bgp-peer my-router \
  --peer-name=on-prem-peer \
  --interface=bgp-0 \
  --peer-ip-address=169.254.0.2 \
  --peer-asn=65001
```

### Cloud Interconnect

Dedicated physical connection:

**Dedicated Interconnect**:
- Direct connection to Google
- 10 Gbps or 100 Gbps circuits
- Requires colocation facility

**Partner Interconnect**:
- Through a service provider
- 50 Mbps to 50 Gbps
- No colocation needed

---

## 4.8 Network Troubleshooting

### Connectivity Tests

```bash
# Test connectivity
gcloud network-management connectivity-tests create my-test \
  --source-instance=projects/PROJECT/zones/ZONE/instances/SOURCE_VM \
  --destination-instance=projects/PROJECT/zones/ZONE/instances/DEST_VM \
  --protocol=TCP \
  --destination-port=80

# Get results
gcloud network-management connectivity-tests describe my-test
```

### VPC Flow Logs

Enable logging for network flows:

```bash
# Enable flow logs on subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=0.5 \
  --logging-metadata=include-all
```

### Firewall Rules Logging

```bash
gcloud compute firewall-rules update my-rule \
  --enable-logging
```

### Common Network Issues

**VM can't reach internet**:
1. Check external IP or Cloud NAT
2. Check firewall egress rules
3. Check routes (default route exists?)

**Can't reach VM from internet**:
1. Check VM has external IP
2. Check firewall ingress rules
3. Check health checks passing (for LB)

**VMs can't communicate**:
1. Check firewall rules allow internal traffic
2. Check VPC peering status
3. Check IP ranges don't overlap
4. Check routes

**Load balancer not working**:
1. Check backend health (health checks passing?)
2. Check firewall allows health check IPs
3. Check URL map configuration
4. Check backend service configuration

---

## Chapter 4 Review Questions

1. What's the difference between auto mode and custom mode VPCs?

2. A VM without an external IP needs to download packages from the internet. What do you configure?

3. Explain the difference between firewall rules using network tags vs service accounts.

4. How would you design network connectivity for a multi-region application?

5. A customer's load balancer is returning 502 errors. What would you check?

6. What's the difference between VPC peering and Shared VPC?

---

## Chapter 4 Hands-On Exercises

### Exercise 4.1: VPC Setup
1. Create a custom mode VPC
2. Create two subnets in different regions
3. Create firewall rules for SSH and internal communication
4. Create VMs in each subnet and verify connectivity

### Exercise 4.2: Cloud NAT
1. Create a VM without an external IP
2. Verify it can't reach the internet
3. Set up Cloud NAT
4. Verify internet access works

### Exercise 4.3: Load Balancer
1. Create a managed instance group with a web server
2. Create an HTTP(S) load balancer
3. Test the load balancer
4. Enable Cloud CDN

---

## Key Takeaways

1. **VPCs are global** - Subnets are regional
2. **Firewall rules are stateful** - Return traffic is automatic
3. **Use Cloud NAT** for outbound-only internet access
4. **Choose the right LB** - Global HTTP(S) for web, regional for others
5. **Private Google Access** - Required for VMs without external IPs to reach APIs
6. **Flow logs** are essential for troubleshooting

---

[Next Chapter: Vertex AI & Machine Learning →](./05-vertex-ai.md)
