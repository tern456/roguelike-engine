# Roguelike Backend Engine — System Design

Oct 23, 2026 · @Kevin

## Overview

We are building a server-authoritative roguelike backend in Node and TypeScript. At its centre is a pure, deterministic turn engine, so every run can be replayed and verified from its seed and its action log.

**Decisions so far**

| Decision | Choice | Why it matters |
| --- | --- | --- |
| Authority | Server-authoritative | The client sends intents only. The server decides every outcome, so scores can be trusted |
| Language | Node 22 LTS + TypeScript (strict mode) | One language front to back, with types shared between client and server |
| Components | Accounts and auth, run manager, turn engine, procedural generation, content registry, combat, inventory, meta-progression, event log and snapshots, leaderboards | Components 1–9 and 12 from the menu |
| Cross-cutting | Structured logging, metrics, concurrency tests | Makes the project read as production-minded |

**What this project should prove on a resume**

- Domain modeling: a game engine with clear boundaries, written as pure functions.
- Data design: a Postgres schema whose constraints do real work, plus event sourcing.
- Correctness under concurrency: races identified, prevented, and proven by tests.
- Operability: logs, metrics and health checks you can show in a demo.

**Example resume bullets (fill in your real numbers at the end)**

- Built a server-authoritative roguelike engine in TypeScript with seeded procedural generation and event-sourced runs that replay deterministically.
- Prevented double-submitted turns and duplicate rewards with optimistic locking and database constraints, verified by concurrent integration tests against real Postgres.
- Verified leaderboard scores by server-side replay and served rankings from Redis, with Postgres as source of truth; p95 turn latency of \[X\] ms at \[N\] concurrent players.

## Architecture

The system is a modular monolith: one deployable Fastify server split into modules with strict boundaries. The engine module is pure — no database, network, clock or Math.random — so it is deterministic and easy to test.

```text
              Web client (TypeScript)
                       |  HTTPS + JSON
+----------------------v-----------------------+
| API layer (Fastify): JWT auth, Zod, pino     |
+----------------------+-----------------------+
                       |
+----------------------v-----------------------+     +----------------------+
| Services: own all I/O and transactions       |     | Data                 |
|   Auth            Meta-progression           |---->|   PostgreSQL (truth) |
|   Runs            Leaderboard                |---->|   Redis (lb cache)   |
+------+---------------------------------------+     +----------------------+
       |  step(state, action)   (only Runs calls the engine)
+------v---------------------------------------+
| Engine: pure TypeScript, deterministic, no IO|
|   Turn reducer  Procgen  Combat              |
|   Inventory     Content registry             |
+----------------------------------------------+
```

Only the Runs service calls the engine, and no arrow leaves the engine: it never touches the database or the network.

**Why a modular monolith, not microservices:** one process keeps transactions simple and deployment cheap. The module boundaries still let you explain how each piece could become a service later, which makes a good interview conversation.

**The core idea: the engine is a reducer**

```ts
// engine/step.ts — nothing in this folder does I/O
export function step(state: RunState, action: Action, content: ContentBundle): StepResult {
  // 1. validate the action against the current state (throws InvalidAction)
  // 2. apply the player's action
  // 3. run monster turns until the player can act again
  // 4. return the new state and the domain events that happened
}

type StepResult = { state: RunState; events: DomainEvent[]; stateHash: string };
```

The RNG state lives inside `RunState`, so the same state and action always produce the same result.

**Module layout**

```text
src/
  platform/      db pool, migrations, logger, redis, config, error types
  modules/
    auth/        routes, service, token logic
    runs/        routes, run service (turn transaction), repository
    meta/        unlocks and lifetime stats
    leaderboard/ submission, verification, Redis cache
  engine/        PURE: state, step, ecs, procgen, combat, inventory, ai, rng
  content/       JSON definitions, Zod schemas, loader
  shared/        types shared with the frontend (Action, ClientView)
test/
  unit/          engine, combat, procgen (no database)
  integration/   routes against real Postgres via Testcontainers
  concurrency/   race tests (see Concurrency)
```

**Tech stack**

| Layer | Choice | Reason |
| --- | --- | --- |
| HTTP server | Fastify | Fast, schema-first, pino logging built in |
| Validation | Zod | One schema gives runtime checks and TypeScript types |
| Database | PostgreSQL 16 | Transactions, partial unique indexes, JSONB |
| Query layer | Kysely + plain SQL migrations | Type-safe but close to SQL, so you learn SQL itself |
| Cache | Redis sorted sets | Leaderboard reads |
| Passwords | argon2id | Current recommended password hash |
| Logging | pino (JSON) | Structured logs carrying request, player and run ids |
| Metrics | prom-client | Exposes /metrics for Prometheus |
| Tests | Vitest, Testcontainers, fast-check | Unit, real-database integration and property tests |
| Local environment | Docker Compose | Postgres and Redis with one command |

## Components

Each component below lists what it does, the key design choices, and what you learn by building it. Numbers match the original menu.

### 1. Accounts and auth

- **Does:** register, log in, refresh, log out; guards every other route.
- **Design:** argon2id password hashes. A short-lived JWT access token (15 minutes) plus a rotating refresh token, stored only as a SHA-256 hash. Reusing an old refresh token revokes the whole token family. Login is rate-limited per IP and username.
- **You learn:** token lifecycles, why raw tokens are never stored, and theft detection through rotation.

### 2. Run manager

- **Does:** creates runs, owns their status, enforces one active run per player.
- **Design:** a state machine: `active → dead | won | abandoned`, with no way back. The database enforces it through a partial unique index and conditional UPDATEs.
- **You learn:** state machines, and pushing invariants into the database instead of trusting app code.

### 3. Turn engine

- **Does:** turns a player action into a new state, then runs monster turns.
- **Design:** a pure `step()` reducer over Entity-Component-System state. Entities are ids; components (Position, Health, Inventory, AI, Speed) are plain data in maps. An energy scheduler decides who acts next, so fast monsters can act twice.
- **Anti-cheat:** the server sends a `ClientView` holding only what the player can see (field of view). The client never receives hidden state, so it cannot leak it.
- **You learn:** ECS, reducers, separating logic from I/O, and fog of war as a security measure.

### 4. Procedural generation

- **Does:** builds each floor's layout, monsters and loot from the run seed.
- **Design:** a hand-written seedable PRNG such as xoshiro128\*\*. Each floor's seed is `hash(runSeed, depth)`, with separate RNG streams for layout, spawns and combat, so a combat change never reshapes the dungeon.
- **Algorithm:** rooms by BSP split, corridors to join them, then a flood fill that rejects floors whose stairs are unreachable. Floors are regenerated from the seed, never stored.
- **You learn:** determinism, seeded RNG streams, and graph algorithms.

### 5. Content registry

- **Does:** holds every monster, item, ability, status effect and class as data.
- **Design:** JSON files in the repo, validated by Zod in CI, published as an immutable content version identified by a checksum. Each run is pinned to the version it started on, so a balance change never breaks replay of old runs.
- **You learn:** data-driven design and versioning, the same problem as API versioning.

### 6. Combat and rules

- **Does:** to-hit rolls, damage, death, and status effects such as poison and stun.
- **Design:** pure functions that take an RNG and two stat blocks and return events. Status effects are data (duration, per-tick effect) processed at the start of a turn. All formulas live in one file.
- **You learn:** testable business logic, and property tests such as "HP never goes below zero".

### 7. Inventory and equipment

- **Does:** pick up, drop, equip, unequip and use items, with capacity and slot rules.
- **Design:** part of engine state, changed only through actions. Rules: a capacity limit, one item per slot, and two-handed weapons block the off-hand. Inventory is saved inside snapshots, not its own table (the Schema section explains why).
- **You learn:** modeling rules as validations, and when not to normalize.

### 8. Meta-progression

- **Does:** permanent unlocks and lifetime stats across runs.
- **Design:** rewards are granted in the same transaction that ends the run. The `active → dead` change can succeed only once, so two racing requests cannot grant rewards twice.
- **You learn:** separating run data from account data, and exactly-once side effects.

### 9. Event log and snapshots

- **Does:** records every accepted action and lets the server rebuild any run.
- **Design:** the log stores inputs (actions), not outcomes; outcomes are recomputed because the engine is deterministic. Each event also stores the state hash after it, which pinpoints the exact turn where a determinism bug appears. A full snapshot every 50 turns makes loading fast: latest snapshot plus the events after it.
- **You learn:** event sourcing, the snapshot trade-off, and replay-based verification.

### 12. Leaderboards

- **Does:** global all-time, weekly, and per-seed rankings.
- **Design:** a finished run is replayed on the server from seed and actions. Its score enters only if the final state hash matches. Postgres keeps each player's best score per board; Redis sorted sets serve reads and can be rebuilt from Postgres at any time.
- **You learn:** caching with a clear source of truth, and why verification beats trust.

### Observability (cross-cutting)

- **Does:** shows what the server is doing and why something failed.
- **Design:** pino JSON logs where every line carries `requestId`, `playerId`, `runId` and `turn`. Metrics: turn latency histogram, 409 conflict count, replay verification failures, database pool usage. `/healthz` reports the process is up; `/readyz` checks Postgres and Redis.
- **You learn:** debugging from logs alone, and which numbers matter in production.

## Database schema

Postgres holds ten tables. Anything queried across runs (accounts, runs, unlocks, rankings) is relational. In-run state (map, monsters, inventory) lives in JSONB snapshots plus an append-only event log.

| Table | Component | Holds | Constraint doing the work |
| --- | --- | --- | --- |
| players | 1 Auth | Accounts | Unique case-insensitive username and email |
| refresh\_tokens | 1 Auth | Hashed refresh tokens, grouped in rotation families | Unique token hash |
| content\_versions | 5 Content | One row per published content bundle | Unique checksum |
| content\_defs | 5 Content | Monster, item, ability, status and class definitions | Primary key (version, kind, key) |
| runs | 2 Run manager | One row per run: seed, status, turn, score | Partial unique index: one active run per player |
| run\_events | 9 Event log | Every accepted action, in order | Primary key (run\_id, seq); unique idempotency key |
| run\_snapshots | 9 Snapshots | Full engine state every 50 turns | Primary key (run\_id, turn) |
| player\_unlocks | 8 Meta | Permanent unlocks | Primary key (player, unlock) makes awards idempotent |
| player\_stats | 8 Meta | Lifetime counters | One row per player |
| leaderboard\_entries | 12 Leaderboards | Best verified score per player per board | Primary key (board, player); upsert keeps the max |

**The big decision: run state as JSONB, not one row per monster or item**

The engine reads and writes the whole state on every turn. Normalizing it would mean dozens of row writes per turn and a data layer fighting the reducer. Nothing outside a run ever queries one goblin's HP.

The trade-off: you cannot run SQL queries inside an active run. If analytics needs that later, add a projection table filled from events. Being able to defend this choice is a strong interview moment.

**DDL**

```sql
CREATE EXTENSION IF NOT EXISTS citext;

-- ── 1. Accounts and auth ─────────────────────────────────
CREATE TABLE players (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  username       citext NOT NULL UNIQUE,
  email          citext NOT NULL UNIQUE,
  password_hash  text   NOT NULL,                 -- argon2id, never plain text
  created_at     timestamptz NOT NULL DEFAULT now(),
  last_login_at  timestamptz
);

CREATE TABLE refresh_tokens (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  player_id    uuid  NOT NULL REFERENCES players(id) ON DELETE CASCADE,
  family_id    uuid  NOT NULL,                    -- one login = one family
  token_hash   bytea NOT NULL UNIQUE,             -- SHA-256 of the token
  expires_at   timestamptz NOT NULL,
  revoked_at   timestamptz,
  replaced_by  uuid REFERENCES refresh_tokens(id),
  created_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX refresh_tokens_live ON refresh_tokens (player_id) WHERE revoked_at IS NULL;

-- ── 5. Content registry ──────────────────────────────────
CREATE TABLE content_versions (
  id            serial PRIMARY KEY,
  checksum      bytea NOT NULL UNIQUE,            -- hash of the whole bundle
  published_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TYPE content_kind AS ENUM ('monster', 'item', 'ability', 'status_effect', 'class');

CREATE TABLE content_defs (
  version_id  int          NOT NULL REFERENCES content_versions(id),
  kind        content_kind NOT NULL,
  key         text         NOT NULL,              -- 'goblin', 'iron_sword'
  data        jsonb        NOT NULL,              -- validated by Zod before insert
  PRIMARY KEY (version_id, kind, key)
);

-- ── 2. Runs ──────────────────────────────────────────────
CREATE TYPE run_status    AS ENUM ('active', 'dead', 'won', 'abandoned');
CREATE TYPE verify_status AS ENUM ('pending', 'verified', 'failed');

CREATE TABLE runs (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  player_id           uuid   NOT NULL REFERENCES players(id),
  seed                bigint NOT NULL,
  custom_seed         boolean NOT NULL DEFAULT false,  -- player-chosen seeds skip global boards
  content_version_id  int    NOT NULL REFERENCES content_versions(id),
  engine_version      text   NOT NULL,          -- code version needed to replay this run
  class_key           text   NOT NULL,
  status              run_status NOT NULL DEFAULT 'active',
  turn                int    NOT NULL DEFAULT 0, -- also the optimistic-lock version
  depth               int    NOT NULL DEFAULT 1,
  state_hash          bytea  NOT NULL,          -- hash of the current state
  score               int,
  verification        verify_status,            -- NULL while active
  started_at          timestamptz NOT NULL DEFAULT now(),
  last_action_at      timestamptz NOT NULL DEFAULT now(),
  ended_at            timestamptz,
  CHECK ((status = 'active') = (ended_at IS NULL)),
  CHECK (turn >= 0 AND depth >= 1)
);
-- One active run per player, enforced by the database, not app code
CREATE UNIQUE INDEX runs_one_active_per_player ON runs (player_id) WHERE status = 'active';
CREATE INDEX runs_by_player ON runs (player_id, started_at DESC);

-- ── 9. Event log and snapshots ───────────────────────────
CREATE TABLE run_events (
  run_id           uuid  NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  seq              int   NOT NULL,              -- the turn this action produced
  action           jsonb NOT NULL,              -- validated player intent
  idempotency_key  uuid  NOT NULL,
  state_hash       bytea NOT NULL,              -- state hash AFTER this action
  created_at       timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (run_id, seq),
  UNIQUE (run_id, idempotency_key)
);

CREATE TABLE run_snapshots (
  run_id      uuid  NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  turn        int   NOT NULL,                   -- 0, 50, 100, ...
  state       jsonb NOT NULL,                   -- entities, inventory, explored tiles, RNG state
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (run_id, turn)
);

-- ── 8. Meta-progression ──────────────────────────────────
CREATE TABLE player_unlocks (
  player_id      uuid NOT NULL REFERENCES players(id) ON DELETE CASCADE,
  unlock_key     text NOT NULL,                 -- 'class:rogue', 'item:lantern'
  source_run_id  uuid REFERENCES runs(id),
  unlocked_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (player_id, unlock_key)
);

CREATE TABLE player_stats (
  player_id      uuid PRIMARY KEY REFERENCES players(id) ON DELETE CASCADE,
  runs_started   int    NOT NULL DEFAULT 0,
  runs_won       int    NOT NULL DEFAULT 0,
  deepest_depth  int    NOT NULL DEFAULT 0,
  total_kills    bigint NOT NULL DEFAULT 0,
  updated_at     timestamptz NOT NULL DEFAULT now()
);

-- ── 12. Leaderboards ─────────────────────────────────────
CREATE TABLE leaderboard_entries (
  board        text NOT NULL,                   -- 'global', 'weekly:2026-W43', 'seed:8812734'
  player_id    uuid NOT NULL REFERENCES players(id) ON DELETE CASCADE,
  run_id       uuid NOT NULL REFERENCES runs(id),
  score        int  NOT NULL CHECK (score >= 0),
  achieved_at  timestamptz NOT NULL,
  PRIMARY KEY (board, player_id)                -- best score per player per board
);
CREATE INDEX leaderboard_rank ON leaderboard_entries (board, score DESC, achieved_at ASC);
```

**Details worth explaining in an interview**

- **Floors are not stored.** They are regenerated from `seed` and `depth`, so a snapshot holds only what changed: entities, inventory, explored tiles and RNG positions.
- **`engine_version` and `content_version_id`** pin each run to the code and data that can replay it. Without them, a balance patch would make old runs fail verification.
- **Ties on the leaderboard** go to whoever reached the score first (`achieved_at ASC`), and the rank index matches that order exactly.
- **Keyset pagination** on `(score, achieved_at)` instead of OFFSET keeps deep leaderboard pages fast.

## Concurrency

Every turn is one short transaction guarded by an optimistic lock on `runs.turn`. The database, not app code, settles every race.

**How a turn is processed**

1. Load the run state: latest snapshot plus the events after it (an in-memory cache keyed by run id and turn can skip this).
2. Run `step()` outside any transaction. It is pure, so throwing the result away costs nothing.
3. Commit the result in one short transaction:

```sql
BEGIN;
-- Claim the turn: the version check and the row lock happen in one statement
UPDATE runs
   SET turn = turn + 1, state_hash = $newHash, depth = $depth,
       status = $newStatus, ended_at = $endedAt, last_action_at = now()
 WHERE id = $runId AND player_id = $playerId
   AND status = 'active' AND turn = $expectedTurn
RETURNING turn;              -- 0 rows → ROLLBACK and answer 409 Conflict

-- Append the action; the primary key and idempotency key are a second guard
INSERT INTO run_events (run_id, seq, action, idempotency_key, state_hash)
VALUES ($runId, $expectedTurn + 1, $action, $key, $newHash);

-- Every 50 turns, in the same transaction
INSERT INTO run_snapshots (run_id, turn, state) VALUES ($runId, $turn, $state);

-- If the run just ended: grant unlocks, update stats, queue verification
COMMIT;
```

**Why optimistic, not `SELECT … FOR UPDATE`:** conflicts are rare (one player, one run), and the engine work happens outside the lock. Pessimistic locking would hold a row lock while game logic runs. Both are defensible; knowing why you chose one is the point.

**Races and their guards**

| Race | What goes wrong without a guard | Guard |
| --- | --- | --- |
| Two tabs or a double-click send the same turn | Both apply; the log forks | `turn = $expectedTurn` in the UPDATE; the loser gets 409 |
| Client retries after a timeout, but the first request committed | The action is applied twice | Idempotency key: on 409, a key already in `run_events` returns the committed result as 200 |
| Two "start run" requests at once | Two active runs | Partial unique index; Postgres error 23505 becomes 409 |
| A fatal move and an abandon arrive together | The run ends twice; rewards granted twice | Both need `status = 'active'`; rewards commit with the status change |
| Two runs submit scores at once | A lower score overwrites a higher one | `INSERT … ON CONFLICT DO UPDATE … WHERE excluded.score > leaderboard_entries.score` |
| Two tabs refresh with the same refresh token | Looks like theft; the family is revoked | Rotate with one conditional UPDATE; 10-second grace for the just-replaced token |
| Redis write fails after Postgres commits | Cached rank differs from the truth | Postgres is truth; write Redis after commit; a rebuild job fixes drift |

**Concurrency test plan**

| Test | How it runs | Must hold |
| --- | --- | --- |
| Parallel turn | 20 requests with the same `expectedTurn`, released together | Exactly one 200 and nineteen 409s; one new event; turn +1 |
| Retry storm | One idempotency key sent 10 times, in parallel and in sequence | One event; every response shows the same resulting turn |
| Parallel run start | 10 `POST /runs` at once for one player | One run created; nine 409s |
| End-of-run race | A fatal move and an abandon released together | One terminal status; unlocks and stats counted once |
| Score race | 10 verified runs with random scores, submitted at once | Stored score equals the maximum; Redis rank equals Postgres rank after rebuild |
| Refresh race | Two refreshes with one token, inside and outside the grace window | Inside: both succeed. Outside: the family is revoked |
| Crash mid-turn | An error thrown after the event insert, before COMMIT | No event, no turn change, no snapshot |
| Determinism | fast-check generates random seeds and action lists | Live final hash = full replay hash = snapshot + tail replay hash |

**Writing a race test**

```ts
it('accepts exactly one of 20 parallel turns', async () => {
  const run = await startRun(player);
  const send = () => api.post(`/runs/${run.id}/actions`, {
    expectedTurn: 0,
    idempotencyKey: randomUUID(),
    action: { type: 'wait' },
  });

  const results = await Promise.all(Array.from({ length: 20 }, send));

  expect(results.filter(r => r.status === 200)).toHaveLength(1);
  expect(results.filter(r => r.status === 409)).toHaveLength(19);
  expect(await countEvents(run.id)).toBe(1);
});
```

- **Use real Postgres** through Testcontainers. Mocks cannot race, so they hide exactly these bugs.
- **Size the connection pool** at least as large as the request count, or requests queue and never truly overlap.
- **Repeat each race test** 50 times in CI. Races are probabilistic, and one pass proves little.
- **Add a test-only barrier** that pauses each request after `step()` and releases them together. This forces maximum overlap at the commit.

## API

A small REST API over JSON is enough for a turn-based game; one request is one turn.

| Method | Path | Does | Notes |
| --- | --- | --- | --- |
| POST | /auth/register | Create an account | Rate-limited |
| POST | /auth/login | Return access and refresh tokens | Rate-limited per IP and username |
| POST | /auth/refresh | Rotate the refresh token | Reuse outside the grace window revokes the family |
| POST | /auth/logout | Revoke the token family |  |
| GET | /me | Profile, lifetime stats, unlocks |  |
| POST | /runs | Start a run: `{ classKey, seed? }` | 201 with the first ClientView; 409 if a run is active |
| GET | /runs/active | Resume the current run |  |
| GET | /runs/:id | Current ClientView | Visible tiles and entities only |
| POST | /runs/:id/actions | Take a turn: `{ expectedTurn, idempotencyKey, action }` | 200 new view and events; 409 stale turn; 422 illegal action |
| POST | /runs/:id/abandon | End the run |  |
| GET | /runs/:id/replay | Seed, versions and action list | Finished runs only |
| GET | /leaderboards/:board | Ranked entries | Keyset pagination with a cursor |
| GET | /leaderboards/:board/me | The caller's rank |  |
| GET | /content/:version | Names, glyphs, descriptions | Cacheable forever; versions are immutable |
| GET | /healthz, /readyz, /metrics | Health and Prometheus metrics |  |

**Actions the client can send**

```ts
type Dir = 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w' | 'nw';
type Slot = 'main_hand' | 'off_hand' | 'body' | 'ring';

type Action =
  | { type: 'move'; dir: Dir }            // moving into a monster attacks it
  | { type: 'wait' }
  | { type: 'pickup'; itemId: string }
  | { type: 'drop'; itemId: string }
  | { type: 'equip'; itemId: string }
  | { type: 'unequip'; slot: Slot }
  | { type: 'use'; itemId: string; targetId?: string }
  | { type: 'descend' };
```

The union is defined once as a Zod schema in `shared/`, so the frontend and backend cannot disagree. Every error has one shape: `{ error: { code, message, requestId } }`, and the `requestId` matches the server log line.

## Guardrails

Tooling enforces the rules below, so the project stays clean without relying on memory. A second set covers how we build it: Claude designs, an AI coding agent implements, and you direct, review and understand every part.

**Engineering guardrails**

| Guardrail | Enforced by | Protects |
| --- | --- | --- |
| `engine/` cannot import the database, Redis, Fastify or `node:fs` | ESLint `no-restricted-imports` (or dependency-cruiser) in CI | The pure-engine architecture |
| No `Math.random`, `Date.now` or `new Date()` inside `engine/` | ESLint `no-restricted-globals` and `no-restricted-syntax` | Deterministic replay |
| Every route validates input with Zod | Route registration helper that requires a schema | No unvalidated data reaches a service |
| TypeScript `strict` and `noUncheckedIndexedAccess`; no `any` | `tsconfig.json` and typescript-eslint | Type safety |
| Consistent formatting | Prettier, run by a husky + lint-staged pre-commit hook | Readable diffs |
| `main` is protected; CI must pass to merge | GitHub branch protection | A working `main` at all times |
| 90% line coverage on `engine/` | Vitest coverage threshold in CI | The core logic stays tested |
| Determinism test and race suite are required checks | CI required status checks | Replay and concurrency correctness |
| No secrets in the repo | `.env.example` only; gitleaks in CI | Credentials |
| Vulnerable dependencies are flagged | Dependabot | Supply-chain security |
| Each big decision is written down | One ADR per decision in `docs/adr/` | The reasoning behind the design |
| Scope stays fixed per milestone | Definition of done (Build order) and a "not now" list in the README | Finishing the project |

**How we work: roles**

Claude is the designer and does not write production code. A separate coder implements from the Implementation spec tab. You own the decisions and must be able to explain every part, because an interviewer will ask.

| Role | Who | Does |
| --- | --- | --- |
| Designer | Claude | Writes and maintains this doc and the Implementation spec. Explains every design decision. Reviews pull requests against the spec and explains the code. |
| Product owner and reviewer | You | Approves each spec before coding starts, makes trade-off decisions, merges pull requests. |
| Coder | An AI coding agent (such as Claude Code) or a person | Implements one task per pull request, exactly as specified. Stops and asks when the spec is unclear. |

**The loop for every milestone**

1. **Design.** Claude writes the milestone's spec: tasks, files, and acceptance criteria. It explains the spec to you in plain words.
2. **Approve.** You ask questions until every part is clear, then approve or change it. Trade-offs come as options; your choice goes into an ADR.
3. **Build.** The coder implements one task per branch and pull request, following the rules at the top of the Implementation spec.
4. **Review.** You and Claude check each pull request against its acceptance criteria. Claude explains the code line by line where it matters.
5. **Teach-back.** At the end of the milestone you explain it back in your own words; Claude points out gaps. This doubles as interview practice.
6. **Log.** `docs/ai-log.md` records key prompts, what the AI got wrong, and what you learned.

Design changes go through the doc first. The coder never invents design: if the code needs something the spec does not say, the spec is fixed, then the code.

Task-level instructions for the coder live in `docs/implementation-spec.md`.

## Build order

Build the engine before the server. Each milestone ends with something you can demo and a test that proves it.

1. **Skeleton.** Fastify with strict TypeScript, Docker Compose for Postgres and Redis, SQL migrations, pino logging, `/healthz` and `/readyz`, and Vitest with Testcontainers in GitHub Actions.
   - *Learn:* project setup and CI.
   - *Done when:* CI is green on a test that talks to a real database.
2. **Engine offline.** PRNG, RunState, ECS, move and wait, BSP floors, field of view, and a terminal renderer for debugging. No database yet.
   - *Learn:* determinism and algorithms.
   - *Done when:* the same seed prints the same dungeon every time.
3. **Content and combat.** Zod-validated JSON content, combat formulas, status effects, chase AI and the energy scheduler.
   - *Learn:* data-driven design and property testing.
   - *Done when:* a scripted fight gives the same result on every run.
4. **Accounts.** Registration, login, rotating refresh tokens and rate limiting.
   - *Learn:* authentication done properly.
   - *Done when:* a test shows token reuse revokes the family.
5. **Runs on the server.** The runs, run\_events and run\_snapshots tables, the turn transaction, idempotency keys, and every concurrency test. This is the milestone to talk about in interviews.
   - *Learn:* transactions, optimistic locking, event sourcing.
   - *Done when:* the race suite passes 50 times in a row.
6. **Inventory and meta-progression.** Items, equipment slots, the end-of-run transaction, unlocks and stats.
   - *Learn:* exactly-once side effects.
   - *Done when:* the end-of-run race test grants rewards once.
7. **Verification and leaderboards.** Server replay, score submission, the Postgres upsert, the Redis cache and its rebuild job.
   - *Learn:* caching with a source of truth.
   - *Done when:* a tampered action log fails verification and never ranks.
8. **Frontend and proof.** A small canvas web client, a README with the architecture diagram, and a k6 load test.
   - *Learn:* presenting your work.
   - *Done when:* the resume bullets carry real latency and concurrency numbers.

## Learning goals

Tick a box only when you can explain it out loud, without notes, in about a minute. Each one is a likely interview question. The teach-back at the end of each milestone checks these.

### Every milestone

- [ ] Turn a design into a task spec an AI coding agent can follow without guessing
- [ ] Review a pull request against its acceptance criteria, and say what you checked
- [ ] Name one thing the AI got wrong and how it was caught (record it in the AI log)
- [ ] Write an ADR: the decision, the alternatives, and why this one won

### Milestone 1: Skeleton

- [ ] Why the app is built by `buildApp()` separately from `server.ts`, and how that makes it testable
- [ ] The difference between `/healthz` (is the process alive?) and `/readyz` (can it reach its databases?)
- [ ] How a request id lets you trace one request through the logs
- [ ] What a migration is, and why a merged migration is never edited
- [ ] Why tests run against real Postgres in containers instead of mocks
- [ ] What CI and branch protection guarantee about `main`
- [ ] Why secrets live in environment variables, and what gitleaks catches

### Milestone 2: Engine offline

- [ ] What makes a function pure, and why the engine has to be
- [ ] How a seeded random number generator works, and why the same seed gives the same game
- [ ] Why there are separate RNG streams for layout, spawns, combat and AI
- [ ] Why floors are regenerated from the seed instead of stored
- [ ] How BSP splits a map into rooms, and how the flood fill rejects broken floors
- [ ] What property-based testing is, and what the determinism test proves
- [ ] Why run state must be plain JSON (no Map, Set or classes)

### Milestone 3: Content and combat

- [ ] What data-driven design means, and why adding a monster needs no code change
- [ ] Why each run is pinned to a content version, and what breaks without it
- [ ] How the energy scheduler lets fast monsters act twice
- [ ] How monster AI finds a path to the player
- [ ] Which combat properties are tested, and why property tests suit formulas

### Milestone 4: Accounts

- [ ] Why passwords use argon2id and not a plain hash like SHA-256
- [ ] Why access tokens are short-lived and refresh tokens are long-lived
- [ ] How refresh-token rotation detects a stolen token
- [ ] Why the database stores token hashes, never raw tokens
- [ ] What a timing attack on login is, and how the dummy hash prevents it
- [ ] Why the refresh token sits in an `httpOnly` cookie and not in localStorage

### Milestone 5: Runs on the server (the interview milestone)

- [ ] What a transaction guarantees, and what happens if the server crashes mid-turn
- [ ] Optimistic versus pessimistic locking, and why this project chose optimistic
- [ ] How idempotency keys make retries safe
- [ ] How a partial unique index enforces one active run per player
- [ ] Event sourcing: why we store actions, not outcomes
- [ ] The snapshot trade-off: storage versus load time
- [ ] Why `ClientView` hides what the player cannot see (server-authoritative anti-cheat)
- [ ] How the race tests force requests to overlap, and why they run 50 times

### Milestone 6: Inventory and meta-progression

- [ ] How rewards are granted exactly once, even when two requests race
- [ ] What `ON CONFLICT DO NOTHING` does, and why it makes unlocks idempotent
- [ ] Derived versus stored data, using equipment bonuses as the example
- [ ] Why run data lives in snapshots while account data lives in tables

### Milestone 7: Verification and leaderboards

- [ ] How server-side replay catches a cheated score
- [ ] What "source of truth" means for Postgres versus the Redis cache
- [ ] Why Redis is written after the Postgres commit, and how a rebuild fixes drift
- [ ] How Redis sorted sets rank players, and how ties are broken
- [ ] How `FOR UPDATE SKIP LOCKED` turns a table into a job queue
- [ ] Why keyset pagination beats `OFFSET` on deep pages

### Milestone 8: Frontend and proof

- [ ] How the client recovers from a 409 conflict
- [ ] Why the access token lives only in memory
- [ ] What p50 and p95 latency mean, and why percentiles beat averages
- [ ] How to read the load-test results and find the bottleneck
- [ ] Walk someone through the whole architecture in five minutes, using the diagram
