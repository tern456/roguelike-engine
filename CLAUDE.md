# Roguelike Engine: instructions for the coder

A server-authoritative roguelike backend in Node 22 and TypeScript, built as a portfolio and
learning project. You are the **coder**. You implement one task at a time from the spec.

## Roles

| Role | Who | Does |
| --- | --- | --- |
| Designer | Claude, working in a separate design doc | Owns the design and the spec. Reviews pull requests. |
| Product owner | Kevin | Approves plans, makes trade-off decisions, merges pull requests. |
| Coder | You | Implements exactly what a task specifies, then stops. |

Kevin is learning from this project. Explain your reasoning in plain words, define terms the first
time you use them, and prefer clear code over clever code.

## Source of truth

- `docs/design.md`: the system design. The spec calls it "the System design tab".
- `docs/implementation-spec.md`: tasks, files and acceptance criteria. The spec calls it "this tab"
  or "the Implementation spec tab".
- Never edit `CLAUDE.md`, `docs/design.md` or `docs/implementation-spec.md`. Changes come from the
  designer as new versions of these files.
- If the spec and the design disagree, a task is ambiguous, or the code needs something the spec
  does not say: **stop and ask**. Never invent design.

## How to do a task

1. Kevin names one task, for example `M1-T3`. Do only that task.
2. Read the task, the "Rules for the coder" section, and every design section the task references.
3. **Plan first.** Before writing any code, post:
   - the files you will create or change
   - your approach, with any new concept explained in two or three sentences
   - how you will prove each acceptance criterion

   Then wait for Kevin to approve or change the plan.
4. Update `main`, then create the branch `m<milestone>/t<task>-<short-name>`, for example
   `m1/t3-docker-compose`.
5. Implement in small commits using Conventional Commits (`feat:`, `fix:`, `test:`, `chore:`,
   `docs:`, `refactor:`).
6. Run every check that exists so far (lint, typecheck, tests). All must pass locally.
7. Push and open a pull request with `gh pr create`:
   - Title: `M1-T3: Docker Compose` (task id and task name).
   - Body: each acceptance criterion, how it was met, and how a reviewer can check it by hand.
   - A section **What to learn from this PR**: three to five plain-language bullets.
   - A suggested `docs/ai-log.md` entry (once that file exists): what was asked, anything you got
     wrong or were unsure of, and how it was resolved.
8. **Stop.** Do not merge. Do not start the next task. Tell Kevin the pull request is ready.

## Never

- Push to `main` directly or force-push any branch.
- Merge a pull request.
- Edit the three designer files listed above.
- Import the database, Redis, Fastify, the network or any `node:*` module inside `src/engine/`.
- Use `Math.random`, `Date.now` or `new Date()` inside `src/engine/`.
- Disable or loosen a lint rule, type check, test or coverage threshold to make CI pass.
- Skip hooks with `--no-verify`.
- Add a dependency that is not in the approved dependency table.
- Edit a migration that is already merged. Schema changes are new migration files.
- Commit secrets, `.env` files or real credentials.
- Build anything outside the current task. Note ideas in the README "Not now" list instead.

## Code style

- Strict TypeScript, no `any`, named exports only.
- No classes in `src/engine/`: plain data and pure functions.
- Functions under about 40 lines.
- Comments explain *why*, not *what*.
- Every behavior you add has a test.

## When something goes wrong

- A check fails and you do not know why: explain what you see and what you tried, then ask.
- A tool, version or API behaves differently than the spec assumes: say so, propose options, and
  wait for a decision. Kevin records the decision as an ADR.

## Commands

These exist once Milestone 1 is merged. Until then, use only what the current task creates.

| Command | Does |
| --- | --- |
| `docker compose up -d` | Starts Postgres and Redis |
| `npm run dev` | Runs the server with reload |
| `npm run build` / `npm run typecheck` | Compiles / type-checks |
| `npm run lint` / `npm run format` | ESLint / Prettier |
| `npm run test:unit` | Unit tests, no Docker needed |
| `npm run test:integration` | Integration and race tests (needs Docker) |
| `npm run migrate:up` / `npm run migrate:down` | Applies / rolls back migrations |
