# ADR-0007: Atomic Versioned JSON Export Contract

- Status: Accepted
- Date: 2026-11-10

## Context
When `live-quiz` concludes, it must hand off the assessment evidence to caller workflows (agents, evaluators, or future learning platforms). We needed a durable, forward-compatible integration boundary that does not expose internal SQLite database implementation details.

## Decision
Establish **`attempt.json` as a standardized, versioned JSON contract** published via an atomic file-replacement pattern:
1. **Atomic Publication Pipeline:**
   - The final attempt data is serialized to JSON conforming to the versioned attempt schema contract (`attempt.schema.json`).
   - Written to a temporary file (`.attempt.json.tmp`) within the target directory.
   - Validated for structural correctness and flushed/fsynced to persistent storage.
   - Atomically renamed to the final destination path (`attempt.json`).
   - *Failure Guarantee:* If publication fails at any stage (e.g. disk full, permissions error), the temporary SQLite journal file set is retained intact, and the process exits with an error status. Recovery (`live-quiz recover`) retries publication under the exact same `attempt_id`.
2. **Reattempt vs. Recovery Semantics:**
   - **Reattempt (`live-quiz start`):** Launches a fresh execution against the *same* immutable assessment definition, generating a distinct new `attempt_id`.
   - **Nonterminal Recovery (`live-quiz recover`):** Resumes an interrupted or paused attempt, restoring clock and drafts under the existing `attempt_id`.
   - **Terminal Export Recovery (`live-quiz recover`):** Retries failed atomic publication directly from the terminal journal without reopening the assessment, restarting the timer, or permitting answer modifications.
3. **Contract Attributes & Canonical Objective Traceability:**
   Every exported attempt contains explicit provenance:
   - `schema_version`
   - `attempt_id`
   - `assessment_id` & `assessment_version` (with `questionnaire_hash`)
   - `workspace` locator
   - `status` (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`)
   - `timing` (active duration seconds, wall-clock duration seconds, pause intervals)
   - `responses` array preserving mappings to associated canonical objective references carried in the questionnaire.

   *Structural vs. Semantic Validation Invariant:*
   The exported payload must conform structurally to the versioned attempt schema contract (`attempt.schema.json`). However, the engine maintains zero semantic awareness of external registries: it does not validate, resolve, or query objective identifiers against external catalogs, preserving them verbatim for downstream evaluation. The exact schema layout for preserving question-to-objective response traceability remains open until the JSON schema contracts are formally authored.
4. **Response State Triad & Attainment Decoupling:**
   Each item in `responses[]` represents its interaction state independently of the assessment lifecycle status (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`):
   - `SUBMITTED`: Explicitly confirmed by the learner. Includes explicit blank submissions where the learner intentionally submitted an empty response.
   - `DRAFT`: Autosaved input acknowledged by SQLite commit-before-ack but unsubmitted when session terminated. Represents an acknowledged learner-generated modification (initial empty fields remain `UNANSWERED`). The latest acknowledged draft payload and server-side receipt timestamp are preserved verbatim. (Per-question active duration is designated provisional pending formal measurement definitions).
   - `UNANSWERED`: Zero acknowledged learner input recorded (payload is `null`).
   
   *Valid Terminal/Response State Constraints:*
   - `COMPLETED`: Strictly requires every questionnaire item to have `response_status: SUBMITTED` (explicit blank submissions qualify).
   - `CLOSED_EARLY`: At least one response must be `DRAFT` or `UNANSWERED`, following an explicit learner termination action.
   - `TIMED_OUT`: At least one response must be `DRAFT` or `UNANSWERED`, following authoritative timer expiration.

   *Generalized Server-Side Serialization & Absorbing Terminal States:*
   All response mutations (draft autosaves, explicit submissions) and terminal lifecycle transitions (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`) are serialized server-side within SQLite transactions. The final required response submission and transition to `COMPLETED` must be committed atomically. Once any terminal lifecycle transition commits, its status is strictly immutable and absorbing: all subsequent response mutations and competing terminal transition attempts are rejected.
   
   *Attainment vs. Submission Independence:*
   Submission status describes the user interaction, not the correctness or cognitive quality of the content. `quiz-evaluator` enforces evidence-first principles across the response triad:
   - Explicit Blank `SUBMITTED`: Evaluated as an intentional empty response against rubric criteria.
   - `DRAFT`: Evaluated as available evidence of learning without automatic penalty for draft status, disclosing draft provenance and reporting demonstrated vs. unestablished criteria.
   - `UNANSWERED`: Reports that no response evidence exists; must not infer misconceptions or conclude that the learner lacks underlying knowledge solely due to absence of recorded input.

## Alternatives Considered
- **Direct SQLite Database Querying by Evaluators / Future LMS:** Rejected. Exposing internal database tables couples external consumers to `live-quiz` schema migrations.
- **Direct In-Place File Overwriting:** Rejected. If a process crash occurs during file write, the destination file is left partially written and corrupted.

## Consequences
- **Positive:** Complete data durability; forward compatibility with any future LMS or evaluator; safe atomic handoff to agent callers.
- **Negative / Trade-offs:** Requires an atomic rename step and validation pass prior to session journal cleanup.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
- [ADR-0006: Ephemeral SQLite Session Journal & Recovery](./0006-ephemeral-sqlite-session-journal.md)
