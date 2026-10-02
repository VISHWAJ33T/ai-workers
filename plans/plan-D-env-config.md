# Plan D — Config-driven env control + stage-scoped .env files

**Status:** ✅ Phases 1–2 coded + live-tested (2026-07-09), awaiting manual review before publish
(see `.claude/AI_WORKFLOW/DEV_TASKS.md` R39). Phase 3 (docs) queued in `EASY_AI_TASKS.md` (DOCS-2).
**Goal:** stop shipping wrong/missing envs. (1) `microfox.config.ts` gains an `env` block that
controls exactly which env keys go into `env.json` — per project AND per group — including an
"all-detected plus these extras" mode (fixes the real incident where a package-consumed env was
never detected and silently dropped). (2) `.env.{stage}` file support with a standard precedence
cascade. All changes live in `ai-router/packages/ai-worker-cli` (compile path only — no runtime,
no cicd, no console changes).

Smallest-risk, highest value-per-effort of the three plans. Do this one FIRST — Plan E and
Plan F both reuse its env-file cascade and env-selection logic.

---

## Current state (verified in code, 2026-07-08)

- **`loadEnvVars()`** (`compile.ts` ~2491) reads ONLY `./.env`, with a naive hand-rolled parser
  (no multiline values, no `export ` prefix, strips surrounding quotes). `hydrateProcessEnvFromDotenv`
  (~2521) loads the same single file into `process.env` (non-overriding).
- **Detection** (`collectEnvUsageForWorkers`) regex-scans worker entry files + their local import
  graph for literal `process.env.X` / `process.env['X']` / `import.meta.env.X` (~lines 100–125).
  It inherently CANNOT see: dynamic access (`process.env[someVar]`), envs consumed inside
  `node_modules` packages, or envs read via config helpers. → the "missed env" incident.
- **env.json filter** (single-group ~3619, per-group ~3550): a key from `.env` survives only if
  `allowedPrefixes.some(p => key.startsWith(p)) || referencedEnvKeys.has(key)`. The prefix list is
  HARDCODED: `OPENAI_ ANTHROPIC_ DATABASE_ MONGODB_ REDIS_ UPSTASH_ WORKER_ WORKERS_ WORKFLOW_
  REMOTION_ QUEUE_JOB_ DEBUG_WORKER_QUEUES`. Anything else that detection misses is silently dropped.
  `AWS_*` always excluded (Lambda-reserved — keep). `ENVIRONMENT/STAGE/NODE_ENV` force-set (their
  'prod' hardcoding is Plan E's problem, not ours).
- **Per-group env.json** already exists (multi-group layout re-runs detection per group's workers)
  — so group scoping has a natural attachment point.
- The env.json build logic is DUPLICATED between the single-group block (~3614–3636) and the
  multi-group block (~3554–3560), with slight drift already (multi-group hardcodes 'prod' inline).
- `resolveMicrofoxConfig()` (~359) already loads `microfox.config.ts` / `microfox.json` and the
  result (`microfoxConfig`) is in scope at both env.json sites — config plumbing is free.

## Target state

### 1. `env` block in microfox.config.ts

```ts
export default {
  // ...existing config...
  env: {
    // 'all-detected' (default) = current behavior (detection + prefix allowlist) + include/exclude.
    // 'explicit' = ONLY include lists ship (no detection, no prefix allowlist).
    mode: 'all-detected',
    include: ['SOME_SDK_KEY', 'MYAPP_*'],   // always shipped if present in env files (fixes detection misses)
    exclude: ['DEBUG_*', 'LOCAL_ONLY_TOKEN'], // never shipped, wins over everything below
    groups: {
      // overlays applied ON TOP of the project-level lists for that group's env.json
      video: { include: ['REMOTION_LICENSE_KEY'] },
      core:  { exclude: ['HEAVY_FEATURE_FLAG'] },
    },
  },
}
```

Resolution order per key (per group in multi-group layout):
1. Platform-required keys always ship and cannot be excluded: `ENVIRONMENT`, `STAGE`, `NODE_ENV`,
   `WORKERS_API_KEY` (via existing `applyWorkersApiKey`).
2. `exclude` (project ∪ group) → dropped.
3. `include` (project ∪ group) → shipped (value must exist in the merged env files; if not → warning).
4. mode `all-detected`: detected keys + legacy prefix allowlist → shipped (unchanged behavior).
   mode `explicit`: nothing else ships.
5. `AWS_*` never ships regardless.

Glob support: `*` wildcard only (prefix/suffix/middle), tiny in-house matcher (~10 lines), no new dep.

### 2. Stage-scoped env files (dotenv-flow convention)

`loadEnvVars()` → `loadEnvFiles(stage)`: merge, later wins:
`.env` → `.env.local` → `.env.{stage}` → `.env.{stage}.local`
(`stage` = the compile `--stage`; today effectively 'prod', becomes meaningful with Plan E; the
Plan F dev server calls the same cascade with stage 'dev'). `hydrateProcessEnvFromDotenv` follows
the same cascade. Real shell/CI env still wins over all files (unchanged rule). Warn (don't fail)
when no `.env*` file exists at all (current behavior).

### 3. Better warnings (cheap, high value)

- `include` key not found in any env file → **yellow warning listing the key + which files were read**.
- Keys that were detected but are NOT in env files: already logged (`missingFromDotEnv`) — keep.
- In `explicit` mode: log the exact final key list per group (names only, never values).

## Design decisions

- **Default = `all-detected` with the legacy prefix allowlist intact** → a project with no `env`
  block compiles byte-identically to today. Zero migration.
- The hardcoded `allowedPrefixes` array stays as the compiled-in default of `all-detected` mode
  (moving it to config would break every existing project); `exclude` lets users prune it.
- `include` does NOT invent values — a key must exist in the merged env files (or shell env) to
  ship. (Server-side/console-set values are Plan E's job.)
- Do NOT swap the parser for the `dotenv` package in this pass (multiline support etc.) — nice to
  have, separate tiny follow-up, avoids conflating a behavior change with this feature.

## Phases

1. **`loadEnvFiles` cascade** — replace both `loadEnvVars()` call sites + `hydrateProcessEnvFromDotenv`.
   Thread `stage` in. (~½ day)
2. **Extract shared `buildEnvJson(envVars, detected, envConfig, group)`** — deduplicate the
   single-group and multi-group env.json blocks into one function, THEN add the include/exclude/mode
   logic + glob matcher + warnings to it. Type the `env` block in the config interface.
3. **Docs + boilerplate**: document the `env` block in `ai-worker-cli.mdx` + the boilerplate
   `microfox.config.ts` template (commented-out example). → EASY_AI_TASKS candidate once code lands.

## Review flags (Anti-Vibe)

- ⚠ **env.json content changes affect DEPLOYED Lambda env** — even though defaults are
  back-compat, the extraction/refactor of duplicated logic touches the deploy artifact. Manual
  review of a before/after `env.json` diff on examples/root (single-group AND multi-group) before
  publishing the CLI.
- ⚠ Publishing: same `@microfox/ai-worker-cli` publish train as any pending CLI changes (R37/R38
  ordering rules still apply).

## Test checklist (for DEV_TASKS when built)

- No `env` block → `env.json` identical to previous CLI version (single + multi group).
- `include: ['FOO']` with FOO in `.env` → ships; FOO absent → warning, not shipped.
- `exclude: ['OPENAI_*']` → OPENAI_API_KEY gone despite allowlist prefix.
- Group overlay: key ships only in that group's `env.json`.
- `explicit` mode ships exactly include-lists + platform keys.
- `.env.production` overrides `.env` for the same key when `--stage production`.
