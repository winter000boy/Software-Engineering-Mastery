# #033 — Build: LRU Cache — API + LinkedHashMap Implementation

**Date:** 2026-10-03  
**Type:** Project — Phase 2 of 5  
**Roadmap row:** 33 — P1 Java Core · Collections & Generics  
**Prerequisite:** complete **#032 Plan & Design** (or use the design in its solution)

---

## Goal

Implement the API from #032 and the **first** implementation, `LinkedHashMapLruCache<K, V>`, together with the **shared contract test suite** that every later implementation will also have to pass.

---

## Tasks

1. **API types** (package `com.mastery.lru`):
   - `interface Cache<K, V>`: `get`, `put`, `remove` (each returning `Optional<V>`), `containsKey`, `size`, `capacity`, `clear`, `keysLruToMru()`, `stats()`
   - `record CacheStats(long hits, long misses, long evictions)` with `requests()` and `hitRate()`
   - `@FunctionalInterface interface EvictionListener<K, V>` with a static `none()` factory
   - a small package-private `StatsCounter` that the implementations share
2. **`LinkedHashMapLruCache<K, V>`**:
   - **composition**: a private access-ordered `LinkedHashMap` whose `removeEldestEntry` enforces the capacity
   - accepts an `EvictionListener<? super K, ? super V>` (explain the wildcards)
   - presize the backing map so it **never resizes** (#020 §3.10)
   - `get` must not confuse "absent" with "present" (how does rejecting `null` values help?)
   - `containsKey` and `keysLruToMru` must not change recency (#023 §2.3: which `LinkedHashMap` operations count as an access?)
3. **`abstract class CacheContractTest`** with an abstract factory method `newCache(capacity, listener)`, and tests for:
   - miss then hit, with stats
   - LRU eviction order and listener notification
   - `get` refreshes recency
   - updating an existing key in a full cache: replaces, refreshes, **doesn't evict**
   - `containsKey`/`keysLruToMru` don't change recency
   - `remove` isn't an eviction and frees room
   - capacity 1
   - `clear` leaves the cache usable
   - nulls and a capacity ≤ 0 are rejected
   - a listener typed `EvictionListener<Object, Object>` is accepted by a `Cache<String, Integer>`
   - 10 000 inserts never exceed capacity, and the last 100 keys survive
4. `LinkedHashMapLruCacheTest extends CacheContractTest`: just the factory.

---

## Hints

- `new LinkedHashMap<>(initialCapacity, 0.75f, true)`: the third argument turns on access order.
- `removeEldestEntry` runs **after** the new entry has been inserted. Think about what that means for the size, and for what happens if your code inside it fails. (#035 comes back to this.)
- Iterating `keySet()` of an access-ordered map is not an access.

---

## Acceptance Criteria

- [ ] API types compile with zero `-Xlint:all` warnings
- [ ] `LinkedHashMapLruCache` uses composition; all operations O(1)
- [ ] The contract suite has ≥ 10 tests, all green
- [ ] Wildcards on the listener justified

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
