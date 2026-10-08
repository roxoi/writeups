---
title: "Module 1: Basics & the Layered Models"
description: "Learn what a network is, how packets travel, how bandwidth and latency differ, what the OSI and TCP/IP layers do, and how switches and routers forward data - from first idea to interview-level detail."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/networking-module-1.png"
tags: [Networking, OSI-Model, TCP-IP, Ethernet, Interview-Prep]
keywords: ["OSI model explained", "TCP/IP model vs OSI model", "Bandwidth vs latency", "Switch vs router vs hub"]
---

# Module 1: Basics & the Layered Models

![Networking Module 1](/images/networking-module-1.png)

## Words you will meet in this module

| Word                 | Simple meaning                                                                             |
| -------------------- | ------------------------------------------------------------------------------------------ |
| **Node / host**      | Any device on a network.                                                                   |
| **Link**             | The connection between two nodes (cable, fiber, Wi-Fi).                                    |
| **Protocol**         | An agreed set of rules for communication.                                                  |
| **Header / payload** | The control information at the front of a packet, and the actual data it carries.          |
| **Bandwidth**        | The maximum amount of data a link can carry per second.                                    |
| **Latency**          | The time for data to get from one point to another.                                        |
| **Throughput**       | The data rate you actually achieve.                                                        |
| **Jitter**           | How much latency varies from packet to packet.                                             |
| **MTU**              | Maximum Transmission Unit: the largest packet a link can carry (1500 bytes on Ethernet).   |
| **Encapsulation**    | Wrapping data in a new header as it passes down a layer.                                   |
| **PDU**              | Protocol Data Unit: the name of a chunk of data at a given layer (segment, packet, frame). |

## 1.1: What Is a Network?

A **network** is two or more devices connected so that they can send data to each other. That is all. Three things make it work:

- **Nodes:** the devices (laptops, phones, servers, printers, routers).
- **Links:** the connections between them (Ethernet cable, fiber, Wi-Fi radio).
- **Protocols:** the rules everyone follows so that the data can be understood.

Think of the postal system. A **letter** is a packet. The **address on the envelope** says where it should go. **Sorting centers** are routers that decide the next stop. The **postal rules** (envelope size, address format) are the protocols. You never carry the letter yourself. You trust the system to pass it along, step by step.

### How networks are sized

| Name                                | Covers                   | Example                       |
| ----------------------------------- | ------------------------ | ----------------------------- |
| **PAN** (personal area network)     | A few meters             | Phone to Bluetooth earbuds    |
| **LAN** (local area network)        | A home, office or campus | Your home Wi-Fi               |
| **MAN** (metropolitan area network) | A city                   | A city-wide fiber ring        |
| **WAN** (wide area network)         | Countries and continents | A company linking its offices |
| **The Internet**                    | The whole world          | A **network of networks**     |

The Internet is not one big network. It is thousands of separate networks, run by **ISPs** (Internet Service Providers), companies, universities and clouds, joined together at meeting points. Your packet crosses several of them on its way to a server.

### How devices are arranged (topologies)

| Topology          | Shape                                     | Today                                      |
| ----------------- | ----------------------------------------- | ------------------------------------------ |
| **Bus**           | All devices on one shared cable           | Old, almost gone                           |
| **Ring**          | Devices in a circle                       | Rare                                       |
| **Star**          | Every device connects to a central switch | **The normal LAN design**                  |
| **Mesh**          | Many direct links between devices         | Used in the Internet core and data centers |
| **Tree / hybrid** | Stars joined together                     | Large buildings and campuses               |

### Who talks to whom (communication models)

- **Client-server:** a _client_ asks and a _server_ answers. Web browsing, email and most apps work this way. The server is always on and has a known address.
- **Peer-to-peer (P2P):** every device can act as both client and server. File sharing and some video calls work this way.

### What your home network looks like

```
 [Phone]  [Laptop]  [TV]
     \       |       /
      ~~~ Wi-Fi ~~~
           |
     [Home router / access point]
           |
        [Modem]
           |
     [ISP network]  ──►  [Internet]  ──►  [Server]
```

Most home boxes combine a **modem**, a **router**, a **switch** and a **Wi-Fi access point** in one device. They are four different jobs. Topic 1.4 separates them.

#### Interview traps

- "Is the Internet a network?" Better answer: it is a **network of networks**, joined by routers using the common IP protocol.
- "Internet vs. World Wide Web?" The Internet is the infrastructure. The Web (HTTP and websites) is just one service running on it, like email is another.
- "LAN vs. WAN?" Do not answer only by distance. A WAN usually uses links leased from a provider, and a LAN is owned by one organization.

#### Tricky question and answer

###### Q1 [SDE-1]: Is your home Wi-Fi "the Internet"?

**Answer:** No. Home Wi-Fi is a **LAN**. It becomes a path to the Internet only because your router connects it to your ISP's network. If the router loses its link to the ISP, your devices can still talk to each other (printing and local file sharing still work), but they cannot reach websites.

## 1.2: Packets, Switching & Network Performance

### The idea in plain words

When you send a 5 MB photo, the network does not push it down the wire as one giant block. It is cut into small pieces called **packets**. Each packet carries:

- a **header**: addresses and control information (like the envelope),
- a **payload**: a slice of your data (like the letter).

Packets from many people share the same links, and each packet is forwarded separately. This is called **packet switching**, and it is how the Internet works.

### Circuit switching vs. packet switching

|          | Circuit switching                           | Packet switching                                  |
| -------- | ------------------------------------------- | ------------------------------------------------- |
| Idea     | Reserve a dedicated path for the whole call | Cut data into packets that share links            |
| Example  | Old telephone network                       | The Internet                                      |
| Link use | Wasted when nobody is talking               | Shared efficiently (**statistical multiplexing**) |
| Setup    | Needs a setup phase                         | None needed                                       |
| Weakness | Blocks if all circuits are busy             | Packets can queue, be delayed or be dropped       |

**Statistical multiplexing** is the key idea: not everyone sends at the same moment, so many users can safely share one link. The price is that, when too many send at once, packets must wait in a queue, and some may be dropped.

### The four words people mix up

| Term           | Meaning                                    | Analogy (a road)                      |
| -------------- | ------------------------------------------ | ------------------------------------- |
| **Bandwidth**  | Maximum data per second the link can carry | How many lanes the road has           |
| **Latency**    | Time for one bit or packet to travel       | How long one car takes to arrive      |
| **Throughput** | Data rate you really achieve               | Cars that actually arrive per minute  |
| **Jitter**     | Variation in latency                       | Some cars fast, some stuck in traffic |

Plus **packet loss**: the percentage of packets that never arrive.

**A key fact:** more bandwidth does not reduce latency (much). A wider road lets more cars through, but it does not make any single car arrive sooner.

### Where the time goes: the four delays

When a packet crosses one router, the time it spends there is the **nodal delay**:

```
nodal delay = processing + queuing + transmission + propagation
```

| Delay            | What it is                                               | Formula                             |
| ---------------- | -------------------------------------------------------- | ----------------------------------- |
| **Processing**   | The router reads the header and decides where to send it | Microseconds                        |
| **Queuing**      | Waiting in line behind other packets                     | Depends on traffic. Can be large    |
| **Transmission** | Pushing all the packet's bits onto the link              | `L / R` (packet length ÷ link rate) |
| **Propagation**  | The signal travelling along the wire                     | `d / s` (distance ÷ signal speed)   |

Signal speed in fiber and copper is about **2 × 10⁸ m/s** (roughly two-thirds of the speed of light).

###### Worked example

Send a 1 MB file over a 100 Mbps link that is 3,000 km long.

- File size: 1 MB = 8,000,000 bits.
- **Transmission delay** = 8,000,000 ÷ 100,000,000 = **0.08 s = 80 ms**.
- **Propagation delay** = 3,000,000 m ÷ 200,000,000 m/s = **0.015 s = 15 ms**.

Transmission is bigger here, so a faster link would help. For a _tiny_ request over the same distance, transmission is almost zero and propagation (15 ms) dominates, so a faster link would help _nothing_. This is why "buy more bandwidth" does not fix a slow, chatty protocol over a long distance.

###### Bandwidth-delay product (BDP)

```
BDP = bandwidth × round-trip time
```

It is the amount of data "in flight" on a link at one moment, like the volume of a pipe. Example: 100 Mbps × 50 ms = 5,000,000 bits ≈ **625 KB**. To keep this link full, a sender must be allowed to have about 625 KB sent but not yet acknowledged. If it is allowed less, it sits idle waiting. Module 3 uses this idea to explain TCP windows.

#### See it yourself

```
$ ping -c 3 example.com
PING example.com (93.184.216.34): 56 data bytes
64 bytes from 93.184.216.34: icmp_seq=0 ttl=56 time=24.1 ms
64 bytes from 93.184.216.34: icmp_seq=1 ttl=56 time=23.8 ms
64 bytes from 93.184.216.34: icmp_seq=2 ttl=56 time=31.5 ms
--- example.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
round-trip min/avg/max = 23.8/26.5/31.5 ms
```

How to read it: `time` is the **round-trip time (RTT)**, the there-and-back latency. The spread between 23.8 and 31.5 ms is **jitter**. `0% packet loss` means all three packets came back. `ttl` is a counter that each router decreases by one (explained in Module 2).

(The numbers above are a typical example. Your values will differ.)

Rough latency numbers worth remembering: inside one data center under 1 ms, within a city a few ms, across a continent tens of ms, across an ocean 70–150 ms, and through a geostationary satellite around 600 ms round trip.

#### Interview traps

- "A 1 Gbps link is always faster than a 10 Mbps link." Not for small requests over long distances, where latency dominates.
- Confusing **bandwidth** (capacity) with **throughput** (achieved speed). Throughput is limited by the slowest link on the path, and by protocol behavior.
- Forgetting that **Mbps** (megabits per second) is not **MBps** (megabytes per second). Divide by 8.
- Ignoring **queuing delay**. It is the only one of the four that varies wildly, and it is the main cause of jitter.

#### Tricky questions and answers

###### Q1 [SDE-1/2]: How long does it take to send a 10 MB file over a 1 Gbps link that is 4,000 km long? (Ignore processing and queuing.)

**Answer:**

- File = 10 MB = 80,000,000 bits.
- Transmission = 80,000,000 ÷ 1,000,000,000 = **80 ms**.
- Propagation = 4,000,000 m ÷ 200,000,000 m/s = **20 ms**.
- The first bit arrives after 20 ms. The last bit arrives after about **100 ms** (80 ms to push everything out, plus 20 ms for the last bit to travel).

###### Q2 [SDE-2]: "Our users are far from the server and pages feel slow. Should we upgrade to a bigger bandwidth link?" What do you say?

**Answer:** Probably not. If pages need many small round trips, the time is dominated by **propagation delay and round-trip count**, which more bandwidth does not shorten. Better fixes: put content closer to users (a **CDN**), reduce round trips (HTTP/2 or HTTP/3, connection reuse, fewer redirects), and cache. Measure first with `ping` and `traceroute` to see where the time goes.

###### Q3 [SDE-2/3]: A link has 100 Mbps bandwidth and 50 ms RTT. A sender can have at most 64 KB unacknowledged at once. What throughput can it reach?

**Answer:** The BDP is about 625 KB, but the sender can only keep 64 KB in flight. It sends 64 KB, then waits one RTT for acknowledgments. Throughput ≈ 64 KB ÷ 0.05 s ≈ **1.28 MB/s ≈ 10 Mbps**, only about a tenth of the link. The fix is a bigger window (window scaling in TCP), not a faster link.

## 1.3: The OSI and TCP/IP Layered Models

Sending data across the world is a huge job. Engineers solved it by **splitting it into layers**, where each layer does one small job and relies on the layer below it. When one layer changes (say, Wi-Fi replaces Ethernet), the layers above do not notice.

It works like sending a gift abroad. You write a card (the message). A gift shop puts it in a box with the recipient's name. A courier adds a shipping label with a street address. A truck driver only needs the label on the outside. Each person does their own step and only looks at their own wrapper.

### The OSI model (7 layers)

| ##  | Layer            | Job                                                              | PDU name                               | Examples                                     |
| --- | ---------------- | ---------------------------------------------------------------- | -------------------------------------- | -------------------------------------------- |
| 7   | **Application**  | Services the user's program uses                                 | Data                                   | HTTP, DNS, SMTP, FTP, SSH                    |
| 6   | **Presentation** | Format, encryption, compression                                  | Data                                   | TLS (often placed here), JPEG, JSON encoding |
| 5   | **Session**      | Start, manage and end conversations                              | Data                                   | Session setup, RPC                           |
| 4   | **Transport**    | Delivery between _programs_ (ports), reliability                 | **Segment** (TCP) / **Datagram** (UDP) | TCP, UDP                                     |
| 3   | **Network**      | Delivery between _hosts across networks_ (IP addresses, routing) | **Packet**                             | IP, ICMP                                     |
| 2   | **Data Link**    | Delivery to the _next device on the same link_ (MAC addresses)   | **Frame**                              | Ethernet, Wi-Fi                              |
| 1   | **Physical**     | Raw bits as electrical, light or radio signals                   | **Bits**                               | Cables, fiber, radio                         |

Memory trick, top to bottom: **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing.

### The TCP/IP model (4 layers): what the Internet really uses

OSI is a teaching model. The Internet was built on the simpler **TCP/IP model**:

| TCP/IP layer              | Matches OSI layers | Examples             |
| ------------------------- | ------------------ | -------------------- |
| **Application**           | 7, 6, 5            | HTTP, DNS, TLS, SMTP |
| **Transport**             | 4                  | TCP, UDP             |
| **Internet**              | 3                  | IP, ICMP             |
| **Link** (Network access) | 2, 1               | Ethernet, Wi-Fi      |

Many books show a 5-layer version that splits Link into Data Link and Physical. In an interview, say which one you are using.

### Encapsulation: how data goes down the stack

When your browser sends a request, each layer **wraps** the data in its own header (and the link layer adds a trailer):

```
Application:   [ HTTP request                                    ]   data
Transport:     [ TCP hdr ][ HTTP request                         ]   segment
Internet:      [ IP hdr ][ TCP hdr ][ HTTP request               ]   packet
Link:    [ Eth hdr ][ IP hdr ][ TCP hdr ][ HTTP request ][ FCS ]    frame
Physical:      10110010 01100110 ...                                 bits
```

At the receiver the process reverses. This is **decapsulation**: the link layer removes the Ethernet header and checks the trailer, the IP layer removes the IP header, the transport layer removes the TCP header, and the application gets the HTTP request.

Each layer talks to its **peer** on the other side (TCP talks to TCP, IP talks to IP), but physically the data goes _down_ one stack and _up_ the other.

#### Sizes you should know

- Ethernet **MTU** is 1500 bytes of payload.
- An IPv4 header is at least **20 bytes**. A TCP header is at least **20 bytes**.
- So the largest TCP data in one packet (the **MSS**) is usually 1500 − 20 − 20 = **1460 bytes**.

### What is each layer's address?

| Layer       | Address it uses             | Identifies                       |
| ----------- | --------------------------- | -------------------------------- |
| Application | Names (`example.com`), URLs | A service or resource            |
| Transport   | **Port number**             | A program on a host              |
| Internet    | **IP address**              | A host on a network              |
| Link        | **MAC address**             | A network card on the local link |

### Interview traps

- "Which layer is a router? A switch? A firewall?" Router is **Layer 3**. A classic switch is **Layer 2**. A basic firewall works at Layers 3–4, and modern ones at Layer 7. A load balancer can be Layer 4 or Layer 7.
- "Where is TLS?" It sits between the application and transport layers. In OSI it is often placed at Layer 6, but in TCP/IP it is just part of the application layer using TCP.
- "Is OSI used in practice?" As a **vocabulary** (people say "Layer 7 load balancer"), yes. As an actual protocol suite, no. The Internet runs TCP/IP.
- PDU names: _segment_ is for TCP, _datagram_ for UDP, _packet_ for IP, _frame_ for the link layer. Mixing these up is common.

### Tricky questions and answers

#### Q1 [SDE-1]: Why are networks built in layers?

**Answer:** Layers split a hard problem into small, independent ones. Each layer has a clear job and a clear interface. You can **replace one layer without changing the others** (Wi-Fi instead of Ethernet, IPv6 instead of IPv4, HTTP/3 instead of HTTP/1.1), and different teams can work on different layers. The cost is some overhead from the extra headers.

#### Q2 [SDE-2]: A packet travels from host A through three routers to host B. Which addresses change at each hop, and which do not?

**Answer:** The **source and destination IP addresses stay the same** from A to B (unless NAT rewrites them, covered in Module 5). The **MAC addresses change at every hop**, because MAC addresses only mean something on one link. At each router, the incoming Ethernet frame is stripped, the router decides the next hop using the IP address, and a **new Ethernet frame** is built with the router's MAC as source and the next device's MAC as destination. The IP header's **TTL** is also reduced by one, and the header checksum is recalculated.

#### Q3 [SDE-2]: Why do we need both MAC addresses and IP addresses?

**Answer:** They solve different problems. A **MAC address** identifies a network card on the _local link_ and is fixed in hardware, but it has no structure, so it cannot be used to route across the world (there is no way to know which direction to send a given MAC). An **IP address** is _hierarchical_: it contains a network part and a host part, so routers can store one route for a whole network and forward packets toward it. IP finds the right network. MAC finishes the delivery on the last link.

## 1.4: Network Devices: Hub, Switch, Router and Friends

Different devices do different jobs, and the cleanest way to tell them apart is: **which layer's address do they read?**

- A **hub** reads nothing. It just repeats electrical signals.
- A **switch** reads the **MAC address** (Layer 2).
- A **router** reads the **IP address** (Layer 3).

### The devices, one by one

| Device                           | Layer  | What it does                                                                   |
| -------------------------------- | ------ | ------------------------------------------------------------------------------ |
| **NIC** (network interface card) | 1–2    | The hardware in your device that connects to the network. Has the MAC address. |
| **Hub**                          | 1      | Repeats every signal to **all** other ports. No intelligence. Obsolete.        |
| **Switch**                       | 2      | Forwards a frame **only to the port where the destination MAC lives**.         |
| **Router**                       | 3      | Forwards packets **between networks** using IP addresses and a routing table.  |
| **Access point (AP)**            | 1–2    | Connects Wi-Fi devices to the wired network.                                   |
| **Modem**                        | 1      | Converts between your ISP's line signal (cable, DSL, fiber) and Ethernet.      |
| **Firewall**                     | 3–7    | Allows or blocks traffic by rules (Module 5).                                  |
| **Load balancer**                | 4 or 7 | Spreads requests across many servers (Module 5).                               |

### How a switch learns

A switch keeps a **MAC address table** (also called a CAM table) that maps each MAC address to a port. It builds the table by watching traffic.

Suppose host A (MAC `AA`) on port 1 sends a frame to host B (MAC `BB`) on port 2, and the table is empty:

1. The frame arrives on **port 1**. The switch **learns**: "`AA` is on port 1."
2. It does not know where `BB` is, so it **floods** the frame out of every port except port 1.
3. B replies. The frame arrives on **port 2**. The switch learns: "`BB` is on port 2."
4. From now on, frames between A and B go **only** to the correct port.

Table entries expire after a few minutes if the device stays silent. A **broadcast** frame (destination `FF:FF:FF:FF:FF:FF`) is always flooded to everyone.

### How a router decides

A router has a **routing table**: a list of "to reach network X, send to next hop Y through interface Z." When a packet arrives, it finds the best match for the destination IP and forwards the packet. Module 2 covers this fully.

### Collision domains and broadcast domains

| Device | Collision domains        | Broadcast domains                                         |
| ------ | ------------------------ | --------------------------------------------------------- |
| Hub    | One for the whole device | One                                                       |
| Switch | **One per port**         | One (unless VLANs are used)                               |
| Router | One per interface        | **One per interface** (routers do not forward broadcasts) |

A **collision domain** is a group of devices that can interfere with each other's transmissions. A **broadcast domain** is a group that receives each other's broadcast frames. A good design keeps broadcast domains small, because every device must process every broadcast.

### Interview traps

- "Does a switch have an IP address?" A basic switch forwards by MAC and needs none. A _managed_ switch gets an IP only so you can log in to configure it. A **Layer 3 switch** can also route.
- "Hub vs. switch?" A hub sends to everyone (one shared medium, collisions possible). A switch sends only where needed, and every port is its own collision domain.
- A home "router" is really four devices in one: router, switch, access point, and often a modem.
- A switch is **not** a firewall. It forwards by MAC and has no security rules by default.

### Tricky questions and answers

#### Q1 [SDE-1/2]: What does a switch do when it receives a frame for a MAC address it has never seen?

**Answer:** It **floods** the frame out of all ports except the one it arrived on (this is called _unknown unicast flooding_). If the destination exists, it will reply, and the switch will then learn which port that MAC is on. This is why a switch works with no configuration.

#### Q2 [SDE-2]: Why can't you build the whole Internet out of switches?

**Answer:** Three reasons. First, a switch's MAC table would need an entry for every device in the world, because MAC addresses have no structure. Second, broadcasts and unknown-unicast flooding would reach everyone, so the network would drown in traffic. Third, there is no way to select a good path across many networks. Routers solve this because IP addresses are hierarchical, so one routing entry can cover millions of hosts, and broadcasts stop at the router.

#### Q3 [SDE-2/3]: Your office LAN is slow, and a packet capture shows huge amounts of broadcast traffic. What do you do?

**Answer:** The broadcast domain is too big. Split it into **VLANs** (virtual LANs), which are separate broadcast domains on the same switches, and connect them with a router or Layer 3 switch so only the traffic that needs to cross does so. Also check for a **loop** (a cable plugged in a circle), which can cause a broadcast storm. Switches use **Spanning Tree Protocol** to block loops.

## 1.5: Ethernet, MAC Addresses and Frames

### The idea in plain words

On one local link, devices need a way to say "this data is for that card." That is the job of the **Data Link layer**, and for wired networks the dominant standard is **Ethernet**. Each network card has a **MAC address**, and the data travels in **frames**.

### The MAC address

- It is **48 bits (6 bytes)**, written in hexadecimal: `00:1A:2B:3C:4D:5E`.
- The first 3 bytes are the **OUI** (Organizationally Unique Identifier), which identifies the manufacturer. The last 3 bytes are assigned by that manufacturer.
- It is **meant to be unique** and is set in the hardware, but software can change ("spoof") it. Phones often use random MACs for privacy.
- Special values: `FF:FF:FF:FF:FF:FF` is the **broadcast** address (everyone). A MAC whose lowest bit in the first byte is 1 is a **multicast** address (a group).

### The Ethernet frame

```
| Dest MAC | Src MAC | EtherType | Payload (46–1500 bytes) | FCS |
|  6 bytes | 6 bytes |  2 bytes  |   up to the MTU         | 4 B |
```

- **EtherType** says what is inside (for example `0x0800` means IPv4, `0x86DD` means IPv6, `0x0806` means ARP).
- **FCS** (frame check sequence) is a checksum (CRC) that lets the receiver detect damage.
- The payload is at most **1500 bytes** (the MTU). The smallest legal frame is **64 bytes**, and short payloads are padded.
- With a **VLAN tag** (802.1Q), 4 more bytes are added to the header.

### Wired vs. wireless

- **Wired Ethernet** today uses **switches with full-duplex links**: each device can send and receive at the same time on its own cable, so collisions do not happen. The old rule for shared cables was **CSMA/CD** (listen, send, detect collisions).
- **Wi-Fi (802.11)** shares the radio air, so devices cannot reliably detect collisions while sending. It uses **CSMA/CA** (listen first, wait a random time, send, and wait for an acknowledgment).

### Look at your own addresses

```
$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 00:1a:2b:3c:4d:5e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
```

Here `link/ether` is the **MAC address**, `mtu 1500` is the MTU, and `inet 192.168.1.10/24` is the **IP address** (Module 2 explains the `/24`). On macOS use `ifconfig`, and on Windows use `ipconfig /all`.

### Interview traps

- "Is the MAC address globally unique and permanent?" Meant to be, but it can be changed in software, and many devices randomize it. Never use it for security.
- "What happens if a packet is bigger than the MTU?" IP **fragments** it into smaller pieces (or drops it and sends an error if the packet is marked "don't fragment"). Fragmentation is costly, so TCP chooses its segment size (MSS) to avoid it.
- "Why is the minimum frame 64 bytes?" Historically, so a sender would still be transmitting when a collision signal came back on a shared cable. Today it is kept for compatibility.

### Tricky questions and answers

#### Q1 [SDE-2]: Host A wants to send to host B on the same LAN. A knows B's IP address but not its MAC. How does it learn it?

**Answer:** It uses **ARP** (Address Resolution Protocol). A broadcasts an Ethernet frame asking "Who has IP 192.168.1.20? Tell 192.168.1.10." Every device on the LAN sees it, but only B answers, with its MAC address in a unicast reply. A stores the answer in its **ARP cache** (`ip neigh` or `arp -a`) so it does not need to ask again. ARP gets a full treatment in Module 2.

#### Q2 [SDE-2/3]: Why does Ethernet use a 1500-byte MTU even though bigger frames would be more efficient?

**Answer:** It is mostly history and compatibility. 1500 bytes became the standard on early Ethernet, and the whole Internet grew up around it. Bigger frames (**jumbo frames**, around 9000 bytes) do reduce per-packet overhead and are used inside data centers, but they only work if **every device on the path** supports them. On the open Internet the lowest common size wins, so 1500 remains the safe default. A mismatch causes dropped or fragmented packets, which is a classic hard-to-find bug.

---

## Hands-On Lab: Explore Your Own Network

Run these on your machine and read the output using what you learned.

```
1. ip addr            ## find your IP address, MAC address and MTU
2. ip route           ## find your default gateway (your router)
3. ping -c 4 <gateway IP>        ## latency to your own router: should be 1–5 ms
4. ping -c 4 example.com         ## latency to the Internet: tens of ms
5. traceroute example.com        ## each line is one router on the path
6. ip neigh                      ## see MAC addresses your machine has learned
```

Questions to answer for yourself:

- Is the latency to your router much smaller than to a distant site? Which delay explains the difference?
- How many routers does the path to `example.com` cross? Does the number change from day to day?
- In `ip neigh`, can you find your router's MAC address? Which layer's address is it?

(`traceroute` works by sending packets with a TTL of 1, 2, 3 and so on. Each router that sees the TTL reach zero throws the packet away and reports back, which reveals itself. Module 2 explains this fully. On Windows the command is `tracert`.)

---

## Module 1 Cheat Sheet

| Concept               | One-line interview answer                                                                 |
| --------------------- | ----------------------------------------------------------------------------------------- |
| Network               | Nodes + links + protocols. The Internet is a network of networks.                         |
| Packet                | Header (control information) + payload (data). Packets share links (packet switching).    |
| Bandwidth vs. latency | Bandwidth = capacity. Latency = travel time. More bandwidth does not cut latency.         |
| Throughput            | The rate you actually achieve, limited by the slowest link and by protocol behavior.      |
| Nodal delay           | Processing + queuing + transmission (`L/R`) + propagation (`d/s`).                        |
| BDP                   | Bandwidth × RTT = data in flight needed to fill the link.                                 |
| OSI layers            | Physical, Data Link, Network, Transport, Session, Presentation, Application.              |
| TCP/IP layers         | Link, Internet, Transport, Application. This is what the Internet uses.                   |
| Encapsulation         | Each layer adds its own header going down. Decapsulation removes them going up.           |
| PDU names             | Bits, frame, packet, segment (TCP) or datagram (UDP), data.                               |
| Hub / switch / router | Layer 1 repeats to all. Layer 2 forwards by MAC. Layer 3 forwards by IP between networks. |
| Switch learning       | Learns source MAC per port. Floods unknown destinations and broadcasts.                   |
| Per-hop changes       | MAC addresses change at every hop. IP addresses stay (without NAT). TTL decreases.        |
| MAC vs. IP            | MAC = local link, flat. IP = hierarchical, routable across networks.                      |
| Frame facts           | MTU 1500, header 14 bytes, FCS 4 bytes, minimum frame 64 bytes. TCP MSS is usually 1460.  |
