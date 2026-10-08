---
title: "Module 2: IP Addressing, Subnetting & Routing"
description: "Learn how IPv4 and IPv6 addresses work, how subnet masks and CIDR divide networks, how DHCP and ARP connect hosts, and how routers choose a path - with worked subnetting examples and interview questions."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/networking-module-2.png"
tags: [Networking, IP-Addressing, Subnetting, Routing, Interview-Prep]
keywords: ["IP addressing and subnetting tutorial", "CIDR notation explained", "How routing tables work", "IPv4 vs IPv6"]
---

# Module 2: IP Addressing, Subnetting & Routing

![Networking Module 2](/images/networking-module-2.png)

**Previous:** [Module 1: Basics & the Layered Models](computer-networks/networking-module-1-basics-and-layered-models)

### How this module connects to Module 1

In Module 1 you learned that the **Internet layer** moves packets between networks, that **IP addresses** are hierarchical (that is why routers can forward by them), and that MAC addresses only work on one link. Module 2 opens that layer: what an IP address looks like, how a network is cut into subnets, how a host gets its address, and how routers decide where each packet goes next.

### Words you will meet in this module

| Word | Simple meaning |
|---|---|
| **IP address** | A logical address with a *network part* and a *host part*. |
| **Subnet** | A smaller network carved out of a bigger one. |
| **Subnet mask** | A pattern that shows which bits of an address are the network part. |
| **CIDR** | Classless Inter-Domain Routing: writing a network as `address/prefix-length`, like `10.0.0.0/8`. |
| **Prefix length** | The number after the slash: how many leading bits are the network part. |
| **Gateway** | The router a host sends traffic to when the destination is on another network. |
| **Routing table** | A router's list of "to reach this network, send to that next hop." |
| **Next hop** | The next router (or device) on the path. |
| **DHCP** | The service that hands out IP addresses automatically. |
| **ARP** | The protocol that finds the MAC address that belongs to an IPv4 address on the local link. |
| **TTL** | Time To Live: a counter in each packet that drops by one at every router. |
| **AS** | Autonomous System: a network under one administrative control, with its own routing policy. |

## 2.1: IPv4 Addresses

### The idea in plain words

An **IPv4 address** is a 32-bit number. People write it as four numbers from 0 to 255, separated by dots (**dotted decimal**): `192.168.1.10`. Each number is one **octet**, which is 8 bits.

Think of a street address: "Flat 12, Green Road, Pune." The **network part** (like "Green Road, Pune") is shared by everyone in the same network. The **host part** (like "Flat 12") picks one device inside it. Routers only look at the network part, so they can send a packet toward the right "street" without knowing every flat in the world.

### Looking at the bits

```
192.168.1.10  =  11000000 . 10101000 . 00000001 . 00001010
                 └────── 4 octets, 32 bits in total ──────┘
```

There are 2³² = about **4.3 billion** possible IPv4 addresses. That sounded like plenty in 1981, and it ran out. This shortage is why NAT (Module 5) and IPv6 (Topic 2.3) exist.

### Public and private addresses

A **public** address is unique on the Internet and can be reached from anywhere. A **private** address is for use inside a home, office or data center. Many different networks reuse the same private ranges, and private addresses are **not routable on the public Internet**.

| Range | CIDR | Size | Used for |
|---|---|---|---|
| `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | 16,777,216 | Large private networks, clouds |
| `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | 1,048,576 | Medium private networks |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | 65,536 | Home and small office |

These three ranges are defined in **RFC 1918**.

### Special addresses

| Address / range | Meaning |
|---|---|
| `127.0.0.0/8` (usually `127.0.0.1`) | **Loopback**: "this machine." Never leaves your computer. |
| `169.254.0.0/16` | **Link-local** (APIPA): a self-assigned address used when DHCP fails. |
| `100.64.0.0/10` | **Carrier-grade NAT** space used by ISPs. |
| `224.0.0.0/4` | **Multicast**: one sender, a group of receivers. |
| `255.255.255.255` | **Limited broadcast**: everyone on the local link. |
| Host bits all 1 (for example `192.168.1.255` in a `/24`) | **Directed broadcast** of that subnet. |
| Host bits all 0 (for example `192.168.1.0` in a `/24`) | The **network address** itself. |
| `0.0.0.0` | "Unspecified" or "this host." In a routing table, `0.0.0.0/0` means the **default route**. |
| `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` | Reserved for documentation and examples. |

### A little history: the old "classes"

Early IP split the space into fixed classes:

| Class | First octet | Default mask | Note |
|---|---|---|---|
| A | 0 – 127 | /8 | Few networks, huge size |
| B | 128 – 191 | /16 | Medium |
| C | 192 – 223 | /24 | Many small networks |
| D | 224 – 239 | n/a | Multicast |
| E | 240 – 255 | n/a | Reserved |

This wasted a lot of addresses. In 1993 it was replaced by **CIDR** (Topic 2.2), which allows any prefix length. You may still hear "Class C" used loosely to mean a `/24`.

### Interview traps

- **`172.32.x.x` is not private.** The private block is only `172.16.0.0` to `172.31.255.255`. This trap appears often.
- "Is `192.168.1.0` a valid host?" In a `/24` it is the **network address**, so it is not for hosts. And `192.168.1.255` is the broadcast. But in a `/23`, `192.168.1.0` *is* a valid host. Always ask for the mask.
- `127.0.0.1` is not a network address you can ping from another machine. It only reaches your own machine.
- Saying "the class of this address" in an interview is outdated. Say "the prefix length."

### Tricky questions and answers

#### Q1 [SDE-1]: Which of these are private addresses? `10.200.5.1`, `172.20.1.1`, `172.32.1.1`, `192.169.1.1`

**Answer:** `10.200.5.1` is private (inside `10/8`). `172.20.1.1` is private (inside `172.16.0.0/12`, which covers `172.16`–`172.31`). `172.32.1.1` is **public**, because it is just outside that range. `192.169.1.1` is **public**, because only `192.168.x.x` is private.

#### Q2 [SDE-2]: Why can two different companies both use `10.0.0.5` and not conflict?

**Answer:** Private addresses are only meaningful inside one network. Packets with private source or destination addresses are not routed on the public Internet. If two such networks need to talk, you need NAT (Module 5), or you must renumber one of them. This is a real problem when two companies merge, or when you connect two cloud networks that both used `10.0.0.0/16`.

## 2.2: Subnet Masks, CIDR and Subnetting

### The idea in plain words

A **subnet mask** tells you where the network part of an address ends and the host part begins. It is 32 bits long: ones for the network part, zeros for the host part.

```
Mask 255.255.255.0 =  11111111.11111111.11111111.00000000
                      └───── network (24 bits) ─────┘└ host (8) ┘
```

Writing the mask in full is clumsy, so we count the ones and write them after a slash. This is **CIDR notation**: `192.168.1.10/24`. The number after the slash is the **prefix length**.

**Subnetting** means taking one network and cutting it into smaller ones, by moving the line between network bits and host bits to the right.

### The three things you can calculate

For any address with a prefix:

1. **Network address**: the address with all host bits set to 0.
2. **Broadcast address**: the address with all host bits set to 1.
3. **Usable hosts**: everything in between. The number is **2^(host bits) − 2** (the two removed are the network and broadcast addresses).

How the computer finds the network address: it does a bitwise **AND** of the address with the mask.

```
192.168.1.10   = 11000000.10101000.00000001.00001010
Mask /24       = 11111111.11111111.11111111.00000000
AND            = 11000000.10101000.00000001.00000000  = 192.168.1.0
```

### Quick reference table

| Prefix | Mask | Block size* | Usable hosts |
|---|---|---|---|
| /8 | 255.0.0.0 | n/a | 16,777,214 |
| /16 | 255.255.0.0 | n/a | 65,534 |
| /24 | 255.255.255.0 | n/a | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |

*Block size = how far apart the subnets are in the last octet: **256 − (mask value in that octet)**.

Special cases: a **/31** has 2 addresses and both are usable (for point-to-point links, defined in RFC 3021). A **/32** is exactly one address (a single host or a specific route).

### The fast method (the "block size" trick)

Find the network of `192.168.10.77/26`:

1. The mask is `255.255.255.192`. The interesting octet is the last one (192).
2. Block size = 256 − 192 = **64**. The subnets start at 0, 64, 128, 192.
3. 77 falls in the block that starts at **64** (64 to 127).
4. So: **network = 192.168.10.64**, **broadcast = 192.168.10.127**, usable hosts = **192.168.10.65 to 192.168.10.126** (62 hosts).

### Splitting a network (subnetting)

Split `192.168.1.0/24` into 4 equal subnets. Four subnets need 2 extra bits (2² = 4), so the prefix goes from /24 to /26:

| Subnet | Range | Usable hosts | Broadcast |
|---|---|---|---|
| `192.168.1.0/26` | .0 – .63 | .1 – .62 | .63 |
| `192.168.1.64/26` | .64 – .127 | .65 – .126 | .127 |
| `192.168.1.128/26` | .128 – .191 | .129 – .190 | .191 |
| `192.168.1.192/26` | .192 – .255 | .193 – .254 | .255 |

### VLSM: different sizes for different needs

**Variable Length Subnet Masking** lets you give each subnet the size it needs. The rule: **allocate the largest subnet first**, so everything stays aligned.

Example: split `192.168.5.0/24` for 100 hosts, 50 hosts, 20 hosts, and one router-to-router link.

| Need | Host bits needed | Prefix | Allocated subnet |
|---|---|---|---|
| 100 hosts | 7 (128 − 2 = 126 usable) | /25 | `192.168.5.0/25` (.0 – .127) |
| 50 hosts | 6 (64 − 2 = 62 usable) | /26 | `192.168.5.128/26` (.128 – .191) |
| 20 hosts | 5 (32 − 2 = 30 usable) | /27 | `192.168.5.192/27` (.192 – .223) |
| 2 (link) | 2 (4 − 2 = 2 usable) | /30 | `192.168.5.224/30` (.224 – .227) |

Addresses `.228` to `.255` are still free for future use.

### Summarization (supernetting)

The opposite of subnetting: combine several adjacent networks into one bigger route. `192.168.0.0/24`, `192.168.1.0/24`, `192.168.2.0/24` and `192.168.3.0/24` can be announced as one route, **`192.168.0.0/22`**. This keeps routing tables small, which is the whole point of hierarchical addressing. The blocks must be adjacent and aligned to the block size.

### Useful commands

```
$ ip addr show eth0
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0

$ python3 -c "import ipaddress as i; n=i.ip_network('192.168.10.77/26',strict=False); print(n, n.broadcast_address, n.num_addresses-2)"
192.168.10.64/26 192.168.10.127 62
```

Python's `ipaddress` module is a quick way to check your subnet math.

### Interview traps

- Forgetting to subtract **2** for the network and broadcast addresses (except /31 and /32).
- Subnetting without aligning to the block size. A `/26` must start at a multiple of 64 in the last octet. `192.168.1.10/26` does not "start" at .10.
- Mixing up the **mask** (255.255.255.192) and the **wildcard mask** (0.0.0.63, used in some router configs), which is the mask flipped.
- Assuming all subnets must be the same size. VLSM exists exactly so they do not have to be.
- In cloud networks (AWS VPC, for example), the provider reserves a few extra addresses in each subnet, so "usable hosts" is lower than the textbook value. Check the provider's rules.

### Tricky questions and answers

#### Q1 [SDE-1/2]: Given `192.168.10.77/26`, what are the network address, broadcast address and usable range?

**Answer:** Mask `255.255.255.192`, block size 64. The address 77 lies in the 64–127 block. **Network `192.168.10.64`**, **broadcast `192.168.10.127`**, usable **`192.168.10.65` – `192.168.10.126`** (62 hosts).

#### Q2 [SDE-2]: Are `10.0.5.20/22` and `10.0.7.9/22` on the same subnet?

**Answer:** A `/22` mask is `255.255.252.0`, so the interesting octet is the third one, with a block size of 256 − 252 = 4. For `10.0.5.20`, the third octet 5 falls in the block 4–7. For `10.0.7.9`, the third octet 7 falls in the same block 4–7. Both are in **`10.0.4.0/22`** (range `10.0.4.0` – `10.0.7.255`), so **yes, the same subnet**. They can talk directly, without a router.

#### Q3 [SDE-2]: How many `/24` networks fit inside a `/16`? And how many hosts does a `/20` hold?

**Answer:** A `/16` has 8 more bits than a `/24` network, so 2⁸ = **256** `/24` networks fit inside it. A `/20` has 32 − 20 = 12 host bits, so 2¹² − 2 = **4,094** usable hosts.

#### Q4 [SDE-3]: You are designing a cloud network for a company. How would you plan the address space?

**Answer:** Choose a private range big enough for growth (for example `10.0.0.0/8` split into `/16` per region or environment). Give each environment its own non-overlapping block, because overlapping blocks make later **peering, VPN or merging** impossible without renumbering. Use smaller subnets for tiers (public, application, database) in each availability zone. Leave **unallocated space** for the future. Keep the blocks aligned so they can be **summarized** into a few routes. Plan for the provider's reserved addresses, and for pods or containers if you use Kubernetes, since they can use many addresses.

## 2.3: IPv6

### The idea in plain words

IPv4 ran out of addresses. **IPv6** is the replacement. Its addresses are **128 bits**, enough for about 3.4 × 10³⁸ addresses, so many that every device can have its own public address and there is no need for NAT.

An IPv6 address is written as **eight groups of four hexadecimal digits**, separated by colons:

```
2001:0db8:0000:0000:0000:ff00:0042:8329
```

### Shortening the notation

Two rules make it readable:

1. Remove **leading zeros** in each group.
2. Replace **one** run of all-zero groups with `::` (only once per address, and use it for the longest run).

```
2001:0db8:0000:0000:0000:ff00:0042:8329
→ 2001:db8:0:0:0:ff00:42:8329         (rule 1)
→ 2001:db8::ff00:42:8329              (rule 2)
```

### Types of IPv6 addresses

| Type | Range | Meaning |
|---|---|---|
| **Global unicast** | `2000::/3` | Public, routable on the Internet |
| **Link-local** | `fe80::/10` | Valid on one link only. Every IPv6 interface has one |
| **Unique local** | `fc00::/7` (in practice `fd00::/8`) | Private, like IPv4 private ranges |
| **Multicast** | `ff00::/8` | One-to-many |
| **Loopback** | `::1` | This machine |
| **Unspecified** | `::` | "No address yet" |
| **Documentation** | `2001:db8::/32` | For examples |

There is **no broadcast** in IPv6. Multicast replaces it.

A normal LAN subnet is a **/64**: the first 64 bits are the network prefix, and the last 64 bits identify the interface. ISPs usually give a home a `/56` or `/48`, so you can create many `/64` subnets.

### What is different from IPv4

- **Bigger address space**, so NAT is not needed for address shortage.
- **Simpler fixed header** (40 bytes), with no header checksum, so routers do less work. Optional features live in **extension headers**.
- **Routers do not fragment packets.** Only the sender may, so path MTU discovery is more important.
- **Auto-configuration (SLAAC):** a host can build its own address from the network prefix that the router advertises, without a DHCP server.
- **NDP** (Neighbor Discovery Protocol) replaces ARP. It runs over ICMPv6 and uses multicast instead of broadcast.
- **Transition tools:** most networks run **dual stack** (both IPv4 and IPv6 at once). Others use tunneling, or translation (NAT64) to connect IPv6-only clients to IPv4 servers.

### Interview traps

- "Does IPv6 make networks more secure?" Not by itself. IPsec support was once required, but it is not enabled by default. Removing NAT does not remove the need for a **firewall**.
- Using `::` twice in one address is invalid, because it would be ambiguous.
- "Why `/64` for every LAN?" Many features, such as SLAAC, expect a 64-bit interface ID. Using smaller subnets breaks them.
- Forgetting that IPv6 addresses in URLs need square brackets: `http://[2001:db8::1]:8080/`.

### Tricky questions and answers

#### Q1 [SDE-2]: Shorten `2001:0db8:0000:0000:0000:0000:0000:0001`.

**Answer:** Remove leading zeros, then collapse the longest run of zero groups: **`2001:db8::1`**.

#### Q2 [SDE-2/3]: Why does IPv6 not need NAT, and why does it have no broadcast?

**Answer:** NAT was invented mainly to share a few public IPv4 addresses among many devices. IPv6 has so many addresses that every device can have a global one, so that pressure is gone. (Hiding internal addresses is not a real security feature. A stateful **firewall** gives the protection.) Broadcast was removed because it interrupts every device on a link even if only one cares. IPv6 uses **multicast** groups, so only interested devices process the packet. This reduces load on all hosts.

#### Q3 [SDE-3]: You are rolling out IPv6 in a company with an IPv4 network. What is your approach?

**Answer:** Start with **dual stack**, since it is the least risky: every device keeps IPv4 and also gets IPv6. Request a large enough prefix from the ISP (such as `/48`), design a `/64` per subnet, and apply the **same firewall policy** to IPv6 as to IPv4 (a common mistake is leaving IPv6 open). Make sure DNS has both `A` and `AAAA` records, test applications that hard-code IPv4 addresses, and monitor both stacks. Later, plan to retire IPv4 internally and use NAT64 where an IPv6-only network must reach legacy IPv4 services.

## 2.4: Getting Connected: DHCP and ARP

### The idea in plain words

You plug a laptop into a network. It has no IP address, does not know its gateway, and does not know anybody's MAC address. How does it become useful in a second or two? Two small protocols do the work:

- **DHCP** gives the laptop an IP address and the other settings it needs.
- **ARP** finds the MAC address that belongs to an IP address on the local link.

### DHCP: the four steps (DORA)

**DHCP** (Dynamic Host Configuration Protocol) runs over UDP: the server uses port **67**, the client uses port **68**.

| Step | Name | What happens |
|---|---|---|
| 1 | **Discover** | The client broadcasts "Is there a DHCP server?" from `0.0.0.0` to `255.255.255.255`. |
| 2 | **Offer** | A server offers an address and settings. |
| 3 | **Request** | The client broadcasts "I accept that offer." (Broadcast, so other servers learn their offers were not chosen.) |
| 4 | **Acknowledge** | The server confirms and starts the **lease**. |

A DHCP reply usually carries: the **IP address**, the **subnet mask**, the **default gateway**, the **DNS servers**, and the **lease time**.

- The client tries to **renew** at 50% of the lease, and again at about 87.5% if the first try fails.
- If no DHCP server answers, many systems self-assign a `169.254.x.x` address (**APIPA**). Seeing this address means "DHCP is broken."
- Broadcasts do not cross routers, so for a DHCP server on another subnet, the router runs a **DHCP relay agent** that forwards the requests.

### ARP: from IP address to MAC address

To put a packet on the local link, the sender needs the destination's **MAC address**. If it only has the IP address, it uses **ARP**:

1. The sender broadcasts: "Who has `192.168.1.20`? Tell `192.168.1.10`." (Destination MAC `FF:FF:FF:FF:FF:FF`.)
2. Every device on the link sees it, but only the owner replies, with a **unicast** message containing its MAC address.
3. The sender stores the answer in its **ARP cache** for a short time.

```
#$ ip neigh                       # (or: arp -a)
192.168.1.1  dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
192.168.1.20 dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE
```

**ARP only works inside one broadcast domain.** To reach a host on another network, the sender does not ARP for that host. It ARPs for its **default gateway's** MAC address, and sends the frame to the gateway (see Topic 2.5).

**Gratuitous ARP** is an unrequested ARP announcement ("my IP is now at this MAC"), used when a device comes up or fails over to a backup.

### The whole story: plugging in a laptop

1. The link comes up.
2. **DHCP DORA**: the laptop gets IP `192.168.1.10/24`, gateway `192.168.1.1`, and DNS server.
3. The user opens `example.com`. DNS (Module 4) returns an IP address, for instance `93.184.216.34`.
4. The laptop checks: is `93.184.216.34` inside my `/24`? No. So the next hop is the **default gateway**.
5. **ARP** asks "Who has `192.168.1.1`?" and gets the router's MAC.
6. The laptop sends the frame: **destination MAC = router**, but **destination IP = `93.184.216.34`**.

### Interview traps

- "Does ARP work across routers?" No. It is limited to the local broadcast domain.
- **ARP spoofing (poisoning):** ARP has no authentication, so an attacker can reply with a false MAC and redirect traffic through itself (a man-in-the-middle attack). Defenses: dynamic ARP inspection on switches, static entries for critical hosts, and encryption (TLS) so intercepted data is useless.
- A **rogue DHCP server** on a network can hand out wrong gateways or DNS servers. Switches offer **DHCP snooping** to allow DHCP replies only from trusted ports.
- Two devices with the **same IP** cause intermittent failures, because the ARP cache flips between two MACs.

### Tricky questions and answers

#### Q1 [SDE-2]: A host sends a packet to a server on another network. Which destination MAC and which destination IP are in the frame on the first hop?

**Answer:** Destination **IP = the server's IP** (it never changes along the way). Destination **MAC = the default gateway's MAC**, because the next device on the local link is the router. The host learned the gateway's MAC through ARP.

#### Q2 [SDE-2]: A laptop shows the address `169.254.12.7`. What happened?

**Answer:** The laptop did not receive an answer to its DHCP Discover, so it gave itself a link-local (APIPA) address. Causes: the DHCP server is down or its address pool is empty, the cable or Wi-Fi link does not reach the server, a VLAN is misconfigured, or a DHCP relay is missing. The laptop can only talk to other `169.254.x.x` devices on the same link, and cannot reach the Internet.

#### Q3 [SDE-3]: How can an attacker abuse ARP, and how do you defend against it?

**Answer:** The attacker sends forged ARP replies saying "the gateway's IP is at *my* MAC." Victims update their ARP caches and send traffic to the attacker, who forwards it onward and reads or changes it. Defenses: **Dynamic ARP Inspection** on switches (validates ARP against the DHCP snooping table), port security that limits MACs per port, static ARP entries on critical hosts, VLAN segmentation, and **end-to-end encryption** such as TLS or SSH, so that a man in the middle sees only ciphertext.

## 2.5: Routing

### The idea in plain words

**Routing** is how a packet finds its way across many networks. Each router only knows enough to decide the **next hop**. It does not plan the whole trip. Like asking for directions at every junction, each router points the packet one step closer to its goal.

### Step 1: the host decides "local or remote?"

Before any router is involved, the sending host does one test. It takes the destination IP and applies **its own subnet mask**:

- If the destination is in **the same subnet**, send it **directly** (ARP for the destination's MAC).
- If not, send it to the **default gateway**.

```
$ ip route
default via 192.168.1.1 dev eth0 proto dhcp metric 100
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
```

The first line is the **default route**: "for anything not matched elsewhere, go via `192.168.1.1`." The second says "the `192.168.1.0/24` network is directly connected on `eth0`."

### Step 2: the router looks up its routing table

A **routing table** is a list of entries: **destination prefix, next hop, outgoing interface (and a metric)**. When a packet arrives, the router picks the entry that matches the destination with the **longest prefix** (the most specific match). This is the **longest prefix match** rule.

Example routing table:

| Destination | Next hop | Interface |
|---|---|---|
| `10.1.0.0/16` | `192.168.1.2` | eth1 |
| `10.1.5.0/24` | `192.168.1.3` | eth2 |
| `0.0.0.0/0` (default) | `192.168.1.1` | eth0 |

- A packet to **`10.1.5.9`** matches `/16`, `/24` and `/0`. The longest is `/24`, so it goes to **`192.168.1.3`**.
- A packet to **`10.1.9.9`** matches `/16` and `/0`. The longest is `/16`, so it goes to **`192.168.1.2`**.
- A packet to **`8.8.8.8`** matches only the default route, so it goes to **`192.168.1.1`**.
- If nothing matches and there is no default route, the router **drops the packet** and sends an ICMP "destination unreachable."

At each hop the router also reduces the **TTL** by one. If the TTL reaches 0, it drops the packet and sends back an ICMP "time exceeded." This prevents packets from circling forever in a routing loop.

### How routers fill their tables

| Method | Idea | Good for |
|---|---|---|
| **Directly connected** | Networks on the router's own interfaces appear automatically | Always |
| **Static routes** | An administrator types them in | Small networks, default routes, stub sites |
| **Dynamic routing** | Routers exchange information using a protocol and adapt to failures | Large or changing networks |

#### Dynamic routing protocols

| Protocol | Type | How it works | Used |
|---|---|---|---|
| **RIP** | Distance-vector | Counts hops (max 15). Simple, slow to converge | Rare today |
| **OSPF** | Link-state | Every router learns the full map and runs a shortest-path (Dijkstra) calculation. Uses areas to scale | Inside organizations (an **IGP**, interior gateway protocol) |
| **BGP** | Path-vector | Exchanges routes between **autonomous systems**, chosen by policy, not just distance. Runs over TCP port 179 | **Between** organizations; it holds the Internet together (an **EGP**) |

An **autonomous system (AS)** is a network under one administration with its own routing policy, identified by an **AS number**. The Internet is thousands of ASes (ISPs, clouds, large companies) connected by BGP.

Two terms: **convergence** is the time it takes all routers to agree on the new best paths after a change. The **control plane** is the part of a router that builds the table (the protocols), and the **data plane** is the part that quickly forwards packets using the table.

### Real incidents to remember

- **BGP mistakes can take services offline or redirect traffic.** In 2021, Facebook's networks became unreachable for hours after a configuration change withdrew its BGP routes, and its DNS servers vanished with them.
- BGP trusts what neighbors announce, so a wrong announcement can hijack traffic. This is a known weakness, and security work such as RPKI aims to fix it.

### Interview traps

- **Longest prefix match, not "first match" or "lowest metric."** The metric only decides between equally specific routes.
- Saying "routers know the full path to the destination." They only know the **next hop**.
- A router with **no default route** drops traffic to unknown networks, and a host with **no default gateway** can only talk to its own subnet. A very common cause of "I can ping local machines but not the Internet."
- Static vs. dynamic: static routes do not adapt to failures by themselves. A static route to a dead next hop silently loses traffic.
- **Asymmetric routing:** the path from A to B can differ from B to A. This is normal on the Internet, but it can break stateful firewalls.

### Tricky questions and answers

#### Q1 [SDE-2]: A router has routes for `10.0.0.0/8` via R1, `10.1.0.0/16` via R2, and a default route via R3. Where do packets to `10.1.2.3`, `10.2.3.4` and `172.16.0.1` go?

**Answer:** `10.1.2.3` matches `/8`, `/16` and the default. The longest prefix is `/16`, so it goes to **R2**. `10.2.3.4` matches `/8` and the default, so it goes to **R1**. `172.16.0.1` matches only the default, so it goes to **R3**.

#### Q2 [SDE-2/3]: Why does the Internet use BGP between organizations instead of OSPF everywhere?

**Answer:** OSPF assumes every router belongs to one organization that shares full topology information and trusts the others. It does not scale to the whole Internet, and different companies will not expose their internal maps. **BGP** carries only reachability for whole prefixes, scales to hundreds of thousands of routes, and lets each AS apply **policy** (which neighbors to prefer, what to announce, and for how much money). Inside an AS, an IGP like OSPF finds the fastest paths. Between ASes, BGP applies business rules.

#### Q3 [SDE-3]: How do routing loops get stopped, and how are they prevented?

**Answer:** Two layers of defense. **During forwarding**, the **TTL** field is reduced at every hop, and a packet whose TTL reaches zero is dropped, so a loop cannot last forever. **In the routing protocols**, each has its own loop prevention: link-state protocols like OSPF give every router the same map, so they calculate loop-free paths. Distance-vector protocols use tricks such as **split horizon** and a maximum hop count (RIP's 15). BGP rejects any route that already contains its own AS number in the path.

## 2.6: The IP Header, ICMP, Ping and Traceroute

### The idea in plain words

The IP header is the "envelope" every packet carries. **ICMP** is IP's built-in **message system**: it is how routers and hosts report problems ("that network is unreachable," "your packet took too long") and how tools like `ping` and `traceroute` work.

### The IPv4 header

A minimum of **20 bytes**:

| Field | Purpose |
|---|---|
| **Version** | 4 for IPv4 |
| **Header length (IHL)** | Where the header ends (options may follow) |
| **DSCP / ECN** | Traffic priority and congestion signals |
| **Total length** | Size of the whole packet (up to 65,535 bytes) |
| **Identification, Flags, Fragment offset** | Used to split and rejoin large packets |
| **TTL** | Hop limit. Decreased by each router |
| **Protocol** | What is inside: **1** = ICMP, **6** = TCP, **17** = UDP |
| **Header checksum** | Detects damage to the header. Recalculated at each hop because TTL changes |
| **Source IP, Destination IP** | The addresses |

### Fragmentation and MTU

If a packet is larger than the **MTU** of the next link, IPv4 can **fragment** it: split it into pieces, each with its own header, which are reassembled **only at the destination**. Fragmentation is costly and fragile (if one piece is lost, the whole packet is lost). Two flags help: **DF** (Don't Fragment) forbids splitting, and **MF** (More Fragments) says "more pieces follow."

**Path MTU Discovery (PMTUD)** avoids fragmentation: the sender sets DF. A router that cannot forward the packet drops it and sends back an ICMP "**Fragmentation needed**" message (Type 3, Code 4) with the smaller MTU. The sender then reduces its packet size.

### ICMP: the messenger

ICMP messages are carried inside IP packets (protocol number 1). The important types:

| Type | Name | Used for |
|---|---|---|
| 8 / 0 | Echo request / Echo reply | **ping** |
| 3 | Destination unreachable | Network, host, port or fragmentation problems |
| 11 | Time exceeded | TTL reached zero (used by **traceroute**) |
| 5 | Redirect | A router telling a host about a better gateway |

### How `ping` works

`ping` sends an **ICMP Echo Request** and waits for an **Echo Reply**, then prints the **round-trip time**. It tells you whether a host is reachable at the IP level, and how long the trip takes. Note: a host may **block** ICMP, so "no ping reply" does not always mean "host is down."

### How `traceroute` works

It uses the TTL on purpose:

1. Send a packet with **TTL = 1**. The first router drops it and returns "Time exceeded," revealing its address.
2. Send a packet with **TTL = 2**. The second router replies.
3. Continue with 3, 4, 5... until the packet reaches the destination.

On Linux and macOS, the probes are UDP packets to high ports (the destination answers with "port unreachable"). On Windows, `tracert` uses ICMP echo.

```
$ traceroute example.com
 1  192.168.1.1     1.2 ms   1.0 ms   0.9 ms
 2  10.20.0.1       8.4 ms   8.1 ms   8.3 ms
 3  * * *
 4  203.0.113.5    14.2 ms  13.9 ms  14.5 ms
 5  93.184.216.34  24.0 ms  23.8 ms  24.3 ms
```

A line of `* * *` means that router did not answer. It may rate-limit or block ICMP, and traffic can still pass through it. It is not necessarily a failure.

**Typical starting TTL values:** 64 on Linux and macOS, 128 on Windows, 255 on many routers. Looking at the TTL in a ping reply gives a rough idea of how many hops away the host is (for example, a reply with TTL 56 from a Linux server started at 64, so it crossed about 8 routers).

### Interview traps

- "Ping works but the website does not." Ping uses ICMP. The website needs TCP port 80 or 443 and DNS. A firewall, a wrong port, a stopped service or a DNS problem can break the site while ping still works.
- "Ping fails but the website works." The server or firewall blocks ICMP. Common and not a sign of failure.
- **Blocking all ICMP breaks PMTUD.** Large packets vanish silently (a "PMTUD black hole"), and connections hang after the handshake. Always allow ICMP "fragmentation needed."
- A `* * *` in traceroute does not mean the path is broken. Look at whether later hops respond.

### Tricky questions and answers

#### Q1 [SDE-2]: Explain how traceroute discovers the routers on a path.

**Answer:** It sends probe packets with a TTL that starts at 1 and increases by one each round. Each router reduces the TTL; when it reaches 0, that router discards the packet and sends an ICMP "Time exceeded" message back to the sender. The source address of that message is the router's address. By repeating with TTL 1, 2, 3 and so on, traceroute gets one router per hop, and the round-trip times show where delay is added. The final destination answers differently (ICMP port unreachable for UDP probes, or an echo reply for ICMP probes), which tells traceroute to stop.

#### Q2 [SDE-2/3]: Small requests to a server work, but downloading a large file hangs. What could be the cause?

**Answer:** A classic **MTU or PMTUD black hole**. Small packets fit through every link, but large full-size packets are too big for some link on the path (for example a VPN or tunnel with less than 1500 bytes of MTU). The packet has DF set, so the router should drop it and send "fragmentation needed." If a firewall blocks that ICMP message, the sender never learns, keeps retrying, and the connection stalls. Fixes: allow ICMP type 3 code 4, lower the MTU on the interface or the TCP MSS (**MSS clamping**), and test with `ping -M do -s <size>` (Linux) to find the largest working size.

#### Q3 [SDE-3]: A user cannot reach a service. Walk through how you would use IP-layer tools to find the fault.

**Answer:** Work **outward** from the user. (1) `ip addr`: does the host have a correct IP, mask and (not `169.254`) address? (2) `ip route`: is there a default gateway? (3) `ping` the gateway: tests the local link and ARP. (4) `ping` an outside IP like `8.8.8.8`: tests routing and the Internet, without DNS. (5) `ping` or `dig` a name: tests DNS. (6) `traceroute` to the destination: find where it stops or where latency jumps. (7) Test the service port with `curl` or `nc`: separates network faults from application faults. Each step eliminates one layer, so you always know which layer is broken.

## Hands-On Lab: Practice Subnetting and Routing

**Part A: Subnet math (pen and paper, then check with Python)**

1. Find the network, broadcast and usable range for `172.16.45.200/20`.
2. Split `10.10.0.0/24` into subnets for 60, 25 and 10 hosts using VLSM.
3. Summarize `192.168.8.0/24` to `192.168.11.0/24` into one route.

Check your answers:

```
python3 -c "import ipaddress as i; n=i.ip_network('172.16.45.200/20',strict=False); print(n, n.network_address, n.broadcast_address)"
```

*(Answers: 1 → `172.16.32.0/20`, range `172.16.32.0`–`172.16.47.255`. 2 → `10.10.0.0/26` for 60 hosts, `10.10.0.64/27` for 25 hosts, `10.10.0.96/28` for 10 hosts. 3 → `192.168.8.0/22`.)*

**Part B: Explore your own machine**

```
#1. ip addr                      # your IP, prefix length and MAC
#2. ip route                     # find the default gateway
#3. ip neigh                     # ARP cache: do you see the gateway's MAC?
#4. ping -c 3 <gateway>          # local link
#5. ping -c 3 8.8.8.8            # routing, without DNS
#6. traceroute 8.8.8.8           # watch TTL at work
#7. ping -c 1 -M do -s 1472 8.8.8.8   # largest packet that fits a 1500 MTU
   (1472 + 8 ICMP + 20 IP = 1500. Try 1473 and watch it fail)
```

Questions to answer for yourself:

- Is your default gateway inside your own subnet? (It must be.)
- Does the TTL in the `ping` replies suggest how many hops away `8.8.8.8` is?
- What changes if you try `-s 1473`? Which header sizes explain the limit?

## Module 2 Cheat Sheet

| Concept | One-line interview answer |
|---|---|
| IPv4 | 32 bits, written as four octets. About 4.3 billion addresses. |
| Private ranges | `10/8`, `172.16/12` (172.16–172.31), `192.168/16`. Not routable on the Internet. |
| Special addresses | `127/8` loopback, `169.254/16` APIPA, `224/4` multicast, `0.0.0.0/0` default route. |
| CIDR | `address/prefix`. The prefix is the number of network bits. |
| Hosts per subnet | `2^(host bits) − 2`. The two removed are network and broadcast. /31 and /32 are special. |
| Network address | Address AND mask. Broadcast = all host bits set to 1. |
| Block size | 256 − mask value in the interesting octet. |
| VLSM | Different subnet sizes. Allocate the largest first. |
| Summarization | Combine adjacent aligned networks into one shorter prefix (`/24`s into a `/22`). |
| IPv6 | 128 bits, hex groups. `::` once, drop leading zeros. No broadcast. LAN = /64. |
| IPv6 types | `2000::/3` global, `fe80::/10` link-local, `fc00::/7` unique local, `ff00::/8` multicast. |
| DHCP | DORA: Discover, Offer, Request, Acknowledge. UDP 67/68. Gives IP, mask, gateway, DNS, lease. |
| ARP | Broadcast "who has IP?", unicast reply with MAC. Local broadcast domain only. |
| Host decision | Same subnet: send directly. Different subnet: send to default gateway. |
| Routing | Longest prefix match wins. Router knows only the next hop. |
| Routing protocols | OSPF (link-state, inside an AS), BGP (path-vector, between ASes, TCP 179), RIP (distance-vector). |
| TTL | Reduced at each hop. At 0 the packet is dropped and ICMP "time exceeded" is sent. |
| IP header | 20 bytes minimum. Protocol: 1 ICMP, 6 TCP, 17 UDP. |
| ICMP | ping = echo (8/0). traceroute uses time exceeded (11). PMTUD uses type 3 code 4. |
| Fragmentation | Only the sender in IPv6. Reassembly at the destination. DF flag with PMTUD avoids it. |

### Where this leads next

**Module 3 (TCP & UDP: the Transport Layer)** climbs one layer up. IP gets a packet to the right *host*, but it gives no promise of delivery, order or even that the packet arrives once. Module 3 shows how **ports** pick the right *program*, how TCP builds reliability on top of IP (the three-way handshake, sequence numbers, retransmission, flow control and congestion control), when UDP is the better choice, and what QUIC changes.