# 02 — `@microfox/ai-worker` (Runtime Package)

Deep documentation of the runtime library. Source: `packages/ai-worker/src/`.

## Module map

| File | Lines | Responsibility |
| --- | --- | --- |
| `index.ts` | 852 | `createWorker()`, `WorkerConfig`/`ScheduleConfig` types, the entire **local-mode dispatch runtime** (inline execution, local job store resolution, local `dispatchWorker`), `createLambdaEntrypoint` |
| `client.ts` | 395 | Remote dispatch: `dispatch`, `dispatchWorker`, `dispatchQueue`, `dispatchLocal`, URL derivation (`getWorkersTriggerUrl`, `getQueueStartUrl`), context serialization, `WorkerQueueRegistry` interface |
| `handler.ts` | ~1300 | Lambda side: `createLambdaHandler` (SQS event processing, job store wiring, webhooks, idempotency), `wrapHandlerForQueue` (queue/HITL/loop state machine), `createDispatchWorker` (SQS worker-to-worker — **the single SQS send point**, with the Plan F local-bridge seam), `getJobStoreKind`/`loadJobRecordById` (store-agnostic selection/read), `createWorkerLogger`, `JobStore` interface |
| `localBridge.ts` (new, Plan F) | ~45 | Dev-server dispatch seam: `set/getLocalDispatchBridge` on `globalThis`, honored only when `AI_WORKER_LOCAL=1` AND the global is installed — impossible to trip in Lambda |
| `localJobStore.ts` (new, Plan F) | ~400 | `WORKER_DATABASE_TYPE=local`: in-memory job + queue-job store with debounced JSON persistence (`.microfox/dev-state.json`, path via `AI_WORKER_LOCAL_STATE_PATH`). State anchored on `globalThis` (tsup bundles each entry separately — module-scope Maps would split state). Plus patch/list helpers serving the dev server's `/dev-store` HTTP API |
| `queue.ts` | 276 | Queue type system: `WorkerQueueConfig`, `WorkerQueueStep` (chain/resume/loop/hitl/retry), `ChainContext`, `HitlResumeContext`, `LoopContext`, `WorkerQueueContext`, `QUEUE_ORCHESTRATION_KEYS`, `defineWorkerQueue`, `repeatStep` |
| `queueJobStore.ts` | 476 | Queue job persistence (Mongo `queue_jobs` / Redis hash): `upsertInitialQueueJob`, `updateQueueJobStepInStore`, `appendQueueJobStepInStore`, `getQueueJob` |
| `mongoJobStore.ts` | 257 | Worker job persistence (Mongo `worker_jobs`): `createMongoJobStore`, `upsertJob`, `getJobById` |
| `redisJobStore.ts` | 201 | Worker job persistence (Upstash): hash per job + **separate list key for `internalJobs`** (atomic RPUSH) |
| `retryConfig.ts` | 195 | `SmartRetryConfig`, built-in patterns (`rate-limit`, `json-parse`, `overloaded`, `server-error`), `executeWithRetry`, `RetryContext` |
| `tokenBudget.ts` | 68 | `createTokenTracker`, `TokenBudgetExceededError` |
| `chainMapDefaults.ts` | 46 | Built-in chain strategies: `defaultMapChainPassthrough`, `defaultMapChainContinueFromPrevious` |
| `hitlConfig.ts` | 50 | `HitlStepConfig`, `HitlUiSpec` (`custom` view / `schema-form`), `defineHitlConfig` |
| `queueInputEnvelope.ts` | 27 | **Deprecated** — envelope schemas no longer needed (runtime strips keys) |
| `config.ts` | 100 | Workers-config HTTP client (`getWorkersConfig`, `resolveQueueUrl`) — *appears unused by the rest of the package; the example app has its own registry* |

Package exports: `.`, `./client`, `./handler`, `./config`, `./queue`, `./queueJobStore`.
Deps: `@aws-sdk/client-sqs`, `@upstash/redis`, `mongodb`, `zod@^4`. Peer: `@microfox/ai-router >= 2.1.6`.

## `createWorker()` contract

```ts
const agent = createWorker({
  id: 'video-processing',
  inputSchema: z.object({ url: z.string() }),        // z.input → dispatch arg, z.infer → handler arg
  outputSchema: z.object({ processedUrl: z.string() }),
  retry: { maxAttempts: 3, on: ['rate-limit', 'json-parse'] },   // optional in-process retry
  handler: async ({ input, ctx }) => { ... },
});
// Separately exported (CLI extracts it at build time — do NOT pass to createWorker):
export const workerConfig: WorkerConfig = {
  timeout: 900, memorySize: 2048, layers: [...], schedule: 'rate(2 hours)',
  group: 'media',                       // multi-group deployment
  sqs: { maxReceiveCount: 1, visibilityTimeout, messageRetentionPeriod, ... },
};
```

### Handler context (`ctx`)

| Field | Notes |
| --- | --- |
| `jobId`, `workerId`, `requestId`, `userId` | identity; `userId` flows from `DispatchOptions.userId` and is audit-logged as `[WORKER_USER:<id>]` |
| `jobStore` | `update({status,progress,output,error,metadata})`, `get()`, `appendInternalJob()`, `getJob(id)`; backed by Mongo or Redis chosen via `WORKER_DATABASE_TYPE` |
| `logger` | `[LEVEL] [workerId] [jobId]`-prefixed console logger; `debug` gated on `DEBUG`/`WORKER_DEBUG` |
| `dispatchWorker(id, input, opts)` | worker-to-worker. Fire-and-forget (optionally `delaySeconds` ≤ 900 via SQS DelaySeconds) or `await: true` (busy-polls the job store, default 2 s interval / 15 min timeout). Appends to parent `internalJobs`. Under `ai-worker dev` the send is intercepted by the local bridge (same append/poll logic, in-process queue instead of SQS) |
| `reportTokenUsage({inputTokens,outputTokens})` | accumulates; throws `TokenBudgetExceededError` over `maxTokens`; persists `metadata.tokenUsage` |
| `getTokenBudget()` | `{used, budget, remaining}` |
| `retryContext` | populated on attempt ≥ 2 with previous error so the handler can self-correct |

### Dispatch modes

- **`auto`** (default): inline local run iff `NODE_ENV==='development'` and
  `WORKERS_LOCAL_MODE !== 'false'`; otherwise remote.
- **`local`** — force inline. **`remote`** — force HTTP trigger even in dev.
- Remote dispatch never talks to SQS directly from the app — it always goes through the deployed
  `POST /workers/trigger` so the Next.js app needs no AWS credentials.

## Lambda lifecycle (`createLambdaHandler`)

```
SQS event → for each record (Promise.all):
  parse SQSMessageBody { workerId, jobId, input, context, webhookUrl, metadata, userId, maxTokens }
  ① idempotency: load job; skip if status ∈ {completed, failed}
  ② upsert job (queued) in selected store; build ctx
  ③ job → running ; if __workerQueue present: upsertInitialQueueJob (step 0) + step → running
  ④ run handler (with worker-level retry config if provided); outputSchema.parse(result)
  ⑤ success: job → completed(+output); webhook {status:'success'} if webhookUrl
     failure: job → failed(+error); queue step → fail; webhook {status:'error'}; rethrow (SQS retry/DLQ)
```

Notes:

- **Input is NOT schema-validated here** — `input as INPUT` (see bug B1).
- Webhooks are single-attempt, non-fatal; the job store is the source of truth.
- `batchSize: 1` is hard-coded in the generated serverless config, so the `Promise.all` over
  records is effectively single-record; raising batch size would need
  `ReportBatchItemFailures` support.

## Queue state machine (`wrapHandlerForQueue`)

State lives in the `queue_jobs` doc: `{ status: running|completed|failed|partial, steps: [{
workerId, workerJobId, status: queued|running|awaiting_approval|completed|failed, input?,
output?, error?, startedAt?, completedAt? }] }`.

Envelope: `input.__workerQueue = { id, stepIndex, arrayStepIndex?, initialInput, queueJobId,
iterationCount? }`. `stepIndex` is the **definition** index (stable across loop iterations);
`arrayStepIndex` is the **store array** position (grows as loop iterations append entries).

Ordering invariant (load-bearing): **append next/loop step BEFORE marking the current step
complete** — `updateQueueJobStepInStore` marks the whole queue `completed` when the completed
step is the last array entry, so completing before appending would close the queue early.
`notifyQueueJobStep('append')` rethrows on failure for the same reason.

### Step features

| Feature | Mechanism |
| --- | --- |
| `chain` | fn `(ChainContext) => input` or `'passthrough'` / `'continueFromPrevious'`; invoked via the generated registry (`invokeChain`) with `previousOutputs` loaded from the queue job store |
| `requiresApproval` | runtime stores the computed next input (+ `hitl.uiSpec`, `__hitlPending`) on the step, marks it `awaiting_approval`, does **not** dispatch. Approve route re-dispatches with `__hitlInput` (reviewer payload) + `__hitlDecision`; `invokeResume` (step `resume` fn or shallow merge) produces the final domain input inside the worker runtime |
| `loop` | after each run, `invokeLoop` (step's `shouldContinue`) decides re-run vs advance; `maxIterations` default 50; loop iterations append new steps[] entries with fixed `stepIndex`, incremented `iterationCount`, `arrayStepIndex = steps.length before append`. Combine with `requiresApproval` for HITL-gated loops |
| `retry` (step-level) | overrides worker-level retry for that step only; resolved via `getStepAt` before envelope stripping |
| `delaySeconds` | SQS DelaySeconds (0–900) on the dispatch of that step |
| `schedule` (queue-level) | CLI generates a queue-starter Lambda with an EventBridge schedule |

### Generated queue registry

`push` writes `.serverless-workers/generated/workerQueues.registry.js`, which `require()`s every
`.queue.ts` module (so `chain`/`resume`/`loop` **function references** are callable in the
Lambda bundle) and exposes `getQueueById/getNextStep/getStepAt/invokeChain/invokeResume/
invokeLoop`. Static regex-scanned step data is the fallback when module resolution fails.

## Job stores

### Worker jobs

- **Mongo** (`worker_jobs`, db default `worker`): doc keyed `_id = jobId`. `update()` is
  read-modify-write for metadata merge; `appendInternalJob` uses atomic `$push`.
- **Redis**: one hash per job; `internalJobs` kept in a **separate list key** (`{jobId}:internal`)
  specifically so concurrent appends are atomic (good); 7-day TTL refreshed on writes.
- **Local** (Plan F, dev only): in-memory Maps + debounced file persistence; parked HITL state and
  job history survive dev-server restarts. Never a fallback — requires the literal
  `WORKER_DATABASE_TYPE=local` (only `ai-worker dev` sets it, and compile warns if it would ship).
- Selection: `WORKER_DATABASE_TYPE` (default `upstash-redis`), with "is configured" guards;
  centralized in `getJobStoreKind()` / `loadJobRecordById()` (Plan F refactor — same precedence
  as before, reused by the idempotency check).

### Queue jobs

- **Mongo** (`queue_jobs`, db default **`mediamake`** ⚠️): `findOne` → mutate steps array →
  `$set` whole array (non-atomic, see bug B4). Append uses `$push` (atomic).
- **Redis**: whole-record load/save per update (non-atomic).

## Retry subsystem

- `executeWithRetry(fn, config, onRetry)` — in-process, same Lambda invocation, job stays
  `running`. Non-matching errors throw immediately. `TokenBudgetExceededError` never retried.
- Built-ins: `rate-limit` (msg/429, 10s·attempt), `json-parse` (SyntaxError/ZodError/msg, 0 delay,
  `injectContext` — *currently ignored, see bug B7*), `overloaded` (529, 15s·attempt),
  `server-error` (5xx, 5s·attempt). Custom: `{ match: RegExp | (err)=>bool, delayMs, injectContext }`.
- Three retry layers exist: worker-level (`createWorker({retry})` — applied in
  `createLambdaHandler` and local mode), step-level (`WorkerQueueStep.retry` — applied in
  `wrapHandlerForQueue`), and SQS redelivery (`sqs.maxReceiveCount`, default 1 = no redelivery,
  DLQ always created).

## Local mode internals (the duplicated runtime)

`createWorker().dispatch` in local mode re-implements most of the Lambda path inline
(~700 lines in `index.ts`): orchestration-key stripping, schema parse, job store resolution
(try `@/app/api/workflows/stores/jobStore` import → `WORKER_JOB_STORE_MODULE_PATH` → HTTP
`WORKER_JOB_STORE_URL`/webhook-derived URL), local `dispatchWorker` (HTTP trigger +
`setTimeout` for delays), token tracker, retry, webhook calls. This duplication is a known
maintenance hazard (see improvements I-9) and the alias-import trick likely never works at
runtime (bug B8).

## Public API quick reference

```ts
// authoring
createWorker(config) → WorkerAgent { id, dispatch, handler, inputSchema, outputSchema, retry }
defineWorkerQueue({ id, steps, schedule? })
defineHitlConfig({ taskKey, ui, inputSchema?, timeoutSeconds?, onTimeout?, assignees? })
repeatStep(count, factory)

// dispatch (server-side only; needs WORKER_BASE_URL)
agent.dispatch(input, { mode?, jobId?, userId?, webhookUrl?, metadata?, maxTokens? })
dispatchWorker(workerId, input?, options?, ctx?)       // no schema validation
dispatchQueue(queueId, initialInput?, options?)        // → POST /queues/{id}/start

// lambda glue (used by generated entrypoints)
createLambdaHandler(handler, outputSchema?, { retry? })
wrapHandlerForQueue(handler, queueRuntime)
createLambdaEntrypoint(agent)

// stores
createMongoJobStore / createRedisJobStore / upsertJob / upsertRedisJob / getJobById / loadJob
upsertInitialQueueJob / updateQueueJobStepInStore / appendQueueJobStepInStore / getQueueJob

// helpers
executeWithRetry, matchesRetryPattern, createTokenTracker, TokenBudgetExceededError
defaultMapChainPassthrough, defaultMapChainContinueFromPrevious
getWorkersTriggerUrl, getQueueStartUrl, SQS_MAX_DELAY_SECONDS

// local dev (Plan F — consumed by `ai-worker dev`; inert in Lambda)
setLocalDispatchBridge / getLocalDispatchBridge          // dispatch seam (globalThis + AI_WORKER_LOCAL=1)
getJobStoreKind / loadJobRecordById                      // store-agnostic selection + read
createLocalJobStore / upsertLocalJob / loadLocalJob / flushLocalJobStore
patchLocalJob / patchLocalQueueJob / appendLocalInternalJob      // /dev-store write surface
listLocalJobs / listLocalJobsByWorker / listLocalQueueJobs / getLocalQueueJob
upsertInitialLocalQueueJob / updateLocalQueueJobStep / appendLocalQueueJobStep
```
