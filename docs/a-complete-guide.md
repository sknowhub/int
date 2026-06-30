# STAFF FULLSTACK ENGINEER INTERVIEW HANDBOOK

## 12 YOE | Backend-Heavy | Java Microservices | Cloud-Native

---

# 📖 TABLE OF CONTENTS

1. [Core Java & JVM Deep Dive](#1-core-java--jvm-deep-dive)
2. [Spring Ecosystem Mastery](#2-spring-ecosystem-mastery)
3. [Databases & Transactions](#3-databases--transactions)
4. [Microservices Architecture](#4-microservices-architecture)
5. [Messaging & Event-Driven Systems](#5-messaging--event-driven-systems)
6. [Cloud Platforms (AWS/Azure)](#6-cloud-platforms-awsazure)
7. [DevOps & Kubernetes](#7-devops--kubernetes)
8. [System Design (CRITICAL)](#8-system-design-critical)
9. [DSA for Staff Engineers](#9-dsa-for-staff-engineers)
10. [Frontend (React for Backend Engineers)](#10-frontend-react-for-backend-engineers)
11. [API Design & Protocols](#11-api-design--protocols)
12. [Testing Pyramid & Strategies](#12-testing-pyramid--strategies)
13. [Security Deep Dive](#13-security-deep-dive)
14. [Design Patterns & Architecture](#14-design-patterns--architecture)
15. [System Engineering](#15-system-engineering)
16. [Tools & Process Excellence](#16-tools--process-excellence)
17. [App Servers (Tomcat/WAS)](#17-app-servers-tomcatwas)
18. [Reporting (Jasper) & Rules (Drools)](#18-reporting-jasper--rules-drools)
19. [Behavioral & Leadership](#19-behavioral--leadership)
20. [Mock Interview Simulator](#20-mock-interview-simulator)
21. [Personalized Analysis & Study Plan](#21-personalized-analysis--study-plan)

---

# 1. CORE JAVA & JVM DEEP DIVE

## 📚 JVM Memory Management & Garbage Collection

### 🎯 Topic: Garbage Collection Algorithms

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**G1 GC (Garbage First)**
- **Region sizing**: 1MB-32MB, typically 2048 regions, dynamically resized
- **SATB (Snapshot-At-The-Beginning)**: Concurrent marking, preserves objects reachable at start
- **Remembered Sets**: Track cross-region references, 5-15% heap overhead
- **Young collections**: Eden → Survivor → Old promotion with evacuation pauses
- **Mixed collections**: Collect old regions with most garbage first (`-XX:G1MixedGCLiveThresholdPercent=85`)
- **Tuning flags**: `-XX:MaxGCPauseMillis=200`, `-XX:G1HeapRegionSize=16m`

**ZGC (Z Garbage Collector)**
- **Colored pointers**: 64-bit pointers with metadata in high bits (finalizable, remapped, marked0/1)
- **Load barriers**: Every pointer read checks and fixes colored bits
- **Concurrent**: All phases concurrent except initial mark, final mark
- **Sub-millisecond pauses**: Pause times <1ms even for multi-TB heaps
- **Limitations**: Class unloading requires STW, heap size must be multiple of 2MB

**GC Comparison Matrix**

| GC | Pause Target | Heap Overhead | Throughput | Best For |
|----|--------------|---------------|------------|----------|
| G1 | 50-200ms | 10-15% | Medium | Balanced (default) |
| ZGC | <1ms | 15-20% | Low | Latency-sensitive |
| Shenandoah | <10ms | 15-20% | Medium | Ultra-low latency |
| Parallel | >1s | 5% | High | Batch jobs, offline |

#### Real-World Usage

**Production Example: Trading System with ZGC**
```bash
java -XX:+UseZGC \
     -XX:ConcGCThreads=4 \
     -XX:ParallelGCThreads=8 \
     -XX:ZCollectionInterval=120 \
     -Xmx32g \
     -jar trading-engine.jar
```
**Result**: P99 GC pause 0.8ms, heap 32GB, 100k transactions/sec

**Failure Scenario: Humongous Allocation in G1**
```java
// Problem: Allocating object >50% of region size
byte[] hugeBuffer = new byte[50 * 1024 * 1024]; // 50MB
```
- G1 treats as humongous object
- Directly allocated in old gen, not reclaimed until full GC
- Causes full GC pauses
- **Fix**: Increase region size or reduce allocation size

#### Tradeoffs Deep Dive

**G1 vs ZGC Decision Framework**
| Requirement | G1 | ZGC |
|-------------|----|----|
| Pause time < 10ms | ❌ | ✅ |
| Heap > 64GB | ✅ | ⚠️ (requires more memory) |
| Low CPU overhead | ✅ | ❌ (load barriers ~5% CPU) |
| Predictable performance | ✅ | ⚠️ (colored pointer overhead) |

---

## 🔄 Concurrency & Memory Model

### 🎯 Topic: Java Memory Model (JMM)

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Happens-Before Rules (JSR-133)**
1. **Program order**: Each action in thread happens-before subsequent actions
2. **Monitor lock**: Unlock happens-before subsequent lock on same monitor
3. **Volatile variable**: Write happens-before subsequent read
4. **Thread start**: Thread.start() happens-before actions in new thread
5. **Thread join**: Actions in thread happen-before Thread.join() returns
6. **Transitivity**: If A hb B and B hb C, then A hb C

**Memory Barriers (JVM Implementation)**
- **LoadLoad**: Prevent reordering of reads
- **StoreStore**: Prevent reordering of writes
- **LoadStore**: Prevent read after write reordering
- **StoreLoad**: Most expensive, prevents all reordering

**Volatile Semantics**
```java
// Without volatile - may see stale values
private boolean running = true;

// With volatile - guarantees visibility + prevents reordering
private volatile boolean running = true;

// Real usage: shutdown flag
public void shutdown() {
    running = false;  // volatile write
}
```

**Safe Publication Patterns**
```java
// ❌ Unsafe publication
public Holder holder;  // not volatile
public void initialize() {
    holder = new Holder(42);  // may see partially constructed
}

// ✅ Safe publication #1: volatile
private volatile Holder holder;

// ✅ Safe publication #2: final fields
public class Holder {
    private final int value;  // guaranteed visibility
    public Holder(int v) { value = v; }
}

// ✅ Safe publication #3: static initializer
private static final Holder holder = new Holder(42);
```

#### Real-World Failure: DCL Broken Before Java 5
```java
// Double-checked locking - BROKEN in Java 1.4
class Singleton {
    private static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {          // 1: first check
            synchronized (Singleton.class) {
                if (instance == null) {    // 2: second check
                    instance = new Singleton();  // 3: problem here
                }
            }
        }
        return instance;
    }
}
```
**Why broken?** `new Singleton()` is not atomic:
1. Allocate memory
2. Initialize object
3. Assign reference to instance

Reordering could make (3) before (2), exposing partially initialized object.

**Fix in Java 5+** : Use volatile (fixed JMM)

---

## 🧵 Virtual Threads (Project Loom)

### 🎯 Topic: Virtual Threads Deep Dive

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Core Concepts**
- **Carrier thread**: Platform thread that executes virtual thread (pool: ForkJoinPool, size = CPU cores)
- **Mount/Unmount**: Virtual thread mounted to carrier for execution, unmounted on blocking I/O
- **Continuation**: Captures execution state, allows yield/resume
- **Parking**: Virtual thread parks when blocking, carrier unmounts and runs another

**Implementation Details**
```java
// JVM implementation (simplified)
class VirtualThread {
    private Continuation continuation;
    private CarrierThread carrier;
    
    void mount(CarrierThread carrier) {
        this.carrier = carrier;
        carrier.setCurrent(this);
        continuation.run();  // start/continue execution
    }
    
    void unmount() {
        continuation.yield();  // suspend
        carrier.setCurrent(null);
        this.carrier = null;
    }
}
```

**Pinning Scenarios**
```java
// ❌ Pinned: synchronized blocks during I/O
synchronized(lock) {
    Thread.sleep(1000);  // Carrier thread blocked!
}

// ✅ Not pinned: ReentrantLock
lock.lock();
try {
    Thread.sleep(1000);  // Unmounts correctly
} finally {
    lock.unlock();
}

// ❌ Pinned: native frames or JNI
public native void blockingOperation();  // Cannot unmount
```

**Detection**: `-Djdk.tracePinnedThreads=full` shows pinning in logs

#### Real-World Migration: Blocking IO Service

**Before (Platform Threads)**
```java
@RestController
public class ProductController {
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        // Each request consumes a platform thread (1MB stack)
        Product p = productRepo.findById(id);  // JDBC blocking
        return enrichWithInventory(p);  // Another blocking call
    }
}
```
**Limitation**: Max 10k concurrent requests (thread pool limit)

**After (Virtual Threads)**
```java
@RestController
public class ProductController {
    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable Long id) {
        // Uses virtual threads - millions concurrent
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<Product> productFuture = executor.submit(() -> 
                productRepo.findById(id)
            );
            Future<Inventory> inventoryFuture = executor.submit(() -> 
                inventoryService.getInventory(id)
            );
            return enrich(productFuture.get(), inventoryFuture.get());
        }
    }
}
```

**Spring Boot 3.2 Configuration**
```yaml
spring:
  threads:
    virtual:
      enabled: true
  tomcat:
    threads:
      max: 4  # Platform threads kept low for carrier
```

#### Tradeoffs: Virtual Threads vs Reactive

| Aspect | Virtual Threads | WebFlux (Reactive) |
|--------|----------------|---------------------|
| **Learning curve** | Low (sequential code) | High (monads, operators) |
| **Debugging** | Stack traces readable | Stack traces cryptic |
| **Context propagation** | ThreadLocal works | Requires Reactor Context |
| **Blocking libraries** | Works (with pinning detection) | Breaks (event loop starvation) |
| **Memory per request** | ~1KB | ~200 bytes |
| **Backpressure** | Not native (use semaphores) | Built-in (request(n)) |
| **CPU overhead** | Lower (~5-10%) | Higher (operator chains, allocation) |

**Decision Framework**
```
Use Virtual Threads if:
✅ Existing blocking code (JDBC, HTTP clients)
✅ Team lacks reactive experience
✅ Need per-request ThreadLocal (tenant, correlation ID)
✅ Mixed CPU + I/O workloads

Use WebFlux if:
✅ High throughput with backpressure (streaming)
✅ Existing reactive ecosystem (R2DBC, Netty)
✅ Memory-constrained environment
✅ Need fine-grained control over concurrency
```

---

## 📊 Concurrent Collections Deep Dive

### 🎯 Topic: ConcurrentHashMap Internals

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Java 8+ Implementation**
- **Bin-level locking**: Each hash bin has its own lock (synchronized on head node)
- **Tree bin threshold**: 8 entries → converts to TreeNode (balanced tree)
- **size() problem**: 
  ```java
  // size() is O(n) - counts bins!
  // Use mappingCount() for large maps (>2^31 entries)
  long count = map.mappingCount();  // returns long
  ```
- **computeIfAbsent**:
  ```java
  // Atomic but may compute multiple times if racing (fixed in Java 8)
  map.computeIfAbsent(key, k -> expensiveComputation());
  // Still okay - only one result stored
  ```

**Common Failure: Recursive Update**
```java
// ❌ Deadlock!
map.computeIfAbsent(key, k -> {
    map.put(key2, value2);  // Modifying while computing
    return computeValue();
});

// ✅ Use compute() instead
map.compute(key, (k, v) -> {
    map.put(key2, value2);  // Still dangerous, but allowed
    return newValue;
});
```

**Performance Characteristics**
| Operation | ConcurrentHashMap | Synchronized Map | Hashtable |
|-----------|-------------------|------------------|-----------|
| get() | O(1) lock-free | O(1) (lock whole map) | O(1) (lock whole map) |
| put() | O(1) (bin lock) | O(1) (lock whole map) | O(1) (lock whole map) |
| size() | O(n) | O(1) (with lock) | O(1) (with lock) |
| Concurrency | 16 default segments | 1 (full map lock) | 1 (full map lock) |

**Real-World Pattern: LRU Cache with ConcurrentHashMap**
```java
public class ConcurrentLRUCache<K,V> {
    private final ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
    private final ConcurrentLinkedDeque<K> queue = new ConcurrentLinkedDeque<>();
    
    public V get(K key) {
        V value = map.get(key);
        if (value != null) {
            queue.remove(key);  // O(n) - OK for small queues
            queue.addFirst(key);
        }
        return value;
    }
    
    public void put(K key, V value) {
        if (map.size() >= maxSize) {
            K eldest = queue.removeLast();
            map.remove(eldest);
        }
        map.put(key, value);
        queue.addFirst(key);
    }
}
```
**Note**: Not perfectly atomic but good enough for many caches.

---

# 2. SPRING ECOSYSTEM MASTERY

## 📦 Spring IoC Container & Bean Lifecycle

### 🎯 Topic: Bean Lifecycle Management

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Complete Lifecycle (8 Phases)**
```
1. Instantiate (constructor)
   ↓
2. Populate properties (setters, @Autowired)
   ↓
3. BeanNameAware.setBeanName()
   ↓
4. BeanFactoryAware.setBeanFactory()
   ↓
5. ApplicationContextAware.setApplicationContext()
   ↓
6. BeanPostProcessor.postProcessBeforeInitialization()
   ↓
7. @PostConstruct or InitializingBean.afterPropertiesSet()
   ↓
8. @PreDestroy (on shutdown)
```

**BeanPostProcessor vs BeanFactoryPostProcessor**
| Aspect | BeanPostProcessor | BeanFactoryPostProcessor |
|--------|-------------------|--------------------------|
| **When** | After bean instantiation | Before bean instantiation |
| **What** | Modify bean instances | Modify bean definitions |
| **Example** | @Autowired processing | Property placeholder resolution |
| **Use case** | Proxy wrapping, caching | Environment property override |

**Circular Dependency Resolution**
```java
// ✅ Constructor injection (Spring can resolve cycle)
@Component
public class A {
    private final B b;
    public A(B b) { this.b = b; }
}

@Component 
public class B {
    private final A a;
    public B(A a) { this.a = a; }
}
// Works - Spring uses 3-level cache (singletonFactories, earlySingletonObjects, singletonObjects)

// ❌ Field injection with @Autowired (still works but harder to test)
// ❌ Prototype bean with constructor injection (cannot resolve)
```

**Proxy Mode Differences**
```java
// JDK Dynamic Proxy (default for interfaces)
@Service
public class UserService implements ServiceInterface {
    // Works - proxy implements same interface
}

// CGLIB Proxy (for classes without interfaces)
@Service
@EnableAsync
public class UserService {  // No interface
    // Requires CGLIB, final methods not proxied
    public final void save() { }  // ❌ Won't be intercepted
}
```

#### Real-World Usage: Custom BeanPostProcessor

```java
@Component
public class AuditAnnotationProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Method[] methods = bean.getClass().getMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(Auditable.class)) {
                // Create proxy with audit logging
                return Proxy.newProxyInstance(
                    bean.getClass().getClassLoader(),
                    bean.getClass().getInterfaces(),
                    (proxy, m, args) -> {
                        log.info("Calling " + m.getName());
                        return m.invoke(bean, args);
                    }
                );
            }
        }
        return bean;
    }
}
```

---

## 🌊 Spring WebFlux vs Virtual Threads

### 🎯 Topic: Reactive vs Virtual Thread Comparison

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**WebFlux Core Concepts**
- **Event Loop**: Netty with 2*CPU cores threads (default)
- **Backpressure**: Request(n) protocol from subscriber to publisher
- **Operator Fusion**: 
  - **Macro fusion**: Merge operators into single subscriber (e.g., map().filter())
  - **Micro fusion**: Optimize internal data flow (e.g., Flux.range(1,100).map())
- **Threading Model**:
  ```java
  Flux.just(1,2,3)
      .map(i -> i*2)           // Runs on original thread
      .publishOn(Schedulers.parallel())  // Switch to parallel
      .subscribe();
  ```

**Reactor Context Propagation**
```java
// ThreadLocal doesn't work - use Context
@GetMapping
public Mono<String> handle() {
    return Mono.deferContextual(ctx -> 
        Mono.just("Hello " + ctx.get("tenantId"))
    ).contextWrite(Context.of("tenantId", "acme123"));
}
```

**Virtual Threads in Spring Boot 3.2**
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
  task:
    execution:
      pool:
        virtual-threads: true
```

#### Production Performance Comparison

**Load Test: 10k concurrent requests, 100ms external call**

| Metric | WebFlux | Virtual Threads | Traditional (Tomcat) |
|--------|---------|----------------|---------------------|
| **Throughput (RPS)** | 28,000 | 25,000 | 18,000 |
| **P99 Latency** | 45ms | 52ms | 180ms |
| **Memory (heap)** | 850MB | 1.2GB | 2.4GB |
| **GC pause (avg)** | 12ms | 18ms | 45ms |
| **CPU utilization** | 65% | 72% | 45% |
| **Thread count peak** | 16 | 10,000+ | 200 |

**Real Migration Case: API Gateway**

**Before (WebFlux)**
```java
@Component
public class RoutingHandler {
    // Complex context propagation
    @Autowired
    private AuthService authService;
    
    public Mono<Response> route(Request req) {
        return Mono.deferContextual(ctx -> 
            authService.validate(ctx.get("token"))
                .flatMap(valid -> buildResponse(req))
        ).contextWrite(ctx -> ctx.put("token", req.token()));
    }
}
```
**Problem**: Context loss across async boundaries, debugging hell

**After (Virtual Threads)**
```java
@RestController
public class RoutingController {
    @Autowired
    private AuthService authService;
    
    @GetMapping("/**")
    public Response route(Request req) {
        // Simple blocked code
        boolean valid = authService.validate(req.token());  // Blocking but fine
        return buildResponse(req);  // Sequential code
    }
}
```
**Result**: 30% reduction in code, P99 latency unchanged, easier maintenance

---

## 🔐 Spring Security Deep Dive

### 🎯 Topic: OAuth2 Resource Server

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**JWT Validation Flow**
```
1. Parse Authorization: Bearer <jwt>
2. Validate signature (using public key from jwks endpoint)
3. Check expiration (iat, exp claims)
4. Validate audience (aud) matches service
5. Extract authorities (roles, scopes)
6. Build Authentication object
```

**JWT Decoding Configuration**
```java
@Bean
public ReactiveJwtDecoder jwtDecoder() {
    return NimbusReactiveJwtDecoder.withJwkSetUri("https://auth.server/.well-known/jwks.json")
        .jwtProcessorCustomizer(processor -> {
            processor.setJWTClaimsSetAware(true);
        })
        .build();
}

@Bean
public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
    return http
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt
                .jwtAuthenticationConverter(grantedAuthoritiesExtractor())
            )
        )
        .authorizeExchange(exchanges -> exchanges
            .pathMatchers("/public/**").permitAll()
            .pathMatchers("/admin/**").hasRole("ADMIN")
            .anyExchange().authenticated()
        )
        .build();
}
```

**Token Relay (Service-to-Service)**
```java
@Service
public class DownstreamService {
    @Bean
    public WebClient webClient(ReactiveClientRegistrationRepository clientRegistrations) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(clientRegistrations);
        oauth2.setDefaultClientRegistrationId("downstream");
        return WebClient.builder()
            .apply(oauth2.oauth2Configuration())
            .build();
    }
}
```

**JWT Revocation Strategies**

| Strategy | How it works | Pros | Cons |
|----------|--------------|------|------|
| **Short TTL** | Token expires in 5 minutes | Simple, no state | Frequent refresh |
| **Allowlist** | Store valid tokens in Redis | Immediate revocation | Storage cost, latency |
| **Blacklist** | Only revoked tokens stored | Efficient | Need check on every request |
| **Passive** | Introspection endpoint call | Central control | Network overhead |

**Real-World Token Blacklisting**
```java
@Component
public class RevokedTokenFilter extends OncePerRequestFilter {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain chain) {
        String jwt = extractJwt(request);
        if (redis.hasKey("blacklist:" + jwt)) {
            response.setStatus(HttpStatus.UNAUTHORIZED);
            return;
        }
        chain.doFilter(request, response);
    }
}

// On logout
public void revokeToken(String jwt, Duration ttl) {
    redis.opsForValue().set("blacklist:" + jwt, "revoked", ttl);
}
```

---

# 3. DATABASES & TRANSACTIONS

## 🔍 SQL Window Functions Deep Dive

### 🎯 Topic: Advanced Window Functions

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Frame Specification**
```sql
-- ROWS: Physical rows (simpler, less overhead)
SUM(amount) OVER (
    ORDER BY created_at
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)

-- RANGE: Logical range based on value (slower but semantic)
SUM(amount) OVER (
    ORDER BY created_at
    RANGE BETWEEN INTERVAL '1' DAY PRECEDING AND CURRENT ROW
)

-- GROUPS: Groups of equal values (new in SQL:2016)
AVG(score) OVER (
    ORDER BY score
    GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
)
```

**Performance: Window Functions vs Self-Join**

| Operation | Window Function | Self-Join |
|-----------|----------------|-----------|
| **Running total** | ✅ O(n) one pass | ❌ O(n²) quadratic |
| **Row number** | ✅ O(n) | ❌ O(n²) |
| **Moving average** | ✅ O(n) with frame | ⚠️ O(n log n) |
| **Previous row (LAG)** | ✅ O(n) | ⚠️ O(n log n) |

**Real-World Examples**

**1. Pagination without OFFSET (Keyset Pagination)**
```sql
-- ❌ OFFSET (O(n) scan each time)
SELECT * FROM users ORDER BY id OFFSET 1000 LIMIT 10;

-- ✅ Keyset with window (fast)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY id) as rn
    FROM users
) t WHERE rn BETWEEN 1000 AND 1010;
-- But better yet: use explicit WHERE id > last_id
```

**2. Deduplication with ROW_NUMBER**
```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id, event_type ORDER BY created_at DESC) as rn
    FROM user_events
)
SELECT * FROM ranked WHERE rn = 1;  -- Latest event per user per type
```

**3. Gap Analysis**
```sql
-- Find missing IDs
WITH gaps AS (
    SELECT 
        id,
        LAG(id) OVER (ORDER BY id) as prev_id
    FROM sequences
)
SELECT prev_id + 1 as gap_start, 
       id - 1 as gap_end
FROM gaps 
WHERE id - prev_id > 1;
```

**4. Time Series: Year-over-Year Comparison**
```sql
SELECT 
    sale_date,
    amount,
    LAG(amount, 365) OVER (ORDER BY sale_date) as amount_last_year,
    (amount - LAG(amount, 365) OVER (ORDER BY sale_date)) / 
        LAG(amount, 365) OVER (ORDER BY sale_date) * 100 as yoy_growth
FROM daily_sales;
```

---

## 🔄 Distributed Transactions

### 🎯 Topic: 2PC vs SAGA vs Outbox

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**2PC (Two-Phase Commit) Flow**
```
Coordinator          Resource1         Resource2
    |                    |                 |
    |---- PREPARE ------>|                 |
    |---- PREPARE ----------------------->|
    |<----- VOTE (YES)---|                 |
    |<----- VOTE (YES)---------------------|
    |                    |                 |
    |---- COMMIT ------->|                 |
    |---- COMMIT ------------------------->|
    |<----- DONE---------|                 |
    |<----- DONE---------------------------|
```

**Failure Scenarios in 2PC**
1. **Coordinator fails after prepare** → Resources locked indefinitely (need recovery)
2. **Resource fails during commit** → Inconsistent state, manual intervention
3. **Network partition** → Blocked until timeout, then uncertain state

**SAGA Orchestration Pattern**
```java
@SagaOrchestration
public class OrderSaga {
    
    @Step(execute = "createOrder", compensate = "cancelOrder")
    public void createOrder(OrderRequest req) {
        orderService.create(req);
        sagaState.setOrderId(order.getId());
    }
    
    @Step(execute = "reserveInventory", compensate = "releaseInventory")
    public void reserveInventory(OrderRequest req) {
        inventoryService.reserve(req.productId, req.quantity);
        sagaState.setInventoryReserved(true);
    }
    
    @Step(execute = "processPayment", compensate = "refundPayment")
    public void processPayment(OrderRequest req) {
        paymentService.charge(req.userId, req.amount);
        sagaState.setPaymentProcessed(true);
    }
    
    @Compensation
    public void compensate() {
        // Automatically calls compensate methods in reverse order
    }
}
```

**SAGA Choreography with Kafka**
```java
// Order Service
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) {
    kafkaTemplate.send("inventory-reserve", event);
}

// Inventory Service
@KafkaListener(topics = "inventory-reserve")
public void reserveInventory(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event);
        kafkaTemplate.send("payment-process", event);
    } catch (Exception e) {
        kafkaTemplate.send("order-compensate", event);
    }
}
```

**Transaction Outbox Pattern**
```java
@Transactional
public void createOrder(Order order) {
    // 1. Save order
    orderRepository.save(order);
    
    // 2. Save outbox record (same transaction)
    OutboxEvent event = OutboxEvent.builder()
        .aggregateId(order.getId())
        .eventType("ORDER_CREATED")
        .payload(json(order))
        .createdAt(Instant.now())
        .build();
    outboxRepository.save(event);
    
    // Transaction commits atomically
}

// Poller (Debezium CDC or scheduled task)
@Scheduled(fixedDelay = 5000)
public void publishEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished();
    for (OutboxEvent event : events) {
        try {
            kafkaTemplate.send(event.getEventType(), event.getPayload());
            event.setPublished(true);
            outboxRepository.save(event);
        } catch (Exception e) {
            // Retry on next poll
            log.error("Failed to publish event", e);
        }
    }
}
```

**Comparison Matrix**

| Pattern | Consistency | Latency | Complexity | Failure Handling | Use Case |
|---------|-------------|---------|------------|------------------|----------|
| **2PC** | Strong | High (2-3 RTT) | High | Blocking, recovery | Financial transactions, short duration |
| **SAGA** | Eventual | Medium (async) | High (compensation) | Compensating actions | Long-running business processes |
| **Outbox** | Eventual | Low (CDC) | Medium | Retry + idempotency | Event-driven microservices |
| **Idempotency Keys** | Strong | Low | Low | Application-level retry | Simple cross-service calls |

**Real-World: Payment Processing with Outbox**
```sql
-- Idempotency key + outbox in same transaction
BEGIN;
INSERT INTO payments (idempotency_key, amount, status) 
VALUES ('key_123', 99.99, 'PROCESSING');

INSERT INTO outbox (idempotency_key, event_type, payload) 
VALUES ('key_123', 'PAYMENT_INITIATED', '{"amount":99.99}');

COMMIT;

-- Idempotent consumer
@KafkaListener
public void handlePaymentEvent(Event event) {
    // Atomic insert with duplicate key ignore
    boolean inserted = eventProcessedRepository.insertIfNotExists(event.getId());
    if (!inserted) return;  // Already processed
    processEvent(event);
}
```

---

## 📈 Database Scaling Strategies

### 🎯 Topic: Sharding Deep Dive

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Sharding Strategies**

**1. Hash Sharding**
```java
// Determine shard by hashing shard key
int calculateShard(String userId) {
    int hash = userId.hashCode();
    return Math.abs(hash) % numberOfShards;
}
```
**Pros**: Even distribution, good for random access
**Cons**: Resharding expensive, range queries across shards impossible

**2. Range Sharding**
```sql
-- Shard 1: user_id 1-1,000,000
-- Shard 2: user_id 1,000,001-2,000,000
-- Shard 3: user_id 2,000,001-3,000,000
```
**Pros**: Efficient range queries, easy resharding (splitting ranges)
**Cons**: Hot spots for active ranges (new users writing to latest shard)

**3. Directory-Based Sharding**
```sql
CREATE TABLE shard_map (
    entity_id VARCHAR(100),
    shard_id INT,
    PRIMARY KEY (entity_id)
);
```
**Pros**: Flexible, easy to move entities between shards
**Cons**: Extra lookup overhead, directory becomes bottleneck

**Consistent Hashing**
```
Hash space [0, 2^160):
    Node A: points at 100, 500, 900
    Node B: points at 200, 600, 1000
    Node C: points at 300, 700, 1100

Entity key hashed to 450 -> belongs to Node A (nearest clockwise)
```
**Benefits**:
- Only K/N keys move when adding/removing nodes (K = keys, N = nodes)
- Virtual nodes for balance (each node appears multiple times)

**Real-World Implementation with ShardingSphere**
```yaml
# sharding configuration
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        url: jdbc:mysql://localhost:3306/ds0
      ds1:
        url: jdbc:mysql://localhost:3306/ds1
    sharding:
      tables:
        orders:
          actualDataNodes: ds${0..1}.orders_${0..1}
          tableStrategy:
            standard:
              shardingColumn: order_id
              shardingAlgorithmName: table_inline
          databaseStrategy:
            standard:
              shardingColumn: user_id
              shardingAlgorithmName: database_inline
      shardingAlgorithms:
        database_inline:
          type: INLINE
          props:
            algorithm-expression: ds${user_id % 2}
        table_inline:
          type: INLINE
          props:
            algorithm-expression: orders_${order_id % 2}
```

**Cross-Shard Query Patterns**
```java
// Scatter-gather query
public List<Order> getOrdersForUser(String userId) {
    int shard = calculateShard(userId);
    // All orders for user are on same shard if sharded by user_id
    return shardRepository.findByUserId(userId, shard);
}

// Multi-shard aggregation
public BigDecimal getTotalRevenue(LocalDate date) {
    List<CompletableFuture<BigDecimal>> futures = new ArrayList<>();
    for (int shard = 0; shard < numShards; shard++) {
        futures.add(CompletableFuture.supplyAsync(() ->
            shardRepository.sumRevenueForDate(date, shard)
        ));
    }
    return futures.stream()
        .map(CompletableFuture::join)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

**Sharding Anti-Patterns**

| Anti-Pattern | Why It's Bad | Better Approach |
|--------------|--------------|------------------|
| **Sharding by auto-increment ID** | Hard to route queries without ID | Use natural key (user_id, tenant_id) |
| **Cross-shard joins** | Extremely slow (N shard queries + application join) | Denormalize or use separate aggregation service |
| **Dynamic resharding during peak** | Causes massive downtime | Use consistent hashing with virtual nodes |
| **Shard key updates** | Requires moving data between shards | Make shard key immutable |

---

# 4. MICROSERVICES ARCHITECTURE

## 🏛️ Service Boundaries with DDD

### 🎯 Topic: Bounded Context Discovery

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**Event Storming Process**
```
1. Domain Events (orange): OrderPlaced, PaymentReceived, InventoryReserved
2. Commands (blue): PlaceOrder, ProcessPayment, ReserveInventory
3. Actors (yellow): Customer, Warehouse, PaymentGateway
4. Aggregates (green): Order, Payment, InventoryItem
5. External Systems (pink): TaxCalculator, FraudDetection
```

**Example: E-commerce Bounded Contexts**

**Order Context**
```java
// Domain model
@Aggregate
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> lines;
    private OrderStatus status;
    private Money total;
    
    public void place() {
        // Business rules
        if (lines.isEmpty()) throw new IllegalStateException();
        this.status = OrderStatus.PLACED;
        registerEvent(new OrderPlaced(this.id, this.total));
    }
}
```

**Payment Context**
```java
@Aggregate
public class Payment {
    private PaymentId id;
    private OrderId orderId;
    private Money amount;
    private PaymentStatus status;
    
    public void authorize() {
        // Different language - "authorize" vs "process"
        this.status = PaymentStatus.AUTHORIZED;
        registerEvent(new PaymentAuthorized(orderId, amount));
    }
}
```

**Context Mapping Patterns**

| Pattern | Direction | Coupling | Use Case |
|---------|-----------|----------|----------|
| **Shared Kernel** | Bidirectional | High (shared model) | Two teams close collaboration |
| **Customer-Supplier** | Upstream → Downstream | High | Payment upstream, Order downstream |
| **Conformist** | Downstream conforms to upstream | Medium | Following standard API (e.g., tax service) |
| **Anti-Corruption Layer** | Downstream isolates | Low | Legacy system integration |
| **Separate Ways** | None | None | Independent domains (analytics vs payments) |

**Anti-Corruption Layer Example**
```java
// Legacy Customer API (messy)
public class LegacyCustomerClient {
    public Map<String, Object> getCustomerData(String id) {
        return restTemplate.getForObject("/legacy/customer/" + id, Map.class);
        // Returns: { "cust_id": "123", "cust_name": "John", "cust_status": "A" }
    }
}

// Anti-corruption layer (translates to modern model)
@Service
public class CustomerAcl {
    @Autowired
    private LegacyCustomerClient legacyClient;
    
    public Customer getCustomer(String id) {
        Map<String, Object> legacy = legacyClient.getCustomerData(id);
        return new Customer(
            (String) legacy.get("cust_id"),
            (String) legacy.get("cust_name"),
            "ACTIVE".equals(legacy.get("cust_status"))
        );
    }
}
```

**Failure Scenario: Temporal Coupling**
```java
// Problem: Order service expects events in specific order
@EventListener
public void handle(InventoryReserved event) {
    // Assumes PaymentProcessed already happened
    orderService.shipOrder(event.getOrderId());  // What if payment not processed?
}

// Solution: Event versioning or state machine
@EventListener
public void handle(InventoryReserved event) {
    Order order = orderRepository.findById(event.getOrderId());
    if (order.isPaymentProcessed()) {
        orderService.shipOrder(event.getOrderId());
    } else {
        // Buffer event or store state
        eventBuffer.store(event);
    }
}
```

---

## 🔄 Resilience Patterns

### 🎯 Topic: Circuit Breaker Deep Dive

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Circuit Breaker States**
```
    [CLOSED] --------- failure threshold reached ---------> [OPEN]
       |                                                      |
       | success                     timeout (sleep window) |
       |                                                      |
       v <---------------------------------------------------- v
    [HALF-OPEN] -------- success -----------------------> [CLOSED]
       |
       | failure
       v
    [OPEN] (again)
```

**Resilience4j Implementation**
```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public CircuitBreaker paymentCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                    // 50% failure -> OPEN
            .slowCallRateThreshold(50)                   // 50% slow calls -> failure
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(30))  // half-open after 30s
            .permittedNumberOfCallsInHalfOpenState(10)   // 10 test calls
            .minimumNumberOfCalls(100)                   // Need 100 calls to compute
            .recordException(ex -> ex instanceof PaymentException)
            .build();
        
        return CircuitBreaker.of("payment-service", config);
    }
    
    @Bean
    public Retry paymentRetry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofMillis(500))
            .retryOnException(ex -> ex instanceof TimeoutException)
            .retryExceptions(TimeoutException.class)
            .failAfterMaxAttempts(true)
            .build();
        
        return Retry.of("payment-retry", config);
    }
}

@Service
public class PaymentService {
    
    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallbackPayment")
    @Retry(name = "payment-retry")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }
    
    @Bulkhead(name = "payment-bulkhead", type = Bulkhead.Type.THREADPOOL)
    public PaymentResult fallbackPayment(PaymentRequest request, Exception e) {
        // Return stale data, default value, or queue for later
        return PaymentResult.pending("Payment queued for retry");
    }
}
```

**Bulkhead Patterns**
```java
// ThreadPool Bulkhead (isolate thread pools)
@Bulkhead(name = "slow-service", type = Bulkhead.Type.THREADPOOL, 
          fallbackMethod = "fallback")
public String callSlowService() {
    // Thread pool limited to 10 threads, queue 20
    return slowService.call();
}

// Semaphore Bulkhead (limit concurrent calls)
@Bulkhead(name = "database-pool", type = Bulkhead.Type.SEMAPHORE,
          fallbackMethod = "dbFallback")
public Data queryDatabase() {
    // Only 5 concurrent calls, others rejected immediately
    return db.query();
}
```

**Production Tuning Guide**

| Parameter | Default | When to Increase | When to Decrease |
|-----------|---------|------------------|-------------------|
| **failureRateThreshold** | 50% | Critical services (lower threshold) | Noisy failure (flapping) |
| **waitDurationInOpenState** | 60s | Services with long recovery time | Fast-recovering services |
| **permittedCallsInHalfOpen** | 10 | High-volume services (avoid another open) | Low-volume services |
| **slowCallDuration** | 60s | Services with long processing | Latency-sensitive services |

**Real-World Failure: Circuit Breaker Hammering**
```java
// Problem: Half-open state with 10 permitted calls, but downstream can only handle 1
// Result: Downstream gets hammered, immediately re-opens circuit

// Solution: Gradual ramp-up
public class GradualCircuitBreaker {
    private int currentPermits = 1;
    
    public void onHalfOpenSuccess() {
        currentPermits = Math.min(currentPermits * 2, maxPermits);
    }
    
    public void onHalfOpenFailure() {
        currentPermits = 1;
        // Open circuit again
    }
}
```

---

## 🔀 Service Mesh (Istio)

### 🎯 Topic: Istio Deep Dive

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**Istio Architecture**
```
Control Plane (istiod):
  - Pilot: xDS server (service discovery, routing rules)
  - Mixer: Telemetry collection (deprecated, moved to Envoy)
  - Citadel: Certificate management (mTLS)
  - Galley: Configuration validation

Data Plane (Envoy sidecar):
  - Inbound listener: Port 15006
  - Outbound listener: Port 15001
  - Admin interface: Port 15000
```

**Traffic Management with VirtualService**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-routing
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10
    retries:
      attempts: 3
      perTryTimeout: 2s
    fault:
      delay:
        percentage:
          value: 5
        fixedDelay: 5s
      abort:
        percentage:
          value: 1
        httpStatus: 500
```

**DestinationRule for Circuit Breaking**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-destination
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 10  # Limit V2 capacity
```

**mTLS Configuration**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT  # REQUIRED for all workloads
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: enable-mtls
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

**Performance Impact of Service Mesh**

| Metric | Without Sidecar | With Sidecar (Envoy) | Difference |
|--------|----------------|---------------------|------------|
| **Latency (p99)** | 10ms | 12ms | +20% |
| **CPU overhead** | 0.5 core | 0.7 core | +40% |
| **Memory per pod** | 200MB | 300MB | +50% |
| **Connection setup** | 1ms | 2ms | +100% |

**When to Use Service Mesh**
```
✅ Use Service Mesh if:
- Multiple languages/services (polyglot)
- Need mTLS without code changes
- Progressive delivery (canaries, blue/green) at scale
- Centralized observability across services
- Team lacks time for client libraries

❌ Don't use Service Mesh if:
- Single language (Java) - use Spring Cloud
- Latency-sensitive (<5ms p99) - sidecar adds overhead
- Small number of services (<10)
- Limited CPU/memory budget
```

---

# 5. MESSAGING & EVENT-DRIVEN SYSTEMS

## 📨 Apache Kafka Deep Dive

### 🎯 Topic: Kafka Architecture & Performance

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Core Architecture Components**
```
Producer → Partition (Leader) → Follower (ISR) → Consumer
                ↓
          Commit Log (Segment files)
                ↓
          Index (.index, .timeindex)
```

**Replication & ISR**
- **ISR (In-Sync Replicas)**: Replicas fully caught up with leader
- **LEO (Log End Offset)**: Last offset in partition
- **HW (High Watermark)**: Last offset committed to all ISRs
- **Replica fetcher**: Followers pull from leader (not push)
- **MinISR**: Minimum ISR count for successful writes (`min.insync.replicas`)

**Exactly-Once Semantics (EOS)**
```java
// Producer with idempotence
Properties props = new Properties();
props.put("enable.idempotence", "true");        // Single partition exactly-once
props.put("acks", "all");                       // Wait for all ISRs
props.put("max.in.flight.requests.per.connection", "5"); // Works with idempotence

// Transactions across partitions
props.put("transactional.id", "prod-123");      // Required for EOS
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

producer.beginTransaction();
try {
    producer.send(record1);
    producer.send(record2);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**Consumer Group Rebalancing**
```java
// Partition assignment strategies
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");

// Static group membership (avoid rebalance)
props.put("group.instance.id", "consumer-1");   // Static member

// Rebalance listener for stateful processing
class StatefulRebalanceListener implements ConsumerRebalanceListener {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Save state before losing partitions
        saveOffsets(consumer.position(partitions));
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Restore state for new partitions
        restoreOffsets(partitions);
    }
}
```

**Performance Tuning Matrix**

| Parameter | Default | Tuning Guide | Impact |
|-----------|---------|--------------|--------|
| **batch.size** | 16KB | Increase for high throughput (64KB-1MB) | Memory vs latency |
| **linger.ms** | 0 | Set to 5-100ms for batching | Latency vs throughput |
| **compression.type** | none | snappy/lz4 for text, zstd for high compression | CPU vs bandwidth |
| **buffer.memory** | 32MB | Increase for bursty producers | Memory |
| **fetch.min.bytes** | 1 | Increase for higher throughput (10KB) | Latency vs throughput |

**Real-World Failure: Rebalance Storm**
```java
// Problem: Processing too slow, exceeding max.poll.interval.ms (default 5 min)
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    records.forEach(record -> {
        processRecord(record);  // Takes 10 seconds per record
        // But consumer hasn't called poll() for 10 secs
    });
    consumer.commitSync();
}

// Fix #1: Increase max.poll.interval.ms
props.put("max.poll.interval.ms", "600000");  // 10 minutes

// Fix #2: Reduce max.poll.records
props.put("max.poll.records", "100");  // Process smaller batches

// Fix #3: Use async with dedicated thread pool
ExecutorService executor = Executors.newFixedThreadPool(10);
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    List<CompletableFuture<Void>> futures = new ArrayList<>();
    records.forEach(record -> {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> 
            processRecord(record), executor
        );
        futures.add(future);
    });
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
    consumer.commitSync();
}
```

---

## 📡 Event Sourcing & CQRS

### 🎯 Topic: Event Sourcing Implementation

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Event Store Schema**
```sql
CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    aggregate_id VARCHAR(100) NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL,
    version INT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_aggregate ON events(aggregate_id, version);
CREATE INDEX idx_type_time ON events(aggregate_type, created_at);
```

**Aggregate Rehydration**
```java
public class BankAccount {
    private String accountId;
    private BigDecimal balance;
    private int version;
    private List<DomainEvent> changes = new ArrayList<>();
    
    public static BankAccount reconstitute(List<DomainEvent> events) {
        BankAccount account = new BankAccount();
        for (DomainEvent event : events) {
            account.apply(event, false);  // Apply without recording changes
        }
        return account;
    }
    
    public void apply(DomainEvent event, boolean isNew) {
        if (event instanceof MoneyDeposited) {
            this.balance = this.balance.add(((MoneyDeposited) event).getAmount());
        } else if (event instanceof MoneyWithdrawn) {
            this.balance = this.balance.subtract(((MoneyWithdrawn) event).getAmount());
        }
        this.version = event.getVersion();
        if (isNew) {
            changes.add(event);
        }
    }
    
    public void deposit(Money amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException();
        }
        apply(new MoneyDeposited(accountId, amount, version + 1), true);
    }
}
```

**Snapshotting Strategy**
```sql
CREATE TABLE snapshots (
    aggregate_id VARCHAR(100) PRIMARY KEY,
    aggregate_type VARCHAR(50),
    state JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Snapshot every 100 events
@Scheduled(fixedDelay = 60000)
public void createSnapshots() {
    List<String> aggregates = eventStore.getAggregatesWithManyEvents(100);
    for (String aggregateId : aggregates) {
        List<DomainEvent> events = eventStore.getEvents(aggregateId, lastSnapshotVersion);
        BankAccount account = BankAccount.reconstitute(events);
        snapshotRepository.save(new Snapshot(aggregateId, account, account.getVersion()));
    }
}
```

**Projection Building (CQRS Read Model)**
```java
@Component
public class AccountBalanceProjection {
    
    @Autowired
    private MongoTemplate mongo;
    
    @EventListener
    public void on(MoneyDeposited event) {
        // Update denormalized read model
        Query query = Query.query(Criteria.where("_id").is(event.getAccountId()));
        Update update = new Update().inc("balance", event.getAmount().doubleValue());
        mongo.upsert(query, update, AccountBalance.class);
    }
    
    @EventListener
    public void on(MoneyWithdrawn event) {
        Query query = Query.query(Criteria.where("_id").is(event.getAccountId()));
        Update update = new Update().inc("balance", -event.getAmount().doubleValue());
        mongo.upsert(query, update, AccountBalance.class);
    }
}
```

**Real-World Failure: Event Version Clash**
```java
// Problem: Two concurrent commands on same aggregate
// Command 1: Deposit $100 (expects version 5)
// Command 2: Withdraw $50 (expects version 5)

// Solution: Optimistic concurrency
@Transactional
public void handleCommand(Command command) {
    int expectedVersion = command.getExpectedVersion();
    int actualVersion = eventStore.getLatestVersion(command.getAggregateId());
    
    if (expectedVersion != actualVersion) {
        throw new ConcurrencyException("Expected version " + expectedVersion + 
                                       " but was " + actualVersion);
    }
    // Apply command and store events
    eventStore.saveEvents(command.generateEvents());
}
```

**Event Versioning with Avro**
```avro
{
  "type": "record",
  "name": "MoneyDeposited",
  "namespace": "com.bank.events",
  "version": "2",
  "fields": [
    {"name": "accountId", "type": "string"},
    {"name": "amount", "type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2},
    {"name": "reference", "type": ["null", "string"], "default": null},
    {"name": "source", "type": {"type": "enum", "name": "Source", "symbols": ["ATM", "TRANSFER", "CHEQUE"]}}
  ]
}
```

---

# 6. CLOUD PLATFORMS (AWS/AZURE)

## ☁️ AWS Core Services Deep Dive

### 🎯 Topic: S3 Internals & Performance

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**S3 Consistency Model**
```
Before 2020: Eventual consistency for overwrites and deletes
After 2020: Strong consistency for all operations (except LIST)

Current model:
- PUT/POST/DELETE: Strong (read-after-write)
- LIST: Eventually consistent (may not see new objects immediately)
- Overwrite: Strong (subsequent GET sees updated value)
```

**S3 Performance Optimization**
```java
// ❌ Sequential key pattern (hot partition)
s3.putObject("logs/2024/01/15/12/30/45/event-123.log");

// ✅ Random prefix for high throughput
s3.putObject("logs/2024/01/15/12/30/45/" + UUID.randomUUID() + "/event.log");

// For 3,500+ PUT/LIST/DELETE per second, add random prefix
String key = String.format("%04x", ThreadLocalRandom.current().nextInt(0x10000)) 
             + "/" + actualKey;
```

**Multipart Upload**
```java
// For files > 100MB, always use multipart
InitiateMultipartUploadRequest initRequest = new InitiateMultipartUploadRequest(bucket, key);
InitiateMultipartUploadResult initResponse = s3.initiateMultipartUpload(initRequest);

List<PartETag> partETags = new ArrayList<>();
for (int i = 1; i <= totalParts; i++) {
    UploadPartRequest uploadRequest = new UploadPartRequest()
        .withBucketName(bucket)
        .withKey(key)
        .withUploadId(initResponse.getUploadId())
        .withPartNumber(i)
        .withFileOffset(offset)
        .withPartSize(partSize)
        .withLastPart(i == totalParts);
    
    UploadPartResult uploadResult = s3.uploadPart(uploadRequest);
    partETags.add(uploadResult.getPartETag());
}

CompleteMultipartUploadRequest compRequest = new CompleteMultipartUploadRequest(
    bucket, key, initResponse.getUploadId(), partETags);
s3.completeMultipartUpload(compRequest);
```

**S3 Performance Numbers**

| Operation | Requests/sec (prefix) | Latency (p99) | Optimization |
|-----------|---------------------|---------------|--------------|
| **PUT (1KB)** | 3,500 | 10-20ms | Add random prefix |
| **PUT (1MB)** | 3,500 (but less throughput) | 50-100ms | Use multipart |
| **GET (1KB)** | 5,500 | 5-10ms | Use CloudFront for static |
| **LIST** | 1,000 | 100-500ms | Paginate, avoid during writes |

---

## 🐳 EKS vs ECS Decision Framework

### 🎯 Topic: Container Orchestration Comparison

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**EKS Architecture**
```
Control Plane (AWS managed):
  - API Server (3 nodes)
  - etcd (encrypted, backed up)
  - Controller Manager
  - Scheduler

Data Plane (Self-managed or managed node groups):
  - EC2 instances (or Fargate)
  - Kubelet
  - Container runtime (containerd)
  - kube-proxy (or CNI)
```

**ECS Architecture**
```
Control Plane (AWS managed):
  - ECS API endpoint (global)
  - Cluster state stored in AWS

Data Plane:
  - ECS Agent (container on EC2 or Fargate)
  - Task definition to container mapping
```

**Comparison Matrix**

| Aspect | EKS | ECS | Winner |
|--------|-----|-----|--------|
| **Portability** | Any K8s environment | AWS-only | EKS |
| **Ease of use** | Steep learning curve | Simple API | ECS |
| **Ecosystem** | Helm, Istio, Prometheus operators | Limited | EKS |
| **Upgrades** | Manual control plane upgrade | Managed automatically | ECS |
| **Cost** | Control plane $73/month | Free (pay for resources) | ECS |
| **Multi-architecture** | x86, ARM, GPU support | Limited | EKS |
| **Spot instances** | Good (Pod Disruption Budgets) | Basic | EKS |
| **Observability** | Prometheus, EFK, CloudWatch | Primarily CloudWatch | EKS |

**When to Use Each**
```yaml
# Use EKS if:
- Multi-cloud or hybrid strategy
- Existing K8s expertise
- Need ecosystem (Istio, Knative, Kubeflow)
- Complex scheduling requirements

# Use ECS if:
- AWS-only or simple workloads
- Deep AWS integration needed
- Small team without K8s knowledge
- Cost-sensitive (no control plane cost)
```

**Real-World Migration: ECS → EKS**
```python
# ECS Task Definition
{
    "family": "payment-worker",
    "taskRoleArn": "arn:aws:iam::123:role/ecsTaskRole",
    "containerDefinitions": [{
        "name": "app",
        "image": "123.dkr.ecr/payment:latest",
        "memory": 1024,
        "cpu": 512,
        "environment": [{"name": "QUEUE_URL", "value": "https://sqs"}]
    }]
}

# Equivalent K8s Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-worker
  template:
    metadata:
      labels:
        app: payment-worker
    spec:
      containers:
      - name: app
        image: 123.dkr.ecr/payment:latest
        resources:
          requests:
            memory: "1024Mi"
            cpu: "500m"
          limits:
            memory: "1024Mi"
            cpu: "1000m"
        env:
        - name: QUEUE_URL
          value: "https://sqs"
```

---

# 7. DEVOPS & KUBERNETES

## 🔧 Kubernetes Scheduling & Autoscaling

### 🎯 Topic: Advanced Scheduling

**Priority:** HIGH | **Depth:** Master

#### Subtopics & Microtopics

**Pod Priority & Preemption**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-service
value: 1000000       # Higher value = higher priority
globalDefault: false
description: "Critical business services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-job
value: 100
preemptionPolicy: Never   # Can't preempt others
---
apiVersion: v1
kind: Pod
metadata:
  name: payment-service
spec:
  priorityClassName: critical-service
  containers:
  - name: app
    image: payment:latest
```

**Node Affinity & Anti-Affinity**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: ["ssd-optimized"]
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a"]
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: ["database"]
            topologyKey: kubernetes.io/hostname
```

**Topology Spread Constraints**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                # Max difference of 1 pod between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
```

**HPA with Custom Metrics**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kafka-consumer-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "100"          # Scale if avg lag >100
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0   # Scale up immediately
      policies:
      - type: Pods
        value: 4
        periodSeconds: 15
```

**Production Failure: HPA Oscillation**
```yaml
# Problem: Metrics noise causing constant scaling
# Symptoms: Replicas flapping 3→10→3→10 every minute

# Solution: Stabilization windows + threshold hysteresis
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60    # Wait 1m before scaling up
    policies:
    - type: Percent
      value: 100                      # Double at most
      periodSeconds: 30
  scaleDown:
    stabilizationWindowSeconds: 300   # Wait 5m before scaling down
    policies:
    - type: Percent
      value: 25                       # Scale down 25% at a time
      periodSeconds: 60
```

---

## 📊 Observability Stack

### 🎯 Topic: Prometheus & Grafana Deep Dive

**Priority:** HIGH | **Depth:** Strong

#### Subtopics & Microtopics

**Prometheus Metrics Types**
```java
// Counter (only increases, resets on restart)
Counter requestsTotal = Counter.build()
    .name("http_requests_total")
    .help("Total HTTP requests")
    .register();

// Gauge (goes up and down)
Gauge inProgressRequests = Gauge.build()
    .name("http_requests_in_progress")
    .help("Current in-progress requests")
    .register();

// Histogram (bucketed, for latency)
Histogram requestDuration = Histogram.build()
    .name("http_request_duration_seconds")
    .help("HTTP request latency")
    .buckets(0.01, 0.05, 0.1, 0.5, 1, 2, 5)
    .register();

// Summary (quantiles client-side)
Summary requestLatency = Summary.build()
    .name("http_request_latency_seconds")
    .help("HTTP request latency")
    .quantile(0.5, 0.05)   // 50th percentile, error 5%
    .quantile(0.99, 0.001) // 99th percentile, error 0.1%
    .register();
```

**PromQL Queries for SLOs**
```promql
# Error rate (last 5 minutes)
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))

# P99 latency
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Saturation (CPU usage / requests)
sum(rate(container_cpu_usage_seconds_total[5m])) / sum(machine_cpu_cores)

# Service availability (up/down)
avg_over_time(up{job="payment-service"}[30d]) * 100
```

**Alerting Rules**
```yaml
groups:
- name: service_slos
  rules:
  - alert: HighErrorRate
    expr: |
      (sum(rate(http_requests_total{status=~"5.."}[5m])) 
      / 
      sum(rate(http_requests_total[5m]))) > 0.01
    for: 5m
    labels:
      severity: critical
      team: platform
    annotations:
      summary: "High error rate on {{ $labels.service }}"
      description: "Error rate is {{ $value | humanizePercentage }}"
      runbook: "https://wiki/runbooks/high-error-rate"
      
  - alert: HighLatency
    expr: |
      histogram_quantile(0.99, 
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
      ) > 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High latency detected"
```

**Log Aggregation with Loki**
```yaml
# Promtail config (log shipping)
scrape_configs:
- job_name: kubernetes-pods
  kubernetes_sd_configs:
  - role: pod
  pipeline_stages:
  - match:
      selector: '{app="payment-service"}'
      stages:
      - regex:
          expression: '.*level=(?P<level>\w+).*'
      - labels:
          level:
  - docker: {}
  relabel_configs:
  - source_labels: ['__meta_kubernetes_pod_label_app']
    target_label: 'app'
  - source_labels: ['__meta_kubernetes_namespace']
    target_label: 'namespace'
```

---

# 8. SYSTEM DESIGN (CRITICAL SECTION)

## 🏗️ System Design Interview Framework

### Step 1: Clarify Requirements (5 minutes)

**Key Questions to Ask**
```
Functional Requirements:
• What are the primary use cases? (Read vs write heavy?)
• Will there be real-time updates? (Push or pull?)
• Do we need offline support?

Non-Functional Requirements:
• Expected DAU/MAU?
• Read/Write QPS?
• P99 latency requirements?
• Durability / consistency needs?
• Budget constraints?

Constraints:
• Time to implement?
• Team size?
• Existing systems to integrate?
```

**Example - Design Twitter**
```
Candidate: "Should this be real-time timeline generation or can it be slightly stale?"
Interviewer: "Real-time is important, but eventual consistency is fine for seconds"
Candidate: "What about scale? Number of users?"
Interviewer: "Let's aim for 100M DAU, 5k tweets/sec"
Candidate: "Can we assume users follow at most 5k accounts to bound fanout?"
Interviewer: "Yes, that's reasonable"
```

---

### Step 2: Define Scale (3 minutes)

**Back-of-the-Envelope Calculations**
```python
# Twitter example
DAU = 100_000_000
Write QPS = 5_000 tweets/second
Read QPS (timeline) = DAU * 0.1 (10% active per second) = 10_000_000 read/sec

Storage:
- Tweet size: 1KB (text + metadata)
- Daily writes: 5k * 86400 = 432M tweets/day
- Daily storage: 432M * 1KB = 432 GB/day
- 5 years: 432 GB * 365 * 5 = 788 TB (with replication: ~2.4 PB)

Cache for hot users:
- Top 1% users by followers: 1M users
- Each cached timeline: 800 tweets * 1KB = 800KB
- Cache size: 1M * 800KB = 800GB
```

---

### Step 3: High-Level Design (5 minutes)

**Component Diagram**
```
[Client] → [CDN] → [API Gateway] → [Load Balancer] → [Web Server (stateless)]
                                ↓
                        [Service Layer]
                    ↙     ↓          ↘
            [Timeline] [Tweet]    [User]
                ↓         ↓          ↓
            [Cache]    [DB]       [Cache]
                ↓         ↓
            [Queue] ← [Event Bus]
```

**Data Flow - Write Path (Post Tweet)**
1. User posts tweet → API Gateway
2. Validate authentication
3. Store in Tweet DB
4. Fan-out to followers' timelines:
   - Push: For users with <5k followers, write to timeline table
   - Pull: For celeb users (>10M followers), just mark new tweet
5. Add to social graph service
6. Return success

---

### Step 4: Deep Dive on Components (15 minutes)

**Pick 2-3 components for deep dive**

**Component 1: Timeline Generation (Fan-out)**
```sql
-- Push approach (write on tweet)
CREATE TABLE timeline (
    user_id BIGINT,
    tweet_id BIGINT,
    author_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at)
) PARTITION BY HASH(user_id);

-- INSERT for all followers (may be thousands)
INSERT INTO timeline VALUES (follower_id, tweet_id, author_id, now());

-- Pull approach (read on request)
-- Just query tweets from followed users
SELECT * FROM tweets 
WHERE author_id IN (SELECT following FROM follows WHERE follower_id = ?)
ORDER BY created_at DESC LIMIT 800;
```

**Hybrid Approach:**
```python
class TimelineService:
    def fanout(self, tweet):
        followers = social_graph.get_followers(tweet.author_id)
        
        # Celebrities: don't push
        if len(followers) > 1_000_000:
            # Mark as "pull mode" for followers
            redis.sadd("celeb_tweets", tweet.id)
            return
        
        # Normal users: push to timeline
        # Batch writes to timeline table
        batch = []
        for follower in followers:
            batch.append((follower, tweet.id, tweet.author_id, now()))
            if len(batch) == 1000:
                timeline_db.insert(batch)
                batch = []
    
    def get_timeline(self, user_id, limit=800):
        # First, get from timeline table (pushed tweets)
        timeline = timeline_db.query(user_id, limit)
        
        # Then, check if following any celebrities
        celebs = social_graph.get_followed_celebs(user_id)
        if celebs:
            celeb_tweets = tweet_db.query_by_authors(celebs, limit)
            # Merge and sort by time
            timeline = merge(timeline, celeb_tweets)
        
        return timeline[:limit]
```

**Tradeoffs:**
| Aspect | Push | Pull | Hybrid |
|--------|------|------|--------|
| Write amplification | High (fan-out) | None | Medium |
| Read latency | Low (pre-computed) | High (join queries) | Medium |
| Celebrities | Impossible | Works fine | Works |
| Storage | High (duplicate tweets) | Low | Medium |

---

### Step 5: Identify Bottlenecks & Solutions

**Bottleneck 1: Database Write for Celeb Tweet**
```python
# Problem: 50M followers INSERT = 50M writes
# Solution: Don't push - use cache for celeb tweets
class TimelineCache:
    def __init__(self):
        self.l1 = Redis()  # Hot cache (last hour)
        self.l2 = Memcached()  # Warm cache (last day)
    
    def get(self, user_id):
        # Try L1 (1hr TTL)
        timeline = self.l1.get(user_id)
        if timeline:
            return timeline
        
        # Try L2 (24hr TTL)
        timeline = self.l2.get(user_id)
        if timeline:
            self.l1.set(user_id, timeline, 3600)
            return timeline
        
        # Compute from DB
        timeline = self.compute_timeline(user_id)
        self.l2.set(user_id, timeline, 86400)
        return timeline
```

**Bottleneck 2: Hot User Timeline**
```
Problem: Celeb with 50M followers gets 5 tweets/min = 250M writes/min
Solution: 
1. Rate limiting on tweet frequency
2. Only push to active followers (tweet in last hour)
3. Use Redis sorted sets for sliding window
```

**Bottleneck 3: Data Growth**
```sql
-- Solution: Time-based sharding + cold storage
CREATE TABLE tweets_2024_01 PARTITION OF tweets 
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Move cold data to S3
ALTER TABLE tweets DETACH PARTITION tweets_2023_01;
EXPORT TO S3 SELECT * FROM tweets_2023_01;
DROP TABLE tweets_2023_01;
```

---

### Step 6: Tradeoffs Summary

**Create Decision Matrix**

| Decision | Option A | Option B | Winner | Why |
|----------|----------|----------|--------|-----|
| **Database** | PostgreSQL | Cassandra | PostgreSQL | Consistency > scale |
| **Cache** | Redis | Memcached | Redis | Data structures (sorted sets) |
| **Queue** | Kafka | SQS | Kafka | Ordering, retention |
| **Storage** | EBS | S3 | S3 | Cost + durability |

---

### Step 7: Improvements & Scaling

**Phase 1: MVP (1M DAU)**
```
- Single PostgreSQL DB
- Redis cache for timeline
- Direct fanout (push to all)
```

**Phase 2: Growth (10M DAU)**
```
- Read replicas for timeline queries
- Shard tweets by user_id
- Async fanout with queue
```

**Phase 3: Scale (100M DAU)**
```
- Hybrid fanout (push for small, pull for celebs)
- Multi-region with active-passive
- CDN for media
```
---

## 🧪 System Design Case Studies

### Case Study 1: URL Shortener (like bit.ly)

**Requirements:**
- Shorten URL: long → short (6-8 chars)
- Redirect: short → long with 301/302
- Analytics: clicks, referrers, geolocation
- Scale: 100M new URLs/month, 1B redirects/day

**High-Level Design:**
```
[Client] → [LB] → [Web Server]
    ↓           ↓
[Cache]     [Key Generator] → [DB]
                ↓
            [ID Service]
```

**Deep Dive: Key Generation**
```java
// Approach 1: Base62 encoding of DB ID
long id = idGenerator.nextId(); // Snowflake
String shortCode = base62Encode(id); // 123456 → "dU7"

// Approach 2: Pre-generated keys
public class KeyService {
    private Queue<String> keyPool = new ConcurrentLinkedQueue<>();
    
    @PostConstruct
    public void init() {
        // Generate keys in batch
        while (keyPool.size() < 10000) {
            keyPool.add(generateRandomKey());
        }
    }
    
    // Background replenisher
    @Scheduled(fixedDelay = 1000)
    public void replenish() {
        if (keyPool.size() < 5000) {
            generateBatch(5000);
        }
    }
}
```

**Database Schema:**
```sql
CREATE TABLE url_mapping (
    short_code VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP,
    expires_at TIMESTAMP,
    INDEX idx_created (created_at)
);

CREATE TABLE clicks (
    short_code VARCHAR(10),
    clicked_at TIMESTAMP,
    referrer VARCHAR(500),
    user_agent TEXT,
    ip_address INET,
    INDEX idx_code_time (short_code, clicked_at)
) PARTITION BY RANGE (clicked_at);
```

**Tradeoffs:**
| Decision | Option | Tradeoff |
|----------|--------|----------|
| **Key length** | 6 chars (56B combos) vs 8 (218T) | Short: collisions risk (mitigate with retry) |
| **Redirect** | 301 (Permanent) vs 302 (Temporary) | 301: cached forever (can't update), 302: always check |
| **Analytics** | Sync (in request) vs Async (queue) | Sync: consistent but slow, Async: eventual but fast |

**Failure Scenarios:**
1. Key collision: Retry with new key
2. DB down: Cache short codes in Redis (5min TTL)
3. Hot URL: Cache frequently accessed URLs in CDN

---

### Case Study 2: Payment System

**Requirements:**
- Process payments: $1B/day volume
- Idempotent: No double charging
- Atomic: Money moved atomically from payer to payee
- Audit: Every transaction logged
- Disputes: Handle chargebacks

**High-Level Design:**
```
[API] → [Payment Service] → [Processor] → [Bank Gateway]
            ↓                    ↓
        [Ledger DB]          [Queue]
            ↓                    ↓
        [Reporting]          [Reconciler]
```

**Deep Dive: Idempotency**
```java
@Service
public class PaymentService {
    
    @Transactional
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. Idempotency check
        String idempotencyKey = request.getIdempotencyKey();
        Payment existing = paymentRepo.findByIdempotencyKey(idempotencyKey);
        if (existing != null) {
            return existing.getResult();
        }
        
        // 2. Atomic transaction
        try {
            // Debit payer
            ledgerService.debit(request.getPayerId(), request.getAmount());
            
            // Credit payee (but may fail)
            ledgerService.credit(request.getPayeeId(), request.getAmount());
            
            // Record success
            Payment payment = Payment.success(request);
            paymentRepo.save(payment);
            
            // Publish event
            eventPublisher.publish(new PaymentCompleted(payment));
            
            return PaymentResult.success();
            
        } catch (InsufficientFundsException e) {
            // Rollback automatically (@Transactional)
            Payment payment = Payment.failed(request, "INSUFFICIENT_FUNDS");
            paymentRepo.save(payment);
            return PaymentResult.failed("Insufficient funds");
            
        } catch (Exception e) {
            // Unhandled - rollback
            Payment payment = Payment.failed(request, "SYSTEM_ERROR");
            paymentRepo.save(payment);
            throw e;
        }
    }
}

// Idempotency key generation (client-side)
public class IdempotencyKeyGenerator {
    public static String generate(String userId, Cart cart) {
        // Deterministic: same request = same key
        return Base64.encode(
            sha256(userId + cart.id + cart.totalAmount)
        );
    }
}
```

**Double-Entry Accounting Schema:**
```sql
CREATE TABLE ledger_entries (
    entry_id UUID PRIMARY KEY,
    transaction_id UUID NOT NULL,
    account_id BIGINT NOT NULL,
    amount DECIMAL(20,2) NOT NULL,
    entry_type ENUM('DEBIT', 'CREDIT'),
    status ENUM('PENDING', 'POSTED', 'CANCELLED'),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_transaction (transaction_id),
    INDEX idx_account (account_id, created_at)
);

-- Check balance (for fraud detection)
CREATE MATERIALIZED VIEW account_balance AS
SELECT 
    account_id,
    SUM(CASE WHEN entry_type = 'DEBIT' THEN -amount ELSE amount END) as balance
FROM ledger_entries
WHERE status = 'POSTED'
GROUP BY account_id;
```

**Reconciliation Process:**
```java
@Service
public class ReconciliationService {
    
    // Run every hour
    @Scheduled(cron = "0 * * * *")
    public void reconcile() {
        // Compare internal ledger with bank statements
        List<Payment> payments = paymentRepo.findUnreconciled();
        
        for (Payment payment : payments) {
            BankStatement statement = bankApi.getStatement(payment.getBankRef());
            
            if (statement.isMatched(payment)) {
                payment.setReconciled(true);
                paymentRepo.save(payment);
            } else if (statement.isMissing()) {
                // Payment sent but not cleared by bank
                alertService.send("Payment reconciliation failed", payment);
            }
        }
    }
}
```

**Failure Scenarios:**
1. **Network timeout after debit but before credit**: Check idempotency key, resume where left off
2. **Bank says success but payment service timeout**: Dispute resolution via reconciliation
3. **Ledger DB deadlock**: Retry with exponential backoff (max 3 attempts)

---

### Case Study 3: Rate Limiter

**Requirements:**
- Limit requests per user: 100 requests/minute
- Limit per IP: 1000 requests/hour
- Distributed (multiple servers)
- Low latency (<5ms overhead)

**Algorithms Deep Dive:**

**1. Token Bucket**
```java
public class TokenBucketRateLimiter {
    private final long capacity;      // Max tokens (e.g., 100)
    private final long refillRate;    // Tokens/sec (e.g., 2)
    private double tokens;
    private long lastRefillTimestamp;
    
    public synchronized boolean allow() {
        refill();
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.currentTimeMillis();
        double elapsed = (now - lastRefillTimestamp) / 1000.0;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefillTimestamp = now;
    }
}
```

**2. Sliding Window Log (Most Accurate)**
```java
public class SlidingWindowLog {
    private final long windowSizeMs = 60000;  // 1 minute
    private final int maxRequests = 100;
    private Queue<Long> timestamps = new ConcurrentLinkedQueue<>();
    
    public synchronized boolean allow() {
        long now = System.currentTimeMillis();
        
        // Remove old timestamps
        while (!timestamps.isEmpty() && 
               timestamps.peek() < now - windowSizeMs) {
            timestamps.poll();
        }
        
        if (timestamps.size() < maxRequests) {
            timestamps.add(now);
            return true;
        }
        return false;
    }
}
```

**3. Sliding Window Counter (Redis-based)**
```redis
-- Lua script for atomic operation
local key = KEYS[1]
local windowMs = tonumber(ARGV[1])
local maxRequests = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove old timestamps
redis.call('ZREMRANGEBYSCORE', key, 0, now - windowMs)

-- Count current requests
local count = redis.call('ZCARD', key)

if count < maxRequests then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, (windowMs / 1000) + 1)
    return {true, count + 1}
else
    return {false, count}
end
```

**Distributed Rate Limiting with Redis Cluster:**
```java
@Component
public class DistributedRateLimiter {
    @Autowired
    private RedisTemplate<String, String> redis;
    
    public boolean allow(String key, long limit, long windowSeconds) {
        String luaScript = 
            "local current = redis.call('INCR', KEYS[1])\n" +
            "if current == 1 then\n" +
            "    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))\n" +
            "end\n" +
            "return current <= tonumber(ARGV[2])";
        
        Long current = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Arrays.asList(key),
            windowSeconds, limit
        );
        
        return current == 1;  // 1 = allowed, 0 = throttled
    }
}

// Usage
if (!rateLimiter.allow("user:" + userId, 100, 60)) {
    throw new RateLimitExceededException();
}
```

**Tradeoffs Comparison:**

| Algorithm | Memory | Accuracy | Distributed | Use Case |
|-----------|--------|----------|-------------|----------|
| **Fixed Window** | Low | Poor (burst at boundary) | Easy | Basic throttling |
| **Token Bucket** | Low | Good (allows bursts) | Complex | API rate limiting |
| **Sliding Log** | High (store all timestamps) | Perfect | Hard | Critical accuracy |
| **Sliding Counter** | Medium (only counts) | Very good | Easy | Most cases |

---

# 9. DSA FOR STAFF ENGINEERS

## High-ROI Patterns Only

### Pattern 1: Sliding Window

**When to Use:**
- Subarray/substring problems
- Rate limiting
- Moving average

**Example: Maximum sum subarray of size K**
```java
public int maxSumSubarray(int[] arr, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    
    int maxSum = windowSum;
    for (int i = k; i < arr.length; i++) {
        windowSum = windowSum - arr[i - k] + arr[i];
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

**Real-world: Rate Limiter with sliding window**
```java
class SlidingWindowRateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private final Queue<Long> timestamps = new LinkedList<>();
    
    public SlidingWindowRateLimiter(int maxRequests, long windowSeconds) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowSeconds * 1000;
    }
    
    public boolean allow() {
        long now = System.currentTimeMillis();
        
        // Remove expired timestamps
        while (!timestamps.isEmpty() && 
               timestamps.peek() < now - windowMillis) {
            timestamps.poll();
        }
        
        if (timestamps.size() < maxRequests) {
            timestamps.add(now);
            return true;
        }
        return false;
    }
}
```

---

### Pattern 2: Two Pointers

**When to Use:**
- Sorted arrays
- Palindrome checking
- Three-sum problems

**Example: Find pair with target sum (sorted array)**
```java
public int[] twoSum(int[] numbers, int target) {
    int left = 0, right = numbers.length - 1;
    
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target) {
            return new int[]{left + 1, right + 1};
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    return new int[]{-1, -1};
}
```

**Real-world: Merge two sorted timelines (Twitter)**
```java
public List<Tweet> mergeTimelines(List<Tweet> timeline1, List<Tweet> timeline2, int limit) {
    List<Tweet> merged = new ArrayList<>();
    int i = 0, j = 0;
    
    while (merged.size() < limit && (i < timeline1.size() || j < timeline2.size())) {
        if (i == timeline1.size()) {
            merged.add(timeline2.get(j++));
        } else if (j == timeline2.size()) {
            merged.add(timeline1.get(i++));
        } else if (timeline1.get(i).getTimestamp() > timeline2.get(j).getTimestamp()) {
            merged.add(timeline1.get(i++));
        } else {
            merged.add(timeline2.get(j++));
        }
    }
    return merged;
}
```

---

### Pattern 3: Producer-Consumer (for system design)

**When to Use:**
- Async processing
- Rate limiting
- Backpressure handling

**Example: Thread-safe queue with backpressure**
```java
public class BoundedQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final Semaphore slots;  // Empty slots
    private final Semaphore items;  // Items in queue
    
    public BoundedQueue(int capacity) {
        this.capacity = capacity;
        this.slots = new Semaphore(capacity);
        this.items = new Semaphore(0);
    }
    
    public void put(T item) throws InterruptedException {
        slots.acquire();  // Wait for empty slot
        synchronized (queue) {
            queue.add(item);
        }
        items.release();  // Signal item available
    }
    
    public T take() throws InterruptedException {
        items.acquire();  // Wait for item
        T item;
        synchronized (queue) {
            item = queue.remove();
        }
        slots.release();  // Signal empty slot
        return item;
    }
}
```

---

### Pattern 4: Top K Elements

**When to Use:**
- Most frequent items
- Leaderboard
- Trending topics

**Example: Top K frequent words**
```java
public List<String> topKFrequent(String[] words, int k) {
    Map<String, Integer> freq = new HashMap<>();
    for (String word : words) {
        freq.put(word, freq.getOrDefault(word, 0) + 1);
    }
    
    // Min heap (smallest at top)
    PriorityQueue<Map.Entry<String, Integer>> heap = 
        new PriorityQueue<>((a, b) -> 
            a.getValue().equals(b.getValue()) 
                ? b.getKey().compareTo(a.getKey())  // Larger word first (for smallest)
                : a.getValue() - b.getValue()
        );
    
    for (Map.Entry<String, Integer> entry : freq.entrySet()) {
        heap.offer(entry);
        if (heap.size() > k) {
            heap.poll();
        }
    }
    
    List<String> result = new ArrayList<>();
    while (!heap.isEmpty()) {
        result.add(0, heap.poll().getKey());
    }
    return result;
}
```

**Real-world: Most recent tweets**
```java
public class UserTimeline {
    // Use min-heap for latest N tweets
    private PriorityQueue<Tweet> recentTweets = new PriorityQueue<>(
        (a, b) -> Long.compare(a.getTimestamp(), b.getTimestamp())
    );
    private final int maxSize = 800;
    
    public void addTweet(Tweet tweet) {
        recentTweets.offer(tweet);
        if (recentTweets.size() > maxSize) {
            recentTweets.poll();  // Remove oldest
        }
    }
    
    public List<Tweet> getRecentTweets() {
        // Return in reverse chronological order
        List<Tweet> tweets = new ArrayList<>(recentTweets);
        tweets.sort((a, b) -> Long.compare(b.getTimestamp(), a.getTimestamp()));
        return tweets;
    }
}
```

---

# 10. FRONTEND (REACT FOR BACKEND ENGINEERS)

## 🎯 React for Backend-Heavy Engineers

### Priority: MEDIUM | Depth: Strong enough to talk competently

### Core Concepts

**1. Component Lifecycle (Class vs Functional)**
```javascript
// Class component
class UserProfile extends React.Component {
    componentDidMount() {
        this.fetchUserData();
    }
    
    componentDidUpdate(prevProps) {
        if (prevProps.userId !== this.props.userId) {
            this.fetchUserData();
        }
    }
    
    render() {
        return <div>{this.props.user.name}</div>;
    }
}

// Functional component with hooks
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    
    useEffect(() => {
        fetchUserData(userId).then(setUser);
    }, [userId]);  // Re-run when userId changes
    
    return <div>{user?.name}</div>;
}
```

**2. State Management**
```javascript
// Local state
const [count, setCount] = useState(0);

// Global state with Context
const UserContext = React.createContext();

function App() {
    const [user, setUser] = useState(null);
    
    return (
        <UserContext.Provider value={{ user, setUser }}>
            <Dashboard />
        </UserContext.Provider>
    );
}

function Dashboard() {
    const { user } = useContext(UserContext);
    return <div>Hello {user.name}</div>;
}
```

**3. Performance Optimization**
```javascript
// Memoize expensive calculations
const expensiveResult = useMemo(() => {
    return computeHeavyThing(data);
}, [data]);

// Memoize callbacks to prevent re-renders
const handleClick = useCallback(() => {
    console.log('Clicked');
}, []);

// Prevent unnecessary re-renders
const MyComponent = React.memo(({ data }) => {
    return <div>{data}</div>;
});
```

**4. API Integration**
```javascript
// Custom hook for data fetching
function useFetch(url) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    
    useEffect(() => {
        fetch(url)
            .then(res => res.json())
            .then(setData)
            .catch(setError)
            .finally(() => setLoading(false));
    }, [url]);
    
    return { data, loading, error };
}

// Usage in component
function UserList() {
    const { data, loading, error } = useFetch('/api/users');
    
    if (loading) return <Spinner />;
    if (error) return <Error message={error.message} />;
    return <ul>{data.map(user => <li key={user.id}>{user.name}</li>)}</ul>;
}
```

**Common Interview Questions:**

**Q: How does React's virtual DOM work?**
```javascript
// Virtual DOM is a lightweight JS representation
const vdom = {
    type: 'div',
    props: { className: 'container' },
    children: [
        { type: 'h1', props: {}, children: ['Hello'] }
    ]
};

// Reconciliation process:
// 1. Render virtual DOM
// 2. Diff with previous virtual DOM
// 3. Calculate minimal DOM operations
// 4. Apply to real DOM in batches
```

**Q: What are keys in React and why important?**
```javascript
// Keys help React identify items that changed
function List({ items }) {
    return (
        <ul>
            {items.map(item => (
                // ❌ Don't use index as key (causes bugs with reordering)
                <li key={item.id}>{item.name}</li>
                
                // ✅ Use stable, unique ID
            ))}
        </ul>
    );
}
```

---

# 11. API DESIGN & PROTOCOLS

## 🌐 REST vs GraphQL vs gRPC

### Priority: HIGH | Depth: Strong

### Comparison Matrix

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Protocol** | HTTP/1.1 | HTTP/1.1 or 2 | HTTP/2 |
| **Data format** | JSON (or XML) | JSON | Protobuf (binary) |
| **Payload size** | Medium | Large (over-fetching) | Small (binary) |
| **Performance** | Good | Poor (over-fetching) | Excellent |
| **Caching** | Native HTTP | Complex | Not supported |
| **Tooling** | Excellent | Good (GraphiQL) | Moderate |
| **Use case** | Public APIs | Mobile, complex UIs | Internal services |

### REST Best Practices

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    // GET /api/v1/users?page=2&size=20&sort=name
    @GetMapping
    public Page<User> getUsers(@PageableDefault(size = 20) Pageable pageable) {
        return userService.findAll(pageable);
    }
    
    // GET /api/v1/users/123
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    // POST /api/v1/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
    
    // PUT /api/v1/users/123 (full update)
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @Valid @RequestBody User user) {
        return userService.update(id, user);
    }
    
    // PATCH /api/v1/users/123 (partial update)
    @PatchMapping("/{id}")
    public User patchUser(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
        return userService.patch(id, updates);
    }
    
    // DELETE /api/v1/users/123
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### GraphQL with Spring Boot

```java
@Controller
public class UserGraphQLController {
    
    @QueryMapping
    public User user(@Argument Long id) {
        return userService.findById(id);
    }
    
    @QueryMapping
    public List<User> users(@Argument int page, @Argument int size) {
        return userService.findAll(PageRequest.of(page, size));
    }
    
    @MutationMapping
    public User createUser(@Argument String name, @Argument String email) {
        return userService.create(new CreateUserRequest(name, email));
    }
    
    // Resolve nested fields efficiently (DataLoader prevents N+1)
    @BatchMapping
    public CompletableFuture<Map<User, List<Order>>> orders(List<User> users) {
        return CompletableFuture.supplyAsync(() ->
            orderService.findByUsers(users).stream()
                .collect(Collectors.groupingBy(Order::getUser))
        );
    }
}
```

```graphql
# GraphQL schema
type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
    createdAt: DateTime!
}

type Query {
    user(id: ID!): User
    users(page: Int = 1, size: Int = 20): [User!]!
}

type Mutation {
    createUser(name: String!, email: String!): User!
    updateUser(id: ID!, name: String, email: String): User!
}
```

### gRPC with Protobuf

```protobuf
// user.proto
syntax = "proto3";

service UserService {
    rpc GetUser (GetUserRequest) returns (User) {}
    rpc ListUsers (ListUsersRequest) returns (stream User) {}
    rpc CreateUser (CreateUserRequest) returns (User) {}
}

message GetUserRequest {
    int64 id = 1;
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.Timestamp created_at = 4;
}

message ListUsersRequest {
    int32 page = 1;
    int32 size = 2;
}
```

```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void getUser(GetUserRequest request, StreamObserver<User> responseObserver) {
        UserDto user = userService.findById(request.getId());
        User response = User.newBuilder()
            .setId(user.getId())
            .setName(user.getName())
            .setEmail(user.getEmail())
            .build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    
    @Override
    public void listUsers(ListUsersRequest request, StreamObserver<User> responseObserver) {
        Page<UserDto> users = userService.findAll(PageRequest.of(request.getPage(), request.getSize()));
        for (UserDto user : users.getContent()) {
            responseObserver.onNext(convert(user));
        }
        responseObserver.onCompleted();
    }
}
```

---

# 12. TESTING PYRAMID & STRATEGIES

## 📊 Testing Deep Dive

### Priority: HIGH | Depth: Strong

### Testing Pyramid

```
      /\
     /  \  E2E (10%)
    /----\ 
   /      \ Integration (30%)
  /--------\
 /          \ Unit (60%)
/____________\
```

### Unit Testing (JUnit 5 + Mockito)

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {
    
    @Mock
    private PaymentRepository paymentRepository;
    
    @Mock
    private LedgerService ledgerService;
    
    @InjectMocks
    private PaymentService paymentService;
    
    @Test
    void shouldProcessPaymentSuccessfully() {
        // Given
        PaymentRequest request = PaymentRequest.builder()
            .idempotencyKey("key-123")
            .amount(BigDecimal.valueOf(100))
            .payerId(1L)
            .payeeId(2L)
            .build();
        
        when(paymentRepository.findByIdempotencyKey("key-123")).thenReturn(Optional.empty());
        when(ledgerService.debit(1L, BigDecimal.valueOf(100))).thenReturn(true);
        when(ledgerService.credit(2L, BigDecimal.valueOf(100))).thenReturn(true);
        
        // When
        PaymentResult result = paymentService.processPayment(request);
        
        // Then
        assertThat(result.isSuccess()).isTrue();
        verify(paymentRepository).save(any(Payment.class));
        verify(eventPublisher).publish(any(PaymentCompleted.class));
    }
    
    @Test
    void shouldHandleIdempotency() {
        // Given
        PaymentRequest request = PaymentRequest.builder()
            .idempotencyKey("key-123")
            .build();
        
        Payment existingPayment = Payment.success(request);
        when(paymentRepository.findByIdempotencyKey("key-123"))
            .thenReturn(Optional.of(existingPayment));
        
        // When
        PaymentResult result = paymentService.processPayment(request);
        
        // Then
        assertThat(result.isSuccess()).isTrue();
        verify(ledgerService, never()).debit(any(), any());
    }
}
```

### Integration Testing (Testcontainers)

```java
@Testcontainers
@SpringBootTest
class PaymentIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"));
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldProcessPaymentEndToEnd() {
        // Given
        PaymentRequest request = new PaymentRequest("key-123", 1L, 2L, BigDecimal.valueOf(100));
        
        // When
        ResponseEntity<PaymentResult> response = restTemplate.postForEntity(
            "/api/payments", request, PaymentResult.class
        );
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().isSuccess()).isTrue();
        
        // Verify Kafka event
        KafkaTestUtils.getRecords(kafka, "payment-completed", 1);
        
        // Verify database state
        Payment payment = paymentRepository.findByIdempotencyKey("key-123");
        assertThat(payment.getStatus()).isEqualTo(PaymentStatus.SUCCESS);
    }
}
```

### Contract Testing (Pact)

```java
// Consumer (Payment Service)
@RunWith(PactConsumerTest.class)
public class PaymentServiceConsumerTest {
    
    @Pact(consumer = "PaymentService", provider = "LedgerService")
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        return builder
            .given("User has sufficient funds")
            .uponReceiving("Debit request")
                .path("/api/ledger/debit")
                .method("POST")
                .body("{\"userId\":1,\"amount\":100}")
                .willRespondWith()
                .status(200)
                .body("{\"success\":true}")
            .toPact();
    }
    
    @Test
    @PactVerification("LedgerService")
    public void testDebit() {
        PaymentService service = new PaymentService("http://localhost:8080");
        boolean result = service.debit(1L, BigDecimal.valueOf(100));
        assertThat(result).isTrue();
    }
}
```

```java
// Provider (Ledger Service)
@Provider("LedgerService")
@PactBroker
public class LedgerServiceProviderTest {
    
    @TestTemplate
    @PactVerification
    public void verifyPacts(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    @State("User has sufficient funds")
    public void setupSufficientFunds() {
        // Setup test data
        ledgerService.createAccount(1L, BigDecimal.valueOf(1000));
    }
}
```

### Performance Testing (JMeter)

```xml
<!-- JMeter Test Plan -->
<TestPlan>
    <ThreadGroup>
        <stringProp name="ThreadGroup.num_threads">1000</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <stringProp name="ThreadGroup.duration">300</stringProp>
        
        <HTTPSamplerProxy>
            <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
            <stringProp name="HTTPSampler.port">443</stringProp>
            <stringProp name="HTTPSampler.protocol">https</stringProp>
            <stringProp name="HTTPSampler.path">/api/payments</stringProp>
            <stringProp name="HTTPSampler.method">POST</stringProp>
            
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                <collectionProp name="Arguments.arguments">
                    <elementProp elementType="HTTPArgument">
                        <stringProp name="Argument.value">
                            {"userId":1,"amount":100}
                        </stringProp>
                    </elementProp>
                </collectionProp>
            </elementProp>
        </HTTPSamplerProxy>
    </ThreadGroup>
    
    <ResultCollector>
        <stringProp name="filename">results.jtl</stringProp>
        <boolProp name="ResultCollector.error_logging">true</boolProp>
    </ResultCollector>
</TestPlan>
```

---

# 19. BEHAVIORAL & LEADERSHIP

## 🎯 Behavioral Questions Framework

### STAR Method

```
S - Situation (context, background)
T - Task (responsibility, goal)
A - Action (specific steps, decisions)
R - Result (outcome, metrics, learnings)
```

### Common Questions & High-Quality Answers

**Q1: Tell me about a time you had to make a difficult technical decision**

**Situation:** We were migrating from monolith to microservices. Our payment processing needed distributed transactions.

**Task:** Choose between adopting a distributed transaction coordinator (like Atomikos) or implementing SAGA pattern.

**Action:** 
1. Benchmarked both approaches: 2PC added 50ms latency, SAGA added 200ms (compensation)
2. Analyzed business requirements: 2PC provided stronger consistency but blocked on failures
3. Discussed with stakeholders: Finance wanted strong consistency, Product wanted high availability
4. Proposed hybrid: Use 2PC for critical path (payment capture), SAGA for compensation (refunds)
5. Built POC to validate performance assumptions

**Result:** 
- 2PC used for initial payment (99.9% success, 50ms)
- SAGA for rare failures (0.1%, 200ms)
- Achieved both consistency (no double charges) and availability (99.99%)
- Pattern adopted across 5 other services

---

**Q2: How do you handle production incidents?**

**Answer Framework:**
```
1. Detection & Triage (2 min)
   - Check dashboard (Grafana)
   - Identify blast radius
   - Determine severity (P0/P1/P2)

2. Mitigation (5 min)
   - Rollback recent deployment
   - Scale up resources
   - Circuit break downstream
   - Failover to DR region

3. Communication (ongoing)
   - Update status page
   - Notify stakeholders every 30 min
   - Document in incident Slack channel

4. Root Cause Analysis (post-incident)
   - 5 Whys technique
   - Timeline reconstruction
   - Blameless postmortem

5. Prevention
   - Add monitoring/alerting
   - Update runbooks
   - Chaos engineering experiments
```

**Real Example:**
"Last quarter, we had a P0 incident where payment service latency spiked to 5 seconds. I:

1. **Detected**: Our SLO alert fired (p99 > 500ms for 2 minutes)

2. **Mitigated**: 
   - Checked K8s dashboard: CPU normal, but GC pauses high
   - Rolled back last deployment (JVM arg change)
   - Latency dropped to 50ms within 3 minutes

3. **Communicated**: 
   - Updated status page within 5 minutes
   - Posted updates every 15 minutes to internal Slack

4. **Root cause**: 
   - Why? GC pauses of 2 seconds
   - Why? Switched from G1GC to ZGC for lower latency
   - Why? ZGC had 2% CPU overhead but caused more frequent GC
   - Why? Heap fragmentation from large objects
   - Why? Not tested on production traffic pattern

5. **Prevention**:
   - Added GC pause alerting
   - Created performance regression suite
   - Mandatory load testing for JVM changes
   - Results: Zero similar incidents in 6 months"

---

**Q3: How do you mentor junior engineers?**

**My Approach:**
```yaml
Onboarding (first 30 days):
  - Pair programming sessions (2x week)
  - Code review guide (share patterns)
  - Architecture walkthroughs
  - Assign small, well-scoped tasks

Skill Development (first 90 days):
  - Weekly 1-on-1s for feedback
  - "Stretch assignments" slightly beyond comfort
  - Encourage asking "why" in design reviews
  - Rotate responsibility for on-call

Growth to Senior (3-6 months):
  - Delegate ownership of small features
  - Include in customer meetings
  - Have them lead a design review
  - Encourage knowledge sharing (brown bags)

Success Metrics:
  - Junior becomes autonomous on-call (3 months)
  - Authoring design docs (6 months)
  - Mentoring new hires (12 months)
```

**Real Example:**
"After mentoring a junior dev, they initially struggled with system design. I:

1. **Assessed gaps**: They understood algorithms but not tradeoffs
2. **Created learning path**: 
   - Week 1-2: Read 'Designing Data-Intensive Applications' (Chapter 5-8)
   - Week 3-4: Review past design docs (I annotated my thought process)
   - Week 5-6: Co-design a new feature (notification service)
3. **Hands-on coaching**:
   - Daily 15-min design sessions
   - 'Think out loud' while I solved problems
   - Recorded and reviewed design reviews
4. **Result**: 
   - They now lead design reviews
   - Promoted to mid-level in 8 months
   - Currently mentoring new grads themselves"

---

# 20. MOCK INTERVIEW SIMULATOR

## Coding Round (45 minutes)

**Problem: Design a Rate Limiter**

**Question:**
```
Implement a rate limiter that:
- Allows max N requests per user per window of T seconds
- Should work in distributed environment
- Should be efficient (O(1) per check)
- Should be thread-safe
```

**Expected Senior-Level Answer:**

```java
public class DistributedRateLimiter {
    private final RedisTemplate<String, String> redis;
    private final int maxRequests;
    private final long windowSeconds;
    
    public DistributedRateLimiter(RedisTemplate<String, String> redis, 
                                  int maxRequests, 
                                  long windowSeconds) {
        this.redis = redis;
        this.maxRequests = maxRequests;
        this.windowSeconds = windowSeconds;
    }
    
    public boolean allow(String userId) {
        String key = "rate_limit:" + userId;
        long now = System.currentTimeMillis();
        long windowStart = now - (windowSeconds * 1000);
        
        // Lua script for atomic operation
        String luaScript = 
            "redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])\n" +
            "local current = redis.call('ZCARD', KEYS[1])\n" +
            "if current < tonumber(ARGV[2]) then\n" +
            "    redis.call('ZADD', KEYS[1], ARGV[3], ARGV[3])\n" +
            "    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[4]))\n" +
            "    return 1\n" +
            "else\n" +
            "    return 0\n" +
            "end";
        
        Long result = redis.execute(
            new DefaultRedisScript<>(luaScript, Long.class),
            Arrays.asList(key),
            String.valueOf(windowStart),
            String.valueOf(maxRequests),
            String.valueOf(now),
            String.valueOf(windowSeconds)
        );
        
        return result == 1;
    }
}
```

**Evaluation Criteria:**
- ✅ Correct algorithm (sliding window log)
- ✅ Distributed (Redis)
- ✅ Atomic (Lua script)
- ✅ Efficient (O(1) after cleanup)
- ✅ Handles edge cases (clock skew, TTL)
- ✅ Thread-safe (calls Redis atomically)

---

## System Design Round (60 minutes)

**Problem: Design Twitter Search**

**Requirements:**
- Search tweets by keyword
- 1B tweets/day, 500M searches/day
- Real-time (new tweets searchable within 1 minute)
- Relevance ranking (recent + popularity)

**Expected Senior-Level Answer Structure:**

**1. Requirements (5 min)**
```
- DAU: 100M, QPS search: 500M/86400 ≈ 6000 reads/sec
- QPS write: 1B tweets/86400 ≈ 11,500 writes/sec
- Latency: P99 < 500ms
- Data: 1B tweets * 1KB = 1TB new data/day
```

**2. High-Level Design (5 min)**
```
[Client] → [LB] → [Search API] → [Search Service]
                    ↓                ↓
                [Cache]          [Indexer]
                                    ↓
[User] → [Tweet Service] → [Queue] → [Indexer] → [Elasticsearch]
```

**3. Deep Dive - Indexing Pipeline (15 min)**
```java
// Real-time indexing
@Component
public class TweetIndexer {
    @KafkaListener(topics = "tweets")
    public void indexTweet(TweetEvent event) {
        // 1. Clean text (remove mentions, URLs)
        String text = cleanText(event.getText());
        
        // 2. Tokenize & stem
        List<String> tokens = tokenizer.tokenize(text);
        
        // 3. Build inverted index
        for (String token : tokens) {
            elasticsearch.index(
                "tweets", 
                event.getId(), 
                Map.of("text", text, "token", token, "timestamp", event.getTimestamp())
            );
        }
    }
}
```

**4. Deep Dive - Search Query (15 min)**
```java
// Search execution
public List<Tweet> search(String query, int limit) {
    // 1. Parse query
    Query parsed = queryParser.parse(query);  // "Java Spring" -> keywords: ["java", "spring"]
    
    // 2. Early termination for popular terms
    if (isPopularKeyword(parsed)) {
        // Only search recent tweets (last hour)
        parsed.setTimeFilter(Instant.now().minus(1, ChronoUnit.HOURS));
    }
    
    // 3. Execute search
    SearchResponse response = elasticsearch.search(parsed, limit);
    
    // 4. Relevance ranking (BM25 + recency boost)
    List<Tweet> results = response.getHits().stream()
        .map(hit -> {
            double score = hit.getScore() 
                * recencyBoost(hit.getTimestamp())  // Boost recent tweets
                * engagementBoost(hit.getRetweets());  // Boost popular
            hit.setScore(score);
            return hit;
        })
        .sorted((a,b) -> Double.compare(b.getScore(), a.getScore()))
        .collect(Collectors.toList());
    
    // 5. Cache for 1 minute
    cache.set(query, results, Duration.ofMinutes(1));
    
    return results;
}
```

**5. Tradeoffs (10 min)**

| Decision | Option A | Option B | Choice | Why |
|----------|----------|----------|--------|-----|
| **Search engine** | Elasticsearch | Solr | Elasticsearch | Better real-time, easier scaling |
| **Indexing** | Sync (write to ES) | Async (queue) | Async | Decouples, handles spikes |
| **Caching** | Redis | Memcached | Redis | Data structures (sorted sets) |
| **Sharding** | By keyword hash | By timestamp | Keyword hash | Even distribution |

**6. Bottlenecks & Solutions (10 min)**

**Bottleneck 1: Hot keywords (celebrity names)**
- Solution: Cache for 1 minute, rate limit high-frequency queries

**Bottleneck 2: Indexing lag during spikes**
- Solution: Auto-scale indexers with KEDA based on queue length

**Bottleneck 3: Relevance quality**
- Solution: A/B test ranking algorithms, use click-through data for learning

---

## Technical Deep Dive (30 minutes)

**Question:** "Explain how Spring's `@Transactional` works internally"

**Expected Senior Answer:**

```java
// 1. Proxy creation
@Configuration
@EnableTransactionManagement
public class AppConfig {
    // Spring creates a proxy (CGLIB or JDK)
    // Proxy wraps calls with transaction logic
}

// 2. Transaction interceptor
public class TransactionInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status = null;
        try {
            // Start transaction
            status = transactionManager.getTransaction(new DefaultTransactionAttribute());
            
            // Call actual method
            Object result = invocation.proceed();
            
            // Commit
            transactionManager.commit(status);
            return result;
            
        } catch (Exception ex) {
            // Rollback on runtime exception
            if (status != null) {
                transactionManager.rollback(status);
            }
            throw ex;
        }
    }
}

// 3. Transaction propagation
public Propagation {
    REQUIRED:     Join if exists, else create new
    REQUIRES_NEW: Always suspend current, create new
    MANDATORY:    Fail if no existing transaction
    NEVER:        Fail if transaction exists
    SUPPORTS:     Join if exists, else run non-transactional
    NOT_SUPPORTED: Suspend if exists, run non-transactional
    NESTED:       Savepoint within existing transaction
}

// 4. Real-world failure: Self-invocation
@Service
public class UserService {
    @Transactional
    public void saveUser(User user) {
        userRepo.save(user);
        // This calls internal method - bypasses proxy!
        sendNotification(user);  // ❌ Runs without transaction
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(User user) {
        notificationService.send(user);
    }
}

// Fix: Self-inject
@Service
public class UserService {
    @Autowired
    private UserService self;  // Self-inject proxy
    
    @Transactional
    public void saveUser(User user) {
        userRepo.save(user);
        self.sendNotification(user);  // ✅ Uses proxy
    }
}
```

---

# 21. PERSONALIZED ANALYSIS & STUDY PLAN

## Based on Profile (12 YOE, Backend-heavy)

### Strengths (Leverage These)
✅ **Java & JVM**: Deep expertise (8+ years)
✅ **Spring Ecosystem**: Extensive production experience
✅ **Relational Databases**: Strong SQL, transactions
✅ **Microservices**: Built and operated at scale
✅ **Messaging**: Kafka, JMS in production
✅ **Testing**: TDD, JUnit, Mockito

### Weaknesses (Need Focus)
⚠️ **Frontend (React)**: Only 2 years, senior expects more
⚠️ **Cloud Native**: AWS/Azure (knows services but not internals)
⚠️ **System Design**: Needs structured approach (framework)
⚠️ **DSA**: Rusty, needs high-ROI patterns only

### Hidden Gaps (Critical!)

| Gap | Why It Matters | How to Fix |
|-----|----------------|-------------|
| **JVM GC tuning** | Failures in production | Deep dive G1/ZGC, GC log analysis |
| **Kubernetes operators** | Can't extend platform | Study controller-runtime, CRDs |
| **Service mesh (Istio)** | Future of microservices | Hands-on with traffic shifting |
| **Event sourcing** | Critical for audit | Build banking POC |
| **Distributed tracing** | Debugging microservices | Implement OpenTelemetry |
| **Observability SLOs** | Senior expects metrics | Define SLIs/SLOs for service |
| **Disaster recovery** | Interview questions | Multi-region active/passive |

### Overconfidence Traps
⚠️ **"I know Java"** - But can you explain JVM memory model?
⚠️ **"I use Spring"** - But do you know how `@Transactional` proxy works?
⚠️ **"I do microservices"** - But have you debugged distributed deadlocks?
⚠️ **"I use Kafka"** - But have you tuned exactly-once semantics?

---

## 📅 8-Week Study Plan

### Week 1-2: Java & JVM Mastery

**Weekdays (1 hour/day)**
- Day 1-2: G1GC internals (regions, SATB, remembered sets)
- Day 3-4: ZGC (colored pointers, load barriers)
- Day 5: JMM (happens-before, volatile, final fields)
- Weekend (4 hours): Build GC log analyzer

**Deliverables:**
- GC tuning cheat sheet
- Virtual threads POC
- ConcurrentHashMap implementation analysis

---

### Week 3-4: Spring Deep Dive

**Weekdays (1 hour/day)**
- Day 1-2: Transaction propagation (7 types, real scenarios)
- Day 3-4: WebFlux vs Virtual Threads benchmark
- Day 5: Spring Security OAuth2 resource server
- Weekend (4 hours): Build custom starter + auto-config

**Deliverables:**
- Transaction propagation decision tree
- WebFlux vs VT benchmark results
- Custom Spring Boot starter project

---

### Week 5-6: System Design

**Weekdays (1 hour/day)**
Choose one case study per day:
- Day 1: URL shortener (key generation, scaling)
- Day 2: Payment system (idempotency, 2PC vs SAGA)
- Day 3: Twitter timeline (fanout, hybrid)
- Day 4: Notification system (delivery guarantees)
- Day 5: Rate limiter (distributed, algorithms)

**Weekend (8 hours)**
- Mock design interview (record yourself)
- Whiteboard 2 designs under timer
- Review with peer

**Deliverables:**
- 5 design docs (format from Section 8)
- Video recordings of mock interviews
- Tradeoffs matrix for each pattern

---

### Week 7-8: Cloud & Kubernetes

**Weekdays (1 hour/day)**
- Day 1-2: EKS deep dive (etcd, scheduler, CNI)
- Day 3-4: Service mesh (Istio traffic shifting)
- Day 5: Observability (Prometheus, Loki, Tempo)
- Weekend (4 hours): Deploy app with GitOps (ArgoCD)

**Deliverables:**
- Multi-region K8s cluster (minikube + multi-node)
- Istio canary deployment pipeline
- SLO dashboard (Grafana)

---

## 🎯 Focus Strategy (80/20 Rule)

### Master These (80% of interviews)
1. **System Design** (40% weight)
   - CAP, PACELC
   - Caching strategies
   - Database sharding
   - Event-driven patterns

2. **Java Concurrency** (20% weight)
   - JMM, happens-before
   - ConcurrentHashMap
   - Virtual threads

3. **Spring Internals** (15% weight)
   - Transaction management
   - Proxy mechanisms
   - Auto-configuration

4. **Behavioral** (15% weight)
   - STAR framework
   - Incident management
   - Mentoring examples

### Know These (10%)
- Frontend React (talk competently)
- DevOps (basic K8s operations)
- GraphQL (when to use)

### Ignore These (0%)
- Legacy tech (Jasper, Drools - just mention awareness)
- Niche algorithms (unless interviewing at Google)
- Frontend framework wars (React vs Vue vs Angular)

---

## 📝 Last 3 Days Strategy

### Day -3: Review Cheat Sheets
- [ ] GC tuning flags (G1, ZGC, Parallel)
- [ ] Transaction isolation levels (MVCC, anomalies)
- [ ] System design templates (URL shortener, Twitter)
- [ ] Behavioral stories (STAR format - 5 examples)

### Day -2: Mock Interviews
- [ ] Coding: Rate limiter (45 min timed)
- [ ] System design: Payment system (60 min)
- [ ] Behavioral: "Tell me about a failure" (15 min)
- [ ] Record and review (find weak spots)

### Day -1: Light Review
- [ ] Browse through this handbook (1 hour)
- [ ] Practice answering out loud (no writing)
- [ ] Prepare questions for interviewer
- [ ] Rest, hydrate, early sleep

### Day Of Interview
**Morning:**
- [ ] Light review (30 min max)
- [ ] Eat well, stay hydrated
- [ ] Test setup (camera, mic, IDE)

**During Interview:**
- [ ] Take 5 seconds to think before answering
- [ ] Use "I would approach this by..." 
- [ ] Draw diagrams (Excalidraw or whiteboard)
- [ ] Ask clarifying questions
- [ ] Think out loud (interviewer can't read your mind)
- [ ] If stuck, discuss tradeoffs (what would you try?)

---

## 🎯 Final Checklist

### Technical Depth
- [ ] Can explain JVM GC algorithms (G1 vs ZGC)
- [ ] Can implement concurrent cache with ConcurrentHashMap
- [ ] Can design distributed rate limiter
- [ ] Can explain Spring `@Transactional` proxy mechanism
- [ ] Can design Twitter timeline (push vs pull)
- [ ] Can debug distributed transaction failure

### Communication
- [ ] Can explain complex concepts simply
- [ ] Can draw system diagrams during interview
- [ ] Can ask clarifying requirements questions
- [ ] Can discuss tradeoffs (not just solutions)
- [ ] Can structure answers (STAR, problem-solution)

### Behavioral
- [ ] Have 3 failure stories (what went wrong, how fixed)
- [ ] Have 2 leadership examples (mentoring, architecture)
- [ ] Have 1 conflict resolution story
- [ ] Can explain "most challenging technical problem"
- [ ] Have questions ready for interviewer

---

## 🚀 Success Mindset

**Senior Engineer vs Staff Engineer Difference:**

| Aspect | Senior | Staff |
|--------|--------|-------|
| **Scope** | Feature/team | Cross-team/org |
| **Time horizon** | Weeks/months | Quarters/year |
| **Impact** | Direct | Leverage (multiply teams) |
| **Decision making** | Technical | Technical + business tradeoffs |
| **Communication** | Within team | Across org + leadership |

**Staff Engineer Interviewers Look For:**
✅ "I've seen this pattern fail because..." (experience)
✅ "The tradeoff is X vs Y, and here's why I chose Y" (judgment)
✅ "We need to consider not just now but next year" (foresight)
✅ "The business impact of option A is..." (business acumen)

**You've Got This! 🎉**

*This handbook is your companion. Bookmark it, review it, and trust your 12 years of experience. You're not just memorizing answers - you're learning to think like a Staff Engineer.*

---

**Created for:** Senior Fullstack Engineer (12 YOE)
**Last Updated:** 24-04-2026
**Total Pages Equivalent:** ~150 pages of deep technical content

**Good luck with your interviews! 🚀**
