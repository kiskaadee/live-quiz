# ADR-0004: Physical Separation of Questionnaire and Rubric

- Status: Accepted
- Date: 2026-11-10

## Context
A core pedagogical invariant of the assessment model is the blind pass: the learner must not receive answer keys, expected solutions, or private evaluation criteria while completing the assessment. We needed an architectural mechanism to ensure this information cannot leak to the client.

## Decision
Enforce complete physical decoupling of the assessment specification at the filesystem level into two distinct artifacts:
1. `questionnaire.json`: The public presentation contract containing question IDs, markdown prompts, diagrams, options, and code stubs. Passed directly to `live-quiz`.
2. `rubric.json`: The private evaluation contract containing expected answers, observable criteria (Complete/Partial/Insufficient), failure mode diagnoses, and misconception signatures. Retained outside `live-quiz` and supplied exclusively to the post-hoc evaluator.

`live-quiz` never receives, loads, or serves `rubric.json`.

## Alternatives Considered
- **Single Unified Assessment JSON with Runtime Filtering:** Rejected. Passing a single file containing both questions and answer keys creates a severe risk of accidental solution leakage through permissive API routes, serialization bugs, or client-side inspection.

## Consequences
- **Positive:** Physical impossibility of solution leakage via browser DevTools, network sniffing, or server memory inspection.
- **Scope Limitation / Boundary:** This architectural separation guarantees that `live-quiz` cannot leak the rubric. It does not protect against an authoring agent inadvertently embedding answers directly into the question prompt text; prompt quality remains an authoring verification concern.

## Related Decisions
- [ADR-0003: Static Code Challenges Without Runtime Execution](./0003-static-code-challenges.md)
- [ADR-0007: Atomic Versioned JSON Export Contract](./0007-atomic-json-export-contract.md)
