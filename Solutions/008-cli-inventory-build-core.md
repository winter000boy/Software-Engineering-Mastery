# #008 — Solution: Build Core — Domain & Repository

**Date:** 2026-08-18  
**Problem:** [008-cli-inventory-build-core.md](../problems/008-cli-inventory-build-core.md)

---

## Exceptions

```java
// com.inventory.exception.ValidationException
public class ValidationException extends RuntimeException {
    public ValidationException(String message) { super(message); }
}

// com.inventory.exception.ProductNotFoundException
public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String id) {
        super("Product not found: " + id);
    }
}

// com.inventory.exception.DuplicateProductException
public class DuplicateProductException extends RuntimeException {
    public DuplicateProductException(String id) {
        super("Product already exists: " + id);
    }
}
```

---

## Product

```java
package com.inventory.domain;

import com.inventory.exception.ValidationException;
import java.math.BigDecimal;
import java.util.Objects;

public class Product {
    private final String id;
    private String name;
    private int quantity;
    private BigDecimal unitPrice;

    public Product(String id, String name, int quantity, BigDecimal unitPrice) {
        validateId(id);
        validateName(name);
        validateQuantity(quantity);
        validatePrice(unitPrice);
        this.id = id.trim();
        this.name = name.trim();
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public int getQuantity() { return quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }

    public void setName(String name) {
        validateName(name);
        this.name = name.trim();
    }

    public void setQuantity(int quantity) {
        validateQuantity(quantity);
        this.quantity = quantity;
    }

    public void setUnitPrice(BigDecimal unitPrice) {
        validatePrice(unitPrice);
        this.unitPrice = unitPrice;
    }

    private static void validateId(String id) {
        if (id == null || id.isBlank()) {
            throw new ValidationException("Product id is required");
        }
    }

    private static void validateName(String name) {
        if (name == null || name.isBlank()) {
            throw new ValidationException("Product name is required");
        }
    }

    private static void validateQuantity(int quantity) {
        if (quantity < 0) {
            throw new ValidationException("Quantity cannot be negative");
        }
    }

    private static void validatePrice(BigDecimal unitPrice) {
        if (unitPrice == null || unitPrice.compareTo(BigDecimal.ZERO) < 0) {
            throw new ValidationException("Unit price cannot be negative");
        }
    }

    @Override
    public boolean equals(Object other) {
        if (this == other) return true;
        if (!(other instanceof Product product)) return false;
        return id.equals(product.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }

    @Override
    public String toString() {
        return String.format("Product{id='%s', name='%s', qty=%d, price=%s}",
                id, name, quantity, unitPrice);
    }
}
```

---

## ProductRepository interface

```java
package com.inventory.repository;

import com.inventory.domain.Product;
import java.util.List;
import java.util.Optional;

public interface ProductRepository {
    Product save(Product product);
    Optional<Product> findById(String id);
    List<Product> findAll();
    Product update(Product product);
    void deleteById(String id);
    List<Product> searchByName(String nameFragment);
    List<Product> findLowStock(int threshold);
    boolean existsById(String id);
}
```

---

## InMemoryProductRepository

```java
package com.inventory.repository;

import com.inventory.domain.Product;
import com.inventory.exception.DuplicateProductException;
import com.inventory.exception.ProductNotFoundException;
import java.util.ArrayList;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Optional;

public class InMemoryProductRepository implements ProductRepository {
    private final Map<String, Product> store = new LinkedHashMap<>();

    @Override
    public Product save(Product product) {
        if (existsById(product.getId())) {
            throw new DuplicateProductException(product.getId());
        }
        store.put(product.getId(), product);
        return product;
    }

    @Override
    public Optional<Product> findById(String id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Product> findAll() {
        return new ArrayList<>(store.values());
    }

    @Override
    public Product update(Product product) {
        if (!existsById(product.getId())) {
            throw new ProductNotFoundException(product.getId());
        }
        store.put(product.getId(), product);
        return product;
    }

    @Override
    public void deleteById(String id) {
        if (store.remove(id) == null) {
            throw new ProductNotFoundException(id);
        }
    }

    @Override
    public List<Product> searchByName(String nameFragment) {
        String needle = nameFragment.toLowerCase(Locale.ROOT);
        return store.values().stream()
                .filter(product -> product.getName().toLowerCase(Locale.ROOT).contains(needle))
                .sorted(Comparator.comparing(Product::getId))
                .toList();
    }

    @Override
    public List<Product> findLowStock(int threshold) {
        return store.values().stream()
                .filter(product -> product.getQuantity() < threshold)
                .sorted(Comparator.comparing(Product::getId))
                .toList();
    }

    @Override
    public boolean existsById(String id) {
        return store.containsKey(id);
    }
}
```

---

## RepositorySmokeTest

```java
package com.inventory;

import com.inventory.domain.Product;
import com.inventory.repository.InMemoryProductRepository;
import com.inventory.repository.ProductRepository;
import java.math.BigDecimal;

public class RepositorySmokeTest {
    public static void main(String[] args) {
        ProductRepository repo = new InMemoryProductRepository();

        repo.save(new Product("P1", "Widget", 10, new BigDecimal("9.99")));
        repo.save(new Product("P2", "Gadget", 3, new BigDecimal("19.50")));

        System.out.println("All: " + repo.findAll());

        Product p1 = repo.findById("P1").orElseThrow();
        p1.setQuantity(25);
        repo.update(p1);

        System.out.println("Search widget: " + repo.searchByName("widget"));
        repo.deleteById("P2");
        System.out.println("After delete: " + repo.findAll());
        System.out.println("Low stock (<5): " + repo.findLowStock(5));
    }
}
```

**Map choice:** `LinkedHashMap` preserves insertion order for predictable listing before explicit sort in service layer.
