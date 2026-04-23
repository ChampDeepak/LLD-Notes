# All Ways to Create a Singleton Class in Java

A Singleton ensures **exactly one instance** of a class exists and provides a global access point to it.

---

## 1. Eager Initialization

The instance is created **at class loading time**, even before anyone asks for it.

```java
public class Singleton {
    // Instance created immediately when class is loaded
    private static final Singleton INSTANCE = new Singleton();

    // Private constructor — no one outside can do new Singleton()
    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

**How it works:**
- `static final` means the JVM creates the instance when the class is loaded into memory.
- The constructor is `private`, so no other class can call `new Singleton()`.

**Pros:**
- Simple and clean.
- Thread-safe automatically — the JVM handles class loading in a thread-safe way.

**Cons:**
- Instance is created even if it's never used — wastes memory if the object is heavy.
- Cannot handle exceptions during creation (no way to pass parameters or do error handling).

**When to use:** When the singleton is lightweight and you're certain it will be used.

---

## 2. Lazy Initialization (Not Thread-Safe)

The instance is created **only when first requested**.

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {          // Check if instance exists
            instance = new Singleton();  // Create only on first call
        }
        return instance;
    }
}
```

**How it works:**
- First call: `instance` is `null`, so it creates the object.
- Subsequent calls: `instance` is not `null`, so it returns the existing one.

**Pros:**
- Lazy — object is created only when needed.

**Cons:**
- **NOT thread-safe.** If two threads call `getInstance()` simultaneously and both see `instance == null`, they both create a new instance. Now you have two singletons — the pattern is broken.

**When to use:** Only in single-threaded applications. In practice, almost never.

---

## 3. Synchronized Method (Thread-Safe but Slow)

Fix the thread-safety problem by synchronizing the entire method.

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    // synchronized = only one thread can enter this method at a time
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**How it works:**
- The `synchronized` keyword ensures only one thread can execute `getInstance()` at a time.
- If Thread A is inside the method, Thread B waits until Thread A finishes.

**Pros:**
- Thread-safe.
- Lazy initialization.

**Cons:**
- **Huge performance hit.** Every single call to `getInstance()` acquires a lock, even after the instance is already created. Synchronization is only needed the first time, but we pay the cost every time.

**When to use:** When simplicity matters more than performance and `getInstance()` is called infrequently.

---

## 4. Double-Checked Locking (DCL)

The gold standard for lazy, thread-safe singletons with minimal performance overhead.

```java
public class Singleton {
    // volatile prevents instruction reordering (explained below)
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                    // 1st check — no lock
            synchronized (Singleton.class) {       // lock only if null
                if (instance == null) {            // 2nd check — with lock
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**How it works — step by step:**

1. **First check (without lock):** If instance already exists, return it immediately. No synchronization overhead. This is the fast path that 99.9% of calls will take.

2. **Synchronized block:** If instance is `null`, acquire the lock. Only one thread enters.

3. **Second check (with lock):** Why check again? Because between the first check and acquiring the lock, another thread might have already created the instance. Without this second check:
   - Thread A sees `null`, enters sync block
   - Thread B also sees `null`, waits for lock
   - Thread A creates instance, exits
   - Thread B enters sync block, creates ANOTHER instance (bug!)

**Why `volatile` is critical:**

The line `instance = new Singleton()` is actually 3 CPU instructions:
1. Allocate memory
2. Call the constructor (initialize the object)
3. Assign the memory address to `instance`

Without `volatile`, the JVM may **reorder** steps 2 and 3 (this is a real optimization CPUs do):
1. Allocate memory
2. Assign the memory address to `instance` (instance is now non-null!)
3. Call the constructor (object not yet initialized!)

If Thread B does the first check between steps 2 and 3, it sees a **non-null but half-constructed** object and returns it. `volatile` prevents this reordering.

**Pros:**
- Thread-safe.
- Lazy initialization.
- Synchronization cost only on the first call.

**Cons:**
- Verbose and tricky to get right.
- Still slightly slower than alternatives due to `volatile` read on every access.

**When to use:** When you need lazy initialization with good performance.

---

## 5. Bill Pugh Singleton (Static Inner Class)

The **recommended approach** by most Java experts. Elegant, lazy, and thread-safe without any synchronization.

```java
public class Singleton {
    private Singleton() {}

    // This inner class is NOT loaded until getInstance() is called
    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

**How it works:**

This exploits a JVM guarantee: **a class is not loaded until it is referenced for the first time.**

- When `Singleton` class is loaded, `SingletonHolder` is NOT loaded (it's a separate class).
- When `getInstance()` is called, THAT is when `SingletonHolder` is referenced for the first time.
- The JVM loads `SingletonHolder` and initializes its static field `INSTANCE`.
- The JVM guarantees that class loading is thread-safe — no two threads can initialize the same class simultaneously.

So you get:
- **Lazy initialization** — instance is created only when `getInstance()` is called.
- **Thread safety** — guaranteed by the JVM's class loading mechanism.
- **No synchronization overhead** — no `synchronized`, no `volatile`.

**Pros:**
- Clean, simple code.
- Lazy and thread-safe with zero synchronization overhead.
- No `volatile` or `synchronized` keywords.

**Cons:**
- Cannot pass arguments to the constructor (same as eager initialization).
- Slightly harder to understand the "why" for beginners.

**When to use:** This is the **default choice** for most singleton implementations.

---

## 6. Enum Singleton

The **simplest and most bulletproof** way. Recommended by Joshua Bloch (author of Effective Java).

```java
public enum Singleton {
    INSTANCE;

    // Add any fields and methods you need
    private int count = 0;

    public void doSomething() {
        count++;
        System.out.println("Count: " + count);
    }
}

// Usage
Singleton.INSTANCE.doSomething();
```

**How it works:**

Java enums are special classes. The JVM guarantees:
- Enum constants are instantiated **exactly once**.
- Enum instantiation is **thread-safe**.
- Enums **cannot be instantiated via reflection** (`Constructor.newInstance()` explicitly blocks enum types).
- Enums handle **serialization** correctly by default — deserializing always returns the same instance.
- Enums **cannot be cloned**.

**Pros:**
- Simplest code — just 3 lines for a basic singleton.
- Immune to reflection attacks.
- Immune to serialization/deserialization attacks.
- Immune to cloning.
- Thread-safe.

**Cons:**
- **Not lazy** — enum constants are created when the enum class is loaded.
- **Cannot extend a class** — enums already implicitly extend `java.lang.Enum`.
- Feels unusual for developers unfamiliar with the pattern.

**When to use:** When you want the most robust singleton that handles all edge cases. Especially useful when serialization is involved.

---

## How Singleton Can Be Broken (and How Each Approach Handles It)

### Attack 1: Reflection

```java
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);   // bypass private
Singleton second = constructor.newInstance(); // new instance!
```

**Defense (for non-enum approaches):**
```java
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance() — reflection not allowed");
    }
}
```

**Enum:** Automatically immune. The JVM throws `IllegalArgumentException` if you try.

---

### Attack 2: Serialization

```java
// Serialize
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
out.writeObject(instance);

// Deserialize — creates a NEW object!
ObjectInputStream in = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton second = (Singleton) in.readObject(); // different object!
```

**Defense:**
```java
// Add this method — JVM calls it during deserialization
protected Object readResolve() {
    return getInstance(); // return existing instance instead of the deserialized one
}
```

**Enum:** Automatically immune. Java's serialization mechanism handles enums specially.

---

### Attack 3: Cloning

```java
Singleton second = (Singleton) instance.clone();
```

**Defense:**
```java
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Singleton — cloning not allowed");
}
```

**Enum:** Automatically immune. `Enum.clone()` always throws `CloneNotSupportedException`.

---

## Summary Comparison

| Approach | Lazy? | Thread-Safe? | Reflection-Safe? | Serialization-Safe? | Performance |
|---|---|---|---|---|---|
| **Eager** | No | Yes | No (needs guard) | No (needs `readResolve`) | Excellent |
| **Lazy (basic)** | Yes | **No** | No | No | N/A (broken) |
| **Synchronized method** | Yes | Yes | No | No | Poor (lock every call) |
| **Double-Checked Locking** | Yes | Yes | No | No | Good |
| **Bill Pugh (inner class)** | Yes | Yes | No | No | Excellent |
| **Enum** | No | Yes | **Yes** | **Yes** | Excellent |

---

## Which One Should You Use?

```
Do you need lazy initialization?
├── No  → Is serialization/reflection a concern?
│         ├── Yes → Enum Singleton
│         └── No  → Eager Initialization
└── Yes → Bill Pugh (Static Inner Class)
          (or Double-Checked Locking if you need constructor arguments)
```

**In a viva, if asked "what is the best way?"** — say Bill Pugh for general use, Enum for bulletproof safety. Be ready to explain *why* for each.
