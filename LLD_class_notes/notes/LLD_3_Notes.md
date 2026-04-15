# LLD Lecture 3 - Detailed Notes
## Topic: Applying SOLID Principles to Design an HR Management System

---

## Quick Recap of SOLID Principles (from previous lectures)

| # | Principle | One-Liner |
|---|-----------|-----------|
| S | **Single Responsibility Principle (SRP)** | Every class/function/package should have **one primary reason to change**. |
| O | **Open-Closed Principle (OCP)** | Classes should be **open for extension, closed for modification**. |
| L | **Liskov Substitution Principle (LSP)** | A parent class object should be **completely replaceable** by any child class object without errors. |
| I | Interface Segregation | (Covered later) |
| D | Dependency Inversion | (Covered later) |

**Key nuance from the teacher:**
- For **functions**, the "single reason" is very specific.
- For **classes**, it is slightly less specific.
- For **packages**, it becomes abstract.
- For **microservices**, it is even more abstract.
- You cannot always enforce SRP perfectly -- over-applying it leads to **class explosion** (too many tiny classes). Be careful.

---

## The Design Journey (Step by Step)

### The Problem Statement

Build a small **HR Management System** with the following requirements:
1. Support two types of employees: **Full-Time Employee** and **Intern**.
2. Implement a **save** method that writes an employee's data to a file in a specific format (e.g., JSON with fields: name, email, type).
3. The same save method should work for **all types of employees**.

---

### V0 -- The Naive / Intuitive Design

**What most students came up with:**

```
+---------------------+
|    Employee (abstract)    |
+---------------------+
| - name              |
| - email             |
| - type              |
+---------------------+
| + save()            |  <-- save method lives HERE
+---------------------+
      ^          ^
      |          |
+-----------+  +-----------+
| FullTime  |  |  Intern   |
| Employee  |  |  Employee |
+-----------+  +-----------+
```

Additionally, there is an **EmployeeRepo** class:
```
+---------------------+
|   EmployeeRepo      |
+---------------------+
| - List<Employee>    |  <-- in-memory list of all employees
+---------------------+
```

**How the client uses it:**
```java
EmployeeRepo repo = new EmployeeRepo();
List<Employee> employees = repo.list;

for (Employee e : employees) {
    e.save();  // writes to file
}
```

**Teacher's note on save():** Since the format is the same for all employee types right now, the save method can live in the parent `Employee` class. If different formats were needed per type, we would make `save()` abstract and implement it in each child class.

---

### V0 Evaluation -- What is Wrong?

The teacher asks: **How many reasons does the Employee class have to change?**

| Change Scenario | Which class changes? |
|----------------|---------------------|
| Add/remove an employee attribute (e.g., add "address") | Employee class |
| Change the save **format** (e.g., name first vs type first) | Employee class (save method) |
| Change the **storage system** (file to SQL database) | Employee class (save method) |

**Verdict:** The Employee class has **multiple reasons to change** --> **SRP VIOLATED**.

**Why the save() method is a "monster method":**
Inside `save()`, all of the following happens:
1. Convert attributes to a formatted string
2. Get the file address
3. Open the file
4. Write to the file
5. Close the file

If the format changes, this method changes. If the file path changes, this method changes. If we switch to a database, this method changes. **Too many responsibilities in one method.**

**Key principle from the teacher:** Entity classes (like Employee) should only change if the **structure** of the entity changes (attributes, intrinsic behavior). They should be **agnostic** of external concerns like storage, formatting, etc.

---

### V1 -- Fixing SRP (Extracting Responsibilities)

**Step 1:** Move `save()` out of Employee class.

Some students suggested moving it to `EmployeeRepo`. But the teacher pointed out: even in the repo, the method is still doing two things (formatting + file I/O).

**Step 2:** Create separate classes for each responsibility.

```
+---------------------+        +---------------------+
|    Serializer       |        | FileStorageService  |
+---------------------+        +---------------------+
| + serialize(Employee)|       | + save(String data) |
|   : String          |        | + load() : String   |
+---------------------+        +---------------------+
```

| Class | Single Responsibility |
|-------|----------------------|
| **Employee** | Defines attributes & intrinsic behavior of an employee |
| **Serializer** | Converts an Employee object into a formatted string |
| **FileStorageService** | Handles all file operations (open, write, read, close) |
| **EmployeeRepo** | Maintains in-memory list of employees; orchestrates save |

**How save() works now (inside EmployeeRepo):**
```java
class EmployeeRepo {
    Serializer serializer;
    FileStorageService fileStorage;

    // Initialized via constructor (not created inside methods)
    EmployeeRepo(Serializer serializer, FileStorageService fileStorage) {
        this.serializer = serializer;
        this.fileStorage = fileStorage;
    }

    void save(Employee employee) {
        String data = serializer.serialize(employee);
        fileStorage.save(data);
    }
}
```

**Why pass via constructor instead of creating inside save()?**
- We only need **one object** of Serializer and FileStorageService (they are stateless -- no data, just functionality).
- Creating them inside the method would create new objects on every call -- wasteful.
- Making them member variables initialized via the constructor ensures reuse.
- (Teacher hints: later we will learn how to enforce that only one object can ever be created -- **Singleton Pattern**.)

---

### V1 Evaluation -- Is SRP Fully Followed Now?

| Change Scenario | Which class changes? |
|----------------|---------------------|
| Change the format | Serializer |
| Change file path | FileStorageService |
| Add/remove employee attribute | Employee |
| **Switch from File to SQL** | **EmployeeRepo** (still!) |

**Remaining problem:** If we switch from File storage to SQL storage, we need to replace `FileStorageService` with `SQLStorageService` inside `EmployeeRepo.save()`. The repo class still changes.

**Teacher says:** "We will come back to this problem after some time and see how to ensure that even if we are updating the database, this function remains unaffected." (This foreshadows the **Dependency Inversion Principle** and **Strategy Pattern** -- covered in later lectures.)

---

### V2 -- Adding Tax Calculation Feature

**New requirement:** Add income tax calculation.
- Tax = **20% income tax + 2% professional tax** (same for all employees initially).

**Where should `calculateTax()` go?**

| Option | Pros | Cons |
|--------|------|------|
| Inside Employee class | Has direct access to salary attribute | Tax rules change every year due to government policy -- Employee class would change for reasons unrelated to its structure --> **SRP Violation** |
| Separate utility class | Employee stays clean; tax logic is isolated | Need to pass employee/salary as argument |

**Teacher's decision:** Create a **separate utility class**.

```
+-------------------------+
| TaxCalculationUtil      |
+-------------------------+
| + calculateTax(Employee)|
|   : double              |
+-------------------------+
```

**Why not inside Employee?**
> "Tax calculation is something which is expected to change every single year because of rules defined by the government. Should we change the Employee class because of it? No."

**Why pass Employee (not just salary)?**
> Tax rules may require the **gender** or **age** of the employee. Passing the whole Employee object gives flexibility.

---

### V3 -- Tax Rules Differ by Employee Type

**Changed requirement:**
- Full-time employees: **30% income tax + 2% professional tax**
- Interns: **20% income tax** (no professional tax)

**Bad approach 1 -- if-else inside one method:**
```java
// BAD: Not maintainable, not extensible
double calculateTax(Employee e) {
    if (e.type == "intern") {
        return e.salary * 0.20;
    } else if (e.type == "fulltime") {
        return e.salary * 0.30 + e.salary * 0.02;
    }
}
```
**Why bad:** Adding a new employee type means modifying this method --> violates OCP. The if-else chain grows uncontrollably.

**Bad approach 2 -- separate methods in one class:**
```java
// BAD: Breaks abstraction
class TaxCalculationUtil {
    double calculateTaxForFT(Employee e) { ... }
    double calculateTaxForIntern(Employee e) { ... }
}
```
**Why bad:**
- Client must know which method to call --> **abstraction is broken**.
- Client must check "is there a method for this employee type?" --> fragile.
- Adding a new employee type requires adding a new method AND updating the client.

---

### V3 Final Design -- Interface + Polymorphism

**Step 1:** Make `TaxCalculatorUtil` an **interface** (since it has only one abstract method, no state):

```java
interface TaxCalculatorUtil {
    double calculateTax(Employee employee);
}
```

**Step 2:** Create **concrete implementations** for each employee type:

```java
class FTTaxCalculator implements TaxCalculatorUtil {
    double calculateTax(Employee employee) {
        return employee.salary * 0.30 + employee.salary * 0.02;
    }
}

class InternTaxCalculator implements TaxCalculatorUtil {
    double calculateTax(Employee employee) {
        return employee.salary * 0.20;
    }
}
```

**Step 3:** Add the tax calculator as an attribute of Employee, injected via constructor:

```java
abstract class Employee {
    String name;
    String email;
    TaxCalculatorUtil tcu;  // interface reference

    Employee(TaxCalculatorUtil tcu) {
        this.tcu = tcu;
    }

    abstract double calculateTax();
}
```

```java
class FullTimeEmployee extends Employee {
    FullTimeEmployee(TaxCalculatorUtil tcu) {
        super(tcu);
    }

    double calculateTax() {
        return tcu.calculateTax(this);  // delegates to FTTaxCalculator
    }
}

class InternEmployee extends Employee {
    InternEmployee(TaxCalculatorUtil tcu) {
        super(tcu);
    }

    double calculateTax() {
        return tcu.calculateTax(this);  // delegates to InternTaxCalculator
    }
}
```

**Step 4:** Object creation wires the correct calculator:

```java
// The correct tax calculator is injected at creation time
Employee ft = new FullTimeEmployee(new FTTaxCalculator());
Employee intern = new InternEmployee(new InternTaxCalculator());
```

**How the client uses it (completely agnostic):**
```java
for (Employee e : employees) {
    double tax = e.calculateTax();  // polymorphism handles the rest
    // Client does NOT know or care which calculator is used
}
```

---

### V3 Evaluation -- OCP Check

| Scenario | What happens? | Existing code modified? |
|----------|--------------|------------------------|
| Add a new employee type (e.g., Contractual) | Create `ContractualEmployee` class + `ContractualTaxCalculator` class | **No** -- nothing existing changes |
| Change tax rules for full-time employees | Modify `FTTaxCalculator.calculateTax()` | Only that one class |
| Change tax rules for interns | Modify `InternTaxCalculator.calculateTax()` | Only that one class |

**Verdict:** Follows **OCP** -- open for extension, closed for modification.

---

### Final Class Diagram (V3)

```
+---------------------------+
|   Employee (abstract)     |
+---------------------------+
| - name: String            |
| - email: String           |
| - tcu: TaxCalculatorUtil  |
+---------------------------+
| + calculateTax(): double  |  (abstract)
+---------------------------+
        ^            ^
        |            |
+---------------+ +---------------+
| FullTimeEmp   | | InternEmp     |
+---------------+ +---------------+
| calculateTax()| | calculateTax()|
+---------------+ +---------------+

+---------------------------+       +---------------------------+
| <<interface>>             |       |     Serializer            |
| TaxCalculatorUtil         |       +---------------------------+
+---------------------------+       | + serialize(Employee):    |
| + calculateTax(Employee)  |       |   String                  |
+---------------------------+       +---------------------------+
        ^            ^
        |            |
+---------------+ +-------------------+
| FTTaxCalc     | | InternTaxCalc     |
+---------------+ +-------------------+

+---------------------------+       +---------------------------+
|   EmployeeRepo            |       |  FileStorageService       |
+---------------------------+       +---------------------------+
| - List<Employee>          |       | + save(String)            |
| - serializer: Serializer  |       | + load(): String          |
| - fileStorage: FileStorage|       +---------------------------+
+---------------------------+
| + save(Employee)          |
+---------------------------+
```

---

## Interview & Exam Quick-Fire Points

1. **SRP does not mean one method per class.** It means one *primary reason to change*. A class can have many methods if they all serve the same responsibility.
2. **Entity classes should not handle persistence.** Employee should not know how or where its data is saved.
3. **Monster methods** are methods that do too many things (formatting + file I/O + error handling). Break them up.
4. **Stateless utility classes** (like Serializer, TaxCalculator) should ideally have only **one object** (Singleton pattern -- covered later).
5. **if-else on type** is almost always a design smell. Use **polymorphism** instead.
6. **Constructor injection** (passing dependencies via constructor) is preferred over creating objects inside methods.
7. **Interfaces should be used** when a class has no state and only defines behavior (like TaxCalculatorUtil).
8. **Abstract classes** should be used when you have shared state/attributes + some shared behavior (like Employee).
9. **OCP check:** "Can I add a new type of [X] without modifying existing tested code?" If yes, OCP is followed.
10. **The design process:** V0 (intuitive) --> Evaluate against SOLID --> V1 (fix violations) --> Evaluate again --> V2 (add features) --> repeat.

---

## Design Decisions -- Why One Approach Over Another

| Decision | Chosen Approach | Rejected Approach | Why |
|----------|----------------|-------------------|-----|
| Where to put `save()`? | EmployeeRepo (orchestrator) | Employee class | Employee should not know about file systems or formats (SRP) |
| How to handle formatting? | Separate Serializer class | Inside save() method | Formatting and file I/O are independent concerns (SRP) |
| Where to put `calculateTax()`? | Separate TaxCalculatorUtil + inside Employee as abstract method | Directly in Employee class | Tax rules change independently of employee structure (SRP) |
| How to handle different tax rules per type? | Interface + concrete classes + polymorphism | if-else chain or separate named methods | Extensible (OCP), clean abstraction for client |
| How to create utility objects? | Pass via constructor (dependency injection) | Create inside methods | Avoids redundant object creation; enables Singleton later |

---

## Teacher's Special Insights

### On Real-World Design
> "You can never get the most perfect design in the first iteration. It is always going to be a process where you will evaluate, make changes, then probably realize the changes are actually violating a lot of things and go back."

### On Deadlines vs Clean Design
> "Most of the times you will prioritize your deadlines. If you have to deliver a feature the next day or the next week, you will not worry about any of these principles being violated. You will worry about whether it is functional or not. Then when you have time, you will revisit your code and refactor it."

### On Requirements Clarity
> "First step is always being clear with the requirements. If it is very clear that there is going to be only one type of employee even in future, there is no sense of creating multiple child classes."

### On Class Explosion
> "Right now we are taking examples which need too many classes. Ideally, when you start developing, you will start with one single class and implement a lot of things inside it. It's not like every use case will come up with four or five new classes."

### On the Design Process (Thumb Rule)
1. Understand the requirements thoroughly.
2. Come up with a V0 (intuitive, possibly naive) design.
3. Evaluate it: Is it maintainable? Understandable? Extensible?
4. Check against SOLID principles.
5. Fix violations --> V1.
6. Add new requirements --> repeat evaluation.
7. **Do all of this BEFORE writing any implementation code.**

---

## Interview Types -- LLD vs Machine Coding

| Aspect | LLD Design Interview | Machine Coding Round |
|--------|---------------------|---------------------|
| **Companies** | Amazon, Google, Microsoft | Flipkart, Udaan, Swiggy |
| **Requirements** | Abstract (e.g., "Design a parking lot") | Well-defined, specific APIs |
| **Your job** | Discuss requirements (10-15 min) --> Class diagram --> Communicate decisions --> Handle follow-up features | Come up with extensible design AND implement it end-to-end |
| **Primary focus** | Extensibility, maintainability of design | **Working code first**, then clean design |
| **Follow-ups** | Interviewer adds new features to test extensibility | Usually no follow-ups; time-boxed |
| **Language** | Language-agnostic (usually) | Often Java (Flipkart, Swiggy); some companies are flexible |

**Teacher's tip for Machine Coding:**
> "If your extensible design has 100-200 classes and you don't have time to implement all of them, implement a bad design that works. Then communicate to the interviewer that you have a better design ready which, given time, you can refactor."

---

## Exam Prep -- Quick Revision Table

| Concept | Definition | Example from this lecture |
|---------|-----------|--------------------------|
| **SRP Violation** | Class/method has multiple reasons to change | Employee.save() handles formatting + file I/O |
| **SRP Fix** | Extract each responsibility into its own class | Serializer for formatting, FileStorageService for I/O |
| **OCP Violation** | Adding new feature requires modifying existing code | if-else on employee type inside calculateTax() |
| **OCP Fix** | Use interfaces + polymorphism | TaxCalculatorUtil interface with per-type implementations |
| **Monster Method** | A method doing too many unrelated things | save() that formats, opens file, writes, closes |
| **Entity Class** | Represents a real-world object with attributes | Employee, FullTimeEmployee, InternEmployee |
| **Utility Class** | Stateless class providing functionality | TaxCalculationUtil, Serializer |
| **Repository Class** | Manages in-memory collection of entities | EmployeeRepo with List<Employee> |
| **Constructor Injection** | Passing dependencies through the constructor | Passing TaxCalculatorUtil to Employee constructor |
| **Delegation** | A method calls another object's method to do the real work | Employee.calculateTax() calls tcu.calculateTax(this) |

---

## Summary of Design Versions

| Version | What Changed | Key Principle Applied |
|---------|-------------|----------------------|
| **V0** | Basic design: Employee has save(), EmployeeRepo has list | -- (intuitive, flawed) |
| **V0 --> V1** | Extracted Serializer + FileStorageService from Employee | **SRP** |
| **V1 --> V2** | Added TaxCalculationUtil as separate class | **SRP** |
| **V2 --> V3** | Made TaxCalculatorUtil an interface with per-type implementations; injected via constructor | **OCP + Polymorphism** |

---

*These notes cover LLD Lecture 3. The remaining SRP issue (storage type change affecting EmployeeRepo) will be addressed in a future lecture using the Dependency Inversion Principle.*
