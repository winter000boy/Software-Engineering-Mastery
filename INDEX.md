# Effective Java — Study Notes Index

> **Project home:** [README.md](README.md) · **Daily problems:** [PROBLEMS-INDEX.md](PROBLEMS-INDEX.md)

| # (CSV sr.no) | Date | Topic | File |
|---|------|-------|------|
| 001 | 2026-08-18 | JVM / JRE / JDK, Bytecode & How Java Runs | [001-jvm-jre-jdk-bytecode-how-java-runs.md](notes/001-jvm-jre-jdk-bytecode-how-java-runs.md) |
| 003 | 2026-09-24 | Read: Effective Java — Reading Notes, Ch. 2–4 (Items 1–25) | [003-effective-java-reading-notes-ch2-ch4.md](notes/003-effective-java-reading-notes-ch2-ch4.md) |
| 004 | 2026-08-18 | Memory Areas: Heap, Stack, Metaspace & Class Loading | [004-memory-areas-heap-stack-metaspace-class-loading.md](notes/004-memory-areas-heap-stack-metaspace-class-loading.md) |
| 012 | 2026-08-18 | Java Memory Model Overview & Stack vs Heap (Study) | [012-jmm-overview-stack-vs-heap-study.md](notes/012-jmm-overview-stack-vs-heap-study.md) |
| 013 | 2026-08-18 | Java Memory Model & Stack vs Heap (Practice / Apply) | [013-jmm-stack-vs-heap-practice-apply.md](notes/013-jmm-stack-vs-heap-practice-apply.md) |
| 014 | 2026-09-24 | Weekly Review + Anki (45 cards) + Fix Gaps — Week 1 | [014-week1-review-anki-fix-gaps.md](notes/014-week1-review-anki-fix-gaps.md) |
| 015 | 2026-09-24 | Paper: *On Designing and Deploying Internet-Scale Services* (Hamilton) | [015-paper-hamilton-designing-deploying-internet-scale-services.md](notes/015-paper-hamilton-designing-deploying-internet-scale-services.md) |
| 016 | 2026-09-24 | Paper Summary + 3 Takeaways + Plan for Week 2 | [016-hamilton-paper-summary-takeaways-plan-next-week.md](notes/016-hamilton-paper-summary-takeaways-plan-next-week.md) |
| 017 | 2026-09-24 | List / Set / Map Internals & When to Use Each | [017-list-set-map-internals-when-to-use.md](notes/017-list-set-map-internals-when-to-use.md) |
| 019 | 2026-09-24 | Read: Effective Java + Java Docs — Collections & Generics | [019-effective-java-java-docs-collections-generics-reading.md](notes/019-effective-java-java-docs-collections-generics-reading.md) |
| 020 | 2026-09-24 | ArrayList vs LinkedList; HashMap Internals (Buckets, Resize, Treeify) | [020-arraylist-vs-linkedlist-hashmap-internals.md](notes/020-arraylist-vs-linkedlist-hashmap-internals.md) |
| 023 | 2026-09-24 | TreeMap / LinkedHashMap; Iteration Order | [023-treemap-linkedhashmap-iteration-order.md](notes/023-treemap-linkedhashmap-iteration-order.md) |
| 026 | 2026-09-24 | Generics, Bounded Types, Wildcards (PECS) | [026-generics-bounded-types-wildcards-pecs.md](notes/026-generics-bounded-types-wildcards-pecs.md) |
| 028 | 2026-09-24 | Anki Cards for Collections (82 cards) | [028-anki-cards-collections.md](notes/028-anki-cards-collections.md) |
| 029 | 2026-09-24 | Comparable vs Comparator | [029-comparable-vs-comparator.md](notes/029-comparable-vs-comparator.md) |
| 037 | 2026-10-03 | Study: Big-O Complexity Review + Collections Cheat Sheet | [037-big-o-review-collections-cheat-sheet.md](notes/037-big-o-review-collections-cheat-sheet.md) |
| 038 | 2026-10-03 | Practice: Big-O (exercises + measured doubling experiment) | [038-big-o-practice-apply.md](notes/038-big-o-practice-apply.md) |
| 039 | 2026-10-03 | Weekly Review + Anki (32 new cards) + Fix Gaps — Week 2 | [039-week2-review-anki-fix-gaps.md](notes/039-week2-review-anki-fix-gaps.md) |
| 040 | 2026-10-03 | Paper: *The Google File System* (Ghemawat, Gobioff, Leung) | [040-paper-google-file-system.md](notes/040-paper-google-file-system.md) |
| 041 | 2026-10-12 | GFS Summary + 3 Takeaways + Plan for Week 3 (Concurrency I) | [041-gfs-summary-takeaways-plan-next-week.md](notes/041-gfs-summary-takeaways-plan-next-week.md) |
| 042 | 2026-10-12 | Threads, Runnable, Thread Lifecycle | [042-threads-runnable-lifecycle.md](notes/042-threads-runnable-lifecycle.md) |
| 044 | 2026-10-12 | Read: *Java Concurrency in Practice*, Ch. 1–4 | [044-java-concurrency-in-practice-ch1-4.md](notes/044-java-concurrency-in-practice-ch1-4.md) |
| 045 | 2026-10-12 | `synchronized`, Intrinsic Locks, Monitors | [045-synchronized-intrinsic-locks-monitor.md](notes/045-synchronized-intrinsic-locks-monitor.md) |
| 048 | 2026-10-12 | `volatile`, Visibility, Happens-Before | [048-volatile-visibility-happens-before.md](notes/048-volatile-visibility-happens-before.md) |

**Rows covered so far: #001, #003, #004, #012–#017, #019, #020, #023, #026, #028, #029, #037–#042, #044, #045, #048** — see `Roadmap.csv` for every row still not covered.

---

## How to use

1. Send me one or more topics each day (e.g. *"Item 1: Consider static factory methods"*, or any Java/concept topic).
2. I create a `.md` file in `notes/` numbered after the matching `sr. no` row in [Roadmap.csv](Roadmap.csv), and update this index.
3. Serial numbers are **never reused for a different row** — but they are **not contiguous**: the number is the CSV row, not the order notes were written.

## Note file format

Each note includes:

- **At a glance** — 5–10 bullet summary (copy straight to handwritten notes)
- **Detailed explanation** — full coverage with examples
- **Key terms & definitions**
- **Code examples** (when relevant)
- **Common pitfalls / interview angles**
- **Handwritten notes cheat sheet** — condensed one-liners at the end
