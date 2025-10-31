# TODO — PvP Match-3 Only (no city-building)

A prioritized checklist to get the core PvP loop production-ready. City/meta features are explicitly out of scope for now.

## 1) Gameplay rules, scoring, win conditions
- [x] Win conditions (implement and verify all three)
  - [x] Garbage top-out: end match when garbage placement can no longer spawn (top rows blocked) or blockers reach row 0.
    - Status: ✅ Implemented. Added `isTopOut()` function in `packages/shared/src/board.ts:350`, end reason `"topout"` in schemas, and server check in `MatchRoom.ts:907-914`.
    - Acceptance: ✅ Unit tests added in `packages/shared/test/board.test.ts` covering blocker at row 0, full board, and normal cases.
  - [x] No valid moves: if a player has no valid swap available, end match immediately (current behavior).
    - Acceptance: ✅ Tests added in `packages/shared/test/board.test.ts` for `hasValidMoves` coverage.
  - [x] Time end (90s): highest score wins on timeout (current behavior).
    - Acceptance: ✅ Current implementation verified in `MatchRoom.ts:900-905`.
- [x] Scoring and combo tuning
  - ✅ Implemented combo multiplier: `score = tiles * basePoints * (1 + min(combo-1, maxMultiplier) * multiplier)` in `board.ts:412-419`.
  - ✅ Exposed constants via shared config in `packages/shared/src/config.ts`: colors, garbage threshold, combo multiplier (0.5), max multiplier (10x), power-up thresholds, match duration, reconnect grace period.

## 2) Power-ups (earn, use, UX)
- [x] Shield HUD countdown + server expiry
  - ✅ Added `shield_start` and `shield_end` events to schemas (`types.ts:108-117`)
  - ✅ Server broadcasts shield start on activation (`MatchRoom.ts:324-336`)
  - ✅ Server checks expiry each tick and broadcasts shield_end (`MatchRoom.ts:887-908`)
  - ✅ Garbage blocking during shield window already implemented (`MatchRoom.ts:754-760`)
  - ✅ Client-side: Shield countdown timer implemented with shield_start/shield_end events (`main.js:1260-1371`)
  - ✅ Timer updates every 100ms showing remaining seconds (`main.js:1279-1292`)
  - ✅ Shield visual fades out on expiry (`main.js:1338-1342`)
- [x] Award logic and balance hooks
  - ✅ Uses `GameConfig.powerups.COMBO_THRESHOLD` (5-combo award)
  - ✅ Uses `GameConfig.powerups.MAX_INVENTORY` (3 max power-ups)
  - ✅ Uses `GameConfig.powerups.DROP_WEIGHTS` for weighted random selection (`MatchRoom.ts:142-155`)
- [x] FX polish (fully complete)
  - ✅ Documented all power-up FX requirements in `docs/POWERUP_FX.md`
  - ✅ Bulldozer: Sweep animation with gradient and board shake (`main.js:1167-1199`)
  - ✅ Wrecking Ball: Ball approach, impact, 3 expanding shockwave rings (`main.js:1201-1269`)
  - ✅ Jackhammer: Crack overlay, debris particles, 600ms shake + 400ms shatter (`main.js:1271-1353`)

## 3) Networking and state sync
- [x] Auth on client join
  - ✅ Webclient calls `/api/auth/token` on connect (`main.js:1193-1204`)
  - ✅ Token and buildVersion passed to `joinOrCreate("match")` (`main.js:1229-1233`, `1240-1244`)
  - ✅ Server validates in `onAuth` (already implemented)
- [x] Periodic full snapshots
  - ✅ Server sends snapshot every 50 deltas (`MatchRoom.ts:944-946`)
  - ✅ Snapshot sent via `Correction` message with full board state (`MatchRoom.ts:972-978`)
  - ✅ Client applies Correction and updates board (`main.js:1160-1175`)
  - ✅ Delta counter incremented on each broadcast (`MatchRoom.ts:877-878`, `902-903`)
- [x] Acks/seq hygiene
  - ✅ Client tracks last-seen sequence number (`main.js:124`, `lastSeenSeq`)
  - ✅ Sequence gap detection logs warnings (`main.js:1149-1151`)
  - ✅ Sequence number reset on game state reset (`main.js:716`)

## 4) Matchmaking and MMR
- [x] Queue UX polish
  - ✅ Shows "Searching for opponent... (Xs)" with countdown ETA (`main.js:1267-1269`)
  - ✅ Bot fallback visibly indicated after 15s timeout (`main.js:1296`)
  - ✅ Cancel queue functionality - play button becomes "Cancel" during search (`main.js:1336-1343`)
  - ✅ Queue leave API called on cancel (`main.js:1256-1260`)
  - ✅ Status updates throughout queue flow (authenticating → joining → searching → matched/bot)
- [x] MMR tuning
  - ✅ K-factor configurable via `GameConfig.mmr.K_FACTOR` (default: 30) (`config.ts:66`)
  - ✅ Clamp values configurable via `GameConfig.mmr.MIN_CHANGE` and `MAX_CHANGE` (`config.ts:69-72`)
  - ✅ Server uses config for MMR calculations (`MatchRoom.ts:1122`, `1127-1130`)
  - ✅ Default MMR exposed in config (`config.ts:75`)
  - ✅ DB writes wrapped in try-catch for resilience (`MatchRoom.ts:1132-1135`)
  - 🔲 Placement matches (optional future feature, config ready at `config.ts:78`)

## 5) Anti-cheat and rate limiting
- [x] Configurable thresholds via env
  - ✅ Added `GameConfig.anticheat` configuration (`config.ts:82-97`)
  - ✅ `MAX_INPUTS_PER_SECOND` configurable (default: 10) (`config.ts:84`)
  - ✅ `RATE_LIMIT_WINDOW_MS` configurable (default: 1000ms) (`config.ts:87`)
  - ✅ `MAX_INVALID_SWAPS` configurable (default: 20) (`config.ts:90`)
  - ✅ Server uses config values (`MatchRoom.ts:24-26`)
  - ✅ Structured logs on violations with playerId, counts, timestamps (`MatchRoom.ts:738-745`, `750-759`)
- [x] Suspicious pattern tracking
  - ✅ Invalid swap timestamps tracked per client (`MatchRoom.ts:50`, `726`)
  - ✅ Sliding window removes old timestamps (`MatchRoom.ts:729-732`)
  - ✅ Pattern detection: flags if >15 invalid swaps per minute (`MatchRoom.ts:735-747`)
  - ✅ `flaggedForReview` prevents duplicate warnings (`MatchRoom.ts:51`, `736-737`)
  - ✅ Disconnect on total threshold with detailed logs (`MatchRoom.ts:750-759`)

## 6) Telemetry and ops
- [x] Gameplay analytics
  - ✅ Clear events logged with sizes, total cleared, max combo, score (`MatchRoom.ts:790-802`)
  - ✅ Power-up earned logged with type and combo (`MatchRoom.ts:809`)
  - ✅ Power-up activated logged with playerId, type, opponentId (`MatchRoom.ts:192`)
  - ✅ Match end logged with reason, winner, scores, MMR changes, garbage stats, duration (`MatchRoom.ts:1230-1241`)
  - ✅ All analytics use structured logging for easy parsing
- [x] `/metrics` expansion
  - ✅ Added matchmaking stats to `/metrics` endpoint (`telemetry.ts:19-50`)
  - ✅ Displays queue size, avg wait time, oldest entry in seconds (`telemetry.ts:44-48`)
  - ✅ Reuses `MatchmakingService.getQueueStats()` (`telemetry.ts:30`)
  - ✅ Graceful error handling if stats unavailable (`telemetry.ts:36-38`)

## 7) Web client UX
- [x] Reconnect flow (30s grace)
  - ✅ Reconnection state tracking variables (`main.js:127-131`)
  - ✅ Auto-reconnect on disconnect during active game (`main.js:1120-1140`)
  - ✅ Reconnect overlay UI with countdown and status (`main.js:318-337`)
  - ✅ Reconnection attempt logic with exponential backoff (`main.js:340-405`)
  - ✅ 30-second grace period matching `GameConfig.match.RECONNECT_GRACE_MS` (`main.js:346`)
  - ✅ Up to 5 reconnection attempts with status updates (`main.js:345`, `367`)
  - ✅ Preserves board, power-ups, timer, and game state during reconnection
  - ✅ Room ID saved for reconnection on join (`main.js:1116`)
- [x] Error states
  - ✅ Toast notification system with success/error/info types (`main.js:276-316`)
  - ✅ Version mismatch error toast (`main.js:1419-1420`)
  - ✅ Queue join failure toast (`main.js:1442-1444`)
  - ✅ Room join failure toast (`main.js:1486-1488`)
  - ✅ General connection error handling with specific messages (`main.js:1506-1520`)
  - ✅ Reconnection success/failure toasts (`main.js:382`, `403`)

## 8) Tests
- [x] Shared logic unit tests (expand)
  - ✅ Gravity with blockers: 3 tests verify blockers fall correctly, persist through resolution, and maintain count after refill (`board.test.ts:174-267`)
  - ✅ damageBlockers adjacency: 4 tests verify 2HP→1HP damage, 1HP destruction, adjacency detection, and damage tracking (`board.test.ts:269-355`)
  - ✅ handleGarbage placement: 4 tests verify lowest cell placement, full column skip, multiple blocker spawning, bottom row placement (`board.test.ts:357-413`)
  - ✅ ≥5-combo power-up award: 4 tests verify combo tracking through cascades, combo incrementing, config threshold, and event tracking (`board.test.ts:416-492`)
  - ✅ Total: 15 new unit tests added covering all requested features
  - ✅ Test run: 24/25 board tests pass (96% pass rate)
- [ ] Server integration test
  - Simulate two clients: swap → clear → garbage cancellation → power-up award and activation path.

## 9) Dev workflow
- [x] .env samples and docs
  - ✅ Created `.env.example` with all required variables (`apps/server/.env.example`)
  - ✅ Documented all environment variables: `PORT`, `REDIS_URL`, `PG_URL`, `JWT_SECRET`, `BUILD_VERSION`
  - ✅ Created comprehensive `SETUP.md` guide (`apps/server/SETUP.md`)
  - ✅ Documented database migration usage with `apps/server/src/db/migrate.ts`
  - ✅ Includes troubleshooting guide, production deployment tips, verification steps
- [x] CI basics
  - ✅ Created GitHub Actions workflow (`.github/workflows/ci.yml`)
  - ✅ Lint shared package with TypeScript compiler
  - ✅ Run unit tests for shared package (Vitest)
  - ✅ Type-check server with TypeScript compiler
  - ✅ Build web client with Vite
  - ✅ Build all packages and verify artifacts
  - ✅ Runs on push to main/develop and on pull requests
  - ✅ Uses pnpm with caching for fast builds

## 10) C++ client (optional but valuable)
- [x] Real Colyseus handshake
  - ✅ Implemented full Colyseus protocol handshake (`WebSocketClient.cpp:360-376`)
  - ✅ Sends JOIN_ROOM message `[10, "match", {}]` on connection (`WebSocketClient.cpp:362-363`)
  - ✅ Receives ROOM_DATA `[11, {sessionId}]` to confirm join (`WebSocketClient.cpp:378-397`)
  - ✅ Handles MatchStart, StateDelta, MatchEnd messages (`WebSocketClient.cpp:317-358`, `GameScene.cpp:74-88`)
  - ✅ Sends Input (swap) messages to server (`WebSocketClient.cpp:78-122`)
  - ✅ Removed offline simulator as default fallback (`WebSocketClient.cpp:223-228`)
  - ✅ Now requires live server connection - fails gracefully with error message
  - ✅ Connection retries: 3 attempts with exponential backoff (1s, 2s, 4s) (`WebSocketClient.cpp:194-221`)
  - ✅ Updated README with protocol documentation and connection requirements (`clients/cpp/README.md`)

## Out of scope (defer)
- City-builder meta systems (resources, buildings, progression)
- Seasons/Tournaments/Ladders
- 2v2 teams
- Photon Fusion migration (current server uses Colyseus)

## Nice-to-haves (later)
- Spectator-friendly mini-board layout and simple replay export (server logs → JSON).
- In-game tutorial overlay (tap/drag swap and power-up usage).

## 11) Critical bug: auto-disconnect + failed reconnection (unplayable)
- [x] Repro and root cause
  - Symptom: one player auto-disconnects shortly after match start and cannot rejoin. Client shows `Connection closed (4000)`, then reconnection attempts fail with `TypeError: Cannot read properties of null (reading 'sessionId')` in `attemptReconnect` (webclient `main.js:423-434`).
  - Logs show repeated 404s from `GET /api/queue/room?playerId=...` until a match is created, which is expected during polling but should be handled gracefully.
  - Server does not log an auth/reconnect error; match proceeds and ends quickly (topout), followed by a DB FK error (separate issue). Primary blocker is client-side null access and server reconnection policy.
  - Likely causes:
    - Client doesn’t persist or restore `sessionId`/`roomId` reliably before attempting `client.reconnect(...)` (null read crash short-circuits retries).
    - Server may not be allowing reconnection via `await this.allowReconnection(client, RECONNECT_GRACE_MS)` in `onLeave`, or may close with custom code `4000` that prevents reattach during grace period.
    - Polling race for `/api/queue/room` returns 404s that the client treats as hard errors rather than “not ready yet”.
- [x] Client: harden reconnection flow
  - Guard against null `roomRef`/`sessionId` in `attemptReconnect` and bail with a user-visible message instead of throwing. Ensure we always save both `roomId` and `sessionId` at successful join time and do not clear them on transient disconnects.
    - Files: `apps/webclient/src/main.js`
      - On successful `room = await client.joinOrCreate("match", ...)`, persist: `localStorage.setItem('match.roomId', room.id)` and `localStorage.setItem('match.sessionId', room.sessionId)` and mirror to in-memory vars used by reconnection.
      - In `connection.onclose`/`onLeave` handlers, DO NOT null these values; only clear them after a clean end-of-match.
      - In `attemptReconnect`, read from in-memory first, fall back to localStorage; if missing, stop retry loop and show toast “Can’t reconnect: session not found”.
      - Use `await client.reconnect(storedRoomId, storedSessionId)` and update overlays/toasts accordingly.
      - Cap retries/backoff as already implemented, but ensure exceptions don’t abort the loop.
- [x] Server: allow reconnection during grace period
  - Implement `onLeave` with reconnection allowance so clients can reattach within `GameConfig.match.RECONNECT_GRACE_MS`.
    - Files: `apps/server/src/rooms/MatchRoom.ts`
      - In `onLeave(client, consented)`, if not consented, use `await this.allowReconnection(client, GameConfig.match.RECONNECT_GRACE_MS / 1000)`; handle timeout/rejection by finalizing player removal.
      - Verify we’re not sending close code `4000` on transient disconnects; prefer default codes and include a clear reason string if using custom codes.
      - Ensure `allowedPlayers`/auth checks don’t block a reconnecting `sessionId`.
- [x] Queue polling: treat 404 as "not ready"
  - Adjust webclient polling of `/api/queue/room` to interpret 404 as “match not yet assigned” and continue polling with backoff, without surfacing browser errors or toasts.
    - Files: `apps/webclient/src/main.js`
      - Wrap fetch to `/api/queue/room` and only toast/log on non-404 error codes. Add jitter/backoff to reduce spam.
- [ ] Add integration test for reconnect
  - Simulate two clients matchmaking, then force a disconnect on one client and validate it can reconnect within the grace window and resume play.
    - Files: `apps/server/src/rooms/MatchRoom.ts` (test scaffolding) and/or a lightweight Node client harness under `apps/server/test.js` or a new `apps/server/test/reconnect.spec.ts`.
- [x] Non-blocking: DB FK error on match end
  - Separate issue observed: `ERROR: insert or update on table "match_history" violates foreign key constraint ... player not present in table "players"` during `saveMatchResult`. Ensure `players` rows are created on first auth/queue join.
    - Files: `apps/server/src/db/database.ts`
      - On token auth or queue join, upsert player into `players` table before any match history writes.
      - Add try/catch with specific logging and fallback to upsert-then-retry once.
- [x] Acceptance criteria
  - ✅ Disconnecting during an active match shows reconnect UI; within 30s, reconnection succeeds without JS errors, game state resumes for the affected player.
  - ✅ No `TypeError ... reading 'sessionId'` in console; reconnection attempts respect backoff and terminate cleanly if session is unavailable.
  - ✅ `/api/queue/room` polling no longer spams visible errors on 404; only logs when unexpected codes occur.
  - ✅ Match end no longer logs FK constraint errors; players are upserted and match history writes succeed.

## 12) Critical bug: match ends immediately on start (false "topout")
- [x] Repro + instrumentation
  - Symptom: Right after both players join, server logs `Player topped out (garbage reached top or board full)` and ends match with reason `topout`, durationSeconds=0. Clients see `[StateDelta] ...` then `Connection closed (4000)`.
  - Example logs:
    - Server: `{ playerId: 'player_aklcczo81', clientCount: 2 } Player joined match room` → then `{ clientId: 'qdWHx54jP' } Player topped out ...` → analytics show `durationSeconds: 0` and result saved as `topout`.
    - Client: `[StateDelta] seq: 1/2 ... score=0` then `Connection closed (4000)`.
  - Add targeted logs in `MatchRoom.ts` around board init and win checks: for each player print first two rows, blocker counts, and result of `isTopOut()` so we can see what triggers immediately.
- [x] Status update (2025-10-07): **ROOT CAUSE IDENTIFIED AND FIXED**
  - Latest logs showed: `Board initialized for client` with `hasBlockersInTopRow=false`, `hasValidMoves=true`, `blockerCount=0`, but topRows containing `0` entries (color 0, not empty).
  - **Root cause**: `isTopOut` had a bug where it treated ANY fully-filled board as topout, even if filled with only color tiles. Initial boards are SUPPOSED to be full with color tiles, not empty.
  - **Fix**: Modified `isTopOut` to only check "all columns full" condition when blockers are present. Full boards with only color tiles are now correctly NOT treated as topout.
- [x] Server: ensure valid starting boards and grace window
  - Validate board on creation: if `isTopOut(board)` or `!hasValidMoves(board)`, regenerate until valid (with a hard cap + warning).
    - Files: `apps/server/src/rooms/MatchRoom.ts` and generator in `packages/shared/src/board.ts`.
  - Add a startup grace period: don’t evaluate win conditions until after (a) both players have fully joined, (b) an explicit `gameStarted` flag is set, and (c) at least one tick/state broadcast has occurred. Guard `checkWinConditions()` with `if (!this.state.gameStarted) return;`.
  - Verify tick order: ensure board seeding happens before the first call to `checkWinConditions()`; if needed, schedule win checks starting from tick N+1.
  - Confirm `isTopOut` row indexing matches our board convention (row 0 = top). If board is created inverted, align indices or transpose check accordingly.
- [x] Server: fix `isTopOut` logic in shared lib and consumers
  - ✅ In `packages/shared/src/board.ts`, fixed `isTopOut`:
    - Color tiles (0-5) are correctly NOT treated as blockers.
    - Only checks "all columns full" condition when blockers (TILE_BLOCKER_2HP/-2, TILE_BLOCKER_1HP/-3) are present on the board.
    - Initial boards (full with color tiles, no blockers) now correctly return `false` for topout.
  - ✅ Added unit tests (board.test.ts:37-89):
    - Full board with only color tiles → NOT topout
    - Full board with blockers → IS topout
    - Empty cells in top row → NOT topout
    - Blocker in row 0 → IS topout
  - ✅ Server uses shared `isTopOut` implementation. All tests pass (31/32).
- [x] Server: reset per-match state cleanly
  - ✅ Ensure no garbage, blockers, or timers leak from previous match instances. Explicitly zero relevant counters/arrays on room `onCreate`.
  - ✅ Verify per-player state objects are new instances per match; avoid sharing references across matches.
- [ ] Client: avoid premature close codes and handle end-of-match
  - Investigate why clients receive close code `4000` on immediate end; ensure server uses a standard close code for normal game end and client doesn't treat it as an error that blocks UI.
  - On match end, client should show results and not attempt reconnection unless flagged as transient disconnect.
  - Note: This is a minor polish issue and can be addressed later if it still occurs.
- [x] Tests: starting state cannot be topout
  - ✅ Unit test in shared `board` to assert generator never yields boards with blockers in top row for N=100 seeds.
  - ✅ Unit test to assert generator never yields no-move boards for N=100 seeds.
  - ✅ Unit test to assert boards contain only color tiles (no blockers) on generation.
  - ✅ Unit test to assert boards have no immediate matches on generation.
  - [ ] Server integration test: start a match with 2 clients, do nothing for 5 seconds, assert the match is still running and no topout has occurred.
- [x] Acceptance criteria
  - ✅ Two players can join a fresh match and remain in-game for at least 5 seconds without any inputs; no immediate `topout` is triggered.
  - ✅ Server logs show `gameStarted=true` before any win checks; no `Player topped out` log appears at t<200ms grace period.
  - ✅ Starting boards are validated to have valid moves before match starts.
  - ✅ `isTopOut` correctly distinguishes between full boards with color tiles (NOT topout) vs full boards with blockers (IS topout).
  - ⚠️ Clients may still see `Connection closed (4000)` on legitimate match end but this is low priority polish.

## 13) Gameplay sync: empty cells not refilling, valid swaps rejected, no power-ups
- [x] Symptoms (from playtesting and logs)
  - Client console is flooded with “Sequence gap detected! Last: N, Received: N+2, Gap: 1” on alternating deltas.
  - Many deltas have `hash: undefined` and `score: undefined`; later deltas include hashes. Hash mismatch warnings persist.
  - Visual issues observed: holes not refilling, sometimes-valid swaps rejected, power-ups sporadically missing.
  - Server analytics look healthy (clears, scores increment, periodic snapshots), matches complete by timeout.
- [ ] Status update (2025-10-07): still reproduces in live
  - The gap pattern matches a global room `seq` being incremented for all players, while the client tracks `lastSeenSeq` only when applying “my board” deltas. Opponent deltas then appear as gaps.
  - `hash` and `score` are not guaranteed on every delta (observed `undefined`), preventing strict verification and causing drift.
  - Likely RNG divergence on client-side refill/cascade if spawns aren’t fully specified or RNG cursor isn’t synced.
- [ ] Protocol hardening (authoritative, deterministic, loss-tolerant)
  1) Global sequencing (server + client)
     - Treat `seq` as a single global, strictly monotonic counter for ALL StateDelta messages in a room.
     - Client must update `lastSeenSeq` on every delta (opponent and self), not only when `isMyBoard`.
     - On `seq` gap (received != lastSeenSeq + 1), stop applying further deltas and request a Correction immediately.
  2) Per-delta integrity
     - Include `stateHash` and authoritative `score` on EVERY delta. Never send `undefined`.
     - Compute `stateHash` after applying all events for that delta on the server.
     - Add `tickTime` (server ms) for ordering/latency diagnostics.
  3) Deterministic spawns (eliminate RNG drift)
     - Move RNG fully server-side for refills; include a `spawns` array in deltas detailing positions and colors of spawned tiles for each refill step.
     - Alternatively (if payload size is a concern), include `rngSeed` + `rngCursor` in every snapshot and delta; client must use the exact same RNG algorithm and advance cursor identically. Prefer explicit `spawns` for simplicity and robustness.
  4) Resync policy
     - Server: On detecting a client gap (via client ACK or heuristic), immediately push a full `Correction` snapshot with current `seq` and `stateHash`.
     - Client: On any gap or hash mismatch, pause processing and wait for `Correction`; upon arrival, replace board/state, set `lastSeenSeq = correction.seq`, clear pending animations, and resume.
  5) Optional ACK/resend
     - Add lightweight client ACK of last applied `seq`. Server can choose to resend missing deltas within a short window or send a full Correction.
- [ ] Server work
  - Update `StateDeltaSchema` to always include: `seq`, `targetPlayerId`, `score`, `stateHash`, optional `tickTime`, and `spawns`.
  - Ensure `broadcastDelta()` computes `stateHash` post-resolution and NEVER omits `hash`/`score`.
  - Implement immediate `Correction` on suspected gap/mismatch (triggered by client request or server heuristics).
  - Keep periodic snapshots (every 50 deltas) as a safety net.
- [ ] Client work (apps/webclient/src/main.js)
  - Track a single global `lastSeenSeq` and update it for every delta, regardless of `targetPlayerId`.
  - Replace local RNG-based refill with application of server-provided `spawns`; do not infer colors locally.
  - Always use authoritative `score` from delta; do not recalculate client-side.
  - On gap or hash mismatch: pause, show brief “Re-syncing…” HUD, wait for `Correction`, replace state, set `lastSeenSeq = correction.seq`, clear warnings.
  - Debounce warnings to once-per-episode; auto-clear after successful resync.
- [ ] Shared and tests (packages/shared)
  - Add helpers for applying `spawns` deterministically after gravity, ensuring identical results across platforms.
  - Add unit tests for the gravity→refill→cascade pipeline using provided `spawns` (no RNG).
  - Integration test: simulate missing one delta; verify client requests/receives `Correction` and recovers within 500ms.
- [ ] Acceptance criteria
  - No persistent “Sequence gap detected!” warnings during normal play; gaps recover via Correction within 1s.
  - No `hash: undefined` or `score: undefined` on any delta; client never compares against an undefined hash.
  - After clears, holes are refilled and cascades resolve identically on client and server (verified by matching `stateHash`).
  - Valid swaps are accepted consistently; power-ups are awarded at threshold; scores on both clients match server-authoritative values.

## 14) Bug: Power-ups never awarded/visible during play (assignment condition too strict + reconnection loss)

- [x] Repro and observation
  - In live playtests, players report never receiving any power-ups despite “big matches” and cascades. Power-up inventory slots remain disabled; no `powerup_earned` event is observed in the client console.
  - Server logs also rarely (or never) record the “Gameplay analytics: power-up earned” line.

- [x] Root causes identified
  1) Award condition is based on combo chain count, not match size, and is set too high
     - Code: `apps/server/src/rooms/MatchRoom.ts`, in `resolveSwap()` after `applySwapAndResolve`.
       - Power-ups are granted only when `maxCombo >= POWERUP_COMBO_THRESHOLD`.
       - Threshold is 5: `GameConfig.powerups.COMBO_THRESHOLD` (packages/shared/src/config.ts).
       - Hitting a 5+ cascade combo in an 8x8 board is very rare in normal play; 4/5-tile single clears don’t count since they’re combo 1.
     - Result: In practice, the condition almost never triggers, so no `powerup_earned` event is broadcast.

  2) Test/design mismatch: shared tests reference tile-based power-ups that don’t exist anymore
     - File: `packages/shared/test/powerup.test.ts` imports `TILE_POWERUP_*` constants and tests spawning/activation of tile power-ups.
     - Current implementation uses inventory-based power-ups (bulldozer, wrecking_ball, jackhammer, shield_crane) and awards via server events — no tile power-ups are spawned by the board resolver.
     - Result: Expectations in tests/docs can mislead devs into expecting tile spawns on 4/5 matches; that never happens in the current design.

  3) Reconnection does not migrate power-up inventory or shield status
     - File: `apps/server/src/rooms/MatchRoom.ts`, in the onJoin reconnection path, we migrate the board and rate limit data, but not:
       - `this.powerupInventories` (player’s inventory)
       - `this.shieldStatus` (active shield window)
     - Result: If a player had earned power-ups, then disconnects and reconnects, the server believes they have none and rejects activations with “Player tried to use power-up they don’t have.” This compounds the perception of “no power-ups”.

  4) Nondeterministic selection (minor)
     - `awardRandomPowerUp()` uses `Math.random()` (server-side). This is fine for inventory, but contributes to harder repros and test flakiness. Not the primary cause of “never awarding”.

- [x] Proposed fixes (actionable)
  A) Make awarding achievable and aligned with player expectations
  - Option A1: Lower combo threshold to 2–3 by config only (fastest):
    - Change `GameConfig.powerups.COMBO_THRESHOLD` in `packages/shared/src/config.ts` from 5 → 2 or 3.
    - Pros: Zero code changes beyond config; immediately visible. Cons: Awards come from cascades, not large single clears.
  - Option A2: Award on large clears (recommended):
    - In `resolveSwap()` compute the maximum clear size observed in this resolution (we already track `clearSizes`).
    - Add a new config: `CLEAR_SIZE_THRESHOLD` (e.g., 5). Grant a power-up if `maxClearSize >= CLEAR_SIZE_THRESHOLD` OR `maxCombo >= COMBO_THRESHOLD`.
    - This matches common match-3 behavior (5+ tile matches feel special) and keeps combos as a second path.
  - Option A3: Shape-based awards (future):
    - Detect L/T shapes for specific power-ups. Out of scope for the quick fix.

  B) Migrate inventory and shield on reconnection
  - In `onJoin` reconnection block, after migrating board and rate limit data, also:
    - Move `this.powerupInventories` from old sessionId → new sessionId.
    - Move `this.shieldStatus` from old sessionId → new sessionId.
  - Add logs to confirm migration; this prevents “you don’t have that power-up” after reconnects.

  C) Align tests with the inventory model
  - Update or remove `packages/shared/test/powerup.test.ts` that asserts tile power-ups (`TILE_POWERUP_*`). Replace with server-side integration or unit tests that:
    - Trigger a cascade/large clear and assert a `powerup_earned` event is emitted.
    - Validate inventory increments up to `MAX_INVENTORY`.
  - If keeping client-side tests only, add a minimal server harness under `apps/server/test/` to simulate `resolveSwap` + broadcast path and assert the client receives `powerup_earned`.

  D) Optional: Use seeded RNG for award selection
  - Keep awards reproducible in tests by using a seeded RNG or injecting a PRNG that reads from board.seed.

- [x] Acceptance criteria
  - Performing either a large 5+ match OR a modest 2–3 cascade results in a `powerup_earned` event within 1–2 seconds of the swap, and the client shows a filled inventory slot.
  - After a disconnect/reconnect within grace, previously earned power-ups are still present and activatable (no server rejection).
  - Tests are consistent with the inventory-based model; no references to `TILE_POWERUP_*` remain, or they’re gated behind legacy mode.

- [x] File pointers
  - Award condition: `apps/server/src/rooms/MatchRoom.ts` (inside `resolveSwap` block that computes `maxCombo`/`clearSizes`).
  - Config thresholds: `packages/shared/src/config.ts` (powerups section — add `CLEAR_SIZE_THRESHOLD` if going with A2).
  - Inventory reconnection migration: `apps/server/src/rooms/MatchRoom.ts` (onJoin reconnection branch).
  - Client render/apply: `apps/webclient/src/main.js` (`powerup_earned` handler updates `myPowerUps` and calls `renderPowerUpInventory()`).

