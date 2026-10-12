# #041 — Paper Notes (Summary + 3 Takeaways) + Plan Next Week

**Date:** 2026-10-12  
**Roadmap row:** 41 — P1 Java Core · Collections & Generics · *Paper*  
**Paper:** Ghemawat, Gobioff, Leung, *The Google File System*, SOSP '03  
**Detailed section notes:** #040

---

## At a Glance (copy to handwritten notes)

- **One-sentence summary:** GFS stores huge, append-mostly files on thousands of unreliable commodity machines by **separating metadata (one in-memory master) from data (replicated 64 MB chunks on chunkservers)**, **relaxing consistency** to what its applications actually need, and making **failure recovery automatic and routine**.
- **Takeaway 1: design for *your* workload.** GFS breaks POSIX assumptions on purpose (64 MB chunks, no data caching, append-oriented, relaxed consistency) because Google's access patterns justify it.
- **Takeaway 2: keep the central coordinator off the data path.** One master is simple and powerful *if* clients only ask it for metadata, cache the answers, and move data directly. Leases delegate authority to primaries.
- **Takeaway 3: weaker guarantees + application-level checks beat expensive global guarantees.** At-least-once record append + checksums + unique record ids give correctness, at scale, without distributed transactions.
- **Next week:** Concurrency & Multithreading I, **rows 42–66**: threads → `synchronized` → wait/notify → `volatile`/JMM → races → deadlock → a **bounded blocking queue project** → JMM deep dive → the **MapReduce** paper (which ran on GFS).

---

## 1. Summary (≈ 250 words)

Google built GFS because its storage needs didn't fit traditional file-system assumptions. **Components fail constantly** (thousands of cheap disks and machines), **files are huge** (multi-GB), **writes are mostly appends**, and **reads are mostly large and sequential**. The design follows directly from those observations.

A cluster has **one master** that keeps all metadata **in memory**: the namespace, the file→chunk mapping (both persisted through a replicated **operation log** with checkpoints), and the chunk locations (**not** persisted, rebuilt from chunkserver reports). File data is split into **64 MB chunks**, each **replicated three times** across racks on **chunkservers**. Clients ask the master *where* data lives, **cache** the answer, and then read and write **directly** with chunkservers, so the master never touches file data.

Writes are ordered by a **primary** replica holding a **60-second lease**; data is **pipelined along a chain** of chunkservers, separately from control messages. Consistency is **relaxed**: concurrent writes can leave regions consistent but *undefined*, and **record append** is atomic but **at-least-once**, so applications append, checkpoint, checksum their records, and remove duplicates by id.

Fault tolerance is pervasive: **crash-only** servers that restart in seconds, prioritised **re-replication**, **shadow masters**, **chunk version numbers** against stale replicas, **32-bit checksums per 64 KB block**, and **lazy garbage collection** (deleted files are recoverable for 3 days).

The measured clusters show a master with only **tens of MB of metadata** handling **hundreds of operations per second**, and re-replicating a lost server's 600 GB in **23 minutes**.

---

## 2. Three Takeaways

### Takeaway 1 — Design for your workload, not for generality

**What the paper does:** it rejects defaults that general-purpose file systems hold sacred: small blocks (→ 64 MB chunks), client data caching (→ none: streaming workloads don't reuse data), POSIX semantics (→ its own API with snapshot and record append), strong consistency for concurrent writes (→ consistent-but-undefined).

**Why it matters to me:** every data structure choice this roadmap measured was workload-driven too: ArrayList vs LinkedList (#018), LRU size vs key skew (#035: 1% capacity gave a 51% hit rate under Zipf traffic but 1% under uniform), BLOCK vs DYNAMIC partitioning (#043). The right design is a function of **access patterns**.

**How I'll apply it:** every design doc starts with a "workload assumptions" section: sizes, read/write ratio, access pattern, concurrency, and failure rate, *before* choosing structures. Then validate those assumptions with measurements, as GFS's §6 does.

### Takeaway 2 — A single coordinator is fine if it's off the data path

**What the paper does:** one master holds all metadata and makes all placement decisions with global knowledge, which is far simpler than distributed consensus for everything. It stays fast because clients **cache** metadata, **large chunks** mean few lookups, data **never** flows through it, and **leases** delegate mutation ordering to primaries for 60 s at a time.

**Why it matters to me:** it's the same principle as minimising **lock scope** in concurrent code (#045, JCIP §2.5): do the expensive work *outside* the critical section, and touch the shared coordinator as briefly and rarely as possible. The master is a "global lock" with a short critical section.

**How I'll apply it:** when a design has a central component (a lock, a database row, a leader), identify what *must* go through it and move everything else (caching, batching, delegation) around it. Measure its load, as GFS did (200–500 ops/s), and know its limit.

### Takeaway 3 — Weaker guarantees plus end-to-end checks can beat strong global guarantees

**What the paper does:** instead of exactly-once, transactional appends across replicas, GFS offers **at-least-once atomic record append** and lets replicas differ in padding and duplicates. Applications make that safe with **checksums** (detect garbage), **unique ids** (drop duplicates) and **checkpoints** (read only what's complete). Each chunkserver checksums **its own** data rather than comparing replicas.

**Why it matters to me:** this is the **end-to-end argument**, and it's exactly the idempotency-key pattern coming in P5 (rows 417–430). It also mirrors #035's listener decision: define precisely what's guaranteed (the cache is consistent; the exception propagates) and let callers handle the rest.

**How I'll apply it:** for any distributed write path, decide explicitly between exactly-once (expensive, often impossible) and **at-least-once + idempotent consumers** (practical), and design record formats with an **id and a checksum** from day one.

---

## 3. Connecting GFS to Concurrency (next week)

| GFS mechanism | The concurrency concept behind it (rows 42–66) |
|---------------|------------------------------------------------|
| Namespace **read-write locks**, acquired in a **consistent total order** to prevent deadlock | Lock ordering (rows 54–55), read-write locks |
| **Leases** with expiry; the master waits for expiry before granting a new primary | Timeouts instead of indefinite waits; avoiding two "owners" (#046's predicates) |
| Primary assigns **serial numbers** → all replicas apply in the same order | Total ordering of operations, like a single lock defining a happens-before order (#045) |
| Op log flushed **before** acknowledging the client | Publication/visibility: don't expose a state change before it's durable (#048) |
| **Record append** from many clients to one file = multi-producer queue | Producer-consumer (#046), bounded blocking queue (rows 57–61) |
| **Copy-on-write** snapshots | Immutable snapshots for safe sharing (#044 Ch. 3) |
| An LRU buffer cache on chunkservers | The rows 32–36 project |

---

## 4. Open Questions (revisit later)

1. GFS's single master was later replaced (Colossus) by **distributed metadata**. Which assumption broke? (Hint: file counts grew, and so did small files.) → P4/P5 sharding.
2. How do modern object stores (S3) provide **strong read-after-write consistency** for every object, and at what cost? → P4 consistency (rows 342–366).
3. Would erasure coding (mentioned as future work in §5.1.2) change re-replication time and network cost? → P6 storage.

---

## 5. Plan for Next Week — Concurrency & Multithreading I (rows 42–66)

**Theme:** write correct concurrent Java from first principles, with threads, intrinsic locks, wait/notify, `volatile` and the JMM, and learn to **reproduce, test and measure** concurrency bugs.

**Primary resource:** *Java Concurrency in Practice* ch. 1–4 (+ ch. 14 for condition queues, ch. 16 for the JMM).

| Day | Rows | Learn | Build | Flex |
|-----|------|-------|-------|------|
| 1 | 42–44 | Threads, Runnable, lifecycle (#042) | Create & coordinate threads (#043) | JCIP ch. 1–4 notes (#044) |
| 2 | 45–47 | `synchronized`, intrinsic locks, monitor (#045) | Producer-consumer with wait/notify (#046) | Concurrency exercises (#047) |
| 3 | 48–50 | `volatile`, visibility, happens-before (#048) | Reproduce, then fix, a race (#049) | Review + refactor (#050) |
| 4 | 51–53 | Race conditions & how they arise | Fix an unsafe class with synchronization | Anki cards for concurrency |
| 5 | 54–56 | Deadlock: causes & prevention | Reproduce a deadlock, then resolve it | Gap-fill the weakest concurrency topic |
| 6–10 | 57–61 | — | **Project: thread-safe bounded blocking queue** from scratch + concurrency tests (plan → build → build → test/refactor → document) | — |
| 11 | 62–63 | Study + practice: JMM deep dive (happens-before, reordering; jcstress) | | |
| 12 | 64 | Weekly review + Anki | | |
| 13 | 65–66 | Paper: **MapReduce** (Dean & Ghemawat, 2004) + notes + plan for Week 4 (Concurrency II + Streams) | | |

### Definition of done for the week

- [ ] Can name all six thread states and what puts a thread in each (observed, not memorised)
- [ ] Can explain `synchronized` = mutual exclusion **+ visibility**, and read a `jstack` dump to find a lock holder
- [ ] Can write a correct `wait`/`notifyAll` loop and explain **both** classic mistakes (measured in #046)
- [ ] Can explain why a plain flag loop may never stop, and why `volatile count++` is still broken
- [ ] Have reproduced at least three races and a deadlock, and fixed them with evidence
- [ ] The blocking-queue project passes a concurrency test suite that is **itself** verified to catch bugs (the #050 lesson)
- [ ] Rows 42–66 marked `TRUE` in `Roadmap.csv` as they're done

### Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Flaky concurrency tests | Latched starts, timeouts on every test, repeated rounds, and mutation-check the tests (#050) |
| "It works on my machine" races | Reproduction harnesses with counts over many runs; use jcstress for memory-model questions (rows 62–63) |
| Over-reaching into `java.util.concurrent` too early | Rows 42–61 use primitives first; executors and friends arrive in rows 67+ |

---

## Handwritten Cheat Sheet

```
GFS IN ONE LINE: one in-memory master (metadata) + replicated 64 MB chunks (data);
                 relaxed consistency; automatic recovery; control ≠ data path
3 TAKEAWAYS:
 1. Design for YOUR workload (break defaults on purpose, then measure)
 2. Central coordinator OK if off the data path (cache, big chunks, leases) ≈ short critical sections
 3. Weaker guarantees + end-to-end checks (at-least-once + checksums + ids) beat global transactions
WEEK 3 (rows 42–66) CONCURRENCY I
 D1 threads/lifecycle · coordinate threads · JCIP 1–4
 D2 synchronized/monitors · wait/notify buffer · exercises
 D3 volatile/hb · reproduce+fix races · review (test the tests!)
 D4 races · fix unsafe class · Anki   D5 deadlock · reproduce/resolve · gap-fill
 D6–10 bounded blocking queue project   D11 JMM deep dive   D12 review   D13 MapReduce
```

---

**Next note:** #042 — Threads, Runnable, thread lifecycle
