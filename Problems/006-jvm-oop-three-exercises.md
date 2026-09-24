# #001 — Solve 3 Exercises on JVM & OOP

**Date:** 2026-08-18  
**Type:** Exercise  
**Topics:** JVM, bytecode, memory model, encapsulation, inheritance, polymorphism

---

## Overview

Complete all three exercises in order. Write answers in your own words first, then compare with the solution file.

---

## Exercise 1 — JVM Memory Trace

Trace where each value lives (stack, heap, or static/metaspace storage).

```java
public class Warehouse {
    static int totalItems = 0;
    String name;
    int quantity;

    Warehouse(String name, int quantity) {
        this.name = name;
        this.quantity = quantity;
        totalItems += quantity;
    }

    public static void main(String[] args) {
        int orderId = 1001;
        Warehouse item = new Warehouse("Widget", 5);
        adjust(item, orderId);
    }

    static void adjust(Warehouse w, int delta) {
        int newQty = w.quantity + delta;
        w.quantity = newQty;
    }
}
```

**Tasks:**

1. Fill in stack vs heap (or static) for: `totalItems`, `name`, `quantity` inside `item`, `orderId`, `item` reference, the `Warehouse` object, `w`, `delta`, `newQty`.
2. What happens to `totalItems` when `adjust` runs — does it change? Why or why not?
3. If `adjust` were an instance method instead of `static`, what would change in memory layout at call time?

---

## Exercise 2 — OOP Encapsulation & Validation

Design a `BankAccount` class in pure OOP (no frameworks) with:

| Requirement | Detail |
|-------------|--------|
| Fields | `accountNumber` (String), `balance` (double) — both private |
| Constructor | Takes account number and initial balance; reject negative initial balance |
| Methods | `deposit(double)`, `withdraw(double)`, `getBalance()` |
| Rules | Withdraw cannot exceed balance; deposit/withdraw amounts must be > 0 |
| Immutability | Account number never changes after construction |

**Tasks:**

1. Write the class with proper encapsulation.
2. Explain why fields are private and what breaks if they were public.
3. Add a `toString()` that does **not** expose raw balance formatting issues (use 2 decimal places conceptually).

---

## Exercise 3 — Polymorphism & Interface Segregation

A shipping system needs to calculate cost differently per carrier:

- **StandardShipping** — flat rate 5.00
- **ExpressShipping** — base 10.00 + 2.00 per kg
- **FreeShipping** — always 0.00

**Tasks:**

1. Define a `ShippingStrategy` interface with `double calculateCost(double weightKg)`.
2. Implement all three concrete strategies.
3. Write a `Shipment` class that holds a `ShippingStrategy` and delegates cost calculation (composition, not inheritance for behavior).
4. In `main`, create three `Shipment` objects with different strategies and print costs for 3.5 kg.
5. Explain how polymorphism lets you add a fourth strategy without changing `Shipment`.

---

## Constraints

- Pure Java — no Spring, no Lombok, no external libraries.
- Each exercise should compile and run with `javac` + `java`.
- Document assumptions in comments where needed.

---

## Self-check

- [ ] Exercise 1: every variable has a justified stack/heap/static answer
- [ ] Exercise 2: invalid operations throw or reject cleanly (your choice — document it)
- [ ] Exercise 3: `Shipment` has zero `if/else` on carrier type for pricing

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
