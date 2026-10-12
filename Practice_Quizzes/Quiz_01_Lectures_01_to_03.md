# Practice Quiz Bank: Lectures 01 to 03 (Exam & Interview Prep)

> **Coverage**: 
> - **Lecture 01**: Network Programming 101 (The 7 Syscalls, Sockets, Ports, Endianness, Signals, TLV)
> - **Lecture 02**: Computer Networking Eagle Eye View (OSI, SS7, SMS Limits, TCP/UDP Headers, Handshakes, BBR)
> - **Lecture 03**: Server Side Considerations for Scale (Concurrency, `select` vs `epoll`, CGI/FastCGI, Little's Law, Zero-Copy)
>
> **Design**: High-yield, rigorous multiple-choice questions matching real classroom quizzes. Every question includes the **correct answer**, **step-by-step mathematical calculations**, and a **"Concept & Trap Breakdown"** explaining why most students choose the wrong option.

---

## Question 1: The Default TCP MSS / MTU Trap
**What is the default Maximum Segment Size (MSS) for a TCP connection if the MSS option is omitted in the SYN segment?**
- A) 1500 bytes
- B) 1460 bytes
- C) 536 bytes
- D) 576 bytes

**Correct Answer: C (536 bytes)**

### Step-by-Step Mathematical Explanation:
1. **The Universal Minimum Reassembly Buffer (RFC 791)**:  
   Every IPv4 host on the internet must be able to receive and reassemble an IP datagram of at least **576 octets** without fragmentation.
2. **Standard IPv4 and TCP Header Sizes**:
   $$\text{Standard IPv4 Header} = 20 \text{ bytes}$$
   $$\text{Standard TCP Header} = 20 \text{ bytes}$$
   $$\text{Total Transport Overhead} = 20 + 20 = 40 \text{ bytes}$$
3. **Default MSS Calculation (RFC 879)**:
   $$\mathbf{\text{Default MSS}} = \text{Minimum IP Datagram} - \text{IP Header} - \text{TCP Header}$$
   $$\mathbf{\text{Default MSS}} = 576 - 20 - 20 = \mathbf{536 \text{ bytes}}$$

### Concept & Trap Breakdown:
- **Why 50% of students choose 1500 bytes**: 1500 bytes is the default **Ethernet MTU (Maximum Transmission Unit)** at Layer 2, not a TCP segment size!
- **Why 41% of students choose 1460 bytes**: 1460 bytes is the typical MSS negotiated on modern Ethernet networks ($1500 - 40 = 1460$). However, if a client or server does *not* explicitly advertise an MSS option in the TCP SYN segment, RFC 879 mandates that the default assumed MSS is strictly **536 bytes**.

---

## Question 2: Bit Shifting Arithmetic
**What is the mathematical impact of bit shifting an integer value to the left by 1 bit (`x << 1`)?**
- A) Doubles the value ($x \times 2$)
- B) Halves the value ($x / 2$)
- C) Increases the value by a power of 3
- D) Leaves the value unchanged

**Correct Answer: A (Doubles the value)**

### Step-by-Step Mathematical Explanation:
1. In a binary positional numeral system, each position represents a power of 2:
   $$\text{Decimal 5 in binary} = 0000\,0101_2 = (1 \times 2^2) + (0 \times 2^1) + (1 \times 2^0) = 4 + 0 + 1 = 5$$
2. Shift all bits to the left by 1 position:
   $$0000\,0101_2 \ll 1 = 0000\,1010_2$$
3. Evaluate the new decimal value:
   $$(1 \times 2^3) + (0 \times 2^2) + (1 \times 2^1) + (0 \times 2^0) = 8 + 0 + 2 + 0 = 10$$
4. General rule for shifting left by $k$ bits:
   $$\mathbf{x \ll k = x \times 2^k}$$
   For $k = 1$: $x \times 2^1 = 2x$ (Doubles).
   *(Conversely, shifting right by 1 bit, `x >> 1`, performs integer division by 2: halves).*

---

## Question 3: Little's Law Concurrency Calculation
**A high-scale web service processes a steady throughput of 4,000 requests per second ($\lambda = 4,000\text{ req/s}$). Due to a database lock contention, the average request latency ($W$) increases from 50 milliseconds to 250 milliseconds. What is the new number of concurrent in-flight connections ($L$) held in memory, and how many additional connections must the server support?**
- A) 200 connections (0 additional)
- B) 1,000 connections (800 additional)
- C) 800 connections (600 additional)
- D) 1,250 connections (1,050 additional)

**Correct Answer: B (1,000 connections; 800 additional)**

### Step-by-Step Mathematical Explanation:
1. **Little's Law Formula**:
   $$\mathbf{L = \lambda \times W}$$
   *(where $L$ = in-flight concurrent connections, $\lambda$ = throughput in requests/second, and $W$ = latency in seconds).*
2. **Initial Concurrency ($L_1$)**:
   $$\lambda = 4,000\text{ req/s}, \quad W_1 = 50\text{ ms} = 0.050\text{ seconds}$$
   $$L_1 = 4,000 \times 0.050 = \mathbf{200 \text{ concurrent connections}}$$
3. **Degraded Concurrency ($L_2$)**:
   $$\lambda = 4,000\text{ req/s}, \quad W_2 = 250\text{ ms} = 0.250\text{ seconds}$$
   $$L_2 = 4,000 \times 0.250 = \mathbf{1,000 \text{ concurrent connections}}$$
4. **Additional Connections**:
   $$\Delta L = L_2 - L_1 = 1,000 - 200 = \mathbf{800 \text{ additional connections}}$$

### Concept & Trap Breakdown:
- **"Slowness is a memory bug"**: Notice that throughput ($\lambda$) did not change at all—it stayed at 4,000 RPS. Yet, because response time increased by $5\times$, the server is forced to hold **$5\times$ more open socket descriptors and kernel buffers in memory**.

---

## Question 4: Missing `htons()` Endianness Byte Swap
**A developer writing a server in C on an x86_64 laptop assigns the port directly without byte order conversion: `addr.sin_port = 2026;`. To which decimal port will the operating system actually bind the socket?**
- A) Port 2026
- B) Port 59911
- C) Port 2602
- D) Port 8080

**Correct Answer: B (Port 59911)**

### Step-by-Step Mathematical Explanation:
1. **Convert Port 2026 to Hexadecimal**:
   $$\frac{2026}{256} = 7 \text{ with remainder } 234$$
   $$\text{Quotient } 7 = \text{0x07}, \quad \text{Remainder } 234 = \text{0xEA}$$
   $$\text{Port 2026 in 16-bit hex} = \mathbf{\text{0x07EA}}$$
   - High byte (MSB) = `0x07`
   - Low byte (LSB) = `0xEA`
2. **Host Byte Order on x86 (Little-Endian)**:
   Little-Endian architectures store the **Least Significant Byte first** in memory:
   $$\text{RAM Layout: } [\text{0xEA}] \quad [\text{0x07}]$$
3. **Network Byte Order Interpretation (Big-Endian)**:
   The network stack and socket layer read those two bytes expecting Big-Endian order (Most Significant Byte first):
   $$\text{Evaluated Hex Value} = \mathbf{\text{0xEA07}}$$
4. **Convert `0xEA07` back to Decimal**:
   $$\text{0xEA} = (14 \times 16) + 10 = 234$$
   $$\text{0x07} = 7$$
   $$\text{Decimal Port} = (234 \times 256) + 7 = 59,904 + 7 = \mathbf{59,911}$$

### Concept & Trap Breakdown:
- On x86 and ARM laptops, failing to call `htons()` causes the low and high bytes to swap in network memory, causing the server to listen on port 59911 instead of 2026.

---

## Question 5: FTP 227 Port Number Calculation
**An FTP client in Passive Mode issues `PASV` and receives the server response: `227 Entering Passive Mode (10,0,0,1,117,48)`. On which TCP port must the client establish its data connection?**
- A) Port 11748
- B) Port 30000
- C) Port 117
- D) Port 48117

**Correct Answer: B (Port 30000)**

### Step-by-Step Mathematical Explanation:
1. In RFC 959, the 227 response encodes the IP address in the first four octets (`10.0.0.1`) and the 16-bit TCP port in the final two numbers as $(p1, p2)$.
2. The port is represented as two 8-bit bytes in Big-Endian order:
   - High byte = $p1 = 117$
   - Low byte = $p2 = 48$
3. **Port Reconstruction Formula**:
   $$\mathbf{\text{Port}} = (\mathbf{p1} \times 256) + \mathbf{p2}$$
4. Calculate:
   $$117 \times 256 = 29,952$$
   $$\text{Port} = 29,952 + 48 = \mathbf{30,000}$$

### Concept & Trap Breakdown:
- In 1971, computer terminal parsers avoided integers larger than 255. The port was split into two 8-bit halves ($p1 \times 2^8 + p2$), which is simply manual Big-Endian reconstruction in ASCII.

---

## Question 6: The SMS 140-Byte GSM-7 Packing Math
**A user sends a text message containing 12 characters: `"Hello World!"`. Using the standard GSM 7-bit alphabet, how many total bits does this message occupy, how many 8-bit octets are required to pack it, and how many bits of zero-padding are appended?**
- A) 96 bits, 12 octets, 0 bits padding
- B) 84 bits, 11 octets, 4 bits padding
- C) 84 bits, 10 octets, 4 bits padding
- D) 72 bits, 9 octets, 0 bits padding

**Correct Answer: B (84 bits, 11 octets, 4 bits padding)**

### Step-by-Step Mathematical Explanation:
1. **Total Bits Needed**:
   Each character in GSM-7 uses strictly 7 bits:
   $$\text{Total Bits} = 12 \text{ characters} \times 7 \text{ bits/char} = \mathbf{84 \text{ bits}}$$
2. **Octet Packaging**:
   Mobile modems pack these 7-bit septets into 8-bit bytes (octets):
   $$\frac{84 \text{ bits}}{8 \text{ bits/octet}} = 10.5 \text{ octets}$$
   Since fractional bytes cannot be transmitted on wire, round up:
   $$\mathbf{\text{Total Octets}} = \lceil 10.5 \rceil = \mathbf{11 \text{ octets}}$$
3. **Padding Bits**:
   Capacity of 11 octets $= 11 \times 8 = 88 \text{ bits}$.
   $$\text{Padding} = 88 \text{ bits} - 84 \text{ bits} = \mathbf{4 \text{ bits of zero-padding}}$$

---

## Question 7: The Emoji Character Collapse
**An engineer sends an SMS containing 75 characters. A colleague adds a single emoji (`🔥`) to the end, making the message 76 characters. How many physical SMS messages will the mobile carrier bill for this single submission?**
- A) 1 SMS
- B) 2 SMS
- C) 3 SMS
- D) 4 SMS

**Correct Answer: B (2 SMS)**

### Step-by-Step Mathematical Explanation:
1. **Standard GSM-7 Limit**:
   $$\text{Max Single SMS Capacity} = 140 \text{ octets} \times 8 \text{ bits} \div 7 = \mathbf{160 \text{ characters}}$$
2. **The Impact of an Emoji**:
   Emojis are not part of the 7-bit GSM alphabet. The modem must switch the `TP-DCS` encoding to **UCS-2 (UTF-16)**, which uses **2 bytes (16 bits) per character**.
3. **New Single-SMS Limit under UCS-2**:
   $$\frac{140 \text{ octets}}{2 \text{ octets/character}} = \mathbf{70 \text{ characters}}$$
4. **Evaluating 76 Characters**:
   Since $76 > 70$, the message exceeds the single-SMS limit. The carrier splits it into **2 SMS segments** (multipart concatenated SMS).

---

## Question 8: Base64 Bandwidth Overhead Math
**An API uploads a 6,000-byte raw binary image encoded in Base64 within an HTTP JSON payload. How many ASCII characters will the encoded string contain (excluding padding), and what is the exact percentage increase in data size?**
- A) 6,000 characters (0% increase)
- B) 8,000 characters (33.33% increase)
- C) 12,000 characters (100% increase)
- D) 7,500 characters (25% increase)

**Correct Answer: B (8,000 characters; 33.33% increase)**

### Step-by-Step Mathematical Explanation:
1. **Base64 Grouping Rule**:
   Base64 groups 3 binary bytes (24 bits) into 4 printable ASCII characters (each carrying 6 bits):
   $$\text{Ratio} = \frac{4 \text{ characters}}{3 \text{ bytes}}$$
2. **Output Size Calculation**:
   $$\text{Output Characters} = \frac{6,000}{3} \times 4 = 2,000 \times 4 = \mathbf{8,000 \text{ characters (bytes)}}$$
3. **Overhead Calculation**:
   $$\text{Overhead} = \frac{8,000 - 6,000}{6,000} = \frac{2,000}{6,000} = \frac{1}{3} \approx \mathbf{33.33\%}$$

---

## Question 9: Total Ethernet Wire Frame Budget
**An application sends a 1,460-byte TCP payload over a standard Ethernet network with an MTU of 1500 bytes. Including standard IPv4, TCP, and Ethernet framing (with FCS), what is the total number of bytes transmitted on the physical wire?**
- A) 1460 bytes
- B) 1500 bytes
- C) 1514 bytes
- D) 1518 bytes

**Correct Answer: D (1518 bytes)**

### Step-by-Step Mathematical Explanation:
1. **Application Payload**: $1,460\text{ bytes}$
2. **Layer 4 (TCP Header)**: $+ 20\text{ bytes}$
   $$\text{TCP Segment Size} = 1460 + 20 = 1480\text{ bytes}$$
3. **Layer 3 (IPv4 Header)**: $+ 20\text{ bytes}$
   $$\text{IP Packet Size (Matches 1500 MTU)} = 1480 + 20 = 1500\text{ bytes}$$
4. **Layer 2 (Ethernet II Framing)**:
   - Destination MAC (6 bytes) + Source MAC (6 bytes) + EtherType (2 bytes) $= 14\text{ bytes}$
   - Frame Check Sequence / CRC-32 (FCS) $= 4\text{ bytes}$
   $$\text{Total Frame On Wire} = 1500 + 14 + 4 = \mathbf{1518 \text{ bytes}}$$

---

## Question 10: Speed of Light in Fiber Optic Latency
**Light travels through optical fiber glass with a refractive index of $n = 1.468$. A cloud ping is measured across an intercity fiber link spanning an exact distance of 2,000 km. What is the theoretical minimum round-trip time (RTT) floor?**
- A) 6.67 ms
- B) 9.80 ms
- C) 19.59 ms
- D) 39.18 ms

**Correct Answer: C (19.59 ms)**

### Step-by-Step Mathematical Explanation:
1. **Speed of Light in Fiber**:
   $$v = \frac{c}{n} = \frac{299,792\text{ km/s}}{1.468} \approx 204,218\text{ km/s}$$
2. **One-Way Latency for 2,000 km**:
   $$t_{\text{one-way}} = \frac{2,000\text{ km}}{204,218\text{ km/s}} \approx 0.0097935\text{ seconds} \approx 9.79\text{ ms}$$
3. **Round-Trip Time Floor (RTT)**:
   $$\text{RTT} = 2 \times t_{\text{one-way}} = 2 \times 9.7935\text{ ms} = \mathbf{19.59 \text{ ms}}$$

---

## Question 11: Ephemeral Port Exhaustion Timeline
**A high-throughput microservice initiates HTTP/1.0 requests to an internal API without connection pooling. The operating system provides 28,000 ephemeral ports. If the client opens and immediately closes 1,400 connections per second, and sockets linger in `TIME_WAIT` for 60 seconds, after how many seconds will the client exhaust its ephemeral port space?**
- A) 60 seconds
- B) 20 seconds
- C) 28 seconds
- D) 14 seconds

**Correct Answer: B (20 seconds)**

### Step-by-Step Mathematical Explanation:
1. Every closed socket enters `TIME_WAIT` for 60 seconds.
2. The client consumes 1,400 ports per second.
3. At time $t$ (for $t \le 60\text{ s}$), total ports occupied in `TIME_WAIT` $= 1,400 \times t$.
4. **Solve for Exhaustion**:
   $$1,400 \times t = 28,000$$
   $$\mathbf{t = \frac{28,000}{1,400} = 20 \text{ seconds}}$$
- After just 20 seconds, all 28,000 ephemeral ports are tied up in `TIME_WAIT`, and subsequent connections fail with `EADDRNOTAVAIL` (*"Cannot assign requested address"*).

---

## Question 12: NGINX Worker Balancing Arithmetic
**In NGINX, a worker process has a connection capacity of 512 slots (`connection_n = 512`). It currently has 50 free connection slots remaining (`free_connection_n = 50`). According to NGINX's load balancing formula:**
$$\text{ngx\_accept\_disabled} = \frac{\text{ngx\_cycle->connection\_n}}{8} - \text{ngx\_cycle->free\_connection\_n}$$
**What is the value of `ngx_accept_disabled`, and what action does the worker take?**
- A) $-14$; the worker continues accepting new connections aggressively.
- B) $+14$; the worker temporarily stops accepting new connections to shed load to sibling workers.
- C) $0$; the worker is perfectly balanced.
- D) $+64$; the worker terminates immediately.

**Correct Answer: B (+14; stops accepting new connections)**

### Step-by-Step Mathematical Explanation:
1. Compute the 1/8th threshold:
   $$\frac{\text{connection\_n}}{8} = \frac{512}{8} = 64$$
2. Apply the formula:
   $$\text{ngx\_accept\_disabled} = 64 - \text{free\_connection\_n} = 64 - 50 = \mathbf{+14}$$
3. **Behavioral Impact**:
   Because `ngx_accept_disabled > 0` (utilization exceeds 87.5%), the worker skips acquiring the accept mutex and stops accepting new connections, giving sibling workers the opportunity to handle incoming traffic.

---

## Question 13: `fd_set` Bitmap Array Size in C
**On a 64-bit Linux system, `glibc` defines `FD_SETSIZE` as 1,024 bits. If `fd_set` is implemented internally as an array of `unsigned long int` (`sizeof(unsigned long) == 8 bytes`), how many elements are in this internal array?**
- A) 1024 elements
- B) 128 elements
- C) 16 elements
- D) 64 elements

**Correct Answer: C (16 elements)**

### Step-by-Step Mathematical Explanation:
1. Total bits required $= 1024\text{ bits}$.
2. Bits per `unsigned long int`:
   $$8 \text{ bytes} \times 8 \text{ bits/byte} = \mathbf{64 \text{ bits per element}}$$
3. **Array Length Calculation**:
   $$\text{Array Size} = \frac{1024 \text{ bits}}{64 \text{ bits/element}} = \mathbf{16 \text{ elements}}$$
   *(In glibc headers: `typedef struct { unsigned long fds_bits[1024 / 64]; } fd_set;`)*

---

## Question 14: Unix Process Exit Code for `SIGPIPE`
**A backend C server attempts a second `write()` to a socket after the client abruptly sent an RST. Because the server did not set `signal(SIGPIPE, SIG_IGN)`, the kernel kills the process with `SIGPIPE` (Signal 13). What exit code does the Linux bash shell report when running `echo $?` immediately after the crash?**
- A) 13
- B) 1
- C) 141
- D) 255

**Correct Answer: C (141)**

### Step-by-Step Mathematical Explanation:
1. In standard POSIX shells, when a command process terminates due to an unhandled signal $N$, the exit status reported is:
   $$\mathbf{\text{Exit Code}} = \mathbf{128 + N}$$
2. For `SIGPIPE` (Signal 13):
   $$\text{Exit Code} = 128 + 13 = \mathbf{141}$$
   *(Similarly: SIGINT (2) $\rightarrow 130$; SIGKILL (9) $\rightarrow 137$; SIGTERM (15) $\rightarrow 143$).*

---

## Question 15: Memory Footprint: OS Threads vs. Goroutines
**An engineering team benchmarks two concurrency models to support 20,000 concurrent idle TCP connections. Model A uses native Linux threads with the default 8 MB stack. Model B uses Go Goroutines with their initial 2 KB dynamic stack. How much memory is reserved by Model A vs. Model B?**
- A) Model A: 160 GB; Model B: 40 MB
- B) Model A: 16 GB; Model B: 400 MB
- C) Model A: 20 GB; Model B: 20 MB
- D) Model A: 80 GB; Model B: 20 MB

**Correct Answer: A (Model A: 160 GB; Model B: 40 MB)**

### Step-by-Step Mathematical Explanation:
1. **Model A (Linux OS Threads)**:
   $$\text{Memory} = 20,000 \times 8 \text{ MB} = \mathbf{160,000 \text{ MB}} = \mathbf{160 \text{ GB of virtual memory}}$$
2. **Model B (Go Goroutines)**:
   $$\text{Memory} = 20,000 \times 2 \text{ KB} = 40,000 \text{ KB} = \mathbf{40 \text{ MB of RAM}}$$
   Goroutines require **4,000 times less memory**, illustrating how event-driven $M:N$ user-space schedulers solve the C10K memory wall.

---

## Question 16: Return Values of the `fork()` System Call
**When a parent process calls `fork()`, what does the system call return to the parent, to the child, and in the case of failure?**
- A) Parent: 0; Child: PID of parent; Failure: -1
- B) Parent: PID of new child ($> 0$); Child: 0; Failure: -1
- C) Parent: 1; Child: 0; Failure: NULL
- D) Parent: PID of child; Child: PID of child; Failure: 0

**Correct Answer: B**  
**Explanation:** `fork()` returns twice on success:
1. In the **child process**, it returns `0` so the child knows it is the newly spawned process.
2. In the **parent process**, it returns the **child's PID** ($> 0$) so the parent can track or wait on the child.
3. On failure (e.g. process limit exceeded), it returns `-1`.

---

## Question 17: The `listen(server_fd, 1)` Parameter
**In the system call `listen(server_fd, 1);`, what does the second argument (`1`) represent?**
- A) The maximum number of concurrent clients the server can serve over its lifetime.
- B) The maximum number of active open file descriptors.
- C) The depth of the kernel Accept Queue (completed 3-way handshakes waiting for `accept()`).
- D) The number of worker processes to spawn.

**Correct Answer: C**  
**Explanation:** The backlog argument defines the maximum length of the kernel's queue for fully established connections that have completed the 3-way handshake but have not yet been retrieved by the application via `accept()`. It is not a ceiling on total open client connections.

---

## Question 18: Skipping System Calls: `bind()` vs. `accept()`
**If a server executes `socket()`, `bind()`, and `listen()`, but enters an infinite sleep loop without ever calling `accept()`, what happens when a client attempts to connect?**
- A) The client's `connect()` call immediately fails with `ECONNREFUSED`.
- B) The client's operating system raises `SIGPIPE`.
- C) The OS kernel completes the TCP 3-way handshake on behalf of the server; the connection is established in the kernel Accept Queue, and the client hangs waiting for application data.
- D) The client's SYN packet is dropped by the server's network card.

**Correct Answer: C**  
**Explanation:** The TCP handshake is handled entirely by the operating system kernel. Even if user-space code never calls `accept()`, the kernel accepts the connection, responds with SYN-ACK, completes the handshake, and parks the socket in the Accept Queue.

---

## Question 19: TCP Reno/CUBIC Sawtooth Multiplicative Decrease
**Under loss-based TCP congestion control (Reno/CUBIC), what happens to the sender's Congestion Window ($CWND$) when a packet drop occurs?**
- A) It drops to 0 packets immediately.
- B) It decreases additively by 1 packet.
- C) It halves ($CWND / 2$), creating a characteristic sawtooth curve.
- D) It doubles to force packet transmission.

**Correct Answer: C**  
**Explanation:** Loss-based TCP uses Additive Increase Multiplicative Decrease (AIMD). The congestion window grows linearly until a packet drops, at which point the sender cuts the window in half ($CWND / 2$).

---

## Question 20: Privileged Ports and the Root Boundary
**Why do modern server architectures bind web applications to port 8080 as an unprivileged user behind an NGINX reverse proxy bound to port 80/443?**
- A) Port 80 does not support SSL/TLS encryption.
- B) Binding to ports below 1024 requires administrative (root) privileges; running complex application logic as root means a code vulnerability or parser bug leads to a full host root compromise.
- C) The Linux kernel restricts port 80 to single-threaded processes.
- D) Port 8080 transfers bytes faster than port 80.

**Correct Answer: B**  
**Explanation:** Following the Principle of Least Privilege, binding $< 1024$ requires root. Placing a minimal, hardened proxy on 80/443 that drops privileges protects the host if an application running on an unprivileged port (8080) is compromised.
