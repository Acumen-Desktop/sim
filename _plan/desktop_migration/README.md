# Sim AI Desktop Migration Planning

This directory contains comprehensive planning documents for migrating Sim AI from a web-based application to a Tauri desktop application.

## Planning Documents

### 1. [Database Options](claude-db-options.md)
**Focus**: Replace PostgreSQL with SQLite
- Zero-configuration single-file database
- Simplified schema with JSON blobs
- Local file storage approach
- Eliminates Docker dependency

### 2. [Architecture Overview](claude-architecture.md)
**Focus**: Hybrid desktop architecture
- Vanilla HTML/CSS/JS frontend (no frameworks)
- Rust backend with Tauri IPC
- Local-first design principles
- Simplified data flow patterns

### 3. [Canvas Implementation](claude-canvas.md)
**Focus**: Replace ReactFlow with vanilla JS
- SVG-based infinite canvas
- Custom drag-and-drop system
- Manual DOM manipulation
- Zero external dependencies

### 4. [AI Integration](claude-ai-integration.md)
**Focus**: Direct API calls from Rust backend
- Provider abstraction system
- Local API key management
- Streaming via Tauri events
- Optional Burn framework integration for local inference

### 5. [MVP Features](claude-mvp-features.md)
**Focus**: Minimal viable product scope
- Core block types (Input, Output, AI Agent, HTTP, Condition, Text)
- Basic workflow execution
- Local file storage
- Essential settings management

### 6. [Migration Strategy](claude-migration-strategy.md)
**Focus**: Step-by-step migration approach
- Fresh start vs. porting approach
- Phase-by-phase implementation plan
- Risk mitigation strategies
- Success metrics and go-live strategy

## Key Principles

### KISS (Keep It Simple, Stupid)
- **No Docker**: Single executable application
- **No frameworks**: Vanilla web technologies
- **No complex database**: SQLite with simple schema
- **No authentication**: Single-user desktop app
- **No cloud features**: Local-only operation

### Desktop-Native Approach
- **Fast startup**: < 2 seconds application launch
- **Offline capable**: Works without internet (except AI API calls)
- **Portable**: Single file database and settings
- **Cross-platform**: Windows, macOS, Linux support

### Simplified Architecture
- **Frontend**: Pure HTML/CSS/JS served directly
- **Backend**: Rust with minimal dependencies
- **Storage**: Local files + SQLite
- **Communication**: Tauri IPC system

## Implementation Timeline

**Total Estimated Time**: 12 weeks for MVP

1. **Weeks 1-2**: Foundation setup and Tauri project structure
2. **Weeks 3-4**: Canvas implementation and basic blocks
3. **Weeks 5-6**: AI integration and provider system
4. **Weeks 7-8**: Execution engine and workflow processing
5. **Weeks 9-10**: Storage system and additional block types
6. **Weeks 11-12**: Testing, polish, and release preparation

## Technical Stack

### Frontend
- **Language**: Vanilla JavaScript (ES6+)
- **Rendering**: Direct DOM manipulation
- **Canvas**: SVG with custom interaction handling
- **Styling**: Pure CSS with CSS Grid/Flexbox

### Backend
- **Language**: Rust
- **Framework**: Tauri 1.0+
- **Database**: SQLite with rusqlite
- **HTTP Client**: reqwest for AI API calls
- **Async Runtime**: Tokio

### Dependencies (Minimal)
- **Tauri**: Desktop app framework
- **rusqlite**: Local database
- **reqwest**: HTTP client for AI APIs
- **serde**: JSON serialization
- **tokio**: Async runtime
- **thiserror**: Error handling

## Expected Benefits

1. **Simplicity**: Dramatically reduced complexity compared to web version
2. **Performance**: Native desktop performance with Rust backend
3. **Reliability**: Fewer dependencies and simpler architecture
4. **Portability**: Single executable with embedded resources
5. **Privacy**: All data stays local on user's machine
6. **Maintainability**: Clear separation of concerns and minimal codebase

## Limitations Accepted

1. **Single User**: No multi-user or collaboration features
2. **Local Only**: No cloud sync or sharing capabilities
3. **Basic UI**: Simplified interface compared to web version
4. **Limited Integrations**: Only essential AI providers and basic tools
5. **Manual Setup**: Users manage their own API keys and settings

This migration approach prioritizes shipping a working, simple desktop application over feature completeness, following the principle that a working MVP is better than a complex system that never ships.