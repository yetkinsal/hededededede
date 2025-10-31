# Brick City Wars — Production Deployment Guide

This guide walks you through deploying the server (Fastify + Colyseus + Postgres + Redis) to Railway, and the web client (Vite static site) to Netlify or Vercel.

It covers prerequisites, environment variables, DB/Redis provisioning, one-time migrations, WebSocket configuration, and verification.

---

## Overview

- Server (TypeScript): `apps/server`
  - HTTP API (Fastify) and Colyseus real-time game server
  - DB: PostgreSQL, optional but recommended (players, matches)
  - Cache/queue: Redis (matchmaking)
- Webclient (Vite): `apps/webclient`
  - Static site; connects to server via Colyseus WebSocket

Important defaults (from code):
- HTTP server listens on `PORT` (defaults to 3000) at `0.0.0.0`
- Colyseus runs on the same HTTP port (no separate `2567` needed)
- Required env vars: `JWT_SECRET`, `REDIS_URL`, `PG_URL` (optional but recommended), `BUILD_VERSION` (optional), `LOG_LEVEL` (optional)

> Note on platforms: Railway typically exposes a single dynamic `PORT`. If you keep Colyseus on port 2567, your service may not be reachable externally on that port. See "Important code tweaks (recommended)" below to run Colyseus on the same HTTP port.

---

## Prerequisites

- GitHub repo connected (this monorepo)
- Accounts:
  - Railway (for server + DB + Redis)
  - Netlify or Vercel (for webclient)
- Node.js 20+ and pnpm 8+ locally (for testing)

---

## Implemented tweaks (already applied)

To simplify deployment on Railway (single exposed port), the code now:

1) Runs Colyseus on the same HTTP server
- `apps/server/src/colyseus.ts`: Colyseus `Server` is attached to Fastify's `app.server`.
- `apps/server/src/index.ts`: Removed the separate `gameServer.listen(2567)`; both HTTP and Colyseus share `PORT`.

2) Makes the webclient WS URL configurable
- `apps/webclient/src/main.js`: Uses `VITE_WS_URL` with a fallback to current origin (`wss://<host>` over HTTPS).
- All hardcoded `:2567` usages were replaced accordingly.

Result: You can point the client at `wss://<your-railway-domain>` without custom ports.

---

## Part A — Deploy the server to Railway

### 1) Create a Railway project and services

- Create a new Railway project
- Add a service from GitHub repository
  - Service root: repository root (Railway auto-detects)
  - Alternatively, set the service to run from `apps/server`

### 2) Add PostgreSQL and Redis

- In the Railway project, add:
  - A PostgreSQL plugin
  - A Redis plugin
- Note the connection strings Railway provides (e.g. `DATABASE_URL` for Postgres; a `REDIS_URL` for Redis depending on plugin — sometimes named `REDIS_URL` already).

### 3) Environment variables

Add the following to the server service in Railway (Variables tab):

- `NODE_ENV=production`
- `JWT_SECRET=<generate a strong random secret>`
- `REDIS_URL=<Redis connection string>`
- `PG_URL=<Postgres connection string>`
  - If Railway provides `DATABASE_URL`, you can set `PG_URL` to the same value (`PG_URL=${DATABASE_URL}`)
- `BUILD_VERSION=0.1.0` (optional)
- `LOG_LEVEL=info` (optional)

The server listens on `0.0.0.0` and uses `PORT` assigned by Railway automatically (no need to set `PORT`).

Redis on Railway usually requires TLS. Use the Redis plugin's Connection URL directly:

- Prefer setting only: `REDIS_URL=<exact Connection URL from the Redis plugin>`
  - If the URL starts with `rediss://`, TLS will be auto-enabled by the server
  - If the URL starts with `redis://` but your provider requires TLS, also set `REDIS_TLS=true`
- Avoid building the URL from other variables or using GitHub Actions-style syntax like `${{VAR}}` in Railway env vars; set the full URL value directly as provided by the plugin.

### 4) Install/build/start commands (Railway)

Recommended (monorepo with pnpm workspaces):

- Install: `pnpm -w install --frozen-lockfile`
- Build: `pnpm --filter @brick-city-wars/server build`
- Start: `pnpm --filter @brick-city-wars/server start`

If your service path is set to `apps/server`, then:

- Install: `pnpm install --frozen-lockfile`
- Build: `pnpm build`
- Start: `pnpm start`

### 5) Run DB migrations (one-time, and on schema updates)

From the Railway service shell or as a one-off deployment command:

- If running from repo root:
  - `pnpm --filter @brick-city-wars/server migrate`
- If running from `apps/server` service path:
  - `pnpm migrate`

This will:
- Create the target DB if missing (connects to `postgres` db first)
- Apply `apps/server/src/db/schema.sql`

### 6) Networking and Colyseus

- Colyseus rides on the same HTTP port; the client should connect to `wss://<your-railway-domain>` (no custom port).

### 7) Verify health/check logs

- After deploy, open `https://<your-railway-domain>/healthz` → should return `{"status":"ok"}`
- Check logs for DB/Redis connect and "Server listening"/"Colyseus listening" messages

---

## Part B — Deploy the webclient

You can use Netlify or Vercel. Steps for both are similar.

### Option 1: Netlify

1) New site from Git
- Pick the `brickcitywars` repo
- Base directory: `apps/webclient`
- Build command: `pnpm build`
- Publish directory: `dist`
- Node version: 20+
- pnpm: enable pnpm in Netlify (via UI or `NETLIFY_USE_PNPM=true` env var), or add a preinstall command

2) Environment variables (required)
- `VITE_WS_URL` → WebSocket URL to your server
  - Example: `wss://match3-production.up.railway.app`
- `VITE_API_BASE` → Base HTTPS URL for REST API calls (auth/queue)
  - Example: `https://match3-production.up.railway.app`

Why: The web app is hosted on Netlify, but API routes like `/api/auth/token` live on your Railway server. Without `VITE_API_BASE`, the client will call Netlify's origin (404). With it set, the client will call the Railway host for API and the same host for WebSocket via `VITE_WS_URL`.

3) Deploy and test
- After deploy, open the site
- Click Play to connect and queue
- If you see connection errors, open the browser console and see Troubleshooting below

### Option 2: Vercel

1) Import Git repo in Vercel
- Project settings → Root directory: `apps/webclient`
- Framework preset: Vite
- Build command: auto or `pnpm build`
- Output directory: `dist`
- Node version: 20+

2) Environment variables
- `VITE_WS_URL` → `wss://<your-railway-domain>`
- `VITE_API_BASE` → `https://<your-railway-domain>`

3) Deploy and test

---

## CORS and security

- The server currently enables CORS with `origin: true` (allow-all). In production, consider restricting to your public web domains.
  - Option: introduce `ALLOWED_ORIGINS` and wire Fastify CORS accordingly
- Ensure you use `wss://` from HTTPS sites; mixed content will be blocked by browsers

---

## End-to-end smoke test

1) Open the deployed webclient domain
2) Open DevTools Console (F12)
3) Click Play
4) Expected:
   - Token creation via `/api/auth/token`
   - Join queue via `/api/queue`
   - WebSocket connects to Colyseus
   - Match is made (or a bot match if implemented)

If using DB:
- Verify tables exist in Postgres (`players`, `match_history`)
- Play a match and confirm match records are inserted

---

## Troubleshooting

- WebSocket connection fails immediately
  - Most common: client is trying to connect to `:2567` but your server only exposes one port
  - Fix: move Colyseus to share the HTTP server (see code tweaks), and set `VITE_WS_URL` to your HTTPS domain (wss)

- API 404s like `POST https://<your-netlify-domain>/api/auth/token 404`
  - Cause: client defaults to current origin for API calls when `VITE_API_BASE` is not set
  - Fix: set `VITE_API_BASE` to `https://<your-railway-domain>` in Netlify/Vercel environment settings and redeploy

- CORS errors on API calls
  - Ensure server CORS allows your web domain
  - From an HTTPS site, only use `https://` and `wss://` URLs

- DB connection fails
  - Verify `PG_URL` is set, and credentials are correct
  - On Railway, `PG_URL` can mirror `DATABASE_URL`

- Redis errors
  - Use the Redis plugin's Connection URL verbatim for `REDIS_URL` (ideally it starts with `rediss://`)
  - If your URL uses `redis://` and your provider requires TLS, also set `REDIS_TLS=true`
  - Do not point `REDISHOST` to your service's `RAILWAY_PRIVATE_DOMAIN` (that's your app, not Redis); use the host from the Redis plugin
  - Avoid `${{...}}` syntax in Railway variables; it won't expand like GitHub Actions — set actual literal values

- 500 errors on `/api/auth/token`
  - Ensure `JWT_SECRET` is set in Railway

- Migrations re-running
  - Script is idempotent-ish; if tables already exist you may see "already exists" notes. This is fine when re-deploying

---

## Quick reference

- Server build/start (from repo root):
  - Build: `pnpm --filter @brick-city-wars/server build`
  - Start: `pnpm --filter @brick-city-wars/server start`
  - Migrate: `pnpm --filter @brick-city-wars/server migrate`
- Webclient build (from repo root):
  - `pnpm --filter @brick-city-wars/webclient build` → outputs to `apps/webclient/dist`

---

## Next steps (optional)

- Add CI/CD (e.g., GitHub Actions) for build checks
- Harden CORS
- Add health checks/dashboard
- Custom domains on Railway + Netlify/Vercel
- Add a `VITE_API_BASE` env var if you later split API and WS hosts/paths
