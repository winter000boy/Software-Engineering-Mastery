# #004 — Solution: Build — Service & CLI

**Date:** 2026-08-18  
**Problem:** [004-cli-inventory-build-cli-service.md](../problems/004-cli-inventory-build-cli-service.md)

---

## InventoryService

```java
package com.inventory.service;

import com.inventory.domain.Product;
import com.inventory.exception.ProductNotFoundException;
import com.inventory.exception.ValidationException;
import com.inventory.repository.ProductRepository;
import java.math.BigDecimal;
import java.util.Comparator;
import java.util.List;

public class InventoryService {
    private final ProductRepository productRepository;

    public InventoryService(ProductRepository productRepository) {
        if (productRepository == null) {
            throw new IllegalArgumentException("ProductRepository required");
        }
        this.productRepository = productRepository;
    }

    public Product addProduct(String id, String name, int quantity, BigDecimal unitPrice) {
        Product product = new Product(id, name, quantity, unitPrice);
        return productRepository.save(product);
    }

    public List<Product> listProducts() {
        return productRepository.findAll().stream()
                .sorted(Comparator.comparing(Product::getId))
                .toList();
    }

    public Product getProduct(String id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
    }

    public Product updateProduct(String id, int newQuantity, BigDecimal newPrice) {
        Product existing = getProduct(id);
        existing.setQuantity(newQuantity);
        existing.setUnitPrice(newPrice);
        return productRepository.update(existing);
    }

    public void removeProduct(String id) {
        productRepository.deleteById(id);
    }

    public List<Product> searchByName(String fragment) {
        if (fragment == null || fragment.isBlank()) {
            throw new ValidationException("Search text cannot be blank");
        }
        return productRepository.searchByName(fragment);
    }

    public List<Product> lowStockReport(int threshold) {
        if (threshold < 0) {
            throw new ValidationException("Threshold cannot be negative");
        }
        return productRepository.findLowStock(threshold);
    }
}
```

**Sort policy:** by `id` ascending — documented and tested in phase 5.

---

## InventoryCli

```java
package com.inventory.cli;

import com.inventory.domain.Product;
import com.inventory.exception.DuplicateProductException;
import com.inventory.exception.ProductNotFoundException;
import com.inventory.exception.ValidationException;
import com.inventory.service.InventoryService;
import java.math.BigDecimal;
import java.util.List;
import java.util.Scanner;

public class InventoryCli {
    private static final int DEFAULT_LOW_STOCK_THRESHOLD = 5;

    private final InventoryService inventoryService;
    private final Scanner scanner;

    public InventoryCli(InventoryService inventoryService, Scanner scanner) {
        this.inventoryService = inventoryService;
        this.scanner = scanner;
    }

    public void run() {
        boolean running = true;
        while (running) {
            printMenu();
            String choice = scanner.nextLine().trim();
            try {
                running = handleChoice(choice);
            } catch (ValidationException | ProductNotFoundException | DuplicateProductException ex) {
                System.out.println("Error: " + ex.getMessage());
            } catch (NumberFormatException ex) {
                System.out.println("Error: invalid number format");
            }
        }
        System.out.println("Goodbye.");
    }

    private boolean handleChoice(String choice) {
        return switch (choice) {
            case "1" -> { handleAdd(); yield true; }
            case "2" -> { handleList(); yield true; }
            case "3" -> { handleFind(); yield true; }
            case "4" -> { handleUpdate(); yield true; }
            case "5" -> { handleDelete(); yield true; }
            case "6" -> { handleSearch(); yield true; }
            case "7" -> { handleLowStock(); yield true; }
            case "8" -> false;
            default -> {
                System.out.println("Invalid choice. Enter 1-8.");
                yield true;
            }
        };
    }

    private void printMenu() {
        System.out.println();
        System.out.println("=== Inventory CLI ===");
        System.out.println("1. Add product");
        System.out.println("2. List all");
        System.out.println("3. Find by id");
        System.out.println("4. Update product");
        System.out.println("5. Delete product");
        System.out.println("6. Search by name");
        System.out.println("7. Low stock report");
        System.out.println("8. Exit");
        System.out.print("Choice: ");
    }

    private void handleAdd() {
        System.out.print("Id: ");
        String id = scanner.nextLine();
        System.out.print("Name: ");
        String name = scanner.nextLine();
        System.out.print("Quantity: ");
        int qty = Integer.parseInt(scanner.nextLine().trim());
        System.out.print("Unit price: ");
        BigDecimal price = new BigDecimal(scanner.nextLine().trim());
        Product saved = inventoryService.addProduct(id, name, qty, price);
        System.out.println("Added: " + saved);
    }

    private void handleList() {
        List<Product> products = inventoryService.listProducts();
        if (products.isEmpty()) {
            System.out.println("No products.");
            return;
        }
        printTable(products);
    }

    private void handleFind() {
        System.out.print("Id: ");
        Product product = inventoryService.getProduct(scanner.nextLine().trim());
        System.out.println(product);
    }

    private void handleUpdate() {
        System.out.print("Id: ");
        String id = scanner.nextLine().trim();
        System.out.print("New quantity: ");
        int qty = Integer.parseInt(scanner.nextLine().trim());
        System.out.print("New unit price: ");
        BigDecimal price = new BigDecimal(scanner.nextLine().trim());
        Product updated = inventoryService.updateProduct(id, qty, price);
        System.out.println("Updated: " + updated);
    }

    private void handleDelete() {
        System.out.print("Id: ");
        String id = scanner.nextLine().trim();
        inventoryService.removeProduct(id);
        System.out.println("Deleted: " + id);
    }

    private void handleSearch() {
        System.out.print("Name fragment: ");
        List<Product> results = inventoryService.searchByName(scanner.nextLine());
        if (results.isEmpty()) {
            System.out.println("No matches.");
        } else {
            printTable(results);
        }
    }

    private void handleLowStock() {
        System.out.print("Threshold (default " + DEFAULT_LOW_STOCK_THRESHOLD + "): ");
        String line = scanner.nextLine().trim();
        int threshold = line.isEmpty() ? DEFAULT_LOW_STOCK_THRESHOLD : Integer.parseInt(line);
        List<Product> low = inventoryService.lowStockReport(threshold);
        if (low.isEmpty()) {
            System.out.println("No low-stock items.");
        } else {
            printTable(low);
        }
    }

    private void printTable(List<Product> products) {
        System.out.printf("%-8s %-20s %6s %10s%n", "ID", "Name", "Qty", "Price");
        System.out.println("-".repeat(48));
        for (Product product : products) {
            System.out.printf("%-8s %-20s %6d %10s%n",
                    product.getId(),
                    truncate(product.getName(), 20),
                    product.getQuantity(),
                    product.getUnitPrice());
        }
    }

    private String truncate(String value, int maxLength) {
        if (value.length() <= maxLength) {
            return value;
        }
        return value.substring(0, maxLength - 3) + "...";
    }
}
```

---

## Main

```java
package com.inventory;

import com.inventory.cli.InventoryCli;
import com.inventory.repository.InMemoryProductRepository;
import com.inventory.repository.ProductRepository;
import com.inventory.service.InventoryService;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        ProductRepository repository = new InMemoryProductRepository();
        InventoryService service = new InventoryService(repository);
        Scanner scanner = new Scanner(System.in);
        InventoryCli cli = new InventoryCli(service, scanner);
        cli.run();
        scanner.close();
    }
}
```

---

## Manual E2E checklist

- [x] Add P1, P2, P3
- [x] List shows sorted table
- [x] Update P2 quantity
- [x] Search partial name
- [x] Delete P1
- [x] Exit cleanly
