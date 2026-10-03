# #033 — Solution: Build — LRU Cache API + LinkedHashMap Implementation

**Date:** 2026-10-03  
**Problem:** [033-lru-cache-build-linkedhashmap.md](../problems/033-lru-cache-build-linkedhashmap.md)

> Verified on Java 21: **11/11 contract tests pass** for this implementation, with **zero `-Xlint:all` warnings**. This is the code **as built in this phase**. Phase #035 later found a bug in its eviction-listener handling (see §5); the fixed version is shown there.

---

## 1. API Types

`Cache`, `CacheStats` and `EvictionListener` are exactly as designed in #032 §2. The only new type is the shared counter:

### `StatsCounter.java`

```java
package com.mastery.lru;

/** Mutable counters shared by the implementations (package-private; not thread-safe on its own). */
final class StatsCounter {
    private long hits;
    private long misses;
    private long evictions;

    void recordHit()      { hits++; }
    void recordMiss()     { misses++; }
    void recordEviction() { evictions++; }

    CacheStats snapshot() {
        return new CacheStats(hits, misses, evictions);
    }
}
```

It's package-private and mutable (an implementation detail); `stats()` hands out immutable `CacheStats` snapshots. It isn't thread-safe on its own; `SynchronizedCache` (#035) provides the locking.

---

## 2. `LinkedHashMapLruCache` (phase-2 version)

### `LinkedHashMapLruCache.java (v1)`

```java
package com.mastery.lru;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * LRU cache built on an access-ordered {@link LinkedHashMap} (#023).
 *
 * <p>Uses COMPOSITION, not inheritance: the cache HAS-A LinkedHashMap. Extending LinkedHashMap
 * would expose every Map method (putAll, entrySet().removeIf, ...) and let callers bypass the
 * stats and the eviction listener (EJ Item 18).
 *
 * <p>All operations O(1). Not thread-safe: wrap with {@link SynchronizedCache} to share.
 */
public final class LinkedHashMapLruCache<K, V> implements Cache<K, V> {

    private final int capacity;
    private final Map<K, V> map;
    private final StatsCounter stats = new StatsCounter();
    private final EvictionListener<? super K, ? super V> listener;   // consumer of K and V → ? super

    public LinkedHashMapLruCache(int capacity) {
        this(capacity, EvictionListener.none());
    }

    public LinkedHashMapLruCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be positive: " + capacity);
        }
        this.capacity = capacity;
        this.listener = Objects.requireNonNull(listener, "listener");
        // Presize so the backing table never resizes: capacity + 1 entries exist briefly during a put.
        int initialBuckets = (int) Math.ceil((capacity + 1) / 0.75);
        this.map = new LinkedHashMap<>(initialBuckets, 0.75f, true) {     // true = access order
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                if (size() > LinkedHashMapLruCache.this.capacity) {
                    stats.recordEviction();
                    listener.onEviction(eldest.getKey(), eldest.getValue());
                    return true;                                       // LinkedHashMap removes the head (the LRU)
                }
                return false;
            }
        };
    }

    @Override
    public Optional<V> get(K key) {
        Objects.requireNonNull(key, "key");
        V value = map.get(key);                       // access-ordered: a hit moves the entry to the tail
        if (value == null) {                          // safe: null values are never stored
            stats.recordMiss();
            return Optional.empty();
        }
        stats.recordHit();
        return Optional.of(value);
    }

    @Override
    public Optional<V> put(K key, V value) {
        Objects.requireNonNull(key, "key");
        Objects.requireNonNull(value, "value");
        return Optional.ofNullable(map.put(key, value));   // existing key: moved to tail; new key: may evict
    }

    @Override
    public Optional<V> remove(K key) {
        Objects.requireNonNull(key, "key");
        return Optional.ofNullable(map.remove(key));
    }

    @Override
    public boolean containsKey(K key) {
        Objects.requireNonNull(key, "key");
        return map.containsKey(key);                  // containsKey is NOT an access in LinkedHashMap (#023)
    }

    @Override
    public int size() {
        return map.size();
    }

    @Override
    public int capacity() {
        return capacity;
    }

    @Override
    public void clear() {
        map.clear();
    }

    @Override
    public List<K> keysLruToMru() {
        return new ArrayList<>(map.keySet());         // iteration order = LRU → MRU, and iterating isn't an access
    }

    @Override
    public CacheStats stats() {
        return stats.snapshot();
    }

    @Override
    public String toString() {
        return "LinkedHashMapLruCache" + map;
    }
}
```

### Walkthrough

- **Composition:** the field `map` is a private, anonymous subclass of `LinkedHashMap`. Users only ever see `Cache`, so they can't bypass the stats with `putAll` or `entrySet().removeIf` (#032 §3).
- **`removeEldestEntry`** is the hook `LinkedHashMap.afterNodeInsertion` calls after every insertion (#023 §2.2). Returning `true` makes the map remove its **head**, which in access order is the LRU entry.
- **Presizing:** at most `capacity + 1` entries exist (briefly, between the insert and the eviction), so `ceil((capacity + 1) / 0.75)` buckets means the table **never resizes** (#020 §3.10).
- **`get` uses `value == null` to detect a miss.** That's only valid because `put` rejects null values. Allowing nulls would mean a second `containsKey` lookup, and a hit would then be two hash lookups.
- **`containsKey` and `keysLruToMru`** don't change recency, because in `LinkedHashMap`, `containsKey` and iteration aren't "accesses" (#023 §2.3 table, verified).
- **`LinkedHashMapLruCache.this.capacity`** inside the anonymous class: an anonymous class has its *own* `size()` (the map's), and the outer class's field needs qualifying to be unambiguous to readers.
- **Wildcards:** `EvictionListener<? super K, ? super V>`. The listener *consumes* keys and values, so a `Cache<String, Integer>` accepts an `EvictionListener<Object, Object>` (tested).

---

## 3. The Contract Suite

### `CacheContractTest.java (phase-2 version: 11 tests)`

```java
package com.mastery.lru;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import org.junit.jupiter.api.Test;

/**
 * The behavioural contract of {@link Cache}. Every implementation gets a tiny subclass that
 * supplies the factory method, so all implementations are held to exactly the same tests.
 */
abstract class CacheContractTest {

    /** Factory supplied by each implementation's subclass. */
    protected abstract <K, V> Cache<K, V> newCache(int capacity, EvictionListener<? super K, ? super V> listener);

    protected <K, V> Cache<K, V> newCache(int capacity) {
        return newCache(capacity, EvictionListener.none());
    }

    @Test
    void getMissThenHit() {
        Cache<String, Integer> c = newCache(2);
        assertEquals(Optional.empty(), c.get("a"));
        c.put("a", 1);
        assertEquals(Optional.of(1), c.get("a"));
        assertEquals(new CacheStats(1, 1, 0), c.stats());
        assertEquals(0.5, c.stats().hitRate());
    }

    @Test
    void evictsLeastRecentlyUsedWhenFull() {
        List<String> evicted = new ArrayList<>();
        Cache<String, Integer> c = newCache(3, (k, v) -> evicted.add(k + "=" + v));
        c.put("a", 1);
        c.put("b", 2);
        c.put("c", 3);
        c.put("d", 4);                                      // a is LRU → evicted
        assertEquals(List.of("a=1"), evicted);
        assertEquals(List.of("b", "c", "d"), c.keysLruToMru());
        assertEquals(3, c.size());
        assertEquals(1, c.stats().evictions());
    }

    @Test
    void getRefreshesRecency() {
        Cache<String, Integer> c = newCache(3);
        c.put("a", 1);
        c.put("b", 2);
        c.put("c", 3);
        c.get("a");                                         // a becomes MRU → b is now LRU
        c.put("d", 4);
        assertEquals(List.of("c", "a", "d"), c.keysLruToMru());
        assertFalse(c.containsKey("b"));
    }

    @Test
    void putOnExistingKeyReplacesRefreshesAndDoesNotEvict() {
        List<String> evicted = new ArrayList<>();
        Cache<String, Integer> c = newCache(2, (k, v) -> evicted.add(k));
        c.put("a", 1);
        c.put("b", 2);
        assertEquals(Optional.of(1), c.put("a", 10));       // returns the previous value
        assertEquals(List.of("b", "a"), c.keysLruToMru());  // a refreshed to MRU
        assertEquals(List.of(), evicted);                   // update of a full cache must NOT evict
        assertEquals(2, c.size());
        c.put("c", 3);                                      // now b (LRU) goes
        assertEquals(List.of("b"), evicted);
        assertEquals(Optional.of(10), c.get("a"));
    }

    @Test
    void containsKeyAndKeysDoNotChangeRecency() {
        Cache<String, Integer> c = newCache(2);
        c.put("a", 1);
        c.put("b", 2);
        assertTrue(c.containsKey("a"));                     // a peek, not a use
        c.keysLruToMru();
        c.put("c", 3);                                      // a is still LRU → evicted
        assertFalse(c.containsKey("a"));
        assertEquals(List.of("b", "c"), c.keysLruToMru());
    }

    @Test
    void removeIsNotAnEviction() {
        List<String> evicted = new ArrayList<>();
        Cache<String, Integer> c = newCache(2, (k, v) -> evicted.add(k));
        c.put("a", 1);
        assertEquals(Optional.of(1), c.remove("a"));
        assertEquals(Optional.empty(), c.remove("a"));
        assertEquals(List.of(), evicted);
        assertEquals(0, c.size());
        c.put("b", 2);
        c.put("c", 3);                                      // room was freed: no eviction
        assertEquals(List.of(), evicted);
    }

    @Test
    void capacityOfOne() {
        Cache<String, Integer> c = newCache(1);
        c.put("a", 1);
        c.put("b", 2);
        assertEquals(List.of("b"), c.keysLruToMru());
        c.put("b", 3);
        assertEquals(Optional.of(3), c.get("b"));
        assertEquals(1, c.stats().evictions());
    }

    @Test
    void clearEmptiesButCacheStaysUsable() {
        Cache<Integer, Integer> c = newCache(3);
        for (int i = 0; i < 3; i++) {
            c.put(i, i);
        }
        c.clear();
        assertEquals(0, c.size());
        assertEquals(List.of(), c.keysLruToMru());
        c.put(9, 9);
        assertEquals(List.of(9), c.keysLruToMru());
    }

    @Test
    void rejectsNullsAndBadCapacity() {
        Cache<String, Integer> c = newCache(2);
        assertThrows(NullPointerException.class, () -> c.put(null, 1));
        assertThrows(NullPointerException.class, () -> c.put("a", null));
        assertThrows(NullPointerException.class, () -> c.get(null));
        assertThrows(IllegalArgumentException.class, () -> newCache(0));
        assertThrows(IllegalArgumentException.class, () -> newCache(-5));
    }

    @Test
    void listenerReceivesEvictedValueAndAcceptsSupertypes() {
        List<Map.Entry<Object, Object>> evicted = new ArrayList<>();
        EvictionListener<Object, Object> anyListener = (k, v) -> evicted.add(Map.entry(k, v));
        Cache<String, Integer> c = newCache(1, anyListener);   // EvictionListener<? super String, ? super Integer>
        c.put("x", 1);
        c.put("y", 2);
        assertEquals(List.of(Map.entry("x", 1)), evicted);
    }

    @Test
    void neverExceedsCapacityUnderManyInserts() {
        Cache<Integer, Integer> c = newCache(100);
        for (int i = 0; i < 10_000; i++) {
            c.put(i, i);
            assertTrue(c.size() <= 100);
        }
        assertEquals(9_900, c.stats().evictions());
        assertEquals(9_900, (int) c.keysLruToMru().get(0));    // the 100 most recent survive, oldest first
    }
}
```

### `LinkedHashMapLruCacheTest.java`

```java
package com.mastery.lru;

class LinkedHashMapLruCacheTest extends CacheContractTest {
    @Override
    protected <K, V> Cache<K, V> newCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        return new LinkedHashMapLruCache<>(capacity, listener);
    }
}
```

```text
$ mvn test -Dtest=LinkedHashMapLruCacheTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
```

**Why an abstract test class?** The contract belongs to the *interface*, not to an implementation. Writing it once means #034's manual version, #035's reference model and the synchronized decorator are all held to **exactly** the same behaviour. Each new implementation costs a 6-line subclass. The factory method is generic (`<K, V> Cache<K, V> newCache(...)`), so each test can choose its own key and value types.

Tests worth noticing:

- `putOnExistingKeyReplacesRefreshesAndDoesNotEvict`: the FR-4 trap. A naive "if size == capacity, evict; then insert" implementation fails it.
- `containsKeyAndKeysDoNotChangeRecency`: proves that observing doesn't change behaviour.
- `listenerReceivesEvictedValueAndAcceptsSupertypes`: a generics test as much as a behaviour test.

---

## 4. Usage

```java
Cache<String, byte[]> thumbnails = new LinkedHashMapLruCache<>(10_000,
        (key, bytes) -> log.debug("evicted {} ({} bytes)", key, bytes.length));

byte[] img = thumbnails.get(url).orElseGet(() -> {
    byte[] loaded = render(url);          // expensive
    thumbnails.put(url, loaded);
    return loaded;
});
CacheStats s = thumbnails.stats();        // e.g. log s.hitRate() periodically
```

---

## 5. ⚠️ Known Issue (found later, in #035)

`removeEldestEntry` calls the **listener** before `LinkedHashMap` has removed the eldest entry, and *after* it has already inserted the new one. If the listener throws, the removal never happens and the cache is left holding **capacity + 1** entries (measured in #035: size 3 with capacity 2). The contract suite above doesn't cover listener failures, which is exactly the kind of gap #035's edge-case hunt is for. The fix (record the victim in `removeEldestEntry`, notify after `map.put` returns) and its regression test are in #035.

---

## Key Takeaways

1. `LinkedHashMap(…, true)` + `removeEldestEntry` is an O(1) LRU in a few lines. Wrap it (composition) so the cache's rules can't be bypassed.
2. Rejecting `null` values turns `get` into a single lookup.
3. Put the **contract in an abstract test class** and run it against every implementation.
4. Hooks like `removeEldestEntry` run in the middle of someone else's operation. Know exactly *when* (here: after the insert, before the removal), and keep the code inside them trivial.
