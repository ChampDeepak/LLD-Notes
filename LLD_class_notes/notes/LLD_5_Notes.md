# LLD Lecture 5 - Detailed Notes

## Topics Covered
1. Interface Segregation Principle (ISP) - 4th SOLID principle
2. Dependency Inversion Principle (DIP) - 5th SOLID principle
3. Dependency Injection vs Inversion of Control
4. Static keyword deep-dive (variables, methods)
5. Singleton Design Pattern (eager, lazy, thread-safe, double-checked locking, volatile)

---

## 1. Interface Segregation Principle (ISP)

### Recap: The Bird Example (from earlier classes)

The teacher starts by revisiting the Bird ecosystem designed in earlier lectures:

```
Bird (abstract class)
  |-- Hen
  |-- Eagle
  |-- Kiwi
  |-- Penguin
  ...
```

- `Bird` is abstract -- you cannot create a generic "Bird" object; every bird is a specific type.
- `fly()` was moved **out** of the Bird class into a **Flyable** interface because flying is not intrinsic to all birds (e.g., Hen, Kiwi cannot fly). This was the Liskov Substitution fix.

### The New Problem: Fat Interfaces

Suppose we add more flight-related methods to the Flyable interface:

```java
interface Flyable {
    void fly();
    void flapWings();  // <-- problem!
}
```

**Why is this a problem?**

- Non-bird entities can also fly: SU-30 (fighter jet), Superman, drones.
- These entities can implement `fly()`, but `flapWings()` makes **no sense** for them.
- If they implement `Flyable`, they are **forced** to implement `flapWings()` -- leading to empty methods or exceptions.
- This is similar to the Liskov Substitution violation: objects are forced to implement irrelevant methods.

### The Fix: Segregate the Interface

Split into lean, specific interfaces:

```java
interface Flyable {
    void fly();
}

interface FlapWingable {
    void flapWings();
}
```

- **Eagle** implements both `Flyable` and `FlapWingable`.
- **SU-30** implements only `Flyable`.
- **Superman** implements only `Flyable`.

> **One class can implement multiple interfaces.** This gives maximum flexibility.

### ISP Definition

> **Interfaces should not be generic. They should be very specific and very lean.**
>
> If an interface is generic (has many methods), any class implementing it is forced to implement ALL methods -- some of which may be irrelevant.

### Key Insight from Teacher

> "If your interfaces are following the Single Responsibility Principle strictly, Interface Segregation will automatically be followed."

| Aspect | Generic Interface | Segregated Interfaces |
|---|---|---|
| Number of methods | Many | Few (ideally one) |
| Flexibility | Low -- all implementors need all methods | High -- pick only what you need |
| Cross-domain reuse | Poor -- tied to original domain | Excellent -- any domain can implement |
| SRP compliance | Weak | Strong |

---

## 2. Dependency Inversion Principle (DIP)

### The Teacher's Design Journey (V0 -> V1 -> V2 -> Final)

This is the most important part -- the teacher walks through iterative design improvements step by step.

#### V0: Starting Point (from previous class)

HR Management System requirements:
- Support full-time employees and interns
- Save employee data to a file system

```
Employee (abstract)
  |-- FullTimeEmployee
  |-- InternEmployee

EmployeeRepo (stores in-memory employee data)
```

Originally, `save()` was in the Employee class itself. That violated SRP because Employee would change if the format or database changed.

#### V1: Move save() to EmployeeRepo

The `save()` method was moved to `EmployeeRepo`. But even there, `save()` had multiple reasons to change:
- Change in serialization format
- Change in storage location/type

So it was split into:
- **Serializer** class -- converts Employee to a formatted string
- **FileStorageService** class -- writes/reads data from files

#### V2: The Problem with Concrete Dependencies

Inside `EmployeeRepo.save()`:

```java
public void save(Employee employee) {
    Serializer serializer = new Serializer();
    FileStorageService fss = new FileStorageService();
    String data = serializer.serialize(employee);
    fss.save(data);
}
```

**Problem:** If we upgrade from file storage to SQL, we must change `save()` -- the line creating `FileStorageService` must become `SQLStorageService`. This is a **hard dependency** on a concrete class.

> **Teacher's Rule:** "If inside a class you are creating an object of any other concrete class, that is a hard dependency. It should not be there."

#### V3 (Final): Introduce Abstraction via Interface

```java
interface IDatabaseService {
    void save(Employee employee);
    Employee searchById(int id);
    Employee searchByName(String name);
    List<Employee> retrieveAll();
}
```

Concrete implementations:

```java
class FileStorageService implements IDatabaseService {
    // file-based implementations of all methods
}

class SQLStorageService implements IDatabaseService {
    // SQL-based implementations of all methods
}
```

**EmployeeRepo now depends on the abstraction:**

```java
class EmployeeRepo {
    private IDatabaseService databaseService;

    public EmployeeRepo(IDatabaseService databaseService) {
        this.databaseService = databaseService;
    }

    public void save(Employee employee) {
        databaseService.save(employee);
    }
}
```

- `EmployeeRepo` never knows if it is using `FileStorageService` or `SQLStorageService`.
- The appropriate object is passed through the **constructor**.
- To switch databases: just pass a different object -- **no code changes** inside `EmployeeRepo`.

### Before vs After Dependency Diagram

**Before (V2):**
```
EmployeeRepo ---depends on---> FileStorageService (concrete)
```
Hard dependency. Any change to storage = change in EmployeeRepo.

**After (V3 -- Final):**
```
EmployeeRepo ---depends on---> IDatabaseService (interface/abstraction)
                                     ^              ^
                                     |              |
                            FileStorageService  SQLStorageService
```
Both EmployeeRepo and FileStorageService depend on the abstraction. Dependencies are **inverted**.

### DIP Definition

> **High-level modules should not depend on low-level modules. Both should depend on abstractions.**
>
> Abstractions should not depend on details. Details should depend on abstractions.

### Interface vs Abstract Class for IDatabaseService?

Teacher's answer:
- Technically either works.
- **In practice, use an interface** because:
  - No state is maintained (it is a service -- just a collection of methods).
  - Implementing classes can still extend other parent classes if needed.

---

## 3. Dependency Injection, Dependency Inversion, and Inversion of Control

### The Three Related but Different Concepts

This is one of the **most frequently asked interview questions** per the teacher.

| Concept | What It Is | Example |
|---|---|---|
| **Dependency Inversion** | A SOLID *principle*: classes should depend on abstractions, not concrete classes | `EmployeeRepo` depends on `IDatabaseService` interface, not `FileStorageService` |
| **Dependency Injection** | A *technique* for passing the dependency into a class | Passing the `IDatabaseService` object via the constructor |
| **Inversion of Control (IoC)** | A *framework mechanism*: the framework manages the lifecycle of all objects | Spring Boot manages bean creation, topological sorting of dependencies |

### Types of Dependency Injection

1. **Constructor Injection** -- pass via constructor (preferred for immutable objects)
2. **Setter Injection** -- pass via setter method
3. **Method Injection** -- pass as argument to a specific method

> **Teacher's Advice:** Constructor injection is preferred because:
> - The dependency is available to ALL methods in the class (not just one).
> - Supports immutable objects (no state change after construction).
> - Setter injection breaks immutability.

### Why IoC / Frameworks Are Needed

In a large application with thousands of classes:
- Class E depends on A and C
- A depends on B
- C depends on B

To create objects manually, you must perform **topological sorting** of the dependency graph. For 500+ classes, this is impractical.

**Solution:** Offload object lifecycle management to a framework like **Spring Boot**.
- Spring reads dependencies from configuration (XML/annotations).
- Spring does the topological sort internally.
- Developer never manually manages object creation order.

> This is called **Inversion of Control** -- the control of object creation is no longer in the developer's hands; it is in the framework's hands.

---

## 4. Why Not Just Use Static Methods?

### The Teacher's Exploration

The teacher systematically explores: "If we only need one object, why not make everything static?"

**What is static?**

| Feature | Static | Non-Static |
|---|---|---|
| **Variable** | Single copy shared across ALL objects; stored separately in memory | Each object gets its own copy |
| **Method** | Called via `ClassName.method()`; no object needed | Called via `object.method()` |
| **Access** | Static methods can only access static variables | Non-static methods can access both |
| **Inner Class** | Can exist without outer class instance | Needs outer class instance |

### Why `main` is static

- To call a method, you need an object.
- To create an object, you need to call a constructor from some method.
- This is a chicken-and-egg problem.
- `main` is made static to break this loop -- it can run without any object.

### Examples of static methods in Java

- `Math.min()`, `Math.max()`, `Math.sqrt()`
- `Collections.sort()`

### Why Static Fails for Our Use Case

If `FileStorageService` methods are all static:

```java
// In EmployeeRepo:
FileStorageService.save(employee);  // Hard-coded class name!
```

**Problems:**
1. **Abstraction is lost** -- the client must know the exact class name.
2. **Polymorphism is impossible** -- static methods cannot be overridden.
3. **Dependency Inversion is broken** -- you are back to depending on a concrete class.
4. **Cannot swap implementations** at runtime (file -> SQL -> Mongo).

> **Teacher's Key Point:** "Static methods are not available for polymorphism. You cannot override them. If only one implementation can exist, you lose all flexibility."

### Can an Interface Have Static Methods?

Yes, but:
- A static method in an interface can only have **one implementation** (in the interface itself).
- It **cannot be overridden** in implementing classes.
- Therefore, you cannot have different implementations for FileStorage vs SQL.

---

## 5. Singleton Design Pattern

### The Problem

For classes like `FileStorageService`, `Serializer`, `TaxCalculationUtility`, `Logger`, `Config`:
- They hold no unique per-instance state.
- They provide functionality (methods) only.
- **Only one object is ever needed.**

We want to restrict object creation to exactly one instance while still supporting polymorphism and abstraction.

### The Teacher's Step-by-Step Design Journey

#### Step 1: Regular Class (Problem)

```java
class FileStorageService {
    String filePath;

    public FileStorageService() {
        this.filePath = "abcd";
    }
}

// Client code:
FileStorageService f1 = new FileStorageService();
FileStorageService f2 = new FileStorageService();
System.out.println(f1.hashCode()); // different
System.out.println(f2.hashCode()); // different -- two objects!
```

Two different hash codes = two different objects. Not what we want.

#### Step 2: Private Constructor

```java
class FileStorageService {
    private FileStorageService() { }
}
```

Now nobody outside the class can call `new FileStorageService()`. But... nobody can create even ONE object.

#### Step 3: Add a Static Instance + Getter

```java
class FileStorageService {
    private static FileStorageService instance = new FileStorageService();

    private FileStorageService() { }

    public static FileStorageService getInstance() {
        return instance;
    }
}
```

- Constructor is private -- no external creation.
- `getInstance()` is static -- callable without an object (solves chicken-and-egg problem).
- `instance` is static -- accessible from the static method.

**This works!** Both calls to `getInstance()` return the same hash code.

**But:** This is **eager loading** -- the object is created at class-load time, even if never used. Wastes memory and CPU if the constructor is heavy.

#### Step 4: Lazy Loading

```java
class FileStorageService {
    private static FileStorageService instance = null;

    private FileStorageService() { }

    public static FileStorageService getInstance() {
        if (instance == null) {
            instance = new FileStorageService();
        }
        return instance;
    }
}
```

Object is created **only when first requested**. Saves resources.

**But:** Not thread-safe!

#### Step 5: Thread-Safety Problem

Two threads T1 and T2:

| Time | T1 | T2 | instance |
|---|---|---|---|
| t1 | Checks `instance == null` -> true | -- | null |
| t2 | Enters if block, pauses | -- | null |
| t3 | -- | Checks `instance == null` -> true | null |
| t4 | -- | Enters if block | null |
| t5 | Creates object A | -- | A |
| t6 | -- | Creates object B | B |

**Result:** Two different objects created! Singleton broken.

#### Step 6: Synchronized Method (Naive Fix)

```java
public static synchronized FileStorageService getInstance() {
    if (instance == null) {
        instance = new FileStorageService();
    }
    return instance;
}
```

Thread-safe, but the **entire method** is locked. Even when instance already exists (99% of calls), every thread must wait for the lock. **Terrible performance.**

#### Step 7: Synchronized Block (Partial Fix)

```java
public static FileStorageService getInstance() {
    if (instance == null) {
        synchronized (FileStorageService.class) {
            instance = new FileStorageService();
        }
    }
    return instance;
}
```

**Still broken!** T1 checks null, enters if, pauses before synchronized block. T2 checks null (still null), enters if, waits at synchronized block. T1 finishes. T2 enters synchronized block and creates ANOTHER object.

#### Step 8: Double-Checked Locking (Correct!)

```java
public static FileStorageService getInstance() {
    if (instance == null) {                          // First check (no lock)
        synchronized (FileStorageService.class) {    // Lock
            if (instance == null) {                  // Second check (with lock)
                instance = new FileStorageService();
            }
        }
    }
    return instance;
}
```

**How it works:**
- First `if`: fast path -- if instance exists, return immediately (no lock overhead).
- `synchronized`: only one thread enters at a time.
- Second `if`: after acquiring lock, re-check because another thread may have already created the instance.

| Time | T1 | T2 | instance |
|---|---|---|---|
| t1 | First check: null -> true | -- | null |
| t2 | Enters synchronized block | -- | null |
| t3 | Pauses | First check: null -> true | null |
| t4 | -- | Waits at synchronized (T1 holds lock) | null |
| t5 | Second check: null -> true. Creates object. | Waiting... | Object A |
| t6 | Exits synchronized block | Enters synchronized block | Object A |
| t7 | -- | Second check: NOT null -> skips creation | Object A |
| t8 | -- | Returns existing instance | Object A |

**Singleton preserved!**

#### Step 9: The Volatile Keyword (Final Piece)

```java
private static volatile FileStorageService instance = null;
```

**Why volatile is needed:**

Object creation (`instance = new FileStorageService()`) has 3 steps:
1. **Allocate memory** for the object
2. **Initialize** the object (run constructor, set fields)
3. **Link the reference** (`instance` now points to the allocated memory)

**Without volatile**, two problems:
1. **Instruction reordering:** The JVM/CPU may reorder steps as 1-3-2. Thread T2 could see a non-null `instance` that is only partially constructed (memory allocated, reference linked, but fields not initialized).
2. **CPU cache inconsistency:** Threads maintain their own CPU cache. T1 may create the object in its cache but not write to main memory (RAM). T2 reads from RAM, sees null, creates another object.

**With volatile:**
- **No reordering** -- all 3 steps complete atomically before any other thread can see the reference.
- **No caching** -- all reads and writes go directly to main memory (RAM). Changes are immediately visible to all threads.

### Final Complete Singleton Implementation

```java
class FileStorageService {
    private static volatile FileStorageService instance = null;

    private FileStorageService() {
        // initialization logic
    }

    public static FileStorageService getInstance() {
        if (instance == null) {                            // 1st check (no lock)
            synchronized (FileStorageService.class) {      // acquire lock
                if (instance == null) {                    // 2nd check (with lock)
                    instance = new FileStorageService();
                }
            }
        }
        return instance;
    }
}
```

### Alternative Singleton Implementations (mentioned briefly)

| Approach | Thread-Safe? | Lazy? | Locking Needed? |
|---|---|---|---|
| Eager initialization | Yes | No | No |
| Double-checked locking + volatile | Yes | Yes | Only on first call |
| **Static inner class (Bill Pugh)** | Yes | Yes | **No** |
| **Enum singleton** | Yes | No | **No** |

> **Teacher's note:** Static inner class and Enum are considered the **best** ways to implement Singleton. Both are inherently thread-safe without locking.

---

## 6. Summary Table: All 5 SOLID Principles

| # | Principle | One-Line Summary | Most Important? |
|---|---|---|---|
| S | Single Responsibility | A class should have only one reason to change | **Yes** -- if followed, others follow automatically |
| O | Open-Closed | Open for extension, closed for modification | Follows from SRP |
| L | Liskov Substitution | Child should fully substitute parent without unexpected behavior | -- |
| I | Interface Segregation | Interfaces should be lean and specific, not fat and generic | Easiest principle |
| D | Dependency Inversion | Depend on abstractions, not concrete classes | **Yes** -- critical for large-scale apps |

> **Teacher's Ranking for Interviews:** SRP and DIP are the most important. If you are optimizing interview prep, focus on these two.

---

## 7. Interview Questions

### Frequently Asked (Teacher explicitly flagged these)

1. **What is Dependency Inversion? What is Dependency Injection? What is Inversion of Control? How are they different?**
   - DI**P** = principle (depend on abstractions)
   - DI = technique (pass dependency via constructor/setter/method)
   - IoC = framework mechanism (Spring manages object lifecycles)

2. **What is the Singleton Design Pattern? How do you implement it?**
   - Private constructor, static volatile instance, public static getInstance(), double-checked locking.

3. **Why is double-checked locking needed? Why not just synchronize the method?**
   - Full sync is slow. Double-check avoids locking after instance is created (99% of calls).

4. **Why is the volatile keyword needed in Singleton?**
   - Prevents instruction reordering (partial construction visibility).
   - Ensures main-memory visibility (no stale cache reads).

5. **What is the difference between static methods and Singleton?**
   - Static: no polymorphism, no abstraction, cannot implement interfaces.
   - Singleton: supports polymorphism, abstraction, dependency inversion.

6. **Can a static method access non-static variables?**
   - No. "Non-static variable referred from static context" error.

7. **Why use interface instead of abstract class for IDatabaseService?**
   - No state to maintain, just method contracts. Allows implementors to extend other classes.

8. **What are the 5 SOLID principles?**

9. **What is Interface Segregation Principle?**
   - Interfaces should be lean. Classes should not be forced to implement irrelevant methods.

### Potential Viva/Exam Questions

10. **Why can you not override static methods?**
    - Static methods belong to the class, not to instances. Method dispatch is at compile time, not runtime. No polymorphism.

11. **What is eager loading vs lazy loading in Singleton?**
    - Eager: instance created at class-load time.
    - Lazy: instance created on first `getInstance()` call.

12. **What is the Bill Pugh Singleton (static inner class)?**
    - Uses a static inner class to hold the instance. Loaded lazily by the JVM. Thread-safe without synchronization.

13. **Can Singleton be broken? How?**
    - Reflection, serialization/deserialization, cloning. (Enum singleton is immune to all of these.)

---

## 8. Design Decisions Summary

| Decision | Why This Way | Why NOT the Other Way |
|---|---|---|
| Interface over abstract class for services | No state needed; just method contracts | Abstract class would restrict single inheritance |
| Constructor injection over setter injection | Works with immutable objects; dependency available to all methods | Setter allows state change after construction |
| Lazy loading over eager loading for Singleton | Saves memory/CPU when instance may not be used | Eager is simpler but wastes resources |
| Double-checked locking over full method sync | 99% of calls skip the lock entirely | Full sync blocks ALL threads even when instance exists |
| Volatile keyword on instance | Prevents partial construction and cache inconsistency | Without it, other threads may see half-constructed objects |
| Singleton over fully-static class | Preserves polymorphism, abstraction, and DIP | Static kills all flexibility and testability |

---

## 9. Teacher's Special Insights

### On the Design Process

> "If you don't know any concept of LLD, you'll create one class with everything in it. When you learn functions, you create multiple functions. When requirements grow, you segregate into classes. **This comes through practice in multiple iterations.** Even after learning all concepts, your first iteration may not be perfect."

### On When to Write Code

> "You haven't yet created any constructor. We are just discussing the design. **We will only start implementation when we are clear with the design.** Spend significant time designing so you invest less time writing, testing, and maintaining code."

### On Mock Classes and Parallel Development

> "When you have an abstraction, you can write **mock classes** for testing. You don't need to wait for the real implementations. This makes **parallel development possible** -- the contract (interface) is the only thing needed."

### On the Real World

> "In Java, you will never manage object lifecycles manually. You offload that to Spring Boot. Otherwise all your time is wasted in topological sorting of dependencies."

### Career Tip on Interviews

> "Out of all 5 SOLID principles, if you're optimizing your chances in interviews, **Dependency Inversion + Injection + IoC** is going to be one of the most important things."

### Thumb Rules

1. **Never create objects of concrete classes inside another class** -- always use abstraction.
2. **Interfaces should ideally have one method** -- maximum flexibility.
3. **If SRP is followed strictly, most other SOLID principles follow automatically.**
4. **Use interfaces (not abstract classes) for services** -- they have no state.
5. **For Singleton: private constructor + static volatile instance + double-checked locking.**

---

## 10. Quick-Fire Exam Prep

### True/False

| Statement | Answer |
|---|---|
| Static methods can access non-static variables | **False** |
| Non-static methods can access static variables | **True** |
| Static variables are stored once regardless of number of objects | **True** |
| An interface can have static methods | **True** (but cannot be overridden in implementing classes) |
| A class can implement multiple interfaces | **True** |
| Eager loading Singleton is NOT thread-safe | **False** (it IS thread-safe) |
| Lazy loading Singleton (without sync) is thread-safe | **False** |
| Volatile prevents CPU cache inconsistency | **True** |
| Dependency Injection and Dependency Inversion are the same thing | **False** |
| IoC is a principle | **False** (it is a framework mechanism) |

### Fill in the Blanks

1. Making a constructor _______ prevents object creation outside the class. --> **private**
2. The _______ keyword ensures a variable is read/written directly from/to main memory. --> **volatile**
3. _______ loading creates the instance at class-load time. --> **Eager**
4. _______ loading creates the instance only when first requested. --> **Lazy**
5. The technique of passing dependencies via constructor is called _______. --> **Dependency Injection**
6. Spring Boot implements _______ by managing object lifecycles. --> **Inversion of Control (IoC)**

### Key Definitions

| Term | Definition |
|---|---|
| **Singleton Pattern** | A design pattern that restricts a class to exactly one instance and provides global access to it |
| **Double-Checked Locking** | A thread-safety technique: check once without lock, acquire lock, check again, then create |
| **Volatile** | A Java keyword ensuring variables are read/written to main memory (no CPU cache optimization) |
| **Dependency Inversion** | High-level modules should not depend on low-level modules; both should depend on abstractions |
| **Dependency Injection** | Passing a dependency (as an abstraction) into a class via constructor, setter, or method |
| **Inversion of Control** | Delegating object lifecycle management to a framework (e.g., Spring) |
| **Interface Segregation** | Interfaces should be small and specific; no class should be forced to implement irrelevant methods |

---

## 11. Code Examples

### Complete Singleton (Thread-Safe, Lazy, Double-Checked Locking)

```java
public class FileStorageService implements IDatabaseService {
    private static volatile FileStorageService instance = null;
    private String filePath;

    // Private constructor -- cannot be called from outside
    private FileStorageService() {
        this.filePath = "/data/employees.txt";
    }

    // Static factory method -- the ONLY way to get an instance
    public static FileStorageService getInstance() {
        if (instance == null) {                             // 1st check (fast, no lock)
            synchronized (FileStorageService.class) {       // Acquire class-level lock
                if (instance == null) {                     // 2nd check (under lock)
                    instance = new FileStorageService();
                }
            }
        }
        return instance;
    }

    @Override
    public void save(Employee employee) {
        // write to file
    }
}
```

### Dependency Inversion with Constructor Injection

```java
// Abstraction
interface IDatabaseService {
    void save(Employee employee);
    Employee searchById(int id);
}

// Concrete implementation 1
class FileStorageService implements IDatabaseService {
    public void save(Employee employee) { /* file logic */ }
    public Employee searchById(int id) { /* file logic */ }
}

// Concrete implementation 2
class SQLStorageService implements IDatabaseService {
    public void save(Employee employee) { /* SQL logic */ }
    public Employee searchById(int id) { /* SQL logic */ }
}

// High-level module -- depends ONLY on abstraction
class EmployeeRepo {
    private IDatabaseService databaseService;  // abstraction, not concrete

    // Dependency injected via constructor
    public EmployeeRepo(IDatabaseService databaseService) {
        this.databaseService = databaseService;
    }

    public void save(Employee employee) {
        databaseService.save(employee);  // does not know which implementation
    }
}

// Client code -- decides which implementation to use
public class Main {
    public static void main(String[] args) {
        IDatabaseService storage = new FileStorageService(); // or SQLStorageService
        EmployeeRepo repo = new EmployeeRepo(storage);
        repo.save(someEmployee);
    }
}
```

### Interface Segregation Example

```java
// BAD: Fat interface
interface Flyable {
    void fly();
    void flapWings();  // irrelevant for jets, superman
}

// GOOD: Segregated interfaces
interface Flyable {
    void fly();
}

interface FlapWingable {
    void flapWings();
}

// Eagle uses both
class Eagle extends Bird implements Flyable, FlapWingable {
    public void fly() { /* ... */ }
    public void flapWings() { /* ... */ }
}

// SU-30 uses only Flyable
class SU30 implements Flyable {
    public void fly() { /* jet propulsion */ }
}
```

### Static vs Non-Static Demonstration

```java
class Demo {
    int value;  // non-static

    Demo(int value) {
        this.value = value;
    }

    // WORKS: static method with only passed-in values
    public static int max(int a, int b) {
        return a > b ? a : b;
    }

    // FAILS: static method trying to use non-static variable
    public static boolean compare(int x) {
        return this.value > x;  // ERROR: non-static variable from static context
    }
}
```

---

*These notes cover the complete content of LLD Lecture 5. All design journeys, teacher insights, interview tips, and code examples have been captured.*
