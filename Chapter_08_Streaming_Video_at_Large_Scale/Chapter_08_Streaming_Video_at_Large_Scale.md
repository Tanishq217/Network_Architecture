# Chapter 08: Streaming Video at Large Scale — The Complete System

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 8 / Week 8 of 8  
> **Core Theme**: *"Every live stream is a scorecard with bigger bytes. It starts with a live cricket scorecard."*  
> **Key Protocols & Concepts**: Real-time event delivery architectures (Pull vs. Push), Polling with conditional ETags, Long Polling, Server-Sent Events (SSE), WebSockets (RFC 6455), MQTT PubSub (EMQX), How Hotstar scaled to 50M concurrent sockets, The cost of held connections (Little's Law applied twice), Cost analysis across 4 architectures (S3, NGINX+ALB, CloudFront, Push Fleet), S3 prefix request rate limits (5,500 GET/s), Request fees vs. Byte fees (The 11 KB break-even line), The Stacked TTL staleness problem, Cache stampede mitigation (`proxy_cache_lock`, `proxy_cache_use_stale`), Negative caching rules, Delta updates & JSON patches, HTTP Live Streaming (HLS / RFC 8216) architecture, Master vs. Media playlists, MPEG-2 Transport Stream (188-byte packets, 13-bit PIDs) vs. fMP4/CMAF, Optimal segment duration trade-offs, Player buffer mechanics, The HLS latency budget (20s vs. Low-Latency HLS ~3s), The Spoiler Problem & Wall-Clock synchronization (`#EXT-X-PROGRAM-DATE-TIME`), Adaptive Bitrate (ABR: Throughput-based vs. Buffer-based), Multi-CDN steering & failover, The Stale Playlist / Deleted Segment 404 trap, Live stream caching matrix, FFmpeg encoding commands & keyframe alignment (`-g 120`), The Four Core Invariants of Video Systems, and 15 exam-grade MCQs.

---

## Table of Contents
1. [The Challenge: Delivering One Live Event to 50 Million Devices](#1-the-challenge-delivering-one-live-event-to-50-million-devices)
2. [Two Architectural Families: Pull (Client Asks) vs. Push (Server Tells)](#2-two-architectural-families-pull-client-asks-vs-push-server-tells)
   - [2.1 Pull Architecture: Polling & Conditional GETs](#21-pull-architecture-polling--conditional-gets)
   - [2.2 Push Architecture 1: Long Polling](#22-push-architecture-1-long-polling)
   - [2.3 Push Architecture 2: Server-Sent Events (SSE)](#23-push-architecture-2-server-sent-events-sse)
   - [2.4 Push Architecture 3: WebSockets (RFC 6455)](#24-push-architecture-3-websockets-rfc-6455)
   - [2.5 Push Architecture 4: MQTT PubSub & The Hotstar Case Study](#25-push-architecture-4-mqtt-pubsub--the-hotstar-case-study)
3. [The Resource & Financial Cost of Scale](#3-the-resource--financial-cost-of-scale)
   - [3.1 What a Held Socket Actually Costs in Hardware](#31-what-a-held-socket-actually-costs-in-hardware)
   - [3.2 Little's Law Applied Twice: 50,000 vs. 50,000,000 Sockets](#32-littles-law-applied-twice-50000-vs-50000000-sockets)
   - [3.3 Four Ways to Pay for One Match: Cost & Failure Analysis](#33-four-ways-to-pay-for-one-match-cost--failure-analysis)
   - [3.4 Why S3 Runs Out of Requests Before Money (The 5,500 Limit)](#34-why-s3-runs-out-of-requests-before-money-the-5500-limit)
   - [3.5 Request Fees vs. Byte Fees: The 11 KB Break-Even Line](#35-request-fees-vs-byte-fees-the-11-kb-break-even-line)
4. [Making Polling Survive at Scale](#4-making-polling-survive-at-scale)
   - [4.1 Conditional GETs (304 Not Modified) Between CDN Tiers](#41-conditional-gets-304-not-modified-between-cdn-tiers)
   - [4.2 The Stacked TTL Problem: Cumulative Staleness](#42-the-stacked-ttl-problem-cumulative-staleness)
   - [4.3 Mitigating Expiry Stampedes: Request Collapsing & Background Refresh](#43-mitigating-expiry-stampedes-request-collapsing--background-refresh)
   - [4.4 Negative Caching Rules for 404 and 5xx Responses](#44-negative-caching-rules-for-404-and-5xx-responses)
   - [4.5 Delta Updates: Sending JSON Patches Instead of Full Cards](#45-delta-updates-sending-json-patches-instead-of-full-cards)
   - [4.6 Why a Versioned Scorecard IS a Video Playlist](#46-why-a-versioned-scorecard-is-a-video-playlist)
5. [HTTP Live Streaming (HLS): Architecture & Wire Formats](#5-http-live-streaming-hls-architecture--wire-formats)
   - [5.1 HLS in One Picture: Decoupling Control from Data](#51-hls-in-one-picture-decoupling-control-from-data)
   - [5.2 The Master Playlist (`master.m3u8`): Rendition Menus](#52-the-master-playlist-masterm3u8-rendition-menus)
   - [5.3 The Media Playlist (`index.m3u8`): The Sliding Window Polling Rules](#53-the-media-playlist-indexm3u8-the-sliding-window-polling-rules)
   - [5.4 VOD vs. Live Playlists (`#EXT-X-ENDLIST`)](#54-vod-vs-live-playlists-ext-x-endlist)
   - [5.5 Wire Packaging: MPEG-2 Transport Streams (188-Byte Packets) vs. fMP4](#55-wire-packaging-mpeg-2-transport-streams-188-byte-packets-vs-fmp4)
   - [5.6 How Long Should a Video Segment Be? The Core Trade-Off](#56-how-long-should-a-video-segment-be-the-core-trade-off)
   - [5.7 Why Playlist Traffic is Harder Than Video Segment Traffic](#57-why-playlist-traffic-is-harder-than-video-segment-traffic)
6. [Playback Buffers, Latency Budgets & Spoilers](#6-playback-buffers-latency-budgets--spoilers)
   - [6.1 The Player Playhead Buffer: Trade-offs](#61-the-player-playhead-buffer-trade-offs)
   - [6.2 The HLS Latency Budget: Why Broadcasts Lag by 20 Seconds](#62-the-hls-latency-budget-why-broadcasts-lag-by-20-seconds)
   - [6.3 Low-Latency HLS (LL-HLS): Shrinking Latency to ~3 Seconds](#63-low-latency-hls-ll-hls-shrinking-latency-to-3-seconds)
   - [6.4 The Spoiler Problem & Wall-Clock Synchronization](#64-the-spoiler-problem--wall-clock-synchronization)
   - [6.5 Adaptive Bitrate (ABR): Throughput-Based vs. Buffer-Based Algorithms](#65-adaptive-bitrate-abr-throughput-based-vs-buffer-based-algorithms)
7. [Multi-CDN Streaming & Live Caching Rules](#7-multi-cdn-streaming--live-caching-rules)
   - [7.1 Multi-CDN Topologies & Content Steering](#71-multi-cdn-topologies--content-steering)
   - [7.2 The Stale Playlist and Deleted Segment Trap](#72-the-stale-playlist-and-deleted-segment-trap)
   - [7.3 Live Stream Caching Matrix for NGINX & CDNs](#73-live-stream-caching-matrix-for-nginx--cdns)
8. [Generating the Stream: Real-World FFmpeg Packaging](#8-generating-the-stream-real-world-ffmpeg-packaging)
   - [8.1 A Simple Single-Bitrate HLS Command](#81-a-simple-single-bitrate-hls-command)
   - [8.2 The Full Adaptive Bitrate (ABR) Ladder Command](#82-the-full-adaptive-bitrate-abr-ladder-command)
   - [8.3 The GOP Keyframe Alignment Rule (`-g 120`)](#83-the-gop-keyframe-alignment-rule--g-120)
9. [Synthesis: The Complete Curriculum in a Single Match](#9-synthesis-the-complete-curriculum-in-a-single-match)
10. [The Four Core Invariants of Video Systems](#10-the-four-core-invariants-of-video-systems)
11. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#11-comprehensive-review--practice-mcqs-with-detailed-explanations)
12. [Session 8 Summary Cheat Sheet](#12-session-8-summary-cheat-sheet)

---

## 1. The Challenge: Delivering One Live Event to 50 Million Devices

Consider the engineering requirements of broadcasting a live cricket World Cup final to 50 million concurrent mobile viewers:
- **Game Rhythm**: 25–45 seconds between individual balls; an over concludes every 4–5 minutes.
- **Audience Scale**: **50 Million peak concurrent devices** (averaging 20 million active users across a 4-hour match).
- **Data Footprint**:
  - Live Scorecard: ~4–5 KB of raw JSON ($\approx 1\text{ KB}$ gzipped).
  - Video Manifest: $\approx 228\text{ bytes}$ gzipped, re-polled every 4 seconds.
  - Video Media Segments: $\approx 823\text{ KB}$ per 4-second chunk (at 480p resolution).

```
THE SCALE OF A 4-HOUR LIVE MATCH (50M PEAK / 20M AVERAGE CONCURRENT USERS):
- Scorecard Polling (every 30s) : 9.6 Billion requests per match (1.67 Million req/sec at peak!)
- Scorecard Data Transferred   : 9.6 Terabytes of gzipped JSON
- Video Media Requests (4s seg) : 72 Billion segment downloads
- Video Data Transferred       : Tens of Petabytes across global transit networks
```

Delivering this scale requires combining every concept covered in the curriculum: OS socket limits, non-blocking event loops, caching topologies, framing protocols, and distributed failure handling.

---

## 2. Two Architectural Families: Pull (Client Asks) vs. Push (Server Tells)

To distribute rapidly updating data to millions of clients, systems choose between two fundamental architectural models:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   THE TWO REAL-TIME DELIVERY FAMILIES                                  │
├───────────────────────────────────┬────────────────────────────────────────────────────────────────────┤
│ 1. PULL ARCHITECTURE (Non-Persistent) │ 2. PUSH ARCHITECTURE (Persistent)                              │
├───────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ - Client initiates requests periodically. │ - Client establishes a long-lived, continuous connection.  │
│ - Short-lived, stateless HTTP/1.1 or H2 requests.│ - Connection remains open for the entire duration of session.│
│ - Origin generates one file; CDNs cache it globally.│ - Server pushes updates the instant new events occur.    │
│ - Any server or edge PoP can answer any client. │ - Server must track state for every active client socket.│
│ - Highly scalable via existing web infrastructure.│ - Cannot be cached by standard CDN reverse proxies.       │
└───────────────────────────────────┴────────────────────────────────────────────────────────────────────┘
```

### 2.1 Pull Architecture: Polling & Conditional GETs
The client polls the endpoint on a fixed or jittered schedule:
```javascript
let etag = null;
const EVERY = 15000; // Poll every 15 seconds

async function poll() {
  const headers = etag ? { 'If-None-Match': etag } : {};
  const res = await fetch('/match/412/score.json', { headers });

  if (res.status === 200) {
    etag = res.headers.get('ETag');
    render(await res.json());
  }
  // If 304 Not Modified: reuse existing local data with zero body transfer!

  const jitter = 0.8 + Math.random() * 0.4; // ±20% jitter prevents synchronized lockstep
  setTimeout(poll, EVERY * jitter);
}
poll();
```
- **ETag Validation**: If content is unchanged, the server returns **`304 Not Modified`** with an empty body, avoiding re-downloading the JSON payload.
- **Jitter ($\pm 20\%$)**: Randomizes poll intervals so 50 million devices do not query the network on the exact same second.

---

### 2.2 Push Architecture 1: Long Polling
The client sends an HTTP request, but the server **delays its response until new data is available** or a timeout expires:
```http
GET /match/412/score?after=1841 HTTP/1.1
```
- The server holds the request open in memory.
- When ball #1842 is bowled (or 25 seconds elapse), the server returns `200 OK` with the update.
- The client receives the payload and immediately issues a new request for `?after=1842`.
- **The Trade-off**: Holding millions of HTTP requests open consumes file descriptors and load balancer connection capacity. If an event occurs (e.g., a wicket falls), millions of held requests complete simultaneously, prompting millions of immediate follow-up requests that create an instant thundering herd.

---

### 2.3 Push Architecture 2: Server-Sent Events (SSE)
A standardized mechanism (RFC 8895) where the server keeps a single HTTP connection open indefinitely and streams text events:
```http
GET /match/412/events HTTP/1.1
Accept: text/event-stream
Last-Event-ID: 1841

HTTP/1.1 200 OK
Content-Type: text/event-stream

id: 1842
event: ball
data: {"over":"15.4", "runs":4, "total":"146/4"}

id: 1843
event: ball
data: {"over":"15.5", "runs":0, "total":"146/4"}
```
- **Framing**: Uses double newlines (`\n\n`) as delimiters between messages.
- **Reconnection State**: If the connection drops, the browser automatically reconnects, passing `Last-Event-ID: 1841` so the server can replay missed events.
- **Limitation**: Unidirectional (server-to-client only) and holds persistent HTTP sockets on backend infrastructure.

---

### 2.4 Push Architecture 3: WebSockets (RFC 6455)
Upgrades an existing HTTP connection into a full-duplex, framed binary socket:
```http
GET /live HTTP/1.1
Host: score.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
- **Handshake Validation**: `Sec-WebSocket-Accept` is computed as `Base64(SHA-1(Key + GUID))`, preventing unintended protocol cross-talk.
- **Binary Framing**: Messages use minimal binary frame headers (1 to 10 bytes) with client-to-server masking to avoid confusing intermediate proxy caches.

---

### 2.5 Push Architecture 4: MQTT PubSub & The Hotstar Case Study
MQTT is a lightweight publish-subscribe protocol using compact binary framing:
- **Architecture**: A publisher posts a message to a topic (`match/412/score`); the broker fans it out to all subscribed connections.
- **Retained Messages**: The broker stores the most recent message on a topic, immediately delivering it to new subscribers upon connection.
- **QoS 1 (At-Least-Once Delivery)**: Ensures message delivery, relying on client-side message deduplication.

#### How Hotstar Handled 50 Million Concurrent Sockets:
In 2019, Hotstar engineered a real-time messaging system to deliver interactive emojis and social features during live cricket matches:
1. **Single Node (EMQX on `c5.4xlarge`)**: Handled 250,000 persistent socket connections per machine.
2. **Cluster Limits**: Splitting publishers from subscribers hit distributed database limits (Erlang Mnesia performs best with $<10$ cluster nodes).
3. **The AWS Load Balancer Limit**: Classic Elastic Load Balancers (ELBs) dropped connections past ~500,000 without pre-warming, hitting a ceiling of ~4 million connections per ELB.
4. **Five Bridged Clusters**: Deployed 5 independent clusters (8 subscriber nodes each) to scale past 10 million connections.
5. **Reverse Bridge Architecture**: Deployed an internal Go service on every subscriber node that pulled updates from the publisher, scaling the footprint to **200 nodes supporting 50 million active connections**.
- *Takeaway*: Persistent push architectures require significant operational overhead and infrastructure investment. While necessary for interactive, individualized data (like social feeds), content that is identical for all users (like scorecards or video) scales more cost-effectively through caching.

---

## 3. The Resource & Financial Cost of Scale

### 3.1 What a Held Socket Actually Costs in Hardware
Maintaining a persistent socket connection consumes operating system and memory resources for its entire duration:
- **Server Density**: Production servers can hold 100,000 to 250,000 persistent sockets per node.
- **Memory Footprint**: Each socket consumes kernel receive/transmit buffers, TLS session state, and user-space process tracking:
  $$\frac{32\text{ GiB RAM}}{250,000\text{ connections}} \approx \mathbf{137\text{ KB of RAM per connection}}$$
- **Keep-Alive Traffic**: To prevent NAT state timeouts on mobile networks, sockets require regular keep-alive pings. At a 60-second ping interval, 20 million idle phones generate:
  $$\frac{20,000,000}{60\text{ seconds}} \approx \mathbf{333,000\text{ pings per second before the first ball is bowled!}}$$
- **Load Balancer Connection Caps**: Upstream load balancers must also track every open socket, creating architectural bottlenecks at the network edge.

---

### 3.2 Little's Law Applied Twice: 50,000 vs. 50,000,000 Sockets
Recall Little's Law ($L = \lambda \times W$):

```
SCENARIO A: POLLING VIA CDN REVERSE PROXY
- Arrival Rate (λ) : 1,670,000 requests per second
- Average Duration (W): 0.030 seconds (30 ms at edge cache)
- In-Flight Concurrency (L) = 1,670,000 * 0.030 = 50,000 CONCURRENT REQUESTS

SCENARIO B: PUSH OVER HELD PERSISTENT SOCKETS
- Average Duration (W): 14,400 seconds (The entire 4-hour match!)
- In-Flight Concurrency (L) = 50,000,000 CONCURRENT HELD SOCKETS
```

Because the connection duration ($W$) shifts from milliseconds to hours, **persistent push requires managing 1,000 times more concurrent sockets** to serve the exact same audience.

---

### 3.3 Four Ways to Pay for One Match: Cost & Failure Analysis

```
┌───────────────────────────┬───────────────┬─────────────────┬───────────────────────────┬─────────────────────────────────┐
│ Architecture Design       │ Cost / Match  │ Cost / 100 Matches│ Primary Cost Driver     │ Primary Operational Failure Mode│
├───────────────────────────┼───────────────┼─────────────────┼───────────────────────────┼─────────────────────────────────┤
│ **1. S3 Direct to Phones**│ ~$4,700       │ $470,000        │ S3 GET Request Fees       │ 5,500 GET/sec per prefix cap    │
│                           │               │                 │ ($3,840 of the bill)      │ causes massive 503 errors!      │
├───────────────────────────┼───────────────┼─────────────────┼───────────────────────────┼─────────────────────────────────┤
│ **2. NGINX Fleet + ALB**  │ ~$1,200–1,800 │ $120,000–180,000│ AWS Egress ($864) +       │ 20M–50M idle Keep-Alive sockets │
│                           │               │                 │ ALB Load Balancer Units   │ overwhelm the ALB tier.         │
├───────────────────────────┼───────────────┼─────────────────┼───────────────────────────┼─────────────────────────────────┤
│ **3. CloudFront CDN**     │ ~$12,500      │ $1,250,000      │ Request Fees:             │ Does not break, but expensive   │
│                           │               │                 │ **92% of the bill**!      │ for small payloads.             │
├───────────────────────────┼───────────────┼─────────────────┼───────────────────────────┼─────────────────────────────────┤
│ **4. Push: 200x c5.4xlarge**│ ~$900–1,600 │ $90,000–160,000 │ Running large EC2 fleet   │ Reconnect storms, load balancing│
│                           │               │                 │ for peak capacity ($680)  │ failover challenges.            │
└───────────────────────────┴───────────────┴─────────────────┴───────────────────────────┴─────────────────────────────────┘
```

---

### 3.4 Why S3 Runs Out of Requests Before Money (The 5,500 Limit)
Why not host `score.json` in an Amazon S3 bucket and have clients poll S3 directly?
- **The S3 Partition Limit**: Amazon S3 supports **5,500 GET requests per second per prefix**.
- At peak, 50 million viewers polling every 30 seconds generate **1,670,000 GET requests per second**.
- This exceeds S3's per-prefix capacity by **303x**, causing S3 to return widespread **`503 Slow Down`** errors.
- Millions of clients retrying those 503 errors trigger an immediate retry storm that overwhelms client connections.

---

### 3.5 Request Fees vs. Byte Fees: The 11 KB Break-Even Line
Cloud CDN pricing models charge for two distinct dimensions:
1. **Per-Request Fees**: Typically \$0.012 per 10,000 HTTPS requests (\$1.20 per million).
2. **Per-Byte Data Transfer Fees**: Typically \$0.072 to \$0.109 per Gigabyte transferred.

```
COST PER 1,000,000 REQUESTS:
Payload Type               Size (Bytes)     Request Fee     Byte Transfer Fee   Dominant Cost Factor
────────────────────────────────────────────────────────────────────────────────────────────────────
Scorecard JSON             1 KB             $1.20           $0.11               REQUEST FEE (92% of bill)
Video Playlist (m3u8)      228 Bytes        $1.20           $0.02               REQUEST FEE (98% of bill)
480p Video Segment         823 KB           $1.20           $59.00 to $90.00    BYTE FEE (98% of bill)
```

$$\mathbf{\text{Break-Even File Size}} \approx \mathbf{11 \text{ Kilobytes}}$$
- For payloads under 11 KB, the bill is dominated by request volume.
- For payloads over 11 KB, the bill is dominated by bandwidth volume.

---

## 4. Making Polling Survive at Scale

To make polling cost-effective and resilient at high volumes, systems combine several caching optimizations:

### 4.1 Conditional GETs (304 Not Modified) Between CDN Tiers
Instead of serving full JSON documents on every poll, the origin assigns an `ETag` matching the content version:
```http
GET /match/412/score.json HTTP/1.1
If-None-Match: "v1841"

HTTP/1.1 304 Not Modified
ETag: "v1841"
Cache-Control: public, max-age=10
```
- Eliminates payload body transfer, JSON parsing, and DOM rendering when content is unchanged.
- While the user still pays the request fee and round-trip latency, **conditional checks between CDN tiers** (e.g., edge-to-shield and shield-to-origin) protect origin infrastructure by returning 304s instead of transferring full files.

---

### 4.2 The Stacked TTL Problem: Cumulative Staleness
When multiple caching layers are chained together, latency accumulates along the path:

```
THE STACKED TTL DELAY:
Origin ──> [ Shield Cache: TTL 30s ] ──> [ Edge PoP: TTL 30s ] ──> [ Client Poll: 30s ]

Maximum Staleness = Shield TTL (30s) + Edge TTL (30s) + Client Poll Interval (30s) = 90 SECONDS!
```

A viewer could be 90 seconds (several balls) behind live play.

#### The Production Rule for Freshness:
To prevent cumulative delays, configure each intermediate cache TTL to **$\le \frac{1}{2}$ of the desired freshness target**:
$$\text{Shield TTL (5s)} + \text{Edge TTL (5s)} + \text{Client Poll (15s)} = \mathbf{25\text{ seconds max staleness}}$$
- **Prefreshing**: Advanced edge configurations (such as Akamai Prefresh) asynchronously refresh cached items ~5 seconds before their TTL expires, ensuring clients always receive warm cache hits.

---

### 4.3 Mitigating Expiry Stampedes: Request Collapsing & Background Refresh
When an edge cache entry expires, thousands of concurrent requests arriving in that instant can cause a cache stampede back to the origin:

```nginx
# NGINX Edge Caching Directives for Live Data:
proxy_cache_lock on;                    # Request Collapsing: 1 fetch to origin; other requests wait
proxy_cache_use_stale updating error;   # Serve stale copy while background request refreshes
proxy_cache_background_update on;       # Asynchronous background refresh
proxy_cache_valid 200 10s;              # Cache valid responses for 10 seconds
```

- **`proxy_cache_lock on;`**: Collapses thousands of concurrent cache misses into a single origin fetch, serving waiting clients from that response.
- **`proxy_cache_use_stale updating;`**: Immediately serves the existing cached copy to users while refreshing the resource asynchronously in the background.

---

### 4.4 Negative Caching Rules for 404 and 5xx Responses
- **`5xx Server Errors`**: Cache for **1 to 2 seconds** (`proxy_cache_valid 500 502 503 504 2s;`). This shields a struggling origin from millions of immediate client retries.
- **`404 Not Found`**: **Avoid caching 404s for long durations on live content.** If an edge caches a 404 for a live segment that is still being written, it will continue returning 404 until the cache expires, stalling video playback. Set 404 cache TTLs to **1 second or less**.

---

### 4.5 Delta Updates: Sending JSON Patches Instead of Full Cards
A full scorecard JSON payload is roughly **4,117 bytes** (1,088 bytes gzipped). An individual ball update contains only score adjustments, which can be represented as an **RFC 6902 JSON Patch** of just **183 bytes gzipped**:
- **Pull Design**: Avoid using URL query strings like `/score?since=1841`. Unique query parameters generate distinct cache keys, bypassing CDN caches and flooding the origin. Instead, publish a single rolling file (`score.json`) containing the latest version number alongside a list of the last 5 updates.

---

### 4.6 Why a Versioned Scorecard IS a Video Playlist
Compare the structure of a rolling scorecard with an HLS media playlist:

```json
// score.json (Polled every 15 seconds)
{
  "version": 1842,
  "recent_balls": [
    {"v": 1839, "ball": "15.1", "runs": 1},
    {"v": 1840, "ball": "15.2", "runs": 0},
    {"v": 1842, "ball": "15.4", "runs": 4}
  ]
}
```

```m3u8
# index.m3u8 (Polled every 4 seconds)
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:1840
#EXTINF:4.000000,
seg_001840.ts
#EXTINF:4.000000,
seg_001841.ts
#EXTINF:4.000000,
seg_001842.ts
```

Both use the same architectural pattern: **a small, frequently polled index file pointing to numbered, immutable data assets that can be cached permanently**.

---

## 5. HTTP Live Streaming (HLS): Architecture & Wire Formats

Standardized by Apple in RFC 8216, **HLS (HTTP Live Streaming)** delivers video over standard HTTP infrastructure by breaking media streams into short file chunks.

### 5.1 HLS in One Picture: Decoupling Control from Data

```
[ Video Player ]
       │
       ├─ 1. Fetches Once ──────────> [ CONTROL: Master Playlist (master.m3u8) ]
       │                               Lists available renditions (720p, 480p, 360p, bitrates)
       │
       ├─ 2. Polls Every 4s ────────> [ CONTROL: Media Playlist (stream_1/index.m3u8) ]
       │                               Sliding window listing the latest 10 segments
       │
       └─ 3. Fetches Once per chunk ─> [ DATA: Media Segments (stream_1/seg_000011.ts) ]
                                       Immutable 4-second video files (823 KB); cached forever!
```

---

### 5.2 The Master Playlist (`master.m3u8`): Rendition Menus
The client reads the master playlist once at startup to select an appropriate stream variant:
```m3u8
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-STREAM-INF:BANDWIDTH=3440800,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
stream_720p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1755600,RESOLUTION=854x480,CODECS="avc1.4d401f,mp4a.40.2"
stream_480p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=950400,RESOLUTION=640x360,CODECS="avc1.4d401e,mp4a.40.2"
stream_360p/index.m3u8
```
- `BANDWIDTH`: Total required bandwidth in bits per second, including audio and video overhead ($(\text{video} + \text{audio}) \times 1.1$).
- `CODECS`: Declares required decoder capabilities (e.g., H.264 video and AAC audio), allowing players to skip unsupported codecs.

---

### 5.3 The Media Playlist (`index.m3u8`): The Sliding Window Polling Rules
RFC 8216 Section 6.3.4 defines specific rules for polling media playlists:
```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:2
#EXTINF:4.000000,
seg_000002.ts
...
#EXTINF:4.000000,
seg_000011.ts
```
1. **If the playlist updated** (contains new segment URLs): The player must wait **at least one target duration** (4 seconds) before polling again.
2. **If the playlist was unchanged**: The player must wait **half the target duration** (2 seconds) before retrying.
3. **If the media sequence repeats with different content**: The player must halt playback to prevent stream corruption.

---

### 5.4 VOD vs. Live Playlists (`#EXT-X-ENDLIST`)
- **Video on Demand (VOD)**: The playlist terminates with the **`#EXT-X-ENDLIST`** tag. All segments are known in advance; the player reads the manifest once and stops polling.
- **Live Streams**: The playlist does not contain an `#EXT-X-ENDLIST` tag. The file represents a rolling sliding window, adding new segments to the bottom and dropping old segments from the top.

---

### 5.5 Wire Packaging: MPEG-2 Transport Streams (188-Byte Packets) vs. fMP4
Historically, HLS segments were packaged as MPEG-2 Transport Streams (`.ts`):

```
┌──────────────────────────────────────┬─────────────────────────────────────────────────┐
│        4-Byte Packet Header          │              184-Byte Payload Slice             │
│ [ 0x47 Sync Byte | 13-bit PID | Ctr ]│ [ Slice of Video (H.264), Audio, or PAT/PMT ]   │
└──────────────────────────────────────┴─────────────────────────────────────────────────┘
 Total Packet Length = Exactly 188 Bytes (aligned with legacy ATM network cells)
```

- **PID (Packet Identifier)**: A 13-bit value that multiplexes elementary audio, video, and subtitle streams into a shared byte sequence.
- **Modern Standards (fMP4 / CMAF)**: Newer deployments use fragmented MP4 (fMP4) and Common Media Application Format (CMAF), packaging data into ISO boxes (`moof`, `mdat`) using Type-Length-Value (TLV) structures.

---

### 5.6 How Long Should a Video Segment Be? The Core Trade-Off

| Segment Length | Peak Segment Requests (50M viewers) | Minimum Latency Floor ($3 \times$) | Operational Characteristics |
| :---: | :---: | :---: | :--- |
| **2 seconds** | 25 Million req/sec | 6 seconds | Low latency, fast adaptation; high request overhead. |
| **4 seconds** | 12.5 Million req/sec | 12 seconds | Standard production balance of request load and latency. |
| **6 seconds** | 8.3 Million req/sec | 18 seconds | Apple's default recommendation; better compression. |
| **10 seconds**| 5.0 Million req/sec | 30 seconds | High latency; unsuitable for live sports. |

- **Shorter Segments**: Reduce latency and adapt faster to network drops, but require more frequent keyframes (which hurts video compression) and increase CDN request rates.
- **Longer Segments**: Improve video compression and reduce request volumes, but increase latency and slow down bitrate switching.

---

### 5.7 Why Playlist Traffic is Harder Than Video Segment Traffic
- **Video Segments (`.ts`)**: Large files (~823 KB) that never change. They can be cached indefinitely across edge PoPs, resulting in high cache hit ratios.
- **Media Playlists (`.m3u8`)**: Small text files (~228 bytes) updated every 4 seconds. They require short cache TTLs ($\le 2\text{ seconds}$), meaning edge caches expire rapidly and generate steady traffic back to origins.

---

## 6. Playback Buffers, Latency Budgets & Spoilers

### 6.1 The Player Playhead Buffer: Trade-offs
A video player maintains an internal memory buffer of downloaded segments ahead of its current playhead position:
- **Large Buffer (e.g., 30–60 seconds)**: Handles brief network interruptions (such as mobile handovers), but introduces latency relative to live action and increases memory consumption.
- **Small Buffer (e.g., 1–2 segments)**: Delivers near-live latency, but risks rebuffering stalls if a segment download experiences jitter.

---

### 6.2 The HLS Latency Budget: Why Broadcasts Lag by 20 Seconds
Standard HLS broadcasts typically lag live action by 20 to 30 seconds:

$$\mathbf{\text{Total Latency}} = \text{Encoding (2s)} + \text{Segment Ingestion (4s)} + \text{CDN Distribution (2s)} + \text{Player Hold-Back (12s)} \approx \mathbf{20 \text{ seconds}}$$

- **Player Hold-Back**: RFC 8216 mandates that players maintain a buffer of at least **3 full segment durations** behind the live edge to ensure smooth playback across variable network conditions.

---

### 6.3 Low-Latency HLS (LL-HLS): Shrinking Latency to ~3 Seconds
Developed by Apple in 2019, LL-HLS reduces broadcast latency to 2–4 seconds using four coordinated mechanisms:
1. **Partial Segments (Parts)**: Segments are divided into ~1-second chunks (`part-0.mp4`, `part-1.mp4`), making video available before full segments finish encoding.
2. **Blocking Playlist Reload**: The player queries the upcoming playlist using query parameters (`?_HLS_msn=1843&_HLS_part=2`). The server **holds the request open using long-polling** until the chunk is available, returning it immediately.
3. **Preload Hints**: The server includes an `#EXT-X-PRELOAD-HINT` tag in responses, advertising the URL of the next chunk before it is written so the player can request it proactively.
4. **Delta Playlists (`EXT-X-SKIP`)**: Instead of transmitting the full manifest on every refresh, the server sends only diffs, dropping older segments to reduce bandwidth.

---

### 6.4 The Spoiler Problem & Wall-Clock Synchronization

```
THE SPOILER TIMELINE:
Time 0s  : Wicket falls at the stadium!
Time 10s : Fast scorecard app receives JSON update ──> App alerts: "WICKET!"
Time 20s : Heavy video stream buffers and renders ──> Video finally shows the play!

A 10-second gap causes score updates to spoil live video playback.
```

#### Synchronizing Video and Data:
To prevent spoilers, the scorecard and video pipeline share synchronized UTC timestamps:
- The scorecard records the event time:
  ```json
  {"version": 1842, "ball": "15.4", "timestamp": "2026-09-27T03:16:44.100Z"}
  ```
- The HLS stream injects matching wall-clock metadata:
  ```m3u8
  #EXT-X-PROGRAM-DATE-TIME:2026-09-27T03:16:44.365+0000
  seg_001842.ts
  ```
- The mobile app buffers the scorecard update in memory, displaying it only when the video playhead's `#EXT-X-PROGRAM-DATE-TIME` matches or passes the scorecard timestamp.

---

### 6.5 Adaptive Bitrate (ABR): Throughput-Based vs. Buffer-Based Algorithms
The player adjusts video quality dynamically based on network conditions:
1. **Throughput-Based ABR**:
   - Measures download speeds of recent segments:
     $$\text{Throughput} = \frac{\text{Segment Size (bits)}}{\text{Download Duration (seconds)}}$$
   - Selects the highest rendition profile that fits within measured bandwidth.
   - *Limitation*: Bandwidth estimates can fluctuate on cellular connections, causing erratic quality switches.
2. **Buffer-Based ABR (BBA)**:
   - Evaluates player buffer fullness rather than estimating bandwidth.
   - If the buffer is nearly empty, step down to lower bitrates to prevent stalls; if the buffer is full, step up to higher resolutions.
   - A/B testing at Netflix demonstrated that buffer-based algorithms achieve **10–20% fewer playback stalls** while maintaining comparable average bitrates.

---

## 7. Multi-CDN Streaming & Live Caching Rules

### 7.1 Multi-CDN Topologies & Content Steering
High-concurrency streaming platforms distribute traffic across multiple CDN vendors:
- **DNS Steering**: Distributes CDN endpoints based on geographic region or ISP performance.
- **Master Playlist Dual-Listing**: The master playlist lists alternative CDN endpoints for each stream variant, allowing players to fail over between CDNs if errors occur.
- **Content Steering (HLS RFC 8216bis)**: The player periodically polls a steering manifest JSON file that dynamically updates CDN priority order based on real-time network telemetry.

---

### 7.2 The Stale Playlist and Deleted Segment Trap
In live streaming configurations, media encoders clean up old segments from disk using sliding windows:
1. An encoder processes live video and deletes older segments (`delete_segments`).
2. RFC 8216 requires segments to remain accessible for their duration plus the playlist window ($\text{Duration} + \text{Window} = 4\text{s} + 40\text{s} = 44\text{ seconds}$).
3. If an origin disruption causes a CDN to serve a stale playlist, the manifest may still reference an old segment that has already been deleted from storage.
4. When a player attempts to fetch that deleted segment, it receives a **`404 Not Found`**, which can get negatively cached and stall video playback.

#### Mitigations:
- Set FFmpeg's `-hls_delete_threshold 12` to retain segments on disk well beyond the active playlist window.
- Restrict media playlist cache TTLs to **$\le \frac{1}{2}$ the target duration**, and serve stale manifests for at most one target duration.

---

### 7.3 Live Stream Caching Matrix for NGINX & CDNs

```
┌───────────────────┬───────────────────────┬──────────────────────────┬────────────────────────────┐
│ Content Asset     │ Cache TTL             │ Serve Stale Policy       │ Negative Cache (404) Rule  │
├───────────────────┼───────────────────────┼──────────────────────────┼────────────────────────────┤
│ **`master.m3u8`** │ 5 to 10 minutes       │ Permitted freely         │ Cache 1s                   │
├───────────────────┼───────────────────────┼──────────────────────────┼────────────────────────────┤
│ **`media.m3u8`**  │ $\le 2$ seconds       │ At most 1 target duration│ **Do not cache** (or 1s)   │
├───────────────────┼───────────────────────┼──────────────────────────┼────────────────────────────┤
│ **`segment.ts`**  │ Hours to days         │ Permitted freely         │ Cache 1s at live edge only │
├───────────────────┼───────────────────────┼──────────────────────────┼────────────────────────────┤
│ **`score.json`**  │ 2 seconds             │ Permitted while updating │ Cache 1s (5xx: 2s)         │
└───────────────────┴───────────────────────┴──────────────────────────┴────────────────────────────┘
```

---

## 8. Generating the Stream: Real-World FFmpeg Packaging

### 8.1 A Simple Single-Bitrate HLS Command
```bash
ffmpeg -re -i input.mp4 \
  -codec: copy \
  -hls_time 4 \
  -hls_list_size 5 \
  -hls_playlist_type event \
  -f hls index.m3u8
```
- `-re`: Reads input at native frame rate, simulating a live camera feed.
- `-codec: copy`: Bypasses re-encoding, copying elementary streams directly.
- `-hls_time 4`: Sets target segment duration to 4 seconds.
- *Caveat*: If the source video uses an 8.33-second keyframe interval (default x264 GOP of 250 frames at 30 fps), FFmpeg cannot cut segments at 4 seconds and will generate 8.33-second segments instead!

---

### 8.2 The Full Adaptive Bitrate (ABR) Ladder Command
```bash
ffmpeg -re -stream_loop -1 -i input.mp4 \
  -filter_complex "[0:v]split=3[v720][v480][v360]; \
                   [v720]scale=w=1280:h=720[v720out]; \
                   [v480]scale=w=854:h=480[v480out]; \
                   [v360]scale=w=640:h=360[v360out]; \
                   [0:a]asplit=3[a0][a1][a2]" \
  -map "[v720out]" -map "[a0]" \
  -map "[v480out]" -map "[a1]" \
  -map "[v360out]" -map "[a2]" \
  -c:v libx264 -preset veryfast -profile:v main \
  -sc_threshold 0 -g 120 -keyint_min 120 \
  -c:a aac -ac 2 -ar 48000 \
  -b:v:0 3000k -maxrate:v:0 3210k -bufsize:v:0 4500k -b:a:0 128k \
  -b:v:1 1500k -maxrate:v:1 1605k -bufsize:v:1 2250k -b:a:1 96k \
  -b:v:2 800k  -maxrate:v:2 856k  -bufsize:v:2 1200k -b:a:2 64k \
  -f hls -hls_time 4 -hls_list_size 10 \
  -hls_flags independent_segments+delete_segments+omit_endlist+program_date_time \
  -hls_segment_filename "out/stream_%v/seg_%06d.ts" \
  -master_pl_name master.m3u8 \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" \
  "out/stream_%v/index.m3u8"
```

---

### 8.3 The GOP Keyframe Alignment Rule (`-g 120`)
- **Keyframe Constraint**: Video encoders can only start a new segment on an **I-frame (Keyframe)**. If an I-frame does not exist at the segment boundary, the packager must delay the cut until the next keyframe appears.
- **The Solution**: Set a fixed Group of Pictures (GOP) size using `-g 120 -keyint_min 120 -sc_threshold 0`.
  - At 30 frames per second, 120 frames produces an I-frame **exactly every 4.0 seconds**:
    $$\frac{120\text{ frames}}{30\text{ fps}} = \mathbf{4.000\text{ seconds}}$$
  - Setting `-sc_threshold 0` disables dynamic scene change detection, ensuring all three rendition variants align keyframes on the exact same frame boundaries. This allows players to switch bitrates seamlessly without decoding artifacts.

---

## 9. Synthesis: The Complete Curriculum in a Single Match

Every concept covered across the 8-week curriculum connects within a live streaming event:

```
┌────────┬─────────────────────────────┬────────────────────────────────────────────────────────────────────────┐
│ Week   │ Curriculum Topic            │ Role in Live Event Architecture                                        │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W1** │ **Network Programming 101** │ Socket lifecycle, ephemeral port allocation, and Base64 WebSocket math.│
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W2** │ **Eagle Eye View**          │ Wire framing: SSE newlines, 188-byte MPEG-TS packets, control vs. data.│
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W3** │ **Server Scale**            │ Managing 50M connections, Little's Law, and kernel memory per socket.  │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W4** │ **NGINX Deep Dive**         │ Edge proxy configuration: `proxy_cache_lock`, `use_stale`, zero-copy. │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W5** │ **Evolution of HTTP**       │ ETags, 304 revalidation, and how HTTP/2 push informed LL-HLS design.  │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W6** │ **(Ab)using CDNs**          │ Consistent hashing, origin shielding, and per-request vs. byte costs.  │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W7** │ **Building for Failures**   │ Full Jitter, circuit breakers, live-edge 404 retries, and fallbacks.   │
├────────┼─────────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ **W8** │ **Streaming Video at Scale**│ HLS playlists, segment caching, ABR algorithms, and clock sync.        │
└────────┴─────────────────────────────┴────────────────────────────────────────────────────────────────────────┘
```

---

## 10. The Four Core Invariants of Video Systems

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE FOUR CORE INVARIANTS                               │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. PULL SCALES THROUGH     : If the payload is identical for everyone, distribute it   │
│    CACHES; PUSH SCALES       via CDN caches. If data is individualized, you must buy   │
│    THROUGH MACHINES          servers to manage stateful connections.                   │
│                                                                                        │
│ 2. NUMBER THE PIECES, THEN : A small, frequently polled manifest pointing to numbered, │
│    CACHE THEM FOREVER        immutable data segments scales reliably.                  │
│                                                                                        │
│ 3. LATENCY IS A SAFETY     : Buffer depth, segment duration, and poll intervals all    │
│    BUDGET                    sum together. Lowering latency reduces tolerance for      │
│                              network jitter.                                           │
│                                                                                        │
│ 4. KNOW WHICH BILL         : Payloads under 11 KB are billed primarily by request count;│
│    YOU ARE ON                payloads over 11 KB are billed primarily by bandwidth.    │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
Why does Little's Law ($L = \lambda \times W$) dictate that a persistent WebSocket or MQTT push system requires significantly more server concurrency ($L$) than a pull-based CDN polling system serving the same audience?
- A) WebSockets use UDP, which consumes more bandwidth than TCP.
- B) In pull polling, each request lasts ~30 ms at the edge cache, whereas persistent connections hold sockets open for the entire 4-hour event duration ($W = 14,400\text{ s}$), increasing concurrent open connections by orders of magnitude.
- C) Modern web browsers limit WebSocket connections to 100 per server.
- D) CDN caches cannot process HTTP GET requests.

**Answer: B**  
**Explanation:** Little's Law states $L = \lambda \times W$. In pull architectures, connection duration ($W$) is brief (~30 ms at the cache), keeping concurrent in-flight requests low. In persistent push systems, $W$ spans hours, forcing servers to hold millions of sockets open simultaneously.

---

### Question 2
Why does serving a frequently polled live scorecard directly from an Amazon S3 bucket fail at large scale, even if storage and bandwidth budgets are sufficient?
- A) Amazon S3 enforces a hard limit of 100 total connections per bucket.
- B) Amazon S3 supports a baseline of 5,500 GET requests per second per prefix; a large event generating over 1.6 million requests per second exceeds this limit, triggering widespread `503 Slow Down` errors.
- C) Amazon S3 requires client authentication using IAM credentials.
- D) S3 buckets do not support JSON data formats.

**Answer: B**  
**Explanation:** Amazon S3 partitions scale up to roughly 5,500 GET requests per second per prefix. Millions of users polling a single key (`/score.json`) exceed this threshold by hundreds of times, resulting in rate-limit errors.

---

### Question 3
At what payload size does CDN delivery cost typically shift from being dominated by per-request fees to being dominated by per-byte bandwidth fees?
- A) 188 Bytes
- B) Approximately 11 Kilobytes
- C) 10 Megabytes
- D) 1 Gigabyte

**Answer: B**  
**Explanation:** At standard CDN list prices (e.g., \$1.20 per million requests vs. ~\$0.10 per GB), the cost of the request fee equals the bandwidth fee at approximately 11 KB. Smaller files are billed primarily by request count; larger files by data volume.

---

### Question 4
In a multi-tier caching architecture (Origin $\rightarrow$ Shield $\rightarrow$ Edge $\rightarrow$ Client), what is the primary risk of configuring 30-second cache TTLs at every layer?
- A) TCP sockets will fail with `ECONNRESET`.
- B) Staleness compounds across layers, meaning a client could receive data that is up to 90 seconds out of date.
- C) Edge servers will run out of file descriptors.
- D) Intermediate proxies will strip ETag headers.

**Answer: B**  
**Explanation:** Cache durations accumulate along the delivery path: $\text{Shield TTL (30s)} + \text{Edge TTL (30s)} + \text{Client Poll (30s)} = 90\text{ seconds}$. Layers must use shorter TTLs ($\le \frac{1}{2}$ the freshness target) to keep updates timely.

---

### Question 5
Why should the `proxy_cache_lock` directive be enabled on reverse proxies serving live media playlists?
- A) It encrypts media playlists on disk using AES-256.
- B) It prevents cache stampedes: when a playlist expires, only one request is sent to the origin, while other concurrent requests wait to be served from the updated cache.
- C) It restricts playlist access to authenticated users.
- D) It converts live streams into VOD assets.

**Answer: B**  
**Explanation:** `proxy_cache_lock` collapses concurrent requests for an expired cache key. The proxy fetches one copy from the origin and serves waiting clients from that response, shielding backends from traffic spikes.

---

### Question 6
Why do video streaming architectures separate HLS streams into Master Playlists, Media Playlists, and Media Segments?
- A) To prevent search engines from indexing video content.
- B) To separate control metadata (rendition options and sliding segment windows) from immutable media data, allowing large media segments to be cached permanently across CDNs.
- C) Because web servers cannot deliver files larger than 1 MB.
- D) To bypass browser CORS security checks.

**Answer: B**  
**Explanation:** Master and media playlists manage stream selection and rolling segment availability, while media segments represent static, immutable audio/video chunks that can be cached indefinitely at the edge.

---

### Question 7
What is the structural packet size of a standard MPEG-2 Transport Stream (`.ts`) video segment?
- A) 140 bytes
- B) Exactly 188 bytes (including a 4-byte header and a 184-byte payload slice)
- C) 1,500 bytes (matching Ethernet MTU)
- D) 4,096 bytes

**Answer: B**  
**Explanation:** MPEG-2 Transport Streams consist of fixed 188-byte packets containing a 4-byte header (`0x47` sync byte, PID) and a 184-byte payload, a design historically aligned with ATM cell routing.

---

### Question 8
What is the primary operational trade-off when selecting video segment durations in live HLS streaming?
- A) Shorter segments increase video resolution but require GPU transcoding.
- B) Shorter segments reduce broadcast latency and speed up bitrate switching, but increase CDN request rates and require more frequent keyframes (which reduces compression efficiency).
- C) Longer segments eliminate the need for master playlists.
- D) Shorter segments bypass CORS preflight checks.

**Answer: B**  
**Explanation:** Shorter segments reduce latency and allow players to adapt quickly to bandwidth changes, but generate higher request volumes and require more I-frames, which reduces overall video compression efficiency.

---

### Question 9
Why does standard HLS streaming introduce an inherent latency delay of roughly 15 to 20 seconds behind live events?
- A) Video packets must traverse geostationary satellite relays.
- B) The latency budget accumulates encoding delays, segment file generation, CDN distribution, and the player's mandatory 3-segment hold-back buffer.
- C) TLS 1.3 requires 15 seconds to negotiate session keys.
- D) Web browsers limit media downloads to 1 Mbps.

**Answer: B**  
**Explanation:** Standard live HLS latency reflects the sum of pipeline stages: encoding delays (~2s), segment accumulation (~4s), CDN propagation (~2s), and the specification's requirement that players buffer 3 full segments (~12s) before playback begins.

---

### Question 10
How does Low-Latency HLS (LL-HLS) reduce broadcast lag to 2–4 seconds without changing baseline segment durations?
- A) By switching transport to unencrypted UDP datagrams.
- B) By splitting segments into ~1-second partial chunks, using blocking playlist reloads (long polling), and advertising upcoming parts via preload hints.
- C) By reducing video frame rates to 5 fps.
- D) By disabling CDN edge caching.

**Answer: B**  
**Explanation:** LL-HLS breaks segments into smaller partial chunks, uses long-polling to notify players the moment new parts are ready, and provides preload hints so players can request chunks as soon as they become available.

---

### Question 11
How do streaming applications resolve the "Spoiler Problem," where fast scorecard notifications reveal plays before the video stream renders them?
- A) By delaying scorecard updates at the server for 60 seconds.
- B) By embedding synchronized UTC timestamps in scorecard updates and matching them to the video stream's `#EXT-X-PROGRAM-DATE-TIME` metadata, revealing updates only as playback passes that timestamp.
- C) By disabling push notifications on mobile devices.
- D) By serving video over WebSockets.

**Answer: B**  
**Explanation:** Stamping data with synchronized timestamps allows client applications to hold score updates in memory, displaying them only when the video player's playhead reaches the matching playback time.

---

### Question 12
How does Buffer-Based Rate Adaptation (BBA) improve video streaming performance compared to traditional throughput-based ABR?
- A) It eliminates the need for video manifest files.
- B) It selects video quality renditions based on playhead buffer fullness rather than fluctuating bandwidth estimates, reducing playback stalls by 10–20%.
- C) It prioritizes audio playback over video decoding.
- D) It forces all viewers onto the lowest available bitrate.

**Answer: B**  
**Explanation:** Throughput-based algorithms can react erratically to temporary cellular bandwidth dips. Buffer-based adaptation uses buffer fullness to smooth out decisions, maintaining higher average quality with fewer rebuffering stalls.

---

### Question 13
What causes the "Deleted Segment 404 Trap" in live streaming architectures?
- A) An edge router runs out of ARP table entries.
- B) An origin packager deletes older segments from storage, but a CDN serves a stale media playlist that still references those deleted files, causing players to request missing URLs.
- C) The video player attempts to parse MPEG-TS packets as JSON.
- D) The client's operating system clock drifts backward.

**Answer: B**  
**Explanation:** If an origin removes expired segments from storage while a CDN serves an outdated playlist that still lists them, players requesting those segments receive 404 errors, which can stall playback if cached.

---

### Question 14
In an FFmpeg adaptive bitrate encoding command, why is the parameter `-g 120` critical when generating 4-second segments at 30 fps?
- A) It limits CPU utilization to 120% across worker threads.
- B) It enforces an exact keyframe (I-frame) interval of 120 frames ($\frac{120}{30} = 4\text{ seconds}$), ensuring segments can be cut cleanly at 4-second boundaries and aligning keyframes across all bitrate renditions.
- C) It allocates 120 megabytes of memory to the video filter.
- D) It generates 120 independent audio channels.

**Answer: B**  
**Explanation:** Video segments must begin on an I-frame. Aligning keyframes to 120 frames at 30 fps guarantees an I-frame every 4 seconds across all renditions, enabling clean segment cuts and smooth bitrate switching.

---

### Question 15
What is the primary architectural lesson of the statement: "Pull scales through caches; push scales through machines"?
- A) Persistent push connections should be used for all web traffic.
- B) Identical public content can be scaled to millions via decentralized edge caches; individualized or stateful connections cannot be cached and require scaling physical backend infrastructure.
- C) Reverse proxies cannot handle video streams.
- D) Caching introduces security vulnerabilities into live streams.

**Answer: B**  
**Explanation:** Pull architectures scale efficiently through shared edge caching because the same data serves all clients. Stateful push connections cannot be shared through standard caches, requiring server hardware to scale directly with the user base.

---

## 12. Session 8 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **Pull vs. Push** | Pull scales through CDN caches (stateless); Push scales through server instances (stateful sockets). |
| **Little's Law Twice** | Polling: $W = 30\text{ ms} \implies 50,000$ in-flight requests. Push: $W = 4\text{ hours} \implies 50,000,000$ open sockets. |
| **Held Socket Overhead**| Consumes ~137 KB of RAM per connection; 20M idle phones generate 333,000 keep-alive pings/second. |
| **S3 Prefix Ceiling** | Capped at 5,500 GET/s per prefix. 50M polling clients require 1.67M GET/s (303x over capacity $\implies$ 503 errors). |
| **11 KB Break-Even Line**| Files $< 11\text{ KB}$ are billed primarily by request fees; files $> 11\text{ KB}$ are billed primarily by bandwidth fees. |
| **Stacked TTL Delay** | Latency accumulates across layers: $\text{Shield TTL} + \text{Edge TTL} + \text{Client Poll} = \text{Total Staleness}$. |
| **Expiry Stampedes** | Prevented via `proxy_cache_lock` (request collapsing) and `proxy_cache_use_stale updating;`. |
| **Scorecard vs. Playlist**| Both use small, frequently polled index files pointing to numbered, immutable data chunks. |
| **HLS Structure** | Master Playlist (rendition menu) $\rightarrow$ Media Playlist (sliding window) $\rightarrow$ Segments (audio/video data). |
| **MPEG-TS Packaging** | Fixed 188-byte packets with 4-byte headers, sync byte `0x47`, and 13-bit PIDs. |
| **Segment Duration** | 4 seconds is standard. Shorter segments reduce latency but increase request overhead and reduce compression. |
| **HLS Latency Budget** | Standard live HLS lags live action by ~20 seconds; LL-HLS reduces lag to 2–4 seconds using partial chunks. |
| **The Spoiler Problem** | Synchronizes scorecard updates with video playhead timing via `#EXT-X-PROGRAM-DATE-TIME` timestamps. |
| **Buffer-Based ABR** | Adjusts video quality based on playhead buffer fullness rather than volatile bandwidth estimates. |
| **GOP Keyframe Rule** | Enforces `-g 120` at 30 fps to place I-frames exactly every 4 seconds, aligning segment boundaries across renditions. |
