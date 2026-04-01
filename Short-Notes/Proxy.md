# 🪞 Proxy Design Pattern — Interview Pitches

---

## 🎯 One-liner Pitch
> **"Proxy places a substitute object in front of a real object to control access to it — the client never talks to the real object directly; instead, the proxy intercepts the call and decides what to do: forward it, cache it, log it, delay it, or guard it."**

---

## 📌 Core Pitches (Say These Confidently)

- **Category:** Structural design pattern.
- **Problem it solves:** You need to add controlled access, lazy initialization, logging, caching, or security checks around a real object — without modifying its code.
- **Core idea:** Proxy and the real object implement the **same interface**. The client can't tell the difference — it talks to the proxy exactly like it would talk to the real object.
- **Proxy holds a reference** to the real object (RealSubject) and delegates to it — before or after doing its own work.
- The real object does the **actual heavy work**; the proxy wraps it with cross-cutting concerns.
- **Key insight:** The proxy is a "gatekeeper" — it stands between the client and the real object and decides how and when the real object is accessed.

---

## 🧠 Intuition Anchor — "The Bodyguard / Personal Assistant"

> Think of a celebrity (the real object) and their personal assistant (the proxy). Fans (clients) never talk directly to the celebrity. The assistant screens calls, checks credentials, schedules meetings, and sometimes handles things entirely without disturbing the celebrity. The celebrity's phone number (interface) is the same — but you always reach the assistant first.

**The celebrity never changes. The assistant controls access.**

---

## 🔀 Four Types of Proxy — Know All Four

| Proxy Type | What it does | Real-World Analogy |
|---|---|---|
| **Virtual Proxy** | Delays creation of an expensive object until it's actually needed (lazy init) | A movie poster before the film loads |
| **Protection Proxy** | Controls access based on permissions/roles | A security guard at a club |
| **Remote Proxy** | Represents an object that lives in a different address space (network, JVM) | An ambassador representing a foreign country |
| **Caching Proxy** | Caches results of expensive operations; returns cached value on repeat calls | A CDN caching a website's assets |

> **Bonus types sometimes asked:** Logging Proxy (audits calls), Smart Reference Proxy (reference counting, auto-cleanup).

---

## ⚡ When To Use — Interview Triggers

| Signal | Proxy Type |
|---|---|
| "Object is expensive to create, create it only when needed" | Virtual Proxy |
| "Add access control / role checks without touching the class" | Protection Proxy |
| "Object lives on a remote server / different JVM" | Remote Proxy |
| "Same computation called repeatedly, cache results" | Caching Proxy |
| "Log every method call without modifying the class" | Logging Proxy |
| "AOP (Aspect-Oriented Programming)" | Proxy is the mechanism behind it |

---

## 🏭 Real-World Examples (Drop These in Interview)

- **Java RMI (Remote Method Invocation)** — stub is a Remote Proxy; client calls the stub, which serializes and forwards calls across the network.
- **Spring AOP (@Transactional, @Cacheable, @Secured)** — Spring wraps your bean in a dynamic proxy at runtime; every annotated method call goes through the proxy first.
- **Hibernate Lazy Loading** — entity relationships (`@OneToMany`) return a proxy object; the actual SQL query fires only when you access the collection (Virtual Proxy).
- **Java `java.lang.reflect.Proxy`** — built-in dynamic proxy mechanism; used by Spring, Mockito, and dozens of frameworks.
- **Mockito mocks** — a mock IS a proxy; it intercepts method calls and returns stubbed values instead of calling real logic.
- **CGLIB proxies** — used by Spring when the class doesn't implement an interface; bytecode-level proxy.
- **Browser Service Workers** — intercept network requests, serve from cache or forward to server (Caching Proxy).
- **Credit card (Virtual Proxy for bank account)** — you hand over a card, not your bank account; the card proxies access to your funds with its own validation layer.

---

## 🔑 Key Properties (Hard Facts)

- Proxy and RealSubject **must implement the same interface** — this is what makes them interchangeable from the client's perspective.
- Proxy holds a **reference to the RealSubject** — it doesn't inherit behavior, it delegates.
- Client code **never changes** when you introduce a proxy — open/closed principle in action.
- A proxy can **choose not to forward** a call at all (e.g., access denied in Protection Proxy).
- Proxy can be **chained** — a caching proxy wrapping a logging proxy wrapping the real object.
- **Dynamic Proxy** (Java `reflect.Proxy`) creates proxy classes at runtime without writing a new class for each interface.
- Proxy adds **indirection** — slight performance overhead per call; usually worth it for the control it provides.

---

## 👥 Participants of Proxy Design Pattern

- **Subject (Interface)** — defines the common interface that both Proxy and RealSubject implement. Ensures the client can use either interchangeably.
- **RealSubject** — the actual object that does the real work. The thing being proxied.
- **Proxy** — implements the Subject interface, holds a reference to RealSubject, and controls access to it. Adds its own logic before/after delegating.
- **Client** — talks only to the Subject interface. Unaware whether it's speaking to the Proxy or the RealSubject.

```
Client → Subject (interface)
              ↑              ↑
           Proxy --------→ RealSubject
         (controls)         (real work)
```

---

## 💻 Code Skeleton (Java-style Pseudocode)

```java
// Subject — shared interface
interface VideoService {
    Video fetchVideo(String id);
}

// RealSubject — expensive real object
class YouTubeVideoService implements VideoService {
    public Video fetchVideo(String id) {
        // Makes a real HTTP call — slow and costly
        return httpClient.get("https://youtube.com/v/" + id);
    }
}

// Proxy — Caching Proxy example
class CachedVideoService implements VideoService {
    private VideoService realService = new YouTubeVideoService();
    private Map<String, Video> cache = new HashMap<>();

    public Video fetchVideo(String id) {
        // Check cache first — forward to real service only on cache miss
        return cache.computeIfAbsent(id, k -> realService.fetchVideo(k));
    }
}

// Client — doesn't know if it's talking to proxy or real service
VideoService service = new CachedVideoService(); // swap to real service anytime
Video v = service.fetchVideo("abc123"); // first call: hits YouTube
Video v2 = service.fetchVideo("abc123"); // second call: served from cache
```

**Key observation:** Client calls `fetchVideo()` identically on both. The proxy silently intercepts and decides — cache hit or delegate to real service.

---
```java
// Subject
interface ITextParser {
    int getWordCount();

    int getSentenceCount();

    boolean searchWord(String w);

    boolean searchRootWord(String w);
}

// RealSubject ( heavy )
class BookParser implements ITextParser {
    public BookParser(String book) {
        // heavy parsing , POS tagging , XML building ...
        System.out.println(" Heavy BookParser initialized ");
    }

    public int getWordCount() {
        /* ... */ return 1000;
    }

    public int getSentenceCount() {
        /* ... */ return 80;
    }

    public boolean searchWord(String w) {
        return true;
    }

    public boolean searchRootWord(String w) {
        return false;
    }
}

// Proxy ( virtual proxy for lazy loading )
class LazyBookParserProxy implements ITextParser {
    private final String book;
    private BookParser realParser;

    public LazyBookParserProxy ( String book ) {
        this.book = book ;
        this.realParser = null ;
    }

    //It needs to be made thread safe.
    private BookParser getReal() {
        if (realParser == null) {
            realParser = new BookParser(book); // construct only on demand
        }
        return realParser;
    }

    @Override
    public int getWordCount() {
        return getReal().getWordCount();
    }

    @Override
    public int getSentenceCount() {
        return getReal().getSentenceCount();
    }

    @Override
    public boolean searchWord(String w) {
        return getReal().searchWord(w);
    }

    @Override
    public boolean searchRootWord(String w){
        return getReal().searchRootWord(w);
    }
}
 
```
**Key observation:** If we had implemented lazy loading in client then each method would had two responsibilities that are lazy loading + their core logic.

---

## 🆚 Distinguish From Similar Patterns

| Pattern | How it differs from Proxy |
|---|---|
| **Decorator** | Also wraps an object, but its purpose is to *add behavior/features*. Proxy's purpose is to *control access*. A decorator enhances; a proxy guards. |
| **Adapter** | Changes the interface to make incompatible types work together. Proxy keeps the **same interface** as the real object. |
| **Facade** | Simplifies a complex subsystem into one entry point. Proxy has the *exact same interface* as what it hides; Facade has a *different, simpler* interface. |
| **Flyweight** | Shares state to save memory. Proxy controls access to *one* real object (no sharing). |
| **Singleton** | Guarantees one instance. Proxy may use Singleton internally but is a different concern. |

> **Critical interview distinction:** Proxy = same interface + access control. Decorator = same interface + added behavior. They look similar in code but have different *intent*.

---

## 🎤 One-Sentence Answers for Common Interview Questions

| Question | Answer |
|---|---|
| What is Proxy? | A structural pattern that places a substitute in front of a real object to control access to it. |
| What must Proxy and RealSubject share? | The same interface (Subject), so the client can use either interchangeably. |
| What are the four proxy types? | Virtual (lazy init), Protection (access control), Remote (network), Caching (memoize results). |
| How is Proxy different from Decorator? | Same structure, different intent: Proxy controls access; Decorator adds behavior. |
| How is Proxy different from Adapter? | Adapter changes the interface; Proxy keeps the same interface. |
| Where is Proxy used in Spring? | Behind every `@Transactional`, `@Cacheable`, `@Secured` annotation — Spring wraps beans in dynamic proxies. |
| Where is Proxy used in Hibernate? | Lazy-loaded associations return a Proxy object; real SQL fires only on first access. |
| What is a Dynamic Proxy? | A proxy class created at runtime (via `java.lang.reflect.Proxy`) without writing a dedicated proxy class. |
| What is the cost of Proxy? | One extra method call indirection per invocation; negligible in most cases, but measurable in hot paths. |
| Can proxies be chained? | Yes — a caching proxy can wrap a logging proxy can wrap the real object; each adds its layer. |

---

## 📐 Proxy Type Cheat Sheet

| | Virtual | Protection | Remote | Caching |
|---|---|---|---|---|
| **Purpose** | Lazy init | Access control | Network bridge | Avoid re-computation |
| **Creates real object?** | On first use | Only if authorized | No (it's remote) | On first call |
| **Java example** | Hibernate lazy load | Spring `@Secured` | Java RMI stub | Spring `@Cacheable` |
| **Key check before delegate** | "Has it been created?" | "Is caller authorized?" | "Serialize & send?" | "Is result cached?" |

---

## 🧪 Quick Self-Check Questions

1. You're building an image viewer. Images are huge and take 3 seconds to load from disk. Most images are never opened. Which proxy type do you use and why?
2. Spring's `@Transactional` is implemented via Proxy. Explain what happens when a method annotated with `@Transactional` is called — step by step through the proxy.
3. What is the difference between Proxy and Decorator? They look identical in code — where does the real difference lie?
4. A Protection Proxy throws `AccessDeniedException` for unauthorized users. The client code catches `RuntimeException`. Does this break the pattern? Why or why not?

---

*Pattern Family: Structural | GoF Book: Yes | Frequency in practice: Very High (Spring, Hibernate, RMI, Mockito all use it heavily)*