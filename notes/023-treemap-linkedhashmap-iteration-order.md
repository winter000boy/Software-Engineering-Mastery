# #023 — TreeMap / LinkedHashMap; Iteration Order

**Date:** 2026-09-24  
**Roadmap row:** 23 — P1 Java Core · Collections & Generics · *Learn*  
**Sources:** OpenJDK 21 source (`java/util/TreeMap.java`, `LinkedHashMap.java`), JDK 21 Javadoc  
**Builds on:** #017 (collection overview), #020 (HashMap internals)

> Code excerpts are copied from the JDK 21.0.10 source. Every "✔︎ observed" line was produced by running the experiment on JDK 21 (tree heights measured by reflecting on `TreeMap.root` with `--add-opens java.base/java.util=ALL-UNNAMED`).

---

## At a Glance (copy to handwritten notes)

- **Iteration order is a contract you choose:** `HashMap` = *unspecified* (it can change after a resize ✔︎) · `LinkedHashMap` = **insertion** order, or **access** order · `TreeMap` = **sorted by key** · `EnumMap` = enum declaration order.
- **`LinkedHashMap` = `HashMap` + a doubly linked list** threaded through all entries (`before`/`after`). Same O(1) operations; iteration is O(size), not O(capacity).
- Insertion order: **re-putting an existing key doesn't move it** ✔︎; remove + put moves it to the end ✔︎.
- **Access order** (`new LinkedHashMap<>(16, 0.75f, true)`): `get`, `put`, `getOrDefault`, `compute*` and `merge` move the entry to the **end**. `containsKey` and iterating over the map **don't** ✔︎. Add `removeEldestEntry` and you have an **LRU cache**.
- **`TreeMap` = red-black tree** (a self-balancing BST). `get`/`put`/`remove`/`floor`/`ceiling` are **O(log n)**. It never uses `hashCode` or `equals`; **only `compareTo`/`compare`** decides equality.
- ✔︎ Height of a 1M-entry TreeMap: **20** when built from a sorted map, **24** with random inserts, **37** with ascending inserts. It's always ≤ 2·log₂(n+1) ≈ 40.
- **`NavigableMap` API:** `floorKey` ≤, `ceilingKey` ≥, `lowerKey` <, `higherKey` >, `first`/`last`, `pollFirst`/`pollLast`, `headMap`/`tailMap`/`subMap` (**live views**), `descendingMap`.
- A comparator that's **inconsistent with equals** merges "equal-by-comparator" keys: a case-insensitive TreeMap keeps **1** entry for "Apple"/"apple" ✔︎.
- Choose by the **order you need to observe**. If nobody depends on order, use `HashMap`.

---

## 1. Iteration Order at a Glance ✔︎

Same 7 keys inserted in the order `banana, apple, cherry, date, elderberry, fig, grape`:

| Map | Iteration order observed | Guarantee |
|-----|--------------------------|-----------|
| `HashMap` | `banana, date, apple, cherry, fig, grape, elderberry` | **None.** It depends on hash codes and capacity |
| `LinkedHashMap` | `banana, apple, cherry, date, elderberry, fig, grape` | Insertion order |
| `TreeMap` | `apple, banana, cherry, date, elderberry, fig, grape` | Sorted by key |
| `ConcurrentSkipListMap` | `apple, banana, cherry, …` | Sorted (the concurrent `TreeMap`) |
| `ConcurrentHashMap` | `banana, date, cherry, fig, grape, elderberry, apple` | None (and different from `HashMap`) |
| `EnumMap` (`NEW`, `WAITING`, `TERMINATED` put as `WAITING, NEW, TERMINATED`) | `NEW, WAITING, TERMINATED` | Enum declaration order |
| `Map.of(...)` | varies | **Deliberately randomised per JVM run** (a `SALT`), so code can't come to rely on it (#017) |

### ✔︎ Why `HashMap` order changes after a resize

```
put 17, 1, 33 into new HashMap (capacity 16)
   17 & 15 = 1,  1 & 15 = 1,  33 & 15 = 1   → all in bucket 1, chain order 17 → 1 → 33
   iteration: [17, 1, 33]

put 20 more keys → capacity 32
   split of bucket 1 by (hash & 16):  17 → hi (bucket 17);  1, 33 → lo (bucket 1, order kept)
   iteration: bucket 1 [1, 33] then bucket 17 [17] → [1, 33, 17]
```

Same keys, different order, and nothing about them changed. **Never write code, tests or expected output that depends on `HashMap`/`HashSet` iteration order.** A common flaky-test source is `assertEquals("[a, b]", set.toString())`.

---

## 2. `LinkedHashMap` Internals

### 2.1 Structure, from the source

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;               // the doubly linked list through ALL entries
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);      // still lives in a HashMap bucket too
    }
}

transient LinkedHashMap.Entry<K,V> head;    // eldest
transient LinkedHashMap.Entry<K,V> tail;    // youngest
final boolean accessOrder;                   // false = insertion order (default), true = access order
```

```
HashMap table (for lookup)                    Linked list (for ordering)
 [0] ─ null                               head ─▶ banana ⇄ apple ⇄ cherry ⇄ date ⇄ … ⇄ grape ◀─ tail
 [1] ─▶ apple ─▶ fig                               (before/after pointers, independent of buckets)
 [2] ─▶ banana
 …
```

Each entry is in **two** structures: its bucket chain (`next`, for O(1) lookup) and the global list (`before`/`after`, for ordered iteration). Memory cost: 2 extra references per entry. Iteration walks the list, so it's **O(size)**, whereas `HashMap` iteration is O(capacity + size) because it scans empty buckets (#020 §3.9).

### 2.2 How `LinkedHashMap` plugs into `HashMap`: the template-method hooks

`HashMap.putVal` calls `afterNodeAccess(e)` when a key already existed and `afterNodeInsertion(evict)` after inserting (#020 §3.5). In `HashMap` they do nothing; `LinkedHashMap` overrides them.

```java
// Moves the entry to the END of the list when accessOrder is on (simplified: the full JDK 21 method
// also handles the Java 21 putFirst/putLast "put modes").
void afterNodeAccess(Node<K,V> e) {
    ...
    if ((putMode == PUT_LAST || (putMode == PUT_NORM && accessOrder)) && (last = tail) != e) {
        // unlink p from its current position, relink it after `last`, make it the tail
        ...
        tail = p;
        ++modCount;                 // an access-ordered get() IS a structural modification!
    }
    ...
}

void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;                   // default: never evict. Override this for a bounded cache
}
```

Note the `++modCount` in `afterNodeAccess`. In an access-ordered map, calling `get()` while iterating over it throws `ConcurrentModificationException`. The Javadoc says so explicitly: *"In access-ordered linked hash maps, merely querying the map with `get` is a structural modification."*

### 2.3 Insertion order vs access order ✔︎

```java
Map<String, Integer> linked = new LinkedHashMap<>();   // insertion order
// [banana, apple, cherry, date, elderberry, fig, grape]
linked.put("banana", 99);               // key exists → value replaced, POSITION KEPT
// [banana, apple, cherry, …]            ✔︎
linked.remove("banana"); linked.put("banana", 6);
// [apple, cherry, date, elderberry, fig, grape, banana]   ✔︎ now a new insertion

LinkedHashMap<String, Integer> access = new LinkedHashMap<>(16, 0.75f, true);  // access order
access.put("a", 0); access.put("b", 0); access.put("c", 0); access.put("d", 0);
access.get("a");                // → a moves to the end
access.put("b", 1);             // → b moves to the end
access.getOrDefault("c", 0);    // → c moves to the end
access.containsKey("d");        // NOT an access: d stays where it is
// ✔︎ [d, a, b, c]   (d is now the "least recently used")
```

| Operation | Counts as an access (moves the entry to the end)? |
|-----------|:---:|
| `get`, `getOrDefault` | ✅ |
| `put` / `putIfAbsent` / `replace` on an existing key | ✅ ✔︎ |
| `compute`, `computeIfAbsent`, `computeIfPresent`, `merge` | ✅ ✔︎ |
| `containsKey`, `containsValue` | ❌ ✔︎ |
| Iterating over `keySet`/`values`/`entrySet` | ❌ ✔︎ |
| `entry.setValue` via an iterator | ❌ ✔︎ |

### 2.4 LRU cache in 10 lines

```java
public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LruCache(int capacity) {
        super(16, 0.75f, true);                     // true = access order
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;                   // called after every insert; true → evict the head
    }
}
```

The head of an access-ordered list is always the **least recently used** entry, so evicting the head *is* LRU. Everything is O(1). This is the "LinkedHashMap version" of the LRU project (rows 32–36). The "manual version" builds the same thing from a `HashMap<K, Node>` plus your own doubly linked list.

Limits: it's not thread-safe (wrap it in `Collections.synchronizedMap`, or use Caffeine), and it's size-bounded only, with no TTL.

### 2.5 Java 21: `putFirst` / `putLast` ✔︎

`LinkedHashMap` is a `SequencedMap`:

```java
LinkedHashMap<String, Integer> seq = new LinkedHashMap<>();
seq.put("x", 1); seq.put("y", 2); seq.put("z", 3);
seq.putFirst("z", 30);   // ✔︎ {z=30, x=1, y=2}: moves (or inserts) to the front
seq.putLast("x", 10);    // ✔︎ {z=30, y=2, x=10}: moves to the end, returns the old value 1
seq.firstEntry(); seq.pollLastEntry(); seq.reversed();   // other SequencedMap methods
```

That's what the `putMode` field in `afterNodeAccess` is for.

---

## 3. `TreeMap` Internals

### 3.1 Structure, from the source

```java
private final Comparator<? super K> comparator;   // null → use the keys' natural ordering
private transient Entry<K,V> root;

static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
    ...
}
private static final boolean RED   = false;
private static final boolean BLACK = true;
```

`Comparator<? super K>`: a comparator of any **supertype** of K works. That's PECS, since comparators are consumers (#019, #026).

### 3.2 Lookup is plain binary-search-tree descent, from the source

```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    Objects.requireNonNull(key);                     // ← why natural-order TreeMaps reject null keys
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;   // ← ClassCastException if not Comparable
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;                                // cmp == 0 means "the same key". equals() is never called
    }
    return null;
}
```

This shows two classic behaviours:

- `new TreeMap<>().put(null, v)` → **`NullPointerException`** (natural order). A comparator like `Comparator.nullsFirst(...)` can allow `null`.
- Putting a key that isn't `Comparable`, with no comparator, → **`ClassCastException`**. Since Java 7 even the *first* `put` fails, because `put` compares the key with itself.

### 3.3 Red-black tree in one screen

A red-black tree is a BST in which every node is red or black, with these rules:

1. The root is black.
2. A red node has no red child.
3. Every path from a node down to its empty leaves has the **same number of black nodes**.

Together these guarantee that the **longest path ≤ 2 × the shortest**, so height ≤ **2·log₂(n + 1)**, and every operation is O(log n) even in the worst case.

- **Insert:** a normal BST insert of a **red** node, then `fixAfterInsertion` restores the rules with **recolourings and at most 2 rotations**.
- **Delete:** a BST delete, then `fixAfterDeletion` (**at most 3 rotations**).

```
Rotation (left) at x — restructures locally, keeps BST order:
        x                    y
       / \                  / \
      a   y      ──▶       x   c
         / \              / \
        b   c            a   b
```

You don't need to memorise the fix-up cases. You do need to know **why the tree is balanced** (the rules), **what that buys** (worst-case O(log n)), and that **rotations are O(1)**.

### 3.4 ✔︎ How tall is a TreeMap, really?

1 000 000 `Integer` keys (log₂(10⁶) ≈ 19.9; red-black bound 2·log₂(n+1) ≈ 39.9):

| How it was built | Height observed | Comparisons per lookup (worst) |
|------------------|----------------:|-------------------------------:|
| `new TreeMap<>(sortedMap)` (bulk build, `buildFromSorted`) | **20** | ~20 (perfectly balanced) |
| Random insertion order | **24** | ~24 |
| **Ascending** insertion order (0, 1, 2, …) | **37** | ~37 (close to the bound) |

An unbalanced BST fed ascending keys would degrade into a **linked list of height 1 000 000**. The red-black rules cap it at 37. Also: if you already have sorted data, **construct the TreeMap from a `SortedMap`**. It's O(n), and the tree comes out perfectly balanced.

### 3.5 Comparison replaces equality ✔︎

```java
TreeMap<String, Integer> ci = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
ci.put("Apple", 1);
ci.put("apple", 2);          // compare("apple", "Apple") == 0 → treated as the SAME key
// ✔︎ {Apple=2}, size 1: value replaced, the ORIGINAL key object "Apple" kept
```

Sometimes that's exactly what you want (case-insensitive headers, `HttpHeaders`-style maps). With an accidental comparator it **silently loses data**: in #027, a `TreeSet` of employees ordered only by salary dropped one of two people who earn the same ✔︎. **Fix: add a tie-breaker on a unique key** (`.thenComparingInt(Employee::id)`).

---

## 4. The `NavigableMap` API ✔︎

Using the sorted fruit map `{apple, banana, cherry, date, elderberry, fig, grape}`:

| Method | Returns | Example → result |
|--------|---------|------------------|
| `firstKey()` / `lastKey()` | Smallest / largest key (throws if empty) | `apple` / `grape` |
| `firstEntry()` / `lastEntry()` | Same as entries (null if empty) | |
| `floorKey(k)` | Greatest key **≤ k** | `floorKey("coconut")` → `cherry` |
| `ceilingKey(k)` | Least key **≥ k** | `ceilingKey("coconut")` → `date` |
| `lowerKey(k)` | Greatest key **< k** | `lowerKey("cherry")` → `banana` ✔︎ |
| `higherKey(k)` | Least key **> k** | `higherKey("cherry")` → `date` ✔︎ |
| `pollFirstEntry()` / `pollLastEntry()` | Remove and return the smallest/largest | `apple=5` removed ✔︎ |
| `headMap(k)` / `headMap(k, incl)` | Keys < k (≤ k) | |
| `tailMap(k)` / `tailMap(k, incl)` | Keys ≥ k (> k) | |
| `subMap(from, fromIncl, to, toIncl)` | A range | `subMap("b", true, "e", false)` → `banana, cherry, date` ✔︎ |
| `descendingMap()` / `descendingKeySet()` | Reverse-order view | `grape, fig, …, apple` ✔︎ |
| `navigableKeySet()` | Keys as a `NavigableSet` | |

### Range views are live ✔︎

```java
NavigableMap<String, Integer> sub = tm.subMap("b", true, "e", false);   // [banana, cherry, date]
tm.put("coconut", 7);                // add to the BACKING map…
sub.keySet();                        // ✔︎ [banana, cherry, coconut, date]: the view sees it
sub.put("zebra", 1);                 // ✔︎ IllegalArgumentException: key out of range
```

The views cost O(1) to create and don't copy anything. They're ideal for "all events between t1 and t2" queries: `events.subMap(t1, true, t2, false).values()`.

### Real-world `TreeMap` patterns

| Problem | Pattern |
|---------|---------|
| "Which price was in effect at time t?" | `prices.floorEntry(t)` |
| Booking/overlap checks | `floorEntry(start)` + `ceilingEntry(start)` (#022 bonus) |
| Consistent-hash ring (row 329) | `ring.ceilingEntry(hash(key))`, wrapping around with `firstEntry()` |
| Leaderboard: rank neighbours | `scores.lowerKey(s)` / `higherKey(s)` |
| Sliding time window | `window.headMap(now - span).clear()`, which removes a whole range through the view |
| Order book (price levels) | `TreeMap<BigDecimal, Queue<Order>>`; best bid = `lastEntry`, best ask = `firstEntry` |

---

## 5. Choosing an Ordered Map

| Need | Choose | Cost |
|------|--------|------|
| Just lookups; order irrelevant | `HashMap` | O(1) |
| Stable, predictable output order (JSON fields, config, reports, tests) | `LinkedHashMap` (insertion) | O(1), +2 refs/entry |
| LRU / "recently used" | `LinkedHashMap(…, true)` + `removeEldestEntry` | O(1) |
| Sorted keys, ranges, nearest-key queries | `TreeMap` | O(log n) |
| Enum keys | `EnumMap` | O(1), a compact array |
| Sorted **and** concurrent | `ConcurrentSkipListMap` | O(log n) expected, lock-free |
| Concurrent, unordered | `ConcurrentHashMap` | O(1) |

Libraries rely on these choices. For example, Jackson deserialises an untyped JSON object into a `LinkedHashMap` (Gson uses its own insertion-ordered `LinkedTreeMap`), so field order survives a read → write round-trip.

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| HashMap vs LinkedHashMap vs TreeMap order? | Unspecified (can change on resize) / insertion (or access) / sorted by key |
| How does LinkedHashMap keep order? | A doubly linked list (`before`/`after`) through the entries, on top of HashMap's buckets; maintained by the `afterNode*` hooks |
| Does re-inserting a key change LinkedHashMap order? | No (insertion mode). Remove + put does |
| Implement an LRU cache quickly? | `LinkedHashMap(cap, 0.75f, true)` + `removeEldestEntry(size() > max)` |
| Why can `get()` throw CME on a LinkedHashMap? | In access order, `get` moves the entry, which is a structural modification |
| What data structure is TreeMap? | A red-black tree: height ≤ 2·log₂(n+1), O(log n) worst case |
| Does TreeMap use `equals`/`hashCode`? | No. Only `compareTo`/`compare`; 0 means the same key |
| Null keys in TreeMap? | NPE with natural ordering; possible with a null-tolerant comparator |
| floor vs lower, ceiling vs higher? | floor ≤, lower <; ceiling ≥, higher > |
| Are `subMap`/`headMap` copies? | No, live views, range-checked on writes |
| Build a TreeMap from sorted data efficiently? | `new TreeMap<>(sortedMap)`: O(n), perfectly balanced |

---

## Handwritten Cheat Sheet

```
ORDER:  HashMap = none (changes on resize!) | LinkedHashMap = insertion / access
        TreeMap = sorted | EnumMap = declaration | Map.of = random per run
LINKEDHASHMAP = HashMap + doubly linked list (before/after), head = eldest, tail = newest
   hooks: afterNodeAccess (move to tail if accessOrder), afterNodeInsertion → removeEldestEntry
   re-put keeps position; remove+put → end
   access order: get/put/getOrDefault/compute/merge move; containsKey/iteration don't
   LRU = new LinkedHashMap<>(16, .75f, true) + removeEldestEntry(size() > cap)
   access-order get() = structural → CME while iterating
   Java 21: putFirst / putLast / firstEntry / pollLastEntry / reversed
TREEMAP = red-black tree {key,value,left,right,parent,color}
   rules: root black · no red-red · equal black height → height ≤ 2·log2(n+1)
   1M keys: height 20 (bulk) / 24 (random) / 37 (ascending)
   getEntry: compareTo descent; cmp==0 ⇒ same key (equals never used)
   null key → NPE (natural); non-Comparable → ClassCastException
   floor ≤ · lower < · ceiling ≥ · higher > · pollFirst/Last
   headMap/tailMap/subMap = LIVE views, out-of-range put → IAE
   comparator inconsistent w/ equals → keys merge (case-insensitive: 1 entry)
```

---

**Next:** row 24 — Implement a simplified HashMap (#024) · row 26 — Generics, bounded types, wildcards (#026)
