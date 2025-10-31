# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

**Brick City Wars** is a multiplayer match-3 puzzle game prototype built as a pnpm monorepo with three main components:

### Project Structure

- **`packages/shared`** — Core TypeScript library containing deterministic board logic, match resolution, RNG, hashing, and Zod schemas for client-server messages. This package is consumed by both the server and web client.
- **`apps/server`** — Fastify + Colyseus multiplayer server that hosts match rooms, validates swaps, broadcasts state deltas, and runs a bot opponent when only one player is connected.
- **`apps/webclient`** — Vite-powered browser client with a 9×9 grid renderer and WebSocket connection to the match room.
- **`clients/cpp`** — Experimental C++ terminal client using CMake, with a minimal WebSocket bridge and offline simulation fallback. Built with Cocos2d-x patterns in mind for future graphical integration.

### Core Concepts

**Deterministic board engine:** All match resolution happens in `packages/shared/src/board.ts` using a seeded RNG (`packages/shared/src/rng.ts`). The server and clients can independently replay the same board state from a seed and RNG call count.

**State synchronization:** The server broadcasts `StateDelta` messages containing events (clear, spawn, hash) with sequence numbers. Clients acknowledge deltas and can detect desync via periodic hash checks (`packages/shared/src/hash.ts`).

**Match-3 resolution:** `applySwapAndResolve()` in `board.ts` validates swaps, applies gravity, detects 3+ matches in rows/columns, clears tiles, spawns new tiles (avoiding instant matches), and cascades until no matches remain. Each cascade increments the combo counter.

**Bot opponent:** `apps/server/src/rooms/MatchRoom.ts` starts a bot after 2 seconds if only one human is connected. The bot scans the 9×9 grid for all valid swaps and picks one at random every 800ms.

## Common Commands

### Install dependencies
```bash
pnpm install
```

### Development (runs server + webclient in parallel)
```bash
pnpm dev
```

### Run individual packages
```bash
pnpm -C packages/shared test           # Run shared package tests (Vitest)
pnpm -C packages/shared lint           # Type-check shared package
pnpm -C apps/server dev                # Start Fastify/Colyseus server (requires .env)
pnpm -C apps/webclient dev             # Start Vite dev server
```

### Build all packages
```bash
pnpm build
```

### Build and run C++ client

**Prerequisites:** Install [libwebsockets](https://libwebsockets.org/) for network support.

```bash
# macOS
brew install libwebsockets

# Ubuntu/Debian
sudo apt-get install libwebsockets-dev

# Windows (vcpkg)
vcpkg install libwebsockets
```

**Build and run:**
```bash
cmake -S clients/cpp -B clients/cpp/build
cmake --build clients/cpp/build
./clients/cpp/build/BrickCityWarsClient
```
On Windows, the executable will be in `clients/cpp/build/Debug/` or `clients/cpp/build/Release/`.

**Network Features:**
- Real WebSocket connection to `ws://localhost:2567`
- Colyseus protocol handshake (JOIN_ROOM message)
- Up to 3 retry attempts with exponential backoff (1s, 2s, 4s)
- Automatic fallback to offline simulation if server unavailable
- Real swap message sending when connected to live server

### Server setup
Copy `apps/server/.env.example` to `apps/server/.env` and configure Redis/PostgreSQL connection strings. The server listens on port 2567 for WebSocket connections at `/match`.

**Matchmaking System:**
- Redis-backed MMR-based matchmaking with expanding search bands
- MMR band starts at ±50, expands by 25 every 3 seconds up to ±200
- Players matched with bot after 15 seconds if no opponent found
- Background worker runs every 1 second to process queue and create matches

**Security Features:**
- JWT authentication with HS256 signing (24-hour token expiry)
- Build version gating - rejects clients with mismatched versions
- Rate limiting: Max 10 inputs/second per client (auto-disconnect on violation)
- Swap validation: Coordinates bounds check, tile existence check, match validity
- Suspicious activity detection: Boots clients after 5 invalid swaps

## Development Notes

- **TypeScript config:** All packages inherit from `tsconfig.base.json` (ES2020, CommonJS, strict mode).
- **Package manager:** Always use `pnpm` (workspace protocol references like `workspace:*`).
- **Testing:** Shared package uses Vitest. Run individual tests with `pnpm -C packages/shared test -- <test-file>`.
- **Server endpoints:**
  - Fastify HTTP API on port 3000: `/healthz`, `/version`
  - Auth API: `POST /api/auth/token`, `POST /api/auth/verify`
  - Matchmaking API: `POST /api/queue`, `POST /api/queue/leave`, `GET /api/queue/room`, `GET /api/queue/stats`
  - Colyseus WebSocket on port 2567 at `/ws`
- **C++ client controls:** Arrow keys to move cursor, Enter to select tiles (swap queued on second selection), Escape to exit.
- **Authentication flow:**
  1. Client calls `POST /api/auth/token` with `{playerId, buildVersion}` to get JWT
  2. Client includes token in Colyseus connection options: `{token, playerId, buildVersion}`
  3. Server validates JWT signature, expiry, and build version in `onAuth`
- **Matchmaking flow:**
  1. Client calls `POST /api/queue` with `{playerId, mmr}`
  2. Client polls `GET /api/queue/room?playerId=X` until match found
  3. Client connects to Colyseus room with `{playerId, token}` in options
  4. Server validates player is allowed in matched room via `onAuth`
