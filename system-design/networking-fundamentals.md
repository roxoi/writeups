---
title: "IP Addresses, OSI Model, TCP and UDP — Networking Fundamentals for System Design"
description: "A deep dive into the networking fundamentals every system designer must know: IP addresses (IPv4/IPv6), the OSI model layers, and the differences between TCP and UDP."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/networking-fundamentals.png"
tags: [networking, ip-address, osi-model, tcp, udp, system-design]
keywords: ["IP address explained", "OSI model layers", "TCP vs UDP", "Networking fundamentals system design", "IPv4 vs IPv6"]
---

# IP Addresses, OSI Model, TCP and UDP — Networking Fundamentals for System Design

Before designing any distributed system, you need a solid mental model of how data physically travels across a network. This post covers the three core networking concepts every system designer must know: IP addressing, the OSI model, and the TCP/UDP protocol distinction.

This post is part of the [System Design series](system-design). If you want a slower, module-by-module treatment of networking, start with the [Computer Networks series](computer-networks).

![Networking Fundamentals](/images/networking-fundamentals.png)

## IP Addresses

An IP (Internet Protocol) address is a **unique identifier assigned to every device on a network**. It enables machines to locate and communicate with each other across the internet or within a local network.

### IPv4

The original Internet Protocol uses a **32-bit numeric dot-decimal notation**, supporting approximately 4.3 billion (2³²) unique addresses.

```
Example: 102.22.192.181
```

With the explosive growth of internet-connected devices, the IPv4 address pool was exhausted (IANA handed out its last free blocks in 2011). NAT, CIDR, and eventually IPv6 are the answers to that shortage.

### IPv6

Standardized in 1998 and still being rolled out today, IPv6 uses a **128-bit hexadecimal notation**, providing approximately 3.4 × 10³⁸ unique addresses — effectively unlimited for foreseeable future demand.

```
Example: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
Short form: 2001:db8:85a3::8a2e:370:7334
```

Leading zeros in each group can be dropped, and one run of all-zero groups can be replaced with `::`.

### Types of IP Addresses

| Type        | Description                                                           | Example                                                |
| ----------- | --------------------------------------------------------------------- | ------------------------------------------------------ |
| **Public**  | Globally routable address that is reachable over the internet         | The address your ISP assigns to your router            |
| **Private** | Reserved for use inside a local network; not routable on the internet | 192.168.x.x, 10.x.x.x, 172.16.x.x – 172.31.x.x         |
| **Static**  | Manually assigned, does not change                                    | Used for servers, VPNs, geo-location services          |
| **Dynamic** | Automatically assigned by DHCP, changes over time                     | Common for consumer devices                            |

Public/private describes **reachability**, while static/dynamic describes **how the address is assigned**. They are independent: a production server typically has a static _public_ IP, while a laptop on your home Wi-Fi has a dynamic _private_ IP.

**System Design Implication:** When designing services that require stable endpoints (databases, APIs, load balancers), static IPs or DNS-based service discovery (see [DNS](system-design/dns)) is essential. Dynamic IPs require additional abstraction layers.

Subnetting, CIDR notation, ARP, and routing are covered in [Module 2: Addressing, Subnetting and Routing](computer-networks/addressing-subnetting-and-routing).

## The OSI Model

The Open Systems Interconnection (OSI) model is a **conceptual framework that standardizes how network communication is structured** across seven abstraction layers. While real-world networks use TCP/IP (which collapses some layers), the OSI model remains the universal reference for understanding network behavior.

```
7  Application   ←  HTTP, DNS, SMTP, SSH         (data)
6  Presentation  ←  TLS, JSON, Protobuf          (data)
5  Session       ←  sessions, RPC                (data)
4  Transport     ←  TCP, UDP                     (segments / datagrams)
3  Network       ←  IP, ICMP, BGP, OSPF          (packets)
2  Data Link     ←  Ethernet, Wi-Fi, ARP         (frames)
1  Physical      ←  cables, fiber, radio         (bits)
```

For a deeper walkthrough with encapsulation diagrams, see [Module 1: The OSI and TCP/IP Layered Models](computer-networks/basics-and-the-layered-models#13-the-osi-and-tcpip-layered-models).

### Layer 7 — Application

The only layer that **directly interacts with user data**. It handles protocols that software applications rely on, not the applications themselves.

- Protocols: HTTP, HTTPS, SMTP, FTP, DNS, WebSocket
- System design relevance: API gateway, [load balancers](system-design/load-balancing) operating at L7, content-based routing

### Layer 6 — Presentation

Handles **data translation, encryption/decryption, and compression**. Ensures that data from the application layer is in a format that the receiving system can understand.

- Responsible for: TLS/SSL encryption, data serialization (JSON, XML, Protobuf)
- Note: in practice, TLS runs on top of TCP and is part of the application layer in the TCP/IP model. OSI conventionally places it here.

### Layer 5 — Session

Manages **opening, maintaining, and closing communication sessions** between devices. Synchronizes data transfer using checkpoints.

- Relevant to: WebSocket sessions, RPC sessions

### Layer 4 — Transport

Provides **end-to-end communication** between processes, including segmentation, flow control, and error correction.

- Protocols: TCP (reliable), UDP (unreliable but fast)
- System design relevance: Load balancers operating at L4 use IP/port for routing without inspecting packet content

### Layer 3 — Network

Responsible for **routing packets** between different networks using logical addressing (IP).

- Protocols: IP, ICMP, routing protocols (BGP, OSPF)
- Devices: Routers

### Layer 2 — Data Link

Handles **data transfer between devices on the same network** using physical addressing (MAC addresses).

- Protocols: Ethernet, Wi-Fi (802.11), ARP (maps an IP address to a MAC address on the local link)
- Devices: Switches, network bridges

### Layer 1 — Physical

The actual **physical transmission medium** — cables, fiber optics, radio waves. Converts digital data to physical signals (bits to electrical/optical/radio signals).

### OSI Model Summary Table

| OSI Layer                                                | TCP/IP Layer   | Data Unit (PDU)                  | Primary Responsibility                                                             | Common Protocols                 | Key Hardware                  |
| -------------------------------------------------------- | -------------- | -------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------- | ----------------------------- |
| 7. Application<br>6. Presentation<br>5. Session          | Application    | Data / Message                   | User interaction, data formatting (encryption, compression), and session tracking. | HTTP, HTTPS, FTP, DNS, SMTP, SSH | Gateway, L7 load balancer     |
| 4. Transport                                             | Transport      | Segments (TCP) / Datagrams (UDP) | End-to-end connection, flow control, error recovery, and port addressing.          | TCP, UDP                         | Firewall, L4 load balancer    |
| 3. Network                                               | Internet       | Packets                          | Logical addressing (IP routing) and finding the best path across routers.          | IPv4, IPv6, ICMP, OSPF           | Router, Layer 3 switch        |
| 2. Data Link                                             | Network Access | Frames                           | Physical addressing (MAC address), hop-to-hop delivery, and error detection.       | Ethernet, Wi-Fi (802.11), PPP, ARP | Switch, Bridge              |
| 1. Physical                                              | Network Access | Bits                             | Cable types, voltage, electrical signaling, and raw bit transmission.              | Cables, Fiber Optics             | Hub, Repeater, NIC            |

> ARP is usually shown at Layer 2 (sometimes called "Layer 2.5") because it resolves IP addresses to MAC addresses on a single link.

## TCP vs UDP

At Layer 4, you must choose between two fundamentally different transport protocols: **TCP** (reliability-first) and **UDP** (speed-first). A full walkthrough of windows, congestion control, and the handshake lives in [Module 3: TCP and UDP](computer-networks/tcp-and-udp-the-transport-layer).

### TCP — Transmission Control Protocol

TCP is a **connection-oriented protocol** that establishes a connection before any data is transmitted (via a three-way handshake: SYN → SYN-ACK → ACK). It guarantees:

- **Ordered delivery** — packets arrive in the sequence they were sent
- **Error checking** — corrupted packets are detected and retransmitted
- **Flow control** — prevents overwhelming the receiver
- **Congestion control** — adapts to network conditions

**Cost:** Higher overhead due to handshaking, acknowledgments, and retransmission mechanisms.

**Use cases:** HTTP/HTTPS, email (SMTP), file transfer (FTP), database connections — anywhere data integrity is critical.

### UDP — User Datagram Protocol

UDP is a **connectionless protocol** that sends packets (datagrams) without establishing a connection first. It provides:

- **No guaranteed delivery** — packets may be lost
- **No ordering** — packets may arrive out of sequence
- **No built-in congestion control** — the application decides how fast to send
- **Minimal overhead** — no handshaking or acknowledgment (the UDP header is only 8 bytes)

**Benefit:** Lower latency and far less overhead, since there is no handshake and no waiting for retransmissions.

**Use cases:** Live video/audio streaming and video calls, online gaming, DNS queries, VoIP, IoT telemetry — anywhere low latency matters more than perfect delivery.

> On-demand streaming (Netflix, YouTube) mostly runs over HTTP on TCP, or over HTTP/3 (QUIC) which itself runs on top of UDP.

### TCP vs UDP Comparison

| Feature            | TCP                        | UDP                                |
| ------------------ | -------------------------- | ---------------------------------- |
| Connection type    | Connection-oriented        | Connectionless                     |
| Delivery guarantee | Guaranteed                 | Not guaranteed                     |
| Ordering           | In-order delivery          | No ordering                        |
| Retransmission     | Yes, on packet loss        | No                                 |
| Speed              | Slower                     | Faster                             |
| Overhead           | High                       | Low                                |
| Broadcasting       | Not supported              | Supported                          |
| Use cases          | HTTPS, SSH, FTP, databases | Live streaming, DNS, VoIP, gaming  |

### System Design Decision: TCP vs UDP

When designing a system, choose based on these principles:

- **Choose TCP** when data correctness is non-negotiable: financial transactions, user authentication, file transfers, API calls.
- **Choose UDP** when latency matters more than completeness: real-time video/audio, live scoreboards, location tracking with frequent updates (old data is worthless anyway).
- **Hybrid approach:** Some protocols like QUIC (used by HTTP/3) build reliability features on top of UDP to get the best of both worlds.

## Putting It All Together

In a typical web request:

1. **DNS** (Layer 7, usually UDP) resolves the domain to an IP address — see [DNS](system-design/dns)
2. **TCP handshake** (Layer 4) establishes a connection to the server's IP
3. **TLS negotiation** (Layer 6) encrypts the session
4. **HTTP request** (Layer 7) is sent over the encrypted TCP connection

Underneath every one of those steps, **IP routing** (Layer 3) moves packets across networks, and **Ethernet/Wi-Fi frames** (Layer 2) carry them across each individual hop.

Understanding which layer a component operates at directly informs how you design load balancers, proxies, firewalls, and monitoring systems.

## Computer Networks Interview Handbook

Neeche interview-focused quick notes hain (Hinglish mein). Detail mein padhna ho toh [Networking for System Design Interviews](computer-networks/networking-for-system-design-interviews) dekho.

### 1. The Core Backbone: OSI Model vs. TCP/IP Model

Interviewers ka sabse basic aur favorite topic. Aapko saari layers, unke exact functions aur protocols yaad hone chahiye. Poori table upar [OSI Model Summary Table](#osi-model-summary-table) mein hai.

TCP/IP model mein 4 layers hoti hain: **Application** (OSI 5–7), **Transport** (OSI 4), **Internet** (OSI 3) aur **Network Access** (OSI 1–2).

### 2. The Protocols SDEs Use Every Day

#### 2.1 HTTP vs. HTTPS (Web Core)

- **HTTP (Port 80):** Data plain text mein bhejta hai — koi encryption nahi hota, isliye beech mein koi bhi (man-in-the-middle) data padh ya badal sakta hai.
- **HTTPS (Port 443):** HTTP + TLS (SSL ka modern successor). Yeh data ko secure (encrypt) karta hai.

**The HTTPS Handshake Mechanism (Interview Favourite):**

1. **Client Hello:** Client plaintext mein supported TLS versions aur cryptographic algorithms (Cipher Suites) ki list bhejta hai.
2. **Server Hello + Certificate:** Server TLS version aur cipher select karke apna Public Key Certificate (CA signed) bhejta hai.
3. **Verification:** Client Certificate Authority (CA) chain se certificate check karta hai (domain match, expiry, signature) ki website asli hai ya nahi.
4. **Key Exchange:** Client aur server ek shared secret banate hain. Classic TLS 1.2 (RSA) flow mein client ek random key generate karke server ki public key se encrypt karke bhejta hai (Asymmetric Encryption). Modern TLS 1.3 mein ECDHE (Diffie-Hellman) use hota hai, jisme secret kabhi network par bheja hi nahi jaata aur forward secrecy milti hai.
5. **Session Key:** Dono us shared secret se ek symmetric session key banate hain, aur aage ka pura data Symmetric Encryption se chalta hai (kyunki symmetric processing fast hoti hai).

Zyada detail ke liye [Module 4: DNS, HTTP and TLS](computer-networks/dns-http-and-tls-the-application-layer) dekho.

#### 2.2 TCP vs. UDP (The Transport Layer Choice)

System Design aur coding problems mein yeh direct use hota hai.

- **TCP (Transmission Control Protocol):**
  - **Connection-Oriented:** Data bhejne se pehle connection set karta hai (3-Way Handshake: SYN → SYN-ACK → ACK).
  - **Reliable:** Har packet ka acknowledgement (ACK) leta hai. Agar packet kho jaye toh use dobara bhejta hai (Retransmission).
  - **Flow Control:** Sliding Window se receiver ko overwhelm hone se bachata hai.
  - **Congestion Control:** Slow start aur congestion avoidance se sending speed adjust karta hai taaki network overload na ho.
  - **Use Case:** Web Browsing (HTTP), Database connections, File Transfers (FTP).
- **UDP (User Datagram Protocol):**
  - **Connectionless:** "Fire and forget". Data bas bhej deta hai bina check kiye ki samne wala ready hai ya nahi.
  - **Unreliable / Fast:** Koi ACK nahi, koi recovery nahi, isliye overhead aur latency bahut kam hoti hai.
  - **Use Case:** Live Video Streaming / Video Calls, Online Gaming, VoIP (Discord voice calls), DNS queries.

### 3. High-Yield Networking Concepts (Online Assessment Specials)

#### 3.1 DNS (Domain Name System) — Port 53

- **Concept:** Yeh internet ka phonebook hai, jo human-readable domains (`google.com`) ko IP addresses (`142.250.190.46`) mein badalta hai. Poori detail [DNS](system-design/dns) post mein hai.
- **DNS Resolution Flow:** Aapne browser mein `apple.com` dala:

1. **Browser/OS Cache:** Pehle aapka device locally check karta hai.
2. **Recursive Resolver (ISP / 8.8.8.8):** Cache mein nahi hai toh query resolver ko jaati hai. Aage ke steps resolver aapki taraf se khud karta hai.
3. **Root Name Server (.):** Yeh batata hai ki `.com` ke TLD servers kahan hain.
4. **TLD Name Server (.com):** Yeh batata hai ki `apple.com` ka authoritative name server kaun sa hai.
5. **Authoritative Name Server:** Yeh `apple.com` ka final IP address return karta hai. Resolver use TTL tak cache karke browser ko de deta hai.

#### 3.2 IP Addressing: IPv4, IPv6 & Subnetting (CIDR)

Assessments mein network math par sawal aate hain.

- **IPv4:** 32-bit address (`192.168.1.1`). Total 2³² (~4.3 billion) addresses possible hain, jo ab practically exhaust ho chuke hain (isi wajah se NAT aur IPv6 aaye).
- **IPv6:** 128-bit hexadecimal address, total 2¹²⁸ combinations.
- **CIDR Notation (Math Trick):** Agar likha hai `10.0.0.0/24`, toh iska matlab hai pehle 24 bits Network ke liye lock hain, aur bache hue 8 bits Host (devices) ke liye free hain.
- **Formula for usable hosts = 2^(free bits) − 2** (2 minus hote hain kyunki 1st address Network ID hota hai aur last address Broadcast ID hota hai).
  - `/24` → 2⁸ − 2 = **254** devices
  - `/26` → 2⁶ − 2 = **62** devices
  - `/30` → 2² − 2 = **2** devices
  - (`/31` aur `/32` special cases hain: point-to-point link aur single host.)

Zyada practice ke liye [Module 2: Addressing, Subnetting and Routing](computer-networks/addressing-subnetting-and-routing) dekho.

#### 3.3 Dynamic Host Configuration Protocol (DHCP)

- **Concept:** Jab bhi aap kisi naye Wi-Fi se connect hote hain, aapko IP manually nahi dalna padta — router/DHCP server aapko automatically ek unique IP assign kar deta hai. DHCP UDP port 67 (server) aur 68 (client) use karta hai.
- **DORA Process (Repeated MCQ):**

1. **Discover:** Client broadcast karta hai, "Kya koi DHCP server hai?"
2. **Offer:** Server response bhejta hai, "Haan, mere paas yeh IP free hai."
3. **Request:** Client kehta hai, "Theek hai, mujhe yeh IP lock kar do."
4. **Acknowledge:** Server confirmation bhejta hai, "Done, ab se yeh IP tumhari lease par hai."

### 4. Top 10 High-Yield Networking Interview Questions (Quick Prep)

#### Q1: Browser mein URL enter karte hi peeche kya hota hai?

**A:** Browser cache check hota hai → DNS lookup se IP pata chalta hai → TCP 3-Way Handshake hota hai → TLS handshake hota hai (agar HTTPS hai) → HTTP GET request jaati hai → Web Server response (HTML/JSON) bhejta hai → Browser page render karta hai.

#### Q2: TCP 3-Way Handshake kya hai?

**A:** Connection setup mechanism. Client SYN flag wala packet bhejta hai → Server SYN-ACK se acknowledge karta hai → Client ACK bhejkar connection establish kar deta hai.

#### Q3: Ping command kis protocol par chalti hai?

**A:** ICMP (Internet Control Message Protocol) par. Yeh TCP/UDP (transport layer) use nahi karti — ICMP Echo Request aur Echo Reply messages seedha IP packets ke andar jaate hain.

#### Q4: Symmetric aur Asymmetric Encryption mein kya farq hai?

**A:** Symmetric mein lock aur unlock karne ki key same hoti hai (fast processing). Asymmetric mein do keys hoti hain: Public Key (sabke saath share hoti hai, encrypt/verify ke liye) aur Private Key (secret rehti hai, decrypt/sign ke liye).

#### Q5: Switch aur Router mein kya core farq hai?

**A:** Switch Layer 2 par chalta hai aur MAC addresses ke base par local network ke andar frames bhejta hai. Router Layer 3 par chalta hai aur IP addresses ke base par do alag-alag networks (LAN to WAN) ko connect karta hai.

#### Q6: ARP (Address Resolution Protocol) kya hai?

**A:** Yeh IP Address ko MAC Address se map (resolve) karta hai. Jab kisi host ko same LAN par destination ka IP pata ho par MAC address na pata ho, toh woh poore LAN par ARP Request broadcast karta hai. Jis device ka wo IP hai, wahi unicast reply mein apna MAC address bhejta hai.

#### Q7: MAC address aur IP address mein kya difference hai?

**A:** MAC address physical (link-layer) address hai jo network card (NIC) mein factory se burn hota hai — par software se badla (spoof) bhi ja sakta hai, aur phones privacy ke liye random MAC use karte hain. IP address logical address hai jo network (DHCP) dynamically assign karta hai (changeable).

#### Q8: Cookies aur Sessions mein kya farq hai backend engineer ke liye?

**A:** HTTP stateless hai (purani request yaad nahi rakhta). State manage karne ke liye Cookies ka data client-side browser mein save hota hai, aur Session ka sensitive data server ki memory/Redis database mein save hota hai (client ke paas sirf session ID cookie hoti hai).

#### Q9: Gateway kya hota hai?

**A:** Kisi bhi local private network ka exit point. Jab aapki machine ko aisa IP call karna ho jo uske local subnet mein nahi hai, toh packet automatically default gateway (Router) ko de diya jaata hai taaki woh use bahar internet par route kare.

#### Q10: Proxy Server aur Reverse Proxy mein kya farq hai?

**A:** Forward Proxy clients ke aage lagta hai aur unki identity web servers se chupata hai (jaise corporate ya anonymizing proxy). Reverse Proxy web servers ke aage lagta hai aur incoming traffic distribute karne ([Load Balancing](system-design/load-balancing)), SSL termination karne, aur caching ke kaam aata hai.

## Continue Learning

**Previous:** [What is System Design?](system-design/what-is-system-design) · **Next:** [DNS](system-design/dns)

### Computer Networks series

- [Module 1: Basics and the Layered Models](computer-networks/basics-and-the-layered-models)
- [Module 2: Addressing, Subnetting and Routing](computer-networks/addressing-subnetting-and-routing)
- [Module 3: TCP and UDP — The Transport Layer](computer-networks/tcp-and-udp-the-transport-layer)
- [Module 4: DNS, HTTP and TLS — The Application Layer](computer-networks/dns-http-and-tls-the-application-layer)
- [Module 5: Firewalls, NAT, VPN and Load Balancing](computer-networks/firewalls-nat-vpn-and-load-balancing)
- [Networking for System Design Interviews](computer-networks/networking-for-system-design-interviews)

### More from System Design

- [DNS](system-design/dns)
- [Load Balancing](system-design/load-balancing)
- [Caching and CDN](system-design/caching-and-cdn)
- [System Design Interview Guide](system-design/system-design-interview-guide)