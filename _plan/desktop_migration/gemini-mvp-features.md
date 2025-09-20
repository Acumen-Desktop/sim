# Gemini's MVP Features Plan for Sim AI Desktop

## 1. Core Principle: A Lean and Extensible Core

The MVP will focus on delivering a minimal, yet fully functional workflow editor that is both useful on its own and easily extensible. We will follow Claude's lean feature set, but implement it using Codex's generic block strategy to build a solid foundation for future growth.

## 2. Must-Have Features

### Workflow Canvas

*   A vanilla JavaScript, SVG-based canvas for creating and editing workflows.
*   Core interactions: pan, zoom, block dragging, and connecting.
*   Save and load workflows from the local SQLite database.

### Generic Block Strategy

Instead of hardcoding block types, we will use a declarative, data-driven approach as suggested by Codex. Each block will be defined by a JSON descriptor that specifies its properties, inputs, and outputs. The Rust backend will have a corresponding handler for each block type.

#### Block Descriptor Example (`http.request`)

```json
{
  "id": "http.request",
  "label": "HTTP Request",
  "category": "Network",
  "inputs": [
    { "id": "url", "type": "string", "required": true },
    { "id": "method", "type": "enum", "options": ["GET", "POST"], "default": "GET" }
  ],
  "outputs": [
    { "id": "body", "type": "json" },
    { "id": "status", "type": "number" }
  ]
}
```

### MVP Core Block Catalog

The MVP will include a small but powerful set of generic blocks:

1.  **Control Flow**
    *   `manual.start`: The entry point for a manually triggered workflow.
    *   `condition.if`: Basic conditional branching (if/else).

2.  **Data**
    *   `variable.set`: Define a variable in the workflow context.
    *   `template.string`: Create a string from a template with variables.

3.  **Network**
    *   `http.request`: Make a generic HTTP request.

4.  **AI**
    *   `ai.chat`: Send a prompt to a configured AI provider.

5.  **I/O**
    *   `file.read`: Read a file from the local filesystem (with user permission).
    *   `file.write`: Write a file to the local filesystem (with user permission).
    *   `output.display`: Display the final result in the UI.

### Workflow Execution Engine

*   A Rust-based engine that executes workflows sequentially.
*   Variable passing between blocks.
*   Real-time logging of execution progress to the UI.
*   Robust error handling.

### Local Data Storage

*   Workflows, execution history, and settings stored in a local SQLite database.
*   Secure, encrypted storage for API keys.

### Basic Settings

*   A simple UI for managing AI provider API keys.
*   Configuration of default models and other application settings.

## 3. Explicitly Excluded from MVP

To maintain focus, the following features will be excluded from the initial release:

*   **User Accounts & Cloud Sync:** The app will be single-user and local-only.
*   **Advanced Workflow Features:** No loops, parallel execution, or sub-workflows.
*   **Complex Integrations:** No OAuth-based integrations or third-party service connectors.
*   **Collaboration:** No real-time editing or sharing.

## 4. User Stories for MVP

*   **As a developer, I want to create a workflow that calls a REST API, processes the result with an AI model, and saves the output to a file.**
    *   `manual.start` -> `http.request` -> `ai.chat` -> `file.write`

*   **As a writer, I want to create a workflow that takes a topic, generates a blog post using an AI, and displays it to me.**
    *   `manual.start` -> `template.string` (to create a detailed prompt) -> `ai.chat` -> `output.display`

This MVP plan provides a clear path to a valuable and extensible product, delivering on the core promise of Sim AI in a lean, desktop-native package.
