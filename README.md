# Computer Networks: Architecture, Systems & Protocols

> **Repository Overview**: Comprehensive, detailed theoretical notes and examination guides for the 8-week Computer Networks master curriculum (`cn-at-scaler`).
> 
> Each chapter breaks down complex, low-level networking topics into beginner-friendly, rigorous explanations covering operating system internals, socket programming, kernel queues, signals, network protocols, server architectures, and distributed systems. Every chapter concludes with high-yield multiple-choice questions (MCQs) for exam preparation.

---

## 📚 Curriculum Roadmap

| Week | Chapter | Focus & Key Concepts | Status |
| :---: | :--- | :--- | :---: |
| **01** | [**Chapter 01: Network Programming 101**](./Chapter_01_Network_Programming_101/Chapter_01_Network_Programming_101.md) | The 7 system calls, Ports & root privilege, Accept queues, SIGPIPE, Ephemeral ports, Endianness (`htons`), Concurrency (`fork`, `select`, `epoll`), TLS/SSL, Protocol framing (TLV, BCD), RPC/gRPC, and 15 MCQs. | ✅ Complete |
| **02** | **Chapter 02: Computer Networking Eagle Eye View** | Holistic internet architecture, routing protocols, packet switching vs circuit switching, transport fundamentals. | ⏳ Upcoming |
| **03** | **Chapter 03: Server Side Considerations for Scale** | Scaling architectures, threading vs event loops, connection pooling, memory limits, socket tuning. | ⏳ Upcoming |
| **04** | **Chapter 04: NGINX Deep Dive** | Master-worker process architecture, reverse proxying, event-driven I/O, configuration internals. | ⏳ Upcoming |
| **05** | **Chapter 05: Evolution of HTTP** | HTTP/0.9 to HTTP/1.1 (Keep-Alive, Pipelining), HTTP/2 (Multiplexing, HPACK), and HTTP/3 (QUIC, UDP). | ⏳ Upcoming |
| **06** | **Chapter 06: (Ab)using CDNs** | Content Delivery Networks, Edge caching, Anycast routing, Cache-Control, invalidation strategies. | ⏳ Upcoming |
| **07** | **Chapter 07: Economics of Cloud Tech** | Cloud networking topologies, egress bandwidth costs, transit gateways, VPC peering economics. | ⏳ Upcoming |
| **08** | **Chapter 08: Building for Failures** | Resilient distributed systems, timeouts, exponential backoff, circuit breakers, network partitions. | ⏳ Upcoming |

---

## 📂 Repository Directory Structure

```text
Network_Architecture/
├── README.md                                    # Master syllabus and navigation
└── Chapter_01_Network_Programming_101/
    └── Chapter_01_Network_Programming_101.md    # In-depth notes, diagrams, and MCQs
```

---

## 🎯 How to Use These Notes

1. **Read Top-Down**: Each chapter is organized logically from foundational concepts to advanced OS kernel mechanics and production architectures.
2. **Review the Edge Cases**: Special attention is given to *what breaks* under the hood (e.g., omitting system calls, `SIGPIPE` crashes, `TIME_WAIT` port exhaustion).
3. **Practice the MCQs**: Test your comprehension using the multiple-choice questions at the end of each module, complete with answer keys and rationale.
