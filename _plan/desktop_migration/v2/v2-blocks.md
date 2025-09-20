# V2 Block System: Generic and Extensible Architecture

## Design Philosophy

This V2 block system adopts **Codex's generic JSON descriptor approach** with **Gemini's data-driven methodology** and **Claude's security-focused execution**. The result is an extensible, maintainable system that can grow beyond MVP without architectural changes.

## Core Architecture: Descriptor-Driven Blocks

### Why Generic Blocks?

**Extensibility:**
- Add new blocks without touching core canvas code
- Community-contributed blocks in the future
- Domain-specific block packs
- Easy A/B testing of block variations

**Maintainability:**
- Single rendering engine for all block types
- Consistent validation and error handling
- Unified configuration UI generation
- Simplified testing and debugging

**Consistency:**
- Uniform look and behavior across all blocks
- Consistent input/output interfaces
- Standardized configuration patterns
- Predictable execution semantics

## Block Descriptor Format

### JSON Schema Structure

```json
{
  "$schema": "https://sim.ai/schemas/block-v1.json",
  "id": "http.request",
  "version": "1.0.0",
  "label": "HTTP Request",
  "description": "Make HTTP requests to external APIs",
  "category": "network",
  "icon": "globe",

  "inputs": [
    {
      "id": "url",
      "label": "URL",
      "type": "string",
      "required": true,
      "description": "The URL to request",
      "validation": {
        "pattern": "^https?://.*",
        "message": "Must be a valid HTTP/HTTPS URL"
      }
    },
    {
      "id": "method",
      "label": "Method",
      "type": "enum",
      "required": false,
      "default": "GET",
      "options": [
        { "value": "GET", "label": "GET" },
        { "value": "POST", "label": "POST" },
        { "value": "PUT", "label": "PUT" },
        { "value": "DELETE", "label": "DELETE" }
      ]
    },
    {
      "id": "headers",
      "label": "Headers",
      "type": "json",
      "required": false,
      "description": "HTTP headers as JSON object",
      "default": "{}",
      "validation": {
        "schema": {
          "type": "object",
          "additionalProperties": { "type": "string" }
        }
      }
    },
    {
      "id": "body",
      "label": "Body",
      "type": "text",
      "required": false,
      "description": "Request body (for POST/PUT)",
      "condition": {
        "field": "method",
        "operator": "in",
        "value": ["POST", "PUT", "PATCH"]
      }
    }
  ],

  "outputs": [
    {
      "id": "status",
      "label": "Status Code",
      "type": "number",
      "description": "HTTP status code"
    },
    {
      "id": "headers",
      "label": "Response Headers",
      "type": "json",
      "description": "Response headers object"
    },
    {
      "id": "body",
      "label": "Response Body",
      "type": "any",
      "description": "Response body (auto-parsed if JSON)"
    }
  ],

  "config": {
    "timeout": {
      "type": "number",
      "label": "Timeout (ms)",
      "default": 30000,
      "min": 1000,
      "max": 300000
    },
    "follow_redirects": {
      "type": "boolean",
      "label": "Follow Redirects",
      "default": true
    }
  },

  "ui": {
    "width": 300,
    "height": 200,
    "color": "#4CAF50",
    "helpUrl": "https://docs.sim.ai/blocks/http-request"
  },

  "execution": {
    "handler": "network::http_request",
    "timeout": 60000,
    "retryable": true,
    "max_retries": 3
  }
}
```

### Type System

```typescript
type BlockInputType =
  | 'string'      // Single-line text
  | 'text'        // Multi-line text
  | 'number'      // Numeric input
  | 'boolean'     // Checkbox
  | 'enum'        // Dropdown selection
  | 'json'        // JSON object
  | 'array'       // Array of values
  | 'file'        // File selector
  | 'datetime'    // Date/time picker
  | 'template'    // Template string with variables
  | 'code'        // Code editor
  | 'any';        // Any type (from previous blocks)

type BlockOutputType = BlockInputType;

interface BlockDescriptor {
  id: string;
  version: string;
  label: string;
  description?: string;
  category: string;
  icon?: string;

  inputs: BlockInput[];
  outputs: BlockOutput[];
  config?: BlockConfig;
  ui?: BlockUI;
  execution: BlockExecution;
}
```

## Block Registry System

### Registry Implementation

```rust
// src-tauri/src/blocks/registry.rs
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BlockRegistry {
    blocks: HashMap<String, BlockDescriptor>,
    categories: HashMap<String, CategoryInfo>,
    handlers: HashMap<String, Box<dyn BlockHandler>>,
}

impl BlockRegistry {
    pub fn new() -> Self {
        Self {
            blocks: HashMap::new(),
            categories: HashMap::new(),
            handlers: HashMap::new(),
        }
    }

    pub fn load_from_directory(&mut self, dir: &Path) -> Result<(), RegistryError> {
        for entry in std::fs::read_dir(dir)? {
            let path = entry?.path();
            if path.extension() == Some("json".as_ref()) {
                let descriptor = self.load_descriptor(&path)?;
                self.register_block(descriptor)?;
            }
        }
        Ok(())
    }

    pub fn register_block(&mut self, descriptor: BlockDescriptor) -> Result<(), RegistryError> {
        self.validate_descriptor(&descriptor)?;
        self.blocks.insert(descriptor.id.clone(), descriptor);
        Ok(())
    }

    pub fn get_descriptor(&self, block_id: &str) -> Option<&BlockDescriptor> {
        self.blocks.get(block_id)
    }

    pub fn get_blocks_by_category(&self, category: &str) -> Vec<&BlockDescriptor> {
        self.blocks.values()
            .filter(|block| block.category == category)
            .collect()
    }

    pub fn validate_descriptor(&self, descriptor: &BlockDescriptor) -> Result<(), RegistryError> {
        // Validate descriptor structure
        // Check for duplicate IDs
        // Validate handler exists
        // Validate input/output types
        Ok(())
    }
}
```

### Block Handler Trait

```rust
// src-tauri/src/blocks/handler.rs
use async_trait::async_trait;

#[async_trait]
pub trait BlockHandler: Send + Sync {
    async fn execute(&self, context: &ExecutionContext) -> Result<BlockOutput, BlockError>;

    fn descriptor(&self) -> &BlockDescriptor;

    async fn validate_inputs(&self, inputs: &BlockInputs) -> Result<(), ValidationError> {
        // Default validation based on descriptor
        let descriptor = self.descriptor();
        for input in &descriptor.inputs {
            if input.required && !inputs.contains_key(&input.id) {
                return Err(ValidationError::MissingRequired(input.id.clone()));
            }

            if let Some(value) = inputs.get(&input.id) {
                self.validate_input_value(input, value)?;
            }
        }
        Ok(())
    }

    fn validate_input_value(&self, input: &BlockInput, value: &serde_json::Value) -> Result<(), ValidationError> {
        // Type validation, pattern matching, etc.
        Ok(())
    }
}
```

## MVP Block Implementations

### Control Flow Blocks

```json
// blocks/control/manual.start.json
{
  "id": "manual.start",
  "label": "Manual Start",
  "description": "Manually triggered workflow entry point",
  "category": "control",
  "icon": "play",

  "inputs": [
    {
      "id": "trigger_data",
      "label": "Input Data",
      "type": "json",
      "required": false,
      "description": "Data to pass to the workflow",
      "default": "{}"
    }
  ],

  "outputs": [
    {
      "id": "data",
      "label": "Output Data",
      "type": "json",
      "description": "The input data passed through"
    }
  ],

  "ui": {
    "width": 200,
    "height": 100,
    "color": "#2E7D32"
  },

  "execution": {
    "handler": "control::manual_start",
    "timeout": 1000
  }
}
```

```json
// blocks/control/condition.if.json
{
  "id": "condition.if",
  "label": "If Condition",
  "description": "Conditional branching based on boolean expression",
  "category": "control",
  "icon": "fork",

  "inputs": [
    {
      "id": "condition",
      "label": "Condition",
      "type": "template",
      "required": true,
      "description": "Boolean expression to evaluate",
      "placeholder": "{{input.value}} > 100"
    },
    {
      "id": "input",
      "label": "Input Data",
      "type": "any",
      "required": true,
      "description": "Data to evaluate condition against"
    }
  ],

  "outputs": [
    {
      "id": "result",
      "label": "Result",
      "type": "boolean",
      "description": "Condition evaluation result"
    },
    {
      "id": "data",
      "label": "Pass-through Data",
      "type": "any",
      "description": "Input data passed through"
    }
  ],

  "ui": {
    "width": 250,
    "height": 150,
    "color": "#FF9800"
  },

  "execution": {
    "handler": "control::condition_if",
    "timeout": 5000
  }
}
```

### AI Blocks

```json
// blocks/ai/ai.chat.json
{
  "id": "ai.chat",
  "label": "AI Chat",
  "description": "Send prompt to AI provider and get response",
  "category": "ai",
  "icon": "brain",

  "inputs": [
    {
      "id": "prompt",
      "label": "Prompt",
      "type": "template",
      "required": true,
      "description": "The prompt to send to the AI",
      "placeholder": "Analyze this data: {{input.data}}"
    },
    {
      "id": "system_prompt",
      "label": "System Prompt",
      "type": "text",
      "required": false,
      "description": "System prompt to set AI behavior"
    },
    {
      "id": "input",
      "label": "Input Data",
      "type": "any",
      "required": false,
      "description": "Data to reference in prompt"
    }
  ],

  "outputs": [
    {
      "id": "response",
      "label": "AI Response",
      "type": "string",
      "description": "The AI's response text"
    },
    {
      "id": "usage",
      "label": "Token Usage",
      "type": "json",
      "description": "Token usage statistics"
    }
  ],

  "config": {
    "provider": {
      "type": "enum",
      "label": "AI Provider",
      "default": "openai",
      "options": [
        { "value": "openai", "label": "OpenAI" },
        { "value": "anthropic", "label": "Anthropic" },
        { "value": "ollama", "label": "Ollama (Local)" }
      ]
    },
    "model": {
      "type": "string",
      "label": "Model",
      "default": "gpt-4",
      "description": "AI model to use"
    },
    "temperature": {
      "type": "number",
      "label": "Temperature",
      "default": 0.7,
      "min": 0,
      "max": 2,
      "step": 0.1
    },
    "max_tokens": {
      "type": "number",
      "label": "Max Tokens",
      "default": 1000,
      "min": 1,
      "max": 4000
    },
    "stream": {
      "type": "boolean",
      "label": "Stream Response",
      "default": true
    }
  },

  "ui": {
    "width": 350,
    "height": 250,
    "color": "#9C27B0"
  },

  "execution": {
    "handler": "ai::chat_completion",
    "timeout": 120000,
    "retryable": true,
    "max_retries": 2
  }
}
```

## Handler Implementations

### HTTP Request Handler

```rust
// src-tauri/src/blocks/handlers/network.rs
use super::*;

pub struct HttpRequestHandler {
    descriptor: BlockDescriptor,
    client: reqwest::Client,
}

impl HttpRequestHandler {
    pub fn new() -> Self {
        let descriptor = BlockDescriptor::load("http.request").unwrap();
        let client = reqwest::Client::builder()
            .timeout(Duration::from_millis(30000))
            .build()
            .unwrap();

        Self { descriptor, client }
    }
}

#[async_trait]
impl BlockHandler for HttpRequestHandler {
    async fn execute(&self, context: &ExecutionContext) -> Result<BlockOutput, BlockError> {
        let inputs = &context.inputs;

        // Extract inputs with validation
        let url = inputs.get_string("url")?;
        let method = inputs.get_string("method").unwrap_or("GET".to_string());
        let headers = inputs.get_json("headers").unwrap_or_default();
        let body = inputs.get_string("body").ok();

        // Build request
        let mut request = self.client.request(
            method.parse().map_err(|_| BlockError::InvalidInput("Invalid HTTP method".into()))?,
            &url
        );

        // Add headers
        if let Some(headers_obj) = headers.as_object() {
            for (key, value) in headers_obj {
                if let Some(value_str) = value.as_str() {
                    request = request.header(key, value_str);
                }
            }
        }

        // Add body if present
        if let Some(body_str) = body {
            request = request.body(body_str);
        }

        // Execute request
        let response = request.send().await.map_err(|e| BlockError::ExecutionError(e.into()))?;

        let status = response.status().as_u16();
        let response_headers = response.headers().clone();
        let response_body = response.text().await.map_err(|e| BlockError::ExecutionError(e.into()))?;

        // Try to parse as JSON, fallback to text
        let parsed_body = serde_json::from_str::<serde_json::Value>(&response_body)
            .unwrap_or_else(|_| serde_json::Value::String(response_body));

        // Build output
        let mut output = BlockOutput::new();
        output.set("status", serde_json::Value::Number(status.into()));
        output.set("headers", serde_json::to_value(response_headers)?);
        output.set("body", parsed_body);

        Ok(output)
    }

    fn descriptor(&self) -> &BlockDescriptor {
        &self.descriptor
    }
}
```

### AI Chat Handler

```rust
// src-tauri/src/blocks/handlers/ai.rs
use super::*;
use crate::ai::AIManager;

pub struct AiChatHandler {
    descriptor: BlockDescriptor,
    ai_manager: Arc<AIManager>,
}

#[async_trait]
impl BlockHandler for AiChatHandler {
    async fn execute(&self, context: &ExecutionContext) -> Result<BlockOutput, BlockError> {
        let inputs = &context.inputs;
        let config = &context.config;

        // Extract inputs
        let prompt = inputs.get_template("prompt", &context.variables)?;
        let system_prompt = inputs.get_string("system_prompt").ok();

        // Extract config
        let provider = config.get_string("provider").unwrap_or("openai".to_string());
        let model = config.get_string("model").unwrap_or("gpt-4".to_string());
        let temperature = config.get_f64("temperature").unwrap_or(0.7);
        let max_tokens = config.get_u32("max_tokens").unwrap_or(1000);
        let stream = config.get_bool("stream").unwrap_or(true);

        // Build AI request
        let mut messages = Vec::new();

        if let Some(system) = system_prompt {
            messages.push(AIMessage::system(system));
        }

        messages.push(AIMessage::user(prompt));

        let ai_request = AIRequest {
            model,
            messages,
            temperature: Some(temperature as f32),
            max_tokens: Some(max_tokens),
            stream,
        };

        // Execute AI request
        let ai_provider = self.ai_manager.get_provider(&provider)
            .ok_or_else(|| BlockError::ConfigError(format!("Provider '{}' not found", provider)))?;

        if stream {
            // Handle streaming response
            let mut stream = ai_provider.stream_completion(ai_request).await?;
            let mut full_response = String::new();

            while let Some(chunk) = stream.recv().await {
                full_response.push_str(&chunk);

                // Emit streaming event
                context.emit_event("ai_stream_chunk", &chunk).await?;
            }

            let mut output = BlockOutput::new();
            output.set("response", serde_json::Value::String(full_response));
            output.set("usage", serde_json::Value::Null); // TODO: Add usage tracking

            Ok(output)
        } else {
            // Handle non-streaming response
            let response = ai_provider.chat_completion(ai_request).await?;

            let mut output = BlockOutput::new();
            output.set("response", serde_json::Value::String(response.content));
            output.set("usage", serde_json::to_value(response.usage)?);

            Ok(output)
        }
    }

    fn descriptor(&self) -> &BlockDescriptor {
        &self.descriptor
    }
}
```

## Frontend Integration

### Block Palette Generation

```javascript
// src/js/blocks.js
class BlockPalette {
    constructor(registry) {
        this.registry = registry;
        this.element = document.getElementById('block-palette');
        this.searchInput = document.getElementById('block-search');

        this.render();
        this.attachEventListeners();
    }

    async render() {
        const categories = await this.registry.getCategories();

        let html = '<div class="palette-search">';
        html += '<input type="text" id="block-search" placeholder="Search blocks...">';
        html += '</div>';

        for (const [categoryId, categoryInfo] of Object.entries(categories)) {
            html += `<div class="palette-category" data-category="${categoryId}">`;
            html += `<h3 class="category-title">${categoryInfo.label}</h3>`;
            html += '<div class="category-blocks">';

            const blocks = await this.registry.getBlocksByCategory(categoryId);
            for (const block of blocks) {
                html += this.renderBlockItem(block);
            }

            html += '</div></div>';
        }

        this.element.innerHTML = html;
    }

    renderBlockItem(descriptor) {
        return `
            <div class="palette-block"
                 data-block-type="${descriptor.id}"
                 draggable="true"
                 title="${descriptor.description}">
                <div class="block-icon" style="color: ${descriptor.ui.color}">
                    ${this.getIcon(descriptor.icon)}
                </div>
                <div class="block-label">${descriptor.label}</div>
            </div>
        `;
    }

    attachEventListeners() {
        // Search functionality
        this.searchInput.addEventListener('input', (e) => {
            this.filterBlocks(e.target.value);
        });

        // Drag and drop
        this.element.addEventListener('dragstart', (e) => {
            const blockType = e.target.dataset.blockType;
            if (blockType) {
                e.dataTransfer.setData('application/block-type', blockType);
            }
        });
    }
}
```

### Dynamic Configuration UI

```javascript
// src/js/block-config.js
class BlockConfigPanel {
    constructor() {
        this.element = document.getElementById('block-config');
        this.currentBlock = null;
    }

    async showConfig(blockId) {
        const block = canvas.getBlock(blockId);
        const descriptor = await BlockRegistry.getDescriptor(block.type);

        this.currentBlock = block;
        this.render(descriptor, block.data);
    }

    render(descriptor, currentData) {
        let html = `<h3>${descriptor.label} Configuration</h3>`;
        html += '<form class="block-config-form">';

        // Render inputs
        for (const input of descriptor.inputs) {
            html += this.renderInput(input, currentData[input.id]);
        }

        // Render config options
        if (descriptor.config) {
            html += '<h4>Settings</h4>';
            for (const [key, config] of Object.entries(descriptor.config)) {
                html += this.renderConfigOption(key, config, currentData.config?.[key]);
            }
        }

        html += '</form>';
        this.element.innerHTML = html;

        this.attachFormListeners();
    }

    renderInput(input, currentValue) {
        const value = currentValue || input.default || '';

        switch (input.type) {
            case 'string':
                return `
                    <div class="form-group">
                        <label>${input.label}</label>
                        <input type="text" name="${input.id}" value="${value}"
                               ${input.required ? 'required' : ''}>
                        ${input.description ? `<small>${input.description}</small>` : ''}
                    </div>
                `;

            case 'text':
                return `
                    <div class="form-group">
                        <label>${input.label}</label>
                        <textarea name="${input.id}" ${input.required ? 'required' : ''}>${value}</textarea>
                        ${input.description ? `<small>${input.description}</small>` : ''}
                    </div>
                `;

            case 'enum':
                let options = '';
                for (const option of input.options) {
                    const selected = option.value === value ? 'selected' : '';
                    options += `<option value="${option.value}" ${selected}>${option.label}</option>`;
                }
                return `
                    <div class="form-group">
                        <label>${input.label}</label>
                        <select name="${input.id}" ${input.required ? 'required' : ''}>
                            ${options}
                        </select>
                        ${input.description ? `<small>${input.description}</small>` : ''}
                    </div>
                `;

            case 'boolean':
                const checked = value ? 'checked' : '';
                return `
                    <div class="form-group">
                        <label class="checkbox-label">
                            <input type="checkbox" name="${input.id}" ${checked}>
                            ${input.label}
                        </label>
                        ${input.description ? `<small>${input.description}</small>` : ''}
                    </div>
                `;

            case 'template':
                return `
                    <div class="form-group">
                        <label>${input.label}</label>
                        <div class="template-editor">
                            <textarea name="${input.id}" class="template-input"
                                      placeholder="${input.placeholder || ''}"
                                      ${input.required ? 'required' : ''}>${value}</textarea>
                            <div class="template-help">
                                Use {{variable.name}} to reference values from previous blocks
                            </div>
                        </div>
                        ${input.description ? `<small>${input.description}</small>` : ''}
                    </div>
                `;

            default:
                return `<!-- Unsupported input type: ${input.type} -->`;
        }
    }
}
```

## Benefits of V2 Block System

### Extensibility Benefits
- **Plugin Architecture**: Easy to add new blocks without core changes
- **Community Contributions**: External developers can create block packs
- **Domain Specialization**: Industry-specific blocks can be developed separately
- **A/B Testing**: Multiple implementations of similar functionality

### Maintainability Benefits
- **Single Rendering Engine**: One codebase handles all block types
- **Consistent Validation**: Centralized input/output validation
- **Unified Testing**: Same test patterns for all blocks
- **Clear Documentation**: Self-documenting JSON schemas

### Performance Benefits
- **Lazy Loading**: Load block descriptors and handlers on demand
- **Caching**: Cache compiled descriptors and validation rules
- **Parallel Execution**: Independent blocks can run concurrently
- **Resource Management**: Timeout and retry logic built-in

### Developer Experience Benefits
- **Type Safety**: JSON schemas provide IDE support and validation
- **Hot Reloading**: Update block descriptors without restart
- **Easy Debugging**: Clear execution context and error reporting
- **Visual Editor**: Generate configuration UI automatically

This V2 block system provides a future-proof foundation that combines the best insights from all three original approaches while maintaining the flexibility needed for long-term growth.