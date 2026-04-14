# APIs and Integrations

## What is it?

An API (Application Programming Interface) is a contract between two pieces of software: one side makes a request in a defined format, the other side responds with defined data. You use APIs constantly — every time your app fetches data from a server, logs in via Google, receives a payment webhook from Stripe, or queries an AI model, you are interacting with an API.

Integrations are the practice of connecting your application to external services through their APIs. The external service handles some concern (payments, email, AI, authentication) so you do not have to build it yourself.

The three dominant API styles you will encounter:

**REST (Representational State Transfer)** uses URLs to address resources and HTTP methods to describe actions. `GET /users/123`, `POST /users`, `DELETE /users/123`. It is the most common style by far. REST is a convention, not a standard — APIs that call themselves REST vary wildly in how well they follow the conventions.

**GraphQL** lets the client specify exactly what data it wants and get everything in one request. Instead of `GET /users/123` and `GET /users/123/orders` returning two separate responses, a GraphQL query fetches both in a single round trip. It is powerful but adds complexity on both client and server.

**gRPC (Google Remote Procedure Call)** uses Protocol Buffers for strongly-typed, binary serialization and HTTP/2 for multiplexed connections. It excels for internal service-to-service communication where performance matters. You will encounter it less often in external APIs but increasingly inside modern microservice architectures.

## Why it matters for vibe coding

When AI generates integration code, it frequently produces code that is structurally correct but contextually wrong: it assumes REST conventions the API does not follow, uses the wrong OAuth2 flow for the architecture, ignores rate limits until production fails at 3am, or stores credentials in plaintext. These are not edge cases — they are the default failure modes of AI-generated integration code.

Specific failure modes you will encounter:

- **AI calls a rate-limited API in a loop.** The AI generates a batch processing script that makes 1,000 API calls in rapid succession. The API responds with 429s. The script fails silently or retries in a tight loop, compounding the problem.
- **AI stores an API key as a string literal in source code.** The key gets committed to version control, appears in error logs, and is rotated out. Now every environment is broken.
- **AI generates webhook handler code that does not validate signatures.** Your endpoint accepts any incoming request. An attacker can now flood your system with fake webhook events.
- **AI generates a GraphQL query that works on one API version but not another**, or queries fields that do not exist in the schema, because it hallucinated the schema.
- **AI generates REST endpoint URLs that look reasonable but do not match the actual API.** The AI invents `/api/listUsers` when the real endpoint is `/api/users`.
- **AI uses the wrong pagination strategy.** The API uses cursor-based pagination; the AI generates offset-based page logic that returns duplicates or skips records.

## The 20% you need to know

### REST deeply: idempotency and resource modeling

REST is not just "URLs + JSON." The most important concept is **idempotency** — whether making the same request multiple times produces the same result.

**Idempotent methods:** GET, PUT, DELETE. Calling `DELETE /users/123` twice should return the same outcome (the user is gone). Calling `PUT /users/123` with the same body twice should leave the resource in the same state.

**Non-idempotent methods:** POST, PATCH. `POST /orders` creates a new order each time. `PATCH /users/123` modifies the user, but the result depends on the current state.

This distinction matters for **retries**. If a `DELETE` request times out, you can safely retry it. If a `POST` request times out, retrying may create a duplicate resource. Production-grade API clients use **idempotency keys** — a unique token in the request header that tells the server "this is a retry of the same logical request, do not create a second resource."

**Resource modeling:** A well-designed REST API models resources as nouns, not verbs. Avoid action-oriented URLs like `/api/getUser` or `/api/createOrder`. Use `/api/users` and `/api/orders`. Verbs are expressed through HTTP methods: `GET /users`, `POST /orders`.

### API authentication: keys, tokens, and HMAC

There are three broad patterns you will see.

**API Keys** are long random strings that identify the calling client. They are simple: pass the key in a header or query parameter, and the server validates it. API keys are appropriate for server-to-server calls where the client is trusted and the key can be kept secret.

The problem: API keys do not expire by default. If one leaks, you have to rotate it everywhere. They also give full access — there is no scoped permission.

**OAuth2 Tokens** are short-lived credentials that represent a user's authorization. OAuth2 is complex because it has multiple **flows** for different architectures:

- **Authorization Code + PKCE** is the correct flow for browser-based apps (SPAs) and mobile apps. The user logs in via a redirect, the app receives a code, exchanges it for tokens. PKCE (Proof Key for Code Exchange) prevents authorization code interception. This is the flow you should default to for any client-side application.
- **Client Credentials** is for machine-to-machine (M2M) communication where there is no user — your server talks to another server directly. Use this for background jobs, cron tasks, and service-to-service calls where no human is involved.
- **Implicit flow** (older, phased out) and **Resource Owner Password Credentials** (only for legacy migration) are not recommended.

Tokens expire. Access tokens typically last minutes to hours. Refresh tokens allow obtaining new access tokens without re-authenticating. Your code must handle token expiration and refresh.

**HMAC (Hash-based Message Authentication Code)** is used when both client and server share a secret key. The client signs the request with the key; the server verifies the signature. This is common in webhook signature validation (e.g., Stripe's webhook signatures use HMAC-SHA256). The server can verify that the request came from the expected sender and was not tampered with in transit.

### Webhooks and how to validate them

A webhook is an API endpoint you expose so an external service can push events to you. Instead of your code polling an API every 30 seconds ("any new orders?"), the external service calls your URL when something interesting happens ("order #4567 was just placed").

The challenge: anyone can POST to your webhook URL. Without validation, an attacker can flood your system with fake events. Every webhook from a reputable service is **signed**.

The standard pattern: the service signs the request body using HMAC and includes the signature in a header (e.g., `Stripe-Signature`, `X-Hub-Signature-256`). Your server reconstructs the signature from the raw request body and compares it. If they match, the request is legitimate.

**Critical:** You must verify the signature **before** processing the body. If you process the body first and validate second, an attacker can send a valid-looking but fake payload that your code partially processes before validation fails.

### Rate limiting and backoff strategies

APIs limit how many requests you can make in a given time window. When you hit the limit, the API returns `429 Too Many Requests`. Common rate limit headers:

- `X-RateLimit-Limit` — total requests allowed in the window
- `X-RateLimit-Remaining` — requests left in current window
- `X-RateLimit-Reset` — Unix timestamp when the window resets
- `Retry-After` — seconds to wait (returned with 429)

**Exponential backoff with jitter** is the standard retry strategy:

```
delay = min(max_delay, base_delay * 2^attempt + random_jitter)
```

Instead of retrying immediately after a 429, wait longer each time. After 1 second, then 2, then 4, then 8. Add randomness (jitter) so multiple clients do not all retry at the same instant and create a thundering herd.

Most SDKs handle this automatically. When using raw HTTP, you must implement it yourself.

### Pagination: cursor vs offset

**Offset-based pagination** uses `?page=2&per_page=50`. You skip the first 50 records and take the next 50. Simple, but has a serious flaw: if records are inserted or deleted between page requests, you get **duplicate records** (if records were inserted before your current page) or **skipped records** (if records were deleted). Offset pagination is also slow for large datasets — skipping 10,000 records requires the database to read and discard them.

**Cursor-based pagination** uses `?after=cursor_value` instead of page numbers. The cursor is typically an opaque pointer to a position in the dataset (the last seen ID, a timestamp, or an encoded position). The API returns records after that cursor. Cursors do not have the duplicate/skip problem because they point to positions, not page numbers. They are also efficient at scale.

Most modern APIs (Stripe, GitHub, Slack) use cursor-based pagination. When AI generates pagination code, it almost always uses offset-based pagination because it is more intuitive to humans. Check which your API uses.

### SDK vs raw HTTP

**SDKs** wrap an API in a library with typed objects, auto-generated from the API schema. They handle auth, serialization, retries, and pagination for you. When an SDK exists and is well-maintained, use it. The productivity gain is substantial.

**Raw HTTP** (using `fetch`, `axios`, `requests`, etc.) means you handle everything yourself. Use raw HTTP when:
- No SDK exists for your language or framework
- The SDK is outdated or has bugs you need to work around
- You need only a small fraction of the API and the SDK would add bulk
- You are vibe coding and the AI can write the HTTP calls directly

The tradeoff: SDKs give you correctness and convenience; raw HTTP gives you flexibility and fewer dependencies.

### OpenAPI specs

An **OpenAPI Specification** (OAS, formerly Swagger) is a machine-readable description of a REST API — endpoints, parameters, request/response shapes, authentication requirements. It is the contract that lets tools generate SDKs, documentation, mock servers, and API explorers automatically.

If you are integrating with a well-documented API, it likely has an OpenAPI spec you can use to validate AI-generated code or generate a type-safe client. GitHub, Stripe, Slack, and most major APIs publish public OpenAPI specs.

## Hands-on exercise

**Goal:** Validate a webhook signature, implement a retry with exponential backoff, and call a paginated API using cursor pagination.

**Time:** 15 minutes

### Part 1: Verify a webhook signature (5 minutes)

Create `webhook-verify.js`. This simulates validating a Stripe-like webhook signature using HMAC-SHA256.

```javascript
// webhook-verify.js
// Simulates HMAC-SHA256 signature verification like Stripe uses

const crypto = require('crypto');

const WEBHOOK_SECRET = 'whsec_test_secret_key_12345';

function computeSignature(payload, secret) {
  return crypto
    .createHmac('sha256', secret)
    .update(payload, 'utf8')
    .digest('hex');
}

function verifyWebhookSignature(rawBody, signatureHeader, secret) {
  // Signature format: "t=timestamp,v1=signature"
  const parts = signatureHeader.split(',');
  const timestampPart = parts.find(p => p.startsWith('t='));
  const signaturePart = parts.find(p => p.startsWith('v1='));

  if (!timestampPart || !signaturePart) {
    return false;
  }

  const timestamp = timestampPart.split('=')[1];
  const receivedSig = signaturePart.split('=')[1];

  // Verify the signature matches the payload
  const payloadWithTimestamp = `${timestamp}.${rawBody}`;
  const expectedSig = computeSignature(payloadWithTimestamp, secret);

  // Use timing-safe comparison to prevent timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(receivedSig, 'hex'),
    Buffer.from(expectedSig, 'hex')
  );
}

// Test with a real payload
const payload = JSON.stringify({ type: 'order.created', data: { id: 'order_123' } });
const timestamp = Math.floor(Date.now() / 1000).toString();
const signatureHeader = `t=${timestamp},v1=${computeSignature(`${timestamp}.${payload}`, WEBHOOK_SECRET)}`;

console.log('Testing VALID signature...');
const valid = verifyWebhookSignature(payload, signatureHeader, WEBHOOK_SECRET);
console.log('Valid:', valid); // Should print: true

console.log('\nTesting TAMPERED payload...');
const tamperedPayload = JSON.stringify({ type: 'order.created', data: { id: 'order_999' } });
const tampered = verifyWebhookSignature(tamperedPayload, signatureHeader, WEBHOOK_SECRET);
console.log('Tampered:', tampered); // Should print: false

console.log('\nTesting WRONG secret...');
const wrong = verifyWebhookSignature(payload, signatureHeader, 'wrong_secret');
console.log('Wrong secret:', wrong); // Should print: false
```

Run it:
```
node webhook-verify.js
```

Success criteria: Valid signature returns `true`, tampered payload and wrong secret both return `false`.

### Part 2: Retry with exponential backoff (5 minutes)

Create `backoff.js`. This implements exponential backoff with jitter and tests it against a mock that returns 429 twice before succeeding.

```javascript
// backoff.js
// Demonstrates exponential backoff with jitter

function randomJitter(maxMs = 1000) {
  return Math.floor(Math.random() * maxMs);
}

async function retryWithBackoff(fn, options = {}) {
  const {
    maxAttempts = 5,
    baseDelayMs = 500,
    maxDelayMs = 30000
  } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const result = await fn(attempt);
      console.log(`Attempt ${attempt}: succeeded`);
      return result;
    } catch (err) {
      if (err.status === 429 && attempt < maxAttempts) {
        const delay = Math.min(maxDelayMs, baseDelayMs * Math.pow(2, attempt - 1)) + randomJitter();
        console.log(`Attempt ${attempt}: received 429. Retrying in ${Math.round(delay)}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        console.log(`Attempt ${attempt}: failed with ${err.status || 'network error'}`);
        throw err;
      }
    }
  }
}

// Mock API that fails twice then succeeds
let callCount = 0;
async function mockApiCall(attempt) {
  callCount++;
  console.log(`  [Mock API] request #${callCount} received`);
  if (attempt < 3) {
    const err = new Error('Rate limited');
    err.status = 429;
    throw err;
  }
  return { success: true, data: 'final result' };
}

console.log('Running retry with exponential backoff...\n');
retryWithBackoff(mockApiCall).then(result => {
  console.log('\nFinal result:', result);
});
```

Run it:
```
node backoff.js
```

Success criteria: You see two 429 retries with increasing delays before the final success.

### Part 3: Cursor-based pagination (5 minutes)

Create `cursor-paginate.js`. This fetches pages from JSONPlaceholder (a public test API) using cursor-based pagination with a `cursor` parameter (the last seen post ID) instead of page offsets.

```javascript
// cursor-paginate.js
// Demonstrates cursor-based pagination using a cursor parameter
// Unlike offset pagination (?page=X), cursor pagination (?cursor=last_id)
// does not skip records and is stable even when data changes mid-iteration

async function fetchAllPosts() {
  const BASE_URL = 'https://jsonplaceholder.typicode.com/posts';
  const PAGE_SIZE = 10;
  let allPosts = [];
  let cursor = null; // null = start from beginning

  while (true) {
    // Build URL with cursor parameter instead of page number
    let url = `${BASE_URL}?_limit=${PAGE_SIZE}`;
    if (cursor !== null) {
      url += `&_cursor=${cursor}`;
    }
    console.log(`Fetching with cursor=${cursor}: ${url}`);

    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const posts = await response.json();

    // Track last seen ID as our cursor
    const lastId = posts.length > 0 ? posts[posts.length - 1].id : null;

    if (posts.length === 0 || lastId === null) break;

    allPosts.push(...posts);
    console.log(`  Got ${posts.length} posts. Total so far: ${allPosts.length}`);

    if (posts.length < PAGE_SIZE) break; // last page

    // Advance cursor to last seen ID for next request
    cursor = lastId;
  }

  return allPosts;
}

console.log('Fetching all posts with cursor-based pagination...\n');
fetchAllPosts()
  .then(posts => {
    console.log(`\nTotal posts fetched: ${posts.length}`);
    console.log('First post:', posts[0]);
    console.log('Last post:', posts[posts.length - 1]);
  })
  .catch(err => console.error('Failed:', err.message));
```

Run it:

```
node cursor-paginate.js
```

Success criteria: Output shows cursor advancing through fetches (null -> 10 -> 20 -> ... -> 100) and a final total of 100 posts.

## Common mistakes

### 1. Storing API keys as string literals in source code

**What happens:** You hardcode `const API_KEY = 'sk_live_abc123'` at the top of your file. It gets committed to git, appears in error logs, GitHub search finds it, and now it is compromised. The API provider rotates the key. Your production system is down until you update all environments.

**Why it happens:** It is the path of least resistance. AI generates code this way because it is straightforward.

**How to fix:** Use environment variables. `process.env.API_KEY` in Node.js, `os.environ['API_KEY']` in Python. Store the actual value in `.env` files (never committed) or a secrets manager (AWS Secrets Manager, HashiCorp Vault). The `.env` file pattern:

```
API_KEY=sk_live_abc123
```

Then in code: `const apiKey = process.env.API_KEY;`

### 2. Retrying without distinguishing retriable from non-retriable errors

**What happens:** You retry every failed request, including 400 Bad Request (fix your code, do not retry), 401 Unauthorized (credentials are wrong, retrying will not help), and 404 Not Found (the resource does not exist). Your retry logic compounds the problem.

**Why it happens:** It is easier to retry everything than to distinguish error types.

**How to fix:** Only retry on 429 (rate limited) and 5xx (server error). Never retry on 4xx. For 429, respect the `Retry-After` header when present.

### 3. Ignoring rate limit headers until a 429 arrives

**What happens:** You call the API without monitoring `X-RateLimit-Remaining`. You make 99 calls successfully, then the 100th gets a 429. In a batch job, this means the last items in your batch fail.

**Why it happens:** Proactive rate limit monitoring is more code than fire-and-forget.

**How to fix:** Check `X-RateLimit-Remaining` before each request in high-volume scenarios. Slow down before hitting the limit rather than recovering from it. When 429 arrives, pause for the duration specified in `Retry-After`.

### 4. Not validating webhook signatures

**What happens:** You process a webhook request and take an action (mark an order as paid, provision a user account) before validating the signature. An attacker floods your webhook endpoint with fake events and your system acts on them.

**Why it happens:** Validation looks like extra boilerplate and it is tempting to "just process the body first to see what the data looks like."

**How to fix:** Validate the signature **before** any other processing. Use the webhook secret and the raw request body (not the parsed/rewritten body). Most frameworks have middleware to do this automatically.

### 5. Using offset pagination on high-volume or frequently-updated data

**What happens:** You request page 5 of a list. Between your first request and the next, 20 records are inserted at the top. Page 5 now has 10 records from page 4 and 10 records from page 6. You show the same records twice and skip others.

**Why it happens:** Offset pagination is intuitive. Cursor pagination requires understanding how the cursor encodes position.

**How to fix:** Use cursor pagination for any list that is large, frequently modified, or returned in real-time. Pass the `cursor` (or `after`, `since`, `pageToken` depending on the API) from the previous response's `next_cursor` field.

## AI-specific pitfalls

### AI calls rate-limited endpoints in tight loops

AI generates batch processing code that calls an API repeatedly without respecting rate limits. The code runs fine in testing (5 records) but fails in production (5,000 records) with a cascade of 429s.

What to look for when reviewing AI output: Any loop that calls an external API. Verify it has rate limit awareness: check `X-RateLimit-Remaining`, handle 429 with backoff, or use the SDK's built-in retry logic.

### AI stores API keys as string literals

AI generates code like:

```javascript
const apiKey = 'sk_live_abc123xyz';
```

or

```python
api_key = "sk_live_abc123xyz"
```

What to look for: Any string that looks like a key or token (`sk_live_`, `api_key`, `Bearer ` followed by a long string) in source code. The fix: `process.env.API_KEY` and a comment pointing to where to set it.

### AI does not handle webhook signature validation

AI generates a webhook handler that parses and processes the body without ever checking the signature. This is one of the most dangerous AI code generation failures because it creates a security vulnerability that looks like working code.

What to look for: Any webhook handler function. Verify it calls a signature validation function on the raw body before processing. The raw body is critical — many frameworks parse the body before your handler runs, which can cause signature mismatch if you do not preserve the original bytes.

### AI generates incorrect GraphQL queries

AI may generate a GraphQL query that references fields that do not exist in the schema, uses the wrong argument names, or assumes a field is nullable when it is required. The query fails at runtime with a schema validation error.

What to look for: AI-generated GraphQL queries against an unfamiliar schema. Always validate against the actual schema (available via introspection or the provider's docs). Look for queries that assume optional arguments are present or that a returned type has fields it does not have.

### AI invents REST endpoints that do not exist

AI confidently generates a call to `GET /api/user/{id}/profile` when the actual endpoint is `GET /api/users/{id}`. The code compiles, the request goes out, and returns a 404. AI may then tell you "the API is down" rather than "I called the wrong endpoint."

What to look for: Any AI-generated URL that is not directly copied from the provider's documentation. Cross-reference endpoint URLs against the official API reference before running the code.

## Quick reference

### When to use which OAuth2 flow

| Architecture | Flow | Why |
|---|---|---|
| Browser SPA (React, Vue) | Authorization Code + PKCE | No secret can be kept in browser; PKCE prevents code interception |
| Mobile app | Authorization Code + PKCE | Same reason as SPA |
| Server-to-server (M2M) | Client Credentials | No user involved; use client ID + secret |
| Traditional server-rendered app | Authorization Code | Server can hold secret; redirect-based |

### Rate limit response headers

```
429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1713000000
```

### Backoff formula

```
delay = min(maxDelay, baseDelay * 2^attempt + random_jitter)

# Example: baseDelay=500ms, maxDelay=30s
# Attempt 1: ~500-1500ms
# Attempt 2: ~1000-2000ms
# Attempt 3: ~2000-3000ms
# Attempt 4: ~4000-5000ms
```

### Pagination decision tree

```
Is the list large (>1000 items)?
  YES -> Use cursor pagination (avoid offset)
  NO  -> Offset pagination is acceptable

Does the data change frequently while iterating?
  YES -> Cursor pagination is required
  NO  -> Either works

Does the API support cursor pagination?
  YES -> Prefer it
  NO  -> Use offset with awareness of duplicates/skips
```

### SDK vs raw HTTP decision tree

```
Is there a well-maintained official SDK for your language?
  YES -> Use the SDK

Do you only need 1-2 endpoints from a large API?
  Maybe -> Raw HTTP may be lighter weight

Is the SDK unmaintained or missing features you need?
  YES -> Raw HTTP

Are you vibe coding with AI writing the HTTP calls directly?
  Maybe -> Raw HTTP is simpler to generate and review
```

### Webhook signature validation checklist

```
1. Receive raw request body (bytes, not parsed object)
2. Extract timestamp and signature from header
3. Compute: HMAC-SHA256(timestamp + "." + rawBody, secret)
4. Compare using timing-safe comparison
5. Reject if signature does not match
6. Process the body only after step 5 passes
```

## Go deeper

- [Stripe: Webhook signature verification](https://stripe.com/docs/webhooks/signatures) — The canonical reference for HMAC-based webhook validation. Every webhook integration should understand this pattern. (verified 2026-04)
- [OAuth 2.0 Security Best Current Practice (IETF)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) — The authoritative document on OAuth2 security pitfalls and which flows to use. (verified 2026-04)
- [Google Cloud: OAuth 2.0 for Mobile & Desktop Apps](https://developers.google.com/identity/protocols/oauth2#mobile-and-desktop) — Clear walkthrough of the PKCE flow for mobile and desktop apps, with code examples. (verified 2026-04)
- [Stripe API: Rate limiting](https://stripe.com/docs/rate-limits) — A real-world example of how a major API handles rate limits, with specific numbers and header formats. (verified 2026-04)
- [GitHub REST API: Pagination](https://docs.github.com/en/rest/guides/using-pagination-in-the-rest-api) — Practical explanation of cursor-based pagination from GitHub's API, with examples of the Link header and cursor tokens. (verified 2026-04)
