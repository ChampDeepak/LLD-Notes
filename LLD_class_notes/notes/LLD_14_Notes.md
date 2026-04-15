# LLD Lecture 14 - Detailed Notes

## Topics Covered
1. **Snake and Ladder Game Design** - Full class diagram, relationships, and design evolution
2. **Strategy Design Pattern** - Applied to game difficulty modes
3. **Composition vs Aggregation vs Association** - Deep dive with real examples
4. **Multi-Level Parking Lot Design** - Classic interview problem with full requirements gathering

---

## Part 1: Snake and Ladder Game Design

### 1.1 The Teacher's Design Journey (V0 to Final)

#### V0 - Understanding the Flow First (Before Writing Any Class)

The teacher emphasizes: **before creating classes, understand the flow of the application.**

| Step | Action | Description |
|------|--------|-------------|
| 1 | Add Players | Client adds players to the game |
| 2 | Start the Game | Game loop begins |
| 3 | Toss the Dice | Current player rolls the dice |
| 4 | Move the Piece | Player moves by the dice value |
| 5 | Check Position | Check if landed on a snake or ladder |
| 6 | Apply Jump | If snake/ladder exists, move accordingly |
| 7 | Check Win | Has the player reached the end? |
| 8 | Next Turn | Move to the next player |

#### V0 - Identifying Classes from the Flow

From the flow above, the classes naturally emerge:

| Flow Step | Class Identified |
|-----------|-----------------|
| Add players | `Player` |
| Board exists | `Board` |
| Toss dice | `Dice` |
| Snake/Ladder (have common attributes: start + end) | `Jump` (abstract), `Snake`, `Ladder` |
| Overall orchestration | `Game` |

**Key Insight**: Snake has head (start) and tail (end). Ladder has bottom (start) and top (end). Both have a start point and end point. This commonality leads to the `Jump` abstraction.

| Entity | Start | End | Condition |
|--------|-------|-----|-----------|
| Ladder | Bottom rung | Top rung | `end > start` |
| Snake | Head | Tail | `start > end` |

#### V0 - Class Attributes and Methods

```
Player
- name: String
- position: int

Dice
- diceCount: int
+ roll(): int  // returns random number

Board
- size: int
- snakes: List<Jump>
- ladders: List<Jump>
- jumpMap: HashMap<Integer, Jump>   // position -> Jump mapping
+ addSnake(start, end)
+ addLadder(start, end)
+ getJump(position): Jump           // O(1) lookup using HashMap

Jump (abstract)
- start: int
- end: int

Snake extends Jump
Ladder extends Jump

Game
- board: Board
- players: List<Player>
- dice: Dice
+ play()                            // master while loop
```

**Important**: The `Board` uses a **HashMap** for O(1) lookup of whether a position has a snake or ladder. This avoids scanning through all snakes and ladders (O(n)) every time a player moves.

---

### 1.2 Relationship Analysis (Composition vs Aggregation vs Association)

This is one of the most detailed discussions on relationships in this lecture. The teacher walks through every single relationship.

#### Game ---> Player: COMPOSITION

| Question | Answer |
|----------|--------|
| Can Player exist without Game? | No - in this requirement, players are temporary (just a count is given, no login/persistence) |
| If Game is deleted, should Player be deleted? | Yes |
| Verdict | **Composition** |

> **Teacher's Insight**: "The type of relationship is dependent on the requirements." If players had login details, statistics, and persistence across games, it would be **Aggregation** (player outlives the game). But since the requirement says "just provide number of players" with temporary IDs, it is **Composition**.

> **Rule**: It would **never** be a simple association because the player object is going to be *inside* the game.

#### Game ---> Dice: ASSOCIATION

| Question | Answer |
|----------|--------|
| Does Dice hold state? | No - it just generates random numbers |
| Is Dice a utility? | Yes - mostly static methods |
| Does Game need an instance of Dice? | No - just calls `Dice.roll()` |
| Verdict | **Association** (Game *uses* Dice) |

> **Teacher's Insight**: "History of dice rolls should NOT be maintained inside the Dice class. That is not its responsibility." Either have a separate class or maintain a stack of moves in the Game class.

> **What if requirements change?** If dice needs to maintain state or you need multiple types of dice, then it becomes Composition or Aggregation. But currently, it is just a utility.

#### Game ---> Board: COMPOSITION

| Question | Answer |
|----------|--------|
| Is Board associated with exactly one Game? | Yes |
| If Game is deleted, should Board be deleted? | Yes |
| Verdict | **Composition** |

#### Board ---> Jump (Snake/Ladder): COMPOSITION

| Question | Answer |
|----------|--------|
| What differentiates two Board objects? | The placement of snakes and ladders |
| Are snakes/ladders reused across boards? | No - you create new ones for each board |
| If Board is deleted, should its snakes/ladders be deleted? | Yes |
| Verdict | **Composition** |

> **Teacher's Insight**: "If you delete this board and then use this snake in a different board - that won't happen. For that you will be creating a new snake."

#### Snake/Ladder ---> Jump: INHERITANCE

- `Jump` can be an abstract class or interface
- `Snake` and `Ladder` extend/implement `Jump`

#### Final Class Diagram Summary

```
Game ------[composition]------> Player
Game ------[association]------> Dice
Game ------[composition]------> Board
Board -----[composition]------> Jump (abstract)
Snake -----[inherits]---------> Jump
Ladder ----[inherits]---------> Jump
```

---

### 1.3 Design Evolution: Handling Difficulty Levels

#### Requirements Added
- Client creates game via a **Factory** providing: board size, player count, difficulty level
- **Easy Mode**: Consecutive sixes - keep playing (no penalty)
- **Hard Mode**: After 3 consecutive sixes, player loses their turn

#### V1 - Naive Approach (Using Factory + Inheritance on Game)

A student proposes:
1. Create an abstract `CreateGame` factory class
2. Two methods: `makeMove()` (business logic) and `createGame()` (abstract)
3. Concrete classes: `EasyGame` and `DifficultGame`
4. Each overrides `handleSix()` differently

```
AbstractGameFactory
+ makeMove()           // contains business pipeline
+ createGame(): Game   // abstract - implemented by subclasses

EasyGameFactory extends AbstractGameFactory
+ createGame(): Game   // returns EasyGame

DifficultGameFactory extends AbstractGameFactory
+ createGame(): Game   // returns DifficultGame
```

#### Problems with V1 (Teacher's Critique)

| Problem | Explanation |
|---------|-------------|
| **Code Duplication** | Both `EasyGame` and `DifficultGame` have a while loop. Most logic (pick player, check snake/ladder, check win) is SAME. Only `handleSix` differs. |
| **Not Reusable** | Same code written in two places. Any change must be made in both. |
| **SRP Violation in Factory** | Factory should ONLY create objects (call constructor + return). It should not contain business logic. |
| **Inheritance Assumption Flaw** | Assumes difficulty is the ONLY axis of variation. What if winning rules also change? |

**Teacher's Critical Point on Inheritance Assumptions**:
> "Whenever you are using inheritance, you are actually taking an assumption. You need to validate that assumption. The assumption is that difficulty is the ONLY criteria based on which two games can be different."

Example of why this breaks:
- Some games: Must roll exact number to win (at position 97, need exactly 3 to reach 100)
- Other games: Any number >= required is a win
- Now you have 2 difficulty modes x 2 winning rules = **4 classes**
- More criteria = exponential class explosion

#### Inside the While Loop - What is Same vs Different

```
While (game not over) {
    1. Pick player          // SAME in all versions
    2. Roll dice            // SAME
    3. Handle consecutive   // DIFFERENT (easy vs hard mode)
       sixes
    4. Move player          // SAME
    5. Check snake/ladder   // SAME
    6. Check if won         // SAME (but could vary in future)
}
```

Only step 3 (and potentially step 6) varies between game modes.

#### V2 - Final Design: Strategy Pattern

Instead of creating subclasses of Game, extract the varying behavior into a **Strategy**.

```
Game
- board: Board
- players: List<Player>
- moveStrategy: GameModeStrategy    // <-- injected strategy
+ play()

GameModeStrategy (abstract/interface)
+ makeMove(player, diceValue): int  // returns final position

DefaultMode extends GameModeStrategy
+ makeMove(...)                     // default rules

EasyMode extends GameModeStrategy
+ makeMove(...)                     // no penalty for consecutive sixes

HardMode extends GameModeStrategy
+ makeMove(...)                     // lose turn after 3 consecutive sixes
```

**How the While Loop Changes**:

```java
// BEFORE (V1 - duplicated in EasyGame and HardGame)
// Both classes had full while loops with mostly same code

// AFTER (V2 - single Game class with strategy)
while (!gameOver) {
    Player current = pickNextPlayer();          // same always
    int diceValue = Dice.roll();                // same always
    int finalPosition = strategy.makeMove(      // DELEGATED to strategy
        current, diceValue
    );
    checkSnakeOrLadder(finalPosition);          // same always
    checkWinCondition(current);                 // same always (or separate strategy)
}
```

#### Why Strategy Pattern is Superior

| Aspect | Inheritance (V1) | Strategy (V2) |
|--------|-------------------|---------------|
| Code duplication | While loop duplicated in every subclass | Single while loop in Game |
| Adding new mode | Create entire new Game subclass | Create small strategy class |
| Multiple varying axes | Class explosion (m x n classes) | Compose strategies independently |
| Open/Closed Principle | Must modify Game hierarchy | Add new strategy, inject it |
| Single Responsibility | Game subclass handles rules + orchestration | Game orchestrates, Strategy handles rules |

> **Teacher's Insight**: "In future, if you want winning rules to also vary, you can have a separate WinningStrategy. Through that, you check whether the player has won or not."

This means the Game class can have **multiple strategy slots**:

```java
class Game {
    MoveStrategy moveStrategy;           // how to handle moves/sixes
    WinningStrategy winningStrategy;     // how to determine winner
    // ... potentially more strategies in future
}
```

---

## Part 2: Multi-Level Parking Lot Design

### 2.1 This is a Classic Interview Problem

> **Teacher's Note**: "This is one of the most commonly asked interview problems."

### 2.2 Requirements Gathering (Interview-Style Q&A)

The teacher emphasizes asking **specific questions** during requirements gathering. Below is the full Q&A:

| Question | Answer |
|----------|--------|
| How does the client create the parking lot? | Provides: number of floors, slot mapping per level, entry gates, gate-to-slot distances |
| What APIs are needed? | `park()`, `exit()`, `getStatus()` |
| Are there different types of slots? | Yes: Small (2-wheeler), Medium (car), Large (bus) |
| Can a motorbike park in a bus slot? | Yes, but pays the bus slot rate |
| Different prices per vehicle type? | No. Pricing is per **slot type**, not vehicle type |
| Different prices per level? | No. Same rates across all levels |
| Are levels identical? | Arrangement can differ, but behavior is same |
| What if slot not available? | Throw an exception (client should check availability first) |
| How is the nearest slot determined? | Using pre-provided distance data from every gate to every slot, **regardless of level** |
| Can a slot on a different floor be "nearer"? | Yes! A slot directly above an entry gate on floor 2 could be nearer than a slot at the far end of floor 1 |
| Multiple parking lots? | No, single parking lot for now |
| Rules for slot arrangement inside a level? | No rules - it's a physical entity that already exists, you map it to software |

### 2.3 API Specifications

#### API 1: `park(vehicleDetails, entryGate, slotType)`

| Parameter | Description |
|-----------|-------------|
| vehicleDetails | Details of the vehicle (number, type, etc.) |
| entryGate | Which gate the vehicle entered from |
| slotType | SMALL, MEDIUM, or LARGE |

**Returns**: `Ticket` object

```
Ticket
- entryTime: DateTime
- slotType: SlotType
- vehicleDetail: VehicleDetail
- slotId: String
```

**Behavior**: Assigns the **nearest available slot** of the requested type from the given entry gate. If no slot available, throws exception.

#### API 2: `exit(ticket)`

| Parameter | Description |
|-----------|-------------|
| ticket | The ticket generated during parking |

**Returns**: `amount` (money to pay)

**Behavior**: Calculates charges based on `hourly rate of slot type x duration`.

#### API 3: `getStatus(slotType)` (optional parameter)

**Returns**: How many slots of which type are available.

### 2.4 Slot Types and Pricing

| Slot Type | Accommodates | Hourly Rate |
|-----------|-------------|-------------|
| SMALL | Two-wheelers | Rate_S |
| MEDIUM | Cars | Rate_M |
| LARGE | Buses | Rate_L |

**Important**: A motorbike CAN park in a LARGE slot but pays LARGE slot rates. Pricing is by **slot type**, not vehicle type.

### 2.5 Distance and Slot Assignment Logic

- Pre-provided: M-to-N mapping of distances (every gate to every slot)
- Slot assignment is **cross-level** - not restricted to the same floor as the entry gate
- Example: User enters Gate 1 on Ground Floor. Nearest available MEDIUM slot might be on Floor 3 directly above the gate, rather than far end of Ground Floor

### 2.6 Design Principles for Parking Lot

| Principle | Application |
|-----------|-------------|
| **Classes should be agnostic of input format** | Whatever format data comes in, write a parser to convert to your objects |
| **Don't overthink the constructor** | Can create empty parking lot and expose `addLevel()`, `addSlot()`, `addGate()` methods |
| **Levels are optional in class design** | Level is a business requirement; if you don't need it for your API, don't force it |
| **If confused between composition/aggregation** | Mark as aggregation first, change to composition when confident |

> **Teacher's Insight**: "Your classes should be independent of input data format. In whichever format the data is coming, your classes are going to be agnostic of it. If the data does not match, you write a separate class (parser) that will convert this data to the desired format."

> **Teacher's Insight on Constructor Design**: "Instead of taking all of these things in the constructor, you can create an empty parking lot and expose functions for adding level, adding slot. That is going to make your job much more easier."

### 2.7 Parking Lot - Proposed Class Structure

Based on the requirements discussed:

```
ParkingLot
- levels: List<Level>
- entryGates: List<EntryGate>
- distanceMap: Map<(Gate, Slot), Distance>    // gate-to-slot distances
+ park(vehicleDetails, entryGate, slotType): Ticket
+ exit(ticket): double
+ getStatus(slotType): Map<SlotType, Integer>
+ addLevel(level)
+ addGate(gate)

Level
- levelNumber: int
- slots: List<Slot>

Slot
- slotId: String
- slotType: SlotType       // SMALL, MEDIUM, LARGE
- isAvailable: boolean
- hourlyRate: double

EntryGate
- gateId: int
- position: ...

Vehicle
- vehicleNumber: String
- vehicleType: VehicleType

Ticket
- ticketId: String
- entryTime: DateTime
- slot: Slot
- vehicle: Vehicle

SlotType (enum)
- SMALL, MEDIUM, LARGE

VehicleType (enum)
- TWO_WHEELER, CAR, BUS
```

### 2.8 Key Design Decisions for Parking Lot

| Decision | Rationale |
|----------|-----------|
| Distance map is M-to-N (gates x slots) | Enables finding nearest slot from any gate in O(1) lookup or sorted structure |
| Slot type determines pricing, not vehicle type | Simplifies billing - just look at the slot |
| Exception on unavailable slot | Client is expected to check `getStatus()` before calling `park()` |
| Parser class for input data | Decouples internal representation from external data format |
| `addLevel()`, `addSlot()` instead of mega-constructor | Builder-like approach, simpler initialization |

---

## Part 3: Key Concepts and Definitions

### Composition vs Aggregation vs Association - Summary Table

| Relationship | "Has-A" strength | Child lifetime tied to parent? | Example from lecture |
|-------------|-----------------|-------------------------------|---------------------|
| **Composition** | Strong | Yes - child dies with parent | Game -> Board, Game -> Player, Board -> Snake/Ladder |
| **Aggregation** | Medium | No - child can outlive parent | Game -> Player (IF players had persistence/login) |
| **Association** | Weak (uses) | No - independent lifecycle | Game -> Dice (utility, static methods) |

### When to Use Strategy Pattern

Use Strategy when:
- You have a family of algorithms that differ in specific behavior
- The main workflow/pipeline is the same, only certain steps vary
- You want to avoid class explosion from inheritance
- You need to swap behavior at runtime

### Factory Pattern - What It Should Do

> **Teacher's Rule**: "Factory should have exactly one responsibility - creating an object. Just call the constructor with the appropriate parameters and return the object."

A factory should NOT contain business logic.

---

## Interview Questions

### Snake and Ladder
1. **Design a Snake and Ladder game. What classes would you create?**
   - Game, Board, Player, Dice, Jump (abstract), Snake, Ladder
   - Use HashMap in Board for O(1) position lookup

2. **What is the relationship between Game and Player? Composition or Aggregation?**
   - Depends on requirements. If players are temporary (no persistence), it's Composition. If players have accounts and persist across games, it's Aggregation.

3. **How would you handle different difficulty levels in the game?**
   - Use Strategy Pattern. Inject a `GameModeStrategy` into the Game class. Different strategies implement different rules (e.g., handling consecutive sixes).

4. **Why not use inheritance (EasyGame, HardGame) for difficulty levels?**
   - Code duplication (while loop repeated), class explosion when multiple axes of variation exist, violates SRP.

5. **Where should dice roll history be stored?**
   - NOT in the Dice class. Either in a separate class or as a stack of moves in the Game class.

### Multi-Level Parking Lot
6. **Design a multi-level parking lot system.**
   - Key entities: ParkingLot, Level, Slot (Small/Medium/Large), EntryGate, Vehicle, Ticket
   - APIs: park(), exit(), getStatus()

7. **How do you assign the nearest slot?**
   - Pre-computed distance mapping from every gate to every slot (across all levels). Find minimum distance available slot of requested type.

8. **Can a motorbike park in a bus slot?**
   - Yes, but it pays the bus slot hourly rate. Pricing is per slot type, not vehicle type.

9. **What if no slot is available?**
   - Throw an exception. The client should check getStatus() before calling park().

10. **How do you handle input data in different formats?**
    - Write a separate parser class. Your core classes should be agnostic of input format.

---

## Exam Prep - Quick-Fire Facts

| Fact | Detail |
|------|--------|
| HashMap for snake/ladder lookup | O(1) instead of O(n) scanning |
| Dice class is a utility | Static methods, no state, Association with Game |
| Board-Snake relationship | Composition (snakes are characteristic of a specific board) |
| Strategy > Inheritance when | Multiple axes of variation, shared workflow, avoid class explosion |
| Factory's ONLY job | Create object, call constructor, return. No business logic. |
| Parking slot pricing | By slot type (SMALL/MEDIUM/LARGE), NOT by vehicle type |
| Nearest slot assignment | Cross-level, based on pre-computed gate-to-slot distances |
| Slot availability check | Separate `getStatus()` API; `park()` throws exception if unavailable |
| Constructor design tip | Prefer empty object + add methods over mega-constructor |
| Classes should be agnostic of | Input data format - use parsers to convert |

---

## Teacher's Special Insights

### On Determining Relationship Types
> "The type of relationship is dependent on the requirements. This should be composition if we want to maintain the object of players persistently. If you want the player object to be persistent... then when you're deleting a game, you are not going to delete the player object."

**Thumb Rule**: Ask yourself - "If I delete the parent, should the child also be deleted?" If yes, Composition. If child can live independently, Aggregation. If parent just *uses* the child, Association.

### On Inheritance Assumptions
> "Whenever you are using inheritance, you are actually taking an assumption. You need to validate that assumption."

Before creating subclasses, ask: "Is this the ONLY axis of variation?" If there could be other axes, prefer composition/strategy over inheritance.

### On Factory Responsibility
> "Factory should have exactly one responsibility which is creating an object. Just call the constructor with the appropriate parameters and return the object."

Never put business logic in a factory.

### On Data Format Independence
> "Your classes should be independent of this data. You assume what type of data you want. In whatever format you get this data, you convert it to your desired format."

Always write a parser/adapter between external data and your internal classes.

### On Practical Constructor Design
> "Instead of taking all of these things in the constructor, you can create an empty parking lot and you can expose the functions of adding level, adding slot. That is going to make your job much more easier."

Builder-style initialization is cleaner than mega-constructors for complex objects.

### On Class Diagram Tips for Interviews
> "Composition and aggregation, if you are confused with that, you can just mark it as aggregation right now. If you are confident that it's going to be a composition, then change it to composition. But it should be a valid class diagram."

When unsure, default to aggregation (the weaker claim) - it is safer.

### On Dice Roll History
> "History of the dice roll should not be maintained inside the dice class. That is not the responsibility of this class."

This is an SRP example - Dice generates numbers, it does not track history.

### On Physical-to-Software Mapping
> "There is a parking lot which already exists. There is a physical entity. What you need to do is you just need to create a software mapping of that parking lot."

In real-world LLD, you often map existing physical systems to software. Don't invent constraints that don't exist.

---

## Code Examples

### Snake and Ladder - Core Game Loop with Strategy Pattern

```java
// Strategy Interface
interface GameModeStrategy {
    int makeMove(Player player, Dice dice);
}

// Easy Mode - no penalty for consecutive sixes
class EasyModeStrategy implements GameModeStrategy {
    @Override
    public int makeMove(Player player, Dice dice) {
        int totalMoves = 0;
        int roll = dice.roll();
        totalMoves += roll;
        while (roll == 6) {
            roll = dice.roll();
            totalMoves += roll;
        }
        return player.getPosition() + totalMoves;
    }
}

// Hard Mode - lose turn after 3 consecutive sixes
class HardModeStrategy implements GameModeStrategy {
    @Override
    public int makeMove(Player player, Dice dice) {
        int totalMoves = 0;
        int consecutiveSixes = 0;
        int roll = dice.roll();
        totalMoves += roll;

        while (roll == 6) {
            consecutiveSixes++;
            if (consecutiveSixes >= 3) {
                return player.getPosition(); // no move, lost turn
            }
            roll = dice.roll();
            totalMoves += roll;
        }
        return player.getPosition() + totalMoves;
    }
}

// Game Class - single class, no subclasses needed
class Game {
    private Board board;
    private List<Player> players;
    private Dice dice;
    private GameModeStrategy strategy;  // injected

    public Game(Board board, List<Player> players,
                Dice dice, GameModeStrategy strategy) {
        this.board = board;
        this.players = players;
        this.dice = dice;
        this.strategy = strategy;
    }

    public void play() {
        int currentPlayerIndex = 0;
        while (!isGameOver()) {
            Player current = players.get(currentPlayerIndex);

            // Delegate move logic to strategy
            int newPosition = strategy.makeMove(current, dice);

            // Check for snake or ladder (same for all modes)
            Jump jump = board.getJump(newPosition);
            if (jump != null) {
                newPosition = jump.getEnd();
            }

            // Update position
            current.setPosition(newPosition);

            // Check win condition (could also be a strategy)
            if (newPosition == board.getSize()) {
                System.out.println(current.getName() + " wins!");
                break;
            }

            // Next player
            currentPlayerIndex =
                (currentPlayerIndex + 1) % players.size();
        }
    }
}
```

### Board with HashMap Lookup

```java
class Board {
    private int size;
    private Map<Integer, Jump> jumpMap = new HashMap<>();

    public Board(int size) {
        this.size = size;
    }

    public void addSnake(int head, int tail) {
        jumpMap.put(head, new Snake(head, tail));
    }

    public void addLadder(int start, int end) {
        jumpMap.put(start, new Ladder(start, end));
    }

    // O(1) lookup
    public Jump getJump(int position) {
        return jumpMap.getOrDefault(position, null);
    }

    public int getSize() { return size; }
}
```

### Jump Class Hierarchy

```java
abstract class Jump {
    protected int start;
    protected int end;

    public Jump(int start, int end) {
        this.start = start;
        this.end = end;
    }

    public int getStart() { return start; }
    public int getEnd() { return end; }
}

class Snake extends Jump {
    // start (head) > end (tail) - moves player DOWN
    public Snake(int head, int tail) {
        super(head, tail);
        if (head <= tail)
            throw new IllegalArgumentException(
                "Snake head must be greater than tail");
    }
}

class Ladder extends Jump {
    // end (top) > start (bottom) - moves player UP
    public Ladder(int bottom, int top) {
        super(bottom, top);
        if (top <= bottom)
            throw new IllegalArgumentException(
                "Ladder top must be greater than bottom");
    }
}
```

### Parking Lot - Core Structure

```java
enum SlotType { SMALL, MEDIUM, LARGE }

class Slot {
    private String slotId;
    private SlotType type;
    private boolean available;
    private double hourlyRate;
    // getters, setters
}

class EntryGate {
    private int gateId;
}

class Ticket {
    private String ticketId;
    private LocalDateTime entryTime;
    private Slot slot;
    private Vehicle vehicle;
}

class ParkingLot {
    private List<Level> levels;
    private List<EntryGate> gates;
    // gate -> (slot -> distance)
    private Map<EntryGate, Map<Slot, Double>> distanceMap;

    public Ticket park(Vehicle vehicle, EntryGate gate,
                       SlotType slotType) {
        Slot nearest = findNearestAvailable(gate, slotType);
        if (nearest == null) {
            throw new SlotNotAvailableException();
        }
        nearest.setAvailable(false);
        return new Ticket(generateId(), LocalDateTime.now(),
                          nearest, vehicle);
    }

    public double exit(Ticket ticket) {
        long hours = ChronoUnit.HOURS.between(
            ticket.getEntryTime(), LocalDateTime.now());
        double rate = ticket.getSlot().getHourlyRate();
        ticket.getSlot().setAvailable(true);
        return hours * rate;
    }

    public Map<SlotType, Integer> getStatus(SlotType filter) {
        // count available slots, optionally filtered by type
    }

    private Slot findNearestAvailable(EntryGate gate,
                                       SlotType type) {
        // From distanceMap, find slot with minimum distance
        // that matches type and is available
        // Works across ALL levels
    }

    // Builder-style methods
    public void addLevel(Level level) { levels.add(level); }
    public void addGate(EntryGate gate) { gates.add(gate); }
}
```

---

## Summary of Design Patterns Used

| Pattern | Where Applied | Why |
|---------|--------------|-----|
| **Strategy** | Game difficulty modes (EasyMode, HardMode) | Avoid code duplication, class explosion; single Game class with swappable behavior |
| **Factory** | Creating Game objects | Client provides params (size, players, difficulty), factory returns correct Game with correct strategy injected |
| **Composition** | Game->Board, Game->Player, Board->Jump | Child objects are tightly bound to parent lifecycle |
| **Association** | Game->Dice | Dice is a utility, no ownership needed |

---

*End of LLD Lecture 14 Notes*
