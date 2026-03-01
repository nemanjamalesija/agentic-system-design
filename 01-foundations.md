# Chapter 1: Foundations

## What an Agentic System Actually Is

An agentic system is not "an AI that does things." It's a **coordination layer** that routes work to specialized AI agents, manages state across their interactions, and enforces domain rules they must follow. The AI is a component, not the system.

Think of it like a restaurant kitchen. The AI models are the cooks. The agentic system is everything else — the ticket system, the station layout, the recipes, the quality checks, the expediter who makes sure dishes leave the pass correctly. A great cook in a disorganized kitchen produces inconsistent food. An average cook in a well-organized kitchen produces reliable results.

## The Three Pillars

Every agentic system rests on three pillars:

### 1. Specialization

General-purpose agents produce mediocre results. Specialized agents produce excellent results within their domain. The moment you give an agent a clear, narrow job — "classify this ticket and scan for relevant patterns" rather than "help me build this feature" — its output quality jumps dramatically.

**Why specialization works:**
- Smaller scope = more focused prompt = more predictable output
- Each agent can use the optimal model for its task (reasoning-heavy tasks get Opus, execution tasks get Sonnet)
- Failures are isolated to one agent, not the entire pipeline
- Each agent can be iterated on independently

**The specialization test:** If you can't describe what an agent does in one sentence, it's doing too much.

### 2. Contracts

Agents communicate through artifacts — files with defined structures. EXPLORATION.md, PLAN.md, TASK.md. These are contracts between agents. The Explorer promises to produce a classification and pattern variance table. The Planner depends on that structure existing.

**Why contracts matter:**
- They make handoffs verifiable (does the file exist? does it have the right sections?)
- They decouple agents (the Explorer doesn't know or care what the Planner does)
- They make the system debuggable (you can read the artifact to understand what went wrong)
- They survive session drops (artifacts are files on disk, not in-memory state)

**The contract test:** If Agent B fails because Agent A's output was malformed, you have a contract problem, not an AI problem.

### 3. Developer Control

Agentic systems that work autonomously might seem impressive in demos but fail in production. Real developers need to understand what the system is doing, approve its decisions, and course-correct when it's wrong. The system should be collaborative, not autonomous.

**Why developer control matters:**
- Trust is earned incrementally, not assumed
- Mistakes in collaborative mode feel like "we both missed that" — mistakes in autonomous mode feel like "I can't trust this tool"
- Developer approval checkpoints prevent cascading errors
- The developer has context the AI doesn't (team conventions, business constraints, deadline pressure)

**The control test:** If a developer can't explain what the system just did, the system is too opaque.

## When to Build an Agentic System

**Build one when:**
- The same multi-step workflow repeats across many tasks (explore → plan → execute → verify)
- Domain knowledge is rich and codifiable (conventions, patterns, wiring rules)
- Multiple people need to follow the same workflow consistently
- Context-building overhead is the bottleneck (developers spend more time explaining conventions to AI than doing work)

**Don't build one when:**
- The task is a one-off (just use the AI directly)
- The workflow changes every time (you can't systematize chaos)
- The domain knowledge is too tacit to codify (if you can't write it down, agents can't follow it)
- A simple script would do (don't use AI to automate deterministic work)

## The Complexity Trap

The most common failure in agentic system design is over-engineering. Systems that start with "what if we need..." instead of "what do we need now?" accumulate complexity faster than value.

**Signs you're over-engineering:**
- You're building configuration systems before you know what developers want to configure
- You have more agent types than workflow steps
- You're designing for hypothetical future requirements
- Your orchestration logic is more complex than the work being orchestrated

**The antidote:** Build the minimum system that delivers the core value. Ship it. Watch what breaks. Fix that. Repeat. Your v2 should be informed by v1's failures, not by v1's speculation about what v2 might need.

## Mental Model: The Assembly Line

The most useful mental model for an agentic system is an assembly line, not a chatbot:

```
Raw Material → Station 1 → QC Check → Station 2 → QC Check → Station 3 → QC Check → Finished Product
(ticket)     (explorer)  (validate)  (planner)  (validate)  (executor)  (verify)    (implemented code)
```

Each station:
- Has a single, defined responsibility
- Receives structured input from the previous station
- Produces structured output for the next station
- Is staffed by a specialist (agent with a focused prompt)
- Has a quality check before passing work downstream

Work flows in one direction. Quality gates prevent defects from propagating. If a station fails, you fix it at that station — you don't redesign the entire line.

## Key Vocabulary

| Term | Meaning |
|------|---------|
| **Agent** | A specialized AI worker with a focused prompt, running in its own context window |
| **Skill** | A developer-facing entry point (slash command) that triggers a workflow |
| **Workflow** | Orchestration logic that sequences agents, validates handoffs, handles errors |
| **Convention** | Codified domain knowledge that agents consume (rules, patterns, examples) |
| **Artifact** | A structured file produced by one agent and consumed by another |
| **Checkpoint** | A saved snapshot of progress that survives session drops |
| **Handoff** | The transfer of work from one agent to the next, via a shared artifact |
| **Deviation** | When an agent encounters something unexpected and must decide how to handle it |

---
*Next: [Chapter 2: Architecture](02-architecture.md)*
