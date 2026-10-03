# #037 — Study: Big-O Complexity Review + Collections Cheat Sheet

**Date:** 2026-10-03  
**Roadmap row:** 37 — P1 Java Core · Collections & Generics · *Design/Theory (study)*  
**Practice companion:** #038 (exercises + a measured doubling experiment)  
**Consolidates:** #017, #018, #020, #021, #023, #024, #035

---

## At a Glance (copy to handwritten notes)

- **Big-O = an upper bound on growth** as n → ∞, ignoring constants and lower-order terms. **Θ** = a tight bound, **Ω** = a lower bound. In conversation, "O" usually means Θ of the worst case.
- **Rules:** drop constants (`3n + 5` → O(n)); keep the dominant term (`n² + n log n` → O(n²)); sequential steps **add**; nested loops **multiply**; different inputs get **different variables** (O(n + m), not O(n)).
- **Ladder:** O(1) < O(log n) < O(√n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!).
- **Amortized** = the average over a sequence of operations (`ArrayList.add`, `HashMap.put`, the monotonic deque). It's *not* the average over random inputs.
- **Best / average / worst case** are separate questions from Big-O. `HashMap.get` is O(1) average and O(log n) worst (treeified bins).
- **Rough budget:** a simple Java loop does ~10⁸–10⁹ basic operations per second, so n = 10⁵ → O(n²) = 10¹⁰ is **too slow**, while O(n log n) ≈ 1.7 × 10⁶ is instant.
- **Big-O ignores the memory hierarchy.** Equal Big-O can differ 5× (ArrayList vs LinkedList iteration, #018), and an O(1) lookup slows down as data outgrows the CPU caches (#038).
- **Hidden O(n) inside a loop → O(n²):** `list.contains`, `list.remove(Object)`, `String +=`, `LinkedList.get(i)`, `removeAll(List)`. All are measured in #038.

---

## 1. The Three Notations

| Notation | Meaning | Analogy |
|----------|---------|---------|
| **O(g(n))** | f grows **at most** as fast as g (an upper bound): f(n) ≤ c·g(n) for all n ≥ n₀ | "≤" |
| **Ω(g(n))** | f grows **at least** as fast as g (a lower bound) | "≥" |
| **Θ(g(n))** | both: f grows **exactly** like g (a tight bound) | "=" |

- Merge sort is Θ(n log n). Saying it's "O(n²)" is technically true but useless.
- **Comparison sorting is Ω(n log n)**: no comparison-based sort can beat it in the worst case. A decision tree with n! leaves has height ≥ log₂(n!) ≈ n log₂ n. Non-comparison sorts (counting sort, radix sort) escape this by exploiting the structure of the keys.
- "Best case O(1)" means nothing useful: almost every algorithm has a trivial best case.

### Formally

f(n) = O(g(n)) ⇔ ∃ c > 0, n₀ such that 0 ≤ f(n) ≤ c·g(n) for all n ≥ n₀.

Example: 3n² + 10n + 7 ≤ 3n² + 10n² + 7n² = 20n² for n ≥ 1, so c = 20, n₀ = 1, and it's O(n²).

---

## 2. Simplification Rules

| Rule | Example |
|------|---------|
| Drop constant factors | `5n` → O(n); `n/2` → O(n) |
| Drop lower-order terms | `n² + 1000n + 10⁶` → O(n²) |
| Sequential code **adds** | loop over n, then loop over n → O(n + n) = O(n) |
| Nested code **multiplies** | for i < n { for j < n } → O(n²); for i < n { for j < i } → n(n−1)/2 → O(n²) |
| Different inputs → **different variables** | for each of n orders, scan m products → O(n·m), **not** O(n²) |
| Logarithm bases don't matter | log₂ n = log₁₀ n / log₁₀ 2, a constant factor → just "log n" |
| Halving or doubling each step → log | `while (n > 1) n /= 2;` → O(log n) |
| A function called k times inside a loop | multiply by its cost: n × `list.contains` (O(n)) = O(n²) |

---

## 3. The Common Classes

| Class | Name | Typical example | n = 10⁶ → operations | Feasible n in ~1 s |
|-------|------|-----------------|---------------------:|-------------------:|
| O(1) | constant | `HashMap.get`, `array[i]`, `ArrayDeque.pollFirst` | 1 | any |
| O(log n) | logarithmic | binary search, `TreeMap.get`, heap `offer`/`poll` | 20 | any |
| O(√n) | square root | trial-division primality | 1 000 | ~10¹⁶ |
| O(n) | linear | one scan, `list.contains`, `HashMap` build | 10⁶ | ~10⁸ |
| O(n log n) | linearithmic | merge sort, TimSort, building a `TreeMap` | 2 × 10⁷ | ~10⁷ |
| O(n²) | quadratic | nested loops over all pairs, insertion sort | 10¹² | ~10⁴ |
| O(n³) | cubic | naive matrix multiply, Floyd–Warshall | 10¹⁸ | ~500 |
| O(2ⁿ) | exponential | all subsets, naive recursive Fibonacci | — | ~25–30 |
| O(n!) | factorial | all permutations, brute-force TSP | — | ~10–11 |

"Feasible n" assumes ~10⁸ simple operations per second: an order-of-magnitude rule for sizing an approach, not a benchmark.

---

## 4. Amortized Analysis

**Amortized cost** = the total cost of a *sequence* of operations ÷ the number of operations, in the **worst case over sequences**. It isn't about probability.

| Structure | Single operation, worst case | Amortized | Why |
|-----------|-----------------------------|-----------|-----|
| `ArrayList.add` | O(n) (grow + copy) | **O(1)** | ×1.5 growth: total copies ≤ ~3n over n adds (#021 Q3) |
| `HashMap.put` | O(n) (resize) | **O(1)** average | Doubling: total rehash work ≤ ~2n |
| `ArrayDeque.addLast` | O(n) (grow) | **O(1)** | Doubling growth |
| Monotonic deque (sliding window max) | One step can pop many | **O(1)** per element | Each element is pushed and popped at most once → ≤ 2n operations (#022) |
| `StringBuilder.append` | O(n) (grow) | **O(1)** per char | Doubling growth |

The **latency** of the expensive single operation still matters in practice: a resize is a spike. That's why you presize (`HashMap.newHashMap(n)`, `new ArrayList<>(n)`) when you know the size (#020 §3.10, #035 presizing).

---

## 5. Recurrences: Analysing Recursion

| Recurrence | Example | Solution |
|------------|---------|----------|
| T(n) = T(n/2) + O(1) | binary search | **O(log n)** |
| T(n) = T(n−1) + O(1) | linear recursion (factorial) | O(n) |
| T(n) = T(n−1) + O(n) | selection sort, recursive | O(n²) |
| T(n) = 2T(n/2) + O(n) | merge sort | **O(n log n)** |
| T(n) = 2T(n/2) + O(1) | tree traversal, divide & combine in O(1) | O(n) |
| T(n) = 2T(n−1) + O(1) | naive Fibonacci, towers of Hanoi | **O(2ⁿ)** |

**Master theorem (simplified).** For T(n) = a·T(n/b) + O(n^d):

| Case | Result |
|------|--------|
| d > log_b a | O(n^d) (the combining work dominates) |
| d = log_b a | O(n^d log n) (balanced: merge sort, where a = b = 2 and d = 1) |
| d < log_b a | O(n^(log_b a)) (the recursion dominates) |

**The recursion-tree intuition for merge sort:** log₂ n levels × O(n) work per level = O(n log n).

**Memoisation** can collapse exponential to polynomial: naive Fibonacci O(2ⁿ) → O(n) with a `HashMap<Integer, Long>` cache, because each subproblem is solved once.

---

## 6. Space Complexity

Count the **extra** memory an algorithm allocates, as a function of n:

| Algorithm | Extra space |
|-----------|-------------|
| Iterative sum | O(1) |
| Recursive sum (depth n) | **O(n) stack frames**, and the risk of `StackOverflowError` at large n (#004) |
| Merge sort (array) | O(n) auxiliary |
| TimSort | O(n) worst (temp arrays for merging) |
| Quicksort (in place) | O(log n) stack on average |
| Dedupe with a `HashSet` | O(n) |
| `HashMap` with n entries | O(n), but ~40–50 bytes per entry in practice (node + table slot + boxed key/value) |

**Time–space trade-off:** the #031 dedupe went from O(n²) time / O(1) extra space to O(n) time / O(n) space, and from 776 ms to 7 ms.

---

## 7. Big-O vs Reality (what the measurements in this repo showed)

| Observation | Measured where | Lesson |
|-------------|----------------|--------|
| ArrayList vs LinkedList iteration: both O(n), **5×** apart | #018 | Cache locality is a constant factor Big-O hides |
| LinkedList middle insert by index: O(n) vs ArrayList's O(n), **19×** apart | #018 | Pointer chasing vs `memmove` |
| The O(n) reference LRU grew **213×** for 100× more data | #035 | Super-linear growth once data outgrows the CPU caches |
| `HashMap.get` (O(1)) slowed **3.5×** as the map grew 8× (100k → 800k entries) | #038 | Random memory access → cache misses. Asymptotically O(1), but not free |
| Comparator-correctness bugs break sorts entirely | #027 | Correctness first, Big-O second |

**Practical rule:** use Big-O to rule out approaches (n = 10⁵ and an O(n²) plan → stop). Use **measurement** (JMH, doubling experiments) to choose between approaches in the same Big-O class.

---

## 8. Java Collections Complexity Cheat Sheet (consolidated)

### Lists, queues, deques

| Operation | `ArrayList` | `LinkedList` | `ArrayDeque` | `PriorityQueue` | `CopyOnWriteArrayList` |
|-----------|:---:|:---:|:---:|:---:|:---:|
| `get(i)` / `set(i)` | **O(1)** | O(n) | — | — | O(1) / O(n) |
| `add(e)` (end) | **O(1)\*** | O(1) | O(1)\* | O(log n) | O(n) |
| `add(0, e)` / `addFirst` | O(n) | O(1) | **O(1)\*** | — | O(n) |
| `remove(0)` / `pollFirst` | O(n) | O(1) | **O(1)** | O(log n) (`poll`) | O(n) |
| `add(i, e)` / `remove(i)` | O(n) | O(n) (walk) | — | — | O(n) |
| `contains` / `indexOf` / `remove(Object)` | O(n) | O(n) | O(n) | O(n) | O(n) |
| `peek` (min/head) | — | O(1) | O(1) | **O(1)** | — |
| Iterate all | O(n), cache-friendly | O(n), pointer chasing | O(n) | O(n), heap order | O(n), snapshot |

### Sets & maps

| Operation | `HashMap`/`HashSet` | `LinkedHashMap`/`Set` | `TreeMap`/`TreeSet` | `EnumMap`/`EnumSet` | `ConcurrentHashMap` |
|-----------|:---:|:---:|:---:|:---:|:---:|
| get / contains / put / remove | **O(1)** avg, O(log n) worst | O(1) | **O(log n)** | O(1) | O(1) |
| min / max / floor / ceiling | O(n) | O(n) (first/last O(1)) | **O(log n)** | — | O(n) |
| Iterate all | O(capacity + n) | **O(n)** | O(n), sorted | O(universe) | O(capacity + n) |
| Ordered iteration | ✗ | insertion/access | sorted | declaration | ✗ |

### Algorithms (`java.util`)

| Method | Complexity |
|--------|-----------|
| `Collections.sort` / `List.sort` / `Arrays.sort(Object[])` | O(n log n), stable (TimSort); O(n) if already sorted |
| `Arrays.sort(int[])` | O(n log n) average (dual-pivot quicksort) |
| `Collections.binarySearch` / `Arrays.binarySearch` | O(log n) on a `RandomAccess` list or an array |
| `Collections.reverse` / `shuffle` / `max` / `min` / `frequency` | O(n) |
| `Collection.removeAll(c)` / `retainAll(c)` | O(n × cost of `c.contains`): **O(n·m) if c is a List**, O(n) if c is a HashSet (#038) |
| `List.removeIf` (ArrayList) | O(n), single pass |
| `new TreeMap<>(sortedMap)` | **O(n)** (bulk build) vs O(n log n) by repeated `put` |
| `String +=` in a loop | **O(n²)** total; use `StringBuilder` (#038) |
| `stream().sorted()` | O(n log n) |
| `stream().distinct()` | O(n) average (hash-based) |

\* amortized

### Space per element (measured or derived in #018, #020)

| Structure | Structure overhead per element (compressed oops) |
|-----------|----------------------------------------|
| `ArrayList` | ~4 B (reference slot) + growth slack |
| `ArrayDeque` | ~4 B + slack |
| `LinkedList` | **24 B** node |
| `HashMap` | ~32 B `Node` + ~4–8 B table slot |
| `LinkedHashMap` | ~40 B entry (+ `before`/`after`) + table slot |
| `TreeMap` | ~40 B entry (key, value, left, right, parent, colour) |
| Boxed `Integer` (if not cached) | 16 B, *on top of* the above |

---

## 9. How to Analyse Code Quickly (checklist)

1. **Name the inputs:** n, m, k… One variable per independent size.
2. **Loops:** count the iterations of each, and multiply nested ones.
3. **Calls inside loops:** look up the cost of every library call (this cheat sheet). `contains`, `remove(Object)`, `indexOf`, `get(i)` on a `LinkedList`, `+=` on a `String` are **O(n) traps**.
4. **Recursion:** write the recurrence; use the tree or the master theorem.
5. **Amortized?** Growth by a factor, or "each element enters/leaves once"?
6. **Space:** extra collections, recursion depth.
7. **Sanity check:** plug in the real n. Is the operation count < ~10⁸ for the time budget?
8. **Measure** if two options tie on Big-O.

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| O vs Θ vs Ω? | Upper / tight / lower bound on growth |
| Is O(2n) different from O(n)? | No: constants drop |
| O(n + m) or O(n)? | O(n + m) when they're independent sizes |
| What is amortized O(1)? | The average per operation over any sequence (e.g. `ArrayList.add`), even though one operation can be O(n) |
| Why is `HashMap.get` "O(1)" but slower for huge maps? | Cache misses: the asymptotic count is O(1), but memory latency grows with the working set |
| Lower bound for comparison sorting? | Ω(n log n) |
| Merge sort recurrence? | T(n) = 2T(n/2) + O(n) → O(n log n) |
| Naive Fibonacci? | O(2ⁿ); memoised O(n) |
| What's wrong with `for (…) result += str`? | Each `+=` copies the whole string → O(n²) total |
| `list.removeAll(otherList)` complexity? | O(n·m), so pass a `HashSet` instead |

---

## Handwritten Cheat Sheet

```
O ≤ upper · Ω ≥ lower · Θ = tight
RULES: drop constants · keep dominant · seq ADD · nested MULTIPLY · n,m separate · log base irrelevant
LADDER: 1 < log n < √n < n < n log n < n² < n³ < 2ⁿ < n!
BUDGET ~1e8 ops/s → n=1e5: n² too slow, n log n fine
AMORTIZED: ArrayList.add, HashMap.put, ArrayDeque, monotonic deque, StringBuilder
RECURRENCES: T(n/2)+1 → log n · 2T(n/2)+n → n log n · 2T(n-1)+1 → 2ⁿ · T(n-1)+n → n²
MASTER a·T(n/b)+n^d: d > log_b a → n^d · = → n^d log n · < → n^(log_b a)
SORT LOWER BOUND Ω(n log n) (comparison)
SPACE: recursion depth counts! (StackOverflowError)
O(n) TRAPS IN LOOPS: list.contains · remove(Object) · indexOf · LinkedList.get(i)
                     String += · removeAll(List) → use HashSet / StringBuilder
REALITY: same Big-O can differ 5–20× (cache) · O(1) slows when data > CPU cache
```

---

**Next note:** #038 — Practice/apply: Big-O review (exercises + measured doubling experiment)
