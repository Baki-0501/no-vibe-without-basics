# Security Hardening

## What is it?

Security hardening is the practice of reducing a system's attack surface — closing the gaps that attackers exploit. It covers input validation, output encoding, dependency hygiene, transport security, and access controls. The goal is to make your application survive contact with hostile input without leaking data or granting unauthorized access.

Think of it like building a house. You don't just lock the front door — you check the windows, the garage, whether the deadbolt actually latches, and whether the locksmith you hired left a copy of the key under the doormat. Hardening is that systematic check of every entry point.

The security landscape is vast. The OWASP Top 10 gives you a prioritised focus list — the ten highest-impact vulnerability classes that account for the majority of real-world breaches. If you understand and defend against these, you're in better shape than most.

## Why it matters for vibe coding

AI code generators produce working code fast, but they optimise for functionality, not security. AI will generate a database query using string interpolation because it produces correct results in the happy path. AI will output user input directly into HTML because it renders correctly in the browser. AI will suggest disabling CORS as a quick fix to a confusing error.

Without security knowledge, you can't review what AI generates. You ship the vulnerable code and find out about it when your user data is leaked, your database is wiped, or your app becomes a spam relay.

**Without this knowledge, the AI will SQL-inject your database and you won't know why it happened.** You'll see an AI-generated `db.query("SELECT * FROM users WHERE id = " + userId)` and think it looks fine because the variable is named descriptively.

**Without this knowledge, AI will echo raw user input into HTML and you won't catch the XSS.** You'll see a search results page that renders user comments verbatim and think that's correct behaviour. You'll be wrong.

**Without this knowledge, you won't know that AI-generated `curl` commands with credentials in URLs get logged by proxies, servers, and browser history.** Every `https://api-key:secret@host.com` is a leaked credential waiting to happen.

Security is the one area where you cannot rely on "it works" as a quality signal. You need to know what secure looks like so you can recognise when AI produces the opposite.

## The 20% you need to know

### OWASP Top 10 (2021) — The Big Picture

The OWASP Top 10 is a regularly-updated list of the most critical web application security risks, published by the Open Web Application Security Project. The 2021 list (still current as of writing) in order:

1. **Broken Access Control** — Users acting outside their intended permissions. Reading other users' data, modifying other users' records, accessing admin functions.
2. **Cryptographic Failures** — Sensitive data exposure due to weak or missing encryption. This used to be called "Sensitive Data Exposure" and is the root cause of most data breach headlines.
3. **Injection** — SQL, NoSQL, OS command, and LDAP injection when untrusted data is interpreted as a command. SQL injection is the canonical example.
4. **Insecure Design** — Architectural flaws. Not just implementation bugs, but missing threat modeling, missing rate limiting, missing access controls at the design level.
5. **Security Misconfiguration** — Default credentials left in place, unnecessary features enabled, verbose error messages exposing stack traces.
6. **Vulnerable and Outdated Components** — Using libraries with known vulnerabilities. This is now the most common root cause of exploitable vulnerabilities.
7. **Identification and Authentication Failures** — Broken authentication, session fixation, weak password policies.
8. **Software and Data Integrity Failures** — Trusting software updates, CI/CD pipelines, or data from untrusted sources without verification.
9. **Security Logging and Monitoring Failures** — Attacks are missed because there's no logging, or breaches aren't detected because no one is watching.
10. **Server-Side Request Forgery (SSRF)** — Fetching internal resources (cloud metadata, internal services) via user-supplied URLs.

Items 1, 3, and 6 cover the majority of incidents you will encounter. Start there.

### SQL Injection — Parameterized Queries

SQL injection occurs when user input is included in a SQL query string and interpreted as SQL code. An attacker submits `' OR '1'='1` as a login field and bypasses authentication entirely.

The fix is simple: never interpolate user input into SQL strings. Use parameterized queries (also called prepared statements), which separate SQL structure from data.

```python
# BAD — user input interpolated directly into query
user_id = request.args.get("id")
query = f"SELECT * FROM users WHERE id = {user_id}"
db.execute(query)

# GOOD — parameterized query, input treated as data
user_id = request.args.get("id")
query = "SELECT * FROM users WHERE id = %s"
db.execute(query, (user_id,))
```

```javascript
// Node.js — BAD
const query = `SELECT * FROM users WHERE id = ${userId}`;
db.query(query);

// GOOD — parameterized
const query = "SELECT * FROM users WHERE id = ?";
db.query(query, [userId]);
```

ORMs (SQLAlchemy, Prisma, Hibernate) generally produce parameterized queries automatically — provided you don't bypass them with raw SQL. Most AI-generated code uses ORM patterns correctly, but AI will sometimes drop into raw SQL for "performance" or "simplicity" reasons. Flag that as a security review requirement.

### Cross-Site Scripting (XSS) — Output Encoding

XSS occurs when an application includes user input in HTML output without sanitizing it. The browser interprets that input as HTML or JavaScript, executing attacker-controlled code in the context of your app.

Three types: **Stored** (saved to database, served to all users), **Reflected** (in URL parameters, executed immediately), and **DOM-based** (client-side only, in client-side JavaScript).

The primary defence is output encoding — transforming user input so it can't be interpreted as HTML or script.

```html
<!-- User submits: <script>stealCookies()</script> as their name -->

<!-- BAD — raw output, script executes -->
<p>Welcome, {{ user.name }}</p>

<!-- GOOD — HTML-escaped, safe to render -->
<p>Welcome, {{ user.name | escape }}</p>
<!-- Or: -->
<p>Welcome, <%= h(user.name) %></p>
```

Modern frameworks (React, Angular, Vue) escape output by default in their templates. The danger zone is:
- Setting `innerHTML` directly with user input
- Generating HTML via string concatenation in JavaScript
- Using `dangerouslySetInnerHTML` in React (explicitly opt-out of escaping)
- Rendering Markdown that supports embedded HTML

Context matters for encoding. HTML encoding won't protect you inside `<script>` tags, inside JSON attributes, or in URL parameters. Use the right encoding for the context: HTML escape for body text, JavaScript escape for inline scripts, URL escape for parameter values, CSS escape for style content.

### Cross-Site Request Forgery (CSRF) — Tokens

CSRF tricks a logged-in user's browser into making an authenticated request to your app without their knowledge. The browser automatically sends cookies, so the malicious site doesn't need to steal credentials — it just needs to trigger the request.

The standard defence is a CSRF token — a secret, per-session, per-request value that the server verifies on state-changing requests (POST, PUT, DELETE). The malicious page can't read the token (same-origin policy blocks cross-origin reads), so it can't include it in the forged request.

```html
<!-- Form includes a CSRF token -->
<form method="POST" action="/transfer">
  <input type="hidden" name="csrf_token" value="{{ csrf_token }}" />
  <input name="to_account" value="..." />
  <input name="amount" value="..." />
  <button type="submit">Transfer</button>
</form>
```

```python
# Server-side verification
@app.route("/transfer", methods=["POST"])
def transfer():
    token = request.form.get("csrf_token")
    if token != session.get("csrf_token"):
        abort(403)
    # proceed
```

Most frameworks have built-in CSRF protection. Verify it's enabled, not disabled. AI sometimes suggests disabling CSRF protection to "fix a confusing error" during development — never do that in production.

### Input Validation — Defence in Depth

Input validation rejects obviously malicious data before it reaches your business logic. It is not a replacement for output encoding or parameterized queries — it is an additional layer.

**Principle:** Validate on the server side (never trust client-side validation), validate against a whitelist of allowed values, and fail safely (reject, don't sanitize).

```python
# BAD — trusting client-side validation
@app.route("/set-age")
def set_age():
    age = request.args.get("age")  # From JavaScript that validates, or doesn't
    user.age = int(age)  # Crashes on non-numeric input
    # Also: negative ages, ages of 5000

# GOOD — server-side validation with a whitelist
@app.route("/set-age")
def set_age():
    age = request.args.get("age", type=int)
    if age is None or age < 0 or age > 150:
        abort(400)
    user.age = age
```

Length limits, type checks, range checks, and format正则 expressions (regex) for structured inputs (email, phone, date) are all forms of input validation. Use them at every entry point: API parameters, form fields, headers, URL path segments.

### Dependency Auditing

Your application has thousands of transitive dependencies. One vulnerable library can compromise your entire stack. Dependency auditing is the practice of scanning your dependency tree against vulnerability databases (CVE, GitHub Advisory Database).

```bash
# npm
npm audit
npm audit fix

# pip
pip audit
# or with Safety
safety check

# Snyk (free tier available)
snyk test

# Docker (scan image layers)
docker scout cves myapp:latest
```

`npm audit` and `pip audit` both check your lockfile against the national vulnerability databases. Run them in CI — if a critical vulnerability is found, the build should fail. Don't just run them locally and ignore the output.

### Rate Limiting

Rate limiting prevents abuse by restricting how many requests a client can make in a given time window. Without it, your API can be exhausted by a single client, or your endpoints can be brute-forced at full speed.

```python
# Flask with flask-limiter
from flask_limiter import Limiter
limiter = Limiter(app, default_limits=["200 per day", "50 per hour"])

@app.route("/login", methods=["POST"])
@limiter.limit("5 per minute")  # Stricter limit on auth endpoint
def login():
    # ...
```

Return `429 Too Many Requests` with a `Retry-After` header. Log rate limit violations so you can distinguish an attack from a legitimate traffic spike.

### Content Security Policy (CSP)

CSP is an HTTP header that tells the browser where resources can be loaded from. It is a defence-in-depth measure against XSS — even if an attacker injects script, CSP can block it from executing if it doesn't match an allowed source.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-randomtoken123'; style-src 'self' 'unsafe-inline'; img-src 'self' https:;
```

A strict CSP:
- `default-src 'self'` — Only load from your own origin by default
- `script-src 'self'` — Only execute scripts from your own origin
- `object-src 'none'` — Disable plugins (Flash, Java) entirely

`unsafe-inline` and `unsafe-eval` weaken CSP significantly. AI sometimes generates CSP headers with these flags to make things "just work." Push back — a weak CSP is only slightly better than no CSP.

### HTTPS Enforcement

HTTPS encrypts traffic between your server and the browser, preventing eavesdropping and man-in-the-middle attacks. It also validates your server's identity via certificates.

Enforce HTTPS at the server level:
- Redirect all HTTP to HTTPS
- Set `Strict-Transport-Security` (HSTS) header to tell browsers to only connect over HTTPS
- Set `Secure` flag on cookies so they're never sent over HTTP

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

```python
# Flask: enforce HTTPS
@app.before_request
def force_https():
    if request.is_secure:
        return
    # Redirect to HTTPS
    url = request.url.replace("http://", "https://")
    return redirect(url, code=301)
```

### Supply Chain Risks

Your code is only as secure as your dependencies and the tools that build it. Supply chain attacks target the pipeline: a malicious package uploaded to npm/pip, a compromised CI environment, a typosquatted package name (`requests` vs `requets`), or a maintainer account compromise.

**Defences:**
- Use lockfiles (`package-lock.json`, `Pipfile.lock`) and verify them in CI — don't auto-update without review
- Pin dependency versions: `"lodash": "4.17.21"` not `"lodash": "^4.17.0"`
- Use a private registry mirror if you have one — you can audit what goes in
- Enable dependency review features in your CI (GitHub Dependabot, Snyk)
- Check package publisher and download counts before adding a new dependency
- Use `--ignore-scripts` for packages you don't trust, to prevent post-install code execution

## Hands-on exercise

**Audit a Node.js project for vulnerabilities and fix what you find.**

Time: 15 minutes

Prerequisites: Node.js installed, a code editor

1. Set up a deliberately vulnerable project:
```bash
mkdir security-audit-demo && cd security-audit-demo
npm init -y
npm install express body-parser lodash@4.17.11
```

Note that `lodash@4.17.11` has a known prototype pollution vulnerability (CVE-2019-10744). You'll find and fix it.

2. Run the audit:
```bash
npm audit
```

You should see output like:
```
lodash  <=4.17.11
Severity: high
Prototype Pollution - https://npmjs.com/advisories/1673
```

3. Fix it by upgrading:
```bash
npm install lodash@4.17.21
npm audit
```

The vulnerabilities should be gone.

4. Now add a SQL injection vulnerability to a test file to see it in action. Create `test-sql.js`:
```javascript
// VULNERABLE — do not use in production
const { Pool } = require("pg");
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getUser(userId) {
  const query = `SELECT * FROM users WHERE id = ${userId}`;
  const result = await pool.query(query);
  return result.rows[0];
}

getUser("1 OR 1=1").then(console.log).catch(console.error);
```

5. Fix it using parameterized queries:
```javascript
async function getUser(userId) {
  const query = "SELECT * FROM users WHERE id = $1";
  const result = await pool.query(query, [userId]);
  return result.rows[0];
}
```

6. Run both versions — the parameterized version will correctly reject the injected input as a string value for `$1`, while the vulnerable version will return all users.

**Expected output:**
- `npm audit` initially shows a high-severity vulnerability in lodash
- After upgrade, `npm audit` shows no vulnerabilities
- Parameterized query treats `"1 OR 1=1"` as a string literal, not SQL
- The vulnerable version returns all users; the fixed version returns nothing (or an error, depending on your DB schema)

## Common mistakes

**Mistake 1: Using string interpolation for SQL queries because "it works for the happy path."**

What happens: An attacker finds a field that accepts user input (a search box, a URL parameter) and crafts input that breaks out of the intended query context. They dump your entire database or authenticate as an admin.

Why it happens: The vulnerable code is shorter and reads more naturally in English. Parameterized queries feel like ceremony. The bug has no visible symptoms during normal testing.

How to fix: Treat parameterized queries as non-negotiable, like syntax. If AI generates a SQL query with string interpolation, reject it. Every time.

**Mistake 2: Treating client-side validation as security instead of UX.**

What happens: You add a max-length attribute to a form field or a JavaScript regex check and call it input validation. An attacker bypasses all of it with `curl` and sends any payload they want.

Why it happens: Client-side validation is convenient and provides immediate feedback. It looks like it works. But the server receives raw HTTP requests that can be crafted independently of any form.

How to fix: Always validate on the server. Client-side validation is a UX improvement, not a security control. Server-side validation is mandatory.

**Mistake 3: Disabling security features to make things "just work."**

What happens: You disable CSRF protection, set `Sec-Fetch-Site: cross-site` CORS policies to `*`, or add `unsafe-inline` to your CSP because something broke and you don't know why. The security feature was doing its job — blocking cross-origin abuse. You disabled it for a development inconvenience and shipped it.

Why it happens: Security misconfiguration often has no immediate visible symptom. The app works. The feature you disabled was invisible. Until someone exploits it.

How to fix: Use environment-based configuration. Security features should be strict in production, relaxed in local dev — but the production config should be correct. Use feature flags or environment variables, not code changes.

**Mistake 4: Not auditing dependencies and assuming "it worked yesterday."**

What happens: A new CVE drops for a library you depend on. You're not monitoring for it, so you don't know. An attacker finds your vulnerable instance via Shodan or similar and exploits it.

Why it happens: Your app has dozens of indirect dependencies. You don't own them, you don't track them, and new vulnerabilities are discovered constantly. The assumption that "no news is good news" is wrong.

How to fix: Run `npm audit` or `pip audit` in CI on every build. Subscribe to security advisories for your key dependencies (GitHub does this automatically for public repos). Use Dependabot or Snyk to auto-open PRs when vulnerabilities are found.

**Mistake 5: Trusting user input in dangerous contexts without considering context.**

What happens: You HTML-encode user input and render it safely in a paragraph, so you think XSS is handled. But the same value is also placed inside a `<script>` tag, a JavaScript event handler (`onclick`), or a URL parameter — each of which requires different encoding.

Why it happens: Data flows through multiple contexts in a single page. You secure one and forget the others. It's not enough to encode at the point of output — you need to consider the context of each output.

How to fix: Map your data flows. When user input is rendered anywhere on a page, audit every location: HTML body, `<head>`, `<script>` blocks, attributes, styles, URLs. Use a framework that handles context-aware encoding automatically, and audit any use of `innerHTML`, `dangerouslySetInnerHTML`, or raw HTML generation.

## AI-specific pitfalls

**AI generates SQL queries using string concatenation or f-strings.** This is the single most common security failure in AI-generated code. When you see `db.query(f"SELECT ... FROM ... WHERE id = {id}")` or equivalent in any language, that is an automatic rejection and rewrite. Parameterized queries are not optional.

**AI outputs HTML with raw user input embedded.** AI will generate templates like `<p>Hello, {{ username }}</p>` where `username` is unsanitized user input. In many frameworks (Django, Rails, Laravel), auto-escaping is on by default. In plain JavaScript, React's JSX, or when using template engines with custom filters, escaping is not automatic. Verify that user input is escaped at render time.

**AI suggests disabling CORS as a fix for cross-origin errors.** A common AI-generated "fix" for CORS errors in development is to set `Access-Control-Allow-Origin: *` or to disable origin checks entirely. This is never the right production answer. CORS is a browser security mechanism. If your frontend and backend are on different origins in production, configure the specific origins that need access. If AI suggests `*` for a production API that handles sensitive data, that's a critical failure.

**AI does not understand the scope of XSS in HTML contexts.** AI will generate HTML that embeds user input in JavaScript contexts: `<script>const name = "{{ user.name }}"</script>`. This is not protected by HTML encoding — the browser parses the HTML first, then the JavaScript. You need JavaScript encoding (escaping `"` as `\x22`, `\` as `\\`) for data placed inside `<script>` tags or event handler attributes. AI will not spontaneously produce this — you need to explicitly prompt for safe embedding in JavaScript contexts.

**AI uses older library versions that have known vulnerabilities.** When AI generates a `package.json` or `requirements.txt`, it often picks library versions that are current in its training data but subsequently patched. Always run `npm audit` or `pip audit` against AI-generated dependency lists before using them.

**AI generates `curl` commands with credentials embedded in URLs.** `curl https://username:password@api.example.com/endpoint` causes that URL to be logged by proxies, web servers, browser history, and referrer headers. AI produces this pattern because it works and is compact. Credentials should always be passed via `-H "Authorization: Bearer $TOKEN"` headers or similar, never in the URL.

**AI doesn't understand SSRF and will generate code that fetches user-supplied URLs.** If an AI generates a feature that accepts a URL from a user and fetches it (a link preview, a webhook validator, an image proxy), that is a potential SSRF vector. The server can be tricked into fetching internal infrastructure: cloud metadata endpoints (`169.254.169.254`), internal databases, or local file URLs (`file:///etc/passwd`). Verify that any URL-fetching code has an allowlist of permitted destinations, not just a blocklist.

## Quick reference

### SQL Injection Quick Check

| Code pattern | Status |
|---|---|
| `f"SELECT ... WHERE id = {id}"` | Vulnerable — reject and rewrite |
| `db.query("SELECT ... WHERE id = ?", [id])` | Safe — parameterized |
| `SELECT * FROM users WHERE id = %s` with tuple binding | Safe — parameterized |
| `text("SELECT * FROM users WHERE id = :id")` with dict binding | Safe — parameterized |
| Raw SQL via ORM `.raw()` or `.execute()` | Suspicious — verify no interpolation |

### XSS Quick Context Guide

| Output context | Encoding needed |
|---|---|
| HTML body text | HTML escape (`<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`, `'` → `&#x27;`, `&` → `&amp;`) |
| Inside `<script>` tags | JavaScript escape (or use a quoted data container) |
| HTML attribute values | HTML escape + attribute quoting |
| CSS content | CSS escape |
| URL parameter values | URL percent encoding |

### CSP Quick Header Builder

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';    # only if needed for inline styles
  img-src 'self' https:;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
```

Remove `unsafe-inline` if possible. It allows inline scripts and defeats much of CSP's XSS protection.

### Dependency Audit Commands

| Tool | Command | Critical action |
|---|---|---|
| npm | `npm audit` | CI should fail on high/critical |
| pip | `pip audit` | CI should fail on high/critical |
| Snyk | `snyk test` | Free tier: `snyk monitor` for ongoing scanning |
| GitHub | Dependabot alerts | Enable on all repos |
| Docker | `docker scout cves <image>` | Scan image layers in CI |

### Security Headers Reference

| Header | Purpose |
|---|---|
| `Strict-Transport-Security` | Enforce HTTPS for this domain |
| `Content-Security-Policy` | Restrict resource loading sources |
| `X-Content-Type-Options: nosniff` | Prevent MIME type sniffing |
| `X-Frame-Options: DENY` | Prevent clickjacking via iframe embedding |
| `X-XSS-Protection: 0` | Disable old XSS auditor (not needed with CSP) |
| `Referrer-Policy: strict-origin-when-cross-origin` | Control referrer header leakage |
| `Permissions-Policy` | Disable unused browser features (camera, mic, geolocation) |

## Go deeper

- [OWASP Top 10 2021](https://owasp.org/Top10/) (verified 2026-04) — The authoritative list with detailed descriptions and example attack scenarios for each vulnerability class.
- [PortSwigger Web Security Academy — SQL Injection](https://portswigger.net/web-security/sql-injection) (verified 2026-04) — Interactive labs covering SQL injection in multiple contexts, with detailed explanations. The free version is excellent.
- [PortSwigger Web Security Academy — Cross-Site Scripting](https://portswigger.net/web-security/cross-site-scripting) (verified 2026-04) — Comprehensive XSS reference covering reflected, stored, and DOM-based XSS with labs.
- [npm audit documentation](https://docs.npmjs.com/cli/v8/commands/npm-audit) (verified 2026-04) — Official docs for running audits in CI/CD pipelines.
- [MDN — Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) (verified 2026-04) — Practical CSP guide with examples of common policies and directives.
