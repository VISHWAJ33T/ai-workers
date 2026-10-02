# 06 — Improvements & Optimizations

Non-bug enhancements, ordered roughly by value/effort. `I#` ids referenced from other docs.

---

## Architecture & correctness foundations

### I-1 — Test suite for the queue state machine (highest leverage)

Both packages have zero tests (vitest is configured in `ai-worker`). The
`wrapHandlerForQueue` + `queueJobStore` combination is a distributed state machine whose
ordering bugs were clearly discovered in production (the append-before-complete comments). Add:

- **Unit**: `executeWithRetry` (patterns, delays, TokenBudget passthrough), `tokenBudget`,
  chain defaults, `getWorkersTriggerUrl`/`getQueueStartUrl` normalization, envelope
  stripping/HITL merge in `wrapHandlerForQueue` (pure parts).
- **Simulation**: in-memory `JobStore`/queue-store fakes + a fake `queueRuntime`; drive
  multi-step queues through chain/HITL/loop paths, assert step arrays, statuses,
  `arrayStepIndex` accounting (would have caught B2/B3).
- **Race tests**: interleave `append` and `updateStep` against a real Mongo (testcontainers) to
  lock in the atomic-update fix (B4).
- **CLI snapshot tests**: run `build()` against `examples/root` fixtures, snapshot
  `serverless.yml` + generated handlers (would have caught B5/B6 regressions).

### I-2 — Replace regex scanning with stub-module evaluation

`push.ts` already contains the right tool: `createSchemaExtractionPlugin` evaluates a worker
module with all heavy imports stubbed. Generalize it into `evaluateModuleMetadata(file)` that
returns `{ id, workerConfig, inputSchema, queueConfig }` for worker *and* queue files, and make
regex the fallback instead of the primary. Kills B16–B19 in one move and unlocks computed
ids/`repeatStep` support in serverless generation.

### I-3 — Single execution core for local + Lambda

`index.ts` local mode duplicates ~700 lines of `handler.ts` (job store wiring, dispatchWorker,
retry, webhooks, token tracking). Extract a shared `executeJob({ message, jobStore, dispatchImpl,
logger })` used by both `createLambdaHandler` and local dispatch; local mode then differs only
in transport (HTTP trigger vs SQS) and store resolution. Halves the surface where bugs like
B1/B9 diverge.

### I-4 — Atomic queue store (pairs with bug B4)

Beyond the positional-update fix, consider modeling queue completion explicitly: store
`totalSteps`/`currentStep` counters and a `pendingNext` flag instead of inferring completion
from "completed last array element" — removes the fragile append-before-complete ordering
dependency entirely.

### I-5 — Versioned message envelope

`SQSMessageBody` and `__workerQueue` have no version field. Add `v: 1` so future changes
(e.g. renaming `hitl` → `__hitl`, B11) can be rolled out with dual-read.

---

## DX

### I-6 — Local queue execution

> ✅ **DONE (Plan F, 2026-07): `ai-worker dev`.** One local HTTP server with the deployed
> core-group surface; workers invoked in-process (real chain/resume/loop functions from the
> user's `.queue.ts`), in-memory queue standing in for SQS, file-persisted local job store
> (parked HITL survives restarts), hot reload without restarts, local DLQ, and a `/dev-store`
> API + boilerplate `localDevAdapter` so the app UI works with zero Redis/Mongo/AWS. Full queue +
> HITL iteration now needs no deploy. (Schedules are printed, not executed; worker_threads
> isolation deferred as a possible `--isolate` v2.)

Original ask (kept for context): queues *required* deployed infra — `dispatchQueue` always
POSTed to the queue-start Lambda; you could not iterate on chain/HITL logic without `push`.

### I-7 — `ai-worker doctor` / `validate` command

Pre-deploy checks: WORKER_BASE_URL reachable, trigger key set, queue steps reference existing
worker ids (the scanner already warns; make it fail), group names valid, first-worker group vs
queue-starter resolution (B5), env keys referenced but missing from `.env`, secrets about to be
written to a git-tracked path (SEC-10).

### I-8 — Typed end-to-end dispatch

`dispatchWorker`/`dispatchQueue` are stringly-typed. Generate a `workers.d.ts` (the CLI already
extracts JSON Schemas) mapping worker id → input/output types, so apps get
`dispatchWorker<'video-processing'>(…)` completion and compile-time input checking.

### I-9 — Progress API sugar

`ctx.jobStore.update({ progress, progressMessage })` works but is verbose and merges into
metadata implicitly. Add `ctx.reportProgress(percent, message?)` and surface it first-class in
`useWorkflowJob` (`output.metadata.progress` today).

### I-10 — Boilerplate auth that fails closed

Ship `auth.ts` with a `requireClientId()` helper and wire the mutating routes through it,
returning 401 until the developer implements it (instead of silently-open routes, SEC-5).
Provide a ready-made HMAC verifier for webhook routes (SEC-6).

### I-11 — README/docs accuracy pass

- README still documents `withQueueOrchestrationEnvelope` (deprecated) and "MongoDB only" job
  updates while the default is `upstash-redis`.
- Document the three retry layers and their interaction (worker retry × step retry × SQS
  redelivery), `arrayStepIndex` semantics, local-mode synchronous behavior (B9), reserved input
  keys (B11), and the SQS 256 KB payload limit.
- `examples/root/docs/HITL_QUEUES.md` is good — link it from the package README.

---

## Performance & cost

### I-12 — Stop busy-polling for child workers (`await: true`)

Parent Lambda burns its full duration polling every 2 s (cost = parent GB-seconds for the whole
child runtime; plus B13's timeout interplay). Options, in increasing effort: (a) document and
cap; (b) poll with backoff; (c) child completion sends an SQS message to a "resume" queue and
the parent splits into two invocations (continuation token in the job store); (d) Step Functions
for fan-out/fan-in pipelines.

### I-13 — SQS batching + partial failures

`batchSize: 1` everywhere. For high-volume workers allow `batchSize: N` +
`ReportBatchItemFailures` (`functionResponseTypes`), returning per-record failures instead of
the current all-or-nothing `Promise.all` (which would re-run *successful* records on a single
failure at batch > 1).

### I-14 — Reuse SQS clients & job-store connections

`createDispatchWorker` constructs a new `SQSClient` per call (`handler.ts:859`, `:811`);
`resolveQueueUrlForWorker` re-resolves queue URLs per call with no cache. Hoist clients to
module scope, memoize `GetQueueUrl` results (the `config.ts` cache pattern exists but is
unused). Same for the per-call dynamic `import('../../stores/jobStore')` in the boilerplate
routes (Next.js caches modules, so this is mostly noise — but the local-store re-import per
call in `index.ts` is real, B8).

### I-15 — Slim Lambda bundles

`packages: 'bundle'` inlines the AWS SDK v3 (present in the node20 runtime) and the full
mongodb/upstash clients into *every* worker bundle, plus `node_modules` ships when
`includeNodeModules` is on. Mark `@aws-sdk/*` external by default (it's in the runtime),
tree-shake job stores by backend (only bundle the one selected by `WORKER_DATABASE_TYPE`), and
report bundle sizes after build.

### I-16 — Parallelize the per-worker build loop

`generateHandlers` builds workers sequentially; esbuild calls are independent —
`Promise.all` with a small concurrency pool cuts push time roughly linearly with worker count.
Same for the config-extraction loop.

### I-17 — Job store TTL/indexes

Mongo collections have no indexes (`workerId`, `status`, `createdAt` for history/list queries;
TTL index to mirror Redis' 7-day expiry). `listJobsByWorker` (new history endpoint) will table
scan.

---

## Pipeline (cicd/v2, microfox CLI, Foxhub)

### I-18 — Deployment queue robustness

`checkAndStartNext` is only triggered after stop/finish events; a crashed PM2 worker leaves
ACTIVE statuses stuck forever (blocks the project+group slot). Add a watchdog: deployments in
ACTIVE states with no heartbeat for N minutes → FAILED + slot release. Persist PM2 worker exit
codes onto the deployment.

### I-19 — Stream deployment logs

Console polls `GET /deployments/:id/logs` for the full log doc each time. Add incremental
fetch (`?after=<timestamp>`) or SSE; logs grow large with `--verbose` serverless output.

### I-20 — `extractBaseUrl` robustness

Regex over CLI text output (`GET - https://…/docs.json`) breaks with serverless output format
changes. Read CloudFormation outputs (the stack already exports endpoint info) instead.

### I-21 — microfox CLI: validate before upload

`push` zips and uploads without checking `serverless.yml` exists in the chosen dir or env.json
sanity; failures surface minutes later in cicd logs. Pre-flight locally; also honor
`.serverless-workers/deployments.json` to warn when re-pushing an identical archive
(hash-based skip).

### I-22 — Console observability for workers/queues

The console shows deployments + raw CloudWatch logs. The structured data already exists for a
proper *jobs* view: `worker_jobs`/`queue_jobs` stores + the `[WORKER_USER:…]` /
`[WorkerEntrypoint]` log markers + `workers/config` schemas. A "Jobs" tab (status, durations,
token usage from `metadata.tokenUsage`, queue step timelines, HITL pending approvals) would be
far more useful than raw log search — and it's mostly a read-only UI over existing collections.

---

## Code health

### I-23 — Delete dead/deprecated surface

- `client.ts`: `WorkerQueueRegistry`, `DispatchOptions.registry`, `onCreateQueueJob` (unused by
  `dispatchQueue` since it became HTTP-based)
- `config.ts` workers-config client (example app has its own registry; nothing imports this)
- `queueInputEnvelope.ts` (deprecated)
- `MapStepInputContext` (deprecated alias)
- `handler.ts` `SQSMessageBody.jobStoreUrl` (deprecated)
- CLI: v1 push path in microfox CLI if v2 is the norm

### I-24 — Consolidate env var aliases

3 aliases for Mongo URI, 3 for Redis URL/token, 3 for TTL, 2 legacy trigger URLs. Pick the
`WORKER_*` canonical set, keep one release of fallback with deprecation warnings, document the
matrix (see 01-architecture.md) in the README.

### I-25 — Structured logging

Logs are `console.log` with ad-hoc prefixes (`[Worker]`, `[WorkerEntrypoint]`, `[queue]`,
`[WORKER_USER:…]`). Emit JSON lines (`{level, ts, jobId, workerId, queueId, event}`) —
CloudWatch Insights queries and the console's log viewer both get dramatically better, and the
`WORKER_USER` grep-marker hack becomes a queryable field.

### I-26 — CI guards

- `sync-boilerplate --check` (already wired as prebuild) in the repo CI
- typecheck + lint both packages (currently only scripts exist)
- snapshot test of `push --skip-deploy` against `examples/root` (I-1)
- secret scanner (gitleaks) — would have caught SEC-1
