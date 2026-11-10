# ADR-0009: Modular, Portable Reference Skills Collection

- Status: Accepted
- Date: 2026-11-10

## Context
AI-assisted assessment involves multiple pedagogical stages: learning objective formulation, assessment design, runtime administration, evidence evaluation, and Socratic debriefing. If implemented as a single monolithic prompt or tightly coupled to Antigravity subagents, the workflow cannot be reused across different agent platforms or by learners with different teaching preferences.

## Decision
Provide an MVP collection of five modular, portable reference skills in the repository's `skills/` directory, designed for independent consumption:

| Skill | Role | Primary Input | Primary Output |
| :--- | :--- | :--- | :--- |
| **`learning-objectives`** | Formulates observable, assessable competencies from learning goals. | Topic / Goals / Context | Objectives & evidence criteria |
| **`quiz-designer`** | Compiles an approved Assessment Brief into public and private contracts. | Assessment Brief | `questionnaire.json`, `rubric.json` |
| **`live-quiz`** | Operational CLI wrapper executing and recovering assessments. | Questionnaire path, CLI flags | Browser session URL, `attempt.json` |
| **`quiz-evaluator`** | Context-isolated post-hoc evaluation of attempt against rubric. | `rubric.json`, `attempt.json` | `evaluation.json` |
| **`tutor`** | Orchestrates learning dialogue, proposes scope, and leads debrief. | Learner interaction | Consolidated learning experience |

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
3. **Objective Modeling & Scoping:**
   - Objectives are maintained as flat collections with directed `enables` relationships (a graph, not a nested tree).
   - Coverage is tracked as cumulative observations (`INTRODUCED`, `EXPLORED`, `PRACTICED`) alongside an active disposition (`ACTIVE` vs. `DEFERRED`).
   - Assessment readiness is evaluated contextually when the Tutor compiles the **Assessment Brief**, rather than as an intrinsic boolean flag in the ledger.

## Alternatives Considered
- **Single Monolithic "Mega-Prompt" Skill:** Rejected. Makes it impossible to customize or substitute individual stages (e.g. using a different evaluator or authoring tool).
- **Hardcoded Antigravity Subagent Hierarchy:** Rejected. Breaks portability for users running in non-Antigravity environments.

## Consequences
- **Positive:** Maximum composability and reusability; clean separation of pedagogical concerns; testable individual skills.
- **Negative / Trade-offs:** Requires maintaining five distinct skill instruction files (`SKILL.md`) and clear interface documentation.

## Related Decisions
- [ADR-0004: Physical Separation of Questionnaire and Rubric](./0004-questionnaire-rubric-separation.md)
- [ADR-0008: Two-Product Repository Architecture](./0008-two-product-repository-architecture.md)
