# PRD: `no-vibe-without-basics` — A Foundational Curriculum for AI-Assisted Development

**Version:** 2.0  
**Author:** Trevor Poon  
**Date:** April 13, 2026  
**Status:** Draft

***

## Overview

`no-vibe-without-basics` is an open-source GitHub repository that teaches developers the foundational technical knowledge required *before* relying on AI-assisted (vibe) coding tools. Most existing resources focus on the tools themselves — Cursor, Claude Code, Copilot — but none address the prerequisite mental models and skills that determine whether AI-generated code succeeds or fails in production. This repo fills that gap.

***

## Problem Statement

Vibe coding lowers the barrier to building software dramatically, but it creates a new class of failure: developers ship AI-generated code without understanding what it does, why it's structured the way it is, or how to fix it when it breaks. The root cause is not the AI — it's the absence of foundational knowledge that would let a developer guide, validate, and debug AI output confidently.

Specific failure patterns observed in the community:

- AI generates working code but developer can't checkpoint or roll back changes (no Git knowledge)
- Dependency conflicts crash projects because no virtual environment was used
- AI-generated database schemas are insecure or unscalable because the developer couldn't review them
- Vibe-coded apps fail to deploy because developer lacks infrastructure/architecture context
- Prompts are vague because developer can't write a structured PRD or CLAUDE.md
- Secrets and API keys committed to public repos due to no `.env` discipline
- AI-written authentication code shipped without understanding session vs. JWT trade-offs
- No tests written, so regressions from iterative AI edits go undetected until production
- App performance degrades at scale because the developer never learned indexing or caching
- Logs missing in production so debugging AI-introduced bugs is nearly impossible

***

## Goals

1. Provide a structured, tiered curriculum covering the technical foundations every vibe coder needs
2. Be language- and tool-agnostic where possible; opinionated where it matters (e.g., recommend `uv` over `pip`)
3. Be completable at the reader's own pace — no login, no paywall, no course platform required
4. Serve both total beginners (Tier 1) and developers expanding into new domains (Tier 3–4)
5. Become the canonical "read this before you vibe code" reference linked by the community
6. Cover AI-native development patterns as a first-class tier — not an afterthought

***

## Non-Goals

- This is not a vibe coding tools tutorial (no Cursor/Claude Code walkthroughs)
- This is not a full computer science curriculum — depth is scoped to what's actionable for vibe coders
- This is not a deployment pipeline guide — infrastructure concepts are covered at the conceptual level
- This repo does not host video content; it links to external resources where appropriate

***

## Target Audience

| Persona | Background | Primary Need |
|---|---|---|
| **The Non-Technical Builder** | Wants to ship an app using AI, zero dev background | Tier 1 fundamentals — Git, env, structure |
| **The Domain Expert** | e.g., analyst, designer — knows their field, not code | Tier 1–2, especially DB and architecture basics |
| **The Junior Developer** | Can code basics but jumps straight to AI tools | Tier 2–3, especially infrastructure and systems design |
| **The Experienced Dev Onboarding to AI** | Strong traditional skills, new to AI-assisted workflow | Tier 4: AI-native development patterns |
| **The Security-Conscious Builder** | Ships fast but wants to avoid breach headlines | Tier 1 (secrets), Tier 2 (auth), Tier 3 (hardening) |

***

## Curriculum Structure

The curriculum is divided into four tiers, each building on the last. Each topic is a standalone Markdown file with: a plain-English explanation of the concept, why it matters for vibe coding specifically, a minimal working example or exercise, common mistakes, and links to deeper resources.

---

### Tier 1 — Essentials (Everyone)

These are non-negotiable prerequisites before writing a single AI prompt for code.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `01-git.md` | init, add, commit, push, branch, merge, PR, .gitignore, rebase, stash, cherry-pick, bisect | Checkpointing AI output; recovering from hallucinated rewrites; bisecting to find when an AI edit broke something |
| `02-virtual-envs.md` | uv, miniforge, conda, nvm, pyenv, venv, lockfiles, dependency pinning | Isolating AI-generated dependencies; reproducible environments; avoiding "works on my machine" |
| `03-terminal.md` | Navigation, file ops, SSH, pipes, redirects, process management, tmux, environment variables, shell scripting basics | Running AI-generated scripts; remote development; understanding what AI-generated shell commands do before running them |
| `04-project-structure.md` | Folder conventions, .env, config separation, README, monorepo vs. polyrepo, src layout vs. flat layout | Giving the AI coherent context about the project; preventing AI from generating files in wrong locations |
| `05-context-engineering.md` | CLAUDE.md, INITIAL.md, PRDs, AGENTS.md, memory files, system prompts, prompt caching | Structured prompting; preventing hallucination and drift; giving AI persistent context across sessions |
| `06-debugging-basics.md` | Reading stack traces, error types, print/log debugging, browser DevTools, breakpoints, isolating bugs, rubber duck method | Diagnosing AI-introduced bugs; understanding error messages the AI generates; validating AI fixes before accepting |
| `07-secrets-and-security-basics.md` | .env files, .gitignore for secrets, environment variable injection, secret scanning (gitleaks, trufflehog), API key rotation, principle of least privilege | Preventing AI from committing secrets; understanding why AI-generated code that hardcodes keys is dangerous |
| `08-reading-code.md` | Control flow tracing, call graph mental model, grep/search strategies, reading diffs, understanding imports/modules, naming conventions | Reviewing AI output critically; understanding what generated code actually does before running it |

---

### Tier 2 — Builder Foundations

For anyone shipping something real beyond a personal script.

| Module | Key Concepts | Vibe Coding Relevance | Expanded Description |
|---|---|---|---|
| `09-web-basics.md` | HTML/CSS/JS mental model, DOM, HTTP lifecycle, REST, status codes, request/response headers, CORS, cookies, JSON | Understanding AI-generated web code; debugging API calls; knowing what to look for in network traces | Builds the mental model for how web pages actually work under the hood. Learners understand the request/response lifecycle of HTTP, how browsers parse HTML into a DOM tree, and why CORS exists as a safeguard. **Sub-topics:** HTTP methods and status code families (2xx, 4xx, 5xx); how cookies and headers travel between client and server; the Same-Origin Policy and CORS preflight mechanics. **Common AI output to review:** An AI-generated `fetch()` call that omits error handling for non-2xx responses; code that assumes CORS is a server-side fix when it also requires client header awareness. |
| `10-frontend-backend.md` | Client/server boundary, rendering strategies (SSR/CSR/SSG/ISR), API contracts, BFF pattern, hydration | Knowing what to ask the AI to build where; preventing AI from blurring server/client boundaries in frameworks like Next.js | Clarifies where code runs and why it matters for app architecture. Learners understand the distinction between client-side and server-side rendering, when each strategy is appropriate, and how the BFF pattern structures API access. **Sub-topics:** SSR vs. CSR vs. SSG; hydration and why it causes flashing when AI misplaces logic; API contract design to avoid client/server drift. **Common AI output to review:** An AI-generated Next.js page that mixes `use client` and server components incorrectly, causing hydration errors; a "smart" AI that puts business logic in the browser that belongs on the server. |
| `11-databases.md` | SQL vs. NoSQL, schemas, normalization, migrations, ORMs, indexing, transactions, ACID, .env secrets, connection pooling | Reviewing AI-generated schemas; preventing data loss; understanding why AI-generated migrations need human review | Covers how data is structured, persisted, and queried in production. Learners develop judgment about when to use a relational vs. document store, how to read an ORM-generated migration, and why database transactions exist. **Sub-topics:** Schema design and normalization forms; writing safe migrations that can be rolled back; indexing strategy — when an AI-generated query needs an index added. **Common AI output to review:** An AI-generated Prisma or SQLAlchemy migration that drops an existing column without a backup plan; a schema that uses a OneToMany relationship where a ManyToMany is actually needed. |
| `12-programming-languages.md` | Python, TypeScript, Go, Rust, Ruby — trade-offs, type systems, memory models, ecosystem maturity, build tooling | Picking the right language before prompting; reading AI output critically; understanding type errors AI produces | Builds the judgment needed to choose a language and interpret AI-generated code in context. Learners compare how different languages handle types, concurrency, and memory, and why those trade-offs affect the code an AI produces. **Sub-topics:** Static vs. dynamic typing and what AI gets wrong with each; memory models — stack vs. heap, ownership in Rust, garbage collection in Python/Go; when ecosystem maturity matters for AI-generated reliability. **Common AI output to review:** Python code with type hints that AI generates confidently but are semantically wrong at runtime; Go code that uses goroutines naively, leading to goroutine leaks under load. |
| `13-apis-and-integrations.md` | REST vs. GraphQL vs. gRPC, API authentication (API keys, OAuth tokens, HMAC), webhooks, rate limiting, pagination, SDK vs. raw HTTP, OpenAPI specs | Integrating third-party services AI suggests; validating AI-generated API client code; avoiding rate limit surprises | Teaches how services communicate over the network and how to evaluate AI suggestions for third-party integrations. Learners understand REST conventions deeply enough to spot when an AI violates them. **Sub-topics:** REST resource modeling and idempotency; OAuth2 flows — Authorization Code with PKCE for SPAs, Client Credentials for machine-to-machine; pagination patterns (cursor-based vs. offset) and their performance implications. **Common AI output to review:** An AI-generated API client that calls a rate-limited endpoint in a tight loop without backoff; code that uses an API key stored in a string literal instead of injected from environment variables. |
| `14-testing-fundamentals.md` | Unit tests, integration tests, end-to-end tests, test doubles (mocks/stubs/fakes), code coverage, TDD basics, property-based testing, snapshot testing | Catching AI-introduced regressions; writing tests to constrain AI edits; understanding why AI-generated tests can be useless | Establishes a practical testing vocabulary so learners can direct AI to write meaningful tests and review what it produces. Learners understand what each test type is actually testing, why high coverage is not the same as good coverage, and how mocks can hide bugs. **Sub-topics:** The testing pyramid — where unit, integration, and E2E tests sit and why each level matters; using test doubles to isolate units; property-based testing for finding edge cases AI would miss. **Common AI output to review:** An AI-generated test suite that achieves 90% coverage by testing trivial getters and setters while leaving business logic untested; snapshot tests committed without human review of the snapshot diff. |
| `15-auth-and-identity.md` | Authentication vs. authorization, password hashing (bcrypt, argon2), sessions vs. JWT, OAuth2 flows (PKCE, client credentials), RBAC, MFA basics, SSO | Reviewing AI-generated auth code for correctness; choosing the right auth pattern before prompting; not shipping broken access control | Demystifies authentication and authorization so learners can catch critical mistakes in AI-generated security code. Learners understand the difference between authentication (who are you?) and authorization (what can you do?). **Sub-topics:** Password hashing — why bcrypt/argon2 and not MD5 or SHA-1; JWT structure (header, payload, signature) and why storing JWTs in localStorage is a known risk; RBAC vs. ABAC and when role-based is sufficient. **Common AI output to review:** AI-generated login code that compares passwords with `===` instead of using a constant-time comparison; an AI that suggests storing JWTs in cookies without setting `HttpOnly`, `Secure`, and `SameSite` flags. |

---

### Tier 3 — Advanced Concepts

For scaling beyond a side project or working on production systems.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `16-infrastructure.md` | Cloud providers (AWS/GCP/Azure), IaaS vs. PaaS vs. FaaS, Docker, containers, Kubernetes basics, secrets management (Vault, AWS Secrets Manager), environment parity, Terraform basics | Deploying AI-built apps; environment debugging; understanding what AI-generated Dockerfiles and infra configs do |
| `17-architecture.md` | Monolith vs. microservices vs. modular monolith, C4 diagrams, API design (versioning, idempotency, error contracts), event-driven architecture, CQRS basics, auth patterns (JWT, OAuth, API gateway) | Designing systems AI can implement coherently; preventing AI from generating architecturally incoherent code |
| `18-deployment.md` | CI/CD basics (GitHub Actions, GitLab CI), reverse proxies (nginx/Caddy), domains and DNS, HTTPS/TLS, zero-downtime deploys (blue-green, canary), health checks, rollback strategies | Getting AI-built apps to production and keeping them running; understanding what AI-generated CI pipelines do |
| `19-observability.md` | Structured logging (JSON logs, log levels), metrics (counters, gauges, histograms), distributed tracing (OpenTelemetry), alerting, dashboards (Grafana, Datadog), SLIs/SLOs/SLAs, error budgets | Debugging AI-introduced production bugs; knowing what to instrument in AI-generated code before deploying |
| `20-performance-and-scaling.md` | Profiling (CPU, memory, I/O), caching strategies (in-memory, Redis, CDN, HTTP cache headers), database query optimization, N+1 problem, connection pooling, horizontal vs. vertical scaling, load balancing | Identifying performance issues in AI-generated code; knowing which AI-generated queries need an index |
| `21-security-hardening.md` | OWASP Top 10, input validation and sanitization, SQL injection, XSS, CSRF, dependency auditing (npm audit, pip audit, Snyk), rate limiting, Content Security Policy, HTTPS enforcement, supply chain risks | Reviewing AI-generated code for security vulnerabilities; understanding why AI output needs a security pass before shipping |

---

### Tier 4 — AI-Native Development

For developers who want to work *with* AI as a force multiplier, not just accept its output passively.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `22-prompt-engineering-for-code.md` | Structured prompts, role/context/task/format framing, chain-of-thought for complex tasks, few-shot examples in prompts, iterative refinement, negative constraints, diff-based prompting | The meta-skill: all other modules become prompts — knowing how to write them well is what separates vibe coders from vibe drifters |
| `23-reviewing-ai-output.md` | Code review checklist for AI output, spotting hallucinated APIs, detecting subtle logic errors, verifying library versions exist, diff review discipline, security red flags, when to reject vs. refine | AI output is never the final word — systematic review is the human's most important role in the loop |
| `24-agentic-workflows.md` | Multi-agent patterns, MCP (Model Context Protocol), tool use and function calling, autonomous coding agents (Claude Code, Devin, Copilot Workspace), agent memory (files vs. vector DBs), context window management, checkpointing | The future of software development — understanding agent capabilities and limits before delegating high-stakes work |
| `25-llm-fundamentals-for-builders.md` | Tokens and context windows, temperature and sampling, system prompts vs. user prompts, model selection trade-offs (cost/speed/capability), prompt caching, structured output (JSON mode), embeddings and RAG basics | Being an intelligent consumer of AI — knowing why the model behaved the way it did and how to adjust |

***

## Repo Structure

```
no-vibe-without-basics/
├── README.md                         ← Roadmap, philosophy, how to use this repo
├── CONTRIBUTING.md                   ← Guide for contributors
├── tier-1-essentials/
│   ├── 01-git.md
│   ├── 02-virtual-envs.md
│   ├── 03-terminal.md
│   ├── 04-project-structure.md
│   ├── 05-context-engineering.md
│   ├── 06-debugging-basics.md
│   ├── 07-secrets-and-security-basics.md
│   └── 08-reading-code.md
├── tier-2-builder/
│   ├── 09-web-basics.md
│   ├── 10-frontend-backend.md
│   ├── 11-databases.md
│   ├── 12-programming-languages.md
│   ├── 13-apis-and-integrations.md
│   ├── 14-testing-fundamentals.md
│   └── 15-auth-and-identity.md
├── tier-3-advanced/
│   ├── 16-infrastructure.md
│   ├── 17-architecture.md
│   ├── 18-deployment.md
│   ├── 19-observability.md
│   ├── 20-performance-and-scaling.md
│   └── 21-security-hardening.md
├── tier-4-ai-native/
│   ├── 22-prompt-engineering-for-code.md
│   ├── 23-reviewing-ai-output.md
│   ├── 24-agentic-workflows.md
│   └── 25-llm-fundamentals-for-builders.md
└── checklists/
    ├── before-you-start.md           ← Tier 1 checklist before first AI prompt
    ├── before-you-ship.md            ← Tier 2–3 checklist before deploying
    └── ai-code-review.md             ← NEW: Checklist for reviewing any AI-generated code
```

***

## Module Template

Every module follows this structure to ensure consistency:

```markdown
# [Topic Name]

## What is it?
Plain-English explanation. No assumed knowledge. Analogies welcome.

## Why it matters for vibe coding
Specific failure modes that occur when this knowledge is missing.
Concrete examples: "Without this, the AI will X and you won't know why."

## The 20% you need to know
Minimal viable understanding — the concepts that cover 80% of real situations.
Sub-sections permitted for distinct sub-topics.

## Hands-on exercise
A concrete, runnable task (not just reading).
Must be completable in 15 minutes or less.
Must produce visible output the learner can verify.

## Common mistakes
The top 3–5 errors beginners make, and how to avoid them.
Each mistake framed as: what happens, why it happens, how to fix it.

## AI-specific pitfalls
Things AI tools specifically get wrong or generate poorly for this topic.
What to look for when reviewing AI output related to this module.

## Quick reference
A cheat sheet, command table, or decision tree the reader can bookmark.

## Go deeper
3–5 curated external links (documentation, tutorials, videos).
Prefer official docs and free resources.
```

***

## Checklist Files

### `before-you-start.md`
A Tier 1 gate before writing the first AI prompt. Run this checklist before you write your first prompt to an AI coding tool.

#### Git & Version Control
- [ ] `git init` run in the project root — repo exists before any code is written
- [ ] `.gitignore` created covering: `.env`, `node_modules/`, `__pycache__/`, `.venv/`, `*.pyc`, `.DS_Store`, build artifacts, and secrets files
- [ ] First commit made with a meaningful message (e.g., "Initial project scaffold")
- [ ] `git log` shows a clean, linear history — no "initial commit" followed by "oops" commits
- [ ] You can run `git status` and `git diff` without errors
- [ ] At least one remote (GitHub/GitLab/Bitbucket) set up and pushing works

#### Environment Setup
- [ ] Virtual environment created (e.g., `uv venv` or `python -m venv .venv`) and activated
- [ ] Language runtime installed and verified (`python --version`, `node --version`, etc.)
- [ ] `requirements.txt`, `pyproject.toml`, `package.json`, or equivalent lockfile exists and is committed
- [ ] You can run a "hello world" in the project without dependency errors
- [ ] `.env` file created with a `.env.example` sibling documenting all required variables

#### Project Context
- [ ] `README.md` exists with: what the project does, how to set it up, and how to run it
- [ ] Project structure is understood by you — you can explain the folder layout in plain English
- [ ] Entry point identified (e.g., `main.py`, `index.js`, `app.py`) and it runs successfully
- [ ] Any external services (database, API keys, third-party APIs) are noted in `.env.example`

#### AI Setup
- [ ] `CLAUDE.md` or `AGENTS.md` created in the project root with: project name, tech stack, folder conventions, and a one-sentence purpose
- [ ] You have tested that your AI tool (Claude Code, Cursor, Copilot) can see and read the project files
- [ ] You know how to point the AI at specific files or folders using the tool's context mechanism
- [ ] You've confirmed the AI respects `.gitignore` and won't suggest committing `.env` or `node_modules`

---

### `before-you-ship.md`
A Tier 2–3 gate before deploying to production or making a feature available to users.

#### Security
- [ ] All secrets (API keys, database URLs, credentials) are in environment variables — zero hardcoded values in source code
- [ ] `.env` is in `.gitignore` and has never been committed to version control
- [ ] Authentication is implemented on every protected route/endpoint — no unprotected `/admin` or debug routes
- [ ] Passwords are hashed (bcrypt/argon2) — no plaintext password storage
- [ ] Rate limiting is configured on public-facing API endpoints
- [ ] CORS policy is explicitly set — no wildcard `*` on production APIs handling cookies or auth headers
- [ ] Input validation runs on all user-supplied data before it reaches the database or business logic

#### Data & Migrations
- [ ] Database schema migrations are written and tested on a staging/clone environment before production
- [ ] Rollback path for the migration exists and has been tested
- [ ] No destructive migration (drop table, drop column) runs without a full backup taken first
- [ ] Foreign key constraints and indexes are in place — AI-generated schemas reviewed for missing relationships
- [ ] Connection pooling is configured for the database (no unbounded connection creation under load)

#### Testing
- [ ] At least one integration test covers the critical user path (e.g., sign up → log in → core action)
- [ ] Unit tests exist for any non-trivial business logic or data transformation
- [ ] All tests pass locally (`npm test`, `pytest`, `go test`, etc.)
- [ ] Test coverage is above 60% for the core domain modules — check with `coverage run -m pytest` or equivalent
- [ ] No test is skipped or commented out — if a test doesn't run, it has a linked issue

#### Observability
- [ ] Structured logging is implemented — JSON logs with `level`, `timestamp`, `message`, and `request_id` fields
- [ ] All HTTP endpoints and database calls have error logging on failure paths
- [ ] A health check endpoint (`/health` or `/ready`) exists and returns 200 when the service is healthy
- [ ] Key business events (user sign-up, payment, error conditions) are logged with enough context to debug

#### Deployment
- [ ] CI pipeline runs on every PR: lint, type check, tests — blocks merge on failure
- [ ] Production environment parity confirmed: same runtime version, same env var names, same database engine
- [ ] Rollback plan documented: how to revert to the previous release in under 5 minutes
- [ ] Zero-downtime deploy configured (blue-green, canary, or rolling update) — no downtime for live users
- [ ] HTTPS enforced on all public endpoints — no HTTP-only routes in production

---

### `ai-code-review.md`
A per-PR checklist for reviewing any AI-generated code. Apply this before merging any AI-assisted change.

#### Correctness
- [ ] Every imported library or package exists at the version referenced — verified with `pip show`, `npm list`, or equivalent
- [ ] No AI-hallucinated APIs — all function calls, method names, and library interfaces verified against real documentation
- [ ] Control flow is traced end-to-end: the code does what the PR description says it does
- [ ] Edge cases handled: empty inputs, null values, boundary conditions — not just the happy path
- [ ] Return types and error responses are consistent — no silent failures or unhandled promise rejections

#### Security
- [ ] No hardcoded secrets, API keys, connection strings, or environment-specific values in the diff
- [ ] All user input is validated and sanitized before use in queries, shell commands, or file paths
- [ ] No SQL built by string concatenation or f-strings — parameterized queries used throughout
- [ ] Auth and authorization checks present on every route/endpoint that accesses protected data
- [ ] No `eval()`, `exec()`, or `subprocess` calls with unsanitized user input
- [ ] Dependencies audited: `npm audit`, `pip audit`, or `cargo audit` shows no high/critical vulnerabilities

#### Performance
- [ ] No N+1 query patterns — database access inside loops is eliminated or batched
- [ ] Indexes reviewed for any new queries — slow queries verified with `EXPLAIN ANALYZE` or query plans
- [ ] No unnecessary memory allocation in hot paths (no repeated string concatenation, no unbounded list growth)
- [ ] Connection pooling is used for all database and HTTP client calls
- [ ] Large responses are streamed or paginated — no unbounded in-memory data structures for large datasets

#### Maintainability
- [ ] No `TODO` or `FIXME` comments without a linked issue or ticket
- [ ] Function and variable names are descriptive — AI-generated names reviewed for accuracy
- [ ] No magic numbers or strings — constants are named and extracted
- [ ] The diff is readable: changes are focused and self-contained, not a rewrite of the entire file
- [ ] Tests added or updated to cover the new code path — AI-generated tests reviewed for actual coverage

***

## Content Guidelines

### Audience and Voice

Every module is written for a developer who is competent in their domain but unfamiliar with how foundational knowledge maps to AI-assisted workflows. They are not a beginner — they can read code, use a terminal, and build something that works. What they lack is the mental model that lets them guide, validate, and debug AI output confidently.

Write as a senior engineer explaining concepts to a smart colleague over coffee. Do not write as a textbook author or a tutorial instructor. The reader is busy and motivated — they will disengage if you over-explain or treat them as a novice.

**Tone principles:**
- **Direct**. State the concept, then state why it matters for vibe coding. Do not bury the lede.
- **Practical first**. Theory earns its place only when it explains a concrete failure mode.
- **Honest about trade-offs**. If something is genuinely disputed in the industry, say so and explain the trade-off rather than picking a side arbitrarily.
- **Conversational but precise**. Prefer short sentences. Avoid filler phrases like "it's worth noting that" or "in this section we will discuss."
- **No condescension**. Do not qualify every statement with "of course" or "simply." These phrases signal impatience and erode trust.

**What "write for them" means in practice:**
- Define terms the reader would encounter in AI prompts but might not fully understand (e.g., "branch," "context window," "migration").
- Do not define terms the reader already knows from their primary domain (e.g., do not explain "stateless" to an infrastructure engineer).
- When an analogy helps, use it — but only to explain something genuinely new. If the analogy requires its own explanation, find a different one.

### Module Length Standards

Each module is scoped to **10–15 minutes of reading time** — not because the concepts are shallow, but because the reader needs to leave with actionable knowledge, not a surface overview.

**Word count target: 1,500–2,500 words** (excluding code blocks, links, and the quick reference section). Modules under 1,200 words likely lack sufficient depth. Modules over 3,000 words are too dense for a single sitting.

Use these benchmarks to estimate:
- 250 words ≈ 1 minute of reading for a technical professional reading carefully
- A section with 4–5 sub-headings and two code examples is roughly the right density
- If your "hands-on exercise" section requires more than 400 words of setup explanation, the exercise is too complex — simplify it

**Sizing guidance by section:**
| Section | Target words | Purpose |
|---|---|---|
| What is it? | 150–300 | Establish the concept with a concrete analogy |
| Why it matters for vibe coding | 200–350 | Connect the concept to a specific AI failure mode — this is the section that makes the module worth reading |
| The 20% you need to know | 600–1,000 | Core substance — concise, not comprehensive |
| Hands-on exercise | 200–400 | Must produce visible, verifiable output in 15 minutes |
| Common mistakes | 250–400 | 3–5 mistakes, each with what/why/fix |
| AI-specific pitfalls | 150–300 | This is non-negotiable — every module gets one |
| Quick reference | Variable | Cheat sheets, command tables, decision trees |
| Go deeper | Variable | 3–5 links, each with `(verified YYYY-MM)` |

### Exercise and Example Recommendations by Tier

Exercises and examples must be concrete and verifiable. The reader must be able to complete them in 15 minutes with a visible result.

**Tier 1 — Essentials**
- **Exercise type:** Terminal commands, file creation, Git operations, `.env` setup
- **Standard:** Every Tier 1 module must end with a terminal exercise the reader can run and verify. Reading alone is insufficient.
- **Time to complete:** Target 5–10 minutes

**Tier 2 — Builder Foundations**
- **Exercise type:** Code that runs, SQL queries, API calls, small integrations
- **Standard:** Exercises should produce a working artifact — a query that returns correct results, a small API that responds, a test that passes.
- **Time to complete:** Target 8–12 minutes

**Tier 3 — Advanced Concepts**
- **Exercise type:** Architecture diagrams, configuration files, log parsing, performance profiling
- **Standard:** Exercises should require the reader to make a decision and justify it — not just follow steps.
- **Time to complete:** Target 10–15 minutes

**Tier 4 — AI-Native Development**
- **Exercise type:** Prompt writing, prompt iteration and refinement, output review using a checklist
- **Standard:** Exercises require the reader to produce prompts and evaluate results against a rubric.
- **Time to complete:** Target 10–15 minutes

### What to Avoid — Rejection Criteria

A PR will be returned or closed if it meets any of the following criteria:

1. **No vibe-coding context in any section.** If a module does not explicitly answer "why does this matter for AI-assisted coding," it does not belong in this curriculum.
2. **Exercise is not runnable or not verifiable.** A module without a hands-on exercise will be returned. Exercises requiring more than 15 minutes of setup will be returned.
3. **Module exceeds 3,500 words.** Split into two modules or scope more tightly.
4. **AI-specific pitfalls section is missing or generic.** "AI can make mistakes" is not an AI-specific pitfall. Specific failure modes are required.
5. **External links lack verification dates.** All links must include `(verified YYYY-MM)`. Links older than 12 months require re-verification.
6. **Module is a tool walkthrough.** This repo does not cover Cursor, Claude Code, Copilot, or any specific AI tool as a primary subject.
7. **Opinionated recommendation without justification.** If you recommend `uv` over `pip`, explain *why* — what failure does it prevent?
8. **Content that is not original or properly attributed.** Summarize in your own words and link to the source.

### Definition of Done

A module is considered complete — ready for review and merge — when all of the following are true:

**Structural requirements:**
- [ ] Follows the module template exactly (all sections present in order)
- [ ] All section headings match the template naming
- [ ] No placeholder text (e.g., "TODO: add example here")
- [ ] Word count is within the 1,500–2,500 word target range
- [ ] Hands-on exercise is runnable and completes within 15 minutes
- [ ] Exercise output is clearly described so the reader knows what success looks like

**Content requirements:**
- [ ] "Why it matters for vibe coding" names a specific, concrete failure mode — not a generic statement
- [ ] "AI-specific pitfalls" names at least 2 specific, actionable pitfalls that a real AI tool has produced
- [ ] "Go deeper" has 3–5 links, each with `(verified YYYY-MM)`
- [ ] "The 20% you need to know" covers the most common real-world situations, not edge cases
- [ ] "Common mistakes" has 3–5 entries, each with what happened, why, and how to fix it

**Quality requirements:**
- [ ] Read the module out loud — if it sounds like a textbook, rewrite it
- [ ] Every code block is tested and produces the described output
- [ ] No opinionated recommendation without a "why" nearby
- [ ] Module connects back to the curriculum's purpose: making AI-assisted development more reliable

***

## Suggested Learning Paths

Rather than forcing a linear read, suggest paths by persona. Answer the questions below to find your starting point.

### Choosing Your Path

Answer these 5 questions to find the right track:

1. **Do you have any coding experience at all?**
   - No / Very little → **Path E**
   - Yes, I can write basic code → Question 2

2. **Do you manage or oversee developers who use AI coding tools?**
   - Yes (tech lead, manager, CTO) → **Path F**
   - No → Question 3

3. **Are you trying to ship a specific app or product?**
   - Yes → **Path A** (if security/quality matters: **Path C**)
   - No, I want to become a better developer first → **Path B**

4. **Does your app need to handle real users, scale, or run continuously?**
   - Yes → **Path D**
   - No / Not sure yet → **Path A**

5. **Is your top priority security and correctness over speed?**
   - Yes → **Path C**
   - I want to go fast but not at the expense of safety → **Path A**

---

### Path A — "I want to ship my first app with AI"

**Best for:** Anyone with basic coding knowledge who wants to build and deploy a real application using AI tools.

**Prerequisites:** None beyond basic computer literacy (can install software, navigate folders, read documentation).

**Estimated time commitment:** 15–25 hours total (Tier 1 modules at ~1–2 hours each, plus 1–2 weeks of guided project work).

**Modules:**
1. Tier 1 in full (`01-git.md` → `08-reading-code.md`)
2. `09-web-basics.md`
3. `11-databases.md`
4. `15-auth-and-identity.md`
5. `checklists/before-you-ship.md`

**You will be able to:**
- Set up a complete, professional project with Git, virtual environments, and proper structure
- Give the AI consistent, structured context via CLAUDE.md so it doesn't lose the plot across sessions
- Understand what the AI generates — not just accept it blindly
- Ship a basic web app with a database, authentication, and environment-variable-based secrets
- Debug and checkpoint AI-generated code without losing work

---

### Path B — "I'm a developer new to AI tools"

**Best for:** Traditional developers (self-taught, bootcamp, or CS background) who can code but have not yet integrated AI tools into their workflow.

**Prerequisites:** Comfortable with at least one programming language and basic Git usage.

**Estimated time commitment:** 10–15 hours total.

**Modules:**
1. `05-context-engineering.md`
2. `08-reading-code.md`
3. `22-prompt-engineering-for-code.md`
4. `23-reviewing-ai-output.md`
5. `24-agentic-workflows.md`

**You will be able to:**
- Write structured prompts that produce usable code rather than vague suggestions
- Critically review and validate AI output before running it
- Set up persistent context files (CLAUDE.md, AGENTS.md) that keep AI from drifting across sessions
- Delegate tasks to autonomous AI agents with appropriate boundaries and checkpoints
- Distinguish between code that looks plausible and code that actually works

---

### Path C — "I need to ship safely, not just fast"

**Best for:** Developers shipping real applications to real users and cannot afford security breaches, data loss, or production outages.

**Prerequisites:** Basic coding knowledge; familiar with at least one language's package manager.

**Estimated time commitment:** 12–18 hours total.

**Modules:**
1. `07-secrets-and-security-basics.md`
2. `14-testing-fundamentals.md`
3. `15-auth-and-identity.md`
4. `21-security-hardening.md`
5. `checklists/ai-code-review.md`

**You will be able to:**
- Prevent AI from committing secrets, API keys, or credentials to version control
- Write tests that catch regressions introduced by iterative AI edits
- Review AI-generated authentication code for correctness and security soundness
- Identify the most common vulnerability patterns (OWASP Top 10) in AI-generated code
- Apply a systematic security checklist to every AI-assisted PR before it reaches production

---

### Path D — "I'm building something that needs to scale"

**Best for:** Developers building production-facing applications expected to serve real users, handle significant traffic, or run continuously.

**Prerequisites:** Comfortable with Tier 2 concepts; has shipped at least one project before.

**Estimated time commitment:** 20–30 hours total (Tier 2 full + Tier 3 selected modules).

**Modules:**
1. Tier 2 in full (`09-web-basics.md` → `15-auth-and-identity.md`)
2. `16-infrastructure.md`
3. `19-observability.md`
4. `20-performance-and-scaling.md`
5. `18-deployment.md`

**You will be able to:**
- Design systems that AI can implement coherently (not just prompts for individual files)
- Understand what AI-generated Dockerfiles, CI pipelines, and infrastructure configs actually do
- Instrument logging and metrics so you can debug production issues introduced by AI edits
- Identify N+1 queries, missing indexes, and connection pool exhaustion in AI-generated code
- Deploy with confidence using blue-green or canary strategies, with a rollback plan in place

---

### Path E — "I'm a total beginner with zero coding experience"

**Best for:** People who have never written code, never used Git, and never touched a terminal — but want to understand how to build with AI tools.

**Prerequisites:** None. This path starts at absolute zero.

**Estimated time commitment:** 25–40 hours of guided learning + 4–8 weeks of guided first project work.

**Modules:**
1. Tier 1 in full (`01-git.md` → `08-reading-code.md`) — take your time here, do every exercise
2. `checklists/before-you-start.md` — complete this checklist in a new project before writing a single AI prompt
3. Guided first project: apply Tier 1 skills to a minimal AI-assisted project (CLI tool, web scraper, or basic CRUD app)
4. After the first project: revisit `08-reading-code.md` and `06-debugging-basics.md` with fresh eyes

**You will be able to:**
- Navigate a terminal, manage files, and understand what commands you are running (and why)
- Use Git to checkpoint your work and recover from mistakes — including AI mistakes
- Set up isolated development environments so AI-generated dependencies don't corrupt other projects
- Give AI consistent project context so it can work across sessions without forgetting what it already built
- Read and trace through code the AI generates so you can catch obvious errors before they cause problems
- Avoid the most common beginner traps: committed secrets, broken environments, "works on my machine" syndrome

---

### Path F — "I manage developers who use AI tools"

**Best for:** Tech leads, engineering managers, and CTOs who need to supervise, review, and mentor developers using AI-assisted workflows.

**Prerequisites:** Basic familiarity with software development processes (code reviews, PRs, deployments). You do not need to write code daily.

**Estimated time commitment:** 8–12 hours for the overview; 2–4 hours for the supervision checklists.

**Modules:**
1. `05-context-engineering.md` — understand how your team gives AI persistent project context
2. `07-secrets-and-security-basics.md` — know what questions to ask when reviewing AI-generated code for secret handling
3. `08-reading-code.md` — mental models for reading and reviewing code quickly
4. `14-testing-fundamentals.md` — understand what "enough testing" means for AI-assisted work
5. `23-reviewing-ai-output.md` — the human review discipline for every AI-assisted PR
6. `24-agentic-workflows.md` — understand what autonomous AI agents can and cannot be trusted to do unsupervised
7. `checklists/ai-code-review.md` — use this as a team standard for all AI-assisted code reviews

**You will be able to:**
- Review AI-assisted PRs with a structured checklist, even if you did not write the code yourself
- Ask the right questions about architecture, security, and correctness without auditing every line
- Set team norms for how AI context files (CLAUDE.md, AGENTS.md) should be maintained
- Identify when a developer is relying too heavily on AI output without sufficient understanding
- Understand the failure modes unique to AI-assisted development so you can catch them in review

***

## Success Metrics

| Metric | Target (6 months post-launch) |
|---|---|
| GitHub Stars | 500+ |
| Community contributions (PRs) | 20+ external contributors |
| Links/references from other repos | 10+ |
| Issues filed (signal of active use) | 50+ |
| Reddit/HN mentions | 3+ organic mentions |
| Modules marked outdated and updated by community | 5+ |

***

## Milestones

| Milestone | Deliverable | Target |
|---|---|---|
| **M0 — Scaffold** | Repo structure, README, contributing guide, module template, all checklist stubs | Week 1 |
| **M1 — Tier 1 Complete** | All 8 Tier 1 modules written + `before-you-start.md` checklist | Week 4 |
| **M2 — Tier 2 Complete** | All 7 Tier 2 modules written | Week 7 |
| **M3 — Tier 3 Complete** | All 6 Tier 3 modules + `before-you-ship.md` checklist | Week 10 |
| **M4 — Tier 4 Complete** | All 4 Tier 4 modules + `ai-code-review.md` checklist | Week 12 |
| **M5 — Launch** | GitHub release, README polished, learning paths added, shared to community (Reddit, X, HN) | Week 13 |
| **M6 — Iteration** | Community PRs reviewed, gaps filled based on issues, stale links refreshed | Ongoing |

***

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Content goes stale as tools evolve | High | Version each module; note "last verified" dates; accept community PRs; Tier 4 especially needs frequent review |
| Scope creep into full CS curriculum | Medium | Enforce the 10–15 min read rule per module; link out rather than explain everything |
| Low discoverability | Medium | Seed in r/vibecoding, submit to `awesome-vibe-coding` lists, cross-link from related repos |
| Duplication with existing guides | Low | Differentiation is the specific "before you vibe" framing and the AI-specific pitfall sections — maintain that angle clearly |
| Tier 4 becomes tool marketing | Medium | Keep Tier 4 tool-agnostic; focus on patterns and mental models, not specific product walkthroughs |

***

## Community & Contributing

We welcome contributors who want to strengthen the foundational knowledge available to vibe coders. This section explains how the project is maintained, how decisions are made, and what is expected of contributors at every level.

### Submitting Content (Pull Request Process)

All contributions go through a pull request:

1. **Fork and branch** from `main`. Use a descriptive branch name: `module/09-web-basics-revision`, `fix/git-stash-example`, `new/module-databases`.
2. **Open a draft PR early** if the change is large — this allows maintainers to give early feedback before you invest significant time.
3. **Fill out the PR template** completely. Incomplete PRs will be closed without review.
4. **Respond to review feedback** within 7 days. PRs with no activity for 14 days will be marked stale and may be closed.
5. **Review SLA:** A maintainer will give an initial review within 5 business days of the PR being marked ready.

### Module Maintainer Expectations

Modules are owned by their original authors, who are responsible for:

- **Keeping content current**: AI tooling evolves quickly. Maintainers should review their modules at least once per quarter and update "last verified" dates on external links.
- **Marking stale content**: If a module has known outdated information, add a header warning note and open a tracking issue.
- **Responding to related issues and PRs**: Maintainers should acknowledge within 5 business days and label appropriately.
- **Handing off ownership**: If you can no longer maintain a module, open an issue saying so. The project will find a new owner.

The repository owner (Trevor Poon) serves as fallback maintainer for orphaned modules and is the final decision-maker on disputes.

### Code of Conduct

By participating in this Repository, you agree to:

- **Be respectful**: Disagreements about technical choices are fine; personal attacks are not.
- **Assume good intent**: Contributors new to the project may not know the conventions yet. Redirect gently.
- **Keep discussions focused**: PR review threads should stay on-topic.
- **No AI-generated content as original contribution**: You may use AI tools to assist writing, but the PR description must disclose when AI generated text has been included verbatim.

### Proposing New Modules

New modules are welcome, but this project has a high bar for inclusion:

**The bar for a new module:**
- The topic must be a prerequisite that affects whether AI-generated code succeeds or fails — not a tool tutorial, not a language-specific deep dive.
- It must fit into one of the four existing tiers. If it does not, propose a new tier first.
- It must be completable in 10–15 minutes of active reading + exercise.
- It must include: `## Why it matters for vibe coding`, `## Hands-on exercise`, `## AI-specific pitfalls`.

**How to propose:**
1. Open an issue using the "Module Proposal" template.
2. Answer: What problem does this solve? Which tier does it belong in? What will the reader be able to do after reading it? What AI-specific pitfalls will it cover?
3. A maintainer will respond within 5 business days with provisional approval or rejection.

### How Success Is Measured

Beyond stars and download counts, the project measures success through qualitative signals:

- **Issue quality**: Are people filing specific, actionable issues — not just "this is broken" but "module X has an outdated link on line Y"?
- **PR depth**: Are community contributions going beyond typo fixes to include new examples, exercises, and AI-specific pitfall updates?
- **Signal-to-noise ratio**: Is the community using the issue tracker for discussion, or for actual content problems?
- **Adoption as prerequisite**: Has another repository, course, or community resource adopted this as a recommended prerequisite?
- **Qualitative feedback channels**: Readers can send feedback via GitHub Discussions. Maintainers will periodically summarize themes and publish them as a GitHub Discussion post.

These signals inform roadmap decisions, new module priorities, and whether existing modules need a complete rewrite.

***

## Open Questions

### 1. Should each module include a "test yourself" quiz?

**Recommendation: Yes — but lightweight, inline, and opt-out.**

Active recall improves retention, and a short quiz gives readers a signal that they have genuinely internalized a module before moving on. The risk of over-engineering is real if quizzes require manual grading or a separate tool.

**Suggested approach:** Pilot with Tier 1 modules only. Build a lightweight GH Actions workflow that reads quiz answers from a designated section at the bottom of each module markdown file. If the pilot shows positive reception, expand to Tiers 2–4. If feedback is negative, remove the feature.

### 2. Is there value in a language-specific track?

**Recommendation: No separate directories — instead, publish annotated learning paths.**

The project philosophy is tool-agnostic. Adding a Python-first directory doubles maintenance burden and undermines the goal of being concept-first.

**Suggested approach:** Keep modules language-agnostic, but add a `/paths/` directory with markdown files (e.g., `paths/python-first.md`, `paths/typescript-first.md`) that annotate the same modules with inline language examples. These are annotations, not replacements.

### 3. Should the repo eventually include starter project templates?

**Recommendation: Yes, but start small and community-sourced.**

Concrete starting points reduce friction and let readers immediately apply concepts from the modules in a real project.

**Suggested approach:** Launch with a `/templates` directory containing three minimal templates only: `python-uv-minimal/`, `typescript-nextjs-minimal/`, and `go-minimal/`. Templates must be PR-submitted by a community member with demonstrated expertise in that ecosystem — not authored by the repository owner.

### 4. Should Tier 4 include a module on evaluating LLM output (evals)?

**Recommendation: Yes — expand Module 23 rather than creating a new module.**

The existing Module 23 (`23-reviewing-ai-output.md`) already covers the review process, but does not address systematic evaluation of AI output over time — which is increasingly important.

**Suggested approach:** Rename `23-reviewing-ai-output.md` to `23-reviewing-and-evaluating-ai-output.md` and add a subsection covering: building a small golden dataset, running regression tests on prompt changes, detecting model regression before shipping, and when to use structured output tests.

### 5. Should `ai-code-review.md` be standalone or embedded in Module 23?

**Recommendation: Keep it standalone, but create clear cross-references.**

A checklist is a daily driver — something a developer pulls up during every PR review. The module is a learning resource that explains the principles behind the checklist. Conflating them reduces the utility of both.

**Suggested approach:** Keep `checklists/ai-code-review.md` standalone. At the top of `23-reviewing-ai-output.md`, add a prominent callout box linking to the checklist. In the checklist file, add a link back to the module for context.
