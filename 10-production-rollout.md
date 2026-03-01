# Chapter 10: Production Rollout

## The Rollout Ladder

Agentic systems fail at rollout more often than at development. The system works for the author because the author understands its assumptions, limitations, and quirks. Other developers don't.

The rollout ladder ensures each rung is solid before stepping to the next:

```
Rung 1: Author dogfooding (3-5 real tickets)
Rung 2: Early adopters (2-3 developers, supervised)
Rung 3: Team rollout (all developers, documented)
Rung 4: Self-service (developers onboard independently)
```

**Never skip a rung.** Each rung surfaces a different class of problems.

## Rung 1: Author Dogfooding

You built the system. Now use it on real work — not synthetic test tickets, not demo scenarios, real tickets from the backlog that would exist regardless of the system.

### What Dogfooding Reveals

**Prompt quality issues:** The Explorer misclassifies a ticket type because the classification tree doesn't account for a common ticket pattern. The Planner generates a plan with a convention rule that contradicts the target page's existing code. The Executor's deviation handling triggers Tier 3 (Ask) for something that should be Tier 1 (auto-fix).

**Workflow friction:** The per-task approval loop is too slow for simple tickets. The checkpoint summary doesn't give enough context for resume. The error recovery options don't cover the actual failure mode.

**Convention gaps:** The integration.md pattern variance table is missing a pattern you encounter on the third ticket. The verification rules are too strict for one category and too loose for another.

### Dogfooding Protocol

1. **Pick 3-5 real tickets** of varying complexity (simple modify, complex modify, edge case)
2. **Run each ticket through the full pipeline** (`/agent-quick`)
3. **Record every friction point** — where you paused, where you had to work around the system, where the output was wrong
4. **Fix each issue** — prompt adjustment, convention update, workflow change
5. **Re-run a previous ticket** after fixes to verify they don't regress

### The Dogfooding Mindset

You must use the system as if you didn't build it. This is hard. You know the workarounds, you know which prompts produce edge cases, you know what the system expects. A new user doesn't.

**Techniques for honest dogfooding:**
- Don't adjust your tickets to fit the system. Use them as-is.
- Don't mentally compensate for known limitations. Let them surface.
- Don't override the system when you know a faster way. Let it complete.
- Take notes the moment friction occurs, not at the end.

## Rung 2: Early Adopters

### Selecting Early Adopters

Choose 2-3 developers who are:
- **Willing to give honest feedback** (not just "it's great")
- **Patient with rough edges** (early adopters see bugs)
- **Representative of the team** (different skill levels, different working styles)
- **Working on real tickets** (not waiting for a special project)

### The First Session

Sit with each early adopter for their first ticket. Not to teach — to observe:
- Where do they get confused?
- Where do they disagree with the system's approach?
- Where does the system fail that it never failed for you?
- What do they try that you never tried?

### What Early Adopters Reveal

**Assumption mismatches:** You assume developers read the plan before approving. Early adopter #2 approves immediately without reading. The system needs a more prominent plan presentation.

**Mental model gaps:** You understand the pipeline (explore → plan → execute → verify). Early adopter #1 doesn't know what the Explorer does or why it's a separate step. The system needs better progress communication.

**Edge cases you never hit:** Early adopter #3 works on a page you never tested against. The Explorer's scanning strategy fails because the page has an unusual structure.

### Feedback Integration

After each early adopter session:
1. Categorize feedback: prompt fix / workflow fix / convention update / documentation gap / out of scope
2. Fix prompt and workflow issues immediately (they affect the next session)
3. Add convention corrections to learnings.md
4. Track documentation gaps for rollout docs
5. Note out-of-scope requests for v2

## Rung 3: Team Rollout

### Rollout Documentation

Before rolling out to the full team, create documentation that answers:

**"What is this?"** — One paragraph explaining what the system does and why.

**"When should I use it?"** — Good-fit vs bad-fit guidance:
- Good fit: add form field, add component, create new page
- Bad fit: bug fixes, pure CSS changes, cross-stack work
- Partial fit: already know what to build (skip Explorer), want convention check only (use Verifier standalone)

**"How do I use it?"** — The 3 commands they need:
- `/agent-quick <ticket>` — full pipeline
- `/agent-verify` — check conventions on existing work
- `/agent-progress` — see where things stand

**"What do I do when it fails?"** — The three recovery options, explained simply.

**"Known limitations"** — Honest list of what doesn't work yet. Developers respect transparency.

### The Rollout Announcement

Don't oversell. The announcement should set appropriate expectations:

> "The agent system is now available for frontend work. It helps with explore→plan→execute→verify
> for standalone tickets. It works best for form fields, components, and new
> modules. It's not meant for bug fixes or cross-stack work. Try it on your next
> ticket with `/agent-quick`. Known limitations: [list]. Feedback: add to
> learnings.md or tell [author]."

### Monitoring After Rollout

For the first 2 weeks after team rollout:
- Check learnings.md daily for new entries
- Ask developers who used it: "What worked? What didn't?"
- Track: how many developers tried it, how many used it a second time, how many stopped
- The "second use" metric is the most important — first use is curiosity, second use is value

## The Learnings Loop

### Why Systems Improve

Agentic systems improve through a feedback loop:

```
Developer uses system → encounters issue → records correction in learnings.md →
agents read correction → system avoids the mistake next time
```

This loop is the system's long-term moat. Raw AI doesn't learn between sessions. A system with a learnings file does.

### learnings.md Format

Keep it simple. Each entry is one line:

```markdown
# Learnings

- [2026-03-05]: Explorer classified "add sorting" as modify-form-field, should be modify-filter — sorting changes the filter service, not the form configuration
- [2026-03-08]: Planner generated separate web-service file for checkout, should reuse existing handlers array — checkout composes handlers
- [2026-03-12]: user-profile page now uses store factory pattern — integration.md variance table needs update (was: dashboard + checkout only)
- [2026-03-15]: Verifier flagged @import in a vendor SCSS file — should only check project SCSS, not vendor imports
```

Each entry includes: date, what happened, what should have happened, why.

### When to Promote Learnings to Conventions

When the same learning appears 3+ times, it should be promoted:
- Recurring classification error → update Explorer's classification tree
- Recurring convention miss → update verification rules
- Recurring pattern variance → update integration.md variance table
- Recurring workflow friction → update the agent prompt or workflow

**Don't promote after one occurrence.** One-off issues don't deserve permanent convention changes.

## Measuring Success

### Adoption Metrics

| Metric | What It Tells You | Target |
|--------|-------------------|--------|
| First-use count | Curiosity / awareness | 100% of team tries it |
| Second-use count | Actual value delivered | 70%+ use it again |
| Weekly active users | Sustained adoption | 50%+ of team uses it weekly |
| Tickets completed with the system | Volume of value | Increasing trend |

### Quality Metrics

| Metric | What It Tells You | Target |
|--------|-------------------|--------|
| Verifier pass rate (first run) | Agent code quality | >80% of verifier checks pass on first execution |
| Learnings entries per week | System gap frequency | Decreasing trend |
| Pipeline completion rate | End-to-end reliability | >90% of started pipelines complete |
| Time to verify (conventions correct) | Convention enforcement | Faster than manual review |

### Anti-Metrics (Don't Measure)

- **Time saved per ticket:** Too variable, too subjective. A developer might spend 20 minutes with the system but would have spent 15 without it — for now. The value is consistency and convention compliance, not speed.
- **Lines of code generated:** Irrelevant. 50 lines of correct, convention-compliant code beats 200 lines of wrong code.
- **Developer satisfaction score:** Too gameable. Watch what developers do (second-use rate), not what they say.

## Common Rollout Failures

### "It Worked For Me"

The author can't reproduce a bug because their mental model compensates for the system's limitations. Early adopters see the bug because they don't know the workarounds.

**Fix:** Always reproduce bugs in a fresh context. Don't use your accumulated knowledge of the system's quirks.

### "Nobody Uses It"

The system is available but developers stick to raw AI assistants or manual development.

**Diagnosis:** Either the onboarding experience is too complex, the first-ticket success rate is too low, or the system solves a problem developers don't feel they have.

**Fix:** Pair with a non-adopter on their next ticket. Watch what they do without the system. Identify the specific moment where the system would have helped — and make sure the system actually handles that moment well.

### "Everyone Uses It Wrong"

Developers invoke the wrong skills, provide wrong inputs, or ignore the system's output.

**Diagnosis:** The mental model gap between how you designed the system and how developers think about their work.

**Fix:** Not more documentation — simpler entry points. If developers keep using `/agent-execute` without running the Explorer first, maybe the Execute skill should detect missing exploration and offer to run it.

### "It Keeps Making the Same Mistake"

The system generates the same convention violation repeatedly, across different developers and tickets.

**Diagnosis:** A gap in the agent prompt that the learnings file is working around instead of fixing.

**Fix:** Promote the learning to a convention change. Update the agent prompt directly. The learnings file is a patch — convention changes are the fix.

## The Long Game

An agentic system's value compounds over time:
- **Week 1:** "Interesting tool, let me try it"
- **Month 1:** "It gets our conventions right most of the time"
- **Month 3:** "New developers use it to learn our patterns"
- **Month 6:** "We can't imagine doing frontend work without it"

The transition from "tool" to "infrastructure" happens when the system knows more about the codebase's conventions than any individual developer. That's the goal. It takes patience, iteration, and honest feedback loops to get there.

---
*This concludes the Agentic System Design guide. Return to [Index](00-index.md).*
