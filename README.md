# Brick City Wars

This repository is a minimal pnpm workspace that hosts the shared game logic and the multiplayer server for the Brick City Wars prototype.

## Quick start

1. Install workspace dependencies:

   ```bash
   pnpm install
   ```

2. Launch the development servers (Fastify/Colyseus and Vite) with a single command:

   ```bash
   pnpm dev
   ```

3. Open the demo client in your browser at [http://localhost:5173](http://localhost:5173) and press **Play** to join a match.

## Prerequisites

- [Node.js](https://nodejs.org/) 20+
- [pnpm](https://pnpm.io/) 8+
- Redis and PostgreSQL (local instances or containers)

## Workspace commands

### Shared package

```bash
pnpm -C packages/shared test
```

### Server

Copy `.env.example` to `.env` inside `apps/server` and adjust the values for your environment. Then run:

```bash
pnpm -C apps/server dev
```

This starts a Fastify server that mounts Colyseus on `/ws` and exposes a basic health check on `/healthz`.

### Web client

```bash
pnpm -C apps/webclient dev
```

This launches a Vite development server serving the simple browser demo.

### C++ prototype client

```bash
cmake -S clients/cpp -B clients/cpp/build
cmake --build clients/cpp/build
./clients/cpp/build/BrickCityWarsClient
```

The stub client opens a terminal UI, connects to `ws://localhost:2567/match`, and falls back to an offline simulation when the server is unavailable.

### Build all projects

```bash
pnpm build
```

## Repository layout

- `packages/shared` — TypeScript utilities and deterministic board helpers used by both clients and server.
- `apps/server` — Fastify + Colyseus application that hosts multiplayer rooms.
- `apps/webclient` — Lightweight Vite front-end that connects to the match room and renders a 9×9 grid.
- `clients/cpp` — Experimental C++ terminal client with a basic WebSocket simulator.

## Development tips

- The workspace inherits compiler options from `tsconfig.base.json`.
- Each package manages its own build and test scripts; use `pnpm -C <path> <command>` to execute them in isolation.

## License

MIT
