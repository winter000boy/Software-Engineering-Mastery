# #029 — Comparable vs Comparator

**Date:** 2026-09-24  
**Roadmap row:** 29 — P1 Java Core · Collections & Generics · *Learn*  
**Sources:** `java.lang.Comparable` / `java.util.Comparator` Javadoc (JDK 21), OpenJDK 21 `Arrays.java`, *Effective Java* Item 14  
**Hands-on companion:** #027 (sorting exercise: 12 tests, including the failure modes below)

> Every ✔︎ result and error message was produced on JDK 21.0.10.

---

## At a Glance (copy to handwritten notes)

- **`Comparable<T>`** = the class's **natural ordering**, *inside* the class: `int compareTo(T o)`. **One** per class. Used by `Collections.sort(list)`, `TreeMap`/`TreeSet` without a comparator, `Arrays.sort(Object[])`, `stream().sorted()`.
- **`Comparator<T>`** = an **external** ordering, a separate object: `int compare(T a, T b)`. **Any number** per class. Works for classes you can't modify.
- Both return **negative / zero / positive**. The **sign** matters, not the magnitude.
- **Contract:** antisymmetric `sgn(cmp(a,b)) == -sgn(cmp(b,a))`, transitive, consistent. Break it and TimSort may throw **"Comparison method violates its general contract!"** ✔︎ (44/50 sorts in #027).
- **Strongly recommended: consistent with `equals`** (`cmp == 0` ⇔ `equals`). Otherwise `TreeSet`/`TreeMap` **merge "equal" elements** (BigDecimal `2.0` vs `2.00` ✔︎; employees with the same salary ✔︎).
- **Never compare by subtraction** (overflow ✔︎). Use `Integer.compare` / `Comparator.comparingInt`.
- **Build comparators with combinators:** `comparing`, `comparingInt`, `thenComparing`, `reversed`, `nullsFirst`/`nullsLast`, `naturalOrder`, `reverseOrder`.
- ⚠️ A trailing `.reversed()` reverses the **whole chain** ✔︎. To reverse one key, use `thenComparing(key, Comparator.reverseOrder())`.
- ⚠️ `Comparator.comparing(p -> p.name()).reversed()` **doesn't compile** (`p` is inferred as `Object`) ✔︎. Use `P::name` or `(P p) -> p.name()`.
- Sorting algorithms ✔︎ (JDK source): **objects** → **TimSort**, **stable**. **Primitives** → **Dual-Pivot Quicksort**, not stable (stability is meaningless for primitives anyway).

---

## 1. Side by Side

| | `Comparable<T>` | `Comparator<T>` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `int compareTo(T other)` | `int compare(T a, T b)` |
| Where the logic lives | **Inside** the class being compared | In a **separate** object (often a lambda or a `static final` constant) |
| How many per class | **One**: the "natural" order | **Unlimited** |
| Works for classes you don't own | ❌ (you can't add `implements`) | ✅ |
| Typical users | `Collections.sort(list)`, `new TreeSet<>()`, `Arrays.sort(arr)`, `Stream.sorted()` | `list.sort(c)`, `new TreeSet<>(c)`, `Stream.sorted(c)`, `Collections.max(coll, c)`, `PriorityQueue(c)` |
| Functional interface? | Technically single-method, but not meant for lambdas | ✅ `@FunctionalInterface`, built for lambdas and combinators |
| JDK examples | `String`, `Integer`, `BigDecimal`, `LocalDate`, `Instant`, all enums | `String.CASE_INSENSITIVE_ORDER`, `Collator`, `Map.Entry.comparingByKey()` |

```java
// Natural order: what "sorted" means for this type, by default
public record Version(int major, int minor, int patch) implements Comparable<Version> {
    private static final Comparator<Version> ORDER =
            Comparator.comparingInt(Version::major)
                      .thenComparingInt(Version::minor)
                      .thenComparingInt(Version::patch);
    @Override public int compareTo(Version o) { return ORDER.compare(this, o); }
}

// External orders: as many as the use cases need
Comparator<Employee> bySalaryDesc = Comparator.comparing(Employee::salary, Comparator.reverseOrder());
Comparator<Employee> byHireDate   = Comparator.comparing(Employee::hireDate);
```

**Records and plain classes are *not* `Comparable` by default** ✔︎. You decide whether a natural order exists. **Enums are always `Comparable`** in **declaration order** (`compareTo` is `final` and based on `ordinal()`; `SMALL.compareTo(LARGE)` ✔︎ = −2).

---

## 2. When to Implement `Comparable`

Implement it when the type has **one obvious, universally agreed ordering**:

| Type | Natural order? | Reason |
|------|:---:|--------|
| `Money`, `Version`, `Isbn`, `Temperature` | ✅ | One obvious order |
| Value types with a unique key (`OrderId`, `MemberId` from #005) | ✅ | Order by the key |
| `Employee`, `Product`, `Book` | Usually ❌ | By name? Salary? Date? No single right answer, so provide comparators instead |
| An entity you want in a `TreeSet` | ✅ only if consistent with `equals` | Or pass a comparator with a unique tie-breaker |

(In #027, `Employee` *does* implement `Comparable` by `id`, with tie-breakers on every other field so that it's consistent with the record's `equals`. The business orderings are all `Comparator`s.)

---

## 3. The Contract (both interfaces)

For all `x`, `y`, `z`, writing `sgn` for the sign function:

| Rule | Formula | What breaks if violated |
|------|---------|-------------------------|
| **Antisymmetry** | `sgn(cmp(x,y)) == -sgn(cmp(y,x))` (so `cmp(x,x) == 0`) | Sorts misbehave or throw; binary search fails |
| **Transitivity** | `cmp(x,y) > 0 && cmp(y,z) > 0` ⇒ `cmp(x,z) > 0` | Sorts produce garbage or throw |
| **Consistency** | `cmp(x,y) == 0` ⇒ `sgn(cmp(x,z)) == sgn(cmp(y,z))` | Tree structures become inconsistent |
| *Recommended:* **consistent with equals** | `(cmp(x,y) == 0) == x.equals(y)` | Sorted sets/maps disagree with `Set`/`Map` semantics |

### Violation 1 — a comparator that never returns 0 ✔︎

```java
Comparator<Integer> broken = (x, y) -> x < y ? -1 : 1;   // cmp(x, x) == 1: not antisymmetric
list.sort(broken);   // ✔︎ in 44 of 50 random 2 000-element sorts:
// java.lang.IllegalArgumentException: Comparison method violates its general contract!
```

TimSort **detects** some violations (it relies on the contract when merging runs) and fails loudly. That's the good case. Other times you just get a silently wrong order. Detection is probabilistic, so the bug can pass tests and fail in production on different data.

### Violation 2 — subtraction overflow ✔︎

```java
(x, y) -> x - y            // Integer.MAX_VALUE - (-1) == Integer.MIN_VALUE: wraps negative!
// ✔︎ sorts [MAX, -1, MIN, 0] into [0, MIN, -1, MAX]: wrong
Integer::compare           // ✔︎ [MIN, -1, 0, MAX]
```

### Violation 3 — inconsistent with equals (legal, but surprising) ✔︎

```java
new BigDecimal("2.0").compareTo(new BigDecimal("2.00"))   // ✔︎ 0
new BigDecimal("2.0").equals(new BigDecimal("2.00"))      // ✔︎ false (different scale)
// HashSet holds 2 elements, TreeSet holds 1 (#003)

TreeSet<Employee> bySalary = new TreeSet<>(Comparator.comparing(Employee::salary));
bySalary.addAll(staff);    // ✔︎ 6 of 7 employees: two earn 120 000, so one is silently DROPPED (#027)
```

**Fix: end every comparator used by a sorted collection with a unique tie-breaker:** `.thenComparingInt(Employee::id)`.

The Javadoc's words: *"the natural ordering for a class C is said to be consistent with equals if and only if e1.compareTo(e2) == 0 has the same boolean value as e1.equals(e2)… It is strongly recommended (though not required) that natural orderings be consistent with equals."*

---

## 4. The `Comparator` Toolkit (Java 8+)

| Factory / method | Meaning | Example |
|------------------|---------|---------|
| `Comparator.comparing(keyFn)` | Compare by an extracted `Comparable` key | `comparing(Employee::name)` |
| `comparing(keyFn, keyComparator)` | …using a specific comparator for the key | `comparing(Employee::name, String.CASE_INSENSITIVE_ORDER)` |
| `comparingInt` / `comparingLong` / `comparingDouble` | Primitive keys, **no boxing** | `comparingInt(String::length)` |
| `.thenComparing(...)` (+ `Int`/`Long`/`Double` variants) | Tie-breaker, applied only when the previous keys are equal | `.thenComparing(Employee::name)` |
| `.reversed()` | Reverse the **entire** comparator so far | |
| `Comparator.naturalOrder()` / `reverseOrder()` | Natural order / its reverse | `comparing(Employee::salary, reverseOrder())` |
| `Collections.reverseOrder()` / `reverseOrder(c)` | Older equivalents | `new TreeSet<>(Collections.reverseOrder())` → `[3, 2, 1]` ✔︎ |
| `nullsFirst(c)` / `nullsLast(c)` | Wraps `c` to tolerate `null`s | `nullsFirst(naturalOrder())` → `[null, a, b]` ✔︎ |
| `Map.Entry.comparingByKey()` / `comparingByValue()` | For sorting map entries | Top-K in #022 |
| `String.CASE_INSENSITIVE_ORDER` | Case-insensitive, **not** locale-aware: `Émile` still sorts *after* `zoe` ✔︎ | |
| `Collator.getInstance(locale)` | Human, locale-aware text order (`Émile` between `Dan` and `Eve` ✔︎) | #027 |

### ⚠️ Where `.reversed()` binds ✔︎

```java
comparingInt(P::age).thenComparing(P::name).reversed()
// ✔︎ reverses BOTH: age DESC, then name DESC → [bo(30), al(30), cy(25)]

comparingInt(P::age).thenComparing(P::name, Comparator.reverseOrder())
// ✔︎ age ASC, name DESC only → [cy(25), bo(30), al(30)]
```

A `reversed()` at the end applies to everything before it. Reverse a single key by passing `reverseOrder()` as **that key's** comparator.

### ⚠️ Lambda inference and `.reversed()` ✔︎

```java
ps.sort(Comparator.comparing(p -> p.name()).reversed());
```
```text
error: cannot find symbol
  symbol:   method name()
  location: variable p of type Object
```

`comparing(...)` is the **receiver** of `.reversed()`, so there's no target type to infer `T` from, and `p` becomes `Object`. Fixes ✔︎:

```java
Comparator.comparing(P::name).reversed()               // a method reference carries its type
Comparator.comparing((P p) -> p.name()).reversed()     // explicit lambda parameter type
Comparator.<P, String>comparing(p -> p.name()).reversed()   // type witness
```

### `null` handling ✔︎

```java
List<String> s = Arrays.asList("b", null, "a");
s.sort(Comparator.naturalOrder());                     // ✔︎ NullPointerException
s.sort(Comparator.nullsFirst(Comparator.naturalOrder()));   // ✔︎ [null, a, b]
Comparator.comparing(Employee::manager, Comparator.nullsLast(Comparator.naturalOrder()))   // null KEYS (#027)
```

`nullsFirst(comparing(Employee::manager))` would be wrong: it handles null *employees*, not null *managers*. Put the null handling on the **key** comparator.

### Performance notes

- **Build comparators once**, as `static final` constants. They're immutable and thread-safe; don't rebuild them in a loop.
- **Primitive specialisations** (`comparingInt`) avoid boxing on every comparison, which matters in sorts of millions of elements.
- A **`Collator`** is much slower than `String.compareTo`. For large sorts, precompute `CollationKey`s: `collator.getCollationKey(s)`, then compare the keys.

---

## 5. Sorting Algorithms in the JDK ✔︎ (from `Arrays.java`, JDK 21)

```java
public static void sort(int[] a) {
    DualPivotQuicksort.sort(a, 0, 0, a.length);            // primitives
}
public static void sort(Object[] a) {
    if (LegacyMergeSort.userRequested)
        legacyMergeSort(a);
    else
        ComparableTimSort.sort(a, 0, a.length, null, 0, 0); // objects: TimSort
}
```

| | Objects (`Arrays.sort(T[])`, `List.sort`, `Collections.sort`) | Primitives (`Arrays.sort(int[])` …) |
|---|---|---|
| Algorithm | **TimSort** (a merge sort + insertion-sort hybrid that exploits existing runs) | **Dual-Pivot Quicksort** (+ insertion sort, heap sort fallback) |
| Stable? | ✅ *"This sort is guaranteed to be stable"* | ❌ (irrelevant, since equal primitives are indistinguishable) |
| Complexity | O(n log n) worst; **O(n) on already-sorted input** | O(n log n) |
| Uses your comparator | ✅ | ❌ (natural numeric order only) |

**Stability** means equal elements keep their relative order. That enables **multi-pass sorting**: sort by name, then (stably) by department, and names stay alphabetical within each department (the #027 test `listSortIsStable` ✔︎). It's usually clearer to write one chained comparator, though.

`List.sort` on an `ArrayList` sorts its internal array directly. `Collections.sort(list)` just calls `list.sort(null)`.

---

## 6. Where Each Is Used

| API | Comparable (natural) | Comparator |
|-----|----------------------|------------|
| `Collections.sort(list)` / `list.sort(null)` | ✅ | |
| `list.sort(c)` / `Collections.sort(list, c)` | | ✅ |
| `new TreeSet<>()` / `new TreeMap<>()` | ✅ (non-Comparable → `ClassCastException` ✔︎) | |
| `new TreeSet<>(c)` / `new TreeMap<>(c)` | | ✅ |
| `new PriorityQueue<>()` / `(c)` | ✅ | ✅ |
| `Arrays.sort(arr)` / `Arrays.sort(arr, c)` | ✅ | ✅ |
| `Collections.max(coll)` / `max(coll, c)` | ✅ | ✅ |
| `stream.sorted()` / `sorted(c)`, `max(c)`, `min(c)` | ✅ | ✅ (`max`/`min` need one) |
| `Arrays.binarySearch(arr, key)` / `(arr, key, c)` | ✅ | ✅ (must be the *same* order used to sort) |

---

## 7. Decision Guide

```
Does the type have ONE obvious order everyone would agree on?
 ├─ yes → implement Comparable<T> (consistent with equals; build it from Comparator combinators)
 └─ no  → don't; provide static final Comparators for each business ordering
Need a different order than the natural one, or ordering a class you don't own?
 └─ Comparator
Putting it in a TreeSet/TreeMap/PriorityQueue?
 └─ end the comparator with a unique tie-breaker, unless you WANT "equal" elements merged
Sorting human-visible text?
 └─ Collator (locale-aware), not String.compareTo / CASE_INSENSITIVE_ORDER
```

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| Comparable vs Comparator? | Natural order inside the class (one) / external strategy objects (many) |
| Which do `TreeSet` and `Collections.sort` use by default? | `Comparable` (natural order) |
| What does "consistent with equals" mean, and why care? | `compareTo == 0` ⇔ `equals`. If not, sorted sets/maps merge elements that `equals` considers different |
| Why is `a - b` wrong? | Integer overflow flips the sign |
| "Comparison method violates its general contract!"? | TimSort detected a broken comparator (not antisymmetric/transitive) |
| Is `Collections.sort` stable? | Yes (TimSort). `Arrays.sort(int[])` uses dual-pivot quicksort |
| How do you sort by salary desc, then name asc? | `comparing(Employee::salary, reverseOrder()).thenComparing(Employee::name)` |
| Why won't `comparing(p -> p.name()).reversed()` compile? | No target type → `p` is `Object`; use `P::name` or a typed lambda |
| How do you handle null fields? | `comparing(keyFn, nullsFirst(naturalOrder()))` on the key |
| Are enums Comparable? | Yes: declaration (ordinal) order; `compareTo` is final |
| Are records Comparable? | No, not unless you implement it |

---

## Handwritten Cheat Sheet

```
Comparable<T>: compareTo(T) — natural order, INSIDE class, ONE (String, Integer, LocalDate, enums)
Comparator<T>: compare(a,b) — external, MANY, lambdas, for classes you don't own
return <0 / 0 / >0 (sign only)
CONTRACT: antisymmetric · transitive · consistent  (+ recommended: consistent with equals)
  broken → TimSort IAE "violates its general contract!" (probabilistic)
  a - b → overflow → use Integer.compare / comparingInt
  inconsistent w/ equals → TreeSet drops "equal" elements → add .thenComparingInt(id)
TOOLKIT: comparing(key[, keyCmp]) · comparingInt/Long/Double · thenComparing
         reversed() (WHOLE chain!) · reverseOrder() per key · nullsFirst/Last on the KEY
         naturalOrder · Map.Entry.comparingByValue · CASE_INSENSITIVE_ORDER · Collator(locale)
LAMBDA: comparing(p -> p.name()).reversed() ✗ (p is Object) → P::name or (P p) ->
SORTS: objects = TimSort, STABLE, O(n) if presorted · primitives = dual-pivot quicksort
ENUM compareTo = ordinal, final · RECORD = not Comparable unless you say so
```

---

**Next:** row 30 — Write a generic container class (#030)
