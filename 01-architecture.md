# 01 — End-to-End Architecture

This document maps the **entire microfox background-worker ecosystem**: five codebases that
together take a TypeScript function colocated in a Next.js app and run it on AWS Lambda behind
SQS, with job tracking, multi-step queues, human-in-the-loop (HITL) approval, observability, and
a managed deployment pipeline.

## The five codebases

| Repo / path | Role |
| --- | --- |
| `ai-router/packages/ai-worker` | **Runtime library.** `createWorker()`, dispatch clients, Lambda handler wrapper, queue/HITL/loop orchestration, job stores (MongoDB / Upstash Redis), smart retries, token budgets |
| `ai-router/packages/ai-worker-cli` | **Build & scaffold CLI.** `compile` (scan → bundle → generate serverless project; build-only since Phase 13), `dev` (Plan F local dev server: in-process workers + in-memory queue + hot reload, zero AWS), `new` (scaffold worker/queue), `boilerplate` (scaffold Next.js API routes/stores/hooks) |
| `ai-router/examples/root` | **Reference app & source of truth for boilerplate.** Next.js app with workers, queues, workflow API routes, `useWorkflowJob` hook, demo pages. `mediamake/apps/mediamake` is the largest real consumer |
| `microfox/packages/cli` (`npx microfox`) | **Platform deploy CLI (owns deployment since Phase 13).** `push [--stage]` zips the generated `.serverless-workers` output (per group) and uploads it to cicd; `deploy` = compile + push; `compile` wraps `ai-worker-cli compile`; `status`/`logs`/`metrics` monitoring; `kickstart` scaffolds projects |
| `cicd/v2` | **Deployment server** (Express + PM2 + Mongo on a VM). Receives archives, queues deployments (per project+group+**stage** since Plan E), runs `npm install` + `serverless deploy` with platform AWS credentials, injects the **durable per-stage env store** (Plan E Phase 3: encrypted project/group/function layers that survive redeploys), syncs S3 assets, updates project URL maps (stage-aware rows), generates embeddings |
| `Foxhub/apps/web` (console subdomain) | **Control plane UI.** Project/org creation, deployment history + step logs (via cicd API), Lambda observability (CloudWatch logs/analytics via Foxhub's own `/api/aws/*` routes) |
| `Foxhub/apps/microfox-reroute` | **Cloudflare Worker** on `*.microfox.app/*`. Resolves a subdomain to its deployed target by reading the Upstash *routemap* (`subdomains:{subdomain}` hash, written by cicd `ProjectService`) and rewriting the request. See [08-microfox-reroute.md](./08-microfox-reroute.md) |

## Full request flow: from code to running worker

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ DEVELOPER MACHINE                                                           │
│                                                                             │
│  app/ai/workers/foo.worker.ts        app/ai/queues/bar.queue.ts             │
│        │  createWorker({id,…})             │  defineWorkerQueue({steps})    │
│        ▼                                   ▼                                │
│  npx ai-worker compile [--stage s]  (= @microfox/ai-worker-cli; build-only) │
│   1. scanWorkers()   — regex-extract worker ids/groups from *.worker.ts     │
│   2. scanQueues()    — regex-extract queue steps from *.queue.ts            │
│   3. generateQueueRegistry() — JS module importing queue files (chain/      │
│      resume/loop fns callable at runtime)                                   │
│   4. generateHandlers() — per-worker esbuild bundle wrapping the handler    │
│      in createLambdaHandler(+wrapHandlerForQueue if in a queue)             │
│   5. import bundled handler → extract workerConfig + inputSchema (zod →     │
│      JSON Schema, with stub-plugin fallback)                                │
│   6. generate workers-config / trigger / docs / queue-starter handlers      │
│   7. generateServerlessConfig() — serverless.yml + SQS queues + DLQs +      │
│      IAM + per-function WORKER_QUEUE_URL_* env                              │
│   8. env.json — stage env-file cascade (.env → .env.local → .env.{stage}    │
│      → .env.{stage}.local; or ONLY stage files with env.files='isolated')   │
│      filtered by the Plan D env block (mode/include/exclude, per-group      │
│      overlays incl. per-group mode) + allowed prefixes + referenced keys    │
│      → output: .serverless-workers/ (single) or .serverless-workers/        │
│        {core,groupA,groupB}/ (multi-group). Stage baked into env.json +     │
│        serverless.yml + SQS names (parallel stack per stage, Plan E)        │
│                                                                             │
│  npx ai-worker dev  (Plan F — no deploy at all: local server on :4100,      │
│   in-process workers, in-memory queue, local job store, hot reload)         │
│                                                                             │
│  npx microfox push [group] [--stage s] / microfox deploy (compile+push)     │
│   - per group: zip dir → POST https://{prod|staging}-v2-cicd.microfox.app   │
│     /api/deployments/local  (headers: x-project-id, x-deployment-group,     │
│     x-deployment-stage; body publish.stage — old cicd ignores both)         │
│   - saves deploymentId → .serverless-workers/deployments.json               │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ CICD/V2 SERVER (Express + PM2, DigitalOcean VM)                             │
│                                                                             │
│  POST /api/deployments/local                                                │
│   - look up Project by x-project-id (Mongo)                                 │
│   - stage = publish.stage | x-deployment-stage | 'prod' (validated)         │
│   - DeploymentQueueService.canStartNow(project, group, stage)               │
│     (MAX_CONCURRENT_DEPLOYMENTS=3, one active per project+group+stage —     │
│      a staging deploy no longer blocks a prod deploy of the same group)     │
│   - extract zip → deployments/deploy-agent-{deploymentId}/                  │
│   - WorkerService.startDeployWorker → PM2 `deploy-agent-{id}` process       │
│                                                                             │
│  LocalDeploymentOrchestrator.run()                                          │
│   COMPILING  → CompilerService (smart-cache npm install, build)             │
│   DEPLOYING  → ServerlessService.deployFunction                             │
│                - rewrite serverless.yml service name to                     │
│                  p-{projectId(15)}[-{group(12)}]                            │
│                - inject env: zip env.json ← console env STORE (Plan E ph3:  │
│                  encrypted project/group layers, store wins) ← platform     │
│                  env (ENCRYPTION_KEY, TASK_UPSTASH_*, BASE_SERVER_URL —     │
│                  always wins); per-function store overrides applied after   │
│                - `npx serverless deploy --stage {stage} --force` using      │
│                  AGENT_AWS_* platform credentials                           │
│                ∥ S3SyncService.syncPublicDirectory (parallel)               │
│   POST_DEPLOYMENT → embeddingsService.embedAgent (RAG index of docs.json),  │
│                PostDeploymentService.enrichProjectFromDeployment            │
│                (fetch docs.json → endpoints, lambdas, urlsMap merge         │
│                 by (type, group, stage); non-prod rows get stage-prefixed   │
│                 paths /staging/agent etc. — reroute Worker unchanged)       │
│   COMPLETED  → Deployment doc updated; queue freed                          │
│                                                                             │
│  Step logs → DeploymentLog (Mongo) ; metrics → Deployment.metrics           │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ AWS (single platform account)                                               │
│                                                                             │
│  Per group stack p-{proj}-{group}:                                          │
│    SQS  {service}-{workerId}-{stage} + DLQ      (per worker)                │
│    Lambda workerFoo (SQS event, batchSize 1)                                │
│  Core stack p-{proj}-core (or single stack):                                │
│    API GW: GET /docs.json | GET /workers/config | POST /workers/trigger     │
│            POST /queues/{id}/start (+ schedule events)                      │
│    Lambda: docs / workers-config / workers-trigger / queue starters         │
│  Job state: MongoDB (worker_jobs, queue_jobs) or Upstash Redis              │
│             (worker:jobs:*, worker:queue-jobs:*) per WORKER_DATABASE_TYPE   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ CONSUMER NEXT.JS APP (examples/root, mediamake)                             │
│                                                                             │
│  Server: /api/workflows/workers/[...slug]  POST trigger / GET status,       │
│          schema, history / POST update, webhook, job                        │
│          /api/workflows/queues/[...slug]   POST trigger / GET status,list / │
│          POST update, approve, webhook                                      │
│          registry/workers.ts — synthetic agents from GET /workers/config    │
│          stores/* — jobStore facade → mongoAdapter | redisAdapter,          │
│          queueJobStore                                                      │
│  Client: hooks/useWorkflowJob.ts — trigger + poll + HITL task derivation +  │
│          submitHitlDecision                                                 │
│  Worker dispatch (server-side): @microfox/ai-worker dispatch/dispatchWorker │
│          /dispatchQueue → POST {WORKER_BASE_URL}/workers/trigger or         │
│          /queues/{id}/start                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FOXHUB CONSOLE (console.microfox.app)                                       │
│  - org/project CRUD        → cicd /api/organizations, /api/projects         │
│    (called directly from the BROWSER via NEXT_PUBLIC_CICD_BASE_URL)         │
│  - deployments tab         → cicd /api/deployments?projectId=…              │
│  - step logs               → cicd /api/deployments/{id}/logs                │
│  - local deploy from UI    → cicd /api/deployments/local (zip upload)       │
│  - observability tab       → Foxhub /api/aws/lambda/{logs,analytics}        │
│    (CloudWatch FilterLogEvents with Foxhub's server-side AWS creds;         │
│     function list comes from project.urlsMap[].service.providerConfig)      │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Runtime execution flow (single worker)

1. **Dispatch** — app calls `agent.dispatch(input, opts)` or `dispatchWorker(id, input, opts)`.
   - *Local dev (preferred since Plan F)*: run `ai-worker dev` and point
     `WORKER_BASE_URL=http://localhost:4100` at it — the app uses the normal REMOTE path against
     a local server (in-process workers, in-memory queue, local file-persisted job store with
     `WORKER_DATABASE_TYPE=local`; the boilerplate's `localDevAdapter` proxies app-side store
     reads/writes to the server's `/dev-store` API, so the whole UI works without Redis/Mongo).
   - *Inline local mode* (legacy: `NODE_ENV=development` && `WORKERS_LOCAL_MODE!=='false'`, or
     `mode:'local'`): handler runs **inline in the Next.js process**, with a local job store
     resolved via `@/app/api/workflows/stores/jobStore` import (fragile — see bugs) or HTTP
     fallback.
   - *Remote mode*: zod-parse input → `POST {WORKER_BASE_URL}/workers/trigger` with
     `{ workerId, body: SQSMessageBody }` (+ optional `x-workers-trigger-key`).
2. **Trigger Lambda** resolves the worker's SQS queue URL (env `WORKER_QUEUE_URL_<ID>` →
   fallback `GetQueueUrl` on `{groupService}-{workerId}-{stage}`) and `SendMessage`s the body.
3. **Worker Lambda** (`createLambdaHandler`):
   - idempotency check (skip if job already terminal in store)
   - upsert job (status `queued`), then `running`
   - build `ctx`: `jobId, workerId, userId, jobStore, logger, dispatchWorker, reportTokenUsage,
     getTokenBudget, retryContext`
   - run handler (optionally inside `executeWithRetry`)
   - `outputSchema.parse(result)` → job `completed` (or `failed` + error)
   - optional webhook POST (fire-once, non-fatal on failure)
4. **Status** — app polls `GET /api/workflows/workers/:workerId/:jobId` which reads the same
   job store the Lambda wrote (both sides have Mongo/Redis adapters with the same schema).

## Queue execution flow (multi-step + HITL + loop)

1. `dispatchQueue(queueId, input)` → `POST /queues/{id}/start` → queue-starter Lambda upserts a
   `queue_jobs` doc (steps[0] queued) and sends the first worker's SQS message with
   `input.__workerQueue = { id, stepIndex: 0, initialInput, queueJobId }`.
2. Each worker in a queue is wrapped by `wrapHandlerForQueue` (generated entrypoint), which:
   - on **HITL resume** (`__hitlInput` present): loads previous outputs, calls the step's
     `resume(ctx)` via the generated registry, replaces `params.input` with the merged domain input
   - strips all envelope keys (`__workerQueue`, `__hitlInput`, `__hitlDecision`, `__hitlPending`,
     `hitl`) so the user handler sees clean domain input
   - runs the handler (with optional per-step retry)
   - **loop check**: if the step defines `loop.shouldContinue` and it returns true (and
     `iterationCount < maxIterations-1`), append a new steps[] entry, mark current complete,
     re-dispatch the same worker with `arrayStepIndex` tracking the appended position (or pause
     as `awaiting_approval` if `requiresApproval`)
   - otherwise **chain advance**: append next step → mark current complete → compute next input
     via `invokeChain` (step's `chain` fn / `'passthrough'` / `'continueFromPrevious'`) →
     dispatch next worker, or store pending input + mark `awaiting_approval` if the next step
     has `requiresApproval`
3. **HITL approve**: app `POST /api/workflows/queues/:queueId/approve` → marks the step running
   (before dispatch, to avoid a store race) → `dispatchWorker(stepWorkerId,
   { ...pendingInput, __hitlInput, __hitlDecision }, { jobId: storedWorkerJobId })`.
4. Queue job completion: `updateQueueJobStepInStore` marks the whole queue `completed` when the
   completed step is the **last array entry** — hence the strict append-before-complete ordering
   in the runtime.

## Identity & naming conventions

| Thing | Convention |
| --- | --- |
| Service name | `p-{projectId stripped of '-', first 15 chars}` (+ `-{group, max 12}` for non-default groups; `core` reserved) |
| SQS queue | `{service}-{workerId}-{stage}` (+ `-dlq-` for DLQ) |
| Lambda fn | `worker{CamelCase(workerId)}`, `queue{CamelCase(queueId)}` |
| Queue URL env | `WORKER_QUEUE_URL_{WORKER_ID with - → _, uppercased}` |
| Job ids | `job-{Date.now()}-{rand}`; queueJobId == first worker's jobId |
| Mongo collections | `worker_jobs`, `queue_jobs` (db default: `worker` for jobs, **`mediamake`** for queue jobs — inconsistent, see bugs) |
| Redis keys | `worker:jobs:{jobId}` (+ `:internal` list), `worker:queue-jobs:{queueJobId}`; TTL 7d default |

## Environment variable matrix

| Variable | Read by | Purpose |
| --- | --- | --- |
| `WORKER_BASE_URL` | ai-worker client, registry | Base URL of deployed workers service; `/workers/trigger`, `/workers/config`, `/queues/{id}/start` derived from it (legacy: `WORKERS_TRIGGER_API_URL`, `WORKERS_CONFIG_API_URL`) |
| `WORKERS_API_KEY` | client + registry + all generated Lambdas | **Unified** worker-endpoint secret (SEC-4). Resolved at `push` and written into `env.json`; required by trigger/config/queue handlers (timing-safe). Falls back to `sha256('microfox-workers:'+projectId)` when unset. |
| `WORKFLOW_INTERNAL_SECRET` | consumer app routes + runtime `sendWebhook` | Shared secret authorizing Lambda→app callbacks (SEC-5). Sent as `x-workflow-secret`; verified by the mutating workflow routes. **Falls back to `WORKERS_API_KEY`** when unset, so one secret can cover both surfaces. `WORKFLOW_` prefix → carried into `env.json`. |
| `WORKFLOW_ALLOW_PUBLIC` | consumer app routes | `'true'` disables the SEC-5 auth gate (local-dev only, logs a warning). |
| `WORKERS_TRIGGER_API_KEY` | client + trigger/queue-start Lambdas | Legacy per-endpoint secret (`x-workers-trigger-key`); still honored, takes precedence over the derived key |
| `WORKERS_CONFIG_API_KEY` | registry + workers-config Lambda | Legacy per-endpoint secret (`x-workers-config-key`); still honored |
| `WORKER_DATABASE_TYPE` | handler, queueJobStore, boilerplate stores, CLI dep filter | `mongodb` \| `upstash-redis` (default) \| `local` (Plan F, dev only — never a fallback; compile warns if it would ship) |
| `AI_WORKER_LOCAL` + bridge global | handler dispatch seam | `'1'` + `globalThis.__AI_WORKER_LOCAL_BRIDGE__` (both set only by `ai-worker dev`) route dispatches to the in-process queue instead of SQS |
| `AI_WORKER_LOCAL_STATE_PATH` | localJobStore | Local store persistence file (default `.microfox/dev-state.json`) |
| `MONGODB_URI` / `DATABASE_MONGODB_URI` / `MONGODB_WORKER_URI` | job stores | Mongo connection (3 aliases) |
| `MONGODB_DB` / `DATABASE_MONGODB_DB` / `MONGODB_WORKER_DB` | job stores | DB name |
| `WORKER_UPSTASH_REDIS_REST_URL/TOKEN` (+ `UPSTASH_REDIS_*` fallbacks) | redis stores | Upstash REST |
| `WORKER_JOBS_TTL_SECONDS` (+ 2 fallbacks) | redis stores | Job TTL |
| `WORKERS_LOCAL_MODE` | createWorker dispatch | `'false'` disables dev inline mode |
| `WORKER_QUEUE_URL_<ID>` | Lambdas (dispatchWorker), trigger/starter | Direct queue URL injection (CLI-generated) |
| `WORKER_SERVICE_NAME` | handler dispatchWorker fallback | SQS GetQueueUrl name construction |
| `WORKER_JOB_STORE_MODULE_PATH` / `WORKER_JOB_STORE_URL` | local mode | Custom job store module / HTTP base |
| `AI_WORKER_QUEUES_DEBUG=1`, `DEBUG_WORKER_QUEUES=1`, `DEBUG`/`WORKER_DEBUG` | runtime | Debug logging |
| `STAGE` / `ENVIRONMENT` | Lambdas, CLI | Stage resolution (Plan E: fixed set `prod|staging|dev`; compile honors `--stage` > `STAGE` env > `prod` — the old force-prod override is gone) |
| `MICROFOX_PROJECT_ID` | CLI | projectId fallback for service naming |
| cicd: `AGENT_AWS_ACCESS_KEY_ID/SECRET/REGION` | ServerlessService | Platform AWS deploy creds |
| cicd: `ENCRYPTION_KEY`, `TASK_UPSTASH_REDIS_*`, `BASE_SERVER_URL` | ServerlessService | **Injected into every deployed Lambda's env** (see security) |
| cicd: `MAX_CONCURRENT_DEPLOYMENTS` | DeploymentQueueService | default 3 |
| cicd: `MICROFOX_SERVICE_KEY` | auth middleware/routes | Static service key (coarse gate); also set in the CLI env (or its baked default) |
| cicd: `CICD_AUTH_ENFORCE` | auth middleware | `'true'` flips auth from warn-only to enforcing (reject + ownership) |
| cicd: `SUPABASE_URL` / `SUPABASE_ANON_KEY` | AuthService | Verify console/approval Supabase JWTs |
| cicd: `MICROFOX_APP_URL` | AuthService | Base for the `/cli-auth?code=` verification URL (default `https://microfox.app`) |
| cicd: `MAX_ARCHIVE_MB` / `MAX_EXTRACTED_MB` / `MAX_ENTRIES` | archive.utils | Plan C upload/extraction caps (200 / 1024 / 50000) |
| CLI: `MICROFOX_TOKEN` / `MICROFOX_SERVICE_KEY` | microfox `utils/auth` | CI token override / service-key override; interactive token saved to `~/.microfox/credentials.json` |
| Foxhub: `NEXT_PUBLIC_CICD_BASE_URL` / `NEXT_PUBLIC_CONSOLE_API_BASE` | console-api.ts | cicd API base (browser-side!) |

## Config files

- **`microfox.json`** (project root) — `{ projectId, publish: { subdomain, handles }, deployment:
  { apiMode, apiVersion, port }, worker: { includeNodeModules, excludeNodeModules, externalDeps,
  groups: { [g]: {...} } }, env: { mode, files, include, exclude, groups } }` (Plan D block —
  see 03). Copied into `.serverless-workers/` by the CLI.
- **`microfox.config.ts`** — `StudioConfig.workflowSettings.{jobStore, deploymentConfig}`;
  fallback source for the same data (CLI evaluates it with esbuild + `new Function`).
- **`.serverless-workers/`** — generated output; `deployments.json` records deployment ids per
  group (read by `microfox status`).

## Trust boundaries (summary — full analysis in 04-security-issues.md)

0. Anyone → consumer-app mutating workflow routes: now gated (SEC-4/SEC-5 fixed) on a user
   session or the `WORKFLOW_INTERNAL_SECRET`; HITL `approve` input is schema-validated.
1. Browser → Foxhub API routes: `/api/*` **bypasses session middleware** (the
   `/api/aws/lambda/{logs,analytics}` routes now enforce auth + project scoping at the route
   level — SEC-3 fixed; other `/api/*` routes still need an audit).
2. Browser/CLI → cicd API: now gated by `authMiddleware` (service key + CLI-token/Supabase-JWT
   identity + per-resource ownership) and archive intake is hardened (SEC-2 fixed, Plan A + C) —
   **phased via `CICD_AUTH_ENFORCE`** (warn-only until rollout completes).
3. Anyone → deployed API GW endpoints: now key-gated by default (SEC-4 fixed) — a stable
   `WORKERS_API_KEY` (or projectId-derived secret) is required unless deployed `--allow-public`.
4. cicd → AWS: single shared platform account, user-controlled `serverless.yml` (IAM!) and
   `npm install` (postinstall scripts) run server-side.
5. Worker Lambda → job stores: direct DB credentials in Lambda env.
