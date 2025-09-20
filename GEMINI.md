# Sim AI Project

## Project Overview

This is a monorepo for **Sim**, an open-source platform for building and deploying AI agent workflows. It features a visual drag-and-drop interface, allowing users to connect to over 100 applications. The project is built with a modern tech stack, including Next.js for the frontend, Bun as the runtime, and PostgreSQL with the `pgvector` extension for the database. The backend is powered by Drizzle ORM, and the application includes features like authentication, real-time updates with Socket.io, and background jobs with Trigger.dev.

The monorepo is managed with Turborepo and contains several applications and packages:

*   **`apps/sim`**: The main web application.
*   **`apps/docs`**: The documentation site, built with Fumadocs.
*   **`packages/db`**: The database schema and migrations, managed with Drizzle ORM.
*   **`packages/python-sdk`**: A Python SDK for interacting with the Sim platform.
*   **`packages/ts-sdk`**: A TypeScript SDK for interacting with the Sim platform.
*   **`packages/cli`**: A command-line interface for the Sim platform.

## Building and Running

### Prerequisites

*   [Bun](https://bun.sh/)
*   [Docker](https://www.docker.com/) (for the database)

### Installation

1.  Clone the repository:
    ```bash
    git clone https://github.com/simstudioai/sim.git
    ```
2.  Navigate to the project directory:
    ```bash
    cd sim
    ```
3.  Install dependencies:
    ```bash
    bun install
    ```

### Database Setup

The application requires a PostgreSQL database with the `pgvector` extension. The recommended way to set this up is with Docker:

```bash
docker run --name simstudio-db \
  -e POSTGRES_PASSWORD=your_password \
  -e POSTGRES_DB=simstudio \
  -p 5432:5432 -d \
  pgvector/pgvector:pg17
```

### Environment Variables

1.  Copy the example environment file in the `apps/sim` directory:
    ```bash
    cp apps/sim/.env.example apps/sim/.env
    ```
2.  Update the `DATABASE_URL` in `apps/sim/.env` with your database connection string:
    ```
    DATABASE_URL="postgresql://postgres:your_password@localhost:5432/simstudio"
    ```

### Database Migrations

Run the database migrations to set up the schema:

```bash
cd packages/db
bunx drizzle-kit migrate --config=./drizzle.config.ts
```

### Running the Application

You can run the main application and the socket server together or separately.

**Run both servers:**

```bash
bun run dev:full
```

**Run servers separately:**

*   Next.js app:
    ```bash
    bun run dev
    ```
*   Real-time socket server (in a separate terminal):
    ```bash
    cd apps/sim
    bun run dev:sockets
    ```

The application will be available at [http://localhost:3000](http://localhost:3000).

## Development Conventions

*   **Linting and Formatting**: The project uses [Biome](https://biomejs.dev/) for linting and formatting. You can run the following commands from the root of the project:
    *   `bun run format`: Format all files.
    *   `bun run lint`: Lint all files.
*   **Database**: The database schema is defined in `packages/db/schema.ts` using Drizzle ORM. When making changes to the schema, you will need to generate a new migration:
    ```bash
    cd packages/db
    bunx drizzle-kit generate
    ```
*   **Monorepo**: The project uses [Turborepo](https://turborepo.org/) to manage the monorepo. The configuration is in `turbo.json`.
*   **Commits**: Commit messages should follow the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) specification.
