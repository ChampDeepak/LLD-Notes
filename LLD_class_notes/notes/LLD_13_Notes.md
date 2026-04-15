# LLD Lecture 13 - UML Class Diagrams, Relationships & Snake and Ladder Design

---

## Table of Contents

1. [Overview](#overview)
2. [Teacher's Design Journey](#teachers-design-journey)
3. [UML Class Diagram Components](#uml-class-diagram-components)
4. [Relationships Between Entities](#relationships-between-entities)
5. [UML Notation and Syntax](#uml-notation-and-syntax)
6. [Snake and Ladder - Requirements & Design Approach](#snake-and-ladder---requirements--design-approach)
7. [Design Process Philosophy](#design-process-philosophy)
8. [Interview Questions](#interview-questions)
9. [Design Decisions](#design-decisions)
10. [Important Points](#important-points)
11. [Exam Prep - Quick Fire Facts](#exam-prep---quick-fire-facts)
12. [Teacher's Special Insights](#teachers-special-insights)
13. [Code Examples](#code-examples)

---

## Overview

This lecture covers two major topics:

1. **UML Class Diagrams** - How to formally represent your LLD designs using standard notation, including entity types, access specifiers, and the four key relationship types.
2. **Snake and Ladder Game Design** - A machine coding round problem. The teacher walks through the requirements, clarifies client interactions, and sets up the design exercise.

---

## Teacher's Design Journey

### Step 0: Inventory of Building Blocks

The teacher begins by taking stock of every component used across ALL previous designs:

| Entity Type       | Description                                    |
|-------------------|------------------------------------------------|
| Class (Concrete)  | Regular classes with attributes and methods     |
| Class (Abstract)  | Cannot be instantiated; meant to be extended    |
| Interface         | Contract that classes implement                 |
| Enum              | Typically an attribute of some class            |
| Record            | Mentioned by students; a data-carrying type     |

> **Key Insight**: "Majorly there are only two entities - classes and interfaces. Enums are going to be an attribute of some of the classes." Objects are the **result** of design, not part of the diagram.

### Step 1: Identify Relationships (More Important Than Entities)

The teacher emphasizes that the **relationships between entities** matter more than the entities themselves. He identifies four relationship types from all previous designs:

1. **Inheritance** (class extends class, or class implements interface)
2. **Association** (loose coupling - entities are related but not contained)
3. **Aggregation** (has-a, but child can exist independently)
4. **Composition** (has-a, child CANNOT exist independently)

### Step 2: Understand Relationships Through Examples

The teacher uses concrete real-world examples to build intuition before touching any code:

**Association Example:**
- Doctor and Patient in a hospital system
- They are related (one doctor treats many patients, one patient sees many doctors)
- But a Patient object is NOT an attribute inside the Doctor class
- Both are fully independent entities

**Aggregation Example:**
- Playlist and Songs
- Songs are part of the playlist (playlist HAS songs)
- If you delete the playlist, the songs still exist in the system
- Proxy, Adapter, Decorator patterns all use aggregation - if you delete the wrapper, the wrapped object still exists

**Composition Example:**
- Board and Snakes/Ladders in Snake and Ladder game
- Snakes/Ladders are part of the board
- If you delete the board, keeping those snakes/ladders makes no sense - they must be deleted too
- Order and OrderItem in e-commerce (Blinkit example)
- An OrderItem is NOT the same as the Item in the catalog. It has specific discount, taxes, price tied to THAT order
- Delete the order, the OrderItem must go too (but the catalog Item remains)

### Step 3: Learn Notation, Then Apply to a Real Problem

After covering notation (see below), the teacher transitions to the Snake and Ladder problem, asking students to design it using everything learned.

### Step 4: V0 Design First, Then Improve

The teacher's prescribed process for the Snake and Ladder problem:

```
V0 (Bad Design) --> Evaluate against SOLID --> Identify violations
    --> Apply design patterns ONLY if needed --> V1 (Improved Design)
    --> Implement the master while loop --> Validate the design
    --> Ask: Is it maintainable? Extensible? --> Final Design
```

---

## UML Class Diagram Components

### Class Representation

A class is drawn as a rectangle divided into two sections:

```
+---------------------------+
|    Attributes Section     |
+---------------------------+
|    Methods Section        |
+---------------------------+
```

### Access Specifier Symbols

| Symbol | Access Level       | Example            |
|--------|--------------------|--------------------|
| `+`    | Public             | `+ name: String`   |
| `-`    | Private            | `- name: String`   |
| `#`    | Protected          | `# name: String`   |
| `~`    | Default (Package)  | `~ name: String`   |

### What to Include in a Class Diagram

| Include                                         | Do NOT Include                        |
|-------------------------------------------------|---------------------------------------|
| Important attributes (name, data type optional)  | Getters and setters (assumed)         |
| Specific methods (e.g., `getInstance()`)        | Obvious boilerplate methods           |
| Relationships between classes                    | Every single attribute                |
| Abstract/Interface labels                        | Access specifiers (nice to have, not critical) |

> **Teacher's Rule**: "You do not need to mention all attributes and all methods. Getters and setters are understood. You only need to mention methods other than getters and setters."

> **Interview Tip**: "Even if you simply specify the method name, that is sufficient. Interviews will not be interested in whether it is public, private, or protected. They would be much more interested in what type of relationships are there between two different entities."

---

## Relationships Between Entities

### The Four Relationship Types

#### 1. Inheritance

- Between two classes (extends) or between a class and an interface (implements)
- Drawn with an **unfilled/empty arrowhead** pointing to the parent

```
+-----------+
|  Parent   |    <--- Write "abstract" here if abstract class
+-----------+        Write "interface" here if interface
     ^
     |        (empty/unfilled arrow)
     |
+-----------+
|   Child   |
+-----------+
```

> **Key Point**: "You don't need to tell the interviewer that this is an abstract class or interface. It should be self-explanatory."

#### 2. Association

- Two entities are **related** but neither contains the other
- One is NOT an attribute of the other
- Drawn with a **simple arrow** (or plain line for bidirectional)

```
+-----------+           +-----------+
|  Doctor   | --------> |  Patient  |    (one-way association)
+-----------+           +-----------+

+-----------+           +-----------+
|  Doctor   | --------- |  Patient  |    (two-way association, no arrows)
+-----------+           +-----------+
```

**Examples**: Doctor-Patient, Laptop-LaptopCover, Person-Building

#### 3. Aggregation

- **Has-a** relationship: one class HAS an object of another class as an attribute
- The child CAN exist independently if the parent is deleted
- Drawn with an **unfilled/empty diamond** on the parent side

```
+-----------+           +-----------+
| Playlist  |<>-------> |   Song    |
+-----------+           +-----------+
  (empty diamond)
```

**Examples**: Playlist-Song, Proxy-RealObject, Adapter-Adaptee, Decorator-Component

#### 4. Composition

- **Has-a** relationship like aggregation, BUT the child CANNOT exist independently
- If parent is deleted, child MUST be deleted (cascade delete)
- Drawn with a **filled/solid diamond** on the parent side

```
+-----------+           +-----------+
|   Board   |<*>------> |   Snake   |
+-----------+           +-----------+
  (filled diamond)

+-----------+           +-----------+
|   Order   |<*>------> | OrderItem |
+-----------+           +-----------+
  (filled diamond)
```

**Examples**: Board-Snake, Board-Ladder, Order-OrderItem

### Decision Framework: Which Relationship Is It?

```
Q1: Does Class A contain an object of Class B as an attribute?
    NO  --> ASSOCIATION
    YES --> Q2

Q2: If you delete the object of Class A, must the object of Class B also be deleted?
    NO  --> AGGREGATION  (child can live independently)
    YES --> COMPOSITION  (child must die with parent)
```

### Summary Table

| Relationship | Contains Object? | Independent Lifecycle? | Arrow Symbol       | Real-World Example       |
|-------------|------------------|------------------------|--------------------|--------------------------|
| Association | No               | Yes                    | Simple arrow/line  | Doctor - Patient         |
| Aggregation | Yes              | Yes (child survives)   | Empty diamond      | Playlist - Song          |
| Composition | Yes              | No (child dies too)    | Filled diamond     | Board - Snake/Ladder     |
| Inheritance | N/A              | N/A                    | Empty arrowhead    | Animal - Dog             |

---

## Snake and Ladder - Requirements & Design Approach

### Requirements (as clarified by the teacher)

1. **Game Creation**:
   - User creates a `Game` object through a **Factory**
   - User provides:
     - `n` - size parameter (board will be n x n = n^2 squares)
     - `n` snakes and `n` ladders
     - Number of players (count only, NOT names/emails)
     - Difficulty level (easy or hard version)

2. **Board Setup**:
   - Snakes and ladders placed **randomly**
   - One square cannot have BOTH a snake AND a ladder
   - Every snake/ladder has a start point and end point
   - **Snake**: start > end (goes down)
   - **Ladder**: start < end (goes up)
   - Start and end must have a **vertical difference of at least 1 row** (cannot be on the same horizontal line)

3. **Gameplay**:
   - Turns are **round-robin** (player 1, player 2, ..., player n, player 1, ...)
   - Only ONE method exposed: `game.makeTurn()`
   - Game auto-identifies whose turn it is
   - Generates random number 1-6 (dice roll)
   - Moves the player accordingly
   - If landing on snake/ladder, move to its end point
   - **If dice shows 6**: player gets another turn

4. **Two Versions (Strategy Pattern candidate)**:
   - **Easy version**: Unlimited consecutive 6s allowed, keep rolling
   - **Hard version**: Three 6s in a row = player loses their turn

5. **Players**:
   - Temporary objects, no persistence needed
   - Random names/IDs are fine
   - Deleted when the game instance is deleted (this means Player-Game is a **composition** relationship)

### Relationship Analysis for Snake and Ladder

| Relationship             | Type          | Reasoning                                                  |
|--------------------------|---------------|------------------------------------------------------------|
| Game - Board             | Composition   | Board cannot exist without the game                        |
| Board - Snake            | Composition   | Snake is specific to that board, meaningless without it    |
| Board - Ladder           | Composition   | Same as snake                                              |
| Game - Player            | Composition   | Players are temporary, deleted with game                   |
| Game - Dice              | Aggregation/Composition | Dice concept can exist independently, but game-specific dice is composition |
| Game - Rules/Strategy    | Aggregation   | Rules can be reused across games                           |

### The Master While Loop (Critical Validation Step)

The teacher insists that after designing the class diagram, you MUST implement the core game loop to validate the design:

```
while (game is not over):
    identify current player (round robin)
    roll dice (random 1-6)
    move player
    check for snake/ladder at new position
    apply rules (easy/hard version for consecutive 6s)
    check win condition
    advance turn (or give another turn if rolled 6)
```

> **Teacher's exact words**: "You should implement this loop. Only then you will understand whether keeping a strategy that way is going to be better for you, or should you keep it in a different class, should you pass a board in the argument, or should you keep a specific method inside the board class itself."

---

## Design Process Philosophy

### The Teacher's Prescribed Design Process

```
1. Ask clarifying questions (HOW does the user interact?)
      |
2. Start from the CLIENT'S perspective
      |
3. Create V0 design (it CAN be bad - that's okay)
      |
4. Evaluate V0 against SOLID principles
      |
5. Apply design patterns ONLY if violations found
      |
6. Implement the master while loop (real validation)
      |
7. Ask evaluation questions:
   - Is it maintainable?
   - Is it extensible?
   - Is it understandable?
   - Does it follow SOLID?
   - If I add a new entity (not just snake/ladder), how many changes?
   - Am I modifying existing classes or just plugging in new ones?
      |
8. Iterate until satisfied --> Final Design
```

### Anti-Patterns the Teacher Warns Against

| Anti-Pattern                                    | Why It's Wrong                                              |
|-------------------------------------------------|-------------------------------------------------------------|
| Starting with "I'll use Decorator pattern"       | You don't know if you need it yet                           |
| Adding login/authentication for players          | Not part of the requirements                                |
| Taking player names/emails as input              | Requirements only ask for player count                      |
| Skipping the while loop implementation           | You can't validate your design without it                   |
| Applying wrapper patterns without justification  | Decorator, Adapter, Proxy all had specific use cases        |

---

## Interview Questions

### Conceptual Questions

1. **What are the four types of relationships in UML class diagrams?**
   - Inheritance, Association, Aggregation, Composition

2. **What is the difference between Association and Aggregation?**
   - Association: entities are related but one does not contain the other as an attribute
   - Aggregation: one entity HAS the other as an attribute, but child can exist independently

3. **What is the difference between Aggregation and Composition?**
   - Both are "has-a" relationships where one object is inside another
   - Aggregation: child can exist independently (Playlist-Song)
   - Composition: child cannot exist independently, must be cascade-deleted (Board-Snake)

4. **Why does the distinction between Composition and Aggregation matter in practice?**
   - It affects **lifecycle management** and **cascade deletion**
   - It affects **database schema design** (cascade delete on foreign keys)
   - It determines how you manage object creation and destruction

5. **Is the relationship between Proxy and RealSubject aggregation or composition?**
   - Aggregation. If you delete the proxy, the real object still exists.

6. **Given an Order and OrderItem in an e-commerce system, what relationship is it?**
   - Composition. The OrderItem is specific to that Order. It has order-specific discounts, taxes. Deleting the Order means deleting its OrderItems.

7. **Is "Company and Position" composition or aggregation?**
   - It depends on the design. "Software Engineer" is a role that exists across companies (aggregation). But if Position is company-specific with company-specific details, it could be composition.

### Design/Machine Coding Questions

8. **Design a Snake and Ladder game** (Full machine coding round question - asked at multiple companies)

9. **In your Snake and Ladder design, what happens if we want to add a new entity (e.g., a portal or power-up)?**
   - Tests Open/Closed Principle. Ideally, you should be able to plug in a new class without modifying existing ones.

10. **How would you handle two different rule sets (easy vs hard) in the game?**
    - Strategy Pattern: inject the rule as a strategy at game creation time.

---

## Design Decisions

### Why Start From the Client's Perspective?

The teacher is emphatic: "If you are not asking how exactly the user is going to interact with the game, you have already started creating a wrong solution."

**Reason**: The client's interaction defines your API surface. In this case:
- `GameFactory.createGame(n, numPlayers, difficulty)` - creates the game
- `game.makeTurn()` - the ONLY action the user performs

Everything else (dice rolling, turn management, snake/ladder resolution) is internal. If you start designing from the inside out, you might expose unnecessary complexity.

### Why V0 First, Patterns Later?

> "We have learned a lot of design patterns, but we do not need to unnecessarily apply them. You should first come up with some solution. If it is violating SOLID principles and if a design pattern is able to solve it, then only you are going to apply it."

**Reason**: Design patterns solve specific problems. If your V0 doesn't have those problems, adding patterns increases complexity for no benefit. The teacher specifically calls out:

- **Decorator**: Only when you have an existing object and want to add features without modifying it
- **Adapter**: Only when you have two incompatible interfaces that need to work together
- **Proxy**: Only when you need controlled access to an object

### Why Implement the While Loop Before Finalizing Design?

The while loop is the **real validation** of your class diagram. On paper, a design might look clean, but when you actually write:

```java
while (!game.isOver()) {
    Player current = game.getCurrentPlayer();
    int roll = dice.roll();
    // ... now you discover: where does move logic live?
    // In Game? In Board? In Player?
}
```

Only at this point do you discover whether your abstractions make sense.

---

## Important Points

1. **Objects are results of design, not part of the design diagram** - You draw classes, not objects.

2. **Polymorphism is NOT a relationship** - It is a functionality/result of inheritance.

3. **The defining question for Aggregation vs Composition**: "If the larger entity is deleted, can the smaller entity exist independently?"

4. **Cascade delete** - In composition, deleting the parent must trigger deletion of children. This applies to both code AND database schema.

5. **Arrowheads point towards the parent class**, not towards the child.

6. **You don't need to show getters/setters** in class diagrams. They are assumed due to encapsulation.

7. **For interviews**: Relationship types matter more than access specifiers. Interviewers will point at relationships and ask if it's composition or aggregation.

8. **The game is a simulation** - The user only presses a button (`makeTurn()`). Everything else is automated.

9. **Three types of dependencies**: Association, Aggregation, Composition. Plus Inheritance as a fourth relationship type.

10. **Design patterns from previous lectures and their relationship types**:
    - Proxy, Adapter, Decorator all use **Aggregation** (wrapped object survives if wrapper is deleted)

---

## Exam Prep - Quick Fire Facts

### Relationship Cheat Sheet

| Question | Answer |
|----------|--------|
| Playlist - Song | Aggregation |
| Board - Snake | Composition |
| Order - OrderItem | Composition |
| Doctor - Patient | Association |
| Proxy - RealSubject | Aggregation |
| Decorator - Component | Aggregation |
| Adapter - Adaptee | Aggregation |
| Game - Player (temporary) | Composition |
| Laptop - LaptopCover | Association |
| Person - Building | Association |

### Access Specifier Symbols

| Symbol | Meaning |
|--------|---------|
| `+` | Public |
| `-` | Private |
| `#` | Protected |
| `~` | Default/Package |

### Arrow Types in UML

| Relationship | Arrow Type | Diamond? |
|-------------|------------|----------|
| Inheritance | Empty/unfilled arrowhead | No |
| Association | Simple arrow or plain line | No |
| Aggregation | Arrow with empty diamond on parent side | Yes (empty) |
| Composition | Arrow with filled diamond on parent side | Yes (filled) |

### Snake and Ladder Key Numbers

| Requirement | Value |
|-------------|-------|
| Board size | n x n (n^2 squares) |
| Dice range | 1 to 6 |
| Extra turn on | Rolling a 6 |
| Lose turn (hard mode) | Three 6s in a row |
| Snake direction | Start > End (goes down) |
| Ladder direction | Start < End (goes up) |
| Vertical difference | Minimum 1 row |
| Max entities per square | 1 (cannot have both snake and ladder) |

### Design Process Mnemonic: C-V-L-E

1. **C**lient perspective first (how do they interact?)
2. **V**0 design (even if bad)
3. **L**oop implementation (master while loop)
4. **E**valuate (SOLID, extensibility, maintainability)

---

## Teacher's Special Insights

### Career Tips

1. **"In practice, even if you simply specify the method name, that is sufficient."** - Don't over-engineer your class diagrams in interviews. Focus on relationships, not syntax perfection.

2. **"Even if you haven't created the correct arrowhead, that's not going to reduce your points in an interview. But you should be aware of whether it is aggregation or composition."** - Understanding matters more than drawing skills.

3. **"This question (Snake and Ladder) has been asked in multiple companies. And this is not asked in a design interview round. This is asked in a machine coding round where you have to implement the whole thing."** - Be prepared to code it end-to-end, not just design it.

4. **"Once the implementation is done, you are going to get follow-up questions which are going to be based on your design. They will always be either adding new features or modifying existing requirements. And they are going to see how much changes you are going to make."** - Your design is judged by how it handles change, not by how it handles the initial requirements.

### Thumb Rules

5. **Do NOT start from a design pattern.** Start from a solution. Apply patterns only when SOLID violations are found.

6. **Build only what the client asked for.** If the requirement says "number of players," don't build a login system with names and emails.

7. **Wrapper patterns (Proxy, Adapter, Decorator) were for legacy codebases.** Don't force them into new designs without a specific use case.

8. **The while loop is your design's real test.** A class diagram that can't translate into a clean game loop is a bad design.

### Real-World Analogies

9. **Blinkit Order Example**: When you order a milk carton on Blinkit, the milk carton (Item) still exists in the catalog. But the OrderItem (with its specific discount, price, taxes) is a composition of the Order. Delete the order, the OrderItem is gone, but the milk carton stays in the catalog.

10. **Doctor-Patient**: Classic two-way association. Neither contains the other. Both exist independently. One doctor has many patients, one patient has many doctors. No ownership.

11. **Playlist-Song**: You aggregate songs into a playlist. Delete the playlist, the songs still exist for other playlists to use.

12. **Board-Snake**: The snake was created for THIS board. It makes no sense to keep it alive after the board is deleted. That's composition.

---

## Code Examples

### Class Diagram Notation (Pseudocode)

```
+----------------------------------+
|          <<interface>>           |
|          GameEntity              |
+----------------------------------+
| + getStart(): int                |
| + getEnd(): int                  |
+----------------------------------+
          ^             ^
          |             |
   +------+------+  +--+----------+
   |    Snake    |  |   Ladder    |
   +-------------+  +-------------+
   | - start: int|  | - start: int|
   | - end: int  |  | - end: int  |
   +-------------+  +-------------+
```

### Access Specifier Examples

```
+-------------------------------+
|           Player              |
+-------------------------------+
| - id: int                     |    // private
| - name: String                |    // private
| - currentPosition: int        |    // private
+-------------------------------+
| + getCurrentPosition(): int   |    // public
| + move(steps: int): void      |    // public
+-------------------------------+
```

### Skeleton of the Master While Loop

```java
// This is the critical validation step the teacher insists on

public class Game {
    private Board board;
    private List<Player> players;
    private Dice dice;
    private int currentPlayerIndex;
    private RuleStrategy ruleStrategy;  // Easy or Hard rules
    private boolean gameOver;

    // Created via Factory
    // Game(int n, int numPlayers, DifficultyLevel level)

    public void makeTurn() {
        if (gameOver) return;

        Player currentPlayer = players.get(currentPlayerIndex);
        int consecutiveSixes = 0;
        boolean turnContinues = true;

        while (turnContinues) {
            int roll = dice.roll();  // random 1-6

            if (roll == 6) {
                consecutiveSixes++;
                // Delegate to strategy: should player lose turn?
                if (ruleStrategy.shouldLoseTurn(consecutiveSixes)) {
                    // Hard mode: 3 sixes in a row = lose turn
                    break;
                }
            } else {
                turnContinues = false;  // No more rolls unless it was a 6
            }

            // Move player
            int newPosition = currentPlayer.getCurrentPosition() + roll;
            if (newPosition > board.getSize()) {
                // Cannot move beyond the board
                break;
            }

            // Check for snake or ladder at new position
            newPosition = board.getFinalPosition(newPosition);
            currentPlayer.setPosition(newPosition);

            // Check win condition
            if (newPosition == board.getSize()) {
                gameOver = true;
                // currentPlayer wins
                return;
            }

            if (roll != 6) break;  // Only continue if rolled 6
        }

        // Round robin: advance to next player
        currentPlayerIndex = (currentPlayerIndex + 1) % players.size();
    }
}
```

### Strategy Pattern for Game Rules

```java
public interface RuleStrategy {
    boolean shouldLoseTurn(int consecutiveSixes);
}

public class EasyRuleStrategy implements RuleStrategy {
    @Override
    public boolean shouldLoseTurn(int consecutiveSixes) {
        return false;  // Never lose turn, keep rolling
    }
}

public class HardRuleStrategy implements RuleStrategy {
    @Override
    public boolean shouldLoseTurn(int consecutiveSixes) {
        return consecutiveSixes >= 3;  // Three 6s = lose turn
    }
}
```

### Factory for Game Creation

```java
public class GameFactory {
    public static Game createGame(int n, int numPlayers, DifficultyLevel level) {
        Board board = new Board(n);          // n x n board
        board.placeRandomSnakes(n);          // n snakes
        board.placeRandomLadders(n);         // n ladders

        List<Player> players = new ArrayList<>();
        for (int i = 0; i < numPlayers; i++) {
            players.add(new Player(i, "Player-" + i));  // Random names
        }

        RuleStrategy strategy = (level == DifficultyLevel.EASY)
            ? new EasyRuleStrategy()
            : new HardRuleStrategy();

        return new Game(board, players, new Dice(), strategy);
    }
}
```

### Extensibility Test: Adding a New Entity (e.g., Portal)

```java
// If your design is good, adding a Portal should require:
// 1. A new Portal class (implements GameEntity or similar interface)
// 2. Board knows how to place and resolve it
// 3. NO changes to Game, Player, Dice, or RuleStrategy

// BAD design: You have to modify Game.makeTurn() with if-else for each entity
// GOOD design: Board.getFinalPosition() handles all entity types polymorphically
```

---

## Key Takeaways

1. **Relationships are the heart of class diagrams** - entities are straightforward, but getting relationships right (especially composition vs aggregation) is what interviewers test.

2. **The lifecycle question determines everything**: "If the parent is deleted, does the child survive?" YES = Aggregation. NO = Composition.

3. **Always start from the client's perspective** and build only what's required.

4. **V0 first, patterns later** - never start a design by choosing a pattern.

5. **The while loop is your design's litmus test** - if you can't write clean code for the core loop, your design is wrong.

6. **Snake and Ladder is a real interview question** in machine coding rounds. The follow-up questions test your design's extensibility, not just correctness.
