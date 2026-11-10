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
                   Objective Ledger (in context)
                   • Flat objectives with `enables` edges
                   • Cumulative coverage observations
                   • Active / Deferred disposition
                               │
                               ▼ (Assessment Milestone & Learner Consent)
                     Assessment Brief
                               │
                               ▼
                     `quiz-designer` Skill
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
    `questionnaire.json`                  `rubric.json`
    (Public Presentation)              (Private Evaluation)
              │                                 │
              ▼                                 │
    [ `live-quiz` Engine ]                      │
    • Standalone Local Web App                  │
    • Zero LLM at runtime                       │
    • Ephemeral SQLite Journal                  │
    • Authoritative Server Timer                │
    • Active time vs Pause tracking             │
              │                                 │
              │ (Atomic Export attempt.json)    │
              ▼                                 ▼
       `attempt.json`  ──────────────►  [ `quiz-evaluator` Skill ]
                                                │
                                                ▼
                                        `evaluation.json`
                                                │
                                                ▼
                                          [ `tutor` Skill ]
                                      (3-Stage Socratic Debrief)
                                                │
                                                ▼
                                    [ Cognitive Consolidation ]
                                   (Selective Project Docs Only)
```

---

## 2. System Responsibilities & Boundaries

The repository distributes two decoupled products:
1. **The Assessment Engine (`live-quiz`):** An AI-agnostic CLI and web application that administers an assessment, tracks timing, preserves drafts, enforces a blind pass, and publishes standardized attempt artifacts.
2. **The Reference Skills Collection (`skills/`):** Five portable agent skills (`learning-objectives`, `quiz-designer`, `live-quiz`, `quiz-evaluator`, and `tutor`) demonstrating how any AI agent environment can create, run, and interpret assessments.

### Component Boundaries

| Component | Responsibility | Inputs | Outputs | Core Invariants |
| :--- | :--- | :--- | :--- | :--- |
| **`learning-objectives`** | Formulates observable, assessable competencies from learning goals. | Topic / Goals / Context | Objectives & evidence criteria | Does not assume mastery; separates intention from demonstration. |
| **`quiz-designer`** | Compiles an approved Assessment Brief into public and private contracts. | Assessment Brief | `questionnaire.json`, `rubric.json` | Max 3-question chains; hermetic context or independent rubrics; no answer leakage. |
| **`live-quiz` Engine** | Ephemeral web server executing the assessment and capturing evidence. | `questionnaire.json`, CLI flags | `attempt.json`, session status | Never receives `rubric.json`; zero runtime LLM; authoritative server clock. |
| **`quiz-evaluator`** | Context-isolated post-hoc evaluation of attempt against frozen rubric. | `rubric.json`, `attempt.json` | `evaluation.json` | Fresh context; purely evaluates $f(\text{rubric}, \text{attempt})$ without chat history. |
| **`tutor`** | Facilitates learning, proposes assessment scope, leads debrief. | Learner dialogue, context | Learning interactions | Does not force assessments; leads Socratic debrief before solution reveal. |
| **Project Repository** | Selective retention of project-specific technical findings or ADRs. | User / Tutor choice | Project docs / Git commits | No mandatory central vault; documentation follows project conventions. |

---

## 3. Data Contracts & Artifact Topology

The system communicates strictly across four versioned JSON contracts:

1. **`questionnaire.json` (Public Presentation Contract):**  
   Consumed by `live-quiz`. Contains question IDs, markdown prompts, Mermaid diagrams, KaTeX equations, interaction types, option labels, and initial code stubs. *Physically excludes answers, rubrics, and misconceptions.*
2. **`rubric.json` (Private Evaluation Contract):**  
   Retained by the calling agent workflow and supplied only to the evaluator. Contains learning objectives, observable criteria (Complete / Partial / Insufficient), categorical failure modes, and misconception signatures.
3. **`attempt.json` (Durable Execution Evidence):**  
   Produced by `live-quiz` via atomic publication. Contains attempt identity, assessment/version hashes, workspace attribution, terminal status (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`), active and wall-clock durations, pause event logs, and submitted responses.
4. **`evaluation.json` (Diagnostic Assessment Report):**  
   Produced by `quiz-evaluator`. Contains criterion-level attainment ratings, failure-mode classifications (`CONCEPTUAL`, `LOGICAL`, `SYNTACTIC`, `OMISSION`, `REGRESSION`), misconception hypotheses with confidence levels, and discussion probes for debriefing.

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
- Pausing commits state and **allows the server process to exit cleanly**, preventing idle background processes.
- Unexpected kills (`SIGKILL`, power failure) leave the journal intact.

### 5.3 Atomic Publication & Journal Deletion
Upon reaching a terminal state (`COMPLETED`, `CLOSED_EARLY`, `TIMED_OUT`):
1. The server serializes the final attempt payload.
2. Writes to a temporary file in the destination directory.
3. Validates the JSON structure and flushes/syncs to disk.
4. Atomically renames the file to the final destination (`attempt.json`).
5. Closes database handles and unlinks the temporary SQLite journal.
6. Exits with code 0.

*Invariant:* The temporary journal is never deleted if export publication fails.

### 5.4 Explicit Recovery Command
To resume an interrupted or paused attempt:
```bash
live-quiz recover --from-path ~/.local/state/live-quiz/sessions/<attempt-id>.sqlite3
```
Recovery validates the journal, restores the authoritative clock, re-mounts drafts, and starts the server without altering the attempt identity.

---

## 6. Pedagogical Scoping & Debrief Protocol

### 6.1 Objective Ledger & Scoping
- **Representation:** Objectives are maintained in the Tutor's active context as a flat collection with directed `enables` edges.
- **Coverage Tracking:** Recorded as cumulative observations (`INTRODUCED`, `EXPLORED`, `PRACTICED`) alongside an active disposition (`ACTIVE` vs. `DEFERRED`).
- **Scoping Gate:** When an assessment milestone is reached, the Tutor compiles a candidate **Assessment Brief**. The learner explicitly reviews and approves the scope before authoring begins.

### 6.2 Three-Stage Socratic Debrief
1. **Results Overview:** High-level summary of overall attainment, demonstrated competencies, and friction points without revealing answers.
2. **Guided Socratic Reflection:** Targeted diagnostic questions exploring identified misconceptions, allowing the learner to formulate reasoning first.
3. **Consolidation:** Canonical explanations revealed, misunderstandings clarified, and future practice priorities established.
4. **Selective Documentation:** Debrief is a cognitive consolidation dialogue. Durable technical notes or ADRs are committed selectively to the active project repository; no mandatory universal vault summaries are generated.
