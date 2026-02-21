# Chapter 8: TCP/IP & The Network Stack

This chapter covers networking fundamentals that are critical for troubleshooting customer issues.

## 7.1 The OSI Model

The OSI (Open Systems Interconnection) model is a conceptual framework for understanding network communication.

```
┌─────────────────────────────────────────────────────────────────┐
│                         OSI Model                                │
├────────┬─────────────────┬──────────────────────────────────────┤
│ Layer  │ Name            │ Function                             │
├────────┼─────────────────┼──────────────────────────────────────┤
│   7    │ Application     │ User interface (HTTP, DNS, FTP)      │
│   6    │ Presentation    │ Data format, encryption (SSL/TLS)    │
│   5    │ Session         │ Connection management                │
│   4    │ Transport       │ End-to-end delivery (TCP, UDP)       │
│   3    │ Network         │ Routing, addressing (IP)             │
│   2    │ Data Link       │ Node-to-node delivery (Ethernet)     │
│   1    │ Physical        │ Bits on wire (cables, signals)       │
└────────┴─────────────────┴──────────────────────────────────────┘
```

### TCP/IP Model (Practical View)

The TCP/IP model is what's actually used in practice:

```
┌──────────────────────┐     ┌──────────────────────┐
│      OSI Model       │     │    TCP/IP Model      │
├──────────────────────┤     ├──────────────────────┤
│ 7. Application       │     │                      │
│ 6. Presentation      │ ──▶ │ 4. Application       │
│ 5. Session           │     │    (HTTP, DNS, SSH)  │
├──────────────────────┤     ├──────────────────────┤
│ 4. Transport         │ ──▶ │ 3. Transport         │
│                      │     │    (TCP, UDP)        │
├──────────────────────┤     ├──────────────────────┤
│ 3. Network           │ ──▶ │ 2. Internet          │
│                      │     │    (IP, ICMP)        │
├──────────────────────┤     ├──────────────────────┤
│ 2. Data Link         │     │ 1. Network Access    │
│ 1. Physical          │ ──▶ │    (Ethernet, WiFi)  │
└──────────────────────┘     └──────────────────────┘
```

### Data Encapsulation

As data moves down the stack, each layer adds its header:

```
Application Layer:  [      DATA      ]
                             ↓
Transport Layer:    [TCP][   DATA    ]  ← Segment
                             ↓
Network Layer:      [IP][TCP][ DATA  ]  ← Packet
                             ↓
Data Link Layer:   [ETH][IP][TCP][DATA][FCS]  ← Frame
```

---

## 7.2 IP Addressing

### IPv4 Addresses

32-bit addresses written in dotted decimal notation.

**Structure**:
```
192.168.1.100
↓   ↓   ↓  ↓
Binary: 11000000.10101000.00000001.01100100
```

**Classes (Historical)**:
| Class | First Octet | Default Mask | Networks |
|-------|-------------|--------------|----------|
| A | 1-126 | /8 | Large organizations |
| B | 128-191 | /16 | Medium organizations |
| C | 192-223 | /24 | Small organizations |

### Private IP Ranges (RFC 1918)

```
Class A: 10.0.0.0    - 10.255.255.255   (10.0.0.0/8)
Class B: 172.16.0.0  - 172.31.255.255   (172.16.0.0/12)
Class C: 192.168.0.0 - 192.168.255.255  (192.168.0.0/16)
```

### CIDR Notation

Classless Inter-Domain Routing allows flexible subnet sizes.

```
10.0.0.0/24 means:
- Network: 10.0.0.0
- Subnet mask: 255.255.255.0
- Host range: 10.0.0.1 - 10.0.0.254
- Broadcast: 10.0.0.255
- Total hosts: 254
```

**Common CIDR blocks**:
| CIDR | Subnet Mask | Hosts |
|------|-------------|-------|
| /32 | 255.255.255.255 | 1 |
| /28 | 255.255.255.240 | 14 |
| /24 | 255.255.255.0 | 254 |
| /20 | 255.255.240.0 | 4,094 |
| /16 | 255.255.0.0 | 65,534 |
| /8 | 255.0.0.0 | 16,777,214 |

### Subnetting Example

Given: 10.0.0.0/16, create 4 equal subnets.

```
Original: 10.0.0.0/16 (65,534 hosts)

Need 4 subnets → Borrow 2 bits → /18

Subnet 1: 10.0.0.0/18    (10.0.0.1   - 10.0.63.254)
Subnet 2: 10.0.64.0/18   (10.0.64.1  - 10.0.127.254)
Subnet 3: 10.0.128.0/18  (10.0.128.1 - 10.0.191.254)
Subnet 4: 10.0.192.0/18  (10.0.192.1 - 10.0.255.254)
```

### IPv6 Addresses

128-bit addresses written in hexadecimal.

```
Full:       2001:0db8:0000:0000:0000:0000:0000:0001
Shortened:  2001:db8::1

Link-local: fe80::/10
Global unicast: 2000::/3
```

---

## 7.3 TCP (Transmission Control Protocol)

Connection-oriented, reliable transport protocol.

### TCP Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬───────┬─┬─┬─┬─┬─┬─┬───────────────────────────────────┤
│  Data │       │U│A│P│R│S│F│                                   │
│ Offset│Reserve│R│C│S│S│Y│I│            Window                 │
│       │       │G│K│H│T│N│N│                                   │
├───────┴───────┴─┴─┴─┴─┴─┴─┼───────────────────────────────────┤
│         Checksum          │         Urgent Pointer            │
├───────────────────────────┴───────────────────────────────────┤
│                    Options (if any)                           │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│                          Data                                 │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### TCP Flags

| Flag | Name | Purpose |
|------|------|---------|
| SYN | Synchronize | Initiate connection |
| ACK | Acknowledge | Acknowledge received data |
| FIN | Finish | Close connection |
| RST | Reset | Abort connection |
| PSH | Push | Send data immediately |
| URG | Urgent | Urgent data present |

### TCP Three-Way Handshake

```
Client                              Server
  │                                    │
  │ ──────── SYN (seq=100) ─────────▶ │
  │                                    │
  │ ◀─── SYN-ACK (seq=300,ack=101) ── │
  │                                    │
  │ ──────── ACK (ack=301) ─────────▶ │
  │                                    │
  │        Connection Established      │
```

**Step-by-step**:
1. **SYN**: Client sends SYN with initial sequence number (ISN)
2. **SYN-ACK**: Server responds with its ISN and ACK of client's ISN+1
3. **ACK**: Client acknowledges server's ISN+1

### TCP Four-Way Termination

```
Client                              Server
  │                                    │
  │ ──────── FIN (seq=x) ──────────▶  │
  │                                    │
  │ ◀──────── ACK (ack=x+1) ────────  │
  │                                    │
  │ ◀──────── FIN (seq=y) ──────────  │
  │                                    │
  │ ──────── ACK (ack=y+1) ─────────▶ │
  │                                    │
  │        Connection Closed           │
```

### TCP Connection States

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCP State Diagram                             │
│                                                                  │
│    CLOSED ──(client sends SYN)──▶ SYN_SENT                      │
│       │                              │                           │
│       │                       (recv SYN-ACK,                     │
│   (server                      send ACK)                         │
│    listens)                          │                           │
│       │                              ▼                           │
│       ▼                         ESTABLISHED ◀────────────┐      │
│    LISTEN                            │                    │      │
│       │                        (send FIN)                 │      │
│   (recv SYN,                         │                    │      │
│    send SYN-ACK)                     ▼                    │      │
│       │                          FIN_WAIT_1         (recv SYN-ACK│
│       ▼                              │               send ACK)   │
│    SYN_RCVD                    (recv ACK)                 │      │
│       │                              │                    │      │
│   (recv ACK)                         ▼                    │      │
│       │                          FIN_WAIT_2               │      │
│       └──────────────────────────────┼────────────────────┘      │
│                                (recv FIN,                        │
│                                 send ACK)                        │
│                                      │                           │
│                                      ▼                           │
│                                  TIME_WAIT                       │
│                                      │                           │
│                               (2*MSL timeout)                    │
│                                      │                           │
│                                      ▼                           │
│                                   CLOSED                         │
└─────────────────────────────────────────────────────────────────┘
```

### TCP Flow Control

**Window Size**: Receiver advertises how much data it can accept.

```
Sender                                Receiver
  │                                      │
  │ ──── Data (seq=1, 1000 bytes) ────▶ │
  │                                      │
  │ ◀──── ACK (ack=1001, win=5000) ──── │  "I can accept 5000 more bytes"
  │                                      │
  │ ──── Data (seq=1001, 5000 bytes) ──▶│
  │                                      │
  │ ◀──── ACK (ack=6001, win=0) ─────── │  "Buffer full, stop sending!"
  │                                      │
  │         (Sender waits)               │
  │                                      │
  │ ◀──── ACK (ack=6001, win=10000) ─── │  "Buffer cleared, continue"
```

### TCP Congestion Control

**Algorithms**: Slow start, congestion avoidance, fast retransmit, fast recovery.

```
cwnd = Congestion Window (sender-side limit)
ssthresh = Slow Start Threshold

Slow Start: cwnd doubles each RTT (exponential growth)
Congestion Avoidance: cwnd increases by 1 MSS each RTT (linear growth)

On packet loss:
  - ssthresh = cwnd / 2
  - cwnd = 1 (for timeout) or cwnd/2 (for duplicate ACKs)
```

---

## 7.4 UDP (User Datagram Protocol)

Connectionless, unreliable transport protocol.

### UDP Header

```
 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|     Source      |   Destination   |
|      Port       |      Port       |
+--------+--------+--------+--------+
|                 |                 |
|     Length      |    Checksum     |
+--------+--------+--------+--------+
|                                   |
|              Data                 |
|                                   |
+-----------------------------------+
```

### TCP vs UDP

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Ordered | Unordered |
| Flow control | Yes | No |
| Overhead | Higher (20+ bytes header) | Lower (8 bytes) |
| Speed | Slower (handshake, ACKs) | Faster |
| Use cases | Web, email, file transfer | DNS, streaming, gaming |

---

## 7.5 ICMP (Internet Control Message Protocol)

Used for network diagnostics and error reporting.

### Common ICMP Messages

| Type | Code | Description |
|------|------|-------------|
| 0 | 0 | Echo Reply (ping response) |
| 3 | 0 | Destination Network Unreachable |
| 3 | 1 | Destination Host Unreachable |
| 3 | 3 | Destination Port Unreachable |
| 3 | 4 | Fragmentation Needed |
| 8 | 0 | Echo Request (ping) |
| 11 | 0 | TTL Exceeded (traceroute) |

### Ping

```bash
# Basic ping
ping -c 4 google.com

# Output analysis
64 bytes from 142.250.185.46: icmp_seq=1 ttl=117 time=10.2 ms
│          │                    │         │      │
│          │                    │         │      └── Round-trip time
│          │                    │         └── Time to Live (hops remaining)
│          │                    └── Sequence number
│          └── Response from IP
└── Packet size
```

### Traceroute

```bash
# Show path to destination
traceroute google.com

# Output
 1  192.168.1.1 (192.168.1.1)  1.234 ms  1.123 ms  1.456 ms
 2  10.0.0.1 (10.0.0.1)  5.678 ms  5.432 ms  5.789 ms
 3  * * *                # No response (firewall blocking ICMP)
 4  142.250.185.46  10.123 ms  10.456 ms  10.789 ms
```

**How traceroute works**:
1. Send packets with TTL=1, 2, 3, etc.
2. Each router decrements TTL
3. When TTL=0, router sends ICMP "Time Exceeded"
4. This reveals each hop along the path

---

## 7.6 ARP (Address Resolution Protocol)

Maps IP addresses to MAC addresses.

### ARP Process

```
Host A (10.0.0.1)                    Host B (10.0.0.2)
MAC: AA:AA:AA:AA:AA:AA               MAC: BB:BB:BB:BB:BB:BB

1. Host A wants to send to 10.0.0.2

2. Host A checks ARP cache (not found)

3. Host A broadcasts ARP Request:
   "Who has 10.0.0.2? Tell 10.0.0.1"

4. Host B responds with ARP Reply:
   "10.0.0.2 is at BB:BB:BB:BB:BB:BB"

5. Host A caches the mapping and sends packet
```

### View ARP Cache

```bash
# Linux
ip neighbor show
# or
arp -a

# Output
10.0.0.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

---

## 7.7 NAT (Network Address Translation)

Translates between private and public IP addresses.

### NAT Types

**Static NAT**: One-to-one mapping
```
Internal 10.0.0.1 ↔ External 203.0.113.1
```

**Dynamic NAT**: Pool of public IPs
```
Internal 10.0.0.1 → External 203.0.113.1 (from pool)
Internal 10.0.0.2 → External 203.0.113.2 (from pool)
```

**PAT (Port Address Translation)**: Many-to-one using ports
```
10.0.0.1:50000 → 203.0.113.1:60000
10.0.0.2:50001 → 203.0.113.1:60001
10.0.0.3:50002 → 203.0.113.1:60002
                  ↑
                Same public IP, different ports
```

### NAT Process

```
┌────────────────────┐                    ┌─────────────────────┐
│   Private Network  │                    │      Internet       │
│                    │                    │                     │
│  ┌──────────────┐ │    ┌─────────┐     │  ┌───────────────┐  │
│  │ 10.0.0.5     │─┼───▶│   NAT   │─────┼─▶│  Web Server   │  │
│  │ src:10.0.0.5 │ │    │ Router  │     │  │  93.184.216.34│  │
│  │ dst:93.x.x.x │ │    │         │     │  └───────────────┘  │
│  └──────────────┘ │    │ Rewrite │     │                     │
│                    │    │ src to  │     │                     │
│                    │    │203.0.113.1    │                     │
│                    │    └─────────┘     │                     │
└────────────────────┘                    └─────────────────────┘

Outbound packet:
  Before NAT: src=10.0.0.5:45000, dst=93.184.216.34:80
  After NAT:  src=203.0.113.1:60000, dst=93.184.216.34:80

Return packet:
  Before NAT: src=93.184.216.34:80, dst=203.0.113.1:60000
  After NAT:  src=93.184.216.34:80, dst=10.0.0.5:45000
```

---

## 7.8 Routing

### How Routing Works

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  Host   │────▶│ Router1 │────▶│ Router2 │────▶│  Host   │
│10.0.0.5 │     │         │     │         │     │172.16.0.5
└─────────┘     └─────────┘     └─────────┘     └─────────┘

Routing Decision at Router1:
1. Receive packet for 172.16.0.5
2. Check routing table
3. Find matching route: 172.16.0.0/16 via Router2
4. Forward packet to Router2
```

### Routing Table

```bash
# View routing table
ip route show
# or
netstat -rn

# Output
Destination     Gateway         Genmask         Flags   Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG      eth0
10.0.0.0        0.0.0.0         255.0.0.0       U       eth1
192.168.1.0     0.0.0.0         255.255.255.0   U       eth0
```

### Route Selection

Routes are selected based on:
1. **Longest prefix match**: /24 beats /16 beats /8
2. **Administrative distance**: Static < OSPF < BGP
3. **Metric**: Lower is preferred

---

## 7.9 Network Troubleshooting Tools

### Essential Commands

```bash
# Test connectivity
ping -c 4 host

# Trace path
traceroute host
mtr host  # Combines ping and traceroute

# Check open ports
netstat -tuln
ss -tuln

# Check connections
netstat -an | grep ESTABLISHED
ss -s  # Summary

# DNS lookup
dig example.com
nslookup example.com
host example.com

# HTTP debugging
curl -v https://example.com
curl -I https://example.com  # Headers only

# Packet capture
tcpdump -i eth0 port 80
tcpdump -i any host 10.0.0.5

# Port connectivity
nc -zv host port
telnet host port
```

### Systematic Debugging Approach

```
1. Check if it's DNS
   dig example.com
   ↓
2. Check if it's routing
   traceroute example.com
   ↓
3. Check if it's firewall
   nc -zv example.com 443
   ↓
4. Check if it's the service
   curl -v https://example.com
   ↓
5. Check application logs
```

---

## 7.10 Common Network Issues

### Can't Connect to Remote Host

```
Symptoms: Connection refused / timeout

Debug steps:
1. Is DNS resolving?
   dig example.com

2. Is host reachable?
   ping example.com

3. Is the path clear?
   traceroute example.com

4. Is the port open?
   nc -zv example.com 443

5. Is firewall blocking?
   Check local and remote firewall rules

6. Is the service running?
   Check service status on remote host
```

### Slow Network Performance

```
Symptoms: High latency, packet loss, low throughput

Debug steps:
1. Check latency
   ping -c 100 host | grep avg

2. Check packet loss
   mtr host

3. Check bandwidth
   iperf3 -c host

4. Check for congestion
   ss -i  # Check retransmits

5. Check MTU issues
   ping -M do -s 1472 host
```

### Intermittent Connectivity

```
Symptoms: Works sometimes, fails sometimes

Possible causes:
1. DNS caching issues
   - Check TTL
   - Try different DNS servers

2. Load balancer issues
   - Multiple backends with different health

3. Network flapping
   - Check interface status
   - Check for errors: ip -s link

4. Resource exhaustion
   - Check connection limits
   - Check port exhaustion
```

---

## Chapter 8 Review Questions

1. Explain the TCP three-way handshake and why each step is necessary.

2. A TCP connection is stuck in TIME_WAIT state. What does this mean and is it a problem?

3. When would you use UDP instead of TCP?

4. A customer reports intermittent connectivity. How would you diagnose?

5. Explain how NAT works and why it's used.

6. What's the difference between ping and traceroute, and when would you use each?

---

## Chapter 8 Hands-On Exercises

### Exercise 7.1: TCP Analysis
1. Use tcpdump to capture a TCP handshake
2. Identify SYN, SYN-ACK, and ACK packets
3. Calculate the round-trip time

### Exercise 7.2: Network Debugging
1. Diagnose a connectivity issue step by step
2. Use ping, traceroute, and netcat
3. Document your findings

### Exercise 7.3: Subnet Calculation
1. Given 10.0.0.0/8, create 16 equal subnets
2. Calculate the IP range for each subnet
3. Verify using an online subnet calculator

---

## Key Takeaways

1. **Know the OSI model cold** - Essential for troubleshooting
2. **TCP is connection-oriented** - Understand handshake and states
3. **UDP is fire-and-forget** - Good for speed, bad for reliability
4. **Subnetting is fundamental** - Practice CIDR calculations
5. **Use the right tool** - ping for connectivity, traceroute for path, tcpdump for packets
6. **Systematic debugging** - Layer by layer, don't skip steps

---

[Next Chapter: DNS, HTTP & Web Protocols →](./08-dns-http.md)
