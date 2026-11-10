# ADR-0007: Atomic Versioned JSON Export Contract

- Status: Accepted
- Date: 2026-11-10

## Context
When `live-quiz` concludes, it must hand off the assessment evidence to caller workflows (agents, evaluators, or future learning platforms). We needed a durable, forward-compatible integration boundary that does not expose internal SQLite database implementation details.

## Decision
Establish **`attempt.json` as a standardized, versioned JSON contract** published via an atomic file-replacement pattern:
1. **Atomic Publication Pipeline:**
   - The final attempt data is serialized to JSON conforming to `attempt.schema.json`.
   - Written to a temporary file (`.attempt.json.tmp`) within the target directory.
   - Validated for structural correctness and flushed/fsynced to persistent storage.
   - Atomically renamed to the final destination path (`attempt.json`).
   - *Failure Guarantee:* If publication fails at any stage (e.g. disk full, permissions error), the temporary SQLite journal is retained, and the process exits with an error status.
2. **Reattempt vs. Recovery Semantics:**
   - **Recovery (`live-quiz recover`):** Resumes an interrupted attempt, preserving the existing `attempt_id` and journal state.
   - **Reattempt (`live-quiz start`):** Launches a fresh execution against the *same* immutable assessment definition, generating a distinct new `attempt_id`.
3. **Contract Attributes:**
   Every exported attempt contains explicit provenance:
   - `schema_version`
   - `attempt_id`
   - `assessment_id` & `assessment_version` (with `questionnaire_hash`)
   - `workspace` locator
   - `status` (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`)
   - `timing` (active duration seconds, wall-clock duration seconds, pause intervals)
   - `responses` array

## Alternatives Considered
- **Direct SQLite Database Querying by Evaluators / Future LMS:** Rejected. Exposing internal database tables couples external consumers to `live-quiz` schema migrations.
- **Direct In-Place File Overwriting:** Rejected. If a process crash occurs during file write, the destination file is left partially written and corrupted.

## Consequences
- **Positive:** Complete data durability; forward compatibility with any future LMS or evaluator; safe atomic handoff to agent callers.
- **Negative / Trade-offs:** Requires an atomic rename step and validation pass prior to session journal cleanup.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
- [ADR-0006: Ephemeral SQLite Session Journal & Recovery](./0006-ephemeral-sqlite-session-journal.md)
