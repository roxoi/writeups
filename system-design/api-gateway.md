---
title: "API Gateway"
description: "What an API gateway does, how it differs from a load balancer and reverse proxy, the BFF pattern, resilience and security concerns, common pitfalls, and popular gateway solutions."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/microservices.png"
tags: [api-gateway, microservices, architecture, bff, security, system-design]
keywords: ["API Gateway", "API Gateway design", "Backend for Frontend", "API Gateway vs load balancer", "Kong", "Microservices entry point"]
---

# API Gateway

An API gateway is a **single entry point** for client requests to a system made of many backend services. It receives every request, applies common policies, and routes it to the right service.

## Why You Need One

Without a gateway, every client talks to every service directly. That causes problems:

- Clients must know the address and API of many services, and update when they change.
- Every service has to re-implement authentication, rate limiting, logging, and TLS.
- Internal services are exposed to the outside world.
- A mobile app might need five calls to render one screen.

A gateway moves these **cross-cutting concerns** into one place and hides the internal structure from clients.

```text
                       ┌──► User Service
Mobile ──┐             │
Web    ───┼──► API Gateway ──► Order Service
Partner ──┘             │
                       └──► Payment Service
```

## Responsibilities

- **Request routing:** forward requests to the correct service based on path, host, headers, or version (`/orders/*` goes to the Order Service).
- **Authentication and authorization:** validate tokens or API keys before forwarding (see [OAuth & JWT](system-design/auth-and-security)).
- **[Rate limiting](system-design/rate-limiting):** prevent abuse and protect backends.
- **SSL/TLS termination:** handle HTTPS at the edge. Internal traffic should still be encrypted where possible (for example with mTLS) rather than assumed safe.
- **Request/response transformation:** adapt between client-facing and backend formats.
- **Protocol translation:** expose REST to clients while talking [gRPC](system-design/messaging-and-communication) internally.
- **Aggregation (API composition):** combine responses from several services into one, reducing client round trips.
- **Caching:** cache responses for frequently requested data.
- **Load balancing:** spread requests across instances of a service.
- **Observability:** centralized access logs, metrics, and trace-ID injection.
- **Resilience:** timeouts, retries, and [circuit breakers](system-design/circuit-breaker) toward backends.

## How a Request Flows

1. Client sends a request to the gateway's public address (often via DNS, a CDN, and a load balancer).
2. Gateway terminates TLS and checks the **authentication** token.
3. It applies **rate limits** and any request validation.
4. It **routes** to a backend service (via service discovery or static config).
5. It applies timeouts and retry policy, and may aggregate several backend calls.
6. It transforms and returns the response, logging metrics and a trace ID.

## API Gateway vs Load Balancer vs Reverse Proxy

These overlap and are often confused:

|                   | Main job                                                                                                              | Understands                   |
| ----------------- | --------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| **Load balancer** | Spread traffic across **instances of the same service**                                                               | Connections (L4) or HTTP (L7) |
| **Reverse proxy** | Sit in front of servers, forward client requests (TLS, caching, compression)                                          | HTTP generally                |
| **API gateway**   | A reverse proxy **plus API-aware features**: auth, rate limiting, routing by API version, aggregation, transformation | APIs, clients, and policies   |

In practice they are layered or combined. A typical setup has a cloud load balancer in front of several gateway instances, and tools like NGINX or Envoy can play all three roles depending on configuration. See [Load Balancing](system-design/load-balancing) and Proxies.

### North-South vs East-West Traffic

- **North-south:** traffic between external clients and your system. This is the gateway's job.
- **East-west:** traffic between your internal services. That is usually handled by a [service mesh](system-design/messaging-and-communication) or direct client libraries, not the gateway.

## Backend for Frontend (BFF) Pattern

A single gateway can end up serving very different clients badly: the web app wants rich data, the mobile app wants small payloads, a partner API wants a stable contract.

The **BFF pattern** creates a **separate gateway tailored to each client type**:

- `web-bff.example.com`: optimized API for the web app
- `mobile-bff.example.com`: smaller payloads, fewer round trips for mobile
- `partner-api.example.com`: stable, versioned, strictly rate-limited

Each BFF is owned by the team that owns that client, so they can change it without coordinating with everyone. The cost is more components to run and some duplicated logic.

## Security Considerations

- Validate identity at the gateway (for example JWT signature, expiry, and audience), then pass trusted identity information to services.
- Do not assume everything behind the gateway is trusted. Services should still authorize requests for their own data and use mutual TLS between services (defense in depth).
- Strip or overwrite internal headers coming from clients so they cannot spoof identity headers.
- Combine with a web application firewall (WAF) and DDoS protection at the edge. See [Security Best Practices](system-design/auth-and-security).

## Pitfalls

| Pitfall                     | Why it hurts                                                    | Mitigation                                                                      |
| --------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Single point of failure** | Every request goes through it                                   | Run multiple instances across zones behind a load balancer                      |
| **Performance bottleneck**  | Adds a network hop and processing to every request              | Keep it lightweight, scale horizontally, cache carefully                        |
| **Business logic creep**    | The gateway becomes a hidden monolith that everyone must change | Keep it to cross-cutting concerns; business logic belongs in services           |
| **Team bottleneck**         | One team owns the gateway and everyone waits on them            | Declarative config, self-service route changes, or per-domain gateways          |
| **Over-aggregation**        | Large fan-out calls make it slow and fragile                    | Aggregate only where it clearly helps; set timeouts and allow partial responses |

## API Gateway Solutions

| Solution                                                                    | Notes                                                                           |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| [Amazon API Gateway](https://aws.amazon.com/api-gateway)                    | Fully managed; integrates with Lambda, ECS, EC2                                 |
| [Kong](https://konghq.com)                                                  | Open-source; large plugin ecosystem; high performance                           |
| [Apigee](https://cloud.google.com/apigee)                                   | Enterprise API management (Google Cloud)                                        |
| [Azure API Management](https://azure.microsoft.com/services/api-management) | Microsoft Azure's API gateway                                                   |
| [NGINX](https://www.nginx.com)                                              | Can be configured as an API gateway                                             |
| [Envoy](https://www.envoyproxy.io)                                          | High-performance proxy, also the data plane of many service meshes and gateways |
| [Traefik](https://traefik.io)                                               | Dynamic configuration and service-discovery friendly                            |

Managed gateways reduce operational work; self-hosted ones give more control and avoid lock-in.

## Trade-offs

| Choice                   | You gain                                                 | You give up                                  |
| ------------------------ | -------------------------------------------------------- | -------------------------------------------- |
| Use a gateway            | One place for auth, limits, logging; clients are simpler | An extra hop and a critical component to run |
| Aggregate in the gateway | Fewer client round trips                                 | Gateway complexity and coupling              |
| BFF per client           | Tailored APIs, team autonomy                             | More components, duplicated logic            |
| Managed gateway          | Less operational burden                                  | Cost and vendor lock-in                      |

## Real-World Usage

- A microservices backend exposing one public API domain, with the gateway routing `/users`, `/orders`, and `/payments` to different services.
- A mobile app whose home screen needs data from four services: the mobile BFF aggregates them into one response.
- A public developer API where the gateway issues API keys, enforces per-key quotas, and versions routes (`/v1`, `/v2`).

## Interview Questions

**Q: What does an API gateway do?**
It is the single entry point for clients. It routes requests to backend services and handles cross-cutting concerns such as authentication, rate limiting, TLS termination, logging, and sometimes aggregation and transformation.

**Q: How is an API gateway different from a load balancer?**
A load balancer distributes traffic across instances of the same service. A gateway is API-aware: it routes between _different_ services and applies policies such as auth and rate limits. They are often used together.

**Q: What is the BFF pattern?**
Backend for Frontend: a separate gateway or backend per client type (web, mobile, partner) so each gets an API shaped for its needs.

**Q: What are the risks of an API gateway?**
It can become a single point of failure and a performance bottleneck, and it can accumulate business logic. Run it redundantly, keep it thin, and scale it horizontally.

**Q: Where should authentication happen?**
Typically validated at the gateway, but services should still authorize access to their own data and not blindly trust internal calls.

## Summary

- An API gateway is the single entry point that centralizes routing and cross-cutting concerns.
- It hides internal services from clients and reduces client complexity.
- It is different from a load balancer (same-service instances) and a plain reverse proxy (no API-level policy).
- Use the BFF pattern when different clients need different APIs.
- Keep it thin, redundant, and horizontally scalable to avoid SPOF, bottleneck, and logic creep.

## Continue Learning

- [Rate Limiting](system-design/rate-limiting) — a core gateway responsibility.
- [Microservices Architecture](system-design/microservices) — the architecture gateways front.
- [Messaging and Communication](system-design/messaging-and-communication) — service discovery, gRPC, and communication between services.
- [Authentication and Security](system-design/auth-and-security) — what the gateway validates.
- [Load Balancing](system-design/load-balancing) — how reverse proxies work.
