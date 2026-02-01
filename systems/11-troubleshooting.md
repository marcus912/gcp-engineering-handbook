# Chapter 11: Troubleshooting Methodology

This chapter is crucial for the Customer Engineer role. You'll be solving customer problems daily.

## 11.1 The Troubleshooting Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                   Troubleshooting Process                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. IDENTIFY    →   What is the problem?                        │
│       │              What are the symptoms?                      │
│       ↓              When did it start?                         │
│                                                                  │
│  2. SCOPE       →   Who/what is affected?                       │
│       │              Is it intermittent or constant?            │
│       ↓              What's the impact?                         │
│                                                                  │
│  3. GATHER DATA →   Logs, metrics, traces                       │
│       │              Error messages                              │
│       ↓              Recent changes                              │
│                                                                  │
│  4. HYPOTHESIZE →   What could cause this?                      │
│       │              Most likely → least likely                  │
│       ↓                                                          │
│                                                                  │
│  5. TEST        →   Verify hypothesis                           │
│       │              One change at a time                        │
│       ↓              Document what you tried                     │
│                                                                  │
│  6. RESOLVE     →   Implement fix                               │
│       │              Verify resolution                           │
│       ↓              Document solution                           │
│                                                                  │
│  7. POSTMORTEM  →   Why did it happen?                          │
│                      How do we prevent recurrence?               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11.2 Gathering Information

### Questions to Ask the Customer

```
WHEN:
- When did the problem start?
- Is it constant or intermittent?
- What time zone are you in?

WHAT:
- What exactly is the error message?
- What were you trying to do?
- What's the expected behavior?

WHERE:
- Which environment (prod, staging, dev)?
- Which region/zone?
- Which service/component?

CHANGES:
- Did anything change recently?
- New deployments?
- Configuration changes?
- Infrastructure changes?

SCOPE:
- Is everyone affected or specific users?
- Is it one service or multiple?
- Can you reproduce it?
```

### Essential Data to Collect

```bash
# For GCP issues
- Project ID
- Resource name/ID
- Region/zone
- Error messages (exact text)
- Timestamps (with timezone)
- Relevant logs
- Recent changes

# Commands to run
gcloud config list                    # Current config
gcloud services list --enabled        # Enabled APIs
gcloud logging read "FILTER" --limit=50
```

---

## 11.3 Common Troubleshooting Scenarios

### Scenario 1: Application Returning 5xx Errors

```
Step 1: Identify which 5xx
─────────────────────────
500 = App error → Check app logs
502 = Bad gateway → Check upstream
503 = Unavailable → Check capacity
504 = Timeout → Check latency

Step 2: Check the request path
─────────────────────────────
Client → LB → App → Database
        │     │      │
        │     │      └── Check DB connectivity/queries
        │     └── Check app logs, health
        └── Check LB config, backend health

Step 3: Narrow down
───────────────────
- Is it all requests or specific?
- Is it all backends or one?
- Does it correlate with traffic?

Step 4: Common fixes
────────────────────
500: Fix application bug
502: Fix/restart upstream, check health checks
503: Scale up, check resource limits
504: Increase timeouts, optimize slow code
```

### Scenario 2: Can't Connect to Database

```
Checklist:
□ Is the database running?
  → Cloud Console, gcloud sql instances describe

□ Is the network path clear?
  → Private IP: VPC peering, firewall rules
  → Public IP: Authorized networks, firewall

□ Are credentials correct?
  → Username, password, database name

□ Is the client configured correctly?
  → Connection string, SSL settings

□ Are there connection limits?
  → Max connections, connection pooling

Debug commands:
─────────────
# Test connectivity
nc -zv DB_IP 3306

# Check from VM in same VPC
gcloud compute ssh vm-name --zone=ZONE

# Check Cloud SQL
gcloud sql instances describe INSTANCE
gcloud sql instances list
```

### Scenario 3: High Latency

```
Layer-by-layer analysis:
────────────────────────

1. Network latency
   - traceroute to destination
   - Check for packet loss (mtr)
   - Check cross-region traffic

2. Load balancer
   - Health check latency
   - Backend distribution
   - Connection draining

3. Application
   - Response time metrics
   - GC pauses
   - Resource contention

4. Database
   - Query execution time
   - Lock contention
   - Connection pool exhaustion

5. External dependencies
   - Third-party API latency
   - DNS resolution time

Tools:
──────
- Cloud Trace (distributed tracing)
- Cloud Profiler (code profiling)
- Cloud Monitoring (metrics)
- Slow query logs
```

### Scenario 4: GKE Pod Issues

```
Pod not starting:
────────────────
kubectl describe pod POD_NAME

Events show:
- "FailedScheduling" → Resource constraints, node selector
- "FailedMount" → Volume issues, secrets not found
- "ErrImagePull" → Image doesn't exist, no pull access

Container crashing:
─────────────────
kubectl logs POD_NAME --previous

Check for:
- Application errors
- Missing configuration
- OOMKilled (memory limit)
- Liveness probe failures

Service not working:
──────────────────
kubectl get endpoints SVC_NAME

If empty:
- Selector doesn't match pod labels
- Pods not ready
- Network policy blocking
```

---

## 11.4 Reading Logs Effectively

### Log Levels

```
FATAL/CRITICAL → Application cannot continue
ERROR          → Something failed, but app continues
WARNING        → Potential problem
INFO           → Normal operations
DEBUG          → Detailed troubleshooting info
TRACE          → Very verbose
```

### What to Look For

```
1. ERROR and FATAL first
   - Stack traces
   - Exception messages

2. Timestamps
   - When did errors start?
   - Correlate with incidents

3. Request IDs
   - Trace requests across services

4. Patterns
   - Same error repeated?
   - Correlation with time/traffic?

5. Context
   - What happened before the error?
   - What was the request?
```

### GCP Logging Queries

```bash
# Basic error search
resource.type="gce_instance"
severity>=ERROR

# Specific service
resource.type="cloud_run_revision"
resource.labels.service_name="my-service"

# Time range
timestamp>="2024-01-15T10:00:00Z"
timestamp<="2024-01-15T11:00:00Z"

# Text search
textPayload:"connection refused"

# JSON field
jsonPayload.status_code=500

# Combine
resource.type="gke_container"
severity=ERROR
jsonPayload.message:"timeout"
```

---

## 11.5 Using Metrics

### Key Metrics to Monitor

```
Golden Signals (per Google SRE):
───────────────────────────────
1. LATENCY    → How long requests take
2. TRAFFIC    → How much demand
3. ERRORS     → Rate of failures
4. SATURATION → How full is the system

Specific metrics:
────────────────
CPU:     utilization, throttling
Memory:  usage, OOM events
Disk:    usage, IOPS, latency
Network: bandwidth, errors, retransmits
```

### Interpreting Metrics

```
High CPU + Normal latency = Working hard, OK
High CPU + High latency = CPU bottleneck
Normal CPU + High latency = Waiting on something (I/O, network, locks)
High errors + High latency = System degraded
High errors + Normal latency = Bug in error path
```

---

## 11.6 Incident Response

### Severity Levels

| Level | Impact | Response Time | Example |
|-------|--------|---------------|---------|
| **P1** | Critical | Immediate | Production down |
| **P2** | High | < 1 hour | Major feature broken |
| **P3** | Medium | < 4 hours | Minor feature broken |
| **P4** | Low | < 1 day | Cosmetic issue |

### Incident Response Process

```
1. ACKNOWLEDGE
   - Confirm the issue exists
   - Assign an owner
   - Start communication

2. TRIAGE
   - Determine severity
   - Identify affected systems
   - Estimate impact

3. MITIGATE
   - Focus on restoring service
   - Workarounds are OK
   - Don't need root cause yet

4. COMMUNICATE
   - Update stakeholders
   - Set expectations
   - Be honest about status

5. RESOLVE
   - Fix the underlying issue
   - Verify the fix
   - Monitor for recurrence

6. POSTMORTEM
   - Document timeline
   - Identify root cause
   - Create action items
```

### Communication During Incidents

```
Good update:
"We've identified a database connectivity issue affecting the
checkout service. We're working on restoring connections.
ETA for resolution: 30 minutes. Next update in 15 minutes."

Bad update:
"We're looking into it."
```

---

## 11.7 Postmortem Template

```markdown
## Incident: [Title]
**Date**: [Date]
**Duration**: [X hours Y minutes]
**Severity**: [P1/P2/P3/P4]
**Author**: [Name]

## Summary
[2-3 sentence summary of what happened]

## Impact
- [Number] of users affected
- [Service/feature] unavailable for [duration]
- [Revenue/business impact if applicable]

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 10:00 | First alert fired |
| 10:05 | Engineer acknowledged |
| 10:15 | Root cause identified |
| 10:30 | Fix deployed |
| 10:45 | Service restored |

## Root Cause
[Detailed explanation of what caused the issue]

## Resolution
[What was done to fix it]

## Lessons Learned
### What went well
- [Item]

### What went poorly
- [Item]

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| [Action] | [Name] | [Date] |
```

---

## 11.8 Troubleshooting Communication

### Explaining Technical Issues to Non-Technical Stakeholders

```
Before: "The database connection pool was exhausted due to
         long-running transactions causing lock contention."

After:  "Think of our database like a restaurant with 100 tables.
         Some customers stayed too long (slow queries), so other
         customers couldn't get seats (new requests failed).
         We increased the tables (connection pool) and are
         moving slow customers to a separate area (query optimization)."
```

### Customer Communication Best Practices

```
1. Acknowledge the impact
   "I understand this is affecting your production system."

2. Be clear about status
   "We've identified the issue and are working on a fix."

3. Set expectations
   "We expect to have this resolved within 2 hours."

4. Provide next steps
   "I'll update you in 30 minutes or sooner if we have news."

5. Follow up
   "The issue is resolved. Here's what happened and how
    we're preventing it in the future."
```

---

## Chapter 11 Review Questions

1. A customer reports "the app is slow." Walk through your troubleshooting process.

2. You're getting 502 errors on a load balancer. What do you check?

3. A GKE pod keeps restarting. How do you diagnose?

4. How do you communicate during an incident to non-technical stakeholders?

5. What information should a good postmortem include?

---

## Key Takeaways

1. **Be systematic** - Don't skip steps or make assumptions
2. **Gather data first** - Don't guess, measure
3. **One change at a time** - Otherwise you don't know what fixed it
4. **Document everything** - For postmortem and future reference
5. **Communicate clearly** - Set expectations, update regularly
6. **Blameless postmortems** - Focus on systems, not people

---

[Next Chapter: Distributed Systems Concepts →](./12-distributed-systems.md)
