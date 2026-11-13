# Issue #4 Authoritative Decision Ledger

**Project:** `live-quiz`  
**Scope:** Authoritative provenance record, settled architectural baseline, accepted `B2` decisions, unapproved `step 850` recommendations (`S1-01`–`S1-17`), deferred implementation boundaries (`DEF-01`–`DEF-05`), and maintenance protocol for Original Issue #4.  
**Repository Baseline:** `main` @ `a4eeccf` ([docs/architecture.md](../docs/architecture.md), [docs/decisions/0001`–`0009](../docs/decisions/README.md)).  
**Companion Artifact:** [issue-4-follow-through.md](issue-4-follow-through.md) (authoritative for execution order, dependencies, and work status).

---

## Section 0 — Authority and Provenance Rules

This ledger governs **decision state and provenance** for Issue #4, while [issue-4-follow-through.md](issue-4-follow-through.md) governs **remaining work, sequencing, and completion conditions**. Together they provide a self-contained handoff baseline without requiring prior chat history.

### 0.1 Four Provenance Categories

1. **Explicitly Accepted User Decisions (`USER_EXPLICIT`):**
   - Commitments explicitly approved or stipulated by the user in direct `USER_INPUT` turns (verified in `transcript_full.jsonl` at steps `820`, `825`, `827`, `829`, `845`, `847`, and `849`).
   - **Rule:** Explicit user input is the sole source of decision authority. These commitments are locked and must not be re-opened unless the user explicitly asks to revise them.

2. **Previously Established Architectural Invariants:**
   - The committed repository architecture on `main` @ `a4eeccf` ([docs/architecture.md](../docs/architecture.md) and [ADR-0001](../docs/decisions/0001-standalone-web-application.md) through [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), resolving Issues #1–#3 and #5–#8) together with the foundational Issue #4 handoff baseline (`steps 820–827`, recorded in Section 1 below).
   - **Rule:** Downstream decisions must respect these invariants. Accepting a high-level architectural direction does **not** implicitly approve unreviewed sub-mechanisms or implementation details attached to it later.

3. **Assistant / Agent Recommendations Awaiting Approval (`UNAPPROVED_RECOMMENDATION — step 850`):**
   - In `step 850` of the discovery transcript, a single assistant turn simulated synthetic user prompts and emitted a premature "Stage 1 Decision Closure Report" marking items `B2.4.1` through `E2` as `Accepted`.
   - **Rule:** A recommendation marked `Accepted` in that earlier assistant-generated closure report is **not** an approved decision. The transcript audit and [issue-4-follow-through.md](issue-4-follow-through.md) Section 0 govern. All 17 atomic items (`S1-01`–`S1-17`, recorded in Section 3 below) remain `PENDING_USER_REVIEW`.

4. **Deferred Implementation Decisions (`DEF-01`–`DEF-05`):**
   - Concrete engineering specifications (exact JSON field names, library selection, CLI flag spellings, wire framing, systemd unit parameters, backup retention numbers) intentionally postponed until after Stage 3 acceptance criteria (`S3-01`).
   - **Rule:** Deferred is not equivalent to decided. An architectural invariant can be accepted while its implementation mechanism remains open; deferred details must not be prematurely written into ADRs as settled architectural commitments.

---

## Section 1 — Settled Architectural Baseline

The following nine foundational commitments (`BL-01`–`BL-09`) are verified in `steps 820, 825, 827` (`USER_EXPLICIT`). Important qualifications are preserved so accepted principles are not broadened into commitments the user did not make.

| Baseline ID | Established Invariant & Scope | Rationale | Preserved Qualifications & Open Boundaries | Target ADR / Doc Destinations |
| :--- | :--- | :--- | :--- | :--- |
| **`BL-01`**<br>*(Cluster A)* | **Immutable Coordinated Assessment Pair & Flat `assessment_id`:**<br>An assessment definition is an immutable coordinated pair $A = (Q, R)$ (`questionnaire.json` + `rubric.json`) identified by a flat opaque `assessment_id` allocated by `learningctl`; modification of either member of the coordinated assessment pair requires registering a new `assessment_id`. Competency $\leftrightarrow$ Objective and Assessment $\leftrightarrow$ Objective relationships are M:N (Competency $\leftrightarrow$ Assessment is derived, never direct). The Tutor owns pedagogical reassessment decisions, performs an `AVAILABLE` artifact pre-flight check before offering reassessment, and requires explicit learner consent before registering a new assessment or starting a reassessment. | Prevents silent drift between questions and grading criteria, decouples assessment identity from curriculum taxonomy edits, and keeps pedagogical judgment in the Tutor rather than the quiz engine. | How mechanical schema up-converters (`clone-upgrade`) are restricted when contract versions change remains open under `S1-05` (`B2.4.5`). | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-02`**<br>*(Cluster A)* | **No `assessment_version` or Version Lineage:**<br>`assessment_version`, revision chains, and parent/child lineage metadata are eliminated across all contracts. Modification of either member ($Q$ or $R$) requires registering a new `assessment_id` without exception. Longitudinal learner progress across assessments is tracked exclusively through shared `learning_objective_id` links. | Avoids version-graph complexity and invalid longitudinal comparisons across mutated assessments while preserving objective-level progress tracking. | Legacy `attempt.json` (`assessment_version`) schema rejection policy is tracked separately under `S1-08` (`C2`). | [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-03`**<br>*(Cluster B1)* | **`B1` Identity Binding & Hash Domain:**<br>Registered $Q$ and $R$ both embed `assessment_id`. $R$ embeds `expected_questionnaire_hash` ($H_q$). $H_q = \operatorname{SHA256}(\operatorname{JCS}(Q))$ is calculated over the complete validated public `questionnaire.json` (including `assessment_id` when present). `attempt.json` embeds `questionnaire_hash` ($H_q$) for definition identity and retains `expected_sha256` ([ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md)) as a distinct per-file transport checksum. Standalone execution (`live-quiz start --questionnaire <path>`) is supported without `assessment_id` or `workspace_id`. | Cryptographically binds $R \to Q$ and `attempt.json` $\to Q$ without exposing $R$ to `live-quiz` ([ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md)), while preserving standalone CLI usability ([ADR-0001](../docs/decisions/0001-standalone-web-application.md)). | Component-level hash calculation/verification authority (`S1-06` / `B2.5.1`) and canonical byte storage (`S1-07` / `B2.5.2`) remain open. | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-04`**<br>*(Cluster C1)* | **`C1` Self-Contained Journal Snapshot & Recovery Independence:**<br>Before serving any learner-facing HTTP route, `live-quiz` durably commits a complete, self-contained questionnaire snapshot and initial execution metadata (`attempt_id`, optional `assessment_id`, $H_q$, optional `workspace_id`, `time_limit_ns`) to the per-attempt SQLite journal. `live-quiz recover` reconstructs the session strictly from the journal snapshot without reading external questionnaire files or querying the Learning Registry. | Guarantees crash/restart self-sufficiency even after temporary materialized questionnaires are deleted or workspace files move. | Exact persisted snapshot format (`S1-07` / `B2.5.2`) and the startup readiness acknowledgment contract (`S1-12` / `D2b`) remain open. | [ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), [docs/architecture.md](../docs/architecture.md) |
| **`BL-05`**<br>*(Cluster C2 partial)* | **Explicitly Accepted Portion of `C2` — No Historical Journal Migration:**<br>No backward-compatibility migration or fallback reconstruction path is needed for historical attempt journals lacking embedded questionnaire snapshots, because no prior deployment exists. | Eliminates dead migration complexity in a greenfield pre-implementation repository. | **Qualification:** Only journal migration is settled here. Legacy `attempt.json` (`assessment_version`) handling remains open under `S1-08` (`C2`). | [ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md) |
| **`BL-06`**<br>*(Cluster D baseline)* | **Foundational Learning Registry, Workspace, Encryption, Provisioning & Service-Boundary Invariants:**<br>• One OS-wide persistent encrypted Learning Registry serves all workspaces on the host.<br>• Workspaces are logical provenance resolved via nearest-ancestor `.learning-workspace` marker (`workspace_id`), explicitly rejecting Git root as workspace boundary.<br>• Host provisioning is separate from `learningctl init` (`init` registers a workspace and never silently recreates a missing global DB).<br>• Agents/skills invoke only `learningctl` (narrow domain CLI; black-box `learningctl.md`/`--help` contract) $\to$ private typed UDS IPC $\to$ `learning-registryd` (`learning-registry` OS user, systemd socket activation `Accept=no`, serialized mutations).<br>• Non-TPM default host key protection stored outside workspaces and separate from DB backups; `learning-registryd` owns Online Backup API encrypted-to-encrypted snapshots on schedule and before disruptive operations.<br>• `learningctl --json` responses carry per-command `schema_version` with additive/breaking compatibility semantics. | Preserves cross-topic learner state across directories, isolates secrets (`rubric.json`) and DB credentials from workspace files and agent shells, and prevents concurrent agent races against the encrypted registry. | **Qualifications:**<br>• `SQLCipher` remains a **candidate**, not a selected implementation.<br>• Exact `"<major>.<minor>"` CLI `schema_version` string syntax remains unverified (`S1-14` / `D3b`, flagged open at `step 827`).<br>• Repo ownership (`S1-09` / `D1a`), marker/error taxonomy (`S1-10` / `D1b`), service delegation cutoff (`S1-14` / `D3b`), and key escrow / backup verification (`S1-15` / `D4`) remain open. | [ADR-0008](../docs/decisions/0008-two-product-repository-architecture.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-07`**<br>*(Cluster D baseline)* | **Foundational Registration & Temporary Questionnaire Handoff Rules:**<br>Registration uses a two-phase lifecycle (`REGISTERING` $\to$ `AVAILABLE` / `ABORTED`). Only `AVAILABLE` definitions are discoverable for execution/reassessment; allocated `assessment_id`s are never reused; registration failures are strictly distinct from attempt crash recovery. For managed runs, `learningctl` materializes a temporary public-only `questionnaire.json` outside the workspace and deletes it idempotently after `live-quiz` commits its durable journal snapshot. | Prevents half-registered pairs from being executed, keeps `rubric.json` inside the registry service boundary, and avoids leaving stale questionnaire files in learner workspaces. | `REGISTERING` status visibility and `ABORTED` tombstone retention (`S1-11` / `D2a`) and the startup-commit acknowledgment protocol (`S1-12` / `D2b`) remain open. | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-08`**<br>*(Cluster D baseline)* | **Foundational Status-Discovery & Attempt-State Authority:**<br>`learningctl status` (`--json`, `--all`) is a read-only command run once at Tutor initialization (no polling loops). `learningctl` discovers candidate attempt journals, while `live-quiz` is the sole authority for active vs. recoverable attempt state without exposing private engine SQLite tables. | Gives the Tutor a single deterministic session-start snapshot while respecting the engine's private persistence encapsulation ([ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md)). | The exact read-only active-vs.-recoverable inspection interface between `learningctl` and `live-quiz` remains open under `S1-13` (`D3a`). | [ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`BL-09`**<br>*(Cluster E baseline)* | **Foundational Interaction Lifecycle, Reversible Abandonment, Debrief Guideline & Atomic Consolidation:**<br>• In managed workflows, `attempt.json` is staged in a persistent non-workspace staging area (no TTL expiry) until `(attempt, evaluation, Tutor observations)` are committed in one atomic registry transaction (`CONSOLIDATED`) with an idempotent receipt. Standalone `attempt.json` files remain untouched user exports.<br>• States include `PENDING_DEBRIEF`, `DEFERRED`, `PENDING_CONSOLIDATION`, `CONSOLIDATED`, and `ABANDONED`.<br>• Abandonment separates **append-only historical events** from **reversible current disposition** (`ABANDONED` is hidden from default `status` and `--all`, and transitions to `DEFERRED` or a new attempt only on explicit learner request).<br>• Bounded debrief (1 consolidated report + ~1–3 follow-up turns) is a **facilitation guideline**, not a hard technical cutoff; interrupted debriefs store lightweight structured progress, never full chat transcripts. | Prevents partial/un-debriefed quiz exports from polluting canonical mastery records while preserving crash-safe staging, learner agency over abandoned attempts, and transcript-free privacy. | **Qualifications:**<br>• Bounded debrief duration is a pedagogical facilitation guideline, not an enforced turn limit.<br>• Per-execution staging slot allocation (`S1-16` / `E1`) and pre-consolidation registry/staging split, pre-debrief discovery, checkpoint storage & `ABANDONED` payload retention (`S1-17` / `E2`) remain open. | [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), [ADR-0009](../docs/decisions/0009-modular-portable-reference-skills.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |

---

## Section 2 — Explicitly Accepted `B2` Decisions (`B2.1`–`B2.3` & `B2.4` Framing Constraints)

### 2.1 Accepted `B2.1`–`B2.3` Subdecisions

| Decision ID & Provenance | Accepted Decision Statement | Rationale | Principal Architectural Consequences | Target ADR / Doc Destinations |
| :--- | :--- | :--- | :--- | :--- |
| **`B2.1`**<br>`ACCEPTED`<br>*(`USER_EXPLICIT`, step `829`)* | **Hash Original Parsed JSON After Strict Validation:**<br>`questionnaire_hash` ($H_q$) is computed over the original parsed JSON data structure after strict validation (`extra="forbid"`, `strict=True`) succeeds, serialized through RFC 8785 JCS. Pydantic's normalized model output (`model_dump()` / `model_dump_json()`) is never used as the hash input. | Decouples the cryptographic identity of an assessment from internal Python/Pydantic model refactoring, field ordering, or default-injection behavior. | • Validators must preserve the raw parsed JSON tree (Layer 1 output) alongside or prior to constructing domain models.<br>• Default semantics and contract evolution are governed independently under `B2.4` (`S1-01`–`S1-05`). | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`B2.2`**<br>`ACCEPTED`<br>*(`USER_EXPLICIT`, step `845`)* | **Project-Specific Strict RFC 8785-Compatible Input Profile (Scoped to `questionnaire.json`):**<br>`questionnaire.json` must satisfy a strict project-specific JCS-compatible input profile at the parsing boundary:<br>1. Valid UTF-8 without BOM; reject unpaired UTF-16 surrogates and Unicode noncharacters; **no Unicode normalization** (such as NFC or NFD) is performed, and decoded code points are preserved exactly (NFC- or NFD-normalized input is accepted as-is without rejection or re-normalization).<br>2. Reject duplicate object keys after string unescaping (`object_pairs_hook`).<br>3. Reject `NaN`, `Infinity`, and `-Infinity` (`parse_constant`).<br>4. Domain restriction: reject all floating-point and exponential lexical tokens (`parse_float`, rejecting `600.0`, `1e0`, `1e400`) and enforce exact integers within $[-(2^{53}-1), +(2^{53}-1)]$ (`parse_int`).<br>5. Scope restriction: the integer-only restriction applies strictly to `questionnaire.json` and is **not** automatically transferred to `attempt.json` or `evaluation.json`. | Eliminates parser-level ambiguity (duplicate keys, float overflow, surrogate edge cases) before canonicalization, preserves exact authored Unicode code points without distorting inputs, and uses an integer-only domain rule where all questionnaire numeric quantities (timers, limits, counts) are naturally discrete. | • Both `live-quiz` and the registry must enforce identical Layer 1 parser hooks on `questionnaire.json`.<br>• `attempt.json` and `evaluation.json` remain free to use finite floats/integers as appropriate to scoring and telemetry.<br>• Shared negative test vectors (`DEF-02`) must cover rejected tokens (`600.0`, `1e0`, `1e400`, duplicate keys, lone surrogates, noncharacters) and positive preservation of NFC/NFD inputs without normalization. | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0005](../docs/decisions/0005-python-fastapi-vite-stack.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |
| **`B2.3`**<br>`ACCEPTED`<br>*(`USER_EXPLICIT`, step `847`)* | **Three-Layer Parsing / Canonicalization / Digest Model with Independent Implementations & Shared CI Conformance Suite:**<br>1. Explicitly separate **Layer 1** (strict JSON parser + `B2.2` input profile), **Layer 2** (full RFC 8785 JCS canonicalizer operating on Layer 1 output to emit deterministic UTF-8 bytes), and **Layer 3** (`SHA-256` lowercase hex digest).<br>2. Keep `live-quiz` and the Learning Registry independently implemented (no shared first-party runtime helper package), with each component using an established, standards-compliant full RFC 8785 library (never a hand-written canonicalizer or a stripped-down integer-only JCS subset).<br>3. Enforce cross-component determinism via a **mandatory shared raw-byte CI conformance test suite**. | Preserves the clean product boundary ([ADR-0008](../docs/decisions/0008-two-product-repository-architecture.md)), avoids custom canonicalizer bugs, and guarantees byte-for-byte agreement across products through shared specification and raw-byte test vectors rather than runtime package coupling. | • Even though Layer 1 rejects floats in `questionnaire.json`, Layer 2 remains a complete standard RFC 8785 JCS serializer.<br>• Exact Python library selection and CI test vector assembly are tracked as `DEF-02`. | [ADR-0004](../docs/decisions/0004-questionnaire-rubric-separation.md), [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md), [ADR-0008](../docs/decisions/0008-two-product-repository-architecture.md), new Issue #4 ADR(s) (`0010`+), [docs/architecture.md](../docs/architecture.md) |

### 2.2 Accepted `B2.4` Framing Constraints (`USER_EXPLICIT`, step `849`)

In `step 849`, the user decomposed `B2.4` into five subdecisions (`B2.4.1`–`B2.4.5`, tracked as `S1-01`–`S1-05`) and established four binding framing constraints:
1. `schema_version` is a top-level field of `questionnaire.json` and is therefore covered by $H_q = \operatorname{SHA256}(\operatorname{JCS}(Q))$.
2. Changing `schema_version` changes $H_q$, which under `BL-01`/`BL-02` requires registering a new `assessment_id` with no version lineage.
3. Registered $(Q, R)$ pairs are never migrated or rewritten in place.
4. Cosmetic UI improvements and internal code refactoring must not require a new `schema_version`.

---

## Section 3 — Unapproved Recommendations Requiring Review (`S1-01` through `S1-17`)

Every entry below originated in `step 850` and requires an explicit user turn assigning a disposition (`ACCEPTED_AS_PROPOSED`, `ACCEPTED_WITH_MODIFICATIONS`, `REJECTED_IN_FAVOR_OF_ALTERNATIVE`, `DEFERRED_WITH_JUSTIFICATION`, or `NOT_APPLICABLE_WITH_JUSTIFICATION`).

### `S1-01` — `B2.4.1` (Contract Version Scope)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-01` |
| **Decision ID** | `B2.4.1` |
| **Decision question** | What exact scope does `questionnaire.json`'s `schema_version` govern (unified document structure + frozen optional defaults + assessment-significant execution semantics vs. structure-only or split version fields), and how is it recorded in the attempt journal and `attempt.json`? |
| **Already-settled constraints** | `BL-03` (`B1` full-document $H_q$), `B2.1` (hash raw parsed JSON), and `step 849` framing constraints (`schema_version` is inside $H_q$). |
| **Current proposal** | Define `schema_version` as a unified **Questionnaire Contract Version** governing structure, frozen optional-field defaults, and assessment-significant execution semantics, and propagate it into the attempt journal snapshot and `attempt.json` provenance (e.g., `questionnaire_schema_version`). |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-01`](issue-4-follow-through.md) |

### `S1-02` — `B2.4.2` (Semantic Immutability Boundary)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-02` |
| **Decision ID** | `B2.4.2` |
| **Decision question** | Which changes to questionnaire interpretation or rendering require a new `schema_version` versus qualifying as contract-preserving implementation improvements? |
| **Already-settled constraints** | `step 849` framing constraint (cosmetic changes and internal refactoring do not require a new contract version). Prerequisite: `S1-01` (`B2.4.1`). |
| **Current proposal** | Require a new `schema_version` for any change altering valid inputs, optional-field meaning, learner interaction mechanics, timing enforcement, or exported response semantics; permit visual/CSS polish, accessibility fixes, internal refactoring, backward-compatible renderer upgrades, and spec-compliance bug fixes within the same version. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-02`](issue-4-follow-through.md) |

### `S1-03` — `B2.4.3` (Defaults and Explicit Configuration)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-03` |
| **Decision ID** | `B2.4.3` |
| **Decision question** | Which behavioral settings in `questionnaire.json` must be explicitly authored, and which optional fields may be omitted with frozen semantics under a contract version? |
| **Already-settled constraints** | `BL-03` (standalone `questionnaire.json` may omit `assessment_id`), `B2.1` (Pydantic defaults are never injected into $H_q$). Prerequisite: `S1-01` (`B2.4.1`). |
| **Current proposal** | Require all assessment-significant behavioral settings (timing policy, question interaction types, selection bounds, objective mappings) to be explicitly authored, allowing omission only where omission permanently means *"absent / not applicable"* (such as omitted `assessment_id` in standalone mode or optional media/stimulus blocks). |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-03`](issue-4-follow-through.md) |

### `S1-04` — `B2.4.4` (Contract Version Support and Retirement)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-04` |
| **Decision ID** | `B2.4.4` |
| **Decision question** | How long must `live-quiz` support historical `schema_version`s, and how do `live-quiz`, the Learning Registry, and the Tutor behave when a contract version is eventually retired? |
| **Already-settled constraints** | `BL-01` (Tutor performs `AVAILABLE` pre-flight check before offering reassessment; registered pairs are immutable). Prerequisites: `S1-01` (`B2.4.1`), `S1-02` (`B2.4.2`). |
| **Current proposal** | Freeze contract version semantics permanently in specifications, allow `live-quiz` to retire execution support only across major releases with fail-closed `UNSUPPORTED_SCHEMA_VERSION` errors on `validate`/`start`/`recover`, keep retired `AVAILABLE` definitions intact in the Registry for historical audit, and include engine `schema_version` support in the Tutor's reassessment pre-flight check. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-04`](issue-4-follow-through.md) |

### `S1-05` — `B2.4.5` (Migration and New Definitions Boundary)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-05` |
| **Decision ID** | `B2.4.5` |
| **Decision question** | Should `learningctl` / `learning-registryd` prohibit mechanical schema up-converters (`clone-upgrade` commands) in addition to prohibiting in-place pair rewrites? |
| **Already-settled constraints** | `BL-01`, `BL-02`, and `step 849` framing constraints (no in-place pair migration; changing `schema_version` yields a new `assessment_id` with no version lineage; longitudinal continuity uses shared `learning_objective_id`s). Prerequisite: `S1-01` (`B2.4.1`). |
| **Current proposal** | Prohibit both in-place rewrites and mechanical CLI up-converters (`clone-upgrade`); when an older assessment uses a retired `schema_version`, require consent-gated authoring and registration of a new $(Q', R')$ pair under a fresh `assessment_id` linked strictly via shared learning objectives. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-05`](issue-4-follow-through.md) |

### `S1-06` — `B2.5.1` (Boundary Hash Calculation and Verification Authority)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-06` |
| **Decision ID** | `B2.5.1` |
| **Decision question** | Which components calculate $H_q$ and which components independently recompute and verify $H_q$ at authoring, registration, startup, recovery, evaluation, and consolidation boundaries? |
| **Already-settled constraints** | `BL-03` (`B1`), `BL-04` (`C1`), `B2.1`–`B2.3` (3-layer strict parser $\to$ JCS $\to$ SHA-256). |
| **Current proposal** | Have `live-quiz validate` output $H_q$ during authoring so `quiz-designer` embeds `expected_questionnaire_hash` in $R$; require `learning-registryd` to independently recompute and verify $H_q$ at registration; require `live-quiz start` to compute and store $H_q$ in the journal snapshot and `live-quiz recover` to re-validate and verify the snapshot digest before serving; and require `quiz-evaluator` and `learning-registryd` consolidation to verify $H_q$ correspondence across $Q$, $R$, `attempt.json`, and `evaluation.json` (while the Tutor never computes $H_q$). |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-06`](issue-4-follow-through.md) |

### `S1-07` — `B2.5.2` (Canonical Questionnaire Persistence Representation)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-07` |
| **Decision ID** | `B2.5.2` |
| **Decision question** | Should the Learning Registry (`AVAILABLE` questionnaire payload) and the `live-quiz` attempt journal (`C1` snapshot) persist **canonical RFC 8785 JCS UTF-8 bytes** alongside $H_q$, or store raw authored JSON text? |
| **Already-settled constraints** | `BL-04` (`C1` self-contained journal snapshot), `BL-07` (temporary questionnaire materialization), `B2.1`–`B2.3`. Prerequisite: `S1-06` (`B2.5.1`). |
| **Current proposal** | Persist exact Layer 2 RFC 8785 JCS canonical UTF-8 bytes alongside $H_q$ in both the Learning Registry and the `live-quiz` attempt journal, and write those stored canonical bytes directly during temporary questionnaire materialization without model re-serialization. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-07`](issue-4-follow-through.md) |

### `S1-08` — `C2` (Legacy `attempt.json` `assessment_version` Policy)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-08` |
| **Decision ID** | `C2` |
| **Decision question** | How should `live-quiz`, `quiz-evaluator`, and `learningctl` treat any `attempt.json` containing the removed `assessment_version` field? |
| **Already-settled constraints** | `BL-02` (`assessment_version` eliminated), `BL-05` (no historical journal migration needed; greenfield repository with zero prior deployments). |
| **Current proposal** | Omit `assessment_version` from the initial `attempt.json` schema from day one and enforce fail-closed rejection via `extra="forbid"` whenever an input payload contains `assessment_version`, with zero legacy stripping or import shims (leaving static files on disk untouched unless passed to a validator). |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-08`](issue-4-follow-through.md) |

### `S1-09` — `D1a` (Registry Repository and Package Ownership)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-09` |
| **Decision ID** | `D1a` |
| **Decision question** | Should `learningctl`, `learning-registryd`, and host provisioning live in a separate dedicated repository (`learning-registry`) or inside the `live-quiz` repository as a multi-component monorepo? |
| **Already-settled constraints** | `BL-06` (`learningctl` + `learning-registryd` service boundary), `B2.3` (independent implementations without a shared runtime package), and [ADR-0008](../docs/decisions/0008-two-product-repository-architecture.md). |
| **Current proposal** | Keep the `live-quiz` repository as a single monorepo housing three strictly isolated top-level components (`live-quiz` engine, `learning-registry` package/daemon/CLI, and portable reference skills) sharing the CI conformance suite (`B2.3`) while enforcing zero cross-package Python imports in CI. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-09`](issue-4-follow-through.md) |

### `S1-10` — `D1b` (Workspace Marker Edge Cases, Fail-Closed Error Taxonomy & Migration Trigger)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-10` |
| **Decision ID** | `D1b` |
| **Decision question** | How should `learningctl` handle duplicate `init` or unregistered `.learning-workspace` markers, classify fail-closed global DB states, and gate registry schema migrations? |
| **Already-settled constraints** | `BL-06` (nearest-ancestor `.learning-workspace` resolution, Git root rejected, host provisioning separate from `learningctl init`, `init` never recreates a missing global DB, pre-disruptive encrypted backup invariant). |
| **Current proposal** | Make same-directory `learningctl init` idempotent when already registered, fail closed (`UNREGISTERED_WORKSPACE_MARKER`) when a marker file's `workspace_id` is unknown to the global DB, classify daemon/DB failures into deterministic categories (`NOT_PROVISIONED`, `MISSING_OR_LOCKED`, `CORRUPT`, `MIGRATION_REQUIRED`, `UNSUPPORTED_SCHEMA`), and require explicit out-of-band admin migration gated on a verified encrypted backup snapshot. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-10`](issue-4-follow-through.md) |

### `S1-11` — `D2a` (`REGISTERING` Status Visibility and `ABORTED` Tombstone Retention)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-11` |
| **Decision ID** | `D2a` |
| **Decision question** | How are incomplete `REGISTERING` records surfaced in `learningctl status`, and how are `ABORTED` registration records and their unverified draft payloads retained? |
| **Already-settled constraints** | `BL-07` (`REGISTERING` $\to$ `AVAILABLE` / `ABORTED`; only `AVAILABLE` is executable; `assessment_id` is never reused; registration failure $\neq$ attempt crash recovery). |
| **Current proposal** | Surface incomplete `REGISTERING` definitions in a dedicated `incomplete_registrations` section of `learningctl status` (with no automatic TTL expiry, so the Tutor can prompt to finalize or abort), and transition aborted registrations to lightweight permanent `ABORTED` metadata tombstones (purging unverified draft $Q$/$R$ blobs while reserving `assessment_id`) hidden from both default `status` and `--all`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-11`](issue-4-follow-through.md) |

### `S1-12` — `D2b` (Startup-Commit Acknowledgment and Temporary Questionnaire Cleanup Protocol)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-12` |
| **Decision ID** | `D2b` |
| **Decision question** | By what mechanism does `live-quiz start` acknowledge that the `C1` journal snapshot is durably committed and the local HTTP server is bound, so `learningctl` can delete the temporary materialized `questionnaire.json` immediately without waiting for the learner to finish the quiz? |
| **Already-settled constraints** | `BL-04` (`C1` durable snapshot before learner access) and `BL-07` (idempotent temp questionnaire cleanup via `learningctl`). Soft/informational implementation dependency: `S1-07` (`B2.5.2`). |
| **Current proposal** | Order `live-quiz start` startup as validate $\to$ durable SQLite journal snapshot commit (`C1`) $\to$ localhost TCP port bind $\to$ emit a single flushed `SESSION_READY` NDJSON line on `stdout` (`attempt_id`, `assessment_id`, `questionnaire_hash`, `url`), enabling immediate temp file cleanup upon receiving `SESSION_READY`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-12`](issue-4-follow-through.md) |

### `S1-13` — `D3a` (Read-Only Active-vs.-Recoverable Attempt Inspection Contract)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-13` |
| **Decision ID** | `D3a` |
| **Decision question** | How does `live-quiz` track active process liveness and expose a read-only (`mode=ro`) inspection interface so `learningctl status` can distinguish `ACTIVE_LIVE_ATTEMPT` from `RECOVERABLE_INTERRUPTED_ATTEMPT` without mutating journals or reading private SQLite tables? |
| **Already-settled constraints** | `BL-04` (`C1`), `BL-08` (`learningctl status` is read-only; `live-quiz` is sole authority for active vs. recoverable state without exposing private SQLite tables), and [ADR-0006](../docs/decisions/0006-ephemeral-sqlite-session-journal.md). |
| **Current proposal** | Have `live-quiz` hold an exclusive OS file lock (`flock`) on a sibling `attempt-<attempt-id>.lock` file for the full duration of a running session (`start` or `recover`) and expose a read-only batch CLI inspection command (`mode=ro` SQLite URI + non-unlinking lock probe) invoked by `learningctl status`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-13`](issue-4-follow-through.md) |

### `S1-14` — `D3b` (Service-Boundary Delegation Cutoff and `schema_version` Format)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-14` |
| **Decision ID** | `D3b` |
| **Decision question** | Should per-command `learningctl --json` `schema_version` standardize on `"<major>.<minor>"` string syntax, and where is the exact boundary between architectural service invariants and deferred host packaging details? |
| **Already-settled constraints** | `BL-06` (`learningctl` $\to$ UDS IPC $\to$ `learning-registryd` under `learning-registry` user, `Accept=no`, serialized mutations, per-command `schema_version` compatibility principles; exact `"<major>.<minor>"` format flagged open at `step 827`). Prerequisite: `S1-09` (`D1a`). |
| **Current proposal** | Adopt `"<major>.<minor>"` (e.g., `"1.0"`) string format for every `learningctl --json` response, keep privileged host provisioning/key-escrow/migration/restore out of agent-facing `learningctl`, and delegate exact systemd unit directives, UDS paths/group names/modes, wire framing, and subcommand spellings to `DEF-04`/`DEF-05`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-14`](issue-4-follow-through.md) |

### `S1-15` — `D4` (Out-of-Band Key Escrow, Backup Verification Gate & Operational Delegation)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-15` |
| **Decision ID** | `D4` |
| **Decision question** | How is non-TPM cross-host portability reconciled with keeping encryption keys out of `learningctl` and DB backups, and how are backup snapshots verified before rotation or destructive maintenance? |
| **Already-settled constraints** | `BL-06` (mandatory encryption, `SQLCipher` candidate only, non-TPM default host key protection separate from workspaces and DB backups, `learning-registryd` Online Backup API snapshots on schedule and pre-disruptive operations). Prerequisite: `S1-10` (`D1b`). |
| **Current proposal** | Require privileged admin out-of-band key escrow/import (never accessible via `learningctl` or stored in backup directories), enforce a stage-then-verify backup pipeline (stage an encrypted-to-encrypted backup $\to$ perform independent keyed integrity verification of the staged snapshot before atomic promotion into the verified catalog, never pruning the last verified recovery point), make the concrete integrity-check primitive conditional on the database/encryption implementation later selected in `DEF-05` (`SQLCipher` remains a candidate, not a decision), and delegate driver/schedule/retention tunables to `DEF-05`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-15`](issue-4-follow-through.md) |

### `S1-16` — `E1` (Per-Execution Staging Slot Allocation and `--output` Destination Discipline)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-16` |
| **Decision ID** | `E1` |
| **Decision question** | How are managed `attempt.json` export paths allocated before `live-quiz` generates `attempt_id` so reassessments and multiple attempts never collide on `--output` (`CONFLICT_DIFFERENT_ATTEMPT`)? |
| **Already-settled constraints** | `BL-04` (`C1`), `BL-09` (persistent non-workspace staging area for managed `attempt.json` until consolidation; standalone exports remain user-controlled files), and [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md). |
| **Current proposal** | Have `learningctl` allocate a unique per-execution staging directory slot (`<staging_root>/<interaction_id>/attempt.json`) during execution preparation and pass that explicit file path to `live-quiz start --output <path>` and `live-quiz recover --output <path>`, preserving [ADR-0007](../docs/decisions/0007-atomic-json-export-contract.md)'s single-attempt idempotent export contract without adding a separate `--output-dir` mode to `live-quiz`. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-16`](issue-4-follow-through.md) |

### `S1-17` — `E2` (Pre-Consolidation Storage Split, Pre-Debrief Status Reconciliation & `ABANDONED` Payload Retention)
| Field | Description |
| :--- | :--- |
| **Task ID** | `S1-17` |
| **Decision ID** | `E2` |
| **Decision question** | Where do pre-consolidation interaction metadata, append-only events, payloads, and structured debrief checkpoints live before `CONSOLIDATED`, how does `learningctl status` surface un-evaluated staged `attempt.json` exports without altering the `BL-09` states, and how does transactional consolidation verify artifact correspondence? |
| **Already-settled constraints** | `BL-09` (settled lifecycle states: `PENDING_DEBRIEF`, `DEFERRED`, `PENDING_CONSOLIDATION`, `CONSOLIDATED`, `ABANDONED`; managed `attempt.json` staged until atomic consolidation of `(attempt, evaluation, Tutor observations)` into `CONSOLIDATED` with idempotent receipt; append-only history vs. reversible `ABANDONED` disposition hidden from `status` and `--all`; bounded ~1–3 turn debrief facilitation guideline; structured debrief progress without chat transcripts). Any proposed modification replacing or altering a settled `BL-09` state requires explicit user authorization. Prerequisites: `S1-06` (`B2.5.1`), `S1-16` (`E1`). |
| **Current proposal** | Keep all five settled `BL-09` interaction lifecycle states unchanged while adding read-only `learningctl status` reconciliation of `E1` staging slots to identify un-evaluated staged `attempt.json` files (where `attempt.json` exists before `evaluation.json`) using `awaiting_evaluation`, where `awaiting_evaluation` is a derived, read-only status classification used to identify an incomplete staged interaction (it is not a persisted lifecycle state, does not modify the `BL-09` state set, and does not replace `PENDING_DEBRIEF`); store lightweight interaction envelopes and append-only lifecycle events in the encrypted registry while keeping unconsolidated payloads (`attempt.json`, `evaluation.json`, structured debrief checkpoints) in the `E1` staging slot (retained across `ABANDONED` so learner-requested reconsideration remains executable); and verify `B2.5.1` cryptographic/identifier correspondence and idempotent bundle digests before atomic `CONSOLIDATED` commit and staging slot cleanup. |
| **Provenance** | `UNAPPROVED_RECOMMENDATION — step 850` |
| **Status** | `PENDING_USER_REVIEW` |
| **Tracker reference** | [issue-4-follow-through.md — `S1-17`](issue-4-follow-through.md) |

---

## Section 4 — Deferred Implementation Responsibilities (`DEF-01` through `DEF-05`)

The following engineering tasks are intentionally postponed to post-Stage-3 implementation:

- **`DEF-01` — JSON Schemas & Strict Pydantic Models:** Exact field names and `.schema.json` specifications for `questionnaire.json`, `rubric.json`, `attempt.json`, and `evaluation.json`, including recursive `extra="forbid"`, `strict=True`, and negative nesting tests.
- **`DEF-02` — RFC 8785 Library Selection & Shared Conformance Suite:** Selection of the concrete Python RFC 8785 package (`rfc8785` / `jcs` candidates) and assembly of the shared raw-byte CI conformance test suite (official RFC 8785 vectors + `B2.2` negative parser vectors + `attempt.json` float vectors).
- **`DEF-03` — `live-quiz` CLI Interface & Output Schemas:** Exact subcommand spellings, flags, and output/JSON schemas for `live-quiz validate`, startup readiness acknowledgment (`D2b`), and read-only attempt inspection (`D3a`), plus authoring `live-quiz.md`.
- **`DEF-04` — `learningctl` Domain CLI Grammar & JSON Schemas:** Exact `.learning-workspace` optional metadata fields, command/flag spellings, and CLI `schema_version` / JSON output schemas (`status`, consolidation receipt, structured debrief checkpoint), plus authoring `learningctl.md`.
- **`DEF-05` — Host Daemon Packaging, Security & Backup Operations:** Exact `learning-registryd` systemd `.socket`/`.service` units, UDS path/group/modes, private IPC wire framing, XDG runtime/state paths, encrypted DB driver (`SQLCipher` candidate), concrete backup integrity-check primitive, credential mechanism (`LoadCredentialEncrypted=` vs. keyfile), admin key-escrow/migrate/restore tooling, backup timer schedule, retention thresholds, and external sync target.

> [!IMPORTANT]
> **Boundary Rules for Deferred Items:**
> 1. **Deferred $\neq$ Decided:** Postponing an implementation mechanism does not select a default candidate (e.g., `SQLCipher` and `rfc8785` remain candidates until `DEF-02`/`DEF-05`).
> 2. **Invariants vs. Mechanisms:** An architectural invariant (such as mandatory at-rest encryption, backup integrity verification before promotion, or 3-layer JCS hashing) is binding once approved, while its concrete implementation remains open under `DEF-01`–`DEF-05`.
> 3. **No Premature ADR Encoding:** Stage 2 ADRs (`S2-01`–`S2-05`) must record the actual user-approved architectural contracts and invariants (including any approved modifications or alternatives), never locking in unapproved `DEF-01`–`DEF-05` implementation specifics as if they were settled architecture.

See [issue-4-follow-through.md — Section 5](issue-4-follow-through.md#5-deferred-implementation-responsibilities-not-completed-architecture) for detailed triggers and completion conditions for `DEF-01`–`DEF-05`.

---

## Section 5 — Decision-Record Maintenance Protocol

All subsequent sessions working on Issue #4 must follow these eight rules:

1. **Explicit User Approval Required:** Never transition a decision from `PENDING_USER_REVIEW` without an explicit `USER_INPUT` turn authorizing one of the five valid dispositions:
   - `ACCEPTED_AS_PROPOSED`
   - `ACCEPTED_WITH_MODIFICATIONS`
   - `REJECTED_IN_FAVOR_OF_ALTERNATIVE`
   - `DEFERRED_WITH_JUSTIFICATION`
   - `NOT_APPLICABLE_WITH_JUSTIFICATION`
2. **Record Modifications Faithfully:** When a decision is `ACCEPTED_WITH_MODIFICATIONS` or `REJECTED_IN_FAVOR_OF_ALTERNATIVE`, replace the summary with the exact user-approved commitment and record the modification delta rather than preserving the original `step 850` recommendation as if unchanged.
3. **Preserve Rejected Alternatives & Rationale:** When alternatives are rejected during review, concisely record the rejected option and why it was rejected so Stage 2 MADR drafting (`S2-01`–`S2-04`) has complete context.
4. **Incremental Dual-Artifact Updates:** Update both [issue-4-decision-ledger.md](issue-4-decision-ledger.md) and [issue-4-follow-through.md](issue-4-follow-through.md) immediately after each atomic decision is resolved, recording the transcript step or date of approval.
5. **Stable Identifiers:** Keep task IDs (`S1-01`–`S1-18`, `CW-01`, `S2-01`–`S2-06`, `S3-01`, `DEF-01`–`DEF-05`, `I9-01`) and decision IDs (`BL-01`–`BL-09`, `B2.1`–`B2.3`, `B2.4.1`–`E2`) stable across sessions.
6. **Immutability of Prior Meaning:** Never silently broaden, narrow, or reinterpret an earlier settled decision (`BL-01`–`BL-09`, `B2.1`–`B2.3`, or any newly resolved `S1-*` item). Any revision to a settled commitment requires an explicit user decision recorded with updated provenance.
7. **New Architectural Questions:** If a new architectural question surfaces during review, assign it a distinct identifier, evaluate whether it is a genuine prerequisite for any existing task, and record it explicitly rather than folding it silently into an existing item.
8. **Separation of Authority:** [issue-4-decision-ledger.md](issue-4-decision-ledger.md) is authoritative for **decision state, provenance, and rationale**; [issue-4-follow-through.md](issue-4-follow-through.md) is authoritative for **work status, dependencies, and execution sequencing**.
