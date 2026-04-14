# LLM Fundamentals for Builders

## What is it?

Large Language Models (LLMs) are neural networks trained on enormous text corpora to predict the next token given a sequence of preceding tokens. When you send a prompt to an LLM, you are not querying a database — you are sampling from a probability distribution the model learned during training. Everything about how these models behave stems from this core fact: they are next-token predictors with a learned statistical model of language.

**Tokens** are the atomic units the model processes. They are not words — a word like "anthropic" might be one token, but "resolving" could be split into "res" + "olving". A rough rule of thumb is 1 token ~= 4 characters ~= 3/4 of a word in English. When you hear "context window of 128k tokens," that means the total input + output you can send in a single API call must fit within that limit.

**The context window** is the total "working memory" available to the model during a single request. Everything — your system prompt, all prior conversation turns, retrieved documents, and your current query — must fit inside it. When the context window is full, older content gets pushed out. This is why long conversations sometimes feel like the model "forgets" things you said earlier.

**Temperature** controls how deterministic the model's output is. At temperature 0, the model always picks the highest-probability next token — the same prompt yields the same answer every time. At higher temperatures (e.g., 0.7–1.0), the model samples from a broader probability distribution, introducing randomness. Think of it as the difference between a GPS always picking the fastest route (temperature 0) versus one that sometimes suggests scenic detours (temperature 0.8).

**System prompts** set persistent behavior — the model's role, constraints, and operating context. **User prompts** are the task-specific inputs. The model processes them together, and the system prompt's influence can fade over a long conversation.

## Why it matters for vibe coding

Without this knowledge, you will hit failure modes you cannot diagnose:

**The model "forgets" context you know you gave it.** You pasted in a 50-page document at the start of the conversation and asked about it on message 15. It cannot answer. The document consumed most of your 128k-token context window and got evicted. Without understanding tokens and context windows, you do not know why, and you do not know how to fix it.

**Code generation is non-deterministic when you want determinism.** You set temperature 0.9 to "get creative with the algorithm," got a working solution, then ran it again and got completely different code. Now you have inconsistent behavior in your codebase with no way to reproduce the original output. Or conversely, you left temperature at 0 for a "creative" task and got bland, generic output — and did not know to raise it.

**JSON output is malformed.** You ask the model to output JSON, it gives you something that looks like JSON but has trailing commas, comments, and unquoted keys. You do not know that you need to explicitly request structured output mode — plain text JSON is not the same as JSON mode.

**Model choice is arbitrary.** You default to the most capable model (o3 or GPT-4.5) for every task, burning through budget at 10x the speed of a cheaper model for simple tasks a 5-cent model could handle. Or you use a cheap model for a task that requires nuanced reasoning and spend hours debugging subtle logic errors it introduced.

**RAG retrieves the wrong context.** You implement retrieval-augmented generation, see the model citing documents, and trust the answer — but the retrieval step pulled semantically similar but topically wrong chunks. The model confidently synthesizes garbage. Without understanding embeddings and retrieval, you do not know where the failure originated.

## The 20% you need to know

### Tokens and context windows

Tokens are the currency of LLM interactions. Every API call costs tokens. Every prompt you send consumes tokens. Understanding this arithmetic is essential:

- **Count tokens before you send.** A rough English text-to-token formula: `tokens ≈ words × 1.3`. A 1,000-word document is roughly 1,300 tokens.
- **Context window is a budget, not a guarantee of use.** Sending 100k tokens in a prompt does not mean 100k tokens are available for your task — the model must also use tokens for its own output generation. A 128k context window with a 4k output limit means you effectively have ~124k tokens of working memory.
- **Input tokens and output tokens are priced separately.** Most providers charge different rates for input vs. output. Long system prompts cost you every single API call.
- **Prompt caching reduces cost.** Some providers ( Anthropic, OpenAI) cache the static parts of your prompt (system prompt, base context). If your prompt structure is stable across calls, you pay less after the first call.

Practical tip: Use the provider's token counting tool or the `tiktoken` library (OpenAI) to count tokens before sending large prompts. Surprises here are expensive.

### Temperature and sampling

Temperature controls output randomness. The correct setting depends entirely on the task:

| Task | Recommended temperature | Why |
|---|---|---|
| Code generation (deterministic, reproducible) | 0.0 | You want the same input to produce the same output. Bugs should be reproducible. |
| JSON/structured output | 0.0–0.1 | JSON syntax must be exact. Any randomness risks malformed output. |
| Bug investigation / root cause analysis | 0.0–0.2 | You want precise, focused reasoning, not creative leaps. |
| Drafting documentation or prose | 0.3–0.5 | Some variation makes writing feel natural, not robotic. |
| Brainstorming, ideation | 0.7–1.0 | You want diverse, unexpected outputs. |
| Refusal / classification tasks | 0.0 | These should be as consistent as possible across calls. |

**Top-p sampling (nucleus sampling)** is an alternative or complement to temperature. `top_p=0.9` means the model only considers tokens that together account for the top 90% of probability mass, discarding long-tail unlikely tokens. Setting both temperature and top-p is redundant for most use cases — pick one. For vibe coding, defaulting to temperature=0 for all code tasks and reserving higher temperatures for brainstorming is a solid rule.

### System prompts vs. user prompts

The system prompt sets the model's operating context for the entire conversation. The user prompt is the task. The model processes them together.

Key distinctions:

- **System prompt content tends to fade in very long conversations.** After 20+ exchanges, the model may subtly drift from the system prompt's instructions even if they are still in the context. This is called "context window decay" or "prompt decay."
- **System prompts have a cost.** Every token in your system prompt is sent on every API call. A 2,000-token system prompt sent 100 times per day is 200,000 input tokens per day — a significant portion of your budget.
- **User prompts override system behavior.** If your system prompt says "always use Python" but the user says "rewrite this in JavaScript," the user wins. The model cannot refuse a direct instruction from the user.

Structuring your system prompt efficiently: put the most important instructions first ( primacy effect ), use clear delimiters for variable sections, and keep it under 1,000 tokens for high-frequency calls.

### Model selection trade-offs

Major providers offer tiered models. Here is the practical trade-off matrix as of early 2026:

| Model family | Representative model | Cost rank | Speed rank | Capability rank | Use when |
|---|---|---|---|---|---|
| Anthropic | o3 (high reasoning) | Highest | Slowest | Highest | Complex multi-step reasoning, architecture decisions, hard bugs |
| Anthropic | Claude 3.7 Sonnet | Mid-high | Moderate | Very high | General coding, balanced workloads |
| Anthropic | Claude 3.5 Haiku | Lowest | Fastest | High (single-step) | Fast edits, simple queries, high-volume simple tasks |
| OpenAI | GPT-4.5 | Highest | Moderate | Highest | General-purpose, strong on world knowledge |
| OpenAI | GPT-4o | Mid | Moderate | Very high | Balanced cost/performance for most tasks |
| OpenAI | GPT-4o mini | Low | Fast | High | Simple, repetitive tasks |
| OpenAI | o3-mini | Mid | Moderate | High (reasoning) | Code-heavy reasoning tasks, cheaper than o3 |

**Practical decision tree:**

1. Is this a simple, single-step task under 50 lines of code? → Use Haiku or GPT-4o mini.
2. Is this complex reasoning (architecture, multi-file refactors, subtle bugs)? → Use o3 or GPT-4.5 class models.
3. Is this a long-running batch job (processing 1,000 records)? → Always use the cheapest capable model; capability differences do not matter at scale.
4. Is this a vibe coding session where you want speed + quality + budget consciousness? → Default to Sonnet or GPT-4o. Escalate to o3 only when the task demands it.

Cost awareness: o3 costs roughly 10-15x more per token than Haiku. Using o3 to write a `for` loop is a budget hemorrhage you will not notice until end of month.

### Prompt caching

Prompt caching lets you declare parts of your prompt as static so the provider only processes them once, then reuses the cached computation for subsequent calls. It is not the same as context caching — the distinction matters:

- **Prompt caching** (API-level): You structure your API call so the provider knows which parts are static. They charge you the full token cost on the first call, then a significantly reduced cost on subsequent calls that reuse the same static prefix. OpenAI's `cache_control` and Anthropic's `cache` block are examples.
- **Context window caching** (in-context): You manually paste static content (documentation, codebase context) into every API call. You pay for those tokens every time.

Use prompt caching when: your system prompt is large (1,000+ tokens), you make many API calls with the same base instructions, or you are building a high-volume application.

**Common mistake:** Using prompt caching for dynamic content. If the "cached" part changes every call, you get no benefit — and you may get charged the full price anyway while the provider struggles to find a cache hit.

### Structured output (JSON mode)

When you need machine-readable output, plain text is unreliable. Models can and will:

- Add explanatory prose outside the JSON block
- Use trailing commas
- Use unquoted keys (`{name: "Trevor"}` instead of `{"name": "Trevor"}`)
- Miss fields you asked for
- Invent fields you did not ask for

**JSON mode** (called `response_format` in OpenAI APIs, `structured_outputs` in Anthropic) constrains the model to produce valid JSON that matches a schema you provide. Use it whenever you are parsing model output programmatically.

```python
# OpenAI structured output example
from openai import OpenAI
client = OpenAI()

response = client.responses.create(
    model="gpt-4o",
    input="Extract the name and role from: Sarah Chen, CTO of Synthwave Inc.",
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "Person",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "role": {"type": "string"},
                    "company": {"type": "string"}
                },
                "required": ["name", "role"]
            }
        }
    }
)
print(response.output[0].content[0].json_value)
# {"name": "Sarah Chen", "role": "CTO", "company": "Synthwave Inc."}
```

When using structured output, always validate the returned JSON in your code. Structured output guarantees valid JSON syntax, not semantic correctness of the content.

### Embeddings and RAG basics

**Embeddings** convert text into vectors — arrays of floating-point numbers that represent the semantic meaning of text in a high-dimensional space. Texts with similar meanings produce vectors that are "close" to each other in this space. "Python list comprehension" and "how to write a for-loop in one line" are close; "Python list comprehension" and "how to bake a cake" are far apart.

**RAG (Retrieval-Augmented Generation)** is a pattern where you:
1. Pre-process your documents and convert them to embeddings
2. Store those embeddings in a vector database (Pinecone, Weaviate, pgvector, Chroma)
3. At query time, embed the user's question, find the nearest document chunks in the vector store
4. Inject those retrieved chunks into the model's context window along with the user's question

RAG is not magic. Its failure modes are specific: if your document chunking is wrong (splitting related content across chunks, or chunking so large that noise dilutes signal), retrieval fails regardless of how good your embedding model is. If your embeddings are stale (the document changed but the vector database was not updated), the model will confidently retrieve wrong information.

## Hands-on exercise

**Run 10 API calls with different temperature settings and compare outputs in 15 minutes.**

This exercise requires an OpenAI or Anthropic API key. If you do not have one, the code below uses a mock that simulates temperature effects — run it without keys.

### Steps

1. Create a file `temp_comparison.py`:

```python
import os
from anthropic import Anthropic

client = Anthropic()  # Set ANTHROPIC_API_KEY in your environment

prompt = """Write a Python function that validates an email address.
Return only the function — no explanation, no tests, no markdown.
"""

temperatures = [0.0, 0.3, 0.7, 1.0]
outputs = {}

for temp in temperatures:
    message = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=256,
        temperature=temp,
        messages=[{"role": "user", "content": prompt}]
    )
    outputs[temp] = message.content[0].text.strip()
    print(f"\n=== Temperature {temp} ===")
    print(outputs[temp])

# Compare consistency
print("\n\n=== Consistency Analysis ===")
print(f"Temperature 0.0 and 0.3 are identical: {outputs[0.0] == outputs[0.3]}")
print(f"Temperature 0.0 and 1.0 are identical: {outputs[0.0] == outputs[1.0]}")
print(f"Temperature 0.7 and 1.0 are identical: {outputs[0.7] == outputs[1.0]}")
```

2. Run it:

```bash
python temp_comparison.py
```

3. **What to observe:**
   - At temperature 0.0: output should be consistent across runs (run it twice to confirm)
   - At temperature 0.7 and 1.0: expect variation in variable names, comment style, or algorithm approach
   - At temperature 0.0, JSON or code should be clean; at higher temperatures, you may see prose or formatting artifacts

4. **Verify the key insight:** Code generation should use temperature 0.0. Notice how even at 0.3, the output may vary slightly — which is fine for drafting prose, but dangerous for reproducible code.

### Expected output

You will see four different code outputs. At 0.0, the function should be clean and reproducible. At 1.0, you should see at least one instance of extra explanatory comments, slightly different variable naming, or non-standard library choices — all harmless individually but chaotic if the same prompt runs in production 1,000 times.

## Common mistakes

**Mistake 1: Forgetting that context windows have a hard limit and no FIFO guarantee.**

What happens: You paste in a long document, continue the conversation for 20 exchanges, then ask about something from the document. The model says it does not have that information. You did not hit the token limit — but the provider's implementation may evict content from the middle of the context (not just the oldest) based on positioning heuristics.

Why it happens: Most documentation focuses on the token count model (oldest tokens get evicted first). Some implementations are more complex.

How to fix it: Never rely on content being present after many turns. If you need the model to retain something across a long session, put it in a file the model can re-read (CLAUDE.md, a memory file) or explicitly re-inject it before asking about it. The "Context Engineering" module covers this in depth.

**Mistake 2: Setting temperature too high for code generation.**

What happens: You are vibe coding, you want "good code," so you leave temperature at 0.7. The AI writes a function, it looks great, you use it. You run it again in a different context and the AI generated a subtly different version — maybe it used `pathlib` instead of `os.path`, or added a different edge case handling. You now have inconsistent behavior across your codebase.

Why it happens: Temperature controls randomness. At 0.7, you are explicitly asking for variation. Code that works differently on different runs is a maintenance nightmare.

How to fix it: Set temperature=0.0 for all code generation. If you want the AI to explore different approaches, do it deliberately — ask for three options at temperature 0.7, then pick one and regenerate it at temperature 0.0 for the final version.

**Mistake 3: Not understanding what token limits mean for long prompts.**

What happens: You build a RAG system, retrieve 20 document chunks, embed them all in your prompt, and wonder why the model ignores half of them. Your prompt is 180k tokens and your context window is 200k — the model has 20k tokens left for output generation, and it cannot reason effectively over 180k tokens of context.

Why it happens: Context window limits are often quoted as if they describe input capacity. They do not account for the tokens the model needs to "think" (chain-of-thought reasoning consumes significant context) or the tokens needed for output.

How to fix it: Treat your context window as 70-80% of its rated capacity for effective reasoning. A 200k context window is reliably usable to about 140k tokens of input. Chunk your retrieval results, rank them by relevance, and only include the top results.

**Mistake 4: Using JSON mode as a substitute for validating the output.**

What happens: You use structured output, get back valid JSON, parse it, and use it directly in your payment processing code without validating the `amount` field. The model put `"amount": "-$500.00"` because it misread the source document. Your validation logic that trusts the JSON structure silently processes the negative amount.

Why it happens: JSON mode guarantees syntactically valid output. It does not guarantee that the values are correct, sensible, or safe to use without further validation.

How to fix it: Always validate the semantic content of structured output. Check that numbers are in expected ranges, that enums are valid values, that required fields are present and non-null. Structured output is a floor, not a ceiling, for output quality.

**Mistake 5: Picking the most capable model for every task.**

What happens: You use o3 for generating a Python list comprehension. It works perfectly and costs $0.50. You do this 50 times a day. You have spent $25 on tasks a $0.005 Haiku call could handle.

Why it happens: The most capable models feel superior in the moment. The cost difference is invisible until end of month.

How to fix it: Build a model selection habit. Simple edit under 30 lines → Haiku or GPT-4o mini. Multi-file refactor or complex bug → Sonnet or GPT-4o. Architectural decision or novel algorithm → o3 or GPT-4.5. If you are not sure, start with the cheaper model and escalate if output quality is insufficient.

## AI-specific pitfalls

**Pitfall 1: AI confidently generates JSON that is syntactically valid but semantically wrong.**

You ask the model to extract structured data from a document. You use JSON mode. The model gives you valid JSON that passes your schema validation — but a field like `revenue` contains `"N/A"` instead of a number, or `employees` says `"10-50"` instead of an integer. The schema validator passes it; the downstream code breaks.

What to look for: Validate the semantic content of every JSON field, not just its type. Use JSON Schema `minimum`, `maximum`, `enum` constraints to catch obviously wrong values at the schema level.

**Pitfall 2: AI "hallucinates" a tokenization count and gives you incorrect budget advice.**

You ask the model to estimate how many tokens a document uses. The model gives you a number. You use that number to plan your prompt, exceed the context window, and lose content. The model was wrong about the token count — it cannot accurately count tokens any better than you can.

What to look for: Never rely on the model for token counting. Use a dedicated tokenizer library (`tiktoken`, Anthropic's tokenizer, or the provider's token counting API). The model is a language predictor, not a token counter.

**Pitfall 3: AI conflates system prompt instructions with user instructions in long conversations.**

You set up a detailed system prompt with role definitions and constraints. After 30+ turns, the AI starts behaving slightly outside those constraints. You did not change anything — but the model is attending more to recent user content than to the system prompt. This is not a bug; it is how attention mechanisms work in long contexts.

What to look for: In long sessions, re-state the critical constraints explicitly in your user message. "Reminder: this is a production codebase — do not add new dependencies." System prompt is the floor; explicit in-message reminders are the ceiling.

**Pitfall 4: AI recommends the same model for all tasks.**

You ask an AI "what model should I use?" It recommends GPT-4o or Claude Sonnet because those are what it knows best. It does not know your cost constraints or the specific requirements of your task. Its recommendation is anchored on capability, not efficiency.

What to look for: Use the decision tree in this module. Do not default to the most expensive model unless the task genuinely requires it.

## Quick reference

### Temperature decision tree

```
Is this a code generation task?
  YES → temperature = 0.0
  NO
  Is this a brainstorming / creative task?
    YES → temperature = 0.7–1.0
    NO
    Is this a classification / refusal task?
      YES → temperature = 0.0
      NO → temperature = 0.3
```

### Model selection matrix

| Task complexity | Single step | Multi-step / Reasoning |
|---|---|---|
| Simple (edit 1 file, <50 lines) | Haiku / GPT-4o mini | Sonnet / GPT-4o |
| Complex (multi-file, architecture) | Sonnet / GPT-4o | o3 / GPT-4.5 |

### Token budgeting rule of thumb

| Context window | Effective input budget (80%) |
|---|---|
| 32k | ~25k tokens |
| 128k | ~100k tokens |
| 200k | ~140k tokens |

### Structured output checklist

- [ ] `response_format` / `structured_outputs` set in API call
- [ ] JSON Schema defined with required fields and types
- [ ] `strict: true` set (Anthropic) to enforce schema exactly
- [ ] Output parsed and semantically validated (not just syntactically)
- [ ] Null/missing required fields handled gracefully

### System prompt efficiency

- Keep system prompts under 1,000 tokens for high-frequency calls
- Put most critical instructions first (primacy effect)
- Use delimiters (e.g., `=== TASK ===`) to separate static and variable sections
- Audit system prompts monthly for instructions the model no longer follows

## Go deeper

- [Anthropic: Token and context window documentation](https://docs.anthropic.com/en/docs/about-claude/models) (verified 2026-04)
- [OpenAI: Structured outputs (JSON mode)](https://platform.openai.com/docs/guides/structured-outputs) (verified 2026-04)
- [OpenAI: Prompt caching](https://platform.openai.com/docs/guides/prompt-caching) (verified 2026-04)
- [Pinecone: What is RAG — retrieval-augmented generation explained](https://docs.pinecone.io/learn/what-is-rag) (verified 2026-04)
- [Brex: The tokenizer — visual token counter for common LLM models](https://platform.openai.com/tokenizer) (verified 2026-04)
