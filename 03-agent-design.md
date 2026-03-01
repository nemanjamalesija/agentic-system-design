# Chapter 3: Agent Design

## Agents Are System Prompts

An agent is not a chatbot. It's a system prompt that defines a specialized worker. The quality of the agent is the quality of the prompt. Everything else — model selection, context management, tool access — is secondary.

This means agent design IS prompt engineering. But it's a specific kind of prompt engineering: structured, repeatable, and optimized for reliable output rather than impressive demos.

## The Inline vs Reference Decision

The most consequential agent design decision: what knowledge goes directly into the prompt vs what the agent reads from files at runtime?

### The Boundary

**Inline** (baked into the prompt): knowledge the agent needs to make judgment calls.
- Classification logic (how to categorize inputs)
- Decision trees (what to do in situation X vs Y)
- Concrete examples (what good output looks like)
- Error handling rules (when to block, when to continue)
- Output format specifications

**Referenced** (agent reads at runtime): supplementary knowledge the agent uses as reference material.
- Convention details (full rule lists, style guides)
- Code patterns (existing codebase examples)
- Configuration (file paths, project-specific settings)

**The test:** Remove the knowledge from the prompt. Can the agent still make the right decisions? If no, it must be inline. If yes, it can be referenced.

### Why This Matters

**Too much inline** = prompt bloat. The agent spends tokens processing knowledge it may not need for this specific task. Prompt size limits become a constraint. Updating knowledge requires editing every agent prompt.

**Too much referenced** = unreliable behavior. The agent must successfully read 5 files before starting real work. Any file read failure degrades quality. The agent interprets referenced knowledge in its own context, which may differ from the prompt author's intent.

**The hybrid approach** gets this right:

```
Explorer prompt (inline):
  - 5 classification patterns with criteria     ← judgment logic
  - Scanning strategy per ticket type            ← decision tree
  - EXPLORATION.md output format                 ← output spec
  - ## BLOCKED error format                      ← error handling

Explorer prompt (referenced):
  - .ai/docs/architecture.md                      ← supplementary
  - .agent/conventions/frontend/integration.md    ← supplementary
```

## Consistent Agent Structure

Every agent should follow the same internal structure. Same sections, same order. This isn't about aesthetics — it's about maintainability, review quality, and prompt engineering consistency.

### The Standard Structure

```markdown
# Agent Name

## Purpose
One sentence. What this agent does.

## Inputs
What files/data the agent receives. What the orchestrator passes to it.

## Process
Numbered steps. The agent follows these in order.

## Inlined Knowledge
The judgment logic, decision trees, examples, and rules
that are baked into the prompt.

## Referenced Files
What to read at runtime, with purpose annotation.

## Output Format
Exact structure of the output file. Section names, field types,
what "complete" looks like.

## Error Handling
When to produce ## BLOCKED output. What constitutes an error
vs a deviation. Recovery expectations.
```

### Why Consistency Matters

1. **Maintainability.** When you update error handling across all agents, you go to the same section in each file.

2. **Review quality.** During prompt iteration, you compare agent outputs and trace problems back to the prompt. Same structure = same muscle memory.

3. **Prompt engineering consistency.** LLMs read structured prompts more reliably than freeform ones. A predictable frame (Purpose → Inputs → Process → ...) helps the model organize its execution.

4. **Onboarding.** A new developer reads one agent prompt and knows the layout of all four.

## Model Routing

Not all agents need the same model. Match the model to the cognitive demand:

| Cognitive Demand | Model Tier | Examples |
|-----------------|------------|----------|
| Deep reasoning, complex generation | Opus (highest) | Planning, architecture decisions, complex plan shapes |
| Pattern scanning, structured output | Sonnet (mid) | Exploration, classification, code generation |
| Rule checking, simple validation | Sonnet or Haiku | Verification, convention checks |

**The principle:** Use the cheapest model that reliably produces the required output quality. Don't use Opus for execution work — Sonnet follows structured instructions just as well. Don't use Haiku for planning — it lacks the reasoning depth for complex decisions.

**Example routing:**
```
Explorer:  Sonnet  (pattern scanning, classification)
Planner:   Opus    (deep reasoning about plan shapes, conventions)
Executor:  Sonnet  (follows plan-as-prompt, structured implementation)
Verifier:  Sonnet  (rule checking, file correlation)
```

The Planner gets Opus because it makes the most consequential decisions: which files to touch, which conventions apply, how to handle variant patterns. The Executor follows those decisions — it doesn't need to reason about them.

## Plan-as-Prompt

One of the most powerful patterns in agentic system design: **the plan IS the executor's prompt.**

Traditional approach:
```
Planner writes a plan document → Developer reads it → Developer translates it into instructions → Executor follows instructions
```

Plan-as-prompt approach:
```
Planner writes a plan-as-prompt → Executor reads it directly as its instruction set
```

No translation step. No information loss. The Planner knows the Executor will read this, so it writes accordingly — with exact file paths, inline convention rules, code examples, and acceptance criteria per task.

**Requirements for plan-as-prompt to work:**
- The plan must be detailed enough that the Executor doesn't need to make design decisions
- Convention rules must be inlined per task, not referenced generically
- Code examples from the target codebase must be included so the Executor matches existing patterns
- Variant decisions must be resolved before the plan is written

**Anti-pattern:** A plan that says "follow BEM naming conventions" without showing what that looks like in the target page's code. The Executor will interpret "BEM" in the abstract rather than matching the specific naming pattern the page uses.

## Show, Don't Tell

For any structured output requirement, **examples beat rules.**

**Rules-only approach (fragile):**
```
"Generate a modify-plan with tasks that target existing files.
Each task should specify the file path, the section to modify,
and the convention rules that apply."
```

**Example-based approach (reliable):**
```
"Generate a plan following this structure:

## Tasks

### Task 1: Add fetchData action to store
**File:** checkout/store.ts
**Section:** actions
**Conventions:** functional pattern only (code-style.md), API client route must match service endpoint name
**Example from target page:**
```ts
// checkout/store.ts:45-52
async fetchData() {
  return apiClient.get({ endpoint: 'checkout' }).fetchData();
}
```
**Acceptance:** New action follows existing pattern, endpoint name matches."
```

The model pattern-matches against the example. The output is structurally consistent. Two examples (one modify-plan, one create-plan) is enough — the model generalizes from concrete examples better than from abstract rules.

## Deviation Handling

Agents encounter unexpected situations. How they handle deviations defines the system's reliability.

### The Three-Tier Model

**Tier 1: Auto-fix (silent)**
The agent fixes the issue without mentioning it. Reserved for mechanical corrections that have exactly one right answer.
- Missing semicolons, syntax errors
- Missing imports that are unambiguous
- ESLint auto-fixable violations

**Tier 2: Notify-and-fix (transparent)**
The agent fixes the issue AND tells the developer what it did. For convention violations where the fix is clear but the developer should know.
- class component → functional component rewrite
- `@import` → `@use` in SCSS
- BEM naming violations

**Tier 3: Ask (collaborative)**
The agent stops and asks the developer. For decisions that affect architecture or could go multiple ways.
- New shared components needed
- Store shape changes
- API contract changes
- Pattern uncertainty (patterns that vary across pages)

**The default must be Ask.** When an agent isn't sure which tier applies, it should ask. Silent autonomous decisions on structure are how agentic systems lose developer trust.

### Operationalizing Deviation Rules

Abstract tier definitions aren't enough. The agent prompt needs **concrete examples per tier:**

```
## Deviation Handling

When you encounter something unexpected:

**Auto-fix silently:**
- Missing import → add it
- Syntax error → fix it
- ESLint violation → auto-fix

**Fix and notify developer:**
- class component code → rewrite to functional component, notify: "Rewrote to functional component per code-style.md"
- @import in SCSS → change to @use, notify: "Changed @import to @use per styling.md"

**Stop and ask developer:**
- Need a new shared component → ask: "This needs a new shared component in shared/components/. Approve?"
- Store shape change → ask: "This changes the store's public interface. Proceed?"
- Unsure which pattern to follow → ask: "Page A does X, Page B does Y. Which approach?"

**Default: If unsure, ask.**
```

The examples ARE the decision tree. They resolve the ambiguous boundaries between tiers through concrete cases rather than abstract rules.

## Agent Scope

Each agent should have a clear, testable scope:

**Too broad:** "Help developers implement frontend features"
**Too narrow:** "Add a fetchData call to store.js"
**Just right:** "Classify ticket type, scan target page, identify universal vs variant patterns, produce structured EXPLORATION.md"

**The scope test:**
1. Can you describe what the agent does in one sentence?
2. Can you define what "good output" looks like for this agent?
3. Can you test this agent in isolation (without running the full pipeline)?

If any answer is "no," the scope is wrong. Too broad → split. Too narrow → merge. Can't test in isolation → the agent depends on too many things.

## Error Output

Agents must fail gracefully. When an agent can't complete its work, it should produce structured error output — not conversational text.

```markdown
## BLOCKED
**Reason:** Target page 'product-detail' not found in src/modules/
**Missing:** Valid page directory
**Suggestion:** Did you mean 'product-list' or 'related-products'?
```

**Why structured errors matter:**
- The orchestrator can detect `## BLOCKED` and trigger recovery automatically
- The developer gets the *reason* surfaced in the workflow, not a generic "agent failed"
- The retry option can include the error context in the next agent invocation
- All agents use the same format — consistent mental model

**Anti-pattern:** Agents that report errors conversationally ("I couldn't find the page you mentioned. Can you check the page name?"). This works in a chatbot. In a pipeline, the orchestrator can't parse it, the error context is lost, and the developer has to read the raw agent output to understand what happened.

---
*Next: [Chapter 4: Multi-Agent Orchestration](04-orchestration.md)*
