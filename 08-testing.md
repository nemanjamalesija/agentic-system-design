# Chapter 8: Testing Agentic Systems

## The Testing Challenge

You can't unit test a prompt. A prompt is not deterministic — the same input can produce different outputs across runs. Traditional testing (assert input X produces output Y) doesn't apply.

But you can still test agentic systems rigorously. You just need different testing strategies for different layers.

## What's Testable

| Component | Testability | Strategy |
|-----------|------------|----------|
| CLI tools | Fully deterministic | Unit tests (standard) |
| Templates | Fully deterministic | Structural validation |
| Convention files | Static content | Manual review |
| Agent prompts | Non-deterministic | Smoke tests + iteration |
| Orchestration logic | Semi-deterministic | Integration tests |
| Full pipeline | Non-deterministic | End-to-end smoke tests |

## Testing CLI Tools

Your state management CLI is the one component that's fully deterministic. Test it like any other code:

```javascript
// nfs-tools.test.cjs
test('create-task produces valid TASK.md', () => {
  const result = createTask('test-slug');
  assert(result.created === true);
  assert(fs.existsSync(result.path));
  // Verify template placeholders are replaced
  const content = fs.readFileSync(result.path, 'utf8');
  assert(!content.includes('{{'));
});

test('slug collision appends numeric suffix', () => {
  createTask('collision-test');
  const result = createTask('collision-test');
  assert(result.slug === 'collision-test-2');
});

test('load-state returns checkpoint JSON', () => {
  createTask('state-test');
  saveState('state-test', { taskIndex: 2 });
  const state = loadState('state-test');
  assert(state.taskIndex === 2);
});
```

Write these tests first. They're fast, reliable, and catch regressions in the infrastructure that all agents depend on.

## Smoke Testing Agent Prompts

Agent prompts can't be unit tested, but they can be smoke tested: run the agent against known inputs and evaluate the output for structural correctness and quality.

### The Smoke Test Process

1. **Write the agent prompt** (first draft)
2. **Run it against a known input** (a synthetic test ticket)
3. **Evaluate the output** — structural correctness first, then content quality
4. **Fix the prompt** — adjust wording, add examples, clarify ambiguous instructions
5. **Run again** — verify the fix worked without breaking other scenarios
6. **Try a different input** — verify the prompt generalizes
7. **Repeat** until output is consistently good across 2-3 inputs

### Evaluating Smoke Test Output

**Level 1: Structural correctness**
- Does the output file exist?
- Does it have the required sections?
- Are sections non-empty?
- Is any machine-readable data (JSON, tables) valid?

**Level 2: Downstream consumability**
- Can the next agent in the pipeline successfully read and act on this output?
- Does the Planner produce a valid plan from this Explorer output?
- Does the Executor understand the tasks in this plan?

**Level 3: Convention compliance**
- Does generated code follow the conventions?
- Is the wiring correct?
- Does the build pass?

A smoke test "passes" when all three levels are satisfied.

### What Makes a Good Smoke Test Input

**Synthetic test tickets** — not real Jira tickets. Synthetic inputs are:
- **Repeatable:** Same input every time, so you can compare outputs across prompt iterations
- **Controlled:** You know the expected classification, file targets, and complexity
- **Version-controlled:** The test inputs live in the repo, not in your head

**Example test tickets:**

```
Test Ticket A (modify-form-field):
"Add a price range filter to the listing-submit form. Users should be able
to set a minimum and maximum price. The filter should use the existing
form-configuration pattern."
Target page: listing-submit

Test Ticket E (create-new-page):
"Create a new 'favorites' page that lists a user's saved listings.
Similar to browse-listings but filtered to saved items."
Target page: (none — new page)
```

### Per-Agent Test Scenarios

Design test scenarios tailored to each agent's risks:

**Explorer (3 golden-path + 2 error-path):**
- Golden: modify-form-field on listing-submit
- Golden: create-new-page (no target)
- Golden: add-component on browse-listings
- Error: vague ticket description (ambiguous type)
- Error: invalid target page name

**Planner (2 golden-path + 1 error-path):**
- Golden: modify-plan from Explorer output
- Golden: create-plan from Explorer output
- Error: ticket that requires PHP changes (out of scope)

**Executor (2 golden-path + 2 error-path):**
- Golden: 3-task modify ticket
- Golden: resume from checkpoint
- Error: build breaks mid-execution
- Error: developer rejects approach

**Verifier (2 golden-path + 2 error-path):**
- Golden: clean code passes all checks
- Golden: code with a deliberate wiring violation
- Error: stale/empty checkpoint
- Error: build infrastructure failure

### The Iteration Loop

Prompt engineering is iterative. Expect to modify each agent's prompt 3-5 times before it produces consistently good output.

**Iteration protocol:**
1. Run smoke test
2. Read the full output carefully
3. Identify the specific prompt instruction that led to the problem
4. Fix that instruction (reword, add example, add constraint)
5. Re-run the SAME smoke test (verify the fix)
6. Run a DIFFERENT smoke test (verify you didn't break generalization)

**Track what you changed and why.** A prompt changelog helps when future changes accidentally reintroduce problems you've already fixed.

## Testing the Full Pipeline

After individual agents produce good output in isolation, test the full pipeline end-to-end:

### Pipeline Smoke Test

Run the complete workflow on a synthetic ticket:
1. Explorer produces EXPLORATION.md
2. Handoff validation passes
3. Planner reads EXPLORATION.md, produces PLAN.md
4. Handoff validation passes
5. Executor reads PLAN.md, implements code
6. Verifier checks the code

**Two pipeline runs:**
- One modify-ticket (the 95% case)
- One create-ticket (the different plan shape)

### What Pipeline Tests Catch

Pipeline tests catch **integration failures** that isolated agent tests miss:
- Explorer output that the Planner can't use (field naming mismatch)
- Plans that the Executor can't follow (ambiguous task descriptions)
- Executor output that the Verifier can't scope (empty filesModified)
- Handoff validation that's too strict (rejects valid output) or too loose (accepts malformed output)

### Pipeline vs Isolation Testing

Don't skip isolation testing and jump to pipeline testing. When the pipeline fails, you need to know WHICH agent caused the failure. If you've only tested the pipeline, debugging is: "somewhere in this 4-agent chain, something went wrong."

Test each agent in isolation FIRST, then test the pipeline. The pipeline test becomes a validation that the individually-correct agents compose correctly.

## Error Path Testing

Golden-path-only testing is how agentic systems fail in production. For each agent, test at least 1-2 error scenarios.

### Why Error Paths Matter

Error paths test:
- Does the agent produce structured error output (`## BLOCKED`)?
- Does the orchestrator correctly detect the error?
- Does the developer get actionable information?
- Is state preserved for recovery?

### Designing Error Test Cases

For each agent, identify the most likely failure modes:

| Agent | Failure Mode | Expected Behavior |
|-------|-------------|-------------------|
| Explorer | Invalid target page | `## BLOCKED` with suggestion |
| Explorer | Ambiguous ticket | Classification with low confidence flag |
| Planner | Out-of-scope ticket | `## BLOCKED` explaining scope limitation |
| Executor | Build breaks | Checkpoint saved, failure presented with options |
| Executor | Developer rejects | Re-present with adjusted approach |
| Verifier | Missing checkpoint | `## BLOCKED` with recovery suggestion |

### The Error Output Test

Every error path should produce output that the orchestrator can detect and the developer can act on. Test that:
1. `## BLOCKED` section exists
2. Reason is specific (not "something went wrong")
3. Suggestion is actionable (not "please try again")
4. State is preserved (checkpoint saved before failing)

## What Not to Test

### Don't Test Prompt Phrasing

If you change "analyze the page" to "examine the page" and the output quality is the same, that's not a regression. Don't write tests that depend on exact prompt wording.

### Don't Test Output Formatting Details

If the Explorer puts pattern variance in a table vs a list, and both are consumable by the Planner, that's not a failure. Test structure (sections exist, data is present), not formatting.

### Don't Automate Semantic Quality

"Is this a good plan?" requires human judgment. Don't build a test that uses another AI to evaluate the first AI's output — you'll test the evaluator's biases, not the agent's quality.

### Don't Test Hypothetical Edge Cases

Test the failure modes that actually occur. If you've never seen a ticket that spans PB + Figura stacks, don't build a test for it. Wait for it to happen in production, then add the test.

## Testing Cadence

### During Development (Phase 2)

Heavy iteration. 3-5 prompt revisions per agent. Smoke test after every revision.

### During Dogfooding (Phase 3-4)

Each real ticket is a test. Track: Did the agent succeed? Where did it fail? What prompt change would fix it?

### After Rollout

Monitor learnings.md entries. Each correction is a signal that the prompt needs refinement. Batch corrections and iterate on the prompt periodically (not per-correction).

---
*Next: [Chapter 9: Extensibility](09-extensibility.md)*
