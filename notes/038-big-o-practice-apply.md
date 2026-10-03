# #038 — Practice/Apply: Big-O Complexity Review + Collections Cheat Sheet

**Date:** 2026-10-03  
**Roadmap row:** 38 — P1 Java Core · Collections & Generics · *Design/Theory (practice)*  
**Theory:** #037

> Part A has 12 analysis exercises (answers hidden in collapsible blocks). Part B is a **doubling experiment** that was actually run on JDK 21 / Apple M4, so the growth rates below are measured, not claimed.

---

## At a Glance (copy to handwritten notes)

- **Analyse:** name the inputs → count the loops → price every library call inside them → recurrences → amortized → space.
- **Verify:** a **doubling experiment** times n, 2n, 4n, 8n. Ratio ≈ **1** → O(1)/O(log n), **2** → O(n), **~2.1–2.3** → O(n log n), **4** → O(n²), **8** → O(n³).
- ✔︎ Measured: `list.contains` dedupe, `String +=`, `removeAll(List)` and the `LinkedList` index loop **all double ×4**, so they're quadratic. Their fixes (`LinkedHashSet`, `StringBuilder`, `removeAll(HashSet)`) double ×2.
- ✔︎ Measured: `HashMap.get` (O(1)) still slowed **3.5×** while the map grew 8×, and binary search 2.1× (log₂ n only grew 15%). Both are the **cache-miss** effect: Big-O counts operations, hardware charges for memory access.
- Small inputs (sub-millisecond) give noisy ratios (×0.7…×1.5). Trust the **large-n** ratios.

---

## Part A — Analyse These (no running)

For each: time complexity, extra space, and (if it's slow) a faster version.

```java
// A1
static boolean hasDuplicate(int[] a) {
    for (int i = 0; i < a.length; i++)
        for (int j = i + 1; j < a.length; j++)
            if (a[i] == a[j]) return true;
    return false;
}

// A2
static int countCommon(List<String> a, List<String> b) {   // sizes n and m
    int c = 0;
    for (String s : a) if (b.contains(s)) c++;
    return c;
}

// A3
static int log2Floor(int n) {
    int k = 0;
    while (n > 1) { n /= 2; k++; }
    return k;
}

// A4
static long sumPairs(int n) {
    long s = 0;
    for (int i = 0; i < n; i++)
        for (int j = 0; j < 100; j++)
            s += i * j;
    return s;
}

// A5
static String join(List<String> words) {
    String out = "";
    for (String w : words) out += w + ",";
    return out;
}

// A6
static int fib(int n) { return n < 2 ? n : fib(n - 1) + fib(n - 2); }

// A7
static void printTriangle(int n) {
    for (int i = 0; i < n; i++)
        for (int j = 0; j < i; j++)
            System.out.print("*");
}

// A8
static List<Integer> topK(List<Integer> xs, int k) {         // n = xs.size()
    List<Integer> copy = new ArrayList<>(xs);
    Collections.sort(copy, Collections.reverseOrder());
    return copy.subList(0, k);
}

// A9
static int sumLinked(LinkedList<Integer> list) {
    int s = 0;
    for (int i = 0; i < list.size(); i++) s += list.get(i);
    return s;
}

// A10
static boolean binarySearchRecursive(int[] a, int key, int lo, int hi) {
    if (lo > hi) return false;
    int mid = (lo + hi) >>> 1;
    if (a[mid] == key) return true;
    return a[mid] < key ? binarySearchRecursive(a, key, mid + 1, hi)
                        : binarySearchRecursive(a, key, lo, mid - 1);
}

// A11
static Map<String, List<String>> groupAnagrams(List<String> words) {   // n words, max length L
    Map<String, List<String>> groups = new HashMap<>();
    for (String w : words) {
        char[] cs = w.toCharArray();
        Arrays.sort(cs);
        groups.computeIfAbsent(new String(cs), k -> new ArrayList<>()).add(w);
    }
    return groups;
}

// A12
static void mergeSort(int[] a, int lo, int hi, int[] tmp) {
    if (hi - lo < 2) return;
    int mid = (lo + hi) >>> 1;
    mergeSort(a, lo, mid, tmp);
    mergeSort(a, mid, hi, tmp);
    merge(a, lo, mid, hi, tmp);          // O(hi - lo)
}
```

<details><summary>Answers</summary>

| # | Time | Extra space | Notes / faster version |
|---|------|-------------|------------------------|
| A1 | **O(n²)** worst (n(n−1)/2 comparisons) | O(1) | `HashSet<Integer>`: `if (!seen.add(x)) return true;` → **O(n)** time, O(n) space. Or sort first: O(n log n), O(1) extra with an in-place sort |
| A2 | **O(n·m)**: `List.contains` is O(m) | O(1) | `Set<String> bs = new HashSet<>(b);` then `bs.contains` → **O(n + m)** |
| A3 | **O(log n)** (halving) | O(1) | `31 - Integer.numberOfLeadingZeros(n)` → O(1) |
| A4 | **O(n)**: the inner loop is a constant 100 → 100n | O(1) | A constant inner bound isn't "n²" |
| A5 | **O(n·L·n) = O(n²·L)**: each `+=` copies the whole string built so far | O(n·L) per copy (garbage) | `String.join(",", words)` or a `StringBuilder` → **O(total length)** (measured in Part B) |
| A6 | **O(2ⁿ)** (≈ φⁿ, φ = 1.618) | O(n) stack depth | Memoise (O(n)) or iterate with two variables (O(n) time, O(1) space) |
| A7 | **O(n²)**: Σ i = n(n−1)/2 | O(1) | Inherent: the output itself has n²/2 characters |
| A8 | **O(n log n)** (the sort dominates) | O(n) copy | A size-k min-heap → **O(n log k)** (#022 Ex. 1); `subList` is an O(1) view |
| A9 | **O(n²)**: `LinkedList.get(i)` walks O(min(i, n−i)) | O(1) | for-each → O(n) (measured in Part B: ×4 per doubling) |
| A10 | **O(log n)**: T(n) = T(n/2) + O(1) | **O(log n) stack** (the recursive version) | The iterative version has O(1) extra space |
| A11 | **O(n · L log L)**: sort each word; the hash/`computeIfAbsent` is O(L) | O(n·L) | Counting signatures (26 counts) → O(n·L) |
| A12 | **O(n log n)**: T(n) = 2T(n/2) + O(n) | O(n) for `tmp` + O(log n) stack | Master theorem case 2 (a = b = 2, d = 1) |

</details>

---

## Part B — Measure It: The Doubling Experiment

### Method

Run each algorithm at n, 2n, 4n and 8n, take the best of 5 runs (after warm-up), and look at the ratios T(2n) / T(n):

| Ratio | Growth |
|------:|--------|
| ≈ 1 | O(1) or O(log n) |
| ≈ 2 | O(n) |
| ≈ 2.1–2.3 | O(n log n) |
| ≈ 4 | O(n²) |
| ≈ 8 | O(n³) |

Algorithms with expensive **setup** (building a map just to look things up in it) time **only the measured section**. The first version of this program didn't, so the HashMap and binary-search rows came out ×2: they were measuring the O(n) build, not the lookups. Separating setup from the measured section is the same discipline as JMH's `@Setup` (#018).

### Code

```java
import java.util.*;
import java.util.function.IntFunction;

/**
 * Doubling experiment: time an algorithm at n, 2n, 4n, 8n. If T(2n)/T(n) ≈ 2^k, the algorithm is
 * ~O(n^k): ratio 1 → O(1)/O(log n), 2 → O(n), ~2.1 → O(n log n), 4 → O(n²).
 * Rough (nanoTime, best of 5 after warm-up), but good enough to see growth rates.
 */
public class Doubling {
    interface Algo { long run(int n); }

    static long best(Algo a, int n) {
        long best = Long.MAX_VALUE, sink = 0;
        for (int i = 0; i < 5; i++) {
            long t = System.nanoTime();
            sink += a.run(n);
            best = Math.min(best, System.nanoTime() - t);
        }
        if (sink == 42) System.out.print("");
        return best;
    }

    static void experiment(String name, int n0, Algo a) {
        for (int i = 0; i < 3; i++) a.run(n0);                 // warm-up
        StringBuilder sb = new StringBuilder(String.format("%-46s", name));
        long prev = -1;
        for (int n = n0; n <= n0 * 8; n *= 2) {
            long t = best(a, n);
            sb.append(String.format(" n=%,9d %9.2f ms", n, t / 1e6));
            if (prev > 0) sb.append(String.format(" (x%.1f)", (double) t / prev));
            prev = t;
        }
        System.out.println(sb);
    }

    /** For algorithms with expensive setup: run() returns the nanos of ONLY the measured section. */
    static void experimentInner(String name, int n0, Algo timedSection) {
        for (int i = 0; i < 3; i++) timedSection.run(n0);
        StringBuilder sb = new StringBuilder(String.format("%-46s", name));
        long prev = -1;
        for (int n = n0; n <= n0 * 8; n *= 2) {
            long t = Long.MAX_VALUE;
            for (int i = 0; i < 5; i++) t = Math.min(t, timedSection.run(n));
            sb.append(String.format(" n=%,9d %9.3f ms", n, t / 1e6));
            if (prev > 0) sb.append(String.format(" (x%.1f)", (double) t / prev));
            prev = t;
        }
        System.out.println(sb);
    }

    static long sink;

    static List<Integer> randomList(int n, int bound) {
        Random r = new Random(n);
        List<Integer> l = new ArrayList<>(n);
        for (int i = 0; i < n; i++) l.add(r.nextInt(bound));
        return l;
    }

    public static void main(String[] args) {
        experimentInner("HashMap.get x10000 (map of size n)", 100_000, n -> {
            Map<Integer, Integer> m = HashMap.newHashMap(n); for (int i = 0; i < n; i++) m.put(i, i);
            long t = System.nanoTime(); long s = 0; for (int i = 0; i < 10_000; i++) s += m.get((int) ((long) i * 7919 % n));
            long elapsed = System.nanoTime() - t; sink += s; return elapsed;
        });
        experiment("sum of ArrayList (for-each)", 1_000_000, n -> {
            List<Integer> l = randomList(n, 100); long s = 0; for (int x : l) s += x; return s;
        });
        experiment("Collections.sort(random ints)", 250_000, n -> {
            List<Integer> l = randomList(n, Integer.MAX_VALUE); Collections.sort(l); return l.get(0);
        });
        experiment("dedupe via list.contains", 5_000, n -> {
            List<Integer> in = randomList(n, n), out = new ArrayList<>(); for (Integer x : in) if (!out.contains(x)) out.add(x); return out.size();
        });
        experiment("dedupe via LinkedHashSet", 5_000, n -> new LinkedHashSet<>(randomList(n, n)).size());
        experiment("String += in a loop", 5_000, n -> {
            String s = ""; for (int i = 0; i < n; i++) s += 'x'; return s.length();
        });
        experiment("StringBuilder.append in a loop", 5_000, n -> {
            StringBuilder sb = new StringBuilder(); for (int i = 0; i < n; i++) sb.append('x'); return sb.length();
        });
        experiment("list.removeAll(otherList)", 5_000, n -> {
            List<Integer> a = randomList(n, n * 2); List<Integer> b = randomList(n, n * 2); a.removeAll(b); return a.size();
        });
        experiment("list.removeAll(new HashSet<>(otherList))", 5_000, n -> {
            List<Integer> a = randomList(n, n * 2); List<Integer> b = randomList(n, n * 2); a.removeAll(new HashSet<>(b)); return a.size();
        });
        experiment("LinkedList index loop get(i)", 2_000, n -> {
            LinkedList<Integer> l = new LinkedList<>(randomList(n, 100)); long s = 0; for (int i = 0; i < l.size(); i++) s += l.get(i); return s;
        });
        experimentInner("binarySearch x10000 (sorted int[] of n)", 1_000_000, n -> {
            int[] a = new int[n]; for (int i = 0; i < n; i++) a[i] = 2 * i;
            long t = System.nanoTime(); long s = 0; for (int i = 0; i < 10_000; i++) s += Arrays.binarySearch(a, (int) ((long) i * 7919 % (2L * n)));
            long elapsed = System.nanoTime() - t; sink += s; return elapsed;
        });
    }
}
```

Run it with `java -Xmx2g Doubling.java` (single-file source launch, #002).

### Results ✔︎ (Apple M4, JDK 21.0.10)

```text
HashMap.get x10000 (map of size n)             n=  100,000     0.075 ms n=  200,000     0.117 ms (x1.6) n=  400,000     0.206 ms (x1.8) n=  800,000     0.261 ms (x1.3)
sum of ArrayList (for-each)                    n=1,000,000      5.40 ms n=2,000,000     10.82 ms (x2.0) n=4,000,000     26.39 ms (x2.4) n=8,000,000     41.99 ms (x1.6)
Collections.sort(random ints)                  n=  250,000     29.04 ms n=  500,000     59.55 ms (x2.1) n=1,000,000    126.24 ms (x2.1) n=2,000,000    285.13 ms (x2.3)
dedupe via list.contains                       n=    5,000      2.92 ms n=   10,000      9.12 ms (x3.1) n=   20,000     38.19 ms (x4.2) n=   40,000    172.92 ms (x4.5)
dedupe via LinkedHashSet                       n=    5,000      0.37 ms n=   10,000      0.25 ms (x0.7) n=   20,000      0.37 ms (x1.4) n=   40,000      0.73 ms (x2.0)
String += in a loop                            n=    5,000      0.41 ms n=   10,000      1.67 ms (x4.1) n=   20,000      7.77 ms (x4.7) n=   40,000     35.07 ms (x4.5)
StringBuilder.append in a loop                 n=    5,000      0.05 ms n=   10,000      0.04 ms (x0.7) n=   20,000      0.05 ms (x1.5) n=   40,000      0.11 ms (x2.0)
list.removeAll(otherList)                      n=    5,000      4.16 ms n=   10,000     16.29 ms (x3.9) n=   20,000     64.82 ms (x4.0) n=   40,000    257.90 ms (x4.0)
list.removeAll(new HashSet<>(otherList))       n=    5,000      0.22 ms n=   10,000      0.43 ms (x2.0) n=   20,000      0.87 ms (x2.0) n=   40,000      1.29 ms (x1.5)
LinkedList index loop get(i)                   n=    2,000      1.31 ms n=    4,000      5.37 ms (x4.1) n=    8,000     21.64 ms (x4.0) n=   16,000     86.41 ms (x4.0)
binarySearch x10000 (sorted int[] of n)        n=1,000,000     0.871 ms n=2,000,000     1.091 ms (x1.3) n=4,000,000     1.508 ms (x1.4) n=8,000,000     1.831 ms (x1.2)
```

### Interpretation

| Algorithm | Predicted | Measured ratio (large n) | Verdict |
|-----------|-----------|--------------------------|---------|
| `Collections.sort` | O(n log n) | ×2.1, ×2.1, ×2.3 | ✅ Exactly the n log n signature: slightly above 2 |
| Sum of ArrayList | O(n) | ×2.0, ×2.4, ×1.6 (≈ 2 on average) | ✅ Linear (noisy, because it includes building the list) |
| `list.contains` dedupe | O(n²) | ×4.2, ×4.5 | ✅ Quadratic |
| `LinkedHashSet` dedupe | O(n) | ×2.0 at the largest step | ✅ Linear (small sizes are noise) |
| `String +=` | O(n²) | ×4.1, ×4.7, ×4.5 | ✅ Quadratic: every `+=` copies the whole string |
| `StringBuilder` | O(n) | ×2.0 at the largest step | ✅ Linear; **~320× faster at n = 40 000** |
| `removeAll(List)` | O(n·m) = O(n²) here | ×3.9, ×4.0, ×4.0 | ✅ Textbook quadratic |
| `removeAll(HashSet)` | O(n + m) | ×2.0, ×2.0 | ✅ Linear; **200× faster at n = 40 000** |
| `LinkedList` index loop | O(n²) | ×4.1, ×4.0, ×4.0 | ✅ Quadratic: the #018 lesson again |
| `HashMap.get` × 10 000 | O(1) per get | ×1.6, ×1.8, ×1.3 (**3.5× total over 8× size**) | ⚠️ Not flat: see below |
| `binarySearch` × 10 000 | O(log n) | ×1.3, ×1.4, ×1.2 (**2.1× total**) | ⚠️ log₂ n only grew 20 → 23 (+15%), but the time doubled |

### Why do the O(1) and O(log n) rows grow?

**Cache misses.** Each entry of this `HashMap<Integer, Integer>` costs roughly 70 bytes (a ~32 B node + two 16 B boxed `Integer`s + its table slot). At 100 000 entries (~7 MB) much of it can stay in the CPU's caches. At 800 000 entries (~55 MB) it can't, so each `get` touches random memory: a likely cache miss at ~100 ns, vs ~1–5 ns for a cache hit. The **operation count** per `get` is still constant. The **cost per operation** went up because the data no longer fits near the CPU.

Binary search on 8M `int`s (32 MB) is the same story: the first few probes (the middle elements) stay cached, but the last several probes per search land on cold cache lines.

> **Big-O tells you how the work grows. The memory hierarchy decides how much each unit of work costs.** For data structures larger than the CPU cache, expect "O(1)" to mean "a constant number of cache misses", and design for **locality** (arrays, contiguous layouts) when it matters (#018, #035).

---

## Part C — Choose the Collection: Applied Drills

For each scenario, name the collection, give the Big-O of the critical operation, and explain why.

1. Check whether a username is taken, among 10M users, 10 000 times per second.
2. Show the 20 most recent log lines from an unbounded stream.
3. Find all orders placed between two timestamps, repeatedly.
4. Process jobs by priority; new jobs arrive constantly.
5. Count word frequencies in a 1 GB text file, then print them alphabetically.
6. Undo/redo in an editor.
7. Keep the 1 000 most recently viewed products for a "recently viewed" widget.
8. Find whether any two numbers in an array sum to a target.

<details><summary>Answers</summary>

| # | Choice | Critical operation | Why |
|---|--------|--------------------|-----|
| 1 | `HashSet<String>` (or a `HashMap` of users) | `contains` **O(1)** | Membership on a huge set; sorting is irrelevant |
| 2 | `ArrayDeque<String>` capped at 20 (`pollFirst` when size > 20) | `offerLast`/`pollFirst` **O(1)** | A fixed-size sliding buffer; O(20) space |
| 3 | `TreeMap<Instant, List<Order>>` + `subMap(t1, true, t2, false)` | **O(log n + k)** | Range queries need sorted keys (#023) |
| 4 | `PriorityQueue<Job>` (`PriorityBlockingQueue` if concurrent) | `offer`/`poll` **O(log n)** | Always the highest-priority job next |
| 5 | `HashMap<String, Integer>` with `merge` while counting (O(1) each), then sort the entries **once**: O(u log u) for u distinct words | Count: **O(N)** words | `TreeMap` during counting costs O(N log u). Counting in a hash map and sorting once at the end is cheaper |
| 6 | Two `ArrayDeque`s (undo stack, redo stack) | `push`/`pop` **O(1)** | LIFO; a new edit clears the redo stack |
| 7 | An LRU cache (`LinkedHashMapLruCache`, #033), capacity 1 000 | **O(1)** | Exactly the LRU problem |
| 8 | `HashSet<Integer>`: for each x, check `target − x` | **O(n)** | Instead of an O(n²) pairs loop (A1-style) |

</details>

---

## Self-Check

- [ ] Part A: ≥ 10/12 correct before opening the answers
- [ ] Can explain *why* `String +=` in a loop is O(n²)
- [ ] Ran the doubling experiment yourself and got ratios of ≈ 4 for the quadratic rows
- [ ] Can explain why `HashMap.get` slows down for very large maps despite being O(1)
- [ ] Part C: a choice + Big-O + reason for all 8 scenarios

---

## Handwritten Cheat Sheet

```
ANALYSE: inputs → loops (multiply) → price every call → recurrence → amortized → space
DOUBLING RATIO: 1 → O(1)/log · 2 → n · 2.1–2.3 → n log n · 4 → n² · 8 → n³
MEASURED ×4 (quadratic): list.contains dedupe · String += · removeAll(List) · LinkedList get(i)
FIXES ×2 (linear): LinkedHashSet · StringBuilder · removeAll(HashSet) · for-each
SPEEDUPS @40k: StringBuilder ~320× · removeAll(HashSet) 200×
O(1) HashMap.get: 3.5× slower at 8× size → cache misses (memory hierarchy, not Big-O)
Time ONLY the measured section (setup ≠ measurement)
```

---

**Next note:** #039 — Weekly review + spaced repetition (Anki); fix gaps (Week 2)
