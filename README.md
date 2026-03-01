# Agentic System Design

A comprehensive guide to designing, building, and shipping multi-agent systems that work in production. Derived from real-world system design, not theory.

## Who This Is For

Engineers building agentic systems on top of LLM platforms — whether using hosted AI platforms or custom runtimes. The principles are platform-agnostic; the examples are grounded in practical primitives.

## Core Thesis

Most agentic systems fail not because the AI isn't capable, but because the system around it — the orchestration, state management, convention enforcement, and developer interaction model — is poorly designed. A mediocre model in a well-designed system outperforms a brilliant model in a poorly designed one.

The best agentic systems are boring. They use simple primitives, predictable patterns, and explicit contracts between components. They don't try to be clever. They try to be reliable.

## Chapters

| # | Chapter | Focus |
|---|---------|-------|
| 1 | [Foundations](01-foundations.md) | What agentic systems actually are, mental models, when to build one |
| 2 | [Architecture](02-architecture.md) | Layered architecture, separation of concerns, the five-layer stack |
| 3 | [Agent Design](03-agent-design.md) | Prompt engineering for agents, structural consistency, model routing |
| 4 | [Multi-Agent Orchestration](04-orchestration.md) | Handoff validation, context-lean orchestration, error recovery |
| 5 | [Convention Systems](05-convention-systems.md) | Knowledge management, ownership boundaries, preventing drift |
| 6 | [State Management](06-state-management.md) | Checkpoints, persistence, resume, crash recovery |
| 7 | [Developer Experience](07-developer-experience.md) | Trust building, collaborative vs autonomous, feedback loops |
| 8 | [Testing](08-testing.md) | Smoke testing prompts, golden path + error paths, pipeline validation |
| 9 | [Extensibility](09-extensibility.md) | Designing for growth without over-engineering the present |
| 10 | [Production Rollout](10-production-rollout.md) | Dogfooding, graduated rollout, learning loops, team adoption |

## How to Read

- **New to agentic systems:** Read chapters 1–4 in order. They build on each other.
- **Designing your first system:** Focus on chapters 2, 3, 5, and 7. Architecture and developer experience are where most systems succeed or fail.
- **Debugging a struggling system:** Jump to chapters 4 (orchestration failures), 6 (state bugs), and 8 (testing gaps).
- **Scaling an existing system:** Chapters 5, 9, and 10 address growth and team adoption.

## License

MIT
