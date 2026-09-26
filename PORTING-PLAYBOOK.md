# Porting playbook

Status: v3. Two Tier-A static ports done (hopsakee-decimal-finder / findjd —
the pilot; ren-afstand — second app, same pattern reused unchanged), and the
first Tier-B port **completed end to end on the real box**: jonkies-tody is
live on Hetzner with the family's real data migrated off Lovable and verified
row-for-row (2026-09-19). Everything under "Tier-B cutover" below comes from
that cutover, not from a sandbox.
Every claim below is either VERIFIED (a real port confirmed it) or ASSUMED (it came
from planning). Never promote an ASSUMED line to VERIFIED without a run that
exercised it.

This file is about **how** to port. For **which** apps, in what order, and what
was decided about each, see [`PORT-SCOPE.md`](PORT-SCOPE.md).

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
  `pg_trigger` afterwards gives the **net** final schema (28 policies in `public`, 11
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
  triggers, 28 RLS policies and its `collect_prize` RPC all work unchanged
  behind **plain Postgres + PostgREST + a JWT we mint ourselves** — no GoTrue,
  no Kong, no Storage service, no Studio. Three backend containers instead of
  five.
  **Count correction, from the real box.** Earlier versions of this playbook
  said 31 policies. The real number is **28 in `public`**, plus 4 on the
  `storage.objects` stub. A sandbox tally is not authoritative; check the
  live database with
  `select count(*) from pg_policies where schemaname='public'`.
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
- **VERIFIED with the real client, through the real routing**:
  `@supabase/supabase-js` works against bare PostgREST, so every `.from()` /
  `.rpc()` call site in a ported app survives untouched — the client just gets
  our JWT instead of a GoTrue session. Only the `auth.*` and `storage.*` call
  sites need replacing. This is what collapses a Tier-B port from "rewrite
  everything" to a few hundred lines.
  Two mechanics worth copying rather than rediscovering: supabase-js appends
  `/rest/v1` to the base URL, so the site block needs
  `handle_path /rest/v1/* { reverse_proxy <rest>:3000 }` (handle_path strips
  the prefix); and injecting the token via `global.fetch` rather than
  `global.headers` lets it refresh on a 401 without any call site knowing.
- **VERIFIED (jonkies-tody) — Caddy `forward_auth` header handling, and two
  traps in it.** An Authelia-header-reading service is only as trustworthy as
  the gate in front of it, so this was tested against real `caddy:2.8` with a
  stand-in authorizer rather than reasoned about:
  1. **Spoofing does not work, and no stripping is needed.** `copy_headers`
     SETS the listed headers on the upstream request, overwriting whatever the
     client sent. Authorizer saying `kind1` + client forging
     `Remote-User: <admin>` → the service saw `kind1`.
  2. **Do NOT "harden" it by stripping client headers first.** The obvious
     `request_header -Remote-User` before `forward_auth` BREAKS the gate:
     `request_header` sorts AFTER `forward_auth` in Caddy's directive order, so
     it deletes the header the gate just set and every request 401s. This was
     written into a draft snippet as a security improvement and would have
     shipped a dead app.
  3. **The real trap**: `copy_headers` sets a header even when the authorizer
     did NOT return it — to the literal unresolved placeholder text
     `{http.reverse_proxy.header.Remote-Email}`. That string reached a JWT
     claim before the shim validated it. List only what the authorizer
     actually returns, and validate the SHAPE of every optional header
     downstream regardless.
- **VERIFIED: a `pg_dump` routine with a tested restore is straightforward and
  should be written as part of the port, not after it.** Dump the WHOLE
  database, not just `public` — `auth.users` holds the identities every
  `profiles` row keys on, and losing them orphans the ledger. Verify the dump
  (gzip integrity + PostgreSQL's own "dump complete" marker) and atomically
  rename it into place, so a truncated dump is never mistaken for a backup.
  Back up any image/file volume too, or a restore yields rows pointing at
  files that no longer exist. Pull from the NAS rather than pushing from the
  box: the internet-facing host then holds no NAS credentials and cannot
  delete backup history. Proven by restoring into an empty Postgres and
  diffing row counts across every app table.
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

## Tier-B cutover — VERIFIED (jonkies-tody, real box, 2026-09-19)

Everything above was proven in a sandbox. This section is what only appeared
once the app was serving a browser and real family data had to move. Read it
before scheduling the next Tier-B cutover; most of these cost a round each.

### Get the data out of Lovable's dashboard, not over a live connection

The plan assumed `pg_dump` against `db.<ref>.supabase.co`. That route is worse
in three separate ways, all discovered the hard way:

- **Supabase shows the database password exactly once, at project creation.**
  There is no way to look it up later, only to reset it — and on a
  Lovable-built app nobody ever saw it, because Lovable abstracts it away.
- **Direct connections are IPv6-only** unless the project has the IPv4
  add-on. A box without IPv6 must use the pooler host instead, and `pg_dump`
  needs **session mode (5432)**, not transaction mode (6543).
- A password with `@ : / #` in it has to be percent-encoded inside the URL,
  and a mis-encoded password fails identically to a wrong one.

Lovable's dashboard exports the whole project as two zips, and that sidesteps
all of it: a **`.backup`** (pg_dump CUSTOM format, zstd) and the storage
bucket's files. Confirm the format with `head -c 5` → `PGDMP`.

**Restore it into a throwaway container and point the existing `01-export.sh`
at that**, rather than writing anything that parses the archive. The verified
half of the pipeline — the import and its guards — then stays verified, and
the source is a container on localhost so the password and IPv6 problems
evaporate. The export script needs one addition for this: a `PG_NETWORK` env
var so its helper containers join the scratch server's docker network.

### `pg_restore --schema=X` does not create schema X

`--schema=auth` restored **nothing**: 247 objects all failing with
`schema "auth" does not exist`. `--schema` selects objects *in* that schema,
and `CREATE SCHEMA auth` is in no schema — its TOC entry reads
`SCHEMA - auth`, where the `-` **is** the schema field. So the filter excludes
the one statement everything else depends on.

`public` survives this silently because a fresh database already has it, which
is exactly what makes the failure look app-specific rather than structural.

So: `CREATE SCHEMA IF NOT EXISTS auth;` before restoring, alongside the
extensions the app's defaults need (`uuid-ossp`, `pgcrypto`).

**And the restore is expected to print errors regardless** — the archive names
Supabase's own roles (`supabase_admin`, `supabase_auth_admin`). `--no-owner
--no-privileges` removes most of it. Never judge the restore by its output;
judge it by counting rows afterwards. A real failure hid in that noise here
precisely because the runbook said to expect noise.

**Constraints may not survive either.** Because `public` was restored while
`auth` did not exist, `profiles`' FK to `auth.users` was never created — so
`ON DELETE CASCADE` did not fire later and deleting an identity left its
profile orphaned. Check orphans explicitly rather than trusting a cascade:

```sql
SELECT (SELECT count(*) FROM public.profiles p
          WHERE NOT EXISTS (SELECT 1 FROM auth.users u WHERE u.id = p.id)) AS orphans;
```

### Source and target Postgres versions will not match

The Supabase project was **17.6**; the box runs **15**. A `pg_dump` new enough
to read the source emits preamble lines 15 rejects, and the import runs under
`ON_ERROR_STOP=1`, so one unknown line aborts the whole load:

- `SET transaction_timeout` — PostgreSQL 17+
- `\restrict` / `\unrestrict` — psql meta-commands, recent minors only
  (emitted by `pg_dump` 16 as well)

Strip them; both are session hygiene with no bearing on the data. **The
trailing space in each pattern is load-bearing** — without it the same match
eats `\.`, the COPY terminator, silently truncating every table while the
restore still reports success:

```
/^(SET transaction_timeout|\\restrict |\\unrestrict )/d
```

Relatedly: **the box has no `postgresql-client`**, and does not need one. Run
`pg_dump`/`psql` from a container, matching the image to the **source** major
version — `pg_dump` refuses a server newer than itself. Note `pg_dump -f`
writes the file *inside* the container; stream to the host instead.

### Real family data contains duplicate identities

The export held **7 identities and 6 profiles**, not the 5 and 5 expected.
Two people had signed up twice in the app's first week. Worse, for one of them
the account displaying as a *different person's name* was the real one — 114
points and a full ledger — while the account carrying his own name was empty.

So **reconcile identities before importing**, and note the danger: every table
is `REFERENCES public.profiles(id) ON DELETE CASCADE`, so deleting the wrong
row of a duplicate pair takes the ledger, activity log, prizes and
contributions with it. Prove a duplicate is empty first:

```sql
SELECT p.display_name, p.total_points,
       (SELECT count(*) FROM public.point_transactions t WHERE t.user_id=p.id) AS tx,
       (SELECT count(*) FROM public.activity_logs a      WHERE a.user_id=p.id) AS activity
  FROM public.profiles p ORDER BY p.display_name;
```

Do the reconciliation **in the scratch container**, so the source project is
never modified and a mistake costs only a re-restore. Then **re-export** — do
not hand-edit `identities.tsv`, which is only half the export; the data dump
still holds the rows you removed, and the mismatch produces exactly the
orphans the verify step exists to catch.

Two more identity notes:
- **Read the identity provider's usernames from its own config**, never infer
  them from display names. Authelia's `users_database.yml` had `famke` with
  displayname `Famke`, while the app's profile said `Famke de Jong`. A wrong
  username is the worst failure mode available: the person logs in
  successfully and lands on a brand-new empty profile, with no error anywhere.
- Budget a human round-trip for the mapping. Only the family can say which
  account is whose; no script can check that a mapping is *right*, only that
  it exists.

### Things that only fail in a browser

Two production-breaking bugs passed every `curl` check.

- **CORS forces the API to be same-origin, not a sibling subdomain.** A
  preflight `OPTIONS` carries no cookies by spec, so a `forward_auth` gate
  cannot authorize it and the browser sees a bare `TypeError: Failed to
  fetch`. **`curl` never reveals this** — it sends no `Origin`, so CORS is
  never engaged and every command-line check passes. Put the API under the
  app's own hostname (`handle_path /rest/v1/*`, `handle /auth/*`,
  `handle /storage/*`) rather than on `<app>-api.hopsakee.top`.
- **PostgREST has no `_FILE` convention.** The rest of the box uses
  `*_FILE=/run/secrets/x`; PostgREST's own syntax is
  `PGRST_JWT_SECRET=@/run/secrets/jwt_secret`. Configured the box's way it
  ignores the variable **in silence**, answers HTTP 500 `PGRST300 "Server
  lacks JWT secret"` to every authenticated request, and still passes a
  container healthcheck. Add a deploy guard that proves the secret loaded —
  a garbage bearer token must be rejected with **401**, not 500:

  ```bash
  docker exec <shim> node -e \
    'fetch("http://<rest>:3000/",{headers:{Authorization:"Bearer not.a.token"}})
       .then(r=>console.log(r.status)).catch(()=>console.log("000"))'
  ```

  It needs no secret of its own and never touches the database.

**Generalise both:** a Tier-B port's verification must include loading the app
in a real browser before it is called done. And check the UI distinguishes
*error* from *empty* — this app's admin page rendered a failed query
identically to "no pending users", which is what hid the PGRST300 bug for an
hour while the queue actually had someone in it.

### What the backup gate actually is

`MIGRATION-PLAN.md`'s hard gate is a tested restore. What doing it for real
added to that:

- **`RESTORE OK` is the gate** — a snapshot replaying into a clean Postgres
  under `ON_ERROR_STOP=1`. No state of the live database can affect that.
- **Comparing restored row counts against *live* is a separate freshness
  check**, and it is only meaningful if the snapshot was taken moments ago.
  Run it against a stale snapshot mid-cutover — when the database is being
  reset on purpose — and it reports `MISMATCH` on every table and declares a
  perfect backup broken. Take a fresh snapshot as the first line of the test.
- Run the gate against **test data, before real data exists**. An empty
  restore proves nothing, and waiting for real data inverts the point.
- **Image/file volumes are not in any database dump.** Back them up
  separately, and move them separately at cutover — the rows arrive intact
  and every image renders blank otherwise.
- **A backup destination must not be a two-way sync folder.** Jelle's rule,
  and it decides where the off-box copy lands. A Synology Drive **sync**
  share (`~/Drive`) propagates a local deletion to the NAS, so a copy there
  is a *replica*, not a backup: whatever deletes the local file deletes the
  remote one. The destination has to be the source of an upload-only
  **Backup** task (`~/Drive-bup`, `sync_direction=1`,
  `ignore_local_remove=1`), whose version store is what holds the history.
  This matters more than it looks, because the snapshots are deliberately
  **one undated file per artifact, overwritten in place** — the NAS version
  history *is* the retention mechanism, so pointing the pull at a sync share
  leaves no retention at all. When a backup path moves, "is the new
  destination still the upload-only Backup source?" is the first thing to
  re-check, and it is worth a probe rather than a memory: every destination
  in these scripts is a `${VAR:-default}` read, so an override in a plist or
  a compose file can move it without touching the default anyone greps for.

### Operational notes

- **The deploy does `git reset --hard origin/main` on the app checkout.** So a
  fix must be *merged* before deploying; a manual `git pull` in that directory
  is at best redundant and creates drift the next deploy silently discards.
  Two rounds were lost to running a script whose fix was still sitting in an
  unmerged PR.
- **`wait_healthy` only checks the first service in a compose file.** A
  multi-container stack can report a green deploy with a broken dependency.
- **Two shell-script lessons that each cost a round**, and generalise well
  beyond this port:
  - Under `set -e`, a failed command substitution makes the **assignment**
    fail, so `x=$(cmd)` exits the script *before* any `[ -z "$x" ]` check can
    report anything. Use `|| true` on the assignment.
  - Never send a probe's stderr to `/dev/null`. An unreachable host, a
    rejected password, a paused project and a stopped docker daemon all look
    identical once the only diagnostic is discarded. Capture it and print it.
- **Bare SQL blocks in a runbook get pasted into a shell.** Write them as a
  runnable `docker exec ... psql <<SQL` heredoc, or expect
  `GRANT: command not found`.
- **Fetch `origin/main` before writing a change — in every repo the change
  touches, not just the one you are working in.** Asked to move the backup
  paths to `~/Drive-bup`, I wrote a complete set of edits for the Mac-side
  repo from a clone two days stale. `origin/main` already had all of it, and
  had it with the *correct* rationale where mine had the reasoning backwards:
  I had asked "is `~/Drive-bup` a sync folder?", been told yes, and built on
  that, when the whole point of the move is that it is **not** one. The work
  was discarded rather than pushed as a duplicate. A multi-repo port means
  more than one clone drifts, and a stale one does not announce itself — it
  reads as a repo where the work simply has not been done yet.
- **Do not confuse an open PR with work owed.** A docs-only PR in a repo with
  no CI cannot change state without the human; a scheduled re-check of it
  produces nothing but noise. Poll something only when it can move on its own.

## Not yet answered
- SQLite backup routine on the box. **No longer blocking jonkies-tody** — that
  app's backend decision landed on Postgres. Still open for any app that ends
  up on SQLite.
- Whether one SQLite file can safely serve more than one container. Same
  status: not blocking jonkies-tody any more.
- ~~Whether a `pg_dump` snapshot routine with a tested restore is what Jelle
  accepts as closing the hard gate for Postgres-backed apps.~~ **Answered:
  yes.** Built, scheduled by cron on the box, and restored for real into a
  throwaway Postgres with row counts and balances compared — see "What the
  backup gate actually is" above for the two refinements that came out of
  running it. Real data was migrated only after it passed.
- **Off-box backup transport is still open** for every app.
  jonkies-tody's snapshots exist on the Hetzner box and restore correctly,
  but nothing pulls them off it yet, so the box remains a single point of
  failure for the family's data. The design is settled (the Mac Mini pulls
  and drops one undated file per artifact into `~/Drive-bup/jonkies-tody/`,
  the source of an upload-only Synology **Backup** task — *not* a Drive sync
  share; see the destination rule under "What the backup gate actually is")
  and the script is written in `hoggle-macmini`; what is left is the
  firewall/key/scheduling work on the Mac. Do not treat a Tier-B port as
  finished until this exists for it.
- ~~Which backend jonkies-tody talks to.~~ **Answered**: neither self-hosted
  Supabase nor a SQLite rewrite, but plain Postgres + PostgREST + an
  Authelia→JWT shim (see the Tier-B pattern above, and
  `jonkies-tody/PORT-NOTES.md`). Pending Jelle's confirmation.
- ~~Whether jonkies-tody gets one login prompt or two.~~ **Answered as a
  consequence**: one. Authelia becomes the identity provider and Google
  sign-in is dropped, so no second login, no second approval list, and no
  Google OAuth client to register at all.
