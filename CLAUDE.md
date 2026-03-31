# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Monorepo Structure

pnpm workspaces with four packages: `frontend` (Vue 3 SPA), `backend` (Koa HTTP + WebSocket), `shared` (Zod schemas + constants), `document`. The `shared` package is consumed by both frontend and backend — changes to it require a rebuild (`pnpm --filter @putongoj/shared build`) before the other packages pick them up.

## Commands

All commands are run from the **package directory**, not the root, unless prefixed with `pnpm --filter`.

### Development (run all together)
```bash
# Start infrastructure first (MongoDB + Redis via Docker or local)
cd backend && pnpm dev       # Starts app (8008), ws (8009), updater, worker concurrently
cd frontend && pnpm dev      # Vite dev server with proxy to localhost:8008 / 8009
```

### Build
```bash
pnpm --filter @putongoj/shared build   # Must rebuild after shared changes
cd backend && pnpm build               # tsc + copies scripts to dist/
cd frontend && pnpm build              # Vite production build
```

### Lint & Typecheck
```bash
cd backend  && pnpm lint           # eslint (@antfu/eslint-config flat config)
cd frontend && pnpm lint
cd backend  && pnpm typecheck      # tsc --noEmit
cd frontend && pnpm typecheck      # vue-tsc --noEmit
```

### Backend Tests
```bash
cd backend && pnpm test            # pretest → nyc ava (serial) → posttest
```
Tests run against a real MongoDB/Redis (configured in `backend/.env.test`). `pretest.ts` seeds the database; `posttest.ts` cleans it. AVA runs serially (`"serial": true`). To run a single test file:
```bash
cd backend && npx ava test/controllers/contest.test.ts
```

## Environment Configuration

Backend uses **dotenv-flow**: reads `.env`, `.env.local`, `.env.{NODE_ENV}`, `.env.{NODE_ENV}.local` in that order. For local dev, create `backend/.env.development.local` with at minimum:
```
PTOJ_WEB_PORT=8008
PTOJ_MONGODB_URL=mongodb://127.0.0.1:27017/oj
PTOJ_REDIS_URL=redis://127.0.0.1:6379
PTOJ_SECRET_KEY=any-local-secret
```
For tests, use `backend/.env.test` (already tracked, points to a test DB).

Frontend dev proxy is configured via `frontend/.env.development.local`:
```
VITE_PROXY_API_TARGET=http://localhost:8008
VITE_PROXY_WS_TARGET=ws://localhost:8009
```

## Architecture

### Backend request flow
```
Koa middleware stack (app.ts):
  parseClientIp → setupAuditLog → koa-session → koaBody → staticServe
  → setupRequestContext → errorHandler → spaFallback → router (/api/*)

routes/index.ts          aggregates all route modules under /api
routes/{resource}.ts     defines paths + middleware (loginRequire, etc.)
controllers/{resource}.ts  business logic, Zod validation, response building
services/{resource}.ts   DB queries and reusable operations
models/{Resource}.ts     Mongoose schema + auto-incrementing IDs (via ID model)
policies/{resource}.ts   loadXxxState helpers that check access and return state objects
```

### Authorization pattern
Controllers call `loadXxxState(ctx)` from `policies/` which returns a state object (`{ isJury, accessible, ... }`). Controllers then gate logic on those fields rather than re-querying. `loadProfile(ctx)` returns the current user; `profile.isAdmin` / `profile.isRoot` are virtual properties.

### Shared package
`shared/src/consts/` — enums (UserPrivilege, Language, LabelingStyle, JudgeStatus, ParticipationStatus, …)
`shared/src/types/model/` — Zod schemas mirroring Mongoose models (used for validation + type inference)
`shared/src/types/api/` — request/response Zod schemas for every endpoint
`shared/src/types/codec.ts` — custom Zod codecs (e.g. `isoDatetimeToDate`, `stringToInt`)

Frontend imports types exclusively from `@putongoj/shared`; backend imports them from the local `shared/src` path (via tsconfig path alias `@/` → `shared/src/`).

### Frontend state & API
- **Pinia stores** in `store/modules/` wrap entity-level state (contest, problem, solution, …); `store/index.ts` is the root store (session, settings).
- **API layer** in `api/{resource}.ts` calls `instanceSafe` (throws on error) or `instance` (returns `{ success, data/message }`).
- **i18n**: all user-facing strings go through `t('ptoj.<key>')`. Keys live in `locales/en.json` and `locales/zh.json`, alphabetically sorted.

### Auto-incrementing IDs
MongoDB ObjectIds are internal. Entities exposed to users (Contest, Problem, Discussion, etc.) use numeric IDs managed by the `ID` model. The `ID` collection stores the current counter per entity name.

## Key Conventions

- **Response helpers**: `createEnvelopedResponse`, `createErrorResponse`, `createZodErrorResponse` in `backend/src/utils/response.ts` — always use these, never write raw `ctx.body`.
- **Audit logging**: `ctx.auditLog.info(...)` for all mutations.
- **Problem access**: `loadProblemState(ctx, pid, fromContestId?)` — pass `fromContestId` when checking access in a contest context.
- **Contest access**: `loadContestState(ctx)` returns `{ accessible, isJury, hasStarted, hasEnded, … }`. `isJury` = admin OR course `manageContest` role.
- **Course roles**: `CourseRole` is a per-member object: `{ basic, viewTestcase, viewSolution, manageProblem, manageContest, manageCourse }`. There is no separate "teacher" privilege level in `UserPrivilege`.
