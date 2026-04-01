# Observer

### There are typically 4 paricipants in Observer Design Pattern:   
  a. Subject: entity whose state or behavior is being observed.   
  b. Observer: entity that issues certain event when there is any state change in the subject.    
  c. Publisher: that notifies observers about state change of the subject and provides subscription mechanism to the orchestrator.    
  d. Orchestrator: initializes subject by passing publisher and adds or removes observer from subscription list.

example:
```java
//Orchestrator
class Application{
 public static void main(String[] args) {
 
 //Observers
 Backround backround = new Backround();
 Music music = new Music(); 
 
 //Publisher
 EventManager eventManager = new EventManager(); 
 eventManager.addObserver("open", backround); 
 eventManager.addObserver("open", music);

 //Subject
 Subject subject = new Subject(eventManager);
 subject.open();
 }
}
```
---
### 2-ways for having publisher class:


**1. Subject == Publisher Design**

- Subject (e.g., Monster) contains:
  - List<Observer> observers
  - notifyAll() → loops through observers and calls update()
  - addObserver(Observer o)
  - removeObserver(Observer o)

- Observer Interface:
  - update()

- Concrete Observers:
  - BackgroundHandler → update(): change background based on state
  - MusicHandler → update(): change music based on state
  - PlayerStateHandler → update(): change player attributes based on state

---

**2. Subject != Publisher Design**

- Subject (e.g., Monster) contains:   
  - Business Logic  

- Publisher contains:
  - List<Observer> observers
  - notifyAll() → loops through observers and calls update()
  - addObserver(Observer o)
  - removeObserver(Observer o)

- Observer Interface:
  - update()

- Concrete Observers:
  - BackgroundHandler → update(): change background based on state
  - MusicHandler → update(): change music based on state
  - PlayerStateHandler → update(): change player attributes based on state

---

### Argument of Observer

Instead of writing a different method signature for every observer, just pass the full subject reference. Each observer picks what it needs. This is more flexible — if new attributes are added to the subject later, no method signature changes are needed.

---

### Handling Legacy/Pre-written Classes — Using Adapter Pattern

Problem: In the textbook Observer pattern, observer classes implement the Observer interface. But in our scenario, the classes (Background, Music, etc.) are pre-written legacy code. We cannot modify them to implement the Observer interface.


#### Adapter Implementation

Do NOT modify existing classes.

Create adapter/wrapper classes that implement the Observer interface.

Each adapter holds an object of the original class internally.

The adapter's update() method calls the appropriate method on the wrapped class.

example:
```java
class BackgroundAdapter implements Observer {
 Background background; // holds the legacy object

 BackgroundAdapter(Background bg) {
  this.background = bg;
 }

 void update() {
  background.changeBackground(); // delegate to legacy method
 }
}
```

Teacher's Insight: Never modify already existing, tested legacy classes. Use adapters to bridge the gap between your new design pattern and old code.

