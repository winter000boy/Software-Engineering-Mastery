# #031 — Gap-Fill: Revisit Your Weakest Collections Topic

**Date:** 2026-10-03  
**Type:** Flex (diagnose → drill)  
**Roadmap row:** 31 — P1 Java Core · Collections & Generics  
**Covers:** rows 17–30 (#017–#030)

---

## Overview

"Revisit your weakest topic" only works if you know which topic that is. So this task has two parts:

1. **Diagnose (15 min):** 12 *predict-the-output* questions, 3 per area. No IDE, no running code. Write your answer for each, then score yourself with the solution.
2. **Drill (45–60 min):** do the drill set for your **lowest-scoring area** (do two areas if you tie). Each drill targets a misconception that the diagnostic questions expose.

| Area | Topics | Source notes |
|------|--------|--------------|
| **A** | Hashing internals, `equals`/`hashCode`, complexity | #017, #020, #024 |
| **B** | Iteration, fail-fast, views vs copies | #017, #019, #021 |
| **C** | Generics, wildcards, erasure | #019, #026, #030 |
| **D** | Ordering, comparators, sorted collections | #023, #027, #029 |

---

## Part 1 — Diagnostic (predict the output)

For each snippet, write exactly what is printed, or the exception's simple name.

```java
record Point(int x, int y) { }
class MutablePoint {                       // equals + hashCode on the MUTABLE field x
    int x;
    MutablePoint(int x) { this.x = x; }
    public boolean equals(Object o) { return o instanceof MutablePoint m && m.x == x; }
    public int hashCode() { return x; }
}
class NoHash {                             // equals overridden, hashCode NOT
    final int id;
    NoHash(int id) { this.id = id; }
    public boolean equals(Object o) { return o instanceof NoHash n && n.id == id; }
}
```

### Area A — internals & complexity

```java
// A1
Map<NoHash, String> m = new HashMap<>();
m.put(new NoHash(1), "a");
System.out.println(m.get(new NoHash(1)));

// A2
Set<MutablePoint> s = new HashSet<>();
MutablePoint p = new MutablePoint(1);
s.add(p);
p.x = 2;
System.out.println(s.contains(p) + " " + s.contains(new MutablePoint(1)) + " " + s.size());

// A3
List<Integer> l = new ArrayList<>(List.of(10, 20, 30));
l.remove(1);
l.remove(Integer.valueOf(10));
System.out.println(l);
```

### Area B — iteration & mutation

```java
// B1
List<String> l = new ArrayList<>(List.of("a", "b", "c"));
for (String s : l) if (s.equals("b")) l.remove(s);
System.out.println(l);

// B2  (same, but removing "a")
List<String> l = new ArrayList<>(List.of("a", "b", "c"));
for (String s : l) if (s.equals("a")) l.remove(s);
System.out.println(l);

// B3
List<Integer> src = new ArrayList<>(List.of(1, 2, 3));
List<Integer> view = Collections.unmodifiableList(src);
List<Integer> copy = List.copyOf(src);
src.add(4);
System.out.println(view.size() + " " + copy.size());
```

### Area C — generics

```java
// C1
List<? extends Number> nums = new ArrayList<>(List.of(1, 2.5));
Number n = nums.get(1);
System.out.println(n.getClass().getSimpleName());

// C2
List<Integer> ints = new ArrayList<>();
List<? super Integer> sink = ints;
sink.add(7);
Object o = sink.get(0);
System.out.println(o.getClass().getSimpleName() + " " + ints);

// C3
System.out.println(new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass());
```

### Area D — ordering

```java
// D1
TreeSet<String> t = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
t.addAll(List.of("b", "A", "a", "B"));
System.out.println(t);

// D2
PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder());
pq.addAll(List.of(3, 1, 4, 1, 5));
System.out.println(pq.poll() + " " + pq.poll() + " size " + pq.size());

// D3
List<Point> ps = new ArrayList<>(List.of(new Point(1, 2), new Point(1, 1), new Point(0, 9)));
ps.sort(Comparator.comparingInt(Point::x).thenComparingInt(Point::y).reversed());
System.out.println(ps);
```

**Scoring:** 1 point per fully correct answer. Your weakest area is the lowest score out of 3. If an area scores 3/3 **and** you could explain *why* for each, skip its drills.

---

## Part 2 — Drill Sets (do the set for your weakest area)

### Drill set A — hashing & complexity

- **A-1.** Fix `NoHash` so that A1 prints `a`. Write a test that fails before the fix and passes after it.
- **A-2.** Integer keys `3, 19, 35, 51` are put into a `HashMap` with capacity 16. By hand: list the bucket(s) and chain order. Then the table resizes to 32: list the buckets and chains again, using only `hash & oldCap` (#020 §3.7).
- **A-3.** This deduplication is O(n²). Prove it by measuring 100 000 elements, then rewrite it in O(n) **preserving first-seen order**:

  ```java
  List<Integer> result = new ArrayList<>();
  for (Integer x : input) if (!result.contains(x)) result.add(x);
  ```

### Drill set B — iteration & mutation

- **B-1.** Explain *exactly* why B1 doesn't throw but B2 does. (Hint: what do `hasNext()` and `next()` check, and when?)
- **B-2.** Remove every `2` from `[1, 2, 2, 3, 2, 4]` in three correct ways (`removeIf`, an explicit `Iterator`, an index loop). Then explain why a **forward** index loop gives the wrong answer.
- **B-3.** Remove all map entries with value < 10 from `{a=5, b=15, c=8, d=20}`. Predict what `map.keySet().forEach(k -> { if (map.get(k) < 10) map.remove(k); })` does, including the **contents of the map afterwards**, then give two correct one-liners.

### Drill set C — generics

- **C-1.** Write the signatures (bodies optional) for: (a) `sum`, which totals any collection of numbers; (b) `copyInto`, which copies elements of type T from a source into a destination; (c) `maxBy`, which returns the largest element using a comparator.
- **C-2.** Each line fails to compile. Name the error and fix it:

  ```java
  List<Number> nums = new ArrayList<Integer>();
  void addOne(List<? extends Number> l) { l.add(1); }
  <T> T[] make(int n) { return new T[n]; }
  ```
- **C-3.** `Object[] a = new String[1]; a[0] = 1;` compiles. What happens at run time, and why do generics refuse the equivalent `List<Object> l = new ArrayList<String>();`?

### Drill set D — ordering

- **D-1.** Explain D3's output. Then write the comparator that gives *x descending, y ascending*.
- **D-2.** Sort tasks by **priority descending**, then **deadline ascending with missing (`null`) deadlines last**, then **id ascending**. Data, as `Task(id, priority, deadline)`: `(1, 2, 2026-10-10)`, `(2, 5, null)`, `(3, 5, 2026-10-05)`, `(4, 2, 2026-10-01)`, `(5, 5, 2026-10-05)`. Predict the id order before running it.
- **D-3.** A word-count `TreeMap` built with `Comparator.comparingInt(String::length)` is fed `cat, dog, horse, ox, zebra` via `merge(w, 1, Integer::sum)`. Predict the map, explain it, and fix it.

---

## Part 3 — Close the Loop

1. Add an Anki card (#028 format) for every diagnostic question you missed. Put the snippet on the front and the output plus the *reason* on the back.
2. Re-run the diagnostic for your weak area in 3 days without looking.
3. Record in your notebook: weakest area, score before, score after.

---

## Acceptance Criteria

- [ ] All 12 diagnostic answers written **before** checking
- [ ] Weakest area identified from the scores
- [ ] Every drill in that area's set done, with code run where the drill asks for it
- [ ] Anki cards added for the misses

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
