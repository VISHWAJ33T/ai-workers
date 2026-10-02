# 04 — Security Issues (All Repos)

Ordered by severity. Each issue lists the affected code, impact, and a concrete fix.

> Severity scale: **S0** = drop everything, **S1** = fix this sprint, **S2** = fix soon,
> **S3** = hardening.

---

## S0

### SEC-1 — Real secrets sit in the working tree (NOT a git leak) <a id="sec-1"></a>

> **Correction (2026-06-12):** the original audit claimed these files were committed. They are
> **not** — `examples/root/.gitignore` contains `.env*` (line 34) and `.serverless-workers`
> (line 46), and `git ls-files` confirms neither is tracked. **Downgraded from a git-leak S0 to
> a working-tree hygiene note.** No history purge is needed.

**Where:** `examples/root/.env`, `examples/root/.serverless-workers/*/env.json` — present on
disk, gitignored.

**What's in them:** live MongoDB connection string (DigitalOcean `doadmin` creds), Upstash Redis
REST URL + token, the live API Gateway `WORKER_BASE_URL`.

**Residual risk:** these are real production credentials in a working directory. They are safe
from accidental `git add` thanks to the ignore rules, but they can still leak via screen-shares,
backups, `zip`/upload of the folder, or an `--force`/`-f` add. They are also the credentials the
CLI copies into generated `env.json` (the input to deployment), so they remain operationally
sensitive.

**Fix (low urgency now that it's not in git):**
1. Treat the committed-elsewhere copies as the real exposure surface; rotate only if you suspect
   the working tree was ever shared. No git history action required.
2. Keep an `.env.example` with placeholders so contributors don't paste real values.
3. CLI hardening (see SEC-10): the CLI should append `.serverless-workers/` to the project
   `.gitignore` automatically and warn if `.env` is tracked, so other consumer apps don't
   re-introduce the leak that this repo already avoids.

### SEC-2 — cicd/v2 API is completely unauthenticated (deploy = RCE on the build server) ✅ FIXED (2026-06-15)

> **Status:** implemented — full [plan-A](./plans/plan-A-cli-auth.md) (device login) + [plan-C](./plans/plan-C-archive-validation.md) (archive hardening).
>
> **Auth (Plan A):** new cicd Mongo models `CliToken` + `CliDeviceSession`, an `AuthService`
> (device-authorization flow + token lifecycle + Supabase JWT verification), `/api/auth/cli/*`
> routes, and an `authMiddleware` applied to `/api/{deployments,projects,organizations}`. It
> validates the static `MICROFOX_SERVICE_KEY`, resolves identity from a CLI token *or* a Supabase
> JWT (`req.auth`), and per-route ownership checks (`assertProjectAccess`/`assertOrgAccess`) gate
> deploy/read/delete and bind created resources to the authed user. Deployments now record the real
> `triggeredBy`. **Phased via `CICD_AUTH_ENFORCE`** (warn-only until the new CLI + Foxhub releases
> are live, then flip to enforce). The `microfox` CLI gained `login`/`logout` (device flow → token
> in `~/.microfox/credentials.json`) and sends `Authorization` + `x-microfox-service-key` on
> push/status; Foxhub gained `/cli-auth` (browser approval) and a "Triggered by" column.
>
> **Archive intake (Plan C):** `/api/deployments/local` now enforces a multer size cap
> (`MAX_ARCHIVE_MB`), a zip magic-byte check, and `validateAndExtractArchive` (zip-slip-safe
> extraction, symlink rejection, entry-count + uncompressed-size caps), with partial-extraction
> cleanup + FAILED marking on violation.
>
> **Remaining (tracked separately):** build sandboxing / `--ignore-scripts` (SEC-2 fix #4, the
> bigger RCE surface) and per-project IAM scoping (fix #5) are **not** done here. Flip
> `CICD_AUTH_ENFORCE=true` only after the CLI + Foxhub rollout; configure `MICROFOX_SERVICE_KEY`,
> `SUPABASE_URL`/`SUPABASE_ANON_KEY`, and `MICROFOX_APP_URL` on cicd.

**Where:** `cicd/v2/src/app.ts` (no auth middleware), all routes in
`routes/{deployment,project,organization}.routes.ts`.

**What anyone on the internet can do:**
- `POST /api/deployments/local` with header `x-project-id: <any existing project id>` and a zip →
  the server **extracts the zip, runs `npm install` (postinstall scripts!) and `npx serverless
  deploy`** with the platform's AWS credentials. That is arbitrary code execution on the cicd VM
  *and* arbitrary CloudFormation/IAM in the platform AWS account (the attacker controls
  `serverless.yml`, including `iam.role.statements`).
- `POST /api/projects` — create projects/orgs with arbitrary `ownerId`/`organizationId`.
- `DELETE /api/projects/:id`, `DELETE …/groups/:group` — **delete anyone's deployed stacks**.
- `GET /api/deployments?projectId=…`, `GET /api/deployments/:id/logs` — read any project's
  deployment logs (which include env-var names, stack output, service URLs).
- `POST /api/deployments/:id/stop` — kill anyone's in-flight deployment.

Project ids are guessable/enumerable via the unauthenticated list endpoints, and one is also
committed in this repo (`p-2bd2b78d320d48d…` appears in `examples/root/.serverless-workers/`).

**Fix (incremental):**
1. Immediate: require a platform API key (header) on every cicd route; put the key only in
   Foxhub's server-side env and the CLI's authenticated session — never `NEXT_PUBLIC_*`.
2. Real fix: issue per-user tokens from Foxhub auth (the console already has sessions); cicd
   validates the token and checks project ownership (`project.ownerId` /
   `organization.members`) before deploy/read/delete. The microfox CLI should do a device-code
   or token login (`microfox login`) instead of relying on a public endpoint.
3. Validate the uploaded archive: size limits, path-traversal-safe extraction (zip-slip — verify
   `extract-zip` config), reject symlinks.
4. Sandbox builds: run install/deploy in a container or at least a separate unprivileged user;
   `npm install --ignore-scripts` plus an allowlist if feasible.
5. Scope AWS: per-project IAM roles (or at minimum a deploy role with a permission boundary),
   instead of one god-credential (`AGENT_AWS_*`) shared by all tenants.

### SEC-3 — Foxhub `/api/aws/lambda/{logs,analytics}` are unauthenticated and unscoped ✅ FIXED (2026-06-15)

> **Status:** implemented. Both routes now call a shared
> `app/api/aws/lambda/scope.ts#resolveProjectLambdaScope(body)` which: (1) requires a Supabase
> session (`createClient().auth.getUser()` — independent of the `/api` middleware bypass);
> (2) requires a `projectId` in the body; (3) verifies the user is the project `ownerId` or a
> member/owner of its organization (fetched server-side from cicd); and (4) resolves the allowed
> function list **server-side** from `project.urlsMap[].service.providerConfig.resources`,
> restricted to the `p-{projectId}` service-name prefix. **Client-sent function names are ignored.**
> The console `aws-context` now sends `projectId`.
>
> **Residual:** the server-side ownership check fetches the project/org from cicd, which is itself
> unauthenticated until [plan-A SEC-2](./plans/plan-A-cli-auth.md) lands — but that only affects the
> *integrity of the project record*, not this route's auth (a logged-in user still can't read another
> project's functions). The broader `/api/*` middleware bypass is unchanged (route-level auth is the
> correct fix); audit other `/api/aws/*` routes similarly.

**Where:** `Foxhub/apps/web/app/api/aws/lambda/logs/route.ts` (+ `analytics`),
`utils/middleware/main.ts` (`pathname.startsWith('/api')` → `NextResponse.next()` — API routes
bypass session handling entirely; the routes themselves do no auth either).

**Impact:** anyone can POST `{ items: [{ name: '<any-lambda-name>', region }] }` and read
**any Lambda's CloudWatch logs in the platform account** (worker logs contain job inputs/outputs,
user ids, sometimes secrets that handlers log). Cross-tenant and platform-internal functions
alike. Same for invocation/error metrics via the analytics route.

**Fix:** require a session in these routes; resolve the allowed function list **server-side**
from the project's `urlsMap` after checking the user owns the project — never trust client-sent
function names. Add a log-group name allowlist prefix (`/aws/lambda/p-{projectId}…`).

---

## S1

### SEC-4 — Deployed worker endpoints are public by default ✅ FIXED (2026-06-15)

> **Status:** implemented per [plan-B](./plans/plan-B-worker-endpoint-auth.md). The CLI now
> resolves a **stable** secret at `push` time (precedence: `WORKERS_API_KEY` → legacy
> `WORKERS_TRIGGER_API_KEY`/`WORKERS_CONFIG_API_KEY` → `sha256('microfox-workers:'+projectId)`),
> writes it into every `env.json`, and the generated `workers-trigger` / `workers-config` /
> queue-starter handlers **require** it with a `crypto.timingSafeEqual` compare (SEC-9 also fixed).
> When no secret resolves, deploy is **public by default** with a loud warning; `--require-auth`
> fails the build instead, `--allow-public` silences the warning. `?debug=1` output on
> `/workers/config` is now gated behind a configured key. The consumer side
> (`@microfox/ai-worker` `client.ts` + `examples/root` `registry/workers.ts`) sends the resolved
> key via `resolveWorkersTriggerKey()` / `resolveWorkersConfigKey()`, and `boilerplate` writes a
> random `WORKERS_API_KEY` into `.env` once so new projects are non-public by default.
>
> **Caveat (unchanged from the plan):** the projectId-derived path is a *soft* gate — for the
> consuming app to send the same key it must have `MICROFOX_PROJECT_ID` set to the same projectId
> used at deploy time (or, preferably, an explicit shared `WORKERS_API_KEY`). The raw projectId is
> never the header value (only its sha256). Rotation = change `WORKERS_API_KEY` + redeploy + update
> the app env.

**Where:** generated `workers-trigger.js`, `workers-config.js`, queue starter handlers
(`packages/ai-worker-cli/src/commands/push.ts`); API keys only checked **if the env var is
set** (`WORKERS_TRIGGER_API_KEY`, `WORKERS_CONFIG_API_KEY`). CORS `*`. The committed example
deployment has no keys set.

**Impact (no key configured):**
- `POST /workers/trigger` — run any worker with attacker-controlled input (cost abuse, data
  injection into job stores, triggering side-effects of handlers).
- `POST /queues/{id}/start` — start any pipeline.
- `GET /workers/config` — enumerate workers, their **input JSON Schemas**, queue URLs, queue
  definitions; `?debug=1` leaks attempted queue names and AWS error messages.

**Fix:** make the CLI **generate a key by default** (write into env.json + print it once) and
require it in the handlers; fail deploy with a loud `--allow-public` escape hatch if someone
really wants keyless. Compare keys with `crypto.timingSafeEqual`. Drop `?debug` output or gate
it behind the key.

### SEC-5 — Example-app mutating routes have no auth; HITL approve accepts arbitrary input ✅ FIXED (2026-06-15)

> **Status:** implemented in the boilerplate (so every consumer app inherits it).
> `auth.ts` now exports `authorizeWorkflowRequest(req)`; **every mutating POST route**
> (`workers/:id` trigger + `update`/`webhook`/`job`, `queues/:id` trigger + `update`/`webhook`/
> `approve`) is gated on it. A request is authorized iff: (1) `getClientId()` resolves a user,
> (2) it carries the internal shared secret `x-workflow-secret` === `WORKFLOW_INTERNAL_SECRET`
> (the deployed runtime's `sendWebhook` now sends this header when the env var is set), or
> (3) `WORKFLOW_ALLOW_PUBLIC === 'true'` (explicit, warned local-dev opt-out). Otherwise **401** with
> an actionable message — secure by default. The HITL `approve` handler now validates the reviewer
> `input` against the step's `hitl.inputSchema` (resolved locally via the registry's new
> `getStepHitlInputSchema`) and dispatches the **parsed** value, so an approver can no longer inject
> arbitrary fields into the resumed step (fails closed on validation error). GET (status/list/
> history) routes remain open for polling — see the read-leak note under SEC-3 for scoping reads.

**Where:** `examples/root/app/api/workflows/...` (and the boilerplate templates, so every
consumer app inherits it): `workers/:id/update`, `workers/:id/webhook`, `workers/:id/job`,
`queues/:id/update`, `queues/:id/webhook`, and especially `queues/:id/approve`.
`auth.ts#getClientId` is a stub returning `undefined` and is only used to *attach* a userId,
never to gate anything.

**Impact:** anyone who can reach the Next.js app can mark jobs completed/failed, corrupt queue
state, and — worst — **approve a HITL step with arbitrary reviewer `input`**, which the route
forwards into `dispatchWorker(...)`, i.e. attacker-controlled execution of the next pipeline
step.

**Fix:** in the boilerplate, gate every mutating route on `getClientId()` returning a user (or
an internal shared secret for Lambda→app callbacks), and document loudly that the stub must be
implemented. Validate reviewer `input` against the step's `hitl.inputSchema` server-side before
dispatch (the schema already exists!).

### SEC-6 — Webhooks have no authenticity (no HMAC)

**Where:** `packages/ai-worker/src/handler.ts#sendWebhook`; consumer webhook routes accept any
POST with a `jobId`.

**Impact:** forged completion callbacks (set any job to completed/failed with chosen output) —
even if SEC-5 is fixed by session auth, webhooks come from Lambda, not a user session, so
they'll likely stay open.

**Fix:** sign payloads (`x-worker-signature: hmac-sha256(body, WORKER_WEBHOOK_SECRET)`); verify
in the boilerplate webhook handlers; CLI generates the secret into both sides' env.

### SEC-7 — Platform secrets injected into every tenant Lambda

**Where:** `cicd/v2/src/services/serverless.service.ts#setEnvironmentVariables` — adds
`ENCRYPTION_KEY`, `TASK_UPSTASH_REDIS_REST_URL/TOKEN`, `BASE_SERVER_URL` from the **cicd
server's own environment** into the project's `env.json`, which becomes Lambda env for *every
function of every tenant*.

**Impact:** any deployed user code can read the platform's encryption key and task-queue Redis
credentials (cross-tenant task forgery, decryption of whatever `ENCRYPTION_KEY` protects).

**Fix:** stop injecting platform secrets into tenant Lambdas. If tenants need task-queue access,
mint scoped per-project credentials.

---

## S2

### SEC-8 — `env.json` puts every secret on every function (CloudFormation-visible)

`provider.environment: ${file(env.json)}` means each Lambda gets the full filtered `.env`
(Mongo URI, Upstash token, OpenAI/Anthropic keys…), visible in the Lambda console and CFN
templates to anyone with read access to the account. Prefer per-function env (the CLI already
does this for queue URLs), SSM/Secrets Manager references, and only the keys a worker's code
actually references (the CLI *already computes* `collectEnvUsageForWorkers` — use it per
function, not just as an env.json filter).

> **Partially mitigated by Plan D (2026-07):** the `env` block gives per-project and per-group
> `mode: 'explicit'` / `include` / `exclude` control (a group can now ship an exact key list),
> and `env.files: 'isolated'` prevents base-`.env` values from reaching staged builds at all.
> Per-function scoping and SSM references remain open. Plan E's Phase-3 server-side env store
> (AES-256-GCM encrypted at rest in cicd Mongo, injected at deploy time) also reduces what needs
> to live in the zip's env.json in the first place.

### SEC-13 — `ai-worker dev` `/dev-store` write surface (dev-only; flag R51, 2026-07-10)

The Plan F dev server exposes HTTP write endpoints over the local job store (the boilerplate's
`local` adapter needs them). Bounded by design: served ONLY when the server's store is `local`
(400 otherwise), gated by the same `WORKERS_API_KEY` rule as `/workers/trigger`, bound to a dev
box, and never part of a deployed stack. Residual risk: an open (`no key`) dev server on a
shared network lets anyone mutate local dev job state — same blast radius as the trigger route
itself. Keep `WORKERS_API_KEY` set if the port is exposed.

### SEC-9 — Non-constant-time API key comparisons ✅ FIXED (2026-06-15)

~~All generated handlers compare `providedKey !== apiKey` directly.~~ Fixed alongside SEC-4: the
trigger, config, and queue-starter handlers now use an inlined `timingSafeEqualStr` wrapper around
`crypto.timingSafeEqual` (length-checked, constant-time). The consumer-side dispatch helpers and
the example app registry resolve keys via the shared `resolveWorkers*Key()` helpers.

### SEC-10 — CLI writes plaintext secrets into a commit-prone directory

`.serverless-workers/env.json` (and per-group copies) duplicates `.env` secrets. The CLI should:
append `.serverless-workers/` to the project `.gitignore` automatically (it already writes other
files), and print a warning when `.env` itself is tracked by git.

### SEC-11 — Unpinned remote code at deploy time

- `ai-worker-cli` deploy: `npx microfox@latest push`, `serverless ^3.38.0`.
- cicd: builds run whatever the archive's `package.json` says.
Pin exact versions for the toolchain; use lockfiles in generated packages.

### SEC-12 — `new Function` evaluation of scanned source

`push.ts` (`scanQueues` schedule objects, `workerConfig` fallback extraction) and
`boilerplate`'s `microfox.config.ts` loader execute project-file snippets in-process. It's the
developer's own machine and their own code, so impact is limited, but combined with the regex
slicing it can execute *partial* expressions. Prefer the esbuild-stub evaluation path
everywhere (see improvement I-2).

---

## S3 (hardening)

- **CORS `*`** on all generated API handlers — restrict to the app origin once keys exist.
- **`x-client-id` header trust** (mediamake workers route): the route prefers
  `req.headers['x-client-id']` over the session — ensure middleware strips this header from
  external requests, otherwise quota checks are bypassable by header injection.
- **Error bodies leak internals**: trigger/config handlers return raw AWS error messages;
  workers route returns stack traces when `NODE_ENV=development`.
- **No rate limiting** anywhere (trigger, queue start, cicd deploy). Cost-abuse vector even with
  keys.
- **SQS message size**: inputs/outputs > 256 KB will fail dispatch; large outputs also bloat the
  job store (Mongo 16 MB doc limit for queue jobs that accumulate step outputs). Consider S3
  offloading for large payloads.
- **cicd `deleteFunction` / delete-worker** run `serverless remove` based on DB state — verify
  project ownership *and* service-name prefix (`p-{projectId}`) before removal to prevent
  cross-project deletions if records are tampered with.
