# #028 — Make Anki Cards for Collections Concepts

**Date:** 2026-09-24  
**Roadmap row:** 28 — P1 Java Core · Collections & Generics · *Flex*  
**Covers:** #017 (List/Set/Map), #018 (benchmark), #019 (EJ generics + Javadoc), #020 (internals), #021 (dynamic array), #022 (exercises), #023 (TreeMap/LinkedHashMap), #024–#025 (HashMap build + review), #026 (generics/PECS), #027 (comparators), #029 (Comparable vs Comparator)

---

## At a Glance

- **82 cards**, grouped by topic in roadmap order, ready to import (pipe-separated, one card per line). The format was checked automatically: every line has exactly one `|`.
- Every answer is **verified**: it matches a measured or tested result in the notes above. Numbers such as resize points, capacities and benchmark ratios come from real runs.
- **Import once, then review daily** (≈ 10–15 min). Don't add more than 20 new cards a day.
- These complement the Week 1 deck in #014. Use the same note type, and tag this one `p1 week2`.

---

## 1. How to Import

1. Copy the whole code block in §3 into a file `week2-collections.txt`. **Keep the three `#` header lines.**
2. Anki desktop → **File → Import** → pick the file. Anki ≥ 2.1.55 reads the headers automatically (separator = pipe, tags = `java p1 week2 collections`). On older versions choose *Fields separated by: Pipe* by hand.
3. Note type: **Basic**. For the definition cards marked `(rev)`, you can switch those notes to **Basic (and reversed card)** afterwards.
4. Optional: put the section name in a second tag (`internals`, `generics`…) if you want filtered decks.

---

## 2. Principles Used to Write These Cards (so you can add your own)

| Principle | Example |
|-----------|---------|
| **One fact per card** | Not "Explain HashMap"; instead *"HashMap bucket index formula?"* |
| **Ask for the *why*, not only the *what*** | *"Why must HashMap capacity be a power of two?"* |
| **Include numbers you measured**, because they make the facts concrete | *"LinkedList vs ArrayList random get, 100k elements (#018)?"* |
| **Cards for your mistakes** | Each self-test miss or bug you hit becomes a card (e.g. `.reversed()` binding, the JOL varargs mistake) |
| **Code on the front, behaviour on the back** | *"`List.of(1,2).contains(null)` → ?"* |
| Keep answers **≤ ~20 words** | Long answers mean you should split the card |

---

## 3. The Deck (82 cards)

```text
#separator:pipe
#html:false
#tags:java p1 week2 collections
Default choice for a List? A Map? A Set? A stack or queue?|ArrayList, HashMap, HashSet, ArrayDeque (for both stack and queue)
Is Map a Collection?|No. Map is a separate hierarchy; it offers keySet/values/entrySet views that are Collections
Hash-based vs tree-based collections: cost and requirement?|Hash: O(1) average, needs equals+hashCode. Tree: O(log n), sorted, needs compareTo or a Comparator
HashSet is implemented how?|A HashMap whose keys are the elements and whose values are one shared dummy object (PRESENT)
Which collection keeps insertion order with O(1) lookups?|LinkedHashMap / LinkedHashSet
Which collection answers "greatest key <= x" efficiently?|TreeMap.floorKey / TreeSet.floor, O(log n)
PriorityQueue: data structure and costs?|Binary min-heap in an array. offer/poll O(log n), peek O(1), remove(Object) O(n)
Does iterating or printing a PriorityQueue give sorted order?|No, heap order (e.g. [1, 2, 4, 5, 3]). Only repeated poll() is sorted
Why can't ArrayDeque or ConcurrentHashMap hold null?|null is the "empty/absent" return value of poll/get, so it would be ambiguous
EnumSet internal representation?|A bit vector: one long for up to 64 constants (RegularEnumSet), else long[] (JumboEnumSet)
List.of(...): mutable? nulls allowed? contains(null)?|Immutable (UnsupportedOperationException); nulls rejected; even contains(null) throws NPE
Collections.unmodifiableList vs List.copyOf?|unmodifiableList is a read-only VIEW (source changes show through); copyOf is an immutable snapshot
Arrays.asList(1,2): add(3)? set(0,9)?|add throws UnsupportedOperationException (fixed size); set works and writes through to the array
Set.of/Map.of iteration order?|Deliberately randomised per JVM run (SALT); never depend on it
What causes ConcurrentModificationException in single-threaded code?|A structural change during iteration that doesn't go through the iterator; fix with removeIf or iterator.remove
How does fail-fast iteration work?|The collection bumps modCount on structural changes; the iterator compares it with expectedModCount on each next()
Why must keys in a HashMap/HashSet not be mutated?|A changed hashCode files the key in the wrong bucket, so it can no longer be found (a ghost entry)
Java 21 sequenced-collection methods?|addFirst, addLast, getFirst, getLast, removeFirst, removeLast, reversed() (plus putFirst/firstEntry on maps)
ArrayList internal storage?|An Object[] elementData plus an int size (Object[] because generic arrays can't be created)
ArrayList default capacity and when is it allocated?|10, allocated lazily on the first add
ArrayList growth rule and capacity sequence?|new = old + old>>1 (x1.5): 10, 15, 22, 33, 49, 73, 109, 163, 244
Does ArrayList ever shrink its array automatically?|No. Only trimToSize() shrinks it
Why is ArrayList.add amortized O(1)?|Growth by a constant factor makes total copying O(n) over n adds, about 3 moves per element
Why does ArrayList.remove null out the freed slot?|So the removed object becomes garbage-collectable (EJ Item 7, obsolete references)
Which ArrayList operations are O(n) and why are they still fast?|add/remove at an index: they shift via System.arraycopy, a vectorised memmove (~7 us per 100k refs measured)
LinkedList node structure and size?|Node{item, next, prev}: 12 B header + 3 refs = 24 B per element (JOL)
How does LinkedList.get(i) find the node?|Walks from the nearer end: first->next if i < size/2, else last->prev. O(n)
Does LinkedList implement RandomAccess? Consequence?|No. An index loop for(i..) get(i) on it is O(n^2)
HashMap default capacity, load factor, first resize?|16 (allocated lazily), 0.75, threshold 12, so it resizes on the 13th put
HashMap hash() function?|h ^ (h >>> 16): mixes the high bits into the low bits (null key -> hash 0)
HashMap bucket index formula?|(n - 1) & hash, where n = table length (a power of two)
Why must HashMap capacity be a power of two?|So (n-1) & hash replaces modulo (fast, safe for negatives) and resize can split buckets with one bit
HashMap equality check order during lookup?|Stored hash ==, then key ==, then key.equals
Java 8+ HashMap: insert at the head or the tail of a bucket?|Tail (Java 7 inserted at the head, enabling a concurrent-resize cycle)
HashMap resize: what happens to each bucket?|Table doubles; each node stays at j or moves to j + oldCap depending on (hash & oldCap); order preserved; no rehash
HashMap treeify conditions?|A bucket reaches 8 nodes AND the table has >= 64 buckets (otherwise it resizes instead); back to a list at 6
Why treeify buckets at all?|Bad hashes or hash-flooding attacks make a chain O(n); a red-black tree bin is O(log n)
Why is treeify rarely triggered with good hashes?|Bucket sizes are Poisson with lambda ~0.5; P(bucket reaches 8) is about 6 in 100 million
new HashMap<>(100): capacity? Does inserting 100 entries resize?|128 buckets, threshold 96, so yes it resizes once. Use HashMap.newHashMap(100) (Java 19+)
What is the int argument to new HashMap<>(n)?|The initial number of BUCKETS (capacity), not the expected number of entries
Is HashMap thread-safe in Java 8+?|No. Use ConcurrentHashMap (no more infinite-loop cycles, but updates can still be lost)
Why is LinkedList slower than ArrayList to iterate although both are O(n)?|Pointer chasing: each node load is a potential cache miss; arrays are contiguous and prefetched
Measured (#018, 100k): LinkedList vs ArrayList for 1000 random get(i)?|About 60,000x slower
Measured (#018, 100k): add(0)/remove(0) x1000, LinkedList vs ArrayList?|LinkedList ~2,200x faster (O(1) vs O(n) shifts)
Measured (#018): best FIFO queue among ArrayDeque, LinkedList, ArrayList?|ArrayDeque (~2.2x faster than LinkedList, constant with size); ArrayList remove(0) is O(n)
LinkedList's one real advantage (measured 76x at 100k)?|Inserting or removing at a position you already hold with a ListIterator
Why use JMH instead of System.nanoTime loops?|Handles JIT warm-up, dead-code elimination, forks, and gives error bars
LinkedHashMap structure?|HashMap buckets + a doubly linked list (before/after) through all entries; head = eldest, tail = newest
LinkedHashMap: does re-putting an existing key change its position?|No (insertion order). remove + put moves it to the end
LinkedHashMap access order: which operations move an entry to the end?|get, getOrDefault, put/replace on an existing key, compute*, merge. Not containsKey, not iteration
LRU cache in two lines with LinkedHashMap?|new LinkedHashMap<>(cap, 0.75f, true) + override removeEldestEntry to return size() > max
Why can get() throw CME on a LinkedHashMap?|In access order, get moves the entry, which is a structural modification
TreeMap data structure and height bound?|Red-black tree; height <= 2 log2(n+1); 1M keys measured 20 bulk, 24 random, 37 ascending
Does TreeMap ever call equals or hashCode on keys?|No. compare/compareTo == 0 means "same key"
TreeMap null key with natural ordering?|NullPointerException (possible only with a null-tolerant comparator)
floorKey vs lowerKey; ceilingKey vs higherKey?|floor <=, lower <; ceiling >=, higher >
Are TreeMap.subMap/headMap/tailMap copies?|No, live range views; a put outside the range throws IllegalArgumentException
Case-insensitive TreeMap: put("Apple",1) then put("apple",2)?|One entry {Apple=2}: the comparator says they're the same key
Why does HashMap iteration order change after a resize?|Buckets split into lo/hi, so entries that shared a bucket end up at different indexes
Is List<Integer> a subtype of List<Number>?|No, generics are invariant. It is a subtype of List<? extends Number>
PECS stands for?|Producer Extends, Consumer Super: read from ? extends T, write into ? super T
What can you do with a List<? extends Number>?|Read elements as Number; you cannot add anything except null
What can you do with a List<? super Integer>?|Add Integers; reads only give Object
Canonical JDK example of PECS?|Collections.copy(List<? super T> dest, List<? extends T> src)
Why Comparator<? super T> rather than Comparator<T>?|Comparators consume T, so a comparator for a supertype (e.g. Comparator<Object>) also works
Why <T extends Comparable<? super T>> in max()?|So types that inherit compareTo from a supertype qualify (java.sql.Timestamp is Comparable<Date>)
List<?> vs List<Object> vs raw List?|Unknown type, type-safe, no adds / holds anything, accepts only List<Object> / no checks at all
What is CAP#1 in a compiler error?|The compiler's name for a captured wildcard type
How do you write through a List<?> (e.g. swap)?|Delegate to a private generic helper <E> void helper(List<E> l) that captures the type
Multiple bounds syntax and erasure?|<T extends Number & Comparable<? super T>>; the class comes first; erasure is the first bound
What is a bridge method?|A compiler-generated method that keeps overriding working after erasure, e.g. compareTo(Object) -> checkcast -> compareTo(Money)
Why is new T() or new T[n] illegal? Workaround?|T is erased at run time. Pass a Supplier<T>, an IntFunction<T[]>, or a Class<T> token
Why can't you overload m(List<String>) and m(List<Integer>)?|Both erase to m(List): "name clash ... have the same erasure"
Comparable vs Comparator?|Comparable: natural order inside the class, one per class. Comparator: external, many, for any class
What does "consistent with equals" mean for compareTo?|compareTo == 0 exactly when equals is true; otherwise TreeSet/TreeMap merge "equal" elements
BigDecimal 2.0 vs 2.00 in HashSet vs TreeSet?|HashSet: 2 elements (not equals). TreeSet: 1 element (compareTo == 0)
"Comparison method violates its general contract!" means?|TimSort detected a broken comparator (not antisymmetric or not transitive)
Why is (a, b) -> a - b a bad comparator?|Integer overflow can flip the sign (MAX - (-1) wraps negative); use Integer.compare
comparing(A).thenComparing(B).reversed() reverses what?|The whole chain (both keys). To reverse only B: thenComparing(B, Comparator.reverseOrder())
Why doesn't Comparator.comparing(p -> p.name()).reversed() compile?|No target type, so p is inferred as Object; use P::name or (P p) -> p.name()
Sorting algorithms: objects vs primitives? Stable?|Objects: TimSort, stable. Primitives: dual-pivot quicksort (stability irrelevant)
How do you sort human-visible names correctly (accents, case)?|Collator.getInstance(locale) as the key comparator
```

---

## 4. Card Checklist for the Rest of the Week

Add a card whenever one of these happens:

- [ ] A test you wrote failed for a reason you didn't expect (e.g. #024: the resize test's filler key `101` also landed in bucket 5)
- [ ] A measurement surprised you (#018 results; #025 collision cost: 737× slower at 50k colliding keys)
- [ ] A compile error you had to look up (#026's `CAP#1`, the anonymous-diamond error)
- [ ] An API you reached for and didn't know (`HashMap.newHashMap`, `NavigableMap.floorEntry`, `Collections.checkedList`)

Re-export the deck at the end of the week (row 39 weekly review) and prune any card you've answered correctly more than 10 times in a row with long intervals. Anki handles that on its own via long intervals; just don't delete cards you still miss.

---

**Next:** row 29 — Comparable vs Comparator (#029)
