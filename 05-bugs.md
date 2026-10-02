# 05 — Bugs & Edge Cases (with fixes)

Correctness issues found in the audit, ordered by impact. `B#` ids are referenced from the other
docs. Security issues live in [04-security-issues.md](./04-security-issues.md).

---

## Runtime (`packages/ai-worker`)

### B1 — Input schema never validated on the Lambda path ⚠️ high

**Where:** `handler.ts:1119` — `handler({ input: input as INPUT, ctx })`. Only `outputSchema`
is parsed (`handler.ts:1120`).

**Consequences:**
- zod `.default()` / `.transform()` / `.coerce` are applied in **local mode**
  (`index.ts:306` does `inputSchema.parse`) but **not in production** → silent behavior
  divergence between dev and deployed.
- Queue chain inputs (whatever `invokeChain` returns) and HITL-resumed inputs are never
  validated at all.
- Anything POSTed to `/workers/trigger` reaches handlers unvalidated.

**Fix:** the generated entrypoint already has `workerAgent.inputSchema` — pass it into
`createLambdaHandler(handler, outputSchema, { inputSchema })` and parse after the queue wrapper
strips envelope keys (i.e. inside `wrapHandlerForQueue` after step 3, and in
`createLambdaHandler` for non-queue workers). Strip orchestration keys before parsing exactly
like local mode does.

### B2 — Loop-step failure marks the wrong step ⚠️ high

**Where:** `handler.ts:1165-1174` (failure path) uses `queueCtxFail.stepIndex`, while the start
path (`handler.ts:1107`) correctly uses `queueCtx.arrayStepIndex ?? queueCtx.stepIndex`.

**Consequence:** when iteration N (array index > definition index) of a looping step fails, the
*original* step row is overwritten as failed; the actually-running row stays `running` forever →
queue job shows inconsistent state and the UI's `findIndex(status==='awaiting_approval'/'running')`
logic targets the wrong row.

**Fix:** `stepIndex: queueCtxFail.arrayStepIndex ?? queueCtxFail.stepIndex`.

### B3 — `previousOutputs` drops loop iterations

**Where:** `handler.ts:557-559` (loop path) and `handler.ts:681-683` (chain path):
`job.steps.slice(0, stepIndex)` slices the **store array** with the **definition index**.
`loadPreviousOutputsBeforeStep` (`handler.ts:262`) has the same issue for HITL resume.

**Consequence:** for a looping step on iteration ≥ 2, `previousOutputs` contains only the steps
before the loop's definition index — earlier loop iterations are missing, contradicting the
documented contract (`queue.ts:209`: "Outputs from all previous steps **(and previous iterations
of this step)**"). Mappers that need the running history (e.g. accumulating sessions) silently
get partial data; they currently survive because the current output is concat'ed manually.

**Fix:** slice with `arrayStepIndex` (pass it through `ChainContext`/`LoopContext`), or include
all completed array entries before the current array position.

### B4 — Queue job store read-modify-write races ⚠️ high

**Where:** `queueJobStore.ts:315-353` (Mongo `updateQueueJobStepInStore`: `findOne` → mutate
`existing.steps` → `$set` **the whole array**) and `:356-388` / `:418-430` (Redis: load whole
hash → save whole hash).

**Consequence:** a concurrent atomic `$push` append (from another Lambda, the approve route, or
a loop dispatch) landing between the read and the write is **erased** when the stale array is
written back. Symptoms: vanished steps, queues stuck `running`, or premature `completed` (the
length check `stepIndex === steps.length - 1` evaluates against stale length). The code works
around the worst case with strict append-before-complete ordering *within one process*, and the
approve route writes step state before dispatching — but cross-process interleavings (HITL
approve + loop, two children of the same queue) remain unsafe.

**Fix (Mongo):** atomic positional updates instead of whole-array `$set`:
```js
coll.updateOne(
  { _id: queueJobId, [`steps.${stepIndex}.workerJobId`]: workerJobId },
  { $set: { [`steps.${stepIndex}.status`]: status, ..., updatedAt: now } }
);
```
and compute queue completion with a follow-up guarded update (`$expr` on last-element status)
or a `$size`-aware aggregation pipeline update.
**Fix (Redis):** per-step keys (`queue-jobs:{id}:step:{i}`) + a step-count key, or a Lua script
for read-modify-write.

### B5 — Multi-group queue starters resolve the wrong queue name ⚠️ high (regression)

**Where:** generated queue handler (`push.ts:1875-1884`):
`queueName = ${SERVICE_NAME}-${FIRST_WORKER_ID}-${stage}` where `SERVICE_NAME` is the **core**
service. Grouped workers' queues are `p-{proj}-{group}-{workerId}-{stage}`.

**Evidence of regression:** the committed example output
(`examples/root/.serverless-workers/core/serverless.yml`) contains an `environment: &ref_0`
block injecting `WORKER_QUEUE_URL_*` (group-aware URLs) onto all core functions — but **no code
in current `push.ts` generates that block** (`generateServerlessConfigCore` adds no function
env). Whatever produced it is gone. A fresh multi-group push today produces queue starters that
throw `Queue URL not found` for any queue whose first worker lives in a non-default group
(e.g. `calculator-session` → `calculator-hitl` in group `calculator`).

**Fix:** either (a) embed `WORKER_GROUPS` + `GROUP_SERVICE_NAMES` into the queue handler
template like `workers-trigger.js` already does, or (b) restore env injection in
`generateServerlessConfigCore`: build one shared env object with convention-based queue URLs for
all workers and attach to trigger/config/queue functions.

### B6 — `generateWorkersMap` queries a wrong CloudFormation stack

**Where:** `push.ts:2760`: `const stackName = \`ai-router-workers-${stage}-${stage}\`` —
stage doubled, and ignores `--service-name`/projectId-derived names. Always falls into the
warning path and emits placeholder URLs containing `${aws:region}` literals.

**Fix:** `\`${serviceName}-${stage}\`` (serverless stack naming), passing the resolved
serviceName through; or drop the feature (Microfox path never uses it).

### B7 — `injectContext` retry flag is dead

**Where:** `retryConfig.ts:173` — `matchesRetryPattern` returns `injectContext` but
`executeWithRetry` destructures only `{ matched, delayMs }` and always builds `retryCtx`.
Docs (`retryConfig.ts:33`, built-in table) promise per-pattern control.

**Fix:** honor the flag (pass `undefined` retryCtx when false) or delete the flag and fix docs.
Honoring it is better: patterns like `rate-limit` shouldn't pollute prompts with error text.

### B8 — Local-mode "direct job store" import can't resolve at runtime

**Where:** `index.ts:317-351` — `await import(nextJsPathAlias)` where the alias is the literal
`'@/app/api/workflows/stores/jobStore'` held in a variable. From compiled `dist/` inside
`node_modules`, Node can't resolve `@/…` and bundlers can't statically analyze a variable
dynamic import → it throws and is silently caught, so `directJobStore` is effectively always
`null` unless `WORKER_JOB_STORE_MODULE_PATH` is set. Additionally `get`/`getJob`/
`appendInternalJob` re-import the module **on every call**.

**Fix:** drop the alias trick; accept an explicit injection point (e.g.
`globalThis.__WORKER_JOB_STORE__` registered by the boilerplate `jobStore.ts`, or a
`setLocalJobStore(store)` export the app calls once). Cache whatever resolution remains.

### B9 — Local dispatch blocks the route and lies about status

**Where:** `index.ts:745-793` — local mode runs the entire handler inline (awaited child polls
default to 15 min) and then returns `{ status: 'queued' }` even though the job is terminal.

**Fix:** return `{ status: 'completed' /* or 'failed' */ }` from local mode (the type already
allows only `'queued'` — widen it), and document that local mode is synchronous; optionally run
the handler unawaited (`queueMicrotask`) to mimic remote behavior.

### B10 — `dispatch()` zod-parse silently strips unknown keys remotely, but local mode strips a different set

**Where:** `client.ts:222` (`inputSchema.parse(input)` → parsed object is what's sent) vs
`index.ts:297-306` (local strips only orchestration keys, then parses).

**Consequence:** with a non-strict zod object, extra keys are dropped before send in remote mode
(fine) but a `.passthrough()` schema keeps them — meanwhile metadata-ish keys callers piggyback
on input behave differently between modes. Minor, but contributes to local/prod divergence with
B1.

**Fix:** after B1 (validate in Lambda), make `dispatch()` send the raw input and validate
only server-side, or validate-and-send-parsed in *both* modes.

### B11 — `hitl` (plain key) is reserved and stripped from worker input

**Where:** `queue.ts:71-77` — `QUEUE_ORCHESTRATION_KEYS` includes `'hitl'`.

**Consequence:** a worker whose domain schema legitimately has a `hitl` field never receives it
(silently deleted by `wrapHandlerForQueue` and local mode).

**Fix:** rename the envelope key to `__hitl` (keep reading both for one release) so all reserved
keys share the dunder prefix.

### B12 — Idempotency check is read-then-act

**Where:** `handler.ts:984-1009`. Two concurrent deliveries of the same message both pass the
terminal-status check and run the handler twice.

**Fix:** at-least-once is acceptable *if documented*; for stronger guarantees do an atomic
claim: `updateOne({_id: jobId, status: {$nin: ['running','completed','failed']}}, {$set:
{status: 'running', …}}, {upsert: true})` and skip when `matchedCount === 0` with a non-queued
status. Redis: `SET key running NX` style guard.

### B13 — Awaited `dispatchWorker` can outlive the parent Lambda

**Where:** `handler.ts:881-905` — default poll timeout 15 min == max Lambda timeout; a parent
with smaller timeout dies mid-poll → SQS redelivers (if `maxReceiveCount > 1`) → duplicate child
dispatches (B12's claim guard would absorb this).

**Fix:** cap `pollTimeoutMs` to `context.getRemainingTimeInMillis() - safety`; document the cost
(parent billed while polling). Longer-term: continuation-style (child sends a "wake parent"
message) or Step Functions for true fan-in.

### B14 — Inconsistent store defaults: queue jobs default to db `mediamake`

**Where:** `queueJobStore.ts:44-48` (`MONGODB_DB || 'mediamake'`) vs `mongoJobStore.ts:13-16`
(`… || 'worker'`).

**Consequence:** with only `MONGODB_URI` set, worker jobs land in db `worker` and queue jobs in
db `mediamake` (a project-specific name leaked into the library).

**Fix:** unify on one default (`worker`), envs `MONGODB_WORKER_DB` honored by both.

### B15 — Token usage state lost on the over-budget call

**Where:** `handler.ts:1050-1058` — `tokenTracker.report()` throws before the store persist
runs, so the final (exceeding) usage numbers are never written.

**Fix:** record usage first, persist, then throw.

---

## CLI (`packages/ai-worker-cli`)

### B16 — `group:` regex matches anywhere in the worker file

**Where:** `push.ts:757` — `content.match(/group:\s*['"]([^'"]+)['"]/)`. A zod field, an API
payload, or a comment containing `group: 'x'` silently reassigns the worker's deployment group
(changing its service/stack!). Fix by reading `group` only from the extracted
`exportedWorkerConfig` (which `build()` obtains later anyway) and dropping the source regex.

### B17 — Worker/queue scanning misses computed metadata

`createWorker({ id: SOME_CONST })`, `requiresApproval: flag`, `steps: [...repeatStep(3, …)]`,
schedule template literals are invisible to `scanWorkers`/`scanQueues`. The runtime registry
covers behavior, but serverless generation (queue env injection, schedule events, HITL
`requiresApproval` static fallback) silently degrades. Fix: evaluate modules with the existing
schema-extraction stub plugin to get *real* config objects (see I-2).

### B18 — Build-time import executes user module side effects

`push.ts:2980` imports the full bundled handler (DB connects, SDK init at module top-level run
on the dev machine; only on *throw* does it fall back to stubs). Reverse the order: try the stub
extraction first, full import only as fallback.

### B19 — `scanQueues` schedule regex can match nested strings

`schedule:` inside a step's `hitl` config or a comment block (line comments are stripped, block
comments are not) becomes the queue schedule. Fix alongside B17.

### B20 — `collectEnvUsageForWorkers` misses env usage in imported npm packages

Only local files are scanned, so e.g. a worker using `@microfox/some-sdk` that reads
`SOME_SDK_KEY` won't get it into env.json unless prefix-allowlisted. Document, or merge
`exportedWorkerConfig.env` (new field) as an explicit override.

> ✅ **Addressed by Plan D (2026-07):** the `env` block's `include` list is exactly this
> explicit override — undetected keys ship by listing them (globs supported, per-group overlays).

### B24 — Generated queue registry drops step-level `retry` config (found 2026-07-09, flag R50)

`generateQueueRegistry`'s `resolveStepData` never includes the step's `retry`, so
`wrapHandlerForQueue`'s lookup (`getStepAt(...).retry`) is always undefined in DEPLOYED stacks —
step-level SmartRetry on queue steps silently does nothing in prod. The `ai-worker dev` runtime
DOES return it, so dev and prod diverge until fixed. One-line fix in the registry template
(add `...(moduleStep?.retry ? { retry: moduleStep.retry } : {})`), pending review.

---

## Example app / boilerplate (`examples/root`)

### B21 — Queue webhook marks the whole queue completed on any worker success

`queues/[...slug]/route.ts#handleQueueWebhook` sets queue-job status from a *worker* webhook
(`status === 'success'`) — but a queue is only complete when its last step completes. If a
webhookUrl is passed through queue dispatch, an intermediate step's success would mark the queue
job `completed`. Guard: only update if the reporting step is the final one, or drop queue-level
webhook handling in favor of store polling.

### B22 — `useWorkflowJob` unstable identities

`options` (inline object) and `api` (closure) are in `useCallback` deps → `trigger` identity
changes every render; consumers using it in effects re-run. Memoize `api` on `baseUrl` and
destructure scalar options into deps.

### B23 — Poll loop spins on persistent 404 until the deadline

Worker poll path treats non-OK as "keep trying" with no 404 short-circuit; a mistyped workerId
polls for 5 minutes silently. Surface 404 immediately after a small grace window (job-store
eventual consistency).

---

## mediamake divergences (downstream copies of fixed code)

These are **fixed in ai-router but still broken in mediamake** — port them over
(see [07-boilerplate-sync.md](./07-boilerplate-sync.md)):

1. `queues/[...slug]/route.ts` — missing the HITL approve race fix (step set `running` *before*
   `dispatchWorker`); mediamake dispatches first, updates after.
2. `hooks/useWorkflowJob.ts` — missing the `terminalHitRef`/`clearThisPolling` fixes: stale
   interval/timeout refs can clear a *newer* poll cycle, and a terminal status hit during the
   first synchronous poll still starts an interval.
