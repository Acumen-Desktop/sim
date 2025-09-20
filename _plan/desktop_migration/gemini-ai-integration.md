# Gemini's AI Integration Plan for Sim AI Desktop

## 1. Philosophy: Secure, Flexible, and Local-First

The AI integration will be designed to be secure, flexible, and aligned with the local-first philosophy of the application. This means direct, secure communication with AI providers from the Rust backend, a flexible provider system, and robust support for local inference.

## 2. Core Architecture: Rust-Powered and Abstracted

All AI-related operations will be handled by the Rust backend. The frontend will never make direct calls to AI providers. We will implement a provider abstraction system, as detailed by Claude, to support multiple AI services through a common interface.

### Provider Trait

A central `AIProvider` trait will define the contract for all AI services.

```rust
use async_trait::async_trait;

#[async_trait]
pub trait AIProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError>;
    async fn stream_completion(&self, request: AIRequest) -> Result<tokio::sync::mpsc::Receiver<String>, AIError>;
    fn provider_name(&self) -> &'static str;
}
```

### Provider Implementations

We will create concrete implementations of this trait for different providers.

*   **`OpenAIProvider`:** For OpenAI and compatible APIs.
*   **`AnthropicProvider`:** For Anthropic's models.
*   **`OllamaProvider`:** For local inference with Ollama.
*   **`BurnProvider`:** (Post-MVP) For running local models directly within the application using the Burn framework.

### AIManager

An `AIManager` struct will be responsible for managing the available providers and dispatching requests to the correct one.

## 3. Security: Secure API Key Management

API keys are sensitive data and will be handled with extreme care, following Codex's security recommendations.

*   **Storage:** Keys will be stored in the encrypted `secrets` table in the SQLite database.
*   **Encryption:** We will use `tauri-plugin-stronghold` or a similar solution to encrypt the keys at rest.
*   **Access:** The Rust backend will be the only component with access to the decrypted keys. They will never be exposed to the frontend.

## 4. Local Inference: Ollama and Burn

Supporting local inference is a key feature for a desktop application. We will provide two paths for this:

### Ollama Integration

*   We will create an `OllamaProvider` that communicates with a locally running Ollama instance via its REST API.
*   This is the easiest way to get started with local models, as it leverages an existing, popular tool.
*   The application will be able to auto-detect if Ollama is running and list the available models.

### Burn Framework (Post-MVP)

*   For a more integrated solution, we will explore using the [Burn](https://burn.dev/) framework to run models directly within the application.
*   This would eliminate the need for users to install Ollama separately.
*   We could bundle a small, capable model with the application for a completely offline-ready experience out of the box.

## 5. Streaming and Cancellation

*   **Streaming:** For a responsive user experience, we will stream responses from the AI providers token by token. The Rust backend will use `reqwest`'s streaming capabilities and forward the tokens to the frontend using Tauri events.
*   **Cancellation:** Long-running AI requests can be cancelled. The frontend will be able to send a `cancel_execution` command, which will cause the backend to drop the corresponding `tokio` task.

## 6. Frontend Integration

The frontend will interact with the AI capabilities through a simple JavaScript client that calls the backend via Tauri's IPC bridge.

```javascript
// Example of calling the backend from the frontend
async function executeAIBlock(block) {
    const { invoke, event } = window.__TAURI__;

    const stream = await invoke('call_ai_provider', {
        provider: block.config.provider,
        request: {
            model: block.config.model,
            messages: [/* ... */],
            stream: true,
        },
    });

    stream.onChunk((chunk) => {
        // Update the UI with the new token
    });
}
```

This AI integration plan provides a secure, flexible, and powerful foundation for Sim AI Desktop, enabling both cloud-based and local-first AI workflows.
