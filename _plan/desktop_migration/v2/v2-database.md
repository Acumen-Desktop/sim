# V2 Database Design: Optimized SQLite with Security

## Design Philosophy

This V2 database design combines **Claude's simplicity** with **Gemini's compressed storage** and **Codex's security focus**. The result is a lean, performant, and secure local database optimized for desktop workflows.

## Core Technology: SQLite with Modern Features

### Why SQLite for V2?
- **Zero Configuration**: No server setup or administration
- **ACID Compliance**: Reliable transactions and data integrity
- **Single File**: Entire database in one portable file
- **Excellent Rust Support**: `rusqlite` provides safe, performant access
- **WAL Mode**: Better concurrent access for desktop apps
- **Built-in Compression**: BLOB compression for large workflows
- **Full-text Search**: Built-in FTS for workflow search

### Database Configuration
```sql
-- Enable Write-Ahead Logging for better performance
PRAGMA journal_mode = WAL;

-- Enable foreign key constraints
PRAGMA foreign_keys = ON;

-- Set secure deletion (overwrite deleted data)
PRAGMA secure_delete = ON;

-- Optimize for desktop usage
PRAGMA cache_size = -64000;  -- 64MB cache
PRAGMA temp_store = memory;
```

## Schema Design: Lean and Efficient

### Core Tables

```sql
-- Workflow storage with compressed data
CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,

    -- Compressed workflow data (JSON -> gzip -> BLOB)
    data BLOB NOT NULL,

    -- Metadata for search and organization
    tags TEXT,  -- JSON array of tags
    category TEXT DEFAULT 'general',

    -- Timestamps
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),

    -- Search index
    search_text TEXT GENERATED ALWAYS AS (
        name || ' ' || COALESCE(description, '') || ' ' || COALESCE(tags, '')
    ) STORED
);

-- Full-text search index for workflows
CREATE VIRTUAL TABLE workflows_fts USING fts5(
    name, description, tags, content='workflows', content_rowid='rowid'
);

-- Triggers to keep FTS in sync
CREATE TRIGGER workflows_fts_insert AFTER INSERT ON workflows BEGIN
    INSERT INTO workflows_fts(rowid, name, description, tags)
    VALUES (new.rowid, new.name, new.description, new.tags);
END;

CREATE TRIGGER workflows_fts_delete AFTER DELETE ON workflows BEGIN
    INSERT INTO workflows_fts(workflows_fts, rowid, name, description, tags)
    VALUES('delete', old.rowid, old.name, old.description, old.tags);
END;

CREATE TRIGGER workflows_fts_update AFTER UPDATE ON workflows BEGIN
    INSERT INTO workflows_fts(workflows_fts, rowid, name, description, tags)
    VALUES('delete', old.rowid, old.name, old.description, old.tags);
    INSERT INTO workflows_fts(rowid, name, description, tags)
    VALUES (new.rowid, new.name, new.description, new.tags);
END;
```

```sql
-- Execution history with summary data
CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL,

    -- Execution metadata
    status TEXT NOT NULL CHECK (status IN ('running', 'success', 'error', 'cancelled')),
    started_at INTEGER NOT NULL,
    completed_at INTEGER,
    duration_ms INTEGER GENERATED ALWAYS AS (
        CASE WHEN completed_at IS NOT NULL
        THEN (completed_at - started_at) * 1000
        ELSE NULL END
    ) STORED,

    -- Results summary (compressed JSON)
    summary BLOB,  -- Compressed execution summary
    error_message TEXT,

    -- Performance metrics
    blocks_executed INTEGER DEFAULT 0,
    tokens_used INTEGER DEFAULT 0,
    api_calls_made INTEGER DEFAULT 0,

    FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE
);

-- Index for run queries
CREATE INDEX idx_runs_workflow_started ON runs(workflow_id, started_at DESC);
CREATE INDEX idx_runs_status_started ON runs(status, started_at DESC);
```

```sql
-- Application settings with versioning
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    value_type TEXT NOT NULL DEFAULT 'string' CHECK (value_type IN ('string', 'number', 'boolean', 'json')),
    description TEXT,
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

-- Default settings
INSERT INTO settings (key, value, value_type, description) VALUES
('app_version', '1.0.0', 'string', 'Application version'),
('theme', 'system', 'string', 'UI theme preference'),
('auto_save_interval', '30', 'number', 'Auto-save interval in seconds'),
('default_ai_provider', 'openai', 'string', 'Default AI provider'),
('execution_timeout', '300', 'number', 'Workflow execution timeout in seconds'),
('canvas_grid_size', '20', 'number', 'Canvas grid size in pixels'),
('enable_telemetry', 'false', 'boolean', 'Enable usage telemetry');
```

### Security Tables

```sql
-- Encrypted secrets storage (using Tauri Stronghold)
CREATE TABLE secrets (
    provider TEXT PRIMARY KEY,
    encrypted_value BLOB NOT NULL,  -- Stronghold encrypted data
    key_hash TEXT NOT NULL,         -- Hash of the key for validation
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

-- Access log for security auditing
CREATE TABLE access_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    action TEXT NOT NULL,  -- 'key_access', 'key_update', 'workflow_export', etc.
    resource TEXT NOT NULL,  -- Resource identifier
    timestamp INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    details TEXT  -- JSON with additional context
);

-- Index for security queries
CREATE INDEX idx_access_log_timestamp ON access_log(timestamp DESC);
CREATE INDEX idx_access_log_action ON access_log(action, timestamp DESC);
```

## Data Compression Strategy

### Workflow Data Compression
```rust
use flate2::{write::GzEncoder, read::GzDecoder, Compression};
use std::io::{Write, Read};

pub fn compress_workflow(workflow: &Workflow) -> Result<Vec<u8>, DatabaseError> {
    let json = serde_json::to_string(workflow)?;
    let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(json.as_bytes())?;
    Ok(encoder.finish()?)
}

pub fn decompress_workflow(compressed: &[u8]) -> Result<Workflow, DatabaseError> {
    let mut decoder = GzDecoder::new(compressed);
    let mut json = String::new();
    decoder.read_to_string(&mut json)?;
    Ok(serde_json::from_str(&json)?)
}
```

### Benefits of Compression
- **Storage Efficiency**: 60-80% size reduction for large workflows
- **Network Efficiency**: Faster backup/sync operations
- **Memory Efficiency**: Smaller memory footprint
- **I/O Efficiency**: Fewer disk operations

## Database Operations Layer

### Connection Management
```rust
use rusqlite::{Connection, Result};
use std::sync::{Arc, Mutex};

pub struct Database {
    conn: Arc<Mutex<Connection>>,
}

impl Database {
    pub fn new(db_path: &str) -> Result<Self> {
        let conn = Connection::open(db_path)?;

        // Configure SQLite for desktop performance
        conn.execute_batch("
            PRAGMA journal_mode = WAL;
            PRAGMA foreign_keys = ON;
            PRAGMA cache_size = -64000;
            PRAGMA temp_store = memory;
            PRAGMA secure_delete = ON;
        ")?;

        Ok(Self {
            conn: Arc::new(Mutex::new(conn)),
        })
    }

    pub async fn save_workflow(&self, workflow: &Workflow) -> Result<()> {
        let compressed_data = compress_workflow(workflow)?;
        let conn = self.conn.lock().unwrap();

        conn.execute(
            "INSERT OR REPLACE INTO workflows
             (id, name, description, data, tags, category, updated_at)
             VALUES (?1, ?2, ?3, ?4, ?5, ?6, strftime('%s', 'now'))",
            (
                &workflow.id,
                &workflow.name,
                &workflow.description,
                &compressed_data,
                &serde_json::to_string(&workflow.tags)?,
                &workflow.category,
            ),
        )?;

        Ok(())
    }
}
```

### Query Optimization
```sql
-- Indexes for common queries
CREATE INDEX idx_workflows_category ON workflows(category);
CREATE INDEX idx_workflows_updated ON workflows(updated_at DESC);
CREATE INDEX idx_workflows_name ON workflows(name);

-- Composite indexes for complex queries
CREATE INDEX idx_runs_workflow_status ON runs(workflow_id, status);
CREATE INDEX idx_runs_performance ON runs(started_at, duration_ms) WHERE status = 'success';
```

## Backup and Recovery

### Automated Backup Strategy
```rust
pub struct BackupManager {
    db: Arc<Database>,
    backup_dir: PathBuf,
}

impl BackupManager {
    pub async fn create_backup(&self) -> Result<PathBuf> {
        let timestamp = chrono::Utc::now().format("%Y%m%d_%H%M%S");
        let backup_path = self.backup_dir.join(format!("sim_backup_{}.db", timestamp));

        // Use SQLite's backup API for consistent snapshots
        let conn = self.db.conn.lock().unwrap();
        let backup_conn = Connection::open(&backup_path)?;

        let backup = rusqlite::backup::Backup::new(&conn, &backup_conn)?;
        backup.run_to_completion(5, Duration::from_millis(100), None)?;

        // Compress the backup
        self.compress_backup(&backup_path).await?;

        Ok(backup_path)
    }

    pub async fn restore_backup(&self, backup_path: &Path) -> Result<()> {
        // Validate backup integrity before restore
        self.validate_backup(backup_path).await?;

        // Restore with transaction safety
        let conn = self.db.conn.lock().unwrap();
        conn.execute_batch("BEGIN IMMEDIATE;")?;

        // Restore logic here...

        conn.execute_batch("COMMIT;")?;
        Ok(())
    }
}
```

### Data Migration Support
```rust
pub struct MigrationManager {
    db: Arc<Database>,
}

impl MigrationManager {
    pub async fn migrate(&self) -> Result<()> {
        let current_version = self.get_schema_version().await?;

        match current_version {
            0 => self.migrate_to_v1().await?,
            1 => self.migrate_to_v2().await?,
            // ... additional migrations
            _ => {} // Already at latest version
        }

        Ok(())
    }

    async fn migrate_to_v1(&self) -> Result<()> {
        let conn = self.db.conn.lock().unwrap();
        conn.execute_batch("
            BEGIN TRANSACTION;

            -- Migration SQL here
            ALTER TABLE workflows ADD COLUMN search_text TEXT;

            -- Update schema version
            INSERT OR REPLACE INTO settings (key, value) VALUES ('schema_version', '1');

            COMMIT;
        ")?;
        Ok(())
    }
}
```

## Performance Characteristics

### Expected Performance
- **Workflow Save**: < 10ms for typical workflows
- **Workflow Load**: < 5ms with decompression
- **Search**: < 50ms for full-text search across 1000+ workflows
- **Run History**: < 20ms for paginated queries
- **Database Size**: ~50% smaller than uncompressed JSON storage

### Optimization Strategies
1. **Connection Pooling**: Single connection with mutex for desktop apps
2. **Prepared Statements**: Reuse compiled SQL for common operations
3. **Batch Operations**: Group multiple inserts in transactions
4. **Compression**: Reduce I/O overhead for large workflows
5. **Indexing**: Strategic indexes for common query patterns

## Security Features

### Data Protection
- **Encryption at Rest**: Stronghold for sensitive data
- **Secure Deletion**: PRAGMA secure_delete enabled
- **Access Logging**: Audit trail for security events
- **Input Validation**: All inputs validated and sanitized
- **SQL Injection Protection**: Parameterized queries only

### Key Management Integration
```rust
use tauri_plugin_stronghold::StrongholdStore;

pub struct SecureKeyManager {
    stronghold: StrongholdStore,
    db: Arc<Database>,
}

impl SecureKeyManager {
    pub async fn store_api_key(&self, provider: &str, key: &str) -> Result<()> {
        // Encrypt with Stronghold
        let encrypted = self.stronghold.encrypt(key.as_bytes()).await?;
        let key_hash = sha256::digest(key);

        // Store encrypted data and hash
        let conn = self.db.conn.lock().unwrap();
        conn.execute(
            "INSERT OR REPLACE INTO secrets
             (provider, encrypted_value, key_hash, updated_at)
             VALUES (?1, ?2, ?3, strftime('%s', 'now'))",
            (provider, &encrypted, &key_hash),
        )?;

        // Log access
        self.log_access("key_update", provider).await?;

        Ok(())
    }
}
```

## Benefits of V2 Database Design

### Performance Benefits
- **Compressed Storage**: 60-80% space savings
- **Fast Queries**: Optimized indexes and FTS
- **WAL Mode**: Better concurrent access
- **In-Memory Temp**: Faster temporary operations

### Security Benefits
- **Encrypted Secrets**: Stronghold integration
- **Access Auditing**: Complete access logs
- **Secure Deletion**: Overwrite deleted data
- **Input Validation**: Prevent injection attacks

### Maintainability Benefits
- **Single File**: Easy backup and migration
- **Version Control**: Schema migration system
- **Clear Structure**: Well-organized tables
- **Good Documentation**: Self-documenting schema

This V2 database design provides a solid, secure, and performant foundation for the desktop application while maintaining the simplicity needed for reliable operation.