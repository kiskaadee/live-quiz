# ADR-0007: Atomic Versioned JSON Export Contract

- Status: Accepted
- Date: 2026-11-10

## Context
When `live-quiz` concludes, it must hand off the assessment evidence to caller workflows (agents, evaluators, or future learning platforms). We needed a durable, forward-compatible integration boundary that does not expose internal SQLite database implementation details.

## Decision
Establish **`attempt.json` as a standardized, versioned JSON contract** published via an atomic file-replacement pattern:
1. **Ten-Step Atomic Publication Pipeline & Directory Durability:**
   Publication follows a strict 10-step sequence protected by dual advisory locks (per-attempt lock `attempt-<attempt-id>.lock` and destination lock `dest-<sha256_dest>.lock`):
   - Acquire exclusive advisory locks (never unlinked while held).
   - Establish write-serialization boundary via `BEGIN IMMEDIATE`:
     - *Path A (Final Submission → `COMPLETED`):* The response mutation to `SUBMITTED`, transition to `COMPLETED`, complete response snapshot, and `publication_intent` (`target_schema_version`, `export_projection_version: "1"`, `canonicalization_version: "1"`, `destination_path`, `expected_sha256`) are committed together in one atomic transaction.
     - *Path B (Timeout / Early Close):* The transaction is established before reading the final response snapshot and committing terminal status (`TIMED_OUT`/`CLOSED_EARLY`) and `publication_intent`.
   - Validate structurally and serialize canonical payload conforming to RFC 8785 Canonical JSON, computing `expected_sha256` prior to `COMMIT;`.
   - Destination conflict check: under destination lock, verify destination does not hold a different `attempt_id` (prohibiting overwriting another attempt's artifact).
   - Write canonical bytes to attempt-specific temporary file `.attempt.<attempt-id>.tmp` in destination directory, flush, and `fsync` file descriptor.
   - Atomically link/rename `.attempt.<attempt-id>.tmp` to `attempt.json` (using atomic no-replace semantics e.g. `renameat2(..., RENAME_NOREPLACE)`).
   - `fsync` the parent directory descriptor to guarantee rename durability across power failure.
   - Verified WAL checkpoint through open writable connection (`PRAGMA wal_checkpoint(TRUNCATE);`).
   - Clean close & ordered journal cleanup: close handles cleanly → unlink `-wal` → unlink `-shm` → unlink primary `.sqlite3` **last**.
   - Emit machine-readable outcome on stdout and exit 0 (`SUCCESS_FINALIZED`).
2. **Reattempt vs. Recovery Semantics & Idempotency Verification:**
   - **Reattempt (`live-quiz start`):** Launches a fresh execution against the *same* immutable assessment definition, generating a distinct new `attempt_id`.
   - **Nonterminal Recovery (`live-quiz recover`):** In-place transactional migration; resumes interrupted or paused live assessment under the existing `attempt_id`.
   - **Terminal Export Recovery (`live-quiz recover`):**
     - *Phase 1 (Read-Only):* Opens SQLite with read-only connection (`mode=ro`), reading existing WAL frames without mutating tables or schema; reconstructs export using `export_projection_version` and validates reconstructed hash against persisted `expected_sha256` (failing closed with exit 3 on mismatch). Legacy journals lacking `publication_intent` are explicitly rejected as unsupported (`UNSUPPORTED_SCHEMA_VERSION`, exit 5). Phase 1 read-only connection is closed cleanly before Phase 2.
     - *Idempotency Verification:* If target exists and matches `attempt_id` and `expected_sha256`, advances directly to Phase 2 cleanup. If different `attempt_id`, exits code 2; if hash mismatch, exits code 3.
     - *Phase 2 (Writable Cleanup):* Opened only after Phase 1 connection is closed and publication durability is established; executes verified WAL checkpoint without schema modifications, closes handles, and unlinks companions then `.sqlite3` last.
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

5. **Structured Process Outcomes & Exit Codes:**
   Operational failures are strictly decoupled from assessment lifecycle outcomes. Standard output emits a single-line JSON result packet (`{"outcome": "...", "attempt_id": "...", ...}`), while standard error carries human-readable logs:

   | Exit Code | Symbolic Name | Meaning | Journal Action |
   | :---: | :--- | :--- | :--- |
   | **`0`** | `SUCCESS_FINALIZED` | Export verified, directory synced, journal unlinked. | Deleted |
   | **`0`** | `SUCCESS_SUSPENDED` | Pause grace period expired cleanly; resume available. | Preserved |
   | **`1`** | `EXPORT_IO_ERROR` | Transient disk/write/sync failure during publication. | Preserved |
   | **`2`** | `CONFLICT_DIFFERENT_ATTEMPT` | Destination occupied by a different `attempt_id`. | Preserved |
   | **`3`** | `CONFLICT_INTEGRITY_MISMATCH` | Destination has matching ID but differing SHA-256 hash. | Preserved |
   | **`4`** | `RECOVERY_CONCURRENCY_LOCKED` | Another process holds the advisory recovery lock. | Preserved |
   | **`5`** | `UNSUPPORTED_SCHEMA_VERSION` | Journal or export schema version unsupported. | Preserved |
   | **`6`** | `CLEANUP_PENDING` | Export durable and verified, but journal removal failed. | Preserved for retry |

## Alternatives Considered
- **Direct SQLite Database Querying by Evaluators / Future LMS:** Rejected. Exposing internal database tables couples external consumers to `live-quiz` schema migrations.
- **Direct In-Place File Overwriting:** Rejected. If a process crash occurs during file write, the destination file is left partially written and corrupted.

## Consequences
- **Positive:** Complete data durability; forward compatibility with any future LMS or evaluator; safe atomic handoff to agent callers.
- **Negative / Trade-offs:** Requires an atomic rename step and validation pass prior to session journal cleanup.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
- [ADR-0006: Ephemeral SQLite Session Journal & Recovery](./0006-ephemeral-sqlite-session-journal.md)
