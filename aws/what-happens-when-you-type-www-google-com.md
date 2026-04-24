# What Happens When You Type www.google.com? 🌐

> **The #1 FAANG Interview Question for DevOps & Senior DevOps Engineers**  
> Complete end-to-end breakdown — from keystroke to rendered page — in under 200ms.

---

## 📊 By the Numbers

| Metric | Value |
|--------|-------|
| Total phases | 10 |
| Total time | < 200ms |
| Servers involved | 1,000+ |
| DNS servers queried | Up to 3 (Root → TLD → Auth) |
| TCP round trips | 1 (handshake) |
| TLS round trips | 1 (TLS 1.3) |

---

## 🗺️ The Big Picture

```
[You]──①DNS──▶[Recursive Resolver]──▶[Root NS]──▶[TLD NS]──▶[Google Auth NS]
  │                                                                    │
  │◀──────────────────── 142.250.80.46 ───────────────────────────────┘
  │
  ├──②TCP──▶[Anycast BGP → Nearest Google Edge]
  │
  ├──③TLS──▶[TLS 1.3 Handshake — 1 RTT]
  │
  ├──④HTTP GET /──▶[Maglev L4 LB]──▶[GFE L7 LB]──▶[WAF]
  │                                                    │
  │                               [Memcache]◀──────────┤
  │                               [Index Servers]◀─────┤
  │                               [Ad Auction]◀────────┘
  │
  ◀──⑤HTML 200 OK (brotli compressed ~50KB)
  │
  └──⑥Browser: Parse HTML → Fetch CSS/JS → Layout → Paint → Composite → 🖥️
```

---

## Phase 1 — Keyboard Input & URL Parsing (~0ms)

```
Keystroke → IRQ → OS Kernel → Browser Message Queue → URL Parser
```

**What the browser does:**

1. Detects Enter keydown event via OS interrupt handler
2. Checks if input is a URL or search query
3. `www.google.com` → recognized TLD `.com` → treat as URL
4. Prepends `https://` via **HSTS Preload List** (baked into browser binaries)
5. Parses canonical URL:

```
scheme   : https
host     : www.google.com
port     : 443  (implicit)
path     : /
```

> **DevOps tip:** Set `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` and submit to the HSTS preload list. Browsers will **never** attempt HTTP — even on first visit, with zero server involvement.

---

## Phase 2–3 — DNS Resolution (~1–120ms)

### Cache Lookup Order (fastest to slowest)

```
Browser DNS cache  →  OS DNS cache  →  /etc/hosts  →  Network
(microseconds)        (microseconds)    (microseconds)  (milliseconds)
```

### Full Resolution Path (cache miss)

```
Browser
  │── DNS query: "what is www.google.com?"
  ▼
Recursive Resolver (8.8.8.8 / 1.1.1.1 / ISP)
  │
  ├──▶ Root NS (13 logical clusters worldwide)
  │       "I don't know, ask .com TLD"
  │
  ├──▶ .com TLD NS (Verisign)
  │       "I don't know, ask Google's NS"
  │
  └──▶ Google Authoritative NS (ns1.google.com)
           "142.250.80.46, TTL=300"
  │
  ◀── Returns IP to browser, caches for 300 seconds
```

### DNS Record Types (Know These)

| Record | Purpose | DevOps Use |
|--------|---------|-----------|
| `A` | IPv4 → IP | Point domain to server/ALB |
| `AAAA` | IPv6 → IP | Dual-stack |
| `CNAME` | Alias to another name | CDN/ALB endpoints |
| `MX` | Mail servers | Email routing |
| `TXT` | Arbitrary text | SPF, DKIM, ACME challenges |
| `NS` | Nameservers | Delegation |
| `SOA` | Zone authority | TTL, serial number |

> **War story:** Low TTL (60s) = fast failover but high DNS load. High TTL (3600s) = slow propagation but lighter infrastructure. Google uses **TTL=300s**. Best practice: lower TTL 24hrs before a planned migration, restore afterward.

---

## Phase 4 — TCP Three-Way Handshake (~5–50ms)

```
Client                                    Server (:443)
  │                                           │
  │──── SYN (seq=1000) ──────────────────────▶│
  │                                           │
  │◀─── SYN-ACK (seq=5000, ack=1001) ─────────│
  │                                           │
  │──── ACK (seq=1001, ack=5001) ────────────▶│
  │                                           │
ESTABLISHED                             ESTABLISHED
  │
  │ Cost: 1 full RTT before any data flows
```

**Key concepts:**

- **ISN (Initial Sequence Number):** Randomly generated, prevents packet injection attacks
- **Window size:** Flow control — how much data the receiver can buffer
- **Slow start:** New connections start with `cwnd=10 MSS (~14KB)`, ramps up exponentially
- **TCP Fast Open (TFO):** Data in SYN packet on reconnect — saves 1 RTT

**Linux tuning for high-traffic servers:**

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
```

---

## Phase 5 — TLS 1.3 Handshake (~15ms, 1 RTT)

```
Client                                         Server
  │                                               │
  │── ClientHello ───────────────────────────────▶│
  │   (supported_versions, key_share, SNI)        │
  │                                               │
  │◀── ServerHello ───────────────────────────────│
  │◀── {Certificate} ─────────────────────────────│  ← encrypted!
  │◀── {CertificateVerify} ───────────────────────│
  │◀── {Finished} ────────────────────────────────│
  │                                               │
  │── {Finished} ────────────────────────────────▶│
  │── [Application Data] ────────────────────────▶│  ← HTTP request!
  
TLS 1.3 vs TLS 1.2:
  TLS 1.2 = 2 RTT     |     TLS 1.3 = 1 RTT
  0-RTT resumption available (with replay protection)
```

**Key concepts:**

- **SNI (Server Name Indication):** Hostname sent in ClientHello *before* encryption → enables virtual hosting on shared IPs
- **ECDHE:** Ephemeral key exchange → **forward secrecy** (past sessions safe if private key compromised)
- **Certificate chain:** Leaf cert → Intermediate CA (GTS CA 1C3) → Root CA
- **OCSP Stapling:** Server includes fresh revocation proof in handshake → eliminates separate revocation round-trip

---

## Phase 6 — Anycast Routing & Load Balancers

```
142.250.80.46 announced simultaneously by Google DCs worldwide via BGP:

     US East DC  ──┐
     EU West DC  ──┼── Same IP, BGP routes to NEAREST
     Asia DC     ──┘

Inside the nearest Google DC:
  Packet → Border Router (BGP)
         → Maglev (L4 Load Balancer, ECMP consistent hashing)
         → GFE (Google Frontend, L7, HTTP/2 termination)
```

**Maglev** — Google's custom L4 LB:
- Hashes `(src_ip, src_port, dst_ip, dst_port, protocol)` → consistent backend
- Stateless — any Maglev instance can handle any packet
- Handles millions of packets/second on commodity hardware

**Your equivalent in AWS/GCP/Azure:**

| Google | AWS | GCP | Azure |
|--------|-----|-----|-------|
| Anycast | Route 53 Latency | Cloud LB (global) | Traffic Manager |
| Maglev | NLB | Cloud LB (L4) | Azure LB |
| GFE | ALB | Cloud LB (L7) | Application Gateway |
| Cloud Armor | WAFv2 + Shield | Cloud Armor | Azure WAF |

---

## Phase 7 — HTTP Request Processing

```
HTTP/2 HEADERS frame (binary):

:method:          GET
:scheme:          https
:authority:       www.google.com
:path:            /
accept-encoding:  gzip, deflate, br
user-agent:       Mozilla/5.0 ...
cookie:           SID=...; HSID=...
```

**HTTP/2 vs HTTP/1.1:**

| Feature | HTTP/1.1 | HTTP/2 |
|---------|---------|--------|
| Format | Text | Binary frames |
| Concurrency | 1 req/connection | Multiplexed streams |
| Headers | Repeated every req | HPACK compressed |
| Server push | ❌ | ✅ |
| Head-of-line blocking | ❌ (per conn) | ✅ (per stream) |

**HTTP/3 (QUIC):** UDP-based, built-in TLS 1.3, eliminates TCP HOL blocking. Google has used this internally since 2015. `alt-svc: h3=":443"` header advertises HTTP/3 support.

---

## Phase 8 — Backend Processing

```
GFE → Search Frontend
         │
         ├─ Memcache check ──▶ HIT? Return immediately (microseconds)
         │                     MISS? Continue...
         │
         ├─ Index Servers (parallel fan-out to thousands of shards)
         │  Inverted index: "google" → [ranked doc list]
         │
         ├─ PageRank / ML Ranking models
         │
         ├─ Ad Auction (parallel, ~10ms budget)
         │
         └─ HTML generation → brotli compress → respond
```

**Google's data stack mapped to common tools:**

| Use Case | Google Internal | AWS Equivalent |
|----------|----------------|----------------|
| Distributed KV | Bigtable | DynamoDB |
| Global SQL | Spanner | Aurora Global |
| Cache | Memcache | ElastiCache |
| Object Storage | Colossus (GFS2) | S3 |
| Messaging | Pub/Sub | SQS / Kafka |
| Service Mesh | Stubby/gRPC | Istio + Envoy |
| Distributed Tracing | Dapper | Jaeger / X-Ray |

---

## Phase 9 — HTTP Response

```
HTTP/2 200 OK

content-encoding:            br                    ← brotli
cache-control:               private, max-age=0    ← personalized
strict-transport-security:   max-age=31536000      ← HSTS
content-security-policy:     object-src 'none'...  ← XSS protection
x-frame-options:             SAMEORIGIN            ← clickjacking
alt-svc:                     h3=":443"; ma=2592000 ← HTTP/3 available
server:                      gws                   ← Google Web Server

Body: ~50KB brotli → ~250KB HTML
```

**Latency breakdown (NYC → Google):**

```
DNS lookup (cache miss)     : ~20ms
TCP handshake               : ~15ms
TLS 1.3 handshake           : ~15ms
HTTP request → first byte   : ~15ms
Full HTML body              : ~20ms
                              ──────
TTFB                        : ~65ms
Total (with rendering)      : ~150–200ms
```

---

## Phase 10 — Browser Rendering Pipeline

```
HTML bytes
    │
    ▼
HTML Parser ──▶ DOM Tree ──┐
    │                      ├──▶ Render Tree ──▶ Layout ──▶ Paint ──▶ Composite ──▶ 🖥️
CSS Parser  ──▶ CSSOM    ──┘
    │
    └── JS Engine (V8) executes scripts, modifies DOM
```

**Core Web Vitals (measure & monitor these):**

| Metric | What it measures | Good threshold |
|--------|-----------------|----------------|
| **LCP** (Largest Contentful Paint) | Loading performance | < 2.5s |
| **INP** (Interaction to Next Paint) | Responsiveness | < 200ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | < 0.1 |
| **TTFB** (Time to First Byte) | Server response | < 800ms |
| **FCP** (First Contentful Paint) | First render | < 1.8s |

> **DevOps owns TTFB.** Everything upstream of the browser — server response time, CDN placement, compression, cache-hit ratio — is DevOps territory and directly impacts Core Web Vitals and SEO rankings.

---

## ⏱️ Complete Timeline

```
T+0ms      Enter pressed → URL parsed → HSTS checked
T+0–1ms    Cache hit: browser/OS DNS cache
T+1–120ms  Cache miss: Full DNS resolution (Root → TLD → Auth NS)
T+5–50ms   TCP 3-way handshake (1 RTT to nearest edge)
T+20–65ms  TLS 1.3 handshake (1 additional RTT)
T+35–80ms  HTTP/2 GET / sent → through LB → WAF → GFE
T+50–100ms Backend: Memcache check → Index → Ranking → HTML
T+65–120ms First byte received (TTFB) — parsing begins immediately
T+80–150ms CSS/JS/fonts fetched via HTTP/2 multiplexing from CDN
T+150–200ms Layout → Paint → Composite → Page visible ✅
```

---

## 🎯 FAANG Interview Cheat Sheet

**Don't just list steps. Connect each to operational concerns.**

| Phase | Say This | Bonus Point |
|-------|---------|-------------|
| DNS | TTL tuning, recursive vs authoritative | Route 53 failover, low TTL before migrations |
| TCP | 3-way handshake, slow start | Connection pooling, sysctl tuning, SYN flood defense |
| TLS | TLS 1.3, SNI, forward secrecy | Cert rotation automation, mTLS in service mesh |
| Routing | Anycast, L4 vs L7, consistent hashing | Health checks, connection draining, sticky sessions |
| HTTP | HTTP/2 multiplexing, brotli compression | TTFB SLOs, CDN cache-hit ratio, Core Web Vitals |
| Backend | Cache layers, fan-out, circuit breakers | Retry budgets, bulkheads, timeout pyramids |

### Failure modes to know:
- **DNS:** Cache poisoning, TTL too high during incident, split-brain DNS
- **TCP:** SYN flood DDoS, `TIME_WAIT` exhaustion, RST injection
- **TLS:** Certificate expiry (alert at 30 days!), weak cipher suites, BEAST/POODLE on TLS 1.0/1.1
- **LB:** Thundering herd on restart, connection draining too short, health check misconfiguration
- **Backend:** Cache stampede on cold start, N+1 queries, cascading failures without circuit breakers

### Diagnostic commands:
```bash
# DNS trace
dig +trace www.google.com

# TCP handshake capture
tcpdump -i eth0 'host 142.250.80.46 and tcp'

# TLS inspection
openssl s_client -connect www.google.com:443 -tls1_3

# HTTP timing breakdown
curl -w "@curl-format.txt" -o /dev/null -s https://www.google.com
# Shows: time_namelookup, time_connect, time_appconnect, time_starttransfer, time_total

# Kubernetes: check DNS resolution from pod
kubectl exec -it <pod> -- nslookup www.google.com
kubectl exec -it <pod> -- curl -v https://www.google.com
```

---


> **Best home:** `aws/` — covers EC2/ALB/Route 53/WAF decisions, Anycast routing, and cloud networking concepts directly applicable to AWS infrastructure design.

---

*Part of the [devops-field-guide](../) series — practical, interview-ready references for working DevOps engineers.*

