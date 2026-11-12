# Interactive Assessment System Architecture Specification

**Product Name:** `live-quiz`  
**Repository:** `/home/kiskaadee/Projects/active/live-quiz`  
**Status:** Approved Architectural Baseline  

---

## 1. Problem Definition & Core Intent

Conversational AI tutoring in typical chat interfaces lacks a stable, invariant assessment contract. During freeform dialogues, each response mutates conversational context, leading to prompt drift, non-deterministic difficulty adjustments, and conversational generosity. Tutors cannot reliably measure performance against an uncompromised standard because the measurement instrument itself shifts in response to learner struggle.

`live-quiz` resolves this by decoupling **pedagogical authoring and evaluation** (handled by AI agent workflows) from **runtime assessment administration** (handled by a lightweight, deterministic application).

```
               [ Learning Context / Instruction ]
                                │
                                ▼
                   [ External Learning System ]
     ┌───────────────────────────────────────────────────────┐
     │ • Canonical Catalog (Competencies, Objectives, IDs)   │
     │ • Learner History (Observations, Contexts, Evidence)  │◄────────────┐
     └───────────────────────────────────────────────────────┘             │
                                │                                          │
                                ▼                                          │
                  `learning-objectives` / `tutor`                          │
               (Candidate search & contextual scoping)                     │
                                │                                          │
                                ▼ (Assessment Milestone & Learner Consent) │
                        Assessment Brief                                   │
                                │                                          │
                                ▼                                          │
                      `quiz-designer` Skill                                │
                                │                                          │
               ┌────────────────┴────────────────┐                         │
               ▼                                 ▼                         │
     `questionnaire.json`                  `rubric.json`                   │
     (Public Presentation                  (Private Evaluation             │
      + Canonical Objective Refs)           + Deterministic Obj Mappings)  │
               │                                 │                         │
               ▼                                 │                         │
     [ `live-quiz` Engine ]                      │                         │
     • Standalone Local Web App                  │                         │
     • Zero LLM at runtime                       │                         │
     • Ephemeral SQLite Journal                  │                         │
     • Authoritative Server Timer                │                         │
     • Structural Traceability of Obj Refs       │                         │
               │                                 │                         │
               │ (Atomic Export attempt.json)    │                         │
               ▼                                 ▼                         │
        `attempt.json`  ──────────────►  [ `quiz-evaluator` Skill ]        │
     (Preserves Question-to-Objective            │ (Deterministic          │
      Response Mappings Verbatim)                ▼  Correlation)           │
                                         `evaluation.json`                 │
                                                 │                         │
                                                 ▼                         │
                                           [ `tutor` Skill ]               │
                                       (3-Stage Socratic Debrief)          │
                                                 │                         │
                                ┌────────────────┴────────────────┐        │
                                ▼                                 ▼        │
                    [ Cognitive Consolidation ]        (Logs Observations  │
                   (Selective Project Docs Only)        & Evidence) ───────┘
```
---

## 2. System Responsibilities & Boundaries

The repository distributes two decoupled products:
1. **The Assessment Engine (`live-quiz`):** An AI-agnostic CLI and web application that administers an assessment, tracks timing, preserves drafts, enforces a blind pass, and publishes standardized attempt artifacts.
2. **The Reference Skills Collection (`skills/`):** Five portable agent skills (`learning-objectives`, `quiz-designer`, `live-quiz`, `quiz-evaluator`, and `tutor`) demonstrating how any AI agent environment can create, run, and interpret assessments.

### Component Boundaries

| Component | Responsibility | Inputs | Outputs | Core Invariants |
| :--- | :--- | :--- | :--- | :--- |
| **`learning-objectives`** | Formulates observable, assessable competencies; searches external catalog to propose new records and relationships. | Topic / Goals / Context, Registry candidates | Proposed objectives & evidence criteria | Does not assume mastery; canonical identity established by registry; separates intention from demonstration. |
| **`quiz-designer`** | Compiles an approved Assessment Brief into public and private contracts. | Assessment Brief, Learner history | `questionnaire.json`, `rubric.json` | Max 3-question chains; hermetic context or independent rubrics; no answer leakage; history informs difficulty without equating exposure to mastery. |
| **`live-quiz` Engine** | Ephemeral web server executing assessment, capturing evidence, and preserving objective references. | `questionnaire.json`, CLI flags | `attempt.json`, session status | Never receives `rubric.json`; zero runtime LLM; authoritative server clock; structural traceability of objective references; zero semantic registry queries. |
| **`quiz-evaluator`** | Context-isolated post-hoc evaluation of attempt against frozen rubric. | `rubric.json`, `attempt.json` | `evaluation.json` | Fresh context; purely evaluates $f(\text{rubric}, \text{attempt})$ without chat history; associates evidence with canonical objective references. |
| **`tutor`** | Facilitates learning, proposes assessment scope, logs contextual observations against stable objective IDs, leads debrief. | Learner dialogue, context, Registry/History | Learning interactions, Contextual observations | Does not force assessments; leads Socratic debrief before solution reveal; preserves repeated encounters; exposure is not mastery. |
| **Project Repository** | Selective retention of project-specific technical findings or ADRs. | User / Tutor choice | Project docs / Git commits | No mandatory central vault; documentation follows project conventions. |

---

## 3. Data Contracts & Artifact Topology

The system communicates strictly across four versioned JSON contracts:

1. **`questionnaire.json` (Public Presentation Contract):**  
   Consumed by `live-quiz`. Contains question IDs, markdown prompts, Mermaid diagrams, KaTeX equations, interaction types, option labels, initial code stubs, and associated canonical objective references. *Physically excludes answers, rubrics, and misconceptions.*
2. **`rubric.json` (Private Evaluation Contract):**  
   Retained by the calling agent workflow and supplied only to the evaluator. Contains learning objectives, observable criteria (Complete / Partial / Insufficient), categorical failure modes, misconception signatures, and explicit canonical objective references for each criterion (or a deterministic question-to-criterion mapping) to eliminate ambiguous natural-language inference.
3. **`attempt.json` (Durable Execution Evidence):**  
   Produced by `live-quiz` via atomic publication. Must conform structurally to the versioned attempt schema contract (`attempt.schema.json`). Contains attempt identity, assessment/version hashes, workspace attribution, terminal lifecycle status (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`), active and wall-clock durations, pause event logs, and an array of responses preserving question-level canonical objective references verbatim. Every item in the responses array explicitly records its interaction state via the **Response State Triad**:
   - `SUBMITTED`: Explicitly confirmed by the learner (including explicit blank submissions).
   - `DRAFT`: Latest acknowledged autosave payload and receipt timestamp preserved at session termination, even if unsubmitted. A draft represents an acknowledged learner-generated modification (initial empty fields remain `UNANSWERED`). (Per-question active duration is provisional pending measurement definitions).
   - `UNANSWERED`: Zero acknowledged learner input recorded (payload is null).
   *Valid Lifecycle & Response Combinations:*
   While interaction status and demonstrated criterion attainment remain independent dimensions, lifecycle outcomes constrain valid response combinations:
   - `COMPLETED`: Strictly requires every questionnaire item to have `response_status: SUBMITTED` (explicit blank submissions qualify).
   - `CLOSED_EARLY`: At least one response must be `DRAFT` or `UNANSWERED`, following an explicit learner termination action.
   - `TIMED_OUT`: At least one response must be `DRAFT` or `UNANSWERED`, following authoritative timer expiration.
   *Generalized Server-Side Serialization & Absorbing Terminal States:* All response mutations (autosaves and submissions) and terminal transitions (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`) are serialized server-side within SQLite transactions. The final required response submission and transition to `COMPLETED` must be committed atomically. Once any terminal lifecycle transition commits, its status is strictly immutable and absorbing: all subsequent response mutations and competing terminal transition attempts are rejected.
4. **`evaluation.json` (Diagnostic Assessment Report):**  
   Produced by `quiz-evaluator`. Contains criterion-level attainment ratings correlated deterministically to canonical objective references, failure-mode classifications (`CONCEPTUAL`, `LOGICAL`, `SYNTACTIC`, `OMISSION`, `REGRESSION`), misconception hypotheses with confidence levels, and discussion probes for debriefing.
   *Evaluator Contract Across the Response State Triad:*
   - Explicit Blank `SUBMITTED`: Evaluated as an intentional empty response against rubric criteria.
   - `DRAFT`: Evaluated as available evidence of learning (identifying valid intermediate reasoning and progress) without automatic penalty for draft status. Discloses draft provenance, avoids hallucinating what the learner would have written with more time, and distinguishes demonstrated criteria from unestablished criteria.
   - `UNANSWERED`: Reports that no response evidence exists. Must not infer a misconception or conclude that the learner lacks underlying competency solely because no response was recorded.

---

## 4. Assessment Administration & Interaction Model

### 4.1 Composable Interaction Primitives
`live-quiz` implements four canonical interaction primitives capable of expressing all assessment challenges without widget bloat:
- `select`: Single-choice (radio) or multiple-choice (checkbox).
- `text`: Single-line or multi-line markdown text area for open-ended explanations.
- `code`: Static code editor (CodeMirror/Monaco) with read-only context/preamble and an editable buffer. **Zero runtime code execution or test-runner safety net.**
- `mapping`: Ordered list sequencing or key-value association.

The frontend is packaged as a static single-page application built via Vite Plus (`vp`) with all third-party libraries (CodeMirror, Mermaid.js, KaTeX) pre-bundled into `dist/`.

### 4.2 Question Chaining & Progression
- **Chain Length Limit:** Maximum of **3 questions** per chain.
- **Context Sharing:** Questions may share an overarching technical scenario or stimulus.
- **Independent Rubrics:** Each question evaluates its own distinct learning objective. Later questions provide sufficient canonical context to remain answerable even if earlier questions were missed, strictly preventing compound penalization (double jeopardy).

### 4.3 Blind Pass Invariant
The runtime provides **zero immediate correctness feedback, validation cues, or hints** during administration. The learner demonstrates unassisted performance against the frozen standard.

---

## 5. Persistence, State Machine & Process Lifecycle

### 5.1 Three Distinct Lifecycles
The system explicitly separates:
- **Server Process:** Ephemeral HTTP server run on demand.
- **Assessment Instance:** Persistent concept identified by stable IDs and immutable version hashes.
- **Assessment Attempt:** A single execution pass against an immutable assessment version.

### 5.2 Ephemeral SQLite Session Journal
During active assessment, `live-quiz` maintains state in a temporary SQLite database (`~/.local/state/live-quiz/sessions/<attempt-id>.sqlite3`):
- Server commits acknowledged draft updates, answer submissions, and timing events atomically to SQLite.
- **Active-Time Representation:** Accumulated active time is tracked and persisted as an integer nanosecond scalar (`accumulated_active_ns`) in the session state table, converted to seconds only for UI display and `attempt.json` export.
- **In-Memory Anchor Advancement Invariant:**
  $$\Delta t = t_{\text{monotonic now}} - t_{\text{last committed}}$$
  $$\text{accumulated\_active\_ns}' = \text{accumulated\_active\_ns} + \Delta t$$
  The in-memory anchor `last_committed_checkpoint_ns` advances strictly **after** the SQLite transaction commits successfully. If commit fails, the anchor does not advance.
- **Checkpoint Triggers & Durability Guarantee:** Checkpoints target approximately 1 second while `RUNNING`, with synchronous immediate checkpoints on pause, autosave, submission, timeout, and early close (all sharing the serialized write boundary `BEGIN IMMEDIATE`). Recovery resumes from the last successfully committed active-time checkpoint; uncommitted elapsed time is not guaranteed to be preserved under sudden process crashes or machine load.
- **Durable Pause Checkpoint & In-Browser Resume:** Clicking **Pause** commits state, freezes active time, records `status = PAUSED` in SQLite, and displays an in-browser Resume control.
- **Configurable Pause Grace Period (Default: 2 Hours):** The server remains alive and accessible in the browser during a 2-hour window (`--pause-grace-period 2h`). If resumed within this window, the assessment continues instantly in the browser.
- **Grace-Period Expiration & Boot Identity:** Pause grace is measured using Linux `CLOCK_BOOTTIME` (which advances across system sleep). Stored metadata includes `{ boot_id, pause_started_boottime_ns, pause_deadline_boottime_ns }`. If the 2-hour grace period expires without resumption on the same boot, the server process terminates cleanly to prevent zombie processes, retaining the journal with status `PAUSED` and emitting `outcome: SUSPENDED` on stdout. Across a machine reboot, the original in-browser server is dead, the persisted boottime deadline is never compared across different boot IDs, and the journal remains intact for manual CLI recovery.
- Unexpected kills (`SIGKILL`, power failure) leave the journal intact for recovery.

### 5.3 Dual-Clock Timing Architecture & Temporal Edge Cases
1. **Definition of Active Assessment Time:**
   Active assessment time is accumulated while the assessment is `RUNNING`, the engine is operational, and the host system is awake. It does not accumulate while `PAUSED`, the host is suspended, or the engine is terminated:

   | Situation | Assessment State | Active Budget | Pause Grace Window |
   | :--- | :--- | :--- | :--- |
   | Learner answering questions | `RUNNING` | Decreases | Not applicable (not paused) |
   | Browser tab hidden / backgrounded | `RUNNING` | Decreases | Not applicable (not paused) |
   | Host system suspended (laptop lid closed) | `RUNNING` | **Frozen** | Not applicable (not paused) |
   | Learner explicitly pauses | `PAUSED` | **Frozen** | Elapses toward deadline |
   | Host system suspended while paused | `PAUSED` | **Frozen** | Elapses toward deadline |
   | Engine crashes or terminates | Last persisted checkpoint | **Frozen** | In-browser server exited (persisted boottime deadline continues elapsing on same boot only if last persisted state was PAUSED) |

2. **Dual-Clock System & Linux Platform Baseline:**
   - **Active Assessment Budget (`CLOCK_MONOTONIC`):** Sampled via `time.monotonic_ns()`. Unaffected by discontinuous wall-clock shifts (manual time adjustments, NTP steps), though subject to gradual frequency adjustments (NTP slewing). Crucially, it halts during kernel suspend (ACPI S3/sleep), automatically freezing the assessment budget when a laptop lid closes without requiring OS D-Bus or `systemd-logind` hooks.
   - **Operational Pause Grace Timer (`CLOCK_BOOTTIME`):** Sampled via `time.clock_gettime_ns(time.CLOCK_BOOTTIME)`. Advances during kernel suspend, ensuring that laptop sleep elapses the in-browser keepalive window.
   - **BOOTTIME-Aware Pause-Grace Watchdog:** Because Python's `asyncio` event loop schedules delayed timers via `loop.time()` (which uses `CLOCK_MONOTONIC` and halts during suspend), an ordinary `asyncio.sleep()` cannot reliably fire a boottime deadline. The pause-grace watchdog re-evaluates the persisted `CLOCK_BOOTTIME` deadline at the first post-wake watchdog execution (triggered when a suspend gap is detected or via a bounded periodic check), executing clean shutdown with `outcome: SUSPENDED` if the deadline has passed while asleep.
3. **Autonomous Watchdog, Immutable Time Limits & Serialized Admission:**
   - **Immutable Attempt Time Limit (`time_limit_ns`):** The assessment duration budget is immutable for each attempt. It is persisted in SQLite session state at attempt initialization and traceable to the immutable questionnaire version. Recovery must never load altered CLI defaults or application configurations that silently mutate an existing attempt's time budget.
   - An `asyncio` background watchdog in FastAPI monitors the remaining active budget. When the deadline is reached, it initiates the terminal transition to `TIMED_OUT` without waiting for client interaction.
   - **Serialized Mutation Admission Rule:** Every mutation endpoint (`POST /api/response/autosave`, `POST /api/response/submit`) independently validates the deadline while holding the SQLite write-serialization boundary (`BEGIN IMMEDIATE`). Monotonic time is sampled *after* acquiring the write lock:
     - $T_{\text{elapsed}} < T_{\text{limit}} \implies$ mutation accepted and checkpointed.
     - $T_{\text{elapsed}} \ge T_{\text{limit}} \implies$ mutation rejected; endpoint commits `TIMED_OUT` and `publication_intent`.
   - A delay in the background watchdog cannot allow late submissions to be accepted.
   - **Overdue Recovery Guard:** If `live-quiz recover` inspects a journal where `accumulated_active_ns >= time_limit_ns`, the engine immediately terminalizes the attempt as `TIMED_OUT` and initiates atomic export before any interactive resume is permitted.
4. **System Suspend Gap Detection:**
   - The engine periodically compares clock deltas:
     $$T_{\text{suspended}} \approx \Delta T_{\text{BOOTTIME}} - \Delta T_{\text{MONOTONIC}}$$
   - When a positive discrepancy exceeds a threshold (indicating the machine slept), the engine appends a `SUSPEND_GAP_DETECTED` event to the `session_intervals` audit table upon wake.
   - `session_intervals` table uses `event_sequence INTEGER PRIMARY KEY AUTOINCREMENT` to guarantee strictly monotonic event ordering independently of wall-clock accuracy.
5. **Browser Presentation & Re-synchronization:**
   - The browser UI is a presentation layer only. Countdown is calculated locally based on the latest server payload (`remaining_seconds`, `is_active`).
   - When a tab is hidden, local visual updates halt to avoid burning CPU on throttled timers.
   - The client listens to the Page Visibility API (`document.visibilityState === 'visible'`) and window reconnect events, issuing `GET /api/session` upon re-entry to resynchronize the countdown without continuous background polling.
6. **Threat Model & Trust Boundary:**
   The timing architecture is **server-authoritative relative to the browser**, guarding against client clock tampering, background timer throttling, network latency games, and wall-clock skew. Host-level root or user tampering (e.g. modifying SQLite directly or patching the Python binary) is explicitly outside the threat model of a local, formative personal assessment tool.

### 5.4 Ten-Step Atomic Publication Pipeline & Durability Contract
Upon reaching a terminal state (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`), publication executes through a strict 10-step durability pipeline protected by dual advisory locks (per-attempt and destination-level):
1. **Dual Advisory Concurrency Locks:** Acquire exclusive locks via `fcntl`/`flock` on:
   - Per-attempt lock file (`~/.local/state/live-quiz/locks/attempt-<attempt-id>.lock`), coordinating publication and recovery.
   - Destination-level lock file (`~/.local/state/live-quiz/locks/dest-<sha256_of_destination_path>.lock`), coordinating publications across different attempts targeting the same path.
   Locks are retained throughout publication and cleanup, and are never unlinked while held.
2. **Transaction-Bounded Payload Preparation & Canonical Serialization:**
   - Execute `BEGIN IMMEDIATE` on the writable SQLite connection to establish the write-serialization boundary before snapshot extraction.
   - **Path A (Final Submission → `COMPLETED`):** The final question submission mutation (`status = SUBMITTED`), the transition to `COMPLETED`, the complete response snapshot, and `publication_intent` are constructed and committed within this single transaction. The final response is never acknowledged as submitted without atomically committing assessment completion.
   - **Path B (Timeout / Early Close):** The transaction lock is established before extracting the snapshot (capturing current drafts and unanswered questions) and committing `TIMED_OUT` or `CLOSED_EARLY` alongside `publication_intent`.
   - Validate payload structurally against target `attempt.schema.json`. Serialize the canonical payload using RFC 8785 Canonical JSON (deterministic key ordering, UTF-8, normalized numbers) and calculate `expected_sha256`.
3. **Atomic Terminal Commitment & Publication Intent:** Within the open transaction, atomically commit:
   - Terminal lifecycle state (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`).
   - Final response snapshot (incorporating the final submission in Path A).
   - Immutable `publication_intent`: `target_schema_version`, `export_projection_version: "1"`, `destination_path`, `canonicalization_version: "1"`, `expected_sha256`.
   - Execute `COMMIT;`. Filesystem publication then proceeds using the already-determined canonical bytes.
4. **Destination Conflict Pre-Check:** Under the destination lock, inspect `destination_path`. If an artifact exists with a different `attempt_id`, abort immediately with `CONFLICT_DIFFERENT_ATTEMPT` (exit 2) without writing or renaming. Overwriting another attempt's artifact is strictly prohibited.
5. **Attempt-Specific Temporary File Write & Fsync:** Write canonical bytes to `.attempt.<attempt-id>.tmp` in the destination directory, flush, and `fsync` the file descriptor.
6. **Atomic Rename / Link:** Atomically rename `.attempt.<attempt-id>.tmp` to `attempt.json` (on Linux, using `renameat2(..., RENAME_NOREPLACE)` or atomic link/rename guarantees no race overwrites another attempt's artifact).
7. **Destination Directory Fsync:** Open parent directory descriptor and `fsync` directory metadata, guaranteeing rename persistence across power failures.
8. **Verified WAL Checkpoint:** Through the open writable connection, execute passive/truncate WAL checkpoint (`PRAGMA wal_checkpoint(TRUNCATE);`) and verify completion (`busy == 0, log == 0`).
9. **Clean Close & Ordered Journal Cleanup:** Close all database handles cleanly. SQLite removes/truncates `-shm` and `-wal`. Then verify companion removal (unlinking `-wal` and `-shm` explicitly if still present), and unlink the primary `.sqlite3` database file **last**.
10. **Structured Process Outcome:** Emit single-line JSON outcome packet on stdout and exit 0 (`SUCCESS_FINALIZED`).

*Operational Decoupling:*
- If publication fails during steps 1–7 (e.g. disk full, permission denied, destination conflict, directory sync failure), the process exits with `EXPORT_IO_ERROR` (code 1) or conflict codes (2, 3) and the complete journal file set remains intact.
- If publication succeeds through step 7 but checkpoint or journal cleanup (steps 8–9) fails, the process exits with `CLEANUP_PENDING` (code 6), signaling that `attempt.json` is durable and verified, but journal removal requires retry.

### 5.5 Two Distinct Recovery Paths & Idempotency Verification
Invoking `live-quiz recover --from-path <journal.sqlite3>` acquires the shared advisory locks (per-attempt and destination lock) and routes deterministically:
1. **Nonterminal Recovery (`RUNNING` or `PAUSED`):**
   - Applies in-place transactional migrations if the internal journal schema version differs from the current engine.
   - Restores the authoritative countdown clock, re-mounts acknowledged drafts, resumes the local HTTP server, and displays an in-browser resume control under the exact same `attempt_id`.
2. **Terminal Export Recovery (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`):**
   - **Phase 1: Read-Only Historical Extraction & Reconstruction:**
     - Opens SQLite with a read-only connection (`mode=ro`), reading existing `-wal` and `-shm` frames without mutating user tables or schema on disk.
     - Reads persisted `publication_intent`. Legacy journals lacking `publication_intent` are explicitly rejected as unsupported (`UNSUPPORTED_SCHEMA_VERSION`, exit 5) because their target schema and reference hash cannot be verified deterministically.
     - Reconstructs canonical bytes using the versioned export projection (`export_projection_version`). Before publishing, recovery calculates the SHA-256 digest of the reconstructed bytes and verifies that it strictly matches `expected_sha256`. If a mismatch occurs, recovery halts closed (`CONFLICT_INTEGRITY_MISMATCH`, exit 3) and preserves the journal.
   - **Idempotency Verification against Persisted Hash:**
     - *Artifact does not exist:* Publishes reconstructed canonical bytes via steps 4–7.
     - *Artifact exists with matching `attempt_id` and matching `expected_sha256`:* Previous publication succeeded; advances directly to Phase 2 cleanup.
     - *Artifact exists with different `attempt_id`:* Halts immediately, preserves journal, emits `CONFLICT_DIFFERENT_ATTEMPT`, and exits code 2.
     - *Artifact exists with matching `attempt_id` but differing SHA-256:* Halts immediately, preserves journal, emits `CONFLICT_INTEGRITY_MISMATCH`, and exits code 3.
   - **Phase 2: Post-Verification Writable Cleanup Connection:**
     - Opened only after Phase 1's read-only connection and read transaction are closed cleanly, ensuring no lingering reader locks prevent checkpointing.
     - Opened only after `attempt.json` has been durably published, verified against `expected_sha256`, and destination directory `fsync` has completed.
     - Connects in writable mode solely to execute `PRAGMA wal_checkpoint(TRUNCATE);` with verified completion (`busy == 0, log == 0`). Schema migrations or table mutations are strictly prohibited on this connection.
     - Closes connection cleanly, unlinks companion files (`-wal`, `-shm`), unlinks `.sqlite3` **last**, emits `SUCCESS_FINALIZED`, and exits 0. If checkpointing fails, journal is kept intact and process exits with `CLEANUP_PENDING` (code 6).
   - **Partial Cleanup Invariant:** Because cleanup unlinks companion files first and `.sqlite3` last, orphaned `-wal` or `-shm` files without `.sqlite3` represent an unexpected anomaly. Recovery halts safely and reports an integrity error rather than silently deleting orphaned files.

---

## 6. Pedagogical Scoping & Debrief Protocol

### 6.1 Canonical Objectives, Learning History & Assessment Scoping
- **Integration Boundary:** Learning workflows resolve competencies and objectives against a persistent external catalog, while learner interactions accumulate as contextual observations in a longitudinal learning history. `live-quiz` has no semantic awareness of the registry, performs no registry queries or identity resolution, and manages no longitudinal history; it merely provides structural traceability of objective references across questionnaire execution and attempt export.
- **Competencies vs. Contextualized Objectives:** The external catalog maintains a many-to-many relationship between broad, reusable competencies (e.g. *Reason about HTTP caching semantics*) and observable, contextualized learning objectives (e.g. *Explain how HTTP caching headers influence browser behavior*).
- **Objective Ledger as In-Context View:** Within active sessions, the Tutor maintains an in-context ledger referencing stable objective identities. Deferral (`ACTIVE` vs. `DEFERRED`) is an independent pedagogical disposition that does not erase or overwrite previous learning observations.
- **Exposure Informs Design, Not Mastery:** Prior observations (e.g. `INTRODUCED`, `EXPLORED`, `PRACTICED` as provisional examples) inform the Quiz Designer regarding familiarity and context novelty to avoid redundant introductory questioning. However, exposure is never equated to mastery: difficulty calibration must preserve distinctions between familiarity, task complexity, context novelty, assistance level, and demonstrated independent performance.
- **Scoping Gate via Assessment Brief:** Assessment readiness is never an intrinsic or permanent flag on an objective. It is determined dynamically when the Tutor compiles a candidate **Assessment Brief** (identifying selected canonical objectives, performance criteria, context, and intended difficulty), which the learner explicitly reviews and approves before authoring begins.
- **Deterministic Evaluator Mapping:** The assessment contract guarantees deterministic correlation between rubric criteria, question responses, and canonical objective identities without requiring `quiz-evaluator` to receive `questionnaire.json` or guess objective associations from similar wording.
- **Unresolved System Questions:** The exact structural schema mapping of objective references across `questionnaire.json`, `rubric.json`, and `attempt.json` remains to be finalized. Furthermore, detailed registry architecture—including the authoritative storage location and synchronization strategy for learner observations (e.g. interim session-local logfiles vs. centralized store), relationship semantics (e.g. refining `enables` beyond a prerequisite DAG), dynamic roles of terminal vs. enabling objectives, and concrete observation schemas—belongs to the broader learning-system architecture and remains explicitly open.

### 6.2 Three-Stage Socratic Debrief
1. **Results Overview:** High-level summary of overall attainment, demonstrated competencies, and friction points without revealing answers.
2. **Guided Socratic Reflection:** Targeted diagnostic questions exploring identified misconceptions, allowing the learner to formulate reasoning first.
3. **Consolidation:** Canonical explanations revealed, misunderstandings clarified, and future practice priorities established.
4. **Selective Documentation:** Debrief is a cognitive consolidation dialogue. Durable technical notes or ADRs are committed selectively to the active project repository; no mandatory universal vault summaries are generated.
