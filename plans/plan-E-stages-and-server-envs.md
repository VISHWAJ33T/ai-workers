# Plan E — Multi-stage deployments + server-side (console) env injection

**Status:** 🔨 Phases 1–4 coded 2026-07-09 (CLI stage plumbing + cicd stage dimension + durable
env store + Foxhub console stage UI/store editor), awaiting review (DEV_TASKS R40–R47, H31–H32).
Decisions D1–D3 ANSWERED — see DEV_TASKS: D1 = subdomain stage-path URLs `{sub}.microfox.app/{stage}`
via stage-prefixed urlsMap row paths (reroute Worker unchanged); D2 = console wins over zip; D3 =
fixed set prod|staging|dev. Remaining: monitoring-tab stage scoping + optional per-function store
editor + optional live→read-only "effective env" collapse (all noted in DEV_TASKS as follow-ups).
**Depends on Plan D** (env-file cascade + env selection) — landed 2026-07-09.
**Goal:** (1) the same project deploys to multiple AWS stages (`prod`, `staging`, `dev`, …) — the
architecture currently force-assumes `prod` end-to-end. (2) Envs managed in the console become a
durable server-side store that cicd **injects at deploy time**, supporting three layers:
**whole-project**, **per-group**, and **per-function** additional envs — so secrets set in the
console never need to exist in the developer's `.env` or travel inside the zip, and (fixes a real
bug) **no longer get clobbered by the next redeploy**.

Touches `ai-router/packages/ai-worker-cli`, `microfox/packages/cli`, `cicd/v2`, `Foxhub/apps/web`.

---

## Current state (verified in code, 2026-07-08)

### Stage is hardcoded to prod in FOUR places
1. CLI `compile.ts:3613` — `envStage = hasRootMicrofoxJson || resolvedMicrofoxConfig ? 'prod' : stage`
   ("backward-compatible behavior": any microfox project ignores `--stage` for env purposes).
2. CLI multi-group env.json (~3554) — `{ ENVIRONMENT:'prod', STAGE:'prod', NODE_ENV:'prod' }` inline.
3. cicd `local-deployment-orchestrator.ts:52` — `this.stage = 'prod'` hardcoded; used for
   `serverless deploy --stage`, agent URL (`${baseUrl}/${this.stage}`), Lambda names
   (`${serviceName}-${this.stage}-${name}`).
4. `microfox push` sends NO stage at all (headers: `x-project-id`, `x-deployment-group` only).

### What already supports stages (good news)
- Generated `serverless.yml` threads `${opt:stage, env:ENVIRONMENT, '<stage>'}` everywhere
  (functions, SQS queue names, DLQs). CloudFormation stack name = `<service>-<stage>` → deploying
  the same service with a different `--stage` creates a PARALLEL stack naturally. Queue-name
  conventions (`${serviceName}-${workerId}-${stage}`) and the generated trigger/queue handlers
  already read stage from `requestContext.stage` / `ENVIRONMENT` / `STAGE` at runtime.
- cicd concurrency lock is per project+group (needs +stage).

### Server-side envs today (the redeploy-clobber bug)
- `env-secrets.service.ts`: **the live Lambda config IS the store** — console env edits do
  `UpdateFunctionConfiguration` per function (preserving `PLATFORM_KEYS`). There is NO durable copy.
- Deploy path (`serverless.service.ts setEnvironmentVariables`): reads the zip's `env.json`, adds
  platform keys (`ENCRYPTION_KEY`, `TASK_UPSTASH_*`, `BASE_SERVER_URL`), runs
  `serverless deploy` → **replaces every function's env with zip content. Console-set envs are
  silently LOST on every redeploy.**
- urlsMap rows are merged by `(type, group)` (`mergeUrlsMapByTypeAndGroup`) — no stage dimension.
  Per-group env slots in the console key by `envSlotKey(group)`.

## Target state

### A. Stage as a first-class dimension WITHIN a project (decision: not separate projects)
- One project → N stages; each (group, stage) pair is its own CloudFormation stack (free, via
  `--stage`). Stage list is free-form strings, default `prod`.
- **Subdomain routing (v1 decision, keep simple): only `prod` writes the `subdomains:{sub}`
  routemap.** Non-prod stages are reached via their raw API Gateway URL shown in the console.
  (Optional later: `{stage}--{sub}.microfox.app` — requires reroute-Worker changes; explicitly out
  of scope now. NOTE: `subdomains:*` must stay in Redis for the reroute Worker regardless.)

### B. Plumbing the stage through
1. **CLI (`microfox` + `ai-worker-cli`)**: `microfox deploy --stage staging` (also on
   `compile`/`push`) → forwarded to `ai-worker-cli compile --stage` AND sent to cicd as
   `x-deployment-stage` header + `publish.stage`. Default `'prod'` everywhere → old CLIs and
   omitted flags behave exactly as today.
2. **CLI compile**: DELETE the force-prod override (`compile.ts:3613`) and the multi-group inline
   'prod' → `envStage = stage`. With Plan D, `.env.{stage}` files feed stage-specific values.
3. **cicd**: `Deployment` model + `deployments/local` route gain `stage` (default `'prod'`);
   orchestrator uses `deployment.stage` (drop line 52); concurrency key → project+group+stage;
   `deleteFunction` paths take the stage. Old deployments/records without stage read as prod.
4. **urlsMap**: rows gain optional `stage?: string` (absent ⇒ `'prod'` — zero migration for
   existing data); merge becomes by `(type, group, stage)`; `agentUrl = ${baseUrl}/${stage}`.
   Project-level `urlsMap` consumers (console Infrastructure/Run&test/History/Env, env-secrets
   `listProjectFunctions`) filter by a selected stage.
5. **Console**: a stage selector in the project header (populated from distinct stages in urlsMap,
   default prod). All monitoring + env tabs scope to the selected stage. Deployments tab shows the
   stage column.

### C. Server-side env store + deploy-time injection (per-function / per-group / project-wide)
1. **New cicd Mongo model `projectEnvs`** — one doc per `(projectId, stage)`:
   ```ts
   {
     projectId, stage,                    // unique compound index
     project:   { KEY: encryptedValue },  // whole-project layer
     groups:    { [group]: { KEY: encryptedValue } },
     functions: { [lambdaName]: { KEY: encryptedValue } },
     updatedAt, updatedBy
   }
   ```
   Values encrypted at rest with the existing `ENCRYPTION_KEY` (AES-256-GCM helper; cicd already
   holds the key). RBAC: reuse existing env permissions (`env.view`/`env.manage`).
2. **Deploy-time merge** (in `serverless.service.setEnvironmentVariables`, per group deploy):
   `zip env.json` ← `projectEnvs.project` ← `projectEnvs.groups[group]` ← platform keys
   (platform keys always win; console layers override zip — the console is where operators fix
   things without redeploying from a laptop). Result written to `env.json` → `serverless deploy`
   applies it stack-wide, exactly like today.
3. **Per-function layer**: applied as a **post-deploy pass** using the existing
   `setFunctionEnv()` per function that has overrides (serverless deploy sets provider-level env;
   rewriting per-function `environment` blocks in the generated YAML from cicd is more invasive —
   post-deploy UpdateFunctionConfiguration reuses proven code).
4. **Console env editor rewire**: writes go to the `projectEnvs` store **and** are pushed live via
   the existing `setFunctionEnv` fan-out (immediate effect, durable across redeploys). Reads show
   the store as the editable truth + live Lambda values as read-only "effective env" per function.
   One-time migration UX: an "Import current live env" button per group/function snapshots today's
   live (owner-visible) vars into the store, so nothing is lost at cutover.
5. `.env`-zip path keeps working standalone (no console) — the store layers are additive.

## Design decisions

- **Stage within project** (not project-per-stage): keeps one console, one member list, one
  routemap identity; the alternative is crude and doubles RBAC/billing surface.
- **Console layer OVERRIDES zip env.json** on key conflict: server-set values are the operator's
  explicit intent; zip values are whatever was on a dev laptop. Logged per-key at deploy
  (key names only, never values).
- **Store is per-stage by construction** — prod secrets and staging secrets never share a doc.
- envs move to Mongo, NOT Redis (consistent with the 2026-07 cicd storage direction; Redis keeps
  only `subdomains:*`).
- Free-form stage names; validate `[a-z0-9-]{1,20}` (they embed into CFN stack + Lambda + SQS names).

## Rollout order (matters)

1. Deploy cicd that ACCEPTS `stage` (defaulting `'prod'`) + urlsMap stage field + env store +
   deploy merge. Old CLIs keep working (no stage sent ⇒ prod).
2. Publish `@microfox/ai-worker-cli` (force-prod removed) then `microfox` CLI (stage forwarding) —
   same publish-order rule as R38.
3. Ship Foxhub console (stage selector + env editor rewire + import-live action).
4. Users press "Import current live env" once per project (or we accept lazy adoption: the first
   console env SAVE after cutover writes the store).

## Phases

1. **CLI stage plumbing** — compile force-prod removal, deploy/push/compile `--stage`, header +
   publish.stage. Small; independently shippable (cicd default absorbs it).
2. **cicd stage dimension** — deployment model/route, orchestrator, concurrency key, urlsMap
   `(type, group, stage)` merge + consumers.
3. **cicd env store + deploy-time injection** — model, crypto helper, merge in
   setEnvironmentVariables, per-function post-deploy pass, REST routes (get/put layers).
4. **Console** — stage selector; env tab: store-backed editor + effective-env view + import-live.

## Review flags (Anti-Vibe)

- ⚠ **Deploy-time env merge changes what lands in EVERY Lambda's env** — destructive-adjacent;
  manual review + a staging-stage dry run before touching anyone's prod.
- ⚠ **Encryption of stored env values** — key handling code must be reviewed (reuses
  ENCRYPTION_KEY; wrong nonce/reuse patterns are a classic footgun).
- ⚠ **urlsMap merge-key change** `(type,group)` → `(type,group,stage)` — the same function that
  bit us before with group slots; needs the same careful test matrix (redeploy existing prod
  project must produce byte-identical urlsMap).
- ⚠ Console env editor changes source-of-truth semantics — needs explicit user sign-off on the
  "console overrides zip" precedence before build.

## Open decisions for the user (answer before Phase 2)

- [ ] Non-prod stage URLs: raw API GW URL only (v1 proposal) — OK, or is subdomain routing for
      staging a launch requirement?
- [ ] Precedence confirm: console/store env WINS over zip env.json on conflict?
- [ ] Free-form stage names vs a fixed set (`prod|staging|dev`)?
