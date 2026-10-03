# #035 — Test & Refactor: LRU Cache

**Date:** 2026-10-03  
**Type:** Project — Phase 4 of 5  
**Roadmap row:** 35 — P1 Java Core · Collections & Generics  
**Prerequisite:** **#033**, **#034**

---

## Goal

The contract suite proves the implementations handle the scenarios *you thought of*. This phase hunts for the ones you didn't, makes the cache safe to share between threads, and **measures** everything.

---

## Tasks

### 1. Reference model + differential testing

- Write `ReferenceLruCache<K, V>`: an **obviously correct, deliberately slow** O(n) implementation (an `ArrayList` of keys in LRU→MRU order + a `HashMap` of values). It must also pass the contract suite.
- Write `DifferentialTest`: feed the same random stream of `get`/`put`/`remove` (fixed seeds, many capacities and key ranges, ≥ 50 trials × 2 000 steps) to the reference and to each fast implementation. After **every step**, assert identical return values, eviction events (keys, in order), LRU→MRU order and stats.

### 2. Thread safety

- Write `SynchronizedCache<K, V>`, a **decorator** that makes any `Cache` thread-safe. Use a **private** lock object. Explain why a `ReadWriteLock` doesn't help an LRU cache.
- Run the whole contract suite through the decorator.
- A **stress test**: 8 threads × 200 000 random operations (70% get) on a capacity-500 cache, all released together by a `CountDownLatch`. Afterwards, verify: hit + miss counts equal the number of `get` calls made; listener calls equal reported evictions; the key list has no duplicates; its size equals `size()` and is ≤ capacity.

### 3. Hunt for an edge case the contract missed

Ask *"what can go wrong that isn't the cache's own fault?"* Specifically: **what happens when the eviction listener throws?** Probe all three implementations, record what state each is left in, decide on the correct behaviour, add a **regression test to the contract**, and fix all implementations.

### 4. Measure

- **JMH:** 1 000 operations per invocation (80% get / 20% put, keys uniform over 2 × capacity) for `LinkedHashMap`, `Manual`, `Synchronized(Manual)` and `Reference`, at capacity 1 000 and 100 000. Explain the results with Big-O and constant factors.
- **Hit-rate experiment:** with 100 000 keys and 2 000 000 read-through lookups, measure the LRU hit rate at cache sizes of 0.1%, 1%, 5%, 10%, 25% and 50% of the keys, under **uniform** and **Zipf(1.0)** traffic. What does this say about when caching pays off?

### 5. Refactor log

For every change you make in this phase, record: the trigger (test failure, probe, review, lint), the change, and the proof (which test went red then green, or which benchmark).

---

## Acceptance Criteria

- [ ] Reference model passes the contract; the differential test is green for every implementation
- [ ] `SynchronizedCache` passes the contract and the stress-test invariants
- [ ] The listener-exception behaviour is probed, decided, regression-tested and fixed in all implementations
- [ ] JMH results table + explanation
- [ ] Hit-rate table + conclusion
- [ ] Refactor log
- [ ] Whole build: tests green, zero `-Xlint:all` warnings

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
