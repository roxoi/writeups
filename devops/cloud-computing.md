---
title: "Cloud Computing: From First Principles to Interview Confidence"
description: "A beginner-friendly guide to cloud computing - service models, regions, compute, storage, networking, databases, security, reliability, IaC, CI/CD and cost, with AWS, Azure and Google Cloud examples and interview questions."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/cloud-computing.png"
tags: [Cloud-Computing, AWS, Azure, Google-Cloud, DevOps, Interview-Preparation]
keywords: ["Cloud computing tutorial for beginners", "IaaS PaaS SaaS explained", "AWS Azure GCP service comparison", "Cloud architecture interview questions", "High availability and disaster recovery in cloud"]
---

# Cloud Computing

![Cloud Computing](/images/cloud-computing.png)

Each topic follows the same path: **the idea in plain words, a real-life picture, how it looks in practice, what happens under the hood, common mistakes, and practice questions** that go from easy to hard.

The ideas are the same on every provider. Examples use AWS names, with Azure and Google Cloud equivalents in a table at the end.

## 1. What Is Cloud Computing?

**The idea:** instead of buying and running your own servers, you **rent computing resources over the internet** (servers, storage, databases, networks) and pay for what you use.

**Picture it:** electricity. You don't build a power plant to run your lights. You plug in and get a bill at the end of the month. Cloud does the same for computing.

**Five core traits (the official definition):**

1. **On-demand self-service:** you create resources yourself, in minutes, with no phone call.
2. **Broad network access:** everything is reached over the network (web console, CLI, API).
3. **Resource pooling:** the provider shares huge pools of hardware between many customers (multi-tenancy).
4. **Rapid elasticity:** scale up when busy, scale down when quiet.
5. **Measured service:** usage is metered, so you pay per second, hour, GB, or request.

**Why companies move to the cloud:**

- No big upfront hardware cost (capital expense becomes operating expense).
- Launch in new countries in minutes.
- Handle traffic spikes without owning idle machines.
- Use managed services (databases, queues, AI) instead of running them yourself.

**Under the hood:** a provider owns giant data centers full of ordinary servers. Software (a _hypervisor_) splits one physical server into many isolated virtual machines. A control system (the "cloud API") decides which physical machine runs your request, wires up the network, and starts billing.

**Common mistake:** thinking cloud is automatically cheaper. It is cheaper when you scale to match demand and use managed services well. A machine left running 24/7 at full size can cost more than owning one.

**Practice**

- **Q: Cloud vs traditional hosting?** Cloud is self-service, elastic, and pay-as-you-go. Traditional hosting usually means fixed contracts and manual provisioning.
- **Q: What is multi-tenancy, and what risk does it bring?** Many customers share the same hardware, kept apart by virtualization and access controls. The risk is isolation failures or a "noisy neighbor" slowing you down.

## 2. Service Models: IaaS, PaaS, SaaS (and Serverless)

**The idea:** cloud services differ in **how much the provider manages for you**.

**Picture it:** making pizza.

- **On-premises:** you buy ingredients, build the oven, make and eat the pizza at home.
- **IaaS:** you rent the kitchen and oven, and you cook.
- **PaaS:** the kitchen and the cooking tools are ready, you just bring the recipe.
- **SaaS:** you order a finished pizza.

| Model                     | You manage                  | Provider manages                           | Example                               |
| ------------------------- | --------------------------- | ------------------------------------------ | ------------------------------------- |
| **IaaS** (Infrastructure) | OS, runtime, app, data      | Hardware, virtualization, network          | EC2 virtual machines                  |
| **PaaS** (Platform)       | App and data                | Everything below, including OS and runtime | Elastic Beanstalk, App Engine, Heroku |
| **FaaS / Serverless**     | Just your function code     | Servers, scaling, patching                 | AWS Lambda                            |
| **SaaS** (Software)       | Only your settings and data | Everything                                 | Gmail, Salesforce, Slack              |

**Rule of thumb:** the more the provider manages, the less control and the less operational work you have. Choose the highest level that still meets your needs.

**Deployment models:**

- **Public cloud:** shared infrastructure run by a provider (AWS, Azure, GCP).
- **Private cloud:** cloud-style infrastructure used by one organization only.
- **Hybrid cloud:** public plus private/on-premises, connected together.
- **Multi-cloud:** using more than one public provider (to avoid lock-in or use the best service of each). It adds complexity, so do it for a reason.

**Practice**

- **Q: Where does a Lambda function fit?** It is "serverless" (FaaS): you upload code, and the provider runs it on demand and scales it automatically.
- **Q: Why might a company choose hybrid?** Data-residency laws, existing hardware investment, or very low latency to on-premises systems.

## 3. The Global Layout: Regions, Availability Zones, Edge

**The idea:** the provider's data centers are organized in layers so you can choose where your app runs and survive failures.

- **Region:** a geographic area (for example, Frankfurt `eu-central-1`) containing multiple data center groups.
- **Availability Zone (AZ):** one or more separate data centers inside a region, with independent power, cooling and networking, connected by fast links. A fire or power cut in one AZ should not affect another.
- **Edge locations / CDN:** many small sites near users that cache content (such as CloudFront or Cloudflare).

**Picture it:** a bank with branches in many cities (regions). Each city has several separate buildings (AZs), so one building failing doesn't close the bank in that city.

**How to choose a region:** latency to users, **data-residency laws** (for example GDPR for EU data), service availability, and price.

**Common mistake:** running everything in a single AZ. If it fails, you are down. Spread across at least two AZs.

**Practice**

- **Q: Region vs AZ?** A region is a geographic area. An AZ is an isolated data center group inside it. Use multiple AZs for high availability, and multiple regions for disaster recovery or global latency.

## 4. Compute: Where Your Code Runs

**The idea:** there are three main ways to run code.

### 4.1 Virtual machines (VMs)

A VM is a **software-simulated computer** with its own OS, running on shared hardware.

**Under the hood:** a _hypervisor_ (KVM, Xen, Nitro) divides CPU, memory and disk among VMs and isolates them. You pick an _instance type_ (CPU/memory size). Common billing choices:

- **On-demand:** pay per second, no commitment, highest price.
- **Reserved / savings plans:** commit for 1-3 years for a big discount.
- **Spot / preemptible:** spare capacity at a large discount, but the provider can take it back with short notice. Great for batch jobs, bad for single critical servers.

### 4.2 Containers

A container packages your app plus everything it needs (libraries, config) so it runs the same everywhere.

**Picture it:** a shipping container. Whatever is inside, the outside is standard, so any ship, truck or crane can handle it.

**Under the hood:** containers share the host's kernel (unlike VMs, which each carry a full OS). They use two Linux features: **namespaces** (each container sees its own processes, network, filesystem) and **cgroups** (limit CPU and memory). That makes them start in milliseconds and use much less memory than VMs, but with weaker isolation.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 1000                          # don't run as root
CMD ["python", "app.py"]
```

**Kubernetes** runs many containers across many machines. It restarts failed ones, scales them, rolls out updates, and routes traffic. Key objects: **Pod** (one or more containers), **Deployment** (keeps N copies running), **Service** (stable address for pods), **Ingress** (HTTP entry point).

### 4.3 Serverless functions

You write a function. The provider runs it only when triggered (an HTTP request, a file upload, a queue message) and bills per execution time.

```python
def handler(event, context):                 # AWS Lambda style
    name = event.get("name", "world")
    return {"statusCode": 200, "body": f"Hello {name}"}
```

**Trade-offs:**

- Pros: no servers to manage, scales to zero, pay only for use.
- Cons: **cold starts** (first call after idle is slower), execution time limits, harder local debugging, and can become costly at constant high load.

**Which one when?**

| Need                                         | Pick                                                   |
| -------------------------------------------- | ------------------------------------------------------ |
| Full control, legacy apps, special OS        | VMs                                                    |
| Many microservices, portability, steady load | Containers (Kubernetes or a managed container service) |
| Spiky or event-driven work, small tasks      | Serverless                                             |

**Practice**

- **Q: VM vs container?** A VM virtualizes hardware and runs a full OS. A container shares the host kernel and isolates processes, so it's lighter and faster to start, with weaker isolation.
- **Q: What is a cold start and how do you reduce it?** The delay when a new function instance starts. Reduce it with smaller packages, lighter runtimes, provisioned concurrency, and avoiding heavy work at startup.

## 5. Storage: Three Types

**The idea:** pick storage by _how you access the data_.

| Type       | What it is                                            | Example             | Good for                                           |
| ---------- | ----------------------------------------------------- | ------------------- | -------------------------------------------------- |
| **Object** | Files ("objects") in flat buckets, accessed over HTTP | S3, Azure Blob, GCS | Images, backups, logs, data lakes, static websites |
| **Block**  | A raw disk attached to one VM                         | EBS, Azure Disk     | Databases, operating systems                       |
| **File**   | Shared network drive many machines mount              | EFS, Azure Files    | Shared content, home directories                   |

**Picture it:** object storage is a warehouse with labeled boxes (you ask for box #123). Block storage is a hard drive inside your computer. File storage is a shared office drive.

**Under the hood of object storage:** data is split and stored redundantly across multiple devices and AZs, giving very high durability (S3 advertises 99.999999999%, "eleven nines"). It is a flat key-value store; "folders" are just key prefixes. Objects are replaced as a whole rather than edited in place.

**Storage classes and lifecycle:** frequently used data stays in a "hot" class. Rarely used data moves automatically to cheaper "cold" or "archive" classes. Lifecycle rules do this for you.

```json
{
  "Rules": [
    {
      "ID": "archive-old-logs",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [{ "Days": 30, "StorageClass": "GLACIER" }],
      "Expiration": { "Days": 365 }
    }
  ]
}
```

**Common mistakes:**

- **Public buckets** (the cause of many data leaks). Block public access by default.
- Treating object storage like a fast database.
- Forgetting versioning and backups.

**Practice**

- **Q: Durability vs availability?** Durability is the chance data is _not lost_. Availability is the chance you can _access it right now_. Data can be durable but briefly unavailable.
- **Q: Why can't you edit part of an S3 object?** Objects are immutable units. You upload a new version of the whole object.

## 6. Networking in the Cloud

**The idea:** you build your own private network inside the provider's network.

**Key building blocks (AWS names):**

- **VPC (Virtual Private Cloud):** your isolated network, defined by an IP range like `10.0.0.0/16`.
- **Subnet:** a slice of the VPC inside one AZ.
  - **Public subnet:** has a route to an **Internet Gateway**, so resources can have public IPs.
  - **Private subnet:** no direct inbound internet access. Outbound goes through a **NAT Gateway**.
- **Route table:** rules deciding where traffic goes.
- **Security group:** a **stateful** firewall attached to a resource. It allows rules only. Return traffic is automatically allowed.
- **Network ACL:** a **stateless** firewall at the subnet level. You must allow both directions.
- **Load balancer:** spreads incoming traffic across many servers.
- **DNS (Route 53):** turns names into addresses, and can route by health, latency or location.
- **CDN:** caches content close to users.

**A standard safe layout:**

```
Internet
   |
[Load Balancer]        <- public subnets (2 AZs)
   |
[App servers]          <- private subnets (2 AZs)
   |
[Database]             <- private subnets, no internet route
```

Only the load balancer is exposed. Servers and databases are not reachable directly.

**CIDR basics:** `10.0.0.0/16` means the first 16 bits are fixed, leaving 65,536 addresses. A `/24` has 256. A bigger number after the slash means a smaller network.

**Common mistakes:**

- Opening SSH (port 22) or database ports to `0.0.0.0/0` ("the whole internet").
- Overlapping IP ranges between networks you later want to connect.
- Putting databases in public subnets.

**Practice**

- **Q: Security group vs network ACL?** Security group: stateful, attached to resources, allow-only. NACL: stateless, attached to subnets, can allow and deny, evaluated in order.
- **Q: How does a private server download updates?** Through a NAT gateway in a public subnet, which allows outbound connections while blocking unsolicited inbound ones.

## 7. Databases in the Cloud

**The idea:** you can run a database yourself on a VM, or use a **managed database** where the provider handles patching, backups, failover and scaling.

| Kind                             | Examples                           | Best for                                        |
| -------------------------------- | ---------------------------------- | ----------------------------------------------- |
| **Relational (SQL)**             | RDS, Aurora, Cloud SQL             | Structured data, joins, transactions            |
| **Key-value / document (NoSQL)** | DynamoDB, Cosmos DB, MongoDB Atlas | Huge scale, simple access patterns, low latency |
| **In-memory cache**              | ElastiCache (Redis), Memcached     | Speeding up repeated reads                      |
| **Data warehouse**               | Redshift, BigQuery, Snowflake      | Analytics on large data                         |
| **Search**                       | OpenSearch, Elasticsearch          | Full-text search                                |

**Scaling databases:**

- **Vertical:** use a bigger machine (simple, limited).
- **Read replicas:** copies that serve reads, reducing load on the main database. Replication is usually _asynchronous_, so replicas can lag slightly.
- **Multi-AZ:** a standby copy in another AZ for automatic failover (for availability, not for read scaling).
- **Sharding:** split data across multiple databases by a key.

**CAP theorem and consistency (a favorite topic):** in a distributed system, when the network splits (**P**artition), you must choose between **C**onsistency (everyone sees the same latest data) and **A**vailability (every request gets an answer). So systems lean toward CP or AP.

- **Strong consistency:** a read always sees the latest write.
- **Eventual consistency:** replicas converge over time, so a read right after a write may be stale.

**Common mistakes:**

- Using one giant database for everything.
- Ignoring replica lag (read-your-own-write bugs).
- No backups, or backups never tested by restoring.

**Practice**

- **Q: Multi-AZ vs read replica?** Multi-AZ is for failover (high availability). Read replicas are for scaling reads. They solve different problems.
- **Q: When would you pick DynamoDB over PostgreSQL?** Predictable key-based access at very large scale with low latency and minimal operations. Pick PostgreSQL for complex queries, joins and flexible reporting.

## 8. Scaling, Load Balancing and Elasticity

**The idea:** handle more load by adding capacity, and remove it when not needed.

- **Vertical scaling (scale up):** a bigger machine. Easy, but it has a ceiling and usually needs downtime.
- **Horizontal scaling (scale out):** more machines. Almost unlimited, but your app must be designed for it.

**Picture it:** a busy shop. Vertical = one super-fast cashier. Horizontal = open more checkout lines.

**What makes an app scalable horizontally:** it must be **stateless**. Any server can handle any request. Keep user sessions in a shared store (Redis, database) rather than in one server's memory.

**Load balancer types:**

- **Layer 7 (application):** understands HTTP, can route by URL or header.
- **Layer 4 (network):** routes TCP/UDP packets, extremely fast.
- Health checks remove unhealthy servers from rotation automatically.

**Auto scaling:** rules add or remove instances based on metrics.

```
Target tracking: keep average CPU at 50%
  CPU rises above 50%  -> add instances
  CPU stays below 50%  -> remove instances
Min 2 instances (for availability), Max 20 (to cap cost)
```

Other scaling ideas: scheduled scaling (before known peaks), queue-based scaling (scale workers by queue length), caching and CDNs (serve repeated requests without touching servers).

**Decoupling with queues:** put a message queue (SQS, Pub/Sub, Kafka) between components. The producer drops messages without waiting. Consumers work at their own pace. This smooths spikes and isolates failures.

**Common mistakes:**

- Storing state on the server, which breaks scaling.
- Scaling servers while the database remains the bottleneck.
- No upper limit on auto scaling (runaway cost).
- Slow scaling reactions: new instances take minutes to become ready.

**Practice**

- **Q: How do you make a stateful app scalable?** Move state out: sessions to a shared cache, files to object storage, data to a managed database. Then add servers behind a load balancer.
- **Q: What if a traffic spike comes faster than auto scaling reacts?** Pre-scale for known events, use queues to absorb bursts, use caching/CDN, and set sensible warm capacity.

## 9. Security: Identity, Access and the Shared Responsibility Model

### 9.1 Shared responsibility

The provider secures the cloud itself (data centers, hardware, hypervisor). **You** secure what you put in it (your data, access rules, OS patches on VMs, app code, configuration). Most cloud breaches come from **customer misconfiguration**, not provider failures.

### 9.2 Identity and Access Management (IAM)

**The idea:** control _who_ can do _what_ on _which_ resource.

- **User:** a person. **Group:** a set of users. **Role:** an identity that is _assumed temporarily_ by a user, a service, or another account (no long-lived password). **Policy:** a JSON document of permissions.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-app-reports/*"
    }
  ]
}
```

This allows reading objects in one bucket only, nothing else.

**Golden rules:**

1. **Least privilege:** grant only what is needed. Avoid `"Action": "*"` and `"Resource": "*"`.
2. **Use roles, not stored keys.** Attach a role to your VM or function so it gets short-lived credentials automatically. Never hardcode secrets in code or Git.
3. **Turn on MFA** for humans, and don't use the root account day to day.
4. **Explicit deny wins** over allow. By default, everything is denied.
5. Store secrets in a **secrets manager** and rotate them.

### 9.3 Encryption

- **In transit:** TLS/HTTPS everywhere.
- **At rest:** disk, database and bucket encryption, with keys in a **KMS** (key management service). Envelope encryption: a data key encrypts your data, and a master key encrypts the data key.

### 9.4 Defense in depth

Layers: network (private subnets, firewalls), identity (IAM), data (encryption), application (input validation), monitoring (logs, alerts), and governance (audits, policies).

**Common mistakes:** public storage buckets, over-broad IAM policies, access keys committed to GitHub, no MFA, unpatched VMs, unrestricted security groups, no logging.

**Practice**

- **Q: Why prefer IAM roles over access keys?** Roles give temporary, automatically rotated credentials. Keys are long-lived and often leaked.
- **Q: What does least privilege mean in practice?** Start with no access, then add only the specific actions and resources needed, and review regularly.

## 10. Reliability: High Availability and Disaster Recovery

**The idea:** things _will_ fail. Design so failures don't become outages.

**Key terms:**

- **Availability:** percentage of time the system works. "Nines": 99.9% allows about 8.8 hours of downtime per year, 99.99% about 53 minutes, 99.999% about 5 minutes.
- **RTO (Recovery Time Objective):** how long you can be down.
- **RPO (Recovery Point Objective):** how much data you can afford to lose (measured in time).
- **SLA / SLO / SLI:** SLA is the promise to customers (often with refunds), SLO is your internal target, SLI is the measured number.

**Disaster recovery strategies (cheapest to fastest):**

| Strategy                         | What it means                                                 | Recovery        | Cost    |
| -------------------------------- | ------------------------------------------------------------- | --------------- | ------- |
| **Backup & restore**             | Keep backups, rebuild after a disaster                        | Hours           | Lowest  |
| **Pilot light**                  | Core pieces (data) always running in another region, rest off | Tens of minutes | Low     |
| **Warm standby**                 | Scaled-down full copy always running                          | Minutes         | Medium  |
| **Active-active (multi-region)** | Full capacity in multiple regions serving live traffic        | Near zero       | Highest |

**Reliability patterns:**

- **Redundancy:** no single point of failure (multiple AZs, multiple instances).
- **Health checks and automatic failover.**
- **Retries with exponential backoff and jitter:** wait longer after each failure, with randomness to avoid everyone retrying at once.
- **Timeouts:** never wait forever.
- **Circuit breaker:** after repeated failures, stop calling the failing service for a while.
- **Idempotency:** repeating a request has the same effect as doing it once, so retries are safe.
- **Graceful degradation:** if recommendations are down, still show the product page.
- **Chaos testing:** deliberately break things in a controlled way to find weaknesses.

**Common mistakes:** backups never restored in a test, a "multi-AZ" setup with the database in one AZ, retries without backoff causing retry storms, and no runbooks.

**Practice**

- **Q: Design for 99.99% availability.** Multiple AZs for each tier, load balancer with health checks, auto scaling, multi-AZ managed database, no single point of failure, automated deploys with rollback, monitoring and alerts. Consider multi-region if the target or compliance demands it.
- **Q: RTO vs RPO example.** If a database is restored from hourly backups, RPO is up to 1 hour of lost data. RTO is how long the restore takes.

## 11. Infrastructure as Code (IaC)

**The idea:** describe your infrastructure in **text files** and let a tool create it. No clicking around consoles.

**Why:** repeatable, reviewable (Git and pull requests), testable, easy to recreate in a new region, and no "it works on my environment" differences.

**Declarative example (Terraform):**

```hcl
resource "aws_s3_bucket" "reports" {
  bucket = "my-app-reports-prod"
}

resource "aws_s3_bucket_public_access_block" "reports" {
  bucket                  = aws_s3_bucket.reports.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags          = { Name = "web-1" }
}
```

Workflow: `terraform plan` shows what _would_ change, `terraform apply` makes it happen. The **state file** records what exists, so store it remotely with locking (for example S3 plus a lock table), never only on a laptop.

**Declarative vs imperative:** declarative says _what_ you want (the tool figures out how). Imperative scripts say _how_, step by step. Declarative tools: Terraform, CloudFormation, Bicep. Pulumi lets you use real programming languages.

**Key ideas:**

- **Idempotent:** running it twice gives the same result.
- **Drift:** real infrastructure changed manually and no longer matches the code. Detect and fix it.
- **Immutable infrastructure:** don't patch running servers. Replace them with new, tested images.
- **Config management** (Ansible, Chef) configures software on servers. IaC provisions the servers themselves.

**Common mistakes:** secrets in code, state file lost or edited by hand, manual console changes (drift), and giant monolithic configurations. Split into modules.

**Practice**

- **Q: What is drift and how do you handle it?** The live environment differs from the code, usually from manual edits. Detect with plan/drift tools, then either update the code or re-apply it, and restrict console write access.

## 12. CI/CD and Deployment Strategies

**The idea:** automate building, testing and releasing so changes ship safely and often.

- **CI (Continuous Integration):** every commit is built and tested automatically.
- **Continuous Delivery:** every passing change is _ready_ to deploy (a human clicks release).
- **Continuous Deployment:** every passing change deploys automatically.

**A typical pipeline:**

```
commit -> build -> unit tests -> security scan -> package (container image)
       -> deploy to staging -> integration tests -> deploy to production -> monitor
```

**Safe release strategies:**

| Strategy          | How it works                                                | Rollback                       |
| ----------------- | ----------------------------------------------------------- | ------------------------------ |
| **Rolling**       | Replace instances a few at a time                           | Roll forward or back gradually |
| **Blue/green**    | Run new (green) beside old (blue), then switch traffic      | Instant: switch back           |
| **Canary**        | Send a small percentage of traffic to the new version first | Stop and revert quickly        |
| **Feature flags** | Ship code turned off, enable for chosen users               | Turn the flag off              |

**Database changes** are the hard part. Use backward-compatible migrations (add a column first, deploy code, remove the old column later) so old and new versions can run together.

**Practice**

- **Q: Blue/green vs canary?** Blue/green switches all traffic at once between two full environments (fast rollback, double capacity cost). Canary shifts traffic gradually and watches metrics (lower risk, more complex).

## 13. Observability: Seeing What Your System Is Doing

**The idea:** you can't fix what you can't see. Three kinds of signals:

- **Metrics:** numbers over time (CPU, request rate, error rate, latency). Cheap and great for alerts.
- **Logs:** detailed event records. Use **structured logs** (JSON) so they can be searched.
- **Traces:** follow one request across many services to see where time is spent.

**What to measure (the "golden signals"):** **latency**, **traffic**, **errors**, **saturation** (how full resources are).

**Percentiles matter:** average latency hides pain. If p50 is 100 ms but p99 is 5 s, 1 in 100 users has a terrible experience.

**Alerting rules:** alert on **symptoms users feel** (error rate, latency), not every cause. Every alert should be actionable. Too many alerts cause "alert fatigue".

```json
{
  "time": "2026-10-03T10:15:00Z",
  "level": "ERROR",
  "service": "checkout",
  "trace_id": "abc123",
  "msg": "payment failed",
  "order_id": "o-42",
  "error": "timeout"
}
```

The shared `trace_id` links logs across services.

Tools: CloudWatch, Azure Monitor, Cloud Operations, Prometheus + Grafana, OpenTelemetry (a vendor-neutral standard for collecting metrics, logs and traces).

**Practice**

- **Q: Monitoring vs observability?** Monitoring watches for known problems. Observability lets you investigate _unknown_ problems by exploring metrics, logs and traces.

## 14. Cost Management (FinOps)

**The idea:** the cloud makes spending easy, so you must manage it deliberately.

**Ways to cut cost:**

1. **Right-size:** use smaller instances if utilization is low.
2. **Turn things off:** shut down dev/test outside working hours, delete unused disks, snapshots, IPs and idle load balancers.
3. **Commit for steady loads:** reserved instances or savings plans.
4. **Use spot** for fault-tolerant batch work.
5. **Use storage tiers and lifecycle rules.**
6. **Autoscale** instead of provisioning for the peak.
7. **Watch data transfer:** data leaving the cloud (egress) and crossing regions or AZs often costs money, and it surprises many teams.
8. **Tag everything** (team, project, environment) so costs can be traced to owners.
9. **Set budgets and alerts.**

**Common mistakes:** forgotten test resources, no tags, unlimited autoscaling, logging everything forever, and cross-region traffic in chatty designs.

**Practice**

- **Q: Your bill doubled. How do you investigate?** Check the cost-explorer breakdown by service, region and tag. Look for new resources, data transfer spikes, or autoscaling runaways. Then set budgets and alerts so it doesn't repeat.

## 15. Designing Well: Principles and the Well-Architected Pillars

Cloud providers describe good design with similar pillars:

1. **Operational excellence:** automate, observe, improve.
2. **Security:** least privilege, encryption, traceability.
3. **Reliability:** recover from failure, scale to demand.
4. **Performance efficiency:** choose the right resources, measure.
5. **Cost optimization:** pay only for what delivers value.
6. **Sustainability:** use resources efficiently.

**Design rules that apply everywhere:**

- **Design for failure.** Assume every component can fail.
- **Make things stateless and loosely coupled** (queues, events, APIs).
- **Automate everything** (IaC, CI/CD, scaling, recovery).
- **Prefer managed services** over running your own, unless you need the control.
- **Secure by default:** private first, open deliberately.
- **Measure, then optimize.**

**Microservices vs monolith:** a monolith is simpler to build and run at first. Microservices let teams deploy independently and scale parts separately, but add network calls, distributed failures and operational overhead. Start simple and split when there is real pain.

**Event-driven design:** services publish events ("order placed"), and other services react. This decouples them and handles spikes, but needs care with duplicate delivery (design idempotent consumers) and ordering.

## 16. A Complete Example: Three-Tier Web App

**Requirement:** a web app that must survive a data center failure and handle traffic spikes.

```
 Users
   |
 [DNS] -> [CDN] (static files from object storage)
   |
 [Load Balancer]           public subnets, 2 AZs
   |
 [Auto Scaling Group of app servers/containers]    private subnets, 2 AZs
   |            \
 [Cache]         [Queue] -> [Worker group]  (emails, reports)
   |
 [Managed database, Multi-AZ + read replica]       private subnets
   |
 [Backups -> object storage, copied to a second region]
```

**Why each piece exists:**

- **CDN + object storage:** serves images and scripts cheaply and fast.
- **Load balancer + 2 AZs:** survives one AZ failing, and spreads load.
- **Auto scaling:** adds servers during spikes, removes them later.
- **Cache:** reduces database load and latency.
- **Queue + workers:** slow work happens in the background, so users aren't kept waiting and spikes are absorbed.
- **Multi-AZ database + replica:** failover plus read scaling.
- **Cross-region backups:** disaster recovery.
- **Private subnets + security groups + IAM roles + encryption:** security layers.
- **IaC + CI/CD + monitoring:** repeatable, safe, observable.

## 17. Quick Reference

### Service name map

| Idea                 | AWS            | Azure                       | Google Cloud                |
| -------------------- | -------------- | --------------------------- | --------------------------- |
| Virtual machines     | EC2            | Virtual Machines            | Compute Engine              |
| Serverless functions | Lambda         | Functions                   | Cloud Functions / Cloud Run |
| Managed Kubernetes   | EKS            | AKS                         | GKE                         |
| Object storage       | S3             | Blob Storage                | Cloud Storage               |
| Block storage        | EBS            | Managed Disks               | Persistent Disk             |
| Managed SQL          | RDS / Aurora   | SQL Database                | Cloud SQL / Spanner         |
| NoSQL                | DynamoDB       | Cosmos DB                   | Firestore / Bigtable        |
| Data warehouse       | Redshift       | Synapse                     | BigQuery                    |
| Virtual network      | VPC            | VNet                        | VPC                         |
| Load balancer        | ELB / ALB      | Load Balancer / App Gateway | Cloud Load Balancing        |
| DNS                  | Route 53       | Azure DNS                   | Cloud DNS                   |
| CDN                  | CloudFront     | Front Door / CDN            | Cloud CDN                   |
| Queue / messaging    | SQS / SNS      | Service Bus                 | Pub/Sub                     |
| Identity             | IAM            | Entra ID + RBAC             | IAM                         |
| Key management       | KMS            | Key Vault                   | Cloud KMS                   |
| Monitoring           | CloudWatch     | Monitor                     | Cloud Monitoring            |
| IaC (native)         | CloudFormation | Bicep / ARM                 | Deployment Manager          |

### One-line memory table

| Topic                     | Remember                                                           |
| ------------------------- | ------------------------------------------------------------------ |
| Cloud                     | Rent computing, pay for use, scale on demand                       |
| IaaS / PaaS / SaaS        | You manage less as you go from IaaS to SaaS                        |
| Region / AZ               | Region = area. AZ = isolated data center inside it. Use 2+ AZs     |
| VM / container / function | Full OS / shared kernel / just code                                |
| Storage                   | Object = files over HTTP, block = disk, file = shared drive        |
| Networking                | Public subnet for load balancer, private for app and database      |
| Scaling                   | Out beats up. Stateless apps scale easily                          |
| Queues                    | Smooth spikes and isolate failures                                 |
| Security                  | Least privilege, roles over keys, MFA, encrypt, private by default |
| Shared responsibility     | Provider secures the cloud, you secure what is in it               |
| HA vs DR                  | HA = survive small failures. DR = recover from big ones (RTO/RPO)  |
| IaC                       | Infrastructure in Git. Plan, review, apply. Avoid drift            |
| Deployments               | Blue/green = instant switch, canary = gradual                      |
| Observability             | Metrics, logs, traces. Alert on symptoms                           |
| Cost                      | Right-size, turn off, commit, tag, watch data transfer             |

## Practice round (mixed difficulty)

1. **What's the difference between scalability and elasticity?** Scalability is the ability to handle more load by adding resources. Elasticity is doing that _automatically and quickly in both directions_ to match demand.
2. **A website is slow only at peak hours. What do you check?** Metrics for CPU, memory, database connections and queue depth. Look for the real bottleneck, often the database or an external call. Then add caching, scale the bottleneck, and use autoscaling.
3. **How would you stop a leaked access key from causing damage?** Disable and rotate the key immediately, review the logs for misuse, use roles instead of keys, enforce least privilege, scan repositories for secrets, and turn on MFA and alerts.
4. **Design a file-upload service for millions of users.** Clients upload directly to object storage with short-lived signed URLs (so your servers don't carry the bytes). An event triggers a function or worker to scan and process the file. Metadata goes in a database. A CDN serves downloads. Lifecycle rules move old files to cheaper tiers.
5. **Why use a queue between the API and a worker?** To respond quickly, absorb spikes, retry failures without losing work, and let each side scale separately. Make workers idempotent, because messages can be delivered twice.
6. **What happens if your single-region app's region goes down?** Without a plan, you are down. With a DR strategy (backups, pilot light, warm standby, or active-active), you restore or fail over. The choice depends on your RTO, RPO and budget.
7. **Compare vertical and horizontal scaling for a database.** Vertical is simple but limited and often needs downtime. Horizontal needs read replicas or sharding, which add complexity (replica lag, cross-shard queries), but scales further.
8. **How do you deploy with zero downtime?** Use rolling, blue/green or canary releases behind a load balancer with health checks, backward-compatible database migrations, and fast rollback.
