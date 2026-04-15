# LLD Lecture 11 — Detailed Notes
## Topic: Strategy Design Pattern | Decoupling Behavior from Entity | Observer Pattern Introduction

---

## 1. Recap: Structural Design Patterns Covered So Far

Before diving into new content, the teacher recaps the **structural design patterns** studied so far:

| Pattern | Purpose | Key Idea |
|---------|---------|----------|
| **Proxy** | Control access to an object | Wrapper that intercepts calls before forwarding |
| **Decorator** | Add new behavior dynamically | Wrapper that adds logic before/after delegating |
| **Adapter** | Make incompatible interfaces work together | Wrapper that translates one interface to another |
| **Flyweight** | Optimize memory usage | Share common state across many objects |

**Teacher's Insight:** Proxy, Decorator, and Adapter are all essentially **wrappers around existing classes**. They allow you to modify/extend legacy code that has already been tested — without changing the original code. Flyweight is different; it solves a **memory optimization** problem.

---

## 2. The Design Journey — Building a List Class (V0 to Final)

### The Problem Statement

> A client needs a `MyList` class (integer-only) injected through their constructor. The class must support:
> - `add(int x)` — append element at end
> - `search(int x)` — return the smallest index where `x` is found
> - `printList()` — print all elements vertically (newline after each element)
> - `sort()` — sort in ascending order

The client code looks like:
```java
class Client {
    IList l;

    Client(IList l) {
        this.l = l;
    }

    void doWork() {
        l.add(5);
        l.add(3);
        l.sort();
        l.printList();
    }
}
```

---

### V0 — The Naive Concrete Class

**Approach:** Create a single class `MyList` that wraps an `ArrayList<Integer>`.

```java
class MyList {
    ArrayList<Integer> list = new ArrayList<>();

    void add(int x) {
        list.add(x);            // delegates to ArrayList.add()
    }

    void sort() {
        Collections.sort(list); // delegates to Collections.sort() (TimSort)
    }

    void printList() {
        for (int val : list) {
            System.out.println(val);  // vertical print
        }
    }

    int search(int x) {
        for (int i = 0; i < list.size(); i++) {
            if (list.get(i) == x) return i;   // linear search — first index
        }
        return -1;
    }
}
```

**Teacher's Key Point:** This class is essentially a **wrapper** around `ArrayList`. Most methods just delegate — `add` delegates to `ArrayList.add()`, `sort` delegates to `Collections.sort()`. Only `search` and `printList` have custom logic.

**Important Clarification:** The teacher explicitly says — *"Time complexity is part of implementation, not design. First give the design, then worry about optimization."*

---

### V0 — What's Wrong? Evaluating Against SOLID

The teacher asks students to evaluate this design against SOLID principles:

| Principle | Violated? | Explanation |
|-----------|-----------|-------------|
| **SRP** | Borderline OK | The class is mostly a thin wrapper; real logic lives in ArrayList and Collections |
| **OCP** | Not evaluated yet | No extensions needed at this stage |
| **LSP** | N/A | No inheritance hierarchy yet |
| **ISP** | N/A | No interfaces yet |
| **DIP** | **YES - VIOLATED** | The client depends on a **concrete class** `MyList`, not an abstraction |

**The Critical Missing Piece: Dependency Inversion Principle (DIP)**

> Two classes should **never** depend directly on each other. Dependencies should always be on **abstractions** (interfaces).

**Why does this matter?** If we want to:
- Switch internal implementation from `ArrayList` to a raw array (for memory optimization)
- Replace `Collections.sort()` (TimSort) with a custom sorting algorithm
- Change the dynamic resizing behavior of the array

...we would need to create a **completely new class** with different implementations. Without an interface, the client code must change.

---

### V1 — Adding the Interface (DIP Fix)

```java
interface IList {
    void add(int x);
    int search(int x);
    void printList();
    void sort();
}

class MyList implements IList {
    ArrayList<Integer> list = new ArrayList<>();

    @Override
    void add(int x) { list.add(x); }

    @Override
    void sort() { Collections.sort(list); }

    @Override
    void printList() {
        for (int val : list) {
            System.out.println(val);
        }
    }

    @Override
    int search(int x) {
        for (int i = 0; i < list.size(); i++) {
            if (list.get(i) == x) return i;
        }
        return -1;
    }
}
```

Now the client depends on `IList`, not `MyList`. If we create `MyList2` with a different internal structure, the client code **does not change**.

**Teacher's Rule:** *"Whenever you are designing anything — especially simple single-use-case scenarios — you MUST have an interface. This is non-negotiable."*

---

### New Requirements — The Behavior Variation Problem

The client now says:
1. **Printing behavior** should be configurable: **vertical** (newline after each element) OR **horizontal** (space-separated, newline at end)
2. The print behavior is specified **at object creation time** and remains consistent for the lifetime of that object

Then the teacher introduces **four different clients** with varying needs:

| Client | Print Behavior | Data Pattern | Best Sorting Algorithm |
|--------|---------------|--------------|----------------------|
| **Client 1** | Configurable (V or H) | Random integers | Merge Sort / Quick Sort / TimSort |
| **Client 2** | Always Horizontal | Only 0, 1, 2 | Count Sort or Dutch National Flag (O(n)) |
| **Client 3** | Always Vertical | No specific pattern | TimSort |
| **Client 4** | Configurable (V or H) | Almost sorted | Insertion Sort |

**The Core Problem:**
- All clients need the **same set of methods** (`add`, `search`, `printList`, `sort`)
- All clients use **integer lists only**
- But the **behavior of `printList()` and `sort()` varies** from client to client
- Different behaviors **overlap across clients** in unpredictable ways

---

### Why Subclassing Fails

The teacher recalls the **Bird problem from Lecture 2** to explain why inheritance won't work here.

**Scenario:** If we create subclasses based on print behavior:
```
IList
├── VerticalPrintList    (vertical print + ???sort)
└── HorizontalPrintList  (horizontal print + ???sort)
```

**Problem:** Within `VerticalPrintList`, different clients need different sorting algorithms. We'd need:
```
IList
├── VerticalPrintMergeSortList
├── VerticalPrintTimSortList
├── VerticalPrintInsertionSortList
├── HorizontalPrintCountSortList
├── HorizontalPrintMergeSortList
└── ... (combinatorial explosion!)
```

**Teacher's Key Insight:** *"Different methods are common amongst different sets of classes. If you segregate based on one feature, you'll repeat code for another feature. This is exactly the Bird problem — fly behavior groups birds one way, hunt behavior groups them differently."*

With **2 print strategies x 4 sort strategies**, you'd already need **8 subclasses**. Add one more varying behavior and it becomes **unmanageable**.

---

### V2 (Final) — The Strategy Design Pattern

**Core Idea:** Instead of creating subclasses for each combination of behaviors, **inject the behavior as an object through the constructor**.

#### Step 1: Define Strategy Interfaces

```java
// Strategy for printing
interface PrintStrategy {
    void print(List<Integer> list);
}

// Strategy for sorting
interface SortStrategy {
    void sort(List<Integer> list);
}
```

#### Step 2: Implement Concrete Strategies

```java
// Print strategies
class VerticalPrintStrategy implements PrintStrategy {
    @Override
    void print(List<Integer> list) {
        for (int val : list) {
            System.out.println(val);
        }
    }
}

class HorizontalPrintStrategy implements PrintStrategy {
    @Override
    void print(List<Integer> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.print(list.get(i));
            if (i < list.size() - 1) System.out.print(" ");
        }
        System.out.println();
    }
}

// Sort strategies
class TimSortStrategy implements SortStrategy {
    @Override
    void sort(List<Integer> list) {
        Collections.sort(list);
    }
}

class CountSortStrategy implements SortStrategy {
    @Override
    void sort(List<Integer> list) {
        // Count sort for 0, 1, 2
        int[] count = new int[3];
        for (int val : list) count[val]++;
        int idx = 0;
        for (int i = 0; i < 3; i++) {
            while (count[i]-- > 0) list.set(idx++, i);
        }
    }
}

class InsertionSortStrategy implements SortStrategy {
    @Override
    void sort(List<Integer> list) {
        for (int i = 1; i < list.size(); i++) {
            int key = list.get(i);
            int j = i - 1;
            while (j >= 0 && list.get(j) > key) {
                list.set(j + 1, list.get(j));
                j--;
            }
            list.set(j + 1, key);
        }
    }
}
```

#### Step 3: MyList Uses Strategies (Composition Over Inheritance)

```java
class MyList implements IList {
    private List<Integer> list = new ArrayList<>();
    private PrintStrategy printStrategy;
    private SortStrategy sortStrategy;

    MyList(PrintStrategy printStrategy, SortStrategy sortStrategy) {
        this.printStrategy = printStrategy;
        this.sortStrategy = sortStrategy;
    }

    @Override
    void add(int x) { list.add(x); }

    @Override
    int search(int x) {
        for (int i = 0; i < list.size(); i++) {
            if (list.get(i) == x) return i;
        }
        return -1;
    }

    @Override
    void printList() {
        printStrategy.print(list);   // DELEGATED to strategy
    }

    @Override
    void sort() {
        sortStrategy.sort(list);     // DELEGATED to strategy
    }
}
```

#### Step 4: Factory Creates the Right Object

```java
class ListFactory {
    static IList createList(String printType, String sortType) {
        PrintStrategy ps;
        SortStrategy ss;

        // Determine print strategy
        if (printType.equals("vertical")) {
            ps = new VerticalPrintStrategy();
        } else {
            ps = new HorizontalPrintStrategy();
        }

        // Determine sort strategy
        switch (sortType) {
            case "timsort":    ss = new TimSortStrategy(); break;
            case "countsort":  ss = new CountSortStrategy(); break;
            case "insertion":  ss = new InsertionSortStrategy(); break;
            default:           ss = new TimSortStrategy(); break;
        }

        return new MyList(ps, ss);
    }
}
```

#### How Each Client Gets Their Object

```java
// Client 1: Configurable print + MergeSort/TimSort
IList list1 = ListFactory.createList("vertical", "timsort");

// Client 2: Horizontal print + CountSort (only 0,1,2)
IList list2 = ListFactory.createList("horizontal", "countsort");

// Client 3: Vertical print + TimSort
IList list3 = ListFactory.createList("vertical", "timsort");

// Client 4: Configurable print + InsertionSort (almost sorted data)
IList list4 = ListFactory.createList("horizontal", "insertion");
```

**The client is completely agnostic** — they don't know if there is 1 list class or 1000. They only know the `IList` interface.

---

## 3. Strategy Design Pattern — Formal Definition

> **Strategy Pattern** allows you to define a family of algorithms, encapsulate each one as a separate class, and make them interchangeable. The algorithm varies independently from the clients that use it.

### Structure

```
                  +------------------+
                  |    Context       |   (MyList)
                  |  - strategy: I   |
                  |  + operation()   |----delegates to----> strategy.execute()
                  +------------------+
                          |
                  uses interface
                          |
                  +------------------+
                  |   IStrategy      |   (PrintStrategy / SortStrategy)
                  |  + execute()     |
                  +------------------+
                    /           \
        +-----------+     +-----------+
        | ConcreteA |     | ConcreteB |
        +-----------+     +-----------+
```

### When to Use Strategy Pattern

| Scenario | Example |
|----------|---------|
| Multiple algorithms for the same task | Different sorting algorithms |
| Behavior varies per object, not per class | Same `MyList` class, different print behaviors |
| Avoid combinatorial explosion of subclasses | 2 behaviors x 3 variants each = 6 subclasses vs 5 strategy classes |
| Behavior is selected at runtime (object creation) | Client specifies behavior via factory |
| You want to add new behaviors without modifying existing code | Add new `SortStrategy` implementation = OCP satisfied |

---

## 4. Key Design Decisions and Why

| Decision | Why |
|----------|-----|
| Interface over concrete class | DIP — client should not depend on implementation details |
| Strategy over subclassing | Avoids combinatorial explosion; behaviors overlap across different client needs |
| Factory for object creation | Client should not call constructors of concrete classes; factory decides internals |
| Behavior set at creation time, not per method call | Consistency — once created, object behavior is fixed |
| One class with injected strategies vs. many subclasses | Composition over inheritance; much more flexible and extensible |

---

## 5. Connection to the Bird Problem (Lecture 2 Callback)

The teacher explicitly connects this to the Bird problem from Lecture 2:

| Bird Problem | List Problem |
|-------------|-------------|
| `fly()` behavior varies per bird | `printList()` behavior varies per client |
| `hunt()` behavior varies differently | `sort()` behavior varies differently |
| Subclassing by `fly` breaks grouping for `hunt` | Subclassing by print breaks grouping for sort |
| **Solution: Inject fly strategy, hunt strategy** | **Solution: Inject print strategy, sort strategy** |

**Teacher's Note on Birds specifically:** Flying is NOT an intrinsic property of all birds (some birds can't fly), so `fly()` should not be in the base Bird class. But for behaviors like hunting, the strategy pattern works perfectly to decouple behavior from the entity.

---

## 6. Observer Pattern — Introduction (Problem Setup)

The teacher introduces the next design pattern problem but **does NOT give the solution** (deferred to next class).

### The Problem: Game Event Broadcasting

> You are designing a game. A monster must be killed. When the monster dies, multiple things must change simultaneously:
> - **Background** must change (handled by `BackgroundRenderer` class)
> - **Music** must change (handled by `MusicPlayer` class)
> - **Player state** must update — health increases, powers gained (handled by `PlayerState` class)
> - Potentially **more classes** in the future that need to react to this event

Each of these classes **already knows what to do** — they have methods like `changeBackground()`, `changeMusic()`, `updatePlayerState()`. The problem is: **how do they find out that the monster died?**

### Approaches Discussed and Rejected

| Approach | Why It Fails |
|----------|-------------|
| **Check state in game loop** — `if (monster.isDead()) { background.change(); music.change(); ... }` | Violates SRP and OCP. Game controller should not know about all dependent classes. |
| **Call changes inside `monster.setState()`** | Violates SRP — Monster class now has two reasons to change (its own state + notifying others). Also violates OCP. |
| **New coordinator class** that calls all change methods | Better, but still violates OCP — adding a new listener requires modifying the coordinator's code, retesting, redeploying. |

### What the Solution Should Achieve

- When the monster dies, all interested classes get **automatically notified**
- **Adding** a new class that cares about the monster's death should require **no changes to existing code**
- **Removing** a listener should also require no code changes to core logic
- The monster (or its manager) should **not know the specific classes** listening to it — only that there are some listeners

**Teacher's Hint:** This is the **Observer Design Pattern** (to be discussed in the next class).

---

## 7. Interview Questions

### Direct Questions

1. **What is the Strategy Design Pattern? When would you use it?**
   - A behavioral pattern that lets you define a family of algorithms, encapsulate each one, and make them interchangeable at runtime.

2. **Strategy vs. Inheritance — when do you choose one over the other?**
   - Use inheritance when behavior groupings are consistent across all methods. Use Strategy when different methods have overlapping behavior across different subclass groupings (combinatorial explosion problem).

3. **Design a Tic-Tac-Toe game. Now, what if you want to support different rule sets?**
   - Use Strategy pattern: inject a `RuleStrategy` into the game controller. Different rule implementations can be swapped without changing the game loop.

4. **How would you design a system where objects of the same class behave differently?**
   - Strategy pattern — inject behavior through the constructor.

5. **What is the Observer Pattern? Give a real-world example.**
   - (Teased in this lecture) A pattern where an object (subject) maintains a list of dependents (observers) and notifies them of state changes automatically. Example: YouTube subscriptions, event-driven game systems.

### Follow-Up / Tricky Questions

6. **Why not just use if-else inside the method to switch behavior?**
   - Violates OCP — every new behavior requires modifying existing code. Also violates SRP if the class now decides which algorithm to use AND executes it.

7. **Can you change the strategy after object creation?**
   - In this lecture's design, NO — behavior is fixed at creation. But the pattern CAN support runtime strategy changes if you add a setter. Whether to allow it depends on requirements.

8. **What sorting algorithm would you use for almost-sorted data?**
   - **Insertion Sort** — O(n) for nearly sorted data, as very few elements need to be moved.

9. **What sorting algorithm for data containing only 0, 1, 2?**
   - **Count Sort** (two-pass) or **Dutch National Flag / 3-pointer algorithm** (single pass, O(n)).

---

## 8. Teacher's Special Insights

### Career & Design Tips

- **"Whenever you design anything, you should never be exposing a concrete class to the client. Always have an interface."** — This is a non-negotiable rule in professional software design.

- **"Time complexity is part of implementation, not design."** — In interviews, give the design first. Don't get bogged down in algorithm choices when the interviewer is asking about structure.

- **"The client will never give you requirements about how to design your class. They only give functional requirements — what methods they need. How to implement them is up to you."**

- **The Strategy Pattern is "one of the most commonly used design patterns when developing applications."** — This is not just academic; it is used extensively in production systems.

### Real-World Analogies

- **Game Design:** In games like chess, tic-tac-toe, snake & ladder — the game controller runs a loop. Rules, scoring, AI behavior — all of these can be injected as strategies. A common interview follow-up is: *"What if we want to change the rules?"* Strategy pattern is the answer.

- **The Bird Problem (recurring analogy):** Just as birds can't be cleanly subclassed by flying behavior (because hunting behavior cuts across differently), list implementations can't be subclassed by printing behavior (because sorting behavior cuts across differently). The solution in both cases is to **extract the varying behavior into separate strategy objects**.

### Thumb Rules

| Rule | Explanation |
|------|-------------|
| **Always program to an interface** | Client should never know the concrete class |
| **Use Factory to create objects** | Client specifies requirements via arguments; factory decides which class/strategy to use |
| **Composition over Inheritance** | Inject behavior objects instead of creating deep inheritance hierarchies |
| **If behaviors overlap across groupings, don't subclass** | Use Strategy pattern instead |

---

## 9. Exam Prep — Quick-Fire Facts

| Topic | Fact |
|-------|------|
| Strategy is a | **Behavioral** design pattern |
| Key principle it satisfies | **OCP** — add new strategies without modifying existing code |
| Strategy vs. Decorator | Decorator wraps and adds behavior; Strategy replaces behavior entirely |
| Strategy vs. Template Method | Template Method uses inheritance (subclass overrides steps); Strategy uses composition (inject algorithm object) |
| Proxy, Decorator, Adapter are all | Wrappers around existing classes |
| Flyweight solves | Memory optimization (sharing common state) |
| DIP says | Depend on abstractions, not concretions |
| Almost sorted data best sort | Insertion Sort |
| Data with only 0,1,2 best sort | Count Sort or Dutch National Flag (O(n)) |
| TimSort is used in | `Collections.sort()` in Java |
| Observer Pattern is for | Broadcasting state changes to multiple dependent objects |

### Design Pattern Classification Table

| Pattern | Type | Problem It Solves |
|---------|------|-------------------|
| Proxy | Structural | Control access to object |
| Decorator | Structural | Add behavior dynamically |
| Adapter | Structural | Interface compatibility |
| Flyweight | Structural | Memory optimization |
| Strategy | **Behavioral** | **Swap algorithms/behaviors per object** |
| Observer | **Behavioral** | **Notify dependents of state changes** (next class) |
| Factory | Creational | Object creation without exposing concrete classes |

---

## 10. Assignments & Upcoming Topics (Mentioned by Teacher)

- **Next class:** Observer Design Pattern solution + SOLID assignment explanations + Singleton + Immutable + Proxy + Adapter assignment explanations
- **New assignments releasing this week:** Proxy and Decorator assignments
- **Teacher expects 100% attendance** next class for assignment explanations — come prepared to present your solutions

---

## 11. Summary: The Complete Design Evolution

```
V0: Concrete MyList class wrapping ArrayList
    Problem: Client depends on concrete class (violates DIP)
        |
        v
V1: Add IList interface, MyList implements IList
    Problem: New requirements — different print/sort behaviors per client
        |
        v
    CONSIDERED: Subclassing (VerticalPrintList, HorizontalPrintList, etc.)
    REJECTED: Combinatorial explosion — behaviors overlap across groupings
        |
        v
V2 (FINAL): Strategy Design Pattern
    - ONE MyList class
    - PrintStrategy and SortStrategy injected via constructor
    - Factory creates objects based on client requirements
    - Client interacts only with IList interface
    - Adding new behaviors = adding new strategy class (no existing code changes)
```

**Key Takeaway:** The Strategy Pattern lets objects of the **same class** behave **differently** by injecting algorithm implementations as separate objects. It replaces complex inheritance hierarchies with simple composition, following the principle: **"Favor composition over inheritance."**
