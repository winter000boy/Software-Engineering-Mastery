# #049 — Reproduce, Then Fix, a Race Condition

**Date:** 2026-10-12  
**Type:** Build  
**Roadmap row:** 49 — P1 Java Core · Concurrency & Multithreading I  
**Topics:** read-modify-write races, check-then-act races, unsafe collections, compound actions on thread-safe collections, `synchronized`, `AtomicLong`, `LongAdder`, CAS loops, `ConcurrentHashMap.merge`, reproducible concurrency tests  
**Background:** #045 (synchronized), #048 (volatile ≠ atomic)

---

## Overview

You can't fix what you can't reproduce. For three classic races: **write the broken version, prove it's broken with numbers, fix it (more than one way where possible), and write tests** that pass on the fixes.

All implementations sit behind small interfaces, so one harness and one set of tests can drive both the broken and the fixed versions.

---

## Race 1 — Lost Updates (read-modify-write)

```java
interface HitCounter { void increment(); long get(); }
```

- **Racy:** `count++` on a plain `long`.
- **Fixes:** (a) `synchronized` with a private lock (reads too), (b) `AtomicLong.incrementAndGet()`, (c) `LongAdder`.

## Race 2 — Check-Then-Act (overselling)

```java
interface SeatInventory { boolean tryBook(); int booked(); int capacity(); }
```

- **Racy:** `if (booked < capacity) { booked++; return true; } return false;`
- **Fixes:** (a) the check and the act under **one** lock, (b) a lock-free **CAS loop** on an `AtomicInteger` (`get` → check → `compareAndSet`, retry on failure).

## Race 3 — Unsafe Map, and a "Safe" Map Used Unsafely

```java
interface WordCounter { void add(String word); Map<String, Integer> snapshot(); }
```

- **Racy A:** a plain `HashMap` with `put(w, getOrDefault(w, 0) + 1)`.
- **Racy B:** the **same code on a `ConcurrentHashMap`**. Predict: is it fixed?
- **Fix:** `ConcurrentHashMap.merge(w, 1, Integer::sum)`.

---

## Tasks

1. **Reproduction harness:** run every implementation 20 times with 8 threads, **released together by a `CountDownLatch`**, and report per implementation: how many runs were wrong, and the worst damage (lost increments, tickets sold beyond capacity, lost counts).
2. **Explain** each race with a concrete interleaving.
3. **Explain** the seat result: why can the racy inventory sell *several times* its capacity, not just a few extra seats?
4. **Tests:** one **abstract contract test class** per interface (sequential behaviour + "no lost updates / no oversell under contention"), with a subclass for each **fixed** implementation. Tests must use a latch-based start, rethrow worker failures, and use no `sleep`.
5. Decide which fix you'd ship for each race, and why. (Row 50 will measure this properly.)

---

## Constraints

- Java 21, JUnit 5; no external concurrency libraries.
- Racy versions must be **simple and obviously written**. No artificial `sleep`s to widen the race window: reproduce real races.

---

## Acceptance Criteria

- [ ] Harness output for all 9 implementations (wrong runs + worst damage)
- [ ] An interleaving for each race; the oversell multiplier explained
- [ ] Contract tests green for every fixed implementation
- [ ] A fix chosen per race, with a justification

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
