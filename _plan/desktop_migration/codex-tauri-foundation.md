# Tauri Foundation Plan

## Guiding Principles
- Pure static assets: `index.html`, `styles/*.css`, `js/*.js` served directly by Tauri dev server.
- Rust backend owns all disk, network, and AI calls; frontend only manipulates DOM + invokes commands.
- Minimal dependencies: avoid bundlers, keep Rust crates lean (Tauri core, `serde`, `tokio`, `reqwest`, `rusqlite`).
- Single-window app sized for desktop productivity, with persisted window state (leveraging `tauri-plugin-window-state`).

## Project Layout
```
src/
  index.html           # Shell layout + root containers
  styles/
    app.css            # Global layout + CSS variables
    canvas.css         # Canvas-specific styles
  js/
    app.js             # Bootstraps UI modules, event bus
    ipc.js             # Thin wrappers around Tauri `invoke`
    state.js           # Local state store (no external lib)
    canvas.js          # Canvas module (see dedicated plan)
    blocks.js          # Block palette + config panels
src-tauri/
  src/
    main.rs            # Tauri setup, command registration
    commands/
      workflows.rs     # load/save/export
      execution.rs     # run workflows
      ai.rs            # provider bridges
    db/
      mod.rs           # SQLite helpers
    models.rs          # Shared DTOs (serde)
  Cargo.toml
  tauri.conf.json
```

## Core Tauri Commands (MVP)
- `list_workflows`, `load_workflow`, `save_workflow`, `delete_workflow`.
- `execute_workflow` (spawns async task, streams logs via `tauri::async_runtime::spawn` + `Emitter`).
- `store_api_key`, `load_api_key` (encrypted via OS keyring or simple XOR fallback if keyring unavailable).
- `export_workflow`, `import_workflow` (JSON files in user-chosen directory).

## IPC & Event Strategy
- Frontend modules import a tiny `invokeCommand(name, payload)` helper that wraps `@tauri-apps/api/core`.
- Long-running execution uses Tauri events: backend emits `execution:log`, `execution:complete`, `execution:error`; frontend subscribes once at boot.
- Keep payload formats identical to existing `ExecutionResult` / `BlockLog` shapes so serialization is straightforward.

## Build & Distribution
- Dev: `cargo tauri dev` (no Bun). Provide npm-free script alias `make dev` if desired.
- Release: `cargo tauri build` producing platform installers (MSI, DMG, AppImage). Document required Apple/notarization steps later.
- CI: simple GitHub workflow building per-platform artifacts; skip until MVP validated.

## Security Defaults
- Disable remote URL access by default (`security.csp` locked down, no `dangerousUseHttpScheme`).
- Freeze JS prototypes to reduce supply-chain risks.
- Bundle only whitelisted local assets; ensure all network calls go through Rust (frontend has zero direct `fetch`).
