# ADR-0001: Standalone Local Web Application

- Status: Accepted
- Date: 2026-11-10

## Context
Interactive assessments in an engineering learning environment require rendering rich visual diagrams (Mermaid flowcharts/sequence diagrams), mathematical notation (KaTeX), syntax-highlighted code editors with read-only context and editable buffers, and drag-and-drop or ordering widgets. We needed to choose the presentation layer and execution modality for the quiz runtime.

## Decision
Build `live-quiz` as a local, standalone web application comprising an ephemeral HTTP backend and a modern browser frontend, launched and managed via a command-line interface (`live-quiz start`).

## Alternatives Considered
- **Interactive Terminal UI (TUI):** Rejected. Terminal renderers for Mermaid diagrams and multi-line code editing introduce significant cognitive noise, distracting from the assessment. Furthermore, sharing a TUI tool with peers imposes non-standard terminal assumptions.
- **Embedded IDE Generative UI / Webview:** Rejected. Tightly couples the tool to a specific IDE client (e.g. Antigravity), breaking portability across different environments (Claude Code, terminal, or standalone peer usage).

## Consequences
- **Positive:** Enables rich, uncompromised DOM rendering (Mermaid.js, CodeMirror/Monaco, KaTeX); isolates the learner cognitively from conversational chat during the assessment; allows seamless sharing with external peers.
- **Negative / Trade-offs:** Introduces local network port allocation, browser process focus switching, and HTTP process lifecycle management.

## Related Decisions
- [ADR-0002: Harness-Agnostic Process Lifecycle](./0002-harness-agnostic-process-lifecycle.md)
- [ADR-0005: Python/FastAPI Backend and Vite Frontend Stack](./0005-python-fastapi-vite-stack.md)
