# Networking — Beginner to Advanced — Complete Offline Reference

> **Who this is for:** A software engineer who can write code but treats "the network" as a black box — and wants to go from "I type a URL and a page appears" to genuinely understanding **how two programs on different machines talk**, layer by layer: cables and MAC addresses, IP addressing and subnetting, how a packet is routed across the planet, TCP's handshake and congestion control, UDP, sockets and ports, DNS, HTTP/1.1→2→3, TLS and certificates, NAT and firewalls, the real-world attacks and defenses, and the command-line tools you use to debug all of it — plus how to write network code yourself. Every topic is explained in **prose first** (what it is, *why* it exists, when and how it's used, the key fields/parameters, best practices, and **security** implications), then made concrete with diagrams, reference tables, real tool output, and **heavily-commented, runnable code**. Read top-to-bottom the first time; afterwards use the Table of Contents as a lookup. Sections are tagged **[B]** beginner, **[I]** intermediate, **[A]** advanced.
>
> **Version note:** Networking fundamentals (the OSI/TCP-IP models, IP, TCP, UDP, DNS) are decades-stable — what you learn here will be true for your whole career. The fast-moving parts, current **as of 2026**, are flagged with **⚡ Version note** and include:
> - **HTTP/3 over QUIC** (RFC 9114, on top of QUIC RFC 9000) is mainstream — running over **UDP**, with TLS 1.3 built in. **HTTP/2** (RFC 9113) is everywhere; HTTP/1.1 is still the floor.
> - **TLS 1.3** (RFC 8446) is the recommended minimum; TLS 1.0/1.1 are deprecated and TLS 1.2 is the lowest you should still accept. **QUIC** folds the TLS handshake into the transport handshake.
> - **IPv6** adoption is past ~45% of Google traffic; you must be dual-stack literate. **HTTP/2 and /3**, **DoH/DoT** (DNS over HTTPS/TLS), and **Encrypted Client Hello (ECH)** are live.
> - Code examples use **Go 1.26**, **Node 24 LTS**, and **Python 3.13**; CLI examples use Linux tooling (`ip`, `ss`, `dig`, `tcpdump`) with Windows equivalents noted (the author is on **Windows 11**; for real packet work you'll most often be on Linux, frequently inside **Docker** — cross-reference the **Docker** guide).
>
> Cross-references to the **Nginx**, **Docker**, **Go net/http**, **Go Gorilla WebSockets**, **Go gRPC**, **Node.js**, and **Better Auth/Supabase** guides appear where relevant. Authoritative sources to confirm details: the **RFCs** (`rfc-editor.org`), `man` pages, and `developer.mozilla.org` for web protocols.

---

## Table of Contents

1. [What "Networking" Is & Why — The Layered Mental Model](#1-what-networking-is--why--the-layered-mental-model) **[B]**
2. [The OSI & TCP/IP Models — Encapsulation Layer by Layer](#2-the-osi--tcpip-models--encapsulation-layer-by-layer) **[B]**
3. [Layer 1 & 2: Physical, Ethernet, MAC, Switches, ARP, Wi-Fi, VLANs](#3-layer-1--2-physical-ethernet-mac-switches-arp-wi-fi-vlans) **[B/I]**
4. [Layer 3: IP — Addressing, Subnetting/CIDR, IPv6](#4-layer-3-ip--addressing-subnettingcidr-ipv6) **[B/I]**
5. [Routing — How a Packet Crosses the Planet (NAT, BGP)](#5-routing--how-a-packet-crosses-the-planet-nat-bgp) **[I]**
6. [Layer 4: TCP vs UDP — Ports, Reliability, Flow & Congestion Control](#6-layer-4-tcp-vs-udp--ports-reliability-flow--congestion-control) **[B/I/A]**
7. [Sockets & the Socket API — Where Code Meets the Network](#7-sockets--the-socket-api--where-code-meets-the-network) **[I]**
8. [DNS — The Internet's Phone Book](#8-dns--the-internets-phone-book) **[B/I]**
9. [Layer 7: HTTP/1.1 → HTTP/2 → HTTP/3](#9-layer-7-http11--http2--http3) **[B/I/A]**
10. [TLS / SSL, Certificates & PKI — Encryption on the Wire](#10-tls--ssl-certificates--pki--encryption-on-the-wire) **[I/A]**
11. [WebSockets, SSE & Other Application Protocols](#11-websockets-sse--other-application-protocols) **[I]**
12. [Network Security — Firewalls, VPNs, Attacks & Defenses](#12-network-security--firewalls-vpns-attacks--defenses) **[I/A]**
13. [Tools & Troubleshooting — ping, traceroute, dig, ss, curl, tcpdump, Wireshark, nmap](#13-tools--troubleshooting--ping-traceroute-dig-ss-curl-tcpdump-wireshark-nmap) **[B/I/A]**
14. [Network Programming — Writing Clients & Servers](#14-network-programming--writing-clients--servers) **[I/A]**
15. [Modern & Cloud Networking — Proxies, Load Balancers, CDNs, Containers, Service Mesh](#15-modern--cloud-networking--proxies-load-balancers-cdns-containers-service-mesh) **[A]**
16. [Performance — Latency, Bandwidth, RTT, Throughput, the BDP](#16-performance--latency-bandwidth-rtt-throughput-the-bdp) **[I/A]**
17. [Gotchas & Best Practices](#17-gotchas--best-practices) **[I/A]**
18. [Study Path & Build-to-Learn Projects](#18-study-path--build-to-learn-projects)

---

## 1. What "Networking" Is & Why — The Layered Mental Model

### 1.1 The one-sentence definition **[B]**

**Networking is the set of agreements (protocols) that let a program on one computer send bytes to a program on another computer, reliably, across hardware nobody controls end-to-end.** Everything in this guide is an elaboration of that sentence: *which* bytes, in *what* order, with *what* envelope around them, sent over *what* medium, found via *what* address, and protected *how*.

When your browser loads `https://example.com`, dozens of protocols cooperate in well under a second: DNS turns the name into an IP address, TCP opens a reliable connection, TLS encrypts it, HTTP requests the page, and IP + Ethernet + Wi-Fi physically shuttle the bits across a dozen machines you've never heard of. The miracle is that **each protocol only has to solve one problem** and trust the others to solve theirs. That division of labor is the single most important idea in networking.

### 1.2 Why layers? The reason everything is stacked **[B]**

Imagine you had to write one giant function that took an HTTP request and turned it directly into voltage changes on a copper wire. It would be unmaintainable, and it would have to be rewritten every time someone invented Wi-Fi, or fiber, or a faster Ethernet. Networking solves this exactly the way good software does: **with abstraction layers that each expose a simple interface and hide their internals.**

The deal between layers is:

- **Each layer talks only to the layer directly above and below it.** HTTP hands its data to TCP and doesn't care how TCP delivers it. TCP hands segments to IP and doesn't care how IP finds a route. IP hands packets to Ethernet and doesn't care whether the wire is copper, fiber, or radio.
- **Each layer adds its own "envelope" (a header) on the way down and removes it on the way up.** This wrapping is called **encapsulation** (Section 2.3). Your data ends up nested like Russian dolls.
- **A layer on the sending machine has a logical conversation with the *same* layer on the receiving machine.** Your browser's HTTP layer "talks to" the server's HTTP layer, even though physically the bytes went down through TCP/IP/Ethernet, across the world, and back up. These are called **peer layers**.

The payoff: you can swap any layer without touching the others. Replace copper with fiber — TCP/IP never notices. Replace HTTP/1.1 with HTTP/3 — the Ethernet card never notices. **This is why a model from the 1980s still describes a network running protocols invented last year.**

### 1.3 Two machines, one conversation — the whole stack in one example **[B]**

Here is the entire journey of "your browser asks a server for a page," named at every layer. Don't worry about the details yet — the point is to see the shape.

```
   YOUR LAPTOP                                         THE SERVER
   ┌───────────────────────┐                           ┌───────────────────────┐
   │ Browser (HTTP request)│  ── application data ───▶  │ Web server (HTTP)     │  L7 Application
   ├───────────────────────┤                           ├───────────────────────┤
   │ TLS (encrypt)         │                           │ TLS (decrypt)         │  (L6-ish)
   ├───────────────────────┤                           ├───────────────────────┤
   │ TCP (port 443, reliab)│  ── segments ──────────▶   │ TCP (port 443)        │  L4 Transport
   ├───────────────────────┤                           ├───────────────────────┤
   │ IP (src/dst addresses)│  ── packets ───────────▶   │ IP                    │  L3 Network
   ├───────────────────────┤                           ├───────────────────────┤
   │ Wi-Fi / Ethernet (MAC)│  ── frames ────────────▶   │ Ethernet              │  L2 Link
   ├───────────────────────┤                           ├───────────────────────┤
   │ Radio / copper / fiber│  ── bits ──────────────▶   │ copper / fiber        │  L1 Physical
   └───────────────────────┘                           └───────────────────────┘
            │                                                       ▲
            └──▶ router ──▶ ISP ──▶ internet backbone ──▶ router ───┘
                 (each hop strips L2, reads L3, re-wraps L2 for the next link)
```

The crucial detail in that bottom line: **the IP layer is end-to-end (your laptop's IP address and the server's IP address stay the same the whole way), but the Ethernet/link layer is hop-by-hop (it is stripped and rebuilt at every single router along the path).** Holding those two facts in your head at once is the moment networking "clicks." We'll prove it in Sections 4 and 5.

### 1.4 Protocols, standards, and RFCs **[B]**

A **protocol** is just a written agreement: "a message of this type has these fields in this order, and here's what each value means." TCP, IP, HTTP, DNS, TLS — all are protocols. They are defined in **RFCs** (Requests for Comments), the numbered specification documents published by the **IETF** (Internet Engineering Task Force). When you need the *exact* truth about a protocol — not a blog's paraphrase — you read the RFC. A few you'll meet:

| Protocol | Defining RFC(s) | One-line purpose |
|---|---|---|
| IP (v4) | RFC 791 | Address and route packets across networks |
| IPv6 | RFC 8200 | IP with a vastly larger address space |
| TCP | RFC 9293 (was 793) | Reliable, ordered byte stream |
| UDP | RFC 768 | Fire-and-forget datagrams |
| DNS | RFC 1034/1035 | Names ↔ IP addresses |
| HTTP/1.1 | RFC 9112 | Request/response document transfer |
| HTTP/2 | RFC 9113 | Multiplexed binary HTTP |
| HTTP/3 | RFC 9114 | HTTP over QUIC/UDP |
| QUIC | RFC 9000 | Reliable, multiplexed transport over UDP |
| TLS 1.3 | RFC 8446 | Encryption + authentication on the wire |

You don't memorize RFC numbers. You just need to know that when an interview question or a production bug hinges on "what is the *defined* behavior," the answer lives in an RFC, and they're all free at `rfc-editor.org`.

### 1.5 A vocabulary you need immediately **[B]**

These words recur constantly; pin them now:

- **Host / node** — any device on a network with an address (your laptop, a server, a phone, a router).
- **Client / server** — roles, not hardware. The **client** initiates a request; the **server** listens and responds. One program can be both.
- **Packet / segment / frame / datagram** — the unit of data at different layers (the same bytes wearing different envelopes). Precise definitions in Section 2.
- **Bandwidth** — how *much* data per second a link can carry (e.g. 1 Gbps). The width of the pipe.
- **Latency** — how *long* one bit takes to get there (e.g. 20 ms). The length of the pipe. **Bandwidth and latency are independent** — a satellite link can have huge bandwidth and terrible latency (Section 16).
- **Throughput** — the bandwidth you *actually achieve* end-to-end (always ≤ the bottleneck link's bandwidth).
- **RTT (round-trip time)** — time for a packet to go there *and back*. The currency of network performance; almost every protocol's speed is measured in RTTs.
- **Protocol stack** — the set of layered protocols a machine implements (the "TCP/IP stack" in your OS kernel).

> **Cross-reference:** the **Nginx** guide explains the *C10k problem* — why event-driven servers handle thousands of connections — which is the practical payoff of understanding TCP connections at scale (Section 6 here). The **Docker** guide's networking chapter builds directly on Sections 4, 5, and 15.

---

## 2. The OSI & TCP/IP Models — Encapsulation Layer by Layer

### 2.1 The two models, and why you learn both **[B]**

There are two reference models for the layered stack, and engineers use them together:

- The **OSI model** (Open Systems Interconnection, 1984) has **7 layers**. It's the *teaching* and *vocabulary* model — when someone says "that's a Layer 7 load balancer" or "a Layer 2 switch" or "an L4 vs L7 proxy," they mean OSI layer numbers. You must know the numbers.
- The **TCP/IP model** (a.k.a. the Internet model) has **4 layers** (sometimes drawn as 5). It's the model the actual internet was *built on* — it's pragmatic and matches how the protocols really group.

They line up like this:

| OSI # | OSI layer | TCP/IP layer | Data unit (PDU) | Example protocols | Addressing |
|---|---|---|---|---|---|
| 7 | Application | Application | data / message | HTTP, DNS, TLS*, SMTP, gRPC | — |
| 6 | Presentation | Application | data | TLS/SSL, encoding (UTF-8, JSON) | — |
| 5 | Session | Application | data | sockets, RPC session | — |
| 4 | Transport | Transport | **segment** (TCP) / **datagram** (UDP) | TCP, UDP, QUIC | **port** |
| 3 | Network | Internet | **packet** | IP, ICMP, IPsec | **IP address** |
| 2 | Data Link | Link | **frame** | Ethernet, Wi-Fi (802.11), ARP | **MAC address** |
| 1 | Physical | Link | **bit** / symbol | copper, fiber, radio | — |

*TLS is awkward to place — it's often called "Layer 6 (presentation)" because it handles encryption/encoding, but it runs as an application-layer library on top of TCP. Don't lose sleep over it; the point of the layers is the *idea*, not bureaucratic placement.

**Why both?** OSI gives you the precise vocabulary (L2/L3/L4/L7) that the whole industry speaks. TCP/IP gives you the honest grouping of how real protocols stack. In practice you'll think in **five** practical layers: **Link (L1+L2), Internet (L3), Transport (L4), and Application (L5-L7)** — which is exactly how the rest of this guide is organized.

### 2.2 What each layer's job actually is **[B]**

Walking top to bottom, here is the single responsibility of each layer — memorize the *job*, not the trivia:

- **Application (7)** — the actual thing you want: fetch a web page (HTTP), resolve a name (DNS), send mail (SMTP). This is where your code lives.
- **Presentation (6)** — translate data into a common form: encryption (TLS), character encoding, (de)serialization. "How the bytes are represented."
- **Session (5)** — establish/maintain/tear down a conversation between two apps. In practice this is mostly absorbed by the transport layer and your socket library.
- **Transport (4)** — get the data **to the right *program*** on the host (via **ports**) and decide the *delivery guarantee*: TCP (reliable, ordered) or UDP (best-effort). **This is the layer most application bugs live near.**
- **Network (3)** — get the packet **to the right *host*** anywhere on Earth, via **IP addresses** and **routing**. End-to-end addressing.
- **Data Link (2)** — get the frame across **one physical hop** to the next device, via **MAC addresses**. Hop-by-hop.
- **Physical (1)** — actually move **bits** as electrical, optical, or radio signals.

A memory hook for the OSI order (7→1): **"All People Seem To Need Data Processing"** (Application, Presentation, Session, Transport, Network, Data link, Physical).

### 2.3 Encapsulation — the Russian-doll mechanism **[B]**

This is the mechanical heart of the whole stack. When you send data, it travels **down** the stack on your machine, and **each layer wraps the data from the layer above in its own header** (and the link layer adds a trailer too). When it arrives, it travels **up** the stack on the other machine, and **each layer reads and strips its own header**, handing the inside to the layer above. Wrapping on the way down is **encapsulation**; unwrapping on the way up is **decapsulation**.

Concretely, your 100-byte HTTP request becomes:

```
                                  ┌──────────────────────────── HTTP request (your data) ─┐
 Application:                     │ GET /index.html HTTP/1.1 ... (≈100 bytes)             │
                                  └───────────────────────────────────────────────────────┘
                       ┌── TCP header (20B): src port, dst port=443, seq#, ack#, flags ──┐
 Transport (segment):  │  [TCP hdr] [ ...... HTTP data ...... ]                          │
                       └────────────────────────────────────────────────────────────────┘
              ┌── IP header (20B): src IP, dst IP, protocol=TCP, TTL ──┐
 Network (packet):     │  [IP hdr] [TCP hdr] [ ...... HTTP data ...... ]                 │
              └─────────────────────────────────────────────────────────────────────────┘
     ┌── Ethernet header (14B): src MAC, dst MAC, type=IPv4 ──┐         ┌─ Eth trailer (4B): CRC ─┐
 Link (frame):    │ [Eth hdr] [IP hdr] [TCP hdr] [ .. HTTP data .. ] [Eth CRC] │
     └────────────────────────────────────────────────────────────────────────────────────────┘
 Physical:         10110100 01101001 11010010 ...  (the frame as bits/signals on the wire)
```

Read that diagram until it's obvious. Three consequences fall out of it immediately:

1. **Overhead is real.** Every packet carries ~14 (Ethernet) + 20 (IP) + 20 (TCP) = **54 bytes of headers minimum**, before your data. For tiny messages this is huge proportional overhead — one reason chatty protocols are slow (Section 16).
2. **Each layer only reads its own header.** A router (a Layer-3 device) reads the **IP header** to decide where to forward, and *never looks inside* the TCP or HTTP data. A switch (Layer-2) reads only the **Ethernet header**. This is why "a firewall that filters by port is operating at Layer 4" and "a load balancer that routes by URL path is operating at Layer 7" — they each peel down to a different depth.
3. **The names finally make sense.** The same bytes are called a **segment** when you're looking at the TCP header, a **packet** when looking at the IP header, and a **frame** when looking at the Ethernet header. People use "packet" loosely for all of them; be precise when it matters.

### 2.4 The MTU and fragmentation — a consequence you'll actually hit **[I]**

The link layer can only carry a frame up to a certain size. For standard Ethernet that **maximum transmission unit (MTU)** is **1500 bytes** of payload. Since the IP + TCP headers eat 40 of those, TCP's usable payload per packet — the **MSS (maximum segment size)** — is typically **1460 bytes**.

What happens when IP must send a packet bigger than the next link's MTU? Two options:

- **IPv4** can **fragment** the packet — split it into MTU-sized pieces that are reassembled at the destination. Fragmentation is fragile (lose one fragment, lose the whole packet) and a security/performance headache, so it's avoided in practice.
- The better mechanism is **Path MTU Discovery (PMTUD)**: the sender sets the "Don't Fragment" bit; if a router can't forward the packet without fragmenting, it drops it and sends back an **ICMP "Fragmentation Needed"** message telling the sender the smaller MTU. The sender then uses a smaller MSS. **⚡ Gotcha:** firewalls that blindly drop *all* ICMP break PMTUD, causing the infamous "small requests work, large uploads hang forever" bug — a real, common production failure. Don't block ICMP type 3.
- **IPv6 routers never fragment** — only the sending host may, using PMTUD. This is one of IPv6's deliberate simplifications.

**Jumbo frames** (MTU up to ~9000 bytes) exist on data-center LANs to cut per-packet overhead, but every device on the path must agree, so you won't see them on the public internet.

---

## 3. Layer 1 & 2: Physical, Ethernet, MAC, Switches, ARP, Wi-Fi, VLANs

### 3.1 The physical layer in one paragraph **[B]**

Layer 1 is where bits become **signals**: voltage on copper (Ethernet over twisted pair, e.g. Cat 6), light pulses on **fiber** (long distance, immune to electrical interference, the backbone of the internet and submarine cables), or modulated **radio** waves (Wi-Fi, cellular). You rarely program at this layer, but two facts matter to you as an engineer: (1) the physical medium sets the **bandwidth ceiling and base latency** (fiber's ~⅔ light-speed propagation is why a packet can't cross the Atlantic faster than ~30 ms no matter how much you pay), and (2) **bit errors happen** (interference, attenuation), which is *why* upper layers carry checksums and retransmission logic.

### 3.2 The link layer & MAC addresses **[B]**

Layer 2's job is to move a **frame** across **one hop** — one physical network segment — to the next device. Its addressing is the **MAC address** (Media Access Control), a **48-bit** hardware address usually written as six hex bytes: `a4:83:e7:1b:9c:50`. Key properties:

- A MAC address is (ideally) **burned into the network interface card (NIC)** at manufacture. The first 3 bytes are the **OUI** (Organizationally Unique Identifier) identifying the vendor; you can look a MAC's vendor up from the OUI.
- MAC addresses are **flat and local** — they have no hierarchy and no geography. They identify a NIC, not a location. This is precisely why we *also* need IP addresses: you can't route across the world using flat MAC addresses, because a router would need a table of every NIC on Earth. (Hierarchy is what makes routing scale — Section 4.)
- `ff:ff:ff:ff:ff:ff` is the **broadcast** MAC — "every device on this segment." Used by ARP (below).
- **MAC addresses can be spoofed in software** (your OS can set any MAC). So never use a MAC as a security identity; it's a convenience, not an authenticator.

**See your own:** `ip link` (Linux), `ipconfig /all` (Windows), `ifconfig` (macOS/old Linux).

### 3.3 Switches vs hubs — how a LAN actually moves frames **[B/I]**

Within one local network (a **LAN** — your home, an office floor, a cloud subnet), frames are moved by a **switch**. Understanding the switch demystifies the LAN:

- A **hub** (obsolete) was a dumb repeater: a frame in one port went out *every* other port. Everyone saw everyone's traffic (a security and performance disaster).
- A **switch** is smart. It maintains a **MAC address table** (CAM table) mapping `MAC address → physical port`. It **learns** by watching the *source* MAC of every frame ("ah, device `aa:bb:..` is on port 7"). When a frame arrives for a known destination MAC, the switch sends it out **only that one port** — a private conversation. If the destination is unknown or broadcast, it **floods** (sends out all ports) until it learns.

The set of devices that share a broadcast (all reachable by one Layer-2 broadcast) is a **broadcast domain**. A switch forwards broadcasts to its whole domain; a **router** does *not* forward broadcasts — which is the boundary between "the local network" and "everywhere else."

> **Security note:** **MAC flooding** attacks overflow a switch's CAM table so it fails open and floods all traffic (letting an attacker sniff). **ARP spoofing** (next) is the more common LAN attack. Switched LANs are *not* automatically private.

### 3.4 ARP — gluing IP to MAC **[I]**

Here's a problem you might not have noticed: your computer wants to send an IP packet to `192.168.1.20` on your LAN, but the *link layer* can only deliver frames to **MAC** addresses. How does it learn the MAC that goes with that IP? **ARP — the Address Resolution Protocol (RFC 826).**

The mechanism is delightfully simple:

1. Your host broadcasts an **ARP request** to the whole LAN (`dst MAC = ff:ff:ff:ff:ff:ff`): *"Who has IP `192.168.1.20`? Tell `192.168.1.10`."*
2. Every device sees it; only `192.168.1.20` replies (unicast) with an **ARP reply**: *"`192.168.1.20` is at MAC `aa:bb:cc:dd:ee:ff`."*
3. Your host caches that mapping in its **ARP table** (`ip neigh` on Linux, `arp -a` on Windows/macOS) and now builds the Ethernet frame with the right destination MAC.

ARP only works *within one LAN* (it's a broadcast). To reach an IP that's **not** local, your host doesn't ARP for the destination at all — it ARPs for its **default gateway** (the router) and sends the frame there, trusting the router to forward it onward (Section 5.1). That single insight — *local destinations you ARP for directly; remote destinations you hand to the gateway* — is the link between Layer 2 and Layer 3.

> **Security note — ARP spoofing / poisoning:** ARP has no authentication. An attacker on your LAN can send forged ARP replies claiming *"the gateway's IP is at *my* MAC,"* causing your traffic to flow through the attacker — a classic **man-in-the-middle (MITM)** setup. Defenses: **Dynamic ARP Inspection** on managed switches, static ARP entries for critical hosts, and — the real defense — **end-to-end encryption (TLS)** so that even a successful MITM sees only ciphertext. (IPv6 replaces ARP with **NDP**, the Neighbor Discovery Protocol, which has the same trust problem unless **SEND** is used.)

### 3.5 Wi-Fi (802.11) — the link layer over radio **[I]**

Wi-Fi is a Layer 1/2 technology (the **IEEE 802.11** family) that replaces the wire with radio but presents the *same* Ethernet-like frame interface to IP above it — which is why your TCP/IP stack doesn't change when you switch from cable to Wi-Fi. Things an engineer should know:

- **It's a shared, half-duplex medium.** Radio is broadcast by nature; devices can't transmit and listen at once and must avoid talking over each other. Wired Ethernet long ago went full-duplex via switches; Wi-Fi still coordinates access (CSMA/CA — listen before you talk, with collision *avoidance*), which is why Wi-Fi throughput and latency are more variable than wired.
- **Generations:** Wi-Fi 5 (802.11ac), **Wi-Fi 6/6E** (802.11ax, adds the 6 GHz band), **Wi-Fi 7** (802.11be) is current in 2026. Higher numbers add bandwidth and efficiency, not new IP behavior.
- **Security is at this layer:** **WPA2** (AES-CCMP) and **WPA3** (current best, adds SAE / forward secrecy and protects against offline password guessing). **Never** run open or WEP/WPA networks. But remember: Wi-Fi encryption protects only the *radio hop* to the access point — beyond it your traffic is as exposed as any other, which is again why **TLS end-to-end** is non-negotiable on untrusted networks (coffee-shop Wi-Fi).

### 3.6 VLANs — slicing one switch into many networks **[I/A]**

A **VLAN (Virtual LAN, IEEE 802.1Q)** lets one physical switch host *multiple isolated Layer-2 networks*. Each frame can carry a **VLAN tag** (a 12-bit VLAN ID, 1–4094) inserted into the Ethernet header. The switch treats each VLAN as a separate broadcast domain — devices on VLAN 10 cannot reach VLAN 20 at Layer 2 even on the same switch; they can only talk *through a router* (which can then apply firewall rules between them).

Why you care as a software engineer: **network segmentation** is a core security control. Production, staging, database, and management traffic are routinely put on separate VLANs/subnets so that a compromise in one tier can't freely reach another. The cloud equivalent is **VPCs and subnets** (Section 15) — same idea, virtualized. A **trunk** port carries many tagged VLANs between switches; an **access** port carries one untagged VLAN to an end device.

---

## 4. Layer 3: IP — Addressing, Subnetting/CIDR, IPv6

### 4.1 What IP does and why it's the "narrow waist" **[B]**

The **Internet Protocol** is the layer that makes the *inter*-net — a network of networks — actually work. Its job: give every host a **globally meaningful address** and define a **packet** format that any network can carry, so a packet can travel across dozens of different physical networks (your Wi-Fi, your ISP's fiber, a submarine cable, a data-center switch) and still find its destination.

IP is famously the **"narrow waist"** of the internet: *everything* runs over IP (HTTP, DNS, video, games), and IP runs over *everything* (Ethernet, Wi-Fi, fiber, cellular). That hourglass shape — many apps on top, many links on the bottom, one IP in the middle — is *the* reason the internet could grow without central coordination. Two properties define IP and explain almost everything about it:

- **IP is connectionless and best-effort.** Each packet is routed independently; IP makes **no promise** of delivery, order, or no-duplication. A packet can be dropped, delayed, reordered, or duplicated, and IP won't tell you. (Reliability is TCP's job, one layer up — Section 6. This separation is deliberate: it keeps the routers in the middle simple and dumb, pushing complexity to the endpoints — the **end-to-end principle**.)
- **IP addresses are hierarchical**, unlike flat MAC addresses. The hierarchy (network part + host part) is what lets routers hold a route to a *block* of addresses rather than to every individual host — the only reason a routing table can fit a planet's worth of devices.

### 4.2 IPv4 addressing **[B]**

An **IPv4** address is **32 bits**, written as four **octets** (bytes) in decimal, dotted: `192.168.1.10`. Each octet is 0–255, so the total space is 2³² ≈ **4.3 billion** addresses — which sounds like a lot but ran out years ago (the reason for NAT and IPv6, below).

Every IPv4 address is split into a **network portion** and a **host portion**. The split is defined by the **subnet mask** (or its modern CIDR form). The network portion says "which network," the host portion says "which device on that network." Devices on the *same* network can talk directly at Layer 2 (ARP + switch); to reach a *different* network you must go through a router.

**Special and reserved ranges you must recognize on sight:**

| Range | CIDR | Meaning |
|---|---|---|
| `10.0.0.0`–`10.255.255.255` | `10.0.0.0/8` | **Private** (RFC 1918) — not routable on the internet |
| `172.16.0.0`–`172.31.255.255` | `172.16.0.0/12` | **Private** (RFC 1918) |
| `192.168.0.0`–`192.168.255.255` | `192.168.0.0/16` | **Private** (RFC 1918) — home/office LANs |
| `127.0.0.0/8` | `127.0.0.1` | **Loopback** — "this machine itself" (localhost) |
| `169.254.0.0/16` | — | **Link-local** (APIPA) — self-assigned when DHCP fails |
| `100.64.0.0/10` | — | **Carrier-grade NAT** (ISP-internal) |
| `0.0.0.0` | — | "any/unspecified" (bind to all interfaces; default route) |
| `255.255.255.255` | — | Limited broadcast |
| `224.0.0.0/4` | — | **Multicast** |

The **private ranges** matter enormously in practice: your laptop's `192.168.x.x` or a container's `172.17.x.x` is *not* unique on the internet — millions of LANs reuse them. They only work because of **NAT** (Section 5.3) translating them to a public address at the edge.

### 4.3 Subnet masks, CIDR & subnetting — the part everyone fears (and shouldn't) **[I]**

A **subnet mask** marks which bits of an address are the network part: a `1` bit means "network," a `0` bit means "host." `255.255.255.0` in binary is twenty-four `1`s then eight `0`s — so the first 24 bits are network, the last 8 are host. **CIDR notation** writes that compactly as the number of network bits after a slash: `/24`. So `192.168.1.10/24` means "address `192.168.1.10`, on the network whose first 24 bits are fixed."

From the prefix length you can derive everything:

- **Host bits** = 32 − prefix. A `/24` has 8 host bits → 2⁸ = **256** addresses in the block.
- **Usable hosts** = 2^(host bits) − 2. You subtract two because the **all-zeros host** is the **network address** (names the network itself) and the **all-ones host** is the **broadcast address** (every host on it). So a `/24` has **254** usable host addresses.
- **Network address** = address with all host bits set to 0 (`192.168.1.0`). **Broadcast** = all host bits 1 (`192.168.1.255`).

A reference table you'll use constantly:

| CIDR | Mask | Host bits | Total addrs | Usable hosts | Typical use |
|---|---|---|---|---|---|
| `/8` | `255.0.0.0` | 24 | 16,777,216 | 16,777,214 | Huge block (e.g. all of `10.0.0.0/8`) |
| `/16` | `255.255.0.0` | 16 | 65,536 | 65,534 | Large network / VPC |
| `/24` | `255.255.255.0` | 8 | 256 | 254 | A typical LAN / subnet |
| `/25` | `255.255.255.128` | 7 | 128 | 126 | Half a /24 |
| `/26` | `255.255.255.192` | 6 | 64 | 62 | Quarter of a /24 |
| `/27` | `255.255.255.224` | 5 | 32 | 30 | Small subnet |
| `/30` | `255.255.255.252` | 2 | 4 | 2 | A point-to-point link (2 routers) |
| `/31` | `255.255.255.254` | 1 | 2 | 2* | Point-to-point (RFC 3021, no net/bcast) |
| `/32` | `255.255.255.255` | 0 | 1 | 1 | A single host (e.g. a firewall rule) |

**Subnetting** is just borrowing host bits to make more, smaller networks. Splitting a `/24` into two `/25`s gives you `192.168.1.0/25` (hosts `.1`–`.126`) and `192.168.1.128/25` (hosts `.129`–`.254`). You do this to **segment** a network (separate broadcast domains, apply firewall rules between segments, organize by team/tier). **CIDR's whole point** is that, unlike the old rigid "Class A/B/C" system, you can pick *any* prefix length to size a block to its need — which both conserves addresses and lets routers **aggregate** many small routes into one summary route (Section 5).

**The one calculation to internalize:** "Is host X on the same subnet as host Y?" Answer: apply the mask (bitwise-AND each address with the mask) and compare the network parts. If they're equal, same subnet → talk directly via ARP. If not, different subnet → must route through the gateway. *This is the exact computation your OS does for every single packet it sends.*

```bash
# Practical CIDR math from the command line (the `ipcalc` tool, or Python):
ipcalc 192.168.1.10/26
#  Network:   192.168.1.0/26
#  HostMin:   192.168.1.1
#  HostMax:   192.168.1.62
#  Broadcast: 192.168.1.63

python3 - <<'PY'
import ipaddress
net = ipaddress.ip_network("192.168.1.0/26")
print(net.network_address, net.broadcast_address, net.num_addresses)   # 192.168.1.0 192.168.1.63 64
print(list(net.hosts())[:3])                                           # first 3 usable hosts
# Is 192.168.1.40 in this subnet?
print(ipaddress.ip_address("192.168.1.40") in net)                     # True
PY
```

### 4.4 How a host gets its IP: static, DHCP, link-local **[B/I]**

An address gets onto an interface one of three ways:

- **Static** — you configure it by hand (servers, routers, infrastructure). Predictable, no dependency, but manual.
- **DHCP (Dynamic Host Configuration Protocol)** — the host **leases** an address automatically from a DHCP server. The exchange is **DORA**: **D**iscover (client broadcasts "anyone have an address for me?"), **O**ffer (server proposes one), **R**equest (client asks for that one), **A**ck (server confirms, with a lease time). DHCP also hands the client its **subnet mask, default gateway, and DNS servers** — everything needed to actually use the network. This is how virtually every laptop, phone, and home device gets configured.
- **Link-local / APIPA** (`169.254.x.x`) — self-assigned when DHCP fails; only works on the local segment. Seeing a `169.254` address is a classic "DHCP isn't working" symptom.

> **Security note:** DHCP, like ARP, is unauthenticated — a **rogue DHCP server** can hand clients a malicious gateway/DNS and MITM them. **DHCP snooping** on managed switches mitigates it.

### 4.5 IPv6 — the address space that doesn't run out **[I]**

IPv4's 4.3 billion addresses were exhausted (IANA allocated the last blocks in 2011). The permanent fix is **IPv6**, with **128-bit** addresses — 2¹²⁸ ≈ **340 undecillion**, enough to give every grain of sand its own subnet. You must be literate in it.

- **Format:** eight groups of four hex digits, colon-separated: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`. Compression rules make it bearable: drop leading zeros in a group, and replace **one** run of all-zero groups with `::`. So that address is `2001:db8:85a3::8a2e:370:7334`. Loopback is `::1` (the IPv6 `127.0.0.1`); "any" is `::`.
- **No NAT needed, by design.** Every device can have a globally unique address again, restoring the end-to-end model. (Firewalls still provide the security NAT incidentally gave you — see below.)
- **No broadcast; no ARP.** IPv6 uses **multicast** and the **Neighbor Discovery Protocol (NDP)** instead of ARP. Hosts can also **autoconfigure** addresses (SLAAC) without DHCP.
- **Address types:** **Global Unicast** (`2000::/3`, internet-routable), **Link-Local** (`fe80::/10`, every IPv6 interface always has one, used for NDP and local comms), **Unique Local** (`fc00::/7`, the IPv6 "private" range).
- **Dual-stack** is the reality of 2026: hosts run IPv4 *and* IPv6 simultaneously, and clients use **Happy Eyeballs** (RFC 8305) to race both and use whichever connects first. As an engineer, **bind your servers to both** and never assume an address is v4 — e.g. a client IP in your logs may arrive as `::ffff:203.0.113.5` (an IPv4-mapped IPv6 address) or `2001:db8::1`.

> **⚡ Version note:** when you write networking code in 2026, prefer APIs and string handling that are address-family-agnostic. In Go, `net.IP` and `netip.Addr` handle both; in Node, `net.isIP()` returns 4 or 6; storing client IPs in a DB, use a type/column wide enough for IPv6 (`inet` in Postgres, not a 32-bit int).

---

## 5. Routing — How a Packet Crosses the Planet (NAT, BGP)

### 5.1 The routing decision — what every host and router does **[I]**

Every device that sends an IP packet — your laptop included — runs the same tiny algorithm for each packet, consulting its **routing table**:

1. Look at the packet's **destination IP**.
2. Find the **most specific matching route** in the table (longest-prefix match — a `/24` route beats a `/8` route beats the default).
3. If the destination is on a **directly connected** network, deliver it on that interface (ARP for the host, build the frame, send).
4. Otherwise, send it to the **next-hop gateway** listed in the matching route, and let *that* device repeat the whole process.
5. If nothing matches, use the **default route** (`0.0.0.0/0`) — "I don't know where this is; send it to my gateway and let someone smarter figure it out."

Look at your own table:

```bash
ip route                       # Linux
# default via 192.168.1.1 dev wlan0 proto dhcp    ← the default route: everything unknown → 192.168.1.1
# 192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.10   ← directly connected LAN

route print                    # Windows
netstat -rn                    # macOS / portable
```

The packet then **hops** from router to router. Here is the part that ties Layers 2 and 3 together and is worth re-reading: at every hop, the router **strips the old Ethernet frame, looks at the IP destination, decides the next hop, and wraps the packet in a brand-new Ethernet frame** addressed (by MAC) to that next hop. **The source/destination IP never change; the source/destination MAC change at every single hop.** The IP addresses are the end-to-end "from/to"; the MAC addresses are the hop-by-hop "hand this to the next guy."

### 5.2 TTL, ICMP, and how `traceroute` exploits them **[I]**

Each IP packet carries a **TTL (Time To Live)** field — an integer (often starting at 64 or 128) that **every router decrements by 1**. If TTL hits **0**, the router **discards** the packet and sends an **ICMP "Time Exceeded"** message back to the source. This exists to kill packets stuck in a routing loop so they don't circle forever.

`traceroute` (Linux/macOS) / `tracert` (Windows) is a beautiful hack on this mechanism: it sends packets with TTL=1 (the first router replies "time exceeded" — now you know hop 1), then TTL=2 (hop 2 replies), and so on, mapping the entire path one router at a time. **ICMP (Internet Control Message Protocol)** is IP's companion "error and diagnostics" protocol — it carries `ping` (Echo Request/Reply), "Destination Unreachable," "Time Exceeded," and "Fragmentation Needed" (the PMTUD message from Section 2.4). ICMP is *not* TCP or UDP — it's a separate Layer-3 protocol. (Section 13 shows the tools in action.)

### 5.3 NAT — how millions of devices share one public IP **[I]**

Because IPv4 ran out, your home or office almost certainly uses **private addresses** (`192.168.x.x`) internally and **one public address** facing the internet, with the router performing **NAT (Network Address Translation)** — specifically **NAPT / PAT (Port Address Translation)**, the kind everyone actually means.

The mechanism: when your laptop (`192.168.1.10:51514`) connects out to a server (`93.184.216.34:443`), the NAT router **rewrites the packet's source** to its *own* public IP and a chosen source port (`203.0.113.7:60001`), and records the mapping in a **translation table**:

```
   inside                         NAT table                         outside
   192.168.1.10:51514  ───▶  (203.0.113.7:60001 ⇄ 192.168.1.10:51514)  ───▶  93.184.216.34:443
   192.168.1.22:51515  ───▶  (203.0.113.7:60002 ⇄ 192.168.1.22:51515)  ───▶  93.184.216.34:443
```

When the reply comes back to `203.0.113.7:60001`, the router looks up the table, rewrites the destination back to `192.168.1.10:51514`, and forwards it inside. Many devices, one public IP, disambiguated by **port**. Consequences every engineer hits:

- **NAT is why "connecting out" just works but "connecting in" doesn't.** Outbound creates a mapping automatically; inbound has no mapping, so unsolicited inbound traffic is dropped. This is incidental security (a stateful firewall by accident) but it's *why you can't just run a server at home and have the world reach it* without **port forwarding** (a manual inbound mapping) or a tunnel.
- **NAT breaks the end-to-end principle** and complicates peer-to-peer (two devices both behind NAT can't directly connect). Solutions: **STUN** (discover your public mapping), **TURN** (relay through a server), and **ICE** (try everything) — the machinery behind WebRTC video calls and online games. **Hole punching** is the clever trick that makes P2P work through NAT.
- **CGNAT (Carrier-Grade NAT)** means your ISP NATs you *too* (you may not even have your own public IP), nesting NAT inside NAT — increasingly common and increasingly painful for P2P.
- **IPv6 removes the need for NAT entirely** (everyone gets a public address) — but you still firewall inbound traffic deliberately, rather than relying on NAT's accidental protection.

### 5.4 Inside vs between organizations: IGP vs BGP **[A]**

How do routers *learn* the routes in their tables? Two regimes:

- **Inside one organization/network** (an **Autonomous System**, AS), routers share routes automatically with an **interior gateway protocol (IGP)** — **OSPF** or **IS-IS** (link-state: every router learns the whole topology and computes shortest paths with Dijkstra) or the older **RIP**. You rarely touch these as a software engineer, but know the term.
- **Between organizations** — between your ISP, Google, Cloudflare, a university, every network on Earth — the glue is **BGP (Border Gateway Protocol)**, "the routing protocol of the internet." Each AS has a number (ASN) and **announces** which IP prefixes it can reach; BGP propagates these announcements globally so every network learns "to reach `93.184.216.0/24`, go toward AS-15133." BGP chooses paths by policy (business relationships, path length), not pure speed.

Why a software engineer should care about BGP: **it's unauthenticated by default and it's how the internet breaks at scale.** A misconfigured or malicious AS can announce prefixes it doesn't own — a **BGP hijack** — and globally reroute traffic to itself (famous outages and interceptions have happened this way). **Route leaks** cause massive outages (e.g. a network accidentally announcing it's the best path to everywhere). The defenses rolling out are **RPKI** (cryptographically validating that an AS is authorized to announce a prefix). When "half the internet went down for an hour," BGP or DNS is very often the culprit.

---

## 6. Layer 4: TCP vs UDP — Ports, Reliability, Flow & Congestion Control

### 6.1 What the transport layer adds: ports & process-to-process delivery **[B]**

IP gets a packet to the right *host*. But a host runs many programs at once — a browser, a mail client, an SSH session, three servers. **Which program gets the data?** That's the transport layer's first job, solved with **port numbers**: a 16-bit number (0–65535) identifying an endpoint *within* a host. The full coordinate of one end of a connection is the **socket**: `IP address : port`. A connection is uniquely identified by the **4-tuple**:

```
   (source IP, source port, destination IP, destination port)
```

Two facts that fall out of the 4-tuple and explain a lot:

- A server listens on a **well-known port** (HTTP 80, HTTPS 443, SSH 22, DNS 53, Postgres 5432). Clients pick a random high **ephemeral port** (typically 49152–65535) for their end. So thousands of clients can all hit a server's port 443 — each connection is distinct because the *client* IP:port differs.
- **One server port can hold millions of simultaneous connections** because what's unique is the whole 4-tuple, not just the server port. (This is the C10k/C10M story from the **Nginx** guide.)

**Port reference you should know cold:**

| Port | Protocol | Service |
|---|---|---|
| 20/21 | TCP | FTP (data/control) — see **FTP** guide |
| 22 | TCP | SSH / SFTP |
| 25, 587, 465 | TCP | SMTP (mail send) |
| 53 | UDP/TCP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110/143/993 | TCP | POP3 / IMAP / IMAPS |
| 123 | UDP | NTP (time) |
| 443 | TCP **& UDP** | HTTPS (TCP for H1/H2, **UDP for HTTP/3/QUIC**) |
| 3306 / 5432 / 6379 / 27017 | TCP | MySQL / PostgreSQL / Redis / MongoDB |
| 8080 / 3000 | TCP | Common dev/app server ports |

Ports 0–1023 are **"well-known"/privileged** (binding them historically requires root/admin) — a reason your dev server runs on 3000/8080, not 80.

### 6.2 UDP — the minimal transport **[B/I]**

**UDP (User Datagram Protocol)** is the thinnest possible layer over IP: it adds *only* ports and a checksum. It is **connectionless** and **unreliable** — fire a datagram and forget it. No handshake, no acknowledgments, no retransmission, no ordering, no congestion control. Its entire header is **8 bytes** (source port, dest port, length, checksum).

Why would you ever want *unreliable*? Because for some workloads, **the cost of reliability is worse than the loss**:

- **Real-time media** (voice/video calls, live streams, games): if a packet of audio from 200 ms ago is lost, you do *not* want TCP to stop everything and re-send it — that stale audio is useless and the retransmission would add lag. Better to skip it and play the next packet. UDP lets the *application* decide what to do about loss.
- **Small request/response where you build your own logic**: **DNS** queries are one UDP packet out, one back — far cheaper than a TCP handshake. (DNS falls back to TCP for big responses.)
- **You want to build your own transport**: **QUIC** (and thus HTTP/3) runs over UDP precisely so it can implement *better* reliability/multiplexing in userspace without being stuck with TCP's kernel implementation (Section 9.4).

UDP's tradeoff in one line: **you get speed and control, and you take responsibility for everything TCP would have done for you.**

### 6.3 TCP — the reliable, ordered byte stream **[B/I]**

**TCP (Transmission Control Protocol)** is the workhorse: it turns IP's unreliable packet delivery into a **reliable, ordered, full-duplex stream of bytes** between two endpoints. When you read from a TCP socket you get exactly the bytes the other side sent, in order, with no gaps or duplicates — or you get an error. It achieves this with sequence numbers, acknowledgments, retransmission, and flow/congestion control, all of which we'll unpack. TCP's guarantees:

- **Connection-oriented** — a handshake establishes shared state before any data flows.
- **Reliable** — every byte is acknowledged; lost data is retransmitted.
- **Ordered** — each byte has a **sequence number**; the receiver reassembles in order even if packets arrive scrambled.
- **Flow-controlled** — the receiver can tell the sender to slow down so it isn't overwhelmed.
- **Congestion-controlled** — the sender backs off when the *network* is overwhelmed (this is what prevents internet collapse).

The cost: a setup handshake (latency before any data), per-connection state in the kernel, and **head-of-line blocking** (one lost packet stalls *everything* behind it, because order is guaranteed — Section 9.3).

### 6.4 The TCP three-way handshake — opening a connection **[B/I]**

Before any data, TCP synchronizes **sequence numbers** with a three-step exchange (SYN, SYN-ACK, ACK). Each side picks a random **Initial Sequence Number (ISN)** and tells the other:

```
   CLIENT                                                          SERVER
     │                                                                │
     │ ───────── SYN, seq=x ─────────────────────────────────────▶   │   "let's talk; my seq starts at x"
     │                                                                │
     │ ◀──────── SYN-ACK, seq=y, ack=x+1 ─────────────────────────    │   "ok; my seq is y; I got your x"
     │                                                                │
     │ ───────── ACK, ack=y+1 ───────────────────────────────────▶   │   "got your y; we're connected"
     │                                                                │
     │ ════════ connection ESTABLISHED — data can flow ══════════    │
```

Two things to take from this:

- **It costs one full round-trip (1 RTT) before a single byte of your data is sent.** On a 100 ms RTT link, that's 100 ms of pure setup latency — and with TLS on top, more (Section 10). This is *why* connection reuse (HTTP keep-alive, connection pools) matters so much for performance.
- The random ISN matters for **security**: predictable sequence numbers historically allowed **TCP spoofing/injection** attacks. **SYN floods** (sending many SYNs and never completing the handshake to exhaust a server's half-open connection table) are a classic DoS; **SYN cookies** are the defense.

Closing is a **four-way** exchange (`FIN`/`ACK` each direction, since each side closes its half independently), after which the initiator sits in **`TIME_WAIT`** for ~2× the max segment lifetime to absorb stray packets. **⚡ Gotcha:** a busy server (or load tester) that opens/closes many short connections can pile up thousands of `TIME_WAIT` sockets and exhaust ephemeral ports — a real scaling wall solved by connection reuse, not by disabling `TIME_WAIT`.

### 6.5 The TCP segment header — the fields that make it work **[I]**

You don't memorize the layout, but recognizing the key fields makes packet captures (Section 13) readable:

| Field | Purpose |
|---|---|
| Source / Dest port | The endpoints (with IPs, the 4-tuple) |
| **Sequence number** | Byte offset of this segment's first byte in the stream |
| **Acknowledgment number** | "I've received everything up to byte N; send N next" (cumulative) |
| **Flags** | `SYN` (start), `ACK` (acknowledging), `FIN` (graceful close), `RST` (abort), `PSH` (deliver now), `URG` |
| **Window size** | Flow control: "I have room for this many more bytes" (Section 6.6) |
| Checksum | Detects corruption |
| Options | MSS, **window scaling**, SACK, timestamps |

A **`RST`** is worth singling out: it means "this connection is invalid, drop it immediately." Connecting to a port with **nothing listening** gets you a `RST` → "connection refused." A `RST` mid-connection means the other side abandoned it abruptly (crash, firewall, timeout).

### 6.6 Reliability, flow control & congestion control — the magic **[A]**

This is TCP's intellectual core, and the difference between flow control and congestion control trips up most people:

**Reliability (ACKs + retransmission).** The receiver acknowledges bytes it has received (cumulative ACK: "got everything through byte N"). If the sender doesn't get an ACK within the **RTO (retransmission timeout)** — estimated from measured RTT — it **retransmits**. **Fast retransmit** is the optimization: three **duplicate ACKs** ("still waiting for byte N" repeated) signal a loss *before* the timer fires, so the sender resends immediately. **SACK (Selective ACK)** lets the receiver say "I have N, and also bytes N+1000..N+2000" so the sender retransmits only the actual gap, not everything after it.

**Flow control (the receive window) — don't overwhelm the *receiver*.** Every ACK carries a **window size**: how much buffer space the receiver has free. The sender may have at most that many *unacknowledged* bytes in flight. If the receiver's application is slow to read, its buffer fills, the advertised window shrinks toward zero, and the sender pauses. This protects a *slow receiver* from a *fast sender*. (Window scaling, a TCP option, lets the window exceed the original 64 KB cap — essential for high-bandwidth links; see the BDP in Section 16.)

**Congestion control — don't overwhelm the *network*.** This is the famous part, and it's what keeps the internet from collapsing. The network in between has no idea how fast to let you go, and **packet loss is TCP's signal that the network is congested.** The sender maintains a **congestion window (cwnd)** — a self-imposed limit, *separate* from the receiver's window — and the amount it can send is the **minimum** of the two windows. The classic algorithm:

- **Slow start:** begin with a tiny cwnd and **double it every RTT** (exponential growth) to quickly probe how much the network can take.
- **Congestion avoidance:** past a threshold, grow **linearly** (additive increase) to gently push the limit.
- **On loss:** **cut cwnd sharply** (multiplicative decrease — e.g. halve it). This **AIMD** (Additive Increase, Multiplicative Decrease) "sawtooth" is what shares bandwidth fairly among competing connections.

**⚡ Version note:** modern systems increasingly use **BBR** (Google's congestion control) instead of loss-based algorithms (Reno/CUBIC). BBR models the link's actual **bandwidth and RTT** rather than treating loss as the only congestion signal — much better on lossy links (Wi-Fi, cellular) and long-distance paths. You can often choose it (`net.ipv4.tcp_congestion_control=bbr` on Linux). The takeaway: **TCP throughput is governed by RTT and loss**, which is why a far-away or lossy server feels slow even on a fat pipe (Section 16).

### 6.7 TCP vs UDP — the decision table **[B/I]**

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Handshake first (1 RTT) | None — just send |
| Reliability | Guaranteed (ACK + retransmit) | None (app's problem) |
| Ordering | Guaranteed | None |
| Flow/congestion control | Yes | No |
| Header overhead | 20+ bytes | 8 bytes |
| Head-of-line blocking | Yes (one loss stalls the stream) | No |
| Best for | Web, APIs, DBs, file transfer, email — anything that must be *correct* | Real-time media, gaming, DNS, VPNs, QUIC — anything *latency-sensitive* that tolerates loss |
| Used by | HTTP/1.1 & /2, TLS, SSH, SMTP, SQL | DNS, DHCP, NTP, VoIP, **QUIC/HTTP/3**, WireGuard |

The one-line heuristic: **TCP when correctness matters more than the last millisecond; UDP when the last millisecond matters more than perfect delivery (or when you'll build your own reliability on top, as QUIC does).**

---

## 7. Sockets & the Socket API — Where Code Meets the Network

### 7.1 What a socket actually is **[I]**

A **socket** is the programming abstraction the operating system gives you for a network endpoint — the API boundary between *your code* and *the kernel's TCP/IP stack*. You write to a socket like a file; the kernel handles segmentation, sequence numbers, retransmission, routing, and frames. The socket API (the "Berkeley sockets" API from 1983) is so foundational that *every* language's networking — Go's `net`, Node's `net`/`dgram`, Python's `socket`, Java's `Socket` — is a thin wrapper over it. Learn the call sequence once and you understand them all.

A socket is identified by the OS as a **file descriptor** (on Unix, sockets *are* files — `everything is a file`). A *connected* TCP socket corresponds to the 4-tuple from Section 6.1.

### 7.2 The server vs client call sequence — the canonical dance **[I]**

The asymmetry between the two sides is the thing to memorize:

```
   SERVER                                  CLIENT
   socket()      create the endpoint       socket()       create the endpoint
   bind()        claim IP:port (e.g. :443)
   listen()      mark it passive, set backlog queue
   accept()  ◀── blocks until a client connects
        │                                   connect()  ──▶ initiates the 3-way handshake
        │   ◀═══════ handshake ═══════════▶
   (accept returns a NEW socket for THIS client)
   read()/write() ◀══ data flows ══▶        write()/read()
   close()                                  close()
```

The subtle, important detail: **`accept()` returns a brand-new socket** dedicated to that one client, while the original **listening socket** stays open to accept more clients. So a server has *one* listening socket and *N* connected sockets (one per client). How the server handles those N concurrently — a thread each (simple, doesn't scale), or an **event loop** over all of them (the C10k solution in the **Nginx** and **Node.js** guides) — is the central design question of server performance.

### 7.3 A raw TCP echo server & client (so you see the syscalls) **[I]**

Here is the same dance in three languages, deliberately low-level so the socket calls are visible (in real apps you'd use `net/http`, Express, etc.). Start with Go:

```go
// echo-server.go — a minimal TCP server. Run, then: nc localhost 9000
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	// net.Listen does socket()+bind()+listen() in one call.
	// "tcp" picks TCP; ":9000" binds all interfaces on port 9000.
	ln, err := net.Listen("tcp", ":9000")
	if err != nil {
		panic(err) // e.g. "address already in use" if the port is taken
	}
	defer ln.Close()
	fmt.Println("listening on :9000")

	for {
		// Accept() blocks until a client connects, then returns a
		// dedicated net.Conn for that client (the per-client socket).
		conn, err := ln.Accept()
		if err != nil {
			continue
		}
		// One goroutine per connection — Go's cheap concurrency makes the
		// "thread-per-connection" model actually scale here (see Go guide).
		go handle(conn)
	}
}

func handle(conn net.Conn) {
	defer conn.Close()
	// Read lines from the client and echo them back uppercased.
	scanner := bufio.NewScanner(conn) // conn is an io.Reader/Writer — like a file
	for scanner.Scan() {
		line := scanner.Text()
		fmt.Fprintf(conn, "ECHO: %s\n", line) // write back to the socket
	}
}
```

The same idea in **Node.js** (the event loop is implicit — no threads):

```js
// echo-server.js — node echo-server.js
import net from "node:net";

// createServer registers a connection callback; Node's event loop handles
// thousands of sockets on one thread (see the Node.js guide).
const server = net.createServer((socket) => {
  // `socket` is the per-client connection (a Duplex stream).
  console.log("client:", socket.remoteAddress, socket.remotePort); // the 4-tuple's far end
  socket.setEncoding("utf8");
  socket.on("data", (chunk) => socket.write(`ECHO: ${chunk}`)); // echo back
  socket.on("end", () => console.log("client disconnected"));
  socket.on("error", (e) => console.error("socket error:", e.message));
});

server.listen(9000, () => console.log("listening on :9000"));
```

And a **Python** client showing the explicit `socket()/connect()` calls:

```python
# echo-client.py — python echo-client.py
import socket

# AF_INET = IPv4, SOCK_STREAM = TCP. (AF_INET6 for IPv6, SOCK_DGRAM for UDP.)
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(("127.0.0.1", 9000))      # the 3-way handshake happens here
    s.sendall(b"hello network\n")        # write bytes to the stream
    data = s.recv(1024)                  # read up to 1024 bytes back
    print("got:", data.decode())         # -> ECHO: hello network
# leaving the `with` block calls close() -> the 4-way teardown
```

> **The single most important socket gotcha:** TCP is a **byte stream, not a message stream.** `recv(1024)` does **not** correspond to one `send()`. Two `send()`s of 10 bytes may arrive as one 20-byte `recv`, or split across reads. The network can merge or split your writes freely. **You must define your own message framing** — newline-delimited, length-prefixed, or a real protocol. Forgetting this causes the classic "it works locally but corrupts under load / over the real internet" bug. (UDP, by contrast, *is* message-oriented: one `sendto` = one `recvfrom`.)

### 7.4 Blocking vs non-blocking, and why servers use event loops **[I/A]**

By default socket calls **block**: `accept()` waits for a client, `read()` waits for data. One thread can therefore serve one slow client at a time. Three models scale beyond that:

- **Thread/process per connection** — simple, but each thread costs memory and context-switching; collapses at ~10k connections (the C10k problem; see **Nginx** §1). Fine for modest loads; Go's cheap goroutines stretch this model a long way.
- **I/O multiplexing / event loop** — one thread watches *thousands* of sockets via `epoll`(Linux)/`kqueue`(BSD)/`IOCP`(Windows), processing only the ready ones. This is **Node.js**, **Nginx**, **Redis**, and Go's runtime under the hood. Non-blocking sockets + readiness notification = massive concurrency on few threads.
- **Async I/O** (`io_uring` on modern Linux) — the newest, highest-performance model.

You usually don't write `epoll` by hand — your runtime does it. But knowing it's there explains *why* "don't block the event loop" is the cardinal rule in Node, and why one Nginx worker handles tens of thousands of connections.

---

## 8. DNS — The Internet's Phone Book

### 8.1 Why DNS exists & the mental model **[B]**

Humans use names (`example.com`); the network routes on IP addresses (`93.184.216.34` / `2606:2800:21f:...`). **DNS (Domain Name System)** is the globally distributed database that translates names → addresses (and more). It's arguably the most important "invisible" system on the internet: nearly every connection starts with a DNS lookup, and **"it's always DNS"** is a sysadmin meme precisely because so many outages trace back to it.

DNS is a **hierarchical, delegated, cached** system — three properties that make a single namespace serve the entire planet without a central server melting:

- **Hierarchical:** read a domain **right to left**. `www.example.com.` is: the **root** (the trailing `.`), then the **TLD** (`com`), then the **domain** (`example`), then the **subdomain** (`www`). Each level delegates authority for what's below it.
- **Delegated:** the root servers don't know `example.com`'s address — they only know who runs `.com`. The `.com` servers don't know `www.example.com` — they know who runs `example.com`. Authority is handed down the tree.
- **Cached:** answers carry a **TTL**; resolvers cache them so the same lookup doesn't traverse the hierarchy again for minutes/hours. Caching is what makes DNS fast and what makes DNS changes take time to propagate.

### 8.2 The resolution walk — what happens on a cache miss **[B/I]**

When your machine needs `www.example.com` and nothing is cached, a **recursive resolver** (run by your ISP, or a public one like `1.1.1.1`/`8.8.8.8`) does the legwork:

```
   your app ──▶ stub resolver (OS) ──▶ recursive resolver (e.g. 1.1.1.1)
                                            │
        1. ask a ROOT server: "where is .com?"        ◀── "ask the .com TLD servers at ..."
        2. ask a .COM TLD server: "where is example.com?"  ◀── "ask example.com's authoritative NS at ..."
        3. ask example.com's AUTHORITATIVE server: "A record for www?"  ◀── "93.184.216.34, TTL 3600"
                                            │
   your app ◀── "93.184.216.34" ◀──────────┘  (resolver caches it for the TTL)
```

So a cold lookup is up to four round-trips *before* your TCP handshake even begins — which is why DNS latency matters, why resolvers cache aggressively, and why **DNS prefetch**/connection warming are real front-end optimizations. The **stub resolver** is the tiny client in your OS; the **recursive resolver** does the iteration; the **authoritative servers** hold the real answers for a zone.

### 8.3 Record types you must know **[B/I]**

A DNS zone is a set of **resource records**. The ones an engineer uses constantly:

| Type | Maps name to | Notes / use |
|---|---|---|
| **A** | IPv4 address | The classic name→IP |
| **AAAA** | IPv6 address | The IPv6 equivalent ("quad-A") |
| **CNAME** | another name (alias) | "this name is really that name" — **can't coexist with other records or sit at the zone apex** |
| **MX** | mail server name (+ priority) | Where email for the domain goes |
| **TXT** | arbitrary text | **SPF/DKIM/DMARC** (email auth), domain-ownership verification |
| **NS** | authoritative name servers | Delegation — who runs this zone |
| **SOA** | zone metadata | Serial, refresh, TTLs — one per zone |
| **PTR** | name (reverse) | IP→name (reverse DNS), in `in-addr.arpa` |
| **SRV** | host+port for a service | Service discovery (e.g. SIP, LDAP, Minecraft) |
| **CAA** | which CAs may issue certs | TLS issuance control |

**⚡ Modern note:** **ALIAS/ANAME** (provider-specific) or **CNAME flattening** solve the "can't CNAME the apex (`example.com` with no `www`)" problem so you can point a bare domain at a CDN. Know that the **apex/root domain can't be a CNAME** per the spec — a real constraint when configuring DNS for a site.

### 8.4 Seeing it yourself with `dig` **[B/I]**

`dig` (Domain Information Groper) is the canonical tool; `nslookup` is the Windows-friendly alternative; in 2026 `resolvectl query` is common on systemd Linux.

```bash
dig example.com A +short          # just the answer: 93.184.216.34
dig example.com AAAA +short       # IPv6 address
dig example.com MX                # mail servers with priorities
dig +trace example.com           # show the FULL delegation walk: root → .com → authoritative
dig @1.1.1.1 example.com         # ask a SPECIFIC resolver (bypass your default — great for debugging)
dig -x 93.184.216.34             # reverse lookup (PTR)

# Windows:
nslookup example.com
nslookup -type=MX example.com
Resolve-DnsName example.com -Type AAAA   # PowerShell
```

The `dig` answer shows the **TTL** (how long it's cacheable) — invaluable when you've changed a record and are waiting for propagation. `dig +trace` is the single best way to *see* the hierarchy from Section 8.2 actually happen.

### 8.5 DNS, caching & security **[I]**

- **Propagation delay is just caching.** When you change a record, resolvers worldwide keep serving the old value until its TTL expires. **Lower the TTL *before* a planned change** (e.g. to 60s a day ahead), cut over, then raise it again. There is no "force the world to refresh" button.
- **DNS round-robin & geo-DNS** are a crude **load-balancing/failover** tool: return multiple A records (clients pick one) or different answers by client location (steer users to the nearest data center / CDN edge). It's coarse (no health awareness, caching defeats fast failover) but ubiquitous.
- **Security — DNS is a juicy target:** classic DNS is **plaintext UDP, unauthenticated**, so it's spoofable (**cache poisoning** — injecting forged answers to redirect users). Defenses: **DNSSEC** (cryptographic signatures proving an answer is authentic — protects *integrity*, not privacy), and **DoH/DoT** (DNS over HTTPS / over TLS — encrypts the query so it's *private* from the network). **⚡ Version note:** DoH/DoT are widely deployed in 2026 (browsers and OSes default to encrypted DNS in many configs). DNS is also abused for **data exfiltration/tunneling** (smuggling data inside DNS queries) — a thing security teams monitor. And DNS is a favorite **DDoS amplification** vector (a small spoofed query triggers a large response aimed at a victim).

---

## 9. Layer 7: HTTP/1.1 → HTTP/2 → HTTP/3

### 9.1 HTTP, the request/response model **[B]**

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol of the web — and, via REST/JSON APIs, of most backend communication you'll ever write. Its model is dead simple and unchanged across all versions: a **client sends a request**, a **server returns a response**. HTTP is **stateless** — each request is independent; the server remembers nothing between requests unless you add state on top (cookies, sessions, tokens — Section 9.5).

An HTTP/1.1 request is human-readable text:

```http
GET /api/users/42 HTTP/1.1          ← method, path (target), version
Host: api.example.com               ← REQUIRED in HTTP/1.1 (virtual hosting: many sites, one IP)
Accept: application/json            ← what the client wants back
Authorization: Bearer eyJhbGci...   ← credentials (see Better Auth / JWT guides)
User-Agent: curl/8.5.0
                                    ← blank line = end of headers
(optional request body goes here, e.g. for POST/PUT)
```

And the response:

```http
HTTP/1.1 200 OK                     ← version, status code, reason
Content-Type: application/json      ← what the body is
Content-Length: 54                  ← body size in bytes
Cache-Control: max-age=300          ← caching policy (see CDN section)
                                    ← blank line
{"id":42,"name":"Ada","email":"ada@example.com"}   ← the body
```

The anatomy never changes; only how those bytes are *framed and transported* changes between HTTP/1.1, /2, and /3.

### 9.2 Methods, status codes & headers — the working vocabulary **[B]**

**Methods** (the verb — what to do with the resource):

| Method | Meaning | Safe? | Idempotent? |
|---|---|---|---|
| `GET` | Read a resource | yes | yes |
| `HEAD` | Like GET but headers only | yes | yes |
| `POST` | Create / submit (non-idempotent action) | no | **no** |
| `PUT` | Replace a resource wholesale | no | yes |
| `PATCH` | Partially update | no | no |
| `DELETE` | Remove a resource | no | yes |
| `OPTIONS` | Capabilities / **CORS preflight** | yes | yes |

*Safe* = no side effects (won't change server state). *Idempotent* = doing it twice has the same effect as once (so it's safe to **retry** — which matters enormously for network reliability: you can auto-retry a `GET` or `PUT` after a timeout, but retrying a `POST` may double-charge a credit card). This is a real design constraint, not trivia.

**Status codes** (the response's outcome), grouped by first digit:

| Class | Meaning | Examples you must know |
|---|---|---|
| **1xx** | Informational | `101 Switching Protocols` (WebSocket upgrade) |
| **2xx** | Success | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | Redirect | `301` permanent, `302/307` temporary, `304 Not Modified` (cache hit) |
| **4xx** | **Client** error | `400` bad request, `401` unauthenticated, `403` forbidden, `404` not found, `405` method not allowed, `409` conflict, `422` validation, `429` rate-limited |
| **5xx** | **Server** error | `500` internal, `502` bad gateway, `503` unavailable, `504` gateway timeout |

The 4xx-vs-5xx split is a contract: **4xx = "you (the client) did something wrong, don't retry unchanged"; 5xx = "I (the server) failed, retrying might work."** Getting these right is the difference between an API that's pleasant to integrate and one that isn't. `502`/`504` specifically mean *a proxy/load balancer couldn't reach or didn't hear back from the upstream* — you'll see them constantly behind **Nginx** (cross-reference its proxy section).

### 9.3 HTTP/1.1's problem: head-of-line blocking **[I]**

HTTP/1.1 sends one request, waits, gets one response, per connection (you can **pipeline** in theory but it's broken in practice). To load a page with 50 assets, browsers open **~6 parallel TCP connections per host** and juggle requests across them — wasteful (6× handshakes, 6× congestion windows) and still limited.

The core defect is **head-of-line (HOL) blocking at the application layer**: on one connection, response #2 can't start until response #1 finishes, even if #2 is ready. Workarounds people used — **domain sharding** (spread assets over `cdn1`, `cdn2`… to get more connections), **spriting** (combine images), **concatenating** JS/CSS — were all hacks to dodge this limitation. HTTP/2 was built to make them unnecessary.

### 9.4 HTTP/2 and HTTP/3 — multiplexing, then escaping TCP **[A]**

**HTTP/2 (2015, RFC 9113)** keeps the same methods/status/headers but changes the wire format completely:

- **Binary framing** instead of text — efficient and unambiguous to parse.
- **Multiplexing:** many concurrent **streams** over a **single TCP connection**, interleaved as frames. Request #2's response can arrive interleaved with #1's — no application-layer HOL blocking, one connection, one handshake, one congestion window. The sharding/spriting hacks become counterproductive.
- **Header compression (HPACK):** headers repeat enormously (same `Host`, `User-Agent`, cookies every request); HPACK compresses them with a shared dynamic table.
- **Server push** (send resources before asked) — added, then largely deprecated in practice; don't rely on it.

But HTTP/2 has a subtle flaw: it multiplexes streams over **one TCP connection**, and **TCP guarantees order for the whole connection.** So if *one* packet is lost, TCP holds back *all* streams until it's retransmitted — **HOL blocking moved from the application layer down to the transport layer.** On a clean network you don't notice; on a lossy one (mobile, Wi-Fi) it hurts.

**HTTP/3 (2022, RFC 9114)** fixes this by abandoning TCP entirely and running over **QUIC** (RFC 9000), a new transport built on **UDP**:

- QUIC implements reliability, ordering, and congestion control **per-stream in userspace**, so a lost packet only stalls *its own* stream — true independent multiplexing, no transport HOL blocking.
- **TLS 1.3 is baked into the QUIC handshake**, so connection setup is **1 RTT (or 0-RTT on resumption)** instead of TCP-handshake-then-TLS-handshake. Faster first byte.
- **Connection migration:** a QUIC connection is identified by a **connection ID**, not the 4-tuple — so it survives your IP changing (Wi-Fi → cellular) without reconnecting. Huge for mobile.
- Being in userspace over UDP, QUIC can *evolve* without waiting for every OS kernel and middlebox to update — the reason it could ship at all.

**⚡ Version note (2026):** HTTP/3 is broadly deployed; browsers and CDNs negotiate it via the **`Alt-Svc`** header (the server advertises "I'm also available over h3") or the HTTPS DNS record. The negotiation chain in practice: a browser connects over TCP+HTTP/2, learns via `Alt-Svc` that h3 is available, and upgrades to QUIC for subsequent requests. For your code, the framework/CDN handles this — but know *why* a slow mobile experience improves on h3.

A one-glance comparison:

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Year / RFC | 1997 / 9112 | 2015 / 9113 | 2022 / 9114 |
| Transport | TCP | TCP | **QUIC over UDP** |
| Framing | text | binary | binary |
| Multiplexing | no (≈6 conns) | yes (1 conn) | yes (1 conn) |
| HOL blocking | app-layer | **transport-layer** | **none** |
| Header compression | none | HPACK | QPACK |
| TLS | separate, optional | separate, ~always | **built into handshake** |
| Conn. setup | TCP (+TLS) | TCP + TLS | 1-RTT / 0-RTT |
| Conn. migration | no | no | **yes (conn ID)** |

### 9.5 State on a stateless protocol: cookies, sessions, tokens **[I]**

Because HTTP is stateless, "logged-in" state is layered on top:

- **Cookies:** the server sends `Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Lax`; the browser returns `Cookie: session=abc` on every subsequent request to that origin. Flags matter for security: **`HttpOnly`** (JS can't read it — blocks XSS theft), **`Secure`** (HTTPS only), **`SameSite`** (mitigates CSRF). See the **Better Auth** guide.
- **Server-side sessions** (the cookie holds an opaque ID; state lives in a DB/**Redis**) vs **stateless tokens** (**JWT** — the token itself carries signed claims, no server lookup). Trade-offs (revocation, scaling, size) are covered in the **Better Auth** and **Go JWT** guides — this guide just places them: they exist because HTTP itself remembers nothing.

### 9.6 CORS — the cross-origin rule that confuses everyone **[I]**

The browser's **Same-Origin Policy** blocks JavaScript on `https://app.com` from reading responses from `https://api.other.com` (different **origin** = scheme + host + port). **CORS (Cross-Origin Resource Sharing)** is how a server *opts in* to being called cross-origin: it returns `Access-Control-Allow-Origin: https://app.com` (and friends). For "non-simple" requests the browser first sends a **preflight `OPTIONS`** asking "am I allowed to do this?" before the real request.

The crucial mental correction: **CORS is enforced by the browser, on the response, to protect the *user* — it is not server security.** It does not stop `curl`, a mobile app, or an attacker's server from calling your API. So "I'll use CORS to lock down my API" is a category error; CORS only governs what *browser JS from other origins* may read. (You'll configure it constantly — in **Nginx**, **Fastify**, **NestJS**, **Go** — and the #1 confusion is expecting it to do authentication's job.)

---

## 10. TLS / SSL, Certificates & PKI — Encryption on the Wire

### 10.1 What TLS gives you, and the threat it answers **[I]**

Everything so far travels the network **in plaintext** by default — any device on the path (your Wi-Fi AP, the coffee shop, your ISP, every router) can **read** it (eavesdropping) or **alter** it (tampering), and you can't be sure you're even talking to the *real* server (impersonation). **TLS (Transport Layer Security**, the modern name for SSL) wraps a plaintext protocol (HTTP → HTTPS, SMTP, etc.) and provides three guarantees:

1. **Confidentiality** — the data is **encrypted**; on-path observers see only ciphertext.
2. **Integrity** — tampering is **detected** (each record is authenticated); a flipped bit fails verification.
3. **Authentication** — you cryptographically verify the **server's identity** (via its certificate), so you know you're talking to the real `example.com` and not a MITM.

Note what TLS does **not** do: it doesn't hide *that* you connected to a host or *which* host (the destination IP is visible, and historically the **SNI** server name was sent in the clear — **Encrypted Client Hello (ECH)** is the 2026 fix), and it doesn't protect against a compromised endpoint. TLS secures the *channel*, not the parties.

### 10.2 The hybrid crypto idea — why both asymmetric and symmetric **[I]**

TLS combines two kinds of cryptography because each solves a different problem:

- **Asymmetric (public-key) crypto** (RSA, ECDSA, the ECDHE key exchange): a **key pair** — a public key anyone can have, a private key only the server holds. It's great for **establishing trust and agreeing on a secret over an insecure channel without having met before**, but it's computationally *slow*.
- **Symmetric crypto** (AES, ChaCha20): one shared key encrypts/decrypts. It's *fast* — but requires both sides to already share the secret key, which is the chicken-and-egg problem.

TLS uses asymmetric crypto **once, during the handshake, only to safely agree on a fresh random symmetric key**, then switches to fast symmetric crypto for the actual data. Best of both: the trust-establishment power of public-key, the speed of symmetric. The modern key agreement uses **(EC)DHE — Ephemeral Diffie-Hellman** — which gives **forward secrecy**: a fresh key per session means that even if the server's private key is later stolen, *past* recorded sessions stay unreadable.

### 10.3 The TLS 1.3 handshake **[I/A]**

**⚡ Version note:** TLS 1.3 (RFC 8446, the 2026 standard) streamlined the handshake to **one round-trip** (1-RTT), with **0-RTT** resumption for repeat visits. Simplified:

```
   CLIENT                                                          SERVER
     │ ── ClientHello ───────────────────────────────────────▶     │
     │    (TLS version, cipher suites, a DH key share, SNI)         │
     │                                                              │
     │ ◀── ServerHello + key share + {Certificate} + {Finished} ──  │
     │    (agreed cipher, server's DH share, its cert chain,        │
     │     all after the {…} encrypted with the just-derived key)   │
     │                                                              │
     │ ── {Finished} ────────────────────────────────────────▶      │
     │ ════ encrypted application data (HTTP) flows ════════════    │
```

What happens conceptually: both sides exchange Diffie-Hellman key shares and independently derive the **same shared secret** without ever sending it; the server proves its identity by presenting a **certificate** and signing the handshake with its **private key** (only the real holder can). The client validates the cert (10.4) and, satisfied, both switch to symmetric encryption. (TLS 1.2 needed 2 RTTs and had more footguns; **don't accept anything below TLS 1.2**, prefer 1.3.) Over **QUIC/HTTP-3**, this handshake is *merged into* the transport handshake — that's the latency win in Section 9.4.

### 10.4 Certificates, Certificate Authorities & the chain of trust **[I/A]**

A **certificate** (X.509) binds a **public key** to an **identity** (a domain name), vouched for by a **Certificate Authority (CA)** that **digitally signs** it. The trust model — **PKI (Public Key Infrastructure)** — works by a **chain**:

```
   Root CA cert (self-signed)         ← pre-installed in your OS/browser "trust store"
        │ signs
   Intermediate CA cert               ← the CA's working cert (roots stay offline)
        │ signs
   Your server's "leaf" certificate   ← for example.com, presented during the handshake
```

When your browser validates `example.com`'s certificate, it checks that: (1) the cert's name **matches** the domain (the SNI/Host), (2) it's **within its validity dates**, (3) it's **not revoked** (via OCSP/CRL), and crucially (4) it's **signed by a chain that leads to a root the browser already trusts**. Your device ships with ~150 trusted root CAs; everything rests on them. If any link fails — wrong name, expired, self-signed, untrusted issuer — you get the dreaded certificate error.

This is the linchpin: **the CA system is what stops a MITM.** An attacker can intercept your traffic, but they **cannot produce a valid certificate for `example.com`** signed by a trusted CA (they don't control a CA and can't pass domain validation), so the impersonation is detected. (Corporate MITM proxies and tools like mitmproxy work *only* because an admin installs their CA into your trust store — proving the model: trust flows from the root store.)

**⚡ Version note:** **Let's Encrypt** made TLS certs **free and automated** (via the **ACME** protocol) — in 2026 there's no excuse for plaintext HTTP. Certs are short-lived (90 days, trending shorter) and auto-renewed by tools like Certbot, Caddy, or your platform. The **Nginx** and **Docker** guides show terminating TLS in practice. **mTLS (mutual TLS)** — where the *client* also presents a cert — is used for service-to-service auth (see the **gRPC** guide) and zero-trust networks.

### 10.5 Inspecting TLS yourself **[I]**

```bash
# Show the cert chain, validity, and handshake a server presents:
openssl s_client -connect example.com:443 -servername example.com </dev/null

# Just the important fields of a cert:
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# curl shows the negotiated version & cipher with -v:
curl -v https://example.com 2>&1 | grep -iE "SSL connection|TLS|subject|issuer"
```

The **SAN (Subject Alternative Name)** field is what actually matters for name-matching (the old "Common Name" is deprecated). A cert can list many SANs (or a wildcard `*.example.com`). When debugging "cert is valid in browser but fails in my code," it's almost always a **missing intermediate** (server didn't send the full chain) or a **name mismatch** — check the SANs.

---

## 11. WebSockets, SSE & Other Application Protocols

### 11.1 The problem: HTTP is client-initiated **[I]**

Plain HTTP can't let a **server push** data to a client spontaneously — the client must ask. For live data (chat, notifications, dashboards, multiplayer, trading), that's a problem. The historical hacks were **polling** (ask every N seconds — wasteful, laggy) and **long-polling** (hold the request open until there's news — better, still clunky). Two real solutions exist today; pick by directionality.

### 11.2 Server-Sent Events (SSE) — one-way, simple **[I]**

**SSE** is a one-way channel: the client opens a normal HTTP request to an endpoint with `Accept: text/event-stream`, and the server **holds it open and streams events** as text whenever it likes. It's just HTTP (works through proxies/CDNs, gets HTTP/2 multiplexing, auto-reconnects via the browser's `EventSource`), but only **server → client**. Perfect for notifications, live feeds, progress, LLM token streaming. Cheap to implement; if you only need to *push*, prefer SSE over WebSockets.

```js
// Server (Node/Express-style): stream events down one open response
res.writeHead(200, { "Content-Type": "text/event-stream", "Connection": "keep-alive", "Cache-Control": "no-cache" });
setInterval(() => res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`), 1000); // "data: …\n\n" framing

// Browser: the built-in EventSource handles reconnection for you
const es = new EventSource("/events");
es.onmessage = (e) => console.log("push:", JSON.parse(e.data));
```

### 11.3 WebSockets — full-duplex **[I]**

When you need **both directions** over one persistent connection (chat, collaborative editing, games), use **WebSockets** (RFC 6455). A WebSocket starts life as an HTTP request with `Upgrade: websocket`; the server replies **`101 Switching Protocols`**, and from then on the **same TCP connection** becomes a bidirectional, message-framed channel — either side can send anytime, with far less overhead than repeated HTTP requests. It rides on TCP (so ordered/reliable), uses `ws://` (plain) or **`wss://`** (over TLS — always use this in production). 

This guide just *places* WebSockets in the stack (an upgraded Layer-7 protocol over TCP); the **Go Gorilla WebSockets** guide covers them in depth — the one-reader/one-writer rule, ping/pong keepalive, the hub/broadcast pattern, **CSWSH** (cross-site WebSocket hijacking — validate `Origin`!), and scaling with a Redis backplane. Cross-reference it.

### 11.4 A quick map of other protocols you'll meet **[I]**

| Protocol | Layer/transport | What it's for |
|---|---|---|
| **gRPC** | HTTP/2 | High-performance service-to-service RPC w/ protobuf — see **gRPC** guide |
| **MQTT** | TCP | Lightweight pub/sub for IoT / constrained devices |
| **SMTP/IMAP/POP3** | TCP | Sending / retrieving email |
| **SSH** | TCP/22 | Encrypted remote shell & tunneling (and SFTP) |
| **FTP/FTPS/SFTP** | TCP | File transfer — see **FTP** guide (FTPS=FTP+TLS, SFTP=over SSH) |
| **NTP** | UDP/123 | Clock synchronization |
| **WebRTC** | UDP (SRTP/DTLS) | Browser P2P audio/video/data (uses STUN/TURN/ICE) |
| **QUIC** | UDP | The transport under HTTP/3 (Section 9.4) |

---

## 12. Network Security — Firewalls, VPNs, Attacks & Defenses

### 12.1 The mindset: defense in depth & zero trust **[I]**

Network security rests on two modern principles. **Defense in depth** — never rely on a single control; layer them (firewall *and* TLS *and* authentication *and* input validation), so one failure isn't catastrophic. **Zero trust** — the old model of "hard perimeter, soft squishy inside" (trust anything on the corporate LAN) is dead; assume the network is *already* hostile and authenticate/encrypt **every** connection regardless of origin. The practical consequence for you: **encrypt internal traffic too**, authenticate service-to-service calls (mTLS, tokens), and segment aggressively (Section 3.6, VLANs/VPCs).

### 12.2 Firewalls — filtering by layer **[I]**

A **firewall** enforces rules about which traffic is allowed, and *which layer* it inspects determines what it can decide on:

- **Packet-filter / stateless (L3/L4):** allow/deny by source/dest IP, port, protocol. Fast, dumb. ("Allow tcp/443 in, deny everything else.")
- **Stateful (L4):** tracks connection state (the TCP 4-tuple + handshake state), so it can allow **return** traffic for connections *you* initiated while blocking unsolicited inbound. This is what `iptables`/`nftables` (Linux), Windows Firewall, and cloud **security groups** do. The default-good posture: **deny all inbound, allow established/outbound.**
- **Application firewall / WAF (L7):** understands HTTP and filters on URLs, headers, and payloads — blocks SQL injection, XSS, bad bots (e.g. Cloudflare, ModSecurity). It sees *inside* the request because it terminates TLS.

Cloud equivalents you'll actually configure: **security groups** (instance-level, stateful, allow-only) and **network ACLs** (subnet-level, stateless). The cardinal rule: **default-deny inbound, open only the specific ports a service needs**, and never expose databases (5432/3306/6379/27017) to the public internet — a stunning number of breaches are just an open database port.

### 12.3 The attack catalog every engineer should recognize **[I/A]**

You don't need to be a pentester, but you must recognize these by name and know the one-line defense:

| Attack | What it does | Primary defense |
|---|---|---|
| **MITM (man-in-the-middle)** | Intercept/alter traffic between two parties | **TLS** (cert validation), avoid untrusted Wi-Fi for plaintext |
| **ARP / DNS spoofing** | Forge LAN/name mappings to redirect traffic | Dynamic ARP Inspection, DNSSEC, **TLS end-to-end** |
| **DDoS** | Overwhelm a target with traffic from many sources | Upstream scrubbing (Cloudflare/Akamai), rate limiting, anycast |
| **Amplification/reflection** | Spoof victim's IP to a service (DNS/NTP/memcached) that floods them with big replies | Don't run open resolvers; **BCP 38** ingress filtering |
| **SYN flood** | Exhaust half-open connection table | **SYN cookies**, connection limits |
| **Port scanning** | Map open ports/services as recon | Default-deny firewall, minimize exposed surface, fail2ban |
| **Packet sniffing** | Read plaintext on a shared/compromised segment | **Encrypt everything** (TLS, VPN) |
| **Spoofing (IP)** | Forge source IP | Ingress/egress filtering, don't trust source IP for auth |
| **Replay** | Re-send captured valid messages | Nonces, timestamps, TLS, short-lived tokens |
| **SSL stripping** | Downgrade HTTPS→HTTP on first contact | **HSTS** header, HSTS preload, redirect + `Strict-Transport-Security` |
| **DNS cache poisoning** | Inject false DNS answers | DNSSEC, DoH/DoT, randomized source ports |

The thread running through the whole table: **most network attacks are defeated by end-to-end encryption + authentication + least-exposure**, which is why TLS-everywhere and default-deny firewalls are non-negotiable.

### 12.4 DoS / DDoS in a bit more depth **[I/A]**

A **Denial of Service** attack makes a service unavailable by exhausting a resource (bandwidth, CPU, connection slots, a database). **Distributed** DoS uses thousands of compromised machines (a **botnet**) so you can't just block one IP. Categories: **volumetric** (saturate the pipe — Gbps/Tbps of junk), **protocol** (SYN floods, exhausting state), and **application-layer** (expensive requests — a flood of searches or logins that look legitimate but melt your DB). Defenses are layered: **anycast** (the same IP announced from many locations so attack traffic is spread and absorbed near its source), **upstream scrubbing services** (Cloudflare, AWS Shield), **rate limiting** (per-IP/per-token — see the **Nginx** and **Redis** guides), **autoscaling**, and **caching/CDNs** so attack traffic never reaches your origin. You usually can't out-build a large DDoS yourself — you buy a provider whose capacity dwarfs the attack.

### 12.5 VPNs & tunnels — encrypted overlays **[I]**

A **VPN (Virtual Private Network)** creates an **encrypted tunnel** across an untrusted network, making two remote endpoints behave as if on the same private network. Mechanically, your packets are **encrypted and encapsulated** inside other packets (tunneling) to the VPN endpoint, which decrypts and forwards them. Uses: secure remote access to a private network, joining cloud VPCs, privacy on hostile networks. Protocols: **WireGuard** (modern, fast, simple — built on UDP, the current default choice), **IPsec** (the traditional standard, operates at L3), **OpenVPN** (TLS-based). **SSH tunneling/port-forwarding** is the lightweight ad-hoc version (`ssh -L 5432:db.internal:5432 bastion` to reach a private database through a **bastion/jump host** — a pattern you *will* use). The cloud version of "reach private resources safely" is a **bastion host** or a managed service like AWS SSM. Remember a VPN moves trust to the VPN provider/endpoint — it's not magic privacy.

---

## 13. Tools & Troubleshooting — ping, traceroute, dig, ss, curl, tcpdump, Wireshark, nmap

This is the section you'll return to most. The skill of debugging networks is **working up the stack**: is it a *physical/link* problem, a *routing/IP* problem, a *DNS* problem, a *transport* (port/firewall) problem, or an *application* (HTTP/TLS) problem? Each tool answers one of those.

### 13.1 The layered debugging ladder **[B/I]**

```
   "I can't reach the service." Work UP the stack:
   1. Link/IP up?     ip addr / ipconfig         → do I even have an IP & gateway?
   2. Reach gateway?  ping <gateway>              → is the LAN ok?
   3. Reach internet? ping 1.1.1.1               → is routing/NAT ok? (IP works)
   4. DNS working?    ping example.com / dig …    → name resolves? (1.1.1.1 works but name doesn't = DNS)
   5. Path ok?        traceroute example.com      → where does it die?
   6. Port open?      nc -vz host 443 / ss        → is the service listening & firewall open?
   7. App responding? curl -v https://…           → HTTP/TLS-level answer
   8. On the wire?    tcpdump / Wireshark         → see the actual packets
```

**The classic split:** if `ping 1.1.1.1` works but `ping example.com` fails, **IP is fine and it's a DNS problem** — one of the most common and most quickly-diagnosed faults in all of networking.

### 13.2 Connectivity & path: ping, traceroute **[B]**

```bash
ping example.com               # ICMP echo: is it reachable, and what's the RTT? (Ctrl-C to stop)
ping -c 4 1.1.1.1              # 4 packets then stop (Linux/mac); Windows: ping -n 4
# Reads: "time=14.2 ms" = RTT; "Request timed out"/100% loss = unreachable or ICMP blocked

traceroute example.com         # Linux/mac: every router hop + per-hop RTT (uses TTL trick, §5.2)
tracert example.com            # Windows
mtr example.com                # traceroute + ping combined, live — the best path-quality tool
# Reading traceroute: rising latency then timeouts often = the failure is at/after that hop.
# '* * *' can be a hop that just doesn't reply to ICMP (common) — not necessarily broken.
```

⚡ Caveat: many hosts/firewalls **deprioritize or block ICMP**, so `ping` failing doesn't always mean "down," and a high-latency hop *mid*-path may just be a router rate-limiting its own ICMP replies while forwarding fine. Judge by the *destination*, not intermediate hops.

### 13.3 DNS: dig / nslookup (recap with debugging focus) **[B/I]**

```bash
dig example.com +short                 # quick answer
dig +trace example.com                 # watch the whole delegation (root→TLD→authoritative)
dig @8.8.8.8 example.com               # bypass your resolver — is it YOUR DNS or theirs?
dig example.com +noall +answer         # just the answer section, with TTLs
# If `dig @8.8.8.8` works but your default doesn't → your local/ISP resolver is the problem.
```

### 13.4 Sockets & ports: ss, netstat, lsof, nc **[I]**

```bash
ss -tulpn                # Linux: TCP+UDP Listening ports, with the owning Process (needs sudo for names)
#   -t TCP  -u UDP  -l listening  -p process  -n numeric (don't resolve)
netstat -ano             # Windows: all connections + owning PID (then: tasklist /fi "pid eq 1234")
netstat -tulpn           # older Linux equivalent of ss
lsof -i :5432            # mac/Linux: what process holds port 5432? (great for "address already in use")

# Is a remote port open? (TCP connect test — no payload)
nc -vz example.com 443        # "succeeded!" = open; "refused" = closed (RST); hang = filtered (firewall)
# PowerShell native equivalent:
Test-NetConnection example.com -Port 443
```

`ss -tan state established` shows live connections; `ss -s` summarizes. The three outcomes of a port probe map exactly to Section 6: **open** (handshake completes), **refused** (a `RST` — host up, nothing listening), **filtered/timeout** (a firewall silently dropped it — host may be up but blocking). Distinguishing "refused" from "filtered" tells you whether it's an *app* problem or a *firewall* problem.

### 13.5 HTTP & TLS: curl **[B/I]**

`curl` is the most useful single network tool an engineer owns:

```bash
curl -v https://example.com          # -v: show the FULL exchange — DNS, TCP, TLS handshake, headers
curl -I https://example.com          # -I: HEAD — just response headers + status (no body)
curl -sS -o /dev/null -w \
  'dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s\n' \
  https://example.com                 # TIMING BREAKDOWN — pinpoint WHICH phase is slow
curl --resolve example.com:443:93.184.216.34 https://example.com   # force an IP (test before DNS cutover)
curl --http3 https://example.com     # force HTTP/3 (if built with it)
curl -X POST -H 'Content-Type: application/json' -d '{"name":"Ada"}' https://api.example.com/users
```

That `-w` timing breakdown is gold: it tells you whether slowness is **DNS**, **TCP connect**, **TLS**, or **the server's first byte (TTFB)** — turning "the site is slow" into a precise diagnosis that maps to the exact layer.

### 13.6 Packet capture: tcpdump & Wireshark **[A]**

When higher tools aren't enough, look at the actual packets:

```bash
sudo tcpdump -i any -n host example.com           # all traffic to/from a host, don't resolve names
sudo tcpdump -i any -n port 443                    # just port 443
sudo tcpdump -i any -n 'tcp[tcpflags] & tcp-syn != 0'   # only SYN packets (watch handshakes / SYN floods)
sudo tcpdump -i any -n -w capture.pcap host example.com  # write to file → open in Wireshark
```

**Wireshark** is the graphical analyzer: open the `.pcap`, and it decodes every layer (Ethernet → IP → TCP → TLS → HTTP), reassembles streams (right-click → *Follow TCP Stream*), and color-codes problems (retransmissions, resets). Use its **display filters** (`http`, `tcp.port == 443`, `ip.addr == 93.184.216.34`, `tcp.analysis.retransmission`). This is how you *prove* things: see the handshake, see the `RST`, see retransmissions (= packet loss), see the TLS `ClientHello`/`ServerHello`, see exactly where a conversation breaks. Nothing teaches the stack faster than watching real packets — capture your own browser loading a page once and you'll never forget encapsulation.

### 13.7 Scanning & inventory: nmap **[A]**

```bash
nmap -sV example.com            # scan common ports + probe service VERSIONS
nmap -p 1-1000 192.168.1.0/24   # scan ports 1-1000 across a whole subnet
nmap -sn 192.168.1.0/24         # ping-sweep: just discover live hosts (no port scan)
```

`nmap` maps what's listening on a host or network — invaluable for **auditing your own** exposed surface ("wait, why is 6379 open to the world?"). **⚠️ Only scan systems you own or are authorized to test** — unauthorized scanning is hostile and often illegal. Use it defensively: scan your own servers from outside to verify the firewall actually closes what you think it does.

---

## 14. Network Programming — Writing Clients & Servers

You saw raw sockets in Section 7. Here's the level you'll actually work at day-to-day — HTTP clients/servers, UDP, and the production concerns (timeouts, pooling, retries) that separate toy code from robust networked code.

### 14.1 An HTTP server & client in Go **[I]**

```go
// A minimal but production-shaped HTTP server. (See the Go net/http guide for depth.)
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"time"
)

func main() {
	mux := http.NewServeMux()
	// Go 1.22+ routing: method + path pattern.
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})
	mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
		id := r.PathValue("id") // path wildcard
		w.Header().Set("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(map[string]string{"id": id, "name": "Ada"})
	})

	// ALWAYS set timeouts on a public server — the #1 production-hardening miss.
	// Without them, a slow/malicious client can hold connections open forever
	// (a Slowloris DoS) and exhaust your sockets.
	srv := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,  // time to read request headers
		ReadTimeout:       10 * time.Second, // whole request
		WriteTimeout:      15 * time.Second, // whole response
		IdleTimeout:       60 * time.Second, // keep-alive idle
	}
	log.Println("listening on :8080")
	log.Fatal(srv.ListenAndServe())
}
```

```go
// An HTTP client done right: a shared client with a timeout and connection pooling.
package main

import (
	"context"
	"io"
	"net/http"
	"time"
)

// Reuse ONE client across your whole app — it pools TCP/TLS connections
// (keep-alive), so you don't pay handshake cost on every call. Creating a
// new client (or using http.Get) per request defeats pooling and leaks sockets.
var client = &http.Client{
	Timeout: 10 * time.Second, // a total deadline — NEVER leave this unset (default is no timeout = hang forever)
}

func fetch(ctx context.Context, url string) ([]byte, error) {
	// context lets a caller cancel/deadline the request (e.g. when the user navigates away).
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err // network/DNS/TLS error or timeout
	}
	defer resp.Body.Close() // ALWAYS close the body or you leak the connection
	return io.ReadAll(io.LimitReader(resp.Body, 1<<20)) // cap reads (don't trust remote size)
}
```

### 14.2 A UDP example (the connectionless model) **[I]**

```go
// UDP server: no accept(), no per-client socket — just receive datagrams.
package main

import (
	"log"
	"net"
)

func main() {
	addr, _ := net.ResolveUDPAddr("udp", ":9001")
	conn, err := net.ListenUDP("udp", addr) // bind only — no listen()/accept()
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	buf := make([]byte, 1500) // one read = one datagram (message-oriented, unlike TCP)
	for {
		n, client, err := conn.ReadFromUDP(buf) // who sent it comes back with the data
		if err != nil {
			continue
		}
		log.Printf("from %s: %s", client, buf[:n])
		_, _ = conn.WriteToUDP([]byte("ack"), client) // reply to that sender
	}
}
```

The contrast with TCP is the whole lesson: **no connection, no accept, no per-client socket, message-oriented** (one `ReadFromUDP` = exactly one datagram), and *you* must handle loss/ordering/retries if you need them. That's the price and the freedom of UDP (Section 6.2).

### 14.3 The production concerns that actually bite **[I/A]**

Writing the happy path is easy; networks are hostile, so robust networked code always handles these:

- **Timeouts everywhere.** A network call can hang *forever* (the remote is slow, a firewall black-holes the packet, the route is gone). **Every** outbound call needs a timeout/deadline — connect, read, write, and total. The most common production incident is a missing timeout causing cascading hangs as threads/goroutines pile up waiting. Default client timeouts in many libraries are **infinite** — set them explicitly.
- **Retries with backoff + jitter — but only for idempotent ops.** Transient failures (a dropped packet, a momentarily-overloaded server, a `503`) often succeed on retry. Retry with **exponential backoff** (wait 1s, 2s, 4s…) and **jitter** (randomize, so a thousand clients don't retry in lockstep and create a **thundering herd**). **Only auto-retry idempotent requests** (`GET`/`PUT`/`DELETE`, not `POST`) unless you use an **idempotency key** — re-sending a non-idempotent `POST` can double-charge/duplicate (Section 9.2).
- **Circuit breakers.** If a downstream service is failing, stop hammering it — "open the circuit," fail fast, and periodically test if it recovered. Prevents one sick dependency from taking your whole system down.
- **Connection pooling.** Reuse TCP/TLS connections (HTTP keep-alive, DB connection pools) — establishing a connection costs RTTs + TLS handshake. Share one HTTP client / one DB pool; don't open a connection per request.
- **Graceful degradation & deadlines propagation.** Pass a **deadline/context** through the whole call chain so a request that's already too late doesn't keep doing expensive downstream work. Cross-reference the **Go**, **NestJS**, and **Fastify** guides where these patterns are shown in framework context.
- **Validate and bound everything from the network.** Cap request body sizes, read with a `LimitReader`, validate inputs — the network is attacker-controlled input (Section 12).

> The mental model: **on a network, every operation can fail, be slow, be duplicated, or be reordered — write code that survives all four.** This is the practical payoff of understanding IP's "best-effort" nature (Section 4.1): the unreliability you learned about at the IP layer reappears as the bugs you must defend against at the application layer.

---

## 15. Modern & Cloud Networking — Proxies, Load Balancers, CDNs, Containers, Service Mesh

### 15.1 Forward vs reverse proxies **[I]**

A **proxy** is an intermediary that forwards traffic. The direction defines the type:

- A **forward proxy** sits in front of *clients* and forwards their outbound requests (corporate egress filtering, caching, anonymity). The server sees the proxy, not the client.
- A **reverse proxy** sits in front of *servers* and forwards inbound requests to them (the client thinks the proxy *is* the server). This is **Nginx**, **HAProxy**, **Envoy**, Caddy, cloud LBs — and it's where TLS termination, load balancing, caching, rate limiting, and routing live. The entire **Nginx** guide is about being a great reverse proxy; cross-reference it.

### 15.2 Load balancers: L4 vs L7 **[I/A]**

A **load balancer** spreads traffic across many backend instances for scale and availability. The layer it operates at is a real architectural choice:

- **L4 (transport) load balancer** routes by IP/port, forwarding raw TCP/UDP without looking inside. Fast, protocol-agnostic, but can't make decisions based on the request content. (AWS NLB.)
- **L7 (application) load balancer** terminates the connection, reads the **HTTP** request, and routes by **path/host/header/cookie** (`/api` → API pool, `/img` → image pool), can do **TLS termination**, retries, and sticky sessions. Smarter, slightly more overhead. (Nginx, AWS ALB, Envoy.)

Balancing **algorithms:** round-robin, least-connections, IP-hash (sticky), weighted. **Health checks** are the critical companion — the LB probes each backend and stops sending traffic to unhealthy ones, which is what gives you **zero-downtime deploys** and failure tolerance. (All shown concretely in the **Nginx** guide's load-balancing section.)

### 15.3 CDNs & edge **[I]**

A **CDN (Content Delivery Network)** is a globally distributed cache of reverse proxies (Cloudflare, Fastly, CloudFront, Akamai). It serves your content from a **PoP (point of presence)** physically near the user, which slashes **latency** (Section 16 — latency is bounded by distance/light-speed, and a CDN shortens the distance) and offloads your origin. Modern CDNs also provide **DDoS protection, WAF, TLS termination, and edge compute** (run code at the edge — Cloudflare Workers, Lambda@Edge). The mechanism that steers users to the nearest PoP is usually **anycast** (the same IP announced from every location; BGP routes each user to the nearest one — Section 5.4) plus geo-DNS. For static assets and cacheable API responses, a CDN is the single biggest real-world performance win available.

### 15.4 Container & cloud networking — the part that surprises app devs **[I/A]**

When you run containers and cloud services, a whole virtual network layer appears — and it's just the fundamentals (Sections 3–6) implemented in software:

- **Docker networking:** each container gets its own **network namespace** (isolated stack), a **virtual Ethernet** pair connects it to a **bridge** (`docker0`), and Docker does **NAT** so containers reach out and you **publish ports** (`-p 8080:80` is just a port-forward / inbound NAT rule) to reach in. Containers on a **user-defined bridge** resolve each other by name via Docker's built-in **DNS**. This is exactly ARP/bridge/NAT/DNS from earlier, virtualized — the **Docker** guide covers it in depth; everything there will now make sense.
- **Kubernetes networking:** every **Pod** gets its own routable IP (the "IP-per-pod" model — no port juggling between pods); a **Service** gives a stable virtual IP + DNS name in front of a changing set of pods (an internal L4 load balancer, implemented by `kube-proxy`/iptables/eBPF); an **Ingress** (often Nginx or Envoy) is the L7 reverse proxy at the edge; and **NetworkPolicies** are the firewall rules between pods. **CNI** plugins (Calico, Cilium) implement the pod network.
- **VPC (Virtual Private Cloud):** your private, software-defined network in the cloud — **subnets** (public vs private, Section 4.3), **route tables** (Section 5.1), **security groups & NACLs** (firewalls, Section 12.2), **NAT gateways** (so private subnets reach out without being reachable), **internet/VPN gateways**, and **VPC peering**. It is the cloud reification of *every* concept in this guide. When you design a VPC, you are doing subnetting, routing, and firewalling — the fundamentals, with a cloud console.

### 15.5 Service mesh — networking pulled out of your app **[A]**

At microservice scale, concerns like mTLS, retries, timeouts, circuit breaking, load balancing, and observability (Section 14.3) get tedious to implement in every service. A **service mesh** (Istio, Linkerd) moves them into a **sidecar proxy** (an Envoy instance next to each service) that transparently intercepts all traffic. The mesh gives you **mTLS everywhere** (zero-trust by default), uniform retries/timeouts, traffic shaping (canary/blue-green by routing percentages), and golden-signal metrics — *without* changing application code. You won't reach for it early, but know what it is and why it exists: it's the network-reliability patterns from Section 14, lifted out of the app and into the platform.

---

## 16. Performance — Latency, Bandwidth, RTT, Throughput, the BDP

### 16.1 The two numbers that aren't the same: bandwidth vs latency **[I]**

The single most common performance misunderstanding: **bandwidth ≠ latency, and you usually can't fix latency by buying bandwidth.**

- **Bandwidth** is *capacity* — bits per second the link can carry (the pipe's width). Easy to increase (pay for a bigger pipe).
- **Latency** is *delay* — how long one bit takes to arrive (the pipe's length). **Bounded by physics:** signals travel at ~⅔ the speed of light in fiber, so a round trip from New York to London is **~70–80 ms minimum**, no matter how much you pay. You cannot buy your way under the speed of light.

The analogy: a truck full of hard drives has *enormous* bandwidth (petabytes) and *terrible* latency (a day). A text message has tiny bandwidth and great latency. They are independent axes. **For most web/API workloads, latency — especially RTT — dominates the user experience, not bandwidth.**

### 16.2 RTT is the currency — why protocols are measured in round-trips **[I/A]**

Total latency has four components: **propagation** (distance ÷ speed of light — the floor), **transmission** (data size ÷ bandwidth), **queuing** (waiting in router buffers — **bufferbloat** is excessive queuing latency), and **processing** (per-hop handling). For small requests over a long path, **propagation dominates**, and it shows up as **RTT (round-trip time)**.

RTT matters because **protocols cost round-trips**, and they add up *serially* before your first byte:

```
   Loading https://far-away.com over a 80 ms RTT link, cold (HTTP/2 over TCP):
     DNS lookup ............... ~1 RTT   (cached after: 0)
     TCP handshake ............ 1 RTT
     TLS 1.3 handshake ........ 1 RTT
     HTTP request → response .. 1 RTT
     ────────────────────────────────
     ≈ 4 RTTs ≈ 320 ms  before the page even starts — on a "fast" connection.
```

This is *why* the entire industry optimizes round-trips: **connection reuse** (keep-alive, pooling — amortize the handshakes over many requests), **TLS 1.3 & 0-RTT** (fewer setup RTTs), **HTTP/2 multiplexing** (don't serialize requests), **HTTP/3/QUIC** (fold TLS into transport, 0-RTT resumption), and **CDNs** (shorten the distance → shrink every RTT). When you understand that "slow" usually means "too many serial round-trips over a long path," you know exactly which lever to pull.

### 16.3 The Bandwidth-Delay Product — why a fat, long pipe can be slow **[A]**

Here's a result that surprises people: on a high-bandwidth, high-latency link (a "long fat network" — e.g. transcontinental), a single TCP connection can fail to use the available bandwidth. Why? TCP only allows **one window's worth of unacknowledged data in flight** at a time (Section 6.6). The amount you *need* in flight to keep the pipe full is the **Bandwidth-Delay Product (BDP)**:

```
   BDP = bandwidth × RTT
   e.g. 1 Gbps × 80 ms = 1,000,000,000 bits/s × 0.08 s = 80,000,000 bits = 10 MB
```

So to saturate that link, **10 MB must be in flight** at once — meaning the TCP window must be ≥10 MB. The original TCP window maxed at 64 KB, which is why **window scaling** (a TCP option) exists, and why undersized socket buffers throttle throughput on long links. Practical takeaways: tune socket buffers for long-fat paths, use **parallel connections** or a protocol built for it, and remember that **a single TCP stream's throughput ≈ window ÷ RTT** — so higher RTT directly caps single-stream throughput regardless of bandwidth. (This is also why downloading from a distant server is slow even on gigabit fiber, and why CDNs — by cutting RTT — raise throughput, not just reduce latency.)

### 16.4 A performance checklist **[I/A]**

| Lever | Why it helps |
|---|---|
| Reuse connections (keep-alive, pools) | Amortize DNS/TCP/TLS handshake RTTs |
| Use a CDN / edge | Shorten distance → cut RTT on every request, offload origin |
| HTTP/2 or /3 | Multiplexing, fewer connections, no app-layer HOL blocking |
| TLS 1.3 (+0-RTT) | Fewer handshake round-trips |
| Cache aggressively (`Cache-Control`, ETags, `304`) | Avoid the request entirely |
| Compress (gzip/brotli) | Fewer bytes → fewer round-trips for large bodies |
| Reduce request count / payload size | Each request costs ≥1 RTT; each byte costs bandwidth |
| Tune socket buffers on long-fat links | Allow a full BDP in flight (window scaling) |
| Put DBs/services close (same region/AZ) | Internal RTT compounds across many calls per request |

The last row is subtle and important: a single user request often fans out into **dozens of internal service/DB calls**, and if each adds even 1 ms of RTT, that compounds. **Co-locating services** (same availability zone) and **batching** internal calls is a major backend performance discipline — the same RTT math, applied inside your system.

---

## 17. Gotchas & Best Practices

A consolidated list of the traps that bite real engineers — most are "I forgot the network is unreliable and adversarial."

**Protocol & correctness**
- **TCP is a byte stream, not messages.** One `read` ≠ one `write`. Always frame your messages (length-prefix or delimiter). The bug hides until production load splits/merges your writes. (Section 7.3)
- **`localhost`/`127.0.0.1` vs `0.0.0.0`.** Binding to `127.0.0.1` accepts *only* local connections; binding to `0.0.0.0` (or `::`) accepts from *any* interface. "Works on my machine, refuses connections in Docker/prod" is almost always a service bound to localhost when it should bind to all interfaces. (Conversely, never bind a database to `0.0.0.0` on a public host.)
- **Idempotency & retries.** Only auto-retry idempotent requests, or use idempotency keys — retried `POST`s double-charge. (Section 9.2/14.3)
- **IPv6 exists.** Don't assume client IPs are v4; store them in an IPv6-capable type; bind dual-stack. (Section 4.5)

**Reliability**
- **Always set timeouts** on every outbound call — connect, read, write, total. Default is often *infinite*. Missing timeouts cause cascading hangs — the most common outage cause. (Section 14.3)
- **Retry with backoff *and jitter*** to avoid thundering-herd retry storms.
- **Circuit-break failing dependencies** so one sick service doesn't sink the system.
- **`TIME_WAIT` exhaustion / ephemeral port exhaustion** under high connection churn — reuse connections instead of opening one per request. (Section 6.4)

**DNS & TLS**
- **"It's always DNS."** When `ping IP` works but `ping name` fails, it's DNS. Check with `dig @8.8.8.8`. (Section 13.1)
- **DNS propagation = TTL.** Lower TTL *before* a planned cutover; there's no global refresh button. (Section 8.5)
- **Cert errors are usually a missing intermediate or a name (SAN) mismatch**, not a "broken" cert. Inspect with `openssl s_client`. Watch for **expired certs** — automate renewal. (Section 10.5)
- **CORS is browser-enforced user protection, not API security.** It won't stop `curl` or a malicious server. (Section 9.6)

**Security**
- **Default-deny inbound firewall;** open only needed ports. **Never expose databases** (5432/3306/6379/27017) publicly. (Section 12.2)
- **Encrypt everything, even internal traffic** (zero trust). Wi-Fi/VPN encryption only covers one hop — TLS is end-to-end. (Section 12.1)
- **Don't trust source IP / MAC for authentication** — both are spoofable. (Sections 3.2, 12.3)
- **Don't block all ICMP** — you'll break Path MTU Discovery and create "large uploads hang" bugs. (Section 2.4)
- **Validate & bound all network input** (body size limits, `LimitReader`) — it's attacker-controlled. (Section 14.3)

**Performance**
- **Optimize round-trips, not just bandwidth.** Most "slow" is too many serial RTTs over a long path. (Section 16.2)
- **Reuse, cache, co-locate, and use a CDN** before micro-optimizing anything else.

---

## 18. Study Path & Build-to-Learn Projects

### 18.1 A suggested order **[B→A]**

1. **Foundations [B]:** Sections 1–2 (the layered model & encapsulation) until you can draw the stack and the encapsulation dolls from memory. This is the scaffold everything hangs on.
2. **The lower stack [B/I]:** Sections 3–5 (Ethernet/ARP, IP/subnetting, routing/NAT). Do the subnetting by hand until `192.168.1.0/26` → "64 addresses, `.1`–`.62` usable, broadcast `.63`" is instant.
3. **The transport core [I/A]:** Section 6 (TCP vs UDP, the handshake, congestion control) — the most interview-relevant and bug-relevant material. Then Section 7 (sockets) — *run* the echo server/client.
4. **The application layer [B/I]:** Sections 8–11 (DNS, HTTP/1→2→3, TLS, WebSockets/SSE). You use these every day; now you understand them.
5. **Defense & ops [I/A]:** Sections 12–13 (security, tools). Spend real time in Section 13 — **the tools are how knowledge becomes skill.**
6. **Engineering it [I/A]:** Sections 14–16 (writing robust networked code, cloud/containers, performance).

### 18.2 Build-to-learn projects **[I/A]**

Nothing cements networking like building. In rough order of difficulty:

1. **Trace a page load.** Open Wireshark (Section 13.6), capture your browser loading a simple site once, and identify by eye: the DNS query/response, the TCP 3-way handshake, the TLS `ClientHello`/`ServerHello`, and the first HTTP request. *Goal: see every layer of this guide in one real capture.*
2. **Subnet calculator.** Write a CLI (in Go/Python/Node) that takes `CIDR` and prints network address, broadcast, mask, usable range, and host count — *without* a library. *Goal: own the binary math of Section 4.3.*
3. **TCP chat server.** Build the echo server (Section 7.3) into a multi-client chat room with **your own message framing** (length-prefixed or newline-delimited) and a broadcast hub. *Goal: internalize "TCP is a byte stream" and concurrent connection handling.*
4. **Mini DNS resolver.** Send a raw DNS query over UDP to `1.1.1.1` and parse the response packet by hand (header, question, answer). *Goal: demystify DNS and binary protocol parsing.*
5. **HTTP/1.1 server from a raw socket.** Parse the request line + headers off a TCP socket and write a valid response yourself — no web framework. *Goal: see that HTTP is "just text over TCP."*
6. **Reverse proxy + load balancer.** Write a tiny L7 reverse proxy that round-robins requests across two backends with a health check. Then do the same with **Nginx** (cross-reference) and compare. *Goal: connect code to the infrastructure you'll actually run.*
7. **Robust API client.** Build an HTTP client with timeouts, retries-with-jitter, a circuit breaker, and connection pooling (Section 14.3). Point it at a server you can make fail/slow on demand. *Goal: practice surviving an unreliable network.*
8. **Packet-loss lab.** Use `tc netem` (Linux: `tc qdisc add dev … netem loss 5% delay 100ms`) to inject loss/latency, then watch TCP retransmissions in Wireshark and feel how throughput drops. *Goal: experience congestion control and the BDP (Section 16) for real.*

### 18.3 Where to go next in this library **[—]**

- **Run the edge:** [Nginx](NGINX_GUIDE.md) — reverse proxy, load balancer, API gateway, TLS termination (Sections 9, 10, 15 made practical).
- **Virtualized networking:** [Docker](DOCKER_GUIDE.md) — namespaces, bridges, NAT, container DNS (Section 15.4).
- **Write servers:** [Go net/http](GO_NET_HTTP_REST_API_GUIDE.md) and [Go Gin](GO_GIN_REST_API_FILE_UPLOAD_GUIDE.md) (Node: [NestJS](NESTJS_GUIDE.md) / [Fastify](FASTIFY_GUIDE.md)) — HTTP servers with the hardening from Section 14.
- **Real-time:** [Go Gorilla WebSockets](GO_GORILLA_WEBSOCKETS_GUIDE.md) (Section 11) · service-to-service: [gRPC](GO_GRPC_RPC_GUIDE.md) (HTTP/2, mTLS).
- **Auth on the wire:** [Better Auth](BETTERAUTH_GUIDE.md) / [Go JWT + Argon2](GO_JWT_ARGON2_GUIDE.md) (cookies/sessions/tokens, Section 9.5; TLS/PKI, Section 10).
- **File transfer:** [FTP Server (Go & Node)](FTP_SERVER_GO_AND_NODE_GUIDE.md) — FTP vs FTPS vs SFTP (Section 11.4).

---

> **You now have the whole stack.** When you type a URL and a page appears, you can narrate every step: DNS resolves the name (8), TCP + TLS handshake over IP across routers and NAT (4–6, 10), HTTP requests the page (9), and Ethernet/Wi-Fi carries the frames hop by hop (3) — all of it encapsulated, layer by layer (2), and debuggable with the tools in Section 13. That mental model is the difference between a programmer who *uses* the network and an engineer who *understands* it.









