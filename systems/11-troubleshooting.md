# Chapter 11: Troubleshooting Methodology

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

## 11.8 Escalation Management

### Escalation Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                    Escalation Pyramid                            │
│                                                                  │
│                         ┌─────┐                                  │
│                         │ Exec│  VP/Director level              │
│                         │ Esc │  Business impact decisions      │
│                       ┌─┴─────┴─┐                               │
│                       │   P1    │  Cross-functional war room    │
│                       │ Bridge  │  Multiple team leads          │
│                     ┌─┴─────────┴─┐                             │
│                     │  Technical  │  Subject matter experts     │
│                     │  Escalation │  Specialist teams           │
│                   ┌─┴─────────────┴─┐                           │
│                   │   First Response │  On-call engineer        │
│                   │   (L1/L2)        │  Initial triage          │
│                 ┌─┴─────────────────┴─┐                         │
│                 │    Customer Report   │  Ticket/Alert          │
│                 └─────────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
```

### When to Escalate

```
ESCALATE IMMEDIATELY when:
□ Customer-facing service is down
□ Data loss or corruption suspected
□ Security breach detected
□ SLA breach imminent or occurred
□ Issue affects multiple major customers
□ You've been stuck for > 30 minutes on P1

ESCALATE PROACTIVELY when:
□ Issue requires expertise outside your domain
□ Fix requires access you don't have
□ Customer is executive-level or strategic account
□ Issue is spreading or worsening
□ You need coordination across teams
```

### Escalation Communication Template

```markdown
## Escalation Request

**Severity**: P1 - Production Down
**Customer**: [Customer Name] - [Account Type: Strategic/Enterprise/Standard]
**Duration**: Issue ongoing for [X] hours
**Impact**: [Number] users affected, [Revenue/SLA impact]

**Summary**:
[2-3 sentences describing the issue]

**What We've Tried**:
1. [Action 1] - Result: [outcome]
2. [Action 2] - Result: [outcome]

**What We Need**:
- [ ] Access to [system/logs]
- [ ] Expertise from [team: Database/Networking/Security]
- [ ] Decision on [rollback/customer communication/etc.]

**Current Theory**:
[Best hypothesis on root cause]

**Next Update**: [Time] or sooner if status changes
```

### War Room Coordination

**Setting Up a War Room**:
```
1. ESTABLISH COMMAND
   - Designate Incident Commander (IC)
   - IC does NOT debug - coordinates only
   - IC makes escalation decisions

2. DEFINE ROLES
   - IC: Coordination, communication, decisions
   - Tech Lead: Technical direction
   - Scribe: Document everything
   - Comms Lead: Customer/stakeholder updates

3. SET RHYTHM
   - Status updates every 15 minutes
   - Round-robin: each team reports status
   - IC summarizes and assigns next actions

4. COMMUNICATION CHANNELS
   - Primary: Video call (for real-time)
   - Secondary: Slack channel (for async, links)
   - Customer: Separate thread, filtered updates
```

**War Room Checklist**:
```
□ Incident Commander assigned and announced
□ All relevant teams on the call
□ Shared document for notes (Google Doc link shared)
□ Customer communication owner assigned
□ Timeline being maintained
□ Regular status cadence established (every 15 min)
□ Escalation path clear (who to call if needed)
□ Clear "we're done" criteria defined
```

### Cross-Team Coordination

**Working Across Teams Effectively**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Multi-Team Incident                           │
│                                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                 │
│   │ Platform │    │ Database │    │ Network  │                 │
│   │   Team   │    │   Team   │    │   Team   │                 │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘                 │
│        │               │               │                        │
│        └───────────────┼───────────────┘                        │
│                        │                                        │
│                        ▼                                        │
│              ┌─────────────────┐                                │
│              │    Incident     │                                │
│              │   Commander     │                                │
│              │   (You/CSE)     │                                │
│              └────────┬────────┘                                │
│                       │                                        │
│                       ▼                                        │
│              ┌─────────────────┐                                │
│              │    Customer     │                                │
│              └─────────────────┘                                │
└─────────────────────────────────────────────────────────────────┘

Your role as incident coordinator:
1. Represent the customer's urgency
2. Translate between teams (DB team ↔ Network team)
3. Keep everyone focused on resolution, not blame
4. Ensure customer is updated
5. Drive momentum - don't let investigation stall
```

**Influence Without Authority**:
```
DO:
✓ Lead with data: "Logs show X, which suggests..."
✓ Create urgency with impact: "Customer losing $X/hour"
✓ Propose next steps: "Can we try X while you investigate Y?"
✓ Acknowledge expertise: "You know this system best - what's your read?"
✓ Remove blockers: "What do you need to move faster?"

DON'T:
✗ Demand or pull rank
✗ Blame other teams
✗ Make assumptions about other team's work
✗ Skip the expert's assessment
✗ Let silence persist - prompt for updates
```

---

## 11.9 On-Call Best Practices

### On-Call Responsibilities

```
BEFORE your shift:
□ Review recent incidents and ongoing issues
□ Check your alerting setup works (PagerDuty, phone)
□ Know the escalation path
□ Have laptop and connectivity ready
□ Review runbooks for common issues

DURING your shift:
□ Acknowledge alerts within SLA (typically 5-15 min)
□ Triage before diving deep
□ Document as you go
□ Escalate when stuck (don't hero)
□ Hand off cleanly at shift end

AFTER your shift:
□ Complete handoff document
□ File bugs for any gaps found
□ Update runbooks if needed
□ Participate in incident reviews
```

### Alert Response Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     Alert Response Flow                          │
│                                                                  │
│  Alert Fires                                                     │
│      │                                                           │
│      ▼                                                           │
│  ┌───────────────┐                                              │
│  │ Acknowledge   │ ◄── Within 5 minutes                         │
│  │ (stop paging) │                                              │
│  └───────┬───────┘                                              │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐                                              │
│  │ Quick Triage  │ ◄── Is it real? What's the impact?          │
│  │ (5 minutes)   │                                              │
│  └───────┬───────┘                                              │
│          │                                                       │
│    ┌─────┴─────┐                                                │
│    ▼           ▼                                                │
│ ┌──────┐  ┌────────┐                                           │
│ │False │  │Real    │                                           │
│ │Positive│ │Issue   │                                           │
│ └──┬───┘  └───┬────┘                                           │
│    │          │                                                 │
│    ▼          ▼                                                 │
│ Resolve &   Mitigate ──► Investigate ──► Resolve ──► Postmortem│
│ Fix alert   (restore     (root cause)   (permanent              │
│             service)                      fix)                  │
└─────────────────────────────────────────────────────────────────┘
```

### Effective Handoffs

```markdown
## On-Call Handoff

**Date**: [Date]
**From**: [Your Name]
**To**: [Next On-Call]

### Active Issues
| Issue | Status | Next Steps | ETA |
|-------|--------|------------|-----|
| [Issue 1] | Monitoring | Check at 2pm | - |
| [Issue 2] | Escalated to DB team | Waiting for response | ~2hrs |

### Resolved This Shift
- [Issue]: [Brief summary and resolution]

### Heads Up
- Deployment scheduled for [time]
- Customer [X] may follow up on ticket #123

### Runbook Updates Needed
- [Gap identified during incident]
```

---

## 11.10 SLOs, SLIs, and SLAs

### Definitions

```
SLI (Service Level Indicator):
  A metric that measures service performance
  Example: "Request latency" or "Error rate"

SLO (Service Level Objective):
  A target value for an SLI
  Example: "99.9% of requests complete in < 200ms"

SLA (Service Level Agreement):
  A contract with consequences for missing SLO
  Example: "If uptime < 99.9%, customer gets credit"

Relationship:
  SLI ──measures──► Service
  SLO ──targets──► SLI
  SLA ──enforces──► SLO with consequences
```

### Common SLIs

```
Availability:
  successful requests / total requests
  Target: 99.9% (three nines) = 8.76 hours downtime/year

Latency:
  Percentage of requests faster than threshold
  Target: 99% of requests < 200ms

Error Rate:
  failed requests / total requests
  Target: < 0.1% error rate

Throughput:
  Requests per second the system handles
  Target: Handle 10,000 RPS
```

### Error Budgets

```
If SLO = 99.9% availability:
  Error Budget = 0.1% = 43.2 minutes/month

┌─────────────────────────────────────────────────────────────────┐
│                    Error Budget Status                           │
│                                                                  │
│  Month: January                                                  │
│  SLO: 99.9% availability                                        │
│  Budget: 43.2 minutes                                           │
│                                                                  │
│  ████████████████████░░░░░  Used: 32 min (74%)                 │
│                                                                  │
│  Remaining: 11.2 minutes                                        │
│                                                                  │
│  Status: CAUTION - Slow down risky changes                      │
└─────────────────────────────────────────────────────────────────┘

When budget is:
  > 50% remaining: Normal operations, deploy freely
  25-50% remaining: Caution, review changes carefully
  < 25% remaining: Freeze non-critical changes
  Exhausted: Focus on reliability, no new features
```

### During Incidents: SLA Awareness

```
Questions to ask:
1. Is this customer on an SLA?
2. What's their SLO and current status?
3. Are we at risk of breaching?
4. What's the business impact of breach?

If SLA breach is imminent:
- Escalate immediately
- Notify account team
- Document timeline carefully
- Focus on mitigation over root cause
- Prepare customer communication
```

---

## 11.11 Troubleshooting Communication

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

### Basic Troubleshooting
1. A customer reports "the app is slow." Walk through your troubleshooting process.

2. You're getting 502 errors on a load balancer. What do you check?

3. A GKE pod keeps restarting. How do you diagnose?

4. How do you communicate during an incident to non-technical stakeholders?

5. What information should a good postmortem include?

### Escalation & On-Call
6. When should you escalate vs continue debugging yourself?

7. You're coordinating a P1 incident with 3 different teams. How do you keep momentum?

8. Describe the role of an Incident Commander in a war room.

9. A customer's SLA is about to breach. What actions do you take?

10. What makes an effective on-call handoff?

11. Explain error budgets and how they influence operational decisions.

---

## Chapter 11 Hands-On Exercises

### Exercise 11.1: Incident Simulation
1. Set up a sample application on GKE
2. Introduce a failure (resource exhaustion, bad config)
3. Practice the troubleshooting flow from alert to resolution

### Exercise 11.2: Escalation Role-Play
1. Write an escalation request for a P1 database issue
2. Practice explaining to a non-technical stakeholder
3. Draft customer communication updates

### Exercise 11.3: Postmortem Writing
1. Take a past incident (real or simulated)
2. Write a complete postmortem using the template
3. Identify actionable follow-ups

---

## Key Takeaways

### Troubleshooting Fundamentals
1. **Be systematic** - Don't skip steps or make assumptions
2. **Gather data first** - Don't guess, measure
3. **One change at a time** - Otherwise you don't know what fixed it
4. **Document everything** - For postmortem and future reference
5. **Communicate clearly** - Set expectations, update regularly
6. **Blameless postmortems** - Focus on systems, not people

### Escalation & On-Call
7. **Escalate early** - 30 minutes stuck on P1 = escalate
8. **Incident Commander coordinates** - doesn't debug
9. **Influence with data** - not authority
10. **SLA awareness** - know when breach is imminent
11. **Clean handoffs** - document ongoing issues
12. **Error budgets drive decisions** - reliability vs velocity

---

[Next Chapter: Distributed Systems Concepts →](../distributed-systems/12-distributed-systems.md)
