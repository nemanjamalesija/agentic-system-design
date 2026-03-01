# Chapter 4: Multi-Agent Orchestration

## The Orchestrator's Job

The orchestrator (workflow) has exactly three responsibilities:
1. **Sequence** agents in the right order
2. **Validate** handoffs between agents
3. **Handle** errors when things go wrong

That's it. The orchestrator does not read full agent outputs. It does not transform data between agents. It does not make domain decisions. It is a traffic controller, not a participant.

## Context-Lean Orchestration

This is the single most important orchestration principle, and the one most systems get wrong.

### The Problem

Each agent gets a fresh context window (e.g., 200k tokens). The orchestrator accumulates context across ALL agent invocations. If the orchestrator reads Agent A's output (3k tokens) before spawning Agent B, then reads Agent B's output (5k tokens) before spawning Agent C, the orchestrator has consumed 8k+ tokens on content that the next agent will read anyway.

In a 5-agent pipeline, a non-lean orchestrator can consume 15-20k tokens of accumulated context — context that could have been used for error recovery, developer interaction, or additional workflow steps.

### The Solution

**Rule: The orchestrator validates file existence and reads checkpoint summaries. Never full file contents.**

```
Agent A completes:
  ✗ Orchestrator reads EXPLORATION.md into its context
  ✓ Orchestrator checks: does EXPLORATION.md exist? Is it non-empty? Has required sections?

Agent B starts:
  ✗ Orchestrator passes EXPLORATION.md contents to Agent B
  ✓ Agent B reads EXPLORATION.md itself, in its own fresh context
```

**Context budget per step:** Target < 5k orchestrator tokens. This includes:
- Handoff validation results (~200 tokens)
- Checkpoint summary reading (~100 tokens)
- Error context if applicable (~500 tokens)
- Developer interaction (~2k tokens)

If you're exceeding 5k per step, you're reading content that belongs in an agent's context, not the orchestrator's.

### How Agents Pass Data

Agents pass data through artifacts on disk, not through the orchestrator:

```
Explorer writes .nfs-state/<slug>/EXPLORATION.md
  → Orchestrator checks: file exists, has ## Classification section
  → Planner reads EXPLORATION.md in its own fresh context

Planner writes .nfs-state/<slug>/PLAN.md
  → Orchestrator checks: file exists, has ## Tasks section
  → Executor reads PLAN.md in its own fresh context
```

The file system is the message bus. Artifacts are the messages. The orchestrator is the router.

## Handoff Validation

Every agent-to-agent transition is a potential failure point. Handoff validation catches failures before they cascade.

### The Validation Protocol

After each agent completes:

1. **CHECK:** Expected output file exists and is non-empty
2. **CHECK:** Required sections present (structural validation via heading search)
3. **IF VALID:** Save checkpoint, proceed to next agent
4. **IF INVALID:** Present to developer with 3 recovery options

### Concrete Validations

Define specific validations for each handoff in your pipeline:

| Handoff | File | Required Sections | Blocking Check |
|---------|------|-------------------|----------------|
| Explorer → Planner | EXPLORATION.md | `## Classification`, `## Pattern Comparison Table` | Classification present AND patterns documented |
| Planner → Executor | PLAN.md | `## Objective`, `## Tasks` | At least 1 task with file target |
| Executor → Verifier | TASK.md checkpoint | `filesModified` array | At least 1 file modified |

**Structural validation, not semantic validation.** The orchestrator checks that the right sections exist — not that their content is correct. Semantic quality is the next agent's problem (the Planner will fail fast if the Explorer's classification is garbage). The orchestrator just ensures the artifact is well-formed enough to be consumed.

### Why Structural Is Enough

You might think: "Shouldn't we check that the classification is actually one of the 5 valid types?" You could, but you're adding classification logic to the orchestrator — logic that belongs in the Explorer and the Planner. The Planner will reject an invalid classification. The orchestrator's job is to make sure there's a classification to reject.

**The validation depth test:** If your handoff validation requires understanding the domain, it's too deep. Validation should be doable with file existence checks and heading searches.

## Error Recovery

Every pipeline step can fail. The recovery protocol must be:
1. **Predictable** — developers always see the same three options
2. **State-preserving** — no work is lost regardless of choice
3. **Developer-controlled** — no autonomous retries or rollbacks

### The Three-Option Pattern

When any step fails, present:

```
[Step N] failed: <concise reason>

Options:
  (a) Retry — re-run this step with error context appended
  (b) Manual — provide the expected output yourself
  (c) Abort — save current state, resume later
```

This pattern is powerful because it's **consistent.** Whether the Explorer can't find the target page or the Executor's build fails, the developer sees the same three options. No new patterns to learn, no context-specific recovery flows.

### Recovery Details

**Retry:** Re-spawn the same agent with the error context added to its prompt. "Your previous run produced an EXPLORATION.md missing the ## Classification section. Here was the error: ..." The agent gets a second chance with more information.

**Manual:** The developer provides the expected artifact themselves. This is the escape hatch for when the agent fundamentally can't do the work (e.g., the ticket requires domain knowledge the agent doesn't have). The workflow skips the agent and uses the developer's artifact for the next step.

**Abort:** Save the checkpoint. The developer can resume later with `/nfs-pb-execute` or hand off to a colleague. All completed steps are preserved. This is the safe option — and the most important one. Developers are far more willing to try agentic workflows when they know "abort" doesn't lose their work.

### Error Context Propagation

When retrying, include the error context in the agent's re-invocation:

```
## Previous Attempt
The previous Explorer run produced a BLOCKED result:
- Reason: Target page 'listing-detail' not found
- Suggestion: Did you mean 'browse-listings'?

The developer has confirmed the target page is 'browse-listings'.
Please re-run with this corrected information.
```

This is one of the few cases where the orchestrator adds substantive content to an agent's prompt. It's acceptable because error context is small (< 500 tokens) and directly relevant.

## Pipeline Patterns

### Linear Pipeline

The simplest and most common pattern:

```
A → B → C → D
```

Each agent depends on the previous one's output. No branching, no parallelism. Good for workflows where each step informs the next.

**When to use:** Most v1 workflows. Explore → Plan → Execute → Verify is a natural linear pipeline.

### Fan-Out / Fan-In

When multiple agents can work in parallel:

```
    → B₁ →
A →       → D
    → B₂ →
```

Agent A's output is consumed by B₁ and B₂ independently. Agent D consumes both outputs.

**When to use:** When two substeps are independent. Example: a verifier that checks conventions AND runs the build could theoretically fan out.

**Warning:** Parallelism adds complexity to handoff validation (wait for both B₁ and B₂) and error recovery (what if B₁ succeeds but B₂ fails?). Don't parallelize until you've proven the linear pipeline works.

### Conditional Pipeline

When the next step depends on the current step's output:

```
A → [if type X] → B₁ → C
    [if type Y] → B₂ → C
```

**When to use:** When different input types need different processing. Example: modify-tickets vs create-tickets could theoretically route to different planners.

**Warning:** In practice, conditional routing adds orchestrator complexity. A single Planner that handles both plan shapes (with examples for each) is simpler than two specialized planners with a router.

## Orchestration Anti-Patterns

### The God Orchestrator

An orchestrator that reads all agent outputs, makes decisions based on them, and passes curated summaries to the next agent. It becomes a bottleneck, accumulates context, and is impossible to debug because the data flow goes through a single point.

**Fix:** Let agents read each other's output directly. The orchestrator validates structure, not content.

### The Retry Loop

An orchestrator that silently retries failed agents 3-5 times before surfacing the error. This wastes tokens, delays feedback, and masks systematic prompt issues.

**Fix:** Fail fast, surface to developer, let them choose. If an agent fails repeatedly, the prompt needs iteration, not retries.

### The Data Transformer

An orchestrator that reads Agent A's output, transforms it (filtering, reformatting, summarizing), and passes the transformed version to Agent B. The transformation logic is hard to test, hard to debug, and splits the "understanding" of the data across two places.

**Fix:** Agent B should read Agent A's raw output. If Agent A's output format doesn't suit Agent B, fix Agent A's output format — don't add a translation layer.

### The Monolith Workflow

A single workflow file that handles all possible flows (quick, plan-only, execute-only, verify-only). The conditional logic becomes unreadable.

**Fix:** Separate workflow files for distinct flows. Two simple workflows beat one complex one.

---
*Next: [Chapter 5: Convention Systems](05-convention-systems.md)*
