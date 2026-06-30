# Core Java Interview Guide

## 1. Overview
At 12+ YOE the bar for Core Java is not “do you know the syntax”. The interview probes:
- **JVM internals** – how your code actually executes, memory model, GC, JIT  
- **Design with language features** – applying Java’s type system, concurrency primitives, and recent additions (records, sealed types, pattern matching, virtual threads) to write safe, performant, maintainable systems  
- **Trade-off thinking** – when to choose *this* collection, *this* concurrency construct, *this* synchronization approach under production load and constraints  
- **Hidden gaps** – you may be very comfortable with classic Java but might not have deep experience with modern additions (Project Loom, VarHandle, Panama) or low-level details (biased locking, non-strong references, invokedynamic)

Real systems: low-latency trading platforms, high-throughput data pipelines, service frameworks (Spring, Quarkus), messaging infrastructure, Kafka consumers/producers, microservice glue libraries – all are built on these fundamentals.

## 2. Deep Knowledge Structure

### 2.1 Language Fundamentals & OOP
#### The Type System
- **Concept** – how Java enforces contracts at compile-time and runtime  
- **Subtopics**: primitive vs. reference types, autoboxing, generics (type erasure, wildcards), covariant returns  
- **Microtopics (Deep Dive)**
  - Autoboxing caches (Integer -128..127), performance hazards in loops, hidden NPE risks
  - Type erasure implications: bridge methods, inability to overload with different generic parameters, heap pollution
  - `? extends` vs `? super` (PECS) – real API design, not just memorized rule
- **Atomic Topics (Interview-Level)**
  - *How does Java decide which overloaded method to invoke?* – compile-time resolution, widening, autoboxing, varargs priority
  - *What happens when you use `List<?>` and try to add an element?*
  - *Why does `someList.remove(1)` behave differently when the list is `List<Integer>` vs `List` of other object?*
- **Real-world Usage** – designing fluent APIs with bounded wildcards; preventing accidental mixing of raw types in legacy codebases
- **Failure Scenarios** – `ClassCastException` from mixing raw types and generics; performance regression from massive autoboxing in a hot loop
- **Tradeoffs** – type safety vs. code verbosity; invariant vs. wildcarded generics
- **Interview Expectations** – don’t just recite rules; explain how you’d evolve a library’s generic signatures without breaking clients
- **Common Mistakes** – using `List<Object>` where `List<?>` would suffice; assuming `List<String>` is a subtype of `List<Object>`

#### Object-Oriented Design
- **Concept** – inheritance, composition, encapsulation, polymorphism  
- **Microtopics**  
  - Method dispatch: `invokevirtual`, `invokespecial`, `invokestatic`, `invokeinterface` bytecodes – impact on inlining
  - Access modifiers: package-private’s role in testing vs. API safety  
  - `equals`/`hashCode` contract – symmetry, transitivity in subclass hierarchies; why `hashCode` must be consistent across JVM restarts only if documented (not guaranteed)
- **Atomic Topics**
  - *Why are fields recommended to be private?* – not just for encapsulation; JIT relies on it for optimisations (no escape from package, final field guarantees)
  - *When would you break the Liskov Substitution Principle?* – typical anti-patterns like throwing `UnsupportedOperationException` from overridden methods
- **Real-world Usage** – designing immutable value classes (java.time), using package-private classes to hide implementation
- **Failure Scenarios** – mutable object used as HashMap key breaks after mutation; subclass overriding `equals` asymmetrically
- **Tradeoffs** – inheritance for code reuse vs. fragility; composition + delegation vs. boilerplate
- **Common Mistakes** – forgetting to override `hashCode` when overriding `equals`; breaking the equals contract for subclasses; using inheritance just for reuse, not for is-a relationship

### 2.2 Collections Framework
#### Core Data Structures (List, Map, Set)
- **Concept** – complexity guarantees, internal structure, thread-safety  
- **Subtopics**: `ArrayList` vs `LinkedList`, `HashMap`, `TreeMap`, `LinkedHashMap`, `IdentityHashMap`, `EnumMap/Set`, `Queue` implementations
- **Microtopics (Deep Dive)**
  - `HashMap` internals: bucket array, collision resolution (linked list → tree), `TREEIFY_THRESHOLD`, `UNTREEIFY`, `hash()` perturbation function, load factor trade-offs (0.75 – why?), capacity (always power of two, why?), rehashing cost
  - `ConcurrentHashMap` evolution: Java 7 segments (lock striping) vs Java 8+ CAS + synchronized per bin head, size() method trade-offs, `computeIfAbsent` livelock risk
  - `ArrayList` vs `LinkedList`: not just “O(1) vs O(n)”, consider cache locality, memory overhead per node, GC impact
  - `TreeMap`/`TreeSet`: Red-Black tree, comparator consistency with equals
- **Atomic Topics**
  - *How does `HashMap` handle hashCode collisions?* → linked list, treeification at 8, why 8 (Poisson distribution), tree degrades back at 6
  - *What happens when you insert a million entries without specifying initial capacity?* – many rehashings, bandwidth waste, STW pauses
  - *`Collections.synchronizedMap()` vs `ConcurrentHashMap`* – not just thread-safety, but performance under contention, fail-fast iteration, compound operations
- **Real-world Usage** – cache implementations (LRU with `LinkedHashMap`), thread-safe LRU using `ConcurrentHashMap` + eviction logic, high-speed correlation engine with `IdentityHashMap`
- **Failure Scenarios** – `ConcurrentModificationException` when iterating and modifying via another thread; `HashMap` infinite loop during resize in pre-JDK8, multi-threaded access
- **Tradeoffs** – memory vs. speed (`HashMap` initial capacity, `ArrayList` pre-sizing), ordering vs. performance (`LinkedHashMap`), thread-safety vs. overhead
- **Interview Expectations** – Go beyond “I’d use `ArrayList` for random access”. Discuss memory footprint, GC pressure, performance in concurrent settings, alternative open-addressing implementations in native code
- **Common Mistakes** – using default capacity for large maps; mixing keys with inconsistent `equals`/`hashCode`; assuming `Vector` or `Hashtable` are still good choices

#### Utility & Specialized Collections
- `EnumSet`/`EnumMap` – bit vector internals, extremely fast, order matches enum declaration  
- `WeakHashMap` – weak references, GC-dependent behaviour, not a cache!  
- `PriorityQueue` – heap, not sorted; iterator order is unpredictable  
- `CopyOnWriteArrayList` – snapshot consistency, cost of writes, useful for listener lists  
- **Atomic** – *Why not use `WeakHashMap` for a cache?* – keys evicted on GC, no value eviction, no sizing control; `Caffeine`/`Guava Cache` needed

### 2.3 Concurrency & Multithreading
#### Core Primitives
- **Concept** – `synchronized`, `volatile`, `wait/notify`, `java.util.concurrent`  
- **Microtopics**
  - Java Memory Model (JMM): happens-before, volatile write-read guarantee, final field freeze
  - `synchronized` vs `ReentrantLock`: interruptibility, fairness, condition variables, tryLock
  - `volatile` – no atomicity, only visibility; compound actions (count++) not safe
- **Atomic Topics**
  - *Why does double-checked locking need `volatile`?* – prevent seeing partially constructed object (instruction reordering)
  - *When to use `Lock` over `synchronized`?* – need for timed/try-lock, multiple condition variables, or lock-breaking in deadlock detection
  - *What guarantees does a `final` field give in a constructor?* – JMM ensures correctly initialised reference to object with final fields (but not if `this` escapes)
- **Failure Scenarios** – lost updates, stale data, deadlocks, livelocks, thread starvation
- **Tradeoffs** – lock contention vs. lock-free, throughput vs. fairness
- **Common Mistakes** – using `volatile` for increment; waiting on a condition without loop; starting thread in constructor

#### High-Level Concurrency
- **Subtopics**: `ExecutorService`, `ForkJoinPool`, `CompletableFuture`, `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`
- **Microtopics**
  - `CompletableFuture` – async chaining, exception handling (`exceptionally`, `handle`, `whenComplete`), thread pool customisation, drawbacks (stack traces, debugging)
  - `ForkJoinPool` – work-stealing, common pool pitfalls (blocking tasks), `ManagedBlocker`
  - `ExecutorService` – bounded vs unbounded queues, rejection policies, thread naming, MDC context propagation
- **Atomic Topics**
  - *When not to use `CompletableFuture.supplyAsync(…)`?* – if using common FJP and tasks block, or need precise thread control
  - *How to cancel a task that is stuck in an I/O call?* – interrupt mechanism, but I/O may not respond; use `Future.cancel(true)` + own interruption logic
- **Real-world** – async service orchestration, parallel batch processing
- **Failure** – using `parallelStream()` on common FJP and blocking; thread pool tasks swallowing exceptions silently if not using `Future.get()`

#### Virtual Threads (Project Loom)
- **Concept** – lightweight user-mode threads, carrier threads, continuations  
- **Microtopics**
  - When to use: high-throughput blocking I/O tasks, not CPU-bound
  - Pinning: `synchronized` blocks or native calls pin carrier thread, scaling collapse
  - Structured concurrency (`StructuredTaskScope`)
- **Atomic** – *How to detect pinning?* – JFR events, logging `-Djdk.tracePinnedThreads=full`
- **Tradeoffs** – simplified code vs. hidden performance traps; not a silver bullet for all concurrent problems
- **Interview Expectations** – demonstrate real experience or deep reading; don’t claim production use if you only did a POC

### 2.4 JVM Internals & Memory Management
#### Garbage Collection
- **Subtopics**: Heap layout (young/old), collection algorithms (Serial, Parallel, CMS, G1, ZGC, Shenandoah)
- **Microtopics**
  - G1: regions, remembered sets, mixed collections, humongous allocations, `-XX:MaxGCPauseMillis`
  - ZGC: concurrent, coloured pointers, load barriers, <1ms pause goals, generational ZGC in recent versions
  - GC roots, safe points, stop-the-world phases
- **Atomic Topics**
  - *How does object promotion work?* – age threshold, survivor spaces, premature promotion
  - *What is a humongous object in G1?* – ≥ half region size, allocated directly in old gen, may cause fragmentation
  - *How to diagnose a memory leak?* – heap dumps, dominator tree, GC logs, `jstat`, allocation profiling
- **Failure** – OOM due to slow leak, long GC pauses, premature promotion causing frequent full GC
- **Tradeoffs** – throughput vs. latency vs. footprint; ZGC great for latency but may use more memory
- **Interview Expectations** – not recite flags, but reason about what collector to pick for a given workload and why

#### JIT Compilation & Optimisations
- **Concept** – C1 (client), C2 (server), tiered compilation, profiling in interpreter  
- **Microtopics**
  - Inlining, escape analysis, scalar replacement, dead code elimination, loop unrolling, lock coarsening
  - On-stack replacement (OSR)
  - Deoptimisation causes: changed class hierarchy, `synchronized` block on object with biased lock revocation
- **Atomic** – *Why might `synchronized` perform better than `ReentrantLock`?* – biased locking, lock elision performed by JIT
- **Real-world** – logging guards (`if (log.isDebugEnabled())`) might not be needed due to JIT optimising away constant folding, but be aware of string concatenation cost

#### Class Loading & Reflection
- **Subtopics**: class loaders (bootstrap, extension/platform, application), delegation model, `Class.forName` vs `ClassLoader.loadClass`
- **Microtopics**
  - Static initializers (`<clinit>`) thread-safe, executed once
  - Reflection overhead – `setAccessible` and `MethodHandle`/`VarHandle` alternatives
  - MetaSpace (since Java 8) vs PermGen
- **Atomic** – *What’s the difference between `ClassNotFoundException` and `NoClassDefFoundError`?* – former at dynamic load, latter when class found at compile time but missing at runtime (or static init failure)
- **Failure** – memory leak through reflection (cached `Method` objects holding references); classloader leaks (logging frameworks, redeploy)

### 2.5 Modern Java (8–21+) Features
#### Functional & Streams
- **Concept** – lambdas, method references, `Stream` API, `Optional`  
- **Atomic Questions**
  - *When not to use streams?* – performance-critical loops with simple mutable reduction; streams overhead, readability for very short operations
  - *What is the difference between `map` and `flatMap`?*
  - *How does `Optional` help, and where should it NOT be used?* – not for fields, not for method parameters; mostly for return types
- **Failure** – using `parallelStream` in environments with small data or blocking operations; using `peek` for side-effects (not guaranteed in all streams)
- **Tradeoffs** – readability vs. performance, eager vs. lazy evaluation

#### Records, Sealed Classes, Pattern Matching
- **Concept** – data carriers, closed type hierarchies, type testing  
- **Microtopics**
  - Records: `equals`, `hashCode`, `toString` generated, immutable, no subclass, `Component` annotation for serialization frameworks
  - Sealed classes: permits list, pattern matching exhaustiveness check
  - Pattern matching `switch` (Java 21)
- **Atomic** – *How does pattern matching improve code?* – type-safe destructuring, no `instanceof` + cast boilerplate; exhaustive checks prevent bugs
- **Real-world** – DTOs, protocol messages, algebraic data types for domain modelling
- **Interview Expectations** – Show you can apply these to make APIs safer and code more concise

### 2.6 I/O, NIO, and Networking
- **Concept** – synchronous vs asynchronous I/O, ByteBuffer, channels, selectors  
- **Microtopics**
  - Direct vs heap buffers – zero-copy, memory management
  - `FileChannel.transferTo`/`transferFrom` – OS-level efficiency
  - Epoll/kqueue under the hood
- **Atomic** – *When would you use NIO over traditional I/O?* – high concurrency servers, need non-blocking
- **Failure** – `OutOfMemoryError: Direct buffer memory` when pooling not used; `select()` waking up spurious; not handling partial read/write
- **Modern** – `java.net.http.HttpClient` (Java 11), reactive streams (Flow API)

### 2.7 Serialization
- **Concept** – Java serialization mechanics (damaging), alternatives (JSON, protobuf, Avro)  
- **Microtopics** – serialVersionUID, `readObject`/`writeObject`, `Externalizable`, security (deserialization gadgets)
- **Atomic** – *Why avoid Java native serialization?* – brittle, slow, huge attack surface, not cross-language
- **Real-world** – `FilteredObjectInputStream`, rejecting all custom classes by default

## 3. Code & Examples

```java
// HashMap capacity calculation example – avoid rehashing
int expectedSize = 1_000_000;
// default load factor 0.75 => capacity = expectedSize/0.75 + 1
int capacity = (int) (expectedSize / 0.75f) + 1;
Map<String, Object> map = new HashMap<>(capacity);

// ConcurrentHashMap computeIfAbsent – potential livelock
ConcurrentMap<String, List<String>> map = new ConcurrentHashMap<>();
// DO NOT modify the same map inside the function!
map.computeIfAbsent("key", k -> {
    List<String> v = new ArrayList<>();
    // map.putIfAbsent(k, v); // cause recursive update -> livelock
    return v;
});

// Volatile guarantee
class Config {
    private volatile boolean initialized = false;
    private Data data; // not volatile

    void init() {
        data = new Data(); // must happen-before initialized = true
        initialized = true;
    }

    Data getData() {
        if (initialized) { // volatile read
            return data; // safe due to happens-before
        }
        return null;
    }
}

// Modern switch with pattern matching (Java 21)
Object obj = ...;
String result = switch (obj) {
    case Integer i -> "Integer: " + i;
    case String s when !s.isEmpty() -> "Non-empty string";
    case null -> "null";
    default -> "something else";
};

// ThreadPool with MDC propagation for logging
ExecutorService executor = Executors.newFixedThreadPool(10, r -> {
    // capture current context
    Map<String, String> context = MDC.getCopyOfContextMap();
    Thread t = new Thread(() -> {
        if (context != null) MDC.setContextMap(context);
        r.run();
    });
    return t;
});
```

## 4. Interview Intelligence

### Topic: HashMap internals

#### ❌ Mid-level Answer
“HashMap uses array of buckets and linked list for collision. Java 8 improves it to balanced tree when collisions exceed 8.”

#### ✅ Senior Answer
“HashMap employs an array of Node objects, each a singly-linked list. Hash perturbation function XORs higher bits into lower to improve distribution because bucket index is computed as `(n-1) & hash`. At load factor 0.75 it resizes (doubles) and rehashes entries. From Java 8, when a bin has ≥ 8 nodes and total capacity ≥ 64, it transforms the list into a balanced tree (Red-Black) to bring worst-case get/put from O(n) to O(log n). Treeification uses `hashCode` ordering of Comparable keys, else uses tie-breaking by identity hash. I’ve tuned initial capacity to avoid resizing in hot maps, e.g., caches expecting 10k entries we set capacity = 10k/0.75+1 to prevent GC pauses from large rehash.”

#### 🚀 Staff-level Answer
“HashMap’s treeification threshold of 8 is based on Poisson distribution with λ=0.5, giving probability < 1 in 10 million for a healthy hash distribution. However, under pathological keys (e.g., deliberate hash collisions) treeification kicks in. Trade-off: tree nodes consume about 2x memory of regular nodes. So the `UNTREEIFY_THRESHOLD` of 6 prevents oscillation. The hash function itself `(h ^ (h>>>16))` helps compensate for poor hash codes that don’t vary in lower bits—critical because bucket index uses lower bits via mask. At scale, I consider not just CPU but memory footprint: each entry is ~32 bytes for Node plus key/value references. For large maps, we might choose open-addressing implementations in native memory (off-heap) to reduce GC pressure and improve cache locality, but lose standard API. In concurrent scenarios, we replace with `ConcurrentHashMap` which uses CAS for insertion and synchronized block only when a bin is modified. I’ve identified a production issue where `HashMap` resize under heavy concurrent read caused infinite loops in JDK 7; migration to `ConcurrentHashMap` solved it and we later adopted JDK 8’s fix.”

### Topic: Garbage Collection Tuning

#### ❌ Mid-level Answer
“Use G1GC for low pause. Set heap size and let it run.”

#### ✅ Senior Answer
“G1 divides heap into regions (~1-32MB) and aims to meet `MaxGCPauseMillis`. It collects young regions only in young GC; mixed GC includes some old regions chosen by liveness. I monitor GC logs (gc+heap=trace) for humongous allocations (≥50% region size) that cause premature old gen promotion and may trigger full GC. Tuned region size with `-XX:G1HeapRegionSize` to match our 32MB allocation pattern, reduced fragmentation. Also set `-XX:InitiatingHeapOccupancyPercent` to 45% to start concurrent marking earlier, preventing evacuation failures.”

#### 🚀 Staff-level Answer
“In a high-throughput, latency-sensitive payment system (SLA 99.99% <50ms), G1’s mixed collections caused occasional spikes >100ms due to remembered set scanning and evacuation of object graphs. We evaluated ZGC (later generational ZGC), which uses coloured pointers and load barriers, achieving sub-1ms pauses. However, ZGC required more headroom (~10-15% more heap) and worked with page sizes via `-XX:ZAllocationSpikeTolerance`. Migration involved validating that all native libraries and thread local handshakes (JVM TI) didn’t pin with virtual threads. We used JFR to measure allocation rate and found that switching to ZGC reduced tail latency by 80% but increased CPU usage by 5% due to concurrent marking. We justified the trade-off because throughput headroom existed and latency was the business KPI. The key was also shrinking object lifetimes to reduce concurrent work, using object pools for buffers, and avoiding finalization. I’d also consider Shenandoah if we had lower core counts.”

## 5. High ROI Questions
- How does HashMap work? What changed in Java 8?
- Explain ConcurrentHashMap internals and compare with `synchronizedMap`.
- What’s the difference between `synchronized` and a `Lock`? Give a scenario where `Lock` is essential.
- Describe the Java Memory Model and the volatile guarantee. Why is double-checked locking broken without volatile?
- How does garbage collection work in Java? Compare G1, ZGC, and choose for a specific workload.
- What are `CompletableFuture` pitfalls? How do you handle exceptions and threads properly?
- What are virtual threads? When should you NOT use them? How to detect pinning?
- Stream API vs traditional loops: when to avoid streams?
- Explain reflection and its performance/security implications. What are method handles?
- How would you design an immutable class? What about serialization?
- What are records and sealed classes, and how do they improve domain modelling?
- Why avoid Java native serialization? What alternatives do you use?
- `ClassNotFoundException` vs `NoClassDefFoundError`? Real scenario causing production crash.

**Tricky Follow-ups:**
- “Your `ConcurrentHashMap.computeIfAbsent` hangs – diagnose.”
- “A `HashMap<String, Integer>` starts returning null for a key you know exists. What could cause it?”
- “You increased `-Xmx` and pause times got worse. Why?”
- “Explain how `String`’s `hashCode` can be used maliciously to DoS a server.”

## 6. Cheat Sheet (Revision)
- **HashMap**: array of nodes; `hash = h ^ (>>>16)`; index = `(n-1) & hash`; treeify ≥8 & cap≥64; load factor 0.75; rehash doubles capacity.
- **CHM**: CAS on empty bin, synchronized on head – not whole map; size() doesn’t lock; `computeIfAbsent` cannot modify map.
- **volatile**: visibility + happens-before; no atomicity.
- **happens-before** key rules: unlock → lock, volatile write → read, thread start → thread’s first action, final field freeze after constructor.
- **GC order for service**: G1 for balanced, ZGC/Shenandoah for ultra-low pause, Parallel for throughput batch; know humongous allocation >50% region size.
- **Virtual threads**: incompatible with `synchronized` that pins, use `ReentrantLock`; monitor with JFR.
- **Streams**: lazy, no side-effects inside intermediate ops; avoid `parallel()` with blocking.
- **Records**: auto-generate constructor, accessors, equals, hashCode, toString; immutable; no extends.
- **Pattern matching switch**: exhaustive, supports guards.
- **Serialization**: avoid `ObjectInputStream` from untrusted sources; use `FilteredObjectInputStream` with blocklist.
- **Thread pools**: always bounded queue + rejection policy; never `Executors.newCachedThreadPool()` for production.

---

### 🎤 Communication Training for Core Java
- **Structure**: When asked a deep technical question, start with a brief high-level description, then dive into internals, then connect to real-world impact, finally note trade-offs.
- **Think like a senior engineer**: Frame your answer in terms of system qualities – performance, maintainability, security, failure modes. For example, “We chose `ConcurrentHashMap` over `Collections.synchronizedMap` not just for thread safety, but because under read-heavy load the lock-free reads reduced contention and prevented an outage where a monitoring thread blocked user requests.”
- **Show progressive reasoning**: “An alternative could be X, but it introduces Y problem; we validated with a benchmark simulating 50K concurrent users…”

### 📈 Personalised Self-Assessment (for a 12 YOE Staff+ Candidate)
- **Strong areas** (likely): Classic collections, OOP design, basic concurrency (`ExecutorService`, `synchronized`), Spring ecosystem, maybe some GC tuning.
- **Weak areas / Hidden gaps**:  
  - **Virtual threads** – hands-on experience vs. theory; be honest if not yet in production.  
  - **VarHandle/MethodHandle** – might not have used them directly; know their role in modern concurrent data structures.  
  - **GC deep mechanics** – card tables, remembered sets, SATB vs. snapshot-at-the-beginning for G1 vs. Shenandoah.  
  - **Class data sharing (CDS)/AppCDS** – reducing startup time; may be overlooked.
- **Overconfidence risks**:  
  - Assuming you know everything about `HashMap` – be ready for collision treeification threshold derivation.  
  - Thinking you’ve “done concurrency” – can you explain why `LongAdder` is better than `AtomicLong` under high contention, and when to use it?  
  - Dismissing security – Java deserialization gadgets are an entire attack surface; be ready to discuss mitigation beyond “don’t use Java serialization”.

Ready to move to the next topic on your command.
