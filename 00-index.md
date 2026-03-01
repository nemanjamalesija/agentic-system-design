# Agentic System Design: A Practitioner's Guide

A comprehensive guide to designing, building, and shipping multi-agent systems that work in production. Derived from real-world system design, not theory.

## Who This Is For

Engineers building agentic systems on top of LLM platforms. The principles are platform-agnostic; the examples are grounded in practical primitives.

## Chapters

1. **[Foundations](01-foundations.md)** — What agentic systems actually are, mental models, when to build one vs when not to
2. **[Architecture](02-architecture.md)** — Layered architecture, separation of concerns, the Skill-Workflow-Agent-Convention-State stack
3. **[Agent Design](03-agent-design.md)** — Prompt engineering for agents, inline vs reference, structural consistency, model routing
4. **[Multi-Agent Orchestration](04-orchestration.md)** — Handoff validation, context-lean orchestration, error recovery protocols
5. **[Convention Systems](05-convention-systems.md)** — Knowledge management, ownership boundaries, source attribution, preventing drift
6. **[State Management](06-state-management.md)** — Checkpoints, persistence, resume, crash recovery, state serialization
7. **[Developer Experience](07-developer-experience.md)** — Interaction models, trust building, collaborative vs autonomous, feedback loops
8. **[Testing Agentic Systems](08-testing.md)** — Smoke testing prompts, golden path + error paths, pipeline validation
9. **[Extensibility](09-extensibility.md)** — Designing for future growth without over-engineering the present
10. **[Production Rollout](10-production-rollout.md)** — Dogfooding, graduated rollout, learnings loops, team adoption

## How to Read

- **New to agentic systems:** Read chapters 1-4 in order. They build on each other.
- **Designing your first system:** Focus on chapters 2, 3, 5, and 7. Architecture and developer experience are where most systems succeed or fail.
- **Debugging a struggling system:** Jump to chapters 4 (orchestration failures), 6 (state bugs), and 8 (testing gaps).
- **Scaling an existing system:** Chapters 5, 9, and 10 address growth and team adoption.

## Core Thesis

Most agentic systems fail not because the AI isn't capable, but because the system around it — the orchestration, state management, convention enforcement, and developer interaction model — is poorly designed. A mediocre model in a well-designed system outperforms a brilliant model in a poorly designed one.

The best agentic systems are boring. They use simple primitives, predictable patterns, and explicit contracts between components. They don't try to be clever. They try to be reliable.

---
*Written from practical experience designing production agentic systems.*
