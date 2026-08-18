# #002 — Plan & Design: CLI Inventory System (Pure OOP)

**Date:** 2026-08-18  
**Type:** Project — Phase 1 of 5  
**Topics:** OOP design, SOLID, layering, CLI UX planning

---

## Project goal

Build a **command-line inventory management system** in pure Java OOP:

- In-memory storage (no database)
- Full CRUD on products
- Clean separation: domain → repository → service → CLI
- Unit tests and refactor passes in later phases

This phase is **design only** — no implementation yet.

---

## Functional requirements

| ID | Requirement |
|----|-------------|
| FR-1 | Add a product (id, name, quantity, unit price) |
| FR-2 | List all products (formatted table) |
| FR-3 | Find product by id |
| FR-4 | Update quantity and/or price |
| FR-5 | Delete product by id |
| FR-6 | Search products by name (partial match, case-insensitive) |
| FR-7 | Show low-stock alert (quantity below configurable threshold) |

---

## Non-functional requirements

- Pure OOP — interfaces for repository and service where substitution matters
- No static mutable global state except `main` entry
- Input validation at service layer
- Meaningful error messages (product not found, duplicate id, invalid quantity)
- Package structure ready for JUnit tests in phase 5

---

## Design tasks

### Task 1 — Class diagram (text or ASCII)

Identify classes, interfaces, and relationships:

- Domain entity: `Product`
- `ProductRepository` interface + `InMemoryProductRepository`
- `InventoryService` (business rules)
- `InventoryCli` or `CommandRouter` (user interaction)
- `Main` bootstrap

Show associations and dependency direction (CLI → Service → Repository → Domain).

### Task 2 — Product model

Define fields, constructors, getters, and validation rules:

- `id`: non-blank, unique (enforced by repository)
- `name`: non-blank
- `quantity`: non-negative int
- `unitPrice`: non-negative; use `BigDecimal` or `double` — **justify your choice**

### Task 3 — CLI command map

Design the menu and command syntax:

```
1. add
2. list
3. find <id>
4. update <id>
5. delete <id>
6. search <name>
7. low-stock [threshold]
8. exit
```

Document expected prompts and error flows for each command.

### Task 4 — Exception strategy

List custom exceptions (e.g. `ProductNotFoundException`, `DuplicateProductException`, `ValidationException`) and which layer throws them.

### Task 5 — Test plan sketch

List 8–10 unit tests you will write in phase 5 (names only, Given-When-Then one-liner each).

---

## Deliverables (this phase)

- [ ] ASCII or described class diagram
- [ ] Field list and validation table for `Product`
- [ ] CLI command map with error flows
- [ ] Exception hierarchy sketch
- [ ] Test plan list

---

## Out of scope (later phases)

- File persistence
- Multi-user / concurrency
- Web or GUI
