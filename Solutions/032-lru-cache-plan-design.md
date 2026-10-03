# #032 — Solution: Plan & Design — LRU Cache

**Date:** 2026-10-03  
**Problem:** [032-lru-cache-plan-design.md](../problems/032-lru-cache-plan-design.md)

> This is the design document the project was built from (#033–#035). The API below is the exact interface that was later compiled and tested; nothing here is aspirational.

---

## 1. Choosing the Data Structures

| Option | get | put (new, full) | evict LRU | Verdict |
|--------|-----|-----------------|-----------|---------|
| `ArrayList` of keys (LRU→MRU) + `HashMap` of values | **O(n)**: find the key in the list and move it to the end | O(n) | O(n): `remove(0)` shifts everything | Obviously correct, so it's the **reference model** for tests, never production |
| `TreeMap<accessTime, key>` + `HashMap<key, (time, value)>` | O(log n): remove the old time, insert the new one | O(log n) | O(log n): `pollFirstEntry` | Works, but log n with a lot of object churn, and it needs a unique, monotonic clock |
| `PriorityQueue` by last-access time | **O(n)**: `PriorityQueue.remove(Object)` is a linear scan | O(log n) | O(log n) | Re-prioritising an element is the problem; worse than the TreeMap |
| **`LinkedHashMap` (access order)** | **O(1)** | **O(1)** | **O(1)**: `removeEldestEntry` | ✅ **Implementation 1**: the platform already does it |
| **`HashMap<K, Node>` + doubly linked list** | **O(1)** | **O(1)** | **O(1)**: unlink `head.next` | ✅ **Implementation 2**: the same structure as `LinkedHashMap`, written by hand |

**Why does a hash map plus a doubly linked list give O(1)?** The map finds *any* node in O(1). A doubly linked list can *unlink any node it's given* in O(1), because the node knows both neighbours, and append at the tail in O(1). Recency is the list order: the head is the LRU and the tail is the MRU. Neither structure is enough on its own. A list alone needs O(n) to *find* a node; a map alone has no order.

---

## 2. API

### `Cache.java`

```java
package com.mastery.lru;

import java.util.List;
import java.util.Optional;

/**
 * A bounded key-value cache with a fixed capacity.
 *
 * <p>Contract shared by every implementation (verified by {@code CacheContractTest}):
 * <ul>
 *   <li>{@code get} and {@code put} count as a <em>use</em> and make the key most-recently-used;</li>
 *   <li>{@code containsKey}, {@code size}, {@code keysLruToMru} do NOT change recency (they "peek");</li>
 *   <li>when a put of a NEW key would exceed the capacity, exactly one entry, the
 *       least-recently-used, is evicted first and reported to the eviction listener;</li>
 *   <li>null keys and null values are rejected with NullPointerException.</li>
 * </ul>
 */
public interface Cache<K, V> {

    /** Value for key (and marks it most-recently-used), or empty on a miss. */
    Optional<V> get(K key);

    /** Inserts or replaces; marks key most-recently-used; may evict the LRU entry. Returns the previous value. */
    Optional<V> put(K key, V value);

    /** Removes the key if present (not an eviction: the listener is not called). Returns the removed value. */
    Optional<V> remove(K key);

    /** Whether key is cached, WITHOUT affecting recency. */
    boolean containsKey(K key);

    int size();

    int capacity();

    void clear();

    /** Snapshot of keys from least- to most-recently-used. O(n); for tests, debugging, admin endpoints. */
    List<K> keysLruToMru();

    /** Hit/miss/eviction counters since creation. */
    CacheStats stats();
}
```

### `CacheStats.java`

```java
package com.mastery.lru;

/** Immutable snapshot of cache counters. */
public record CacheStats(long hits, long misses, long evictions) {

    public long requests() {
        return hits + misses;
    }

    /** Fraction of get() calls that hit; 0.0 when there were no requests. */
    public double hitRate() {
        long requests = requests();
        return requests == 0 ? 0.0 : (double) hits / requests;
    }
}
```

### `EvictionListener.java`

```java
package com.mastery.lru;

/** Called synchronously, inside the put() that caused the eviction. Keep it fast. */
@FunctionalInterface
public interface EvictionListener<K, V> {

    void onEviction(K key, V value);

    /** A listener that does nothing. */
    static <K, V> EvictionListener<K, V> none() {
        return (k, v) -> { };
    }
}
```

### API decisions

| Question | Decision | Why |
|----------|----------|-----|
| Return `null` or `Optional` on a miss? | **`Optional<V>`** | A miss is a normal, expected outcome, so the caller must handle it (EJ Item 55, #019). `null` would also be ambiguous if null values were allowed |
| Allow `null` keys/values? | **No** (NPE) | It removes the "absent vs present-but-null" ambiguity. It's the same choice as `ConcurrentHashMap`, `List.of`, and Caffeine |
| What is a "use"? | `get` and `put` **only** | `containsKey`, `size` and `keysLruToMru` are *observations*; if they changed recency, monitoring code would change cache behaviour |
| Does `put` on an existing key in a full cache evict? | **No** | Nothing new is added, so nothing needs to leave (FR-4). A naive "if full, evict, then insert" implementation gets this wrong |
| Does `remove` call the listener? | **No** | The listener reports *capacity pressure*, not explicit deletions (FR-5) |
| `keysLruToMru` returns? | A **snapshot copy**, O(n) | Exposing a live view would leak internals and break under concurrent modification |
| Listener parameter type | `EvictionListener<? super K, ? super V>` | The listener **consumes** keys and values, so `? super` (PECS, #026). An `EvictionListener<Object, Object>` works for any cache |

---

## 3. Inheritance vs Composition (LinkedHashMap version)

| | `class LruCache<K,V> extends LinkedHashMap<K,V>` | `class LinkedHashMapLruCache<K,V> implements Cache<K,V>` **has-a** `LinkedHashMap` |
|---|---|---|
| Lines of code | Fewer | A few more (delegation) |
| Stats and listener | **Bypassable**: callers can use `putAll`, `compute`, `entrySet().removeIf`, `getOrDefault`… which skip our counting | Every access goes through our methods |
| Exposed API | ~40 inherited `Map` methods, all of them now part of our contract | Exactly the `Cache` interface |
| Swappable implementation | No | Yes, behind `Cache` |
| Verdict | | ✅ **Composition** (EJ Item 18) |

The `LinkedHashMap` is still subclassed, but as a **private anonymous class**, purely to override `removeEldestEntry`. That inheritance is invisible to users.

---

## 4. Sentinel Nodes (manual version)

Without sentinels, `unlink` needs special cases:

```java
if (node.prev == null) head = node.next; else node.prev.next = node.next;   // first?
if (node.next == null) tail = node.prev; else node.next.prev = node.prev;   // last?
```

With two permanent dummy nodes (`head` before the LRU, `tail` after the MRU), every real node *always* has both neighbours:

```java
node.prev.next = node.next;
node.next.prev = node.prev;
```

That's no branches and no empty/first/last special cases, and those special cases are exactly where linked-list bugs live. The cost is two extra objects per cache.

---

## 5. Thread Safety

**Decision: a separate decorator, `SynchronizedCache`,** wrapping any `Cache`. The implementations stay simple and fast for single-threaded use; thread safety is opt-in and written once (the same idea as `Collections.synchronizedMap`).

**Can a `ReadWriteLock` help?** **No.** In an LRU cache, `get` is a **write**: it moves the entry to the MRU position, mutating the linked list. Two "readers" moving nodes at the same time would corrupt the list. With a read-write lock, every `get` would need the write lock, which is just a mutex with extra overhead. High-throughput caches (Caffeine) solve this differently: reads record accesses into lock-free buffers and apply the reorderings later in batches. That's out of scope here (#036 limitations).

---

## 6. Class Diagram & Layout

```
                     «interface» Cache<K,V>
          ┌──────────────┬──────┴───────┬────────────────────┐
LinkedHashMapLruCache  ManualLruCache  ReferenceLruCache   SynchronizedCache ──delegate──▶ Cache
  has-a LinkedHashMap   has-a HashMap    has-a ArrayList     (decorator)
  (anon. subclass)      + Node list      + HashMap
          └──────── all use ────────┘
              StatsCounter (package-private) ──snapshot──▶ «record» CacheStats
              «functional interface» EvictionListener<K,V>

com.mastery.lru          API + implementations
com.mastery.lru.bench    JMH benchmark, hit-rate experiment
src/test/.../lru         CacheContractTest (abstract) + one subclass per implementation,
                         DifferentialTest
```

---

## 7. Test Plan

| Layer | What | How |
|-------|------|-----|
| **Contract** | FR-1…FR-9 for **every** implementation | `abstract class CacheContractTest` with an abstract factory; one 5-line subclass per implementation, so the same tests run on all of them |
| **Differential** | Behaviour identical to an obviously correct model on random input | `ReferenceLruCache` (the O(n) list version) vs each fast version: the same seeded random stream, comparing return values, evictions, order and stats after **every** step |
| **Concurrency** | FR-10 | 8 threads hammering `SynchronizedCache`; invariants: stats add up, listener calls = evictions, no duplicate keys, size ≤ capacity |
| **Lint** | Code hygiene | `-Xlint:all` in the compiler config, zero warnings |

## 8. Benchmark Plan & Predictions

- **Workload:** 1 000 operations per invocation, 80% `get` / 20% `put`, keys uniform over 2 × capacity (the cache stays full; ~50% of gets hit).
- **Parameters:** capacity ∈ {1 000, 100 000} × implementation ∈ {LinkedHashMap, Manual, Synchronized(Manual), Reference}.
- **Predictions (written before measuring):**
  1. LinkedHashMap ≈ Manual (same data structure), roughly flat as capacity grows (O(1)).
  2. Synchronized adds a small constant (an uncontended lock costs tens of ns).
  3. Reference grows ~100× from 1 000 → 100 000 (O(n) per operation).
- (The results are in #035: predictions 1 and 2 held; prediction 3 held in shape, at ~210×.)

---

## 9. Risks & Open Questions

| # | Risk / question | Plan |
|---|-----------------|------|
| R1 | `removeEldestEntry` semantics (when exactly is it called?) | Read the JDK source (#023 §2.2); covered by contract tests |
| R2 | Pointer bugs in the manual list | Sentinels + the differential test against the reference model |
| R3 | Benchmark artefacts (dead code, warm-up) | JMH; return results; fixed seeds (#018) |
| Q1 | What if the eviction listener throws? | **Open.** Not specified yet, to be probed in #035 |
| Q2 | Expiry (TTL)? Weights? | Out of scope; listed in #036 limitations |

(Q1 turned out to matter: #035 found a real bug there.)

---

## Key Takeaways

1. O(1) LRU = **hash map (find) + doubly linked list (order)**. Neither alone is enough.
2. Define precisely what counts as a **use**; observation methods must not change state.
3. **Composition over inheritance** keeps the API small and the bookkeeping unbypassable.
4. **Sentinels** delete a whole class of linked-list edge cases.
5. In an LRU cache, **every read is a write**, so read-write locks don't help.
6. Write benchmark predictions *before* measuring, and list open questions explicitly, since that's where the bugs hide.
