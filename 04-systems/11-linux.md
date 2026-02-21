# Chapter 11: Linux Systems Administration

## 9.1 Process Management

### Process States

```
RUNNING (R)  ←──┐
    │          │
    ▼          │
SLEEPING (S) ──┘
    │
    ▼
STOPPED (T)
    │
    ▼
ZOMBIE (Z) → TERMINATED
```

### Essential Commands

```bash
# View processes
ps aux                    # All processes
ps -ef                    # Full format
ps aux --sort=-%mem       # Sort by memory
ps aux --sort=-%cpu       # Sort by CPU

# Real-time monitoring
top                       # Basic
htop                      # Better (interactive)

# Process tree
pstree -p                 # With PIDs

# Find process
pgrep -f "pattern"        # Find by pattern
pidof nginx               # Find by name

# Kill processes
kill PID                  # SIGTERM (graceful)
kill -9 PID               # SIGKILL (force)
kill -HUP PID             # SIGHUP (reload config)
pkill -f "pattern"        # Kill by pattern
```

### /proc Filesystem

```bash
# Process info
ls /proc/PID/             # Process directory
cat /proc/PID/cmdline     # Command line
cat /proc/PID/environ     # Environment variables
cat /proc/PID/status      # Detailed status
cat /proc/PID/fd/         # Open file descriptors
cat /proc/PID/maps        # Memory mappings

# System info
cat /proc/cpuinfo         # CPU information
cat /proc/meminfo         # Memory information
cat /proc/loadavg         # Load average
cat /proc/uptime          # System uptime
```

---

## 9.2 Memory Management

### Memory Overview

```bash
# Memory usage
free -h

# Output explained:
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.5Gi       2.1Gi       500Mi       8.9Gi       10Gi
Swap:          2.0Gi         0B       2.0Gi

# Key metrics:
# - total: Total physical memory
# - used: Memory in use (includes buffers/cache)
# - free: Completely unused memory
# - buff/cache: Memory used for buffers and cache (reclaimable)
# - available: Memory available for new processes
```

### Memory Analysis

```bash
# Per-process memory
ps aux --sort=-%mem | head -20

# Detailed memory map
pmap PID

# Memory stats
vmstat 1                  # Every second
vmstat -s                 # Summary

# Cache/buffer info
cat /proc/meminfo | grep -E "(Cached|Buffers|MemFree)"
```

### OOM Killer

When memory is exhausted, Linux kills processes:

```bash
# Check OOM score
cat /proc/PID/oom_score

# Check OOM killer logs
dmesg | grep -i "out of memory"
journalctl -k | grep -i "oom"

# Protect process from OOM killer
echo -1000 > /proc/PID/oom_score_adj
```

---

## 9.3 Disk and I/O

### Disk Usage

```bash
# Disk space
df -h                     # Filesystem usage
df -i                     # Inode usage

# Directory size
du -sh /path              # Single directory
du -sh /*                 # All top-level dirs
du -h --max-depth=1       # One level deep

# Find large files
find / -size +100M -type f 2>/dev/null
```

### I/O Monitoring

```bash
# I/O statistics
iostat -x 1               # Extended stats, every second

# Output explained:
Device  r/s    w/s   rkB/s   wkB/s  await  %util
sda     10.5   25.3  420.0   1012.0  2.5    15.2
#       │      │     │       │       │      │
#       │      │     │       │       │      └── Utilization %
#       │      │     │       │       └── Average wait time (ms)
#       │      │     │       └── Write KB/s
#       │      │     └── Read KB/s
#       │      └── Writes per second
#       └── Reads per second

# Per-process I/O
iotop                     # Interactive
pidstat -d 1              # I/O stats per process
```

### File System Operations

```bash
# Mount filesystems
mount /dev/sdb1 /mnt/data
mount -t nfs server:/share /mnt/nfs
umount /mnt/data

# Check filesystem
fsck /dev/sdb1            # Check and repair (unmount first!)

# Extend filesystem
resize2fs /dev/sdb1       # For ext4
xfs_growfs /mount/point   # For xfs
```

---

## 9.4 Network Commands

```bash
# Interface info
ip addr show              # IP addresses
ip link show              # Interface status
ip route show             # Routing table

# Connection status
ss -tuln                  # Listening ports
ss -tunp                  # Connections with process
netstat -an               # All connections

# Network statistics
ip -s link                # Interface stats
ss -s                     # Socket summary

# Packet capture
tcpdump -i eth0           # All traffic on eth0
tcpdump -i eth0 port 80   # HTTP traffic
tcpdump -i eth0 host 10.0.0.5  # Traffic to/from host
tcpdump -w capture.pcap   # Save to file
```

---

## 9.5 Systemd and Services

### Service Management

```bash
# Control services
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx    # Reload config without restart
systemctl status nginx

# Enable/disable at boot
systemctl enable nginx
systemctl disable nginx

# Check if enabled
systemctl is-enabled nginx

# List services
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service
```

### Viewing Logs

```bash
# Journal logs
journalctl                # All logs
journalctl -u nginx       # Specific service
journalctl -f             # Follow (like tail -f)
journalctl -p err         # Only errors
journalctl --since "1 hour ago"
journalctl -k             # Kernel messages

# Traditional logs
tail -f /var/log/syslog
tail -f /var/log/messages
tail -f /var/log/auth.log
```

### Creating a Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# Reload and start
systemctl daemon-reload
systemctl enable myapp
systemctl start myapp
```

---

## 9.6 Performance Analysis

### CPU Analysis

```bash
# CPU usage
top                       # Overall view
mpstat 1                  # Per-CPU stats
pidstat 1                 # Per-process CPU

# Load average
uptime
cat /proc/loadavg

# Load average interpretation:
# 0.50 1.00 0.75
# │    │    │
# │    │    └── 15-minute average
# │    └── 5-minute average
# └── 1-minute average
#
# On 4-CPU system: load of 4.0 = 100% utilized
```

### System Call Tracing

```bash
# Trace system calls
strace -p PID             # Attach to process
strace -f command         # Trace command and children
strace -c command         # Summary statistics
strace -e open,read,write command  # Filter calls
```

### Performance Profiling

```bash
# perf (Linux performance tool)
perf top                  # Real-time function profiling
perf record -g command    # Record profile
perf report               # Analyze recording

# Generate flame graph
perf record -g -p PID
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

---

## 9.7 User and Permission Management

### User Commands

```bash
# User management
useradd -m username       # Create user with home dir
userdel -r username       # Delete user and home dir
usermod -aG group user    # Add user to group
passwd username           # Set password

# View user info
id username               # UID, GID, groups
groups username           # Group membership
whoami                    # Current user
```

### File Permissions

```
Permission format: rwxrwxrwx
                   │││││││││
                   │││││││└┴─ Others (o)
                   │││└┴┴──── Group (g)
                   └┴┴─────── Owner (u)

Numeric: r=4, w=2, x=1
  755 = rwxr-xr-x
  644 = rw-r--r--
  777 = rwxrwxrwx
```

```bash
# Change permissions
chmod 755 file
chmod u+x file            # Add execute for owner
chmod g-w file            # Remove write for group
chmod -R 755 directory    # Recursive

# Change ownership
chown user:group file
chown -R user:group dir   # Recursive
```

### Special Permissions

```bash
# SUID (4): Run as file owner
chmod u+s /usr/bin/passwd
ls -l: -rwsr-xr-x

# SGID (2): Run as group / inherit directory group
chmod g+s directory
ls -l: drwxr-sr-x

# Sticky bit (1): Only owner can delete
chmod +t /tmp
ls -l: drwxrwxrwt
```

---

## 9.8 Common Troubleshooting Scenarios

### High CPU Usage

```bash
# 1. Identify the process
top -c
ps aux --sort=-%cpu | head -10

# 2. Analyze the process
strace -p PID
perf top -p PID

# 3. Check what it's doing
cat /proc/PID/stack
ls -l /proc/PID/fd
```

### High Memory Usage

```bash
# 1. Check overall memory
free -h

# 2. Find memory hogs
ps aux --sort=-%mem | head -10

# 3. Check for memory leaks
pmap PID
cat /proc/PID/smaps | grep -E "^(Size|Rss|Pss)"

# 4. Check OOM events
dmesg | grep -i "out of memory"
```

### Disk Space Issues

```bash
# 1. Check disk usage
df -h

# 2. Find large files
du -sh /* | sort -rh | head -20
find / -size +100M -type f 2>/dev/null

# 3. Find deleted but open files
lsof | grep deleted

# 4. Check inodes
df -i
```

### Process Won't Die

```bash
# 1. Try graceful shutdown
kill PID

# 2. Force kill
kill -9 PID

# 3. If still won't die, check state
cat /proc/PID/status | grep State

# D state = uninterruptible sleep (usually I/O wait)
# Can't be killed until I/O completes
```

---

## Chapter 11 Review Questions

1. A process is using 100% CPU. How do you diagnose and resolve?

2. The system has high load average but low CPU usage. What could be the cause?

3. How do you find what's consuming disk space?

4. Explain the difference between RSS and VSZ in process memory.

5. A service fails to start. How do you troubleshoot?

6. What's the sticky bit and when would you use it?

---

## Key Takeaways

1. **/proc is your friend** - Rich source of system information
2. **Know your monitoring tools** - top, iostat, vmstat, ss
3. **systemd manages services** - journalctl for logs
4. **strace reveals behavior** - Essential for debugging
5. **Memory ≠ RAM** - Understand buffers, cache, swap
6. **Load average ≠ CPU** - Could be I/O wait

---

[Next Chapter: Containers & Kubernetes →](./10-containers-kubernetes.md)
