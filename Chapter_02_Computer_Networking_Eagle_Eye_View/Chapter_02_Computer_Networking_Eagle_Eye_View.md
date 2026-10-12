# Chapter 02: Computer Networking Eagle Eye View — Architecture, Stacks & Protocols

> **Course**: Computer Networks (cn-at-scaler)  
> **Session**: Session 2 / Week 2  
> **Core Theme**: *"Seven syscalls make a server. Seven layers of nesting carry eleven bytes. Today we open both ends."*  
> **Key Protocols & Concepts**: OSI Model, Physical Layer Realities (CSMA/CD vs. CSMA/CA, Starlink LEO), SS7 Telecom Stack vs. TCP/IP, The 140-Byte SMS Limit & GSM-7 Packing Trick, Cellular Signaling (MAP, ISUP, 5G SIP/RTP), TCP Handshake & Teardown Mechanics, SYN Cookies & Mitnick Attack, The Round-Trip Latency Tax, BBR vs. Loss-Based Congestion Control, UDP Mechanics, QUIC & HTTP/3, and Text Protocols (SMTP, POP3, IMAP, FTP Active vs. Passive).

---

## Table of Contents
1. [Core Architectural Themes: The Four Invariants](#1-core-architectural-themes-the-four-invariants)
2. [Sockets Recap: The Seven System Calls and Connection Dynamics](#2-sockets-recap-the-seven-system-calls-and-connection-dynamics)
   - [2.1 The Seven Calls and Address Bookkeeping](#21-the-seven-calls-and-address-bookkeeping)
   - [2.2 Client Symmetry and Ephemeral Ports](#22-client-symmetry-and-ephemeral-ports)
   - [2.3 Abortive Teardown: `SO_LINGER` Mechanics](#23-abortive-teardown-so_linger-mechanics)
3. [The OSI 7-Layer Reference Model & Physical Layer Realities](#3-the-osi-7-layer-reference-model--physical-layer-realities)
   - [3.1 Deconstructing the 7 Layers](#31-deconstructing-the-7-layers)
   - [3.2 Wire vs. Radio: CSMA/CD vs. CSMA/CA](#32-wire-vs-radio-csmacd-vs-csmaca)
   - [3.3 Layering in the Real World: Starlink LEO Satellite Constellation](#33-layering-in-the-real-world-starlink-leo-satellite-constellation)
4. [SS7: The Shadow Internet of Telephony](#4-ss7-the-shadow-internet-of-telephony)
   - [4.1 The Great Divide: In-Band vs. Out-of-Band Signaling](#41-the-great-divide-in-band-vs-out-of-band-signaling)
   - [4.2 The SS7 Protocol Stack vs. The TCP/IP Stack](#42-the-ss7-protocol-stack-vs-the-tcpip-stack)
   - [4.3 Packet Encapsulation: The Onion Model](#43-packet-encapsulation-the-onion-model)
   - [4.4 Seven Layers of Nesting to Carry Eleven Bytes: GSM MO-ForwardSM](#44-seven-layers-of-nesting-to-carry-eleven-bytes-gsm-mo-forwardsm)
5. [The Engineering Behind SMS (Short Message Service)](#5-the-engineering-behind-sms-short-message-service)
   - [5.1 Why an SMS is Exactly 140 Bytes](#51-why-an-sms-is-exactly-140-bytes)
   - [5.2 The GSM-7 160-Character Packing Trick](#52-the-gsm-7-160-character-packing-trick)
   - [5.3 The Emoji Trap: UCS-2 / UTF-16 Encoding Collapse](#53-the-emoji-trap-ucs-2--utf-16-encoding-collapse)
   - [5.4 SMS Routing Architecture: Store-and-Forward via SMSC & HLR](#54-sms-routing-architecture-store-and-forward-via-smsc--hlr)
6. [Call Setup Evolution: SS7 ISUP to 5G IMS / SIP](#6-call-setup-evolution-ss7-isup-to-5g-ims--sip)
   - [6.1 Traditional Call Setup: ISUP 4-Message Flow](#61-traditional-call-setup-isup-4-message-flow)
   - [6.2 Modern Voice: 5G / VoLTE via SIP and RTP](#62-modern-voice-5g--volte-via-sip-and-rtp)
   - [6.3 Reliability: Per-Hop (SS7) vs. End-to-End (TCP)](#63-reliability-per-hop-ss7-vs-end-to-end-tcp)
7. [Congestion Control: Loss-Based (Reno/CUBIC) vs. BBR](#7-congestion-control-loss-based-renocubic-vs-bbr)
   - [7.1 The Sawtooth Flaw of Loss-Based TCP](#71-the-sawtooth-flaw-of-loss-based-tcp)
   - [7.2 BBR: Bottleneck Bandwidth and Round-Trip Time](#72-bbr-bottleneck-bandwidth-and-round-trip-time)
8. [Transport Layer Deep Dive: TCP, UDP, and QUIC](#8-transport-layer-deep-dive-tcp-udp-and-quic)
   - [8.1 TCP 3-Way Handshake: Why Three Legs?](#81-tcp-3-way-handshake-why-three-legs)
   - [8.2 Random Initial Sequence Numbers (ISN) & The Mitnick Attack](#82-random-initial-sequence-numbers-isn--the-mitnick-attack)
   - [8.3 The SYN Flood Attack & Stateless SYN Cookies (1996)](#83-the-syn-flood-attack--stateless-syn-cookies-1996)
   - [8.4 TCP Teardown: Why Four Segments? The Half-Close State](#84-tcp-teardown-why-four-segments-the-half-close-state)
   - [8.5 Sockets Stuck in `CLOSE_WAIT` vs. `TIME_WAIT`](#85-sockets-stuck-in-close_wait-vs-time_wait)
   - [8.6 The Round-Trip Latency Tax: TCP, TLS 1.2, TLS 1.3, TFO, QUIC](#86-the-round-trip-latency-tax-tcp-tls-12-tls-13-tfo-quic)
   - [8.7 UDP: Minimalist Transport (8-Byte Header)](#87-udp-minimalist-transport-8-byte-header)
   - [8.8 QUIC & HTTP/3: Defeating Head-of-Line Blocking](#88-quic--http3-defeating-head-of-line-blocking)
9. [Human-Readable Text Protocols: SMTP, POP3, IMAP, FTP](#9-human-readable-text-protocols-smtp-pop3-imap-ftp)
   - [9.1 SMTP: Simple Mail Transfer Protocol & Security Patches](#91-smtp-simple-mail-transfer-protocol--security-patches)
   - [9.2 MIME & Base64: Transporting Binary Data as Text](#92-mime--base64-transporting-binary-data-as-text)
   - [9.3 Mail Retrieval: POP3 vs. IMAP Architecture](#93-mail-retrieval-pop3-vs-imap-architecture)
   - [9.4 The Taxonomy of Protocol Framing: Delimiter, Length, and Both](#94-the-taxonomy-of-protocol-framing-delimiter-length-and-both)
   - [9.5 FTP: Dual-Connection Architecture (Active vs. Passive Mode)](#95-ftp-dual-connection-architecture-active-vs-passive-mode)
   - [9.6 Decoding FTP 227 Port Numbers: Big-Endian Math](#96-decoding-ftp-227-port-numbers-big-endian-math)
10. [Comprehensive Review & Practice MCQs (with Detailed Explanations)](#10-comprehensive-review--practice-mcqs-with-detailed-explanations)
11. [Session 2 Summary Cheat Sheet](#11-session-2-summary-cheat-sheet)

---

## 1. Core Architectural Themes: The Four Invariants

Across computer networking history, four fundamental design principles recur regardless of whether data travels across copper cables, microwave radios, fiber optics, or cellular switches:

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE FOUR INVARIANTS                                    │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. ENCAPSULATION           : One layer's headers are another layer's payload body.     │
│                              (Ethernet ⊃ IP ⊃ TCP ⊃ HTTP)  ==  (MTP2 ⊃ MTP3 ⊃ MAP)     │
│                                                                                        │
│ 2. FRAMING IS BINARY       : There are only two fundamental ways to frame a message:   │
│                              By Length or By Delimiter (or a combination of both).     │
│                                                                                        │
│ 3. RELIABILITY HAS AN      : Where does error recovery live?                           │
│    ADDRESS                   SS7 = Layer 2 (per-hop).                                  │
│                              TCP = Layer 4 (end-to-end).                               │
│                              QUIC = User space over UDP.                               │
│                                                                                        │
│ 4. SEPARATION OF CONTROL   : The control plane (signaling) must never share the same   │
│    AND DATA PLANES           channel as the data plane (media/payload).                │
│                              SS7 killed blue boxes; SIP decouples voice; FTP uses 2 fds│
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Sockets Recap: The Seven System Calls and Connection Dynamics

### 2.1 The Seven Calls and Address Bookkeeping
A TCP server in POSIX C is fundamentally structured around seven system calls:
$$\text{socket} \longrightarrow \text{bind} \longrightarrow \text{listen} \longrightarrow \text{accept} \longrightarrow \text{read} \longrightarrow \text{write} \longrightarrow \text{close}$$

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = INADDR_ANY;        // Listen on all network interfaces
addr.sin_port = htons(2026);              // Convert port 2026 to Network Byte Order (Big-Endian)

bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
listen(server_fd, 1);                     // Backlog queue depth = 1

while (1) {
    int client_fd = accept(server_fd, NULL, NULL); // Extracts completed connection
    char buf[4096];
    int n = read(client_fd, buf, sizeof(buf));
    write(client_fd, buf, n);
    close(client_fd);
}
```

#### Crucial Nuances:
1. **The `listen(server_fd, 1)` Backlog**:
   The second argument (`1`) is **not a limit on total concurrent connections**. It defines the maximum length of the kernel's **Accept Queue** (completed 3-way handshakes waiting for user space to call `accept()`). This distinction is critical when analyzing **SYN Flood Denial-of-Service attacks**.
2. **`htons()` (Host to Network Short)**:
   The physical wire requires **Big-Endian** byte ordering (Most Significant Byte first). Client laptops (x86/ARM) are **Little-Endian** (Least Significant Byte first). Failing to call `htons()` causes the CPU to byte-swap the port number in memory, binding to an unintended port.

---

### 2.2 Client Symmetry and Ephemeral Ports
The TCP client uses a mirrored subset of socket calls:
```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = inet_addr("127.0.0.1");
addr.sin_port = htons(8080);

connect(fd, (struct sockaddr*)&addr, sizeof(addr)); // Triggers the 3-way handshake!
write(fd, "hello", 5);
close(fd);
```

- **Why Clients Omit `bind()` and `listen()`**:
  A server must occupy a deterministic, well-known port so clients can address it. A client does not care what its source port is; the operating system kernel dynamically assigns an **Ephemeral Port** (e.g., in the range `32768–60999` on Linux) during the `connect()` execution.

---

### 2.3 Abortive Teardown: `SO_LINGER` Mechanics

```
Standard Close (Graceful):
close(fd) ───> Sends FIN ───> Waits for ACK ───> Enters TIME_WAIT (2 * MSL = 60s)

Abortive Close (SO_LINGER {1, 0}):
close(fd) ───> Sends RST ───> Discards pending buffers ───> NO TIME_WAIT (Immediate death)
```

- **Graceful Close**: A standard call to `close()` initiates the four-way FIN handshake. It guarantees all in-flight buffers drain cleanly, and the socket endpoint lingers in `TIME_WAIT` for $2 \times \text{MSL}$ (Maximum Segment Lifetime, ~60 seconds) to catch delayed duplicate packets.
- **Abortive Close via `SO_LINGER`**:
  ```c
  struct linger l = {1, 0}; // l_onoff = 1 (enable), l_linger = 0 (timeout = 0 seconds)
  setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
  close(fd);
  ```
  - Discards all unsent and unread data in the socket buffers immediately.
  - Sends an immediate **TCP `RST` (Reset)** segment instead of a polite `FIN`.
  - Bypasses the `TIME_WAIT` state entirely.
  - *Warning*: This is a lossy, non-standard teardown used primarily in testing or to combat malicious connections.

---

## 3. The OSI 7-Layer Reference Model & Physical Layer Realities

Standardized in 1984 by the International Organization for Standardization (ISO), the **Open Systems Interconnection (OSI)** model provides a theoretical framework for network communications.

```
+---+----------------------+---------------------------------+-----------------------------------+
| # | Layer Name           | Primary Responsibility          | Protocols / Standards             |
+---+----------------------+---------------------------------+-----------------------------------+
| 7 | Application Layer    | Human-to-software interface     | HTTP, SMTP, DNS, FTP, POP3, IMAP  |
| 6 | Presentation Layer   | Data syntax, encryption, codec  | TLS, ASN.1, MIME, ASCII, UTF-8    |
| 5 | Session Layer        | Connection/session management   | RPC, NetBIOS, SIP, Sockets API    |
| 4 | Transport Layer      | Process-to-process reliability  | TCP, UDP, QUIC, SCTP              |
| 3 | Network Layer        | Host-to-host packet routing     | IPv4, IPv6, ICMP, BGP, OSPF       |
| 2 | Data Link Layer      | Hop-to-hop frame delivery (MAC) | Ethernet, 802.11 Wi-Fi, ARP, PPP  |
| 1 | Physical Layer       | Raw bit transmission over media | Copper twisted pair, Fiber, Radio |
+---+----------------------+---------------------------------+-----------------------------------+
```

### 3.1 Deconstructing the 7 Layers
- **Layer 4: TCP vs. UDP Trade-offs**:
  - **TCP (Transmission Control Protocol)**: Connection-oriented, strictly ordered, reliable byte stream with flow control (sliding window), congestion control, and error recovery. Incurs connection latency and head-of-line blocking.
  - **UDP (User Datagram Protocol)**: Connectionless, lightweight (8-byte header), best-effort delivery without ordering or retransmission guarantees. Ideal for latency-sensitive streaming, voice, and gaming.

---

### 3.2 Wire vs. Radio: CSMA/CD vs. CSMA/CA

At Layers 1 and 2, physical transmission media dictate radically different collision-handling mechanisms:

```
┌───────────────────────────────────────┬───────────────────────────────────────┐
│     WIRED ETHERNET (802.3)            │          WIRELESS WI-FI (802.11)      │
│     CSMA/CD: Collision Detection      │      CSMA/CA: Collision Avoidance     │
├───────────────────────────────────────┼───────────────────────────────────────┤
│ - Transceiver transmits and listens   │ - Half-duplex radio: Transmit power   │
│   simultaneously on the physical wire.│   drowns out incoming receiver signals│
│ - Detects voltage spikes on collision.│ - CANNOT detect collisions during tx! │
│ - Aborts immediately and sends Jam.   │ - Must actively AVOID collisions.     │
│ - Exponential backoff before retry.   │ - Mandatory explicit L2 ACK per frame.│
└───────────────────────────────────────┴───────────────────────────────────────┘
```

#### Why Wi-Fi Cannot Use CSMA/CD:
Because a radio antenna cannot listen while transmitting—its own output signal is orders of magnitude stronger than any incoming signal from a distant transmitter—it cannot detect an in-flight collision.

#### The 802.11 CSMA/CA Protocol Flow:
1. **Carrier Sense**: Listen to the radio channel.
2. **Channel Busy?**: If someone else is transmitting, defer transmission and wait.
3. **Contention Window & Random Backoff**: Select a random backoff slot counter. Decrement while the medium remains idle.
4. **Transmit Frame**: Transmit the entire 802.11 data frame over the air.
5. **Mandatory L2 ACK**: The receiver must return an explicit **Layer 2 ACK frame** within a Short Inter-Frame Space (SIFS).
6. **No ACK Received?**: Assume a collision occurred, double the contention window (exponential backoff), and retransmit.

> **Performance Penalty**: CSMA/CA introduces approximately **$\mathbf{\approx 2\text{ ms}}$ of contention delay** before a single frame even leaves the antenna. Every single Wi-Fi frame is acknowledged at Layer 2 long before Layer 4 (TCP) is aware of any transmission.

---

### 3.3 Layering in the Real World: Starlink LEO Satellite Constellation

The primary justification for network layering is **abstraction**: higher layers operate without needing to know how lower layers transmit bits.

```
[ Laptop ] 
    │ (Ethernet / Wi-Fi)
[ Starlink Terminal (Phased-Array Dish) ]
    │ (RF Uplink: Ku / Ka-Band microwave)
[ LEO Satellite in Space (~550 km orbit, ~20 ms latency) ]
    │ (Laser Inter-Satellite Crosslink: Optional vacuum optical path)
[ Second LEO Satellite ]
    │ (RF Downlink)
[ Terrestrial Ground Gateway Station ]
    │ (Fiber Optic Backbone)
[ Destination Web Server / The Internet ]
```

- A packet travels across **nine physical hops**, including an ascent into low-earth orbit ($550\text{ km}$ straight up into space).
- Yet, the **IPv4 header (Layer 3)** generated by your laptop arrives at the destination completely unchanged (except for the decremented Time-To-Live `TTL`, updated header checksum, and any intermediate NAT table translations).
- **Layer 3, Layer 4, and Layer 7 never learn that outer space was involved.**

---

## 4. SS7: The Shadow Internet of Telephony

Before the public internet existed, telecommunication carriers built a global, high-speed, packet-switched network: **Signaling System No. 7 (SS7)**, standardized in 1975 by the ITU-T.

```
Every time you place a phone call, receive an SMS, or roam internationally,
you are using the SS7 network.
```

### 4.1 The Great Divide: In-Band vs. Out-of-Band Signaling

```
HISTORICAL IN-BAND SIGNALING (Vulnerable to Phreaking):
┌────────────────────────────────────────────────────────────────────────┐
│ Voice Channel: [ Hello Alice... ] [ 2600 Hz Tone ] [ Dialed MF Digits ]│
└────────────────────────────────────────────────────────────────────────┘
 Anyone who could generate audio tones could control the telephone network.

MODERN OUT-OF-BAND SIGNALING (SS7 / 1975):
┌────────────────────────────────────────────────────────────────────────┐
│ Voice Bearer Channel : 64 kbps DS0 Circuit (Voice conversation ONLY)   │
├────────────────────────────────────────────────────────────────────────┤
│ Signaling Network    : SS7 Packet-Switched Network (MTP/SCCP/MAP/ISUP) │
│                        (Physically separated; caller has NO access)    │
└────────────────────────────────────────────────────────────────────────┘
 Complete decoupling of the Control Plane and Data Plane.
```

#### The Fall of In-Band Signaling:
In early telephone systems, the control signals (seizing a trunk, routing calls, billing) were transmitted as audio tones directly inside the telephone audio band.
- A **$2600\text{ Hz}$ tone** signaled to telephone switches that an inter-office trunk was idle.
- In the 1970s, telephone hackers ("phreakers" like John Draper / Captain Crunch, Steve Jobs, and Steve Wozniak) used **Blue Boxes** to generate a $2600\text{ Hz}$ frequency, seizing trunk lines to place free international calls and control switching gear.

#### The Solution: Out-of-Band Signaling
SS7 separated telephone architecture into two independent networks:
1. **Bearer Channels (Data Plane)**: 64 kbps DS0 circuits dedicated exclusively to digitizing and streaming voice audio.
2. **Signaling Links (Control Plane)**: A distinct, secure, packet-switched network that handles routing, call setup, billing, and SMS delivery. Callers have zero physical access to the signaling network.

---

### 4.2 The SS7 Protocol Stack vs. The TCP/IP Stack

Both SS7 and TCP/IP solve identical networking problems across the 7 OSI layers, but use different protocols and trust models:

| OSI Layer | SS7 Telecom Protocol Stack | Internet TCP/IP Protocol Stack | Core Responsibilities |
| :---: | :--- | :--- | :--- |
| **L7** | **MAP / INAP / CAP / ISUP** | **HTTP / DNS / SMTP / SIP** | Call setup, roaming, SMS, application logic |
| **L5–L6**| **TCAP** (Transaction Capabilities) | **TLS / Application Framing** | ASN.1 BER encoding, component dialogues |
| **L4** | **SCCP** (Signaling Connection Control) | **TCP / UDP** | Subsystem numbers, Global Title Translation |
| **L3** | **MTP3** (Message Transfer Part 3) | **IP** (Internet Protocol) | Point-code routing, network load sharing |
| **L2** | **MTP2** (Message Transfer Part 2) | **Ethernet / PPP / HDLC** | Signal units, sequence numbers, CRC-16 check |
| **L1** | **MTP1** (Message Transfer Part 1) | **Copper / Fiber / Microwave** | 64 kbps DS0 channel on E1 (Europe) / T1 (US) |

> **Key Difference**: SS7 is an entire vertically integrated network stack; TCP is merely a single Layer 4 protocol within the TCP/IP suite.

---

### 4.3 Packet Encapsulation: The Onion Model

```
Packet Encapsulation Hierarchy (Ethernet ⊃ IP ⊃ TCP ⊃ HTTP):

┌──────────────────────────────────────────────────────────────────────────────┐
│ L2 ETHERNET II FRAME                                                         │
│ [ Dst MAC (6B) | Src MAC (6B) | EtherType: 0x0800 (2B) ]                     │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │ L3 IPv4 PACKET (20 Bytes Header)                                         │ │
│ │ [ Ver/IHL (1B) | Total Len (2B) | TTL (1B) | Proto: 6=TCP | IP Addrs(8B)]│ │
│ │ ┌──────────────────────────────────────────────────────────────────────┐ │ │
│ │ │ L4 TCP SEGMENT (20 Bytes Header)                                     │ │ │
│ │ │ [ Src Port (2B) | Dst Port (2B) | Seq (4B) | Ack (4B) | Flags (2B) ] │ │ │
│ │ │ ┌──────────────────────────────────────────────────────────────────┐ │ │ │
│ │ │ │ L7 APPLICATION PAYLOAD                                           │ │ │ │
│ │ │ │ GET / HTTP/1.1\r\nHost: example.com\r\n\r\n                      │ │ │ │
│ │ │ └──────────────────────────────────────────────────────────────────┘ │ │ │
│ │ └──────────────────────────────────────────────────────────────────────┘ │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│ [ Frame Check Sequence: CRC-32 (4 Bytes) ]                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### Wire Byte Budget Breakdown:
$$\text{Standard Ethernet MTU (Maximum Transmission Unit)} = 1500 \text{ Bytes}$$
$$\text{IPv4 Base Header} = 20 \text{ Bytes}$$
$$\text{TCP Base Header} = 20 \text{ Bytes}$$
$$\mathbf{\text{TCP MSS (Maximum Segment Size)}} = 1500 - 20 - 20 = \mathbf{1460 \text{ Bytes}}$$

When transmitted across the physical wire, add the Ethernet framing:
$$\text{Ethernet Header (14 Bytes)} + \text{Payload (1500 Bytes)} + \text{FCS Checksum (4 Bytes)} = \mathbf{1518 \text{ Bytes Total Frame On Wire}}$$

---

### 4.4 Seven Layers of Nesting to Carry Eleven Bytes: GSM MO-ForwardSM

When a mobile phone sends the text message `"Hello World!"`, the telecommunications network nests seven distinct protocol headers:

```
Encapsulation Nesting for Mobile-Originated SMS (MO-ForwardSM):

1. MTP2 MSU     : [ Flag | BSN+BIB | FSN+FIB | LI | CRC-16 ] (SIF cap: 273 octets)
   └─ 2. MTP3   : [ SIO: 0x83 | DPC: 2065 | OPC: 2050 | SLS: 5 ] (129 octets)
      └─ 3. SCCP: [ UDT Msg: 0x09 | Class 1 | Called GT | Calling GT ] (99 octets)
         └─ 4. TCAP: [ BEGIN Tag: 0x62 | OTID: 0x00000001 | ACN = shortMsgMO-Relay ]
            └─ 5. Invoke: [ Tag: 0xA1 | Invoke ID: 1 | opCode: 46 (mo-ForwardSM) ] (47 octets)
               └─ 6. MAP : [ MO-ForwardSM-Arg | sm-RP-DA | sm-RP-OA | sm-RP-UI ] (25 octets)
                  └─ 7. TPDU: [ SMS-SUBMIT | MTI | TP-MR | TP-DCS: 0x00 | TP-UDL: 0x0C ]
                     └─ 8. TP-UD: C8 32 9B FD 06 5D DF 72 36 39 04
                                  ("Hello World!" · 12 chars · 84 bits · 11 octets)
```

The SS7 architecture follows the exact same onion encapsulation principle as `Ethernet ⊃ IP ⊃ TCP ⊃ HTTP`, but caps the total payload at 272 octets rather than 1460 bytes.

---

## 5. The Engineering Behind SMS (Short Message Service)

### 5.1 Why an SMS is Exactly 140 Bytes
The 140-byte limit of an SMS is not an arbitrary product decision; it is a direct consequence of telecom hardware constraints in the 1980s:

```
┌────────────────────────────────────────────────────────┐
│ 272 Octets : MTP2 SIF (Signaling Information Field)    │ <── Hard physical ceiling of one MSU
├────────────────────────────────────────────────────────┤
│ ~200 Octets: MAP sm-RP-UI Field                        │ <── MAP opaque user data allocation
├────────────────────────────────────────────────────────┤
│ 140 Octets : TP-UD (SMS User Data Field)               │ <── The maximum SMS payload
└────────────────────────────────────────────────────────┘
```

#### Zero Segmentation, Zero Reassembly:
- By capping the SMS text payload at **140 octets (1120 bits)**, telecom engineers guaranteed that the entire SMS, including all MAP, TCAP, SCCP, MTP3, and MTP2 signaling headers, **would always fit inside a single 272-octet Message Signal Unit (MSU)**.
- **Why this was crucial**: 1980s cellular switches had minimal memory and CPU capacity. Because an SMS required **zero packet segmentation and zero reassembly**, switches did not need to maintain state buffers.
- This is why SMS functions during major emergencies and network blackouts even when mobile broadband data (4G/5G) collapses.

---

### 5.2 The GSM-7 160-Character Packing Trick
How do phones fit **160 characters** into a **140-byte** payload?

$$\text{Total Available Bits} = 140 \text{ octets} \times 8 \text{ bits/octet} = 1120 \text{ bits}$$
$$\text{Standard ASCII} = 8 \text{ bits per character} \implies 1120 \div 8 = 140 \text{ characters}$$
$$\mathbf{\text{GSM-7 Alphabet}} = \mathbf{7 \text{ bits per character}} \implies 1120 \div 7 = \mathbf{160 \text{ characters}}$$

#### Bit Packing Mechanics:
Instead of aligning characters to 8-bit byte boundaries, 7-bit character chunks ("septets") are concatenated into an unbroken bitstream and re-sliced into 8-bit octets:

```
Char 1 (7 bits) + 1 bit of Char 2  ──> Packed Octet 1
Remaining 6 bits of Char 2 + 2 bits of Char 3 ──> Packed Octet 2
...
12 characters ("Hello World!") × 7 bits = 84 bits ──> Stored in 11 octets (88 bits with 4 bits padding).
```

---

### 5.3 The Emoji Trap: UCS-2 / UTF-16 Encoding Collapse

The moment a user types a single character outside the 128-character GSM-7 alphabet (such as an emoji like 😊, or non-Latin scripts like Devanagari, Arabic, or Chinese):
1. The cellular modem alters the **TP-DCS (Data Coding Scheme)** header field from `0x00` (GSM 7-bit) to `0x08` (**UCS-2 / UTF-16**).
2. UCS-2 requires **2 full bytes (16 bits) per character**.
3. The underlying physical transport buffer remains fixed at **140 octets**.

$$\mathbf{\text{Maximum Message Length}} = 140 \text{ octets} \div 2 \text{ bytes/character} = \mathbf{70 \text{ characters!}}$$

> **Real-World Impact**: Inserting a single emoji drops your SMS character limit from **160 characters down to 70**. A 71-character text with an emoji is silently split into two SMS messages, doubling the transmission cost.

---

### 5.4 SMS Routing Architecture: Store-and-Forward via SMSC & HLR

An SMS is not delivered peer-to-peer; it is a **database lookup wrapped in an asynchronous queue**:

```
Phone A           VMSC / VLR          SMSC               HLR               MSC B          Phone B
  │                   │                │                  │                  │               │
  │── SMS-SUBMIT ────>│                │                  │                  │               │
  │   (140 octets)    │── mo-ForwardSM>│                  │                  │               │
  │                   │<── TCAP END ───│                  │                  │               │
  │                   │   (Accepted)   │                  │                  │               │
  │                   │                │ [STORED IN QUEUE]│                  │               │
  │                   │                │                  │                  │               │
  │                   │                │── sendRouting ──>│                  │               │
  │                   │                │   InfoForSM      │                  │               │
  │                   │                │<─ IMSI + MSC B ──│                  │               │
  │                   │                │                  │                  │               │
  │                   │                │── mt-ForwardSM (SMS-DELIVER)───────>│               │
  │                   │                │                                     │── Over-Air ──>│
  │                   │                │<── Delivery ACK ────────────────────│<── ACK ───────│
  │<── Status Report (Optional) ───────│                                     │               │
```

1. **Submission**: Phone A sends an `SMS-SUBMIT` packet to the Visiting Mobile Switching Center (VMSC).
2. **Queuing**: The VMSC relays the payload to the **SMSC (Short Message Service Center)** via `mo-ForwardSM`. The SMSC acknowledges receipt (`TCAP END`), taking full custody of the message. Phone A is now free to disconnect.
3. **Location Query**: The SMSC queries the recipient's **HLR (Home Location Register)** using `sendRoutingInfoForSM(MSISDN)` to locate Phone B.
4. **Resolution**: The HLR returns Phone B's internal **IMSI** and the IP/Point-Code of the switch currently serving Phone B (**MSC B**).
5. **Delivery**: The SMSC forwards the payload to MSC B via `mt-ForwardSM`, which delivers it over the radio interface (`SMS-DELIVER`).

---

## 6. Call Setup Evolution: SS7 ISUP to 5G IMS / SIP

### 6.1 Traditional Call Setup: ISUP 4-Message Flow
In legacy telephone networks, voice calls are established across the SS7 network using **ISUP (ISDN User Part)**:

```
Caller (Phone A)        MSC A (Originating)      MSC B (Terminating)        Callee (Phone B)
       │                         │                        │                         │
       │── Dialed Digits ───────>│                        │                         │
       │                         │─── 1. IAM ────────────>│                         │
       │                         │    (Initial Address)   │── Ring Signal ─────────>│
       │                         │<── 2. ACM ─────────────│                         │
       │<── Audible Ringback ────│    (Address Complete)  │                         │
       │                         │                        │<── Phone Picked Up ─────│
       │                         │<── 3. ANM ─────────────│                         │
       │                         │    (Answer Message)    │                         │
       │═════════════════════════╪════════════════════════╪═════════════════════════│
       │      [ 64 kbps DS0 Dedicated Voice Bearer Circuit Cut Through ]            │
       │      SS7 Signaling Network is COMPLETELY IDLE during conversation!         │
       │════════════════════════════════════════════════════════════════════════════│
       │── Hangs Up (On-Hook) ──>│                        │                         │
       │                         │─── 4. REL (Release) ──>│                         │
       │                         │<── 5. RLC (Complete) ──│                         │
```

- **Four core messages** establish and tear down the call:
  1. **IAM (Initial Address Message)**: Contains caller/callee numbers and requests a circuit.
  2. **ACM (Address Complete Message)**: Callee phone is ringing; caller switch generates ringback tone.
  3. **ANM (Answer Message)**: Callee answers; switches connect the 64 kbps voice circuit.
  4. **REL / RLC (Release / Release Complete)**: Tear down and deallocate the voice circuit.
- **Key Insight**: Once the voice circuit is connected, **the signaling network is completely idle**. It operates exclusively at the edges of the call lifecycle.

---

### 6.2 Modern Voice: 5G / VoLTE via SIP and RTP

In 4G (VoLTE) and 5G networks, circuit-switched ISUP is replaced by **SIP (Session Initiation Protocol)** running over standard IP:

```
Alice (User Agent)               Proxy / SBC (IMS Core)             Bob (User Agent)
       │                                  │                                  │
       │─── 1. INVITE sip:bob@scaler.com ─>│                                  │
       │       (SDP Offer: Audio Codecs)  │─── 1. INVITE (Forwarded) ───────>│
       │<── 2. 100 Trying ────────────────│                                  │
       │                                  │<── 3. 180 Ringing ───────────────│
       │<── 3. 180 Ringing ───────────────│                                  │
       │                                  │<── 4. 200 OK (SDP Answer) ───────│
       │<── 4. 200 OK ────────────────────│                                  │
       │─── 5. ACK (Directly to Bob) ────────────────────────────────────────>│
       │                                                                     │
       │<════════════════════════════════════════════════════════════════════>│
       │    RTP / SRTP Media Flow (Direct peer-to-peer streaming over UDP)   │
       │    The IMS Proxy / Signaling Core is bypassed during media flow!     │
       │<════════════════════════════════════════════════════════════════════>│
       │                                                                     │
       │─── 6. BYE ──────────────────────────────────────────────────────────>│
       │<── 7. 200 OK ───────────────────────────────────────────────────────│
```

- **Architectural Borrowing**: SIP borrowed HTTP's text formatting and status codes (`100 Trying`, `180 Ringing`, `200 OK`).
- **Separation of Concerns**:
  - **Signaling**: Plaintext SIP messages navigate intermediate proxy servers and Session Border Controllers (SBCs).
  - **Media Stream**: Audio packets stream peer-to-peer using **RTP (Real-time Transport Protocol)** over **UDP**, keeping high-volume audio payloads off the signaling servers.

---

### 6.3 Reliability: Per-Hop (SS7) vs. End-to-End (TCP)

```
SS7 Hop-by-Hop Reliability (Every link is strictly guaranteed):
[ Signaling Point A ] ──(Reliable L2)──> [ Signal Transfer Point ] ──(Reliable L2)──> [ Signaling Point B ]

TCP End-to-End Reliability (Intermediate routers are best-effort):
[ Client Endpoint ] ──(Untrusted IP)──> [ Router 1 ] ──(Untrusted IP)──> [ Router 2 ] ──> [ Server Endpoint ]
  ▲                                                                                         ▲
  └────────────────────────────── L4 End-to-End Recovery ──────────────────────────────────┘
```

| Dimension | SS7 Telephony Architecture | Internet TCP/IP Architecture |
| :--- | :--- | :--- |
| **Reliability Scope** | **Hop-by-Hop**: MTP2 ensures error-free delivery between adjacent switches. | **End-to-End**: Intermediate routers are stateless and unreliable; endpoints handle recovery. |
| **Sequencing** | 7-bit FSN (wraps at 128) per signal unit. | 32-bit sequence numbers per byte. |
| **Flow Control** | SIB (Status Indication Busy): Blunt binary stop/go. | Dynamic sliding window advertised per segment. |
| **Congestion Control**| Transfer-controlled hop-by-hop via MTP3. | End-to-end algorithmic control (Reno, CUBIC, BBR). |
| **Security Model** | Implicit trust: Any entity with a Point Code is trusted. | Zero trust natively: Layered cryptographic security (TLS). |

---

## 7. Congestion Control: Loss-Based (Reno/CUBIC) vs. BBR

For decades, the dominant TCP congestion control algorithms (Reno, New Reno, CUBIC) operated on a single core assumption:
$$\mathbf{\text{Packet Loss}} = \mathbf{\text{Network Congestion}}$$

### 7.1 The Sawtooth Flaw of Loss-Based TCP
- **Mechanism**: The sender increases its Congestion Window ($CWND$) additively until a packet is dropped, then cuts $CWND$ in half (**Multiplicative Decrease**), producing a distinct **Sawtooth pattern**.

```
In-Flight Data / CWND
     ▲
     │       /\          /\          /\          /\
     │      /  \        /  \        /  \        /  \
     │     /    \      /    \      /    \      /    \     <-- Sawtooth Curve (Reno/CUBIC)
     │    /      \    /      \    /      \    /      \        Fills buffers until packet drops!
     │   /        \  /        \  /        \  /        \
     │  /          \/          \/          \/          \
     │──────────────────────────────────────────────────
     │  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~   <-- BBR (Smooth, operating at BDP)
     └───────────────────────────────────────────────────► Time
```

#### Why Loss-Based Congestion Control Fails Today:
1. **Bufferbloat**: Modern routers contain massive memory buffers. Reno/CUBIC fill these buffers to 100% capacity before a packet drops, introducing hundreds of milliseconds of artificial queueing latency.
2. **Lossy Wireless Media**: Wi-Fi, 5G, and satellite links drop packets frequently due to radio interference, multipath fading, or handoffs—**not congestion**. Loss-based TCP misinterprets radio noise as network congestion, slashing transmission rates unnecessarily.

---

### 7.2 BBR: Bottleneck Bandwidth and Round-Trip Time
Developed by Google in 2016, **BBR (Bottleneck Bandwidth and RTT)** decouples congestion control from packet loss.

- Instead of probing until packets drop, BBR continuously models two physical properties of the network path:
  1. **$\text{BtlBw}$ (Bottleneck Bandwidth)**: The maximum physical data rate the narrowest link can support.
  2. **$\text{RTprop}$ (Round-Trip Propagation Time)**: The physical transit delay when router queues are empty.
- BBR sets its target in-flight data volume to the **Bandwidth-Delay Product (BDP)**:
  $$\mathbf{\text{BDP}} = \mathbf{\text{BtlBw}} \times \mathbf{\text{RTprop}}$$
- **Result**: BBR transmits at the exact rate the bottleneck can process, keeping intermediate queues near empty. On lossy connections, BBR can deliver up to **$2,000\times$ higher throughput** while drastically lowering latency.

---

## 8. Transport Layer Deep Dive: TCP, UDP, and QUIC

### 8.1 TCP 3-Way Handshake: Why Three Legs?

```
Client                                                   Server
  │                                                        │
  │─── 1. SYN (seq=1000, MSS=1460, wscale=7) ─────────────>│ [SYN RECEIVED: half-open backlog]
  │                                                        │
  │<── 2. SYN-ACK (seq=5000, ack=1001, MSS=1460) ──────────│
  │                                                        │
  │─── 3. ACK (seq=1001, ack=5001) ───────────────────────>│ [ESTABLISHED: accept() returns]
  │                                                        │
  │─── 4. PSH-ACK (seq=1001, first data bytes) ───────────>│
```

#### Why are three segments required?
TCP provides a **full-duplex** connection. Both sides maintain independent sequence numbers:
- **Segment 1 (SYN)**: The client proposes its Initial Sequence Number ($ISN_c = 1000$).
- **Segment 2 (SYN-ACK)**: The server acknowledges the client's sequence number ($ack = 1001$) and proposes its own ($ISN_s = 5000$).
- **Segment 3 (ACK)**: The client acknowledges the server's sequence number ($ack = 5001$).
- **Why SYN and FIN consume a sequence number**: Both `SYN` and `FIN` control flags consume **one virtual sequence number** even when carrying zero data bytes. If they did not, an ACK for a bare SYN would be indistinguishable from a duplicate packet retransmission.

---

### 8.2 Random Initial Sequence Numbers (ISN) & The Mitnick Attack
- In early TCP implementations, the $ISN$ incremented predictably (e.g., by a fixed constant every second).
- **The Mitnick Attack (1995)**: Kevin Mitnick exploited predictable sequence numbers to execute a blind TCP spoofing attack:
  1. Silence the trusted machine using a SYN flood.
  2. Send a spoofed SYN to the target server impersonating the trusted machine's IP.
  3. Predict the server's $ISN$, send a forged ACK matching that prediction, and establish a blind TCP connection without ever seeing return packets.
  4. Inject unauthorized rsh/rlogin commands.
- **Modern Defense (RFC 6528)**: Modern operating systems generate unpredictable $ISNs$ using a keyed cryptographic hash:
  $$\mathbf{ISN} = M + \text{HMAC-SHA256}(\text{Source IP, Source Port, Dest IP, Dest Port, Secret Key})$$
  *(where $M$ is a monotonic clock timer).*

---

### 8.3 The SYN Flood Attack & Stateless SYN Cookies (1996)

Between receiving Segment 1 (SYN) and Segment 3 (ACK), the server stores connection state in its finite **SYN Queue (Backlog)**.

```
THE SYN FLOOD ATTACK:
Attacker sends millions of SYNs with spoofed source IPs ───> Server fills SYN Queue 
Server awaits ACKs that never arrive ───> Legitimate users receive "Connection Refused"!
```

#### The Defense: SYN Cookies (Invented by Dan Bernstein, 1996)
When the SYN Queue overflows, the server transitions to **stateless connection tracking**:
1. The server allocates **zero memory and zero backlog entries**.
2. It encodes all critical connection state directly into its own 32-bit $ISN$:

```
┌─────────────────────────┬──────────────────────┬────────────────────────────────┐
│  Timestamp (t mod 32)   │   MSS Index (3 bits) │ Cryptographic Hash (24 bits)   │
│       5 bits            │       3 bits         │ HMAC(4-tuple, secret, t)       │
└─────────────────────────┴──────────────────────┴────────────────────────────────┘
 Total: 32 Bits of the TCP Initial Sequence Number (ISN)
```

3. When a legitimate client sends Segment 3 (`ACK`), its `ack_number` is $ISN + 1$.
4. The server inspects the ACK, subtracts 1, checks that the timestamp is fresh, recomputes the cryptographic hash, extracts the MSS index, and **reconstructs the connection socket on the fly from a packet it has never seen before**.

> **The Trade-off**: Because an $ISN$ offers only 32 bits of storage, advanced TCP options like **Window Scaling** and **SACK (Selective ACK)** are lost during SYN cookie mode. Consequently, kernels only activate SYN cookies when the backlog is under active pressure.

---

### 8.4 TCP Teardown: Why Four Segments? The Half-Close State

Because TCP is full-duplex, terminating a connection requires closing each direction independently:

```
Active Closer (Client)                               Passive Closer (Server)
      │                                                         │
      │─── 1. FIN-ACK (seq=2000, ack=6000) ────────────────────>│ [CLOSE_WAIT]
      │                                                         │ App notified of EOF (read returns 0)
      │<── 2. ACK (ack=2001) ───────────────────────────────────│
      │ [FIN_WAIT_2: Half-Closed]                               │ Server can still send data!
      │                                                         │ ...drains remaining buffers...
      │<── 3. FIN-ACK (seq=6000, ack=2001) ─────────────────────│
      │                                                         │
      │─── 4. ACK (ack=6001) ──────────────────────────────────>│ [CLOSED]
      │ [TIME_WAIT: 2 * MSL (60s)]                              │
```

- **The Half-Close State**: After Segment 2, the client has signaled it will transmit no further data, but it can still receive data from the server.
- **Why Segments 2 and 3 Cannot be Combined**: While a server's SYN and ACK can be combined during connection setup, the server's teardown ACK and FIN usually cannot be combined because the application process may still have pending outbound data to write before it calls `close()`.

---

### 8.5 Sockets Stuck in `CLOSE_WAIT` vs. `TIME_WAIT`
- **`CLOSE_WAIT`**: The remote peer closed the connection (sent FIN), and the OS kernel acknowledged it. The local socket remains in `CLOSE_WAIT` waiting for **the local application code to call `close()`**.
  > **Diagnostic Rule**: If your server has hundreds of sockets stuck in `CLOSE_WAIT`, **it is an application code bug (unclosed resource leak)**, not a network problem!
- **`TIME_WAIT`**: Occurs on the endpoint that initiates the active close. It lasts for $2 \times \text{MSL}$ (60 seconds) to ensure delayed packets dissipate. Sockets in this state are managed entirely by the kernel.

---

### 8.6 The Round-Trip Latency Tax

Bandwidth can be upgraded with hardware, but propagation latency is constrained by the speed of light:
$$\text{Mumbai to Virginia Round-Trip Time} \approx 190 \text{ ms}$$

```
LATENCY TAX BEFORE FIRST APPLICATION DATA BYTE:

1. Plain TCP                  : [ RTT ]                                     = 1 RTT (~190 ms)
2. TCP + TLS 1.2              : [ RTT ] [ RTT ] [ RTT ]                     = 3 RTT (~570 ms)
3. TCP + TLS 1.3              : [ RTT ] [ RTT ]                             = 2 RTT (~380 ms)
4. TCP Fast Open (TFO)        : [ RTT ] (Data inside SYN for returning client)= 1 RTT (~190 ms)
5. QUIC / HTTP/3              : [ RTT ] (Transport + TLS 1.3 combined)      = 1 RTT (~190 ms)
   (QUIC 0-RTT Resumption)    : [ 0 RTT ] (Data sent in very first packet)  = 0 RTT (0 ms tax!)
```

---

### 8.7 UDP: Minimalist Transport (8-Byte Header)

```
TCP Header (Minimum 20 Bytes, up to 60 with options):
[ Src Port | Dst Port | Seq Number | Ack Number | Offset/Flags | Window | Checksum | Urgent | Options... ]

UDP Header (Strictly 8 Bytes):
┌──────────────────────────────┬──────────────────────────────┐
│     Source Port (2 Bytes)    │  Destination Port (2 Bytes)  │
├──────────────────────────────┼──────────────────────────────┤
│     Length (2 Bytes)         │     Checksum (2 Bytes)       │
└──────────────────────────────┴──────────────────────────────┘
 Payload: Up to 65,507 Bytes (65,535 - 20 Byte IP Header - 8 Byte UDP Header)
```

- UDP contains only four fields: Source Port, Destination Port, Length, and Checksum.
- Used in real-time communications (DNS, Zoom, WebRTC, multiplayer gaming) where an out-of-order or late packet is useless and retransmission would only compound latency.

---

### 8.8 QUIC & HTTP/3: Defeating Head-of-Line Blocking

#### The Problem with HTTP/2 over TCP:
HTTP/2 multiplexes multiple logical streams over a single TCP connection. However, because TCP views all data as one contiguous byte stream:
- If a single packet belonging to **Stream 1** is dropped in transit, **TCP halts delivery of all other streams (Stream 2, Stream 3)** until the missing packet is retransmitted. This is called **Transport-Layer Head-of-Line (HoL) Blocking**.

```
HTTP/2 over TCP (Single Stream Bottleneck):
[ Stream 1 | Stream 2 | Stream 3 ] ──> Packets on wire ──> [ PACKET LOST ]
                                                                 │
 ALL streams blocked in kernel receive buffer until retransmitted! ◄─┘

HTTP/3 over QUIC (Independent Byte Streams):
[ Stream 1 ] ──> Packet Lost ──> Only Stream 1 waits!
[ Stream 2 ] ─────────────────> Delivered to browser immediately!
[ Stream 3 ] ─────────────────> Delivered to browser immediately!
```

#### Why Userspace Matters:
- TCP is implemented inside the operating system kernel. Modifying TCP requires updating every client and server OS globally, a process that can take a decade.
- **QUIC is implemented in user space** on top of UDP. Application developers (and browsers like Chrome) can deploy new congestion control algorithms and protocol updates instantly without waiting for operating system releases.

---

## 9. Human-Readable Text Protocols: SMTP, POP3, IMAP, FTP

### 9.1 SMTP: Simple Mail Transfer Protocol & Security Patches

Standardized in RFC 821 (1982) and updated in RFC 5321:
- **Port 25**: Server-to-server relay (plaintext).
- **Port 587**: Client-to-server submission (uses STARTTLS).
- **Port 465**: SMTPS (implicit TLS from connection start).

#### The Security Deficit of Base SMTP:
SMTP was designed in a high-trust academic environment. The base protocol contains **zero authentication**:
```text
EHLO scaler-demo
MAIL FROM:<ceo@google.com>       <-- Any arbitrary email can be typed!
RCPT TO:<victim@scaler.com>
DATA
From: Sundar Pichai <ceo@google.com>
To: Victim <victim@scaler.com>
Subject: Urgent Bonus

Click here to collect your bonus.
.
QUIT
```

#### Envelope vs. Letter Discrepancy:
- **Envelope (`MAIL FROM` / `RCPT TO`)**: Directs intermediate mail routing switches.
- **Letter (`From:` / `To:` inside `DATA`)**: The text displayed in the user's email client interface.
- They do not have to match, a discrepancy frequently exploited in email spoofing attacks.

#### Modern Defense Mechanisms:
1. **SPF (Sender Policy Framework)**: A DNS TXT record defining which server IP addresses are authorized to send email for a domain.
2. **DKIM (DomainKeys Identified Mail)**: The sending mail server signs the email body and headers with a private key. The receiver verifies this signature using the sender's public key published in DNS.
3. **DMARC**: Specifies policy actions (reject, quarantine) if SPF or DKIM validation fails.

#### SMTP Framing & Dot-Stuffing:
SMTP signals the end of a message with a single period on its own line: `\r\n.\r\n`.
- **Dot-Stuffing**: If a user's legitimate email body contains a line starting with a period (`.`), the sending client prepends an additional period (`..`). The receiving server strips the leading period, preserving the content without premature message termination.

---

### 9.2 MIME & Base64: Transporting Binary Data as Text

Because early email transport systems only supported 7-bit ASCII, binary attachments (images, PDFs) must be converted into safe printable characters using **Base64**:

$$\text{3 Raw Binary Bytes (24 Bits)} \longrightarrow \text{4 Printable ASCII Characters (24 Bits)}$$
$$\text{Expansion Ratio} = \frac{4}{3} \approx \mathbf{33.3\% \text{ Bandwidth Overhead}}$$

MIME (RFC 2045) separates email segments using boundary strings:
```text
Content-Type: multipart/mixed; boundary="=_boundary123"

--=_boundary123
Content-Type: text/plain

Here is your receipt.
--=_boundary123
Content-Type: image/png
Content-Transfer-Encoding: base64

iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==
--=_boundary123--
```

---

### 9.3 Mail Retrieval: POP3 vs. IMAP Architecture

```
┌───────────────────────────────────────┬───────────────────────────────────────┐
│     POP3 (Port 110 / POP3S 995)       │      IMAP (Port 143 / IMAPS 993)      │
├───────────────────────────────────────┼───────────────────────────────────────┤
│ - Mail downloaded to local device;    │ - Mail resides permanently on server; │
│   server mailbox is emptied.          │   client is a synchronized cache.     │
│ - No synchronization across devices.  │ - State changes sync across all phones│
│ - Single folder: Inbox only.          │ - Supports arbitrary folder trees.    │
│ - Search executed locally after dl.   │ - Server-side search (`SEARCH`).      │
│ - Download is all-or-nothing (`RETR`).│ - Supports partial fetch (`BODY[1]`). │
└───────────────────────────────────────┴───────────────────────────────────────┘
```

#### Core POP3 Commands:
- `USER <name>` / `PASS <password>`: Plaintext authentication.
- `STAT`: Returns message count and total mailbox byte size.
- `LIST`: Lists individual message numbers and byte sizes.
- `RETR <n>`: Downloads message $n$.
- `DELE <n>`: Marks message $n$ for deletion (applied upon `QUIT`).

---

### 9.4 The Taxonomy of Protocol Framing: Delimiter, Length, and Both

```
┌─────────────────────────────┬─────────────────────────────┬─────────────────────────────┐
│      1. DELIMITER-BASED     │       2. LENGTH-PREFIXED    │           3. BOTH           │
├─────────────────────────────┼─────────────────────────────┼─────────────────────────────┤
│ - Read until sentinel token │ - Read explicit length field│ - Delimit header, length-   │
│ - SMTP: "\r\n.\r\n"         │ - SMS: TP-UDL (septets)     │   prefix the payload body   │
│ - HTTP Headers: CRLF CRLF   │ - HTTP: Content-Length      │ - HTTP/1.1: CRLF + Length   │
│ - Redis: CRLF               │ - Protobuf varint length    │ - IMAP literals: {310}\r\n  │
│ - Pro: Easy to stream       │ - Pro: Binary safe; no scan │ - Pro: Pragmatic & safe     │
│ - Con: Requires escaping    │ - Con: Must know size upfront│ - Con: Request Smuggling risk│
└─────────────────────────────┴─────────────────────────────┴─────────────────────────────┘
```

> **Security Vulnerability: HTTP Request Smuggling**: When an architecture combines delimiter-based framing (Chunked Transfer Encoding) and length-prefixed framing (`Content-Length`), discrepancies between how a reverse proxy and an application server prioritize these headers allow attackers to inject hidden requests into the pipeline.

---

### 9.5 FTP: Dual-Connection Architecture (Active vs. Passive Mode)

Unlike most protocols, **FTP (File Transfer Protocol, RFC 959)** separates control commands and file data across **two distinct TCP connections**:
1. **Control Connection (Port 21)**: Remains open throughout the session for authentication and issuing commands (`USER`, `PASS`, `LIST`, `RETR`, `STOR`, `PASV`).
2. **Data Connection**: A temporary TCP connection opened and torn down for every individual file transfer or directory listing.

```
ACTIVE MODE (1971):
Client (Port 1050) ─── Connects to Server Port 21 (Control) ───> Server
Client tells server: "Connect to my port 1051" (PORT command)
Server (Port 20) ──── Dials OUTBOUND to Client Port 1051 ─────> Client
   FAILED! Modern client firewalls and NAT routers block incoming server dials.

PASSIVE MODE (Modern Standard):
Client (Port 1050) ─── Connects to Server Port 21 (Control) ───> Server
Client sends: PASV
Server replies: 227 Entering Passive Mode (127,0,0,1,117,48)
Client (Port 1052) ─── Dials OUTBOUND to Server Port 30000 ────> Server
   SUCCESS! Both connections are outbound from the client; NAT/firewalls allow it.
```

---

### 9.6 Decoding FTP 227 Port Numbers: Big-Endian Math

When a client sends `PASV`, the FTP server returns a comma-delimited response:
```text
227 Entering Passive Mode (127,0,0,1,117,48)
```
- The first four numbers represent the IPv4 address: `127.0.0.1`.
- The last two numbers ($p1, p2$) encode the 16-bit TCP port number in **Network Byte Order (Big-Endian)**:

$$\mathbf{\text{Port Number}} = (\mathbf{p1} \times 256) + \mathbf{p2}$$
$$\text{Port Number} = (117 \times 256) + 48 = 29952 + 48 = \mathbf{30000}$$

#### Why was the port split across two numbers?
In 1971, early computer terminals and mini-computers struggled to parse numbers larger than 255 (1 byte) out of text strings. Splitting the 16-bit integer into two 8-bit halves allowed systems to parse the port as two simple byte values.

---

## 10. Comprehensive Review & Practice MCQs (with Detailed Explanations)

---

### Question 1
Why does Wi-Fi (802.11) use CSMA/CA (Collision Avoidance) instead of Ethernet's CSMA/CD (Collision Detection)?
- A) Radio signals propagate faster than electrical signals in copper.
- B) A radio transmitter's output signal drowns out incoming signals, preventing it from detecting collisions while transmitting.
- C) 802.11 Wi-Fi frames do not contain cyclic redundancy checks (CRC).
- D) Wireless routers cannot buffer data frames.

**Answer: B**  
**Explanation:** Radios operate in half-duplex mode during transmission; the emitted RF energy is far stronger than any incoming signal from a remote station. Because a station cannot listen for collisions while transmitting, Wi-Fi must actively avoid collisions using carrier sensing, random backoff contention windows, and explicit Layer 2 ACKs.

---

### Question 2
What was the primary vulnerability of telephone networks that used in-band signaling prior to SS7?
- A) Slower call connection times due to satellite relays.
- B) Control tones shared the same channel as caller audio, allowing unauthorized users to control switches using generated tones (e.g., 2600 Hz).
- C) Telephone switches could not process touch-tone dialing.
- D) Voice conversations were uncompressed.

**Answer: B**  
**Explanation:** In-band signaling carried both voice audio and trunk control tones on the same circuit. Generating a 2600 Hz tone signaled to the switch that a trunk line was idle, allowing phreakers using "blue boxes" to seize lines and route toll-free calls. SS7 solved this by moving all signaling to an out-of-band packet network.

---

### Question 3
Why was the SMS (Short Message Service) payload length restricted to 140 octets in GSM networks?
- A) Early SIM cards only had 140 bytes of total memory.
- B) To ensure the entire SMS operation fit inside a single 272-octet SS7 Message Signal Unit (MSU) without requiring packet segmentation or reassembly.
- C) To match the 140-character limit of early computer monitors.
- D) Because radio spectrum regulators restricted text transmissions to 140 bytes.

**Answer: B**  
**Explanation:** In SS7 MTP2, the Signaling Information Field (SIF) is capped at 272 octets. Limiting the SMS payload to 140 octets guaranteed that all MAP, TCAP, SCCP, and MTP headers would fit into a single signaling packet, eliminating the need for segmentation or reassembly on resource-constrained 1980s switches.

---

### Question 4
How does the GSM-7 encoding standard achieve a 160-character message length within a 140-octet storage limit?
- A) It uses Huffman compression algorithms on vowels.
- B) It encodes characters using 7-bit septets rather than 8-bit octets ($1120 \text{ bits} \div 7 = 160$).
- C) It omits punctuation characters from the character set.
- D) It stores vowels in the packet header.

**Answer: B**  
**Explanation:** 140 bytes provide $140 \times 8 = 1120$ bits. Because the GSM-7 alphabet uses 7 bits per character instead of 8 bits, $1120 \div 7 = 160$ printable characters can be packed into the field by concatenating the septets.

---

### Question 5
What happens to an SMS message's length capacity if a sender includes a single emoji (e.g., 😊)?
- A) The message capacity expands to 280 characters.
- B) The message fails to send because mobile networks only support Latin characters.
- C) The encoding switches to UCS-2 (16 bits per character), reducing the maximum character limit from 160 to 70.
- D) The mobile phone automatically compresses the emoji into 7 bits.

**Answer: C**  
**Explanation:** Non-GSM characters require UCS-2 (UTF-16) encoding, which consumes 2 bytes (16 bits) per character. Because the physical field remains fixed at 140 bytes, $140 \div 2 = 70$ characters maximum.

---

### Question 6
During an SS7 telephone call, what is the role of the SS7 signaling network while the two parties are actively speaking?
- A) It continuously streams digitized voice packets.
- B) It is completely idle; signaling occurs only at call setup (IAM/ACM/ANM) and teardown (REL/RLC).
- C) It monitors voice frequencies for echo cancellation.
- D) It checks billing balances every 100 milliseconds.

**Answer: B**  
**Explanation:** SS7 operates strictly out-of-band as a control plane. Once the 64 kbps voice bearer circuit is established between local exchanges, the signaling network has completed its work and remains idle until one party hangs up.

---

### Question 7
How does BBR congestion control differ from traditional loss-based algorithms like TCP Reno and CUBIC?
- A) BBR uses UDP instead of TCP.
- B) BBR treats packet loss as a primary indicator of network congestion.
- C) BBR models bottleneck bandwidth and round-trip propagation time, transmitting at the rate the bottleneck link can support to keep router queues empty.
- D) BBR disables the TCP 3-way handshake.

**Answer: C**  
**Explanation:** Loss-based algorithms increase window size until buffers overflow and packets drop (causing bufferbloat). BBR continuously measures maximum bottleneck bandwidth and minimum RTT, pacing traffic to match the path's actual capacity without filling intermediate buffers.

---

### Question 8
Why does the second segment of a standard TCP 3-way handshake acknowledge the client's sequence number as $ISN_c + 1$ even when the SYN carries no payload data?
- A) To account for the 1-byte IP version field.
- B) Because the SYN control flag consumes one logical sequence number, allowing the receiver to distinguish new connections from duplicate retransmissions.
- C) Because TCP sequence numbers count in 2-byte increments.
- D) To confirm the client's window scale option.

**Answer: B**  
**Explanation:** Both SYN and FIN flags consume one sequence number in the sequence space. This ensures their delivery is reliably acknowledged; otherwise, an ACK for an empty SYN would look identical to a retransmitted segment.

---

### Question 9
How do TCP SYN Cookies defend a server against SYN Flood attacks?
- A) By dropping all incoming connection attempts from unknown subnets.
- B) By encoding connection state into the server's Initial Sequence Number, allowing the server to avoid allocating memory until the client's final ACK arrives.
- C) By encrypting the server's Accept Queue with AES-256.
- D) By increasing the kernel's `somaxconn` parameter to infinity.

**Answer: B**  
**Explanation:** During a SYN flood, the server's SYN queue is overwhelmed by half-open connections. With SYN cookies, the server creates no backlog entry. It encodes a timestamp, MSS index, and cryptographic hash of the 4-tuple into its ISN. When the client returns the final ACK, the server recomputes the hash and instantiates the connection on demand.

---

### Question 10
If a server has hundreds of sockets lingering in the `CLOSE_WAIT` state, what is the most likely root cause?
- A) Severe packet loss on intermediate network routers.
- B) An application code bug where the server failed to call `close()` on socket descriptors after reading an EOF.
- C) A misconfigured local firewall dropping incoming FIN packets.
- D) The client operating system crashed before sending an ACK.

**Answer: B**  
**Explanation:** A socket enters `CLOSE_WAIT` when it receives a FIN from the remote peer and returns an ACK. It remains in this state until the local application calls `close()`. Accumulating sockets in `CLOSE_WAIT` points to an application resource leak.

---

### Question 11
What is the primary architectural cause of Transport-Layer Head-of-Line (HoL) Blocking in HTTP/2?
- A) HTTP/2 uses text-based header formatting.
- B) Multiple streams are multiplexed over a single TCP connection, so a single dropped packet causes TCP to pause delivery of all streams until retransmitted.
- C) HTTP/2 relies on TLS 1.2 instead of TLS 1.3.
- D) Operating systems limit HTTP/2 to six concurrent streams.

**Answer: B**  
**Explanation:** Because TCP guarantees a single, strictly ordered byte stream, a dropped segment halts the delivery of all data behind it in the kernel receive buffer—even if the trailing bytes belong to completely unrelated HTTP/2 streams. QUIC solves this by giving each stream its own sequence space over UDP.

---

### Question 12
Why does base SMTP allow a client to specify any arbitrary sender in the `MAIL FROM:` command?
- A) Early ARPANET implementations lacked DNS servers.
- B) Base SMTP (RFC 821) was designed in an era of mutual trust and includes no built-in sender authentication mechanisms.
- C) The `MAIL FROM:` command is only used for billing, not routing.
- D) SMTP automatically authenticates senders using their MAC address.

**Answer: B**  
**Explanation:** When SMTP was drafted in 1982, the internet was a closed network of trusted academic and military institutions. Sender verification mechanisms like SPF, DKIM, and DMARC were developed decades later to address this lack of authentication.

---

### Question 13
What is the purpose of "dot-stuffing" in the SMTP protocol?
- A) To compress repeated whitespace in email text.
- B) To prevent a line in the email body that begins with a period from being misinterpreted as the end-of-message delimiter (`\r\n.\r\n`).
- C) To pad short email messages up to the minimum packet size.
- D) To encrypt sensitive email headers.

**Answer: B**  
**Explanation:** SMTP terminates messages when it encounters a single period on its own line (`\r\n.\r\n`). If a line in the body begins with a period, the sender prepends an extra period (`..`). The receiver strips the leading dot, preventing premature message termination.

---

### Question 14
Why does FTP Active Mode routinely fail on modern home and corporate networks?
- A) Active Mode requires modern IPv6 addressing.
- B) Active Mode requires the server to initiate an inbound TCP connection to the client's data port, which modern NAT routers and firewalls block.
- C) Active Mode is restricted to uncompressed text files.
- D) Active Mode limits file transfers to 140 bytes.

**Answer: B**  
**Explanation:** In Active Mode, the client sends a `PORT` command telling the server where to connect, and the server dials back into the client from port 20. Stateful firewalls and NAT routers drop unsolicited incoming connections from external servers. Passive Mode (`PASV`) resolves this by having the client initiate both connections outbound.

---

### Question 15
An FTP server in Passive Mode sends the response: `227 Entering Passive Mode (192,168,1,10,120,50)`. What TCP port must the client connect to for the data transfer?
- A) Port 12050
- B) Port 30770
- C) Port 120
- D) Port 50120

**Answer: B**  
**Explanation:** The port is encoded as two 8-bit values in Big-Endian order:
$$\text{Port} = (p1 \times 256) + p2 = (120 \times 256) + 50 = 30720 + 50 = 30770$$

---

## 11. Session 2 Summary Cheat Sheet

| Topic / Concept | Key Takeaway / Value |
| :--- | :--- |
| **Encapsulation** | "One layer's headers are another layer's body." Ethernet $\supset$ IP $\supset$ TCP $\supset$ HTTP. |
| **CSMA/CD vs. CSMA/CA**| Wire allows collision detection. Radio cannot listen while transmitting; requires collision avoidance and explicit L2 ACKs ($\approx 2\text{ ms}$ delay). |
| **Starlink Layering** | Traverses 9 physical hops (including 550 km into space); IP header arrives unchanged. |
| **In-Band vs Out-of-Band**| In-band mixes voice and control (vulnerable to blue boxes). SS7 moved signaling to an isolated packet network. |
| **SMS 140-Byte Limit**| Derived from MTP2's 272-octet SIF limit. Guarantees zero segmentation/reassembly on 1980s switches. |
| **GSM-7 Packing** | 140 bytes $\times$ 8 bits = 1120 bits. $1120 \div 7\text{ bits} = \mathbf{160\text{ characters}}$. |
| **The Emoji Trap** | Adding an emoji switches encoding to UCS-2 (16 bits/char), halving message capacity to **70 characters**. |
| **BBR Congestion Control**| Measures Bottleneck Bandwidth and RTT ($BDP = \text{BtlBw} \times \text{RTprop}$). Eliminates bufferbloat; up to $2,000\times$ faster on lossy links. |
| **3-Way Handshake** | Synchronizes and acknowledges independent sequence numbers in both directions. SYN and FIN each consume 1 sequence number. |
| **SYN Cookies** | Stateless defense against SYN floods. Encodes 4-tuple hash, timestamp, and MSS index into the 32-bit ISN. |
| **Sockets in `CLOSE_WAIT`**| Application bug: user code forgot to call `close()` after receiving EOF. |
| **QUIC / HTTP/3** | Implements independent streams over UDP in userspace. Eliminates TCP transport head-of-line blocking. |
| **Envelope vs Letter** | `MAIL FROM` (envelope) routes the message. `From:` (letter) is display-only text. They do not have to match. |
| **FTP 227 Port Math** | Port $= (p1 \times 256) + p2$. Big-Endian 16-bit port split into two ASCII numbers. |
