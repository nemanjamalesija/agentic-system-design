# Chapter 5: Convention Systems

## Why Conventions Are the Foundation

Without conventions, agents improvise. They generate code that looks reasonable in isolation but doesn't match the codebase's existing patterns. They make architectural decisions that conflict with team standards. They solve problems in ways that create new problems.

Conventions solve this by codifying domain knowledge into artifacts that agents consume. They're the difference between "write a data store" (generic, inconsistent) and "write a data store following the team's preferred pattern with `apiClient` using the route key matching the service endpoint name" (specific, consistent).

## The Ownership Boundary

The single most important convention system decision: **which knowledge lives where?**

### The Problem

In a real codebase, domain knowledge exists in multiple places:
- Architecture docs (`.ai/frontend/architecture.md`)
- Code style guides (`.ai/frontend/code-style.md`)
- Framework docs (`.ai/docs/architecture.md`)
- Tribal knowledge (patterns that developers know but haven't written down)

An agentic system needs all of this knowledge. The naive approach is to copy it into convention files. This creates **knowledge drift** — when the source is updated, the copy becomes stale. Two developers get different answers depending on which file the agent reads.

### The Solution: Reference, Never Duplicate

Establish a clear ownership boundary:

```
.ai/frontend/          ← Source of truth for:
  architecture.md        - Application structure, decision trees
  code-style.md          - Naming, imports, linting
  styling.md             - BEM, SCSS patterns
  api.md                 - API conventions

.agent/conventions/frontend/   ← Source of truth for:
  integration.md         - How application layers wire together (page→store→service→API)
  deviation-rules.md     - When agents should auto-fix vs ask
  verification-rules.md  - Machine-actionable compliance checklist
```

The convention files own ONLY knowledge that the existing docs don't cover. They reference the existing docs by file path — never copy content.

### Source Attribution

Every piece of knowledge in a convention file should be attributed to its source:

```markdown
### Store Patterns

Stores must use the functional pattern.

> Source: .ai/docs/code-style.md — "All frontend code uses functional patterns"

The `apiClient` call in the store must use a `route` key that matches
the service endpoint's exported key.

> Source: Derived from cross-module analysis of all application modules
```

**Why attribution matters:**
- When the source changes, you know which convention entries to update
- When an agent's output violates a convention, you can trace back to the authoritative source
- When a developer questions a convention, you can point to the evidence

### The Ownership Test

Before adding knowledge to a convention file, ask:
1. Is this knowledge already documented somewhere else? → Reference it
2. Is this knowledge unique to the agentic system's needs? → Own it
3. Is this knowledge derivable from the codebase? → Document the derivation, own the result

## Convention File Design

### Be Machine-Actionable

Convention files are consumed by AI agents, not (primarily) by humans. They should be structured for reliable machine parsing:

**Bad (prose-heavy):**
```markdown
When writing stores, you should generally use the functional pattern
rather than the class-based pattern. This is important because our codebase
has standardized on functional patterns for consistency reasons.
```

**Good (rule-based):**
```markdown
### STO-01: Functional Pattern Only [blocking]
**What to check:** Store files use `createStore('name', { ... })` with options object
**Pass:** No class decorators, no inheritance chains, no mutable singletons
**Fail:** Any class-based patterns in store files
**Source:** .ai/docs/code-style.md
```

The second format gives the agent:
- A unique rule ID (for reporting)
- A severity level (blocking vs warning)
- Concrete pass/fail conditions (no interpretation needed)
- A source (for traceability)

### Categorize by Concern

Group rules by what they check, not by where they come from:

```
verification-rules.md:
  ## Imports (IMP-01, IMP-02, IMP-03)
  ## Naming (NAM-01, NAM-02, NAM-03)
  ## Store Patterns (STO-01, STO-02, STO-03)
  ## Wiring (WIR-01, WIR-02, WIR-03)
  ## Build (BLD-01, BLD-02)
```

This makes it easy for agents to find relevant rules (the Verifier checks wiring rules, the Executor references store patterns) and for developers to understand the scope of each category.

### Document Pattern Variance

In any non-trivial codebase, patterns vary across components. Some patterns are universal (followed by everyone), others are page-specific or team-specific. Convention files must distinguish between them.

**Universal pattern (safe to enforce):**
```markdown
### Route Definition
**Pattern:** Routes defined in web.i18n.yaml with page handler reference
**Universality:** ALL pages (11/11)
**Action:** Enforce automatically
```

**Variant pattern (must ask):**
```markdown
### Store Factory Function
**Pattern:** Store created via factory function returning defineStore
**Universality:** VARIANT — dashboard, checkout only (2/11)
**Action:** ASK developer which approach to follow
**Pages using variant:** dashboard/store.ts, checkout/store.ts
**Pages using standard:** product-list, search, + 7 others
```

**The golden rule:** If a pattern isn't followed by 100% of the codebase, agents must ask the developer which approach to follow. Never assume.

## The Pre-Computed Index Pattern

Some convention knowledge is expensive to derive but cheap to consume. Instead of having agents re-derive it every invocation, derive it once and store the result.

**Example:** Integration wiring patterns across 11 application modules.

**Without pre-computed index:** The Explorer scans all 11 modules every time, identifies universal vs variant patterns, builds a comparison table. Uses 60% of its context window. Takes 30+ seconds.

**With pre-computed index:** The `integration.md` convention file contains a pattern variance table derived from scanning all modules. The Explorer reads the table (~500 tokens), deep-scans only the target module, and references the table for cross-module context. Uses 15% of context. Takes 5 seconds.

**When to pre-compute:**
- The knowledge is derived from codebase analysis (not external sources)
- The derivation is expensive (scanning multiple files/directories)
- The result changes infrequently (codebase patterns don't shift daily)
- Multiple agents need the same knowledge

**When not to pre-compute:**
- The knowledge changes frequently (dependencies, API responses)
- Only one agent needs it (just let that agent derive it)
- The derivation is cheap (reading a single config file)

### Maintaining Pre-Computed Knowledge

Pre-computed indexes can go stale. Address this through:

1. **Learnings file:** A team-shared file (`.agent/learnings.md`) where developers record corrections. Agents read this at the start of planning and execution. "The integration.md table says store factory is only dashboard + checkout, but user-profile also uses it now."

2. **Periodic re-derivation:** As part of major updates, re-scan the codebase and update the convention file. Not every sprint — only when significant changes accumulate.

3. **Trust but verify:** Agents trust the pre-computed index for general context but verify the target page directly. If the target page contradicts the index, the agent flags the discrepancy.

## Convention Layering for Multi-Stack Systems

When your system supports multiple tech stacks, conventions should be scoped by stack:

```
.agent/conventions/
  frontend/              ← Frontend conventions
    integration.md
    deviation-rules.md
    verification-rules.md
  backend/               ← Backend conventions (future)
    integration.md
    deviation-rules.md
    verification-rules.md
  mobile/                ← Mobile conventions (future)
    integration.md
    deviation-rules.md
    verification-rules.md
```

**Same file names, different directories.** A new stack = create a directory and populate it. No renaming, no prefixes, no namespace conflicts.

**The extensibility test:** Can you add support for a new stack by adding a directory? If yes, your convention system scales. If you'd need to rename existing files or restructure, fix that now.

## Convention Anti-Patterns

### The Knowledge Dump

A convention file that tries to document everything about a topic. 2000 lines of frontend architecture, code patterns, style rules, and historical decisions. No agent will reliably extract the relevant subset.

**Fix:** Split by concern. Integration wiring is one file. Deviation rules is another. Verification checklist is a third. Each file has a focused scope.

### The Stale Convention

A convention file written during system design that's never been updated. The codebase has evolved but the conventions reflect the state from 6 months ago.

**Fix:** Source attribution + learnings file. When a developer notices a stale rule, they update the convention or add a learning entry. Agents read learnings at runtime.

### The Duplicated Truth

Convention knowledge that exists in two places — both the original doc and the convention file. One gets updated, the other doesn't. Agents get conflicting instructions.

**Fix:** Reference, never duplicate. Convention files point to source docs by file path. If the source changes, the pointer still works.

### The Unenforced Convention

A convention rule that's documented but never checked. Developers learn to ignore it because nothing happens when they violate it.

**Fix:** Verification rules with pass/fail conditions. If a rule matters, the Verifier checks it. If it doesn't matter enough to check, delete it from the conventions.

---
*Next: [Chapter 6: State Management](06-state-management.md)*
