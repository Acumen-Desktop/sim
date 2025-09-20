# Database Options for Tauri Desktop Migration

## Current State: PostgreSQL + pgvector
- Complex setup requiring Docker or manual installation
- Heavy dependency for desktop application
- pgvector extension for embeddings (overkill for MVP)
- Network-based connection model

## Recommended: SQLite with rusqlite

### Why SQLite?
- **Zero Configuration**: Single file database, no server setup
- **Portable**: Database travels with the application
- **Fast**: Sufficient performance for desktop workflows
- **Simple**: No complex installation or maintenance
- **Tauri Compatible**: Direct rusqlite integration in Rust backend

### Implementation Strategy

#### Rust Backend (via Tauri)
```rust
use rusqlite::{Connection, Result};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Workflow {
    id: String,
    name: String,
    data: String, // JSON serialized workflow
    created_at: String,
    updated_at: String,
}

#[tauri::command]
fn save_workflow(workflow: Workflow) -> Result<(), String> {
    let conn = Connection::open("sim_desktop.db")?;
    // Insert workflow logic
    Ok(())
}
```

#### Simplified Schema
```sql
-- Workflows table
CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    data TEXT NOT NULL, -- JSON blob of entire workflow
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Simple settings storage
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

-- API keys storage (encrypted)
CREATE TABLE api_keys (
    provider TEXT PRIMARY KEY,
    key_data TEXT NOT NULL -- Encrypted key
);
```

## Embeddings Alternative

### Current: pgvector for AI embeddings
### MVP: Simple JSON storage

For knowledge base functionality, store embeddings as JSON arrays:
```sql
CREATE TABLE knowledge_items (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    embedding TEXT, -- JSON array of floats
    metadata TEXT   -- JSON metadata
);
```

**Pros:**
- No complex vector database
- Simple similarity search with basic math
- Sufficient for MVP with small datasets

**Cons:**
- Not optimized for large-scale vector operations
- Manual similarity calculations needed

### Future: Consider alternatives
- **Qdrant**: Rust-native vector database (when scaling up)
- **Chroma**: Python/SQLite hybrid (if needed)
- **Keep simple**: JSON embeddings work for desktop MVP

## Data Migration Strategy

### From PostgreSQL to SQLite
1. **Export current schema** as simplified SQLite tables
2. **Flatten complex relationships** into JSON blobs where appropriate
3. **Remove user/auth tables** (single-user desktop app)
4. **Combine related tables** into workflow JSON data

### Example Migration
```sql
-- Current: Multiple normalized tables
workflows, workflow_blocks, workflow_connections, block_configs

-- New: Single denormalized table
workflows (id, name, data) -- where data contains entire workflow as JSON
```

## File Storage

### Current: Cloud/S3 storage
### Desktop: Local file system

```rust
#[tauri::command]
fn save_file(filename: String, content: Vec<u8>) -> Result<String, String> {
    let app_dir = dirs::data_dir()
        .unwrap()
        .join("sim_desktop")
        .join("files");

    std::fs::create_dir_all(&app_dir)?;
    let file_path = app_dir.join(filename);
    std::fs::write(&file_path, content)?;

    Ok(file_path.to_string_lossy().to_string())
}
```

## Benefits of This Approach

1. **No Docker**: Eliminates container dependency
2. **No Network**: Local file-based database
3. **No Setup**: Works immediately after app installation
4. **Portable**: Database file can be backed up/shared easily
5. **Fast**: Local access, no network latency
6. **Simple**: Single file to manage

## Potential Limitations

1. **Concurrent Access**: SQLite has limitations with multiple writers
2. **Size Limits**: Large datasets may be slower than PostgreSQL
3. **Vector Operations**: Manual similarity search implementation needed
4. **Backup**: User responsible for backing up database file

## Recommendation

**Start with SQLite + rusqlite for MVP**. This provides the simplest path forward with zero configuration overhead. The performance will be adequate for single-user desktop workflows, and the simplicity aligns perfectly with the KISS principle.

If the application grows beyond desktop MVP scope, migration to a more complex database can be considered, but SQLite should handle thousands of workflows and reasonable knowledge base sizes without issues.