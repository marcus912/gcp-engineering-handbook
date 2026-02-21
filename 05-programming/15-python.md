# Chapter 15: Python for Cloud Engineers

Python is essential for automation, scripting, and working with GCP APIs.

## 14.1 Python Essentials Review

### Data Types

```python
# Strings
name = "Alice"
name = 'Alice'
name = """Multi-line
string"""

# Numbers
count = 42          # int
price = 19.99       # float

# Lists
items = [1, 2, 3]
items.append(4)
items[0]            # Access
items[1:3]          # Slice
len(items)          # Length

# Dictionaries
person = {
    "name": "Alice",
    "age": 30
}
person["name"]      # Access
person.get("name")  # Safe access
person.keys()       # All keys
person.values()     # All values

# Sets
unique = {1, 2, 3}
unique.add(4)

# Tuples (immutable)
point = (10, 20)
x, y = point        # Unpacking
```

### Control Flow

```python
# Conditionals
if x > 0:
    print("positive")
elif x < 0:
    print("negative")
else:
    print("zero")

# Loops
for item in items:
    print(item)

for i in range(10):
    print(i)

for i, item in enumerate(items):
    print(f"{i}: {item}")

for key, value in person.items():
    print(f"{key}: {value}")

while condition:
    do_something()

# Comprehensions
squares = [x**2 for x in range(10)]
even = [x for x in range(10) if x % 2 == 0]
word_lengths = {word: len(word) for word in words}
```

### Functions

```python
def greet(name, greeting="Hello"):
    """Return a greeting message."""
    return f"{greeting}, {name}!"

# *args and **kwargs
def log(*args, **kwargs):
    print("Args:", args)
    print("Kwargs:", kwargs)

log(1, 2, 3, name="Alice", age=30)
# Args: (1, 2, 3)
# Kwargs: {'name': 'Alice', 'age': 30}

# Lambda
square = lambda x: x**2

# Type hints
def process(data: list[str]) -> dict[str, int]:
    return {item: len(item) for item in data}
```

---

## 14.2 Error Handling

```python
# Try/except
try:
    result = risky_operation()
except FileNotFoundError:
    print("File not found")
except Exception as e:
    print(f"Error: {e}")
finally:
    cleanup()

# Raising exceptions
def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# Custom exceptions
class ValidationError(Exception):
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")
```

---

## 14.3 Working with Files

```python
# Reading files
with open("file.txt", "r") as f:
    content = f.read()       # Entire file
    lines = f.readlines()    # List of lines

# Writing files
with open("file.txt", "w") as f:
    f.write("Hello, World!")

# JSON files
import json

with open("data.json", "r") as f:
    data = json.load(f)

with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

# YAML files
import yaml

with open("config.yaml", "r") as f:
    config = yaml.safe_load(f)
```

---

## 14.4 HTTP Requests

```python
import requests

# GET request
response = requests.get("https://api.example.com/users")
response.status_code  # 200
response.json()       # Parse JSON
response.text         # Raw text

# GET with parameters
response = requests.get(
    "https://api.example.com/search",
    params={"q": "python", "limit": 10}
)

# POST request
response = requests.post(
    "https://api.example.com/users",
    json={"name": "Alice", "email": "alice@example.com"},
    headers={"Authorization": "Bearer TOKEN"}
)

# Error handling
response.raise_for_status()  # Raises exception for 4xx/5xx

# Session for multiple requests
session = requests.Session()
session.headers.update({"Authorization": "Bearer TOKEN"})
session.get("https://api.example.com/resource1")
session.get("https://api.example.com/resource2")
```

---

## 14.5 GCP Client Libraries

### Authentication

```python
from google.auth import default
from google.oauth2 import service_account

# Default credentials (ADC)
credentials, project = default()

# Service account file
credentials = service_account.Credentials.from_service_account_file(
    "service-account.json",
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)
```

### Cloud Storage

```python
from google.cloud import storage

# Create client
client = storage.Client()

# List buckets
for bucket in client.list_buckets():
    print(bucket.name)

# Get bucket
bucket = client.bucket("my-bucket")

# Upload file
blob = bucket.blob("path/to/file.txt")
blob.upload_from_filename("local-file.txt")
blob.upload_from_string("Hello, World!")

# Download file
blob = bucket.blob("path/to/file.txt")
blob.download_to_filename("local-file.txt")
content = blob.download_as_string()

# Generate signed URL
url = blob.generate_signed_url(
    version="v4",
    expiration=datetime.timedelta(hours=1),
    method="GET"
)
```

### BigQuery

```python
from google.cloud import bigquery

client = bigquery.Client()

# Run query
query = """
    SELECT name, count
    FROM `project.dataset.table`
    WHERE count > 10
    LIMIT 100
"""
results = client.query(query)

for row in results:
    print(f"{row.name}: {row.count}")

# Query to DataFrame
df = client.query(query).to_dataframe()

# Create table
table_id = "project.dataset.new_table"
schema = [
    bigquery.SchemaField("name", "STRING"),
    bigquery.SchemaField("count", "INTEGER"),
]
table = bigquery.Table(table_id, schema=schema)
table = client.create_table(table)

# Load data from GCS
job_config = bigquery.LoadJobConfig(
    source_format=bigquery.SourceFormat.CSV,
    skip_leading_rows=1,
)
load_job = client.load_table_from_uri(
    "gs://bucket/data.csv",
    table_id,
    job_config=job_config
)
load_job.result()  # Wait for completion
```

### Compute Engine

```python
from google.cloud import compute_v1

# List instances
client = compute_v1.InstancesClient()
instances = client.list(project="my-project", zone="us-central1-a")

for instance in instances:
    print(f"{instance.name}: {instance.status}")

# Start/stop instance
client.start(project="my-project", zone="us-central1-a", instance="my-vm")
client.stop(project="my-project", zone="us-central1-a", instance="my-vm")
```

### Pub/Sub

```python
from google.cloud import pubsub_v1

# Publisher
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("my-project", "my-topic")

# Publish message
future = publisher.publish(
    topic_path,
    data=b"Hello, World!",
    attribute_key="attribute_value"
)
message_id = future.result()

# Subscriber
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path("my-project", "my-subscription")

def callback(message):
    print(f"Received: {message.data}")
    message.ack()

streaming_pull_future = subscriber.subscribe(subscription_path, callback=callback)

# Pull messages synchronously
response = subscriber.pull(subscription_path, max_messages=10)
for msg in response.received_messages:
    print(msg.message.data)
    subscriber.acknowledge(subscription_path, [msg.ack_id])
```

---

## 14.6 Automation Scripts

### GCP Resource Cleanup Script

```python
#!/usr/bin/env python3
"""Delete old resources in a GCP project."""

from datetime import datetime, timedelta
from google.cloud import compute_v1, storage

PROJECT = "my-project"
MAX_AGE_DAYS = 30

def cleanup_old_instances():
    """Delete instances older than MAX_AGE_DAYS."""
    client = compute_v1.InstancesClient()
    cutoff = datetime.now() - timedelta(days=MAX_AGE_DAYS)

    # List all zones
    zones_client = compute_v1.ZonesClient()
    zones = zones_client.list(project=PROJECT)

    for zone in zones:
        instances = client.list(project=PROJECT, zone=zone.name)
        for instance in instances:
            if instance.name.startswith("temp-"):
                created = datetime.fromisoformat(instance.creation_timestamp[:-6])
                if created < cutoff:
                    print(f"Deleting {instance.name} in {zone.name}")
                    client.delete(project=PROJECT, zone=zone.name, instance=instance.name)

def cleanup_old_buckets():
    """Delete old objects in cleanup bucket."""
    client = storage.Client(project=PROJECT)
    bucket = client.bucket("cleanup-bucket")
    cutoff = datetime.now(timezone.utc) - timedelta(days=MAX_AGE_DAYS)

    for blob in bucket.list_blobs():
        if blob.updated < cutoff:
            print(f"Deleting {blob.name}")
            blob.delete()

if __name__ == "__main__":
    cleanup_old_instances()
    cleanup_old_buckets()
```

### Health Check Script

```python
#!/usr/bin/env python3
"""Check health of services and report."""

import requests
import sys
from concurrent.futures import ThreadPoolExecutor
from google.cloud import monitoring_v3

SERVICES = [
    ("API", "https://api.example.com/health"),
    ("Web", "https://www.example.com/health"),
    ("Admin", "https://admin.example.com/health"),
]

def check_service(name, url):
    """Check if service is healthy."""
    try:
        response = requests.get(url, timeout=10)
        return name, response.status_code == 200, response.elapsed.total_seconds()
    except Exception as e:
        return name, False, 0

def main():
    print("Checking services...")

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(check_service, name, url) for name, url in SERVICES]
        results = [f.result() for f in futures]

    all_healthy = True
    for name, healthy, latency in results:
        status = "OK" if healthy else "FAIL"
        print(f"  {name}: {status} ({latency:.2f}s)")
        if not healthy:
            all_healthy = False

    sys.exit(0 if all_healthy else 1)

if __name__ == "__main__":
    main()
```

---

## 14.7 Testing

```python
import pytest
from unittest.mock import Mock, patch

# Basic test
def test_add():
    assert add(2, 3) == 5

# Test with fixtures
@pytest.fixture
def sample_data():
    return {"name": "Alice", "age": 30}

def test_process_data(sample_data):
    result = process_data(sample_data)
    assert result["processed"] is True

# Mocking
@patch("mymodule.requests.get")
def test_fetch_data(mock_get):
    mock_get.return_value.json.return_value = {"data": "test"}
    result = fetch_data("https://api.example.com")
    assert result == {"data": "test"}
    mock_get.assert_called_once()

# Mocking GCP clients
@patch("mymodule.storage.Client")
def test_upload_file(mock_storage):
    mock_bucket = Mock()
    mock_storage.return_value.bucket.return_value = mock_bucket

    upload_file("bucket-name", "file.txt", b"content")

    mock_bucket.blob.assert_called_with("file.txt")
```

---

## 14.8 CLI Tools with Click

```python
import click

@click.group()
def cli():
    """GCP Management Tool."""
    pass

@cli.command()
@click.option("--project", required=True, help="GCP project ID")
@click.option("--zone", default="us-central1-a", help="Compute zone")
def list_vms(project, zone):
    """List all VMs in a project/zone."""
    from google.cloud import compute_v1
    client = compute_v1.InstancesClient()
    instances = client.list(project=project, zone=zone)
    for instance in instances:
        click.echo(f"{instance.name}: {instance.status}")

@cli.command()
@click.argument("bucket_name")
@click.argument("prefix", default="")
def list_objects(bucket_name, prefix):
    """List objects in a bucket."""
    from google.cloud import storage
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    for blob in bucket.list_blobs(prefix=prefix):
        click.echo(blob.name)

if __name__ == "__main__":
    cli()

# Usage:
# python tool.py list-vms --project=my-project
# python tool.py list-objects my-bucket logs/
```

---

## Chapter 15 Review Questions

1. How do you authenticate with GCP client libraries?

2. Write a script that lists all Cloud Storage buckets and their sizes.

3. How do you mock GCP client libraries in tests?

4. Write a function that publishes a message to Pub/Sub and waits for acknowledgment.

5. How do you run BigQuery queries and get results as a pandas DataFrame?

---

## Key Takeaways

1. **ADC is preferred** - Use default credentials when possible
2. **Client libraries are powerful** - Don't use raw REST APIs
3. **Use context managers** - `with` for files and resources
4. **Handle errors gracefully** - Try/except, proper logging
5. **Write tests** - Mock external services
6. **CLI tools** - Click makes it easy

---

[Next Chapter: Developer Tools →](./15-developer-tools.md)
