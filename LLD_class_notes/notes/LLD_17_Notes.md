# LLD Lecture 17 -- Detailed Notes

## Topics Covered
1. **BookMyShow Design Review** -- Critique of a student's design, identifying SOLID violations, and refactoring guidance
2. **Elevator System Design** -- New problem introduction, requirement gathering, and design framework
3. **Interview Framework** -- The teacher's universal approach to solving any LLD problem

---

## Part 1: BookMyShow Design Review (Student Presentation + Teacher Critique)

### The Student's Presented Design (V0)

A student (Hushita) presented an exhaustive design for a BookMyShow-like ticket booking system. Here is a summary of the classes and relationships presented:

| Component | Description |
|-----------|-------------|
| `BookingSystem` (main class) | Central orchestrator |
| `User` class | Two types: Customer, Admin |
| `ICreateItem` interface | Methods for creating Movie, Theatre, Show |
| `CreateMovie`, `CreateTheatre`, `CreateShow` | Implementations of `ICreateItem` |
| `Payment` (abstract class) | `makePayment()` method |
| `UPIPayment`, `CardPayment` | Concrete payment implementations |
| `IBooking` interface | `bookSeat()`, `cancelBooking()` methods |
| `SeatType` enum | Base prices for different seat categories |
| `IDisplayStrategy` interface | `getItem(city)`, `getScreen(theatre)`, `getMovie(screen)` |
| `ShowTheater`, `ShowMovie` classes | Implementations of display strategy |
| `ITotalAmount` interface | Strategy for computing total amount differently per theatre |
| `IRefundAmount` interface / abstract class | `UPIRefund`, `CardRefund` implementations |
| Entity classes | `Movie`, `Theatre`, `Show`, `Ticket`, `Screen` |

### Teacher's Critique -- Problems Found

#### Problem 1: IDisplayStrategy Violates Interface Segregation Principle (ISP)

The `IDisplayStrategy` interface had multiple methods:
- `getItem(city)` -- get list of theatres by city
- `getScreen(theatre)` -- get list of screens by theatre
- `getMovie(screen)` -- get list of movies by screen

**Why it violates ISP:**
- The implementing classes (`ShowTheater`, `ShowMovie`) have very specific names, but each class is forced to implement ALL methods in the interface.
- There are NOT different behaviors/algorithms for these methods -- each method has a single, fixed implementation.
- When there is only ONE behavior for a method, you do NOT need a strategy pattern.

**Teacher's Fix:**
> Remove the interface entirely. Keep all these methods inside one single class. Strategy pattern is only needed when there are MULTIPLE interchangeable behaviors for the same operation.

**Key Rule:**
> Strategy pattern = multiple algorithms/behaviors for the same operation. If there is only one behavior, use a simple concrete class.

---

#### Problem 2: ICreateItem Interface with Single Object Storage

The `BookingSystem` stored ONE object of `ICreateItem`. That means at runtime it could only hold ONE of: `CreateMovie` OR `CreateTheatre` OR `CreateShow`.

**Why this is wrong:**
- An admin needs to create movies AND theatres AND shows -- all during the same session.
- Storing a single `ICreateItem` restricts access to only one implementation at a time.

**Teacher's Fix:**
> Remove this interface. These are distinct operations (not interchangeable strategies). They should be concrete methods in a service or directly available.

---

#### Problem 3: Where Should Admin/Customer Methods Live?

**Student's idea:** Create `Customer` and `Admin` subclasses of `User`, put respective methods (bookTicket in Customer, addMovie/addTheatre in Admin) directly inside them.

**Teacher's strong opinion -- DO NOT do this:**

> All actions are performed by some type of user. Does that mean we put ALL methods inside a User class? No!

**Teacher's Rule:**
> Inside User subclasses, ONLY keep user-specific attributes and methods related to those attributes. All business logic should go in separate Service and Repository classes.

**Correct Architecture:**

```
User class
  |-- Common attributes (name, email, etc.)
  |-- Common methods (related to user attributes only)

Services (separate classes)
  |-- MovieService: addMovie(), getMovie(), getMoviesByTheatre()
  |-- TheatreService: addTheatre(), getTheatre()
  |-- ShowService: addShow(), getShow()
  |-- BookingService: bookTicket(), cancelBooking()

Repositories (separate classes)
  |-- MovieRepository: stores movies, maintains indexes (HashMaps)
  |-- TheatreRepository: stores theatres, maintains indexes
  |-- ShowRepository: stores shows, maintains indexes
```

**Why Repositories with Indexes?**
- "Find all shows of a given movie" -- you cannot linearly iterate; you need a HashMap.
- "Find all movies in a given theatre" -- again, needs a HashMap index.
- All data structures for storage and indexing belong in Repository classes.

**Call Chain:** Service --> Repository

---

### Authentication Discussion

**With a Framework (e.g., Rails, Spring):**
- Authentication layer exists; you always know the current user.
- Use `before_action` or equivalent to validate user roles BEFORE calling admin-only methods.
- User info comes as an encrypted token, decrypted on the backend.

**Without a Framework (Plain Java, as in this course):**
- Add a `role` attribute to the User class (or a `Role` enum).
- Before calling any restricted function, check if the user has the appropriate role.
- Pass the User object to service methods that need authorization.

**Important Design Discussion -- Can a User Have Multiple Roles?**

| Scenario | Design |
|----------|--------|
| User has exactly ONE role | Simple `Role` enum attribute on User |
| User can have MULTIPLE roles | Separate Role class, store relationships separately (e.g., `List<Role>`) |

> Always ask the interviewer: "Can a user have multiple roles?" -- this changes the design.

**Teacher's Honesty:**
> Passing the user object to every method is an anti-pattern. Ideally we should NOT do that. But without an authentication layer, we have to.

---

### Concurrency Issues to Handle in BookMyShow

1. **Booking the same seat** -- If one person is on the payment page for a seat, another person should see that seat as unavailable. The `bookTicket()` function MUST resolve this concurrency issue.

2. **Adding overlapping shows** -- When adding two shows, you must check:
   - No show exists at the exact same timing
   - No show overlaps with an existing show's time range
   - This is also a DSA problem (interval overlap detection)

---

### Pricing Strategy -- Correctly Needed

Unlike the display strategy (which was incorrectly applied), pricing DOES need a strategy pattern because:

- Base prices exist per seat category (in the SeatType enum)
- Additional factors can INCREASE the price (never decrease):
  - Movie popularity
  - Time of day
  - Demand/occupancy
- These rules are configurable -- an admin picks a set of rules
- New pricing rules should be addable by creating a new class (Open-Closed Principle)

**This is a correct use of Strategy Pattern** -- multiple interchangeable pricing algorithms.

---

### Teacher's Design Journey Summary for BookMyShow (V0 --> V1 --> Final)

```
V0 (Student's initial design):
  - IDisplayStrategy with multiple methods --> WRONG (ISP violation)
  - ICreateItem stored as single object --> WRONG (restricts access)
  - Methods inside User subclasses --> WRONG (bloated user classes)
  - Payment strategy --> CORRECT
  - Pricing strategy --> MISSING but needed

V1 (After critique):
  - Remove IDisplayStrategy; use single concrete class
  - Remove ICreateItem interface; use direct concrete methods
  - Move business methods OUT of User into Services + Repositories
  - Add role-based authentication on User
  - Add pricing strategy (multiple algorithms)
  - Handle concurrency in bookTicket()
  - Handle show overlap validation

Final Architecture:
  User (with role) --> Services --> Repositories (with HashMap indexes)
  Strategy Pattern ONLY where multiple behaviors exist (Payment, Pricing)
  Concrete classes where single behavior exists (Display)
```

---

## Part 2: Elevator System Design (New Problem)

### Problem Statement

Design an object-oriented elevator system for a building with multiple elevator cars.

### Requirement Gathering (Teacher-Led Discussion)

#### User Interfaces (Touch Points)

**External Panel (one per floor, shared across all elevator cars):**

| Button | Function |
|--------|----------|
| Up | User wants to go up |
| Down | User wants to go down |

- Top floor: only Down button
- Bottom floor: only Up button
- All other floors: both Up and Down

**Internal Panel (one per elevator car):**

| Button | Function |
|--------|----------|
| Floor buttons (one per floor) | Select destination floor |
| Open door | Open elevator door |
| Close door | Close elevator door |
| Alarm | Emergency -- ALL elevators stop, alarm rings |

#### Key Design Requirements

| Requirement | Details |
|-------------|---------|
| Multiple elevator cars | System manages ALL cars in a building, not just one |
| External buttons shared | Pressing Up on a floor can summon ANY available car |
| One car per request | Only ONE elevator should respond to a button press (energy optimization) |
| Pressing same button twice | Should NOT summon two elevators |
| Both Up and Down pressed | May bring two different elevators (one for each direction) |
| After elevator leaves floor | Another elevator CAN then come to that floor |
| All cars stop on all floors | No restricted floors |
| Floors can be added dynamically | Building can add floors; panels update accordingly |
| Weight limit | Default 700 kg, can vary per car, can change with hardware updates |
| Exceeding weight limit | Car does not move, stays open, plays a warning sound |
| Maintenance state | Some cars may be under maintenance (non-operational, ignore all calls) |

#### Four States of an Elevator Car

| State | Description |
|-------|-------------|
| **Moving Up** | In transit going up |
| **Moving Down** | In transit going down |
| **Idle** | Stationary at a specific floor, ready to be dispatched |
| **Under Maintenance** | Non-operational, does not respond to any calls |

#### Elevator Dispatching Algorithms (Strategy Pattern Required)

The teacher explicitly stated: **design so new algorithms can be injected effortlessly** (Open-Closed Principle).

| Algorithm | Description |
|-----------|-------------|
| **First Come First Serve (FCFS)** | Whichever button pressed first gets served first |
| **Shortest Seek First (SSF)** | Elevator goes to the nearest request first, even if it was pressed later |
| (Future algorithms) | Should be addable by creating a new class only |

#### Irrelevant Questions (Rejected by Teacher)

- "What if there is no power?" -- Software won't work either; irrelevant.
- Sensor details for floor detection -- Can be hardcoded or assumed; out of scope.
- Physical/hardware questions -- No impact on OO design.

### What to Design

You are designing the **software behind the interfaces:**
- Define what function is called when each button is pressed
- Implement dispatching logic (which car to send)
- Handle state transitions for elevator cars
- Handle alarm/emergency behavior
- Handle weight limit checks

### Teacher's Hints for Implementation

- The design is SMALL compared to BookMyShow (fewer classes, fewer APIs, fewer validations).
- Floor detection can be hardcoded or assumed via an API call.
- Focus on functional requirements only.
- Target: design + full implementation in 30 minutes.

---

## Part 3: The Teacher's Universal LLD Interview Framework

### The Framework (Applied Identically to Every Problem)

```
Step 1: REQUIREMENTS
  --> Ask: What are the APIs to implement?
  --> Ask: How does the client interact with the system?
  --> Ask: Are there different BEHAVIORS for the same entity?
  --> DO NOT discuss irrelevant things

Step 2: BASIC DESIGN (V0 -- the "brute force")
  --> Start with a BAD design. Just get something on paper.
  --> Use concrete classes. No premature abstraction.
  --> No design patterns yet.

Step 3: EVALUATE
  --> Does it satisfy all requirements?
  --> Does it follow SOLID principles?
  --> Are there future requirements that would violate Open-Closed?

Step 4: REFACTOR (V1, V2...)
  --> Apply design patterns ONLY to solve identified problems
  --> Restructure classes if needed
  --> Stop when SOLID is satisfied and requirements are met
```

### What Makes a Question Relevant in Requirement Gathering?

| Question Type | Relevance | Why |
|---------------|-----------|-----|
| **Behavior questions** (different behaviors of same entity?) | HIGH -- impacts design | May need subclasses, inheritance, strategy |
| **Generalization questions** (can entities be grouped?) | HIGH -- impacts design | May need abstract classes, interfaces |
| **Implementation questions** (how exactly does X work internally?) | LOW in LLD rounds | Expected outcome is class diagram, not code |
| **Hardware/physical questions** | IRRELEVANT | No impact on OO design |

### Three Categories of LLD Problems

| Category | Examples | Key Approach |
|----------|----------|-------------|
| **Physical entities** (hard to imagine as code) | Pen, Bird, Parking Lot | Start with APIs, not physical properties. You are NOT designing a physical pen. |
| **Software systems you use daily** | BookMyShow, Cab Booking, Food Delivery, Library Management | Constrain requirements ruthlessly. They are NOT asking you to design the ENTIRE system. |
| **Intersection / Looks like DSA** | Parking Lot, Elevator, (upcoming lectures) | Focus on APIs + behaviors. May contain DSA sub-problems. |

---

## Interview Questions and Answers

### Q1: When should you use the Strategy Pattern vs a concrete class?
**A:** Use Strategy when there are MULTIPLE interchangeable behaviors/algorithms for the same operation. If only ONE behavior exists, use a simple concrete class. Unnecessary abstraction violates YAGNI and can violate ISP.

### Q2: Where should business logic live -- in entity/model classes or separate?
**A:** In separate Service and Repository classes. Entity classes (like User) should only contain attributes and methods related to those attributes. Business operations (addMovie, bookTicket) go in Services. Data storage and indexing go in Repositories.

### Q3: How do you handle authentication in a plain Java LLD design (no framework)?
**A:** Add a `role` attribute (or `List<Role>`) to the User class. Pass the User object to service methods. Check the role before executing restricted operations. This is an anti-pattern but necessary without a framework.

### Q4: How do you handle concurrent seat booking?
**A:** Use synchronization/locking on the seat. When a user proceeds to payment, the seat must be locked so other users see it as unavailable. The `bookTicket()` function must be thread-safe.

### Q5: How do you handle overlapping show timings?
**A:** This is a DSA interval overlap problem. Before adding a new show to a screen, check that its time range does not overlap with any existing show on that screen.

### Q6: Design an elevator system -- what are the key entities?
**A:** External button panel (per floor), Internal button panel (per car), ElevatorCar (with state: MovingUp, MovingDown, Idle, Maintenance), DispatchStrategy (FCFS, Shortest Seek First, etc.), Floor, Building.

### Q7: What are the four states of an elevator car?
**A:** Moving Up, Moving Down, Idle (at a specific floor), Under Maintenance (non-operational).

### Q8: Why is the elevator question considered hard in interviews?
**A:** Candidates go blank because they cannot think of correct requirements and cannot ask the right questions. The key is to focus on the interfaces (buttons), the APIs (what happens on button press), and the dispatching logic (which car to send).

---

## Important Thumb Rules and Teacher's Special Insights

### On Starting a Design
> "Start with the basics. First, do not have the abstraction layer. Keep all of them as concrete classes. Have their factories. Then if there is a problem, then we will come up with a design pattern to solve it."

### On Memorization
> "You should NOT try to memorize anything. Any questions specifically, the frequently asked questions. Because in every interview, you will get a DIFFERENT VERSION of the same question. The requirements may not be the same every time."

### On Factories
> "You can have multiple factories. What is the need of this interface right now? First, do not have this abstraction layer."

### On the LLD Framework Being Universal
> "The framework, if you see in every single question, we are following the exact same steps."

### On Constraining Scope in Interviews
> "If you are asked to design a ticket booking system, they are NOT asking you to design the ENTIRE BookMyShow. You have to constrain the requirements. You should NOT just take all the requirements of BookMyShow and start discussing them."

### On Relevant vs Irrelevant Questions
> "How do you know whether a question is relevant or not? If it is going to have an impact in the design, then it is relevant. If it is going to have an impact in the implementation, then it is slightly relevant. In a LLD round where the expected outcome is a class diagram, implementation-based questions can also be termed as irrelevant."

### On the DSA Analogy
> "Very similar to how we solve DSA problems. We start with the brute force. So have a bad design. Then evaluate that design."

### Career Tip -- The 30-Minute Implementation Drill
> "Set a timer of at max 30 minutes and in those 30 minutes try to implement it end to end. You may hardcode some things."

### On When to Apply Design Patterns
> "If there is a problem, if the SOLID principles are being violated, if there are future requirements which are going to violate principles like Open-Close Principle, if you have to make significant changes in the code -- THEN you need to redesign. And THEN you think of a design pattern which can help you solve it."

---

## Exam Prep -- Quick-Fire Facts

| Fact | Detail |
|------|--------|
| ISP violation indicator | Interface with many methods, implementations don't need all of them |
| When to use Strategy | Multiple interchangeable algorithms for same operation |
| When NOT to use Strategy | Only one behavior exists for the operation |
| Where business logic goes | Services (not in entity/model classes) |
| Where data structures/indexes go | Repositories |
| Call chain | Service --> Repository |
| Anti-pattern mentioned | Passing user object to every method (necessary without auth framework) |
| Elevator car states | Moving Up, Moving Down, Idle, Under Maintenance |
| Elevator external buttons | Up + Down (per floor, shared across all cars) |
| Elevator internal buttons | Floor buttons + Open + Close + Alarm |
| Alarm behavior | ALL elevators stop, alarm rings |
| Weight exceeded behavior | Car stays open, does not move, plays warning sound |
| Default weight limit | 700 kg (variable per car, can change) |
| Elevator dispatch algorithms | FCFS, Shortest Seek First (must be pluggable via Strategy) |
| Pressing same button twice | Should NOT bring two elevators |
| Both Up and Down pressed | May bring two different elevators |

---

## Assignments Due (Next Monday)

1. **BookMyShow** -- Complete class diagram + basic implementation with:
   - `bookTicket()` function with concurrency handling
   - Show overlap validation
2. **Elevator System** -- Complete class diagram + implementation (target: 30 minutes)

---

## Code Example: Skeleton of Elevator System (Based on Discussion)

```java
// --- Enums ---
enum ElevatorState {
    MOVING_UP, MOVING_DOWN, IDLE, UNDER_MAINTENANCE
}

enum Direction {
    UP, DOWN
}

// --- Elevator Car ---
class ElevatorCar {
    private int id;
    private int currentFloor;
    private ElevatorState state;
    private double weightLimit; // default 700 kg
    private double currentWeight;
    private InternalPanel internalPanel;

    public boolean isAvailable() {
        return state != ElevatorState.UNDER_MAINTENANCE;
    }

    public void moveToFloor(int targetFloor) {
        if (currentWeight > weightLimit) {
            // Stay open, play warning, do NOT move
            return;
        }
        if (targetFloor > currentFloor) {
            state = ElevatorState.MOVING_UP;
        } else if (targetFloor < currentFloor) {
            state = ElevatorState.MOVING_DOWN;
        }
        // ... move logic, update currentFloor ...
        state = ElevatorState.IDLE;
    }
}

// --- Button Panels ---
class ExternalPanel {
    private int floor;
    private Button upButton;   // null on top floor
    private Button downButton; // null on bottom floor

    public void pressUp() {
        // Calls dispatcher to assign an elevator car
    }

    public void pressDown() {
        // Calls dispatcher to assign an elevator car
    }
}

class InternalPanel {
    private List<Button> floorButtons;
    private Button openButton;
    private Button closeButton;
    private Button alarmButton;

    public void pressFloor(int floor) {
        // Add floor to this car's destination queue
    }

    public void pressAlarm() {
        // Stop ALL elevator cars, ring alarm
    }
}

// --- Dispatch Strategy (Strategy Pattern) ---
interface DispatchStrategy {
    ElevatorCar selectElevator(int requestedFloor, Direction direction,
                                List<ElevatorCar> cars);
}

class FCFSDispatchStrategy implements DispatchStrategy {
    @Override
    public ElevatorCar selectElevator(int requestedFloor, Direction direction,
                                       List<ElevatorCar> cars) {
        // First available idle car gets assigned
        // ...
    }
}

class ShortestSeekFirstStrategy implements DispatchStrategy {
    @Override
    public ElevatorCar selectElevator(int requestedFloor, Direction direction,
                                       List<ElevatorCar> cars) {
        // Find the nearest available car to the requested floor
        // ...
    }
}

// --- Elevator System (Orchestrator) ---
class ElevatorSystem {
    private List<ElevatorCar> cars;
    private List<ExternalPanel> externalPanels; // one per floor
    private DispatchStrategy dispatchStrategy;

    public void requestElevator(int floor, Direction direction) {
        ElevatorCar car = dispatchStrategy.selectElevator(
            floor, direction, cars);
        if (car != null) {
            car.moveToFloor(floor);
        }
    }

    public void setDispatchStrategy(DispatchStrategy strategy) {
        this.dispatchStrategy = strategy;
    }
}
```

---

## Code Example: BookMyShow Architecture Skeleton (Based on Teacher's Corrections)

```java
// --- User with Role ---
enum Role { CUSTOMER, ADMIN }

class User {
    private String id;
    private String name;
    private String email;
    private Role role;  // Ask interviewer: can user have multiple roles?
}

// --- Repository (data storage + indexing) ---
class MovieRepository {
    private List<Movie> movies;
    private Map<Theatre, List<Movie>> moviesByTheatre;  // index
    private Map<String, Movie> moviesById;              // index

    public void addMovie(Movie movie) { /* ... */ }
    public List<Movie> getMoviesByTheatre(Theatre t) {
        return moviesByTheatre.getOrDefault(t, new ArrayList<>());
    }
}

class ShowRepository {
    private List<Show> shows;
    private Map<Movie, List<Show>> showsByMovie;  // index

    public void addShow(Show show) {
        // Validate: no overlapping shows on same screen
        // This is a DSA interval problem
    }
}

// --- Service (business logic) ---
class MovieService {
    private MovieRepository movieRepository;

    public void addMovie(User user, Movie movie) {
        if (user.getRole() != Role.ADMIN) {
            throw new UnauthorizedException("Only admins can add movies");
        }
        movieRepository.addMovie(movie);
    }
}

class BookingService {
    public synchronized Ticket bookTicket(User user, Show show, Seat seat) {
        // Must handle concurrency -- synchronized or use locks
        if (seat.isBooked()) {
            throw new SeatUnavailableException("Seat already booked");
        }
        seat.setBooked(true);
        // Process payment, create ticket...
    }
}

// --- Pricing Strategy (correctly applied) ---
interface PricingStrategy {
    double calculatePrice(Seat seat, Show show);
}

class BasePricingStrategy implements PricingStrategy {
    public double calculatePrice(Seat seat, Show show) {
        return seat.getType().getBasePrice();
    }
}

class DemandBasedPricingStrategy implements PricingStrategy {
    public double calculatePrice(Seat seat, Show show) {
        double base = seat.getType().getBasePrice();
        double multiplier = getDemandMultiplier(show);
        return base * multiplier;
    }
}
```

---

*End of LLD Lecture 17 Notes*
