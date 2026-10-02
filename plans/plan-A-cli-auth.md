# Plan A — CLI Auth + cicd Authentication + Per-Deployment User Attribution

**Status:** ✅ **implemented (2026-06-15), shipped in warn-only mode (`CICD_AUTH_ENFORCE` unset).**
cicd: `models/cli-token.model.ts`, `models/cli-device-session.model.ts`, `services/auth.service.ts`,
`middleware/auth.ts`, `routes/auth.routes.ts` (mounted in `app.ts`), `deployment.model` `triggeredBy`,
and ownership checks across deployment/project/organization routes. microfox CLI: `commands/login.ts`,
`commands/logout.ts`, `utils/auth.ts`, auth headers on `push`/`status`. Foxhub:
`app/(protected)/(other)/cli-auth/` (page + approve server action) and a "Triggered by" row in the
deployments tab. Rollout: deploy cicd → ship CLI → ship Foxhub → set `CICD_AUTH_ENFORCE=true` and
remove the ownerId fallback.
**Goal:** stop anyone from deploying to / reading from cicd by guessing a `projectId`. Add a
`microfox login` flow whose **token lives in cicd's MongoDB (not Foxhub Supabase)**, authenticate
every cicd endpoint with **that token + a static service key**, derive the **real triggering
user** from the token, persist it on the deployment, and show it in the console.

Addresses [04-security-issues.md SEC-2](../04-security-issues.md). Touches `microfox/packages/cli`,
`cicd/v2`, and `Foxhub/apps/web`.

---

## Current state (what exists today)

- `microfox push` → POSTs the zip to `https://{mode}-v2-cicd.microfox.app/api/deployments/local`
  with only `x-project-id` + `x-deployment-group` headers. **No auth.**
- cicd `deployment.routes.ts` looks up the project by `x-project-id` and sets
  `deployment.userId = project.ownerId` — i.e. **every deployment is attributed to the project
  owner**, regardless of who actually ran `push`.
- cicd already has Mongo (mongoose), `Deployment.userId` (required, indexed), and
  `Organization.members: string[]` / `Project.ownerId`. Identities are Foxhub **Supabase user
  ids** (Foxhub creates projects in cicd with the Supabase uid as `ownerId`).
- Foxhub auth = Supabase sessions. Console calls cicd directly from the **browser** via
  `NEXT_PUBLIC_CICD_BASE_URL` (so any secret used there is public — see constraints).

## Target state

1. `microfox login` performs a device-authorization flow; the user approves in the browser while
   logged into Foxhub. cicd mints a long-lived **CLI token** and stores it (hashed) in Mongo,
   linked to the Supabase `userId`. CLI saves the token to `~/.microfox/credentials.json`.
2. Every cicd request carries `Authorization: Bearer <cliToken>` **and**
   `x-microfox-service-key: <STATIC_KEY>`. cicd middleware validates both, resolves `userId`,
   and checks project ownership/membership before deploy/read/delete.
3. `deployment.userId` = the **real** authenticated user; a denormalized `triggeredBy
   { userId, email?, name? }` is stored for display.
4. Console deployment list shows **who triggered** each deployment (resolved from `triggeredBy`).

---

## Design decisions & constraints

- **Token store = cicd Mongo**, per the requirement. New collection `cli_tokens`. Foxhub's
  Supabase is used only to *authenticate the human* during `login` (verify their session), not to
  store the CLI token.
- **Static service key is a coarse gate, not the real auth.** A CLI is distributed publicly, so a
  key baked into it is not truly secret. Its job: (a) block unauthenticated internet noise, (b)
  let cicd distinguish "known client" traffic, (c) be rotatable. The **per-user CLI token +
  ownership check is the real enforcement.** Set the static key via env on both sides
  (`MICROFOX_SERVICE_KEY` in CLI env / baked default; `MICROFOX_SERVICE_KEY` in cicd env). Do
  **not** put it in `NEXT_PUBLIC_*`.
- **Console → cicd is browser-side today.** Two options (pick in Phase 4):
  - (A) Keep browser calls but require the user's Supabase access token (`Authorization: Bearer
    <supabase-jwt>`); cicd verifies the JWT (Supabase JWT secret) → userId. No static key needed
    for the console path (it's user-authenticated).
  - (B) Proxy console→cicd through Foxhub server routes (`/api/console/*`) that attach the
    service key + the session user server-side. Cleaner trust boundary; more routes to add.
  - **Recommended:** (A) for reads, (B) for any mutation (deploy/delete) so the service key never
    reaches the browser.
- **Back-compat / rollout:** ship cicd auth in **warn-only mode** first (accept unauthenticated,
  log + attribute to `ownerId` as today), then flip to **enforce** once the CLI release with
  `login` is out and Foxhub is updated. Gate with `CICD_AUTH_ENFORCE=true`.

---

## Phase 1 — cicd: data model + token service

**New collection `cli_tokens`** (`cicd/v2/src/models/cli-token.model.ts`):

```ts
interface ICliToken {
  tokenHash: string;       // sha256(token); never store the raw token
  userId: string;          // Supabase uid
  email?: string;          // denormalized for display/audit
  name?: string;
  label?: string;          // e.g. "macbook-cli"
  lastUsedAt?: Date;
  expiresAt?: Date;        // optional; default long-lived, revocable
  revokedAt?: Date;
  createdAt: Date; updatedAt: Date;
}
// indexes: { tokenHash: 1 } unique, { userId: 1 }
```

**New collection `cli_device_sessions`** (pending device-auth flows), TTL-indexed:

```ts
interface ICliDeviceSession {
  deviceCode: string;      // opaque, CLI-held secret
  userCode: string;        // short human code shown in CLI + entered/links in browser
  status: 'pending' | 'approved' | 'denied' | 'expired';
  userId?: string;         // set on approval
  email?: string; name?: string;
  expiresAt: Date;         // TTL index → auto-cleanup
  createdAt: Date;
}
```

**`AuthService`** (`cicd/v2/src/services/auth.service.ts`):
- `startDeviceSession()` → `{ deviceCode, userCode, verificationUrl, interval, expiresAt }`.
- `approveDeviceSession(userCode, { userId, email, name })` (called after Foxhub verifies the
  human) → marks approved.
- `pollDeviceSession(deviceCode)` → when approved, mint a CLI token (`crypto.randomBytes(32)`),
  store `sha256` in `cli_tokens`, delete/expire the device session, return the **raw token once**.
- `resolveToken(rawToken)` → `{ userId, email, name } | null` (hash lookup, check
  revoked/expired, bump `lastUsedAt`).
- `revokeToken(...)` / `listTokens(userId)` for a future `microfox logout` / token management.

## Phase 2 — cicd: auth routes + middleware

**Routes** (`cicd/v2/src/routes/auth.routes.ts`, mounted `/api/auth`):
- `POST /api/auth/cli/start` → `AuthService.startDeviceSession()`. (Requires static service key.)
- `POST /api/auth/cli/approve` → body `{ userCode }`, **authenticated as a Foxhub user** (Supabase
  JWT in `Authorization`, verified here) → `approveDeviceSession`. Called by the Foxhub
  `cli-auth` page (server action) — see Phase 5.
- `POST /api/auth/cli/token` → body `{ deviceCode }` → `pollDeviceSession`; `202` while pending,
  `200 { token, userId }` once approved. (Requires static service key.)
- `POST /api/auth/cli/revoke` (optional, for `microfox logout`).

**Middleware** (`cicd/v2/src/middleware/auth.ts`), applied to `/api/deployments`,
`/api/projects`, `/api/organizations`:
1. Check `x-microfox-service-key === process.env.MICROFOX_SERVICE_KEY` (timing-safe). Missing/bad
   → `401` (unless `CICD_AUTH_ENFORCE!=='true'`, then warn + continue).
2. Resolve identity, in order:
   - `Authorization: Bearer <cliToken>` → `AuthService.resolveToken` → `req.auth = { userId,
     email, name, via: 'cli' }`.
   - else `Authorization: Bearer <supabaseJwt>` → verify Supabase JWT → `req.auth = { …, via:
     'console' }` (the console read path, option A).
3. **Authorization** (per route): deploy/read/delete require `req.auth.userId` to be
   `project.ownerId` **or** a member of `project.organizationId` (`OrganizationService` →
   `org.members.includes(userId)`). Enforced in the route handlers (they already load the
   project).

Phase the enforcement: env `CICD_AUTH_ENFORCE`. Until true, attach identity when present but
don't reject — so old CLIs keep working during rollout.

## Phase 3 — cicd: real user attribution on deployments

In `deployment.routes.ts` `POST /local`:
- replace `userId: project.ownerId` with `userId: req.auth?.userId ?? project.ownerId`
  (fallback only while `CICD_AUTH_ENFORCE` is off).
- add `triggeredBy: { userId, email, name, via }` to the `Deployment` doc (new field on
  `deployment.model.ts`; `Schema.Types.Mixed` or a small subschema).
- same for `/stop`, `/delete`, group-delete: attribute the actor.

`Deployment` model change:
```ts
triggeredBy?: { userId: string; email?: string; name?: string; via?: 'cli' | 'console' };
```
(Keep `userId` as-is for back-compat + indexing; `triggeredBy` is the display source of truth.)

## Phase 4 — microfox CLI: `login` + authenticated `push`

**New command** `microfox login` (`microfox/packages/cli/src/commands/login.ts`):
1. `POST {cicd}/api/auth/cli/start` (with static service key) → `{ deviceCode, userCode,
   verificationUrl }`.
2. Print the `userCode` and open `verificationUrl` (`https://microfox.app/cli-auth?code=USERCODE`)
   in the browser (`open`/`child_process`).
3. Poll `POST {cicd}/api/auth/cli/token` every `interval`s until `200` → save
   `{ token, userId, cicdBase }` to `~/.microfox/credentials.json` (mode `600`).
4. `microfox logout` → delete file + best-effort `revoke`.

**Auth helper** (`microfox/packages/cli/src/utils/auth.ts`):
- `getCredentials()` → read `~/.microfox/credentials.json` (env override
  `MICROFOX_TOKEN`/`MICROFOX_SERVICE_KEY` for CI).
- `authHeaders()` → `{ Authorization: 'Bearer '+token, 'x-microfox-service-key': serviceKey }`.

**Update `push.ts` / `status.ts`:** attach `authHeaders()` to the axios calls. On `401` with a
clear "not logged in" body, print `Run \`microfox login\` first.` For CI, support
`MICROFOX_TOKEN` env (a token minted in the dashboard) instead of the interactive flow.

Static service key: ship a default baked constant **and** allow `MICROFOX_SERVICE_KEY` override.
Document that it's a soft gate; rotate by releasing a new CLI + updating cicd env (support 2 keys
during rotation).

## Phase 5 — Foxhub: the browser approval page + console display

**`/cli-auth` page** (`Foxhub/apps/web/app/(protected)/cli-auth/page.tsx`): protected (Supabase
session required). Reads `?code=USERCODE`, shows "Authorize CLI access for {email}? [Approve]".
On approve → server action calls `POST {cicd}/api/auth/cli/approve` with the user's Supabase JWT
+ `{ userCode }`. Show success/failure.

**Console deployment list** (`components/subapps/console/components/deployments-tab.tsx` +
`lib/console-api.ts`): include `triggeredBy` in `DeploymentDto`; render the member
(email/name/avatar). If only `userId` is present (old rows), resolve via the org member list
Foxhub already has. Add a "Triggered by" column / chip per deployment.

**Console → cicd auth (option A/B):** for reads, send the Supabase JWT (cicd verifies). For the
"deploy from UI" / stop / delete actions, route through a Foxhub server endpoint that adds the
service key server-side (so it never ships to the browser).

---

## Files to create / change (checklist)

cicd/v2:
- `src/models/cli-token.model.ts`, `src/models/cli-device-session.model.ts` (new)
- `src/services/auth.service.ts` (new)
- `src/routes/auth.routes.ts` (new) + mount in `src/app.ts`
- `src/middleware/auth.ts` (new) + apply to deployment/project/organization routes
- `src/models/deployment.model.ts` — add `triggeredBy`
- `src/routes/deployment.routes.ts` — use `req.auth.userId`, set `triggeredBy`, ownership checks
- `src/routes/project.routes.ts`, `organization.routes.ts` — ownership checks
- env: `MICROFOX_SERVICE_KEY`, `SUPABASE_JWT_SECRET` (to verify console JWTs), `CICD_AUTH_ENFORCE`

microfox/packages/cli:
- `src/commands/login.ts`, `logout.ts` (new) + register in `cli.ts`
- `src/utils/auth.ts` (new)
- `src/commands/push.ts`, `status.ts` — attach auth headers, handle 401

Foxhub/apps/web:
- `app/(protected)/cli-auth/page.tsx` + approve server action (new)
- `components/subapps/console/lib/console-api.ts` — `triggeredBy` in DTO + auth header/proxy
- `components/subapps/console/components/deployments-tab.tsx` — show triggerer
- (option B) `app/api/console/**` proxy routes that inject the service key server-side

---

## Security notes

- Store only `sha256(token)`; compare with `timingSafeEqual`. Tokens are `randomBytes(32)` hex.
- Rate-limit `/api/auth/cli/*` and the deploy route.
- Verify the Supabase JWT properly (issuer, expiry, signature with `SUPABASE_JWT_SECRET`) — don't
  trust a `userId` header.
- The static service key is **defense-in-depth only**; never rely on it for authorization. All
  authorization decisions use the resolved `userId` + project ownership/membership.
- Keep `CICD_AUTH_ENFORCE=false` until the CLI + Foxhub releases are live, then flip and remove
  the `ownerId` fallback.

## Rollout sequence

1. cicd: ship models + routes + middleware in **warn-only** (`CICD_AUTH_ENFORCE=false`), deploy.
2. CLI: ship `login`/`logout` + authed push (still works against warn-only cicd).
3. Foxhub: ship `/cli-auth` + console `triggeredBy` display + console auth path.
4. Flip `CICD_AUTH_ENFORCE=true`; announce minimum CLI version; remove `ownerId` fallback.
