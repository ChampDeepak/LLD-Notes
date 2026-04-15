# LLD Lecture 6 - Immutable Classes & Builder Design Pattern

---

## Table of Contents
1. [Recap: Dependency Inversion & Singleton](#1-recap-dependency-inversion--singleton)
2. [Immutable Objects - Concept & Need](#2-immutable-objects---concept--need)
3. [Teacher's Design Journey: Making a Class Immutable (V0 to Final)](#3-teachers-design-journey-making-a-class-immutable-v0-to-final)
4. [Teacher's Design Journey: Builder Pattern (V0 to Final)](#4-teachers-design-journey-builder-pattern-v0-to-final)
5. [Builder Design Pattern - Final Summary](#5-builder-design-pattern---final-summary)
6. [Code Examples](#6-code-examples)
7. [Interview Questions](#7-interview-questions)
8. [Design Decisions Summary](#8-design-decisions-summary)
9. [Important Points & Definitions](#9-important-points--definitions)
10. [Exam Prep - Quick-Fire Facts](#10-exam-prep---quick-fire-facts)
11. [Teacher's Special Insights](#11-teachers-special-insights)

---

## 1. Recap: Dependency Inversion & Singleton

### Dependency Inversion Principle (from previous class)
- Earlier: objects of a concrete class were created **inside** another class = tight coupling.
- Fix: pass an object of an **abstraction** (interface/abstract class) through the constructor or method.
- Benefit: if you follow the **last** SOLID principle (Dependency Inversion), the **first two** (Single Responsibility and Open-Closed) are automatically followed.
- Your code becomes extensible -- to change behavior, you just pass a different object from the caller side. No code changes inside the class.

### Singleton - Double Check Locking (homework from previous class)
- Even with double check locking, the Singleton can **still break** in some scenarios.
- **Homework given:** Find scenarios where double check locking fails, and how to prevent it.
- **This is a very frequently asked interview question:** "Implement double check locking, then break it, then fix it."

---

## 2. Immutable Objects - Concept & Need

### What is an Immutable Object?
An object whose attribute values **cannot be changed** after creation. Once constructed, its state is fixed forever.

> **Immutable does NOT mean Singleton.** You can have as many immutable objects as you want. The constraint is that once created, values don't change.

### Where Are Immutable Objects Needed?

| Use Case | Explanation |
|---|---|
| **DTOs (Data Transfer Objects)** | Should be immutable -- you don't want data to change during transit |
| **Config objects** | e.g., game configuration for a specific game instance |
| **Constants** | Values that should never change |
| **String class in Java** | Every modification creates a new String; the original is never mutated |

### How Java String Immutability Works
- When you add a character to a String, internally a **new String** is created.
- If the new string already exists in the **String Pool**, the reference simply points to that existing string.
- The original string is never modified.

---

## 3. Teacher's Design Journey: Making a Class Immutable (V0 to Final)

This is the step-by-step process the teacher walked through to convert a regular class into an immutable class. This is the **most important section** -- it shows the thinking process.

### Starting Class (V0 - Mutable)

```java
class DemoImmutable {
    int a;
    String b;
    ArrayList<Integer> list;
    ArrayList<ArrayList<Integer>> list2D;

    DemoImmutable(int a, String b, ArrayList<Integer> list,
                  ArrayList<ArrayList<Integer>> list2D) {
        this.a = a;
        this.b = b;
        this.list = list;
        this.list2D = list2D;
    }

    // public getters and setters for all fields
    public int getA() { return a; }
    public void setA(int a) { this.a = a; }
    public ArrayList<Integer> getList() { return list; }
    public void setList(ArrayList<Integer> list) { this.list = list; }
    // ... etc.
}
```

**Problem:** This class is completely mutable. Anyone can change any field at any time.

---

### Step 1: Make all fields `private`

```java
private int a;
private String b;
private ArrayList<Integer> list;
private ArrayList<ArrayList<Integer>> list2D;
```

**Why:** `private` means the fields cannot be accessed directly outside the class.

**Access Modifier Recap:**

| Modifier | Who Can Access |
|---|---|
| `private` | Only the same class |
| `default` (no modifier) | Any class in the same package |
| `protected` | Same package + subclasses |
| `public` | Anywhere |

---

### Step 2: Make all fields `final`

```java
private final int a;
private final String b;
private final ArrayList<Integer> list;
private final ArrayList<ArrayList<Integer>> list2D;
```

**Why:** `final` means once a value is assigned, it cannot be reassigned.

**`final` keyword -- three uses:**

| Applied To | Meaning |
|---|---|
| **Variable** | Value cannot be reassigned after initialization |
| **Method** | Subclasses cannot override this method |
| **Class** | Class cannot be extended (no inheritance) |

---

### Step 3: Remove all setters

Once fields are `final`, setters become useless (they give compilation errors). Delete them.

**Why:** Setters allow changing state, which violates immutability.

---

### Step 4: Deep copy in the constructor (CRITICAL)

**The Bug:** If you simply assign `this.list = list` in the constructor, the caller still holds a reference to the same list. They can modify it from outside!

```java
// In main:
ArrayList<Integer> myList = new ArrayList<>();
DemoImmutable obj = new DemoImmutable(10, "hello", myList, ...);

System.out.println(obj.getList().size()); // 0

myList.add(100);  // Modifying from OUTSIDE the class!
System.out.println(obj.getList().size()); // 1  <-- IMMUTABILITY BROKEN!
```

**Fix:** Create a **deep copy** in the constructor:

```java
DemoImmutable(int a, String b, ArrayList<Integer> list,
              ArrayList<ArrayList<Integer>> list2D) {
    this.a = a;
    this.b = b;
    this.list = new ArrayList<>(list);           // deep copy
    this.list2D = new ArrayList<>();             // deep copy for 2D list
    for (ArrayList<Integer> inner : list2D) {
        this.list2D.add(new ArrayList<>(inner));
    }
}
```

**Key Insight:** Both references were pointing to the **exact same memory location**. By creating a deep copy, the internal list is now independent of the external one.

---

### Step 5: Deep copy in the getters (CRITICAL)

**The Bug:** Even if the constructor uses deep copy, the getter returns the actual reference. The caller can then modify it!

```java
ArrayList<Integer> stolen = obj.getList();
stolen.add(999);  // IMMUTABILITY BROKEN AGAIN!
```

**Fix:** Return a deep copy from the getter:

```java
public ArrayList<Integer> getList() {
    return new ArrayList<>(this.list);  // return deep copy
}
```

---

### Step 6: Make the class `final`

```java
public final class DemoImmutable { ... }
```

**Why:** Without `final`, someone can extend the class, override methods, and manipulate internal data through the overridden methods.

---

### Step 7: Prevent internal methods from mutating state

**The Bug:** Even with all the above, a method inside the class itself could mutate the list:

```java
public void addToList(int value) {
    this.list.add(value);  // This compiles! final doesn't prevent this!
}
```

**Why does `final` not prevent this?**

> `final` on a reference variable means the **reference** cannot point to a different object. But you can still **modify the object** it points to. The list reference stays the same, but elements can be added/removed from that same list.

**This applies to any object reference, not just lists.** If you have a `final FileStorageService` reference, you can still call its setters and change its internal state.

**Solution (homework):** For ArrayList, there is a built-in solution (e.g., `Collections.unmodifiableList()`). For custom objects, you may need a wrapper class. The teacher said the wrapper pattern will be studied with more design patterns later.

---

### Final Immutable Class Checklist

| Step | What To Do | Why |
|---|---|---|
| 1 | All fields `private` | Prevent direct external access |
| 2 | All fields `final` | Prevent reassignment |
| 3 | Remove all setters | Prevent state changes via setter methods |
| 4 | Deep copy in constructor | Prevent external reference mutation |
| 5 | Deep copy in getters | Prevent returned reference mutation |
| 6 | Class is `final` | Prevent subclass override attacks |
| 7 | No internal mutating methods | Prevent the class itself from breaking immutability |

---

## 4. Teacher's Design Journey: Builder Pattern (V0 to Final)

Now the teacher identified a **new problem** with immutable classes: how do you conveniently create objects when there are many attributes and no setters?

### The Problem

- Immutable class has, say, 20 attributes.
- All values must be passed via the constructor (no setters allowed).
- The client must **remember the exact order** of 20 arguments.
- If a new attribute is added, all existing client code breaks.

---

### V0: Telescoping Constructors

**Idea:** Create multiple constructors, each calling another with fewer parameters.

```java
// Constructor 1: only 'a'
DemoImmutable(int a) {
    this.a = a;
}

// Constructor 2: 'a' + 'list'
DemoImmutable(int a, ArrayList<Integer> list) {
    this(a);           // calls Constructor 1
    this.list = list;
}

// Constructor 3: 'a' + 'list' + 'list2D'
DemoImmutable(int a, ArrayList<Integer> list,
              ArrayList<ArrayList<Integer>> list2D) {
    this(a, list);     // calls Constructor 2
    this.list2D = list2D;
}

// Constructor 4: all fields
DemoImmutable(int a, ArrayList<Integer> list,
              ArrayList<ArrayList<Integer>> list2D, String b) {
    this(a, list, list2D);  // calls Constructor 3
    this.b = b;
}
```

**Advantages:**
- Backward compatible -- old constructors still work.
- Each new constructor calls existing ones, reducing code duplication.

**Problems:**
- Client still needs to remember the argument order for large constructors.
- Exponential growth of constructors if you want all permutations.
- Not scalable beyond a few attributes.

**Teacher's verdict:** Not a good solution.

---

### V1: Pass a HashMap

**Idea:** Instead of individual arguments, pass a `HashMap<String, Object>` to the constructor.

```java
DemoImmutable(Map<String, Object> params) {
    this.a = (int) params.get("a");
    this.b = (String) params.get("b");
    // ...
}
```

**Problem:** HashMap requires all values to be the same type. Using `Object` as the value type is **not type-safe** -- any object can be passed, leading to runtime errors. The teacher strongly advises against this:

> "We can use the Object class and solve a lot of problems but we'll have one single concern everywhere -- any object can enter. Even with polymorphism, when we wanted all birds that can fly, we created an interface. We didn't keep the Object class because that allows all classes."

**Teacher's verdict:** Rejected -- not type-safe.

---

### V2: Pass an Object of a Helper Class

**Idea:** Create a separate class with the same attributes + public setters. Pass an object of this class to the immutable class's constructor.

```java
class X {
    int a;
    String b;
    ArrayList<Integer> list;

    public void setA(int a) { this.a = a; }
    public void setB(String b) { this.b = b; }
    // ...
}
```

The constructor of the immutable class takes an `X` object:

```java
DemoImmutable(X x) {
    this.a = x.a;
    this.b = x.b;
    this.list = new ArrayList<>(x.list);  // deep copy!
}
```

**Client usage:**

```java
X x = new X();
x.setA(10);
x.setB("hello");
DemoImmutable obj = new DemoImmutable(x);
```

**Key Insight:** We are **segregating two concerns:**
1. **Initialization of values** -- happens in the helper class (mutable, has setters).
2. **Immutability** -- enforced in the actual class (no setters, deep copies).

**Problem:** Maintaining two separate classes is hard. If you add a field to the immutable class, you must remember to add it to class X too.

---

### V3: Make the Helper Class an Inner Class

**Idea:** Place class X **inside** the immutable class as a `static inner class`.

**Why static?** Because we don't want to need an object of the outer class to create the builder -- that is a chicken-and-egg problem.

**Why inner class?**
- Easier to maintain -- both classes are in the same file.
- Client interacts with only one class name: `DemoImmutable.Builder`.
- Changes to the outer class naturally remind you to update the inner class.

---

### V4 (Final): Add `build()` Method + Setter Chaining

**Two final touches:**

1. **Setters return `this`** (the builder object) -- enables method chaining.
2. **`build()` method** -- converts the builder object into an immutable object.

**This is the Builder Design Pattern.**

---

## 5. Builder Design Pattern - Final Summary

### Structure

```
+---------------------------------------+
|       DemoImmutable (final)           |
|---------------------------------------|
| - private final int a                 |
| - private final String b             |
| - private final ArrayList<> list     |
|---------------------------------------|
| + getA(): int                         |
| + getB(): String                      |
| + getList(): ArrayList (deep copy)    |
|                                       |
|  +-------------------------------+    |
|  |   Builder (static inner)      |    |
|  |-------------------------------|    |
|  | - int a                       |    |
|  | - String b                    |    |
|  | - ArrayList<> list            |    |
|  |-------------------------------|    |
|  | + setA(int): Builder          |    |
|  | + setB(String): Builder       |    |
|  | + setList(ArrayList): Builder |    |
|  | + build(): DemoImmutable      |    |
|  +-------------------------------+    |
+---------------------------------------+
```

### Client Usage

```java
DemoImmutable obj = new DemoImmutable.Builder()
    .setA(10)
    .setB("hello")
    .setList(someList)
    .build();    // <-- Converts to immutable object
```

### Key Points About Setters in Builder
- Each setter **returns `this`** (the builder object), enabling chaining.
- They are NOT `void` -- they return `Builder`.

```java
public Builder setA(int a) {
    this.a = a;
    return this;  // enables chaining
}
```

### The `build()` Method

```java
public DemoImmutable build() {
    // Optional: validation logic here
    // e.g., if (a == 0) throw new IllegalArgumentException(...);
    return new DemoImmutable(this);
}
```

- Calls the private constructor of the outer class, passing `this` (the builder).
- Can include **validation logic** -- check if required fields are set, throw exceptions if not.

### Real-World Example: StringBuilder

`StringBuilder` in Java is exactly this pattern:
- You keep appending characters (mutable operations).
- You call `.toString()` (the "build" method) to get an **immutable String**.

### When Is Builder Pattern Needed?

| Scenario | Need Builder? |
|---|---|
| Immutable class with many attributes | **YES** |
| Mutable class with many attributes | **NO** -- just use setters on the class itself |
| Immutable class with few attributes (2-3) | **Not necessary** -- constructor is fine |
| Any class where construction is complex | **Possibly** -- builder adds clarity |

> **Builder is only needed for immutable classes.** If the class is not immutable, you can have setters directly and don't need a builder.

---

## 6. Code Examples

### Complete Immutable Class with Builder Pattern

```java
public final class DemoImmutable {

    private final int a;
    private final String b;
    private final ArrayList<Integer> list;
    private final ArrayList<ArrayList<Integer>> list2D;

    // Private constructor -- only Builder can call this
    private DemoImmutable(Builder builder) {
        this.a = builder.a;
        this.b = builder.b;
        this.list = new ArrayList<>(builder.list);           // deep copy
        this.list2D = new ArrayList<>();
        for (ArrayList<Integer> inner : builder.list2D) {
            this.list2D.add(new ArrayList<>(inner));         // deep copy
        }
    }

    // Getters return deep copies for object types
    public int getA() { return a; }
    public String getB() { return b; }

    public ArrayList<Integer> getList() {
        return new ArrayList<>(list);  // deep copy
    }

    public ArrayList<ArrayList<Integer>> getList2D() {
        ArrayList<ArrayList<Integer>> copy = new ArrayList<>();
        for (ArrayList<Integer> inner : list2D) {
            copy.add(new ArrayList<>(inner));
        }
        return copy;  // deep copy
    }

    // ---- Static Inner Builder Class ----
    public static class Builder {
        private int a;
        private String b;
        private ArrayList<Integer> list = new ArrayList<>();
        private ArrayList<ArrayList<Integer>> list2D = new ArrayList<>();

        public Builder setA(int a) {
            this.a = a;
            return this;
        }

        public Builder setB(String b) {
            this.b = b;
            return this;
        }

        public Builder setList(ArrayList<Integer> list) {
            this.list = list;
            return this;
        }

        public Builder setList2D(ArrayList<ArrayList<Integer>> list2D) {
            this.list2D = list2D;
            return this;
        }

        public DemoImmutable build() {
            // Add validation here if needed
            return new DemoImmutable(this);
        }
    }
}
```

### Client Code

```java
public class Main {
    public static void main(String[] args) {
        ArrayList<Integer> myList = new ArrayList<>();
        myList.add(1);
        myList.add(2);

        DemoImmutable obj = new DemoImmutable.Builder()
            .setA(10)
            .setB("hello")
            .setList(myList)
            .build();

        // The object is now immutable
        System.out.println(obj.getA());           // 10
        System.out.println(obj.getList().size());  // 2

        // Trying to break immutability from outside:
        myList.add(3);
        System.out.println(obj.getList().size());  // Still 2! (deep copy in constructor)

        // Trying to break via getter:
        ArrayList<Integer> stolen = obj.getList();
        stolen.add(99);
        System.out.println(obj.getList().size());  // Still 2! (deep copy in getter)
    }
}
```

---

## 7. Interview Questions

### Q1: How do you make a class immutable in Java?
**Answer:** Apply all 7 steps from the checklist above: private final fields, no setters, deep copy in constructor, deep copy in getters, final class, no internal mutating methods.

### Q2: Why is making fields `final` not enough to guarantee immutability for object types?
**Answer:** `final` only prevents reassignment of the reference. The object the reference points to can still be mutated (e.g., `list.add()`). You need deep copies in both constructor and getters.

### Q3: What is the Builder Design Pattern? When would you use it?
**Answer:** Builder separates the construction of a complex/immutable object from its representation. Use it when an immutable class has many attributes, making constructor calls unwieldy. The builder provides a fluent API with setter chaining and a `build()` method.

### Q4: Can you implement Singleton with double-check locking, then break it?
**Answer:** (Homework from this class -- the teacher said this is a very common interview question. Hint: without `volatile`, instruction reordering by the JVM can cause another thread to see a partially constructed object.)

### Q5: What is a telescoping constructor? What are its problems?
**Answer:** A pattern where multiple constructors are defined, each calling another with fewer parameters. Problems: client must remember argument order; exponential growth of constructors for all permutations; not scalable.

### Q6: Why must the Builder be a `static` inner class?
**Answer:** If it were non-static, you would need an object of the outer class to create a Builder -- but the Builder's purpose is to create that outer class object. That is a chicken-and-egg problem. Static inner class can be instantiated independently: `new OuterClass.Builder()`.

### Q7: How does `StringBuilder` relate to the Builder pattern?
**Answer:** `StringBuilder` maintains a mutable internal char array. You chain `.append()` calls. When you call `.toString()`, it produces an immutable `String` object. This is exactly the builder pattern.

### Q8: How do you ensure required fields are set in a Builder?
**Answer:** Inside the `build()` method, validate that all required fields have been initialized. If any are missing, throw an exception instead of calling the constructor.

### Q9: What is the difference between shallow copy and deep copy?
**Answer:** Shallow copy copies the reference (both point to the same object in memory). Deep copy creates a new object with copied values (independent of the original). Immutable classes require deep copies for all mutable object-type fields.

---

## 8. Design Decisions Summary

| Decision | Why |
|---|---|
| Deep copy over reference assignment | Prevents external mutation of internal state |
| Inner class over separate class for Builder | Easier maintenance; client sees one class; changes to outer class remind you to update builder |
| Static inner class over non-static | Avoids chicken-and-egg problem; can create Builder without outer class object |
| Setter returns `this` over `void` | Enables fluent method chaining (`setA().setB().build()`) |
| Builder over telescoping constructors | Scalable, readable, no argument-order issues |
| Builder over HashMap | Type safety; HashMap with Object values allows any type, no compile-time checks |
| `final` class over non-final | Prevents subclass from overriding methods and breaking immutability |

---

## 9. Important Points & Definitions

| Term | Definition |
|---|---|
| **Immutable Object** | An object whose state cannot change after creation |
| **Immutable != Singleton** | Immutable is about state; Singleton is about count. They are independent concepts |
| **`final` variable** | Cannot be reassigned after initialization |
| **`final` method** | Cannot be overridden by subclasses |
| **`final` class** | Cannot be extended (no inheritance) |
| **Telescoping Constructor** | Multiple constructors calling each other with increasing parameters |
| **Builder Pattern** | A creational design pattern that separates object construction from representation |
| **Deep Copy** | Creates a completely independent copy of an object |
| **Shallow Copy** | Copies only the reference; both variables point to the same memory |
| **Static Inner Class** | An inner class that belongs to the outer class (not to an instance); can be used without an outer class object |
| **Method Chaining** | Each method returns the object itself, allowing `obj.setA().setB().setC()` |
| **`build()` method** | The method in a Builder that creates and returns the final immutable object |

---

## 10. Exam Prep - Quick-Fire Facts

1. **Immutable class requires:** private final fields, no setters, deep copies (constructor + getters), final class.
2. **`final` on a reference** prevents reassignment, NOT modification of the object it points to.
3. **Builder pattern is for immutable classes** with many attributes.
4. **Builder is a static inner class** to avoid needing an outer class instance.
5. **Setters in Builder return `this`** for chaining.
6. **`build()` calls the outer class constructor** passing the builder object.
7. **StringBuilder in Java** is a real-world example of the Builder pattern.
8. **String in Java is immutable** -- every modification creates a new String.
9. **DTOs, config objects, constants** are common use cases for immutability.
10. **Telescoping constructors** = calling one constructor from another; not scalable.
11. **Access modifiers (least to most access):** private < default < protected < public.
12. **Double-check locking can still break** -- this is a common interview question (hint: `volatile` keyword).
13. **Dependency Inversion** makes Open-Closed and Single Responsibility automatically followed.
14. **Validation logic** in Builder goes inside the `build()` method.

### Comparison Table: Approaches to Creating Immutable Objects

| Approach | Type Safe? | Scalable? | Order-Free? | Maintainable? | Verdict |
|---|---|---|---|---|---|
| Single large constructor | Yes | No | No | Yes | Bad for many fields |
| Telescoping constructors | Yes | No | No | No | Exponential growth |
| HashMap<String, Object> | **No** | Yes | Yes | Yes | Rejected |
| Separate helper class | Yes | Yes | Yes | **No** (two classes) | Improvement |
| **Builder (inner class)** | **Yes** | **Yes** | **Yes** | **Yes** | **Best solution** |

---

## 11. Teacher's Special Insights

### Career & Interview Tips
- **"They will first ask you to implement double check locking and then they will ask you to break it."** -- Know both the solution and its failure modes. This is a very common interview progression.
- **"If you are able to follow the last SOLID principle (Dependency Inversion), then the first two will automatically be followed."** -- Focus on Dependency Inversion as the cornerstone.
- **"Just by the name Builder, whoever is going through your code would know what is the purpose."** -- Design patterns act as a shared vocabulary. Naming matters.
- **"You don't need to explain anyone explicitly. You can just mention that it is a builder. This much should be sufficient."** -- Design patterns are a communication shortcut in industry.

### Study Tips
- **"Best time to go through the notes is right after the class."** -- Reinforces concepts while context is fresh.
- **"45 minutes daily on LLD is more than sufficient."** -- 15 min reading notes + 15-20 min implementing code.
- **"If you don't create it, you will forget it."** -- Always implement the code yourself; don't just read.
- **"If you read the same notes after one month, you will feel they are very complicated."** -- Time degrades understanding without reinforcement.

### Thumb Rules
- **Never use `Object` class as a generic type** unless absolutely necessary. It sacrifices type safety.
- **Any class that is immutable should never expose its internal mutable references** -- always deep copy.
- **Setters destroy immutability.** If you see setters in what's supposed to be an immutable class, that's a bug.
- **The process of reaching the solution might be complicated, but the final solution is simple.** -- Builder pattern is just: inner class + same attributes + setters returning `this` + `build()` method.

### Real-World Analogies
- **StringBuilder** is the canonical real-world example of Builder pattern in Java.
- **String Pool** demonstrates immutability in practice -- strings are shared because they can never change.

---

## Homework from This Class
1. **Find scenarios where double-check locking Singleton breaks**, and how to fix it.
2. **Find a way to prevent internal methods from mutating lists** in an immutable class (hint: `Collections.unmodifiableList()`).
3. **Implement the Builder pattern** from scratch for a class with multiple attributes.
