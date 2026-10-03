# #032 — Plan & Design: LRU Cache from Scratch (LinkedHashMap + Manual) with Tests & Benchmark

**Date:** 2026-10-03  
**Type:** Project — Phase 1 of 5  
**Roadmap row:** 32 — P1 Java Core · Collections & Generics  
**Series:** #032 Plan & design → #033 Build (LinkedHashMap) → #034 Build (manual) → #035 Test & refactor → #036 Document & review

---

## Project Goal

Build a **bounded LRU (least-recently-used) cache** library in pure Java, twice:

1. on top of `LinkedHashMap` (the "use the platform" version)
2. **manually**, from a `HashMap` plus your own doubly linked list (the "understand the platform" version)

Both must meet one shared contract, pass the same test suite, and be benchmarked against each other and against a naive baseline.

This phase is **design only**: no implementation code beyond interfaces.

---

## What an LRU Cache Is

A cache holds at most `capacity` entries. When it's full and a **new** key is inserted, it evicts the entry that was **used least recently**. A "use" is a read or a write of that key. Typical uses are database query results, rendered pages, decoded images, and DNS lookups: anywhere recomputing is expensive and recent items are likely to be requested again.

```
capacity 3:  put a, put b, put c        LRU → MRU: [a, b, c]
             get a                      [b, c, a]        a refreshed
             put d   (full, d is new)   [c, a, d]        b evicted
```

---

## Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-1 | `get(key)` returns the value if present and marks the key **most recently used**; a miss returns "absent" (not `null`) |
| FR-2 | `put(key, value)` inserts or replaces, marks the key MRU, and returns the previous value if any |
| FR-3 | When a put of a **new** key would exceed capacity, evict **exactly one** entry: the LRU |
| FR-4 | Updating an **existing** key in a full cache must **not** evict anything |
| FR-5 | `remove(key)` deletes an entry. This is **not** an eviction |
| FR-6 | `containsKey`, `size` and a debugging snapshot of keys in LRU→MRU order must **not** change recency |
| FR-7 | An optional **eviction listener** is told the key and value of every eviction |
| FR-8 | Statistics: hits, misses, evictions, hit rate |
| FR-9 | `null` keys/values and capacity ≤ 0 are rejected |
| FR-10 | A **thread-safe** variant usable from many threads |

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-1 | `get`/`put`/`remove`: **O(1)** time for both real implementations |
| NFR-2 | One interface; implementations are interchangeable (program to the interface) |
| NFR-3 | Generic in `K`, `V` with correct wildcards (#026) |
| NFR-4 | Tests: one **shared contract suite** for every implementation, plus a **differential test** against a simple reference model |
| NFR-5 | JMH benchmark comparing the implementations at capacity 1 000 and 100 000 |
| NFR-6 | No external dependencies except JUnit 5 and JMH |

---

## Design Tasks

1. **Choose the data structures.** Fill in this table (Big-O for `get`/`put`/evict) and pick the two you'll build:

   | Option | get | put | evict LRU | Notes |
   |--------|-----|-----|-----------|-------|
   | `ArrayList` of keys + `HashMap` of values | | | | |
   | `TreeMap<timestamp, key>` + `HashMap` | | | | |
   | `PriorityQueue` by last-access time | | | | |
   | `LinkedHashMap` in access order | | | | |
   | `HashMap<K, Node>` + doubly linked list | | | | |

2. **Design the API.** Write the `Cache<K, V>` interface: method names, return types (`Optional` or `null`?), and the exact contract for each method (FR-1…FR-9).
3. **Inheritance or composition** for the `LinkedHashMap` version? Decide and justify (hint: EJ Item 18, #003).
4. **Sentinels:** for the manual linked list, will you use sentinel head/tail nodes? Compare the pointer code with and without them.
5. **Thread safety:** synchronise inside each implementation, or a separate **decorator**? Can a `ReadWriteLock` help when `get` mutates recency?
6. **Test plan:** list the contract tests, the differential-test design (what's the reference model?), and the concurrency test's invariants.
7. **Benchmark plan:** workload (read/write mix, key distribution), parameters, what you expect to see.
8. Draw the class diagram and a package layout.

---

## Deliverables

- Design doc (one page per section is plenty) with the completed table
- `Cache<K, V>` interface and supporting types (`CacheStats`, `EvictionListener`), as signatures with Javadoc
- Test plan and benchmark plan
- Risks and open questions

---

## Acceptance Criteria

- [ ] Data-structure table complete, with a justified choice
- [ ] API with a precise contract per method (what counts as a "use"?)
- [ ] Composition-vs-inheritance decision justified
- [ ] Thread-safety approach chosen, and the `ReadWriteLock` question answered
- [ ] Test plan covers FR-1…FR-10, with a reference model identified
- [ ] Benchmark plan with predictions written down

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
