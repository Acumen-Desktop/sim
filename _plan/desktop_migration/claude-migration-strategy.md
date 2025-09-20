# Migration Strategy: Web to Tauri Desktop

## Migration Philosophy: Fresh Start, Not Port

**Key Decision**: Start fresh with Tauri rather than attempting to port the existing Next.js/React codebase. This allows us to eliminate complexity and embrace desktop-native patterns from the beginning.

## Phase-Based Migration Approach

### Phase 1: Foundation Setup (Week 1-2)

#### Tauri Project Initialization
```bash
# Create new Tauri project
cargo install tauri-cli
cargo tauri init sim-desktop

# Project structure setup
src-tauri/
├── src/main.rs
├── Cargo.toml
└── tauri.conf.json

src/
├── index.html
├── styles/app.css
└── js/app.js
```

#### Essential Dependencies
```toml
[dependencies]
tauri = { version = "1.0", features = ["api-all"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1.0", features = ["full"] }
rusqlite = { version = "0.29", features = ["bundled"] }
reqwest = { version = "0.11", features = ["json", "stream"] }
thiserror = "1.0"
```

#### Window Configuration
```json
{
  "build": {
    "distDir": "../src",
    "devPath": "../src"
  },
  "tauri": {
    "windows": [{
      "title": "Sim AI Desktop",
      "width": 1400,
      "height": 900,
      "minWidth": 800,
      "minHeight": 600,
      "resizable": true,
      "maximizable": true
    }]
  }
}
```

#### Minimal HTML Shell
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
        <div id="header">
            <h1>Sim AI Desktop</h1>
            <div id="controls"></div>
        </div>
        <div id="main">
            <div id="canvas-container"></div>
            <div id="sidebar"></div>
        </div>
    </div>
    <script src="js/app.js"></script>
</body>
</html>
```

### Phase 2: Data Layer Migration (Week 3)

#### Database Schema Simplification
```sql
-- From complex PostgreSQL schema to simple SQLite
CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    data TEXT NOT NULL, -- JSON blob containing entire workflow
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

CREATE TABLE api_keys (
    provider TEXT PRIMARY KEY,
    encrypted_key TEXT NOT NULL
);
```

#### Data Migration Script (Rust)
```rust
// Optional: Extract data from existing PostgreSQL
#[tauri::command]
async fn import_legacy_workflows(postgres_url: String) -> Result<Vec<String>, String> {
    // Connect to existing PostgreSQL
    // Extract workflow data
    // Convert to simplified format
    // Import into SQLite
    Ok(imported_ids)
}
```

#### Storage Implementation
```rust
// src-tauri/src/storage/mod.rs
pub struct Database {
    conn: rusqlite::Connection,
}

impl Database {
    pub fn new() -> Result<Self, rusqlite::Error> {
        let conn = rusqlite::Connection::open("sim_desktop.db")?;
        // Initialize schema
        Ok(Self { conn })
    }

    pub fn save_workflow(&self, workflow: &Workflow) -> Result<(), rusqlite::Error> {
        // Implement workflow persistence
    }
}
```

### Phase 3: Canvas Implementation (Week 4-5)

#### Extract Canvas Logic from ReactFlow
**Current**: Complex React components with hooks
**Target**: Vanilla JavaScript with direct DOM manipulation

```javascript
// src/js/canvas.js - New implementation
class WorkflowCanvas {
    constructor(containerId) {
        this.container = document.getElementById(containerId);
        this.svg = this.createSVG();
        this.blocks = new Map();
        this.connections = new Map();
        this.transform = { x: 0, y: 0, scale: 1 };

        this.initializeEventListeners();
    }

    createSVG() {
        const svg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
        svg.setAttribute('width', '100%');
        svg.setAttribute('height', '100%');
        svg.setAttribute('id', 'workflow-canvas');

        this.container.appendChild(svg);
        return svg;
    }

    // Implement methods from claude-canvas.md
}
```

#### Block System Migration
**Extract block types from**: `apps/sim/blocks/`
**Target structure**:
```rust
// src-tauri/src/blocks/mod.rs
pub enum BlockType {
    Input,
    Output,
    AIAgent,
    HTTP,
    Condition,
    Text,
}

pub struct Block {
    pub id: String,
    pub block_type: BlockType,
    pub position: Position,
    pub config: serde_json::Value,
}
```

### Phase 4: AI Integration Migration (Week 6-7)

#### Provider System
**Extract from**: `apps/sim/providers/`
**Simplify to**: Direct API calls without complex auth flows

```rust
// Migrate from:
// - Complex OAuth flows
// - Server-side API management
// - Streaming via Server-Sent Events

// To:
// - Direct API calls from Rust
// - Local API key storage
// - Tauri event streaming
```

#### API Key Management
```rust
#[tauri::command]
async fn set_api_key(provider: String, key: String) -> Result<(), String> {
    // Encrypt and store locally
    let encrypted = encrypt_api_key(&key)?;
    Database::store_api_key(&provider, &encrypted)?;
    Ok(())
}
```

### Phase 5: Execution Engine Migration (Week 8-9)

#### Executor Simplification
**Extract from**: `apps/sim/executor/`
**Simplify to**: Sequential execution without complex features

```rust
// Remove complex features:
// - Loop management
// - Parallel execution
// - Complex path tracking
// - Background job integration

// Keep core:
// - Sequential block execution
// - Variable passing
// - Error handling
// - Basic logging
```

#### Execution Implementation
```rust
pub struct WorkflowExecutor {
    pub async fn execute(&self, workflow: &Workflow) -> Result<ExecutionResult, ExecutionError> {
        let mut context = ExecutionContext::new();

        for block in &workflow.blocks {
            let result = self.execute_block(block, &context).await?;
            context.set_variable(&block.id, result);
        }

        Ok(ExecutionResult::success(context))
    }
}
```

### Phase 6: Feature Parity (Week 10-11)

#### Essential Features Migration Priority
1. **Workflow save/load** (local files instead of database)
2. **Basic block types** (Input, Output, AI Agent, HTTP)
3. **AI streaming** (via Tauri events instead of SSE)
4. **Settings management** (local storage instead of user accounts)

#### Removed Features (Intentionally)
- User authentication and accounts
- Real-time collaboration
- Cloud sync and sharing
- Complex scheduling and triggers
- Advanced tool integrations
- Background job processing

### Phase 7: Testing and Polish (Week 12)

#### Cross-Platform Testing
```bash
# Build for all platforms
cargo tauri build --target x86_64-pc-windows-msvc
cargo tauri build --target x86_64-apple-darwin
cargo tauri build --target x86_64-unknown-linux-gnu
```

#### Performance Optimization
- Canvas rendering optimization
- Memory usage monitoring
- Startup time optimization
- Bundle size minimization

## Data Migration Tools

### Workflow Export Tool (Optional)
If users want to migrate existing workflows:

```rust
#[tauri::command]
async fn export_workflows_from_web(export_url: String) -> Result<Vec<Workflow>, String> {
    // Connect to existing Sim instance
    // Export workflows as JSON
    // Convert to desktop format
    Ok(workflows)
}
```

### Import Existing Workflows
```rust
#[tauri::command]
async fn import_workflow_file(file_path: String) -> Result<String, String> {
    // Read JSON file
    // Validate workflow structure
    // Import into local database
    Ok(workflow_id)
}
```

## Key Architectural Changes

### From Complex to Simple

#### Authentication
**Before**: Complex user management, JWT tokens, OAuth
**After**: Single-user desktop app, no authentication

#### Storage
**Before**: PostgreSQL with complex relations, pgvector
**After**: SQLite with simple schema, JSON blobs

#### Frontend
**Before**: React, Next.js, complex state management
**After**: Vanilla JavaScript, direct DOM manipulation

#### Backend
**Before**: Node.js/Bun, complex API routes, middleware
**After**: Rust with Tauri, simple command handlers

#### Deployment
**Before**: Docker containers, cloud infrastructure
**After**: Single executable, local installation

## Risk Mitigation

### Technical Risks
1. **Canvas Performance**: Profile and optimize SVG rendering
2. **Cross-Platform Issues**: Test early and often on all platforms
3. **API Rate Limits**: Implement proper rate limiting and error handling
4. **Data Loss**: Implement robust save/backup mechanisms

### User Experience Risks
1. **Learning Curve**: Keep UI similar to web version where possible
2. **Feature Parity**: Clearly communicate MVP scope
3. **Migration Path**: Provide clear import tools for existing users

## Success Metrics

### Technical Metrics
- Application startup time < 2 seconds
- Workflow execution latency < 500ms (excluding AI calls)
- Memory usage < 200MB for typical workflows
- Binary size < 50MB

### User Metrics
- Can complete basic workflow creation in < 5 minutes
- Successful AI integration setup in < 2 minutes
- Zero-configuration installation and first run

## Go-Live Strategy

### Beta Release
1. Internal testing with simplified workflows
2. Limited beta with power users
3. Gather feedback and iterate

### Full Release
1. Parallel deployment with web version
2. Clear migration documentation
3. Feature comparison guide
4. User support resources

This migration strategy prioritizes simplicity and maintainability over feature completeness, aligning with the KISS principle and desktop-native user expectations.