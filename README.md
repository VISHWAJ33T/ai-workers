# AI Worker — System Documentation & Analysis

> Generated 2026-06-12 from a full read of `@microfox/ai-worker`, `@microfox/ai-worker-cli`,
> `examples/root`, and the connected repos (`mediamake`, `cicd/v2`, `Foxhub/apps/web` console,
> `microfox/packages/cli`).

This folder documents the **entire background-worker ecosystem** end to end and collects every
bug, security issue, improvement, and optimization found during the audit.

## Contents

| File | What it covers |
| --- | --- |
| [01-architecture.md](./01-architecture.md) | End-to-end system architecture: every repo, every hop from `createWorker()` to a running Lambda, env-var matrix, data stores, deployment pipeline |
| [02-package-ai-worker.md](./02-package-ai-worker.md) | Deep documentation of the runtime package: modules, job lifecycle, queue/HITL/loop state machine, retry & token budget subsystems |
| [03-package-ai-worker-cli.md](./03-package-ai-worker-cli.md) | CLI deep-dive: `push` build pipeline, generated artifacts, `boilerplate`, `new`, multi-group layout |
| [04-security-issues.md](./04-security-issues.md) | **All security findings across all repos**, ordered by severity, with concrete fixes |
| [05-bugs.md](./05-bugs.md) | Correctness bugs with `file:line` references and suggested fixes |
| [06-improvements.md](./06-improvements.md) | DX, performance, architecture improvements and optimizations |
| [07-boilerplate-sync.md](./07-boilerplate-sync.md) | The boilerplate template sync workflow (new `sync-boilerplate` script) + the mediamake ↔ examples/root divergence report |
| [08-microfox-reroute.md](./08-microfox-reroute.md) | The Cloudflare Worker that resolves `*.microfox.app` subdomains to deployed URLs via the Upstash routemap — how it fits the pipeline, and an investigation of the recent outage |
| [plans/](./plans/) | Detailed implementation plans: security work (0/A/B/C — all implemented) and the 2026-07 feature wave (D: env config ✅ coded, E: stages + server env store ✅ coded, F: local dev server ✅ coded + tested) |

## TL;DR of the audit

**The system solves a real problem** (long-running AI agents beyond Vercel timeout limits, with a
genuinely good colocated-worker DX) and the runtime design is sound. The biggest risks are:

1. ~~Live credentials committed to this repo~~ — **corrected**: `.env*` and `.serverless-workers`
   are gitignored and untracked; the secrets exist only in the working tree (hygiene note, not a
   git leak). See [04-security-issues.md SEC-1](./04-security-issues.md#sec-1).
2. ~~🚨 **Nearly every HTTP surface in the pipeline is unauthenticated**~~ — **addressed
   (2026-06-15).** The deployed trigger/config/queue endpoints now require a stable key (SEC-4),
   the example app's mutating routes require a user/internal-secret + validate HITL input (SEC-5),
   the cicd/v2 API is gated by service-key + CLI-token/JWT auth and ownership with hardened archive
   intake (SEC-2, Plan A+C, warn-only until `CICD_AUTH_ENFORCE=true`), and Foxhub's
   `/api/aws/lambda/{logs,analytics}` require a session + project scoping (SEC-3). Remaining
   hardening: build sandboxing + per-project IAM (SEC-2 #4/#5), HMAC webhooks (SEC-6).
3. **~14 concrete correctness bugs** in the runtime + CLI, the worst being: input schemas are
   never validated on Lambda, loop-step failures mark the wrong step, queue job stores have
   read-modify-write races, and multi-group queue starters resolve the wrong queue name.
4. **Zero tests** in both packages despite a complex distributed state machine.
5. **Maintenance hazards**: regex-based source scanning in the CLI, a 2,800-line duplicated
   template blob (now fixed — templates are generated from `examples/root` by
   `scripts/sync-boilerplate.mjs`), and a ~700-line duplicated local-mode runtime.

## Priority order (suggested)

| Priority | Item | Where | Plan |
| --- | --- | --- | --- |
| ✅ P0 | **DONE** — fix empty core `agent.url` (cicd: preserve URL on redeploy + `serverless info` fallback; per-group rows kept so project urlsMap retains all groups) + reroute picks the core/non-empty agent row | cicd/v2, microfox-reroute | [plan-0](./plans/plan-0-core-agent-url.md) |
| ✅ P0 | **DONE** (warn-only) — auth on cicd/v2 API: `microfox login` device flow + CLI token in cicd Mongo + static service key + middleware/ownership checks + real user attribution. Flip `CICD_AUTH_ENFORCE=true` after rollout | cicd/v2, microfox CLI, Foxhub | [plan-A](./plans/plan-A-cli-auth.md) |
| ✅ P0 | **DONE** — auth Foxhub `/api/aws/lambda/{logs,analytics}` (require Supabase session) + scope functions server-side from the project's `urlsMap` after an owner/member check; client-sent names ignored | Foxhub | (see [04 SEC-3](./04-security-issues.md)) |
| ✅ P0 | **DONE** — validate uploaded archive (zip magic, zip-slip, symlinks, size + entry caps) | cicd/v2 | [plan-C](./plans/plan-C-archive-validation.md) |
| ✅ P1 | **DONE** — require trigger/config/queue endpoint auth using a stable `WORKERS_API_KEY` (or projectId-derived); timing-safe compare; `--allow-public` default when absent, `--require-auth` to enforce | ai-worker + CLI | [plan-B](./plans/plan-B-worker-endpoint-auth.md) |
| P1 | Investigate microfox-reroute outage (likely Upstash/Cloudflare, plus real code smells) | Foxhub/microfox-reroute | [08-microfox-reroute](./08-microfox-reroute.md) |
| P1 | Validate worker input schema inside the Lambda handler | ai-worker | [05 B1](./05-bugs.md) |
| P1 | Fix queue store races (atomic Mongo positional updates / per-step Redis keys) | ai-worker | [05 B4](./05-bugs.md) |
| P1 | Fix loop-step `arrayStepIndex` bugs (fail path + previousOutputs) | ai-worker | [05 B2/B3](./05-bugs.md) |
| P1 | HMAC-sign webhooks | ai-worker + CLI | [04 SEC-6](./04-security-issues.md) |
| P2 | Fix multi-group queue-starter queue-name resolution | ai-worker-cli | [05 B5](./05-bugs.md) |
| P2 | Replace regex scanning with stub-evaluation of worker/queue modules | ai-worker-cli | [06 I-2](./06-improvements.md) |
| P2 | Test suite for `wrapHandlerForQueue` + queue stores | ai-worker | [06 I-1](./06-improvements.md) |
| P3 | Everything else in [06-improvements.md](./06-improvements.md) | all | — |

> ~~P0 "rotate leaked credentials"~~ removed — SEC-1 was a false alarm (files are gitignored).

### Implementation plans

Detailed implementation plans live in [`plans/`](./plans/) (all currently implemented; E's
server half still needs staging-cicd verification — see `.claude/AI_WORKFLOW/DEV_TASKS.md`):

| Plan | Scope |
| --- | --- |
| [plan-0-core-agent-url.md](./plans/plan-0-core-agent-url.md) | ✅ **Implemented.** Fixes the empty core `agent.url` — cicd was clobbering it with `''` on redeploys when `extractBaseUrl` returned null. Fix preserves the existing URL + adds a `serverless info` fallback; **per-group rows kept** (two stores: the project urlsMap keeps all groups for the console; the subdomain urlsMap feeds reroute). microfox-reroute now picks the core/non-empty agent row. Needs a core redeploy to backfill data |
| [plan-A-cli-auth.md](./plans/plan-A-cli-auth.md) | `microfox login` device flow → auth token stored in **cicd Mongo** (not Foxhub Supabase) → cicd auth middleware (token **+** static service key) → record real triggering user per deployment → console shows the member who deployed |
| [plan-B-worker-endpoint-auth.md](./plans/plan-B-worker-endpoint-auth.md) | Authenticate generated `/workers/trigger`, `/workers/config`, `/queues/*/start` using a stable secret (projectId-derived or a static env key) set **once** at boilerplate time — not regenerated per push; `--allow-public` is the default when no secret is found |
| [plan-C-archive-validation.md](./plans/plan-C-archive-validation.md) | Harden cicd archive intake: size caps, zip-slip-safe extraction, symlink rejection, file-count/type limits |
| [plan-D-env-config.md](./plans/plan-D-env-config.md) | ✅ **Coded + user-tested (2026-07).** `env` block in the microfox config (mode all-detected/explicit incl. per-group mode, include/exclude globs, per-group overlays) + stage env-file cascade (`.env` → `.env.local` → `.env.{stage}` → `.env.{stage}.local`) with opt-in `files:'isolated'` (stage files only, hard-error if missing, no hydration leaks). CLI-only |
| [plan-E-stages-and-server-envs.md](./plans/plan-E-stages-and-server-envs.md) | ✅ **Coded (2026-07), server half pending staging-cicd verification (H31/H32/R43).** Multi-stage deploys (fixed set prod/staging/dev; stage plumbed CLI→cicd→urlsMap→console; parallel CFN stacks; `/staging/*` path rows ride the unchanged reroute Worker) + durable encrypted per-stage env store injected at deploy time (fixes redeploy-clobbers-console-envs) + Foxhub console stage selector/env-store editor |
| [plan-F-local-dev-server.md](./plans/plan-F-local-dev-server.md) | ✅ **Coded + tested end-to-end (2026-07).** `ai-worker dev`: local HTTP server with the deployed core-group surface, in-process worker invocation, in-memory queue, file-persisted local job store (survives restarts, parked HITL included), hot reload without restarts, local DLQ, `/dev-store` API + boilerplate `localDevAdapter` so the app UI works store-less |
