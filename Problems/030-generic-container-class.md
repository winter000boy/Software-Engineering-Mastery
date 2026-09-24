# #030 — Write a Generic Container Class

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 30 — P1 Java Core · Collections & Generics  
**Topics:** generic classes and methods, bounded types, PECS, recursive bounds, generic records, static generic factories, `@SafeVarargs`, custom `Collector`s, fail-fast iterators  
**Theory:** #019 (EJ Ch. 5), #026 (generics mechanics)

---

## Overview

Design and build **`Multiset<E>`** (a "bag"): a collection that remembers **how many times** each element was added. `{apple ×2, pear ×1}` is a multiset. Java has no built-in one (Guava's `Multiset` is the well-known library version), so it's an ideal exercise: it's small, genuinely useful, and every method needs a generics decision.

---

## Required API

| Member | Requirement |
|--------|-------------|
| `record Entry<E>(E element, int count)` | Nested generic record; validates `element != null` and `count > 0` |
| `static <E> Multiset<E> create()` | Hash-based, unordered |
| `static <E> Multiset<E> linked()` | Iterates in first-insertion order |
| `static <E extends Comparable<? super E>> Multiset<E> sorted()` | Natural order. **Why `? super E`?** |
| `static <E> Multiset<E> sorted(Comparator<? super E> c)` | Comparator order |
| `@SafeVarargs static <E> Multiset<E> of(E... elements)` | Justify why it's safe |
| `static <E> Multiset<E> copyOf(Iterable<? extends E> source)` | |
| `int add(E e)` / `int add(E e, int occurrences)` | Return the **previous** count; reject `null` and negatives; guard against count overflow |
| `boolean remove(Object o)` / `int remove(Object o, int occurrences)` | Parameter type `Object`, like `Collection.remove`. Never store zero counts |
| `int count(Object o)` | 0 when absent, **including** a foreign type in a sorted multiset (no `ClassCastException` escapes) |
| `int size()` (total occurrences, **O(1)**), `int distinctCount()`, `boolean isEmpty()`, `boolean contains(Object)` | |
| `void addAll(Iterable<? extends E> source)` | Producer |
| `void drainTo(Collection<? super E> sink)` | Consumer; empties the multiset |
| `Set<E> elementSet()` | Unmodifiable live view |
| `List<Entry<E>> entries()` / `List<Entry<E>> mostCommon(int n)` | `mostCommon`: by count descending; ties keep the multiset's own order |
| `static <T extends Comparable<? super T>> Optional<T> max(Multiset<? extends T> m)` | Like `Collections.max` |
| `static <T> Collector<T, ?, Multiset<T>> toMultiset()` and `toMultiset(Function<? super T, ? extends K>)` | For `stream.collect(...)`, **including parallel streams** |
| `Iterator<E> iterator()` | Yields each element **count** times; fail-fast |
| `equals` / `hashCode` / `toString` | Equal if the same elements have the same counts, **regardless of ordering**; `toString` like `[apple x 2, fig]` |

---

## Generics Checks (must compile, and are verified by tests)

```java
Multiset<Number> numbers = Multiset.create();
numbers.addAll(List.of(1, 2, 2));          // List<Integer> into Multiset<Number>
List<Object> sink = new ArrayList<>();
numbers.drainTo(sink);                     // Multiset<Number> into List<Object>

Multiset<Integer> m = Multiset.sorted(Comparator.comparing(Object::toString));   // Comparator<Object>

Multiset<Timestamp> times = ...;           // Timestamp is Comparable<Date>, not Comparable<Timestamp>
Optional<Timestamp> latest = Multiset.max(times);
Optional<Date> asDate      = Multiset.max(times);

Multiset<Integer> lengths = words.stream().collect(Multiset.toMultiset(String::length));
```

---

## Tasks

1. Implement `Multiset<E>` (backed by a `Map<E, Integer>`; the factory chooses the map implementation).
2. For every signature with a wildcard or bound, write one sentence: producer or consumer, and what would fail to compile without it.
3. Tests covering: counting and previous-count returns; `null`/negative rejection; the three orderings (and that they're `equals` to each other); both PECS directions; a `Comparator<Object>`; the `Timestamp` `max` case; `mostCommon` tie stability; the collector on a sequential **and** a parallel stream (10 000 elements); iteration that repeats elements; fail-fast; a foreign-type `count` on a sorted multiset.
4. Answer: why reject `null` elements?

---

## Stretch Goals

- `setCount(E e, int count)` and `retainAll`/`removeAll` with the right wildcard types.
- `unmodifiableView()` returning a read-only multiset.
- Replace the `Integer` counts with a mutable int holder to avoid boxing on every `add`, and measure the difference with JMH (#018).

---

## Acceptance Criteria

- [ ] Every API member implemented; `size()` is O(1)
- [ ] Every wildcard/bound justified in writing
- [ ] The generics-check snippet compiles and behaves as stated
- [ ] The collector works in parallel (the combiner is correct)
- [ ] Zero unchecked warnings except one justified, narrowly suppressed cast
- [ ] All tests green

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
