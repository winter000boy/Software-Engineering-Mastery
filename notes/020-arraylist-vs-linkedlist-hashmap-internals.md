# #020 — ArrayList vs LinkedList; HashMap Internals (Buckets, Resize, Treeify)

**Date:** 2026-09-24  
**Roadmap row:** 20 — P1 Java Core · Collections & Generics · *Learn*  
**Sources:** OpenJDK 21 source (`java/util/ArrayList.java`, `LinkedList.java`, `HashMap.java`, `jdk/internal/util/ArraysSupport.java`, taken from the JDK's `lib/src.zip`)

> **Every code excerpt below is copied from the JDK 21.0.10 source**, and every "✔︎ observed" result came from running the experiment on JDK 21 (reflection on private fields with `--add-opens java.base/java.util=ALL-UNNAMED`). Measured performance is in #018.

---

## At a Glance (copy to handwritten notes)

- **`ArrayList`** = `Object[] elementData` + `size`. Default capacity **10**, allocated **lazily** on the first `add`. Grows by **×1.5** (`old + (old >> 1)`) via `Arrays.copyOf`. It **never shrinks** by itself (`trimToSize()`).
- Middle insert/remove = `System.arraycopy` shift, **O(n)** but very fast (a `memmove`). `remove` **nulls the freed slot** so GC can reclaim it (EJ Item 7).
- **`LinkedList`** = doubly linked `Node{item, next, prev}` + `first`/`last`. `get(i)` walks from the **nearer end**: O(n/2). Each element costs a **24-byte node** (✔︎ measured).
- **`HashMap`** = `Node<K,V>[] table`, **power-of-two** length, default **16**, load factor **0.75**, lazily allocated.
- **`hash(key) = h ^ (h >>> 16)`** mixes the high bits into the low bits. **Bucket index = `(n - 1) & hash`**, a bit-mask instead of `%`.
- **Resize** when `size > threshold` (`capacity × loadFactor`): the table **doubles**, and each bucket **splits** into "lo" (same index) and "hi" (index + oldCap) by testing **one bit**, `hash & oldCap`. No rehashing needed.
- **Treeify:** a bucket with **≥ 8** nodes becomes a **red-black tree** (`TreeNode`), but **only if the table has ≥ 64** buckets; otherwise the table **resizes** instead ✔︎. It turns back into a list at **≤ 6**.
- Lookup: go to the bucket, compare **stored hash** first, then `==`, then `equals`. That's why **`equals` + `hashCode` must be consistent** (EJ Items 10–11).
- **Presizing:** `new HashMap<>(100)` gives 128 buckets (threshold 96), so it **still resizes** for 100 entries ✔︎. `HashMap.newHashMap(100)` (Java 19+) sizes it correctly ✔︎.
- Java 7's HashMap inserted at the **head** of a bucket, and a concurrent resize could create a **cycle** (an infinite loop). Java 8+ inserts at the **tail** and preserves order on split, but it's still **not thread-safe**: use `ConcurrentHashMap`.

---

## Part 1 — `ArrayList` Internals

### 1.1 Fields

```java
private static final int DEFAULT_CAPACITY = 10;
private static final Object[] EMPTY_ELEMENTDATA = {};                 // for new ArrayList<>(0)
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; // for new ArrayList<>()
transient Object[] elementData; // non-private to simplify nested class access
private int size;
```

- `elementData` is an **`Object[]`**, not an `E[]`, because generic arrays can't be created (EJ Item 28). Elements are cast back on read: `(E) elementData[index]` (EJ Item 29, option 2).
- `transient`: serialization writes only the `size` live elements, not the empty tail of the array.
- Two different empty-array sentinels exist so `grow()` can tell *"default constructor → jump straight to 10"* apart from *"explicitly asked for capacity 0 → grow minimally"*.

### 1.2 Growth, from the source

```java
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
                minCapacity - oldCapacity, /* minimum growth */
                oldCapacity >> 1           /* preferred growth */);
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}

// jdk.internal.util.ArraysSupport
public static int newLength(int oldLength, int minGrowth, int prefGrowth) {
    int prefLength = oldLength + Math.max(minGrowth, prefGrowth); // might overflow
    if (0 < prefLength && prefLength <= SOFT_MAX_ARRAY_LENGTH) {  // Integer.MAX_VALUE - 8
        return prefLength;
    } else {
        return hugeLength(oldLength, minGrowth);
    }
}
```

**✔︎ Observed capacities** while adding 200 elements to `new ArrayList<>()`:

```
0 (lazy) → 10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244
```

Each step is `old + old/2`. After removing an element, capacity stays at 244; `trimToSize()` → 199 ✔︎.

**Why ×1.5 and not ×2?** It's a trade-off. Growth by a constant *factor* keeps `add` **amortized O(1)**: with n appends the total copying is O(n), since each copy is paid for by the appends before it. A smaller factor wastes less memory (≤ 33% unused after a grow, vs ≤ 50% with ×2).

**Amortized O(1), intuitively:** to reach 1 000 000 elements, the list copies roughly 1M × (1 + 2/3 + 4/9 + …) ≈ **3M** element moves in total. That's about **3 moves per element**, a constant.

### 1.3 Add / insert / remove, from the source

```java
// append (the common path)
private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}

// insert at an index: shift the tail right by one
public void add(int index, E element) {
    rangeCheckForAdd(index);
    modCount++;
    final int s;
    Object[] elementData;
    if ((s = size) == (elementData = this.elementData).length)
        elementData = grow();
    System.arraycopy(elementData, index,
                     elementData, index + 1,
                     s - index);
    elementData[index] = element;
    size = s + 1;
}

// remove: shift the tail left by one, then null the last slot
private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize;
    if ((newSize = size - 1) > i)
        System.arraycopy(es, i + 1, es, i, newSize - i);
    es[size = newSize] = null;          // ← EJ Item 7: eliminate the obsolete reference
}
```

- `add(0, e)` / `remove(0)` shift **all** n elements, so they're O(n). But `System.arraycopy` is an intrinsic (vectorised `memmove`). Shifting 100k references (≈400 KB) takes on the order of **microseconds**, which is why ArrayList often beats LinkedList even at O(n) (#018).
- `modCount++` on every **structural** change drives the **fail-fast** iterator (#017 §8).

### 1.4 Memory layout

```
ArrayList object (24 B)                       Object[] elementData (16 B header + 4 B × capacity)
┌──────────────────────┐                      ┌──────┬──────┬──────┬──────┬─────┬──────┐
│ header 12 B          │                      │ ref0 │ ref1 │ ref2 │ ref3 │ ... │ null │  (contiguous)
│ modCount 4 · size 4  │──elementData────────▶└──┬───┴──┬───┴──┬───┴──────┴─────┴──────┘
│ elementData ref 4    │                         ▼      ▼      ▼
└──────────────────────┘                     Integer Integer Integer     (16 B each, anywhere on the heap)
```

✔︎ **JOL measurement** (100 000 `Integer`s, compressed oops): the ArrayList structure is **426 880 bytes**, i.e. a `106 710`-slot array × 4 B + headers ≈ **4.3 bytes per element** (#018).

---

## Part 2 — `LinkedList` Internals

### 2.1 Node and fields, from the source

```java
private static class Node<E> {     // static nested class: no hidden outer reference (EJ Item 24)
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}

transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

```
 first                                                    last
   │                                                        │
   ▼                                                        ▼
┌──────┐ next ┌──────┐ next ┌──────┐          ┌──────┐
│ null │◀──── │prev  │◀──── │prev  │   ...    │prev  │
│ item │ ───▶ │ item │ ───▶ │ item │ ───▶     │ item │ ───▶ null
│ next │ prev │ next │ prev │ next │          │ next │
└──────┘      └──────┘      └──────┘          └──────┘
  each Node: 12 B header + 3 refs × 4 B = 24 B, scattered across the heap
```

### 2.2 Indexed access walks from the nearer end, from the source

```java
Node<E> node(int index) {
    if (index < (size >> 1)) {          // first half → walk forward from first
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {                            // second half → walk backward from last
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

`get(i)`, `set(i)`, `add(i, e)` and `remove(i)` all call `node(i)` first: **O(min(i, n - i))**, which is O(n) in general. Linking and unlinking itself is O(1) (changing 4 pointers). So LinkedList's O(1) insertion only happens **at the ends** or **at an iterator position you already hold**.

### 2.3 Why LinkedList loses in practice

| Factor | ArrayList | LinkedList |
|--------|-----------|------------|
| Traversal | Sequential memory, so the CPU **prefetcher** streams the cache lines | **Pointer chasing**: each `next` can be a cache miss (~100 ns from DRAM vs ~1 ns from L1) |
| Per-element memory | ~4 B (a reference) | **24 B node** + the reference inside it (✔︎ JOL: 2 400 032 B for 100k) |
| GC | One array object | n node objects to allocate, trace and collect |
| Index loop `for (i…) get(i)` | O(n) | **O(n²)**. `LinkedList` doesn't implement `RandomAccess` for this reason (#019) |
| Its one true win | — | `ListIterator.add/remove` at a held position; O(1) at both ends (but `ArrayDeque` does that better) |

Joshua Bloch, who wrote `LinkedList`, has said publicly that he doesn't use it himself.

---

## Part 3 — `HashMap` Internals

### 3.1 Constants, from the source

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```

### 3.2 Structure

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;     // cached spread hash: never recomputed, compared before equals()
    final K key;
    V value;
    Node<K,V> next;     // next node in the same bucket
}

transient Node<K,V>[] table;   // length is ALWAYS a power of two
transient int size;
transient int modCount;
int threshold;                 // resize when size exceeds this (= capacity × loadFactor)
final float loadFactor;
```

```
table (capacity 16)
 idx
  0  ─ null
  1  ─▶ [h=17, k="x", v] ─▶ [h=33, k="y", v] ─▶ null        ← collision chain (singly linked)
  2  ─ null
  3  ─▶ [h=3, k="z", v] ─▶ null
  …
  9  ─▶ TreeNode (red-black tree root)                       ← bucket with ≥ 8 nodes, table ≥ 64
        ╱        ╲
     TreeNode  TreeNode
  …
 15  ─ null
```

### 3.3 Step 1: spreading the hash, from the source

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

- **`null` key → hash 0 → bucket 0.** That's why `HashMap` allows exactly one `null` key.
- **Why `h ^ (h >>> 16)`?** The bucket index uses only the **low bits** (`(n-1) & hash`). With a 16-slot table that's just the lowest **4** bits. Hash codes that differ only in their high bits (common with `Float` keys, or small integers shifted left) would all collide. XOR-ing the top 16 bits into the bottom 16 lets the high bits affect the index, at the cost of one shift and one XOR.

✔︎ Observed: `h = 0x12345678` → `h >>> 16 = 0x00001234` → spread `0x1234444c` → index in a 16-table = **12** (it would be **8** without the spread).

### 3.4 Step 2: bucket index

```java
index = (n - 1) & hash      // n = table.length, a power of two
```

When `n` is a power of two, `n - 1` is all ones in binary (16 − 1 = `0b1111`), so `& (n - 1)` equals `hash mod n` for non-negative values, but it's a single AND instead of a division, and it **works for negative hashes** (`%` would give a negative index). This is **why** the capacity must be a power of two, and why the constructor rounds up with `tableSizeFor`:

```java
static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

✔︎ Observed: `new HashMap<>(10)` → 16 · `(16)` → 16 · `(17)` → 32 · `(100)` → 128 · `(1000)` → 1024.

### 3.5 Step 3: `putVal`, from the source (annotated)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;                      // ① LAZY: table allocated on first put
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);         // ② empty bucket → just place the node
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;                                        // ③ first node is the same key
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);   // ④ tree bucket
        else {
            for (int binCount = 0; ; ++binCount) {        // ⑤ walk the chain
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);   //   append at the TAIL (Java 8+)
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);            //   chain reached 8+ → try to treeify
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;                                //   found an existing key
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;                          // ⑥ replace the value; return the old one
            afterNodeAccess(e);                           //   hook: LinkedHashMap moves it (access order)
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();                                         // ⑦ grow after inserting
    afterNodeInsertion(evict);                            //   hook: LinkedHashMap.removeEldestEntry → LRU
    return null;
}
```

**The equality check order matters:** `p.hash == hash` (a cheap int compare, which rules out most non-matches) → `==` (the same object) → `equals` (possibly expensive). If `hashCode` is inconsistent with `equals`, step ③/⑤ never matches, and a second copy of the "same" key gets inserted (EJ Item 11).

`afterNodeAccess` / `afterNodeInsertion` are empty in `HashMap`. They're the **template-method hooks** that `LinkedHashMap` overrides to maintain its linked list and support LRU eviction (#017, rows 32–36).

### 3.6 Step 4: `getNode`, from the source

```java
final Node<K,V> getNode(Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & (hash = hash(key))]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);   // O(log k) in the bucket
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);                             // O(k) chain walk
        }
    }
    return null;
}
```

### 3.7 Resize, from the source (key part)

Triggered when `++size > threshold`. The table **doubles**, and so does the threshold.

```java
else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
         oldCap >= DEFAULT_INITIAL_CAPACITY)
    newThr = oldThr << 1; // double threshold
...
for (int j = 0; j < oldCap; ++j) {
    Node<K,V> e;
    if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)
            newTab[e.hash & (newCap - 1)] = e;                         // single node: re-index it
        else if (e instanceof TreeNode)
            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);         // tree: split (may untreeify)
        else { // preserve order
            Node<K,V> loHead = null, loTail = null;
            Node<K,V> hiHead = null, hiTail = null;
            Node<K,V> next;
            do {
                next = e.next;
                if ((e.hash & oldCap) == 0) {   // the ONE new index bit is 0 → stays at j
                    if (loTail == null) loHead = e; else loTail.next = e;
                    loTail = e;
                }
                else {                           // the new bit is 1 → moves to j + oldCap
                    if (hiTail == null) hiHead = e; else hiTail.next = e;
                    hiTail = e;
                }
            } while ((e = next) != null);
            if (loTail != null) { loTail.next = null; newTab[j] = loHead; }
            if (hiTail != null) { hiTail.next = null; newTab[j + oldCap] = hiHead; }
        }
    }
}
```

**The trick:** going from `n` to `2n` adds **one bit** to the index mask. Each node either keeps its index `j` (that bit is 0) or moves to `j + oldCap` (that bit is 1). The stored hash is reused, so **no `hashCode()` calls happen during resize**.

✔︎ Observed with keys 5, 21 and 37 (all in bucket 5 when n = 16): after resizing to 32, `hash & 16` = 0, 16, 0, so 5 and 37 stay in bucket **5** and 21 moves to bucket **21** (= 5 + 16).

✔︎ **Observed resize points** for `new HashMap<>()`, which match `threshold = capacity × 0.75`:

| After put # | Capacity | Threshold |
|------------:|---------:|----------:|
| 1 | 16 (lazy allocation) | 12 |
| 13 | 32 | 24 |
| 25 | 64 | 48 |
| 49 | 128 | 96 |
| 97 | 256 | 192 |

Resize is O(n), but it happens at doubling sizes, so `put` stays **amortized O(1)**. Individual puts that trigger a resize are slow (a latency spike), which is one reason to **presize** maps whose size you know.

### 3.8 Treeification, from the source

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();                                  // small table: RESIZE rather than treeify
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // convert the Node chain into a TreeNode chain, then build the red-black tree
        ...
        hd.treeify(tab);
    }
}
```

- **Why?** A bucket that's a linked list makes lookups O(k), and with a terrible `hashCode` (or a deliberate **hash-flooding DoS attack** using crafted colliding keys, like the `"Aa"`/`"BB"` string collisions) k → n, so O(n) per lookup. As a **red-black tree**, the bucket is O(log k).
- **Why only when the table has ≥ 64 buckets?** In a small table, a long chain more likely means *the table is too small*, not that the hashes are bad. Resizing spreads the chain out and is cheaper than building a tree.
- **Why 8?** With a reasonable hash and load factor 0.75, bucket sizes follow a **Poisson distribution** with λ ≈ 0.5. The source comment gives the probability of a bucket reaching 8 entries as **0.00000006** (about 1 in 10 million). So trees only appear when something is wrong with the hashes.
- **Why back at 6, not 8?** **Hysteresis**: the gap between 8 and 6 stops a bucket flipping between list and tree when its size hovers around the threshold.
- **How does the tree order keys?** By hash first; for equal hashes, by `compareTo` **if the keys are `Comparable`** and of the same class; otherwise by a tie-breaker (`identityHashCode`). **Make keys `Comparable`** if collisions are possible, or the tree can't search efficiently.
- A `TreeNode` is about twice the size of a `Node` (parent/left/right/prev + a red flag), another reason to avoid trees unless needed.

✔︎ **Observed timeline**: inserting keys that all have `hashCode() == 42`:

| Colliding put # | Table capacity | Bucket type | What happened |
|----------------:|---------------:|-------------|---------------|
| 1–8 | 16 | `Node` | Chain grows to 8 |
| 9 | **32** | `Node` | Chain would hold 9 → `treeifyBin` → table < 64 → **resize** instead |
| 10 | **64** | `Node` | Still < 64 at the check → **resize** again |
| 11 | 64 | **`TreeNode`** | Table is now ≥ 64 → **treeify** |
| 12 | 64 | `TreeNode` | Inserted into the red-black tree |

### 3.9 Complexity summary

| Operation | Average | Worst (Java 8+) | Worst (Java 7) |
|-----------|---------|-----------------|----------------|
| `get` / `containsKey` | O(1) | O(log n) (tree bucket) | O(n) |
| `put` | O(1) amortized | O(log n) + occasional O(n) resize | O(n) |
| `remove` | O(1) | O(log n) | O(n) |
| Iteration | O(capacity + size) | same | same |

Iteration visits **every bucket**, including empty ones. A map that was grown to 1M entries and then cleared still iterates over 1M+ empty slots, since the table **never shrinks**. Create a new map instead.

### 3.10 Presizing ✔︎

```java
new HashMap<>(100);        // capacity 128, threshold 96 → inserting 100 entries RESIZES once ✔︎
HashMap.newHashMap(100);   // Java 19+: capacity computed as ceil(100 / 0.75) → 256, 0 resizes ✔︎
// pre-19 idiom:
new HashMap<>((int) Math.ceil(expected / 0.75));
```

The `int` constructor argument is the **initial capacity (number of buckets)**, *not* the number of entries. That's one of the most common HashMap misunderstandings, and it's why `newHashMap(int numMappings)` was added.

### 3.11 Java 7 vs Java 8+ HashMap

| | Java 7 | Java 8+ |
|---|---|---|
| Bucket structure | Linked list only | List → red-black tree at ≥ 8 (table ≥ 64) |
| Insert position | **Head** of the chain | **Tail** of the chain |
| Resize | Re-inserts each node at the head → **reverses** order | lo/hi split **preserves** order |
| Concurrent resize bug | Two threads resizing could create a **cycle** → `get` spins forever (100% CPU) | No cycle, but lost updates and corrupted size are still possible |
| Hash function | Several shifts/XORs | A single `h ^ (h >>> 16)` (trees handle bad hashes) |

**Lesson:** `HashMap` is **not thread-safe** in any version. For shared maps use `ConcurrentHashMap` (P1 Concurrency, rows 67–91).

### 3.12 How the other maps and sets relate

| Class | Relation to HashMap |
|-------|---------------------|
| `HashSet<E>` | Wraps a `HashMap<E, Object>`; every value is the same static `PRESENT` object |
| `LinkedHashMap` | `extends HashMap`; its entries add `before`/`after` pointers (a doubly linked list through all entries); overrides the `afterNode*` hooks |
| `LinkedHashSet` | `extends HashSet`, backed by a `LinkedHashMap` |
| `ConcurrentHashMap` | Similar table + treeify design, but with CAS on empty buckets, `synchronized` on the bucket's first node for updates, and cooperative multi-threaded resize. No `null`s |
| `Hashtable` | Legacy: `% length` indexing, synchronised methods, no `null`s, no treeify |

---

## Part 4 — ArrayList vs LinkedList: Decision Summary

Measured in #018 (Apple M4, JDK 21, JMH; ratios at n = 100 000):

| Operation | ArrayList | LinkedList | Winner in practice (measured) |
|-----------|-----------|------------|-------------------------------|
| `add(e)` at end | Amortized O(1) | O(1) + a node allocation | **ArrayList**, 1.6× |
| `get(i)` | O(1) | O(n/2) | **ArrayList**, ≈ 60 000× (1 000 random gets) |
| Iterate | O(n), sequential | O(n), pointer chasing | **ArrayList**, 5.2× |
| `contains` | O(n) | O(n) | **ArrayList**, 4.4× |
| `add(0, e)` / `remove(0)` | O(n) arraycopy | O(1) | LinkedList, 2 197×, **but `ArrayDeque` is 2.3× faster still** |
| Insert in the middle **by index** | O(n) shift | O(n) walk + O(1) link | **ArrayList**, 19× (the walk is slower than the shift) |
| Insert at a held `ListIterator` position | O(n) shift | **O(1)** | **LinkedList**, 76×: its one real win |
| Memory per element (structure) | 4.3 B | 24 B | **ArrayList**, 5.6× less |

**Rule:** `ArrayList` by default; `ArrayDeque` for queues and stacks; `LinkedList` only when you've *measured* an iterator-position-insert workload that benefits.

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| How does `HashMap.put` work? | `hash()` spreads the hash → `(n-1) & hash` picks the bucket → empty: place it; otherwise walk the chain/tree comparing hash, then `==`/`equals` → replace or append → `++size > threshold` → resize |
| Why is capacity a power of two? | So `(n-1) & hash` replaces `%` (fast, and safe for negative hashes), and resize can split buckets with one bit |
| Why `h ^ (h >>> 16)`? | Small tables use only the low bits; this mixes the high bits in so they affect the index |
| Load factor 0.75: why? | A time/space trade-off. Lower → fewer collisions but more memory; higher → less memory but longer chains |
| What happens on resize? | Capacity ×2, threshold ×2, each bucket splits into lo (same index) / hi (index + oldCap) using `hash & oldCap`, order preserved, no rehash |
| When does a bucket become a tree? | ≥ 8 nodes **and** table ≥ 64 (otherwise resize); back to a list at ≤ 6 |
| Why not always use trees? | TreeNodes are ~2× bigger and slower for small k; with good hashes buckets are tiny (Poisson) |
| What if two keys are equal but have different hashCodes? | They land in different buckets: duplicate "equal" keys, and `get` misses |
| Can a `HashMap` have a null key? | Yes, one; it goes in bucket 0 (hash 0). `ConcurrentHashMap`/`Hashtable`/`TreeMap` (natural order): no |
| Java 7 infinite loop? | Head insertion + concurrent resize could create a cycle in a bucket; fixed by tail insertion in 8, but HashMap is still not thread-safe |
| `new HashMap<>(100)` for 100 entries? | Capacity 128, threshold 96, so it still resizes; use `HashMap.newHashMap(100)` |
| ArrayList growth? | Lazy 10, then ×1.5 via `Arrays.copyOf`; never shrinks (`trimToSize`) |
| Why does `ArrayList.remove` null out the slot? | So the removed object can be garbage-collected (EJ Item 7) |
| LinkedList `get(i)` complexity? | O(min(i, n-i)): walks from the nearer end |

---

## Handwritten Cheat Sheet

```
ARRAYLIST: Object[] elementData + size | lazy cap 10 | grow ×1.5 (old + old>>1) Arrays.copyOf
           caps: 10 15 22 33 49 73 109 163 244 | never shrinks (trimToSize)
           add(i)/remove(i) = System.arraycopy shift O(n) | remove nulls slot (EJ7)
           ~4 B/elem structure (JOL)
LINKEDLIST: Node{item,next,prev} 24 B | first/last | node(i) walks from nearer end
           O(1) only at ends or held ListIterator | not RandomAccess → index loop O(n²)

HASHMAP: Node{final hash, final key, value, next}[] table, length = 2^k
  defaults: cap 16 (lazy) · LF 0.75 · threshold = cap × LF
  hash = h ^ (h >>> 16)   (null key → 0 → bucket 0)
  index = (n-1) & hash    (power of 2 ⇒ & == mod)
  put: empty → place | match: hash == && (== || equals) → replace | else append TAIL
       ++size > threshold → resize
  resize: cap×2, thr×2, split bucket by (hash & oldCap): 0 → j, 1 → j+oldCap, order kept
  resizes (default): puts #1→16, #13→32, #25→64, #49→128, #97→256
  treeify: bin ≥ 8 AND table ≥ 64 (else resize) → red-black tree O(log k); untreeify ≤ 6
           make keys Comparable; P(bin ≥ 8) ≈ 6e-8 with good hashes
  presize: HashMap.newHashMap(n) (19+) — new HashMap<>(100) still resizes!
  Java 7: head insert → concurrent resize cycle; Java 8: tail insert. STILL NOT THREAD-SAFE
  hooks afterNodeAccess/afterNodeInsertion → LinkedHashMap LRU
```

---

**Next:** row 21 — Implement your own dynamic array (use Part 1 as the spec) · row 24 — Implement a simplified HashMap (use Part 3 as the spec)
