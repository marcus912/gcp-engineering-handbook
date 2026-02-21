# Chapter 15: Building Developer Tools

## 15.1 Why Build Developer Tools?

Cloud engineers regularly need to:
- Automate repetitive debugging tasks
- Build tools to improve workflows
- Create scripts for faster diagnosis
- Develop testing utilities for reproduction

```
┌─────────────────────────────────────────────────────────────────┐
│                 Developer Tools Spectrum                         │
│                                                                  │
│  Quick Scripts ────────────────────────────────► Full Products  │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  Bash    │  │  Python  │  │   CLI    │  │  Internal    │   │
│  │ One-liner│  │  Script  │  │   Tool   │  │  Platform    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
│                                                                  │
│  Minutes        Hours         Days          Weeks               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 15.2 Automation Scripts

### GCP Diagnostic Script

```python
#!/usr/bin/env python3
"""
GCP Project Diagnostic Tool
Quickly gather diagnostic info for customer issues
"""

import subprocess
import json
import sys
from datetime import datetime

def run_cmd(cmd):
    """Run a shell command and return output."""
    try:
        result = subprocess.run(
            cmd, shell=True, capture_output=True, text=True, timeout=30
        )
        return result.stdout.strip()
    except subprocess.TimeoutExpired:
        return "TIMEOUT"
    except Exception as e:
        return f"ERROR: {e}"

def diagnose_project(project_id):
    """Run diagnostic checks on a GCP project."""
    print(f"=== GCP Diagnostic Report ===")
    print(f"Project: {project_id}")
    print(f"Time: {datetime.utcnow().isoformat()}Z")
    print()

    checks = [
        ("Project Info", f"gcloud projects describe {project_id} --format=json"),
        ("Enabled APIs", f"gcloud services list --enabled --project={project_id} --format='value(name)'"),
        ("Compute Instances", f"gcloud compute instances list --project={project_id} --format=json"),
        ("GKE Clusters", f"gcloud container clusters list --project={project_id} --format=json"),
        ("Recent Errors", f"gcloud logging read 'severity>=ERROR' --project={project_id} --limit=10 --format=json"),
        ("IAM Policy", f"gcloud projects get-iam-policy {project_id} --format=json"),
        ("Quotas", f"gcloud compute project-info describe --project={project_id} --format=json"),
    ]

    results = {}
    for name, cmd in checks:
        print(f"Checking: {name}...")
        results[name] = run_cmd(cmd)

    return results

def check_instance_health(project_id, zone, instance):
    """Detailed health check for a specific instance."""
    checks = {
        "status": f"gcloud compute instances describe {instance} --zone={zone} --project={project_id} --format='value(status)'",
        "serial_output": f"gcloud compute instances get-serial-port-output {instance} --zone={zone} --project={project_id} --start=0 2>&1 | tail -50",
        "network": f"gcloud compute instances describe {instance} --zone={zone} --project={project_id} --format='value(networkInterfaces[0].networkIP)'",
    }

    results = {}
    for name, cmd in checks.items():
        results[name] = run_cmd(cmd)

    return results

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: diagnose.py PROJECT_ID [--instance ZONE INSTANCE]")
        sys.exit(1)

    project_id = sys.argv[1]
    results = diagnose_project(project_id)

    # Output as JSON for further processing
    print("\n=== Raw Results ===")
    print(json.dumps(results, indent=2))
```

### Log Analysis Script

```python
#!/usr/bin/env python3
"""
Log Pattern Analyzer
Find common errors and patterns in GCP logs
"""

import json
import re
from collections import Counter
from datetime import datetime, timedelta

def parse_logs(log_file):
    """Parse JSON log entries."""
    logs = []
    with open(log_file) as f:
        for line in f:
            try:
                logs.append(json.loads(line))
            except json.JSONDecodeError:
                continue
    return logs

def extract_errors(logs):
    """Extract and categorize errors."""
    errors = []
    for log in logs:
        severity = log.get('severity', 'DEFAULT')
        if severity in ['ERROR', 'CRITICAL', 'ALERT', 'EMERGENCY']:
            message = log.get('textPayload', '') or \
                      log.get('jsonPayload', {}).get('message', '')
            errors.append({
                'timestamp': log.get('timestamp'),
                'severity': severity,
                'message': message[:200],  # Truncate long messages
                'resource': log.get('resource', {}).get('type', 'unknown'),
            })
    return errors

def find_patterns(errors):
    """Find common error patterns."""
    # Normalize messages (remove variable parts)
    normalized = []
    for error in errors:
        msg = error['message']
        # Remove IPs, IDs, timestamps
        msg = re.sub(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', 'IP_ADDR', msg)
        msg = re.sub(r'\b[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}\b', 'UUID', msg)
        msg = re.sub(r'\b\d{10,}\b', 'TIMESTAMP', msg)
        normalized.append(msg)

    return Counter(normalized).most_common(10)

def analyze_timeline(errors):
    """Analyze error timeline."""
    if not errors:
        return {}

    timestamps = []
    for e in errors:
        try:
            ts = datetime.fromisoformat(e['timestamp'].replace('Z', '+00:00'))
            timestamps.append(ts)
        except:
            continue

    if not timestamps:
        return {}

    return {
        'first_error': min(timestamps).isoformat(),
        'last_error': max(timestamps).isoformat(),
        'total_errors': len(timestamps),
        'errors_per_minute': len(timestamps) / max(1, (max(timestamps) - min(timestamps)).seconds / 60),
    }

def generate_report(log_file):
    """Generate a complete analysis report."""
    print("=== Log Analysis Report ===\n")

    logs = parse_logs(log_file)
    print(f"Total log entries: {len(logs)}")

    errors = extract_errors(logs)
    print(f"Error entries: {len(errors)}")

    print("\n--- Top Error Patterns ---")
    patterns = find_patterns(errors)
    for pattern, count in patterns:
        print(f"  [{count:4d}x] {pattern[:80]}...")

    print("\n--- Timeline Analysis ---")
    timeline = analyze_timeline(errors)
    for key, value in timeline.items():
        print(f"  {key}: {value}")

    print("\n--- Errors by Resource Type ---")
    resources = Counter(e['resource'] for e in errors)
    for resource, count in resources.most_common(5):
        print(f"  {resource}: {count}")

if __name__ == "__main__":
    import sys
    if len(sys.argv) < 2:
        print("Usage: analyze_logs.py LOG_FILE.json")
        sys.exit(1)
    generate_report(sys.argv[1])
```

---

## 15.3 Building CLI Tools

### CLI Tool with Click (Python)

```python
#!/usr/bin/env python3
"""
gcp-debug: A CLI tool for GCP debugging
"""

import click
import subprocess
import json
from rich.console import Console
from rich.table import Table

console = Console()

@click.group()
@click.option('--project', '-p', envvar='GCP_PROJECT', help='GCP Project ID')
@click.pass_context
def cli(ctx, project):
    """GCP Debugging CLI Tool"""
    ctx.ensure_object(dict)
    ctx.obj['project'] = project

@cli.command()
@click.argument('instance')
@click.option('--zone', '-z', required=True, help='Instance zone')
@click.pass_context
def vm_health(ctx, instance, zone):
    """Check VM instance health"""
    project = ctx.obj['project']

    console.print(f"[bold]Checking VM: {instance}[/bold]")

    # Get instance details
    cmd = f"gcloud compute instances describe {instance} --zone={zone} --project={project} --format=json"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        console.print(f"[red]Error: {result.stderr}[/red]")
        return

    data = json.loads(result.stdout)

    table = Table(title="VM Health Check")
    table.add_column("Check", style="cyan")
    table.add_column("Status", style="green")

    table.add_row("Status", data.get('status', 'UNKNOWN'))
    table.add_row("Machine Type", data.get('machineType', '').split('/')[-1])
    table.add_row("Internal IP", data.get('networkInterfaces', [{}])[0].get('networkIP', 'N/A'))

    # Check for external IP
    access_configs = data.get('networkInterfaces', [{}])[0].get('accessConfigs', [])
    external_ip = access_configs[0].get('natIP', 'None') if access_configs else 'None'
    table.add_row("External IP", external_ip)

    console.print(table)

@cli.command()
@click.argument('service')
@click.option('--namespace', '-n', default='default', help='Kubernetes namespace')
@click.pass_context
def k8s_debug(ctx, service, namespace):
    """Debug a Kubernetes service"""
    console.print(f"[bold]Debugging K8s Service: {service}[/bold]")

    # Get service
    cmd = f"kubectl get svc {service} -n {namespace} -o json"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        console.print(f"[red]Service not found: {result.stderr}[/red]")
        return

    svc = json.loads(result.stdout)

    # Get endpoints
    cmd = f"kubectl get endpoints {service} -n {namespace} -o json"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    endpoints = json.loads(result.stdout) if result.returncode == 0 else {}

    # Get pods
    selector = svc.get('spec', {}).get('selector', {})
    selector_str = ','.join(f"{k}={v}" for k, v in selector.items())
    cmd = f"kubectl get pods -n {namespace} -l {selector_str} -o json"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    pods = json.loads(result.stdout) if result.returncode == 0 else {}

    # Display results
    table = Table(title=f"Service: {service}")
    table.add_column("Property", style="cyan")
    table.add_column("Value", style="green")

    table.add_row("Type", svc.get('spec', {}).get('type', 'N/A'))
    table.add_row("Cluster IP", svc.get('spec', {}).get('clusterIP', 'N/A'))
    table.add_row("Ports", str(svc.get('spec', {}).get('ports', [])))

    # Endpoint count
    subsets = endpoints.get('subsets', [])
    endpoint_count = sum(len(s.get('addresses', [])) for s in subsets)
    table.add_row("Endpoints", str(endpoint_count))

    # Pod status
    pod_items = pods.get('items', [])
    ready_pods = sum(1 for p in pod_items if p.get('status', {}).get('phase') == 'Running')
    table.add_row("Pods", f"{ready_pods}/{len(pod_items)} Running")

    console.print(table)

    # Show pods table
    if pod_items:
        pod_table = Table(title="Pods")
        pod_table.add_column("Name")
        pod_table.add_column("Status")
        pod_table.add_column("Restarts")
        pod_table.add_column("Age")

        for pod in pod_items:
            name = pod['metadata']['name']
            phase = pod['status'].get('phase', 'Unknown')
            containers = pod['status'].get('containerStatuses', [])
            restarts = sum(c.get('restartCount', 0) for c in containers)
            pod_table.add_row(name, phase, str(restarts), "")

        console.print(pod_table)

@cli.command()
@click.option('--hours', '-h', default=1, help='Hours to look back')
@click.pass_context
def recent_errors(ctx, hours):
    """Show recent errors from Cloud Logging"""
    project = ctx.obj['project']

    console.print(f"[bold]Recent Errors (last {hours}h)[/bold]")

    cmd = f"""gcloud logging read 'severity>=ERROR' \
        --project={project} \
        --limit=20 \
        --format=json \
        --freshness={hours}h"""

    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

    if result.returncode != 0:
        console.print(f"[red]Error: {result.stderr}[/red]")
        return

    logs = json.loads(result.stdout) if result.stdout else []

    table = Table(title="Recent Errors")
    table.add_column("Time", style="cyan", width=20)
    table.add_column("Resource", style="yellow", width=20)
    table.add_column("Message", style="red", width=60)

    for log in logs[:20]:
        timestamp = log.get('timestamp', '')[:19]
        resource = log.get('resource', {}).get('type', 'unknown')
        message = log.get('textPayload', '') or \
                  str(log.get('jsonPayload', {}).get('message', ''))[:60]
        table.add_row(timestamp, resource, message)

    console.print(table)

if __name__ == '__main__':
    cli()
```

### CLI Tool with Cobra (Go)

```go
// cmd/root.go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var project string

var rootCmd = &cobra.Command{
	Use:   "gcp-debug",
	Short: "GCP debugging CLI tool",
	Long:  `A CLI tool for debugging GCP resources and services.`,
}

func Execute() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}

func init() {
	rootCmd.PersistentFlags().StringVarP(&project, "project", "p", "", "GCP Project ID")
}

// cmd/vm.go
package cmd

import (
	"encoding/json"
	"fmt"
	"os/exec"

	"github.com/spf13/cobra"
)

var zone string

var vmCmd = &cobra.Command{
	Use:   "vm [instance-name]",
	Short: "Check VM health",
	Args:  cobra.ExactArgs(1),
	Run: func(cmd *cobra.Command, args []string) {
		instance := args[0]
		checkVMHealth(instance, zone, project)
	},
}

func init() {
	rootCmd.AddCommand(vmCmd)
	vmCmd.Flags().StringVarP(&zone, "zone", "z", "", "Instance zone (required)")
	vmCmd.MarkFlagRequired("zone")
}

func checkVMHealth(instance, zone, project string) {
	cmdStr := fmt.Sprintf(
		"gcloud compute instances describe %s --zone=%s --project=%s --format=json",
		instance, zone, project,
	)

	out, err := exec.Command("bash", "-c", cmdStr).Output()
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	var data map[string]interface{}
	json.Unmarshal(out, &data)

	fmt.Printf("Instance: %s\n", instance)
	fmt.Printf("Status: %s\n", data["status"])
	fmt.Printf("Machine Type: %s\n", data["machineType"])
}
```

---

## 15.4 Testing Tools

### Reproduction Test Framework

```python
#!/usr/bin/env python3
"""
Issue Reproduction Framework
Systematically reproduce customer issues
"""

import time
import requests
import concurrent.futures
from dataclasses import dataclass
from typing import List, Callable

@dataclass
class TestResult:
    name: str
    passed: bool
    duration: float
    error: str = None
    details: dict = None

class ReproductionTest:
    """Base class for reproduction tests."""

    def __init__(self, name: str):
        self.name = name
        self.results: List[TestResult] = []

    def setup(self):
        """Override to set up test environment."""
        pass

    def teardown(self):
        """Override to clean up test environment."""
        pass

    def run_test(self, test_func: Callable) -> TestResult:
        """Run a single test function."""
        start = time.time()
        try:
            result = test_func()
            return TestResult(
                name=test_func.__name__,
                passed=True,
                duration=time.time() - start,
                details=result
            )
        except Exception as e:
            return TestResult(
                name=test_func.__name__,
                passed=False,
                duration=time.time() - start,
                error=str(e)
            )

    def run_all(self, tests: List[Callable]) -> List[TestResult]:
        """Run all tests and return results."""
        self.setup()
        try:
            for test in tests:
                result = self.run_test(test)
                self.results.append(result)
                status = "✓" if result.passed else "✗"
                print(f"  {status} {result.name} ({result.duration:.2f}s)")
        finally:
            self.teardown()
        return self.results


class LoadTest(ReproductionTest):
    """Load testing for reproducing performance issues."""

    def __init__(self, url: str, concurrency: int = 10, requests_per_worker: int = 100):
        super().__init__("Load Test")
        self.url = url
        self.concurrency = concurrency
        self.requests_per_worker = requests_per_worker

    def make_request(self, _):
        """Make a single request and return timing."""
        start = time.time()
        try:
            response = requests.get(self.url, timeout=30)
            return {
                'status': response.status_code,
                'duration': time.time() - start,
                'error': None
            }
        except Exception as e:
            return {
                'status': 0,
                'duration': time.time() - start,
                'error': str(e)
            }

    def run_load_test(self):
        """Run concurrent load test."""
        print(f"Running load test: {self.concurrency} workers x {self.requests_per_worker} requests")

        all_results = []
        with concurrent.futures.ThreadPoolExecutor(max_workers=self.concurrency) as executor:
            total_requests = self.concurrency * self.requests_per_worker
            results = list(executor.map(self.make_request, range(total_requests)))
            all_results.extend(results)

        # Analyze results
        durations = [r['duration'] for r in all_results]
        errors = [r for r in all_results if r['error']]
        status_codes = {}
        for r in all_results:
            code = r['status']
            status_codes[code] = status_codes.get(code, 0) + 1

        return {
            'total_requests': len(all_results),
            'avg_duration': sum(durations) / len(durations),
            'min_duration': min(durations),
            'max_duration': max(durations),
            'p95_duration': sorted(durations)[int(len(durations) * 0.95)],
            'error_count': len(errors),
            'error_rate': len(errors) / len(all_results),
            'status_codes': status_codes,
        }


class ConnectionTest(ReproductionTest):
    """Test connectivity to various endpoints."""

    def __init__(self, endpoints: List[dict]):
        super().__init__("Connection Test")
        self.endpoints = endpoints

    def test_endpoint(self, endpoint: dict) -> dict:
        """Test a single endpoint."""
        import socket

        host = endpoint['host']
        port = endpoint['port']
        timeout = endpoint.get('timeout', 5)

        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)
            start = time.time()
            result = sock.connect_ex((host, port))
            duration = time.time() - start
            sock.close()

            return {
                'host': host,
                'port': port,
                'reachable': result == 0,
                'duration': duration,
                'error': None if result == 0 else f"Connection failed: {result}"
            }
        except Exception as e:
            return {
                'host': host,
                'port': port,
                'reachable': False,
                'duration': 0,
                'error': str(e)
            }

    def run_connectivity_tests(self):
        """Test all endpoints."""
        results = []
        for endpoint in self.endpoints:
            result = self.test_endpoint(endpoint)
            status = "✓" if result['reachable'] else "✗"
            print(f"  {status} {result['host']}:{result['port']}")
            results.append(result)
        return results


# Example usage
if __name__ == "__main__":
    # Load test example
    load_test = LoadTest(
        url="http://example-service.default.svc.cluster.local:8080/health",
        concurrency=10,
        requests_per_worker=50
    )
    results = load_test.run_load_test()
    print(f"\nLoad Test Results:")
    print(f"  Total Requests: {results['total_requests']}")
    print(f"  Avg Duration: {results['avg_duration']:.3f}s")
    print(f"  P95 Duration: {results['p95_duration']:.3f}s")
    print(f"  Error Rate: {results['error_rate']:.2%}")

    # Connection test example
    conn_test = ConnectionTest([
        {'host': 'database.internal', 'port': 5432},
        {'host': 'redis.internal', 'port': 6379},
        {'host': 'api.external.com', 'port': 443},
    ])
    conn_test.run_connectivity_tests()
```

---

## 15.5 Debugging Tools

### Network Debug Tool

```python
#!/usr/bin/env python3
"""
Network Debug Tool
Diagnose network connectivity issues
"""

import subprocess
import socket
import ssl
import time
from urllib.parse import urlparse

def dns_lookup(hostname):
    """Perform DNS lookup."""
    print(f"\n=== DNS Lookup: {hostname} ===")
    try:
        # A record
        ips = socket.gethostbyname_ex(hostname)
        print(f"Hostname: {ips[0]}")
        print(f"Aliases: {ips[1]}")
        print(f"IPs: {ips[2]}")
        return ips[2]
    except socket.gaierror as e:
        print(f"DNS Error: {e}")
        return []

def tcp_connect(host, port, timeout=5):
    """Test TCP connectivity."""
    print(f"\n=== TCP Connect: {host}:{port} ===")
    try:
        start = time.time()
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((host, port))
        duration = time.time() - start
        sock.close()

        if result == 0:
            print(f"Connected successfully in {duration:.3f}s")
            return True
        else:
            print(f"Connection failed: error code {result}")
            return False
    except Exception as e:
        print(f"Connection error: {e}")
        return False

def tls_check(host, port=443):
    """Check TLS certificate."""
    print(f"\n=== TLS Check: {host}:{port} ===")
    try:
        context = ssl.create_default_context()
        with socket.create_connection((host, port), timeout=5) as sock:
            with context.wrap_socket(sock, server_hostname=host) as ssock:
                cert = ssock.getpeercert()
                print(f"TLS Version: {ssock.version()}")
                print(f"Cipher: {ssock.cipher()}")
                print(f"Subject: {dict(x[0] for x in cert['subject'])}")
                print(f"Issuer: {dict(x[0] for x in cert['issuer'])}")
                print(f"Valid Until: {cert['notAfter']}")
                return True
    except ssl.SSLError as e:
        print(f"TLS Error: {e}")
        return False
    except Exception as e:
        print(f"Connection Error: {e}")
        return False

def http_check(url):
    """Check HTTP endpoint."""
    print(f"\n=== HTTP Check: {url} ===")
    try:
        import urllib.request
        start = time.time()
        req = urllib.request.Request(url, headers={'User-Agent': 'NetworkDebug/1.0'})
        with urllib.request.urlopen(req, timeout=10) as response:
            duration = time.time() - start
            print(f"Status: {response.status}")
            print(f"Duration: {duration:.3f}s")
            print(f"Headers:")
            for header, value in response.headers.items():
                print(f"  {header}: {value}")
            return True
    except Exception as e:
        print(f"HTTP Error: {e}")
        return False

def traceroute(host):
    """Run traceroute."""
    print(f"\n=== Traceroute: {host} ===")
    try:
        result = subprocess.run(
            ['traceroute', '-m', '15', host],
            capture_output=True,
            text=True,
            timeout=30
        )
        print(result.stdout)
    except FileNotFoundError:
        # Try Windows tracert
        try:
            result = subprocess.run(
                ['tracert', '-h', '15', host],
                capture_output=True,
                text=True,
                timeout=30
            )
            print(result.stdout)
        except:
            print("traceroute/tracert not available")
    except Exception as e:
        print(f"Traceroute error: {e}")

def full_diagnosis(url):
    """Run full network diagnosis."""
    parsed = urlparse(url)
    host = parsed.hostname
    port = parsed.port or (443 if parsed.scheme == 'https' else 80)

    print("=" * 60)
    print(f"Full Network Diagnosis for: {url}")
    print("=" * 60)

    # DNS
    ips = dns_lookup(host)

    # TCP
    if ips:
        for ip in ips[:1]:  # Test first IP
            tcp_connect(ip, port)

    # TLS (if HTTPS)
    if parsed.scheme == 'https':
        tls_check(host, port)

    # HTTP
    http_check(url)

    # Traceroute
    traceroute(host)

if __name__ == "__main__":
    import sys
    if len(sys.argv) < 2:
        print("Usage: network_debug.py URL")
        print("Example: network_debug.py https://api.example.com/health")
        sys.exit(1)

    full_diagnosis(sys.argv[1])
```

### Memory Analyzer

```python
#!/usr/bin/env python3
"""
Process Memory Analyzer
Analyze memory usage of processes
"""

import os
import re
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class MemoryInfo:
    pid: int
    name: str
    rss_kb: int  # Resident Set Size
    vsz_kb: int  # Virtual Size
    shared_kb: int
    private_kb: int

def parse_proc_status(pid: int) -> Optional[dict]:
    """Parse /proc/[pid]/status for memory info."""
    try:
        with open(f'/proc/{pid}/status') as f:
            content = f.read()

        info = {}
        for line in content.split('\n'):
            if ':' in line:
                key, value = line.split(':', 1)
                info[key.strip()] = value.strip()
        return info
    except (FileNotFoundError, PermissionError):
        return None

def parse_proc_smaps(pid: int) -> dict:
    """Parse /proc/[pid]/smaps for detailed memory info."""
    totals = {
        'Rss': 0,
        'Pss': 0,  # Proportional Set Size
        'Shared_Clean': 0,
        'Shared_Dirty': 0,
        'Private_Clean': 0,
        'Private_Dirty': 0,
        'Swap': 0,
    }

    try:
        with open(f'/proc/{pid}/smaps') as f:
            for line in f:
                for key in totals:
                    if line.startswith(key + ':'):
                        value = int(re.search(r'(\d+)', line).group(1))
                        totals[key] += value
    except (FileNotFoundError, PermissionError):
        pass

    return totals

def get_process_memory(pid: int) -> Optional[MemoryInfo]:
    """Get memory info for a process."""
    status = parse_proc_status(pid)
    if not status:
        return None

    smaps = parse_proc_smaps(pid)

    return MemoryInfo(
        pid=pid,
        name=status.get('Name', 'unknown'),
        rss_kb=int(status.get('VmRSS', '0 kB').split()[0]),
        vsz_kb=int(status.get('VmSize', '0 kB').split()[0]),
        shared_kb=smaps['Shared_Clean'] + smaps['Shared_Dirty'],
        private_kb=smaps['Private_Clean'] + smaps['Private_Dirty'],
    )

def find_processes_by_name(name: str) -> List[int]:
    """Find PIDs by process name."""
    pids = []
    for entry in os.listdir('/proc'):
        if entry.isdigit():
            pid = int(entry)
            status = parse_proc_status(pid)
            if status and name.lower() in status.get('Name', '').lower():
                pids.append(pid)
    return pids

def analyze_container_memory(container_id: str):
    """Analyze memory for a container."""
    # Find container's cgroup
    cgroup_path = f'/sys/fs/cgroup/memory/docker/{container_id}'

    if not os.path.exists(cgroup_path):
        print(f"Container cgroup not found: {container_id}")
        return

    # Read cgroup memory stats
    stats = {}
    try:
        with open(f'{cgroup_path}/memory.stat') as f:
            for line in f:
                parts = line.strip().split()
                if len(parts) == 2:
                    stats[parts[0]] = int(parts[1])

        with open(f'{cgroup_path}/memory.usage_in_bytes') as f:
            stats['usage'] = int(f.read().strip())

        with open(f'{cgroup_path}/memory.limit_in_bytes') as f:
            stats['limit'] = int(f.read().strip())
    except FileNotFoundError:
        pass

    print(f"Container Memory Analysis: {container_id[:12]}")
    print(f"  Usage: {stats.get('usage', 0) / 1024 / 1024:.1f} MB")
    print(f"  Limit: {stats.get('limit', 0) / 1024 / 1024:.1f} MB")
    print(f"  RSS: {stats.get('rss', 0) / 1024 / 1024:.1f} MB")
    print(f"  Cache: {stats.get('cache', 0) / 1024 / 1024:.1f} MB")
    print(f"  Swap: {stats.get('swap', 0) / 1024 / 1024:.1f} MB")

    usage_pct = stats.get('usage', 0) / stats.get('limit', 1) * 100
    print(f"  Usage %: {usage_pct:.1f}%")

def top_memory_processes(n: int = 10):
    """Show top N processes by memory usage."""
    processes = []

    for entry in os.listdir('/proc'):
        if entry.isdigit():
            pid = int(entry)
            info = get_process_memory(pid)
            if info and info.rss_kb > 0:
                processes.append(info)

    # Sort by RSS
    processes.sort(key=lambda x: x.rss_kb, reverse=True)

    print(f"Top {n} Processes by Memory:")
    print(f"{'PID':>8} {'Name':<20} {'RSS (MB)':>10} {'Private (MB)':>12}")
    print("-" * 52)

    for proc in processes[:n]:
        print(f"{proc.pid:>8} {proc.name:<20} {proc.rss_kb/1024:>10.1f} {proc.private_kb/1024:>12.1f}")

if __name__ == "__main__":
    import sys

    if len(sys.argv) > 1:
        if sys.argv[1] == '--container' and len(sys.argv) > 2:
            analyze_container_memory(sys.argv[2])
        elif sys.argv[1] == '--process' and len(sys.argv) > 2:
            pids = find_processes_by_name(sys.argv[2])
            for pid in pids:
                info = get_process_memory(pid)
                if info:
                    print(f"PID {info.pid} ({info.name}): RSS={info.rss_kb/1024:.1f}MB")
        else:
            print("Usage:")
            print("  memory_analyzer.py                    # Top processes")
            print("  memory_analyzer.py --container ID     # Container analysis")
            print("  memory_analyzer.py --process NAME     # Process by name")
    else:
        top_memory_processes()
```

---

## 15.6 Tool Development Best Practices

### Code Organization

```
my-tool/
├── README.md              # Usage documentation
├── setup.py               # Python package setup
├── requirements.txt       # Dependencies
├── my_tool/
│   ├── __init__.py
│   ├── cli.py            # CLI entry point
│   ├── commands/         # Subcommands
│   │   ├── __init__.py
│   │   ├── diagnose.py
│   │   └── analyze.py
│   ├── lib/              # Core logic
│   │   ├── __init__.py
│   │   ├── gcp.py
│   │   └── k8s.py
│   └── utils/            # Utilities
│       ├── __init__.py
│       └── output.py
└── tests/
    ├── test_diagnose.py
    └── test_analyze.py
```

### Error Handling

```python
class ToolError(Exception):
    """Base exception for tool errors."""
    pass

class ConfigurationError(ToolError):
    """Configuration or setup error."""
    pass

class ConnectionError(ToolError):
    """Network or service connection error."""
    pass

def safe_run(func):
    """Decorator for safe execution with proper error handling."""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except ToolError as e:
            print(f"Error: {e}")
            sys.exit(1)
        except KeyboardInterrupt:
            print("\nCancelled by user")
            sys.exit(130)
        except Exception as e:
            print(f"Unexpected error: {e}")
            if os.environ.get('DEBUG'):
                import traceback
                traceback.print_exc()
            sys.exit(1)
    return wrapper
```

### Output Formatting

```python
from rich.console import Console
from rich.table import Table
from rich.progress import Progress

console = Console()

def print_table(title, columns, rows):
    """Print a formatted table."""
    table = Table(title=title)
    for col in columns:
        table.add_column(col['name'], style=col.get('style', ''))
    for row in rows:
        table.add_row(*row)
    console.print(table)

def print_json(data):
    """Print formatted JSON."""
    import json
    from rich.syntax import Syntax
    syntax = Syntax(json.dumps(data, indent=2), "json")
    console.print(syntax)

def with_progress(items, description):
    """Iterate with progress bar."""
    with Progress() as progress:
        task = progress.add_task(description, total=len(items))
        for item in items:
            yield item
            progress.advance(task)
```

---

## Chapter 15 Review Questions

1. When should you write a script vs a full CLI tool?

2. How would you design a tool to help customers reproduce their issues?

3. What makes a good debugging tool for production use?

4. How do you handle errors gracefully in CLI tools?

5. Describe a tool you would build to speed up your daily debugging work.

---

## Chapter 15 Hands-On Exercises

### Exercise 15.1: Build a Diagnostic Script
1. Create a script that gathers GCP project diagnostics
2. Include compute, networking, and logging checks
3. Output results in a format suitable for support tickets

### Exercise 15.2: Create a CLI Tool
1. Build a CLI tool with multiple subcommands
2. Add proper error handling and help text
3. Include output formatting (tables, JSON)

### Exercise 15.3: Build a Test Framework
1. Create a framework for reproducing performance issues
2. Include load testing and connectivity testing
3. Generate reports with results

---

## Key Takeaways

1. **Start simple** - Bash one-liner before Python script before full tool
2. **Automate repetitive tasks** - If you do it twice, script it
3. **Handle errors gracefully** - Users shouldn't see stack traces
4. **Format output clearly** - Tables, colors, progress bars
5. **Document as you build** - Help text and README
6. **Test your tools** - They need tests too
7. **Share with team** - Good tools multiply productivity

---

[Next Chapter: Kubernetes Monitoring & Observability →](../kubernetes/16-monitoring-observability.md)
