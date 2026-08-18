# #005 — Solution: Test & Refactor

**Date:** 2026-08-18  
**Problem:** [005-cli-inventory-test-refactor.md](../problems/005-cli-inventory-test-refactor.md)

---

## Sample repository tests (JUnit 5)

```java
package com.inventory.repository;

import com.inventory.domain.Product;
import com.inventory.exception.DuplicateProductException;
import com.inventory.exception.ProductNotFoundException;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryProductRepositoryTest {
    private InMemoryProductRepository repository;

    @BeforeEach
    void setUp() {
        repository = new InMemoryProductRepository();
    }

    @Test
    void save_newProduct_persistsAndReturns() {
        Product product = new Product("A1", "Hammer", 5, new BigDecimal("12.00"));
        repository.save(product);
        assertTrue(repository.findById("A1").isPresent());
    }

    @Test
    void save_duplicateId_throwsDuplicateProductException() {
        repository.save(new Product("A1", "Hammer", 5, new BigDecimal("12.00")));
        assertThrows(DuplicateProductException.class,
                () -> repository.save(new Product("A1", "Other", 1, new BigDecimal("1.00"))));
    }

    @Test
    void findById_missing_returnsEmpty() {
        assertTrue(repository.findById("missing").isEmpty());
    }

    @Test
    void update_existing_changesFields() {
        repository.save(new Product("A1", "Hammer", 5, new BigDecimal("12.00")));
        Product loaded = repository.findById("A1").orElseThrow();
        loaded.setQuantity(10);
        repository.update(loaded);
        assertEquals(10, repository.findById("A1").orElseThrow().getQuantity());
    }

    @Test
    void update_missing_throwsProductNotFoundException() {
        Product ghost = new Product("X", "X", 1, BigDecimal.ONE);
        assertThrows(ProductNotFoundException.class, () -> repository.update(ghost));
    }

    @Test
    void deleteById_existing_removesProduct() {
        repository.save(new Product("A1", "Hammer", 5, new BigDecimal("12.00")));
        repository.deleteById("A1");
        assertTrue(repository.findById("A1").isEmpty());
    }

    @Test
    void searchByName_caseInsensitive_partialMatch() {
        repository.save(new Product("A1", "Super Widget", 1, BigDecimal.ONE));
        List<Product> results = repository.searchByName("widget");
        assertEquals(1, results.size());
    }

    @Test
    void findLowStock_returnsOnlyBelowThreshold() {
        repository.save(new Product("A1", "Low", 2, BigDecimal.ONE));
        repository.save(new Product("A2", "Ok", 10, BigDecimal.ONE));
        assertEquals(1, repository.findLowStock(5).size());
    }
}
```

---

## Sample service tests

```java
package com.inventory.service;

import com.inventory.domain.Product;
import com.inventory.exception.ProductNotFoundException;
import com.inventory.exception.ValidationException;
import com.inventory.repository.InMemoryProductRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class InventoryServiceTest {
    private InventoryService service;

    @BeforeEach
    void setUp() {
        service = new InventoryService(new InMemoryProductRepository());
    }

    @Test
    void addProduct_invalidQuantity_throwsValidationException() {
        assertThrows(ValidationException.class,
                () -> service.addProduct("A1", "Hammer", -1, BigDecimal.TEN));
    }

    @Test
    void addProduct_blankName_throwsValidationException() {
        assertThrows(ValidationException.class,
                () -> service.addProduct("A1", "  ", 1, BigDecimal.TEN));
    }

    @Test
    void getProduct_unknownId_throwsProductNotFoundException() {
        assertThrows(ProductNotFoundException.class, () -> service.getProduct("nope"));
    }

    @Test
    void listProducts_returnsSortedById() {
        service.addProduct("B2", "B", 1, BigDecimal.ONE);
        service.addProduct("A1", "A", 1, BigDecimal.ONE);
        List<Product> list = service.listProducts();
        assertEquals("A1", list.get(0).getId());
        assertEquals("B2", list.get(1).getId());
    }
}
```

**Test double choice:** use real `InMemoryProductRepository` in service tests — fast, no mock framework needed for this scope.

---

## REFACTOR_LOG

| # | Before | After | Why |
|---|--------|-------|-----|
| 1 | Table printing inline in each handler | Extracted `printTable()` + `truncate()` | DRY; easier to test formatting later |
| 2 | Magic number `5` for low stock in CLI | `DEFAULT_LOW_STOCK_THRESHOLD` constant | Single place to change default |
| 3 | Repeated validation strings in Product | Private static validate methods | Centralized rules; clearer errors |

---

## SOLID self-review

1. **`ProductRepository` demonstrates DIP** — high-level `InventoryService` depends on abstraction, not `InMemoryProductRepository`.
2. **`InventoryCli` slightly violates SRP** — menu + parsing + formatting. Could split `CommandParser` and `ProductTableFormatter` if CLI grows.
3. **`InventoryService` is mostly OCP** — new report types could add methods; for many reports, extract a `ReportService` instead of bloating inventory service.

---

## Run tests

```bash
# Maven
mvn test

# Gradle
./gradlew test
```

All 12 tests above should pass before marking phase complete.
