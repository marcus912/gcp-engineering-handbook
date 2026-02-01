# Chapter 1: GCP Core Concepts

## 1.1 Google Cloud Platform Overview

Google Cloud Platform (GCP) is Google's suite of cloud computing services, running on the same infrastructure that Google uses internally for its end-user products like Google Search, Gmail, and YouTube.

### Why GCP Matters for This Role

As a Customer Solutions Engineer, you'll help customers:
- Migrate to GCP from on-premises or other clouds
- Troubleshoot production issues on GCP
- Design and optimize GCP architectures
- Understand GCP-specific features and best practices

### GCP vs AWS: Mental Model for You

Since you have AWS experience, here's how to think about GCP:

| Concept | AWS | GCP | Key Difference |
|---------|-----|-----|----------------|
| Organization | AWS Organizations | GCP Organization | GCP has stronger hierarchical IAM |
| Account | AWS Account | GCP Project | Projects are more lightweight |
| Region | Region | Region | Similar concept |
| Availability Zone | AZ | Zone | Similar concept |
| CLI | aws cli | gcloud cli | gcloud is more interactive |
| Console | AWS Console | Cloud Console | GCP console is generally cleaner |

---

## 1.2 GCP Resource Hierarchy

Understanding the resource hierarchy is **critical** for troubleshooting IAM and resource management issues.

```
Organization (example: company.com)
    │
    ├── Folder (example: "Engineering")
    │   │
    │   ├── Folder (example: "Production")
    │   │   ├── Project (prod-app-1)
    │   │   └── Project (prod-app-2)
    │   │
    │   └── Folder (example: "Development")
    │       ├── Project (dev-app-1)
    │       └── Project (dev-app-2)
    │
    └── Folder (example: "Finance")
        └── Project (finance-reporting)
```

### Key Concepts

**Organization**
- Root node of the hierarchy
- Tied to a Google Workspace or Cloud Identity domain
- Organization Admin has complete control
- Enables organization-wide policies

**Folders**
- Grouping mechanism for projects
- Can nest up to 10 levels deep
- Useful for department or environment separation
- Policies can be applied at folder level

**Projects**
- Core organizational unit in GCP
- All resources belong to exactly one project
- Has a unique project ID (globally unique, immutable)
- Has a project name (can be changed)
- Has a project number (assigned by Google)
- Billing is tracked at project level

### Why This Matters for Troubleshooting

When a customer reports an issue:
1. **First question**: "Which project is this in?"
2. IAM permissions inherit down the hierarchy
3. Organization policies can block actions at any level
4. Quotas are often per-project

---

## 1.3 Identity and Access Management (IAM)

IAM is one of the most common sources of customer issues. Master this thoroughly.

### The IAM Model

GCP IAM answers: **"Who can do what on which resource?"**

```
WHO (Identity) + WHAT (Role) + WHERE (Resource) = IAM Policy
```

### Identities (WHO)

**Google Account**
- Individual user (person@gmail.com)
- Represents a single human

**Service Account**
- Non-human identity for applications
- Format: name@project-id.iam.gserviceaccount.com
- Has keys (JSON or P12) for authentication
- Can be attached to resources (VMs, Cloud Functions)

**Google Group**
- Collection of identities
- Managed in Google Workspace or Cloud Identity
- Recommended for managing access at scale

**Google Workspace Domain**
- All users in a domain (everyone@company.com)

**Cloud Identity Domain**
- Similar to Workspace but without productivity apps

**allAuthenticatedUsers**
- Any authenticated Google account
- Use with extreme caution

**allUsers**
- Anyone on the internet
- Use with extreme caution (public access)

### Roles (WHAT)

**Primitive Roles** (Legacy - avoid in production)
- `roles/owner` - Full control + manage IAM
- `roles/editor` - Read/write most resources
- `roles/viewer` - Read-only access

**Predefined Roles** (Recommended)
- Created by Google for specific services
- Follow principle of least privilege
- Examples:
  - `roles/compute.instanceAdmin` - Manage VMs
  - `roles/storage.objectViewer` - Read GCS objects
  - `roles/cloudsql.client` - Connect to Cloud SQL

**Custom Roles**
- Created by you for specific needs
- Combine specific permissions
- Can be created at org or project level

### Permissions

The atomic unit of access. Format: `service.resource.verb`

Examples:
- `compute.instances.start` - Start a VM
- `storage.objects.get` - Read a GCS object
- `bigquery.tables.create` - Create a BigQuery table

### IAM Policy

A policy is a collection of bindings attached to a resource:

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@company.com",
        "serviceAccount:my-app@project.iam.gserviceaccount.com"
      ]
    },
    {
      "role": "roles/storage.objectAdmin",
      "members": [
        "group:storage-admins@company.com"
      ]
    }
  ]
}
```

### Policy Inheritance

Policies are additive and inherit downward:

```
Organization Policy: Group A has Viewer on entire org
        ↓
Folder Policy: Group B has Editor on this folder
        ↓
Project Policy: User C has Admin on this project
        ↓
Resource Policy: Service Account D has specific access
```

**Key principle**: You cannot remove inherited permissions at a lower level. You can only ADD permissions.

### Common IAM Troubleshooting

**Problem**: User can't access a resource
**Debug steps**:
1. Check if user has the required permission
2. Check at which level the permission is granted
3. Check for Deny policies (new feature)
4. Check organization policy constraints
5. Check service account permissions if app is making the call

```bash
# Check what permissions a user has on a resource
gcloud asset analyze-iam-policy \
  --organization=ORG_ID \
  --identity="user:alice@company.com" \
  --full-resource-name="//storage.googleapis.com/projects/_/buckets/my-bucket"

# Check who has access to a resource
gcloud asset analyze-iam-policy \
  --organization=ORG_ID \
  --full-resource-name="//compute.googleapis.com/projects/my-project/zones/us-central1-a/instances/my-instance"
```

---

## 1.4 Service Accounts Deep Dive

Service accounts are critical for application authentication and a common source of issues.

### Types of Service Accounts

**User-Managed Service Accounts**
- Created by you
- You control the keys and permissions
- Format: `name@project-id.iam.gserviceaccount.com`

**Default Service Accounts**
- Auto-created when you enable certain APIs
- Compute Engine default: `PROJECT_NUMBER-compute@developer.gserviceaccount.com`
- App Engine default: `project-id@appspot.gserviceaccount.com`
- **Warning**: Default SAs often have broad permissions

**Google-Managed Service Accounts**
- Created and managed by Google
- Used by GCP services internally
- Format: `service-PROJECT_NUMBER@*.iam.gserviceaccount.com`

### Service Account Keys

**Types**:
- Google-managed keys: Rotated automatically by Google
- User-managed keys: JSON or P12 files you download

**Best Practices**:
1. Avoid user-managed keys when possible
2. Use Workload Identity for GKE
3. Use attached service accounts for Compute Engine
4. If you must use keys, rotate them regularly
5. Never commit keys to source control

### Service Account Impersonation

A powerful feature for secure access patterns:

```bash
# Impersonate a service account
gcloud config set auth/impersonate_service_account \
  my-sa@project.iam.gserviceaccount.com

# Or per-command
gcloud compute instances list \
  --impersonate-service-account=my-sa@project.iam.gserviceaccount.com
```

For this to work, your user needs `roles/iam.serviceAccountTokenCreator` on the service account.

### Workload Identity (GKE)

Maps Kubernetes service accounts to GCP service accounts:

```yaml
# Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-k8s-sa
  annotations:
    iam.gke.io/gcp-service-account: my-gcp-sa@project.iam.gserviceaccount.com
```

This allows pods to access GCP APIs without key files.

---

## 1.5 Regions and Zones

### Geographic Concepts

**Region**: A specific geographic location (e.g., `us-central1`, `asia-southeast1`)
- Independent geographic area
- Contains multiple zones
- Resources in different regions have higher latency

**Zone**: Deployment area within a region (e.g., `us-central1-a`, `us-central1-b`)
- Single failure domain
- Independent power, cooling, networking
- Low latency within a zone

**Multi-Region**: A geographic area containing multiple regions
- Used for data storage (GCS, BigQuery)
- Examples: `US`, `EU`, `ASIA`

### Singapore Region

Since you're targeting Singapore:

**Region**: `asia-southeast1` (Singapore)
**Zones**:
- `asia-southeast1-a`
- `asia-southeast1-b`
- `asia-southeast1-c`

**Nearby regions**:
- `asia-southeast2` (Jakarta)
- `asia-east1` (Taiwan)
- `asia-east2` (Hong Kong)
- `asia-northeast1` (Tokyo)

### Resource Scope

| Scope | Description | Examples |
|-------|-------------|----------|
| Zonal | Exists in single zone | VM instances, persistent disks |
| Regional | Replicated across zones | Regional managed instance groups, regional disks |
| Multi-regional | Replicated across regions | GCS multi-region buckets |
| Global | Not tied to location | VPC networks, firewall rules, images |

---

## 1.6 Billing and Quotas

### Billing Structure

```
Billing Account
    │
    ├── Project A (charges go here)
    ├── Project B (charges go here)
    └── Project C (charges go here)
```

### Common Billing Concepts

**SKU (Stock Keeping Unit)**: Individual billable item
- Example: "N1 Standard VM running in us-central1"

**Committed Use Discounts (CUDs)**:
- 1-year or 3-year commitments
- Up to 57% discount on compute
- Resource-based (specific machine type) or spend-based

**Sustained Use Discounts (SUDs)**:
- Automatic discounts for running resources
- Up to 30% off for resources running all month

**Preemptible VMs / Spot VMs**:
- Up to 91% discount
- Can be terminated with 30-second notice
- Good for batch processing, fault-tolerant workloads

### Quotas

Quotas limit resource usage and protect against runaway costs.

**Types**:
- Rate quotas: Limit API calls per time period
- Allocation quotas: Limit number of resources

**Common quota issues**:
```bash
# Check quotas
gcloud compute project-info describe --project=PROJECT_ID

# Request quota increase
# Done through Cloud Console → IAM & Admin → Quotas
```

**Troubleshooting quota errors**:
1. Identify which quota is exceeded
2. Check current usage vs limit
3. Request increase or optimize usage
4. Consider regional distribution

---

## 1.7 Cloud SDK and CLI

### gcloud CLI

The primary tool for interacting with GCP:

```bash
# Authentication
gcloud auth login                    # User authentication
gcloud auth application-default login  # Application default credentials
gcloud auth activate-service-account   # Service account authentication

# Configuration
gcloud config set project PROJECT_ID
gcloud config set compute/zone us-central1-a
gcloud config set compute/region us-central1

# Configuration profiles
gcloud config configurations create my-config
gcloud config configurations activate my-config
gcloud config configurations list

# Common commands
gcloud compute instances list
gcloud container clusters list
gcloud sql instances list
gcloud services list --enabled
```

### gsutil (Cloud Storage)

```bash
# List buckets
gsutil ls

# Copy files
gsutil cp local-file.txt gs://bucket/
gsutil cp gs://bucket/file.txt ./

# Sync directories
gsutil -m rsync -r local-dir gs://bucket/dir

# Set permissions
gsutil iam ch user:alice@example.com:objectViewer gs://bucket
```

### bq (BigQuery)

```bash
# Run query
bq query --use_legacy_sql=false 'SELECT * FROM dataset.table LIMIT 10'

# List datasets
bq ls

# Create dataset
bq mk --dataset project:dataset_name

# Load data
bq load --source_format=CSV dataset.table gs://bucket/data.csv
```

---

## 1.8 Cloud Console and APIs

### Cloud Console

Web-based interface at console.cloud.google.com

**Key sections to know**:
- **Navigation menu**: Access all services
- **Project selector**: Switch between projects
- **Cloud Shell**: Browser-based terminal
- **Activity**: Recent API calls and changes
- **Notifications**: Alerts and updates

### APIs

Every GCP service has a REST API:

**API enablement**:
```bash
# Enable an API
gcloud services enable compute.googleapis.com

# List enabled APIs
gcloud services list --enabled
```

**API Explorer**: Try APIs directly in browser
- https://developers.google.com/apis-explorer

**Client Libraries**: Official SDKs for major languages
- Python, Java, Go, Node.js, C#, Ruby, PHP

---

## 1.9 Logging and Monitoring Overview

### Cloud Logging (formerly Stackdriver Logging)

Centralized log management:

```bash
# View logs
gcloud logging read "resource.type=gce_instance" --limit=10

# Write a log entry
gcloud logging write my-log "Test message"
```

**Log types**:
- Platform logs: Generated by GCP services
- User-written logs: Your application logs
- Security logs: Audit and access logs

### Cloud Monitoring (formerly Stackdriver Monitoring)

Metrics, dashboards, and alerting:

**Key concepts**:
- **Metrics**: Time-series data (CPU, memory, custom)
- **Dashboards**: Visualize metrics
- **Alerting policies**: Get notified on conditions
- **Uptime checks**: Monitor endpoint availability

---

## Chapter 1 Review Questions

1. What is the difference between a project ID and project number?

2. A user has `roles/viewer` at the organization level and `roles/editor` at the project level. What effective permissions do they have on project resources?

3. Why should you avoid using default service accounts in production?

4. A customer's application is getting "Permission denied" errors. What's your troubleshooting approach?

5. What's the difference between a zonal and regional resource? Give examples.

6. How does Workload Identity improve security compared to service account keys?

---

## Chapter 1 Hands-On Exercises

### Exercise 1.1: Project Setup
1. Create a new GCP project
2. Enable the Compute Engine API
3. Set the project as your default in gcloud

### Exercise 1.2: IAM Exploration
1. View the IAM policy on your project
2. Create a custom role with compute.instances.list and compute.instances.get
3. Create a service account and assign the custom role

### Exercise 1.3: Resource Hierarchy
1. Use the Asset Inventory to list all resources in your project
2. Analyze the IAM policy for a specific resource

```bash
# List all resources
gcloud asset list --project=PROJECT_ID

# Search for resources
gcloud asset search-all-resources --scope=projects/PROJECT_ID
```

---

## Key Takeaways

1. **GCP is project-centric** - Everything lives in a project
2. **IAM is hierarchical** - Permissions inherit downward
3. **Service accounts are identities** - Treat them with the same security as user accounts
4. **Quotas matter** - Common source of customer issues
5. **Know your tools** - gcloud, gsutil, bq are essential

---

[Next Chapter: Compute Services →](./02-gcp-compute.md)
