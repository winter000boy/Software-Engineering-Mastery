# #031 — Solution: Gap-Fill: Revisit Your Weakest Collections Topic

**Date:** 2026-10-03  
**Problem:** [031-gap-fill-weakest-collections-topic.md](../problems/031-gap-fill-weakest-collections-topic.md)

> Every output below was produced by running the snippet on JDK 21.0.10. The timing in A-3 is from an Apple M4.

---

## Part 1 — Diagnostic Answers

| # | Output | Why | Revisit |
|---|--------|-----|---------|
| **A1** | `null` | `NoHash` overrides `equals` but not `hashCode`, so the two equal objects have different identity hash codes and land in different buckets. `get` never even calls `equals` on the stored key. (Technically the two identity hashes *could* collide; in practice they essentially never do, so the lookup misses.) | #020 §3.5, EJ Item 11 (#003) |
| **A2** | `false false 1` | `p` was filed under hash 1. After `p.x = 2`, `contains(p)` looks in bucket **2**, so `false`. `new MutablePoint(1)` looks in bucket 1 and finds `p`, but `equals` compares 1 with `p.x` = 2, so `false`. The entry is still there (`size 1`), but nobody can reach it. | #017 §9 |
| **A3** | `[30]` | `remove(1)` is `remove(int index)`, so it removes **20**. `remove(Integer.valueOf(10))` is `remove(Object)`, so it removes the value 10. | #017 §2 |
| **B1** | `[a, c]` | The trick question: **no exception.** Removing "b" (the second-to-last element) makes `size` = 2. The iterator's `cursor` is already 2, so `hasNext()` (`cursor != size`) returns **false**, and the loop ends before `next()` can run its `modCount` check. "c" is silently never visited. | see B-1 |
| **B2** | `ConcurrentModificationException` | Removing "a" leaves `size` = 2 with `cursor` = 1, so `hasNext()` is true, and `next()` detects `modCount != expectedModCount`. | #017 §8 |
| **B3** | `4 3` | `unmodifiableList` is a read-only **view** of `src`, so it sees the add. `List.copyOf` is a **snapshot**. | #017 §7 |
| **C1** | `Double` | You can **read** from `? extends Number` as `Number`. The runtime object is still a `Double`. | #026 §3 |
| **C2** | `Integer [7]` | You can **add** `Integer`s to `? super Integer`, but reads come back typed as `Object` (the runtime class is still `Integer`). `sink` and `ints` are the same list. | #026 §3 |
| **C3** | `true` | **Erasure:** both are just `ArrayList` at run time. | #019 Part D |
| **D1** | `[A, b]` | The comparator treats "a"/"A" and "b"/"B" as equal, so a `TreeSet` keeps the **first** one inserted of each pair ("b", then "A") and prints them in sorted order. | #023 §3.5 |
| **D2** | `5 4 size 3` | A max-heap via `reverseOrder()`; it allows duplicates (`1` twice), so 5 − 2 = 3 elements remain. | #017 §4 |
| **D3** | `[Point[x=1, y=2], Point[x=1, y=1], Point[x=0, y=9]]` | The trailing `.reversed()` reverses the **whole** chain: x descending **and** y descending. | #029 §4 |

---

## Drill Set A — Hashing & Complexity

### A-1. Fix `NoHash`

```java
@Override public int hashCode() { return Integer.hashCode(id); }   // same field as equals
// or simply: record NoHash(int id) { }  (generated equals + hashCode)
```

The test: `map.put(new NoHash(1), "a"); assertEquals("a", map.get(new NoHash(1)));`. It fails (`null`) before the fix and passes after it.

### A-2. Buckets by hand

Small `Integer` keys hash to themselves (`h ^ (h >>> 16)` = h when h < 65 536).

| Key | `& 15` (capacity 16) | `hash & 16` (new bit) | Bucket at capacity 32 |
|----:|----:|:---:|----:|
| 3 | 3 | 0 → **lo** | 3 |
| 19 | 3 | 16 → **hi** | 19 (= 3 + 16) |
| 35 | 3 | 0 → lo | 3 |
| 51 | 3 | 16 → hi | 19 |

- Capacity 16: **bucket 3 → 3 → 19 → 35 → 51** (all four collide, in insertion order, since appends go at the tail)
- Capacity 32: **bucket 3 → 3 → 35** and **bucket 19 → 19 → 51**, with relative order preserved by the lo/hi split

### A-3. O(n²) dedupe → O(n)

```java
List<Integer> result = new ArrayList<>(new LinkedHashSet<>(input));   // keeps first-seen order
```

**Measured** on 100 000 random ints with 43 233 unique values: `list.contains` loop **776 ms**; `LinkedHashSet` **7 ms**; identical results. `contains` on a list is O(size), and calling it n times is O(n²). A hash set makes each check O(1). `LinkedHashSet` rather than `HashSet` keeps the order (#022 Ex. 2).

---

## Drill Set B — Iteration & Mutation

### B-1. Why B1 is silent but B2 throws

`ArrayList`'s iterator (#021 has the same design):

```java
public boolean hasNext() { return cursor != size; }                // NO modCount check here
public E next() { checkForComodification(); ... cursor++; ... }    // the check is only here
```

| | Before the removal | After `l.remove(...)` | Next loop test |
|---|---|---|---|
| **B1** (remove "b" at index 1) | `cursor` = 2 (just returned "b") | `size` 3 → **2** | `hasNext()`: 2 != 2 → **false** → loop exits; `next()` never runs, so no CME; **"c" is skipped** |
| **B2** (remove "a" at index 0) | `cursor` = 1 | `size` → 2 | `hasNext()`: 1 != 2 → true → `next()` → `modCount` changed → **CME** |

The lesson: fail-fast is **best-effort** (the Javadoc says so). B1 is arguably worse than B2, because it silently skips an element.

### B-2. Removing every `2` from `[1, 2, 2, 3, 2, 4]`

```java
list.removeIf(x -> x == 2);                                        // ✔ [1, 3, 4] (best: one O(n) pass)

for (Iterator<Integer> it = list.iterator(); it.hasNext(); )       // ✔ [1, 3, 4]
    if (it.next() == 2) it.remove();

for (int i = list.size() - 1; i >= 0; i--)                         // ✔ [1, 3, 4] (backwards!)
    if (list.get(i) == 2) list.remove(i);
```

The **forward** index loop gives `[1, 2, 3, 4]` (verified). Removing at `i` shifts the next element into position `i`, and then `i++` skips it, so adjacent 2s survive. Iterating **backwards** avoids this, because the shifted elements have already been checked.

(`x == 2` with `Integer`s works here because −128..127 are cached. For general values use `x.equals(2)` or `x.intValue() == 2`.)

### B-3. Map removal

```java
map.keySet().forEach(k -> { if (map.get(k) < 10) map.remove(k); });
// ✔ throws ConcurrentModificationException... AFTER doing the work: the map is {b=15, d=20}
```

`HashMap.KeySet.forEach` runs the whole loop and checks `modCount` **once, at the end**. So the removals happened, *and* you got an exception. That's a nasty combination in production code, since a caller who catches the exception may assume nothing changed.

Correct one-liners (both verified, both giving `{b=15, d=20}`):

```java
map.entrySet().removeIf(e -> e.getValue() < 10);
map.values().removeIf(v -> v < 10);
```

---

## Drill Set C — Generics

### C-1. Signatures

```java
static double sum(Collection<? extends Number> numbers)                      // (a) producer
static <T> void copyInto(Collection<? super T> dest, Iterable<? extends T> src)   // (b) PECS
static <T> Optional<T> maxBy(Iterable<? extends T> items, Comparator<? super T> cmp) // (c)
```

### C-2. Compile errors

| Line | `javac` error (#026 §8) | Fix |
|------|-------------------------|-----|
| `List<Number> nums = new ArrayList<Integer>();` | `incompatible types: ArrayList<Integer> cannot be converted to List<Number>` | `List<Number> nums = new ArrayList<>();` or `List<? extends Number> nums = …` |
| `l.add(1)` on `List<? extends Number>` | `incompatible types: int cannot be converted to CAP#1` | Make the parameter `List<? super Integer>` (it's a consumer) |
| `return new T[n];` | `generic array creation` | `static <T> T[] make(IntFunction<T[]> gen, int n) { return gen.apply(n); }`, called as `make(String[]::new, n)` |

### C-3. Arrays vs generics

```java
Object[] a = new String[1];
a[0] = 1;              // ✔ ArrayStoreException: java.lang.Integer
```

Arrays are **covariant** (`String[]` *is an* `Object[]`) and **reified** (the array knows it's a `String[]`), so the bad store is caught at **run time**. Generics are **invariant** and **erased**: the run-time `ArrayList` doesn't know its element type, so it couldn't catch the bad insert later. The compiler therefore forbids the assignment up front (EJ Item 28, #019).

---

## Drill Set D — Ordering

### D-1. Explaining D3

`comparingInt(Point::x).thenComparingInt(Point::y).reversed()` reverses the *combined* comparator: x descending, then y descending. So `(1,2)` comes before `(1,1)`. For x descending, y ascending:

```java
Comparator.comparingInt(Point::x).reversed().thenComparingInt(Point::y)
// or: Comparator.comparing(Point::x, Comparator.reverseOrder()).thenComparingInt(Point::y)
```

### D-2. Task order

```java
tasks.sort(Comparator.comparingInt(Task::priority).reversed()
        .thenComparing(Task::deadline, Comparator.nullsLast(Comparator.naturalOrder()))
        .thenComparingInt(Task::id));
// ✔ ids [3, 5, 2, 4, 1]
```

The priority 5 group: 3 and 5 share the deadline 10-05 and tie-break on id, then 2 (null deadline) goes last. The priority 2 group: 4 (10-01), then 1 (10-10).

The `.reversed()` here comes **before** `thenComparing`, so it applies only to priority (contrast D3).

### D-3. Length-only `TreeMap`

```java
TreeMap<String, Integer> byLen = new TreeMap<>(Comparator.comparingInt(String::length));
// fed: cat, dog, horse, ox, zebra
// ✔ {ox=1, cat=2, horse=2}
```

The comparator says "dog" **equals** "cat" (both length 3) and "zebra" equals "horse" (both length 5). `merge` treats them as the same key, so it adds to the existing count and keeps the **first** key object. Two words vanish and the counts are wrong. **Fix: a tie-breaker** that makes the order total:

```java
new TreeMap<>(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()))
// ✔ {ox=1, cat=1, dog=1, horse=1, zebra=1}
```

---

## Key Takeaways

1. Diagnose before you drill. Rereading notes doesn't tell you what you can't do.
2. **B1 is the scariest result:** fail-fast iteration can *silently skip* elements instead of throwing.
3. Bulk operations that "fail" may have already done the work (`keySet().forEach` + `remove`).
4. Mutable keys and `equals`-without-`hashCode` break hash collections without any exception at all.
5. Comparators that aren't total orders **merge data** in sorted maps and sets.
6. Every miss becomes an Anki card, so the next review targets exactly what you got wrong.
