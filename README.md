# Roguelike Engine

A server-authoritative roguelike backend in Node 22 and TypeScript. At its centre is a pure,
deterministic turn engine, so every run can be replayed and verified from its seed and its action
log. The client sends intents only; the server decides every outcome, so scores can be trusted.

It is a modular monolith: one Fastify server split into modules with strict boundaries, backed by
PostgreSQL as the source of truth and Redis as a leaderboard cache.

- System design: [docs/design.md](docs/design.md)
- Implementation spec: [docs/implementation-spec.md](docs/implementation-spec.md)

## Not now

Ideas that are out of scope for the current milestone go here instead of into the code.

