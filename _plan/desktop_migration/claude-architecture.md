# Tauri Desktop Architecture for Sim AI

## Current Web Architecture vs. Proposed Desktop Architecture

### Current (Web/Cloud):
```
Browser → Next.js (React) → Node.js/Bun → PostgreSQL + pgvector
   ↓
Socket.io → Trigger.dev → Docker → External Services
```

### Proposed (Desktop):
```
Tauri WebView → Vanilla HTML/CSS/JS → Rust Backend → SQLite
   ↓
Direct API calls → Local execution → File system
```

## Core Architecture Principles

### 1. Hybrid Desktop Model
- **Frontend**: WebView with vanilla web technologies (no frameworks)
- **Backend**: Rust binary for system operations and business logic
- **Communication**: Tauri IPC (Inter-Process Communication)
- **Storage**: Local SQLite database and file system

### 2. No External Dependencies
- **No Docker**: Pure executable application
- **No Servers**: Local-only execution
- **No Network Services**: Direct API calls only
- **No Build Steps**: Serve static HTML/CSS/JS directly

## Component Architecture

### Frontend Layer (WebView)
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Sim AI Desktop</title>
    <link rel="stylesheet" href="styles/app.css">
</head>
<body>
    <div id="app">
        <div id="canvas-container">
            <!-- Workflow canvas rendered here -->
        </div>
        <div id="sidebar">
            <!-- Block palette and tools -->
        </div>
    </div>

    <script src="js/canvas.js"></script>
    <script src="js/blocks.js"></script>
    <script src="js/app.js"></script>
</body>
</html>
```

**Responsibilities:**
- Render workflow canvas
- Handle user interactions (drag/drop, connections)
- Display block configurations
- Show execution results
- Manage UI state

### Backend Layer (Rust)
```rust
// src-tauri/src/main.rs
use tauri::{Builder, generate_context, generate_handler};

mod database;
mod executor;
mod ai_providers;
mod blocks;

fn main() {
    Builder::default()
        .manage(database::Database::new())
        .invoke_handler(generate_handler![
            save_workflow,
            load_workflow,
            execute_workflow,
            call_ai_provider,
            save_file,
            load_file
        ])
        .run(generate_context!())
        .expect("error while running tauri application");
}
```

**Responsibilities:**
- Workflow persistence (SQLite)
- Workflow execution engine
- AI provider integrations
- File system operations
- Application state management

### Communication Layer (IPC)
```javascript
// Frontend JavaScript
import { invoke } from '@tauri-apps/api/tauri';

async function saveWorkflow(workflow) {
    return await invoke('save_workflow', { workflow });
}

async function executeWorkflow(workflowId) {
    return await invoke('execute_workflow', { workflowId });
}
```

```rust
// Backend Rust
#[tauri::command]
async fn save_workflow(workflow: Workflow) -> Result<String, String> {
    // Save workflow to SQLite
    Ok(workflow.id)
}

#[tauri::command]
async fn execute_workflow(workflow_id: String) -> Result<ExecutionResult, String> {
    // Execute workflow blocks sequentially
    Ok(result)
}
```

## Simplified Data Flow

### 1. Workflow Creation
```
User creates block on canvas
→ Frontend updates local state
→ Calls save_workflow() via IPC
→ Rust saves to SQLite
→ Returns success to frontend
```

### 2. Workflow Execution
```
User clicks "Run"
→ Frontend calls execute_workflow()
→ Rust loads workflow from SQLite
→ Executes blocks sequentially
→ Returns results to frontend
→ Frontend displays execution logs
```

### 3. AI Integration
```
AI block executed
→ Rust calls AI provider API directly
→ Streams response back to frontend via events
→ Frontend updates block output display
```

## Directory Structure

### Tauri Project Layout
```
sim-desktop/
├── src-tauri/           # Rust backend
│   ├── src/
│   │   ├── main.rs      # App entry point
│   │   ├── database.rs  # SQLite operations
│   │   ├── executor.rs  # Workflow execution
│   │   ├── ai/          # AI provider integrations
│   │   └── blocks/      # Block type definitions
│   ├── Cargo.toml
│   └── tauri.conf.json
├── src/                 # Frontend (vanilla web)
│   ├── index.html
│   ├── styles/
│   │   ├── app.css      # Main styles
│   │   └── canvas.css   # Canvas-specific styles
│   └── js/
│       ├── app.js       # Main application logic
│       ├── canvas.js    # Canvas implementation
│       ├── blocks.js    # Block management
│       └── utils.js     # Utility functions
└── README.md
```

## Key Architectural Decisions

### 1. No Framework Frontend
- **Pure HTML/CSS/JS**: Eliminates build complexity
- **Direct DOM manipulation**: Simple and fast
- **Custom canvas implementation**: Tailored to needs
- **No bundling**: Faster development iteration

### 2. Rust Backend Benefits
- **Performance**: Fast execution engine
- **Safety**: Memory safety and error handling
- **Ecosystem**: Rich crates for AI, database, file operations
- **Cross-platform**: Single codebase for Windows/Mac/Linux

### 3. Local-First Design
- **No authentication**: Single-user desktop app
- **No cloud sync**: Local file storage only
- **No real-time collaboration**: Simplified UI
- **Direct API calls**: No proxy servers needed

### 4. Simplified State Management
- **Frontend**: Simple JavaScript object state
- **Backend**: SQLite as single source of truth
- **No complex state sync**: Direct database queries

## Performance Considerations

### Frontend Optimizations
- **Lazy block rendering**: Only render visible blocks
- **Canvas virtualization**: Efficient large workflow handling
- **Minimal DOM updates**: Direct manipulation where possible
- **Simple event handling**: No complex framework overhead

### Backend Optimizations
- **SQLite prepared statements**: Fast database access
- **Async execution**: Non-blocking AI API calls
- **Connection pooling**: Efficient resource usage
- **Error recovery**: Graceful handling of failures

## Security Considerations

### Data Protection
- **Local storage**: No data leaves the machine
- **API key encryption**: Secure storage in SQLite
- **File permissions**: Restrict access to app directory
- **Input validation**: Sanitize all user inputs

### Network Security
- **HTTPS only**: All AI provider calls over TLS
- **Certificate validation**: Verify SSL certificates
- **Rate limiting**: Prevent API abuse
- **Error sanitization**: Don't leak sensitive data in errors

## Benefits of This Architecture

1. **Simplicity**: No complex build tools or frameworks
2. **Performance**: Native desktop performance with Rust
3. **Reliability**: Fewer moving parts, less to break
4. **Portability**: Single executable with embedded resources
5. **Development Speed**: Fast iteration with vanilla technologies
6. **Maintainability**: Clear separation of concerns

## Potential Limitations

1. **Frontend Complexity**: Manual DOM management for complex UIs
2. **State Management**: Need careful coordination between frontend/backend
3. **Plugin System**: Less flexible than web-based extensibility
4. **Debugging**: Different tools needed for Rust vs. JavaScript

## Recommendation

This architecture prioritizes simplicity and maintainability over feature richness. For an MVP desktop application, this approach eliminates the complexity of modern web frameworks while providing a solid foundation for core workflow functionality.