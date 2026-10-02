# 03 — `@microfox/ai-worker-cli` (Build & Scaffold CLI)

Binaries: `ai-worker` and `ai-bg`. Commands: `compile`, `dev`, `new`, `boilerplate`
(+ hidden deprecated `push` alias = build-only compile).
Source: `packages/ai-worker-cli/src/`.

> **Phase 13 decoupling (2026-07):** `push` (build+deploy) was renamed to **`compile`**
> (build-only). Deployment belongs to the `microfox` CLI (`microfox push` / `microfox deploy`).
> The `deploy()` half of the old pipeline is gone from this package; `compile` ends with an
> explicit `process.exit(0)` (user module imports during config extraction can hold sockets).

## `compile` — build pipeline

`compile [group] [-s stage] [-r region] [--ai-path app/ai] [--service-name X] [--skip-group "a,b"]
[--require-auth] [--allow-public]` runs `build()` only. Stage resolution (Plan E):
`--stage` flag > `STAGE` env var > `prod`; validated against the fixed set `prod|staging|dev`.
The old force-prod override for microfox-config projects is gone — the stage is baked into
env.json (`ENVIRONMENT/STAGE/NODE_ENV`), the serverless.yml stage fallback, and SQS queue names,
so each stage deploys as a parallel CloudFormation stack.

### build()

1. **`scanWorkers(aiPath)`** — globs `app/ai/**/*.worker.ts`. Extracts:
   - worker `id` — regex on `createWorker({ id: '...' })`
   - `group` — regex `group:\s*['"]...['"]` **anywhere in the file** (false-positive prone)
   - `handlerPath` — `handlers/{dir}/{name}` mirrored from the source path
2. **`scanQueues(aiPath)`** — globs `app/ai/queues/**/*.queue.ts`. Bracket-balanced extraction of
   the `steps: [...]` array, then per-step regexes for `workerId`, `delaySeconds`,
   `requiresApproval`, presence of `chain`/`resume`/`loop`. `schedule` parsed via regex +
   `new Function` for object literals. (Limitations: computed values, `repeatStep(...)` spreads,
   template literals are invisible — the runtime registry compensates for step *behavior*, but
   serverless env injection relies on this static data.)
3. **`generateQueueRegistry()`** → `generated/workerQueues.registry.js` — `require()`s every
   queue module so chain/resume/loop fns are real; embeds the static `QUEUES` JSON as fallback;
   exports `getQueueById/getNextStep/getStepAt/invokeChain/invokeResume/invokeLoop`.
4. **`generateHandlers()`** — per worker, writes a temp TS entrypoint that imports the worker
   module (default or named `createWorker` export, detected by regex), wraps with
   `createLambdaHandler` (+ `wrapHandlerForQueue` + registry + `getQueueJob` if the worker
   appears in any queue), re-exports `exportedWorkerConfig` and `exportedInputSchema`, then
   esbuild-bundles it (`platform: node`, `target: node20`, `format: cjs`, `packages: 'bundle'`,
   external: `aws-sdk` + `microfox.json worker.externalDeps`). A post-build plugin
   (`fixLazyCachePlugin`) regex-patches known bundling landmines: `lazy-cache`/`clone-deep`
   eager-require, `import_meta.url` in CJS, `createRequire(undefined)`.
5. **Config extraction** — `await import(bundledHandler)`:
   - `exportedWorkerConfig` → `worker.workerConfig` (+ group override)
   - `exportedInputSchema` → `z.toJSONSchema()` → `worker.inputSchema`
   - on import failure: regex + `new Function` extraction of `export const workerConfig = {...}`
     from source, and **stub-based schema extraction** (`createSchemaExtractionPlugin`: bundles
     only the worker file + real zod; `@microfox/ai-worker` becomes a minimal `createWorker`
     stub; every other import becomes a callable Proxy stub — so module-level SDK/DB init can't
     throw). The JSON-Schema conversion runs *inside* the bundle to avoid cross-zod-instance
     mismatches.
6. **Generated service handlers** (all esbuild-bundled the same way):
   - `handlers/api/docs.js` — static OpenAPI for Microfox discovery
   - `handlers/api/workers-trigger.js` — `POST /workers/trigger`: optional
     `x-workers-trigger-key` check → resolve queue URL (`WORKER_QUEUE_URL_*` env →
     `GetQueueUrl` on `{GROUP_SERVICE_NAMES[group]}-{workerId}-{stage}`) → `SendMessage`
   - `handlers/api/workers-config.js` — `GET /workers/config`: returns `{ workers: {id →
     queueUrl}, schemas, queues }` (queue URLs resolved per group service name; `?debug=1`
     exposes attempted names + errors)
   - `handlers/queues/{queueId}.js` — queue starter: HTTP `POST /queues/{id}/start` or schedule
     event → `upsertInitialQueueJob` → send first worker message with `__workerQueue` envelope.
     ⚠️ resolves the first worker's queue with the **core/service name only** (bug B5)
7. **`generateServerlessConfig()`** — per worker: SQS queue + DLQ (visibilityTimeout =
   timeout+60 default, `maxReceiveCount` default 1), Lambda fn (SQS event batchSize 1 + schedule
   events), per-function `WORKER_QUEUE_URL_*` env for detected callees
   (`collectCalleeWorkerIds` — regex `dispatchWorker('id')` over the import graph — merged with
   queue adjacency via `mergeQueueCallees`); IAM scoped to the stack's queue ARNs +
   `GetQueueUrl: '*'`; `environment: ${file(env.json)}`.
8. **`env.json`** (Plan D — shared `buildEnvJson()` across all 3 write sites):
   - **Value sourcing** — stage-scoped dotenv-flow cascade: `.env` → `.env.local` →
     `.env.{stage}` → `.env.{stage}.local` (later wins; real shell/CI env wins over all files).
     Opt-in `env.files: 'isolated'` in the config reads ONLY `.env.{stage}(.local)` — no
     inheritance from `.env`; compile hard-errors if the stage file is missing, and the
     pre-config process.env hydration is un-hydrated so nothing (incl. WORKERS_API_KEY
     resolution) leaks past isolation.
   - **Key shipping** — resolution order per key: platform keys (`ENVIRONMENT/STAGE/NODE_ENV`,
     always win) > `AWS_*` (never ships) > `exclude` (glob) > `include` (glob) > mode
     `all-detected` (legacy prefix allowlist — `OPENAI_`, `ANTHROPIC_`, `DATABASE_`, `MONGODB_`,
     `REDIS_`, `UPSTASH_`, `WORKER_`, `WORKERS_`, `WORKFLOW_`, `REMOTION_`, `QUEUE_JOB_`,
     `DEBUG_WORKER_QUEUES` — plus keys referenced by worker code via
     `collectEnvUsageForWorkers`) or `explicit` (nothing else ships). `WORKERS_API_KEY` applied
     after and un-excludable.
   - **Config block** (`deploymentConfig.env`, sibling of `worker`): `{ mode, files, include,
     exclude, groups: { <name>: { mode?, include?, exclude? } } }` — group lists APPEND to
     project lists; group `mode` OVERRIDES the project mode for that group only. No `env` block
     = byte-identical legacy behavior.
   - Guard: warns loudly when `WORKER_DATABASE_TYPE=local` (the `ai-worker dev` store) would be
     baked into a deployed env.json.
9. **Multi-group layout** — when workers declare ≥ 2 distinct `group`s (requires `projectId`):
   - `.serverless-workers/core/` — trigger + config + docs + queue starters only (no worker
     lambdas); IAM `SendMessage` on every group's queue ARN pattern
   - `.serverless-workers/{group}/` — that group's worker lambdas + SQS; cross-group callees get
     convention-based `https://sqs...` URLs + scoped `SendMessage` IAM
   - root staging `handlers/`/`generated/` cleaned up afterwards
   - per-group `package.json` with only that group's runtime deps (job-store backend filtered by
     `WORKER_DATABASE_TYPE`)

### Deployment (moved out of this CLI — Phase 13)

- `npx microfox push [group] [--stage s]` — upload an existing `.serverless-workers/` build
  (stage read from the build's env.json when `--stage` omitted; mismatch prints a warning;
  sent to cicd as `publish.stage` + `x-deployment-stage`, ignored by older cicd).
- `npx microfox deploy [group] [--stage s] [--skip-group "a,b"] [--skip-compile]` — compile +
  push in one step.
- The raw `npx serverless deploy` fallback was removed (R37): self-hosted =
  `ai-worker compile` + manual `cd .serverless-workers && npx serverless deploy`.

## `dev` — local dev server (Plan F, 2026-07)

`dev [-p 4100] [--ai-path app/ai] [-c 5] [--max-receive-count 3]` — runs workers locally with
zero AWS and zero platform: one hono HTTP server exposing the deployed core-group surface
(`POST /workers/trigger`, `GET /workers/config`, `GET /docs.json`, `POST /queues/{id}/start` —
same shapes + same WORKERS_API_KEY auth rule) plus dev extras (`GET /jobs/{jobId}` run tree,
`GET /jobs`, `GET /dlq`, `GET /health`, and the `/dev-store/*` API the boilerplate's `local`
store adapter reads/writes through). Point an app at it with `WORKER_BASE_URL=http://localhost:4100`.

- **In-process invocation**: workers loaded per-invocation via jiti (tsconfig `paths` honored);
  queue chain/resume/loop run the REAL functions from `.queue.ts`; the runtime wrapper
  (`createLambdaHandler` + `wrapHandlerForQueue`) runs unchanged against synthetic
  single-record SQS events.
- **In-memory queue**: per-worker FIFO lanes with a concurrency cap; `delaySeconds` via
  setTimeout; failures that never reach the job store retry up to `--max-receive-count` then
  land in a local DLQ. Handler errors recorded as `failed` are NOT redelivered (matches deployed
  idempotency semantics).
- **Local job store**: `WORKER_DATABASE_TYPE=local` auto-selected when no Upstash/Mongo env
  exists; parked HITL state survives restarts via `.microfox/dev-state.json`. Real stores are
  used automatically when their env is present.
- **Hot reload without restarts**: chokidar + a fresh jiti module graph per change — the next
  invocation of any worker imports fresh code; worker/queue file add/remove re-scans topology
  live; `rs` + Enter forces a re-scan. Server process, queues, and parked HITL survive edits.
- **Env**: Plan D cascade with stage `dev`; `ENVIRONMENT/STAGE/NODE_ENV` forced to `dev`;
  compile-parity missing-env warnings at startup. Queue schedules are printed but NOT executed.
- Fidelity limits (documented): shared event loop + process.env, exactly-once delivery.

## `new` — scaffolding

Interactive (or `--type worker|queue`) generator:
- `app/ai/workers/{id}.worker.ts` — createWorker skeleton + `workerConfig` export
  (`--timeout`, `--memory`, `--schedule`)
- `app/ai/queues/{id}.queue.ts` — `defineWorkerQueue` skeleton with commented chain/HITL/loop
  examples

## `boilerplate` — Next.js integration scaffold

Writes the consumer-side files into the user's app (skips existing unless `--force`):

```
app/api/workflows/auth.ts                      getClientId stub
app/api/workflows/stores/{jobStore,mongoAdapter,redisAdapter,queueJobStore}.ts
app/api/workflows/stores/localDevAdapter.ts    NEW (Plan F): WORKER_DATABASE_TYPE=local — proxies
                                               store reads/writes to `ai-worker dev`'s /dev-store API
app/api/workflows/registry/workers.ts          config-API-driven registry (+ getWorkerSchema)
app/api/workflows/workers/[...slug]/route.ts   trigger/status/schema/history/update/webhook/job
app/api/workflows/queues/[...slug]/route.ts    trigger/status/list/update/approve/webhook
hooks/useWorkflowJob.ts                        client trigger+poll+HITL hook
```

Plus: merges `workflowSettings` into `microfox.config.ts` (line-based insertion; `--skip-config`
to skip) and adds/upgrades `REQUIRED_DEPENDENCIES` (`@microfox/ai-worker`, `@upstash/redis`,
`mongodb`, `zod`) in `package.json`.

### Template sync (new — 2026-06-12)

Templates are **no longer hand-maintained inline**. They live in
`src/commands/boilerplate.templates.generated.ts`, generated from `examples/root` by
`scripts/sync-boilerplate.mjs`:

```bash
npm run sync-boilerplate          # regenerate from examples/root
npm run sync-boilerplate:check    # CI guard; also runs as prebuild
```

See [07-boilerplate-sync.md](./07-boilerplate-sync.md) for the workflow and the
mediamake divergence report.

## Generated artifact layout (single group)

```
.serverless-workers/
├── serverless.yml          # service, SQS+DLQ resources, functions, IAM, env file ref
├── env.json                # filtered env (⚠️ contains secrets — must be gitignored)
├── package.json            # runtime deps (workers' npm imports) + serverless devDeps
├── microfox.json           # copied platform config (when present)
├── deployments.json        # written by microfox CLI push (deployment ids per group)
├── generated/workerQueues.registry.js
└── handlers/
    ├── api/{docs,workers-trigger,workers-config}.js
    ├── queues/{queueId}.js
    └── workers/{...mirrored worker paths}.js   # one self-contained bundle per worker
```

## Known weak points (details in 05-bugs.md / 06-improvements.md)

- Regex scanning of TS source (ids, groups, steps, schedules) — fragile, `new Function` use.
- Bundled-handler `import()` during build executes user module top-level code before the safe
  stub path is tried (mitigated: `compile` now force-exits after the build).
- `fixLazyCachePlugin` string-patches esbuild output.
- Multi-group queue starters can't resolve grouped first-worker queues (B5).
- `npx microfox@latest` / `serverless@^3.38` unpinned at deploy time.
- NEW (found 2026-07-09, R50): the generated queue registry's `resolveStepData` omits the step's
  `retry` config, so step-level SmartRetry silently does nothing in DEPLOYED stacks (the
  `ai-worker dev` runtime returns it, so dev and prod diverge until fixed).
