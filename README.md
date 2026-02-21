# GCP Engineering Handbook

A comprehensive handbook for Google Cloud Platform engineering.

## Overview

This handbook covers GCP architecture, systems fundamentals, and practical troubleshooting skills for cloud engineers.

## Contents

### Part I: GCP Fundamentals
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [01](01-fundamentals/01-core-concepts.md) | Core Concepts | Resource hierarchy, IAM, service accounts, projects |
| [02](01-fundamentals/02-compute.md) | Compute Services | Compute Engine, GKE, Cloud Run, Cloud Functions |
| [03](01-fundamentals/03-storage-databases.md) | Storage & Databases | GCS, Cloud SQL, Spanner, Firestore, BigQuery |
| [04](01-fundamentals/04-networking.md) | GCP Networking | VPC, firewalls, load balancing, hybrid connectivity |
| [05](01-fundamentals/05-distributed-systems.md) | Distributed Systems | CAP theorem, consistency, replication, patterns |

### Part II: AI/ML on GCP
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [06](02-ai-ml/06-vertex-ai.md) | Vertex AI | AutoML, custom training, MLOps, Feature Store |
| [07](02-ai-ml/07-generative-ai.md) | Generative AI | Gemini, embeddings, RAG, grounding, function calling |

### Part III: Networking Deep Dive
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [08](03-networking/08-tcp-ip.md) | TCP/IP | OSI model, TCP/UDP, routing, NAT, ICMP |
| [09](03-networking/09-dns-http.md) | DNS & HTTP | DNS resolution, HTTP methods, TLS, status codes |
| [10](03-networking/10-network-debugging.md) | Network Debugging | ping, traceroute, tcpdump, dig, ss, routing, iptables, curl, troubleshooting strategies |

### Part IV: Systems & Debugging
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [11](04-systems/11-linux.md) | Linux Systems | Process management, memory, disk I/O, systemd |
| [12](04-systems/12-containers-kubernetes.md) | Containers & K8s | Docker, Kubernetes, cgroups, perf, strace, eBPF, distributed debugging |
| [13](04-systems/13-troubleshooting.md) | Troubleshooting | Methodology, escalation management, on-call, SLOs/SLIs/SLAs |

### Part V: Programming & Tools
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [14](05-programming/14-go.md) | Go | Syntax, concurrency, goroutines, channels |
| [15](05-programming/15-python.md) | Python | GCP client libraries, automation, scripting |
| [16](05-programming/16-developer-tools.md) | Developer Tools | Automation, CLI tools, testing frameworks, debugging tools |

### Part VI: Kubernetes Operations
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [17](06-kubernetes/17-monitoring-observability.md) | K8s Monitoring & Observability | Prometheus, Grafana, kube-state-metrics, alerting, dashboards |
| [18](06-kubernetes/18-debugging.md) | K8s Debugging Deep Dive | Pod lifecycle, scheduling, storage, RBAC, Helm, CRDs, tools |
| [19](06-kubernetes/19-networking.md) | K8s Networking Deep Dive | CNI, kube-proxy, DNS, network policies, ingress, service mesh |
| [20](06-kubernetes/20-security.md) | K8s Security & Access Control | Pod Security Standards, secrets, image security, audit logging |
| [21](06-kubernetes/21-cluster-operations.md) | K8s Cluster Operations | Control plane, etcd, upgrades, capacity planning, DR, runbooks |
| [22](06-kubernetes/22-reliability.md) | K8s Reliability Engineering | HA patterns, chaos engineering, progressive delivery, auto-healing |
| [23](06-kubernetes/23-troubleshooting-scenarios.md) | K8s Troubleshooting Scenarios | Production scenarios, incident response, diagnostic workflows |

### Hands-on
| Resource | Description |
|----------|-------------|
| [Exercises](07-exercises/README.md) | Practical labs using GCP free tier |

## Quick Start

1. **Start with fundamentals** - Chapters 1-5 cover GCP basics
2. **Deep dive on interests** - AI/ML, networking, or systems
3. **Practice hands-on** - Each chapter has exercises
4. **Test yourself** - Use review questions at end of chapters

## Prerequisites

- Basic programming knowledge
- Familiarity with command line
- (Helpful) AWS or other cloud experience

## How to Use This Guide

### For Learning
- Read chapters in order for comprehensive understanding
- Do hands-on exercises in GCP free tier
- Answer review questions without looking

### For Reference
- Use as lookup during work
- Search for specific topics
- Bookmark frequently used sections

## Resources

### Official Google Resources
- [Google Cloud Documentation](https://cloud.google.com/docs)
- [Google Cloud Skills Boost](https://www.cloudskillsboost.google/)
- [Google SRE Books](https://sre.google/books/)

### Certifications
- Professional Cloud Architect
- Professional Cloud DevOps Engineer
- Professional Machine Learning Engineer

## License

MIT
