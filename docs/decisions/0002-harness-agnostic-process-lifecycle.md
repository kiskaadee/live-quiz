# ADR-0002: Harness-Agnostic Process Lifecycle

- Status: Accepted
- Date: 2026-11-10

## Context
When an AI agent or skill invokes `live-quiz`, the execution must not block agent reasoning indefinitely, yet the agent must be notified when the assessment concludes. We needed a process coordination mechanism that works cleanly without tight coupling to proprietary agent harnesses.

## Decision
Design `live-quiz` as a standard foreground UNIX process whose termination signals session conclusion. In agent environments that support managed asynchronous tasks (such as Antigravity background tasks), the agent launches `live-quiz start`, yields the URL to the user, and terminates its turn. When `live-quiz` reaches a terminal state, its process exits with code 0, naturally triggering reactive wakeup of the agent harness.

## Alternatives Considered
- **Antigravity Sidecars and `agentapi`:** Rejected. Testing revealed that `agentapi` relies on Antigravity internal language server environment variables (`ANTIGRAVITY_LS_ADDRESS`, `ANTIGRAVITY_CSRF_TOKEN`), breaking execution when run outside the agent environment and introducing unnecessary daemon infrastructure.
- **Indefinite Synchronous Blocking Command:** Rejected. Fails over human time scales; if a learner pauses or steps away, long-running agent tool calls timeout or hang.
- **Polling Loop:** Rejected. Wastes tokens and agent execution turns polling local status endpoints.

## Consequences
- **Positive:** Complete decoupling from harness internals; standard OS process semantics (`exit 0`); portable across Antigravity, Claude Code, terminal shells, and CI pipelines.
- **Negative / Trade-offs:** Requires the calling host environment or user to manage process execution (e.g. running as a background task).

## Related Decisions
- [ADR-0001: Standalone Local Web Application](./0001-standalone-web-application.md)
- [ADR-0006: Ephemeral SQLite Session Journal & Recovery](./0006-ephemeral-sqlite-session-journal.md)
