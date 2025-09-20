# Local Data Persistence Options

## Requirements
- Single-user desktop storage with easy backup/export.
- Simple schema migrating existing workflows (id, name, JSON payload, timestamps).
- Store execution history + settings without Postgres.
- Allow optional encryption for sensitive data (API keys).

## Option 1: SQLite (Recommended)
- Bundled via `rusqlite` (`bundled` feature ensures cross-platform sqlite3).
- Schema sketch:
  ```sql
  CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    data BLOB NOT NULL, -- compressed JSON
    updated_at INTEGER NOT NULL
  );
  CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL,
    status TEXT NOT NULL,
    started_at INTEGER NOT NULL,
    duration_ms INTEGER,
    summary_json TEXT,
    FOREIGN KEY (workflow_id) REFERENCES workflows(id)
  );
  CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL
  );
  CREATE TABLE secrets (
    provider TEXT PRIMARY KEY,
    encrypted_value BLOB NOT NULL
  );
  ```
- Pros: ACID, queryable logs, concurrency safe, easy export.
- Cons: Need migration story (use `rusqlite_migration` or simple `PRAGMA user_version`).

## Option 2: JSON Files
- Store each workflow as `~/SimAI/workflows/<id>.json`, runs as newline-delimited logs.
- Pros: Human-readable, trivial backups.
- Cons: Harder to query, risk of partial writes, no transactions.
- Use as fallback import/export format rather than primary store.

## Option 3: Key-Value Store (Sled, Redb)
- Simple API but adds native dependency weight; limited tooling compared to SQLite.
- Not necessary for MVP unless SQLite footprint becomes an issue.

## Migration Path from Cloud
- Provide import command to read legacy workflow JSON exported via existing web app (`serializer` output remains compatible).
- Optional utility to connect to cloud Postgres one-time and extract workflows into local SQLite (Rust CLI script, not part of runtime app).

## Backup & Sync Strategy
- Daily auto-backup to zipped archive stored in `~/SimAI/backups/` (configurable retention).
- Manual export button in UI to produce `.simworkflow` bundle (workflow + assets + metadata).
- Document how to move database between machines; avoid hidden directories.

## Encryption Considerations
- Only secrets need encryption at rest; workflows/runs can remain plain unless user requests.
- Use `ring` crate for AES-GCM with key derived from OS keyring token; if unavailable, prompt user for master password stored in memory only.
