# #024 — Implement a Simplified HashMap

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 24 — P1 Java Core · Collections & Generics  
**Topics:** hashing, separate chaining, power-of-two tables, hash spreading, load factor, resizing, null keys, fail-fast iteration, differential testing  
**Spec source:** #020 Part 3 (`HashMap` internals)

---

## Overview

Build `SimpleHashMap<K, V>`, a from-scratch hash table that follows the **real `java.util.HashMap` design** (#020), minus treeification. Don't use any JDK map or collection inside it. Arrays and `Objects` only.

---

## Required API

| Method | Behaviour |
|--------|-----------|
| `SimpleHashMap()` | Capacity 16, load factor 0.75; the **table is allocated lazily** on the first `put` |
| `SimpleHashMap(int initialCapacity, float loadFactor)` | Round the capacity up to a **power of two**; reject a negative capacity and a load factor that's ≤ 0 or `NaN` |
| `V put(K key, V value)` | Insert or replace; returns the previous value or `null` |
| `V get(Object key)` | Value or `null` |
| `boolean containsKey(Object key)` | Must be correct when the value is `null` |
| `V remove(Object key)` | Removes and returns the value, or `null` |
| `int size()`, `boolean isEmpty()`, `void clear()` | |
| `Iterator<Entry<K,V>> iterator()` | Visits every entry exactly once; **fail-fast** |
| `toString()` | `{k1=v1, k2=v2}` |

---

## Design Rules (from #020)

1. **`hash(key) = h ^ (h >>> 16)`**, where `null` → 0 (so one `null` key is supported, in bucket 0).
2. **Index = `hash & (length - 1)`**, where the table length is always a power of two.
3. **Chaining:** each bucket is a singly linked list of `Entry{final int hash; final K key; V value; Entry next}`. The **cached hash** is compared before `equals`.
4. **Append new entries at the tail** (like Java 8+).
5. **Resize** when `size > threshold` (`capacity × loadFactor`): double the table and **split each chain into lo/hi lists using `hash & oldCap`**, preserving order and without recomputing any `hashCode()`.
6. `Entry` is a **static** nested class (why?).
7. The generic array is created as `(Entry<K,V>[]) new Entry[n]`. Explain why `new Entry<K,V>[n]` won't compile.

---

## Tasks

1. Implement the class.
2. Tests, at minimum:
   - put / get / replace-returns-old / remove / size
   - `null` key; a `null` value vs an absent key (`containsKey`)
   - **resize points** for the default constructor: capacity must become 16, 32, 64, 128, 256 after puts **#1, #13, #25, #49, #97** (the same as the JDK, #020)
   - `tableSizeFor`: 10 → 16, 16 → 16, 17 → 32, 0 → 1; `(100, 0.75f)` gives 128 buckets
   - **collisions:** 100 keys with an identical `hashCode()` all land in one chain and are all still retrievable; remove the head, the middle and the tail of that chain
   - **resize split:** keys 5, 21, 37 share bucket 5 at capacity 16; after resizing to 32, 5 and 37 stay in bucket 5 and 21 moves to bucket 21
   - iteration visits all 1 000 entries once; modifying during iteration throws CME
   - **differential test:** 200 000 random `put`/`remove`/`get`/`containsKey` calls (including `null` keys) against `java.util.HashMap`, with identical results at every step
3. Explain: why can't `SimpleHashMap` be used where a `Map<K,V>` is expected? (That's the start of row 25's review.)

---

## Hints

- Stash the initial capacity in `threshold` until the first `put` allocates the table (that's what the JDK does).
- For the resize-split test, work out the bucket index of **every** key you insert by hand. A filler key can land in the bucket you're asserting on.
- `Objects.equals(a, b)` handles `null` keys cleanly.

---

## Acceptance Criteria

- [ ] All API methods; average O(1)
- [ ] Hash spreading, a power-of-two table, tail insertion, lo/hi resize split
- [ ] Resize points identical to the JDK's
- [ ] `null` key and `null` values correct
- [ ] Collision and resize-split tests pass
- [ ] Differential test (200 000 operations) passes

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
