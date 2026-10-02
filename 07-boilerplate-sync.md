# 07 — Boilerplate Sync Workflow & mediamake Divergence Report

## The problem

The `ai-worker boilerplate` command scaffolds ~2,800 lines of Next.js integration code (job
stores, API routes, the `useWorkflowJob` hook). Until 2026-06-12 those templates were
**hand-pasted string literals** inside `boilerplate.ts`, originally copied from `examples/root`.
Meanwhile `mediamake` (the main real consumer) evolved its own copies. Result: **three diverging
copies** of the same code — CLI templates, `examples/root`, and `mediamake` — each with fixes
the others lacked.

## Divergence audit (2026-06-12)

Compared: `examples/root/app/api/workflows/**` + `hooks/useWorkflowJob.ts` vs
`mediamake/apps/mediamake/app/api/workflows/**` + `hooks/useWorkflowJob.ts` vs the CLI's
embedded templates.

| File | examples/root | mediamake | CLI templates (before) |
| --- | --- | --- | --- |
| `stores/jobStore.ts`, `mongoAdapter.ts`, `redisAdapter.ts`, `queueJobStore.ts` | ✅ identical | ✅ identical | ✅ identical |
| `auth.ts` | generic stub ✔ | app-specific (Upstash session store) — *not portable* | stub ✔ |
| `registry/workers.ts` | had `retry` step resolution; **lacked** `schemas`/`getWorkerSchema` | had `schemas` + `getWorkerSchema` (pairs with the CLI's new inputSchema feature); **lacked** `retry` | lacked **both** |
| `workers/[...slug]/route.ts` | lacked `GET /schema`, `GET /history` | had schema+history **plus** app-specific quota/admin/x-client-id logic | lacked schema/history |
| `queues/[...slug]/route.ts` | ✅ newest — HITL approve **race fix** (step → running *before* dispatch) | ❌ stale — dispatches before updating (race) | ❌ stale |
| `hooks/useWorkflowJob.ts` | ✅ newest — `terminalHitRef` + `clearThisPolling` polling fixes | ❌ stale | ✅ had the fixes |

**Verdict:** neither side was strictly newer. mediamake had newer *worker-facing* features
(schema/history endpoints, schemas in registry); ai-router had newer *queue/HITL/hook* fixes.

## What was merged into `examples/root` (this session)

Generic mediamake improvements ported (app-specific quota/auth/clientId code deliberately
excluded):

1. `registry/workers.ts`
   - `WorkersConfig.schemas?: Record<string, unknown>` (JSON Schemas embedded by the CLI's
     workers-config handler at build time)
   - `getWorkerSchema(workerId)` export
2. `workers/[...slug]/route.ts`
   - `GET /api/workflows/workers/:workerId/schema` → returns the worker's input JSON Schema
     (404 when unavailable)
   - `GET /api/workflows/workers/:workerId/history` → all jobs for a worker via
     `listJobsByWorker`, newest first

`examples/root` is now the **superset** and the single source of truth.

## ⚠️ Backport needed in mediamake

These ai-router fixes are still missing in `mediamake/apps/mediamake` (see bug B24 in
[05-bugs.md](./05-bugs.md)):

1. **`app/api/workflows/queues/[...slug]/route.ts`** — in `handleQueueApprove`, move the
   `updateQueueStep(id, targetStepIndex, { status: 'running', startedAt, input })` call to
   **before** `dispatchWorker(...)`. Otherwise the resumed Lambda can append/overwrite the steps
   array before the approve route's read-modify-write completes (lost-update race).
2. **`hooks/useWorkflowJob.ts`** — adopt the `terminalHitRef` + per-cycle `clearThisPolling`
   pattern from examples/root. mediamake's version (a) starts an interval even when the first
   synchronous poll already hit a terminal status, and (b) clears `intervalRef`/`timeoutRef`
   globally, so an old cycle's timeout can kill a newer cycle's polling after `reset()` +
   re-trigger.

Easiest path: copy both files from `examples/root` and re-apply mediamake's local additions
(quota check block + `x-client-id` handling in the workers route only; the queues route and hook
have no mediamake-specific code).

## The new sync workflow

```
examples/root (source of truth)
   │  edit boilerplate-relevant files here, test them in the running example app
   ▼
packages/ai-worker-cli/scripts/sync-boilerplate.mjs
   │  npm run sync-boilerplate        (regenerates)
   │  npm run sync-boilerplate:check  (CI / prebuild guard)
   ▼
src/commands/boilerplate.templates.generated.ts   (AUTO-GENERATED, do not edit)
   ▼
ai-worker boilerplate   (scaffolds into user projects)
```

### How it works

- `TEMPLATE_SOURCES` in the script maps each template key (destination path relative to
  `app/api/workflows/` in the user project) to a source file in `examples/root`:

  | Template key | Source |
  | --- | --- |
  | `auth.ts` | `app/api/workflows/auth.ts` |
  | `stores/jobStore.ts` (+ mongo/redis/queue adapters) | same paths |
  | `stores/localDevAdapter.ts` (Plan F, 2026-07-10) | same path — `WORKER_DATABASE_TYPE=local` adapter proxying store reads/writes to the `ai-worker dev` server's `/dev-store` API |
  | `registry/workers.ts` | same path |
  | `workers/[...slug]/route.ts`, `queues/[...slug]/route.ts` | same paths |
  | `../../../hooks/useWorkflowJob.ts` | `hooks/useWorkflowJob.ts` |

  (10 templates as of 2026-07-10.)

- Content is CRLF-normalized and escaped (`\`` ` ``, `${`, `\`) for embedding in template
  literals.
- `boilerplate.ts` now just does `import { TEMPLATES } from './boilerplate.templates.generated.js'`
  (the 2,811-line inline blob is gone; file went from 3,117 → 308 lines).
- **`prebuild` runs the `--check` mode**, so a stale generated file fails the package build —
  you cannot publish a CLI whose templates lag `examples/root`.

### Adding a new boilerplate file

1. Create + test the file in `examples/root`.
2. Add one line to `TEMPLATE_SOURCES` in `scripts/sync-boilerplate.mjs`.
3. `npm run sync-boilerplate` and commit both the script change and the regenerated module.

### Keeping mediamake in sync (recommended follow-up)

mediamake-specific code (auth, quotas) lives in the same files as generic boilerplate, which is
why it drifted. Two options:

- **Light:** add a small script in mediamake that diffs its workflow files against
  `examples/root` (ignoring whitespace/CR) and prints divergence — run it when bumping
  `@microfox/ai-worker`.
- **Better:** refactor mediamake's app-specific logic out of the route bodies (e.g. a
  `withWorkerGuards(handler)` wrapper around the generic route, an injected `auth.ts`), so the
  generic files can be overwritten wholesale by `ai-worker boilerplate --force`.
