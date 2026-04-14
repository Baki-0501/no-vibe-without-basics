# Reviewing AI Output

## What is it?

Reviewing AI output is the discipline of systematically examining every piece of code an AI tool generates before you run it, merge it, or ship it. It is not proofreading — it is a technical inspection that verifies correctness, security, and fitness for purpose. The analogy is a senior engineer reviewing a junior's pull request: same standard, same checklist, same questions.

The critical difference from normal code review is that with AI output, you did not write the code, you do not necessarily know how it works, and the AI has no skin in the game when it is wrong. A junior engineer who writes bad code at least understands the intent. AI generates code that fits a prompt perfectly while being completely wrong — and it will defend the code confidently.

## Why it matters for vibe coding

Without systematic review, AI becomes a liability rather than a multiplier. Here are the specific failure modes:

**Without API verification, AI will use library functions that do not exist.** AI generates code referencing `numpy.linalg.inv()`, `pandas.Series.sum(axis=2)`, and `requests.json()` — methods that either do not exist or do not mean what the name implies. It invents plausible-sounding APIs from training data patterns. You run the code and get a `NameError` or `AttributeError` in production, or worse, a silent wrong answer.

**Without security review, AI will hardcode credentials and skip input validation.** AI knows what a secure implementation looks like conceptually, but it frequently generates insecure code in practice — `eval(input)` when asked to process user data, hardcoded API keys, SQL queries built with f-strings. Shipping AI-generated auth code without review has caused real breaches.

**Without diff discipline, AI will rewrite entire files and you will lose track of what changed.** AI tends to regenerate an entire file rather than produce a targeted edit. Without reading the diff carefully, you approve changes that silently remove error handling, change business logic, or drop entire functions.

**Without logic tracing, AI bugs look like mystery failures.** AI generates loops that run one too many times, conditions that are inverted, return values that are silently discarded. The code "looks fine" and fails only under specific conditions that you did not test. Without tracing the control flow, you cannot see the bug.

**Without version verification, AI uses library features from the wrong version.** AI trained on documentation from a newer library version will generate code using `numpy.ndarray.take(indices, axis=1)` when your pinned version is numpy 1.18, which does not support that signature. The code fails at import or at runtime with a confusing error.

## The 20% you need to know

### The systematic review checklist

For every AI-generated code change, apply this checklist before merging:

1. **Verify every library call exists at the pinned version.** Check `pip show library-name` or `npm list library-name` against what the code calls. If the code references a method, open the real documentation and confirm the signature.

2. **Trace the control flow end-to-end.** Follow the code from entry point to exit. Does it do what the PR description says? Are there paths that return early, drop errors silently, or skip cleanup?

3. **Scan for security issues.** Hardcoded secrets, string-concatenated SQL, `eval()` on user input, missing auth checks, unvalidated user input flowing into shell commands or file paths.

4. **Check error handling.** Does every external call (database, HTTP, file system) handle failure? Are errors logged? Are error messages informative or just `"Error occurred"`?

5. **Verify logic with a mental execution.** Run through the core logic with a concrete example. If the function is supposed to calculate a 10% discount on orders over $100, walk through `$150 order with 10% discount` and confirm the result is $135, not $15 or $140.

6. **Check for hallucinated imports.** AI will sometimes import modules that seem plausible but do not exist: `from mypackage import helper_utils`. Verify the package exists in `requirements.txt` or equivalent.

7. **Look at the diff, not just the new file.** AI often regenerates entire files. Use `git diff` to see exactly what changed. A function that was present in the old file but silently dropped is a real risk.

### Spotting hallucinated APIs

AI hallucination is not random noise — it is plausible-sounding code that looks correct until you verify it. The most common patterns:

**Method name hallucinations.** AI uses a method name that almost matches a real one but is wrong. `list.flatten()` (should be `sum(list, [])` or `itertools.chain.from_iterable()`), `dict.contains()` (should be `key in dict`), `string.includes()` (should be `in` operator or `find() != -1`).

**Signature hallucinations.** Real method, wrong arguments. `numpy.array.reshape(array, (3, 4))` — the argument order in modern numpy is actually `array.reshape((3, 4))`. `pandas.DataFrame.query('column > 5').filter()` — `filter` was removed in pandas 0.13.

**Non-existent standard library functions.** `time.time_ns()` was added in Python 3.7. `pathlib.Path.read_json()` does not exist — it is `Path.read_text()` then `json.loads()`. `collections.OrderedDict` is deprecated in Python 3.7+ because dicts are now ordered by default.

**How to verify:** When in doubt, paste the exact method call into a Python REPL or Node.js console and call `help()` or `doc()`. Or search the official docs directly. The verification takes 30 seconds and catches the most common class of AI bugs.

### Verifying library versions

AI trained on documentation from 2024–2025 may generate code using features from library versions that are newer than what you have pinned. Always check:

```bash
# Python — check installed version against what AI claims
pip show pandas | grep Version
pip show numpy | grep Version

# JavaScript — check installed version
npm list pandas
npm list numpy
```

If the AI uses a feature that requires a newer version than what you have, you either need to upgrade (with a lockfile update and regression tests) or reject the code and ask for a downgrade.

For Python specifically, check the installed version in your virtual environment — not the system Python. AI running in a fresh context without your actual environment activated will not know what version you have.

### Reading a diff critically

The diff is your primary tool for reviewing AI changes. Read it like a detective examining evidence:

**What to look for in every AI diff:**
- Lines deleted that should not have been (error handling, validation, edge case handlers)
- Entire files rewritten when only a few lines needed to change
- Subtle logic changes (a `>` becomes `>=`, a `+` becomes `-`)
- New imports that are unnecessary or hallucinated
- Missing test updates when business logic changed

**Workflow:**
```bash
# See the diff before staging
git diff

# Or for a specific file
git diff path/to/modified/file.py

# If AI has staged changes already, unstage and review
git restore --staged .
git diff
```

Always read the diff before staging and committing. If the diff looks like AI rewrote more than it needed to, reset, add only what you need, and ask AI to make a targeted edit instead.

### Security red flags in AI output

These are the patterns that appear regularly in AI-generated code and demand immediate rejection or remediation:

**Hardcoded secrets:**
```python
# AI-generated — looks fine, is a breach
api_key = "sk-abc123xyz789..."

# Correct — injected from environment
import os
api_key = os.environ.get("API_KEY")
```

**SQL injection via string concatenation:**
```python
# AI-generated — SQL injection vulnerability
query = f"SELECT * FROM users WHERE id = {user_id}"

# Correct — parameterized query
query = "SELECT * FROM users WHERE id = %s"
cursor.execute(query, (user_id,))
```

**Eval on user input:**
```python
# AI-generated — arbitrary code execution
result = eval(user_expression)

# If you must evaluate math expressions: use ast.literal_eval for literals
# or a proper expression parser like numexpr or pyparsing
```

**Missing authentication checks:**
```python
# AI-generated — route without auth check
@app.get("/orders")
def get_orders():
    return db.query("SELECT * FROM orders")

# Correct — auth check on every protected route
@app.get("/orders")
@require_auth
def get_orders():
    return db.query("SELECT * FROM orders WHERE user_id = %s", (current_user.id,))
```

**No input validation:**
```python
# AI-generated — trusts user input implicitly
def send_email(to_address, subject, body):
    smtp.sendmail("app@company.com", to_address, subject, body)

# Correct — validates before sending
def send_email(to_address: str, subject: str, body: str) -> None:
    if not re.match(r"^[\w.-]+@[\w.-]+\.\w+$", to_address):
        raise ValueError(f"Invalid email address: {to_address}")
```

### When to reject vs. refine

Not every problem in AI output means you should discard it and start over. The decision:

**Reject completely when:**
- The AI fundamentally misunderstood the requirement (it built a REST API when you asked for a GraphQL endpoint, and the codebase uses GraphQL)
- The security vulnerability is structural — it would require a complete rewrite to fix, not a line edit
- The hallucinated API is central to the code and replacing it would cascade into many changes
- The AI regenerated an entire file and the diff shows critical logic was silently dropped

**Refine when:**
- The logic is correct but missing edge cases (you can add tests and validation without rewriting)
- The API usage is slightly off but fixable with a targeted correction
- The code is functional but the error handling is absent or generic
- The performance issue is a missing index or a loop that can be optimized without redesign

When refining, be specific about what to change and why. Do not say "fix this" — say "the `filter()` call on line 42 should be `where()` because `filter()` applies after the query, not before, and this is causing all rows to be loaded into memory." AI can fix targeted, explained corrections. It cannot fix vague dissatisfaction.

## Hands-on exercise

**Review a piece of AI-generated code against the checklist and fix the issues.**

Time: 15 minutes

1. Create this AI-generated "example" file that contains deliberate issues:
```python
# ai_generated_payment.py
# This code was "written by an AI" for a payment processing feature

import os
import stripe
from datetime import datetime

stripe.api_key = "sk_live_abc123xyz789"  # hallucinated API key (not real)

def process_payment(amount, customer_email, card_token):
    # No input validation
    result = stripe.Charge.create(
        amount=amount,  # assumes amount is in dollars, Stripe uses cents
        currency="usd",
        source=card_token,
        description=f"Charge for {customer_email}"
    )
    return result

def get_customer_orders(customer_id):
    # SQL injection vulnerability
    query = f"SELECT * FROM orders WHERE customer_id = {customer_id}"
    cursor.execute(query)
    return cursor.fetchall()

def calculate_discount(order_total, discount_percent):
    # Logic error: applies discount incorrectly
    if discount_percent > 0:
        return order_total - (order_total * (discount_percent / 100))
    else:
        return order_total  # should apply base discount here

def log_payment(payment_id, status):
    # Uses eval on log data — security risk
    print(f"Payment {payment_id} status: {eval(status)}")
```

2. Apply the checklist. For each issue, write down:
   - What is wrong
   - Why it is wrong
   - How to fix it

3. Create a corrected version:
```python
# payment.py — corrected version

import os
import stripe
from datetime import datetime

stripe.api_key = os.environ.get("STRIPE_API_KEY")  # loaded from environment

def process_payment(amount: float, customer_email: str, card_token: str) -> dict:
    # Validate inputs before processing
    if amount <= 0:
        raise ValueError("Amount must be positive")
    if not isinstance(customer_email, str) or "@" not in customer_email:
        raise ValueError(f"Invalid email address: {customer_email}")

    # Convert dollars to cents — Stripe uses smallest currency unit
    amount_cents = int(amount * 100)

    result = stripe.Charge.create(
        amount=amount_cents,
        currency="usd",
        source=card_token,
        description=f"Charge for {customer_email}"
    )
    return result

def get_customer_orders(cursor, customer_id: int) -> list[dict]:
    # Parameterized query — prevents SQL injection
    query = "SELECT * FROM orders WHERE customer_id = %s"
    cursor.execute(query, (customer_id,))
    return cursor.fetchall()

def calculate_discount(order_total: float, discount_percent: float) -> float:
    # Correct discount calculation
    if discount_percent > 0:
        discount_amount = order_total * (discount_percent / 100)
        return order_total - discount_amount
    return order_total

def log_payment(payment_id: str, status: str) -> None:
    # Never eval user data — just log it as-is
    print(f"Payment {payment_id} status: {status}")
```

4. Compare your corrected version against the AI-generated version. Note:
   - What was wrong that a quick diff would have caught
   - What required tracing the logic end-to-end to spot
   - What required security knowledge to identify

**Success criteria:**
- You identified at least 5 distinct issues (hardcoded key, SQL injection, eval, missing validation, amount unit mismatch, logic error)
- Each fix has a clear why — not just "changed it to this" but "why this is correct"

## Common mistakes

**Mistake 1: Running AI code without reading it first.**

What happens: You ask AI to add a feature, it gives you a Python file, you run `python app.py`, it works on the first try, and you ship it. Weeks later you discover the code was sending emails to the wrong address due to a swapped variable, or connecting to a production database because no environment check was present.

Why it happens: AI code often "works" on the surface — no syntax errors, imports resolve, basic happy path runs fine. The bugs are semantic, structural, or security-related and do not surface in a quick local test.

How to fix: Read every file before running it. Even 60 seconds of reading catches the most egregious issues. Use the checklist on every file, not just on code you suspect is wrong.

**Mistake 2: Trusting the import list.**

What happens: AI adds a new import that looks correct. You approve it. The code actually imports `from my_company.utils import format_date` which does not exist in the real codebase, but the import error only fires when that code path is hit, not at import time. The bug ships silently.

Why it happens: Python imports resolve the module name but not the attribute access until runtime. `from mypackage import helper` succeeds if `mypackage` exists, even if `helper` does not. You do not know until the function is called.

How to fix: For any unfamiliar import, verify the attribute exists. `python -c "from mypackage import helper; print(helper)"` catches most hallucinated imports. Or check `pip show mypackage` and read the actual module's `__all__` list.

**Mistake 3: Not checking what the diff deleted.**

What happens: AI regenerates a 500-line file to add a 10-line feature. You read the new code and it looks fine. You miss that the old code had a `try/except` block that handled a specific edge case, or that an `if __name__ == "__main__":` cleanup routine was dropped. These deletions cause silent failures in production.

Why it happens: When AI regenerates a whole file, diff tools show every line as a change. It is easy to approve the diff without noticing that 30 lines of the old implementation vanished.

How to fix: Use `git diff OLD_FILE NEW_FILE | less` and look at deleted lines specifically. Or use a visual diff tool that highlights deletions in red. If you see a large deletion that does not have an obvious replacement, trace the logic before and after to confirm nothing was lost.

**Mistake 4: Accepting AI's explanation of its own code.**

What happens: AI generates a function and when you ask "does this handle the case where the database is unavailable?", it confidently says yes. You accept the answer without checking. The code does not handle it — the AI hallucinated a confident-sounding but incorrect explanation.

Why it happens: AI that generated the code has no awareness of whether the code it wrote actually handles the edge case. It will confidently describe what it *intended* to write, not what it actually wrote.

How to fix: Verify behavior by tracing the code path, not by asking the AI to self-assess. Ask "what happens when database connection is None" and then look at the code yourself to see if it handles it.

**Mistake 5: Not checking the library version dependency.**

What happens: AI generates code using `pandas.DataFrame.agg()` with a dict argument that was deprecated in pandas 1.3 and removed in pandas 2.0. Your CI fails when it runs the test suite against pandas 2.x. You spend an hour debugging before finding the version mismatch.

Why it happens: AI uses patterns from its training data, which spans many library versions. Without knowledge of your pinned version, it defaults to whatever was most common in its training corpus.

How to fix: Always check `pip show pandas` (or relevant library) before merging AI code that uses advanced library features. If the feature is newer than your pinned version, either upgrade or ask AI to use the older API.

## AI-specific pitfalls

**AI invents plausible but non-existent library methods.** This is the single most common AI code bug. It generates `numpy.matrix.flatten()`, `pandas.Series.sum(axis=1)`, `json.loads(file)` (should be `json.load(file)` — note the missing `d`), `list.remove_all(item)`. Always verify method names in the REPL.

**AI uses deprecated or removed library features.** Deprecations change faster than AI training data updates. `pandas.DataFrame.append()` was deprecated in 1.4 and removed in 2.0. `numpy.string_` is deprecated in favor of `numpy.bytes_`. AI will generate code that runs on older library versions but fails on newer pinned versions.

**AI generates code with the wrong data type assumptions.** `int("1,000")` fails in Python. `float("1,000.00")` fails if the system locale uses comma as a decimal separator. AI often assumes US formatting. When processing user input or data from files, these silent failures are hard to catch without boundary testing.

**AI generates code that is correct in the training data but wrong for your version.** `pathlib.Path.exists()` has been supplemented by ` Path.is_file()` and `Path.is_dir()` for clarity. AI uses the older API. This is not wrong, but it signals that AI may not be using the most idiomatic current API.

**AI generates secure-looking code that is structurally insecure.** AI knows what bcrypt is and will write `bcrypt.hashpw(password)` but forget to call `bcrypt.checkpw()` during login, or vice versa. It knows what parameterized queries are but sometimes inserts the parameter marker in the wrong place in the query string. Security code requires more scrutiny than typical code.

**AI uses `console.log` for application logging in Node.js.** `console.log` is not structured — it writes to stdout, which most production log aggregators cannot parse. AI does not know your logging infrastructure unless you tell it explicitly. Specify structured logging requirements in your prompt.

## Quick reference

### AI code review checklist

| Check | Action if failed |
|---|---|
| Every library call exists at installed version | Look up real signature in docs REPL |
| No hardcoded secrets | Move to `os.environ.get()` |
| No string-concatenated SQL | Replace with parameterized query |
| No `eval()` on user input | Replace with safe parser or direct comparison |
| Auth check on every protected route | Add decorator or middleware |
| Input validation on user-supplied data | Add validation before use |
| Diff shows only what changed | Reject if file was silently rewritten |
| Error paths are handled | Add `try/except` with logging |
| Tests cover new logic | Add test or update existing test |

### When to reject vs. refine

```
Is the fundamental approach wrong (wrong API, wrong paradigm)?
  YES -> REJECT. Ask AI to rewrite from scratch with correct context.

Is the code structurally sound but missing validation or edge cases?
  YES -> REFINE. Add the missing pieces with specific instructions.

Does the diff show critical logic was silently dropped?
  YES -> REJECT. Restore original. Ask AI for a targeted edit only.

Is the API usage slightly off but fixable with one line?
  YES -> REFINE. Point to the exact line and correct signature.

Does the code have a security vulnerability that requires structural rework?
  YES -> REJECT. Do not ship insecure code. Rewrite with security as constraint.
```

### Hallucination verification in the REPL

```python
# Verify a method exists and get its signature
import numpy as np
help(np.ndarray.flatten)  # shows docstring and signature

# Verify pandas method
import pandas as pd
help(pd.DataFrame.query)
# Or in Jupyter: pd.DataFrame.query?

# Verify standard library
import collections
print(dir(collections))  # lists all attributes
```

## Go deeper

- [Python requests library — verifying installations](https://requests.readthedocs.io/) (verified 2026-04) — The `requests` library is one of the most commonly hallucinated APIs. The official docs show real method signatures.
- [Stripe API Reference: Authentication](https://docs.stripe.com/api/authentication) (verified 2026-04) — Real example of how API keys must be passed — via `Authorization: Bearer` header, never hardcoded in source.
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) (verified 2026-04) — The authoritative guide to parameterized queries, the defense AI-generated SQL most commonly skips.
- [npm SemVer calculator](https://semver.npmjs.com/) (verified 2026-04) — Check if a library version range in `package.json` actually resolves to the version you think, and whether features AI referenced exist at that version.
- [PyPI and package version verification with pip](https://pip.pypa.io/en/stable/cli/pip_show/) (verified 2026-04) — The canonical reference for `pip show` and `pip freeze`, which let you verify exactly what is installed in your environment.
