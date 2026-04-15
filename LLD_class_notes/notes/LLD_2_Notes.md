# LLD Lecture 2 — Detailed Notes
## Topic: Object-Oriented Design of a Bird | Abstraction, Inheritance, Interfaces & SOLID Principles (First 3)

---

## 1. The Problem Statement

> Design an object-oriented system for a **Bird**. There can be different types of birds (hen, eagle, crow, kiwi, parrot, etc.) and each bird can have different behaviors (fly, walk, eat, etc.). Different birds exhibit **different** flying behaviors.

---

## 2. The Design Journey (Step by Step)

### V0 — The Naive Single Class Design

```
class Bird {
    // Attributes (properties)
    color, type, name, weight, height, wingSpread

    // Behaviors (methods)
    fly() { print("bird is flying") }
    walk() { ... }
    eat() { ... }
}
```

- Create objects using a parameterized constructor:
  ```
  Bird hen = new Bird("hen", ...)
  Bird eagle = new Bird("eagle", ...)
  ```
- **Attributes** are different for each bird (weight, color, etc.)
- **But behavior is identical** — `hen.fly()` and `eagle.fly()` do the exact same thing

**Teacher's Key Point:** When designing real-life entities, focus on **design, not implementation**. You don't know where this bird will be used (graphical game? hardware robot?). Just define the structure. However, for real-world problem-solving questions, your code must be fully functional.

---

### V0.5 — Adding if-else to differentiate behavior

```java
fly() {
    if (type == "hen") {
        // fly low
    } else if (type == "eagle") {
        // fly high
    } else if (type == "crow") {
        // fly medium
    }
    // ... one block per bird
}
```

#### Why this is BAD:

| Problem | Explanation |
|---------|-------------|
| **Low Maintainability** | 50 birds = 50 else-if blocks. Debugging is a nightmare. |
| **Low Understandability** | You must understand ALL algorithms in one method. High cognitive load. |
| **Low Extensibility** | Adding a new bird = adding else-if in **every** method (fly, walk, eat...). You must re-test ALL existing behaviors. |

> **Important:** These are NOT simple if-else returning values. Each block contains an **independent algorithm**. That's what makes it problematic.

---

### V1 — Using Inheritance (Abstract Class + Child Classes)

```
abstract class Bird {
    color, type, name, weight, height, wingSpread

    abstract fly()     // No implementation here
    walk() { ... }
    eat() { ... }
}

class Hen extends Bird {
    fly() { /* fly low */ }
}

class Eagle extends Bird {
    fly() { /* fly high */ }
}

class Crow extends Bird {
    fly() { /* fly medium */ }
}
```

**Why abstract?**
- The requirement says ALL birds have different flying behavior
- You will never have a "just a bird" object — it's always a hen, eagle, crow, etc.
- So `Bird` should **never be instantiated** → make it `abstract`
- `fly()` has no "default" implementation → make it `abstract` too

#### Quick Recap — Abstract Class Rules:

| Rule | Answer |
|------|--------|
| Can an abstract class have methods WITH implementation? | Yes |
| Can an abstract class have ZERO abstract methods? | Yes |
| Can a regular (concrete) class have an abstract method? | **No** — because you could create an object and call a method with no implementation |
| Can you create an object of an abstract class? | **No** — but you CAN have a **reference** of abstract class type |

---

### The Kiwi Problem — Birds That Can't Fly

Kiwi is a bird. But kiwi **cannot fly**. Since `fly()` is abstract in `Bird`, Kiwi MUST implement it.

**Option 1: Throw an exception from `Kiwi.fly()`**

```java
class Kiwi extends Bird {
    fly() {
        throw new Exception("Kiwi cannot fly!");
    }
}
```

#### Why this is a TERRIBLE design:

**The Client Argument (very important concept):**

- **Who is a client?** Anyone who uses YOUR code — another team, another developer, even someone in your own team calling your functions.
- Example: You build the Bird library for Angry Birds game. Another team builds the rendering/game engine. They write:

```java
void renderBird(Bird b) {
    b.fly();   // expects this to ALWAYS work
}
```

- If `b` is a Kiwi → **exception is thrown** → client's code **breaks**
- Client now has to:
  1. Know which birds can fly and which can't
  2. Add if-else/try-catch in THEIR code
  3. Go into the details of YOUR code
  4. Update their code every time you add a new bird

> **This BREAKS ABSTRACTION** — the most important concept in OOD and LLD.

---

## 3. Abstraction — The Core Concept

**Definition:** Abstraction means defining contracts (methods) and hiding internal details. The client should NOT need to know how things work internally.

### Real-life examples the teacher gave:
- **Light switch** — you press it, light turns on. You don't think about how current flows.
- **Fan switch** — you press it, fan rotates. You don't apply Fleming's Left Hand Rule.
- **HashMap** — you call `put(key, value)`. You don't worry about hashCode calculation, array buckets, or balanced BSTs.

> **Teacher's Important Point:** HashMap was implemented using LinkedList in Java 7, changed to Balanced BST in Java 8 for performance. **Nothing changed for the user.** It just added a new interview question. That's the power of abstraction.

### The Rule:
> All work must be done on YOUR side. Your client should NEVER be asked to check things in your class, validate outputs, or handle your design flaws. If a method exists, it MUST work.

---

### V1.5 — Remove fly() from Bird class?

**Suggestion:** Since not all birds fly, remove `fly()` from `Bird`.

**Problem:**
```java
void renderBird(Bird b) {
    b.fly();   // COMPILE ERROR — fly() doesn't exist in Bird class
}
```

- Client would need separate functions for each bird type
- Polymorphism is lost
- Abstraction is broken again

---

## 4. Polymorphism (explained through this problem)

> **Polymorphism** = One single object can exhibit different types of behaviors / different forms.

- In **procedural programming**: if a function needs to accept both `ArrayList` and `Array`, you need 2 separate functions
- In **OOP with inheritance**: if `Hen`, `Eagle`, `Crow` all extend `Bird`, a function accepting `Bird` type can accept ALL of them

```java
void renderBird(Bird b) {   // accepts any child of Bird
    b.fly();
}
```

- **Reference** is of parent type (`Bird`)
- **Actual object** is of child type (`Hen`, `Eagle`, etc.)
- At **runtime**, the child's implementation is called

> With polymorphism, you can only call methods present in the **parent class/interface**. Methods exclusive to child classes can't be called through a parent reference.

---

### V2 — The Interface Solution (Final Design)

```
abstract class Bird {
    color, type, name, weight, height, wingSpread
    // NO fly() method here
    walk() { ... }
    eat() { ... }
}

interface Flyable {
    fly()       // just the method signature, no implementation
}

class Hen extends Bird implements Flyable {
    fly() { /* fly low */ }
}

class Eagle extends Bird implements Flyable {
    fly() { /* fly high */ }
}

class Kiwi extends Bird {
    // does NOT implement Flyable — no fly() method needed!
}
```

**Client code now:**
```java
void renderFlyingBird(Flyable f) {
    f.fly();   // GUARANTEED to work — only flyable birds accepted
}
```

- Kiwi **cannot** be passed here → **compile-time error** (caught early!)
- vs. the exception approach → **runtime error** (caught late, in production!)

> **Key insight:** Compile-time errors > Runtime errors. Always design to catch problems at compile time.

### What is an Interface?

| Property | Detail |
|----------|--------|
| Definition | A collection of methods (think: purely abstract class) |
| Attributes? | Ideally NO (technically possible, discussed later) |
| Method implementation? | Ideally NO (technically possible, discussed later) |
| Purpose | Define a **contract** — any class implementing it MUST provide implementations |
| Can a class implement multiple interfaces? | **Yes** (unlike extending multiple classes) |
| Can interface reference hold child objects? | Yes — just like parent class reference |

---

## 5. Why Not Multiple Inheritance? (Why Interfaces exist)

**Q: Why can't we just have a `FlyableBird` class and use multiple inheritance?**

### The Diamond Inheritance Problem

```
class Bird {
    flapWings() { /* method 1 */ }
}

class FlyableBird {
    flapWings() { /* method 2 */ }
}

class Eagle extends Bird, FlyableBird {   // NOT allowed in Java
    // which flapWings() is called?
}
```

- If both parents have the same method with different implementations → **ambiguity**
- Even if you say "pick the first one" → developers would need to check source code to know which is used → **abstraction broken**
- One dev changes the order to fix their method → breaks another method

> **Java removed multiple inheritance** to simplify collaborative development. C++ allows it but it creates confusion.

**Solution:** Use **interfaces** — since interfaces (ideally) have NO implementation, there's no ambiguity. Only one implementation exists (in the class or in the one parent class).

---

## 6. Why Not a Class Hierarchy? (Bird → FlyableBird → Hen, Eagle...)

```
Bird
├── FlyableBird
│   ├── Hen
│   ├── Eagle
│   └── Parrot
└── UnflyableBird
    └── Kiwi
```

**Problem:** What about hunting behavior? Maybe Hen, Parrot, and Kiwi hunt the same way but Eagle hunts differently. You'd need a DIFFERENT hierarchy for hunting. You can never find ONE clean hierarchy for all behaviors.

> **Thumb Rule:** If you need more than **one level of inheritance**, something is WRONG in your design. Reconsider everything, delete, start from scratch.

> **Another Thumb Rule:** If a method **throws an exception** because "this class shouldn't have it," or if you leave a method **blank/empty** — you've made a wrong design decision. Go back and redesign.

---

## 7. SOLID Principles (First 3 covered)

### S — Single Responsibility Principle (SRP)

> Every code entity must have **one single reason to change**.

| Scope | One responsibility means... |
|-------|---------------------------|
| Function | One behavior |
| Class | One entity |
| Package | One domain |
| Microservice | One bounded context |

**Example from the Bird problem:**
- BAD: `fly()` with 50 else-if blocks → changes whenever ANY bird's flying behavior changes (50 reasons to change)
- GOOD: `Hen.fly()` only changes if Hen's flying behavior changes (1 reason to change)

> **Teacher's Career Tip:** When you join a job, you'll have goals/numbers to chase. Ensure you are responsible for ONE thing and others aren't responsible for the same thing. Otherwise, you won't achieve your goals.

---

### O — Open/Closed Principle (OCP)

> Code should be **open for extension** but **closed for modification**.

- If something is tested and working in production → **DON'T TOUCH IT**
- Your design must allow adding new features WITHOUT modifying existing tested code

**Example:**
- BAD: Adding a new bird = modifying the `fly()` method (touching tested code)
- GOOD: Adding a new bird = creating a new class that extends `Bird` (existing code untouched)

---

### L — Liskov Substitution Principle (LSP)

> An object of a **child class** must be able to **completely replace** an object of the parent class.

- If parent has `fly()`, child MUST have a working `fly()`
- If child throws exception from `fly()` → child CANNOT replace parent → **LSP violated**

**Why it matters:** For anyone outside your ecosystem, they shouldn't need to know implementation details. They should trust that child objects work exactly like parent objects promised.

---

## 8. Remaining SOLID Principles (Mentioned, covered later)

| Principle | Full Name | Brief Hint |
|-----------|-----------|------------|
| **I** | Interface Segregation Principle | Don't force classes to implement methods they don't need |
| **D** | Dependency Inversion Principle | Depend on abstractions, not concrete implementations |

---

## 9. Interview & Exam Quick-Fire Points

1. **Abstract class** can have concrete methods and zero abstract methods
2. **Abstract method** can ONLY exist in an abstract class
3. You **cannot create an object** of an abstract class, but you CAN have a **reference**
4. **Interface** = contract = collection of method signatures (ideally no implementation)
5. **Diamond Problem** = why Java doesn't allow multiple inheritance of classes
6. **Polymorphism** = parent reference, child object, runtime method resolution
7. **Compile-time error** is always better than **runtime error** (design for it!)
8. **SOLID** = Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
9. If you need **>1 level of inheritance** → redesign
10. If a method **throws exception / is blank** because the class shouldn't have it → redesign

---

## 10. Assignments & Action Items (mentioned by teacher)

- Assignments released on **GitHub** — due by **next Monday**
- You'll be given **badly designed code** that violates SOLID principles
- Task: **identify which principle is violated** and **fix the code**
- Some students will be **picked to explain their solution in class** — prepare a short description
- Additional typed notes and book recommendations (soft copies) will be shared
- **Recommended:** Keep reading the books alongside the course for deeper understanding

---

## 11. What's Coming Next

- **Design Patterns** — the teacher mentioned that later they will remove all these child classes and have ONE bird class exhibiting different behaviors (hint: Strategy Pattern)
- Object creation problem → **solved by a design pattern** (hint: Factory Pattern)
- Interfaces with default implementations — why they exist and when to use them
- Interface Segregation Principle and Dependency Inversion Principle
- Practice problems in the next class
