# Chapter 07: Building for Failures — Resilient Distributed Systems

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 7 / Week 7 of 8  
> **Core Theme**: *"One discipline: fail on purpose. Timeouts, retries, backoff, circuit breakers — and what a video player does when the network lies to it."*  
> **Key Protocols & Concepts**: The 6-stage request failure taxonomy, HTTP status classification for resilience, Strict definition of a Retry vs. Handling vs. Failover, Idempotency semantics (RFC 9110 §9.2) and Idempotency Keys, The Naive Retry Loop trap, Exponential Backoff with AWS Full Jitter, Server-guided pacing via `Retry-After`, The Circuit Breaker Pattern (Closed, Open, Half-Open), Case study: Video streaming player failure modes (96 reasons to retry), The 404 live-edge exception, Fallbacks over retries (DRM tiers, codec substitution, manifest switching, ad dropping, iOS `ResetMedia`), The Fastly 49-minute outage & `stale-if-error` (RFC 5861), The Second Outage (Thundering herd on recovery), Graceful degradation, Feature flags, and 15 exam-grade MCQs.

---

## Table of Contents
1. [The Philosophy of Network Resilience: Fail on Purpose](#1-the-philosophy-of-network-resilience-fail-on-purpose)
2. [The Request Failure Taxonomy: Where Systems Actually Break](#2-the-request-failure-taxonomy-where-systems-actually-break)
   - [2.1 The Six Stages of a Request Lifecycle](#21-the-six-stages-of-a-request-lifecycle)
   - [2.2 DNS Timeout vs. Read Timeout: The Asymmetry of Risk](#22-dns-timeout-vs-read-timeout-the-asymmetry-of-risk)
3. [The Retry Decision Framework](#3-the-retry-decision-framework)
   - [3.1 What is a Retry? The Strict Architectural Definition](#31-what-is-a-retry-the-strict-architectural-definition)
   - [3.2 The Four Classification Gates](#32-the-four-classification-gates)
   - [3.3 HTTP Status Codes: "Will Anything Be Different Next Time?"](#33-http-status-codes-will-anything-be-different-next-time)
   - [3.4 When NOT to Retry (Structural Failures)](#34-when-not-to-retry-structural-failures)
   - [3.5 When to Retry (Transient Failures)](#35-when-to-retry-transient-failures)
4. [Idempotency: The Precondition for Safe Retries](#4-idempotency-the-precondition-for-safe-retries)
   - [4.1 Idempotent HTTP Methods (GET, PUT, DELETE)](#41-idempotent-http-methods-get-put-delete)
   - [4.2 The Danger of POST Retries & Double Execution](#42-the-danger-of-post-retries--double-execution)
   - [4.3 Idempotency Keys: Deduplication Architecture](#43-idempotency-keys-deduplication-architecture)
5. [The Mechanics of Retrying](#5-the-mechanics-of-retrying)
   - [5.1 The Naive Retry Loop & The Self-Inflicted Outage](#51-the-naive-retry-loop--the-self-inflicted-outage)
   - [5.2 Exponential Backoff: Decaying the Load](#52-exponential-backoff-decaying-the-load)
   - [5.3 Full Jitter: Breaking the Synchronization Lockstep](#53-full-jitter-breaking-the-synchronization-lockstep)
   - [5.4 Server-Directed Pacing: The `Retry-After` Header](#54-server-directed-pacing-the-retry-after-header)
6. [The Circuit Breaker Pattern: Fleet-Level Protection](#6-the-circuit-breaker-pattern-fleet-level-protection)
   - [6.1 The Three States: Closed, Open, Half-Open](#61-the-three-states-closed-open-half-open)
   - [6.2 Protecting the Caller and Protecting the Callee](#62-protecting-the-caller-and-protecting-the-callee)
7. [Case Study: The 96 Failure Modes of a Video Streaming Player](#7-case-study-the-96-failure-modes-of-a-video-streaming-player)
   - [7.1 Why Player Resilience is Harder Than API Resilience](#71-why-player-resilience-is-harder-than-api-resilience)
   - [7.2 Network Issues Under a Tighter Clock](#72-network-issues-under-a-tighter-clock)
   - [7.3 The Live Segment 404 Exception](#73-the-live-segment-404-exception)
   - [7.4 When the Answer is Fallback, Not Retry](#74-when-the-answer-is-fallback-not-retry)
   - [7.5 Business-Rule Retries vs. Transport Signals](#75-business-rule-retries-vs-transport-signals)
8. [Systems That Survive: Architecture Beyond the Single Request](#8-systems-that-survive-architecture-beyond-the-single-request)
   - [8.1 The Fastly 49-Minute Outage & `stale-if-error` (RFC 5861)](#81-the-fastly-49-minute-outage--stale-if-error-rfc-5861)
   - [8.2 The Three Levers: Failover, Feature Flags, Graceful Degradation](#82-the-three-levers-failover-feature-flags-graceful-degradation)
   - [8.3 The Second Outage: The Thundering Herd on Recovery](#83-the-second-outage-the-thundering-herd-on-recovery)
9. [The Unified Failure Resolution Flowchart](#9-the-unified-failure-resolution-flowchart)
10. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#10-comprehensive-review--practice-mcqs-with-detailed-explanations)
11. [Session 7 Summary Cheat Sheet](#11-session-7-summary-cheat-sheet)

---

## 1. The Philosophy of Network Resilience: Fail on Purpose

In distributed computing, failures are not anomalies; they are guaranteed operational events. As systems scale across availability zones, public clouds, and global CDNs:
- Fiber cables get severed.
- Routers drop packets under transient buffer saturation.
- Microservices stall on garbage collection pauses or database locks.
- DNS resolvers experience packet loss.

A naive client treats every error as an unexpected exception. A resilient system is designed with a deliberate failure policy: **it classifies the nature of every failure before deciding whether to retry, wait, fail over, degrade quality, or abort.**

---

## 2. The Request Failure Taxonomy: Where Systems Actually Break

A network call is not an atomic operation. An HTTP request traverses six distinct operational stages, and a failure at one stage has completely different implications than a failure at another:

```
THE SIX STAGES OF A REQUEST LIFECYCLE:
[ 1. DNS Lookup ] ──> [ 2. TCP Handshake ] ──> [ 3. TLS Handshake ] ──> [ 4. Request Write ] ──> [ 5. Server Processing ] ──> [ 6. Response Read ]
   (name -> IP)          (SYN/SYN-ACK/ACK)        (Key Exchange)          (Bytes sent to wire)       (Upstream business work)   (Bytes arrive in chunks)
```

### 2.1 The Six Stages of a Request Lifecycle
1. **Stage 1: DNS Lookup (`name -> IP`)**:
   - The client queries recursive resolvers via UDP port 53 or DNS-over-HTTPS.
   - *Failure Mode*: Network drop, timeout, or DNS server failure.
   - *State Impact*: **Zero server state**. The request never left the client's host.
2. **Stage 2: TCP Handshake (`SYN / SYN-ACK / ACK`)**:
   - The OS kernel attempts to negotiate connection establishment with the target port.
   - *Failure Mode*: `ECONNREFUSED` (no process listening) or `ETIMEDOUT` (SYN dropped).
   - *State Impact*: **Zero server state**. No application payload was transmitted.
3. **Stage 3: TLS Handshake (Cipher & Key Exchange)**:
   - Client and server exchange cryptographic certificates and negotiate symmetric session keys.
   - *Failure Mode*: Certificate expired, untrusted CA, SNI mismatch, or TLS version alert.
   - *State Impact*: **Zero application state**. Connection aborted before HTTP bytes moved.
4. **Stage 4: Request Write (Bytes Leaving Local Socket)**:
   - Application copies HTTP request line, headers, and body into the kernel socket send buffer.
   - *Failure Mode*: Connection reset by peer (`ECONNRESET`) or broken pipe (`EPIPE` / `SIGPIPE`).
   - *State Impact*: Ambiguous. The server may have received partial headers before disconnecting.
5. **Stage 5: Server Processing (Upstream Execution)**:
   - The server application parses the request, executes database queries, and prepares a response.
   - *Failure Mode*: Worker timeout, database lock, unhandled exception, or upstream gateway timeout (`504 Gateway Timeout`).
   - *State Impact*: **High risk**. The server may have already committed database mutations!
6. **Stage 6: Response Read (Bytes Arriving Across Wire)**:
   - Client reads the response body from its kernel socket buffer.
   - *Failure Mode*: Connection dropped mid-stream after partial body transfer.
   - *State Impact*: **Server finished work successfully**, but the client failed to receive the confirmation.

---

### 2.2 DNS Timeout vs. Read Timeout: The Asymmetry of Risk
In monitoring dashboards, both errors are often lumped together under generic labels like `"timeout"`:
- **DNS Timeout**: The request never reached a server. Retrying is completely safe because no remote mutation could have occurred.
- **Read Timeout**: The request was transmitted and the server began processing. The remote operation may have completed fully before the connection dropped. Retrying blindly risks double-executing transactions.

---

## 3. The Retry Decision Framework

### 3.1 What is a Retry? The Strict Architectural Definition
In distributed systems, engineers often confuse retries with general error handling:

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│           TRUE RETRY                 │         HANDLING / FAILOVER          │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ Trying the EXACT SAME step again,    │ Executing a DIFFERENT action in      │
│ with NOTHING changed.                │ response to the failure.             │
│ - Same HTTP method.                  │ - Refreshing an expired auth token.  │
│ - Same request body.                 │ - Falling back to a backup CDN.      │
│ - Same headers.                      │ - Routing to a replica database.     │
│ - Betting that a TRANSIENT failure   │ - Changing request parameters.       │
│   clears after waiting.              │                                      │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

> **The Architectural Test**:  
> *"If you changed nothing about the request and only waited, would you call the second attempt identical to the first? If the answer is NO (e.g., new token, different endpoint, altered payload), you are NOT retrying; you are handling."*

---

### 3.2 The Four Classification Gates
Before touching a retry loop or incrementing an attempt counter, an application must evaluate four classification criteria:

```
                    ┌──────────────────────────────────────────────┐
                    │          THE FOUR RETRY GATES                │
                    └──────────────────────┬───────────────────────┘
                                           │
 1. WHERE DID IT FAIL?  ───────────────────┼──> Before connection (safe) vs. Mid-response (risky)
 2. WHO ADMITS FAULT?   ───────────────────┼──> 5xx ("I failed" -> retry) vs. 4xx ("You failed" -> abort)
 3. IS IT IDEMPOTENT?   ───────────────────┼──> GET/PUT (safe) vs. POST (requires idempotency key)
 4. TRANSIENT OR STRUCTURAL? ──────────────┴──> Glitch (will clear) vs. Missing resource (won't clear)
```

---

### 3.3 HTTP Status Codes: "Will Anything Be Different Next Time?"

| HTTP Status Class | Meaning | Underlying Event | Retry Strategy |
| :---: | :--- | :--- | :--- |
| **`1xx`** | Informational | Interim response; protocol transition. | Not applicable (keep listening). |
| **`2xx`** | Success | Server completed requested operation. | No retry needed. |
| **`3xx`** | Redirection | Resource moved; follow new URI location. | Follow redirect (not a retry). |
| **`4xx`** | Client Error | Server rejected request syntax or semantics. | **DO NOT RETRY** (same request fails again). |
| **`5xx`** | Server Error | Server admits internal processing failure. | **RETRY CANDIDATE** (transient backend issue). |

---

### 3.4 When NOT to Retry (Structural Failures)
Retrying is counterproductive when a failure is structural rather than transient:
1. **`404 Not Found`**: If a resource does not exist, retrying 200 ms later will yield another 404 (except for live video segments).
2. **Pre-Handshake Failure on Dead Hosts**: If DNS returns `NXDOMAIN` (non-existent domain), retrying burns CPU and socket resources without benefit.
3. **Geo-Blocking (`403 Forbidden` / `451 Unavailable for Legal Reasons`)**: Request is refused due to network policy or geographical restrictions.
4. **Preflight Rejection (CORS / Mixed Content)**: The browser refused to send the underlying request due to security constraints.
5. **Permanent Authentication Failure (`401 Unauthorized`)**: Requires acquiring new user credentials, not re-submitting expired tokens.

---

### 3.5 When to Retry (Transient Failures)
Retrying is appropriate when the failure was caused by a temporary disruption:
1. **Server Faults (`500`, `502`, `503`, `504`)**: Indicates temporary capacity limits, container restarts, or gateway timeouts.
2. **Connection Timeouts**: TCP handshake failed due to a transient network drop or temporary accept-queue saturation.
3. **Read Timeouts**: Connection was established, but stalled during data transfer due to intermediate packet loss.
4. **DNS Server Timeouts**: Upstream resolver dropped a UDP packet (no response received, distinct from an authoritative `NXDOMAIN`).
5. **Network Handover**: Mobile client transitioned interfaces mid-flight (e.g., Wi-Fi disconnected, switching to cellular data).
6. **Transient TLS Handshake Failure**: Dropped packets or timeouts during key exchange on a healthy server.

---

## 4. Idempotency: The Precondition for Safe Retries

RFC 9110 Section 9.2 defines an HTTP method as **idempotent** if:
$$\mathbf{\text{Executing the request } N \text{ times produces the exact same side-effects as executing it once.}}$$

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 IDEMPOTENCY TAXONOMY                                   │
├─────────┬──────────────┬──────────────────┬────────────────────────────────────────────┤
│ Method  │ Idempotent?  │ Retry Safe?      │ Operational Rationale                      │
├─────────┼──────────────┼──────────────────┼────────────────────────────────────────────┤
│ **GET** │ **YES**      │ Safe blindly     │ Read-only; causes no remote side-effects.  │
├─────────┼──────────────┼──────────────────┼────────────────────────────────────────────┤
│ **PUT** │ **YES**      │ Safe blindly     │ "Set resource to X". Doing it twice leaves │
│         │              │                  │ resource set to X.                         │
├─────────┼──────────────┼──────────────────┼────────────────────────────────────────────┤
│**DELETE**│ **YES**     │ Safe blindly     │ Deleting an object twice leaves it deleted.│
│         │              │                  │ (Second call may return 404, but state ok).│
├─────────┼──────────────┼──────────────────┼────────────────────────────────────────────┤
│ **POST**│ **NO**       │ **UNSAFE**       │ "Append/Charge". Retrying risks double     │
│         │              │ without key      │ payments or duplicate database records!    │
└─────────┴──────────────┴──────────────────┴────────────────────────────────────────────┘
```

### 4.1 Idempotent HTTP Methods (GET, PUT, DELETE)
- `GET /user/42`: Reading a user profile twice does not alter database state. Safe to retry on read timeouts.
- `PUT /user/42/status`: Setting status to `"active"` multiple times produces the identical end state.
- `DELETE /cart/item/3`: Removing item 3 leaves the item absent whether executed once or five times.

---

### 4.2 The Danger of POST Retries & Double Execution
Consider an e-commerce checkout flow:
1. Client sends `POST /api/charge {"amount": 100}`.
2. Server charges the user's credit card and updates the database.
3. Before the server can return `200 OK`, a network router drops the TCP connection.
4. The client experiences a read timeout.
5. If the client retries blindly, the server processes the `POST` again: **the user is charged twice**.

---

### 4.3 Idempotency Keys: Deduplication Architecture
To make non-idempotent operations safe to retry, production APIs (such as Stripe) use **Idempotency Keys**:

```
Client                                                   Server Application
  │                                                              │
  │─── POST /charge (Idempotency-Key: "uuid-1234") ─────────────>│ Checks Redis cache
  │    (Network drops confirmation response!)                    │ Key NOT found ──> Executes charge
  │                                                              │ Caches response against "uuid-1234"
  │                                                              │
  │─── RETRY: POST /charge (Idempotency-Key: "uuid-1234") ──────>│ Checks Redis cache
  │                                                              │ Key FOUND! Skips execution.
  │<── Returns CACHED 200 OK Response ───────────────────────────│ Returns original result safely!
```

- The client generates a unique UUID for the logical operation and passes it via `Idempotency-Key: <UUID>`.
- The server records the key in a shared distributed cache (e.g., Redis) before executing the transaction.
- If a duplicate request arrives with the same key, the server bypasses execution and returns the cached response.

---

## 5. The Mechanics of Retrying

### 5.1 The Naive Retry Loop & The Self-Inflicted Outage
The most common error in network client design is the unthrottled retry loop:

```python
# DANGEROUS PATTERN:
while True:
    try:
        return call_upstream(request)
    except TimeoutError:
        continue  # Immediately retry forever!
```

#### Why This Crashes Systems:
1. **No Maximum Attempt Count**: If an upstream dependency experiences a hard outage, the client enters an infinite loop, exhausting local CPU and thread pools instead of failing fast.
2. **Zero Delay**: The client hammers the failing service at wire speed, ensuring an overloaded server cannot recover.
3. **The Thundering Herd**: When thousands of clients retry simultaneously without coordination, their retries align on the same clock ticks, creating recurring traffic spikes that knock recovering servers offline.

---

### 5.2 Exponential Backoff: Decaying the Load
To allow failing backends to recover, retries should introduce progressively longer delays:

$$\mathbf{\text{Delay}} = \min\left(\mathbf{\text{Cap}}, \mathbf{\text{Base}} \times 2^{\mathbf{\text{attempt}}}\right)$$

```python
def retry_with_backoff(fn, max_attempts=5, base=0.2, cap=8.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except RetryableError:
            if attempt == max_attempts - 1:
                raise  # Max attempts exceeded; fail fast
            delay = min(cap, base * (2 ** attempt))
            time.sleep(delay)
```
- Spacing retries exponentially causes aggregate client traffic to decay over time, reducing load on struggling services.

---

### 5.3 Full Jitter: Breaking the Synchronization Lockstep
Even with exponential backoff, if 10,000 clients fail simultaneously (e.g., during a 1-second network blip), they all calculate the **exact same backoff delay**:
- Attempt 1: All 10,000 clients wait 0.4 seconds $\rightarrow$ Synchronized traffic spike!
- Attempt 2: All 10,000 clients wait 0.8 seconds $\rightarrow$ Synchronized traffic spike!

In 2015, AWS Architecture research demonstrated that adding **Full Jitter** breaks this lockstep:

$$\mathbf{\text{Delay}}_{\text{Full Jitter}} = \mathbf{\text{random.uniform}}\left(0, \min\left(\mathbf{\text{Cap}}, \mathbf{\text{Base}} \times 2^{\mathbf{\text{attempt}}}\right)\right)$$

```python
delay = min(cap, base * (2 ** attempt))
delay = random.uniform(0, delay)  # AWS Full Jitter: Uniform distribution across [0, delay]
time.sleep(delay)
```

```
AGGREGATE RETRY TRAFFIC ARRIVING AT UPSTREAM OVER TIME:

Without Jitter (Synchronized Spikes):
Requests
   ▲
   │    ||||||||                ||||||||                ||||||||
   │    ||||||||                ||||||||                ||||||||
   │    ||||||||                ||||||||                ||||||||
   └────┴───────────────────────┴───────────────────────┴────────► Time

With Full Jitter (Smoothly Decayed Background Noise):
Requests
   ▲
   │    . : : . . .
   │    . . . . . . . . . . . . .
   │    . . . . . . . . . . . . . . . . . . . . . . .
   └─────────────────────────────────────────────────────────────► Time
```

Selecting a uniform random value between $0$ and the exponential cap decorrelates client attempts across time, spreading retries into a manageable, declining background load.

---

### 5.4 Server-Directed Pacing: The `Retry-After` Header
When an upstream server sheds load, it can tell clients explicitly how long to pause:
- **`429 Too Many Requests`**: Client exceeded its allocated rate limit.
- **`503 Service Unavailable`**: Server is performing maintenance or shedding load.

Both status codes can include the `Retry-After` header:
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```
- **Rule of Architecture**: If the server provides a `Retry-After` value, **honor that duration directly**. Do not use local exponential backoff curves; the server has authoritative knowledge of its recovery timeline.

---

## 6. The Circuit Breaker Pattern: Fleet-Level Protection

Documented by Michael Nygard in *Release It!* (2007):
> *"Exponential backoff protects a service from a SINGLE client retrying too fast.  
> A Circuit Breaker protects a service from an ENTIRE FLEET calling a dependency everyone already knows is dead."*

```
                             [ CLOSED STATE ]
                      (Requests execute normally;
                       monitor failure threshold)
                            │              ▲
            Failure rate    │              │ Probe succeeds
            exceeds 50%     ▼              │
                      [ OPEN STATE ]       │
                 (Fails immediately;       │
                  no network calls made)   │
                            │              │
             Cooldown timer │              │
             expires        ▼              │
                     [ HALF-OPEN STATE ] ──┘
                     (Send 1 trial probe;
                      if fail -> OPEN)
```

### 6.1 The Three States: Closed, Open, Half-Open
1. **CLOSED**:
   - Requests execute normally.
   - The circuit breaker tracks success and failure rates in a rolling time window.
   - If the failure rate crosses a threshold (e.g., >50% errors over 10 seconds), the circuit trips **OPEN**.
2. **OPEN**:
   - The circuit breaker **fails incoming calls immediately in user-space code**.
   - No network packets leave the machine.
   - The caller receives an instant fallback response without waiting on timeouts, preserving local threads.
   - Protects downstream services from being overwhelmed while recovering.
3. **HALF-OPEN**:
   - After a cooldown period (e.g., 30 seconds), the breaker allows a **single probe request** through.
   - If the probe succeeds, the service has recovered: the breaker resets to **CLOSED**.
   - If the probe fails, the breaker returns to **OPEN** for another cooldown interval.

---

### 6.2 Protecting the Caller and Protecting the Callee
- **Protection for the Caller**: Instead of wasting 5 seconds blocked on a dead connection, the caller fails in 0 milliseconds, freeing worker threads to serve other requests.
- **Protection for the Callee**: Stops thousands of application servers from sending doomed requests to a struggling dependency.

---

## 7. Case Study: The 96 Failure Modes of a Video Streaming Player

In video streaming architectures (e.g., Netflix, YouTube, Hotstar), client resilience requires handling distinct failure types across multiple layers.

### 7.1 Why Player Resilience is Harder Than API Resilience
- **Failure Visibility**: If an API client fails, an error is logged. If a video player fails, playback freezes and the user sees a loading spinner.
- **Heterogeneous Requests**: A single playback session makes hundreds of distinct requests across different asset classes:
  - Master Playlists / Manifests (`.m3u8`, `.mpd`)
  - Media Initialization Segments (`init.mp4`)
  - Audio and Video Chunks (`segment_1080p_001.m4s`)
  - DRM License Keys (Widevine / FairPlay)
  - Video Advertisements (VAST / VMAP)
  - Telemetry & Analytics
- **Tight Latency Budgets**: A user will not tolerate minutes of exponential backoff while watching video; rebuffering budgets are measured in seconds.

---

### 7.2 Network Issues Under a Tighter Clock

```
┌──────────────────────────────┬────────────────────────────────────────────┬────────────────────────────┐
│ Failure Event                │ Player Context                             │ Resilient Action           │
├──────────────────────────────┼────────────────────────────────────────────┼────────────────────────────┤
│ **DNS Server Timeout**       │ Hostname resolution stalled.               │ Fail over to secondary CDN.│
├──────────────────────────────┼────────────────────────────────────────────┼────────────────────────────┤
│ **Manifest Read Timeout**    │ Player cannot discover upcoming segments.  │ Retry with short timeout.  │
├──────────────────────────────┼────────────────────────────────────────────┼────────────────────────────┤
│ **Segment Read Timeout**     │ Video chunk download stalled.              │ Drop bitrate rendition!    │
├──────────────────────────────┼────────────────────────────────────────────┼────────────────────────────┤
│ **Network Interface Switch** │ IP changed (Wi-Fi dropped -> LTE active).  │ Abort socket; retry fresh. │
├──────────────────────────────┼────────────────────────────────────────────┼────────────────────────────┤
│ **Wi-Fi Dropped Entirely**   │ Local buffer holds 20 seconds of video.    │ Keep playing; defer retry! │
└──────────────────────────────┴────────────────────────────────────────────┴────────────────────────────┘
```

---

### 7.3 The Live Segment 404 Exception
Under normal HTTP rules, `404 Not Found` means do not retry.

**The Exception in Live Streaming**:
- When watching a live broadcast, the player calculates that Segment #500 should exist based on stream timing.
- However, the origin encoder may still be writing the segment, or the CDN edge may still be propagating the file.
- The CDN returns a **`404 Not Found`**.
- The segment **will become available within 200–500 milliseconds**.
- **Player Policy**: Live stream players implement a **bounded short retry** (2–3 attempts with a 200 ms pause) specifically for live-edge segments. (A 404 on on-demand video remains a non-retryable error).

---

### 7.4 When the Answer is Fallback, Not Retry
Often, the correct response to a failure is falling back rather than retrying the same request:
1. **DRM License Denied**: If a Widevine L1 license request fails (e.g., HDCP hardware failure), retrying the same request will yield another denial. **Fallback**: Request Widevine L3 or drop to standard definition (SD).
2. **Unsupported Codec / Decoder Crash**: If an HEVC or AV1 hardware decoder crashes on an Android TV device, retrying the segment fails. **Fallback**: Switch to an H.264 (AVC) rendition stream.
3. **Master Manifest Failure**: If the MPEG-DASH manifest fails to load, fall back to the HLS (`.m3u8`) stream.
4. **Ad Request Timeout**: If a third-party ad server stalls, **drop the ad and resume video playback immediately**. The user should not see a frozen screen due to a failing advertisement tracker.
5. **iOS `ResetMedia`**: When Apple's low-level `AVPlayer` media pipeline enters an unrecoverable state, tear down the audio/video engine and re-instantiate it cleanly.

---

### 7.5 Business-Rule Retries vs. Transport Signals
Mature systems evaluate both transport-layer signals and application-layer metadata:
- If an API returns `403 Forbidden` with body `{"error": "subscription_expired"}`, this is a business rule: **do not retry**.
- If an API returns `403 Forbidden` with header `X-RateLimit-Action: challenge`, the client should route the user through a CAPTCHA challenge rather than repeating the request.

---

## 8. Systems That Survive: Architecture Beyond the Single Request

### 8.1 The Fastly 49-Minute Outage & `stale-if-error` (RFC 5861)
On June 8, 2021, a single customer configuration update triggered an edge software bug across Fastly's global CDN:
- **85% of Fastly's global network returned errors**, taking down Reddit, Spotify, Twitch, and government websites worldwide.
- The fault was identified in ~1 minute, but broad recovery took **49 minutes**.
- **What kept some services online?** Systems configured with **`stale-if-error` (RFC 5861)**.

```http
Cache-Control: max-age=600, stale-if-error=86400
```
- Instructs caches to serve their existing copy for up to 24 hours if the origin or upstream is unreachable.
- Even with backends offline, clients received valid cached responses rather than 502/504 errors.

---

### 8.2 The Three Levers: Failover, Feature Flags, Graceful Degradation

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│           ARCHITECTURAL LEVER        │        OPERATIONAL BEHAVIOR          │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ 1. **FAILOVER**                      │ Circuit breaker trips OPEN on        │
│                                      │ Primary -> routes calls to Backup    │
│                                      │ (secondary region, fallback CDN).    │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ 2. **FEATURE FLAGS**                 │ Toggle off non-essential components  │
│                                      │ (recommendations, comments) to shed  │
│                                      │ load from core paths (checkout).     │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ 3. **GRACEFUL DEGRADATION**          │ Return smaller or partial answers    │
│                                      │ instead of hard errors (lower video  │
│                                      │ quality, slightly stale pricing).    │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

---

### 8.3 The Second Outage: The Thundering Herd on Recovery
The most dangerous moment in an incident is often **when a recovering dependency comes back online**:
1. A database crashes under heavy load.
2. Thousands of application servers enter retry loops, waiting on backoff timers or circuit breaker cooldowns.
3. The database restarts with cold caches and empty connection pools.
4. The instant it accepts connections, thousands of waiting clients fire their retries simultaneously.
5. The recovering database receives 100% of normal traffic within its first second of uptime, **and immediately crashes again**.

```
THE RECOVERY SPIKE HAZARD:
Database Recovers ──> Thousands of waiting clients fire simultaneously ──> Database Crashes Again!
```

#### Defenses Against Recovery Collapses:
- **Full Jitter**: Distributes client retry attempts across a broad time window.
- **Staggered Circuit Breaker Probing**: Staggers Half-Open trial requests across instances rather than testing all at once.
- **Server-Side Slow-Start / Traffic Ramping**: Incoming traffic to recovering instances is ramped up gradually (e.g., 5% $\rightarrow$ 20% $\rightarrow$ 100%) to allow caches to warm up safely.

---

## 9. The Unified Failure Resolution Flowchart

```
                          [ INCOMING REQUEST FAILS ]
                                      │
                                      ▼
                      ┌───────────────────────────────┐
                      │    1. CLASSIFY THE FAILURE    │
                      │  - Where on timeline did it   │
                      │    fail? (DNS / Handshake /   │
                      │    Read Timeout)              │
                      │  - Is it 4xx, 5xx, or dropped?│
                      │  - Is the method idempotent?  │
                      └───────────────┬───────────────┘
                                      │
            ┌─────────────────────────┴─────────────────────────┐
            ▼                                                   ▼
   [ STRUCTURAL FAILURE ]                              [ TRANSIENT FAILURE ]
   - 404 Not Found                                     - 5xx Server Error
   - 401/403 Policy / Bad Auth                         - Connection / Read Timeout
   - Geo-blocked                                       - Network handover (4G -> 5G)
   - Preflight CORS rejection                          - DNS query dropped
            │                                                   │
            ▼                                                   ▼
   [ STOP RETRYING ]                                   [ SAFE TO RETRY? ]
   - Handle via business flow                          - Check idempotency or key
   - Refresh tokens                                             │
   - Show error message                                         ▼
                                                       ┌───────────────────────────────┐
                                                       │   2. EVALUATE CIRCUIT BREAKER │
                                                       └───────────────┬───────────────┘
                                                                       │
                                              ┌────────────────────────┴────────────────────────┐
                                              ▼                                                 ▼
                                     [ BREAKER IS OPEN ]                              [ BREAKER IS CLOSED ]
                                     - Fail immediately                               - Check Retry-After
                                     - Zero network calls                             - Compute Exponential Backoff
                                     - Trigger Fallback /                             - Apply AWS Full Jitter
                                       Graceful Degradation                           - Execute bounded retry attempt
                                                                                                │
                                                                       ┌────────────────────────┴────────────────────────┐
                                                                       ▼                                                 ▼
                                                              [ ATTEMPTS EXCEEDED ]                             [ RETRY SUCCEEDS ]
                                                              - Trip circuit breaker                            - Reset attempt counter
                                                              - Fall back to secondary CDN/region               - Return response to user
                                                              - Serve stale cache (stale-if-error)
```

---

## 10. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
Why are a DNS timeout and an HTTP response read timeout considered fundamentally different failure modes, despite both appearing as "timeouts" in logs?
- A) DNS timeouts use TCP, while read timeouts use UDP.
- B) A DNS timeout occurs before a connection is established, guaranteeing no server mutation occurred; a read timeout occurs after data was transmitted, meaning the server may have completed the operation before the connection dropped.
- C) Read timeouts are caused by hardware bit errors.
- D) DNS timeouts only occur on local loopback interfaces.

**Answer: B**  
**Explanation:** A DNS failure happens before any connection is made, so the remote server has not processed the request. A read timeout occurs while waiting for the response after the request was sent, meaning the server may have executed the transaction, making retries risky for non-idempotent operations.

---

### Question 2
Under RFC 9110 Section 9.2, why is it safe to retry an HTTP `PUT` request after a network timeout, but dangerous to blindly retry an HTTP `POST` request?
- A) `PUT` requests do not carry request bodies.
- B) `PUT` is idempotent, meaning applying it multiple times produces the same end state; `POST` is non-idempotent, meaning retrying risks duplicate side effects (e.g., charging a card twice).
- C) `POST` requests are limited to 1,024 bytes.
- D) Web servers reject retried `PUT` requests with status 400.

**Answer: B**  
**Explanation:** Idempotent methods like `PUT` replace resource state, so executing them repeatedly leaves the resource in the same state. `POST` operations typically append or process data, so repeating them without deduplication can result in duplicate transactions.

---

### Question 3
How does an Idempotency Key protect systems when retrying `POST` operations?
- A) It encrypts the payload using AES-GCM.
- B) The client sends a unique identifier with the request, allowing the server to recognize duplicate attempts, skip re-execution, and return the cached result.
- C) It bypasses intermediate reverse proxies.
- D) It forces the operating system to avoid packet fragmentation.

**Answer: B**  
**Explanation:** By associating requests with a unique Idempotency Key, the server can record processed keys in a cache. If a retried request arrives with the same key, the server returns the cached response rather than running the transaction again.

---

### Question 4
Why is AWS Full Jitter ($\text{delay} = \text{random.uniform}(0, \text{backoff})$) more effective at preventing thundering herd problems than standard exponential backoff ($\text{delay} = \text{base} \times 2^{\text{attempt}}$)?
- A) Full Jitter increases maximum retry delays to one hour.
- B) Standard exponential backoff causes clients that failed at the same time to retry in synchronized waves; Full Jitter distributes retries uniformly across the entire backoff window, smoothing aggregate traffic.
- C) Full Jitter runs within the operating system kernel.
- D) Full Jitter eliminates the need for retry limits.

**Answer: B**  
**Explanation:** Plain exponential backoff keeps synchronized clients aligned on the same retry intervals. Full Jitter selects a random value between 0 and the current backoff ceiling, decorrelating retry times across the client fleet.

---

### Question 5
If a server returns `HTTP/1.1 429 Too Many Requests` with the header `Retry-After: 20`, what should the client do?
- A) Retry immediately using exponential backoff.
- B) Pause for at least 20 seconds before retrying, respecting the server's rate-limiting schedule.
- C) Switch transport protocols to UDP.
- D) Abort the connection and mark the host as permanently unavailable.

**Answer: B**  
**Explanation:** The `Retry-After` header indicates when the server expects to accept requests again. Clients should honor this value rather than generating independent backoff schedules.

---

### Question 6
What is the primary function of the **OPEN** state in the Circuit Breaker pattern?
- A) To accept incoming connections and spawn worker threads.
- B) To fail requests immediately in client code without attempting network calls, protecting callers from timeout delays and preventing load on failing dependencies.
- C) To run background data integrity checks.
- D) To bypass TLS certificate validation.

**Answer: B**  
**Explanation:** When tripped to OPEN, the circuit breaker fails calls locally without making network requests. This prevents callers from blocking on timeouts and shields struggling backends from additional load.

---

### Question 7
In the Circuit Breaker pattern, how does the system transition from **OPEN** to **CLOSED**?
- A) An administrator must restart the application server.
- B) After a cooldown interval, the breaker transitions to HALF-OPEN and sends a trial probe; if the probe succeeds, the breaker resets to CLOSED.
- C) The operating system flushes its socket buffers.
- D) The client flushes its DNS cache.

**Answer: B**  
**Explanation:** After a cooldown period, the breaker enters HALF-OPEN mode and allows a single probe request through. If it succeeds, the breaker resets to CLOSED and resumes normal traffic flow.

---

### Question 8
Why do live video streaming players make an exception to retry on `404 Not Found` responses for the newest stream segments?
- A) 404 responses are treated as network timeouts by video decoders.
- B) In live streaming, a player may request a newly generated segment before origin encoding or CDN propagation finishes; the segment will typically become available within a few hundred milliseconds.
- C) Live video streams do not use standard HTTP status codes.
- D) Caching proxies transform 404 responses into video frames.

**Answer: B**  
**Explanation:** Live stream players request segments close to the live edge. A 404 often means the segment has not finished propagating through the CDN yet, so a brief, bounded retry allows the player to fetch it once propagation completes.

---

### Question 9
If a video player encounters an unrecoverable DRM license denial (e.g., hardware security level L1 failure), what is the appropriate recovery action?
- A) Retrying the same license request with exponential backoff.
- B) Falling back to a lower DRM security tier (e.g., L3) or requesting a standard-definition rendition, rather than repeating an identical request.
- C) Closing the underlying TCP socket and clearing DNS.
- D) Re-requesting the master manifest file.

**Answer: B**  
**Explanation:** A DRM denial is typically a policy or hardware limitation rather than a transient network drop. Retrying the same request will produce the same result, so the player must fall back to an alternate security level or rendition.

---

### Question 10
Why should video players fall back to main content immediately when a video advertisement fails to load, rather than initiating a retry loop?
- A) Ad servers reject retried requests.
- B) Viewers have minimal patience for ad loading delays, and stalling playback on a failed ad degrades user experience without delivering value.
- C) Ad payloads cannot be delivered over HTTPS.
- D) Advertising standards require closing player sockets on failure.

**Answer: B**  
**Explanation:** Ad insertions have strict time budgets. If an ad tracker or video asset fails, retrying risks stalling playback; the player should fall back directly to primary content.

---

### Question 11
How does the `stale-if-error` Cache-Control directive (RFC 5861) improve availability during origin outages?
- A) It directs browsers to ignore 404 errors.
- B) It allows caches to serve expired content if the origin server is unreachable or returning 5xx errors, maintaining availability with existing data.
- C) It reroutes traffic through alternative DNS servers.
- D) It enables client-side database synchronization.

**Answer: B**  
**Explanation:** `stale-if-error` instructs caches to return their existing copy if the origin is failing. Serving slightly stale data keeps sites functional during backend outages.

---

### Question 12
Why is the moment a failing service recovers often considered the most dangerous point of an operational incident?
- A) The service runs with inverted firewall rules.
- B) Numerous clients backing off or waiting on circuit breakers retry simultaneously when the service becomes reachable, overwhelming cold caches and causing a secondary crash.
- C) Operating systems reset their file descriptor limits on recovery.
- D) DNS records are cleared when services restart.

**Answer: B**  
**Explanation:** Waiting clients often fire retries as soon as a service becomes available. A recovering service with cold caches and empty connection pools can be overwhelmed by this traffic burst and crash again.

---

### Question 13
How does using Feature Flags support graceful degradation during infrastructure incidents?
- A) By rebuilding and redeploying application containers.
- B) By disabling non-critical features (e.g., recommendations, comment feeds) via runtime configuration, reducing load on core paths without code deployments.
- C) By encrypting database tables during traffic spikes.
- D) By clearing cache indexes in shared memory.

**Answer: B**  
**Explanation:** Feature flags allow teams to turn off heavy, non-essential background tasks at runtime. Shedding non-critical load helps protect core transaction paths during traffic surges or partial outages.

---

### Question 14
In the failure classification framework, how should a client handle a `401 Unauthorized` response with an expired access token?
- A) Retry the exact same request immediately.
- B) Treat it as a recovery flow: request a new token using a refresh token and submit a new request with the updated `Authorization` header.
- C) Trip the circuit breaker to OPEN.
- D) Discard all client-side cached assets.

**Answer: B**  
**Explanation:** Submitting an expired token again will yield another 401 error. The client must execute a token refresh flow and send a new request with the updated credential.

---

### Question 15
What is the core takeaway of the principle: "A system that can answer with slightly-wrong data survives outages that a system demanding perfectly-fresh-or-nothing does not"?
- A) Caching should be disabled across distributed architectures.
- B) Systems designed for graceful degradation and stale data fallbacks maintain availability during partial failures, whereas brittle architectures crash when any dependency stalls.
- C) Database consistency is unachievable in distributed networks.
- D) Network timeouts should be set to infinity.

**Answer: B**  
**Explanation:** Architectures that can return cached, estimated, or degraded results during disruptions maintain service availability. Systems that require complete, fresh data across all dependencies fail when any component encounters an issue.

---

## 11. Session 7 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **6-Stage Lifecycle** | DNS $\rightarrow$ TCP $\rightarrow$ TLS $\rightarrow$ Write $\rightarrow$ Processing $\rightarrow$ Read. Stage location determines retry safety. |
| **DNS vs Read Timeout** | DNS failure carries zero remote side-effects; Read timeout carries risk of unconfirmed backend execution. |
| **Retry vs Handling** | A retry repeats the **exact same request**. Refreshing tokens or calling fallbacks is handling/failover. |
| **HTTP Status Rules** | `4xx` = client error (do not retry). `5xx` = server error (retryable if idempotent). |
| **Idempotency (RFC 9110)**| `GET`, `PUT`, `DELETE` are idempotent (safe to retry). `POST` is non-idempotent (requires `Idempotency-Key`). |
| **Naive Retry Flaws** | Infinite loops, zero backoff delay, and synchronized client thundering herds. |
| **Exponential Backoff** | $\text{delay} = \min(\text{cap}, \text{base} \times 2^{\text{attempt}})$. Progressively decays aggregate client load. |
| **AWS Full Jitter** | $\text{delay} = \text{random.uniform}(0, \text{backoff})$. Decorrelates client retries across time. |
| **`Retry-After` Header**| Authoritative server directive (`429` / `503`). Clients should honor the specified duration directly. |
| **Circuit Breaker** | Tracks dependency health: **CLOSED** (normal), **OPEN** (fast local fail), **HALF-OPEN** (trial probe). |
| **Live Segment 404** | Exception to 404 rules: live video chunks can take hundreds of ms to encode and propagate. |
| **Fallback Over Retry** | DRM denials $\rightarrow$ lower security tier; Decoder crash $\rightarrow$ fallback codec; Ad timeout $\rightarrow$ play content. |
| **`stale-if-error` (RFC 5861)**| Caches serve existing copies during backend outages, maintaining availability during incidents. |
| **The Second Outage** | Recovering services face synchronized retry spikes from waiting clients. Mitigate with jitter and traffic ramping. |
