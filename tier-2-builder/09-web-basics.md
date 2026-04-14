# Web Basics

## What is it?

The web runs on a simple client-server conversation: a browser (client) sends a request to a server, the server processes it and sends back a response, and the browser renders the result. Every web page you have ever visited works this way — no exceptions.

Understanding the web means understanding four interconnected layers:

**HTML** is a markup language that structures content. A browser reads HTML and builds a **Document Object Model (DOM)** — a tree of nodes where each tag (`<div>`, `<p>`, `<button>`) is a node. This tree is what JavaScript manipulates when it "changes the page."

**CSS** controls how that tree looks. It attaches visual rules (colors, layout, spacing) to nodes in the DOM. The browser renders CSS and HTML together into what you see on screen.

**JavaScript** is the programming language that runs in your browser. It can read and modify the DOM, make network requests, handle user events (clicks, typing), and run arbitrary logic. When AI generates "interactive" web code, it is writing JavaScript.

**HTTP** (Hypertext Transfer Protocol) is the wire protocol that carries requests from browser to server and responses back. It is the language both ends of the web conversation speak.

### A mental model worth internalizing

Think of the DOM as a living, in-memory representation of the page. The HTML is the source text. The DOM is the parsed object tree. CSS rules attach to nodes in that tree. JavaScript reads and mutates the tree. HTTP is the postal service delivering messages between your browser and servers.

When you interact with a web page, here is what actually happens:

1. Browser requests an HTML file from a server.
2. Server sends HTML. Browser parses it into a DOM tree.
3. Browser sees `<link>` tags for CSS and `<script>` tags for JavaScript and requests those files too.
4. CSS is applied to nodes in the DOM. JavaScript executes and may modify the DOM.
5. The result is rendered pixels on screen.

## Why it matters for vibe coding

When AI generates web code, it is generating HTML, CSS, and JavaScript — and that code interacts with HTTP APIs. Without a mental model for how these pieces fit together, you cannot:

- Debug why a button click does nothing (JavaScript event not wired to DOM node?)
- Understand why an AI-generated chart renders blank (data fetch failed silently?)
- Know why an AI's "smart" API integration breaks in production (CORS? Wrong status code handling? Auth token in the wrong place?)

Specific failure modes you will encounter:

- **AI generates a `fetch()` call that never checks the response status.** The code treats a 404 or 500 the same as a 200. Data is missing. You do not know why.
- **AI generates API calls that assume the server is on the same origin as the page.** CORS errors appear. AI tells you to "just add CORS headers" on the server — but the real fix may involve how the request is made from the client.
- **AI puts authentication tokens in the URL query string.** Tokens end up in server logs, browser history, and browser extensions. Anyone with access to those logs can hijack the session.
- **AI generates code that calls an API endpoint that does not exist or has different parameters than the AI assumed.** You get a vague error with no context.
- **AI assumes JSON responses are always valid.** It does not handle the case where the server returns HTML (an error page) or an empty response body.

## The 20% you need to know

### HTTP: the request/response lifecycle

Every HTTP exchange has the same shape:

```
Client -> [METHOD] [URL] [HEADERS] [BODY?]
Server -> [STATUS CODE] [HEADERS] [BODY?]
```

A **request** has a method (GET, POST, PUT, DELETE, PATCH), a URL, headers (key-value metadata), and optionally a body (for POST/PUT).

A **response** has a numeric status code, headers, and a body (the data or HTML you came for).

HTTP is stateless — each request is independent. There is no automatic memory of previous requests. Cookies and headers are how state survives across requests.

### REST: conventions, not rules

REST is a set of conventions for structuring APIs. The core idea: treat data as resources identified by URLs, and use HTTP methods to describe what you are doing to those resources.

```
GET    /users        <- fetch all users
GET    /users/123    <- fetch user 123
POST   /users        <- create a new user
PUT    /users/123    <- replace user 123
PATCH  /users/123    <- partially update user 123
DELETE /users/123    <- delete user 123
```

REST is a convention. APIs that do not follow these conventions exactly still work. But when AI generates API calls, it often assumes REST conventions that the real API does not follow. Knowing this lets you catch the mismatch.

### HTTP status codes: the families

You need to know what each family means. Everything else is detail.

| Family | Meaning | What to do |
|--------|---------|------------|
| **2xx** | Success | Process the response body |
| **3xx** | Redirect | Follow it (or handle manually) |
| **4xx** | Client error | Fix the request — the server is saying you asked for something wrong |
| **5xx** | Server error | The server failed, not you — retry later |

Common codes you will hit constantly:

- `200 OK` — success, response body contains what you asked for
- `201 Created` — POST succeeded, new resource created
- `204 No Content` — success, but nothing to return (common for DELETE)
- `400 Bad Request` — your request was malformed; check the format
- `401 Unauthorized` — you are not authenticated (no valid credentials)
- `403 Forbidden` — you are authenticated but not allowed to do this
- `404 Not Found` — the resource does not exist at this URL
- `429 Too Many Requests` — rate limited; back off and retry
- `500 Internal Server Error` — server-side failure; not your fault
- `502 Bad Gateway` / `503 Service Unavailable` — server is overloaded or misconfigured

**AI almost always generates code that only handles the 2xx case.** Every `fetch()` call needs a check for non-2xx responses.

### Request and response headers

Headers carry metadata. A few matter constantly:

**Request headers** (sent by the client):
- `Authorization: Bearer <token>` — how you prove who you are
- `Content-Type: application/json` — tells the server what format your body is in
- `Accept: application/json` — tells the server what format you want back
- `Cookie: <cookies>` — sends cookies back to the server

**Response headers** (sent by the server):
- `Content-Type: application/json` — the format of the response body
- `Set-Cookie: <cookie>` — instructs the browser to set a cookie
- `Access-Control-Allow-Origin: *` — the CORS header that permits browser cross-origin requests
- `Cache-Control: max-age=3600` — tells browsers how long to cache this response

The `Authorization` header is where auth tokens belong — **not in the URL**. URLs are logged everywhere: server logs, proxy logs, browser history. A token in a URL is a compromised token.

### CORS: what it is and why it exists

The **Same-Origin Policy** is a browser security rule: a web page from `https://myapp.com` cannot read data from `https://api.otherdomain.com` unless that server explicitly allows it.

**CORS (Cross-Origin Resource Sharing)** is the mechanism servers use to opt-in to being accessed by cross-origin web pages. It works via headers:

```
Access-Control-Allow-Origin: https://myapp.com
```

When a browser sees this header in a response, it allows JavaScript on `https://myapp.com` to read that response. Without it, the browser blocks the response and throws a CORS error.

**Preflight requests:** For "non-simple" requests (POST with JSON body, requests with custom headers), the browser first sends an `OPTIONS` request to ask "is this allowed?" before sending the actual request. The server must respond to that OPTIONS with the right CORS headers. This is called a preflight request.

Common AI mistake: AI generates a `fetch()` call that sends a POST with a JSON body, then tells you to "just add CORS headers on the server." This ignores that:
1. The server must handle the OPTIONS preflight as well as the actual POST.
2. The client must send the right headers (`Content-Type: application/json`) to trigger preflight in the first place.

### Cookies: what they are and how they work

Cookies are small pieces of data that a server asks the browser to store and send back with every request to that domain. They are the primary mechanism for session management — keeping you logged in across page reloads.

How a cookie is set:

1. Server sends `Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax`
2. Browser stores this cookie.
3. On every subsequent request to that domain, the browser sends `Cookie: session_id=abc123`

Cookie flags matter:

- **HttpOnly** — JavaScript cannot read this cookie (prevents XSS theft). Always set this for session tokens.
- **Secure** — Cookie only sent over HTTPS (prevents network interception).
- **SameSite** — Controls when the cookie is sent in cross-site requests. `SameSite=Lax` is the modern default. `SameSite=Strict` prevents all cross-site sending. `SameSite=None` allows cross-site but requires `Secure`.

AI commonly generates code that sets cookies without these flags, leaving sessions vulnerable to XSS and network interception.

### JSON: the lingua franca of web APIs

JSON (JavaScript Object Notation) is a text format for structured data. It maps directly to JavaScript objects, which is why it became the default for web APIs.

JSON rules:
- Strings are double-quoted: `"name"` not `'name'`
- Keys are double-quoted: `{"name": "Alice"}`
- No trailing commas: `[1, 2, 3]` not `[1, 2, 3,]`
- No comments

When a response has `Content-Type: application/json`, the body is JSON. Parse it with `JSON.parse(responseBody)` in JavaScript (or let `fetch().then(r => r.json())` do it automatically).

AI frequently generates code that assumes a response is JSON when it is not — or assumes JSON will always parse cleanly.

## Hands-on exercise

**Goal:** Inspect a live HTTP exchange in your browser, observe CORS in action, and make a fetch request that handles errors properly.

**Time:** 10-15 minutes

### Part 1: Watch HTTP in action (5 minutes)

1. Open your browser (Chrome/Edge/Firefox).
2. Press `F12` to open DevTools.
3. Click the **Network** tab.
4. Go to any website (e.g., `https://example.com`).
5. Click on any request in the Network panel.
6. Look at the **Headers** tab. You will see: Request URL, Request Method, Status Code, and Response Headers.
7. Click the **Response** or **Preview** tab to see the actual body.

You just inspected a real HTTP exchange. The headers you see are exactly what the module describes.

### Part 2: Make a fetch request with error handling (10 minutes)

Create a file called `http-practice.html` and open it in your browser:

```html
<!DOCTYPE html>
<html>
<head>
  <title>HTTP Practice</title>
</head>
<body>
  <h1>HTTP Practice</h1>
  <button id="btn-success">Request Success (200)</button>
  <button id="btn-notfound">Request Not Found (404)</button>
  <button id="btn-cors">Request to Different Origin</button>
  <pre id="output">Click a button to see results.</pre>

  <script>
    const output = document.getElementById('output');

    function log(label, data) {
      output.textContent = `${label}\n\n${JSON.stringify(data, null, 2)}`;
    }

    // Use httpbin.org — a public API that echoes back request details
    async function makeRequest(url, options = {}) {
      try {
        const response = await fetch(url, options);
        const data = await response.json(); // httpbin returns JSON
        log(`OK: ${response.status} from ${url}`, data);
      } catch (err) {
        // Network failure (CORS, offline, DNS failure)
        log(`NETWORK ERROR: ${err.message}`, {});
      }
    }

    // Better version: check response.ok
    async function makeRequestWithStatusCheck(url, options = {}) {
      const response = await fetch(url, options);
      const data = await response.json().catch(() => ({})); // handle non-JSON responses

      if (!response.ok) {
        log(`ERROR: HTTP ${response.status} from ${url}\nThis would silently fail in AI-generated code.`, {
          status: response.status,
          statusText: response.statusText,
          headers: Object.fromEntries(response.headers.entries()),
          body: data
        });
        return;
      }

      log(`SUCCESS (${response.status}): Request headers sent to ${url}`, {
        status: response.status,
        contentType: response.headers.get('Content-Type'),
        body: data
      });
    }

    document.getElementById('btn-success').addEventListener('click', () => {
      // GET request to httpbin — returns 200 with echo of our (non-existent) headers
      makeRequestWithStatusCheck('https://httpbin.org/get');
    });

    document.getElementById('btn-notfound').addEventListener('click', () => {
      // httpbin returns 404 for /status/404
      makeRequestWithStatusCheck('https://httpbin.org/status/404');
    });

    document.getElementById('btn-cors').addEventListener('click', () => {
      // POST with JSON body — triggers a CORS preflight
      makeRequestWithStatusCheck('https://httpbin.org/post', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: 'Alice', role: 'engineer' })
      });
    });
  </script>
</body>
</html>
```

**What to observe:**
1. Open DevTools **Console** alongside the page.
2. Click **Request Success** — notice the 200 and the response body.
3. Click **Request Not Found** — notice the error handling fires and shows the 404 status. Without `if (!response.ok)`, the AI-generated version would silently continue as if nothing went wrong.
4. Click **Request to Different Origin** — open the **Network** tab. Before the POST request, you will see an `OPTIONS` request (the preflight). The POST request follows. This is CORS in action.

**Success criteria:** You can see the Network tab showing both the preflight `OPTIONS` and the `POST` request. You see the error output when clicking 404.

## Common mistakes

### 1. Treating all responses as success

**What happens:** You make a `fetch()` call, use `.then(r => r.json())` on the result, and your code silently proceeds as if the data is valid. A 404 or 500 response means there is no valid JSON body — your `.json()` call throws an exception or returns empty data. Downstream code fails with no clear error message.

**Why it happens:** It is natural to write code for the happy path. HTTP errors are the default in production.

**How to fix:** Always check `response.ok` before treating a response as valid data. Wrap `.json()` in a try/catch to handle non-JSON responses (error pages, redirects).

```javascript
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}
const data = await response.json();
```

### 2. Sending auth tokens in URLs

**What happens:** You use `fetch('https://api.example.com/data?token=abc123')`. The token appears in server access logs, proxy logs, browser history, and browser extension data. Anyone with access to those logs has a valid session token.

**Why it happens:** It is easier to put the token in the URL than to configure headers. Some AI-generated code does this because it is simpler to write.

**How to fix:** Always send tokens in the `Authorization` header:

```javascript
fetch('https://api.example.com/data', {
  headers: { 'Authorization': 'Bearer abc123' }
});
```

### 3. Not understanding CORS as a browser-enforced rule

**What happens:** You see a CORS error in the browser console. You add `Access-Control-Allow-Origin` to your server and it still does not work. Or you get a CORS error when calling an API from a script you wrote — but `curl` to the same URL works fine.

**Why it happens:** CORS is a browser security mechanism. It only applies when JavaScript in a browser makes a cross-origin request. `curl` has no such restriction. The CORS headers must be on the server responding to the cross-origin request — but the browser also needs the request to include the right headers (`Content-Type`, `Authorization`) to trigger preflight.

**How to fix:** Check the Network tab for the `OPTIONS` preflight. If it is missing or returns the wrong headers, the server is not configured correctly. If it is present but the actual request still fails, the `Access-Control-Allow-Origin` header on the response does not match your origin.

### 4. Sending cookies with every cross-origin request

**What happens:** You make a cross-origin `fetch()` call and include credentials (cookies), but the server does not return the right CORS headers and the browser blocks the response. Or you unintentionally send session cookies to a third-party API, leaking session data.

**Why it happens:** `fetch()` with `credentials: 'include'` sends cookies on every request, including cross-origin ones, if the server allows it.

**How to fix:** Use `credentials: 'same-origin'` by default (only sends cookies to the same origin). Only use `'include'` when you explicitly need cross-origin cookie sending and have verified the server sets `Access-Control-Allow-Credentials: true`.

### 5. Not handling non-JSON responses

**What happens:** Your code calls `.json()` on a response that is actually an HTML error page or an empty response (e.g., a 204 No Content). The `.json()` call throws a syntax error that is hard to debug because the error message does not indicate what the actual response body was.

**Why it happens:** Not all API responses are JSON. Error pages are often HTML. Some endpoints return empty bodies.

**How to fix:** Always validate `Content-Type` before parsing, and wrap `.json()` in try/catch:

```javascript
const contentType = response.headers.get('Content-Type');
if (!contentType || !contentType.includes('application/json')) {
  throw new Error(`Expected JSON but got ${contentType}`);
}
const data = await response.json();
```

## AI-specific pitfalls

### AI generates `fetch()` without checking `response.ok`

This is the single most common AI web code mistake. The generated code will look like:

```javascript
const data = await fetch(url).then(r => r.json());
```

This silently ignores all HTTP errors. If the server returns 404, 401, or 500, the code proceeds anyway and `r.json()` either throws or returns unexpected data.

What to look for when reviewing AI output: Search for every `fetch` call and verify it has a `response.ok` check or equivalent error handling. If it does not, add it.

### AI does not understand the CORS preflight requirement

AI will generate code like:

```javascript
fetch('https://api.example.com/data', {
  method: 'POST',
  body: JSON.stringify({ name: 'Alice' })
});
```

With `Content-Type: application/json`, this triggers a CORS preflight. AI often does not anticipate this or understand why the OPTIONS request fails if the server is not configured for it.

What to look for: Any `fetch()` with `Content-Type: application/json`, `Authorization` header, or non-standard header. Verify the server handles OPTIONS requests at that endpoint.

### AI puts auth tokens in URLs or localStorage

AI sometimes generates:

```javascript
// Wrong — token in URL
fetch(`https://api.example.com/data?token=${token}`)

// Also wrong — token in localStorage (vulnerable to XSS)
localStorage.setItem('token', token);
fetch('https://api.example.com/data')
```

What to look for: Any URL with a token or key embedded in it (`?api_key=`, `?token=`). Any code that reads from `localStorage` for auth. The correct pattern is `Authorization: Bearer <token>` header with the token stored in an `HttpOnly` cookie set by the server.

### AI assumes API responses always match the happy-path schema

AI generates TypeScript types or destructuring that assumes the response always has the expected fields. In reality, a 404 error response has a completely different shape than a 200 success response. AI-generated code often lacks a guard that handles the error response shape differently.

What to look for: Code that destructures a response without first checking the status code. Add a status code check before assuming the response has the "success" shape.

## Quick reference

### HTTP methods

| Method | Semantics | Has Body? |
|--------|-----------|-----------|
| GET | Fetch a resource | No |
| POST | Create a new resource | Yes |
| PUT | Replace a resource entirely | Yes |
| PATCH | Partially update a resource | Yes |
| DELETE | Remove a resource | No (usually) |

### Status code families

```
2xx  Success        <- response body is what you asked for
3xx  Redirect       <- follow it or handle it explicitly
4xx  Client error   <- fix your request
5xx  Server error   <- retry later, not your fault
```

### CORS checklist

```
Preflight triggered by (any of):
  - POST with Content-Type: application/json
  - POST with custom headers
  - PUT, PATCH, DELETE
  - Authorization header on GET

Server must respond to OPTIONS with:
  - Access-Control-Allow-Origin: <your-origin> (or * if no credentials)
  - Access-Control-Allow-Methods: <methods allowed>
  - Access-Control-Allow-Headers: <headers allowed>

Client must send:
  - Content-Type: application/json (if posting JSON)
  - Authorization: Bearer <token> (if auth needed)
  - credentials: 'include' only when cross-origin cookies are needed
```

### Cookie flags

```
HttpOnly   <- JS cannot read this cookie (XSS protection)
Secure     <- only sent over HTTPS
SameSite=Lax  <- modern default; sent on same-site + top-level navigations
SameSite=Strict <- only sent on same-site
SameSite=None  <- sent on all cross-site requests; requires Secure
```

## Go deeper

- [MDN: How HTTP works](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) — The definitive browser-friendly explainer. Read this before anything else. (verified 2026-04)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) — Full CORS reference including preflight, headers, and credentials. (verified 2026-04)
- [httpbin.org](https://httpbin.org/) — A public HTTP testing service. Use it to inspect headers, test auth, trigger specific status codes, and observe CORS in real time. (verified 2026-04)
- [REST API Tutorial — HTTP Status Codes](https://restfulapi.net/http-status-codes/) — A clean reference for status code families with advice on when each applies. (verified 2026-04)
- [Google Web Dev: SameSite cookies explained](https://web.dev/articles/samesite-cookies-explained) — The clearest explanation of SameSite flags and why they matter for modern web security. (verified 2026-04)
