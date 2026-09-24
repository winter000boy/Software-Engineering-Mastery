# Effective Java — Daily Problems Index

> **Project home:** [README.md](README.md) · **Study notes:** [INDEX.md](INDEX.md)

| # (CSV sr.no) | Date | Topic | Type | Problem | Solution |
|---|------|-------|------|---------|----------|
| 002 | 2026-09-24 | Set Up JDK + Build Tool (Maven/Gradle), Hello World | Build | [002-jdk-build-tool-hello-world.md](problems/002-jdk-build-tool-hello-world.md) | [002-jdk-build-tool-hello-world.md](solutions/002-jdk-build-tool-hello-world.md) |
| 005 | 2026-09-24 | Model a Library Domain with Classes & Interfaces | Build | [005-library-domain-classes-interfaces.md](problems/005-library-domain-classes-interfaces.md) | [005-library-domain-classes-interfaces.md](solutions/005-library-domain-classes-interfaces.md) |
| 006 | 2026-08-18 | Solve 3 Exercises on JVM & OOP | Exercise | [006-jvm-oop-three-exercises.md](problems/006-jvm-oop-three-exercises.md) | [006-jvm-oop-three-exercises.md](solutions/006-jvm-oop-three-exercises.md) |
| 007 | 2026-08-18 | Plan & Design: CLI Inventory System (Pure OOP) | Project | [007-cli-inventory-plan-design.md](problems/007-cli-inventory-plan-design.md) | [007-cli-inventory-plan-design.md](solutions/007-cli-inventory-plan-design.md) |
| 008 | 2026-08-18 | Build: CLI Inventory System — Core Domain & Repository | Project | [008-cli-inventory-build-core.md](problems/008-cli-inventory-build-core.md) | [008-cli-inventory-build-core.md](solutions/008-cli-inventory-build-core.md) |
| 009 | 2026-08-18 | Build (cont.): CLI Inventory System — Service & CLI | Project | [009-cli-inventory-build-cli-service.md](problems/009-cli-inventory-build-cli-service.md) | [009-cli-inventory-build-cli-service.md](solutions/009-cli-inventory-build-cli-service.md) |
| 010 | 2026-08-18 | Test & Refactor: CLI Inventory System | Project | [010-cli-inventory-test-refactor.md](problems/010-cli-inventory-test-refactor.md) | [010-cli-inventory-test-refactor.md](solutions/010-cli-inventory-test-refactor.md) |
| 011 | 2026-08-18 | Document & Review: CLI Inventory System | Project | [011-cli-inventory-document-review.md](problems/011-cli-inventory-document-review.md) | [011-cli-inventory-document-review.md](solutions/011-cli-inventory-document-review.md) |
| 018 | 2026-09-24 | Benchmark ArrayList vs LinkedList (JMH + JOL) | Build | [018-benchmark-arraylist-vs-linkedlist.md](problems/018-benchmark-arraylist-vs-linkedlist.md) | [018-benchmark-arraylist-vs-linkedlist.md](solutions/018-benchmark-arraylist-vs-linkedlist.md) |
| 021 | 2026-09-24 | Implement Your Own Dynamic Array | Build | [021-implement-dynamic-array.md](problems/021-implement-dynamic-array.md) | [021-implement-dynamic-array.md](solutions/021-implement-dynamic-array.md) |
| 022 | 2026-09-24 | Collections Exercises: Top-K, First Unique, Sliding Window Max, Calendar | Exercise | [022-collections-exercises.md](problems/022-collections-exercises.md) | [022-collections-exercises.md](solutions/022-collections-exercises.md) |
| 024 | 2026-09-24 | Implement a Simplified HashMap | Build | [024-implement-simplified-hashmap.md](problems/024-implement-simplified-hashmap.md) | [024-implement-simplified-hashmap.md](solutions/024-implement-simplified-hashmap.md) |
| 025 | 2026-09-24 | Review & Refactor the HashMap (AbstractMap, measured collisions) | Review | [025-review-refactor-hashmap.md](problems/025-review-refactor-hashmap.md) | [025-review-refactor-hashmap.md](solutions/025-review-refactor-hashmap.md) |
| 027 | 2026-09-24 | Sort Objects with Custom Comparators | Build | [027-sort-with-custom-comparators.md](problems/027-sort-with-custom-comparators.md) | [027-sort-with-custom-comparators.md](solutions/027-sort-with-custom-comparators.md) |
| 030 | 2026-09-24 | Generic Container Class: Multiset<E> | Build | [030-generic-container-class.md](problems/030-generic-container-class.md) | [030-generic-container-class.md](solutions/030-generic-container-class.md) |

**Rows covered so far: #002, #005–#011, #018, #021, #022, #024, #025, #027, #030** — see `Roadmap.csv` for every row still not covered.

---

## How to use

1. Send me one or more problems each day (exercises, projects, or interview-style tasks).
2. I create a `.md` file in `problems/` numbered after the matching `sr. no` row in [Roadmap.csv](Roadmap.csv), and a matching solution in `solutions/`.
3. I update this index automatically — serial numbers are **never reused for a different row**, but are **not contiguous** (the number is the CSV row, not creation order).

## Folder layout

```
Effective Java/
├── PROBLEMS-INDEX.md     ← this file
├── problems/             ← problem statements
└── solutions/            ← worked solutions (same serial + slug)
```

## Problem file format

Each problem includes:

- **Problem statement** — clear description with examples
- **Constraints** — input limits, edge cases
- **Topics / tags** — e.g. arrays, hash map, OOP, concurrency
- **Acceptance criteria** — checklist to know when done

## Solution file format

Each solution includes:

- **Link back** to the problem file
- **Approach** — intuition and step-by-step strategy
- **Solution** — Java code with explanation
- **Complexity** — time and space (when relevant)
- **Key takeaways** — condensed bullets for review

## Project series: CLI Inventory (#007–#011)

Five linked phases — work in order:

| Phase | Focus |
|-------|-------|
| 007 | Plan & design |
| 008 | Domain + repository |
| 009 | Service + CLI |
| 010 | Tests + refactor |
| 011 | Document + review |
