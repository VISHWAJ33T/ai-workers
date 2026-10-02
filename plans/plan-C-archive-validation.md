# Plan C — Harden cicd Archive Intake (size, zip-slip, symlinks, zip-bomb)

**Status:** ✅ **implemented (2026-06-15)** in `cicd/v2/src/utils/archive.utils.ts`
(`validateAndExtractArchive`: zip magic check, `onEntry` symlink reject + zip-slip assert +
entry-count/uncompressed-size caps, post-extract symlink sweep) and wired into
`routes/deployment.routes.ts` `/local` (multer `fileSize`/`files`/`fields` limits, 413 mapping,
partial-extraction cleanup + FAILED marking). Caps are env-tunable: `MAX_ARCHIVE_MB` (200),
`MAX_EXTRACTED_MB` (1024), `MAX_ENTRIES` (50000). Phase 4 (install sandboxing / `--ignore-scripts`)
remains a separate follow-up.
**Goal:** make `POST /api/deployments/local` safe against malicious or malformed uploads:
enforce size limits, extract without path traversal (zip-slip), reject symlinks, and cap
entry count / uncompressed size (zip-bomb). Pairs with Plan A (auth) — auth stops *who*, this
plan limits *what* an authorized upload can do to the build host.

Addresses [04-security-issues.md SEC-2 fix #3](../04-security-issues.md).

---

## Current state

`cicd/v2/src/routes/deployment.routes.ts`:
```js
const upload = multer({ dest: path.join(process.cwd(), 'uploads') });  // no limits
router.post('/local', upload.single('archive'), async (req, res) => {
  ...
  await extract(filePath, { dir: targetDir });   // extract-zip, default options
  try { await fs.promises.unlink(filePath); } catch {}
  ...
});
```

Problems:
- **No multer limits** — unbounded upload size / field count.
- **No archive validation** — not verified to be a real zip; no entry-count / uncompressed-size
  cap (zip-bomb can fill the disk).
- **extract-zip defaults** — modern `extract-zip` (v2) does block `..` path traversal, but this
  isn't asserted/tested here, and **symlink entries** can still point outside `targetDir` (a
  symlink + a later entry written "through" it, or a symlink that the subsequent `npm install` /
  build follows). Should be explicitly rejected.
- **Temp cleanup** only on the happy path / best-effort; errors can leak files in `uploads/`.

---

## Design

### 1. multer limits

```js
const upload = multer({
  dest: path.join(process.cwd(), 'uploads'),
  limits: {
    fileSize: MAX_ARCHIVE_BYTES,   // e.g. 100 MB (env MAX_ARCHIVE_MB, default 100)
    files: 1,
    fields: 10,
  },
});
```
Handle multer's `LIMIT_FILE_SIZE` error → `413 Payload Too Large`.

### 2. Validate it's a zip before extracting

- Check magic bytes (`PK\x03\x04`) on the first 4 bytes of the uploaded file.
- Reject otherwise → `400`.

### 3. Safe extraction (zip-slip + symlink + bomb)

Replace the bare `extract(filePath, { dir })` with a guarded routine. Two options:

**Option A — keep `extract-zip`, add an `onEntry` guard** (extract-zip ≥ 2 supports it):
```js
const targetReal = await fs.promises.realpath(targetDir);
await extract(filePath, {
  dir: targetDir,
  onEntry: (entry /* yauzl entry */, zipfile) => {
    // a) reject symlinks (external attribute high bits) and non-file/dir types
    const unixMode = (entry.externalFileAttributes >>> 16) & 0xffff;
    const isSymlink = (unixMode & 0o170000) === 0o120000;
    if (isSymlink) throw new Error(`Symlink entry rejected: ${entry.fileName}`);
    // b) zip-slip: ensure resolved path stays inside targetDir
    const dest = path.resolve(targetDir, entry.fileName);
    if (dest !== targetReal && !dest.startsWith(targetReal + path.sep)) {
      throw new Error(`Path traversal entry rejected: ${entry.fileName}`);
    }
    // c) bomb: accumulate entry.uncompressedSize and count; throw over caps
  },
});
```
(`onEntry` throwing aborts extraction.)

**Option B — switch to `yauzl` directly** for full control (entry iteration, per-entry caps,
explicit file-type allowlist). More code, but no reliance on extract-zip internals.

Recommended: **Option A** (less churn), with explicit caps:
- `MAX_ENTRIES` (e.g. 20,000)
- `MAX_TOTAL_UNCOMPRESSED_BYTES` (e.g. 1 GB, env-tunable)
- per-entry `entry.uncompressedSize` sanity (skip 0-size dirs).

### 4. Post-extract guards

- After extraction, **scan for symlinks** as a belt-and-suspenders check
  (`fs.lstat().isSymbolicLink()` over the tree; reject + cleanup if any slipped through).
- Confirm an expected entrypoint exists (`serverless.yml` or `microfox.json` / per-group dir);
  reject empty/garbage archives early.

### 5. Robust cleanup

- `try/finally` around the whole handler: always `unlink` the uploaded temp file and, on any
  validation failure, `rm -rf` the partial `targetDir` and mark the deployment FAILED with a
  clear `error.step = 'compiling'` reason.

### 6. (Defense in depth, optional) install safety

The subsequent `CompilerService` runs `npm install`, which executes **postinstall scripts** from
the uploaded `package.json`. Even with archive validation, consider `npm install
--ignore-scripts` (or an allowlist) and/or running the build as an unprivileged user / container.
Track separately — it's the bigger RCE surface but a larger change. (Cross-ref Plan A SEC-2 fix
#4.)

---

## Phases

1. **multer limits + zip magic check** (small, high value).
2. **`onEntry` guard** (symlink reject + zip-slip assert + entry/size caps).
3. **post-extract symlink sweep + entrypoint sanity + robust cleanup.**
4. (separate) **install sandboxing** — `--ignore-scripts` / container.

## Files to change

- `cicd/v2/src/routes/deployment.routes.ts` — multer limits, magic check, guarded extract,
  cleanup. Consider extracting the logic into `cicd/v2/src/utils/archive.utils.ts`
  (`validateAndExtractArchive(filePath, targetDir, caps)`).
- env: `MAX_ARCHIVE_MB`, `MAX_EXTRACTED_MB`, `MAX_ENTRIES`.
- (Phase 4) `cicd/v2/src/services/compiler.service.ts` — install flags / sandbox.

## Tests

- Unit `validateAndExtractArchive` against fixtures: a normal archive (passes); a `../escape`
  entry (rejected); a symlink entry (rejected); an over-size archive (rejected); an over-count
  archive (rejected); a non-zip blob (rejected). These are cheap and lock the behavior.

## Notes

- This is mostly independent of Plans A/B and can ship first as a quick risk reduction — it
  limits blast radius even while the API is still being locked down.
- Keep limits env-tunable so legitimate large agents (e.g. with Remotion assets) aren't blocked;
  start generous and tighten with telemetry.
