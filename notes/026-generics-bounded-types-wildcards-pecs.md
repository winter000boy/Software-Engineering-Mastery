# #026 — Generics, Bounded Types, Wildcards (PECS)

**Date:** 2026-09-24  
**Roadmap row:** 26 — P1 Java Core · Collections & Generics · *Learn*  
**Sources:** JLS §4.5 (parameterized types), §4.10.2 (subtyping), §5.1.10 (capture conversion), §18 (inference); *Effective Java* Ch. 5 (summarised in #019)  
**Relation to #019:** #019 is the *Effective Java* reading notes (the rules). This note is the **mechanics**: how the compiler reasons, what the errors mean, and the patterns to use.

> Every error message below is **real `javac` 21 output** from compiling the snippet shown. Every ✔︎ result was produced by running the code.

---

## At a Glance (copy to handwritten notes)

- **Generics = compile-time type checking + automatic casts**, then **erased** (`List<String>` → `List`, `T` → its bound).
- **Type parameter** `<T>` = a name you **declare** (on a class or method). **Wildcard** `?` = an unknown type you **use** (in a parameter or variable type).
- **Invariance:** `List<Integer>` is **not** a `List<Number>` ✔︎. Wildcards add controlled variance:
  - `List<? extends Number>` accepts `List<Integer>`, `List<Double>`: you can **read** `Number`s, you **can't add** ✔︎.
  - `List<? super Integer>` accepts `List<Integer>`, `List<Number>`, `List<Object>`: you can **add** `Integer`s, but reads come back as `Object` ✔︎.
- **PECS: Producer `extends`, Consumer `super`.** Comparators and Comparables are consumers → `Comparator<? super T>`.
- **Bounds:** `<T extends Number>` (upper bound; `extends` also means "implements"), `<T extends Number & Comparable<? super T>>` (multiple: the class first, then interfaces). Type parameters have **no `super` bound**.
- **Recursive bound:** `<T extends Comparable<? super T>>` ("comparable to itself or a supertype"); `Builder<T extends Builder<T>>` (self-type).
- **Capture:** the compiler names each `?` `CAP#1` ✔︎. Use a **private generic helper** to write through a `List<?>`.
- **Can't:** `new T()`, `new T[]`, `new List<String>[]`, `instanceof List<String>`, `List<int>`, static fields of type `T`, overloads that differ only in type arguments ✔︎ (all compile errors below).
- **Bridge methods:** `compareTo(Object)` is generated to call `compareTo(Money)` after erasure ✔︎ (`javap`).

---

## 1. Vocabulary

```java
class Box<T> { ... }                        // T = type PARAMETER (declared)
Box<String> b;                              // String = type ARGUMENT; Box<String> = parameterized type
Box<?> any;                                 // ? = WILDCARD (unknown type argument)
Box<? extends Number> producer;             // upper-bounded wildcard
Box<? super Integer> consumer;              // lower-bounded wildcard
static <E> List<E> listOf(E e) { ... }      // generic METHOD: <E> declared before the return type
class NumBox<T extends Number> { ... }      // BOUNDED type parameter
Box raw;                                    // RAW type: no type argument, no checking (legacy only)
```

| You want… | Syntax |
|-----------|--------|
| A class/interface that works for many types | `class Box<T>` |
| A method that works for many types | `static <T> void m(T t)` |
| To restrict what `T` can be | `<T extends Bound>` |
| A **parameter** that accepts many parameterizations | a wildcard: `List<? extends T>`, `List<? super T>`, `List<?>` |
| Two parameters (or a parameter and the return type) that must be the **same** type | a type parameter: `<T> T first(List<T> l)` |

**Rule of thumb:** type parameters express *relationships* between types in a signature; wildcards express *flexibility* for a single parameter.

---

## 2. Invariance, and Why It Protects You

```java
List<Integer> ints = new ArrayList<>();
List<Number> nums = ints;
```
```text
error: incompatible types: List<Integer> cannot be converted to List<Number>
```

If this were allowed, `nums.add(3.14)` would put a `Double` into `ints`. Arrays **are** covariant, and that exact bug becomes a runtime `ArrayStoreException` (#019, Item 28). Generics refuse at compile time.

### The subtyping picture

```
                        List<?>                      (any List)
                   ┌───────┴────────┐
        List<? extends Number>   List<? super Integer>
           ┌───────┴───────┐        ┌────────┴────────┐
    List<Integer>   List<Double>  List<Number>   List<Object>
          │                            │
          └── also a List<? super Integer> ┘ (List<Integer> fits both families)

ArrayList<Integer> <: List<Integer> <: Collection<Integer>   (the class hierarchy still works)
List<Integer> is NOT <: List<Number>                          (type arguments are invariant)
```

---

## 3. Wildcards: What You Can and Can't Do

### `? extends T`: read as `T`, can't write

```java
void m(List<? extends Number> nums) {
    Number n = nums.get(0);          // ✅ whatever it is, it's some Number
    nums.add(Integer.valueOf(1));    // ❌
}
```
```text
error: incompatible types: Integer cannot be converted to CAP#1
    CAP#1 extends Number from capture of ? extends Number
```

The compiler doesn't know *which* subtype of `Number` the list holds (it could be a `List<Double>`), so it can't allow any `add`. The only thing you can add is `null`.

### `? super T`: write `T`, reads give `Object`

```java
void m(List<? super Integer> sink) {
    sink.add(42);                    // ✅ whatever it is, it can hold an Integer
    Integer i = sink.get(0);         // ❌
    Object o = sink.get(0);          // ✅
}
```
```text
error: incompatible types: CAP#1 cannot be converted to Integer
    CAP#1 extends Object super: Integer from capture of ? super Integer
```

### `?`: read as `Object`, can't write (except `null`)

`List<?>` means "a list of some specific type I don't know". It's different from `List<Object>`, which means "a list that can hold anything":

| | `List<Object>` | `List<?>` | raw `List` |
|---|---|---|---|
| Accepts `List<String>` as an argument | ❌ | ✅ | ✅ (warning) |
| `add("x")` | ✅ | ❌ | ✅ (unchecked!) |
| `get(0)` type | `Object` | `Object` | `Object` |
| Type-safe | ✅ | ✅ | ❌ |

### The PECS table

| Parameter role | Wildcard | Example in the JDK |
|----------------|----------|--------------------|
| **P**roducer: you read `T`s from it | `? extends T` | `Collection.addAll(Collection<? extends E>)`, `new ArrayList<>(Collection<? extends E>)` |
| **C**onsumer: you write `T`s into it | `? super T` | `Collections.fill(List<? super T>, T)`, `Consumer<? super T>` |
| Both | `T` (no wildcard) | `Collections.swap` (implemented via a helper), `List.replaceAll` |
| Comparator / Comparable (always consumers) | `? super T` | `List.sort(Comparator<? super E>)`, `TreeMap(Comparator<? super K>)` |
| Function: consumes its input, produces its output | `Function<? super T, ? extends R>` | `Stream.map`, `Optional.map`, `Map.computeIfAbsent` |

```java
// java.util.Collections, a perfect PECS signature
public static <T> void copy(List<? super T> dest, List<? extends T> src)
//                            consumer ↑               producer ↑
```

In the #030 `Multiset<E>`: `addAll(Iterable<? extends E>)` (a producer), `drainTo(Collection<? super E>)` (a consumer), `sorted(Comparator<? super E>)` (a consumer). The tests put a `List<Integer>` into a `Multiset<Number>` and drain a `Multiset<Number>` into a `List<Object>` ✔︎.

**Don't use wildcards in return types.** `List<? extends Number> getNums()` forces every caller to deal with the wildcard. Return `List<Number>` or a type parameter.

---

## 4. Capture Conversion and the Helper Pattern

Each time an expression of wildcard type is used, the compiler **captures** the `?` as a fresh, unnamed type (`CAP#1`). Two captures of the same `?` are **different** types:

```java
static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));    // ❌
}
```
```text
error: incompatible types: Object cannot be converted to CAP#1
    CAP#1 extends Object from capture of ?
```

`list.get(i)` returns `CAP#1`, seen as `Object`, and `set` needs a `CAP#1`. The compiler can't prove they're the same. **Fix: a private generic helper** gives the captured type a name:

```java
public static void swap(List<?> list, int i, int j) {   // clean public API: a wildcard
    swapHelper(list, i, j);                             // the capture is inferred as E
}
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));              // ✅ everything is E
}
// ✔︎ swap(["a","b","c"], 0, 2) → [c, b, a]
```

This is EJ Item 31's advice: *"if a type parameter appears only once in a method declaration, replace it with a wildcard"*, and then use a private helper to write through it. Interestingly, the JDK's own `Collections.swap` takes the shortcut, and says so in a comment ✔︎ (JDK 21 source):

```java
public static void swap(List<?> list, int i, int j) {
    // instead of using a raw type here, it's possible to capture
    // the wildcard but it will require a call to a supplementary
    // private method
    final List l = list;
    l.set(i, l.set(j, l.get(i)));
}
```

Inside the JDK, a raw type is an accepted trade-off. In your code, prefer the helper: it keeps full type checking and costs one tiny method.

---

## 5. Bounded Type Parameters

### Upper bound

```java
class Stats<T extends Number> {                 // T must be Number or a subclass
    double sum(List<T> xs) {
        double s = 0;
        for (T x : xs) s += x.doubleValue();    // ✅ Number's methods are available on T
        return s;
    }
}
```

`extends` is used for both classes and interfaces in a bound. There's **no `super` bound for type parameters** (`<T super Integer>` doesn't compile); lower bounds exist only on wildcards.

### Multiple bounds: `&`

```java
static <T extends Number & Comparable<? super T>> T clampMax(List<T> values, T ceiling) { ... }
// ✔︎ clampMax([3, 9, 7, 5], 7) → 7
```

- At most **one class**, and it must come **first**; then any number of interfaces.
- The erasure of `T` is the **first** bound (`Number` here), so order affects the binary signature.

### Recursive bounds

```java
<T extends Comparable<T>>            // "T can be compared to T"
<T extends Comparable<? super T>>    // better: also accepts types that inherit compareTo from a supertype
```

Why `? super T` matters: `java.sql.Timestamp extends Date`, and `Date implements Comparable<Date>`, so `Timestamp` is **`Comparable<Date>`**, not `Comparable<Timestamp>`. With `<T extends Comparable<T>>`, `max(List<Timestamp>)` wouldn't compile. The #030 test `maxWorksForTypesComparableOnlyToASupertype` proves the `? super` version accepts it ✔︎.

When the bound isn't satisfied:

```java
static <T extends Comparable<T>> T max(List<T> l) { ... }
max(List.of(new Object()));
```
```text
error: method max in class E cannot be applied to given types;
  required: List<T>
  found:    List<Object>
  reason: inference variable E has incompatible bounds
```

### Self-types: the builder hierarchy (EJ Item 2) ✔︎

```java
abstract static class Pizza {
    abstract static class Builder<T extends Builder<T>> {      // T = "my own concrete builder type"
        final Set<String> toppings = new TreeSet<>();
        public T addTopping(String t) { toppings.add(t); return self(); }
        abstract Pizza build();
        protected abstract T self();                            // subclasses return `this`
    }
}
static final class Calzone extends Pizza {
    static final class Builder extends Pizza.Builder<Builder> {
        boolean sauceInside;
        public Builder sauceInside() { sauceInside = true; return this; }
        @Override Calzone build() { return new Calzone(this); }
        @Override protected Builder self() { return this; }
    }
}

new Calzone.Builder().addTopping("ham").sauceInside().addTopping("cheese").build();
// ✔︎ inherited addTopping returns Calzone.Builder, so sauceInside() can be chained after it
```

The same pattern appears in `Enum<E extends Enum<E>>` (so `compareTo` only accepts the same enum type) and in AssertJ's fluent assertions.

---

## 6. Type Inference

```java
List<String> a = new ArrayList<>();                 // diamond (Java 7+): inferred from the target type
List<String> b = Collections.emptyList();           // inferred from the target
int n = Collections.<String>emptyList().size();     // explicit TYPE WITNESS when there's no target
var c = List.of(1, 2.0);                            // E inferred as lub(Integer, Double) = Number & Comparable<…> & Constable & ConstantDesc
// ✔︎ elements are Integer and Double
```

**Anonymous classes with a diamond (Java 9+) need a target type** ✔︎:

```java
new Comparator<>() { public int compare(String x, String y) { return 0; } }.compare("a", "b");
```
```text
error: <anonymous Works$1> is not abstract and does not override abstract method compare(Object,Object) in Comparator
error: method does not override or implement a method from a supertype
```

With no target, `<>` infers `Comparator<Object>`, so `compare(String, String)` is an overload, not an override. Assign it to `Comparator<String> byLen = new Comparator<>() { … }` and it works ✔︎. (That's also a reminder to always write `@Override`.)

---

## 7. Erasure: What Happens After Compilation

| Source | After erasure |
|--------|---------------|
| `class Box<T> { T value; }` | `class Box { Object value; }` |
| `class NumBox<T extends Number> { T v; }` | `Number v` (the first bound) |
| `String s = box.get();` | `String s = (String) box.get();`, a **compiler-inserted `checkcast`** |
| `List<String>`, `List<Integer>` | both just `List` at run time (#019: `getClass()` is equal ✔︎) |

### Heap pollution in action ✔︎

```java
List<String> strs = new ArrayList<>();
List raw = strs;          // raw alias
raw.add(42);              // compiles (unchecked warning): an Integer in a List<String>
Object o = strs.get(0);   // ✔︎ fine: no cast is inserted when the target is Object
String s = strs.get(0);   // ✔︎ ClassCastException HERE, from the compiler-inserted checkcast
```

The failure happens **far from the bug**. For debugging legacy code, wrap collections with `Collections.checkedList(list, String.class)`: it checks at `add` time ✔︎ (*"Attempt to insert class java.lang.Integer element into collection with element type class java.lang.String"*).

`@SuppressWarnings` on a **local variable declaration** only covers that declaration's initializer. In the snippet above, the later `raw.add(42)` still produced its own warning ✔︎.

### Bridge methods ✔︎

```java
static final class Money implements Comparable<Money> {
    public int compareTo(Money o) { return Long.compare(cents, o.cents); }
}
```
```text
$ javap -p Works$Money.class
  public int compareTo(Works$Money);
  public int compareTo(java.lang.Object);          ← generated BRIDGE method

$ javap -c -p Works$Money.class   (the bridge)
  public int compareTo(java.lang.Object);
       0: aload_0
       1: aload_1
       2: checkcast     #8     // class Works$Money
       5: invokevirtual #19    // Method compareTo:(LWorks$Money;)I
       8: ireturn
```

After erasure, `Comparable.compareTo` takes an `Object`. The compiler generates this bridge so polymorphic calls through `Comparable` still reach your typed method. It's also why calling `compareTo` with a wrong type through a raw `Comparable` throws `ClassCastException`.

---

## 8. The Restrictions, with Real Compiler Errors

| You write | `javac` 21 says | Why |
|-----------|-----------------|-----|
| `T[] arr = new T[10];` / `new List<String>[3]` | `error: generic array creation` | Arrays need their element type at run time; `T` is erased |
| `o instanceof List<String>` | `error: Object cannot be safely cast to List<String>` | The type argument doesn't exist at run time (`instanceof List<?>` is fine) |
| `void print(List<String>)` + `void print(List<Integer>)` | `error: name clash: print(List<Integer>) and print(List<String>) have the same erasure` | Both become `print(List)` |
| `List<int> xs;` | `error: unexpected type` / `required: reference` / `found: int` | Type arguments must be reference types, hence boxing and `IntStream` |
| `class E<T> { static T instance; }` | `error: non-static type variable T cannot be referenced from a static context` | One static field is shared by *all* parameterizations |
| `return new T();` | `error: unexpected type` / `required: class` / `found: type parameter T` | The class isn't known at run time. Pass a `Supplier<T>` or `Class<T>` instead |
| `class GenEx<T> extends Exception` | `error: a generic class may not extend java.lang.Throwable` | `catch` clauses test the exception class at run time; `T` is erased |
| `catch (T e)` with `<T extends Exception>` | `error: unexpected type` | Same reason: the catch type must be a real class |

Workarounds:

```java
static <T> T make(Supplier<? extends T> factory) { return factory.get(); }      // make(ArrayList::new)
static <T> T[] newArray(IntFunction<T[]> gen, int n) { return gen.apply(n); }    // newArray(String[]::new, 10)
static <T> T create(Class<T> type) throws ReflectiveOperationException {
    return type.getDeclaredConstructor().newInstance();                          // type token (EJ Item 33)
}
```

---

## 9. Generics Design Checklist (for your own classes)

1. Will callers use it with different element types? → Make it **generic** (EJ Item 29).
2. For each method parameter: is it only read from, only written to, or both? → `extends` / `super` / plain `T` (**PECS**).
3. Does `T` need methods (`compareTo`, `doubleValue`)? → **Bound** it, and prefer `Comparable<? super T>`.
4. Is a type parameter used only once? → Replace it with a **wildcard** (+ a private helper if you need to write).
5. Return concrete parameterized types, **never wildcards**.
6. Generic varargs? → `@SafeVarargs` only if you don't store or expose the array (EJ Item 32).
7. Need `new T` / `T[]`? → Take a `Supplier<T>`, `IntFunction<T[]>` or `Class<T>`.
8. Zero unchecked warnings. Any suppression goes on the smallest scope, with a justifying comment.

---

## Pitfalls & Interview Angles

| Question | Answer |
|----------|--------|
| Is `List<Integer>` a subtype of `List<Number>`? | No (invariance). It's a subtype of `List<? extends Number>` |
| Why can't you add to `List<? extends Number>`? | The actual type is unknown (it could be `List<Double>`); only `null` is safe |
| What does `? super T` give you? | Writing `T`s is safe; reads are only `Object` |
| Explain PECS with a JDK method | `Collections.copy(List<? super T> dest, List<? extends T> src)` |
| `List<?>` vs `List<Object>`? | Unknown type, read-only-ish, accepts any List / holds anything, accepts only `List<Object>` |
| What is `CAP#1`? | The compiler's name for a captured wildcard type |
| Multiple bounds syntax? | `<T extends Class & Iface1 & Iface2>`, class first; erasure is the first bound |
| What's type erasure? | Type arguments are removed after checking; casts and bridge methods are inserted |
| What's a bridge method? | A synthetic method that keeps overriding working after erasure (`compareTo(Object)` → `compareTo(Money)`) |
| Why no `new T()`? | `T` is erased; pass a `Supplier<T>` or `Class<T>` |
| What is `Enum<E extends Enum<E>>` for? | A recursive bound: `compareTo` and friends accept only the same enum type |

---

## Handwritten Cheat Sheet

```
<T> declares · ? uses · raw = no checks (legacy only)
INVARIANT: List<Integer> ≠ List<Number>
? extends T → READ T, no add (producer)      ? super T → ADD T, read Object (consumer)
PECS: Collections.copy(List<? super T> dst, List<? extends T> src)
Comparator/Comparable = consumer → Comparator<? super T>
Function<? super T, ? extends R>
BOUNDS: <T extends Number> · <T extends Number & Comparable<? super T>> (class first)
        no <T super X>; lower bounds only on wildcards
RECURSIVE: <T extends Comparable<? super T>> (Timestamp is Comparable<Date>)
           Builder<T extends Builder<T>> + self() · Enum<E extends Enum<E>>
CAPTURE: ? → CAP#1; to write through List<?> use private <E> helper(List<E>)
INFERENCE: diamond · target typing · Collections.<String>emptyList() witness
           anonymous diamond needs a target type (else infers Object)
ERASURE: T → bound; checkcast inserted; List<String>.class == List<Integer>.class
         bridge: compareTo(Object) → checkcast → compareTo(Money)
CAN'T: new T() · new T[] · new List<String>[] · instanceof List<String>
       List<int> · static T · overloads by type argument (same erasure)
FIX: Supplier<T> · IntFunction<T[]> · Class<T> token · checkedList for debugging
```

---

**Next:** row 27 — Sort objects with custom comparators (#027) · row 29 — Comparable vs Comparator (#029)
