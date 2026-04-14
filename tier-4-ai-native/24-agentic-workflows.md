# Agentic Workflows

## What is it?

An **agentic workflow** is a software system where an AI model takes multiple actions over time to accomplish a goal — not just answering a question or generating a response, but *doing work*: reading files, writing code, running commands, calling tools, and iterating until the job is done.

At the core of this is a **coding agent** — an AI system equipped with tools (file system access, shell execution, web search, etc.) that can operate autonomously to complete programming tasks. Think of it as a junior engineer who can take a high-level instruction like "add OAuth login to this app" and go off and do it without you watching every keystroke.

Three concepts tie together everything in this module:

- **Tool use / function calling**: The mechanism by which an agent triggers external actions (run a test, read a file, call an API).
- **Multi-agent patterns**: Architectures where multiple agents coordinate to solve problems larger than any single agent can handle.
- **State management**: How agents remember what they've done, what they're working on, and what got dropped when context overflows.

MCP (Model Context Protocol) is the emerging standard for connecting agents to tools. It provides a consistent interface so you don't have to write custom integration code every time you want an agent to access a new resource. MCP is to agents what USB-C is to hardware — a standard plug, so you can swap components without rewiring.

## Why it matters for vibe coding

Without a grounding in agentic workflows, you will hit three failure modes constantly:

1. **The agent does something surprising and you don't know why.** Agents operate with a world model built from their context. If you don't understand how context window size and truncation work, you cannot predict what the agent forgot or what assumption it made mid-task.

2. **You hand off a task that requires judgment, and the agent makes the wrong call silently.** An agent told "upgrade our auth system" might delete your user table and recreate it. Without understanding checkpointing and rollback, you have no safety net.

3. **Multi-step tasks degrade as context grows.** The agent starts strong on step 1, starts step 2 okay, and by step 5 is reasoning about files it read in step 1 that got evicted from context. The output looks plausible but is subtly broken. You won't catch it without knowing about context window management.

The deeper point: vibe coding depends on delegation. You cannot vibe code if you are manually reviewing every action an agent takes. You need to develop calibrated trust — knowing when an agent can handle something end-to-end and when you need to stay hands-on.

## The 20% you need to know

### What an autonomous coding agent actually does

An agent operates in a loop:

1. **Observe** — read its context (conversation history, files, tool results, instructions).
2. **Plan** — decide the next action.
3. **Act** — call a tool or write output.
4. **Evaluate** — check if the goal is met, or if something went wrong.
5. **Iterate** — repeat until done or blocked.

When people say "the agent lost track," what usually happened is that the observe step got incomplete data — either because the context window filled up and older information was dropped, or because the agent's reasoning drifted and it forgot what it was optimizing for.

### Tool use and function calling

Function calling is the API mechanism. You define a schema (name, parameters, description) and the model decides when to invoke it. The tool does the work and returns results to the agent's context.

MCP standardizes this interface. An MCP server exposes tools as resources, prompts, or capabilities that an MCP client (the agent runtime) can query and invoke. Instead of hardcoding "this agent can run shell commands," you configure an MCP server that defines shell execution as a tool. The agent runtime connects via MCP and sees the tool as available.

**What you need to know about tools:**
- Tools are only as reliable as their output. A tool that returns malformed JSON will confuse the agent.
- Tools can be chained — agent calls tool A, result feeds into tool B, result feeds into tool C. Long chains are fragile; each hop is an opportunity for the agent to misinterpret.
- Some tools have side effects (deleting files, sending emails, deploying code). Agents will call them if the task seems to call for them. You need to know what your agent's toolset includes.

### Multi-agent patterns

The two patterns you will encounter most:

**Supervervisor / worker pattern.** A supervisor agent receives a high-level task, decomposes it into subtasks, and dispatches each subtask to a worker agent. Workers do their piece and report back. Supervisor synthesizes results and handles final output. This is how large tasks get handled — "write the backend," "write the tests," "write the frontend" are three workers, the supervisor is the glue.

**Hierarchical pattern.** Similar to supervisor/worker, but with multiple levels of supervisors, each responsible for a domain. A top-level supervisor might route "fix the performance regression" to a performance-specialist supervisor, which dispatches to workers that profile, patch, and test.

The key thing to understand: **communication between agents is mediated by messages, which live in context**. If the supervisor's context overflows, it can lose track of which workers are still running and what they reported. Design multi-agent systems knowing that context is the shared currency and it has a hard limit.

### Agent memory: files vs. vector DBs

Agents have two ways to remember things across sessions or long tasks:

**File-based memory** stores structured data (JSON, Markdown) on disk. The agent reads and writes files as part of its workflow. This is simple, inspectable, and works for anything you can serialize. The downside: the agent has to explicitly read relevant memory before acting, and there's no semantic search — it knows the file exists, but doesn't have a way to find "the file about auth bugs from last week" without reading everything.

**Vector database memory** stores embeddings of text — numerical representations of meaning. When you query a vector DB, you get semantically similar content, not exact matches. This is powerful for "find everything related to auth failures" but adds infrastructure complexity, and the embedding quality determines whether retrieval actually works.

For vibe coding purposes: file-based memory is almost always sufficient for personal projects and one-off tasks. You write a `agent-memory.md` file, and your agent reads it at the start of a session. Vector DBs matter when you have a team or a large codebase where searching semantically is genuinely useful — not for most individual work.

### Context window management

The context window is how much the model can see at once. Every tool result, file read, conversation message, and instruction you give lives in the context window. When it fills up, older content is dropped — usually from the middle of the conversation history.

**What fits in a typical 128K–200K token context:**
- 10–15 medium-sized files (a few hundred lines each)
- Several tool call results
- A decent conversation history

**What gets dropped first:**
- Older conversation turns
- File contents that were read but are no longer relevant
- Tool results from intermediate steps

**The practical consequence:** for long tasks, design your agent's workflow to work in stages with explicit checkpointing between them. Don't expect a single agent invocation to handle 50 steps of reasoning without losing track of early steps. If you need long-horizon coherence, structure the task so the agent writes progress to a file and reads it back at the start of each new stage.

### Checkpointing strategies

Checkpointing is how you prevent lost work when an agent session ends or context overflows. Simple approaches work best:

1. **Task log file.** The agent writes every significant action to a `task-log.md` — "Step 1: read config.yaml. Step 2: identified missing env var. Step 3: added default." At the start of each new session, the agent reads this log before doing anything else.

2. **Artifact files.** For code generation work, write each major artifact to disk before moving on. Don't hold generated code only in context — persist it so you can inspect it later and resume from it.

3. **Decision log.** Record why the agent made each major decision. "Why did I rename this function? Because the original name was generic and risked confusion with the auth handler." This is critical for review — you can audit the agent's reasoning without re-running everything.

4. **Explicit suspend/resume.** At the end of a long task, write a summary: what was done, what's left, what the current state of the codebase is. A new agent session reads this and resumes. This is the manual version of what a good autonomous agent should do automatically, but don't assume the agent will do it — build it into your process.

## Hands-on exercise

**Build a two-agent supervisor system that solves a multi-step coding task.**

Time: ~15 minutes. You'll write two Python scripts: a supervisor agent and a worker agent, using a simple message-passing approach (no need for a full MCP setup or external service — this is a mental model exercise using plain files).

```
supervisor.py    — receives task, decomposes into subtasks, calls worker
worker.py       — accepts subtask, produces output, writes result to disk
task-state.md   — structured state file that passes context between agents
```

**Steps:**

1. Create a directory for this exercise.
2. Write `worker.py` that:
   - Accepts a task description from a JSON file (`input.json`)
   - Performs a simple code transformation (e.g., reads a source file, adds docstrings, writes the result)
   - Writes its output to `output.txt` and a log entry to `task-state.md`
3. Write `supervisor.py` that:
   - Reads a `mission.txt` file containing a high-level request
   - Decomposes it into two subtasks and writes them to `input.json` (one at a time, in sequence)
   - After each worker run, reads `task-state.md` to see what happened
   - Produces a final report in `final-report.md`
4. Create a sample `mission.txt` (e.g., "Add docstrings to all functions in sample.py and count how many were added")
5. Create a simple `sample.py` with 5–6 functions
6. Run `supervisor.py` and verify `final-report.md` contains the expected output

This exercise demonstrates the supervisor/worker pattern, explicit state management via files, and the concept of a task that is decomposed, executed in stages, and summarized — the core loop of multi-agent workflows. You can extend it by adding a third worker for a different subtask or by making the workers write richer checkpoint entries.

## Common mistakes

**1. Delegating high-stakes work without understanding what the agent will actually do.**
You tell an agent to "migrate the database to a new schema" and it drops the old tables before creating the new ones. You have no rollback because you didn't ask the agent to checkpoint first. The fix: for any destructive or irreversible action, specify the expected state before and after, require a checkpoint before proceeding, and review the checkpoint entry before letting the agent continue.

**2. Not realizing context has been truncated mid-task.**
The agent starts working on a task, context fills up, earlier conversation history gets dropped. The agent is now acting without knowledge of the constraints you gave it 20 minutes ago. The fix: write critical constraints and requirements to a file the agent can re-read. Don't rely on conversation history for anything that must persist across context overflow.

**3. Leaving multi-step tasks without a checkpointing structure.**
You kick off a 15-step refactor, session expires at step 9, and you have no idea what the agent actually completed. You either re-run and waste time, or try to manually finish and risk inconsistency. The fix: structure long tasks as a series of stage-completed artifacts. After each stage, the agent writes what was done and what remains to `TASK_STATUS.md`. Before starting the next stage, it reads that file.

**4. Not auditing agent decisions — treating output as correct because it looks plausible.**
The agent generates code that passes a linter but implements the wrong business logic. You deploy it because it "looks fine." The fix: always have a human-in-the-loop review for anything that affects user data, billing, security, or core business logic. Agent output needs human judgment on correctness, not just syntax.

**5. Running agents without knowing what tools they have access to.**
You assume the agent can only read files and run tests. It can actually delete files and push to git because those tools are wired up to the runtime you gave it access to. The fix: explicitly enumerate the toolset available to any agent you configure. Treat tool access like filesystem permissions — grant only what the task requires.

## AI-specific pitfalls

**Agents can hallucinate tool behavior.** An agent may assume a tool does something it doesn't — that a "database migration" tool will back up data before migrating, or that a "deploy" tool will run a smoke test first. When an AI generates code that calls a tool, it may describe the tool's behavior based on its training data rather than the actual tool specification. Always verify the actual tool API or schema before accepting the agent's assumptions about what it does.

**Agents lose coherence on long, multi-stage tasks.** Even with checkpointing, a single agent session can degrade in quality after many tool calls. The agent starts optimizing for the immediate step rather than the overall goal. Watch for this: if a task requires more than 10–15 significant tool interactions, consider whether the task should be split across agent sessions or handed to a supervisor/worker structure rather than one monolithic agent.

**Agent-written code may not match your codebase conventions.** An agent generating a new module may use patterns from its training data that don't match your project's style. Before integrating agent-generated code, review it against your existing codebase. A quick sanity check: does the new code import from the same places as your existing code? Does it follow the same naming conventions? Agents are inconsistent about this.

**Multi-agent systems amplify misalignment risk.** In a supervisor/worker setup, each agent may interpret the same instruction slightly differently. Over several handoffs, the final output can be far from what was intended. Keep multi-agent chains short. If you need more than two levels of hierarchy, the task decomposition is probably too coarse — break it into a different structure rather than adding depth.

**Context window overflow is silent.** The agent will keep operating normally after its context truncates — it won't tell you "I just forgot everything from the first hour of this task." The only defense is architectural: design your workflows to be robust to context loss by writing state to disk and having agents read it back at logical breakpoints.

## Quick reference

| Concept | What it is | Key concern |
|---|---|---|
| Function calling | Model triggers a defined tool | Tool schema must be accurate |
| MCP (Model Context Protocol) | Standard interface for agent-tool connections | Not all tools support MCP yet |
| Supervisor/worker | One agent decomposes and delegates | Context shared between agents has limits |
| File-based memory | State stored in disk files | Agent must explicitly read to recall |
| Vector DB memory | Semantic search over embeddings | Infrastructure cost, embedding quality |
| Context window | Max tokens model can see at once | Older content drops when full |
| Checkpointing | Persisting task state to survive interruptions | Must be explicit — agents don't auto-checkpoint |
| Context overflow | Context window fills and older content is dropped | Silent failure — agent doesn't signal it |

**Decision tree: should I delegate this to an agent?**

- Is the task reversible? If yes: delegate freely. If no: require checkpointing and human review before irreversible steps.
- Does it require knowledge of my specific codebase? If yes: provide the agent with the relevant files in context, don't assume it knows your project.
- Is it a single, bounded task (add feature X, fix bug Y)? If yes: a single agent can handle it. Is it open-ended (research our architecture, figure out why this is slow)? If yes: stay hands-on or use a tightly-scoped supervisor/worker setup.
- Does the task have side effects (delete, deploy, send, write to a shared resource)? If yes: explicitly scope the agent's tool access and require confirmation before destructive actions.

## Go deeper

- **MCP Official Documentation** (Model Context Protocol) — https://modelcontextprotocol.io — covers the protocol spec, server implementation, and client integrations. (verified 2026-04)
- **Anthropic's Agent SDK documentation** — covers tool use, function calling patterns, and multi-agent orchestration patterns. https://docs.anthropic.com/en/docs/claude-code — (verified 2026-04)
- **"Building Effective Agents"** — Anthropic's practical guide to agent design, covering context management, checkpointing, and failure modes. https://docs.anthropic.com/en/docs/build-agentic-systems — (verified 2026-04)
- **"Multi-Agent Architectures for LLM Systems"** — an accessible breakdown of supervisor/worker, hierarchical, and peer-to-peer agent patterns with practical implementation notes. https://www.anthropic.com/research — (verified 2026-04)
- **"Tool Use and Function Calling"** — Anthropic's conceptual guide to how models interact with external tools, including schema design best practices. https://docs.anthropic.com/en/docs/tool-use — (verified 2026-04)