# Deployment

## What is it?

Deployment is the process of getting your application from your development machine to a place where real users can access it. That "place" is usually a server — a remote computer running 24/7 — with a domain name pointing to it, a web server in front of it, and TLS encrypting every connection.

The full stack looks like this: your code lives on GitHub. A CI pipeline (GitHub Actions) detects new code, runs tests, and packages it. A deployment tool ships that package to a server. A reverse proxy (nginx or Caddy) receives incoming traffic and routes it to your running application. DNS points a domain name to the server's IP. TLS encrypts the connection so no one can intercept data in transit.

Each layer has its own job. The application layer runs your code. The proxy layer handles SSL, routing, and often caching. The DNS layer maps names to IPs. The CI layer automates everything that used to happen manually — "SSH in, pull, restart."

## Why it matters for vibe coding

Vibe coding creates a specific and frequent failure mode: **AI builds your app, you have no idea how to ship it, and you either ship it wrong or not at all.**

Here's what actually happens:

- You ask AI to "deploy this to production." It generates a Docker configuration or a shell script. You run it. It works on port 3000 on your machine. On the server, port 3000 is firewalled off, nothing is listening on 80/443, and there's no process manager keeping it alive after you close your terminal.
- You ask AI to "set up HTTPS." It suggests buying a certificate and editing config files. You do that. Three months later the certificate expires and your site goes dark at 2am because no one was watching.
- You ask AI to "deploy an update." It overwrites the running application without draining connections first. Active users get errors mid-request.
- You ask AI to "set up CI." It generates a workflow that runs `npm install` and `npm start`. No tests. No build step. No deployment step. It just... sits there.

The core problem: deployment requires decisions AI can't make for you. It needs to know your server IP, your domain registrar, your TLS situation, your process manager, your budget. AI generates scaffolding. You need to know enough to know what the scaffolding is missing.

## The 20% you need to know

### CI/CD: turning code into running software automatically

CI (Continuous Integration) means: every time you push code, automated tools check that it works. Run tests. Build artifacts. CD (Continuous Deployment) means: if those checks pass, the code automatically goes to production without a human pressing buttons.

GitHub Actions is the most common CI tool for GitHub-hosted projects. A workflow is defined in `.github/workflows/*.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm run build
      - run: npm run deploy  # your deploy script
```

This is the skeleton. The `test` job runs your tests. The `deploy` job only runs if tests pass. The `needs: test` dependency enforces this ordering.

The critical thing AI often misses: **CI pipelines need to actually test something and then deploy something.** Many AI-generated pipelines have a `test` step that just echoes "tests passed" or skips testing entirely. Always verify your pipeline runs real tests.

### Reverse proxies: nginx and Caddy

Your application listens on a port (say, 3000). The internet sends traffic to ports 80 (HTTP) and 443 (HTTPS). A reverse proxy sits in front of your application, receives the traffic, and forwards it.

**nginx** — the industry standard. Fast, configurable, battle-tested. Handles SSL termination, load balancing, static file serving, and routing. The config looks like:

```
server {
    listen 80;
    server_name myapp.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Caddy** — simpler. TLS certificates are automatic and built-in. The entire config for a reverse proxy with HTTPS is:

```
myapp.com {
    reverse_proxy localhost:3000
}
```

Caddy provisions and renews TLS certificates via Let's Encrypt automatically. If you want zero-config HTTPS, Caddy is the choice. nginx gives you more control but requires you to manage certificates separately (usually with Certbot).

### DNS: A records and CNAMEs

DNS maps domain names to IP addresses. Two record types you need to know:

**A record** — maps a domain (or subdomain) directly to an IPv4 address:
```
A    myapp.com    203.0.113.42
```

**CNAME record** — maps one domain to another domain (aliases):
```
CNAME    www.myapp.com    myapp.com
```

The CNAME says "look up this other name instead." You cannot use a CNAME at the root (`@`) record on most DNS providers — only at subdomains. For the root domain, use an A record pointing to a static IP, or use an ALIAS record (a provider-specific extension that acts like a CNAME for the root).

Typical setup: an A record for `myapp.com` pointing to your server IP, a CNAME for `www.myapp.com` pointing to `myapp.com`.

### HTTPS/TLS: what it actually is

TLS (Transport Layer Security) encrypts data in transit between your users and your server. Without it, anyone on the same WiFi or same ISP can intercept passwords, tokens, and data.

TLS requires a certificate — a file that proves your server owns the domain. Certificates are signed by Certificate Authorities (CAs). Let's Encrypt is the free CA that most people use. Certificates expire (typically 90 days for Let's Encrypt) and must be renewed.

The failure mode AI doesn't handle well: AI might generate a self-signed certificate for production ("it works locally"). Browsers will show a security warning to your users. Self-signed certificates are only appropriate for local development or internal networks.

Certificate renewal is the part that bites you: if you manage TLS manually and forget to renew, your site goes down at the worst time. Use Certbot (nginx) or Caddy (automatic) to handle renewal automatically.

### Zero-downtime deploys: blue-green and canary

**Blue-green deployment** — you have two identical production environments: "blue" (current) and "green" (new). Traffic goes to blue. You deploy to green, test it, then switch traffic to green. If something breaks, switch back to blue.

```
Blue (live):  v1  --> traffic
Green (staging): v2 --> test it, then flip traffic
```

On a single server without load balancing, this is harder. You can simulate it by deploying to green, testing on a different port, then switching the proxy. With Docker Compose, you can run both versions simultaneously on different ports and update the proxy config to point to the new one.

**Canary deployment** — you route a small percentage of traffic to the new version (e.g., 5%) before rolling out fully. This lets you catch problems with real traffic before affecting everyone.

Neither strategy is built-in to most AI-generated deploy scripts. AI will typically generate a naive deploy that stops the old version and starts the new one — causing a brief window of downtime. You need to know this is wrong and build in proper zero-downtime deploy logic.

### Health check endpoints

A health check is a URL your application exposes that returns 200 OK when the app is healthy. Load balancers, orchestrators (Kubernetes, Docker Compose), and CI pipelines use it to know if your app is ready to receive traffic.

In Express.js:
```javascript
app.get('/health', (req, res) => {
    res.json({ status: 'ok', uptime: process.uptime() });
});
```

In FastAPI (Python):
```python
@app.get("/health")
def health():
    return {"status": "ok", "uptime": time.time()}
```

The health endpoint should be lightweight — no database calls, no external API calls. It checks whether the process is alive and the basic routing works.

AI-generated applications often don't include health endpoints. Without one, you can't use Docker healthchecks, you can't configure load balancer probes, and you can't build meaningful rollback logic.

### Rollback strategies

When a deployment goes wrong, you need to be able to go back. The strategy depends on your infrastructure:

**Simple (no Kubernetes):**
- Keep the previous version's code or Docker image tagged (e.g., `myapp:v1`, `myapp:v2`)
- If v2 breaks, redeploy v1
- In a shell script: before deploying, tag the current image as `myapp:previous`

**Git-based rollback:**
- `git revert HEAD` creates a new commit that undoes the last bad commit
- Push the revert, CI pipeline deploys it automatically
- Simple but slow (CI runs again)

**Docker image rollback:**
```bash
# Tag the last good image
docker tag myapp:good myapp:latest
# Or pull a specific previous tag
docker pull myapp:2024-01-15
```

The important part: **your rollback plan must be defined before the deployment, not after**. After something goes wrong is the worst time to figure out how to undo it.

## Hands-on exercise

**Goal:** Set up a GitHub Actions CI pipeline that builds a Node.js app, runs a test, and simulates a deployment. Takes 15 minutes.

### Prerequisites
- A GitHub account
- A repository with a Node.js app (or create one for this exercise)

### Step 1: Create a minimal Node.js app with a test

```bash
mkdir vibe-deploy-practice && cd vibe-deploy-practice
npm init -y
npm install express
npm install --save-dev jest
```

Create `app.js`:
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello from vibe deploy!');
});

app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
});

module.exports = app;
```

Create `server.js`:
```javascript
const app = require('./app');
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Running on port ${PORT}`));
```

Create a test in `app.test.js`:
```javascript
const request = require('supertest');
const app = require('./app');

test('GET / returns message', async () => {
    const res = await request(app).get('/');
    expect(res.text).toContain('Hello from vibe deploy!');
});

test('GET /health returns ok', async () => {
    const res = await request(app).get('/health');
    expect(res.body.status).toBe('ok');
});
```

Update `package.json` scripts:
```json
"scripts": {
    "start": "node server.js",
    "test": "jest"
}
```

### Step 2: Push to GitHub

```bash
git init
git add .
git commit -m "Initial app with tests"
# Create a repo on GitHub, then:
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

### Step 3: Add a GitHub Actions workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build  # add a build step to catch build errors

  # Simulate deployment (replace with real deploy in production)
  deploy-check:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - run: echo "Deploy step would run here"
```

Push this file and watch the pipeline run at `https://github.com/YOUR_USERNAME/YOUR_REPO/actions`.

### Step 4: Verify

- The workflow should run on push
- The `test` job should run Jest and pass
- The `deploy-check` job should only run after `test` passes and only on `main` branch

### What to observe

The `needs: test` dependency is what prevents bad code from being deployed. If you break a test and push, the pipeline fails before deploy-check even starts. This is the CI pattern: gates before deploy.

## Common mistakes

### 1. Deploying without a health check

**What happens:** You deploy a broken version. Traffic routes to it before it can respond. Users get 502 errors. Load balancers keep retrying and timing out. The deploy looks successful (the process started) but the app is broken.

**Why it happens:** Health checks feel optional. They're not.

**How to fix:** Add a `/health` endpoint to your application. Configure your reverse proxy and container orchestrator to check it before routing traffic. For nginx: `proxy_set_header Host $host; proxy_pass http://localhost:3000;` needs a corresponding health check directive. For Docker Compose, add `healthcheck:` to your service definition.

### 2. Not knowing which branch deploys to production

**What happens:** A feature branch gets pushed. CI deploys it somewhere unexpected. Or main gets pushed but no one has set up the production deploy step, so it never actually goes live.

**Why it happens:** CI configuration is generated without clear mapping of branches to environments.

**How to fix:** Explicitly define in your workflow file: `if: github.ref == 'refs/heads/main'` gates the production deploy. Use naming conventions like `deploy-staging.yml` and `deploy-production.yml` to make the mapping obvious. Document which branch maps to which environment.

### 3. No rollback plan

**What happens:** A deployment goes wrong. You have no way to quickly revert. Minutes of downtime turn into an hour while you diagnose what broke and try to manually fix it.

**Why it happens:** Rollback is boring and feels like unnecessary planning. It's not.

**How to fix:** Before every significant deploy: what's the previous known-good state? How do you get back to it? Keep tagged Docker images (`myapp:stable`, `myapp:v1.2.3`). Use a deploy script that can take a version argument: `./deploy.sh v1.2.3` and `./rollback.sh`.

### 4. Forgetting that DNS changes are slow

**What happens:** You change your DNS records to point to a new server. It works for you immediately. Your users are still hitting the old server for the next 24-48 hours because their ISPs cache the old DNS response.

**Why it happens:** DNS TTL (Time To Live) determines how long resolvers cache your records. If you set a TTL of 86400 seconds (1 day), users will use the old IP for up to a day.

**How to fix:** Before a migration, lower your DNS TTL to 300 (5 minutes) hours in advance. This makes the switch faster. Raise it back after the migration stabilizes. When you see advice like "DNS changes can take up to 48 hours," it means you have high TTLs — you can avoid this by planning ahead.

### 5. HTTPS misconfiguration (mixed content, expired certs)

**What happens:** Your site has HTTPS but loads resources (images, scripts, stylesheets) over HTTP. Browsers block the insecure resources and show a mixed content warning. Or your TLS cert expires and browsers block the site.

**Why it happens:** You migrated to HTTPS but some hardcoded URLs in your app still use `http://`. Or you set up Certbot but forgot to automate renewal.

**How to fix:** Search your codebase for `http://` URLs — they should all be protocol-relative (`//`) or use HTTPS. For nginx, use Certbot's auto-renewal: `sudo certbot --nginx` followed by `sudo certbot renew --dry-run`. Or just use Caddy which renews automatically.

## AI-specific pitfalls

### AI generates CI pipelines without test steps

This is the most common CI failure mode from AI. The pipeline `npm install`s and `npm start`s but never runs tests. It always "succeeds" because there's nothing to fail.

**What to look for:** No `npm test` in the workflow file. Or a test step that just runs `echo "tests passed"`. A valid test step must actually execute your test framework (Jest, pytest, Go test, etc.) and fail the build if tests fail.

**How to fix:** Verify your pipeline has a real test invocation. Check the exit code of the test step. Set `if: always()` only if you intentionally want to continue on test failure (usually wrong for production pipelines).

### AI doesn't understand why HTTPS requires a certificate

AI sometimes generates configurations that reference TLS certificates that don't exist, or suggests disabling SSL verification for production ("to make it work"). It may also generate self-signed certificates and imply they're suitable for public sites.

**What to look for:** Any config that sets `ssl: false` or disables certificate verification in a production context. Any config that references certificate paths without showing how to provision the cert.

**How to fix:** For any production deployment, use Let's Encrypt (via Certbot for nginx, or Caddy's automatic HTTPS). If AI suggests a self-signed cert, reject it unless it's strictly internal (local dev, internal network).

### AI generates configs that cause downtime on deploy

The naive deploy script: stop old version, start new version. Between stop and start, nothing is listening. Users get connection refused.

**What to look for:** Any deploy script that runs `docker-compose down` or `pkill` without a corresponding start of the new version before the old one stops. Any deploy script that updates code in-place without a restart mechanism that maintains availability.

**How to fix:** Use a blue-green pattern: start the new version on a new port, update the proxy to point to the new port, then stop the old version. Or use a process manager (systemd, PM2) that can restart with zero-downtime via `restart` rather than `stop` then `start`.

### AI generates nginx configs that don't handle upstream failures

AI-generated nginx configs often proxy to an upstream without configuring what happens when the upstream is down. Default nginx behavior returns a 502, but the upstream might be starting up (the "thundering herd" problem where all requests hit a starting upstream simultaneously).

**What to look for:** No `proxy_next_upstream` directive, no health check configuration, no retry logic.

**How to fix:** Add at minimum:
```
proxy_connect_timeout 5s;
proxy_next_upstream error timeout http_502;
```

This tells nginx to try the next upstream (if you have multiple) or return a clean error rather than a blank 502.

### AI assumes all ports are reachable

In many cloud environments, servers have firewalls that only allow specific ports (80, 443, and maybe SSH on 22). AI-generated configurations may reference ports like 3000, 5000, 8080 as if they're publicly accessible.

**What to look for:** Any configuration that exposes a non-standard port to the internet without firewall instructions.

**How to fix:** Verify which ports your cloud provider's security group allows. If only 80/443 are open, your application must bind to those ports directly or behind a reverse proxy on the allowed ports.

## Quick reference

### GitHub Actions workflow decision tree

```
New code pushed to:
  ├── feature branch
  │     └── Run: test (on pull_request)
  │
  └── main branch
        └── Run: test
              └── If test passes: deploy
```

### Reverse proxy comparison

| Feature | nginx | Caddy |
|---------|-------|-------|
| SSL certificates | Manual (Certbot) | Automatic |
| Config complexity | High | Low |
| Performance | Very high | High |
| Learning curve | Steeper | Gentle |
| Best for | Complex routing, custom setups | Quick HTTPS, simple proxies |

### DNS record types

| Record | Use case | Example |
|--------|----------|---------|
| A | Root domain to IPv4 | `myapp.com` -> `203.0.113.42` |
| CNAME | Subdomain alias | `www.myapp.com` -> `myapp.com` |
| AAAA | Root domain to IPv6 | `myapp.com` -> `2001:db8::1` |

### Blue-green deploy checklist

```
Before deploy:
[ ] Previous version tagged/preserved
[ ] New version deployed on parallel port
[ ] Health check returns 200 on new version
[ ] Run smoke tests against new version
[ ] Flip proxy to new version
[ ] Monitor error rates for 5 minutes
[ ] If broken: flip proxy back, investigate
```

### TLS certificate providers

| Provider | Cost | Renewal | Setup complexity |
|----------|------|---------|-------------------|
| Let's Encrypt (Certbot) | Free | Automatic | Medium |
| Caddy built-in | Free | Automatic | Low |
| Cloudflare | Free tier available | Automatic | Low |
| Commercial (DigiCert, etc.) | Paid | Manual or automatic | High |

## Go deeper

- [GitHub Actions Documentation](https://docs.github.com/en/actions) — Official docs for GitHub's CI/CD platform. Start with "Understanding GitHub Actions" and the workflow syntax reference. (verified 2026-01)
- [nginx Reverse Proxy Documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) — The definitive reference for nginx proxy configuration, including all proxy_* directives. (verified 2026-01)
- [Caddy Automatic HTTPS Documentation](https://caddyserver.com/docs/automatic-https) — Explains how Caddy handles TLS automatically, including certificate provisioning and renewal. (verified 2026-01)
- [Let's Encrypt — Certbot Instructions](https://certbot.eff.org/) — Select your web server and OS to get step-by-step instructions for setting up automatic TLS with Let's Encrypt. (verified 2026-01)
- [Kubernetes Canary Deployments](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments) — If you're ready to move beyond single-server deploys, this explains canary patterns in the context of Kubernetes (the orchestrator most production environments use). (verified 2026-01)
