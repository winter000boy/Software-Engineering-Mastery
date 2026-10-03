# #039 — Weekly Review + Spaced Repetition (Anki); Fix Gaps — Week 2

**Date:** 2026-10-03  
**Roadmap row:** 39 — P1 Java Core · Collections & Generics · *Review*  
**Covers:** rows 17–38: #017 List/Set/Map · #018 benchmark · #019 EJ generics · #020 internals · #021 dynamic array · #022 exercises · #023 TreeMap/LinkedHashMap · #024–#025 HashMap build + review · #026 generics · #027 comparators · #028 Anki · #029 Comparable vs Comparator · #030 Multiset · #031 gap-fill · #032–#036 LRU project · #037–#038 Big-O

---

## At a Glance (copy to handwritten notes)

- **Week 2 in one line:** *pick the structure by its operations' cost → know how it's built → type it safely with generics → order it correctly → prove it with tests and measurements.*
- **Self-test:** 30 questions below (answers hidden). Same scoring as #014. Every miss becomes a card.
- **Anki:** keep reviewing #028's 82 cards; **import the 32 new cards below** (rows 31–38).
- **Readiness bar for Week 3 (Concurrency I, rows 42–66):** ≥ 24/30, and you can explain *why* `HashMap` is not thread-safe and *what* `SynchronizedCache` protects.

---

## 1. The Week on One Page

```
CHOOSE (by operation cost)                 BUILD (know the internals)
 List  → ArrayList (ArrayDeque for ends)    ArrayList: Object[] ×1.5, arraycopy shifts       [#020, #021]
 Set   → HashSet / LinkedHashSet / TreeSet  HashMap: 2^k table, h^(h>>>16), lo/hi resize,
 Map   → HashMap / LinkedHashMap / TreeMap           treeify ≥8 & ≥64                        [#020, #024]
 Queue → ArrayDeque / PriorityQueue         LinkedHashMap: + before/after list, access order [#023]
 LRU   → LinkedHashMap(…,true)              TreeMap: red-black tree, height ≤ 2·log₂(n+1)    [#023]
         or HashMap + doubly linked list    LRU: map (find) + DLL (order), sentinels         [#032–#034]
                         [#017, #037]
TYPE SAFELY (generics)                      ORDER CORRECTLY
 invariance; PECS; ? super for consumers    Comparable (natural, one) vs Comparator (many)  [#029]
 bounds, recursive bounds, capture helper   combinators; .reversed() binds to the WHOLE chain
 erasure, bridge methods, no new T[]        consistent with equals or TreeSet drops data    [#027]
                         [#019, #026, #030]
PROVE IT
 contract tests in an abstract class · reference model + differential tests · JMH · doubling
 experiments · -Xlint:all · probe hostile collaborators (the listener bug)        [#018, #025, #035, #038]
```

---

## 2. Row-by-Row Recap

| Row | Topic | The one idea to keep | Re-read if shaky |
|----:|-------|----------------------|------------------|
| 17 | List/Set/Map, when to use | Hash = O(1) + `equals`/`hashCode`; tree = O(log n) + `compareTo` | #017 §11 decision guide |
| 18 | ArrayList vs LinkedList (measured) | Same Big-O can differ 5–60 000×; LinkedList only wins at a held iterator | #018 results table |
| 19 | EJ generics + Javadoc | PECS; no raw types; empty collections, not null | #019 Item 31 |
| 20 | Internals | `(n-1) & hash`, resize split, treeify conditions | #020 Part 3 |
| 21 | Dynamic array | Multiplicative growth ⇒ amortized O(1) | #021 Q3 |
| 22 | Exercises | Heap for top-k; LinkedHashSet; monotonic deque; TreeMap neighbours | #022 summary |
| 23 | TreeMap/LinkedHashMap | Access order + `removeEldestEntry` = LRU; range views are live | #023 §2.3, §4 |
| 24–25 | HashMap build + review | Implement `Map` via `AbstractMap`, overriding the O(n) defaults; measure weak spots | #025 Part 3 |
| 26 | Generics mechanics | Capture, bounds, erasure, bridge methods; real compiler errors | #026 §8 |
| 27 | Comparators (practice) | Collator for human text; tie-break on a unique key | #027 §3 |
| 28 | Anki | Cards come from misses and surprises | #028 §2 |
| 29 | Comparable vs Comparator | Contract; consistent with equals; TimSort stable | #029 §3 |
| 30 | Generic container | Every wildcard justified; a parallel-safe Collector | #030 §2 |
| 31 | Gap-fill | Fail-fast can silently skip (B1); bulk ops can half-complete | #031 B-1, B-3 |
| 32–36 | LRU project | HashMap + DLL; contract + differential tests; notify **after** mutation | #035 §4 |
| 37–38 | Big-O | Doubling ratios; hidden O(n) calls in loops; caches make O(1) non-flat | #038 Part B |

---

## 3. Self-Test (30 questions)

**Rules:** notes closed, 30 minutes, write the answers down, then check.

### A. Choosing & internals

1. Which collection for: (a) a FIFO queue, (b) the top-k largest of a stream, (c) "has this id been seen?", (d) prices by date with "price in effect on date d"?
2. ArrayList growth formula and its first five capacities?
3. Why must `HashMap` capacity be a power of two? Give two reasons.
4. When does a `HashMap` bucket become a tree?
5. `new HashMap<>(100)`: how many buckets, and does inserting 100 entries resize?
6. Why does `LinkedHashMap` iterate in O(n) while `HashMap` iterates in O(capacity + n)?
7. Which `LinkedHashMap` operations count as an access in access order? Name two that don't.
8. Height of a 1M-entry `TreeMap` built from ascending keys vs from a sorted map (measured)?

<details><summary>Answers A</summary>

1. (a) `ArrayDeque`; (b) a size-k min-`PriorityQueue`; (c) `HashSet`; (d) `TreeMap<LocalDate, Price>.floorEntry(d)`.
2. `new = old + (old >> 1)`; 10, 15, 22, 33, 49 (lazily allocated on the first add).
3. `(n−1) & hash` replaces modulo (fast, and safe for negative hashes); resize splits each bucket by a single bit (`hash & oldCap`) with no rehash.
4. When the bucket reaches 8 nodes **and** the table has ≥ 64 buckets (otherwise it resizes instead); it reverts at 6.
5. 128 buckets, threshold 96 → yes, it resizes once. `HashMap.newHashMap(100)` avoids that.
6. `LinkedHashMap` walks its doubly linked list of entries; `HashMap` scans every bucket, including empty ones.
7. Access: `get`, `getOrDefault`, `put`/`replace` on an existing key, `compute*`, `merge`. Not: `containsKey`, `containsValue`, iteration.
8. 37 (ascending inserts) vs 20 (bulk-built); random inserts gave 24.
</details>

### B. Iteration, views, correctness

9. Why does removing the second-to-last element in a for-each over an `ArrayList` not throw?
10. What does `map.keySet().forEach(k -> map.remove(k))` do?
11. `unmodifiableList` vs `List.copyOf`?
12. Why must hash keys be immutable (at least in the fields used by `equals`/`hashCode`)?
13. Three correct ways to remove elements while iterating a list?
14. `Arrays.asList(1, 2).add(3)`?

<details><summary>Answers B</summary>

9. `hasNext()` (which has no `modCount` check) returns false because `cursor == size` after the removal; `next()`, which does the check, never runs. The last element is silently skipped.
10. It performs the removals, **then** throws `ConcurrentModificationException` (`HashMap.KeySet.forEach` checks `modCount` after the loop).
11. A read-only **view** that sees later changes to the source vs an immutable **snapshot**.
12. A changed hash files the key in the wrong bucket, so it can't be found: a ghost entry.
13. `removeIf`; an explicit `Iterator` with `it.remove()`; an index loop running **backwards**.
14. `UnsupportedOperationException` (fixed-size; `set` works).
</details>

### C. Generics

15. Is `List<Integer>` a `List<Number>`? A `List<? extends Number>`?
16. State PECS and give the JDK's canonical example.
17. Why `<T extends Comparable<? super T>>` instead of `<T extends Comparable<T>>`?
18. Why `EvictionListener<? super K, ? super V>` in the LRU cache?
19. What's `CAP#1`, and how do you write through a `List<?>`?
20. Why is `new T[n]` illegal, and what do you do instead?
21. What's a bridge method?

<details><summary>Answers C</summary>

15. No (invariance); yes.
16. Producer `extends`, Consumer `super`: `Collections.copy(List<? super T> dest, List<? extends T> src)`.
17. So types that inherit `compareTo` from a supertype qualify (`Timestamp` is `Comparable<Date>`).
18. The listener **consumes** keys and values; an `EvictionListener<Object, Object>` should work for any cache.
19. The compiler's name for a captured wildcard type; delegate to a private generic helper `<E> void helper(List<E> l)`.
20. `T` is erased, and arrays need their element type at run time. Pass an `IntFunction<T[]>` (`String[]::new`), a `Supplier`, or a `Class<T>`.
21. A compiler-generated method that keeps overriding working after erasure: `compareTo(Object)` → checkcast → `compareTo(Money)`.
</details>

### D. Ordering

22. Comparable vs Comparator in one sentence each?
23. What does `comparing(A).thenComparing(B).reversed()` reverse?
24. Why is `(a, b) -> a - b` wrong?
25. What happens when a `TreeSet`'s comparator says two different objects are equal?
26. Which sort is stable: `Collections.sort` or `Arrays.sort(int[])`? Why does stability matter?

<details><summary>Answers D</summary>

22. Comparable: the class's single natural order, inside the class. Comparator: any number of external orders.
23. **Both** keys. Reverse just B with `thenComparing(B, reverseOrder())`.
24. Integer overflow can flip the sign; use `Integer.compare`.
25. It keeps only the first one: silent data loss. Add a unique tie-breaker.
26. `Collections.sort` (TimSort) is stable; `Arrays.sort(int[])` uses dual-pivot quicksort (stability is irrelevant for primitives). Stability lets multi-pass sorts keep earlier orderings.
</details>

### E. LRU project & Big-O

27. Why does an O(1) LRU need **both** a hash map and a doubly linked list?
28. What bug did the throwing-listener probe find, and what's the general fix?
29. What does a doubling ratio of ≈ 4 mean? ≈ 2.1?
30. `HashMap.get` is O(1), so why did 10 000 gets take 3.5× longer on an 800k map than on a 100k map?

<details><summary>Answers E</summary>

27. The map finds any node in O(1); the list keeps recency order and can unlink a known node and append in O(1). Neither alone does both.
28. The `LinkedHashMap` version ended with capacity + 1 entries (the listener ran inside `removeEldestEntry` before the removal); the others evicted without inserting. Fix: **complete the mutation, then notify**.
29. O(n²); O(n log n).
30. Cache misses: the larger map no longer fits in the CPU caches, so each get pays memory latency. The operation count is constant; the cost per operation isn't.
</details>

| Score | Action |
|------:|--------|
| 27–30 | Start Week 3 |
| 24–26 | Start Week 3; add every miss to Anki today |
| 18–23 | Redo #031's diagnostic and its drills for the weakest area first |
| < 18 | Redo #022 and #027 before moving on |

---

## 4. New Anki Cards — Rows 31–38 (32 cards)

Import exactly like #014/#028 (keep the header lines; separator = pipe).

```text
#separator:pipe
#html:false
#tags:java p1 week2 lru bigo
Removing the second-to-last element inside an ArrayList for-each: what happens?|No exception: hasNext() sees cursor == size and the loop ends; the last element is silently skipped
Why does hasNext() not throw ConcurrentModificationException?|Only next() checks modCount; hasNext() just compares cursor with size
map.keySet().forEach(k -> map.remove(k)) on a HashMap?|Removals happen, THEN ConcurrentModificationException at the end (modCount checked after the loop)
Correct one-liners to remove map entries by value?|map.entrySet().removeIf(e -> ...) or map.values().removeIf(v -> ...)
Why is a forward index loop with list.remove(i) wrong?|Elements shift left into i, then i++ skips them; iterate backwards instead
TreeMap with comparator comparingInt(String::length) fed cat, dog?|One entry: dog is "equal" to cat, merge adds to cat's count; add a tie-breaker
O(1) LRU cache data structures?|HashMap key -> node for lookup + doubly linked list for recency order (head = LRU, tail = MRU)
Why must an LRU list node store its key?|Eviction starts from the list head and must also remove the map entry, which needs the key
What do sentinel head/tail nodes buy?|Every real node always has prev and next, so unlink/link need no null checks or special cases
Which operations count as a "use" in our LRU contract?|get (hit) and put; containsKey, size and keysLruToMru only observe
Must updating an existing key in a full LRU cache evict?|No: nothing new is added, so nothing should leave
Is remove() an eviction?|No: the eviction listener reports capacity pressure only
LinkedHashMap LRU: composition or inheritance? Why?|Composition: extending exposes ~40 Map methods that bypass stats and the listener (EJ Item 18)
Why is a ReadWriteLock useless for an LRU cache?|get() reorders the list, so every read is a write
SynchronizedCache: why a private lock object?|Outside code can't lock the same monitor, which could cause deadlocks or starvation
Bug found in #035?|A throwing eviction listener left the LinkedHashMap LRU with capacity+1 entries and the others without the new entry
General fix for callback-during-mutation bugs?|Complete the mutation first, then notify; let the exception propagate with consistent state
Why couldn't the LinkedHashMap version notify inside removeEldestEntry?|LinkedHashMap has already inserted the new entry and removes the eldest only after the hook returns true
What is a reference model in testing?|A slow, obviously-correct implementation used as the oracle for differential tests
What is differential testing?|Run the same random operations on the reference and the real implementation; compare all observable results each step
Why put contract tests in an abstract test class?|Every implementation runs the same spec via a tiny subclass that supplies a factory
Measured LRU cost at 100k entries (LinkedHashMap vs O(n) list)?|About 16 ns/op vs 28 us/op (~1,800x)
Measured uncontended synchronized overhead on the LRU?|About 3 ns/op (~15%)
LRU hit rate with 1% capacity: uniform vs Zipf(1.0) traffic over 100k keys?|Uniform ~1%; Zipf ~51%. Key skew decides cache value
Big-O vs Theta vs Omega?|Upper bound / tight bound / lower bound on growth
Big-O simplification rules?|Drop constants and lower terms; sequential adds; nested multiplies; separate variables for separate inputs
Merge sort recurrence and solution?|T(n) = 2T(n/2) + O(n) -> O(n log n)
Naive recursive Fibonacci complexity, and the fix?|O(2^n); memoize or iterate for O(n)
Doubling experiment ratios?|~1 constant/log; ~2 linear; ~2.1-2.3 n log n; ~4 quadratic; ~8 cubic
Why is String += in a loop O(n^2)?|Each += copies the whole string built so far; use StringBuilder (measured ~320x faster at 40k)
list.removeAll(otherList) complexity, and the fix?|O(n*m) because otherList.contains is O(m); pass a HashSet (measured 200x faster at 40k)
Why can O(1) HashMap.get slow down as the map grows?|The working set outgrows the CPU caches, so each get pays memory latency (cache misses)
```

---

## 5. Gap-Fix Protocol & Schedule

Use #014 §5 (write it, correct it, run code, card it, log it) for every miss.

| When | Re-do |
|------|-------|
| +1 day | Missed self-test questions |
| +3 days | #031 diagnostic (all 12) without notes |
| +7 days | Rebuild `ManualLruCache` from memory in 30 minutes, then run #035's contract and differential tests against it |
| +14 days | Explain the §1 map aloud in 5 minutes |
| +30 days | Implement a simplified `HashMap` from scratch (#024) |

---

## 6. Readiness Checklist for Week 3 — Concurrency & Multithreading I (rows 42–66)

- [ ] Self-test ≥ 24/30
- [ ] New cards imported; #028's deck reviewed
- [ ] Can explain why `HashMap` isn't thread-safe and what goes wrong (#020 §3.11)
- [ ] Can explain what `SynchronizedCache` guarantees, and its limits (#035 §3, #036 §5)
- [ ] Revisit the JMM notes (#012–#013): happens-before is the foundation of Week 3
- [ ] Rows 17–39 marked `TRUE` in `Roadmap.csv` as you complete them

---

## Handwritten Cheat Sheet

```
WEEK 2 SPINE: choose by op cost → know internals → type with generics → order correctly → prove it
STRUCTURES: ArrayList/ArrayDeque/HashMap/HashSet default · Linked* order · Tree* sorted · PQ priority
INTERNALS: ×1.5 · 2^k + h^(h>>>16) + (n-1)&h · lo/hi split · treeify 8 & 64 · LHM before/after
ITERATION TRAPS: 2nd-to-last remove skips silently · keySet.forEach remove = work + CME · iterate back
GENERICS: invariance · PECS · ? super for comparators/listeners · capture helper · erasure/bridges
ORDER: Comparable 1 vs Comparator many · .reversed() whole chain · tie-break unique key · Collator
LRU: HashMap + DLL + sentinels · get = write · notify AFTER mutation · contract + reference + differential
BIG-O: doubling ratios 2/2.1/4 · hidden O(n) in loops · caches make O(1) not flat
```

---

**Next note:** #040 — Research paper: *The Google File System* (Ghemawat, Gobioff, Leung, 2003)
