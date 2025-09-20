# AI Integration for Tauri Desktop

## Current Web Implementation
- Multiple provider integrations (OpenAI, Anthropic, Google, etc.)
- Complex authentication flows
- Server-side API key management
- Streaming responses via Server-Sent Events

## Proposed Desktop Implementation

### Core Design Principles
1. **Direct API Calls**: Rust backend communicates directly with AI providers
2. **Local Storage**: API keys stored securely on device
3. **Streaming Support**: Real-time response streaming via Tauri events
4. **Provider Abstraction**: Unified interface for different AI services
5. **Local Inference Option**: Support for local models via Burn framework

## AI Provider Architecture

### Provider Trait System

```rust
// src-tauri/src/ai/mod.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIRequest {
    pub model: String,
    pub messages: Vec<Message>,
    pub stream: bool,
    pub max_tokens: Option<u32>,
    pub temperature: Option<f32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Message {
    pub role: String,
    pub content: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIResponse {
    pub content: String,
    pub finish_reason: Option<String>,
    pub usage: Option<Usage>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Usage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

#[async_trait]
pub trait AIProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError>;
    async fn stream_completion(&self, request: AIRequest) -> Result<tokio::sync::mpsc::Receiver<String>, AIError>;
    fn provider_name(&self) -> &'static str;
}

#[derive(Debug, thiserror::Error)]
pub enum AIError {
    #[error("API Error: {0}")]
    ApiError(String),
    #[error("Network Error: {0}")]
    NetworkError(String),
    #[error("Authentication Error")]
    AuthError,
    #[error("Rate Limit Exceeded")]
    RateLimitError,
}
```

### OpenAI Provider Implementation

```rust
// src-tauri/src/ai/openai.rs
use super::{AIProvider, AIRequest, AIResponse, AIError};
use async_trait::async_trait;
use reqwest::Client;
use serde_json::json;

pub struct OpenAIProvider {
    client: Client,
    api_key: String,
    base_url: String,
}

impl OpenAIProvider {
    pub fn new(api_key: String) -> Self {
        Self {
            client: Client::new(),
            api_key,
            base_url: "https://api.openai.com/v1".to_string(),
        }
    }
}

#[async_trait]
impl AIProvider for OpenAIProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        let payload = json!({
            "model": request.model,
            "messages": request.messages,
            "max_tokens": request.max_tokens,
            "temperature": request.temperature,
            "stream": false
        });

        let response = self.client
            .post(&format!("{}/chat/completions", self.base_url))
            .header("Authorization", format!("Bearer {}", self.api_key))
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        if !response.status().is_success() {
            return Err(AIError::ApiError(format!("HTTP {}", response.status())));
        }

        let response_body: serde_json::Value = response.json().await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        let content = response_body["choices"][0]["message"]["content"]
            .as_str()
            .unwrap_or("")
            .to_string();

        Ok(AIResponse {
            content,
            finish_reason: response_body["choices"][0]["finish_reason"]
                .as_str()
                .map(|s| s.to_string()),
            usage: None, // Parse usage from response_body if needed
        })
    }

    async fn stream_completion(&self, request: AIRequest) -> Result<tokio::sync::mpsc::Receiver<String>, AIError> {
        let (tx, rx) = tokio::sync::mpsc::channel(100);

        let payload = json!({
            "model": request.model,
            "messages": request.messages,
            "max_tokens": request.max_tokens,
            "temperature": request.temperature,
            "stream": true
        });

        let client = self.client.clone();
        let api_key = self.api_key.clone();
        let base_url = self.base_url.clone();

        tokio::spawn(async move {
            // Implement Server-Sent Events streaming
            // Parse each chunk and send via tx
        });

        Ok(rx)
    }

    fn provider_name(&self) -> &'static str {
        "openai"
    }
}
```

### Anthropic Provider Implementation

```rust
// src-tauri/src/ai/anthropic.rs
use super::{AIProvider, AIRequest, AIResponse, AIError};
use async_trait::async_trait;

pub struct AnthropicProvider {
    client: reqwest::Client,
    api_key: String,
}

impl AnthropicProvider {
    pub fn new(api_key: String) -> Self {
        Self {
            client: reqwest::Client::new(),
            api_key,
        }
    }
}

#[async_trait]
impl AIProvider for AnthropicProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        // Convert messages to Anthropic format
        let system_message = request.messages.iter()
            .find(|m| m.role == "system")
            .map(|m| m.content.clone());

        let messages: Vec<_> = request.messages.iter()
            .filter(|m| m.role != "system")
            .collect();

        let mut payload = json!({
            "model": request.model,
            "max_tokens": request.max_tokens.unwrap_or(1000),
            "messages": messages
        });

        if let Some(system) = system_message {
            payload["system"] = json!(system);
        }

        if let Some(temp) = request.temperature {
            payload["temperature"] = json!(temp);
        }

        let response = self.client
            .post("https://api.anthropic.com/v1/messages")
            .header("x-api-key", &self.api_key)
            .header("anthropic-version", "2023-06-01")
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        // Parse Anthropic response format
        let response_body: serde_json::Value = response.json().await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        let content = response_body["content"][0]["text"]
            .as_str()
            .unwrap_or("")
            .to_string();

        Ok(AIResponse {
            content,
            finish_reason: response_body["stop_reason"]
                .as_str()
                .map(|s| s.to_string()),
            usage: Some(Usage {
                prompt_tokens: response_body["usage"]["input_tokens"].as_u64().unwrap_or(0) as u32,
                completion_tokens: response_body["usage"]["output_tokens"].as_u64().unwrap_or(0) as u32,
                total_tokens: 0, // Calculate from above
            }),
        })
    }

    async fn stream_completion(&self, request: AIRequest) -> Result<tokio::sync::mpsc::Receiver<String>, AIError> {
        // Similar implementation with Anthropic streaming format
        todo!("Implement Anthropic streaming")
    }

    fn provider_name(&self) -> &'static str {
        "anthropic"
    }
}
```

### Provider Manager

```rust
// src-tauri/src/ai/manager.rs
use super::{AIProvider, OpenAIProvider, AnthropicProvider};
use std::collections::HashMap;

pub struct AIManager {
    providers: HashMap<String, Box<dyn AIProvider + Send + Sync>>,
}

impl AIManager {
    pub fn new() -> Self {
        Self {
            providers: HashMap::new(),
        }
    }

    pub fn add_provider(&mut self, name: String, provider: Box<dyn AIProvider + Send + Sync>) {
        self.providers.insert(name, provider);
    }

    pub fn get_provider(&self, name: &str) -> Option<&(dyn AIProvider + Send + Sync)> {
        self.providers.get(name).map(|p| p.as_ref())
    }

    pub async fn initialize_from_config(&mut self, config: &AIConfig) -> Result<(), String> {
        if let Some(openai_key) = &config.openai_api_key {
            self.add_provider(
                "openai".to_string(),
                Box::new(OpenAIProvider::new(openai_key.clone()))
            );
        }

        if let Some(anthropic_key) = &config.anthropic_api_key {
            self.add_provider(
                "anthropic".to_string(),
                Box::new(AnthropicProvider::new(anthropic_key.clone()))
            );
        }

        Ok(())
    }
}

#[derive(serde::Deserialize)]
pub struct AIConfig {
    pub openai_api_key: Option<String>,
    pub anthropic_api_key: Option<String>,
    pub default_provider: String,
    pub default_model: String,
}
```

### Tauri Commands

```rust
// src-tauri/src/ai/commands.rs
use tauri::{command, State, Window};
use super::{AIManager, AIRequest, AIResponse};

#[command]
pub async fn call_ai_provider(
    provider: String,
    request: AIRequest,
    window: Window,
    ai_manager: State<'_, tokio::sync::Mutex<AIManager>>
) -> Result<AIResponse, String> {
    let manager = ai_manager.lock().await;

    let provider_impl = manager.get_provider(&provider)
        .ok_or_else(|| format!("Provider '{}' not found", provider))?;

    if request.stream {
        // Handle streaming
        let mut stream = provider_impl.stream_completion(request).await
            .map_err(|e| e.to_string())?;

        tokio::spawn(async move {
            while let Some(chunk) = stream.recv().await {
                window.emit("ai_stream_chunk", chunk).ok();
            }
            window.emit("ai_stream_end", ()).ok();
        });

        Ok(AIResponse {
            content: "".to_string(),
            finish_reason: None,
            usage: None,
        })
    } else {
        provider_impl.chat_completion(request).await
            .map_err(|e| e.to_string())
    }
}

#[command]
pub async fn set_api_key(
    provider: String,
    api_key: String,
    ai_manager: State<'_, tokio::sync::Mutex<AIManager>>
) -> Result<(), String> {
    // Store encrypted API key in SQLite
    // Reinitialize provider with new key
    Ok(())
}
```

## Local Inference with Burn Framework

### Burn Integration

```rust
// src-tauri/src/ai/local.rs
use burn::prelude::*;
use burn_ndarray::{NdArray, NdArrayDevice};

pub struct LocalAIProvider {
    device: NdArrayDevice,
    // Model would be loaded here
}

impl LocalAIProvider {
    pub fn new() -> Result<Self, String> {
        let device = NdArrayDevice::default();

        Ok(Self {
            device,
        })
    }

    pub async fn load_model(&mut self, model_path: &str) -> Result<(), String> {
        // Load ONNX model or Burn model
        // This would use burn-import for ONNX models
        todo!("Implement model loading")
    }
}

#[async_trait]
impl AIProvider for LocalAIProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        // Tokenize input
        // Run inference
        // Decode output
        todo!("Implement local inference")
    }

    async fn stream_completion(&self, request: AIRequest) -> Result<tokio::sync::mpsc::Receiver<String>, AIError> {
        // Implement streaming inference with token-by-token generation
        todo!("Implement local streaming")
    }

    fn provider_name(&self) -> &'static str {
        "local"
    }
}
```

### Ollama Integration (Alternative to Burn)

```rust
// src-tauri/src/ai/ollama.rs
pub struct OllamaProvider {
    client: reqwest::Client,
    base_url: String,
}

impl OllamaProvider {
    pub fn new(base_url: Option<String>) -> Self {
        Self {
            client: reqwest::Client::new(),
            base_url: base_url.unwrap_or_else(|| "http://localhost:11434".to_string()),
        }
    }

    pub async fn list_models(&self) -> Result<Vec<String>, String> {
        let response = self.client
            .get(&format!("{}/api/tags", self.base_url))
            .send()
            .await
            .map_err(|e| e.to_string())?;

        let models: serde_json::Value = response.json().await
            .map_err(|e| e.to_string())?;

        Ok(models["models"]
            .as_array()
            .unwrap_or(&vec![])
            .iter()
            .filter_map(|m| m["name"].as_str())
            .map(|s| s.to_string())
            .collect())
    }
}

#[async_trait]
impl AIProvider for OllamaProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        let payload = json!({
            "model": request.model,
            "messages": request.messages,
            "stream": false
        });

        let response = self.client
            .post(&format!("{}/api/chat", self.base_url))
            .json(&payload)
            .send()
            .await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        let response_body: serde_json::Value = response.json().await
            .map_err(|e| AIError::NetworkError(e.to_string()))?;

        Ok(AIResponse {
            content: response_body["message"]["content"]
                .as_str()
                .unwrap_or("")
                .to_string(),
            finish_reason: Some("stop".to_string()),
            usage: None,
        })
    }

    // ... streaming implementation
}
```

## Frontend Integration

### JavaScript AI Client

```javascript
// src/js/ai.js
class AIClient {
    constructor() {
        this.activeStreams = new Map();
    }

    async callProvider(provider, request) {
        const { invoke } = window.__TAURI__.tauri;

        if (request.stream) {
            return this.streamCompletion(provider, request);
        } else {
            return await invoke('call_ai_provider', { provider, request });
        }
    }

    async streamCompletion(provider, request) {
        const { invoke, event } = window.__TAURI__;
        const streamId = Math.random().toString(36);

        const unlisten = await event.listen('ai_stream_chunk', (event) => {
            const callback = this.activeStreams.get(streamId);
            if (callback) {
                callback(event.payload);
            }
        });

        const endUnlisten = await event.listen('ai_stream_end', () => {
            this.activeStreams.delete(streamId);
            unlisten();
            endUnlisten();
        });

        await invoke('call_ai_provider', { provider, request });

        return {
            onChunk: (callback) => {
                this.activeStreams.set(streamId, callback);
            },
            stop: () => {
                this.activeStreams.delete(streamId);
                unlisten();
                endUnlisten();
            }
        };
    }

    async setApiKey(provider, apiKey) {
        const { invoke } = window.__TAURI__.tauri;
        return await invoke('set_api_key', { provider, apiKey });
    }
}

// Usage in block execution
async function executeAIBlock(block) {
    const aiClient = new AIClient();

    const request = {
        model: block.config.model,
        messages: [
            { role: 'user', content: block.config.prompt }
        ],
        stream: true,
        temperature: block.config.temperature
    };

    const stream = await aiClient.callProvider(block.config.provider, request);

    let fullResponse = '';
    stream.onChunk((chunk) => {
        fullResponse += chunk;
        updateBlockOutput(block.id, fullResponse);
    });
}
```

## Benefits of This Approach

1. **Direct API Access**: No proxy servers or complex authentication flows
2. **Local Storage**: API keys stored securely on device
3. **Provider Flexibility**: Easy to add new AI providers
4. **Local Inference Support**: Option for completely offline operation
5. **Streaming Support**: Real-time response updates
6. **Simple Configuration**: User manages their own API keys

## Recommended Implementation Order

1. **Phase 1**: OpenAI provider with basic completion
2. **Phase 2**: Add streaming support
3. **Phase 3**: Anthropic provider
4. **Phase 4**: API key management and encryption
5. **Phase 5**: Ollama integration for local models
6. **Phase 6**: Burn framework integration (if needed)

This design provides a solid foundation for AI integration while maintaining the simplicity and offline-capable nature of the desktop application.