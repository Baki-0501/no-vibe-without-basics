# Prompt Engineering for Code

## What is it?

Prompt engineering is the skill of writing instructions for an AI that consistently produces the output you want — not just an output that looks right, but one that is actually correct, well-structured, and ready to use. It is the meta-skill of vibe coding: every other module in this curriculum becomes a prompt you write to an AI. If you cannot write the prompt well, you cannot vibe code well.

Think of it like giving instructions to a brilliant but literal-minded colleague. They will do exactly what you ask, not what you meant. "Write a function that processes data" produces something completely different from "Write a Python function that reads a CSV from `/data/input.csv`, aggregates the `amount` column by `category`, returns a dictionary, and raises `ValueError` if the file is missing." The more specific you are, the more useful the output.

## Why it matters for vibe coding

Without prompt engineering skills, vibe coding produces a specific failure pattern: **vibe drift**. You describe what you want, the AI generates something that seems right, you accept it, and three hours later you discover it does not actually solve your problem. You drifted away from your goal without realizing it.

Vibe drift happens for three reasons:

**Prompts are too vague.** "Add auth to my API" produces something completely different from "Add Bearer token authentication to the `/api/users` endpoint using the existing `auth.py` module, following the pattern used in `/api/orders`." Without specificity, the AI fills in the gaps with whatever seems most plausible — and plausibility and correctness are not the same thing.

**Prompts ask for too much at once.** "Build a user authentication system with signup, login, password reset, email verification, and session management" is one prompt too many. The AI will produce something shallow on every dimension instead of something deep on one. You get a system where nothing works well.

**No context about the codebase.** "Write a function that validates email" in a project that uses Pydantic v2 produces a different answer than in a project that uses Django forms or a custom validator. Without telling the AI what already exists in your project, it generates code that does not integrate — it looks fine in isolation and breaks when you try to use it.

The AI does not know what you have not told it. Every gap in your prompt becomes a gap in the output.

## The 20% you need to know

### The four framing elements: role, context, task, format

Every effective prompt for code has four elements. Missing one or more is the single most common cause of bad output.

**Role** tells the AI who to be. "You are a senior Python backend engineer" or "Act as a database architect specializing in PostgreSQL performance." Role framing primes the AI to generate code at the right level of sophistication and in the right style.

**Context** is everything the AI needs to know that already exists. The tech stack, the existing code, the file structure, what libraries are available, what the goal of the project is. Context is not filler — it is the difference between code that integrates and code that stands alone.

**Task** is the specific thing you want. One task per prompt is the rule. "Refactor the user authentication function to be async" is one task. "Add error handling, make it async, add tests, and update the documentation" is four tasks that should be four separate prompts.

**Format** tells the AI how to present the output. "Return a complete Python file" versus "Give me a diff showing only the changed lines." "Explain in three sentences" versus "List the trade-offs as a bullet table." Format controls the shape of the response, not just the content.

A complete prompt looks like this:

```
Role: You are a senior Python backend engineer working on a FastAPI project.
Context: The project uses SQLAlchemy 2.0 with async, Pydantic v2 schemas, and follows a
  layered architecture (routes/ -> services/ -> models/). Existing auth is in
  `core/auth.py` using python-jose for JWTs.
Task: Add a `/api/users/me` endpoint that returns the currently authenticated user's
  profile. Read the user ID from the JWT Bearer token in the Authorization header.
  Follow the same dependency injection pattern used in `/api/orders.py`.
Format: Provide the complete updated `routes/users.py` file showing only the new
  endpoint and any necessary imports. Include a brief diff comment explaining
  what changed.
```

That is a good prompt. Every element is present.

### Chain-of-thought for complex tasks

For multi-step tasks, ask the AI to reason through the problem before generating output. A simple addition to your prompt — "think through this step by step" or "walk me through your approach before writing code" — dramatically improves the quality of complex outputs.

Chain-of-thought works because it forces the AI to plan before executing. For a task like "Design the database schema for a multi-tenant SaaS app," the AI planning aloud will surface assumptions ("do tenants share a database or have separate ones?"), identify edge cases, and pick a coherent approach before writing a single CREATE TABLE statement. Without the chain-of-thought, the AI goes straight to code and the assumptions are buried in the implementation.

When you need chain-of-thought, structure your prompt explicitly:

```
Task: Design the data model for a task management API that supports
  multiple workspaces per user.

Before writing code:
1. List the core entities and their relationships
2. State your assumptions about multi-tenancy
3. Choose between a shared-database or per-workspace schema strategy
4. Then provide the SQLAlchemy models

[then the rest of the prompt]
```

This is not just about getting better output — it is about getting output you can verify. If the AI's stated assumptions are wrong, you catch it before the code is written.

### Few-shot examples

When you need the AI to follow a specific pattern that is hard to describe in words, give examples. Few-shot prompting means including 2-3 examples of the input/output pattern you want in your prompt.

"Follow this pattern" with no example is an invitation for the AI to improvise. "Follow this pattern:" with a concrete before/after example is a template.

Example: You want the AI to write Git commit messages in the conventional commits format. Instead of explaining the format, show it:

```
Task: Write a commit message for the following change.

Change: Add rate limiting to the `/api/search` endpoint using Redis.
  Reject requests with status 429 after 100 requests per minute per IP.

Format: Follow this pattern:
  feat(api): add rate limiting to /api/search endpoint

  - Uses Redis to track request counts per IP
  - Returns 429 with Retry-After header when limit exceeded
  - Limit is 100 requests/minute

Now write the commit message for:
Change: Refactor the user authentication middleware to support both
  Bearer tokens and API keys in the same Authorization header.
```

The example is a template the AI matches against. It is far more reliable than describing the format in prose.

### Iterative refinement: describe what you want, then describe what you got vs. what you expected

The first output from an AI is rarely the final output. The real skill is in the refinement loop: look at what the AI produced, compare it to what you wanted, and write a follow-up prompt that corrects only the gap.

The gap-description follow-up is more useful than re-describing the original task. Instead of:

```
Prompt: "Write a function that does X."
[AI produces code that does X but uses a different naming convention]

Bad follow-up: "Write a function that does X using camelCase."
Good follow-up: "The function uses camelCase for variables but the project uses
  snake_case. Rewrite all variable names to snake_case and keep the logic the same."
```

The good follow-up identifies exactly what is wrong and what you want instead. It does not re-explain the entire task.

Iterative refinement loops look like this:

1. Prompt with role, context, task, format
2. AI produces output
3. You compare output to intent
4. Follow-up: "The output does X but I need Y. Here is exactly what to change..."
5. Repeat until satisfied

Do not skip step 3. Looking at the output and explicitly identifying the gap is not optional — it is the step that prevents vibe drift.

### Negative constraints: what not to do

Telling the AI what not to do is as important as telling it what to do. Negative constraints prevent the AI from taking plausible but wrong paths.

Common negative constraints for code:

- "Do not use `print()` for logging — use the structured logger from `core/logger.py`"
- "Do not write any SQL — all database access goes through the SQLAlchemy models"
- "Do not add new dependencies — use only the libraries already in `requirements.txt`"
- "Do not modify the existing API contract — the `/api/users` response shape must stay the same"
- "Do not use `any` type hints — use explicit types or `Unknown`"

Negative constraints are not pessimistic — they are direct. The AI treats "do not do X" as a hard rule when it is stated clearly in the constraints section of the prompt.

### Diff-based prompting: give existing code to modify

One of the highest-leverage prompting techniques for code is giving the AI existing code and asking it to produce a diff rather than generating from scratch. This is called diff-based prompting.

Instead of: "Write a function that authenticates a user"

Say: "Here is the existing `auth.py`. Add support for API keys alongside Bearer tokens. Show the minimum diff — only the lines that change."

Why this works: the AI has the full context of what already exists, it understands the existing style and patterns, and it produces targeted output you can review line-by-line. Diff-based prompting also makes the AI's changes reviewable — you can see exactly what was added or removed rather than reading a new file and mentally diffing it against the old one.

Diff-based prompting is especially powerful for:
- Adding a feature to an existing file
- Modifying a function's behavior
- Updating an API endpoint's implementation
- Changing a component's styling

When using diff-based prompting, always specify the output format: "Show only the changed lines with line numbers" or "Provide a unified diff." This controls the shape of the output and makes the diff easier to apply.

### CLAUDE.md as a prompt foundation

`CLAUDE.md` (covered in the Context Engineering module) is the prompt foundation that sits under every conversation. Everything in this module assumes `CLAUDE.md` exists and is accurate. The prompts you write in conversation layer on top of it.

A good `CLAUDE.md` means you do not have to repeat context in every prompt. You say "Add a `/api/users` endpoint" instead of "Add a `/api/users` endpoint to our FastAPI project that uses SQLAlchemy 2.0 async with Pydantic v2 schemas in a layered architecture where routes go in `routes/`..." The context is already in `CLAUDE.md`.

What makes a good `CLAUDE.md` for prompt engineering purposes:

- **Tech stack is specific and accurate.** "Python with FastAPI" is vague. "Python 3.12+ with FastAPI 0.110+, SQLAlchemy 2.0 with async, Pydantic v2, Alembic migrations" is specific.
- **Conventions are explicit rules, not preferences.** "We prefer snake_case" is vague. "All Python identifiers use snake_case. HTTP endpoints use kebab-case." is a rule.
- **Constraints are concrete.** "Be careful with production" is not a constraint. "Never write raw SQL in route handlers. All queries go through the repository layer." is a constraint.
- **File locations are specific.** "API routes go in `src/routes/`" is actionable. "Routes are organized logically" is not.

The better your `CLAUDE.md`, the less context you need in each individual prompt.

## Hands-on exercise

**Refine an AI-generated function through three prompt iterations in 15 minutes.**

### Setup

You will use an AI coding tool (Claude Code, ChatGPT, Cursor, etc.) to generate and refine a Python function. Keep track of each prompt and the AI's response to each.

### Steps

1. **Create a new file `exercise_prompts.py`** in your project. You will save your prompts and the AI's responses here.

2. **First prompt — vague (intentionally):**
   ```
   Write a Python function that processes some data.
   ```

   Save the output. This is your baseline — notice how generic and useless it is.

3. **Second prompt — specific with role, context, task, format:**
   ```
   Role: You are a Python engineer.
   Context: A CSV file at `data/sales.csv` has columns: `date`, `region`, `product`, `amount`.
     The `amount` column contains USD values as strings with commas (e.g., "1,234.56").
   Task: Write a function that reads the CSV, converts `amount` to float, groups by `region`,
     and returns total sales per region as a dictionary.
   Format: Return a single Python file with the function and a `main()` that prints the result.
   ```

   Save the output. Compare it to the first output — notice the specificity.

4. **Third prompt — refinement loop:**
   Look at the second output. Identify one thing you would change (e.g., error handling, the return type, naming). Write a follow-up that describes exactly what to change, not the whole task again.

   Example follow-up: "The function works but it does not handle missing `region` values. Add a check that raises a `ValueError` with the row number if `region` is empty or None. Keep everything else the same."

   Save this exchange.

5. **Fourth prompt — add a negative constraint:**
   Now ask: "Add a check that prevents this function from being called with a file that does not exist. Use `Path` from `pathlib`, not `os.path`. Do not modify anything else."

   Save this exchange.

### Expected output

A file `exercise_prompts.py` containing four prompts and their outputs. Review the progression: vague prompt produces vague code, specific prompt produces specific code, refinement produces targeted changes, negative constraints prevent specific errors. This is the full cycle of prompt engineering in miniature.

## Common mistakes

**Mistake 1: One prompt, too many tasks.**

What happens: You ask the AI to "build a REST API with authentication, rate limiting, and error handling" in one prompt. The AI produces a shallow implementation of all three that breaks as soon as you look at any one piece closely. None of the three features are done well.

Why it happens: It feels efficient to bundle everything together. One prompt, one response, done.

How to fix it: One task per prompt. If you have three features, that is three prompts in sequence. Each one produces complete, well-thought-out code. The combined result is better than one mega-prompt.

**Mistake 2: Prompts with no context about the existing codebase.**

What happens: The AI generates code that looks correct but uses libraries, conventions, or patterns that do not match the project. You spend more time rewriting the AI's code than you would have spent writing it from scratch.

Why it happens: It is natural to focus on the new thing you want and forget to describe the existing environment. You think "the AI should know my project" but it does not unless you tell it.

How to fix it: Every prompt that involves your codebase needs at minimum: the relevant existing files or modules, the tech stack from `CLAUDE.md`, and the conventions being followed. If you find yourself rewriting significant portions of AI output, add more context to the prompt.

**Mistake 3: Not specifying the output format.**

What happens: You get a long prose explanation when you wanted code. You get code when you wanted a diff. You get a complete file when you wanted only the changed function. The AI produces something technically correct but in the wrong shape.

Why it happens: Format feels like an afterthought — the content is the important part. But the format determines how usable the output is.

How to fix it: Always specify the format. "Return only the changed function" versus "Show the complete updated file" versus "Give me a unified diff." If you do not specify, the AI guesses and guesses wrong roughly half the time.

**Mistake 4: Accepting the first output without refinement.**

What happens: The first output looks reasonable so you use it. Three days later you realize it does not handle an edge case that matters in production, or it uses a pattern that does not integrate with the rest of the codebase.

Why it happens: First outputs feel "good enough." Refining feels like extra work. The cost of accepting a mediocre first output is paid later, invisibly.

How to fix it: The refinement loop is not optional. Always do at least one follow-up: "Does this handle [edge case]? Does this follow the project's [convention]?" If the answer is no or uncertain, refine. The cost of a 30-second follow-up is far less than the cost of a production bug.

**Mistake 5: Negative constraints are missing or too vague.**

What happens: The AI uses `print()` for logging, adds a new dependency, writes raw SQL, or modifies an existing API contract. Each of these violates project conventions that you knew about but did not state explicitly.

Why it happens: Constraints feel obvious. You think "obviously it should not use `print()`" but the AI does not know what is obvious to you.

How to fix it: Name the constraint explicitly. "Do not use `print()` — use `logger.info()` from `core/logger`." "Do not modify the response schema of `/api/users`." "Do not add new dependencies." The AI cannot guess what you have not said.

## AI-specific pitfalls

**Pitfall 1: AI generates code that is confidently wrong.**

Code that looks syntactically correct and follows the right patterns can still be semantically wrong — using the wrong algorithm, misunderstanding a requirement, or implementing the inverse of what was asked. This is the most dangerous failure mode because it does not look like a failure.

What to look for: The code looks professional and plausible. Read it line-by-line against the requirements, not just for style. Ask: "Does this actually do what I asked for?" If you cannot verify the logic in 30 seconds, the AI's confidence has fooled you.

**Pitfall 2: AI continues generating code past the part you actually asked about.**

The AI may keep generating additional functionality, tests, or variations beyond what you requested. "Write a function that does X" produces not just the function but a full module with error handling, tests, and documentation. This wastes time and introduces code you did not ask for and may not want.

What to look for: If the output is significantly longer than expected, it has likely gone beyond your request. Trim the scope explicitly in the next prompt: "Ignore the tests. I only want the function itself with its imports."

**Pitfall 3: AI assumes a different tech stack than your project uses.**

The AI may suggest `express.Router()` in a FastAPI project, or use SQLAlchemy syntax for a Prisma project, or recommend a library that is not in your `requirements.txt`. This happens because the AI defaults to the most common pattern for the language, not the pattern your project actually uses.

What to look for: Verify the imports and library calls in AI output match your project's actual dependencies. If the AI uses `from sqlalchemy import Column` in a Prisma project, it has lost context of the stack.

**Pitfall 4: AI does not read its own previous output correctly.**

When given a long response and asked to modify a specific part, the AI may lose track of what it originally wrote and produce inconsistent changes. It may also hallucinate line numbers, function names, or variable names that do not exist in its previous output.

What to look for: When asking for a targeted modification, always repeat the relevant portion of the previous output in your prompt. "In the function you just wrote, change `x` to `y`" is ambiguous. "In the `parse_csv` function on line 12, the variable `total` should be named `total_sales`" is unambiguous. Repeat the exact code you want modified to anchor the AI.

**Pitfall 5: AI's chain-of-thought reasoning is treated as verified truth.**

When you ask for chain-of-thought reasoning, the AI produces a plausible-sounding explanation of its approach. Plausible does not mean correct. The reasoning can be wrong even when the final code looks right — or the reasoning can be right but the code implementation diverges from the stated approach.

What to look for: Read the reasoning, not just the code. If the AI's stated assumptions do not match your requirements, correct the assumptions in a follow-up before accepting the code. The chain-of-thought is a planning tool, not a truth — verify it.

## Quick reference

### Prompt template — the four framing elements

```
Role: [who the AI should be]
Context: [what already exists — tech stack, relevant files, constraints]
Task: [one specific thing, stated precisely]
Format: [how to present the output — code, diff, explanation, etc.]
```

### When to use each technique

| Technique | When to use |
|---|---|
| Role framing | Any time you need code at a specific sophistication level |
| Chain-of-thought | Multi-step tasks, architectural decisions, database design |
| Few-shot examples | When the desired output pattern is easier to show than describe |
| Iterative refinement | Always — first output is never the final output |
| Negative constraints | Every code generation — name at least 2 things NOT to do |
| Diff-based prompting | Modifying existing files — give the file, ask for the diff |

### Refinement follow-up template

```
The output [describes what was produced].
I need [what you actually want].
Specifically: [the exact change required].
Keep everything else the same.
```

### Negative constraint checklist (include at least 2 per code prompt)

- [ ] Do not use [library/pattern] — use [the correct one]
- [ ] Do not modify [existing contract/file]
- [ ] Do not add new dependencies
- [ ] Do not use [anti-pattern — e.g., raw SQL, print(), any type]
- [ ] Do not change [the API response shape, function signature, etc.]

### Diff-based prompting checklist

- [ ] Provide the exact file content or relevant section
- [ ] State what should change (add, modify, remove)
- [ ] Specify output format: "show only changed lines" vs. "complete file"
- [ ] Review the diff line-by-line before applying

## Go deeper

- [Anthropic: Effective prompting techniques](https://docs.anthropic.com/en/docs/build-a-claude-app/prompting-strategies) (verified 2026-04)
- [Anthropic: Claude Code — writing effective prompts](https://docs.anthropic.com/en/docs/claude-code/usage-tips) (verified 2026-04)
- [Prompt engineering guide — chain-of-thought, few-shot, and more](https://www.promptingguide.ai/techniques) (verified 2026-04)
- [GitHub: Diff-based prompting patterns for code review](https://github.blog/developer-skills/github-copilot/how-to-write-code-with-github-copilot-prompt-engineering/) (verified 2026-04)
- [Google: Prompt engineering best practices for code generation](https://developers.google.com/people/techniques/prompt-engineering) (verified 2026-04)
