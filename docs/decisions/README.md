# Architecture Decision Records (ADRs)

This directory contains the Architectural Decision Records for the `live-quiz` assessment engine and its companion reference skills, following the Markdown Any Decision Record (MADR) format.

## Decision Index

| ADR ID | Title | Status | Date |
| :--- | :--- | :--- | :--- |
| [ADR-0001](./0001-standalone-web-application.md) | Standalone Local Web Application | Accepted | 2026-11-10 |
| [ADR-0002](./0002-harness-agnostic-process-lifecycle.md) | Harness-Agnostic Process Lifecycle | Accepted | 2026-11-10 |
| [ADR-0003](./0003-static-code-challenges.md) | Static Code Challenges Without Runtime Execution | Accepted | 2026-11-10 |
| [ADR-0004](./0004-questionnaire-rubric-separation.md) | Physical Separation of Questionnaire and Rubric | Accepted | 2026-11-10 |
| [ADR-0005](./0005-python-fastapi-vite-stack.md) | Python/FastAPI Backend and Vite Plus (`vp`) Frontend Stack | Accepted | 2026-11-10 |
| [ADR-0006](./0006-ephemeral-sqlite-session-journal.md) | Ephemeral SQLite Session Journal & Recovery | Accepted | 2026-11-10 |
| [ADR-0007](./0007-atomic-json-export-contract.md) | Atomic Versioned JSON Export Contract | Accepted | 2026-11-10 |
| [ADR-0008](./0008-two-product-repository-architecture.md) | Two-Product Repository Architecture | Accepted | 2026-11-10 |
| [ADR-0009](./0009-modular-portable-reference-skills.md) | Modular, Portable Reference Skills Collection | Accepted | 2026-11-10 |

## Template

New decisions should follow this structure:

```markdown
# ADR-XXXX: [Short Title of Solved Problem]

- Status: [Proposed | Accepted | Superseded]
- Date: [YYYY-MM-DD]

## Context
[What is the context, constraints, and problem driving this decision?]

## Decision
[What architectural approach and invariants were chosen?]

## Alternatives Considered
- [Alternative 1 and why it was rejected]
- [Alternative 2 and why it was rejected]

## Consequences
- [Positive impacts and guarantees]
- [Negative trade-offs and operational costs]

## Related Decisions
- [Links to related ADRs]
```
