# Spring Interview Guide

## 1. Overview
At Staff+ level you are expected to **command the Spring ecosystem** not as a framework user but as a **platform architect**. You should know:
- **How Spring works internally** – bean lifecycle, context hierarchies, AOP proxying, transaction management
- **Why Spring Boot decisions were made** – auto-configuration, opinionated defaults, actuator, testing slices
- **Scaling designs** – how to structure a multi-module Spring project, handle multi-tenancy, migrate monoliths to microservices (Spring Cloud, gateway, config)
- **Failure modes and performance** – common misconfigurations, N+1 queries, bean creation bottlenecks, reactive pitfalls
- **Trade-offs** – Spring MVC vs WebFlux, JPA vs JDBC, declarative vs programmatic transactions, embedding a container vs external

Real systems: high-throughput REST/gRPC services, event-driven microservices (Kafka, RabbitMQ), cloud-native deployments on Kubernetes, complex transactional boundaries in banking/payments, reactive streaming pipelines.

## 2. Deep Knowledge Structure

### 2.1 Spring Core & IoC Container

#### Inversion of Control and Dependency Injection
- **Concept** – Container manages object creation and wiring  
- **Subtopics**: `BeanFactory` vs `ApplicationContext`, XML vs annotation vs Java config, `@Autowired` modes
- **Microtopics (Deep Dive)**
  - `ApplicationContext` hierarchy: parent-child contexts (e.g., Spring MVC’s `DispatcherServlet` context and root context), bean visibility, `@Primary`, `@Qualifier`
  - `@Autowired` resolution: `byType` with fallback to `byName` via `DefaultListableBeanFactory`
  - How `@Configuration` works – CGLIB subclassing to intercept `@Bean` method calls, ensuring singleton semantics
  - Injection types: field injection pitfalls (testing, immutability, circular references hidden), constructor injection preferred for mandatory dependencies
- **Atomic Topics (Interview-Level)**
  - *What are the differences between `BeanFactory` and `ApplicationContext`?* – not just “lazy vs eager”, but `ApplicationContext` adds `MessageSource`, event publishing, AOP, internationalisation, etc.
  - *How does Spring resolve circular dependencies?* – via three-level caching: singleton objects, early singleton objects (setter injection proxies), singleton factories; only works for singletons with setter injection, not constructor injection; why it’s a code smell.
  - *What happens if you have two beans of the same type and no `@Primary`/`@Qualifier`?* – `NoUniqueBeanDefinitionException`
- **Real-world Usage** – Defining service interfaces and multiple implementations (e.g., payment gateways) with `@Qualifier` or custom annotations
- **Failure Scenarios** – creating two bean definition with same name via scanning and XML; `@Configuration` accidentally not proxied (non-static inner class, final class) leading to multiple instances of a `@Bean` defined method
- **Tradeoffs** – field injection vs constructor injection: readability vs testability/implicit dependencies
- **Common Mistakes** – injecting prototype bean into singleton without a proxy (`@Scope("prototype")` + `proxyMode`), causing a single instance; using `ApplicationContext.getBean()` outside bootstrap/legacy code

#### Bean Scopes and Lifecycle
- **Concept** – Singleton, Prototype, Request, Session, WebSocket, custom scopes  
- **Microtopics**
  - Lifecycle callbacks: `@PostConstruct`, `@PreDestroy`, `InitializingBean`, `DisposableBean`, `BeanPostProcessor`, `BeanFactoryPostProcessor`
  - `BeanPostProcessor` hooks: postProcessBeforeInitialization, postProcessAfterInitialization – used for proxying and wrapping
  - `BeanFactoryPostProcessor` allows modifying bean definitions before creation (e.g., property placeholders)
- **Atomic Questions**
  - *When would you use `BeanFactoryPostProcessor` vs `BeanPostProcessor`?* – `BFPP` modifies bean metadata, `BPP` wraps bean instances; crucial for understanding Spring’s internal magic
  - *How does `@Scope("request")` work in a web context?* – proxy that delegates to `RequestAttributes`; without a web context, it fails
  - *How to force a bean to be recreated?* – `@RefreshScope` (Spring Cloud) or prototyping; custom scope with lifecycle management
- **Failure Scenarios** – memory leak via request/session scoped beans in non-web environment; prototype bean instances not managed post-creation (no destroy callback unless explicitly registered)

#### Spring Expression Language (SpEL) & Resource Abstraction
- **Concept** – Dynamic expression evaluation, `@Value`, resource loading  
- **Microtopics**
  - SpEL parser, compilation (Java 9+) for performance, accessing system properties, calling methods, referencing beans
  - `Resource` interface: `ClassPathResource`, `FileSystemResource`, `UrlResource`
- **Tradeoffs** – SpEL readability vs XML config, potential performance overhead when overused
- **Atomic** – *How to prevent SpEL injection?* – validate parameters, never allow user-controlled strings to be evaluated directly in expressions; use `SimpleEvaluationContext` which restricts privileges

### 2.2 Aspect-Oriented Programming (AOP)
- **Concept** – Cross-cutting concerns: transactions, security, logging, caching, retries  
- **Microtopics**
  - Proxy mechanisms: JDK dynamic proxy (interfaces only) vs CGLIB (subclassing)
  - Pointcut designators: `execution`, `within`, `@annotation`, `args`, etc.
  - Advice types: Before, AfterReturning, AfterThrowing, After (finally), Around
  - `@Transactional` deep dive: proxy self-invocation problem, visibility (`@Transactional` on package-private method ignored in CGLIB proxy), transaction propagation
- **Atomic Questions**
  - *Why doesn’t `@Transactional` work when calling a method within the same class?* – AOP proxy only intercepts external calls; internal calls bypass the proxy; workaround: self-injection or `AopContext.currentProxy()` with `exposeProxy=true`
  - *When to use `@Transactional(propagation = Propagation.REQUIRES_NEW)`?* – audit logging, independent transactions that must commit regardless of outer transaction; rollback/exception isolation
  - *How to order multiple aspects?* – `@Order` or `Ordered` interface
- **Real-world Failures** – service method with `@Transactional` that catches exception and doesn’t rethrow but transaction rollbacks because of unexpected `RuntimeException` post-commit (wrong rollback rules)
- **Tradeoffs** – AOP adds complexity; debugging proxy chains; performance overhead minimal but measurable in hot loops

### 2.3 Spring Boot & Auto-Configuration
- **Concept** – Opinionated defaults, fast setup, embedded server, production-ready features  
- **Microtopics**
  - `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
  - Auto-configuration mechanism: `spring.factories`, `@Conditional*` annotations (`OnClass`, `OnBean`, `OnProperty`), auto-configuration ordering (`@AutoConfigureBefore/After`)
  - Custom auto-configuration – registering via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Spring Boot 2.7+)
- **Atomic Questions**
  - *How does Spring Boot decide which auto-configurations to apply?* – classpath presence, property values, existing beans; evaluate `@Conditional` chain
  - *How to exclude a specific auto-configuration?* – `spring.autoconfigure.exclude` property or `@EnableAutoConfiguration(exclude=…)`
  - *What are the pitfalls of `@SpringBootTest`?* – loading full context unnecessarily slow in large projects; use `@DataJpaTest`, `@WebMvcTest` slices
- **Real-world** – multi-tenant config where auto-configuration per tenant profile
- **Failure** – two libraries providing competing beans, one using `@ConditionalOnMissingBean` creating unexpected winner; custom auto-config not loaded due to missing `imports` file or wrong module structure

#### Embedded Containers (Tomcat, Undertow, Netty)
- **Concept** – JAR deployment with embedded server  
- **Microtopics**
  - Container configuration: `WebServerFactoryCustomizer`, SSL, compression, connection limits
  - Thread pool: Tomcat’s NIO connector max threads, accept count; Undertow’s XNIO worker threads
  - Graceful shutdown: `server.shutdown=graceful`, health indicators during shutdown
- **Atomic** – *How to handle high-concurrency REST APIs?* – tune max connections, enable keep-alive, offload heavy work to async task executors, consider reactive stack
- **Failure** – graceful shutdown timeout too low, killing inflight requests; `-Xmx` too small causing OOM under load

### 2.4 Spring MVC & Web
- **Subtopics**: `DispatcherServlet`, handler mapping, request lifecycle, interceptors, filters, exception handling, content negotiation
- **Microtopics**
  - `@ExceptionHandler`, `@ControllerAdvice`, `ResponseEntityExceptionHandler`
  - Async request processing: `DeferredResult`, `Callable`, Servet 3.0 async
- **Atomic**
  - *Filter vs HandlerInterceptor?* – Filters operate outside Spring MVC, before servlet, can modify request/response; interceptors within Spring MVC, access to handler object, `ModelAndView`
  - *How does `@ResponseBody` work?* – `HttpMessageConverter` chain, `MappingJackson2HttpMessageConverter` for JSON
- **Real-world** – building APIs that return `application/stream+json` for large datasets using `StreamingResponseBody` or reactive `Flux`
- **Failure** – returning a large collection causing OOM; missing `@Valid` on `@RequestBody` leading to invalid data

### 2.5 Spring WebFlux & Reactive
- **Concept** – Non-blocking, reactive streams, Netty, Reactor  
- **Microtopics**
  - `Mono`/`Flux`, backpressure, `Schedulers`, concurrency model
  - Reactor Netty as default, event loop groups
  - Reactive repository support (MongoDB, R2DBC, Cassandra)
- **Atomic Questions**
  - *When to choose WebFlux over MVC?* – high concurrency, streaming, I/O-bound services; not for CPU-heavy tasks without careful scheduling; necessary skills: non-blocking from controller to database (full reactive stack)
  - *How to handle blocking operations in WebFlux?* – `subscribeOn(Schedulers.boundedElastic())` for legacy blocking calls
  - *What are common reactive pitfalls?* – accidentally blocking in a non-blocking thread, forgetting to subscribe, context not propagated
- **Failure** – using a blocking JDBC driver with R2DBC proxy, leading to thread starvation
- **Tradeoffs** – complexity in debugging/tracing vs throughput; reactor stack traces are hard to read

### 2.6 Data Access & Transactions
- **Subtopics**: JDBC template, Spring Data JPA, Hibernate, MyBatis, JOOQ
- **JPA/Hibernate internals** (since it’s so common):
  - Entity life cycle (transient, managed, detached, removed)
  - First-level cache (persistence context), `EntityManager` usage
  - Flush modes (AUTO/COMMIT), dirty checking, `@Version` for optimistic locking
  - Lazy loading & N+1 problem: `JOIN FETCH`, `@NamedEntityGraph`, `EntityGraph`
  - `@OneToMany`/`@ManyToOne` pitfalls: eager fetching, `FetchMode.SUBSELECT`, batch fetching (`hibernate.default_batch_fetch_size`)
- **Atomic Questions**
  - *How does `@Transactional` interact with JPA’s persistence context?* – same transaction scope = same EntityManager; detached entities after transaction commit
  - *Explain Hibernate’s `SessionFactory` vs `EntityManagerFactory`?* – Hibernate Session internally wraps JPA EntityManager
  - *What is the "read-only" optimization?* – `@Transactional(readOnly=true)` hints Hibernate to skip dirty checking, improve performance; with JDBC, allows driver optimizations
  - *How to solve N+1?* – batch fetching, join fetch in query, entity graphs; but trading off memory and single query complexity
- **Real-world Failures** – transaction spanning multiple remote calls (long transaction), holding database locks; `LazyInitializationException` in serialization due to closed session; using `OSIV` (Open Session In View) causing long transactions and connection starvation—should disable it in modern architectures
- **Tradeoffs** – Spring Data JPA for rapid development vs raw JDBC/JOOQ for complex queries and performance; automatic dirty checking vs explicit updates

- **Transaction Propagation** – `REQUIRED`, `REQUIRES_NEW`, `NESTED`, etc.; `@Transactional` inner method called from outer: behavior depends on proxy; `REQUIRES_NEW` suspends outer transaction, can lead to deadlock if same resource
- **Atomic** – *What happens if an inner `@Transactional(REQUIRES_NEW)` throws an exception and outer catches?* – inner rolls back, outer can continue; careful with rollback-only markers
- **Failure** – inner transaction marks outer as rollback-only, then outer tries to commit and throws `UnexpectedRollbackException`

### 2.7 Spring Security
- **Microtopics**
  - Filter chain, `SecurityContextHolder`, `Authentication`/`Principal`
  - OAuth2 Resource Server, Keycloak, JWT validation
  - Method security: `@PreAuthorize`, `@PostAuthorize`, `@Secured`, SpEL expressions; AOP proxy again
- **Atomic**
  - *How to prevent common vulnerabilities?* – CSRF protection (state-changing endpoints), `X-Content-Type-Options: nosniff`, CORS policy, content security policy, using `@PreAuthorize` for authorization not just authentication
  - *How does `SecurityContextHolder` strategy work?* – `MODE_THREADLOCAL` default, `MODE_INHERITABLETHREADLOCAL` for async threads, `MODE_GLOBAL` for special cases; must propagate context to child threads (use `DelegatingSecurityContextRunnable` etc.)
- **Failure** – forgetting to propagate security context to `@Async` methods, leading to anonymous access
- **Tradeoffs** – method security adds boilerplate but centralizes authorization; filter level security performs broad checks

### 2.8 Spring Cloud & Distributed Systems
- **Microtopics**
  - Service Discovery: Netflix Eureka vs Consul vs Kubernetes Service
  - Config Server: refresh, `@RefreshScope`, bus-based refresh, vault integration
  - Circuit Breaker: Resilience4j (the recommended replacement for Hystrix) – sliding window, timeouts, fallback
  - API Gateway: Spring Cloud Gateway (reactive), predicates, filters, rate limiting
  - Distributed tracing: Micrometer, Sleuth, Zipkin/Brave
- **Atomic Questions**
  - *How does `@RefreshScope` work?* – proxy that caches bean instances; on `RefreshScope.refresh()` it clears cache and next access re-resolves bean; requires `@ConfigurationProperties`
  - *What are the trade-offs of using a centralized config server?* – single point of failure (needs high availability), eventual consistency after refresh; alternatives: ConfigMap or mounted volumes in Kubernetes
  - *How to implement idempotency in Spring microservices?* – use request IDs, `@Idempotent` custom annotation + AOP, distributed cache (Redis) for request state
- **Real-world** – handling config drift, zero-downtime config updates without full restart
- **Failure** – gateway misconfigured with load balancer causing retry storms; circuit breaker half-open state allowing too many requests too soon

### 2.9 Testing
- **Subtopics**: `@SpringBootTest`, slicing (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`), Mockito integration, testcontainers
- **Microtopics**
  - Context caching – which configs are shared across tests; speed optimization
  - `@MockBean` vs `@SpyBean` – context refresh overhead, prefer constructor injection + pure mocks over `@MockBean` to avoid dirty context
  - Integration tests with real DB using testcontainers or H2 (Dangers of H2 – syntax differences)
- **Atomic** – *Why is `@MockBean` slow in large test suites?* – it dirties the context, forcing recreation for next test; use test slices or pure unit tests
- **Failure** – H2 compatibility: production on Postgres but test on H2 hiding SQL dialect issues; ignore migration scripts or custom functions

### 2.10 Observability & Actuator
- **Actuator endpoints** – health, metrics, env, loggers, thread dump, heap dump
- **Custom health indicators** – database, Kafka, external services, aggregation
- **Micrometer** – metrics abstraction, integration with Prometheus, Datadog, CloudWatch
- **Atomic** – *How to expose custom business metrics?* – `MeterRegistry.counter()`, `@Timed`, gauge, distribution summary; must then export to monitoring system

## 3. Code & Examples

```java
// --- Bean lifecycle and @Configuration proxy ---
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource ds) {
        // despite calling dataSource(), Spring ensures singleton DataSource is injected
        return new JdbcTemplate(ds);
    }
}

// --- @Transactional self-invocation problem and fix ---
@Service
public class OrderService {
    @Transactional
    public void process(Order order) {
        // ...
        this.internalProcess(order); // bypasses proxy -> @Transactional ignored
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void internalProcess(Order order) { ... }
}

// Solution: self-injection or AopContext
@Service
public class OrderServiceFixed {
    @Autowired
    private OrderServiceFixed self; // self-reference to proxy

    @Transactional
    public void process(Order order) {
        self.internalProcess(order); // via proxy
    }
}

// --- Custom health indicator ---
@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean ok = checkService();
        if (ok) return Health.up().withDetail("latency", "20ms").build();
        else return Health.down().withDetail("error", "timeout").build();
    }
}

// --- Reactive Router Function ---
@Bean
public RouterFunction<ServerResponse> route(OrderHandler handler) {
    return RouterFunctions
        .route(GET("/orders/{id}").and(accept(APPLICATION_JSON)), handler::getOrder)
        .andRoute(POST("/orders").and(contentType(APPLICATION_JSON)), handler::createOrder);
}
```

## 4. Interview Intelligence

### Topic: `@Transactional` Internals

#### ❌ Mid-level Answer
“`@Transactional` uses AOP proxy to begin/commit/rollback transaction. It works only for public methods.”

#### ✅ Senior Answer
“Spring’s `@Transactional` creates a proxy around the bean. The proxy intercepts calls and invokes `TransactionInterceptor` which obtains a `PlatformTransactionManager` to begin transaction, execute method, commit on success or rollback on `RuntimeException`/`Error`. The proxy model—JDK dynamic proxy or CGLIB—means internal calls within the same bean bypass the proxy, so transactional advice is not applied. I work around this by self-injection or restructuring the code. I also set `readOnly=true` for read operations to optimize JPA dirty checking and database hints.”

#### 🚀 Staff-level Answer
“`@Transactional` leverages AOP with a chain of `Advisor`s, the core being `BeanFactoryTransactionAttributeSourceAdvisor`. The proxy’s `invoke` method checks if a transaction attribute is present for the method. The interceptor uses `TransactionSynchronizationManager` to bind resources (Connection, EntityManager) to the current thread, ensuring multiple DAO calls within the same transaction share the same persistence context. I’ve debugged cases where `REQUIRES_NEW` caused deadlocks because the suspended outer transaction held a database lock that the inner needed; we resolved by reordering operations. Another subtlety: `@Transactional` on a `private` method silently ignored due to CGLIB proxy (cannot subclass private methods); we enforce `proxyTargetClass=true` and avoid private transactional. For performance, I configure transaction-specific `PlatformTransactionManager` if there are multiple data sources, using `@Transactional("txManager")`. We also disabled Open Session In View to prevent accidental long-lived transactions in web layer, and moved data aggregation to DTOs via explicit `JOIN FETCH` queries. In reactive stack, `@Transactional` doesn't work with WebFlux because it relies on thread-bound resources; instead, we use `TransactionOperator` in Reactor pipelines. For distributed transactions, I evaluate Saga pattern using Spring’s `@TransactionalEventListener` with phase `AFTER_COMMIT` to publish events that coordinate compensating actions.”

### Topic: Spring Boot vs Traditional Spring

#### ❌ Mid-level Answer
“Spring Boot simplifies configuration with auto-configuration and provides embedded server.”

#### ✅ Senior Answer
“Spring Boot removes boilerplate via `@EnableAutoConfiguration` which uses `@Conditional` to configure beans based on classpath and properties. The embedded container (Tomcat/Netty) makes deployment trivial. I use starter parent for dependency management, customize via `application.yml` property overrides. For production, I expose actuator endpoints for health/metrics and use `@Scheduled`/`@Async` with custom thread pools.”

#### 🚀 Staff-level Answer
“Spring Boot’s auto-configuration underpins our platform’s ‘paved road’ – we’ve built internal starters that encapsulate logging, security, metrics, and client libraries (e.g., for Kafka, Redis) with sane defaults across hundreds of services. I’ve contributed custom auto-configuration that detects the environment (cloud, on-prem) and sets up appropriate `DataSource`, vault config, or mock beans for local development. I know the ordering of auto-config classes matters: I use `@AutoConfigureBefore/After` when our starter must wrap or replace an auto-configured bean. To minimize context start time in integration tests, we use context hierarchy – a parent context with shared beans (embedded DB, Kafka) and child contexts per test slice; this reduces CPU usage significantly. I also profile startup time using `ApplicationRunner` timing and Spring’s `StartupStep` to detect bean initialization bottlenecks (e.g., loading a large schema via `data.sql`). For microservices, we enforce that all config is external (Config Server or K8s ConfigMap) with no hardcoded defaults; using `@ConfigurationProperties` with immutable records gives type-safe and refreshable configuration. We’ve also migrated sensitive configs to Vault with `spring-cloud-starter-vault-config` using Kubernetes auth, zero-trust.”

## 5. High ROI Questions
- Explain the Spring bean lifecycle and what `BeanPostProcessor` does. How is `@Transactional` implemented?
- Why does `@Transactional` sometimes not roll back? How to fix self-invocation?
- What’s the difference between `Propagation.REQUIRED` and `REQUIRES_NEW`? Real scenario where REQUIRES_NEW was wrong.
- How do you avoid N+1 queries in Spring Data JPA? Explain entity graph, batch fetching, and join fetching trade-offs.
- How would you design a multi-tenant Spring Boot application with per-tenant schema and routing?
- Spring WebFlux vs Spring MVC: decision criteria, performance pitfalls, when to use each.
- How does Spring Security integrate with JWT and OAuth2? How to propagate security context to async methods?
- How to handle distributed transactions in a microservices architecture using Spring?
- Explain custom auto-configuration in Spring Boot and how to ensure it loads after a default one.
- How would you reduce startup time of a large Spring Boot application?
- What are the downsides of `spring.jpa.open-in-view=true`? Have you disabled it? What challenges?

**Tricky Follow-ups:**
- “Your `@Transactional` method calls another `@Transactional` method in a different bean with `REQUIRES_NEW` and the outer fails after the inner succeeds. What happens?”
- “You have a prototype-scoped bean injected into a singleton – how to get a new instance each time?”
- “You see `DataIntegrityViolationException` intermittently. Could it be related to transaction isolation? How to diagnose?”
- “Explain `@Cacheable` proxy, cache eviction, and a scenario where it caused stale data in production.”

## 6. Cheat Sheet (Revision)
- **IoC**: `@Autowired` resolved by type then qualifier; `@Primary` indicates primary bean; `BeanFactory` core, `ApplicationContext` superset; bean scopes: singleton (default), prototype, request, session; circular dependency solved by early singleton reference caching (constructor injection fails).
- **AOP**: `@Transactional` relies on AOP proxy (JDK dynamic or CGLIB); self-invocation bypasses proxy → annotation ignored. Use `AopContext.currentProxy()` or self-injection.
- **Transactions**: `@Transactional(propagation=REQUIRES_NEW)` suspends outer; rollback on `RuntimeException`; `readOnly` optimization.
- **JPA**: N+1 fix with `JOIN FETCH`, `@EntityGraph`, batch size. Avoid OSIV (false in production). `LazyInitializationException` because no session; use DTOs.
- **Boot**: `@SpringBootApplication` = config + auto-config + component scan; auto-config via `@Conditional` classes; override with properties/exclusions.
- **WebFlux**: non-blocking, Reactor Netty; block calls on `boundedElastic` scheduler; `TransactionOperator` for reactive transactions; context propagation via `Hooks.enableAutomaticContextPropagation()` (Reactor).
- **Security**: `SecurityContextHolder` thread-local; propagate with `DelegatingSecurityContextExecutor`; method security with `@PreAuthorize`.
- **Testing**: Use slicing (`@WebMvcTest`, `@DataJpaTest`) for speed; avoid `@MockBean` excessive context recreation; testcontainers for real DB integration.
- **Actuator**: health, metrics (`micrometer`), custom indicators; expose only necessary endpoints.

---

### 🎤 Communication Training for Spring
- **Frame answers around platform thinking**: “We built internal starters to enforce that all services use the same circuit breaker configuration, centralized logging, and security defaults. This reduced configuration duplication and misconfigurations by 80% across 100+ services.”
- **Use the “what, how, why” structure**: When asked about a Spring feature, explain what it does, how it works internally (proxies, post-processors), and why it’s designed that way. For example: “`@Transactional` uses AOP proxies – this introduces the self-invocation limitation. We run into this in service refactoring and handle it by self-injection, but the root cause is mixing cross-cutting concerns with domain logic. So we often extract the transactional logic into a dedicated boundary service to keep domain services pure and testable.”
- **Highlight failure stories**: “We once had a production incident because a `@Transactional(REQUIRES_NEW)` method was called within a loop, creating too many connections and deadlocks. We fixed by restructuring into a batch job with a single transaction and savepoints.”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas** (likely): Spring Boot, MVC, Data JPA, transactional knowledge, security basics, integration patterns
- **Weak areas / Hidden gaps**:  
  - **Reactive & WebFlux** – deep understanding of threading model, debugging stack, reactor operator fusion, context leaks; if you haven’t shipped production WebFlux, that’s a gap.  
  - **Spring Cloud Gateway custom filters** – beyond simple rate limiting, how to implement content-based routing or request transformation.  
  - **AOT compilation / GraalVM native image** – Spring Boot 3+ support; understanding what can’t be used (e.g., reflection, `@Configuration` CGLIB proxies replaced by AOT)  
  - **Spring Batch** – partitioning, remote chunking for large-scale batch processing.  
  - **Integration with Kubernetes** – `spring-cloud-kubernetes` for discovery, config, pod health indicators, graceful shutdown signals.
- **Overconfidence risks**:  
  - Assuming `@Transactional` always works as expected; deep pitfalls around isolation levels, savepoints, propagation in complex flows.  
  - Thinking Hibernate is always the right choice; knowing when to drop down to JDBC or JOOQ for critical performance paths.  
  - Neglecting observability – metrics, distributed tracing integration with Micrometer/Sleuth. Interviewers will probe for production-monitoring mindset.

Proceed to the next topic when ready.
