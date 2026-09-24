# #019 — Read: Effective Java + Java Docs — Collections & Generics Reading Notes

**Date:** 2026-09-24  
**Roadmap row:** 19 — P1 Java Core · Collections & Generics · *Flex*  
**Sources:**
- Joshua Bloch, *Effective Java*, 3rd ed.: **Ch. 5 Generics (Items 26–33)**, plus **Item 54** (empty collections, not null), **Item 55** (optionals), **Item 58** (for-each), **Item 64** (refer to objects by their interfaces)
- JDK 21 Javadoc: `java.util` package summary, the `Collection`, `List`, `Set`, `Map`, `Iterator`, `RandomAccess` and `Collections` class docs, and the *Collections Framework Overview*

> Every code example marked ✔︎ was compiled and run on JDK 21.0.10 while writing these notes, including the compiler warnings shown.

---

## At a Glance (copy to handwritten notes)

- **Item 26:** never use **raw types** (`List`). Use `List<Object>` for "anything" or `List<?>` for "unknown type".
- **Item 27:** eliminate every **unchecked warning**. If one is truly safe, `@SuppressWarnings("unchecked")` on the **smallest scope**, with a comment saying why.
- **Item 28:** **prefer lists to arrays.** Arrays are covariant and **reified** (they fail at run time); generics are invariant and **erased** (they fail at compile time).
- **Item 29/30:** make your types and methods **generic**. Recursive bound: `<E extends Comparable<? super E>>`.
- **Item 31: PECS** (*Producer-Extends, Consumer-Super*): `<? extends T>` when you **read** from a parameter, `<? super T>` when you **write** into it. Never use wildcards in return types.
- **Item 32:** generic varargs are dangerous (**heap pollution**). Annotate *safe* ones with `@SafeVarargs`; never expose or return the varargs array.
- **Item 33:** **typesafe heterogeneous container**: `Map<Class<?>, Object>` + `type.cast(...)`.
- **Item 54:** return **empty collections**, never `null`. **Item 55:** `Optional` for return values only, never for collections, fields or map values.
- **Item 58:** prefer **for-each**. **Item 64:** declare variables, parameters and returns with **interface types** (`List`, `Map`).
- **Javadoc contract language:** "*optional operation*" = may throw `UnsupportedOperationException`; "*fail-fast*" = best-effort CME; "*RandomAccess*" = marker for O(1) `get(i)`.

---

## Part A — Effective Java, Chapter 5: Generics

### Terminology (Bloch's table)

| Term | Example |
|------|---------|
| Parameterized type | `List<String>` |
| Actual type parameter | `String` |
| Generic type | `List<E>` |
| Formal type parameter | `E` |
| Unbounded wildcard type | `List<?>` |
| Raw type | `List` |
| Bounded type parameter | `<E extends Number>` |
| Recursive type bound | `<T extends Comparable<T>>` |
| Bounded wildcard type | `List<? extends Number>` |
| Generic method | `static <E> List<E> asList(E[] a)` |
| Type token | `String.class` |

### Item 26 — Don't use raw types

```java
static void unsafeAdd(List list, Object o) { list.add(o); }   // raw type

List<String> strings = new ArrayList<>();
unsafeAdd(strings, Integer.valueOf(42));   // compiles (with a warning)!
String s = strings.get(0);                 // ✔︎ ClassCastException HERE, far from the actual bug
```

- Raw types exist only for **migration compatibility** with pre-Java-5 code.
- `List<Object>` ≠ `List`: you **can't** pass a `List<String>` to a `List<Object>` parameter (generics are invariant), so the compiler protects you. With a raw `List` it can't.
- For "a set of some unknown type", use **`Set<?>`**. You can read elements as `Object` but can't add anything except `null`, which is exactly the safety you want.
- Two legitimate raw-type uses: **class literals** (`List.class`, since `List<String>.class` is illegal) and **`instanceof`** (`o instanceof Set`; then cast to `Set<?>`).

### Item 27 — Eliminate unchecked warnings

Actual `javac -Xlint:unchecked` output for the examples in this note ✔︎:

```text
Gen.java:6: warning: [unchecked] unchecked call to add(E) as a member of the raw type List
    static void unsafeAdd(List list, Object o) { list.add(o); }
Gen.java:28: warning: [unchecked] Possible heap pollution from parameterized vararg type T
    static <T> T[] toArray(T... args) { return args; }
Gen.java:29: warning: [unchecked] unchecked generic array creation for varargs parameter of type T[]
```

- **Every** unchecked warning is a potential `ClassCastException` at run time. Fix them all.
- If you can prove a cast is safe, suppress on the **narrowest possible declaration** (even a local variable), never a whole class, and **comment why**:

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // This cast is correct because the array we're creating is of the same type as the one passed in, which is T[].
        @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    ...
}
```

### Item 28 — Prefer lists to arrays

| | Arrays | Generics |
|---|---|---|
| Variance | **Covariant**: `Long[]` *is a* `Object[]` | **Invariant**: `List<Long>` is *not* a `List<Object>` |
| Type info at run time | **Reified**: knows its element type | **Erased**: `List<String>` and `List<Integer>` are the same class at run time ✔︎ |
| Where type errors surface | **Run time** | **Compile time** |

```java
Object[] objs = new Long[1];
objs[0] = "I don't fit in";                  // ✔︎ ArrayStoreException at RUN time

List<Object> ol = new ArrayList<Long>();      // ❌ doesn't COMPILE: the error is caught earlier
```

Consequences:

- **Generic array creation is illegal:** `new List<E>[]`, `new List<String>[]` and `new E[]` don't compile. (If they did, you could smuggle a wrong type into a generic array.)
- Mixing arrays and generics gives warnings. Replace the array with a `List` and they go away, at a small performance cost that rarely matters.

```java
// Array version: unchecked warnings, may blow up
public class Chooser<T> { private final T[] choiceArray; ... (T[]) choices.toArray(); }
// List version: typesafe, no warnings
public class Chooser<T> {
    private final List<T> choiceList;
    public Chooser(Collection<T> choices) { choiceList = new ArrayList<>(choices); }
    public T choose() { return choiceList.get(ThreadLocalRandom.current().nextInt(choiceList.size())); }
}
```

### Item 29 — Favor generic types

Generify a class that stores `Object`s (Bloch's `Stack`). Because you can't do `new E[]`, there are two workarounds:

1. `elements = (E[]) new Object[DEFAULT_CAPACITY];`: one unchecked cast in the constructor (suppressed and justified), with `E[]` everywhere else. **Readable; the most common choice.**
2. Keep `Object[] elements` and cast each element on `pop()`: `E result = (E) elements[--size];`. Many casts, but no heap pollution.

`ArrayList` itself uses option 2: `transient Object[] elementData` and `E elementData(int index) { return (E) elementData[index]; }` (see #020).

Bounded type parameters restrict what's allowed: `class DelayQueue<E extends Delayed>`, so inside you can call `Delayed` methods on `E`.

### Item 30 — Favor generic methods

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {        // type parameter list before the return type
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

- **Generic singleton factory**: one immutable object reused for all types (`Collections.emptySet()`, `Function.identity()`), which works because the type is erased:

  ```java
  private static final UnaryOperator<Object> IDENTITY_FN = t -> t;
  @SuppressWarnings("unchecked")
  public static <T> UnaryOperator<T> identityFunction() { return (UnaryOperator<T>) IDENTITY_FN; }
  ```
- **Recursive type bound**: "a type comparable to itself":

  ```java
  public static <E extends Comparable<E>> E max(Collection<E> c)
  ```
  The improved version with PECS (Item 31) is below.

### Item 31 — Use bounded wildcards to increase API flexibility (PECS)

**Problem:** generics are invariant, so a `Stack<Number>.pushAll(Iterable<E>)` rejects an `Iterable<Integer>`, even though every `Integer` *is* a `Number`.

**PECS: Producer-Extends, Consumer-Super**

| The parameter… | Use | Mnemonic |
|----------------|-----|----------|
| **produces** `E`s that you **read** | `<? extends E>` | "gives me E or subtypes" |
| **consumes** `E`s that you **write** into it | `<? super E>` | "accepts E or supertypes" |
| does both | plain `E` | no wildcard |

```java
public void pushAll(Iterable<? extends E> src) { for (E e : src) push(e); }      // src PRODUCES
public void popAll(Collection<? super E> dst)  { while (!isEmpty()) dst.add(pop()); } // dst CONSUMES

Deque<Number> stack = new ArrayDeque<>();
pushAll(stack, List.of(1, 2, 3));           // ✔︎ Iterable<Integer> into Deque<Number>
List<Object> sink = new ArrayList<>();
popAll(stack, sink);                        // ✔︎ Deque<Number> into Collection<Object> → [3, 2, 1]
```

The JDK's canonical example:

```java
// java.util.Collections
public static <T> void copy(List<? super T> dest, List<? extends T> src)
//                              ↑ consumer                ↑ producer
Collections.copy(listOfObjects, List.of(1, 2, 3));          // ✔︎ works
```

**The fully general `max`:**

```java
public static <E extends Comparable<? super E>> E max(Collection<? extends E> c)
```

- `Collection<? extends E>`: the collection **produces** `E`s.
- `Comparable<? super E>`: a `Comparable` **consumes** `E`s (it takes them as the argument to `compareTo`). This lets `max` work for types that inherit `compareTo` from a supertype: `ScheduledFuture` is `Comparable<Delayed>`, not `Comparable<ScheduledFuture>`. ✔︎ Verified with a subclass that inherits `compareTo`.
- **Comparables and Comparators are always consumers** → `Comparator<? super T>` (e.g. `List.sort(Comparator<? super E>)`, `TreeMap(Comparator<? super K>)`).

More rules:

- **Don't use wildcards in return types.** They force callers to write wildcards too.
- If a type parameter appears **only once** in a method signature, replace it with a wildcard: `swap(List<?> list, int i, int j)` rather than `<E> swap(List<E> ...)`. Use a private **capture helper** (`swapHelper(List<E> ...)`) to implement it.
- If users have to think about wildcards to call your API, the API is wrong.

### Item 32 — Combine generics and varargs judiciously

A varargs parameter is an **array**, so generic varargs (`T... args`) create a generic array, which leads to **heap pollution** (a variable of a parameterized type refers to an object that isn't of that type).

```java
static <T> T[] toArray(T... args) { return args; }          // ⚠ exposes the varargs array
static <T> T[] pickTwo(T a, T b, T c) { return toArray(a, b); }

String[] attrs = pickTwo("Good", "Fast", "Cheap");            // ✔︎ ClassCastException:
// inside pickTwo, T is erased → the compiler creates an Object[] → cast to String[] fails
```

A generic varargs method is **safe** if it (1) **doesn't store** anything into the varargs array and (2) **doesn't let a reference to the array escape** to untrusted code. Then annotate it with **`@SafeVarargs`** (allowed on static, final and private methods), e.g. `List.of(E...)`, `Arrays.asList(T...)`, `EnumSet.of(E, E...)`. The alternative is to take a `List<? extends T>` instead of varargs.

### Item 33 — Consider typesafe heterogeneous containers

Normally generics fix the number of type parameters per container (`Map<K, V>`). To store values of **many different types** safely, **parameterize the key** instead:

```java
public final class Favorites {
    private final Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Objects.requireNonNull(type), type.cast(instance));  // cast guards against raw-type abuse
    }
    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));                              // dynamic cast, no unchecked warning
    }
}

f.putFavorite(String.class, "Java");
f.putFavorite(Integer.class, 0xcafebabe);
f.getFavorite(Integer.class);   // ✔︎ an Integer, no cast needed at the call site
```

- `Class<T>` used this way is a **type token**.
- Limitation: it can't hold **non-reifiable** types (`List<String>.class` doesn't exist). Spring's `ParameterizedTypeReference` and Jackson's `TypeReference` work around this with "super type tokens".
- Real-world uses: `AnnotatedElement.getAnnotation(Class<T>)`, Spring's `ApplicationContext.getBean(Class<T>)`, per-request attribute maps.

---

## Part B — Other Effective Java Items on Collections

### Item 54 — Return empty collections or arrays, not nulls

```java
// ❌ every caller must remember a null check
public List<Cheese> getCheeses() { return cheesesInStock.isEmpty() ? null : new ArrayList<>(cheesesInStock); }

// ✅
public List<Cheese> getCheeses() { return new ArrayList<>(cheesesInStock); }
// If allocation profiling proves it matters: Collections.emptyList() / List.of() (shared immutable instance)
// Arrays: return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);  with private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
```

Returning `null` gains nothing and costs every caller a null check (and a production NPE when one caller forgets).

### Item 55 — Return optionals judiciously

- `Optional<T>` = "a result may be absent". Use it for **return values** where absence is normal and callers must handle it: `stream().max(...)` returns `Optional` ✔︎ (`Optional.empty` for an empty list).
- **Never** return `Optional<List<T>>`; return an **empty list** (Item 54). The same goes for maps, streams, arrays and other containers.
- **Never** use `Optional` as a **field, a parameter, a map value or a collection element**: it's wasted allocation and awkward to use.
- Primitive versions avoid boxing: `OptionalInt`, `OptionalLong`, `OptionalDouble`.
- API: `orElse(default)`, `orElseGet(supplier)` (lazy), `orElseThrow()`, `map`, `filter`, `ifPresent`, `ifPresentOrElse`, `or`, `stream()`. Avoid `isPresent()` + `get()`.
- Example from #005: `Loan.returnedOn()` returns `Optional<LocalDate>`, while the **field** stays a plain nullable `LocalDate`.

### Item 58 — Prefer for-each loops to traditional for loops

- Hides the iterator and index variables, which are where bugs live (the classic nested-loop `i.next()` bug calls `next()` on the wrong iterator).
- Works on any `Iterable` and on arrays, with no performance penalty.
- **Three cases where you can't use it:** *destructive filtering* (use `removeIf`, see #017), *transforming* elements in place (you need `ListIterator.set` or an index), and *parallel iteration* of several collections in lockstep.

### Item 64 — Refer to objects by their interfaces

```java
Set<Son> sonSet = new LinkedHashSet<>();          // ✅ interface type
LinkedHashSet<Son> sonSet = new LinkedHashSet<>(); // ❌ ties every user to the implementation
```

This lets you switch `HashMap` → `LinkedHashMap` → `TreeMap` by changing **one line**. The exceptions are value classes (`String`, `BigInteger`), class-based frameworks, and implementations that offer extra methods you really need (e.g. `PriorityQueue.comparator()`, or `TreeMap.floorKey`, where `NavigableMap` is then the right interface).

---

## Part C — Java Docs: What the Collections Javadoc Actually Promises

The Javadoc **is the specification**. Interview answers and code reviews hinge on its exact wording.

### C1. The contract vocabulary

| Javadoc phrase | Meaning |
|----------------|---------|
| **"(optional operation)"** | An implementation may throw `UnsupportedOperationException` (unmodifiable, fixed-size or immutable collections) |
| **"unspecified"** / **"no guarantees as to the order"** | Don't depend on it: `HashMap`/`HashSet` order may change between runs, JDK versions or resizes |
| **"fail-fast"** | The iterator throws `ConcurrentModificationException` *on a best-effort basis*. *"Fail-fast iterators should be used only to detect bugs"*, never relied on for correctness |
| **"weakly consistent"** | Concurrent iterators: never throw CME; reflect some or all changes made after creation |
| **"structural modification"** | Adding or removing elements (for `ArrayList`, also explicitly resizing). Merely `set(i, e)` isn't structural. `LinkedHashMap` in access order: a `get` **is** structural |
| **"permits null"** / **"null elements are prohibited"** | Varies per implementation (see #017 table) |
| **"consistent with equals"** | For `SortedSet`/`SortedMap`: `compareTo == 0` must match `equals`, or the set violates the general `Set` contract |

### C2. `Collection` interface: key points

- `boolean add(E)` returns `true` if the collection **changed** (`Set.add` of a duplicate returns `false`).
- `contains`/`remove` use **`equals`** (`o == null ? e == null : o.equals(e)`). Hash-based implementations use `hashCode` first.
- `toArray()` returns `Object[]`. `toArray(T[] a)` returns an array of the runtime type. Java 11+: `toArray(String[]::new)`.
- **Bulk ops:** `addAll`, `removeAll`, `retainAll` (intersection), `containsAll`, `removeIf` (8+), `stream()`/`parallelStream()`.
- `equals`/`hashCode` on `Collection` itself aren't specified. **`List`** and **`Set`** specify them precisely: two lists are equal if they have the same elements in the same order, whatever the implementation (`new ArrayList<>(x).equals(new LinkedList<>(x))` is `true`). Sets are equal if they have the same elements.

### C3. `List`

- `hashCode` is defined as `31 * h + (e == null ? 0 : e.hashCode())` over the elements, the same recipe as EJ Item 11.
- `subList(from, to)` is a **view**: changes write through, and **structural changes to the parent invalidate it** (CME). `list.subList(10, 20).clear()` is the idiom for removing a range.
- `List.of` / `List.copyOf` (9/10+) are unmodifiable and null-hostile (#017).
- `sort(Comparator<? super E>)` (8+): pass `null` for natural order. It's a **stable** sort (TimSort for objects).

### C4. `Set`

- *"Great care must be exercised if mutable objects are used as set elements"*: exactly the mutable-key trap from #017.
- A set may not contain itself as an element.

### C5. `Map`

- Not a `Collection`. The **three views**, `keySet()`, `values()` and `entrySet()`, are *live views*: removing from them removes from the map, but you can't add through them.
- *"Great care must be exercised if mutable objects are used as map keys."*
- Default methods (8+): `getOrDefault`, `putIfAbsent`, `merge`, `compute`, `computeIfAbsent`, `computeIfPresent`, `replaceAll`, `forEach`. In `compute*` and `merge`, a mapping function returning **`null` removes the entry**.
- `Map.entry(k, v)` (9+) makes an immutable entry.

### C6. `RandomAccess` marker interface

*"Marker interface used by List implementations to indicate that they support fast (generally constant time) random access."* ✔︎ `ArrayList` implements it; `LinkedList` doesn't. Generic algorithms check it to choose a strategy. `Collections.binarySearch`, for example, uses index-based access for `RandomAccess` lists and iterator-based access otherwise:

```java
if (list instanceof RandomAccess || list.size() < BINARYSEARCH_THRESHOLD)
    return indexedBinarySearch(list, key);
else
    return iteratorBinarySearch(list, key);
```

The lesson: write index loops only for `RandomAccess` lists, or just use for-each.

### C7. `Collections` utility class: the useful methods

| Method | Purpose |
|--------|---------|
| `sort`, `binarySearch`, `reverse`, `shuffle`, `swap`, `rotate`, `frequency`, `disjoint`, `nCopies`, `max`/`min` | Algorithms |
| `emptyList/Set/Map`, `singletonList/Set/Map` | Shared immutable instances |
| `unmodifiableX` | Read-only **views** |
| `synchronizedX` | Thread-safe wrappers. You **must** still synchronise manually while iterating: `synchronized (list) { for (...) }` |
| `checkedX` | Runtime type-checked view, for debugging heap pollution from raw-type legacy code |
| `addAll(c, elems...)` | Varargs bulk add |
| `newSetFromMap(map)` | A `Set` from any `Map` (e.g. a concurrent set from a `ConcurrentHashMap`, now also `ConcurrentHashMap.newKeySet()`) |

### C8. `Iterator`

- `hasNext()`, `next()` (throws `NoSuchElementException` when exhausted), `remove()` (optional; at most once per `next()`, otherwise `IllegalStateException`).
- `ListIterator` adds `hasPrevious`/`previous`, `nextIndex`/`previousIndex`, `set`, and `add`. That's `LinkedList`'s one real strength (#018 B6).

---

## Part D — Generics Under the Hood: Type Erasure

- The compiler checks types, then **erases** them: `List<String>` becomes `List`, a type parameter `T` becomes its bound (`Object`, or `Comparable` for `<T extends Comparable<T>>`), and **casts are inserted** at use sites. ✔︎ `new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()` is `true`.
- **Bridge methods** are generated so overriding still works after erasure (e.g. `compareTo(Object)` delegating to `compareTo(MyType)`).
- Consequences: no `new T()`, no `new T[]`, no `instanceof List<String>`, no primitive type arguments (`List<int>`, hence boxing), and no overloads that differ only in type arguments (`m(List<String>)` + `m(List<Integer>)` clash after erasure).
- **Reifiable** types (full type info at run time): primitives, non-generic types, raw types, and `List<?>`. Only reifiable types are allowed in `instanceof` and array creation.

---

## Connections

| Idea | Where it's applied |
|------|--------------------|
| PECS | Row 26 (Generics/PECS), row 30 (generic container), `Comparator<? super T>` in row 27 |
| Item 29 generic `Stack`/`E[]` cast | Row 21 (own dynamic array), row 24 (simplified HashMap) |
| Item 33 type tokens | Spring `getBean(Class<T>)` in P2 |
| Items 54, 55 | Repository `findById` → `Optional<T>`, `findAll` → empty `List` (P2–P3) |
| `RandomAccess` | Why an index loop on `LinkedList` is O(n²) (#018 B8) |
| Fail-fast / structural modification | #017 §8, #020 `modCount` |

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| `List<Object>` vs `List<?>` vs `List`? | Holds any object and is typesafe / some unknown type (read-only-ish) / raw type with no checks, so avoid it |
| Why can't you create `new T[]`? | Erasure: at run time `T` is unknown, and arrays must know their element type (they're reified) |
| Is `List<Integer>` a subtype of `List<Number>`? | No, generics are invariant. `List<Integer>` **is** a subtype of `List<? extends Number>` |
| Explain PECS | Read from it → `extends`; write into it → `super`. `Collections.copy(List<? super T>, List<? extends T>)` |
| Why `Comparable<? super E>` in `max`? | So types inheriting `compareTo` from a supertype also qualify |
| What is heap pollution? | A parameterized-type variable referring to an object of the wrong type; caused by raw types, unchecked casts and generic varargs |
| What does `@SafeVarargs` promise? | The method doesn't store into or expose its generic varargs array |
| Return `null` or an empty list? | An empty list, always (Item 54) |
| `Optional` as a field? | No. Only for return values (Item 55) |
| What does "optional operation" mean in the Javadoc? | The implementation may throw `UnsupportedOperationException` |

---

## Handwritten Cheat Sheet

```
EJ CH5 GENERICS
26 no raw types: List<Object> (anything) / List<?> (unknown)
27 kill unchecked warnings; @SuppressWarnings on smallest scope + comment
28 lists > arrays: arrays covariant+reified (runtime fail), generics invariant+erased (compile fail)
   no new E[] / new List<String>[]
29 generic types: (E[]) new Object[n] + suppress, or cast per element
30 generic methods: <E> Set<E> union(...); recursive bound <E extends Comparable<? super E>>
31 PECS: Producer-Extends, Consumer-Super; Comparators are consumers
   Collections.copy(List<? super T> dst, List<? extends T> src)
   no wildcards in return types
32 generic varargs → heap pollution; @SafeVarargs if not stored/exposed
33 typesafe heterogeneous container: Map<Class<?>,Object> + type.cast()

54 return empty collection, never null
55 Optional for return values only (not fields/params/collections)
58 for-each; can't for removal (removeIf), in-place transform, lockstep
64 declare with interfaces (List/Map/Set)

JAVADOC: optional op → UOE | fail-fast = best effort | subList = view
         Map views live | RandomAccess = O(1) get | compareTo consistent w/ equals
ERASURE: List<String>.class == List<Integer>.class; bridge methods; no List<int>
```

---

**Next note:** #020 — ArrayList vs LinkedList; HashMap internals (buckets, resize, treeify)
