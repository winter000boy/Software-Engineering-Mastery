# #016 — Paper Notes (Summary + 3 Takeaways) + Plan Next Week

**Date:** 2026-09-24  
**Roadmap row:** 16 — P1 Java Core · Java runtime & OOP foundations · *Paper*  
**Paper:** James Hamilton, *On Designing and Deploying Internet-Scale Services*, LISA '07  
**Detailed section notes:** #015

---

## At a Glance (copy to handwritten notes)

- **One-sentence summary:** operating a large service cheaply and reliably is mostly decided at **design time**. Build services that **expect failure, stay simple, and are automatable**, so that machines, not people, handle almost every failure.
- **Takeaway 1: design for failure, and prove it by failing on purpose.** If you can't kill any server at any time, you aren't redundant.
- **Takeaway 2: automation is a design constraint, not a script you add later.** Make operational decisions trivially simple (sync replication ⇒ trivial failover).
- **Takeaway 3: if you can't see it, you can't run it.** Instrument everything, keep alerts actionable, and track the fault-tolerance mechanisms that hide failures.
- **Next week:** Collections & Generics, rows **17–41**: List/Set/Map, ArrayList vs LinkedList, HashMap internals, TreeMap/LinkedHashMap, generics/PECS, Comparable/Comparator, the LRU cache project, Big-O review, and the **Google File System** paper.

---

## 1. Summary (≈ 250 words)

James Hamilton distils what Microsoft's biggest services (MSN, Windows Live, Xbox Live, Exchange Hosted Services) learned about running at internet scale. He measures operational efficiency with the **system-to-administrator ratio**, which ranges from 2:1 in manual shops to 2,500:1 in the best automated services. His central claim is that the gap comes mainly from the **service's design**, not from better ops tooling: **80% of operations problems originate in design and development**.

Three tenets run through the paper: **expect failures, keep things simple, automate everything**. From them come concrete practices in ten areas. **Design** for failure with redundancy so any server can be killed without warning, use commodity hardware, run a single version, partition finely through a look-up table, and version everything so n and n+1 coexist. **Automation:** constrain the design so decisions are simple (synchronous replication makes failover trivial), keep config with code, and deliberately fail components in production. **Dependencies:** timeouts, bounded idempotent retries, fail fast. **Releases:** ship often, stage rollouts from one server to global, always have tested roll-back, deploy mid-day. **Hardware:** a few standard SKUs behind an abstraction, and always use CNAMEs. **Operations:** developers share responsibility ("you built it, you manage it"), soft deletes, one change at a time, everything configurable. **Monitoring:** instrument everything, make every alert actionable, and track fault-tolerance events. **Overload:** admission control at every layer, a tested "big red switch" to shed non-critical work, and metered recovery. Finally, **communicate openly** with customers and press, and let customers **self-provision**.

The paper reads like a blueprint for today's DevOps, SRE, cloud and chaos engineering, written in 2007.

---

## 2. Three Takeaways

### Takeaway 1 — Design for failure, and prove it by failing on purpose

**What the paper says:** at 10,000 servers, failures are a daily event. The acid test is whether ops can **bring down any server at any time without draining it first**. Following Armando Fox's crash-only idea, **never shut down normally; hard-fail**, because failure paths that aren't exercised don't work. "Fail services regularly": take down racks and data centres.

**Why it matters to me:** most code I write treats failure as the exception path. This flips it: the failure path should be *the most-tested path*.

**How I'll apply it (concretely):**

| Where | Action |
|-------|--------|
| Now (P1) | In the CLI inventory and Library projects, every repository/service method has a test for its failure case (not found, duplicate, invalid input), not just the happy path |
| P2–P3 | Spring services: graceful shutdown **and** a test that kills the process mid-request and checks that no data is corrupted |
| P5 (rows 417–441) | Add fault injection (e.g. a dependency that fails 10% of calls) and verify circuit breakers and retries |
| P6 | A "game day" in the capstone: kill pods under load and observe |

### Takeaway 2 — Automation is a design constraint, not an afterthought

**What the paper says:** people under pressure make mistakes **~20%** of the time, so recovery must not depend on them. But you can only automate what's **simple to decide**. Async replication forces a human judgement call (lose data or stay down?). **Paying for synchronous replication** turns failover into "primary down → use secondary". The most automated services cost **two orders of magnitude** less in people than the most manual ones.

**Why it matters to me:** "we'll write a script later" rarely works. When I design something, I should ask *"what will operations have to decide when this breaks, and can a machine make that decision?"*

**How I'll apply it:**

- In every system-design write-up (P4 onwards), add a section: **"Failure modes & automated responses"**, with a table of failure → detection → automatic action → human fallback.
- Treat **configuration and code as one deployable unit** (P6 CI/CD), and keep scripts and ops tooling in the same repo, reviewed and tested (the paper: "treat operations utilities as part of the service").
- Prefer designs where recovery is **idempotent and restartable** (rows 429–430: idempotency keys, compensating transactions).

### Takeaway 3 — If you can't see it, you can't run it: instrument and alert with intent

**What the paper says:** ops can't add instrumentation after deployment. **Instrument everything**, collect *actual numbers*, and make every alert **actionable**. Tune alerts with two metrics: alerts-to-tickets ≈ **1**, and missed incidents ≈ **0**. **Track fault-tolerance mechanisms**, because retries and failovers *hide* failures: a 2,000-machine service eroded to 400 before anyone noticed. Error messages should state probable causes and fixes.

**Why it matters to me:** logging is usually the last thing added and the first thing that's missing in an incident.

**How I'll apply it:**

- From P2 onwards, every service gets structured logs, request latency + throughput metrics, and a health endpoint **in its first version** (the "skeleton" idea from the paper).
- Count **retries, fallbacks and circuit-breaker openings** as metrics, not just log lines (P5, rows 467–491).
- Write error messages with the fix in them: `"Product not found: id=42 — check the id with 'list'"` (already practised in the CLI inventory project's error handling).

---

## 3. Connecting the Paper to This Week's Java Work

The paper is about systems of 10,000 servers, but the same ideas apply to single-JVM code:

| Paper principle | Java-level equivalent from rows 1–13 |
|-----------------|--------------------------------------|
| Zero trust of underlying components | Validate inputs at boundaries; `Objects.requireNonNull`; value objects that can't be invalid (`Isbn.of`, #005) |
| Keep things simple and robust | Immutable objects (EJ Item 17): one state, thread-safe for free |
| Version everything, n/n+1 coexist | Program to interfaces (`List`, `MembershipPolicy`) so implementations can change |
| Dependencies need timeouts | Later: `CompletableFuture.orTimeout` (row 70), HTTP client timeouts |
| Don't build the same functionality twice | A single `Library` service owns the rules; no duplicate checks in the CLI |
| Inject what varies | Injected `Clock` (#005), constructor DI (EJ Item 5) |
| Safe under concurrent change | The JMM: happens-before, safe publication (#012–#013) |

---

## 4. Open Questions (to revisit later)

1. How does synchronous replication's latency cost change across data centres? (→ Spanner paper, row 290; CAP/PACELC, rows 345, 362)
2. Hamilton favours a partition look-up table over static hashing. When is **consistent hashing** the better choice? (→ Karger paper, row 340)
3. How do modern SLO/error-budget practices formalise his "tune goals in real time"? (→ SRE book, rows 467–491)

---

## 5. Plan for Next Week — Collections & Generics (rows 17–41)

**Theme:** know the JDK collections well enough to (a) pick the right one instantly, (b) explain its internals and Big-O, (c) build simplified versions yourself, and (d) type them safely with generics.

**Primary resources:** *Effective Java* (Ch. 5 Generics, plus Items 54, 55, 58, 64), the `java.util` Javadoc and the OpenJDK source (`ArrayList.java`, `HashMap.java`).

### Day-by-day

| Day | Rows | Learn | Build | Flex |
|-----|------|-------|-------|------|
| 1 | 17–19 | #017 List/Set/Map internals & when to use each | #018 Benchmark ArrayList vs LinkedList (JMH) | #019 Read EJ + Java docs, take notes |
| 2 | 20–22 | #020 ArrayList vs LinkedList; HashMap internals | Implement your own dynamic array (row 21) | 2–3 Collections exercises (row 22) |
| 3 | 23–25 | TreeMap / LinkedHashMap; iteration order (row 23) | Implement a simplified HashMap (row 24) | Review + refactor yesterday's build (row 25) |
| 4 | 26–28 | Generics, bounded types, wildcards / PECS (row 26) | Sort objects with custom comparators (row 27) | Anki cards for Collections (row 28) |
| 5 | 29–31 | Comparable vs Comparator (row 29) | Write a generic container class (row 30) | Gap-fill the weakest topic (row 31) |
| 6–10 | 32–36 | — | **Project: LRU cache** from scratch (`LinkedHashMap` version + manual `HashMap` + doubly-linked list version), with tests & benchmark: plan → build → build → test/refactor → document | — |
| 11 | 37–38 | Study + apply: Big-O review + collections cheat sheet | | |
| 12 | 39 | Weekly review + Anki; fix gaps | | |
| 13 | 40–41 | Paper: *The Google File System* (Ghemawat, Gobioff, Leung, 2003) + notes/takeaways + plan week 3 (Concurrency I) | | |

### Goals / definition of done for the week

- [ ] Can draw `HashMap`'s table → bucket → node/tree structure and explain `hash()`, index calculation, resize and treeify **from memory**
- [ ] Can state the Big-O of get/add/remove/contains for `ArrayList`, `LinkedList`, `ArrayDeque`, `HashMap`, `TreeMap`, `HashSet`, `PriorityQueue`
- [ ] Have **measured** (not guessed) ArrayList vs LinkedList with JMH and can explain the results with cache locality
- [ ] Own dynamic array and simplified HashMap pass tests, including resize and collision cases
- [ ] Can explain PECS with `Collections.copy(List<? super T>, List<? extends T>)`
- [ ] LRU cache: both implementations, tests green, benchmark written up
- [ ] Rows 17–41 updated to `TRUE` in `Roadmap.csv` as each one is done

### Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Getting lost in the `HashMap` source (it's 2,500 lines) | Read only `hash()`, `putVal()`, `resize()`, `treeifyBin()`, `getNode()`; skip the TreeNode internals |
| JMH setup eats the day | Use the ready-made Maven setup from #018's solution |
| The LRU project over-runs | Days 6–10 are timeboxed; the manual version is MVP-first (get/put/evict), stretch goals only after tests pass |

### Carry-over from Week 1

- Any self-test misses from #014 go into Anki before day 1.
- Re-do #013 Part D drills on day 3 (the +3-day spaced repetition from #014).

---

## Handwritten Cheat Sheet

```
HAMILTON IN ONE LINE: ops cost & reliability are designed in, not bolted on.
3 TAKEAWAYS:
 1. Design for failure + prove it (kill any server anytime; hard-fail; chaos)
 2. Automation = design constraint (make decisions trivial: sync repl → easy failover)
 3. Observe everything (actionable alerts, alerts:tickets≈1, track retries/failovers)

WEEK 2 (rows 17–41) COLLECTIONS & GENERICS
 D1 List/Set/Map · JMH ArrayList vs LinkedList · EJ generics
 D2 HashMap internals · own dynamic array
 D3 TreeMap/LinkedHashMap · own HashMap
 D4 Generics/PECS · comparators
 D5 Comparable vs Comparator · generic container
 D6–10 LRU cache project (LinkedHashMap + manual)
 D11 Big-O  D12 review  D13 GFS paper
```

---

**Next note:** #017 — List / Set / Map internals & when to use each
