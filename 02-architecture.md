# Chapter 2: Architecture

## The Layered Stack

A production agentic system has five distinct layers. Each layer has a single responsibility. Mixing responsibilities across layers is the fastest path to an unmaintainable system.

```
┌─────────────────────────────────┐
│  Skills (Entry Points)          │  Developer-facing commands
├─────────────────────────────────┤
│  Workflows (Orchestration)      │  Agent sequencing, handoff validation
├─────────────────────────────────┤
│  Agents (Workers)               │  Specialized AI with focused prompts
├─────────────────────────────────┤
│  Conventions (Knowledge)        │  Domain rules, patterns, examples
├─────────────────────────────────┤
│  State (Persistence)            │  Checkpoints, artifacts, progress
└─────────────────────────────────┘
```

### Layer 1: Skills

Skills are the surface area of your system. They're what developers invoke — `/nfs-pb-quick`, `/nfs-pb-plan`, `/nfs-pb-verify`. A skill parses arguments, selects a workflow (or orchestrates inline for simple flows), and returns results to the developer.

**Design rules:**
- Every skill must be explicitly invoked. No auto-triggering. Developers should never wonder "why did this just run?"
- Skill names form a namespace. Use prefixes to group related skills (`/nfs-pb-*`, `/nfs-vue-*`).
- Simple skills (1-2 agent calls) can orchestrate inline. Complex skills (3+ agents, handoff validation) delegate to a workflow file.
- Skills own the developer interaction — they present results, collect approvals, offer recovery options.

**Anti-pattern:** Skills that do real work. A skill should dispatch to agents, not implement logic.

### Layer 2: Workflows

Workflows sequence agents, validate handoffs between them, and handle errors. They are the "glue" layer — the most important and most underestimated layer in the system.

**Design rules:**
- Workflows are context-lean. They never read full agent output files. They check file existence, read checkpoint summaries, validate structural requirements, and spawn the next agent.
- Every handoff has validation: expected file exists, required sections present, and content is non-empty.
- Every failure has three recovery options: retry with error context, developer provides manual input, or abort with state preserved.
- Workflows track < 5k tokens of orchestration state per step. If you're reading full artifacts into the workflow, the design is wrong.

**Why context-lean matters:**

Each agent gets a fresh context window (200k tokens). But the workflow accumulates context across all steps. If the workflow reads Explorer output (maybe 3k tokens) + Planner output (5k tokens) + Executor state (2k tokens), it's burning 10k+ tokens on content that the *next* agent will read anyway in its own fresh window.

The correct data flow:
```
Explorer writes EXPLORATION.md → workflow checks existence → Planner reads it (fresh context)
Planner writes PLAN.md → workflow checks existence → Executor reads it (fresh context)
Executor updates TASK.md → workflow reads checkpoint summary → Verifier reads it (fresh context)
```

The workflow validates and routes. Agents read and write.

**Anti-pattern:** Workflows that transform agent output before passing it to the next agent. Each agent should read its predecessor's raw output. Transformation means the workflow is doing work that belongs in an agent.

### Layer 3: Agents

Agents are specialized workers. Each runs in its own context window, reads conventions, processes input, and produces structured output. They are the core of the system.

Covered in depth in [Chapter 3: Agent Design](03-agent-design.md).

### Layer 4: Conventions

Conventions are codified domain knowledge. They define the rules, patterns, and examples that agents follow. Without conventions, agents improvise — and improvisation in a production codebase is a bug.

Covered in depth in [Chapter 5: Convention Systems](05-convention-systems.md).

### Layer 5: State

State is everything the system remembers across steps and sessions. Checkpoints, artifacts, progress markers. State must survive session drops, crashes, and developer context switches.

Covered in depth in [Chapter 6: State Management](06-state-management.md).

## Separation of Concerns

The most critical architectural principle: **each layer should only know about its immediate neighbors.**

```
Skills know about: Workflows, State (to present progress)
Workflows know about: Agents (to spawn them), State (to validate)
Agents know about: Conventions (to follow them), State (to read/write artifacts)
Conventions know about: Nothing (they're static knowledge)
State knows about: Nothing (it's passive storage)
```

**Violations to watch for:**
- Skills that read convention files directly (should go through agents)
- Agents that spawn other agents (should go through workflows)
- Workflows that modify conventions (should be manual updates)
- State that contains business logic (should be in conventions)

## The Pipeline Pattern

Most agentic workflows follow a pipeline pattern:

```
Input → Agent A → Artifact A → Agent B → Artifact B → Agent C → Output
```

Each agent transforms input into output. Artifacts are the handoff points. The pipeline is linear — no branching, no cycles, no conditional agent selection (in v1).

**Pipeline vs. Graph:**

Some systems use graph-based orchestration where agents can call each other dynamically. This is almost always a mistake in v1:
- Graphs are hard to debug (which path did it take?)
- Graphs are hard to test (exponential path combinations)
- Graphs accumulate context unpredictably
- Graphs make error recovery complex (where do you retry from?)

Start with a pipeline. Graduate to a graph only when the pipeline's limitations are proven, not theoretical.

## File System as Infrastructure

In platform-native agentic systems (built on Claude Code, Cursor, etc.), the file system IS your infrastructure:

- **Agents** are markdown files (`.claude/agents/*.md`)
- **Skills** are markdown files (`.claude/skills/*/SKILL.md`)
- **Conventions** are markdown files (`.nfs/conventions/*.md`)
- **Artifacts** are markdown files (`.nfs-state/<slug>/*.md`)
- **State** is JSON embedded in markdown (checkpoint sections)
- **Configuration** is JSON files

No databases. No message queues. No custom runtimes. Files are:
- Version-controllable (git tracks everything)
- Human-readable (developers can inspect any artifact)
- Debuggable (open the file, read it)
- Portable (copy the directory, the system moves with it)

**The file system test:** Can a developer understand the system's current state by reading files on disk? If yes, your architecture is transparent. If no, you've built a black box.

## Directory Structure as Architecture

Your directory structure should mirror your architectural layers:

```
.claude/
  skills/          # Layer 1: Entry points
  agents/          # Layer 3: Workers
.nfs/
  workflows/       # Layer 2: Orchestration
  conventions/     # Layer 4: Knowledge
    pb/            # Stack-scoped
    vue/           # Stack-scoped
  templates/       # Artifact scaffolds
  bin/             # CLI tooling
.nfs-state/        # Layer 5: State (gitignored)
  <slug>/          # Per-task state
```

**Principles:**
- Committed to git: skills, agents, workflows, conventions, templates, CLI tools
- Gitignored: per-task runtime state (artifacts, checkpoints)
- Namespaced: agents use `nfs-` prefix, skills use `nfs-pb-*` namespace, conventions use stack directories

**The onboarding test:** Can a new developer understand the system by looking at the directory structure? Each directory should answer "what lives here?" without opening any files.

## Architecture Decision Records

Every architectural decision should be documented with rationale and outcome. Not in a separate wiki — in the project itself, next to the code.

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| 4 agents, not 6 | Merged explorer+synthesizer, dropped plan-checker | Fewer handoffs, simpler pipeline |
| Pipeline, not graph | Debuggability and testability over flexibility | Can upgrade to graph in v2 if needed |
| File system, not database | Transparency, portability, zero infrastructure | Scales to team of 10, may not scale to 100 |

Every decision you don't document is a decision a future developer will reverse without understanding why it was made.

---
*Next: [Chapter 3: Agent Design](03-agent-design.md)*
