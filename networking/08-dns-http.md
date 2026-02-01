# Chapter 8: DNS, HTTP & Web Protocols

## 8.1 DNS (Domain Name System)

DNS is the phonebook of the internet - translating human-readable names to IP addresses.

### DNS Hierarchy

```
                           . (root)
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
          com                org                net
           │                  │                  │
     ┌─────┴─────┐      ┌────┴────┐             │
     │           │      │         │             │
  google    example  wikipedia  apache       cloudflare
     │           │      │
   www       mail    en
```

### DNS Components

**Root Servers**: 13 root server clusters (a.root-servers.net through m.root-servers.net)

**TLD Servers**: Manage top-level domains (.com, .org, .net, country codes)

**Authoritative Servers**: Hold actual DNS records for a domain

**Recursive Resolvers**: Query on behalf of clients, cache results

### DNS Resolution Process

```
User types: www.example.com

1. Browser cache → Not found
2. OS cache → Not found
3. Query recursive resolver (e.g., 8.8.8.8)

Recursive resolver:
4. Query root server → "Try .com TLD server"
5. Query .com TLD → "Try ns1.example.com"
6. Query ns1.example.com → "93.184.216.34"
7. Return result to client
8. Client caches result

Total: ~100ms (if not cached)
Cached: ~1ms
```

### DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | IPv4 address | example.com → 93.184.216.34 |
| **AAAA** | IPv6 address | example.com → 2606:2800:220:1:... |
| **CNAME** | Alias to another name | www.example.com → example.com |
| **MX** | Mail server | example.com → mail.example.com (priority 10) |
| **TXT** | Text (SPF, DKIM, verification) | "v=spf1 include:_spf.google.com ~all" |
| **NS** | Name server | example.com NS ns1.example.com |
| **SOA** | Start of Authority | Zone metadata |
| **SRV** | Service location | _http._tcp.example.com |
| **PTR** | Reverse DNS | 34.216.184.93.in-addr.arpa → example.com |
| **CAA** | Certificate Authority Authorization | Allowed CAs |

### DNS Query Tools

```bash
# Basic lookup
dig example.com

# Specific record type
dig example.com A
dig example.com MX
dig example.com TXT

# Use specific DNS server
dig @8.8.8.8 example.com

# Trace resolution path
dig +trace example.com

# Short answer only
dig +short example.com

# Reverse lookup
dig -x 93.184.216.34

# All records
dig example.com ANY
```

### dig Output Explained

```bash
$ dig example.com

; <<>> DiG 9.16.1 <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            86400   IN      A       93.184.216.34
                        │               │       │
                        │               │       └── The IP address
                        │               └── Record type
                        └── TTL in seconds (24 hours)

;; Query time: 23 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Mon Jan 15 10:00:00 UTC 2024
;; MSG SIZE  rcvd: 56
```

### TTL (Time To Live)

How long DNS records are cached:

```
High TTL (86400 = 24 hours):
  + Less DNS queries
  + Faster resolution
  - Slower propagation of changes

Low TTL (300 = 5 minutes):
  + Faster propagation
  + Better for failover
  - More DNS queries
  - Higher latency
```

### Common DNS Issues

**NXDOMAIN**: Domain doesn't exist
```bash
$ dig nonexistent.example.com
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN
```

**SERVFAIL**: Server failure (timeout, misconfiguration)
```bash
$ dig example.com
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL
```

**DNS Propagation**: Changes take time to spread
- Wait for TTL to expire
- Clear local cache: `sudo systemd-resolve --flush-caches`

**Split-horizon DNS**: Different answers for internal vs external
- Common in enterprises
- Check which DNS server you're querying

---

## 8.2 HTTP/HTTPS

### HTTP Request/Response

```
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP Request                              │
├─────────────────────────────────────────────────────────────────┤
│ GET /api/users HTTP/1.1                    ← Request line       │
│ Host: api.example.com                      ← Required header    │
│ User-Agent: curl/7.68.0                    ← Client info        │
│ Accept: application/json                   ← Desired format     │
│ Authorization: Bearer eyJhbGc...           ← Auth token         │
│                                            ← Empty line         │
│ {"name": "John"}                           ← Body (optional)    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        HTTP Response                             │
├─────────────────────────────────────────────────────────────────┤
│ HTTP/1.1 200 OK                            ← Status line        │
│ Content-Type: application/json             ← Response format    │
│ Content-Length: 27                         ← Body size          │
│ Cache-Control: no-cache                    ← Caching directive  │
│                                            ← Empty line         │
│ {"id": 1, "name": "John"}                  ← Response body      │
└─────────────────────────────────────────────────────────────────┘
```

### HTTP Methods

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| **GET** | Retrieve resource | Yes | Yes |
| **POST** | Create resource | No | No |
| **PUT** | Replace resource | Yes | No |
| **PATCH** | Partial update | No | No |
| **DELETE** | Remove resource | Yes | No |
| **HEAD** | Get headers only | Yes | Yes |
| **OPTIONS** | Get allowed methods | Yes | Yes |

### HTTP Status Codes

**1xx - Informational**
```
100 Continue - Server received headers, send body
101 Switching Protocols - Upgrading to WebSocket
```

**2xx - Success**
```
200 OK - Request succeeded
201 Created - Resource created
204 No Content - Success, no body
```

**3xx - Redirection**
```
301 Moved Permanently - URL changed forever
302 Found - Temporary redirect
304 Not Modified - Use cached version
307 Temporary Redirect - Keep method on redirect
308 Permanent Redirect - Keep method, permanent
```

**4xx - Client Error**
```
400 Bad Request - Invalid syntax
401 Unauthorized - Authentication required
403 Forbidden - Authenticated but not allowed
404 Not Found - Resource doesn't exist
405 Method Not Allowed - Wrong HTTP method
408 Request Timeout - Client too slow
429 Too Many Requests - Rate limited
```

**5xx - Server Error**
```
500 Internal Server Error - Generic server error
502 Bad Gateway - Upstream server error
503 Service Unavailable - Server overloaded/maintenance
504 Gateway Timeout - Upstream timeout
```

### Important HTTP Headers

**Request Headers**
```
Host: api.example.com              # Required in HTTP/1.1
User-Agent: Mozilla/5.0            # Client identification
Accept: application/json           # Desired response format
Accept-Language: en-US             # Preferred language
Accept-Encoding: gzip, deflate     # Compression support
Authorization: Bearer TOKEN        # Authentication
Cookie: session=abc123             # Session data
Content-Type: application/json     # Request body format
Content-Length: 100                # Body size
Cache-Control: no-cache            # Caching directive
If-None-Match: "etag123"           # Conditional request
```

**Response Headers**
```
Content-Type: application/json     # Response format
Content-Length: 256                # Body size
Content-Encoding: gzip             # Compression used
Cache-Control: max-age=3600        # Caching rules
ETag: "etag123"                    # Resource version
Last-Modified: Mon, 15 Jan 2024    # When changed
Set-Cookie: session=xyz            # Set cookie
Location: /new-path                # Redirect target
Access-Control-Allow-Origin: *     # CORS header
X-Request-ID: abc-123              # Tracking ID
```

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No (1 request/connection) | Yes | Yes |
| Header compression | No | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Connection setup | TCP handshake | TCP + TLS | 0-RTT possible |
| Head-of-line blocking | Yes | Reduced | No |

### HTTP/2 Multiplexing

```
HTTP/1.1 (6 parallel connections, one request each):
───────────────────────────────────────────────────
Conn 1: [Request 1]─────────[Response 1]
Conn 2: [Request 2]─────────[Response 2]
Conn 3: [Request 3]─────────[Response 3]
Conn 4: [Request 4]─────────[Response 4]
Conn 5: [Request 5]─────────[Response 5]
Conn 6: [Request 6]─────────[Response 6]

HTTP/2 (1 connection, multiplexed):
───────────────────────────────────────────────────
Conn 1: [R1][R2][R3][R4][R5][R6][Resp1][Resp2]...
        └── All on single connection, interleaved
```

---

## 8.3 TLS/SSL

### TLS Handshake (TLS 1.3)

```
Client                                          Server
   │                                               │
   │──── ClientHello ─────────────────────────────▶│
   │     (supported ciphers, key share)           │
   │                                               │
   │◀─── ServerHello ─────────────────────────────│
   │     (selected cipher, key share)             │
   │◀─── EncryptedExtensions ─────────────────────│
   │◀─── Certificate ─────────────────────────────│
   │◀─── CertificateVerify ───────────────────────│
   │◀─── Finished ────────────────────────────────│
   │                                               │
   │──── Finished ────────────────────────────────▶│
   │                                               │
   │          ═══ Encrypted Communication ═══      │
```

### Certificate Chain

```
Root CA Certificate (trusted by browser)
         │
         ▼
Intermediate CA Certificate
         │
         ▼
Server Certificate (example.com)
```

### Debugging TLS Issues

```bash
# Check certificate details
openssl s_client -connect example.com:443 -servername example.com

# Show certificate chain
openssl s_client -connect example.com:443 -showcerts

# Check certificate expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Verify certificate
openssl verify -CAfile ca-bundle.crt certificate.crt

# Check supported protocols
nmap --script ssl-enum-ciphers -p 443 example.com
```

### Common TLS Errors

```
SSL_ERROR_HANDSHAKE_FAILURE
  - Cipher suite mismatch
  - Protocol version incompatible

CERTIFICATE_VERIFY_FAILED
  - Certificate expired
  - Self-signed (not trusted)
  - Wrong domain (CN mismatch)

SSL_ERROR_RX_RECORD_TOO_LONG
  - Connecting to HTTP port with HTTPS
  - Server not configured for TLS
```

---

## 8.4 WebSockets

Full-duplex communication over single TCP connection.

### WebSocket Handshake

```
Client Request:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

── Connection upgraded, now WebSocket frames ──
```

### WebSocket vs HTTP

| Feature | HTTP | WebSocket |
|---------|------|-----------|
| Connection | New connection per request | Persistent |
| Direction | Request-response | Full duplex |
| Overhead | Headers on every request | Low overhead frames |
| Use case | Traditional web | Real-time apps |

---

## 8.5 REST API Best Practices

### URL Structure

```
Good:
GET    /users              # List users
GET    /users/123          # Get user 123
POST   /users              # Create user
PUT    /users/123          # Update user 123
DELETE /users/123          # Delete user 123
GET    /users/123/orders   # User's orders

Bad:
GET    /getUser?id=123     # Don't use verbs in URL
POST   /createUser         # Method implies action
GET    /users/delete/123   # Don't put action in URL
```

### Response Format

```json
// Success response
{
  "data": {
    "id": 123,
    "name": "John",
    "email": "john@example.com"
  },
  "meta": {
    "request_id": "abc-123"
  }
}

// Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      {
        "field": "email",
        "message": "This field is required"
      }
    ]
  },
  "meta": {
    "request_id": "abc-124"
  }
}

// List response with pagination
{
  "data": [...],
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 20,
    "next_page": "/users?page=2"
  }
}
```

---

## 8.6 Debugging HTTP Issues

### Using curl

```bash
# Basic request
curl https://api.example.com/users

# Verbose output (see headers)
curl -v https://api.example.com/users

# Show only response headers
curl -I https://api.example.com/users

# POST with JSON body
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John"}'

# With authentication
curl -H "Authorization: Bearer TOKEN" \
  https://api.example.com/users

# Follow redirects
curl -L https://example.com/redirect

# Show timing information
curl -w "@curl-format.txt" -o /dev/null -s https://example.com
# Where curl-format.txt contains:
#   time_namelookup: %{time_namelookup}\n
#   time_connect: %{time_connect}\n
#   time_starttransfer: %{time_starttransfer}\n
#   time_total: %{time_total}\n

# Ignore certificate errors (don't use in production!)
curl -k https://self-signed.example.com
```

### Common HTTP Debugging Scenarios

**502 Bad Gateway**
```
Cause: Upstream server error

Debug:
1. Check upstream server is running
2. Check upstream server logs
3. Check load balancer health checks
4. Check network connectivity to upstream
5. Check timeout settings
```

**503 Service Unavailable**
```
Cause: Server overloaded or in maintenance

Debug:
1. Check server resources (CPU, memory)
2. Check if server is in maintenance mode
3. Check connection pool exhaustion
4. Check for circuit breaker activation
```

**504 Gateway Timeout**
```
Cause: Upstream took too long

Debug:
1. Check upstream response time
2. Increase timeout settings
3. Check for slow database queries
4. Check for external API delays
```

**SSL/TLS Errors**
```
Debug:
1. openssl s_client -connect host:443
2. Check certificate validity
3. Check certificate chain
4. Check for SNI requirements
```

---

## 8.7 CORS (Cross-Origin Resource Sharing)

### How CORS Works

```
Browser on https://app.example.com
wants to call https://api.example.com

1. Preflight request (for non-simple requests):
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type

2. Server response:
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400

3. Actual request (if preflight succeeded):
POST /api/users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Content-Type: application/json

4. Server response:
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
```

### CORS Headers

```
Access-Control-Allow-Origin: https://example.com  # or *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Access-Control-Expose-Headers: X-Custom-Header
```

---

## Chapter 8 Review Questions

1. Walk through the complete DNS resolution process for www.example.com.

2. What's the difference between HTTP 301 and 302 redirects? When would you use each?

3. A customer is getting 502 errors intermittently. How would you debug?

4. Explain the TLS handshake process.

5. What are the key differences between HTTP/1.1 and HTTP/2?

6. A customer's application is getting CORS errors. What do you check?

---

## Chapter 8 Hands-On Exercises

### Exercise 8.1: DNS Investigation
1. Use dig to trace DNS resolution for a domain
2. Find all record types for a domain
3. Compare results from different DNS servers

### Exercise 8.2: HTTP Debugging
1. Use curl to make various HTTP requests
2. Analyze response headers
3. Measure response times

### Exercise 8.3: TLS Analysis
1. Use openssl to examine a certificate
2. Check certificate chain
3. Verify certificate validity

---

## Key Takeaways

1. **DNS is hierarchical** - Root → TLD → Authoritative
2. **TTL matters** - Balance between performance and flexibility
3. **HTTP status codes tell stories** - Know the common ones
4. **Headers are important** - For caching, auth, content negotiation
5. **HTTP/2+ improves performance** - Multiplexing, compression
6. **TLS secures everything** - Understand the handshake

---

[Next Chapter: Linux Systems Administration →](./09-linux-systems.md)
