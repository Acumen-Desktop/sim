# AI Integration Plan

## MVP Scope
- Start with single OpenAI-compatible chat completions endpoint (user supplies API key).
- Support streaming tokens back to frontend via Tauri events.
- Provide request inspector for debugging (show prompt, model, cost estimate).

## Backend Architecture
- `ai::ClientManager` keeps encrypted API keys (per provider) and instantiates HTTP clients with sane timeouts.
- `ai::providers::openai` implements chat + embeddings (optional) using `reqwest` and `serde_json`.
- Responses converted to simplified `AgentBlockOutput` structure reused from existing executor to preserve block compatibility.

## Key Storage & Security
- Prefer OS secure storage via `tauri-plugin-store` or `tauri-plugin-stronghold` for API keys; fallback to AES-encrypted SQLite column using master password stored in OS keyring.
- No network requests originate from frontend; Rust logs redacted request bodies before writing to console.

## Local / Offline Options
- Evaluate embedding a small local runtime for offline inference later:
  - **Ollama**: simple HTTP interface if the user already runs it; keep optional.
  - **Burn Runtime**: `tracel-ai/burn` supports ONNX import and multiple backends (CPU via `NdArray`, GPU via `Wgpu`). We can compile selected small models (e.g., Phi-2 distilled) and expose them through a `LocalModel` provider. Burn's backend router allows hybrid CPU/GPU execution when available.
  - Provide configuration file `~/.sim-desktop/models.toml` listing local models, backend preference, and context limits.

## Streaming & Cancellation
- Use `reqwest`'s streaming support to forward incremental tokens; push over Tauri `emit` with chunk metadata (block id, content, timestamp).
- Support cancellation by tracking `tokio::Select` on an `oneshot::Receiver`; expose `cancel_execution(execution_id)` command.

## Safety & Rate Limits
- Enforce simple rate limiter in Rust (token bucket per provider) to mimic `services/queue/RateLimiter.ts` without external deps.
- Surface provider errors clearly (quota, auth, network) with actionable hints in UI.

## Future Enhancements
- Provider abstraction trait so we can plug in Anthropic, Groq, etc. once core stable.
- Prompt templates stored locally with versioning; allow user edits via YAML files.
- On-device embedding generation for knowledge base once local model story solid.
