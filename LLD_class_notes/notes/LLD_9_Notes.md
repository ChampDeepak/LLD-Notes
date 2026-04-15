# LLD Lecture 9 -- Decorator Pattern, Adapter Pattern, and SOLID Principles Review

---

## Table of Contents

1. [Decorator Design Pattern -- The Full Design Journey](#1-decorator-design-pattern----the-full-design-journey)
2. [Adapter Design Pattern -- Real-World Merger Problem](#2-adapter-design-pattern----real-world-merger-problem)
3. [SOLID Principles Assignment Walkthrough (Q3 and Q4)](#3-solid-principles-assignment-walkthrough)
4. [Teacher's Special Insights and Career Tips](#4-teachers-special-insights-and-career-tips)
5. [Interview Questions](#5-interview-questions)
6. [Exam Prep -- Quick-Fire Revision](#6-exam-prep----quick-fire-revision)

---

## 1. Decorator Design Pattern -- The Full Design Journey

### Problem Statement

You have a notification framework with different notifier types (Email, SMS, WhatsApp, Slack). A client already holds a notifier object. **At runtime**, you need to add extra features (extra notification channels) to that same object **without** modifying existing code.

---

### V0 -- The Naive Thought Process (What NOT To Do)

| Idea | Why It Fails |
|------|-------------|
| Copy-paste the WhatsApp notification code into a new combined class | Same code at two places -- violates DRY principle |
| Create a new object of WhatsApp notifier inside the wrapper and call its `notify()` | Violates Dependency Inversion Principle (DIP) -- though the teacher notes it is "not a very big violation" since the wrapper is specific to WhatsApp, it is still not ideal |

---

### V1 -- The Wrapper Approach (Building Towards Decorator)

**Key insight from the teacher:** We need a class that:
1. **Implements the same `Notifier` interface** -- so the client sees no difference.
2. **Holds an object of `Notifier`** (injected via constructor) -- so it can delegate to whatever notifier was passed in.
3. **Extends the concrete notifier class** (e.g., `WhatsAppNotifier`) -- so it inherits all the notification-sending capabilities without creating a separate internal object.

#### Step-by-Step Construction

**Step 1:** Create a `WhatsAppWrapper` class.

```java
// Step 1: Wrapper implements the same interface
public class WhatsAppWrapper extends WhatsAppNotifier implements Notifier {

    private Notifier wrappedNotifier;  // holds the object being decorated

    // Step 2: Inject any Notifier through constructor
    public WhatsAppWrapper(Notifier notifier) {
        this.wrappedNotifier = notifier;
    }

    // Step 3: First delegate, then add own behavior
    @Override
    public void notify() {
        wrappedNotifier.notify();   // whatever the wrapped object does, do that first
        super.notify();             // then send WhatsApp notification (inherited from WhatsAppNotifier)
    }
}
```

**Why `extends WhatsAppNotifier`?**
- The WhatsApp notifier class may have multiple helper methods needed to actually send a WhatsApp message.
- By extending it, the wrapper gets all those capabilities for free.
- Avoids creating an internal object of `WhatsAppNotifier` (which would violate DIP).

**Step 2:** Create an `SMSWrapper` class in the exact same fashion.

```java
public class SMSWrapper extends SMSNotifier implements Notifier {

    private Notifier wrappedNotifier;

    public SMSWrapper(Notifier notifier) {
        this.wrappedNotifier = notifier;
    }

    @Override
    public void notify() {
        wrappedNotifier.notify();   // delegate first
        super.notify();             // then send SMS
    }
}
```

---

### Final Design -- Chaining Decorators at Runtime

This is where the magic happens. You chain wrappers by passing one wrapper into the constructor of another:

```java
// Start with a base notifier
Notifier n = new EmailNotifier();

// Decorate with WhatsApp
n = new WhatsAppWrapper(n);   // now n does: Email + WhatsApp

// Decorate with SMS
n = new SMSWrapper(n);        // now n does: Email + WhatsApp + SMS

// Single call triggers all three
n.notify();
```

#### Execution Flow (Call Stack Trace)

```
n.notify()                          // SMSWrapper.notify()
  |
  +-- wrappedNotifier.notify()      // WhatsAppWrapper.notify()
  |     |
  |     +-- wrappedNotifier.notify() // EmailNotifier.notify()
  |     |     --> SENDS EMAIL
  |     |
  |     +-- super.notify()
  |           --> SENDS WHATSAPP
  |
  +-- super.notify()
        --> SENDS SMS
```

**Result order:** Email --> WhatsApp --> SMS

---

### Class Diagram

```
          <<interface>>
            Notifier
           + notify()
          /    |     \
         /     |      \
EmailNotifier  SMSNotifier  WhatsAppNotifier
                |                |
           SMSWrapper      WhatsAppWrapper
           - wrappedNotifier: Notifier
           + notify()
```

**Key structural rules:**
- Every wrapper **implements** the common `Notifier` interface (so clients are agnostic).
- Every wrapper **extends** its corresponding concrete class (to reuse notification-sending logic).
- Every wrapper **holds a `Notifier` reference** (to delegate to the wrapped object).

---

### Design Decisions -- Why Decorator?

| Question | Answer |
|----------|--------|
| Why not just add boolean flags for each channel? | Booleans lead to if-else chains. Adding a new channel means modifying existing code (violates OCP). |
| Why not use a list of notifier types? | Could work in some cases, but Decorator gives you **behavioral composition** -- each wrapper can modify/extend behavior, not just toggle it. |
| When should you use Decorator? | When you need to **add features at runtime** to an existing object, and those features involve **behavior changes**, not just attribute changes. |

---

### Teacher's Critical Warning About Decorator

> "Decorator is NOT a pattern you will use at a lot of places. You have to be very careful about where exactly you are using the decorator design pattern."

The teacher explicitly says:
- Many times the job can be done with **dynamic attributes** (a list) or **simple boolean flags**.
- Evaluate the need first. Only use Decorator when simpler approaches fail.
- Most of the time, you can write maintainable, extensible code without Decorator.

**Contrast with Proxy:** The teacher mentioned Proxy is used "very, very frequently in multiple use cases," but Decorator is more niche.

---

### Real-World Examples of Decorator (from the teacher)

1. **Gaming -- Character Powers:**
   - A character starts with basic abilities.
   - After earning points, the character unlocks new powers (longer jump, fire ability, etc.).
   - Same object, new features at runtime --> pass the object through a new wrapper constructor.

2. **Gaming -- Stone Game:**
   - A stone object can become a "fiery stone" or start "spitting smaller stones."
   - These are **behavioral changes**, not just new attributes.

3. **Notifications (teaching example only):**
   - The teacher explicitly warns: "In reality, notifications are triggered in an async way. They are pushed to a queue like Kafka. This is just an example."

---

## 2. Adapter Design Pattern -- Real-World Merger Problem

### The Snapdeal Story (Teacher's Personal Experience)

This is a real story from the teacher's time working at Snapdeal.

#### Context

- **Team:** Seller Search -- provides search functionality for sellers.
- **API examples:** `getSellersBySUPC()`, `getSellersBySKU()` -- returns list of sellers for a product.
- **Consumer:** Seller Ranking team -- determines which seller appears first when a buyer clicks a product.

```java
// Seller Ranking code (the client)
class SellerRanking {
    private ISellerSearchService sellerSearchService; // depends on interface

    public SellerRanking(ISellerSearchService service) {
        this.sellerSearchService = service;
    }

    public List<Seller> getSellerList(String sku) {
        return sellerSearchService.getSellersBySKU(sku);
    }
}
```

#### The Problem -- Company Merger

**Exclusively.com** (another e-commerce company) was acquired by Snapdeal. They had:
- Completely different codebase and database.
- Different class names: `Merchant` instead of `Seller`, `Customer` instead of `User`.
- Different method names: `getMerchantBySKU()` instead of `getSellersBySKU()`.

The Seller Ranking feature needed to work for Exclusively's data too.

#### Failed Approaches

| Approach | Why It Fails |
|----------|-------------|
| **Rename methods** in Exclusively's service to match Snapdeal's names | Those methods are used by other microservices in Exclusively's ecosystem. Renaming would break them. |
| **Add wrapper methods** inside Exclusively's class (e.g., add `getSellersBySKU()` that calls `getMerchantBySKU()`) | Violates OCP -- modifying a tested, working class. Also, this class doesn't implement Snapdeal's `ISellerSearchService` interface, so it still can't be injected. |
| **Make Exclusively's class implement Snapdeal's interface** | This is legacy code. People who wrote it have left. "Do not touch it." |

#### Teacher's Rule About Legacy Code

> "If this is legacy code -- people who wrote it have left, there is no one to maintain it -- ideally we should not be touching it. Just test whether it works. If it works, let it be. Do not make any changes."

---

### The Solution -- Adapter Pattern

Create a **new wrapper class** that:
1. **Implements** Snapdeal's `ISellerSearchService` interface (so it can be injected into Seller Ranking).
2. **Holds an object** of Exclusively's `ExclusivelySellerSearchService`.
3. **Delegates** each method call to the corresponding method in Exclusively's service.

```java
// The Adapter
public class ExclusivelySellerSearchAdapter implements ISellerSearchService {

    private ExclusivelySellerSearchService exclusivelyService;

    public ExclusivelySellerSearchAdapter(ExclusivelySellerSearchService service) {
        this.exclusivelyService = service;
    }

    @Override
    public List<Seller> getSellersBySKU(String sku) {
        return exclusivelyService.getMerchantBySKU(sku);  // just delegate!
    }

    @Override
    public List<Seller> getSellersBySUPC(String supc) {
        return exclusivelyService.getMerchantBySUPC(supc);  // just delegate!
    }
}
```

Now the Seller Ranking code works unchanged:

```java
// For Snapdeal sellers
SellerRanking ranking1 = new SellerRanking(new SnapdealSellerSearchService());

// For Exclusively sellers -- just plug in the adapter
SellerRanking ranking2 = new SellerRanking(new ExclusivelySellerSearchAdapter(exclusivelyService));
```

---

### Adapter Pattern -- Definition

> An Adapter creates a wrapper around an incompatible class so that its object **fits where a different type of object is expected**. It implements the target interface and internally delegates to the adaptee's methods.

**Key characteristics:**
- The adapter does **no actual work** -- it only delegates.
- It translates method names/signatures from one interface to another.
- The original classes remain completely untouched.

---

### Teacher's Insight on Company Mergers

> "This is something which keeps happening whenever you have mergers of companies. We do not really merge the entire code base. We create wrappers which translate from one type of classes to the other."

---

### Adapter vs Decorator -- Comparison

| Aspect | Decorator | Adapter |
|--------|-----------|---------|
| **Purpose** | Add new behavior/features at runtime | Make incompatible interfaces compatible |
| **Modifies behavior?** | Yes -- adds functionality | No -- only translates/delegates |
| **Implements same interface as wrapped object?** | Yes | Implements a **different** interface than the wrapped object |
| **Extends concrete class?** | Yes (to reuse behavior) | No (just delegates) |
| **Chaining** | Multiple decorators can be chained | Typically a single adapter |
| **When to use** | Runtime feature addition | Code integration / legacy compatibility |

---

## 3. SOLID Principles Assignment Walkthrough

### Question 3 -- Eligibility Engine (OCP Violation Fix)

#### Original Problem
The eligibility engine had hard-coded if-else blocks checking conditions like:
- Does the child have disciplinary actions?
- Does the child meet attendance requirements?
- Does the child meet CGR (CGPA) requirements?

**Violation:** Adding a new eligibility rule requires modifying the existing class (OCP violation).

#### Solution Design

1. Create an `EligibilityRule` interface with a method `boolean check(Child child)`.
2. Each rule becomes its own class implementing the interface:
   - `AttendanceRule implements EligibilityRule`
   - `CGRRule implements EligibilityRule`
   - `DisciplinaryFlagRule implements EligibilityRule`
3. The `EligibilityEngine` takes a `List<EligibilityRule>` via its constructor.
4. To check eligibility, iterate over the list and call `check()` on each rule.

```java
interface EligibilityRule {
    boolean check(Child child);
}

class AttendanceRule implements EligibilityRule {
    public boolean check(Child child) {
        return child.getAttendance() >= MINIMUM_ATTENDANCE;
    }
}

class EligibilityEngine {
    private List<EligibilityRule> rules;

    public EligibilityEngine(List<EligibilityRule> rules) {
        this.rules = rules;
    }

    public boolean isEligible(Child child) {
        for (EligibilityRule rule : rules) {
            if (!rule.check(child)) {
                return false;
            }
        }
        return true;
    }
}
```

**Benefit:** Adding a new rule just means creating a new class and adding it to the list -- no changes to existing code.

---

### Question 4 -- Hostel Fee Calculator (OCP Violation Fix)

#### Original Problem
Room types (Single, Double, Triple, Deluxe) and add-ons (Gym, Mess, Laundry) were hard-coded with if-else/switch blocks in the fee calculator.

#### Solution Design

1. **Room interface:** `RoomType` with methods `boolean isSelected()` and `double getPrice()`.
2. **Add-on interface:** `AddOn` with methods `boolean isSelected()` and `double getPrice()`.
3. Each room type and add-on is a separate class implementing the respective interface.
4. The `HostelFeeCalculator` takes `List<RoomType>` and `List<AddOn>`, iterates over them.

```java
interface RoomType {
    boolean isSelected();
    double getPrice();
}

interface AddOn {
    boolean isSelected();
    double getPrice();
}

class HostelFeeCalculator {
    private List<RoomType> rooms;
    private List<AddOn> addOns;

    public double calculateFee() {
        double total = 0;
        for (RoomType room : rooms) {
            if (room.isSelected()) total += room.getPrice();
        }
        for (AddOn addOn : addOns) {
            if (addOn.isSelected()) total += addOn.getPrice();
        }
        return total;
    }
}
```

**Benefit:** Adding a "5-person room" or a "Swimming Pool" add-on just means creating a new class -- zero changes to `HostelFeeCalculator`.

---

## 4. Teacher's Special Insights and Career Tips

### Coding Best Practices (Teacher's Rules)

1. **Avoid boolean variables as much as possible:**
   > "Boolean variables are going to result into if-else. If you have a boolean which holds true or false, this is going to be used in some if-else in your code. So booleans should be avoided or used very wisely."

2. **Avoid `break` and `continue` statements:**
   > "Break and continue make your debugging process harder. You have to test extra conditions. Design your loop so that the loop condition itself is sufficient. This applies even to your DSA programs."

3. **Legacy code policy:**
   > "If it is legacy code, people who wrote it have left, there is no one to maintain -- do not touch it. Just test whether it works. If it works, let it be."

4. **Notifications are async in production:**
   > "In reality, notifications are never triggered inside the code like this. They will be pushed to a queue like Kafka, and consumed asynchronously."

5. **Evaluate before using Decorator:**
   > "Most of the times you will be able to see that even if you don't do it, you can write maintainable, extensible and understandable code very easily."

---

## 5. Interview Questions

### Conceptual Questions

1. **What is the Decorator Design Pattern? When would you use it?**
   - A structural pattern that allows adding new behaviors to objects at runtime by wrapping them in decorator objects. Use when you need runtime feature composition without modifying existing classes.

2. **What is the Adapter Design Pattern? Give a real-world example.**
   - A structural pattern that allows incompatible interfaces to work together by creating a wrapper that translates one interface to another. Real example: company mergers where codebases use different class/method names for the same functionality.

3. **What is the difference between Decorator and Adapter?**
   - Decorator adds new behavior; Adapter only translates interfaces. Decorator implements the same interface as the wrapped object; Adapter implements a different interface.

4. **In Decorator, why does the wrapper class both implement the interface AND extend the concrete class?**
   - Implements the interface so the client sees no difference (polymorphism). Extends the concrete class to reuse the actual notification-sending logic without creating an internal object.

5. **Why should you avoid boolean variables in design?**
   - They inevitably lead to if-else statements, making code harder to extend (OCP violation).

6. **Why should you avoid `break` and `continue`?**
   - They make debugging harder -- extra conditions to test, harder to reason about loop behavior.

7. **How do you handle legacy code during a company merger?**
   - Do not modify it. Create adapter/wrapper classes that translate between the two codebases.

### Coding/Design Questions

8. **Design a notification system where channels can be added at runtime.**
   - Use Decorator pattern: base `Notifier` interface, concrete notifiers, wrapper classes that chain behavior.

9. **You have two services with identical functionality but different method names. How do you integrate them?**
   - Use Adapter pattern: create a wrapper implementing the target interface, delegating to the adaptee.

10. **Refactor an eligibility engine that uses if-else blocks for each rule.**
    - Extract rules into an interface, create concrete rule classes, pass a list to the engine (Strategy + OCP).

---

## 6. Exam Prep -- Quick-Fire Revision

### Key Definitions

| Term | Definition |
|------|-----------|
| **Decorator Pattern** | Structural pattern to add features to objects at runtime by wrapping them in decorator classes |
| **Adapter Pattern** | Structural pattern to make incompatible interfaces work together via a translation wrapper |
| **DRY** | Don't Repeat Yourself -- avoid duplicating code |
| **OCP** | Open/Closed Principle -- open for extension, closed for modification |
| **DIP** | Dependency Inversion Principle -- depend on abstractions, not concretions |
| **Legacy Code** | Existing code where original authors are gone; avoid modifying, create wrappers instead |

### Decorator Pattern Checklist

- [ ] Wrapper **implements** the same interface as the object being wrapped
- [ ] Wrapper **extends** the concrete class whose behavior it adds
- [ ] Wrapper **holds a reference** to a `Notifier` (the wrapped object)
- [ ] Constructor **accepts** the interface type (not a concrete type)
- [ ] `notify()` first calls `wrappedObject.notify()`, then `super.notify()`
- [ ] Decorators can be **chained**: pass one wrapper into another's constructor

### Adapter Pattern Checklist

- [ ] Adapter **implements** the target interface (the one the client expects)
- [ ] Adapter **holds an object** of the adaptee class (the incompatible class)
- [ ] Each method **delegates** to the corresponding adaptee method
- [ ] Adapter does **no actual work** -- only translates

### Things to Avoid in Code Design

| Bad Practice | Why | Better Alternative |
|-------------|-----|-------------------|
| Lots of boolean flags | Lead to if-else chains, violate OCP | Use polymorphism (interfaces + list of implementations) |
| `break` / `continue` in loops | Hard to debug, extra test conditions | Design loop conditions to be self-sufficient |
| Modifying legacy code | Risky, no maintainers, could break other systems | Create adapters/wrappers |
| Hard-coded if-else for types | Violates OCP | Strategy pattern / interface + list |
| Duplicating code in wrappers | Violates DRY | Use inheritance (`extends`) to reuse behavior |

### Pattern Usage Frequency (Teacher's Guidance)

| Pattern | Frequency of Use |
|---------|-----------------|
| Proxy | Very frequently, many use cases |
| Decorator | Rare, evaluate carefully before using |
| Adapter | Common during integrations/mergers |
| Factory | (Coming in next class) |

### Upcoming Assignments

- Proxy and Factory pattern problems to be uploaded (deadline: Monday afternoon).
- Solutions for SOLID principles exercises to be released this week.
