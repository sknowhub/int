# System Design & System Engineering Interview Guide

## 1. Overview
At Staff+ level, the system design interview is the definitive signal. It’s not about reciting architectures — it’s about **structured problem-solving, deep trade-off analysis, and operationalizing a system at scale**. You must demonstrate:

- **System Design:** Requirements gathering, high-level architecture, data modeling, API design, scaling strategies, failure handling, and evolution.
- **System Engineering:** Non-functional requirements (availability, reliability, latency, throughput), capacity planning, performance modeling, deployment topology, monitoring, chaos engineering, cost optimization, and resilience engineering.

Together they prove you can **design a system and ensure it runs in production** at internet scale. Interviewers will look for breadth (covering all aspects) and depth (explaining why a specific database, replication strategy, or safety mechanism was chosen and what happens if it fails).

## 2. Deep Knowledge Structure

### 2.1 Problem-Solving Framework
A repeatable framework is essential. Use **PEDALS** or similar:

1. **P**roblem Clarification: functional & non-functional requirements, constraints, scale (users, requests, data).
2. **E**stimation & Capacity: back-of-envelope calculations (throughput, storage, memory, bandwidth).
3. **D**esign High-Level: core components, services, database choices, communication patterns.
4. **A**rchitecture Deep Dive: data model, API, detailed component responsibilities.
5. **L**everage Existing: discuss trade-offs (SQL vs NoSQL, sync vs async, monolith vs microservices).
6. **S**calability & Failure: bottlenecks, sharding, replication, caching, CDN, failure scenarios and mitigations.

### 2.2 Core Design Principles
- **Separation of Concerns**: presentation, business logic, data access; microservices bounded contexts.
- **Loose Coupling & High Cohesion**: minimize dependencies; well-defined interfaces.
- **Fault Tolerance**: no single point of failure; graceful degradation; retry, circuit breaker, bulkhead.
- **Scaling**: horizontal scaling (scale out) vs vertical; stateless over stateful; data partitioning.
- **Caching**: client-side, CDN, reverse proxy, application, database query, object cache; invalidation strategies (TTL, write-through, write-behind).
- **Data Management**: ACID vs BASE; read replicas, sharding, eventual consistency, CQRS.
- **Security**: defense in depth; authentication, authorization, encryption, minimal privilege.

### 2.3 Key System Building Blocks

#### Load Balancing & Reverse Proxies
- **Algorithms**: round robin, least connections, consistent hashing (session affinity / sticky sessions).
- **Health checks** and circuit breakers at LB level.
- **Global Load Balancing**: DNS-based (Route 53, Traffic Manager), Anycast.
- **Trade-offs**: layer 4 (TCP) vs layer 7 (HTTP); L7 can route by URL, authenticate, rate limit.

#### Caching Layers
- **Browser/Client cache**: ETag, Cache-Control, Service Workers.
- **CDN**: edge caching for static/assets, dynamic content acceleration; cache invalidation (purge, versioned URLs).
- **Application cache**: Redis/Memcached; caching query results, objects, sessions.
- **Database caching**: internal buffer pool, integrated cache (e.g., MySQL query cache, though deprecated), materialized views.
- **Write policies**: cache-aside (lazy load), write-through, write-behind.
- **Consistency**: cache invalidation is hard; ensure data gets updated after DB write; use TTL to bound staleness.

#### Databases (covered in Databases guide, but usage in design)
- How to choose: relational for ACID, joins; NoSQL for schema flexibility, horizontal scaling, high write throughput.
- Read replicas for scale reads; leader-follower topology; multi-leader; synchronous vs asynchronous replication.
- Sharding: key-based (partition key), range, directory-based; cross-shard queries challenge.

#### Messaging (covered in Cloud & Messaging guide)
- When to use: decoupling services, backpressure, async processing, event-driven.
- Integration: as part of system design for task queues, event streams, log aggregation.

#### Service Communication
- Synchronous: REST (JSON over HTTP), gRPC (Protobuf, HTTP/2, streaming).
- Asynchronous: message queues, pub/sub, event streaming.
- Service mesh for resilient communication.

#### Data Processing Pipelines
- Batch vs real-time: MapReduce / Spark vs Kafka Streams / Flink.
- Lambda architecture: speed layer + batch layer + serving layer; Kappa architecture (stream only).
- Trade-offs: Complexity of two codebases vs. reprocessing ability.

### 2.4 Design Case Studies (10+)

For each, outline functional/non-func requirements, capacity estimates, high-level architecture, data model, deep dive into critical components, and scaling.

1. **Design a URL Shortener** – hash generation, redirect, analytics, scaling to billions.
2. **Design a Chat System** – one-on-one and group chat, message synchronization, online/offline, storage.
3. **Design a Ride-Hailing Service (Uber/Lyft)** – geo-indexing, matching, pricing, real-time tracking.
4. **Design a Social Media Feed (Twitter/Instagram)** – fanout on write vs fanout on read, timeline generation, celebrity problem.
5. **Design a Video Streaming Platform (YouTube)** – upload, transcoding, adaptive bitrate streaming, CDN.
6. **Design a Distributed File Storage (Dropbox/Google Drive)** – metadata, block storage, sync conflicts, sharing.
7. **Design a Payment System** – double-entry ledger, idempotency, reconciliation, fraud detection.
8. **Design a Rate Limiter** – algorithms (token bucket, sliding window), distributed enforcement.
9. **Design a Notification System** – multi-channel (push, email, SMS), delivery tracking, preference management.
10. **Design a Search Autocomplete System** – trie, frequency, real-time updates, sharding.
11. **Design a Hotel Booking System** – inventory/availability, pricing, concurrency (double booking prevention), search.
12. **Design a Collaborative Real-time Editor (Google Docs)** – operational transformation / CRDT, conflict resolution, version history.
13. **Design an Ad Click Aggregator** – high-throughput ingestion, deduplication, real-time and batch aggregation.
14. **Design a Distributed Message Queue** – durable storage, consumer groups, replication.
15. **Design a Metrics/Monitoring System** – time-series database, query layer, alerting engine.

### 2.5 System Engineering

#### Non-Functional Requirements Engineering
- **Availability**: uptime percentage, MTBF/MTTR, active-active vs active-passive, data durability.
- **Reliability**: fault tolerance, resilience patterns, circuit breaker, redundancy.
- **Scalability**: handling load growth; elasticity; data partitioning; statelessness.
- **Performance**: latency (P50, P99), throughput, capacity planning; load testing, profiling.
- **Maintainability**: modularity, observability, DevOps maturity.
- **Security & Compliance**: encryption at rest/transit, PCI-DSS, GDPR, HIPAA.
- **Cost**: total cost of ownership; design-to-cost.

#### Capacity Planning
- **Traffic estimates**: QPS, read/write ratio, data size growth.
- **Server resources**: CPU, memory, IOPS required per service; horizontal scaling limits.
- **Database capacity**: storage, IOPS, connection pools, shard size.
- **Network**: bandwidth, inter-AZ/region latency, data transfer cost.
- **Tools**: Load test (JMeter, K6), profiling, simulation.

#### Performance Modeling & Bottleneck Analysis
- Little’s Law for concurrency: L = λW.
- Identify bottlenecks: saturated resources (CPU, disk, net), lock contention, sequential processing, queue buildup.
- Amdahl’s Law for parallelization.
- Approach: monitor → profile → tune → test.
- Database query optimization, connection pooling, caching.

#### Resilience & Disaster Recovery
- **RTO/RPO**: Recovery Time Objective, Recovery Point Objective.
- **Backup**: full/incremental, offsite, point-in-time recovery.
- **Failover**: manual, automatic; DNS failover, database promotion.
- **Disaster Recovery Drills**: regular chaos experiments, game days.
- **Bulkhead**: isolate compute/storage for different components.
- **Graceful Degradation**: feature flags to disable non-critical functionality when stressed.

#### Chaos Engineering
- Principles: start with “steady state” hypothesis, vary real-world events, automate experiments.
- Tools: Chaos Mesh, Litmus, Gremlin.
- Examples: kill a pod, simulate latency, drain a node, corrupt disk; verify monitoring and auto-healing.

#### Deployment & Operations at Scale
- Blue-green, canary, rolling deployments (see DevOps guide).
- Auto-scaling: based on metrics (CPU, memory, custom); capacity buffer for spikes.
- Fleet management: immutable infrastructure, containers (Kubernetes).
- GitOps and Infrastructure as Code.

### 2.6 Failure Handling & Trade-off Patterns
- **No single point of failure**: replicate stateless services, use leader election for active-passive.
- **Graceful Handling of overload**: load shedding, throttling, backpressure, request debouncing.
- **Distributed consistency trade-offs**: better to be eventually consistent with idempotency than strongly consistent with reduced availability.
- **Data redundancy**: replication but risk of stale reads; decide acceptable staleness.
- **Operational simplicity vs performance**: sometimes a simple Postgres with read replicas outperforms a complex NoSQL cluster.

## 3. Diagrams & Examples (Textual)

```
# High-level architecture of a URL shortener
Client -> DNS -> CDN (optional) -> Reverse Proxy (Nginx) -> Shortening Service (stateless)
                                 -> Read Service (cached) -> Cache (Redis) -> DB (PostgreSQL)
Shortening Service generates unique ID (base62) using distributed ID generator (Snowflake)
Click tracking: Kafka message per redirect -> Aggregation -> OLAP for analytics
```

```java
// ID generator using Snowflake (simplified)
public class Snowflake {
    private final long epoch = 1609459200000L;
    private final long workerIdBits = 5L;
    private final long sequenceBits = 12L;
    // ... bit shifts
    public synchronized long nextId() { ... }
}
```

## 4. Interview Intelligence

### Topic: Designing a Scalable Chat System

#### ❌ Mid-level Answer
“I’d use a WebSocket server, store messages in MySQL. When user sends message, push to receiver. Use Redis for online status.”

#### ✅ Senior Answer
“Chat system: WebSocket for real-time, message queues for async delivery. Database: group messages in a sharded PostgreSQL by chat_id. Use Redis for presence, and a fanout service to deliver messages to all group members. I’d store offline messages and deliver on reconnect. For media, use S3 + CDN. Problems: heavy fanout for large groups, use batch processing or dedicated pub/sub.”

#### 🚀 Staff-level Answer
“I’d design a chat platform for 1B daily active users. Core: Messaging Gateway (WebSocket/HTTP long-polling) serving connections, stateless, fronted by load balancers with session affinity. Message processing: receive message, persist to a durable log (Kafka partitioned by conversation ID), then a delivery service reads and pushes to recipients’ gateway connections. Storage: Messages are stored in a time-series optimized store (HBase or Cassandra) with TTL for history. For group chats with thousands of members, I’d avoid fanout-on-write; instead use fanout-on-read: message stored in group timeline, and users query when online. But for active groups, we can still push to online members via an in-memory pub/sub (Redis Stream). We ensure idempotency with client-generated sequence numbers per conversation. Data model: conversation meta in MySQL, messages in Cassandra. For cross-region, we deploy multiple clusters with Kafka mirroring and local storage; user connection routed to nearest region. Critical challenges: message ordering under reconnection? We use vector clocks per conversation. Latency? Minimize hops: gateway directly reads from Kafka local partition. Observability: track end-to-end latency from send to receive. I would also design a media compression pipeline and thumbnail generation using PaaS. Trade-offs: fanout-on-read introduces read latency for large groups but reduces write amplification; we’ll cache recent messages aggressively. Disaster recovery: Kafka replication across AZs, Cassandra multi-AZ, graceful degradation to user presence only if storage unavailable.”

### Topic: Capacity Planning and Non-Functional Requirements

#### ❌ Mid-level Answer
“We estimate traffic and add 2x capacity. We monitor CPU and memory.”

#### ✅ Senior Answer
“I start with QPS and storage estimates from business projections. Derive required instances considering a load balancer overhead. Set HPA with target CPU 70%. Use load testing to validate. Plan for failover: reserve 50% headroom for AZ failure. Monitor P99 latency and set SLO with error budgets.”

#### 🚀 Staff-level Answer
“Capacity planning is an ongoing process, not a one-time exercise. I build a performance model mapping user operations to resource demands (RPCs, I/O, CPU milliseconds). We run load tests scaling from baseline to 2x expected peak, measuring degradation and identifying bottlenecks (e.g., database connection pool exhaustion at 5k concurrent). Based on the model, we provision with headroom for failure domains: if an AZ goes down, remaining AZs must handle full load without performance drop. We validate via chaos experiments (kill AZ). For cost efficiency, we use spot instances for stateless web tier and fallback to on-demand during shortage, coupled with intelligent auto-scaling. We also define SLOs per service: e.g., 99.95% availability, P99 < 50ms for search service. Error budgets dictate release velocity; when consumed, we freeze new features and focus on reliability. I implemented a capacity dashboard showing remaining headroom per service and predicted time to saturation, allowing proactive scaling. We reduced over-provisioning by 25% while maintaining SLO.”

## 5. High ROI Questions
- Design a URL shortener like TinyURL.
- Design a video streaming platform (Netflix).
- Design a ride-sharing service (Uber backend).
- Design a chat application (WhatsApp).
- Design a social media news feed (Twitter).
- Design a concurrent booking system for hotels/flights.
- Design a rate limiter (single-host and distributed).
- Design a key-value store (DynamoDB-like).
- Design a distributed message queue.
- Explain how you would handle scaling a system from 1K to 10M users.
- Walk through a capacity planning exercise for a new microservice.
- How do you ensure high availability in a multi-AZ deployment?
- Describe a time you resolved a critical performance bottleneck in production.
- How do you prioritize non-functional requirements when conflicting?

**Tricky Follow-ups:**
- “Your design relies on a cache, but the cache fails – what then?”
- “How do you handle a celebrity / hot partition problem in a database?”
- “Explain why two-phase commit is not sufficient for distributed transactions and what you propose instead.”
- “What happens if a critical network partition splits your cluster?”

## 6. Cheat Sheet (Revision)
- **Framework**: Clarify → Estimate → High-level → Deep dive → Scale.
- **Estimation**: QPS = DAU × avg requests/day / 86,400; Storage = requests × data size; Memory cache: 20% hot data.
- **Scaling**: web tier (stateless, load balancer); app tier (microservices, async, message queue); data tier (read replicas, sharding, NoSQL).
- **Resilience**: redundancy, failover, circuit breaker, retries, graceful degradation.
- **Key numbers**: CDN 20-100ms latency, database read 1-5ms, write 5-20ms, Redis 1ms.
- **Trade-offs**: SQL vs NoSQL (joins, scaling); sync vs async (latency vs decouple); cache (freshness vs performance).
- **System Engineering**: SLO, error budget, capacity headroom, RTO/RPO, graceful shutdown, chaos.

---

### 🎤 Communication Training
- **Be structured**: Use the PEDALS framework explicitly; call out each step.
- **Think out loud**: “I’d start by asking clarifying questions: is this global? How many users? We need strongly consistent or eventual? … Now, let’s estimate.”
- **Proactively discuss trade-offs**: “I could use a relational DB for simplicity, but to support 100K writes/sec we may need sharding, which adds complexity. Alternative: use Cassandra for writes, and PostgreSQL for the transactional metadata. Here’s the trade-off…”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas**: likely you've designed systems or at least large components; you know patterns.
- **Hidden gaps**:
  - **Structured problem-solving under time constraint**: practice with a timer.
  - **Deep system engineering**: capacity modeling formulas, chaos engineering practice.
  - **Real-world scale numbers**: knowing exact throughputs of Kafka partitions, database instance limits.
- **Overconfidence risks**: skimming over failure scenarios; not addressing observability and operability; jumping into microservices without analyzing monolith suitability.

Next combined topic ready when you say "next".
