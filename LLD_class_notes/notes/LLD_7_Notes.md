# LLD Lecture 7 - Detailed Notes

## Topics Covered
1. Builder Pattern (Recap)
2. Singleton Pattern - Breaking & Protecting (Reflection, Serialization)
3. Simple Factory Design Pattern
4. Factory Method Pattern (Abstract Factory of Spawners)
5. Prototype Design Pattern
6. Summary of Creational Design Patterns

---

## 1. Builder Pattern (Recap from Previous Lecture)

### Problem
You have an **immutable class** with **many attributes** and you want **flexibility** in object creation (skip optional fields, set fields in any order).

### The Teacher's Design Journey

**Step 1 - Identify the need:** The outer class is immutable (no setters). It has many attributes. Passing all of them via a constructor is rigid and error-prone.

**Step 2 - Create an inner class (the Builder):** This inner class has the **exact same attributes** as the outer class. The outer class constructor takes **one parameter**: an object of this inner Builder class.

**Step 3 - Make the Builder static:** Since the client needs to create a Builder object *without* first having an outer class object, the Builder must be a **static inner class**. (Static inner classes can be instantiated without an instance of the outer class.)

**Step 4 - Add setters with return `this`:** Each setter in the Builder class returns the Builder object itself (`return this`). This enables **method chaining**.

**Step 5 - Add a `build()` method:** This method inside the Builder class calls `new OuterClass(this)` and returns the immutable outer object.

### How the Client Uses It

```java
Student student = Student.Builder()  // create builder
    .setName("Alice")                // chain setters (any order)
    .setAge(21)                      // skip optional fields
    .setEmail("alice@example.com")
    .build();                        // convert to immutable object
```

### Key Rules

| Rule | Reason |
|------|--------|
| Builder must be a **static** inner class | So it can be created without an outer class instance |
| Outer class constructor takes **Builder object** | Single parameter instead of many |
| Setters return `this` | Enables method chaining |
| `build()` method calls outer constructor | Converts mutable Builder to immutable object |
| Outer class has **no setters** | Ensures immutability after creation |

---

## 2. Singleton Pattern - Breaking & Protecting

### Recap: Double-Checked Locking (Final Solution from Previous Class)

```java
public class Singleton {
    private static volatile Singleton instance = null;

    private Singleton() {
        // private constructor
    }

    public static Singleton getInstance() {
        if (instance == null) {                    // 1st check (no lock)
            synchronized (Singleton.class) {       // acquire lock
                if (instance == null) {            // 2nd check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Why `volatile`?** Prevents instruction reordering during object creation. Without it, a half-initialized object could be visible to another thread.

### Loophole 1: Reflection Attack

#### The Problem
Java's **Reflection API** can:
- Access the private constructor
- Change its accessibility to public
- Call the constructor from outside the class
- Create a **second object**, violating singleton

#### The Fix: Guard Inside the Constructor

```java
private Singleton() {
    if (instance != null) {
        throw new RuntimeException(
            "Cannot create another instance - use getInstance()");
    }
}
```

**Key Insight:** The constructor is already thread-safe (only one thread enters at a time due to the synchronized block above), so there is no race condition concern with this check inside the constructor.

#### Why No Synchronization Needed in the Constructor?
A student asked: "What if two threads both see `instance == null` inside the constructor?" The teacher clarified: The constructor is already locked. At a time, only one single thread can create the object. So no additional synchronization is needed inside the constructor.

### Loophole 2: Serialization Attack

The teacher mentioned that **serialization/deserialization** can also break singleton behavior. This was left as a **self-study topic**.

**Homework:** Research how serialization breaks singleton and how to fix it (hint: implement `readResolve()` method).

### Interview Checklist for Singleton

| Attack Vector | Fix |
|--------------|-----|
| Multiple threads calling `getInstance()` | Double-checked locking + `synchronized` |
| Instruction reordering | `volatile` keyword |
| Reflection API bypassing private constructor | Throw exception inside constructor if instance already exists |
| Serialization/Deserialization | Self-study: `readResolve()` method |

---

## 3. Simple Factory Design Pattern

### The Teacher's Design Journey (V0 to Final)

#### Scenario
A game where stones (small, medium, large) are spawned and the character must dodge them.

#### V0 - The Naive Approach (Tightly Coupled)

```java
public class Game {
    public void play() {
        // Logic and object creation MIXED together
        Stone s1 = new MediumStone();
        // ... some game logic ...
        Stone s2 = new LargeStone();
        // ... more logic ...
        if (someCondition) {
            Stone s3 = new SmallStone();
        }
    }
}
```

**Problems identified by the teacher:**
1. Logic and object creation are **tightly coupled**
2. Client must know about all concrete classes (`SmallStone`, `MediumStone`, `LargeStone`)
3. Client must know constructor details (arguments, types)
4. Adding a new stone type requires changing the Game class

#### V1 - Extract Object Creation into a Separate Class

**Teacher's reasoning:** "The logic and the object creation should not be happening at the same place. Let us segregate them."

First, create the class hierarchy:

```java
// Abstraction
public abstract class Stone {
    // common stone properties
}

// Concrete implementations
public class SmallStone extends Stone { }
public class MediumStone extends Stone { }
public class LargeStone extends Stone { }
```

Then, create the factory:

```java
public class StoneFactory {
    public static Stone createStone(String type) {
        if (type.equals("large")) {
            return new LargeStone();
        } else if (type.equals("medium")) {
            return new MediumStone();
        } else if (type.equals("small")) {
            return new SmallStone();
        }
        return null;
    }
}
```

**Why static?** So the client does not need to create an object of the factory class. They call it directly using the class name.

#### Client Code After Simple Factory

```java
public class Game {
    public void play() {
        Stone s1 = StoneFactory.createStone("medium");
        // ... game logic only, no constructors ...
        Stone s2 = StoneFactory.createStone("large");
    }
}
```

### Does Simple Factory Violate SRP?

**Teacher's Answer: No.** The only reason for change is adding a new type of object. The factory's single responsibility is object creation. All game logic lives elsewhere.

### When to Use Simple Factory

> **Teacher's Rule:** "Whenever you have inheritance, whenever you have multiple different type of classes, and you want an appropriate object at runtime based on some conditions/parameters, implement a simple factory. You're not going to have your logic and object creation at the same time, at the same place."

### Benefits

| Benefit | Explanation |
|---------|-------------|
| Abstraction | Client only knows about `Stone`, not concrete types |
| Decoupling | Logic and object creation are separated |
| Maintainability | Adding new stone type does NOT change the Game class |
| Simplicity | Client calls one method instead of multiple constructors |

---

## 4. Factory Method Pattern / Abstract Factory (Stone Spawner)

### The Teacher's Design Journey (Building on Simple Factory)

#### New Problem
Different levels need different **strategies** for spawning stones:
- Level 1: Random stones
- Level 2: Equal distribution of all stone types
- Level 3: Custom ratio
- Level 4: Introduces a 4th stone type

The Simple Factory creates **one object at a time** based on a parameter. Now we need entire **strategies/algorithms** for how stones are produced over time.

#### Key Constraint (from teacher)
The client calls a `generate()` method **one stone at a time** (not a list). The function must **remember** what stones it has already produced and maintain the strategy internally.

#### V0 - Naive Idea
Return a list of stones. **Rejected** because: "The client wants to call the function for generating one single stone at a time. They don't want a list because these stones are not going to be generated as a list. There will be one stone coming at a time in the pixel."

#### V1 - Create a Spawner Hierarchy

```java
// Abstraction for spawning strategy
public interface StoneSpawner {
    Stone generate();
}

// Random spawning
public class RandomSpawner implements StoneSpawner {
    public Stone generate() {
        int random = (int)(Math.random() * 3);
        switch (random) {
            case 0: return new SmallStone();
            case 1: return new MediumStone();
            case 2: return new LargeStone();
        }
        return null;
    }
}

// Uniform spawning (equal distribution)
public class UniformSpawner implements StoneSpawner {
    private int counter = 0;  // NOT static - instance-level

    public Stone generate() {
        Stone stone;
        switch (counter) {
            case 0: stone = new SmallStone(); break;
            case 1: stone = new MediumStone(); break;
            case 2: stone = new LargeStone(); break;
            default: stone = null;
        }
        counter = (counter + 1) % 3;  // cycles: 0 -> 1 -> 2 -> 0 -> ...
        return stone;
    }
}
```

#### V2 - Apply Simple Factory on Top of Spawners

**Teacher's key design principle:** "If in your design you are asking the client to create an object of a concrete class, then you are not providing the level of abstraction which you can provide."

The client should NOT do `new RandomSpawner()`. Apply the same factory idea again:

```java
public class StoneSpawnerFactory {
    public static StoneSpawner createSpawner(String type) {
        if (type.equals("random")) {
            return new RandomSpawner();
        } else if (type.equals("uniform")) {
            return new UniformSpawner();
        }
        return null;
    }
}
```

#### Final Client Code (Fully Abstract)

```java
public class Game {
    public void play() {
        // Client specifies strategy, not concrete class
        StoneSpawner spawner = StoneSpawnerFactory.createSpawner("uniform");

        // Generate stones one at a time
        for (int i = 0; i < 100; i++) {
            Stone stone = spawner.generate();
            // use stone in game...
        }
    }
}
```

### Why the Counter Should NOT Be Static

**Teacher's insight (interview-relevant):** Each game instance gets its own spawner object. If the counter were static, all game instances would share the same counter, meaning one game's spawning would affect another game. The counter must be an **instance variable**.

> "If a new object is being created, then you can assume that things are starting from scratch... If we select a uniform stone generator, it should be uniform in that specific instance of the game, not across all the instances."

### Uniform Spawner - The Elegant Modulo Trick

```
counter = (counter + 1) % 3
```

| Call # | counter before | Stone returned | counter after |
|--------|---------------|---------------|---------------|
| 1 | 0 | SmallStone | 1 |
| 2 | 1 | MediumStone | 2 |
| 3 | 2 | LargeStone | 0 |
| 4 | 0 | SmallStone | 1 |
| ... | ... | ... | ... |

> **Teacher:** "No need to maintain any queue, no need to maintain any history, no need to maintain any list. One single integer can solve this problem for us."

### Design Layers Summary

```
Client (Game class)
  |
  v
StoneSpawnerFactory  -->  creates --> StoneSpawner (interface)
                                          |
                          +---------------+----------------+
                          |                                |
                    RandomSpawner                   UniformSpawner
                          |                                |
                          v                                v
                  StoneFactory (or direct constructor calls)
                          |
              +-----------+-----------+
              |           |           |
         SmallStone  MediumStone  LargeStone
```

---

## 5. Prototype Design Pattern

### The Teacher's Design Journey

#### Problem Statement
Continuing the stone game example: creating stone objects is **computationally expensive** (DB calls, complex algorithms in constructor). But many objects are **largely similar** -- most attributes have the same values, only a few differ.

#### V0 - Students' Over-Engineered Solutions (What NOT to Do)

The teacher presented this problem and students suggested:
1. Another class that returns a Builder -- **overcomplicated**
2. A caching layer (Redis) -- **not needed at this level**
3. Maintaining unique IDs -- **does not solve the problem**
4. Concurrency solutions -- **not related**

> **Teacher's insight:** "This is what happens when you over-complicate things."

#### The Actual Solution: Copy-Paste (Clone)

> **Teacher:** "You have done multiple times, you have already done whatever is the solution, you have implemented it and you do it on a regular basis."

**The approach:**
1. Create **one** object the expensive way (full constructor with all computations)
2. For all subsequent similar objects, **clone** the first object (deep copy)
3. Modify only the attributes that differ using setters

```java
public abstract class Stone implements Cloneable {
    private int weight;
    private String color;
    private int size;
    // ... many more expensive-to-compute attributes

    @Override
    public Stone clone() {
        try {
            return (Stone) super.clone();  // deep copy
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }

    // setters for mutable attributes
    public void setColor(String color) {
        this.color = color;
    }
}
```

#### Prototype Registry

When you have **multiple classes of objects** from the same class (e.g., 1000 short movies and 1000 long movies), maintain a **registry** of prototype objects:

```java
public class StoneRegistry {
    private Map<String, Stone> prototypes = new HashMap<>();

    // Register a prototype (created once, expensive)
    public void registerPrototype(String key, Stone prototype) {
        prototypes.put(key, prototype);
    }

    // Clone from registry (cheap)
    public Stone createStone(String key) {
        Stone prototype = prototypes.get(key);
        if (prototype != null) {
            return prototype.clone();  // DEEP copy, not shallow
        }
        return null;
    }
}
```

#### Usage

```java
// One-time expensive setup
StoneRegistry registry = new StoneRegistry();
registry.registerPrototype("large", new LargeStone());  // expensive constructor
registry.registerPrototype("small", new SmallStone());   // expensive constructor

// Cheap creation (no constructor, just clone)
Stone s1 = registry.createStone("large");
s1.setColor("red");  // modify only what differs

Stone s2 = registry.createStone("large");
s2.setColor("blue");
```

### Critical Detail: Deep Copy vs Shallow Copy

> **Teacher (emphasized):** "Create a **deep copy**, not a shallow copy."

| Copy Type | Behavior | Safe for Prototype? |
|-----------|----------|-------------------|
| Shallow Copy | Copies references to nested objects | NO - modifying clone modifies original |
| Deep Copy | Creates new instances of all nested objects | YES - clone is fully independent |

### Why Clone Is Faster Than Constructor

A student asked: "How is a deep copy faster than a constructor?"

**Teacher's answer:** The constructor does expensive operations: DB calls, caching layer fetches, complex algorithm computations. A deep copy skips ALL of that. You are fetching data **in-memory from the existing object** itself, which is not an expensive operation.

### When to Use Prototype Pattern

| Condition | Example |
|-----------|---------|
| Constructor is expensive (DB calls, heavy computation) | Object attributes computed via complex algorithms |
| Many similar objects needed | 1000 stones with same weight/size but different colors |
| Objects differ only in a few attributes | All large stones same except color |
| Multiple "classes" of objects from one class | Short movies vs Long movies (same Movie class) |

---

## 6. Summary: All Creational Design Patterns

| # | Pattern | Problem It Solves | Key Idea |
|---|---------|-------------------|----------|
| 1 | **Singleton** | Restrict to exactly one instance | Private constructor + static getInstance() + double-checked locking |
| 2 | **Builder** | Flexible creation of immutable objects with many attributes | Static inner class with chained setters + build() method |
| 3 | **Simple Factory** | Decouple object creation from business logic when there are multiple types | Static method that takes a parameter and returns the right subclass |
| 4 | **Factory Method** | Different strategies/algorithms for creating families of objects | Factory that returns different factory/spawner objects (factory of factories) |
| 5 | **Prototype** | Avoid expensive constructor calls for similar objects | Clone a prototype object, modify only what differs |

> **Teacher:** "All these design patterns are known as **creational design patterns** because they talk about creating an object."

---

## 7. Interview Questions from This Lecture

### Direct Questions
1. What is the Builder pattern? When would you use it?
2. How do you break a Singleton? (Reflection, Serialization)
3. How do you protect a Singleton from Reflection attacks?
4. What is the Simple Factory pattern?
5. Does Simple Factory violate SRP? (Answer: No)
6. What is the Prototype pattern? When is it useful?
7. What is the difference between shallow copy and deep copy?
8. How is cloning faster than calling a constructor?
9. What are all the creational design patterns? What problems do they solve?

### Tricky / Follow-Up Questions
- In Uniform Spawner, should the counter be static or instance-level? Why?
- In Singleton, why is `volatile` needed even with double-checked locking?
- In Singleton, does the constructor check need synchronization? (No -- constructor is already locked)
- Can you combine Factory and Prototype patterns? (Yes -- factory can internally use prototype registry)

### Machine Coding Round Question
Design a stone-spawning game engine where:
- Multiple types of stones exist (small, medium, large)
- Client should not create objects directly
- Different spawning strategies (random, uniform) should be supported
- Strategy should be selectable without knowing concrete classes

---

## 8. Teacher's Special Insights & Career Tips

### Design Principles (Repeated Throughout)

1. **"Logic and object creation should not be at the same place."** -- This is the foundational motivation for Factory patterns.

2. **"Client should only deal with the abstraction."** -- If your design asks the client to create concrete class objects, you are not providing proper abstraction.

3. **"Don't over-complicate things."** -- When students suggested caching layers, concurrency solutions, and builder-returning classes for the prototype problem, the teacher pointed out the simplest solution was just copy-paste (clone).

4. **"Whenever you have inheritance and need runtime object selection, use a factory."** -- This is a universal thumb rule for deciding when to apply the factory pattern.

### Abstraction Layering Pattern
The teacher demonstrated a recurring technique: when you have a hierarchy of concrete classes, **never** expose them to the client. Always put a factory in front. If the factories themselves become a hierarchy, put another factory in front of *them*. Each layer adds abstraction.

### Self-Study Assignments
1. Read about **Reflection API** in Java -- how to access private constructors
2. Research **Serialization attacks** on Singleton and how to fix them
3. Complete all SOLID principle assignments and push to GitHub
4. Prepare to explain SOLID solutions in < 5 minutes using iPad (not reading code)

---

## 9. Exam Prep - Quick-Fire Facts

| Fact | Detail |
|------|--------|
| Builder inner class must be | `static` |
| Singleton instance must be | `static volatile` |
| Singleton protection from reflection | Throw exception in constructor if instance exists |
| Simple Factory method must be | `static` |
| Prototype requires | `clone()` / deep copy |
| Prototype registry data structure | `HashMap<String, Object>` |
| Uniform distribution trick | `counter = (counter + 1) % n` |
| All patterns in this lecture are | **Creational** design patterns |
| Next lecture topic | **Structural** design patterns |
| Creational patterns count | 5 (Singleton, Builder, Simple Factory, Factory Method, Prototype) |

### Pattern Selection Guide

```
Need exactly one instance?          --> Singleton
Need flexible immutable creation?   --> Builder
Need to hide concrete class from client? --> Simple Factory
Need different creation strategies? --> Factory Method
Need many similar expensive objects? --> Prototype
```
