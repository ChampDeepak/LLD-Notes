## Decorator Design Pattern -- The Full Design Journey

### Problem Statement

You have a notification framework with different notifier types (Email, SMS, WhatsApp, Slack). A client already holds a notifier object. **At runtime**, you need to add extra features (extra notification channels) to that same object **without** modifying existing code.

---

### V0 -- The Naive Thought Process (What NOT To Do)

| Idea | Why It Fails |
|------|-------------|
| Copy-paste the WhatsApp notification code into a new combined class | Same code at two places -- violates DRY principle |
| Create a new object of WhatsApp notifier inside the wrapper and call its `notify()` | Violates Dependency Inversion Principle (DIP) -- though the teacher notes it is "not a very big violation" since the wrapper is specific to WhatsApp, it is still not ideal |

---

### V1 -- The Wrapper Approach (Building Towards Decorator)

**Key insight from the teacher:** We need a class that:
1. **Implements the same `Notifier` interface** -- so the client sees no difference.
2. **Holds an object of `Notifier`** (injected via constructor) -- so it can delegate to whatever notifier was passed in.
3. **Extends the concrete notifier class** (e.g., `WhatsAppNotifier`) -- so it inherits all the notification-sending capabilities without creating a separate internal object.

#### Step-by-Step Construction

**Step 1:** Create a `WhatsAppWrapper` class.

```java
// Step 1: Wrapper implements the same interface
public class WhatsAppWrapper extends WhatsAppNotifier implements Notifier {

    private Notifier wrappedNotifier;  // holds the object being decorated

    // Step 2: Inject any Notifier through constructor
    public WhatsAppWrapper(Notifier notifier) {
        this.wrappedNotifier = notifier;
    }

    // Step 3: First delegate, then add own behavior
    @Override
    public void notify() {
        wrappedNotifier.notify();   // whatever the wrapped object does, do that first
        super.notify();             // then send WhatsApp notification (inherited from WhatsAppNotifier)
    }
}
```

**Why `extends WhatsAppNotifier`?**
- The WhatsApp notifier class may have multiple helper methods needed to actually send a WhatsApp message.
- By extending it, the wrapper gets all those capabilities for free.
- Avoids creating an internal object of `WhatsAppNotifier` (which would violate DIP).

**Step 2:** Create an `SMSWrapper` class in the exact same fashion.

```java
public class SMSWrapper extends SMSNotifier implements Notifier {

    private Notifier wrappedNotifier;

    public SMSWrapper(Notifier notifier) {
        this.wrappedNotifier = notifier;
    }

    @Override
    public void notify() {
        wrappedNotifier.notify();   // delegate first
        super.notify();             // then send SMS
    }
}
```

---

### Final Design -- Chaining Decorators at Runtime

This is where the magic happens. You chain wrappers by passing one wrapper into the constructor of another:

```java
// Start with a base notifier
Notifier n = new EmailNotifier();

// Decorate with WhatsApp
n = new WhatsAppWrapper(n);   // now n does: Email + WhatsApp

// Decorate with SMS
n = new SMSWrapper(n);        // now n does: Email + WhatsApp + SMS

// Single call triggers all three
n.notify();
```

#### Execution Flow (Call Stack Trace)

```
n.notify()                          // SMSWrapper.notify()
  |
  +-- wrappedNotifier.notify()      // WhatsAppWrapper.notify()
  |     |
  |     +-- wrappedNotifier.notify() // EmailNotifier.notify()
  |     |     --> SENDS EMAIL
  |     |
  |     +-- super.notify()
  |           --> SENDS WHATSAPP
  |
  +-- super.notify()
        --> SENDS SMS
```

**Result order:** Email --> WhatsApp --> SMS

---

### Class Diagram

```
          <<interface>>
            Notifier
           + notify()
          /    |     \
         /     |      \
EmailNotifier  SMSNotifier  WhatsAppNotifier
                |                |
           SMSWrapper      WhatsAppWrapper
           - wrappedNotifier: Notifier
           + notify()
```

**Key structural rules:**
- Every wrapper **implements** the common `Notifier` interface (so clients are agnostic).
- Every wrapper **extends** its corresponding concrete class (to reuse notification-sending logic).
- Every wrapper **holds a `Notifier` reference** (to delegate to the wrapped object).

---

### Design Decisions -- Why Decorator?

| Question | Answer |
|----------|--------|
| Why not just add boolean flags for each channel? | Booleans lead to if-else chains. Adding a new channel means modifying existing code (violates OCP). |
| Why not use a list of notifier types? | Could work in some cases, but Decorator gives you **behavioral composition** -- each wrapper can modify/extend behavior, not just toggle it. |
| When should you use Decorator? | When you need to **add features at runtime** to an existing object, and those features involve **behavior changes**, not just attribute changes. |

---

### Teacher's Critical Warning About Decorator

> "Decorator is NOT a pattern you will use at a lot of places. You have to be very careful about where exactly you are using the decorator design pattern."

The teacher explicitly says:
- Many times the job can be done with **dynamic attributes** (a list) or **simple boolean flags**.
- Evaluate the need first. Only use Decorator when simpler approaches fail.
- Most of the time, you can write maintainable, extensible code without Decorator.

**Contrast with Proxy:** The teacher mentioned Proxy is used "very, very frequently in multiple use cases," but Decorator is more niche.

---

### Real-World Examples of Decorator (from the teacher)

1. **Gaming -- Character Powers:**
   - A character starts with basic abilities.
   - After earning points, the character unlocks new powers (longer jump, fire ability, etc.).
   - Same object, new features at runtime --> pass the object through a new wrapper constructor.

2. **Gaming -- Stone Game:**
   - A stone object can become a "fiery stone" or start "spitting smaller stones."
   - These are **behavioral changes**, not just new attributes.

3. **Notifications (teaching example only):**
   - The teacher explicitly warns: "In reality, notifications are triggered in an async way. They are pushed to a queue like Kafka. This is just an example."

