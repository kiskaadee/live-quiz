# ADR-0008: Two-Product Repository Architecture

- Status: Accepted
- Date: 2026-11-10

## Context
Initial designs conflated the assessment engine with the author's personal Antigravity learning workflows, private `learning-contract.md`, and centralized knowledge vault (`~/Brain/`). This prevented `live-quiz` from being shared with peers as a standalone tool and created unnecessary coupling to a specific agent harness.

## Decision
Structure the repository as two decoupled, independently consumable products:
1. **The Assessment Engine (`live-quiz`):** An AI-agnostic CLI and web engine that consumes `questionnaire.json` and produces `attempt.json`. It knows nothing about AI, prompts, or Antigravity.
2. **The Reference Skills Collection (`skills/`):** Five portable agent skills demonstrating how any AI environment can compose learning and assessment workflows without private dependencies.

Additionally:
- **Retirement of Centralized Vault Persistence:** Formally eliminate mandatory generation of Markdown journals or notes into `~/Brain/00-inbox/`. The assessment debrief is a cognitive consolidation event. Any technical documentation or architectural records belong selectively to the active project repository according to that project's conventions.
- **Sovereign Personal Skills:** The author's personal `assess` skill in `~/Projects/skills/kiskaadee` remains a private composition that imports and uses `live-quiz`, layering on personal invariants without polluting the public repository.

## Alternatives Considered
- **Monolithic Agent Tooling Repo:** Rejected. Embedding personal learning contracts and vault habits into `live-quiz` prevents public distribution and forces non-standard conventions on peers.
- **Pure Engine Without Skills:** Rejected. Leaving `live-quiz` without companion reference skills leaves the engine without an accessible demonstration of AI-assisted authoring and evaluation.

## Consequences
- **Positive:** Public reusability; clean separation of concerns; elimination of curation debt from centralized vaults; modularity for peers.
- **Negative / Trade-offs:** Requires maintaining reference skills separately from private agent profiles.

## Related Decisions
- [ADR-0001: Standalone Local Web Application](./0001-standalone-web-application.md)
- [ADR-0009: Modular, Portable Reference Skills Collection](./0009-modular-portable-reference-skills.md)
