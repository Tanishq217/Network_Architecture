# Chapter 05: Evolution of HTTP — 1996 to Now

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 5 / Week 5  
> **Core Theme**: *"Same words. Three completely different ways of writing them down."*  
> **Key Protocols & RFCs**: HTTP/1.0 (RFC 1945), Early HTTP/1.1 (RFC 2068), Modern HTTP/1.1 (RFC 2616, RFC 7230–7235, RFC 9110/9112), HTTP/2 (RFC 7540, RFC 9113), HPACK (RFC 7541), QUIC (RFC 9000, RFC 9001), HTTP/3 (RFC 9114), and QPACK (RFC 9204).

---

## Table of Contents
1. [The Four Generations of HTTP: Overview & Core Philosophy](#1-the-four-generations-of-http-overview--core-philosophy)
2. [HTTP/1.0 (RFC 1945, May 1996): The Ephemeral Web](#2-http10-rfc-1945-may-1996-the-ephemeral-web)
   - [2.1 The Complete Protocol on One Slide](#21-the-complete-protocol-on-one-slide)
   - [2.2 The Socket-Per-Request Dilemma & Serialization Cost](#22-the-socket-per-request-dilemma--serialization-cost)
   - [2.3 The Missing `Host` Header & The 1:1 IP Address Bottleneck](#23-the-missing-host-header--the-11-ip-address-bottleneck)
   - [2.4 Caching with Clocks: The Three Flaws of Timestamps](#24-caching-with-clocks-the-three-flaws-of-timestamps)
3. [Early HTTP/1.1 (RFC 2068, Jan 1997): Persistence & Extensibility](#3-early-http11-rfc-2068-jan-1997-persistence--extensibility)
   - [3.1 Persistent Connections by Default & The EOF Dilemma](#31-persistent-connections-by-default--the-eof-dilemma)
   - [3.2 The Mandatory `Host` Header: One IP, Millions of Sites](#32-the-mandatory-host-header-one-ip-millions-of-sites)
   - [3.3 Fingerprints Over Clocks: `ETag` & Conditional Requests](#33-fingerprints-over-clocks-etag--conditional-requests)
   - [3.4 Content Negotiation & The `Vary` Header](#34-content-negotiation--the-vary-header)
   - [3.5 Range Requests: Resumable Transfers & Cloud Data Lakes](#35-range-requests-resumable-transfers--cloud-data-lakes)
   - [3.6 Chunked Transfer Encoding: Sending Bodies Without Known Sizes](#36-chunked-transfer-encoding-sending-bodies-without-known-sizes)
   - [3.7 The Rest of the Vocabulary: Methods, Status Codes & Cache-Control](#37-the-rest-of-the-vocabulary-methods-status-codes--cache-control)
   - [3.8 The "Two-Connection" Rule & Domain Sharding](#38-the-two-connection-rule--domain-sharding)
4. [Modern HTTP/1.1: Tightening, Attack Vectors & RFC Restructuring](#4-modern-http11-tightening-attack-vectors--rfc-restructuring)
   - [4.1 RFC 2616 (1999): Redefining Redirects & The 307 Fix](#41-rfc-2616-1999-redefining-redirects--the-307-fix)
   - [4.2 Postel's Law Loses: The Mechanics of HTTP Request Smuggling](#42-postels-law-loses-the-mechanics-of-http-request-smuggling)
   - [4.3 Why JWTs Use Base64URL: Grammar Restrictions](#43-why-jwts-use-base64url-grammar-restrictions)
   - [4.4 Microservices for RFCs: From RFC 2616 to RFC 7230–7235 and RFC 9110–9114](#44-microservices-for-rfcs-from-rfc-2616-to-rfc-72307235-and-rfc-91109114)
   - [4.5 Status 418: The Permanent Easter Egg](#45-status-418-the-permanent-easter-egg)
5. [The Bill for HTTP/1.1: Three Structural Flaws](#5-the-bill-for-http11-three-structural-flaws)
   - [5.1 Application Head-of-Line (HoL) Blocking & Pipelining Failure](#51-application-head-of-line-hol-blocking--pipelining-failure)
   - [5.2 The Root Cause: Order IS the Identifier](#52-the-root-cause-order-is-the-identifier)
   - [5.3 Header Bloat: 97.5% Byte-for-Byte Redundancy](#53-header-bloat-975-byte-for-byte-redundancy)
   - [5.4 The Nagle + Delayed ACK Latency Penalty](#54-the-nagle--delayed-ack-latency-penalty)
6. [HTTP/2 (RFC 7540 / RFC 9113): Binary Framing & Multiplexing](#6-http2-rfc-7540--rfc-9113-binary-framing--multiplexing)
   - [6.1 Google SPDY & The Path to Standardization](#61-google-spdy--the-path-to-standardization)
   - [6.2 The Connection Preface: 24 Bytes That Are Also a Joke](#62-the-connection-preface-24-bytes-that-are-also-a-joke)
   - [6.3 Protocol Negotiation: TLS ALPN vs. Cleartext `h2c`](#63-protocol-negotiation-tls-alpn-vs-cleartext-h2c)
   - [6.4 The 9-Byte Binary Frame Header & Stream Identifiers](#64-the-9-byte-binary-frame-header--stream-identifiers)
   - [6.5 The Ten Frame Types](#65-the-ten-frame-types)
   - [6.6 HPACK (RFC 7541): Static Tables, Dynamic Tables & Huffman Encoding](#66-hpack-rfc-7541-static-tables-dynamic-tables--huffman-encoding)
   - [6.7 Why Not Gzip Headers? The CRIME Attack (2012)](#67-why-not-gzip-headers-the-crime-attack-2012)
   - [6.8 Multiplexing in Action: Out-of-Order Concurrent Streams](#68-multiplexing-in-action-out-of-order-concurrent-streams)
   - [6.9 The Thread-Safety Trap: Handling Streams Concurrently](#69-the-thread-safety-trap-handling-streams-concurrently)
   - [6.10 The Death of Server Push & The Rise of 103 Early Hints](#610-the-death-of-server-push--the-rise-of-103-early-hints)
7. [The Fatal Flaw of HTTP/2 & The Rise of QUIC / HTTP/3](#7-the-fatal-flaw-of-http2--the-rise-of-quic--http3)
   - [7.1 Transport-Layer (TCP) Head-of-Line Blocking](#71-transport-layer-tcp-head-of-line-blocking)
   - [7.2 Why UDP? Middlebox Ossification & Protocol 253](#72-why-udp-middlebox-ossification--protocol-253)
   - [7.3 Handshake Consolidation: 1-RTT & 0-RTT Resumption](#73-handshake-consolidation-1-rtt--0-rtt-resumption)
   - [7.4 0-RTT Security Trade-offs: Replay Attacks & 425 Too Early](#74-0-rtt-security-trade-offs-replay-attacks--425-too-early)
   - [7.5 True Stream Independence: Dropped Packets Isolated](#75-true-stream-independence-dropped-packets-isolated)
   - [7.6 Connection Migration: 64-Bit Connection IDs Survive IP Changes](#76-connection-migration-64-bit-connection-ids-survive-ip-changes)
   - [7.7 QPACK (RFC 9204): Solving HPACK's Head-of-Line Blocking](#77-qpack-rfc-9204-solving-hpacks-head-of-line-blocking)
   - [7.8 Protocol Discovery, Alt-Svc & The UDP Drop Fallback](#78-protocol-discovery-alt-svc--the-udp-drop-fallback)
8. [The State of Global Adoption in 2026](#8-the-state-of-global-adoption-in-2026)
9. [The Four Core Architectural Lessons](#9-the-four-core-architectural-lessons)
10. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#10-comprehensive-review--practice-mcqs-with-detailed-explanations)
11. [Session 5 Summary Cheat Sheet](#11-session-5-summary-cheat-sheet)

---

## 1. The Four Generations of HTTP: Overview & Core Philosophy

The Hypertext Transfer Protocol (HTTP) has evolved across four distinct wire formats since its inception. While the core application semantics (`GET`, `POST`, status codes, headers) have remained remarkably consistent for three decades, the underlying wire representation was rewritten repeatedly to eliminate transport-level performance bottlenecks:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   THE EVOLUTION OF HTTP ON THE WIRE                                    │
├───────────┬──────────────┬──────────────────┬──────────────────┬───────────────────────────────────────┤
│ Version   │ Standard RFC │ Transport Layer  │ Wire Format      │ Primary Limitation / Bottleneck       │
├───────────┼──────────────┼──────────────────┼──────────────────┼───────────────────────────────────────┤
│ **HTTP/1.0**│ RFC 1945     │ TCP              │ Plaintext ASCII  │ One request per TCP connection;       │
│           │ (May 1996)   │                  │                  │ Socket closed at response EOF.        │
├───────────┼──────────────┼──────────────────┼──────────────────┼───────────────────────────────────────┤
│ **HTTP/1.1**│ RFC 2068/2616│ TCP              │ Plaintext ASCII  │ Persistent connections, but strict    │
│           │ (1997 / 1999)│                  │                  │ in-order responses (HoL Blocking).    │
├───────────┼──────────────┼──────────────────┼──────────────────┼───────────────────────────────────────┤
│ **HTTP/2**  │ RFC 7540/9113│ TCP + TLS        │ Binary Framing   │ Multiplexed streams over single TCP;  │
│           │ (2015 / 2022)│                  │ (HPACK)          │ TCP packet loss stalls ALL streams.   │
├───────────┼──────────────┼──────────────────┼──────────────────┼───────────────────────────────────────┤
│ **HTTP/3**  │ RFC 9000/9114│ QUIC over UDP    │ Binary Framing   │ Fully independent streams;            │
│           │ (2021 / 2022)│ + TLS 1.3        │ (QPACK)          │ 3–5% of networks block UDP traffic.   │
└───────────┴──────────────┴──────────────────┴──────────────────┴───────────────────────────────────────┘
```

> **The Fundamental Realization**:  
> *"Text based — you die early, or you live long enough..."* Text framing is why HTTP won the 1990s: anyone could write a server in shell or C, and anyone could debug it with `telnet`. But text without explicit lengths and without stream identifiers created structural scalability limits. HTTP/2 and HTTP/3 did not replace HTTP; they provided new binary encodings for the exact same semantic language.

---

## 2. HTTP/1.0 (RFC 1945, May 1996): The Ephemeral Web

### 2.1 The Complete Protocol on One Slide
Published as an informational memo in May 1996 by Tim Berners-Lee, Roy Fielding, and Henrik Frystyk, RFC 1945 documented the deployed practice of the early web:
- **Methods**: Strictly three core verbs: `GET`, `HEAD`, `POST` (Appendix D noted experimental draft methods `PUT`, `DELETE`, `LINK`, `UNLINK`).
- **Headers**: Exactly 16 headers: `Allow`, `Authorization`, `Content-Encoding`, `Content-Length`, `Content-Type`, `Date`, `Expires`, `From`, `If-Modified-Since`, `Last-Modified`, `Location`, `Pragma`, `Referer`, `Server`, `User-Agent`, `WWW-Authenticate`.
- **Status Codes**: 16 standard codes across the 2xx, 3xx, 4xx, and 5xx families (compared to 62 officially registered by IANA today).

---

### 2.2 The Socket-Per-Request Dilemma & Serialization Cost
Section 1.3 of RFC 1945 specified the foundational operational rule of HTTP/1.0:
> *"Except for experimental protocol extensions, current practice requires that the connection be established by the client prior to each request and closed by the server after sending the response."*

```
HTTP/1.0 CONNECTION LIFECYCLE (FETCHING 24 ASSETS AT 200ms RTT):

Asset 1 : [ TCP Handshake: 200ms ] ──> [ Request/Response: 200ms ] ──> [ FIN Teardown ] (400ms)
Asset 2 : [ TCP Handshake: 200ms ] ──> [ Request/Response: 200ms ] ──> [ FIN Teardown ] (400ms)
...
Asset 24: [ TCP Handshake: 200ms ] ──> [ Request/Response: 200ms ] ──> [ FIN Teardown ] (400ms)

Total Latency Tax = (24 * 200ms Handshake) + (24 * 200ms Transfer) = 9.6 SECONDS!
```

#### The Real Performance Costs:
1. **Handshake Serialization**: Every single inline CSS file, JavaScript bundle, and image required an independent 3-way TCP handshake ($1\text{ RTT}$).
2. **TCP Slow Start Penalty**: Every new TCP connection starts with an initial congestion window ($CWND$) of just 1 to 4 segments. Before TCP could ramp up throughput, the transfer finished and the socket was destroyed.
3. **The EOF Hack**: In HTTP/1.0, the server signaled the end of the HTTP response body simply by **closing the TCP socket (sending FIN)**. The client read until `read() == 0` (EOF). Attempting to send a second request on the same socket failed immediately because the connection was already closed.

---

### 2.3 The Missing `Host` Header & The 1:1 IP Address Bottleneck
In HTTP/1.0, the client request format was:
```text
GET /index.html HTTP/1.0\r\n
\r\n
```
Notice what is missing: **There is no `Host` header.**
- The server only knew which website was being requested from the **destination IP address** in the TCP/IP packet header (`socket` 4-tuple).
- **The Consequence**: A single physical server hosting 50 independent websites required **50 unique public IPv4 addresses**!
- With ~1.1 billion websites online today, the lack of a `Host` header would have consumed over one-third of the entire IPv4 address space ($2^{32} \approx 4.29\text{ billion}$) purely for web server binding.

---

### 2.4 Caching with Clocks: The Three Flaws of Timestamps
HTTP/1.0 introduced cache validation using human-readable clock dates:
- Server sends: `Last-Modified: Fri, 04 Sep 2026 07:11:14 GMT`
- Client revalidates: `If-Modified-Since: Fri, 04 Sep 2026 07:11:14 GMT`
- If unchanged, the server returns `304 Not Modified` with zero body bytes.

```
THE THREE FLAWS OF TIMESTAMP-BASED CACHING:
┌────────────────────┬───────────────────────────────────────────────────────────────────┐
│ 1. 1-Second Window │ Timestamps carry no sub-second precision. Two writes within one   │
│                    │ second appear identical to all downstream caches.                 │
├────────────────────┼───────────────────────────────────────────────────────────────────┤
│ 2. Clock Drift     │ Server clocks, client clocks, and proxy clocks drift apart. RFCs  │
│                    │ had to write complex heuristics to arbitrate clock discrepancies. │
├────────────────────┼───────────────────────────────────────────────────────────────────┤
│ 3. mtime != Identity│ File modification time (mtime) does not represent content. A git   │
│                    │ checkout or asset rebuild updates mtimes without altering a byte. │
└────────────────────┴───────────────────────────────────────────────────────────────────┘
```

> **Did HTTP/1.0 have Head-of-Line (HoL) Blocking?**  
> Historically, many claim HTTP/1.0 suffered from Head-of-Line blocking. Strictly speaking, **it did not**. Head-of-Line blocking requires a queue to block on, and HTTP/1.0 lacked pipelining or persistent queues entirely. HTTP/1.0 suffered from a **pure serialization cost** (one round trip per asset). True application Head-of-Line blocking was introduced in HTTP/1.1 as an unintended side effect of persistent pipelining!

---

## 3. Early HTTP/1.1 (RFC 2068, Jan 1997): Persistence & Extensibility

In January 1997, the IETF published RFC 2068, followed by RFC 2616 in June 1999, fundamentally modernizing HTTP.

### 3.1 Persistent Connections by Default & The EOF Dilemma
- In HTTP/1.0, `Connection: keep-alive` was an experimental, non-standard extension that servers frequently ignored or mishandled.
- In HTTP/1.1, **connections are persistent by default**. A client or server must explicitly send `Connection: close` to opt out.
- Multiple requests and responses now execute over a single TCP connection, eliminating repeated 3-way handshakes.

#### The Framing Challenge:
Because the connection remains open, the server can no longer signal the end of a response by closing the socket (EOF).
This introduces a critical requirement:
$$\mathbf{\text{Where does this response end and the next request begin?}}$$
- In HTTP/1.1, the parser **must read exactly `Content-Length` bytes** and stop. Byte $N+1$ belongs to the subsequent request. If a server miscalculates `Content-Length` by a single byte, the entire connection desynchronizes.

---

### 3.2 The Mandatory `Host` Header: One IP, Millions of Sites
RFC 2068 Section 14.23 made the `Host` header mandatory:
```http
GET / HTTP/1.1
Host: site-a.local
```
- A client omitting `Host` in HTTP/1.1 must be rejected with **`400 Bad Request`**.
- This enabled **Name-Based Virtual Hosting**: a single server process listening on one IP address can host hundreds of thousands of independent domains.

```
THE LOGICAL DESTINATION PATTERN ACROSS THIRTY YEARS:
Layer          The Problem                           The Architectural Solution
───────────────────────────────────────────────────────────────────────────────────
HTTP/1.0       4-tuple is the only address           Host: header inside message
TLS            Host header is encrypted              SNI (Server Name Indication) in ClientHello
HTTP/2         Headers are compressed & binary       :authority pseudo-header
Cloud / K8s    Many services on one IP               Ingress Host rules / Routing keys
```

---

### 3.3 Fingerprints Over Clocks: `ETag` & Conditional Requests
RFC 2068 introduced the **Entity Tag (`ETag`)**: an opaque cryptographic fingerprint (e.g., an SHA-256 hash or inode/mtime hash) representing the exact state of a resource:
- Server sends: `ETag: "55274ac22c651b90"`
- Client revalidates: `If-None-Match: "55274ac22c651b90"`
- If matched, server returns `304 Not Modified`, saving entire file transfers.

#### Strong vs. Weak Validators:
- **Strong ETag (`"xyz"`)**: Guarantees byte-for-byte identity. Mandatory for `Range` requests.
- **Weak ETag (`W/"xyz"`)**: Guarantees semantic equivalence (e.g., an HTML page with updated whitespace or compressed via gzip).
- **`If-Match` for Concurrency**: Solves the "lost update" problem in REST APIs. A client sending `PUT /article` includes `If-Match: "version-1"`. If another user modified the article in the interim, the server aborts the write with **`412 Precondition Failed`**.

---

### 3.4 Content Negotiation & The `Vary` Header
Clients announce decompression capabilities via `Accept-Encoding: gzip, deflate, br`.
- When tested against realistic text payloads, Gzip compresses web assets down to **2.8% of their original size**.
- *The Key Insight*: Text was never the performance bottleneck of HTTP/1.1 payloads; the body was compressed. The bottleneck was that **HTTP/1.1 headers remained uncompressed text**.

#### The `Vary` Header:
Informs caching proxies that multiple representations of a URL exist:
- `Vary: Accept-Encoding`: Cache stores separate gzip and raw entries based on client encoding support.
- `Vary: User-Agent`: **Anti-pattern**. Variations in user-agent strings create thousands of distinct cache keys, reducing cache hit rates to near zero.
- `Vary: *`: Disables downstream caching entirely.

---

### 3.5 Range Requests: Resumable Transfers & Cloud Data Lakes
HTTP/1.1 allows requesting byte ranges via the `Range` header:
```http
Range: bytes=100-199
```
- Server response: **`206 Partial Content`**, with `Content-Range: bytes 100-199/202000`.
- Suffix ranges: `Range: bytes=-50` (requests the final 50 bytes of a file).
- Out-of-bounds range: **`416 Range Not Satisfiable`**.

```
WHAT BYTE RANGES ENABLED:
1. Video Scrubbing     : Players jump to specific timestamps without downloading intermediate data.
2. Resumable Downloads : Interrupted downloads resume from the exact byte offset where they failed.
3. Columnar Data Lakes : In Parquet, ORC, and ZIP files, the directory footer is located at the 
                         END of the file. Clients read the final few KB, parse the footer, and 
                         use Range requests to query only the specific columns needed!
```

---

### 3.6 Chunked Transfer Encoding: Sending Bodies Without Known Sizes
When serving dynamic content (such as database streams or real-time templates), the server does not know the total byte size in advance. Calculating `Content-Length` would require buffering the entire response in memory.

RFC 2068 resolved this with **`Transfer-Encoding: chunked`**:
```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

4\r\n
Wiki\r\n
6\r\n
pedia \r\n
E\r\n
in \r\n
chunks.\r\n
0\r\n
\r\n
```

- Each chunk starts with its size in **hexadecimal ASCII**, followed by `\r\n`, the chunk payload, and another `\r\n`.
- A zero-length chunk (`0\r\n\r\n`) signals the end of the stream.
- **Architectural Pattern**: Chunked encoding nests length-prefixed chunks within an empty-chunk sentinel delimiter. This enables streaming JSON, Server-Sent Events (SSE), and gRPC trailers.

---

### 3.7 The Rest of the Vocabulary: Methods, Status Codes & Cache-Control
- **`OPTIONS`**: Queries supported methods on a resource (used by browsers for CORS preflight checks).
- **`PUT` & `DELETE`**: Standard write and remove operations for REST APIs.
- **`TRACE`**: Echoes request headers back to the client (disabled globally across modern web servers due to Cross-Site Tracing attacks).
- **`Expect: 100-continue`**: Used by clients uploading large files. The client sends headers with `Expect: 100-continue`; the server responds with `100 Continue` if accepted, or `401`/`413` to reject the upload before large payloads are transmitted.
- **`Cache-Control`**:
  - `no-cache`: The asset may be cached, but **must be revalidated with the origin server** via an ETag check before every use.
  - `no-store`: The asset must **never be written to disk or memory cache** under any circumstances.

---

### 3.8 The "Two-Connection" Rule & Domain Sharding
Section 8.1.4 of RFC 2068 stated:
> *"A single-user client SHOULD NOT maintain more than 2 connections with any server or proxy."*

- **The Browser Rebellion**: To load asset-heavy web pages faster, browser vendors increased their internal connection limits to **6 concurrent TCP connections per host**.
- **Domain Sharding**: Web developers worked around this cap by hosting static assets across multiple subdomains (`img1.example.com`, `img2.example.com`, `img3.example.com`). This allowed browsers to open up to 18 parallel TCP connections to the same backend infrastructure.
- In 2014, RFC 7230 formally removed the 2-connection restriction from the specification.

---

## 4. Modern HTTP/1.1: Tightening, Attack Vectors & RFC Restructuring

### 4.1 RFC 2616 (1999): Redefining Redirects & The 307 Fix
- Under RFC 2068, status code **`302`** specified that the request method should be preserved across the redirect.
- However, popular web browsers rewrote incoming `POST` redirects to `GET` requests upon receiving a 302.
- **The Standard Yields to Deployed Reality**: Rather than breaking existing browser behavior, RFC 2616 renamed 302 from "Moved Temporarily" to "Found", formalized the method rewrite, and introduced **`307 Temporary Redirect`** to enforce strict method preservation.

---

### 4.2 Postel's Law Loses: The Mechanics of HTTP Request Smuggling
Jon Postel's Robustness Principle states: *"Be conservative in what you send, be liberal in what you accept."*

In RFC 2616 Section 4.4, the IETF applied this principle:
> *"If a message is received with both a Transfer-Encoding header field and a Content-Length header field, the latter MUST be ignored."*

#### The Vulnerability: HTTP Request Smuggling
When requests pass through a chain of proxies (e.g., a CDN in front of an origin server), security breaks down if the two systems prioritize conflicting framing headers differently:

```
ATTACK PAYLOAD TRANSMITTED ON A SINGLE TCP CONNECTION:
POST / HTTP/1.1
Host: vulnerable-site.com
Content-Length: 6
Transfer-Encoding: chunked

0\r\n
\r\n
GET /admin HTTP/1.1
Host: vulnerable-site.com
```

```
┌────────────────────────────────────────────────────────────────────────┐
│ CDN / FRONTEND PROXY (Uses Content-Length: 6):                         │
│ Reads exactly 6 bytes ("0\r\n\r\n\r\n"). Sees ONE harmless POST request. │
├────────────────────────────────────────────────────────────────────────┤
│ BACKEND APPLICATION (Uses Transfer-Encoding: chunked):                 │
│ Reads chunk 0 (end of request 1). The trailing text "GET /admin" is     │
│ parsed as an entirely NEW unauthenticated request from the pipeline!  │
└────────────────────────────────────────────────────────────────────────┘
```

#### The Reversal:
In June 2022, RFC 9112 Section 6.3 reversed the historical rule:
> *"A server MAY reject a request that contains both... Regardless, the server MUST close the connection after responding."*

Postel's Law works between two cooperating systems, but introduces vulnerabilities when applied across distributed proxy chains.

---

### 4.3 Why JWTs Use Base64URL: Grammar Restrictions
RFC 7230 Section 3.2.6 defines valid characters for HTTP header tokens:
```text
tchar = "!" / "#" / "$" / "%" / "&" / "'" / "*" / "+" / "-" / "." / "^" / "_" / "`" / "|" / "~" / DIGIT / ALPHA
excluded separators: " ( ) , / : ; < = > ? @ [ \ ] { }
```
- Because curly braces `{ }` and quotes `"` are illegal in HTTP header tokens, raw JSON cannot be placed directly inside an `Authorization` header.
- **Base64URL** maps data into safe printable characters ($A–Z, a–z, 0–9$), replacing `+` and `/` with `-` and `_` to remain safe for URL query strings, cookies, and HTTP headers without escaping.

---

### 4.4 Microservices for RFCs: From RFC 2616 to RFC 7230–7235 and RFC 9110–9114
In June 2014, the massive RFC 2616 specification was split into six modular documents:
- **RFC 7230**: Message Syntax and Routing
- **RFC 7231**: Semantics and Content
- **RFC 7232**: Conditional Requests
- **RFC 7233**: Range Requests
- **RFC 7234**: Caching
- **RFC 7235**: Authentication

In June 2022, the IETF updated this structure with RFCs 9110–9114:
- **RFC 9110 (HTTP Semantics)** & **RFC 9111 (HTTP Caching)**: Core protocol definitions shared across all HTTP versions.
- **RFC 9112**: HTTP/1.1 wire format
- **RFC 9113**: HTTP/2 wire format
- **RFC 9114**: HTTP/3 wire format

---

### 4.5 Status 418: The Permanent Easter Egg
Defined in RFC 2324 (April 1, 1998) as part of the Hyper Text Coffee Pot Control Protocol (HTCPCP):
- **`418 I'm a teapot`**
- In RFC 9110 Section 15.5.19, status 418 is formally marked as **"Unused"** but reserved permanently. Because widespread developer implementations had adopted the easter egg over two decades, the working group preserved the code to prevent conflicting assignments.

---

## 5. The Bill for HTTP/1.1: Three Structural Flaws

Despite its widespread success, HTTP/1.1 encountered three major architectural limits as web applications grew in complexity:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE THREE ARCHITECTURAL WALLS OF HTTP/1.1                              │
├──────────────────────────────────┬──────────────────────────────────┬──────────────────────────────────┤
│ 1. APPLICATION HEAD-OF-LINE      │ 2. UNCOMPRESSED HEADER BLOAT     │ 3. TRANSPORT HEAD-OF-LINE        │
│    BLOCKING (HOLB)               │                                  │    BLOCKING (TCP LEVEL)          │
├──────────────────────────────────┼──────────────────────────────────┼──────────────────────────────────┤
│ Pipelined requests must be       │ 97.5% of uploaded request headers│ Sockets multiplexed over TCP     │
│ answered in strict order. One    │ are byte-for-byte redundant      │ stall completely if a single     │
│ slow response stalls all subsequent│ cookies and user-agent strings,  │ packet drops in transit, waiting │
│ assets on the connection.        │ consuming upload bandwidth.      │ on TCP kernel retransmission.    │
└──────────────────────────────────┴──────────────────────────────────┴──────────────────────────────────┘
```

### 5.1 Application Head-of-Line (HoL) Blocking & Pipelining Failure
HTTP/1.1 introduced **Pipelining**, allowing clients to send multiple requests without waiting for intermediate responses.
- **The Problem**: A slow or heavy request (such as a database query taking 2 seconds) blocks all subsequent responses on that connection. Even if five trailing assets are ready in microseconds, the server cannot transmit them until the first response finishes.
- Due to widespread proxy bugs and head-of-line stalls, major web browsers disabled HTTP/1.1 pipelining by default.

---

### 5.2 The Root Cause: Order IS the Identifier
Why does HTTP/1.1 require strict in-order responses?
> **Because there is NO Request Identifier on the wire.**

- An HTTP/1.1 response header contains a status code, but does not specify which request it is answering. The client matches responses to requests purely by **counting the order in which they arrive**.
- Multiplexing requires explicit identity. TCP uses port numbers; SS7 uses transaction IDs; FastCGI uses `requestId`. Because HTTP/1.1 lacked stream identifiers, responses were bound to strict sequential order.

---

### 5.3 Header Bloat: 97.5% Byte-for-Byte Redundancy
A typical web page load issues roughly 80 asset requests:
- Headers average **805 bytes per request** (cookies, `User-Agent`, `Accept-*` headers).
- Total header data uploaded:
  $$80 \text{ requests} \times 805 \text{ bytes} = \mathbf{64,400 \text{ bytes}}$$
- Unique information (the URI path): **~1,600 bytes (2.5%)**.
- Redundant header repeats: **~62,800 bytes (97.5%)**.
- On mobile connections with limited upload bandwidth, uploading redundant headers introduced measurable latency before application data began transferring.

---

### 5.4 The Nagle + Delayed ACK Latency Penalty
When HTTP Keep-Alive was enabled, applications encountered a performance interaction between two lower-level algorithms:

```
NAGLE'S ALGORITHM (RFC 896):
"Do not send a small packet if an earlier packet remains unacknowledged."

DELAYED ACKNOWLEDGMENT (RFC 1122):
"Do not acknowledge a packet immediately; wait up to 40 ms to piggyback the ACK on data."
```

#### The Bug Pattern:
1. The server code issues two separate write calls: `write(header);` followed by `write(body);`.
2. The header packet is transmitted immediately.
3. Because the header is unacknowledged, **Nagle holds the body packet in the local buffer**.
4. The client has no response data to send, so **Delayed ACK waits 40 milliseconds**.
5. The connection stalls for 40 ms on every request until the delayed ACK timer expires.
- **The Fix**: Web servers coalesce header and body data into a single `write()` call and set the **`TCP_NODELAY`** socket option to disable Nagle's algorithm.

---

## 6. HTTP/2 (RFC 7540 / RFC 9113): Binary Framing & Multiplexing

### 6.1 Google SPDY & The Path to Standardization
In November 2009, Google engineers Mike Belshe and Roberto Peon introduced **SPDY**, an experimental protocol implemented in Chrome and Google services that demonstrated up to 55% reductions in page load times.
- In May 2015, the IETF standardized SPDY as **HTTP/2 (RFC 7540)** alongside **HPACK (RFC 7541)**.
- In June 2022, RFC 9113 superseded RFC 7540, deprecating unused features like cleartext upgrades and stream priority trees.
- Named **HTTP/2** (dropping the minor `.0` version) to clarify that future updates would use major version revisions.

---

### 6.2 The Connection Preface: 24 Bytes That Are Also a Joke
When initiating an HTTP/2 connection, the client transmits an exact 24-byte magic string:
```text
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```
```text
Hex: 50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a 0d 0a 53 4d 0d 0a 0d 0a
```
- If an older HTTP/1.1 server receives this sequence, it parses `PRI` as an unknown HTTP verb and returns **`501 Not Implemented`**, terminating the connection safely instead of attempting to parse binary frames as text.
- *(Historical Note: The characters `PRI...SM` referenced the PRISM surveillance revelations active during the 2013 drafting sessions).*

---

### 6.3 Protocol Negotiation: TLS ALPN vs. Cleartext `h2c`
- **Over TLS (The Standard Path)**: Uses **ALPN (Application-Layer Protocol Negotiation, RFC 7301)**. The client lists `h2` and `http/1.1` within the TLS `ClientHello`. The server selects `h2` in its `ServerHello`, establishing the protocol with **zero extra round trips**.
- **Cleartext `h2c`**: RFC 7540 defined an `Upgrade: h2c` header dance for unencrypted HTTP/2 over port 80. Browser vendors declined to support cleartext HTTP/2, requiring TLS encryption for all production implementations. RFC 9113 formally deprecated `h2c`.

---

### 6.4 The 9-Byte Binary Frame Header & Stream Identifiers
HTTP/2 replaces text strings with binary frames structured by a standardized 9-byte header:

```
┌───────────────────────────────┬───────────────┬───────────────┬───┬───────────────────────────────┐
│       Length (24 bits)        │   Type (8b)   │   Flags (8b)  │ R │    Stream Identifier (31b)    │
└───────────────────────────────┴───────────────┴───────────────┴───┴───────────────────────────────┘
 Payload: 0 to 16,384 bytes (Up to 16 MB if negotiated via SETTINGS_MAX_FRAME_SIZE)
```

- **Length (24 bits)**: Defines frame payload size up to 16,384 bytes by default.
- **Type (8 bits)**: Identifies frame purpose (`DATA`, `HEADERS`, `SETTINGS`, etc.).
- **Flags (8 bits)**: Frame-specific boolean flags (e.g., `END_STREAM = 0x01`, `END_HEADERS = 0x04`).
- **Stream Identifier (31 bits)**: **The explicit identity missing in HTTP/1.1.**
  - **Odd IDs ($1, 3, 5, \dots$)**: Initiated by the client.
  - **Even IDs ($2, 4, 6, \dots$)**: Initiated by the server (e.g., server push).
  - **Stream 0**: Reserved for connection-level control frames (`SETTINGS`, `PING`, `GOAWAY`).

---

### 6.5 The Ten Frame Types

| Type | Frame Name | Purpose / Responsibility | What it Replaced in HTTP/1.1 |
| :---: | :--- | :--- | :--- |
| **`0x00`** | **DATA** | Carries arbitrary body bytes. | `Content-Length` / chunked body |
| **`0x01`** | **HEADERS** | Opens a stream and carries compressed HPACK headers. | Request line and header block |
| **`0x02`** | **PRIORITY** | Stream dependency weights (deprecated by RFC 9113).| None |
| **`0x03`** | **RST_STREAM** | Immediately aborts a single stream without closing TCP.| Closing the TCP connection |
| **`0x04`** | **SETTINGS** | Advertises connection parameters in both directions. | None |
| **`0x05`** | **PUSH_PROMISE**| Announces resources pushed by the server. | None |
| **`0x06`** | **PING** | Measures round-trip time and tests connection health. | None |
| **`0x07`** | **GOAWAY** | Signals graceful connection shutdown, announcing last processed stream.| `Connection: close` |
| **`0x08`** | **WINDOW_UPDATE**| Enforces flow control per stream and per connection. | TCP-level flow control only |
| **`0x09`** | **CONTINUATION**| Extends header blocks that exceed a single frame. | None |

---

### 6.6 HPACK (RFC 7541): Static Tables, Dynamic Tables & Huffman Encoding
HPACK compresses headers using three coordinated mechanisms:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 HPACK COMPRESSION ARCHITECTURE                         │
├─────────────────────┬─────────────────────────────────┬────────────────────────────────┤
│ 1. STATIC TABLE     │ 2. DYNAMIC TABLE                │ 3. STATIC HUFFMAN ENCODING     │
├─────────────────────┼─────────────────────────────────┼────────────────────────────────┤
│ - 61 pre-defined,   │ - Connection-specific table.    │ - Fixed canonical Huffman code │
│   immutable entries.| - Stores newly observed headers │   built from web corpora.      │
│ - Index 2 represents│   from index 62 onward.         │ - Frequent characters are short│
│   ":method: GET".   │ - A 300-byte session cookie is  │   ("e" = 5 bits, "X" = 8 bits).│
│ - Transmitting GET  │   sent once; subsequent requests│ - Never adapts to data flowing │
│   costs a single    │   reference it using a 1-byte   │   through the connection!      │
│   byte: 0x82!       │   index.                        │                                │
└─────────────────────┴─────────────────────────────────┴────────────────────────────────┘
```

#### Performance Comparison:
- Standard HTTP/1.1 Request Headers: **174 bytes**
- HPACK First Transmission (cold): **79 bytes (55% reduction)**
- HPACK Subsequent Requests (warm dynamic table): **6 bytes (97% reduction)**

---

### 6.7 Why Not Gzip Headers? The CRIME Attack (2012)
Why did HTTP/2 introduce HPACK instead of using standard Gzip compression on headers?
- **The CRIME Attack (Compression Ratio Info-leak Made Easy)**:
  - Gzip uses DEFLATE (LZ77), an adaptive dictionary compressor that replaces repeated substrings with pointers.
  - If an attacker injects a parameter into a request (e.g., `/search?q=secret_token=a`), and the header also contains a secret cookie (`Cookie: secret_token=s3cr3t`), the compressed packet size shrinks whenever the attacker's guess matches the secret string.
  - By watching packet sizes over the wire, an attacker can extract private session tokens one character at a time.
- **HPACK's Defense**:
  - The Huffman table is **static and immutable**, preventing adaptive compression attacks.
  - The dynamic table requires exact name/value matches rather than substring matches.
  - Sensitive headers can be marked with the `never indexed` flag.

---

### 6.8 Multiplexing in Action: Out-of-Order Concurrent Streams
By breaking requests into independent frames stamped with 31-bit Stream IDs, HTTP/2 multiplexes multiple streams over a single TCP connection:
- If Stream 1 (`/big.txt`) takes 2 seconds to generate, the server immediately transmits frames for Stream 3 (`/style.css`, 4.9 ms) and Stream 5 (`/app.js`, 6.9 ms).
- Responses are delivered out of order as soon as they are ready, eliminating application-level Head-of-Line blocking.

```
HTTP/1.1 Pipelining (Strict Sequential Order):
Request 1 (/big.txt) ─── Delay 2000ms ────────────────────────> Response 1 (2000ms)
Request 2 (/style.css) ────────────────── Blocked behind Req 1 ─> Response 2 (2001ms)

HTTP/2 Multiplexing (Frames Interleaved by Stream ID):
Stream 1: [ DATA s=1 ] ─────────────────────────────────────────> [ DATA s=1 (2000ms) ]
Stream 3: [ DATA s=3 (4.9ms) ] ── Delivered Immediately!
Stream 5: [ DATA s=5 (6.9ms) ] ── Delivered Immediately!
```

---

### 6.9 The Thread-Safety Trap: Handling Streams Concurrently
A common error when building HTTP/2 servers is processing requests inline within the event read loop:
- Reading a request on Stream 1 and calling a synchronous handler blocks the loop, causing subsequent streams to wait and reproducing HTTP/1.1 latency profiles.
- **The Solution**: Dispatch incoming streams to asynchronous worker pools while maintaining a thread-safe lock around the shared TCP connection socket for frame writes.

---

### 6.10 The Death of Server Push & The Rise of 103 Early Hints
- **Server Push (`PUSH_PROMISE`)**: Allowed servers to proactively push resources (e.g., CSS stylesheets) to clients before the HTML was parsed.
- **Why It Failed**: The server does not know the state of the client's cache. Servers frequently pushed assets that the browser already had cached, wasting mobile bandwidth and competing for TCP congestion window space.
- Chrome removed Server Push support in 2022 (accounting for <1% of web traffic).
- **The Replacement: 103 Early Hints (RFC 8297)**:
  - While generating the main response, the server returns an early `103 Early Hints` status containing `Link: </style.css>; rel=preload`.
  - The browser checks its local cache and initiates requests only for missing assets.

---

## 7. The Fatal Flaw of HTTP/2 & The Rise of QUIC / HTTP/3

### 7.1 Transport-Layer (TCP) Head-of-Line Blocking
While HTTP/2 resolved application-level blocking, it exposed a lower-level bottleneck within TCP itself:

```
TCP BYTE STREAM (Packets In-Flight):
[ Packet 1 (Stream 1) ] [ Packet 2 (Stream 3) ] [ Packet 3 (Stream 5) ]
            │
      PACKET 1 DROPPED!
            │
[ TCP Kernel Receive Buffer ]: Holds Packets 2 and 3 in memory.
  The kernel CANNOT deliver Packets 2 and 3 to the browser because TCP 
  guarantees a strictly ordered byte stream!
  All streams stall until Packet 1 is retransmitted.
```

On connections with 2% packet loss (common on mobile and Wi-Fi networks), HTTP/2's single TCP connection can perform worse than multiple parallel HTTP/1.1 connections.

---

### 7.2 Why UDP? Middlebox Ossification & Protocol 253
Why not design a new Layer 4 transport protocol (like SCTP, IP protocol 132)?
- **Middlebox Ossification**: Internet routers, firewalls, and NAT devices are programmed to pass TCP (protocol 6) and UDP (protocol 17). Packets using unknown protocol numbers are dropped.
- **The Solution**: **QUIC runs over UDP**, using an existing protocol footprint that traverses firewalls and middleboxes.
- **Encryption by Default**: QUIC encrypts packet numbers and transport headers, preventing intermediate middleboxes from inspecting or altering protocol mechanics.

---

### 7.3 Handshake Consolidation: 1-RTT & 0-RTT Resumption

```
TCP + TLS 1.3 + HTTP/2 (3 RTTs Cold):
Client                                                  Server
  │─── 1. SYN ────────────────────────────────────────────>│
  │<── 2. SYN-ACK ─────────────────────────────────────────│ (1 RTT)
  │─── 3. ACK + TLS ClientHello ──────────────────────────>│
  │<── 4. TLS ServerHello + EncryptedExtensions ───────────│ (2 RTT)
  │─── 5. HTTP GET ───────────────────────────────────────>│
  │<── 6. HTTP 200 OK ─────────────────────────────────────│ (3 RTT)

QUIC + HTTP/3 (1 RTT Cold):
Client                                                  Server
  │─── 1. QUIC Initial (TLS ClientHello inside) ──────────>│
  │<── 2. QUIC Handshake (TLS ServerHello inside) ─────────│ (1 RTT: Connection & Crypto Ready!)
  │─── 3. HTTP/3 GET ─────────────────────────────────────>│
  │<── 4. HTTP/3 200 OK ───────────────────────────────────│ (2 RTT Total!)
```

On resumed connections, **QUIC 0-RTT** transmits application data inside the very first packet, eliminating handshake round trips entirely.

---

### 7.4 0-RTT Security Trade-offs: Replay Attacks & 425 Too Early
0-RTT data is vulnerable to **Replay Attacks**: an attacker intercepting an initial 0-RTT packet can replay it against the server.
- **Rule of Architecture**: 0-RTT must only be used for **idempotent requests** (`GET`, `HEAD`). It must never be used for state-changing operations like financial transactions.
- Servers can reject unsafe 0-RTT requests using the **`425 Too Early`** status code, instructing the client to retry after completing a full handshake.

---

### 7.5 True Stream Independence: Dropped Packets Isolated
In QUIC, each stream maintains its own independent sequence number and flow control:
- If a packet carrying data for Stream 1 is lost, **only Stream 1 pauses**.
- Packets for Stream 3 and Stream 5 continue to be delivered to the application immediately.

---

### 7.6 Connection Migration: 64-Bit Connection IDs Survive IP Changes
- TCP connections are identified by the 4-tuple (`Src IP, Src Port, Dst IP, Dst Port`). Switching from Wi-Fi to mobile data changes the client IP, terminating active TCP connections.
- QUIC identifies connections using a random **64-bit Connection ID (CID)**. When a device changes IP addresses, it continues communicating using the existing CID, avoiding connection resets.

---

### 7.7 QPACK (RFC 9204): Solving HPACK's Head-of-Line Blocking
Because HTTP/2 assumes in-order TCP delivery, HPACK relies on a single shared dynamic table.
In QUIC, out-of-order stream delivery would cause HPACK to stall if Stream 12 referenced a dynamic table entry sent on Stream 8 that had not yet arrived.

**QPACK resolves this** by moving dynamic table modifications to dedicated unidirectional control streams:
- **Encoder Stream**: Notifies the decoder of table insertions.
- **Decoder Stream**: Acknowledges observed table entries.
- The encoder only references dynamic entries that the decoder has explicitly confirmed, maintaining stream independence.

---

### 7.8 Protocol Discovery, Alt-Svc & The UDP Drop Fallback
Because browsers connect over TCP by default, servers advertise HTTP/3 support via the `Alt-Svc` response header:
```http
Alt-Svc: h3=":443"; ma=86400
```
- The browser caches this instruction and uses QUIC over UDP for subsequent requests.
- Modern clients also check **HTTPS DNS Resource Records (Type 65)** to discover HTTP/3 capabilities during DNS resolution.
- **Fallback**: If an enterprise firewall drops UDP traffic (affecting ~3–5% of networks), clients detect the timeout and fall back to TCP + TLS.

---

## 8. The State of Global Adoption in 2026

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│  METRIC (Web Almanac Crawl Data)     │  PERCENTAGE SHARE                    │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ HTTP/1.1 Traffic                     │ ~15% of global requests              │
│ HTTP/2 & HTTP/3 Traffic              │ ~85% of global requests              │
│ Websites Advertising HTTP/3          │ ~28% via Alt-Svc, ~10% via DNS RR    │
└──────────────────────────────────────┴──────────────────────────────────────┘
```
HTTP/1.1 remains active across legacy infrastructure, internal microservices, and embedded hardware. Production systems regularly interact with all three generations of the protocol.

---

## 9. The Four Core Architectural Lessons

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE FOUR CORE LESSONS                                  │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. MULTIPLEXING NEEDS      : Without a request ID, ordering becomes mandatory.         │
│    IDENTITY                  Adding a 31-bit stream ID enabled out-of-order streams.   │
│                                                                                        │
│ 2. EVERY FIX CREATES       : Pipelining introduced HoL blocking. HTTP/2 solved it,     │
│    THE NEXT PROBLEM          exposing TCP-level HoL blocking. QUIC fixed TCP, breaking │
│                              HPACK and requiring QPACK.                                │
│                                                                                        │
│ 3. SEMANTICS AND           : RFC 9110 defines HTTP semantics; RFCs 9112, 9113, and     │
│    ENCODING DIFFER           9114 define wire encodings. Business logic remains stable │
│                              across protocol generations.                              │
│                                                                                        │
│ 4. DEPLOYED REALITY WINS   : Browsers rewrote 302 to GET, leading to 307. Firewalls    │
│                              blocked custom transports, forcing QUIC to use UDP.       │
│                              Real-world network constraints dictate protocol designs.  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
Why did HTTP/1.0 suffer from high connection overhead when loading asset-heavy web pages?
- A) HTTP/1.0 packets were limited to 140 bytes.
- B) The connection was closed after each response, requiring a new 3-way TCP handshake and slow-start ramp-up for every asset.
- C) HTTP/1.0 used UDP instead of TCP.
- D) Web servers were limited to 1,024 requests per day.

**Answer: B**  
**Explanation:** RFC 1945 specified that HTTP/1.0 connections close after each response. Loading multiple assets required sequential TCP handshakes and repeated slow-start cycles.

---

### Question 2
What was the infrastructure consequence of HTTP/1.0 lacking a `Host` header?
- A) Web browsers could not render inline JPEG images.
- B) Servers could not host multiple domains on a single IP address, requiring a unique public IPv4 address for every website.
- C) Caching proxies could not store HTML responses.
- D) Clients could not issue POST requests.

**Answer: B**  
**Explanation:** Without a `Host` header, servers relied entirely on the destination IP address to determine which site was requested, preventing name-based virtual hosting.

---

### Question 3
Why are strong entity tags (`ETag`) required for HTTP Range requests, while weak entity tags (`W/"..."`) are insufficient?
- A) Weak ETags are generated using MD5.
- B) Weak ETags represent semantic equivalence rather than exact byte identity, which could result in corrupt spliced ranges if content changed.
- C) Weak ETags are only valid on UDP connections.
- D) Strong ETags are required by TLS 1.3.

**Answer: B**  
**Explanation:** Range requests reassemble byte chunks from a resource. Weak ETags indicate semantic similarity (e.g., whitespace adjustments) without guaranteeing identical byte offsets, making strong ETags necessary to avoid corruption.

---

### Question 4
In HTTP/1.1 chunked transfer encoding, how does the sender indicate that the entire body has been transmitted?
- A) By closing the underlying TCP socket.
- B) By transmitting an empty chunk with a length of zero (`0\r\n\r\n`).
- C) By sending an HTTP `416` status code.
- D) By transmitting an ASCII null byte (`\0`).

**Answer: B**  
**Explanation:** Chunked encoding formats data as size-prefixed blocks. An empty chunk of length zero indicates the end of the payload while keeping the connection open.

---

### Question 5
Why was status code `307 Temporary Redirect` introduced in RFC 2616 alongside the existing `302 Found` code?
- A) Status 302 was restricted to HTTPS connections.
- B) Browsers historically rewrote POST requests to GET on receiving a 302; status 307 explicitly requires preserving the original HTTP method.
- C) Status 302 lacked caching support.
- D) Status 307 permits caching on proxy servers.

**Answer: B**  
**Explanation:** While 302 was originally intended to preserve methods, early browsers altered POST requests to GET. RFC 2616 documented this reality for 302 and introduced 307 to guarantee method preservation.

---

### Question 6
What is the underlying mechanism of an HTTP Request Smuggling attack?
- A) Injecting SQL statements into user-agent headers.
- B) Exploiting discrepancies in how front-end proxies and back-end servers prioritize conflicting `Content-Length` and `Transfer-Encoding: chunked` headers.
- C) Intercepting unencrypted Wi-Fi frames in promiscuous mode.
- D) Overflowing the TCP sliding window buffer.

**Answer: B**  
**Explanation:** When a proxy and backend disagree on whether to prioritize `Content-Length` or `Transfer-Encoding`, the backend may interpret part of an initial request body as a separate, unauthenticated request.

---

### Question 7
Why did HTTP/1.1 pipelining fail to achieve widespread production adoption?
- A) It was incompatible with HTML tables.
- B) HTTP/1.1 lacked request identifiers, requiring responses to return in strict sequential order and causing head-of-line blocking if a request was slow.
- C) It was limited to two assets per domain.
- D) It was unsupported on Linux operating systems.

**Answer: B**  
**Explanation:** Without stream identifiers, responses had to return in the order requested. A single slow backend task held up all subsequent responses in the pipeline.

---

### Question 8
What causes artificial 40 ms delays on persistent HTTP/1.1 connections when writing headers and bodies separately?
- A) The operating system kernel enforces a mandatory pause between HTTP requests.
- B) The interaction between Nagle's algorithm (holding small unacknowledged packets) and Delayed ACKs (waiting up to 40 ms before returning an acknowledgment).
- C) DNS resolver lookup timeouts.
- D) TCP SYN cookie verification delays.

**Answer: B**  
**Explanation:** If an application writes headers and bodies in separate calls, Nagle's algorithm buffers the body until the header is acknowledged, while the client's Delayed ACK waits up to 40 ms, creating artificial latency. Setting `TCP_NODELAY` avoids this delay.

---

### Question 9
What is the purpose of the 24-byte client connection preface (`PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`) in HTTP/2?
- A) To synchronize client and server system clocks.
- B) To cause legacy HTTP/1.1 servers to reject the connection with a 501 error rather than misinterpreting binary frames as text.
- C) To transmit initial HPACK compression dictionary entries.
- D) To configure TCP receive window sizes.

**Answer: B**  
**Explanation:** The preface appears as an unknown method `PRI` to older HTTP/1.1 servers, allowing them to fail safely with `501 Not Implemented` instead of corrupting binary data.

---

### Question 10
How does HPACK prevent the CRIME compression attack that affected SPDY?
- A) By disabling TLS encryption on HTTP headers.
- B) By using a fixed static Huffman code and an exact-match dynamic table, preventing attackers from deducing secrets from adaptive compression ratios.
- C) By restricting header lengths to 64 bytes.
- D) By encrypting headers using AES-256.

**Answer: B**  
**Explanation:** The CRIME exploit used DEFLATE's adaptive dictionary to guess secret cookies character by character based on output sizes. HPACK eliminates this vector by using non-adaptive Huffman encoding and exact-match index tables.

---

### Question 11
What is the purpose of the 31-bit Stream Identifier in the HTTP/2 frame header?
- A) It specifies the exact byte offset of the file on disk.
- B) It identifies the logical stream each frame belongs to, enabling out-of-order multiplexing over a single TCP connection.
- C) It tracks total server memory consumption.
- D) It authenticates the client's TLS certificate.

**Answer: B**  
**Explanation:** The Stream Identifier tags individual frames, allowing servers to interleave responses from multiple requests over a single connection without strict sequential ordering.

---

### Question 12
Why was HTTP/2 Server Push largely abandoned by web browsers?
- A) It was incompatible with JavaScript files.
- B) Servers could not reliably determine client cache state, leading to redundant pushes that consumed bandwidth and congested connections.
- C) It increased server CPU consumption by 800%.
- D) It was deprecated in RFC 7540.

**Answer: B**  
**Explanation:** Because servers cannot inspect client caches, they often pushed resources the browser had already cached, wasting bandwidth. The feature was superseded by `103 Early Hints`.

---

### Question 13
What is Transport-Layer Head-of-Line Blocking in HTTP/2?
- A) When a database query blocks an NGINX worker thread.
- B) When a dropped TCP packet causes the operating system kernel to pause delivery of all subsequent multiplexed streams until the missing segment is retransmitted.
- C) When a browser exceeds its six-connection limit.
- D) When an HTTP/2 frame exceeds 16,384 bytes.

**Answer: B**  
**Explanation:** TCP enforces an unbroken, ordered byte stream. If a packet is lost, the kernel buffers all subsequent packets until the missing data is retransmitted, blocking all concurrent HTTP/2 streams on that connection.

---

### Question 14
Why is QUIC built on top of UDP rather than defining a new transport protocol number at Layer 4?
- A) UDP provides built-in encryption.
- B) Intermediate network middleboxes and firewalls routinely drop packets using unrecognized Layer 4 protocol numbers, making new transport protocols unusable across the public internet.
- C) UDP is faster than fiber optic transmission.
- D) Operating systems lack support for TCP.

**Answer: B**  
**Explanation:** Network ossification causes intermediate routers and firewalls to drop traffic that is not TCP or UDP. Building QUIC over UDP allows it to traverse existing internet infrastructure without modification.

---

### Question 15
How does QUIC maintain active connections when a client transitions between networks (e.g., from Wi-Fi to cellular)?
- A) By re-running the TCP 3-way handshake on the new IP address.
- B) By identifying connections using an independent 64-bit Connection ID (CID) rather than the IP/Port 4-tuple.
- C) By keeping the Wi-Fi radio active in the background.
- D) By routing traffic through an SS7 signaling gateway.

**Answer: B**  
**Explanation:** TCP connections are tied to the 4-tuple (`Src IP, Src Port, Dst IP, Dst Port`), terminating if an IP changes. QUIC decouples connection identity from addressing by using a persistent Connection ID that survives network changes.

---

## 11. Session 5 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **HTTP/1.0 Lifetime** | Connection terminates on response EOF. Multiple assets incur repeated TCP handshakes and slow-start penalties. |
| **Missing `Host` Header**| HTTP/1.0 tied sites directly to IP addresses. HTTP/1.1 made `Host` mandatory, enabling name-based virtual hosting. |
| **Timestamps vs. ETags** | Timestamps lack sub-second resolution and drift. ETags provide cryptographic content fingerprints. |
| **Chunked Transfer** | Transmits dynamic bodies using size-prefixed hex chunks terminated by an empty chunk (`0\r\n\r\n`). |
| **Request Smuggling** | Discrepancies between `Content-Length` and `Transfer-Encoding` handling allow attackers to smuggle hidden requests. |
| **Pipelining Failure** | Lacked stream identifiers; responses had to return in strict sequential order, causing head-of-line stalls. |
| **HTTP/2 Preface** | 24-byte magic string (`PRI * HTTP/2.0...`) ensures legacy servers reject binary connections with status 501. |
| **Frame Header** | 9-byte header: Length (24b), Type (8b), Flags (8b), Reserved (1b), Stream ID (31b). |
| **HPACK Compression** | Uses static tables, dynamic tables, and fixed Huffman codes. Avoids CRIME-style compression attacks. |
| **Early Hints (103)** | Replaced Server Push. Notifies clients to preload resources while the main response is generated. |
| **TCP HoL Blocking** | In HTTP/2, a single lost TCP packet stalls all multiplexed streams in the operating system receive buffer. |
| **QUIC Foundations** | Runs over UDP to bypass firewall ossification. Encrypts transport headers and operates in user space. |
| **1-RTT / 0-RTT Handshake**| Combines transport and TLS 1.3 handshakes into a single round trip (or zero on resumption). |
| **Connection Migration** | Uses 64-bit Connection IDs (CIDs) rather than the 4-tuple, allowing connections to survive network handoffs. |
| **QPACK vs. HPACK** | Moves dynamic table updates to separate control streams to prevent cross-stream head-of-line blocking. |
