# #030 — Solution: Write a Generic Container Class

**Date:** 2026-09-24  
**Problem:** [030-generic-container-class.md](../problems/030-generic-container-class.md)

> Verified on Java 21: **10/10 JUnit tests pass** (including a parallel-stream collector test over 10 000 elements), and the code compiles with **zero warnings under `javac -Xlint:all`**. It contains exactly one narrowly scoped, justified unchecked cast.

---

## 1. Code

### `Multiset.java`

```java
package com.mastery.container;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.NoSuchElementException;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.TreeMap;
import java.util.function.Function;
import java.util.stream.Collector;

/**
 * A generic multiset ("bag"): a collection that counts how many times each element occurs.
 *
 * <p>Backed by a {@code Map<E, Integer>}; the factory method chooses the ordering
 * (hash = unordered, linked = insertion order, sorted = by comparator). Null elements are
 * rejected (sorted maps can't hold them, and a null count would be ambiguous).
 *
 * @param <E> element type
 */
public final class Multiset<E> implements Iterable<E> {

    /** One distinct element and its count. A generic record: immutable, value-equal. */
    public record Entry<E>(E element, int count) {
        public Entry {
            Objects.requireNonNull(element, "element");
            if (count <= 0) {
                throw new IllegalArgumentException("count must be positive: " + count);
            }
        }
    }

    private final Map<E, Integer> counts;
    private int size;             // total occurrences, kept in sync so size() is O(1)
    private int modCount;

    private Multiset(Map<E, Integer> counts) {
        this.counts = counts;
    }

    // ---- static generic factories (EJ Items 1 and 30) ------------------------------------

    /** Unordered, O(1) operations. The type parameter is inferred from the target: Multiset<String> m = create(). */
    public static <E> Multiset<E> create() {
        return new Multiset<>(new HashMap<>());
    }

    /** Iterates in first-insertion order. */
    public static <E> Multiset<E> linked() {
        return new Multiset<>(new LinkedHashMap<>());
    }

    /**
     * Iterates in natural order. The bound says: E must be comparable to itself OR to a
     * supertype, e.g. java.sql.Timestamp is Comparable<Date>, not Comparable<Timestamp>.
     */
    public static <E extends Comparable<? super E>> Multiset<E> sorted() {
        return new Multiset<>(new TreeMap<>());
    }

    /** Iterates in comparator order. {@code ? super E}: a Comparator<Object> works for any E. */
    public static <E> Multiset<E> sorted(Comparator<? super E> comparator) {
        return new Multiset<>(new TreeMap<>(Objects.requireNonNull(comparator)));
    }

    /** Safe varargs: the array is only read, never stored or exposed (EJ Item 32). */
    @SafeVarargs
    public static <E> Multiset<E> of(E... elements) {
        Multiset<E> m = create();
        for (E e : elements) {
            m.add(e);
        }
        return m;
    }

    /** {@code ? extends E}: copy from any source that PRODUCES E's (a List<Integer> into Multiset<Number>). */
    public static <E> Multiset<E> copyOf(Iterable<? extends E> source) {
        Multiset<E> m = create();
        m.addAll(source);
        return m;
    }

    // ---- counting ----------------------------------------------------------------------

    /** Adds one occurrence; returns the count before the add. */
    public int add(E element) {
        return add(element, 1);
    }

    /** Adds {@code occurrences} (≥ 0) copies; returns the previous count. */
    public int add(E element, int occurrences) {
        Objects.requireNonNull(element, "element");
        if (occurrences < 0) {
            throw new IllegalArgumentException("occurrences must be >= 0: " + occurrences);
        }
        int previous = count(element);
        if (occurrences == 0) {
            return previous;
        }
        long updated = (long) previous + occurrences;
        if (updated > Integer.MAX_VALUE) {
            throw new IllegalArgumentException("count would overflow int");
        }
        counts.put(element, (int) updated);
        size += occurrences;
        modCount++;
        return previous;
    }

    /** Removes one occurrence if present; returns whether anything changed. */
    public boolean remove(Object element) {
        return remove(element, 1) > 0;
    }

    /**
     * Removes up to {@code occurrences} copies; returns the count before removal.
     * Takes Object, like Collection.remove: asking to remove an Integer from a
     * Multiset<String> is legal and simply finds nothing.
     */
    public int remove(Object element, int occurrences) {
        if (occurrences < 0) {
            throw new IllegalArgumentException("occurrences must be >= 0: " + occurrences);
        }
        int previous = count(element);
        if (previous == 0 || occurrences == 0) {
            return previous;
        }
        @SuppressWarnings("unchecked")   // safe: previous > 0 means element IS a key, so it's an E
        E key = (E) element;
        int removed = Math.min(previous, occurrences);
        if (removed == previous) {
            counts.remove(key);           // never store zero counts
        } else {
            counts.put(key, previous - removed);
        }
        size -= removed;
        modCount++;
        return previous;
    }

    /** Number of occurrences of {@code element} (0 if absent). */
    public int count(Object element) {
        if (element == null) {
            return 0;
        }
        try {
            return counts.getOrDefault(element, 0);
        } catch (ClassCastException e) {   // TreeMap.get with an incomparable type
            return 0;
        }
    }

    public boolean contains(Object element) {
        return count(element) > 0;
    }

    /** Total occurrences. */
    public int size() {
        return size;
    }

    public int distinctCount() {
        return counts.size();
    }

    public boolean isEmpty() {
        return size == 0;
    }

    // ---- PECS in both directions --------------------------------------------------------

    /** Producer → {@code ? extends E}. */
    public void addAll(Iterable<? extends E> source) {
        for (E e : source) {
            add(e);
        }
    }

    /** Consumer → {@code ? super E}: drains into a Collection<Object>, Collection<Number>, ... */
    public void drainTo(Collection<? super E> sink) {
        for (E e : this) {
            sink.add(e);
        }
        counts.clear();
        size = 0;
        modCount++;
    }

    // ---- views and queries -----------------------------------------------------------------

    /** The distinct elements, as an unmodifiable live view in this multiset's order. */
    public Set<E> elementSet() {
        return Collections.unmodifiableSet(counts.keySet());
    }

    /** All (element, count) pairs, in this multiset's order. A snapshot. */
    public List<Entry<E>> entries() {
        List<Entry<E>> result = new ArrayList<>(counts.size());
        counts.forEach((e, c) -> result.add(new Entry<>(e, c)));
        return result;
    }

    /** The n most frequent elements; ties keep the multiset's own iteration order (stable sort). */
    public List<Entry<E>> mostCommon(int n) {
        List<Entry<E>> all = entries();
        all.sort(Comparator.comparingInt(Entry<E>::count).reversed());
        return all.subList(0, Math.min(n, all.size()));
    }

    /**
     * Largest element. Static generic method with the full PECS signature (like Collections.max):
     * the multiset PRODUCES T's ({@code ? extends T}), the Comparable CONSUMES them ({@code ? super T}).
     */
    public static <T extends Comparable<? super T>> Optional<T> max(Multiset<? extends T> multiset) {
        T best = null;
        for (T t : multiset.elementSet()) {
            if (best == null || t.compareTo(best) > 0) {
                best = t;
            }
        }
        return Optional.ofNullable(best);
    }

    // ---- stream integration --------------------------------------------------------------------

    /** {@code stream.collect(Multiset.toMultiset())} */
    public static <T> Collector<T, ?, Multiset<T>> toMultiset() {
        return toMultiset(Function.identity());
    }

    /** Counts a derived key: {@code words.collect(toMultiset(String::length))}. */
    public static <T, K> Collector<T, ?, Multiset<K>> toMultiset(Function<? super T, ? extends K> keyFn) {
        return Collector.of(
                Multiset::create,                                   // supplier
                (m, t) -> m.add(keyFn.apply(t)),                    // accumulator
                (a, b) -> {                                         // combiner (parallel streams)
                    b.counts.forEach(a::add);
                    return a;
                });
    }

    // ---- iteration: every occurrence, element by element ---------------------------------------

    @Override
    public Iterator<E> iterator() {
        return new Iterator<>() {
            private final Iterator<Map.Entry<E, Integer>> entries = counts.entrySet().iterator();
            private final int expectedModCount = modCount;
            private E current;
            private int remaining;

            @Override
            public boolean hasNext() {
                return remaining > 0 || entries.hasNext();
            }

            @Override
            public E next() {
                if (modCount != expectedModCount) {
                    throw new java.util.ConcurrentModificationException();
                }
                if (remaining == 0) {
                    if (!entries.hasNext()) {
                        throw new NoSuchElementException();
                    }
                    Map.Entry<E, Integer> e = entries.next();
                    current = e.getKey();
                    remaining = e.getValue();
                }
                remaining--;
                return current;
            }
        };
    }

    // ---- Object methods: equal if same elements with same counts, regardless of ordering ---------

    @Override
    public boolean equals(Object o) {
        return this == o
                || (o instanceof Multiset<?> other
                    && size == other.size
                    && new HashMap<>(counts).equals(new HashMap<>(other.counts)));   // order-independent
    }

    @Override
    public int hashCode() {
        return counts.hashCode();   // Map.hashCode = sum of (key ^ value) hashes: order-independent, like equals
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("[");
        counts.forEach((e, c) -> {
            if (sb.length() > 1) {
                sb.append(", ");
            }
            sb.append(e);
            if (c > 1) {
                sb.append(" x ").append(c);
            }
        });
        return sb.append(']').toString();
    }
}
```

---

## 2. Every Wildcard and Bound, Justified

| Signature | Role | What fails without it |
|-----------|------|-----------------------|
| `static <E extends Comparable<? super E>> Multiset<E> sorted()` | The bound guarantees the `TreeMap` can compare elements. `? super E` because the elements **consume** themselves in `compareTo` | Without the bound, a non-Comparable `E` compiles and then throws `ClassCastException` on the first `add`. With `Comparable<E>` instead of `? super E`, `Multiset.<Timestamp>sorted()` wouldn't compile (`Timestamp` is `Comparable<Date>`) |
| `static <E> Multiset<E> sorted(Comparator<? super E> c)` | The comparator **consumes** `E`s | You couldn't pass a `Comparator<Object>` (`comparing(Object::toString)`) to a `Multiset<Integer>`, which the test does |
| `@SafeVarargs static <E> Multiset<E> of(E... elements)` | Generic varargs; safe because the array is **only read**, never stored or returned (EJ Item 32) | Without the annotation, every caller gets an *unchecked generic array creation* warning |
| `static <E> Multiset<E> copyOf(Iterable<? extends E> source)` | The source **produces** `E`s | `Multiset<Number> copy = copyOf(List.of(1, 2, 2))` wouldn't compile |
| `void addAll(Iterable<? extends E> source)` | Producer | `numbers.addAll(listOfIntegers)` wouldn't compile |
| `void drainTo(Collection<? super E> sink)` | Consumer | You couldn't drain a `Multiset<Number>` into a `List<Object>` |
| `static <T extends Comparable<? super T>> Optional<T> max(Multiset<? extends T> m)` | The multiset **produces** `T`s; the `Comparable` **consumes** them (same shape as `Collections.max`) | Without `? super T`, `max(timestamps)` fails. Without `? extends T`, `Optional<Date> d = max(timestamps)` fails |
| `static <T, K> Collector<T, ?, Multiset<K>> toMultiset(Function<? super T, ? extends K> keyFn)` | A function **consumes** `T` and **produces** `K` | You couldn't use a `Function<Object, Integer>` (like `Object::hashCode`) on a `Stream<String>` |
| `Collector<T, ?, Multiset<T>>` (the middle `?`) | The accumulator type is an implementation detail hidden from callers | Nothing breaks, but callers would see an internal type |
| `boolean remove(Object o)`, `int count(Object o)` | `Object`, not `E`, like `Collection.remove`/`contains` | Callers holding an `Object` (e.g. from a heterogeneous list) would have to cast first |
| `record Entry<E>(E element, int count)` | A generic **record**: immutable, value-equal, validated in the compact constructor | — |

### The one unchecked cast

```java
@SuppressWarnings("unchecked")   // safe: previous > 0 means element IS a key, so it's an E
E key = (E) element;
```

`remove(Object)` needs an `E` to call `counts.put(key, …)`. The cast is only reached when `count(element) > 0`, which means `element` is already a key of `Map<E, Integer>`, so it really is an `E`. The suppression sits on a **local variable declaration**, the narrowest scope possible (EJ Item 27).

### Why reject `null` elements (Task 4)

1. `sorted()` multisets use a `TreeMap`, which throws `NullPointerException` on `null` keys with natural ordering. Allowing `null` in some factories but not others would make the type behave inconsistently.
2. `count(null) == 0` is unambiguous only if `null` can never be stored.
3. Guava's `Multiset` and the immutable `List.of`/`Set.of` make the same choice: **null elements are almost always a bug**.

---

## 3. Design Notes

- **The factory decides the ordering**, and the class stays `final` with a private constructor. That's EJ Item 1 (static factories with names that explain the behaviour) plus Item 17 (no subclassing surprises).
- **`size` is a cached field**, updated on every add and remove, so `size()` is O(1) instead of summing all counts.
- **Zero counts are never stored**, so `elementSet()` and `distinctCount()` are always right without filtering.
- **`count(foreignType)` catches `ClassCastException`.** On a `TreeMap`, `get(42)` in a map of `String` keys would throw, because the comparator casts. `Collection.contains` is specified to return `false` for "wrong type" arguments (or optionally throw); returning 0 is the friendlier choice.
- **The collector's combiner** merges `b` into `a` with `b.counts.forEach(a::add)`. That's needed for **parallel streams**, where each thread builds its own partial multiset. The test runs 10 000 elements in parallel and checks the exact counts (`1429` zeros for `i % 7`).
- **`equals` is order-independent.** A linked and a sorted multiset with the same counts are equal, and `hashCode` is `counts.hashCode()`, which (per `Map`'s contract) is the sum of the entry hashes and so is also order-independent.
- **Iteration yields each element `count` times** (like Guava), so `for (E e : multiset)` sees the multiset as a collection of occurrences, and `forEach(list::add)` reproduces them.

---

## 4. Tests

### `test: MultisetTest.java`

```java
package com.mastery.container;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.sql.Timestamp;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.ConcurrentModificationException;
import java.util.Date;
import java.util.List;
import java.util.Optional;
import java.util.stream.Stream;
import org.junit.jupiter.api.Test;

class MultisetTest {

    @Test
    void countsOccurrences() {
        Multiset<String> m = Multiset.of("a", "b", "a", "c", "a");
        assertEquals(3, m.count("a"));
        assertEquals(0, m.count("zzz"));
        assertEquals(5, m.size());
        assertEquals(3, m.distinctCount());
        assertEquals(3, m.add("a", 2));          // returns previous count
        assertEquals(5, m.count("a"));
        assertEquals(5, m.remove("a", 3));
        assertEquals(2, m.count("a"));
        assertEquals(2, m.remove("a", 99));      // removes only what exists
        assertFalse(m.contains("a"));
        assertEquals(List.of("b", "c"), m.elementSet().stream().sorted().toList());   // no zero-count keys left
    }

    @Test
    void rejectsNullAndNegative() {
        Multiset<String> m = Multiset.create();
        assertThrows(NullPointerException.class, () -> m.add(null));
        assertThrows(IllegalArgumentException.class, () -> m.add("x", -1));
        assertThrows(IllegalArgumentException.class, () -> m.remove("x", -1));
        assertEquals(0, m.count(null));
        assertThrows(IllegalArgumentException.class, () -> new Multiset.Entry<>("x", 0));
    }

    @Test
    void orderingDependsOnFactory() {
        List<String> input = List.of("pear", "apple", "fig", "apple");
        Multiset<String> linked = Multiset.linked();
        Multiset<String> sorted = Multiset.sorted();
        Multiset<String> byLength = Multiset.sorted(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()));
        linked.addAll(input);
        sorted.addAll(input);
        byLength.addAll(input);
        assertEquals("[pear, apple x 2, fig]", linked.toString());
        assertEquals("[apple x 2, fig, pear]", sorted.toString());
        assertEquals("[fig, pear, apple x 2]", byLength.toString());
        assertEquals(linked, sorted);            // equality ignores ordering
        assertEquals(linked.hashCode(), sorted.hashCode());
    }

    @Test
    void pecsProducerAndConsumer() {
        Multiset<Number> numbers = Multiset.create();
        List<Integer> ints = List.of(1, 2, 2);
        List<Double> doubles = List.of(3.5);
        numbers.addAll(ints);                    // Iterable<Integer> → Multiset<Number>: ? extends E
        numbers.addAll(doubles);
        assertEquals(4, numbers.size());

        Multiset<Number> copy = Multiset.copyOf(ints);   // T inferred as Number from the target type
        assertEquals(2, copy.count(2));

        List<Object> sink = new ArrayList<>();
        numbers.drainTo(sink);                   // Multiset<Number> → Collection<Object>: ? super E
        assertEquals(4, sink.size());
        assertTrue(numbers.isEmpty());
    }

    @Test
    void comparatorOfSupertypeIsAccepted() {
        Comparator<Object> byToString = Comparator.comparing(Object::toString);
        Multiset<Integer> m = Multiset.sorted(byToString);     // Comparator<? super Integer>
        m.addAll(List.of(10, 9, 100));
        assertEquals("[10, 100, 9]", m.toString());            // string order, not numeric
    }

    @Test
    void maxWorksForTypesComparableOnlyToASupertype() {
        // java.sql.Timestamp extends Date and is Comparable<Date>, NOT Comparable<Timestamp>.
        // Without <T extends Comparable<? super T>> this call would not compile.
        Multiset<Timestamp> times = Multiset.create();
        times.add(new Timestamp(1_000));
        times.add(new Timestamp(5_000));
        times.add(new Timestamp(3_000));
        Optional<Timestamp> latest = Multiset.max(times);
        assertEquals(new Timestamp(5_000), latest.orElseThrow());
        Optional<Date> asDate = Multiset.max(times);            // Multiset<? extends T> with T = Date
        assertTrue(asDate.isPresent());
        assertEquals(Optional.empty(), Multiset.max(Multiset.<String>create()));
    }

    @Test
    void mostCommonIsStableForTies() {
        Multiset<String> m = Multiset.linked();
        m.addAll(List.of("b", "a", "b", "c", "a", "d"));
        assertEquals(List.of(new Multiset.Entry<>("b", 2), new Multiset.Entry<>("a", 2), new Multiset.Entry<>("c", 1)),
                m.mostCommon(3));
    }

    @Test
    void collectorsFromStreams() {
        Multiset<String> words = Stream.of("to", "be", "or", "not", "to", "be").collect(Multiset.toMultiset());
        assertEquals(2, words.count("to"));
        Multiset<Integer> lengths = Stream.of("to", "be", "or", "not", "to", "be").collect(Multiset.toMultiset(String::length));
        assertEquals(5, lengths.count(2));
        assertEquals(1, lengths.count(3));
        Multiset<Integer> parallel = Stream.iterate(0, i -> i + 1).limit(10_000).parallel()
                .collect(Multiset.toMultiset(i -> i % 7));      // exercises the combiner
        assertEquals(10_000, parallel.size());
        assertEquals(1429, parallel.count(0));
    }

    @Test
    void iteratesEveryOccurrenceAndIsFailFast() {
        Multiset<String> m = Multiset.sorted();
        m.add("x", 3);
        m.add("y");
        List<String> seen = new ArrayList<>();
        m.forEach(seen::add);
        assertEquals(List.of("x", "x", "x", "y"), seen);
        assertThrows(ConcurrentModificationException.class, () -> {
            for (String s : m) {
                m.add("z");
            }
        });
    }

    @Test
    void removeWithForeignTypeFindsNothing() {
        Multiset<String> sorted = Multiset.sorted();
        sorted.add("a");
        assertEquals(0, sorted.count(42));       // TreeMap would throw ClassCastException; we return 0
        assertFalse(sorted.remove(42));
    }
}
```

```text
$ mvn test -Dtest=MultisetTest
Tests run: 10, Failures: 0, Errors: 0, Skipped: 0
```

---

## 5. Project `pom.xml` (all week-2 builds: rows 21, 22, 24, 25, 27, 30)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>week2</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.11.4</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
            </plugin>
        </plugins>
    </build>
</project>
```

Layout: `src/main/java/com/mastery/{dynarray, exercises, hashmap, sorting, container}` and the matching test packages.

```text
$ mvn test
Tests run: 55, Failures: 0, Errors: 0, Skipped: 0
```

---

## Key Takeaways

1. **Static generic factories** give each variant a name (`create`, `linked`, `sorted`) and let the compiler infer `E` from the target.
2. **PECS everywhere:** read-only inputs `? extends E`, write-only outputs `? super E`, comparators and functions `? super` in, `? extends` out.
3. **`<T extends Comparable<? super T>>`** is the correct "sortable" bound; `Timestamp` proves why.
4. **Custom `Collector`s** need a correct **combiner** for parallel streams, so test in parallel.
5. One unchecked cast, provably safe, suppressed on a single local variable. Everything else is fully type-checked.
