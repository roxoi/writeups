---
title: "Circuit Breaker Pattern"
description: "How circuit breakers prevent cascading failures: the three states, configuration, fallbacks, combining with timeouts, retries, and bulkheads, common pitfalls, and popular libraries."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/microservices.png"
tags: [circuit-breaker, resilience, fault-tolerance, microservices, reliability, system-design]
keywords: ["Circuit breaker pattern", "Cascading failure", "Resilience patterns", "Half-open state", "Resilience4j", "Bulkhead and retry"]
---

# Circuit Breaker Pattern

The circuit breaker prevents **cascading failures** by **stopping calls to a failing service** and giving it time to recover. It is named after the electrical device that cuts power when current is dangerous, protecting the rest of the circuit.

## The Problem

Imagine Service A calls Service B, and B becomes slow or starts failing.

1. A's requests to B hang until they time out.
2. While waiting, A's threads and connections are all tied up.
3. A runs out of resources and becomes slow or unresponsive.
4. Everything that calls A is now affected, and the failure spreads up the chain.

Two things make this worse:

- **A slow dependency is often more dangerous than a dead one.** A dead service fails fast; a slow one holds your resources hostage.
- **Retries amplify the damage.** If a struggling service gets _more_ traffic from retries, it cannot recover. If three layers each make 3 attempts, the bottom service can see up to 27 calls for a single user request (a **retry storm**).

The circuit breaker fixes this by failing fast once a dependency is known to be unhealthy.

## How It Works

### States

| State         | Description                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Closed**    | Normal operation. Requests flow through and failures are counted.                                                                                 |
| **Open**      | The failure threshold was exceeded. All requests **immediately fail** (or use a fallback) without calling the service.                            |
| **Half-Open** | After a wait time, a **limited number of trial requests** are allowed through. If they succeed, the circuit closes; if they fail, it opens again. |

```text
             failures exceed threshold
   CLOSED ───────────────────────────────► OPEN
      ▲                                      │
      │ trial calls succeed                  │ wait time elapses
      │                                      ▼
      └───────────────────────────────── HALF-OPEN
                  trial calls fail ──► back to OPEN
```

While open, the failing service gets breathing room, and callers get an instant answer instead of a long wait.

## Configuration

Typical settings (names vary by library):

| Setting                       | Meaning                                                                     | Considerations                                                               |
| ----------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Failure threshold**         | When to open: a failure _rate_ (for example 50%) or a count within a window | Rate-based is usually better than a raw count because it scales with traffic |
| **Sliding window**            | The recent calls or time period used to measure failures                    | Too small is jumpy; too large reacts slowly                                  |
| **Minimum number of calls**   | Do not evaluate until at least this many calls occurred                     | Avoids opening on 1 failure out of 2 calls                                   |
| **Open duration (wait time)** | How long to stay open before trying half-open                               | Long enough for recovery, short enough to restore service quickly            |
| **Half-open permitted calls** | How many trial calls to allow                                               | A small number so a recovering service is not flooded                        |
| **Slow-call threshold**       | Treat calls slower than X as failures                                       | Catches the "slow, not dead" problem                                         |

### What Counts as a Failure?

- **Count:** timeouts, connection errors, and server errors (5xx).
- **Usually do not count:** client errors (4xx), which mean the request was bad, not that the service is unhealthy.
- **Consider:** calls that succeed but are far too slow.

## Fallbacks: What To Do When It's Open

An open circuit should lead to a **graceful degradation**, not just an error:

- **Cached data:** return the last known good value (product details, recommendations).
- **Default or empty response:** show a generic list instead of personalized recommendations.
- **Alternative service:** route to a backup provider.
- **Queue for later:** accept the request and process it when the dependency recovers (see [Message Queues](system-design/messaging-and-communication)).
- **Fail fast with a clear error** when no safe fallback exists, such as a payment authorization. Never fake a success.

Decide per dependency whether it is **critical** (no fallback exists) or **non-critical** (degrade quietly).

## Working with Other Resilience Patterns

The circuit breaker is one tool of several. They work best together:

| Pattern                                          | Role                                                                                                                                               |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Timeout**                                      | Limits how long any single call waits. Without timeouts, a breaker never sees the failure.                                                         |
| **Retry with backoff and jitter**                | Recovers from brief, transient errors. Retry only idempotent operations. See [Idempotency and Retries](system-design/messaging-and-communication). |
| **Circuit breaker**                              | Stops repeated calls when a dependency is persistently unhealthy.                                                                                  |
| **Bulkhead**                                     | Isolates resources (thread or connection pools) per dependency so one slow dependency cannot use them all.                                         |
| **[Rate limiting](system-design/rate-limiting)** | Protects a service from too much incoming load.                                                                                                    |
| **Fallback**                                     | Provides a degraded result when the call fails or the circuit is open.                                                                             |

Guidelines for combining them:

- Give **every attempt a timeout**, and let the breaker count each failed attempt.
- **Do not retry when the circuit is open.** Retrying a call that is being deliberately rejected defeats the purpose.
- Keep retries few, and use exponential backoff with jitter so clients do not retry in lock-step.
- Put a **separate breaker per dependency** (and often per endpoint), not one shared breaker for everything.

See [Fault Tolerance](system-design/high-availability) for the bigger picture.

## Where to Place It

- **Client side**, around outbound calls to each dependency. This is the classic placement.
- In a **service mesh** or sidecar proxy, which can eject unhealthy instances (often called outlier detection) without changing application code. See [Service Discovery and Service Mesh](system-design/messaging-and-communication).
- In the **[API gateway](system-design/api-gateway)** for calls to backend services.

## Libraries and Tools

| Tool                   | Language / platform                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------ |
| Resilience4j           | Java                                                                                 |
| Polly                  | .NET                                                                                 |
| opossum                | Node.js                                                                              |
| gobreaker              | Go                                                                                   |
| Envoy / service meshes | Proxy level, language-independent                                                    |
| Hystrix                | Java, the pioneer of the pattern; now in maintenance mode, so prefer newer libraries |

## Pitfalls

| Pitfall                         | Problem                                                            | Fix                                                            |
| ------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------- |
| **No timeouts**                 | Slow calls never register as failures                              | Always set timeouts below the caller's own deadline            |
| **Bad thresholds**              | Opens on noise, or never opens                                     | Use a rate with a minimum call count; tune with real traffic   |
| **One breaker for everything**  | One bad endpoint blocks healthy ones                               | One breaker per dependency or endpoint                         |
| **Thundering herd on recovery** | Closing the circuit sends all traffic at once to a fragile service | Allow only a few half-open trial calls, ramp up gradually      |
| **Per-instance state**          | Each instance learns separately, so detection is slower            | Usually acceptable; shared state adds latency and a dependency |
| **Masking real problems**       | Fallbacks hide an outage nobody notices                            | Emit metrics and alerts whenever the state changes             |
| **Counting 4xx as failures**    | Bad client requests open the circuit                               | Count only errors that indicate dependency health              |
| **Unsafe fallbacks**            | Returning fake or stale data for critical operations               | Fail fast for critical operations                              |

Also **test it**: inject failures and latency in a test or staging environment (chaos testing) to confirm the breaker opens, fallbacks work, and recovery behaves as expected.

## Observability

Treat breaker state as a first-class signal:

- Track **state changes** (closed to open, open to half-open) and alert on them.
- Track the **number of rejected calls**, **failure rate**, and **slow-call rate** per dependency.
- Dashboards should show which dependencies are currently open. See [Observability](system-design/observability-and-deployment).

## Trade-offs

| Choice              | You gain               | You give up                         |
| ------------------- | ---------------------- | ----------------------------------- |
| Open quickly        | Fast protection        | More false positives                |
| Open slowly         | Fewer false positives  | More damage before it trips         |
| Short open duration | Fast recovery          | Risk of hammering a fragile service |
| Fallbacks           | Better user experience | Possibly stale or reduced data      |

## Real-World Usage

- An **e-commerce product page** calls a recommendation service. When it fails, the breaker opens and the page shows a default "popular items" list instead of an error.
- A **checkout service** calls a payment provider. If the provider is down, the breaker fails fast with a clear message (or queues the order) rather than letting requests pile up.
- A **service mesh** ejects unhealthy instances of a service from the load-balancing pool automatically.

## Interview Questions

**Q: What is a circuit breaker and what problem does it solve?**
It stops calls to a failing dependency after failures cross a threshold, preventing cascading failures, wasted resources, and retry storms, and giving the dependency time to recover.

**Q: Explain the three states.**
Closed: normal, failures counted. Open: calls fail immediately or use a fallback. Half-open: after a wait, a few trial calls test recovery; success closes the circuit, failure reopens it.

**Q: Why is a slow service worse than a failed one?**
A failed service returns errors quickly, while a slow one holds threads and connections until they run out, which spreads the problem to callers.

**Q: How do circuit breakers, retries, and timeouts work together?**
Timeouts bound each attempt; retries with backoff and jitter handle brief errors; the breaker stops everything when failure is persistent. Don't retry while the circuit is open.

**Q: What would you return when the circuit is open?**
A fallback appropriate to the dependency: cached data, a default response, or a queued request for non-critical calls; a fast, clear error for critical operations that cannot be faked.

**Q: What is a bulkhead?**
Isolating resources such as thread or connection pools per dependency so that a failure or slowdown in one does not exhaust resources needed by others.

## Summary

- A circuit breaker fails fast when a dependency is unhealthy, stopping cascades and retry storms.
- States: closed (normal), open (reject), half-open (test recovery).
- Use failure rate with a minimum call count, a sensible open duration, and treat slow calls as failures.
- Always pair with timeouts, bounded retries with backoff and jitter, and bulkheads, and plan fallbacks.
- Use one breaker per dependency, monitor state changes, and test with injected failures.

## Continue Learning

- [High Availability](system-design/high-availability) — the wider goal these patterns serve: failover, redundancy, and graceful degradation.
- [Messaging and Communication](system-design/messaging-and-communication) — idempotency and retrying safely.
- [Microservices Architecture](system-design/microservices) — where cascading failures are a real risk.
- [Observability and Deployment](system-design/observability-and-deployment) — seeing breaker state in production.
