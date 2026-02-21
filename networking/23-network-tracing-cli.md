# Chapter 23: Network Tracing & CLI Tools

This chapter covers hands-on network debugging with CLI tools — the practical counterpart to the theory in Chapters 7 and 8.

## 23.1 Connectivity Testing: ping & arping

### ping

The most basic connectivity test — sends ICMP Echo Requests and measures round-trip time.

**Essential flags**:

| Flag | Purpose | Example |
|------|---------|---------|
| `-c` | Count (stop after N packets) | `ping -c 5 host` |
| `-i` | Interval between packets (seconds) | `ping -i 0.2 host` |
| `-s` | Packet size in bytes | `ping -s 1400 host` |
| `-W` | Timeout waiting for reply (seconds) | `ping -W 2 host` |
| `-M do` | Don't fragment (MTU discovery) | `ping -M do -s 1472 host` |
| `-t` | TTL value | `ping -t 10 host` |
| `-q` | Quiet — only show summary | `ping -q -c 100 host` |

**Reading ping output**:

```bash
$ ping -c 4 10.128.0.5

PING 10.128.0.5 (10.128.0.5) 56(84) bytes of data.
64 bytes from 10.128.0.5: icmp_seq=1 ttl=64 time=0.428 ms
64 bytes from 10.128.0.5: icmp_seq=2 ttl=64 time=0.351 ms
64 bytes from 10.128.0.5: icmp_seq=3 ttl=64 time=0.367 ms
64 bytes from 10.128.0.5: icmp_seq=4 ttl=64 time=0.354 ms

--- 10.128.0.5 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 0.351/0.375/0.428/0.031 ms
```

```
Field breakdown:
  64 bytes       → Response size (56 data + 8 ICMP header)
  icmp_seq=1     → Sequence number (gaps = packet loss)
  ttl=64         → Hops remaining (start 64 = Linux, 128 = Windows, 255 = network device)
  time=0.428 ms  → Round-trip time

Summary:
  0% packet loss → All packets returned
  rtt avg        → Mean latency
  mdev           → Standard deviation (jitter)
```

**MTU discovery with ping**:

```bash
# Find the maximum payload that doesn't fragment
# Ethernet MTU = 1500, minus 20 IP + 8 ICMP = 1472 max payload
$ ping -M do -s 1472 -c 1 10.128.0.5
PING 10.128.0.5 (10.128.0.5) 1472(1500) bytes of data.
1480 bytes from 10.128.0.5: icmp_seq=1 ttl=64 time=0.512 ms

# Too large — path MTU is smaller
$ ping -M do -s 1473 -c 1 10.128.0.5
ping: local error: message too long, mtu=1500
```

### arping

Layer 2 reachability test — sends ARP requests instead of ICMP. Useful when ICMP is blocked by firewalls.

```bash
# Check if host is reachable at Layer 2
$ arping -c 3 10.128.0.5
ARPING 10.128.0.5 from 10.128.0.2 eth0
Unicast reply from 10.128.0.5 [42:01:0a:80:00:05]  0.563ms
Unicast reply from 10.128.0.5 [42:01:0a:80:00:05]  0.471ms
Unicast reply from 10.128.0.5 [42:01:0a:80:00:05]  0.489ms

# Detect duplicate IP addresses
$ arping -D -c 3 10.128.0.5
# No response = no duplicate; response = IP conflict
```

### IPv6: ping6

```bash
# IPv6 ping (some systems use ping -6)
$ ping6 -c 3 2001:db8::1
$ ping -6 -c 3 2001:db8::1

# Link-local address (requires interface)
$ ping6 -c 3 fe80::1%eth0
```

---

## 23.2 Path Tracing: traceroute, tracepath & mtr

### How traceroute works

```
Traceroute exploits the IP TTL field. Each router decrements TTL by 1.
When TTL hits 0, the router sends back an ICMP "Time Exceeded" message,
revealing its address.

Host            Router A         Router B         Destination
 │                │                │                  │
 │──TTL=1────────▶│                │                  │
 │◀──ICMP Exceeded│                │                  │
 │                │                │                  │
 │──TTL=2─────────┼──TTL=1───────▶│                  │
 │                │◀──ICMP Exceeded│                  │
 │◀───────────────┤                │                  │
 │                │                │                  │
 │──TTL=3─────────┼──TTL=2────────┼──TTL=1──────────▶│
 │                │               │◀──Port Unreachable│
 │◀───────────────┤───────────────┤                  │
 │                │                │                  │
 │  Hop 1 found    Hop 2 found     Destination found │
```

### traceroute flags

| Flag | Purpose | Example |
|------|---------|---------|
| `-I` | Use ICMP Echo instead of UDP | `traceroute -I host` |
| `-T` | Use TCP SYN (port 80 default) | `traceroute -T host` |
| `-n` | Don't resolve hostnames | `traceroute -n host` |
| `-p` | Set destination port | `traceroute -T -p 443 host` |
| `-w` | Wait time per probe (seconds) | `traceroute -w 3 host` |
| `-q` | Number of probes per hop | `traceroute -q 1 host` |
| `-m` | Max TTL (hops) | `traceroute -m 20 host` |

**Choosing the probe mode**:

```
UDP (default):   May be blocked by firewalls. Uses high-numbered ports.
ICMP (-I):       Works well when ICMP is allowed. Same as ping.
TCP (-T):        Best for firewall traversal. Uses SYN packets.
                 Choose port 443 to mimic HTTPS traffic.
```

```bash
# Default UDP traceroute
$ traceroute 10.128.0.50

traceroute to 10.128.0.50 (10.128.0.50), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.234 ms  1.123 ms  1.456 ms
 2  10.0.0.1 (10.0.0.1)  5.678 ms  5.432 ms  5.789 ms
 3  * * *
 4  72.14.236.208 (72.14.236.208)  8.123 ms  8.456 ms  8.789 ms
 5  10.128.0.50 (10.128.0.50)  10.234 ms  10.123 ms  10.456 ms

# TCP traceroute on port 443 — better for firewall traversal
$ traceroute -T -p 443 10.128.0.50
```

### tracepath

Like traceroute, but also discovers Path MTU. Does not require root.

```bash
$ tracepath 10.128.0.50

 1?: [LOCALHOST]                     pmtu 1500
 1:  gateway                          0.345ms
 1:  gateway                          0.289ms
 2:  10.0.0.1                         5.678ms asymm  3
 3:  10.128.0.50                     10.234ms reached
     Resume: pmtu 1500 hops 3 back 3
```

The `pmtu` value at the end tells you the maximum packet size the entire path supports.

### mtr — combined ping + traceroute

`mtr` continuously probes each hop, building live statistics. It's the single best tool for diagnosing path-level issues.

```bash
# Interactive mode (live updating)
$ mtr 10.128.0.50

# Report mode (for scripting and sharing)
$ mtr --report -c 100 10.128.0.50

Start: 2026-02-20T10:00:00+0000
HOST: web-server-01            Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- gateway                 0.0%   100    0.3   0.4   0.2   1.2   0.1
  2.|-- 10.0.0.1                0.0%   100    5.4   5.6   5.1   8.2   0.4
  3.|-- 72.14.236.208           2.0%   100    8.5   8.8   8.1  15.3   1.2
  4.|-- 10.128.0.50             0.0%   100   10.2  10.4   9.8  14.1   0.6

# TCP mode — bypasses firewalls that block ICMP/UDP
$ mtr --tcp --port 443 10.128.0.50

# UDP mode on a specific port
$ mtr --udp --port 53 8.8.8.8
```

**Reading mtr output**:

```
Loss%  → Packet loss at this hop
Snt    → Packets sent
Last   → Last probe RTT
Avg    → Average RTT
Best   → Minimum RTT
Wrst   → Maximum RTT
StDev  → Standard deviation (high = inconsistent)
```

**Interpreting common patterns**:

```
Pattern: Loss at one hop, but NOT at subsequent hops
  1.|-- gateway          0.0%    0.3ms
  2.|-- isp-router       5.0%    5.4ms   ← 5% loss here
  3.|-- destination      0.0%   10.2ms   ← but 0% here!
Diagnosis: Router at hop 2 is rate-limiting ICMP replies.
           This is NOT real packet loss — it's cosmetic.

Pattern: Loss starts at one hop AND continues to destination
  1.|-- gateway          0.0%    0.3ms
  2.|-- isp-router      15.0%    5.4ms   ← loss starts
  3.|-- destination     15.0%   10.2ms   ← loss continues
Diagnosis: Real packet loss at or before hop 2.

Pattern: *** at a hop
  1.|-- gateway          0.0%    0.3ms
  2.|-- ???             100.0%
  3.|-- destination      0.0%   10.2ms
Diagnosis: Hop 2 is a firewall or router that doesn't send
           ICMP Time Exceeded. Traffic still passes through.
```

---

## 23.3 DNS Debugging: dig, nslookup & host

### dig — the DNS Swiss Army knife

**Essential queries**:

```bash
# Standard lookup
$ dig example.com

# Short answer only
$ dig +short example.com
93.184.216.34

# Query a specific DNS server
$ dig @8.8.8.8 example.com

# Get all sections with stats
$ dig +all example.com

# Just the stats (query time, server used)
$ dig +stats example.com

# Find authoritative nameservers
$ dig +nssearch example.com
SOA ns1.example.com. admin.example.com. 2024011501 7200 3600 1209600 86400 from server 93.184.216.34 in 12 ms.
```

### Tracing the full DNS delegation chain

`dig +trace` walks the entire resolution path from root servers to the authoritative answer:

```bash
$ dig +trace example.com

; <<>> DiG 9.16.1 <<>> +trace example.com
.                       518400  IN  NS  a.root-servers.net.
.                       518400  IN  NS  b.root-servers.net.
;; Received 239 bytes from 8.8.8.8#53(8.8.8.8) in 12 ms

com.                    172800  IN  NS  a.gtld-servers.net.
com.                    172800  IN  NS  b.gtld-servers.net.
;; Received 936 bytes from 198.41.0.4#53(a.root-servers.net) in 24 ms

example.com.            172800  IN  NS  ns1.example.com.
example.com.            172800  IN  NS  ns2.example.com.
;; Received 276 bytes from 192.5.6.30#53(a.gtld-servers.net) in 31 ms

example.com.            86400   IN  A   93.184.216.34
;; Received 56 bytes from 93.184.216.34#53(ns1.example.com) in 8 ms
```

```
What to look for:
  1. Root servers → which TLD servers are returned
  2. TLD servers → which authoritative NSes are returned
  3. Authoritative NS → the actual answer
  4. Timing at each step → where is the latency?
  5. Unexpected delegations → misconfigured NS records
```

### nslookup & host

```bash
# nslookup — interactive mode
$ nslookup
> server 8.8.8.8
> set type=MX
> example.com
Server:  8.8.8.8
Address: 8.8.8.8#53

example.com     mail exchanger = 10 mail.example.com.

# host — quick one-line lookups
$ host example.com
example.com has address 93.184.216.34
example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946
example.com mail is handled by 10 mail.example.com.

# Reverse lookup
$ host 93.184.216.34
34.216.184.93.in-addr.arpa domain name pointer example.com.
```

### Debugging slow DNS

```
Symptom: Slow DNS resolution

Step 1: Check query time
  $ dig example.com | grep "Query time"
  ;; Query time: 230 msec    ← Anything > 100ms is suspect

Step 2: Compare resolvers
  $ dig @8.8.8.8 example.com     | grep "Query time"
  $ dig @1.1.1.1 example.com     | grep "Query time"
  $ dig @169.254.169.254 example.com | grep "Query time"

Step 3: Check if it's a caching issue
  $ dig example.com    # First query (cold cache)
  ;; Query time: 230 msec
  $ dig example.com    # Second query (cached)
  ;; Query time: 1 msec

Step 4: Check TTL
  $ dig +short +ttlid example.com
  # Very low TTL (< 60s) means frequent re-resolution

Step 5: Trace the delegation
  $ dig +trace example.com
  # Identify which step is slow
```

**Common DNS failure modes**:

| Status | Meaning | Likely Cause |
|--------|---------|--------------|
| `NOERROR` | Success | — |
| `NXDOMAIN` | Domain doesn't exist | Typo, expired domain, wrong zone |
| `SERVFAIL` | Server failure | Upstream NS down, DNSSEC failure, timeout |
| `REFUSED` | Server refused | Not authorized to query this server |
| `FORMERR` | Format error | Malformed query (rare) |

---

## 23.4 Packet Capture: tcpdump

`tcpdump` is the foundational packet capture tool. Understanding its filter syntax and output is essential for network debugging.

### Filter syntax

```bash
# Filter by host
tcpdump host 10.128.0.5
tcpdump src host 10.128.0.5
tcpdump dst host 10.128.0.5

# Filter by network
tcpdump net 10.128.0.0/24

# Filter by port
tcpdump port 443
tcpdump src port 8080
tcpdump dst port 53

# Filter by protocol
tcpdump icmp
tcpdump tcp
tcpdump udp

# Combine with logical operators
tcpdump 'host 10.128.0.5 and port 443'
tcpdump 'src host 10.128.0.5 or src host 10.128.0.6'
tcpdump 'port 80 and not host 10.128.0.99'

# Parentheses for grouping (quote the expression)
tcpdump '(src host 10.128.0.5 or src host 10.128.0.6) and dst port 443'
```

### Advanced filters

```bash
# TCP flags — capture only SYN packets (new connections)
tcpdump 'tcp[tcpflags] & tcp-syn != 0'

# SYN but NOT ACK (initial SYN only, not SYN-ACK)
tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'

# RST packets (connection resets)
tcpdump 'tcp[tcpflags] & tcp-rst != 0'

# FIN packets (connection teardown)
tcpdump 'tcp[tcpflags] & tcp-fin != 0'

# VLAN-tagged traffic
tcpdump vlan

# Packets larger than 1000 bytes
tcpdump 'greater 1000'

# ICMP unreachable messages
tcpdump 'icmp[icmptype] == icmp-unreach'
```

### Reading tcpdump output

```bash
$ tcpdump -i eth0 -nn 'host 10.128.0.5 and port 443' -c 10

10:15:32.123456 IP 10.128.0.2.54321 > 10.128.0.5.443: Flags [S], seq 1234567890, win 65535, options [mss 1460,sackOK,TS val 123 ecr 0,nop,wscale 7], length 0
10:15:32.124012 IP 10.128.0.5.443 > 10.128.0.2.54321: Flags [S.], seq 987654321, ack 1234567891, win 65535, options [mss 1460,sackOK,TS val 456 ecr 123,nop,wscale 7], length 0
10:15:32.124089 IP 10.128.0.2.54321 > 10.128.0.5.443: Flags [.], ack 1, win 512, length 0
10:15:32.125234 IP 10.128.0.2.54321 > 10.128.0.5.443: Flags [P.], seq 1:518, ack 1, win 512, length 517
```

**TCP flag reference**:

```
Flag    Symbol   Meaning
──────────────────────────────────────
SYN     [S]      Connection initiation
SYN-ACK [S.]     Connection acknowledgment
ACK     [.]      Acknowledgment
PSH-ACK [P.]     Data push with ack
FIN     [F.]     Connection close
RST     [R.]     Connection reset
RST-ACK [R]      Reset with ack
```

**Decoding a line**:

```
10:15:32.123456 IP 10.128.0.2.54321 > 10.128.0.5.443: Flags [S], seq 1234567890, win 65535, length 0
│               │  │              │    │              │         │   │              │          │
│               │  │              │    │              │         │   │              │          └─ Payload bytes
│               │  │              │    │              │         │   │              └─ Window size
│               │  │              │    │              │         │   └─ Sequence number
│               │  │              │    │              │         └─ TCP flags
│               │  │              │    │              └─ Destination port
│               │  │              │    └─ Destination IP
│               │  │              └─ Source port
│               │  └─ Source IP
│               └─ Protocol
└─ Timestamp
```

### Capture options

```bash
# Write to pcap file (for later analysis in Wireshark)
tcpdump -i eth0 -w capture.pcap

# Read from pcap file
tcpdump -r capture.pcap

# Limit capture to N packets
tcpdump -c 100 -i eth0

# Set snap length (bytes per packet to capture)
tcpdump -s 96 -i eth0          # Headers only (fast)
tcpdump -s 0 -i eth0           # Full packet

# Ring buffer — rotate across 5 files of 100MB each
tcpdump -i eth0 -w trace.pcap -C 100 -W 5

# Don't resolve hostnames or ports (faster output)
tcpdump -nn -i eth0

# Capture on all interfaces
tcpdump -i any
```

### Practical recipes

```bash
# Capture a TLS handshake
tcpdump -i eth0 -nn 'tcp port 443 and (tcp[tcpflags] & tcp-syn != 0 or tcp[((tcp[12:1] & 0xf0) >> 2):1] = 0x16)'

# Find TCP retransmissions (duplicate seq numbers)
# Capture to file, then analyze:
tcpdump -i eth0 -w /tmp/capture.pcap 'host 10.128.0.5'
# Analyze with: tshark -r /tmp/capture.pcap -Y 'tcp.analysis.retransmission'

# Diagnose RST packets (unexpected connection resets)
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'

# Capture DNS queries and responses
tcpdump -i eth0 -nn port 53

# Watch HTTP requests (unencrypted)
tcpdump -i eth0 -nn -A 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

---

## 23.5 Socket & Connection Inspection: ss & netstat

### ss — the modern socket inspector

`ss` (socket statistics) is the replacement for `netstat`. It's faster and provides more detail.

**Essential commands**:

```bash
# Listening TCP sockets with process info
$ ss -tlnp
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
LISTEN  0       128     0.0.0.0:22            0.0.0.0:*          users:(("sshd",pid=1234,fd=3))
LISTEN  0       511     0.0.0.0:80            0.0.0.0:*          users:(("nginx",pid=5678,fd=6))
LISTEN  0       128     0.0.0.0:443           0.0.0.0:*          users:(("nginx",pid=5678,fd=7))

# Flags: -t=TCP  -l=listening  -n=numeric  -p=process
```

```bash
# Socket statistics summary
$ ss -s
Total: 342
TCP:   156 (estab 89, closed 12, orphaned 3, timewait 12)

Transport    Total   IP    IPv6
RAW          1       0     1
UDP          8       5     3
TCP          144     98    46
INET         153     103   50
FRAG         0       0     0
```

**Filter by TCP state**:

```bash
# All established connections
$ ss state established

# TIME_WAIT sockets (potential port exhaustion)
$ ss state time-wait

# SYN-RECV (potential SYN flood)
$ ss state syn-recv

# Close-wait (application not closing sockets)
$ ss state close-wait
```

**Filter by address**:

```bash
# Connections to a specific destination
$ ss dst 10.128.0.5

# Connections from a specific source port range
$ ss sport gt 1024

# Connections to port 443
$ ss dst :443

# Combined: established connections to a host on port 443
$ ss state established dst 10.128.0.5 dport = :443
```

**Detailed socket info** (timer, memory, congestion):

```bash
$ ss -ti state established
ESTAB   0   0   10.128.0.2:54321   10.128.0.5:443
     cubic wscale:7,7 rto:204 rtt:1.5/0.5 ato:40 mss:1448 pmtu:1500
     rcvmss:1448 advmss:1448 cwnd:10 bytes_sent:1024 bytes_acked:1024
     bytes_received:2048 segs_out:15 segs_in:12 send 77.2Mbps
     lastsnd:120 lastrcv:80 lastack:80 pacing_rate 154.3Mbps
     delivery_rate 48.5Mbps retrans:0/0 rcv_space:29200 rcv_ssthresh:29200
```

### netstat (legacy but still useful)

```bash
# Routing table
$ netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         10.128.0.1      0.0.0.0         UG        0 0          0 eth0
10.128.0.0      0.0.0.0         255.255.255.0   U         0 0          0 eth0

# Interface statistics
$ netstat -i
Kernel Interface table
Iface      MTU    RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
eth0      1460  1234567      0      3      0   987654      0      0      0 BMRU
lo       65536    56789      0      0      0    56789      0      0      0 LRU
```

### Diagnosing common socket issues

**Port exhaustion**:

```bash
# Count connections by state
$ ss -s
# Look for high timewait count

# Count ephemeral ports in use
$ ss state established | wc -l

# Check ephemeral port range
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768   60999
# That's 28,231 ports — if established + timewait approaches this, you're exhausted
```

**TIME_WAIT accumulation**:

```bash
# Count TIME_WAIT sockets
$ ss state time-wait | wc -l

# See which destinations are accumulating TIME_WAIT
$ ss state time-wait dst :443 | awk '{print $4}' | sort | uniq -c | sort -rn | head
   1523 10.128.0.5:443
    892 10.128.0.6:443
```

---

## 23.6 Routing & IP Tools: ip route, ip neigh, ip link

### ip route — routing table inspection

```bash
# Show the main routing table
$ ip route show
default via 10.128.0.1 dev eth0 proto dhcp src 10.128.0.2 metric 100
10.128.0.0/24 dev eth0 proto kernel scope link src 10.128.0.2
169.254.169.254 via 10.128.0.1 dev eth0  # GCP metadata server
```

**The single most useful routing command — `ip route get`**:

```bash
# How would the kernel route a packet to this destination?
$ ip route get 10.130.0.5
10.130.0.5 via 10.128.0.1 dev eth0 src 10.128.0.2 uid 1000
    cache

$ ip route get 8.8.8.8
8.8.8.8 via 10.128.0.1 dev eth0 src 10.128.0.2 uid 1000
    cache
```

This tells you exactly: the next-hop gateway, the outgoing interface, and the source IP the kernel will use.

```bash
# Show all routing tables (including policy routing)
$ ip route show table all

# Show a specific table
$ ip route show table local
```

### ip neigh — ARP/NDP table

```bash
# Show ARP cache (Layer 2 neighbor table)
$ ip neigh show
10.128.0.1 dev eth0 lladdr 42:01:0a:80:00:01 REACHABLE
10.128.0.5 dev eth0 lladdr 42:01:0a:80:00:05 STALE

# States:
#   REACHABLE  — Recently confirmed reachable
#   STALE      — Was reachable, needs re-verification
#   DELAY      — Waiting to re-verify
#   FAILED     — Neighbor unreachable
#   INCOMPLETE — ARP request sent, no reply yet
```

**Diagnosing ARP issues**:

```bash
# Clear a specific neighbor entry (force re-resolution)
$ ip neigh del 10.128.0.5 dev eth0

# Flush all entries for a device
$ ip neigh flush dev eth0

# Watch for ARP changes in real time
$ ip monitor neigh
```

### ip link — interface state and statistics

```bash
# Show all interfaces
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP qlen 1000
    link/ether 42:01:0a:80:00:02 brd ff:ff:ff:ff:ff:ff

# Key flags:
#   UP          — Admin up (ip link set up)
#   LOWER_UP    — Physical link detected
#   NO-CARRIER  — No physical link (cable unplugged)
```

**Interface error counters** — critical for diagnosing hardware or driver issues:

```bash
$ ip -s link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP qlen 1000
    link/ether 42:01:0a:80:00:02 brd ff:ff:ff:ff:ff:ff
    RX:  bytes  packets errors dropped  missed   mcast
      1234567    98765      0       3       0       0
    TX:  bytes  packets errors dropped carrier collsns
       987654    87654      0       0       0       0

# What to look for:
#   errors   — CRC errors, frame errors (bad cable, driver bug)
#   dropped  — Kernel ring buffer full (increase ring buffer or reduce traffic)
#   missed   — NIC ring buffer overflow
#   carrier  — Link flaps (bad cable, negotiation issues)
```

### Policy routing

```bash
# Show policy routing rules
$ ip rule show
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default

# Example: Route traffic from 10.128.1.0/24 through a different table
$ ip rule add from 10.128.1.0/24 table 100
$ ip route add default via 10.128.1.1 table 100
```

---

## 23.7 HTTP & API Debugging: curl

### Verbose connection output

```bash
$ curl -v https://api.example.com/health

*   Trying 93.184.216.34:443...
* Connected to api.example.com (93.184.216.34) port 443 (#0)
* ALPN: offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: CN=api.example.com
*  start date: Jan 15 00:00:00 2026 GMT
*  expire date: Apr 15 23:59:59 2026 GMT
*  issuer: C=US; O=Let's Encrypt; CN=R3
> GET /health HTTP/2
> Host: api.example.com
> User-Agent: curl/7.81.0
> Accept: */*
>
< HTTP/2 200
< content-type: application/json
< content-length: 15
<
{"status":"ok"}
```

```
What to read:
  *  lines → Connection and TLS info
  >  lines → Request sent to server
  <  lines → Response from server
```

### Timing breakdown with -w

This is one of the most powerful debugging techniques — pinpointing exactly where latency occurs.

```bash
$ curl -o /dev/null -s -w "\
    namelookup: %{time_namelookup}s\n\
       connect: %{time_connect}s\n\
    appconnect: %{time_appconnect}s\n\
   pretransfer: %{time_pretransfer}s\n\
 starttransfer: %{time_starttransfer}s\n\
         total: %{time_total}s\n\
    http_code:  %{http_code}\n" \
  https://api.example.com/health

    namelookup: 0.023s
       connect: 0.045s
    appconnect: 0.089s
   pretransfer: 0.089s
 starttransfer: 0.134s
         total: 0.135s
    http_code:  200
```

```
Timing interpretation:

  namelookup    0.023s  ─── DNS resolution
                        │
  connect       0.045s  ─── TCP handshake complete (connect - namelookup = TCP RTT)
                        │
  appconnect    0.089s  ─── TLS handshake complete (appconnect - connect = TLS time)
                        │
  starttransfer 0.134s  ─── First byte received (TTFB = starttransfer - appconnect)
                        │
  total         0.135s  ─── Transfer complete

  DNS slow?     → namelookup > 0.1s → check resolver, try different DNS
  Network slow? → connect - namelookup > 0.1s → high RTT, check path
  TLS slow?     → appconnect - connect > 0.2s → TLS negotiation issue
  Server slow?  → starttransfer - appconnect > 0.5s → backend processing time
```

### Bypass DNS and test specific backends

```bash
# Force resolution of api.example.com to a specific IP
$ curl --resolve api.example.com:443:10.128.0.5 https://api.example.com/health

# Useful for:
#   - Testing a specific backend behind a load balancer
#   - Verifying a new server before updating DNS
#   - Bypassing DNS caching issues
```

### Other useful curl flags

```bash
# Use a proxy
$ curl -x http://proxy.internal:8080 https://api.example.com/health

# Skip TLS verification (self-signed certs, debugging only)
$ curl -k https://self-signed.internal:8443/health

# Set a timeout (connect + total)
$ curl --connect-timeout 5 --max-time 30 https://api.example.com/health

# Show only response headers
$ curl -I https://api.example.com/health

# Follow redirects
$ curl -L https://example.com/old-path

# Send a specific Host header (test virtual hosting)
$ curl -H "Host: api.example.com" http://10.128.0.5/health
```

---

## 23.8 Firewall & iptables Tracing

### Reading iptables rules

```bash
# List all rules with counters and line numbers
$ iptables -L -n -v --line-numbers

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source          destination
1     1.2M  890M ACCEPT     all  --  lo     *       0.0.0.0/0       0.0.0.0/0
2      45K   12M ACCEPT     tcp  --  *      *       0.0.0.0/0       0.0.0.0/0       tcp dpt:22
3     890K  234M ACCEPT     tcp  --  *      *       0.0.0.0/0       0.0.0.0/0       tcp dpt:443
4        0     0 DROP       all  --  *      *       10.0.0.0/8      0.0.0.0/0

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source          destination

Chain OUTPUT (policy ACCEPT 1.5M packets, 456M bytes)
num   pkts bytes target     prot opt in     out     source          destination
```

```
Reading the output:
  pkts/bytes → Counter — has traffic hit this rule?
  target     → What to do (ACCEPT, DROP, REJECT, LOG, jump to chain)
  prot       → Protocol filter
  in/out     → Interface filter
  source/dst → Address filter

Key insight: A rule with 0 pkts has NEVER matched.
             Compare before/after to see if your traffic hits a rule.
```

### Netfilter chain order

Understanding which chain your traffic traverses is critical:

```
                               ┌────────────┐
                               │   Network  │
                               └─────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │ PREROUTING  │  ← DNAT, raw, mangle
                              └──────┬──────┘
                                     │
                              ┌──────▼──────┐
                              │  Routing    │
                              │  Decision   │
                              └──┬──────┬───┘
                                 │      │
                    For this host│      │For another host
                                 │      │
                          ┌──────▼──┐ ┌─▼────────┐
                          │  INPUT  │ │ FORWARD   │
                          └──┬──────┘ └─┬─────────┘
                             │          │
                     ┌───────▼──┐  ┌────▼────────┐
                     │  Local   │  │ POSTROUTING │  ← SNAT, masquerade
                     │ Process  │  └─────┬───────┘
                     └───────┬──┘        │
                             │     ┌─────▼──────┐
                      ┌──────▼──┐  │  Network   │
                      │ OUTPUT  │  └────────────┘
                      └──┬──────┘
                         │
                  ┌──────▼───────┐
                  │ POSTROUTING  │
                  └──────┬───────┘
                         │
                  ┌──────▼──────┐
                  │   Network   │
                  └─────────────┘
```

```
Traffic TO this host:      PREROUTING → INPUT → local process
Traffic FROM this host:    local process → OUTPUT → POSTROUTING
Traffic THROUGH this host: PREROUTING → FORWARD → POSTROUTING
```

### Packet tracing with iptables

When you can't figure out where traffic is being dropped:

```bash
# Enable TRACE on specific traffic (raw table, PREROUTING chain)
$ iptables -t raw -A PREROUTING -p tcp --dport 443 -s 10.0.0.5 -j TRACE
$ iptables -t raw -A OUTPUT -p tcp --sport 443 -d 10.0.0.5 -j TRACE

# View trace output in kernel log
$ dmesg -w | grep TRACE

# Example output:
TRACE: raw:PREROUTING:policy:2 IN=eth0 ... SRC=10.0.0.5 DST=10.128.0.2 ... DPT=443
TRACE: filter:INPUT:rule:3 IN=eth0 ... SRC=10.0.0.5 DST=10.128.0.2 ... DPT=443

# IMPORTANT: Remove trace rules when done (they generate a LOT of output)
$ iptables -t raw -D PREROUTING -p tcp --dport 443 -s 10.0.0.5 -j TRACE
$ iptables -t raw -D OUTPUT -p tcp --sport 443 -d 10.0.0.5 -j TRACE
```

### conntrack — connection tracking table

```bash
# List all tracked connections
$ conntrack -L
tcp      6 300 ESTABLISHED src=10.128.0.2 dst=10.128.0.5 sport=54321 dport=443 \
  src=10.128.0.5 dst=10.128.0.2 sport=443 dport=54321 [ASSURED] mark=0 use=1

# Watch new connections in real time
$ conntrack -E

# Count connections by state
$ conntrack -L -o extended | awk '{print $4}' | sort | uniq -c | sort -rn
   8923 ESTABLISHED
   1234 TIME_WAIT
     56 SYN_SENT
     12 CLOSE_WAIT

# Check conntrack table size vs limit
$ sysctl net.netfilter.nf_conntrack_count
net.netfilter.nf_conntrack_count = 10234
$ sysctl net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_max = 262144
# If count approaches max, new connections will be dropped
```

### Diagnosing dropped traffic — counter-based approach

```bash
# Step 1: Zero all counters
$ iptables -Z

# Step 2: Reproduce the problem (send the traffic)

# Step 3: Check which rules matched
$ iptables -L -n -v | grep -v "0     0"
# Any rule with non-zero counters was hit

# Step 4: Check for policy drops (end of chain with no match)
# If INPUT policy is DROP and no rule matched → traffic was policy-dropped

# Step 5: Check kernel drop counters
$ cat /proc/net/netstat | grep -i drop
$ nstat -z | grep -i drop
```

---

## 23.9 Troubleshooting Strategies

### Layer-by-layer methodology

Always start at the bottom and work up. Don't skip layers.

```
Layer 1 (Physical):
  └─ Is the link up?
     $ ip link show eth0           # Look for UP, LOWER_UP
     $ ethtool eth0                # Link detected? Speed? Duplex?

Layer 2 (Data Link):
  └─ Can we reach the gateway at L2?
     $ arping -c 3 GATEWAY_IP
     $ ip neigh show               # Is gateway in ARP table?

Layer 3 (Network):
  └─ Can we reach the destination at L3?
     $ ping -c 3 DEST_IP
     $ ip route get DEST_IP        # Which route is used?
     $ traceroute DEST_IP          # Where does it fail?

Layer 4 (Transport):
  └─ Can we connect to the port?
     $ ss -tlnp                    # Is the port listening?
     $ nc -zv DEST_IP PORT         # Can we TCP connect?
     $ tcpdump port PORT           # Do we see SYN-ACK?

Layer 7 (Application):
  └─ Does the application respond correctly?
     $ curl -v https://DEST:PORT/  # Full HTTP exchange
     $ dig DOMAIN                  # DNS resolving?
     $ openssl s_client ...        # TLS working?
```

### "Can't connect to service" — systematic checklist

```
1. DNS resolving?
   $ dig service.example.com
   If NXDOMAIN → wrong name, check spelling and zone
   If SERVFAIL → DNS server issue, try dig @8.8.8.8

2. Route exists?
   $ ip route get <service IP>
   If "unreachable" → no route, check routing table
   If wrong interface → misroute, check route specificity

3. Path clear?
   $ mtr --report -c 10 <service IP>
   If 100% loss at hop N → something blocking between N-1 and N
   If loss only at final hop → host or firewall issue

4. Port reachable?
   $ nc -zv <service IP> <port>
   "Connection refused"  → port not listening (service down?)
   "Connection timed out" → firewall dropping traffic
   "Connection reset"    → firewall rejecting traffic

5. Firewall allowing?
   $ iptables -L -n -v | grep <port>
   Check rule counters, check policy
   On GCP: check VPC firewall rules in console

6. Service running?
   $ ss -tlnp | grep <port>
   If not listening → service crashed, check logs
   $ systemctl status <service>
   $ journalctl -u <service> --since "5 minutes ago"
```

### "Application is slow" — latency breakdown

```bash
# Step 1: Where is the time spent?
$ curl -o /dev/null -s -w "\
DNS:     %{time_namelookup}s\n\
Connect: %{time_connect}s\n\
TLS:     %{time_appconnect}s\n\
TTFB:    %{time_starttransfer}s\n\
Total:   %{time_total}s\n" https://slow-service.example.com/api

DNS:     0.250s    ← Slow DNS? Check resolver
Connect: 0.252s    ← Fast TCP (2ms) — network is fine
TLS:     0.290s    ← Normal TLS (38ms)
TTFB:    1.850s    ← SLOW! 1.56s server processing
Total:   1.855s

# Diagnosis: Server-side latency. Check application logs, database queries,
# upstream API calls on the server.
```

### "Intermittent packet loss" — mtr + tcpdump correlation

```bash
# Step 1: Run mtr to identify where loss occurs
$ mtr --report -c 200 destination

# Step 2: If loss at a specific hop, capture at both ends
# On source:
$ tcpdump -i eth0 -w /tmp/source.pcap host destination

# On destination (if accessible):
$ tcpdump -i eth0 -w /tmp/dest.pcap host source

# Step 3: Compare — packets in source.pcap but not dest.pcap
# were lost in transit. Packets in both but with delays
# indicate congestion.
```

### "Works from one host, not another" — diff approach

```bash
# On working host:
$ ip route get DEST > /tmp/route-good.txt
$ iptables -L -n -v > /tmp/fw-good.txt
$ ss -tlnp > /tmp/listen-good.txt
$ cat /etc/resolv.conf > /tmp/dns-good.txt
$ traceroute DEST > /tmp/trace-good.txt

# On broken host:
$ ip route get DEST > /tmp/route-bad.txt
$ iptables -L -n -v > /tmp/fw-bad.txt
$ ss -tlnp > /tmp/listen-bad.txt
$ cat /etc/resolv.conf > /tmp/dns-bad.txt
$ traceroute DEST > /tmp/trace-bad.txt

# Compare:
$ diff /tmp/route-good.txt /tmp/route-bad.txt
$ diff /tmp/fw-good.txt /tmp/fw-bad.txt
$ diff /tmp/trace-good.txt /tmp/trace-bad.txt
```

### Quick reference: symptom → tool

| Symptom | First Tool | Second Tool | Third Tool |
|---------|-----------|-------------|------------|
| Can't reach host | `ping` | `traceroute` / `mtr` | `ip route get` |
| Can't connect to port | `nc -zv` | `ss -tlnp` (on server) | `iptables -L -n -v` |
| Slow connections | `curl -w` (timing) | `mtr --report` | `tcpdump` (retransmits) |
| DNS not resolving | `dig +trace` | `dig @8.8.8.8` | `cat /etc/resolv.conf` |
| Intermittent loss | `mtr --report -c 200` | `tcpdump` | `ip -s link` (errors) |
| Connection resets | `tcpdump` (RST flags) | `ss -tlnp` | `conntrack -L` |
| TLS errors | `curl -v` | `openssl s_client` | `dig` (cert name vs DNS) |
| Port exhaustion | `ss -s` | `ss state time-wait` | `conntrack -L` (count) |
| Packet drops | `ip -s link` | `iptables -L -n -v` | `nstat` / `netstat -s` |
| Routing wrong | `ip route get DEST` | `ip route show` | `ip rule show` |

---

## Chapter 23 Review Questions

1. You run `mtr --report -c 100` and see 10% packet loss at hop 3, but 0% at the final destination. Is hop 3 losing packets? Explain.

2. A curl request shows `time_namelookup: 2.5s` but `time_connect: 2.52s`. Where is the bottleneck and how would you fix it?

3. Write a tcpdump filter to capture only TCP SYN packets (not SYN-ACK) destined for port 443 from the 10.128.0.0/24 network.

4. You see 50,000 TIME_WAIT sockets on a busy proxy server. Is this a problem? What would you check and what are the mitigation options?

5. A service works when you `curl` it from the server itself but not from another host. Walk through your debugging methodology.

6. Explain the difference between `traceroute -I`, `traceroute -T`, and default `traceroute`. When would you use each?

---

## Chapter 23 Hands-On Exercises

### Exercise 23.1: Connectivity Diagnosis
1. Use `ping` with `-M do` to discover the path MTU to a remote host
2. Run `mtr --report -c 50` to the same host and identify each hop
3. Compare results from `traceroute` (default), `traceroute -I`, and `traceroute -T`

### Exercise 23.2: DNS Investigation
1. Use `dig +trace` to walk the full delegation chain for a domain
2. Compare query times from your local resolver, `8.8.8.8`, and `1.1.1.1`
3. Use `dig +nssearch` to find all authoritative servers and their SOA serial numbers

### Exercise 23.3: Packet Capture Analysis
1. Start a `tcpdump` capture filtering on port 443
2. Open an HTTPS connection with `curl -v` to any site
3. Identify the TCP handshake (SYN, SYN-ACK, ACK) and TLS handshake in the capture
4. Save to a pcap file and count the total packets exchanged

### Exercise 23.4: Socket and Routing Inspection
1. Use `ss -tlnp` to list all listening services on your machine
2. Use `ss -s` to get a summary and note the number of established connections
3. Use `ip route get` to check the route to three different destinations (local subnet, another VPC, internet)
4. Examine `ip -s link` for any interface errors or drops

### Exercise 23.5: End-to-End Troubleshooting
1. Use `curl -w` with timing variables to profile an HTTPS request
2. Identify the DNS, TCP, TLS, and server processing times
3. Determine which phase is the bottleneck
4. Propose and test a fix (e.g., switch DNS resolver, use `--resolve` to bypass DNS)

---

## Key Takeaways

1. **Start at the bottom layer** — Don't assume it's the application. Check L1/L2/L3 before L7.
2. **mtr is your best friend** — Combined ping + traceroute with statistics reveals path problems that one-shot traceroute misses.
3. **dig +trace shows the full picture** — Don't guess at DNS issues; trace the delegation chain.
4. **tcpdump flags tell the story** — Learn to read [S], [S.], [R.], [F.] — they reveal exactly what's happening on the wire.
5. **ss replaces netstat** — Use `ss -tlnp` for listening sockets, `ss -s` for summaries, state filters for targeted diagnosis.
6. **ip route get is definitive** — Don't read routing tables manually; ask the kernel which route it would use.
7. **curl -w timing breaks down latency** — DNS vs connect vs TLS vs TTFB tells you where to focus.
8. **Zero iptables counters, reproduce, check** — The counter-based approach cuts through firewall rule complexity.
9. **Compare working vs broken** — When one host works and another doesn't, diff their routing, firewall, and DNS configs.

---

[Previous Chapter: K8s Troubleshooting Scenarios ←](../kubernetes/22-troubleshooting-scenarios.md)
