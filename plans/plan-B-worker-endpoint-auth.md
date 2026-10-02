# Plan B — Authenticate Generated Worker Endpoints (stable secret, public-by-default-when-absent)

**Status:** ✅ **implemented (2026-06-15).** All five phases landed in `packages/ai-worker-cli`
(`push.ts` `resolveWorkersApiKey` + handler templates + env.json writers + `--require-auth` /
`--allow-public` flags), `packages/ai-worker` (`client.ts` `resolveWorkersTriggerKey` /
`resolveWorkersConfigKey` / `deriveWorkersApiKey`), `examples/root` registry (re-synced into the
boilerplate template), and `boilerplate.ts` (writes a random `WORKERS_API_KEY` to `.env`). See
[04 SEC-4/SEC-9](../04-security-issues.md#s1).
**Goal:** require a secret on the deployed `/workers/trigger`, `/workers/config`, and
`/queues/*/start` endpoints **using a stable secret that is not regenerated on every push**, so
existing dispatchers keep working. When **no secret is found, deploy publicly by default**
(`--allow-public` is the implicit default), with a loud warning.

Addresses [04-security-issues.md SEC-4 / SEC-9](../04-security-issues.md). Touches
`packages/ai-worker-cli` (handler generation) and `packages/ai-worker` (client/registry that must
send the key) and the boilerplate.

---

## Requirements (from the user)

- **Do not generate a new key on every compile/push** — that would break already-deployed
  consumers each time.
- Use a **stable** secret: either the **projectId** (already in `microfox.json`, "kind of a
  secret that should not be exposed") **or a dedicated static env key** created once.
- **`--allow-public` is the default when no secret is found** — i.e. presence of a secret opts you
  into enforcement; absence means public (current behavior) but warned.

## Current state

Generated `workers-trigger.js` / `workers-config.js` / queue starters check
`WORKERS_TRIGGER_API_KEY` / `WORKERS_CONFIG_API_KEY` **only if the env var is set**; otherwise
fully open. CORS `*`. `?debug=1` on `/workers/config` leaks attempted queue names + AWS errors.
The client (`ai-worker` `dispatch`/`dispatchWorker`/`dispatchQueue`) and the app registry already
send `x-workers-trigger-key` / `x-workers-config-key` **when the env var is present**.

So the wiring exists — what's missing is a **stable, low-friction secret source** and
**enforcement semantics**.

---

## Design

### Secret resolution (build time, in the CLI)

Resolve a single `WORKERS_API_KEY` at `push` time with this precedence (first hit wins):

1. Explicit per-endpoint keys (back-compat): `WORKERS_TRIGGER_API_KEY` / `WORKERS_CONFIG_API_KEY`
   if set → keep using them (unchanged behavior).
2. **`WORKERS_API_KEY`** (new unified env) if set → use for both trigger and config.
3. **projectId** (from `microfox.json` / config) → derive a key, e.g.
   `WORKERS_API_KEY = sha256('microfox-workers:' + projectId)` (so the raw projectId isn't the
   literal header value, but it's deterministic and stable across pushes). This is the
   **zero-config** path.
4. **Nothing resolvable** → **public deploy** (implicit `--allow-public`), print a loud warning.

> Recommended primary: a dedicated **`WORKERS_API_KEY`** created **once** by `boilerplate`/`new`
> (random, written to `.env` + reminded to add to the deployed env). projectId-derived is the
> zero-config fallback. Rationale: projectId already travels to cicd in `x-project-id` headers and
> `microfox.json`; reusing it as the worker API key couples two secrets and widens its blast
> radius. A separate key keeps concerns isolated. Offer both; default to `WORKERS_API_KEY` when
> present, projectId-derived otherwise.

Because the source is stable (projectId never changes; `WORKERS_API_KEY` is written once and
reused), **re-pushes don't rotate the secret** — satisfying the "don't regenerate every push"
requirement.

### Enforcement semantics

- If a secret resolves → generate handlers that **require** it (timing-safe compare); write the
  key into `env.json` so the Lambda has it; ensure the **consumer side** sends it (see below).
- If no secret resolves → generate **public** handlers (current behavior) + print:
  `⚠️  Workers deployed PUBLICLY (no WORKERS_API_KEY / projectId). Anyone can trigger them. Set
  WORKERS_API_KEY to require auth.` This is `--allow-public` as the default-when-absent.
- Add an explicit `--require-auth` flag that **fails the build** if no secret resolves (for teams
  that want to guarantee non-public deploys in CI). Inverse of the implicit default.

### Consumer side (must send the same key)

The Next.js app dispatches via `@microfox/ai-worker` and the registry. Today they read
`WORKERS_TRIGGER_API_KEY` / `WORKERS_CONFIG_API_KEY`. Add `WORKERS_API_KEY` as a fallback in:
- `client.ts` (`dispatch`, `dispatchWorker`, `dispatchQueue`) — header `x-workers-trigger-key`.
- `examples/root` `registry/workers.ts` (`fetchWorkersConfig`, trigger) — both headers.
- For the projectId-derived path, the app must compute the same `sha256('microfox-workers:' +
  projectId)`. Simplest: the CLI **writes the resolved key into `env.json` as `WORKERS_API_KEY`**
  and the app reads it from env — no client-side derivation needed. (The app already loads the
  same env.)

So the contract becomes: **CLI resolves the key once → writes `WORKERS_API_KEY` to env.json →
both the deployed Lambdas and the consuming app read it from env.** No per-push rotation, no
client math.

### Hardening (bundled with this change)

- Replace `providedKey !== apiKey` with `crypto.timingSafeEqual` (SEC-9) in all generated
  handlers.
- Gate `?debug=1` output on a valid key (or drop it).
- Optionally tighten CORS from `*` to the configured app origin when a key is required (a key +
  `*` CORS is still fine since the key is a header, but worth offering).

---

## Phases

**Phase 1 — CLI secret resolution.** Add `resolveWorkersApiKey(microfoxConfig, env)` to
`push.ts` returning `{ key, source }` or `null`. Thread it into `generateTriggerHandler`,
`generateWorkersConfigHandler`, `generateQueueHandler`, and the multi-group/core variants. Write
`WORKERS_API_KEY` into `env.json` (and per-group `env.json`s) when resolved.

**Phase 2 — Generated handler templates.** Update the three handler string templates to:
- read `process.env.WORKERS_API_KEY` (plus legacy `WORKERS_TRIGGER_API_KEY` /
  `WORKERS_CONFIG_API_KEY` for back-compat),
- if a key is configured → require + timing-safe compare → `401` on mismatch,
- if not → open (no check),
- timing-safe compare helper inlined.

**Phase 3 — Public-by-default + flags.** Implement the warning when no key resolves; add
`--require-auth` (fail build if none) and keep behavior public otherwise. Document
`--allow-public` as the implicit default (no flag needed).

**Phase 4 — Consumer side.** `client.ts` + registry: read `WORKERS_API_KEY` (fallback to legacy
vars). Confirm `dispatchQueue` sends it to `/queues/{id}/start` (it already sends
`x-workers-trigger-key` when present — just add the env fallback).

**Phase 5 — Scaffolding.** `boilerplate` / `new`: generate a random `WORKERS_API_KEY` into the
project `.env` once (if absent) and mention it in the next-steps output, so new projects are
non-public by default without per-push rotation.

---

## Files to change

- `packages/ai-worker-cli/src/commands/push.ts` — `resolveWorkersApiKey`, thread into all handler
  generators + env.json writers; warning/flag logic.
- `packages/ai-worker/src/client.ts` — `WORKERS_API_KEY` fallback header on all dispatch fns.
- `examples/root/app/api/workflows/registry/workers.ts` — same fallback (then re-run
  `npm run sync-boilerplate`).
- `packages/ai-worker-cli/src/commands/boilerplate.ts` / `new.ts` — write a random
  `WORKERS_API_KEY` to `.env` once.
- Docs: update [04 SEC-4/SEC-9](../04-security-issues.md) + README env matrix once shipped.

## Notes / trade-offs

- **projectId-as-secret caveat:** it's not a strong secret (it travels in headers/config and is
  guessable-ish), so the sha256-derived form is a mild improvement but still only a soft gate.
  For anything sensitive, prefer a real random `WORKERS_API_KEY`. Document this honestly.
- This is **independent of Plan A** (that secures the *deploy/control plane*; this secures the
  *deployed data plane*). Both should ship, but neither blocks the other.
- Stable key means a compromised key requires a deliberate rotation (`WORKERS_API_KEY` change +
  redeploy + app env update) — acceptable and expected; document the rotation steps.
