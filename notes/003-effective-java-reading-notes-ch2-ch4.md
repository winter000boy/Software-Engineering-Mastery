# #003 — Read: Effective Java (Bloch) — Reading Notes, Chapters 2–4

**Date:** 2026-09-24  
**Roadmap row:** 3 — P1 Java Core · Java runtime & OOP foundations · *Flex*  
**Source:** Joshua Bloch, *Effective Java*, 3rd ed. (2018) — Ch. 2 *Creating and Destroying Objects* (Items 1–9), Ch. 3 *Methods Common to All Objects* (Items 10–14), Ch. 4 *Classes and Interfaces* (Items 15–25)

> Why these chapters now: this week's focus is OOP foundations. Chapters 2–4 are the OOP core of the book: how objects are born, how they compare, and how classes are shaped. Collections/generics chapters are covered in #019.

---

## At a Glance (copy to handwritten notes)

- **Item 1:** Prefer **static factory methods** (`of`, `from`, `valueOf`) to constructors when naming, caching or subtype-returning helps.
- **Item 2:** Many parameters (4+ or optional ones) → **Builder**.
- **Item 3/4:** Singleton → **single-element enum**. Utility class → **private constructor**.
- **Item 5:** **Inject dependencies** through the constructor; don't hard-wire resources.
- **Item 6/7:** Don't create needless objects (autoboxing, `new String`, regex recompiling). **Null out obsolete references** in classes that manage their own memory.
- **Item 8/9:** **No finalizers/cleaners.** Always use **try-with-resources**.
- **Item 10/11:** `equals` must be reflexive, symmetric, transitive, consistent and non-null. **Override `hashCode` whenever you override `equals`.**
- **Item 12/13/14:** Always override `toString`. Avoid `clone` (use copy constructors). Implement `Comparable` with `Comparator` combinators.
- **Item 15/16/17:** Minimise accessibility, no public fields, **make classes immutable** unless there's a good reason not to.
- **Item 18/19:** **Composition over inheritance.** Design for inheritance or forbid it (`final`).
- **Item 20–25:** Prefer interfaces, add default methods carefully, no constant interfaces, no tagged classes, `static` nested classes, one top-level class per file.

---

## Chapter 2 — Creating and Destroying Objects

### Item 1 — Consider static factory methods instead of constructors

A **static factory** is a static method that returns an instance. It's *not* the GoF Factory Method pattern.

```java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;   // no allocation: cached instances
}
```

| Advantage | Example |
|-----------|---------|
| They have **names** | `BigInteger.probablePrime(bits, rnd)` reads better than `new BigInteger(bits, 5, rnd)` |
| They **don't have to create a new object** each time (caching, instance control) | `Boolean.valueOf`, `Integer.valueOf(-128..127)`, enum-like classes |
| They can return **any subtype** of the declared return type | `List.of(...)` returns hidden `ImmutableCollections.ListN`; callers depend only on `List` |
| The returned class can **vary with the input** | `EnumSet.noneOf` returns `RegularEnumSet` (≤64 constants, one `long`) or `JumboEnumSet` |
| The returned class **needn't exist when the API is written** | Service-provider frameworks: JDBC `DriverManager.getConnection` |

| Disadvantage | Mitigation |
|--------------|------------|
| A class with only private constructors **can't be subclassed** | Arguably good: it pushes composition (Item 18) |
| **Hard to find** in Javadoc | Use the naming conventions below |

**Naming conventions:** `from` (one param, type conversion: `Date.from(instant)`), `of` (aggregation: `EnumSet.of(A, B)`), `valueOf`, `instance`/`getInstance`, `create`/`newInstance` (new each call), `getType`/`newType` (factory in another class: `Files.newBufferedReader`), `type` (`Collections.list(enumeration)`).

**My take:** reach for a static factory first when there's any caching, naming or "return an interface" benefit; plain constructors are fine for simple value classes.

---

### Item 2 — Consider a builder when faced with many constructor parameters

Three options for many optional parameters:

1. **Telescoping constructors:** `Pizza(size)`, `Pizza(size, cheese)`, `Pizza(size, cheese, pepperoni)`… Hard to read, and easy to swap two `int`s by mistake.
2. **JavaBeans** (no-arg constructor + setters): the object is **inconsistent mid-construction** and **can't be immutable**.
3. **Builder** ✅: named, readable, validated, immutable result.

```java
public final class NutritionFacts {
    private final int servingSize;   // required
    private final int servings;      // required
    private final int calories;      // optional
    private final int sodium;        // optional

    public static final class Builder {
        private final int servingSize;
        private final int servings;
        private int calories = 0;
        private int sodium = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }
        public Builder calories(int val) { calories = val; return this; }
        public Builder sodium(int val)   { sodium = val;   return this; }
        public NutritionFacts build()    { return new NutritionFacts(this); }
    }

    private NutritionFacts(Builder b) {
        if (b.servingSize <= 0) throw new IllegalArgumentException("servingSize");
        servingSize = b.servingSize;
        servings = b.servings;
        calories = b.calories;
        sodium = b.sodium;
    }
}

NutritionFacts coke = new NutritionFacts.Builder(240, 8).calories(100).sodium(35).build();
```

- Validate **in the private constructor** (after copying the fields), so the checks apply to the object's final state.
- Builders also work with class hierarchies: a generic `Builder<T extends Builder<T>>` with an abstract `self()` method (the "simulated self-type" idiom).
- Cost: an extra object and more code. Worth it at roughly 4+ parameters, or when parameters are likely to grow.
- In real code, Lombok's `@Builder` or a `record` with a compact constructor often replaces the hand-written version. Still write one by hand to understand it.

---

### Item 3 — Enforce the singleton property with a private constructor or an enum type

```java
// 1. Public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
}

// 2. Static factory: can later change to per-thread, generic, or Supplier (Elvis::getInstance)
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    public static Elvis getInstance() { return INSTANCE; }
}

// 3. Single-element enum ✅ preferred
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { }
}
```

The enum form is free of **serialization** problems (options 1/2 need `readResolve` and `transient` fields, or deserialisation creates a second instance) and of **reflection attacks** (`AccessibleObject.setAccessible` can call a private constructor, but you can't reflectively create an enum constant). The one limit: an enum can't extend another class.

**Interview angle:** singletons make testing harder, since they act as global state. Prefer dependency injection (Item 5) and let a container like Spring manage the single instance.

---

### Item 4 — Enforce noninstantiability with a private constructor

Utility classes (`Math`, `Arrays`, `Collections`) group static methods. Without a declared constructor, the compiler adds a public no-arg one.

```java
public final class StringUtils {
    private StringUtils() {
        throw new AssertionError("no instances");   // guards against calls from inside the class too
    }
    public static boolean isBlank(String s) { return s == null || s.isBlank(); }
}
```

Making the class `abstract` does **not** work: someone can subclass it and instantiate the subclass, and it wrongly suggests the class is meant to be extended.

---

### Item 5 — Prefer dependency injection to hardwiring resources

```java
// ❌ Hard-wired: can't swap the dictionary, can't test
public class SpellChecker {
    private static final Lexicon dictionary = new EnglishLexicon();
}

// ✅ Injected through the constructor
public class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
}
```

- Static utility classes and singletons are wrong for classes whose behaviour **depends on a resource**.
- A variant is to pass a **factory**: `Supplier<? extends Tile>` (bounded wildcard, see #019).
- Frameworks (Spring, Guice, Dagger) automate this wiring. The principle, **constructor injection**, is what we use in the CLI inventory project (#007–#011).

---

### Item 6 — Avoid creating unnecessary objects

```java
String s = new String("bikini");   // ❌ a new object on every execution
String s = "bikini";               // ✅ interned literal, reused

// ❌ Pattern compiled on every call
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
// ✅ Compile once
private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
static boolean isRomanNumeral(String s) { return ROMAN.matcher(s).matches(); }

// ❌ Autoboxing: Long sum creates ~2^31 Long objects
Long sum = 0L;
for (long i = 0; i <= Integer.MAX_VALUE; i++) sum += i;
// ✅ primitive
long sum = 0L;
```

**Prefer primitives to boxed primitives and watch for unintended autoboxing.** Counterpoint: don't build object pools for cheap objects. Modern GCs make small short-lived objects almost free (they die young in Eden, see #004). Pools are only worth it for expensive objects such as DB connections.

---

### Item 7 — Eliminate obsolete object references

Java has GC, but **memory leaks still happen** when you keep references you no longer need.

```java
public Object pop() {
    if (size == 0) throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;    // ✅ eliminate the obsolete reference
    return result;
}
```

Without the `null`, the array slot keeps the object **reachable**, so GC can't reclaim it. This is exactly what `ArrayList.remove` does internally (`elementData[--size] = null`, see #020).

The three usual leak sources:

1. **Classes that manage their own memory** (arrays as storage, like stacks, pools and caches). Null out slots when done.
2. **Caches:** use `WeakHashMap` (entries go when the key is no longer strongly referenced), a size-bounded `LinkedHashMap` with `removeEldestEntry` (the LRU project, rows 32–36), or Caffeine.
3. **Listeners/callbacks** registered and never removed. Store them as weak references or deregister them.

Don't null everything out. Letting a local variable go out of scope is the normal way to drop a reference.

---

### Item 8 — Avoid finalizers and cleaners

- `finalize()` (deprecated since 9, *deprecated for removal* since 18, JEP 421) and `Cleaner` (9+) run **unpredictably or never**. They're slow (≈50× the cost of allocation with a finalizer), swallow exceptions, and open **finalizer attacks** (a subclass's finalizer can resurrect a half-built object).
- **Never** rely on them to release resources or update persistent state.
- Legitimate use: a **safety net** for when the owner forgets to call `close()`, and for native peers.
- Instead, implement `AutoCloseable` and require clients to call `close()` (Item 9).

---

### Item 9 — Prefer try-with-resources to try-finally

```java
// ❌ try-finally: ugly when nested; an exception in close() hides the original one
BufferedReader br = new BufferedReader(new FileReader(path));
try {
    return br.readLine();
} finally {
    br.close();
}

// ✅ try-with-resources (Java 7+)
try (BufferedReader br = new BufferedReader(new FileReader(path))) {
    return br.readLine();
}

// ✅ Several resources: closed in reverse order
try (InputStream in = new FileInputStream(src);
     OutputStream out = new FileOutputStream(dst)) {
    in.transferTo(out);
}
```

If both the body and `close()` throw, the body's exception propagates and the `close()` exception is attached as **suppressed** (`Throwable.getSuppressed()`), so nothing is lost. Any class implementing `AutoCloseable` works.

---

## Chapter 3 — Methods Common to All Objects

### Item 10 — Obey the general contract when overriding `equals`

**Don't override** when each instance is unique (threads), when there's no need for logical equality, when a superclass `equals` already fits, or when the class is private and `equals` is never called.

**Override** for **value classes** (`Money`, `Isbn`, `Point`).

The contract (for non-null `x`, `y`, `z`):

| Property | Rule |
|----------|------|
| Reflexive | `x.equals(x)` is true |
| Symmetric | `x.equals(y)` ⇔ `y.equals(x)` |
| Transitive | `x.equals(y) && y.equals(z)` ⇒ `x.equals(z)` |
| Consistent | Repeated calls give the same answer while no fields used in `equals` change |
| Non-null | `x.equals(null)` is false |

Classic violations:

- **Symmetry:** a `CaseInsensitiveString.equals(String)` returns true, but `String.equals(CaseInsensitiveString)` returns false.
- **Transitivity:** `ColorPoint extends Point` adds `color`. *There is no way to extend an instantiable class and add a value component while preserving the equals contract* (unless you give up OO abstraction). Fix: **composition**, where `ColorPoint` *has* a `Point` (Item 18).
- `getClass()`-based equals breaks the **Liskov substitution principle**. Prefer `instanceof` plus composition.

Recipe:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                          // 1. identity shortcut
    if (!(o instanceof PhoneNumber pn)) return false;    // 2. type check (+ pattern variable, Java 16+); false for null
    return areaCode == pn.areaCode                       // 3. compare significant fields
        && prefix == pn.prefix
        && lineNum == pn.lineNum;
}
```

- Compare `float`/`double` with `Float.compare`/`Double.compare` (because of `NaN` and `-0.0`), arrays with `Arrays.equals`, and nullable fields with `Objects.equals`.
- Compare the fields most likely to differ, and cheapest to check, first.
- **Always** use the `Object` parameter type. `equals(PhoneNumber o)` is an *overload*, not an override. `@Override` catches that mistake.
- A **`record`** (Java 16+) generates a correct `equals`/`hashCode`/`toString` for you.

---

### Item 11 — Always override `hashCode` when you override `equals`

Contract: **equal objects must have equal hash codes.** Unequal objects *may* collide, but fewer collisions means faster hash tables.

Break it and `HashMap`/`HashSet` stop working:

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
m.get(new PhoneNumber(707, 867, 5309));   // null! different hash → different bucket (#020)
```

Recipe:

```java
@Override
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
// One-liner (slower: varargs array + boxing): Objects.hash(areaCode, prefix, lineNum)
```

- Why 31? It's an odd prime, and `31 * i == (i << 5) - i`, which the JIT optimises.
- Use the **same fields** as `equals`, and never leave a significant field out to "speed things up".
- For expensive immutable objects, cache the hash lazily (like `String.hash`).
- Don't document the exact hash algorithm, so you keep the freedom to change it.

---

### Item 12 — Always override `toString`

`PhoneNumber@163b91` tells you nothing. A good `toString` returns **all interesting information**, makes logs, debuggers and assertion messages useful, and should say in its docs whether the format is guaranteed (a value class like `LocalDate` can specify a parseable format). Offer **accessors** for everything in the string, so clients never have to parse `toString`.

---

### Item 13 — Override `clone` judiciously

`Cloneable` is a broken mixin interface. It has no `clone` method; it changes how the protected `Object.clone` behaves. `clone` makes shallow copies and fights `final` fields. **Prefer a copy constructor or copy factory:**

```java
public Yum(Yum yum) { ... }                    // copy constructor
public static Yum newInstance(Yum yum) { ... } // copy factory
new ArrayList<>(otherCollection);              // "conversion constructors", which take an interface type
```

The exception is arrays: `array.clone()` is the best way to copy them.

---

### Item 14 — Consider implementing `Comparable`

Implementing `Comparable<T>` gives a **natural ordering** and plugs into `Collections.sort`, `TreeSet`, `TreeMap`, `Arrays.binarySearch` and `stream().sorted()`.

```java
private static final Comparator<PhoneNumber> COMPARATOR =
        Comparator.comparingInt((PhoneNumber pn) -> pn.areaCode)
                  .thenComparingInt(pn -> pn.prefix)
                  .thenComparingInt(pn -> pn.lineNum);

@Override
public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

- The contract mirrors `equals`: `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))`, it's transitive, and it's **strongly recommended** that `(x.compareTo(y) == 0) == x.equals(y)`. `BigDecimal` breaks this (`1.0` vs `1.00`), so a `HashSet` holds 2 elements but a `TreeSet` holds 1.
- **Never** compare by subtraction (`return a - b;`). It **overflows** for large values of opposite sign. Use `Integer.compare(a, b)` or the comparator combinators.
- Deep dive: row 29 (Comparable vs Comparator).

---

## Chapter 4 — Classes and Interfaces

### Item 15 — Minimize the accessibility of classes and members

**Information hiding / encapsulation** decouples components, so they can be developed, tested, optimised and replaced independently.

| Modifier | Visible to |
|----------|-----------|
| `private` | Declaring top-level class |
| *package-private* (none) | Same package |
| `protected` | Package + subclasses (it becomes part of the exported API) |
| `public` | Everyone |

Rules:

- Make each member **as inaccessible as possible**. Start `private`; widen only if needed.
- Instance fields of public classes should **rarely** be public. Mutable public fields make a class **not thread-safe**.
- A `public static final` **array field**, or an accessor that returns one, is a security hole, because callers can modify the contents. Return `List.copyOf(...)`/`Collections.unmodifiableList(...)` or `array.clone()`.
- Package-private for testing is acceptable; wider access is not.
- **Modules** (Java 9+): a package that isn't `exports`-ed is invisible outside the module, even for its `public` classes.

### Item 16 — In public classes, use accessor methods, not public fields

```java
class Point { public double x; public double y; }   // ❌ in a public class you can't change the representation later
```

Public fields are OK in **package-private or private nested** classes (the scope is limited) and for `public static final` constants of immutable types. Or just use a `record Point(double x, double y)`.

### Item 17 — Minimize mutability

An **immutable** object's state never changes after construction (`String`, `BigInteger`, `LocalDate`, records with immutable components).

**Five rules:**

1. No **mutator** methods (setters).
2. The class **can't be extended** (`final`, or a private constructor + static factories).
3. All fields are **`final`**.
4. All fields are **`private`**.
5. **Exclusive access to mutable components:** make **defensive copies** in constructors, accessors and `readObject`.

```java
public final class Complex {
    private final double re;
    private final double im;

    public static final Complex ZERO = new Complex(0, 0);

    private Complex(double re, double im) { this.re = re; this.im = im; }
    public static Complex valueOf(double re, double im) { return new Complex(re, im); }

    public Complex plus(Complex c) {                 // "functional" style: returns a new instance
        return new Complex(re + c.re, im + c.im);
    }
}
```

**Benefits:** simple (only one state ever), **inherently thread-safe with no synchronisation** (the `final` fields guarantee safe publication, see JMM #012), freely shareable and cacheable, great map keys and set elements, failure atomicity for free.

**Cost:** a new object for each distinct value. Fix that with a **mutable companion class** (`String` ↔ `StringBuilder`).

> *"Classes should be immutable unless there's a very good reason to make them mutable… If a class cannot be made immutable, limit its mutability as much as possible."* Make every field `private final` unless there's a good reason not to.

### Item 18 — Favor composition over inheritance

Inheritance **across package boundaries** (from a concrete class you don't control) **breaks encapsulation**: the subclass depends on the superclass's implementation details.

The classic bug:

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;
    @Override public boolean add(E e) { addCount++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) { addCount += c.size(); return super.addAll(c); }
}
s.addAll(List.of("Snap", "Crackle", "Pop"));
s.getAddCount();  // 6, not 3! HashSet.addAll calls add() internally → double counting
```

**Fix: a wrapper (decorator) class plus a reusable forwarding class.**

```java
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }
    public boolean add(E e)                       { return s.add(e); }
    public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
    // ... forward every other Set method
}

public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
    public InstrumentedSet(Set<E> s) { super(s); }
    @Override public boolean add(E e) { addCount++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) { addCount += c.size(); return super.addAll(c); }
    public int getAddCount() { return addCount; }
}
```

It now works with **any** `Set` (`TreeSet`, `HashSet`…) and doesn't depend on how `addAll` is implemented.

**Use inheritance only for a genuine "is-a"** relationship, where every B really is an A. JDK mistakes: `Stack extends Vector`, `Properties extends Hashtable`. (This is the same idea as the `ShippingStrategy` exercise in #006.)

### Item 19 — Design and document for inheritance or else prohibit it

A class designed for inheritance must **document its self-use of overridable methods** (`@implSpec`: "this implementation calls `add`…"). That's the documented version of the Item 18 bug. Also:

- **Constructors must not call overridable methods.** The subclass override runs **before the subclass constructor**, so it sees uninitialised fields:

  ```java
  class Super { Super() { overrideMe(); } void overrideMe() { } }
  final class Sub extends Super {
      private final Instant instant;
      Sub() { instant = Instant.now(); }
      @Override void overrideMe() { System.out.println(instant); }  // prints null first!
  }
  ```
- The same applies to `clone` and `readObject`.
- **Test a class meant for extension by writing 3 subclasses** before you release it.
- Otherwise, **prohibit subclassing**: `final`, or private constructors plus static factories.

### Item 20 — Prefer interfaces to abstract classes

- A class can implement **many** interfaces but extend only **one** class, so interfaces are ideal for **mixins** (`Comparable`, `AutoCloseable`).
- Existing classes can easily be retrofitted to implement a new interface.
- Interfaces allow **non-hierarchical** type frameworks (`SingerSongwriter extends Singer, Songwriter`).
- **Skeletal implementation** (the "Template Method" combination): interface + `AbstractXxx` class, e.g. `List` + `AbstractList`, `Map` + `AbstractMap`. Implementors get most methods for free:

  ```java
  static List<Integer> intArrayAsList(int[] a) {
      return new AbstractList<>() {
          @Override public Integer get(int i) { return a[i]; }
          @Override public Integer set(int i, Integer v) { int old = a[i]; a[i] = v; return old; }
          @Override public int size() { return a.length; }
      };
  }
  ```

### Item 21 — Design interfaces for posterity

Java 8 **default methods** let you add methods to existing interfaces, but existing implementations don't know about them. Example: `Collection.removeIf` default iterates and removes, which **breaks** Apache Commons `SynchronizedCollection`: the default isn't synchronised and can throw `ConcurrentModificationException`. Adding a default method is risky. Design interfaces carefully up front, and test them with multiple implementations.

### Item 22 — Use interfaces only to define types

❌ **Constant interface anti-pattern** (`interface PhysicalConstants { double AVOGADROS_NUMBER = ...; }`): implementing it leaks constants into the class's public API forever. Put constants in the relevant class/enum, or in a noninstantiable utility class (Item 4), and use `import static`.

### Item 23 — Prefer class hierarchies to tagged classes

❌ **Tagged class:** one class with a `kind` field (`RECTANGLE`/`CIRCLE`), a `switch` in every method, fields that are irrelevant for some tags, and it's easy to forget a case.
✅ An **abstract root + one subclass per flavour**. Java 17+ **sealed interfaces + records + pattern-matching `switch`** make this even better and exhaustively checked:

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

static double area(Shape s) {
    return switch (s) {                 // compiler checks it's exhaustive, no default needed
        case Circle c    -> Math.PI * c.r() * c.r();
        case Rectangle r -> r.w() * r.h();
    };
}
```

### Item 24 — Favor static member classes over nonstatic

| Kind | Has enclosing instance? | Use for |
|------|:---:|---------|
| **Static member class** | No | Helper classes used with the outer class: `Map.Entry`, `Builder`, `Node` |
| Nonstatic (inner) class | **Yes** (hidden `Outer.this` reference) | Adapters that need the outer instance, e.g. a collection's `Iterator` |
| Anonymous class | If declared in a non-static context | One-off small implementations (mostly replaced by lambdas) |
| Local class | If declared in a non-static context | Rare |

**If a member class doesn't need access to the enclosing instance, always make it `static`.** Otherwise each instance carries a hidden reference to its outer object: wasted memory, and a **memory leak** risk (Item 7), because the outer object can't be collected while the inner one lives. `HashMap.Node` and `LinkedList.Node` are `static` for this reason (#020).

### Item 25 — Limit source files to a single top-level class

Two top-level classes in one file can make **compilation order** change which definition wins. Put each top-level class in its own file, or make the extras static member classes.

---

## Connections to Other Notes

| Item | Where it shows up |
|------|-------------------|
| 7, 24 | `ArrayList.remove` nulls slots; `static` `Node` classes (#020) |
| 10, 11 | Why `HashMap` keys need both `equals` and `hashCode` (#017, #020) |
| 14 | `TreeMap`/`TreeSet` ordering (#017); Comparable vs Comparator (row 29) |
| 17 | `final` fields + safe publication (#012 JMM); immutable domain objects in #005 |
| 18 | Strategy via composition (#006 Ex. 3); `Library` design in #005 |
| 5 | Constructor injection in the CLI inventory service (#009) |

---

## Pitfalls & Interview Angles

| Question | Short answer |
|----------|--------------|
| Static factory vs constructor? | Names, caching, can return subtypes / hidden classes. Can't be subclassed if there are no public constructors |
| When to use a Builder? | Many (esp. optional) params; want immutability + readability |
| Best singleton? | Single-element `enum`: safe against serialization and reflection |
| Can Java leak memory? | Yes, through obsolete references: array-backed storage, caches, listeners |
| Why not `finalize()`? | Unpredictable, slow, security issues, deprecated for removal (JEP 421) |
| `equals` without `hashCode`? | Breaks `HashMap`/`HashSet`: equal keys land in different buckets |
| Why is `a - b` a bad comparator? | Integer overflow → wrong sign |
| Why immutability? | Thread-safe for free, shareable, simple, great keys |
| Composition vs inheritance? | Inheritance breaks encapsulation across packages (`InstrumentedHashSet`); wrap and forward instead |
| Why must constructors not call overridable methods? | The override runs before the subclass is initialised |
| Static vs inner class? | Inner holds a hidden outer reference (memory + leak); default to `static` |

---

## Handwritten Cheat Sheet

```
CH2 CREATE/DESTROY
 1 static factory: of/from/valueOf — named, cached, returns subtype
 2 Builder for 4+/optional params; validate in private ctor
 3 singleton = enum INSTANCE
 4 utility class: private ctor + throw AssertionError
 5 inject deps via ctor (Supplier<? extends T> for factories)
 6 no needless objects: literals, precompiled Pattern, long not Long
 7 null out obsolete refs (stack pop), watch caches/listeners
 8 no finalize/Cleaner (safety net only)
 9 try-with-resources; close() exceptions → suppressed

CH3 OBJECT METHODS
10 equals: reflexive, symmetric, transitive, consistent, x.equals(null)=false
   this==o → instanceof → compare fields; Double.compare, Objects.equals
11 override hashCode with equals; 31*result + field
12 always toString (all interesting info)
13 avoid clone → copy constructor / copy factory
14 Comparable via Comparator.comparingInt(...).thenComparing(...); never a-b

CH4 CLASSES/INTERFACES
15 least access; no public mutable fields/arrays
16 accessors, not public fields (or records)
17 IMMUTABLE: no setters, final class, private final fields, defensive copies
18 composition > inheritance (InstrumentedHashSet counts 6 not 3)
19 document self-use or make final; ctor must not call overridable methods
20 interfaces > abstract classes; skeletal AbstractXxx
21 default methods can break old impls (removeIf)
22 no constant interfaces
23 no tagged classes → hierarchy / sealed + records + switch
24 static nested class unless you need Outer.this
25 one top-level class per file
```

---

**Next note:** #004 — Memory areas (heap, stack, metaspace, class loading)
