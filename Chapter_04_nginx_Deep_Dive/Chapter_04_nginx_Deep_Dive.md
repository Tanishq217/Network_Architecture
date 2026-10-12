# Chapter 04: NGINX Deep Dive — Architecture, Internals & The Code That Explains It

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 4 / Week 4  
> **Core Theme**: *"A trimmed history · what NGINX actually changed · the features you configure · and one long walk through the source."*  
> **Key Protocols & Concepts**: Web server historical lineage (CERN httpd, NCSA HTTPd, Apache, IIS), The C10K challenge, Igor Sysoev's architecture shift (deleting the worker), Master-Worker multi-process model, The 6-line core event loop (`ngx_process_events_and_timers`), Decentralized worker load balancing (`ngx_accept_disabled`), The Thundering Herd & `accept_mutex` to `EPOLLEXCLUSIVE`/`SO_REUSEPORT`, The Stale Event Bug (Pointer bit tagging), Zero-Copy `sendfile()` vs. user-space copies & kTLS, Forward vs. Reverse Proxying, Shared memory caching (`proxy_cache_path`, LRU queues, `proxy_cache_lock`, `proxy_cache_use_stale`), Location routing precedence (Ternary Search Tree vs. Regex array), Smooth Weighted Round Robin algorithm, `try_files` fallback mechanics, NGINX memory pools (`ngx_palloc`), Intrusive lists (`ngx_queue`), The 27-state resumable HTTP parser, The 11 HTTP request processing phases, and 15 exam-grade MCQs.

---

## Table of Contents
1. [The Real Historical Lineage of Web Servers](#1-the-real-historical-lineage-of-web-servers)
   - [1.1 Common Myths vs. Historical Reality](#11-common-myths-vs-historical-reality)
   - [1.2 How Apache Spent a Decade Trying to Outrun `fork()`](#12-how-apache-spent-a-decade-trying-to-outrun-fork)
   - [1.3 Java's Answer: Pre-Allocation and Servlets](#13-javas-answer-pre-allocation-and-servlets)
   - [1.4 The 2002 Bottleneck: Dan Kegel's C10K Problem](#14-the-2002-bottleneck-dan-kegels-c10k-problem)
2. [What NGINX Changed: Deleting the Per-Connection Worker](#2-what-nginx-changed-deleting-the-per-connection-worker)
   - [2.1 Igor Sysoev and the Rambler Portal (2002–2004)](#21-igor-sysoev-and-the-rambler-portal-20022004)
   - [2.2 The Fundamental Shift: Sockets are State, Not Threads](#22-the-fundamental-shift-sockets-are-state-not-threads)
   - [2.3 The Master-Worker Architecture](#23-the-master-worker-architecture)
   - [2.4 All of NGINX in Six Lines of C](#24-all-of-nginx-in-six-lines-of-c)
3. [Worker Synchronization & Kernel Mechanics](#3-worker-synchronization--kernel-mechanics)
   - [3.1 Decentralized Worker Load Balancing in One Arithmetic Line](#31-decentralized-worker-load-balancing-in-one-arithmetic-line)
   - [3.2 The Thundering Herd Problem: From `accept_mutex` to `SO_REUSEPORT`](#32-the-thundering-herd-problem-from-accept_mutex-to-so_reuseport)
   - [3.3 `select` is $O(\text{watched})$, `epoll` is $O(\text{active})$](#33-select-is-owatched-epoll-is-oactive)
   - [3.4 The Stale Event Bug: Pointer Tagging and Generation Bits](#34-the-stale-event-bug-pointer-tagging-and-generation-bits)
4. [Zero-Copy Data Transfer: `sendfile()` & kTLS](#4-zero-copy-data-transfer-sendfile--ktls)
   - [4.1 Four Memory Copies Become Two](#41-four-memory-copies-become-two)
   - [4.2 The Conflict: Why `gzip on;` Breaks `sendfile on;`](#42-the-conflict-why-gzip-on-breaks-sendfile-on)
   - [4.3 Kernel-Space TLS (kTLS)](#43-kernel-space-tls-ktls)
   - [4.4 The Loopback Benchmark Reality](#44-the-loopback-benchmark-reality)
5. [Reverse Proxying: The Architecture of the Edge](#5-reverse-proxying-the-architecture-of-the-edge)
   - [5.1 Forward Proxy vs. Reverse Proxy](#51-forward-proxy-vs-reverse-proxy)
   - [5.2 Absorbing Slow Clients (2G Buffering)](#52-absorbing-slow-clients-2g-buffering)
6. [Shared-Memory Caching Architecture](#6-shared-memory-caching-architecture)
   - [6.1 Cache Storage: Shared Memory Index + Disk Subdirectories](#61-cache-storage-shared-memory-index--disk-subdirectories)
   - [6.2 `inactive` vs. `proxy_cache_valid` (TTL)](#62-inactive-vs-proxy_cache_valid-ttl)
   - [6.3 Intrusive Doubly-Linked Lists & The Single-Line LRU Eviction](#63-intrusive-doubly-linked-lists--the-single-line-lru-eviction)
   - [6.4 Two Directives That Save You at 3 AM: Cache Locks & Stale Fallbacks](#64-two-directives-that-save-you-at-3-am-cache-locks--stale-fallbacks)
7. [Routing Precedence & Data Structures](#7-routing-precedence--data-structures)
   - [7.1 The Four Location Directives and Evaluation Precedence](#71-the-four-location-directives-and-evaluation-precedence)
   - [7.2 Why 500 Prefixes are Free and 500 Regexes are Not](#72-why-500-prefixes-are-free-and-500-regexes-are-not)
   - [7.3 Why NGINX Has No `.htaccess`](#73-why-nginx-has-no-htaccess)
8. [Upstream Load Balancing & Fallbacks](#8-upstream-load-balancing--fallbacks)
   - [8.1 Smooth Weighted Round Robin Algorithm](#81-smooth-weighted-round-robin-algorithm)
   - [8.2 `try_files` and Pretty URLs](#82-try_files-and-pretty-urls)
9. [A Walkthrough of the NGINX Source Code](#9-a-walkthrough-of-the-nginx-source-code)
   - [9.1 Source Tree Layout](#91-source-tree-layout)
   - [9.2 Memory Management: `ngx_palloc` (Pools without `free()`)](#92-memory-management-ngx_palloc-pools-without-free)
   - [9.3 Intrusive Doubly-Linked Lists: `ngx_queue.h`](#93-intrusive-doubly-linked-lists-ngx_queueh)
   - [9.4 The 27-State Resumable HTTP Parser](#94-the-27-state-resumable-http-parser)
   - [9.5 The Eleven HTTP Request Processing Phases](#95-the-eleven-http-request-processing-phases)
10. [The Seven Steps of Server Evolution](#10-the-seven-steps-of-server-evolution)
11. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#11-comprehensive-review--practice-mcqs-with-detailed-explanations)
12. [Session 4 Summary Cheat Sheet](#12-session-4-summary-cheat-sheet)

---

## 1. The Real Historical Lineage of Web Servers

To understand why NGINX was built, we must first understand the software lineage that preceded it.

```
THE TRUE HISTORICAL TIMELINE:
1990: CERN httpd (Tim Berners-Lee on a NeXT cube) ─── The First Web Server
        │
1993: NCSA HTTPd (Rob McCool) ──────────────────────── Invented CGI (Common Gateway Interface)
        │ (1994: McCool departs; webmasters exchange patches by email)
1995: Apache HTTP Server ───────────────────────────── "A Patchy Server" (Brian Behlendorf et al.)
        │
1996: Apache becomes #1 Web Server ─────────────────── Dominates global hosting for 20 years
        │
1999: Dan Kegel articulates the C10K Problem ───────── Hardware outpaces thread/process models
        │
2002: Igor Sysoev starts NGINX at Rambler ──────────── Solves C10K by deleting the worker
        │
2004: NGINX released on October 4 (Sputnik Day) ────── Master-Worker event-driven reverse proxy
```

### 1.1 Common Myths vs. Historical Reality
A frequent misconception in computer science history claims:
> *"Tim Berners-Lee wrote the first web server called Mosaic, and Mosaic evolved into Apache."*

**The Historical Reality**:
1. **CERN httpd (1990)**: Written by Tim Berners-Lee on a NeXT Computer at CERN. This was the world's first HTTP web server.
2. **NCSA Mosaic (1993)**: Written by Marc Andreessen and Eric Bina at the National Center for Supercomputing Applications (NCSA). **Mosaic was a web browser**, not a server. It popularized the World Wide Web by introducing inline images.
3. **NCSA HTTPd (1993)**: Written by Rob McCool at NCSA. This was a separate server program that introduced the **Common Gateway Interface (CGI)**.
4. **Apache HTTP Server (1995)**: When Rob McCool left NCSA in 1994, development stalled. Webmasters around the world traded bug fixes and feature patches via mailing lists. A group of developers compiled these into a coherent distribution dubbed "A Patchy Server" (**Apache**). By 1996, Apache surpassed NCSA HTTPd to become the #1 web server on earth.

---

### 1.2 How Apache Spent a Decade Trying to Outrun `fork()`
Apache's core scaling challenge was that its original design bound one operating system execution context to each active client:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        APACHE MPM (MULTI-PROCESSING MODULE) EVOLUTION                  │
├─────────────────────┬─────────────────────────────────┬────────────────────────────────┤
│ 1. PREFORK MPM      │ 2. WORKER MPM                   │ 3. EVENT MPM                   │
│ (Apache 1.3, 1995)  │ (Apache 2.0, 2002)              │ (Apache 2.4, 2012)             │
├─────────────────────┼─────────────────────────────────┼────────────────────────────────┤
│ - Pre-forks pool of │ - Multi-process, multi-threaded.│ - Introduces a dedicated       │
│   isolated processes│ - Multiple pthreads per process.│   Listener Thread per process. │
│ - 1 process parks on│ - Lower memory than prefork, but│ - Parks idle Keep-Alive sockets│
│   1 connection.     │   1 thread remains pinned to an │   on epoll.                    │
│ - Perfect isolation;│   idle HTTP Keep-Alive client.  │ - Hands worker threads only    │
│   massive RAM cost. │ - Thread safety bugs in modules.│   connections with active data.│
└─────────────────────┴─────────────────────────────────┴────────────────────────────────┘
```

- **Apache 1.3 (Prefork MPM)**: Moved the expensive `fork()` off the request path by pre-forking worker processes at boot. However, each process still handled only one connection at a time.
- **Apache 2.0 (Worker MPM)**: Replaced processes with POSIX threads (`pthreads`). While threads reduced memory overhead, **an entire OS thread remained blocked waiting on idle clients** during HTTP/1.1 Keep-Alive sessions.
- **Apache 2.4 (Event MPM)**: In 2012, Apache adopted an event loop architecture where a dedicated listener thread monitored idle connections via `epoll`, passing sockets to worker threads only when active requests arrived.

---

### 1.3 Java's Answer: Pre-Allocation and Servlets
In 1997, Sun Microsystems introduced **Java Servlets**, followed by the **J2EE** enterprise specification in 1999:
```text
// CGI (1993): Process destroyed per hit
per request: fork() + exec() + load interpreter + parse script + execute + exit

// Java Servlet (1997): Object outlives request
at startup : Load class byte-code, instantiate Servlet object once in memory
per request: servlet.service(request, response)  <── In-memory method call on thread pool
```
- **The Core Pattern**: Servlets, FastCGI, and pre-forking all applied the same optimization: **amortize initialization costs by paying them once at startup**, keeping long-lived execution contexts in memory.

---

### 1.4 The 2002 Bottleneck: Dan Kegel's C10K Problem
In 1999, engineer Dan Kegel published a paper outlining the **C10K Problem**: the challenge of designing a web server capable of handling **10,000 concurrent connections** on a single machine.

By 2002, hardware had become fast enough to handle 10,000 clients, but server software hit three structural walls:
1. **The Descriptor Wall**: Sockets were managed using `select()`, which was capped at **1,024 descriptors** by the compile-time constant `FD_SETSIZE`.
2. **The Memory Wall**: Allocating 10,000 OS threads (each with an 8 MB default stack) required **80 GB of address space** before any data was read.
3. **The Scheduler Wall**: Managing thousands of threads caused excessive CPU context switching in the operating system scheduler.

While existing projects focused on making workers cheaper, a different approach emerged: **eliminate the per-connection worker entirely**.

---

## 2. What NGINX Changed: Deleting the Per-Connection Worker

### 2.1 Igor Sysoev and the Rambler Portal (2002–2004)
In 2002, Russian systems administrator **Igor Sysoev** worked at **Rambler**, one of Russia's largest search and web portals. Rambler's Apache clusters were crashing under high connection volumes. The servers were not overwhelmed by request processing; they were running out of resources because thousands of idle client connections were holding onto Apache workers.

Sysoev began writing **NGINX** in 2002 to solve the C10K problem directly:
- **First Public Release**: October 4, 2004 (commemorating the launch of Sputnik).
- **Production Maturation**: Version 1.0.0 was released in April 2011 after seven years of large-scale production use.
- **Design Objective**: Sysoev did not set out to write an application server. He designed an asynchronous **reverse proxy** that could maintain tens of thousands of mostly-idle connections in a single process, offloading dynamic request execution to separate backend runtimes.

---

### 2.2 The Fundamental Shift: Sockets are State, Not Threads
- **Apache's Question**: *"How do I make the worker per connection cheaper?"*
- **NGINX's Question**: *"Why is there a worker per connection at all?"*

```
THE NGINX DESIGN PRINCIPLE:
A connection does NOT need a thread. 
A connection needs a few hundred bytes of memory state and an entry in a kernel list 
until it has data to transmit. 
When data arrives, a CPU core processes it, returns to the loop, and moves on.
```

By decoupling network connections from operating system threads, NGINX eliminated thread stacks, synchronization locks, and context switching overhead.

---

### 2.3 The Master-Worker Architecture
NGINX uses a multi-process architecture consisting of a single master process and a fixed number of worker processes:

```
                          [ Master Process ]
                          - Runs as root (binds :80 / :443)
                          - Reads & validates nginx.conf
                          - Spawns and monitors workers
                          - Manages Unix signals (reload, rotate)
                          - NEVER touches a client connection!
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
  [ Worker Process 0 ]     [ Worker Process 1 ]     [ Cache Manager ]
  - Runs as unprivileged   - Runs as unprivileged   - Runs periodically
  - 1 process per CPU core - 1 process per CPU core - Evicts LRU files from
  - Single-threaded event  - Single-threaded event    disk to enforce
    loop (epoll/kqueue)      loop (epoll/kqueue)      max_size limits
  - Handles 10,000+ fds    - Handles 10,000+ fds
```

#### Process Stability:
Running `ps -o pid,ppid,args -C nginx` shows that the worker process count remains constant under load:
```text
2663     1  nginx: master process /usr/sbin/nginx
2665  2663  nginx: worker process
2666  2663  nginx: worker process
2667  2663  nginx: cache manager process
```
While a `fork()`-per-request server spawns thousands of short-lived processes under traffic spikes, NGINX's process count does not change. Worker processes are spawned once at startup and run indefinitely.

---

### 2.4 All of NGINX in Six Lines of C
The core execution loop of an NGINX worker resides in `src/event/ngx_event.c` within the function `ngx_process_events_and_timers()`:

```c
timer = ngx_event_find_timer();
ngx_process_events(cycle, timer, flags);                /* Executes epoll_wait() */
ngx_event_process_posted(cycle, &ngx_posted_accept_events);
ngx_shmtx_unlock(&ngx_accept_mutex);
ngx_event_expire_timers();
ngx_event_process_posted(cycle, &ngx_posted_events);
```

#### How the Loop Works:
1. **Line 1 (`ngx_event_find_timer`)**: Finds the earliest expiring timer across all tracked connections (e.g., keep-alive or client read timeouts).
2. **Line 2 (`ngx_process_events`)**: Executes `epoll_wait()`. The worker spends over **99.9% of its lifecycle asleep** here. The calculated `timer` is passed as the sleep timeout. NGINX does not poll for timeouts; the kernel wakes the process when I/O occurs or the timer expires.
3. **Line 3 (`ngx_posted_accept_events`)**: New client connection events are processed first, adding established sockets to the event loop.
4. **Line 4 (`ngx_shmtx_unlock`)**: The shared memory accept mutex is released so other workers can accept incoming connections.
5. **Line 5 (`ngx_event_expire_timers`)**: Expired timers trigger their associated callback functions.
6. **Line 6 (`ngx_posted_events`)**: Active read and write events are processed from the queue.

---

## 3. Worker Synchronization & Kernel Mechanics

### 3.1 Decentralized Worker Load Balancing in One Arithmetic Line
To balance connection loads across worker processes without a central coordinator, NGINX uses a calculation in `src/event/ngx_event_accept.c`:

```c
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

- **Mechanism**: Every worker tracks its total connection pool capacity (`connection_n`) and its currently available connection slots (`free_connection_n`).
- **The Threshold**: If a worker uses more than **7/8ths (87.5%)** of its allocated connection slots, `ngx_accept_disabled` becomes positive.
- **The Action**: While `ngx_accept_disabled > 0`, the worker stops competing for new incoming connections on the listening socket. It skips acquiring the accept mutex, decrements the counter, and focuses on servicing existing connections, allowing sibling workers to accept new traffic.
- **Why this is resilient**: No shared memory counters, locks, or leader election schemes are required. Each worker makes local routing decisions based on its own state.

---

### 3.2 The Thundering Herd Problem: From `accept_mutex` to `SO_REUSEPORT`

```
THE THUNDERING HERD (2004):
1. A new TCP client connects to Port 80.
2. The operating system kernel wakes up ALL 8 worker processes waiting on accept().
3. Only ONE worker successfully accepts the socket; the other 7 receive EAGAIN.
4. Severe CPU waste due to 7 unnecessary context switches per connection!
```

#### The Legacy Fix: `accept_mutex`
To prevent the thundering herd, NGINX implemented an internal shared-memory mutex (`ngx_accept_mutex`). Only the single worker holding the lock added the listening socket to its `epoll` interest set.

#### The Modern Solution: Kernel-Level Load Balancing
In modern Linux, `accept_mutex` is **disabled by default** (since NGINX 1.11.3) because operating system kernels now resolve the issue natively:
1. **`EPOLLEXCLUSIVE` (Linux 4.5+)**: When multiple processes monitor the same socket via `epoll`, the kernel wakes up only one worker process upon an event.
2. **`SO_REUSEPORT` (Linux 3.9+)**: Each worker allocates its own independent listening socket on port 80. The Linux kernel uses a 4-tuple hash to distribute incoming connections across worker queues automatically.

---

### 3.3 `select` is $O(\text{watched})$, `epoll` is $O(\text{active})$

| Metric / Dimension | POSIX `select()` | Linux `epoll()` |
| :--- | :--- | :--- |
| **Watch List Storage** | Resides in user space. Rebuilt on every iteration. | Maintained in kernel memory via a Red-Black tree. |
| **Registration** | Passed to the kernel on every call. | Registered once using `epoll_ctl()`. |
| **Complexity** | $O(N)$ scanning all watched descriptors. | $O(\text{ready})$ returning only fired events. |
| **Work for 10,000 idle / 10 active**| **30,000 operations** (rebuild, copy, scan). | **10 operations** (returns only active events). |
| **Descriptor Ceiling** | Hardcoded `FD_SETSIZE = 1024`. | Scalable up to `ulimit -n` (millions). |
| **Impact of `ulimit -n`** | None. Capped by the compile-time bitmap. | Allows allocating millions of descriptors. |

---

### 3.4 The Stale Event Bug: Pointer Tagging and Generation Bits

A challenging bug in high-concurrency event loops involves stale event processing:

```
THE STALE EVENT HAZARD:
1. epoll_wait() returns a batch of 32 ready events for processing.
2. Event #1 triggers application logic that forcibly closes the socket at Event #7.
3. An incoming client connects, and the OS kernel recycles the newly freed fd for Event #7.
4. When the loop processes Event #7 in the current batch, it delivers old data to the new client!
```

#### NGINX's Solution: Pointer Tagging via Data Alignment
In `src/event/modules/ngx_epoll_module.c`, NGINX resolves this without extra memory allocations:
```c
c = event_list[i].data.ptr;
instance = (uintptr_t) c & 1;                              /* Extract lowest bit */
c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1); /* Clear lowest bit */

if (c->fd == -1 || rev->instance != instance) {
    /* Stale event detected: this descriptor was closed earlier in the batch */
    continue;
}
```
- Because C structures are aligned to 4-byte or 8-byte boundaries in memory, **the lowest bit of a pointer is always zero**.
- NGINX borrows this unused bit as an **`instance` generation flag**.
- When a connection structure is closed and reused, NGINX flips this bit (`instance = !instance`).
- If an event arrives with a mismatched generation bit, the event loop detects it as a stale event from a closed connection and discards it.

---

## 4. Zero-Copy Data Transfer: `sendfile()` & kTLS

### 4.1 Four Memory Copies Become Two
When serving static files using standard `read()` and `write()` calls, data traverses the user/kernel boundary multiple times:

```
Traditional read() + write() (4 Data Copies, 4 Context Switches):
[ Hard Disk ] ──DMA──> [ Kernel Page Cache ] ──CPU Copy──> [ User-Space Buffer ]
                                                                 │
[ NIC Wire ]  <──DMA── [ Kernel Socket Buffer ] <──CPU Copy──────┘

Zero-Copy sendfile() (2 DMA Copies, 0 User-Space Copies):
[ Hard Disk ] ──DMA──> [ Kernel Page Cache ] ──Descriptor Ptr──> [ Socket Buffer ] ──DMA──> [ NIC Wire ]
```

By calling `sendfile(sock_fd, file_fd, NULL, bytes)`, data transfers directly from the kernel page cache to the network card via Direct Memory Access (DMA). The data never enters user-space memory, reducing CPU overhead and eliminating user-space memory copies.

---

### 4.2 The Conflict: Why `gzip on;` Breaks `sendfile on;`
- **The Core Constraint**: `sendfile` operates entirely within the operating system kernel.
- If an application needs to **inspect or modify bytes**—such as compressing content with Gzip, applying string substitutions, or encrypting data with TLS—the bytes must be loaded into user-space memory.
- Consequently, enabling runtime compression (`gzip on;`) disables `sendfile`, forcing NGINX to fall back to user-space buffering. (Pre-compressed static assets via `gzip_static on;` avoid this issue by serving `.gz` files directly via `sendfile`).

---

### 4.3 Kernel-Space TLS (kTLS)
Historically, serving static files over HTTPS required loading data into user space to run OpenSSL encryption, preventing the use of `sendfile`.

Linux 4.13 introduced **kTLS (Kernel TLS)**:
- The initial TLS handshake (key negotiation and certificate validation) runs in user space via OpenSSL.
- Once symmetric encryption keys (e.g., AES-GCM) are established, they are offloaded to the Linux kernel via `setsockopt(..., SOL_TLS, ...)`.
- NGINX can then execute `sendfile()` over encrypted HTTPS connections: the kernel streams data directly from the page cache, encrypts it using hardware-accelerated AES-NI instructions in the kernel, and passes it directly to the NIC.

---

### 4.4 The Loopback Benchmark Reality
When benchmarking `sendfile` against `read`/`write` on local loopback (`127.0.0.1`):
- `sendfile` may not show significant wall-clock speedups because loopback interfaces do not use physical network hardware or DMA; the kernel performs a memory copy in both cases.
- **The Real Performance Gain**: `sendfile` reduces thousands of system calls to a single call and keeps large files out of user-space memory. The benefit appears as **increased concurrency headroom**, freeing CPU cycles to serve other clients.

---

## 5. Reverse Proxying: The Architecture of the Edge

### 5.1 Forward Proxy vs. Reverse Proxy

```
FORWARD PROXY (e.g., Squid, 1996):
[ Client A ] ──┐
[ Client B ] ──┼──> [ Forward Proxy ] ─── WAN ───> [ The Public Internet ]
[ Client C ] ──┘    (Protects client network, caches outbound downloads)

REVERSE PROXY (e.g., NGINX, 2004):
[ The Public Internet ] ───> [ Reverse Proxy (NGINX) ] ──┬──> [ App Server 1 ]
                             (Absorbs slow connections,  ├──> [ App Server 2 ]
                              terminates TLS, caches)    └──> [ Database API  ]
```

- **Forward Proxy**: Sits at the edge of client networks. Caches outbound web traffic and enforces organizational security policies.
- **Reverse Proxy**: Sits in front of origin servers. Routes requests, terminates SSL/TLS, balances load, and shields application servers from slow clients.

---

### 5.2 Absorbing Slow Clients (2G Buffering)
A key advantage of placing NGINX in front of application runtimes (such as Python WSGI, Ruby Unicorn, or Node.js) is **request buffering**:
- A mobile client on a slow 2G connection may take 10 seconds to upload an HTTP POST request.
- If an application server connects directly to that client, its worker thread remains blocked for 10 seconds waiting for the upload to finish.
- **NGINX Request Buffering**: NGINX uses its event loop to read incoming bytes over 10 seconds without blocking worker processes. Once the request payload is fully received, NGINX forwards it to the application server over a local Unix domain socket in less than a millisecond.

---

## 6. Shared-Memory Caching Architecture

### 6.1 Cache Storage: Shared Memory Index + Disk Subdirectories
NGINX configures proxy caching using the `proxy_cache_path` directive:
```nginx
proxy_cache_path /data/nginx/cache 
                 levels=1:2 
                 keys_zone=demo:10m 
                 max_size=100m 
                 inactive=60s 
                 use_temp_path=off;
```

```
┌──────────────────────────────────────┬──────────────────────────────────────┐
│  SHARED MEMORY (keys_zone=demo:10m)  │      DISK STORAGE (levels=1:2)       │
├──────────────────────────────────────┼──────────────────────────────────────┤
│ - 10 MB RAM allocated in shared      │ - Cache payload files stored on disk.│
│   memory (accessible by all workers).│ - MD5 hash generates the path:       │
│ - Holds Red-Black Tree (lookup index)│   MD5 = b7f8a9...c2                  │
│   and Doubly-Linked LRU Queue.       │   Path: /data/nginx/cache/2/c2/b7... │
│ - Stores metadata: ~8,000 keys per MB│ - levels=1:2 prevents putting 1M     │
│   (10 MB holds ~80,000 cached URLs). │   files in one directory.            │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

---

### 6.2 `inactive` vs. `proxy_cache_valid` (TTL)
These two cache expiration controls serve different purposes:
- **`proxy_cache_valid 200 10s;`**: Defines the **Cache TTL (Freshness)**. A response is fresh for 10 seconds. After 10 seconds, it is marked as stale, and subsequent requests fetch a fresh copy from the origin.
- **`inactive=60s;`**: Defines the **Storage Eviction Timer**. If an item is not accessed for 60 seconds, NGINX deletes it from disk and memory, regardless of its TTL.

```
Scenario: proxy_cache_valid = 10s; inactive = 60s;
Time 0s  : Object fetched and cached. (Fresh)
Time 5s  : Request arrives. Served as Cache HIT.
Time 15s : Request arrives. Object is Stale (15s > 10s TTL). Refetches from origin.
Time 80s : No requests for 65 seconds (> 60s inactive). Object is deleted from disk.
```

---

### 6.3 Intrusive Doubly-Linked Lists & The Single-Line LRU Eviction
In `src/http/ngx_http_file_cache.c`, the Cache Manager process evicts entries using a doubly-linked list:
```c
q = ngx_queue_last(&cache->sh->queue);
fcn = ngx_queue_data(q, ngx_http_file_cache_node_t, queue);

wait = fcn->expire - now;
if (wait > 0) { break; }           /* Tail item is still fresh */

if (fcn->count == 0) {             /* Safe: no clients are currently streaming this file */
    ngx_http_file_cache_delete(cache, q, name);
}
```
- **Intrusive Design**: Pointers live directly inside the cache structure, avoiding extra memory allocations during list operations.
- **Reference Counting (`count == 0`)**: An object currently being streamed to a client has an active reference count, preventing it from being deleted during eviction passes.

---

### 6.4 Two Directives That Save You at 3 AM: Cache Locks & Stale Fallbacks

#### 1. `proxy_cache_lock on;` (Defeating Cache Stampedes)
- **The Problem**: When a high-traffic cached resource expires, hundreds of concurrent requests arrive simultaneously, generating a sudden burst of cache misses that can overwhelm the backend database.
- **The Solution**: With `proxy_cache_lock on;`, only **one request** is sent to the upstream origin to re-populate the cache. Other concurrent requests wait for the response and serve from the newly updated cache.

#### 2. `proxy_cache_use_stale` (High Availability Fallback)
```nginx
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
```
- **The Solution**: If the origin server crashes, times out, or returns a 500-series error, NGINX serves the expired cached copy rather than returning an error to the user.
- **`updating` Parameter**: When a cache item is being refreshed by one worker, other incoming requests are immediately served the stale copy without waiting.

---

## 7. Routing Precedence & Data Structures

### 7.1 The Four Location Directives and Evaluation Precedence
NGINX location blocks are evaluated according to specific precedence rules, **not their order in the configuration file**:

```
EVALUATION PRECEDENCE ORDER:
1. Exact Match          : location = /path
2. Preferential Prefix  : location ^~ /path    (Stops regex search if matched)
3. Regular Expressions  : location ~ \.php$    (Evaluated in configuration file order)
4. Longest Prefix Match : location /path       (Used if no regex matches)
```

| # | Directive Form | Matching Rule | Search Behavior | Example Match for `/images/a.php` |
| :---: | :--- | :--- | :--- | :---: |
| **1** | `location = /exact` | Exact URI match | Match wins immediately. Search stops. | No match |
| **2** | `location ^~ /images/`| Longest prefix match | Matches and **suppresses regular expression evaluation**! | **WINS (Rule 2)** |
| **3** | `location ~ \.php$` | Regular expression | Evaluated in configuration order. First regex match wins. | Suppressed by `^~` |
| **4** | `location /` | Generic prefix match | Longest prefix match used only if no regex matches. | Fallback only |

> **Common Pitfall**: In the example above, `/images/a.php` matches `location ^~ /images/` instead of `location ~ \.php$`. The `^~` prefix flag instructs NGINX to skip regex evaluations on a match.

---

### 7.2 Why 500 Prefixes are Free and 500 Regexes are Not

```
PREFIX MATCHING (Ternary Search Tree):
         [ / ]
        /     \
    [ /api ]  [ /images ] ───> O(URI length) lookup. 
      /                            Adding 500 prefixes costs almost zero CPU!
   [ /api/v1 ]

REGEX MATCHING (Linear Array):
   [ Regex 1 ] ──> [ Regex 2 ] ──> ... ──> [ Regex 500 ]  ───> O(N) linear evaluation.
   Checked sequentially on every single request. 500 regexes hurt CPU throughput!
```

- **Prefix Locations**: Stored in a **Ternary Search Tree (TST)** constructed at configuration parse time. Lookups scale with URI length rather than the number of location blocks.
- **Regex Locations**: Maintained in an array and evaluated sequentially on every request. Large numbers of regex locations add measurable latency to request handling.

---

### 7.3 Why NGINX Has No `.htaccess`
- In Apache, directory-level configuration via `.htaccess` required the server to issue multiple `stat()` system calls checking every parent directory on every request:
  $$\text{Request: } \texttt{/var/www/site/images/pic.png} \implies \text{stat(/var), stat(/var/www), stat(/var/www/site), ...}$$
- NGINX does not support `.htaccess`. Configurations are parsed into memory at startup, eliminating per-request filesystem checks.

---

## 8. Upstream Load Balancing & Fallbacks

### 8.1 Smooth Weighted Round Robin Algorithm
When distributing traffic across backends with uneven capacities (e.g., weights `3:1`), simple round-robin implementations produce bursts (e.g., `A, A, A, B`).

NGINX uses **Smooth Weighted Round Robin** to distribute traffic evenly:
```c
peer->current_weight += peer->effective_weight;
total += peer->effective_weight;

if (best == NULL || peer->current_weight > best->current_weight) {
    best = peer;
}
best->current_weight -= total;
```

#### Example Execution (Server A weight 3, Server B weight 1, Total = 4):
- **Request 1**: A=3, B=1 $\rightarrow$ Select **A** $\rightarrow$ A drops to $3 - 4 = -1$.
- **Request 2**: A=$(-1+3)=2$, B=$(1+1)=2$ $\rightarrow$ Tie-break selects **B** $\rightarrow$ B drops to $2 - 4 = -2$.
- **Request 3**: A=$(2+3)=5$, B=$(-2+1)=-1$ $\rightarrow$ Select **A** $\rightarrow$ A drops to $5 - 4 = 1$.
- **Request 4**: A=$(1+3)=4$, B=$(-1+1)=0$ $\rightarrow$ Select **A** $\rightarrow$ A drops to $4 - 4 = 0$.

$$\mathbf{\text{Selection Sequence: }} \mathbf{A} \longrightarrow \mathbf{B} \longrightarrow \mathbf{A} \longrightarrow \mathbf{A}$$

The load maintains the desired 3:1 ratio without sending consecutive bursts to the primary server.

---

### 8.2 `try_files` and Pretty URLs
Single Page Applications (React, Vue) and modern web frameworks rely on `try_files` for URL routing:
```nginx
location / {
    try_files $uri $uri/ /index.html?$args;
}
```
1. Checks if the URI matches a static file on disk (`$uri`).
2. Checks if the URI matches a directory (`$uri/`).
3. If neither exists, routes to `/index.html?$args`.
- **Key Behavior**: The final argument is an **unconditional fallback** and is not checked against the filesystem.

---

## 9. A Walkthrough of the NGINX Source Code

### 9.1 Source Tree Layout
- **`src/core/`**: NGINX standard library (`ngx_string`, `ngx_array`, `ngx_rbtree`, `ngx_queue`, `ngx_palloc`).
- **`src/event/`**: Event loop engine (`ngx_event.c`) and platform pollers (`ngx_epoll_module.c`, `ngx_kqueue_module.c`).
- **`src/http/`**: HTTP state machine, request processing phases, and modules.
- **`src/os/unix/`**: OS abstraction layer (`ngx_process.c`, `ngx_linux_sendfile_chain.c`).
- **`src/stream/`**: Layer 4 TCP/UDP proxying and load balancing.
- **`src/mail/`**: Mail proxy engine (SMTP, IMAP, POP3).

---

### 9.2 Memory Management: `ngx_palloc` (Pools without `free()`)
NGINX avoids calling `free()` for individual allocations during request processing. Instead, it uses **Memory Pools** (`ngx_palloc.h`):

```
┌────────────────────────────────────────────────────────┐
│           REQUEST MEMORY POOL (ngx_pool_t)             │
├────────────────────────────────────────────────────────┤
│ - Allocated when client connects / initiates request. │
│ - Allocations (headers, URI tokens) pull from pool.   │
│ - Zero per-allocation free() calls during processing.  │
│ - When request completes: ngx_destroy_pool() frees     │
│   the entire memory block at once!                     │
└────────────────────────────────────────────────────────┘
```
This design reduces heap fragmentation, removes the risk of memory leaks, and eliminates cleanup error-handling paths.

---

### 9.3 Intrusive Doubly-Linked Lists: `ngx_queue.h`
NGINX links structures using intrusive lists:
```c
struct ngx_http_file_cache_node_s {
    ngx_queue_t queue;       /* List pointers (next, prev) embedded inside data! */
    time_t expire;
    u_char key[2 * 16];
};
```
Inserting or removing a node does not require separate memory allocations. Container structures are accessed via pointer arithmetic (`offsetof` macros).

---

### 9.4 The 27-State Resumable HTTP Parser
In `src/http/ngx_http_parse.c`, `ngx_http_parse_request_line()` parses incoming requests using a byte-by-byte state machine:
```c
state = r->state;  /* Resumes where parsing last stopped */
for (p = b->pos; p < b->last; p++) {
    ch = *p;
    switch (state) {
        case sw_start: ...
        case sw_method: ...
        case sw_spaces_before_uri: ...
        case sw_uri: ...
        case sw_http_09: ...
    }
}
```
Because network packets may arrive in fragments over non-blocking sockets, the parser stores its current state in the request structure (`r->state`). When more data arrives, parsing resumes from the exact state without buffering the entire string.

---

### 9.5 The Eleven HTTP Request Processing Phases
NGINX processes requests through 11 distinct phases (`src/http/ngx_http_core_module.c`):

```
1. POST_READ    ──> Reads raw request headers.
2. SERVER_REWRITE──> Virtual host URI rewrites.
3. FIND_CONFIG  ──> Matches location block (TST / regex).
4. REWRITE      ──> Location-level URI rewrites.
5. POST_REWRITE ──> Prevents infinite rewrite loops.
6. PREACCESS    ──> Rate limiting (limit_req, limit_conn).
7. ACCESS       ──> Access controls (auth_basic, allow/deny).
8. POST_ACCESS  ──> Access rule validation.
9. PRECONTENT   ──> Pre-content handlers (try_files).
10. CONTENT     ──> Serves content (static, proxy_pass, fastcgi_pass).
11. LOG         ──> Access logging (ngx_http_log_module).
```

Each phase executes non-blocking handlers. If a handler returns `NGX_AGAIN`, execution is suspended, returning control to the event loop until the required I/O event fires.

---

## 10. The Seven Steps of Server Evolution

```
┌────┬──────────────┬──────────────────┬─────────────────┬───────────────────────────────────────────┐
│ #  | Server Model | Historical Origin| Scalability Cap | Key Bottleneck / Architectural Takeaway   |
├────┼──────────────┼──────────────────┼─────────────────┼───────────────────────────────────────────┤
│ 01 | fork()       | Apache 1.3 (1995)| Hundreds        | Process creation cost on request path     |
│ 02 | pthreads     | Apache 2.0 (1999)| Thousands       | 8 MB stack per thread reserves 80 GB RAM  |
│ 03 | select()     | BSD (1983)       | 1,024 clients   | FD_SETSIZE bitmap limit; O(N) scanning    |
│ 04 | epoll()      | NGINX (2004)     | Millions        | O(ready) performance; scales to C10K/C10M |
│ 05 | sendfile()   | Linux 2.2        | NIC Line Rate   | Zero-copy DMA transfers for static data   |
│ 06 | FastCGI      | OpenMarket (1996)| App Dependent   | Persistent worker pool over sockets       |
│ 07 | Edge Proxy   | Modern NGINX     | Internet Scale  | Reverse proxy: buffers slow clients, TLS  |
└────┴──────────────┴──────────────────┴─────────────────┴───────────────────────────────────────────┘
```

---

## 11. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
What was the primary design motivation behind Igor Sysoev's creation of NGINX in 2002?
- A) To create a web browser with inline image support.
- B) To serve tens of thousands of mostly-idle connections using an event loop in a single process, offloading dynamic content generation to backend application servers.
- C) To replace Linux kernel networking drivers with user-space DPDK.
- D) To implement an in-process PHP interpreter.

**Answer: B**  
**Explanation:** Sysoev developed NGINX to resolve the C10K problem at Rambler, where Apache servers were failing under large numbers of idle Keep-Alive connections. NGINX was architected as an asynchronous reverse proxy rather than an in-process application server.

---

### Question 2
In NGINX's architecture, what is the role of the Master Process?
- A) It executes non-blocking `epoll_wait()` calls to service client HTTP requests.
- B) It binds privileged network ports (e.g., 80/443), reads configuration files, manages worker lifecycles, and handles signals, without handling client connections directly.
- C) It stores cached response payloads on local disk.
- D) It compiles PHP scripts into native bytecode.

**Answer: B**  
**Explanation:** The NGINX master process runs as root to bind privileged ports and manage configuration and workers. It never handles client network I/O; all client traffic is processed by unprivileged worker processes.

---

### Question 3
How does an NGINX worker determine when to sleep and when to wake up inside its core event loop?
- A) It continuously loops, consuming 100% CPU on a spinlock.
- B) It finds the nearest expiring connection timer and passes that duration as the timeout to `epoll_wait()`, allowing the kernel to wake it on I/O or timer expiration.
- C) It relies on an external cron daemon to send interrupt signals every millisecond.
- D) It sleeps for a hardcoded 10-second interval on every loop.

**Answer: B**  
**Explanation:** NGINX calls `timer = ngx_event_find_timer()` and passes the value directly to `epoll_wait()`. The worker sleeps until an I/O event arrives or the timer expires, avoiding polling loops.

---

### Question 4
How does the load balancing arithmetic `ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;` prevent worker overload?
- A) It asks a central manager process to reassign connections.
- B) Once a worker utilizes more than 7/8ths (87.5%) of its connection pool, `ngx_accept_disabled` becomes positive, causing the worker to stop accepting new connections so sibling workers can handle them.
- C) It drops incoming TCP SYN packets with an RST.
- D) It halves the size of the worker's socket buffers.

**Answer: B**  
**Explanation:** When a worker's free connection pool falls below 1/8th of its total capacity, `ngx_accept_disabled` becomes greater than zero. The worker temporarily stops acquiring the accept mutex, allowing sibling workers to accept new connections.

---

### Question 5
How does NGINX resolve the "Stale Event Bug" that occurs when an `epoll_wait()` batch contains an event for a file descriptor that was closed earlier in the same iteration?
- A) It flushes and restarts the `epoll` instance after every processed event.
- B) It uses the lowest bit of the structure pointer as an `instance` generation flag, skipping the event if the flag does not match the active descriptor's generation.
- C) It introduces a 50-millisecond delay between event iterations.
- D) It forces the operating system to avoid reusing file descriptors.

**Answer: B**  
**Explanation:** Memory alignment ensures the lowest bit of structure pointers is zero. NGINX tags pointers with this bit as a generation counter (`instance`), flipping it on reuse. If an event in an active batch carries a mismatched generation bit, NGINX identifies it as stale and skips it.

---

### Question 6
Why does enabling runtime Gzip compression (`gzip on;`) disable zero-copy `sendfile` in NGINX?
- A) `sendfile` is restricted to UDP datagrams.
- B) `sendfile` transfers data entirely within kernel space; compressing bytes requires loading them into user-space memory.
- C) Gzip compression requires root privileges.
- D) The Linux kernel does not support Gzip compression.

**Answer: B**  
**Explanation:** `sendfile` streams data directly from the kernel page cache to the network card via DMA. Because data never passes through user space, NGINX cannot modify or compress the bytes while using `sendfile`.

---

### Question 7
How does Kernel-Space TLS (kTLS) allow NGINX to use `sendfile` over encrypted HTTPS connections?
- A) It bypasses the TLS handshake entirely.
- B) The TLS handshake runs in user space, but symmetric encryption keys are offloaded to the kernel, allowing `sendfile` to stream and encrypt page cache data directly.
- C) It converts HTTPS traffic into plaintext HTTP.
- D) It offloads TLS processing to the client's router.

**Answer: B**  
**Explanation:** kTLS allows user space to negotiate TLS parameters and offload symmetric cipher keys to the kernel. NGINX can then execute `sendfile()`, letting the kernel encrypt data in-flight using hardware-accelerated instructions.

---

### Question 8
What is the primary operational difference between a Forward Proxy and a Reverse Proxy?
- A) Forward proxies handle UDP, whereas reverse proxies handle TCP.
- B) A forward proxy sits near clients to cache outbound internet requests, whereas a reverse proxy sits in front of origin servers to balance load and protect backends.
- C) Reverse proxies cannot terminate TLS certificates.
- D) Forward proxies require multi-threaded processors.

**Answer: B**  
**Explanation:** Forward proxies (e.g., Squid) sit on client networks to protect corporate downlinks and cache outbound traffic. Reverse proxies (e.g., NGINX) sit on server networks to route traffic, terminate SSL, and buffer requests.

---

### Question 9
In the directive `proxy_cache_path /cache keys_zone=demo:10m levels=1:2 inactive=60s;`, what is stored inside the 10 MB `keys_zone`?
- A) The raw HTML, video, and image payload data.
- B) The shared-memory index, including the Red-Black tree lookup and the LRU queue.
- C) SSL private certificates.
- D) Operating system kernel page tables.

**Answer: B**  
**Explanation:** The `keys_zone` allocates shared memory for the cache index (Red-Black tree and LRU metadata). The cached content itself is written to disk under the specified directory path.

---

### Question 10
If NGINX is configured with `proxy_cache_valid 200 10s;` and `inactive=60s;`, what happens to a cached object 30 seconds after its creation if no new requests have arrived?
- A) It is deleted from the disk cache.
- B) It remains on disk but is considered stale (expired TTL); an incoming request will trigger a refresh from the origin.
- C) NGINX returns an HTTP 504 Gateway Timeout.
- D) It is automatically converted into a dynamic FastCGI record.

**Answer: B**  
**Explanation:** The object's TTL is 10 seconds, so it is stale after 10 seconds. However, because it has been inactive for only 30 seconds (less than the 60-second `inactive` threshold), it remains stored on disk.

---

### Question 11
What is the purpose of the `proxy_cache_lock on;` directive?
- A) It locks the cached files on disk to prevent writes.
- B) It prevents cache stampedes: when a popular cache entry expires, only one request is sent to the origin, while other concurrent requests wait for the cache to update.
- C) It requires clients to authenticate with a password to access cached files.
- D) It prevents NGINX worker processes from restarting.

**Answer: B**  
**Explanation:** `proxy_cache_lock on;` prevents cache stampedes. When a cached resource expires, NGINX permits only one request to fetch an update from the backend, serving other waiting requests from the refreshed cache.

---

### Question 12
Given the location blocks:
1. `location ^~ /images/`
2. `location ~ \.php$`
3. `location /`  
Which location block processes the request `GET /images/avatar.php`?
- A) `location ~ \.php$` because regular expressions take absolute precedence.
- B) `location ^~ /images/` because preferential prefix matching suppresses regular expression searches.
- C) `location /` because it is the default root location.
- D) The request fails with an HTTP 404 error.

**Answer: B**  
**Explanation:** The `^~` modifier denotes a preferential prefix match. If matched, NGINX skips checking regular expression location blocks, making `location ^~ /images/` the winning block.

---

### Question 13
Why does having 500 prefix location blocks have minimal impact on NGINX performance, whereas 500 regular expression location blocks can significantly degrade throughput?
- A) Regex locations are stored on disk, whereas prefixes are stored in RAM.
- B) Prefix locations are indexed in a Ternary Search Tree evaluated in $O(\text{URI length})$ time, whereas regex locations are stored in a linear array evaluated sequentially on every request.
- C) Regex evaluation disables the Linux kernel TCP stack.
- D) Prefix locations run on the master process.

**Answer: B**  
**Explanation:** NGINX compiles static prefix locations into a Ternary Search Tree at startup, allowing lookups to scale with URI length. Regular expression blocks are stored in an array and evaluated sequentially, which can increase CPU overhead as the number of rules grows.

---

### Question 14
How does NGINX's Smooth Weighted Round Robin algorithm differ from standard naive weighted round-robin?
- A) It assigns all requests to the server with the lowest IP address.
- B) It interleaves requests smoothly across backends (e.g., `A, B, A, A` for 3:1) rather than sending consecutive bursts to the highest-weighted backend (`A, A, A, B`).
- C) It requires sticky browser sessions.
- D) It routes requests over UDP instead of TCP.

**Answer: B**  
**Explanation:** Naive weighted round-robin clusters requests onto higher-weighted servers. NGINX's smooth algorithm dynamically adjusts peer weights to distribute requests evenly without traffic bursts.

---

### Question 15
How does NGINX's `ngx_palloc` memory pool improve performance and reliability?
- A) It executes garbage collection sweeps every 5 seconds.
- B) It allocates memory from a pool tied to the request and destroys the entire pool when the request completes, eliminating per-request `free()` calls and preventing memory leaks.
- C) It stores request variables in operating system environment variables (`envp`).
- D) It disables memory paging in the Linux kernel.

**Answer: B**  
**Explanation:** `ngx_palloc` binds memory allocations to a request pool. When the request lifecycle ends, NGINX frees the entire pool at once, eliminating the overhead of individual `free()` calls and reducing memory leak risks.

---

## 12. Session 4 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **Historical Lineage** | CERN httpd (1990) $\rightarrow$ NCSA HTTPd (1993, CGI) $\rightarrow$ Apache (1995) $\rightarrow$ NGINX (2004, Igor Sysoev). Mosaic was a browser. |
| **Deleting the Worker** | Connections are lightweight memory states in a kernel list, not dedicated OS threads. |
| **Master vs. Worker** | Master binds ports, reads configs, and manages signals. Workers (1 per core) handle all client I/O via `epoll`. |
| **6-Line Loop** | `epoll_wait()` uses timer arguments to sleep efficiently. Handlers run from posted queues after releasing the accept mutex. |
| **Worker Balancing** | `ngx_accept_disabled = (conn_n / 8) - free_conn_n`. Workers step back when utilization exceeds 87.5%. |
| **Thundering Herd** | Resolved historically by `accept_mutex`; handled natively in modern kernels via `EPOLLEXCLUSIVE` and `SO_REUSEPORT`. |
| **Stale Event Bug** | Solved using pointer tagging: the lowest bit of aligned pointers stores an `instance` generation flag. |
| **Zero-Copy `sendfile`**| Transfers data from kernel page cache directly to socket buffers via DMA. Incompatible with runtime `gzip`. |
| **Kernel TLS (kTLS)** | Allows `sendfile` over HTTPS by offloading symmetric encryption to the Linux kernel. |
| **Slow Client Absorption**| NGINX buffers slow 2G uploads asynchronously, forwarding completed requests to backends over fast local sockets. |
| **Shared Memory Cache** | `keys_zone` stores the Red-Black tree and LRU metadata in shared RAM; payload files are stored on disk. |
| **`inactive` vs `valid`** | `proxy_cache_valid` controls content freshness (TTL). `inactive` deletes unread files from disk. |
| **Anti-Stampede Lock** | `proxy_cache_lock on;` sends only one request to the origin on cache misses, preventing origin overloads. |
| **Location Precedence** | Exact (`=`) $\rightarrow$ Preferential Prefix (`^~`) $\rightarrow$ Regex (`~` / `~*`) $\rightarrow$ Longest Prefix (`/`). |
| **Prefixes vs. Regexes** | Prefixes use a Ternary Search Tree ($O(\text{URI length})$); regexes use a linear array ($O(N)$). |
| **Memory Pools** | `ngx_palloc` frees entire request pools at once, eliminating per-allocation `free()` calls. |
| **11 Request Phases** | Modular execution pipeline (`POST_READ` to `LOG`) functioning as a cooperative coroutine in C. |
