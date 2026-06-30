# Design Patterns Interview Guide

## 1. Overview
At Staff+ level, design patterns are a shared vocabulary for solving recurring design problems, not a checklist to be applied blindly. You must demonstrate:
- **Deep understanding** of the forces that lead to each pattern, its trade-offs, and when it becomes an anti-pattern.
- **Ability to communicate design decisions** using pattern names and Architectural Decision Records (ADRs).
- **Modern Java usage** – patterns with lambdas, streams, dependency injection, records, sealed types.
- **Real-world judgement** – how patterns affect testability, performance, and maintainability in large codebases.

## 2. Pattern-by-Pattern Deep Dive
*(For each pattern: simple explanation, problem it solves, structure, how to implement, when to avoid, pros/cons, and code example.)*

### 2.1 Creational Patterns

#### Singleton
- **Simple Explanation**: Ensure a class has only one instance and provide a global access point.
- **Why (Problem)**: Some objects should be shared (caches, thread pools, logging). Multiple instances cause bugs or waste.
- **What**: A static method returns the same instance; constructor is private.
- **How**:
  1. Make constructor `private`.
  2. Create a `private static` instance (eager or lazy with double-checked locking).
  3. Expose a `public static getInstance()`.
  Better: use enum or let a DI container manage singleton scope.
- **When Not to Use**: When the instance is not truly a single resource, or when it makes testing difficult (hidden global state). Avoid in distributed systems where “singleton” across JVMs is impossible.
- **Pros & Cons**:
  - Pros: controlled access, lazy loading, avoids resource duplication.
  - Cons: hard to unit test (global state), can become a bottleneck, hidden dependencies, scoping issues in application servers.
- **Code Example** (using enum, best practice):
```java
public enum CacheManager {
    INSTANCE;
    private final Map<String, Object> cache = new ConcurrentHashMap<>();
    public Object get(String key) { return cache.get(key); }
    public void put(String key, Object val) { cache.put(key, val); }
}
```

#### Factory Method
- **Simple Explanation**: Define an interface for creating an object, but let subclasses decide which class to instantiate. Or a static method that creates objects.
- **Why**: Decouple object creation from its usage. Client code shouldn’t bind to concrete classes.
- **What**: A method (often abstract) returns a product object; subclasses override to return a specific product.
- **How**:
  1. Define a `Product` interface.
  2. Create an abstract `Creator` with a `factoryMethod()` returning `Product`.
  3. Concrete creators implement `factoryMethod()` to instantiate `ConcreteProduct`.
  Alternative: simple static factory methods.
- **When Not to Use**: When creation logic is trivial or never changes; when you just need a simple constructor call.
- **Pros & Cons**:
  - Pros: flexibility, encapsulation of creation, supports open/closed principle.
  - Cons: can increase number of classes, may hide instantiation complexity.
- **Code Example**:
```java
interface Transport { void deliver(); }
class Truck implements Transport { public void deliver() { /*...*/ } }
class Ship implements Transport { public void deliver() { /*...*/ } }
abstract class Logistics {
    abstract Transport createTransport(); // factory method
    void planDelivery() { Transport t = createTransport(); t.deliver(); }
}
class RoadLogistics extends Logistics {
    Transport createTransport() { return new Truck(); }
}
```

#### Abstract Factory
- **Simple Explanation**: Factory of factories. Create families of related objects without specifying their concrete classes.
- **Why**: When system needs to be independent of how its objects are created, and products often come in families (e.g., Windows/Mac UI components).
- **What**: An interface declares methods to create each type of product; concrete factories implement those methods to return a family of products.
- **How**:
  1. Define interfaces for each product type (`Button`, `Checkbox`).
  2. Define `GUIFactory` interface with `createButton()`, `createCheckbox()`.
  3. Concrete factories like `WindowsFactory`, `MacFactory` return Windows/Mac variants.
  4. Client uses factory interface, not specific factory.
- **When Not to Use**: When product families are unlikely to change or are few; adds complexity. If only one product type varies, a simple Factory Method suffices.
- **Pros & Cons**:
  - Pros: ensures products from same family used together, easy to switch families, promotes consistency.
  - Cons: many interfaces and classes; adding new product types requires changes across all factories.
- **Code Example**:
```java
// Products
interface Button { void render(); }
class WinButton implements Button { public void render() { /* Windows style */ } }
class MacButton implements Button { public void render() { /* Mac style */ } }
// Factory
interface GUIFactory { Button createButton(); }
class WinFactory implements GUIFactory { public Button createButton() { return new WinButton(); } }
class MacFactory implements GUIFactory { public Button createButton() { return new MacButton(); } }
// Client uses GUIFactory without knowing concrete factory.
```

#### Builder
- **Simple Explanation**: Separate construction of a complex object from its representation, allowing the same construction process to create different representations.
- **Why**: To avoid telescoping constructors (many parameters), make immutable objects with many optional fields, improve readability.
- **What**: A builder class with methods to set optional parameters, and a `build()` method that returns the final product.
- **How**:
  1. Create a static inner `Builder` class inside the product, with fields matching product properties.
  2. Each method returns `this` for chaining.
  3. `build()` validates and returns a new product instance (often with private constructor).
- **When Not to Use**: For simple objects with few mandatory fields – builder adds unnecessary boilerplate. If mutability is acceptable, setters might suffice.
- **Pros & Cons**:
  - Pros: readable, enforces immutability, easy validation, flexible for optional parameters.
  - Cons: verbosity, possible performance overhead from temporary Builder objects.
- **Code Example**:
```java
public class Pizza {
    private final String size;
    private final boolean cheese;
    private final boolean pepperoni;
    private Pizza(Builder b) { this.size = b.size; this.cheese = b.cheese; this.pepperoni = b.pepperoni; }
    public static class Builder {
        private String size; // mandatory?
        private boolean cheese = false;
        private boolean pepperoni = false;
        public Builder(String size) { this.size = size; }
        public Builder addCheese() { cheese = true; return this; }
        public Builder addPepperoni() { pepperoni = true; return this; }
        public Pizza build() { return new Pizza(this); }
    }
}
// Usage: Pizza p = new Builder("large").addCheese().build();
```

#### Prototype
- **Simple Explanation**: Create new objects by cloning a prototype instance instead of calling a constructor.
- **Why**: When object creation is expensive (e.g., DB calls, complex initialization) and you need similar objects; or when you want to avoid subclass explosion.
- **What**: A prototype interface declares a `clone()` (or copy) method. Concrete prototypes implement cloning.
- **How**:
  1. Define an interface with `clone()` method.
  2. Implement classes that provide deep copies (e.g., serialization, copy constructors).
  3. Maintain a registry of prototypes; clients request a cloned instance.
- **When Not to Use**: If object has many dependencies that can’t be easily duplicated; if shallow copy is enough but prototype requires deep copy; overkill for simple objects.
- **Pros & Cons**:
  - Pros: reduces subclassing, can be more efficient than calling constructors repeatedly, dynamic creation at runtime.
  - Cons: deep copy can be complex, risk of shared mutable state if not cloned correctly.
- **Code Example**:
```java
interface Shape {
    Shape clone();
    void draw();
}
class Circle implements Shape, Cloneable {
    public int radius;
    public Circle(int r) { this.radius = r; }
    public Shape clone() { return new Circle(this.radius); }
    public void draw() { /* ... */ }
}
// Registry
Map<String, Shape> prototypes = new HashMap<>();
prototypes.put("bigCircle", new Circle(10));
Shape another = prototypes.get("bigCircle").clone();
```

#### Dependency Injection (not GoF, but essential)
- **Simple Explanation**: Instead of objects creating their own dependencies, they are “injected” from the outside (constructor, setter, interface). The framework/IoC container wires them.
- **Why**: Decouple construction from business logic; improve testability, configurable system.
- **What**: A container manages object lifecycles and injects required components.
- **How**: Define constructor arguments as interfaces; Spring (or Guice) scans and autowires. Use `@Autowired`, constructor injection preferred.
- **When Not to Use**: In very simple programs where DI adds complexity; if the team lacks DI tooling experience. Avoid field injection (testing/circular dependency issues).
- **Pros & Cons**:
  - Pros: testability, loose coupling, centralized configuration, lifecycle management.
  - Cons: runtime wiring can be magical, startup performance impact, debugging complexity.
- **Code Example** (Spring):
```java
@Service
public class OrderService {
    private final PaymentGateway paymentGateway;
    public OrderService(PaymentGateway paymentGateway) { this.paymentGateway = paymentGateway; }
    // ...
}
```

### 2.2 Structural Patterns

#### Adapter
- **Simple Explanation**: Wraps an existing class with a new interface so it can work with what the client expects.
- **Why**: Integrate legacy components or third-party libraries without modifying them; translate one interface to another.
- **What**: An adapter class implements the target interface and delegates to an adaptee.
- **How**:
  1. Identify the target interface.
  2. Create an adapter class implementing target interface, holding a reference to adaptee.
  3. Each method translates calls to adaptee.
- **When Not to Use**: When you can change the adaptee’s interface directly. Overuse can hide poor design.
- **Pros & Cons**:
  - Pros: single responsibility, open/closed principle, reusability of existing classes.
  - Cons: extra layer, may increase complexity.
- **Code Example**:
```java
// Existing
class OldPrinter { void printText(String text) { ... } }
// New interface
interface ModernPrinter { void print(String document); }
// Adapter
class PrinterAdapter implements ModernPrinter {
    private OldPrinter old;
    PrinterAdapter(OldPrinter old) { this.old = old; }
    public void print(String doc) { old.printText(doc); }
}
```

#### Decorator
- **Simple Explanation**: Attach additional responsibilities to an object dynamically by wrapping it.
- **Why**: Add behavior without altering original class or using inheritance; fine-grained control over multiple optional features (e.g., adding caching, logging, compression to a data stream).
- **What**: Both the decorator and the component implement the same interface. The decorator holds a reference to the component and adds behavior before/after delegation.
- **How**:
  1. Define component interface.
  2. Concrete component implements it.
  3. Abstract decorator implements the interface and holds a component reference.
  4. Concrete decorators override methods to add behaviors.
- **When Not to Use**: When the number of possible combinations is small and inheritance is simpler; too many decorators can create debugging difficulty.
- **Pros & Cons**:
  - Pros: flexible addition of behaviors, follows open/closed principle, replaces cascading subclasses.
  - Cons: many small objects, order of decorations can matter, hard to debug stack.
- **Code Example** (Java I/O style):
```java
interface DataSource { String readData(); }
class FileDataSource implements DataSource { /* reads file */ }
abstract class DataSourceDecorator implements DataSource {
    protected DataSource wrappee;
    DataSourceDecorator(DataSource src) { this.wrappee = src; }
}
class EncryptionDecorator extends DataSourceDecorator {
    EncryptionDecorator(DataSource src) { super(src); }
    public String readData() { String data = wrappee.readData(); return encrypt(data); }
    private String encrypt(String data) { ... }
}
// Usage: DataSource ds = new EncryptionDecorator(new FileDataSource("file.txt"));
```

#### Proxy
- **Simple Explanation**: Provide a surrogate (placeholder) for another object to control access, delay creation, or add functionality.
- **Why**: Lazy initialization (virtual proxy), access control (protection proxy), remote service (remote proxy), or logging.
- **What**: Proxy class implements the same interface as the real subject and holds a reference to it (or knows how to create it). Clients think they’re using the real object.
- **How**:
  1. Define subject interface.
  2. RealSubject implements it.
  3. Proxy implements it, holds reference to RealSubject (initially null), creates on demand or delegates with checks.
- **When Not to Use**: When overhead of proxy is not justified (e.g., simple object creation is cheap). Can add complexity.
- **Pros & Cons**:
  - Pros: control over access, lazy loading, security, logging without changing original code.
  - Cons: additional indirection, potential for identity confusion (`equals/hashCode`), added complexity.
- **Code Example** (Virtual Proxy for lazy DB connection):
```java
interface Database { void query(String sql); }
class RealDatabase implements Database { /* heavy init */ }
class DatabaseProxy implements Database {
    private RealDatabase real = null;
    public void query(String sql) {
        if (real == null) real = new RealDatabase();
        real.query(sql);
    }
}
```

#### Facade
- **Simple Explanation**: Provide a unified high-level interface to a set of interfaces in a subsystem, making it easier to use.
- **Why**: Simplify complex subsystems, reduce dependencies of external code, provide an entry point.
- **What**: A facade class exposes simplified methods that orchestrate calls to subsystem classes.
- **How**:
  1. Identify a complex subsystem with many classes.
  2. Create a facade class with methods that combine calls to those classes.
  3. Clients use facade instead of subsystem directly.
- **When Not to Use**: When the facade becomes a “god object” or when clients need fine-grained control; overuse can hide necessary flexibility.
- **Pros & Cons**:
  - Pros: reduces coupling, improves readability, hides complexity.
  - Cons: potential for becoming monolithic, may obscure advanced functionality.
- **Code Example**:
```java
class PaymentFacade {
    private AccountService account;
    private FraudService fraud;
    private TransferService transfer;
    public boolean makePayment(String from, String to, double amount) {
        if (!fraud.check(from)) return false;
        if (account.balance(from) < amount) return false;
        transfer.execute(from, to, amount);
        return true;
    }
}
```

#### Composite
- **Simple Explanation**: Compose objects into tree structures to represent part-whole hierarchies. Clients treat individual objects and compositions uniformly.
- **Why**: Need to work with nested structures (e.g., file systems, UI components, organization charts) where a single element and a group of elements have common behavior.
- **What**: A component interface (`Graphic`) with methods like `draw()`. Leaf implements directly; Composite holds a collection of children and implements `draw()` by iterating them.
- **How**:
  1. Define component interface.
  2. Implement leaf class.
  3. Implement composite class that stores components; methods delegate to children.
- **When Not to Use**: When hierarchy depth is fixed or simplicity is preferred. If leaf and composite have very different operations.
- **Pros & Cons**:
  - Pros: simplifies client code, easy to add new components, follows open/closed principle.
  - Cons: makes design overly general, might be hard to restrict what can be added to a composite.
- **Code Example**:
```java
interface FileSystemNode { int size(); }
class File implements FileSystemNode {
    private int size;
    public int size() { return size; }
}
class Directory implements FileSystemNode {
    private List<FileSystemNode> children = new ArrayList<>();
    public void add(FileSystemNode node) { children.add(node); }
    public int size() { return children.stream().mapToInt(FileSystemNode::size).sum(); }
}
```

#### Bridge
- **Simple Explanation**: Separate abstraction from implementation so both can vary independently.
- **Why**: When both abstractions and their implementations might need to be extended via inheritance, causing cartesian product explosion; e.g., different shapes × drawing APIs.
- **What**: Two layers: abstraction (high-level control) and implementation (low-level operations). Abstraction holds a reference to implementation interface and delegates.
- **How**:
  1. Define `Implementation` interface.
  2. Create concrete implementations.
  3. Define abstract `Abstraction` with reference to `Implementation`, constructor injection.
  4. Refined abstractions extend `Abstraction`.
- **When Not to Use**: If there’s only one abstraction or one implementation, or if they are unlikely to change independently.
- **Pros & Cons**:
  - Pros: decouples interface from implementation, improved extensibility, reduces subclass proliferation.
  - Cons: increased complexity, indirection.
- **Code Example**:
```java
interface Color { String fill(); }
class Red implements Color { public String fill() { return "Red"; } }
class Blue implements Color { public String fill() { return "Blue"; } }
abstract class Shape {
    protected Color color;
    Shape(Color c) { this.color = c; }
    abstract void draw();
}
class Circle extends Shape {
    Circle(Color c) { super(c); }
    void draw() { System.out.println("Circle with " + color.fill()); }
}
```

### 2.3 Behavioral Patterns

#### Strategy
- **Simple Explanation**: Define a family of algorithms, encapsulate each one, and make them interchangeable.
- **Why**: Avoid conditional statements for choosing algorithm; allow runtime selection; easy to add new algorithms.
- **What**: `Context` class holds a reference to a `Strategy` interface and delegates the algorithm. Concrete strategies implement different algorithms.
- **How**:
  1. Define `Strategy` interface with `execute()` method.
  2. Implement multiple concrete strategies.
  3. Context has a setter/constructor for strategy; calls `strategy.execute()`.
- **When Not to Use**: When algorithms are unlikely to change or require many parameters; if there are only a few algorithm variants and they’re stable, a simple conditional might suffice.
- **Pros & Cons**:
  - Pros: separation of concerns, promotes open/closed principle, easy testing, eliminates complex conditionals.
  - Cons: increases number of classes, client must be aware of strategies to select appropriate one.
- **Code Example**:
```java
interface SortingStrategy { void sort(int[] data); }
class QuickSort implements SortingStrategy { public void sort(int[] data) { /* ... */ } }
class MergeSort implements SortingStrategy { public void sort(int[] data) { /* ... */ } }
class Sorter {
    private SortingStrategy strategy;
    void setStrategy(SortingStrategy s) { this.strategy = s; }
    void sort(int[] data) { strategy.sort(data); }
}
```

#### Observer
- **Simple Explanation**: When one object changes state, all its dependents are notified and updated automatically.
- **Why**: Decouple subject and observers; subject doesn’t need to know who is listening. Used in event-driven systems, UI.
- **What**: Subject maintains list of Observer interfaces. Observers register; subject calls `notify()` on each observer, often passing data.
- **How**:
  1. Define `Observer` interface with `update(Event)`.
  2. Concrete observers implement it.
  3. Subject implements `add/remove/notifyObservers`.
- **When Not to Use**: If there’s a strictly sequential, one-to-one dependency; if observer list grows causing performance issues, or if observers cause indefinite loops.
- **Pros & Cons**:
  - Pros: loose coupling, dynamic relationships, supports broadcast communication.
  - Cons: notification order can be undefined, memory leaks if observers not removed.
- **Code Example**:
```java
interface Observer { void update(String message); }
class EmailAlerts implements Observer { public void update(String msg) { /* send email */ } }
class Stock {
    private List<Observer> observers = new ArrayList<>();
    private double price;
    public void addObserver(Observer o) { observers.add(o); }
    public void setPrice(double p) {
        this.price = p;
        for (Observer o : observers) o.update("Price changed to " + p);
    }
}
```

#### Chain of Responsibility
- **Simple Explanation**: Pass a request along a chain of handlers; each handler decides whether to process it or pass to the next.
- **Why**: Avoid coupling sender of request to a specific receiver; dynamically configure handling sequence.
- **What**: Handler interface with `handle(Request)` and pointer to next handler. Concrete handlers process or delegate.
- **How**:
  1. Define handler interface/abstract class with `setNext(Handler)`.
  2. Each concrete handler checks if it can process; if not, calls next handler.
  3. Client sends request to first handler.
- **When Not to Use**: When requests are not processed by multiple components, or if order is fixed.
- **Pros & Cons**:
  - Pros: decouples sender and receivers, flexible chains, adhering to single responsibility.
  - Cons: request may fall through unhandled, debugging difficult with long chains, performance cost.
- **Code Example**:
```java
abstract class Logger {
    public static int INFO = 1, DEBUG = 2, ERROR = 3;
    protected int level;
    private Logger next;
    void setNext(Logger next) { this.next = next; }
    void logMessage(int level, String msg) {
        if (this.level <= level) write(msg);
        if (next != null) next.logMessage(level, msg);
    }
    abstract protected void write(String msg);
}
```

#### Command
- **Simple Explanation**: Encapsulate a request as an object, thereby allowing parameterization of clients with different requests, queuing, logging, undo/redo.
- **Why**: Decouple sender and receiver; support queuing, delayed execution, transactional behavior.
- **What**: Command interface with `execute()` and maybe `undo()`. Concrete commands bind to a receiver.
- **How**:
  1. Define `Command` interface.
  2. Concrete command classes store receiver and parameters; implement `execute()` calling receiver.
  3. Invoker holds and invokes command.
- **When Not to Use**: When simple callbacks or direct method calls suffice; adds complexity for trivial operations.
- **Pros & Cons**:
  - Pros: decouples invoker and receiver, supports composite commands, undo/redo easy.
  - Cons: may lead to many classes, overkill for simple invocations.
- **Code Example** (turn light on/off):
```java
interface Command { void execute(); void undo(); }
class Light {
    void on() { /*...*/ }
    void off() { /*...*/ }
}
class LightOnCommand implements Command {
    private Light light;
    LightOnCommand(Light l) { this.light = l; }
    public void execute() { light.on(); }
    public void undo() { light.off(); }
}
class RemoteControl {
    private Command slot;
    void setCommand(Command c) { slot = c; }
    void pressButton() { slot.execute(); }
}
```

#### State
- **Simple Explanation**: Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.
- **Why**: Avoid complex conditionals based on state; each state is an encapsulated class with its own transitions.
- **What**: Context maintains a reference to a `State` interface. Concrete states implement behavior and can trigger transitions back to the context.
- **How**:
  1. Define `State` interface with methods representing events/actions.
  2. Concrete states implement methods and may set a new state on the context.
  3. Context delegates behavior to current state.
- **When Not to Use**: For small number of states and simple state transitions where a state machine enum suffices.
- **Pros & Cons**:
  - Pros: simplifies code, eliminates conditionals, makes state transitions explicit, easy to add new states.
  - Cons: many classes, states depend on context interface, can introduce coupling.
- **Code Example** (Order lifecycle):
```java
interface OrderState { void ship(Order ctx); void cancel(Order ctx); }
class Pending implements OrderState {
    public void ship(Order ctx) { ctx.setState(new Shipped()); }
    public void cancel(Order ctx) { ctx.setState(new Cancelled()); }
}
class Shipped implements OrderState {
    public void ship(Order ctx) { throw new IllegalStateException(); }
    public void cancel(Order ctx) { throw new IllegalStateException(); }
}
class Cancelled implements OrderState { ... }
class Order {
    private OrderState state = new Pending();
    void setState(OrderState s) { this.state = s; }
    void ship() { state.ship(this); }
    void cancel() { state.cancel(this); }
}
```

#### Template Method
- **Simple Explanation**: Define the skeleton of an algorithm in a method, deferring some steps to subclasses.
- **Why**: Reuse common algorithm structure, avoid code duplication among similar processes with variations.
- **What**: Abstract class with a template method (final) that calls abstract/overridable methods. Subclasses supply specific steps.
- **How**:
  1. Create abstract class with a final template method.
  2. Define abstract primitive operations that template calls.
  3. Subclasses implement those operations.
- **When Not to Use**: When algorithms vary too much at the structural level; Strategy or composition might be better. Also, when inheritance is undesirable.
- **Pros & Cons**:
  - Pros: code reuse, enforces algorithm structure, easy to add variations.
  - Cons: limitation of inheritance, can be rigid, subclass must know internal steps order.
- **Code Example**:
```java
abstract class DataProcessor {
    public final void process() { readData(); processData(); writeData(); }
    abstract void readData();
    abstract void processData();
    abstract void writeData();
}
class CSVProcessor extends DataProcessor { /* override steps */ }
```

#### Iterator
- **Simple Explanation**: Provide a way to access elements of a collection sequentially without exposing its underlying representation.
- **Why**: Uniform traversal interface for different collection types.
- **What**: `Iterator` interface with `hasNext()` and `next()`; concrete collection returns an iterator.
- **How**: Implement `Iterable` on collection, returning an `Iterator` class that knows the internal structure.
- **When Not to Use**: If collection is simple and performance critical; using for-each loop directly may suffice.
- **Pros & Cons**:
  - Pros: decouples traversal from collection, multiple simultaneous traversals.
  - Cons: can be verbose when using manual iterator in languages with for-each.
- **Code Example** (Java built-in):
```java
class MyList<T> implements Iterable<T> { ... }
MyList<String> list = new MyList<>();
for (String s : list) { ... }
```

#### Mediator
- **Simple Explanation**: Define an object that encapsulates how a set of objects interact. Promotes loose coupling by keeping objects from referring to each other explicitly.
- **Why**: Complex many-to-many interactions between components (e.g., chatroom, flight control). Without mediator, components become tightly coupled mesh.
- **What**: Mediator interface defines communication methods; concrete mediator coordinates colleague objects. Colleagues know mediator, not each other.
- **How**:
  1. Define `Mediator` interface.
  2. Concrete mediator holds references to all colleague objects.
  3. Colleagues notify mediator about events; mediator decides what to update.
- **When Not to Use**: When interactions are simple or one-to-one; otherwise it can become a God object.
- **Pros & Cons**:
  - Pros: reduces coupling, centralizes behavior, easier to change interactions.
  - Cons: mediator can become monolithic, adds indirection.
- **Code Example** (Chat room):
```java
interface ChatMediator { void sendMessage(String msg, User user); }
class ChatRoom implements ChatMediator { ... }
abstract class User {
    protected ChatMediator mediator;
    public User(ChatMediator m) { this.mediator = m; }
    abstract void send(String msg);
}
```

#### Memento
- **Simple Explanation**: Without violating encapsulation, capture and externalize an object’s internal state so the object can be restored to this state later.
- **Why**: Implement undo mechanisms, savepoints.
- **What**: Originator creates a memento snapshot of its state; Caretaker stores it; later Originator restores from memento.
- **How**: Memento is an immutable object (or opaque to caretaker) storing necessary state. Originator has `saveToMemento()` and `restoreFromMemento()`.
- **When Not to Use**: If state is huge, storing mementos can be memory-expensive; consider Command pattern for undo.
- **Pros & Cons**:
  - Pros: preserves encapsulation, simplifies undo.
  - Cons: memory overhead, caretaker must manage memento lifecycle.
- **Code Example** (text editor undo):
```java
class Editor {
    private String text;
    public Memento save() { return new Memento(text); }
    public void restore(Memento m) { text = m.getText(); }
    static class Memento {
        private final String text;
        Memento(String text) { this.text = text; }
        String getText() { return text; }
    }
}
class History {
    private Stack<Editor.Memento> history = new Stack<>();
    // push, pop
}
```

#### Visitor
- **Simple Explanation**: Separate an algorithm from the object structure on which it operates, allowing adding new operations without modifying the element classes.
- **Why**: When you have a stable set of element types (file, directory) but many operations (size calculation, search, format) that you want to add without changing element code.
- **What**: Element interface accepts a `Visitor`. Concrete elements call `visitor.visit(this)`. Visitor interface has a visit method for each concrete element.
- **How**:
  1. Define `Visitor` interface with overloaded `visit(ElementX)` for each element type.
  2. Concrete visitor implements those methods.
  3. Elements implement `accept(Visitor v) { v.visit(this); }`.
- **When Not to Use**: When element hierarchy changes often (then visitors break), or when number of operations is small and stable.
- **Pros & Cons**:
  - Pros: easy to add new operations, keeps related operations together.
  - Cons: adding new element classes breaks all visitors, may expose element internals.
- **Code Example** (insurance clients):
```java
interface Visitor { void visit(Bank bank); void visit(Resident resident); }
interface Client { void accept(Visitor v); }
class Bank implements Client { public void accept(Visitor v) { v.visit(this); } }
class Resident implements Client { public void accept(Visitor v) { v.visit(this); } }
class InsuranceMailer implements Visitor { /* send different letters */ }
```

### 2.4 Enterprise Patterns (quick mention)
- **Repository**: Mediates between domain and data mapping layers, acting like an in-memory collection. Used extensively in Spring Data.
- **Unit of Work**: Keeps track of everything you do during a business transaction that can affect the database; typically JPA `EntityManager`.
- **CQRS**: Separate read and write models.
- **Event Sourcing**: Persist application state as a sequence of events.

## 3. Anti-Patterns & When Not to Pattern
- **Patternitis**: Applying patterns for their own sake, increasing complexity without solving real problems.
- **Golden Hammer**: Using the same pattern everywhere (e.g., Singleton for all dependencies).
- **God Object**: Overusing Facade/Mediator leading to a single class that knows too much.
- **Over-engineering**: When a problem is simple, a direct method call is better than Command + Factory + Strategy.
- **Signs to avoid patterns**: you don’t have a concrete variation scenario, the team is unfamiliar with the pattern, the problem is trivial.

## 4. How to Discuss Patterns at Staff Level
- **Use them as vocabulary**: “I decided to use a Strategy pattern here so we can plug in different pricing engines without touching the checkout flow.”  
- **Explain forces**: “We needed to add caching and logging to file access without modifying existing classes, so Decorator was a natural fit.”  
- **Show trade-off thinking**: “The Builder pattern made object construction readable but added verbosity. For a simple POJO, I’d stick with fluent setters.”
- **Connect to real incidents**: “We had a production bug because the Singleton cache wasn’t synchronized, causing race conditions. We fixed it with a `ConcurrentHashMap` and made it a proper enum singleton.”

## 5. Cheat Sheet
- Creational: Builder for complex objects; Factory for product families/object creation decoupling; DI for wiring.
- Structural: Adapter to integrate; Decorator to add behavior; Proxy to control access; Facade to simplify.
- Behavioral: Strategy to switch algorithms; Observer for event notification; Chain of Responsibility for pipelines; State for object state changes.
- Principle: Favor composition over inheritance; pattern when it reduces complexity.

---

### 🎤 Communication Training
- “I use design patterns as a tool, not a goal. When I see a switch statement growing, I think of Strategy or State. When I need to add cross-cutting concerns, Decorator comes to mind. I communicate decisions using pattern names to quickly convey architecture.”

### 📈 Personalised Self-Assessment
- **Strong**: likely you know classic patterns; you’ve used Observer, Strategy, maybe Builder.
- **Gaps**: deep understanding of less common patterns (Memento, Visitor, Bridge); recognizing over-engineering; translating pattern knowledge into architectural decisions.

Next topic when you say "next".
