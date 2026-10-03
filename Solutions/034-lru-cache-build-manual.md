# #034 — Solution: Build (cont.) — LRU Cache, Manual HashMap + Doubly Linked List

**Date:** 2026-10-03  
**Problem:** [034-lru-cache-build-manual.md](../problems/034-lru-cache-build-manual.md)

> Verified on Java 21: the **unchanged** contract suite from #033 passes, **11/11**, with zero `-Xlint:all` warnings. This is the code **as built in this phase**. #035 later changes the eviction ordering (§5).

---

## 1. Code (phase-3 version)

### `ManualLruCache.java (v1)`

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
        if (index.size() == capacity) {                 // full: evict BEFORE inserting
            evictLeastRecentlyUsed();
        }
        Node<K, V> node = new Node<>(key, value);
        index.put(key, node);
        linkAtTail(node);
        return Optional.empty();
    }

    private void evictLeastRecentlyUsed() {
        Node<K, V> lru = head.next;                     // never the sentinel: capacity > 0 and the cache is full
        unlink(lru);
        index.remove(lru.key);
        stats.recordEviction();
        listener.onEviction(lru.key, lru.value);
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

### `ManualLruCacheTest.java`

```java
package com.mastery.lru;

class ManualLruCacheTest extends CacheContractTest {
    @Override
    protected <K, V> Cache<K, V> newCache(int capacity, EvictionListener<? super K, ? super V> listener) {
        return new ManualLruCache<>(capacity, listener);
    }
}
```

```text
$ mvn test -Dtest=ManualLruCacheTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
```

---

## 2. Why the Node Is `static` and Stores the Key

- **`static`:** a node needs nothing from the cache instance. A non-static inner class would give every node a hidden `ManualLruCache.this` reference: one extra pointer per entry, × capacity (EJ Item 24, #003). `LinkedHashMap.Entry` and `LinkedList.Node` are static for the same reason.
- **The key:** eviction starts from the **list** (`head.next`) and must also remove the entry from the **map**, which needs the key. Without it you'd have to search the map for the node, which is O(n).

---

## 3. Pointer Surgery: `moveToTail(B)` on `head ⇄ A ⇄ B ⇄ C ⇄ tail`

```
Start:          head ⇄ A ⇄ B ⇄ C ⇄ tail

unlink(B):      A.next = C            (node.prev.next = node.next)
                C.prev = A            (node.next.prev = node.prev)
                B.prev = B.next = null
                head ⇄ A ⇄ C ⇄ tail        B detached

linkAtTail(B):  B.prev = C            (node.prev = tail.prev)
                B.next = tail         (node.next = tail)
                C.next = B            (tail.prev.next = node)
                tail.prev = B         (tail.prev = node)
                head ⇄ A ⇄ C ⇄ B ⇄ tail    B is now MRU
```

That's 6 pointer writes in total (plus 2 nulling writes), with **no branches** thanks to the sentinels. `moveToTail` skips everything when `node.next == tail`, which is the common case of re-reading the most recent key.

**Eviction** is `unlink(head.next)` plus `index.remove(key)`. The sentinels guarantee that `head.next` is a real node whenever the cache is full and capacity > 0.

---

## 4. Comparison with the LinkedHashMap Version

| | `LinkedHashMapLruCache` | `ManualLruCache` |
|---|---|---|
| Code | ~40 lines of logic | ~90 lines of logic |
| Objects per entry | 1 (`LinkedHashMap.Entry`: hash, key, value, next, before, after) | 2 (a `HashMap.Node` + our `Node`) |
| Hash lookups per `get` hit | 1 | 1 |
| Control over semantics | Bound by `LinkedHashMap`'s hooks and their timing | Total: you choose when to evict and when to notify |
| Measured speed (#035) | 15.7–20.0 µs per 1 000 ops | 18.9–20.5 µs per 1 000 ops |

The manual version uses a little more memory (two objects per entry) and is ~0–20% slower, but it gives **full control over ordering**. That turns out to matter in #035: fixing the listener bug in the manual version was a straightforward reordering, while the `LinkedHashMap` version needed a workaround (a "pending eviction" field), because the hook's timing is fixed by the JDK.

---

## 5. ⚠️ Known Issue (found later, in #035)

`put` evicts (and **notifies the listener**) *before* inserting the new entry. If the listener throws, the LRU entry is already gone but the new entry is **never inserted**. The cache stays internally consistent (size 1, keys `[b]` in the #035 probe), but the caller's `put` silently lost data. #035 moves the notification to **after** the insertion.

---

## Key Takeaways

1. An LRU cache *is* a hash map plus a doubly linked list. Building it by hand demystifies `LinkedHashMap`'s access order.
2. **Sentinel nodes** remove all edge cases from linked-list code.
3. Static nested node classes save one reference per element.
4. The node must carry its **key**, so eviction (which starts from the list) can update the map.
5. Hand-written structures cost code and memory, and buy control over the **order of side effects**.
