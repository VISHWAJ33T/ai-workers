# Plan 0 — Empty core `agent.url` + reroute core-only routing — IMPLEMENTED

**Status:** ✅ implemented 2026-06-13 (cicd/v2 + microfox-reroute). Needs a **core redeploy** to
backfill existing data — see "Rollout".

**Symptom (reported):** a project's (mediamake) `urlsMap` had the **core** group's `agent` row
with `url: ""` even though core deployed fine (its `openapi` row had the real
`https://fz72h3pgt8…amazonaws.com/prod/docs.json`). Non-core groups' `agent` rows are
legitimately empty (SQS-only workers, no API Gateway).

> **Important correction (vs. an earlier draft of this plan):** there are **two** urlsMap stores
> in Redis, and they must NOT be collapsed:
>
> | Store | Written by | Shape | Consumed by |
> | --- | --- | --- | --- |
> | **`projects:{key}`** hash | `ProjectService` (`projectStore.set`) | full urlsMap **with** `service.providerConfig.resources` — **all groups** | the **console** (observability/settings/delete-group) |
> | **`subdomains:{subdomain}`** hash | `ProjectService` (`subdomainStore.set`) | `urlsMap.map(({service, ...rest}) => rest)` — lean rows (type/path/url/group), **all groups** | the **microfox-reroute** Cloudflare Worker |
>
> The subdomain map is **derived** from the project map (same rows, minus `service`). The project
> map **must keep every group's rows + resources**. So the fix keeps the per-group structure
> untouched and only (a) stops core's URL from being blanked, and (b) makes the reroute pick the
> core agent row out of the multi-row subdomain map.

---

## Root cause (debugged)

mediamake deploys with `publish.handles` (`agent:'/agent'`, `openapi:'/docs.json'`). The core
deployment ran (in `local-deployment-orchestrator.ts`):

```js
const agentUrl = baseUrl ? `${baseUrl}/${stage}` : '';   // '' when baseUrl null
...
if (handles.openapi && baseUrl) maps.push({ OPENAPI ... });  // SKIPPED when baseUrl null
```

`baseUrl` came from `ServerlessService.extractBaseUrl()` — a regex over `serverless deploy`
stdout — which returns `null` on some redeploys (no-changes / `--force` / format change). On a
core **redeploy** with `baseUrl=null`: the agent row was rewritten `url:''` (replacing the good
value via `mergeUrlsMapByTypeAndGroup`), while the openapi row was **not** re-emitted (guarded by
`&& baseUrl`), so the merge kept the old real openapi URL. → `agent:""`, `openapi:real`. **Core
overwriting its own agent URL on a redeploy — not another group.**

(Non-core groups are unaffected by the fix: their `baseUrl` is always `undefined` by design, so
their agent rows stay `url:''` with their lambda resources — exactly as before.)

## Fixes implemented

### 1. cicd: never clobber a good URL; resolve baseUrl robustly (per-group structure unchanged)

`cicd/v2/src/workers/local-deployment-orchestrator.ts`:
- Added `priorUrlFor(type)` (keyed on the row's `(type, group slot)`) and resolve
  `agentUrlResolved = baseUrl ? \`${baseUrl}/${stage}\` : priorUrlFor(AGENT)` (same for openapi).
  So when a redeploy can't resolve `baseUrl`, the **existing** URL is preserved instead of being
  overwritten with `''`. Non-core groups keep `url:''` (their `priorUrlFor` is `''`).
- Added a `serverless info` fallback: when the core/ungrouped stack has the HTTP API but the
  deploy output didn't surface an endpoint, call `serverless.resolveBaseUrlViaInfo(stage)`.
- **Everything else is unchanged** — still per-group rows (`group` on the row), still
  `mergeUrlsMapByTypeAndGroup`, all groups' rows + resources still written to the project urlsMap.

`cicd/v2/src/services/serverless.service.ts`:
- Broadened `extractBaseUrl` to match any `execute-api` host (docs.json line preferred, generic
  host fallback) instead of only the `GET - …/docs.json` line.
- Added `resolveBaseUrlViaInfo(stage)` (runs `serverless info --verbose`, extracts the base URL).

### 2. cicd: the subdomain urlsMap only carries routing-relevant rows

`cicd/v2/src/services/project.service.ts`:
- Added `buildSubdomainUrls(urlsMap)` and used it at both `subdomainStore.set` sites (create +
  update). It strips `service` **and filters out rows with an empty `url`** — so the lean
  `subdomains:{subdomain}` map no longer carries the non-core agent rows (`url:""`). The
  **project** hash still keeps every group (full rows + resources) for the console; only the
  routing copy is slimmed.
- Result, after a redeploy/update: the subdomain map is just the core rows
  (`agent` + `openapi` [+ `public`]) — no empty `scraper`/`calculator`/`test`/`media` rows.

### 3. reroute: left unchanged

The reroute Worker keeps its original simple match (`rule.path && path.startsWith(rule.path)`).
Because `buildSubdomainUrls` (fix #2) now strips empty-URL rows at the source, the subdomain map
the Worker reads only contains routing-relevant rows (core agent/openapi/public), so no
Worker-side filtering is needed.

## What was NOT changed (deliberately)

- The **project urlsMap keeps all groups** (per-group rows + per-group `service` resources). The
  console observability/settings/delete-group continue to work exactly as before.
- `IUrlMap.group`, `mergeUrlsMapByTypeAndGroup`, `findServerlessUrlsMapEntryForGroup`,
  `post-deployment.service.ts`, `delete-group-worker.ts` — all untouched (reverted to original).

## Rollout / backfill ⚠️

Existing project/subdomain maps still have the empty core agent URL. After deploying the cicd
build:

1. **Redeploy the core group** (`microfox push core`). The orchestrator now resolves `baseUrl`
   (deploy output or `serverless info`) and writes a real core `agent.url`; `ProjectService`
   propagates it to both the project hash and the `subdomains:{subdomain}` routemap.
2. Verify: `hget subdomains:{subdomain} urlsMap` → the core `agent` row has the real URL (non-core
   agent rows remain empty — expected).
3. `wrangler deploy` the reroute Worker.

Nothing breaks before the redeploy: the reroute filter already ignores empty-URL/non-core agent
rows; the agent path simply won't route until the core URL is repopulated.

## Files changed

- `cicd/v2/src/workers/local-deployment-orchestrator.ts` — preserve-prior-URL + `serverless info`
  fallback + `normalizeUrlsMapGroupSlot` import. (per-group structure intact)
- `cicd/v2/src/services/serverless.service.ts` — robust `extractBaseUrl` + `resolveBaseUrlViaInfo`.
- `cicd/v2/src/services/project.service.ts` — `buildSubdomainUrls()`: the subdomain (routing) map
  drops empty-URL rows; the project map still keeps all groups.
- (microfox-reroute Worker left unchanged — the subdomain map is cleaned at the source.)

## Follow-ups (not in this change)

- Prefer CloudFormation `ServiceEndpoint`/`ApiEndpoints` stack output over CLI-text parsing for
  `baseUrl` ([06 I-20](../06-improvements.md)).
- Reroute R1/R3/R4 (error masking, path-boundary match, Host header) remain open
  ([08-microfox-reroute.md](../08-microfox-reroute.md)).
