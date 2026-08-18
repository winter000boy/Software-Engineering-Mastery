# #005 — Test & Refactor: CLI Inventory System

**Date:** 2026-08-18  
**Type:** Project — Phase 4 of 5  
**Topics:** JUnit 5, test doubles, refactoring, SOLID review

---

## Prerequisites

Complete **#004 Build CLI & Service** — working end-to-end CLI.

---

## Part A — Unit tests

Add JUnit 5 tests (Maven or Gradle project structure recommended).

### Repository tests (`InMemoryProductRepositoryTest`)

| Test | Scenario |
|------|----------|
| `save_newProduct_persistsAndReturns` | Happy path save |
| `save_duplicateId_throwsDuplicateProductException` | Duplicate guard |
| `findById_missing_returnsEmpty` | Optional empty |
| `update_existing_changesFields` | Update happy path |
| `update_missing_throwsProductNotFoundException` | Update guard |
| `deleteById_existing_removesProduct` | Delete happy path |
| `searchByName_caseInsensitive_partialMatch` | Search behavior |
| `findLowStock_returnsOnlyBelowThreshold` | Threshold filter |

### Service tests (`InventoryServiceTest`)

Use a **fake or mock** `ProductRepository` OR in-memory repo — justify choice.

| Test | Scenario |
|------|----------|
| `addProduct_invalidQuantity_throwsValidationException` | Negative qty |
| `addProduct_blankName_throwsValidationException` | Blank name |
| `getProduct_unknownId_throwsProductNotFoundException` | Not found |
| `listProducts_returnsSortedById` | Sorting contract |

---

## Part B — Refactor checklist

Apply at least **three** refactors; document each in a `REFACTOR_LOG.md` snippet in your solution:

1. **Extract method** — e.g. table formatting from CLI
2. **Replace magic numbers** — default low-stock threshold as named constant
3. **Introduce value object** (optional) — e.g. `Money` or `ProductId` if it simplifies validation
4. **Remove duplication** — repeated validation strings
5. **Improve naming** — any unclear method/variable names

For each refactor record:

- **Before:** one-line description
- **After:** one-line description
- **Why:** benefit (readability, testability, etc.)

---

## Part C — SOLID self-review

Answer in prose (5–8 sentences total):

1. Which SOLID principle does `ProductRepository` interface best demonstrate?
2. Does `InventoryCli` violate SRP? If slightly, what would you split?
3. Is `InventoryService` open for extension (new report types) without modification?

---

## Constraints

- Tests must not require manual CLI input (no `Scanner` in tests)
- All tests green before marking complete
- Refactors must preserve behavior — run tests after each change

---

## Acceptance criteria

- [ ] Minimum 10 automated tests, all passing
- [ ] At least 3 documented refactors
- [ ] SOLID self-review answered
- [ ] No regression in manual CLI smoke test
