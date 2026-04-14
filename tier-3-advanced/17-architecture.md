# Architecture

## What is it?

Architecture is the discipline of structuring a software system so that it can be built, maintained, and extended by people over time. It sits above individual functions or classes and below the product requirements — it is the scaffolding that determines how easy or hard everything else will be.

The core architectural question is: **how do you divide the system into parts, and how do those parts communicate?**

Three major patterns answer that question differently:

**Monolith** — Everything in one process. The entire application is deployed as a single unit. All modules call each other directly through in-memory function calls. One database. One deployment. Simple to understand and debug.

**Modular monolith** — Same deployment model as a monolith, but the code is divided into well-separated modules with defined boundaries and interfaces. You could, in theory, extract a module into a separate service without changing its code — only the communication path changes.

**Microservices** — Each module runs as an independent process, often with its own database. Services communicate over a network (HTTP, gRPC, message bus). Each can be deployed, scaled, and maintained independently.

The spectrum is continuous, not categorical. A system might start as a monolith, extract one high-traffic module into a separate service while keeping everything else together — that is a **modular monolith with one service extracted**, not a full microservices architecture.

## Why it matters for vibe coding

Without architectural knowledge, you cannot evaluate what AI generates. AI will confidently produce a microservices architecture for a todo app that has five users. It will generate API endpoints without thinking about versioning. It will generate code that assumes a service always has access to the same database instance. It will hallucinate service boundaries that do not hold up under real requirements.

Specific failure modes:

- **AI generates microservices for a project that needs a monolith.** You ask for a "scalable architecture" and AI produces eight services with individual Docker containers, Kubernetes configs, and service meshes for an app serving 50 concurrent users. The complexity is 10x and the latency is worse because everything is now a network call.

- **AI does not understand where service boundaries should be.** It will split a system along "technical" lines (one service for auth, one for email, one for database) rather than business capabilities. This creates tight coupling through the network that did not exist when they were in-memory calls.

- **AI generates APIs that break backward compatibility.** It adds a required field to a response, removes a field, or changes the type of a return value without versioning. Existing clients break silently.

- **AI generates code that assumes a shared database across services.** This is the most common microservices mistake: creating services that query each other's data stores directly rather than going through APIs, which couples services at the data layer and defeats the purpose of separation.

## The 20% you need to know

### Monolith vs. modular monolith vs. microservices: the real tradeoffs

| | Monolith | Modular Monolith | Microservices |
|---|---|---|---|
| Deployment | Single unit | Single unit | Independent per service |
| Scaling | Whole app scales | Whole app scales | Scale individual services |
| Complexity | Low (few moving parts) | Medium (module boundaries) | High (network, discovery, contracts) |
| Debugging | One process, familiar | One process, module-level | Distributed tracing required |
| Fault isolation | Poor (one bug crashes all) | Moderate | Good (one service can fail independently) |
| Team independence | Low | Moderate | High |
| Latency | In-memory calls (fast) | In-memory calls (fast) | Network calls (slower) |
| Data consistency | Easy (single DB transactions) | Easy | Hard (eventual consistency, sagas) |

**When to use each:**

- **Monolith** — Small teams, early-stage products, less than ~10k concurrent users, simple business rules. The overhead of microservices is not worth it. A well-structured monolith is more maintainable than a poorly-designed microservices architecture.

- **Modular monolith** — Growing teams that want code separation without network overhead. The right choice for most projects between "solo hack" and "at-scale tech company." Module boundaries provide the mental model for eventual extraction.

- **Microservices** — Large teams (20+ engineers), very high traffic on specific subsystems, strict fault isolation requirements, different subsystems having genuinely different scaling needs. Only start here when you have the operational maturity to manage distributed systems (monitoring, tracing, deployment pipelines, on-call rotation).

**The dirty secret:** Most projects never need microservices. A well-structured monolith with clear module boundaries will carry you further than a microservices architecture managed by a team that cannot keep up with its complexity.

### The C4 model for thinking about system design

C4 (Context, Containers, Components, Code) is a lightweight modeling technique that maps four levels of abstraction. You do not need to produce formal C4 diagrams — understanding the mental model is what matters.

**Level 1 — Context (the big picture):** Who are the actors? What external systems does your system interact with? What are the trust boundaries? This level answers: "what system is this, and what sits around it?"

**Level 2 — Container (the deployment topology):** Each container is a separately deployable process. In a monolith, there is one container. In a microservices architecture, each service is a container. In a web app, the browser and the server are two containers. This level shows how processes communicate and where data lives.

**Level 3 — Component (the module boundaries):** Inside each container, what are the major modules? In a well-structured monolith, this maps to the modules. In a microservices architecture, this maps to the internal structure of each service. Components within a container communicate through in-memory calls; components across containers use the network.

**Level 4 — Code (the implementation details):** The actual class/function structure. Most AI-generated code lives here.

**The practical use of C4 for vibe coding:** When AI generates code, you should be able to place each piece at the right C4 level. If AI generates an "auth service" that is called via an HTTP request from the same codebase, ask whether this belongs at the Container level (separate deployment) or the Component level (just a module). If it is the latter, the HTTP call is unnecessary overhead.

### API versioning: why it matters and how it works

An API is a contract. Once clients depend on it, you cannot change it arbitrarily without breaking them. **API versioning** is the discipline of managing changes so that old clients continue to work while new clients use new features.

The three common versioning strategies:

**URL path versioning** — `GET /v1/users`, `GET /v2/users`. Simple and explicit. The version is visible in the URL. Downside: it looks like a new resource rather than a new version of the same resource, and some consider it not RESTful.

**Header versioning** — `GET /users` with `API-Version: 2024-01-01` header. Cleaner URL, but the version is hidden and must be documented. Clients can forget to send it.

**Query parameter versioning** — `GET /users?version=2`. Rarely used in practice but exists.

**The golden rule:** Once an API is in production and has clients, any breaking change requires a new version. A breaking change is:
- Removing a field from a response
- Changing the type of a field (string to number)
- Adding a required field to a request
- Changing the semantics of an existing field

A **non-breaking change:**
- Adding an optional field to a request
- Adding a field to a response
- Adding a new endpoint

AI frequently generates API changes that are breaking without versioning. When reviewing AI-generated API code, ask: "what do existing clients see?" If the answer is "a different response shape than before," you have a breaking change and need a new version.

### Idempotency in APIs

Idempotency means that calling an operation multiple times produces the same result as calling it once. This is not an academic property — it is essential for reliable distributed systems.

**Why it matters:** Network requests fail. When a request fails, the client retries. Without idempotency, a retry creates duplicate resources, duplicate charges, or duplicate state changes. With idempotency, the retry is safe.

**How it works in practice:** The client generates a unique **idempotency key** (a UUID) and sends it with the request in the `Idempotency-Key` header. The server stores the result for that key (typically for 24 hours). If the same key is received again, the server returns the cached result rather than reprocessing.

```
POST /orders
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890

{
  "amount": 5000,
  "currency": "usd",
  "customer": "cus_123"
}
```

The server sees the idempotency key on the first request, processes it normally, caches the response. If the client retries (because the first request timed out), the server sees the same key and returns the cached response without reprocessing.

**What AI gets wrong:** AI generates idempotency keys but does not use them on retries. Or it generates idempotency at the wrong level (using a resource ID as the idempotency key instead of a client-generated UUID). Or it treats idempotent GET requests as if they need idempotency keys (they do not — GETs are inherently idempotent by definition).

### Error contracts

An **error contract** is an agreed-upon format for error responses. Instead of arbitrary error messages, the API returns a structured error that clients can parse and handle programmatically.

```
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body is invalid",
    "details": [
      { "field": "email", "message": "Must be a valid email address" },
      { "field": "amount", "message": "Must be a positive integer" }
    ],
    "request_id": "req_a1b2c3d4"
  }
}
```

The `code` is a machine-readable string clients can switch on. `message` is human-readable. `details` is specific field-level validation errors. `request_id` lets the client reference the exact request when contacting support.

**Why AI gets this wrong:** AI generates error responses like `{ "message": "Something went wrong" }` or `{ "error": true }`. It omits the `code`, making it impossible for clients to handle specific error types programmatically. It returns different error shapes on different endpoints within the same API.

**The rule:** Pick one error response shape and use it consistently everywhere in your API. Make the `code` field specific (e.g., `RESOURCE_NOT_FOUND`, not `ERROR`).

### Event-driven architecture: the concept

In a **request-driven** architecture, a client sends a request and waits for a response. The entire operation happens in one synchronous transaction.

In an **event-driven** architecture, the system communicates by publishing and subscribing to events — discrete facts about what happened in the system. Instead of "call this service to do X," you publish "X happened" and any interested services react.

**Example:** An order is placed. Instead of the checkout service calling the inventory service, the email service, the analytics service, and the shipping service directly, it publishes an `order.placed` event. The inventory service, email service, analytics service, and shipping service each subscribe to that event and react independently.

**Benefits:**
- Loose coupling: the publisher does not know who subscribes
- Easy to add new subscribers without modifying the publisher
- Each service reacts to events at its own pace (asynchronous processing)
- Fault isolation: if one subscriber is down, others continue

**Drawbacks:**
- Eventual consistency: the system does not guarantee immediate consistency across services
- Complexity: tracking events across services, debugging distributed flows, handling duplicate events
- Ordering guarantees vary by broker (Kafka guarantees ordering within a partition; SQS does not)

**Where AI gets into trouble:** AI generates event-driven code that treats events as synchronous calls. It publishes an event and then immediately queries the result, expecting it to be available before any subscriber has processed it. Or it publishes events in an order that assumes all subscribers process at the same speed.

### CQRS at a conceptual level

CQRS (Command Query Responsibility Segregation) is the pattern of using a different model for reading data than for writing it.

In a conventional system, the same data model handles both reads and writes. `createUser()` and `getUser()` both touch the same `User` entity.

In CQRS:
- **Command side** — writes go to a specialized write model optimized for mutations (e.g., normalized structure, validation logic)
- **Query side** — reads come from a read model optimized for queries (e.g., denormalized views, materialized queries)
- The two are kept in sync, typically through events

**Why it exists:** For simple CRUD applications, a single model is fine. But when reads and writes have very different shapes, performance characteristics, or scaling needs, forcing them to share a model creates problems.

**Example:** A blog post. The write model stores content as markdown. The read model stores it as pre-rendered HTML (so reads are fast and require no transformation). The write model stores tags as a list. The read model stores tags as a separate lookup table (so filtering by tag is fast).

**When to consider CQRS:**
- Read and write views are radically different
- Read query patterns change frequently without changing business logic
- You need to scale reads and writes independently
- You are already using event-driven architecture (CQRS pairs naturally with it — the read model is updated by consuming events)

**When not to use CQRS:** Most applications. It adds significant complexity (two models to keep in sync, eventual consistency in reads, additional infrastructure). If you do not have a clear reason, a single model is simpler and correct.

### Auth Patterns

Authentication and authorization patterns determine how a system verifies identity and enforces permissions. Three concepts appear repeatedly in production systems.

**JWT (JSON Web Token)** — A token format with three parts: header (algorithm), payload (claims), and signature. The payload contains identity information that the server can verify cryptographically without storing session state. The risk: storing JWTs in localStorage exposes them to XSS attacks. The standard mitigation is to store the JWT in an HttpOnly cookie (inaccessible to JavaScript) and keep refresh tokens on the server for rotation. A compromised JWT should be short-lived (minutes to hours); the refresh token is longer-lived but rotatable after each use.

**OAuth** — Used when your app needs to act on behalf of a user with another service (delegated auth). The Authorization Code + PKCE flow is the standard for web and mobile apps: the user is redirected to an authorization server, which issues a code, which is exchanged for tokens. PKCE adds a proof key that prevents interception of the code exchange. Use OAuth when you need access to a third-party API on behalf of a user, not when you are building your own user authentication system.

**API Gateway** — Acts as the auth boundary at the edge of your system. It validates JWTs, checks tokens against the authorization server, and enforces rate limits per tenant before requests reach your business logic. Your services do not need to validate tokens — the gateway handles it. It also handles CORS, meaning your services do not need to implement CORS logic themselves.

### Hands-on exercise

**Goal:** Design and document a system using C4 levels, then identify three architectural mistakes in AI-generated code.

**Time:** 15 minutes

### Part 1: Model a system at all four C4 levels (8 minutes)

Create `architecture-c4.md`. For a hypothetical food delivery app, sketch all four levels in plain text.

**Level 1 — Context:**
```
Actors: Customer, Restaurant, Driver, Payment Provider
Systems: Food Delivery App, Restaurant POS, GPS Tracking, Payment Gateway
Trust boundaries: Customer uses HTTPS to the app; Payment provider has its own trust boundary
```

**Level 2 — Container:**
```
1. Web App (Browser SPA) — serves the customer interface
2. API Server (Node.js) — handles business logic
3. Database (PostgreSQL) — persistent storage
4. Message Broker (RabbitMQ) — async events between services
5. Notification Service (external) — sends SMS/email
```

**Level 3 — Component (inside API Server):**
```
Modules:
- OrderService: handles order creation, lifecycle
- RestaurantService: manages restaurant catalog and menus
- DriverService: tracks driver location and assignment
- PaymentService: coordinates with payment provider
- NotificationService: publishes events to notification service
```

**Level 4 — Code (key classes/functions):**
```
- OrderController: HTTP endpoints for orders
- OrderService.createOrder(): business logic
- OrderRepository: data access for orders
- DriverAssigner: algorithm for assigning drivers to orders
```

Spend 8 minutes writing this out. This exercise trains you to think at the right level of abstraction — essential when reviewing AI-generated code, which lives almost entirely at Level 4.

### Part 2: Spot the architectural mistakes (7 minutes)

Create `mistakes.js`. Review the following "AI-generated" code snippets and identify what is wrong with each one architecturally. Write your findings as comments.

```javascript
// mistakes.js

// MISTAKE 1: microservices for the wrong reasons
// Imagine this is AI-generated for a solo project with 50 users
const microservices = [
  { name: 'user-service', port: 3001, db: 'postgres_users' },
  { name: 'order-service', port: 3002, db: 'postgres_orders' },
  { name: 'notification-service', port: 3003, db: 'postgres_notifications' },
  { name: 'analytics-service', port: 3004, db: 'postgres_analytics' },
  { name: 'search-service', port: 3005, db: 'postgres_search' }
];
// What is wrong with this architectural decision?
// YOUR ANSWER: (write 2-3 sentences)


// MISTAKE 2: API breaking change without versioning
// AI modified this endpoint from returning { id, name } to returning { id, name, email }
// but the old clients still expect { id, name }
function getUser(req, res) {
  res.json({
    id: 1,
    name: 'Alice',
    email: 'alice@example.com'  // What problem does this cause?
  });
}
// YOUR ANSWER: (write 2-3 sentences)


// MISTAKE 3: wrong idempotency key
// AI wrote this code for retrying a payment
async function retryPayment(paymentId) {
  const response = await fetch('/payments', {
    method: 'POST',
    headers: { 'Idempotency-Key': paymentId, ... }
  });
  // What is wrong with using paymentId as the idempotency key?
// YOUR ANSWER: (write 2-3 sentences)


// MISTAKE 4: direct database access across "service" boundaries
// AI generated two services that each access each other's tables
// In user-service:
function getUserWithOrders(userId) {
  const user = db.query('SELECT * FROM users WHERE id = $1', userId);
  const orders = db.query('SELECT * FROM orders WHERE user_id = $1', userId); // accessing another service's table!
  return { user, orders };
}
// What is wrong here?
// YOUR ANSWER: (write 2-3 sentences)


// MISTAKE 5: synchronous event processing
// AI generated this code for an order placed event
async function onOrderPlaced(order) {
  await publish('order.placed', order);
  const inventory = await queryInventoryService(order.items); // waiting for result immediately
  await reserveInventory(inventory);
  await sendConfirmationEmail(order.customer);
}
// What is wrong with this pattern?
// YOUR ANSWER: (write 2-3 sentences)

console.log('Exercise: fill in YOUR ANSWER comments above');
console.log('Compare your answers against the Common Mistakes section');
```

Run it to confirm it executes:
```
node mistakes.js
```

Success criteria: File runs without syntax errors. Your written answers identify the core issue in each case (over-engineered microservices, breaking API change, wrong idempotency key, cross-service DB coupling, synchronous event assumption).

## Common mistakes

### 1. Choosing microservices for a project that needs a monolith

**What happens:** You deploy eight services. Each requires its own Docker image, deployment pipeline, monitoring configuration, and environment variables. A change that used to take one file edit now requires coordinating across five repositories. Engineers spend more time managing infrastructure than writing business logic.

**Why it happens:** Microservices are "what the big tech companies use." AI recommends them because they appear in architecture diagrams for systems you admire. The tradeoffs (complexity, operational overhead, network latency) are not visible in a diagram.

**How to avoid it:** Start with a monolith. Structure it into modules with clear boundaries. Only extract a service when you have a specific, documented reason (a specific scaling need, a specific fault isolation requirement, a specific team boundary). If you cannot name the reason, do not extract.

### 2. Breaking backward compatibility without API versioning

**What happens:** You change a response field. Existing mobile app users on version 2.3 get crashes or blank screens because their code expected the old field shape. You scramble to either revert the change or rush out a new API version.

**Why it happens:** AI modifies the code to add a new field or change a type without thinking about existing clients. The server runs the new code, returns the new shape, and the error only surfaces with real clients.

**How to avoid it:** Every change to a public API response requires a version bump. Document your versioning strategy before you need it. Use deprecation headers (`Deprecation: true`, `Sunset: Sat, 01 Jan 2028 00:00:00 GMT`) to give clients notice before you remove old versions.

### 3. Sharing a database across "microservices"

**What happens:** Service A and Service B each have their own process and deployment, but both connect to the same PostgreSQL instance and query each other's tables directly. Service A's schema change breaks Service B. You cannot deploy Service B independently without coordinating with Service A.

**Why it happens:** AI generates services with database connections without specifying that each service owns its data. It is the path of least resistance — one database is simpler than managing multiple.

**How to avoid it:** Each service owns its data. If Service A needs data from Service B, it calls an API provided by Service B — it does not query Service B's tables directly. This is the data ownership rule: **services do not share databases**.

### 4. Not handling eventual consistency in event-driven systems

**What happens:** You publish an event, then immediately query for the result. The subscriber has not processed the event yet, so the query returns stale or missing data. You call the query again. The data is still not there. You conclude there is a bug.

**Why it happens:** AI treats events as synchronous calls. Publish an event, expect it to be processed by the time the next line executes. In event-driven systems, processing is asynchronous by design.

**How to avoid it:** When you publish an event and need a result, you need a different pattern: either wait for a callback/ webhook from the subscriber, poll for the result with a timeout, or use a request/response pattern over the message broker (rare). Never assume immediate processing.

### 5. Using the wrong idempotency key

**What happens:** A payment retry uses the `paymentId` as the idempotency key. On the first attempt, the server creates a new payment and returns `paymentId: "pay_abc"`. On retry, the same idempotency key is sent, so the server returns the cached result — but the cached result does not reflect that the first attempt actually failed on the network. The user is charged once but thinks the payment failed.

**Why it happens:** AI uses the resource ID as the idempotency key. The resource ID is only available after the first successful creation, so it cannot be used for the first request.

**How to avoid it:** The idempotency key must be generated client-side **before** the first request, as a randomly generated UUID. Never use a resource ID as an idempotency key.

## AI-specific pitfalls

### AI recommends microservices for projects that should be monoliths

AI will recommend microservices architecture when asked about "scalability" or "enterprise patterns" even for small projects. It has learned that microservices appear in architecture diagrams for complex systems, so it associates them with "advanced" or "professional."

What to look for: Any AI-generated architecture description that includes more than 3-4 services for a project without significant scale, multiple independent teams, or clearly different operational requirements. For a solo project or small team, a modular monolith is almost always the right answer.

### AI does not design service boundaries coherently

AI will split a system along technical lines (auth service, database service, email service) rather than business capability lines (order management, driver dispatch, restaurant catalog). Technical splits create tight coupling through the network because related data is spread across services.

What to look for: Services named after technical components (database, cache, queue) rather than business capabilities. Ask AI to explain why a particular boundary exists. If the answer is "because it is a different technology," the boundary is probably wrong.

### AI generates APIs without backward compatibility

AI will modify an API endpoint to add, remove, or change fields based on new requirements without versioning. It does this because it is working in a context where it does not know who depends on the old contract.

What to look for: Any modification to an existing endpoint's request or response shape. When AI modifies a field in a response object, ask: "what breaks for existing clients?" If the answer is "anything relying on this field," you need a new API version.

### AI generates code that assumes synchronous responses in event-driven systems

AI will write event publishers and immediately query for results, or write consumers that process events and return responses over the same channel. It does not model the asynchrony.

What to look for: Code that publishes an event and then awaits a result in the same function. Event-driven means eventual — the publisher cannot wait for subscribers to complete. If you see `await` after a publish, verify that it is awaiting something the broker returns (a delivery acknowledgment), not the result of subscriber processing.

### AI generates service discovery or network configuration without explaining it

AI will generate Kubernetes service definitions, Docker Compose files, or network configurations that include service names as hostnames without explaining how service discovery works or what the network configuration means. The code "looks right" but fails in deployment because the network assumptions are wrong.

What to look for: Any AI-generated network configuration (Docker Compose, Kubernetes manifests, environment variable service URLs). Verify that service names resolve correctly in your deployment environment. Ask AI to explain how services discover each other.

### AI puts auth logic inside your service instead of at the edge

AI will generate code where your API server validates JWTs directly — checking signatures, expiry, claims — before processing business logic. This mixes auth concerns with business logic and requires every service to have access to signing keys or public keys. The correct pattern is to validate auth at the API gateway once, then pass identity information in a header (e.g., `X-User-ID`) to your services. Services trust that header because the gateway already validated the token.

What to look for: JWT validation code (`jwt.verify`, `jwt.decode`) inside service-level route handlers. Ask: "Does this service need to know about signing keys?" If not, the gateway should be handling this.

### AI generates JWTs with weak or placeholder signatures

AI will produce JWT implementation code where the signature verification uses `algorithm: 'none'` or a hardcoded weak secret like `"secret"` or `"your-secret-key"`. This completely undermines JWT security — anyone can forge tokens. Production JWT implementations must use strong asymmetric algorithms (RS256, ES256) or at minimum HS256 with a properly generated secret.

What to look for: JWT code that uses `algorithm: 'none'`, hardcoded secret strings in source code, or comments saying "replace with real secret." Verify the algorithm and secret management approach before deploying anything AI-generated that touches auth.

## Quick reference

### Architecture pattern selection

```
Small team (< 5 engineers), early stage?
├─ YES → Monolith (start here, stay here as long as you can)
└─ NO ↓

Growing team, separate module ownership needed?
├─ YES → Modular Monolith (clear module boundaries, single deploy)
└─ NO ↓

Large team (> 20 engineers), independent scaling, strict fault isolation?
├─ YES → Microservices (only if you have operational maturity)
└─ NO → Modular Monolith
```

### API versioning decision tree

```
Adding a field to response (optional)?
├─ YES → Non-breaking, no version bump needed
└─ NO ↓
Removing a field / changing type / changing semantics?
├─ YES → Breaking change: new API version required
└─ NO ↓
New required field in request?
├─ YES → Breaking change: new API version
└─ NO → Non-breaking change
```

### Idempotency key rules

```
1. Always generate client-side (UUID) BEFORE the first request
2. Never use a server-assigned resource ID as the idempotency key
3. Include it in POST and PATCH requests (creates/updates)
4. GET and DELETE requests do not need idempotency keys (inherently idempotent)
5. Store idempotency results for at least 24 hours
```

### C4 level quick reference

| Level | Question | Example |
|---|---|---|
| Context | What system is this, who are the actors? | Customer, Restaurant, Driver |
| Container | What processes/deployable units? | Web SPA, API Server, DB, Message Broker |
| Component | What modules inside each container? | OrderService, DriverService, PaymentService |
| Code | What classes/functions? | OrderController, createOrder(), DriverAssigner |

### Event-driven checklist

```
Before publishing an event:
[ ] All subscribers identified (at least conceptually)
[ ] Event schema is stable (will not break subscribers)
[ ] Consumers understand it is async (no immediate result assumption)

When consuming events:
[ ] Handle duplicate events (idempotent processing)
[ ] Handle out-of-order delivery (add sequence/timestamp checks if ordering matters)
[ ] Do not reply on the event channel — use a separate response mechanism
```

## Go deeper

- [C4 Model — Simon Brown](https://c4model.com/) (verified 2026-04) — The authoritative resource on C4. Simon Brown invented the technique. The site has diagrams, examples, and tools for creating C4 models.
- [Martin Fowler — Monolith First](https://martinfowler.com/articles/darksonian-fallacy.html) (verified 2026-04) — Famous essay on why most new projects should start as monoliths. Required reading before adopting microservices.
- [API Versioning Strategies (Stripe)](https://stripe.com/docs/api versioning) (verified 2026-04) — Stripe's approach to API versioning, considered a gold standard. Includes discussion of breaking vs. non-breaking changes.
- [Event-Driven Architecture — Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html) (verified 2026-04) — Clear conceptual introduction to event-driven architecture, Commands vs. Events, and the tradeoffs involved.
- [CQRS — Martin Fowler](https://martinfowler.com/articles/cqrs.html) (verified 2026-04) — The definitive article on CQRS. Explains the pattern, when it makes sense, and when it is overkill.
