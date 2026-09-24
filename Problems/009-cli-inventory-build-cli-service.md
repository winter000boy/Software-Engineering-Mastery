# #004 — Build (cont.): CLI Inventory System — Service & CLI

**Date:** 2026-08-18  
**Type:** Project — Phase 3 of 5  
**Topics:** service layer, dependency injection, command-loop CLI

---

## Prerequisites

Complete **#003 Build Core** — domain and repository working.

---

## Build tasks

### Task 1 — `InventoryService`

Orchestrate business rules on top of `ProductRepository`:

| Method | Behavior |
|--------|----------|
| `addProduct(id, name, qty, price)` | Validate inputs, build `Product`, delegate save |
| `listProducts()` | Return sorted list (by id or name — pick one, document) |
| `getProduct(id)` | Return or throw `ProductNotFoundException` |
| `updateProduct(id, newQty, newPrice)` | Load, apply changes, save |
| `removeProduct(id)` | Delegate delete |
| `searchByName(fragment)` | Delegate search |
| `lowStockReport(threshold)` | Delegate findLowStock |

- Use **constructor injection** for `ProductRepository`
- Keep validation in service (single place for business rules)

### Task 2 — `InventoryCli`

Interactive loop reading from `Scanner`:

```
=== Inventory CLI ===
1. Add product
2. List all
3. Find by id
4. Update product
5. Delete product
6. Search by name
7. Low stock report
8. Exit
Choice:
```

- Catch expected exceptions; print user-friendly messages (do not dump stack traces for validation errors)
- Format list output as aligned columns: ID | Name | Qty | Price

### Task 3 — `Main`

Wire dependencies manually (pure OOP, no DI framework):

```java
ProductRepository repo = new InMemoryProductRepository();
InventoryService service = new InventoryService(repo);
InventoryCli cli = new InventoryCli(service);
cli.run();
```

### Task 4 — Edge cases to handle in CLI

| Scenario | Expected UX |
|----------|-------------|
| Duplicate id on add | Message: product already exists |
| Invalid number input | Reprompt or show error, do not crash |
| Empty search | Show all or warn — document choice |
| Exit | Clean shutdown message |

---

## Constraints

- CLI must not call repository directly — only through service
- No static service/repository singletons
- Scanner closed on exit

---

## Acceptance criteria

- [ ] Full CRUD works through CLI
- [ ] Service has no `Scanner` / System.in logic
- [ ] CLI has no `Map` or direct storage access
- [ ] Manual end-to-end test: add 3 products, update 1, delete 1, search, exit
