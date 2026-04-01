# 🪶 Flyweight Design Pattern — Interview Pitches

---

## 🎯 One-liner Pitch
> **"Flyweight shares common state across many objects to save memory — instead of each object holding all its data, they share the reusable parts and only keep what's unique to themselves."**

---

## 📌 Core Pitches (Say These Confidently)

- **Category:** Structural design pattern.
- **Problem it solves:** Too many fine-grained objects consuming excessive memory.
- **Core idea:** Split object state into **intrinsic** (shared, immutable) and **extrinsic** (unique per context, passed in at runtime).
- **Intrinsic state** = stored inside the flyweight object; shared across all users. Example: a character's font, size, color in a text editor.
- **Extrinsic state** = NOT stored inside the flyweight; passed by the client at runtime. Example: the position of that character on screen.
- **Flyweight Factory** manages a pool (cache) of flyweight objects — creates a new one only if it doesn't already exist.
- **Key trade-off:** Saves memory at the cost of CPU time (computing extrinsic state on the fly instead of storing it).

---

## 🧠 Intuition Anchor — "Forest of Trees"

> Imagine rendering a forest with 1 million trees. Each tree has a **type** (oak, pine — shared: texture, mesh, color) and a **position** (unique to each tree). Without Flyweight: 1M objects, each storing texture + position. With Flyweight: 3 flyweight objects (one per tree type) + 1M lightweight context objects holding only position.

**Memory saved: ~99%** of the texture/mesh data is shared.


---

## ⚡ When To Use — Interview Triggers

| Signal | Meaning |
|---|---|
| "Millions of similar objects" | Flyweight territory |
| "Objects differ only by a few fields" | Split into intrinsic/extrinsic |
| "Memory is a bottleneck" | Flyweight is the go-to |
| "Game particles, characters, tiles" | Classic Flyweight use cases |
| "Object pool" mentioned | Often related; distinguish them |

---

## 🏭 Real-World Examples (Drop These in Interview)

- **Java `String.intern()`** — string literals in the string pool are flyweights.
- **Java `Integer.valueOf(-128 to 127)`** — cached integers, same instance returned.
- **Text editors (MS Word internals)** — each character glyph is a flyweight; position is extrinsic.
- **Game engines** — particle systems share particle type data (texture, physics params); position/velocity are extrinsic.
- **Web browsers** — font rendering caches glyph bitmaps as flyweights.
- **Database connection pools** — connections are shared flyweights (connection config = intrinsic, current query = extrinsic).
- **Tile-based game maps** — tile texture is flyweight; map coordinate is extrinsic.

---

## 🔑 Key Properties (Hard Facts)

- Flyweights **must be immutable** — since many objects share the same instance, mutation would corrupt all users.
- The pattern trades **space for time** — CPU does more work to compute/pass extrinsic state.
- Flyweight is only worth it if **intrinsic state >> extrinsic state** in size.
- The **factory is essential** — without it, clients create duplicates and the pattern breaks.
- Flyweight objects are typically **not instantiated directly** — always go through the factory.
- **Thread-safe by design** — immutable shared state means no race conditions on the flyweight itself.

---

## 🆚 Distinguish From Similar Patterns

| Pattern | How it differs from Flyweight |
|---|---|
| **Singleton** | One instance total; Flyweight has multiple instances (one per type). |
| **Object Pool** | Pool recycles mutable objects; Flyweight shares immutable objects permanently. |
| **Prototype** | Prototype clones objects; Flyweight never clones — it reuses the same instance. |
| **Proxy** | Proxy controls access to one real object; Flyweight shares state across many. |
| **Factory Method** | General creation pattern; Flyweight Factory specifically caches and reuses. |

---

## ⚠️ Pitfalls & Trade-offs (Shows Depth)

- **If extrinsic state is large**, the memory savings evaporate — Flyweight only helps when intrinsic >> extrinsic.
- **Not useful for small-scale** — overhead of factory + state separation isn't worth it for < a few thousand objects.

---

## Participants of Flyweight Design Pattern

- **Flyweight** — a shared immutable entity which holds common/intrinsic attributes which exists in large number of objects. 
- **FlyweightFactory** — an entity that acts as flyweight provider. It maintains cache of flyweights and a key to access them. 
- **Context** — it is the main entity which keep its intrinsic attributes in the flyweight and extrinsic properties with itself. 
- **Client** — it is the entity that uses the context entity. Client passes the flywieght in the constructor of the context in order to create its instance. 

For better clarity refer pseudocode section of the https://refactoring.guru/design-patterns/flyweight

---

## 💻 Code Skeleton (Java-style Pseudocode)

```java
// Flyweight — Intrinsic state — shared, immutable
class TreeType {
    String name, color, texture; // heavy data
    void draw(int x, int y) { /* use x,y (extrinsic) + this.texture */ }
    // the draw method is not written in context class becuase if it was written in the context class then it will need to read heavy attributes from flywieght class but here flyweight class itself is having those so this function does not need to fetch them from somewhere else but it still need to fetch extrinsic properties and it do adds litle latency in execution. 
}

// Flyweight Factory — caches instances
class TreeFactory {
    Map<String, TreeType> cache = new HashMap<>();
    TreeType get(String name, String color, String texture) {
        String key = name + color + texture;
        return cache.computeIfAbsent(key, k -> new TreeType(name, color, texture));
    }
}

// Context — holds extrinsic state + reference to flyweight
class Tree {
    int x, y;              // extrinsic — unique per tree
    TreeType type;         // intrinsic — shared flyweight
    void draw() { type.draw(x, y); }
}

// Client
TreeFactory factory = new TreeFactory();
List<Tree> forest = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    TreeType oak = factory.get("oak", "green", "oak_texture.png"); // reused!
    forest.add(new Tree(randomX(), randomY(), oak));
}
```

**Key observation:** `factory.get("oak", ...)` returns the same `TreeType` object every time — 1M trees, but only 3 `TreeType` objects in memory.

---

## 📐 Intrinsic vs Extrinsic Cheat Sheet

| | Intrinsic | Extrinsic |
|---|---|---|
| **Stored in** | Flyweight object | Client / Context object |
| **Shared?** | Yes, across all users | No, unique per use |
| **Mutable?** | No — must be immutable | Yes |
| **Examples** | Font, texture, mesh, color | Position, velocity, ID |
| **Who owns it?** | Flyweight Factory | Caller / Context |

---

## 🎤 One-Sentence Answers for Common Interview Questions

| Question | Answer |
|---|---|
| What is Flyweight? | A structural pattern that shares intrinsic state across many objects to reduce memory. |
| What's intrinsic state? | Immutable, shareable data stored in the flyweight (e.g., texture, font). |
| What's extrinsic state? | Context-specific data passed in at runtime, not stored in the flyweight (e.g., position). |
| Why must flyweights be immutable? | Because many objects share the same instance — mutation would corrupt all of them. |
| What's the role of the factory? | It caches flyweights by key and prevents duplicate creation. |
| Where is Flyweight used in Java? | `Integer.valueOf()` (-128 to 127 cache), `String.intern()`, `Boolean.TRUE/FALSE`. |
| What's the trade-off? | Less memory, more CPU (extrinsic state must be computed/passed each time). |
| How is it different from Object Pool? | Pool reuses mutable objects; Flyweight shares immutable objects permanently. |

---

## 🧪 Quick Self-Check Questions

1. In a chess game, which parts of a chess piece are intrinsic and which are extrinsic?
2. Why would Flyweight **not** help if each object's "shared" data is only 4 bytes but extrinsic data is 200 bytes?
3. You have `Integer a = 100` and `Integer b = 100`. Does `a == b` return `true`? Why? Which pattern is at play?
4. A developer mutates a flyweight's intrinsic field at runtime. What goes wrong?
5. Draw the participants of the Flyweight pattern and describe how they interact.

---

*Pattern Family: Structural | GoF Book: Yes | Frequency in practice: Medium (mostly in performance-critical systems)*