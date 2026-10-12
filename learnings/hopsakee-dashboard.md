# Port learnings — My Project Hub (`hopsakee-dashboard`)

Date: 2026-09-27 (survey) — 2026-10-03 (built and rehearsed)
Ported by: Claude Code on the web
Tier: B, and the first server-rendered app (outlier)

Status: **built and rehearsed, not cut over.** Everything below the "real
server" heading is for after the cutover. The app-side decisions are in
`hopsakee-dashboard/PORT-NOTES.md` and the procedure is in
`hopsakee-dashboard/docs/CUTOVER-RUNBOOK.md`.

## What I assumed that was wrong

- **"It is a Cloudflare app, so a Node target is a gamble."** It is not. The
  `@cloudflare/vite-plugin` dependency and the `wrangler.jsonc` are leftovers.
  `@lovable.dev/vite-tanstack-config` 2.x builds through **Nitro 3**. Nitro
  defaults to `cloudflare-module` and forces it only inside a Lovable sandbox.
  One line (`nitro: { preset: "node-server" }`) produced a Node server that
  rendered, hydrated and redirected in Chromium on the first try. The
  `vite.config.ts` comment saying "@cloudflare/vite-plugin builds from this"
  was stale. Read the config wrapper's `dist/index.js` rather than its comments.
- **"Covers in Postgres needs a privileged reader, because `<img>` cannot send
  a token."** I first proposed an anonymous read on the covers table. Jelle
  asked why the image request could not carry the token like every other data
  request. It can: fetch with the header, then hand the `<img>` an object URL.
  This is the same lesson as jonkies-tody's "is the binary real?". I framed a
  trade-off around a constraint (`<img src>`) that was an implementation
  choice, not a requirement.
- **"jonkies-tody's Path C prerequisites are complete."** They do not grant
  `authenticated` usage on schema `auth`. Here, every query then failed with
  `permission denied for schema auth`. Why jonkies-tody works without it was
  not investigated. Its policies may reach `auth.uid()` only through
  `SECURITY DEFINER` functions.
- **"Survey row 'Auth is a good fit; jonkies-tody solved this exact shape'."**
  Mostly true. But this app's middleware verified tokens through GoTrue
  (`supabase.auth.getClaims`), and there is no GoTrue now. The replacement
  (`whoami()` through PostgREST) is new, not copied.

- **"The storage export keeps the bucket's folders."** It does not. The real
  export (2026-10-12) is 33 flat `<project id>.png` files; the database says
  `<user id>/<project id>.png`. Rehearsing on the real export, with Jelle's
  go-ahead, is what caught it. The import's missing-file refusal would have
  stopped the cutover safely, but on cutover day rather than before it.

## Where the iterations went

Ranked by time:

1. **The npm lockfile — five builds.** First, `bun.lock` resolves from
   Lovable's private registry mirror, so it is unusable. Then a host-generated
   lockfile was incomplete because it reused `node_modules`. A
   `--package-lock-only` one inside Alpine lacked rollup's musl binary. Then
   `npm ci` rejected npm's own lockfile over an optional peer (`ajv@^8` of the
   unused `@hookform/resolvers`). Last, the build guard's own regex matched
   example URLs in postgrest-js's doc comments.
2. **The secret loader vanished from the bundle.** `"sideEffects": false` in
   `package.json` made the bundler drop `import "./lib/load-secret-files"`
   silently. Only an end-to-end sync attempt caught it.
3. **The missing `auth` grant.** It was found only once a real browser drove
   real server functions through real PostgREST.

## Gotchas discovered

- **Lovable's `bun.lock` cannot be installed outside Lovable.** Every
  `resolved` URL points at `europe-west4-npm.pkg.dev/lovable-core-prod/…`.
  Check with `grep -o 'https://[^/]*' bun.lock | sort | uniq -c`.
- **Generate the npm lockfile inside the runtime image, with a full install:**
  `docker run --rm -v "$PWD/package.json:/w/package.json:ro" -w /w node:22-alpine npm install`,
  then copy `package-lock.json` out. Then run `npm ci` in the same container
  before trusting it.
- **`"sideEffects": false` drops bare imports.** Check with
  `grep -rl '<a string from the module>' .output/server`. If it is absent, the
  module never ran.
- **`auth.uid()` must tolerate an empty claims setting.** In a pooled session
  a transaction-local GUC reads back as `''`, and `''::json` raises. Use
  `nullif(current_setting('request.jwt.claims', true), '')`.
- **react-query retries three times before reporting an error.** An
  error-vs-empty check has to wait about 8 seconds, or it sees the loading
  skeleton and concludes nothing.
- **This app's four list screens rendered a failed query as empty** — the
  same shape jonkies-tody's admin page had. Fixed with one shared
  `QueryError` component. Expect it in every Lovable app.
- **`01-export.sh` picked the host `pg_dump` when `PG_NETWORK` was set**, then
  failed to resolve the container name. That stays latent on the box (no
  `postgresql-client`) and appears on any machine that has one.
- **A sandbox `docker build` cannot reach npm** (TLS interception). Pass the CA
  without touching the Dockerfile:
  `docker buildx build --build-context sandboxca=<dir> -f <copy-with-two-lines>`.

## What should become a template

- `backend/auth-shim/` (lookup-only variant), `backend/sql/00-schema-prereqs.sql`
  (with the grant and the hardened `auth.uid()`), and the `whoami()` pattern.
  These carry over to any Path C app whose server needs to know the caller
  without holding the JWT secret.
- `src/components/cover-image.tsx` + `src/routes/covers.$id.ts`: authenticated
  images from Postgres, with caching and lazy loading.
- `src/components/query-error.tsx`.
- The restore-gate seed (`90-restore-test-seed.sql`) for apps where no test
  data can be created through the UI before the import.

## Proposed playbook edits

Folded into `PORTING-PLAYBOOK.md` in the same change: a new section,
"Server-rendered apps, and Path C refinements".

## Open questions for Jelle

- Can `LOVABLE_API_KEY` be read out of Lovable? If not, AI classification and
  cover generation are off until issue #1 (everything else works).
- The layout of Lovable's storage export for this project. The import
  tolerates two layouts and refuses on a missing file, but has not seen the
  real one.
- Off-box: add Project Hub to the `hoggle-macmini` pull (one artifact).

## What only showed up on the real server

(Fill in after the cutover. Leave empty until then.)
