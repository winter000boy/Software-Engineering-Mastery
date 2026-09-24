# #027 — Sort Objects with Custom Comparators

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 27 — P1 Java Core · Collections & Generics  
**Topics:** `Comparable` vs `Comparator`, comparator combinators, multi-key sorts, reversing single keys, null handling, locale-aware text (`Collator`), stable sorting, natural ("human") string order, comparator contract violations  
**Theory:** #029 (Comparable vs Comparator)

---

## Overview

Model employees and sort them in every way an HR dashboard needs. Then break comparators deliberately and watch what happens, so you recognise these bugs in real code.

---

## Data Model

```java
enum Department { ENGINEERING, PRODUCT, SALES, HR }          // declaration order = the business order

record Employee(int id, String name, Department department, BigDecimal salary,
                LocalDate hireDate, String manager)           // manager is null for top-level people
```

Test data (use exactly this):

| id | name | department | salary | hired | manager |
|---:|------|------------|-------:|-------|---------|
| 5 | zoe | SALES | 70 000 | 2021-03-01 | Bob |
| 1 | Bob | ENGINEERING | 150 000 | 2015-06-01 | *(null)* |
| 3 | Émile | ENGINEERING | 120 000 | 2018-01-15 | Bob |
| 2 | adam | ENGINEERING | 120 000 | 2019-09-09 | Bob |
| 4 | Chloé | HR | 80 000 | 2020-02-02 | Bob |
| 6 | Dan | SALES | 90 000 | 2017-11-11 | zoe |
| 7 | Eve | PRODUCT | 110 000 | 2016-04-04 | *(null)* |

(The mixed case and accents are deliberate.)

---

## Tasks

**Predict each resulting order on paper before running it.**

1. **Natural order:** make `Employee` `Comparable` by `id`. Then make it **consistent with `equals`** (a record's `equals` compares every component) and explain why that matters.
2. **Salary descending, then name ascending.** Reverse *only* the salary key.
3. **Department in enum order, then most senior first** (earliest `hireDate`).
4. **By manager with `null` managers last**, then by name.
5. **Names three ways:** (a) `String` natural order, (b) `String.CASE_INSENSITIVE_ORDER`, (c) a French `Collator`. Explain every difference.
6. **Stability:** sort by name (Collator), then sort again by department only. Show that names stay in order within each department, and explain why.
7. **Top-2 earners per department**, departments in enum order (streams + `groupingBy` + `EnumMap`).
8. **Natural string order:** write a `Comparator<String>` so that `file2.txt < file10.txt`, `File1.txt` / `file1.txt` / `file02.txt` / `file2.txt` sort sensibly, and digit runs longer than a `long` work. It must be a **total order** (0 only for equal strings). Prove the contract with a randomised test (antisymmetry and `compare == 0` ⇒ equal).
9. **Break it on purpose:**
   - (a) sort `[MAX_VALUE, -1, MIN_VALUE, 0]` with `(x, y) -> x - y`. What happens, and why?
   - (b) sort random lists of 2 000 small integers with `(x, y) -> x < y ? -1 : 1`. What exception, and why only sometimes?
   - (c) put all employees in a `TreeSet` ordered **only by salary**. How many survive? Fix it.

---

## Constraints

- Build comparators with `Comparator` combinators (`comparing`, `thenComparing`, `reverseOrder`, `nullsLast`…), not hand-written `compare` bodies, except for task 8.
- Store reusable comparators as `static final` constants.
- Use `List.sort` on a **copy**; never sort the shared test data in place.

---

## Acceptance Criteria

- [ ] Tasks 1–8 implemented, with a JUnit test asserting the exact expected order for each
- [ ] Task 5 difference explained (UTF-16 code units vs case folding vs collation)
- [ ] Task 6 stability demonstrated
- [ ] Task 8 comparator passes the contract test
- [ ] Task 9: all three failures reproduced in tests and explained; (c) fixed with a tie-breaker

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
