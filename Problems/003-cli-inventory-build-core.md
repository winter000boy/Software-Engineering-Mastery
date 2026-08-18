# #003 — Build: CLI Inventory System — Core Domain & Repository

**Date:** 2026-08-18  
**Type:** Project — Phase 2 of 5  
**Topics:** domain modeling, repository pattern, in-memory CRUD (partial)

---

## Prerequisites

Complete **#002 Plan & Design** (or use the provided design in the solution file).

---

## Build tasks

Implement the **core layers** only — no CLI yet.

### Package structure

```
com.inventory
├── domain/
│   └── Product.java
├── repository/
│   ├── ProductRepository.java
│   └── InMemoryProductRepository.java
└── exception/
    ├── ProductNotFoundException.java
    ├── DuplicateProductException.java
    └── ValidationException.java
```

### Task 1 — `Product`

- Immutable `id` and validated fields per design
- Factory or constructor validation (reject null/blank id, negative quantity/price)
- `equals` / `hashCode` based on `id`
- `toString()` suitable for later table display

### Task 2 — `ProductRepository` interface

Methods (minimum):

```java
Product save(Product product);
Optional<Product> findById(String id);
List<Product> findAll();
Product update(Product product);
void deleteById(String id);
List<Product> searchByName(String nameFragment);
List<Product> findLowStock(int threshold);
boolean existsById(String id);
```

### Task 3 — `InMemoryProductRepository`

- Back with `Map<String, Product>` (or concurrent map if you prefer — document choice)
- `save`: throw `DuplicateProductException` if id exists
- `update`: throw `ProductNotFoundException` if missing
- `deleteById`: throw `ProductNotFoundException` if missing
- `searchByName`: case-insensitive contains
- `findLowStock`: quantity `< threshold`

### Task 4 — Smoke test in `main` (temporary)

Write a small `RepositorySmokeTest` class (not JUnit yet) that:

1. Saves two products
2. Lists all
3. Updates one
4. Searches by name
5. Deletes one
6. Prints results to stdout

Remove or relocate before phase 5 formal tests.

---

## Constraints

- No CLI menu in this phase
- No service layer yet (phase 4)
- Constructor injection ready — repository accepts no external deps for now

---

## Acceptance criteria

- [ ] All repository methods implemented
- [ ] Duplicate id on save fails fast
- [ ] Update/delete on missing id throws domain exception
- [ ] Code compiles with `javac`
- [ ] Smoke test runs without errors
