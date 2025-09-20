# Gemini's Architecture Plan for Sim AI Desktop

## 1. Core Philosophy: A Pragmatic Fresh Start

This plan adopts a "pragmatic fresh start" approach, blending the best ideas from both Claude and Codex. We will build a new Tauri application from the ground up to avoid inheriting technical debt, but will strategically reuse existing data structures and concepts where it makes sense. The primary goal is a lean, maintainable, and performant desktop application that is true to the spirit of Sim AI.

## 2. High-Level Architecture

The application will follow a classic hybrid desktop model, with a clear separation of concerns between the frontend and backend.

```
+-----------------------------------+
|       Tauri Application           |
|                                   |
|  +-----------------------------+  |
|  |        Frontend             |  |
|  | (Vanilla JS, HTML, CSS)     |  |
|  +-----------------------------+  |
|               ^                 |
|               | IPC             |
|               v                 |
|  +-----------------------------+  |
|  |        Backend              |  |
|  | (Rust, Tauri)               |  |
|  +-----------------------------+  |
|                                   |
+-----------------------------------+
```

### Frontend (WebView)

*   **Stack:** Vanilla JavaScript (ES6+), HTML5, and CSS3.
*   **Rendering:** Direct DOM manipulation. No virtual DOM or frontend frameworks.
*   **Responsibilities:**
    *   UI rendering and state management.
    *   User interaction handling (drag-and-drop, etc.).
    *   Invoking backend commands via Tauri's IPC bridge.
    *   Listening for and reacting to events from the backend.

### Backend (Rust)

*   **Stack:** Rust with the Tauri framework.
*   **Core Crates:**
    *   `tauri`: The core application framework.
    *   `tokio`: For asynchronous operations.
    *   `rusqlite`: For SQLite database interaction.
    *   `reqwest`: For making HTTP requests to AI provider APIs.
    *   `serde`: For data serialization and deserialization.
    *   `thiserror`: For robust error handling.
*   **Responsibilities:**
    *   All business logic.
    *   Workflow execution.
    *   Database operations.
    *   Filesystem access.
    *   Direct interaction with external APIs (e.g., AI providers).
    *   Securely managing secrets and API keys.

## 3. Communication: Tauri IPC & Events

Communication between the frontend and backend will be handled exclusively through Tauri's built-in mechanisms.

*   **Commands:** The frontend will use `invoke` to call specific Rust functions exposed as Tauri commands. This is for request-response style interactions (e.g., "save this workflow").
*   **Events:** The backend will `emit` events to the frontend for asynchronous notifications and streaming data (e.g., "a new log message is available," "here is the next token from the AI").

This ensures a decoupled and secure architecture where the frontend has no direct access to system resources.

## 4. Directory Structure

The project will follow a standard Tauri layout, promoting a clean separation of frontend and backend code.

```
sim-desktop/
├── src/                      # Frontend code
│   ├── index.html
│   ├── styles/
│   │   └── app.css
│   └── js/
│       ├── app.js
│       ├── canvas.js
│       └── ipc.js
├── src-tauri/                # Backend code (Rust)
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands.rs
│   │   ├── database.rs
│   │   ├── executor.rs
│   │   └── error.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
└── README.md
```

## 5. Key Architectural Decisions

*   **No Web Frameworks:** We are intentionally avoiding React, Vue, etc., to minimize complexity, reduce bundle size, and maximize performance.
*   **Rust as the Engine:** All significant logic will be in Rust. This leverages Rust's performance, safety, and rich ecosystem of crates.
*   **Local-First:** The application will be designed to work entirely offline, with the exception of features that explicitly require an internet connection (like calling a cloud-based AI provider).
*   **Simple State Management:** Frontend state will be managed with simple JavaScript objects and an event-driven pattern. We will avoid complex state management libraries.
*   **Security First:** API keys and other secrets will be handled securely on the Rust backend, using plugins like `tauri-plugin-stronghold` or `tauri-plugin-store` with encryption. The frontend will never have direct access to these secrets.

This architecture provides a solid and scalable foundation for the Sim AI desktop application, prioritizing simplicity, performance, and security.
