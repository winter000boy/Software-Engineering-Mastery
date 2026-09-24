# #021 — Solution: Implement Your Own Dynamic Array

**Date:** 2026-09-24  
**Problem:** [021-implement-dynamic-array.md](../problems/021-implement-dynamic-array.md)

> Verified on Java 21: **11/11 JUnit tests pass**, including a 50 000-operation differential test against `java.util.ArrayList`, and the code compiles with **zero warnings under `javac -Xlint:all`**. The code below is the verified code.

---

## 1. Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Storage | `Object[] elements` | `new E[n]` is illegal: type parameters are erased, and arrays need a runtime element type (#026 §8). Elements are cast back in **one** private method, `elementAt`, the only `@SuppressWarnings("unchecked")` in the class |
| Empty state | A shared static `EMPTY` array | Lazy allocation: an unused `DynamicArray` costs no array (the same idea as `ArrayList`'s `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`) |
| Growth | `max(minCapacity, old + old/2)`, first allocation 10 | The same sequence as `ArrayList`; computed in `long` so that `old + old/2` can't overflow near `Integer.MAX_VALUE` |
| Max size | `Integer.MAX_VALUE - 8` → `OutOfMemoryError` beyond it | Some JVMs reserve header words in arrays (`ArraysSupport.SOFT_MAX_ARRAY_LENGTH`) |
| Removal | `System.arraycopy` left, then **null the last slot** | O(n) shift; nulling prevents a memory leak (EJ Item 7) |
| Structural changes | `modCount++` in `add`, `add(i)`, `removeAt`, `remove`, `clear`, `ensureCapacity`, `trimToSize` | `set` isn't structural: it doesn't invalidate iterators (matches `ArrayList`) |
| Iterator | A **non-static** inner class `Itr` | It needs the enclosing array's fields. That's the one legitimate reason for a non-static member class (EJ Item 24) |
| `equals`/`hashCode` | The `List` recipe: `31 * h + hash(e)` | The hash equals `List.of(...)`'s for the same elements, so it can interoperate with `List`-based code |
| Class | `final` | Not designed for extension (EJ Item 19) |

---

## 2. Code

### `DynamicArray.java`

```java
package com.mastery.dynarray;

import java.util.Arrays;
import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.NoSuchElementException;
import java.util.Objects;

/**
 * A resizable array, the core of java.util.ArrayList, written from scratch.
 *
 * <p>Amortized O(1) {@code add(e)}; O(1) {@code get}/{@code set}; O(n) insert/remove by index.
 * Not thread-safe. Iterators are fail-fast. Permits {@code null} elements.
 *
 * @param <E> element type
 */
public final class DynamicArray<E> implements Iterable<E> {

    static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY = {};
    /** Largest array the JVM reliably allocates (same limit ArrayList uses). */
    static final int MAX_CAPACITY = Integer.MAX_VALUE - 8;

    private Object[] elements;     // Object[], not E[]: generic arrays can't be created (EJ Item 28)
    private int size;
    private int modCount;          // bumped on every structural change → fail-fast iterators

    /** Creates an empty array; storage is allocated lazily on the first add. */
    public DynamicArray() {
        this.elements = EMPTY;
    }

    public DynamicArray(int initialCapacity) {
        if (initialCapacity < 0) {
            throw new IllegalArgumentException("Negative capacity: " + initialCapacity);
        }
        this.elements = initialCapacity == 0 ? EMPTY : new Object[initialCapacity];
    }

    // ---- queries -----------------------------------------------------------------------

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    /** Current length of the backing array (for tests and learning; ArrayList hides this). */
    int capacity() {
        return elements.length;
    }

    public E get(int index) {
        Objects.checkIndex(index, size);
        return elementAt(index);
    }

    public int indexOf(Object o) {
        for (int i = 0; i < size; i++) {
            if (Objects.equals(o, elements[i])) {
                return i;
            }
        }
        return -1;
    }

    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    // ---- modification ------------------------------------------------------------------

    public E set(int index, E element) {
        Objects.checkIndex(index, size);
        E old = elementAt(index);
        elements[index] = element;    // not structural: modCount unchanged (same as ArrayList)
        return old;
    }

    /** Appends to the end. Amortized O(1). */
    public void add(E element) {
        modCount++;
        if (size == elements.length) {
            grow(size + 1);
        }
        elements[size++] = element;
    }

    /** Inserts at {@code index}, shifting later elements right. O(n - index). */
    public void add(int index, E element) {
        if (index < 0 || index > size) {                // index == size is allowed (append)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
        modCount++;
        if (size == elements.length) {
            grow(size + 1);
        }
        System.arraycopy(elements, index, elements, index + 1, size - index);
        elements[index] = element;
        size++;
    }

    /** Removes and returns the element at {@code index}, shifting later elements left. */
    public E removeAt(int index) {
        Objects.checkIndex(index, size);
        E old = elementAt(index);
        fastRemove(index);
        return old;
    }

    /** Removes the first occurrence of {@code o}; returns whether anything was removed. */
    public boolean remove(Object o) {
        int i = indexOf(o);
        if (i < 0) {
            return false;
        }
        fastRemove(i);
        return true;
    }

    public void clear() {
        modCount++;
        Arrays.fill(elements, 0, size, null);   // let GC reclaim the elements (EJ Item 7)
        size = 0;
    }

    /** Guarantees room for at least {@code minCapacity} elements without further growth. */
    public void ensureCapacity(int minCapacity) {
        if (minCapacity > elements.length) {
            modCount++;
            grow(minCapacity);
        }
    }

    /** Shrinks the backing array to exactly {@code size}. */
    public void trimToSize() {
        if (size < elements.length) {
            modCount++;
            elements = size == 0 ? EMPTY : Arrays.copyOf(elements, size);
        }
    }

    // ---- internals ---------------------------------------------------------------------

    @SuppressWarnings("unchecked")       // safe: only E's are ever stored via add/set
    private E elementAt(int index) {
        return (E) elements[index];
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0) {
            System.arraycopy(elements, index + 1, elements, index, numMoved);
        }
        elements[--size] = null;          // eliminate the obsolete reference
    }

    /** Grows to max(minCapacity, 1.5 × current), starting at DEFAULT_CAPACITY. */
    private void grow(int minCapacity) {
        if (minCapacity < 0 || minCapacity > MAX_CAPACITY) {   // int overflow shows up as negative
            throw new OutOfMemoryError("Required array length too large: " + minCapacity);
        }
        int oldCapacity = elements.length;
        int newCapacity;
        if (oldCapacity == 0) {
            newCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        } else {
            long preferred = (long) oldCapacity + (oldCapacity >> 1);   // long: can't overflow
            newCapacity = (int) Math.min(MAX_CAPACITY, Math.max(preferred, minCapacity));
        }
        elements = Arrays.copyOf(elements, newCapacity);
    }

    // ---- iteration ---------------------------------------------------------------------

    @Override
    public Iterator<E> iterator() {
        return new Itr();
    }

    /** Non-static inner class on purpose: it needs the enclosing array's state (EJ Item 24). */
    private final class Itr implements Iterator<E> {
        private int cursor;                   // index of the next element to return
        private int lastReturned = -1;        // index of the last element returned; -1 if none
        private int expectedModCount = modCount;

        @Override
        public boolean hasNext() {
            return cursor < size;
        }

        @Override
        public E next() {
            checkForComodification();
            if (cursor >= size) {
                throw new NoSuchElementException();
            }
            lastReturned = cursor++;
            return elementAt(lastReturned);
        }

        @Override
        public void remove() {
            if (lastReturned < 0) {
                throw new IllegalStateException("next() not called, or remove() already called");
            }
            checkForComodification();
            fastRemove(lastReturned);
            cursor = lastReturned;            // the next element shifted into this slot
            lastReturned = -1;
            expectedModCount = modCount;      // our own change is legitimate
        }

        private void checkForComodification() {
            if (modCount != expectedModCount) {
                throw new ConcurrentModificationException();
            }
        }
    }

    // ---- Object methods (same contract as java.util.List) ------------------------------

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof DynamicArray<?> other) || other.size != size) {
            return false;
        }
        for (int i = 0; i < size; i++) {
            if (!Objects.equals(elements[i], other.elements[i])) {
                return false;
            }
        }
        return true;
    }

    @Override
    public int hashCode() {
        int h = 1;
        for (int i = 0; i < size; i++) {
            h = 31 * h + Objects.hashCode(elements[i]);   // List.hashCode recipe
        }
        return h;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < size; i++) {
            if (i > 0) {
                sb.append(", ");
            }
            sb.append(elements[i] == this ? "(this)" : elements[i]);
        }
        return sb.append(']').toString();
    }
}
```

### Notes on the tricky parts

- **`add(int index, E e)` accepts `index == size`.** Inserting "just past the end" is an append. `index > size` is an error. That's why it can't simply reuse `Objects.checkIndex(index, size)`, which rejects `size`.
- **`Itr.remove()` resets `cursor = lastReturned`.** After removing element `i`, everything shifts left, so the element that was at `i + 1` is now at `i`, and it's the next one to return. Then `expectedModCount = modCount`, because the iterator's own removal is legitimate.
- **`toString` handles self-containment** (`a.add(a)`), printing `(this)` instead of recursing forever, just as `AbstractCollection.toString` does.
- **`clear()` nulls the slots** with `Arrays.fill`, rather than just setting `size = 0`, so the elements can be collected.

---

## 3. Tests

### `test: DynamicArrayTest.java`

```java
package com.mastery.dynarray;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.lang.ref.WeakReference;
import java.util.ArrayList;
import java.util.ConcurrentModificationException;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.Random;
import org.junit.jupiter.api.Test;

class DynamicArrayTest {

    @Test
    void startsEmptyAndAllocatesLazily() {
        DynamicArray<String> a = new DynamicArray<>();
        assertTrue(a.isEmpty());
        assertEquals(0, a.capacity());
        a.add("x");
        assertEquals(DynamicArray.DEFAULT_CAPACITY, a.capacity());
    }

    @Test
    void growsByOneAndAHalf() {
        DynamicArray<Integer> a = new DynamicArray<>();
        List<Integer> capacities = new ArrayList<>();
        for (int i = 0; i < 200; i++) {
            a.add(i);
            if (capacities.isEmpty() || capacities.get(capacities.size() - 1) != a.capacity()) {
                capacities.add(a.capacity());
            }
        }
        assertEquals(List.of(10, 15, 22, 33, 49, 73, 109, 163, 244), capacities);   // same as ArrayList (#020)
    }

    @Test
    void getAndSetWithBoundsChecks() {
        DynamicArray<String> a = new DynamicArray<>();
        a.add("a");
        a.add("b");
        assertEquals("b", a.get(1));
        assertEquals("b", a.set(1, "B"));
        assertEquals("B", a.get(1));
        assertThrows(IndexOutOfBoundsException.class, () -> a.get(2));
        assertThrows(IndexOutOfBoundsException.class, () -> a.get(-1));
    }

    @Test
    void insertAndRemoveShiftElements() {
        DynamicArray<String> a = new DynamicArray<>();
        a.add("a");
        a.add("c");
        a.add(1, "b");
        a.add(0, "start");
        a.add(a.size(), "end");                 // index == size appends
        assertEquals("[start, a, b, c, end]", a.toString());
        assertEquals("b", a.removeAt(2));
        assertTrue(a.remove("start"));
        assertFalse(a.remove("missing"));
        assertEquals("[a, c, end]", a.toString());
        assertThrows(IndexOutOfBoundsException.class, () -> a.add(4, "x"));
    }

    @Test
    void supportsNulls() {
        DynamicArray<String> a = new DynamicArray<>();
        a.add(null);
        a.add("x");
        assertTrue(a.contains(null));
        assertEquals(0, a.indexOf(null));
        assertTrue(a.remove(null));
        assertEquals("[x]", a.toString());
    }

    @Test
    void removedElementsBecomeGarbageCollectable() {
        DynamicArray<Object> a = new DynamicArray<>();
        Object payload = new Object();
        WeakReference<Object> ref = new WeakReference<>(payload);
        a.add(payload);
        a.removeAt(0);
        payload = null;
        for (int i = 0; i < 20 && ref.get() != null; i++) {
            System.gc();
        }
        assertNull(ref.get(), "array still referenced the removed object (EJ Item 7 leak)");
    }

    @Test
    void trimAndEnsureCapacity() {
        DynamicArray<Integer> a = new DynamicArray<>();
        a.ensureCapacity(1000);
        assertEquals(1000, a.capacity());
        a.add(1);
        a.trimToSize();
        assertEquals(1, a.capacity());
        a.clear();
        a.trimToSize();
        assertEquals(0, a.capacity());
    }

    @Test
    void iteratorVisitsAllAndSupportsRemove() {
        DynamicArray<Integer> a = new DynamicArray<>();
        for (int i = 0; i < 10; i++) {
            a.add(i);
        }
        for (Iterator<Integer> it = a.iterator(); it.hasNext(); ) {
            if (it.next() % 2 == 0) {
                it.remove();
            }
        }
        assertEquals("[1, 3, 5, 7, 9]", a.toString());
        Iterator<Integer> it = a.iterator();
        assertThrows(IllegalStateException.class, it::remove);
    }

    @Test
    void iteratorIsFailFast() {
        DynamicArray<Integer> a = new DynamicArray<>();
        a.add(1);
        a.add(2);
        assertThrows(ConcurrentModificationException.class, () -> {
            for (Integer x : a) {
                a.add(3);
            }
        });
        Iterator<Integer> exhausted = new DynamicArray<Integer>().iterator();
        assertThrows(NoSuchElementException.class, exhausted::next);
    }

    @Test
    void equalsAndHashCodeFollowListContract() {
        DynamicArray<String> a = new DynamicArray<>();
        DynamicArray<String> b = new DynamicArray<>(50);
        for (String s : List.of("x", "y")) {
            a.add(s);
            b.add(s);
        }
        assertEquals(a, b);                                           // capacity doesn't matter
        assertEquals(a.hashCode(), b.hashCode());
        assertEquals(List.of("x", "y").hashCode(), a.hashCode());     // same recipe as java.util.List
    }

    @Test
    void behavesLikeArrayListUnderRandomOperations() {
        Random rnd = new Random(7);
        DynamicArray<Integer> mine = new DynamicArray<>();
        List<Integer> reference = new ArrayList<>();
        for (int step = 0; step < 50_000; step++) {
            int op = rnd.nextInt(5);
            int value = rnd.nextInt(100);
            switch (op) {
                case 0, 1 -> { mine.add(value); reference.add(value); }
                case 2 -> {
                    int i = rnd.nextInt(reference.size() + 1);
                    mine.add(i, value);
                    reference.add(i, value);
                }
                case 3 -> {
                    if (!reference.isEmpty()) {
                        int i = rnd.nextInt(reference.size());
                        assertEquals(reference.remove(i), mine.removeAt(i));
                    }
                }
                default -> assertEquals(reference.remove(Integer.valueOf(value)), mine.remove(value));
            }
            assertEquals(reference.size(), mine.size());
        }
        assertEquals(reference.toString(), mine.toString());
    }
}
```

```text
$ mvn test -Dtest=DynamicArrayTest
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
```

The **GC test** (`removedElementsBecomeGarbageCollectable`) is the one that proves EJ Item 7. Comment out `elements[--size] = null` in `fastRemove` and it fails: the `WeakReference` never clears, because the array still points at the object.

The **differential test** is the strongest one. It doesn't check any specific scenario; it checks that across 50 000 random operations the array always behaves exactly like `ArrayList`. That catches edge cases you'd never think to write by hand, such as inserting at `size` right after a grow.

---

## 4. Q3: Why Is `add(e)` Amortized O(1)?

Most appends are O(1): write a slot, increment `size`. Occasionally an append triggers a grow that copies all `k` current elements. With ×1.5 growth, the copies happen at sizes 10, 15, 22, 33, … up to n. Their total is a **geometric series**:

```
total copies ≈ n × (1 + 2/3 + (2/3)² + …) = n × 1/(1 − 2/3) = 3n
```

So n appends cost about n writes + 3n copies = **O(n) total → O(1) per append, amortized**. ("Amortized" means averaged over a sequence of operations; a *single* append can still be O(n), which is a latency spike.) With ×2 growth the series is `n × (1 + 1/2 + 1/4 + …) = 2n`: fewer copies, the same Big-O.

If the array grew by a **constant** amount instead (say +10), the copies would be 10 + 20 + 30 + … ≈ n²/20, which is **O(n) per append**. The *multiplicative* factor is what makes it amortized O(1).

## 5. Q4: Why ×1.5 Rather Than ×2?

| | ×1.5 (`ArrayList`) | ×2 (e.g. many C++ `std::vector` implementations) |
|---|---|---|
| Worst-case wasted space right after a grow | ≤ 33% | ≤ 50% |
| Copies per element (amortized) | ~3 | ~2 |
| Number of grows to reach n | more (log₁.₅ n) | fewer (log₂ n) |
| Memory reuse | A factor below the golden ratio (~1.618) lets a later allocation fit in the combined space of earlier freed blocks, which helps some allocators. On the JVM, the GC compacts the heap anyway, so the main win is **less wasted memory** | Freed blocks are never large enough to reuse |

Java picks the memory-friendlier factor; the extra copying is cheap because `Arrays.copyOf` is a fast bulk copy.

---

## 6. Stretch Goal Sketches

**`AbstractList` version:** extend `AbstractList<E>` and implement `get`, `size`, `set`, `add(int, E)` and `remove(int)`. You then inherit `indexOf`, `contains`, `iterator`/`listIterator` (index-based), `subList`, `equals`/`hashCode`/`toString` and `removeIf`. Also implement `RandomAccess`. Override `indexOf`/`contains` and `clear` for speed (the inherited versions go through the iterator).

**Single-pass `removeIf`:**

```java
public boolean removeIf(Predicate<? super E> filter) {
    int w = 0;                                   // write index
    for (int r = 0; r < size; r++) {             // read index
        @SuppressWarnings("unchecked") E e = (E) elements[r];
        if (!filter.test(e)) {
            elements[w++] = e;                   // keep: compact toward the front
        }
    }
    boolean changed = w != size;
    Arrays.fill(elements, w, size, null);        // release the tail
    if (changed) { size = w; modCount++; }
    return changed;
}
```

That's O(n) in total, vs O(n²) for calling `removeAt` k times. `ArrayList.removeIf` uses the same compaction idea.

---

## Key Takeaways

1. A dynamic array = an `Object[]` + `size` + multiplicative growth. That's the whole trick behind `ArrayList`.
2. **Amortized O(1)** comes from growing by a *factor*, not by a constant.
3. `Object[]` + one narrowly suppressed cast is the standard way to store generic elements (EJ Item 29).
4. **Null out freed slots**, or the data structure leaks memory.
5. Fail-fast = `modCount` + `expectedModCount`. The iterator's own `remove` re-syncs them.
6. **Differential testing** against the JDK class you're imitating finds bugs example-based tests miss.
