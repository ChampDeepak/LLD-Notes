# LLD Lecture 10 Notes -- Decorator Design Pattern (via Notification Service & Beverage Cost Problem)

---

## Table of Contents

1. [Quick Recap: Proxy Design Pattern](#1-quick-recap-proxy-design-pattern)
2. [Problem Statement: Notification Service](#2-problem-statement-notification-service)
3. [Teacher's Design Journey (V0 to Final)](#3-teachers-design-journey-v0-to-final)
4. [The Beverage/Condiment Analogy](#4-the-beveragecondiment-analogy)
5. [Dynamic Attributes vs Dynamic Features](#5-dynamic-attributes-vs-dynamic-features)
6. [Decorator Design Pattern -- Final Solution](#6-decorator-design-pattern----final-solution)
7. [Code Examples](#7-code-examples)
8. [Interview Questions](#8-interview-questions)
9. [Design Decisions -- Why One Approach Over Another](#9-design-decisions----why-one-approach-over-another)
10. [Important Points & Definitions](#10-important-points--definitions)
11. [Exam Prep -- Quick-Fire Facts](#11-exam-prep----quick-fire-facts)
12. [Teacher's Special Insights](#12-teachers-special-insights)

---

## 1. Quick Recap: Proxy Design Pattern

The lecture begins with a brief wrap-up of the previous class on the **Proxy Design Pattern**. Key reminder points:

| Use Case | How Proxy Helps |
|---|---|
| Access control | Gate access to a resource via a proxy |
| Remote resource management | Proxy manages the connection to a remote object |
| Lazy loading | Proxy defers creation of expensive object until actually needed |
| Retry logic | Proxy can transparently retry failed operations |
| Exception handling | Proxy wraps calls and handles exceptions centrally |

> **Teacher's note:** "This is one of the most commonly used design patterns which you will be using across different type of problems."

---

## 2. Problem Statement: Notification Service

### Initial Requirement (Single Client, Single Notification Type)

- A **client** needs a **notification service**.
- The client only needs **email notifications**.
- The service must expose a single method: `notify()`.

### Evolving Requirements

| Stage | Requirement |
|---|---|
| Stage 1 | One client, email only |
| Stage 2 | A second notification type -- WhatsApp |
| Stage 3 | SMS notification added |
| Stage 4 | A client wants **both** email AND WhatsApp from a single `notify()` call |
| Stage 5 | Clients may want **any arbitrary combination** (email + SMS, WhatsApp + SMS + Slack, etc.) |

---

## 3. Teacher's Design Journey (V0 to Final)

This is the core of the lecture. The teacher walks through increasingly better designs, showing why each version fails before arriving at the Decorator pattern.

---

### V0 -- Naive Direct Implementation (Rejected Immediately)

**Approach:** Build an `EmailNotifier` class directly and give it to the client.

**Problem:** The client becomes tightly coupled to the `EmailNotifier` concrete class. This violates **Dependency Inversion Principle** -- the client should depend on abstractions, not concretions.

> **Teacher's words:** "We will not directly be building the email notifier because we want the client to have dependency inversion. We want the client to be agnostic of all the concrete classes."

---

### V1 -- Interface + Concrete Classes (Good, but Limited)

**Approach:**

```
Notifier (interface)
  |-- notify()
  |
  |--- EmailNotifier (implements Notifier)
  |--- WhatsAppNotifier (implements Notifier)
  |--- SMSNotifier (implements Notifier)
```

- A **Factory** creates the correct notifier object and gives it to the client.
- The client code is always the same:

```java
Notifier notifier = factory.create(...);
notifier.notify();
```

**What works:**
- Adding a new notification type (e.g., SMS) = just add a new class. No existing code changes. **Open/Closed Principle** satisfied.
- Each class has one job. **Single Responsibility Principle** satisfied.
- Client depends only on the `Notifier` interface. **Dependency Inversion** satisfied.

**What breaks:**
- A client wants `notifier.notify()` to send **both** email AND WhatsApp. The factory method returns ONE `Notifier` object -- it cannot return two.

---

### V2 -- Asking the Client to Manage Multiple Notifiers (Rejected)

**Approach:** Have the factory return a list of notifiers, or ask the client to create multiple notifier objects and loop over them.

```java
// Option A: Factory returns a list
List<Notifier> notifiers = factory.createMultiple(...);
for (Notifier n : notifiers) {
    n.notify();
}

// Option B: Client creates multiple
Notifier email = factory.create("email");
Notifier whatsapp = factory.create("whatsapp");
email.notify();
whatsapp.notify();
```

**Why the teacher rejects this:**
- The client has to do **extra work** (looping, managing multiple objects).
- The client's requirement is simple: "When I call `notifier.notify()`, both notifications should fire. How you implement it is not my problem."
- We are pushing implementation complexity onto the client, which is bad design.

> **Teacher's words:** "We are asking the client to do more work. The client has a simple requirement -- when I do notify, both should get triggered. How is it implemented on your side? That's not my problem."

---

### V3 -- Combination Wrapper Classes (Works, but Class Explosion)

**Approach:** Create a new class for each combination.

```
EmailWhatsAppNotifier implements Notifier {
    EmailNotifier email;
    WhatsAppNotifier whatsapp;

    notify() {
        email.notify();
        whatsapp.notify();
    }
}
```

Similarly: `SMSWhatsAppNotifier`, `EmailSMSNotifier`, `EmailWhatsAppSMSNotifier`, `EmailSlackNotifier`, etc.

**What works:**
- Client code stays the same: `notifier.notify()`.
- Each wrapper implements `Notifier`, so the client sees no difference.
- Reuses existing notifier logic (no reimplementation).

**What breaks -- CLASS EXPLOSION:**

| Number of notification types | Number of combination classes needed |
|---|---|
| 2 | 1 (AB) |
| 3 | 4 (AB, AC, BC, ABC) |
| 4 | 11 (all 2/3/4 combos) |
| N | 2^N - N - 1 |

> **Teacher's words:** "There is going to be a class explosion. We are going to have all type of combinations coming up as classes."

**Also:** Adding a new notifier type (e.g., Slack) means creating new combination classes with every existing type. This **violates Open/Closed Principle**.

---

### V3.5 -- Single Wrapper with a List of Notifiers (Student Suggestion, Partially Rejected)

A student suggests: Create ONE wrapper class that holds a `List<Notifier>` and loops over them.

```java
class CompositeNotifier implements Notifier {
    List<Notifier> notifiers;

    notify() {
        for (Notifier n : notifiers) {
            n.notify();
        }
    }
}
```

**Teacher's analysis:**
- This works for simple cases (just calling `notify()` on each).
- But if we add a new notifier type, we may need to modify the composite class logic -- **violates Open/Closed Principle**.
- More importantly, this only solves **dynamic attributes** (adding more notifiers to a list). It does NOT solve **dynamic features** (adding new behavior like pre-processing, headers, signatures to an existing object at runtime).

> **Teacher's words:** "If we add a new type of notifier, then we'll have to make changes in that code as well."

**This is actually the Composite pattern (useful, but not the focus of this lecture).**

---

### V4 -- Wrapper That Adds Behavior (The Decorator Pattern -- Final Solution)

**Key insight from the teacher:** We do not just want to combine notifiers. We want to **add new behavior to an existing object at runtime** without modifying its class.

**Approach:** Create wrapper classes where:
1. Each wrapper **implements the same `Notifier` interface**.
2. Each wrapper **holds a reference to another `Notifier` object** (the "wrappee").
3. In `notify()`, it first calls `wrappee.notify()`, then does its own work.

```
WhatsAppWrapper implements Notifier {
    Notifier wrappedNotifier;  // <-- abstract type, not concrete!

    WhatsAppWrapper(Notifier wrappedNotifier) {
        this.wrappedNotifier = wrappedNotifier;
    }

    notify() {
        wrappedNotifier.notify();   // delegate to wrapped object
        // then send WhatsApp message
        sendWhatsApp();
    }
}
```

**How to build any combination:**

```java
// Email only
Notifier n = new EmailNotifier();

// Email + WhatsApp
Notifier n = new WhatsAppWrapper(new EmailNotifier());

// Email + WhatsApp + SMS
Notifier n = new SMSWrapper(new WhatsAppWrapper(new EmailNotifier()));

// WhatsApp + SMS (no email)
Notifier n = new SMSWrapper(new WhatsAppNotifier());
```

**Why this is the final solution:**

| Concern | How Decorator Solves It |
|---|---|
| Client simplicity | Client always calls `notifier.notify()` -- no loops, no lists |
| No class explosion | One wrapper per notification type, not per combination |
| Open/Closed | Adding Slack = add ONE `SlackWrapper` class. Zero changes elsewhere |
| Single Responsibility | Each wrapper does exactly one extra thing |
| Dependency Inversion | Everything depends on the `Notifier` interface |
| Runtime flexibility | Objects can be wrapped and re-wrapped at runtime |

---

## 4. The Beverage/Condiment Analogy

The teacher uses a coffee shop example to make the concept more tangible.

### Setup

- **Beverage** (abstract class/interface) with a `cost()` method.
- Concrete beverages: `Espresso` (cost=15), `Latte` (cost=20), `Cappuccino` (cost=25).
- Condiments: `Sugar` (+2), `Cream` (+5), `ExtraEspressoShot` (+10).

### The Problem

A customer can add **any combination** of condiments to any beverage, and the cost must reflect all additions.

### Why Attributes Don't Work

| Approach | Problem |
|---|---|
| Add boolean/int attributes (`hasSugar`, `creamCount`) to Beverage | These attributes don't make sense for all beverages (e.g., `BananaShake` doesn't need `espressoShot`). Violates good design. |
| Subclass every combination (`LatteWithSugarAndCream`) | Class explosion -- same problem as V3 above |

### How Decorator Solves It

Each condiment is a decorator:

```java
Beverage order = new Espresso();                      // cost = 15
order = new SugarDecorator(order);                     // cost = 15 + 2 = 17
order = new CreamDecorator(order);                     // cost = 17 + 5 = 22
order = new ExtraEspressoShotDecorator(order);         // cost = 22 + 10 = 32
```

Each decorator wraps the previous object, calls its `cost()`, and adds its own cost on top.

---

## 5. Dynamic Attributes vs Dynamic Features

This is a crucial conceptual distinction the teacher draws.

### Dynamic Attributes (What a List Solves)

- Keep a `List<Condiment>` inside the beverage.
- At runtime, add/remove items from the list.
- All items implement a common interface (e.g., `getCost()`).
- Different objects of the same class can have different numbers/types of attributes.

> **Teacher's key point:** "Instead of keeping all attributes individually inside the class, we keep a list. Anything you want to keep can implement an attribute interface."

### Dynamic Features (What Decorator Solves)

- Not just adding data, but **changing behavior** of methods at runtime.
- Example: Before sending a WhatsApp notification, add a signature. Before sending email, add headers and subject. These are **behavioral changes**, not just attribute additions.
- You cannot add new methods to a class at runtime in Java. But you CAN **change what an existing method does** by wrapping the object.

> **Teacher's words:** "What if we are not just interested in adding attributes, we are also interested in adding new features?... Can we dynamically change the behavior of a method?"

**Answer:** Yes -- with the Decorator pattern. You wrap the object in a new decorator, and the decorated `notify()` method now does something different (extra processing, extra calls, etc.).

---

## 6. Decorator Design Pattern -- Final Solution

### UML Structure

```
<<interface>> Notifier
  + notify()
      ^
      |
      |--- EmailNotifier (implements Notifier)
      |--- WhatsAppNotifier (implements Notifier)
      |--- SMSNotifier (implements Notifier)
      |
      |--- NotifierDecorator (implements Notifier, HAS-A Notifier)
              ^
              |--- WhatsAppDecorator
              |--- SMSDecorator
              |--- EmailDecorator
              |--- SlackDecorator
```

### Key Structural Rules

1. **Decorator implements the same interface** as the object it wraps.
2. **Decorator HAS-A reference** to the interface type (not a concrete type).
3. **Decorator delegates** to the wrapped object first, then adds its own behavior.
4. **Decorators can be stacked** (a decorator wrapping a decorator wrapping a concrete object).

### How the Client Uses It

```java
// Factory builds whatever combination is needed:
Notifier notifier = new SMSDecorator(
                        new WhatsAppDecorator(
                            new EmailNotifier()
                        )
                    );

// Client code -- unchanged regardless of how many decorators are stacked:
notifier.notify();
// Result: Email sent -> WhatsApp sent -> SMS sent
```

---

## 7. Code Examples

### Notifier Interface

```java
public interface Notifier {
    void notify(String message);
}
```

### Concrete Notifiers

```java
public class EmailNotifier implements Notifier {
    @Override
    public void notify(String message) {
        System.out.println("Sending EMAIL: " + message);
    }
}

public class WhatsAppNotifier implements Notifier {
    @Override
    public void notify(String message) {
        System.out.println("Sending WhatsApp: " + message);
    }
}

public class SMSNotifier implements Notifier {
    @Override
    public void notify(String message) {
        System.out.println("Sending SMS: " + message);
    }
}
```

### Base Decorator (Abstract)

```java
public abstract class NotifierDecorator implements Notifier {
    protected Notifier wrappedNotifier;

    public NotifierDecorator(Notifier wrappedNotifier) {
        this.wrappedNotifier = wrappedNotifier;
    }

    @Override
    public void notify(String message) {
        wrappedNotifier.notify(message);  // delegate to wrapped object
    }
}
```

### Concrete Decorators

```java
public class WhatsAppDecorator extends NotifierDecorator {
    public WhatsAppDecorator(Notifier wrappedNotifier) {
        super(wrappedNotifier);
    }

    @Override
    public void notify(String message) {
        super.notify(message);                          // call wrapped notifier
        System.out.println("Sending WhatsApp: " + message);  // add own behavior
    }
}

public class SMSDecorator extends NotifierDecorator {
    public SMSDecorator(Notifier wrappedNotifier) {
        super(wrappedNotifier);
    }

    @Override
    public void notify(String message) {
        super.notify(message);
        System.out.println("Sending SMS: " + message);
    }
}
```

### Client Code

```java
public class Client {
    public static void main(String[] args) {
        // Client 1: Email only
        Notifier client1Notifier = new EmailNotifier();
        client1Notifier.notify("Hello from Client 1");

        // Client 2: Email + WhatsApp
        Notifier client2Notifier = new WhatsAppDecorator(new EmailNotifier());
        client2Notifier.notify("Hello from Client 2");

        // Client 3: Email + WhatsApp + SMS
        Notifier client3Notifier = new SMSDecorator(
                                        new WhatsAppDecorator(
                                            new EmailNotifier()
                                        )
                                   );
        client3Notifier.notify("Hello from Client 3");
    }
}
```

**Output for Client 3:**
```
Sending EMAIL: Hello from Client 3
Sending WhatsApp: Hello from Client 3
Sending SMS: Hello from Client 3
```

### Beverage Example (Cost Decorator)

```java
public interface Beverage {
    double cost();
    String description();
}

public class Espresso implements Beverage {
    public double cost() { return 15.0; }
    public String description() { return "Espresso"; }
}

public abstract class CondimentDecorator implements Beverage {
    protected Beverage wrappedBeverage;
    public CondimentDecorator(Beverage beverage) {
        this.wrappedBeverage = beverage;
    }
}

public class SugarDecorator extends CondimentDecorator {
    public SugarDecorator(Beverage beverage) { super(beverage); }
    public double cost() { return wrappedBeverage.cost() + 2.0; }
    public String description() { return wrappedBeverage.description() + ", Sugar"; }
}

public class CreamDecorator extends CondimentDecorator {
    public CreamDecorator(Beverage beverage) { super(beverage); }
    public double cost() { return wrappedBeverage.cost() + 5.0; }
    public String description() { return wrappedBeverage.description() + ", Cream"; }
}

// Usage:
Beverage order = new CreamDecorator(new SugarDecorator(new Espresso()));
System.out.println(order.description());  // Espresso, Sugar, Cream
System.out.println(order.cost());          // 22.0
```

---

## 8. Interview Questions

### Conceptual Questions

1. **What is the Decorator Design Pattern?**
   A structural design pattern that allows adding new behavior to an existing object at runtime by wrapping it inside a decorator object that implements the same interface.

2. **How is Decorator different from Proxy?**
   - Proxy controls **access** to an object (same behavior, controlled access).
   - Decorator **adds new behavior** to an object (enhanced behavior, same access).
   - Both use wrapping/composition with the same interface, but the **intent** differs.

3. **How is Decorator different from Composite?**
   - Composite treats a group of objects as one (tree structure, uniform treatment).
   - Decorator adds behavior to a single object by stacking wrappers.

4. **Why not use inheritance to add features?**
   - Inheritance is static (compile-time). You cannot change what a subclass does at runtime.
   - Leads to class explosion when combinations are needed.
   - Decorator gives runtime flexibility.

5. **Can decorators be stacked? In what order?**
   Yes. Order matters -- each decorator wraps the previous one. The outermost decorator's method runs last (or first, depending on where you place `super.notify()`).

6. **Which SOLID principles does the Decorator pattern uphold?**
   - **Open/Closed:** New behavior via new decorator class, no modification of existing code.
   - **Single Responsibility:** Each decorator handles one concern.
   - **Dependency Inversion:** Everything depends on the interface.
   - **Interface Segregation:** Decorator interface is the same small interface as the component.

7. **What is the difference between dynamic attributes and dynamic features?**
   - Dynamic attributes: using a list to hold varying numbers of data objects at runtime.
   - Dynamic features: using decorators to change/extend method behavior at runtime.

### Design/Coding Questions

8. **Design a notification system where clients can receive notifications via any combination of email, SMS, WhatsApp, and Slack.** (Full solution above.)

9. **Design a coffee cost calculator where any combination of condiments can be added to a base beverage.** (Full solution above.)

10. **Given a `FileReader` class, add buffering, encryption, and compression features using the Decorator pattern.**

---

## 9. Design Decisions -- Why One Approach Over Another

| Decision Point | Option Chosen | Option Rejected | Reason |
|---|---|---|---|
| Client depends on concrete or abstract? | Abstract (interface) | Concrete class | Dependency Inversion Principle; client should not know implementation details |
| Multiple notifications: list in client? | No | Yes | Pushes complexity to client; client wants single `notify()` call |
| Multiple notifications: one class per combo? | No | Yes | Class explosion (2^N combinations); violates Open/Closed when new types added |
| Multiple notifications: composite with list? | Partially | -- | Works for simple delegation but does not support adding *behavior* (only attributes); modifying the composite when new types arrive violates Open/Closed |
| Multiple notifications: decorator wrappers? | **Yes (Final)** | -- | One wrapper per type, stack for any combination, runtime flexibility, all SOLID principles satisfied |
| Condiments as attributes on Beverage? | No | Yes | Not all condiments apply to all beverages; wastes space; not extensible |
| Add extra methods to specific notifiers? | No | Yes | Violates Interface Segregation (method not relevant to all); requires type-checking (violates Dependency Inversion); changes behavior for all clients (not just the one who wants it) |

---

## 10. Important Points & Definitions

- **Decorator Pattern:** A structural design pattern that wraps an object inside another object of the same interface to add behavior at runtime.
- **Class Explosion:** When the number of required classes grows combinatorially (exponentially) with the number of features/types. This is a design smell.
- **Dynamic Attributes:** Adding varying numbers of data members to an object at runtime via a list of interface-typed objects.
- **Dynamic Features (Behavior):** Changing what a method does at runtime -- achieved by wrapping the object in a decorator.
- **The wrapper MUST implement the same interface** as the wrapped object. This is non-negotiable -- it ensures the client sees no difference.
- **The wrapper's reference to the wrapped object MUST be of the abstract type** (interface), not a concrete type. This is what enables stacking and flexibility.
- **Factory + Decorator work together:** The factory can construct the decorated chain and return it to the client as a single `Notifier` object.

---

## 11. Exam Prep -- Quick-Fire Facts

| Fact | Detail |
|---|---|
| Pattern Name | Decorator (also called Wrapper) |
| Pattern Category | Structural |
| Key Relationship | HAS-A (composition) + IS-A (same interface) |
| Core Idea | Wrap object in another object of same type to add behavior |
| Number of classes for N notification types (with Decorator) | N concrete + N decorators = 2N |
| Number of classes for N types WITHOUT Decorator (all combos) | 2^N - 1 |
| Proxy vs Decorator | Same structure, different intent (access control vs behavior addition) |
| Runtime vs Compile-time | Decorator = runtime. Inheritance = compile-time. |
| Real-world Java example | `BufferedReader(new FileReader(...))` -- BufferedReader decorates FileReader |
| Key interface rule | Decorator implements the same interface as the object it decorates |
| Teacher's summary of previous class technique | "Wrapping the objects of concrete classes inside another class and making this class implement the same interface so that for the client nothing changes" |

### SOLID Principles Cheat Sheet for This Lecture

| Principle | How It Applies |
|---|---|
| **S** - Single Responsibility | Each notifier/decorator handles exactly one type of notification |
| **O** - Open/Closed | New notification type = new class, zero changes to existing code |
| **L** - Liskov Substitution | Any decorator can be used wherever `Notifier` is expected |
| **I** - Interface Segregation | One clean `notify()` method; no bloated interfaces |
| **D** - Dependency Inversion | Client and decorators depend on `Notifier` interface, not concrete classes |

---

## 12. Teacher's Special Insights

### Thumb Rules

1. **"Even if there is only one single concrete class, the dependency should always be on abstraction."**
   - Do not skip creating an interface just because you have only one implementation today. Tomorrow you may have more.

2. **"There has to be a factory which is responsible for creating objects."**
   - Never let clients directly instantiate concrete classes.

3. **"We should not ask the client to do more work."**
   - If the client's requirement is simple (`notifier.notify()`), the solution must keep it simple for the client, regardless of internal complexity.

### Design Thinking Process (How the Teacher Approaches Problems)

1. **Start with the simplest case** -- one client, one type.
2. **Apply SOLID immediately** -- even for the simple case, use interfaces and factories.
3. **Incrementally add complexity** -- what if there's a second type? A third?
4. **Evaluate each solution against SOLID** -- if it violates a principle, reject it.
5. **Look for patterns** -- "this is very similar to what we did in the previous class" (Proxy wrapping). Recognize structural similarities across patterns.
6. **Distinguish between attribute addition and behavior addition** -- this guides whether you need a list (composite) or a decorator.

### Real-World Analogies

- **Coffee shop:** Base beverage + condiments stacked on top = base object + decorators stacked on top. Each condiment adds cost, just as each decorator adds behavior.
- **Notification system:** One `notify()` call triggers a chain of wrapped notifiers, each adding its own notification channel.

### Career Tips (Implied)

- The Proxy and Decorator patterns look structurally identical (wrapper + same interface). In interviews, **always clarify the intent**: Proxy = access/control, Decorator = behavior addition.
- When asked to "add features to an existing class without modifying it," the answer is almost always Decorator.
- Recognize class explosion as a smell early -- if you find yourself creating classes for every combination, step back and think Decorator.

---

## Summary: The Complete Design Evolution

```
V0: Direct concrete class          --> Violates Dependency Inversion
V1: Interface + concrete classes   --> Works for single notification per client
V2: Client manages multiple objects --> Pushes complexity to client
V3: One class per combination      --> Class explosion (2^N classes)
V3.5: Single class with list       --> Cannot add behavior, only attributes
V4: DECORATOR PATTERN              --> One wrapper per type, stackable,
                                       runtime flexible, all SOLID satisfied
```

This is the design journey the teacher wants you to internalize. In an interview, walk through these versions to show your thought process, and then arrive at the Decorator as the clean solution.
