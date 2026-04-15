# LLD Lecture 18 -- Detailed Notes
## Topics: Elevator System Design (Completion) + Rate Limiter System Design (Google Interview Question)

---

## Part A: Elevator System Design -- Final Design Review

---

### 1. The Complete Class Diagram (Teacher's Walkthrough)

A student presents their elevator design, and the teacher critiques and extends it. Below is the full design as discussed.

#### 1.1 Core Classes and Their Attributes/Methods

| Class | Attributes | Methods |
|-------|-----------|---------|
| **Building** | `List<Floor> floors`, `List<Elevator> elevators`, `ElevatorController controller` | -- |
| **Floor** | `int floorNumber`, `Button upButton`, `Button downButton` | -- |
| **ElevatorController** | `List<Elevator> elevators`, `Queue<Request> pendingRequests` | `submitExternalRequest()`, `submitInternalRequest()`, `assignElevator()`, `step()` |
| **Elevator** | `int id`, `int currentFloor`, `Direction direction`, `State state`, `int capacity`, `int currentLoad`, `TreeSet<Integer> upStops`, `TreeSet<Integer> downStops`, `Panel panel` | `move()`, `addInternalRequest()`, `addExternalRequest()`, `openDoor()`, `closeDoor()`, `isIdle(): boolean` |
| **Panel** | `Map<Integer, Button> buttons` | `pressButton(int floor)` |
| **Button** | `boolean pressed` | `press()`, `reset()` |
| **Door** | -- | `open()`, `close()` |
| **Display** | `int currentFloor`, `Direction direction` | `update()` |

#### 1.2 Enums

| Enum | Values |
|------|--------|
| **Direction** | `UP`, `DOWN`, `IDLE` |
| **State** | `MOVING`, `IDLE`, `MAINTENANCE` |

#### 1.3 Request Interface and Implementations

```
interface Request {
    int sourceFloor
    Direction direction
    Timestamp timing

    getSourceFloor()
    getDestinationFloor()
}

class ExternalRequest implements Request {
    // Created when a button on a FLOOR is pressed
    // Only has source floor + direction (up/down)
}

class InternalRequest implements Request {
    // Created when a button INSIDE the elevator is pressed
    // Has source floor + destination floor
}
```

**Key Difference:**
- **External Request** = someone on a floor presses up/down (we know WHERE they are, not WHERE they want to go)
- **Internal Request** = someone inside the elevator presses a floor number (we know WHERE they want to go)

#### 1.4 Relationships Between Classes

| Relationship | Type | Why |
|-------------|------|-----|
| Building -> Floor | **Composition** | Floors cannot exist without the building |
| Building -> Elevator | **Composition** | Elevators are part of the building |
| ElevatorController -> Elevator | **Aggregation** | Elevators can exist without the controller |
| Elevator -> Door | **Composition** | Door cannot exist without the elevator |
| Elevator -> Display | **Composition** | Display cannot exist without the elevator |
| Elevator -> Panel | **Composition** | Panel is part of the elevator |
| Panel -> Button | **Composition** | Buttons are part of the panel |

#### 1.5 General Flow of the System

```
1. User presses Up/Down button on a Floor
2. button.press() is called
3. An ExternalRequest is created
4. ExternalRequest goes to ElevatorController
5. ElevatorController.assignElevator() returns the best elevator
6. ElevatorController.step() simulates elevator movement
7. Check if destination floor is reached
8. If yes -> door opens
9. User enters elevator, presses a floor button (InternalRequest)
10. Elevator processes internal requests based on scheduling strategy
```

---

### 2. The Teacher's Critical Design Extensions

This is where the teacher takes the student's design and makes it interview-ready.

#### 2.1 Strategy Pattern for Elevator Assignment

> **Teacher's Point:** "The assignment should NOT directly be done by the elevator controller. Instead, this should have an object of strategy."

```java
// The ElevatorController should NOT contain the assignment algorithm directly
class ElevatorController {
    ElevatorAssignmentStrategy strategy;  // <-- Strategy object
    List<Elevator> elevators;
    Queue<Request> pendingRequests;

    Elevator assignElevator(Request request) {
        return strategy.assign(request, elevators);
    }
}

interface ElevatorAssignmentStrategy {
    Elevator assign(Request request, List<Elevator> elevators);
}

class NearestElevatorStrategy implements ElevatorAssignmentStrategy { ... }
class LeastLoadedStrategy implements ElevatorAssignmentStrategy { ... }
class ZoneBasedStrategy implements ElevatorAssignmentStrategy { ... }
```

**Why:** Requirements stated that we may want to change the algorithm. Strategy pattern allows swapping without modifying existing code (Open-Closed Principle).

#### 2.2 Strategy Pattern for Task Scheduling Within an Elevator

> **Teacher's Point:** "Inside the elevator, we have these queues. But again, we may want to change the ordering in which the tasks are handled."

```java
class Elevator {
    TaskSchedulingStrategy schedulingStrategy;
    // ...
}

interface TaskSchedulingStrategy {
    Request getNextRequest(TreeSet<Integer> upStops, TreeSet<Integer> downStops);
}

class FCFSScheduling implements TaskSchedulingStrategy { ... }
class SCANScheduling implements TaskSchedulingStrategy { ... }  // Like disk scheduling
class LOOKScheduling implements TaskSchedulingStrategy { ... }
```

**Two separate strategy points:**

| Strategy Location | Purpose | Example Implementations |
|-------------------|---------|------------------------|
| ElevatorController | Which elevator to assign to a request | Nearest, Least Loaded, Zone-based |
| Individual Elevator | In what order to process its assigned tasks | FCFS, SCAN, LOOK |

#### 2.3 Observer Pattern for Sensors

> **Teacher's Insight:** "How does the elevator know what is the current floor? How does it know the current load? There has to be sensors."

The elevator has `currentFloor` and `currentLoad` as attributes. But these values change in real-time. There are physical **sensors** that detect these changes.

```java
// Sensor is the SUBJECT (Observable)
class FloorSensor {  // Subject
    int currentFloor;
    List<Observer> observers;  // Elevator, Display, etc.

    void setFloor(int floor) {
        this.currentFloor = floor;
        notifyObservers();  // Update everyone
    }
}

class WeightSensor {  // Subject
    int currentWeight;
    List<Observer> observers;

    void setWeight(int weight) {
        this.currentWeight = weight;
        notifyObservers();  // If weight > limit, trigger alarm
    }
}

// Elevator is an OBSERVER
class Elevator implements Observer {
    void update(Subject sensor) {
        if (sensor instanceof FloorSensor) {
            this.currentFloor = ((FloorSensor) sensor).getCurrentFloor();
            this.display.update(this.currentFloor);  // Update display too
        }
        if (sensor instanceof WeightSensor) {
            this.currentLoad = ((WeightSensor) sensor).getCurrentWeight();
            if (this.currentLoad > this.capacity) {
                ringAlarm();
            }
        }
    }
}
```

**What the sensor updates:**
- `currentFloor` attribute of the Elevator class
- Internal display (inside elevator showing current floor)
- External display (outside elevator showing current floor + direction)

> **Teacher:** "We don't need to go into the details of how exactly the sensor is getting that data. But it always is going to have the data which will keep getting updated."

---

### 3. Teacher's Interview Strategy for Elevator Design

> **"Whenever you're solving these type of problems, break it down into two parts."**

**Part 1: Come up with the design (PRIORITY)**
- All entities, relationships, class diagrams
- Identify WHERE algorithms will be implemented
- Make it extensible and maintainable using design patterns
- Even if you don't know the exact algorithm, show the strategy interfaces

**Part 2: Implement the algorithm (SECONDARY)**
- Even a brute-force solution is acceptable
- Show that your code is functional

#### 3.1 Scale Analysis (Why Brute Force is OK)

> **Teacher's Thumb Rule:** "Even if you have an N-cube algorithm, that is going to be very efficient for such small data."

| Parameter | Realistic Max | N-square operations |
|-----------|--------------|---------------------|
| Elevators in one building | 20-25 | 625 |
| Floors | ~1000 (Burj Khalifa = 160) | 1,000,000 |
| Requests at a time | Very few | Negligible |

**Key Insight:** In LLD interviews, design ALWAYS takes priority over algorithm optimization. A brute force solution with great design beats an optimized algorithm with poor design.

#### 3.2 Implementation Tips

- Sensor data and floor changes can be **hard-coded** or **simulated**
- Use an array representing floors, iterate to simulate elevator movement
- Apply `Thread.sleep()` / weight if you want non-instant simulation
- **Must have:** A working algorithm for elevator assignment and request processing sequence
- Complexity does not matter -- even O(N^3) is fine for this scale

---

## Part B: Rate Limiter System Design (Google Interview Question)

---

### 4. What is a Rate Limiter?

A rate limiter restricts the number of requests that can be made to a system within a given time period.

#### 4.1 Why Do We Need Rate Limiting?

| Reason | Explanation |
|--------|-------------|
| **Prevent server overload** | Too many concurrent requests can crash the server, making it unavailable for legitimate users |
| **Prevent DDoS attacks** | Malicious users can run scripts to bombard the server |
| **Cost control** | Remote/cloud APIs charge per request; unlimited usage = unlimited cost |
| **Fair usage** | Ensure all users get equitable access to resources |

#### 4.2 Rate Limiting Parameters

Rate limiting can be based on:
- **User ID** -- X requests per user
- **IP Address** -- X requests per IP
- **Region/Location** -- X requests per region
- **API key** -- X requests per key
- **Global** -- X total requests system-wide

> **Teacher:** "In the request, you will always be getting all of this data. The API request is always going to contain the IP address, the user detail, all the location."

---

### 5. The Teacher's Design Journey for Rate Limiter

#### 5.1 Step 1: Where to Implement? (Client vs Server vs Middleware)

```
Client (Frontend) --> [WHERE?] --> Application Server
```

**Option A: Client Side**
- Store rate limit data on client's device
- Example: Hotstar lets you watch 5 minutes free without login. If you clear local storage, you get another 5 minutes.
- **REJECTED** -- Can always be bypassed by the user

**Option B: Server Side (inside app server)**
- Part of your actual application
- Suitable for monolith architectures

**Option C: Middleware (between client and server)**
- Sits between client and server (e.g., API Gateway)
- Most API gateways have built-in rate limiters you can configure
- Recommended for microservice architectures

> **Teacher's Rule:** "Do NOT implement it on the client side because it can always be bypassed."

**Interview Answer:** Always recommend server-side or middleware. Never client-side.

#### 5.2 Step 2: How Does the Client Know the Limit is Reached?

**HTTP Status Code 429 -- Too Many Requests**

```
If rate limit NOT reached:
    --> Process request normally, return actual response

If rate limit IS reached:
    --> Return HTTP 429 (Too Many Requests)
```

| Status Code | Meaning |
|-------------|---------|
| 200 | OK -- request processed |
| 429 | Too Many Requests -- rate limit reached |

#### 5.3 Step 3: Rate Limiting Algorithms

The teacher discusses three approaches:

##### Algorithm 1: Fixed Window

```
Timeline: |----W1----|----W2----|----W3----|

Config: window = 3 seconds, max requests = 3

Window 1: [R1, R2, R3] -> All served
          [R4] -> DENIED (limit reached in this window)

Window 2: [R5] -> Served (new window, counter reset)
```

- Divide timeline into fixed-size, non-overlapping windows
- Each window has its own counter
- Windows are independent of each other
- Counter resets at the start of each new window

**Problem:** Burst at window boundary. If 3 requests come at end of W1 and 3 at start of W2, you get 6 requests in a small time span.

##### Algorithm 2: Sliding Window (To Be Implemented)

```
Config: x = 3 (max requests), t = 4 seconds (window size)

Timeline:  0   1   2   3   4   5   6   7   8
           R1  R2      R3  R4      R5  R6

At t=4: Window [1,4] has R1,R2,R3,R4 = 4 requests > 3 -> R4 DENIED
At t=5: Window [2,5] has R2,R3 = 2 requests -> R5 can be SERVED
At t=6: Window [3,6] has R3,R5 = 2 requests -> R6 can be SERVED
```

- Window slides every second
- At ANY point in time, look back `t` seconds and count requests
- If count >= x, deny. Otherwise, allow.

> **Teacher's detailed walkthrough:**
> - "If the first request comes here, since this is the first request, this can be served."
> - "When the window moves, we check: for this window of length 4, there should be at max 3 requests."
> - "This request should be denied. The first 3 will be served, the next one is denied."

##### Algorithm 3: Token Bucket (mentioned but not detailed)

- Not discussed in depth, but mentioned as another possibility

**Design Requirement:** Whatever algorithm you implement, the design should allow switching between algorithms WITHOUT changing existing code.

---

### 6. The Specific Problem Statement

> **Teacher:** "We are NOT implementing a rate limiter for our own APIs. We are implementing a rate limiter for a remote resource that we are going to use inside our code."

#### 6.1 The Setup

```
You have:
- A remote resource class (NOT written by you)
- It has a method: getResponse()
- This method costs money per call
- Multiple classes in YOUR code need to call this method
- You must limit how many times it gets called

You do NOT have:
- Control over the remote resource
- The remote resource has no built-in rate limiting
```

#### 6.2 The Architecture

```
Client (Frontend)
    |
    v
Your Application Server
    |-- Controller
    |     |-- API 1 --> Service 1 --> [conditionally] --> Remote Resource API
    |     |-- API 2 --> Service 2 --> [conditionally] --> Remote Resource API
    |
    v
Remote Resource (costs money per call)
```

**Critical Observation by Teacher:**

> "Inside these services, we are going to reach out to the remote resource IF NEEDED, which means somewhere inside this code, we have an if block. If that condition is met, THEN we call the remote resource API."

This means:
- NOT every incoming request triggers the remote API call
- Rate limiting at the controller level would wrongly block requests that do not even need the remote resource
- **Rate limiting must be tightly coupled with the actual remote API calls**

#### 6.3 Where NOT to Rate Limit

| Location | Why NOT |
|----------|---------|
| At the controller (entry point) | Would block requests that don't even need the remote resource |
| At the service level (before the if-check) | Same reason -- many requests can be served without calling the remote API |
| On the remote resource itself | We don't control it |

**Correct Location:** Right at the point where the remote API is being called, inside the conditional block.

---

### 7. Design Patterns Used in the Solution

#### 7.1 Proxy Design Pattern (for accessing Remote Resource)

> **Teacher:** "Should we directly be calling the external functions? NO."

**Why not call the remote resource directly?**

| Problem | Explanation |
|---------|-------------|
| **Violates Dependency Inversion** | Your code depends directly on a concrete external class |
| **No abstraction** | If the remote API changes (function name, arguments), you must update every call site |
| **Eager initialization** | Creating the remote resource object even when the if-condition is false = wasteful |
| **No flexibility** | Cannot swap implementations at runtime |
| **Cannot add cross-cutting concerns** | Rate limiting, caching, logging -- nowhere to put them |

**Solution: Proxy Pattern**

```java
// Step 1: Interface (abstraction layer)
interface RemoteService {
    Response getResponse(Request request);
}

// Step 2: Actual remote resource (we don't modify this)
class ActualRemoteResource implements RemoteService {
    Response getResponse(Request request) {
        // Actual API call -- costs money
        return externalApi.call(request);
    }
}

// Step 3: Proxy with rate limiting
class RemoteServiceProxy implements RemoteService {
    private ActualRemoteResource actualResource;  // Lazy initialized
    private RateLimiterStrategy rateLimiter;

    Response getResponse(Request request) {
        if (rateLimiter.isLimitReached()) {
            return new Response(429, "Too Many Requests");
        }

        // Lazy initialization
        if (actualResource == null) {
            actualResource = new ActualRemoteResource();
        }

        rateLimiter.recordRequest();
        return actualResource.getResponse(request);
    }
}
```

**Benefits of Proxy:**
1. Abstraction -- code depends on interface, not concrete class
2. Lazy initialization -- remote resource object created only when needed
3. Rate limiting -- check before making the actual call
4. Caching -- can cache responses if applicable
5. Runtime flexibility -- can inject different implementations

#### 7.2 Strategy Pattern (for Rate Limiting Algorithm)

```java
interface RateLimiterStrategy {
    boolean isLimitReached();
    void recordRequest();
}

class SlidingWindowRateLimiter implements RateLimiterStrategy {
    int maxRequests;       // x
    int windowSizeSeconds; // t
    Queue<Long> requestTimestamps;

    boolean isLimitReached() {
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSizeSeconds * 1000);

        // Remove timestamps outside the window
        while (!requestTimestamps.isEmpty() && requestTimestamps.peek() < windowStart) {
            requestTimestamps.poll();
        }

        return requestTimestamps.size() >= maxRequests;
    }

    void recordRequest() {
        requestTimestamps.add(System.currentTimeMillis());
    }
}

class FixedWindowRateLimiter implements RateLimiterStrategy {
    int maxRequests;
    int windowSizeSeconds;
    long windowStart;
    int currentCount;

    boolean isLimitReached() {
        long now = System.currentTimeMillis();
        if (now - windowStart > windowSizeSeconds * 1000) {
            // New window
            windowStart = now;
            currentCount = 0;
        }
        return currentCount >= maxRequests;
    }

    void recordRequest() {
        currentCount++;
    }
}

class TokenBucketRateLimiter implements RateLimiterStrategy { ... }
```

---

### 8. Complete Design -- Putting It All Together

```java
// ============ INTERFACE ============
interface RemoteService {
    Response getResponse(Request request);
}

// ============ ACTUAL REMOTE RESOURCE (not written by us) ============
class ActualRemoteResource implements RemoteService {
    Response getResponse(Request request) {
        System.out.println("Calling actual remote API...");
        return new Response(200, "Success");
    }
}

// ============ RATE LIMITER STRATEGY ============
interface RateLimiterStrategy {
    boolean isLimitReached();
    void recordRequest();
}

class SlidingWindowRateLimiter implements RateLimiterStrategy {
    private int maxRequests;
    private int windowSizeSeconds;
    private Queue<Long> timestamps = new LinkedList<>();

    SlidingWindowRateLimiter(int maxRequests, int windowSizeSeconds) {
        this.maxRequests = maxRequests;
        this.windowSizeSeconds = windowSizeSeconds;
    }

    boolean isLimitReached() {
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSizeSeconds * 1000L);
        while (!timestamps.isEmpty() && timestamps.peek() < windowStart) {
            timestamps.poll();
        }
        return timestamps.size() >= maxRequests;
    }

    void recordRequest() {
        timestamps.add(System.currentTimeMillis());
    }
}

// ============ PROXY ============
class RemoteServiceProxy implements RemoteService {
    private ActualRemoteResource actualResource;
    private RateLimiterStrategy rateLimiter;

    RemoteServiceProxy(RateLimiterStrategy rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    Response getResponse(Request request) {
        if (rateLimiter.isLimitReached()) {
            System.out.println("HTTP 429 - Too Many Requests");
            return new Response(429, "Too Many Requests");
        }
        if (actualResource == null) {
            actualResource = new ActualRemoteResource();
        }
        rateLimiter.recordRequest();
        return actualResource.getResponse(request);
    }
}

// ============ YOUR SERVICE (uses the proxy, not the actual resource) ============
class MyService {
    private RemoteService remoteService;  // Depends on INTERFACE

    MyService(RemoteService remoteService) {
        this.remoteService = remoteService;
    }

    void handleRequest(Request request) {
        if (someCondition(request)) {
            // Only calls remote when needed
            Response response = remoteService.getResponse(request);
            // process response...
        } else {
            // Handle locally, no remote call needed
        }
    }
}

// ============ MAIN / WIRING ============
class Main {
    public static void main(String[] args) {
        RateLimiterStrategy limiter = new SlidingWindowRateLimiter(3, 4);
        RemoteService proxy = new RemoteServiceProxy(limiter);
        MyService service = new MyService(proxy);
        // service.handleRequest(...)
    }
}
```

---

### 9. Sliding Window Algorithm -- Detailed Walkthrough

**Configuration:** x = 3 requests max, t = 4 seconds window

```
Time(s): 0    1    2    3    4    5    6    7    8
Events:  R1   R2        R3   R4        R5   R6

Step-by-step:

t=0: Window [0,4) -> Requests: [R1] -> Count=1 <= 3 -> R1 SERVED
t=1: Window [0,4) -> Requests: [R1,R2] -> Count=2 <= 3 -> R2 SERVED
t=3: Window [0,4) -> Requests: [R1,R2,R3] -> Count=3 <= 3 -> R3 SERVED
t=4: Window [1,4] -> Requests: [R1,R2,R3,R4] -> Count=4 > 3 -> R4 DENIED
     (Actually t=4 is boundary. Teacher says: "t equals to 4 should also be denied")
t=5: Window [2,5] -> Requests: [R2,R3] -> Count=2 <= 3 -> R5 SERVED
t=6: Window [3,6] -> Requests: [R3,R5] -> Count=2 <= 3 -> R6 SERVED
```

**Data Structure:** Use a `Queue<Long>` (LinkedList) to store timestamps of served requests.

**Algorithm:**
1. Get current time
2. Calculate window start = current time - t
3. Remove all timestamps from queue that are before window start
4. If queue size >= x, DENY (return 429)
5. Otherwise, add current timestamp to queue, ALLOW (make actual call)

---

### 10. Rate Limiter for Your Own API (Homework)

> **Teacher:** "In the homework, try to figure this out. You have your own API. There is no remote resource. Probably a proxy will not be needed, but how do you ensure that before you make the actual function calls of your API, you check for the rate limit?"

**Hint:** When it is your own API:
- No proxy needed (you own the code)
- Rate limit check happens BEFORE the actual service method executes
- Can be implemented as middleware/interceptor/decorator
- The strategy pattern for the algorithm remains the same

---

## 11. Interview Questions

### Elevator System

| Question | Key Points |
|----------|------------|
| What relationships exist between Building, Floor, Elevator? | Building-Floor: Composition; Building-Elevator: Composition; Controller-Elevator: Aggregation |
| Why Aggregation for Controller-Elevator? | Elevator can exist without the controller |
| Why use Strategy for elevator assignment? | May want to change algorithm; Open-Closed Principle |
| How does the elevator know the current floor? | Sensors + Observer Pattern |
| Why two types of requests (Internal/External)? | External: only source + direction; Internal: source + destination; handled differently |
| Why use TreeSet for up/down stops? | Sorted order, efficient lookup and removal |

### Rate Limiter

| Question | Key Points |
|----------|------------|
| Client-side vs Server-side rate limiting? | Always server-side; client-side can be bypassed |
| What HTTP status code for rate limit? | 429 -- Too Many Requests |
| Fixed window vs Sliding window? | Fixed: burst problem at boundaries; Sliding: smoother but more computation |
| Why Proxy pattern for remote resource? | Abstraction, lazy init, rate limiting injection, dependency inversion |
| Why Strategy pattern for rate limiting? | Swap algorithms (sliding/fixed/token bucket) without changing code |
| Where should rate limiting be applied? | As close to the actual API call as possible, not at the controller level |

---

## 12. Design Decisions Summary

| Decision | Chosen Approach | Why |
|----------|----------------|-----|
| Elevator assignment | Strategy Pattern | Algorithm may change; extensibility |
| Task scheduling in elevator | Strategy Pattern | Order of processing may change |
| Sensor data propagation | Observer Pattern | Multiple components need real-time updates |
| Remote resource access | Proxy Pattern | Abstraction, lazy init, rate limiting |
| Rate limiting algorithm | Strategy Pattern | Multiple algorithms possible |
| Rate limit location | At the remote call site, not controller | Avoids blocking requests that don't need remote resource |
| Client vs Server rate limit | Server-side | Client-side is bypassable |

---

## 13. Important Points / Key Concepts

1. **In LLD interviews, design > algorithm.** Even a brute-force O(N^3) solution is acceptable if the design is clean and extensible.

2. **Scale matters for deciding optimization.** Elevators: max 20-25. Floors: max ~1000. At this scale, brute force is perfectly efficient.

3. **Sensor data in real systems uses Observer Pattern.** The sensor is the Subject; elevator, display, and alarm system are Observers.

4. **Proxy Pattern is the go-to for wrapping external/remote resources.** It provides abstraction, lazy initialization, and a place to inject cross-cutting concerns like rate limiting.

5. **Rate limiting must be placed where the actual cost occurs.** Not at the entry point of your system.

6. **HTTP 429** = Too Many Requests. This is the standard response for rate-limited requests.

7. **Sliding window** is generally preferred over fixed window because it avoids the burst-at-boundary problem.

8. **Composition vs Aggregation reminder:**
   - Composition: child cannot exist without parent (Door without Elevator)
   - Aggregation: child can exist independently (Elevator without Controller)

---

## 14. Exam Prep -- Quick-Fire Facts

| Fact | Answer |
|------|--------|
| HTTP status for rate limit | 429 |
| Burj Khalifa floors | 160 (teacher mentioned ~780m height, 160 floors) |
| Design patterns in Elevator | Strategy (x2), Observer |
| Design patterns in Rate Limiter | Proxy, Strategy |
| Elevator-Door relationship | Composition |
| Controller-Elevator relationship | Aggregation |
| Should rate limiter be client-side? | NO -- can be bypassed |
| TreeSet used for what in Elevator? | Storing up-stops and down-stops (sorted) |
| Sensor updates which components? | currentFloor attribute, internal display, external display |
| Where is rate limit logic in proxy solution? | Inside the Proxy class, before delegating to actual resource |
| What SOLID principle does Proxy follow? | Dependency Inversion (depend on interface, not concrete class) |
| What SOLID principle does Strategy follow? | Open-Closed (open for extension, closed for modification) |

---

## 15. Teacher's Special Insights

### Career/Interview Tips

1. **"Part one of solving these questions is come up with a design which is extensible, maintainable. Even if you are not sure with the logic, that is completely okay."** -- Design first, algorithm second.

2. **"Even if you implement a brute force solution... this is a design round. The design is always going to take priority when it comes to judgment."** -- Interviewers care about your design thinking, not algorithm optimization in LLD rounds.

3. **"You can say: whenever I have a more efficient algorithm, I have these strategies, I can always implement it."** -- Show awareness of extensibility.

4. **"This is a question asked in Google, repeatedly, multiple times."** -- Rate limiter is a high-value interview question.

5. **"Generally the interview length is not one and a half hour, but sometimes the companies will allow you to extend if you are going in the right direction."** -- Being on the right track matters more than finishing.

6. **On the Hotstar example:** Client-side rate limiting using local storage is trivially bypassed. Never rely on the client for security or limiting.

7. **"Do not implement it on the client side because it can always be bypassed."** -- Security/limiting logic must live server-side.

### Real-World Analogies

- **Hotstar free viewing:** Client-side rate limiting example. Delete local storage = reset the limit. This is why client-side is unreliable.
- **Cloud API billing:** Real motivation for rate limiting -- every API call costs money; users exploiting your system drain your budget.
- **DDoS attack:** Malicious script bombarding server with requests, making it unavailable for legitimate users.

### Thumb Rules

- **Max elevators in a building:** 20-25 (realistic), 100 (unrealistic upper bound)
- **Max floors:** ~1000 (Burj Khalifa has 160)
- **For these scales:** O(N^2) or even O(N^3) algorithms are efficient enough
- **Viva duration:** 20-30 minutes. Quick answers = 20 min. Code implementation = up to 30 min.

---

## 16. Viva/Exam Announcements (from lecture)

- **Viva starts Monday**; slots released by Friday
- **Syllabus:** OOP principles, Design Patterns, Assignment Questions
- **Question types:**
  - Describe and implement a given design pattern
  - Explain design choices from your assignment code
  - Redesign a section from an assignment (e.g., parking lot)
  - Use-case based: identify which design pattern solves it, then implement
- **NO DSA questions** -- functions will have print statements, no algorithm problems
- **Duration:** 20-30 minutes
- **Submissions open until Monday:** Book My Show, Elevator, Parking Lot (reopened)
- For Book My Show and Elevator: implement only the APIs discussed in class

---

## 17. Summary of Design Patterns Used in This Lecture

| Pattern | Where Used | Purpose |
|---------|-----------|---------|
| **Strategy** | ElevatorController -> ElevatorAssignmentStrategy | Swap elevator assignment algorithms |
| **Strategy** | Elevator -> TaskSchedulingStrategy | Swap request processing order |
| **Observer** | FloorSensor/WeightSensor -> Elevator, Display | Real-time updates when sensor data changes |
| **Proxy** | RemoteServiceProxy -> ActualRemoteResource | Abstraction, lazy init, rate limiting for external API |
| **Strategy** | RemoteServiceProxy -> RateLimiterStrategy | Swap rate limiting algorithms (sliding/fixed/token bucket) |
