# #024 — Solution: Implement a Simplified HashMap

**Date:** 2026-09-24  
**Problem:** [024-implement-simplified-hashmap.md](../problems/024-implement-simplified-hashmap.md)

> Verified on Java 21: **8/8 JUnit tests pass**, including a **200 000-operation differential test** against `java.util.HashMap` (with `null` keys), and the code compiles with **zero warnings under `javac -Xlint:all`**.

---

## 1. Design, Mapped to the JDK (#020)

| Element | This implementation | `java.util.HashMap` |
|---------|--------------------|---------------------|
| Hash | `h ^ (h >>> 16)`, `null` → 0 | Identical |
| Index | `hash & (length - 1)` | Identical |
| Table | Lazy; power of two; initial capacity stashed in `threshold` | Identical |
| Bucket | Singly linked chain; cached `hash` compared before `equals` | Same, plus treeification at ≥ 8 |
| Insert position | Tail | Tail (Java 8+) |
| Resize | ×2, lo/hi split by `hash & oldCap`, order kept, no rehash | Identical (plus a tree split) |
| Iteration | Bucket by bucket, fail-fast | Same |
| Implements `Map`? | **No**, the main weakness (fixed in #025) | Yes |

`Entry` is a **static** nested class. It needs no reference to the map, and a non-static class would add a hidden `SimpleHashMap.this` field to every entry: 4–8 wasted bytes × millions of entries (EJ Item 24).

**Why `new Entry<K,V>[n]` doesn't compile:** that would be generic array creation (#026 §8). So we create a raw `new Entry[n]` and cast it. The method carries `@SuppressWarnings({"rawtypes", "unchecked"})`, which is exactly the suppression on `java.util.HashMap.resize`. It's safe because the array is private and only ever holds `Entry<K,V>`.

---

## 2. Code

### `SimpleHashMap.java`

```java
package com.mastery.hashmap;

import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.Objects;

/**
 * A simplified java.util.HashMap: separate chaining, power-of-two table, hash spreading,
 * load-factor-driven resize with the lo/hi bucket split. No treeification.
 *
 * <p>Average O(1) get/put/remove; O(n) worst case when many keys collide.
 * Permits one null key and null values. Not thread-safe. Iterators are fail-fast.
 */
public final class SimpleHashMap<K, V> implements Iterable<SimpleHashMap.Entry<K, V>> {

    static final int DEFAULT_CAPACITY = 16;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /** A key-value pair in a bucket's singly linked chain. Static: no hidden outer reference. */
    public static final class Entry<K, V> {
        final int hash;           // cached spread hash: compared first, reused on resize
        final K key;
        V value;
        Entry<K, V> next;

        Entry(int hash, K key, V value, Entry<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public K key() {
            return key;
        }

        public V value() {
            return value;
        }

        @Override
        public String toString() {
            return key + "=" + value;
        }
    }

    private Entry<K, V>[] table;     // lazily allocated; length is always a power of two
    private int size;
    private int threshold;           // resize when size exceeds this
    private final float loadFactor;
    private int modCount;

    public SimpleHashMap() {
        this(DEFAULT_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    public SimpleHashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0) {
            throw new IllegalArgumentException("Illegal capacity: " + initialCapacity);
        }
        if (!(loadFactor > 0) || Float.isNaN(loadFactor)) {
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        }
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);   // holds the initial capacity until first put
    }

    // ---- hashing -----------------------------------------------------------------------

    /** Mixes the high 16 bits into the low 16, because the index only uses the low bits. */
    static int hash(Object key) {
        int h;
        return key == null ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    /** Smallest power of two ≥ cap (1 for cap ≤ 1). */
    static int tableSizeFor(int cap) {
        int n = -1 >>> Integer.numberOfLeadingZeros(Math.max(cap, 1) - 1);
        return n < 0 ? 1 : n >= MAXIMUM_CAPACITY ? MAXIMUM_CAPACITY : n + 1;
    }

    private static int indexFor(int hash, int length) {
        return hash & (length - 1);              // == hash mod length, because length is 2^k
    }

    // ---- queries -----------------------------------------------------------------------

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    public V get(Object key) {
        Entry<K, V> e = findEntry(key);
        return e == null ? null : e.value;
    }

    public boolean containsKey(Object key) {
        return findEntry(key) != null;           // not get(key) != null: the value may be null
    }

    private Entry<K, V> findEntry(Object key) {
        if (table == null) {
            return null;
        }
        int h = hash(key);
        for (Entry<K, V> e = table[indexFor(h, table.length)]; e != null; e = e.next) {
            if (e.hash == h && Objects.equals(e.key, key)) {   // cheap int compare first, then equals
                return e;
            }
        }
        return null;
    }

    // ---- modification ------------------------------------------------------------------

    /** Associates value with key; returns the previous value, or null. */
    public V put(K key, V value) {
        if (table == null) {
            resize();                            // lazy allocation on first put
        }
        int h = hash(key);
        int i = indexFor(h, table.length);
        Entry<K, V> last = null;
        for (Entry<K, V> e = table[i]; e != null; e = e.next) {
            if (e.hash == h && Objects.equals(e.key, key)) {
                V old = e.value;
                e.value = value;                 // existing key: replace; not a structural change
                return old;
            }
            last = e;
        }
        Entry<K, V> fresh = new Entry<>(h, key, value, null);
        if (last == null) {
            table[i] = fresh;
        } else {
            last.next = fresh;                   // append at the TAIL, like Java 8+
        }
        modCount++;
        if (++size > threshold) {
            resize();
        }
        return null;
    }

    /** Removes the mapping for key; returns the removed value, or null. */
    public V remove(Object key) {
        if (table == null) {
            return null;
        }
        int h = hash(key);
        int i = indexFor(h, table.length);
        Entry<K, V> prev = null;
        for (Entry<K, V> e = table[i]; e != null; prev = e, e = e.next) {
            if (e.hash == h && Objects.equals(e.key, key)) {
                if (prev == null) {
                    table[i] = e.next;           // unlink the head
                } else {
                    prev.next = e.next;          // unlink from the middle/tail
                }
                modCount++;
                size--;
                return e.value;
            }
        }
        return null;
    }

    public void clear() {
        if (table != null && size > 0) {
            modCount++;
            java.util.Arrays.fill(table, null);
            size = 0;
        }
    }

    /**
     * Doubles the table (or allocates it). Each chain splits in one pass: nodes whose hash has the
     * new bit ({@code hash & oldCap}) clear stay at index j; the rest move to j + oldCap.
     * Relative order is preserved and no hashCode() is recomputed.
     */
    // rawtypes + unchecked: Java can't create a generic array (new Entry<K,V>[n] is illegal), so we
    // create a raw Entry[] and cast. Safe: the array is private and only ever holds Entry<K,V>.
    // (java.util.HashMap.resize carries exactly this suppression.)
    @SuppressWarnings({"rawtypes", "unchecked"})
    private void resize() {
        Entry<K, V>[] oldTab = table;
        int oldCap = oldTab == null ? 0 : oldTab.length;
        int newCap;
        if (oldCap == 0) {
            newCap = threshold > 0 ? threshold : DEFAULT_CAPACITY;   // initial capacity stashed in threshold
        } else if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;                           // can't grow; just accept collisions
            return;
        } else {
            newCap = oldCap << 1;
        }
        threshold = newCap >= MAXIMUM_CAPACITY ? Integer.MAX_VALUE : (int) (newCap * loadFactor);
        Entry<K, V>[] newTab = (Entry<K, V>[]) new Entry[newCap];    // generic array creation: raw + cast

        for (int j = 0; j < oldCap; j++) {
            Entry<K, V> e = oldTab[j];
            if (e == null) {
                continue;
            }
            oldTab[j] = null;                                        // help GC
            Entry<K, V> loHead = null, loTail = null, hiHead = null, hiTail = null;
            while (e != null) {
                Entry<K, V> next = e.next;
                if ((e.hash & oldCap) == 0) {
                    if (loTail == null) loHead = e; else loTail.next = e;
                    loTail = e;
                } else {
                    if (hiTail == null) hiHead = e; else hiTail.next = e;
                    hiTail = e;
                }
                e = next;
            }
            if (loTail != null) {
                loTail.next = null;
                newTab[j] = loHead;
            }
            if (hiTail != null) {
                hiTail.next = null;
                newTab[j + oldCap] = hiHead;
            }
        }
        table = newTab;
    }

    // ---- iteration ---------------------------------------------------------------------

    /** Iterates entries bucket by bucket (unspecified order). Fail-fast; no remove(). */
    @Override
    public Iterator<Entry<K, V>> iterator() {
        return new Iterator<>() {
            private final int expectedModCount = modCount;
            private int bucket = 0;
            private Entry<K, V> next = advance(null);

            private Entry<K, V> advance(Entry<K, V> current) {
                if (current != null && current.next != null) {
                    return current.next;                 // continue down the chain
                }
                if (table == null) {
                    return null;
                }
                while (bucket < table.length) {          // find the next non-empty bucket
                    Entry<K, V> head = table[bucket++];
                    if (head != null) {
                        return head;
                    }
                }
                return null;
            }

            @Override
            public boolean hasNext() {
                return next != null;
            }

            @Override
            public Entry<K, V> next() {
                if (modCount != expectedModCount) {
                    throw new ConcurrentModificationException();
                }
                if (next == null) {
                    throw new NoSuchElementException();
                }
                Entry<K, V> current = next;
                next = advance(current);
                return current;
            }
        };
    }

    // ---- introspection for tests / learning -----------------------------------------------

    int capacity() {
        return table == null ? 0 : table.length;
    }

    /** Length of the chain in the bucket that {@code key} maps to. */
    int chainLength(Object key) {
        if (table == null) {
            return 0;
        }
        int n = 0;
        for (Entry<K, V> e = table[indexFor(hash(key), table.length)]; e != null; e = e.next) {
            n++;
        }
        return n;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("{");
        for (Entry<K, V> e : this) {
            if (sb.length() > 1) {
                sb.append(", ");
            }
            sb.append(e);
        }
        return sb.append('}').toString();
    }
}
```

### Walkthrough of the non-obvious parts

- **`threshold` does double duty.** Before the first `put`, it holds the rounded initial capacity (`tableSizeFor`). `resize()` sees `oldCap == 0 && threshold > 0` and uses it. After that, it's the real resize threshold (`capacity × loadFactor`).
- **`containsKey` uses `findEntry(key) != null`, not `get(key) != null`.** With `put("k", null)`, `get` returns `null` although the key is present.
- **The `put` walk remembers `last`,** so appending at the tail needs no second pass.
- **Replacing a value doesn't bump `modCount`.** It's not a structural change, so live iterators stay valid (the same as the JDK).
- **The iterator snapshots `modCount` once.** Any structural change after the iterator is created makes the next `next()` throw CME.

---

## 3. Tests

### `test: SimpleHashMapTest.java`

```java
package com.mastery.hashmap;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.ArrayList;
import java.util.ConcurrentModificationException;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Random;
import java.util.Set;
import org.junit.jupiter.api.Test;

class SimpleHashMapTest {

    /** Every instance has the same hashCode: forces all keys into one chain. */
    record Colliding(int id) {
        @Override public int hashCode() { return 7; }
    }

    @Test
    void putGetReplaceRemove() {
        SimpleHashMap<String, Integer> m = new SimpleHashMap<>();
        assertNull(m.put("a", 1));
        assertNull(m.put("b", 2));
        assertEquals(1, m.put("a", 10));      // replace returns old value
        assertEquals(10, m.get("a"));
        assertEquals(2, m.size());
        assertEquals(2, m.remove("b"));
        assertNull(m.remove("b"));
        assertNull(m.get("b"));
        assertEquals(1, m.size());
    }

    @Test
    void nullKeyAndNullValues() {
        SimpleHashMap<String, String> m = new SimpleHashMap<>();
        m.put(null, "nullKey");
        m.put("k", null);
        assertEquals("nullKey", m.get(null));
        assertTrue(m.containsKey("k"));        // present even though the value is null
        assertNull(m.get("k"));
        assertFalse(m.containsKey("absent"));
        assertEquals("nullKey", m.remove(null));
        assertFalse(m.containsKey(null));
    }

    @Test
    void lazyAllocationAndResizePoints() {
        SimpleHashMap<Integer, Integer> m = new SimpleHashMap<>();
        assertEquals(0, m.capacity());
        List<String> events = new ArrayList<>();
        int last = 0;
        for (int i = 1; i <= 100; i++) {
            m.put(i, i);
            if (m.capacity() != last) {
                events.add(i + "->" + m.capacity());
                last = m.capacity();
            }
        }
        assertEquals(List.of("1->16", "13->32", "25->64", "49->128", "97->256"), events);   // same as JDK (#020)
    }

    @Test
    void initialCapacityRoundsUpToPowerOfTwo() {
        assertEquals(16, SimpleHashMap.tableSizeFor(10));
        assertEquals(16, SimpleHashMap.tableSizeFor(16));
        assertEquals(32, SimpleHashMap.tableSizeFor(17));
        assertEquals(1, SimpleHashMap.tableSizeFor(0));
        SimpleHashMap<Integer, Integer> m = new SimpleHashMap<>(100, 0.75f);
        m.put(1, 1);
        assertEquals(128, m.capacity());
        assertThrows(IllegalArgumentException.class, () -> new SimpleHashMap<>(-1, 0.75f));
        assertThrows(IllegalArgumentException.class, () -> new SimpleHashMap<>(16, Float.NaN));
        assertThrows(IllegalArgumentException.class, () -> new SimpleHashMap<>(16, 0f));
    }

    @Test
    void collisionsChainAndStayCorrect() {
        SimpleHashMap<Colliding, Integer> m = new SimpleHashMap<>();
        for (int i = 0; i < 100; i++) {
            m.put(new Colliding(i), i);
        }
        assertEquals(100, m.chainLength(new Colliding(0)));   // one bucket holds everything: O(n) lookups
        for (int i = 0; i < 100; i++) {
            assertEquals(i, m.get(new Colliding(i)));
        }
        assertEquals(50, m.remove(new Colliding(50)));       // unlink from the middle of the chain
        assertEquals(0, m.remove(new Colliding(0)));         // unlink the head
        assertEquals(99, m.remove(new Colliding(99)));       // unlink the tail
        assertEquals(97, m.size());
        assertNull(m.get(new Colliding(50)));
        assertEquals(51, m.get(new Colliding(51)));
    }

    @Test
    void resizeSplitsBucketsIntoLoAndHi() {
        SimpleHashMap<Integer, String> m = new SimpleHashMap<>(16, 0.75f);
        m.put(5, "a");
        m.put(21, "b");      // 21 & 15 == 5 → same bucket as 5
        m.put(37, "c");      // 37 & 15 == 5
        assertEquals(3, m.chainLength(5));
        for (int i = 200; i < 210; i++) {
            m.put(i, "x");   // 13 entries > threshold 12 → resize to 32; 200..209 land in buckets 8..17
        }
        assertEquals(32, m.capacity());
        assertEquals(2, m.chainLength(5));    // 5 and 37 stay (hash & 16 == 0)
        assertEquals(1, m.chainLength(21));   // 21 moved to bucket 21 (= 5 + 16)
        assertEquals("b", m.get(21));
    }

    @Test
    void iterationVisitsEveryEntryOnceAndIsFailFast() {
        SimpleHashMap<Integer, Integer> m = new SimpleHashMap<>();
        for (int i = 0; i < 1000; i++) {
            m.put(i, i * i);
        }
        Set<Integer> seen = new HashSet<>();
        for (SimpleHashMap.Entry<Integer, Integer> e : m) {
            assertTrue(seen.add(e.key()));
            assertEquals(e.key() * e.key(), e.value());
        }
        assertEquals(1000, seen.size());
        assertThrows(ConcurrentModificationException.class, () -> {
            for (SimpleHashMap.Entry<Integer, Integer> e : m) {
                m.put(-1, -1);
            }
        });
    }

    @Test
    void matchesJavaUtilHashMapUnderRandomOperations() {
        Random rnd = new Random(11);
        SimpleHashMap<Integer, Integer> mine = new SimpleHashMap<>();
        Map<Integer, Integer> reference = new HashMap<>();
        for (int step = 0; step < 200_000; step++) {
            Integer key = rnd.nextInt(10) == 0 ? null : rnd.nextInt(5_000);
            int value = rnd.nextInt();
            switch (rnd.nextInt(4)) {
                case 0, 1 -> assertEquals(reference.put(key, value), mine.put(key, value));
                case 2 -> assertEquals(reference.remove(key), mine.remove(key));
                default -> {
                    assertEquals(reference.get(key), mine.get(key));
                    assertEquals(reference.containsKey(key), mine.containsKey(key));
                }
            }
            assertEquals(reference.size(), mine.size());
        }
        Map<Integer, Integer> copied = new HashMap<>();
        mine.forEach(e -> copied.put(e.key(), e.value()));
        assertEquals(reference, copied);
    }
}
```

```text
$ mvn test -Dtest=SimpleHashMapTest
Tests run: 8, Failures: 0, Errors: 0, Skipped: 0
```

### A test bug I actually hit (worth a card, #028)

The first version of `resizeSplitsBucketsIntoLoAndHi` used keys **100–109** as filler to trigger the resize, and it failed:

```text
SimpleHashMapTest.resizeSplitsBucketsIntoLoAndHi:112 expected: <2> but was: <3>
```

The **map was right and the test was wrong**. At capacity 32, `101 & 31 = 5`, so key 101 also lands in bucket 5, and the chain holds {5, 37, **101**}. The fix was filler keys 200–209, which map to buckets 8–17. The lessons: (1) in hashing tests, compute the bucket of **every** key, including the incidental ones; (2) when a test fails, check the test before "fixing" the code. The differential test (which *did* pass) was the evidence that the implementation was fine.

---

## 4. Answer: Why Can't It Be Used as a `Map<K,V>`?

It doesn't `implement Map`, so it can't be passed to anything typed `Map` (`new TreeMap<>(m)`, `Collections.unmodifiableMap(m)`, Jackson, Spring…). It has no `keySet()`/`values()`/`entrySet()`, no `getOrDefault`/`merge`/`computeIfAbsent`, and no `equals`/`hashCode` against other maps. Its `Entry` isn't a `Map.Entry`. #025 reviews exactly this and refactors it.

---

## Key Takeaways

1. A hash map is an array of chains + a hash-spreading function + a resize rule; the JDK's version is short enough to reimplement faithfully.
2. **Power-of-two capacity** makes indexing a bit-mask and makes resize a one-bit split.
3. **Cache the hash** in the node. It speeds up lookups (int compare first) and makes resize free of `hashCode()` calls.
4. `containsKey` ≠ `get != null` when `null` values are allowed.
5. A **differential test** against `java.util.HashMap` is the fastest way to trust a reimplementation, and it tells you whether a failure is in the code or in the test.
