# ADR-0005: Python/FastAPI Backend and Vite Plus (`vp`) Frontend Stack

- Status: Accepted
- Date: 2026-11-10

## Context
`live-quiz` requires a fast, reliable CLI, an ephemeral local HTTP REST API, strict schema validation, and a browser UI that renders rich markdown, Mermaid diagrams, KaTeX equations, and code editors offline. We needed to select an implementation stack that minimizes friction, maximizes maintainability, and aligns with the author's development workflow.

## Decision
Implement `live-quiz` using:
- **Backend & CLI:** Python 3.11+, FastAPI, Pydantic v2, and managed via `uv`.
- **Static Analysis:** Pyright (type checking) and Ruff (linting/formatting).
- **Frontend & TypeScript Tooling:** Single-page application built via the Rust-powered Vite Plus (`vp`), with all third-party libraries (CodeMirror, Mermaid.js, KaTeX) fully bundled into static assets (`dist/`). FastAPI mounts and serves `dist/` directly.

## Alternatives Considered
- **Go Backend:** Rejected. While offering single-binary compilation, it introduces cognitive overhead and unfamiliar web/routing paradigms for the primary maintainer, diverting energy from core pedagogical features.
- **Node/TypeScript Backend:** Rejected. TypeScript is not the preferred backend language for this environment, and running Node on the server side adds unnecessary runtime dependencies.
- **Standard npm/Vite without Rust Tooling:** Rejected. Vite Plus (`vp`) provides unified, high-performance Rust-powered bundling and tooling.
- **CDN-hosted Frontend Assets:** Rejected. Fails when working offline or with spotty internet connectivity.

## Consequences
- **Positive:** Pydantic v2 provides a single source of truth for runtime validation and schema generation; `uv` ensures instant startup and hermetic environment management; pre-bundled Vite Plus assets guarantee 100% offline operation on Linux/NixOS.
- **Negative / Trade-offs:** Requires a frontend build step via `vp` during distribution.

## Related Decisions
- [ADR-0001: Standalone Local Web Application](./0001-standalone-web-application.md)
- [ADR-0006: Ephemeral SQLite Session Journal & Recovery](./0006-ephemeral-sqlite-session-journal.md)
