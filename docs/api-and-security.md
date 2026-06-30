# APIs & Security Interview Guide

## 1. Overview
At Staff+ level, API design and security are inseparable. You cannot design a robust API without considering authentication, authorization, data validation, rate limiting, and attack surfaces. Conversely, you cannot secure a system without understanding its API surface, versioning, and contract management.

This guide covers:
- **API Design** – RESTful principles, gRPC, GraphQL, AsyncAPI, contract-first development, versioning, backward compatibility
- **API Security** – OWASP Top 10 (API and web), authentication standards (OAuth2, OIDC, SAML), JWT, mTLS, authorization (RBAC/ABAC/ReBAC), rate limiting, input validation, cryptography
- **Cross-cutting concerns** – observability for APIs, API gateways, threat modeling, secure SDLC

Real systems: public developer platforms (Stripe, Twilio), internal microservice meshes, mobile backend APIs, zero-trust architectures, PCI/HIPAA compliant APIs.

## 2. Deep Knowledge Structure

### 2.1 API Design Fundamentals

#### RESTful API Design
- **Resource Modeling**: nouns, not verbs; nested resources vs flat; sub-resources and lifecycle
- **HTTP Methods**: GET (safe, idempotent), POST, PUT (idempotent), PATCH (not necessarily idempotent), DELETE (idempotent)
- **Status Codes**: meaningful usage (201 Created, 202 Accepted, 409 Conflict, 429 Too Many Requests, 503 Service Unavailable)
- **Versioning**: URI (`/v1/...`), header (`Accept: application/vnd.company.api+json;version=1`), query param; contract-first evolution
- **Pagination**: offset/limit (dangerous for real-time), cursor-based (keyset), Link header (RFC 5988)
- **Filtering, Sorting, Field Selection**: simple query parameters to reduce payload
- **HATEOAS**: maturity levels, practical use for discoverability (rarely adopted fully)
- **Microtopics**
  - N+1 problem in API responses: use `expand`, `include`, compound documents (JSON:API)
  - Hypermedia: affordances; real-world examples (GitHub, Stripe)
  - POST/PUT/PATCH semantics: idempotency key for POST (Stripe pattern) to safely retry
- **Atomic Interview Q**:
  - *How would you design a REST API for a banking transaction?* – use POST with idempotency key, 201 with transaction resource, 409 if duplicate, support GET /transactions?filter[customer]=..., pagination cursor-based because of mutable ordering
  - *When is it okay to use GET for state-changing operations?* – never (violates RFC and safety), except for tracking pixels (but use POST there ideally)
- **Real-world Failures**: API versioning ignored leads to broken clients; using offsets causing duplicates when data inserted; long URLs due to deep nesting.

#### gRPC Design
- **Protocol Buffers**: schema-driven, strongly typed, backward/forward compatibility rules (field numbers, reserved)
- **Service Definitions**: unary, server-streaming, client-streaming, bidirectional streaming
- **Error Handling**: standard error model (gRPC status codes), metadata for details; richer error via `google.rpc.Status` and details (ErrorInfo, RetryInfo)
- **Load Balancing**: client-side (LB policy, resolver), proxy-based (Envoy gRPC bridging)
- **Interceptors** for auth, logging, retries
- **Microtopics**
  - gRPC-Web for browser clients
  - Deadlines & cancellations: propagate deadlines across services
  - Protobuf versioning best practices
- **Atomic Q**:
  - *Compare gRPC vs REST for a microservice ecosystem.* – gRPC: strong contracts, faster serialization, streaming, but limited browser support, less cacheable, not as easily testable with curl; REST: ubiquitous, human-readable, cacheable, but schema optional, more overhead.

#### GraphQL Design
- **Type System**: queries, mutations, subscriptions; strict typing
- **Resolver Model**: per-field functions; N+1 problem solved with DataLoader (batching + caching)
- **Security & Performance**: depth limits, rate limiting based on query cost analysis, introspection control
- **Versioning**: do not version; evolve schema continuously with deprecation strategy
- **Microtopics**
  - Subscription implementation: WebSocket, SSE, Apollo
  - Federation for microservices: Apollo Federation, schema stitching
- **Atomic Q**:
  - *When would you choose GraphQL over REST?* – multiple clients with diverse data needs, aggregated views; risk: poorly optimized resolvers cause massive database load

#### AsyncAPI & Event-Driven APIs
- **Definition**: similar to OpenAPI but for messaging protocols (Kafka, MQTT, AMQP)
- **Schema registry**: ensure event contract compatibility (Avro, Protobuf)
- Backward/forward compatibility checks

#### Contract-First & API Lifecycle
- **OpenAPI/Swagger**: design first, generate server stubs and client SDKs
- **Linting**: Spectral for OAS validation
- **Consumer-Driven Contracts** (Pact) for microservices
- **API Governance**: linting in CI, breaking change detection (openapi-diff, Swagger Diff)

### 2.2 API Security 

#### OWASP API Security Top 10 (2023)
- **Broken Object Level Authorization** (BOLA/IDOR): missing per-object ACL checks
- **Broken Authentication**: weak token handling, missing protection
- **Broken Object Property Level Authorization**: excessive data exposure, mass assignment
- **Lack of Resources & Rate Limiting**: DoS, brute force
- **Broken Function Level Authorization**: admin endpoints exposed
- **Unrestricted Access to Sensitive Business Flows**: e.g., scalping inventory
- **Server-Side Request Forgery** (SSRF): forging requests from server
- **Security Misconfiguration**: verbose errors, missing security headers
- **Improper Inventory Management**: shadow APIs, zombie endpoints
- **Unsafe Consumption of APIs**: trusting third-party APIs too much

#### Authentication & Authorization
- **OAuth2 & OIDC**:
  - Grant types: Authorization Code + PKCE (public clients), Client Credentials (service-to-service), Refresh Token
  - Token types: opaque vs JWT; JWT signing (RS256, ES256), encryption (JWE)
  - Token validation: expected issuer, audience, expiry, nonce, cnf (mTLS binding)
  - OIDC adds identity layer; ID token vs Access token
- **JWT Best Practices**:
  - Avoid storing PII or sensitive data; keep payload small
  - Validate signature, algorithm (reject `none`), set max lifespan ~15min
  - Distribute via short-lived tokens + refresh rotation; rotate signing keys
  - Token binding to client (TLS/DPoP)
- **Service-to-Service**:
  - mTLS: certificate-based mutual authentication; SPIFFE/SPIRE for workload identity
  - Signed JWT with private key (service account)
  - API Key + secret (less secure, for legacy)
- **Authorization Models**:
  - RBAC: role-based; straightforward but role explosion
  - ABAC: attribute-based (XACML, OPA); flexible, externalized policy
  - ReBAC: relationship-based (Google Zanzibar); for social graphs
  - Policy engines: OPA/Rego, AWS Cedar
- **Atomic Q**:
  - *How do you secure a public API endpoint that third parties call?* – API key or OAuth2 client_credentials; rate limiting per key; input validation; TLS; openAPI spec; monitoring for abuse.
  - *What is the difference between an ID token and an Access token?* – ID token proves authentication (for client), Access token is used to access resource server (Bearer); ID token contains user claims, access token contains scopes/permissions.

#### API Gateway Security
- **Rate limiting**: per API key/ IP/ user tier; token bucket/ sliding window
- **Authentication & Authorization**: validate JWT, OAuth scopes, enforce policies
- **WAF**: protect against injection attacks, bot control
- **Logging & Alerts**: detect reconnaissance patterns (404 spikes), credential stuffing
- **TLS Termination**: enforce HTTPS; HSTS headers

#### Secure Coding Practices
- **Input validation**: whitelist approach; prevent injection (SQL, XSS, etc.)
- **Output encoding**: context-appropriate
- **Use of security libraries**: Spring Security, Passport, Helmet
- **Least privilege**: minimal scopes/roles; avoid overly permissive policies

#### Cryptography for APIs
- **TLS**: at least 1.2, forward secrecy, disable weak ciphers
- **Hashing**: bcrypt/argon2 for passwords
- **Message security**: integrity with HMAC; signatures for webhooks (Stripe signature verification pattern)
- **Secrets management**: rotate keys frequently, use vault

#### Threat Modeling for APIs
- **STRIDE** (Spoofing, Tampering, Repudiation, Info Disclosure, Denial of Service, Elevation of Privilege)
- Build Data Flow Diagrams, identify trust boundaries; iteratively mitigate

## 3. Code & Examples

```java
// Spring Boot REST API with idempotency key check
@RestController
public class PaymentController {
    @PostMapping("/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @Valid @RequestBody PaymentRequest request) {
        return paymentService.process(idempotencyKey, request);
    }
}
@Service
public class PaymentService {
    @Transactional
    public ResponseEntity<PaymentResponse> process(String idempotencyKey, PaymentRequest req) {
        Optional<IdempotentRecord> existing = idempotentRepo.findByKey(idempotencyKey);
        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get().getResponse());
        }
        // business logic, save payment
        IdempotentRecord record = new IdempotentRecord(idempotencyKey, response);
        idempotentRepo.save(record);
        return ResponseEntity.status(201).body(response);
    }
}
```

```protobuf
// gRPC service with field deprecation
service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
}
message CreateOrderRequest {
  string customer_id = 1 [deprecated = true]; // kept for backward compatibility
  string new_customer_uuid = 2;
  // ...
}
```

```yaml
# OpenAPI snippet with security definition
paths:
  /users:
    get:
      security:
        - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

```javascript
// Node.js webhook signature verification (simplified)
const crypto = require('crypto');
app.post('/webhook', (req, res) => {
  const signature = req.headers['x-signature'];
  const hmac = crypto.createHmac('sha256', secret);
  const digest = hmac.update(JSON.stringify(req.body)).digest('hex');
  if (crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(digest))) {
    // process
    res.status(200).end();
  } else {
    res.status(403).end();
  }
});
```

## 4. Interview Intelligence

### Topic: Designing a Secure Public API for a Fintech Platform

#### ❌ Mid-level Answer
“I’d use REST with OAuth2, JWT for authentication, HTTPS, rate limiting. Validate inputs.”

#### ✅ Senior Answer
“I’d design a REST API following OpenAPI spec, Auth code flow with PKCE for SPA/mobile, client credentials for partners. JWTs short-lived, refresh token rotation. All endpoints require scopes. Implement idempotency for mutations. Rate limit per client globally to prevent abuse. Use parameterized queries and WAF. Version in URL. Audit logs.”

#### 🚀 Staff-level Answer
“I’d lead API design using a contract-first approach: OpenAPI as source of truth, auto-generated SDKs for clients. Authentication: OIDC with certified provider; access tokens as JWTs signed with RS256, validated by API gateway. Service-to-service: mTLS with SPIFFE IDs enforced via Istio. Authorization: policy as code using OPA, evaluating RBAC plus fine-grained ABAC based on resource attributes and user context. API abuse protection: tiered rate limiting per client and per user, with sliding window algorithms; risk-based step-up for high-value operations. Input validation: allowed whitelists, input length limits; output sanitization. To prevent BOLA, we use a centralized authorization filter that retrieves resource ownership from a graph database (Zanzibar-like). All API keys are stored hashed. We use canary tokens to detect leaked credentials. Monitoring: detect credential stuffing, abnormal usage patterns. Panic code: kill switch to block compromised client. The API gateway (Envoy) performs request validation against OpenAPI schema to reject malformed input early. We apply threat modeling in design phase, and our CI pipeline includes OWASP ZAP active scan against staging APIs. I also built an internal developer portal with all APIs cataloged and their ownership, so shadow APIs are eliminated. Compliance: we align with PCI DSS, encrypt card data with KMS, and never log full PAN.”

### Topic: GraphQL vs REST vs gRPC Decision

#### ❌ Mid-level Answer
“REST is simple and widely used. GraphQL gives flexible queries. gRPC is fast. Choose based on need.”

#### ✅ Senior Answer
“I evaluate criteria: client requirements, performance, tooling, team expertise, and ecosystem. REST is universal, cacheable, good for public APIs. GraphQL avoids over-fetching, ideal for complex UIs with many nested entities, but security and N+1 require careful resolver design. gRPC for high-performance internal microservices where streaming matters. I’ve used GraphQL for mobile backends to allow frontend to evolve without server changes.”

#### 🚀 Staff-level Answer
“I base my decision on the architectural characteristics of the system. For a public Bank API, REST is better because of mature caching (HTTP caching, CDN), auditability, and wide consumer base. For internal high-throughput data pipelines, gRPC’s binary protocol and multiplexed streams reduce latency and resource consumption. GraphQL I’d choose for a customer-facing dashboard where data complexity varies per page and we need to avoid under/over-fetching; but I’ll mitigate risks: enforce query depth and cost analysis with persistence, use DataLoader batching to avoid N+1, and disable introspection in production. I also consider tooling: GraphQL code generation, type sharing, and the need for client-driven development. If there’s need for both REST and real-time events, I combine REST for commands/queries and AsyncAPI/SSE for subscriptions. I’ve also introduced a thin gateway layer that can translate between protocols (REST externally, gRPC internally) to decouple external API from internal service evolution.”

## 5. High ROI Questions
- Design a RESTful API for a library management system, covering versioning, pagination, and error handling.
- How do you secure a REST API? Cover OWASP Top 10 for APIs.
- Explain OAuth2 Authorization Code flow with PKCE. Why PKCE?
- Compare and contrast JWT and opaque tokens. When would you use each?
- How do you prevent Broken Object Level Authorization (BOLA/IDOR)?
- Discuss API versioning strategies and their trade-offs.
- How do you handle idempotency for POST requests?
- What are the best practices for API error responses?
- How do you secure communication between microservices?
- What service mesh security features would you enable for an API?
- How would you identify and eliminate shadow/rogue APIs?
- Explain the differences between symmetric and asymmetric JWT signatures.

**Tricky Follow-ups:**
- “A user’s JWT is stolen; how do you mitigate?” (short-lived, refresh token rotation, revoke, token binding)
- “How do you prevent mass assignment in a REST API?” (DTO with allowed fields, `@JsonIgnore`/`@JsonView`, Spring `@InitBinder` allowed/blocked fields)
- “Your API is being scraped by competitors; how do you detect and prevent?”

## 6. Cheat Sheet (Revision)
- **REST**: resources, proper methods, status codes, pagination (cursor for mutable data), versioning (URI or header)
- **gRPC**: strong contract, protobuf, streaming, deadlines, backward compatibility via field numbers
- **GraphQL**: flexible queries, N+1 solve with DataLoader, query cost analysis, deprecation
- **Security**: OWASP API Top 10; use OAuth2/OIDC; validate tokens; BOLA fix: always check ownership; rate limit; input validation
- **Idempotency**: header, unique key, store response
- **API Gateway**: auth, rate limiting, routing, analytics
- **SCA**: check OpenAPI in CI, no breaking changes

---

### 🎤 Communication Training
- **Think like a platform designer**: “When I build an API, I treat it as a product for developers. I provide client SDKs, clear docs with examples, a sandbox, and stable contracts with deprecation policy.”
- **Security mindset**: “I always ask: what if an attacker has a valid token? I design authorization to enforce least privilege, and monitor for anomalous behavior.”

### 📈 Personalised Self-Assessment (12 YOE)
- **Strong areas**: likely REST design, some security (OAuth2, JWT), OpenAPI.
- **Hidden gaps**: deep OWASP API Top 10 experience, GraphQL/gRPC in production, API threat modeling, consumer-driven contract testing.
- **Overconfidence risks**: thinking API key is sufficient; ignoring BOLA; assuming versioning is trivial; not having practiced incident response for API abuse.

Ready for the next combined topic when you say "next".
