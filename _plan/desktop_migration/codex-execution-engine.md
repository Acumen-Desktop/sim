# Execution Engine Migration Plan

## Source References
- Current TypeScript executor at `apps/sim/executor/` with handlers per block type.
- Workflow serialization schema in `apps/sim/serializer/types.ts`.
- Block catalogs under `apps/sim/blocks/blocks/*.ts` providing metadata and runtime logic.

## Strategy Overview
1. **Keep workflow schema JSON-compatible** so frontend + legacy exports remain interoperable.
2. **Reimplement executor in Rust** for reliability + native async, but expose a thin compatibility layer so we can stub blocks in JS while porting.
3. **Layered design**: parser → scheduler → block runtime → result sink.

## Modules
- `execution/mod.rs`: entry point, orchestrates run pipeline.
- `execution/parser.rs`: validates incoming workflow JSON, builds adjacency lists, detects cycles.
- `execution/state.rs`: holds block states, variable stack, loop counters.
- `execution/blocks/`: one Rust module per supported block family (trigger, agent, http, response, condition).
- `execution/ai.rs`: delegates to provider adapters (see AI plan).
- `execution/events.rs`: standard structs for logs, traces, errors emitted back to frontend.

## MVP Block Coverage
- **Starter/Trigger**: manual start only.
- **Response**: final output aggregator.
- **Agent (LLM)**: single-step chat completion via OpenAI-compatible API.
- **HTTP/API**: basic GET/POST with JSON body.
- **Condition**: simple expression evaluation using `rhai` or a minimal safe interpreter.
- Defer loops, parallels, marketplace integrations until base app stable.

## Execution Flow
```
load workflow JSON
→ build DAG order
→ initialize state (inputs, variables, env)
→ for each layer:
    resolve inputs (map from previous outputs)
    execute block handler async
    record BlockLog + trace span
→ aggregate results → emit events → persist run history
```

## Error Handling & Debugging
- Wrap each block call in `anyhow::Result`; enrich errors with block id + friendly message.
- Stream partial logs so UI can highlight failing block immediately.
- Preserve `traceSpans` structure to keep compatibility with existing log viewers (subset only).

## Persistence Hooks
- After run completes, store execution summary in SQLite `runs` table (workflow_id, started_at, duration, status, summary_json).
- Allow frontend to request recent runs via `list_runs(workflow_id)` command.

## Extensibility
- Each block handler implements trait `BlockExecutor` with methods `supports(&self, block_type)`, `execute(&self, ctx, inputs) -> ExecutionResult`.
- Registry built at startup; handlers can be toggled via config file for plugin-style expansion.

## Transition Plan
- Phase 1: stub Rust executor that simply calls existing TypeScript execution logic via `tauri::invoke` to JS (if needed) while we finish Rust port.
- Phase 2: migrate critical blocks to Rust; keep TypeScript fallback for unsupported block types.
- Phase 3: remove JS executor once coverage acceptable.
