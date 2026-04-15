# LLD Lecture 8 - Proxy Design Pattern & Flyweight Design Pattern (Structural Design Patterns)

---

## Table of Contents

1. [Overview and Context](#overview-and-context)
2. [The Teacher's Design Journey](#the-teachers-design-journey)
3. [Proxy Design Pattern](#proxy-design-pattern)
4. [Flyweight Design Pattern](#flyweight-design-pattern)
5. [Structural Design Patterns - The Big Picture](#structural-design-patterns---the-big-picture)
6. [Interview Questions](#interview-questions)
7. [Design Decisions and Trade-offs](#design-decisions-and-trade-offs)
8. [Important Points and Key Concepts](#important-points-and-key-concepts)
9. [Exam Prep - Quick-Fire Facts](#exam-prep---quick-fire-facts)
10. [Teacher's Special Insights](#teachers-special-insights)
11. [Code Examples](#code-examples)
12. [SOLID Assignment Discussion Summary](#solid-assignment-discussion-summary)

---

## Overview and Context

This lecture introduces **Structural Design Patterns** -- patterns that deal with how classes are composed and structured (NOT how objects are created). Two patterns are covered:

1. **Proxy Design Pattern** -- a wrapper class that controls access to another object
2. **Flyweight Design Pattern** -- sharing heavy/common data across many objects to save memory

Previously covered patterns (Singleton, Factory, Prototype, Builder) were all **Creational** patterns focused on object creation. Structural patterns focus on organizing relationships between classes.

---

## The Teacher's Design Journey

### Problem Setup (V0 -- The Naive Approach)

**Scenario:** There is a third-party NLP library with a `ITextParser` interface. Multiple implementations exist: `BookParser`, `WikiParser`, `RedditParser`. Each parser's constructor is very heavy -- it does chunking, POS tagging, syntactic parsing, semantic parsing, builds XML trees, vector sets, hash maps, etc.

**Step 1: Direct dependency (BAD)**

```
TextProcessor {
    someMethod() {
        BookParser bp = new BookParser(text);  // <-- Direct creation
        bp.getWordCount("hello");
    }
}
```

**Problem identified:** This violates the **Dependency Inversion Principle (DIP)**. The higher-level module (`TextProcessor`) directly depends on a lower-level module (`BookParser`).

**Consequences:**
- If you want to switch to `WikiParser` or `RedditParser`, you must change `TextProcessor`
- If `BookParser` gets deprecated or its constructor signature changes, you must change your tested, production code
- Violates Open/Closed Principle as well

---

### V1 -- Constructor Injection (Better, but still has a problem)

**Step 2: Inject the dependency through the constructor**

```java
class TextProcessor {
    private ITextParser parser;

    public TextProcessor(ITextParser textParser) {
        this.parser = textParser;
    }

    // methods use this.parser
}
```

Now `TextProcessor` depends on the abstraction `ITextParser`, not on `BookParser` directly. You can pass any implementation at runtime.

**But a new problem emerges...**

**Step 3: Introduce a Client class**

```
Client {
    clientFunction1() { textProcessor.f1(); }   // uses TextProcessor
    clientFunction2() { /* self-contained */ }   // does NOT use TextProcessor
    clientFunction3() { if (cond) textProcessor.f2(); }  // conditional use
    // ... 7 functions total, only 4 use TextProcessor, and those are conditional
}
```

To create a `Client` object:
```java
Client client = new Client(
    new TextProcessor(
        new BookParser(hugeBook)  // <-- HEAVY constructor called here
    )
);
```

**The core problem:** Creating a `Client` forces creation of `BookParser`, which is extremely heavy in both computation and memory. But the client may NEVER call those 4 functions, or the conditions inside them may never be true. The `BookParser` object is **eagerly loaded** even when not needed.

---

### Exploring Dead-End Solutions

| Solution Attempted | Why It Fails |
|---|---|
| Make TextParser a Singleton | Singleton restricts to one instance -- we may need multiple objects. Singleton solves only the "restrict to one instance" problem. |
| Lazy-load inside each function of TextProcessor | Writing `new BookParser()` inside functions violates DIP. Tomorrow if you switch to RedditParser, you change every function. |
| Ask the client to create objects lazily | Destroys abstraction. Client should not know internal details of TextProcessor. |
| Use Factory pattern inside TextProcessor | Whether created via factory or directly, the constructor still runs and takes time. Factory does not solve the lazy-loading problem. |

---

### V2 -- The Proxy Solution (FINAL DESIGN)

**Key Insight:** Create a NEW class that implements the SAME interface (`ITextParser`) but internally does lazy loading.

```
BookParserProxy implements ITextParser {
    private BookParser actualParser = null;  // starts as NULL

    private BookParser getInstance() {
        if (actualParser == null) {
            actualParser = new BookParser(book);  // created ONLY when needed
        }
        return actualParser;
    }

    getWordCount(word) {
        return getInstance().getWordCount(word);  // delegates to real object
    }

    getWordCountInContext(word, context) {
        return getInstance().getWordCountInContext(word, context);
    }
    // ... all other interface methods delegate similarly
}
```

**How client code changes:**

```java
// BEFORE:
Client client = new Client(new TextProcessor(new BookParser(book)));  // heavy

// AFTER:
Client client = new Client(new TextProcessor(new BookParserProxy(book)));  // lightweight!
```

- `BookParserProxy` is lightweight to create (no heavy computation)
- The actual `BookParser` is created ONLY when a method is first called
- Client code does NOT change at all (client deals with the `ITextProcessor` interface)
- `TextProcessor` code does NOT change (it receives an `ITextParser` object)

**On the DIP violation inside the Proxy:**

> The teacher acknowledges that `BookParserProxy` has a direct dependency on `BookParser`, which technically violates DIP. However, this class is NOT a generic class. It will NEVER need any other type of parser. It is a proxy specifically and only for `BookParser`. Since it will never need to change to another parser type, the DIP violation causes no practical issues.

---

## Proxy Design Pattern

### Definition

The Proxy Design Pattern provides a **wrapper/surrogate** for another object to control access to it. The proxy implements the **same interface** as the real object, holds a reference to the real object, and delegates calls to it -- but adds additional logic (lazy loading, access control, caching, retry logic, etc.).

### Structure

```
ITextParser (Interface)
    |
    +-- BookParser (Real/Heavy class)
    +-- WikiParser
    +-- RedditParser
    +-- BookParserProxy (Proxy -- implements same interface, wraps BookParser)
```

### Four Use Cases of the Proxy Pattern

#### 1. Virtual Proxy (Lazy Loading)
- **Problem:** Object is too heavy to create upfront
- **Solution:** Proxy delays creation until a method is actually called
- **Example:** The BookParser scenario above

#### 2. Protection Proxy (Access Control)

**Scenario:** You have a `IDocument` interface with `read()`, `write()`, `edit()`, `rename()` methods, and implementations: `PDF`, `Word`, `Notes`. Different user types need different access:

| User Type | PDF | Word | Notes |
|---|---|---|---|
| Employee | read, write, edit, rename | ALL | ALL |
| Student | read ONLY | ALL | ALL |
| Mentor | read ONLY | read ONLY | read ONLY |

You cannot change the `PDF`, `Word`, `Notes` classes (they work correctly in production). Instead, create proxy classes:

```
PDFProxy implements IDocument {
    private PDF actualPdf;
    private UserRole role;

    read()   { return actualPdf.read(); }           // always allowed
    write()  { if (role == EMPLOYEE) actualPdf.write(); else throw AccessDenied; }
    edit()   { if (role == EMPLOYEE) actualPdf.edit(); else throw AccessDenied; }
    rename() { if (role == EMPLOYEE) actualPdf.rename(); else throw AccessDenied; }
}
```

#### 3. Remote Proxy (Handling Remote Resources)

**Scenario:** You call an external API that frequently times out due to poor architecture.

**Without proxy:** Exception propagates to client, breaking client code.

**With proxy:**
- Wrap the remote resource inside a proxy class
- Implement retry logic (if timeout, wait and retry)
- Return a proper custom exception with a meaningful message if all retries fail
- Client is agnostic of the remote resource details

**Teacher's Rule:** "Whenever you are using any remote resource, you should NOT be directly calling those APIs in your actual code. You should have your own class and always call the functions of your own class. Inside that class, you have an object of that resource and then you hit the APIs."

#### 4. Caching Proxy

**Scenario:** Some function calls return the same result every time for the same input.

**Solution:** Add a cache layer inside the proxy:
- Before delegating to the real object, check the cache
- If cached, return directly (no computation/DB hit)
- Can implement LRU cache or other eviction strategies inside the proxy

### Key Rules of the Proxy Pattern

1. The proxy **MUST implement the same interface** as the real object
2. The proxy **has an object** of the concrete class it wraps
3. Each proxy is typically **specific to one concrete class** (e.g., `BookParserProxy` only wraps `BookParser`)
4. You **can** also have generic proxies that are agnostic of the concrete type (depends on use case)
5. The proxy does NOT write the actual business logic -- it **delegates** to the real object

---

## Flyweight Design Pattern

### The Teacher's Design Journey for Flyweight

**Context:** Continuing from the previous lecture's stone-dodging game (Factory pattern was used to create stones randomly/uniformly).

**Step 1: Analyze the stone object's memory**

Each stone has:
| Attribute | Size |
|---|---|
| x-coordinate | 4 bytes |
| y-coordinate | 4 bytes |
| speed | 4 bytes |
| weight | few bytes |
| **image** | **~20 KB** |

Total per stone: ~20 KB (image dominates)

**Step 2: Calculate memory for many objects**

| Number of Stones | Memory Required |
|---|---|
| 100,000 | ~2 GB |
| 1,000,000 | ~20 GB |

This is **main memory (RAM)**, not disk. Even 2 GB for a simple stone-dodging game is unacceptable.

**Step 3: Real-world observation**

> "Have you played CS2, BGMI, FIFA? In FIFA, thousands of audience members sit there. If you look closely, all the trees are exactly the same. You might have 10-20 different types of trees, but there is a fixed number of unique trees, and all others are exact replicas. The audience is the exact same set of people being repeated."

**Step 4: Separate intrinsic vs. extrinsic properties**

| Property Type | Definition | Examples | Shared? |
|---|---|---|---|
| **Intrinsic** | Properties that are the same across all objects of a type, regardless of state | Image, texture, appearance | YES -- shared across all objects |
| **Extrinsic** | Properties that change per object or per state | Position (x, y), speed, weight | NO -- unique per object |

**Step 5: The Flyweight Solution**

BEFORE (every stone has its own image copy):
```
Stone1: {x, y, speed, image(20KB)}
Stone2: {x, y, speed, image(20KB)}
Stone3: {x, y, speed, image(20KB)}
...
1,000,000 stones = 20 GB
```

AFTER (all stones of the same type share ONE image):
```
LargeStoneImage: image(20KB)   <-- ONE object in memory

Stone1: {x, y, speed, ref -> LargeStoneImage}  (~12 bytes + reference)
Stone2: {x, y, speed, ref -> LargeStoneImage}
Stone3: {x, y, speed, ref -> LargeStoneImage}
...
1,000,000 stones = ~12 MB + 20 KB (one image)
```

**Critical distinction from Prototype:** This is NOT cloning. All stones hold the **same reference** to one image object. In Prototype, you create copies. In Flyweight, you share the exact same object.

### Memory Savings Table

| Approach | 1 Million Stones |
|---|---|
| Without Flyweight | ~20 GB |
| With Flyweight | ~12 MB + 20 KB |
| Savings | ~99.94% |

### Key Rules of Flyweight Pattern

1. Identify **intrinsic** (shared, heavy, immutable) vs. **extrinsic** (unique, lightweight, mutable) properties
2. Extract intrinsic properties into a shared object
3. All instances hold a **reference** (not a copy) to the shared object
4. Since coordinates and speed differ, objects render differently despite sharing the image
5. You may have multiple categories of shared objects (10 tree types, 5 house types, etc.) -- but within each category, only ONE instance of the heavy data exists

---

## Structural Design Patterns -- The Big Picture

| Pattern Category | Focus | Examples |
|---|---|---|
| **Creational** | How objects are created | Singleton, Factory, Prototype, Builder |
| **Structural** | How classes are composed/structured | Proxy, Flyweight, (more in next lecture) |
| **Behavioral** | How objects interact/communicate | (upcoming lectures) |

> "In both Proxy and Flyweight, we are structuring the classes in a way so that they solve problems for us. We are not creating objects. We already have the classes, but we want to structure them in a way so that the code becomes maintainable, understandable, and in the case of Flyweight, optimal with respect to memory."

---

## Interview Questions

### Proxy Pattern

1. **What is the Proxy Design Pattern?** A structural pattern where a wrapper class controls access to another object by implementing the same interface and delegating calls.

2. **What are the types of proxies?** Virtual (lazy loading), Protection (access control), Remote (handles remote resources), Caching (stores results).

3. **How does Proxy differ from Decorator?** Both wrap objects, but Proxy controls ACCESS to the object (may delay creation, restrict access), while Decorator adds new BEHAVIOR/functionality.

4. **Does the Proxy class violate DIP?** Technically yes (it has a direct dependency on the concrete class), but since the proxy is specific to that one class and will never need another type, this violation causes no practical problems.

5. **When would you use a generic proxy vs. a specific proxy?** Use specific proxies when the logic is tied to a particular class. Use generic proxies when the added behavior (e.g., logging, caching) is independent of the concrete type.

6. **How would you handle API timeouts in production code?** Wrap the remote resource in a proxy that implements retry logic, backoff, and returns meaningful exceptions instead of propagating raw timeout errors to the client.

### Flyweight Pattern

7. **What is the Flyweight Design Pattern?** A structural pattern that minimizes memory by sharing common/heavy data (intrinsic state) among many objects while keeping unique data (extrinsic state) separate.

8. **What is the difference between intrinsic and extrinsic state?** Intrinsic = shared, immutable, context-independent (e.g., image texture). Extrinsic = unique per object, mutable, context-dependent (e.g., position, speed).

9. **How is Flyweight different from Prototype?** Prototype CLONES objects (each clone is independent). Flyweight SHARES the same reference (no duplication in memory).

10. **Give a real-world example of Flyweight.** In games like CS2/BGMI/FIFA: trees, houses, audience members, weapons all reuse the same image/texture. In FIFA, thousands of audience members use maybe 10-20 unique models.

11. **What is the memory impact of Flyweight?** For 1 million stones with 20KB images: without Flyweight = ~20 GB, with Flyweight = ~12 MB. Reduction of ~99.94%.

### Combined / General

12. **What are structural design patterns?** Patterns that deal with how classes are organized and composed to form larger structures, as opposed to creational patterns (object creation) or behavioral patterns (object interaction).

13. **Why not use Singleton to solve the heavy object problem?** Singleton restricts to ONE instance. If you need multiple objects (multiple parsers for different books), Singleton is wrong. Singleton solves only the "restrict to one instance" problem.

---

## Design Decisions and Trade-offs

### Why Proxy over Lazy-Loading Inside the Class?

| Approach | Pros | Cons |
|---|---|---|
| Lazy-load in TextProcessor methods | Simple | Violates DIP (must use `new BookParser()`), violates OCP, SRP |
| Ask client to lazy-load | No changes on your side | Destroys abstraction, leaks implementation details |
| Factory inside TextProcessor | Cleaner than `new` | Constructor still runs immediately; DIP still violated |
| **Proxy class** | **Clean separation, follows DIP (mostly), client unchanged** | **One extra class per concrete type** |

### Why Accept DIP Violation in Proxy?

The proxy is **intentionally coupled** to one specific concrete class. It is a dedicated wrapper. It will never need to work with a different type. The DIP exists to prevent problems from future changes -- but this proxy is designed to change only when its specific target class changes, which is expected and correct behavior.

### Flyweight: Reference Sharing vs. Cloning

| Flyweight (sharing) | Prototype (cloning) |
|---|---|
| One object in memory, many references | Many independent copies in memory |
| Intrinsic state must be immutable | Each clone can be modified independently |
| Massive memory savings | No memory savings |
| Use when objects share heavy identical data | Use when you need independent copies with slight variations |

---

## Important Points and Key Concepts

1. **Dependency Inversion Principle (DIP):** Higher-level modules should not depend on lower-level modules. Both should depend on abstractions. Always inject dependencies through constructors using interfaces.

2. **Eager vs. Lazy Loading:** Eager = create immediately (wastes resources if unused). Lazy = create on first use (saves resources, adds slight latency on first call).

3. **Proxy = Same Interface + Delegation + Extra Logic.** The proxy never writes business logic itself. It always delegates to the real object.

4. **Flyweight = Separate Intrinsic from Extrinsic.** Share intrinsic (heavy, same-for-all) data. Keep extrinsic (unique-per-object) data in each instance.

5. **Structural patterns restructure class relationships** to improve maintainability, extensibility, or memory usage -- without changing existing classes.

6. **The proxy class is typically one-to-one with a concrete class.** `BookParserProxy` wraps `BookParser`. `WikiParserProxy` wraps `WikiParser`. Each proxy handles its specific resource.

7. **In games, the number of unique assets is always much smaller than the number of rendered objects.** 1000 trees on screen might use only 10 unique tree images.

---

## Exam Prep -- Quick-Fire Facts

| Fact | Answer |
|---|---|
| Proxy implements the same _____ as the real object | Interface |
| Proxy creates the real object in its _____ method (lazy loading) | `getInstance()` / first method call |
| Flyweight separates _____ and _____ state | Intrinsic and Extrinsic |
| Intrinsic state is _____ across objects | Shared |
| Extrinsic state is _____ per object | Unique |
| Proxy and Flyweight are _____ design patterns | Structural |
| Singleton, Factory, Prototype, Builder are _____ design patterns | Creational |
| Creating BookParser eagerly is called _____ loading | Eager |
| Creating BookParser only when needed is called _____ loading | Lazy |
| Flyweight uses the same _____, not a clone | Reference |
| Proxy that controls who can call methods is a _____ proxy | Protection |
| Proxy that delays object creation is a _____ proxy | Virtual |
| Proxy that wraps an external API is a _____ proxy | Remote |
| Proxy that stores computed results is a _____ proxy | Caching |
| In Flyweight, the image for all large stones is stored _____ time(s) | Once (1) |
| If 1M stones each have 20KB image, total without Flyweight | ~20 GB |
| Flyweight is named after _____ in games | Weighted objects flying around |

### Pattern Comparison Table

| Aspect | Proxy | Flyweight |
|---|---|---|
| Primary Goal | Control access to an object | Reduce memory usage |
| Implements same interface? | Yes | Not necessarily (often yes) |
| Number of wrapper objects | One proxy per concrete class | One shared object per category |
| When to use | Heavy constructor, access control, remote resources, caching | Many objects with shared heavy attributes |
| Creates new classes? | Yes (proxy classes) | Yes (shared state objects) |
| Changes existing classes? | No | No |

---

## Teacher's Special Insights

### Career Tip: Always Wrap Remote Resources
> "Whenever you are using any remote resource, you should not be directly calling those APIs in your actual code. You should have your own class and you should always be calling the functions of your own class."

This means: NEVER write `httpClient.get("external-api.com/...")` directly in your business logic. Always create a wrapper class that handles retries, timeouts, exceptions, and response formatting.

### Real-World Teaser
> The teacher mentioned that in the **next lecture**, he will share an example from **Snapdeal's codebase** where one of the design patterns solved a very big problem. This suggests proxy or a similar structural pattern was used in production at scale.

### Thumb Rule: Singleton is Not a Universal Solution
> "The singleton design pattern solves only one problem -- restricting the object creation to only one. If there is any other problem, that cannot be solved using the singleton design pattern."

Students often try to use Singleton for everything. It is a common mistake in interviews.

### Thumb Rule: DIP Violation is Acceptable When Intentional and Scoped
> A proxy class having a direct dependency on a concrete class is a conscious, acceptable trade-off because the proxy is purpose-built for that one class and will never need substitution.

### Gaming Insight: How Graphics Work in Real Games
> In multiplayer games (CS2, BGMI, FIFA), assets are heavily reused. In FIFA, thousands of audience members are just 10-20 unique models repeated. Trees, houses, weapons -- all follow the Flyweight pattern. Without this, no game could run on consumer hardware.

### Study Advice from the Teacher
> "Implementation is most important. If you don't implement, if you are just reading, that is not going to stay with you for long. Please ensure that you do the implementations, do experimentations, try out different things on your own. Only then you will be able to solve the questions which will be there in the exam."

---

## Code Examples

### Proxy Pattern -- Complete Implementation

```java
// ===== The Interface (from third-party library) =====
interface ITextParser {
    int getWordCount(String word);
    int getWordCountInContext(String word, String context);
    // ... other NLP methods
}

// ===== Heavy Implementation (from third-party library) =====
class BookParser implements ITextParser {
    private XMLTree xmlTree;
    private List<double[]> vectors;
    private List<String> words;
    private Map<String, Integer> hashMap;

    public BookParser(Text text) {
        // VERY HEAVY CONSTRUCTOR
        // 1. Chunking sentences
        // 2. POS tagging
        // 3. Syntactic parsing
        // 4. Semantic parsing
        // 5. Building XML tree, vectors, word lists, hash maps
        // Takes significant time and memory
    }

    public int getWordCount(String word) {
        // actual implementation using internal data structures
        return hashMap.getOrDefault(word, 0);
    }

    public int getWordCountInContext(String word, String context) {
        // actual implementation
        return 0;
    }
}

// ===== The Proxy Class (OUR code) =====
class BookParserProxy implements ITextParser {
    private BookParser actualParser = null;  // NOT created yet
    private Text book;

    public BookParserProxy(Text book) {
        this.book = book;  // Just store the reference, do NOT create BookParser
    }

    private BookParser getInstance() {
        if (actualParser == null) {
            actualParser = new BookParser(book);  // Created ONLY on first use
        }
        return actualParser;
    }

    @Override
    public int getWordCount(String word) {
        return getInstance().getWordCount(word);  // Delegate
    }

    @Override
    public int getWordCountInContext(String word, String context) {
        return getInstance().getWordCountInContext(word, context);  // Delegate
    }
}

// ===== TextProcessor (OUR code -- depends on abstraction) =====
class TextProcessor implements ITextProcessorService {
    private ITextParser parser;

    public TextProcessor(ITextParser textParser) {
        this.parser = textParser;  // Could be real or proxy -- doesn't matter
    }

    public int f1(String word) {
        return parser.getWordCount(word);
    }

    public int f2(String word, String context) {
        return parser.getWordCountInContext(word, context);
    }
}

// ===== Client Code =====
class Client {
    private ITextProcessorService processor;

    public Client(ITextProcessorService processor) {
        this.processor = processor;
    }

    public void clientFunction1() {
        // Uses processor -- BookParser created lazily on FIRST call
        int count = processor.f1("hello");
    }

    public void clientFunction2() {
        // Does NOT use processor -- BookParser never created if only this runs
        System.out.println("Independent function");
    }
}

// ===== Usage =====
// BEFORE (eager, heavy):
// Client c = new Client(new TextProcessor(new BookParser(book)));

// AFTER (lazy, lightweight):
Client c = new Client(new TextProcessor(new BookParserProxy(book)));
// BookParser is NOT created yet -- only created when clientFunction1() is called
```

### Protection Proxy Example

```java
interface IDocument {
    String read();
    void write(String content);
    void edit(String content);
    void rename(String newName);
}

class PDF implements IDocument {
    public String read() { /* ... */ return content; }
    public void write(String content) { /* ... */ }
    public void edit(String content) { /* ... */ }
    public void rename(String newName) { /* ... */ }
}

class PDFProxy implements IDocument {
    private PDF actualPdf;
    private String userRole;

    public PDFProxy(PDF pdf, String userRole) {
        this.actualPdf = pdf;
        this.userRole = userRole;
    }

    public String read() {
        return actualPdf.read();  // Everyone can read
    }

    public void write(String content) {
        if ("employee".equals(userRole)) {
            actualPdf.write(content);
        } else {
            throw new AccessDeniedException("Only employees can write to PDF");
        }
    }

    public void edit(String content) {
        if ("employee".equals(userRole)) {
            actualPdf.edit(content);
        } else {
            throw new AccessDeniedException("Only employees can edit PDF");
        }
    }

    public void rename(String newName) {
        if ("employee".equals(userRole)) {
            actualPdf.rename(newName);
        } else {
            throw new AccessDeniedException("Only employees can rename PDF");
        }
    }
}
```

### Flyweight Pattern -- Stone Game Example

```java
// ===== Shared intrinsic state =====
class StoneImage {
    private byte[] imageData;  // 20 KB

    public StoneImage(String imagePath) {
        this.imageData = loadImage(imagePath);
    }

    public void render(int x, int y) {
        // Render this image at position (x, y)
    }
}

// ===== Flyweight Factory -- ensures one image per stone type =====
class StoneImageFactory {
    private static Map<String, StoneImage> imageCache = new HashMap<>();

    public static StoneImage getImage(String stoneType) {
        if (!imageCache.containsKey(stoneType)) {
            imageCache.put(stoneType, new StoneImage(stoneType + ".png"));
        }
        return imageCache.get(stoneType);  // Same reference every time
    }
}

// ===== Lightweight stone object (extrinsic state only) =====
class Stone {
    private int x, y;           // 8 bytes  (extrinsic)
    private int speed;          // 4 bytes  (extrinsic)
    private StoneImage image;   // 4-8 byte REFERENCE (points to shared intrinsic data)

    public Stone(int x, int y, int speed, String stoneType) {
        this.x = x;
        this.y = y;
        this.speed = speed;
        this.image = StoneImageFactory.getImage(stoneType);  // Shared!
    }

    public void render() {
        image.render(x, y);  // Each stone renders at its own position
    }

    public void move() {
        this.y += speed;  // Each stone moves independently
    }
}

// ===== Usage =====
// Creating 1 million large stones -- only ONE image in memory
List<Stone> stones = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    stones.add(new Stone(randomX(), randomY(), 5, "large"));
    // All share the SAME StoneImage object for "large"
}
// Memory: ~12 MB (objects) + 20 KB (one image) instead of ~20 GB
```

---

## SOLID Assignment Discussion Summary

The lecture ended with a student presenting a SOLID principles assignment about refactoring a Cafe Data System. Key takeaways from the presentation:

1. **Original Problem:** A `checkout` function that mixed menu lookup, pricing, tax rules, discount rules, printing, and persistence -- violating SRP.

2. **Solution Applied:**
   - Split into separate classes: `TaxRulesInterface` (with `StudentTaxRules`, `StaffTaxRules`), `DiscountRulesInterface`, `Resolver` (maps customer type to correct policies), `InvoiceService`, `InvoiceFormatter`
   - Database abstracted behind an interface (was a concrete `FileStore` class)
   - Menu lookup separated
   - Logger class for printing

3. **Teacher's Presentation Tip:** "No need to write code during presentations. Draw boxes, write class names and function names, then explain how you broke down responsibilities. Writing code consumes too much time."

---

*Next lecture: Two more structural design patterns + a real-world example from Snapdeal's codebase + introduction to behavioral design patterns.*
