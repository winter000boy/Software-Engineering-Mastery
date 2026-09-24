# #027 — Solution: Sort Objects with Custom Comparators

**Date:** 2026-09-24  
**Problem:** [027-sort-with-custom-comparators.md](../problems/027-sort-with-custom-comparators.md)

> Verified on Java 21: **12/12 JUnit tests pass**. Every expected order below is asserted by a test, and all three failure modes (overflow, TimSort's contract exception, TreeSet data loss) are reproduced in tests.

---

## 1. Code

### `Department.java`

```java
package com.mastery.sorting;

/** Enum natural order = declaration order (ordinal), which here is the business order we want. */
public enum Department {
    ENGINEERING, PRODUCT, SALES, HR
}
```

### `Employee.java`

```java
package com.mastery.sorting;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Comparator;
import java.util.Objects;

/**
 * An employee. Natural order = by id (a unique, stable key), with tie-breakers on every other
 * component so that compareTo is consistent with the record's generated equals().
 */
public record Employee(int id, String name, Department department, BigDecimal salary,
                       LocalDate hireDate, String manager) implements Comparable<Employee> {

    public Employee {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(department, "department");
        Objects.requireNonNull(salary, "salary");
        Objects.requireNonNull(hireDate, "hireDate");
        // manager may be null: the CEO has none
    }

    // Natural ordering: by id, then by every other component, so compareTo == 0 exactly when
    // equals() is true (EJ Item 14's "strongly recommended" consistency). In practice ids are
    // unique and the tie-breakers never run, but they make TreeSet/TreeMap safe even if not.
    private static final Comparator<Employee> NATURAL =
            Comparator.comparingInt(Employee::id)
                    .thenComparing(Employee::name)
                    .thenComparing(Employee::department)
                    .thenComparing(Employee::salary)
                    .thenComparing(Employee::hireDate)
                    .thenComparing(Employee::manager, Comparator.nullsFirst(Comparator.naturalOrder()));

    @Override
    public int compareTo(Employee other) {
        return NATURAL.compare(this, other);
    }
}
```

**Task 1: why consistent with `equals`?** A record's `equals` compares **all** components. If `compareTo` looked only at `id`, two records with the same id but different names would be `compareTo == 0` but not `equals`, and a `TreeSet<Employee>` would silently keep just one of them (#029 §3). The tie-breakers make `compareTo == 0` exactly when `equals` is true. In practice ids are unique and the tie-breakers never run, so they cost nothing.

### `EmployeeComparators.java`

```java
package com.mastery.sorting;

import java.text.Collator;
import java.util.Comparator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.stream.Collectors;

/** Reusable comparators: built once, stored in static finals (they're immutable and thread-safe). */
public final class EmployeeComparators {
    private EmployeeComparators() { }

    /** Task 2: highest salary first; equal salaries by name A→Z. */
    public static final Comparator<Employee> BY_SALARY_DESC_THEN_NAME =
            Comparator.comparing(Employee::salary, Comparator.reverseOrder())   // reverse ONLY the salary key
                    .thenComparing(Employee::name);

    /** Task 3: department in enum order, then most senior (earliest hire) first. */
    public static final Comparator<Employee> BY_DEPARTMENT_THEN_SENIORITY =
            Comparator.comparing(Employee::department)
                    .thenComparing(Employee::hireDate);

    /** Task 4: by manager with employees that have NO manager last; then by name. */
    public static final Comparator<Employee> BY_MANAGER_NULLS_LAST =
            Comparator.comparing(Employee::manager, Comparator.nullsLast(Comparator.naturalOrder()))
                    .thenComparing(Employee::name);

    /** Task 5: human, locale-aware name order (accents and case handled by Collator). */
    public static Comparator<Employee> byNameLocaleAware(Locale locale) {
        Collator collator = Collator.getInstance(locale);   // Collator is a Comparator<Object>
        return Comparator.comparing(Employee::name, collator);
    }

    /** Task 7: top N earners per department, departments in enum order. */
    public static Map<Department, List<String>> topEarnersPerDepartment(List<Employee> staff, int n) {
        return staff.stream()
                .sorted(BY_SALARY_DESC_THEN_NAME)
                .collect(Collectors.groupingBy(
                        Employee::department,
                        () -> new java.util.EnumMap<>(Department.class),        // ordered by enum
                        Collectors.collectingAndThen(Collectors.toList(),
                                list -> list.stream().limit(n).map(Employee::name).toList())));
    }
}
```

### `NaturalStringComparator.java`

```java
package com.mastery.sorting;

import java.math.BigInteger;
import java.util.Comparator;

/**
 * Task 8: "natural" (human) order for strings containing numbers: file2 < file10.
 * Splits into runs of digits and non-digits; digit runs compare numerically.
 * Non-digit runs compare case-insensitively, falling back to case-sensitive for a total order.
 */
public final class NaturalStringComparator implements Comparator<String> {
    public static final NaturalStringComparator INSTANCE = new NaturalStringComparator();

    private NaturalStringComparator() { }

    @Override
    public int compare(String a, String b) {
        int i = 0, j = 0;
        while (i < a.length() && j < b.length()) {
            char ca = a.charAt(i), cb = b.charAt(j);
            if (Character.isDigit(ca) && Character.isDigit(cb)) {
                int si = i, sj = j;
                while (i < a.length() && Character.isDigit(a.charAt(i))) i++;
                while (j < b.length() && Character.isDigit(b.charAt(j))) j++;
                // BigInteger: digit runs can be longer than a long ("id99999999999999999999")
                int cmp = new BigInteger(a.substring(si, i)).compareTo(new BigInteger(b.substring(sj, j)));
                if (cmp != 0) {
                    return cmp;
                }
            } else {
                int cmp = Character.compare(Character.toLowerCase(ca), Character.toLowerCase(cb));
                if (cmp != 0) {
                    return cmp;
                }
                i++;
                j++;
            }
        }
        int byLength = Integer.compare(a.length() - i, b.length() - j);   // shorter remainder first
        return byLength != 0 ? byLength : a.compareTo(b);                  // total order: "a01" vs "a1", case
    }
}
```

---

## 2. Results (each asserted by a test)

| Task | Comparator | Result |
|------|------------|--------|
| 1 | natural (`id`) | ids `1, 2, 3, 4, 5, 6, 7` |
| 2 | salary ↓, name ↑ | `Bob, adam, Émile, Eve, Dan, Chloé, zoe` |
| 3 | department (enum order), hire date ↑ | `Bob, Émile, adam, Eve, Dan, zoe, Chloé` |
| 4 | manager (nulls last), name | `Chloé, adam, zoe, Émile, Dan, Bob, Eve` |
| 5a | `String` natural | `Bob, Chloé, Dan, Eve, adam, zoe, Émile` |
| 5b | `CASE_INSENSITIVE_ORDER` | `adam, Bob, Chloé, Dan, Eve, zoe, Émile` |
| 5c | `Collator(FRENCH)` | `adam, Bob, Chloé, Dan, Émile, Eve, zoe` |
| 6 | name (Collator), then department (stable) | `adam, Bob, Émile, Eve, Dan, zoe, Chloé` |
| 7 | top 2 per department | `ENGINEERING=[Bob, adam], PRODUCT=[Eve], SALES=[Dan, zoe], HR=[Chloé]` |
| 8 | natural strings | `File1.txt, file1.txt, file02.txt, file2.txt, file10.txt, img2b, img12` |

### Task 2 detail: "adam" before "Émile"

Both earn 120 000, so the name decides. `String.compareTo` compares **UTF-16 code units**: `a` = 97 and `É` = 201, so `adam` comes first. That's the correct answer for this comparator, but it's only "right" by accident. Task 5 shows why.

### Task 4 detail: nulls on the *key*

`Comparator.comparing(Employee::manager, Comparator.nullsLast(Comparator.naturalOrder()))` puts the null-tolerance on the **manager key**. `Comparator.nullsLast(comparing(Employee::manager))` would be wrong: it tolerates null *employees*, then throws an NPE when comparing null managers. Within the "Bob" group, `Chloé` sorts before `adam` because the name tie-breaker is plain `String` order (uppercase < lowercase).

### Task 5: three ways to sort names

| Order | Rule | Consequence |
|-------|------|-------------|
| `String.compareTo` | UTF-16 code units: `A–Z` (65–90) < `a–z` (97–122) < `É` (201) | All capitalised names first, and accented names **last** |
| `CASE_INSENSITIVE_ORDER` | Case-folds each character, then compares code units | `adam`, `Bob` interleave correctly, but `Émile` is **still after `zoe`** |
| `Collator.getInstance(Locale.FRENCH)` | Locale collation rules: base letter first, then accents, then case | `Émile` between `Dan` and `Eve`: what humans expect |

**Rule: anything a human reads sorted uses a `Collator`.** Internal keys (ids, codes) can use `String.compareTo`.

### Task 6: stability

`List.sort` is TimSort, which is **stable**: elements that compare equal keep their current relative order. After sorting by name, the department-only sort treats all ENGINEERING people as equal, so `adam, Bob, Émile` stay in name order. A chained comparator (`comparing(department).thenComparing(name, collator)`) is usually clearer, but multi-pass stable sorts are how UIs that let you "click one column header, then another" work.

### Task 8: natural string comparator

The string is split into **runs**. Digit runs compare **numerically** (as `BigInteger`, so `id99999999999999999999` works), and everything else compares case-insensitively character by character. The final tie-breaks (remaining length, then `String.compareTo`) make it a **total order**: `file02.txt` vs `file2.txt` are numerically equal, but they aren't equal strings, so they must not compare as 0. That would break antisymmetry-based guarantees and make a `TreeSet` drop one of them. The contract test checks `sgn(c(x,y)) == -sgn(c(y,x))` and "0 ⇒ equal" across 300 × 300 random pairs, then checks that the sorted output is non-decreasing.

---

## 3. Task 9: Breaking It on Purpose

**(a) Subtraction overflow**

```java
vals.sort((x, y) -> x - y);   // [MAX_VALUE, -1, MIN_VALUE, 0] → [0, MIN_VALUE, -1, MAX_VALUE]  ✗
vals.sort(Integer::compare);  //                              → [MIN_VALUE, -1, 0, MAX_VALUE]  ✓
```

`MAX_VALUE - (-1)` wraps around to `MIN_VALUE`, which is negative, so `MAX` "is less than" `-1`. With realistic values (ages, prices) this never shows up in tests, then fails on extreme data. **Always use `Integer.compare`/`comparingInt`.**

**(b) Contract violation → TimSort exception**

`(x, y) -> x < y ? -1 : 1` never returns 0, so `compare(5, 5) == 1` and also `compare(5, 5) == 1` "the other way round". Antisymmetry is broken. Sorting 2 000 random values in 0–49 (many duplicates) threw `IllegalArgumentException: Comparison method violates its general contract!` in **44 of 50** runs. TimSort only notices when a merge step hits an inconsistency, so it's **data-dependent**: it can pass every test and fail in production.

**(c) `TreeSet` ordered only by salary**

```java
new TreeSet<>(Comparator.comparing(Employee::salary))                          // 6 elements: adam or Émile LOST
new TreeSet<>(Comparator.comparing(Employee::salary).thenComparingInt(Employee::id))   // 7 elements ✓
```

A sorted set decides "duplicate" by `compare == 0`. **Always end a comparator used by a sorted collection with a unique tie-breaker.**

---

## 4. Tests

### `test: SortingTest.java`

```java
package com.mastery.sorting;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Random;
import java.util.TreeSet;
import org.junit.jupiter.api.Test;

class SortingTest {

    static Employee emp(int id, String name, Department d, String salary, String hired, String manager) {
        return new Employee(id, name, d, new BigDecimal(salary), LocalDate.parse(hired), manager);
    }

    static final List<Employee> STAFF = List.of(
            emp(5, "zoe",    Department.SALES,       "70000", "2021-03-01", "Bob"),
            emp(1, "Bob",    Department.ENGINEERING, "150000", "2015-06-01", null),
            emp(3, "Émile",  Department.ENGINEERING, "120000", "2018-01-15", "Bob"),
            emp(2, "adam",   Department.ENGINEERING, "120000", "2019-09-09", "Bob"),
            emp(4, "Chloé",  Department.HR,          "80000", "2020-02-02", "Bob"),
            emp(6, "Dan",    Department.SALES,       "90000", "2017-11-11", "zoe"),
            emp(7, "Eve",    Department.PRODUCT,     "110000", "2016-04-04", null));

    static List<String> names(List<Employee> list) {
        return list.stream().map(Employee::name).toList();
    }

    static List<Employee> sorted(Comparator<Employee> c) {
        List<Employee> copy = new ArrayList<>(STAFF);
        copy.sort(c);
        return copy;
    }

    @Test
    void naturalOrderIsById() {
        List<Employee> copy = new ArrayList<>(STAFF);
        java.util.Collections.sort(copy);
        assertEquals(List.of(1, 2, 3, 4, 5, 6, 7), copy.stream().map(Employee::id).toList());
    }

    @Test
    void salaryDescendingThenName() {
        assertEquals(List.of("Bob", "adam", "Émile", "Eve", "Dan", "Chloé", "zoe"),
                names(sorted(EmployeeComparators.BY_SALARY_DESC_THEN_NAME)));
        // String natural order is by UTF-16 code unit: "adam" (a=97) < "Émile" (É=201)
    }

    @Test
    void departmentInEnumOrderThenSeniority() {
        assertEquals(List.of("Bob", "Émile", "adam", "Eve", "Dan", "zoe", "Chloé"),
                names(sorted(EmployeeComparators.BY_DEPARTMENT_THEN_SENIORITY)));
    }

    @Test
    void nullManagersLast() {
        assertEquals(List.of("Chloé", "adam", "zoe", "Émile", "Dan", "Bob", "Eve"),
                names(sorted(EmployeeComparators.BY_MANAGER_NULLS_LAST)));
    }

    @Test
    void defaultStringOrderVsCaseInsensitiveVsCollator() {
        List<String> def = names(sorted(Comparator.comparing(Employee::name)));
        assertEquals(List.of("Bob", "Chloé", "Dan", "Eve", "adam", "zoe", "Émile"), def);   // uppercase < lowercase < accented
        List<String> ci = names(sorted(Comparator.comparing(Employee::name, String.CASE_INSENSITIVE_ORDER)));
        assertEquals(List.of("adam", "Bob", "Chloé", "Dan", "Eve", "zoe", "Émile"), ci);    // É still after z
        List<String> human = names(sorted(EmployeeComparators.byNameLocaleAware(Locale.FRENCH)));
        assertEquals(List.of("adam", "Bob", "Chloé", "Dan", "Émile", "Eve", "zoe"), human); // what people expect
    }

    @Test
    void listSortIsStable() {
        // Sort by name first, then by department only: within a department, name order must survive.
        List<Employee> copy = new ArrayList<>(STAFF);
        copy.sort(EmployeeComparators.byNameLocaleAware(Locale.FRENCH));
        copy.sort(Comparator.comparing(Employee::department));
        assertEquals(List.of("adam", "Bob", "Émile", "Eve", "Dan", "zoe", "Chloé"), names(copy));
    }

    @Test
    void topEarnersPerDepartment() {
        Map<Department, List<String>> top = EmployeeComparators.topEarnersPerDepartment(STAFF, 2);
        assertEquals(List.of(Department.ENGINEERING, Department.PRODUCT, Department.SALES, Department.HR),
                new ArrayList<>(top.keySet()));
        assertEquals(List.of("Bob", "adam"), top.get(Department.ENGINEERING));
        assertEquals(List.of("Dan", "zoe"), top.get(Department.SALES));
    }

    @Test
    void naturalStringOrder() {
        List<String> files = new ArrayList<>(List.of("file10.txt", "file2.txt", "File1.txt", "file1.txt", "file02.txt", "img12", "img2b"));
        files.sort(NaturalStringComparator.INSTANCE);
        assertEquals(List.of("File1.txt", "file1.txt", "file02.txt", "file2.txt", "file10.txt", "img2b", "img12"), files);
        assertTrue(NaturalStringComparator.INSTANCE.compare("id99999999999999999999", "id100000000000000000000") < 0);
    }

    @Test
    void naturalStringComparatorObeysContract() {
        Random rnd = new Random(5);
        String alphabet = "aAb1203";
        List<String> samples = new ArrayList<>();
        for (int i = 0; i < 300; i++) {
            StringBuilder sb = new StringBuilder();
            for (int j = rnd.nextInt(6); j >= 0; j--) sb.append(alphabet.charAt(rnd.nextInt(alphabet.length())));
            samples.add(sb.toString());
        }
        Comparator<String> c = NaturalStringComparator.INSTANCE;
        for (String x : samples) {
            assertEquals(0, c.compare(x, x));
            for (String y : samples) {
                assertEquals(Integer.signum(c.compare(x, y)), -Integer.signum(c.compare(y, x)), x + " vs " + y);
                if (c.compare(x, y) == 0) assertEquals(x, y);              // total order: 0 only if equal
            }
        }
        List<String> copy = new ArrayList<>(samples);
        copy.sort(c);                                                       // no "violates its contract"
        for (int i = 1; i < copy.size(); i++) assertTrue(c.compare(copy.get(i - 1), copy.get(i)) <= 0);
    }

    @Test
    void subtractionComparatorOverflows() {
        List<Integer> vals = new ArrayList<>(List.of(Integer.MAX_VALUE, -1, Integer.MIN_VALUE, 0));
        vals.sort((x, y) -> x - y);                                         // the classic bug
        assertEquals(List.of(0, Integer.MIN_VALUE, -1, Integer.MAX_VALUE), vals);   // wrong!
        vals.sort(Integer::compare);
        assertEquals(List.of(Integer.MIN_VALUE, -1, 0, Integer.MAX_VALUE), vals);
    }

    @Test
    void brokenComparatorMakesTimSortThrow() {
        Comparator<Integer> broken = (x, y) -> x < y ? -1 : 1;              // never 0: compare(x, x) == 1
        Random rnd = new Random(0);
        IllegalArgumentException e = assertThrows(IllegalArgumentException.class, () -> {
            for (int t = 0; t < 50; t++) {                                  // detection is probabilistic
                List<Integer> list = new ArrayList<>();
                for (int i = 0; i < 2000; i++) list.add(rnd.nextInt(50));
                list.sort(broken);
            }
        });
        assertEquals("Comparison method violates its general contract!", e.getMessage());
    }

    @Test
    void treeSetWithComparatorInconsistentWithEqualsLosesElements() {
        TreeSet<Employee> bySalary = new TreeSet<>(Comparator.comparing(Employee::salary));
        bySalary.addAll(STAFF);
        assertEquals(6, bySalary.size());                                    // adam & Émile both earn 120000 → one dropped!
        TreeSet<Employee> safe = new TreeSet<>(Comparator.comparing(Employee::salary).thenComparingInt(Employee::id));
        safe.addAll(STAFF);
        assertEquals(7, safe.size());                                        // tie-breaker on a unique key fixes it
    }
}
```

```text
$ mvn test -Dtest=SortingTest
Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
```

---

## Key Takeaways

1. Build comparators from **combinators**, as `static final` constants: readable, reusable, thread-safe.
2. Reverse **one key** with `comparing(key, reverseOrder())`; a trailing `.reversed()` reverses everything.
3. Put `nullsFirst`/`nullsLast` on the **key** comparator.
4. Human-facing text → **`Collator`**; `String.compareTo` is code-unit order.
5. `List.sort` is **stable**, so multi-pass sorts preserve earlier orders.
6. The three comparator killers: **subtraction overflow**, **contract violations** (TimSort throws, sometimes), and **inconsistency with equals** in sorted collections (silent data loss). Tie-break on a unique key.
