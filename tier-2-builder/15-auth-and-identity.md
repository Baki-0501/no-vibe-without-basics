# Authentication and Identity

## What is it?

Authentication answers: "Who are you?" Authorization answers: "What are you allowed to do?" These are two entirely different problems that AI code generators consistently confuse.

**Authentication** — Verifying identity. The system checks credentials (password, token, biometric) and concludes "this user is Alice."

**Authorization** — Checking permissions. The system asks "is Alice allowed to access this resource?" This happens after authentication is complete.

Think of it like airport security. Your boarding pass is your **authentication** — it proves who you are. The gate agent checking your ticket and letting you board is **authorization** — they're verifying you have permission to be on that specific flight. You need the boarding pass to get to the gate, but having a boarding pass doesn't mean you can board any plane.

### Password Hashing

Plaintext passwords cannot be stored. Ever. If your database is breached, every user's password is exposed. The solution is hashing — a one-way function that transforms a password into a fixed-length string. The same password always produces the same hash, but the hash cannot be reversed to reveal the original password.

**MD5 and SHA-1 are broken for password storage.** These are fast hash functions designed for checksums, not security. Attackers can compute billions of hashes per second with commodity hardware. Any AI-generated code that uses MD5 or SHA-1 for password hashing is a critical failure.

**bcrypt** — The older standard. Uses a configurable cost factor (work factor) that makes hashing slower. Purpose-built for password hashing. Still secure today, but argon2 is better for new projects.

**argon2** — Winner of the Password Hashing Competition in 2015. Memory-hard, meaning it resists GPU/ASIC attacks better than bcrypt. This is the recommended algorithm for new projects. Three variants: argon2i (resists side-channel attacks), argon2d (resists GPU attacks), argon2id (hybrid, recommended for most uses).

The fundamental principle: password hashing must be slow. The work factor exists so that even if an attacker gets your database, cracking passwords one by one takes years.

### Sessions vs. JWT

**Sessions** — The server stores user state. When you log in, the server creates a session record in a database or memory store, sends the client a session ID (a random string), and the client sends that ID with every request. The server looks up the session on each request to know who you are.

**JWT (JSON Web Tokens)** — A self-contained token. The server creates a signed payload that encodes user info, sends it to the client, and the client sends it back on every request. The server verifies the signature but doesn't need to store anything. The token is stateless.

```
JWT structure: header.payload.signature
header: { "alg": "HS256", "typ": "JWT" }
payload: { "sub": "user123", "role": "admin", "exp": 1234567890 }
signature: HMACSHA256(header.payload, secret)
```

Sessions are simpler to reason about and easy to invalidate (delete the session record). JWTs eliminate server-side storage and scale horizontally without a shared session store, but come with significant caveats.

### OAuth2 Flows

OAuth2 is a delegation framework. It lets users grant a third-party application access to their data without sharing their password. There are several flows for different use cases.

**Authorization Code + PKCE** — For browser-based apps (SPAs, mobile apps). The app generates a random string (code verifier), hashes it (code challenge), and redirects the user to the authorization server. The user logs in at the authorization server, grants permission, and the app receives an authorization code that it exchanges for tokens. This flow is designed so the authorization code cannot be intercepted or exchanged by an attacker.

**Client Credentials** — For machine-to-machine (M2M) communication. There's no user — the client application authenticates as itself using a client ID and client secret. It directly exchanges these credentials for an access token. Use this for background jobs, microservices calling other microservices, or any scenario where a user isn't involved.

**Avoid the Implicit Flow.** It was designed for SPAs before PKCE existed, but it exposes tokens in URLs and has been deprecated in OAuth 2.1. Any AI-generated code using Implicit Flow should be flagged.

### RBAC

Role-Based Access Control assigns permissions to roles, then assigns roles to users. Instead of saying "Alice can delete posts," you say "Alice has the editor role, and editors can delete posts."

```
Users → Roles → Permissions
User: { id: 1, email: "alice@example.com" }
Role: { name: "editor" }
Permission: { action: "delete", resource: "posts" }
RolePermission: { role_id: "editor", permission_id: "delete_posts" }
```

RBAC is simpler to audit and manage than permission checks scattered throughout code ("if user.isAdmin"). It's not perfect — if you need fine-grained, resource-level permissions, you may need ABAC (Attribute-Based Access Control) — but RBAC covers 80% of real applications.

### MFA

Multi-Factor Authentication requires two or more independent authentication factors:

- **Something you know** — Password, PIN
- **Something you have** — Phone, hardware key (YubiKey, Google Titan)
- **Something you are** — Fingerprint, Face ID

TOTP (Time-based One-Time Password) is the most common software MFA. A shared secret is provisioned on setup, and both the server and the user's authenticator app compute the same 6-digit code using the current time window. If the codes match, MFA is satisfied.

## Why it matters for vibe coding

Authentication code is security-critical, and AI code generators consistently produce broken, insecure, or incomplete auth implementations. Here's why this is the most dangerous area for vibe coding:

**AI confuses authentication and authorization constantly.** It will generate an `/admin` endpoint that checks if a user is logged in but forgets to check if they're an admin. This is called "missing authorization" — the most common web vulnerability class in the OWASP Top 10. You won't notice because the UI only shows the page to admins, but a curl command bypasses the UI entirely.

**AI generates localStorage code for JWTs.** When AI generates "JWT authentication," it almost always puts the token in localStorage. localStorage is accessible to any JavaScript on your page — including XSS (cross-site scripting) attacks. If an attacker injects a script tag, they read localStorage and steal the token. The correct approach is HttpOnly cookies with SameSite and Secure flags. AI won't tell you this unless you specifically ask about HttpOnly.

**AI uses === for password comparison.** A string comparison with `===` is not constant-time — an attacker can measure how long the comparison takes and use timing attacks to guess the correct password character by character. Constant-time comparison (like `crypto.timingSafeEqual` in Node.js) takes the same amount of time regardless of where the mismatch occurs. AI does not generate this by default.

**AI doesn't invalidate tokens on password change.** When a user changes their password, old tokens should be invalidated. AI generates "password update" functionality that saves the new hash but doesn't revoke existing sessions. If an attacker already has a session token, they remain authenticated after a password change.

**AI omits CSRF protection.** API endpoints that accept cookies are vulnerable to Cross-Site Request Forgery. AI generates POST endpoints that work but don't include CSRF tokens or SameSite cookies, leaving your app vulnerable to attacks where a malicious site tricks a logged-in user into making requests.

**AI suggests MD5/SHA for password hashing.** This is the most alarming pattern. Despite being widely known as broken, AI still generates MD5 and SHA-1 based password hashing code when asked about "password encryption" (AI also calls hashing "encryption" — another red flag). You must catch this.

## The 20% you need to know

### Password Hashing (the right way)

```javascript
// Node.js with argon2
import argon2 from 'argon2';

async function hashPassword(password) {
  return await argon2.hash(password, {
    type: argon2.argon2id,  // recommended variant
    memoryCost: 2 ** 16,    // 64MB
    timeCost: 3,            // 3 iterations
    parallelism: 1
  });
}

async function verifyPassword(hash, password) {
  return await argon2.verify(hash, password);  // constant-time by default
}
```

If you must use bcrypt:
```javascript
import bcrypt from 'bcrypt';
const hash = await bcrypt.hash(password, 12);  // cost factor 12, never below 10
const match = await bcrypt.compare(password, hash);  // constant-time
```

Always use a work factor. If it's too low, hashing is fast and attackable. If it's too high, your server spends seconds hashing every login. Benchmark on your target hardware and pick the highest cost factor that keeps login under 500ms.

### JWT: How to store and transmit safely

JWTs in browsers must use HttpOnly cookies. Never localStorage.

**What HttpOnly does:** The cookie cannot be read by JavaScript. Even if your site has an XSS vulnerability, the attacker cannot exfiltrate the token because `document.cookie` won't show it.

**What Secure does:** The cookie is only sent over HTTPS. Prevents tokens from being intercepted on the wire.

**What SameSite does:** Controls when the cookie is sent with cross-site requests. `SameSite=Strict` sends the cookie only to the site that set it. `SameSite=Lax` (the default in modern browsers) sends it for top-level navigation but not subresource requests. `SameSite=None` allows cross-site sending but requires Secure.

```
Set-Cookie: token=eyJhbGciOiJIUzI1NiJ9...; HttpOnly; Secure; SameSite=Lax
```

For server-to-server APIs (M2M, microservices), Authorization: Bearer tokens in headers are appropriate. The localStorage rule applies only to browser environments.

### JWT Structure and Validation

```javascript
import jwt from 'jsonwebtoken';

// Creating a token
const token = jwt.sign(
  { userId: 123, role: 'editor' },
  process.env.JWT_SECRET,
  { expiresIn: '1h', algorithm: 'HS256' }
);

// Verifying a token
try {
  const decoded = jwt.verify(token, process.env.JWT_SECRET);
  // decoded.userId, decoded.role are available
} catch (err) {
  // Token invalid or expired
}
```

**Critical:** Always verify the signature. Never decode a JWT without verifying — anyone can craft a JWT with `{ "role": "admin" }` in the payload. The signature is what makes it trustworthy.

### OAuth2 PKCE Flow (for SPAs)

```
1. Generate random code_verifier (32 bytes base64url encoded)
2. Compute code_challenge = BASE64URL(SHA256(code_verifier))
3. Redirect user to authorization server with code_challenge
4. User authenticates and grants permission
5. Authorization server redirects back with authorization_code
6. Exchange code + code_verifier for tokens at token endpoint
7. Server verifies code_verifier matches original code_challenge
```

The code_verifier never leaves the browser except as a hash in the initial redirect. Even if an attacker intercepts the redirect URL, they cannot exchange the code without the original code_verifier.

### RBAC Implementation Pattern

```javascript
// Define permissions as constants
const PERMISSIONS = {
  POSTS_READ: 'posts:read',
  POSTS_WRITE: 'posts:write',
  POSTS_DELETE: 'posts:delete',
  USERS_MANAGE: 'users:manage'
};

// Define role-permission mappings
const ROLE_PERMISSIONS = {
  viewer: [PERMISSIONS.POSTS_READ],
  editor: [PERMISSIONS.POSTS_READ, PERMISSIONS.POSTS_WRITE],
  admin: [PERMISSIONS.POSTS_READ, PERMISSIONS.POSTS_WRITE, PERMISSIONS.POSTS_DELETE, PERMISSIONS.USERS_MANAGE]
};

// Check permission middleware
function requirePermission(permission) {
  return (req, res, next) => {
    const userRole = req.user?.role;
    if (!userRole) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    const permissions = ROLE_PERMISSIONS[userRole] || [];
    if (!permissions.includes(permission)) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}

// Usage
app.delete('/posts/:id', requirePermission(PERMISSIONS.POSTS_DELETE), deletePost);
```

### MFA with TOTP

```javascript
import speakeasy from 'speakeasy';

// Server-side: generate provisioning URI and QR code URL
function generateMfaSecret(userEmail) {
  const secret = speakeasy.generateSecret({ name: `MyApp (${userEmail})` });
  return {
    base32: secret.base32,
    otpauthUrl: secret.otpauth_url  // Encode this in a QR code
  };
}

// Server-side: verify TOTP
function verifyMfaToken(base32Secret, token) {
  return speakeasy.totp.verify({
    secret: base32Secret,
    encoding: 'base32',
    token: token,
    window: 1  // Allow 1 step before/after for clock drift
  });
}
```

## Hands-on Exercise

**Build a complete auth flow with password hashing, session management, and role-based access control.**

You will create a Node.js/Express API with working authentication, secure password hashing, and RBAC. You will verify the output by making HTTP requests.

**Time: 15 minutes**

Prerequisites: Node.js 18+, npm

1. Set up the project:
```bash
mkdir auth-demo && cd auth-demo
npm init -y
npm install express argon2 cookie-parser jsonwebtoken
```

2. Create the server:
```javascript
// server.js
import express from 'express';
import argon2 from 'argon2';
import cookieParser from 'cookie-parser';
import jwt from 'jsonwebtoken';

const app = express();
app.use(express.json());
app.use(cookieParser());

const JWT_SECRET = process.env.JWT_SECRET || 'change-this-in-production';
const PERMISSIONS = { POSTS_READ: 'posts:read', POSTS_WRITE: 'posts:write', POSTS_DELETE: 'posts:delete' };
const ROLE_PERMISSIONS = { viewer: [PERMISSIONS.POSTS_READ], editor: [PERMISSIONS.POSTS_READ, PERMISSIONS.POSTS_WRITE], admin: Object.values(PERMISSIONS) };

// In-memory user store (use a real DB in production)
const users = [];

// Middleware to verify JWT from HttpOnly cookie
function authenticate(req, res, next) {
  const token = req.cookies?.token;
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    req.user = jwt.verify(token, JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Middleware to check permission
function requirePermission(permission) {
  return (req, res, next) => {
    const perms = ROLE_PERMISSIONS[req.user?.role] || [];
    if (!perms.includes(permission)) return res.status(403).json({ error: 'Forbidden' });
    next();
  };
}

// Register with argon2 hashing
app.post('/register', async (req, res) => {
  const { email, password, role = 'viewer' } = req.body;
  if (!email || !password) return res.status(400).json({ error: 'Email and password required' });
  if (users.find(u => u.email === email)) return res.status(409).json({ error: 'User exists' });

  const hash = await argon2.hash(password);
  users.push({ id: users.length + 1, email, password: hash, role });
  res.json({ message: 'User registered' });
});

// Login — sets HttpOnly cookie
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = users.find(u => u.email === email);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  const valid = await argon2.verify(user.password, password);
  if (!valid) return res.status(401).json({ error: 'Invalid credentials' });

  const token = jwt.sign({ userId: user.id, email: user.email, role: user.role }, JWT_SECRET, { expiresIn: '1h' });
  res.cookie('token', token, { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'lax' });
  res.json({ message: 'Logged in', role: user.role });
});

// Protected route — requires authentication
app.get('/me', authenticate, (req, res) => {
  res.json({ userId: req.user.userId, email: req.user.email, role: req.user.role });
});

// Protected route — requires specific permission
app.delete('/posts/:id', authenticate, requirePermission(PERMISSIONS.POSTS_DELETE), (req, res) => {
  res.json({ message: `Post ${req.params.id} deleted by ${req.user.role}` });
});

app.listen(3000, () => console.log('Auth demo running on http://localhost:3000'));
```

3. Run the server:
```bash
node server.js
```

4. Test the flow with curl:
```bash
# Register a new user
curl -X POST http://localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"secret123","role":"editor"}'

# Login (receives HttpOnly cookie)
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"email":"alice@example.com","password":"secret123"}'

# Check current user (sends cookie automatically)
curl -b cookies.txt http://localhost:3000/me

# Try to delete (editor role — should fail with 403)
curl -b cookies.txt -X DELETE http://localhost:3000/posts/1

# Register an admin and try again
curl -X POST http://localhost:3000/register \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"adminpass","role":"admin"}'

curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -c cookies_admin.txt \
  -d '{"email":"admin@example.com","password":"adminpass"}'

# Admin can delete (should succeed)
curl -b cookies_admin.txt -X DELETE http://localhost:3000/posts/1

# Logout (clear the cookie)
curl -b cookies.txt -X POST http://localhost:3000/login \
  --cookie-jar /dev/null -H "Content-Type: application/json"
```

5. Verify the output:
- Registration creates a hashed password (check `users` array — password is not "secret123")
- Login returns success with the user's role
- `GET /me` returns user data from the decoded JWT
- Editor making DELETE returns 403 Forbidden
- Admin making DELETE returns success
- Cookie has `httpOnly` flag (check `curl -v` output)

## Common mistakes

**Mistake 1: Storing passwords as plaintext or with reversible encryption.**

What happens: A database breach exposes every user's password. Attackers try those passwords on other sites (password reuse is rampant). Your company becomes responsible for credential stuffing attacks on other services.

Why it happens: Hashing adds complexity. MD5 seems simpler. "It's just a internal app" is the rationalization.

How to fix: Always use a purpose-built password hash function. argon2 is the default choice. bcrypt is acceptable if argon2 isn't available in your language. Never "encrypt" passwords — encryption implies decryption, and you don't need that.

**Mistake 2: Using localStorage for JWTs in browser applications.**

What happens: Any XSS vulnerability lets attackers steal tokens. With a stolen token, they have full access to the user's account for the token's lifetime.

Why it happens: localStorage is the most obvious place to store data in a browser. HttpOnly cookies require server-side setup, which feels like more work. AI often generates this pattern by default.

How to fix: Use HttpOnly cookies with `Secure` and `SameSite=Lax`. If you need CSRF protection, use the Double Submit Cookie pattern or a CSRF token. Do not use localStorage for tokens.

**Mistake 3: Checking authentication but not authorization.**

What happens: A logged-in user can access endpoints they shouldn't. They might not discover this through the UI, but a direct API call bypasses the UI entirely. This is how data breaches happen through authenticated API calls.

Why it happens: Authentication middleware is often added ("make sure user is logged in") but authorization checks ("make sure user has permission") are treated as optional or forgotten.

How to fix: Every protected endpoint must have both authentication and authorization checks. Use middleware that composes cleanly — `authenticate` then `requirePermission(SPECIFIC_PERMISSION)`. Audit endpoints for missing permission checks.

**Mistake 4: Using the Implicit OAuth flow or missing PKCE in SPA auth.**

What happens: Tokens leak through URL fragments, referrer headers, or browser history. Attackers intercept them and gain access.

Why it happens: The Implicit Flow was designed before PKCE existed and is easier to implement for SPAs. AI may generate this because it's in training data from older tutorials.

How to fix: Use Authorization Code + PKCE for all browser-based and mobile apps. There is no situation where Implicit Flow is the right choice in 2026. If you see Implicit Flow in AI output, reject it.

**Mistake 5: Not implementing token expiration or revocation.**

What happens: A token stolen today is valid forever. An employee who leaves still has a valid session. Compromised tokens remain usable until the arbitrary expiration time you set (if you set one at all).

Why it happens: Token expiration is easy to forget when the happy path (legitimate login) works fine. Revocation adds server-side state, which feels contrary to the stateless JWT philosophy.

How to fix: Set short token expiration (15 minutes to a few hours). Implement a token blocklist for revocation, or use refresh tokens with rotation. For sessions, store session state server-side and delete it on logout or password change.

## AI-specific pitfalls

**AI generates MD5 or SHA-1 for password hashing.** Despite being universally known as broken for this use case, AI code generators still produce MD5 and SHA-1 based password hashing when asked about "password encryption" or "password hashing." The fix is obvious but only if you catch it. Always verify the hashing algorithm in any AI-generated auth code.

**AI uses === for password comparison instead of constant-time comparison.** This is a timing attack vulnerability. An attacker can measure response times and deduce the correct password byte-by-byte. When reviewing AI output, look for any direct password comparison (`password === hash` or `password == storedPassword`) and replace with `crypto.timingSafeEqual` or the language's equivalent.

**AI stores JWTs in localStorage without mentioning HttpOnly cookies.** AI will implement working JWT auth that functions correctly but uses localStorage, leaving tokens vulnerable to XSS theft. It works — until it doesn't. When AI generates auth code, specifically ask about HttpOnly/Secure/SameSite flags and reject localStorage implementations for browser apps.

**AI generates auth code that checks authentication but not authorization.** AI will happily generate an `/admin/users` endpoint that verifies a token exists but doesn't check the role field. The UI might hide admin features from non-admins, but a single curl command accesses them directly. Every AI-generated endpoint that involves permissions must be reviewed for both checks.

**AI calls bcrypt with a cost factor that's too low or missing entirely.** When AI does use bcrypt (which is more responsible than MD5), it often uses a cost factor of 10 or lower, or omits the cost factor entirely (defaulting to a value from years ago when hardware was slower). Specify cost factor 12 or higher when prompting AI.

**AI generates OAuth Implicit Flow or omits PKCE.** If you ask AI to "add OAuth login to a React app," it will often generate Implicit Flow because it's simpler to implement. You must catch this. Always specify Authorization Code + PKCE flow when prompting AI about OAuth for SPAs.

**AI doesn't handle token revocation on password change.** Password change flows often look like simple UPDATE operations. AI won't think to invalidate existing sessions. Specify this requirement explicitly: "when a user changes their password, invalidate all existing sessions."

## Quick Reference

### Password Hashing Comparison

| Algorithm | Type | Recommendation |
|---|---|---|
| MD5 | Fast hash | Never use for passwords |
| SHA-1 | Fast hash | Never use for passwords |
| SHA-256 | Fast hash | Not recommended — too fast |
| bcrypt | Slow hash | Acceptable if argon2 unavailable |
| argon2id | Memory-hard slow hash | Recommended for new projects |

### JWT Storage Decision Tree

```
Is the client a browser?
├─ YES → HttpOnly cookie with Secure + SameSite=Lax
└─ NO (server-to-server, M2M) → Authorization: Bearer header
```

### OAuth Flow Selection

```
What type of client?
├─ SPA (browser) or Mobile app → Authorization Code + PKCE
├─ Machine-to-machine (no user) → Client Credentials
└─ Server-side app with backend → Authorization Code (with client secret)
```

### Cookie Security Flags

| Flag | Purpose |
|---|---|
| HttpOnly | Prevents JavaScript access (XSS protection) |
| Secure | Only sent over HTTPS |
| SameSite=Strict | Never sent on cross-site requests |
| SameSite=Lax | Sent on top-level navigation, not subresources (recommended default) |
| SameSite=None | Sent on all cross-site requests (requires Secure) |

### RBAC Permission Check Pattern

```javascript
// Always use middleware layers — never inline both checks
app.delete('/resource/:id',
  authenticate,           // 1. Verify user is logged in
  requirePermission('resource:delete'),  // 2. Verify user has this permission
  handler
);
```

### Constant-Time Comparison

| Language | Method |
|---|---|
| Node.js | `crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))` |
| Python | `secrets.compare_digest(a, b)` |
| Go | `subtle.ConstantTimeCompare([]byte(a), []byte(b))` |
| Java | `MessageDigest.isEqual()` |

## Go deeper

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) (verified 2026-04) — Detailed guidance on work factors, algorithms, and implementation.
- [JWT.io](https://jwt.io/) (verified 2026-04) — Interactive JWT debugger. Paste a token and decode it, verify signatures, and understand the structure.
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) (verified 2026-04) — IETF draft covering PKCE requirements, security best practices, and deprecated flows.
- [Supabase: Auth Deep Dive](https://supabase.com/docs/guides/auth) (verified 2026-04) — Practical implementation guide for auth patterns in modern applications.
- [WebAuthn.io](https://webauthn.io/) (verified 2026-04) — FIDO2/WebAuthn demonstration. The emerging standard for passwordless authentication that goes beyond TOTP.
