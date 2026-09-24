# #015 — Research Paper: *On Designing and Deploying Internet-Scale Services* (J. Hamilton)

**Date:** 2026-09-24  
**Roadmap row:** 15 — P1 Java Core · Java runtime & OOP foundations · *Paper*  
**Paper:** James Hamilton (Windows Live Services Platform, Microsoft), *LISA '07: 21st Large Installation System Administration Conference*, 2007, pp. 231–242  
**PDF in repo:** [`Reaseach Papers/1.On Designing and Deploying.pdf`](../Reaseach%20Papers/1.On%20Designing%20and%20Deploying.pdf)

> These are section-by-section study notes, following the paper's own structure. The distilled summary, 3 takeaways and next week's plan are in **#016**.

---

## At a Glance (copy to handwritten notes)

- **Thesis:** the cost and reliability of running a big service are decided mostly by **how the service is designed**, not by the operations team. Build **"operations-friendly"** services that detect and recover from almost all failures **without a human**.
- **Metric:** the *system-to-administrator ratio*, from **2:1** for small manual services to **2,500:1** for the best automated ones.
- **Three tenets (Bill Hoffman):** **Expect failures · Keep things simple · Automate everything.**
- **80% of operations issues originate in design and development.**
- **Design basics:** design for failure; redundancy + fault recovery; commodity hardware; **single-version** software; multi-tenancy.
- **Acid test:** can ops take down **any** server at **any** time **without draining it first**?
- **Automation needs a constrained design:** e.g. pay for synchronous replication so that failover is a trivial decision. Humans under pressure make mistakes **~20%** of the time.
- **Release:** ship often (≤ 3 months, trending weekly); staged rollout; **always** have tested roll-back; deploy **mid-day**, not at night.
- **Ops:** soft delete only; one change at a time; make everything configurable; **"you built it, you manage it"**.
- **Monitoring:** alerts must be **actionable**. Target an alerts-to-trouble-ticket ratio ≈ 1. Track fault-tolerance mechanisms, because they *hide* failures.
- **Overload:** **admission control** at every level, a **"big red switch"** to shed non-critical load, and **metered** ramp-up after recovery.

---

## 1. Context & Motivation

- Written in 2007 from ~20 years of experience with high-scale data systems. Most recently Hamilton led **Exchange Hosted Services** (~700 servers, 2.2M users); contributors came from Windows Live Search, Hotmail, Xbox Live, MSN, Messenger and others, some with **> 250M users**.
- Builds on **Recovery-Oriented Computing** (Berkeley, Patterson) and **Crash-Only Software** (Stanford, Fox).
- Microsoft's **Autopilot** (automatic data-centre management) is often credited for Windows Live Search's high automation, but Hamilton's point is that **auto-administration only works if the service itself is efficient to automate.**

**Definition: an operations-friendly service** needs little human intervention and **detects and recovers from all but the most obscure failures without administrative intervention.**

### The three tenets (repeated throughout)

| Tenet | Meaning in practice |
|-------|--------------------|
| **Expect failures** | Any component, or a dependency, can crash or stop at any time. Networks fail, disks fill up. Handle all of it gracefully |
| **Keep things simple** | Complexity breeds problems. Avoid unnecessary dependencies; simple installation; a failure on one server must not affect the rest of the data centre |
| **Automate everything** | People make mistakes, need sleep and forget things. Automated processes are testable, fixable and ultimately more reliable |

The paper then gives best practices in **10 areas**, covered in §2–§11 below.

---

## 2. Overall Application Design (the largest and most important section)

> *"We have long believed that 80% of operations issues originate in design and development."*

When systems fail, people instinctively blame operations, because that's where the failure *happened*. But the fix almost always belongs in design and development. Low administration cost **correlates with how closely dev, test and ops work together** (an early statement of what became **DevOps**).

### 2.1 Five basic design tenets

| Tenet | Detail |
|-------|--------|
| **Design for failure** | At 10,000 servers / 50,000 disks, failures happen **multiple times a day**. If a hardware failure needs *immediate* human action, the service can't scale. The recovery path must be simple and **tested frequently**. Armando Fox: **never shut the service down normally; just hard-fail it**, because failure paths that aren't exercised won't work when needed (**crash-only software**) |
| **Redundancy & fault recovery** | Mainframes (one huge redundant box) are expensive and still not reliable enough. The fifth 9 needs **redundancy across machines**. **Acid test:** *is ops willing and able to bring down any server at any time without draining the workload first?* If yes, you have synchronous redundancy (no data loss), failure detection and automatic take-over. Apply **security threat modelling** to failures: list every failure mode *and combination*, and either survive it or explicitly accept the risk. **"Rare combinations can become commonplace"** at scale |
| **Commodity hardware slice** | Many cheap servers beat a few big ones: (1) cheaper, (2) CPU speed grows faster than I/O, so small servers are better balanced, (3) power scales linearly with server count but **cubically with clock frequency**, (4) a small server that fails takes out a smaller share of the load |
| **Single-version software** | A service only targets **one** internal deployment and doesn't support old versions for a decade (unlike enterprise products). Don't let customers choose versions. That needs care not to make big UX changes between releases |
| **Multi-tenancy** | Host all customers in the same service without physical isolation; same cost argument as single-version (automation + scale) |

These are **deliberate constraints** on the design, *to make automation possible and reduce cost*. That's the opposite of the IT-outsourcing model (people-intensive, customer-specific configurations).

### 2.2 Specific best practices

| Practice | What it says | Modern equivalent |
|----------|--------------|-------------------|
| **Quick service health check** | A fast "sniff test" (build verification test) a developer can run locally before check-in | Smoke tests in CI, pre-commit checks |
| **Develop in the full environment** | Unit tests **plus** testing the full service with your change. Needs single-server deployment (§5) | `docker compose up`, Testcontainers |
| **Zero trust of underlying components** | Assume dependencies fail; keep serving, e.g. from **cached data in read-only mode**, or for all but a tiny fraction of users while failing over | Graceful degradation, stale-while-revalidate |
| **Don't build the same functionality in multiple components** | Duplicate code means fixes are needed in several places, and the code base deteriorates fast | DRY at the service level; shared libraries/platforms |
| **One pod/cluster must not affect another** | Pods should be ~100% independent with no correlated failures. Global services are a central point of failure; keep what a cluster needs *inside* it | Cells / bulkheads / cell-based architecture |
| **Allow (rare) emergency human intervention** | Design to need none, but plan for it. **Operator error at 2 a.m. is a common cause of catastrophic data loss.** Write recovery procedures as **tested scripts**, not multi-step docs, and hold **"fire drills"** in production | Runbooks-as-code, game days |
| **Keep things simple and robust** | "Simple and nearly stupid is almost always better." Only take optimisations that give an **order-of-magnitude** gain; percentage gains aren't worth the extra failure modes | YAGNI; boring technology |
| **Enforce admission control at all levels** | Better to refuse work than to accept it and **thrash**. Needed at the front door *and* at every major component boundary. Degrade gracefully; block entry before giving uniformly poor service to everyone | Rate limiting, load shedding, backpressure |
| **Partition the service** | Partitions must be **fine-grained and infinitely adjustable**, **not tied to real-world entities** (a big company outgrows a per-company partition; "all the P's" outgrow a name-prefix partition). Use a **mid-tier look-up table** mapping fine-grained entities (users) to servers, so partitions move freely | Directory-based sharding; virtual nodes (→ consistent hashing, row 329) |
| **Understand the network design** | Know the traffic within a rack, across racks and across data centres; review it early with networking specialists | Topology-aware placement |
| **Analyse throughput and latency** | Measure core interactions *while* maintenance jobs run. Derive a **capacity metric** (e.g. requests/sec per server) | Load testing, capacity models |
| **Treat operations utilities as part of the service** | Ops tools must be code-reviewed, checked in and tested like product code. They're often mission-critical yet untested | Platform/tooling as product |
| **Understand access patterns** | Every feature spec should answer: *"What impacts will this feature have on the rest of the infrastructure?"* Then measure it when it goes live | Design-doc "load impact" section |
| **Version everything** | Expect a **mixed-version** environment during rollout: versions **n and n+1 must coexist** | Backward-compatible APIs and schemas, expand/contract migrations |
| **Keep the unit/functional tests from the previous release** | Verify n-1 functionality still works; also run **service verification tests in production** continuously | Regression suites, synthetic monitoring |
| **Avoid single points of failure** | Prefer **stateless** implementations. **Don't affinitise** clients to servers; load-balance over interchangeable servers. **Static hashing suffers data/query skew.** Databases are the classic SPOF: fine-grained partitioning, **no cross-partition operations**, a redundant hot standby, and failover **tested frequently in production** | Stateless services + LB, DB replicas |

---

## 3. Automatic Management and Provisioning

**Problem:** alert-a-human-and-wait means an expensive 24×7 staff, and *"if operations engineers are asked to make tough decisions under pressure, about **20%** of the time they will make mistakes."*

**Key insight: designing for automation constrains the design.**

> Example: asynchronous replication to a secondary database. When the primary dies, failing over **loses recent data**, but not failing over means **downtime**. That trade-off needs human judgement, so it **can't be automated**. If you instead **pay the latency and throughput cost of synchronous replication**, failover becomes trivial ("primary down → route to secondary") and **fully automatable**.

The spread in people cost between the most manual and the most automated services Hamilton saw: **two orders of magnitude**.

| Practice | Detail |
|----------|--------|
| **Be restartable and redundant** | All operations restartable; all persistent state stored redundantly |
| **Support geo-distribution** | Run across several data centres; without it you can't move load to free capacity elsewhere |
| **Automatic provisioning and installation** | Manual installs are costly and error-prone, and small config differences **spread and make diagnosis harder** (configuration drift) |
| **Configuration and code as a unit** | Dev ships code + config together, test deploys it exactly as ops will, ops deploys it as a unit. Any production config change gets an **audit log** entry (what, when, who, which servers). **Regularly scan servers** to check actual state = intended state |
| **Manage server roles/personalities, not servers** | Each role deploys onto as many or as few servers as needed ("cattle, not pets") |
| **Multi-system failures are common** | Power, network switches and rollouts take down many hosts at once, so stateful services must be **topology-aware** |
| **Recover at the service level** | Handle failures where the full context exists (the service), not in lower layers |
| **Never rely on local storage for non-recoverable information** | Replicate all non-ephemeral state |
| **Keep deployment simple** | **File copy** is ideal. Minimise external dependencies and complex install scripts. Different versions of a component should be able to run on the same server |
| **Fail services regularly** | Take down data centres, racks and servers. *"Those unwilling to test in production aren't yet confident that the service will continue operating through failures."* |

(The last row is essentially **chaos engineering**, written years before Netflix popularised it with Chaos Monkey.)

---

## 4. Dependency Management

**Rule:** depending on *small* components/services rarely saves enough to justify the complexity. Dependencies make sense when:

1. the dependency is **substantial** in size or complexity (storage, consensus implementations), or
2. its value comes from being a **single central instance** (identity, group management).

| Practice | Detail | Links to roadmap |
|----------|--------|------------------|
| **Expect latency** | Every call gets a **timeout**. Use **operational idempotency** so requests can be retried after a timeout even if they partly completed. **Report and bound retries**, so a failing request can't eat ever more resources | Timeouts/retries/idempotency (rows 417, 429) |
| **Isolate failures** | Prevent cascading failures. **Fail fast**: mark a failed dependency as down and stop calling it, so threads aren't tied up waiting | **Circuit breaker** (row 420) |
| **Use shipping and proven components** | Stable, proven software and hardware beat the bleeding edge | |
| **Implement inter-service monitoring and alerting** | Know when you're overloading a dependency; back off automatically or alert; have engineering contacts on both teams | |
| **Dependent services need the same design point** | A dependency must commit to **at least the same SLA** as you | |
| **Decouple components** | Keep running in degraded mode when others fail. Example: keep a **session key refreshed every N hours** instead of re-authenticating on each connect, so a network blip doesn't create a **login storm** | Bulkheads, caching |

---

## 5. Release Cycle and Testing

**Testing in production is a reality.** Test labs are never full-fidelity, and as fidelity approaches production, cost approaches production cost.

So: unit → functional → test lab → **limited production** as the final test phase, under four rules:

1. enough **redundancy** to recover state quickly if the new release fails catastrophically
2. **data corruption** must be extremely unlikely (functional tests pass first)
3. errors are detected, and **the engineering team, not ops, monitors** the code under test
4. **fast roll-back** of all changes, and the roll-back itself is **tested before** going into production

**Staged rollout:** one server in one data centre for a few days → one server in each data centre → a whole data centre → global, if quality and performance goals are met. **Big-bang deployments are very dangerous.** (This is what we now call **canary releases**.)

**Deploy mid-day, not at night:** at night there's a greater risk of mistakes and fewer engineers around.

| Practice | Detail |
|----------|--------|
| **Ship often** | More frequent releases mean smaller changes and higher quality. The **acid test:** user experience may change, but operational issues (availability, latency) **should not** change during the release cycle. He favours **≤ 3-month** cycles and predicts weekly; **> 3 months is dangerous** |
| **Use production data to find problems** | *"Quality assurance in a large-scale system is a **data-mining and visualisation problem**, not a testing problem."* Measurable release criteria; **tune goals in real time** (start with an acceptable target and ratchet it up); collect the **actual numbers**, not red/green summaries; **minimise false positives** (hiding real problems as collateral damage is often acceptable); analyse **trends**; make **health highly visible**; **monitor continuously**, as someone's explicit job |
| **Invest in engineering** | Don't just grow the ops team to deal with scale |
| **Support version roll-back** | **Mandatory**, tested and proven before roll-out. "A rip cord that should always be available" |
| **Maintain forward and backward compatibility** | File formats, interfaces, logging and monitoring are all risks. Don't remove old-format support until roll-back to it is impossible |
| **Single-server deployment** | The whole service must run on **one machine** for dev and test. Write an **emulator** for anything that can't. Otherwise developers take a component view, not a systems view |
| **Stress test for load** | Run a small subset of production at **2×** load to understand behaviour above expected load |
| **Capacity & performance testing before each release** | At the service level and per component, since workloads change |
| **Build and deploy shallowly and iteratively** | Get a **skeleton of the full service** up early, even if it hardly does anything (a **walking skeleton**) |
| **Test with real data** | Fork production traffic and data into test, but keep privacy commitments and never let the data leak back |
| **Run system-level acceptance tests** | Keep local tests at the system level to avoid heavy maintenance costs |
| **Test and develop in full environments** | Use the same data-collection tooling as production |

---

## 6. Hardware Selection and Standardization

The bigger win of standard hardware isn't bulk discounts. It's **speed**: a **"services fabric"** with a few SKUs plus shared automated provisioning, so a new or growing service gets machines through a **web service request** instead of a month-long procurement. (This is a description of the internal **cloud** before public cloud was mainstream; Hamilton later became a VP/Distinguished Engineer at AWS.)

| Practice | Detail |
|----------|--------|
| **Use only standard SKUs** | Resources move fluidly between services |
| **Purchase full racks** | Racking and stacking in the data centre is inexplicably expensive; let the vendor do it |
| **Write to a hardware abstraction** | Describe hardware virtually (CPUs, disks, minimum memory); **don't exploit** SKU-specific details, so hardware can be upgraded |
| **Abstract the network and naming** | *"Always, always use a CNAME."* Never hard-code machine names. Keep the DNS TTL low enough for fast changes |

---

## 7. Operations and Capacity Planning

**Goal:** a highly reliable **24×7 service run by a small 8×5 operations staff.**

- Anticipate what ops will need to do in emergencies and **automate and test** those actions up-front. Build **"recovery kernels"**: small, tested recovery actions that can be combined for broad classes of failure.
- *"Nothing works if it isn't tested frequently, so don't implement anything the team doesn't have the courage to use."* A small disaster often becomes a big one because a recovery step doesn't work as expected.

| Practice | Detail |
|----------|--------|
| **Make the development team responsible** | Amazon: **"you built it, you manage it."** If dev gets paged at night, **automation** follows; if ops gets paged, the ops team grows |
| **Soft delete only** | Never delete; **mark** as deleted. Keep a rolling **≥ 2-week history** of changes. The forgotten `WHERE` clause on a `DELETE` deletes **all logical copies**, and **RAID and mirroring can't protect against that** |
| **Track resource allocation** | Have a load metric (concurrent users, requests/sec) with a **known correlation to hardware**. Sales/marketing forecasts feed capacity planning (e.g. forecasts every 90 days, capacity plan and orders every 30) |
| **Make one change at a time** | When in trouble; otherwise cause and effect can't be correlated |
| **Make everything configurable** | Anything that might need to change in production should be **tunable without a code change**. A config change is much faster and safer than code → compile → test → deploy in an incident. (Test with the production configuration, and don't turn knobs casually) |

---

## 8. Auditing, Monitoring and Alerting

- **Ops can't instrument a service after deployment.** Every component must emit performance, health and throughput data from day one.
- **Audit every configuration change** (what, who, when). The first question in any incident is *"what changed recently?"*, and without an audit trail the answer is always "nothing".
- **"Alerting is an art."** Version-one services over-alert and people learn to ignore alerts. Every alert must represent a **real problem**.

**Two metrics for tuning alerting:**

1. **alerts-to-trouble-ticket ratio**, goal ≈ **1**
2. **number of system health issues *without* a corresponding alert**, goal ≈ **0**

| Practice | Detail |
|----------|--------|
| **Instrument everything** | Measure every customer interaction. "Runners" (synthetic user workloads) alone aren't enough; it can take days to notice a serious problem that way |
| **Data is the most valuable asset** | Understand normal behaviour, or you can't recognise abnormal. Many services learned of catastrophic failures "only when the phones started ringing" |
| **Have a customer view of service** | End-to-end tests; runners must cover critical paths (e.g. new-user login). A runner failure that isn't important should be changed until it is |
| **Instrument for production testing** | Safe production testing requires complete monitoring |
| **Latencies are the toughest problem** | Slow I/O and "not quite failing but slow" are the hardest to detect |
| **Have sufficient production data** | Performance counters (latency + ops/sec) for all operations; **audit all operations** (and don't let everyone share one admin account); **track all fault-tolerance mechanisms**, because they hide failures (*a 2,000-machine service slowly fell to **400** available machines over a few days without being noticed*); audit logs per important entity (**data lineage**); **asserts** everywhere, with the results mined; keep **historical data** for trending |
| **Configurable logging** | Turn extra logging on or off at run time; deploying a special build during a failure is dangerous |
| **Expose health information** | Make external monitoring easy (→ health/readiness endpoints, row 476) |
| **Make all reported errors actionable** | An error message should state **possible causes and suggest fixes**; un-actionable errors get ignored |
| **Enable quick diagnosis** | Give enough information: not "10 queries returned no results" but "…and here is the list, and the times". Keep a **chain of evidence** (logs from start to end). Prefer **snapshotting / memory dumps shipped out of production** over debugging live boxes. **Record all significant actions** |

---

## 9. Graceful Degradation and Admission Control

Motivation: sudden spikes (DoS, usage changes). On **9/11** most news sites melted down completely. **Reliably serving a subset of articles** would have been better.

| Mechanism | Detail |
|-----------|--------|
| **"Big red switch"** (from Windows Live Search) | A **designed and tested** action to **shed non-critical load** when the SLA can't be met, or soon won't be. Candidates: work that can be **queued and processed later**, or non-essential features (e.g. advanced querying) that can be disabled while core transactions continue. It must be **reversible**, and resetting it (bringing batch jobs etc. back) must also be tested |
| **Control admission** | If current load can't be processed, taking in more work just gives **more users** a bad experience. Hamilton's email service: when over capacity, **stop accepting mail and let it queue at the source**, which *increased* throughput and *decreased* overall latency, because processing slowed down as queues grew. You can also prioritise (premium over free, known users over guests) |
| **Meter admission** | After an outage, **ramp up slowly**: 1 user, then 10 users/s, and so on, with a fine-grained knob. Clients must be told the service is down and when to retry. Add **jitter** so clients don't all retry in sync (a **thundering herd**) |

(Links to the rate-limiter design in rows 382–386 and retries with backoff in row 417.)

---

## 10. Customer and Press Communication Plan

- Communicate through several **opt-in** channels (RSS, web, IM, email, **through the client**). A client can be told to **back off** or to run in disconnected/cached mode, and can show status and an ETA.
- **Transparency increases satisfaction:** users who know what's happening and when it'll be back are less likely to leave, even for free services. (That's today's public **status pages**.)
- Have a **press plan** prepared in advance for mass data loss, security breaches, privacy violations and long outages: who to call, when, and how.

## 11. Customer Self-Provisioning and Self-Help

Let customers sign up and start using the service on the web, with no call queue. It's cheaper **and** customers are happier.

## 12. Conclusion (paper)

Reducing ops costs and improving reliability **starts with writing the service to be operations-friendly**. The paper defines the term and collects the best practices across design, development, deployment and operation.

---

## 13. How This Paper Maps to the Rest of the Roadmap

| Paper idea | Where we study it |
|------------|-------------------|
| Design for failure, redundancy, no SPOF | P4 replication (rows 342–366), P5 resilience (rows 417–441) |
| Partitioning via a look-up table; static hashing skew | Sharding & consistent hashing (rows 317–341) |
| Timeouts, retries, idempotency, fail fast | Resilience4j, circuit breaker, idempotency keys (rows 417–430) |
| Admission control, big red switch | Rate limiter design (rows 382–386) |
| Version everything, n/n+1 coexist | API versioning (row 104), Flyway migrations (row 270) |
| Single-server deployment, full environment | Docker/compose (rows 492–516), Testcontainers (row 176) |
| Staged rollout, roll-back, ship often | CI/CD (rows 517–541) |
| Instrument everything, actionable alerts | Observability, SLI/SLO (rows 467–491) |
| Soft delete, audit logs | Data layer design (P3) |
| Keep things simple | Every design write-up: justify complexity |

### What has aged, and what hasn't (2007 → today)

| Still true | Changed since 2007 |
|------------|--------------------|
| Design for failure, automate, keep it simple | "Commodity hardware slice" is now **cloud instances / Kubernetes nodes**; you rarely buy racks |
| Admission control, graceful degradation | "Ship on ≤ 3-month cycles" is now **continuous deployment**, many times a day |
| Actionable alerts, measure real numbers | "RSS" for status updates is now status pages + social media |
| Test in production (carefully), canaries | Chaos engineering, feature flags and SRE (SLOs/error budgets) are now formal disciplines built on these ideas |
| "You build it, you run it" | Now mainstream DevOps/SRE culture |

---

## 14. Glossary

| Term | Meaning |
|------|---------|
| **Operations-friendly** | Needs little human intervention; detects and recovers from almost all failures itself |
| **System-to-administrator ratio** | Servers per admin; a rough measure of operational efficiency |
| **Crash-only software** | Software whose *only* way to stop is to crash and whose *only* way to start is recovery, so the recovery path is always exercised |
| **SKU** | Stock-keeping unit, here a standard server hardware configuration |
| **Multi-tenancy** | Many customers share one service instance without physical isolation |
| **Pod / cluster** | An independent group of servers serving part of the load |
| **Admission control** | Refusing or delaying new work when the system can't handle it |
| **Big red switch** | A pre-built, tested, reversible switch to shed non-critical load |
| **Runner** | A synthetic workload simulating user actions in production |
| **Soft delete** | Marking data as deleted instead of physically removing it |

---

## 15. Discussion Questions (answer in #016 or your notebook)

1. Why does Hamilton say the *service design*, not the ops tooling, is the most important factor in automation? Give one example from the paper.
2. The paper recommends paying the cost of **synchronous** replication. When would you still choose asynchronous replication, and what would you have to add to stay automatable?
3. "Never shut the service down normally, just hard-fail it." How would you apply that to a Spring Boot service on Kubernetes?
4. Why are partitions keyed to real-world entities (companies, name prefixes) a bad idea? What does the look-up table buy you, and what does it cost?
5. Why does accepting less work (admission control) make **throughput go up**?
6. Fault-tolerance mechanisms "hide failures". How would you detect the 2,000 → 400 machine erosion?
7. Which of the practices can a *single developer* apply today, in the CLI inventory or Library project?

---

## Handwritten Cheat Sheet

```
HAMILTON 2007 — OPERATIONS-FRIENDLY SERVICES
Ratio: 2:1 (manual) … 2,500:1 (automated)        80% of ops issues = design/dev
TENETS: expect failures · keep it simple · automate everything
DESIGN: design for failure (hard-fail, crash-only) · redundancy
        ACID TEST: kill ANY server ANY time, no drain
        commodity HW · single version · multi-tenant
        zero trust of deps · pods independent · admission control everywhere
        partition fine-grained via look-up table (not by company/name)
        version everything (n & n+1 coexist) · no SPOF, stateless, no affinity
AUTOMATION: humans err ~20% under pressure → design so decisions are trivial
        (sync replication ⇒ trivial failover) · config+code as a unit + audit
        roles not servers · fail services regularly (chaos)
DEPS:   timeouts · idempotent retries (bounded) · fail fast · same SLA · decouple
RELEASE: staged rollout (1 box → 1/DC → 1 DC → global) · tested roll-back
        ship often · deploy mid-day · single-server deploy · 2× load test
OPS:    you build it, you manage it · SOFT DELETE · one change at a time
        make everything configurable · recovery scripts tested in prod
ALERTS: actionable only · alerts:tickets ≈ 1 · missed issues ≈ 0
        track fault-tolerance (2,000 → 400 unnoticed!) · latency = hardest
OVERLOAD: big red switch (shed non-critical, reversible) · admission control
        meter ramp-up + client back-off + jitter
COMMS:  status to customers + press plan · self-service signup
```

---

**Next note:** #016 — Paper notes (summary + 3 takeaways) + plan next week
