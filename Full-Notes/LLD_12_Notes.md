# LLD Lecture 12 — Detailed Notes
## Topics: Observer Design Pattern | Design Interview Problem: "Design a Pen" | Strategy Pattern in Practice | Decorator Pattern | Interface Segregation

---

## 1. Recap: The Problem Statement (from Last Class)

> There is **one class** (the "subject") whose **state changes**. When the state changes, a **message must be delivered** to a bunch of **other classes** (the "observers") that are waiting for an update. All observer classes are already implemented. The task is to provide a **framework** for this message passing in an **extensible** way — so that new subscriber classes can be added/removed easily, and when the state changes, a specific function in each subscriber gets called automatically.

---

## 2. The Design Journey: Observer Pattern (V0 -> V1 -> Final)

### V0 — The Naive Facade / Wrapper Approach

A student (Jinesh) proposes:

- Create a class called `Changed` with a **static method** `change()`.
- Inside `change()`, manually call all required change methods:

```java
class Changed {
    static void change() {
        background.change();
        music.change();
        player.change();
    }
}
```

- The client simply calls `Changed.change()` and all changes happen.

**Advantage:**
- Client gets abstraction — doesn't need to know which classes are changing.

**Problems:**

| Problem | Explanation |
|---------|-------------|
| **Violates Open-Close Principle (OCP)** | To add/remove a change, you must modify the `change()` method — touching tested code. |
| **Violates Single Responsibility Principle (SRP)** | Multiple reasons to change: adding a new observer, removing one, modifying one. |
| **High re-testing & redeployment cost** | Every time game logic changes, the whole method must be tested and redeployed. |

> **Teacher's Insight:** If these changes are NOT frequent, this facade approach is actually a good-enough solution. The complexity of a design pattern is only justified when changes are frequent.

---

### V1 — Interface + List of Observers (Student's Improved Solution)

Jinesh improves the design:

1. Create an interface `IChange` with a single method `void change()`.
2. For each observer, create a **concrete class** implementing `IChange`:
   - `BackgroundChange implements IChange`
   - `MusicChange implements IChange`
   - `PlayerChange implements IChange`
3. Inside each concrete class's `change()`, call the actual method from the original class (e.g., `background.changeBackground()`).
4. The `Changed` class now holds an `ArrayList<IChange>` instead of hardcoded calls.
5. The `change()` method simply **iterates** over the list and calls `change()` on each item.

```java
class Changed {
    List<IChange> observers = new ArrayList<>();

    void change() {
        for (IChange observer : observers) {
            observer.change();
        }
    }

    void addObserver(IChange obs) { observers.add(obs); }
    void removeObserver(IChange obs) { observers.remove(obs); }
}
```

**Advantage:**
- Adding/removing observers is just `add()` or `remove()` on the list — **no code changes needed** in the `change()` method.
- OCP is now satisfied.

---

### V2 — The Full Observer Design Pattern (Teacher's Final Design)

The teacher formalizes the pattern with proper terminology:

**Key Components:**

| Component | Role |
|-----------|------|
| **Subject** | The object being observed (e.g., Monster). Holds a list of observers. Has a method to notify all observers. |
| **Observer Interface** | Common interface with an `update()` method. All observers implement this. |
| **Concrete Observers** | Classes like BackgroundHandler, MusicHandler, PlayerStateHandler. Each implements `update()` with its own logic. |

**Structure:**

```
Subject (e.g., Monster)
  - List<Observer> observers
  - notifyAll() { for each observer: observer.update() }
  - addObserver(Observer o)
  - removeObserver(Observer o)

<<interface>> Observer
  + update()

BackgroundHandler implements Observer
  + update() { // change background based on state }

MusicHandler implements Observer
  + update() { // change music based on state }

PlayerStateHandler implements Observer
  + update() { // change player attributes based on state }
```

---

### How Observers Get State Data (Two Approaches)

**Problem:** Different observers may need different data from the subject. E.g., background needs health, music needs alive/dead status, player state needs other attributes.

| Approach | How It Works | When to Use |
|----------|--------------|-------------|
| **Push Model** | `update(Subject subject)` — pass the subject reference as an argument to `update()` | When observers need varying data; future-proof if new attributes are added |
| **Pull Model** | Each observer holds a reference to the subject object. When `update()` is called, it reads whatever attributes it needs. | When observers are tightly coupled to subject anyway |

> **Teacher's Key Point:** Instead of writing a different method signature for every observer, just pass the full subject reference. Each observer picks what it needs. This is more flexible — if new attributes are added to the subject later, no method signature changes are needed.

---

### Handling Legacy/Pre-written Classes — Using Adapter Pattern

**Problem:** In the textbook Observer pattern, observer classes implement the Observer interface. But in our scenario, the classes (Background, Music, etc.) are **pre-written legacy code**. We cannot modify them to implement the Observer interface.

**Solution: Adapter Pattern**

- Do NOT modify existing classes.
- Create **adapter/wrapper classes** that implement the Observer interface.
- Each adapter holds an object of the original class internally.
- The adapter's `update()` method calls the appropriate method on the wrapped class.

```java
class BackgroundAdapter implements Observer {
    Background background;  // holds the legacy object

    BackgroundAdapter(Background bg) {
        this.background = bg;
    }

    void update() {
        background.changeBackground();  // delegate to legacy method
    }
}
```

> **Teacher's Insight:** Never modify already existing, tested legacy classes. Use adapters to bridge the gap between your new design pattern and old code.

---

## 3. Observer Pattern — Summary Table

| Aspect | Detail |
|--------|--------|
| **Pattern Name** | Observer (also known as Publish-Subscribe) |
| **Problem Solved** | One-to-many dependency: when one object changes state, all dependents are notified automatically |
| **Key Principle** | Open-Close Principle — add/remove observers without changing subject code |
| **Subject** | Maintains list of observers, notifies them on state change |
| **Observer** | Interface with `update()` method |
| **Adding new observer** | Create class implementing Observer, add to subject's list |
| **Removing observer** | Remove from subject's list |
| **Legacy classes** | Use Adapter pattern to wrap them |
| **Data passing** | Pass subject reference or let observers hold subject reference |

---

## 4. The Design Interview Problem: "Design a Pen"

### Context: Real Interview Experience

> **Teacher's Story:** This question was asked in an **Amazon SD2 (Software Developer 2) interview**. The candidate had **5 DSA rounds** — all rated **"strong hire."** The team was already discussing compensation. Then the **LLD round** happened with this question: **"Design a Pen."** The candidate got **rejected**. Final verdict: Rejected.

**Lesson:** LLD rounds can make or break your interview, even after acing all DSA rounds.

---

### The Problem Statement

> **Design a Pen.** You have 30 minutes.

That's it. That's the full problem statement.

---

### Phase 1: Asking the Right Questions (First 5 Minutes)

#### Questions Students Asked (and Teacher's Evaluation)

| # | Question Asked | Teacher's Verdict | Why |
|---|---------------|-------------------|-----|
| 1 | What will the pen have? | Unnecessary | You decide the attributes; you're designing the class |
| 2 | What behaviors can a pen have? (stabbing, throwing?) | Partially relevant | But should focus on client API, not physical behaviors |
| 3 | Type of pen? (gel, ball, ink) | Partially relevant | Better question: "Does behavior change based on pen type?" |
| 4 | Color of the pen? | Unnecessary | Color is just an attribute/variable — design is agnostic of how many colors exist |
| 5 | Digital or physical pen? | Unnecessary | It's an OOP class design — neither digital nor physical |
| 6 | Can it be a pencil? | Good question | Helps define scope |
| 7 | Components or working? | Decent | Answer: Both |
| 8 | Refill or use-and-throw? | Good | Clarifies scope |
| 9 | Cap or click? | Unnecessary from client perspective | It's YOUR design decision for internals |
| 10 | Who will be the client? | Good | Helps understand usage |
| 11 | Can color change? | Decent | Yes, on refill |
| 12 | What is a pen? (philosophical) | Unnecessary | Wastes time |

---

### The CORRECT First Question to Ask

> **"How exactly will the client use the pen? What API does the client need?"**

This is the **single most important question** in any LLD interview.

#### Client Usage (The Answer):

```java
// Creating a pen (via factory):
Pen pen = PenFactory.createPen(PenType.BALLPOINT, Color.RED, OpenType.CAP);

// Using the pen:
pen.start();             // Must be called before writing
pen.write("Hello");      // Writes the string
pen.close();             // Close after writing
pen.refill(Color.BLUE);  // Refill with new ink color
```

**Client provides 3 inputs when creating a pen:**
1. **Type of pen** — ballpoint, gel, ink
2. **Color of ink** — initial ink color
3. **Opening mechanism** — cap or click

---

### Teacher's Framework: Three Types of Questions in Design Interviews

| Type | Description | Example | When to Ask |
|------|-------------|---------|-------------|
| **Design Questions** | Impact the class structure, number of classes, relationships | "Does write behavior change based on pen type?" | ALWAYS ask these |
| **Implementation Questions** | Impact how a method is coded, not the structure | "Should ink decrease linearly when writing?" | Ask only if time permits; usually NOT needed |
| **Irrelevant Questions** | Neither impact design nor implementation | "How many colors can there be?" "What if the cap is lost?" | NEVER ask these |

> **Teacher's Thumb Rule:** If there's no function exposed to lose the cap, the cap can never be lost. Don't go into unnecessary details. Be focused.

---

### The CORRECT Questions to Ask

1. **Does the behavior of `write()` change based on pen type?** -- YES
2. **Does the behavior of `refill()` change based on pen type?** -- YES (ink pen vs gel pen refill differently)
3. **Does the behavior of `start()`/`close()` change based on pen type?** -- YES (cap vs click mechanism)
4. **What happens if `write()` is called before `start()`?** -- Throw an exception
5. **Can refill happen even if ink is full?** -- Yes, it simply refills with the new color

---

### Final Requirements Summary

| Requirement | Detail |
|------------|--------|
| Methods | `write(String)`, `refill(Color)`, `start()`, `close()` |
| Pen types | Ballpoint, Gel, Ink (extensible) |
| Opening mechanism | Cap or Click |
| Color | Fixed per pen, changes only on refill |
| Pencil support | No (pen only, forever) |
| Use-and-throw | No (refill only, forever) |
| Start before write | Mandatory, else throw exception |

---

## 5. Design Journey: Pen Design (V0 -> V1 -> Final)

### V0 — Abstract Class with All Methods Abstract

```
abstract class Pen {
    abstract void write(String text);
    abstract void refill(Color color);
    abstract void start();
    abstract void close();
}

class InkPen extends Pen { ... }
class BallPen extends Pen { ... }
class GelPen extends Pen { ... }
```

**Problem:**

| Issue | Explanation |
|-------|-------------|
| **Code duplication in `refill()`** | `BallPen` and `GelPen` may have the same refill logic. `InkPen` and `SketchPen` may share theirs. Making it abstract forces each class to re-implement it. |
| **Code duplication in `start()`/`close()`** | Only two implementations exist (cap vs click), but every concrete pen class must re-implement one of them. |
| **Class Explosion** | If you create separate classes for each combo (InkPenWithCap, InkPenWithClick, BallPenWithCap, ...) you get `N_types x M_mechanisms` classes. |

---

### V1 (Final) — Strategy Pattern for Varying Behaviors

**Key Insight:** Identify which behaviors vary independently and extract them into strategies.

| Method | Varies By | Solution |
|--------|-----------|----------|
| `write()` | Pen type (guaranteed different for each type) | Keep as **abstract method** in pen class |
| `refill()` | Pen type (but some share same logic) | Extract into **RefillStrategy** |
| `start()` / `close()` | Opening mechanism (cap vs click) | Extract into **OpenCloseStrategy** |

**Final Class Diagram:**

```
abstract class Pen {
    RefillStrategy refillStrategy;
    OpenCloseStrategy openCloseStrategy;
    Color inkColor;
    boolean isOpen;

    abstract void write(String text);   // Each pen type implements differently

    void refill(Color color) {
        refillStrategy.refill(color);   // Delegate to strategy
        this.inkColor = color;
    }

    void start() {
        openCloseStrategy.open();       // Delegate to strategy
        isOpen = true;
    }

    void close() {
        openCloseStrategy.close();      // Delegate to strategy
        isOpen = false;
    }
}

// --- Pen Types (only implement write) ---
class InkPen extends Pen {
    void write(String text) { /* ink pen specific writing */ }
}
class BallPen extends Pen {
    void write(String text) { /* ball pen specific writing */ }
}
class GelPen extends Pen {
    void write(String text) { /* gel pen specific writing */ }
}

// --- Refill Strategies ---
<<interface>> RefillStrategy {
    void refill(Color color);
}
class InkRefillStrategy implements RefillStrategy { ... }
class CartridgeRefillStrategy implements RefillStrategy { ... }

// --- Open/Close Strategies ---
<<interface>> OpenCloseStrategy {
    void open();
    void close();
}
class CapStrategy implements OpenCloseStrategy {
    void open() { /* remove cap */ }
    void close() { /* put cap back */ }
}
class ClickStrategy implements OpenCloseStrategy {
    void open() { /* click to extend */ }
    void close() { /* click to retract */ }
}
```

**Factory injects the correct strategies:**

```java
class PenFactory {
    static Pen createPen(PenType type, Color color, OpenType openType) {
        Pen pen;
        RefillStrategy refillStrategy;
        OpenCloseStrategy openCloseStrategy;

        // Select pen type
        switch(type) {
            case BALLPOINT:
                pen = new BallPen();
                refillStrategy = new CartridgeRefillStrategy();
                break;
            case GEL:
                pen = new GelPen();
                refillStrategy = new CartridgeRefillStrategy();
                break;
            case INK:
                pen = new InkPen();
                refillStrategy = new InkRefillStrategy();
                break;
        }

        // Select open/close mechanism
        switch(openType) {
            case CAP: openCloseStrategy = new CapStrategy(); break;
            case CLICK: openCloseStrategy = new ClickStrategy(); break;
        }

        pen.setRefillStrategy(refillStrategy);
        pen.setOpenCloseStrategy(openCloseStrategy);
        pen.setInkColor(color);
        return pen;
    }
}
```

> **Teacher's Key Point:** Don't overcomplicate. `write()` doesn't need a strategy because it is guaranteed to be different for every pen type — a simple abstract method suffices. Only use Strategy when different concrete classes **share** the same implementation of a behavior.

---

## 6. Follow-Up Questions in the Interview

### Follow-Up 1: Add Grip Feature

**Requirement:** Add a grip to any pen. When grip is added, the `write()` method should first print "It feels comfortable" and THEN do the original write behavior. No changes to existing classes.

**Solution: Decorator Pattern**

```java
class GrippedPen extends Pen {
    Pen wrappedPen;  // holds any pen object

    GrippedPen(Pen pen) {
        this.wrappedPen = pen;
    }

    void write(String text) {
        System.out.println("It feels comfortable");  // Added behavior
        wrappedPen.write(text);                       // Original behavior
    }

    // Delegate other methods to wrappedPen
    void refill(Color c) { wrappedPen.refill(c); }
    void start() { wrappedPen.start(); }
    void close() { wrappedPen.close(); }
}
```

**Usage:**
```java
Pen ballPen = PenFactory.createPen(BALLPOINT, RED, CAP);
Pen grippedBallPen = new GrippedPen(ballPen);  // Decorated!
grippedBallPen.write("Hello");
// Output: "It feels comfortable"
//         "Hello" (written in ballpoint style)
```

> **Key:** All existing classes remain completely untouched. The decorator wraps any pen and adds behavior.

---

### Follow-Up 2: Add Pencil Support

**Requirement:** Pencil can write but cannot refill, and has no start/close mechanism. In future, other writing instruments (coal, chalk) may be added.

**Solution: Interface Segregation Principle (ISP)**

- Extract `write()` into a separate **`Writable`** interface.
- Pen implements `Writable` (and has additional methods).
- Pencil implements only `Writable`.

```
<<interface>> Writable {
    void write(String text);
}

abstract class Pen implements Writable {
    // has refill, start, close + write
}

class Pencil implements Writable {
    void write(String text) { /* pencil writing logic */ }
}

// In future:
class Coal implements Writable { ... }
class Chalk implements Writable { ... }
```

**Client code becomes flexible:**
```java
Writable instrument = getWritingInstrument();  // Could be pen, pencil, coal...
instrument.write("Hello");
```

---

## 7. Interview Questions

### Conceptual Questions

1. **What is the Observer design pattern? When would you use it?**
   - One-to-many dependency. When one object changes state, all dependents are notified and updated automatically. Use when multiple objects need to react to a single object's state change.

2. **What are the two models for passing state in Observer pattern?**
   - Push model (subject sends data to observers via method args) vs Pull model (observers query subject for data).

3. **How do you apply Observer pattern to legacy classes that you cannot modify?**
   - Use the Adapter pattern to wrap legacy classes in adapters that implement the Observer interface.

4. **What is the Strategy pattern? How is it different from inheritance?**
   - Strategy encapsulates interchangeable algorithms. Unlike inheritance which bakes behavior into the class hierarchy, Strategy allows behavior to be swapped at runtime via composition.

5. **When should you use Strategy vs Abstract Method?**
   - Use abstract method when every subclass MUST have a different implementation. Use Strategy when some subclasses share the same implementation of a behavior.

6. **What is the Decorator pattern? How does it differ from inheritance?**
   - Decorator adds behavior at runtime by wrapping objects. Inheritance adds behavior at compile time. Decorator avoids class explosion.

7. **What is the Interface Segregation Principle? Give an example.**
   - Clients should not be forced to depend on interfaces they don't use. Example: Pencil shouldn't implement refill/start/close. Separate `Writable` interface from `Pen` methods.

### Design Interview Tips (from Teacher)

8. **What is the first question to ask in an LLD interview?**
   - "How will the client use this? What API does the client need?"

9. **What are the three things evaluated in an LLD interview?**
   - (a) Quality of your clarifying questions, (b) Quality of your design, (c) Ability to convey your design via class diagrams.

---

## 8. Design Decisions — Why One Approach Over Another

| Decision Point | Option A | Option B (Chosen) | Why B is Better |
|---------------|----------|-------------------|-----------------|
| Observer notification | Hardcoded method calls in a facade | List of Observer interface + iteration | OCP compliance; add/remove observers without code change |
| Observer data passing | Different method signatures per observer | Pass full subject reference | Future-proof; no signature changes when new attributes are added |
| Legacy observer classes | Modify legacy classes to implement interface | Adapter wrappers around legacy classes | Never modify tested legacy code |
| Pen behaviors that vary | Make all methods abstract (each subclass implements everything) | Strategy pattern for shared behaviors | Avoids code duplication; no class explosion |
| Write method design | Strategy pattern | Abstract method | Write is guaranteed different for every pen type; strategy is overkill |
| Adding grip feature | Create new subclasses (GrippedBallPen, GrippedInkPen...) | Decorator pattern | Avoids class explosion; works with any pen type |
| Adding pencil | Force pencil into Pen hierarchy | Interface Segregation (Writable interface) | Pencil doesn't need refill/start/close; don't force unnecessary methods |

---

## 9. Important Points / Key Concepts

1. **Observer Pattern** = Subject + Observer interface + List of observers + Notify method
2. **SRP Litmus Test**: "How many reasons does this class/method have to change?" More than one = violation.
3. **OCP is critical for frequently changing code.** If changes are rare, a simpler facade may suffice.
4. **Strategy vs Abstract Method**: Use Strategy when behavior is shared across subclasses; use abstract method when each subclass has unique behavior.
5. **Class Explosion** occurs when you create classes for every combination of features. Strategy pattern prevents it.
6. **Decorator** adds behavior to objects without modifying existing classes. Perfect for "add feature to existing design" follow-ups.
7. **Interface Segregation** is the answer when a new entity shares some but not all behaviors with an existing hierarchy.
8. **Adapter Pattern** bridges the gap between a new interface and legacy classes.
9. **Factory Pattern** encapsulates object creation logic, injecting correct strategies based on client input.

---

## 10. Teacher's Special Insights / Career Tips

### On Asking Questions in Interviews

> "The first thing you have to ask is: how exactly a client will use it. In none of these methods, there is an argument which is pressure applied, right? Client is never going to tell how much pressure because this is code."

- **Don't confuse the real-world entity with the class.** A student class has attributes and methods — so does a pen class. There's no physical cap to lose, no pressure to apply.
- **Don't suggest new requirements.** If the interviewer gives you the methods, you are NOT supposed to add more.
- **Don't ask irrelevant questions** like "how many colors exist" — your design should be agnostic of such counts.

### On Over-engineering

> "You don't need to overcomplicate things. Write can still be just an abstract method. We don't need a writing strategy."

- Only apply a design pattern when it solves a real problem.
- If a behavior is guaranteed to differ for every subclass, a simple abstract method is cleaner than a strategy.

### On SRP Pragmatism

> "As you keep zooming into your code, you will see that every part of your code is in some way violating SRP. You cannot have your code follow it to 100%. But ensure it is followed to a large extent."

### On Class Diagrams

> "If you are simply saying it in words, that is a red flag. You should have a concrete design — a class diagram."

- In interviews, **draw class diagrams**, don't just write code or talk.
- In real company discussions, class diagrams are what you create during technical design meetings.

### On Design Interview Evaluation

Three things are evaluated:
1. **Are you asking relevant questions?** (not physical-world questions, but client-API questions)
2. **Is your design following SOLID principles?** (extensible, maintainable, understandable)
3. **Can you convey the design?** (via class diagrams, not just words)

### The Amazon Interview Story

> A candidate aced 5 DSA rounds ("strong hire"), the team was discussing compensation — then he got **rejected** in the LLD round on "Design a Pen."

**Lesson:** LLD is a make-or-break round. Technical skill alone is not enough.

---

## 11. Exam Prep / Quick Revision

### Design Patterns Covered in This Lecture

| Pattern | Problem It Solves | Key Mechanism |
|---------|------------------|---------------|
| **Observer** | Notify multiple objects when one object's state changes | Subject holds list of observers; iterates and calls `update()` |
| **Adapter** | Make legacy classes work with new interfaces | Wrapper class implements new interface, delegates to legacy object |
| **Strategy** | Avoid code duplication when behaviors are shared across subclasses | Extract behavior into interface + concrete strategies; inject via constructor |
| **Decorator** | Add features to objects without modifying existing classes | Wrapper class holds original object; adds behavior before/after delegation |
| **Factory** | Encapsulate complex object creation | Static method creates and configures objects based on input parameters |
| **Interface Segregation** | Prevent classes from being forced to implement irrelevant methods | Split large interfaces into smaller, focused ones |

### Pen Design Quick Reference

| Component | Pattern Used | Why |
|-----------|-------------|-----|
| Pen type hierarchy | Abstract class + inheritance | Each pen type has unique write behavior |
| Refill behavior | Strategy pattern | Some pen types share refill logic |
| Start/Close behavior | Strategy pattern | Only 2 variants (cap/click), independent of pen type |
| Adding grip | Decorator pattern | Adds behavior without modifying existing classes |
| Adding pencil | Interface Segregation | Pencil only needs `write()`, not refill/start/close |
| Object creation | Factory pattern | Client specifies type + color + mechanism; factory assembles |

### Quick-Fire Facts

- Observer pattern is also called **Publish-Subscribe**.
- In Observer, you can pass data via **push** (argument) or **pull** (observer holds subject reference).
- Strategy avoids **class explosion** caused by combinatorial feature sets.
- Decorator follows **Open-Close Principle** — extends behavior without modifying existing code.
- Always ask **"How will the client use this?"** before designing.
- Class diagrams, not code, are the expected deliverable in LLD interviews.
- SRP litmus test: **"How many reasons to change?"**
- If changes to a class are **infrequent**, a simple facade may be better than a complex pattern.

---

## 12. What's Coming Next

- **Class Diagram Notations** — symbols, relationships, how to draw proper UML class diagrams.
- **More Interview Design Problems** — each remaining class will tackle a real interview LLD problem.
- **Assignment Discussion** — pushed to the next session.
