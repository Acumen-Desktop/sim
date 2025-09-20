# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sim is an AI agent workflow platform that allows users to build and deploy AI workflows with a visual, block-based interface. This is a TypeScript/JavaScript monorepo built with Next.js and Bun, featuring real-time collaboration, extensive tool integrations, and background job processing.

## Architecture

### Monorepo Structure
- **apps/sim** - Main Next.js application (workflow editor, execution engine)
- **apps/docs** - Documentation site built with Fumadocs
- **packages/db** - Shared database schema and utilities (Drizzle ORM)
- **packages/cli** - NPM package (`simstudio`) for easy deployment
- **packages/ts-sdk** - TypeScript SDK for integrations
- **packages/python-sdk** - Python SDK for integrations

### Tech Stack
- **Runtime**: Bun (not Node.js)
- **Framework**: Next.js 15 with App Router
- **Database**: PostgreSQL with pgvector extension (required for AI embeddings)
- **ORM**: Drizzle ORM with migrations
- **UI**: Shadcn/ui components with Tailwind CSS
- **State Management**: Zustand stores
- **Real-time**: Socket.io server
- **Background Jobs**: Trigger.dev
- **Authentication**: Better Auth
- **Code Quality**: Biome for linting and formatting

### Key Architecture Patterns

**Executor System** (`apps/sim/executor/`)
- Workflow execution engine that processes block-based workflows
- Handles loops, parallels, conditionals, and tool routing
- Manages execution state and flow control

**Block Registry** (`apps/sim/blocks/`)
- Modular system for workflow components
- Each block type defines its structure, validation, and execution logic
- Extensible for custom block types

**Tool System** (`apps/sim/tools/`)
- 50+ integrations (Google, Slack, GitHub, databases, etc.)
- Standardized tool interface with parameter validation
- Dynamic tool loading and registration

**Real-time Collaboration** (`apps/sim/socket-server/`)
- Socket.io server for live workflow collaboration
- Handles room management and real-time updates
- Separate from main Next.js server

## Development Commands

### Starting Development
```bash
# Start both main app and realtime server (recommended)
bun run dev:full

# Alternative: Start servers separately
bun run dev              # Main Next.js app only
bun run dev:sockets      # Socket server only (from apps/sim)
```

### Building and Testing
```bash
bun run build           # Build all packages
bun run test            # Run tests across all packages
bun run test:watch      # Run tests in watch mode (from apps/sim)
bun run test:coverage   # Run tests with coverage (from apps/sim)
```

### Code Quality
```bash
bun run lint            # Lint and fix with Biome
bun run lint:check      # Check linting without fixing
bun run format          # Format code with Biome
bun run format:check    # Check formatting without fixing
bun run type-check      # TypeScript type checking
```

### Database Management
```bash
# From packages/db directory:
bun run db:migrate      # Run database migrations
bun run db:push         # Push schema changes
bun run db:studio       # Open Drizzle Studio
```

## Testing Strategy

- **Test Framework**: Vitest with React Testing Library
- **Test Files**: `*.test.ts` and `*.test.tsx` files
- **Location**: Tests are co-located with source files
- **Configuration**: `apps/sim/vitest.config.ts` with path aliases
- **Coverage**: Available via `bun run test:coverage`

## Database Requirements

PostgreSQL with pgvector extension is **required** for:
- AI embeddings in knowledge bases
- Semantic search functionality
- Vector similarity operations

## Environment Setup

Key environment variables (see `apps/sim/.env.example`):
- `DATABASE_URL` - PostgreSQL connection string
- `BETTER_AUTH_SECRET` - Authentication secret
- `BETTER_AUTH_URL` - Application URL
- `COPILOT_API_KEY` - For self-hosted instances

## Code Style Guidelines

- **Imports**: Organized by groups (Node, React, packages, local)
- **Formatting**: 2-space indentation, single quotes, semicolons as needed
- **TypeScript**: Strict mode enabled, prefer explicit types
- **Components**: Use Shadcn/ui patterns, avoid inline styles
- **File Naming**: kebab-case for directories, camelCase for TypeScript files

## Deployment Options

1. **Cloud**: https://sim.ai (hosted service)
2. **CLI**: `npx simstudio` (requires Docker)
3. **Docker Compose**: Production and Ollama configurations available
4. **Manual**: Bun runtime with PostgreSQL setup

## Package Management

- Use `bun install` instead of `npm install`
- Workspace dependencies managed via `packages/*` structure
- Shared dependencies in root `package.json`
- Individual package configs in respective directories