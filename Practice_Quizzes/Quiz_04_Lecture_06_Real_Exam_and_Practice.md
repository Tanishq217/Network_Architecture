# Practice Quiz Bank: Lecture 06 Real Exam & Advanced Practice

> **Coverage**: 
> - **Section 1**: Complete Deconstruction of the **Official Lecture 06 Quiz** (All 7 real classroom exam questions on CDN Colos, speed of light, fiber refractive index, TCP handshake round trips, packet loss physics, and proxy vs CDN TTFB dynamics).
> - **Section 2**: Advanced High-Yield Practice Bank (15 tricky quantitative and systems engineering questions with step-by-step arithmetic: physical fiber optic latency derivations, TTFB decomposition, TLS 1.2 vs 1.3 handshake RTT budgets, cache hit ratio offload math, consistent hashing redistribution, and BGP Anycast routing).

---

# Section 1: Official In-Class Lecture 06 Quiz (Fully Deconstructed)

---

### Question 1: CDN Edge Colocation Naming Conventions
**What does "colo" represent in CDN architecture?**
- A) City code *(Selected in Class: 40%)*
- B) Airport code *(Selected in Class: 40%)*
- C) ISO code *(Selected in Class: 20%)*

**Correct Answer: B (Airport code / IATA 3-Letter Code)**

#### Deep Technical Explanation:
- In Content Delivery Networks (CDNs) and global edge infrastructure providers (such as Cloudflare, Fastly, Akamai, and AWS CloudFront), edge data centers are referred to as **Colocation Facilities ("Colos")** or **PoPs (Points of Presence)**.
- Across the global telecommunications industry, every CDN colo is standardized and identified using its nearest **IATA (International Air Transport Association) 3-letter airport code**:
  - `BOM`: Chhatrapati Shivaji Maharaj International Airport (Mumbai, India)
  - `DEL`: Indira Gandhi International Airport (Delhi, India)
  - `BLR`: Kempegowda International Airport (Bengaluru, India)
  - `FRA`: Frankfurt Airport (Frankfurt, Germany)
  - `LHR`: London Heathrow Airport (London, United Kingdom)
  - `IAD`: Washington Dulles International Airport (Ashburn/Northern Virginia, USA)
  - `SFO`: San Francisco International Airport (California, USA)
- **Why Option A ("City code") trapped 40% of students**: 
  - Many cities have multiple airports, and city names do not have universally recognized 3-letter networking codes. The telecommunications and CDN industries universally adopted the aviation IATA standard because every major international peering exchange point sits directly adjacent to commercial aviation hubs.

---

### Question 2: Speed of Light in Vacuum
**What is the speed of light in vacuum?**
- A) 299,792 km/s *(Selected in Class: 50%)*
- B) 299,792 m/s *(Selected in Class: 50%)*
- C) 299,792 cm/s *(Selected in Class: 0%)*

**Correct Answer: A (299,792 km/s)**

#### Step-by-Step Mathematical Explanation:
1. **The Fundamental Physical Constant ($c$)**:
   The exact defined speed of light in vacuum is:
   $$c = 299,792,458 \text{ meters/second}$$
2. **Convert to Kilometers per Second**:
   $$\text{Since } 1 \text{ km} = 1,000 \text{ meters:}$$
   $$c = \frac{299,792,458}{1,000} \text{ km/s} \approx \mathbf{299,792 \text{ km/s}} \quad (\approx 3 \times 10^5 \text{ km/s})$$
3. **The Unit Trap**: 
   - Exactly **50% of students fell into the trap** by selecting Option B (`299,792 m/s`), forgetting that $299,792 \text{ m/s}$ is roughly $300\text{ km/s}$—one thousand times too slow!

---

### Question 3: Refractive Index of Optical Fiber Glass
**What is the refractive index of glass used in optical fiber telecommunications?**
- A) 2.00 *(Selected in Class: 0%)*
- B) 1.468 *(Selected in Class: 94%)*
- C) 0.98 *(Selected in Class: 6%)*

**Correct Answer: B (1.468)**

#### Step-by-Step Mathematical Derivation:
1. **Physical Principle of Fiber Optic Transmission**:
   Light does not travel through empty space in a fiber optic cable; it travels through ultra-pure silica glass ($SiO_2$).
2. **Refractive Index ($n$) Definition**:
   The refractive index $n$ is the ratio of the speed of light in vacuum ($c$) to the phase velocity of light in the medium ($v$):
   $$n = \frac{c}{v} \implies v = \frac{c}{n}$$
3. **Standard Telecommunications Silica Core ($n \approx 1.468$)**:
   $$v = \frac{299,792 \text{ km/s}}{1.468} \approx \mathbf{204,218 \text{ km/s}}$$
   $$\approx \mathbf{200,000 \text{ km/s}} = \mathbf{\frac{2}{3} c}$$
4. **Latency per Kilometer in Glass Fiber**:
   $$t = \frac{1 \text{ km}}{204,218 \text{ km/s}} \approx 4.896 \times 10^{-6} \text{ seconds/km} \approx \mathbf{4.9 \ \mu\text{s per km}}$$
   $$\implies \mathbf{\approx 5 \ \mu\text{s per km (or } 4.9\text{ ms per } 1,000\text{ km)}}.$$

---

### Question 4: TCP Handshake Round-Trip Count
**How many round trips does it take to start a TCP connection (TCP handshake)?**
- A) 1 *(Selected in Class: 22%)*
- B) 1.5 *(Selected in Class: 22%)*
- C) 2 *(Selected in Class: 22%)*
- D) 3 *(Selected in Class: 35%)*

**Correct Answer: B (1.5 round trips)**

#### Step-by-Step Message & Round-Trip Analysis:
1. **The 3-Way Handshake Message Exchange**:
   - **Message 1 (Client $\to$ Server)**: `SYN` (travels 1 one-way half-trip $= 0.5 \text{ RTT}$)
   - **Message 2 (Server $\to$ Client)**: `SYN-ACK` (travels 1 one-way half-trip $= 0.5 \text{ RTT}$)
   - **Message 3 (Client $\to$ Server)**: `ACK` (travels 1 one-way half-trip $= 0.5 \text{ RTT}$)
2. **Summing the Round-Trip Traversals**:
   $$\text{Total Traversals} = 3 \text{ one-way legs}$$
   $$\text{Total Round Trips} = 3 \times 0.5 \text{ RTT} = \mathbf{1.5 \text{ Round Trips}}$$
3. **Concept & Trap Breakdown**:
   - **Why 35% chose 3**: Confused the *count of packets/messages* (3) with the *number of round-trip times* (1.5).
   - **Why 22% chose 1**: From the perspective of the client, the client can begin sending data along with the final `ACK` after 1 RTT (SYN + SYN-ACK). However, across the complete end-to-end distributed system, the full handshake completes at the server only when the 3rd leg arrives, totaling **1.5 full traversals ($1.5 \text{ RTT}$)**.

---

### Question 5: Geographical Distance vs Packet Loss
**Will the packet loss be higher with distance?**
- A) Yes *(Selected in Class: 81%)*
- B) No *(Selected in Class: 19%)*

**Correct Answer: A (Yes)**

#### Deep Technical & Mathematical Explanation:
- As the physical distance between client and server expands:
  1. **Hop Count Compounding**: Packets must traverse more Autonomous Systems (AS), BGP peering border routers, and Internet Exchange Points (IXPs). If each hop has a small drop probability $p$, the end-to-end delivery probability across $N$ hops is:
     $$P_{\text{success}} = (1 - p)^N \implies P_{\text{loss}} = 1 - (1 - p)^N$$
     As $N$ grows with distance, $P_{\text{loss}}$ strictly increases.
  2. **Submarine Cable Repeaters**: Transoceanic fiber cables require optical amplifiers every 50–100 km, introducing signal-to-noise degradation and chromatic dispersion.
  3. **Queue Sizing & Bufferbloat**: High-latency WAN links aggregate traffic from thousands of routers; transient queue exhaustion occurs more frequently across long backhauls.

---

### Question 6: TTFB Impact of an Intermediate Proxy
**If I add a proxy between origin and browser, will the TTFB (Time to First Byte) increase or decrease?**
- A) Increase *(Selected in Class: 80%)*
- B) Decrease *(Selected in Class: 20%)*

**Correct Answer: A (Increase)**

#### Deep Technical Explanation:
- **Time to First Byte (TTFB)** is the duration from when the client initiates the request until the first byte of response data arrives at the browser:
  $$\text{TTFB} = \text{DNS} + \text{Connection Setup} + \text{Request Dispatch} + \text{Processing} + \text{Response Transit}$$
- If you introduce a **proxy** (without local caching):
  1. The client must establish a TCP/TLS connection to the proxy.
  2. The proxy must read and parse the client's HTTP request.
  3. The proxy must establish (or lease) a separate TCP/TLS connection to the origin server.
  4. The origin processes the request and sends the response back to the proxy.
  5. The proxy buffers/parses the upstream response and forwards it to the client.
- Every intermediate network hop and software hop adds serialization latency, socket buffer copying, and TCP scheduling delay. Therefore, for an uncached payload, adding an intermediate proxy **strictly increases TTFB**.

---

### Question 7: Architectural Comparison of Worst TTFB
**Given what you know so far, which option would give the worst TTFB for a customer?**
- A) Reverse proxy *(Selected in Class: 50%)*
- B) Proxy *(Selected in Class: 36%)*
- C) CDN *(Selected in Class: 14%)*

**Correct Answer: A (Reverse proxy)**

#### Deep Technical Comparison:
1. **CDN (Best TTFB)**:
   - Thousands of edge PoPs globally distributed directly next to users (e.g. within 2–10 ms).
   - Terminates TCP and TLS at the edge (eliminating WAN handshake latency).
   - Serves cached assets directly from memory/SSD, achieving single-digit millisecond TTFB.
2. **Forward Proxy (Moderate TTFB)**:
   - Typically deployed close to the user's local network (corporate office LAN or ISP edge).
   - Local DNS and local caching speed up requests.
3. **Reverse Proxy (Worst TTFB)**:
   - Sits in front of the origin server (often in the exact same centralized datacenter, e.g. AWS `us-east-1`).
   - A user in Bengaluru querying an origin in Frankfurt still suffers the full **120+ ms transcontinental WAN speed-of-light delay** to reach the reverse proxy.
   - On top of this maximum geographical distance, the reverse proxy introduces internal inspection, TLS re-encryption/decryption, load balancing algorithms, and an additional internal network hop before reaching the application backend.

---

# Section 2: Advanced High-Yield Practice Bank (With Step-by-Step Math)

---

### Question 8: Transcontinental Submarine Fiber Latency Floor
**A direct submarine fiber optic cable connects Mumbai (India) to Ashburn (Virginia, USA) with a physical cable distance of $13,500\text{ km}$. Assuming telecommunications silica glass ($n = 1.468$), what is the absolute physical speed-of-light propagation floor for a single Round-Trip Time (RTT) across this link?**
- A) ~66 ms
- B) ~132 ms
- C) ~90 ms
- D) ~264 ms

**Correct Answer: B (~132 ms)**

#### Step-by-Step Mathematical Calculation:
1. **Speed of Light in Fiber Core**:
   $$v = \frac{c}{n} = \frac{299,792 \text{ km/s}}{1.468} \approx 204,218 \text{ km/s}$$
2. **One-Way Propagation Delay ($t_{\text{one-way}}$)**:
   $$t_{\text{one-way}} = \frac{\text{Distance}}{v} = \frac{13,500 \text{ km}}{204,218 \text{ km/s}} \approx 0.0661 \text{ seconds} \approx \mathbf{66.1 \text{ ms}}$$
3. **Round-Trip Propagation Time ($\text{RTT}_{\text{floor}}$)**:
   $$\text{RTT}_{\text{floor}} = 2 \times t_{\text{one-way}} = 2 \times 66.1 \text{ ms} = \mathbf{132.2 \text{ ms}}$$
- **Significance**: No software optimization, CDN caching configuration, or protocol upgrade can ever make an origin round trip faster than this 132 ms physical limit. This is why edge CDN PoPs are mandatory.

---

### Question 9: TLS 1.2 vs TLS 1.3 TTFB Latency Budget
**A browser in Tokyo connects to an origin server in London ($\text{RTT} = 220\text{ ms}$). Assuming DNS is already resolved and server processing time is $30\text{ ms}$, what is the total TTFB for the first HTTPS request under TLS 1.2 vs TLS 1.3 (with fresh TCP and TLS handshakes)?**
- A) TLS 1.2: 470 ms; TLS 1.3: 250 ms
- B) TLS 1.2: 690 ms; TLS 1.3: 470 ms
- C) TLS 1.2: 910 ms; TLS 1.3: 690 ms
- D) TLS 1.2: 250 ms; TLS 1.3: 250 ms

**Correct Answer: B (TLS 1.2: 690 ms; TLS 1.3: 470 ms)**

#### Step-by-Step Mathematical Calculation:
1. **Handshake RTT Requirements**:
   - **TCP Handshake**: Requires **1 RTT** before client can initiate TLS.
   - **TLS 1.2 Handshake**: Requires **2 RTTs** (ClientHello $\to$ ServerHello/Cert $\to$ KeyExchange $\to$ Finished).
   - **TLS 1.3 Handshake (RFC 8446)**: Requires strictly **1 RTT** (combines Key Exchange into ClientHello).
   - **HTTP Request / Response**: Client sends request, server processes ($30\text{ ms}$), and first byte travels back ($0.5\text{ RTT}$ request $+ 0.5\text{ RTT}$ response $= \mathbf{1\text{ RTT}}$ total).

2. **Calculate TLS 1.2 TTFB**:
   $$\text{Total RTTs} = \text{TCP (1)} + \text{TLS 1.2 (2)} + \text{HTTP (1)} = \mathbf{4 \text{ RTTs}}$$
   $$\text{TTFB}_{\text{TLS 1.2}} = (3 \times 220\text{ ms}) + 30\text{ ms} + (0.5 \times 220\text{ ms}) = \mathbf{690 \text{ ms}}$$
   *(Or: 3 full round trips before response starts returning: $3 \times 220 + 30 = 690\text{ ms}$).*

3. **Calculate TLS 1.3 TTFB**:
   $$\text{Total RTTs} = \text{TCP (1)} + \text{TLS 1.3 (1)} + \text{HTTP (1)} = \mathbf{3 \text{ RTTs}}$$
   $$\text{TTFB}_{\text{TLS 1.3}} = (2 \times 220\text{ ms}) + 30\text{ ms} + (0.5 \times 220\text{ ms}) = \mathbf{470 \text{ ms}}$$
   $$\text{Latency Reduction} = 690 - 470 = \mathbf{220 \text{ ms (exactly 1 full RTT saved!)}}$$

---

### Question 10: CDN Cache Hit Ratio (CHR) & Origin Offload
**An e-commerce website experiences a peak traffic surge of $80,000$ requests per second. The engineering team enables Cloudflare CDN caching, which achieves a Cache Hit Ratio (CHR) of $97.5\%$. How many requests per second actually reach the upstream origin backend?**
- A) 8,000 req/s
- B) 2,000 req/s
- C) 20,000 req/s
- D) 78,000 req/s

**Correct Answer: B (2,000 req/s)**

#### Step-by-Step Mathematical Calculation:
1. **Cache Miss Percentage**:
   $$\text{Miss Rate} = 100\% - \text{CHR} = 100\% - 97.5\% = \mathbf{2.5\%} = 0.025$$
2. **Compute Origin Traffic**:
   $$\text{Origin Requests} = \text{Total Traffic} \times \text{Miss Rate}$$
   $$\text{Origin Requests} = 80,000 \times 0.025 = \mathbf{2,000 \text{ requests/second}}$$
3. **Offload Ratio**:
   $$\text{Offloaded Traffic} = 80,000 - 2,000 = \mathbf{78,000 \text{ req/s absorbed at the edge}}.$$

---

### Question 11: Consistent Hashing vs Modulo Redistribution
**A distributed CDN edge tier has $10$ caching servers. Due to a hardware failure, $1$ server crashes. Under standard Modulo Hashing ($\text{hash}(key) \pmod N$), what percentage of keys must be remapped to different servers, compared to Consistent Hashing?**
- A) Modulo: 10%; Consistent Hashing: 10%
- B) Modulo: 90%; Consistent Hashing: 10%
- C) Modulo: 100%; Consistent Hashing: 0%
- D) Modulo: 50%; Consistent Hashing: 50%

**Correct Answer: B (Modulo: 90%; Consistent Hashing: 10%)**

#### Step-by-Step Mathematical Derivation:
1. **Modulo Hashing ($N \to N-1$)**:
   - When the divisor changes from $10$ to $9$, almost every key hash maps to a completely different bucket:
     $$\text{Keys Remapped} = \frac{N - 1}{N} = \frac{10 - 1}{10} = \frac{9}{10} = \mathbf{90\%}$$
   - This triggers a massive **Cache Stampede / Thundering Herd** that crashes the origin.
2. **Consistent Hashing (Ring Topology)**:
   - When a node is removed from the ring, only the keys that were assigned to the failed node are reassigned to its immediate clockwise successor.
   - All other keys remain on their existing servers:
     $$\text{Keys Remapped} = \frac{1}{N} = \frac{1}{10} = \mathbf{10\%}$$

---

### Question 12: Mathis Equation Throughput vs RTT
**According to the Mathis formula for single-stream TCP throughput:
$$\text{Throughput} \le \frac{1.46 \times \text{MSS}}{\text{RTT} \times \sqrt{p}}$$
If a CDN edge PoP terminates TCP connections locally at an RTT of $10\text{ ms}$ instead of the client connecting directly to an origin server at an RTT of $160\text{ ms}$ (assuming the same packet loss rate $p = 0.01$ and MSS = 1460 bytes), by what factor does the maximum theoretical TCP throughput increase?**
- A) $4\times$
- B) $8\times$
- C) $16\times$
- D) $256\times$

**Correct Answer: C ($16\times$)**

#### Step-by-Step Mathematical Calculation:
1. **Analyze the Proportionality**:
   $$\text{Throughput} \propto \frac{1}{\text{RTT}}$$
2. **Ratio of Throughputs**:
   $$\frac{\text{Throughput}_{\text{CDN}}}{\text{Throughput}_{\text{Origin}}} = \frac{\text{RTT}_{\text{Origin}}}{\text{RTT}_{\text{CDN}}} = \frac{160 \text{ ms}}{10 \text{ ms}} = \mathbf{16}$$
- **Conclusion**: Reducing the RTT from 160 ms to 10 ms provides an immediate **$16\times$ increase** in single-stream TCP throughput!

---

### Question 13: BGP Anycast Equal-Cost Routing Flapping
**Why does BGP Anycast work flawlessly for stateless protocols (DNS over UDP) but requires careful engineering for long-lived TCP connections?**
- A) Anycast cannot route TCP packets
- B) If an intermediate tier router performs ECMP (Equal-Cost Multi-Path) route adjustments during network flap, subsequent packets in the same TCP stream may be routed to a different CDN PoP, which responds with a TCP `RST` because it has no record of the connection
- C) Anycast limits MTU to 512 bytes
- D) BGP Anycast disables IP checksum validation

**Correct Answer: B (State synchronization across Anycast PoPs & ECMP routing flap)**

#### Deep Technical Breakdown:
- Anycast announces the exact same IP prefix (e.g. `1.1.1.1`) from hundreds of data centers worldwide.
- UDP DNS requests are single request/response pairs: even if successive queries go to different PoPs, both can answer statelessly.
- TCP is **stateful**. If a route flap occurs during a multi-megabyte download and mid-stream packets land on PoP `DEL` instead of PoP `BOM`, `DEL`'s kernel has no TCP Transmission Control Block for that 4-tuple and immediately transmits a **`TCP RST` (Connection Reset)**, terminating the user's download.

---

### Question 14: Origin Shielding Architecture
**What is the primary role of an "Origin Shield" (Parent Cache) in a multi-tier CDN architecture?**
- A) To encrypt traffic with quantum cryptography
- B) To aggregate cache misses from hundreds of edge PoPs into a single centralized caching layer, preventing hundreds of concurrent edge misses from thundering herd against the origin backend
- C) To replace origin databases with DNS servers
- D) To eliminate the need for SSL certificates

**Correct Answer: B (Request collapsing and multi-tier cache miss aggregation)**

#### Deep Technical Breakdown:
- Without an Origin Shield: If 300 global edge PoPs experience a cache miss for a newly released video segment simultaneously, all 300 PoPs issue parallel backend requests to the origin.
- With an Origin Shield: All 300 edge PoPs forward their misses to 1 or 2 designated **Shield PoPs** (located adjacent to the origin). The Shield PoP performs **request collapsing** (`proxy_cache_use_stale updating`), fetching the object **once** from the origin and distributing it to all 300 edge nodes.

---

### Question 15: Edge Side Includes (ESI) Assembly
**How does Edge Side Includes (ESI) allow CDNs to cache dynamic web pages?**
- A) By converting PHP into static HTML
- B) By breaking a web page into static structural templates (cached for hours at the edge) and dynamic XML tags (`<esi:include src="/user/cart" />`) that the edge fetches and stitches together before returning the final response to the browser
- C) By running full PostgreSQL databases inside the CDN edge kernel
- D) By serving all users the same cached cart contents

**Correct Answer: B (Edge fragment assembly and dynamic tag stitching)**

#### Architecture Flow:
```text
Client Request
      │
      ▼
┌────────────── CDN Edge PoP ──────────────┐
│  1. Serves cached template (Header, CSS)  │
│  2. Identifies <esi:include src="/cart"/> │
│  3. Fetches /cart from Origin Backend    │
│  4. Stitches dynamic fragment into page  │
└──────────────────────────────────────────┘
      │
      ▼
Complete HTML Page Delivered to Client
```

---

### Question 16: `cf-cache-status` Header Inspection
**When inspecting HTTP response headers from a Cloudflare CDN, which status code indicates that the asset was expired in the edge cache, was revalidated with the origin via `If-None-Match`, and the origin replied with `304 Not Modified`?**
- A) `HIT`
- B) `MISS`
- C) `REVALIDATED`
- D) `EXPIRED`

**Correct Answer: C (`REVALIDATED`)**

#### Cloudflare Cache Status Taxonomy:
- **`HIT`**: Object served fresh directly from edge memory/disk.
- **`MISS`**: Object was not in edge cache; fetched completely from origin.
- **`EXPIRED`**: Object was in cache but exceeded TTL; fetched new copy from origin.
- **`REVALIDATED`**: Expired object was verified against origin using conditional headers (`If-Modified-Since` / `ETag`); origin confirmed unchanged via `304 Not Modified`, and edge served the existing cached copy.
- **`BYPASS`**: Client/server explicitly bypassed cache (e.g. `Cache-Control: no-cache`).
- **`DYNAMIC`**: Request was not eligible for caching (e.g. `POST` request or API route).

---

### Question 17: Fiber Optic Circuit Detour Factor (Routing Penalty)
**In real-world fiber deployments, physical fiber cables cannot be laid in a straight line (Great-Circle Distance) between cities due to roads, mountains, and continental coastlines. The telecommunications industry applies an average "detour routing factor" of $1.3\times$. If the Great-Circle distance between London and New York is $5,585\text{ km}$, what is the realistic one-way fiber propagation delay?**
- A) ~18.6 ms
- B) ~35.6 ms
- C) ~71.2 ms
- D) ~27.3 ms

**Correct Answer: B (~35.6 ms)**

#### Step-by-Step Mathematical Calculation:
1. **Actual Fiber Cable Distance ($D_{\text{fiber}}$)**:
   $$D_{\text{fiber}} = \text{Great-Circle Distance} \times \text{Detour Factor}$$
   $$D_{\text{fiber}} = 5,585 \text{ km} \times 1.3 = \mathbf{7,260.5 \text{ km}}$$
2. **Speed of Light in Fiber Core**:
   $$v = \frac{c}{1.468} \approx 204,218 \text{ km/s}$$
3. **One-Way Latency**:
   $$t_{\text{one-way}} = \frac{7,260.5 \text{ km}}{204,218 \text{ km/s}} \approx 0.03555 \text{ seconds} \approx \mathbf{35.6 \text{ ms}}$$
4. **Resulting Theoretical RTT Floor**:
   $$\text{RTT} = 2 \times 35.6 \text{ ms} = \mathbf{71.2 \text{ ms}}$$

---

### Question 18: HTTP/2 Stream Multiplexing & HOL Blocking
**Why does HTTP/2 eliminate application-layer Head-of-Line (HOL) blocking but remain susceptible to transport-layer HOL blocking?**
- A) Because HTTP/2 still uses text headers
- B) Because HTTP/2 multiplexes multiple streams over a single shared TCP connection; if a single TCP packet is lost on the wire, the kernel TCP receiver buffer stops delivering all subsequent packets to userspace until the lost segment is retransmitted
- C) Because HTTP/2 disables stream prioritization
- D) Because HTTP/2 only supports 6 concurrent requests

**Correct Answer: B (Multiplexing over a single in-order TCP byte stream)**

#### Deep Technical Breakdown:
- In HTTP/1.1, multiple connections were required because each connection could only process one request-response cycle at a time (application HOL blocking).
- HTTP/2 introduced **binary framing** with 31-bit Stream IDs, allowing hundreds of requests to interleave frames concurrently over a single TCP socket.
- However, TCP guarantees an **in-order reliable byte stream**. If Packet 3 (belonging to Stream A) is dropped on a lossy cellular link, the operating system kernel cannot pass Packet 4 (belonging to Stream B) to the web browser until Packet 3 is retransmitted. This transport-level HOL blocking was the primary architectural motivation for creating **QUIC and HTTP/3 over UDP**.

---

### Question 19: Cache Poisoning via Unkeyed Query Parameters
**An attacker sends a request to a CDN with the header `X-Forwarded-Host: evil.com` for a public asset `/main.js`. The origin includes this header value in a generated script URL, and the CDN caches the response for all users. What vulnerability has occurred?**
- A) SQL Injection
- B) Web Cache Deception
- C) Web Cache Poisoning
- D) DNS Amplification

**Correct Answer: C (Web Cache Poisoning)**

#### Technical Mechanics:
- **Cache Key**: A CDN uniquely identifies cached objects using a **Cache Key** (typically `Host + Path`).
- If an application uses an HTTP header (like `X-Forwarded-Host` or `X-Original-URL`) to alter response content, but the CDN does **not** include that header in its Cache Key (it is **unkeyed**), the malicious response is stored under the legitimate asset's key. All subsequent normal users receive the poisoned, malicious payload from the edge cache!

---

### Question 20: RFC 9842 Shared Dictionary Compression
**How does the modern RFC 9842 (Compression Dictionary Transport) standard dramatically reduce bandwidth for web applications?**
- A) It removes all vowels from text
- B) It allows browsers to use a previously fetched version of an application bundle (or CSS file) as a pre-shared dictionary for Brotli/Zstandard compression, transmitting only the minute diff (often achieving $>90\%$ size reduction)
- C) It converts all JSON into XML
- D) It enforces gzip level 1 compression

**Correct Answer: B (Shared dictionary differential compression)**

#### Technical Efficiency:
- When updating an app from `v1.0.0` (500 KB) to `v1.0.1` (502 KB), standard gzip re-compresses the entire 502 KB file (~120 KB download).
- Under RFC 9842, the browser specifies `Sec-Available-Dictionary: <hash_of_v1>`, and the CDN uses `v1.0.0` as the external dictionary to compress `v1.0.1`, transmitting **under 2 KB of pure patch data**!

---

### Question 21: Anycast DDoS Sinkholing Capacity
**A malicious botnet targets an enterprise with a $500\text{ Gbps}$ volumetric UDP flood. The enterprise routes its traffic through a global CDN Anycast network with 300 PoPs, each equipped with $50\text{ Gbps}$ of transit capacity. How does the Anycast architecture survive the attack?**
- A) By turning off DNS globally
- B) The Anycast IP distributes the attack geographically across all 300 PoPs simultaneously; each PoP absorbs and filters only $\approx 1.67\text{ Gbps}$ locally, well within its $50\text{ Gbps}$ capacity
- C) By forwarding the attack traffic to Google
- D) By dropping all UDP packets across the internet

**Correct Answer: B (Geographical attack distribution and distributed sinkholing)**

#### Step-by-Step Mathematical Calculation:
1. **Total Attack Bandwidth**:
   $$\text{Attack Volume} = 500 \text{ Gbps}$$
2. **Number of Anycast Edge Ingress Nodes**:
   $$\text{Edge PoPs} = 300$$
3. **Average Ingress Load per PoP**:
   $$\text{Load per PoP} = \frac{500 \text{ Gbps}}{300 \text{ PoPs}} \approx \mathbf{1.67 \text{ Gbps per PoP}}$$
4. **Capacity Headroom**:
   - Each PoP possesses $50 \text{ Gbps}$ capacity.
   - $1.67 \text{ Gbps} \ll 50 \text{ Gbps}$ ($< 3.5\%$ link utilization).
   - The attack is absorbed and scrubbed at the edge without single point of failure.

---

### Question 22: NGINX Stale Cache Handling Under Upstream Failure
**When an origin backend crashes, which NGINX directive allows the reverse proxy/edge cache to continue serving cached content to clients (even after its TTL has expired) while shielding users from 502/504 errors?**
- A) `proxy_pass http://localhost;`
- B) `proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;`
- C) `proxy_set_header X-Real-IP $remote_addr;`
- D) `fastcgi_pass unix:/var/run/php.sock;`

**Correct Answer: B (`proxy_cache_use_stale ...`)**

#### RFC 5861 / NGINX Implementation:
- Enabling `proxy_cache_use_stale` instructs NGINX: *"If the upstream origin returns a 502 Bad Gateway, 504 Gateway Timeout, or drops the connection, do not serve an error to the user; immediately serve the expired cached copy and retry upstream asynchronously."*
