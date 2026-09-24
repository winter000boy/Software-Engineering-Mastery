# #005 — Solution: Model a Library Domain with Classes & Interfaces

**Date:** 2026-09-24  
**Problem:** [005-library-domain-classes-interfaces.md](../problems/005-library-domain-classes-interfaces.md)

> Verified: compiled on Java 21 with Maven; **11/11 JUnit tests pass** and the demo runs. All code below is the verified code.

---

## 1. Class Diagram

```
                         «abstract class»
                           LibraryItem
                    ─────────────────────────
                    - id: ItemId  - title
                    + kind(): String  {abstract}
                    + equals/hashCode by id  {final}
                     ▲          ▲           ▲
          extends    │          │           │   extends
     ┌───────────────┘          │           └──────────────────┐
     │                          │                              │
«final» Book               «final» Dvd                  «final» ReferenceBook
 - isbn: Isbn               - runtimeMinutes              - isbn: Isbn
 - authors: List<Author>        │                          (NOT Borrowable)
 - publicationYear              │
     │ implements               │ implements
     └───────────┐   ┌──────────┘
                 ▼   ▼
             «interface» Borrowable
             + loanPeriodDays()
             + dailyLateFeeCents()

«final» Member ──has-a──▶ «interface» MembershipPolicy ◀── implements ── «enum» StandardMembership
 - id: MemberId                + tierName()                               STUDENT, ADULT, PREMIUM
 - name                        + maxActiveLoans()
 - policy (mutable)            + loanPeriodMultiplier()
                               + waivesLateFees() {default}

«final» Loan (immutable) ──▶ LibraryItem, Member, borrowedOn, dueOn, returnedOn?

«final» Library (service) ──has-a──▶ Clock (injected)
   catalogue: Map<ItemId, LibraryItem>   members: Map<MemberId, Member>
   activeLoans: Map<ItemId, Loan>        history: List<Loan>

Value objects: «final» Isbn, «record» Author, «record» ItemId, «record» MemberId
Exceptions:    LibraryException ◀── ItemNotFound / ItemNotBorrowable / ItemUnavailable / LoanLimitExceeded
```

---

## 2. Design Decisions (D1–D9)

| # | Choice | Why |
|---|--------|-----|
| **D1** | `LibraryItem` = **abstract class**; `Borrowable` and `MembershipPolicy` = **interfaces** | `LibraryItem` shares **state and behaviour** (id, title, equality): a true "is-a" skeleton, so it's an abstract class. `Borrowable` is a **capability/mixin** that unrelated types could have (it could even be added to a laptop-lending class that isn't a `LibraryItem`), so it's an interface (EJ Item 20). `MembershipPolicy` is a **strategy contract** with no shared state. |
| **D2** | `Book`, `Dvd` implement `Borrowable`; `ReferenceBook` doesn't | **Interface Segregation Principle**: a reference book would otherwise need a `loanPeriodDays()` that throws `UnsupportedOperationException`, a Liskov violation waiting to happen. The type system now encodes "can be borrowed". |
| **D3** | `Member` **has-a** `MembershipPolicy`; tiers are an **enum implementing the interface** | Composition (EJ Item 18): changing tier is one field assignment, not changing the object's class, which inheritance (`StudentMember extends Member`) can't do at run time. `Library` only calls `policy().maxActiveLoans()`, so no tier `if`/`switch` exists. A new tier is a new enum constant **or** any new class implementing `MembershipPolicy` (e.g. a `CorporateMembership` loaded from config) with zero changes to `Library`/`Member`. |
| **D4** | Value objects: `Isbn` (final class), `Author`/`ItemId`/`MemberId` (**records**). Entities: `LibraryItem`, `Member` compare **by id** | Two `Author("Joshua Bloch")` objects are the same author, so value equality. Two copies of *Effective Java* share an ISBN but are **different shelf items**, so identity equality by `ItemId`. That's why the item id is **not** the ISBN. |
| **D5** | `Isbn.of(raw)` static factory, validating before construction; private constructor | Invalid ISBNs can never exist, so all later code can trust them (EJ Item 1: the factory can normalise the input, strip hyphens, and reject it before any object is created). |
| **D6** | `List.copyOf(authors)` in the `Book` constructor | Defensive + unmodifiable copy (EJ Item 17, rule 5). Mutating the caller's original list doesn't affect the book, and `authors().add(...)` throws. No need to copy again in the accessor. |
| **D7** | `Library(Clock clock)` + `LocalDate.now(clock)`; a `MutableClock` test double | Dependency injection of **time** (EJ Item 5). Tests can say "25 days later" deterministically. `Clock.systemDefaultZone()` in production. |
| **D8** | `LibraryException extends RuntimeException` + 4 specific subclasses | Callers (a CLI, later a REST controller) can catch the base type once and map it to a message/HTTP 409, or catch a specific one. Unchecked, because these are **business-rule violations**, not recoverable I/O conditions the compiler should force every caller to handle. |
| **D9** | Concrete classes `final`; `LibraryItem.equals/hashCode/id/title` `final`; all fields `private` | Nothing is designed for extension except `LibraryItem`, and even there the **equality contract is locked** so no subclass can break symmetry by adding fields to `equals` (EJ Items 10, 19). |

### Why `instanceof Borrowable` is fine but `instanceof StudentTier` isn't

`item instanceof Borrowable` asks about a **capability defined by the type system**. The alternative, a boolean `isBorrowable()` field, could disagree with whether the methods exist. Pattern matching (`instanceof Borrowable b`) then gives a typed variable, so there's no cast.

Checking **which tier** a member has would put tier-specific *rules* in the service, and every new tier would mean editing `Library`. That breaks **Open/Closed**. The rule belongs **in** the policy object, so `Library` asks the policy (polymorphism) instead of asking about it.

---

## 3. Code — Value Objects

### `domain/Isbn.java`

```java
package com.library.domain;

import java.util.Objects;

/** Value object: a validated ISBN-13. Immutable; equality by value. */
public final class Isbn {
    private final String value;

    private Isbn(String value) {
        this.value = value;
    }

    public static Isbn of(String raw) {
        Objects.requireNonNull(raw, "isbn");
        String digits = raw.replace("-", "").replace(" ", "");
        if (!digits.matches("\\d{13}")) {
            throw new IllegalArgumentException("ISBN-13 must have 13 digits: " + raw);
        }
        if (!hasValidChecksum(digits)) {
            throw new IllegalArgumentException("Invalid ISBN-13 checksum: " + raw);
        }
        return new Isbn(digits);
    }

    // ISBN-13: weights alternate 1,3,1,3...; the weighted sum of all 13 digits must be divisible by 10.
    private static boolean hasValidChecksum(String digits) {
        int sum = 0;
        for (int i = 0; i < 13; i++) {
            int d = digits.charAt(i) - '0';
            sum += (i % 2 == 0) ? d : d * 3;
        }
        return sum % 10 == 0;
    }

    public String value() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        return this == o || (o instanceof Isbn other && value.equals(other.value));
    }

    @Override
    public int hashCode() {
        return value.hashCode();
    }

    @Override
    public String toString() {
        return value;
    }
}
```

**Checksum worked example** for `978-0-13-468599-1` (13 digits including the final check digit; weights alternate 1, 3, 1, 3, …):

```
digit : 9  7  8  0  1  3  4  6  8  5  9  9  1
weight: 1  3  1  3  1  3  1  3  1  3  1  3  1
prod  : 9 21  8  0  1  9  4 18  8 15  9 27  1   → total 130 → 130 % 10 == 0 ✅
```

Change the last digit to 2 and the total becomes 131, so the ISBN is rejected. The test `Isbn.of("978-0-13-468599-2")` proves exactly that.

### `domain/Author.java`

```java
package com.library.domain;

import java.util.Objects;

/** Immutable value type. A record gives equals/hashCode/toString for free (EJ Items 10-12). */
public record Author(String name) {
    public Author {
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) {
            throw new IllegalArgumentException("Author name must not be blank");
        }
        name = name.strip();
    }
}
```

The **compact constructor** (`public Author { ... }`) validates and can normalise (`name = name.strip()`) *before* the fields are assigned.

### `domain/ItemId.java`

```java
package com.library.domain;

import java.util.Objects;

/** Identity of one physical copy on the shelf (two copies of the same book have different ItemIds). */
public record ItemId(String value) {
    public ItemId {
        Objects.requireNonNull(value, "value");
        if (value.isBlank()) {
            throw new IllegalArgumentException("ItemId must not be blank");
        }
    }

    @Override
    public String toString() {
        return value;
    }
}
```

### `domain/MemberId.java`

```java
package com.library.domain;

import java.util.Objects;

public record MemberId(String value) {
    public MemberId {
        Objects.requireNonNull(value, "value");
        if (value.isBlank()) {
            throw new IllegalArgumentException("MemberId must not be blank");
        }
    }

    @Override
    public String toString() {
        return value;
    }
}
```

Tiny `ItemId`/`MemberId` records instead of raw `String`s: `borrow(MemberId, ItemId)` can't be called with its arguments swapped, since that won't compile. This is the "avoid primitive obsession" idea.

---

## 4. Code — Item Hierarchy

### `domain/Borrowable.java`

```java
package com.library.domain;

/**
 * Capability interface: only items that can leave the building implement it.
 * Kept separate from LibraryItem so that reference-only items are not forced to
 * implement borrowing methods they can't honour (Interface Segregation).
 */
public interface Borrowable {

    /** Standard loan length in days for this kind of item. */
    int loanPeriodDays();

    /** Daily late fee in cents. */
    long dailyLateFeeCents();
}
```

### `domain/LibraryItem.java`

```java
package com.library.domain;

import java.util.Objects;

/**
 * Skeletal base class for everything in the catalogue: identity, title and
 * entity equality by id. Subclasses add their own attributes.
 */
public abstract class LibraryItem {
    private final ItemId id;
    private final String title;

    protected LibraryItem(ItemId id, String title) {
        this.id = Objects.requireNonNull(id, "id");
        Objects.requireNonNull(title, "title");
        if (title.isBlank()) {
            throw new IllegalArgumentException("Title must not be blank");
        }
        this.title = title.strip();
    }

    public final ItemId id() {
        return id;
    }

    public final String title() {
        return title;
    }

    /** Short human-readable type label, e.g. "Book". */
    public abstract String kind();

    public final boolean isBorrowable() {
        return this instanceof Borrowable;
    }

    // Entity semantics: two LibraryItems are the same item iff they have the same id.
    // final so that subclasses can't break symmetry (EJ Item 10).
    @Override
    public final boolean equals(Object o) {
        return this == o || (o instanceof LibraryItem other && id.equals(other.id));
    }

    @Override
    public final int hashCode() {
        return id.hashCode();
    }

    @Override
    public String toString() {
        return kind() + "[" + id + ", \"" + title + "\"]";
    }
}
```

### `domain/Book.java`

```java
package com.library.domain;

import java.util.List;
import java.util.Objects;

public final class Book extends LibraryItem implements Borrowable {
    private final Isbn isbn;
    private final List<Author> authors;
    private final int publicationYear;

    public Book(ItemId id, String title, Isbn isbn, List<Author> authors, int publicationYear) {
        super(id, title);
        this.isbn = Objects.requireNonNull(isbn, "isbn");
        Objects.requireNonNull(authors, "authors");
        if (authors.isEmpty()) {
            throw new IllegalArgumentException("A book needs at least one author");
        }
        this.authors = List.copyOf(authors);   // defensive, unmodifiable copy (EJ Item 17)
        this.publicationYear = publicationYear;
    }

    public Isbn isbn() {
        return isbn;
    }

    public List<Author> authors() {
        return authors;                        // already unmodifiable, safe to hand out
    }

    public int publicationYear() {
        return publicationYear;
    }

    @Override
    public String kind() {
        return "Book";
    }

    @Override
    public int loanPeriodDays() {
        return 21;
    }

    @Override
    public long dailyLateFeeCents() {
        return 25;
    }
}
```

### `domain/Dvd.java`

```java
package com.library.domain;

public final class Dvd extends LibraryItem implements Borrowable {
    private final int runtimeMinutes;

    public Dvd(ItemId id, String title, int runtimeMinutes) {
        super(id, title);
        if (runtimeMinutes <= 0) {
            throw new IllegalArgumentException("runtimeMinutes must be positive");
        }
        this.runtimeMinutes = runtimeMinutes;
    }

    public int runtimeMinutes() {
        return runtimeMinutes;
    }

    @Override
    public String kind() {
        return "DVD";
    }

    @Override
    public int loanPeriodDays() {
        return 7;
    }

    @Override
    public long dailyLateFeeCents() {
        return 100;
    }
}
```

### `domain/ReferenceBook.java`

```java
package com.library.domain;

/** Encyclopedias, atlases… can be read in the library but never borrowed: NOT Borrowable. */
public final class ReferenceBook extends LibraryItem {
    private final Isbn isbn;

    public ReferenceBook(ItemId id, String title, Isbn isbn) {
        super(id, title);
        this.isbn = java.util.Objects.requireNonNull(isbn, "isbn");
    }

    public Isbn isbn() {
        return isbn;
    }

    @Override
    public String kind() {
        return "Reference";
    }
}
```

---

## 5. Code — Members & Policies

### `policy/MembershipPolicy.java`

```java
package com.library.policy;

/** Strategy: the rules that differ per membership tier. Member HAS-A policy (composition). */
public interface MembershipPolicy {

    String tierName();

    int maxActiveLoans();

    /** Multiplier applied to an item's loan period, e.g. 2.0 for premium members. */
    double loanPeriodMultiplier();

    /** Late fee waived entirely? */
    default boolean waivesLateFees() {
        return false;
    }
}
```

### `policy/StandardMembership.java`

```java
package com.library.policy;

public enum StandardMembership implements MembershipPolicy {
    // Enum = fixed set of stateless singletons (EJ Item 3), each a MembershipPolicy.
    STUDENT("Student", 3, 1.0),
    ADULT("Adult", 5, 1.0),
    PREMIUM("Premium", 10, 2.0) {
        @Override
        public boolean waivesLateFees() {
            return true;
        }
    };

    private final String tierName;
    private final int maxActiveLoans;
    private final double loanPeriodMultiplier;

    StandardMembership(String tierName, int maxActiveLoans, double loanPeriodMultiplier) {
        this.tierName = tierName;
        this.maxActiveLoans = maxActiveLoans;
        this.loanPeriodMultiplier = loanPeriodMultiplier;
    }

    @Override
    public String tierName() {
        return tierName;
    }

    @Override
    public int maxActiveLoans() {
        return maxActiveLoans;
    }

    @Override
    public double loanPeriodMultiplier() {
        return loanPeriodMultiplier;
    }
}
```

- `PREMIUM { ... }` is a **constant-specific class body**: that one constant overrides `waivesLateFees()`. Behaviour varies per constant without a `switch`.
- Enum constants are singletons, stateless and thread-safe, and serialise safely (EJ Item 3).

### `domain/Member.java`

```java
package com.library.domain;

import com.library.policy.MembershipPolicy;
import java.util.Objects;

public final class Member {
    private final MemberId id;
    private final String name;
    private MembershipPolicy policy;     // mutable on purpose: members can upgrade/downgrade

    public Member(MemberId id, String name, MembershipPolicy policy) {
        this.id = Objects.requireNonNull(id, "id");
        Objects.requireNonNull(name, "name");
        if (name.isBlank()) {
            throw new IllegalArgumentException("Member name must not be blank");
        }
        this.name = name.strip();
        this.policy = Objects.requireNonNull(policy, "policy");
    }

    public MemberId id() {
        return id;
    }

    public String name() {
        return name;
    }

    public MembershipPolicy policy() {
        return policy;
    }

    public void changeMembership(MembershipPolicy newPolicy) {
        this.policy = Objects.requireNonNull(newPolicy, "newPolicy");
    }

    @Override
    public boolean equals(Object o) {
        return this == o || (o instanceof Member other && id.equals(other.id));
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }

    @Override
    public String toString() {
        return "Member[" + id + ", " + name + ", " + policy.tierName() + "]";
    }
}
```

`Member` is deliberately **not** immutable: tier changes are part of the domain. Its identity (`id`) is `final`, and so are `equals`/`hashCode`, which depend **only** on it. A mutable field used in `hashCode` would corrupt any `HashMap`/`HashSet` holding the member (see #020).

---

## 6. Code — Loan (immutable event)

### `domain/Loan.java`

```java
package com.library.domain;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.Objects;
import java.util.Optional;

/** One borrowing event. Immutable: returning an item produces a new, closed Loan. */
public final class Loan {
    private final LibraryItem item;
    private final Member member;
    private final LocalDate borrowedOn;
    private final LocalDate dueOn;
    private final LocalDate returnedOn;   // null while the loan is open

    private Loan(LibraryItem item, Member member, LocalDate borrowedOn, LocalDate dueOn, LocalDate returnedOn) {
        this.item = item;
        this.member = member;
        this.borrowedOn = borrowedOn;
        this.dueOn = dueOn;
        this.returnedOn = returnedOn;
    }

    static Loan open(LibraryItem item, Member member, LocalDate borrowedOn, LocalDate dueOn) {
        Objects.requireNonNull(item, "item");
        Objects.requireNonNull(member, "member");
        Objects.requireNonNull(borrowedOn, "borrowedOn");
        Objects.requireNonNull(dueOn, "dueOn");
        if (dueOn.isBefore(borrowedOn)) {
            throw new IllegalArgumentException("dueOn before borrowedOn");
        }
        return new Loan(item, member, borrowedOn, dueOn, null);
    }

    /** Factory used by the service layer, which lives in another package. */
    public static Loan start(LibraryItem item, Member member, LocalDate borrowedOn, LocalDate dueOn) {
        return open(item, member, borrowedOn, dueOn);
    }

    public Loan close(LocalDate returnDate) {
        Objects.requireNonNull(returnDate, "returnDate");
        if (!isOpen()) {
            throw new IllegalStateException("Loan already closed");
        }
        if (returnDate.isBefore(borrowedOn)) {
            throw new IllegalArgumentException("returnDate before borrowedOn");
        }
        return new Loan(item, member, borrowedOn, dueOn, returnDate);
    }

    public boolean isOpen() {
        return returnedOn == null;
    }

    public boolean isOverdue(LocalDate today) {
        LocalDate reference = isOpen() ? today : returnedOn;
        return reference.isAfter(dueOn);
    }

    public long daysLate(LocalDate today) {
        LocalDate reference = isOpen() ? today : returnedOn;
        return Math.max(0, ChronoUnit.DAYS.between(dueOn, reference));
    }

    public LibraryItem item()      { return item; }
    public Member member()         { return member; }
    public LocalDate borrowedOn()  { return borrowedOn; }
    public LocalDate dueOn()       { return dueOn; }
    public Optional<LocalDate> returnedOn() { return Optional.ofNullable(returnedOn); }

    @Override
    public String toString() {
        return "Loan[" + item.id() + " -> " + member.id() + ", due " + dueOn
                + (isOpen() ? ", open" : ", returned " + returnedOn) + "]";
    }
}
```

- `close()` returns a **new** `Loan` (the "wither" style, like `LocalDate.plusDays`).
- `returnedOn()` returns `Optional<LocalDate>`, so the "not returned yet" state is explicit in the API, while the field itself stays a plain nullable reference. Don't make fields `Optional`; see #019, EJ Item 55.

---

## 7. Code — Exceptions

### `exception/LibraryException.java`

```java
package com.library.exception;

/** Base type so callers can catch every business-rule violation in one place. */
public class LibraryException extends RuntimeException {
    public LibraryException(String message) {
        super(message);
    }
}
```

### `exception/ItemNotFoundException.java`

```java
package com.library.exception;

public class ItemNotFoundException extends LibraryException {
    public ItemNotFoundException(String id) {
        super("No such item: " + id);
    }
}
```

### `exception/ItemNotBorrowableException.java`

```java
package com.library.exception;

public class ItemNotBorrowableException extends LibraryException {
    public ItemNotBorrowableException(String description) {
        super("Item cannot be borrowed: " + description);
    }
}
```

### `exception/ItemUnavailableException.java`

```java
package com.library.exception;

public class ItemUnavailableException extends LibraryException {
    public ItemUnavailableException(String id) {
        super("Item is already on loan: " + id);
    }
}
```

### `exception/LoanLimitExceededException.java`

```java
package com.library.exception;

public class LoanLimitExceededException extends LibraryException {
    public LoanLimitExceededException(String memberId, int limit) {
        super("Member " + memberId + " already has the maximum of " + limit + " loans");
    }
}
```

---

## 8. Code — `Library` Service

### `service/Library.java`

```java
package com.library.service;

import com.library.domain.Author;
import com.library.domain.Borrowable;
import com.library.domain.ItemId;
import com.library.domain.LibraryItem;
import com.library.domain.Loan;
import com.library.domain.Member;
import com.library.domain.MemberId;
import com.library.exception.ItemNotBorrowableException;
import com.library.exception.ItemNotFoundException;
import com.library.exception.ItemUnavailableException;
import com.library.exception.LibraryException;
import com.library.exception.LoanLimitExceededException;
import com.library.domain.Book;
import java.time.Clock;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Objects;

/**
 * Application service: owns the catalogue, members and loans, and enforces the rules.
 * The Clock is injected (EJ Item 5) so tests can control "today".
 */
public final class Library {
    private final Clock clock;
    private final Map<ItemId, LibraryItem> catalogue = new LinkedHashMap<>(); // keeps insertion order for listings
    private final Map<MemberId, Member> members = new HashMap<>();
    private final Map<ItemId, Loan> activeLoans = new HashMap<>();            // at most one open loan per item
    private final List<Loan> history = new ArrayList<>();

    public Library(Clock clock) {
        this.clock = Objects.requireNonNull(clock, "clock");
    }

    // ---- catalogue ----------------------------------------------------------------

    public void addItem(LibraryItem item) {
        Objects.requireNonNull(item, "item");
        if (catalogue.putIfAbsent(item.id(), item) != null) {
            throw new LibraryException("Duplicate item id: " + item.id());
        }
    }

    public LibraryItem findItem(ItemId id) {
        LibraryItem item = catalogue.get(id);
        if (item == null) {
            throw new ItemNotFoundException(id.value());
        }
        return item;
    }

    public List<LibraryItem> searchByTitle(String fragment) {
        String needle = fragment.toLowerCase(Locale.ROOT);
        List<LibraryItem> result = new ArrayList<>();
        for (LibraryItem item : catalogue.values()) {
            if (item.title().toLowerCase(Locale.ROOT).contains(needle)) {
                result.add(item);
            }
        }
        return result;
    }

    public List<Book> booksBy(Author author) {
        List<Book> result = new ArrayList<>();
        for (LibraryItem item : catalogue.values()) {
            if (item instanceof Book book && book.authors().contains(author)) {
                result.add(book);
            }
        }
        return result;
    }

    public boolean isAvailable(ItemId id) {
        findItem(id);
        return !activeLoans.containsKey(id);
    }

    // ---- members ------------------------------------------------------------------

    public void registerMember(Member member) {
        Objects.requireNonNull(member, "member");
        if (members.putIfAbsent(member.id(), member) != null) {
            throw new LibraryException("Duplicate member id: " + member.id());
        }
    }

    public Member findMember(MemberId id) {
        Member member = members.get(id);
        if (member == null) {
            throw new LibraryException("No such member: " + id);
        }
        return member;
    }

    // ---- loans --------------------------------------------------------------------

    public Loan borrow(MemberId memberId, ItemId itemId) {
        Member member = findMember(memberId);
        LibraryItem item = findItem(itemId);

        if (!(item instanceof Borrowable borrowable)) {
            throw new ItemNotBorrowableException(item.toString());
        }
        if (activeLoans.containsKey(itemId)) {
            throw new ItemUnavailableException(itemId.value());
        }
        int limit = member.policy().maxActiveLoans();
        if (loansOf(memberId).size() >= limit) {
            throw new LoanLimitExceededException(memberId.value(), limit);
        }

        LocalDate today = LocalDate.now(clock);
        long days = Math.round(borrowable.loanPeriodDays() * member.policy().loanPeriodMultiplier());
        Loan loan = Loan.start(item, member, today, today.plusDays(days));
        activeLoans.put(itemId, loan);
        return loan;
    }

    /** Returns the closed loan. The late fee is available via {@link #lateFeeCents(Loan)}. */
    public Loan giveBack(ItemId itemId) {
        Loan open = activeLoans.remove(itemId);
        if (open == null) {
            throw new LibraryException("Item is not on loan: " + itemId);
        }
        Loan closed = open.close(LocalDate.now(clock));
        history.add(closed);
        return closed;
    }

    public long lateFeeCents(Loan loan) {
        if (loan.member().policy().waivesLateFees()) {
            return 0;
        }
        Borrowable borrowable = (Borrowable) loan.item();   // safe: only Borrowables are ever loaned
        return loan.daysLate(LocalDate.now(clock)) * borrowable.dailyLateFeeCents();
    }

    public List<Loan> loansOf(MemberId memberId) {
        List<Loan> result = new ArrayList<>();
        for (Loan loan : activeLoans.values()) {
            if (loan.member().id().equals(memberId)) {
                result.add(loan);
            }
        }
        return result;
    }

    public List<Loan> overdueLoans() {
        LocalDate today = LocalDate.now(clock);
        List<Loan> result = new ArrayList<>();
        for (Loan loan : activeLoans.values()) {
            if (loan.isOverdue(today)) {
                result.add(loan);
            }
        }
        return result;
    }

    public Collection<LibraryItem> catalogue() {
        return List.copyOf(catalogue.values());
    }

    public List<Loan> history() {
        return List.copyOf(history);
    }
}
```

Collection choices (a preview of this week's topic, #017):

| Field | Type | Why |
|-------|------|-----|
| `catalogue` | `LinkedHashMap` | O(1) lookup by id **and** predictable insertion-order iteration for listings |
| `members` | `HashMap` | O(1) lookup; order irrelevant |
| `activeLoans` | `HashMap<ItemId, Loan>` | Key by item: "is it on loan?" is an O(1) `containsKey`, and the one-loan-per-item rule is structural |
| `history` | `ArrayList` | Append-only, iterated in order |

`loansOf(memberId)` is O(active loans). Fine at this size; a `Map<MemberId, List<Loan>>` index would make it O(1) at the cost of keeping two structures in sync.

---

## 9. Demo

### `LibraryDemo.java`

```java
package com.library;

import com.library.domain.Author;
import com.library.domain.Book;
import com.library.domain.Dvd;
import com.library.domain.Isbn;
import com.library.domain.ItemId;
import com.library.domain.Loan;
import com.library.domain.Member;
import com.library.domain.MemberId;
import com.library.domain.ReferenceBook;
import com.library.exception.LibraryException;
import com.library.policy.StandardMembership;
import com.library.service.Library;
import java.time.Clock;
import java.util.List;

public class LibraryDemo {
    public static void main(String[] args) {
        Library library = new Library(Clock.systemDefaultZone());

        Author bloch = new Author("Joshua Bloch");
        library.addItem(new Book(new ItemId("B-1"), "Effective Java", Isbn.of("978-0-13-468599-1"),
                List.of(bloch), 2018));
        library.addItem(new Book(new ItemId("B-2"), "Java Concurrency in Practice", Isbn.of("978-0-321-34960-6"),
                List.of(new Author("Brian Goetz"), bloch), 2006));
        library.addItem(new Dvd(new ItemId("D-1"), "The Social Network", 120));
        library.addItem(new ReferenceBook(new ItemId("R-1"), "Oxford English Dictionary", Isbn.of("978-0-19-861186-8")));

        library.registerMember(new Member(new MemberId("M-1"), "Asha", StandardMembership.STUDENT));

        Loan loan = library.borrow(new MemberId("M-1"), new ItemId("B-1"));
        System.out.println("Borrowed: " + loan);
        System.out.println("Books by Bloch: " + library.booksBy(bloch));

        try {
            library.borrow(new MemberId("M-1"), new ItemId("R-1"));
        } catch (LibraryException e) {
            System.out.println("Refused: " + e.getMessage());
        }

        Loan closed = library.giveBack(new ItemId("B-1"));
        System.out.println("Returned: " + closed + ", fee = " + library.lateFeeCents(closed) + " cents");
    }
}
```

Output (run on 2026-09-24):

```text
Borrowed: Loan[B-1 -> M-1, due 2026-10-15, open]
Books by Bloch: [Book[B-1, "Effective Java"], Book[B-2, "Java Concurrency in Practice"]]
Refused: Item cannot be borrowed: Reference[R-1, "Oxford English Dictionary"]
Returned: Loan[B-1 -> M-1, due 2026-10-15, returned 2026-09-24], fee = 0 cents
```

---

## 10. Tests

### `test: MutableClock.java`

```java
package com.library;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZoneOffset;

/** Test double: a Clock we can move forward to simulate days passing. */
final class MutableClock extends Clock {
    private Instant now;

    MutableClock(Instant start) {
        this.now = start;
    }

    void advanceDays(long days) {
        now = now.plus(Duration.ofDays(days));
    }

    @Override public ZoneId getZone()               { return ZoneOffset.UTC; }
    @Override public Clock withZone(ZoneId zone)    { throw new UnsupportedOperationException(); }
    @Override public Instant instant()              { return now; }
}
```

### `test: LibraryTest.java`

```java
package com.library;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertNotEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import com.library.domain.Author;
import com.library.domain.Book;
import com.library.domain.Dvd;
import com.library.domain.Isbn;
import com.library.domain.ItemId;
import com.library.domain.Loan;
import com.library.domain.Member;
import com.library.domain.MemberId;
import com.library.domain.ReferenceBook;
import com.library.exception.ItemNotBorrowableException;
import com.library.exception.ItemUnavailableException;
import com.library.exception.LoanLimitExceededException;
import com.library.exception.LibraryException;
import com.library.policy.StandardMembership;
import com.library.service.Library;
import java.time.Instant;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class LibraryTest {
    private static final MemberId STUDENT = new MemberId("M-1");
    private static final MemberId PREMIUM = new MemberId("M-2");

    private MutableClock clock;
    private Library library;

    @BeforeEach
    void setUp() {
        clock = new MutableClock(Instant.parse("2026-09-01T10:00:00Z"));
        library = new Library(clock);
        library.addItem(book("B-1", "Effective Java", "978-0-13-468599-1"));
        library.addItem(book("B-2", "Java Concurrency in Practice", "978-0-321-34960-6"));
        library.addItem(book("B-3", "Clean Code", "978-0-13-235088-4"));
        library.addItem(book("B-4", "Refactoring", "978-0-13-475759-9"));
        library.addItem(new Dvd(new ItemId("D-1"), "The Social Network", 120));
        library.addItem(new ReferenceBook(new ItemId("R-1"), "Oxford English Dictionary", Isbn.of("978-0-19-861186-8")));
        library.registerMember(new Member(STUDENT, "Asha", StandardMembership.STUDENT));
        library.registerMember(new Member(PREMIUM, "Ravi", StandardMembership.PREMIUM));
    }

    private static Book book(String id, String title, String isbn) {
        return new Book(new ItemId(id), title, Isbn.of(isbn), List.of(new Author("Some Author")), 2020);
    }

    @Test
    void isbnValidatesChecksumAndHasValueEquality() {
        assertEquals(Isbn.of("978-0-13-468599-1"), Isbn.of("9780134685991"));
        assertThrows(IllegalArgumentException.class, () -> Isbn.of("978-0-13-468599-2"));
        assertThrows(IllegalArgumentException.class, () -> Isbn.of("12345"));
    }

    @Test
    void itemsAreEqualByIdOnly() {
        Book a = book("X-1", "Title A", "978-0-13-468599-1");
        Book b = book("X-1", "Title B", "978-0-321-34960-6");
        assertEquals(a, b);
        assertEquals(a.hashCode(), b.hashCode());
        assertNotEquals(a, book("X-2", "Title A", "978-0-13-468599-1"));
    }

    @Test
    void bookAuthorsCannotBeModifiedFromOutside() {
        List<Author> authors = new ArrayList<>(List.of(new Author("A")));
        Book b = new Book(new ItemId("X-9"), "T", Isbn.of("978-0-13-468599-1"), authors, 2020);
        authors.add(new Author("Sneaky"));                           // mutate the caller's list
        assertEquals(1, b.authors().size());                         // defensive copy protected us
        assertThrows(UnsupportedOperationException.class, () -> b.authors().add(new Author("B")));
    }

    @Test
    void borrowSetsDueDateFromItemTypeAndMembership() {
        Loan bookLoan = library.borrow(STUDENT, new ItemId("B-1"));
        assertEquals(LocalDate.of(2026, 9, 22), bookLoan.dueOn());   // 21 days × 1.0

        Loan dvdLoan = library.borrow(PREMIUM, new ItemId("D-1"));
        assertEquals(LocalDate.of(2026, 9, 15), dvdLoan.dueOn());    // 7 days × 2.0
    }

    @Test
    void referenceItemsCannotBeBorrowed() {
        assertThrows(ItemNotBorrowableException.class, () -> library.borrow(STUDENT, new ItemId("R-1")));
    }

    @Test
    void anItemCanOnlyBeOnLoanOnce() {
        library.borrow(STUDENT, new ItemId("B-1"));
        assertFalse(library.isAvailable(new ItemId("B-1")));
        assertThrows(ItemUnavailableException.class, () -> library.borrow(PREMIUM, new ItemId("B-1")));
    }

    @Test
    void studentLoanLimitIsEnforced() {
        library.borrow(STUDENT, new ItemId("B-1"));
        library.borrow(STUDENT, new ItemId("B-2"));
        library.borrow(STUDENT, new ItemId("B-3"));
        assertThrows(LoanLimitExceededException.class, () -> library.borrow(STUDENT, new ItemId("B-4")));
    }

    @Test
    void lateReturnIsChargedPerDayAndPremiumIsWaived() {
        Loan studentLoan = library.borrow(STUDENT, new ItemId("B-1"));   // due 2026-09-22
        Loan premiumLoan = library.borrow(PREMIUM, new ItemId("D-1"));   // due 2026-09-15
        clock.advanceDays(25);                                           // today = 2026-09-26

        assertEquals(2, library.overdueLoans().size());

        Loan closed = library.giveBack(studentLoan.item().id());
        assertEquals(4, closed.daysLate(LocalDate.of(2026, 9, 26)));
        assertEquals(4 * 25, library.lateFeeCents(closed));              // 4 days × 25c

        Loan closedPremium = library.giveBack(premiumLoan.item().id());
        assertEquals(0, library.lateFeeCents(closedPremium));
        assertTrue(library.isAvailable(new ItemId("B-1")));
        assertEquals(2, library.history().size());
    }

    @Test
    void returningSomethingNotOnLoanFails() {
        assertThrows(LibraryException.class, () -> library.giveBack(new ItemId("B-1")));
    }

    @Test
    void upgradingMembershipChangesTheLimit() {
        library.borrow(STUDENT, new ItemId("B-1"));
        library.borrow(STUDENT, new ItemId("B-2"));
        library.borrow(STUDENT, new ItemId("B-3"));
        library.findMember(STUDENT).changeMembership(StandardMembership.ADULT);   // limit 3 → 5
        library.borrow(STUDENT, new ItemId("B-4"));
        assertEquals(4, library.loansOf(STUDENT).size());
    }

    @Test
    void searchIsCaseInsensitive() {
        assertEquals(1, library.searchByTitle("effective").size());
        assertEquals(2, library.searchByTitle("JAVA").size());
    }
}
```

```text
$ mvn verify
Tests run: 11, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

Build: reuse the `pom.xml` from #002, changing `artifactId` to `library-domain` and `mainClass` to `com.library.LibraryDemo`.

---

## 11. Stretch Goal Sketches

**Reservations:** add `Map<ItemId, Deque<MemberId>> holds` to `Library` (FIFO queue per item; `ArrayDeque`, see #017). In `giveBack`, if the item's queue isn't empty, mark it *held for* the head member until `today + 3`; `borrow` then only lets that member take it.

**Renewals:** `Loan renew(int days)` returns a new `Loan` with a later `dueOn` and a `renewals` count; `Library.renew` rejects the renewal if `holds.get(itemId)` isn't empty or the count is ≥ 1.

**Sealed hierarchy:**

```java
public abstract sealed class LibraryItem permits Book, Dvd, ReferenceBook { ... }
```

Gain: an exhaustive `switch (item) { case Book b -> …; case Dvd d -> …; case ReferenceBook r -> …; }` with no `default`, and nobody outside the codebase can add item types. Loss: adding a new item type means editing the `permits` clause. That's a good trade for a closed domain like this one; for a plugin-style catalogue, keep it open.

---

## Key Takeaways

1. **Abstract class = shared state + skeleton for an "is-a" family. Interface = capability or strategy.** One class can have many capabilities.
2. **Model capabilities as separate interfaces** (ISP), so the type system says what an object can do, and nothing throws `UnsupportedOperationException`.
3. **Composition + strategy beats a subclass per variant** whenever the variant can change at run time (membership tier).
4. **Value objects** (immutable, value equality, validated in a factory) vs **entities** (equality by a stable id) is the most useful modelling distinction in backend code.
5. **Inject the clock.** Time is a dependency like any other.
6. Defensive copies (`List.copyOf`) and `final` everywhere give you a model that callers can't corrupt.
