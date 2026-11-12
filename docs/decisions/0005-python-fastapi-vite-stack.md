# ADR-0005: Python/FastAPI Backend and Vite Plus (`vp`) Frontend Stack

- Status: Accepted
- Date: 2026-11-10

## Context
`live-quiz` requires a fast, reliable CLI, an ephemeral local HTTP REST API, strict schema validation, and a browser UI that renders rich markdown, Mermaid diagrams, KaTeX equations, and code editors offline. We needed to select an implementation stack that minimizes friction, maximizes maintainability, and aligns with the author's development workflow.

## Decision
Implement `live-quiz` using:
- **Backend & CLI:** Python 3.11+, FastAPI, Pydantic v2, and managed via `uv` on Linux (the supported runtime platform).
- **Static Analysis:** Pyright (type checking) and Ruff (linting/formatting).
- **Frontend & TypeScript Tooling:** Single-page application built via the Rust-powered Vite Plus (`vp`), with all third-party libraries (CodeMirror, Mermaid.js, KaTeX) fully bundled into static assets (`dist/`). FastAPI mounts and serves `dist/` directly.

### Temporal Architecture & Lifecycle Coordination
1. **Platform Clocks & Linux Runtime Baseline:**
   The backend relies on Linux kernel clock facilities exposed via Python's standard library:
   - `time.monotonic_ns()` (`CLOCK_MONOTONIC`) for active assessment time. It is unaffected by discontinuous wall-clock steps (e.g. manual time changes, NTP steps), though subject to gradual frequency adjustments (NTP slewing). Crucially, it halts during kernel suspend (ACPI S3/sleep), automatically freezing the assessment budget when a laptop lid closes.
   - `time.clock_gettime_ns(time.CLOCK_BOOTTIME)` (`CLOCK_BOOTTIME`) for the operational pause-grace keepalive timer. It advances during kernel suspend, ensuring that laptop sleep consumes the server keepalive window.
   - Host boot identity is verified via `/proc/sys/kernel/random/boot_id` so that persisted boottime deadlines are never compared across system reboots.
   - **BOOTTIME-Aware Pause-Grace Watchdog:** Because Python's `asyncio` loop timers wrap `CLOCK_MONOTONIC` and halt during system sleep, standard `asyncio.sleep()` cannot accurately expire a boottime grace period. The server re-evaluates the persisted `CLOCK_BOOTTIME` deadline at the first post-wake watchdog execution (e.g. when a suspend gap is detected or via a bounded periodic watchdog), terminating cleanly with `outcome: SUSPENDED` if the deadline expired during sleep.
2. **FastAPI Autonomous Watchdog, Immutable Time Limits & Serialized Admission:**
   - **Immutable Attempt Budget (`time_limit_ns`):** The effective duration budget is immutable for each attempt, persisted in SQLite session state at attempt initialization and validated against the frozen questionnaire version. Recovery must never load altered CLI defaults or application configurations that silently mutate an existing attempt's time budget.
   - **Autonomous Watchdog:** An `asyncio` background task monitors remaining active budget. When elapsed active time reaches the budget, it triggers the terminal transition to `TIMED_OUT` without waiting for client interaction.
   - **Serialized Mutation Admission Rule:** Every response mutation endpoint (`POST /api/response/autosave`, `POST /api/response/submit`) independently validates the deadline while holding the SQLite write-serialization boundary (`BEGIN IMMEDIATE`). Monotonic time is sampled *after* acquiring the lock:
     - $T_{\text{elapsed}} < T_{\text{limit}} \implies$ mutation accepted and checkpointed.
     - $T_{\text{elapsed}} \ge T_{\text{limit}} \implies$ mutation rejected; endpoint atomically commits `TIMED_OUT` and `publication_intent`.
   - A delay or stall in the background watchdog cannot allow late submissions to be accepted.
3. **Vite Plus Frontend Client Timing Behavior:**
   - **Presentation Only:** The browser treats server state as authoritative. Countdown is a local display projection based on the latest server payload (`remaining_seconds`, `is_active`).
   - **Background Tab Throttling Mitigation:** When the browser tab is hidden or backgrounded, local visual updates halt to avoid burning CPU cycles against throttled timers.
   - **Visibility Lifecycle Listener:** The client attaches a listener to the Page Visibility API (`document.visibilityState === 'visible'`) and window focus/reconnect events. Upon re-entering visibility, it immediately issues `GET /api/session` to resynchronize the displayed countdown without continuous polling.
4. **Threat Model & Trust Boundary:**
   The timing architecture is **server-authoritative relative to the browser**, guarding against client clock tampering, background timer throttling, network latency games, and wall-clock skew. Host-level root or user tampering (e.g., modifying SQLite directly or patching the Python binary) is explicitly outside the threat model of a local, formative personal assessment tool.

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
