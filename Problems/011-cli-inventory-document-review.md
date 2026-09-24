# #011 — Document & Review: CLI Inventory System

**Date:** 2026-08-18  
**Type:** Project — Phase 5 of 5  
**Topics:** documentation, code review, architecture summary, retrospective

---

## Prerequisites

Complete **#010 Test & Refactor** — tests green, refactors logged.

---

## Part A — Project README

Write `README.md` for the inventory project covering:

1. **Overview** — what the app does (2–3 sentences)
2. **Requirements** — JDK version, build tool (Maven/Gradle), how to compile
3. **Run instructions** — exact commands
4. **CLI menu** — screenshot or copy of sample session
5. **Architecture** — layer diagram (CLI → Service → Repository → Domain)
6. **Design decisions** — BigDecimal vs double, in-memory choice, exception strategy
7. **Testing** — how to run tests
8. **Future improvements** — file persistence, JSON export, concurrency

---

## Part B — JavaDoc

Add JavaDoc to all **public** types and methods in:

- `ProductRepository`
- `InventoryService`
- Custom exceptions (class-level)

Keep descriptions concise — one sentence per member where obvious.

---

## Part C — Code review checklist

Perform a self-review and fill this table honestly:

| Area | Pass? | Notes |
|------|-------|-------|
| No business logic in CLI | | |
| No persistence logic in service | | |
| Exceptions used vs raw strings | | |
| Input validation centralized | | |
| equals/hashCode on domain entity | | |
| Tests cover happy + sad paths | | |
| No dead code / commented blocks | | |
| Consistent formatting | | |

Fix any **Fail** items before marking complete.

---

## Part D — Retrospective (short)

Answer:

1. What was hardest — design, CLI, tests, or refactor?
2. One OOP concept this project reinforced most strongly?
3. One thing you would redesign if starting over?

---

## Part E — Handoff artifact

Produce a **one-page architecture summary** (can live in README or `ARCHITECTURE.md`):

- Layer responsibilities (one bullet each)
- Dependency graph (ASCII)
- Extension points (e.g. swap `InMemoryProductRepository` for file-based)

---

## Acceptance criteria

- [ ] README complete and accurate
- [ ] Public API JavaDoc present
- [ ] Code review checklist all Pass (or fixed)
- [ ] Retrospective answered
- [ ] Architecture summary included

---

## Project complete when

All phases #007–#011 acceptance criteria met and linked from `PROBLEMS-INDEX.md`.
