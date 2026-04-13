# PRD: `no-vibe-without-basics` — A Foundational Curriculum for AI-Assisted Development

**Version:** 1.0  
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

***

## Goals

1. Provide a structured, tiered curriculum covering the technical foundations every vibe coder needs
2. Be language- and tool-agnostic where possible; opinionated where it matters (e.g., recommend `uv` over `pip`)
3. Be completable at the reader's own pace — no login, no paywall, no course platform required
4. Serve both total beginners (Tier 1) and developers expanding into new domains (Tier 3)
5. Become the canonical "read this before you vibe code" reference linked by the community

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
| **The Experienced Dev Onboarding to AI** | Strong traditional skills, new to AI-assisted workflow | Context engineering, PRD writing, prompt discipline |

***

## Curriculum Structure

The curriculum is divided into three tiers, each building on the last. Each topic is a standalone Markdown file with: a plain-English explanation of the concept, why it matters for vibe coding specifically, a minimal working example or exercise, and links to deeper resources.

### Tier 1 — Essentials (Everyone)

These are non-negotiable prerequisites before writing a single AI prompt for code.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `01-git.md` | init, add, commit, push, branch, merge, PR, .gitignore | Checkpointing AI output; recovering from hallucinated rewrites |
| `02-virtual-envs.md` | uv, miniforge, conda, nvm, pyenv | Isolating AI-generated dependencies; reproducible environments |
| `03-terminal.md` | Navigation, file ops, SSH, pipes, tmux | Running AI-generated scripts; remote development |
| `04-project-structure.md` | Folder conventions, .env, config separation, README | Giving the AI coherent context about the project |
| `05-context-engineering.md` | CLAUDE.md, INITIAL.md, PRDs, PRP blueprints | Structured prompting; preventing hallucination and drift |

### Tier 2 — Builder Foundations

For anyone shipping something real beyond a personal script.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `06-web-basics.md` | HTML/CSS/JS mental model, HTTP lifecycle, REST, status codes | Understanding AI-generated web code; debugging API calls |
| `07-frontend-backend.md` | Client/server boundary, rendering strategies (SSR/CSR/SSG), API contracts | Knowing what to ask the AI to build where |
| `08-databases.md` | SQL vs. NoSQL, schemas, migrations, ORMs, indexing, .env secrets | Reviewing AI-generated schemas; preventing data loss |
| `09-programming-languages.md` | Python, TypeScript, Go, Rust — trade-offs and use cases | Picking the right language before prompting; reading AI output critically |

### Tier 3 — Advanced Concepts

For scaling beyond a side project or working on production systems.

| Module | Key Concepts | Vibe Coding Relevance |
|---|---|---|
| `10-infrastructure.md` | Cloud providers, Docker, containers, secrets management, environment parity | Deploying AI-built apps; environment debugging |
| `11-architecture.md` | Monolith vs. microservices, C4 diagrams, API design, auth patterns (JWT, OAuth) | Designing systems AI can implement coherently |
| `12-deployment.md` | CI/CD basics, reverse proxies (nginx/Caddy), domains, HTTPS, monitoring | Getting AI-built apps to production and keeping them running |

***

## Repo Structure

```
no-vibe-without-basics/
├── README.md                    ← Roadmap, philosophy, how to use this repo
├── CONTRIBUTING.md              ← Guide for contributors
├── tier-1-essentials/
│   ├── 01-git.md
│   ├── 02-virtual-envs.md
│   ├── 03-terminal.md
│   ├── 04-project-structure.md
│   └── 05-context-engineering.md
├── tier-2-builder/
│   ├── 06-web-basics.md
│   ├── 07-frontend-backend.md
│   ├── 08-databases.md
│   └── 09-programming-languages.md
├── tier-3-advanced/
│   ├── 10-infrastructure.md
│   ├── 11-architecture.md
│   └── 12-deployment.md
└── checklists/
    ├── before-you-start.md      ← Tier 1 checklist before first AI prompt
    └── before-you-ship.md       ← Tier 2–3 checklist before deploying
```

***

## Module Template

Every module follows this structure to ensure consistency:

```markdown
# [Topic Name]

## What is it?
Plain-English explanation. No assumed knowledge.

## Why it matters for vibe coding
Specific failure modes that occur when this knowledge is missing.

## The 20% you need to know
Minimal viable understanding — the concepts that cover 80% of real situations.

## Hands-on exercise
A concrete, runnable task (not just reading).

## Common mistakes
The top 3 errors beginners make, and how to avoid them.

## Go deeper
3–5 curated external links (documentation, tutorials, videos).
```

***

## Content Guidelines

- **Opinionated where it helps**: recommend `uv` over `pip`, recommend `git commit` messages in imperative mood, recommend Postgres over SQLite for anything production-facing
- **Tool-agnostic where it doesn't matter**: don't prescribe a specific cloud provider, OS, or editor
- **Always answer "why does this matter for AI coding?"** — every module must connect back to the vibe coding context, not be generic programming education
- **Minimal, not exhaustive**: each module should be readable in 10–15 minutes. Link out for depth
- **Practical exercises, not just reading**: every module ends with something the reader *does*, not just reads

***

## Success Metrics

| Metric | Target (6 months post-launch) |
|---|---|
| GitHub Stars | 500+ |
| Community contributions (PRs) | 20+ external contributors |
| Links/references from other repos | 10+ |
| Issues filed (signal of active use) | 50+ |
| Reddit/HN mentions | 3+ organic mentions |

***

## Milestones

| Milestone | Deliverable | Target |
|---|---|---|
| **M0 — Scaffold** | Repo structure, README, contributing guide, module template | Week 1 |
| **M1 — Tier 1 Complete** | All 5 Tier 1 modules written + `before-you-start.md` checklist | Week 3 |
| **M2 — Tier 2 Complete** | All 4 Tier 2 modules written | Week 5 |
| **M3 — Tier 3 Complete** | All 3 Tier 3 modules + `before-you-ship.md` checklist | Week 7 |
| **M4 — Launch** | GitHub release, README polished, shared to community (Reddit, X, HN) | Week 8 |
| **M5 — Iteration** | Community PRs reviewed, gaps filled based on issues | Ongoing |

***

## Risks & Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Content goes stale as tools evolve | High | Version each module; note "last verified" dates; accept community PRs |
| Scope creep into full CS curriculum | Medium | Enforce the 10–15 min read rule per module; link out rather than explain everything |
| Low discoverability | Medium | Seed in r/vibecoding, submit to `awesome-vibe-coding` lists, cross-link from related repos |
| Duplication with existing guides | Low | Differentiation is the specific "before you vibe" framing and the tier structure — maintain that angle clearly in README |

***

## Open Questions

1. Should Tier 1 include a module on **how to write effective prompts** (prompt engineering basics), or is that out of scope for a "pre-vibe" curriculum?
2. Should each module include a **"test yourself" quiz** or is that over-engineering the repo?
3. Is there value in a **language-specific track** (e.g., a Python-first path vs. a TypeScript-first path) or should all modules remain language-agnostic?
4. Should the repo eventually include **starter project templates** (e.g., a minimal Python project with correct folder structure, `.env`, virtual env setup) as companion resources?
