# Chapter 7: Developer Experience

## Trust Is the Product

Your agentic system's technical architecture can be perfect. If developers don't trust it, they won't use it. Developer trust is not a feature — it's the product.

Trust is earned through:
- **Predictability:** The system does what the developer expects
- **Transparency:** The developer can see what the system is doing and why
- **Control:** The developer can intervene at any point
- **Safety:** Mistakes are recoverable, not catastrophic

## Collaborative vs Autonomous

This is the defining design decision for developer experience. Most agentic systems get it wrong by defaulting to autonomous.

### The Autonomous Temptation

Autonomous execution is impressive in demos:
> "Give it a ticket, come back in 5 minutes, code is written."

Autonomous execution is terrifying in production:
> "It modified 7 files, 2 of which I don't understand, one has a subtle bug that will take me 30 minutes to find."

### The Collaborative Model

Every significant decision involves the developer:

```
System: "Task 1: Add getData action to store.ts."
Developer: ✓ Approved

System: [implements]

System: "✓ Task 1 complete. Modified: store.ts. Checkpoint saved."
System: "Task 2: Add service handler to service.ts."
Developer: "Actually, use domain+action pattern, not page pattern."

System: [adjusts and implements]
```

The developer stays in the loop. They catch mistakes before they compound. They provide context the AI doesn't have. They feel ownership over the result.

### The Overhead Question

"But per-task approval is slow! The developer has to approve 5 times for a simple ticket!"

Do the math:
- 5 tasks × 10 seconds of reading + approving = 50 seconds of developer time
- 5 tasks × 0 seconds (autonomous) + 15 minutes debugging an unexpected result = 15 minutes of developer time

The collaborative model is faster in practice because it prevents the expensive failure mode: discovering problems after they've compounded across multiple tasks.

### When Autonomous Is OK

- **Tier 1 deviations:** Auto-fixing missing semicolons, adding obvious imports. These are mechanical, have one right answer, and aren't worth interrupting the developer.
- **Build verification:** Running `npm run build` doesn't need approval.
- **File existence checks:** The Verifier doesn't need to ask before reading files.

The principle: **automate the mechanical, collaborate on the meaningful.**

## Interaction Granularity

### Per-Task (Recommended for v1)

The executor presents its approach for each task, gets approval, implements, then shows a brief summary:

```
Approach: "I'll add a getData action to store.ts that calls apiClient
          with endpoint 'checkout'. Files: store.ts"
[Developer approves]
[Implementation]
Summary: "✓ Task 1 complete. Modified: store.ts. Checkpoint saved."
```

**Why per-task:**
- Matches the plan's task boundaries (natural granularity)
- Developer catches directional errors before they affect downstream tasks
- Each approval is a checkpoint (resume granularity)
- 4-6 interactions per ticket is manageable

### Per-File (Too Granular)

```
"I'll modify store.ts line 45-52. Approve?"
"I'll modify store.ts line 78-85. Approve?"
"I'll modify service.ts line 12-20. Approve?"
```

Developers will rubber-stamp after the third approval. That defeats the purpose.

### Per-Plan (Too Coarse)

```
"Here's my approach for all 5 tasks. Approve?"
[Implements everything]
"Done. Here's what changed."
```

Developer loses the ability to course-correct mid-execution. A wrong assumption in task 2 affects tasks 3-5 before the developer sees any output.

## The Approach Presentation

When the executor presents its approach before implementing, it should be a **confirmation checkpoint**, not a second planning step.

### High-Level Summary (Right)

```
"I'll add a getData action to store.ts and a service handler to
service.ts. Files: store.ts, service.ts."
```

1-3 sentences. Developer catches directional errors in 2 seconds. "Wrong store file" — caught. "Wrong handler name" — caught. "Right approach" — approved, move on.

### Pseudo-Code Preview (Wrong)

```
"I'll add this function:
  getData() {
    return apiClient.get({ endpoint: 'checkout' }).fetchData();
  }
And this handler:
  export async function fetchData(api) {
    return api.get('/checkout/data');
  }
Approve?"
```

The developer is now reviewing code twice — pseudo-code, then real code. They'll start skipping the pseudo-code review, which means it's pure overhead.

### Why The Plan Makes Previews Redundant

The developer already approved a detailed plan (PLAN.md) with exact file paths, convention rules, and code examples. The pre-task presentation is confirming they still agree with that specific task's approach, not re-presenting the details.

## Task Summaries

After each task, a brief summary gives the developer a running log:

```
✓ Task 1 complete. Modified: store.ts. Checkpoint saved.
✓ Task 2 complete. Modified: service.ts. 1 auto-fix: added missing import. Checkpoint saved.
✓ Task 3 complete. Modified: index.vue, store.ts. Checkpoint saved.
```

**Why summaries matter:**
- Developer sees progress accumulate (motivating)
- Deviations are surfaced in context ("1 auto-fix" — ok, makes sense)
- File modification history is visible (useful for review)
- One line per task — zero overhead

**Anti-pattern:** Silent progression — implement, checkpoint, immediately present next task. The developer loses track of what happened. After 5 tasks, they can't remember which files changed.

## Error Presentation

When things fail, presentation matters as much as content.

### Bad Error Presentation

```
Error: The agent encountered an issue while processing your request.
The EXPLORATION.md file could not be generated successfully.
Please try again or contact support.
```

Generic, unhelpful, doesn't tell the developer what to do.

### Good Error Presentation

```
[Explorer] BLOCKED: Target page 'product-detail' not found

Missing: Valid page directory in src/modules/
Suggestion: Did you mean 'product-list' or 'related-products'?

Options:
  (a) Retry with corrected page name
  (b) Provide target page manually
  (c) Abort (state saved, resume later)
```

Specific, actionable, gives the developer clear options.

### The Error Presentation Checklist

Every error shown to the developer should have:
1. **Which agent/step failed** — so they know where in the pipeline the problem occurred
2. **What went wrong** — specific, not generic
3. **What's missing or needed** — the gap between current state and expected state
4. **What to do about it** — concrete options, always including "abort safely"

## First Impressions

Your system will be judged in the first 3 minutes of use. Design the first-run experience deliberately.

### What Developers Notice First

1. **How fast does it start?** If the system takes 30 seconds to "set up" before doing anything visible, developers assume it's slow. Show progress immediately.

2. **How much do they have to configure?** If the first interaction is a configuration wizard, developers assume it's complex. Smart defaults, zero config for v1.

3. **Does it produce correct code?** If the first generated code has a convention violation, trust is damaged. Get the golden path perfect before shipping.

4. **Can they stop it?** If developers feel trapped in a workflow they can't exit, they won't start it. "Abort with state preserved" must be visible and reliable.

### The Demo Ticket

Before rolling out to the team, prepare one specific ticket that you know works perfectly end-to-end. Walk each early adopter through this ticket first. Their first experience should be the system at its best — not a random ticket that might hit an edge case.

## Progressive Disclosure

Don't show developers everything at once. Reveal complexity as they need it.

### Level 1: The Happy Path

`/agent-quick "Add price filter"` → system runs → code appears → verification passes.

The developer sees: a command, progress updates, and results. They don't see: agent handoffs, checkpoint saves, deviation processing.

### Level 2: Decision Points

When the system encounters a variant pattern or the developer rejects an approach, they see the decision-making layer: "Pattern varies across pages. Page A does X, Page B does Y. Which approach?"

### Level 3: Debugging

When things fail, developers see the full pipeline: which agent failed, what it produced, what was expected. `/agent-progress` shows checkpoint state, file modifications, deviations.

### Level 4: Customization

After using the system successfully, developers might want to customize: skip Explorer for tickets where they already know the pattern, run Verifier standalone on existing code, adjust convention strictness.

Design for Level 1 first. Everything else is discovered, not taught.

## Feedback Loops

### The Learnings File

`.agent/learnings.md` — a committed, team-shared file where developers record corrections:

```markdown
- [2026-03-05]: Explorer classified "add sorting" as modify-form-field, should be modify-filter
- [2026-03-08]: Planner generated separate service file for checkout, should reuse existing handlers
- [2026-03-12]: user-profile page now uses store factory pattern (integration.md variance table needs update)
```

Agents read this file at the start of planning and execution. Corrections compound — the same mistake doesn't repeat across developers or tickets.

### Why Manual Is Right for v1

You might be tempted to automate the learnings loop: have the system detect its own mistakes and self-correct. Don't. For v1:
- You don't know which mistakes are systematic vs one-off
- Auto-correction requires meta-reasoning about reasoning quality
- Developers need to validate corrections before they become permanent

Manual entries, reviewed by the team, committed to git. Simple, reliable, transparent.

---
*Next: [Chapter 8: Testing Agentic Systems](08-testing.md)*
