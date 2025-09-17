# Sim AI Local Setup Plan (Mac, No Docker)

## Overview
Follow these steps to run Sim AI locally on macOS for evaluation without using Docker containers.

## Current machine readiness (PostgreSQL)
- Client: `psql` from Postgres.app 17.6 (`/Applications/Postgres.app/...`).
- Server: Homebrew PostgreSQL 15.14 is running (`brew services` shows `postgresql@15 started`).
- Port: default 5432 reachable; `psql -d postgres -c 'select version();'` returns 15.14 (Homebrew).
- Databases present: `postgres`, `template0`, `template1`, and `dify`. No `simstudio` yet.
- Extensions in `postgres` DB: only `plpgsql`. Homebrew `pgvector` is now installed (0.8.1) but not yet enabled in a database.
- Bun: 1.2.22 (meets requirement `>=1.2.13`).
- Env file: `apps/sim/.env` exists and currently points `DATABASE_URL` to `postgresql://postgres:password@localhost:5432/postgres` — this is unlikely to work on your local Homebrew server (no `postgres` password set by default).

## Gaps to fix before running
- Create the application database (recommend `simstudio`) and enable the `pgvector` extension in it.
- Update `apps/sim/.env` with a working `DATABASE_URL` for your local server and generate required secrets.

## 1. Prepare the System
1. Verify macOS 13+ and install [Homebrew](https://brew.sh/) if missing (`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`).
2. Update Homebrew formulas: `brew update`.

## 2. Database setup (Homebrew PostgreSQL 15 + pgvector)
1. Bun is already installed (1.2.22). Confirm: `bun --version`.
2. Keep your existing Homebrew PostgreSQL 15 service running. Confirm: `brew services list | grep postgresql`.
3. Install pgvector for Postgres 15:
   - Try: `brew install pgvector`
   - If Homebrew reports a tap requirement, use: `brew install pgvector/brew/pgvector`
4. Create the application database owned by your macOS user (e.g., `james`):
   - `createdb simstudio`
5. Enable the pgvector extension inside `simstudio`:
   - `psql -d simstudio -c "CREATE EXTENSION IF NOT EXISTS vector;"`
6. Verify availability (should list `vector`):
   - `psql -d simstudio -Atqc "select extname from pg_extension order by 1;"`

## 3. Fetch Source Code
1. Clone the repo: `git clone https://github.com/simstudioai/sim.git`.
2. Change into the project directory: `cd sim`.

## 4. Configure Environment
1. Ensure an env file exists: `cp apps/sim/.env.example apps/sim/.env` (already present in this repo).
2. Set `DATABASE_URL` to your local DB. For Homebrew Postgres with OS user auth, this typically works without a password:
   - `DATABASE_URL="postgresql://localhost:5432/simstudio"`
   - If you prefer to be explicit: `postgresql://james@localhost:5432/simstudio`
3. Generate and set required secrets (32 bytes hex each):
   - `BETTER_AUTH_SECRET=$(openssl rand -hex 32)`
   - `ENCRYPTION_KEY=$(openssl rand -hex 32)`
   - `INTERNAL_API_SECRET=$(openssl rand -hex 32)`
   - `BETTER_AUTH_URL=http://localhost:3000`
   - `NEXT_PUBLIC_APP_URL=http://localhost:3000`
4. Optional: add provider keys as needed (OpenAI, Anthropic, etc.).

## 5. Install Dependencies
1. Install workspace packages from the repo root: `bun install`.
2. If you plan to use the CLI or SDKs separately, build them with `bun run -C packages/cli build` or `bun run -C packages/ts-sdk build`.

## 6. Prepare the Database Schema
1. Sync schema with Drizzle from `apps/sim`: `bun run -C apps/sim db:push`.
2. Confirm tables exist: `psql -d simstudio -c "\dt"` should list the schema.

## 7. Launch the Application
1. Start full development stack with sockets: `bun run -C apps/sim dev:full`.
2. Visit http://localhost:3000 to confirm the studio loads and the realtime console connects.

## 8. Optional Validation
1. Run unit tests: `bun run test` (repo root) or scoped `bun run -C apps/sim test`.
2. Check linting/formatting before committing: `bun run lint:all` and `bun run format:check`.
3. Stop services when done: `Ctrl+C` in the dev terminal and `brew services stop postgresql@15` if you no longer need the database.

## Notes
- You currently have both Postgres.app tooling and a Homebrew-managed PostgreSQL server. That’s fine; the Homebrew server is the active one. If you prefer to standardize, we can switch entirely to Postgres.app 17, but it is not required.
- The repository’s `AGENTS.md` recommends `bun run -C apps/sim db:push` for schema sync, which is reflected above. A `db:migrate` script also exists if you prefer running generated migrations.

## Homebrew/pgvector install status
- Attempted `brew install pgvector`, but encountered active Homebrew locks originating from another running brew process (e.g., lock on `ca-certificates`).
- Removed specific stale lock files (`pgvector.formula.lock`, `icu4c@77.formula.lock`), but a separate brew process is still holding a lock.
- Next steps to resolve:
  - Option A (wait): Allow the running `brew upgrade`/build to finish, then rerun `brew install pgvector`.
  - Option B (terminate): Stop the running brew processes, then run:
    - `brew install pgvector`
    - `createdb simstudio`
    - `psql -d simstudio -c "CREATE EXTENSION IF NOT EXISTS vector;"`
    - Verify: `psql -d simstudio -Atqc "select extname from pg_extension order by 1;"` should include `vector`.
 - Resolved: Stopped lingering Homebrew processes, removed locks, and successfully installed `pgvector 0.8.1`.

## Questions
1) Do you want to keep using Homebrew PostgreSQL 15 (recommended, already running) or switch to Postgres.app 17 as the server?
2) Do you want to use your macOS user (no password) for local development, or should I create a dedicated `sim` role with a password and update `DATABASE_URL` accordingly?
3) Would you like me to update `apps/sim/.env` now with a working `DATABASE_URL` and generated secrets?
