# Repository Guidelines

You are a senior engineer, format all markdown files you create with the naming convention "codex-<topic>-<filename>-##.md". Ask numbered questions and I will give numbered answers.

## Project Structure & Module Organization

- `apps/sim/` houses the Next.js 15 studio, including the workflow executor (`executor/`), UI components (`components/`), and API routes under `app/api/`.
- `apps/docs/` contains the Fumadocs-driven documentation site; author pages in `content/`, shared widgets in `components/`.
- `packages/cli`, `packages/ts-sdk`, and `packages/python-sdk` ship the CLI and SDKs; share types via `packages/ts-sdk/src`.
- Infra automation sits in `docker/`, `helm/`, and `scripts/`; Drizzle migrations live in `apps/sim/db/`.

## Build, Test, and Development Commands

- Install dependencies with `bun install` at the repo root; Turbo handles caching.
- Launch studio plus sockets via `bun run dev:full` inside `apps/sim`; `bun run dev` for UI only, `bun run dev:sockets` for realtime debugging.
- Use `bun run build` and `bun run test` to invoke the monorepo pipelines (Turbo).
- Quality gates: `bun run lint:all`, `bun run format:check`, `bun run type-check`.

## Coding Style & Naming Conventions

- Biome enforces 2-space indentation, LF endings, ≤100-char lines, single quotes, and sorted imports; run `bun run format` to auto-fix.
- Name React components `PascalCase`, hooks `useCamelCase`, keep files kebab-case like `workflow-node.tsx`, and export typed boundaries.
- Python SDK uses Black/isort (100-char width); after `pip install -e .[dev]` run `black . && isort .`.

## Testing Guidelines

- Keep Vitest specs near code (`*.test.ts(x)`); executor integrations live in `apps/sim/executor/tests/`.
- Run `bun run test` before pushing; for coverage-sensitive work use `bun run test:coverage` in `apps/sim`.
- Python client tests rely on Pytest (`cd packages/python-sdk && pytest`); fixtures sit in `tests/` with `test_<feature>.py` names.

## Commit & Pull Request Guidelines

- Follow Conventional Commits (`feat(executor): guard parallel branches`); releases follow `vX.Y.Z:` tags.
- Limit PRs to one scope; include summary, linked Linear/GitHub issue, local test results, and UI screenshots or Looms when visuals change.
- Note migrations or new env vars in the PR body so ops can update `.env` and Helm values.

## Environment & Configuration Tips

- Copy `apps/sim/.env.example` to `.env`, fill `DATABASE_URL`, `BETTER_AUTH_*`, telemetry keys; never commit secrets.
- Sync schema with `bun run -C apps/sim db:push` against pgvector and commit generated SQL.
