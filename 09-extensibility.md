# Chapter 9: Extensibility

## The Extensibility Paradox

You must design for extensibility without building for it. The moment you start implementing features for hypothetical future requirements, you're over-engineering. But the moment you make a naming decision that boxes you into one stack, you're creating technical debt.

The goal: **make the right structural decisions now so that future work is additive, not refactoring.**

## Structural Decisions That Matter

### Directory Structure Over Naming Conventions

**Wrong:** Flat files with prefixes
```
.agent/conventions/
  frontend-integration.md
  frontend-deviation-rules.md
  frontend-verification-rules.md
  backend-integration.md        ← Added later, requires inventing prefix
  backend-deviation-rules.md
```

**Right:** Stack-scoped directories
```
.agent/conventions/
  frontend/
    integration.md
    deviation-rules.md
    verification-rules.md
  backend/                      ← Added later, copy the directory structure
    integration.md
    deviation-rules.md
    verification-rules.md
```

**Why directories win:**
- Adding a new stack = creating a directory (additive)
- File names are consistent across stacks (no prefix gymnastics)
- `ls .agent/conventions/` shows which stacks are supported
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
  sys-explorer.md      ← your system, stack-agnostic
  sys-frontend-planner.md    ← your system, frontend-specific
  sys-frontend-executor.md   ← your system, frontend-specific
  sys-frontend-verifier.md   ← your system, frontend-specific
```

**When to use system prefix vs stack prefix:**
- System prefix (`sys-`): always, to distinguish from other systems in the same repo
- Stack prefix (`frontend-`): when the agent is stack-specific
- No stack prefix: when the agent is stack-agnostic (e.g., Explorer classifies across stacks)

### Classification Fields Over Hard-Coded Routing

**Wrong:** The orchestrator hard-codes which planner to use
```
// In workflow
spawn('sys-frontend-planner')  // Always frontend, no routing
```

**Right:** The Explorer's classification includes a stack field, and the orchestrator routes based on it
```
// In EXPLORATION.md
Stack: frontend
Type: modify-form-field

// In workflow
const stack = readClassification().stack;
spawn(`sys-${stack}-planner`);  // Routes to the right planner
```

For v1, the stack is always `frontend`. The routing logic exists but always takes the same path. When a new stack is added, the routing works without changing the orchestrator — only the Explorer's classification logic needs to handle new stack tickets.

**Cost of adding this now:** One field in EXPLORATION.md, one routing line in the workflow. Trivial.
**Cost of adding this later:** Refactoring the orchestrator, updating all handoff contracts, retesting the pipeline.

## What to Design For Now

### 1. Stack-Scoped Conventions
Move convention files into stack directories. Cost: 10 minutes now. Cost of doing later: updating every agent's file references.

### 2. Stack Field in Classification
Add `Stack: frontend` to EXPLORATION.md. Cost: 1 line now. Cost of doing later: updating the Explorer prompt, the EXPLORATION.md template, and all downstream consumers.

### 3. Namespaced Agents
Use `sys-frontend-*` naming. Cost: 0 (naming is free). Cost of doing later: renaming files, updating all references, confusion during transition.

### 4. Separated Shared vs Stack-Specific Components
Identify which system components are stack-agnostic:
- CLI tools (`agent-tools.js`) — stack-agnostic
- TASK.md template — stack-agnostic
- Explorer — stack-agnostic (classifies all stacks)
- Planner, Executor, Verifier — stack-specific
- Deviation rules, verification rules — stack-specific
- Learnings file — could be shared or stack-specific

When future stacks are added, stack-agnostic components are reused. Stack-specific components are duplicated and specialized.

## What Not to Design For Now

### Don't Build Multi-Stack Routing
The Explorer always returns `Stack: frontend`. The orchestrator always routes to `sys-frontend-*` agents. Don't build a routing table, a stack registry, or a discovery mechanism. When a new stack is added, add an if-else. You'll have at most 3 stacks — a routing table is over-engineering.

### Don't Build Cross-Stack Agents
A "universal planner" that handles frontend, backend, and mobile plans would need to understand all three stacks deeply. Three specialized planners, each focused on one stack, will produce better results. Don't merge them preemptively.

### Don't Build Stack Configuration
"Which stacks are enabled? What model should each stack's planner use? Which conventions apply?" These questions don't need answering until you have more than one stack. Hard-code everything for frontend. Extract configuration when the second stack forces you to.

### Don't Build Migration Tooling
"Eventually we'll need to migrate legacy stack to the primary framework." Yes, eventually. But migration tooling is the highest-complexity, lowest-confidence feature. Build it after the core pipeline is battle-tested, not before.

## The Extension Checklist

Before shipping v1, verify these extensibility requirements:

| Check | Question | Pass Criteria |
|-------|----------|---------------|
| Directory structure | Can I add a new stack by creating a directory? | `.agent/conventions/<stack>/` pattern |
| Agent naming | Do agent names include stack scope? | `sys-frontend-*` naming convention |
| Classification output | Does the Explorer output a stack field? | `Stack:` field in EXPLORATION.md |
| Template reuse | Are templates stack-agnostic? | TASK.md works for any stack |
| CLI reuse | Is state management stack-agnostic? | `agent-tools.js` doesn't reference stack-specific concepts |
| Convention boundaries | Is each convention file clearly stack-scoped? | No mixed frontend/backend content in any file |

If all pass, your system extends naturally. If any fail, fix it before it becomes a refactoring project.

## The YAGNI Boundary

YAGNI (You Ain't Gonna Need It) applies to features, not to structure:

**Apply YAGNI to features:**
- Don't build backend agents until you need backend support
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
