# Chapter 06: (Ab)using CDNs — The Architecture of the Edge

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 6 / Week 6 of 8  
> **Core Theme**: *"Someone else's computers, closer to your users — and everything they can do while the traffic is passing through."*  
> **Key Protocols & Concepts**: Physics of latency (Speed of light in fiber, 4.9 ms/1000 km, Mumbai to Ashburn floor), The Mathis Equation & throughput degradation, Anycast BGP routing, Anatomy of a CDN PoP (Reverse proxy rented in 330+ cities), Consistent Hashing (Karger et al., 1997) vs. Modulo hashing, The CDN software landscape (Akamai, CloudFront, Fastly Varnish/H2O, Cloudflare Pingora/Oxy), Cache keys and default file-extension caching, The 8-state `cf-cache-status` state machine, Caching dynamic content (SSI, ESI, Ajax, Railgun, RFC 9842 Compression Dictionary Transport), Shared-cache security vulnerabilities (Web Cache Poisoning, Web Cache Deception), Transport acceleration (Edge BBR, Shared connection pooling, Tiered caching/Origin shields, Request collapsing), The Edge as a Security Perimeter (WAF normalization, Rate limiting algorithms, Volumetric DDoS absorption via Anycast), Edge API Gateways & Origin lockdown (mTLS, Authenticated Origin Pulls, Cloudflare Tunnels), Edge Compute via V8 Isolates (Cloudflare Workers), and 15 exam-grade MCQs.

---

## Table of Contents
1. [The Physical Reality of Latency: Why the Edge Exists](#1-the-physical-reality-of-latency-why-the-edge-exists)
   - [1.1 The Inviolable Speed-of-Light Budget in Optical Fiber](#11-the-inviolable-speed-of-light-budget-in-optical-fiber)
   - [1.2 The Mathis Equation: Why Distance Kills TCP Throughput](#12-the-mathis-equation-why-distance-kills-tcp-throughput)
   - [1.3 The Two Values of the Edge: Handshake Absorption vs. Cache Hits](#13-the-two-values-of-the-edge-handshake-absorption-vs-cache-hits)
2. [What a CDN Actually Is: Architecture & Routing](#2-what-a-cdn-actually-is-architecture--routing)
   - [2.1 A Reverse Proxy Rented in 300+ Cities](#21-a-reverse-proxy-rented-in-300-cities)
   - [2.2 Anycast BGP Routing: One IP Address, Hundreds of Data Centers](#22-anycast-bgp-routing-one-ip-address-hundreds-of-data-centers)
   - [2.3 The Airport Code Architecture (`colo` & `cf-ray`)](#23-the-airport-code-architecture-colo--cf-ray)
   - [2.4 The Founding Algorithm of CDNs: Consistent Hashing vs. Modulo Hashing](#24-the-founding-algorithm-of-cdns-consistent-hashing-vs-modulo-hashing)
   - [2.5 The Software Stack Inside the Major CDNs](#25-the-software-stack-inside-the-major-cdns)
3. [Edge Caching Mechanics: The Defaults & The State Machine](#3-edge-caching-mechanics-the-defaults--the-state-machine)
   - [3.1 Default File-Extension Rules: Why HTML is Never Cached by Default](#31-default-file-extension-rules-why-html-is-never-cached-by-default)
   - [3.2 The Cache Key Question: URL vs. Cookies & Headers](#32-the-cache-key-question-url-vs-cookies--headers)
   - [3.3 Deconstructing the 8-State `cf-cache-status` Machine](#33-deconstructing-the-8-state-cf-cache-status-machine)
4. [Caching the Uncacheable: Dynamic Content at the Edge](#4-caching-the-uncacheable-dynamic-content-at-the-edge)
   - [4.1 The Evolution: SSI $\rightarrow$ ESI $\rightarrow$ Ajax $\rightarrow$ Delta Compression](#41-the-evolution-ssi-rightarrow-esi-rightarrow-ajax-rightarrow-delta-compression)
   - [4.2 Edge-Side Includes (ESI): Assembling Personalized HTML at the PoP](#42-edge-side-includes-esi-assembling-personalized-html-at-the-pop)
   - [4.3 Cloudflare Railgun to RFC 9842: Compression Dictionary Transport](#43-cloudflare-railgun-to-rfc-9842-compression-dictionary-transport)
5. [Shared Caches as Attack Surfaces: Poisoning & Deception](#5-shared-caches-as-attack-surfaces-poisoning--deception)
   - [5.1 Web Cache Poisoning: Weaponizing Unkeyed Headers](#51-web-cache-poisoning-weaponizing-unkeyed-headers)
   - [5.2 Web Cache Deception: Stealing Private Account Data](#52-web-cache-deception-stealing-private-account-data)
6. [Accelerating Byte Delivery: Transport & Topology](#6-accelerating-byte-delivery-transport--topology)
   - [6.1 Edge BBR: Pacing TCP on the Long Haul](#61-edge-bbr-pacing-tcp-on-the-long-haul)
   - [6.2 Connection Pooling: Per-Worker Pools vs. Pingora's Shared Pool](#62-connection-pooling-per-worker-pools-vs-pingoras-shared-pool)
   - [6.3 Tiered Caching & Origin Shields: Preventing Global Stampedes](#63-tiered-caching--origin-shields-preventing-global-stampedes)
   - [6.4 Request Collapsing: Single-Flight Origin Fetches](#64-request-collapsing-single-flight-origin-fetches)
7. [The Edge as a Security Perimeter](#7-the-edge-as-a-security-perimeter)
   - [7.1 Web Application Firewalls (WAF) & The Normalization Arms Race](#71-web-application-firewalls-waf--the-normalization-arms-race)
   - [7.2 Rate Limiting Algorithms: Fixed Window vs. Leaky Bucket vs. Sliding Window Estimate](#72-rate-limiting-algorithms-fixed-window-vs-leaky-bucket-vs-sliding-window-estimate)
   - [7.3 Absorbing Volumetric DDoS Floods: 31.4 Tbps & The Anycast Sinkhole](#73-absorbing-volumetric-ddos-floods-314-tbps--the-anycast-sinkhole)
   - [7.4 Edge API Gateways & Origin Lockdown (mTLS & Cloudflare Tunnels)](#74-edge-api-gateways--origin-lockdown-mtls--cloudflare-tunnels)
8. [Edge Compute: V8 Isolates vs. Containers](#8-edge-compute-v8-isolates-vs-containers)
9. [The Bill: What You Give Up When You Put a Stranger in the Middle](#9-the-bill-what-you-give-up-when-you-put-a-stranger-in-the-middle)
10. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#10-comprehensive-review--practice-mcqs-with-detailed-explanations)
11. [Session 6 Summary Cheat Sheet](#11-session-6-summary-cheat-sheet)

---

## 1. The Physical Reality of Latency: Why the Edge Exists

No matter how efficient your database queries or server code are, network latency is bound by the laws of physics.

### 1.1 The Inviolable Speed-of-Light Budget in Optical Fiber
The speed of light in a vacuum ($c$) is approximately $299,792\text{ km/s}$. However, internet data travels through terrestrial glass optical fibers, which have a refractive index of approximately $n \approx 1.468$:

$$v_{\text{fiber}} = \frac{c}{n} = \frac{299,792\text{ km/s}}{1.468} \approx \mathbf{204,218\text{ km/s}}$$

$$\mathbf{\text{Fiber Latency Floor}} \approx \mathbf{4.90\text{ ms per 1,000 km (one-way)}}$$

```
PHYSICAL ROUND-TRIP TIME (RTT) FLOORS:
┌─────────────────────────────┬─────────────────────┬──────────────────┬─────────────────┐
│ Route Path                  │ Great-Circle Dist   │ Straight Glass   │ Real Measured   │
├─────────────────────────────┼─────────────────────┼──────────────────┼─────────────────┤
│ Mumbai ──> Chennai (return) │ ~1,030 km           │ ~10 ms           │ ~23 ms          │
│ Mumbai ──> Ashburn (return) │ ~12,800 km          │ ~126 ms          │ ~194 ms         │
└─────────────────────────────┴─────────────────────┴──────────────────┴─────────────────┘
```

#### The Latency Tax on Handshakes:
If an origin server is hosted in Northern Virginia (`us-east-1`) and a user is in Mumbai, India:
- A single round trip costs **at least 126 ms** (and typically ~194 ms due to fiber routing deviations).
- A cold connection using standard TCP + TLS 1.3 requires **2 to 3 round trips** before the first byte of application data is received:
  $$\text{Time to First Byte (TTFB)} \approx 3 \times 126\text{ ms} = \mathbf{378\text{ to } 580\text{ ms of idle transit delay!}}$$
- No hardware upgrade, compiler optimization, or database indexing on the origin can reduce this speed-of-light delay.

---

### 1.2 The Mathis Equation: Why Distance Kills TCP Throughput
Distance degrades not only latency, but also raw bandwidth throughput. In 1997, Matthew Mathis et al. published the mathematical relationship governing TCP throughput under packet loss:

$$\mathbf{\text{Throughput}} \approx \frac{\mathbf{\text{MSS}}}{\mathbf{\text{RTT}} \times \sqrt{\mathbf{p}}}$$

*(where $\text{MSS}$ is the Maximum Segment Size, $\text{RTT}$ is Round-Trip Time, and $p$ is the packet loss rate).*

```
TCP THROUGHPUT DEGRADATION OVER DISTANCE (AKAMAI MEASUREMENTS, 2010):
┌─────────────────────────┬──────────────┬──────────────┬──────────────────┬─────────────────┐
│ Distance from Edge      │ RTT          │ Packet Loss  │ TCP Throughput   │ 4 GB Download   │
├─────────────────────────┼──────────────┼──────────────┼──────────────────┼─────────────────┤
│ Local (< 100 miles)     │ 1.6 ms       │ 0.6%         │ 44 Mbps          │ 12 minutes      │
│ Regional (500–1,000 mi) │ 16 ms        │ 0.7%         │ 4 Mbps           │ 2.2 hours       │
│ Cross-Continent (~3k mi)│ 48 ms        │ 1.0%         │ 1 Mbps           │ 8.2 hours       │
│ Global (~6,000 miles)   │ 96 ms        │ 1.4%         │ 0.4 Mbps         │ 20 hours        │
└─────────────────────────┴──────────────┴──────────────┴──────────────────┴─────────────────┘
```

- When distance doubles, latency increases while packet loss along long router paths also compounds.
- Because throughput scales inversely with both $\text{RTT}$ and $\sqrt{p}$, **a server 60 times farther away can deliver throughput that is 110 times slower.**

---

### 1.3 The Two Values of the Edge: Handshake Absorption vs. Cache Hits

```
BENCHMARK: User ──(10 ms)──> Edge PoP (Mumbai) ──(194 ms)──> Origin (Virginia)

Scenario                                       Content Type    User TTFB     Relative Cost
──────────────────────────────────────────────────────────────────────────────────────────
1. Direct to origin (new TCP + TLS 1.3 conn)    Dynamic         621 ms        3.0 RTT
2. Direct to origin (connection kept alive)     Dynamic         208 ms        1.0 RTT
3. Edge proxy with NO pool (unpooled proxy)     Dynamic         625 ms        3.1 RTT (Slower!)
4. Edge proxy with WARM origin pool             Dynamic         233 ms        1.1 RTT
5. Edge proxy with CACHE HIT                    Static           34 ms        0.2 RTT
6. Edge proxy with CACHE HIT + Keep-Alive       Static           11 ms        0.1 RTT
```

#### Key Architectural Realizations:
1. **A Cache Miss is Not Faster**: If an edge proxy does not have a cached object, routing through the edge takes the same time as going direct (or slightly longer due to the extra proxy hop).
2. **Handshake Absorption**: The edge terminates the client's expensive TCP and TLS handshakes locally within **10 ms**, maintaining warm, pre-established connection pools over the high-latency 194 ms origin path.
3. **Cache Elimination**: When an asset is cached at the edge, the 194 ms origin transit delay is reduced to **0 ms**, delivering responses in 11 to 34 ms.

---

## 2. What a CDN Actually Is: Architecture & Routing

### 2.1 A Reverse Proxy Rented in 300+ Cities
Fundamentally, a Content Delivery Network (CDN) is **Session 4's reverse proxy (such as NGINX or Varnish) deployed across hundreds of data centers globally and sold as an infrastructure service**:

```
                       [ BGP ANYCAST IP: 104.16.12.34 ]
                                       │
     ┌─────────────────────────────────┼─────────────────────────────────┐
     ▼                                 ▼                                 ▼
[ Edge PoP: Mumbai ]          [ Edge PoP: London ]            [ Edge PoP: Ashburn ]
- NGINX/Pingora proxy         - NGINX/Pingora proxy           - NGINX/Pingora proxy
- TLS termination (10ms)      - TLS termination (10ms)        - TLS termination (10ms)
- Shared RAM & Disk Cache     - Shared RAM & Disk Cache       - Shared RAM & Disk Cache
- Edge WAF & Rate Limiter     - Edge WAF & Rate Limiter       - Edge WAF & Rate Limiter
     │                                 │                                 │
     └─────────────────────────────────┼─────────────────────────────────┘
                                       │ (Persistent Origin Pools)
                                       ▼
                       [ Regional Tiered Cache Shield ]
                                       │
                                       ▼
                            [ Your Origin Server ]
```

---

### 2.2 Anycast BGP Routing: One IP Address, Hundreds of Data Centers
How does a user in Mumbai reach the Mumbai edge server while a user in London reaches the London edge server using the exact same domain and IP address?

- **Unicast Routing**: A single IP address routes to a single physical network interface card in one data center.
- **Anycast BGP Routing**: The CDN announces the **exact same IP prefix** (e.g., `104.16.0.0/12` or `1.1.1.1`) into the global **Border Gateway Protocol (BGP)** table from over 300 data centers simultaneously.
- When an ISP router forwards packets toward that IP, standard internet routing naturally directs traffic along the topologically shortest network path to the nearest local Point of Presence (PoP).

---

### 2.3 The Airport Code Architecture (`colo` & `cf-ray`)
CDNs label their Points of Presence using standard three-letter **IATA Airport Codes**:
- `BOM`: Mumbai, India
- `DEL`: New Delhi, India
- `MAA`: Chennai, India
- `SIN`: Singapore
- `LHR`: London Heathrow, UK
- `IAD`: Washington Dulles, Virginia, USA

#### Tracing the Serving Node:
```bash
$ curl -s https://www.cloudflare.com/cdn-cgi/trace | grep colo
colo=BOM
```
Every response header returned by Cloudflare contains the `cf-ray` header:
```http
cf-ray: 8c1a94e2b0123456-BOM
```
The suffix indicates the specific PoP facility that terminated the request, providing visibility when diagnosing regional cache misses or latency issues.

---

### 2.4 The Founding Algorithm of CDNs: Consistent Hashing vs. Modulo Hashing
When a multi-node edge PoP distributes URLs across internal cache servers, how does it determine which server holds which URL?

#### The Modulo Hashing Trap:
$$\mathbf{\text{Node Index}} = \mathbf{\text{Hash(URL)}} \pmod N$$
If a cluster has $N = 10$ cache machines and adds an 11th node ($N = 11$):
- Because the modulo denominator changes, **90.9% of all existing URLs hash to a different node index!**
- Adding capacity causes a near-total cache flush, triggering a stampede of cache misses back to the origin server.

```
MODULO HASHING RE-ALLOCATION TEST (100,000 URLs, 10 Caches -> 11 Caches):
Scheme                               URLs that Move          Busiest vs. Mean Node
──────────────────────────────────────────────────────────────────────────────────
Modulo Hashing (hash(url) % N)       90.9% (Catastrophic)    1.02x
Consistent Hashing (1 point/node)     3.3%                   2.93x (Severe Load Imbalance)
Consistent Hashing (160 points/node)  8.2% (~1/11 fair share) 1.14x (Balanced!)
```

#### Consistent Hashing (Karger et al., 1997):
Developed at MIT by David Karger, Tom Leighton, and Danny Lewin (the mathematical foundation upon which **Akamai** was founded):
1. The hash space is mapped to an abstract circular ring ($0$ to $2^{32} - 1$).
2. Both cache server IDs and content URLs are hashed onto this continuum. A URL is stored on the first server encountered moving clockwise around the ring.
3. **Adding a Server**: When an 11th node is introduced, it claims only a fraction of the ring from its neighboring node. **Only $\approx \frac{1}{N+1}$ of keys move**, leaving all other cache mappings intact.
4. **Virtual Points (Replicas)**: To prevent load hotspots, each physical server hashes into 160 distinct "virtual tokens" across the ring. (Supported in NGINX via `hash $request_uri consistent;`).

---

### 2.5 The Software Stack Inside the Major CDNs
CDNs build on the reverse proxy concepts explored in earlier sessions:

```
┌──────────────────┬──────────────┬──────────────────────────────────────────────────────────────┐
│ CDN Provider     │ Founded Year │ Internal Software Core & Architectural Highlights            │
├──────────────────┼──────────────┼──────────────────────────────────────────────────────────────┤
│ **Akamai**       │ 1998         │ Custom C++ "Ghost" servers, ESI (2001), BBR congestion      │
├──────────────────┼──────────────┼──────────────────────────────────────────────────────────────┤
│ **Amazon CF**    │ 2008         │ AWS Edge infrastructure, BBR delivery to viewers since 2019 │
├──────────────────┼──────────────┼──────────────────────────────────────────────────────────────┤
│ **Fastly**       │ 2011         │ Heavily modified Varnish (VCL) + H2O web server; sub-sec purge│
├──────────────────┼──────────────┼──────────────────────────────────────────────────────────────┤
│ **Google Media** │ 2022         │ YouTube edge infrastructure; QUIC and BBR everywhere         │
├──────────────────┼──────────────┼──────────────────────────────────────────────────────────────┤
│ **Cloudflare**   │ 2010         │ Started as NGINX+LuaJIT. Replaced with Pingora (Rust, 2022)  │
│                  │              │ for origin proxying, and Oxy (Rust, 2025) for client edge.   │
└──────────────────┴──────────────┴──────────────────────────────────────────────────────────────┘
```

---

## 3. Edge Caching Mechanics: The Defaults & The State Machine

### 3.1 Default File-Extension Rules: Why HTML is Never Cached by Default
By default, CDNs cache static assets based on file extensions (typically ~68 extensions including `.css`, `.js`, `.png`, `.jpg`, `.webp`, `.woff2`, `.mp4`, `.pdf`).

**CDNs DO NOT cache HTML or JSON by default.**
- If a CDN cached `/index.html` by default, an application deploying an update or rendering personalized user account data would serve stale content to all visitors.
- Any response containing `Set-Cookie`, or `Cache-Control: private`, `no-store`, `no-cache`, or `max-age=0` bypasses the cache and routes directly to the origin.

---

### 3.2 The Cache Key Question: URL vs. Cookies & Headers
By default, a CDN computes the cache key using only the request URL:
$$\mathbf{\text{Default Cache Key}} = \mathbf{\text{scheme}} + \mathbf{\text{host}} + \mathbf{\text{path}} + \mathbf{\text{query}}$$

- Headers, cookies, and authentication tokens are **excluded from the default cache key**.
- While suitable for public assets like `/app.css`, this behavior creates security issues if applied to dynamic paths like `/account`. If `/account` is cached under this rule, User B will receive the cached HTML generated for User A. Custom cache rules and the `Vary` header are required to differentiate responses.

---

### 3.3 Deconstructing the 8-State `cf-cache-status` Machine

The `cf-cache-status` response header provides visibility into how the edge processed an asset:

```
┌──────────────┬────────────────────────────────────────────┬────────────┬──────────────────────────────┐
│ Status Value │ Operational Event                          │ Origin Hit?│ Underlying Mechanism         │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **HIT**      │ Served directly from edge cache.           │ **NO**     │ Asset is fresh; within TTL.  │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **MISS**     │ Eligible asset was not present in cache.   │ **YES**    │ Fetched and written to cache.│
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **EXPIRED**  │ Cached copy was present, but TTL elapsed.  │ **YES**    │ Re-fetched fresh from origin.│
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│**REVALIDATED**│ TTL elapsed; origin validated with 304.    │ Cheap 304  │ ETag / If-None-Match match.  │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **UPDATING** │ Serving stale copy while updating in bg.   │ Async      │ `stale-while-revalidate`.    │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **STALE**    │ Origin is unreachable; serving stale copy. │ Failed     │ `stale-if-error` fallback.   │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **DYNAMIC**  │ File type ineligible (e.g. HTML/API).      │ **YES**    │ Default behavior for HTML.   │
├──────────────┼────────────────────────────────────────────┼────────────┼──────────────────────────────┤
│ **BYPASS**   │ Eligible file, but origin sent `no-store`. │ **YES**    │ Cache-Control bypassed cache.│
└──────────────┴────────────────────────────────────────────┴────────────┴──────────────────────────────┘
```

---

## 4. Caching the Uncacheable: Dynamic Content at the Edge

Modern web architectures split web pages into large static shells and small dynamic components to maximize cache efficiency.

### 4.1 The Evolution: SSI $\rightarrow$ ESI $\rightarrow$ Ajax $\rightarrow$ Delta Compression

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              THE DYNAMIC CONTENT CACHING ARC                           │
├─────────────────────┬──────────────────┬───────────────────────────────────────────────┤
│ Architecture        │ Introduced       │ Where Personalization is Injected             │
├─────────────────────┼──────────────────┼───────────────────────────────────────────────┤
│ 1. **SSI**          │ 1994 (NCSA)      │ At the Origin Web Server (before sending).    │
│ 2. **ESI**          │ 2001 (Akamai)    │ At the CDN Edge PoP (stitched at the edge).   │
│ 3. **Ajax**         │ 2005 (Web 2.0)   │ Inside the Client Browser (via JavaScript).   │
│ 4. **RFC 9842**     │ 2025 (IETF)      │ Delta Compression over wire using Dictionary. │
└─────────────────────┴──────────────────┴───────────────────────────────────────────────┘
```

---

### 4.2 Edge-Side Includes (ESI): Assembling Personalized HTML at the PoP
Standardized by Akamai and Oracle in 2001:
1. The origin returns an unpersonalized page template containing XML fragment tags:
   ```html
   <header>
     <esi:include src="/fragment/user_nav" />
     <esi:include src="/fragment/cart_count" />
   </header>
   ```
2. Header sent: `Surrogate-Control: content="ESI/1.0"`.
3. The CDN caches the generic 40 KB shell globally.
4. When a user requests the page, the local Edge PoP fetches only the small dynamic fragments (`/fragment/user_nav`), stitches the HTML together in edge memory, and streams the assembled document to the user.
- **The Trade-off**: Origin bandwidth drops by **~98%**, but origin request volume can increase because each personal fragment is fetched as a separate subrequest.

---

### 4.3 Cloudflare Railgun to RFC 9842: Compression Dictionary Transport
- **Railgun (2013–2024)**: A daemon on the origin and an agent at the edge maintained the previous version of dynamic web pages. When content changed, the origin transmitted only the binary byte delta over a persistent WAN connection.
- **RFC 9842 (Compression Dictionary Transport, Sep 2025)**: Standardized this mechanism between browsers and servers:
  - The client announces previously downloaded assets as compression dictionaries: `Available-Dictionary: <sha256-hash>`.
  - The server compresses the updated page using Zstandard (`dcz`) or Brotli (`dcb`) referencing the cached dictionary.
  - A 24 KB dynamic page can compress down to **436 bytes (1.8% of original size)**, reducing dynamic transfer overhead.

---

## 5. Shared Caches as Attack Surfaces: Poisoning & Deception

Because a CDN cache is shared across thousands of users, discrepancies between cache keys and backend behavior can introduce security vulnerabilities.

### 5.1 Web Cache Poisoning: Weaponizing Unkeyed Headers
Documented by security researcher James Kettle in 2018:
- An origin application reads an HTTP request header (e.g., `X-Forwarded-Host`) and reflects it into the HTML page:
  ```html
  <script src="https://{{request.headers['X-Forwarded-Host']}}/app.js"></script>
  ```
- An attacker issues a request containing a malicious header:
  ```http
  GET /home HTTP/1.1
  Host: target.com
  X-Forwarded-Host: evil.example
  ```
- Because `X-Forwarded-Host` is **not in the CDN cache key**, the CDN stores the response under the key `https://target.com/home`.
- For the duration of the cache TTL, **every visitor requesting `/home` receives the poisoned HTML containing the attacker's script**.
- **Defense**: Strip unkeyed headers at the edge, add reflected headers to the cache key, or declare `Vary: X-Forwarded-Host`.

---

### 5.2 Web Cache Deception: Stealing Private Account Data
Discovered by Omer Gil in 2017:
- An attacker lures an authenticated user into visiting a path with a static file extension:
  ```text
  https://example.com/account/settings/profile.css
  ```
- The backend web framework ignores the `.css` suffix and renders the victim's private JSON account profile.
- The CDN edge identifies the `.css` extension, assumes the response is a public stylesheet, and **stores the private profile data in the public cache**.
- The attacker then requests `https://example.com/account/settings/profile.css` and reads the victim's private data from the edge cache.
- **Defense**: Never cache based on file extensions alone; inspect the response `Content-Type` and enforce `Cache-Control: private, no-store`.

---

## 6. Accelerating Byte Delivery: Transport & Topology

### 6.1 Edge BBR: Pacing TCP on the Long Haul
CDNs control both the last-mile connection to the client and the long-haul link back to the origin:
- By switching from loss-based TCP congestion control (CUBIC) to **BBR** at the edge, CDNs pace packet delivery to match physical link capacity without filling router buffers.
- CloudFront saw a **+22% throughput increase** on viewer paths, and Akamai deployed BBR across **>80%** of its end-user streaming connections.

---

### 6.2 Connection Pooling: Per-Worker Pools vs. Pingora's Shared Pool
In Session 3 and 4, we noted that NGINX isolates worker processes, meaning each worker maintains an independent upstream connection pool:
- When traffic is spread across 32 worker processes, connections in individual pools often go idle and close before reuse.
- **Cloudflare's Pingora (2022)**: Replaced NGINX's architecture with a multi-threaded Rust proxy that manages a single, shared connection pool across all CPU cores.
- **Production Impact**: Origin connection reuse increased from **87.1% to 99.92%**—a 160x reduction in new origin TCP and TLS handshakes.

---

### 6.3 Tiered Caching & Origin Shields: Preventing Global Stampedes

```
WITHOUT TIERED CACHE (Direct Edge Misses):
300 Global PoPs ─── 300 simultaneous cache misses ───> Origin Server (Overloaded!)

WITH TIERED CACHING (Origin Shield Architecture):
300 Edge PoPs ─── Fetch from Regional Tier ───> [ Origin Shield ] ─── 1 Fetch ───> Origin
```

- When an un-cached asset is requested globally, hundreds of PoPs can experience simultaneous cache misses.
- **Tiered Caching / Origin Shielding**: Edges query designated regional parent nodes (e.g., an AWS CloudFront Origin Shield or Cloudflare Tiered Cache). The shield aggregates requests, issues a single fetch to the origin, and populates the regional cache network.

---

### 6.4 Request Collapsing: Single-Flight Origin Fetches
Within an individual edge PoP, **Request Collapsing** prevents concurrent cache stampedes:
- If 50 users request an uncached video file simultaneously, the edge sends **one request** to the origin.
- The other 49 client requests are paused in memory and served from the response of the primary fetch as soon as bytes arrive.

---

## 7. The Edge as a Security Perimeter

Because all traffic routes through the CDN, the edge has evolved into an external security boundary:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              THE EDGE SECURITY PERIMETER                               │
├─────────────────────┬──────────────────┬───────────────────────────────────────────────┤
│ Security Discipline │ Mechanism        │ Operational Function                          │
├─────────────────────┼──────────────────┼───────────────────────────────────────────────┤
│ 1. **DDoS Defense** │ Anycast Sinks    │ Absorbs volumetric network floods across 300+ │
│                     │                  │ PoPs before packets reach origin pipes.       │
│ 2. **WAF**          │ Normalization    │ Evaluates payloads against SQLi/XSS signatures│
│                     │ & Text Inspection│ before forwarding requests.                   │
│ 3. **Rate Limiting**│ Sliding Window   │ Enforces IP and token budgets at the edge.    │
│ 4. **API Gateway**  │ Token Validation │ Verifies JWT signatures and routes traffic.   │
└─────────────────────┴──────────────────┴───────────────────────────────────────────────┘
```

### 7.1 Web Application Firewalls (WAF) & The Normalization Arms Race
A WAF inspects request parameters for known attack patterns before they reach application servers.
- **The Core Challenge**: Attackers obfuscate payloads using URL encoding, nested character escapes, and SQL comment insertions:
  ```sql
  ' OR 1=1 --                <── Blocked by standard SQLi rule
  '/**/OR/**/1=1--           <── Comment evasion bypasses naive regex
  ```
- **Text Normalization**: A WAF must recursively decode and normalize strings (stripping comments, resolving character escapes) before evaluating inspection rules.
- **Limitation**: WAFs provide defense-in-depth against mass automated scanning, but do not replace parameterized database queries and secure origin coding practices.

---

### 7.2 Rate Limiting Algorithms: Fixed Window vs. Leaky Bucket vs. Sliding Window Estimate
- **Fixed Window**: Tracks counters within a fixed timeframe (e.g., 100 requests per 10 seconds).
  - *Vulnerability*: An attacker can send 100 requests at $t = 9.999\text{ s}$ and another 100 at $t = 10.001\text{ s}$, delivering a burst of **200 requests in 2 milliseconds**.
- **Leaky Bucket (Strict)**: Enforces a steady transmission rate with zero burst allowance, which can reject legitimate browser behaviors that naturally open multiple concurrent asset connections.
- **Cloudflare's Sliding Window Estimate**:
  Tracks two counters per client (current window and previous window), estimating rate by weighting the previous window by the remaining overlap fraction:
  $$\mathbf{\text{Estimated Rate}} = \mathbf{\text{Current Count}} + \left( \mathbf{\text{Previous Count}} \times \frac{\text{Remaining Time}}{\text{Window Size}} \right)$$
  Measured across 400 million requests, this estimate maintains a **99.997% accuracy rate** while requiring only two counters in memory.

---

### 7.3 Absorbing Volumetric DDoS Floods: 31.4 Tbps & The Anycast Sinkhole
- In December 2025, Cloudflare recorded a peak Distributed Denial of Service (DDoS) attack reaching **31.4 Terabits per second (Tbps)** from the Aisuru botnet.
- No origin data center has network capacity to absorb a 31 Tbps flood directly.
- **How Anycast Defeats Floods**: Because the target IP address is announced across hundreds of global PoPs, BGP routing splits the attack traffic geographically. A 31 Tbps flood is distributed across hundreds of data centers as smaller, manageable streams that can be filtered locally.

---

### 7.4 Edge API Gateways & Origin Lockdown (mTLS & Cloudflare Tunnels)

```
THE BYPASS VULNERABILITY:
[ Attacker ] ── Bypasses CDN URL ──> [ Origin IP Directly: 198.51.100.22 ] ──> EXPLOITED!

THE ORIGIN LOCKDOWN:
[ Public Internet ] ──> [ Edge CDN / WAF / JWT ] ──mTLS / Tunnel──> [ Origin Server ]
                                                                       - Firewall drops all
                                                                         non-CDN traffic!
```

- If an origin server exposes a public IP address, attackers can bypass the CDN edge and send exploit payloads directly to origin ports.
- **Lockdown Mechanisms**:
  1. **Mutual TLS (mTLS / Authenticated Origin Pulls)**: The origin's TLS configuration requires the client to present a trusted client certificate issued by the CDN, rejecting direct connections.
  2. **Cloudflare Tunnels (`cloudflared`)**: The origin server establishes an outbound connection to the CDN PoP, operating without public open inbound ports.

---

## 8. Edge Compute: V8 Isolates vs. Containers

Modern CDNs allow running custom application logic directly on edge nodes using lightweight runtimes:

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│  CONTAINER COMPUTE (Docker / VM)     │  EDGE WORKER (V8 Isolate)            │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ - Boots complete user-space OS.      │ - Runs within a shared V8 runtime.   │
│ - Memory: 35 MB to 512 MB per unit.  │ - Memory: ~3 MB per isolate.         │
│ - Cold-start latency: 200ms to 2s.   │ - Cold-start latency: ~5 ms.         │
│ - High isolation; resource-heavy.    │ - High tenant density; fast spin-up. │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

Edge workers allow developers to run request filtering, token verification, and dynamic routing directly inside edge PoPs with minimal initialization overhead.

---

## 9. The Bill: What You Give Up When You Put a Stranger in the Middle

While CDNs provide significant performance and security capabilities, placing an intermediary in the request path introduces architectural trade-offs:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE ARCHITECTURAL TRADEOFFS                            │
├─────────────────────┬──────────────────────────────────────────────────────────────────┤
│ 1. TLS Private Keys │ The edge terminates HTTPS, giving the provider access to decrypt │
│                     │ and inspect all in-flight application payloads.                  │
├─────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 2. Single Point     │ Outages at major providers affect broad segments of the internet │
│    of Failure       │ (e.g., the June 2021 Fastly outage impacting Reddit and Spotify).│
├─────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 3. Traffic Control  │ CDN providers enforce acceptable-use and policy controls over     │
│                     │ intermediate traffic (e.g., historical bans on domain fronting). │
├─────────────────────┼──────────────────────────────────────────────────────────────────┤
│ 4. Vendor Lock-In   │ Migrating edge-specific WAF rules, caching logic, and serverless │
│                     │ workers to alternative providers requires architectural rewrites.│
└─────────────────────┴──────────────────────────────────────────────────────────────────┘
```

---

## 10. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
Why is it physically impossible for a direct network request between a client in Mumbai, India and an origin server in Northern Virginia, USA (`us-east-1`) to complete in less than 126 milliseconds round-trip time?
- A) Operating system kernels enforce a mandatory delay on intercontinental packets.
- B) Light traveling through glass optical fibers slows down by roughly one-third relative to vacuum ($v \approx 204,218\text{ km/s}$), creating a physical latency floor of ~4.9 ms per 1,000 km.
- C) Undersea fiber optic cables require hourly packet retransmissions.
- D) BGP routing protocols restrict transoceanic throughput to 10 Mbps.

**Answer: B**  
**Explanation:** Optical fibers have a refractive index of $n \approx 1.468$, slowing light to ~204,218 km/s (~4.9 ms per 1,000 km one-way). Over a great-circle distance of ~12,800 km, the physical round-trip floor is roughly 126 ms, irrespective of hardware or bandwidth.

---

### Question 2
According to the Mathis Equation ($\text{Throughput} \approx \frac{\text{MSS}}{\text{RTT} \times \sqrt{p}}$), how does increasing network distance affect TCP throughput on a connection with packet loss?
- A) Throughput increases linearly with distance.
- B) Throughput drops inversely with both the increased Round-Trip Time and the square root of packet loss, causing throughput to degrade significantly on long-haul paths.
- C) Throughput remains constant because TCP sliding windows adjust automatically.
- D) Packet loss decreases as distance increases.

**Answer: B**  
**Explanation:** The Mathis Equation demonstrates that TCP throughput ceiling decreases inversely with $\text{RTT}$ and $\sqrt{p}$. Because longer network paths encounter higher latency and cumulative loss, throughput drops sharply over distance.

---

### Question 3
How does BGP Anycast route user requests to the topologically nearest CDN edge data center?
- A) By assigning a unique, dedicated public IPv4 address to every edge server worldwide.
- B) By advertising the exact same IP prefix into the global BGP routing table from hundreds of data centers simultaneously, allowing internet routers to forward traffic along the shortest network path.
- C) By looking up client GPS coordinates through DNS servers.
- D) By maintaining a central routing database that directs traffic via software redirects.

**Answer: B**  
**Explanation:** BGP Anycast advertises identical IP prefixes from hundreds of locations simultaneously. Standard internet routing mechanics naturally direct client packets to the nearest advertising PoP.

---

### Question 4
Why does modulo hashing ($\text{hash(URL)} \pmod N$) cause severe performance degradation when scaling cache cluster nodes?
- A) Modulo operations consume excessive CPU cycles.
- B) Adding or removing a single node causes roughly $\frac{N-1}{N}$ (e.g., ~90%) of existing URLs to map to different cache servers, invalidating the cache and overwhelming the origin.
- C) Modulo hashing requires uniform URL lengths.
- D) Modulo hashing is restricted to 1,024 entries.

**Answer: B**  
**Explanation:** In modulo hashing, altering the divisor $N$ changes the hash assignments for nearly all keys (~90.9% when moving from 10 to 11 nodes), causing a widespread cache flush.

---

### Question 5
How does Consistent Hashing prevent widespread cache invalidation when adding a new cache server?
- A) By clearing the entire cache on the origin server.
- B) By mapping both servers and keys onto a circular hash ring, so adding a node claims only a proportional segment of the ring ($\sim \frac{1}{N+1}$ of keys) without altering other assignments.
- C) By encrypting cache keys with AES-256.
- D) By disabling caching for dynamic URLs.

**Answer: B**  
**Explanation:** Consistent Hashing places nodes and keys along a shared hash ring. Adding a server redistributes only the keys that fall within its newly claimed segment, preserving existing mappings across the rest of the cluster.

---

### Question 6
Why do CDNs avoid caching HTML web pages by default, while caching images, CSS, and JavaScript files?
- A) HTML files are too large to fit in edge memory.
- B) HTML pages often contain personalized content or dynamic user state; caching them under a standard URL key risks exposing private account data or serving stale markup.
- C) Browsers do not support caching HTML documents.
- D) HTML markup cannot be compressed with Gzip.

**Answer: B**  
**Explanation:** HTML files frequently contain personalized data or dynamic session markup. Caching HTML under default URL-based keys risks serving private account details to other users or showing outdated content.

---

### Question 7
In the `cf-cache-status` header, what does the status **`REVALIDATED`** indicate?
- A) The asset was missing from the cache and fetched fresh from the origin.
- B) The cached asset's TTL had expired, but an origin check returned an HTTP 304 Not Modified, refreshing the cache entry without re-downloading the body.
- C) The client bypassed the cache using `Cache-Control: no-store`.
- D) The origin server timed out.

**Answer: B**  
**Explanation:** `REVALIDATED` means the cached item was expired, but the edge revalidated it with the origin using conditional headers (`If-None-Match`). The origin confirmed the content was unchanged (`304 Not Modified`), refreshing the edge TTL without transferring payload data.

---

### Question 8
How does Edge-Side Includes (ESI) reduce origin server workloads on personalized web pages?
- A) By executing client JavaScript inside the edge proxy.
- B) By caching a shared HTML shell globally at the edge and fetching only small dynamic user fragments from the origin, assembling the final personalized page at the PoP.
- C) By compressing video streams using UDP.
- D) By disabling SSL/TLS encryption.

**Answer: B**  
**Explanation:** ESI splits pages into static templates and dynamic fragment tags (`<esi:include>`). The edge caches the static shell globally, requests only dynamic fragments from the origin, and stitches the completed document together before streaming to the client.

---

### Question 9
How does RFC 9842 (Compression Dictionary Transport) achieve high compression ratios on dynamic web pages?
- A) By removing HTML tag brackets.
- B) By using previously downloaded versions of a page or stylesheet as a static shared compression dictionary, transmitting only the binary delta of updated content.
- C) By converting HTTP responses into raw UDP frames.
- D) By encrypting payloads with one-time pads.

**Answer: B**  
**Explanation:** RFC 9842 allows a client to offer a previously cached resource as a dictionary for subsequent fetches. The server compresses the updated response against that known base, transmitting only the changes.

---

### Question 10
What vulnerability is introduced in a Web Cache Poisoning attack?
- A) An attacker overflows the edge server's TCP socket buffer.
- B) An attacker sends a request with an unkeyed header (e.g., `X-Forwarded-Host`) that the origin reflects into the HTML body, causing the edge to cache malicious script references for subsequent visitors.
- C) An attacker brute-forces origin passwords through the CDN.
- D) An attacker modifies DNS root zone records.

**Answer: B**  
**Explanation:** Cache poisoning exploits headers reflected by the backend that are not included in the CDN cache key. An injected payload is cached under the public URL, serving the malicious markup to subsequent visitors.

---

### Question 11
How does a Web Cache Deception attack compromise private account information?
- A) By intercepting TLS private keys in transit.
- B) An attacker tricks an authenticated user into visiting a private path appended with a static extension (e.g., `/account/profile.css`), prompting the CDN to treat the private profile as a public asset and store it in cache.
- C) By exploiting buffer overflows in BGP routers.
- D) By flooding the origin with SYN packets.

**Answer: B**  
**Explanation:** If an application serves dynamic account content on paths containing static extensions, a CDN configured to cache strictly by extension may store the private response in its public cache, allowing attackers to retrieve it.

---

### Question 12
Why did Cloudflare develop the Pingora proxy in Rust to replace per-worker NGINX upstream connection pools?
- A) NGINX was unable to parse HTTP/1.1 headers.
- B) NGINX maintained separate connection pools per worker process, leading to fragmented pools where connections idled and closed prematurely; Pingora uses a unified pool shared across all CPU cores.
- C) Rust executes system calls without CPU context switching.
- D) Pingora eliminates the need for TLS certificates.

**Answer: B**  
**Explanation:** In NGINX's multi-process model, each worker maintains its own upstream connection pool. Pingora uses a multi-threaded architecture with a shared connection pool across all cores, increasing connection reuse from 87.1% to 99.92%.

---

### Question 13
What is the function of an Origin Shield (Tiered Caching) in a global CDN?
- A) It blocks search engine crawlers from indexing web assets.
- B) It acts as an intermediate caching tier between regional edge PoPs and the origin, consolidating multiple edge misses into a single fetch to protect origin capacity.
- C) It translates IPv6 traffic into IPv4 packets.
- D) It enforces biometric user authentication.

**Answer: B**  
**Explanation:** An Origin Shield aggregates cache misses across global edge PoPs. Instead of hundreds of edge nodes querying the origin simultaneously, they query the shield, which issues a single request to the origin and distributes the response.

---

### Question 14
How does Request Collapsing protect an origin server during a sudden traffic spike on an uncached resource?
- A) By dropping incoming user connections with an HTTP 503 error.
- B) By identifying concurrent requests for the same uncached URL at an edge PoP, sending one request to the origin, and serving all waiting clients from that single response.
- C) By compressing incoming HTTP headers using Gzip.
- D) By caching responses directly in client browser storage.

**Answer: B**  
**Explanation:** Request collapsing identifies concurrent requests for the same missing cache key, dispatches a single fetch to the origin, and streams the incoming response to all waiting clients simultaneously.

---

### Question 15
Why is an Edge API Gateway or WAF ineffective if the origin server's public IP address remains exposed directly to the internet?
- A) Edge WAF rules only process encrypted traffic.
- B) Attackers can bypass the CDN entirely by sending requests directly to the origin IP, circumventing edge security filters unless the origin enforces mTLS or tunnels.
- C) DNS servers will refuse to resolve the domain name.
- D) Browsers block connections that traverse reverse proxies.

**Answer: B**  
**Explanation:** If an origin accepts direct connections from the internet, attackers can target its public IP to bypass edge WAF, rate-limiting, and authentication layers. Securing the origin requires firewalling all traffic except authenticated CDN connections (via mTLS or tunnels).

---

## 11. Session 6 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **Fiber Latency Floor** | Light in fiber travels at $\approx 204,218\text{ km/s}$ ($4.9\text{ ms per 1,000 km}$). Physical Mumbai-to-Ashburn floor is **126 ms RTT**. |
| **The Mathis Equation** | Throughput $\approx \frac{\text{MSS}}{\text{RTT} \times \sqrt{p}}$. Long distances compound latency and packet loss, sharply limiting TCP throughput. |
| **Anycast BGP Routing** | The same IP prefix is announced globally from 300+ data centers; BGP directs users to the nearest PoP. |
| **Airport Codes (`colo`)**| PoPs are designated by IATA codes (e.g., `BOM`, `DEL`, `LHR`, `IAD`), visible via `cf-ray` and `/cdn-cgi/trace`. |
| **Consistent Hashing** | Maps nodes and URLs onto a shared hash ring with virtual tokens (160/node). Adding a node moves only $\sim \frac{1}{N+1}$ of keys. |
| **Default Cache Rules**| Caches static extensions (`.css`, `.js`, `.jpg`); **never caches HTML or JSON by default**. |
| **Cache Key Formula** | Defaults to `scheme + host + path + query`. Excludes headers and cookies by default. |
| **`cf-cache-status`** | 8 distinct states: `HIT`, `MISS`, `EXPIRED`, `REVALIDATED` (304), `UPDATING`, `STALE`, `DYNAMIC`, `BYPASS`. |
| **Edge-Side Includes (ESI)**| Caches static HTML templates globally; assembles dynamic personalized fragments at the edge PoP. |
| **RFC 9842 Compression**| Uses cached responses as compression dictionaries for subsequent fetches, compressing dynamic pages down to ~1.8% of raw size. |
| **Cache Poisoning** | Injects malicious payloads via unkeyed headers reflected in origin markup, poisoning the public cache entry. |
| **Cache Deception** | Tricking users into opening private paths with static extensions (`.css`) prompts the CDN to cache private responses publicly. |
| **Shared Connection Pools**| Pingora's shared pool across CPU cores increased connection reuse from 87.1% to 99.92%, minimizing origin TLS handshakes. |
| **Origin Shielding** | Regional parent tiers consolidate global cache misses, preventing stampedes on origin backends. |
| **Origin Lockdown** | Origins must enforce mutual TLS (mTLS) or private tunnels (`cloudflared`) to prevent attackers from bypassing the edge. |
| **V8 Isolates** | Edge workers run inside lightweight V8 isolates (~5 ms start, ~3 MB RAM), avoiding container spin-up overhead. |
