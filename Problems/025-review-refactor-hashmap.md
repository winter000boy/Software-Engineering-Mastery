# #025 — Review Previous Day's Build; Refactor Weak Spots

**Date:** 2026-09-24  
**Type:** Review + Refactor  
**Roadmap row:** 25 — P1 Java Core · Collections & Generics · *Flex*  
**Reviews:** yesterday's build, `SimpleHashMap` (#024), with notes on #021  
**Topics:** code review, API contracts (`Map`, `Map.Entry`), `AbstractMap` skeletal implementation (EJ Item 20), live views, measuring a weakness before fixing it, differential testing

---

## Overview

Yesterday's `SimpleHashMap` passes its tests, but "tests pass" isn't "good". Today you **review it like a senior colleague would**, find its weak spots, **measure** the worst one, and **refactor** into a version that's a genuine `java.util.Map`, without breaking anything.

---

## Part 1 — Review

Review `SimpleHashMap` against this checklist. For **each** item, record *OK* or a *finding* (what's wrong, its impact, and a proposed fix).

| # | Review question |
|---|-----------------|
| R1 | **API fitness:** can it be passed to code that expects a `Map<K,V>`? Does it support `equals`/`hashCode` against other maps, `keySet()`/`values()`/`entrySet()`, and `getOrDefault`/`merge`/`computeIfAbsent`? |
| R2 | **Iterator completeness:** can a caller filter the map in place while iterating? |
| R3 | **Entry contract:** does the exposed entry type follow `Map.Entry`'s `equals`/`hashCode`/`setValue` contract? |
| R4 | **Worst-case performance:** what happens when many keys share a `hashCode`? *Measure it* against `java.util.HashMap`. |
| R5 | **Duplication:** is the chain-walking logic written more than once? |
| R6 | **Input validation:** constructor arguments, `NaN` load factor? |
| R7 | **Memory:** does the table ever shrink? What does iteration cost after `clear()`? |
| R8 | **Tests:** do they compare against a trusted reference? Did any test contain a bug? |

---

## Part 2 — Measure Before You Fix

Write a small program comparing `SimpleHashMap` and `java.util.HashMap` when **every key has the same `hashCode()`** (but keys are `Comparable`). For n ∈ {1 000, 10 000, 50 000} report:

- the time to insert n keys
- the time for 1 000 lookups **spread across the whole key range**

Explain the growth rate of each implementation. (Hint: what does `HashMap` do with a bucket of more than 8 entries, #020 §3.8?)

---

## Part 3 — Refactor

Create `BetterHashMap<K, V>` that fixes the findings you decide to fix:

1. **`extends AbstractMap<K, V>`**. Find out which *single* method `AbstractMap` requires, and which methods you **must still override** because the inherited versions are O(n).
2. `entrySet()` returns a **live view** whose iterator supports `remove()`; `contains`/`remove` on the view are O(1).
3. The entry type implements `Map.Entry` correctly: `equals` against **any** `Map.Entry`, `hashCode = keyHash ^ valueHash`, and a write-through `setValue`.
4. One shared chain-walking helper, used by `get`/`containsKey`/`put`/`remove`.
5. A copy constructor `BetterHashMap(Map<? extends K, ? extends V>)` that **presizes** so it never resizes while copying.
6. Decide explicitly whether to implement treeification now, and record the decision and the reason.

---

## Part 4 — Prove Nothing Broke

- Keep all of #024's tests passing (the core algorithm is unchanged).
- New tests:
  - `BetterHashMap` equals a `java.util.HashMap` with the same mappings **in both directions**, with equal `hashCode`s
  - `merge`, `getOrDefault`, `computeIfAbsent` and `computeIfPresent` (returning `null` removes) work with **no extra code**
  - `entrySet().iterator().remove()`, `keySet().removeIf(...)`, `values().remove(...)`
  - `setValue` through an entry writes through
  - fail-fast on outside modification
  - **differential test:** 100 000 random operations *including* `merge`, view removals and bulk `values().removeIf` against `java.util.HashMap`

---

## Acceptance Criteria

- [ ] Review table completed: every item has OK or a finding
- [ ] Collision cost measured at three sizes, with the growth rates explained
- [ ] `BetterHashMap` is a `Map`; the O(n) `AbstractMap` defaults are overridden
- [ ] Live `entrySet` with iterator `remove()`
- [ ] Contract-correct `Map.Entry`
- [ ] Treeification decision recorded
- [ ] Old + new tests green, including the differential test

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
