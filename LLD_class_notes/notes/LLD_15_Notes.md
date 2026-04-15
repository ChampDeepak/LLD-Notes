# LLD Lecture 15 - Snake & Ladder Design + Parking Lot Concurrency

---

## Table of Contents

1. [Teacher's Design Journey (V0 to Final)](#1-teachers-design-journey-v0-to-final)
2. [Interview Questions & Tips](#2-interview-questions--tips)
3. [Design Decisions](#3-design-decisions)
4. [Important Concepts & Definitions](#4-important-concepts--definitions)
5. [Parking Lot Design Walkthrough](#5-parking-lot-design-walkthrough)
6. [Concurrency & DSA in Parking Lot](#6-concurrency--dsa-in-parking-lot)
7. [Exam Prep - Quick-Fire Facts](#7-exam-prep---quick-fire-facts)
8. [Teacher's Special Insights](#8-teachers-special-insights)
9. [Code Examples](#9-code-examples)

---

## 1. Teacher's Design Journey (V0 to Final)

This is the most critical section. The teacher demonstrates how to iteratively build a design for the **Snake & Ladder** game, starting simple and evolving based on identified problems.

### Step 0: Understand the Two Approaches

| Approach | Description | Best For |
|----------|-------------|----------|
| **Top-Down** | Start with high-level entities, then go into details | Design rounds (class diagram only) |
| **Bottom-Up** | Start with a working main function, then refactor | Machine coding rounds (working product required) |

**Key Insight:** In machine coding rounds, a working product beats a perfect design that doesn't run. In design rounds, start top-down.

---

### Step 1: V0 -- Identify Major Entities

Start by listing the obvious entities from the problem statement:

```
Game
 |-- Board        (composition)
 |-- Player(s)    (composition, one-to-many)
 |-- Dice         (composition)
```

**Board contains:**
```
Board
 |-- Snake    (composition)
 |-- Ladder   (composition)
```

**Rules at V0:**
- Do NOT worry about whether Dice should be Singleton, static, abstract, etc.
- Do NOT add features not in the requirements (e.g., player statistics, chat).
- Just draw the high-level entities and their relationships.

**Why composition for all?**
- Board is created only for one Game; if Game is deleted, Board is deleted.
- Players are created for this specific game (no login/profile system in requirements).
- Snakes and Ladders are created specifically for a given Board.

---

### Step 2: V0 -- The Core Game Loop

Every game boils down to a **while loop**:

```
while (game not over):
    player = queue.poll()          // pick next player
    player makes a turn             // roll dice, apply rules
    decide: keep player or remove   // winner check
    queue.add(player)              // back to queue (if not winner)
```

**Goal:** This while loop should do NO work itself. It should only **delegate** to other classes/methods. When requirements change, we change those classes, not the loop.

---

### Step 3: V0 -- Data Structure for Players

**Question:** How to store players so they take turns one by one?

| Option | Pros | Cons |
|--------|------|------|
| List | Simple | Need modulo logic to cycle through |
| **Queue** | Natural FIFO, pop and push back | Perfect fit for round-robin turns |

**Answer: Use a Queue.** Pop a player, they take their turn, push them back. First-in, first-out matches round-robin perfectly.

---

### Step 4: V0 Problem -- Make Move Logic Is In the While Loop

When a player rolls a 6, they get another turn. This creates nested logic (a loop within the loop). Problems:
- The while loop is doing all the work itself.
- Not delegating responsibility.
- Changes to move rules require modifying the core loop.

**Fix:** Create a **separate class** for move logic.

```
MakeMoveStrategy
 |-- method: makeMove(currentPosition) -> finalPosition
```

This class:
- Calls the Dice to generate a number.
- Handles the "what if 6" logic internally.
- Returns the final position to the caller.

**Where does this class live?**
- In Snake & Ladder: inside the **Game** class (all players share the same strategy).
- In Chess (hypothetical): inside the **Player** class (human vs AI have different strategies).

---

### Step 5: V1 -- Problem: Two Game Modes Violate SOLID

The requirement says two modes:
- **Easy:** Player keeps rolling on consecutive 6s.
- **Hard:** Three consecutive 6s = player loses their turn.

Having one `makeMove` method with if-else for game type violates:
- **Single Responsibility Principle** -- one method handling multiple behaviors.
- **Open/Closed Principle** -- adding a new difficulty means modifying existing code.
- **Dependency Inversion Principle** -- concrete class injected into another concrete class.

**Fix: Strategy Pattern**

```
<<interface>> MakeMoveStrategy
    + makeMove(position: int) -> int
         |
    +----|----+
    |         |
ContinuousTurnStrategy    SkipTurnStrategy
(easy mode: keep rolling)  (hard mode: 3 sixes = skip)
```

- Game class holds a reference to the **interface**, not a concrete class.
- Inject the appropriate strategy at game creation time.
- Game is now agnostic about which strategy is used.

---

### Step 6: V1 -- Where to Handle Snake & Ladder Resolution

**Key Design Insight:** Snake and Ladder are visually different but structurally identical.

| Attribute | Snake | Ladder |
|-----------|-------|--------|
| Start point | Head (top) | Bottom |
| End point | Tail (bottom) | Top |
| Behavior | Move to end point | Move to end point |

Both have: `start` and `end`. If player lands on `start`, move to `end`. **Same behavior, same attributes.**

**Where to resolve?** The **Board** class.
- Board has a map of positions to entities.
- Board calls: `board.resolvePosition(finalPosition)` -> returns actual final position.
- Board is agnostic of entity type (snake or ladder). It just calls the entity's method.

**Should Snake and Ladder share an interface?**

Ask the interviewer: "Will new board entities be added in the future (e.g., bombs, portals)?"
- **If yes:** Create a common interface (e.g., `BoardEntity` with `getEndPosition()`). This follows Dependency Inversion.
- **If no:** Keep them as concrete classes. Just tell the interviewer you know a principle is violated and how to fix it.

---

### Step 7: V1 -- Edge Case: Snake/Ladder After Every 6 or Only at Final Position?

Two interpretations:
1. Snake/Ladder checked **only at the final position** after all 6s are rolled.
2. Snake/Ladder checked **after each individual roll**.

**Impact on design:**
- If only at final position: resolution stays in the main loop (delegated to Board).
- If after each roll: resolution must move **inside** the `makeMove` strategy method.

**This must be clarified with the interviewer.**

---

### Step 8: V1 -- Winner Detection

When a player reaches cell N^2:
- One version: exact landing required (if overshoot, stay put).
- Another version: any landing >= N^2 = win.

This logic can be folded into `board.resolvePosition()` -- treating the winning cell like another positional event (similar to snake/ladder).

**A student suggested:** Instead of separate snake/ladder resolution and win-check, merge them into one `resolvePosition` method that handles all positional events. Teacher agreed this is clean.

**Important rule:** Functions that modify a collection of objects should NOT live in the object's class. The class holding the collection (Game holds the player queue) should manage adding/removing from it.

---

### Step 9: V1 -- Object Creation with Factory

The client should NOT call constructors directly. Use a **Factory**.

```
GameFactory
 |-- creates Game
      |-- needs BoardFactory to create Board
           |-- needs SnakeLadderGenerationStrategy to place entities
```

**Board generation can vary:**
- Random placement
- Rule-based (one snake per row, alternating, etc.)

So `BoardFactory` uses a **SnakeLadderGenerationStrategy** (another Strategy Pattern).

**Inside the factory:**
1. Create an empty Board object.
2. Use the generation strategy to populate snakes and ladders.
3. Return the fully constructed Board.

**Immutability of Board:**
- Once created, the board never changes during a game.
- Board should be an **immutable class**.
- Use **Builder Pattern** to construct immutable Board inside the factory.
- If short on time, skip immutability -- just mention it to the interviewer.

---

### Step 10: Final Design -- Rules Class

When a game has multiple strategies (move strategy, winner strategy, player-picking strategy), instead of cluttering the Game class:

```
Rules
 |-- MakeMoveStrategy makeMoveStrategy
 |-- WinnerDecidingStrategy winnerStrategy
 |-- PlayerPickingStrategy playerPickStrategy
```

Game holds one `Rules` object. The while loop calls:
```
rules.makeMove(...)
rules.decideWinner(...)
rules.pickNextPlayer(...)
```

Different game configurations inject different Rules objects with different strategy combinations.

---

## 2. Interview Questions & Tips

### Common Interview Questions from This Lecture

1. **How would you design a Snake & Ladder game?** (Full design problem)
2. **How do you handle two different game modes with different rules?** (Strategy Pattern)
3. **Where should the make-move logic live -- Game, Player, Board, or Dice?** (Depends on requirements)
4. **Should Snake and Ladder share an interface?** (Depends on future scope)
5. **How do you handle concurrency in a parking lot system?** (Thread safety, critical sections)
6. **What data structure would you use for finding the nearest parking slot?** (TreeSet / Self-balancing BST)
7. **What is the time complexity of deleting an arbitrary element from a priority queue?** (O(n), not O(log n))
8. **How do you make the board immutable?** (Builder Pattern)

### What Interviewers Look For

| Good Sign | Red Flag |
|-----------|----------|
| Discussing requirements first | Jumping straight to design |
| Asking clarifying questions about scope | Adding features not in requirements |
| Starting simple, evolving design | Starting with complex design patterns |
| Knowing when NOT to apply a pattern | Applying patterns everywhere "just in case" |
| Identifying concurrency issues proactively | Ignoring thread safety |
| Knowing time complexities | Guessing data structure performance |

---

## 3. Design Decisions

### Why Queue over List for Players?

Players take turns in round-robin order. A Queue naturally supports poll (get next) and offer (add back). A List requires maintaining a pointer and modulo arithmetic.

### Why Strategy Pattern for MakeMove?

Two game modes (easy/hard) need different behaviors for the same action. Strategy lets us inject behavior at runtime without if-else chains in the game logic.

### Why Board Resolves Snakes/Ladders (Not Game)?

Board **knows** which cells have entities. Board has the map. Game shouldn't need to know internal board layout. This follows the principle of **information expert** -- the class with the data should have the behavior.

### Why NOT a Jump Strategy for Snake/Ladder?

Snake and Ladder are not "strategies" -- they always do the same thing (move player from start to end). There's no variation in behavior. No need for a strategy when there's only one behavior.

### Why Factory + Strategy for Board Generation?

Board generation is not the Board's responsibility (a class should not create its own objects). Different placement algorithms (random, rule-based) are variations of the same task -- classic Strategy Pattern use case.

### When to Violate Principles Intentionally

The teacher explicitly says: if violating Dependency Inversion for Snake/Ladder (no interface) doesn't cause problems given current requirements, and there's no future scope for new entities, it's OK to skip. **Just tell the interviewer you know it's a violation and how to fix it.**

---

## 4. Important Concepts & Definitions

### KISS Principle

**Keep It Simple, Stupid.** Do not over-engineer. Your design should be:
- Extensible
- Readable
- Maintainable
- Following SOLID (to a reasonable extent)

But NOT more complex than the requirements demand.

### Composition vs Aggregation vs Association

| Relationship | Lifetime Dependency | Example in This Design |
|-------------|-------------------|----------------------|
| **Composition** | Child dies when parent dies | Game-Board, Game-Player, Board-Snake |
| **Aggregation** | Child can exist independently | Not used here (players don't have independent profiles) |
| **Association** | Just a reference | Board-Player (player has a position on board) |

### When Strategy Parameters Differ

If one strategy's method needs parameters A, B and another needs C, D (different signatures), two solutions:
1. **Pass the entire parent object** as the parameter (strategy extracts what it needs).
2. **Store the parent object inside the strategy** (injected via constructor), so the method takes no arguments.

This is the same pattern used in **Observer** when different observers need different data from the subject.

### Immutable Class

A class whose state cannot change after construction. In Java:
- All fields are `final` and `private`.
- No setters.
- Use **Builder Pattern** to construct.

Board is a candidate for immutability since it never changes during a game.

---

## 5. Parking Lot Design Walkthrough

A student presented the Parking Lot design. Teacher praised it as "the type of diagram expected in an interview."

### Entities

```
ParkingLot (Singleton)
 |-- List<ParkingLevel>         (composition)
 |-- List<EntryGate>            (composition)
 |-- FareCalculator
 |-- SlotAssignmentStrategy

ParkingLevel
 |-- Map<SlotType, List<ParkingSlot>> slotMapping

ParkingSlot
 |-- slotId
 |-- slotType (SMALL, MEDIUM, LARGE)
 |-- Map<EntryGate, Distance> distanceToGates
 |-- isAvailable

EntryGate
 |-- gateId

FareCalculator
 |-- Map<SlotType, HourlyRate>
 |-- calculateFare(slotType, duration) -> amount

SlotAssignmentStrategy (interface)
 |-- findSlot(slotType, entryGate) -> ParkingSlot
      |
 NearestSlotStrategy    RandomSlotStrategy (future)
```

### APIs

```java
Receipt park(Vehicle vehicle, LocalDateTime entryTime, SlotType type, EntryGate gate)
double exit(Receipt receipt)
```

### Enums

```java
enum SlotType { SMALL, MEDIUM, LARGE }
enum VehicleType { TWO_WHEELER, CAR, TRUCK }
```

---

## 6. Concurrency & DSA in Parking Lot

### The Concurrency Problem

Multiple entry gates can assign the **same slot** to two vehicles simultaneously.

**Solution:** Make the slot-finding method **thread-safe**.
- Do NOT synchronize the entire method (too coarse).
- Identify the **critical section** (the part where a slot is read and marked as taken).
- Lock only that section.

### The Hidden DSA Problem: Finding Nearest Slot

Each gate needs to find the nearest available slot. Three approaches discussed:

#### Approach 1: Sorted List -- O(n) per query
- Simple. Iterate through all slots, find nearest available.
- Acceptable in some interviews.

#### Approach 2: Priority Queue (Min-Heap) per Gate -- Problems

Each gate has a priority queue of slots sorted by distance.

| Operation | Time Complexity |
|-----------|----------------|
| Pop minimum (nearest) | O(log n) |
| Delete arbitrary element (slot booked from another gate) | **O(n)** -- must rebuild heap |

**Lazy deletion optimization:**
- Don't delete from other gates' queues.
- Mark slot as `unavailable` in the object itself.
- When popping, skip unavailable slots.
- **Worst case:** O(n log n) -- if all slots are unavailable, must pop all to find one.
- **Amortized:** O(log n) -- but worst case is still bad.

#### Approach 3: TreeSet (Self-Balancing BST) per Gate -- OPTIMAL

```
TreeSet<ParkingSlot> (sorted by distance to this gate)
```

| Operation | Time Complexity |
|-----------|----------------|
| Find minimum (nearest) | O(log n) |
| Delete arbitrary element | **O(log n)** |
| Search | O(log n) |

**Why TreeSet works:**
- TreeSet in Java is backed by a **Red-Black Tree** (self-balancing BST).
- All operations are O(log n) guaranteed.
- No need to implement balancing yourself.

**Key Interview Point:**

> "What is the time complexity of deleting an arbitrary element from a priority queue?"
>
> **Answer: O(n)**, not O(log n). Deleting the root is O(log n), but finding and deleting any other element requires a linear scan + rebuild. This is a common mistake in interviews.

---

## 7. Exam Prep - Quick-Fire Facts

### Design Patterns Used in This Lecture

| Pattern | Where Used | Why |
|---------|-----------|-----|
| **Strategy** | MakeMoveStrategy (easy/hard game modes) | Different behaviors for the same action |
| **Strategy** | SnakeLadderGenerationStrategy (random/rule-based) | Different board generation algorithms |
| **Strategy** | SlotAssignmentStrategy (nearest/random) | Different slot-finding algorithms |
| **Factory** | GameFactory, BoardFactory | Object creation separated from business logic |
| **Builder** | Board construction (mentioned, optional) | Creating immutable objects |
| **Singleton** | ParkingLot | Only one parking lot instance |

### SOLID Principles Referenced

| Principle | Violation Caught | Fix Applied |
|-----------|-----------------|-------------|
| SRP | makeMove handling both easy and hard logic | Split into two strategy classes |
| OCP | Adding new difficulty = modifying makeMove | Strategy Pattern |
| DIP | Concrete strategy injected into Game | Introduced interface |
| DIP | Concrete Snake/Ladder in Board | Add interface (only if future scope demands) |

### Key Data Structures

| Data Structure | Use Case | Key Property |
|---------------|----------|-------------|
| Queue | Player turn order | FIFO = round-robin |
| Map | Cell -> Entity (snake/ladder) | O(1) lookup |
| Priority Queue | Nearest slot (basic) | O(log n) min, O(n) arbitrary delete |
| TreeSet | Nearest slot (optimal) | O(log n) for ALL operations |

### Common Mistakes (From Student Submissions)

1. Too many classes for no reason.
2. Using multiple design patterns unnecessarily.
3. Adding features not in requirements (player statistics, chat).
4. Not discussing requirements before designing.
5. Board class generating its own objects.
6. Putting collection-management logic inside the element class.

---

## 8. Teacher's Special Insights

### The "Two-Hour Rule"

> In an interview, you get at most 2 hours: discuss requirements, create design, AND implement a working product. If your design is too complicated, you waste time and fail.

### The KISS Principle Story

> For people who tend to over-engineer: it's called "Keep It Simple, **Stupid**." Don't add complexity that requirements don't demand.

### Real Interview Rejection Story

> A senior software engineer candidate was asked to design tic-tac-toe. Instead of focusing on the game, they started discussing:
> - Players should communicate with each other (chat).
> - Need to filter abusive messages.
> - Need an AI model to blur abusive content.
>
> **Result: Rejected.** The candidate went completely off-track from the requirements.

### The "Good Questions vs Bad Questions" Framework

| Question Type | Example | Impact |
|--------------|---------|--------|
| **Design-impacting** (ask these!) | "Can there be different types of strategies?" | Changes class structure |
| **Implementation-impacting** (ask these) | "Should snake/ladder resolve after each roll or only at final position?" | Changes where code lives |
| **No-impact (AVOID)** | "How many players?" / "What color is the board?" | Concrete answers don't change design |

> **Thumb Rule:** Ask **generic** questions, not **specific** ones.
> - BAD: "How many colors are possible?"
> - GOOD: "Can there be different colors?"
> - BAD: "How many strategies can there be?"
> - GOOD: "Can there be different types of strategies?"
>
> The number doesn't matter. Whether it's 4, 5, or 8, your design stays the same.

### When to Apply Design Patterns

> "Do not start with design patterns. Start with a V0 design. Evaluate it. If there are problems, fix them with patterns. If there are no problems, leave it alone."

### When to Skip a Principle

> "If you have time, implement it. If you don't, skip it. Just let the interviewer know you're aware of the violation and how to fix it."

### Proactive Concurrency Identification

> "If you can point out concurrency issues on your own, it adds extra points. If the interviewer has to point it out, you should at least be able to solve it."

### Client vs Interviewer

> "You can suggest features to a client. But you should NOT suggest features to an interviewer. With an interviewer, you only design for the given requirements."

---

## 9. Code Examples

### Game While Loop (Pseudocode -- Final Version)

```java
class Game {
    Queue<Player> playerQueue;
    MakeMoveStrategy makeMoveStrategy;  // injected via constructor
    Board board;

    void play() {
        while (playerQueue.size() > 1) {
            Player player = playerQueue.poll();

            // Step 1: Make move (handles dice rolls, consecutive 6s)
            int finalPosition = makeMoveStrategy.makeMove(player.getPosition());

            // Step 2: Resolve board entities (snake, ladder, win)
            int actualPosition = board.resolvePosition(finalPosition);

            // Step 3: Update player
            player.setPosition(actualPosition);

            // Step 4: Check winner
            if (actualPosition == board.getLastCell()) {
                System.out.println(player.getName() + " wins!");
            } else {
                playerQueue.offer(player);  // back to queue
            }
        }
    }
}
```

### Strategy Pattern for MakeMove

```java
interface MakeMoveStrategy {
    int makeMove(int currentPosition);
}

class ContinuousTurnStrategy implements MakeMoveStrategy {
    private Dice dice;

    public int makeMove(int currentPosition) {
        int totalMoves = 0;
        int roll;
        do {
            roll = dice.roll();  // returns 1-6
            totalMoves += roll;
        } while (roll == 6);
        return currentPosition + totalMoves;
    }
}

class SkipOnTripleSixStrategy implements MakeMoveStrategy {
    private Dice dice;

    public int makeMove(int currentPosition) {
        int totalMoves = 0;
        int consecutiveSixes = 0;
        int roll;
        do {
            roll = dice.roll();
            if (roll == 6) {
                consecutiveSixes++;
                if (consecutiveSixes == 3) {
                    return currentPosition;  // lose turn, stay put
                }
                totalMoves += roll;
            } else {
                totalMoves += roll;
                break;  // non-six ends the turn
            }
        } while (true);
        return currentPosition + totalMoves;
    }
}
```

### Board Entity Resolution

```java
class Board {
    Map<Integer, BoardEntity> entityMap;  // position -> snake or ladder
    int lastCell;  // n * n

    int resolvePosition(int position) {
        if (position > lastCell) {
            return position - (position - lastCell);  // bounce back (or stay)
        }
        if (entityMap.containsKey(position)) {
            return entityMap.get(position).getEndPosition();
        }
        return position;
    }
}

// Common interface for board entities
interface BoardEntity {
    int getEndPosition();
}

class Snake implements BoardEntity {
    int start, end;  // start > end (goes down)
    public int getEndPosition() { return end; }
}

class Ladder implements BoardEntity {
    int start, end;  // start < end (goes up)
    public int getEndPosition() { return end; }
}
```

### Parking Lot -- TreeSet for Nearest Slot

```java
class EntryGate {
    int gateId;
    TreeSet<ParkingSlot> slotsByDistance;  // sorted by distance to this gate

    // O(log n) -- find and remove nearest available slot
    ParkingSlot findNearestAvailable() {
        ParkingSlot nearest = slotsByDistance.first();  // O(log n)
        return nearest;
    }

    // O(log n) -- remove a slot booked from another gate
    void removeSlot(ParkingSlot slot) {
        slotsByDistance.remove(slot);  // O(log n) in TreeSet!
    }
}
```

### Priority Queue vs TreeSet Comparison

```
Priority Queue (Min-Heap):
  - findMin():    O(log n)  [poll]
  - deleteMin():  O(log n)
  - deleteAny():  O(n)      <-- PROBLEM

TreeSet (Red-Black Tree):
  - findMin():    O(log n)  [first()]
  - deleteMin():  O(log n)  [pollFirst()]
  - deleteAny():  O(log n)  [remove()]  <-- SOLVED
```

---

## Summary of Design Evolution

```
V0: Identify entities (Game, Board, Player, Dice, Snake, Ladder)
    |
    v
V0 Validation: While loop does too much work
    |
    v
V1: Extract MakeMoveStrategy (delegation)
    |
    v
V1 Validation: Two game modes break SRP/OCP/DIP
    |
    v
V2: Strategy Pattern for MakeMoveStrategy interface
    |
    v
V2 Validation: Where does snake/ladder logic go?
    |
    v
V3: Board.resolvePosition() handles entities
    |
    v
V3 Validation: Need interface for BoardEntity? (ask interviewer)
    |
    v
V4: Factory for Game/Board creation + Generation Strategy
    |
    v
V4 Enhancement: Board immutability via Builder (mention, optional)
    |
    v
V5: Rules class bundles multiple strategies (for complex games)
    |
    v
FINAL: Clean, simple, extensible design
```

---

*These notes cover the complete LLD Lecture 15 on Snake & Ladder design evolution and Parking Lot concurrency/DSA optimization.*
