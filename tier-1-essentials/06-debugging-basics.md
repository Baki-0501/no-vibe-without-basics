# Debugging Basics

## What is it?

Debugging is the process of finding the root cause of code that doesn't work as expected. It is not "fixing" — it is *investigating*. The fix is the last step, not the first. Beginners rush to change code when they see an error. Debugging is the discipline of understanding *why* the error happened before touching anything.

Think of it like a doctor who won't prescribe medicine before diagnosing. You have symptoms (a crash, a wrong output), you gather evidence (stack trace, logs, variable values), you form a hypothesis (what's causing it), and only then do you intervene. The stack trace is your starting evidence. Every error has a story to tell — your job is to read it.

## Why it matters for vibe coding

Vibe coding amplifies every debugging failure mode and adds new ones. When you run an AI-generated script and it crashes, the AI's error message is often the *only* error output you see. If you don't know how to read a stack trace, you can't tell the AI what went wrong. You just say "it broke" and the AI guesses.

Specific failure modes:

**You accept AI fixes without verifying them.** AI tools will "fix" a bug by deleting the code that had the bug, not by fixing the logic. If `process_order()` raises a `KeyError`, the AI might remove the `process_order()` call entirely — the error goes away, but the functionality is gone. Without reading the stack trace yourself, you can't tell the difference between a real fix and a symptom suppression.

**You can't diagnose AI-generated errors.** AI generates code that produces error messages the AI itself doesn't understand well. It will say "that's a TypeError" when it's actually a logic error that only *manifests* as a TypeError. Reading the stack trace yourself tells you the real story.

**The AI ignores the error type.** Error types (`TypeError`, `KeyError`, `ReferenceError`) are diagnostic signals. When the AI sees "error on line 42", it often treats all errors the same. But a `KeyError` means something different from a `TypeError`, and responding to them correctly requires knowing the difference. The AI will suggest fixes that are technically wrong for the error type.

**You run code with no way to see what went wrong.** AI-generated code often runs "successfully" (no exception thrown) but produces wrong output. Without logging and without running the code to inspect intermediate values, you accept wrong output as correct because you have no evidence otherwise.

## The 20% you need to know

### Reading stack traces

A stack trace is a report of the call stack at the point where an error occurred. It shows you *the path the program took* to get to the failure. Every line in the stack trace is a function call. You read it from the bottom up: the last line is where the program actually crashed, and the lines above it show who called that code.

**Python stack trace:**

```
Traceback (most recent call last):
  File "src/billing.py", line 42, in calculate_total
    subtotal = items[0]['price'] * items[0]['qty']
  File "src/billing.py", line 18, in process_order
    items = parse_order(order_id)
  File "src/billing.py", line 5, in parse_order
    return db.fetch_one(ORDER_QUERY, order_id)
KeyError: 'price'
```

Read this bottom-up:
- `KeyError: 'price'` — the error type and message. Something looked up `'price'` in a dict and didn't find it.
- Line 42 in `calculate_total` — where it crashed. The lookup `items[0]['price']` failed.
- Line 18 in `process_order` — who called `calculate_total`.
- Line 5 in `parse_order` — who called `process_order`.

The crash happens at the bottom. The *reason* you got there is in the lines above.

**JavaScript/browser stack trace:**

```
TypeError: Cannot read properties of undefined (reading 'email')
    at UserProfile.render (http://localhost:3000/src/components/UserProfile.jsx:42:24)
    at ReactRenderer.renderComponent (http://localhost:3000/node_modules/react-dom.js:5898:17)
    at Object.render (http://localhost:3000/node_modules/react-dom.js:1247:34)
```

Same structure: read bottom-up. The crash is at `UserProfile.jsx:42`. It tried to read `.email` on `undefined`. The call chain above shows it came from React's rendering system.

**Key signals in any stack trace:**
- **Error type** — `KeyError`, `TypeError`, `ReferenceError`, `SyntaxError`, `RangeError`
- **Error message** — the specific thing that went wrong ("cannot read property 'email' of undefined")
- **File and line number** — exactly where the error occurred
- **Call chain** — the sequence of function calls that led there

The file:line is your starting coordinate. Everything else is context.

### Error types you will see most often

`TypeError`: An operation was performed on the wrong type. Calling a method on `None`/`null`, adding a string to a number, passing the wrong number of arguments.

`KeyError` / `ReferenceError`: A lookup failed. The key didn't exist in the dict, or the variable was never defined.

`SyntaxError`: The code is malformed. Python can't parse it, or the JavaScript engine choked on it. These always crash *before* any code runs — they are a category of their own.

`AttributeError`: You tried to call a method or access a property that doesn't exist on an object.

`IndexError`: You tried to access an index that is out of bounds in a list or array.

`ConnectionError` / `Timeout`: Network-level failures. These tell you the problem is not in your code logic but in the connection itself.

### Print vs. logging

`print()` is fine for quick investigation. `logging` is what you use in real code.

`print()` output goes to stdout — it disappears in production, it cannot have severity levels, and it cannot be turned off without editing code. `logging` lets you set levels (`DEBUG`, `INFO`, `WARNING`, `ERROR`), write to files or external systems, and silence verbose output without changing code.

For AI-vibe-coded code, add logging when you need to understand what the AI generated. A common pattern: wrap AI-generated functions with a logger to capture what inputs they receive and what outputs they produce.

```python
import logging
logger = logging.getLogger(__name__)

def process_order(order_id):
    logger.info(f"Processing order: {order_id}")
    result = ai_generate_order_processing(order_id)  # AI-generated
    logger.info(f"Result for {order_id}: {result}")
    return result
```

In JavaScript/Node.js, `console.log` is your equivalent. `console.error` goes to stderr. For production Node.js, use a logger like `pino` or `winston`.

```javascript
const logger = require('./logger');

async function processOrder(orderId) {
  logger.info(`Processing order: ${orderId}`);
  const result = await aiGenerateOrderProcessing(orderId);
  logger.info(`Result for ${orderId}:`, result);
  return result;
}
```

Rule: If you expect the code to run more than once, use logging. If it's a throwaway script and you just want to see a value right now, `print`/`console.log` is fine.

### Browser DevTools basics

When debugging web applications, DevTools is the primary environment. Three panels matter most:

**Console tab (Sources panel or dedicated Console):** JavaScript output, error messages, and an interactive REPL. Type JavaScript here to execute it in the page's context. This is where you see runtime errors.

**Sources panel:** The debugger. Set breakpoints, step through code, inspect variables. The watch panel lets you track specific expression values as you step.

**Network tab:** All HTTP requests. If a request fails, you see the status code, response body, and timing. If your API call returns a 500 error, it shows here.

For most vibe-coding debugging:
1. Open DevTools (`F12` or `Cmd+Option+I`)
2. Go to Console to see errors first
3. For deeper inspection, use the Sources panel and set a breakpoint on the line the error stack trace mentions

### Breakpoints

A breakpoint stops code execution at a specific line so you can inspect the program state. Instead of adding `print()` statements, you set a breakpoint and the program pauses right when it gets to that line.

In DevTools Sources panel: click the line number gutter to set a breakpoint. The program stops there, and you can hover over variables to see their values, use the Console to evaluate expressions, and step through the next lines one by one.

Types of breakpoints:
- **Line breakpoint:** stops at a specific line. Use when you know roughly where the bug is.
- **Conditional breakpoint:** stops only when a condition is true (e.g., `items.length === 0`). Use to skip over code that runs correctly until the bug case.
- **XHR breakpoint:** stops when an HTTP request is made to a specific URL. Use when debugging API calls.

For Python, if you are using VS Code with the Python extension, click in the gutter next to the line number to set a breakpoint. Press `F5` to start the debugger and it will stop at your breakpoints.

### Isolating bugs

Isolating a bug means reducing the problem to its smallest reproducible case. This is the most important debugging skill, and it is what makes debugging systematic rather than random.

Steps:
1. **Identify the input that triggers the bug.** What exact data causes the failure?
2. **Remove everything that doesn't affect the bug.** Comment out processing steps, simplify the input, strip away layers until only the bug remains.
3. **Reproduce it in isolation.** If you can run the minimal case and get the error, you've isolated it.

A common pattern: the AI generates a complex pipeline, and it fails. Rather than debugging the entire pipeline, start with just the first step, confirm it works, add the second step, confirm it works, and so on. When you add step N and it breaks, step N is where the bug lives.

For a web app: open DevTools, go to the Network tab, clear it, perform the action that triggers the bug, and inspect the exact request and response. If the request is wrong, the bug is in the code that *made* the request. If the request is right but the response is wrong, the bug is on the server.

### Rubber duck method

Explain your code line by line to an inanimate object (or your empty terminal). The act of describing what each line does forces you to confront the gap between what you *think* the code does and what it *actually* does.

The specific failure mode this catches: code that looks correct but does something unexpected because of a subtle logic error. When you explain `items[0]['price'] * items[0]['qty']` out loud, you might realize that `items` is a dict, not a list, and `items[0]` is not the first item — it's the key `'0'` which doesn't exist.

This works because you are the only one who fully knows what the code is supposed to do. The rubber duck doesn't know anything, so it can't fill in the gaps. You have to make every assumption explicit.

## Hands-on exercise

**Debug two intentionally broken scripts in 15 minutes.**

Set up:
```bash
mkdir debugging-exercise && cd debugging-exercise
```

Create this Python file as `buggy_python.py`:

```python
def calculate_discount(price, discount_percent):
    return price - (price * discount_percent)

def apply_discount_to_items(items, discount_percent):
    discounted = []
    for item in items:
        # BUG: discount_percent passed as int, not converted to decimal
        discount = calculate_discount(item['price'], discount_percent)
        discounted.append(discount)
    return discounted

cart = [
    {'name': 'Widget', 'price': 100},
    {'name': 'Gadget', 'price': 250},
    {'name': 'Gizmo', 'price': 75},
]

print(f"Original prices: {[item['price'] for item in cart]}")
result = apply_discount_to_items(cart, 20)  # Should apply 20% discount
print(f"After discount (should be 80, 200, 60): {result}")
```

Create this JavaScript file as `buggy_js.js`:

```javascript
// Run with: node buggy_js.js

function getUserEmail(users, userId) {
    for (const user of users) {
        if (user.id === userId) {
            return user.email;
        }
    }
    return null;
}

const users = [
    { id: 1, name: 'Alice', email: 'alice@example.com' },
    { id: 2, name: 'Bob', email: 'bob@example.com' },
];

console.log('Looking for user 3...');
const email = getUserEmail(users, 3);
console.log('Email:', email.toUpperCase());  // BUG: calling .toUpperCase() on null
```

**Run and observe:**

```bash
python buggy_python.py
node buggy_js.js
```

**Your task:** Read the errors. Identify the root cause. Fix both files.

1. In `buggy_python.py`: the discount is `20` (an integer percent, e.g. 20%) but the code treats it as `20.0` (a decimal multiplier, e.g. 0.20). The fix is to divide `discount_percent` by 100 before using it.
2. In `buggy_js.js`: when `getUserEmail` doesn't find the user, it returns `null`. Calling `.toUpperCase()` on `null` throws a TypeError. The fix is to handle the null case.

**Verification:** After fixing both:
- `python buggy_python.py` should print `80, 200, 60`
- `node buggy_js.js` should print `Email: null` (not crash)

If either script crashes, keep reading the stack trace and adjusting.

**Time target:** 10 minutes to read the errors and identify the causes, 5 minutes to fix them.

## Common mistakes

**Mistake 1: Changing code without reading the stack trace.**

What happens: You see an error, panic, and start modifying code based on the error message's most visible words. You change a variable name, or add a guard clause, or wrap everything in a try/except. The error changes or moves, but the bug is still there. You are now further from understanding the problem.

Why it happens: Stack traces look intimidating. The instinct is to make the red go away, not to understand it.

How to fix it: Before touching anything, read the stack trace from the bottom up. Identify the error type, the message, the file:line where it crashed, and the function call chain. Write down "Error: [type] at [file:line]: [message]" on a sticky note. Now you have a starting coordinate. Every change you make after this is in service of that specific problem.

**Mistake 2: Treating all errors as equally serious.**

What happens: You see an error and immediately assume the worst — the database is down, the API is broken, the whole system is failing. You start investigating infrastructure when the error is actually a simple `KeyError` from a mistyped field name.

Why it happens: Error messages use scary-sounding names (`ConnectionError`, `TimeoutError`, `InternalError`). The name describes the category, not the severity.

How to fix it: Understand the error type. A `KeyError` in Python means a dict lookup failed — almost never a system-level issue, almost always a data issue. A `ConnectionError` from your HTTP library usually means the server isn't running, not that the code is logically wrong. Match your investigation to the error type.

**Mistake 3: Adding print statements instead of using breakpoints.**

What happens: You pepper the code with `print()` statements trying to find where values go wrong, then forget to remove them, leaving debug output in production.

Why it happens: `print()` is immediate — you see output right away. Setting up a debugger feels slower.

How to fix it: For any bug more complex than "what value is X here?", use a breakpoint. It is faster to step through 10 lines in a debugger than to add 10 print statements and re-run. For one-liner inspection, `print()` is fine. For anything involving a loop or a conditional, use the debugger.

**Mistake 4: Not reproducing the bug before trying to fix it.**

What happens: You hear a report that something is broken, and you start modifying code in a different area that you *think* might be related. The fix doesn't solve the reported issue. You've now changed code and haven't confirmed the fix works.

Why it happens: You want to be helpful and move fast. You skip the "reproduce the bug" step to save time, but end up spending more time because you don't know if your fix works.

How to fix it: Before writing any fix, reproduce the bug in your environment. Confirm you can make it fail on demand. Then make your change and confirm you can make it pass on demand. If you can't make it fail in your environment, you don't understand the bug well enough to fix it.

**Mistake 5: Assuming the AI's fix is correct without testing it.**

What happens: The AI says "I fixed it" and you accept the change without running the code. The AI's fix removes the problematic code rather than fixing it, or introduces a new bug, or doesn't address the root cause.

Why it happens: When the AI says "fixed", you want to believe it. You trust the AI has understood the problem.

How to fix it: Always run the code after any AI-assisted change. Verify the output is correct, not just that it doesn't crash. If the AI's fix deleted code, ask whether the deleted code's original purpose is still served. Check the git diff to understand what changed.

## AI-specific pitfalls

**Pitfall 1: AI suggests a fix without reading the stack trace.**

This is the most common AI debugging failure. The AI sees an error message and generates a plausible-sounding fix — usually adding a null check, or wrapping code in a try/except — without understanding why the error occurred. The fix makes the error go away but doesn't fix the underlying bug.

What to look for: When the AI says "I see the error — just add a null check here", ask it what the *root cause* is. If it cannot explain why the value is null in the first place, the fix is a guess. Press it: "Walk me through the call chain that leads to this error." If it can't, the fix is unsubstantiated.

**Pitfall 2: AI ignores the error type and suggests fixes that are technically wrong for it.**

AI will sometimes describe an error as one type when it is another, or suggest fixes that address a different error type than what actually occurred. For example, calling a "KeyError" a "missing key issue" and suggesting a `.get()` method when the real problem is that the data structure itself is wrong.

What to look for: When you see an error type in the stack trace (e.g., `TypeError`), read what the AI says about it. If it describes the error in vague terms ("something went wrong") rather than the specific technical term, it may not understand the error correctly. Demand precision: "Is this a TypeError or a KeyError?"

**Pitfall 3: AI "fixes" bugs by deleting the buggy code.**

The AI sees a crash at `process_order()`, removes the call to `process_order()`, and the crash stops. Mission accomplished — except the functionality that `process_order()` was supposed to perform is gone. The AI will often describe this as "fixed the issue" without acknowledging that functionality was removed.

What to look for: After any AI fix, check the git diff. If code was deleted, ask whether that code's purpose is still served. If `process_order()` was called in three places and the AI deleted all three calls to make the error go away, the bug is fixed but the feature is gone. A real fix keeps the functionality and changes the implementation.

**Pitfall 4: AI generates code with no logging and you can't figure out what went wrong.**

AI-generated code often runs "successfully" (no exception) but produces wrong output. Without logging, you have no visibility into what the code actually did versus what you expected it to do. The AI says the code works and you believe it, but the output is wrong.

What to look for: After running AI-generated code, explicitly verify the output against what you expect. Check intermediate values, not just the final result. If the code does a calculation, log each step. If the code calls an API, log the request and response. AI that cannot explain its intermediate steps is AI you shouldn't trust for complex logic.

**Pitfall 5: AI "hallucinates" the fix to an error it has never seen.**

When an AI doesn't understand an error, it will sometimes make up a fix — code that it believes will work but is actually completely unrelated to the error. This happens most often with custom error types or errors from less-common libraries.

What to look for: If you don't understand why the AI's suggested fix addresses the error, verify the fix manually. Run the fix in isolation to see if it makes the error go away. If you can't confirm the fix works, revert to the original error and debug from the stack trace yourself.

## Quick reference

### Reading a stack trace

```
Traceback (most recent call last):
  File "src/utils.py", line 15, in parse_config
    return yaml.load(data)
  File "src/utils.py", line 8, in load_yaml_file
    with open(path) as f:
FileNotFoundError: 'config.yaml' does not exist
```

**Step-by-step:**
1. Error type: `FileNotFoundError`
2. Error message: `'config.yaml' does not exist`
3. Crash location: `src/utils.py:8` — `open(path)` failed
4. Call chain: `parse_config` called `load_yaml_file`

**Decision:** The file doesn't exist at the path being passed to `open()`. Check the path, or check the working directory.

### Error type quick reference

| Error Type | Python | JavaScript | Common Cause |
|---|---|---|---|
| Wrong type used | `TypeError` | `TypeError` | Calling method on null/undefined, wrong operand type |
| Lookup failed | `KeyError` | `ReferenceError` | Key not in dict, variable not defined |
| Out of bounds | `IndexError` | `TypeError` (sometimes) | List access at invalid index |
| Malformed code | `SyntaxError` | `SyntaxError` | Parse error — code can't run at all |
| Network failure | `ConnectionError`, `Timeout` | `TypeError` (fetch) | API unreachable, DNS failure |

### Debugging decision tree

```
Error occurred?
  YES
  |
  +-- Read stack trace (bottom-up: error type -> message -> file:line -> call chain)
  |
  +-- Is it a SyntaxError?
  |     YES -> Fix the syntax error first. Nothing else runs.
  |     NO
  |
  +-- Can you reproduce it?
  |     YES -> Is the crash point clear from the stack trace?
  |       YES -> Isolate to that function. Set breakpoint. Inspect variables.
  |       NO -> Add logging to trace the data flow through the call chain.
  |     NO -> This is a logic error (no exception). Add logging to find where output diverges.
  |
  +-- After fix: did you test the output, not just the absence of a crash?
```

### Breakpoint setup

**Chrome DevTools:**
1. Open DevTools (`F12`)
2. Go to Sources panel
3. Find the file in the file tree (left sidebar)
4. Click the line number gutter to set a breakpoint
5. Trigger the action that causes the bug
6. Step with F10 (next line), F11 (step into), F8 (resume)

**VS Code Python:**
1. Click in gutter next to line number
2. Press `F5` to start debugger
3. Hover over variables to inspect values
4. Use Debug Console to evaluate expressions

### When to use print vs. breakpoints vs. logging

| Situation | Tool |
|---|---|
| Quick value inspection during development | `print()` / `console.log()` |
| Complex control flow, need to see all variables | Breakpoint |
| Code that runs in production, need ongoing visibility | Logging |
| Bug involves a specific input you can trigger | Breakpoint at that point |
| Bug happens in production and you can't reproduce locally | Logging |

## Go deeper

- [Chrome DevTools: Debug JavaScript](https://developer.chrome.com/docs/devtools/javascript/) (verified 2026-04)
- [Python logging tutorial — Real Python](https://realpython.com/python-logging/) (verified 2026-04)
- [Node.js debugging with VS Code](https://nodejs.org/en/learn/getting-started/debugging) (verified 2026-04)
- [MDN: Debugging JavaScript](https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Page_types/Code_example_guide) (verified 2026-04)
- [Python traceback module documentation](https://docs.python.org/3/library/traceback.html) (verified 2026-04)