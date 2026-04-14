# Context Engineering

## What is it?

Context engineering is the discipline of giving an AI persistent, structured knowledge about your project so it can work accurately across sessions — not just within a single conversation. It is the difference between an AI that behaves like a confused new hire reading scattered sticky notes, and one that behaves like a senior engineer who has been on the project for months.

The primary artifact of context engineering is a `CLAUDE.md` file — a project-specific instruction document that lives in your repository root. Similar files exist for different AI tooling contexts: `AGENTS.md` for Anthropic's agent protocol, `INITIAL.md` for Claude Code's startup sequence, and a `PRD.md` that captures *why* the project exists and what it is trying to accomplish.

Think of it this way: when you hire a new engineer, you don't just throw them into the codebase and hope they figure it out. You onboard them — you explain the project structure, coding conventions, architectural decisions, and what "good" looks like. `CLAUDE.md` is that onboarding document for an AI.

## Why it matters for vibe coding

Without structured context, vibe coding fails in specific, predictable ways:

**Context loss after a few turns.** AI models have a finite context window — they can only "remember" what fits in the conversation. After roughly 5–10 exchanges (depending on project size and message length), the AI starts forgetting earlier instructions. If your only instructions are in a chat thread, they vanish mid-project. A `CLAUDE.md` file is re-read on every new conversation, resetting the AI's understanding instantly.

**AI generates code that violates project conventions.** Without documented conventions, every new file the AI creates might use different import ordering, naming patterns, or folder placements. Over weeks of vibe coding, you end up with a codebase that feels like five different developers wrote it. `CLAUDE.md` tells the AI what the conventions *are*, so it follows them from the first line.

**The AI doesn't know the project's purpose.** An AI that doesn't know what your app does will make decisions that are technically correct but strategically wrong. A payments app needs idempotency keys on every transaction. A data pipeline needs retry logic on ingestion failures. Without a PRD and a clear project purpose in `CLAUDE.md`, the AI cannot make those contextual decisions — it will generate code that looks fine but breaks in production.

**Drift: AI slowly misaligns with the project over time.** Starting a project with a clear `CLAUDE.md` does not guarantee alignment after weeks of edits. As the project evolves, the `CLAUDE.md` must be updated too. If you don't update it, the AI's understanding gradually diverges from reality — it starts suggesting code based on outdated information, which is harder to catch than a completely wrong answer.

**AI doesn't know what not to do.** Constraints are as important as instructions. A `CLAUDE.md` that says "use PostgreSQL" but doesn't say "never use SQLite in production" leaves a gap the AI will happily fill with the wrong tool. Explicit constraints prevent AI-tooling-generated footguns.

## The 20% you need to know

### CLAUDE.md: The core artifact

`CLAUDE.md` is the file Anthropic's AI tools read automatically at the start of every new conversation. It is plain markdown, placed in the repository root. It is not a system prompt — it is project context that gets prepended to your conversation automatically.

A minimal `CLAUDE.md` answers four questions:

1. **What is this project?** A one-sentence description of what the application does.
2. **What tech stack does it use?** Language, framework, key libraries, and version constraints.
3. **What is the folder structure and why?** The layout and the reasoning behind it.
4. **What are the non-negotiable conventions?** Naming conventions, architectural patterns, forbidden tools.

A functional `CLAUDE.md` for a Python FastAPI project looks like this:

```markdown
# Project Overview

This is a REST API backend for a task management application. It exposes endpoints for creating, reading, updating, and deleting tasks. Authentication is via Bearer tokens.

## Tech Stack

- Python 3.12+ with FastAPI
- PostgreSQL 15+ via SQLAlchemy 2.0 with async support
- Pydantic v2 for request/response schemas
- Alembic for database migrations
- pytest for testing

## Project Structure

```
src/
├── main.py           # FastAPI app entry point
├── routes/           # API route handlers (one file per resource)
├── models/           # SQLAlchemy ORM models
├── schemas/          # Pydantic request/response models
├── core/             # Config, security, dependencies
└── db.py             # Database connection and session management
tests/
├── conftest.py       # Shared pytest fixtures
└── test_routes/      # Route-level integration tests
```

## Conventions

- Import ordering: stdlib, third-party, local — never mix them
- Route handlers go in `src/routes/`, not elsewhere
- All database access goes through SQLAlchemy models — no raw SQL in route handlers
- Environment variables are loaded from `.env` at startup via `pydantic-settings`
- Never commit `.env` to version control

## Constraints

- Do not use SQLite in production
- Do not use `print()` for logging — use the structured logger
- Do not add new dependencies without a lockfile update
```

This is the minimum viable `CLAUDE.md`. It fits on one screen. An AI reading it for the first time can generate correct code in under a minute.

### The difference between CLAUDE.md, AGENTS.md, and INITIAL.md

These files serve different tooling contexts and are not interchangeable:

**`CLAUDE.md`** is read by Anthropic's consumer-facing tools (Claude.ai, Claude Code) at the start of every conversation. It is the broadest-purpose context file and the one you should create first.

**`AGENTS.md`** is part of Anthropic's agent protocol (MCP-compatible). It defines how autonomous agents should behave when working on your project — specifically, what tasks they can perform autonomously versus what requires human approval. If you are using Claude Code in agent mode or running multi-agent workflows, `AGENTS.md` sets the rules. If you are doing single-user vibe coding with a consumer tool, you likely do not need `AGENTS.md`.

**`INITIAL.md`** is specific to Claude Code's startup behavior. When Claude Code starts in a project, it reads `INITIAL.md` (if it exists) before doing anything else — before reading `CLAUDE.md`, before loading the project. This makes it useful for one-time setup actions: running a migration, seeding a database, printing a welcome message with current project status. It is not persistent context — it runs once at startup and is done.

Practical rule: **start with `CLAUDE.md`**. Add `AGENTS.md` only when you are running autonomous agents. Use `INITIAL.md` only if you need something to happen exactly once at the start of every session.

### The PRD: Your project's birth certificate

A Product Requirements Document (`PRD.md`) explains *why* the project exists — the problem it solves, the users it serves, and the success criteria. Unlike `CLAUDE.md`, which focuses on *how* to work in the codebase, the PRD focuses on *what* the project is trying to accomplish and *why* each decision was made.

A PRD matters for vibe coding because:

- It prevents AI-generated code that is technically correct but strategically wrong. A feature that makes sense for a B2C app is wrong for an enterprise B2B tool.
- It gives the AI judgment to resolve ambiguity. When two equally valid implementation paths exist, the PRD's stated goals disambiguate.
- It survives context window rotations. Even if the AI forgets earlier conversation, the PRD in the repository is always there.

A PRD does not need to be long. Three sections suffice:

1. **What does this app do?** One paragraph.
2. **Who uses it and what do they need?** The user story.
3. **What does success look like?** Measurable criteria (e.g., "API response time under 200ms at p99").

### Memory files: Long-term context across sessions

Beyond `CLAUDE.md`, some workflows benefit from dedicated memory files that persist project-specific knowledge. These are not read automatically — you reference them in your prompts or in `CLAUDE.md`.

Common patterns:

- `DECISIONS.md` — a log of architectural decisions and the reasoning behind them. When the AI asks "should I use a queue or a webhook here?", the answer is in `DECISIONS.md`.
- `TASKS.md` — a running list of in-progress work, blockers, and pending decisions. Keeps the AI from re-debating settled questions.
- `STACK.md` — when `CLAUDE.md` grows too long, extracting the tech stack details into a separate file keeps it readable.

Reference these files from `CLAUDE.md`:
```markdown
## Additional Context

- Architectural decisions: see `DECISIONS.md`
- Current task status: see `TASKS.md`
```

### Prompt caching: What it is and when it matters

Prompt caching is a technique where you explicitly structure your prompts so that the AI can reuse previously processed context across multiple calls. In practice for vibe coding, this means:

- **Chunk your instructions.** Rather than dumping all context into one long message, separate `CLAUDE.md` (static, read once), your current request (specific task), and reference material (logs, code snippets) into distinct sections. This lets the caching layer reuse the static portions.
- **Put static context first.** Most AI tooling reads from top to bottom. Putting your most stable context (project overview, conventions) at the top means it gets cached first.
- **Be aware of staleness.** Cached context becomes stale when your project changes. If you update `CLAUDE.md`, the AI using cached context from the previous session may not pick up the changes immediately. Always verify the AI is working from current context.

## Hands-on exercise

**Create a CLAUDE.md for this project in 10 minutes.**

If you are in an existing project that does not have a `CLAUDE.md`, create one now. If you are starting a new project, scaffold the file before writing any code.

### Steps

1. **Navigate to your project root.** Confirm you are in the right directory with `ls` or `dir`.

2. **Create `CLAUDE.md` in the root.** Open it in your editor.

3. **Write the four required sections:**
   - Project name and one-sentence purpose
   - Tech stack (language, framework, key libraries)
   - Project structure with folder descriptions
   - Conventions and constraints (at least 3 each)

4. **Commit it to version control.** This is not optional — the file must be in Git so the AI can always retrieve it.

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md with project context"
```

5. **Verify the AI can read it.** In a new conversation with your AI tool, say: "What is the tech stack for this project according to CLAUDE.md?" If it cannot answer, the file is not in the right location or is not committed.

### Expected output

A `CLAUDE.md` file in your project root, committed to Git, that an AI can accurately summarize when asked. You should be able to ask three questions and get correct answers:

1. "What language and framework does this project use?"
2. "Where do API route handlers live?"
3. "What is an explicit constraint this project has?"

If the AI cannot answer all three correctly, revise the `CLAUDE.md` until it can.

## Common mistakes

**Mistake 1: CLAUDE.md is too long to read.**

What happens: You dump everything you know about the project into `CLAUDE.md` — every architectural decision, every historical context, every edge case. The file becomes so long that the AI does not read it fully (or runs out of context budget before getting to the actual task).

Why it happens: The instinct to be thorough wins over the instinct to be useful. Engineers who know a lot about their project over-communicate.

How to fix it: Keep `CLAUDE.md` under 200 lines. Push history and detailed decisions into `DECISIONS.md` and reference it. If the file cannot be scanned in 30 seconds, it is too long.

**Mistake 2: CLAUDE.md is written for humans, not AIs.**

What happens: The file uses natural language prose without structural signals. "We generally follow good practices when building things" is meaningless to an AI. The AI cannot infer conventions from vague statements.

Why it happens: Developers write `CLAUDE.md` the way they write documentation — narrative, not instruction.

How to fix it: Use bullet points, explicit rules, and concrete examples. "Import ordering: stdlib, third-party, local" is a rule. "We care about code quality" is not. Use the imperative voice: "Do not use SQLite in production" rather than "SQLite is not preferred."

**Mistake 3: CLAUDE.md is created but never committed.**

What happens: The file exists locally but is in `.gitignore` or otherwise untracked. The AI tool (running in a fresh session or on a different machine) cannot find it. The developer loses the context they created.

Why it happens: `CLAUDE.md` gets treated as a personal preference file rather than a project artifact.

How to fix it: Commit `CLAUDE.md` to the repository. It is documentation, not secrets. Add it to `.gitignore` never — it should be in version control so every collaborator and every AI session has access.

**Mistake 4: CLAUDE.md and the actual project diverge.**

What happens: The project evolves — a new service is added, a library is replaced, the folder structure changes — but `CLAUDE.md` stays static. The AI reads the file, generates code based on outdated information, and produces something wrong.

Why it happens: There is no habit of updating `CLAUDE.md` alongside code changes. The file is treated as a one-time setup rather than a living document.

How to fix it: Make updating `CLAUDE.md` part of your commit checklist. When you add a new service, update the project structure. When you change a convention, update the conventions section. Review it monthly as part of project maintenance.

**Mistake 5: Not using a PRD at all.**

What happens: The project has a `CLAUDE.md` but no `PRD.md`. The AI knows *how* to work in the codebase but not *what the project is trying to accomplish*. This leads to technically correct code that solves the wrong problem.

Why it happens: PRDs feel like enterprise bureaucracy. Developers skip them because the project "just needs to get built."

How to fix it: Write a minimal PRD — three sections, under 300 words. It lives in `docs/PRD.md` alongside `CLAUDE.md`. If the project is small enough that you cannot write three paragraphs about what it does, the project may not be worth building yet.

## AI-specific pitfalls

**Pitfall 1: AI does not read CLAUDE.md on its own.**

Most AI coding tools do read `CLAUDE.md` at session start, but not always. Claude Code reads it by default. Other tools may require you to explicitly reference it in your prompt. If you are using a new tool, test whether it reads the file: ask it a specific question that can only be answered from `CLAUDE.md`. If it cannot answer, the tool is not reading the file — you must paste its contents manually into the conversation or find the tool's context injection mechanism.

What to look for: After creating `CLAUDE.md`, ask the AI "What constraints does this project have?" If the answer is wrong or empty, the AI is not reading the file.

**Pitfall 2: AI loses context after a few turns and starts violating conventions.**

This is the most common failure mode in long vibe coding sessions. The AI starts fine (reading `CLAUDE.md`), but after 10–15 exchanges, it begins generating code that violates naming conventions, puts files in the wrong folders, or uses the wrong import ordering. This happens because the conversation grows and the `CLAUDE.md` content gets pushed out of the context window.

What to look for: In long sessions, re-inject context explicitly. Paste a brief summary of key conventions into your message: "Reminder: imports go stdlib, third-party, local. Routes go in src/routes/. Do not use raw SQL." This is a band-aid, not a fix — the real fix is keeping `CLAUDE.md` updated so the AI can re-read it on context window rotation.

**Pitfall 3: AI generates code for the wrong language or framework because it misremembered the stack.**

The AI may confidently suggest Go code for a Python project, or use SQLAlchemy syntax for an app that uses Prisma. This happens when the AI's training data has strong patterns for certain code structures and it applies them without verifying the project stack.

What to look for: In every code generation, verify the first line: does it import from a library that actually exists in `CLAUDE.md`'s tech stack? If the AI suggests `from sqlalchemy import Column` in a project that uses Prisma, that is a signal the AI has lost context of the actual stack.

**Pitfall 4: AI invents configuration options or environment variables that do not exist.**

A well-documented `CLAUDE.md` helps, but the AI may still hallucinate environment variable names, CLI flags, or configuration keys. It generates code using `DATABASE_URL` when the project actually uses `POSTGRES_URL`. The code looks plausible and compiles, but fails at runtime.

What to look for: Cross-reference any environment variable name in AI-generated code against your `.env.example` or actual `.env` file. If the variable does not exist there, the AI invented it.

**Pitfall 5: AI does not update its understanding when CLAUDE.md changes.**

If you update `CLAUDE.md` to reflect a new architectural decision, the AI working from a cached context window (from an earlier session) may not pick up the change. This is especially dangerous when the change is a constraint removal — the AI still thinks something is forbidden when it is now allowed, or vice versa.

What to look for: After updating `CLAUDE.md`, explicitly tell the AI: "I updated CLAUDE.md. Please confirm your understanding of [specific change]." This forces the AI to re-process the file rather than relying on stale cached context.

## Quick reference

### File comparison

| File | Purpose | When to use | Auto-read? |
|---|---|---|---|
| `CLAUDE.md` | Project context, conventions, stack | Every new conversation | Yes (Anthropic tools) |
| `AGENTS.md` | Autonomous agent behavior rules | Multi-agent workflows, agent mode | Yes (agent protocol) |
| `INITIAL.md` | One-time startup actions | Seeding DB, printing welcome message | Yes (Claude Code only, once) |
| `PRD.md` | Project purpose and goals | Every conversation, referenced in CLAUDE.md | No — reference explicitly |
| `DECISIONS.md` | Architectural decision log | When AI needs context on past decisions | No — reference explicitly |

### CLAUDE.md checklist

- [ ] Project name and one-sentence purpose
- [ ] Tech stack (language, framework, key libraries)
- [ ] Project structure with folder descriptions
- [ ] At least 3 conventions
- [ ] At least 2 explicit constraints (what NOT to do)
- [ ] Committed to Git (not in `.gitignore`)
- [ ] Under 200 lines

### PRD minimum

- [ ] What does this app do? (one paragraph)
- [ ] Who uses it and what do they need? (user story)
- [ ] What does success look like? (measurable criteria)

### When to re-read CLAUDE.md

- At the start of every new conversation
- After any architectural change to the project
- When AI output starts violating conventions (re-inject context)
- Monthly, even if nothing has changed (verify it is still accurate)

## Go deeper

- [Anthropic: CLAUDE.md documentation](https://docs.anthropic.com/en/docs/claude-code) (verified 2026-04)
- [Anthropic: Agent protocol and AGENTS.md](https://docs.anthropic.com/en/docs/mcp/agents) (verified 2026-04)
- [GitHub: Writing great CLAUDE.md (community guide)](https://github.com/anthropics/claude-code/tree/main/docs) (verified 2026-04)
- [Structured prompting techniques — Anthropic cookbook](https://docs.anthropic.com/en/docs/build-a-claude-app/video/structured-prompts) (verified 2026-04)
- [Prompt engineering guide — context window management](https://www.promptingguide.ai/techniques/context) (verified 2026-04)