# Plan F — Local dev server (`ai-worker dev`): in-process workers + hot reload

**Status:** ✅ CODED (2026-07-09) — Phases 1–3 complete + Phase 4 mostly (colored prefixes, /dlq,
env-parity warnings done; boilerplate script + docs pending: DOCS-4 in EASY_AI_TASKS, H34 in
DEV_TASKS). Live-tested end-to-end on examples/root incl. restart-while-parked HITL resume and
hot reload. Review flags R48–R50 open in DEV_TASKS.md. One intentional deviation: handler errors
recorded as `failed` in the job store are NOT redelivered locally (matches deployed idempotency
behavior — SQS redelivery no-ops on terminal jobs); the local DLQ catches pre-store failures
(module-load errors etc.). **Benefits from Plan D** (reuses the env-file
cascade with stage `dev`); no dependency on Plan E.
**Goal:** run and test workers locally with ZERO AWS and ZERO microfox platform — the pillar of
"this project works without the microfox architecture". One process, one port: an HTTP server
exposing the same core-group surface (trigger/config/docs/queue-start), workers invoked
**in-process** (direct module invocation — NOT serverless-offline, NOT spawned node scripts,
NOT pm2), an **in-memory queue** standing in for SQS, an **in-memory + file-persisted job store**
standing in for Upstash/Mongo, and **hot reload without server restarts** (changed worker modules
are re-imported on their next invocation; in-flight jobs and parked HITL state survive edits).

Explicitly OUT of scope (agreed): schedule EXECUTION (schedules are printed at startup, not run),
worker_threads isolation (documented as a possible later `--isolate` v2), serverless-offline /
Lambda-emulation mode, local MongoDB/Redis installs.

Touches `ai-router/packages/ai-worker` (runtime: dispatch bridge + local store) and
`ai-router/packages/ai-worker-cli` (the `dev` command). Nothing in cicd/Foxhub.

---

## Current state (verified in code, 2026-07-08)

- **No local runtime exists.** The only way to run a worker is deploy-to-AWS.
- **Runtime dispatch is SQS-hardwired** (`ai-worker/src/handler.ts`): `dispatchWorker` →
  `resolveQueueUrlForWorker` (line ~808: env `WORKER_QUEUE_URL_<SANITIZED_ID>`, else
  `GetQueueUrlCommand` on the convention name) → `SQSClient.SendMessageCommand` (~892). Hard
  error when no queue URL resolves (~949).
- **Workers are Lambda-wrapped**: `createLambdaHandler` (~1013) takes `(SQSEvent, LambdaContext)`,
  parses `SQSMessageBody` ({workerId, jobId, input, context, webhookUrl, metadata, userId,
  maxTokens}), does idempotency checks against the job store, logs the `[AIWORKER_RUN]` marker.
  → the dev server can invoke workers by constructing SQSEvent-shaped records; the wrapper and
  everything inside it (queue orchestration via `wrapHandlerForQueue`, HITL parking, retries,
  token budget) runs UNCHANGED.
- **Job stores**: `WORKER_DATABASE_TYPE` selects `upstash-redis` (default) or `mongodb`
  (`redisJobStore.ts` / `mongoJobStore.ts`, plus `queueJobStore.ts` for queue docs). Both need
  cloud credentials — the gap for standalone dev.
- **Core-group HTTP surface** is generated as code strings in `compile.ts`
  (`generateTriggerHandler`, `generateWorkersConfigHandler`, `generateDocsHandler`,
  `generateQueueHandler`) — routes: `POST /workers/trigger`, `GET /workers/config`,
  `GET /docs.json`, `POST /queues/{id}/start`. The dev server reimplements these natively
  (small, well-understood shapes) rather than executing the generated Lambda strings.
- **Worker discovery**: `scanWorkers(aiPath)` + queue discovery already exist in the CLI; reusable.

## Target state

### 1. Runtime: local dispatch bridge (`@microfox/ai-worker`)

A seam in `handler.ts`, strictly inert in production:

```ts
// set by the dev server before any worker code loads
globalThis.__AI_WORKER_LOCAL_BRIDGE__ = {
  enqueue(workerId, sqsMessageBody, delaySeconds) { ... }
};
```

- In `dispatchWorker` (and the queue runtime's next-step send): if the bridge global exists,
  hand the message to it instead of resolving/sending SQS. `delaySeconds` honored via `setTimeout`
  (matches SQS DelaySeconds semantics; cap logging at the SQS 900s max for parity warnings).
- A global (not an env flag alone) keeps the runtime dependency-free and impossible to trip in
  Lambda (nothing sets the global there). Guarded additionally by `process.env.AI_WORKER_LOCAL`.

### 2. Runtime: local job store (`WORKER_DATABASE_TYPE=local`)

- New `localJobStore.ts` implementing the same surface as the redis/mongo stores (JobStore +
  queue-doc functions used by `queueJobStore.ts`): in-memory `Map`s + debounced JSON persistence
  to `.microfox/dev-state.json` (gitignored) so parked HITL queues and job history survive a dev
  server restart. Selected when `WORKER_DATABASE_TYPE=local` (dev server's default when no
  upstash/mongo env is present; devs can still point at real Upstash/Mongo via env — unchanged).
- Prod is untouched: `local` is never a fallback outside the dev command.

### 3. CLI: `ai-worker dev` command (new `commands/dev.ts` + `dev/` module dir)

- **One HTTP server** (hono — light, fetch-shaped, works with plain node http) on `--port`
  (default 4100), exposing the deployed-core-compatible surface:
  `POST /workers/trigger`, `GET /workers/config`, `GET /docs.json`, `POST /queues/{id}/start`,
  plus dev conveniences: `GET /jobs/{jobId}` (run tree, same shape as the boilerplate queue-job
  route), `GET /health`. Same request/response shapes as prod so existing clients
  (`triggerWorker`, boilerplate `useWorkflowJob`, curl scripts) work by swapping the base URL.
  Auth: honor `WORKERS_API_KEY` if set, open otherwise (dev default) — same rule the trigger
  handler applies.
- **In-memory queue + invoker**: per-worker FIFO, concurrency limit (default 5, `--concurrency`),
  constructs `SQSEvent` records (single-record batches) + a fake `LambdaContext`
  (`awsRequestId = randomUUID()`), calls the worker's wrapped handler directly. Failure →
  re-enqueue up to `maxReceiveCount` (default 3) then a local DLQ list (printed + queryable via
  `GET /dlq`). try/catch around every invocation so one worker's crash never kills the server.
- **Worker registry**: reuse `scanWorkers`/queue discovery against `--ai-path` (default `app/ai`).
  TS loaded directly via `jiti` — no esbuild bundling step in dev.
- **Env**: Plan D's `loadEnvFiles('dev')` cascade (`.env` → `.env.local` → `.env.dev` →
  `.env.dev.local`) + the `env` config include/exclude, hydrated into `process.env` before
  loading worker modules. `ENVIRONMENT/STAGE/NODE_ENV=dev`. Print the same missing-env warnings
  compile prints (parity: what would fail on deploy fails loudly in dev).
- **Schedules**: printed at startup with a "not executed locally" note. Nothing else (agreed).
- **Logs**: unified stdout, per-worker colored prefix, keep the real `[AIWORKER_RUN]` /
  `[Worker]` markers (log shapes match prod, greppable the same way).

### 4. Hot reload (no restarts)

- **chokidar** watches `aiPath` + each worker's local import graph (jiti exposes resolved deps;
  fall back to watching the whole project source dir minus node_modules).
- On change: invalidate the jiti cache for the changed file **and its dependents**; bump a
  registry generation counter. The invoker resolves the handler module **per invocation** →
  next run of any affected worker imports fresh code. The server process, in-memory queues,
  in-flight jobs, and parked HITL steps all survive. (This is the key DX win: edit a step-2
  worker WHILE step 1 runs; the queue picks up new code when it advances.)
- Topology changes (worker file added/removed, `microfox.config.ts` changed) → re-scan the
  registry in-place; log added/removed workers. Config-schema-level changes never require killing
  the process; a plain `rs` stdin shortcut does a full manual re-scan anyway (nodemon habit).

## Design decisions

- **In-process direct invocation, NOT serverless-offline**: serverless-offline runs handlers
  in-process anyway (its value-add is API-Gateway event mocking, which our runtime wrapper makes
  irrelevant), doesn't emulate SQS (would need ElasticMQ + one process/port per group), and
  can't do per-invocation module reload. Multi-group needs NO extra ports here: groups are just
  registry metadata locally; all workers share the one server.
- **NOT pm2 / spawned scripts**: no daemon, no per-process log scatter, no IPC; crash isolation
  is handled by try/catch at the invocation boundary (accepted dev trade-off, documented).
- **Fidelity limits documented up front** (README section): shared event loop (CPU-bound work
  blocks other runs), shared process.env/globals (group-env isolation not enforced locally),
  exactly-once in-memory delivery (SQS is at-least-once). Local dev = functional correctness;
  a real AWS stage (Plan E) = concurrency/infra fidelity. `--isolate` (worker_threads) is the
  named future answer if these bite.
- **Standalone-first**: no projectId, no microfox.json, no login required. The dev server must
  run in a bare repo with only workers + a `.env`.

## Phases

1. **Runtime seams** (`@microfox/ai-worker`): local bridge global in dispatch + queue next-step
   send; `localJobStore` (jobs + queue docs) with file persistence. Publishable alone (inert).
2. **`ai-worker dev` v1**: registry scan, hono server with the 4 core routes + /jobs/{id},
   in-memory queue/invoker with retries + DLQ, env cascade, schedule printout. Static modules
   (restart-on-change via watcher → full registry reload) to make it usable early.
3. **Hot reload proper**: jiti cache invalidation with dependent tracking, per-invocation module
   resolution, topology re-scan, `rs` shortcut.
4. **Polish + docs**: colored log prefixes, `GET /dlq`, boilerplate `package.json` script
   (`"dev:workers": "ai-worker dev"`), docs page (also covers fidelity limits), sync-boilerplate.

## Review flags (Anti-Vibe)

- ⚠ **Phase 1 touches the PRODUCTION dispatch path** (`handler.ts`). The bridge check must be a
  cheap, impossible-to-trip guard (global existence + env flag). Manual review before publishing
  `@microfox/ai-worker`; regression-test a real deployed dispatch chain after publish.
- ⚠ New `local` job store must never be silently selected in a deployed Lambda (only the dev
  command sets it; compile keeps rejecting/ignoring it for env.json unless explicitly set —
  review the `WORKER_DATABASE_TYPE` handling in `filterDepsForJobStore`/`getJobStoreType`).
- ⚠ `jiti` + `hono` + `chokidar` are new CLI dependencies — license/size sanity check (all three
  are standard, MIT, small).

## Test checklist (for DEV_TASKS when built)

- Bare repo (no microfox config): `ai-worker dev` boots, `/workers/config` lists workers.
- Trigger `demo` (dispatch-demo variant): parent + awaited child + fire-and-forget child all run;
  `/jobs/{id}` returns the same tree shape the console renders.
- Queue `demo-data-processor`: step 0 runs, step 1 parks `awaiting_approval`; approve via the
  boilerplate route pointed at localhost → resumes; **restart the dev server while parked** →
  state restored from `.microfox/dev-state.json`, approval still works.
- Edit the step-1 worker file while step 0 is running → step 1 executes the NEW code, no restart.
- Worker throws → retried ×3 → lands in `/dlq`; server stays alive; other workers unaffected.
- With `UPSTASH_*` env present: dev server uses real Redis store (opt-out of `local`).
