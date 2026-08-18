# #006 — Solution: Document & Review

**Date:** 2026-08-18  
**Problem:** [006-cli-inventory-document-review.md](../problems/006-cli-inventory-document-review.md)

---

## README (project artifact)

```markdown
# CLI Inventory System

Command-line inventory manager built in pure Java OOP with in-memory storage.

## Requirements

- JDK 17+
- Maven 3.9+ (or Gradle 8+)

## Build & run

mvn compile exec:java -Dexec.mainClass="com.inventory.Main"

## CLI menu

1. Add product
2. List all
3. Find by id
4. Update product
5. Delete product
6. Search by name
7. Low stock report
8. Exit

## Sample session

=== Inventory CLI ===
Choice: 1
Id: P1
Name: Widget
Quantity: 10
Unit price: 9.99
Added: Product{id='P1', name='Widget', qty=10, price=9.99}

Choice: 2
ID       Name                    Qty      Price
------------------------------------------------
P1       Widget                   10       9.99

Choice: 8
Goodbye.

## Architecture

CLI → InventoryService → ProductRepository → Product

## Design decisions

- BigDecimal for money (no double rounding bugs)
- In-memory Map for simplicity; interface allows file/DB swap
- Custom runtime exceptions with clear messages

## Tests

mvn test

## Future improvements

- JSON/file persistence
- Export CSV
- Thread-safe repository for concurrent access
```

---

## JavaDoc samples

```java
/**
 * Application service for inventory operations.
 * Validates business rules and delegates persistence to {@link ProductRepository}.
 */
public class InventoryService { ... }

/**
 * Persistence port for {@link Product} entities.
 */
public interface ProductRepository {
    /** Saves a new product; id must not already exist. */
    Product save(Product product);
    // ...
}

/**
 * Thrown when a product id is not found in the repository.
 */
public class ProductNotFoundException extends RuntimeException { ... }
```

---

## Code review checklist (completed)

| Area | Pass? | Notes |
|------|-------|-------|
| No business logic in CLI | ✅ | Only I/O and formatting |
| No persistence logic in service | ✅ | Delegates to repository |
| Exceptions used vs raw strings | ✅ | Typed exceptions |
| Input validation centralized | ✅ | Product + service |
| equals/hashCode on domain entity | ✅ | By id |
| Tests cover happy + sad paths | ✅ | 12 tests |
| No dead code | ✅ | Smoke test optional/remove |
| Consistent formatting | ✅ | Spotless or IDE default |

---

## Retrospective

1. **Hardest part:** keeping CLI thin while handling all parse errors gracefully.
2. **OOP concept reinforced:** **Dependency Inversion** — service tests swap repository without changing service code.
3. **Redesign if starting over:** introduce `ProductId` value object earlier to avoid string id duplication in signatures.

---

## ARCHITECTURE.md (one-page summary)

```
Layers
------
InventoryCli        — user input/output, menu loop
InventoryService    — validation orchestration, sort policy
ProductRepository   — CRUD + query port (interface)
InMemoryProductRepository — HashMap-backed adapter
Product             — domain entity with invariants

Dependency graph
----------------
Main → Cli → Service → Repository(interface) ← InMemoryImpl
                              ↓
                           Product

Extension points
----------------
- New repository: FileProductRepository implements ProductRepository
- New UI: InventoryGui implements same service calls as CLI
- New reports: InventoryReportService injected alongside service
```

---

## Project status

Phases #002–#006 complete. Linked from [PROBLEMS-INDEX.md](../PROBLEMS-INDEX.md).
