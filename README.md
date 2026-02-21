# GCP Engineering Handbook

A comprehensive handbook for Google Cloud Platform engineering.

## Overview

This handbook covers GCP architecture, systems fundamentals, and practical troubleshooting skills for cloud engineers.

## Contents

### Part I: GCP Fundamentals
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [01](fundamentals/01-core-concepts.md) | Core Concepts | Resource hierarchy, IAM, service accounts, projects |
| [02](fundamentals/02-compute.md) | Compute Services | Compute Engine, GKE, Cloud Run, Cloud Functions |
| [03](fundamentals/03-storage-databases.md) | Storage & Databases | GCS, Cloud SQL, Spanner, Firestore, BigQuery |
| [04](fundamentals/04-networking.md) | GCP Networking | VPC, firewalls, load balancing, hybrid connectivity |

### Part II: AI/ML on GCP
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [05](ai-ml/05-vertex-ai.md) | Vertex AI | AutoML, custom training, MLOps, Feature Store |
| [06](ai-ml/06-generative-ai.md) | Generative AI | Gemini, embeddings, RAG, grounding, function calling |

### Part III: Networking Deep Dive
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [07](networking/07-tcpip.md) | TCP/IP | OSI model, TCP/UDP, routing, NAT, ICMP |
| [08](networking/08-dns-http.md) | DNS & HTTP | DNS resolution, HTTP methods, TLS, status codes |
| [23](networking/23-network-tracing-cli.md) | Network Tracing & CLI Tools | ping, traceroute, tcpdump, dig, ss, routing, iptables, curl, troubleshooting strategies |

### Part IV: Systems & Debugging
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [09](systems/09-linux.md) | Linux Systems | Process management, memory, disk I/O, systemd |
| [10](systems/10-containers-kubernetes.md) | Containers & K8s | Docker, Kubernetes, cgroups, perf, strace, eBPF, distributed debugging |
| [11](systems/11-troubleshooting.md) | Troubleshooting | Methodology, escalation management, on-call, SLOs/SLIs/SLAs |

### Part V: Distributed Systems
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [12](distributed-systems/12-distributed-systems.md) | Distributed Systems | CAP theorem, consistency, replication, patterns |

### Part VI: Programming & Tools
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [13](programming/13-go.md) | Go | Syntax, concurrency, goroutines, channels |
| [14](programming/14-python.md) | Python | GCP client libraries, automation, scripting |
| [15](programming/15-developer-tools.md) | Developer Tools | Automation, CLI tools, testing frameworks, debugging tools |

### Part VII: Kubernetes Operations
| Chapter | Topic | Key Concepts |
|---------|-------|--------------|
| [16](kubernetes/16-monitoring-observability.md) | K8s Monitoring & Observability | Prometheus, Grafana, kube-state-metrics, alerting, dashboards |
| [17](kubernetes/17-debugging.md) | K8s Debugging Deep Dive | Pod lifecycle, scheduling, storage, RBAC, Helm, CRDs, tools |
| [18](kubernetes/18-networking.md) | K8s Networking Deep Dive | CNI, kube-proxy, DNS, network policies, ingress, service mesh |
| [19](kubernetes/19-security.md) | K8s Security & Access Control | Pod Security Standards, secrets, image security, audit logging |
| [20](kubernetes/20-cluster-operations.md) | K8s Cluster Operations | Control plane, etcd, upgrades, capacity planning, DR, runbooks |
| [21](kubernetes/21-reliability.md) | K8s Reliability Engineering | HA patterns, chaos engineering, progressive delivery, auto-healing |
| [22](kubernetes/22-troubleshooting-scenarios.md) | K8s Troubleshooting Scenarios | Production scenarios, incident response, diagnostic workflows |

### Hands-on
| Resource | Description |
|----------|-------------|
| [Exercises](exercises/README.md) | Practical labs using GCP free tier |

## Quick Start

1. **Start with fundamentals** - Chapters 1-4 cover GCP basics
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
