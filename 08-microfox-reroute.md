# 08 — microfox-reroute (Subdomain Routing Worker)

`Foxhub/apps/microfox-reroute` is a **Cloudflare Worker** that fronts every `*.microfox.app`
request and rewrites it to the deployed target URL using the Upstash *routemap*. It's the piece
that makes `{subdomain}.microfox.app` resolve to a project's API Gateway / S3 / agent URL.

## Where it sits in the pipeline

```
Browser → {subdomain}.microfox.app
        → Cloudflare route  *.microfox.app/*   (wrangler.toml [[routes]])
        → microfox-reroute Worker (src/worker.js)
             ├─ Redis HGET  subdomains:{subdomain}  field "urlsMap"
             │     (ROUTEMAP_UPSTASH_REDIS_REST_URL/TOKEN — wrangler secrets)
             ├─ pick longest matching urlsMap[].path that prefixes the request path
             └─ fetch(rule.url + remainingPath, request)   ← rewrite to target
        → (no match / error) → fetch(request) → origin (Vercel app)
```

### Who writes the routemap it reads

`cicd/v2/src/services/project.service.ts` (`ProjectService`) writes the
`subdomains:{subdomain}` hash via `@microfox/db-upstash` `CrudHash` into the **same**
`ROUTEMAP_UPSTASH_REDIS_*` Redis the Worker reads:

- on **project create** (`create()`): `subdomainStore.set(subdomain, { id, projectKey, urlsMap:
  leanUrls })` where `leanUrls = urlsMap.map(({service, ...rest}) => rest)` (service stripped).
- on **update** (`update()`): re-set when `subdomainChanged || urlsMapChanged`.
- the `urlsMap` content itself is produced by `LocalDeploymentOrchestrator.run()` +
  the urlsMap merge during deployment (agent / openapi / public rows, per group).
- **Stages (Plan E, 2026-07):** the merge key is now `(type, group, stage)` and non-prod deploys
  add rows with a `stage` field and **stage-prefixed paths** (`/staging/agent`,
  `/staging/docs.json`, `/staging/public`). Prod rows are byte-identical to before. The Worker
  itself is intentionally UNCHANGED — stage routing rides the existing longest-prefix match
  (D1 decision: `{sub}.microfox.app/{stage}/...`).

So the data contract is: deployment → cicd Mongo project.urlsMap → cicd writes lean copy to
Upstash `subdomains:` hash → Worker reads it. The Foxhub Next.js app also reads the same hash in
`utils/middleware/subdomain.ts` for its own subdomain handling.

## Config

- `wrangler.toml`: route `*.microfox.app/*` on zone `microfox.app`; `workers_dev = true`;
  observability on.
- Secrets (set via `wrangler secret put`): `ROUTEMAP_UPSTASH_REDIS_REST_URL`,
  `ROUTEMAP_UPSTASH_REDIS_REST_TOKEN`. **Not** in `wrangler.toml` (commented `[vars]` only).
- Deps: `@upstash/redis` (cloudflare entrypoint), `wrangler`.

## "Stopped working recently with no code change" — investigation

The user reports the Worker stopped routing despite no code edits. The Worker **swallows all
failures** (catch → fall through to origin), so an outage looks identical whether the cause is
a Redis credential change, missing data, a Cloudflare route/zone change, or a code edge case.
Most-likely-first:

### Most likely (external, matches "no code change")

1. **Routemap Upstash credentials rotated / DB migrated.** If `ROUTEMAP_UPSTASH_REDIS_REST_URL`
   or `_TOKEN` secret no longer matches the live DB, **every** `hget` throws → catch block →
   `fetch(errorUrl, request)` to origin → all subdomains fall through. This is the single most
   consistent explanation for "everything broke at once without code change." **Check first:**
   `wrangler secret list`, then a manual `hget subdomains:{known-subdomain} urlsMap` against the
   DB the secret points at.
2. **The `subdomains:` hash is empty / in a different DB.** If cicd was re-pointed to a new
   Upstash routemap DB (or `CrudHash` key prefix changed) but the Worker's secret still points
   at the old one, lookups return `null` → fall through. Confirm both sides use the same Upstash
   instance and the `subdomains:` prefix.
3. **Cloudflare route / zone binding.** If the `*.microfox.app/*` route was detached from the
   Worker (zone change, plan change, a re-deploy that didn't re-bind the route, or Cloudflare
   for SaaS / custom-hostname change), requests never reach the Worker at all. `compatibility_date
   = 2025-09-01` is fine; check the dashboard route binding and that `wrangler deploy` last
   succeeded.
4. **Upstash REST endpoint / TLS / plan change.** Free-tier Upstash DBs can be paused after
   inactivity; a paused DB makes `hget` throw → fall through.

### Genuine code issues (worth fixing regardless; could explain *partial* breakage)

> These are real bugs in `src/worker.js` but more likely cause *some* subdomains to misroute
> than a total outage. Listed because the audit should capture them.

- **R1 — failures are invisible (masks the real cause).** Every error path silently
  `fetch(request)`/`fetch(errorUrl, request)`. There is no way to tell a Redis-down outage from
  a no-match. **Fix:** return a distinct response (e.g. `503` with a header) on Redis error vs.
  origin-passthrough on genuine no-match, and rely on the existing `observability` logs. At
  minimum, don't treat a Redis exception as "no route."
- **R2 — rows with empty `url` are valid match candidates. ✅ FIXED (2026-06-13).** Multi-group
  deployments write urlsMap rows with `url: ''` for non-core groups that have no API Gateway
  (`local-deployment-orchestrator.ts`). The Worker filtered only on
  `rule.path && path.startsWith(rule.path)` and sorted by path length — it did **not** skip
  empty `url`, so an empty-`url` agent row sharing path `/agent` could win and build an invalid
  `targetUrl = '' + remainingPath`. **Now fixed:** the filter is
  `rule.path && rule.url && isCoreGroup(rule.group) && path.startsWith(rule.path)` — it skips
  empty URLs **and** restricts routing to the core group (only core exposes the HTTP API). See
  [plans/plan-0-core-agent-url.md](./plans/plan-0-core-agent-url.md).

  > Note: the multi-row subdomain urlsMap (one agent row per group; non-core ones empty) is
  > **intended** — it's derived from the project urlsMap which must keep all groups. The reroute
  > simply picks the core/non-empty agent row. This was compounded by a **cicd bug** that left
  > even the *core* agent row's `url` empty on redeploys; fixed in Plan 0 (preserve the URL +
  > robust baseUrl resolution). A core redeploy is needed to backfill the routemap. See
  > [plans/plan-0-core-agent-url.md](./plans/plan-0-core-agent-url.md).
- **R3 — prefix match without a boundary.** `path.startsWith(rule.path)` makes `/agent` match
  `/agentic`. **Fix:** require the next char to be `/`, `?`, or end-of-string, or compare
  segment-wise.
- **R4 — original `Request` reused as `fetch` init forwards the wrong `Host`.**
  `fetch(targetUrl, request)` carries the original headers, including `Host:
  {subdomain}.microfox.app`. API Gateway custom domains and some origins reject or misroute on a
  mismatched Host. **Fix:** build a fresh `Request(targetUrl, { method, headers (Host stripped),
  body, redirect })`. (Wouldn't suddenly break, but is a latent correctness bug.)
- **R5 — apex/utility subdomains hit the lookup.** `console`, `staging`, `www`, and the apex
  resolve a subdomain string and do a (wasted) Redis lookup before falling through. Harmless but
  adds a Redis round-trip to every such request; consider a skip-list.
- **R6 — `urlsMapChanged` in cicd uses reference comparison.**
  `project.service.ts`: `data.urlsMap !== existing.urlsMap` compares array references, so the
  routemap re-write fires on essentially every update (mostly harmless, occasionally a no-op
  miss if the same array instance is reused). Not a Worker bug, but it governs whether the
  routemap stays fresh.

### Recommended triage order

1. `wrangler deployments list` / dashboard — confirm the Worker is deployed and the
   `*.microfox.app/*` route is bound.
2. `wrangler secret list` — confirm `ROUTEMAP_UPSTASH_REDIS_REST_URL/TOKEN` exist; re-`put` them
   from the current cicd `.env` values.
3. Manually `hget subdomains:{a-known-working-subdomain} urlsMap` against that DB — empty/❌ ⇒
   data/DB mismatch (cause #2); throws ⇒ creds (cause #1).
4. Tail `wrangler tail` while hitting a subdomain — the `console.log`s reveal exactly which
   branch is taken (no match vs. caught error vs. matched-but-bad-url R2).
5. Only if all the above are healthy, ship R1–R4.

**Conclusion:** no code change + total outage points strongly at #1 (rotated/paused Upstash
routemap creds) or #3 (Cloudflare route binding), both consistent with the user's note. The
Worker's blanket catch hides which one — fixing **R1 first** would have made this diagnosable.
R2 is the most likely *code* bug to cause real (partial) breakage and should be fixed regardless.
