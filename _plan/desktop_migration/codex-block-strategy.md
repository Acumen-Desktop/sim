# Generic Block Strategy

## Philosophy
- Treat blocks as pluggable "operations" with declarative metadata so we can pivot between workflow automation (n8n-style) and AI orchestration without rewriting the canvas or executor.
- Favor a lean core catalog with clear extension points (JSON descriptors + Rust handlers) rather than hard-coding vertical-specific logic.

## Block Descriptor Shape
```json
{
  "id": "http.request",
  "label": "HTTP Request",
  "category": "network",
  "inputs": [
    { "id": "url", "type": "string", "required": true },
    { "id": "method", "type": "enum", "options": ["GET", "POST", "PUT", "DELETE"], "default": "GET" },
    { "id": "body", "type": "json", "required": false }
  ],
  "outputs": [
    { "id": "body", "type": "json" },
    { "id": "status", "type": "number" }
  ],
  "config": {
    "timeout": { "type": "number", "default": 30000 }
  }
}
```
- Frontend renders forms from descriptors; executor loads matching handler via registry.
- Supports future dynamic loading from local `blocks/` directory.

## MVP Core Categories
1. **Control**
   - `start.manual`: entry point triggered by user.
   - `condition.basic`: evaluate boolean rule (`if`, `else`).
2. **Data**
   - `variable.set` / `variable.get`: simple key-value to pass data between steps.
   - `transform.json`: apply template / map fields (using handlebars-style syntax).
3. **Network / IO**
   - `http.request`: generic REST call.
   - `file.read` / `file.write`: interact with local filesystem (opt-in permissions).
4. **AI**
   - `ai.chat`: send prompt to configured provider; return text stream + tokens.
   - `ai.embedding` (optional stretch if we need vector output later).
5. **Output**
   - `response.display`: final result aggregator shown in UI.
   - `notification.local`: send desktop notification (invoke OS toast).

This set maps to core automation primitives and can be extended quickly; no domain-specific connectors baked in.

## Extension Mechanism
- Blocks defined in `blocks/registry.json` with pointers to Rust handler module name.
- Handlers implement trait:
  ```rust
  #[async_trait]
  pub trait BlockExecutor {
      fn descriptor(&self) -> &'static BlockDescriptor;
      async fn execute(&self, ctx: &ExecutionContext) -> anyhow::Result<BlockOutput>;
  }
  ```
- New blocks can be dropped in by shipping additional Rust modules + descriptor entries without touching canvas code.

## Migration Aid
- Existing cloud workflows can be mapped by alias table (e.g., `starter` → `start.manual`, `agent` → `ai.chat`).
- Unsupported legacy blocks fall back to "unsupported" placeholder on import; user can swap to equivalent generic block or custom script.

## Future Flexibility
- Support scripting block (`code.rhai`) to let power users run short scripts locally.
- Allow community-owned block packs by loading descriptors from user directory and dynamically registering handlers via Rust plugin trait objects.
