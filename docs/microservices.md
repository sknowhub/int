# Microservices Interview Guide

## 1. Overview
At Staff+ level, microservices knowledge goes far beyond creating REST endpoints with Spring Boot. You must demonstrate:
- **Architectural reasoning** – when to adopt microservices, when to avoid them, and how to decompose a monolith with clear bounded contexts.
- **Communication patterns** – synchronous request/response, asynchronous messaging, event-driven choreography, CQRS, and the trade-offs each introduces.
- **Resilience engineering** – circuit breakers, bulkheads, retries with jitter, timeouts, idempotency, distributed transactions/Sagas, and how to prevent cascading failures.
- **Operational excellence** – centralized logging, distributed tracing, health checks, monitoring, canary releases, feature flags, service mesh, API gateway design.
- **Organizational impact** – how microservices affect team topology, release cadence, ownership, and how you drive adoption of best practices across multiple squads.
- **Hidden pitfalls** – debugging distributed systems, data consistency, eventual consistency in UX, network latency optimization, cross-service query complexity.

Real systems: e-commerce platforms with hundreds of services, streaming platforms, financial transaction processors, SaaS multi-tenant backends decomposed into billing, auth, notification, core domain services.

## 2. Deep Knowledge Structure

### 2.1 Principles & Decomposition
- **Concept** – Independent deployability, scalability, resilience, alignment with business capabilities.
- **Domain-Driven Design (DDD)**:
  - **Bounded Contexts** – how to define a service boundary; shared kernel, customer-supplier, anti-corruption layer.
  - **Subdomains** (Core, Supporting, Generic) – prioritize effort on core.
  - **Event Storming** – discovering aggregates, domain events, commands, and boundaries collaboratively.
- **Decomposition Strategies**:
  - Decompose by business capability (e.g., Order Management, Payment, Inventory)
  - Decompose by subdomain (core vs supporting)
  - Decompose by transaction boundary (aggregate)
  - Strangler Fig pattern to extract services from monolith gradually.
- **Microtopics (Deep Dive)**:
  - How to identify service boundaries: look at data that changes together (aggregates), transactional consistency requirements, and team ownership (Conway’s law).
  - Anti-patterns: “entity services” (CRUD per entity) leading to anemic, chatty services; “nano-services” (too fine-grained, operational overhead).
  - How to avoid distributed monolith: enforce contracts, separate databases, use asynchronous integration.
- **Atomic Interview Questions**:
  - *How do you decide the right size for a microservice?* – bounded context, team size (two-pizza team), data ownership, independent deployability; not lines of code.
  - *How would you decompose an existing monolith?* – identify seams, start with bounded contexts, extract high-change/high-scalability modules first, apply Strangler Fig.
  - *What is the Anti-Corruption Layer pattern?* – an isolation layer that translates between two bounded contexts to prevent one model leaking into another; essential when integrating with legacy or third-party systems.
- **Real-world Usage** – an e-commerce platform split into Catalog, Cart, Checkout, Payment, Fulfillment, each with own DB and team.
- **Failure Scenarios** – splitting services too small causing chatty communication, latency buildup, and debugging nightmare; adopting microservices without strong DevOps culture (manual deployments, no monitoring) leads to chaos.
- **Tradeoffs** – complexity vs scalability, development speed vs operational overhead, consistency vs availability.

### 2.2 Communication Patterns
#### Synchronous vs Asynchronous
- **Concept** – request-reply (HTTP/gRPC) vs messaging (Kafka, RabbitMQ) vs events.
- **RESTful APIs** – resource-oriented, HATEOAS (maybe overkill), versioning strategies (URI, header, content negotiation), backward compatibility.
- **gRPC** – Protobuf, performance, streaming (unary, server, client, bidirectional), load balancing (need for proxy or client-side), strong typing.
- **Event-Driven Communication**:
  - Events vs Commands: commands expect an action, events state that something happened.
  - Choreography vs Orchestration: choreography (services react to events) decouples but logic scattered; orchestration (central saga coordinator) explicit but coupling to coordinator.
  - Event schema evolution: backward/forward compatibility, schema registry (Avro/Protobuf).
- **Microtopics**:
  - Synchronous anti-patterns: chain of calls creating deep latency and failure propagation; use async where possible.
  - Exactly-once delivery semantics (impossible, idempotency and deduplication required).
  - Idempotency keys: client-generated UUID, store in service alongside processing status, expire later.
  - Transactional Outbox: avoid dual writes (DB + message) by writing event to outbox table in same transaction, then a CDC process (Debezium) publishes to Kafka; ensures at-least-once delivery.
- **Atomic Questions**:
  - *When would you use gRPC over REST?* – inter-service communication in high-performance/low-latency environments, need for streaming, strong contracts, polyglot environments; gRPC requires HTTP/2 and careful load balancing.
  - *How do you handle temporary outages of an external service?* – circuit breaker + fallback (cache, default), retries with backoff and jitter, queue-based async for eventual processing.
  - *Explain the Outbox pattern and why it’s needed.* – to atomically update database and send a message without 2PC; the outbox table is polled or tailed, ensuring the message is sent only after the DB commit.
  - *How do you achieve idempotency across retries?* – use idempotency keys, store key → response mapping, handle race conditions with unique constraint or distributed lock.
- **Failure Scenarios** – message ordering issues when relying on Kafka partitions but repartitioning data; missing idempotency leading to duplicate charges; using only synchronous calls and causing cascading timeout failures.
- **Tradeoffs** – sync easier to reason about but fragile; async decoupled but more complex to monitor and debug.

#### API Gateway & Backend for Frontend (BFF)
- **Concept** – single entry point, routing, authentication, rate limiting, aggregation, protocol translation.
- **Gateways**: Spring Cloud Gateway (reactive), Kong, Envoy, AWS API Gateway.
- **BFF pattern**: separate API gateway per client type (web, mobile) to tailor responses.
- **Microtopics**:
  - Gateway as an anti-corruption layer for external consumers.
  - Service mesh (Istio, Linkerd) shifts cross-cutting concerns to infrastructure sidecar, reducing duplication in each service.
- **Atomic**: *Should the API gateway perform orchestration?* – only simple aggregation or transformation, but complex orchestration belongs in a dedicated service to avoid gateway becoming bottleneck/monolith.

### 2.3 Service Discovery & Configuration
- **Service Discovery** – client-side (Netflix Eureka + Ribbon) vs server-side (Kubernetes DNS, AWS Cloud Map, Consul + Envoy).
- **Configuration Management** – centralized config server (Spring Cloud Config, Consul KV), hot reload with `@RefreshScope`, Vault for secrets.
- **Microtopics**:
  - Kubernetes native: Service/Endpoints API, kube-proxy, headless services for service discovery.
  - Health checks and readiness probes for proper load balancing.
  - Trade-offs: external config server vs Kubernetes ConfigMap/Secrets; config server adds complexity but enables feature flags and dynamic refresh across non-K8s environments.
- **Atomic**: *How do you handle database migrations in a CI/CD pipeline without downtime?* – backward-compatible changes (expand-contract), apply schema changes first (new column, no not-null), deploy services that write to both old and new, backfill, then drop old.

### 2.4 Resilience & Fault Tolerance
- **Fundamental patterns**:
  - **Circuit Breaker** (Resilience4j): closed, open, half-open states; failure rate threshold, slow call threshold, wait duration; fallback methods.
  - **Retry** with backoff and jitter (exponential backoff, random jitter to prevent thundering herd).
  - **Timeout**: set per call, propagate deadline, avoid unbounded waiting.
  - **Bulkhead**: isolate resources per dependency (thread pools, semaphores) to prevent one slow service from starving others.
  - **Rate Limiter**: protect downstream services, token bucket algorithm.
- **Microtopics**:
  - Cascade failures: service A → B → C, C slow causes threads in B to block, B exhausts thread pool, A fails. Circuit breaker halts calls to B early.
  - Idempotency required for retries: must ensure same effect.
  - Graceful degradation: feature toggles to disable non-critical functionality under load.
- **Atomic Questions**:
  - *What is the difference between a circuit breaker and a retry?* – retry handles transient failures; circuit breaker prevents hammering a failing service and allows it to recover; they often work together but circuit breaker opens after retries exhausted.
  - *How would you design a system to handle a flash sale spike?* – use gate queue (Redis sorted set) + rate limiter, pre-warm caches, isolate checkout service with bulkheads, leverage event-driven processing after order placed.
- **Failure** – using infinite retries causing retry storms; not incorporating jitter leading to synchronized retry waves; ignoring half-open circuit allowing too many trial calls.
- **Interview Expectations** – you’ll be asked to diagram a resilient system for a given scenario, identifying all possible failure points and countermeasures.

### 2.5 Distributed Transactions & Sagas
- **Concept** – maintaining consistency across multiple services without 2PC.
- **Saga**: sequence of local transactions, each publishing events triggering next step; compensating transactions for rollback.
  - **Choreography**: each service listens to events and acts; compensating events published on failure.
  - **Orchestration**: a saga orchestrator coordinates steps via commands, handles compensation.
- **Microtopics**:
  - Idempotency and exactly-once processing of saga events essential.
  - Using transactional outbox ensures event published only after local DB commit, keeping state consistent.
  - Failure scenarios: a compensating transaction itself failing – need mechanisms like retry, mark for manual intervention (saga log).
  - Routing slips: payload with sequence of steps.
  - Temporal/Cadence for long-running sagas with durable execution.
- **Atomic Questions**:
  - *Compare orchestration vs choreography for sagas.* – orchestration: explicit workflow, easier to understand, but coupling to orchestrator; choreography: more decoupled but logic spread, complex to track.
  - *How do you handle a saga that fails and the compensation fails?* – persist saga state with pending compensation, alert, and manual resolution; possibly use crontab to retry.
- **Real-world** – order placement saga: create order → reserve inventory → charge payment → confirm; if payment fails, cancel inventory reservation.
- **Tradeoffs** – eventual consistency accepted; final user might see temporary inconsistency.

### 2.6 CQRS & Event Sourcing
- **CQRS** – separate read and write models; optimize each independently; often used with Event Sourcing.
- **Event Sourcing**: state changes as sequence of events; event store append-only; state reconstructed by replay.
- **Microtopics**:
  - Projections: build read model by handling events.
  - Snapshots to speed up replay.
  - Consistency: eventual consistency between write and read models.
  - Challenges: event schema evolution, upcasting, duplicate handling.
- **Atomic**: *When NOT to use CQRS/ES?* – simple domains, overhead not justified; lack of expertise in team; reporting needs can be served by separate read replicas.
- **Failure** – read model lag causing user to see stale data after command; must communicate or serve from write model for critical immediate needs.

### 2.7 Observability & Debugging
- **Three pillars**: Logging (structured, correlation IDs), Metrics (RED: Rate, Errors, Duration; USE; custom business), Tracing (distributed tracing with consistent trace context).
- **Correlation ID** – propagate via HTTP headers (`X-Request-ID`), MDC, message headers; enables log aggregation.
- **Distributed Tracing**: OpenTelemetry, Jaeger/Zipkin; spans and traces; propagate trace context via W3C Trace Context; sampling strategies (head-based, tail-based).
- **Health Checks & Readiness** – liveness vs readiness; detailed health indicators (DB, Kafka connectivity).
- **Dashboards**: Service-level SLOs, error budgets, latency percentiles.
- **Atomic**: *How to debug a slow request across 10 microservices?* – start with trace, find span with largest duration; correlate with logs by trace ID; check service metrics for that time; identify bottleneck (DB query, network, GC pause, external call).
- **Real-world** – missing correlation ID propagation across Kafka messages leading to broken trace, fix via instrumented producer/consumer.
- **Failure** – not sampling enough traces to debug rare issues; trace not propagated through async boundaries.

### 2.8 Testing Strategies
- **Test Pyramid** (unit, integration, component, contract, end-to-end) modified for microservices.
- **Contract Testing**: provider-side and consumer-driven contracts (Pact). Ensures changes don't break consumers.
- **Component/Service Tests**: test service in isolation with dependencies stubbed (wiremock, testcontainers for DB).
- **Integration Testing**: with real downstream services? can become fragile; prefer environment with service virtualization or use test containers for dependencies.
- **End-to-End**: minimal, smoke tests across critical paths; not many.
- **Fault Injection / Chaos Engineering**: to validate resilience patterns; tools like Chaos Monkey, Litmus.
- **Atomic**: *How do you ensure backward compatibility?* – consumer-driven contract tests, CI pipeline that runs provider tests against consumer contracts, versioning strategy.
- **Failure** – heavy reliance on end-to-end tests in microservices; flakiness and slow feedback; shift left.

### 2.9 Deployment & Operations
- **CI/CD pipelines** – independent per service, build, test, push image, deploy.
- **Release Strategies**:
  - Rolling update, blue-green, canary deployment, feature flags.
  - Canary analysis: analyze metrics (error rate, latency) versus baseline before full rollout.
- **Infrastructure**: container orchestration (Kubernetes), service mesh.
- **Atomic**: *How to deploy a breaking change safely?* – never break immediately; always use expand-contract: add new field/api version, support both, deprecate old after all consumers migrated, then remove.
- **Database migrations in microservices**: each service owns its schema; use flyway/liquibase; backward compatibility critical.

### 2.10 Security
- **Authentication & Authorization**: OAuth2/OIDC, JWT, mutual TLS, API keys.
- **Defense in depth**: secure internal communication (mTLS, service mesh encryption), always validate tokens even service-to-service.
- **Rate limiting** to prevent DDoS.
- **Secrets management**: vault, sealed secrets, cloud provider key management.
- **Atomic**: *How do you secure interservice communication?* – mTLS via service mesh (Istio, Linkerd) ensures both encryption and service identity; or manually with cert-manager; use short-lived tokens (JWT with customer identity) but service identity via mTLS.

## 3. Code & Examples

```yaml
# Resilience4j Circuit Breaker config (application.yml)
resilience4j.circuitbreaker:
  instances:
    paymentService:
      registerHealthIndicator: true
      slidingWindowSize: 10
      minimumNumberOfCalls: 5
      failureRateThreshold: 50
      waitDurationInOpenState: 10s
      permittedNumberOfCallsInHalfOpenState: 3
resilience4j.retry:
  instances:
    paymentService:
      maxAttempts: 3
      waitDuration: 500ms
      retryExceptions:
        - java.net.SocketTimeoutException
```

```java
// Using Resilience4j annotations in a service
@Service
public class OrderService {
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.charge(request);
    }
    PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
        // e.g., store for later processing, event fail
        return new PaymentResponse("PENDING");
    }
}

// Idempotency handling with a database table
@Transactional
public void processOrder(OrderCommand cmd) {
    // idempotencyKey unique constraint
    if (idempotencyStore.exists(cmd.getIdempotencyKey())) {
        return idempotencyStore.getResponse(cmd.getIdempotencyKey());
    }
    // business logic
    Order order = ...;
    orderRepository.save(order);
    // store response for future duplicate
    idempotencyStore.store(cmd.getIdempotencyKey(), order.getId());
    // publish event via outbox
    eventPublisher.publish(new OrderPlacedEvent(order));
}
```

```dockerfile
# Dockerfile for a Spring Boot microservice
FROM eclipse-temurin:21-jre-alpine
COPY target/service.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

```yaml
# Kubernetes Deployment (minimal)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: app
        image: order-service:1.2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
```

```protobuf
// gRPC service definition (order.proto)
syntax = "proto3";
service OrderService {
  rpc PlaceOrder (PlaceOrderRequest) returns (PlaceOrderResponse);
}
message PlaceOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}
message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
}
message PlaceOrderResponse {
  string order_id = 1;
  string status = 2;
}
```

## 4. Interview Intelligence

### Topic: Handling Distributed Transactions with Sagas

#### ❌ Mid-level Answer
“We use sagas to maintain consistency. Each service does its work and publishes an event; the next service reacts. If something fails, we trigger compensating actions to undo previous steps.”

#### ✅ Senior Answer
“We implement choreography-based sagas for ordering. After order service creates the order, it emits `OrderPlaced`. Inventory service reserves stock and emits `InventoryReserved` or `InventoryReservationFailed`. If failed, the order service listens for that and sets order status to `CANCELLED`, and perhaps a notification sends email. We ensure idempotency with keys. For state tracking, we store saga state in a table. We handle failures carefully: compensations must themselves be idempotent. We use the Transactional Outbox pattern to atomically update the local DB and publish the event, avoiding losing events.”

#### 🚀 Staff-level Answer
“We adopted orchestration-based sagas for our payment and fulfillment flow because the business logic was complex and required explicit error handling and retries. I evaluated choreography vs orchestration and chose an orchestrator due to the need for compensating actions ordering (e.g., uncapture payment before releasing inventory). We built a lightweight saga framework using a state machine (Spring State Machine or custom) that persists saga instance state. The orchestrator uses asynchronous request/response commands; each step is tracked with a unique saga ID, and a timeout/retry policy per step. For consistency, all service steps use the outbox pattern, so the command is stored in the database and later sent by Debezium to Kafka. If a service step fails and the compensating transaction also fails, the saga goes into a `COMPENSATION_FAILED` state and we alert on-call to manually resolve. We also implemented a saga log dashboard for operations. Additionally, we handle the problem of exactly-once command delivery: each service consuming a command uses an idempotent consumer (deduplication by message ID). For queries across services during saga, we use CQRS: the read model is updated by events after each step, ensuring that UI doesn’t see inconsistent state; but we also show `processing` status to the user. This design reduced order failure rate from 2% to 0.05% in peak load.”

### Topic: Decomposing a Monolith

#### ❌ Mid-level Answer
“We split the monolith into REST services around entities like User, Order, Product. Each team took one.”

#### ✅ Senior Answer
“I used Domain-Driven Design to identify bounded contexts: Customer Management, Order Processing, Catalog, etc. I applied the Strangler Fig pattern: for each new capability, we built a new microservice and redirected API Gateway to it. For existing functionality, we integrated via events and slowly migrated data. We prioritized extracting high-change and high-scalability modules first, retaining the core as a modular monolith until fully extracted.”

#### 🚀 Staff-level Answer
“In our monolith-to-microservices migration, we followed a value-driven approach. I collaborated with product and business to identify the most painful areas (time-to-market for new features, scaling bottlenecks). We started with event storming workshops across teams to build a shared understanding of domain boundaries. The first extraction was ‘Pricing’—a bounded context that had high change frequency and required A/B testing. We applied the Strangler Fig pattern: we built the new pricing service, replicated the relevant data via Change Data Capture from the monolith’s DB to its own store, and gradually redirected calls through an API Gateway using canary routing. To ensure consistency, we used the transactional outbox pattern in the monolith to emit events for the new service to consume (continuing data sync until final cutover). We also introduced consumer-driven contract testing early to avoid interface mismatches. Organizationally, we formed a dedicated team for pricing aligned with the service, reducing coordination overhead. The hardest part was the shared database; we had to enforce that no new joins across contexts were added, and eventually we split the schema. After 8 months, the pricing service handled 100% of traffic, and we removed the monolith module. We continued extracting other contexts step by step, always measuring lead time and deployment frequency improvements to justify investment.”

## 5. High ROI Questions
- What are the key principles of microservices? Explain bounded contexts, data ownership, independent deployability.
- How do you handle transactions across multiple services? Describe Saga patterns and compensation.
- Compare synchronous (REST/gRPC) and asynchronous (message) communication in microservices. When would you use each?
- How do you prevent cascading failures? Talk about circuit breakers, bulkheads, timeouts, retries, and backpressure.
- How would you design an idempotent service? What is an idempotency key and how do you persist it?
- Explain the Outbox pattern and how to reliably publish events in a transactional system.
- How do you manage configuration and secrets across dozens of services?
- What is CQRS and Event Sourcing, and what are their trade-offs?
- How do you ensure API backward compatibility? What versioning strategies do you use?
- How would you monitor and debug a distributed system? Discuss trace propagation, correlation IDs, centralized logging, metrics.
- What are the benefits and challenges of a service mesh?
- How would you test microservices? Discuss contract testing, component testing, and end-to-end trade-offs.
- Walk me through a zero-downtime deployment of a breaking database schema change in a microservice.

**Tricky Follow-ups:**
- “A circuit breaker opens and never closes again. What could be wrong?” (half-open permit count, fallback not resetting, persistent downstream failure.)
- “You have a saga where step 3 succeeds but step 4 fails, and the compensation for step 3 fails. How do you recover?”
- “How do you handle duplicate messages in an event-driven system?”
- “Your API gateway becomes a bottleneck and single point of failure. How do you mitigate?”

## 6. Cheat Sheet (Revision)
- **Definition**: loosely coupled, independently deployable services around business capabilities, with own data stores.
- **Decomposition**: bounded contexts, aggregate root, strangler fig.
- **Communication**: sync for commands needing immediate response; async/events for eventual consistency. gRPC for high-perf inter-service.
- **Resilience**: Circuit breaker (open/half-open/closed), retry with backoff + jitter, timeout, bulkhead, rate limiter.
- **Transactions**: Sagas (orchestration/choreography); compensating transactions; outbox pattern for atomic DB+message.
- **Idempotency**: key passed by client, stored with response, unique constraint; delete after retention.
- **Observability**: correlation ID in logs, traces (OpenTelemetry), metrics (RED). Health checks (liveness/readiness).
- **Testing**: consumer-driven contract tests (Pact), component tests (in-memory dep), minimal E2E.
- **Deployment**: CI/CD per service, blue-green/canary, feature flags, expand-contract for DB.
- **Security**: OAuth2/OIDC, mTLS between services, API gateway rate limiting, vault for secrets.
- **Pitfalls**: distributed monolith, premature adoption, over-chatty communication, lack of DevOps maturity.

---

### 🎤 Communication Training for Microservices
- **Use the “architecture mindset”**: Frame answers as decisions balancing forces: “I chose asynchronous messaging over synchronous REST for checkout because it decouples payment processing from order placement, allowing us to handle bursts without timeout failures. The trade-off was eventual consistency: users see ‘pending’ for a few seconds, which we managed with optimistic UI updates.”
- **Structure with patterns**: When describing a solution, name the patterns you used (Saga, Outbox, Circuit Breaker, Backend for Frontend) and explain why in that context.
- **Tell a story of failure and learning**: “We once had a cascading failure because we didn’t set timeouts on HTTP calls. After that, we implemented a global timeout configuration and enforced it via code review and client library wrappers.”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas** (likely): REST API design, Spring Boot, basic service decomposition, some resilience patterns (maybe Hystrix/Resilience4j), Docker/K8s basics.
- **Weak areas / Hidden gaps**:
  - **Event-driven architecture deep knowledge**: outbox pattern, idempotent consumers, event schema registry, exactly-once semantics handling. Many engineers know concepts but haven’t implemented them end-to-end.
  - **Saga implementation experience**: might not have built orchestration with compensating transaction failure recovery.
  - **Distributed tracing adoption**: you might have used logs correlation IDs but not end-to-end tracing via OpenTelemetry; may not have defined sampling strategies or trace-based alerting.
  - **Contract testing**: if not used, that’s a gap—know Pact or Spring Cloud Contract.
  - **Service mesh**: understanding of sidecar proxy, mTLS, traffic splitting; candidates often overlook it.
- **Overconfidence risks**:
  - Assuming microservices solve all scalability problems; be ready to discuss when a modular monolith is better.
  - Overlooking data management: sharing databases, lack of separate schemas; unaware of the challenges of data migration across services.
  - Believing that a simple API gateway is enough; Staff engineers must think about security, rate limiting, resiliency, and developer experience.
  - Neglecting organizational impact: not addressing team topologies, ownership models, and the cultural changes required.

Ready for next topic when you say "next".
