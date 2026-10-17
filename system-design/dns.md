---
title: "Domain Name System (DNS) — How the Internet's Phone Book Works"
description: "A complete guide to DNS: how domain name resolution works, the types of DNS servers, record types, caching and TTL, DNS-based routing and failover in system design, and how to troubleshoot with dig."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/dns.png"
tags: [dns, networking, system-design, infrastructure, domain-name]
keywords: ["How DNS works", "DNS record types", "DNS caching and TTL", "Domain Name System system design", "DNS resolver vs authoritative nameserver", "DNS failover and GeoDNS", "dig command"]
---

# Domain Name System (DNS) — How the Internet's Phone Book Works

Humans remember names. Computers communicate via IP addresses. The **Domain Name System (DNS)** is the hierarchical, decentralized naming infrastructure that bridges this gap — translating human-readable domain names like `google.com` into machine-readable IP addresses like `142.250.80.46`.

Without DNS, every user would need to memorize raw IP addresses to access any website. With DNS, the internet becomes navigable by name.

![Domain Name System](/images/dns.png)

## How DNS Resolution Works

DNS lookup is not a single step — it involves a chain of servers working together to resolve a domain name into an IP address.

![DNS Resolution Flow](/images/dns-resolution.png)

Here is the full 8-step DNS resolution process (when nothing is cached):

1. A user types `example.com` into a browser. The request goes to a **stub resolver** (the small DNS client built into the operating system), which sends the query to a configured **DNS Resolver** (typically provided by the ISP, or a public one such as `8.8.8.8` for Google DNS).
2. The DNS Resolver queries a **DNS Root Nameserver**, which has no specific domain information but knows where to route the query next.
3. The Root Nameserver responds with the address of the appropriate **TLD (Top-Level Domain) Nameserver** based on the domain extension (`.com`, `.org`, `.io`, etc.).
4. The DNS Resolver queries the `.com` **TLD Nameserver**.
5. The TLD Nameserver responds with the address of the **Authoritative Nameserver** for `example.com`.
6. The DNS Resolver queries the **Authoritative Nameserver** for `example.com`.
7. The Authoritative Nameserver returns the **IP address** (A record) for `example.com`.
8. The DNS Resolver returns the IP address to the browser (and caches it). The browser can now make an HTTP request to that IP.

An uncached lookup typically takes tens of milliseconds up to a couple of hundred, and subsequent lookups are dramatically faster due to caching. In practice the resolver also caches the root and TLD referrals, so **root servers are rarely contacted** for popular domains.

**Protocol detail:** DNS normally uses **UDP on port 53** for speed, and falls back to **TCP** when a response is too large or for zone transfers. Encrypted variants (DNS over HTTPS, DNS over TLS) protect queries from eavesdropping on the network.

## The Four Types of DNS Servers

### 1. DNS Resolver (Recursive Resolver)

The **first stop** in a DNS query. It acts as an intermediary between the client and the DNS namespace. Upon receiving a query, it either serves a cached response or begins the resolution process by querying root, TLD, and authoritative nameservers.

Commonly operated by ISPs, or available as public resolvers (Google: `8.8.8.8`, Cloudflare: `1.1.1.1`).

### 2. DNS Root Nameserver

There are **13 root server addresses** (named A through M), but each is replicated using Anycast routing across well over a thousand server instances worldwide. They don't know specific domain IP addresses — they direct resolvers to the appropriate TLD nameserver.

Operated by a dozen independent organizations (for example Verisign, ICANN, universities, and research and government bodies), not by one company.

### 3. TLD Nameserver

Maintains information for all domains sharing a common extension. There are two main categories:

- **Generic TLDs (gTLD):** `.com`, `.org`, `.net`, `.edu`, `.gov`
- **Country Code TLDs (ccTLD):** `.uk`, `.us`, `.in`, `.jp`, `.de`

Each TLD is run by a **registry operator** (for example, Verisign operates `.com` and `.net`). [IANA](https://www.iana.org), a function operated under [ICANN](https://www.icann.org), maintains the **root zone** — the list saying which registry is responsible for each TLD — but it does not run the TLD servers themselves.

### 4. Authoritative DNS Server

The **final authority** for a specific domain. It holds the actual DNS records for `example.com` and returns the definitive answer. If the domain doesn't exist, it returns an `NXDOMAIN` response. You usually pick this when you choose a DNS host (Route 53, Cloudflare, and so on).

## DNS Query Types

| Query Type        | Description                                                                                                                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Recursive**     | The client asks the resolver to return a complete answer or an error. The resolver does all the legwork. This is what your device sends to its resolver.                                  |
| **Iterative**     | The server returns the best answer it has — possibly a referral to another nameserver — and the asker follows up. This is how the resolver talks to root, TLD, and authoritative servers. |
| **Non-Recursive** | The server answers directly from data it already has (its own zone or its cache) without contacting any other server.                                                                     |

## DNS Record Types

DNS records (zone files) store specific information about a domain. Each record has a **TTL (Time to Live)** value that controls how long it can be cached.

| Record Type | Full Name                             | Purpose                                                                                                                      |
| ----------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **A**       | Address                               | Maps domain → IPv4 address                                                                                                   |
| **AAAA**    | IPv6 Address                          | Maps domain → IPv6 address                                                                                                   |
| **CNAME**   | Canonical Name                        | Makes one name an **alias** for another name; the resolver then looks up the target. It is a DNS alias, not an HTTP redirect |
| **MX**      | Mail Exchanger                        | Directs email to mail servers (with priority)                                                                                |
| **TXT**     | Text Record                           | Stores arbitrary text; used for SPF, DKIM, DMARC, domain verification                                                        |
| **NS**      | Name Server                           | Specifies authoritative nameservers for the domain                                                                           |
| **SOA**     | Start of Authority                    | Contains admin metadata (primary nameserver, admin email, serial number, timers)                                             |
| **SRV**     | Service Location                      | Specifies hostname and port for specific services                                                                            |
| **PTR**     | Reverse Pointer                       | Maps IP address → domain (reverse DNS lookup)                                                                                |
| **CAA**     | Certification Authority Authorization | Lists which certificate authorities may issue TLS certificates for the domain                                                |

**Example DNS Records for `example.com` (illustrative values):**

```
example.com.     300    IN    A       93.184.216.34
example.com.     300    IN    AAAA    2606:2800:220:1:248:1893:25c8:1946
www.example.com. 300    IN    CNAME   example.com.
example.com.     3600   IN    MX      10 mail.example.com.
example.com.     3600   IN    TXT     "v=spf1 include:_spf.google.com ~all"
```

### CNAME Rules Worth Knowing

- A name with a CNAME **cannot have any other record type** at the same name.
- You **cannot put a CNAME at the zone apex** (`example.com` itself), because the apex must hold SOA and NS records. Many DNS providers offer a workaround (called ALIAS, ANAME, or an "alias record" in Route 53) that behaves like a CNAME at the apex.
- CNAMEs add an extra lookup, so long chains slow resolution.

## DNS Caching

Caching is what makes DNS practical at internet scale. Without caching, every single web request would require several round trips across the globe.

**Where DNS is cached:**

1. **Browser cache** — Modern browsers cache DNS for seconds to minutes.
2. **Operating system cache** — The OS maintains its own DNS resolver cache.
3. **Recursive resolver cache** — The ISP's or public resolver caches responses for the duration of the record's TTL, shared by all its users.
4. **Application/runtime caches** — Some runtimes and libraries (for example, JVM settings) cache lookups on their own, sometimes longer than the TTL.

**TTL (Time to Live):** Every DNS record specifies how long it can be cached (in seconds). When the TTL expires, the cache discards the record and a fresh lookup is performed.

**Negative caching:** "this name doesn't exist" (`NXDOMAIN`) answers are also cached, for a period set by the zone's SOA record. So creating a record right after someone looked it up can appear not to work for a while.

**System Design Consideration:**

- **Short TTL (60–300s):** Use when you need quick changes or failover. Increases load on your DNS servers and adds lookup latency.
- **Long TTL (3600–86400s):** Use for stable endpoints to reduce DNS query load. Changes take longer to take effect.
- **Lower the TTL ahead of a planned migration** (at least one old-TTL period before), make the change, then raise it again.

> DNS does not "propagate" in the sense of pushing updates around the world. A change simply becomes visible as each cache's old copy expires. That's why different users see the change at different times.

Even with a low TTL, some resolvers, clients, and runtimes hold on to old answers, so DNS failover is never perfectly instant.

## Subdomains

A subdomain is an additional part of the main domain, logically separating a website into distinct sections:

```
blog.example.com      → Company blog
api.example.com       → REST API endpoint
admin.example.com     → Internal admin panel
cdn.example.com       → Static assets
```

Each subdomain can have independent DNS records pointing to entirely different servers or services.

## DNS Load Balancing

DNS can perform rudimentary load balancing by returning multiple A records for a single domain (Round-Robin DNS). When a client queries `example.com`, the resolver returns multiple IP addresses in rotating order.

**Limitations of plain round-robin DNS:**

- It has **no health checking** of its own — it keeps returning the same IPs even if a server is down.
- Caching means clients may keep using a failed server until the TTL expires, and distribution is uneven because resolvers cache answers for many users.
- No support for session affinity (sticky sessions) or load awareness.

Managed DNS services improve on this with **health checks and routing policies** (next section), but they are still bound by caching. For production systems, DNS typically points at a proper load balancer (AWS ELB, NGINX, HAProxy), which does fast, health-aware balancing. See [Load Balancing](system-design/load-balancing).

## DNS in System Design

DNS is often the **first** layer of traffic management for a large system.

### Routing Policies

Managed DNS (such as Route 53) chooses the answer based on a policy:

| Policy            | How it chooses                                                | Typical use                                                                  |
| ----------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Simple**        | Returns the record(s) as configured                           | Single endpoint                                                              |
| **Weighted**      | Splits traffic by percentage                                  | Gradual rollouts, canary releases, A/B tests                                 |
| **Latency-based** | Returns the region with the lowest latency to the user        | Multi-region applications                                                    |
| **Geolocation**   | Based on the user's country or region                         | Data residency, localized content                                            |
| **Failover**      | Primary while healthy, then a standby when health checks fail | Disaster recovery (see [High Availability](system-design/high-availability)) |

Health checks run from the DNS provider's side and remove unhealthy endpoints from answers, subject to TTL and caching delays.

### Other Common Uses

- **CDN entry point:** `cdn.example.com` is a CNAME to the CDN's hostname, and the CDN's DNS steers each user to a nearby edge server. See [Caching and CDN](system-design/caching-and-cdn).
- **Service discovery inside a cluster:** platforms such as Kubernetes give each service a DNS name (for example `payments.shop.svc.cluster.local`) that resolves to the service's stable address.
- **Email authentication:** MX, SPF, DKIM, and DMARC records are all stored in DNS.
- **Domain ownership and certificates:** TXT records prove ownership; CAA records restrict who can issue certificates.

### DNS as a Dependency

If DNS fails, users can't reach you even when every server is healthy. A well-known example is the 2016 attack on the DNS provider Dyn, which made many popular sites unreachable for hours. To reduce the risk, use a highly available managed DNS (Anycast-based) and consider **multiple DNS providers** for critical domains.

## Reverse DNS

Reverse DNS does the opposite of forward DNS — it resolves an **IP address back to a domain name**, using PTR records. Commonly used by:

- **Email servers** — To check that a sending server's IP corresponds to its claimed domain (an anti-spam measure).
- **Security logging** — To identify the hostname associated with a suspicious IP address.
- **Network diagnostics** — Tools like `traceroute` use reverse DNS to display hostnames.

## Troubleshooting DNS

```bash
dig example.com                  # query the default resolver
dig example.com A +short         # just the answer
dig @8.8.8.8 example.com         # ask a specific resolver
dig example.com NS               # find the authoritative nameservers
dig +trace example.com           # walk the chain: root → TLD → authoritative
nslookup example.com             # simpler alternative
```

Useful checks:

- Compare the answer from your resolver with one from the **authoritative** server. A difference usually means a cache hasn't expired yet.
- Look at the **TTL** in the answer to see how long the old value may linger.
- `NXDOMAIN` means the name doesn't exist; `SERVFAIL` means the resolver couldn't get a valid answer (often a broken authoritative server or DNSSEC problem).

## Widely Used DNS Solutions

| Provider                                              | Notes                                                                          |
| ----------------------------------------------------- | ------------------------------------------------------------------------------ |
| [Amazon Route 53](https://aws.amazon.com/route53)     | AWS-native, health checks, routing policies (weighted, latency, geo, failover) |
| [Cloudflare DNS](https://www.cloudflare.com/dns)      | 1.1.1.1 public resolver; authoritative DNS with DDoS protection                |
| [Google Cloud DNS](https://cloud.google.com/dns)      | Anycast-based, 100% availability SLA                                           |
| [Azure DNS](https://azure.microsoft.com/services/dns) | Integrated with Azure resource management                                      |
| [NS1](https://ns1.com)                                | Traffic management and intelligent routing                                     |

## System Design Takeaways

- DNS is a **critical dependency** — outages can take down entire services even when servers are healthy. Use redundant, Anycast-based DNS, and consider multiple providers for critical domains.
- Use **short TTLs ahead of deployments or failovers** so changes take effect quickly, then raise them again. Remember that DNS failover is never instant because of caching.
- **DNS-based routing** (weighted, latency, geolocation) is a simple, cost-effective way to direct users to the nearest regional deployment, and to roll out changes gradually.
- Use a **load balancer** for fast, health-aware balancing; use DNS to choose the region or the balancer.
- **DNSSEC** adds cryptographic signatures to DNS records so resolvers can verify they weren't tampered with (protecting against spoofing and cache poisoning). It doesn't encrypt queries; DNS over HTTPS/TLS does that.
- Monitor DNS records and resolution time as part of your infrastructure observability.

## Interview Questions

**Q: What happens when you type a URL into a browser, as far as DNS is concerned?**
The browser and OS check their caches; if there's no answer, the stub resolver asks the recursive resolver, which (if not cached) walks root → TLD → authoritative nameserver and returns the IP, caching it for the TTL. Then the browser connects to that IP.

**Q: What's the difference between a recursive resolver and an authoritative server?**
The recursive resolver does the lookup work on behalf of clients and caches results. The authoritative server holds the actual records for a domain and gives the definitive answer.

**Q: How does TTL affect failover and deployments?**
Caches keep an answer until its TTL expires, so a long TTL means slow changes and slow failover, and a short TTL means faster change at the cost of more DNS queries. Lower the TTL before planned changes.

**Q: Why can't you use a CNAME at the root of a domain?**
The zone apex must have SOA and NS records, and a CNAME can't coexist with other records at the same name. Providers offer ALIAS/ANAME or alias records as a workaround.

**Q: Can DNS alone provide load balancing and high availability?**
Only roughly. Round-robin DNS spreads clients across IPs but, on its own, doesn't check health, and caching delays failover. Managed DNS adds health checks and policies, but a load balancer is better for fast, precise balancing.

**Q: How would you route users to the nearest region?**
Use latency-based or geolocation routing in a managed DNS (or Anycast), with health checks and failover so a failed region is removed from answers.

**Q: What does DNSSEC protect against?**
Forged DNS answers and cache poisoning, by signing records so they can be verified. It provides authenticity, not privacy.

## Summary

- DNS translates names to IPs through a hierarchy: stub resolver → recursive resolver → root → TLD → authoritative server.
- Caching with TTL makes DNS fast, and also means changes and failover are never instant.
- Know the record types, especially A, AAAA, CNAME (and its apex limitation), MX, TXT, NS, and SOA.
- In system design, DNS provides routing policies (weighted, latency, geo, failover), CDN entry, and service discovery, but it is also a critical dependency.
- Use `dig` (including `+trace`) to troubleshoot, and compare answers from resolvers and authoritative servers.

## Continue Learning

- [Load Balancing](system-design/load-balancing) — fast, health-aware balancing behind DNS.
- [Caching and CDN](system-design/caching-and-cdn) — how CDNs use DNS to steer users to the edge.
- [High Availability](system-design/high-availability) — DNS failover and multi-region design.
- [Networking Fundamentals](system-design/networking-fundamentals) — the layers DNS runs on.
