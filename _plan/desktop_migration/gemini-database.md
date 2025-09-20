# Gemini's Database Plan for Sim AI Desktop

## 1. Core Technology: SQLite

As recommended by both Claude and Codex, we will use SQLite as the database for the desktop application. This choice is ideal for a local-first desktop application for several reasons:

*   **Serverless:** SQLite is a self-contained, serverless database engine.
*   **Zero-Configuration:** It requires no complex setup or administration.
*   **Portable:** The entire database is stored in a single file, making it easy to back up and move.
*   **Reliable:** SQLite is ACID-compliant, ensuring data integrity.
*   **Excellent Rust Support:** The `rusqlite` crate provides a simple and efficient interface to SQLite.

## 2. Database Schema

We will adopt a schema that is a simplified version of the existing PostgreSQL schema, incorporating the detailed suggestions from Codex. This provides a good balance between simplicity and the ability to store structured data for workflows, execution runs, and settings.

```sql
-- Stores the main workflow data
CREATE TABLE workflows (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  data BLOB NOT NULL, -- The entire workflow, serialized to JSON and compressed
  created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
  updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

-- Stores a history of workflow executions
CREATE TABLE runs (
  id TEXT PRIMARY KEY,
  workflow_id TEXT NOT NULL,
  status TEXT NOT NULL, -- e.g., 'success', 'error'
  started_at INTEGER NOT NULL,
  duration_ms INTEGER,
  summary_json TEXT, -- A JSON object with a summary of the execution
  FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE
);

-- A simple key-value store for application settings
CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

-- A secure store for API keys and other secrets
CREATE TABLE secrets (
  provider TEXT PRIMARY KEY,
  encrypted_value BLOB NOT NULL
);
```

### Schema Notes:

*   **`workflows.data`**: Storing the workflow as a `BLOB` (binary large object) allows us to compress the JSON data, which can save space and improve performance.
*   **`runs` table:** This table will store a summary of each execution, not the full logs. This keeps the database size manageable. Full logs can be stored as separate files if needed.
*   **`secrets` table:** The `encrypted_value` will be encrypted using a strong encryption algorithm like AES-GCM. The encryption key will be managed by a secrets management plugin.

## 3. Secrets Management

API keys and other secrets will be stored encrypted in the database. We will use a dedicated Tauri plugin for this, as recommended by Codex.

*   **Primary Choice:** `tauri-plugin-stronghold`. This plugin uses the IOTA Stronghold library to provide a secure, encrypted database for secrets.
*   **Fallback:** `tauri-plugin-store` with an encryption layer. If Stronghold proves to be too heavy, we can use the simpler key-value store plugin and encrypt the values manually using a crate like `ring`.

The master encryption key will be derived from the user's OS-level keyring or a user-provided password.

## 4. Data Migration

Since we are starting with a fresh database, there is no direct data migration from the existing PostgreSQL database. However, we will provide tools to import and export workflows.

*   **Import:** Users will be able to import workflows from JSON files that are compatible with the existing web application's format.
*   **Export:** Users will be able to export their workflows to JSON files for backup or sharing.

An optional, separate CLI tool could be created to connect to a user's existing PostgreSQL database and export their workflows to the new format.

## 5. Backup and Portability

The entire database will be contained in a single file (e.g., `sim_ai_desktop.db`) located in the user's application data directory. This makes it easy for users to back up their data by simply copying the file. We will also provide a built-in backup utility to create compressed archives of the database.

This database strategy provides a robust and secure foundation for the Sim AI desktop application, while remaining simple, local-first, and user-friendly.
