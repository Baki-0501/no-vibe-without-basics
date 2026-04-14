# Reading Code

## What is it?

Reading code is the skill of tracing what a program actually does — not what you hope or assume it does, but what the execution path reveals. It is fundamentally different from reading prose. Prose moves in one direction through a single narrative. Code is a graph of decisions, branches, and function calls that can converge, diverge, and loop back on themselves.

The core activity is **control flow tracing**: following the path of execution from entry point to exit, tracking which branches are taken under which conditions, and building a mental model of what happens to data as it moves through the system.

Closely related is the **call graph**: a map of which functions call which other functions. When you see a function called `send_email`, you should immediately wonder: who calls `send_email`? What arguments does it receive? What does it return? What side effects does it produce? The call graph tells you the answer.

The other half of reading code is **search-driven navigation**: using grep and structural search to find patterns, definitions, and usages across the codebase. You rarely read a file from top to bottom. You jump to what matters, verify what you find, and build a picture from fragments.

## Why it matters for vibe coding

Vibe coding amplifies every failure mode of not reading code carefully. When you ask an AI to write a feature and it produces code that looks plausible, your job is to verify that code does what you intended. If you cannot read code critically, you cannot verify — you are just trusting the AI's confidence level.

Specific failures that occur without this skill:

**AI generates plausible but incorrect code and you cannot tell.** The AI produces a function that looks correct — correct syntax, reasonable variable names, plausible logic — but the branching logic is wrong. A condition is inverted, a loop doesn't handle the empty case, an early return skips cleanup. Without the ability to trace control flow, you run the code and get wrong behavior, then spend hours debugging something that a 30-second code read would have caught.

**AI uses an API from the wrong library version.** The AI pulls syntax from documentation for a newer version of a library than the one you have installed. The code compiles, but at runtime you get `AttributeError: module 'x' has no attribute 'y'`. Without understanding how imports and module structure work, you cannot read the error and map it back to the version mismatch.

**AI generates code based on a misremembered dependency contract.** The AI writes a function that assumes `get_user()` returns `None` when a user is not found, but the actual codebase raises a `UserNotFoundError` exception instead. The code that calls it has no error handling. Without reading the actual dependency, you have no way to catch this before running it.

**You cannot review AI diffs critically.** AI tools show you git diffs of what they changed. A diff shows lines added and removed — but that is not the same as understanding what the change does. Without the ability to read a diff and ask "what changed, and why is this change correct?", you are approving changes you do not understand.

**AI generates code with bad naming and you adopt it.** The AI calls a function `process_data()` and calls a variable `tmp`. Without understanding naming conventions as a readability tool, you accept this and the codebase degrades. Readable code is a choice — AI will generate mediocre naming unless you understand why it matters and can push back.

## The 20% you need to know

### Tracing control flow

Control flow is the order in which statements execute. To trace it:

1. **Find the entry point.** For a script, it is the top-level code. For a function, it is the first line after the signature. For an event handler, it is the callback registration.

2. **Follow the branches.** At every `if`, identify which branch is taken for your test case. At every loop, identify how many iterations run and what happens on each. At every `return`, note what value is returned and whether the function continues after.

3. **Track variable state.** Variables change. When you see `result = filter(users, active=True)`, ask: what is `result`? A list? A Query object? When does it get evaluated — now or later?

4. **Identify side effects.** A function that returns a value has one kind of effect. A function that writes to a file, sends a network request, or mutates a global has another. Side effects are where bugs hide — a function can return the right value but corrupt state on the way out.

A practical technique: **rubber duck tracing**. Pick a specific input to the function and walk through it line by line, narrating what happens. "User `alice` calls `get_user_roles` with `user_id=42`. The first line calls `db.query(User).filter(id=42)`. That returns a Query object, not a User. The next line calls `.first()` on it, which returns either a User instance or None." If you cannot narrate it, you do not understand it.

### The call graph mental model

You do not need to draw an actual graph. You need to build a lightweight mental model of who calls what.

When you encounter a function, immediately ask:
- Who calls this function? (Find all callers with grep.)
- What does this function call? (Look inside the function body.)
- What does it return? What does it assume about its inputs?

The call graph is directional: A calls B does not mean B calls A. A function can have many callers (it is a "leaf" in the graph) or many callees (it is a coordinating function that delegates to others). High-level coordinating functions are where you get the overall logic; leaf functions are where the work happens.

For vibe coding, the call graph matters most when you are verifying AI output: when the AI modifies a function, you need to know what else it affects. If the AI changes the return type of a helper function, every caller is potentially affected.

### Grep and search strategies

grep is your fastest way to navigate an unfamiliar codebase. The key is knowing what to search for:

**Find a function definition:** `grep -n "^def function_name" src/` — the `^` anchors to line start, filtering class methods and false matches.

**Find all usages of a symbol:** `grep -rn "symbol_name" .` — `-r` recursive, `-n` line numbers. The `-w` flag forces word matching so `user` doesn't match `username`.

**Find a pattern across file types:** `grep -rn --include="*.py" "TODO\|FIXME\|HACK" .` — find all marked code.

**Find import statements:** `grep -n "^import\|^from" file.py` — shows what a file depends on.

**Context around matches:** `grep -C 3 "pattern" file.py` — shows 3 lines before and after each match, so you see the surrounding logic.

**Inverse search:** `grep -v "pattern" file.py` — shows lines that do NOT contain the pattern. Useful for finding unhandled cases.

IDE features (Go to Definition, Find References) do the same thing with a mouse. Learn the keyboard shortcuts. In VS Code, `F12` goes to definition, `Shift+F12` finds references.

### Reading diffs

A diff shows two versions of a file. Lines starting with `-` are removed; lines starting with `+` are added. Context lines (unchanged) are shown without a prefix.

To read a diff critically:

1. **Read the file name and hunk headers first.** The diff shows which file changed and which function was modified. This tells you the scope.

2. **Ask: what changed at the function level?** Don't just read line-by-line. Ask: did the function gain a new parameter? Did it move from returning one thing to returning another? Did a new branch appear inside it?

3. **Check for structural changes.** A diff that removes 20 lines and adds 20 lines in a different order is different from a diff that removes 20 lines and adds 5. Deleted code is deleted capability. Added code is new capability. Both need scrutiny.

4. **Verify the diff matches the intent.** If you asked the AI to "add error handling to `send_email`" and the diff only changes a variable name in an unrelated function, the AI did not do what you asked.

5. **Read the unchanged context around changes.** The lines immediately before and after a change often explain why the change was made. Without context, you see the what without the why.

### Import and module structure

Imports are the dependency map of a codebase. Reading them tells you what a file needs to work.

In Python, `import os` brings the `os` module into the namespace. `from os import path` brings only `path`. `from . import utils` is a relative import from the current package.

When you see an import, ask:
- Is this a standard library module? (`os`, `sys`, `json` — no external install needed)
- Is this a third-party package? (`requests`, `numpy` — check `requirements.txt`)
- Is this a local module? (`from .models import User` — relative to this file's location)

The **module search path** is the list of directories Python searches when you write `import foo`. The first match wins. This matters when you have naming conflicts: if a file in your project is named `email.py`, it will shadow the standard library `email` module. AI tools sometimes generate code that accidentally creates this shadow.

To find the actual module being used: in Python, `print(foo.__file__)` shows the path to the module on disk. This is the fastest way to resolve a "which `email` module is this?" question.

### Naming conventions as a readability tool

Good naming encodes meaning. When you read `def calculate_average(numbers)`, you know: the input is a collection, the output is a single value representing the average. When you read `x = filter(users, active=True)`, the variable name `users` suggests it is a collection, and `active=True` suggests a filter criterion.

Naming patterns to recognize:

- **Boolean variables and functions** are often prefixed with `is_`, `has_`, `can_`, `should_`: `is_active`, `has_permission`. This signals the return type is bool.
- **Plural nouns for collections**: `users`, `items`, `results`. Singular would be wrong.
- **Verb phrases for functions**: `send_email`, `validate_input`, `get_user_by_id`. Functions do things.
- **Constants in UPPER_SNAKE_CASE**: `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT`. This signals the value should not change.
- **Private/internal signals**: `_internal_function` (leading underscore in Python). This signals the function is not part of the public API.

When AI generates code with bad naming — `data`, `temp`, `result2` — the fix is not cosmetic. Renaming is refactoring. A variable named `data` tells you nothing about what it contains. Rename it to `user_records` and the code reads itself. Push back on AI naming in code review.

## Hands-on exercise

**Trace control flow in an unfamiliar file and document the call graph.**

Time: 15 minutes. You will produce a written artifact.

### Setup

Pick any `.py` file in this project that is longer than 50 lines and that you have not read recently. If this is a new project, use the main entry point (`main.py` or `app.py`).

### Steps

1. **Find all functions defined in the file.** Use grep:
   ```bash
   grep -n "^def " your_file.py
   ```
   List every function name and its line number.

2. **Pick one function.** Choose the most important one — usually the one called from the entry point or the one with the most parameters.

3. **Trace its control flow.** For the chosen function, manually walk through the logic line by line. Write down:
   - Every branch (`if`/`elif`/`else`) and what condition it checks
   - Every loop and what it iterates over
   - Every function call and what it returns
   - Every `return` statement and what value is returned

4. **Build the call graph.** For each function call inside your chosen function, use grep to find where that function is defined:
   ```bash
   grep -n "^def function_name" .
   ```
   Do this for every callee. Draw a simple hierarchy: your function at top, the functions it calls below, and those functions' callees below them.

5. **Document your findings.** Write out:
   - The entry point and its path through the function
   - The call graph (can be ASCII art: `main -> process -> validate -> db`)
   - Any side effects you identified
   - One thing that surprised you or that you did not expect

### Expected output

A written trace with:
- At least one function fully traced
- A call graph showing at least 3 functions
- A note of one surprising finding

Example format:
```
Function: process_order
Entry: order_id=123
Path:
  1. get_order(123) -> Order object or raises NotFound
  2. if order.status == 'pending': validate_payment(order)
  3. validate_payment -> calls payment_gateway.verify()
  4. payment_gateway.verify() -> returns bool

Call graph:
  process_order
    -> get_order
    -> validate_payment
       -> payment_gateway.verify

Surprise: validate_payment is called even though it has no error handling.
If payment_gateway.verify() raises, the exception propagates uncaught.
```

## Common mistakes

**Mistake 1: Reading code top-to-bottom like prose.**

What happens: You start at line 1 and read every line sequentially. After 500 lines, you have lost the thread — you remember the beginning but not why anything matters.

Why it happens: Prose is linear. Code is not. You were never meant to read code like a novel.

How to fix it: Read code with a mission. "I need to understand how `send_email` handles failures." Start at the function, read its body, follow calls outward. Do not start at the top of the file.

**Mistake 2: Assuming the code does what the comments say.**

What happens: A comment reads "this validates the input" and you move on. But the actual code does not validate — it only checks that the input is non-None, not that it is valid.

Why it happens: Comments are written once and rarely updated. They are not executable. Code is the ground truth.

How to fix it: Read the code, not the comments. If the comment and code disagree, trust the code and either fix the comment or flag it as outdated. Comments that contradict the code are a bigger maintenance hazard than no comments.

**Mistake 3: Not tracking what happens to return values.**

What happens: A function `get_user(id)` is called but its return value is never stored. The next line uses `current_user` which was set to something else earlier. The developer assumed `get_user` set a global, but it does not.

Why it happens: In some languages, functions can modify globals or accept global state. Reading only the function body without the call site misses this.

How to fix it: Every function call, ask: what does it return? What happens to that return value? If the return value is not assigned, either the call is a no-op or it has a side effect. Verify which.

**Mistake 4: Not verifying which version of a function or module is imported.**

What happens: `import json` in Python 2 vs Python 3 behaves differently for unicode handling. `import email` in a project with `email.py` in the root shadows the stdlib module. The code looks correct but fails in specific runtime conditions.

Why it happens: Python's import system uses the first match on the search path. Without knowing how it works, you do not know when it has surprised you.

How to fix it: When debugging import-related failures, `python -c "import module; print(module.__file__)"` tells you exactly which module is being loaded. This one command resolves most "wrong module" confusions.

**Mistake 5: Skimming diffs instead of reading them.**

What happens: You see "5 files changed, 50 insertions, 20 deletions" and approve the PR. The diff contains a security regression — a permission check was removed. You did not see it because you were skimming for the summary, not reading for correctness.

Why it happens: Diffs are dense and take effort. The numbers (files changed, lines added) feel like enough information. They are not.

How to fix it: Read every diff as if you have to explain it to someone who will execute it in production. Ask: what changed, what new behavior does this introduce, what could go wrong? If you cannot answer those three questions from the diff, you have not read it.

## AI-specific pitfalls

**Pitfall 1: AI generates code with correct syntax but wrong logic, and you run it without reading it critically.**

This is the most common vibe coding failure. The AI writes code that looks right — correct indentation, plausible variable names, appropriate function calls — but the branching logic is inverted, the loop doesn't handle the base case, or the return value is wrong. Because it looks correct, you trust it.

The fix is not to distrust AI — it is to have the skill to read code and catch the error. Every piece of AI output needs to be read before it is run. If you do not know how to trace control flow, you cannot verify the output.

What to look for: After the AI generates a function, trace it manually with a specific input before running. Write down what should happen. Compare to what the code actually does.

**Pitfall 2: AI uses an API from a library version that is newer than the one you have.**

AI training data includes documentation from many versions of popular libraries. When it generates code, it may use syntax from the latest version — features that did not exist in the version you have installed. The code looks correct and fails at runtime with a confusing error.

What to look for: Check the library version in your project and compare it to what the AI's code uses. Run `pip show library-name` or check `requirements.txt`. If the AI calls `client.foo()` and you do not see `foo` in the source, the version mismatch is likely the cause.

**Pitfall 3: AI does not understand a dependency's actual behavior and generates incorrect usage.**

AI can describe a library's interface from training data, but training data describes what the library does in general — not what this particular version does, not what this specific edge case means, not what this particular dependency chain looks like in your project.

Example: the AI generates code that assumes `requests.get()` raises an exception on HTTP error codes (it does not — it returns a Response object with `status_code` set). The calling code has no `try/except` and the error goes undetected.

What to look for: Verify every external call's error handling. Ask: what does this function do when it fails? Does it raise an exception, return None, return an error object? Then verify the calling code handles all three cases.

**Pitfall 4: AI generates plausible variable names that do not match the codebase's conventions.**

If the project uses `get_user_by_id(id: int) -> User`, the AI may generate `fetch_user(id)` with different argument names and return type annotations. The code is internally consistent but breaks the project's naming conventions, making it harder to search and cross-reference.

What to look for: Compare AI-generated function names and argument names against existing functions in the same module. If the project uses `get_X`, the AI should not introduce `fetch_X` or `retrieve_X` in the same context. Consistency matters for searchability.

**Pitfall 5: AI does not read the actual call graph and generates code that is disconnected from how the project works.**

The AI may generate a new function that calls a non-existent helper, or that returns a value in a format the existing callers do not expect. This happens because the AI does not have live access to the call graph — it infers it from training data, not from the actual codebase.

What to look for: Before running AI-generated code, verify every function call the new code makes actually exists in the project. Use grep to search for function definitions. If `process_batch()` is called but there is no `def process_batch` in the project, the AI invented it.

## Quick reference

### Control flow trace checklist

- [ ] Find the entry point
- [ ] Identify every branch (if/elif/else) and which is taken
- [ ] Identify every loop and its termination condition
- [ ] Track every return value
- [ ] Identify every side effect (I/O, mutation, network)
- [ ] Verify the function's callers exist and expect what it returns

### Grep command quick reference

| Goal | Command |
|---|---|
| Find function definition | `grep -n "^def func_name" .` |
| Find all usages | `grep -rn "symbol" .` |
| Word-boundary match | `grep -rnw "symbol" .` |
| Find in specific file type | `grep -rn --include="*.py" "pattern" .` |
| Show context (3 lines) | `grep -C 3 "pattern" file.py` |
| Show only filenames | `grep -rl "pattern" .` |
| Find import statements | `grep -n "^import\|^from" file.py` |
| Invert match | `grep -v "pattern" file.py` |

### Diff reading checklist

- [ ] Read the file name and function name first
- [ ] Ask: what changed at the function level?
- [ ] Check added code vs deleted code (structural change?)
- [ ] Verify the diff matches the requested change
- [ ] Look at unchanged context lines
- [ ] Ask: what could go wrong with this change?

### Module import troubleshooting

```python
# Find the actual path of any imported module
import os
print(os.__file__)  # /usr/lib/python/os.py

# Find the path of a specific module
import email
print(email.__file__)  # shows which email module is loaded
```

### Naming convention signals

| Pattern | Meaning |
|---|---|
| `is_`, `has_`, `can_` | Returns boolean |
| `get_`, `fetch_`, `load_` | Retrieves data |
| `create_`, `build_`, `make_` | Creates new data |
| `update_`, `modify_` | Changes existing data |
| `validate_`, `check_` | Returns boolean or raises |
| `_private` | Internal, not for public use |
| `UPPER_SNAKE_CASE` | Constant value |
| Plural nouns | Collection/iterable |

## Go deeper

- [Python documentation: import system](https://docs.python.org/3/reference/import.html) (verified 2026-04)
- [VS Code: navigation and search shortcuts](https://code.visualstudio.com/docs/editor/editingevolved) (verified 2026-04)
- [GitHub: reading git diffs effectively](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-comparing-branches-in-pull-requests) (verified 2026-04)
- [Pyangler: control flow tracing techniques (blog post)](https://pangian.com/blog/2018/reading-code/) (verified 2026-04)
- [Google: code review guide — reading code critically](https://google.github.io/eng-review/reviewer/) (verified 2026-04)