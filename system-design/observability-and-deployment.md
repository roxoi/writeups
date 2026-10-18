---
title: "Observability and Deployment — Metrics, Logs, Traces, SLOs, Safe Releases, and Disaster Recovery"
description: "How to run systems in production: the three pillars of observability, golden signals, alerting and SLOs with error budgets, deployment strategies (rolling, blue-green, canary, feature flags), database migrations, disaster recovery (RPO and RTO), and incident management."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
created: "2026-10-17"
updated: "2026-10-17"
thumbnail: "/images/devops.png"
tags: [observability, monitoring, slo, deployment, canary, blue-green, disaster-recovery, devops, sre, system-design]
keywords: ["Observability metrics logs traces", "SLI SLO SLA error budget", "Canary vs blue green deployment", "RPO and RTO disaster recovery", "Golden signals", "Prometheus Grafana", "Incident management postmortem"]
---

# Observability and Deployment — Metrics, Logs, Traces, SLOs, Safe Releases, and Disaster Recovery

Designing a system is half the job. The other half is **seeing what it is doing in production** and **changing it safely**. Interviewers for backend and DevOps roles ask about both: "How would you know the system is unhealthy?", "How do you release without downtime?", "What if a whole region fails?"

This guide covers observability, reliability targets, deployment strategies, disaster recovery, and incident handling. Containers and orchestration (Docker, Kubernetes) have their own dedicated tutorials.

![Observability and Deployment](/images/devops.png)

## 1. Observability

**Monitoring** tells you _that_ something is wrong (known failure modes). **Observability** is being able to ask _new_ questions about a running system and find out _why_, using the data it emits. You need three kinds of data.

### The Three Pillars

| Pillar      | What it is                                                              | Good for                                                             | Cost                      |
| ----------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------- |
| **Metrics** | Numeric measurements over time (request rate, error rate, latency, CPU) | Dashboards, alerts, trends, capacity planning                        | Cheap and compact         |
| **Logs**    | Timestamped records of discrete events                                  | Debugging the details of what happened                               | Expensive at volume       |
| **Traces**  | The path of one request across services, as timed spans                 | Finding _where_ in a chain of services time is spent or errors occur | Moderate; usually sampled |

Use them together: **metrics** show that latency is high, **traces** show which service is slow, **logs** show the error inside it.

### Metrics

- **Counters** only increase (requests served, errors). Look at their **rate**.
- **Gauges** go up and down (memory used, queue depth, active connections).
- **Histograms** record distributions, so you can compute **percentiles** (p50, p95, p99).

**Averages hide problems.** If most requests take 50 ms but 1% take 5 seconds, the average looks fine while some users suffer. Always track **p95/p99 latency**, not only the mean. Percentiles can't be averaged across servers, so aggregate histograms instead.

**Watch out for high-cardinality labels.** Adding a label with millions of values (user ID, request ID) to a metric creates millions of time series and can overwhelm the metrics system. Put such details in logs and traces instead.

### What to Measure

Three standard checklists keep you from forgetting something:

| Method                               | Signals                                              | Best for                               |
| ------------------------------------ | ---------------------------------------------------- | -------------------------------------- |
| **Four Golden Signals** (Google SRE) | **Latency**, **Traffic**, **Errors**, **Saturation** | Any user-facing service                |
| **RED**                              | **R**ate, **E**rrors, **D**uration                   | Request-driven microservices           |
| **USE**                              | **U**tilization, **S**aturation, **E**rrors          | Resources (CPU, disk, network, queues) |

Also track **business metrics** (orders per minute, sign-ups) because a system can look healthy technically while the business is broken.

### Logs

- Use **structured logs** (JSON with fields such as `timestamp`, `level`, `service`, `trace_id`, `user_id`) instead of free text, so they can be searched and aggregated.
- Include a **correlation (trace) ID** in every log line, passed through every service call, so you can follow one request across services.
- Use log **levels** sensibly, **never log secrets or personal data**, and set **retention** limits to control cost.
- **Centralize** logs: services write to stdout or a local agent, which ships to a central store (for example ELK/OpenSearch or Loki). Don't rely on SSH-ing into servers.

### Distributed Tracing

A **trace** represents one request; it is made of **spans** (one per operation, with start time and duration), linked by parent-child relationships.

```text
GET /checkout  [ 840 ms ]
├─ auth-service         [  20 ms ]
├─ cart-service         [  45 ms ]
├─ inventory-service    [ 610 ms ]   ← the slow part
│   └─ DB query         [ 590 ms ]
└─ payment-service      [ 120 ms ]
```

- A **trace ID** is generated at the edge and **propagated** in request headers to every downstream call.
- Tracing every request is expensive, so systems **sample** (for example 1%, plus all slow or failed requests).
- **OpenTelemetry** is the vendor-neutral standard for generating metrics, logs, and traces; Jaeger, Zipkin, and Tempo are common trace backends.

### Common Tool Stack

| Need               | Common tools                                            |
| ------------------ | ------------------------------------------------------- |
| Metrics            | Prometheus (collects by scraping), Grafana (dashboards) |
| Logs               | ELK / OpenSearch, Loki                                  |
| Traces             | Jaeger, Zipkin, Tempo                                   |
| Instrumentation    | OpenTelemetry                                           |
| Alerting           | Alertmanager, PagerDuty, Opsgenie                       |
| Managed all-in-one | Datadog, New Relic, CloudWatch, Google Cloud Operations |

### Alerting That People Don't Hate

- **Alert on symptoms users feel** (error rate, latency, failed checkouts), not on every internal cause (CPU at 80%).
- Every alert should be **actionable**: someone must need to do something now. If not, make it a dashboard or a ticket.
- Include a **runbook link** with what to check first.
- Use **severity levels** and route only the urgent ones to a pager.
- Avoid **alert fatigue**: noisy alerts get ignored, so tune or delete them.
- Always watch the **monitoring system itself** (a dead-man's-switch alert that fires if the alerts stop).
- Add **health checks** and **synthetic tests** (a script that performs a real user flow every minute).

## 2. SLIs, SLOs, SLAs, and Error Budgets

| Term                              | Meaning                                               | Example                                                            |
| --------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------ |
| **SLI** (Service Level Indicator) | A measurement of service behavior                     | Fraction of requests that succeed; fraction served in under 300 ms |
| **SLO** (Service Level Objective) | The **internal target** for an SLI                    | 99.9% of requests succeed over 30 days                             |
| **SLA** (Service Level Agreement) | A **contract** with customers, usually with penalties | 99.5% monthly uptime or service credits                            |

Keep the SLO **stricter than the SLA**, so you notice and fix problems before a contractual breach.

### Error Budget

```text
Error budget = 100% − SLO
```

A 99.9% SLO over 30 days allows about **43 minutes** of unavailability. That budget is a tool:

- **Budget left:** ship features and take risks.
- **Budget nearly spent:** slow down, freeze risky releases, and focus on reliability.

This turns the "ship faster vs stay stable" argument into a shared, data-driven rule. Teams often alert on **burn rate** (how fast the budget is being consumed), so a sharp spike pages immediately and a slow leak raises a ticket.

### Picking Good SLIs

- Measure what **users experience**, ideally at the edge (load balancer) or client.
- Prefer ratios: `good events / total events`.
- Not everything needs 99.99%. Each extra nine costs much more (see [High Availability](system-design/high-availability)).

## 3. Deployment Strategies

The goal: release changes **without downtime** and with the ability to **undo quickly**.

### Rolling Deployment

Replace instances **a few at a time** with the new version, while the rest keep serving traffic.

- **Pros:** no extra infrastructure, simple, no downtime.
- **Cons:** old and new versions **run side by side** during the rollout, so they must be compatible (APIs, database schema, message formats). Rollback means rolling forward again, which is slow.

### Blue-Green Deployment

Maintain two identical environments: **blue** (live) and **green** (new). Deploy and test on green, then **switch traffic** (load balancer or DNS) from blue to green all at once.

- **Pros:** instant switch, **instant rollback** (switch back), full testing before exposure.
- **Cons:** double the infrastructure during release; **database changes** must work for both versions; all users hit the new version at the same time.

### Canary Deployment

Send a **small percentage** of traffic (1%, then 5%, 25%, 100%) to the new version, watching metrics at each step. If error rate or latency worsens, **automatically roll back**.

- **Pros:** limits the **blast radius** of a bad release and tests with real traffic.
- **Cons:** more complex (traffic splitting, automated metric comparison), and needs good observability.

### Feature Flags

Deploy the code **turned off**, and enable it with a configuration switch, per user, percentage, or region.

- **Decouples deployment from release.** You can ship code any time and enable it later.
- **Instant "kill switch"** if the feature misbehaves, without a redeploy.
- Enables A/B tests and gradual rollouts.
- **Cost:** flags add complexity. Remove old flags promptly to avoid a tangle of dead code paths.

### Shadow (Dark) Traffic

Copy real production traffic to the new version **without returning its responses to users**, to compare behavior and performance safely. Useful for rewrites and new rate-limit rules (see [Design a Rate Limiter](system-design/rate-limiter-design)).

### Comparison

| Strategy     | Downtime | Rollback speed            | Extra cost | Risk exposure | Best for                                    |
| ------------ | -------- | ------------------------- | ---------- | ------------- | ------------------------------------------- |
| Rolling      | None     | Slow                      | Low        | Gradual       | Default for stateless services              |
| Blue-green   | None     | **Instant**               | High (2x)  | All at once   | Critical releases needing fast rollback     |
| Canary       | None     | Fast (shift traffic back) | Low–medium | **Smallest**  | High-traffic, high-risk changes             |
| Feature flag | None     | Instant (toggle)          | Low        | Controlled    | Releasing features independently of deploys |

### Make Deployments Safe

- **Health checks and readiness checks** so traffic only goes to ready instances, plus **connection draining** so in-flight requests finish (see [Load Balancing](system-design/load-balancing)).
- **Automated tests** and a CI/CD pipeline that builds once and promotes the **same artifact** through environments.
- **Automatic rollback** triggers based on metrics.
- **Backward-compatible changes** across versions (see next section).
- Avoid deploying risky changes on Fridays or during peak hours, and keep a rollback plan.

### Database Migrations Without Downtime: Expand and Contract

Old and new application versions often run at the same time, so schema changes must be **backward compatible**. The safe pattern has several releases:

1. **Expand:** add the new column or table (nullable, with a default). Old code ignores it.
2. **Dual write / backfill:** new code writes to both old and new, and a job backfills existing rows.
3. **Switch reads** to the new column once data is complete and verified.
4. **Contract:** after all code stops using the old column, drop it.

**Never** rename or drop a column in a single step while old code is running. Large table changes need online schema-change tools to avoid locking.

## 4. Disaster Recovery

High availability handles _common_ failures (a server dies). **Disaster recovery (DR)** handles _large_ failures (data center or region loss, data corruption, ransomware, a catastrophic human error).

### Two Numbers That Define DR

| Term                               | Meaning                                                          | Question it answers            |
| ---------------------------------- | ---------------------------------------------------------------- | ------------------------------ |
| **RPO** (Recovery Point Objective) | The maximum **amount of data loss** you accept, measured in time | "How far back can we restore?" |
| **RTO** (Recovery Time Objective)  | The maximum **downtime** you accept                              | "How quickly must we be back?" |

Example: RPO = 5 minutes means you must have a copy of the data no more than 5 minutes old; RTO = 1 hour means service must be restored within 1 hour. **Smaller numbers cost much more.**

### DR Strategies (cheapest to most expensive)

| Strategy                         | What's kept in the DR region                                     | RTO             | RPO                       | Cost                                   |
| -------------------------------- | ---------------------------------------------------------------- | --------------- | ------------------------- | -------------------------------------- |
| **Backup and restore**           | Only backups; rebuild infrastructure after a disaster            | Hours           | Hours (since last backup) | Lowest                                 |
| **Pilot light**                  | Data replicated and a minimal core running; scale up on disaster | Tens of minutes | Minutes                   | Low                                    |
| **Warm standby**                 | A scaled-down full copy always running                           | Minutes         | Seconds to minutes        | Medium                                 |
| **Active-active (multi-region)** | Full capacity in several regions serving traffic                 | Near zero       | Near zero                 | Highest, with data-conflict complexity |

### Backups: The Rules

- **3-2-1 rule:** 3 copies of data, on 2 different media, with 1 off-site (another region or account).
- **Automate** backups and keep **point-in-time recovery** (continuous log archiving) for databases.
- **Test restores regularly.** A backup that has never been restored is only a hope.
- Protect backups from deletion (immutability, separate credentials), because attackers and mistakes target them too.
- Replication is **not** a backup: a bad delete or corruption replicates instantly to every copy.

### Multi-Region Considerations

- Data replication across regions is usually **asynchronous** (a failover may lose the last few seconds), and conflict handling is needed for multi-writer designs (see [Databases](system-design/databases)).
- Route users with **GeoDNS or Anycast** and health-based failover (see [DNS](system-design/dns)).
- **Rehearse failover** (game days). An untested DR plan usually fails when needed.
- Mind **data residency** laws that restrict where data may live.

## 5. Incident Management

Every system fails eventually. What matters is how fast you detect, mitigate, and learn.

### During an Incident

1. **Detect:** alert fires (ideally before customers report it).
2. **Triage and assign roles:** an **incident commander** coordinates, others investigate and communicate. One person leads.
3. **Mitigate first, diagnose later:** roll back, disable a feature flag, fail over, or scale up to **stop the user impact**, then find the root cause.
4. **Communicate:** keep stakeholders and a status page updated at regular intervals.
5. **Resolve and verify** using the same metrics that detected the problem.

### After: The Blameless Postmortem

- Write up: timeline, impact, **root causes and contributing factors**, what went well, what went badly, and **concrete action items** with owners.
- **Blameless:** focus on how the system and process allowed the failure, not on who made the mistake. People hide problems when they fear blame.
- Track action items to completion; unaddressed postmortems repeat.

### Useful Metrics

- **MTTD** (mean time to detect) and **MTTR** (mean time to recover/resolve): improve them with better alerts, runbooks, and automation.

### Resilience Practices

- **Chaos engineering:** deliberately inject failures (kill instances, add latency) in controlled conditions to find weaknesses before real outages do.
- **Load testing** before big events, and **capacity planning** with headroom.
- **Runbooks** for common problems, and an on-call rotation that is sustainable.

## Putting It Together: An Example

For a checkout service, a mature setup looks like this:

- **SLO:** 99.9% of checkout requests succeed, and 95% complete in under 800 ms, measured at the load balancer.
- **Dashboards:** the four golden signals per service, queue depth, dependency health, and business metrics (orders per minute).
- **Alerts:** burn-rate alert on the SLO (page), plus ticket-level alerts for slow budget burn.
- **Tracing** with sampled traces plus all errors, using a correlation ID in every log.
- **Release:** CI builds one artifact, deploys via **canary** (5% → 25% → 100%) with automatic rollback if the error rate rises; risky features hidden behind **feature flags**.
- **Database changes:** expand/contract migrations.
- **DR:** warm standby in a second region, RPO 1 minute (continuous replication), RTO 30 minutes, with a quarterly failover drill.
- **Incidents:** runbooks, an on-call rotation, and blameless postmortems.

## Interview Questions

**Q: What is the difference between monitoring and observability?**
Monitoring watches known conditions and alerts when they break. Observability lets you investigate unknown problems by exploring rich data (metrics, logs, traces) to understand why the system behaves as it does.

**Q: What are the three pillars of observability, and when do you use each?**
Metrics for trends and alerting, logs for detailed event debugging, traces for following a request across services. Metrics tell you something is wrong, traces locate it, logs explain it.

**Q: What are the four golden signals?**
Latency, traffic, errors, and saturation.

**Q: Why use percentiles instead of averages for latency?**
Averages hide slow outliers. p99 shows what the slowest 1% of users experience, and these tail latencies often dominate real user pain.

**Q: What is an error budget?**
The amount of unreliability allowed by an SLO (100% minus the SLO). It balances feature velocity and stability: spend it on releases, and when it is exhausted, prioritize reliability work.

**Q: Blue-green vs canary?**
Blue-green switches all traffic between two full environments, giving instant rollback but doubling infrastructure and exposing everyone at once. Canary shifts traffic gradually and watches metrics, limiting the blast radius at the cost of more complex tooling.

**Q: How do you change a database schema with zero downtime?**
Expand and contract: add the new structure in a backward-compatible way, dual-write and backfill, switch reads, then remove the old structure in a later release.

**Q: Define RPO and RTO, and name DR strategies.**
RPO is the acceptable data loss (in time); RTO is the acceptable downtime. Strategies in increasing cost: backup and restore, pilot light, warm standby, and active-active.

**Q: Is replication a backup?**
No. Corruption or accidental deletion replicates to all copies. Backups need to be independent, versioned, protected, and regularly test-restored.

**Q: What do you do first when an incident happens?**
Mitigate user impact (roll back, flip a flag, fail over), keep communication clear with one incident commander, and investigate the root cause afterward, followed by a blameless postmortem.

## Summary

- Use **metrics, logs, and traces** together, track the **golden signals** and **percentiles**, and alert on **user-visible symptoms**.
- **SLIs and SLOs** define reliability targets, and the **error budget** balances speed and stability.
- Release safely with **rolling, blue-green, canary, and feature-flag** strategies, backward-compatible changes, and automatic rollback.
- Change databases with **expand and contract**.
- Plan disaster recovery with explicit **RPO and RTO**, test backups and failovers, and remember that replication is not a backup.
- Handle incidents with clear roles, mitigate first, and learn through **blameless postmortems**.

## Continue Learning

- [High Availability](system-design/high-availability) — redundancy, failover, and availability targets.
- [Load Balancing](system-design/load-balancing) — health checks, draining, and traffic shifting for releases.
- [Microservices Architecture](system-design/microservices) — why tracing and correlation IDs matter.
- [Circuit Breaker Pattern](system-design/circuit-breaker) — failure containment you should be able to observe.
- [System Design Cheat Sheet](system-design/cheat-sheet) — quick revision for interviews.
