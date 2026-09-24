# #007 — Solution: Plan & Design — CLI Inventory System

**Date:** 2026-08-18  
**Problem:** [007-cli-inventory-plan-design.md](../problems/007-cli-inventory-plan-design.md)

---

## Task 1 — Class diagram (ASCII)

```
┌─────────────┐
│    Main     │
└──────┬──────┘
       │ creates
       ▼
┌─────────────┐      uses       ┌──────────────────┐
│ InventoryCli│ ──────────────► │ InventoryService │
└─────────────┘                 └────────┬─────────┘
                                         │ uses
                                         ▼
                              ┌──────────────────────┐
                              │ «interface»          │
                              │ ProductRepository    │
                              └──────────┬───────────┘
                                         △
                                         │ implements
                              ┌──────────┴───────────┐
                              │ InMemoryProduct      │
                              │ Repository           │
                              └──────────┬───────────┘
                                         │ stores
                                         ▼
                              ┌──────────────────────┐
                              │ Product (domain)     │
                              └──────────────────────┘

Exceptions: ValidationException, ProductNotFoundException,
            DuplicateProductException (thrown from repo/service)
```

**Dependency rule:** outer layers depend inward; domain has no outward deps.

---

## Task 2 — Product model

| Field | Type | Validation |
|-------|------|------------|
| `id` | `String` | Non-null, non-blank, immutable |
| `name` | `String` | Non-null, non-blank |
| `quantity` | `int` | >= 0 |
| `unitPrice` | `BigDecimal` | >= 0, scale 2 |

**Why `BigDecimal`?** Money arithmetic with `double` introduces rounding errors (`0.1 + 0.2`). Inventory prices need predictable decimal behavior.

**Identity:** `equals` / `hashCode` on `id` only.

---

## Task 3 — CLI command map

| Choice | Action | Prompts | Error flows |
|--------|--------|---------|-------------|
| 1 add | Create product | id, name, qty, price | Duplicate id → message, return to menu |
| 2 list | Print table | — | Empty → "No products" |
| 3 find | Show one | id | Not found → message |
| 4 update | Change qty/price | id, new qty, new price | Not found / invalid numbers |
| 5 delete | Remove | id | Not found → message |
| 6 search | Partial name | name fragment | Empty fragment → warn, no search |
| 7 low-stock | Report | threshold (default 5) | — |
| 8 exit | Quit | — | Close scanner |

---

## Task 4 — Exception strategy

| Exception | Layer | When |
|-----------|-------|------|
| `ValidationException` | Service | Blank id/name, negative qty/price |
| `DuplicateProductException` | Repository | Save with existing id |
| `ProductNotFoundException` | Repository / Service | Update/delete/get missing id |

CLI catches these and prints short messages; unexpected errors may log stack trace in dev only.

---

## Task 5 — Test plan (phase 5)

1. `save_newProduct_persistsAndReturns` — save valid product, findById present
2. `save_duplicateId_throws` — second save same id fails
3. `update_existing_changesFields` — qty and price updated
4. `update_missing_throws` — unknown id on update
5. `deleteById_existing_removes` — gone after delete
6. `searchByName_caseInsensitive` — "widget" finds "Super Widget"
7. `findLowStock_belowThreshold` — only low qty returned
8. `addProduct_invalidQuantity_throws` — service rejects negative qty
9. `getProduct_unknown_throws` — service not found path
10. `listProducts_sortedById` — stable sort order

---

## Design decisions summary

- **Repository interface** enables swap to file/DB later without touching service/CLI.
- **Service layer** owns validation and sorting policy.
- **CLI** owns I/O only — SRP for testability.
