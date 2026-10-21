---
title: "High Availability"
description: "How to design systems that stay up: availability math, redundancy, failover, failure domains, split-brain, and the difference between high availability, fault tolerance, and disaster recovery."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-04-18"
updated: "2026-10-17"
thumbnail: "/images/system-design-arch.png"
tags: [availability, high-availability, redundancy, failover, reliability, system-design]
keywords: ["High availability system design", "Availability nines", "Active-active vs active-passive", "Failover", "Split brain", "Single point of failure"]
---

# High Availability

Availability is the **percentage of time a system is operational**. High availability (HA) is the practice of designing a system so that it keeps serving users despite failures of machines, networks, or whole data centers. This guide covers the math, the redundancy patterns, how failover works, and the traps that make redundant systems fail anyway.

## Measuring Availability

Availability is commonly expressed in "nines":

| Nines   | Availability | Downtime per Year | Downtime per Month |
| ------- | ------------ | ----------------- | ------------------ |
| 2 nines | 99%          | 3.65 days         | 7.31 hours         |
| 3 nines | 99.9%        | 8.77 hours        | 43.8 minutes       |
| 4 nines | 99.99%       | 52.6 minutes      | 4.38 minutes       |
| 5 nines | 99.999%      | 5.26 minutes      | 26.3 seconds       |

> Figures use an average year of 365.25 days; a month is one twelfth of that.

Each extra nine cuts allowed downtime by 10x, and cost usually rises much more than 10x. Five nines leaves so little time that a human cannot even respond to an alert, so recovery has to be fully automatic.

Availability connects to two time measures:

```text
Availability = MTBF / (MTBF + MTTR)
```

- **MTBF** (Mean Time Between Failures): how long the system runs before failing.
- **MTTR** (Mean Time To Repair): how long it takes to recover.

You can raise availability by failing less often _or_ recovering faster. Fast, automated recovery is usually cheaper and more reliable than trying to prevent every failure.

### Availability in Series vs Parallel

**Series** (a request needs _all_ components): availability decreases.

```text
Total = A × B
Example: 99.9% × 99.9% = 99.8%
```

**Parallel** (redundant copies, any one can serve): availability increases.

```text
Total = 1 − (1 − A) × (1 − B)
Example: 1 − (0.001 × 0.001) = 99.9999%
```

Two takeaways:

1. Every extra **required** dependency in a request path lowers availability. A request that touches five services at 99.9% each is only about 99.5% available (0.999⁵).
2. Redundancy raises availability, but only if failures are **independent**. Two servers on the same rack, power feed, or deployment pipeline can fail together, so the real number is lower than the formula suggests.

## Eliminate Single Points of Failure

A **single point of failure (SPOF)** is any component whose failure takes the whole system down. High availability starts by walking through every layer and asking _"what if this dies?"_

| Layer            | Common SPOF                        | Fix                                                    |
| ---------------- | ---------------------------------- | ------------------------------------------------------ |
| DNS              | One DNS provider                   | Multiple providers or a managed, redundant DNS service |
| Load balancer    | A single balancer                  | A redundant pair or a managed multi-AZ balancer        |
| App servers      | One instance                       | Multiple stateless instances                           |
| Database         | One primary                        | Replicas with automatic failover                       |
| Network          | One link or one switch             | Redundant paths                                        |
| Data center      | One site                           | Multiple availability zones or regions                 |
| People / process | One person who knows how to deploy | Runbooks, automation, shared ownership                 |

Note that adding a redundant component can add a new SPOF (the thing that decides which copy is active). Keep asking the question after each fix.

## Redundancy Patterns

### Active-Passive (Failover)

One node serves traffic. A standby waits to take over if the active node fails.

| Standby type | State                                      | Failover speed | Cost    |
| ------------ | ------------------------------------------ | -------------- | ------- |
| **Hot**      | Running and kept in sync                   | Seconds        | Highest |
| **Warm**     | Running but scaled down or slightly behind | Minutes        | Medium  |
| **Cold**     | Not running; started on demand             | Longest        | Lowest  |

Simple to reason about, but the standby capacity sits idle, and failover itself is a moment of risk (it may fail, or be slow).

### Active-Active

All nodes serve traffic at the same time. If one fails, the others absorb its load.

- Better resource use and no failover event for stateless tiers.
- Harder for **stateful** components. Multiple writers can create conflicting data, so you need conflict handling (see [Consistency Models](system-design/databases) and Replication).

### N+1 Redundancy and Capacity After Failure

If you have N nodes and tolerate one failure, the remaining N − 1 must carry all the traffic. After one failure, utilization becomes:

```text
new utilization = old utilization × N / (N − 1)
```

Three nodes running at 70% each look healthy, but if one dies, the other two must run at 70% × 3 / 2 = **105%**, so they overload and the failure cascades. Redundancy only works if you keep **headroom**.

## Failure Domains

A failure domain is a group of resources that can fail together. Spread redundant copies across domains so one event cannot take out everything.

| Domain                 | Example failure                                        |
| ---------------------- | ------------------------------------------------------ |
| Host                   | Hardware crash                                         |
| Rack                   | Power or top-of-rack switch failure                    |
| Availability Zone (AZ) | Data center power, cooling, or network issue           |
| Region                 | Large-scale natural disaster, regional cloud outage    |
| Software               | A bad deploy or a bug that hits every node identically |

**Multi-AZ** is the standard baseline for HA: low latency between zones, and it survives a data center failure. **Multi-region** survives a regional outage but brings higher latency, higher cost, and data-replication challenges (see [Disaster Recovery](system-design/observability-and-deployment)).

Software is also a failure domain. Ten identical servers running the same buggy release are not redundant against that bug, which is why [deployment strategies](system-design/observability-and-deployment) like canary releases matter for availability.

## How Failover Works

Failover has four steps, and each can go wrong:

1. **Detect** the failure using health checks or heartbeats. Too sensitive and you fail over on a hiccup; too slow and users suffer.
2. **Decide** which node takes over (often with a quorum or a coordinator).
3. **Redirect** traffic to the new node: a floating IP, a load balancer, or DNS.
4. **Recover** the old node and rejoin it safely.

Practical details:

- **DNS-based failover** is limited by caching: clients and resolvers may keep using the old address past its TTL. Load balancers and virtual IPs fail over faster.
- **Health check depth:** a shallow check (process is up) can miss real failures. A deep check that tests every dependency can remove _all_ servers at once when a shared dependency blips. Keep checks meaningful but not over-eager.
- **Database failover and data loss:** with **asynchronous replication**, the replica may be missing the latest writes when promoted, so you may lose recent data. **Synchronous replication** avoids loss but adds write latency and makes the primary depend on the replica being reachable.

### Split-Brain

Split-brain happens when a network partition makes two nodes each believe they are the leader. Both accept writes and the data diverges.

Prevention techniques:

- **Quorum:** a node only acts as leader if it can reach a majority of the cluster. With a partition, at most one side has a majority.
- **Fencing:** the old leader is cut off from shared resources (using fencing tokens, or by forcibly powering it off).

See [Consensus and Coordination](system-design/distributed-systems-essentials) for how leader election does this safely.

## High Availability vs Fault Tolerance vs Disaster Recovery

These terms are related but different:

| Term                  | Meaning                                                          | Typical behavior                                                          |
| --------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **High Availability** | Minimize downtime                                                | A failure may cause a brief interruption (seconds) while failover happens |
| **Fault Tolerance**   | Keep operating _without interruption_ through component failures | Requires full redundancy running in lockstep; much more expensive         |
| **Disaster Recovery** | Restore service after a major event                              | Defined by RPO (data loss tolerated) and RTO (time to recover)            |
| **Reliability**       | Behave correctly                                                 | A system can be up (available) and still wrong (unreliable)               |

Most business systems aim for HA, and reserve true fault tolerance for cases like payment switches or flight control where even a brief interruption is unacceptable. See Fault Tolerance.

## Design Principles for High Availability

- **Remove single points of failure** at every layer.
- **Detect failures quickly** and **automate recovery**. Do not rely on a human at 3 a.m.
- **Keep headroom** so surviving nodes can absorb the load of failed ones.
- **Limit the blast radius.** Split users or tenants into independent cells so a bad deploy or noisy customer affects only one slice.
- **Degrade gracefully.** Serve cached or reduced functionality rather than an error page.
- **Separate critical from non-critical dependencies.** If recommendations are down, the product page should still load.
- **Test failure on purpose.** Failover that has never been exercised usually does not work. Practices like game days and chaos engineering (randomly killing instances in production-like environments) find problems before real outages do.
- **Measure what users experience** using SLIs and SLOs (see [SLI, SLO, and SLA](system-design/observability-and-deployment)).

## Trade-offs

| Choice                  | You gain                             | You give up                                            |
| ----------------------- | ------------------------------------ | ------------------------------------------------------ |
| More nines              | Less downtime                        | Cost and complexity rise steeply                       |
| Active-active           | Better utilization, instant takeover | Conflict handling and complexity                       |
| Synchronous replication | No data loss on failover             | Higher write latency, availability tied to the replica |
| Multi-region            | Survives regional outages            | Latency, cost, and consistency challenges              |

## Real-World Usage

- **Web application:** multiple stateless instances across AZs behind a managed load balancer, with a replicated database and automatic failover.
- **Managed cloud databases** typically offer a multi-AZ option that keeps a synchronous standby in another zone and promotes it automatically.
- **Large streaming and e-commerce platforms** run in multiple regions and routinely rehearse region evacuation.

## Interview Questions

**Q: How many hours of downtime per year does 99.9% availability allow?**
About 8.77 hours (0.1% of 8,766 hours).

**Q: Two components each have 99.9% availability. What is the availability if both are required? If either one is enough?**
Required in series: about 99.8%. Redundant in parallel: 99.9999%, assuming independent failures.

**Q: What is the difference between active-passive and active-active?**
Active-passive has a standby that takes over on failure, which is simple but leaves idle capacity and has a failover moment. Active-active has all nodes serving traffic, which uses resources better but makes shared state and conflicts harder.

**Q: What is split-brain and how do you prevent it?**
Two nodes both believing they are the leader after a partition, causing divergent data. Prevent it with quorum-based leader election and fencing of the old leader.

**Q: Three servers run at 70% CPU each. One fails. What happens?**
The remaining two must carry the load, reaching about 105% utilization, so they overload and may fail too. You need headroom (N+1 sizing) or autoscaling that reacts fast enough.

## Summary

- Availability = MTBF / (MTBF + MTTR). Raise it by failing less or recovering faster, and prefer fast automated recovery.
- Series dependencies lower availability; parallel redundancy raises it, but only for independent failures.
- Find and remove every single point of failure, including the ones your fixes introduce.
- Spread copies across failure domains (hosts, AZs, regions), and keep headroom for N+1.
- Failover has detection, decision, redirection, and recovery steps, and each needs testing.
- HA, fault tolerance, and disaster recovery are different goals with different costs.

## Continue Learning

- [Databases](system-design/databases) — replication, failover, and quorums.
- [Observability and Deployment](system-design/observability-and-deployment) — RPO, RTO, and multi-region.
- [Load Balancing](system-design/load-balancing) — routing around failed servers.
