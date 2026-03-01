# Chapter 6: State Management

## State Is Survival

In a single-session AI interaction, state doesn't matter. The conversation has context, and when it ends, everything is disposable. In an agentic system, state is the difference between "session dropped, all work lost" and "session dropped, resume from task 3 of 5."

State management is boring infrastructure. It's also the infrastructure that determines whether developers trust your system enough to use it on real work.

## What State Captures

### Progress State

Where are we in the workflow? Which tasks are done? Which is next?

```json
{
  "taskIndex": 3,
  "tasks": [
    { "name": "Add store action", "status": "complete" },
    { "name": "Add API handler", "status": "complete" },
    { "name": "Wire into data fetching", "status": "complete" },
    { "name": "Add component", "status": "pending" },
    { "name": "Update index.vue", "status": "pending" }
  ]
}
```

### Artifact State

What files has the system produced? What did each agent generate?

```
.agent-state/<slug>/
  TASK.md          ← Progress tracking + checkpoint
  EXPLORATION.md   ← Explorer output
  PLAN.md          ← Planner output
  SUMMARY.md       ← Verifier output (after completion)
```

### Change State

What files in the codebase were modified? What was their state before modification?

```json
{
  "filesModified": [
    "src/modules/checkout/store.ts",
    "src/modules/checkout/service.ts"
  ],
  "fileHashes": {
    "store.ts": "a1b2c3d4",
    "service.ts": "e5f6g7h8"
  }
}
```

### Decision State

What choices did the developer make during the workflow? What variant patterns were resolved?

```json
{
  "deviations": [
    { "type": "notify-and-fix", "description": "Changed @import to @use in component.scss" },
    { "type": "auto-fix", "description": "Added missing import for useFormStore" }
  ]
}
```

## Checkpoint Protocol

### When to Checkpoint

**After each task completes:** The minimum checkpoint granularity. If the session drops, you lose at most one task's work.

**On deviation:** When the agent auto-fixes or notify-and-fixes something, checkpoint the deviation immediately. This populates the deviations array in real-time rather than reconstructing it from memory at summary time.

**Never checkpoint mid-task.** A half-completed task is worse than no task — the codebase is in an inconsistent state. Either complete the task and checkpoint, or abort the task and checkpoint the pre-task state.

### Checkpoint Data

Every checkpoint should capture:

```json
{
  "taskIndex": 2,
  "tasks": ["Add store action [complete]", "Add handler [complete]", "Wire data fetching [pending]"],
  "filesModified": ["store.ts", "service.ts"],
  "fileHashes": { "store.ts": "a1b2c3d4", "service.ts": "e5f6g7h8" },
  "deviations": [{ "type": "auto-fix", "description": "Added missing import" }],
  "errors": [],
  "summary": "2 of 5 tasks complete. Store action and handler added.",
  "lastUpdated": "2026-03-01T14:30:00Z"
}
```

**Key fields:**
- `taskIndex` — where to resume from
- `filesModified` — what the Verifier should check
- `fileHashes` — for resume verification
- `deviations` — for the Verifier and summary
- `summary` — for the context-lean orchestrator (reads this one line, not the full state)

### State Serialization

State should be embedded in human-readable artifacts, not stored in opaque binary formats:

```markdown
## Checkpoint

```json
{
  "taskIndex": 2,
  "tasks": [...],
  "filesModified": [...],
  "summary": "2 of 5 tasks complete."
}
```​
```

JSON embedded in markdown fenced code blocks. Readable by humans (open the file), parseable by machines (regex extraction), editable in emergencies (developer can manually fix state).

## Resume Protocol

### Session Resume Flow

When a developer resumes work (same developer, same machine, next day):

1. **Load checkpoint** — read the last saved state
2. **Verify file integrity** — hash files in `filesModified` and compare to `fileHashes`
3. **If files match** — present status summary, continue from `taskIndex`
4. **If files changed** — tell the developer which files changed, ask whether to continue or re-run from a specific task

### Why Hash-Based Verification

You might think `git diff` is the right tool for resume verification. It's not, for one reason: **the developer may not have committed.**

Between sessions, the executor's changes and the developer's manual edits are all uncommitted. `git diff` shows everything relative to the last commit — it can't distinguish "executor wrote this" from "developer edited this after."

File content hashes solve this cleanly:
- At checkpoint time: hash each modified file's content
- At resume time: hash the same files again
- If hashes match: files are exactly as the executor left them
- If hashes differ: someone edited them (developer, colleague, another tool)

No git dependency. No commit requirement. Works in any state of the working tree.

### Developer Handoff

For v1, "handoff to a colleague" is aspirational. The state directory is gitignored (per-ticket runtime state shouldn't be committed). Developer B pulling the branch won't have Developer A's `.agent-state/<slug>/` directory.

**Pragmatic v1 approach:** Scope state persistence to "same developer can pause and resume." The primary use case is overnight interruptions, not team handoff.

**v2 approach:** Add an `export-state` / `import-state` mechanism that bundles state into a shareable artifact. Or move state to a non-gitignored location with developer-controlled commits.

## State Tooling

### CLI Over Direct File Manipulation

Agents should interact with state through CLI commands, not by parsing markdown directly:

```bash
# Create task state
node .agent/bin/agent-tools.js create-task "add-price-filter"

# Load checkpoint
node .agent/bin/agent-tools.js load-state "add-price-filter"

# Save checkpoint
node .agent/bin/agent-tools.js save-state "add-price-filter" --data '{"taskIndex": 2, ...}'
```

**Why CLI:**
- Agents don't need to know the markdown format (checkpoint regex, file locations)
- The CLI handles edge cases (slug collision, missing files, malformed JSON)
- JSON output is easy for agents to parse
- Error handling is centralized (CLI returns exit codes, error messages to stderr)

### CLI Design Principles

1. **JSON output only.** Agents parse JSON, not human-readable text. Output JSON to stdout, errors/usage to stderr.

2. **No external dependencies.** Use only language built-ins (Node.js `fs`, `path`). The CLI should work without `npm install`.

3. **Idempotent operations.** `save-state` replaces the checkpoint, not appends. `create-task` with a collision appends a numeric suffix (`-2`, `-3`), not fails.

4. **Deterministic paths.** State lives in `.agent-state/<slug>/`. The slug is derived from the task description. No randomness, no timestamps in paths.

## State Anti-Patterns

### The Invisible State

State stored in the AI's conversation context (memory) rather than on disk. When the session ends, state evaporates.

**Fix:** All state must be persisted to files. If it's not on disk, it doesn't exist.

### The Monolith State File

A single state file that contains everything: progress, artifacts, history, configuration, learnings. It grows unboundedly and becomes hard to parse.

**Fix:** Separate files by concern. TASK.md for progress. EXPLORATION.md for analysis. PLAN.md for the plan. SUMMARY.md for results. Each file has a focused scope.

### The Over-Checkpointed System

Checkpointing every 10 seconds, after every file read, after every developer interaction. The state file becomes a log rather than a snapshot.

**Fix:** Checkpoint at meaningful boundaries only. After task completion. On deviation. Not mid-task, not mid-thought.

### The Unverified Resume

Resuming from a checkpoint without verifying that the codebase matches the checkpoint's assumptions. The executor continues from task 3, but task 2's files have been manually edited, and task 3 builds on stale assumptions.

**Fix:** Hash-based verification on resume. Quick, non-destructive, catches the real failure mode.

### The Orphaned State

State directories that accumulate indefinitely. After 50 tickets, `.agent-state/` contains 50 directories, most long-completed. No cleanup mechanism.

**Fix:** For v1, accept the accumulation (disk is cheap). For v1.1, add a `cleanup` command that archives or deletes state directories older than N days, with confirmation.

---
*Next: [Chapter 7: Developer Experience](07-developer-experience.md)*
