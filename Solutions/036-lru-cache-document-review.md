# #036 — Solution: Document & Review — LRU Cache

**Date:** 2026-10-03  
**Problem:** [036-lru-cache-document-review.md](../problems/036-lru-cache-document-review.md)

> Code: #033 (LinkedHashMap), #034 (manual), #035 (reference model, decorator, fix, benchmarks). Every number here is taken from #035's measured results.

---

## 1. README (project artifact)

````markdown
# lru-cache

A small, dependency-free, bounded **LRU (least-recently-used) cache** for Java 21, with two
O(1) implementations behind one interface, hit/miss/eviction stats, eviction listeners and an
opt-in thread-safe wrapper.

## Which implementation?

| Class | Use it when | Notes |
|-------|-------------|-------|
| `LinkedHashMapLruCache` | **Default.** Single-threaded or externally synchronized use | Fastest in our benchmark (~16–20 ns/op), 1 object per entry |
| `ManualLruCache` | You want to read/modify the algorithm itself | Same O(1) behaviour, ~0–20% slower, 2 objects per entry |
| `SynchronizedCache(any)` | The cache is shared between threads | One lock; ~3 ns/op overhead uncontended |
| `ReferenceLruCache` | Tests only (reference model) | O(n) — 1 800× slower at 100k entries |

## Quick start

```java
Cache<String, Report> cache = new SynchronizedCache<>(
        new LinkedHashMapLruCache<>(5_000, (key, report) -> metrics.increment("report.evicted")));

Report r = cache.get(id).orElseGet(() -> {
    Report loaded = reportService.load(id);   // expensive
    cache.put(id, loaded);
    return loaded;
});

double hitRate = cache.stats().hitRate();     // export to your metrics periodically
```

## Contract

- **Capacity** is fixed (> 0). The cache never holds more entries than its capacity.
- A **use** is `get` (hit) or `put`. A use makes the key most-recently-used.
  `containsKey`, `size`, `keysLruToMru` and `stats` are observations and never change recency.
- When a `put` of a **new** key finds the cache full, exactly one entry — the least recently
  used — is evicted. Updating an existing key never evicts.
- `remove` and `clear` are not evictions and do not call the listener.
- `null` keys and values are rejected (`NullPointerException`); misses return `Optional.empty()`.
- **Eviction listener:** called synchronously inside the `put` that caused the eviction,
  *after* the cache has finished updating. If it throws, the exception propagates to the
  `put` caller, but the cache is already in its correct state (new entry stored, LRU evicted).
- Not thread-safe unless wrapped in `SynchronizedCache`.

## Performance (Apple M4, JDK 21, JMH; 80% get / 20% put)

| | capacity 1 000 | capacity 100 000 |
|---|---:|---:|
| LinkedHashMapLruCache | 19 ns/op | 16 ns/op |
| ManualLruCache | 20 ns/op | 19 ns/op |
| SynchronizedCache(Manual) | 23 ns/op | 22 ns/op |

**Sizing:** hit rate depends on your key distribution. With skewed (Zipf) traffic over 100k keys,
1% capacity gave a 51% hit rate; with uniform traffic, 1% capacity gave 1%. Measure with `stats()`.

## Build

    mvn package                     # compiles with -Xlint:all, runs 52 tests, builds benchmarks.jar
    java -jar target/benchmarks.jar LruBenchmark
    java -cp target/benchmarks.jar com.mastery.lru.bench.HitRateExperiment
````

---

## 2. Design Decisions Record

| # | Decision | Context | Alternatives | Consequences |
|---|----------|---------|--------------|--------------|
| D1 | **Hash map + doubly linked list** for O(1) | Need O(1) find *and* O(1) reorder | List (O(n)), TreeMap by time (O(log n)), PriorityQueue (O(n) re-prioritise) | Two pointers + a map entry per element; all operations O(1) (#032 §1) |
| D2 | **Composition** around `LinkedHashMap` | Stats and listener must not be bypassable | `extends LinkedHashMap` (inherits ~40 public `Map` methods) | Small API; a private anonymous subclass only for `removeEldestEntry` |
| D3 | **`Optional<V>`** returns; **reject nulls** | A miss is normal; avoid the absent/null ambiguity | Return `null`; allow null values | Callers must handle misses; `get` is one lookup |
| D4 | **Sentinel** head/tail in the manual list | Linked-list edge cases cause bugs | `null`-terminated list with special cases | Branch-free `unlink`/`link`; two extra objects per cache |
| D5 | Thread safety as a **decorator** with a **private lock** | Most uses are single-threaded; `get` mutates order | `synchronized` in each implementation; `ReadWriteLock` (useless: reads write) | Opt-in; ~15% uncontended overhead; one lock limits scalability (§5) |
| D6 | **Notify after mutation**; propagate listener exceptions | Found in #035: notify-before-mutation left broken or half-done state | Swallow and log; wrap the exception | The cache is always consistent; callers still see failures |
| D7 | **Contract tests in an abstract class** + a reference model + differential tests | Several implementations must behave identically | Per-implementation tests | One spec, run against 4 implementations; random-sequence coverage |
| D8 | **Presize** the backing map (`capacity + 1` / 0.75; `HashMap.newHashMap`) | Size is known up front | Default capacity | No resize latency spikes (#020 §3.10) |

---

## 3. Requirements Traceability (#032 → evidence)

| Req | Requirement | Evidence | Status |
|-----|-------------|----------|:---:|
| FR-1 | `get` returns the value and marks MRU; a miss → empty | `getMissThenHit`, `getRefreshesRecency` | ✅ |
| FR-2 | `put` inserts/replaces, marks MRU, returns the previous value | `putOnExistingKeyReplacesRefreshesAndDoesNotEvict` | ✅ |
| FR-3 | Evict exactly the LRU entry when full | `evictsLeastRecentlyUsedWhenFull`, `neverExceedsCapacityUnderManyInserts`, `DifferentialTest` | ✅ |
| FR-4 | Updating a key in a full cache doesn't evict | `putOnExistingKeyReplacesRefreshesAndDoesNotEvict` | ✅ |
| FR-5 | `remove` isn't an eviction | `removeIsNotAnEviction` | ✅ |
| FR-6 | Observations don't change recency | `containsKeyAndKeysDoNotChangeRecency` | ✅ |
| FR-7 | Eviction listener | `evictsLeastRecentlyUsedWhenFull`, `listenerReceivesEvictedValueAndAcceptsSupertypes`, `throwingListenerLeavesCacheConsistent` | ✅ (after the #035 fix) |
| FR-8 | Stats | `getMissThenHit`, `DifferentialTest` (stats compared every step), stress test | ✅ |
| FR-9 | Reject nulls, bad capacity | `rejectsNullsAndBadCapacity` | ✅ |
| FR-10 | Thread-safe variant | `SynchronizedCacheTest` (contract + 8-thread stress test) | ✅ |
| NFR-1 | O(1) operations | JMH: flat from 1k → 100k (#035 §5) | ✅ |
| NFR-2 | One interface, interchangeable implementations | 4 implementations pass one contract suite | ✅ |
| NFR-3 | Correct generics | Listener `? super` test; `-Xlint:all` clean | ✅ |
| NFR-4 | Contract + differential tests | 52 tests | ✅ |
| NFR-5 | JMH at 1k and 100k | #035 §5 | ✅ |
| NFR-6 | No dependencies beyond JUnit/JMH | `pom.xml` | ✅ |

---

## 4. Code Review

| Check | Result | Note / finding |
|-------|:---:|----------------|
| Contracts documented | ✅ | `Cache` Javadoc + README "Contract" section |
| Invariants stated and tested | ✅ | size ≤ capacity; map and list agree; LRU at the head |
| Complexity documented | ✅ | Class Javadocs; README table |
| Generics & wildcards | ✅ | `EvictionListener<? super K, ? super V>`; generic test factory |
| Immutability where possible | ✅ | `CacheStats` record; `final` fields; snapshot `keysLruToMru` |
| No leaked internals | ✅ | `keysLruToMru` returns a copy; nodes and maps are private |
| Exception behaviour defined | ✅ (after #035) | Listener exceptions: the cache is consistent, the exception propagates. **This was the review's main finding, caught in #035** |
| Tests independent & deterministic | ⚠️ | Contract and differential tests use fixed seeds; the **stress test** uses `ThreadLocalRandom` (non-deterministic by design). Its assertions are invariants, so it can't flake on a correct implementation, but a failure wouldn't be exactly reproducible |
| Benchmarks reproducible | ✅ | Fixed seed, the same stream for every implementation, JMH forks, hardware recorded |
| `StatsCounter` thread safety | ⚠️ documented | Plain `long` fields, safe only under the decorator's lock. Documented in its Javadoc |
| Listener runs under the lock (when decorated) | ⚠️ limitation | A slow listener blocks every other thread. Documented: "Keep it fast" |

---

## 5. Limitations & Future Work (vs. Caffeine)

| Area | This library | What production caches (e.g. **Caffeine**) do |
|------|-------------|---------------------------------------------|
| Eviction policy | Pure LRU | **W-TinyLFU**: a frequency sketch decides *admission*, so one-off scans don't flush popular entries. Higher hit rates on real traffic |
| Concurrency | One global lock | Lock-free reads; access events recorded in ring buffers and applied in batches; striped writes |
| Expiry | None | `expireAfterWrite` / `expireAfterAccess` / per-entry expiry, driven by a timer wheel |
| Size | Entry count | Also **weight**-based (`maximumWeight` + a `Weigher`), e.g. a bytes budget |
| Loading | Manual `get` then `put` (two lock acquisitions, a thundering herd on a miss) | `get(key, loader)`: atomic compute-if-absent, so one load per key |
| Listener | Synchronous, under the lock | Asynchronous via an `Executor` |
| Stats | Hits/misses/evictions | Plus load times, load failures, eviction weight; Micrometer integration |

**Biggest functional gap:** there's no atomic `computeIfAbsent`-style `get(key, loader)`. Two threads that miss on the same key both load it. Next step: `V get(K key, Function<? super K, ? extends V> loader)` holding the lock for the miss path, or per-key futures for slow loaders.

**Pure LRU's weakness:** a single **scan** (e.g. a batch job reading 1M keys once) evicts the entire working set. That's why real caches add frequency information (LFU, ARC, TinyLFU).

---

## 6. Retrospective

### What went well

- **Design first (#032)** gave a precise contract, so the tests were written against the interface and every implementation was checked against them for free.
- **The reference model + differential test** gave very high confidence: 300 000 random steps, with everything compared after each one.
- **Predictions before measuring:** O(1) flatness and a small lock overhead were confirmed; the O(n) blow-up was bigger than predicted (213× vs ~100×), which led to the cache-locality explanation.

### What went wrong

- **The listener-exception bug shipped through two build phases.** The contract suite only used well-behaved listeners, and the design doc had even listed "what if the listener throws?" as an *open question* (#032 §9) without a test. In the `LinkedHashMap` version it broke the core invariant (capacity + 1 entries). **Lesson:** every open question in a design doc should become a test before the build phase ends.
- **Fixing it was uneven:** trivial in the manual version, a workaround in the `LinkedHashMap` version, because the JDK hook's timing can't be changed. **Lesson:** code inside framework hooks should only *record* information; act on it after the framework call returns.

### What I'd do differently

1. Turn every "open question" in the design into a test (even a `@Disabled` one with a TODO) on day 1.
2. Add a "hostile collaborator" category to every contract: a listener that throws, a key whose `hashCode` throws, a value that's mutated after `put`.
3. Make the concurrency test reproducible: record the seed per thread and print it on failure.

---

## Key Takeaways

1. A README states **contracts and trade-offs**, not just usage. The listener's exception semantics belong in it.
2. A **traceability table** shows that every requirement has evidence, and exposes the ones that don't.
3. A **decisions record** with alternatives lets future readers change a decision knowingly.
4. Know the gap between a learning project and production (W-TinyLFU, async loading, expiry, lock-free reads), and say so explicitly.
5. The retrospective's most valuable line is the one about **what went wrong** and the process change that prevents it next time.
