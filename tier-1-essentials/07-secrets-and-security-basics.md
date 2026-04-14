# Secrets and Security Basics

## What is it?

A **secret** is any credential, token, key, or piece of information that grants access to a system you shouldn't have by default. API keys, database passwords, encryption keys, OAuth tokens, private keys, and AWS access credentials all count. So does any string that, if exposed, would let an attacker impersonate you or access data they shouldn't.

The threat model is straightforward: anything in your codebase that an attacker could use is a target. Secrets are not theoretical risks — automated scanners run by attackers constantly sweep public GitHub repositories looking for exposed API keys. 발견되면 즉시 악용됩니다.

The solution isn't to just be careful. It's to never put secrets in code, and to have tooling that verifies you haven't.

## Why it matters for vibe coding

Vibe coding creates a specific, high-frequency failure mode: **AI puts secrets in code and you don't notice until it's a disaster.**

Here's what actually happens:

- You ask an AI to "write a function that calls the OpenAI API." It writes `api_key = "sk-1234567890abcdef..."` right in the source file. You copy-paste it. It's committed. It's in your git history forever.
- You ask an AI to "set up authentication." It generates a `.env.example` file that contains placeholders — but also sometimes contains real values the AI hallucinated as "example" keys.
- You ask an AI to "help me debug why my API calls are failing." It suggests printing the full request headers to stdout for inspection. Your API key is now in a log file that gets committed, or worse, exposed in a error tracking service.
- You say "don't commit secrets" and the AI doesn't understand that the database password in `settings.py` is a secret. It commits it.

The pattern: AI has no concept of "this string should never appear in code." It sees patterns. It sees `api_key = ` and generates a value. You have to architect against this, not rely on the AI to understand.

## The 20% you need to know

### What counts as a secret

Be paranoid. Anything that grants access counts:

- API keys and tokens (OpenAI, AWS, Stripe, GitHub, etc.)
- Database connection strings and passwords
- Encryption keys and salts
- Private keys (SSH, TLS)
- Session secrets and JWT secrets
- Any placeholder string that looks like a real credential (`sk-...`, `AKIA...`, `ghp_...`)

Service account emails and public identifiers are usually fine. The test: if it starts with `sk-`, `AKIA`, `ghp_`, `xoxb-`, or any similar prefix, treat it as a secret. If it looks like a UUID or hash that was generated for authentication, treat it as a secret.

### Environment variables are the answer — but not the only answer

The correct pattern: store secrets in environment variables, never in source code.

```python
# Wrong — secret is in source code
api_key = "sk-1234567890abcdef"

# Right — secret comes from the environment
api_key = os.environ.get("OPENAI_API_KEY")
```

The environment variable is injected at runtime, never compiled into the binary, never appears in `git diff`, and is easy to rotate without changing code.

`.env` files are the common local development pattern: a file that contains `OPENAI_API_KEY=sk-...` and is loaded by your application. The `.env` file itself is a secret and must never be committed.

### .gitignore is your first line of defense

Your `.gitignore` must include every file that could contain secrets:

```
.env
.env.local
.env.*.local
*.pem
*.key
config/secrets.yml
secrets.json
credentials.json
```

**Critical nuance:** Adding a file to `.gitignore after it is already tracked does not remove it from history. Git history is immutable. If you committed `.env` before adding it to `.gitignore`, the secret is still in history. Anyone who cloned before you added `.gitignore` has it.

The fix for already-committed secrets in history is to treat it as a security incident: rotate the key immediately, then use tools like `git filter-branch` or BFG Repo-Cleaner to rewrite history and force-push.

### Secret scanning: your automated safety net

Two tools handle this well:

**Gitleaks** — scans your repository for secrets, runs in CI and locally:

```bash
# Install
brew install gitleaks  # macOS
# or: Download from https://github.com/gitleaks/gitleaks/releases

# Scan your repo
gitleaks detect --source .
```

**Trufflehog** — more thorough, also scans git history and artifacts:

```bash
pip install trufflehog
trufflehog filesystem .
```

Both should run in your CI pipeline. Set them to fail the build if they find anything. This catches the AI mistakes before they reach production.

### Environment variable injection in code

Different languages handle this differently. The pattern is the same everywhere:

**Python:**

```python
import os
api_key = os.environ.get("OPENAI_API_KEY")
if not api_key:
    raise ValueError("OPENAI_API_KEY environment variable is not set")
```

**JavaScript/Node.js:**

```javascript
const apiKey = process.env.OPENAI_API_KEY;
if (!apiKey) {
  throw new Error("OPENAI_API_KEY environment variable is not set");
}
```

**Go:**

```go
import "os"
apiKey := os.Getenv("OPENAI_API_KEY")
if apiKey == "" {
    log.Fatal("OPENAI_API_KEY environment variable is not set")
}
```

The key principle: fail fast if the secret is missing. Don't silently continue with an empty string.

### The principle of least privilege

Every credential should only have the permissions it needs — nothing more.

If your Stripe integration only needs to read charges and issue refunds, your API key should have read and write on charges but nothing else. Not full account access.

This matters because: if your key is leaked, least privilege limits the blast radius. A key with full admin access to your cloud account is a catastrophic leak. A key with scoped read-only access is a manageable incident.

When AI generates code that uses secrets, it doesn't know what permissions your keys have. You do. Review the IAM roles or API key scopes before using any generated code in production.

### API key rotation

Secrets expire. Keys get compromised. Rotation is the process of replacing an old key with a new one without downtime.

The workflow:
1. Generate a new key in the service's dashboard
2. Deploy the new key to your environment (update `.env`, update CI secrets, etc.)
3. Verify the new key works in production
4. Revoke the old key

The practice that makes this easy: externalize all secrets. If your key is in code, you have to redeploy code to rotate it. If it's in environment variables (or a secrets manager like AWS Secrets Manager, HashiCorp Vault, or Doppler), rotation is a config change, not a code change.

## Hands-on exercise

**Goal:** Set up a project that correctly handles a secret, uses `.gitignore`, and runs a secret scan to verify nothing leaked. Takes 15 minutes.

### Step 1: Create a project with a "leaked" secret

```bash
mkdir secret-practice && cd secret-practice
git init
echo "Test project" > README.md
git add README.md && git commit -m "Initial commit"
```

Create a file `config.py` with a hardcoded secret:

```python
# config.py
DATABASE_URL = "postgresql://admin:SuperSecret123@db.example.com:5432/myapp"
print(f"Connecting to database...")
```

Add it and commit:

```bash
git add config.py
git commit -m "Add database config"
```

### Step 2: Fix it the right way

Create a `.env` file (this is the file that will hold your real secret):

```
DATABASE_URL=postgresql://admin:SuperSecret123@db.example.com:5432/myapp
```

Create a new version of `config.py` that reads from the environment:

```python
# config.py
import os

DATABASE_URL = os.environ.get("DATABASE_URL")
if not DATABASE_URL:
    raise ValueError("DATABASE_URL environment variable is not set")

print(f"Connecting to database: {DATABASE_URL.split('@')[1] if '@' in DATABASE_URL else '...'}")
```

### Step 3: Set up .gitignore

Create `.gitignore`:

```
.env
__pycache__/
*.pyc
```

Add and commit the fix:

```bash
git add .gitignore config.py
git commit -m "Move secret to environment variable"
```

### Step 4: Install and run gitleaks to verify

```bash
# macOS
brew install gitleaks

# Or on Windows (using winget or downloading from releases)
# winget install gitleaks

# Scan the repo — if gitleaks finds the old commit, that's expected
# because the secret is in history
gitleaks detect --source .

# Also scan the current state
gitleaks detect --verbose --source .
```

### Step 5: Verify the current state is clean

The current commit should show no secrets if you've done it right. Check:

```bash
git log --oneline
cat config.py  # Should show os.environ.get, no hardcoded secret
```

The old commit still contains the secret — this is why you must treat a committed secret as a security incident and rotate it immediately. The point of this exercise is to see that pattern and build the habit of never committing secrets in the first place.

## Common mistakes

### 1. Committing `.env` because "it's just a local file"

**What happens:** It gets pushed to GitHub. Anyone who ever had read access to the repo now has your production database password. If the repo is public, it's indexed by automated scanners within hours.

**Why it happens:** Beginners create `.env` for local development, add it with `git add .`, and push. The file seems harmless because "it's not really production."

**How to fix:** `echo ".env" >> .gitignore` before your first commit. If you already committed it, treat it as compromised and rotate the credential immediately.

### 2. Thinking `.gitignore` is retroactive

**What happens:** You add `.env` to `.gitignore` after it's been committed. The file stays in the repository. Your secret is still exposed.

**Why it happens:** `.gitignore` only prevents new untracked files from being added. It does not remove already-tracked files from history.

**How to fix:** Remove the file from git tracking first: `git rm --cached .env`. Then commit that removal. The file on disk remains (for your local use), but git stops tracking it. For secrets already in history, you need to rewrite history (BFG Repo-Cleaner or git filter-branch) and force-push.

### 3. Printing secrets in debug output

**What happens:** You add `print(f"API KEY: {api_key}")` to debug why your API call is failing. The print statement ends up in logs, in error tracking services (Sentry, Datadog), or in CI output that gets stored.

**Why it happens:** Debug output feels temporary. It's not.

**How to fix:** When debugging API issues, log the non-sensitive parts (endpoint URL, response status code, latency). If you must log headers, log the header names, not the values. Use structured logging that has explicit allowlists for what gets stored.

### 4. Using the same key across environments

**What happens:** Your Stripe test key and production key are the same. You test in production by accident. Or one environment gets compromised and the attacker has access to everything.

**Why it happens:** It's easier to manage one key. It feels like overkill to create separate keys for dev, staging, and production.

**How to fix:** Each environment gets its own credentials. Use a secrets manager (AWS Secrets Manager, Vault, Doppler) or at minimum separate `.env` files per environment (`.env.development`, `.env.production`) that never get committed.

### 5. Hardcoding secrets in Docker images

**What happens:** You set `ENV OPENAI_API_KEY=sk-...` in a Dockerfile. The image is pushed to a registry. Anyone with access to the image can extract the secret with `docker history`.

**Why it happens:** It works and it's simple.

**How to fix:** Pass secrets at runtime via environment variables or mounted secret files. Use Docker secrets (for Docker Swarm) or Kubernetes secrets (mounted from a secret object, not plain environment variables if you can help it). At minimum, use multi-stage builds so the secret doesn't end up in the final image layer.

## AI-specific pitfalls

### AI hardcodes API keys in generated code

This is the most common AI-generated security failure. The AI sees a pattern like `api_key = ` and fills it in. When it doesn't know a real key, it may hallucinate a plausible-looking one.

**What to look for:** Every `=` assignment in generated code that assigns a string value to something that looks like a credential name (`api_key`, `password`, `secret`, `token`). If the value looks like a real credential format (starts with `sk-`, `AKIA`, `ghp_`, `xoxb-`, etc.), it's almost certainly wrong.

**How to fix:** Before running any AI-generated code, search the file for credential patterns: `grep -n "sk-\|AKIA\|ghp_\|xoxb-" *.py`. Replace hardcoded values with `os.environ.get()` calls.

### AI suggests committing `.env` or creating a template that includes real values

AI sometimes generates `.env.example` files that contain placeholder values. Occasionally it includes values that look like real credentials (hallucinated or pulled from training data).

**What to look for:** Any `.env` file in AI output. Check every variable. Verify placeholder values are clearly fake (e.g., `your-key-here`, `sk-...`).

**How to fix:** Your `.env.example` or `.env.template` should contain only the variable names, never real values: `OPENAI_API_KEY=`. Commit a file with real values in it is a security incident.

### AI doesn't understand that environment variables are still visible in logs

When AI generates debug code or error handling, it may print environment variables, include them in error messages, or log the full request context that contains them.

**What to look for:** Any logging statement in AI output that includes `os.environ`, `process.env`, or HTTP request/response objects. Review what those logs capture before running the code.

**How to fix:** Set a logging filter at the application level that redacts known sensitive field names. Or explicitly log only safe fields (status codes, URLs without query params, timing data).

### AI suggests storing secrets in source code "because it's more secure than environment variables"

This is an occasionally surfacing misconception. Some AI outputs suggest storing secrets in encrypted config files or "secure" source code patterns. While there are legitimate patterns for this (AWS KMS-encrypted configs, HashiCorp Vault integration), the simple version is: environment variables are injected at runtime, are not in the repo, and are standard practice. Don't let AI overcomplicate this into a security risk.

### AI generates SSH keys or TLS certificates in code

If you ask AI to "set up authentication" for a server, it may generate SSH private keys or self-signed TLS certificates as part of the output. These should never be in source code.

**What to look for:** Any `-----BEGIN` blocks in generated code (SSH keys, certificates, private keys).

**How to fix:** Generate these with proper tools (`ssh-keygen`, `openssl`), store them in proper locations (`~/.ssh/`, managed secrets), and reference them by path in your code.

## Quick reference

### Secret patterns to watch for

| Pattern | Example | Risk |
|---------|---------|------|
| OpenAI key | `sk-1234567890abcdef...` | High — monetary cost, data exposure |
| AWS access key | `AKIAIOSFODNN7EXAMPLE` | High — cloud account takeover |
| GitHub token | `ghp_1234567890abcdef...` | High — repo access, CI compromise |
| Stripe key | `sk_live_1234567890...` | High — payment fraud |
| Generic secret | `password = "..."` | High — credential reuse |

### .gitignore minimum for any project

```
.env
.env.local
.env.*.local
*.pem
*.key
__pycache__/
node_modules/
```

### Environment variable checklist

```
Before shipping any code:
[ ] No hardcoded credential strings in source
[ ] All secrets read via os.environ.get() / process.env
[ ] .env is in .gitignore
[ ] .env.example exists with only keys, no values
[ ] CI runs secret scanner (gitleaks / trufflehog)
[ ] Failed scan = blocked build
```

### Secret scanning commands

```bash
# Gitleaks — scan working directory
gitleaks detect --source .

# Gitleaks — scan a specific branch
gitleaks detect --source . --branch main

# Trufflehog — scan filesystem
trufflehog filesystem .

# Trufflehog — scan git history (finds previously committed secrets)
trufflehog git https://github.com/your-org/your-repo.git
```

### Decision tree: Secret found in repo

```
Secret found in committed code:
  1. Rotate the credential immediately (assume it's compromised)
  2. Remove the secret from code (fix the source)
  3. Commit the fix
  4. For history: use BFG Repo-Cleaner or git filter-branch
     - BFG: bfg --delete-files .env
     - git filter-branch: git filter-branch --force --index-filter \
       'git rm --cached --ignore-unmatch .env' --prune-empty --tag-name-filter cat -- --all
  5. Force push to update remote history
  6. Verify with gitleaks
  7. Notify relevant stakeholders (this is an security incident)
```

## Go deeper

- [Gitleaks — GitHub Secret Scanning Tool](https://github.com/gitleaks/gitleaks) — The standard open-source tool for scanning repos for secrets. Runs in CI and locally. (verified 2026-01)
- [Trufflehog — Find credentials in your codebase](https://github.com/trufflesecurity/trufflehog) — Goes deeper than gitleaks, scans git history and various data sources. (verified 2026-01)
- [OWASP Security Knowledge Fundamentals](https://cheatsheetseries.owasp.org/cheatsheets/ secrets_management_cheat_sheet.html) — The authoritative security cheatsheet series, including secrets management. (verified 2026-01)
- [GitHub Secret Scanning Documentation](https://docs.github.com/en/code-security/secret-scanning) — If you use GitHub, this is free and runs automatically on public repos and optionally on private repos. (verified 2026-01)
- [AWS IAM Best Practices — Least Privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#best-practices-least-privilege) — The principle of least privilege explained in depth with AWS examples. (verified 2026-01)