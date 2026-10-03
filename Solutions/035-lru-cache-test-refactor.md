# #035 — Solution: Test & Refactor — LRU Cache

**Date:** 2026-10-03  
**Problem:** [035-lru-cache-test-refactor.md](../problems/035-lru-cache-test-refactor.md)

> Verified on Java 21, Apple M4: **52/52 tests pass**. That's the 12-test contract × 4 implementations, a multi-threaded stress test, and 3 differential tests (50 trials × 2 000 steps each). **Zero `-Xlint:all` warnings.** All numbers below are measured.

---

## 1. Reference Model

### `ReferenceLruCache.java`

```java
package com.mastery.lru;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * Deliberately simple O(n) LRU: a list of keys ordered LRU → MRU + a HashMap of values.
 * Every "use" removes the key from the list (O(n) scan + shift) and appends it.
 *
 * <p>Purpose: (1) the obviously-correct REFERENCE MODEL that differential tests compare the fast
 * implementations against; (2) the baseline in the benchmark. Not for production use.
 */
public final class ReferenceLruCache<K, V> implements Cache<K, V> {

    private final int capacity;
    private final List<K> order = new ArrayList<>();       // index 0 = LRU, last = MRU
    private final Map<K, V> values = new HashMap<>();
    private final StatsCounter stats = new StatsCounter();
    private final EvictionListener<? super K, ? super V> listener;

    public ReferenceLruCache(int capacity) {
        this(capacity, EvictionListener.none());
    }

    public ReferenceLruCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be positive: " + capacity);
        }
        this.capacity = capacity;
        this.listener = Objects.requireNonNull(listener, "listener");
    }

    private void touch(K key) {
        order.remove(key);           // O(n): find + shift
        order.add(key);
    }

    @Override
    public Optional<V> get(K key) {
        Objects.requireNonNull(key, "key");
        V v = values.get(key);
        if (v == null) {
            stats.recordMiss();
            return Optional.empty();
        }
        touch(key);
        stats.recordHit();
        return Optional.of(v);
    }

    @Override
    public Optional<V> put(K key, V value) {
        Objects.requireNonNull(key, "key");
        Objects.requireNonNull(value, "value");
        if (values.containsKey(key)) {
            touch(key);
            return Optional.of(values.put(key, value));
        }
        K evictedKey = null;
        V evictedValue = null;
        if (values.size() == capacity) {
            evictedKey = order.remove(0);
            evictedValue = values.remove(evictedKey);
        }
        order.add(key);
        values.put(key, value);
        if (evictedKey != null) {                       // notify after the cache is consistent (#035)
            stats.recordEviction();
            listener.onEviction(evictedKey, evictedValue);
        }
        return Optional.empty();
    }

    @Override
    public Optional<V> remove(K key) {
        Objects.requireNonNull(key, "key");
        V v = values.remove(key);
        if (v != null) {
            order.remove(key);
        }
        return Optional.ofNullable(v);
    }

    @Override
    public boolean containsKey(K key) {
        Objects.requireNonNull(key, "key");
        return values.containsKey(key);
    }

    @Override public int size()     { return values.size(); }
    @Override public int capacity() { return capacity; }

    @Override
    public void clear() {
        order.clear();
        values.clear();
    }

    @Override
    public List<K> keysLruToMru() {
        return new ArrayList<>(order);
    }

    @Override
    public CacheStats stats() {
        return stats.snapshot();
    }
}
```

It's deliberately boring. Every operation is "find it in a list and move it", so it's easy to convince yourself it's correct by reading it. That's the whole point of a reference model: its job is to be **obviously right**, not fast. It passes the same contract suite (`ReferenceLruCacheTest`, 12/12).

A subtle generics detail: `order.remove(key)` with `K` = `Integer` calls `remove(Object)`, **not** `remove(int index)`. Overloads are resolved at compile time against the *erased* type parameter (`Object`), so the #031 A3 trap can't happen here.

---

## 2. Differential Testing

### `DifferentialTest.java`

```java
package com.mastery.lru;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.function.BiFunction;
import org.junit.jupiter.api.Test;

/**
 * Drives the fast implementations and the obviously-correct ReferenceLruCache with the SAME random
 * operation stream and asserts identical observable behaviour after every step: return values,
 * eviction events (key AND order), sizes, LRU→MRU order and stats.
 */
class DifferentialTest {

    private static void runAgainstReference(
            BiFunction<Integer, EvictionListener<Integer, Integer>, Cache<Integer, Integer>> factory, long seed) {
        Random rnd = new Random(seed);
        for (int trial = 0; trial < 50; trial++) {
            int capacity = 1 + rnd.nextInt(20);
            int keySpace = capacity + 1 + rnd.nextInt(capacity * 2);
            List<Integer> refEvicted = new ArrayList<>();
            List<Integer> subjectEvicted = new ArrayList<>();
            Cache<Integer, Integer> ref = new ReferenceLruCache<>(capacity, (k, v) -> refEvicted.add(k));
            Cache<Integer, Integer> subject = factory.apply(capacity, (k, v) -> subjectEvicted.add(k));
            for (int step = 0; step < 2_000; step++) {
                int key = rnd.nextInt(keySpace);
                int op = rnd.nextInt(10);
                String where = "trial " + trial + " step " + step + " op " + op + " key " + key;
                if (op < 5) {
                    assertEquals(ref.get(key), subject.get(key), where);
                } else if (op < 9) {
                    int value = rnd.nextInt(1000);
                    assertEquals(ref.put(key, value), subject.put(key, value), where);
                } else {
                    assertEquals(ref.remove(key), subject.remove(key), where);
                }
                assertEquals(refEvicted, subjectEvicted, where);
                assertEquals(ref.keysLruToMru(), subject.keysLruToMru(), where);
                assertEquals(ref.stats(), subject.stats(), where);
            }
        }
    }

    @Test
    void linkedHashMapVersionMatchesReference() {
        runAgainstReference(LinkedHashMapLruCache::new, 1);
    }

    @Test
    void manualVersionMatchesReference() {
        runAgainstReference(ManualLruCache::new, 2);
    }

    @Test
    void synchronizedWrapperMatchesReference() {
        runAgainstReference((cap, l) -> new SynchronizedCache<>(new ManualLruCache<>(cap, l)), 3);
    }
}
```

Each trial picks a random capacity (1–20) and a key space a bit larger than it, so the cache is constantly full and evicting, which is where bugs live. It compares **everything observable** after **every** step, and the assertion message pinpoints the trial, step, operation and key, so a failure is reproducible from the fixed seed.

```text
Tests run: 3, Failures: 0   (LinkedHashMap, Manual, Synchronized(Manual), each 50 × 2 000 steps vs the reference)
```

---

## 3. Thread Safety

### `SynchronizedCache.java`

```java
package com.mastery.lru;

import java.util.List;
import java.util.Objects;
import java.util.Optional;

/**
 * Thread-safe DECORATOR for any Cache: every method holds the same lock (this wrapper's monitor).
 *
 * <p>Why a single lock and not a ReadWriteLock? In an LRU cache even get() is a WRITE: it moves
 * the entry to the MRU position. So readers can't share a read lock; a plain mutex is the honest
 * choice. (Highly concurrent caches like Caffeine use per-segment structures + buffered
 * recency updates instead.)
 *
 * <p>Happens-before (#012): every method enters/exits the same monitor, so all writes made inside
 * one call are visible to the next call on any thread.
 */
public final class SynchronizedCache<K, V> implements Cache<K, V> {

    private final Cache<K, V> delegate;
    private final Object lock = new Object();   // private lock: callers can't synchronize on it and cause deadlocks

    public SynchronizedCache(Cache<K, V> delegate) {
        this.delegate = Objects.requireNonNull(delegate, "delegate");
    }

    @Override public Optional<V> get(K key)            { synchronized (lock) { return delegate.get(key); } }
    @Override public Optional<V> put(K key, V value)   { synchronized (lock) { return delegate.put(key, value); } }
    @Override public Optional<V> remove(K key)         { synchronized (lock) { return delegate.remove(key); } }
    @Override public boolean containsKey(K key)        { synchronized (lock) { return delegate.containsKey(key); } }
    @Override public int size()                        { synchronized (lock) { return delegate.size(); } }
    @Override public int capacity()                    { return delegate.capacity(); }   // immutable: no lock needed
    @Override public void clear()                      { synchronized (lock) { delegate.clear(); } }
    @Override public List<K> keysLruToMru()            { synchronized (lock) { return delegate.keysLruToMru(); } }
    @Override public CacheStats stats()                { synchronized (lock) { return delegate.stats(); } }

    @Override
    public String toString() {
        synchronized (lock) {
            return "Synchronized(" + delegate + ")";
        }
    }
}
```

### `SynchronizedCacheTest.java`

```java
package com.mastery.lru;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import org.junit.jupiter.api.Test;

/** Runs the full contract through the decorator, plus a multi-threaded stress test. */
class SynchronizedCacheTest extends CacheContractTest {

    @Override
    protected <K, V> Cache<K, V> newCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        return new SynchronizedCache<>(new ManualLruCache<>(capacity, listener));
    }

    @Test
    void staysConsistentUnderConcurrentLoad() throws Exception {
        int threads = 8, opsPerThread = 200_000, capacity = 500;
        AtomicLong listenerCalls = new AtomicLong();
        Cache<Integer, Integer> cache = new SynchronizedCache<>(
                new ManualLruCache<>(capacity, (k, v) -> listenerCalls.incrementAndGet()));
        AtomicLong gets = new AtomicLong();
        CountDownLatch start = new CountDownLatch(1);
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        List<Future<?>> futures = new java.util.ArrayList<>();
        for (int t = 0; t < threads; t++) {
            futures.add(pool.submit(() -> {
                start.await();                                       // all threads start together
                ThreadLocalRandom rnd = ThreadLocalRandom.current();
                for (int i = 0; i < opsPerThread; i++) {
                    int key = rnd.nextInt(2_000);
                    if (rnd.nextInt(10) < 7) {
                        cache.get(key);
                        gets.incrementAndGet();
                    } else {
                        cache.put(key, key);
                    }
                }
                return null;
            }));
        }
        start.countDown();
        for (Future<?> f : futures) {
            f.get();                                                 // rethrows any exception from a worker
        }
        pool.shutdown();
        assertTrue(pool.awaitTermination(10, TimeUnit.SECONDS));

        CacheStats s = cache.stats();
        assertEquals(gets.get(), s.requests());                      // no lost hit/miss updates
        assertEquals(listenerCalls.get(), s.evictions());            // every eviction reported exactly once
        List<Integer> keys = cache.keysLruToMru();
        assertEquals(cache.size(), keys.size());                     // map and list agree
        assertEquals(keys.size(), new HashSet<>(keys).size());       // no duplicate nodes
        assertTrue(cache.size() <= capacity);
        Set<Integer> fromList = new HashSet<>(keys);
        for (Integer k : fromList) {
            assertTrue(cache.containsKey(k));
        }
    }
}
```

Design notes:

- **A decorator:** it works with any `Cache`, and the single-threaded implementations stay lock-free.
- **A private lock object** rather than `synchronized` methods. Code outside can't lock our monitor (which could cause a deadlock, or starve the cache).
- **Why not a `ReadWriteLock`:** `get` reorders the list, so it's a write (#032 §5).
- **`capacity()` takes no lock:** it's immutable after construction (a `final` field read).
- **The stress test's latch** releases all 8 threads at once to maximise interleaving. `future.get()` rethrows any worker exception, so a crash inside a thread fails the test instead of being swallowed. The invariants catch lost updates (stats), double-linking (duplicate keys) and map/list divergence (sizes).

```text
SynchronizedCacheTest: Tests run: 13, Failures: 0   (12 contract tests + the 8-thread × 200 000-op stress test)
```

---

## 4. Edge-Case Hunt: A Throwing Eviction Listener 🐛

### The probe

What happens when user code inside the cache fails? A listener that always throws, on a capacity-2 cache holding `a`, `b`, then `put("c", 3)`:

```text
                put(c)   size  capacity  keys        evictions
LinkedHashMap   threw    3     2         [a, b, c]   1          ✗ INVARIANT BROKEN: size > capacity
Manual          threw    1     2         [b]         1          ✗ "c" silently not stored, "a" gone
Reference       threw    1     2         [b]         1          ✗ same as Manual
```

### Root causes

| Implementation | Sequence inside `put` | Failure point |
|----------------|----------------------|---------------|
| LinkedHashMap | `LinkedHashMap.putVal` inserts **c** → `afterNodeInsertion` → our `removeEldestEntry` → **listener throws** | `removeEldestEntry` never returns `true`, so **a** is never removed: 3 entries |
| Manual / Reference | evict **a** → **listener throws** → (insert c never runs) | The cache is consistent, but the `put` was half-done: an entry was evicted for an insert that never happened |

None of the 11 contract tests covered this: every test listener was well-behaved. It's a classic gap, because **user callbacks are inputs too**.

### The decision

> Finish the cache mutation completely (evict + insert), and **only then** notify the listener. If the listener throws, the exception propagates to the caller (it isn't silently swallowed), but the cache is already in its correct post-`put` state.

Alternatives considered: *swallow and log* listener exceptions (Caffeine logs them), which was rejected because this library has no logging dependency and silent failure hides bugs; *wrap in a custom unchecked exception*, which adds API surface for no benefit.

### Regression test (added to the contract, so all 4 implementations get it)

```java
@Test
void throwingListenerLeavesCacheConsistent() {
    // Regression test for the bug found in #035: a listener exception used to leave the
    // LinkedHashMap version with capacity + 1 entries, and the others without the new entry.
    EvictionListener<String, Integer> boom = (k, v) -> {
        throw new IllegalStateException("listener failed for " + k);
    };
    Cache<String, Integer> c = newCache(2, boom);
    c.put("a", 1);
    c.put("b", 2);
    IllegalStateException e = assertThrows(IllegalStateException.class, () -> c.put("c", 3));
    assertEquals("listener failed for a", e.getMessage());    // the caller still learns about it
    assertEquals(2, c.size());                                 // capacity respected
    assertEquals(List.of("b", "c"), c.keysLruToMru());         // a evicted, c stored
    assertEquals(Optional.of(3), c.get("c"));
    assertEquals(1, c.stats().evictions());
}
```

### The fixes

**LinkedHashMap version.** The hook's timing is fixed by the JDK, so `removeEldestEntry` now only **records** the victim, and `put` notifies after `map.put` has returned (by which time the eldest entry has been removed):

```java
protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    if (size() > LinkedHashMapLruCache.this.capacity) {
        pendingEviction = Map.entry(eldest.getKey(), eldest.getValue());
        return true;
    }
    return false;
}

public Optional<V> put(K key, V value) {
    Objects.requireNonNull(key, "key");
    Objects.requireNonNull(value, "value");
    V previous = map.put(key, value);              // existing key: moved to tail; new key: may evict
    Map.Entry<K, V> evicted = pendingEviction;
    if (evicted != null) {                         // the cache is consistent again: now it's safe to notify
        pendingEviction = null;
        stats.recordEviction();
        listener.onEviction(evicted.getKey(), evicted.getValue());
    }
    return Optional.ofNullable(previous);
}
```

**Manual version.** We own the ordering, so: detach the LRU, insert the new node, then notify:

```java
Node<K, V> evicted = null;
if (index.size() == capacity) {                 // full: make room BEFORE inserting
    evicted = detachLeastRecentlyUsed();
}
Node<K, V> node = new Node<>(key, value);
index.put(key, node);
linkAtTail(node);
if (evicted != null) {                          // notify only once the cache is consistent (#035)
    stats.recordEviction();
    listener.onEviction(evicted.key, evicted.value);
}
return Optional.empty();
```

The **Reference** version got the same reordering. After the fix, the probe shows all three in the same, correct state:

```text
LinkedHashMap put(c) threw -> size 2 (capacity 2), keys [b, c], evictions 1
Manual        put(c) threw -> size 2 (capacity 2), keys [b, c], evictions 1
Reference     put(c) threw -> size 2 (capacity 2), keys [b, c], evictions 1
```

### Final implementations

### `LinkedHashMapLruCache.java (final)`

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
    private Map.Entry<K, V> pendingEviction;                          // set by removeEldestEntry, consumed by put

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
                    // Don't call the listener here: LinkedHashMap has ALREADY inserted the new entry and
                    // removes `eldest` only after we return true. A throwing listener would skip that
                    // removal and leave capacity + 1 entries (found in #035). Record it; notify later.
                    pendingEviction = Map.entry(eldest.getKey(), eldest.getValue());
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
        V previous = map.put(key, value);              // existing key: moved to tail; new key: may evict
        Map.Entry<K, V> evicted = pendingEviction;
        if (evicted != null) {                         // the cache is consistent again: now it's safe to notify
            pendingEviction = null;
            stats.recordEviction();
            listener.onEviction(evicted.getKey(), evicted.getValue());
        }
        return Optional.ofNullable(previous);
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

### `ManualLruCache.java (final)`

```java
package com.mastery.lru;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

/**
 * LRU cache built from scratch: a HashMap for O(1) lookup + a doubly linked list for O(1)
 * recency updates. This is what LinkedHashMap does internally (#023), written out by hand.
 *
 * <pre>
 *   head ⇄ [LRU] ⇄ ... ⇄ [MRU] ⇄ tail        (head/tail are sentinels: never hold data)
 *   map: key → node, so any node can be found and unlinked in O(1)
 * </pre>
 *
 * Not thread-safe: wrap with {@link SynchronizedCache} to share.
 */
public final class ManualLruCache<K, V> implements Cache<K, V> {

    /** Static: a node needs no reference to the cache (EJ Item 24). */
    private static final class Node<K, V> {
        final K key;            // kept in the node so eviction can remove the map entry
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> index;
    private final Node<K, V> head = new Node<>(null, null);   // sentinel before the LRU node
    private final Node<K, V> tail = new Node<>(null, null);   // sentinel after the MRU node
    private final StatsCounter stats = new StatsCounter();
    private final EvictionListener<? super K, ? super V> listener;

    public ManualLruCache(int capacity) {
        this(capacity, EvictionListener.none());
    }

    public ManualLruCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be positive: " + capacity);
        }
        this.capacity = capacity;
        this.listener = Objects.requireNonNull(listener, "listener");
        this.index = HashMap.newHashMap(capacity);     // Java 19+: sized for `capacity` entries, never resizes
        head.next = tail;
        tail.prev = head;
    }

    // ---- list primitives: each is O(1) and touches at most 4 pointers --------------------

    private void unlink(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
        node.prev = null;                    // help GC and catch bugs (a stale node can't be relinked twice)
        node.next = null;
    }

    private void linkAtTail(Node<K, V> node) {          // tail side = most recently used
        node.prev = tail.prev;
        node.next = tail;
        tail.prev.next = node;
        tail.prev = node;
    }

    private void moveToTail(Node<K, V> node) {
        if (node.next != tail) {             // already MRU? nothing to do
            unlink(node);
            linkAtTail(node);
        }
    }

    // ---- Cache API ---------------------------------------------------------------------------

    @Override
    public Optional<V> get(K key) {
        Objects.requireNonNull(key, "key");
        Node<K, V> node = index.get(key);
        if (node == null) {
            stats.recordMiss();
            return Optional.empty();
        }
        moveToTail(node);
        stats.recordHit();
        return Optional.of(node.value);
    }

    @Override
    public Optional<V> put(K key, V value) {
        Objects.requireNonNull(key, "key");
        Objects.requireNonNull(value, "value");
        Node<K, V> existing = index.get(key);
        if (existing != null) {                         // update: replace value + refresh recency
            V old = existing.value;
            existing.value = value;
            moveToTail(existing);
            return Optional.of(old);
        }
        Node<K, V> evicted = null;
        if (index.size() == capacity) {                 // full: make room BEFORE inserting
            evicted = detachLeastRecentlyUsed();
        }
        Node<K, V> node = new Node<>(key, value);
        index.put(key, node);
        linkAtTail(node);
        if (evicted != null) {                          // notify only once the cache is consistent (#035)
            stats.recordEviction();
            listener.onEviction(evicted.key, evicted.value);
        }
        return Optional.empty();
    }

    /** Removes the LRU node from both structures and returns it (key/value intact); does NOT notify. */
    private Node<K, V> detachLeastRecentlyUsed() {
        Node<K, V> lru = head.next;                     // never the sentinel: capacity > 0 and the cache is full
        unlink(lru);                                    // clears prev/next only
        index.remove(lru.key);
        return lru;
    }

    @Override
    public Optional<V> remove(K key) {
        Objects.requireNonNull(key, "key");
        Node<K, V> node = index.remove(key);
        if (node == null) {
            return Optional.empty();
        }
        unlink(node);
        return Optional.of(node.value);
    }

    @Override
    public boolean containsKey(K key) {
        Objects.requireNonNull(key, "key");
        return index.containsKey(key);                  // peek: no moveToTail
    }

    @Override
    public int size() {
        return index.size();
    }

    @Override
    public int capacity() {
        return capacity;
    }

    @Override
    public void clear() {
        index.clear();
        head.next = tail;                               // drop the whole chain; GC reclaims the nodes
        tail.prev = head;
    }

    @Override
    public List<K> keysLruToMru() {
        List<K> keys = new ArrayList<>(index.size());
        for (Node<K, V> n = head.next; n != tail; n = n.next) {
            keys.add(n.key);
        }
        return keys;
    }

    @Override
    public CacheStats stats() {
        return stats.snapshot();
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("ManualLruCache{");
        for (Node<K, V> n = head.next; n != tail; n = n.next) {
            if (n != head.next) {
                sb.append(", ");
            }
            sb.append(n.key).append('=').append(n.value);
        }
        return sb.append('}').toString();
    }
}
```

---

## 5. Performance (JMH)

### `LruBenchmark.java`

```java
package com.mastery.lru.bench;

import com.mastery.lru.Cache;
import com.mastery.lru.LinkedHashMapLruCache;
import com.mastery.lru.ManualLruCache;
import com.mastery.lru.ReferenceLruCache;
import com.mastery.lru.SynchronizedCache;
import java.util.SplittableRandom;
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Param;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Warmup;

/**
 * 1 000 mixed operations per invocation: 80% get / 20% put, keys uniform over 2 × capacity
 * (so the cache is always full and roughly half the gets hit). Same pre-generated operation
 * stream for every implementation (fixed seed).
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(value = 1, jvmArgsAppend = {"-Xms2g", "-Xmx2g"})
@State(Scope.Thread)
public class LruBenchmark {

    @Param({"1000", "100000"})
    int capacity;

    @Param({"LinkedHashMap", "Manual", "SynchronizedManual", "Reference"})
    String impl;

    static final int OPS = 1_000;

    Cache<Integer, Integer> cache;
    int[] keys;
    boolean[] isGet;
    int cursor;

    @Setup
    public void setUp() {
        cache = switch (impl) {
            case "LinkedHashMap" -> new LinkedHashMapLruCache<>(capacity);
            case "Manual" -> new ManualLruCache<>(capacity);
            case "SynchronizedManual" -> new SynchronizedCache<>(new ManualLruCache<>(capacity));
            case "Reference" -> new ReferenceLruCache<>(capacity);
            default -> throw new IllegalArgumentException(impl);
        };
        SplittableRandom rnd = new SplittableRandom(42);
        for (int i = 0; i < capacity; i++) {
            cache.put(rnd.nextInt(2 * capacity), i);          // start full
        }
        int n = 1 << 16;                                       // a long pre-generated stream, cycled
        keys = new int[n];
        isGet = new boolean[n];
        for (int i = 0; i < n; i++) {
            keys[i] = rnd.nextInt(2 * capacity);
            isGet[i] = rnd.nextInt(10) < 8;
        }
    }

    @Benchmark
    public long mixedOps() {
        long acc = 0;
        int c = cursor;
        for (int i = 0; i < OPS; i++) {
            int idx = c++ & (keys.length - 1);
            int k = keys[idx];
            if (isGet[idx]) {
                acc += cache.get(k).orElse(0);
            } else {
                cache.put(k, k);
            }
        }
        cursor = c;
        return acc;
    }
}
```

### Results: µs per 1 000 mixed operations (lower is better; ± = 99.9% CI)

| Implementation | capacity 1 000 | capacity 100 000 | Per operation |
|----------------|---------------:|-----------------:|--------------:|
| `LinkedHashMapLruCache` | 19.3 ± 0.6 | 15.8 ± 1.0 | **≈ 16–19 ns** |
| `ManualLruCache` | 20.4 ± 0.4 | 19.3 ± 3.4 | ≈ 19–20 ns |
| `SynchronizedCache(Manual)` | 23.3 ± 0.6 | 22.1 ± 0.6 | ≈ 22–23 ns |
| `ReferenceLruCache` (O(n)) | 133.8 ± 2.5 | **28 463.7 ± 6 688.4** | 0.13 µs → **28 µs** |

(Measured before the listener fix. Re-running the two fixed implementations afterwards gave LinkedHashMap 20.0 / 15.7 and Manual 20.5 / 18.9, so **the fix cost nothing measurable**.)

### Reading the results

1. **O(1) holds:** both real implementations stay flat (even slightly faster at 100k, within noise) while the cache grows 100×.
2. **LinkedHashMap ≥ Manual:** it's one object per entry instead of two (`HashMap.Node` + our `Node`), so there are fewer pointer hops and less allocation. The JDK version is tuned; the hand-written one is a learning tool.
3. **Uncontended locking costs ~3 ns per operation (~15%).** A cheap insurance policy if the cache is shared.
4. **The O(n) reference grew 213× for 100× more elements.** That's more than linear, because at 100k the list of keys (~400 KB of references plus the `Integer`s they point to) no longer fits in the fast caches, so every `indexOf` scan also pays memory latency. The same effect showed up in #018.

---

## 6. Is an LRU Cache Worth It? Hit-Rate Experiment

### `HitRateExperiment.java`

```java
package com.mastery.lru.bench;

import com.mastery.lru.Cache;
import com.mastery.lru.ManualLruCache;
import java.util.SplittableRandom;

/**
 * How useful is an LRU cache? Hit rate vs cache size for two traffic shapes over 100 000 keys:
 * uniform (every key equally likely) and Zipf(s = 1.0) (a few keys are very popular, like real
 * web/product/user traffic). 2 000 000 lookups each, read-through: a miss loads the key.
 */
public final class HitRateExperiment {

    private static final int KEYS = 100_000;
    private static final int REQUESTS = 2_000_000;

    /** Zipf sampler: precomputed CDF + binary search. P(rank r) ∝ 1 / r^s. */
    static final class Zipf {
        private final double[] cdf;

        Zipf(int n, double s) {
            cdf = new double[n];
            double sum = 0;
            for (int r = 1; r <= n; r++) {
                sum += 1.0 / Math.pow(r, s);
                cdf[r - 1] = sum;
            }
            for (int i = 0; i < n; i++) {
                cdf[i] /= sum;
            }
        }

        int next(SplittableRandom rnd) {
            double u = rnd.nextDouble();
            int lo = 0, hi = cdf.length - 1;
            while (lo < hi) {
                int mid = (lo + hi) >>> 1;
                if (cdf[mid] < u) lo = mid + 1; else hi = mid;
            }
            return lo;
        }
    }

    interface KeySource { int next(SplittableRandom rnd); }

    static double hitRate(int capacity, KeySource source) {
        Cache<Integer, Integer> cache = new ManualLruCache<>(capacity);
        SplittableRandom rnd = new SplittableRandom(7);
        for (int i = 0; i < REQUESTS; i++) {
            int key = source.next(rnd);
            if (cache.get(key).isEmpty()) {
                cache.put(key, key);                        // read-through: load on miss
            }
        }
        return cache.stats().hitRate();
    }

    public static void main(String[] args) {
        Zipf zipf = new Zipf(KEYS, 1.0);
        System.out.println("cache size (% of 100k keys) | uniform hit rate | Zipf(1.0) hit rate");
        for (double pct : new double[]{0.1, 1, 5, 10, 25, 50}) {
            int capacity = (int) (KEYS * pct / 100);
            double uniform = hitRate(capacity, rnd -> rnd.nextInt(KEYS));
            double skewed = hitRate(capacity, zipf::next);
            System.out.printf("%8d (%5.1f%%)            | %6.1f%%           | %6.1f%%%n",
                    capacity, pct, uniform * 100, skewed * 100);
        }
    }
}
```

### Results (100 000 keys, 2 000 000 read-through lookups)

| Cache size (% of keys) | Uniform traffic | Zipf(1.0) traffic |
|-----------------------:|----------------:|------------------:|
| 100 (0.1%) | 0.1% | **29.0%** |
| 1 000 (1%) | 1.0% | **50.6%** |
| 5 000 (5%) | 5.0% | 66.5% |
| 10 000 (10%) | 10.0% | 73.6% |
| 25 000 (25%) | 24.8% | 83.3% |
| 50 000 (50%) | 49.3% | 90.6% |

**Conclusion:** under uniform traffic, an LRU cache's hit rate just equals the fraction of keys it holds, so caching buys almost nothing. Under **skewed** traffic (Zipf: a few items are very popular, which is typical of real users, products and pages), caching **1% of the keys serves half of all requests**. **Measure your key distribution before sizing a cache.** The `stats().hitRate()` method exists for exactly this.

---

## 7. Refactor Log

| # | Trigger | Change | Proof |
|---|---------|--------|-------|
| 1 | Edge-case probe (listener throws) | All 3 implementations: **notify after mutation**. LinkedHashMap version: `pendingEviction` hand-off | New contract test `throwingListenerLeavesCacheConsistent`; probe before/after (§4) |
| 2 | Code review of fix #1 | `detachLeastRecentlyUsed` first returned a **copy** of the node (a needless allocation per eviction); changed to return the unlinked node itself, since `unlink` clears only the links | All 52 tests still green |
| 3 | Design (#032 §5) | Thread safety as a **decorator**, not inside each implementation | The contract suite runs through it; stress test |
| 4 | Testing strategy | Contract tests in an **abstract base class** + a reference model + differential tests | 4 implementations × 12 tests; 3 × 100 000 differential steps |
| 5 | Lint | `-Xlint:all` in `pom.xml` `compilerArgs` | 0 warnings |

---

## 8. Project Build

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mastery</groupId>
    <artifactId>lru-cache</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.11.4</junit.version>
        <jmh.version>1.37</jmh.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>${jmh.version}</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>${jmh.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <compilerArgs>
                        <arg>-Xlint:all</arg>
                    </compilerArgs>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.openjdk.jmh</groupId>
                            <artifactId>jmh-generator-annprocess</artifactId>
                            <version>${jmh.version}</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <finalName>benchmarks</finalName>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>org.openjdk.jmh.Main</mainClass>
                                </transformer>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

```text
$ mvn package
Tests run: 52, Failures: 0, Errors: 0, Skipped: 0
$ java -jar target/benchmarks.jar LruBenchmark
$ java -cp target/benchmarks.jar com.mastery.lru.bench.HitRateExperiment
```

---

## Key Takeaways

1. **A reference model + differential testing** turns "I tested the cases I thought of" into "it behaves identically to an obviously correct model on 100 000 random steps".
2. **User callbacks are inputs.** Test what happens when they throw. Mutate first, notify after.
3. Hooks you don't own (`removeEldestEntry`) have **fixed timing**. Keep side effects out of them; record and act afterwards.
4. Thread safety via a **decorator with a private lock**; read-write locks don't help when reads mutate.
5. O(1) vs O(n) at 100k entries: **~1 800×**. Uncontended locking: ~15%.
6. **The key distribution decides cache value:** 1% of the keys can serve 50% of traffic, or 1%.
