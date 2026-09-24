# #021 — Implement Your Own Dynamic Array

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 21 — P1 Java Core · Collections & Generics  
**Topics:** resizable arrays, amortized analysis, generics with `Object[]` storage, fail-fast iterators, `equals`/`hashCode` contracts, obsolete references (EJ Item 7)  
**Spec source:** #020 Part 1 (`ArrayList` internals)

---

## Overview

Build `DynamicArray<E>`, a from-scratch version of the core of `java.util.ArrayList`. Don't extend or wrap any JDK collection. The only things you may use are `Object[]`, `Arrays.copyOf`, `System.arraycopy` and `Objects`.

---

## Required API

| Method | Behaviour | Target complexity |
|--------|-----------|-------------------|
| `DynamicArray()` | Empty; **allocate storage lazily** on the first add | O(1) |
| `DynamicArray(int initialCapacity)` | Reject negative values | O(1) |
| `int size()`, `boolean isEmpty()` | | O(1) |
| `E get(int index)` | Bounds-checked (`IndexOutOfBoundsException`) | O(1) |
| `E set(int index, E e)` | Returns the old element | O(1) |
| `void add(E e)` | Append | **amortized O(1)** |
| `void add(int index, E e)` | Insert, shifting right. `index == size` is allowed | O(n − index) |
| `E removeAt(int index)` | Remove, shifting left; return the removed element | O(n − index) |
| `boolean remove(Object o)` | Remove the first occurrence (`null`-safe) | O(n) |
| `int indexOf(Object o)`, `boolean contains(Object o)` | `null`-safe | O(n) |
| `void clear()` | Must release references for GC | O(n) |
| `void ensureCapacity(int min)`, `void trimToSize()` | Capacity management | O(n) |
| `Iterator<E> iterator()` | **Fail-fast**; supports `remove()` | O(1) per step |
| `equals`, `hashCode`, `toString` | Same contract and format as `java.util.List` | O(n) |

The class must implement `Iterable<E>` so for-each works.

---

## Design Rules

1. **Growth:** start at 10 on the first add, then grow by **×1.5** (`old + (old >> 1)`), or to the requested minimum if that's larger. Guard against `int` overflow near `Integer.MAX_VALUE`.
2. **Storage** is an `Object[]` (explain why not `E[]`). Keep exactly one `@SuppressWarnings("unchecked")`, on the smallest possible scope, with a comment justifying it.
3. **No memory leaks:** removing an element must null out the freed slot.
4. **Fail-fast:** track structural modifications in a `modCount`. `set` is **not** structural. The iterator's own `remove()` is allowed.
5. `null` elements are permitted.

---

## Tasks

1. Implement the class.
2. Write JUnit 5 tests for at least:
   - lazy allocation (capacity 0 → 10 on the first add)
   - the exact capacity sequence while adding 200 elements (it must match `ArrayList`: 10, 15, 22, …)
   - bounds checks on `get`/`set`/`add(index)`
   - insert/remove at the front, middle and end
   - `null` handling in `contains`/`indexOf`/`remove`
   - **a removed element is garbage-collectable** (hint: `WeakReference` + `System.gc()`)
   - iterator `remove()`, `IllegalStateException` on a double remove, `ConcurrentModificationException` on outside modification
   - `equals`/`hashCode` equal to `List.of(...)`'s for the same elements
   - a **differential test**: 50 000 random operations applied to both your array and an `ArrayList`, and the results must always match
3. Answer: why is `add(e)` amortized O(1)? Show the arithmetic.
4. Answer: why does `ArrayList` use ×1.5 instead of ×2? What are the trade-offs?

---

## Stretch Goals

- Implement `RandomAccess` and `List<E>` by extending `AbstractList` (what do you get for free, and which methods must you override for performance?).
- Add `removeIf(Predicate<? super E>)` in a **single O(n) pass** (not O(n²) repeated shifting).
- Benchmark your `add`/`get` against `ArrayList` with the JMH setup from #018.

---

## Acceptance Criteria

- [ ] All methods in the API table, with the target complexities
- [ ] Capacity sequence identical to `ArrayList` (10, 15, 22, 33, 49, 73, 109, 163, 244)
- [ ] Exactly one justified unchecked cast
- [ ] Freed slots nulled, proven by the GC test
- [ ] Fail-fast iterator with a working `remove()`
- [ ] `hashCode()` equals `List.of(sameElements).hashCode()`
- [ ] Differential test passes
- [ ] Questions 3 and 4 answered

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
