# LLD Frameworks & Interview Guide

A self-contained, step-by-step system for designing robust low-level designs from scratch — and performing well in design interviews.

---

# PART 1: The LLD Design Framework

> Follow these steps in order. Each step builds on the previous one. By the end, you will have a design you can defend with confidence.

---

## Step 0: Understand the Requirements

This is the most important step. A design built on misunderstood requirements is worthless no matter how elegant it is.

### Ask the Right First Question

Before anything else, ask: **"How will the client use this system? What is the API?"**

This single question anchors your entire design. If you're designing a parking lot, the answer might be `park(vehicle)`, `exit(ticket)`, `getStatus()`. If you're designing a pen, it's `pen.write()`, `pen.open()`, `pen.close()`. Everything flows from here.

### Classify Your Questions

Not all questions are equal. Before asking a clarifying question, classify it:

| Type | Definition | Should You Ask? | Example |
|------|-----------|-----------------|---------|
| **Design-impacting** | Changes the class structure or relationships | Yes, always | "Can there be different types of payments?" |
| **Implementation-impacting** | Changes where code lives or which data structure to use | Yes, when relevant | "Should pricing be dynamic or fixed?" |
| **Irrelevant / Over-specific** | Doesn't affect the design at all | No, skip it | "How many parking floors exactly?" |

**Rule of thumb**: Ask *generic* questions, not *specific* ones. Ask "Can there be different types of X?" instead of "How many types of X are there?" The generic question reveals whether variation exists; the specific number doesn't affect your class structure.

### Separate Functional from Non-Functional

- **Functional**: What the system does (park a car, book a ticket, send a notification)
- **Non-functional**: How well it does it (thread safety, latency, scalability, data format independence)

Write both down. Non-functional requirements often drive pattern choices (concurrency → synchronization, multiple formats → Adapter, etc.).

### Output of This Step
- A clear list of **APIs** (how the client interacts with the system)
- A list of **functional requirements** (what it does)
- A list of **non-functional requirements** (constraints and qualities)
- Any assumptions you've made (state them explicitly)

---

## Step 1: Identify Entities & Attributes

Now that you know what the system does, figure out **what things exist** in the system.

### The Noun-Verb-Adjective Technique

Go through your requirements line by line:

- **Nouns** → Candidate classes or attributes
  - "A *parking lot* has multiple *floors*, each floor has *slots* of different *types*"
  - Candidates: `ParkingLot`, `Floor`, `Slot`, `SlotType`
- **Verbs** → Candidate methods
  - "The system should *park* a vehicle and *generate* a ticket"
  - Candidates: `park()`, `generateTicket()`
- **Adjectives / Qualifiers** → Candidate attributes or enums
  - "Slots can be *small*, *medium*, or *large*"
  - Candidate: `SlotType` enum with `SMALL, MEDIUM, LARGE`

### Entity vs Attribute Decision

Not every noun deserves its own class. Apply this filter:

- **Does it have behavior (methods)?** → It's probably a class
- **Does it have its own attributes?** → It's probably a class
- **Is it just a label or value with no behavior?** → It's probably an attribute or enum

Example: In a movie booking system, `Seat` has a position, type, and availability state — it's a class. But `SeatType` (GOLD, SILVER, PLATINUM) is just a label — it's an enum.

### Identify the Core Entities First

Start with 4-6 core entities. Don't try to find everything at once. You'll discover more entities as you assign behaviors and relationships.

### Output of This Step
- A list of **classes** with their key attributes
- A list of **enums** for fixed categories
- A rough sense of which classes are "big" (many attributes/behaviors) vs "small"

---

## Step 2: Identify Relationships

Now connect your entities. There are four types of relationships, and choosing correctly matters.

### The Relationship Decision Tree

```
Are A and B related?
│
├── Does A contain B as an attribute?
│   ├── NO → ASSOCIATION (A uses B, but doesn't own it)
│   │         Example: Game uses Dice, but Dice isn't inside Game
│   │         Arrow: A ──── > B
│   │
│   └── YES → Does B survive if A is destroyed?
│       ├── NO → COMPOSITION (B dies with A)
│       │         Example: Game-Board, Game-Player (board doesn't exist without the game)
│       │         Arrow: A ◆────── B (filled diamond)
│       │
│       └── YES → AGGREGATION (B lives independently)
│                   Example: Playlist-Song (song exists without playlist)
│                   Arrow: A ◇────── B (empty diamond)
│
└── Is B a specialization of A?
    └── YES → INHERITANCE
              Example: Eagle extends Bird
              Arrow: B ──▷ A (empty arrowhead)
```

### Inheritance Depth Rule

If your inheritance hierarchy goes deeper than 2 levels, **redesign**. Deep hierarchies are brittle and hard to extend. Prefer composition over inheritance when in doubt.

### Output of This Step
- A relationship map between your entities
- Clear ownership: who contains whom
- Inheritance hierarchies (kept shallow)

---

## Step 3: Assign Behaviors

Now decide **which class owns which method**. This is where most design mistakes happen.

### The Ownership Principle

A method belongs to the class that **has the data needed to execute it**.

- `calculateBill()` belongs to `Booking` (it has seat info, show info, pricing)
- `isAvailable()` belongs to `Slot` (it knows its own state)
- `findNearestSlot()` belongs to `Floor` or `ParkingLot` (they know the slot layout)

### Collection Management Rule

**The container manages its collection, not the element.**

- `addSong()` belongs to `Playlist`, not `Song`
- `addPlayer()` belongs to `Game`, not `Player`
- `addSlot()` belongs to `Floor`, not `Slot`

If you find yourself putting `addToPlaylist()` inside `Song`, stop — you've inverted the ownership.

### Behavior Delegation

If a class is doing too much, **delegate** part of its work to a helper:

- `Game.makeTurn()` delegates move logic to `MoveStrategy`
- `Booking.calculateBill()` delegates pricing to `PricingStrategy`
- `ParkingLot.park()` delegates slot finding to `SlotFinder`

This keeps classes focused and sets you up for the SOLID validation in the next step.

### Output of This Step
- Every method assigned to a specific class
- Helper classes identified for delegated behaviors
- A rough class diagram taking shape

---

## Step 4: Validate with SOLID (The Confidence Checklist)

This is where you gain confidence. Run each SOLID principle as a **litmus test** against your design. If any test fails, you know exactly what to fix.

### S — Single Responsibility Principle

**Test**: "Does this class have more than one reason to change?"

If `Game` handles both turn logic AND board rendering AND score tracking, it has three reasons to change. Extract `TurnManager`, `BoardRenderer`, `ScoreTracker`.

**Calibration**: Don't over-apply SRP. A class that has 3 closely related methods is fine. SRP means one *responsibility* (a cohesive area), not one *method*. Over-splitting leads to class explosion, which is its own problem.

### O — Open/Closed Principle

**Test**: "If I add a new type or variant, do I need to modify existing, tested code?"

If adding a new `VehicleType` requires editing `ParkingLot.park()` with a new `if-else` branch, OCP is violated. Instead, use polymorphism — each vehicle type knows its own parking rules.

**Fix**: Replace type-based `if-else` chains with polymorphism or Strategy pattern.

### L — Liskov Substitution Principle

**Test**: "Can every child fully replace its parent without breaking anything?"

If `Penguin extends Bird` but `Penguin.fly()` throws an exception, LSP is violated. The parent promised flying capability; the child broke that promise.

**Fix**: Restructure the hierarchy. Use interfaces (`Flyable`) instead of forcing all birds to fly.

### I — Interface Segregation Principle

**Test**: "Is any class forced to implement methods it doesn't need?"

If `Robot implements Worker` and `Worker` has `eat()`, the Robot is forced to implement a meaningless method.

**Fix**: Split into lean interfaces (`Workable`, `Eatable`). A class should only implement interfaces whose methods it actually needs.

### D — Dependency Inversion Principle

**Test**: "Am I depending on concrete classes instead of abstractions?"

If `NotificationService` directly creates `new EmailSender()`, it's coupled to email. Adding SMS requires modifying the service.

**Fix**: Depend on `NotificationChannel` interface. Inject the concrete implementation via constructor (Dependency Injection).

**Key Distinction**:
- **Dependency Inversion** = the principle (depend on abstractions)
- **Dependency Injection** = the technique (pass dependencies via constructor)
- **Inversion of Control** = the framework mechanism (Spring, Guice create and inject for you)

### Output of This Step
- A list of SOLID violations found (if any)
- Specific fixes for each violation
- **Confidence**: if your design passes all 5 tests, it's solid

---

## Step 5: Detect Pattern Triggers & Apply

Design patterns are **solutions to recurring problems**. Don't start with a pattern and look for a problem. Start with a problem, and the pattern will reveal itself.

### The Pattern Trigger Map

Use this as a lookup table. When you see a trigger in your requirements, reach for the corresponding pattern.

| Trigger (What you see in requirements) | Pattern | SOLID Principle It Fixes |
|-----------------------------------------|---------|--------------------------|
| Multiple algorithms/strategies for the same task (e.g., different pricing, different difficulty modes) | **Strategy** | OCP — add new strategies without modifying existing code |
| Need to add features/behaviors dynamically at runtime (e.g., toppings on pizza, notification channels) | **Decorator** | OCP — wrap objects to add behavior without subclassing |
| Expensive object creation or need to control/delay access (e.g., lazy loading images, access control) | **Proxy** | SRP — separate access control from business logic |
| Two incompatible interfaces must work together (e.g., legacy system integration, third-party API) | **Adapter** | OCP — adapt without modifying either side |
| Many objects share heavy common state (e.g., game sprites, document characters with fonts) | **Flyweight** | Memory optimization — share intrinsic state |
| One object's state change must notify many listeners (e.g., stock price alerts, event systems) | **Observer** | OCP — add new listeners without modifying the subject |
| Complex object with many optional fields, especially if immutable | **Builder** | Clean construction — avoid telescoping constructors |
| Only one instance should exist globally (e.g., config manager, connection pool) | **Singleton** | Controlled access to shared resource |
| Client shouldn't know which concrete class is instantiated (e.g., shape creator, vehicle spawner) | **Factory** | DIP — depend on abstraction, not concrete class |
| Constructor is expensive (DB calls, heavy computation) and object can be cloned | **Prototype** | Performance — deep copy is cheaper than reconstruction |

### When NOT to Use a Pattern

**The Hawthorne Effect**: Don't use a pattern just because you know it. Every pattern adds complexity. Use it only when:
1. You can name the **specific problem** it solves in your design
2. The problem is **real and present**, not hypothetical
3. The pattern makes the design **simpler to extend**, not just "more patterny"

### Combining Patterns

Real designs often use multiple patterns together:
- **Strategy + Factory**: Factory creates the right strategy based on input
- **Observer + Adapter**: Wrap legacy listeners with adapters to fit the observer interface
- **Builder + Prototype**: Build a template object, then clone it for variations
- **Decorator + Factory**: Factory creates the right combination of decorators

### Output of This Step
- Patterns identified with clear justification (which trigger, which SOLID principle)
- Updated class diagram with pattern structures

---

## Step 6: Handle Cross-Cutting Concerns

### Concurrency

Ask yourself: **"Can two threads access the same mutable state simultaneously?"**

If yes:
1. **Identify the critical section** — the smallest piece of code that must not run simultaneously
2. **Synchronize only that section**, not the entire method
3. **Choose the right data structure**:
   - Need ordered access + arbitrary deletion? → `TreeSet` (O(log n) for all ops)
   - Need only min/max? → `PriorityQueue` (O(log n) insert/remove-min, but O(n) arbitrary delete)
   - Need O(1) lookup by key? → `HashMap`
4. **Consider optimistic approaches**: temporary locks with timeout (e.g., seat reservation during payment)

Common concurrency scenarios:
- Two users booking the same seat → lock the seat during payment, release on timeout
- Two cars entering the same parking slot → synchronize slot assignment
- Multiple observers being notified → consider async notification

### Data Format Independence

Your classes should not know how input arrives (JSON, XML, CSV, database). Use parsers at the boundary; core classes work with objects only.

### Extensibility Check

Mentally ask: **"What follow-up would an interviewer ask?"** Common ones:
- "Now add a new type of X" → your Strategy/Factory should handle this
- "Now add a new feature to X" → your Decorator should handle this
- "Now make it thread-safe" → your synchronization approach should be clear
- "Now support a different data source" → your Adapter/parser layer should handle this

### Output of This Step
- Concurrency strategy documented
- Data flow from external input to internal objects clear
- Ready for follow-up requirements

---

## Step 7: Validate the Final Design

Three tests to confirm your design is good:

### Test 1: The Trace Test

Pick the main use case (e.g., `game.makeTurn()` or `parkingLot.park(vehicle)`) and **trace the entire flow** through your classes:

1. Which class receives the call?
2. Which methods does it call on which other classes?
3. Does the data flow make sense?
4. Does it terminate cleanly?

If you can trace it smoothly, your design works. If you get stuck or confused, there's a gap.

### Test 2: The Extension Test

Pick a realistic new requirement and mentally add it:
- **Good sign**: You only need to add new classes (new Strategy, new Decorator, new Observer)
- **Bad sign**: You need to modify existing classes, especially with new `if-else` branches

### Test 3: The UML Test

Draw a UML class diagram of your design:
- If you can draw it cleanly with clear relationships → the design is clean
- If the diagram is a tangled mess → the design needs simplification

**UML Quick Reference**:
- Class: Box with three sections (name | attributes | methods)
- Access: `+` public, `-` private, `#` protected
- Inheritance: Empty arrowhead → parent
- Association: Simple arrow
- Aggregation: Empty diamond ◇
- Composition: Filled diamond ◆

---

## Anti-Patterns & Red Flags

Watch for these. If you see them in your design, fix immediately.

| Red Flag | What's Wrong | Fix |
|----------|-------------|-----|
| `if-else` or `switch` on object type | Violates OCP; every new type requires editing | Replace with polymorphism |
| Class explosion (2^N classes for N features) | Combinatorial growth from trying to subclass everything | Use Strategy or Decorator |
| Boolean flags controlling behavior (`if (isX) doA() else doB()`) | Hidden type discrimination | Extract into separate classes |
| A class with 10+ methods doing unrelated things | Violates SRP — God class | Split into focused classes with clear responsibilities |
| Modifying working, tested legacy code | Risky and unnecessary | Create Adapters or Wrappers around it |
| Synchronizing entire methods | Performance bottleneck; blocks more than necessary | Narrow synchronization to the critical section only |
| Deep inheritance (3+ levels) | Brittle, hard to understand and extend | Flatten with interfaces and composition |
| Starting with patterns, then finding problems | Cargo cult design; adds complexity for no reason | Start with the problem; let the pattern emerge |
| Element managing its own collection (Song.addToPlaylist()) | Inverted ownership | Move to the container class |
| Depending on concrete classes everywhere | Tight coupling; hard to extend or test | Introduce interfaces (DIP) |

---

## Quick Reference: The 7 Steps at a Glance

```
Step 0: REQUIREMENTS    → What does the system do? What's the API?
Step 1: ENTITIES        → What things exist? (Nouns → Classes)
Step 2: RELATIONSHIPS   → How are they connected? (Composition / Aggregation / Association)
Step 3: BEHAVIORS       → Who does what? (Assign methods to owners)
Step 4: SOLID CHECK     → Run 5 litmus tests. Fix violations.
Step 5: PATTERN MATCH   → See a trigger? Apply the pattern.
Step 6: CROSS-CUTTING   → Concurrency? Extensibility? Data formats?
Step 7: VALIDATE        → Trace, Extend, Draw. If all pass, you're done.
```

---
---

# PART 2: Performing Well in Design Interviews

---

## Understanding the Two Types of Rounds

| Aspect | LLD Interview | Machine Coding Round |
|--------|--------------|---------------------|
| **Requirements** | Abstract, intentionally vague | Well-defined, specific |
| **Focus** | Design quality + extensibility | Working code that runs |
| **Deliverable** | Class diagram + design walkthrough | Complete, executable implementation |
| **Follow-ups** | Yes — to test your design's flexibility | Usually none |
| **Approach** | Top-down (design first, code later) | Bottom-up (get it working, then refine) |
| **What's Evaluated** | Quality of questions, SOLID adherence, communication | Code quality, completeness, time management |

Know which round you're in. The strategies are different.

---

## The Three Things Interviewers Evaluate

1. **Quality of your clarifying questions** — Do you ask the right things? Do you uncover hidden complexity? Do you avoid wasting time on irrelevant details?

2. **Quality of your design** — Does it follow SOLID? Is it extensible? Are patterns used appropriately (not forced)?

3. **Ability to communicate your design** — Can you explain *why* you made each choice? Can you draw a clear diagram? Do you think out loud?

All three matter equally. A perfect design you can't explain is as bad as a bad design you explain perfectly.

---

## Time Management (45-60 Minute Round)

| Phase | Time | What to Do |
|-------|------|-----------|
| **Clarify Requirements** | 5-8 min | Ask design-impacting questions. List APIs. State assumptions. |
| **Identify Entities & Relationships** | 5-7 min | Core entities, enums, and how they connect. |
| **Design V0** | 8-10 min | First pass — assign behaviors, draw initial diagram. |
| **SOLID Validation & Patterns** | 10-12 min | Run the checklist. Apply patterns where triggered. |
| **Discuss & Refine** | 10-15 min | Walk through the design. Handle follow-ups. |
| **Code (if asked)** | Remaining | Write key classes, not everything. Focus on pattern implementations. |

**Critical**: Don't spend 20 minutes on requirements. You need time to design and refine. If in doubt about a requirement, state an assumption and move on.

---

## How to Ask Clarifying Questions

### Good Questions (Design-Impacting)
- "Can there be different types of [entity]?" → Reveals need for polymorphism
- "Can [feature] vary independently?" → Reveals need for Strategy
- "Can [features] be combined?" → Reveals need for Decorator
- "Does [entity] need to notify others when it changes?" → Reveals need for Observer
- "Is [entity] created once or many times?" → Reveals Singleton possibility
- "Can multiple users do this simultaneously?" → Reveals concurrency needs

### Bad Questions (Avoid)
- "How many users will there be?" → Doesn't affect class design
- "What database should I use?" → Implementation detail, not LLD
- "What's the exact format of input?" → Your classes should be format-independent
- "Should I use Java or Python?" → Usually irrelevant to design discussion

### The Golden Rule
**Ask questions that could change your class diagram.** If the answer wouldn't change any class, attribute, method, or relationship — don't ask.

---

## The Iterative Reveal Strategy

Don't present a perfect design all at once. Show your **thinking process**:

### V0: Start Naive
- "Here's my first pass — simple classes, straightforward relationships"
- Deliberately simple. Maybe even has some if-else chains.

### V1: Apply SOLID
- "I notice this violates OCP because adding a new type requires modifying this class. Let me extract it using Strategy."
- Show that you can **identify problems** and **fix them with principles**.

### V2: Handle Follow-Ups
- Interviewer says: "Now add feature X"
- "Since I used Decorator here, I can add this without touching existing code. I just create a new decorator."

**Why this works**: It shows you don't just memorize patterns — you **derive** them from principles. This is far more impressive than presenting a perfect design and saying "I used Strategy here."

---

## Communicating Your Design

### Think Out Loud
- "I'm making this an interface because I can see there might be different implementations..."
- "I'm putting this method in Class A rather than B because A has the data needed..."
- "I'm noticing this could be a concurrency issue, so I'll synchronize this section..."

### Use the Right Vocabulary
- Say "This violates OCP" not "This doesn't feel right"
- Say "I'll use Strategy pattern here" not "I'll make different classes"
- Say "This is composition because B can't exist without A" not "A has B"

### Draw Clean UML
- Use the standard notation (arrowheads, diamonds)
- Label relationships
- Show access modifiers
- Keep it organized — don't let lines cross unnecessarily

---

## Handling Follow-Up Questions

Follow-ups are where interviews are won or lost. The interviewer is testing if your design is truly extensible.

### Common Follow-Ups and How to Handle Them

| Follow-Up | What They're Testing | Good Response |
|-----------|---------------------|---------------|
| "Add a new type of X" | OCP compliance | "I add a new class implementing the interface. No existing code changes." |
| "Add a new feature to X" | Decorator readiness | "I create a new Decorator that wraps the existing object." |
| "Make it thread-safe" | Concurrency awareness | "I identify the shared mutable state here, and synchronize this critical section." |
| "Support a new data format" | Adapter/parser layer | "I create an adapter for the new format. Core logic stays untouched." |
| "What if this operation is expensive?" | Proxy/caching awareness | "I wrap it in a Proxy for lazy loading / caching." |

### The Key Mindset
If the follow-up requires **adding** new classes → your design is good.
If it requires **modifying** existing classes → acknowledge it, explain the trade-off, and refactor.

---

## Common Mistakes That Get Candidates Rejected

These are real failure modes from interviews:

1. **Not asking clarifying questions** — Jumping straight into design without understanding what you're building. This signals you can't handle ambiguity.

2. **Over-engineering from the start** — Using 5 patterns in your first draft. Start simple, improve iteratively. KISS (Keep It Simple, Stupid).

3. **Can't justify decisions** — "I used Strategy because... it felt right" vs "I used Strategy because the pricing algorithm varies independently of the booking logic, and I want to add new pricing strategies without modifying the booking class."

4. **Ignoring concurrency** — If the system is multi-user, you must address thread safety proactively. Don't wait for the interviewer to ask.

5. **Type-based if-else chains** — The single most common design smell. If you have `if (type == A) ... else if (type == B) ...`, refactor immediately.

6. **God classes** — One class doing everything. Shows you don't understand SRP.

7. **Modifying legacy code in follow-ups** — When the interviewer adds a requirement, don't go back and edit existing classes. Use patterns (Adapter, Decorator, new Strategy) to extend.

8. **Not drawing a diagram** — A verbal-only explanation is hard to follow. Always draw. It helps both you and the interviewer.

9. **Spending too long on requirements** — 5-8 minutes max. State assumptions and move forward. You can always revisit.

10. **Freezing on follow-ups** — If your design doesn't handle a follow-up cleanly, don't panic. Acknowledge the limitation, explain what you'd change, and show you understand the trade-off.

---

## The Interview Checklist (Use Before Saying "I'm Done")

Before you present your final design, quickly check:

- [ ] Did I identify the core API (how the client uses the system)?
- [ ] Does every class have a single, clear responsibility?
- [ ] Can I add new types without modifying existing code?
- [ ] Are my interfaces lean (no forced method implementations)?
- [ ] Am I depending on abstractions, not concrete classes?
- [ ] Did I address concurrency if the system is multi-user?
- [ ] Can I trace the main flow through my classes end-to-end?
- [ ] Is my UML diagram clean and readable?
- [ ] Can I justify every pattern I used with a specific problem it solves?
- [ ] Have I kept it as simple as possible (no unnecessary patterns or classes)?

If you can check all of these, present with confidence. Your design is strong.

---

## Final Words

Design is iterative. You will never get it perfect on the first attempt, and interviewers don't expect you to. What they expect is:

1. A **systematic approach** (not guessing)
2. **Awareness of principles** (SOLID, not vibes)
3. **Ability to improve** your own design when you spot a problem
4. **Clear communication** of what you're doing and why

Follow the 7-step framework. Validate with SOLID. Let patterns emerge from triggers. Communicate your reasoning. That's all there is to it.
