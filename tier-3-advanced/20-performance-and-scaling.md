# Performance and Scaling

## What is it?

Performance is how fast your code runs and how many resources it consumes. Scaling is how your system behaves as load increases — whether adding users, data, or requests pushes it to breaking point or keeps it running smoothly.

Think of it like a restaurant kitchen. Performance is whether the chef can cook a dish in 10 minutes or 45. Scaling is whether the kitchen can handle 10 orders an hour or 500 without the whole operation collapsing. A well-tuned system is both fast and ready to grow.

In web applications, the core resources to measure are:

- **CPU** — how much processing power is being used. High CPU means your code is compute-bound.
- **Memory (RAM)** — how much data is being held in flight. High memory usage means you're either buffering a lot of data or leaking memory somewhere.
- **I/O** — input/output, meaning disk reads/writes and network calls. Slow I/O is the most common bottleneck in real applications, because disk and network are orders of magnitude slower than CPU.

**Latency** is the time for a single operation (e.g., "this API call takes 200ms"). **Throughput** is how many operations you can handle per second (e.g., "this server handles 500 requests/second"). A system can have low latency but low throughput (a single fast worker), or high latency but high throughput (many parallel workers). Know which one matters for your use case.

## Why it matters for vibe coding

AI coding tools are generous. They will generate code that works for the example in your prompt — a list of 10 items, a single user, a tiny dataset. But they have no idea what your production load looks like. Without performance knowledge, you will ship code that:

- Makes a database query for every item in a list instead of loading them all at once (the N+1 problem)
- Creates a new database connection for every request instead of reusing a pool
- Sends the same slow query every time a page loads instead of caching the result
- Holds an entire dataset in memory when you only needed a summary
- Scales vertically forever when you should be distributing load horizontally

These patterns are invisible in development. A page loads fine with 5 users. Then you ship, get 500 users, and your database connection limit freezes every request.

## The 20% you need to know

### Profiling basics

Before you optimize anything, measure. A profile tells you where time is actually being spent, as opposed to where you think it is. The three types that cover most situations:

**CPU profiling** shows which functions are consuming the most CPU time. In Node.js, use the built-in `--prof` flag or `clinic.js`. In Python, `cProfile` or `py-spy`. In Go, `pprof`. The output is usually a sorted list of functions with time percentages.

**Memory profiling** shows which code paths allocate the most memory. Memory leaks (objects kept alive unintentionally) are the most dangerous — they cause gradual slowdown until your process crashes. Heap profilers show live object sizes and allocation sites.

**I/O profiling** shows how much time is spent waiting on network or disk. The critical insight: if your code is spending 80% of its time waiting on I/O, no amount of CPU optimization will help. You need to reduce I/O (caching, batching) or parallelize it.

The practical workflow: **symptom first, profile second**. Is the page slow? Measure the request. Is the database slow? Measure queries. Don't guess, don't optimize theoretical bottlenecks.

### Caching strategies

Caching is storing the result of an expensive operation so you don't have to repeat it. There are four main layers, each at a different distance from your code:

| Layer | Latency | Scope | Best for |
|---|---|---|---|
| In-memory (process) | ~0.1ms | Single instance | Request-scoped data, computed values used multiple times in one request |
| Redis/Memcached | ~1ms | Shared across instances | Session data, API responses, frequently-read database results |
| CDN | ~10-50ms | geographically distributed | Static assets, public API responses with long expiry |
| HTTP cache headers | ~0ms (no round trip) | Browser/client | Any response that is identical for all users and changes rarely |

Cache invalidation is the hard part. The rule: **cache for the minimum time that gives you a meaningful speedup**. If your data changes every second, a 60-second cache is wrong. If your data changes every week, a 5-minute cache is fine.

The most common cache patterns:

- **Cache-aside** (lazy loading): your code checks the cache first; if miss, reads from source and populates cache. Good for read-heavy workloads.
- **Write-through**: writes go to cache and source simultaneously. Ensures cache is always fresh but slows down writes.
- **Cache-aside with TTL**: cached data expires after a time-to-live. Simpler than write-through, used when eventual consistency is acceptable.

### The N+1 problem

This is the most common database performance killer in ORMs and AI-generated code. Here's the pattern:

```python
# Bad: N+1 query problem
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
    # This runs ONE query per user — 101 queries total!
    user.posts = db.query(f"SELECT * FROM posts WHERE user_id = {user.id}")
```

```python
# Good: fetch in one query with a JOIN or WHERE IN
users = db.query("SELECT * FROM users LIMIT 100")
user_ids = [u.id for u in users]
posts = db.query(f"SELECT * FROM posts WHERE user_id IN ({user_ids})")
posts_by_user = defaultdict(list)
for post in posts:
    posts_by_user[post.user_id].append(post)
for user in users:
    user.posts = posts_by_user[user.id]
```

Or with an ORM using eager loading:

```python
# Eager loading: tells the ORM to fetch posts in one query
users = db.query("SELECT * FROM users LIMIT 100").include("posts")
```

N+1 also happens with:
- Loading a list, then fetching each related object individually
- Pagination where each row triggers a lookup
- APIs that call the database in a loop

The fix is always the same: **batch your reads** — use JOINs, `WHERE IN`, or eager loading to fetch everything in 1-2 queries instead of N.

### Database query optimization

**EXPLAIN ANALYZE** is your most important tool. Every major database (PostgreSQL, MySQL, SQLite) supports it. You write `EXPLAIN ANALYZE` before your query and the database shows you its execution plan — whether it scans the whole table, uses an index, joins in a nested loop, etc.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 42
AND created_at > '2024-01-01';
```

Key things to look for in an EXPLAIN output:
- **Seq Scan** on a large table — means it's reading every row. Usually needs an index.
- **Index Scan** — good. It's using an index to find rows directly.
- **Nested Loop** — may be fine for small datasets, but can be catastrophic for large joins without indexes.
- **Hash Join** vs **Merge Join** — hash joins are usually fine; nested loops in join position are the red flag.

Beyond EXPLAIN, basic index hygiene:
- Add an index on columns you frequently filter with `WHERE`, `ORDER BY`, or join on with `JOIN ON`
- Composite indexes (on multiple columns) are used for queries that filter on all columns in order, or the leftmost prefix
- Don't index everything — writes become slower because the index must be updated

### Connection pooling

Every database connection consumes memory and CPU on both the client and server. Creating a new connection per request (no pooling) means:

- Connection setup overhead on every request (50-200ms of latency)
- Server running out of connections under load (PostgreSQL default limit is ~100)

A **connection pool** maintains a set of open connections and reuses them across requests. Your code borrows a connection from the pool, uses it, and returns it — it never creates or closes a raw connection.

Pool settings to know:
- **Pool size** — typically 5-20 connections per application instance. Match to your database's `max_connections`.
- **Idle timeout** — connections closed if unused for this long. Prevents stale connections.
- **Connection timeout** — how long to wait for a connection from the pool before giving up.

In Node.js with `pg`: `new Pool({ max: 20, idleTimeoutMillis: 30000 })`
In Python with `psycopg2`: `pool = psycopg2.pool.ThreadedConnectionPool(5, 20, dsn)`

### Horizontal vs. vertical scaling

**Vertical scaling** (scaling up): make the existing machine more powerful. More CPU, more RAM. Simple, no code changes. You hit a ceiling — one machine can only get so big.

**Horizontal scaling** (scaling out): add more machines. Multiple instances of your application running behind a load balancer. More complex — your application must be **stateless** (no data stored in process memory that other instances need).

The rule: **scale vertically first, horizontally second**. Vertical scaling handles a lot before you need to distribute load. Horizontal scaling is for when you've maxed out a single machine or need high availability (surviving a machine failure).

When you horizontal scale, a **load balancer** distributes incoming requests across your instances. It also enables **health checks** — routing traffic away from crashed or hung instances. Common choices: nginx, HAProxy, cloud load balancers (AWS ALB, Cloudflare), or managed options like Railway/Render/Koyeb.

**Stateless design** means: store session data, user state, and application data in a database or cache — not in process variables. If instance A stores a user's session in a local variable and the next request hits instance B, that user is logged out. This is a direct failure mode of AI-generated code that stores auth tokens or user data in module-level variables.

## Hands-on exercise

**Profile a slow endpoint and fix an N+1 query**

Time: 15 minutes. You need Node.js installed.

Setup:
```bash
mkdir perf-exercise && cd perf-exercise
npm init -y
npm install sqlite3 express
```

Create `server.js`:
```javascript
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const app = express();

// Setup in-memory database
const db = new sqlite3.Database(':memory:');
db.serialize(() => {
  db.run('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');
  db.run('CREATE TABLE orders (id INTEGER PRIMARY KEY, user_id INTEGER, amount REAL)');
  // Insert 100 users, each with 5 orders
  for (let i = 1; i <= 100; i++) {
    db.run(`INSERT INTO users (id, name) VALUES (${i}, 'User ${i}')`);
    for (let j = 0; j < 5; j++) {
      db.run(`INSERT INTO orders (user_id, amount) VALUES (${i}, ${Math.random() * 100})`);
    }
  }
});

// BAD endpoint: N+1 query
app.get('/bad', (req, res) => {
  const start = Date.now();
  db.all('SELECT * FROM users LIMIT 20', (err, users) => {
    let queryCount = 1; // first query
    const results = [];
    let pending = users.length;
    users.forEach(user => {
      db.all(`SELECT * FROM orders WHERE user_id = ${user.id}`, (err, orders) => {
        queryCount++;
        results.push({ ...user, orders });
        pending--;
        if (pending === 0) {
          res.json({ users: results, queries: queryCount, ms: Date.now() - start });
        }
      });
    });
  });
});

// GOOD endpoint: single query with JOIN
app.get('/good', (req, res) => {
  const start = Date.now();
  db.all(`
    SELECT u.id as user_id, u.name, o.id as order_id, o.amount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id IN (SELECT id FROM users LIMIT 20)
  `, (err, rows) => {
    // Group results in memory
    const userMap = {};
    for (const row of rows) {
      if (!userMap[row.user_id]) {
        userMap[row.user_id] = { id: row.user_id, name: row.name, orders: [] };
      }
      if (row.order_id) {
        userMap[row.user_id].orders.push({ id: row.order_id, amount: row.amount });
      }
    }
    const users = Object.values(userMap);
    res.json({ users, queries: 1, ms: Date.now() - start });
  });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

Run it:
```bash
node server.js
```

In another terminal:
```bash
curl http://localhost:3000/bad
curl http://localhost:3000/good
```

**Expected output:**
- `/bad` returns with `queries: 21` (1 + 20) and slower ms
- `/good` returns with `queries: 1` and faster ms

The difference in query count is the smoking gun. The `/bad` endpoint makes 21 database round-trips; `/good` makes 1. For 20 users with 5 orders each, that's the N+1 pattern in action.

## Common mistakes

**1. Adding an index without measuring first**

What happens: you add an index, it makes writes slower, and the query you're trying to fix is still slow because the real problem is something else (锁 contention, wrong query plan, network latency).

How to fix: always run EXPLAIN ANALYZE before and after adding an index. Confirm the index is actually being used.

**2. Caching data that changes constantly**

What happens: users see stale data, you spend hours debugging "why didn't the order update?" only to find a 5-minute cache TTL that made it look like nothing happened.

How to fix: set cache TTL based on how often the data changes, not how fast you want the cache to be. If in doubt, cache less.

**3. Storing session or auth state in process memory**

What happens: your app works on your laptop with one instance. You deploy with 3 instances behind a load balancer and users get logged out randomly because instance A doesn't know about instance B's session.

How to fix: store sessions in Redis, database, or a signed JWT cookie. Never in module-level variables or process memory that isn't shared.

**4. Ignoring I/O in async code**

What happens: you write `async` functions but inside them, you use synchronous library calls that block the event loop. Your app appears async but still hangs under load.

How to fix: use async-native libraries (e.g., `async`/`await` database drivers, not the sync versions). Profile the event loop if requests are queueing up unexpectedly.

**5. Connection pool set too high**

What happens: you set pool size to 100, deploy 10 instances, and now you have 1000 potential connections but your database only allows 200. Connections pile up waiting, queries timeout.

How to fix: `max_connections_per_instance = floor(database_max_connections / number_of_instances)`. Leave headroom for admin connections and temporary spikes.

## AI-specific pitfalls

**AI generates N+1 queries without knowing it**

AI loves ORMs and will happily generate code that loops over a list and accesses a related object on each item — the classic N+1. The code works perfectly with 5 items in your test prompt. In production with 10,000 items, you get 10,001 database queries.

When reviewing AI-generated database code: always ask "how many queries does this make?" If there's a loop over database objects and a query inside it, that's N+1. Look for `.include()`, `.eager_load()`, or explicit JOINs instead.

**AI doesn't understand why connection pooling matters**

AI will often generate code that creates a new database connection per request because the code "looks cleaner" and the AI has no concept of your database's connection limit. Watch for `new DbConnection()` or equivalent inside request handlers. Connections should be created once at application startup and reused via a pool.

**AI uses caching indiscriminately**

AI will sometimes suggest caching for everything, including data that changes every second. You'll find cache TTLs of 300 seconds on a real-time inventory count, or caching the result of an auth check. Before implementing any AI-suggested cache, ask: "how often does this data actually change?"

**AI ignores EXPLAIN ANALYZE**

When asked to optimize a slow query, AI will often suggest adding an index, changing the JOIN order, or rewriting the query based on intuition. It will rarely say "run EXPLAIN ANALYZE first." Always verify AI's optimization advice with actual query plans.

**AI stores data in module-level variables thinking it's a singleton**

AI might generate something like `let cachedData = null; app.get('/data', (req, res) => { if (!cachedData) cachedData = fetchFromDb(); res.json(cachedData); })`. This works on a single instance but causes cache stampedes under load and breaks with horizontal scaling. If you see module-level mutable caches, treat them as a red flag.

## Quick reference

**Which cache layer to use?**

```
Does the data change per user or per request?        → No cache (compute fresh)
Does the data change rarely and is the same for all users?
  Static assets (images, CSS, JS)?                  → CDN + HTTP cache headers
  Computed values, API responses?                   → Redis/Memcached
Does the data change frequently but is per-user?
  Session data?                                     → Redis
  User-specific computed views?                     → In-memory (with short TTL) or Redis
Does the data need to be fresh?
  Never cache (or cache-aside with TTL of seconds) → compute fresh
```

**N+1 checklist — look for these in AI output:**

```
❌ for (user in users) { db.query(`WHERE user_id = ${user.id}`) }
❌ for (item in items) { item.category = db.query(`SELECT * FROM categories WHERE id = ${item.category_id}`) }
✅ users = db.query("SELECT * FROM users").include("posts")
✅ db.query("SELECT * FROM orders WHERE user_id IN (...)")
```

**EXPLAIN ANALYZE red flags:**

```
Seq Scan on large table  → needs index
Nested Loop on large sets → needs index or hash join
High actual rows vs estimated → statistics may be stale (ANALYZE)
```

**Connection pool sizing:**

```
database_max_connections = 100
app_instances = 4
pool_per_instance = 20 (leaving ~20 headroom)
```

## Go deeper

- **PostgreSQL EXPLAIN ANALYZE tutorial** — official documentation covering all plan node types and how to read them. [postgres docs](https://www.postgresql.org/docs/current/using-explain.html) (verified 2026-04)
- **Redis caching patterns** — Redis University's free course covers cache-aside, write-through, and invalidation strategies with real examples. [Redis U](https://university.redis.com/courses/ru204/) (verified 2026-04)
- **N+1 problem in Rails/ActiveRecord** — a thorough walkthrough of the problem, detection tools, and solutions (applies to any ORM). [HDNET Blog](https://blog.hdnet.com/development/n-plus-one-problem) (verified 2026-04)
- **Connection pools explained** — Django docs section on connection pooling, database-backend agnostic concepts. [Django docs](https://docs.djangoproject.com/en/5.0/topics/db/connections/) (verified 2026-04)
- **Horizontal vs. vertical scaling** — Cloudflare's simple explanation of scaling strategies with when-to-use guidance. [Cloudflare](https://www.cloudflare.com/learning/performance/types-of-scaling/) (verified 2026-04)
