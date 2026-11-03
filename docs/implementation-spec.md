# Implementation spec

The coder works from this tab, one task per pull request. A task is done only when every acceptance criterion holds and CI is green.

## Rules for the coder

These rules apply to every task. If you are an AI coding agent, treat this section as your standing instructions.

1. **The System design tab is the source of truth.** Read it before the first task. If this spec and the design disagree, or anything is unclear, stop and ask. Never invent design.
2. **One task, one branch, one pull request.** Branch names follow `m<milestone>/t<task>-<short-name>`, for example `m1/t3-docker-compose`.
3. **Commit messages follow Conventional Commits:** `feat:`, `fix:`, `test:`, `chore:`, `docs:`, `refactor:`.
4. **Every pull request includes tests** for the behavior it adds. CI must be green before review.
5. **The pull request description** lists each acceptance criterion, how it was met, and how a reviewer can check it by hand.
6. **Only approved dependencies.** Anything not in the dependency table needs approval first. Pin exact versions (`save-exact=true` in `.npmrc`).
7. **Never edit a merged migration.** Schema changes are new migration files.
8. **Follow the guardrails** in the System design tab. Do not disable a lint rule to make CI pass; ask instead.
9. **Code style:** named exports only, no `any`, no classes in `src/engine/` (plain data and functions), functions under about 40 lines.
10. **Explain as you go.** Comment the why, not the what. The pull request should be readable by someone learning the codebase.
11. **Out of scope means out of scope.** Note ideas in the README "not now" list instead of building them.

## Environment

**Prerequisites on the developer machine:** Node 22 LTS, npm, Docker Desktop (or another Docker engine), Git, and optionally the GitHub CLI.

**Commands** (all defined as npm scripts in Milestone 1)

| Command | Does |
| --- | --- |
| `docker compose up -d` | Starts Postgres and Redis |
| `npm run dev` | Runs the server with reload (tsx watch) |
| `npm run build` | Compiles TypeScript to `dist/` |
| `npm start` | Runs the compiled server |
| `npm run typecheck` | `tsc --noEmit` |
| `npm run lint` / `npm run format` | ESLint / Prettier |
| `npm test` | All tests |
| `npm run test:unit` | Unit tests only; no Docker needed |
| `npm run test:integration` | Integration and race tests against Testcontainers |
| `npm run migrate:up` / `migrate:down` | Applies or rolls back migrations |
| `npm run db:reset` | Drops, recreates and migrates the local database |
| `npm run play` | Terminal client for the engine (Milestone 2) |

**Environment variables** (validated at startup; listed in `.env.example`)

| Variable | Example | Used by |
| --- | --- | --- |
| `NODE_ENV` | `development` | Everything |
| `PORT` | `3000` | Server |
| `LOG_LEVEL` | `info` | Logger |
| `DATABASE_URL` | `postgres://rogue:rogue@localhost:5432/rogue` | Database pool, migrations |
| `REDIS_URL` | `redis://localhost:6379` | Leaderboard cache |
| `JWT_SECRET` | 32+ random bytes, base64 | Access tokens (Milestone 4) |

**Approved dependencies**

| Package | Purpose | From milestone |
| --- | --- | --- |
| `fastify`, `fastify-type-provider-zod` | HTTP server with Zod route schemas | 1 |
| `zod` | Validation and types | 1 |
| `kysely`, `pg` | Type-safe queries over the Postgres driver | 1 |
| `node-pg-migrate` | Runs plain `.sql` migration files | 1 |
| `ioredis` | Redis client | 1 |
| `prom-client` | Prometheus metrics | 1 |
| `pino` | Logger options and redaction (Fastify uses it underneath) | 1 |
| `@noble/hashes` | Pure-JS SHA-256; the only hashing allowed inside `src/engine/` | 2 |
| `argon2` | Password hashing | 4 |
| `jose` | JWT signing and verification | 4 |
| `@fastify/rate-limit` | Login rate limiting | 4 |
| Dev: `typescript`, `tsx` | Compile and run. TypeScript is pinned to the latest 6.0.x release until typescript-eslint supports 7 (ADR 0004) | 1 |
| Dev: `@types/node` | Types for `process.env` and `node:*` modules | 1 |
| Dev: `@types/pg` | Types for the `pg` driver, which ships none | 1 |
| Dev: `pino-pretty` | Readable log output in development only | 1 |
| Dev: `vitest`, `@vitest/coverage-v8` | Tests and coverage | 1 |
| Dev: `testcontainers`, `@testcontainers/postgresql`, `@testcontainers/redis` | Real databases in tests | 1 |
| Dev: `fast-check` | Property tests | 2 |
| Dev: `eslint`, `@eslint/js`, `typescript-eslint`, `prettier`, `eslint-config-prettier` | Lint and format | 1 |
| Dev: `husky`, `lint-staged` | Pre-commit hook | 1 |
| Frontend (Milestone 8): `vite` | Dev server and bundler for the web client | 8 |

**Type packages are pre-approved:** an `@types/*` package for an approved package that ships no types of its own may be added as a dev dependency without asking. Name it in the pull request description.

Tools outside npm: k6 for load testing (Milestone 8), gitleaks and Dependabot in CI (Milestone 1).

## Milestone 1: Skeleton

Goal: a repo that builds, lints, tests against real Postgres and Redis, and runs CI on every push. No game code yet.

### M1-T1 Repository setup

**Before this task (done once, directly on `main`):** the initial commit holds only the designer's files: `CLAUDE.md` at the root, `docs/design.md` and `docs/implementation-spec.md`. An empty repo has no `main` to open a pull request against, so this is the only commit that skips the branch-and-PR rule. The coder never edits these three files; design changes come from the designer as new exports.

- **Create:** `package.json` (`"type": "module"`, `"engines": { "node": ">=22" }`), `.nvmrc` (`22`), `.npmrc` (`save-exact=true`), `.gitignore` (node\_modules, dist, coverage, .env), `.editorconfig`, MIT `LICENSE`, `README.md` with a project summary and an empty "Not now" list.
- **`tsconfig.json`:** `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `module` and `moduleResolution` `NodeNext`, `target` `ES2023`, `outDir` `dist`, `rootDir` `src`.
- **Folders:** create the `src/` layout from the System design tab, with an empty `index.ts` in each module.
- **Acceptance:** `npm run build` and `npm run typecheck` pass with zero errors.

### M1-T2 Lint, format and pre-commit hook

- **ESLint flat config** with `typescript-eslint` strict-type-checked presets and `eslint-config-prettier`. Global rule: `@typescript-eslint/no-explicit-any: error`.
- **Engine purity rules,** scoped to `src/engine/**`:
  - `no-restricted-imports` blocking `pg`, `kysely`, `ioredis`, `fastify`, any `node:*` module, and paths into `src/platform/` or `src/modules/`.
  - `no-restricted-properties` blocking `Math.random` and `Date.now`.
  - `no-restricted-syntax` blocking `new Date()`.
- **Prettier:** single quotes, semicolons, print width 100.
- **husky + lint-staged:** pre-commit runs ESLint with `--fix` and Prettier on staged files.
- **Acceptance:**
  - A file in `src/engine/` that imports `pg` fails `npm run lint`.
  - `Math.random()` or `new Date()` in `src/engine/` fails `npm run lint`.
  - The same code in `src/modules/` passes.
  - Committing a badly formatted file formats it automatically.

**TypeScript configs (also part of M1-T2):** `tsconfig.json` type-checks everything (`src/`, `test/`, `scripts/` and root `*.config.ts` files) with `noEmit`. `tsconfig.build.json` extends it with `rootDir: src`, `outDir: dist` and `include: ["src"]`. `npm run build` uses the build config; `npm run typecheck` uses `tsconfig.json`. ESLint's type-aware rules use `parserOptions.projectService`, so every linted file belongs to a config. Extra acceptance: a type error in a file under `test/` fails `npm run typecheck`.

### M1-T3 Docker Compose

- **`docker-compose.yml`:** `postgres:16` (user, password and database all `rogue`, port 5432, named volume) and `redis:7` (port 6379). Both have healthchecks (`pg_isready`, `redis-cli ping`).
- **`.env.example`** with every variable from the Environment table.
- **Acceptance:** `docker compose up -d` shows both services healthy within 30 seconds.

### M1-T4 Config and logger

- **`src/platform/config.ts`:** parses `process.env` with a Zod schema and exports a typed, frozen `config` object. On failure it prints every missing or invalid variable and exits with code 1.
- **`src/platform/logger.ts`:** pino options with level from config, pretty output only in development, and redaction of `req.headers.authorization`, `*.password` and `*.refreshToken`.
- **Acceptance:** starting without `DATABASE_URL` exits with a message naming `DATABASE_URL`. A logged object with a `password` field shows `[Redacted]`.

### M1-T5 Fastify app

- **`src/app.ts`** exports `buildApp(deps)`, which returns a configured Fastify instance without listening, so tests can use `app.inject()`. `deps` holds the database, Redis client and config.
- **`src/server.ts`** builds the app and listens on `PORT`.
- **Request ids:** `genReqId` returns a UUID. Each response carries it in an `x-request-id` header, and every log line includes it.
- **Error handler:** all errors return `{ error: { code, message, requestId } }`. Zod failures give 400 `VALIDATION_ERROR`; unknown errors give 500 `INTERNAL` with no stack trace in the body.
- **Routes:**
  - `GET /healthz` returns 200 `{ status: 'ok' }` without touching dependencies.
  - `GET /readyz` runs `SELECT 1` and Redis `PING` with a 1-second timeout each. It returns 200 when both pass, or 503 naming the one that failed.
  - `GET /metrics` returns prom-client default metrics.
- **Graceful shutdown:** on SIGTERM or SIGINT, stop accepting requests, finish in-flight ones, close the pool and Redis, then exit.
- **Acceptance:** `/readyz` returns 503 naming Redis when Redis is stopped. A thrown error returns the standard error shape with a matching `x-request-id`.

### M1-T6 Database and first migration

- **`src/platform/db.ts`:** creates a `pg.Pool` from `DATABASE_URL` and a Kysely instance. Database types live in `src/platform/db-types.ts`, written by hand to match the schema.
- **`migrations/0001_initial_schema.sql`:** the full DDL from the System design tab, with an "up" and a "down" part. Down drops everything in reverse order.
- **Acceptance:** on an empty database, `migrate:up` creates all ten tables. `migrate:down` then `migrate:up` succeeds. Inserting a second active run for one player fails with error 23505.

### M1-T7 Test harness

- **Vitest projects:** `unit` (`test/unit/**`, no Docker) and `integration` (`test/integration/**` and `test/concurrency/**`).
- **`test/integration/global-setup.ts`:** starts one Postgres and one Redis Testcontainer per test run, runs migrations, and exposes their URLs through environment variables.
- **`test/helpers/db.ts`:** `truncateAll()` empties every table between tests (`TRUNCATE … RESTART IDENTITY CASCADE`).
- **Coverage:** v8 provider; threshold of 90% lines for `src/engine/**`, enforced from Milestone 2.
- **Acceptance:** an integration test calls `/readyz` through `app.inject()` against the containers and gets 200. `npm run test:unit` passes with Docker stopped.

### M1-T8 CI and repository settings

- **`.github/workflows/ci.yml`** runs on every push and pull request, on `ubuntu-latest` (Docker is available there): checkout, setup-node 22 with npm cache, `npm ci`, lint, typecheck, unit tests, integration tests, and a coverage report uploaded as an artifact.
- **Separate job:** gitleaks secret scan.
- **`.github/dependabot.yml`:** weekly updates for npm and GitHub Actions.
- **Repository settings** (the product owner does this in GitHub): protect `main`, require pull requests, and require the CI and gitleaks checks.
- **Acceptance:** a pull request with a failing test cannot be merged. A committed fake AWS key fails the gitleaks job.

### M1-T9 Decision records and logs

- **`docs/adr/0000-template.md`:** sections for Context, Decision, Alternatives considered, and Consequences.
- **ADRs to write now:** `0001-modular-monolith.md`, `0002-pure-deterministic-engine.md`, `0003-kysely-and-sql-migrations.md`, `0004-typescript-6-pin.md`. Content comes from the System design tab.
- **`docs/ai-log.md`:** a dated entry format with the prompt, what happened, what was wrong, and what was learned.
- **Acceptance:** the files exist and the README links to them.

**Milestone 1 is done when** CI is green on `main`, all nine tasks are merged, and you can explain to Claude what each file in the repo is for.

## Milestone 2: Engine offline

Goal: a player can walk a generated dungeon in the terminal, and the same seed always gives the same game. Everything lives in `src/engine/` with no I/O, plus one dev script outside it.

### M2-T1 Seeded random numbers

- **`src/engine/rng.ts`** implements xoshiro128\*\*. Its state is four unsigned 32-bit numbers in a plain array, so it serializes to JSON.
- **Functions** (pure; each returns the value and the new state): `nextU32`, `nextFloat` (range 0 to 1), `int(min, max)` (inclusive, without modulo bias), `pick(array)`, `shuffle(array)`.
- **Stream seeding:** a stream's state is the first 16 bytes of `sha256("<runSeed>:<depth>:<stream>")`, read as four 32-bit numbers. All-zero state is replaced by a fixed constant.
- **Streams:** `layout` and `spawn` are derived fresh per floor and never stored. `combat` and `ai` are derived once at run start (depth 0) and stored in `RunState`.
- **Acceptance:** a known-answer test fixes the first 10 outputs for seed `42`. `int(1, 6)` over 60,000 draws gives each face between 9,500 and 10,500 times.

### M2-T2 Hashing and canonical JSON

- **`src/engine/hash.ts`:** `canonicalJson(value)` serializes with object keys sorted at every level. `stateHash(state)` returns the hex SHA-256 of it, using `@noble/hashes`.
- **Acceptance:** two objects with the same content but different key order hash identically.

### M2-T3 Run state

- **`src/engine/state.ts`** defines the state. It must be plain JSON: no `Map`, `Set`, class instances, `undefined` values or `bigint`.

```ts
export type EntityId = number;
export type RngState = [number, number, number, number];

export interface RunState {
  schemaVersion: 1;
  seed: string;                 // 64-bit seed as a decimal string (JSON-safe)
  depth: number;                // starts at 1
  turn: number;                 // starts at 0
  status: 'active' | 'dead' | 'won';
  rng: { combat: RngState; ai: RngState };
  nextEntityId: number;
  playerId: EntityId;
  components: {
    position: Record<EntityId, { x: number; y: number }>;
    renderable: Record<EntityId, { glyph: string; name: string }>;
    health: Record<EntityId, { hp: number; max: number }>;
    // Milestone 3 adds: actor, stats, ai, statusEffects
    // Milestone 6 adds: inventory, equipment, item
  };
  explored: string;             // base64 bitset, one bit per tile of the current floor
  messages: string[];           // newest last, capped at 20
}
```

- **`newRun(seed)`** builds depth 1, places the player in the start room, and sets HP to 20 of 20 (a placeholder until classes arrive in Milestone 3).
- **Acceptance:** `JSON.parse(JSON.stringify(state))` deep-equals `state` for a new run.

### M2-T4 Floor generation

- **`src/engine/procgen/floor.ts`** exports `generateFloor(seed, depth)`, returning `{ width: 80, height: 21, tiles, rooms, start, stairs }`. Tiles use `'#'` wall, `'.'` floor and `'>'` stairs down.
- **Floors are never stored.** `getFloor(seed, depth)` memoizes the last few floors in memory; this is allowed because the result is deterministic.
- **BSP algorithm:**
  1. Split the map recursively, alternating mostly horizontal and vertical cuts chosen by the `layout` stream. Stop when a leaf is under 10 by 8.
  2. Carve one room per leaf: random size from 4 by 3 up to the leaf size minus 2, at a random position inside it.
  3. Join sibling subtrees with an L-shaped corridor between room centres.
  4. Start = centre of a random room. Stairs = the floor tile farthest from the start by breadth-first search.
  5. Validate with a flood fill from the start: every floor tile reachable. On failure, keep drawing from the same stream and retry, up to 10 times, then throw.
- **Acceptance:** 1,000 seeds at depths 1 to 5 all generate valid floors with at least 4 rooms. A snapshot test fixes the ASCII output for seed `42`, depth 1.

### M2-T5 Field of view

- **`src/engine/fov.ts`:** symmetric shadowcasting, radius 8, walls block sight. Returns visible tiles as a sorted array of tile indexes.
- **`explored`** gains every visible tile after each turn and resets on a new floor.
- **Acceptance:** hand-made test maps show that a wall hides tiles behind it, a corridor limits the view to its line, and visibility is symmetric (A sees B exactly when B sees A).

### M2-T6 The step function

- **`src/engine/step.ts`:** `step(state, action) → { state, events, stateHash }`. It never mutates its input.
- **Milestone 2 actions:** `move` (eight directions), `wait`, `descend`.
- **Rules:**
  - Moving into a wall is illegal.
  - A diagonal move between two walls (corner cutting) is illegal.
  - `descend` is legal only on the stairs. It generates the next floor and places the player at its start.
  - Every legal action increments `turn`.
- **Illegal actions** throw `InvalidActionError` with a code: `BLOCKED`, `NOT_ON_STAIRS`, `RUN_OVER`, `UNKNOWN_ACTION`. The state is unchanged.
- **Events** for this milestone: `Moved`, `Waited`, `Descended`.
- **Acceptance:** unit tests cover each rule and error code. The input state is deep-frozen in tests, so any mutation throws.

### M2-T7 Terminal client (dev tool)

- **`scripts/play.ts`** sits outside `src/engine/`, so it may use `node:readline`. Run it with `npm run play -- --seed 123`.
- **Display:** visible tiles in normal colour, explored but not visible tiles dimmed, unexplored blank, `@` for the player. A status line shows seed, depth, turn and HP.
- **Keys:** arrows or `hjklyubn` to move, `.` to wait, `>` to descend, `q` to quit.
- **Acceptance:** the product owner can play through 3 floors.

### M2-T8 Determinism tests

- **Property test with fast-check:** for 200 random seeds and random sequences of up to 300 legal actions, replaying the same sequence from `newRun(seed)` gives the same final `stateHash`.
- **Snapshot simulation:** at a random turn, serialize the state to JSON, parse it back, and continue. The final hash must match the uninterrupted run.
- **Guard test:** a test that scans `src/engine/` for `Math.random`, `Date`, and `node:` imports, as a backstop to the lint rule.
- **ADR:** `0005-seeded-rng-streams.md` explains the per-floor and per-run streams.
- **Acceptance:** all tests pass and `src/engine/` line coverage is at least 90%.

**Milestone 2 is done when** the same seed prints the same dungeon every time, the determinism suite passes, and you can explain why floors are regenerated rather than stored.

## Milestones 3 to 8

These milestones have their tasks, key rules and acceptance criteria fixed. Before each one starts, Claude expands it to the same level of detail as Milestones 1 and 2 and you approve it. Earlier milestones always teach something that changes later details, so specs written too early go stale.

### Milestone 3: Content and combat

- **M3-T1 Content schemas.** JSON files in `content/` (monsters, items, status effects, classes, unlocks), validated by Zod schemas in `src/content/schemas.ts`. A monster has: key, name, glyph, hp, attack, defense, damage `[min, max]`, speed, xp, depth range `[min, max]`, and AI kind `chase` or `wander`. Starter content: 6 monsters, 10 items, 2 status effects (poison, stun), 2 classes (warrior by default, rogue unlockable).
- **M3-T2 Content bundle.** The loader builds a `ContentBundle`, and `step` becomes `step(state, action, content)`. Checksum = SHA-256 of the canonical JSON. `npm run content:publish` inserts into `content_versions` and `content_defs`, doing nothing if the checksum already exists.
- **M3-T3 New components.** `actor { energy, speed }`, `stats { attack, defense, damage }`, `ai { kind, lastSeenPlayerAt? }`, `statusEffects [{ key, turnsLeft }]`. Add `stats { xpEarned, kills, deepestDepth }` to `RunState` for scoring.
- **M3-T4 Energy scheduler.** Each tick, every actor gains energy equal to its speed and acts at 100 or more, paying 100. Normal speed is 100. Ties go to the lower entity id. After the player acts, ticks continue until the player can act again.
- **M3-T5 Combat.** Moving into a monster attacks it. Hit chance = clamp(70 + 5 × (attack − defense), 5, 95) percent. Damage = a roll within the weapon range, minus half the defender's defense rounded down, with a minimum of 1. All rolls use the `combat` stream. These numbers are starting values, kept in `src/engine/combat/formulas.ts`.
- **M3-T6 Status effects.** Processed when the affected entity acts: poison deals 1 damage, stun skips the action. `turnsLeft` counts down and the effect ends at zero.
- **M3-T7 Monster AI** (uses the `ai` stream). If the monster can see the player, it steps along the shortest path toward them. Otherwise it heads to where it last saw the player, or wanders.
- **M3-T8 Spawns and winning.** Each floor gets 3 + depth monsters, chosen by depth range from the `spawn` stream, never in the start room. Descending the stairs on depth 10 wins the run.
- **Acceptance:** property tests show HP never drops below 0, hit chance stays within 5–95%, and every hit deals at least 1 damage. A scripted fight with a fixed seed produces the same message log every time. Stun skips exactly the right number of turns. Two entities never share a tile. Invalid content fails CI.

### Milestone 4: Accounts

- **M4-T1 Register.** Username 3–20 characters of `a-z`, `0-9`, `_`; valid email; password 10–128 characters, hashed with argon2id. Errors: 409 `USERNAME_TAKEN` or `EMAIL_TAKEN`. Also creates the `player_stats` row.
- **M4-T2 Login.** Wrong username and wrong password both return 401 `INVALID_CREDENTIALS`. An unknown username still runs argon2 against a dummy hash, so response timing does not reveal which usernames exist.
- **M4-T3 Access tokens.** jose, HS256, claims `sub` (player id), `iss` (`roguelike`) and `exp` (15 minutes). A `requireAuth` hook sets `request.playerId` or returns 401 `UNAUTHENTICATED`.
- **M4-T4 Refresh tokens.** 32 random bytes, base64url, sent in an `httpOnly`, `Secure`, `SameSite=Strict` cookie scoped to `/auth`. The database stores only the SHA-256 hash; tokens expire after 30 days. Rotation revokes the old token and issues a new one in one conditional UPDATE. Reuse within 10 seconds of rotation issues a fresh token in the same family. Later reuse revokes the whole family and returns 401 `TOKEN_REUSED`.
- **M4-T5 Logout and rate limits.** Logout revokes the family. Login is limited to 5 attempts per minute per IP and username; registration to 3 per minute per IP.
- **M4-T6 `GET /me`.** Returns profile, stats and unlocks.
- **Acceptance:** the refresh race test from the System design tab passes. Passwords and tokens never appear in logs (a test captures log output and checks). Reusing a token after the grace window revokes the family.

### Milestone 5: Runs on the server

- **M5-T1 Repository.** All run SQL lives in `src/modules/runs/repo.ts`.
- **M5-T2 Start a run.** The seed is a random positive 63-bit integer, unless the player supplies one (then `custom_seed = true`). The class must be unlocked. One transaction inserts the run and the turn-0 snapshot and increments `runs_started`. Error 23505 becomes 409 `ACTIVE_RUN_EXISTS`.
- **M5-T3 Load state.** Load the latest snapshot, then replay the events after it. Keep recent states in an in-memory LRU cache (500 entries) keyed by run id and turn. If the replayed hash differs from `runs.state_hash`, log an error and return 500 `STATE_CORRUPT`.
- **M5-T4 Take a turn.** Exactly the transaction in the System design tab's Concurrency section. Check the idempotency key first: a known key returns 200 with the current view. Another player's run returns 404, so run ids are not revealed. A stale turn returns 409 `STALE_TURN` with `currentTurn`. An illegal action returns 422 with the engine's error code. Snapshot every 50 turns.
- **M5-T5 Client view.** The response shape is defined below. It contains only what the player can see.

```ts
export interface ClientView {
  runId: string;
  turn: number;
  depth: number;
  status: 'active' | 'dead' | 'won' | 'abandoned';
  map: {
    width: number;
    height: number;
    rows: string[];      // explored tiles as glyphs; unexplored tiles are spaces
    visible: string;     // base64 bitset of tiles visible this turn
  };
  entities: { id: number; glyph: string; name: string; x: number; y: number; hp?: number; maxHp?: number }[]; // visible only
  player: { hp: number; maxHp: number; attack: number; defense: number; statusEffects: string[] };
  messages: string[];    // the last 5
}
```

- **M5-T6 Other routes.** `GET /runs/active`, `GET /runs/:id`, `POST /runs/:id/abandon`.
- **M5-T7 Concurrency suite.** Parallel turn, retry storm, parallel run start, and crash mid-turn, from the test plan in the System design tab.
- **M5-T8 Metrics.** Histogram `turn_duration_seconds`, counter `turn_conflicts_total`, gauges for database pool use.
- **ADRs:** `0006-optimistic-locking.md`, `0007-jsonb-run-state.md`.
- **Acceptance:** the race suite passes 50 times in a row. A monster behind a wall never appears in `ClientView`.

### Milestone 6: Inventory and meta-progression

- **M6-T1 Item actions.** `pickup`, `drop`, `equip`, `unequip`, `use`. Capacity 12. Slots: main hand, off hand, body, ring. A two-handed weapon fills the main hand and blocks the off hand. Item randomness uses the `combat` stream.
- **M6-T2 Derived stats.** Equipment bonuses are computed when stats are read, never stored twice.
- **M6-T3 Item spawns.** 2 + ⌊depth ÷ 2⌋ items per floor, from the `spawn` stream.
- **M6-T4 Score.** Score = XP earned + 100 × deepest depth + 1,000 if won. A pure function `score(state)` in the engine computes it.
- **M6-T5 End of run.** When a step ends the run, the same turn transaction sets status, `ended_at`, `score` and `verification = 'pending'`, updates `player_stats`, and inserts unlocks with `ON CONFLICT DO NOTHING`. Unlock rules are content, for example `{ key: 'class:rogue', rule: { type: 'reachDepth', depth: 5 } }`. Abandoned runs update stats but grant no unlocks.
- **Acceptance:** the end-of-run race test grants rewards exactly once. Every slot and capacity rule has a unit test.

### Milestone 7: Verification and leaderboards

- **M7-T1 Verification worker.** Runs in-process every 5 seconds. It claims pending runs with `FOR UPDATE SKIP LOCKED`, 10 at a time, and replays each from `newRun(seed)` with its pinned content version. Matching final hash and score marks it `verified`; anything else marks it `failed` and logs the reason. A run from a different `engine_version` fails with `ENGINE_VERSION_MISMATCH`.
- **M7-T2 Submission.** In the same transaction as `verified`, upsert into boards: `global` and `weekly:<ISO week>` for random seeds, plus `seed:<seed>` for every run. The upsert keeps only a higher score.
- **M7-T3 Redis cache.** After commit, `ZADD lb:<board>` with member = player id and value = score × 10^10 + (9,999,999,999 − end time in epoch seconds). Earlier achievement wins ties. Scores are capped at 899,999 so the value stays exact in a double.
- **M7-T4 Read routes.** `GET /leaderboards/:board?limit=50&cursor=<rank>` and `GET /leaderboards/:board/me` read Redis. A missing key triggers a rebuild from Postgres first.
- **M7-T5 Rebuild and replay.** `npm run leaderboard:rebuild` rebuilds all boards. `GET /runs/:id/replay` returns seed, versions and actions for finished runs.
- **ADR:** `0008-replay-verification.md`.
- **Acceptance:** an action edited directly in the database makes the run fail verification, and it never ranks. The score race test passes. Redis ranks equal Postgres ranks after a rebuild.

### Milestone 8: Frontend and proof

- **M8-T1 Web client.** `client/` with Vite and TypeScript, importing types from `src/shared/`. Screens: log in or register, choose a class and start, play, end-of-run summary, leaderboard.
- **M8-T2 Renderer.** A canvas tile grid drawing glyphs, with explored tiles dimmed. Same keys as the terminal client. Each action sends `expectedTurn` and a fresh idempotency key; on 409 the client refetches the view.
- **M8-T3 Tokens.** The access token lives in memory only; the refresh cookie renews it automatically.
- **M8-T4 Load test.** A k6 script registers players, starts runs and takes turns for 2 minutes at 10, 50 and 100 virtual users. Results (p50 and p95 turn latency, error rate) go in `docs/performance.md`.
- **M8-T5 README.** Architecture diagram, setup steps, ADR index, testing approach, performance results, and a link to the AI log.
- **Acceptance:** a stranger can clone the repo, follow the README, and play within 10 minutes. The resume bullets carry the measured numbers.
