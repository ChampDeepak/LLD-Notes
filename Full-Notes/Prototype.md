# Prototype Design Pattern

The Prototype pattern says: **instead of creating a new object from scratch, clone an existing one and modify it.**

## The Problem Prototype Solves





### 🔴 The Core Pain: Expensive or Complex Object Creation

Imagine you're building a game. You have a `Monster` object that:
- Loads a 3D mesh from disk
- Fetches AI behavior config from a database
- Runs 200ms of initialization logic

Now you need **50 monsters of the same type** in a level.

```java
// Naive approach — you're paying the full cost 50 times
for (int i = 0; i < 50; i++) {
    Monster m = new Monster(); // 200ms init EVERY TIME 😭
}
// Total: 50 × 200ms = 10 seconds just to spawn monsters
```

That's Problem #1: **Creation is expensive** (I/O, DB calls, heavy computation).

---

### 🔴 Problem #2: You Don't Always Know the Exact Class at Runtime

Say your code receives objects from an external source (a config file, network, user input). You want to duplicate them — but you only have a reference of type `Shape`, not `Circle` or `Rectangle`.

```java
Shape s = getShapeFromSomewhere(); // could be Circle, Triangle, anything
Shape copy = new Shape(s); // ❌ WRONG — you lose the actual subtype's data
```

You can't just call `new` on the base type. You'd lose all subclass-specific fields.

---

### 🔴 Problem #3: Tight Coupling to Concrete Classes

When you write `new Monster()`, your code is **hardcoded** to `Monster`. If tomorrow you want `IceMonster` or `FireMonster`, you have to change the spawning code everywhere.

---

### ✅ The Insight Prototype Uses

> **"Instead of building a new object from scratch, copy an existing one that's already fully set up."**










---

## High-Level Structure

```
         ┌─────────────────────────┐
         │     <<interface>>       │
         │       Prototype         │
         │─────────────────────────│
         │ + clone(): Prototype    │
         └────────────┬────────────┘
                      │ implements
          ┌───────────┴───────────┐
          │                       │
┌─────────▼──────────┐ ┌─────────▼──────────┐
│  ConcretePrototype1 │ │  ConcretePrototype2 │
│────────────────────│ │────────────────────│
│ - field1           │ │ - fieldA           │
│ - field2           │ │ - fieldB           │
│────────────────────│ │────────────────────│
│ + clone()          │ │ + clone()          │
└────────────────────┘ └────────────────────┘

          ┌──────────────────────────────┐
          │      PrototypeRegistry       │
          │      (optional)              │
          │──────────────────────────────│
          │ - registry: Map<Key, Proto>  │
          │──────────────────────────────│
          │ + addPrototype(key, proto)   │
          │ + getPrototype(key): Proto   │
          └──────────────────────────────┘
```

**How to read this diagram:**

- The `Prototype` interface sits at the top. It declares one essential method: `clone()`.
- Concrete classes implement this interface and provide their own cloning logic.
- An optional `PrototypeRegistry` stores pre-built prototype objects keyed by a name or type, so clients can request clones by key without knowing the concrete class.

---

## Participants — In Depth

### 1. Prototype (Interface / Abstract Class)

**Role:** Declares the contract that all cloneable objects must follow. It is the common type that the client codes against.

**Attributes:** None of its own — it's a pure interface (or abstract class with no fields).

**Methods:**

| Method | What It Does |
|---|---|
| `clone(): Prototype` | Returns a copy of the object. Each concrete class decides whether this is a shallow or deep copy. |

**Relationships:**
- Every concrete prototype **implements** this interface.
- The client and the registry reference objects through this type — they never need to know the concrete class to make a copy.

**Why it matters:** Without this interface, the client would need to know the concrete class of every object it wants to clone. The interface decouples the "act of cloning" from "which specific class is being cloned."

```java
public interface Prototype {
    Prototype clone();
}
```

> **Note:** In Java, you can also use the built-in `Cloneable` interface and override `Object.clone()`. However, writing your own `clone()` method gives you full control over shallow vs. deep copying, avoids the pitfalls of `Cloneable` (which is widely considered a broken API), and makes the intent explicit.

---

### 2. ConcretePrototype (One or More Implementing Classes)

**Role:** The actual objects that can be cloned. Each concrete prototype knows how to copy itself — including all its internal state.

**Attributes:** Whatever fields define the object's state. For example, a `Shape` might have `color`, `x`, `y`, `width`, `height`.

**Methods:**

| Method | What It Does |
|---|---|
| `clone(): Prototype` | Creates a new instance of the same class, copies all field values into it, and returns it. This is where you decide shallow vs. deep copy. |
| Constructor (copy constructor) | Often, `clone()` is implemented by calling a **private copy constructor** — a constructor that takes an instance of the same class and copies its fields. This is a clean, safe pattern. |

**Relationships:**
- **Implements** the `Prototype` interface.
- Has **no dependency** on other concrete prototypes — each class is self-contained in its cloning logic.
- Is **stored in** the registry (if one exists) as a template instance.

**Why it matters:** The cloning logic lives inside the class itself. This is critical because:
- The class has access to its own private fields (external code might not).
- The class knows which fields need deep copies vs. shallow copies.
- If a subclass adds new fields, it overrides `clone()` to copy those too.

```java
public class Circle implements Prototype {
    private int x;
    private int y;
    private int radius;
    private String color;

    // Normal constructor — used when building from scratch
    public Circle(int x, int y, int radius, String color) {
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.color = color;
    }

    // Copy constructor — used by clone()
    // private: only this class calls it
    private Circle(Circle source) {
        this.x = source.x;
        this.y = source.y;
        this.radius = source.radius;
        this.color = source.color;  // String is immutable, shallow copy is fine
    }

    @Override
    public Prototype clone() {
        return new Circle(this);  // delegate to copy constructor
    }

    // getters, setters, toString...
}
```

---

### 3. Prototype Registry (Optional but Common)

**Role:** A centralized store of pre-built prototype objects. Clients request a clone by key (like `"large-red-circle"`) instead of finding or constructing a template themselves.

**Attributes:**

| Attribute | Type | Purpose |
|---|---|---|
| `registry` | `Map<String, Prototype>` | Maps a key/name to a prototype instance. The stored instances are never returned directly — only their clones are. |

**Methods:**

| Method | What It Does |
|---|---|
| `addPrototype(key, prototype)` | Stores a prototype template in the registry under the given key. |
| `getPrototype(key): Prototype` | Looks up the prototype by key and returns a **clone** (not the original). This is crucial — if you return the original, consumers would mutate it and corrupt the template. |

**Relationships:**
- **Contains** (has-a) one or more `Prototype` references.
- **Does not know** the concrete types — it interacts with prototypes purely through the `Prototype` interface.
- **Used by** the client to obtain clones without knowing which concrete class is behind the key.

**Why it matters:** Without a registry, the client must hold onto prototype instances itself. The registry centralizes this responsibility:
- Pre-configured templates are set up once (e.g., at startup).
- Any part of the code can get a clone by asking the registry.
- Adding a new variation means adding one entry to the registry — no new classes needed.

```java
public class PrototypeRegistry {
    private Map<String, Prototype> registry = new HashMap<>();

    public void addPrototype(String key, Prototype prototype) {
        registry.put(key, prototype);
    }

    public Prototype getPrototype(String key) {
        Prototype proto = registry.get(key);
        if (proto == null) {
            throw new IllegalArgumentException("No prototype registered for: " + key);
        }
        return proto.clone();  // ALWAYS return a clone, never the original
    }
}
```

---

### 4. Client

**Role:** The code that needs new objects. Instead of calling constructors directly, it asks for clones from a prototype instance or from the registry.

**Relationships:**
- **Depends on** the `Prototype` interface (not on concrete classes).
- **Uses** the registry (if present) to get clones by key.
- **Modifies** the cloned object after receiving it (e.g., changes position, color, etc.).

**Why it matters:** The client is completely decoupled from the concrete classes. It doesn't know (or care) whether it's cloning a `Circle`, a `Rectangle`, or some future shape that doesn't exist yet. This makes the system open for extension without modifying client code.

---

## Shallow Copy vs. Deep Copy — A Critical Decision

This is the most important implementation detail in the Prototype pattern. Get it wrong and you'll have shared mutable state causing mysterious bugs.

### Shallow Copy

Copies the field values as-is. For primitive types (`int`, `boolean`) this creates independent copies. For reference types (objects, arrays, lists), it copies the **reference** — both the original and clone point to the **same object in memory**.

```
Original:  color = "red"    list = [A, B, C] ──→ [A, B, C] in heap
                                                       ↑
Clone:     color = "red"    list = ─────────────────────┘
                                   (same reference!)
```

**Problem:** If the clone adds an item to `list`, the original's list changes too.

### Deep Copy

Recursively copies all referenced objects, creating completely independent copies.

```
Original:  color = "red"    list = [A, B, C] ──→ [A, B, C] in heap

Clone:     color = "red"    list = [A, B, C] ──→ [A, B, C] in heap (separate!)
```

### Rules of Thumb

| Field Type | Shallow Copy Safe? | Why |
|---|---|---|
| Primitives (`int`, `boolean`, etc.) | Yes | Primitives are always copied by value |
| Immutable objects (`String`, `Integer`, `LocalDate`) | Yes | They can't be changed, so sharing is safe |
| Mutable objects (`ArrayList`, `Date`, custom objects) | **No** | Both copies share the same mutable object — changes to one affect the other |
| Arrays | **No** | Arrays are mutable reference types |

**In your `clone()` method:** copy primitives and immutable objects directly, but create new copies of mutable objects.

```java
// Inside clone() or copy constructor:
this.name = source.name;                           // String — immutable, safe
this.scores = new ArrayList<>(source.scores);      // List — mutable, deep copy
this.address = new Address(source.address);        // Custom obj — deep copy
this.id = source.id;                               // int — primitive, safe
```

---

## Cartoon Implementation Example: Monster Factory in a Game

Imagine you're building a game. You have different types of monsters. Creating a monster from scratch involves loading sprites, computing stats from a formula, setting up AI behavior — it's expensive. Instead, you create one template of each monster type and clone it whenever you need to spawn one.

### Step 1: The Prototype Interface

```java
public interface MonsterPrototype {
    MonsterPrototype clone();
    void showInfo();
}
```

### Step 2: Concrete Prototypes

```java
public class Zombie implements MonsterPrototype {
    private String name;
    private int health;
    private int attackPower;
    private List<String> abilities;  // mutable — needs deep copy

    // Normal constructor — expensive, used only for creating templates
    public Zombie(String name, int health, int attackPower, List<String> abilities) {
        this.name = name;
        this.health = health;
        this.attackPower = attackPower;
        this.abilities = abilities;

        // Simulate expensive setup
        System.out.println("[Zombie] Heavy construction: loading sprites, computing AI...");
    }

    // Copy constructor — cheap, used by clone()
    private Zombie(Zombie source) {
        this.name = source.name;
        this.health = source.health;
        this.attackPower = source.attackPower;
        this.abilities = new ArrayList<>(source.abilities);  // DEEP copy of mutable list
        // No expensive setup — we just copied the pre-computed state
    }

    @Override
    public MonsterPrototype clone() {
        return new Zombie(this);
    }

    // Allow modification after cloning
    public void setName(String name) { this.name = name; }
    public void setHealth(int health) { this.health = health; }

    @Override
    public void showInfo() {
        System.out.println("Zombie [" + name + "] HP=" + health
            + " ATK=" + attackPower + " abilities=" + abilities);
    }
}
```

```java
public class Dragon implements MonsterPrototype {
    private String name;
    private int health;
    private int attackPower;
    private String element;          // "fire", "ice", etc.
    private List<String> abilities;

    // Normal constructor
    public Dragon(String name, int health, int attackPower, String element, List<String> abilities) {
        this.name = name;
        this.health = health;
        this.attackPower = attackPower;
        this.element = element;
        this.abilities = abilities;
        System.out.println("[Dragon] Heavy construction: loading model, particle effects...");
    }

    // Copy constructor
    private Dragon(Dragon source) {
        this.name = source.name;
        this.health = source.health;
        this.attackPower = source.attackPower;
        this.element = source.element;           // String — immutable, shallow copy fine
        this.abilities = new ArrayList<>(source.abilities);  // deep copy
    }

    @Override
    public MonsterPrototype clone() {
        return new Dragon(this);
    }

    public void setName(String name) { this.name = name; }
    public void setHealth(int health) { this.health = health; }

    @Override
    public void showInfo() {
        System.out.println("Dragon [" + name + "] HP=" + health
            + " ATK=" + attackPower + " element=" + element + " abilities=" + abilities);
    }
}
```

### Step 3: The Registry

```java
import java.util.HashMap;
import java.util.Map;

public class MonsterRegistry {
    private Map<String, MonsterPrototype> templates = new HashMap<>();

    public void register(String key, MonsterPrototype prototype) {
        templates.put(key, prototype);
    }

    public MonsterPrototype spawn(String key) {
        MonsterPrototype template = templates.get(key);
        if (template == null) {
            throw new IllegalArgumentException("Unknown monster type: " + key);
        }
        return template.clone();  // clone, never return the template itself
    }
}
```

### Step 4: Client Code — Putting It All Together

```java
import java.util.Arrays;

public class Game {
    public static void main(String[] args) {

        // === SETUP PHASE (done once at game start) ===

        // Create template monsters — this is the expensive part
        Zombie zombieTemplate = new Zombie(
            "Basic Zombie", 100, 15, Arrays.asList("bite", "scratch")
        );

        Dragon dragonTemplate = new Dragon(
            "Basic Dragon", 500, 80, "fire", Arrays.asList("fireball", "tail-swipe", "fly")
        );

        // Register templates
        MonsterRegistry registry = new MonsterRegistry();
        registry.register("zombie", zombieTemplate);
        registry.register("dragon", dragonTemplate);

        System.out.println("\n=== SPAWNING PHASE (cloning is cheap) ===\n");

        // === SPAWNING PHASE (done many times during gameplay) ===

        // Spawn zombies — no heavy construction, just cloning
        Zombie z1 = (Zombie) registry.spawn("zombie");
        z1.setName("Zombie #1");
        z1.setHealth(120);  // a tougher variant

        Zombie z2 = (Zombie) registry.spawn("zombie");
        z2.setName("Zombie #2");
        // z2 keeps default health (100)

        // Spawn a dragon
        Dragon d1 = (Dragon) registry.spawn("dragon");
        d1.setName("Smaug");
        d1.setHealth(999);

        // Show all spawned monsters
        z1.showInfo();
        z2.showInfo();
        d1.showInfo();

        System.out.println("\n=== TEMPLATE IS UNAFFECTED ===\n");

        // Prove that the original template is untouched
        zombieTemplate.showInfo();
        dragonTemplate.showInfo();
    }
}
```

### Output

```
[Zombie] Heavy construction: loading sprites, computing AI...
[Dragon] Heavy construction: loading model, particle effects...

=== SPAWNING PHASE (cloning is cheap) ===

Zombie [Zombie #1] HP=120 ATK=15 abilities=[bite, scratch]
Zombie [Zombie #2] HP=100 ATK=15 abilities=[bite, scratch]
Dragon [Smaug] HP=999 ATK=80 element=fire abilities=[fireball, tail-swipe, fly]

=== TEMPLATE IS UNAFFECTED ===

Zombie [Basic Zombie] HP=100 ATK=15 abilities=[bite, scratch]
Dragon [Basic Dragon] HP=500 ATK=80 element=fire abilities=[fireball, tail-swipe, fly]
```

**Key observations from the output:**

1. **"Heavy construction" prints only twice** — once per template. All clones avoid this cost.
2. **Clones are independent** — changing `z1`'s name and health didn't affect `z2` or the template.
3. **Templates are untouched** — the registry's stored prototypes remain in their original state.

---

## How the Participants Interact — Sequence of Events

```
  Client               Registry              Prototype (Zombie template)
    │                     │                          │
    │  spawn("zombie")    │                          │
    │────────────────────>│                          │
    │                     │  clone()                 │
    │                     │─────────────────────────>│
    │                     │                          │── creates new Zombie
    │                     │                          │   via copy constructor
    │                     │    return new Zombie     │
    │                     │<─────────────────────────│
    │   return clone      │                          │
    │<────────────────────│                          │
    │                                                │
    │── modifies clone                               │
    │   (setName, etc.)                              │
    │                                 template is unchanged
```

---

## Prototype vs. Other Creational Patterns

| Question | Answer | Pattern |
|---|---|---|
| "I need exactly one instance" | Restrict construction | **Singleton** |
| "I need to create families of related objects" | Abstract the factory | **Abstract Factory** |
| "I need to construct complex objects step by step" | Separate construction from representation | **Builder** |
| "I need a copy of an existing configured object" | Clone it | **Prototype** |
| "I need to delegate creation to subclasses" | Let subclasses decide | **Factory Method** |

**Prototype and Abstract Factory can work together:** An Abstract Factory can store a set of prototypes and return clones instead of calling constructors. This is useful when the products are expensive to create.

---

## Common Mistakes to Avoid

### 1. Returning the Original Instead of a Clone

```java
// WRONG — returns the template itself
public MonsterPrototype spawn(String key) {
    return templates.get(key);  // consumers will mutate the template!
}

// CORRECT — returns a clone
public MonsterPrototype spawn(String key) {
    return templates.get(key).clone();
}
```

### 2. Forgetting Deep Copy for Mutable Fields

```java
// WRONG — shared list between original and clone
private Zombie(Zombie source) {
    this.abilities = source.abilities;  // both point to the same list!
}

// CORRECT — independent copy
private Zombie(Zombie source) {
    this.abilities = new ArrayList<>(source.abilities);
}
```

### 3. Not Handling Inheritance in Clone

If `FastZombie extends Zombie`, the `Zombie`'s copy constructor will only copy `Zombie` fields. `FastZombie` must override `clone()` and have its own copy constructor that calls `super`'s copy constructor, then copies its own fields.

```java
public class FastZombie extends Zombie {
    private int sprintSpeed;

    private FastZombie(FastZombie source) {
        super(source);  // copies Zombie's fields
        this.sprintSpeed = source.sprintSpeed;  // copies FastZombie's field
    }

    @Override
    public MonsterPrototype clone() {
        return new FastZombie(this);  // NOT new Zombie(this)
    }
}
```

If `FastZombie` doesn't override `clone()`, calling `clone()` on a `FastZombie` would create a `Zombie` — **slicing off** the subclass data.

---

## Viva Quick-Fire Answers

**Q: What is the Prototype pattern in one sentence?**
A: Create new objects by cloning an existing instance instead of calling a constructor.

**Q: When would you use it?**
A: When object creation is expensive, when you need copies of objects whose concrete type you don't know at compile time, or when you want pre-configured object templates.

**Q: What's the key method?**
A: `clone()` — defined in the Prototype interface, implemented by each concrete class.

**Q: Shallow vs deep copy — which should you use?**
A: Deep copy for any mutable reference fields. Shallow copy is safe only for primitives and immutable objects.

**Q: How is it different from just using `new`?**
A: `new` requires knowing the concrete class and repeating all configuration. `clone()` works through the interface (no concrete class knowledge needed) and preserves the existing object's state.

**Q: What's the role of the registry?**
A: It stores pre-built prototypes by key so clients can get clones without holding references to templates themselves. It's optional but common in practice.

**Q: Can Prototype work with inheritance?**
A: Yes, but each subclass must override `clone()` and provide its own copy constructor. Otherwise you get object slicing — the clone loses the subclass data.
