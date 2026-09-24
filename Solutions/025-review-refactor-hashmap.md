# #025 — Solution: Review Previous Day's Build; Refactor Weak Spots

**Date:** 2026-09-24  
**Problem:** [025-review-refactor-hashmap.md](../problems/025-review-refactor-hashmap.md)

> Verified on Java 21 (Apple M4): **7/7 new tests + all 8 of #024's tests pass**, including a 100 000-operation differential test against `java.util.HashMap` that exercises views, `merge` and bulk `removeIf`. The whole project compiles with **zero `-Xlint:all` warnings**. The collision numbers below were measured.

---

## Part 1 — Review of `SimpleHashMap` (#024)

| # | Question | Verdict | Finding → fix |
|---|----------|---------|---------------|
| R1 | Usable as a `Map`? | ❌ **Finding (high)** | Doesn't implement `Map`: no views, no `equals`/`hashCode` with other maps, no default methods. **Fix:** `extends AbstractMap` |
| R2 | Filter in place while iterating? | ❌ **Finding (medium)** | The iterator has no `remove()`, so callers must collect keys and remove them afterwards. **Fix:** a live `entrySet` with an iterator `remove()` |
| R3 | Entry contract | ❌ **Finding (medium)** | `Entry` isn't a `Map.Entry`: no `setValue`, no `equals`/`hashCode`. A `Set` of these entries couldn't equal another map's `entrySet()`. **Fix:** `Node implements Map.Entry` with `key^value` hashing |
| R4 | Worst case with colliding keys | ❌ **Finding (high, measured below)** | Chains become O(n): up to **737×** slower than the JDK at 50k keys. **Decision:** measured and documented, not fixed yet (see Part 3) |
| R5 | Duplicated chain walking | ⚠️ Minor | `findEntry`, `put` and `remove` each walk a chain. **Fix:** a shared `findNode(key, hash)`; `remove` keeps its own walk because it needs the predecessor |
| R6 | Constructor validation | ✅ OK | Negative capacity, a load factor ≤ 0 and `NaN` are all rejected (`!(lf > 0)` is also true for `NaN`) |
| R7 | Memory: does it ever shrink? | ⚠️ Accepted | The table never shrinks, so iterating after `clear()` still scans every bucket. That's the same as the JDK. **Documented; the fix is to create a new map** |
| R8 | Tests trustworthy? | ✅ with a note | The differential test compares against `java.util.HashMap`. One test *did* have a bug (the filler key in bucket 5, #024 §3), and the differential test is what proved the map itself was correct |

Also found by `javac -Xlint:all` while refactoring:

- **`[rawtypes]`** on `new Entry[n]`: the suppression covered only `unchecked`. Fixed with `@SuppressWarnings({"rawtypes","unchecked"})` plus a justification comment.
- **`[this-escape]`** (a Java 21 lint) on `BetterHashMap`'s first draft: *"possible 'this' escape before subclass is fully initialized"*, because the copy constructor calls the overridable `putAll`. That's **EJ Item 19** (#003) detected by the compiler. **Fix:** make the class `final`, since it isn't designed for extension. `SimpleHashMap` and `DynamicArray` were made `final` for the same reason.

---

## Part 2 — Measure Before Fixing (R4)

### `CollisionCost.java`

```java
package com.mastery.hashmap;

import java.util.HashMap;
import java.util.Map;

/**
 * Rough measurement (nanoTime, not JMH; fine for 10×+ differences) of the "all keys collide"
 * weak spot: SimpleHashMap chains → O(n) per lookup; java.util.HashMap treeifies → O(log n).
 */
public final class CollisionCost {
    /** Same hashCode for every instance, but Comparable, so the JDK's tree bins can order them. */
    record BadKey(int id) implements Comparable<BadKey> {
        @Override public int hashCode() { return 42; }
        @Override public int compareTo(BadKey o) { return Integer.compare(id, o.id); }
    }

    public static void main(String[] args) {
        for (int n : new int[]{1_000, 10_000, 50_000}) {
            SimpleHashMap<BadKey, Integer> simple = new SimpleHashMap<>();
            Map<BadKey, Integer> jdk = new HashMap<>();
            long t0 = System.nanoTime();
            for (int i = 0; i < n; i++) simple.put(new BadKey(i), i);
            long simplePut = System.nanoTime() - t0;
            t0 = System.nanoTime();
            for (int i = 0; i < n; i++) jdk.put(new BadKey(i), i);
            long jdkPut = System.nanoTime() - t0;

            long simpleGet = time(() -> { long s = 0; for (int i = 0; i < 1000; i++) s += simple.get(new BadKey((int) ((long) i * n / 1000))); return s; });
            long jdkGet = time(() -> { long s = 0; for (int i = 0; i < 1000; i++) s += jdk.get(new BadKey((int) ((long) i * n / 1000))); return s; });
            System.out.printf("n=%,6d | build: Simple %,8.1f ms  JDK %,6.1f ms | 1000 gets: Simple %,9.1f µs  JDK %,6.1f µs  (%.0f× slower)%n",
                    n, simplePut / 1e6, jdkPut / 1e6, simpleGet / 1e3, jdkGet / 1e3, (double) simpleGet / jdkGet);
        }
    }

    interface Work { long run(); }

    /** Best of 20 runs after warm-up. */
    static long time(Work w) {
        long sink = 0, best = Long.MAX_VALUE;
        for (int i = 0; i < 200; i++) sink += w.run();          // warm-up for the JIT
        for (int i = 0; i < 20; i++) {
            long t = System.nanoTime();
            sink += w.run();
            best = Math.min(best, System.nanoTime() - t);
        }
        if (sink == 42) System.out.print("");                     // keep the result alive
        return best;
    }
}
```

### Results (Apple M4, JDK 21; best of 20 after warm-up; `nanoTime`, which is enough for differences this large)

```text
n= 1,000 | build: Simple     25.8 ms  JDK    2.1 ms | 1000 gets: Simple     768.3 µs  JDK   30.8 µs  (25× slower)
n=10,000 | build: Simple    106.3 ms  JDK    3.3 ms | 1000 gets: Simple   9,934.1 µs  JDK   47.9 µs  (208× slower)
n=50,000 | build: Simple  2,234.8 ms  JDK    6.0 ms | 1000 gets: Simple  45,117.1 µs  JDK   61.2 µs  (737× slower)
```

| | SimpleHashMap | java.util.HashMap |
|---|---|---|
| Lookup cost | Grows **linearly**: 0.8 → 9.9 → 45 ms for ×10 and ×5 more keys → **O(n)** per lookup (one chain) | Barely grows: 31 → 48 → 61 µs → **O(log n)** (a red-black tree bin, since `BadKey` is `Comparable`) |
| Build cost | **Quadratic**: each `put` walks the whole chain to check for an existing key → 2.2 s at 50k | Near-linear |

**A measurement mistake I caught:** the first version sampled lookups with `i * 7 % n`. For i < 1 000 that only reaches keys < 7 000, so the 50k map looked *faster* than the 10k map (6.1 ms vs 7.5 ms), which is impossible for O(n) lookups. Spreading the lookups over the whole range (`i * n / 1000`) fixed it. When a result contradicts the theory, **suspect the benchmark first**.

**Why it matters:** this isn't only about badly written `hashCode()`s. **Hash flooding** is a real denial-of-service attack. An attacker sends many keys with colliding hashes (e.g. crafted `String`s, since `"Aa".hashCode() == "BB".hashCode()`) as request parameters, turning each lookup into a linear scan. Java 8's treeification was the defence.

---

## Part 3 — The Refactor: `BetterHashMap`

### What `AbstractMap` gives you, and the trap

`AbstractMap` requires **only `entrySet()`**. Everything else has a default built on it: `equals`, `hashCode`, `toString`, `keySet()`, `values()`, `containsValue`, `putAll`, and (via `Map`'s defaults) `getOrDefault`, `merge`, `compute*` and `replaceAll`.

**The trap:** those defaults are **O(n) scans**. From the JDK 21 source:

```java
// java.util.AbstractMap
public V get(Object key) {
    Iterator<Entry<K,V>> i = entrySet().iterator();
    ...                               // linear search through every entry!
}
public V put(K key, V value) {
    throw new UnsupportedOperationException();
}
```

So `BetterHashMap` **overrides** `get`, `containsKey`, `put`, `remove`, `clear` and `size` with the O(1) hash-table versions, and its `EntrySet` overrides `contains`/`remove`, which `AbstractCollection` would also do by scanning.

### `BetterHashMap.java`

```java
package com.mastery.hashmap;

import java.util.AbstractMap;
import java.util.AbstractSet;
import java.util.Arrays;
import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.Map;
import java.util.NoSuchElementException;
import java.util.Objects;
import java.util.Set;

/**
 * Refactor of {@link SimpleHashMap} after review (row 25): now a real {@link java.util.Map}.
 *
 * <ul>
 *   <li>extends {@link AbstractMap}: equals/hashCode/toString, keySet/values views and every
 *       Map default method (getOrDefault, merge, computeIfAbsent, ...) for free;</li>
 *   <li>overrides get/containsKey/put/remove/clear/size, because AbstractMap's versions are
 *       O(n) scans of entrySet();</li>
 *   <li>entrySet() is a live view whose iterator supports remove();</li>
 *   <li>Node implements Map.Entry with a contract-correct equals/hashCode and setValue;</li>
 *   <li>one chain-walking helper (findNode) shared by all lookups.</li>
 * </ul>
 * Still no treeification: see the review for why that was deliberately deferred.
 */
public final class BetterHashMap<K, V> extends AbstractMap<K, V> {   // final: not designed for extension (EJ Item 19)

    static final int DEFAULT_CAPACITY = 16;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int MAXIMUM_CAPACITY = 1 << 30;

    static final class Node<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        @Override public K getKey()   { return key; }
        @Override public V getValue() { return value; }

        @Override
        public V setValue(V newValue) {       // write-through: changes the map's value
            V old = value;
            value = newValue;
            return old;
        }

        // Map.Entry contract: equal to ANY Map.Entry with equal key and value...
        @Override
        public boolean equals(Object o) {
            return o instanceof Map.Entry<?, ?> e
                    && Objects.equals(key, e.getKey())
                    && Objects.equals(value, e.getValue());
        }

        // ...and hashCode is keyHash XOR valueHash, so entries from different Map impls agree.
        @Override
        public int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        @Override
        public String toString() {
            return key + "=" + value;
        }
    }

    private Node<K, V>[] table;
    private int size;
    private int threshold;
    private final float loadFactor;
    private int modCount;
    private Set<Map.Entry<K, V>> entrySet;   // cached view (views are stateless wrappers)

    public BetterHashMap() {
        this(DEFAULT_CAPACITY, DEFAULT_LOAD_FACTOR);
    }

    public BetterHashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0) {
            throw new IllegalArgumentException("Illegal capacity: " + initialCapacity);
        }
        if (!(loadFactor > 0) || Float.isNaN(loadFactor)) {
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        }
        this.loadFactor = loadFactor;
        this.threshold = SimpleHashMap.tableSizeFor(initialCapacity);
    }

    /** Copy constructor: pre-sizes so copying never resizes (the newHashMap lesson from #020). */
    public BetterHashMap(Map<? extends K, ? extends V> source) {
        this((int) Math.ceil(source.size() / (double) DEFAULT_LOAD_FACTOR), DEFAULT_LOAD_FACTOR);
        putAll(source);
    }

    private static int hash(Object key) {
        return SimpleHashMap.hash(key);
    }

    // ---- the O(1) core: override AbstractMap's O(n) defaults ------------------------------

    @Override
    public int size() {
        return size;
    }

    @Override
    public V get(Object key) {
        Node<K, V> n = findNode(key);
        return n == null ? null : n.value;
    }

    @Override
    public boolean containsKey(Object key) {
        return findNode(key) != null;
    }

    @Override
    public V put(K key, V value) {
        if (table == null) {
            resize();
        }
        int h = hash(key);
        Node<K, V> existing = findNode(key, h);
        if (existing != null) {
            return existing.setValue(value);
        }
        int i = h & (table.length - 1);
        Node<K, V> fresh = new Node<>(h, key, value, null);
        Node<K, V> tail = table[i];
        if (tail == null) {
            table[i] = fresh;
        } else {
            while (tail.next != null) {
                tail = tail.next;
            }
            tail.next = fresh;
        }
        modCount++;
        if (++size > threshold) {
            resize();
        }
        return null;
    }

    @Override
    public V remove(Object key) {
        Node<K, V> n = removeNode(key);
        return n == null ? null : n.value;
    }

    @Override
    public void clear() {
        if (table != null && size > 0) {
            modCount++;
            Arrays.fill(table, null);
            size = 0;
        }
    }

    // ---- single implementation of chain walking ----------------------------------------------

    private Node<K, V> findNode(Object key) {
        return table == null ? null : findNode(key, hash(key));
    }

    private Node<K, V> findNode(Object key, int h) {
        for (Node<K, V> n = table[h & (table.length - 1)]; n != null; n = n.next) {
            if (n.hash == h && Objects.equals(n.key, key)) {
                return n;
            }
        }
        return null;
    }

    private Node<K, V> removeNode(Object key) {
        if (table == null) {
            return null;
        }
        int h = hash(key);
        int i = h & (table.length - 1);
        Node<K, V> prev = null;
        for (Node<K, V> n = table[i]; n != null; prev = n, n = n.next) {
            if (n.hash == h && Objects.equals(n.key, key)) {
                if (prev == null) {
                    table[i] = n.next;
                } else {
                    prev.next = n.next;
                }
                modCount++;
                size--;
                return n;
            }
        }
        return null;
    }

    @SuppressWarnings({"rawtypes", "unchecked"})   // generic array creation, same justification as SimpleHashMap
    private void resize() {
        Node<K, V>[] oldTab = table;
        int oldCap = oldTab == null ? 0 : oldTab.length;
        int newCap;
        if (oldCap == 0) {
            newCap = threshold > 0 ? threshold : DEFAULT_CAPACITY;
        } else if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        } else {
            newCap = oldCap << 1;
        }
        threshold = newCap >= MAXIMUM_CAPACITY ? Integer.MAX_VALUE : (int) (newCap * loadFactor);
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        for (int j = 0; j < oldCap; j++) {
            Node<K, V> n = oldTab[j];
            Node<K, V> loHead = null, loTail = null, hiHead = null, hiTail = null;
            while (n != null) {
                Node<K, V> next = n.next;
                if ((n.hash & oldCap) == 0) {
                    if (loTail == null) loHead = n; else loTail.next = n;
                    loTail = n;
                } else {
                    if (hiTail == null) hiHead = n; else hiTail.next = n;
                    hiTail = n;
                }
                n = next;
            }
            if (loTail != null) { loTail.next = null; newTab[j] = loHead; }
            if (hiTail != null) { hiTail.next = null; newTab[j + oldCap] = hiHead; }
        }
        table = newTab;
    }

    // ---- the one abstract method AbstractMap needs --------------------------------------------

    @Override
    public Set<Map.Entry<K, V>> entrySet() {
        Set<Map.Entry<K, V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

    /** Live view: reads and removals go straight to the map. Adding through a view is unsupported. */
    private final class EntrySet extends AbstractSet<Map.Entry<K, V>> {
        @Override public int size()   { return size; }
        @Override public void clear() { BetterHashMap.this.clear(); }

        @Override
        public Iterator<Map.Entry<K, V>> iterator() {
            return new EntryIterator();
        }

        @Override
        public boolean contains(Object o) {         // O(1) instead of AbstractCollection's O(n) scan
            if (!(o instanceof Map.Entry<?, ?> e)) {
                return false;
            }
            Node<K, V> n = findNode(e.getKey());
            return n != null && n.equals(e);
        }

        @Override
        public boolean remove(Object o) {
            if (contains(o)) {
                removeNode(((Map.Entry<?, ?>) o).getKey());
                return true;
            }
            return false;
        }
    }

    private final class EntryIterator implements Iterator<Map.Entry<K, V>> {
        private Node<K, V> next;
        private Node<K, V> current;
        private int bucket;
        private int expectedModCount = modCount;

        EntryIterator() {
            next = firstFrom(0);
        }

        private Node<K, V> firstFrom(int start) {
            if (table == null) {
                return null;
            }
            for (bucket = start; bucket < table.length; bucket++) {
                if (table[bucket] != null) {
                    return table[bucket++];
                }
            }
            return null;
        }

        @Override
        public boolean hasNext() {
            return next != null;
        }

        @Override
        public Map.Entry<K, V> next() {
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            if (next == null) {
                throw new NoSuchElementException();
            }
            current = next;
            next = current.next != null ? current.next : firstFrom(bucket);
            return current;
        }

        @Override
        public void remove() {                      // the missing feature from SimpleHashMap
            if (current == null) {
                throw new IllegalStateException();
            }
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
            removeNode(current.key);                // `next` was computed already, so it's unaffected
            current = null;
            expectedModCount = modCount;
        }
    }
}
```

### Design notes

- **`EntryIterator.remove()` is safe mid-iteration:** `next` is computed *before* `current` is returned, so unlinking `current` doesn't affect where iteration continues. Then `expectedModCount` is re-synced.
- **`keySet()` and `values()` come from `AbstractMap`**, built on our `entrySet()`. Their `remove`/`removeIf` work because they delegate to our iterator's `remove()`.
- **`Node.hashCode() = hash(key) ^ hash(value)`** is mandated by the `Map.Entry` contract. That's why `mine.hashCode() == jdk.hashCode()` holds: `Map.hashCode` is the sum of the entry hashes.
- **The copy constructor presizes** with `ceil(size / 0.75)`, the #020 lesson that `new HashMap<>(n)` alone still resizes.

### Treeification: decision record

| | |
|---|---|
| **Decision** | Don't implement treeified bins now |
| **Why** | The code cost is large (the JDK's `TreeNode` is ~600 lines: red-black insert/delete, splitting on resize, `Comparable` tie-breaking), and it only helps when many keys collide. The learning goal of rows 24–25 is the table/chain/resize design and the `Map` contract |
| **Risk accepted** | O(n) worst case under collisions and hash flooding (measured: 737× at 50k). **Never use this class with untrusted keys** |
| **Cheaper mitigation if needed** | A randomised per-map hash seed (`h ^ seed`), which defeats *precomputed* collisions, though not keys with genuinely equal `hashCode()`s |
| **Revisit** | If the class were used for real, or as a stretch goal after the LRU project (rows 32–36) |

---

## Part 4 — Tests

### `test: BetterHashMapTest.java`

```java
package com.mastery.hashmap;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.ArrayList;
import java.util.ConcurrentModificationException;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Random;
import org.junit.jupiter.api.Test;

class BetterHashMapTest {

    @Test
    void isARealMapWithValueEquality() {
        Map<String, Integer> mine = new BetterHashMap<>();
        mine.put("a", 1);
        mine.put("b", 2);
        Map<String, Integer> jdk = new HashMap<>(Map.of("a", 1, "b", 2));
        assertEquals(jdk, mine);                       // AbstractMap.equals: any Map with the same mappings
        assertEquals(mine, jdk);                       // symmetric
        assertEquals(jdk.hashCode(), mine.hashCode()); // sum of entry hashCodes: needs a correct Node.hashCode
        assertEquals(jdk.entrySet(), mine.entrySet());
        assertTrue(mine.toString().equals("{a=1, b=2}") || mine.toString().equals("{b=2, a=1}"));
    }

    @Test
    void defaultMethodsWorkForFree() {
        Map<String, Integer> counts = new BetterHashMap<>();
        for (String w : List.of("x", "y", "x", "z", "x")) {
            counts.merge(w, 1, Integer::sum);
        }
        assertEquals(3, counts.get("x"));
        assertEquals(0, counts.getOrDefault("missing", 0));
        Map<String, List<Integer>> multi = new BetterHashMap<>();
        multi.computeIfAbsent("k", k -> new ArrayList<>()).add(1);
        multi.computeIfAbsent("k", k -> new ArrayList<>()).add(2);
        assertEquals(List.of(1, 2), multi.get("k"));
        counts.computeIfPresent("x", (k, v) -> null);  // returning null removes
        assertFalse(counts.containsKey("x"));
    }

    @Test
    void iteratorRemoveAndViewRemoval() {
        Map<Integer, Integer> m = new BetterHashMap<>();
        for (int i = 0; i < 100; i++) {
            m.put(i, i);
        }
        for (Iterator<Map.Entry<Integer, Integer>> it = m.entrySet().iterator(); it.hasNext(); ) {
            if (it.next().getKey() % 2 == 0) {
                it.remove();
            }
        }
        assertEquals(50, m.size());
        m.keySet().removeIf(k -> k < 50);              // keySet view built by AbstractMap on entrySet
        assertEquals(25, m.size());
        m.values().remove(99);                         // values view
        assertFalse(m.containsKey(99));
        Iterator<Map.Entry<Integer, Integer>> it = m.entrySet().iterator();
        assertThrows(IllegalStateException.class, it::remove);
    }

    @Test
    void setValueWritesThrough() {
        Map<String, Integer> m = new BetterHashMap<>();
        m.put("a", 1);
        m.entrySet().forEach(e -> e.setValue(e.getValue() * 100));
        assertEquals(100, m.get("a"));
    }

    @Test
    void failFastOnOutsideModification() {
        Map<Integer, Integer> m = new BetterHashMap<>();
        m.put(1, 1);
        m.put(2, 2);
        assertThrows(ConcurrentModificationException.class, () -> {
            for (Integer k : m.keySet()) {
                m.put(k + 100, 0);
            }
        });
    }

    @Test
    void copyConstructorPresizes() {
        Map<Integer, Integer> src = new HashMap<>();
        for (int i = 0; i < 100; i++) {
            src.put(i, i);
        }
        assertEquals(src, new BetterHashMap<>(src));
    }

    @Test
    void differentialAgainstHashMapIncludingViews() {
        Random rnd = new Random(21);
        Map<Integer, Integer> mine = new BetterHashMap<>();
        Map<Integer, Integer> ref = new HashMap<>();
        for (int step = 0; step < 100_000; step++) {
            Integer k = rnd.nextInt(20) == 0 ? null : rnd.nextInt(2_000);
            int v = rnd.nextInt(1000);
            switch (rnd.nextInt(7)) {
                case 0, 1 -> assertEquals(ref.put(k, v), mine.put(k, v));
                case 2 -> assertEquals(ref.remove(k), mine.remove(k));
                case 3 -> assertEquals(ref.merge(k, v, Integer::sum), mine.merge(k, v, Integer::sum));
                case 4 -> assertEquals(ref.entrySet().remove(Map.entry(0, v)), mine.entrySet().remove(Map.entry(0, v)));
                case 5 -> assertEquals(ref.getOrDefault(k, -1), mine.getOrDefault(k, -1));
                default -> {
                    if (step % 5_000 == 0) {                       // occasional bulk filter via iterator.remove
                        int mod = 2 + rnd.nextInt(5);
                        ref.values().removeIf(x -> x % mod == 0);
                        mine.values().removeIf(x -> x % mod == 0);
                    }
                }
            }
            assertEquals(ref.size(), mine.size());
        }
        assertEquals(ref, mine);
    }
}
```

```text
$ mvn test
Tests run: 55, Failures: 0, Errors: 0, Skipped: 0      (whole week-2 project; BetterHashMapTest: 7, SimpleHashMapTest: 8)
```

Notice what the tests **didn't** need: `merge`, `getOrDefault`, `computeIfAbsent`, `computeIfPresent`, `keySet().removeIf` and `values().remove` all work with **zero lines** of code in `BetterHashMap`. That's the payoff of implementing the standard interface via its skeletal class (EJ Item 20).

---

## Key Takeaways

1. **Review against a checklist, and record each verdict.** "Tests pass" doesn't answer API fitness, contracts or worst-case behaviour.
2. **Measure the weak spot before fixing it**, and sanity-check the measurement (the sampling bug).
3. **Implement the standard interface** (`extends AbstractMap`) and you get views, `equals`/`hashCode` and a dozen default methods for free. But **override the O(n) defaults**.
4. **`Map.Entry` has a contract** (`equals` against any entry, `key^value` hashing); map equality depends on it.
5. Some weaknesses are **accepted deliberately**. Write the decision down, with the risk and a revisit trigger.
6. `-Xlint:all` is a free reviewer: it found a missing suppression and a constructor escaping `this` (EJ Item 19).
