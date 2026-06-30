# Databases Interview Guide

## 1. Overview
At 12+ YOE, database knowledge extends far beyond writing SQL queries. The interviewer expects you to:
- **Design robust schemas** – normalization, denormalization trade-offs, domain-driven aggregates, CQRS/event sourcing readiness
- **Reason about ACID, isolation, consistency** – not just recite levels, but predict anomaly scenarios under concurrent workloads and choose appropriate strategies
- **Master indexing & query optimization** – understand B-tree/GiST/GIN internals, partial/covering indexes, statistics, execution plans and how to rewrite queries for 100x performance improvements
- **Operate at scale** – replication, partitioning, sharding, connection pooling, failover, disaster recovery, backup strategies
- **Evaluate non-relational solutions** – document, key-value, column-family, graph, time-series, and when NOT to use a relational database
- **Identify failure modes** – connection leaks, deadlocks, replication lag, silent data corruption, cache invalidation

Real systems: transactional payment ledgers, high-throughput event stores, distributed metadata catalogs, search-optimized read models, analytics pipelines, multi-tenant SaaS backends.

## 2. Deep Knowledge Structure

### 2.1 Relational Database Fundamentals

#### ACID and Transaction Isolation
- **Concept** – Atomicity, Consistency, Isolation, Durability  
- **Subtopics**: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`, snapshot isolation
- **Microtopics (Deep Dive)**
  - Anomalies: dirty read, non-repeatable read, phantom read, serialization anomaly
  - How MVCC (Multi-Version Concurrency Control) works: tuples have xmin/xmax, transaction IDs, snapshots, vacuum (PostgreSQL) vs undo segments (InnoDB)
  - Write skew and how only SERIALIZABLE (or `SELECT FOR UPDATE`) prevents it
  - Pessimistic vs optimistic locking: row-level locks, `SELECT … FOR UPDATE`, application-level version columns
- **Atomic Topics (Interview-Level)**
  - *Explain phantom reads and how PostgreSQL’s REPEATABLE READ prevents them* – PostgreSQL uses snapshot at first query in the transaction; no phantom reads; but serialization anomaly remains
  - *How does MVCC cause bloating?* – old row versions not immediately removed; vacuum processes clean them; autovacuum tuning to prevent table bloat
  - *When would you use `SERIALIZABLE` in practice?* – critical financial calculations (e.g., transfer between accounts) where race conditions are otherwise too complex to guard
  - *What is a deadlock and how do you resolve it?* – DBMS automatically detects and kills one transaction; design order of resource access to avoid; use advisory locks when needed
- **Real-world Usage** – payment processing: `BEGIN TRANSACTION` → `SELECT ... FOR UPDATE` on account row → debit & credit → `COMMIT`, ensuring no double spend
- **Failure Scenarios** – relying on `READ COMMITTED` for balance checks causing oversold inventory; failing to understand that `SERIALIZABLE` may fail and need retry logic
- **Tradeoffs** – stronger isolation → more contention, more rollback retries; Snapshot Isolation (SERIALIZABLE in PostgreSQL 9.1+ using SSI) vs traditional 2PL is significant
- **Interview Expectations** – go beyond definitions; describe real production bugs caused by isolation level mismatch and how you fixed them
- **Common Mistakes** – assuming `REPEATABLE READ` is enough for everything; using `SERIALIZABLE` without idempotent retry logic

#### Indexing Internals
- **Concept** – Data structures to speed up access  
- **Subtopics**: B-tree, hash, GiST, GIN, BRIN, partial indexes, covering indexes, expression indexes
- **Microtopics (Deep Dive)**
  - B-tree structure: pages, fill factor, fan-out, split/merge operations, impact of UUID vs sequential key on insert performance (write amplification)
  - Composite index column order – most selective first vs. query pattern driven; index skip scan (Oracle 9i, PostgreSQL 17+)
  - Covering indexes: `INCLUDE` clause to avoid heap lookup, index-only scans, visibility map
  - Partial indexes: `WHERE` clause, useful for filtering soft-deletes or active records
  - BRIN (Block Range INdex) for large append-only tables with natural ordering (time-series)
  - GiST/GIN for full-text search and JSONB
- **Atomic Questions**
  - *Why is a sequential UUID bad as a primary key in a B-tree?* – random inserts cause page splits, index bloat, fragmentation; prefer ULID or UUID v7 (time-ordered)
  - *How does a covering index improve performance?* – avoids fetching rows from heap; index-only scan when all selected columns are in index
  - *Explain a scenario where a partial index is better than a full index* – table has millions of rows but only 5% active; partial index `WHERE deleted_at IS NULL` smaller, faster
  - *What statistics does the planner use?* – `pg_stats` contains most-common values, histograms, null fraction, correlations
- **Real-world Failures** – adding an index without considering write overhead in high-throughput INSERT-only system leading to severe performance degradation; missing index on FK column causing sequential scan on every join
- **Tradeoffs** – more indexes → slower writes, more disk space; partial indexes reduce overhead but only applicable when WHERE clause is constant
- **Interview Expectations** – when asked to optimize a slow query, walk through reading the execution plan, identifying bottlenecks (sequential scan, hash/nested loop join, sort), and designing a suitable index (composite, covering, partial)

#### Query Optimization & Execution
- **Subtopics**: `EXPLAIN`/`EXPLAIN ANALYZE`, cost model, join algorithms (nested loop, hash, merge), statistics, hints
- **Microtopics**
  - Nested loop: efficient when inner table has index on join key; cheap if small
  - Hash join: builds hash table from inner; good for larger sets without indexes
  - Merge join: presorted inputs; requires sort or indexes
  - Bitmap scans (PostgreSQL): combine multiple index scans, useful for complex OR/AND
  - Plan caching and parameter sniffing issues (SQL Server) or bad generic plan (PostgreSQL can choose custom plan for first few executions)
- **Atomic**
  - *How to interpret `EXPLAIN ANALYZE` output?* – actual time vs estimated, loops, rows, shared hit/read; identify if estimates are off
  - *Why might the planner choose a slower index scan?* – poor statistics (stale, not enough sample), correlated columns, functional dependencies not captured
  - *What is a correlated subquery and how to optimize it?* – often can be rewritten to JOIN or LATERAL subquery
- **Failure** – statistics not updated causing sequential scan where index exists; plans wildly different after DB upgrade
- **Tradeoffs** – `enable_*` parameters for planner tuning (risky) vs. query refactoring

### 2.2 Schema Design & Data Modeling
- **Normalization & Denormalization**
  - Normal forms (1NF to 3NF/BCNF), when to break: reporting, performance, simplicity
  - Denormalization: materialized views, duplicated columns, aggregate tables; data inconsistency risk
- **Entity-Relationship & Domain-Driven Design**
  - Aggregates and transactional boundaries align with microservices
  - Polymorphic associations (table-per-hierarchy, table-per-type, table-per-class) via JPA inheritance strategies – performance traps
- **Time-series patterns** – partition by time, downsampling, hypertables (TimescaleDB)
- **Multi-tenancy** – separate database, separate schema, shared table with tenant_id; security, isolation, connection pooling nuances
- **Atomic** – *How would you design a ledger system with double-entry bookkeeping?* – immutability, append-only, balance computed via sum, integrity constraints, partitioned by account or date

### 2.3 Distributed Databases & Scaling
- **Replication** – physical (streaming replication), logical (decoding, pub/sub), synchronous vs asynchronous, failover/switchover
  - PostgreSQL synchronous replication: `synchronous_commit`, remote_apply, failover with Patroni/Stolon/Raft
  - Leader-follower vs multi-leader (conflict resolution with CRDTs, application-level)
- **Sharding** – horizontal partitioning by tenant, key-range, hash; cross-shard queries, rebalancing, Citus, Vitess, built-in (MongoDB, Cassandra)
- **CAP Theorem & PACELC** – in failure: choose between availability and consistency; normally: latency vs consistency; actual implementations (CockroachDB, Spanner provide strict consistency, Mongo adjustable write concern)
- **Atomic**
  - *How to handle replication lag in an application?* – read from primary when immediate consistency required (write-then-read), use GTID/LSN to wait for replica, or use stale reads with tolerance
  - *Design a globally distributed database – what trade-offs?* – geo-partitioning for low latency, inter-region replication for disaster recovery, conflict resolution
- **Failure** – failing to test failover and losing committed transactions due to asynchronous replication; split-brain in multi-master

### 2.4 Non-Relational Databases (NoSQL)
- **Concept** – Polyglot persistence: choose the right tool for the job  
- **Key-Value Stores** (Redis, DynamoDB)
  - Redis: data structures, persistence (RDB/AOF), replication, sentinel, cluster; use as cache vs primary store
  - Atomic questions: *How to implement distributed locks with Redis?* – Redlock algorithm, safety concerns, fencing token
- **Document Stores** (MongoDB, Couchbase)
  - Flexible schema trade-offs, indexing, aggregation pipeline, write concerns, majority reads
  - Failures: losing data with default write concern (no acknowledgement); not designing indexes for query patterns
- **Column-Family** (Cassandra, HBase)
  - Partition key, clustering columns, LSM-tree internals, compaction, read repair, hinted handoff
  - Data modeling: query-first design, denormalization, time-to-live (TTL)
- **Graph** (Neo4j, Amazon Neptune) – for highly connected data, recursive queries (variable-length paths)
- **Time-Series** (InfluxDB, TimescaleDB) – high write throughput, downsampling, retention policies
- **Tradeoffs** – NoSQL: eventual consistency, lack of joins, limited transactions (unless recent ACID enhancements), but horizontal scalability

### 2.5 Internals & Performance Tuning
- **Storage Engine** – heap vs index-organised tables, page layout, TOAST (oversized attributes), compression
- **Write-Ahead Log (WAL)** – how transactions are made durable, checkpointing, recovery
- **Vacuuming & Table Bloats** (PostgreSQL) – auto-vacuum tuning, `dead_tuple` count, transaction ID wraparound
- **Connection Pooling** – PgBouncer (transaction pooling), HikariCP; sizing, `maxLifetime`, `leakDetectionThreshold`
- **Caching** – DB buffer pool, application-level caching (Redis), cache invalidation strategies (write-through, cache-aside, event-based)
- **Atomic**
  - *How to find the worst-performing queries?* – `pg_stat_statements`, `slow_query_log`, `performance_schema.events_statements_summary_by_digest`
  - *How does PostgreSQL’s checkpoint tuning affect performance?* – `checkpoint_timeout`, `max_wal_size`, reduce spikes by spreading writes

### 2.6 Data Integrity & Operations
- **Backup & Restore** – logical (`pg_dump`), physical (`pg_basebackup`, WAL archiving), PITR (Point-In-Time Recovery)
- **Migration strategies** – expand-contract pattern, online schema changes (pt-online-schema-change, gh-ost), zero-downtime deployment
- **Audit trails** – temporal tables, trigger-based logging, CDC (Debezium)
- **Security** – encryption at rest/transit, parameterized queries to prevent injection, row-level security (RLS)

## 3. Code & Examples

```sql
-- Transaction with explicit locking to prevent lost update
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- compute new balance in application
UPDATE accounts SET balance = 1000 WHERE id = 1;
COMMIT;

-- Using advisory lock for serialization of application-level job
SELECT pg_advisory_xact_lock(123);
-- job work
COMMIT; -- lock released automatically

-- Composite index design for query patterns
-- Query: SELECT * FROM orders WHERE customer_id = ? AND order_date BETWEEN ? AND ?
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Covering index to enable index-only scan
CREATE INDEX idx_orders_covering ON orders(customer_id, order_date) INCLUDE (status, total);

-- Partial index for soft-deletes
CREATE INDEX idx_orders_active ON orders(order_date) WHERE deleted_at IS NULL;

-- Materialized view for report performance
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', order_date) AS day, sum(total) AS revenue
FROM orders
GROUP BY date_trunc('day', order_date);
-- Refresh periodically: REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;

-- Querying with LIMIT and keyset pagination (seek method) to avoid offset problems
SELECT * FROM orders
WHERE (order_date, id) > ('2024-01-01', 123456)
ORDER BY order_date, id
LIMIT 1000;

-- PostgreSQL logical replication setup (publication/subscription)
CREATE PUBLICATION mypub FOR TABLE orders, customers;
-- On replica:
CREATE SUBSCRIPTION mysub CONNECTION 'host=...' PUBLICATION mypub;

-- Avoiding N+1 in SQL: use JOIN + conditional aggregation instead of per-row queries
SELECT u.id, u.name,
       COUNT(o.id) FILTER (WHERE o.status = 'delivered') AS delivered_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;

-- Use window functions for running totals without self-join
SELECT id, amount,
       SUM(amount) OVER (ORDER BY id) AS running_total
FROM transactions;
```

## 4. Interview Intelligence

### Topic: Isolation Levels and MVCC

#### ❌ Mid-level Answer
“READ COMMITTED prevents dirty reads. REPEATABLE READ prevents non-repeatable reads. SERIALIZABLE is the safest. I usually use READ COMMITTED.”

#### ✅ Senior Answer
“In PostgreSQL, READ COMMITTED sees each query snapshot at its start; REPEATABLE READ uses a single snapshot for the whole transaction, preventing non-repeatable reads. However, REPEATABLE READ can still experience serialization anomalies like write skew. SERIALIZABLE uses Serializable Snapshot Isolation (SSI) which monitors read/write dependencies and may cause serialization failures needing application retry. I choose REPEATABLE READ for most transactional operations with `SELECT FOR UPDATE` to protect critical updates; for multi-row invariant checks, SERIALIZABLE is necessary but I also design idempotent retries.”

#### 🚀 Staff-level Answer
“We operate financial ledgers where isolation is crucial. I’ve deep-dived into PostgreSQL’s MVCC: each row version has xmin/xmax, and visibility depends on transaction snapshot. We had a production bug where a balance check under REPEATABLE READ allowed an overdraft because a concurrent transfer was invisible. Rewriting to `SELECT ... FOR UPDATE` prevented the race condition. In another service, we adopted SERIALIZABLE for account provisioning to avoid double allocation, but we had to build client-side retry logic with exponential backoff because serialization failures spike under contention. We also tuned `max_pred_locks_per_page` and monitored `pg_stat_database.xact_commit/rollback`. Additionally, we disabled `default_transaction_isolation` change across the board; each module explicitly sets the level. I’ve also debugged a long-running transaction causing vacuum to unable to clean dead tuples, leading to table bloat and degraded performance—the solution was setting `idle_in_transaction_session_timeout` to kill stuck transactions, and using `statement_timeout` with a retry wrapper.”

### Topic: Indexing and Query Optimization

#### ❌ Mid-level Answer
“Create an index on columns used in WHERE clause. Use EXPLAIN to check if index is used. Avoid SELECT *.”

#### ✅ Senior Answer
“I analyze slow queries from `pg_stat_statements`. For a query with WHERE and ORDER BY on different columns, I design a composite index where equality columns precede the range/order column. I also use partial indexes to trim irrelevant data. Covering indexes with INCLUDE eliminate heap fetches. I verify with `EXPLAIN (ANALYZE, BUFFERS)` that index-only scans are used, and watch for sorts that can be avoided. I’ve also used expression indexes for queries on functions like `lower(email)`.”

#### 🚀 Staff-level Answer
“In our platform, a dashboard query performing a `GROUP BY` on a timestamp generated over 10M rows caused an outage. I re-engineered it: first, we added a BRIN index on the timestamp because data is append-only and physically correlated, reducing index size by 100x vs B-tree. Second, I introduced a materialized view with hourly rollups, using `REFRESH MATERIALIZED VIEW CONCURRENTLY` every 10 minutes. For near real-time queries, I used the BRIN index combined with parallel query. I also rewrote the query to use `LATERAL` to avoid a correlated subquery that was executing for each aggregated row. Further, we implemented keyset pagination (seek method) for API endpoints instead of offset, because offset scanning deep pages is disastrous. On the application side, I enforced statement timeouts via JDBC and HikariCP query timeout, and added metrics to track query durations by digest, alerting when 99th percentile exceeds threshold. I also reviewed statistics to ensure `default_statistics_target` was sufficient for skewed data. For a multi-tenant index, I used composite with tenant_id first, which improves both performance and row-level security via RLS policies.”

## 5. High ROI Questions
- Explain ACID and each isolation level. Provide real-world anomaly examples.
- How does MVCC work in your DB of choice? What is vacuum and why is it needed?
- Walk through a slow query: how do you identify the bottleneck and what steps would you take?
- What are the trade-offs between using an ORM and raw SQL?
- Design a database schema for a ride-hailing app (or hotel booking) – talk through normalization, indexing, concurrency.
- How do you handle database migrations with zero downtime?
- Compare database scaling strategies: replication, partitioning, sharding – when to use which?
- How would you implement a distributed transaction across two services each with its own DB? Discuss Sagas, outbox pattern.
- What factors influence connection pool sizing? How do you avoid connection leaks?
- How would you migrate from a monolithic DB to microservices, managing downstream data access?
- CAP theorem: how does it apply to your current database choices? Give concrete examples.
- Explain eventual consistency and how to design UI/API around it.

**Tricky Follow-ups:**
- “You have a table with 1 billion rows and need to add a column with not null default – how to do it without locking?”
- “An `UPDATE` causes a deadlock with a `SELECT` – how did that happen? (lock ordering, `SELECT … FOR UPDATE`)”
- “Your replica lags significantly – what’s the impact and how to mitigate?”
- “Why would a B-tree index not be used despite being present? Give at least three reasons.”

## 6. Cheat Sheet (Revision)
- **ACID**: Atomic, Consistent (constraints), Isolated, Durable.
- **Isolation anomalies**: Dirty read (RC+ prevents), Non-repeatable read (RR+ prevents), Phantom (Serializable prevents), Serialization anomaly (2PL/SSI prevents).
- **MVCC**: Snapshot per transaction; dead rows vacuumed; long transactions block cleanup.
- **Index types**: B-tree (default, range lookups), Hash (only `=`), GIN (composite values, full-text), GiST (geometric, full-text), BRIN (block range min/max summaries, small size).
- **Composite index column order**: equality columns first, then range/order; skip unscanned columns.
- **Covering index**: `INCLUDE` non-key columns; enables index-only scan.
- **Partial index**: `WHERE` clause; good for sparse filters.
- **Connection pool**: `pool_size = ((core_count * 2) + effective_spindle_count)`; HikariCP `maxLifetime` < DB `wait_timeout`.
- **Deadlock prevention**: always lock in the same order; use advisory locks for application-level serialization.
- **Replication**: physical (PG streaming) for DR, logical for selective replication and online upgrades.
- **Pagination**: avoid `OFFSET` large values; use keyset/seek method with composite cursor.
- **Serializable retry**: exponential backoff, idempotency.
- **Migration**: expand-contract: add new column (nullable), write both columns, backfill, switch reads to new, drop old.
- **Distributed**: Saga (orchestration/choreography), Outbox pattern for reliable events, CDC with Debezium.

---

### 🎤 Communication Training for Databases
- **Frame everything in operational impact**: “When the slow query hit, P99 latency jumped from 50ms to 2000ms, causing upstream queue buildup. I analyzed the execution plan, found a sequential scan on a billion-row table, added a composite partial index covering the query, and brought latency back to sub-100ms. I also added a graph in our monitoring dashboard tracking `pg_stat_statements` execution time to catch regressions early.”
- **Show trade-off analysis**: “We considered normalizing the schema fully, but that would add 3 JOINs per query. Under 10K QPS, the additional CPU cost was unacceptable, so we denormalized critical columns and used triggers to keep consistency, accepting the added complexity for performance.”
- **Use the pattern “what I learned from a failure”**: “We once introduced transaction isolation mismatch causing lost updates in inventory. After the incident, we enforced explicit locking patterns and built integration tests with concurrent goroutines to verify correctness. Now every PR that touches a critical write path must include concurrency tests.”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas** (likely): SQL, basic indexing, normalization, using an ORM, replication concepts, some performance tuning.
- **Weak areas / Hidden gaps**:
  - **MVCC vacuuming / bloat deep tuning** – many devs don’t know how to monitor and tune autovacuum thresholds, leading to table bloat. Understand `autovacuum_vacuum_scale_factor` vs `autovacuum_vacuum_threshold`.
  - **Non-relational modeling** – especially time-series (compression, retention) and graph – if you only have relational experience, be ready to discuss when you’d choose something else and why.
  - **Distributed consistency** – beyond CAP, understand consensus (Raft), write quorums, read repair, hinted handoff in Cassandra-style systems.
  - **CDC and event streaming** – Debezium, Kafka Connect, outbox pattern for microservices; might be less practiced but expected at Staff level.
  - **Online schema change tools** – gh-ost or pt-online-schema-change for zero-downtime migrations.
- **Overconfidence risks**:
  - Assuming your ORM-generated queries are optimal without verifying – you can be surprised by N+1 or suboptimal joins.
  - Believing indexing solves all performance issues; sometimes you need query rewrite, caching, or data model changes.
  - Overlooking security – SQL injection, encryption at rest, RLS. Staff engineers are expected to enforce database security standards.
  - Neglecting operational side – backup verification, restore practice, disaster recovery runbooks. Interviewers look for production rigor.

Proceed to next topic when ready.
