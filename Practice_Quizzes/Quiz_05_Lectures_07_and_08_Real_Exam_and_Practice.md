# Practice Quiz Bank: Lectures 07 & 08 Real Exam & Advanced Practice

> **Coverage**: 
> - **Section 1**: Complete Deconstruction of the **Official Lectures 07 & 08 Quiz** (All 11 real classroom exam questions on large-scale cricket scorecard delivery, polling vs WebSockets economics, why 79% of students fail the CDN caching question, child manifest caching in HLS, and video player buffer mechanics).
> - **Section 2**: Advanced High-Yield Practice Bank (15 tricky quantitative and systems engineering questions with step-by-step arithmetic: Hotstar 50M socket memory calculation, S3 5,500 GET/s prefix sharding, 11 KB S3 break-even line, HLS latency budgets, ABR ladder bitrate switching, and AWS Full Jitter backoff math).

---

# Section 1: Official In-Class Lectures 07 & 08 Quiz (Fully Deconstructed)

---

### Question 1: Real-World Event Cadence in Cricket
**How long is the duration of a ball in cricket?**
- A) 5 seconds *(Selected in Class: 65%)*
- B) 10 seconds *(Selected in Class: 15%)*
- C) 45 seconds *(Selected in Class: 15%)*
- D) 1 minute *(Selected in Class: 5%)*

**Correct Answer: C (45 seconds)**

#### Deep Technical & Architectural Explanation:
- **The 65% Student Trap**: 
  - 65% of students chose **5 seconds** because they instinctively visualised the physical act of the bowler releasing the ball until the batsman plays it.
- **The Distributed Systems Architecture Reality**:
  - From the perspective of a systems architect designing a live score delivery engine, the **lifecycle of a ball** encompasses:
    1. Bowler walks back to the mark ($15\text{–}20\text{ seconds}$).
    2. Setting the field and captain instructions ($5\text{–}10\text{ seconds}$).
    3. Bowler run-up and delivery ($5\text{ seconds}$).
    4. Batsman stroke, running between wickets, ball returned to the keeper ($10\text{ seconds}$).
  - In international cricket, the overall cycle per ball averages **~45 seconds**!
- **Architectural Consequence**: If a mobile app polls the backend every 1 second, **44 out of 45 requests are completely redundant**, returning identical score data and wasting millions of dollars in bandwidth and server CPU!

---

### Question 2: Rapid Bowler Cadence (Ravindra Jadeja)
**How much time does Ravindra Jadeja take for a ball?**
- A) 5 seconds *(Selected in Class: 4%)*
- B) 10 seconds *(Selected in Class: 22%)*
- C) 25 seconds *(Selected in Class: 65%)*
- D) 45 seconds *(Selected in Class: 9%)*

**Correct Answer: C (25 seconds)**

#### Real-World Context:
- Spin bowlers—exemplified famously by Indian left-arm spinner Ravindra Jadeja—have exceptionally short run-ups and minimal delay between deliveries. Jadeja routinely bowls an entire 6-ball over in under 2.5 minutes, averaging **~20 to 25 seconds per ball**.
- **System Impact**: Polling intervals and CDN cache TTLs cannot be hardcoded statically to 45 seconds; they must dynamically account for worst-case peak cadences ($\approx 20\text{–}25\text{ s}$) to prevent stale score displays during rapid overs.

---

### Question 3: Over Duration Calculation
**How long is an over in cricket?**
- A) 1 minute *(Selected in Class: 0%)*
- B) 2 minutes *(Selected in Class: 13%)*
- C) 5 minutes *(Selected in Class: 74%)*
- D) 10 minutes *(Selected in Class: 13%)*

**Correct Answer: C (5 minutes)**

#### Step-by-Step Mathematical Calculation:
1. **Balls per Over**:
   $$\text{Legal Deliveries} = 6 \text{ balls}$$
2. **Standard Ball Duration**:
   $$\text{Average Time per Ball} = 45 \text{ seconds}$$
3. **Raw Ball Time**:
   $$\text{Ball Time} = 6 \times 45 \text{ seconds} = 270 \text{ seconds} = 4.5 \text{ minutes}$$
4. **Inter-Over Transition Overhead**:
   Field repositioning, bowler rotation, and umpire checks add $\approx 30 \text{ seconds}$.
5. **Total Duration**:
   $$\text{Total Time per Over} = 270\text{ s} + 30\text{ s} = 300 \text{ seconds} = \mathbf{5 \text{ minutes}}$$

---

### Question 4: Cricket Scorecard JSON Payload Sizing
**How big should a cricket scorecard JSON be? (Detailed scores, player stats, etc.)**
- A) 5 MB *(Selected in Class: 4%)*
- B) 10 MB *(Selected in Class: 4%)*
- C) 3 KB *(Selected in Class: 91%)*
- D) 1 TB *(Selected in Class: 0%)*

**Correct Answer: C (3 KB)**

#### Deep Technical Explanation:
- A complete international match scorecard contains:
  - 2 Teams (names, short codes, toss, status)
  - 22 Players (names, IDs, batting status, bowling figures)
  - Current innings score, overs, wickets, run rate, required run rate
  - Fall of wickets list and recent ball outcomes
- Formatted as clean, structured JSON, this entire data structure occupies strictly **~3 KB** of uncompressed text.
- **Why this matters for TCP**:
  - The default Initial Congestion Window ($initcwnd$) in modern Linux kernels (RFC 6928) is **10 MSS** ($\approx 10 \times 1460 \text{ bytes} = \mathbf{14.6 \text{ KB}}$).
  - A **3 KB payload fits entirely inside a single TCP round trip** without needing window expansion or triggering TCP slow-start delays!

---

### Question 5: Scorecard Payload Reduction Techniques
**How can we reduce the size of this scorecard JSON? (Choose from following)**
- A) Reduce the team size *(Selected in Class: 0%)*
- B) Reduce the number of overs *(Selected in Class: 7%)*
- C) Compress the JSON *(Selected in Class: 90%)*
- D) Get them out quickly *(Selected in Class: 3%)*

**Correct Answer: C (Compress the JSON)**

#### Technical Mechanics:
- Standard HTTP compression algorithms (Gzip or Brotli) exploit the highly repetitive structural syntax of JSON (keys like `"name"`, `"runs"`, `"wickets"` repeating 22 times).
- Gzipping a 3 KB scorecard compresses it down to **$< 800\text{ bytes}$** ($>73\%$ size reduction), allowing it to travel comfortably inside a single minimal IP packet.
- Advanced optimization: Using **JSON Delta / RFC 6902 JSON Patch** to send only the 30-byte diff (e.g. `{"runs": 4, "total": 184}`) on subsequent updates.

---

### Question 6: Delivery Mechanisms for Live Data
**Which of the following options could work for delivering scorecard JSON to clients (apps / web pages)?**
- A) 42 *(Selected in Class: 4%)*
- B) Persistent connections *(Selected in Class: 93%)*
- C) Send a pigeon *(Selected in Class: 4%)*

**Correct Answer: B (Persistent connections)**

#### Technical Mechanics:
- **Persistent Connection Protocols**:
  - **WebSockets (`ws://`, `wss://`)**: Full-duplex persistent TCP connection after HTTP/1.1 `101 Switching Protocols` handshake.
  - **Server-Sent Events (SSE)**: Unidirectional persistent HTTP stream (`text/event-stream`) allowing the server to push new score events the millisecond a ball is recorded.
  - **HTTP/2 & HTTP/3 Streaming**: Long-lived streams multiplexed over a single persistent transport connection.

---

### Question 7: Economic Comparison: Persistent Connections vs Polling
**Will it be costlier to do persistent connections or polling for score cards?**
- A) Persistent connections *(Selected in Class: 22%)*
- B) Polling *(Selected in Class: 33%)*
- C) It depends *(Selected in Class: 44%)*

**Correct Answer: C (It depends)**

#### Deep Architectural Tradeoff Breakdown:
The cost equation fundamentally reverses depending on **scale** and **caching topology**:

1. **When Polling is Far Cheaper (The 50 Million Hotstar Scenario)**:
   - If 50 million concurrent fans watch an IPL final:
   - **Persistent Connections**: 50 million held WebSocket sockets require $50\text{M} \times 4\text{ KB} = \mathbf{200 \text{ GB RAM}}$ just for kernel socket buffers, plus thousands of active stateful gateway servers.
   - **Polling + CDN**: All 50 million users poll a single static JSON URL via CDN edge PoPs. With a 1-second CDN cache, the CDN absorbs 99.999% of requests. The origin backend sees only **1 request per second**! Polling via CDN is infinitely cheaper and simpler to scale.

2. **When Persistent Connections are Cheaper (Small to Mid Scale, High Update Cadence)**:
   - If 10,000 internal traders track high-frequency stock updates (10 updates/sec):
   - Polling generates 100,000 HTTP request/response headers per second, wasting massive header bandwidth and CPU.
   - A single persistent WebSocket connection delivers raw pushed frames with only a **2-byte framing overhead**.

---

### Question 8: The CDN Dynamic Content Paradox
**Scorecards change so frequently, will it help to add a CDN?**
- A) Yes *(Selected in Class: 21%)*
- B) No *(Selected in Class: 79%)*

**Correct Answer: A (Yes!)**

#### Why 79% of Students Picked "No" & The Mathematics of Microcaching:
- **The Student Intuition Trap**: 79% of students reason: *"Scorecards change every ball. If data changes constantly, cached data will be stale, so a CDN is useless!"*
- **The Mathematical Reality of Scale**:
  - Assume **30 Million concurrent users** polling the scorecard every **3 seconds**:
    $$\text{Total Request Rate} = \frac{30,000,000}{3} = \mathbf{10,000,000 \text{ requests/second}}$$
  - Without a CDN, your backend servers must process **10 Million requests every second**, immediately crashing all application databases.
  - **With a CDN configured with a 1-second Microcache (`Cache-Control: public, max-age=1`)**:
    - During any given 1-second window, the first request reaches the origin.
    - All remaining $9,999,999$ requests in that second are served directly from edge memory!
    $$\mathbf{\text{Origin Request Rate}} = \mathbf{1 \text{ request/second (per edge PoP)}}!$$
    $$\mathbf{\text{Origin Offload Ratio}} = \mathbf{99.9999\%}!$$
- Even for dynamic data that changes every 20 seconds, microcaching for 1 or 2 seconds provides **astronomical origin shielding**.

---

### Question 9: Conditional GET (`ETag` / `If-Modified-Since`)
**Should we implement `If-Modified-Since` / `ETag` match for CDN optimization?**
- A) Yes *(Selected in Class: 88%)*
- B) No *(Selected in Class: 12%)*

**Correct Answer: A (Yes)**

#### Technical Mechanics:
- When an edge cache's TTL expires, it doesn't need to re-download the entire 3 KB JSON.
- It sends a conditional upstream request:
  ```http
  GET /scorecard.json HTTP/1.1
  If-None-Match: "v148-over12.4"
  ```
- If the ball has not yet been bowled, the origin responds with:
  ```http
  HTTP/1.1 304 Not Modified
  ```
- A `304 Not Modified` response has **zero body payload** ($\approx 200\text{ bytes}$ of headers), instantly refreshing the edge cache while saving 100% of body transmission bandwidth.

---

### Question 10: Caching HLS Child Manifests
**Can I cache a child manifest?**
- A) Yes *(Selected in Class: 83%)*
- B) No *(Selected in Class: 17%)*

**Correct Answer: A (Yes)**

#### Deep Technical Explanation:
- In HTTP Live Streaming (HLS), the **Child (Media) Playlist** (`chunklist.m3u8`) lists the exact media segments (e.g. `segment_101.ts`, `segment_102.ts`).
- In live streaming, the child manifest is updated every time a new video segment is encoded (e.g. every 6 seconds).
- **Can it be cached? YES!** 
  - It is cached with a **short TTL equal to half the segment target duration** (e.g. $\text{TTL} = 2\text{ to } 3\text{ seconds}$).
  - If 50 million video players could not cache the child manifest, every video player would hit the origin origin storage 50 million times every few seconds, causing immediate collapse!

---

### Question 11: Absorbing Network Jitter via Playout Buffering
**If there is a new video segment every 5 seconds, how do we handle an intermittent internet connection failure? (slow downloads, failures)**
- A) Buffer some segments in advance *(Selected in Class: 94%)*
- B) Retry a failed segment *(Selected in Class: 0%)*
- C) Switch to audio mode *(Selected in Class: 6%)*

**Correct Answer: A (Buffer some segments in advance)**

#### Video Engineering Mechanics:
- Every HLS/DASH video player implements a **Forward Playout Buffer**:
  - The player starts playback only after pre-downloading **3 to 4 segments** in advance ($15\text{–}20\text{ seconds}$ of video buffered in device RAM).
  - While the user watches Segment 1, the player is already downloading Segment 4 in the background.
  - If a transient network glitch, cellular tower handover, or TCP retransmission causes a download to stall for 8 seconds, the player continues rendering from its forward playout buffer without stuttering, rebuffering, or dropping playback frames!

---

# Section 2: Advanced High-Yield Practice Bank (With Step-by-Step Math)

---

### Question 12: Hotstar 50M Persistent Socket Memory Footprint
**Suppose a video streaming platform attempts to serve 50 Million concurrent cricket viewers using persistent WebSockets on Linux servers. If each TCP socket requires an absolute minimum kernel buffer allocation of $4\text{ KB}$ for `rmem` and `wmem` (plus socket metadata), how much total RAM is consumed across the server fleet purely by socket file descriptors?**
- A) 2 GB
- B) 20 GB
- C) 200 GB
- D) 2 TB

**Correct Answer: C (200 GB)**

#### Step-by-Step Mathematical Calculation:
1. **Total Concurrent Connections**:
   $$N = 50,000,000 \text{ sockets}$$
2. **Memory per Socket**:
   $$M = 4 \text{ KB} = 4 \times 10^3 \text{ bytes} \quad (\text{or } 4096 \text{ bytes})$$
3. **Total Memory Consumed**:
   $$\text{Total RAM} = 50,000,000 \times 4096 \text{ bytes} = 204,800,000,000 \text{ bytes} \approx \mathbf{200 \text{ GB}}$$
- **Significance**: 200 GB is strictly the bare minimum kernel socket metadata. When userspace process runtimes (V8, JVM, or Go goroutine stacks of $2\text{–}8\text{ KB}$ per connection) are included, the actual memory footprint easily reaches **$500\text{ GB to } 1\text{ TB}$**, requiring a massive server fleet compared to stateless CDN caching.

---

### Question 13: S3 5,500 GET/s Prefix Limit & Sharding Math
**AWS S3 enforces a hard limit of $5,500$ GET requests per second per partitioned prefix. In a live sporting event with 50 Million viewers, a new 6-second video segment is published. All 50 Million players request this segment over a 6-second window ($8,333,333\text{ req/s}$). If requests are routed directly to S3 without a CDN, what is the minimum number of distinct S3 prefixes required to prevent `503 Slow Down` throttling?**
- A) 152 prefixes
- B) 1,516 prefixes
- C) 9,091 prefixes
- D) 500 prefixes

**Correct Answer: B (1,516 prefixes)**

#### Step-by-Step Mathematical Calculation:
1. **Compute Required Requests per Second**:
   $$\text{Peak Request Rate} = \frac{50,000,000 \text{ requests}}{6 \text{ seconds}} \approx \mathbf{8,333,333 \text{ req/sec}}$$
2. **S3 Limit per Prefix**:
   $$\text{Capacity per Prefix} = 5,500 \text{ req/sec}$$
3. **Calculate Required Prefixes**:
   $$\text{Prefixes} = \left\lceil \frac{8,333,333}{5,500} \right\rceil = \lceil 1,515.15 \rceil = \mathbf{1,516 \text{ partitioned prefixes}}$$
- **Architectural Lesson**: This demonstrates why serving live video segments directly from cloud object storage (S3/GCS) is impossible without an intermediate CDN caching tier!

---

### Question 14: The 11 KB S3 Break-Even Line
**On AWS S3, a GET request costs $\approx \$0.0004$ per $1,000$ requests ($\$0.0000004$ per request). Outbound data transfer costs $\approx \$0.036$ per GB ($\$3.35 \times 10^{-11}$ per byte). At what exact payload size does the cost of the S3 GET request fee equal the cost of the transferred bandwidth bytes?**
- A) ~1.1 KB
- B) ~11.9 KB
- C) ~119 KB
- D) ~1.1 MB

**Correct Answer: B (~11.9 KB / The Famous "11 KB Line")**

#### Step-by-Step Mathematical Derivation:
1. **Cost of 1 S3 GET Request**:
   $$C_{\text{req}} = \frac{\$0.0004}{1,000} = \$4.0 \times 10^{-7}$$
2. **Cost per Transferred Byte**:
   $$C_{\text{byte}} = \frac{\$0.036}{10^9 \text{ bytes}} = \$3.6 \times 10^{-11} \text{ per byte}$$
3. **Set $C_{\text{req}} = C_{\text{byte}} \times S$ and Solve for Size ($S$)**:
   $$S = \frac{C_{\text{req}}}{C_{\text{byte}}} = \frac{4.0 \times 10^{-7}}{3.6 \times 10^{-11}} = 11,111 \text{ bytes} \approx \mathbf{11.1 \text{ to } 11.9 \text{ KB}}$$
- **Architectural Principle**: For any request smaller than $11\text{ KB}$ (such as a 3 KB scorecard), **you are paying more for the HTTP GET request fee than for the actual data transfer!** This is why polling micro-objects directly from S3 is financially catastrophic.

---

### Question 15: HLS End-to-End Live Latency Budget
**A live sports broadcast encodes video into $6\text{-second}$ segments. The HLS specification mandates that a client video player must buffer at least $3$ full segments before initiating playback. If video encoding takes $1.5\text{ seconds}$ and CDN edge propagation takes $0.5\text{ seconds}$, what is the total glass-to-glass latency of the stream?**
- A) 6 seconds
- B) 18 seconds
- C) 20 seconds
- D) 36 seconds

**Correct Answer: C (20 seconds)**

#### Step-by-Step Mathematical Calculation:
1. **Player Buffer Delay**:
   $$\text{Buffer Delay} = 3 \text{ segments} \times 6 \text{ seconds/segment} = \mathbf{18 \text{ seconds}}$$
2. **Encoding Pipeline Latency**:
   $$\text{Encoder Delay} = \mathbf{1.5 \text{ seconds}}$$
3. **CDN Distribution Delay**:
   $$\text{CDN Delay} = \mathbf{0.5 \text{ seconds}}$$
4. **Total Glass-to-Glass Latency**:
   $$\text{Total Latency} = 18\text{ s} + 1.5\text{ s} + 0.5\text{ s} = \mathbf{20.0 \text{ seconds}}$$
- **The "Spoiler Problem"**: A user watching on TV (cable latency $\approx 4\text{ seconds}$) cheers a six 16 seconds before their neighbor on standard HLS sees the bowler deliver the ball!

---

### Question 16: Adaptive Bitrate (ABR) Throughput Margin Selection
**An ABR video player monitors network download throughput. The streaming server provides 3 bitrate profiles: 1080p ($5.0\text{ Mbps}$), 720p ($2.5\text{ Mbps}$), and 480p ($1.0\text{ Mbps}$). If the video player applies a standard safety headroom factor of $20\%$ ($\text{Target Bitrate} \le 0.80 \times \text{Measured Throughput}$) and the user's available bandwidth drops to $3.0\text{ Mbps}$, which profile will the ABR algorithm select?**
- A) 1080p (5.0 Mbps)
- B) 720p (2.5 Mbps)
- C) 480p (1.0 Mbps)
- D) Stall and rebuffer

**Correct Answer: C (480p / 1.0 Mbps)**

#### Step-by-Step Mathematical Calculation:
1. **Compute Usable Bitrate Ceiling**:
   $$\text{Max Allowed Bitrate} = \text{Measured Bandwidth} \times (1 - 0.20)$$
   $$\text{Max Allowed Bitrate} = 3.0 \text{ Mbps} \times 0.80 = \mathbf{2.4 \text{ Mbps}}$$
2. **Compare Against Available Ladders**:
   - 1080p: $5.0 \text{ Mbps} > 2.4 \text{ Mbps}$ (Unsafe)
   - 720p: $2.5 \text{ Mbps} > 2.4 \text{ Mbps}$ (Unsafe - exceeds 2.4 Mbps ceiling by 100 kbps!)
   - 480p: $1.0 \text{ Mbps} \le 2.4 \text{ Mbps}$ (**Safe**)
3. **Selection**: The algorithm steps down to **480p** to ensure the forward playout buffer is not drained.

---

### Question 17: MPEG-2 TS Framing Overhead
**Traditional HLS segments use MPEG-2 Transport Stream (`.ts`) container files comprising fixed $188\text{-byte}$ packets. Each packet contains a $4\text{-byte}$ header. What is the minimum framing overhead percentage incurred purely by MPEG-TS packet headers?**
- A) 0.5%
- B) 2.13%
- C) 4.0%
- D) 8.5%

**Correct Answer: B (2.13%)**

#### Step-by-Step Mathematical Calculation:
1. **Packet Dimensions**:
   $$\text{Header Size} = 4 \text{ bytes}$$
   $$\text{Total Packet Size} = 188 \text{ bytes}$$
   $$\text{Payload Data} = 188 - 4 = 184 \text{ bytes}$$
2. **Header Overhead Percentage**:
   $$\text{Overhead} = \frac{4 \text{ bytes}}{188 \text{ bytes}} \times 100 = \mathbf{2.1276\%} \approx \mathbf{2.13\%}$$
- **Modern Shift**: This overhead (plus audio/video demuxing complexity) led the industry to adopt **fMP4 (fragmented MP4) / CMAF**, which reduces packetization overhead to $< 0.1\%$.

---

### Question 18: Conditional GET Bandwidth Savings
**A client polls a 3 KB scorecard every 2 seconds for an entire 3-hour match ($5,400\text{ polls}$). If the score only changes once every 45 seconds ($240\text{ updates}$ total) and unchanged requests return a $300\text{-byte}$ `304 Not Modified` header, how much total bandwidth is consumed with conditional GET vs unconditional polling?**
- A) Unconditional: 16.2 MB; Conditional: ~2.27 MB (86% savings)
- B) Unconditional: 5.4 MB; Conditional: 5.4 MB (0% savings)
- C) Unconditional: 16.2 MB; Conditional: 16.2 MB
- D) Unconditional: 100 MB; Conditional: 10 MB

**Correct Answer: A (Unconditional: 16.2 MB; Conditional: ~2.27 MB / 86% savings)**

#### Step-by-Step Mathematical Calculation:
1. **Total Polls**:
   $$\text{Total Polls} = \frac{3 \times 3600 \text{ s}}{2 \text{ s}} = \mathbf{5,400 \text{ polls}}$$
2. **Unconditional Polling (Always 3 KB)**:
   $$\text{Bandwidth} = 5,400 \times 3 \text{ KB} = \mathbf{16,200 \text{ KB}} = \mathbf{16.2 \text{ MB}}$$
3. **Conditional Polling**:
   - Changed polls ($240$ times): $240 \times 3 \text{ KB} = 720 \text{ KB}$.
   - Unchanged polls ($5,400 - 240 = 5,160$ times): $5,160 \times 0.3 \text{ KB} = 1,548 \text{ KB}$.
   - Total = $720 + 1,548 = \mathbf{2,268 \text{ KB}} \approx \mathbf{2.27 \text{ MB}}$.
4. **Savings**:
   $$\text{Savings} = \frac{16.2 - 2.27}{16.2} \times 100 = \mathbf{86.0\% \text{ bandwidth saved}}.$$

---

### Question 19: AWS Full Jitter Exponential Backoff Formula
**Under the AWS Full Jitter retry algorithm (RFC 9110 compliant):
$$t = \text{random}(0, \min(M, B \times 2^i))$$
Where base backoff $B = 100\text{ ms}$, maximum backoff $M = 3000\text{ ms}$, and retry attempt $i = 4$. What is the range of possible sleep durations for this retry?**
- A) Exactly 1600 ms
- B) Uniformly distributed between 0 ms and 1600 ms
- C) Uniformly distributed between 0 ms and 3000 ms
- D) Exactly 3000 ms

**Correct Answer: B (Uniformly distributed between 0 ms and 1600 ms)**

#### Step-by-Step Mathematical Calculation:
1. **Exponential Term**:
   $$B \times 2^i = 100 \text{ ms} \times 2^4 = 100 \times 16 = \mathbf{1600 \text{ ms}}$$
2. **Cap by Maximum ($M$)**:
   $$\min(3000, 1600) = \mathbf{1600 \text{ ms}}$$
3. **Randomization**:
   $$t \in [0, 1600\text{ ms}]$$
- **Significance**: Unlike deterministic backoff (which causes synchronized retry waves), Full Jitter spreads retry attempts uniformly across the time window, completely eliminating **Thundering Herd** spikes.

---

### Question 20: GOP Keyframe Alignment (`-g 120`)
**In an FFmpeg adaptive bitrate encoding pipeline producing $5\text{-second}$ segments for a $24\text{ fps}$ live video stream, what must the Group of Pictures (GOP) size (`-g` parameter) be set to so that every segment starts with an IDR keyframe?**
- A) `-g 24`
- B) `-g 60`
- C) `-g 120`
- D) `-g 240`

**Correct Answer: C (`-g 120`)**

#### Step-by-Step Mathematical Calculation:
1. **Formula**:
   $$\text{GOP Size} = \text{Frame Rate (fps)} \times \text{Segment Duration (seconds)}$$
2. **Evaluate**:
   $$\text{GOP Size} = 24 \text{ frames/second} \times 5 \text{ seconds} = \mathbf{120 \text{ frames}}$$
- **Trap**: If `-g` is misconfigured to 100, segment boundaries will cut across P/B-frames instead of I-frames (IDR), causing client video decoders to corrupt or display black screens when switching quality levels.

---

### Question 21: Circuit Breaker Half-Open State
**In the Circuit Breaker pattern (Martin Fowler), what triggers the transition from the `OPEN` state to the `HALF-OPEN` state?**
- A) A successful request from a client
- B) The expiration of a predefined sleep/cooldown timer ($T_{\text{reset}}$)
- C) Restarting the client application
- D) Receiving a 200 OK from the database

**Correct Answer: B (Expiration of the reset cooldown timer)**

#### State Machine Flow:
```text
┌──────────┐   Failure Threshold Exceeded    ┌──────────┐
│  CLOSED  │ ──────────────────────────────> │   OPEN   │
└──────────┘                                 └──────────┘
     ▲                                             │
     │ All Canary Requests Pass                    │ Cooldown Timer
     │                                             │ Expires
┌──────────┐                                       ▼
│   PASS   │ <────────────────────────────── ┌──────────┐
└──────────┘      Canary Probe Fails         │HALF-OPEN │
                                             └──────────┘
```

---

### Question 22: Fastly `stale-if-error` (RFC 5861)
**When an origin returns an HTTP 500 Internal Server Error, which Cache-Control directive allows an edge CDN proxy to serve an expired cached object for up to 24 hours while shielding the user?**
- A) `Cache-Control: no-cache`
- B) `Cache-Control: max-age=60, stale-if-error=86400`
- C) `Cache-Control: private, must-revalidate`
- D) `Cache-Control: s-maxage=0`

**Correct Answer: B (`Cache-Control: max-age=60, stale-if-error=86400`)**

#### Mathematical Unit Conversion:
- $86,400 \text{ seconds} = \frac{86,400}{3600} = \mathbf{24 \text{ hours}}$.
- The CDN serves fresh content for 60 seconds; if the origin fails anytime during the next 86,400 seconds, the edge serves the stale cached asset without passing the 500 error to users.

---

### Question 23: WebSocket 4-Byte Masking Key Overhead
**In RFC 6455 WebSockets, client-to-server frames MUST include a 4-byte masking key to prevent cache poisoning in transparent proxies. For a minimal 2-byte JSON ping (`{}`), what is the percentage framing overhead added by the WebSocket header (2 bytes base + 4 bytes masking)?**
- A) 50%
- B) 300%
- C) 200%
- D) 75%

**Correct Answer: B (300% overhead)**

#### Step-by-Step Mathematical Calculation:
1. **Payload Size**:
   $$\text{Payload} = 2 \text{ bytes}$$
2. **WebSocket Header Size (with masking)**:
   $$\text{Base Header} = 2 \text{ bytes}$$
   $$\text{Masking Key} = 4 \text{ bytes}$$
   $$\text{Total Framing} = 2 + 4 = \mathbf{6 \text{ bytes}}$$
3. **Overhead Ratio**:
   $$\text{Overhead} = \frac{6 \text{ bytes framing}}{2 \text{ bytes payload}} \times 100 = \mathbf{300\%}$$

---

### Question 24: S3 Negative Caching Configuration
**If an origin video packager deletes an old live video segment, causing S3 to return `404 Not Found`, what happens if the CDN does NOT configure negative caching (`proxy_cache_valid 404 1s;`)?**
- A) The video plays in reverse
- B) Millions of video players repeatedly request the missing segment, bombarding S3 with uncached 404 queries and triggering account-level rate throttling
- C) S3 automatically regenerates the video
- D) The CDN crashes

**Correct Answer: B (Uncached 404 request bombardment / Thundering Herd on Object Storage)**

---

### Question 25: LL-HLS vs Traditional HLS Latency
**Low-Latency HLS (LL-HLS) reduces live streaming latency from $20\text{ seconds}$ down to under $3\text{ seconds}$ primarily by:**
- A) Reducing video resolution to 240p
- B) Dividing segments into tiny CMAF partial chunks ($200\text{–}500\text{ ms}$) and transmitting them using HTTP/2 chunked transfer before the full segment finishes encoding
- C) Switching from TCP to UDP broadcast
- D) Removing audio tracks

**Correct Answer: B (CMAF Partial Chunking and HTTP/2 Pre-announced push)**

---

### Question 26: Multi-CDN DNS Steering vs Player Steering
**Why is client-side video player steering superior to DNS-based steering for Multi-CDN switching during a mid-stream CDN outage?**
- A) DNS steering is illegal under RFC 1035
- B) DNS TTL caching (at local ISP resolvers) delays failover by 60–300 seconds, whereas the video player can instantly switch the next segment download URL to CDN B within 200 ms
- C) DNS does not support IPv6
- D) Video players do not use IP addresses

**Correct Answer: B (ISP DNS TTL caching delay vs millisecond-level player URL switching)**
