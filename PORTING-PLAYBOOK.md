# Porting playbook

Status: v2. Two Tier-A static ports done (hopsakee-decimal-finder / findjd —
the pilot; ren-afstand — second app, same pattern reused unchanged), plus the
first Tier-B app scoped and its container half built and verified
(jonkies-tody — backend decision still open, see its `PORT-NOTES.md`).
Every claim below is either VERIFIED (a real port confirmed it) or ASSUMED (it came
from planning). Never promote an ASSUMED line to VERIFIED without a run that
exercised it.

## Target pattern
- VERIFIED (hopsakee-decimal-finder pilot): static apps build with Node in a
  throwaway stage, then serve from Caddy in a shared final-stage base image
  (`hopsakee-server/base/node-static.Dockerfile`, `node-static-base:20`, built
  once per box). One HTTP server implementation across the whole box, so not
  nginx.
- VERIFIED, with a correction: non-root, fixed UID 1001, real home dir,
  chowned, nologin shell — but **the exact user-creation command depends on
  the base image's userland, not just "Debian vs Alpine" as a fixed fact per
  language**. The Python apps' `ghcr.io/astral-sh/uv:...-trixie-slim` is
  Debian (`groupadd`/`useradd`); `caddy:2.8` is Alpine (`docker run --rm
  --entrypoint sh caddy:2.8 -c 'cat /etc/os-release'` → Alpine 3.20) and needs
  BusyBox's `addgroup -g 1001 -S appuser` / `adduser -S -D -u 1001 -G appuser
  -s /sbin/nologin -h /home/appuser appuser` instead — `groupadd`/`useradd`
  don't exist there (exit 127). Check `/etc/os-release` inside the actual base
  image before writing this block; don't assume from the app's language.
  **Follow-up, checked directly against the registry**: there is no
  Debian-based Caddy image to switch to even if we wanted one. The manifest's
  own OCI annotations for `caddy:2.8` say `org.opencontainers.image.base.name:
  alpine:3.20`, and the full tag list only has `2.8` / `2.8-alpine` /
  `2.8-builder` / `2.8-builder-alpine` (+ Windows variants) — both Linux tags
  resolve to the same Alpine image. Caddy publishes Alpine-only now. So
  `node-static-base:20` being Alpine, while every Python app's base is Debian,
  is not an inconsistency to fix — it's the only option the upstream image
  offers, short of hand-rolling and maintaining our own Debian+Caddy image.
  Decision: keep Alpine as the one deliberate exception on the box.
- VERIFIED (ren-afstand): each app defaults to `<repo-name>.hopsakee.top`
  unless Jelle says otherwise for that specific app — confirmed directly
  after the pilot ("most often I want the repo name to be the same as the
  web-url start"). The pilot's `hd.hopsakee.top` was the deliberate
  exception, not a new pattern; `ren-afstand` uses the plain default with no
  divergence between repo/container/subdomain names.
- **Before treating a repo as absent/empty and reconstructing its source
  from the live Lovable project, check whether it's private, not missing.**
  This session's GitHub access is scoped to a fixed repo list at session
  start; a private repo Jelle hasn't added yet is invisible to
  `list_repos`/`add_repo` the same way a genuinely nonexistent repo would
  be — there's no access-denied signal to tell them apart from inside the
  session. Hit this on `ren-afstand`: assumed empty, had it reconstructed
  from Lovable, then Jelle clarified it already had real commit history and
  was just private. Caught before any damage by diffing the reconstruction
  against the real `origin/main` (file list + content) before pushing
  anything — but the right move is to ask "could this be private?" before
  reconstructing, not after.
- VERIFIED, three-for-three now (hopsakee-decimal-finder, lovable-porting,
  ren-afstand): a brand-new or previously-untouched repo **rejects pushes
  with a 403** ("Claude doesn't have GitHub access to `<repo>` for your
  organization... An org admin can install the Claude GitHub App...") even
  after `add_repo` reports it added and readable. Read access and push
  access are separately scoped — `add_repo` only confirms the former. This
  isn't a one-time setup fluke; expect it on **every** app repo the first
  time a port touches it, and budget for a "please install the GitHub App
  on this repo" round-trip before the app-side PR can actually open. Once
  Jelle installs it for a given repo, later pushes to that same repo work
  without asking again.
- **Confirmed, deliberate design (Jelle, after the ren-afstand PRs): the
  per-app reference copies (`deploy.sh`, `caddy-snippet.txt`, and — for a
  different reason, see below — the app's own `Caddyfile`) living in the
  app repo alongside the real files in `hopsakee-server` are intentional
  duplication, not accidental.** `deploy.sh`/`caddy-snippet.txt` exist so
  the deploy contract for an app is readable/documented next to its source
  without needing the `hopsakee-server` repo open; they're never executed
  automatically and must be hand-kept in sync with the real
  `hopsakee-server/server_setup/deploy-<app>.sh` and the block already
  pasted into `hopsakee-server/config/caddy/conf/Caddyfile`. This was
  questioned directly and explicitly kept as-is — don't "clean up" this
  duplication in a future port without asking again. Separately, the app
  repo's own `Caddyfile` is **not** a duplicate of anything — it configures
  a completely different Caddy process (the one baked into the app's own
  Docker image, serving that app's static files with cache headers) from
  the shared box-wide Caddy in `hopsakee-server` (which only does
  hostname-based TLS routing to each app's container). The two look
  nothing alike because they do different jobs; that's expected, not a bug.

## Known traps
- **VERIFIED (jonkies-tody) — and the real shape is worse than this line
  used to describe.** Vite reads env at BUILD time; `VITE_*` must be compose
  `build.args`, never `environment:`. The old ASSUMED wording said getting it
  wrong makes the app "silently ship pointing at the old cloud project". The
  mechanism that makes that happen, found on the first app to actually use
  `VITE_*`: **the Lovable-exported repo COMMITS its `.env`** (git-tracked,
  and `.gitignore` has no `env` entry at all), holding the old cloud
  project's URL and publishable key. Vite reads it out of the build context
  automatically. So a build with no env configuration at all does not fail
  loudly — it produces a working-looking app still talking to the old cloud
  Supabase project. Confirmed directly: a plain `npm run build` baked the
  old `*.supabase.co` host into the bundle.
  Also confirmed, the half that makes a fix possible: **a shell/ARG value
  does override the committed `.env`** — same tree, `VITE_SUPABASE_URL=...
  npm run build`, and the new host is what lands in the bundle.
  So the fix is two halves, and one alone is not enough: `rm -f .env` inside
  the build stage so the committed file can never win by default, AND a
  build-time guard that exits non-zero if the args are missing, turning a
  silently-wrong production app into a build failure. Both verified through
  a real `docker build` (argless build exits 1; correct build ships a bundle
  containing the new host and no trace of the old project ref). Copy this
  pattern into every remaining Tier-B port — see `jonkies-tody/Dockerfile`.
  Related, so it isn't mistaken for a secrets leak: the publishable ("anon")
  key is public by design — it ships inside the browser bundle, and access
  control is RLS+JWT, not key secrecy. Docker's `SecretsUsedInArgOrEnv` lint
  fires on it and is safe to ignore *for that key only*. The service-role
  key and OAuth client secrets are real secrets and belong on the secrets
  volume.
- VERIFIED (three ports): the committed `package-lock.json` from a
  Lovable-exported repo reliably fails `npm ci` under this sandbox's npm
  version (rollup/vitest optional-dependency drift — not the same missing
  packages each time, but the same failure shape). Three-for-three now, not
  pilot-specific bad luck — **expect to `rm package-lock.json && npm
  install` on every port** and budget for it up front rather than
  discovering it each time. On `jonkies-tody` (the newer
  `new_style_vite_react_shadcn_ts_testing_2026-01-08` template) the missing
  entries were specifically the **test-scaffolding** packages
  (`@testing-library/dom` and its tree): that template adds vitest/testing-
  library to `package.json` but ships a lockfile that never saw them. So the
  newer template does not fix this trap — if anything its extra devDeps are
  exactly what's missing.
- VERIFIED (hopsakee-decimal-finder pilot): PWA service workers cause
  stale-index.html after redeploy unless `sw.js`/`manifest.webmanifest`/
  `index.html` get `Cache-Control: no-cache` (hashed `/assets/*` files are
  safe to cache for a year — their filename changes with their content).
  **Caddy-specific gotcha found while verifying this**: Caddy's `path`
  matcher checks the *original* request URI, not what `try_files` rewrites it
  to. A matcher listing only `/index.html` does not apply its header to a
  plain `GET /`, even though `try_files {path} /index.html` serves
  `index.html`'s bytes for it — `curl -I /` showed no `Cache-Control` header
  at all until `/` was added to the matcher explicitly. List `/` in the
  matcher every time.
- ASSUMED, still untested: `.wasm` needs `Content-Type: application/wasm`.
  The pilot app's Caddyfile sets this defensively (`@wasm path *.wasm`), but
  the app ships no `.wasm` file, so this was never actually exercised
  end-to-end. Verify for real on the first app that has one.

## Verifying a Tier-B port without touching real data
- VERIFIED (jonkies-tody): an app's whole migration chain can be replayed
  against a throwaway `postgres:15-alpine` container in the sandbox, which
  is the "replay app migrations against the empty new DB first" step
  `MIGRATION-PLAN.md` already calls for — done early, for free, before any
  real data or credentials exist. It needs ~40 lines of stubs for the
  Supabase-provided bits the migrations assume (`auth.users`, `auth.uid()`,
  `storage.buckets`/`storage.objects`, the `anon`/`authenticated`/
  `service_role` roles). 25 of jonkies-tody's 26 replayed cleanly first try.
  Two things this buys that reading the SQL does not: it proves the chain
  actually applies in filename order, and querying `pg_policies`/`pg_proc`/
  `pg_trigger` afterwards gives the **net** final schema (31 policies, 11
  functions, 8 triggers) rather than a raw count of `CREATE` statements that
  later migrations have already dropped and replaced. Do this on every
  Tier-B port before estimating a rewrite.
- The one migration that failed did so against the *stub*, not Postgres: it
  calls `storage.extension(name)`, a Supabase Storage built-in, inside an
  upload policy. Worth knowing which migrations depend on Supabase-specific
  helpers — those are precisely the ones that need hand-reimplementation if
  an app moves off Supabase.

## Tier-B backend pattern — VERIFIED (jonkies-tody)
- **Supabase-exported apps do not need Supabase to keep their security model.**
  Verified end-to-end in a sandbox: the app's 8 tables, 11 functions, 8
  triggers, 31 RLS policies and its `collect_prize` RPC all work unchanged
  behind **plain Postgres + PostgREST + a JWT we mint ourselves** — no GoTrue,
  no Kong, no Storage service, no Studio. Three backend containers instead of
  five.
  The hinge is that `auth.uid()` is not a Supabase feature. It is three lines
  of SQL over a request-scoped setting that any JWT-validating gateway
  populates:
  `SELECT nullif(current_setting('request.jwt.claims', true)::json->>'sub','')::uuid`
  PostgREST (the same component Supabase uses for its data API) populates it.
  So the choice for a ported app is **not** "self-host all of Supabase or
  rewrite everything" — the schema is portable to bare Postgres, and only
  auth/storage need replacing. GoTrue is the redundant part once Authelia is
  the identity provider, which is exactly what Jelle objected to paying for.
  Verified with real requests: anonymous → `[]`; authenticated → correct
  RLS-filtered rows; participant self-promotion silently reverted;
  admin-only inserts 403; unowned-prize collection rejected by the RPC's own
  check. `handle_new_user()` also fires on a plain `INSERT INTO auth.users`,
  so the signup flow works against our own identity table.
  Reproduction scripts live in `jonkies-tody/docs/pathc-prototype/`.
- **`@supabase/supabase-js` still works against bare PostgREST**, so every
  `.from()` / `.rpc()` call site in a ported app survives untouched — the
  client just gets our JWT instead of a GoTrue session. Only the `auth.*` and
  `storage.*` call sites need replacing. This is what collapses a Tier-B port
  from "rewrite everything" to a few hundred lines.
- **Check for silent-revert triggers before writing any cutover runbook.**
  jonkies-tody's `protect_profile_columns()` reverts `role`/`is_approved`/
  `total_points` whenever `is_admin(auth.uid())` is false — and `auth.uid()` is
  NULL on *any* connection without a JWT: plain `psql`, a restore, a
  maintenance script. It raises no error, it just discards the change. So
  "restore, then correct the data" can appear to succeed and do nothing.
  Bootstrapping the first admin on a fresh database requires dropping the
  trigger, making the change, and recreating it. Always verify a correction
  landed instead of assuming.
- **Replaying the schema is also a bug-finding exercise, not just a
  compatibility check.** Running jonkies-tody's real RPC against the replayed
  schema surfaced a live data-integrity bug in the app as shipped (prize
  collections recorded but never deducted, because the balance trigger's own
  UPDATE is reverted by the guard above when a participant triggers it).
  Budget for the possibility that a Tier-B port finds real bugs in the app it
  is porting, and that fixing them belongs before the cutover rather than
  after.

## Not yet answered
- SQLite backup routine on the box. **No longer blocking jonkies-tody** — that
  app's backend decision landed on Postgres. Still open for any app that ends
  up on SQLite.
- Whether one SQLite file can safely serve more than one container. Same
  status: not blocking jonkies-tody any more.
- Whether a `pg_dump` snapshot routine with a tested restore is what Jelle
  accepts as closing the hard gate for Postgres-backed apps.
- ~~Which backend jonkies-tody talks to.~~ **Answered**: neither self-hosted
  Supabase nor a SQLite rewrite, but plain Postgres + PostgREST + an
  Authelia→JWT shim (see the Tier-B pattern above, and
  `jonkies-tody/PORT-NOTES.md`). Pending Jelle's confirmation.
- ~~Whether jonkies-tody gets one login prompt or two.~~ **Answered as a
  consequence**: one. Authelia becomes the identity provider and Google
  sign-in is dropped, so no second login, no second approval list, and no
  Google OAuth client to register at all.
