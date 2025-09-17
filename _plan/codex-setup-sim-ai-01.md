# Sim AI Local Setup Plan (Mac, No Docker)

## Overview
Follow these steps to run Sim AI locally on macOS for evaluation without using Docker containers.

## 1. Prepare the System
1. Verify macOS 13+ and install [Homebrew](https://brew.sh/) if missing (`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`).
2. Update Homebrew formulas: `brew update`.

## 2. Install Runtime Dependencies
1. Install Bun (required repo-wide): `curl -fsSL https://bun.sh/install | bash` and add Bun to your shell profile.
2. Ensure Node.js 20+ is available (Bun bundles one; verify with `bun --version`).
3. Install PostgreSQL 16 and pgvector natively:
   - `brew install postgresql@16`
   - `brew install pgvector/brew/pgvector`
4. Initialize and start the database service:
   - `brew services start postgresql@16`
   - Create the Sim database user and db: `createuser sim --createdb` and `createdb simstudio -O sim`.
5. Enable the pgvector extension inside the database:
   - `psql -d simstudio -c "CREATE EXTENSION IF NOT EXISTS vector;"`

## 3. Fetch Source Code
1. Clone the repo: `git clone https://github.com/simstudioai/sim.git`.
2. Change into the project directory: `cd sim`.

## 4. Configure Environment
1. Copy the sample environment file: `cp apps/sim/.env.example apps/sim/.env`.
2. Fill required secrets (`DATABASE_URL=postgresql://sim@localhost:5432/simstudio`, `BETTER_AUTH_*`, 3rd-party keys) using a text editor.
3. For telemetry or optional integrations, stub values if you do not have production credentials.

## 5. Install Dependencies
1. Install workspace packages from the repo root: `bun install`.
2. If you plan to use the CLI or SDKs separately, build them with `bun run -C packages/cli build` or `bun run -C packages/ts-sdk build`.

## 6. Prepare the Database Schema
1. Run Drift/Drizzle migrations from `apps/sim`: `bun run -C apps/sim db:push`.
2. Confirm tables exist: `psql -d simstudio -c "\dt"` should list the schema.

## 7. Launch the Application
1. Start full development stack with sockets: `bun run -C apps/sim dev:full`.
2. Visit http://localhost:3000 to confirm the studio loads and the realtime console connects.

## 8. Optional Validation
1. Run unit tests: `bun run test` (repo root) or scoped `bun run -C apps/sim test`.
2. Check linting/formatting before committing: `bun run lint:all` and `bun run format:check`.
3. Stop services when done: `Ctrl+C` in the dev terminal and `brew services stop postgresql@16` if you no longer need the database.
