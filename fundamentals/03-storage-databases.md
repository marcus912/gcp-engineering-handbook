# Chapter 3: Storage & Databases on GCP

## 3.1 Storage Options Overview

GCP offers storage solutions for every use case:

```
┌─────────────────────────────────────────────────────────────────┐
│                      GCP Storage Options                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Object Storage          Block Storage         File Storage      │
│  ┌──────────────┐       ┌──────────────┐      ┌──────────────┐ │
│  │Cloud Storage │       │Persistent Disk│      │  Filestore   │ │
│  │   (GCS)      │       │  Local SSD    │      │              │ │
│  └──────────────┘       └──────────────┘      └──────────────┘ │
│                                                                  │
│  Databases                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Relational        │ NoSQL              │ Data Warehouse   │  │
│  │ ┌──────────────┐ │ ┌────────────────┐ │ ┌─────────────┐  │  │
│  │ │  Cloud SQL   │ │ │   Firestore    │ │ │  BigQuery   │  │  │
│  │ │ Cloud Spanner│ │ │   Bigtable     │ │ │             │  │  │
│  │ │   AlloyDB    │ │ │  Memorystore   │ │ └─────────────┘  │  │
│  │ └──────────────┘ │ └────────────────┘ │                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Cloud Storage (GCS)

Object storage service, equivalent to AWS S3.

### Storage Classes

| Class | Use Case | Availability | Min Duration | Price (US) |
|-------|----------|--------------|--------------|------------|
| **Standard** | Frequently accessed data | 99.99% | None | $0.020/GB |
| **Nearline** | Once per month | 99.9% | 30 days | $0.010/GB |
| **Coldline** | Once per quarter | 99.9% | 90 days | $0.004/GB |
| **Archive** | Once per year | 99.9% | 365 days | $0.0012/GB |

**Key insight**: All classes have same latency and throughput. Difference is retrieval cost and minimum storage duration.

### Bucket Configuration

```bash
# Create a bucket
gsutil mb -l us-central1 -c standard gs://my-bucket-name

# Create with specific settings
gsutil mb \
  -p PROJECT_ID \
  -c nearline \
  -l us-central1 \
  --pap enforced \
  gs://my-bucket-name
```

### Location Types

| Type | Example | Use Case |
|------|---------|----------|
| **Region** | us-central1 | Data locality, lowest latency |
| **Dual-region** | nam4 (Iowa + S. Carolina) | High availability, specific regions |
| **Multi-region** | US, EU, ASIA | Global access, highest availability |

### Access Control

**Uniform bucket-level access** (recommended):
- IAM-only access control
- No object-level ACLs
- Simpler, more secure

```bash
# Enable uniform bucket-level access
gsutil uniformbucketlevelaccess set on gs://my-bucket

# Grant access via IAM
gsutil iam ch user:alice@example.com:objectViewer gs://my-bucket
gsutil iam ch serviceAccount:sa@project.iam.gserviceaccount.com:objectAdmin gs://my-bucket
```

**Fine-grained (legacy)**:
- ACLs at object level
- More complex to manage

### Signed URLs

Grant temporary access without IAM changes:

```python
from google.cloud import storage
import datetime

def generate_signed_url(bucket_name, blob_name):
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)

    url = blob.generate_signed_url(
        version="v4",
        expiration=datetime.timedelta(hours=1),
        method="GET",
    )
    return url
```

```bash
# Using gsutil
gsutil signurl -d 1h key.json gs://bucket/object
```

### Object Lifecycle Management

Automatically transition or delete objects:

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
```

```bash
gsutil lifecycle set lifecycle.json gs://my-bucket
```

### Object Versioning

Keep previous versions of objects:

```bash
# Enable versioning
gsutil versioning set on gs://my-bucket

# List all versions
gsutil ls -a gs://my-bucket

# Restore a version
gsutil cp gs://my-bucket/file#VERSION gs://my-bucket/file
```

### Retention Policies and Holds

Prevent object deletion:

```bash
# Set retention policy (minimum time before deletion)
gsutil retention set 1y gs://my-bucket

# Lock retention policy (cannot be reduced)
gsutil retention lock gs://my-bucket

# Set hold on specific object
gsutil retention temp set gs://my-bucket/object
```

### Transfer Service

For large-scale data migration:

```bash
# Create a transfer job from S3
gcloud transfer jobs create \
  s3://source-bucket \
  gs://destination-bucket \
  --source-creds-file=aws-creds.json
```

---

## 3.3 Persistent Disk

Block storage for VMs.

### Disk Types

| Type | IOPS (Read) | Throughput | Use Case |
|------|-------------|------------|----------|
| **pd-standard** | 0.75/GB | 0.12 MB/s/GB | Boot disks, cold data |
| **pd-balanced** | 6/GB | 0.28 MB/s/GB | Most workloads |
| **pd-ssd** | 30/GB | 0.48 MB/s/GB | High IOPS needs |
| **pd-extreme** | Configurable | Configurable | Highest performance |

### Regional vs Zonal

**Zonal disks**:
- Single zone
- Snapshots for backup

**Regional disks**:
- Synchronous replication across two zones
- Higher availability
- 2x cost

```bash
# Create regional disk
gcloud compute disks create my-regional-disk \
  --size=500GB \
  --type=pd-ssd \
  --region=us-central1 \
  --replica-zones=us-central1-a,us-central1-b
```

### Snapshots

Point-in-time backups:

```bash
# Create snapshot
gcloud compute disks snapshot my-disk \
  --snapshot-names=my-snapshot \
  --zone=us-central1-a

# Create snapshot schedule
gcloud compute resource-policies create snapshot-schedule my-schedule \
  --max-retention-days=14 \
  --on-source-disk-delete=keep-auto-snapshots \
  --daily-schedule \
  --start-time=04:00 \
  --region=us-central1
```

### Disk Resize

```bash
# Resize disk (only increase, never decrease)
gcloud compute disks resize my-disk --size=500GB --zone=us-central1-a

# Then resize filesystem inside VM
# For ext4:
sudo resize2fs /dev/sdb
# For xfs:
sudo xfs_growfs /mount/point
```

---

## 3.4 Filestore

Managed NFS file shares.

### Tiers

| Tier | Capacity | Use Case |
|------|----------|----------|
| Basic HDD | 1-63.9 TB | Dev/test, batch workloads |
| Basic SSD | 2.5-63.9 TB | General file shares |
| High Scale SSD | 10-100 TB | High-performance computing |
| Enterprise | 1-10 TB | Mission-critical workloads |

```bash
# Create Filestore instance
gcloud filestore instances create my-filestore \
  --zone=us-central1-a \
  --tier=BASIC_SSD \
  --file-share=name=vol1,capacity=2.5TB \
  --network=name=default
```

### Mounting in VMs

```bash
# Install NFS client
sudo apt-get install nfs-common

# Create mount point
sudo mkdir /mnt/filestore

# Mount
sudo mount 10.0.0.2:/vol1 /mnt/filestore

# Add to /etc/fstab for persistence
10.0.0.2:/vol1 /mnt/filestore nfs defaults 0 0
```

---

## 3.5 Cloud SQL

Managed MySQL, PostgreSQL, and SQL Server.

### Creating an Instance

```bash
# Create MySQL instance
gcloud sql instances create my-mysql \
  --database-version=MYSQL_8_0 \
  --cpu=2 \
  --memory=7680MB \
  --region=us-central1 \
  --root-password=PASSWORD

# Create PostgreSQL instance
gcloud sql instances create my-postgres \
  --database-version=POSTGRES_14 \
  --cpu=2 \
  --memory=7680MB \
  --region=us-central1
```

### High Availability

```bash
# Create HA instance
gcloud sql instances create my-ha-instance \
  --database-version=POSTGRES_14 \
  --cpu=2 \
  --memory=7680MB \
  --region=us-central1 \
  --availability-type=REGIONAL  # This enables HA
```

**How HA works**:
- Primary and standby in different zones
- Synchronous replication
- Automatic failover (usually 60-120 seconds)
- Same IP address after failover

### Read Replicas

```bash
# Create read replica
gcloud sql instances create my-replica \
  --master-instance-name=my-mysql \
  --region=us-central1
```

### Connecting to Cloud SQL

**1. Public IP with authorized networks**:
```bash
# Authorize your IP
gcloud sql instances patch my-instance \
  --authorized-networks=YOUR_IP/32
```

**2. Cloud SQL Auth Proxy** (recommended):
```bash
# Download proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.0.0/cloud-sql-proxy.linux.amd64

# Connect
./cloud-sql-proxy --port=3306 PROJECT:REGION:INSTANCE

# Then connect via localhost
mysql -u root -p -h 127.0.0.1
```

**3. Private IP**:
```bash
# Enable private IP
gcloud sql instances patch my-instance \
  --network=projects/PROJECT/global/networks/VPC_NAME \
  --no-assign-ip  # Remove public IP
```

### Backups

```bash
# Configure automated backups
gcloud sql instances patch my-instance \
  --backup-start-time=04:00 \
  --enable-bin-log  # For MySQL point-in-time recovery

# Create on-demand backup
gcloud sql instances backups create my-instance

# List backups
gcloud sql backups list --instance=my-instance
```

### Common Issues

**Can't connect**:
1. Check authorized networks (public IP)
2. Check VPC peering (private IP)
3. Check IAM permissions (Cloud SQL Client role)
4. Check firewall rules
5. Check SSL requirement settings

**Slow queries**:
1. Check Cloud SQL Insights (query analysis)
2. Check CPU/memory usage
3. Check slow query log
4. Consider read replicas for read-heavy workloads

---

## 3.6 Cloud Spanner

Globally distributed, horizontally scalable relational database.

### Key Features

- **Global scale**: Distribute data across regions
- **Strong consistency**: ACID transactions globally
- **99.999% SLA**: When multi-regional
- **Automatic sharding**: Data distributed automatically

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Spanner Instance                    │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                     Database                             ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     ││
│  │  │   Table 1   │  │   Table 2   │  │   Table 3   │     ││
│  │  │  ┌───────┐  │  │  ┌───────┐  │  │  ┌───────┐  │     ││
│  │  │  │Split 1│  │  │  │Split 1│  │  │  │Split 1│  │     ││
│  │  │  ├───────┤  │  │  ├───────┤  │  │  ├───────┤  │     ││
│  │  │  │Split 2│  │  │  │Split 2│  │  │  │Split 2│  │     ││
│  │  │  └───────┘  │  │  └───────┘  │  │  └───────┘  │     ││
│  │  └─────────────┘  └─────────────┘  └─────────────┘     ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Processing Units (compute capacity)                         │
└─────────────────────────────────────────────────────────────┘
```

### Creating an Instance

```bash
# Create regional instance (lower latency, lower cost)
gcloud spanner instances create my-spanner \
  --config=regional-us-central1 \
  --processing-units=100 \
  --description="My Spanner instance"

# Create multi-regional instance (higher availability)
gcloud spanner instances create my-spanner-mr \
  --config=nam3 \  # North America multi-region
  --processing-units=1000
```

### Schema Design

```sql
-- Use interleaved tables for related data
CREATE TABLE Singers (
  SingerId   INT64 NOT NULL,
  FirstName  STRING(1024),
  LastName   STRING(1024),
) PRIMARY KEY (SingerId);

-- Albums interleaved in Singers (stored together)
CREATE TABLE Albums (
  SingerId     INT64 NOT NULL,
  AlbumId      INT64 NOT NULL,
  AlbumTitle   STRING(MAX),
) PRIMARY KEY (SingerId, AlbumId),
  INTERLEAVE IN PARENT Singers ON DELETE CASCADE;
```

### When to Use Spanner

**Use when**:
- Need global scale with strong consistency
- Financial transactions across regions
- Gaming leaderboards at scale
- Supply chain/inventory systems

**Don't use when**:
- Small scale (< 1 TB)
- Cost-sensitive (minimum ~$65/month)
- Don't need global distribution

---

## 3.7 Firestore

NoSQL document database.

### Data Model

```
Collection: users
    │
    ├── Document: user_123
    │   ├── name: "Alice"
    │   ├── email: "alice@example.com"
    │   │
    │   └── Subcollection: orders
    │       ├── Document: order_1
    │       └── Document: order_2
    │
    └── Document: user_456
        └── name: "Bob"
```

### Firestore Modes

**Native Mode**:
- Real-time listeners
- Offline support
- Mobile/web SDK support
- Recommended for new projects

**Datastore Mode**:
- Compatible with legacy Datastore
- Server-side only
- Higher write throughput

### Basic Operations

```python
from google.cloud import firestore

db = firestore.Client()

# Create/Update document
doc_ref = db.collection('users').document('user_123')
doc_ref.set({
    'name': 'Alice',
    'email': 'alice@example.com',
    'created': firestore.SERVER_TIMESTAMP
})

# Read document
doc = doc_ref.get()
if doc.exists:
    print(doc.to_dict())

# Query
users_ref = db.collection('users')
query = users_ref.where('age', '>=', 18).limit(10)
results = query.stream()

# Real-time listener
def on_snapshot(doc_snapshot, changes, read_time):
    for doc in doc_snapshot:
        print(f'Received doc: {doc.id}')

doc_ref.on_snapshot(on_snapshot)
```

### Indexes

**Single-field indexes**: Created automatically
**Composite indexes**: Must be created for queries with multiple conditions

```bash
# Create composite index
gcloud firestore indexes composite create \
  --collection-group=users \
  --field-config=field-path=age,order=ascending \
  --field-config=field-path=created,order=descending
```

---

## 3.8 Bigtable

Wide-column NoSQL database for massive scale.

### Use Cases

- Time-series data (IoT, monitoring)
- Financial data (market data, transactions)
- Graph data
- Machine learning features

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Bigtable Table                        │
│                                                          │
│  Row Key │ Column Family: cf1   │ Column Family: cf2    │
│          │ col1  │ col2  │ col3 │ colA  │ colB         │
│ ─────────┼───────┼───────┼──────┼───────┼─────────────│
│ row1     │ val   │ val   │ val  │ val   │ val          │
│ row2     │ val   │       │ val  │ val   │              │
│ row3     │       │ val   │      │       │ val          │
└─────────────────────────────────────────────────────────┘
```

### Key Concepts

**Row key**: Single indexed value per row
- Design is critical for performance
- Avoid hotspots (don't use timestamps as prefix)

**Column families**: Groups of related columns
- Set at table creation
- Each has TTL and garbage collection settings

**Cells**: Intersection of row and column
- Can have multiple versions (timestamps)

### Schema Design

```python
# Good row key design for time-series
# Reverse timestamp prevents hotspot
row_key = f"{device_id}#{(MAX_TIMESTAMP - timestamp)}"

# Bad: timestamp prefix creates hotspot
row_key = f"{timestamp}#{device_id}"  # DON'T DO THIS
```

### Creating a Cluster

```bash
# Create instance
gcloud bigtable instances create my-bigtable \
  --display-name="My Bigtable" \
  --cluster=my-cluster \
  --cluster-zone=us-central1-a \
  --cluster-num-nodes=3

# Create table
cbt createtable my-table
cbt createfamily my-table cf1
```

---

## 3.9 BigQuery

Serverless data warehouse.

### Key Features

- **Serverless**: No infrastructure to manage
- **Petabyte scale**: Handle massive datasets
- **Standard SQL**: ANSI SQL:2011 compliant
- **Built-in ML**: Train models with SQL (BigQuery ML)

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                       BigQuery                           │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐│
│  │              Project: my-project                     ││
│  │  ┌────────────────────────────────────────────────┐ ││
│  │  │           Dataset: my_dataset                  │ ││
│  │  │  ┌───────────────┐  ┌───────────────┐         │ ││
│  │  │  │ Table: users  │  │ Table: orders │         │ ││
│  │  │  │               │  │               │         │ ││
│  │  │  │  Partitioned  │  │  Clustered    │         │ ││
│  │  │  │  by date      │  │  by user_id   │         │ ││
│  │  │  └───────────────┘  └───────────────┘         │ ││
│  │  └────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Pricing Model

**On-demand**: $5 per TB scanned
**Flat-rate**: Reserved slots (100 slot minimum)

### Optimization Techniques

**Partitioning**: Divide table by date/time

```sql
CREATE TABLE project.dataset.events
PARTITION BY DATE(event_timestamp)
AS SELECT * FROM source_table;

-- Queries filtered by partition are cheaper
SELECT * FROM project.dataset.events
WHERE DATE(event_timestamp) = '2024-01-15';
```

**Clustering**: Sort data within partitions

```sql
CREATE TABLE project.dataset.events
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT * FROM source_table;
```

**Cost control**:
```sql
-- Check query cost before running
SELECT * FROM dataset.table
-- BigQuery will show estimated bytes processed

-- Limit bytes billed
bq query --maximum_bytes_billed=1000000000 "SELECT ..."
```

### BigQuery ML

Train ML models using SQL:

```sql
-- Create a model
CREATE MODEL project.dataset.my_model
OPTIONS(model_type='logistic_reg') AS
SELECT feature1, feature2, label
FROM project.dataset.training_data;

-- Make predictions
SELECT * FROM ML.PREDICT(
  MODEL project.dataset.my_model,
  (SELECT feature1, feature2 FROM new_data)
);
```

---

## 3.10 Memorystore

Managed Redis and Memcached.

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Rich (lists, sets, hashes) | Simple key-value |
| Persistence | Optional | None |
| Replication | Yes | No |
| Use case | Caching, sessions, queues | Simple caching |

### Creating Redis Instance

```bash
gcloud redis instances create my-redis \
  --size=1 \
  --region=us-central1 \
  --redis-version=redis_6_x \
  --tier=standard  # standard has replication
```

### Connecting

```python
import redis

# Get connection info
# gcloud redis instances describe my-redis --region=us-central1

r = redis.Redis(
    host='10.0.0.3',  # Internal IP
    port=6379,
    db=0
)

r.set('key', 'value')
print(r.get('key'))
```

---

## 3.11 Database Selection Guide

```
┌─────────────────────────────────────────────────────────────────┐
│                    Which Database to Use?                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Need SQL?                                                       │
│    YES ──┬── Global scale? ──── YES ──→ Cloud Spanner           │
│          │                                                       │
│          └── No ──┬── Need PostgreSQL compatibility?            │
│                   │     YES ──→ AlloyDB                          │
│                   │     NO ───→ Cloud SQL                        │
│                                                                  │
│    NO ───┬── Real-time updates + offline?                       │
│          │     YES ──→ Firestore (Native Mode)                   │
│          │                                                       │
│          ├── Time-series at massive scale?                       │
│          │     YES ──→ Bigtable                                  │
│          │                                                       │
│          ├── Analytics / Data Warehouse?                         │
│          │     YES ──→ BigQuery                                  │
│          │                                                       │
│          └── Caching?                                            │
│                YES ──→ Memorystore                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Chapter 3 Review Questions

1. When would you choose Nearline vs Archive storage class?

2. A customer's Cloud SQL instance is running out of connections. What are possible solutions?

3. Explain the difference between Cloud SQL and Cloud Spanner.

4. How would you design a Bigtable schema for IoT sensor data?

5. What's the difference between BigQuery partitioning and clustering?

6. A customer needs to migrate 50TB from S3 to GCS. What's the recommended approach?

---

## Chapter 3 Hands-On Exercises

### Exercise 3.1: Cloud Storage
1. Create a bucket with lifecycle rules
2. Upload files and set up object versioning
3. Generate a signed URL

### Exercise 3.2: Cloud SQL
1. Create a PostgreSQL instance with HA
2. Connect using Cloud SQL Auth Proxy
3. Create a read replica

### Exercise 3.3: BigQuery
1. Create a partitioned and clustered table
2. Write a query and check bytes processed
3. Create a simple BigQuery ML model

---

## Key Takeaways

1. **Match storage to access patterns** - Hot vs cold data
2. **Cloud SQL for traditional apps** - MySQL/PostgreSQL compatibility
3. **Spanner for global scale** - When you need strong consistency everywhere
4. **BigQuery for analytics** - Serverless, pay per query
5. **Design Bigtable row keys carefully** - Avoid hotspots
6. **Use lifecycle policies** - Automate storage class transitions

---

[Next Chapter: Networking on GCP →](./04-networking.md)
