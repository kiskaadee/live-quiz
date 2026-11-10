# ADR-0003: Static Code Challenges Without Runtime Execution

- Status: Accepted
- Date: 2026-11-10

## Context
Code exercises (completion, refactoring, bug-fixing) are critical for measuring engineering understanding. We needed to decide whether `live-quiz` should execute code locally and run test suites, or collect code purely as structured text artifacts.

## Decision
Treat all code challenges in `live-quiz` strictly as static text artifacts. The browser provides a rich, syntax-highlighted code editor (e.g. CodeMirror) with read-only context and editable buffers, but there is no "Run" button, local test runner, or subshell code execution in the application. Code submissions are evaluated post-hoc by the LLM evaluator against the frozen rubric.

## Alternatives Considered
- **Local Code Execution Sandbox in `live-quiz`:** Rejected. Introduces substantial security attack surfaces, language runtime dependencies, and complex test-runner orchestration. Crucially, running tests locally provides immediate feedback, turning a blind-pass assessment into an interactive trial-and-error debugging session.

## Consequences
- **Positive:** Keeps `live-quiz` lightweight, secure, and disposable; preserves the pedagogical invariant of a single blind pass measuring unassisted reasoning; eliminates execution environment dependencies on the host machine.
- **Negative / Trade-offs:** Cannot verify dynamic runtime behavior, performance benchmarks, or edge-case execution directly during the quiz; relies on the post-hoc evaluator to analyze code logic statically.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
