# ADR-0009: Modular, Portable Reference Skills Collection

- Status: Accepted
- Date: 2026-11-10

## Context
AI-assisted assessment involves multiple pedagogical stages: learning objective formulation, assessment design, runtime administration, evidence evaluation, and Socratic debriefing. If implemented as a single monolithic prompt or tightly coupled to Antigravity subagents, the workflow cannot be reused across different agent platforms or by learners with different teaching preferences.

## Decision
Provide an MVP collection of five modular, portable reference skills in the repository's `skills/` directory, designed for independent consumption:

| Skill | Role | Primary Input | Primary Output |
| :--- | :--- | :--- | :--- |
| **`learning-objectives`** | Formulates observable, assessable competencies; searches external catalog to propose new records and relationships. | Topic / Goals / Context, Registry candidates | Proposed objectives & evidence criteria |
| **`quiz-designer`** | Compiles an approved Assessment Brief into public and private contracts; uses learning history to inform context and difficulty. | Assessment Brief, Learner history | `questionnaire.json`, `rubric.json` |
| **`live-quiz`** | Operational CLI wrapper executing and recovering assessments; structurally preserves canonical objective references. | Questionnaire path, CLI flags | Browser session URL, `attempt.json` |
| **`quiz-evaluator`** | Context-isolated post-hoc evaluation of attempt against rubric; deterministically correlates evidence with canonical objective references. | `rubric.json`, `attempt.json` | `evaluation.json` |
| **`tutor`** | Orchestrates learning dialogue, proposes scope, logs contextual observations against stable objective IDs, and leads debrief. | Learner interaction, Registry/History | Consolidated learning experience, Contextual observations |

### Architectural Invariants:
1. **Independent Usability:**
   - `/learning-objectives` can be used standalone for workshop or curriculum design.
   - `/quiz-designer` accepts an assessment brief directly without requiring `/tutor`.
   - `/live-quiz` operates without any AI skill.
   - `/quiz-evaluator` evaluates any compatible attempt representation without conversation history.
   - `/tutor` composes the specialists, but the specialists do not depend on `/tutor`.
2. **Harness Portability:**
   - The reference skills operate as standard portable markdown instructions.
   - They do not mandate Antigravity-specific subagent APIs (`invoke_subagent`). In single-agent environments (e.g. Claude Code, Cursor), they execute sequentially in the active agent context.
   - Skills do not hardcode host filesystem layouts or private paths.
3. **Objective Modeling, Learning History & Scoping (Provisional Pedagogical Model):**
   - **Persistent Catalog vs. In-Context Ledger:** Objectives and competencies are resolved against an external catalog with many-to-many relationships (avoiding rigid parent-child hierarchies or assumed prerequisite DAGs). `learning-objectives` proposes new records; canonical identity is established by the registry.
   - **Contextual History vs. Single State:** Learner engagement is logged as longitudinal observations (e.g. `INTRODUCED`, `EXPLORED`, `PRACTICED` as provisional examples) associated with stable objective IDs. Repeated encounters are preserved rather than overwritten. Deferral (`ACTIVE` vs. `DEFERRED`) is an independent pedagogical disposition that does not erase past history.
   - **Exposure vs. Mastery:** Prior exposure informs assessment design to avoid redundant introductory testing, but exposure is never equated to mastery. Difficulty calibration accounts for familiarity, task complexity, context novelty, assistance level, and demonstrated independent performance.
   - **Scoping Gate:** Assessment readiness is evaluated contextually when the Tutor compiles the **Assessment Brief**, rather than as an intrinsic boolean flag in the catalog.
   - **Deterministic Evaluation Correlation:** Evaluation evidence is correlated deterministically to canonical objective references using explicit mappings across rubric criteria and attempt responses, eliminating ambiguous natural-language guessing.
   - **Abstract Capability Dependency:** Reference skills depend on an abstract registry and history capability (e.g. via CLI, local HTTP, or file protocols) rather than hardcoding specific storage implementations or database technologies. Detailed registry architecture, sync strategies, and exact schema mappings remain open design questions.

## Alternatives Considered
- **Single Monolithic "Mega-Prompt" Skill:** Rejected. Makes it impossible to customize or substitute individual stages (e.g. using a different evaluator or authoring tool).
- **Hardcoded Antigravity Subagent Hierarchy:** Rejected. Breaks portability for users running in non-Antigravity environments.

## Consequences
- **Positive:** Maximum composability and reusability; clean separation of pedagogical concerns; testable individual skills.
- **Negative / Trade-offs:** Requires maintaining five distinct skill instruction files (`SKILL.md`) and clear interface documentation.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
- [ADR-0008: Two-Product Repository Architecture](./0008-two-product-repository-architecture.md)
