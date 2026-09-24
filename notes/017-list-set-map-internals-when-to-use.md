# #017 — List / Set / Map Internals & When to Use Each

**Date:** 2026-09-24  
**Roadmap row:** 17 — P1 Java Core · Collections & Generics · *Learn*  
**Sources:** `java.util` Javadoc (JDK 21), OpenJDK source, *Effective Java* Items 54, 58, 64  
**Deep dives:** `ArrayList`/`LinkedList`/`HashMap` internals → #020 · benchmark → #018 · reading notes → #019

> Every behaviour marked ✔︎ was run on JDK 21.0.10 while writing this note.

---

## At a Glance (copy to handwritten notes)

- **`Collection`** → `List` (ordered, indexed, duplicates OK), `Set` (no duplicates), `Queue`/`Deque` (ends-oriented). **`Map`** is separate (key → value, keys unique) and is *not* a `Collection`.
- **Default picks:** `ArrayList` for lists · `HashMap` for maps · `HashSet` for sets · `ArrayDeque` for stacks **and** queues · `PriorityQueue` for "smallest/largest next".
- **Need order?** Insertion order → `LinkedHashMap`/`LinkedHashSet`. Sorted order → `TreeMap`/`TreeSet` (O(log n), `floor`/`ceiling`/range views).
- **Hash-based** = O(1) average, relying on **`equals` + `hashCode`**. **Tree-based** = O(log n), relying on **`compareTo`/`Comparator`**.
- **`LinkedList` is almost never the right choice.** `ArrayDeque` beats it as a queue/stack and `ArrayList` beats it as a list (cache locality, #018).
- **Immutable:** `List.of`/`Set.of`/`Map.of`/`copyOf` reject `null`s ✔︎ and throw on modification ✔︎. `Collections.unmodifiableX` is only a **view** of a collection that can still change ✔︎.
- **Iteration order** of `HashMap`/`HashSet` is unspecified and can change after a resize. `Set.of`/`Map.of` order **differs between JVM runs** ✔︎.
- **Fail-fast iterators:** modifying a collection while iterating (except through the iterator) throws `ConcurrentModificationException` ✔︎. Use `iterator.remove()` or `removeIf`.
- **Never mutate an object while it's a key** in a hash-based collection: it becomes unfindable ✔︎.
- **Java 21 sequenced collections:** `addFirst`/`getFirst`/`getLast`/`reversed()` on `List`, `Deque`, `LinkedHashSet`/`LinkedHashMap`, `TreeSet`/`TreeMap`.

---

## 1. The Hierarchy

```
                              Iterable<E>
                                  │
                             Collection<E>
          ┌───────────────────────┼─────────────────────────┐
          │                       │                         │
       List<E>                  Set<E>                   Queue<E>
          │                       │                         │
          │                SequencedSet<E>¹             Deque<E>
          │                       │                         │
          │                  SortedSet<E>                   │
          │                       │                         │
          │                 NavigableSet<E>                 │
          │                                                 │
  ArrayList   LinkedList──────────────────────────────────▶ (also a Deque)
  Vector→Stack (legacy)     HashSet   LinkedHashSet   TreeSet   EnumSet
  CopyOnWriteArrayList      ArrayDeque   PriorityQueue   ConcurrentLinkedQueue   BlockingQueue impls

                               Map<K,V>            (NOT a Collection)
                                  │
                           SequencedMap<K,V>¹
                                  │
                            SortedMap<K,V>
                                  │
                          NavigableMap<K,V>
  HashMap   LinkedHashMap   TreeMap   EnumMap   IdentityHashMap   WeakHashMap
  Hashtable (legacy)   ConcurrentHashMap   ConcurrentSkipListMap

¹ Java 21 (JEP 431). List and Deque are also SequencedCollections.
```

**Program to the interface** (EJ Item 64): declare `List<Book> books = new ArrayList<>();`, not `ArrayList<Book> books`. You can then swap the implementation without touching callers.

---

## 2. Lists

A `List` is an **ordered** sequence with **positional access**; duplicates and (usually) `null`s are allowed.

| | `ArrayList` | `LinkedList` | `CopyOnWriteArrayList` |
|---|---|---|---|
| Internal structure | Resizable `Object[]` | **Doubly linked** nodes (`prev`, `item`, `next`) | Array replaced by a copy on **every** write |
| `get(i)` / `set(i)` | **O(1)** | O(n) (walks from the nearer end) | O(1) |
| `add(e)` at end | **Amortized O(1)** (grows ×1.5) | O(1) | O(n) (copies the array) |
| `add(i, e)` / `remove(i)` in the middle | O(n) (shifts elements via `System.arraycopy`) | O(n) to *find* the position + O(1) to link | O(n) |
| `remove` via `ListIterator` at the current position | O(n) (shift) | **O(1)** | Not supported |
| `contains` / `indexOf` | O(n) | O(n) | O(n) |
| Memory per element | ~4–8 bytes (a reference in the array) | ~24–40 bytes (a node object + 3 refs + header) | Like `ArrayList` |
| CPU cache friendliness | **Excellent** (contiguous) | Poor (pointer chasing) | Excellent |
| Thread-safe | No | No | **Yes** (reads never lock) |
| Use when | **Almost always** | Rarely: many inserts/removes *through an iterator* in huge lists | Read-mostly, rarely-modified lists (listener lists) |

Legacy: `Vector` (every method `synchronized`) and `Stack extends Vector` (a design mistake, EJ Item 18). Don't use them; use `ArrayList` / `ArrayDeque`.

### `List` gotchas

```java
List<Integer> nums = new ArrayList<>(List.of(10, 20, 30));
nums.remove(1);                    // removes the element at INDEX 1 → [10, 30]
nums.remove(Integer.valueOf(10));  // removes the VALUE 10 → [30]

Arrays.asList(1, 2).add(3);        // ✔︎ UnsupportedOperationException: fixed-size view of the array
Arrays.asList(1, 2).set(0, 9);     // ✔︎ OK: set is allowed, and writes through to the backing array
```

---

## 3. Sets

A `Set` has **no duplicate elements**. "Duplicate" is defined by `equals` (hash sets) or by `compareTo == 0` (tree sets).

| | `HashSet` | `LinkedHashSet` | `TreeSet` | `EnumSet` |
|---|---|---|---|---|
| Backed by | A `HashMap` (elements are keys; the value is a shared dummy `PRESENT` object) | A `LinkedHashMap` | A `TreeMap` (**red-black tree**) | A **bit vector**: one `long` for ≤ 64 constants (`RegularEnumSet` ✔︎), else a `long[]` (`JumboEnumSet`) |
| `add` / `remove` / `contains` | **O(1)** average | O(1) average | **O(log n)** | **O(1)**, a bitwise op |
| Iteration order | **Unspecified** | **Insertion order** | **Sorted** (natural or `Comparator`) | Enum declaration order |
| Needs | `equals` + `hashCode` | `equals` + `hashCode` | `Comparable` or a `Comparator` | Enum type |
| `null` | One `null` allowed | One `null` allowed | ❌ with natural ordering | ❌ |
| Extra API | — | Sequenced (`getFirst`/`getLast`, 21+) | `first`, `last`, `floor`, `ceiling`, `lower`, `higher`, `headSet`, `tailSet`, `subSet`, `descendingSet`, `pollFirst` | `range`, `complementOf`, `allOf` |
| Use when | Default: fast membership | Dedupe while **keeping input order** | Need a **sorted** view or "nearest element" queries | A set of enum flags (replaces bit fields) |

```java
// Dedupe while keeping first-seen order
List<String> unique = new ArrayList<>(new LinkedHashSet<>(List.of("b", "a", "b", "c", "a")));  // [b, a, c]

// Closest-match queries (a leaderboard, time buckets)
TreeSet<Integer> scores = new TreeSet<>(List.of(10, 20, 30));
scores.floor(25);    // 20: greatest ≤ 25
scores.ceiling(25);  // 30: least ≥ 25
```

---

## 4. Queues & Deques

| | `ArrayDeque` | `PriorityQueue` | `LinkedList` (as a Deque) |
|---|---|---|---|
| Structure | **Circular resizable array** (head/tail indices) | **Binary min-heap** in an array | Doubly linked nodes |
| Ends: `offerFirst`/`offerLast`/`pollFirst`/`pollLast`/`peek` | **O(1)** amortized | — | O(1) |
| `offer` / `poll` | O(1) | **O(log n)** | O(1) |
| `peek` | O(1) | **O(1)** (the smallest element) | O(1) |
| `remove(Object)` / `contains` | O(n) | O(n) | O(n) |
| Order | FIFO or LIFO, depending on the methods you use | **Priority** (natural order or `Comparator`) | FIFO/LIFO |
| `null` | ❌ ✔︎ (`null` is used as the "empty" signal by `poll`) | ❌ | ✅ |
| Use as | **Stack** (`push`/`pop`/`peek`) **and queue** (`offer`/`poll`) | Scheduling, top-k, Dijkstra, merging sorted streams | Avoid |

```java
Deque<Integer> stack = new ArrayDeque<>();   // replaces java.util.Stack
stack.push(1); stack.push(2); stack.pop();   // 2

Queue<Task> queue = new ArrayDeque<>();      // FIFO
queue.offer(t1); queue.poll();

PriorityQueue<Integer> pq = new PriorityQueue<>();
for (int x : new int[]{5, 4, 3, 2, 1}) pq.offer(x);
System.out.println(pq);                     // [1, 2, 4, 5, 3]: HEAP order, not sorted!
while (!pq.isEmpty()) System.out.print(pq.poll() + " ");   // 1 2 3 4 5: poll() is sorted
```

⚠️ Iterating over or printing a `PriorityQueue` does **not** give sorted order. Only repeated `poll()` does.

**Method pairs:** `add`/`remove`/`element` **throw** on failure; `offer`/`poll`/`peek` **return** `false`/`null`. Use the second set for bounded queues.

Concurrent queues (for P1 Concurrency, rows 42–91): `ConcurrentLinkedQueue` (lock-free), `ArrayBlockingQueue`/`LinkedBlockingQueue` (producer-consumer), `PriorityBlockingQueue`, `DelayQueue`.

---

## 5. Maps

A `Map` associates **unique keys** with values.

| | `HashMap` | `LinkedHashMap` | `TreeMap` | `EnumMap` | `ConcurrentHashMap` |
|---|---|---|---|---|---|
| Structure | **Array of buckets**; each bucket is a linked list, turned into a **red-black tree** when it gets long (#020) | `HashMap` + a **doubly linked list** through all entries | **Red-black tree** | Array indexed by `ordinal()` | Bucket array with per-bucket CAS/locking |
| `get` / `put` / `remove` / `containsKey` | **O(1)** average; O(log n) worst per bucket (Java 8+) | O(1) | **O(log n)** | O(1) | O(1) |
| `containsValue` | O(n) | O(n) | O(n) | O(n) | O(n) |
| Iteration order | Unspecified | **Insertion** order, or **access** order ✔︎ | **Sorted by key** | Enum order | Unspecified |
| `null` key / value | ✅ one null key ✔︎ / ✅ | ✅ / ✅ | ❌ key (natural order) ✔︎ / ✅ | ❌ / ✅ | ❌ / ❌ ✔︎ |
| Key requirement | `equals` + `hashCode` | `equals` + `hashCode` | `Comparable` / `Comparator` | Enum | `equals` + `hashCode` |
| Use when | **Default** | Predictable order; **LRU caches** | Sorted keys, range queries (`floorKey`, `headMap`, `tailMap`, `subMap`) | Enum keys | Shared between threads |

Specialised maps:

- **`IdentityHashMap`**: compares keys with `==`, not `equals`. For graph algorithms and serialisation (tracking visited objects).
- **`WeakHashMap`**: entries disappear once the key is no longer strongly referenced. For metadata caches keyed by objects (EJ Item 7).
- **`Hashtable`**: legacy, fully synchronised, no nulls. Use `ConcurrentHashMap`.

### `LinkedHashMap` as an LRU cache (preview of rows 32–36)

```java
LinkedHashMap<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true);  // accessOrder = true
lru.put("a", 1); lru.put("b", 2); lru.put("c", 3);
lru.get("a");
lru.keySet();   // ✔︎ [b, c, a]: least recently used first

class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LruCache(int capacity) { super(16, 0.75f, true); this.capacity = capacity; }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;   // evict the LRU entry after each put
    }
}
```

### `TreeMap` navigation ✔︎

```java
TreeMap<Integer, String> tm = new TreeMap<>(Map.of(10, "ten", 20, "twenty", 30, "thirty"));
tm.floorKey(25);   // 20
tm.ceilingKey(25); // 30
tm.headMap(20);    // {10=ten}                    exclusive
tm.tailMap(20);    // {20=twenty, 30=thirty}      inclusive
```

Typical uses: time-series lookups ("price in effect at time t" = `floorEntry(t)`), IP-range tables, consistent-hash rings (row 329, where `ceilingEntry(hash)` finds the next node).

### Modern `Map` API (Java 8+). Use these instead of get-then-put.

```java
Map<String, Integer> counts = new HashMap<>();
counts.merge(word, 1, Integer::sum);                          // word count, one line
counts.getOrDefault("x", 0);
counts.putIfAbsent("y", 0);
counts.computeIfAbsent(key, k -> new ArrayList<>()).add(v);   // multimap idiom
counts.computeIfPresent(key, (k, v) -> v > 1 ? v - 1 : null); // returning null REMOVES the entry
for (Map.Entry<String, Integer> e : counts.entrySet()) { ... } // iterate entries, not keySet() + get()
```

---

## 6. Hash-Based vs Tree-Based: the Core Trade-off

| | Hash (`HashMap`, `HashSet`) | Tree (`TreeMap`, `TreeSet`) |
|---|---|---|
| Speed | O(1) average | O(log n) (≈ 20 comparisons for 1M entries) |
| Ordering | None | Sorted, plus navigation and range queries |
| Contract needed | `equals` + consistent `hashCode` (EJ Items 10–11) | `compareTo` **consistent with `equals`** (EJ Item 14) |
| Worst case | Bad hash → many collisions (mitigated by treeify, #020) | Always O(log n) (the tree is balanced) |
| Memory | Table + nodes | Nodes with left/right/parent + colour |

**Consistency with `equals` matters.** ✔︎ `new BigDecimal("1.0")` and `new BigDecimal("1.00")` are **2** elements in a `HashSet` (not `equals`) but **1** in a `TreeSet` (`compareTo == 0`). The same data gives a different set size.

---

## 7. Immutable & Unmodifiable Collections

| Construct | Mutable through it? | Backing can change? | `null`s | Notes |
|-----------|:---:|:---:|:---:|-------|
| `List.of(...)`, `Set.of`, `Map.of`, `Map.ofEntries` (9+) | ❌ UOE ✔︎ | — (truly immutable) | ❌ NPE, even `contains(null)` ✔︎ | `Set.of(1, 1)` → `IllegalArgumentException` ✔︎. Iteration order of `Set.of`/`Map.of` is **randomised per JVM run** ✔︎ |
| `List.copyOf(c)` (10+) | ❌ | No, it's a snapshot ✔︎ | ❌ | Returns `c` itself if it's already an immutable list |
| `Collections.unmodifiableList(c)` | ❌ | **Yes**: a read-only **view** ✔︎ | ✅ | Good for exposing internal state read-only; changes to the source show through |
| `Arrays.asList(arr)` | `set` ✅, `add`/`remove` ❌ ✔︎ | Writes through to the array | ✅ | Fixed-size |
| `stream().toList()` (16+) | ❌ | No | ✅ | Unmodifiable |

```java
List<Integer> src  = new ArrayList<>(List.of(1, 2));
List<Integer> view = Collections.unmodifiableList(src);
List<Integer> copy = List.copyOf(src);
src.add(3);
// ✔︎ view sees [1, 2, 3], copy sees [1, 2]
```

Rule of thumb (EJ Items 17 & 50): accept any `Collection` in constructors, store a `List.copyOf`, and return it directly (it's already immutable).

---

## 8. Iteration, Fail-Fast & Safe Removal

```java
List<Integer> l = new ArrayList<>(List.of(1, 2, 3));
for (Integer x : l) if (x == 1) l.remove(x);        // ✔︎ ConcurrentModificationException

l.removeIf(x -> x == 1);                             // ✔︎ OK (Java 8+), the clearest option
for (Iterator<Integer> it = l.iterator(); it.hasNext(); )
    if (it.next() == 2) it.remove();                 // OK, the classic way
```

- **How fail-fast works:** each collection keeps a `modCount` that every structural change increments. The iterator stores `expectedModCount` and checks it on each `next()`. It's **best-effort** detection of bugs, not a thread-safety mechanism.
- **Weakly consistent** iterators (`ConcurrentHashMap`, `CopyOnWriteArrayList`) never throw CME. They may or may not reflect concurrent changes.
- Prefer **for-each** over index loops (EJ Item 58), except when you need the index, or when removing (use `removeIf`).

---

## 9. The Mutable-Key Trap ✔︎

```java
List<Integer> key = new ArrayList<>(List.of(1));
Set<List<Integer>> set = new HashSet<>();
set.add(key);
key.add(2);                     // key's hashCode changes while it's inside the set
set.contains(key);              // false! It's filed in the bucket for its OLD hash
set.size();                     // 1: an unreachable "ghost" entry
```

Keys (and set elements) must be **immutable**, or at least never change the fields used by `equals`/`hashCode`/`compareTo` while they're stored. This is why `String`, `Integer`, records and the `Isbn`/`ItemId` value objects from #005 make ideal keys.

---

## 10. Java 21 Sequenced Collections (JEP 431) ✔︎

A uniform API for "collections with a defined encounter order":

| Interface | Methods | Implemented by |
|-----------|---------|----------------|
| `SequencedCollection<E>` | `addFirst`, `addLast`, `getFirst`, `getLast`, `removeFirst`, `removeLast`, `reversed()` | `List`, `Deque`, `LinkedHashSet`, `SortedSet` |
| `SequencedSet<E>` | `reversed()` returns a `SequencedSet` | `LinkedHashSet`, `TreeSet` |
| `SequencedMap<K,V>` | `firstEntry`, `lastEntry`, `pollFirstEntry`, `putFirst`, `putLast`, `reversed()`, `sequencedKeySet()`… | `LinkedHashMap`, `TreeMap` |

```java
List<Integer> seq = new ArrayList<>(List.of(1, 2, 3));
seq.addFirst(0);        // [0, 1, 2, 3]
seq.getLast();          // 3    (was: seq.get(seq.size() - 1))
seq.reversed();         // [3, 2, 1, 0]: a reversed VIEW, not a copy
```

(`addFirst` on an `ArrayList` is still O(n): it shifts everything. The API is uniform; the costs aren't.)

---

## 11. Decision Guide

```
Need key → value lookup?
 ├─ yes → Map
 │    ├─ shared across threads?        → ConcurrentHashMap
 │    ├─ keys are an enum?             → EnumMap
 │    ├─ need sorted keys / ranges?    → TreeMap
 │    ├─ need insertion/access order?  → LinkedHashMap (LRU: accessOrder=true)
 │    └─ otherwise                     → HashMap
 └─ no → a collection of elements
      ├─ unique elements / membership tests?  → Set
      │    ├─ enum values?          → EnumSet
      │    ├─ sorted / nearest?     → TreeSet
      │    ├─ keep insertion order? → LinkedHashSet
      │    └─ otherwise             → HashSet
      ├─ process by priority?       → PriorityQueue
      ├─ add/remove at the ends only (stack, queue, sliding window)? → ArrayDeque
      └─ ordered, indexed, duplicates OK → ArrayList
           (read-mostly + multi-threaded → CopyOnWriteArrayList)
           (fixed data → List.of / List.copyOf)
```

### Scenario drill

| Scenario | Pick | Why |
|----------|------|-----|
| Shopping cart line items | `ArrayList` | Ordered, indexed, duplicates possible |
| "Has this user id been seen?" | `HashSet<Long>` | O(1) membership |
| Word frequency count | `HashMap<String,Integer>` + `merge` | O(1) update |
| Word frequency, printed alphabetically | `TreeMap<String,Integer>` | Sorted keys |
| Recently viewed items, 10 max | `LinkedHashMap` (access order + `removeEldestEntry`) | O(1) LRU |
| Undo stack | `ArrayDeque` (`push`/`pop`) | LIFO, O(1) |
| Print-job queue | `ArrayDeque` (`offer`/`poll`) | FIFO |
| Hospital triage | `PriorityQueue<Patient>` with a severity `Comparator` | Highest priority first |
| "Price in effect at time t" | `TreeMap<Instant, Price>.floorEntry(t)` | Floor lookup, O(log n) |
| Days a store is open | `EnumSet<DayOfWeek>` | Compact, O(1) |
| Config loaded once at startup | `Map.copyOf(...)` | Immutable, safe to share |
| Event listeners on a UI component | `CopyOnWriteArrayList` | Iterated often, changed rarely, multi-threaded |

---

## 12. Complexity Cheat Table

| Operation | ArrayList | LinkedList | ArrayDeque | PriorityQueue | HashMap/Set | LinkedHashMap/Set | TreeMap/Set |
|-----------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Access by index | O(1) | O(n) | — | — | — | — | — |
| Search / contains | O(n) | O(n) | O(n) | O(n) | **O(1)** | O(1) | O(log n) |
| Insert at end / add | O(1)* | O(1) | O(1)* | O(log n) | O(1) | O(1) | O(log n) |
| Insert at front | O(n) | O(1) | O(1)* | — | — | — | — |
| Remove from middle (by index / value) | O(n) | O(n) | O(n) | O(n) | O(1) (by key) | O(1) | O(log n) |
| Min / max | O(n) | O(n) | — | **O(1) peek** | O(n) | O(n) | O(log n) (`first`/`last`) |
| Ordered iteration | index order | index order | FIFO/LIFO | ❌ | ❌ | insertion/access | **sorted** |

\* amortized

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| ArrayList vs LinkedList? | ArrayList: O(1) random access, compact, cache-friendly. LinkedList: O(1) at the ends and at an iterator's position, but slow in practice. Prefer ArrayList (and ArrayDeque for queues). Details in #018, #020 |
| HashMap vs TreeMap? | O(1) with no order vs O(log n) sorted with navigation. Hash needs equals/hashCode; tree needs compareTo |
| HashMap vs Hashtable vs ConcurrentHashMap? | Not thread-safe, allows nulls / legacy, synchronised on every method, no nulls / thread-safe with fine-grained locking, no nulls |
| How does HashSet work? | It's a HashMap whose keys are the elements and whose values are a dummy object |
| Why can't ArrayDeque or ConcurrentHashMap hold `null`? | `null` is the "absent/empty" return value of `poll`/`get`; allowing it would make results ambiguous |
| Is `Collections.unmodifiableList` immutable? | No: a read-only **view**; the underlying list can still change |
| What causes `ConcurrentModificationException`? | A structural change during iteration that doesn't go through the iterator (even single-threaded) |
| Why is iterating a PriorityQueue not sorted? | It's a heap; only the root is guaranteed to be the minimum |
| Why must keys be immutable? | A key whose hash changes is filed in the wrong bucket and can no longer be found |
| Stack class? | Legacy (`extends Vector`); use `ArrayDeque` |

---

## Handwritten Cheat Sheet

```
DEFAULTS: ArrayList · HashMap · HashSet · ArrayDeque (stack+queue) · PriorityQueue
ORDER:    insertion → Linked*   sorted → Tree*   enum → Enum*
HASH  = O(1), needs equals+hashCode      TREE = O(log n), needs compareTo (consistent w/ equals)
LIST:  ArrayList get O(1), add-end O(1)*, mid O(n) | LinkedList get O(n), avoid
SET:   HashSet = HashMap keys | LinkedHashSet keeps order | TreeSet floor/ceiling
QUEUE: ArrayDeque circular array, no nulls | PQ = binary heap, peek O(1), poll O(log n),
       iteration NOT sorted
MAP:   HashMap 1 null key | TreeMap no null key | CHM no nulls
       LinkedHashMap(16,.75f,true)+removeEldestEntry = LRU
       merge / computeIfAbsent / getOrDefault
IMMUTABLE: List.of/copyOf (no nulls, UOE)  vs  unmodifiableList = VIEW
ITERATE:   modify during for-each → CME → use removeIf / iterator.remove
KEYS:      never mutate a key inside a HashMap/HashSet
JAVA 21:   getFirst/getLast/addFirst/reversed on sequenced collections
```

---

**Next note:** #019 — Read: Effective Java + Java docs (Collections & Generics) · **Build task:** #018 Benchmark ArrayList vs LinkedList
