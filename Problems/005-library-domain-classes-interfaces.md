# #005 — Model a Library Domain with Classes & Interfaces

**Date:** 2026-09-24  
**Type:** Build  
**Roadmap row:** 5 — P1 Java Core · Java runtime & OOP foundations  
**Topics:** domain modelling, abstract classes vs interfaces, interface segregation, composition vs inheritance, value objects vs entities, immutability, `equals`/`hashCode`, enums as strategies, dependency injection (`Clock`), custom exceptions

---

## Overview

Model the core of a small public library **in pure Java** (no frameworks, no database, everything in memory). The goal isn't a big app. It's to make **deliberate OOP design decisions** and be able to justify each one: which things are classes, which are interfaces, where inheritance fits and where composition fits, what is immutable, and how equality works.

This is a warm-up for the CLI Inventory project (#007–#011). Keep it smaller, but make the design sharper.

---

## Domain Description

A library holds **items**. Every item has an id (one per **physical copy**) and a title.

| Item type | Extra attributes | Borrowable? | Loan period | Late fee / day |
|-----------|------------------|:---:|---:|---:|
| **Book** | ISBN-13, one or more authors, publication year | ✅ | 21 days | 25¢ |
| **DVD** | runtime in minutes | ✅ | 7 days | 100¢ |
| **Reference book** (encyclopedia, dictionary) | ISBN-13 | ❌ (read in library only) | — | — |

**Members** have an id, a name and a **membership tier**:

| Tier | Max active loans | Loan period multiplier | Late fees |
|------|---:|---:|-----------|
| Student | 3 | ×1.0 | charged |
| Adult | 5 | ×1.0 | charged |
| Premium | 10 | ×2.0 | **waived** |

A member can change tier at any time (upgrade/downgrade).

A **loan** records which item was borrowed by whom, when, when it's due, and when it was returned.

---

## Functional Requirements

1. **Catalogue:** add items (reject duplicate ids), find by id, search by title (case-insensitive substring), list the books by a given author.
2. **Members:** register (reject duplicate ids), find by id.
3. **Borrow** `(memberId, itemId)`:
   - the item must exist and be **borrowable**, otherwise reject
   - the item must not already be on loan, otherwise reject
   - the member must be below their tier's loan limit, otherwise reject
   - due date = today + round(item loan period × tier multiplier)
4. **Return** `(itemId)`: close the loan, keep it in history, make the item available again.
5. **Late fees:** `days late × item's daily fee`, or 0 if the member's tier waives fees.
6. **Reports:** a member's active loans; all overdue loans as of today.

---

## Design Requirements (the real point of the exercise)

| # | Requirement |
|---|-------------|
| D1 | Use **at least one interface** and **at least one abstract class**, and explain why each concept is an interface or an abstract class |
| D2 | "Borrowable" is a **capability**. A reference book must **not** be forced to implement borrowing methods it can't honour |
| D3 | Membership rules must be swappable **without** `if (tier == STUDENT) … else if …` chains anywhere. A new tier = new code, not edited code |
| D4 | **Value objects** (ISBN, author, ids) are immutable and equal by value. **Entities** (item copy, member) are equal by **identity (id)** |
| D5 | ISBN-13 is **validated** (13 digits + checksum) when created; hyphens and spaces are allowed in the input |
| D6 | No caller can mutate an object's internal collections (e.g. a book's author list) |
| D7 | "Today" must be controllable in tests. Don't call `LocalDate.now()` with no arguments in business logic |
| D8 | Business-rule violations are **specific unchecked exceptions** with one common base type |
| D9 | Every class is as closed as possible: `final` where extension isn't designed for, `private` fields |

---

## Tasks

1. **Model first, code second.** Draw a class diagram (boxes, `implements`/`extends`/`has-a` arrows) for all types. Mark each type as *interface*, *abstract class*, *final class*, *record* or *enum*.
2. Write down, for each of the design decisions D1–D9, **which construct you chose and why** (1–2 sentences each).
3. Implement the model in packages `domain`, `policy`, `service`, `exception`.
4. Write a `Library` service class with the operations above.
5. Write JUnit 5 tests covering at least:
   - ISBN checksum validation and value equality (`"978-0-13-468599-1"` equals `"9780134685991"`)
   - entity equality by id
   - defensive copy of the author list
   - due date for a Student book loan vs a Premium DVD loan
   - reference book can't be borrowed
   - the same item can't be lent twice
   - loan limit enforced, and lifted after upgrading tier
   - a late return charges the right fee; Premium pays 0
6. Write a small `main` demo that exercises the happy path and one rejection.

---

## Constraints

- Java 17+ (records, pattern-matching `instanceof`, sealed types allowed).
- No Lombok, no Spring, no persistence. Plain `Map`/`List` in memory.
- No `public` mutable fields. No getters that return mutable internals.
- Don't use `instanceof` checks on *tier* anywhere. (Using `instanceof Borrowable` for a capability check is fine; explain why that's different.)

---

## Stretch Goals

- **Reservations:** a member can place a hold on an item that's on loan; when it's returned, it's held for the first reserver for 3 days.
- **Renewals:** extend a loan once, unless someone has reserved the item.
- Make `LibraryItem` a **sealed** hierarchy and compare the result with the open abstract class.
- Replace the fee's `long` cents with a `Money` value object (`BigDecimal` + `Currency`).

---

## Acceptance Criteria

- [ ] Class diagram drawn; every type labelled with its construct
- [ ] D1–D9 each justified in writing
- [ ] `ReferenceBook` does **not** implement `Borrowable`
- [ ] Adding a 4th tier requires **no** change to `Library` or `Member`
- [ ] `Isbn`, `Author`, ids: immutable, value equality, validated on construction
- [ ] `Book.authors()` can't be used to modify the book
- [ ] `Library` takes a `Clock` in its constructor; tests move time forward
- [ ] All tests in task 5 pass
- [ ] Demo `main` runs

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
