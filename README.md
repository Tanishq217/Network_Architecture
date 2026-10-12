# Computer Networks: Architecture, Systems & Protocols

> **Repository Overview**: Comprehensive, detailed theoretical notes and examination guides for the 8-week Computer Networks master curriculum (`cn-at-scaler`).
> 
> Each chapter breaks down complex, low-level networking topics into beginner-friendly, rigorous explanations covering operating system internals, socket programming, kernel queues, signals, network protocols, server architectures, distributed systems, and real-time streaming media. Every chapter concludes with high-yield multiple-choice questions (MCQs) for exam preparation.

---

## 📚 Curriculum Roadmap

| Week | Chapter | Focus & Key Concepts | Status |
| :---: | :--- | :--- | :---: |
| **01** | [**Chapter 01: Network Programming 101**](./Chapter_01_Network_Programming_101/Chapter_01_Network_Programming_101.md) | The 7 system calls, Ports & root privilege, Accept queues, SIGPIPE, Ephemeral ports, Endianness (`htons`), Concurrency (`fork`, `select`, `epoll`), TLS/SSL, Protocol framing (TLV, BCD), RPC/gRPC, and 15 MCQs. | ✅ Complete |
| **02** | [**Chapter 02: Computer Networking Eagle Eye View**](./Chapter_02_Computer_Networking_Eagle_Eye_View/Chapter_02_Computer_Networking_Eagle_Eye_View.md) | The 4 Invariants, OSI layers & Starlink LEO, CSMA/CD vs CSMA/CA, SS7 Telecom stack vs TCP/IP, The 140-byte SMS limit & GSM-7 160-char packing trick, Cellular signaling (MAP, ISUP, 5G SIP/RTP), BBR vs Loss-based congestion control, TCP 3-way handshake & random ISN, SYN Cookies & Mitnick attack, Sockets in `CLOSE_WAIT`, QUIC/HTTP/3 vs TCP HoL blocking, Text protocols (SMTP, POP3, IMAP, FTP Active vs Passive), Big-Endian 227 port decoding, and 15 MCQs. | ✅ Complete |
| **03** | [**Chapter 03: Server Side Considerations for Scale**](./Chapter_03_Server_Side_Considerations_for_Scale/Chapter_03_Server_Side_Considerations_for_Scale.md) | How servers hold 10,000+ connections; Concurrency (Fork vs Thread vs Loop vs Hybrid); `select()` mechanics & $O(1)$ array deletion; The 1024 wall (`FD_SETSIZE`, `ulimit -n`, C10K); `epoll()` $O(\text{ready})$ Red-Black tree & Ready List; `SO_REUSEPORT` kernel load balancing; Timeouts & Deadlines; CGI process-per-request & three-pipe plumbing; HTTP headers to environment variables; Fall of in-process modules (`mod_php`); FastCGI binary records & empty record delimiters; Java Servlets pre-threading; Stateless vs Stateful services; The Four Walls (Descriptors, Memory, Context switches, Queuing); OS 8MB threads vs Go 2KB Goroutines; Zero-Copy `sendfile()`, `io_uring`, Kernel Bypass (DPDK); Little's Law ($L = \lambda \times W$); Real-world architectures (Apache MPM Prefork/Worker/Event, HAProxy, Varnish, NGINX), and 15 MCQs. | ✅ Complete |
| **04** | [**Chapter 04: NGINX Deep Dive**](./Chapter_04_nginx_Deep_Dive/Chapter_04_nginx_Deep_Dive.md) | Historical lineage (CERN httpd, NCSA HTTPd/Mosaic, Apache); Igor Sysoev at Rambler & deleting the worker; Master-Worker architecture; The 6-line event loop; Decentralized worker balancing (`ngx_accept_disabled`); Thundering herd, `accept_mutex`, `EPOLLEXCLUSIVE`, `SO_REUSEPORT`; Stale event bug & pointer bit tagging; Zero-Copy `sendfile()` & kTLS; Forward vs Reverse proxying & absorbing slow 2G clients; Shared memory cache (`proxy_cache_path`, `inactive` vs TTL, LRU eviction); Anti-stampede (`proxy_cache_lock`) & `proxy_cache_use_stale`; Location routing precedence (`=`, `^~`, `~`, `/`); Ternary Search Tree vs Regex array; Why NGINX has no `.htaccess`; Smooth Weighted Round Robin; `try_files` fallbacks; Memory pools (`ngx_palloc`) without `free()`; Intrusive lists (`ngx_queue`); 27-state resumable HTTP parser; 11 HTTP processing phases, and 15 MCQs. | ✅ Complete |
| **05** | [**Chapter 05: Evolution of HTTP**](./Chapter_05_Evolution_of_HTTP/Chapter_05_Evolution_of_HTTP.md) | HTTP/1.0 (RFC 1945, ephemeral sockets, missing Host, timestamp caching); Early HTTP/1.1 (RFC 2068, persistent default, mandatory Host, ETags, Vary, Range 206, Chunked transfer, 2-connection limit, domain sharding); Modern HTTP/1.1 (RFC 2616/7230/9110, 307 fix, Request Smuggling, Postel's law reversal, Base64URL in JWTs, Status 418); The Bill for 1.1 (HoL blocking, Order IS identity, 97.5% header repeats, Nagle + Delayed ACK bug); HTTP/2 (SPDY origin, 24-byte preface, ALPN vs h2c, 9-byte frame header, 31-bit stream IDs, 10 frame types, HPACK tables vs CRIME, multiplexing, Server Push failure & 103 Early Hints); QUIC & HTTP/3 (TCP HoL blocking, UDP foundation, Middlebox ossification, 1-RTT & 0-RTT replay security / 425 Too Early, true stream independence, 64-bit Connection IDs, QPACK encoder/decoder streams, Alt-Svc / DNS RR 65), and 15 MCQs. | ✅ Complete |
| **06** | [**Chapter 06: (Ab)using CDNs**](./Chapter_06_Abusing_CDNs/Chapter_06_Abusing_CDNs.md) | Speed of light in fiber (4.9 ms/1000 km, Mumbai-Ashburn floor); The Mathis Equation ($Throughput \approx MSS / (RTT \times \sqrt{p})$); BGP Anycast routing & IATA airport PoPs (`colo`, `cf-ray`); Consistent Hashing (Karger 1997) vs Modulo hashing; CDN software stacks (Akamai, CloudFront, Fastly, Pingora); Default file extension rules & cache keys; 8-state `cf-cache-status` machine; Dynamic content caching (SSI $\rightarrow$ ESI $\rightarrow$ Ajax $\rightarrow$ Railgun / RFC 9842 Dictionary Transport); Shared cache exploits (Web Cache Poisoning & Web Cache Deception); Edge BBR & Pingora shared connection pools (87.1% $\rightarrow$ 99.92% reuse); Tiered caching & Request collapsing; Edge security perimeter (WAF normalization, Sliding-window rate limiting, 31.4 Tbps DDoS absorption via Anycast); Origin lockdown (mTLS & Cloudflare Tunnels); Edge Compute via V8 Isolates vs Containers, and 15 MCQs. | ✅ Complete |
| **07** | [**Chapter 07: Building for Failures**](./Chapter_07_Building_for_Failures/Chapter_07_Building_for_Failures.md) | The 6 request failure stages (DNS $\rightarrow$ TCP $\rightarrow$ TLS $\rightarrow$ Write $\rightarrow$ Server $\rightarrow$ Read); DNS timeout vs Read timeout asymmetry; Strict definition of Retry vs Handling vs Failover; The 4 retry classification gates; HTTP status retryability; Idempotency semantics (RFC 9110 §9.2) & Idempotency Keys; The naive retry loop trap; Exponential backoff with AWS Full Jitter; `Retry-After` header pacing; The Circuit Breaker pattern (Closed, Open, Half-Open); Video streaming player case study (96 failure modes); The live segment 404 exception; Fallbacks over retries (DRM tiers, codec substitution, ad dropping, iOS `ResetMedia`); Fastly 49-min outage & `stale-if-error` (RFC 5861); The Second Outage (Thundering herd on recovery); Failover, Feature Flags, Graceful Degradation, and 15 MCQs. | ✅ Complete |
| **08** | [**Chapter 08: Streaming Video at Large Scale**](./Chapter_08_Streaming_Video_at_Large_Scale/Chapter_08_Streaming_Video_at_Large_Scale.md) | Pull vs Push architectures for 50M concurrent viewers; Long Polling, SSE, WebSockets (101 Upgrade, masking), and MQTT PubSub; Hotstar 50M socket case study; The cost of held sockets; Little's Law twice (50k vs 50M sockets); 4 deployment cost models (S3, NGINX+ALB, CloudFront, EC2 Push); S3 prefix limit (5,500 GET/s); The 11 KB request fee vs byte fee break-even line; Stacked TTL staleness; Request collapsing & `proxy_cache_use_stale updating;`; Negative caching; JSON Patch diffs; HLS architecture (Master vs Media playlists); MPEG-2 TS (188-byte packets, 13-bit PID) vs fMP4/CMAF; Segment length trade-offs; HLS latency budget (20s vs LL-HLS ~3s); The Spoiler Problem & `#EXT-X-PROGRAM-DATE-TIME` sync; Throughput-based vs Buffer-based ABR; Multi-CDN steering; Stale playlist / deleted segment 404 trap; Live stream caching matrix; FFmpeg ABR encoding & GOP keyframe alignment (`-g 120`), and 15 MCQs. | ✅ Complete |

---

## 🎯 Exam Practice Quizzes & Calculations

* **[Practice Quiz Bank 01 (Lectures 01 to 03)](./Practice_Quizzes/Quiz_01_Lectures_01_to_03.md)**: 20 high-yield, calculation-heavy practice questions modeled directly after real classroom exam quizzes. Features detailed step-by-step arithmetic (default TCP MSS 536 bytes, bit shifts, little-endian port byte swaps, Base64 calculations, SMS GSM-7 bit packing, Little's Law concurrency, fiber optic latency floors, and NGINX worker throttling formulas).

---

## 📂 Repository Directory Structure

```text
Network_Architecture/
├── README.md                                                        # Master syllabus and navigation
├── Practice_Quizzes/
│   └── Quiz_01_Lectures_01_to_03.md                                # 20 calculation-heavy exam MCQs
├── Chapter_01_Network_Programming_101/
│   └── Chapter_01_Network_Programming_101.md                       # In-depth notes, diagrams, and MCQs
├── Chapter_02_Computer_Networking_Eagle_Eye_View/
│   └── Chapter_02_Computer_Networking_Eagle_Eye_View.md            # In-depth notes, diagrams, and MCQs
├── Chapter_03_Server_Side_Considerations_for_Scale/
│   └── Chapter_03_Server_Side_Considerations_for_Scale.md          # In-depth notes, diagrams, and MCQs
├── Chapter_04_nginx_Deep_Dive/
│   └── Chapter_04_nginx_Deep_Dive.md                               # In-depth notes, diagrams, and MCQs
├── Chapter_05_Evolution_of_HTTP/
│   └── Chapter_05_Evolution_of_HTTP.md                             # In-depth notes, diagrams, and MCQs
├── Chapter_06_Abusing_CDNs/
│   └── Chapter_06_Abusing_CDNs.md                                  # In-depth notes, diagrams, and MCQs
├── Chapter_07_Building_for_Failures/
│   └── Chapter_07_Building_for_Failures.md                         # In-depth notes, diagrams, and MCQs
└── Chapter_08_Streaming_Video_at_Large_Scale/
    └── Chapter_08_Streaming_Video_at_Large_Scale.md                # In-depth notes, diagrams, and MCQs
```

---

## 🎯 How to Use These Notes

1. **Read Top-Down**: Each chapter is organized logically from foundational concepts to advanced OS kernel mechanics, telecommunications protocols, server scaling, edge architectures, resilient distributed systems, and petabyte-scale video distribution.
2. **Review the Real-World Case Studies**: Pay close attention to practical networking problems (e.g., bufferbloat, why an emoji halves SMS length, `CLOSE_WAIT` leaks, the 1024 file descriptor wall, Little's Law memory bugs, zero-copy `sendfile()`, NGINX location routing precedence, HTTP request smuggling, HPACK vs CRIME, QUIC connection migration, consistent hashing, cache poisoning, Anycast DDoS defense, circuit breaker recovery spikes, and HLS live spoiler synchronization).
3. **Practice the MCQs & Calculation Workouts**: Test your speed and precision using the chapter MCQs and the dedicated [Practice Quiz Bank](./Practice_Quizzes/Quiz_01_Lectures_01_to_03.md).
