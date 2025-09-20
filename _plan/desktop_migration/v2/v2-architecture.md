# V2 Architecture: Unified Desktop App Design

## Synthesis Philosophy

This V2 architecture combines the **KISS principles** from Claude, the **extensible modularity** from Codex, and the **pragmatic security focus** from Gemini. The result is a bulletproof MVP that is simple enough to ship quickly but architected for sustainable growth.

## Core Architecture: Hybrid Desktop Pattern

### High-Level Structure
```
┌─────────────────────────────────────────┐
│           Tauri Application             │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │         Frontend (WebView)        │  │
│  │    Vanilla JS + SVG Canvas        │  │
│  │                                   │  │
│  │  ┌─────────────┐ ┌─────────────┐  │  │
│  │  │ Canvas.js   │ │ Blocks.js   │  │  │
│  │  │ (Hybrid)    │ │ (Generic)   │  │  │
│  │  └─────────────┘ └─────────────┘  │  │
│  │                                   │  │
│  │  ┌─────────────┐ ┌─────────────┐  │  │
│  │  │ State.js    │ │ IPC.js      │  │  │
│  │  │ (Local)     │ │ (Tauri)     │  │  │
│  │  └─────────────┘ └─────────────┘  │  │
│  └───────────────────────────────────┘  │
│                   ▲                     │
│                   │ IPC + Events        │
│                   ▼                     │
│  ┌───────────────────────────────────┐  │
│  │         Backend (Rust)            │  │
│  │                                   │  │
│  │  ┌─────────────┐ ┌─────────────┐  │  │
│  │  │ Commands    │ │ Executor    │  │  │
│  │  │ (Tauri)     │ │ (Async)     │  │  │
│  │  └─────────────┘ └─────────────┘  │  │
│  │                                   │  │
│  │  ┌─────────────┐ ┌─────────────┐  │  │
│  │  │ Database    │ │ AI Manager  │  │  │
│  │  │ (SQLite)    │ │ (Providers) │  │  │
│  │  └─────────────┘ └─────────────┘  │  │
│  │                                   │  │
│  │  ┌─────────────┐ ┌─────────────┐  │  │
│  │  │ Block       │ │ Security    │  │  │
│  │  │ Registry    │ │ (Stronghold)│  │  │
│  │  └─────────────┘ └─────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Directory Structure: Clean Separation

**Adopting Codex's clean structure with Gemini's security considerations:**

```
sim-desktop/
├── src/                           # Frontend (served directly by Tauri)
│   ├── index.html                 # Shell layout + containers
│   ├── styles/
│   │   ├── app.css                # Global layout + CSS variables
│   │   ├── canvas.css             # Canvas-specific styles
│   │   └── blocks.css             # Block palette styles
│   └── js/
│       ├── app.js                 # Bootstrap + event bus
│       ├── ipc.js                 # Tauri invoke wrappers
│       ├── state.js               # Local state management
│       ├── canvas.js              # Hybrid canvas implementation
│       ├── blocks.js              # Block palette + config UI
│       └── utils.js               # Shared utilities
├── src-tauri/                     # Backend (Rust)
│   ├── src/
│   │   ├── main.rs                # Tauri setup + command registration
│   │   ├── commands/
│   │   │   ├── mod.rs             # Command exports
│   │   │   ├── workflows.rs       # CRUD operations
│   │   │   ├── execution.rs       # Workflow execution
│   │   │   ├── blocks.rs          # Block registry management
│   │   │   └── ai.rs              # AI provider commands
│   │   ├── database/
│   │   │   ├── mod.rs             # Database abstraction
│   │   │   ├── migrations.rs      # Schema migrations
│   │   │   └── models.rs          # Data models
│   │   ├── executor/
│   │   │   ├── mod.rs             # Execution engine
│   │   │   ├── engine.rs          # Core execution logic
│   │   │   └── context.rs         # Execution context
│   │   ├── blocks/
│   │   │   ├── mod.rs             # Block system
│   │   │   ├── registry.rs        # Block registry
│   │   │   ├── descriptors.rs     # Block definitions
│   │   │   └── handlers/          # Block handlers
│   │   │       ├── mod.rs
│   │   │       ├── control.rs     # Control flow blocks
│   │   │       ├── data.rs        # Data manipulation blocks
│   │   │       ├── network.rs     # HTTP blocks
│   │   │       ├── ai.rs          # AI blocks
│   │   │       └── io.rs          # File I/O blocks
│   │   ├── ai/
│   │   │   ├── mod.rs             # AI provider system
│   │   │   ├── manager.rs         # Provider manager
│   │   │   ├── providers/         # Provider implementations
│   │   │   │   ├── mod.rs
│   │   │   │   ├── openai.rs
│   │   │   │   ├── anthropic.rs
│   │   │   │   └── ollama.rs
│   │   │   └── types.rs           # Shared AI types
│   │   ├── security/
│   │   │   ├── mod.rs             # Security utilities
│   │   │   └── keys.rs            # API key management
│   │   └── error.rs               # Error types
│   ├── Cargo.toml
│   └── tauri.conf.json
└── blocks/                        # Block definitions (JSON)
    ├── control/
    │   ├── manual.start.json
    │   └── condition.if.json
    ├── data/
    │   ├── variable.set.json
    │   └── template.string.json
    ├── network/
    │   └── http.request.json
    ├── ai/
    │   └── ai.chat.json
    └── io/
        ├── file.read.json
        ├── file.write.json
        └── output.display.json
```

## Frontend Layer: Hybrid Canvas + Vanilla JS

**Combining Claude's vanilla approach with Codex's hybrid rendering:**

### Canvas Implementation (Hybrid)
- **Edges**: SVG for crisp vector connections (following Claude's approach)
- **Nodes**: HTML `<div>` elements for rich content rendering (Codex's insight)
- **Background**: SVG grid pattern for infinite canvas feel
- **Interactions**: Direct event handling on DOM elements

### State Management (Simple)
```javascript
// Simple, centralized state object (no external libraries)
const AppState = {
  workflows: new Map(),
  currentWorkflow: null,
  canvas: {
    nodes: new Map(),
    edges: new Map(),
    viewport: { x: 0, y: 0, scale: 1 },
    selection: new Set()
  },
  execution: {
    running: false,
    logs: [],
    results: new Map()
  }
}
```

### IPC Layer (Thin Wrappers)
```javascript
// Thin wrappers around Tauri commands
class IPC {
  static async invoke(command, payload) {
    return await window.__TAURI__.tauri.invoke(command, payload)
  }

  static async listen(event, callback) {
    return await window.__TAURI__.event.listen(event, callback)
  }
}
```

## Backend Layer: Modular Rust

**Adopting Codex's trait-based extensibility with Gemini's security focus:**

### Core Dependencies (Minimal)
```toml
[dependencies]
tauri = { version = "1.5", features = ["api-all"] }
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rusqlite = { version = "0.29", features = ["bundled"] }
reqwest = { version = "0.11", features = ["json", "stream"] }
thiserror = "1.0"
anyhow = "1.0"
uuid = { version = "1.0", features = ["v4"] }
tauri-plugin-stronghold = "1.0"  # Secure key storage
async-trait = "0.1"
```

### Command System (Clean Interface)
```rust
// Tauri commands with consistent error handling
#[tauri::command]
async fn save_workflow(
    workflow: Workflow,
    db: State<'_, Database>
) -> Result<String, String> {
    db.save_workflow(&workflow)
        .await
        .map_err(|e| e.to_string())
}

#[tauri::command]
async fn execute_workflow(
    workflow_id: String,
    executor: State<'_, WorkflowExecutor>,
    window: Window
) -> Result<String, String> {
    executor.execute_async(workflow_id, window)
        .await
        .map_err(|e| e.to_string())
}
```

## Security: Defense in Depth

**Following Codex's security principles with Gemini's key management:**

### API Key Security
- **Storage**: Encrypted in SQLite using `tauri-plugin-stronghold`
- **Access**: Only Rust backend can decrypt keys
- **Transmission**: Keys never sent to frontend
- **Rotation**: Easy key update via secure commands

### Frontend Security
- **CSP**: Locked down Content Security Policy
- **Prototype Freezing**: Freeze JS prototypes to prevent tampering
- **No Direct Network**: All network calls through Rust backend
- **Input Validation**: All user inputs validated in Rust

### System Security
- **File Access**: Limited to user-selected directories via Tauri dialogs
- **Network**: Only whitelisted domains for AI providers
- **Permissions**: Minimal permission model

## Communication: Events + Commands

**Tauri IPC for synchronous requests, Events for async streaming:**

### Commands (Request/Response)
```rust
// Synchronous operations
- save_workflow
- load_workflow
- get_block_descriptors
- set_api_key
```

### Events (Async Streaming)
```rust
// Asynchronous notifications
- execution:started
- execution:block_complete
- execution:log
- execution:error
- execution:complete
- ai:stream_chunk
- ai:stream_end
```

## Performance: Optimized for Desktop

### Frontend Optimizations
- **Canvas Virtualization**: Only render visible elements
- **Debounced Auto-save**: 500ms debounce for state changes
- **RequestAnimationFrame**: Smooth animations and interactions
- **DOM Recycling**: Reuse connection preview elements

### Backend Optimizations
- **Async Execution**: Non-blocking workflow execution
- **Connection Pooling**: Reuse HTTP connections for AI providers
- **SQLite WAL Mode**: Better concurrent access
- **Compressed Storage**: BLOB compression for large workflows

## Benefits of V2 Architecture

### From Claude (KISS)
- **Minimal Dependencies**: Only essential crates and libraries
- **Vanilla Frontend**: No framework complexity or build steps
- **Clear Separation**: Clean frontend/backend boundaries

### From Codex (Extensibility)
- **Generic Blocks**: JSON-defined blocks with trait handlers
- **Modular Structure**: Clean module organization
- **Security First**: Frozen prototypes and CSP

### From Gemini (Pragmatism)
- **Hybrid Rendering**: Best of SVG and HTML
- **Secure Storage**: Stronghold for sensitive data
- **Compressed Data**: Efficient workflow storage

## Implementation Benefits

1. **Bulletproof MVP**: Solid foundation for growth
2. **Fast Development**: Clear module boundaries
3. **Easy Testing**: Trait-based architecture enables mocking
4. **Secure by Default**: Multiple layers of security
5. **Desktop Native**: Optimized for desktop performance
6. **Future Proof**: Extensible without breaking changes

This V2 architecture provides a robust foundation that combines the best insights from all three planning approaches while maintaining the simplicity needed for a successful MVP.