# Observer Design Pattern

> **Also known as:** Event-Subscriber, Listener  
> **Category:** Behavioral Pattern  
> **One-liner:** *"Don't call us, we'll call you."*

---

## 1. The Core Intuition

Imagine you're waiting for a package. You have two options:

- **Polling (bad):** Call the delivery company every 30 minutes → *"Is it here yet?"*
- **Observer (good):** Give them your number → they call YOU when it arrives

That's the entire pattern. **The subject (publisher) maintains a list of interested parties (subscribers) and notifies them automatically when something changes.**

The key insight: **the publisher doesn't need to know *who* is listening or *what they'll do* with the event — it just announces it.**

---

## 2. The Problem It Solves

### Without Observer — the "polling nightmare"

```java
// Every component has to manually check the store's state
class PriceTracker {
    void run() {
        while (true) {
            double price = store.getIphonePrice(); // keep asking!
            if (price < 999) sendAlert();
            Thread.sleep(5000); // wasteful
        }
    }
}
```

Problems:
- Tight coupling: `PriceTracker` must hold a reference to `Store` and know its API
- Wasteful: checks even when nothing changed
- Not scalable: every new component needs its own polling loop

### Without Observer — the "god object" anti-pattern

```java
class Store {
    void setProductAvailable(Product p) {
        this.product = p;
        emailService.sendToAll(p);      // now Store knows about EmailService
        priceTracker.update(p);         // and PriceTracker
        analyticsService.record(p);     // and Analytics
        mobileApp.pushNotification(p);  // and MobileApp
    }
}
```

Problems:
- Store is now coupled to every consumer — violates SRP and OCP
- Add a new consumer? Modify Store. Remove one? Modify Store again.

---

## 3. The Solution — Structure

```
┌─────────────────────────────────┐
│         <<interface>>           │
│           Publisher             │
│  + subscribe(Subscriber)        │
│  + unsubscribe(Subscriber)      │
│  + notifySubscribers()          │
└───────────────┬─────────────────┘
                │ implemented by
┌───────────────▼─────────────────┐
│        ConcretePublisher        │
│  - subscribers: List<Subscriber>│
│  - state: SomeState             │
│                                 │
│  + subscribe(s) { list.add(s) } │
│  + unsubscribe(s){list.remove(s)}│
│  + notifySubscribers() {        │
│      for s in list: s.update()  │
│    }                            │
│  + businessLogic() {            │
│      // change state            │
│      notifySubscribers()        │
│    }                            │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│         <<interface>>           │
│           Subscriber            │
│  + update(context)              │
└───────────────┬─────────────────┘
                │ implemented by
       ┌────────┴──────────┐
       ▼                   ▼
 ConcreteSubA         ConcreteSubB
 + update(ctx)        + update(ctx)
   { log it }           { send email }
```

### The 4 Participants

| Role | Responsibility | Example |
|---|---|---|
| **Publisher (Subject)** | Holds state, manages subscriber list, fires notifications | `Store`, `StockMarket`, `UIButton` |
| **Subscriber interface** | Declares `update()` — the contract | `EventListener`, `Observer` |
| **Concrete Subscriber** | Does something useful on notification | `EmailAlerter`, `Logger`, `Analytics` |
| **Client** | Wires them together at runtime | `Application`, `main()` |

---

## 4. Concrete Java Example — Stock Price Alert

```java
// === SUBSCRIBER INTERFACE ===
interface StockObserver {
    void update(String ticker, double newPrice);
}

// === PUBLISHER ===
class StockMarket {
    private Map<String, List<StockObserver>> observers = new HashMap<>();
    private Map<String, Double> prices = new HashMap<>();

    public void subscribe(String ticker, StockObserver observer) {
        observers.computeIfAbsent(ticker, k -> new ArrayList<>()).add(observer);
    }

    public void unsubscribe(String ticker, StockObserver observer) {
        observers.getOrDefault(ticker, Collections.emptyList()).remove(observer);
    }

    public void setPrice(String ticker, double price) {
        prices.put(ticker, price);
        notifyObservers(ticker, price);  // triggers all subscribers
    }

    private void notifyObservers(String ticker, double price) {
        List<StockObserver> list = observers.getOrDefault(ticker, Collections.emptyList());
        for (StockObserver o : list) {
            o.update(ticker, price);    // publisher doesn't care WHO is listening
        }
    }
}

// === CONCRETE SUBSCRIBERS ===
class PriceAlertSubscriber implements StockObserver {
    private String name;
    private double threshold;

    PriceAlertSubscriber(String name, double threshold) {
        this.name = name;
        this.threshold = threshold;
    }

    @Override
    public void update(String ticker, double newPrice) {
        if (newPrice < threshold) {
            System.out.println("[ALERT] " + name + ": " + ticker + " dropped below " + threshold);
        }
    }
}

class PortfolioLogger implements StockObserver {
    @Override
    public void update(String ticker, double newPrice) {
        System.out.println("[LOG] " + ticker + " is now $" + newPrice);
    }
}

// === CLIENT — wires it all together ===
class Main {
    public static void main(String[] args) {
        StockMarket market = new StockMarket();

        StockObserver alice = new PriceAlertSubscriber("Alice", 150.0);
        StockObserver logger = new PortfolioLogger();

        market.subscribe("AAPL", alice);
        market.subscribe("AAPL", logger);

        market.setPrice("AAPL", 160.0); // logger fires, alice doesn't (above threshold)
        market.setPrice("AAPL", 140.0); // both fire

        market.unsubscribe("AAPL", alice);
        market.setPrice("AAPL", 130.0); // only logger fires now
    }
}
```

**Output:**
```
[LOG] AAPL is now $160.0
[LOG] AAPL is now $140.0
[ALERT] Alice: AAPL dropped below 150.0
[LOG] AAPL is now $130.0
```

Notice: `StockMarket` never imports or knows about `PriceAlertSubscriber` or `PortfolioLogger` — it only knows the `StockObserver` interface.

---

## 5. Push vs Pull Model

This is a subtle but important design decision.

### Push Model (Publisher sends data)
The publisher pushes the relevant data as arguments to `update()`.

```java
// Push
interface Observer {
    void update(double newPrice, double oldPrice, String ticker); // publisher decides what to send
}
```
**Pros:** Subscriber is simple — data arrives ready to use.  
**Cons:** If different subscribers need different data, the `update()` signature becomes bloated.

### Pull Model (Subscriber fetches data)
The publisher passes itself (or a minimal context object), and the subscriber pulls what it needs.

```java
// Pull
interface Observer {
    void update(StockMarket source); // subscriber fetches what it needs
}

// Inside Concrete Subscriber:
public void update(StockMarket source) {
    double price = source.getPrice("AAPL");
    double volume = source.getVolume("AAPL"); // subscriber decides what to fetch
}
```
**Pros:** Flexible — each subscriber fetches only what it needs.  
**Cons:** Subscriber is coupled to the publisher's interface.

### Real-world default: Hybrid
Send a small context object with the most relevant data, but also include a reference to the publisher for anyone who needs more.

```java
void update(StockEvent event) {
    // event has: ticker, newPrice, timestamp, and a reference to the StockMarket
}
```

---

## 6. Where You've Already Seen This

| Context | Publisher | Subscriber | Event |
|---|---|---|---|
| Java `ActionListener` | `JButton` | `ActionListener` impl | button click |
| Android Views | `View` | `OnClickListener` | tap |
| Java `PropertyChangeListener` | Any bean | `PropertyChangeListener` | field change |
| Spring Events | `ApplicationEventPublisher` | `@EventListener` methods | custom events |
| RxJava/Reactor | `Observable/Flux` | `Subscriber/Consumer` | data stream item |
| DOM Events (JS) | `document.getElementById(...)` | callback passed to `addEventListener` | click, keyup, etc. |
| Database triggers | DB engine | trigger function | INSERT/UPDATE |
| Git hooks | git | hook scripts | pre-commit, post-push |

The Observer pattern is arguably the most widely deployed pattern in all of software — you use it every time you write `button.setOnClickListener(...)`.

---

## 7. SOLID Principles Analysis

### Open/Closed Principle ✅
- Add a new subscriber (`SMSAlertSubscriber`) → **zero changes to publisher**
- The publisher is closed for modification, open for extension via the interface

### Single Responsibility Principle ✅
- Publisher only manages state + subscription list
- Each subscriber handles its own response logic

### Dependency Inversion Principle ✅
- Publisher depends on the `Subscriber` *interface*, not concrete classes
- High-level module (publisher) doesn't depend on low-level modules (concrete subscribers)

### Liskov Substitution ✅
- Any `ConcreteSubscriber` can replace any other — publisher treats them all identically

---

## 8. Pros and Cons

### Pros
- **Loose coupling:** Publisher and subscribers are decoupled — they communicate through an interface only
- **Dynamic relationships:** Subscribe/unsubscribe at runtime
- **OCP-compliant:** Add new subscribers without touching publisher code
- **Broadcast communication:** One event → many reactions simultaneously

### Cons
- **Unpredictable order:** Subscribers are notified in list order (or randomly) — don't rely on ordering
- **Memory leaks:** If a subscriber never unsubscribes, the publisher holds a reference → prevents GC
  ```java
  // Classic leak: subscriber is "dead" but publisher still holds reference
  editor.subscribe("save", new HeavySubscriber()); // nobody ever calls unsubscribe!
  ```
- **Cascading updates:** If subscriber A's `update()` triggers publisher B, which notifies subscriber C... you can get unexpected chains or even infinite loops
- **Debugging is hard:** When something goes wrong, tracing through a chain of observers is painful — the flow is implicit, not explicit
- **No guarantee of delivery:** If publisher fires and subscriber throws an exception, other subscribers may not be notified (depends on implementation)

---

## 9. Implementation Gotcha — Thread Safety

In multi-threaded environments, the subscriber list can be modified while `notifySubscribers()` is iterating it → `ConcurrentModificationException`.

**Fix: Iterate over a snapshot copy**
```java
private void notifyObservers(String ticker, double price) {
    List<StockObserver> snapshot = new ArrayList<>(observers.getOrDefault(ticker, List.of()));
    for (StockObserver o : snapshot) {  // iterate copy, not the live list
        o.update(ticker, price);
    }
}
```

Or use `CopyOnWriteArrayList` which handles this automatically (good for read-heavy, write-rare scenarios).

---

## 10. Observer vs Related Patterns

### Observer vs Mediator
Both decouple objects from each other.

| | Observer | Mediator |
|---|---|---|
| Communication | One-to-many (one publisher → many subscribers) | Many-to-many (any component → mediator → any other) |
| Knowledge | Publisher knows nothing about subscribers | Mediator knows about all components |
| Control flow | Distributed (each subscriber reacts independently) | Centralized (mediator orchestrates) |
| Use when | "Broadcast this event to all interested parties" | "Route messages between N components that shouldn't know about each other" |

**Analogy:** Observer = a radio station broadcasting to any listener. Mediator = an air traffic controller that coordinates between planes.

> Note: Mediator is often *implemented using* Observer — the mediator is the publisher and components are subscribers.

### Observer vs Event Bus / Message Queue
An Event Bus is essentially a global Observer where you don't need a direct reference to the publisher.

```java
// Observer: you need the publisher reference
publisher.subscribe(subscriber);

// Event Bus: fully decoupled through a bus
EventBus.subscribe("PRICE_CHANGE", subscriber);
EventBus.publish("PRICE_CHANGE", data);
```

The Event Bus trades directness for even looser coupling — but makes tracing harder.

### Observer vs Callback
A callback is essentially Observer with a single subscriber. When you pass a single function to be called later, that's a one-subscriber Observer without the subscription management infrastructure.

---

## 11. Interview-Ready Pitch (30 seconds)

> *"Observer is a behavioral pattern where a publisher maintains a list of subscribers and notifies them automatically when its state changes. The key is that publisher and subscribers are decoupled through a subscriber interface — the publisher only knows the interface, not the concrete classes. This satisfies OCP: you can add new subscribers without modifying the publisher. Classic use cases are UI event listeners, reactive streams, and notification systems. The main tradeoffs are unpredictable notification order and potential memory leaks if subscribers don't unsubscribe."*

---

## 12. When to Use vs When NOT to Use

### Use Observer when:
- One object's state change should trigger reactions in multiple other objects
- You don't know at compile time which objects need to react (dynamic subscription)
- You want to avoid tight coupling between the "source of truth" and its consumers
- Classic: GUI event handling, real-time feeds, pub/sub systems

### Don't use Observer when:
- There's only ONE subscriber that never changes → just call the method directly
- You need guaranteed ordering of subscriber execution
- The notification chain could become circular or cause infinite loops
- Performance is critical and the overhead of a list traversal + virtual dispatch matters

---

## 13. Real-World LLD Scenario — Movie Booking Seat Hold Expiry

This connects directly to your movie ticket booking LLD work.

When a seat is held (TTL mechanism), multiple components need to react when the hold expires:

```java
interface SeatHoldObserver {
    void onHoldExpired(String holdId, ShowSeat seat);
}

class SeatHoldManager {  // Publisher
    private List<SeatHoldObserver> observers = new ArrayList<>();

    public void subscribe(SeatHoldObserver o) { observers.add(o); }

    // Called by a scheduler/TTL thread when hold expires
    public void expireHold(String holdId) {
        ShowSeat seat = holdStore.get(holdId);
        seat.setStatus(SeatStatus.AVAILABLE);  // release the seat
        observers.forEach(o -> o.onHoldExpired(holdId, seat));
    }
}

class SeatAvailabilityUpdater implements SeatHoldObserver {
    public void onHoldExpired(String holdId, ShowSeat seat) {
        // Update seat status in DB → seat becomes bookable again
    }
}

class UserNotificationService implements SeatHoldObserver {
    public void onHoldExpired(String holdId, ShowSeat seat) {
        // Notify the user their hold expired via email/push
    }
}

class WaitlistProcessor implements SeatHoldObserver {
    public void onHoldExpired(String holdId, ShowSeat seat) {
        // Offer the newly freed seat to next person in waitlist
    }
}
```

Each concern is cleanly separated — adding a new reaction (e.g., analytics logging) means adding one new subscriber class, zero changes to `SeatHoldManager`.

---

## 14. Check Your Understanding — Problems

### Conceptual Questions
1. **Why is the Subscriber interface crucial?** What would break if the publisher held direct references to `EmailAlertsListener` instead of `EventListener`?

2. **Push vs Pull:** In the stock market example, which model would you choose if different subscribers need very different data about each price change? Justify your answer.

3. **Memory Leak:** You create an anonymous inner class subscriber and register it with a publisher. When does this become a memory leak? How do you fix it?

4. **Ordering:** Your system has 3 subscribers to a "payment processed" event: `InventoryUpdater`, `EmailReceiptSender`, `AnalyticsLogger`. Does it matter what order they run in? What if `InventoryUpdater` throws an exception — should the other two still run?

### Design Problems

**Problem 1 — Newspaper Subscription System**
Design a newspaper subscription system where:
- A `Newspaper` can publish different sections: SPORTS, TECH, POLITICS
- Subscribers can subscribe to specific sections (not all)
- A subscriber can subscribe to multiple sections
- Draw the class diagram and write the key Java interfaces/classes

**Problem 2 — Event Filtering**
You have a `UserActivityPublisher` that fires events of types: LOGIN, LOGOUT, PURCHASE, VIEW.
A `FraudDetector` subscriber only cares about PURCHASE events.
How do you avoid calling `FraudDetector.update()` for LOGIN/LOGOUT/VIEW events?
Implement a solution — there are at least two valid approaches.

**Problem 3 — Unsubscribe Puzzle**
```java
button.addOnClickListener(new OnClickListener() {
    public void onClick() {
        System.out.println("Clicked!");
        // How do you unsubscribe THIS listener from inside update()?
    }
});
```
This is a real problem in Android development. What's the issue and how do you solve it?

**Problem 4 — LLD Extension**
In the Movie Booking system: a `ShowSeat` can transition through states: AVAILABLE → HELD → BOOKED → CANCELLED.
Using Observer, design a system where:
- Booking service gets notified on any state change
- Analytics gets notified only when state becomes BOOKED
- Waitlist only cares about AVAILABLE transitions
Without filtering logic inside the publisher. Hint: think about what the "event type" should carry.

---

## 15. Quick Reference Card

```
Pattern:     Observer (Behavioral)
Also known:  Event-Subscriber, Listener, Pub-Sub (informal)

Core idea:   Publisher fires → all Subscribers react
             Publisher knows only the Subscriber INTERFACE

Key roles:   Publisher     → maintains list, fires notify()
             Subscriber    → interface with update()
             ConcreteSubX  → actual reaction logic
             Client        → wires them at runtime

Coupling:    Publisher ──→ <<Subscriber interface>> ←── ConcreteSubscriber
             (publisher never imports concrete subscriber)

OCP:         Add new subscribers without changing publisher ✅
SRP:         Each subscriber has one job ✅
DIP:         Publisher depends on abstraction, not concretion ✅

Pitfalls:    - Memory leaks (unsubscribe!)
             - Random notification order
             - Cascading event chains
             - Thread safety (snapshot the list)

In Java:     java.util.Observer (deprecated in Java 9)
             PropertyChangeListener / PropertyChangeSupport
             EventListener (Swing/AWT)
             Flow.Publisher / Flow.Subscriber (Java 9 reactive)
```