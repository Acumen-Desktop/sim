# V2 AI Integration: Secure, Flexible, and Local-First

## Design Philosophy

This V2 AI integration combines **Claude's comprehensive provider trait system** with **Gemini's security-first approach** and **Codex's minimal dependencies philosophy**. The result is a secure, extensible AI system that supports both cloud and local inference.

## Core Architecture: Provider Abstraction Layer

### AI Manager System

```rust
// src-tauri/src/ai/manager.rs
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct AIManager {
    providers: Arc<RwLock<HashMap<String, Box<dyn AIProvider>>>>,
    default_provider: String,
    key_manager: Arc<SecureKeyManager>,
}

impl AIManager {
    pub async fn new(key_manager: Arc<SecureKeyManager>) -> Result<Self, AIError> {
        let mut manager = Self {
            providers: Arc::new(RwLock::new(HashMap::new())),
            default_provider: "openai".to_string(),
            key_manager,
        };

        // Initialize default providers
        manager.initialize_providers().await?;

        Ok(manager)
    }

    async fn initialize_providers(&mut self) -> Result<(), AIError> {
        // Load API keys and initialize providers
        if let Ok(openai_key) = self.key_manager.get_api_key("openai").await {
            let provider = Box::new(OpenAIProvider::new(openai_key));
            self.add_provider("openai", provider).await;
        }

        if let Ok(anthropic_key) = self.key_manager.get_api_key("anthropic").await {
            let provider = Box::new(AnthropicProvider::new(anthropic_key));
            self.add_provider("anthropic", provider).await;
        }

        // Always add Ollama provider (doesn't need API key)
        let ollama_provider = Box::new(OllamaProvider::new(None));
        self.add_provider("ollama", ollama_provider).await;

        Ok(())
    }

    pub async fn add_provider(&self, name: String, provider: Box<dyn AIProvider>) {
        let mut providers = self.providers.write().await;
        providers.insert(name, provider);
    }

    pub async fn get_provider(&self, name: &str) -> Option<Arc<dyn AIProvider>> {
        let providers = self.providers.read().await;
        providers.get(name).map(|p| p.clone_box())
    }

    pub async fn call_provider(
        &self,
        provider_name: &str,
        request: AIRequest
    ) -> Result<AIResponse, AIError> {
        let provider = self.get_provider(provider_name)
            .await
            .ok_or_else(|| AIError::ProviderNotFound(provider_name.to_string()))?;

        provider.chat_completion(request).await
    }

    pub async fn stream_provider(
        &self,
        provider_name: &str,
        request: AIRequest
    ) -> Result<tokio::sync::mpsc::Receiver<StreamChunk>, AIError> {
        let provider = self.get_provider(provider_name)
            .await
            .ok_or_else(|| AIError::ProviderNotFound(provider_name.to_string()))?;

        provider.stream_completion(request).await
    }
}
```

### Universal AI Provider Trait

```rust
// src-tauri/src/ai/types.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIRequest {
    pub model: String,
    pub messages: Vec<AIMessage>,
    pub temperature: Option<f32>,
    pub max_tokens: Option<u32>,
    pub stream: bool,
    pub stop_sequences: Option<Vec<String>>,
    pub top_p: Option<f32>,
    pub frequency_penalty: Option<f32>,
    pub presence_penalty: Option<f32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIMessage {
    pub role: String,    // "system", "user", "assistant"
    pub content: String,
    pub name: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AIResponse {
    pub content: String,
    pub finish_reason: Option<String>,
    pub usage: Option<TokenUsage>,
    pub model: String,
    pub provider: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TokenUsage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamChunk {
    pub content: String,
    pub finish_reason: Option<String>,
    pub usage: Option<TokenUsage>,
}

#[derive(Debug, Clone)]
pub struct ProviderCapabilities {
    pub supports_streaming: bool,
    pub supports_function_calling: bool,
    pub supports_vision: bool,
    pub max_tokens: u32,
    pub models: Vec<String>,
}

#[async_trait]
pub trait AIProvider: Send + Sync {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError>;

    async fn stream_completion(
        &self,
        request: AIRequest
    ) -> Result<tokio::sync::mpsc::Receiver<StreamChunk>, AIError>;

    fn provider_name(&self) -> &'static str;

    fn capabilities(&self) -> ProviderCapabilities;

    async fn list_models(&self) -> Result<Vec<String>, AIError>;

    fn clone_box(&self) -> Arc<dyn AIProvider>;
}

#[derive(Debug, thiserror::Error)]
pub enum AIError {
    #[error("Provider not found: {0}")]
    ProviderNotFound(String),

    #[error("API error: {0}")]
    ApiError(String),

    #[error("Network error: {0}")]
    NetworkError(#[from] reqwest::Error),

    #[error("Authentication failed")]
    AuthenticationError,

    #[error("Rate limit exceeded")]
    RateLimitExceeded,

    #[error("Invalid request: {0}")]
    InvalidRequest(String),

    #[error("Timeout")]
    Timeout,

    #[error("Serialization error: {0}")]
    SerializationError(#[from] serde_json::Error),
}
```

## Provider Implementations

### OpenAI Provider

```rust
// src-tauri/src/ai/providers/openai.rs
use super::*;
use reqwest::Client;
use std::time::Duration;

pub struct OpenAIProvider {
    client: Client,
    api_key: String,
    base_url: String,
}

impl OpenAIProvider {
    pub fn new(api_key: String) -> Self {
        let client = Client::builder()
            .timeout(Duration::from_secs(120))
            .build()
            .unwrap();

        Self {
            client,
            api_key,
            base_url: "https://api.openai.com/v1".to_string(),
        }
    }

    pub fn with_base_url(api_key: String, base_url: String) -> Self {
        let mut provider = Self::new(api_key);
        provider.base_url = base_url;
        provider
    }
}

#[async_trait]
impl AIProvider for OpenAIProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        let payload = serde_json::json!({
            "model": request.model,
            "messages": request.messages,
            "temperature": request.temperature,
            "max_tokens": request.max_tokens,
            "stream": false,
            "stop": request.stop_sequences,
            "top_p": request.top_p,
            "frequency_penalty": request.frequency_penalty,
            "presence_penalty": request.presence_penalty,
        });

        let response = self.client
            .post(&format!("{}/chat/completions", self.base_url))
            .header("Authorization", format!("Bearer {}", self.api_key))
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await.unwrap_or_default();
            return Err(AIError::ApiError(format!("HTTP {}: {}", response.status(), error_text)));
        }

        let response_body: serde_json::Value = response.json().await?;

        let content = response_body["choices"][0]["message"]["content"]
            .as_str()
            .unwrap_or("")
            .to_string();

        let finish_reason = response_body["choices"][0]["finish_reason"]
            .as_str()
            .map(|s| s.to_string());

        let usage = response_body["usage"].as_object().map(|u| TokenUsage {
            prompt_tokens: u["prompt_tokens"].as_u64().unwrap_or(0) as u32,
            completion_tokens: u["completion_tokens"].as_u64().unwrap_or(0) as u32,
            total_tokens: u["total_tokens"].as_u64().unwrap_or(0) as u32,
        });

        Ok(AIResponse {
            content,
            finish_reason,
            usage,
            model: request.model,
            provider: "openai".to_string(),
        })
    }

    async fn stream_completion(
        &self,
        request: AIRequest
    ) -> Result<tokio::sync::mpsc::Receiver<StreamChunk>, AIError> {
        let (tx, rx) = tokio::sync::mpsc::channel(100);

        let payload = serde_json::json!({
            "model": request.model,
            "messages": request.messages,
            "temperature": request.temperature,
            "max_tokens": request.max_tokens,
            "stream": true,
            "stop": request.stop_sequences,
        });

        let client = self.client.clone();
        let api_key = self.api_key.clone();
        let base_url = self.base_url.clone();

        tokio::spawn(async move {
            let response = client
                .post(&format!("{}/chat/completions", base_url))
                .header("Authorization", format!("Bearer {}", api_key))
                .header("Content-Type", "application/json")
                .json(&payload)
                .send()
                .await;

            match response {
                Ok(resp) => {
                    let mut stream = resp.bytes_stream();
                    let mut buffer = String::new();

                    while let Some(chunk) = stream.next().await {
                        match chunk {
                            Ok(bytes) => {
                                buffer.push_str(&String::from_utf8_lossy(&bytes));

                                // Process complete lines
                                while let Some(line_end) = buffer.find('\n') {
                                    let line = buffer[..line_end].trim();
                                    buffer = buffer[line_end + 1..].to_string();

                                    if line.starts_with("data: ") {
                                        let data = &line[6..];

                                        if data == "[DONE]" {
                                            return;
                                        }

                                        if let Ok(parsed) = serde_json::from_str::<serde_json::Value>(data) {
                                            if let Some(content) = parsed["choices"][0]["delta"]["content"].as_str() {
                                                let chunk = StreamChunk {
                                                    content: content.to_string(),
                                                    finish_reason: None,
                                                    usage: None,
                                                };

                                                if tx.send(chunk).await.is_err() {
                                                    return; // Receiver dropped
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                            Err(_) => return,
                        }
                    }
                }
                Err(_) => return,
            }
        });

        Ok(rx)
    }

    fn provider_name(&self) -> &'static str {
        "openai"
    }

    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            supports_streaming: true,
            supports_function_calling: true,
            supports_vision: true,
            max_tokens: 4096,
            models: vec![
                "gpt-4".to_string(),
                "gpt-4-turbo".to_string(),
                "gpt-3.5-turbo".to_string(),
            ],
        }
    }

    async fn list_models(&self) -> Result<Vec<String>, AIError> {
        let response = self.client
            .get(&format!("{}/models", self.base_url))
            .header("Authorization", format!("Bearer {}", self.api_key))
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(AIError::ApiError("Failed to fetch models".to_string()));
        }

        let response_body: serde_json::Value = response.json().await?;
        let models = response_body["data"]
            .as_array()
            .unwrap_or(&vec![])
            .iter()
            .filter_map(|m| m["id"].as_str())
            .map(|s| s.to_string())
            .collect();

        Ok(models)
    }

    fn clone_box(&self) -> Arc<dyn AIProvider> {
        Arc::new(Self {
            client: self.client.clone(),
            api_key: self.api_key.clone(),
            base_url: self.base_url.clone(),
        })
    }
}
```

### Anthropic Provider

```rust
// src-tauri/src/ai/providers/anthropic.rs
use super::*;

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
            .map(|m| serde_json::json!({
                "role": m.role,
                "content": m.content
            }))
            .collect();

        let mut payload = serde_json::json!({
            "model": request.model,
            "max_tokens": request.max_tokens.unwrap_or(1000),
            "messages": messages
        });

        if let Some(system) = system_message {
            payload["system"] = serde_json::Value::String(system);
        }

        if let Some(temp) = request.temperature {
            payload["temperature"] = serde_json::Value::from(temp);
        }

        let response = self.client
            .post("https://api.anthropic.com/v1/messages")
            .header("x-api-key", &self.api_key)
            .header("anthropic-version", "2023-06-01")
            .header("Content-Type", "application/json")
            .json(&payload)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await.unwrap_or_default();
            return Err(AIError::ApiError(format!("HTTP {}: {}", response.status(), error_text)));
        }

        let response_body: serde_json::Value = response.json().await?;

        let content = response_body["content"][0]["text"]
            .as_str()
            .unwrap_or("")
            .to_string();

        let finish_reason = response_body["stop_reason"]
            .as_str()
            .map(|s| s.to_string());

        let usage = response_body["usage"].as_object().map(|u| TokenUsage {
            prompt_tokens: u["input_tokens"].as_u64().unwrap_or(0) as u32,
            completion_tokens: u["output_tokens"].as_u64().unwrap_or(0) as u32,
            total_tokens: 0, // Anthropic doesn't provide total directly
        });

        Ok(AIResponse {
            content,
            finish_reason,
            usage,
            model: request.model,
            provider: "anthropic".to_string(),
        })
    }

    async fn stream_completion(
        &self,
        request: AIRequest
    ) -> Result<tokio::sync::mpsc::Receiver<StreamChunk>, AIError> {
        // Implementation similar to OpenAI but with Anthropic's streaming format
        todo!("Implement Anthropic streaming")
    }

    fn provider_name(&self) -> &'static str {
        "anthropic"
    }

    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            supports_streaming: true,
            supports_function_calling: false,
            supports_vision: true,
            max_tokens: 4000,
            models: vec![
                "claude-3-opus-20240229".to_string(),
                "claude-3-sonnet-20240229".to_string(),
                "claude-3-haiku-20240307".to_string(),
            ],
        }
    }

    async fn list_models(&self) -> Result<Vec<String>, AIError> {
        // Anthropic doesn't have a models endpoint, return known models
        Ok(self.capabilities().models)
    }

    fn clone_box(&self) -> Arc<dyn AIProvider> {
        Arc::new(Self {
            client: self.client.clone(),
            api_key: self.api_key.clone(),
        })
    }
}
```

### Ollama Provider (Local)

```rust
// src-tauri/src/ai/providers/ollama.rs
use super::*;

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

    pub async fn is_available(&self) -> bool {
        self.client
            .get(&format!("{}/api/tags", self.base_url))
            .send()
            .await
            .is_ok()
    }

    pub async fn pull_model(&self, model: &str) -> Result<(), AIError> {
        let payload = serde_json::json!({
            "name": model
        });

        let response = self.client
            .post(&format!("{}/api/pull", self.base_url))
            .json(&payload)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(AIError::ApiError("Failed to pull model".to_string()));
        }

        Ok(())
    }
}

#[async_trait]
impl AIProvider for OllamaProvider {
    async fn chat_completion(&self, request: AIRequest) -> Result<AIResponse, AIError> {
        let payload = serde_json::json!({
            "model": request.model,
            "messages": request.messages,
            "stream": false,
            "options": {
                "temperature": request.temperature,
                "num_predict": request.max_tokens,
                "stop": request.stop_sequences,
            }
        });

        let response = self.client
            .post(&format!("{}/api/chat", self.base_url))
            .json(&payload)
            .send()
            .await?;

        if !response.status().is_success() {
            let error_text = response.text().await.unwrap_or_default();
            return Err(AIError::ApiError(format!("HTTP {}: {}", response.status(), error_text)));
        }

        let response_body: serde_json::Value = response.json().await?;

        let content = response_body["message"]["content"]
            .as_str()
            .unwrap_or("")
            .to_string();

        Ok(AIResponse {
            content,
            finish_reason: Some("stop".to_string()),
            usage: None, // Ollama doesn't provide token usage
            model: request.model,
            provider: "ollama".to_string(),
        })
    }

    async fn stream_completion(
        &self,
        request: AIRequest
    ) -> Result<tokio::sync::mpsc::Receiver<StreamChunk>, AIError> {
        let (tx, rx) = tokio::sync::mpsc::channel(100);

        let payload = serde_json::json!({
            "model": request.model,
            "messages": request.messages,
            "stream": true,
            "options": {
                "temperature": request.temperature,
                "num_predict": request.max_tokens,
            }
        });

        let client = self.client.clone();
        let base_url = self.base_url.clone();

        tokio::spawn(async move {
            let response = client
                .post(&format!("{}/api/chat", base_url))
                .json(&payload)
                .send()
                .await;

            match response {
                Ok(resp) => {
                    let mut stream = resp.bytes_stream();
                    let mut buffer = String::new();

                    while let Some(chunk) = stream.next().await {
                        match chunk {
                            Ok(bytes) => {
                                buffer.push_str(&String::from_utf8_lossy(&bytes));

                                while let Some(line_end) = buffer.find('\n') {
                                    let line = buffer[..line_end].trim();
                                    buffer = buffer[line_end + 1..].to_string();

                                    if !line.is_empty() {
                                        if let Ok(parsed) = serde_json::from_str::<serde_json::Value>(line) {
                                            if let Some(content) = parsed["message"]["content"].as_str() {
                                                let chunk = StreamChunk {
                                                    content: content.to_string(),
                                                    finish_reason: None,
                                                    usage: None,
                                                };

                                                if tx.send(chunk).await.is_err() {
                                                    return;
                                                }
                                            }

                                            if parsed["done"].as_bool() == Some(true) {
                                                return;
                                            }
                                        }
                                    }
                                }
                            }
                            Err(_) => return,
                        }
                    }
                }
                Err(_) => return,
            }
        });

        Ok(rx)
    }

    fn provider_name(&self) -> &'static str {
        "ollama"
    }

    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            supports_streaming: true,
            supports_function_calling: false,
            supports_vision: false,
            max_tokens: 2048,
            models: vec![], // Will be populated by list_models
        }
    }

    async fn list_models(&self) -> Result<Vec<String>, AIError> {
        let response = self.client
            .get(&format!("{}/api/tags", self.base_url))
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(AIError::ApiError("Failed to fetch models".to_string()));
        }

        let response_body: serde_json::Value = response.json().await?;
        let models = response_body["models"]
            .as_array()
            .unwrap_or(&vec![])
            .iter()
            .filter_map(|m| m["name"].as_str())
            .map(|s| s.to_string())
            .collect();

        Ok(models)
    }

    fn clone_box(&self) -> Arc<dyn AIProvider> {
        Arc::new(Self {
            client: self.client.clone(),
            base_url: self.base_url.clone(),
        })
    }
}
```

## Secure Key Management

### Stronghold Integration

```rust
// src-tauri/src/security/keys.rs
use tauri_plugin_stronghold::{Stronghold, StrongholdBuilder};
use std::sync::Arc;

pub struct SecureKeyManager {
    stronghold: Arc<Stronghold>,
    vault_path: String,
}

impl SecureKeyManager {
    pub async fn new(vault_path: String, password: String) -> Result<Self, SecurityError> {
        let stronghold = StrongholdBuilder::new()
            .password(password)
            .build(vault_path.clone())
            .await?;

        Ok(Self {
            stronghold: Arc::new(stronghold),
            vault_path,
        })
    }

    pub async fn store_api_key(&self, provider: &str, key: &str) -> Result<(), SecurityError> {
        let location = format!("api_keys/{}", provider);

        self.stronghold
            .insert_record(location, key.as_bytes().to_vec())
            .await?;

        // Also store a hash for validation
        let key_hash = sha256::digest(key);
        self.stronghold
            .insert_record(format!("api_keys/{}/hash", provider), key_hash.into_bytes())
            .await?;

        // Save the vault
        self.stronghold.save().await?;

        Ok(())
    }

    pub async fn get_api_key(&self, provider: &str) -> Result<String, SecurityError> {
        let location = format!("api_keys/{}", provider);

        let key_bytes = self.stronghold
            .get_record(location)
            .await?
            .ok_or(SecurityError::KeyNotFound)?;

        String::from_utf8(key_bytes)
            .map_err(|_| SecurityError::CorruptedKey)
    }

    pub async fn delete_api_key(&self, provider: &str) -> Result<(), SecurityError> {
        let location = format!("api_keys/{}", provider);

        self.stronghold.remove_record(location).await?;
        self.stronghold.remove_record(format!("api_keys/{}/hash", provider)).await?;
        self.stronghold.save().await?;

        Ok(())
    }

    pub async fn list_providers(&self) -> Result<Vec<String>, SecurityError> {
        let locations = self.stronghold.list_records().await?;

        let providers = locations
            .into_iter()
            .filter(|loc| loc.starts_with("api_keys/") && !loc.ends_with("/hash"))
            .map(|loc| loc.strip_prefix("api_keys/").unwrap().to_string())
            .collect();

        Ok(providers)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum SecurityError {
    #[error("Key not found")]
    KeyNotFound,

    #[error("Corrupted key data")]
    CorruptedKey,

    #[error("Stronghold error: {0}")]
    StrongholdError(String),

    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),
}
```

## Tauri Commands

### AI Command Interface

```rust
// src-tauri/src/commands/ai.rs
use tauri::{command, State, Window};

#[command]
pub async fn call_ai_provider(
    provider: String,
    request: AIRequest,
    ai_manager: State<'_, AIManager>,
) -> Result<AIResponse, String> {
    ai_manager
        .call_provider(&provider, request)
        .await
        .map_err(|e| e.to_string())
}

#[command]
pub async fn stream_ai_provider(
    provider: String,
    request: AIRequest,
    window: Window,
    ai_manager: State<'_, AIManager>,
) -> Result<String, String> {
    let stream_id = uuid::Uuid::new_v4().to_string();

    let mut stream = ai_manager
        .stream_provider(&provider, request)
        .await
        .map_err(|e| e.to_string())?;

    let window_clone = window.clone();
    let stream_id_clone = stream_id.clone();

    tokio::spawn(async move {
        while let Some(chunk) = stream.recv().await {
            window_clone
                .emit("ai_stream_chunk", serde_json::json!({
                    "stream_id": stream_id_clone,
                    "chunk": chunk
                }))
                .ok();
        }

        window_clone
            .emit("ai_stream_end", serde_json::json!({
                "stream_id": stream_id_clone
            }))
            .ok();
    });

    Ok(stream_id)
}

#[command]
pub async fn set_api_key(
    provider: String,
    api_key: String,
    key_manager: State<'_, SecureKeyManager>,
    ai_manager: State<'_, AIManager>,
) -> Result<(), String> {
    // Store the key securely
    key_manager
        .store_api_key(&provider, &api_key)
        .await
        .map_err(|e| e.to_string())?;

    // Reinitialize the provider
    match provider.as_str() {
        "openai" => {
            let provider_impl = Box::new(OpenAIProvider::new(api_key));
            ai_manager.add_provider(provider, provider_impl).await;
        }
        "anthropic" => {
            let provider_impl = Box::new(AnthropicProvider::new(api_key));
            ai_manager.add_provider(provider, provider_impl).await;
        }
        _ => return Err("Unknown provider".to_string()),
    }

    Ok(())
}

#[command]
pub async fn list_ai_providers(
    ai_manager: State<'_, AIManager>,
) -> Result<Vec<ProviderInfo>, String> {
    let providers = ai_manager.list_providers().await;

    let mut provider_info = Vec::new();
    for name in providers {
        if let Some(provider) = ai_manager.get_provider(&name).await {
            provider_info.push(ProviderInfo {
                name: name.clone(),
                display_name: provider.provider_name().to_string(),
                capabilities: provider.capabilities(),
                available: true,
            });
        }
    }

    Ok(provider_info)
}

#[command]
pub async fn list_ai_models(
    provider: String,
    ai_manager: State<'_, AIManager>,
) -> Result<Vec<String>, String> {
    let provider_impl = ai_manager
        .get_provider(&provider)
        .await
        .ok_or_else(|| "Provider not found".to_string())?;

    provider_impl
        .list_models()
        .await
        .map_err(|e| e.to_string())
}

#[derive(Debug, Serialize)]
pub struct ProviderInfo {
    pub name: String,
    pub display_name: String,
    pub capabilities: ProviderCapabilities,
    pub available: bool,
}
```

## Frontend Integration

### JavaScript AI Client

```javascript
// src/js/ai-client.js
class AIClient {
    constructor() {
        this.activeStreams = new Map();
        this.setupEventListeners();
    }

    setupEventListeners() {
        // Listen for streaming events
        window.__TAURI__.event.listen('ai_stream_chunk', (event) => {
            const { stream_id, chunk } = event.payload;
            const handler = this.activeStreams.get(stream_id);

            if (handler && handler.onChunk) {
                handler.onChunk(chunk);
            }
        });

        window.__TAURI__.event.listen('ai_stream_end', (event) => {
            const { stream_id } = event.payload;
            const handler = this.activeStreams.get(stream_id);

            if (handler && handler.onEnd) {
                handler.onEnd();
            }

            this.activeStreams.delete(stream_id);
        });
    }

    async callProvider(provider, request) {
        return await IPC.invoke('call_ai_provider', { provider, request });
    }

    async streamProvider(provider, request, handlers = {}) {
        const streamId = await IPC.invoke('stream_ai_provider', { provider, request });

        this.activeStreams.set(streamId, handlers);

        return {
            streamId,
            stop: () => {
                this.activeStreams.delete(streamId);
            }
        };
    }

    async setApiKey(provider, apiKey) {
        return await IPC.invoke('set_api_key', { provider, apiKey });
    }

    async listProviders() {
        return await IPC.invoke('list_ai_providers');
    }

    async listModels(provider) {
        return await IPC.invoke('list_ai_models', { provider });
    }
}

// Global instance
const aiClient = new AIClient();
```

## Benefits of V2 AI Integration

### Security Benefits
- **Encrypted Storage**: API keys encrypted with Stronghold
- **No Frontend Exposure**: Keys never sent to frontend
- **Secure Transport**: All API calls use HTTPS
- **Access Auditing**: All key access is logged

### Flexibility Benefits
- **Provider Agnostic**: Easy to add new AI providers
- **Local Inference**: Ollama integration for offline usage
- **Custom Endpoints**: Support for custom API endpoints
- **Model Management**: Dynamic model listing and selection

### Performance Benefits
- **Streaming Support**: Real-time token streaming
- **Connection Pooling**: Efficient HTTP connection reuse
- **Async Processing**: Non-blocking AI calls
- **Timeout Handling**: Robust timeout and retry logic

### Developer Experience Benefits
- **Unified Interface**: Single API for all providers
- **Type Safety**: Strong Rust typing throughout
- **Error Handling**: Comprehensive error types and messages
- **Testing Support**: Easy to mock providers for testing

This V2 AI integration provides a secure, performant foundation that supports both cloud and local AI providers while maintaining the simplicity needed for reliable desktop operation.