# LLD Lecture 16 — Detailed Notes
## Topics: Builder Pattern (Recap & Deep Dive), Singleton Pattern (Challenges & Solutions), Movie Ticket Booking System (Design Problem)

---

## 1. Builder Pattern — Recap & Complete Solution

### The Problem Statement (Recap from Previous Class)

We have an `IncidentTicket` class that is **mutable** — it has public setters and a public constructor. The `TicketService` class creates a ticket and then mutates it later, which is bad practice. Validation rules for ticket creation are **scattered** across the codebase.

**Goals:**
1. Make `IncidentTicket` **immutable**
2. Provide a clean way to create tickets with **validation**
3. Centralize all validation logic in one place

---

### The Design Journey (V0 to Final)

#### V0 — The Problematic Mutable Design

```java
class IncidentTicket {
    private String email;
    private int id;
    private String title;
    // ... other fields

    // Public constructor — anyone can create objects
    public IncidentTicket(int id, String email, String title) { ... }

    // Public setters — anyone can mutate the object after creation
    public void setEmail(String email) { this.email = email; }
    public void setId(int id) { this.id = id; }
    public void setTitle(String title) { this.title = title; }

    // Public getters
    public String getEmail() { return email; }
    // ...
}
```

```java
class TicketService {
    public IncidentTicket createTicket() {
        IncidentTicket ticket = new IncidentTicket(1, "a@b.com", "Issue");
        // Later, mutating the ticket — BAD!
        ticket.setTitle("Changed Issue");
        return ticket;
    }
}
```

**Problems with V0:**

| Problem | Explanation |
|---------|-------------|
| **Mutable object** | Anyone can change any field at any time after creation |
| **No validation gate** | Object can be created with invalid data |
| **Scattered validation** | Validation checks are spread across multiple methods/classes |
| **Unsafe to share** | If you pass this object to another class, they can modify it |

---

#### V1 — Checklist of Changes (Step-by-Step Transformation)

The teacher outlined a clear **checklist** to transform the mutable class into an immutable one:

| Step | Action | Why |
|------|--------|-----|
| 1 | **Remove all setters** | Prevent external mutation of fields |
| 2 | **Make all fields `final`** | Once a value is set, it can never be changed |
| 3 | **Make the constructor `private`** | Prevent external creation of objects via `new` |
| 4 | **Create a static inner Builder class** | Provide a controlled way to create objects |
| 5 | **Add validation in the `build()` method** | Centralize all validation before object creation |
| 6 | **Getters return deep copies** | Prevent indirect mutation of internal objects |

---

#### V2 — The Final Immutable Design with Builder

```java
public final class IncidentTicket {
    // Step 2: All fields are final
    private final String email;
    private final int id;
    private final String title;
    private final String normalId;

    // Step 3: Private constructor — only Builder can call this
    private IncidentTicket(Builder builder) {
        this.email = builder.email;
        this.id = builder.id;
        this.title = builder.title;
        this.normalId = builder.normalId;
    }

    // Getters only (no setters) — return deep copies for objects
    public String getEmail() { return email; }
    public int getId() { return id; }
    public String getTitle() { return title; }

    // ---- Static Inner Builder Class ----
    public static class Builder {
        // Builder has the SAME fields as IncidentTicket (mutable copy)
        private String email;
        private int id;
        private String title;
        private String normalId;

        // Each setter returns the Builder itself (for chaining)
        public Builder id(int id) {
            this.id = id;
            return this;      // <-- KEY: returns 'this' for chaining
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public Builder title(String title) {
            this.title = title;
            return this;
        }

        public Builder normalId(String normalId) {
            this.normalId = normalId;
            return this;
        }

        // "from" method — create a builder pre-filled from an existing ticket
        public static Builder from(IncidentTicket ticket) {
            Builder builder = new Builder();
            builder.id = ticket.getId();
            builder.email = ticket.getEmail();
            builder.title = ticket.getTitle();
            // ... copy all fields
            return builder;
        }

        // Build method — validates, then creates the immutable object
        public IncidentTicket build() {
            // Step 5: Centralized validation BEFORE object creation
            Validation.checkEmail(this.email);
            Validation.checkId(this.id);
            Validation.checkTitle(this.title);
            // ... all other validation rules

            // Only if all validations pass:
            return new IncidentTicket(this);
        }
    }
}
```

**Usage — Method Chaining:**

```java
IncidentTicket ticket = new IncidentTicket.Builder()
    .id(101)
    .email("user@example.com")
    .title("Login issue")
    .normalId("N-101")
    .build();    // Validation happens here, then object is created
```

**Usage — Modifying an Immutable Object (Creating a New Copy):**

```java
// You want to change just the title. The old ticket remains untouched.
IncidentTicket updatedTicket = IncidentTicket.Builder
    .from(existingTicket)     // Pre-fill from existing ticket
    .title("Updated title")   // Override just the title
    .build();                 // Creates a NEW ticket
```

> **Teacher's Key Insight:** The essence of immutability is that the previous ticket remains as-is. If you make any changes, you create a NEW ticket to reflect those changes. The old object is never modified.

---

### Why the Builder Class is `static`

The `Builder` class must be `static` because:
- The parent class (`IncidentTicket`) has a **private constructor**
- You cannot create a parent object first (which a non-static inner class would require)
- With `static`, you can call `new IncidentTicket.Builder()` directly using the class name

---

### Validation Inside Build

```java
// INSIDE build() method — single point of validation
public IncidentTicket build() {
    Validation.checkEmail(this.email);
    Validation.checkId(this.id);
    Validation.checkTitle(this.title);
    // All rules checked in ONE place
    return new IncidentTicket(this);
}
```

> **Teacher's Insight on Open-Closed Principle:** Chaining methods does not violate OCP because none of the existing methods are being changed. However, having multiple validation calls scattered in `build()` could violate it. **Better approach:** Call a single `validate()` method inside `build()`, and that method internally handles all validations. This way, the `build()` method itself doesn't change when new validation rules are added.

---

### Important Design Decisions — Builder Pattern

| Question | Answer |
|----------|--------|
| Is making the constructor private **required** by the Builder pattern? | **No.** It is not a requirement of the pattern. However, it makes sense because once an immutable object is created through the builder, there is no need to expose the constructor. |
| Can reflection APIs violate the private constructor? | **Yes.** But Builder's primary purpose is NOT to restrict object creation (that's Singleton). Builder is for **constructing** objects with validation. |
| Does the Builder pattern ensure only one object? | **No.** Builder creates different objects. Singleton ensures only one object. |

> **Teacher's Insight on Abstraction:** "When we say a function or feature is hidden from someone, it doesn't mean we don't want them to see the code. The code is available — like Java Collections source code. Abstraction means they should NOT NEED to see the code to use it. That saves their time and effort."

---

## 2. Singleton Pattern — Complete Deep Dive

### What is Singleton?

A Singleton is a class where you want **only one instance** to exist throughout the application's lifetime.

### Basic Singleton (Meyer's Singleton)

```java
public class Singleton {
    private static Singleton instance;

    // Private constructor — prevents external instantiation
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

---

### Challenges to Singleton (and Solutions)

#### Challenge 1: Thread Safety

**Problem:** Multiple threads can race to the `if (instance == null)` check. Two threads could both see `null` and both create instances.

**Solution 1 — Synchronized Block:**

```java
public static Singleton getInstance() {
    synchronized (Singleton.class) {
        if (instance == null) {
            instance = new Singleton();
        }
    }
    return instance;
}
```

**Problem with Solution 1:** Every thread acquires a lock, even after the instance is already created. This is wasteful.

**Solution 2 — Double-Checked Locking (Optimization):**

```java
public static Singleton getInstance() {
    if (instance == null) {                      // First check (no lock)
        synchronized (Singleton.class) {
            if (instance == null) {              // Second check (with lock)
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

| Step | What Happens |
|------|-------------|
| First `if` | Threads skip the synchronized block entirely if instance already exists |
| `synchronized` | Only one thread enters the critical section |
| Second `if` | Prevents the second thread (that was waiting) from creating another instance |

> **Key Point:** Threads only acquire a lock if `instance == null`. After the instance is created, no thread ever acquires a lock again. This is a significant performance optimization.

---

#### Challenge 2: Volatile Keyword

**Problem:** In multi-core systems, Thread A and Thread B may run on different CPU cores with **separate local caches**. When Thread A creates the singleton instance, it updates its local cache but NOT Thread B's cache. Thread B may still see `null`.

**Solution:** Mark the instance as `volatile`.

```java
private static volatile Singleton instance;
```

| Without `volatile` | With `volatile` |
|--------------------|-----------------|
| Each core caches independently | Cache coherency algorithm runs on every read/write |
| Thread B may see stale `null` value | All cores see the updated value immediately |
| Multiple instances possible | Guaranteed single instance |

> **Teacher's Special Insight:** "The `volatile` keyword in Java is VERY different from `volatile` in C/C++. In C/C++, `volatile` means the compiler should NOT optimize the variable out (i.e., don't remove the assembly code). In Java, `volatile` means ensure cache coherency across CPU cores."

---

#### Challenge 3: Deserialization Attack

**Problem:** If your singleton class implements `Serializable`, someone can serialize the instance to a binary stream and then deserialize it, creating a **new copy**.

**Solution:** Override the `readResolve()` method.

```java
public class Singleton implements Serializable {
    private static volatile Singleton instance;

    // ... constructor, getInstance, etc.

    // Prevent deserialization from creating a new instance
    protected Object readResolve() {
        throw new RuntimeException("Deserialization of Singleton is not allowed!");
    }
}
```

> **Teacher's Insight:** Some code returns `getInstance()` from `readResolve()`, but this doesn't make sense because the binary data being deserialized could have a very different structure from your actual singleton. **Better to just throw an exception.**

---

#### Challenge 4: Reflection Attack

**Problem:** Using Java's Reflection API, someone can:
1. Access the private constructor
2. Change its accessibility from private to public
3. Call the constructor directly — creating multiple instances

**Naive Solution:**

```java
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance() method!");
    }
}
```

**Why the Naive Solution Fails:**

| Attack Order | Result |
|-------------|--------|
| Call `getInstance()` first, then reflection attack | **Blocked** — `instance` is not null, exception thrown |
| Reflection attack first (never call `getInstance()`) | **NOT blocked** — `instance` is still null, constructor executes normally |

> **Key Insight:** The attacker can simply skip `getInstance()` and directly invoke the constructor via reflection. Since `instance` is `null` at that point, the guard check fails.

**Teacher's Open Question:** If the constructor is made public via reflection and multiple threads call it simultaneously, does the constructor need to be thread-safe? This was left as an exercise for students to explore.

---

#### Challenge 5: Cloneable Interface

**Problem:** If your singleton class (or any parent class) implements `Cloneable`, someone can call `.clone()` to create a deep copy.

**Solution:** Override `clone()` and throw an exception.

```java
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Cloning of Singleton is not allowed!");
}
```

---

### The Enum Solution (Recommended)

```java
public enum Singleton {
    INSTANCE;

    // Add your methods here
    public void doSomething() {
        // ...
    }
}
```

**Why Enum solves everything:**

| Challenge | Enum Handles It? |
|-----------|-----------------|
| Thread Safety | Yes (internally guaranteed by JVM) |
| Deserialization | Yes (protected automatically) |
| Reflection Attack | Yes (protected automatically) |
| Cloning | Yes (protected automatically) |

---

### Inner Class Singleton (Lazy Loading + Thread Safe)

```java
public class Singleton {
    private Singleton() {
        // Guard against reflection
        if (InnerClass.INSTANCE != null) {
            throw new RuntimeException("Use getInstance()!");
        }
    }

    private static class InnerClass {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return InnerClass.INSTANCE;
    }
}
```

| Property | Explanation |
|----------|-------------|
| **Lazy Loading** | `InnerClass` is not loaded until `getInstance()` is called |
| **Thread Safe** | Class loading in Java is thread-safe by JVM specification |
| **No synchronized needed** | JVM handles it internally during class loading |
| **Private constructor** | Safe from direct instantiation |

> **Teacher's Note:** "There were two implementations you were supposed to explore — Enum and Inner Class. The Inner Class is a solution which is thread-safe AND lazy-loading."

---

### Singleton — Complete Summary Table

| Approach | Thread Safe | Lazy Loading | Reflection Safe | Deserialization Safe | Clone Safe |
|----------|:-----------:|:------------:|:---------------:|:--------------------:|:----------:|
| Basic (Meyer's) | No | Yes | No | No | No |
| Synchronized | Yes | Yes | No | No | No |
| Double-Checked Locking + Volatile | Yes | Yes | No | No | No |
| Inner Class | Yes | Yes | Partially (with guard) | No | No |
| Enum | Yes | No (eager) | Yes | Yes | Yes |

---

## 3. Movie Ticket Booking System — Design Problem

### The Problem Statement

Design a system for **booking movie tickets** (similar to BookMyShow).

---

### Required APIs (From Teacher's Discussion)

#### Customer-Facing APIs

| # | API | Input | Output |
|---|-----|-------|--------|
| 1 | Get all theatres in a city | City name | List of theatres |
| 2 | Get all movies in a city | City name | List of movies playing in theatres of that city |
| 3 | Get shows for a movie | Movie ID (+ city) | List of shows (theatre + timing) |
| 4 | Get seat map for a show | Show ID | Seat map with status (booked/free) |
| 5 | Get total amount | Show ID + selected seats | Total bill |
| 6 | Make payment | Payment details + method | Booking confirmation |
| 7 | Cancel booking + refund | Booking ID | Refund processed via original payment method |

#### Admin APIs

| # | API | Input | Output |
|---|-----|-------|--------|
| 1 | Add Movie | Movie details | Movie created |
| 2 | Add Theatre | Theatre details | Theatre created |
| 3 | Add Show | Show details (movie, theatre, screen, timing) | Show created |

---

### User Flow (Step-by-Step)

```
Home Screen (Select City - dropdown)
        |
   +---------+---------+
   |                   |
Movies Path         Theatres Path
   |                   |
List all movies     List all theatres
in city              in city
   |                   |
Select a movie      Select a theatre
   |                   |
List all shows      List all movies/shows
(theatre + timing)   for that theatre
   |                   |
Select a show       Select a show
   |                   |
   +------- MERGE -----+
            |
    Show Seat Map
    (booked vs free)
            |
    Select Seats
            |
    Generate Bill
            |
    Select Payment Method
    (UPI / Card / Net Banking)
            |
    Make Payment
            |
    Booking Confirmed
```

---

### Key Design Requirements & Constraints

#### Entity Relationships

```
City
  └── Theatre (many theatres per city)
        └── Screen (many screens per theatre)
              └── Seat (many seats per screen, each has a category)
              └── Show (many shows per screen)
                    └── Movie (one movie per show)
                    └── Timing (start time, end time)
```

> **Teacher's Clarification:** "Inside one theatre there can be multiple screens. Inside one screen, there can be multiple shows at different timings."

#### User Identification

- **Email address** is the unique identifier for users

---

### Pricing Strategy (Critical Design Decision)

| Factor | Details |
|--------|---------|
| **Base Price** | Determined by seat **category** (Gold, Diamond, etc.) — same for all movies in a given theatre |
| **Show-based Increment** | Price can increase above base price depending on the show |
| **Never decreases** | Price will never go below the base price |
| **Dynamic Factors** | Day of week, week of month, demand (how many seats already filled) |
| **Same category, different prices** | Even within the same category, different shows can have different prices |
| **Demand-based pricing** | If only 2 seats remain out of 100, price may be higher than what the first 98 were booked at |

> **Teacher's Clarification:** "Pricing is NOT set by a person manually. It is **rule-based**. The admin selects which pricing rules to apply, and the system calculates prices automatically. Our design must allow injecting these rules, and the admin should be able to pick rules at runtime."

**Design Implication:** This screams for a **Strategy Pattern** for pricing!

```
PricingStrategy (interface)
  ├── BasePriceStrategy
  ├── DayOfWeekStrategy
  ├── DemandBasedStrategy
  ├── WeekOfMonthStrategy
  └── ... (new strategies can be added without modifying existing code)
```

---

### Concurrency Handling (Seat Booking)

**Scenario:** Two users open the same show's seat map simultaneously.

| Step | User A | User B |
|------|--------|--------|
| 1 | Opens seat map — all seats show free | Opens seat map — all seats show free |
| 2 | Selects seats 5, 6, 7 | Still sees all seats as free (no live updates) |
| 3 | Proceeds to payment | Selects seats 5, 6, 7 |
| 4 | Seats 5, 6, 7 are **temporarily locked** for User A | Proceeds to payment |
| 5 | — | Gets error: "Seats no longer available. Select again." |

**Key Rules:**

| Rule | Details |
|------|---------|
| **No live updates** | When User A selects seats, User B's screen does NOT update in real time |
| **Temporary lock** | When a user proceeds to payment, seats are locked for a configurable duration |
| **Lock expiry** | If payment is not completed within the duration, seats are freed |
| **Permanent booking** | If payment succeeds, seats are permanently booked |
| **Conflict resolution** | If two users select the same seats, only one succeeds. The other gets an error message |

> **Teacher's Insight:** "This is the same thing you handled in the parking lot — same slot cannot be assigned to two users at the same time."

**Design Implication:** This requires **locking/synchronization** similar to the parking lot design.

---

### Cancellation & Refund

| Requirement | Details |
|-------------|---------|
| Users can cancel bookings | Cancel API required |
| Refund via original payment method | If paid by UPI, refund goes to UPI |
| No internal currency | No tokens/coins/wallet — direct refund |

---

### Payment System

- Support multiple payment methods: **UPI, Card, Net Banking**
- External payment APIs will be used (not built internally)

**Design Implication:** This is a perfect use case for the **Strategy Pattern** for payments.

```
PaymentStrategy (interface)
  ├── UPIPayment
  ├── CardPayment
  └── NetBankingPayment
```

---

### What is NOT Required

| Not Required |
|-------------|
| Discount policies / coupon codes |
| Add-ons (food, drinks, etc.) |
| Internal currency (tokens, coins) |
| Live seat map updates |
| Movies auto-added to all theatres (shows must be explicitly created) |

---

### Admin Flow for Data Setup

```
1. Admin adds Theatres (with screens and seats per screen)
2. Admin adds Movies (movie details, certificate, etc.)
3. Admin creates Shows (linking a Movie + Theatre Screen + Timing)
```

> **Key Clarification:** "Adding a movie does NOT automatically add it to all theatres. You must explicitly create shows which link the movie to a specific theatre screen and timing."

---

## 4. Interview Questions

### Builder Pattern

1. **What is the Builder pattern? When would you use it?**
   - Used when constructing complex objects that require validation and have many fields. Especially useful for creating immutable objects.

2. **How does the Builder pattern help create immutable objects?**
   - Builder accumulates all field values in a mutable intermediate object, validates them, then creates the immutable target object in one shot via a private constructor.

3. **Why does the Builder class return `this` from each setter method?**
   - To enable method chaining: `.id(1).email("a@b.com").title("X").build()`

4. **How do you modify an immutable object?**
   - You don't. You create a NEW object using `Builder.from(existingObject)`, override the fields you want, and call `.build()`.

5. **Why is the Builder class static?**
   - Because the parent class has a private constructor, you cannot create a parent object first. A static inner class can be instantiated independently.

### Singleton Pattern

6. **What is Double-Checked Locking? Why is it needed?**
   - It adds an `if (instance == null)` check before the `synchronized` block so that threads don't acquire a lock once the instance is already created. Optimization over plain synchronized.

7. **Why do we need the `volatile` keyword with Singleton?**
   - To ensure cache coherency across CPU cores. Without it, Thread B on a different core might see a stale `null` value from its local cache.

8. **How does deserialization break Singleton?**
   - Deserializing a binary stream creates a new object, bypassing the private constructor. Override `readResolve()` and throw an exception.

9. **How does reflection break Singleton?**
   - Reflection can access the private constructor, make it public, and invoke it. Guard with `if (instance != null) throw exception` in the constructor, but this is not foolproof.

10. **What is the most robust way to implement Singleton in Java?**
    - Use an **Enum**. It handles thread safety, deserialization, reflection, and cloning automatically.

11. **Difference between Enum singleton and Inner Class singleton?**
    - Enum is eager (loaded when class is loaded), Inner Class is lazy (loaded only when `getInstance()` is called). Enum provides more protection out of the box.

### Movie Booking System

12. **How would you handle concurrent seat booking?**
    - Lock seats temporarily when a user proceeds to payment. Use synchronized/locking mechanisms. Lock expires after a configurable timeout.

13. **How would you design a dynamic pricing system?**
    - Strategy pattern. Define a `PricingStrategy` interface. Implement strategies for different rules (demand, day of week, etc.). Admin selects which rules to apply at runtime.

---

## 5. Teacher's Special Insights & Career Tips

| # | Insight |
|---|---------|
| 1 | **Abstraction is NOT about hiding code.** "Abstraction means they should NOT NEED to see the code. The code is available — like Collections source code. You don't need to understand how HashMap works internally to use it." |
| 2 | **Builder vs Singleton distinction:** Builder is for constructing different objects with validation. Singleton is for restricting to one object. Don't confuse the two. |
| 3 | **`volatile` in Java vs C/C++:** Completely different semantics. Java = cache coherency. C/C++ = don't optimize out the variable. |
| 4 | **Immutability principle:** The old object is NEVER modified. You always create a new object for changes. This is the essence of immutable design. |
| 5 | **Validation centralization:** All validation should go through one method. Don't scatter checks across the codebase. |
| 6 | **Real system design:** Start by asking "What are the APIs?" If you don't ask this, you'll probably come up with the wrong design. |
| 7 | **Team design strategy:** For large systems — list requirements, create V0 class diagram, divide work, each person refines their part and implements. |
| 8 | **Deep copies from getters:** Immutable classes must ensure getters return deep copies of objects, not references to internal state. |
| 9 | **OCP and chaining:** Method chaining doesn't violate OCP. But multiple validation calls in `build()` could. Use a single `validate()` method instead. |

---

## 6. Exam Prep — Quick-Fire Facts

### Builder Pattern

| Fact | Detail |
|------|--------|
| Builder is a **Creational** pattern | Used to construct complex objects step-by-step |
| Builder class is **static inner class** | So it can be created without a parent instance |
| Setter methods return `this` | Enables method chaining |
| `build()` method does validation | Single point of validation before object creation |
| `from()` method | Creates a pre-filled builder from an existing immutable object |
| Parent constructor is **private** | Only Builder can create the parent object |
| Parent fields are **final** | Cannot be changed after construction |

### Singleton Pattern

| Fact | Detail |
|------|--------|
| Singleton is a **Creational** pattern | Ensures only one instance of a class |
| Double-Checked Locking | Two `if (instance == null)` checks — one outside, one inside `synchronized` |
| `volatile` keyword | Required for cache coherency across CPU cores in Java |
| Reflection attack | Access private constructor, make it public, call it |
| Deserialization attack | Override `readResolve()`, throw exception |
| Clone attack | Override `clone()`, throw exception |
| **Enum singleton** | Most robust — handles all attacks automatically |
| **Inner class singleton** | Lazy-loading + thread-safe via JVM class loading |

### Movie Booking System

| Fact | Detail |
|------|--------|
| Two user types | Customer and Admin |
| User identity | Email address (unique key) |
| Seat pricing | Base price (by category) + show-based increment (never decreases) |
| Pricing rules | Rule-based, admin selects rules, system calculates automatically |
| Concurrent booking | Temporary lock on seats during payment, configurable timeout |
| Cancellation | Refund via original payment method, no internal currency |
| Theatre structure | Theatre -> Screens -> Seats (with categories) + Shows |
| Show creation | Explicit — movie is NOT auto-added to theatres |
| No discounts/coupons | Not required in this design |
| No add-ons | Not required in this design |

---

## 7. Key Vocabulary

| Term | Definition |
|------|-----------|
| **Immutable Class** | A class whose instances cannot be modified after creation. All fields are `final`, no setters, constructor is private. |
| **Method Chaining** | Calling multiple methods in sequence on the same object by having each method return `this`. |
| **Double-Checked Locking** | An optimization for thread-safe singleton that avoids acquiring a lock after the instance is already created. |
| **Volatile (Java)** | Ensures a variable is always read from and written to main memory, triggering cache coherency across CPU cores. |
| **Reflection** | A language feature to inspect and modify class structure (fields, methods, constructors) at runtime. |
| **Deserialization** | Converting a binary/text stream back into an object. In Java, it's a binary stream. |
| **Cache Coherency** | An algorithm that ensures all CPU core caches have the same up-to-date value for shared variables. |
| **Lazy Loading** | Delaying object creation until it is actually needed (first access). |
| **Eager Loading** | Creating the object as soon as the class is loaded, regardless of whether it's needed. |

---

## 8. Homework / Action Items from the Teacher

1. **Explore Inner Class Singleton** — Understand how it achieves lazy loading + thread safety
2. **Reflection attack on constructor** — If the private constructor is made public via reflection and called by multiple threads, does the constructor need to be thread-safe? Find the answer.
3. **Design the Movie Ticket Booking System** — Work in groups of 4-6, list requirements, create V0 class diagram, divide work, implement
4. **Be ready to present** — One team/student will present their booking system design in the next class
5. **Prepare for VIVAs** — Covers all OOP principles, design patterns, homeworks, assignments, and design problems covered so far
