# Project Structure

## What is it?

Project structure is the organized layout of files and folders in a codebase — what goes where, why it's there, and what role each piece plays. It's the difference between a project you can navigate and one that feels like a black box.

Think of it like a well-organized kitchen versus a chaotic one. In a well-organized kitchen, you know exactly where the cutting board lives, where the knives go, and where to find a spatula without thinking about it. A codebase with good structure works the same way: when you need to add a test, you know it goes in a `tests/` folder. When you need to configure the database connection, you know to look in a config file, not buried in application logic.

Structure includes:
- **Folder layout** — how directories are organized and named
- **File naming conventions** — consistent patterns for `MyComponent.tsx` vs `my_component.py`
- **Config separation** — keeping environment-specific values separate from code
- **Entry points** — where the application starts, where tests live, where documentation lives

## Why it matters for vibe coding

Here is the concrete failure mode: you ask an AI to add a login feature to your project. The AI has no concept of your project structure. It generates `login.py` in the root directory when it should go in `src/features/auth/`. It puts the database password directly in the code instead of loading it from an environment variable. It creates a `README.txt` in the root while you have a `README.md` — now you have two readmes and the AI gets confused about which one to update.

Without a consistent project structure, every AI session starts from scratch. The AI has no guidance on where files should live, so it makes its best guess — and its best guess often doesn't match what you would have chosen. Over time, you get a project that looks like it was assembled by a committee where no one talked to each other.

The specific failure patterns you will encounter:

- **AI generates files in wrong directories** — you ask for a new utility function and the AI creates `utils.py` in the root instead of `src/utils/` because it doesn't know your convention
- **AI hardcodes secrets in source code** — it sees a database URL and writes it directly into the connection logic instead of reading it from `process.env.DB_URL` or a `.env` file
- **AI creates duplicate files** — `auth.py` and `authentication.py` both exist because the AI doesn't know which one you meant to use
- **AI generates inconsistent naming** — `UserService.ts`, `user_auth.py`, and `userAuthentication.js` in the same project because it doesn't know your naming convention
- **AI ignores existing structure** — creates a new config file in the project root instead of using the existing `config/` directory

Good structure gives the AI a map. Every time you point it at your project, it can see the layout and follow your conventions.

## The 20% you need to know

### src/ layout versus flat layout

The most important structural decision in any codebase is whether to use a **src/ layout** or a **flat layout**.

**Flat layout** puts everything in the project root:

```
myproject/
  app.py
  models.py
  utils.py
  config.py
  tests.py
```

This works for small scripts and experiments. It does not scale.

**src/ layout** moves application code into a `src/` directory:

```
myproject/
  src/
    app.py
    models/
    utils/
  tests/
  config/
  README.md
  .env.example
```

The `src/` directory signals "everything in here is application code." Anything at the root level is project infrastructure: configuration, documentation, tooling, tests. This separation makes it obvious what belongs where.

**Why `src/` specifically?** It is a widely-adopted convention across Python (Python packaging standards), JavaScript/TypeScript (most frameworks default to `src/`), Go, and Rust. It predates AI tools but it is exactly the kind of convention that AI tools recognize and respect. When you say "add a new endpoint in `src/api/`" the AI immediately knows the context.

Recommendation: use `src/` layout for all projects, even small ones. The habit transfers, and it makes every project immediately recognizable to anyone who has worked with a `src/` project before.

### Config files and where they belong

Configuration has one job: store values that change between environments (development, staging, production) so you do not have to change code to move between them.

The standard locations:

**`.env` file** — environment variables for local development. This file should **never** be committed to version control. It is personal to your machine.

**`.env.example`** — a template showing what variables the project needs. This file **should** be committed. It tells anyone joining the project what environment variables exist without exposing your actual values.

**`config/` directory** — for static configuration that does not change per environment (feature flags, third-party service names, non-sensitive defaults). Some projects use `config/` for environment-specific files like `config/development.py` and `config/production.py`. This is fine, but keep sensitive values out and rely on environment variables for those.

The critical rule: **secrets do not go in code, they do not go in config files that get committed, they go in environment variables.** API keys, database passwords, JWT secrets, encryption keys — all of these live in `.env` and get loaded into `process.env` or `os.environ` at runtime.

### README structure

A README has one job: get a new person from zero to running code in under five minutes. Everything beyond that is nice-to-have.

A minimal README that works:

```markdown
# Project name

One sentence describing what this does.

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env
# edit .env with your values
```

## Run

```bash
python src/main.py
```

## Structure

Explain the folder layout in two or three sentences.
```

That is it. A README longer than one page is a sign that either the setup process is too complex (a problem to fix) or the README contains history and philosophy that belongs in documentation, not a getting-started guide.

### Where tests go

Tests live close to what they test, or in a dedicated `tests/` directory at the project root — not both.

**Closest-to-source layout** (common in Python):

```
src/
  utils.py
  utils_test.py    ← test lives next to the code it tests
```

**Dedicated tests directory** (common in TypeScript/JavaScript, Go, Rust):

```
src/
  utils.py
tests/
  test_utils.py   ← tests in their own top-level directory
```

Both patterns work. Pick one and be consistent. The dedicated `tests/` directory scales better for larger projects because tests are never accidentally excluded from test runs based on glob patterns.

**The rule that matters:** tests should never be in the same directory as production code in a way that causes them to be deployed. In Python, `__pycache__` and `.pyc` files from tests should not be shipped. In JavaScript, test files should be excluded from the bundle. If your bundler or runtime is accidentally shipping test code to production, your structure needs fixing.

### Where documentation goes

- **`README.md`** — project root, front-facing, setup and overview only
- **`docs/`** — detailed documentation for architecture, workflows, and decisions (ADR — Architecture Decision Records)
- **Inline comments** — only for non-obvious business logic, not for explaining what obvious code does
- **`CHANGELOG.md`** — version-by-version history of what changed (can be auto-generated from commit messages)

Do not create a `documentation/` directory. `docs/` is the convention. Do not create a `wiki/` unless your team actively uses the GitHub wiki — they rarely do.

### Monorepo versus polyrepo

A **polyrepo** is one repository per project. You have `auth-service`, `frontend-app`, `payment-api` as three separate repos.

A **monorepo** is one repository containing multiple projects that may have previously been separate repos. You have one `mycompany/` repo containing `services/auth/`, `apps/frontend/`, and `apis/payment/`.

**When to use polyrepo:** different teams own different services, deployment cycles are completely independent, you want fine-grained access control per repo.

**When to use monorepo:** the projects are tightly coupled (a change in `auth-service` always requires changes in `frontend-app`), you want shared tooling and consistent dependency versions across all projects, your team is small-to-medium and can manage the larger repo.

**The practical reality for vibe coders:** most projects you build will start as polyrepos — one repo per application. A monorepo is a scaling strategy, not a starting point. Start with one repo, one project, one `src/` directory. If you later need to split into multiple services that happen to live in the same repo, that is when you adopt monorepo tooling (Nx, Turborepo, Bazel, Lerna).

The failure mode is over-engineering at the start: setting up a monorepo for a project that will never need it, creating elaborate shared package structures before there is anything shared. AI tools will sometimes suggest this because it is a "best practice" they have been trained on — you do not need it until you have the problem it solves.

## Hands-on exercise

**Time: 10 minutes**

You will create a proper `src/` layout from scratch and verify it with a quick Python project, then generate an `.env.example` file.

### Step 1: Create the layout

In a new directory, run:

```bash
mkdir -p src tests config docs
touch src/.gitkeep tests/.gitkeep config/.gitkeep
touch .env.example .gitignore
echo "src/" >> .gitignore
echo "tests/" >> .gitignore
echo ".env" >> .gitignore
echo "config/" >> .gitignore
echo "__pycache__/" >> .gitignore
```

### Step 2: Create the .env.example

Add the following to your `.env.example`:

```bash
# Database
DATABASE_URL=postgres://user:password@localhost:5432/mydb

# Authentication
JWT_SECRET=your-secret-key-here

# External API
EXTERNAL_API_KEY=your-api-key-here
```

### Step 3: Create a minimal runnable file

Create `src/main.py`:

```python
import os

def main():
    # Load environment variables from .env (requires python-dotenv in real projects)
    db_url = os.environ.get("DATABASE_URL", "not configured")
    jwt_secret = os.environ.get("JWT_SECRET", "not configured")
    api_key = os.environ.get("EXTERNAL_API_KEY", "not configured")

    print("Loaded environment variables:")
    print(f"  DATABASE_URL: {db_url}")
    print(f"  JWT_SECRET: {jwt_secret[:4]}..." if jwt_secret != "not configured" and jwt_secret != "your-secret-key-here" else f"  JWT_SECRET: {jwt_secret}")
    print(f"  EXTERNAL_API_KEY: {api_key[:4]}..." if api_key != "not configured" and api_key != "your-api-key-here" else f"  EXTERNAL_API_KEY: {api_key}")

if __name__ == "__main__":
    main()
```

Run it:

```bash
python src/main.py
```

**Expected output:**
```
Loaded environment variables:
  DATABASE_URL: not configured
  JWT_SECRET: not configured
  EXTERNAL_API_KEY: not configured
```

### Step 4: Test with real .env values

Create a `.env` file (remember, this is in `.gitignore` and never gets committed):

```bash
DATABASE_URL=postgres://devuser:devpass@localhost:5432/devdb
JWT_SECRET=super-secret-dev-key-12345
EXTERNAL_API_KEY=ak_dev_abcdef123456
```

Run again:

```bash
python src/main.py
```

**Expected output:**
```
Loaded environment variables:
  DATABASE_URL: postgres://devuser:devpass@localhost:5432/devdb
  JWT_SECRET: supe...
  EXTERNAL_API_KEY: ak_d...
```

The key insight: the code never changes between "not configured" and "loaded with real values." The environment variable loading mechanism is the same. What changed was the environment.

### Verification checklist

- `.env` is not committed (verify with `cat .gitignore`)
- `.env.example` is committed and contains no real secrets
- `src/` contains all application code
- `tests/` contains all test code
- You can run `python src/main.py` from the project root without errors

## Common mistakes

### Mistake 1: Committing .env to version control

**What happens:** Your database password, API keys, and secrets are in a public GitHub repository. Someone finds them, uses them, and now you are dealing with a breach.

**Why it happens:** The `.env` file was created in a rush, the developer forgot to add it to `.gitignore`, or they committed it before the ignore rule was in place.

**How to fix it:** If you have already committed `.env`, treat it as compromised — rotate all secrets immediately. Then:

```bash
# Remove from git history (this does not erase the file from history, but stops tracking)
git rm --cached .env
echo ".env" >> .gitignore
git add .gitignore
git commit -m "chore: stop tracking .env file"
```

For existing commits, use `git filter-repo` or BFG Repo-Cleaner to expunge the file from history.

### Mistake 2: Mixing config with code

**What happens:** You hardcode a feature flag value directly in `app.py`. Later, you need to change it for a production deployment but cannot do so without editing source code and redeploying.

**Why it happens:** It is faster to write `feature_flags = {"new_login": True}` than to extract it into a config file. In small projects, this seems fine.

**How to fix it:** At minimum, use environment variables for anything that differs between development and production. In Python: `os.environ.get("FEATURE_NEW_LOGIN", "false")`. In JavaScript/TypeScript: `process.env.FEATURE_NEW_LOGIN ?? "false"`.

### Mistake 3: No entry point, or too many entry points

**What happens:** A new developer opens the project and does not know where the application starts. Is it `app.py`, `main.py`, `index.js`, `server.js`, `run.py`? There are four of them.

**Why it happens:** Multiple scripts get created over time for different purposes, and none are clearly designated as the main entry point.

**How to fix it:** Have exactly one entry point per project. Name it consistently: `main.py` (Python), `index.js` (Node), `main.go` (Go), `src/main.rs` (Rust). If you need scripts for different purposes, put them in a `scripts/` directory and document them in the README.

### Mistake 4: Inconsistent folder naming

**What happens:** Some directories use camelCase (`UserAuth`), some use snake_case (`user_auth`), some use kebab-case (`user-auth`). Finding anything requires guessing.

**Why it happens:** No convention was established at the start, or the convention was not enforced when the first violations occurred.

**How to fix it:** Pick a convention at the start of the project. For Python: snake_case for files and directories. For TypeScript/JavaScript: camelCase for variables and files, PascalCase for components. Document it in `CONTRIBUTING.md` or `CLAUDE.md`. The AI needs to know this too — put it in your project context file.

### Mistake 5: Flat layout for growing projects

**What happens:** A project starts with five files in the root. Six months later, it has forty files. Nobody can find anything.

**Why it happens:** The project was small at the start and nobody migrated the structure as it grew.

**How to fix it:** Migrate to `src/` layout before it becomes painful. Move all application code into `src/`. Move tests to `tests/`. Add `src/` to `.gitignore` patterns if needed. It takes an hour now and saves days later.

## AI-specific pitfalls

### Pitfall 1: AI generates files in the wrong directory

**What it looks like:** You ask the AI to create a utility function for formatting dates. It creates `date_formatter.py` in the project root. You wanted it in `src/utils/`.

**Why it happens:** The AI does not know your folder convention unless you have told it. When given a task like "add a date formatter," it creates a file with a reasonable name in a reasonable location — but it has no information about your project layout beyond what you have explicitly shared.

**What to do:** Put your project structure in `CLAUDE.md` or `AGENTS.md`. Write: "All application code lives in `src/`. Utilities go in `src/utils/`." The AI will read this and follow the convention.

**What to look for when reviewing:** Every new file the AI proposes — check its path against your expected structure before approving it.

### Pitfall 2: AI hardcodes secrets instead of using .env

**What it looks like:** The AI generates a database connection:

```python
conn = psycopg2.connect(
    host="localhost",
    database="myapp",
    user="admin",
    password="super-secret-password"
)
```

**Why it happens:** The AI sees a connection pattern and fills in plausible values. It does not know these values should come from environment variables unless you have told it that pattern.

**What to do:** Add to your project context: "All secrets and connection strings come from environment variables. No hardcoded credentials."

**What to look for when reviewing:** Any string that looks like a password, API key, secret token, or connection URL — and check if it is being loaded from `os.environ` or `process.env`. If it is a string literal, it is wrong.

### Pitfall 3: AI does not generate or update a README

**What it looks like:** The AI builds a complete application in `src/` but the project has no README. Or it updates code without updating the README to reflect the changes.

**Why it happens:** The AI's training data includes many projects without READMEs. It does not naturally prioritize documentation as part of the deliverable.

**What to do:** After any significant AI-assisted implementation, ask specifically: "Update the README to reflect the current project structure and setup instructions."

**What to look for when reviewing:** Check for README existence and accuracy whenever you accept a significant amount of AI-generated code. Verify that the setup steps still work by following the README from a clean state.

### Pitfall 4: AI creates duplicate files with different names

**What it looks like:** You ask for a user service. The AI creates `user_service.py` and later when you ask for authentication logic, it creates `auth.py` — and both contain overlapping code for user authentication. You now have two files to maintain and they are not consistent with each other.

**Why it happens:** The AI does not know what existing files contain unless you show it. It creates new files based on the name you give it, not by checking for existing files that serve a similar purpose.

**What to do:** Before asking the AI to create a new file, run `ls src/` and show the AI the existing files: "Here are the existing files in `src/`. Do not create duplicates." Or ask "Does `src/` already have a user authentication module?"

**What to look for when reviewing:** Check for semantic overlap between new files and existing files. If two files exist with different names but similar content, that is a duplicate that needs merging or deleting.

## Quick reference

### Project structure decision tree

```
Is the project a small script or experiment?
  YES → Flat layout is fine
  NO  → Continue

Does the project have or will it have more than one module or feature?
  YES → Use src/ layout
  NO  → Flat layout is still fine, but src/ is a good habit

Will the project be deployed to production?
  YES → Use src/ layout, separate tests/, add config/, create .env.example
  NO  → Continue

Are there multiple related projects that share code?
  YES → Consider a monorepo tool (Turborepo, Nx)
  NO  → Keep as separate repos
```

### File location guide

| File type | Where it goes | Committed? |
|---|---|---|
| Application code | `src/` | Yes |
| Tests | `tests/` or `src/` alongside code | Yes |
| Environment variables (local) | `.env` | No |
| Environment variable template | `.env.example` | Yes |
| Static configuration | `config/` | Yes |
| Documentation | `docs/` | Yes |
| README | Project root | Yes |
| Entry point | `src/main.py` or `src/index.js` | Yes |
| Third-party dependencies | `requirements.txt`, `package.json`, `go.mod` | Yes |
| Lockfile | `requirements.lock`, `package-lock.json`, `go.sum` | Yes |

### Minimum viable project structure

```
myproject/
  src/              ← all application code
  tests/            ← all test code
  .env.example      ← documents required env vars (no real values)
  .gitignore        ← ignores .env, node_modules/, __pycache__/, etc.
  README.md         ← one page, setup + run instructions
  requirements.txt  ← or package.json, go.mod, etc.
```

## Go deeper

- [GitHub - Well-Supported Project Structures](https://docs.github.com/en/repositories/creating-and-managing-repositories/about-repositories) — Official documentation on repository structures (verified 2026-04)
- [Python Packaging User Guide: src Layout](https://packaging.python.org/en/latest/tutorials/packaging-projects/) — Official Python packaging documentation recommending `src/` layout (verified 2026-04)
- [Twelve-Factor App: Config](https://12factor.net/config) — The canonical reference for storing config in environment variables (verified 2026-04)
- [BFG Repo-Cleaner Documentation](https://rtyley.github.io/bfg-repo-cleaner/) — Tool for removing sensitive files from Git history (verified 2026-04)
- [Turborepo: When to Use a Monorepo](https://turbo.build/repo/docs/handbook) — Practical guide to monorepo trade-offs from the Turborepo team (verified 2026-04)