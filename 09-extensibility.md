# Chapter 9: Extensibility

## The Extensibility Paradox

You must design for extensibility without building for it. The moment you start implementing features for hypothetical future requirements, you're over-engineering. But the moment you make a naming decision that boxes you into one stack, you're creating technical debt.

The goal: **make the right structural decisions now so that future work is additive, not refactoring.**

## Structural Decisions That Matter

### Directory Structure Over Naming Conventions

**Wrong:** Flat files with prefixes
```
.nfs/conventions/
  pb-integration.md
  pb-deviation-rules.md
  pb-verification-rules.md
  vue-integration.md        ← Added later, requires inventing prefix
  vue-deviation-rules.md
```

**Right:** Stack-scoped directories
```
.nfs/conventions/
  pb/
    integration.md
    deviation-rules.md
    verification-rules.md
  vue/                      ← Added later, copy the directory structure
    integration.md
    deviation-rules.md
    verification-rules.md
```

**Why directories win:**
- Adding a new stack = creating a directory (additive)
- File names are consistent across stacks (no prefix gymnastics)
- `ls .nfs/conventions/` shows which stacks are supported
- Each stack's conventions are self-contained

### Namespace Over Name

**Wrong:** Generic names
```
.claude/agents/
  explorer.md          ← Whose explorer? What stack?
  planner.md           ← Conflict risk with other systems
```

**Right:** Namespaced names
```
.claude/agents/
  nfs-explorer.md      ← NFS system, stack-agnostic
  nfs-pb-planner.md    ← NFS system, PB-specific
  nfs-pb-executor.md   ← NFS system, PB-specific
  nfs-pb-verifier.md   ← NFS system, PB-specific
```

**When to use system prefix vs stack prefix:**
- System prefix (`nfs-`): always, to distinguish from other systems in the same repo
- Stack prefix (`pb-`): when the agent is stack-specific
- No stack prefix: when the agent is stack-agnostic (e.g., Explorer classifies across stacks)

### Classification Fields Over Hard-Coded Routing

**Wrong:** The orchestrator hard-codes which planner to use
```
// In workflow
spawn('nfs-pb-planner')  // Always PB, no routing
```

**Right:** The Explorer's classification includes a stack field, and the orchestrator routes based on it
```
// In EXPLORATION.md
Stack: presentation-backend
Type: modify-form-field

// In workflow
const stack = readClassification().stack;
spawn(`nfs-${stack}-planner`);  // Routes to the right planner
```

For v1, the stack is always `presentation-backend`. The routing logic exists but always takes the same path. When Vue support is added, the routing works without changing the orchestrator — only the Explorer's classification logic needs to handle Vue tickets.

**Cost of adding this now:** One field in EXPLORATION.md, one routing line in the workflow. Trivial.
**Cost of adding this later:** Refactoring the orchestrator, updating all handoff contracts, retesting the pipeline.

## What to Design For Now

### 1. Stack-Scoped Conventions
Move convention files into stack directories. Cost: 10 minutes now. Cost of doing later: updating every agent's file references.

### 2. Stack Field in Classification
Add `Stack: presentation-backend` to EXPLORATION.md. Cost: 1 line now. Cost of doing later: updating the Explorer prompt, the EXPLORATION.md template, and all downstream consumers.

### 3. Namespaced Agents
Use `nfs-pb-*` naming. Cost: 0 (naming is free). Cost of doing later: renaming files, updating all references, confusion during transition.

### 4. Separated Shared vs Stack-Specific Components
Identify which system components are stack-agnostic:
- CLI tools (`nfs-tools.cjs`) — stack-agnostic
- TASK.md template — stack-agnostic
- Explorer — stack-agnostic (classifies all stacks)
- Planner, Executor, Verifier — stack-specific
- Deviation rules, verification rules — stack-specific
- Learnings file — could be shared or stack-specific

When future stacks are added, stack-agnostic components are reused. Stack-specific components are duplicated and specialized.

## What Not to Design For Now

### Don't Build Multi-Stack Routing
The Explorer always returns `Stack: presentation-backend`. The orchestrator always routes to `nfs-pb-*` agents. Don't build a routing table, a stack registry, or a discovery mechanism. When Vue support is added, add an if-else. You'll have at most 3 stacks — a routing table is over-engineering.

### Don't Build Cross-Stack Agents
A "universal planner" that handles PB, Vue, and Figura plans would need to understand all three stacks deeply. Three specialized planners, each focused on one stack, will produce better results. Don't merge them preemptively.

### Don't Build Stack Configuration
"Which stacks are enabled? What model should each stack's planner use? Which conventions apply?" These questions don't need answering until you have more than one stack. Hard-code everything for PB. Extract configuration when the second stack forces you to.

### Don't Build Migration Tooling
"Eventually we'll need to migrate Figura to PB." Yes, eventually. But migration tooling is the highest-complexity, lowest-confidence feature. Build it after the core pipeline is battle-tested, not before.

## The Extension Checklist

Before shipping v1, verify these extensibility requirements:

| Check | Question | Pass Criteria |
|-------|----------|---------------|
| Directory structure | Can I add a new stack by creating a directory? | `.nfs/conventions/<stack>/` pattern |
| Agent naming | Do agent names include stack scope? | `nfs-pb-*` naming convention |
| Classification output | Does the Explorer output a stack field? | `Stack:` field in EXPLORATION.md |
| Template reuse | Are templates stack-agnostic? | TASK.md works for any stack |
| CLI reuse | Is state management stack-agnostic? | `nfs-tools.cjs` doesn't reference PB-specific concepts |
| Convention boundaries | Is each convention file clearly stack-scoped? | No mixed PB/Vue content in any file |

If all pass, your system extends naturally. If any fail, fix it before it becomes a refactoring project.

## The YAGNI Boundary

YAGNI (You Ain't Gonna Need It) applies to features, not to structure:

**Apply YAGNI to features:**
- Don't build Vue agents until you need Vue support
- Don't build migration tooling until migration is a priority
- Don't build cross-stack coordination until you have cross-stack tickets
- Don't build configuration UIs until defaults prove insufficient

**Don't apply YAGNI to structure:**
- DO use stack-scoped directories (even with only one stack)
- DO use namespaced agent names (even with only one system)
- DO include a stack classification field (even if it's always the same value)
- DO separate shared vs stack-specific components (even if there's only one stack)

Structural decisions are cheap to make now and expensive to change later. Feature decisions are expensive to make now and cheap to add later. Invest in structure. Defer features.

---
*Next: [Chapter 10: Production Rollout](10-production-rollout.md)*
