# #001 — Solution: 3 Exercises on JVM & OOP

**Date:** 2026-08-18  
**Problem:** [001-jvm-oop-three-exercises.md](../problems/001-jvm-oop-three-exercises.md)

---

## Exercise 1 — JVM Memory Trace

### Answers

| Variable / object | Location | Notes |
|-------------------|----------|-------|
| `totalItems` (value) | Static storage (Metaspace area for class data) | Belongs to `Warehouse` class, not any instance |
| `name` inside `item` | **Heap** | Instance field on the object |
| `quantity` inside `item` | **Heap** | Instance field on the object |
| `orderId` | **Stack** | Local in `main`'s frame |
| `item` reference | **Stack** | Reference variable in `main`; points to heap object |
| `Warehouse` object | **Heap** | Allocated by `new Warehouse(...)` |
| `w` | **Stack** | Parameter reference in `adjust`'s frame |
| `delta` | **Stack** | Primitive parameter in `adjust`'s frame |
| `newQty` | **Stack** | Local primitive in `adjust`'s frame |

### Q2: Does `totalItems` change when `adjust` runs?

**No.** `adjust` only reads/writes `w.quantity`. `totalItems` was updated once in the constructor (`totalItems += quantity`). Unless `adjust` modifies `totalItems`, it stays at `5`.

### Q3: If `adjust` were an instance method?

- `main` would call `item.adjust(orderId)` instead of `Warehouse.adjust(item, orderId)`.
- Memory layout is similar: `orderId` still on stack; `item` reference on stack; object on heap.
- **`this`** would implicitly refer to the heap object — no extra heap allocation; `this` is a stack reference like `w`.
- Call site is more idiomatic OOP; JVM still uses stack frames and heap objects the same way.

---

## Exercise 2 — `BankAccount`

```java
public class BankAccount {
    private final String accountNumber;
    private double balance;

    public BankAccount(String accountNumber, double initialBalance) {
        if (accountNumber == null || accountNumber.isBlank()) {
            throw new IllegalArgumentException("Account number required");
        }
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Withdraw amount must be positive");
        }
        if (amount > balance) {
            throw new IllegalStateException("Insufficient funds");
        }
        balance -= amount;
    }

    public double getBalance() {
        return balance;
    }

    public String getAccountNumber() {
        return accountNumber;
    }

    @Override
    public String toString() {
        return "BankAccount{accountNumber='" + accountNumber + "', balance="
                + String.format("%.2f", balance) + "}";
    }
}
```

**Why private fields?** Encapsulation ensures invariants (non-negative balance, valid amounts) are enforced only through methods. Public fields would let callers set `balance = -1000` or skip audit logic.

---

## Exercise 3 — Shipping Strategies

```java
// ShippingStrategy.java
public interface ShippingStrategy {
    double calculateCost(double weightKg);
}

// StandardShipping.java
public class StandardShipping implements ShippingStrategy {
    private static final double FLAT_RATE = 5.00;

    @Override
    public double calculateCost(double weightKg) {
        return FLAT_RATE;
    }
}

// ExpressShipping.java
public class ExpressShipping implements ShippingStrategy {
    private static final double BASE = 10.00;
    private static final double PER_KG = 2.00;

    @Override
    public double calculateCost(double weightKg) {
        return BASE + PER_KG * weightKg;
    }
}

// FreeShipping.java
public class FreeShipping implements ShippingStrategy {
    @Override
    public double calculateCost(double weightKg) {
        return 0.00;
    }
}

// Shipment.java
public class Shipment {
    private final ShippingStrategy strategy;
    private final double weightKg;

    public Shipment(ShippingStrategy strategy, double weightKg) {
        if (strategy == null) {
            throw new IllegalArgumentException("Strategy required");
        }
        if (weightKg < 0) {
            throw new IllegalArgumentException("Weight cannot be negative");
        }
        this.strategy = strategy;
        this.weightKg = weightKg;
    }

    public double cost() {
        return strategy.calculateCost(weightKg);
    }
}

// Demo
public class ShippingDemo {
    public static void main(String[] args) {
        double weight = 3.5;
        System.out.println(new Shipment(new StandardShipping(), weight).cost()); // 5.00
        System.out.println(new Shipment(new ExpressShipping(), weight).cost());  // 17.00
        System.out.println(new Shipment(new FreeShipping(), weight).cost());     // 0.00
    }
}
```

**Polymorphism benefit:** Add `OvernightShipping` implementing `ShippingStrategy` — pass it to `Shipment` with zero changes to `Shipment`. Open/Closed Principle in action.

---

## Key takeaways

1. **Stack** holds frames, local primitives, and references; **heap** holds objects; **static** class data lives with the class metadata.
2. **Encapsulation** protects invariants — always validate in constructors and mutators.
3. **Strategy + composition** beats giant `if/else` on type; new behavior = new class, not modified client code.
