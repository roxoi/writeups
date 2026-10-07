---
title: "Introduction to Computer Networking"
description: "A beginner-friendly hub for learning computer networking - start with packets and layers, move on to IP, TCP, DNS and HTTP, and build a full picture of how data travels across the internet."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/intro-networking.png"
tags: [Networking, TCP-IP, DNS, HTTP, Interview-Prep]
keywords: ["Introduction to Computer Networking", "Networking tutorial for beginners", "OSI and TCP/IP model explained", "Networking for system design interviews"]
---

# Introduction to Computer Networking

![Networking Fundamentals](/images/intro-networking.png)

The tutorial is written in simple language. You can start with no background. Each topic begins with a plain-words idea and then goes step by step into how things really work, ending with the traps and questions that appear in MAANG interviews.


## The tutorial at a glance

| #   | Module                                      | What it answers                                                                                                    | Read more                                                                            |
| --- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| 1   | **Basics & the Layered Models**             | What is a network, how do packets travel, what are the OSI and TCP/IP layers, and what do switches and routers do? | [Open Module 1](computer-networks/basics-and-the-layered-models)                     |
| 2   | **IP Addressing, Subnetting & Routing**     | How are devices addressed, how does subnet math work, and how does a packet find its way?                          | [Open Module 2](computer-networks/addressing-subnetting-and-routing)                 |
| 3   | **TCP & UDP: the Transport Layer**          | How does reliable delivery work, and when do you pick UDP instead?                                                 | [Open Module 3](computer-networks/tcp-and-udp-the-transport-layer)                   |
| 4   | **DNS, HTTP & TLS: the Application Layer**  | How are names resolved, how does the web talk, and how is it kept secure?                                          | [Open Module 4](computer-networks/dns-http-and-tls-the-application-layer)            |
| 5   | **Firewalls, NAT, VPNs & Load Balancing**   | How is traffic protected, shared and spread across servers?                                                        | [Open Module 5](computer-networks/firewalls-nat-vpn-and-load-balancing)              |
| 6   | **Networking for System Design Interviews** | What happens when you type a URL, how do you troubleshoot, and which numbers should you remember?                  | [Open Module 6](computer-networks/networking-for-system-design-interviews)           |

## Networking in one minute (basic definitions)

| Term             | Simple meaning                                                                                  | Example                                       |
| ---------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **Network**      | Two or more devices connected so they can share data.                                           | Your phone and laptop on home Wi-Fi.          |
| **Host / node**  | Any device on a network.                                                                        | A laptop, a server, a printer.                |
| **Protocol**     | An agreed set of rules for talking.                                                             | HTTP, TCP, IP.                                |
| **Packet**       | A small chunk of data with a header (address and control info) and a payload (the actual data). | One piece of a web page.                      |
| **IP address**   | A logical address that says _which network and which host_.                                     | `192.168.1.10`                                |
| **MAC address**  | A hardware address of a network card, used on the local link.                                   | `00:1A:2B:3C:4D:5E`                           |
| **Port**         | A number that says _which program_ on a host should receive the data.                           | `443` for HTTPS                               |
| **Router**       | A device that forwards packets _between_ networks using IP addresses.                           | Your home router.                             |
| **Switch**       | A device that forwards frames _inside_ one network using MAC addresses.                         | The box in an office wiring closet.           |
| **DNS**          | The system that turns names into IP addresses.                                                  | `example.com` → `93.184.216.34`               |
| **TCP / UDP**    | The two main transport protocols. TCP is reliable. UDP is fast and light.                       | Web pages use TCP. Video calls often use UDP. |
| **HTTP / HTTPS** | The language of the web. HTTPS is HTTP protected by TLS encryption.                             | Every website you open.                       |
| **Bandwidth**    | How much data a link can carry per second.                                                      | 100 Mbps.                                     |
| **Latency**      | How long one piece of data takes to arrive.                                                     | 30 ms.                                        |

## The big picture: the layers

Every piece of data you send passes down a stack of layers on your device, travels across the network, and climbs back up the stack on the other side. Each layer has one job.

```
  Application   HTTP, DNS, SMTP        "what are we saying?"
  Transport     TCP, UDP               "which program, and reliably?"
  Internet      IP, ICMP               "which network and which host?"
  Link          Ethernet, Wi-Fi        "how do we reach the next device?"
```

[Module 1](computer-networks/basics-and-the-layered-models) explains this stack in detail. Modules 2–5 then take one layer at a time.

## Suggested reading paths

**If you are new:** read Modules 1 → 6 in order. Read the "Words you will meet" table at the top of each module first.

**If you already know the basics:** skim each module's plain-words opening and spend your time on **Interview traps** and **Tricky questions and answers**. Use the cheat sheet at the end of each module for a final review.

**If an interview is close:** [Module 6](computer-networks/networking-for-system-design-interviews) first, then [Module 3](computer-networks/tcp-and-udp-the-transport-layer) and [Module 4](computer-networks/dns-http-and-tls-the-application-layer) (TCP, DNS and HTTP are asked most), then the cheat sheets.

**If you only want a short system-design summary:** read [Networking Fundamentals](system-design/networking-fundamentals).

## Quick finder: "Where do I read about...?"

| If you want to understand...                   | Go to                                                                                                                        |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| OSI vs TCP/IP model, encapsulation             | [Module 1, Topic 1.3](computer-networks/basics-and-the-layered-models#13-the-osi-and-tcpip-layered-models)                   |
| Bandwidth, latency, throughput, delay formulas | [Module 1, Topic 1.2](computer-networks/basics-and-the-layered-models#12-packets-switching--network-performance)             |
| Hub vs switch vs router                        | [Module 1, Topic 1.4](computer-networks/basics-and-the-layered-models#14-network-devices-hub-switch-router-and-friends)      |
| MAC addresses and Ethernet frames              | [Module 1, Topic 1.5](computer-networks/basics-and-the-layered-models#15-ethernet-mac-addresses-and-frames)                  |
| Subnetting, CIDR, private IP ranges            | [Module 2](computer-networks/addressing-subnetting-and-routing)                                                              |
| Routing tables, default gateway, ARP, DHCP     | [Module 2](computer-networks/addressing-subnetting-and-routing)                                                              |
| TCP handshake, congestion control, UDP         | [Module 3](computer-networks/tcp-and-udp-the-transport-layer)                                                                |
| DNS resolution, HTTP versions, TLS             | [Module 4](computer-networks/dns-http-and-tls-the-application-layer)                                                         |
| NAT, firewalls, VPN, load balancers, CDN       | [Module 5](computer-networks/firewalls-nat-vpn-and-load-balancing)                                                           |
| "What happens when I type a URL?"              | [Module 6](computer-networks/networking-for-system-design-interviews)                                                        |

## Tools you will use (Linux / macOS, with Windows names)

| Tool                                           | What it does                                      |
| ---------------------------------------------- | ------------------------------------------------- |
| `ping`                                         | Checks reachability and measures round-trip time. |
| `traceroute` (`tracert` on Windows)            | Shows the routers between you and a destination.  |
| `ip addr` / `ifconfig` (`ipconfig` on Windows) | Shows your IP and MAC addresses.                  |
| `ip route` / `netstat -rn`                     | Shows the routing table.                          |
| `ip neigh` / `arp -a`                          | Shows the ARP table (IP to MAC).                  |
| `dig` / `nslookup`                             | Asks DNS questions.                               |
| `curl -v`                                      | Makes an HTTP request and shows the details.      |
| `ss -tulpn` / `netstat -an`                    | Shows open ports and connections.                 |
| `tcpdump` / Wireshark                          | Captures and shows raw packets.                   |