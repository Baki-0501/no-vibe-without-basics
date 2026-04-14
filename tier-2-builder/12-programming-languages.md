# Programming Languages: Knowing What Your AI Is Actually Generating

## What is it?

A programming language is a set of rules that governs how you express computation. But choosing a language is not just about syntax — it determines what kinds of errors your code can catch automatically, how memory is managed, how your project is structured, and how reliably AI tools can generate correct code for you.

The five languages in this module represent distinct design philosophies:

- **Python** — dynamically typed, interpreted, garbage-collected. Reads almost like English.
- **TypeScript** — statically typed superset of JavaScript. Adds compile-time type safety to a dynamically typed base.
- **Go** — statically typed, compiled, garbage-collected. Designed for concurrency and simplicity.
- **Rust** — statically typed, compiled, ownership-based memory management. No garbage collector. Maximum control.
- **Ruby** — dynamically typed, interpreted, object-oriented. Prioritizes developer happiness over performance.

The differences between them are not cosmetic. They determine whether AI-generated code will be obviously wrong or subtly broken.

## Why it matters for vibe coding

Without a mental model of these languages, you cannot critically evaluate what AI produces. You will see code that looks fine, run it, and encounter errors that make no sense because you don't know what the language guarantees versus what it doesn't.

Specific failure modes:

**AI generates Python that works on the first try but breaks in production.** Python's dynamic typing means AI can generate code that passes arguments in the wrong order, uses the wrong type, or mutates the wrong object — and it runs without error until a specific input triggers the bug. You won't catch it in development if you don't test that specific input.

**AI generates TypeScript that type-checks syntactically but is semantically wrong.** TypeScript's type system is flexible enough that AI can write code that passes the TypeScript compiler but produces logically incorrect behavior at runtime. The type errors AI produces will be confusing because the types are technically valid but not what the code should do.

**AI generates Go code with goroutine leaks.** Go's concurrency model is simple but its mental model is unfamiliar. AI will generate code that spawns goroutines without proper cancellation, leaks memory, or creates deadlocks — and the code will appear to work in small tests only to fail under load.

**AI doesn't understand Rust's borrow checker.** Rust's ownership model is unique. AI will generate code that the borrow checker rejects in ways that are incomprehensible to anyone who hasn't internalized the rules. The error messages are accurate but cryptic without context.

**AI generates Ruby that is slow by default.** Ruby is flexible and forgiving, but that same flexibility lets AI generate code that works but is dramatically slower than it needs to be — looping where it should use built-in methods, creating objects where it should reuse them.

Picking the right language before prompting determines which failure modes you're fighting.

## The 20% you need to know

### Static vs. Dynamic Typing

Static typing means the compiler knows the type of every value before the program runs. Dynamic typing means types are resolved at runtime. This is the most consequential split in modern languages.

| Language | Typing | When types are checked |
|---|---|---|
| Python | Dynamic | Runtime |
| TypeScript | Static (with dynamic escape hatch) | Compile time |
| Go | Static | Compile time |
| Rust | Static | Compile time |
| Ruby | Dynamic | Runtime |

Static typing catches more errors before code runs. Dynamic typing lets code run faster to reach the error. Neither is categorically better — they represent different trade-offs.

**Static typing benefits for vibe coding:**
- The compiler catches entire classes of bugs that AI commonly introduces — wrong argument types, missing fields, incorrect method signatures
- Refactoring is safer because the compiler tells you what breaks
- AI-generated code that doesn't type-check is immediately flagged

**Dynamic typing benefits for vibe coding:**
- Faster prototyping — no type annotations needed
- More flexible to AI-generated variations
- Easier to get something running quickly

**The practical reality:** TypeScript and Go catch more AI mistakes at compile time. Python and Ruby let AI make mistakes that only surface at runtime. Rust catches the most mistakes but requires more explicit guidance from AI.

### Memory Models

How a language manages memory determines whether you get unexplained crashes, silent data corruption, or slow leaks.

**Stack vs. Heap:**
- The **stack** is fast, small, and automatically managed. Function calls and local variables live here. When a function returns, its data is gone.
- The **heap** is a large pool of memory for values that need to persist beyond a single function call, or that are shared across the program.

Languages differ in who manages heap allocation:

**Python, Ruby (Garbage Collected):** You create objects freely. The garbage collector periodically scans the heap, finds objects no longer referenced, and frees them. You never call `free`. This is convenient but introduces pauses and makes memory usage less predictable.

**Go (Garbage Collected):** Similar to Python and Ruby, but Go's GC is more concurrent and designed for lower latency. Still a GC though.

**TypeScript/JavaScript (Garbage Collected):** Same model as Python. Objects are allocated on the heap and collected when no references remain.

**Rust (Ownership Model):** No garbage collector. Memory is freed when the last reference to it goes away — automatically, at compile time. This is called RAII (Resource Acquisition Is Initialization). The compiler tracks ownership at compile time, which means you get compile errors instead of runtime crashes for most memory errors.

Understanding the stack/heap split matters for AI because:
- Passing large objects by value copies them (stack) vs. passing by reference shares them (heap)
- AI often creates unnecessary copies, degrading performance
- AI in Rust needs explicit understanding of ownership — it cannot guess the rules

### Type Systems and Where They Fail

**Python:** Type hints are optional and not enforced at runtime. `def greet(name: str) -> str: return f"Hello {name}"` still works if you pass an integer. Type checkers like `mypy` enforce hints if you run them. AI often generates type hints that are syntactically valid but express the wrong constraint — the function says `str` but should accept `int` as well, or vice versa.

**TypeScript:** The type system is structural (types are compatible if they have the right shape) and includes sophisticated features — generics, union types, mapped types. AI frequently generates overly complex types that technically pass but don't reflect the actual data. AI also struggles with TypeScript's escape hatch: `any`, type assertions, and `as` casts that bypass type safety entirely.

**Go:** Struct types are nominal (names must match exactly). Interfaces are structural (any type with the right methods satisfies an interface). Go's type system is simple but limited — no generics until Go 1.18, and even now generics are more limited than TypeScript or Rust. AI often generates Go code that doesn't use interfaces where they should be used, or uses them incorrectly.

**Rust:** The type system includes lifetimes, which express how long references are valid. This is unique among mainstream languages. AI generates code that the compiler rejects because lifetimes are a second-order concern that AI rarely reasons about correctly. Expect to rewrite AI's Rust code more than any other language.

**Ruby:** No static type system at all. Type annotations exist (RBS, Rbi files) but are optional and rarely used. AI has free reign to produce anything. Types are only discovered at runtime when something crashes.

### Ecosystem Maturity

AI generates code that relies on libraries. The maturity of those libraries determines whether AI-generated code is reliable or full of edge cases that the library authors already solved but AI doesn't know about.

**Python:** Enormous ecosystem. Web frameworks (Django, FastAPI, Flask), data science (pandas, numpy, scikit-learn), automation (Selenium, requests). Mature libraries have comprehensive documentation that AI can reference. But old libraries have uneven quality and some are unmaintained. AI will sometimes use deprecated APIs.

**TypeScript/JavaScript:** Massive ecosystem on npm. Frontend frameworks (React, Vue, Next.js) are well-documented and AI-friendly. Backend (Node.js) is mature. But npm has a quality variance problem — many small libraries are poorly maintained. AI's recommendations may not be the best library for the task.

**Go:** Smaller ecosystem than Python or JavaScript, but what exists is high quality. The Go standard library is unusually comprehensive — net/http, encoding/json, database/sql cover most needs. Third-party libraries tend to be well-maintained. Go's philosophy favors simplicity, so AI-generated code is closer to standard library idioms.

**Rust:** Smaller ecosystem but growing. Crates.io is the package registry. Some critical libraries (async runtime — tokio, web frameworks — axum, actix) are excellent. Others are immature or abandonware. AI has less training data on Rust, so it generates less reliably.

**Ruby:** Mature ecosystem for web (Rails), but much smaller than Python or JavaScript for other tasks. AI-generated Ruby for web applications is generally reliable. AI-generated Ruby for system tools or data processing is less reliable because the ecosystem is less suited to those tasks.

### Build Tooling

How you build, test, and run code varies dramatically:

| Language | Build tool | Test runner | Package manager |
|---|---|---|---|
| Python | None (interpreted) | pytest, unittest | pip, poetry, uv |
| TypeScript | tsc, esbuild, swc, Vite | Vitest, Jest | npm, yarn, pnpm |
| Go | go build | go test | go mod |
| Rust | cargo build | cargo test | cargo (crates.io) |
| Ruby | None (interpreted) | rspec, minitest | gem, bundler |

**Python:** No build step. You run `.py` files directly. This seems convenient but means type checking is optional and separate. Running `mypy` or `ruff` is an extra step AI rarely reminds you to take.

**TypeScript:** Requires a build step (compilation to JavaScript). Development servers handle this automatically. Production builds need explicit configuration. AI generated a React app? It probably used `create-react-app` or Vite, which handle builds. But production optimizations (code splitting, tree shaking) are on you.

**Go:** `go build` produces a single binary. No interpreter, no virtual machine needed at runtime. This is the simplest deployment model. `go mod` handles dependencies. AI-generated Go code is easy to build and deploy.

**Rust:** `cargo build` compiles with a visible progress bar. Rust compile times are long. The first build downloads and compiles dependencies. Subsequent builds are faster but still slower than Go. AI rarely mentions compile times. Budget extra time for Rust.

**Ruby:** Interpreted. No build step. `bundle install` installs dependencies, then you run `.rb` files. Rails adds significant tooling but it's all gem-based.

## Hands-on Exercise

**Compare AI-generated code in three languages and identify the failure modes.**

Time: 15 minutes.

You will prompt an AI to generate the same function in three languages and compare the output. This exercise teaches you to see language-specific failure modes.

1. Create a directory for this exercise:
```bash
mkdir language-comparison
cd language-comparison
```

2. Create a file with the specification. Prompt an AI (Claude, ChatGPT, etc.) with:
```
Write a function called `process_orders` that:
- Takes a list of order objects, each with `id` (string), `amount` (number), and `status` ('pending'|'completed'|'refunded')
- Returns a dictionary with three keys: `pending_total` (sum of amounts for pending orders), `completed_total` (sum for completed), `total_refunded` (sum for refunded)
- Returns None if any order has an unknown status
- Includes basic error handling for missing fields
```

3. Save the response for Python, TypeScript, and Go (three separate files).

4. Create the Python file by copying the AI's Python response:
```bash
# Save AI's Python response to process_orders.py
```

5. Create the TypeScript file:
```bash
# Save AI's TypeScript response to process_orders.ts
```

6. Create the Go file:
```bash
# Save AI's Go response to process_orders.go
```

7. For each language, identify what the AI got right and what it got wrong:

**Python (dynamic typing):** Does the AI handle an order with `amount: "fifty"` (string instead of number) differently from an order with `amount: 50`? In Python, the first would either raise a TypeError at runtime or produce wrong math. Check if AI added type hints and whether a type checker would catch the error.

**TypeScript (structural types):** Does the AI define an interface that matches the input shape? Does it handle the union type for status correctly? Does it use `any` anywhere and bypass type safety? Look for places where the AI could have used stronger types.

**Go (interfaces and error handling):** Does the AI define a struct type with explicit fields? Does it return an error from `process_orders` instead of returning `nil`? Go idiom is to return errors explicitly. AI that returns zero values instead of errors is a failure.

8. Run each file and verify behavior:
```bash
# For Python
python process_orders.py

# For TypeScript (requires Node.js and typescript)
npx tsc process_orders.ts && node process_orders.js

# For Go
go run process_orders.go
```

9. Document what you found:
- Which language's output was most correct on first run?
- Which required the most fixes?
- Which errors would a type checker have caught vs. which required runtime testing?

## Common mistakes

**Mistake 1: Assuming Python type hints are enforced.**

What happens: You write `def greet(name: str) -> str: return f"Hello {name}"`, call it with `greet(42)`, and it runs without error until something downstream breaks when the string concatenation fails.

Why it happens: Python type hints are documentation and (optionally) checked by external tools like mypy. They are not enforced at runtime.

How to avoid: Run `mypy` or use a tool like `pydantic` to enforce schemas at runtime. Don't assume type hints provide runtime safety.

**Mistake 2: Using TypeScript `any` as an escape hatch.**

What happens: AI generates code with `any` types to bypass type checking. The code compiles but type safety is completely lost — any downstream error is a runtime crash.

Why it happens: AI sees `any` as a way to make type errors disappear without solving them. It's the TypeScript equivalent of `try: except: pass`.

How to avoid: Replace `any` with proper types. If you don't know the shape of a value, use `unknown` (which forces you to narrow before use) or define an explicit interface.

**Mistake 3: Forgetting that Go copies slices by reference.**

What happens: You write a function that modifies a slice thinking it modifies the original, or you write a function that you think doesn't modify the original but it does.

Why it happens: In Go, a slice is a reference to an underlying array. Passing a slice to a function gives it the same underlying array. Modifying the slice with `append` beyond its capacity creates a new backing array, which does not affect the original. This is a common source of subtle bugs.

How to avoid: Be explicit: if you want to modify the original, pass a pointer. If you don't, make a copy first. Go's FAQ covers this specifically.

**Mistake 4: Creating Ruby objects in loops that should be reused.**

What happens: AI generates Ruby code that works but is 10x slower than idiomatic Ruby because it creates new objects for each iteration instead of reusing.

Why it happens: Ruby has methods like `map`, `select`, `reduce` that operate on collections without explicit loops. AI often falls back to explicit `for` loops with object allocation inside.

How to avoid: Know Ruby's enumerable methods. When you see a `for` loop in Ruby, ask if it should be a `map` or `reduce`.

**Mistake 5: Rust borrow checker violations.**

What happens: AI generates Rust code that the compiler rejects with errors about borrowing, lifetimes, or ownership that make no sense.

Why it happens: Rust's ownership model is unique. Every value has exactly one owner. References must be valid for the lifetime of the borrow. AI frequently generates patterns where a value is moved and then used again, or where a reference outlives the data it points to.

How to avoid: Understand the ownership rules before prompting for Rust. The [Rust Book](https://doc.rust-lang.org/book/) chapters on ownership, borrowing, and lifetimes are prerequisites for vibe coding Rust effectively.

## AI-specific pitfalls

**AI generates Python type hints that are syntactically valid but semantically wrong.**

Python type hints can express the wrong constraint. AI might write `def process(data: dict) -> list:` when the function actually assumes `data` is a dictionary of specific shape (e.g., `{ "items": [...], "count": 5 }`). The type hint `dict` is technically correct but insufficient. The code passes type checking and fails at runtime when it tries to access `data["items"]` on a dict that doesn't have that key.

What to look for: Type hints that describe the gross structure but not the contract. Always verify that the type hint matches the actual usage in the function body.

**AI generates TypeScript generics that are overly complex and unnecessary.**

AI loves generics. It will generate `function process<T extends Record<string, any>, K extends keyof T>(data: T, key: K): T[K]` when `function process(data: any): any` would have been clearer and produced the same runtime behavior. Complex generics are a sign that AI is showing off rather than solving the problem.

What to look for: Generic type parameters that aren't actually needed. If a function doesn't need to preserve the type of its input across calls, it probably doesn't need generics. TypeScript's structural typing makes simple types work more often than AI expects.

**AI generates Go code with goroutine leaks.**

AI knows Go supports concurrency with `go func()`. It will use goroutines liberally without considering cancellation, wait groups, or context propagation. Code that spawns goroutines for background work will accumulate them, leak memory, and eventually crash under load.

What to look for: Every `go` keyword should have a corresponding synchronization mechanism. Look for `sync.WaitGroup`, `context.Context`, or channel-based signals. If the AI spawns goroutines without explaining how they terminate, that's a leak.

**AI doesn't understand Rust's borrow checker and generates unfixable code.**

Rust borrow checker errors cannot be worked around with casts or type assertions. If AI generates code that violates ownership, you cannot make it work by tweaking syntax — you must restructure the algorithm. AI will sometimes propose workarounds (using `Rc<RefCell<T>>` or similar) that add runtime overhead and defeat the purpose of using Rust.

What to look for: Before accepting AI's Rust code, verify that the ownership model makes sense. Does each value have exactly one owner? Are borrows temporary and scoped? If you see `Rc` or `RefCell` in AI output, ask if the ownership model is correct or if a different data structure would eliminate the need for shared ownership.

**AI generates Ruby without considering performance implications.**

Ruby is slow compared to other languages. AI will generate Ruby code that is correct but inefficient — using `each` loops instead of `map/reduce`, creating objects in tight loops, using string concatenation in loops (`+=` creates new strings each iteration). Ruby has idiomatic solutions for all of these, but AI defaults to the most obvious (and slowest) approach.

What to look for: Any `for` loop in Ruby is a candidate for replacement with an enumerable method. Any `+=` on a string in a loop should use an array and `join`. Ruby rewards knowing the standard library methods.

## Quick Reference

### Language Selection Decision Tree

1. **Need maximum ecosystem and libraries?** → Python (data, web, automation) or TypeScript (web, Node.js)
2. **Need performance and single binary deployment?** → Go (simpler) or Rust (faster, more control)
3. **Need memory safety without GC pauses?** → Rust
4. **Need fast development and flexibility?** → Python or Ruby
5. **Need static types that catch AI mistakes?** → TypeScript or Go (Go is more limited but catches more mistakes)
6. **Working on web frontend?** → TypeScript (non-negotiable)

### Memory Model Comparison

| Language | Memory management | GC pauses | Control level |
|---|---|---|---|
| Python | Garbage collected | Yes (stop-the-world) | Low |
| TypeScript | Garbage collected | Yes (V8 optimized) | Low |
| Go | Garbage collected | Yes (concurrent, low-latency) | Medium |
| Rust | Ownership + RAII | None | High |

### Common Type Error Meanings

| Error (TypeScript) | Meaning |
|---|---|
| `Type 'X' is not assignable to type 'Y'` | Shape mismatch — missing or extra fields |
| `Argument of type 'X' is not assignable to parameter of type 'Y'` | Wrong type passed to function |
| `Property 'X' does not exist on type 'Y'` | Accessing field that doesn't exist |
| `Type 'X' is not a function` | Calling something that isn't callable |
| `Expression has no type or is used in a context requiring a variable` | Syntax error or parser confusion |

### Rust Ownership Rules (memorize these)

1. Each value has exactly one owner.
2. When the owner goes out of scope, the value is dropped.
3. You can have either one mutable reference OR any number of immutable references — never both at the same time.
4. References must not outlive the data they point to.

## Go deeper

- [Python Type Hints Documentation](https://docs.python.org/3/library/typing.html) (verified 2026-04) — Official docs on Python's type system.
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/) (verified 2026-04) — Official TypeScript docs covering types, generics, and type narrowing.
- [Go by Example](https://gobyexample.com/) (verified 2026-04) — Practical Go idioms with annotated examples. Better for vibe coding than the official tour.
- [The Rust Book: Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html) (verified 2026-04) — The chapter that unlocks the borrow checker. Read this before generating Rust code with AI.
- [Ruby Monocle: Performance Tips](https://monocle.tw/) (verified 2026-04) — Ruby performance patterns that cover common AI-generated inefficiencies.
