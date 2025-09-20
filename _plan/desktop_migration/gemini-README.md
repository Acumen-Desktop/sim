# Gemini's Plan for Sim AI Desktop Migration

This directory contains a comprehensive plan for migrating Sim AI from a web-based application to a Tauri desktop application. This plan synthesizes the best ideas from the initial proposals by Claude and Codex, augmented with up-to-date research on the latest technologies.

## Core Philosophy

The migration will be a **pragmatic fresh start**. We will build a new application from the ground up to create a lean, performant, and maintainable desktop-native experience, while strategically reusing proven concepts and data structures from the existing application.

## Key Planning Documents

1.  **[Architecture (`gemini-architecture.md`)]**: Outlines the high-level architecture of the application, featuring a vanilla JavaScript frontend and a Rust-based backend, communicating via a secure IPC bridge.

2.  **[Canvas (`gemini-canvas.md`)]**: Details the implementation of a custom, SVG-based workflow canvas that is both lightweight and performant, while maintaining compatibility with the existing workflow format.

3.  **[Database (`gemini-database.md`)]**: Describes the local-first data persistence strategy, using SQLite for structured data and a secure, encrypted store for API keys.

4.  **[AI Integration (`gemini-ai-integration.md`)]**: Explains the plan for a flexible and secure AI integration, including a provider abstraction system and support for local inference with Ollama and the Burn framework.

5.  **[MVP Features (`gemini-mvp-features.md`)]**: Defines the minimal viable product, focusing on a core set of generic, extensible blocks and essential features for a useful initial release.

6.  **[Migration Strategy (`gemini-migration-strategy.md`)]**: Provides a phased, step-by-step plan for the migration, from initial project setup to final deployment.

## Technical Stack

*   **Frontend:** Vanilla JavaScript (ES6+), HTML5, CSS3
*   **Backend:** Rust, Tauri
*   **Database:** SQLite
*   **Core Crates:** `tokio`, `rusqlite`, `reqwest`, `serde`, `thiserror`
*   **Security:** `tauri-plugin-stronghold` (or similar)

This plan provides a clear and comprehensive roadmap for the successful migration of Sim AI to a modern, desktop-native application.
