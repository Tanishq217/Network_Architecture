# Practice Quiz Bank: Lecture 04 Real Exam & Advanced Practice

> **Coverage**: 
> - **Section 1**: Complete Deconstruction of the **Official Lecture 04 Quiz** (All 31 real classroom exam questions with exact student statistics, correct answers, mathematical calculations, and trap explanations).
> - **Section 2**: Advanced High-Yield Practice Bank (15 tricky quantitative and systems engineering questions with step-by-step arithmetic and architectural analysis).

---

# Section 1: Official In-Class Lecture 04 Quiz (Fully Deconstructed)

---

### Question 1: Socket Family Identification
**What does `AF_INET` represent in a `socket()` system call?**
- A) IP socket *(Selected in Class: 76%)*
- B) TCP socket *(Selected in Class: 18%)*
- C) UDP socket *(Selected in Class: 0%)*
- D) Unix socket *(Selected in Class: 6%)*

**Correct Answer: A (IP socket / IPv4 Address Family)**

#### Deep Technical Explanation:
- The `socket()` syscall prototype is:
  ```c
  int socket(int domain, int type, int protocol);
  ```
- **Domain (`AF_INET`)**: `AF` stands for **Address Family**. `AF_INET` designates the **IPv4 Internet protocol family** (32-bit addresses). For IPv6, the constant is `AF_INET6`. For local inter-process communication on the same machine via the filesystem, it is `AF_UNIX` (or `AF_LOCAL`).
- **Why Option B is a trap**: `AF_INET` does **not** specify TCP! It only specifies the network layer (IP). The transport protocol (TCP vs UDP) is decided by the second argument: `type` (`SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP).

---

### Question 2: Socket Type Identification
**What does `SOCK_STREAM` represent in a `socket` system call?**
- A) TCP stream *(Selected in Class: 83%)*
- B) UDP stream *(Selected in Class: 11%)*
- C) Raw stream *(Selected in Class: 6%)*
- D) IP stream *(Selected in Class: 0%)*

**Correct Answer: A (TCP stream)**

#### Deep Technical Explanation:
- `SOCK_STREAM` requests a sequenced, reliable, two-way, connection-based byte stream. In the `AF_INET` family, this maps directly to **TCP (Transmission Control Protocol)**.
- For UDP (connectionless, unreliable datagrams with fixed message boundaries), the type argument is `SOCK_DGRAM`.
- For raw access to IP headers or ICMP packets without transport encapsulation, the type is `SOCK_RAW`.

---

### Question 3: The `INADDR_ANY` Client Misconception
**Can I use `INADDR_ANY` in client code to connect to a server?**
- A) Yes *(Selected in Class: 27%)*
- B) No *(Selected in Class: 73%)*

**Correct Answer: B (No)**

#### Deep Technical Explanation:
- `INADDR_ANY` evaluates to the IPv4 address `0.0.0.0`.
- In **server code**, binding a socket to `INADDR_ANY` instructs the operating system kernel: *"Listen for incoming packets on all available local network interfaces (Ethernet, Wi-Fi, loopback 127.0.0.1) on the specified port."*
- In **client code**, when calling `connect(sockfd, &serv_addr, sizeof(serv_addr))`, the client must supply the **explicit, routable IP address** of the remote destination host. If a client attempts to `connect()` to `0.0.0.0`, the kernel either returns an error (`ECONNREFUSED` / `ENETUNREACH`) or routes it locally to loopback; it cannot magically discover which server on the network you intend to communicate with.

---

### Question 4: Byte Swapping with `htons`
**What would be the output of `htons(2026)` on a little-endian machine?**
- A) 2026 *(Selected in Class: 6%)*
- B) 59911 *(Selected in Class: 91%)*
- C) 10000 *(Selected in Class: 0%)*
- D) 2620 *(Selected in Class: 4%)*

**Correct Answer: B (59911)**

#### Step-by-Step Mathematical Calculation:
1. **Convert the Decimal Port `2026` to Hexadecimal**:
   $$\lfloor 2026 / 256 \rfloor = 7 \quad (\text{High Byte})$$
   $$2026 - (7 \times 256) = 2026 - 1792 = 234_{10} = \text{0xEA} \quad (\text{Low Byte})$$
   $$\implies 2026_{10} = \mathbf{\text{0x07EA}}$$

2. **Little-Endian Memory Layout (Host Byte Order on x86/ARM)**:
   - Byte 0 (Least Significant Byte, LSB): `0xEA` (value $234$)
   - Byte 1 (Most Significant Byte, MSB): `0x07` (value $7$)

3. **Function `htons()` (Host to Network Short)**:
   - Network byte order is strictly **Big-Endian** (Most Significant Byte first).
   - On a little-endian architecture, `htons()` swaps the two bytes:
     $$\text{Swapped Hex} = \mathbf{\text{0xEA07}}$$

4. **Convert `0xEA07` back to Decimal**:
   $$\text{High Byte} = \text{0xEA}_{16} = (14 \times 16) + 10 = 234_{10}$$
   $$\text{Low Byte} = \text{0x07}_{16} = 7_{10}$$
   $$\text{Decimal Value} = (234 \times 256) + 7 = 59,904 + 7 = \mathbf{59,911}$$

---

### Question 5: Port Number Upper Limit
**Can I bind on port 70000?**
- A) Yes *(Selected in Class: 15%)*
- B) No *(Selected in Class: 85%)*

**Correct Answer: B (No)**

#### Step-by-Step Mathematical Explanation:
1. **Transport Layer Header Bit Width**:
   - Both TCP and UDP header specifications allocate strictly **16 bits** for the Source Port and **16 bits** for the Destination Port.
2. **Maximum Representable Value in 16 Bits**:
   $$\text{Max Port} = 2^{16} - 1 = 65,536 - 1 = \mathbf{65,535}$$
3. Any attempt to pass `70000` to a socket function will cause integer truncation or overflow, and the kernel will reject it (`EINVAL`).

---

### Question 6: Kernel Modification for Invalid Ports
**Can I change OS kernel to connect to port 70000?**
- A) Yes *(Selected in Class: 16%)*
- B) No *(Selected in Class: 84%)*

**Correct Answer: B (No)**

#### Deep Technical Explanation:
- Even if you recompile your own Linux kernel to store 32-bit port numbers internally, you **cannot violate the RFC 793 TCP protocol wire format**.
- A TCP segment transmitted across the physical wire has exactly 2 bytes (16 bits) reserved for the destination port.
- Any remote server or intermediate router executing standard networking code would interpret only 16 bits of the port, connecting to:
  $$70,000 \pmod{65,536} = 70,000 - 65,536 = \mathbf{4,464}$$
- The remote server would never receive a connection on "port 70000".

---

### Question 7: Origin of the 65,535 Limit
**Where does the 65535 limit come from?**
- A) Linux operating system limitation *(Selected in Class: 19%)*
- B) It's a soft limit *(Selected in Class: 6%)*
- C) It comes from TCP *(Selected in Class: 60%)*
- D) It comes from IP *(Selected in Class: 15%)*

**Correct Answer: C (It comes from TCP / Layer 4 Transport protocols)**

#### Concept & Trap Breakdown:
- **Why Option D is wrong**: The Internet Protocol (IP) at Layer 3 does **not** know what a port is! IP headers contain 32-bit (IPv4) or 128-bit (IPv6) Source and Destination Host Addresses, TTL, and Protocol ID. Ports are an abstraction created purely by Layer 4 (TCP and UDP) to achieve process multiplexing.
- **Why Option A is wrong**: Operating systems follow RFC specifications. The 16-bit port width is an architectural invariant embedded into the binary protocol format.

---

### Question 8: Subsetting Listening IP Addresses
**If I have 3 IP addresses, can I listen on only two of them with a single socket?**
- A) Yes *(Selected in Class: 22%)*
- B) No *(Selected in Class: 78%)*

**Correct Answer: B (No)**

#### Deep Technical Explanation:
- The BSD socket API `bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)` allows binding a socket file descriptor to:
  1. A **single specific IP address** (e.g., `192.168.1.10`), OR
  2. **`INADDR_ANY` (`0.0.0.0`)**, which instructs the kernel to accept packets arriving on **all** interfaces.
- The POSIX socket API does **not** support passing an array, mask, or arbitrary list of 2 out of 3 addresses to a single socket.
- **The Architectural Solution**: To serve requests on exactly 2 out of 3 IPs, an application must create **two independent sockets**, `bind()` each socket to its respective IP, and monitor both file descriptors concurrently using an event loop (`epoll`, `kqueue`, or `select`).

---

### Question 9: Privileged Port Boundary
**What's the minimum IP port I can bind to as a non-root user?**
- A) 1023 *(Selected in Class: 7%)*
- B) 1024 *(Selected in Class: 87%)*
- C) 65535 *(Selected in Class: 7%)*
- D) 65536 *(Selected in Class: 0%)*

**Correct Answer: B (1024)**

#### Deep Technical Explanation:
- **Privileged / Well-Known Ports**: Ports in the range **$0 \le \text{Port} \le 1023$** require superuser (`root`) privileges or the `CAP_NET_BIND_SERVICE` Linux capability.
- **Non-Privileged Ports**: The first port available to standard, unprivileged user accounts is strictly **1024** (up to $65,535$).
- Binding to port 80 or 443 without root credentials results in error code `EACCES` (*Permission denied*).

---

### Questions 10 & 11: POSIX Signal Numeric Codes
**What is the code for `SIGKILL`?**
- A) 15 *(Selected in Class: 7%)*
- B) 9 *(Selected in Class: 86%)*
- C) 2 *(Selected in Class: 5%)*
- D) 13 *(Selected in Class: 2%)*

**Correct Answer: B (9)**

#### Essential POSIX Signal Codes Summary:
| Signal Number | Signal Constant | Default Action | Can Be Caught / Blocked? | Real-World Network Context |
| :--- | :--- | :--- | :--- | :--- |
| **`1`** | `SIGHUP` | Terminate | Yes | Triggers NGINX config reload without downtime |
| **`2`** | `SIGINT` | Terminate | Yes | User pressed `Ctrl+C` in terminal |
| **`9`** | **`SIGKILL`** | **Force Kill** | **NO (Kernel level)** | Immediate termination; cannot clean up sockets or buffers |
| **`13`** | `SIGPIPE` | Terminate | Yes | Writing to a closed/broken TCP socket (`RST` received) |
| **`15`** | `SIGTERM` | Terminate | Yes | Graceful termination signal (allows server cleanup) |

---

### Question 12: Standard HTTPS Port
**What is the conventional port number for HTTPS production servers?**
- A) 80 *(Selected in Class: 9%)*
- B) 8080 *(Selected in Class: 13%)*
- C) 443 *(Selected in Class: 76%)*
- D) 110 *(Selected in Class: 2%)*

**Correct Answer: C (443)**

#### Well-Known Protocol Ports Quick-Reference:
- **Port 80**: Cleartext HTTP (RFC 9110)
- **Port 443**: Encrypted HTTPS over TLS (RFC 8446)
- **Port 8080**: Alternative HTTP / Development Proxy port
- **Port 110**: POP3 (Post Office Protocol v3)
- **Port 143**: IMAP (Internet Message Access Protocol)

---

### Question 13: Protocol Layer Separation of Concerns
**Is port a TCP concern or IP?**
- A) TCP *(Selected in Class: 72%)*
- B) IP *(Selected in Class: 28%)*

**Correct Answer: A (TCP)**

#### Deep Technical Explanation:
- **Layer 3 (IP)**: Resolves **Host-to-Host** addressing. It knows how to deliver a packet from Machine A's IP address to Machine B's IP address across routers and networks.
- **Layer 4 (TCP / UDP)**: Resolves **Process-to-Process** multiplexing. A machine runs hundreds of independent applications simultaneously (database, web server, SSH, DNS). Ports exist solely to map incoming transport segments to the specific operating system process listening on that socket.

---

### Question 14: High-Concurrency Service Strategies
**What options do we have for serving many clients at once?**
- A) `fork` *(Selected in Class: 85%)*
- B) Larger listen queue *(Selected in Class: 15%)*

**Correct Answer: A (`fork`)**

#### Concept & Trap Breakdown:
- **Why Option B is a fatal trap**: The listen backlog queue (`listen(sockfd, backlog)`) does **not** process or serve requests! It is merely a FIFO buffer maintained by the operating system kernel holding connections that completed the 3-way handshake while waiting for an idle worker to invoke `accept()`.
- Increasing the queue from 128 to 10,000 does not serve any client; it only delays the inevitable timeout while memory is consumed.
- To actually serve concurrent clients, an application must employ **concurrency models**: process creation (`fork`), multi-threading (`pthread_create`), or non-blocking event loops (`epoll`/`kqueue`).

---

### Question 15: Return Value Semantics of `fork()`
**If the return value of `fork()` is zero (0), are you the:**
- A) Parent process *(Selected in Class: 35%)*
- B) Child process *(Selected in Class: 65%)*

**Correct Answer: B (Child process)**

#### Deep Technical Explanation:
- The `fork()` system call creates an identical clone of the calling process and returns **twice**:
  ```c
  pid_t pid = fork();
  if (pid < 0) {
      // Fork failed (out of memory / PID limit)
  } else if (pid == 0) {
      // Returned 0: WE ARE THE CHILD PROCESS
      // Child can find parent PID using getppid()
  } else {
      // Returned > 0: WE ARE THE PARENT PROCESS
      // 'pid' contains the PID of the newly spawned child
  }
  ```

---

### Question 16: Parent Process `fork()` Return Value
**What is the significance of `fork()`'s return value in the parent process?**
- A) Number of child *(Selected in Class: 8%)*
- B) Child's port *(Selected in Class: 4%)*
- C) Child's PID *(Selected in Class: 81%)*
- D) Number of accepted connections *(Selected in Class: 8%)*

**Correct Answer: C (Child's PID)**

#### Deep Technical Explanation:
- A parent process may spawn hundreds of child processes, but it has no implicit way of knowing which PID the operating system assigned to the child. Therefore, `fork()` returns the **Process ID (PID)** of the new child to the parent so the parent can manage, monitor, or reap it via `waitpid()`.

---

### Question 17: File Descriptor Inheritance Across `fork()`
**When a connection is accepted and a process is forked, how does the child process get access to the accepted file descriptor?**
- A) Pipes *(Selected in Class: 13%)*
- B) Everything is copied during fork *(Selected in Class: 80%)*
- C) Environment variable *(Selected in Class: 4%)*
- D) Program arguments *(Selected in Class: 2%)*

**Correct Answer: B (Everything is copied during fork / Inherited File Descriptor Table)**

#### Deep Technical Explanation:
- When `fork()` is invoked, the kernel duplicates the parent's entire execution context, including virtual memory page tables (using Copy-on-Write) and the **process File Descriptor (FD) table**.
- If the parent held an open socket at FD `4`, the child inherits FD `4`, which points to the **exact same underlying `struct file` entry in the kernel's open file table**.
- The kernel increments the reference count of the socket. Both parent and child can write to and read from it unless one explicitly calls `close(fd)`.

---

### Question 18: Purpose of `SO_REUSEADDR`
**Why do we use `SO_REUSEADDR`?**
- A) So that read/write works across fork *(Selected in Class: 28%)*
- B) So that servers can be restarted immediately *(Selected in Class: 70%)*
- C) It's unused *(Selected in Class: 2%)*

**Correct Answer: B (So that servers can be restarted immediately)**

#### Deep Technical Explanation:
- When a server initiates an active close on a TCP connection, the socket enters the mandatory **`TIME_WAIT`** state for a duration of $2 \times \text{MSL}$ (Maximum Segment Lifetime, typically 60 to 120 seconds).
- Without `SO_REUSEADDR`, if the server process crashes or is restarted, calling `bind()` on the same local port fails immediately with:
  `EADDRINUSE (Address already in use)`
- Enabling `SO_REUSEADDR` via `setsockopt()` informs the kernel:
  ```c
  int opt = 1;
  setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
  ```
  *"Allow this socket to bind to a local port even if existing sockets on that port are currently lingering in the `TIME_WAIT` state."*

---

### Question 19: `tcpdump` on Encrypted Traffic
**Can I use `tcpdump` for SSL/TLS connections?**
- A) Yes *(Selected in Class: 65%)*
- B) No *(Selected in Class: 11%)*
- C) Yes, but what's the point? *(Selected in Class: 24%)*

**Correct Answer: C (Yes, but what's the point?)**

#### Concept & Trap Breakdown:
- **Can it capture packets?**: Yes. `tcpdump` operates at Layer 2/3 (using `libpcap` and `AF_PACKET` raw sockets). It captures every TCP segment regardless of payload contents.
- **Why "what's the point?" is the chosen answer**: Because TLS encrypts the application layer payload with symmetric ciphers (AES-GCM, ChaCha20). Unless you possess the pre-master secret session keys (e.g. via `SSLKEYLOGFILE`), `tcpdump` displays solely unreadable ciphertext bytes. You can inspect the IP/TCP headers, packet timings, and the cleartext SNI in the TLS ClientHello, but not the actual HTTP request/response data.

---

### Question 20: Frame Boundary Detection in TCP Headers
**In TCP headers, how does the receiver know the header is completely received?**
- A) They are fixed length *(Selected in Class: 54%)*
- B) They are delimited *(Selected in Class: 13%)*
- C) Type and length are sent in advance *(Selected in Class: 34%)*

**Correct Answer: A (They are fixed length / Fixed offset with 4-bit Data Offset)**

#### Deep Technical Explanation:
- A base TCP header has a strictly **fixed size of 20 bytes**.
- The 4-bit **Data Offset** field (Header Length) specifies the total length of the header in 32-bit (4-byte) words:
  $$\text{Header Length in Bytes} = \text{Data Offset Value} \times 4$$
- The minimum value is `5` ($5 \times 4 = 20\text{ bytes}$), and the maximum is `15` ($15 \times 4 = 60\text{ bytes}$ with options).
- The receiver does **not** scan for delimiter strings like `\r\n` and does **not** receive an external length prefix; the binary structure is self-describing and fixed by offset.

---

### Question 21: Delimiting Headers from Body in HTTP
**In HTTP, how is the header and body separated?**
- A) Fixed length *(Selected in Class: 6%)*
- B) Delimited *(Selected in Class: 76%)*
- C) Type length prefix *(Selected in Class: 19%)*

**Correct Answer: B (Delimited)**

#### Deep Technical Explanation:
- In HTTP/1.0 and HTTP/1.1 (RFC 9112), headers are ASCII key-value pairs separated by Carriage Return Line Feeds (`\r\n`).
- The transition between the end of headers and the beginning of the message body is marked strictly by an empty line:
  $$\mathbf{\backslash r \backslash n \backslash r \backslash n \quad (\text{Hex: } \text{0x0D 0x0A 0x0D 0x0A})}$$
- This is a textbook example of a **delimited framing protocol**.

---

### Question 22: Length-Prefixed Protocol Identification
**Which of these protocols uses a length prefix?**
- A) TCP *(Selected in Class: 20%)*
- B) HTTP *(Selected in Class: 33%)*
- C) IP *(Selected in Class: 47%)*

**Correct Answer: B (HTTP - via `Content-Length: <N>`)**

#### Concept & Trap Breakdown:
- While the IP header features a 16-bit "Total Length" field, in application-level network programming, **HTTP** explicitly switches between framing techniques:
  1. Delimited headers (`\r\n\r\n`).
  2. **Length-prefixed payload body**: Defined by the `Content-Length: <bytes>` header. The receiver reads headers until the delimiter, parses the integer length prefix, and issues a blocking read for exactly that quantity of body octets!

---

### Question 23: Binary Coded Decimal (BCD)
**What is the number 2026 written in Binary Coded Decimal (BCD)?**
- A) 2026 *(Selected in Class: 45%)*
- B) 07EA *(Selected in Class: 38%)*
- C) 2620 *(Selected in Class: 17%)*

**Correct Answer: A (2026)**

#### Step-by-Step Mathematical Calculation:
1. **Definition of BCD (Binary Coded Decimal)**:
   In packed BCD, each decimal digit ($0$ through $9$) is encoded as an independent 4-bit nibble:
   $$\text{'2'} \implies 0010_2$$
   $$\text{'0'} \implies 0000_2$$
   $$\text{'2'} \implies 0010_2$$
   $$\text{'6'} \implies 0110_2$$

2. **Concatenating the Nibbles**:
   $$\text{Binary Stream} = 0010 \; 0000 \; 0010 \; 0110_2$$

3. **Interpreting as Hexadecimal**:
   $$\text{Nibble 1} = 0010_2 = \mathbf{2}_{16}$$
   $$\text{Nibble 2} = 0000_2 = \mathbf{0}_{16}$$
   $$\text{Nibble 3} = 0010_2 = \mathbf{2}_{16}$$
   $$\text{Nibble 4} = 0110_2 = \mathbf{6}_{16}$$
   $$\implies \mathbf{\text{0x2026}}$$

4. **The Trap Breakdown**:
   - 38% of students converted $2026_{10}$ to pure binary/hexadecimal: $2026_{10} = \text{0x07EA}$. That is **pure binary**, NOT BCD!
   - In BCD, reading the hex representation yields the **literal decimal digits: `2026`**!

---

### Question 24: JWT Token Inspection
**What is the email/identity payload in this JWT?**
`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6ImplZXRAc2NhbGVyLmNvbSIsImlhdCI6MTUxNjIzOTAyMn0.K8AnNYS797KEmNNzJf00kGTpCBZA508aIwlGSKlt5Is`
- A) `abhimanyu@scaler.com` *(Selected in Class: 13%)*
- B) `jeet@scaler.com` *(Selected in Class: 82%)*
- C) `jeet@lsip.com` *(Selected in Class: 5%)*

**Correct Answer: B (`jeet@scaler.com`)**

#### Step-by-Step Decoding:
1. **JWT Architecture**: A JSON Web Token consists of 3 dot-separated parts: `Header.Payload.Signature`.
2. **Extracting Payload Segment**:
   `eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6ImplZXRAc2NhbGVyLmNvbSIsImlhdCI6MTUxNjIzOTAyMn0=`
3. **Base64URL Decoding**:
   Decoding this string yields the JSON dictionary:
   ```json
   {
     "sub": "1234567890",
     "name": "jeet@scaler.com",
     "iat": 1516239022
   }
   ```
4. Hence, the embedded identity email is **`jeet@scaler.com`**.

---

### Question 25: Base64 String Decoding
**What is the Base64 decode for `"aGVsbG8="`?**
- A) hello *(Selected in Class: 81%)*
- B) bye *(Selected in Class: 10%)*
- C) hello world *(Selected in Class: 10%)*

**Correct Answer: A (hello)**

#### Step-by-Step Mathematical Calculation:
1. Base64 index mapping:
   - `a` $= 26 = 011010_2$
   - `G` $= 6 = 000110_2$
   - `V` $= 21 = 010101_2$
   - `s` $= 44 = 101100_2$
   - `b` $= 27 = 011011_2$
   - `G` $= 6 = 000110_2$
   - `8` $= 60 = 111100_2$
   - `=` $=$ padding
2. Concatenating 6-bit chunks and regrouping into 8-bit octets:
   - $01101000_2 = 104_{10} = \mathbf{\text{'h'}}$
   - $01100101_2 = 101_{10} = \mathbf{\text{'e'}}$
   - $01101100_2 = 108_{10} = \mathbf{\text{'l'}}$
   - $01101100_2 = 108_{10} = \mathbf{\text{'l'}}$
   - $01101111_2 = 111_{10} = \mathbf{\text{'o'}}$
3. Result = `"hello"`.

---

### Question 26: Legacy Mail Protocol Ports
**What port does the POP (POP3) protocol use?**
- A) 143 *(Selected in Class: 31%)*
- B) 110 *(Selected in Class: 64%)*
- C) 80 *(Selected in Class: 6%)*

**Correct Answer: B (110)**

#### Concept & Trap Breakdown:
- **Port 110**: POP3 (Post Office Protocol version 3 - simple download and delete).
- **Port 143**: IMAP (Internet Message Access Protocol - server-synchronized folders and state). 31% of students confuse POP3 with IMAP.

---

### Question 27: TCP Teardown Message Count
**TCP connect is a 3-way handshake. How many messages are exchanged for closing a TCP connection?**
- A) 1 *(Selected in Class: 4%)*
- B) 2 *(Selected in Class: 16%)*
- C) 3 *(Selected in Class: 4%)*
- D) 4 *(Selected in Class: 76%)*

**Correct Answer: D (4 messages)**

#### Message Sequence Diagram:
$$\begin{array}{ccc}
\mathbf{\text{Endpoint A}} & & \mathbf{\text{Endpoint B}} \\
\text{[ESTABLISHED]} & \xrightarrow{\quad\mathbf{1.\; \text{FIN}}\quad} & \text{[CLOSE\_WAIT]} \\
\text{[FIN\_WAIT\_2]} & \xleftarrow{\quad\mathbf{2.\; \text{ACK}}\quad} & \\
& \xleftarrow{\quad\mathbf{3.\; \text{FIN}}\quad} & \text{[LAST\_ACK]} \\
\text{[TIME\_WAIT]} & \xrightarrow{\quad\mathbf{4.\; \text{ACK}}\quad} & \text{[CLOSED]}
\end{array}$$
Because TCP is **full-duplex**, each transmission direction must be terminated independently with its own FIN and corresponding ACK. Total = **4 messages**.

---

### Question 28: TCP Header Field Ordering
**What is sent first in TCP headers? Source port or Destination port?**
- A) Source port *(Selected in Class: 85%)*
- B) Destination port *(Selected in Class: 15%)*

**Correct Answer: A (Source port)**

#### RFC 793 TCP Header Layout:
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
```
- **Bits 0–15 (Bytes 0–1)**: Source Port.
- **Bits 16–31 (Bytes 2–3)**: Destination Port.

---

### Question 29: SMS Payload Size in SS7 Signaling
**How many bytes is an SMS in the SS7 protocol family?**
- A) 272 bytes *(Selected in Class: 21%)*
- B) 140 bytes *(Selected in Class: 54%)*
- C) 160 bytes *(Selected in Class: 25%)*
- D) Unlimited *(Selected in Class: 0%)*

**Correct Answer: B (140 bytes)**

#### Step-by-Step Mathematical Explanation:
- The Signaling System No. 7 (SS7) Mobile Application Part (MAP) protocol rides on existing signaling channels with a strict maximum user payload limit of **140 octets (140 bytes)**.
- **The Trap**: 25% of students pick 160 bytes. **160 is the character limit**, NOT the byte limit! $160 \text{ characters} \times 7 \text{ bits} = 1120 \text{ bits} = 140 \text{ bytes}$.

---

### Question 30: SMS Character Packing with Limited Alphabet
**If we only allowed small letters and space, how many characters could we send in a 140-byte SMS?**
- A) 160 *(Selected in Class: 44%)*
- B) 140 *(Selected in Class: 24%)*
- C) 220+ *(Selected in Class: 30%)*
- D) 300+ *(Selected in Class: 2%)*

**Correct Answer: C (220+)**

#### Step-by-Step Mathematical Calculation:
1. **Determine the Alphabet Size ($S$)**:
   $$\text{Lowercase letters } ('a'\text{ through }'z') = 26$$
   $$\text{Space character } ('\ ') = 1$$
   $$\text{Total Alphabet Size } S = 26 + 1 = \mathbf{27 \text{ distinct symbols}}$$

2. **Calculate the Minimum Bits Required per Character ($b$)**:
   $$2^4 = 16 < 27 \le 32 = 2^5$$
   $$\implies \mathbf{b = 5 \text{ bits per character}}$$

3. **Calculate Total Available Bits in a 140-byte SMS**:
   $$\text{Total Bits} = 140 \text{ bytes} \times 8 \text{ bits/byte} = \mathbf{1120 \text{ bits}}$$

4. **Calculate Maximum Packed Characters**:
   $$\text{Max Characters} = \left\lfloor \frac{1120 \text{ bits}}{5 \text{ bits/character}} \right\rfloor = \mathbf{224 \text{ characters}}$$
   $$224 \text{ is in the range of } \mathbf{220+!}$$

---

### Question 31: SYN Flood Attack Mitigation
**How did we solve the SYN flooding attack problem?**
- A) We bought boats *(Selected in Class: 0%)*
- B) SYN cookies *(Selected in Class: 91%)*
- C) Trackers to carry people out *(Selected in Class: 4%)*
- D) They are unsolvable *(Selected in Class: 5%)*

**Correct Answer: B (SYN cookies)**

#### Deep Technical Explanation:
- In a classic SYN flood, an attacker sends thousands of spoofed TCP SYN packets, filling the server's SYN backlog queue (Half-Open connections) and exhausting kernel memory.
- **SYN Cookies Solution (RFC 4987)**: The server allocates **zero state/memory** in the kernel upon receiving a SYN. Instead, it encodes the connection parameters (MSS, timestamp, secret key hash) directly into the 32-bit initial sequence number ($ISN$). When the legitimate client sends the final `ACK`, the server extracts the sequence number, validates the cryptographic cookie, and only then allocates the socket control block.

---

### Question 32: Base64 Overhead Calculation
**How many bytes would a 300-byte image take when transferred using Base64 encoding?**
- A) 200 *(Selected in Class: 6%)*
- B) 300 *(Selected in Class: 10%)*
- C) 400 *(Selected in Class: 73%)*
- D) 600 *(Selected in Class: 12%)*

**Correct Answer: C (400 bytes)**

#### Step-by-Step Mathematical Calculation:
1. **Base64 Expansion Ratio Formula**:
   Base64 encodes every 3 binary input bytes into 4 ASCII output characters:
   $$\text{Output Size} = 4 \times \left\lceil \frac{N}{3} \right\rceil$$
2. **Applying to 300 Bytes**:
   $$\text{Output Size} = 4 \times \frac{300}{3} = 4 \times 100 = \mathbf{400 \text{ bytes}}$$
   $$\text{Overhead} = \frac{400 - 300}{300} = +33.33\%$$

---

# Section 2: Advanced High-Yield Practice Bank (With Step-by-Step Math)

---

### Question 33: Little's Law Queue Capacity Calculation
**A high-scale API gateway experiences a sustained arrival rate ($\lambda$) of $12,500$ requests per second. Each request requires an average server processing latency ($W$) of $80\text{ ms}$. Under steady-state conditions, how many concurrent requests are being processed in the system ($L$)?**
- A) 1,000
- B) 100
- C) 10,000
- D) 15,625

**Correct Answer: A (1,000 concurrent requests)**

#### Step-by-Step Mathematical Calculation:
1. **Formula (Little's Law)**:
   $$L = \lambda \times W$$
2. **Standardize Units to Seconds**:
   $$\lambda = 12,500 \text{ req/sec}$$
   $$W = 80 \text{ ms} = \frac{80}{1000} \text{ s} = 0.08 \text{ seconds}$$
3. **Compute System Concurrency ($L$)**:
   $$L = 12,500 \times 0.08 = \mathbf{1,000 \text{ concurrent requests}}$$
- **Trap**: Forgetting to convert milliseconds to seconds ($12,500 \times 80 = 1,000,000$).

---

### Question 34: SMS 6-Bit Alphanumeric Packing
**Suppose an engineering team creates a custom SMS packing protocol that supports 64 characters (uppercase letters, lowercase letters, digits, space, and period). In a standard 140-byte cellular SMS payload, what is the exact maximum number of characters that can be packed?**
- A) 160 characters
- B) 186 characters
- C) 140 characters
- D) 224 characters

**Correct Answer: B (186 characters)**

#### Step-by-Step Mathematical Calculation:
1. **Bits per character for 64 symbols**:
   $$2^b \ge 64 \implies 2^6 = 64 \implies \mathbf{b = 6 \text{ bits}}$$
2. **Total available bits in 140 bytes**:
   $$\text{Bits} = 140 \times 8 = \mathbf{1120 \text{ bits}}$$
3. **Compute character count**:
   $$\text{Max Characters} = \left\lfloor \frac{1120}{6} \right\rfloor = \left\lfloor 186.666... \right\rfloor = \mathbf{186 \text{ characters}}$$
4. **Remaining Unused Bits**:
   $$1120 - (186 \times 6) = 1120 - 1116 = \mathbf{4 \text{ bits of padding}}.$$

---

### Question 35: TCP Window Scaling Factor Math
**In a TCP header, the Window Size field is 16 bits wide ($0 \text{ to } 65,535$ bytes). If the TCP Window Scale option (RFC 7323) is negotiated with a shift count of $S = 7$, what is the maximum effective window size achievable by setting the 16-bit field to its maximum value ($65,535$)?**
- A) 65,535 bytes (~64 KB)
- B) 458,745 bytes (~448 KB)
- C) 8,388,480 bytes (~8.38 MB)
- D) 1,073,741,824 bytes (~1 GB)

**Correct Answer: C (8,388,480 bytes / ~8.38 MB)**

#### Step-by-Step Mathematical Calculation:
1. **Window Scale Formula**:
   $$\text{Effective Window} = \text{Window Header Value} \ll S = \text{Window Header Value} \times 2^S$$
2. **Evaluate for $S = 7$**:
   $$2^7 = 128$$
3. **Multiply by Maximum 16-bit Window**:
   $$\text{Effective Window} = 65,535 \times 128 = \mathbf{8,388,480 \text{ bytes}} \approx \mathbf{8.38 \text{ MB}}$$
- **Trap**: Multiplying $65535 \times 7 = 458,745$ (Option B). Bit shifting by $S$ means multiplying by $2^S$, NOT by $S$!

---

### Question 36: Base64 Padding Character Count
**A binary payload has a size of $500$ bytes. When encoded using Base64, what is the exact number of output characters produced, and how many `'='` padding characters will appear at the end?**
- A) 668 characters with two `'='` padding characters
- B) 668 characters with one `'='` padding character
- C) 666 characters with zero padding characters
- D) 664 characters with two `'='` padding characters

**Correct Answer: A (668 characters with two '=' padding characters)**

#### Step-by-Step Mathematical Calculation:
1. **Input modulo 3 evaluation**:
   $$500 \pmod 3 = 2$$
   When $N \pmod 3 = 1 \implies$ **two** `'='` padding characters are appended.
   When $N \pmod 3 = 2 \implies$ **one** `'='` padding character is appended.
   Wait! Let's check:
   - 1 input byte (8 bits) $\implies$ $8$ bits $\to$ two 6-bit Base64 blocks + 4 padding zero bits $\implies$ 2 output chars + 2 `'='` = 4 chars.
   - 2 input bytes (16 bits) $\implies$ $16$ bits $\to$ three 6-bit Base64 blocks + 2 padding zero bits $\implies$ 3 output chars + 1 `'='` = 4 chars.
   Since $500 \div 3 = 166 \text{ with remainder } 2$:
   There are 2 bytes remaining in the final group $\implies$ **one** `'='` padding character!
2. **Total Encoded Length Formula**:
   $$\text{Length} = 4 \times \left\lceil \frac{500}{3} \right\rceil = 4 \times 167 = \mathbf{668 \text{ characters}}$$
   $$\text{Padding: exactly } \mathbf{1 \text{ equal sign ('=')}}.$$

*(Corrected answer matches Option B: 668 characters with one '=' padding character).*

---

### Question 37: Endianness Conversion of Port 80
**What is the decimal output of `htons(80)` when executed on a little-endian x86_64 CPU?**
- A) 80
- B) 20480
- C) 160
- D) 40960

**Correct Answer: B (20480)**

#### Step-by-Step Mathematical Calculation:
1. **Convert 80 to Hexadecimal**:
   $$80_{10} = \mathbf{\text{0x0050}}$$
   - MSB = `0x00`
   - LSB = `0x50`
2. **Little-Endian byte swap (`htons`)**:
   $$\text{Swapped Hex} = \mathbf{\text{0x5000}}$$
3. **Convert `0x5000` to Decimal**:
   $$\text{0x50}_{16} = (5 \times 16) + 0 = 80_{10}$$
   $$\text{Decimal} = 80 \times 256 = \mathbf{20,480}$$

---

### Question 38: TCP Header Length from 4-bit Data Offset
**A packet capture inspection reveals that the 4-bit Data Offset field in a TCP segment header contains the binary value `1001` ($9_{10}$). What is the total length of this TCP header, and how many bytes of TCP options are present?**
- A) 36 bytes total; 16 bytes of options
- B) 9 bytes total; 0 bytes of options
- C) 18 bytes total; 0 bytes of options
- D) 36 bytes total; 36 bytes of options

**Correct Answer: A (36 bytes total; 16 bytes of options)**

#### Step-by-Step Mathematical Calculation:
1. **Total TCP Header Size Formula**:
   $$\text{Total Header Length} = \text{Data Offset} \times 4 \text{ bytes}$$
   $$\text{Total Header Length} = 9 \times 4 = \mathbf{36 \text{ bytes}}$$
2. **Base Header Size**:
   $$\text{Standard Base TCP Header} = \mathbf{20 \text{ bytes}}$$
3. **Options Length**:
   $$\text{Options Size} = \text{Total} - \text{Base} = 36 - 20 = \mathbf{16 \text{ bytes}}$$

---

### Question 39: Bandwidth-Delay Product (BDP) & Buffer Sizing
**A transcontinental fiber optic connection has a bandwidth of $10\text{ Gbps}$ and a round-trip time (RTT) of $80\text{ ms}$. What is the Bandwidth-Delay Product (BDP) in Megabytes (MB), representing the minimum TCP socket buffer size required to fully saturate the link?**
- A) 800 MB
- B) 100 MB
- C) 12.5 MB
- D) 1,000 MB

**Correct Answer: B (100 MB)**

#### Step-by-Step Mathematical Calculation:
1. **Formula**:
   $$\text{BDP (bits)} = \text{Bandwidth (bps)} \times \text{RTT (seconds)}$$
2. **Substitute Values**:
   $$\text{Bandwidth} = 10 \times 10^9 \text{ bits/second}$$
   $$\text{RTT} = 0.080 \text{ seconds}$$
   $$\text{BDP (bits)} = 10^{10} \times 0.08 = 800,000,000 \text{ bits}$$
3. **Convert to Megabytes (MB)**:
   $$\text{BDP (Bytes)} = \frac{800,000,000}{8} = 100,000,000 \text{ Bytes} = \mathbf{100 \text{ MB}}$$
- **Trap**: Forgetting to divide by 8 bits per byte yields 800 MB (Option A).

---

### Question 40: IPv4 Subnet Usable Host Calculation
**A network engineer is configuring an isolated backend microservice subnet with the CIDR block `10.240.16.0/28`. How many usable IP addresses can be assigned to compute hosts within this subnet?**
- A) 16
- B) 14
- C) 28
- D) 30

**Correct Answer: B (14 usable hosts)**

#### Step-by-Step Mathematical Calculation:
1. **Host bits ($h$)**:
   $$h = 32 - \text{Prefix} = 32 - 28 = 4 \text{ bits}$$
2. **Total Addresses**:
   $$2^h = 2^4 = 16 \text{ total addresses}$$
3. **Reserved Addresses (Network & Broadcast)**:
   - First address (`10.240.16.0`): Subnet Network ID
   - Last address (`10.240.16.15`): Subnet Directed Broadcast
4. **Usable Host Addresses**:
   $$\text{Usable Hosts} = 2^h - 2 = 16 - 2 = \mathbf{14 \text{ hosts}}$$

---

### Question 41: NGINX `ngx_accept_disabled` Throttling
**An NGINX worker process is configured with `worker_connections 2048`. During a traffic spike, the worker currently has $128$ free connections remaining (`free_connection_n = 128`). According to the NGINX core throttling formula, what is the value of `ngx_accept_disabled`?**
- A) -128
- B) 0
- C) +128
- D) +256

**Correct Answer: C (+128)**

#### Step-by-Step Mathematical Calculation:
1. **Official NGINX Formula**:
   $$\text{ngx\_accept\_disabled} = \left( \frac{\text{connection\_n}}{8} \right) - \text{free\_connection\_n}$$
2. **Substitute Parameters**:
   $$\text{connection\_n} = 2048$$
   $$\frac{2048}{8} = 256$$
   $$\text{free\_connection\_n} = 128$$
3. **Evaluate**:
   $$\text{ngx\_accept\_disabled} = 256 - 128 = \mathbf{+128}$$
4. **Behavioral Implication**: Because the value is positive ($> 0$), the worker temporarily removes its listening socket from the `epoll` interest set, allowing peer workers to accept incoming traffic until its load decreases.

---

### Question 42: BCD Representation of Year 1999
**How is the decimal number `1999` represented in packed Binary Coded Decimal (BCD)?**
- A) `0x07CF`
- B) `0x1999`
- C) `1999` decimal
- D) `0x9991`

**Correct Answer: B (`0x1999`)**

#### Step-by-Step Calculation:
- Digit 1: $0001_2 = \text{0x1}$
- Digit 9: $1001_2 = \text{0x9}$
- Digit 9: $1001_2 = \text{0x9}$
- Digit 9: $1001_2 = \text{0x9}$
- Combined Hex: `0x1999`. Pure binary conversion would be $1999_{10} = \text{0x07CF}$ (the trap!).

---

### Question 43: Zero-Copy Performance via `sendfile()`
**When serving a 1 GB static video file over HTTP on Linux, how many kernel-to-user-space data copies and how many context switches occur using `sendfile()` compared to traditional `read()` + `write()` loops?**
- A) Traditional: 2 copies, 2 switches; `sendfile()`: 1 copy, 1 switch
- B) Traditional: 2 CPU copies, 4 switches; `sendfile()`: 0 CPU copies, 2 switches
- C) Traditional: 4 copies, 4 switches; `sendfile()`: 4 copies, 2 switches
- D) `sendfile()` eliminates the socket write buffer completely

**Correct Answer: B (Traditional: 2 CPU copies, 4 switches; `sendfile()`: 0 CPU copies, 2 switches)**

#### Deep Technical Breakdown:
- **`read()` + `write()`**:
  1. `read()` syscall: User $\to$ Kernel context switch (1). DMA copies disk $\to$ page cache (0 CPU). CPU copies page cache $\to$ user buffer (**CPU Copy 1**). Switch back to user (2).
  2. `write()` syscall: User $\to$ Kernel context switch (3). CPU copies user buffer $\to$ socket buffer (**CPU Copy 2**). Switch back to user (4). DMA copies socket buffer $\to$ NIC (0 CPU).
  - Total: **4 context switches, 2 CPU copies**.
- **`sendfile()`**:
  1. Single syscall invocation: User $\to$ Kernel context switch (1).
  2. Data transfers inside kernel space directly from page cache to socket descriptor via SG-DMA (Scatter-Gather Direct Memory Access).
  3. Context switch back to user (2).
  - Total: **2 context switches, 0 CPU copies**.

---

### Question 44: Mathis Throughput Equation
**According to the Mathis formula for TCP throughput over lossy links, if the Maximum Segment Size (MSS) is $1460\text{ bytes}$, the round trip time (RTT) is $50\text{ ms}$, and the packet loss probability ($p$) is $1\%$ ($0.01$), what is the maximum theoretical TCP throughput?**
$$\text{Throughput} \le \frac{\text{MSS}}{\text{RTT} \times \sqrt{p}} \times C \quad (C \approx 1.22)$$
- A) ~2.85 Mbps
- B) ~28.5 Mbps
- C) ~1 Gbps
- D) ~285 Kbps

**Correct Answer: A (~2.85 Mbps)**

#### Step-by-Step Mathematical Calculation:
1. **Parameters**:
   $$\text{MSS} = 1460 \text{ bytes} = 1460 \times 8 \text{ bits} = 11,680 \text{ bits}$$
   $$\text{RTT} = 0.050 \text{ seconds}$$
   $$p = 0.01 \implies \sqrt{p} = \sqrt{0.01} = 0.1$$
2. **Denominator**:
   $$\text{Denominator} = \text{RTT} \times \sqrt{p} = 0.050 \times 0.1 = 0.005 \text{ seconds}$$
3. **Calculate Throughput**:
   $$\text{Throughput} \approx \frac{11,680 \text{ bits}}{0.005} \times 1.22 \approx 2,336,000 \times 1.22 \approx \mathbf{2,850,000 \text{ bps}} \approx \mathbf{2.85 \text{ Mbps}}$$
- **Significance**: Even on a 10 Gbps fiber line, a 1% packet loss over 50 ms limits TCP CUBIC single-stream throughput to under 3 Mbps!

---

### Question 45: `select()` vs `epoll()` Scalability
**Why does `epoll` scale to $100,000$ concurrent connections while `select()` degrades exponentially past $1,024$ connections?**
- A) `select()` is implemented in userspace; `epoll` runs in hardware
- B) `select()` requires passing and linear-scanning an entire $O(N)$ bitmask on every tick; `epoll` uses an in-kernel Red-Black tree and returns only the $O(k)$ ready descriptors via a shared ready list
- C) `epoll` uses multi-threading automatically; `select()` is single-threaded
- D) `select()` only supports UDP sockets

**Correct Answer: B (`select()` $O(N)$ linear scan vs `epoll` $O(k)$ event readiness)**

#### Deep Technical Breakdown:
- `select()` relies on `fd_set`, a fixed-width bitmap bounded by `FD_SETSIZE` (1024 on Linux). Every call to `select()` copies the entire bitmap between user space and kernel space, and the kernel must iterate over all $N$ descriptors to check state.
- `epoll` maintains monitored sockets inside an in-kernel **Red-Black tree** (`epoll_ctl`). Sockets register hardware callbacks; when data arrives, the kernel prepends only the active file descriptor to a **Doubly Linked Ready List**. `epoll_wait()` returns in $O(\text{ready})$ time without scanning idle sockets.

---

### Question 46: TCP Handshake Backlog Allocation
**What happens to an incoming TCP connection request when a server's complete listen backlog (both SYN backlog and Accept backlog) is completely full?**
- A) The server sends a TCP FIN packet immediately
- B) The server kernel silently drops the SYN packet so the client retransmits via exponential backoff
- C) The server OS automatically terminates the process with SIGPIPE
- D) The client connection is converted to UDP

**Correct Answer: B (The server kernel silently drops the SYN packet)**

#### Deep Technical Breakdown:
- When the listen queue is exhausted, sending a `RST` would cause the client to immediately abort with "Connection Refused".
- Instead, the Linux kernel **silently drops the incoming SYN**.
- This forces the client's TCP stack to retransmit the SYN after an exponential backoff timer ($1\text{s}, 3\text{s}, 6\text{s}\dots$), giving the server application time to catch up and call `accept()` without failing client transactions.

---

### Question 47: Signal Handling for Graceful Server Teardown
**Which signal should an orchestration system (like Kubernetes or systemd) send to an NGINX master process to trigger a graceful shutdown where active requests are allowed to complete?**
- A) `SIGKILL` (Signal 9)
- B) `SIGQUIT` (Signal 3)
- C) `SIGINT` (Signal 2)
- D) `SIGSEGV` (Signal 11)

**Correct Answer: B (`SIGQUIT` / Signal 3)**

#### Signal Mapping for NGINX:
- **`SIGQUIT` (3)**: Graceful shutdown (workers stop accepting new connections, finish existing requests, and exit).
- **`SIGTERM` / `SIGINT` (15 / 2)**: Fast / Immediate shutdown (abruptly closes all active client connections).
- **`SIGHUP` (1)**: Configuration reload (spawns new workers with updated configuration and gracefully drains old workers).
