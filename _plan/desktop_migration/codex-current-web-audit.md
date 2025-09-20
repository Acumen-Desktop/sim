# Current Web App Audit (apps/sim)

## High-Impact Subsystems
- **Workflow Canvas (`app/workspace/.../workflow.tsx`)**: ReactFlow-driven editor with Zustand stores (`stores/workflows`, `stores/panel`) and collaborative hooks (`hooks/use-collaborative-workflow.ts`). Any migration must replace ReactFlow, multi-user sockets, and state sync patterns.
- **Execution Pipeline (`executor/`)**: Browser-side TypeScript orchestrator that resolves inputs, handles loops/parallels, and streams results into UI stores. Depends heavily on serialized workflow shapes from `serializer/` and block definitions in `blocks/`.
- **API Layer (`app/api/**`)**: Next.js route handlers performing auth, persistence, and workflow actions through Drizzle + PostgreSQL (`@sim/db`). For a desktop MVP we can discard HTTP routing and re-expose only the minimal commands we still need via Tauri IPC.
- **Real-time/Collaboration (`socket-server/`, `useCollaborativeWorkflow`)**: Socket.io stack syncing workflow edits and execution status. Local desktop MVP can drop multi-user sync entirely.
- **Providers & Integrations (`providers/`, `services/queue`)**: Pluggable AI/model providers (OpenAI, Anthropic, Groq, etc.) and rate limiting. MVP can start with a single provider (e.g. OpenAI REST via Tauri backend) and add abstractions later.

## Data Model & Persistence
- **Drizzle Schema (`packages/db/schema.ts`)**: Complex multi-tenant tables (workspaces, workflows, permissions, logs, runs, knowledge, etc.). For desktop MVP we can collapse this into a handful of SQLite tables or even JSON blobs storing workflows and run history.
- **Workflow Serialization (`serializer/`)**: Converts ReactFlow nodes/edges + block configs into normalized workflow documents consumed by the executor. The shapes defined here should remain the contract between vanilla UI and the new backend.

## UX Surface Area to Retain for MVP
- Canvas editing (nodes, edges, block inspector, basic keyboard shortcuts).
- Workflow execution with inline logs/trace summaries.
- Minimal settings for API keys.
- Basic file import/export for workflows (replaces cloud sharing).

## Safe to Defer or Remove in MVP
- Account management, invitations, multi-workspace dashboards.
- Marketplace/publishing, analytics, usage telemetry.
- Background Trigger.dev jobs, queues, and webhooks.
- Multi-provider selection UI, rate limiting, org-wide settings.

## Technical Debt to Avoid Recreating
- Global Zustand store sprawl for features we can now scope locally.
- React-specific patterns (hooks, context) that complicate vanilla migration.
- Server/client duplication of validator logic; prefer single Rust source of truth for validation going forward.
