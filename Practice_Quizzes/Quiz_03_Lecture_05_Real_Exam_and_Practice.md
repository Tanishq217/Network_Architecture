# Practice Quiz Bank: Lecture 05 Real Exam & Advanced Practice

> **Coverage**: 
> - **Section 1**: Complete Deconstruction of the **Official Lecture 05 Quiz** (All 6 real classroom exam questions with exact student percentage statistics, correct answers, deep architectural analysis, and step-by-step bit-level mathematical derivations).
> - **Section 2**: Advanced High-Yield Practice Bank (15 tricky quantitative and systems engineering questions featuring FTP port calculations, GSM-7 septet bit shifting, TCP Reno AIMD math, BBR congestion control, L2 hop-by-hop framing, and file descriptor lifecycle mechanics).

---

# Section 1: Official In-Class Lecture 05 Quiz (Fully Deconstructed)

---

### Question 1: Unclosed Socket File Descriptors & Process Exit
**What happens if you don't call the `close()` system call in your server / client?**
- A) Kernel cleans it up *(Selected in Class: 48%)*
- B) Memory leak *(Selected in Class: 48%)*
- C) `SIGPIPE` *(Selected in Class: 4%)*

**Correct Answer: A (Kernel cleans it up)**

#### Deep Technical Explanation:
- **The Process Termination Guarantee**: In POSIX and Linux operating systems, when a process terminates (whether via `exit()`, `return 0` from `main()`, or receiving a fatal signal like `SIGKILL`), the OS kernel invokes its internal cleanup routine (`do_exit()` / `exit_files()`).
- During this teardown, the kernel traverses the terminating process's **File Descriptor Table** and calls the internal `filp_close()` on every active file descriptor, including all open TCP sockets.
- For TCP sockets, the kernel automatically initiates the graceful connection termination sequence by sending a **`FIN` packet** to the remote peer.
- **Why Option B ("Memory leak") trapped 48% of students**: 
  - If a **long-running server process** continuously accepts connections inside a loop and forgets to call `close(client_fd)` *while the process stays alive*, the open socket file descriptors will accumulate, leaking kernel socket buffers (`struct sock`) and eventually causing `EMFILE` (*Too many open files*).
  - However, the question asks what happens to the resources when the program finishes/runs without explicit close: **the operating system kernel guarantees complete reclamation of all open FDs and kernel buffers upon process exit**. No process-allocated memory can leak past kernel process teardown.

---

### Question 2: Original TCP Congestion Control Signals
**What mechanism does the original TCP algorithm use for deciding on congestion?**
- A) Packet loss *(Selected in Class: 76%)*
- B) Round trip propagation time *(Selected in Class: 24%)*

**Correct Answer: A (Packet loss)**

#### Deep Technical Explanation:
- The foundational TCP congestion control algorithms formulated by Van Jacobson in 1988 (**TCP Tahoe** and **TCP Reno**, RFC 2581) were based strictly on **Packet Loss**.
- They treated any lost segment—detected via a **Retransmission Timeout (RTO)** or the receipt of **3 Duplicate ACKs**—as an unambiguous signal that intermediate router buffers had overflowed due to network congestion. Upon detecting loss, they drastically slashed the Congestion Window ($cwnd$).
- **Why Option B is modern, not original**: Using Round-Trip Time / propagation delay as the primary congestion metric was introduced much later:
  - Conceptually by **TCP Vegas** (1994), and
  - Formally perfected in production by Google's **BBR (Bottleneck Bandwidth and RTT)** in 2016.
  Original TCP completely ignored RTT variations when adjusting window sizes, which led to the modern phenomenon of **Bufferbloat**.

---

### Question 3: Mitigating SYN Flood Attacks
**How does the TCP protocol defend against SYN flood attacks?**
- A) TLS *(Selected in Class: 9%)*
- B) HTTPS *(Selected in Class: 3%)*
- C) SYN cookies *(Selected in Class: 88%)*

**Correct Answer: C (SYN cookies)**

#### Deep Technical Explanation:
- In a classic SYN flood attack, a malicious actor transmits high volumes of spoofed `SYN` packets without completing the 3-way handshake. The victim server allocates a Transmission Control Block (TCB) in memory and holds the socket in `SYN_RECV` state inside the half-open SYN backlog queue until memory is exhausted.
- **SYN Cookies (RFC 4987)** solve this by making the server **entirely stateless** during the initial handshake:
  1. The server receives the `SYN` but allocates **zero bytes** of kernel memory.
  2. The server synthesizes the 32-bit Initial Sequence Number ($ISN$) as an encrypted hash:
     $$ISN = \text{HMAC}(\text{SrcIP}, \text{DstIP}, \text{SrcPort}, \text{DstPort}, \text{SecretKey}) + \text{MSS Index} + t$$
  3. The client responds with an `ACK` containing `ack_seq = ISN + 1`.
  4. The server subtracts 1, recomputes the HMAC, verifies that the cryptographic cookie is authentic, and only then allocates socket memory.
- TLS and HTTPS operate at Layer 5–7 and have zero ability to protect the transport layer against TCP SYN exhaustion.

---

### Question 4: FTP Passive Mode (PASV) Port Calculation
**For FTP, a PASV response of `(127,0,0,1, 48, 117)` results in what port number?**
- A) 30002 *(Selected in Class: 16%)*
- B) 12405 *(Selected in Class: 76%)*
- C) 30000 *(Selected in Class: 8%)*

**Correct Answer: B (12405)**

#### Step-by-Step Mathematical Calculation:
1. **The FTP Passive Mode (RFC 959) Specification**:
   In Passive Mode (PASV), the FTP server opens a random high-numbered data port and replies to the client over the control connection (Port 21) with a 6-tuple:
   $$\mathbf{(h1, h2, h3, h4, p1, p2)}$$
   - $h_1, h_2, h_3, h_4$ form the 32-bit IPv4 address: `127.0.0.1`.
   - $p_1$ and $p_2$ are two 8-bit integers representing the 16-bit TCP port number in **Network Byte Order (Big-Endian)**.

2. **The Port Calculation Formula**:
   $$\mathbf{\text{Port}} = (\mathbf{p_1} \times 256) + \mathbf{p_2} = (\mathbf{p_1} \ll 8) + \mathbf{p_2}$$

3. **Plug in the Given Values ($p_1 = 48$, $p_2 = 117$)**:
   $$\text{Port} = (48 \times 256) + 117$$
   $$48 \times 256 = 12,288$$
   $$\text{Port} = 12,288 + 117 = \mathbf{12,405}$$

4. **Binary & Hexadecimal Verification**:
   - $p_1 = 48_{10} = \text{0x30}$
   - $p_2 = 117_{10} = \text{0x75}$
   - Concatenated 16-bit Hex: $\text{0x3075}$
   - $\text{0x3075}_{16} = (3 \times 4096) + (0 \times 256) + (7 \times 16) + 5 = 12,288 + 0 + 112 + 5 = \mathbf{12,405}_{10}$.

---

### Question 5: Layer 2 Hop-by-Hop Destination MAC Address
**A laptop in Bengaluru sends a packet to a web server in Frankfurt. What appears in the destination MAC field of the Ethernet frame leaving the host's NIC?**
- A) Laptop's MAC *(Selected in Class: 5%)*
- B) Web server's MAC *(Selected in Class: 5%)*
- C) Laptop's gateway's MAC *(Selected in Class: 83%)*
- D) Laptop's Wi-Fi router's MAC *(Selected in Class: 7%)*

**Correct Answer: C (Laptop's gateway's MAC)**

#### Deep Technical Explanation:
- **Layer 3 vs Layer 2 Boundary Invariant**:
  - **IP Addresses (Layer 3)** are **end-to-end**. The source IP is the laptop (`Bengaluru`), and the destination IP is the web server (`Frankfurt`). This IP header remains unchanged across intermediate routers.
  - **MAC Addresses (Layer 2)** are **strictly hop-by-hop local**. MAC addresses cannot cross a router boundary.
- **The Routing Mechanism**:
  1. The laptop's operating system checks its routing table and determines that the web server's IP is outside the local subnet.
  2. To reach external networks, the packet must be forwarded to the **Default Gateway**.
  3. The laptop issues an **ARP (Address Resolution Protocol)** request: *"Who has the Gateway IP? Tell me your MAC."*
  4. The Ethernet frame encapsulated by the laptop's NIC contains:
     - **Source MAC**: Laptop's NIC MAC address.
     - **Destination MAC**: **Default Gateway's MAC address**.
- When the gateway receives the frame, it strips the L2 header, checks the destination IP, decrements TTL, and encapsulates the packet in a brand-new L2 frame with the next router's MAC address. The Frankfurt server's MAC address is never known or used by the Bengaluru laptop.

---

### Question 6: GSM-7 SMS Bit-Packing Derivation
**"Hello World!" starts with 'H', which is `0x48` in ASCII. In SMS encoding it starts with `0xC8`. Why?**
- A) Base64 *(Selected in Class: 26%)*
- B) 7-bit encoding *(Selected in Class: 74%)*

**Correct Answer: B (7-bit encoding / GSM 03.38 Septet Packing)**

#### Step-by-Step Mathematical Derivation:
1. **ASCII vs GSM-7 Character Values**:
   - In standard 8-bit ASCII:
     $$\text{'H'} = 72_{10} = \mathbf{0100\,1000_2} = \mathbf{\text{0x48}}$$
     $$\text{'e'} = 101_{10} = \mathbf{0110\,0101_2} = \mathbf{\text{0x65}}$$
   - In the GSM 7-bit default alphabet:
     $$\text{'H'} = 72_{10} = 100\,1000_2 \quad (\text{7 bits})$$
     $$\text{'e'} = 101_{10} = 110\,0101_2 \quad (\text{7 bits})$$

2. **How GSM-7 Packs 7-bit Septets into 8-bit Octets**:
   - Cellular networks packing 7-bit characters into 8-bit octets shift bits across character boundaries:
     $$\text{Octet 1 (8 bits)} = [\text{Bit 0 of Char 2}] \;\|\; [\text{Bits 6..0 of Char 1}]$$
   - Let's look at Character 1 (`'H'`):
     $$\text{Bits 6..0 of 'H'} = \mathbf{100\,1000_2}$$
   - Let's look at Character 2 (`'e'`):
     $$\text{'e'} = 110\,010\mathbf{1}_2$$
     $$\mathbf{\text{Bit 0 (least significant bit) of 'e'}} = \mathbf{1}$$

3. **Constructing Octet 1 on the Wire**:
   - Place Bit 0 of `'e'` into **Bit 7** (MSB) of Octet 1:
     $$\text{Bit 7} = \mathbf{1}$$
   - Place the 7 bits of `'H'` into **Bits 6..0** of Octet 1:
     $$\text{Bits 6..0} = \mathbf{100\,1000}$$
   - Combined 8-bit binary value of Octet 1:
     $$\text{Octet 1} = \mathbf{1}\mathbf{100\,1000_2}$$

4. **Convert `11001000_2` to Hexadecimal**:
   $$\text{High Nibble } 1100_2 = 12_{10} = \mathbf{\text{C}}_{16}$$
   $$\text{Low Nibble } 1000_2 = 8_{10} = \mathbf{8}_{16}$$
   $$\implies \mathbf{\text{0xC8}}$$

- **Conclusion**: In ASCII, Bit 7 is `0`, producing `0x48`. In GSM-7 packed format, the second character (`'e'`) contributes its lowest bit (`1`) into the first byte's most significant bit, turning `01001000` (`0x48`) into `11001000` ($\mathbf{\text{0xC8}}$)!

---

# Section 2: Advanced High-Yield Practice Bank (With Step-by-Step Math)

---

### Question 7: Passive FTP Port Calculation (Arbitrary Port)
**An FTP client issues a `PASV` command to an enterprise backup server. The server responds with `227 Entering Passive Mode (192, 168, 10, 5, 216, 18)`. What TCP port must the client connect to for the incoming data stream?**
- A) 55314
- B) 4634
- C) 21618
- D) 55296

**Correct Answer: A (55314)**

#### Step-by-Step Mathematical Calculation:
1. **Formula**:
   $$\text{Port} = (p_1 \times 256) + p_2$$
2. **Identify $p_1$ and $p_2$**:
   $$p_1 = 216, \quad p_2 = 18$$
3. **Execute the Arithmetic**:
   $$216 \times 256 = 55,296$$
   $$\text{Port} = 55,296 + 18 = \mathbf{55,314}$$
4. **Hexadecimal Verification**:
   - $216_{10} = \text{0xD8}$
   - $18_{10} = \text{0x12}$
   - Hex value = $\text{0xD812}$
   - $\text{0xD812}_{16} = (13 \times 4096) + (8 \times 256) + (1 \times 16) + 2 = 53,248 + 2048 + 16 + 2 = \mathbf{55,314}_{10}$.

---

### Question 8: Second Octet Value in GSM-7 SMS Packing
**Continuing the packing of "Hello World!", character 2 is `'e'` (`1100101_2`) and character 3 is `'l'` (`1101100_2`). What is the exact 8-bit hexadecimal value of the second octet transmitted over the air?**
- A) `0x65`
- B) `0x32`
- C) `0x32`
- D) `0x65`
- E) `0x32`

Wait, let's derive the exact binary:
$$\text{Octet 2} = [\text{Bits 1..0 of Char 3}] \;\|\; [\text{Bits 6..1 of Char 2}]$$
Let's calculate:
- Character 2 (`'e'`): $1100101_2$. Bits 6..1 are: $110010_2$.
- Character 3 (`'l'`): $1101100_2$. Bits 1..0 are: $00_2$.
- Octet 2 binary: $00 \;\|\; 110010_2 = 0011\,0010_2$.
- In Hex: $0011_2 = 3$, $0010_2 = 2 \implies \mathbf{\text{0x32}}$!

**Correct Answer: `0x32`**

#### Step-by-Step Mathematical Calculation:
1. Since bit 0 of `'e'` was consumed by Octet 1, `'e'` has 6 remaining bits: `110010`.
2. Octet 2 needs 8 bits, so it takes $8 - 6 = 2$ bits from character 3 (`'l'`).
3. The lower 2 bits of `'l'` ($1101100_2$) are `00`.
4. Putting them together:
   $$\text{Octet 2} = \mathbf{00}11\,0010_2 = \mathbf{\text{0x32}}$$

---

### Question 9: TCP Reno Congestion Avoidance Math
**A TCP Reno connection has a current congestion window of $cwnd = 32 \text{ MSS}$. It is in the Congestion Avoidance phase. Over the course of the next Round Trip Time (RTT), all 32 segments are successfully acknowledged without loss. What is the new $cwnd$ value at the end of this RTT?**
- A) $64 \text{ MSS}$
- B) $33 \text{ MSS}$
- C) $48 \text{ MSS}$
- D) $32.5 \text{ MSS}$

**Correct Answer: B ($33 \text{ MSS}$)**

#### Step-by-Step Mathematical Calculation:
1. **Slow Start vs Congestion Avoidance Rules**:
   - In **Slow Start**: $cwnd$ doubles every RTT (increases by $1\text{ MSS}$ per ACK received).
   - In **Congestion Avoidance**: $cwnd$ increases **additively** by $1\text{ MSS}$ per RTT (AIMD - Additive Increase Multiplicative Decrease).
2. **Per-ACK Formula in Congestion Avoidance**:
   $$\Delta cwnd = \frac{1}{cwnd} \text{ MSS per received ACK}$$
3. **Total Increase for 32 ACKs**:
   $$\text{Total Increase} = 32 \times \left( \frac{1}{32} \right) = \mathbf{1 \text{ MSS}}$$
   $$\text{New } cwnd = 32 + 1 = \mathbf{33 \text{ MSS}}$$
- **Trap**: Doubling from 32 to 64 MSS (Option A) only occurs in Slow Start ($cwnd < ssthresh$).

---

### Question 10: Multiplicative Decrease on Packet Loss
**A TCP Reno sender with $cwnd = 40 \text{ MSS}$ experiences packet loss detected via 3 Duplicate ACKs (Fast Retransmit). According to TCP Reno rules, what will be the new Slow Start Threshold ($ssthresh$) and new $cwnd$ for Fast Recovery?**
- A) $ssthresh = 20 \text{ MSS}, \quad cwnd = 1 \text{ MSS}$
- B) $ssthresh = 20 \text{ MSS}, \quad cwnd = 20 \text{ MSS}$
- C) $ssthresh = 20 \text{ MSS}, \quad cwnd = 23 \text{ MSS}$
- D) $ssthresh = 10 \text{ MSS}, \quad cwnd = 20 \text{ MSS}$

**Correct Answer: C ($ssthresh = 20 \text{ MSS}, \quad cwnd = 23 \text{ MSS}$)**

#### Step-by-Step Mathematical Calculation:
1. **Compute $ssthresh$**:
   $$ssthresh = \max\left( \frac{cwnd}{2}, 2 \right) = \frac{40}{2} = \mathbf{20 \text{ MSS}}$$
2. **Compute $cwnd$ in Fast Recovery (RFC 5681)**:
   $$cwnd = ssthresh + 3 \text{ MSS}$$
   $$cwnd = 20 + 3 = \mathbf{23 \text{ MSS}}$$
   *(The $+3$ accounts for the 3 packets that successfully left the network and triggered the 3 duplicate ACKs).*
- **Tahoe vs Reno Trap**: Dropping $cwnd$ to $1\text{ MSS}$ (Option A) is **TCP Tahoe** behavior or occurs on full **Retransmission Timeout (RTO)**, NOT on 3 duplicate ACKs in Reno!

---

### Question 11: File Descriptor Limit & Memory Exhaustion
**A backend Linux server has `ulimit -n` set to $2048$. If the server process currently holds $1500$ established TCP connections and fails to call `close()` on completed short-lived connections arriving at $100 \text{ req/sec}$, after how many seconds will new incoming connections start failing with `EMFILE`?**
- A) 5.48 seconds
- B) 20.48 seconds
- C) 54.8 seconds
- D) 15.0 seconds

**Correct Answer: A (5.48 seconds)**

#### Step-by-Step Mathematical Calculation:
1. **Available File Descriptors**:
   $$\text{Max FDs} = 2048$$
   $$\text{Active Used FDs} = 1500$$
   $$\text{Remaining Free FDs} = 2048 - 1500 = \mathbf{548 \text{ FDs}}$$
2. **Leak Rate**:
   $$\text{Leak Rate} = 100 \text{ connections/second}$$
3. **Time to Depletion ($T$)**:
   $$T = \frac{548 \text{ FDs}}{100 \text{ FDs/sec}} = \mathbf{5.48 \text{ seconds}}$$
- Once depleted, any subsequent call to `accept()` or `socket()` immediately returns `-1` with `errno = EMFILE` (*Too many open files*).

---

### Question 12: IP Packet TTL Hop Decrement
**A client host transmits an IPv4 packet initialized with a TTL (Time To Live) of $64$. The packet traverses $12$ intermediate routers to reach its destination server. What is the value of the TTL field inside the IP header when the packet is delivered to the destination server's network stack?**
- A) 64
- B) 52
- C) 51
- D) 0

**Correct Answer: B (52)**

#### Step-by-Step Mathematical Calculation:
1. **RFC 791 Router Invariant**:
   Every Layer 3 router that receives and forwards an IP packet decrements the 8-bit TTL field by strictly **1**.
2. **Calculation**:
   $$\text{Final TTL} = \text{Initial TTL} - \text{Number of Routers}$$
   $$\text{Final TTL} = 64 - 12 = \mathbf{52}$$
- **Significance**: If the TTL reaches `0` before arriving at the destination, the router drops the packet and emits an ICMP Type 11 (*Time Exceeded*) message back to the sender (which is how `traceroute` works).

---

### Question 13: Cellular SMS Packing Compression Ratio
**Standard 8-bit ASCII requires 1 byte per character. What is the theoretical data compression efficiency achieved by the GSM 7-bit packing algorithm compared to raw 8-bit ASCII?**
- A) 12.5% reduction
- B) 25.0% reduction
- C) 14.3% reduction
- D) 33.3% reduction

**Correct Answer: A (12.5% reduction)**

#### Step-by-Step Mathematical Calculation:
1. **Bits per character comparison**:
   $$\text{Standard ASCII} = 8 \text{ bits/character}$$
   $$\text{GSM Septet} = 7 \text{ bits/character}$$
2. **Bits saved per character**:
   $$\text{Savings} = 8 - 7 = 1 \text{ bit}$$
3. **Percentage Savings**:
   $$\text{Efficiency} = \frac{1 \text{ bit}}{8 \text{ bits}} = 0.125 = \mathbf{12.5\% \text{ bandwidth savings}}$$
4. **Resulting Character Capacity Ratio**:
   $$\frac{8}{7} = 1.142857... \implies \mathbf{+14.28\% \text{ more characters in the same 140 bytes}} \quad (140 \times \frac{8}{7} = 160).$$

---

### Question 14: ARP Table Poisoning & Layer 2 Forwarding
**When host A wants to send an Ethernet frame to host B on the same local subnet ($192.168.1.0/24$), host A sends an ARP request. What MAC address is placed in the destination field of that ARP Request Ethernet frame?**
- A) `00:00:00:00:00:00`
- B) Host B's MAC address
- C) `FF:FF:FF:FF:FF:FF`
- D) Default Gateway's MAC address

**Correct Answer: C (`FF:FF:FF:FF:FF:FF` / Broadcast MAC)**

#### Deep Technical Explanation:
- Host A does not yet know Host B's physical hardware address.
- Therefore, the ARP Request frame is addressed to the Ethernet Layer 2 broadcast address: **`FF:FF:FF:FF:FF:FF`** (all 48 bits set to 1).
- The network switch floods this broadcast frame out of all ports except the ingress port. Host B recognizes its own IP inside the payload and unicasts an ARP Reply directly to Host A's MAC address.

---

### Question 15: SYN Cookie Cryptographic Hash Structure
**In Linux SYN Cookie generation, why is the timestamp component restricted to a small number of bits (typically 3 bits) in the 32-bit Initial Sequence Number ($ISN$)?**
- A) Because timestamps are unnecessary in networking
- B) To allow sequence numbers to fit the TCP 32-bit limit while reserving bits for the cryptographic hash and MSS index
- C) Because the system clock only updates every 64 seconds
- D) To prevent TCP sequence number wrapping

**Correct Answer: B (Bit allocation constraints in the 32-bit ISN)**

#### Step-by-Step Bit Allocation:
1. **The 32-bit ISN structure in Linux kernel**:
   - **Bits 0–2 (3 bits)**: Encoded MSS table index (representing one of 8 typical MSS values like 536, 1460, etc.).
   - **Bits 3–7 (5 bits)**: Coarse time counter (increments every 64 seconds, wrapping every 32 minutes to prevent replay attacks).
   - **Bits 8–31 (24 bits)**: Cryptographic secret hash ($HMAC$ of 4-tuple and secret key).
2. The server must fit all these critical validation attributes inside exactly 32 bits without exceeding the standard TCP sequence space.

---

### Question 16: Ethernet Frame Minimum Size & Padding
**The minimum payload for an IEEE 802.3 Ethernet frame is $46 \text{ bytes}$. If a minimal TCP ACK segment (20 bytes IPv4 header + 20 bytes TCP header = 40 bytes) is transmitted, how many bytes of padding must the NIC append to meet the minimum Ethernet frame size?**
- A) 0 bytes
- B) 6 bytes
- C) 24 bytes
- D) 14 bytes

**Correct Answer: B (6 bytes of padding)**

#### Step-by-Step Mathematical Calculation:
1. **Minimum L2 Payload**: $46 \text{ bytes}$.
2. **Actual L3/L4 Segment Size**:
   $$\text{IP Header} (20) + \text{TCP Header} (20) + \text{Data} (0) = \mathbf{40 \text{ bytes}}$$
3. **Required Padding**:
   $$\text{Padding} = 46 - 40 = \mathbf{6 \text{ bytes}}$$
4. **Total Frame on the Wire (including 14-byte Ethernet header + 4-byte FCS)**:
   $$\text{Total Size} = 14 + 46 + 4 = \mathbf{64 \text{ bytes (minimum Ethernet frame size)}}.$$

---

### Question 17: BBR vs CUBIC Congestion Metric Differentiation
**Why does TCP BBR achieve significantly higher throughput and lower latency than TCP CUBIC on high-bandwidth, lossy connections (e.g., satellite or long-distance transatlantic fiber)?**
- A) BBR uses UDP instead of TCP
- B) BBR models the physical network by measuring Bottleneck Bandwidth ($BtlBw$) and Round-Trip Propagation Delay ($RTprop$) instead of misinterpreting non-congestive packet loss as buffer collapse
- C) BBR skips the 3-way handshake entirely
- D) BBR disables TCP checksumming

**Correct Answer: B (Physics-based network modeling vs loss-based heuristics)**

#### Deep Technical Breakdown:
- **CUBIC (Loss-based)**: Assumes any dropped packet equals router queue overflow. On wireless or long-distance links with $1\%$ random noise loss, CUBIC repeatedly halves its window, capping throughput to single-digit Mbps (via the Mathis formula).
- **BBR (Model-based)**: Maintains an internal state model of the link's maximum delivery rate and minimum physical propagation delay. It operates at the **Kleinrock optimal operating point**, keeping inflight data equal to the exact Bandwidth-Delay Product without overfilling buffers or panicking over random loss.

---

### Question 18: TIME_WAIT Socket Table Saturation Rate
**A high-frequency trading proxy performs non-keepalive HTTP/1.0 queries to a local microservice. The OS configured `tcp_fin_timeout` keeps sockets in `TIME_WAIT` for $60 \text{ seconds}$. The ephemeral port range has $30,000$ available ports. At what sustained request rate (req/s) will the client experience ephemeral port exhaustion (`EADDRNOTAVAIL`)?**
- A) 30,000 req/s
- B) 500 req/s
- C) 60 req/s
- D) 1,800,000 req/s

**Correct Answer: B (500 req/s)**

#### Step-by-Step Mathematical Calculation:
1. **Apply Little's Law ($L = \lambda \times W$)**:
   - $L = 30,000 \text{ available ephemeral ports (capacity)}$
   - $W = 60 \text{ seconds (holding duration in TIME\_WAIT)}$
2. **Solve for Maximum Sustained Arrival Rate ($\lambda$)**:
   $$\lambda = \frac{L}{W} = \frac{30,000 \text{ ports}}{60 \text{ seconds}} = \mathbf{500 \text{ requests/second}}$$
- **Significance**: If the proxy issues $> 500$ req/s without HTTP connection pooling (`Keep-Alive`), all 30,000 ephemeral ports will be locked in `TIME_WAIT`, and subsequent `connect()` calls will crash with `EADDRNOTAVAIL`.

---

### Question 19: IP Header Checksum 16-Bit Math
**How is the Checksum field in an IPv4 header computed?**
- A) CRC-32 cyclic redundancy check
- B) 16-bit one's complement sum of all 16-bit words in the header
- C) MD5 cryptographic hash of header and payload
- D) XOR of the source and destination IP addresses

**Correct Answer: B (16-bit one's complement sum)**

#### Technical Verification:
- Per RFC 791, the checksum field is the 16-bit one's complement of the one's complement sum of all 16-bit words in the header.
- Routers must verify and recompute this checksum at **every single hop** because the TTL field is decremented by 1!

---

### Question 20: Socket Inheritance across `fork()` and Reference Counting
**A parent process accepts a TCP connection on file descriptor 5 (`conn_fd = 5`) and forks a child process. The child process handles the connection, sends the HTTP response, and calls `close(5)`. Why does the TCP connection remain open to the client until the parent also calls `close(5)`?**
- A) It is a bug in Linux
- B) Because the underlying kernel socket has an open reference count of 2; `close()` only decrements the count and sends `FIN` when the count drops to 0
- C) Because the child must send `SIGKILL` to the parent first
- D) Because file descriptors cannot be shared across processes

**Correct Answer: B (Kernel socket struct reference counting)**

#### Deep Technical Breakdown:
- Open files and sockets in Linux maintain an internal reference counter in `struct file`.
- When `fork()` is executed, the child inherits the file table, incrementing the socket's reference count from **1 to 2**.
- When the child process calls `close(5)`, the kernel decrements the count from **2 to 1**. The underlying socket remains active in the ESTABLISHED state.
- Only when the parent also invokes `close(5)` (or terminates) does the reference count reach **0**, triggering the transmission of the TCP `FIN` packet.

---

### Question 21: MTU with VLAN Tagging (802.1Q)
**A standard Ethernet LAN has an MTU of $1500 \text{ bytes}$. An enterprise switch injects an IEEE 802.1Q VLAN tag ($4 \text{ bytes}$) into the frame header. If the network hardware does not support baby-giant frames, what is the maximum TCP MSS that can be transmitted without causing IP fragmentation?**
- A) 1460 bytes
- B) 1456 bytes
- C) 1496 bytes
- D) 1500 bytes

**Correct Answer: B (1456 bytes)**

#### Step-by-Step Mathematical Calculation:
1. **Standard MSS Calculation**:
   $$\text{Base MTU} = 1500 \text{ bytes}$$
   $$\text{IPv4 Header} = 20 \text{ bytes}, \quad \text{TCP Header} = 20 \text{ bytes}$$
   $$\text{Standard MSS} = 1500 - 40 = 1460 \text{ bytes}$$
2. **Subtract VLAN Tagging Overhead**:
   $$\text{VLAN Tag Size} = 4 \text{ bytes}$$
   $$\mathbf{\text{Effective MTU}} = 1500 - 4 = 1496 \text{ bytes}$$
   $$\mathbf{\text{Effective MSS}} = 1496 - 40 = \mathbf{1456 \text{ bytes}}$$
