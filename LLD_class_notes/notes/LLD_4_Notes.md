# LLD Lecture 4 -- Detailed Notes

## Topics Covered
- Recap of SOLID Principles (SRP, OCP, LSP)
- Hands-on Design Exercise: HR Management System
- Design Journey from V0 to V1 (iterative refinement)
- Introduction to Serializers, Storage Services, and Tax Calculators
- Interview Guidance: LLD Interview vs Machine Coding Round

---

## 1. Quick Recap of First Three SOLID Principles

### Single Responsibility Principle (SRP)
- Every function, class, package, or microservice should have **one primary reason to change**.
- The "granularity" of that reason varies by scope:
  - **Functions** -- very specific reason
  - **Classes** -- slightly less specific
  - **Packages** -- abstract
  - **Microservices** -- even more abstract
- **Danger of over-applying SRP**: If you try to enforce SRP where it is already satisfied or not needed, you may end up with **class explosion** (too many tiny classes).

### Open-Closed Principle (OCP)
- Classes should be **open for extension** (adding new features) but **closed for modification** (tested, working code should not be changed).
- If code is working and tested, and no change is expected in that part, it should remain untouched.

### Liskov Substitution Principle (LSP)
- Whatever exists in the parent class should work correctly for the child class.
- A parent class object should be **completely replaceable** by a child class object.
- If a function expects a parent reference, any child reference should be passable **without exceptions or issues**.

---

## 2. The Teacher's Design Journey -- HR Management System

This is the core of the lecture. The teacher walks through building an HR Management System from scratch, evaluating each iteration against SOLID principles and improving it.

---

### V0 -- The Naive Design

#### Requirements
1. Support two types of employees: **Full-Time** and **Intern**.
2. Implement a `save()` method that writes an employee's data to a file in a specific format.
3. The same method should work for all employee types.

#### V0 Class Diagram

```
+---------------------+
|  Employee (abstract) |
|---------------------|
| - name              |
| - email             |
| - type              |
|---------------------|
| + save()            |
+---------------------+
      ^          ^
      |          |
+------------+ +---------+
| FullTimeEmp| | Intern  |
+------------+ +---------+

+---------------------+
| EmployeeRepo        |
|---------------------|
| - list<Employee>    |
+---------------------+
```

#### How It Works
- `Employee` is an abstract class with common attributes and a `save()` method.
- `FullTimeEmployee` and `Intern` extend `Employee`.
- `EmployeeRepo` maintains an in-memory list of all employees.
- Client code: get the list from `EmployeeRepo`, iterate, call `e.save()` for each employee.

#### File Format (same for all employees)
```json
{ "name": "...", "email": "...", "type": "full_time/intern" }
```

#### Teacher's Evaluation of V0

**Question asked**: Is this design following SRP?

**Answer**: **No.** The `save()` method inside `Employee` is a **monster method** that does too much:
1. Converts attributes to a formatted string
2. Opens a file (needs file address)
3. Writes data to the file
4. Closes the file

**Multiple reasons to change the Employee class**:
| Change Scenario | Which class changes? |
|---|---|
| Add/remove employee attributes | Employee |
| Change the output format | Employee (save method) |
| Move from file to SQL database | Employee (save method) |

> **Teacher's Insight**: "Ideally, entity classes should not be affected because of the type of database you are using or because of external factors. The employee class should only be changed if there is a change in the *structure* of the employee -- its attributes or intrinsic behavior."

---

### V0.5 -- Moving save() to EmployeeRepo

#### Student Suggestion
Move `save()` out of `Employee` into `EmployeeRepo`.

#### Teacher's Evaluation
Better, but **SRP is still violated**. The method is still responsible for:
1. Creating the formatted string
2. Writing to the file

If either the format or the storage mechanism changes, the same method changes.

---

### V1 -- Proper Separation of Concerns

#### Key Idea: Split into Three Responsibilities

| Responsibility | Class |
|---|---|
| Create the formatted string | `Serializer` |
| Write/read data to/from file | `FileStorageService` |
| Maintain in-memory employee list + orchestrate save | `EmployeeRepo` |

#### V1 Class Diagram

```
+---------------------+
|  Employee (abstract) |
|---------------------|
| - name, email, type |
+---------------------+
      ^          ^
      |          |
+------------+ +---------+
| FullTimeEmp| | Intern  |
+------------+ +---------+

+---------------------+
| Serializer          |
|---------------------|
| + serialize(emp): String |
+---------------------+

+-----------------------+
| FileStorageService    |
|-----------------------|
| + save(data)          |
| + load(): data        |
+-----------------------+

+---------------------+
| EmployeeRepo        |
|---------------------|
| - list<Employee>    |
| - serializer: Serializer        |
| - fileStorage: FileStorageService|
|---------------------|
| + save(employee)    |
+---------------------+
```

#### How `save()` Works in V1
```java
class EmployeeRepo {
    Serializer serializer;
    FileStorageService fileStorage;

    // Objects injected through constructor
    EmployeeRepo(Serializer serializer, FileStorageService fileStorage) {
        this.serializer = serializer;
        this.fileStorage = fileStorage;
    }

    void save(Employee emp) {
        String data = serializer.serialize(emp);  // Step 1: format
        fileStorage.save(data);                    // Step 2: persist
    }
}
```

#### Why Constructor Injection, Not Local Object Creation?

The teacher discusses whether `Serializer` and `FileStorageService` objects should be created inside `save()` or passed through the constructor.

- **Only one object of each is ever needed** (they hold no state, just provide functionality).
- Creating them inside `save()` means a new object on every call -- wasteful.
- **Better approach**: Pass them via the constructor and store as member variables.
- **Even better (future topic)**: Ensure only one object can ever be created (Singleton pattern -- "we will learn how to do that soon").

#### SRP Evaluation of V1

| Change Scenario | Which class changes? |
|---|---|
| Change format (e.g., JSON to XML) | `Serializer` only |
| Change file path | `FileStorageService` only |
| Add/remove employee attributes | `Employee` only |
| **Switch from File to SQL** | **`EmployeeRepo.save()` still changes!** |

> **Remaining Problem**: If you switch from file storage to SQL, you need to replace `FileStorageService` with an `SQLStorageService` object inside `EmployeeRepo`. The `save()` method in `EmployeeRepo` must be modified. This problem is left open -- the teacher says "we will come back to this."

> **Teacher's Insight**: This is where Dependency Inversion / an interface for the storage service would help. (Hinted at but not fully resolved in this lecture.)

---

### V1.1 -- Adding Tax Calculation

#### New Requirement
Add tax calculation:
- Same for all employees: **20% income tax + 2% professional tax**

#### Where Should `calculateTax()` Go?

| Option | Verdict | Why |
|---|---|---|
| Inside `Employee` class | **No** | Tax rules change every year due to government regulations. Employee class should not change because of external tax policy. |
| Inside `EmployeeRepo` | **No** | Completely unrelated responsibility. Repo maintains in-memory data. |
| **Separate utility class** | **Yes** | Single responsibility -- only implements government-defined tax rules. |

```
+---------------------------+
| TaxCalculationUtil        |
|---------------------------|
| + calculateTax(Employee): double |
+---------------------------+
```

**Why pass `Employee` and not just `salary`?** Tax rules may require gender, age, or other employee details in addition to salary.

---

### V1.2 -- Different Tax Rules Per Employee Type

#### Changed Requirement
- Full-time employees: **30% income tax + 2% professional tax**
- Interns: **20% income tax only**

#### Bad Approaches Evaluated

**Bad Approach 1 -- if-else inside calculateTax()**:
```java
double calculateTax(Employee emp) {
    if (emp.type == "FullTime") {
        return emp.salary * 0.30 + emp.salary * 0.02;
    } else if (emp.type == "Intern") {
        return emp.salary * 0.20;
    }
}
```
**Problems**: Not maintainable, not extensible. Adding a new employee type means modifying existing tested code.

**Bad Approach 2 -- Separate methods per type in the same class**:
```java
class TaxCalculationUtil {
    double calculateTaxForFT(Employee emp) { ... }
    double calculateTaxForIntern(Employee emp) { ... }
}
```
**Problems**:
- Client loses **abstraction** -- must know which method to call.
- Client must read the entire class to understand available methods.
- If a new employee type is added, client cannot trust whether a method exists for it.

#### Correct Approach -- Interface + Polymorphism

```
+----------------------------+
| <<interface>>              |
| TaxCalculatorUtil          |
|----------------------------|
| + calculateTax(Employee)   |
+----------------------------+
       ^              ^
       |              |
+----------------+ +------------------+
| FTTaxCalculator| | InternTaxCalc    |
+----------------+ +------------------+
| +calculateTax()| | +calculateTax()  |
+----------------+ +------------------+
```

#### How the Mapping Works -- Injecting the Right Calculator

The `Employee` class gets a `TaxCalculatorUtil` reference as a member variable, passed through the constructor.

```java
abstract class Employee {
    TaxCalculatorUtil tcu;

    Employee(TaxCalculatorUtil tcu) {
        this.tcu = tcu;
    }

    abstract double calculateTax();
}

class FullTimeEmployee extends Employee {
    FullTimeEmployee(TaxCalculatorUtil tcu) {
        super(tcu);
    }

    double calculateTax() {
        return tcu.calculateTax(this);
    }
}

class InternEmployee extends Employee {
    InternEmployee(TaxCalculatorUtil tcu) {
        super(tcu);
    }

    double calculateTax() {
        return tcu.calculateTax(this);
    }
}
```

#### Object Creation -- Wiring It Together
```java
FullTimeEmployee ft = new FullTimeEmployee(new FTTaxCalculator());
InternEmployee intern = new InternEmployee(new InternTaxCalculator());
```

- At construction time, you already know the type, so you pass the correct calculator.
- The client only ever calls `e.calculateTax()` -- completely **agnostic** of the actual type.
- The correct implementation is dispatched automatically via polymorphism.

#### Why This Works

| Scenario | What changes? | What stays the same? |
|---|---|---|
| Tax rule changes for full-time | `FTTaxCalculator` only | Everything else |
| Tax rule changes for interns | `InternTaxCalculator` only | Everything else |
| New employee type (e.g., Contractual) | Add new class + new calculator | Nothing existing is modified |

> **This follows OCP**: Open for extension (add new class + calculator), closed for modification (nothing existing changes).

> **Teacher's Insight on Singleton**: "These calculator classes don't maintain state. They just provide functionality. So one single object is sufficient. We need to design these classes so that only one object is created -- no matter how many times `new` is called, the same object is returned. We will learn how to do this soon." (Foreshadowing the **Singleton Pattern**.)

---

## 3. The Design Process -- Teacher's Golden Framework

This is the most important takeaway of the lecture.

### Step-by-Step Design Process

1. **Be very clear with the requirements.** Understand the current scope and future scope.
2. **Start with an intuitive V0 design.** Don't overthink it. Put something on paper.
3. **Evaluate V0 against SOLID principles.** Ask: Is it maintainable? Understandable? Extensible?
4. **Identify violations and improve.** Move to V1.
5. **Repeat.** Evaluate V1, find remaining issues, improve further.
6. **Accept imperfection.** You can never get the most perfect design in the first iteration. It is always iterative.

### When to NOT Over-Engineer

> "If it is very clear that there is going to be only one type of employee, even in future you will never have different types, then there is no sense of creating multiple child classes. You can have just one single class."

> "Sometimes you will purposefully violate some of the principles. Most of the times you will **prioritize your deadlines**. If you have to deliver a feature the next day, you will not worry about principles being violated. You will worry about whether it is functional. Then when you have time, you will revisit and refactor."

---

## 4. Interview Preparation

### Two Types of Design Interviews

| Aspect | LLD Design Interview | Machine Coding Round |
|---|---|---|
| **Requirements** | Abstract, unclear (e.g., "Design Tic-Tac-Toe") | Well-defined, specific (APIs listed) |
| **First Step** | Spend 10-15 mins discussing & clarifying requirements | Requirements are already clear |
| **Deliverable** | Class diagram + design discussion | Working code + good design |
| **Flow** | Design -> Follow-up feature questions to test extensibility | Implement end-to-end |
| **Primary Focus** | Extensibility, maintainability, communication | **Functionality first** -- code must work |
| **Who asks this** | Large companies (Amazon, Google, Microsoft) | Startups & mid-size (Flipkart, Udaan, Swiggy) |
| **Language** | Sometimes language-agnostic | Often Java (Flipkart, Swiggy) but varies |

### Common LLD Interview Questions (mentioned in class)
- Design a Bird (Amazon -- discussed in previous class)
- Design Tic-Tac-Toe
- Design a Parking Lot
- Design an ATM Machine
- Design Splitwise

### Teacher's Interview Tips

1. **Always clarify requirements first.** Spend 10-15 minutes asking questions before drawing anything.
2. **Communicate your decisions** to the interviewer as you go. Don't design in silence.
3. **Expect follow-up features.** After you submit a design, the interviewer will test extensibility by asking "What if we add X?"
4. **If time is short, implement a bad design that works, then explain the better design.**

> "It's recommended that you implement a bad design, make it work, but then communicate to the interviewer that because of lack of time you implemented it this way. However, you have a better design ready which, if given time, you can make extensible, maintainable, and understandable."

---

## 5. Key Design Decisions Summary

| Decision | Chosen Approach | Why |
|---|---|---|
| Where to put `save()` | Not in `Employee`. In `EmployeeRepo` using injected collaborators. | Employee should only change for structural changes to the employee itself. |
| How to handle formatting | Separate `Serializer` class | Format changes should not touch Employee or Repo. |
| How to handle file I/O | Separate `FileStorageService` class | Storage mechanism changes should be isolated. |
| Where to put `calculateTax()` | Not in `Employee` directly. Delegated to `TaxCalculatorUtil` interface implementations. | Tax rules change yearly due to government policy -- unrelated to employee structure. |
| How to dispatch correct tax logic | Inject the right `TaxCalculatorUtil` implementation via constructor | Client stays agnostic; polymorphism handles dispatch. |

---

## 6. Important Concepts & Definitions

| Concept | Definition |
|---|---|
| **Entity Class** | A class that represents a real-world entity (e.g., Employee). Should only change when the entity's structure changes. |
| **Utility Class** | A class that provides a specific utility/functionality (e.g., TaxCalculationUtil). Usually stateless. |
| **Repository Class** | A class responsible for maintaining the in-memory collection of entities and orchestrating persistence. |
| **Serializer** | A class responsible for converting an object into a specific string/format for storage or transmission. |
| **Storage Service** | A class responsible for reading from and writing to a specific storage medium (file, database, etc.). |
| **Monster Method** | A method that does too many things -- multiple reasons to change. Anti-pattern. |
| **Class Explosion** | Having too many tiny classes due to over-application of SRP. Also an anti-pattern. |
| **Constructor Injection** | Passing dependencies (collaborator objects) through the constructor rather than creating them internally. |

---

## 7. Exam Prep -- Quick-Fire Facts

1. SRP applies at all levels: function, class, package, microservice.
2. Over-applying SRP leads to class explosion.
3. Entity classes should be agnostic of storage mechanisms.
4. If a method has to change for format changes AND storage changes, SRP is violated.
5. Utility/stateless classes should ideally have only one instance (Singleton pattern -- coming soon).
6. Use interfaces when you need multiple implementations of the same behavior.
7. Use abstract classes when you have shared attributes/state + some shared behavior.
8. Constructor injection is preferred over creating dependencies inside methods.
9. The design process is always iterative: V0 -> evaluate -> V1 -> evaluate -> ...
10. In machine coding rounds: **functionality first**, design second.
11. In LLD interviews: **design first**, then follow-up feature questions.
12. Amazon example question: "Design a Bird" -- pure design, no code expected.

---

## 8. Teacher's Special Insights & Real-World Wisdom

- **"You can never get the most perfect design in the first iteration."** -- Design is always iterative. Accept that V0 will be flawed.
- **"Most of the times you will prioritize your deadlines."** -- In real companies, shipping on time beats perfect design. Refactor later.
- **"Before you go and start implementation, hold yourself, take a step back."** -- The purpose of LLD is to think before coding.
- **"We haven't done any implementations as of now. These are just boxes."** -- Design is about structure, not code. Code comes after confidence in the design.
- **On the number of classes feeling overwhelming**: "Ideally, when you start developing, you will start with one single class and implement many things inside it. We are skipping the normal methods to focus on the design decisions that create new classes/interfaces."
- **"First step is always being clear with the requirements."** -- Both in interviews and real work.
- **On future-proofing vs over-engineering**: Only create abstractions if you know (or strongly expect) variation. If only one type of employee will ever exist, one class is fine.

---

## 9. What Was Left Open / Coming Soon

1. **How to ensure only one object of a stateless class exists** -- Singleton Pattern (hinted multiple times).
2. **How to make EmployeeRepo agnostic of the storage type** (file vs SQL vs Mongo) -- likely Dependency Inversion Principle or Strategy Pattern with a storage interface.
3. **Remaining two SOLID principles** (Interface Segregation Principle and Dependency Inversion Principle) -- not yet covered.

---

## 10. Assignments Released

- A set of assignments with detailed README files containing hints.
- Each assignment tells you which SOLID principle is being violated.
- Task: Refactor the code to fix violations.
- Submit: GitHub repo link via a form.
- Deadline: Monday morning (form released Sunday evening).
- Additional resources: Books, notes, and code snippets added to the syllabus sheet.
