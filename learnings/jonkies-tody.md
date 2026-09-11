# Port learnings — jonkies-tody

Date: 2026-09-11
Ported by: Claude Code on the web
Tier: B (first one) — **scoping session only, not a completed port**

Status: the container half is built and verified; the backend architecture
decision is open and deliberately left open. See `jonkies-tody/PORT-NOTES.md`
for the decision memo itself. This file is only the process learnings.

## What I assumed that was wrong

- **I expected the "is this repo private or absent?" mixup again** (the trap
  ren-afstand left behind) and checked for it first. It did not apply —
  `Hopsakee/jonkies-tody` was already cloned into the session with 87 real
  commits and all 26 migrations present. Worth recording as a negative
  result: the check is cheap, but three ports in, the failure has only
  actually occurred once.
- **I expected the newer Lovable template
  (`new_style_vite_react_shadcn_ts_testing_2026-01-08`) to differ in ways
  that mattered.** It barely does. The differences are: a `vitest.config.ts`
  + `src/test/setup.ts` + one placeholder test, and `test`/`test:watch`
  scripts. Same Vite/React/shadcn/Tailwind scaffolding, same
  `lovable-tagger` dev plugin, same `src/integrations/supabase/client.ts`
  shape. Nothing about the container build, the Caddy serving pattern or the
  auth wiring changes because of the template. The one place it *does* bite
  is the lockfile trap — see below.

## Gotchas discovered

- **The `VITE_*` build-arg trap, finally exercised — and it fails silently
  by construction, not by oversight.** The playbook framed it as "remember to
  pass the vars". The actual mechanism is that the repo **commits its
  `.env`** (git-tracked; `.gitignore` has no `env` entry), so Vite picks the
  old Lovable cloud project's URL straight out of the build context and a
  zero-config build ships a working-looking app pointed at the wrong
  backend. Verified both directions, then verified the fix
  (`rm -f .env` + a build-time guard that exits 1) through a real
  `docker build`. Full detail promoted into `PORTING-PLAYBOOK.md`.
- **Same `package-lock.json` drift, third app running** — but this time the
  missing entries were the *testing-library* tree, i.e. exactly the deps the
  newer template adds. The newer template does not fix the trap; its extra
  devDeps are what's missing from the lockfile it ships.
- **Replaying the migration chain against a throwaway Postgres is cheap and
  should be routine for Tier-B.** ~40 lines of stubs for `auth.users`,
  `auth.uid()`, `storage.*` and the three Supabase roles, then apply the
  migrations in filename order. Two payoffs: it proves the chain applies at
  all (25/26 here, first try), and querying `pg_policies`/`pg_proc`/
  `pg_trigger` afterwards yields the **net** schema rather than a count of
  `CREATE` statements that later migrations already dropped. That net number
  — 31 policies, 11 functions (10 `SECURITY DEFINER`), 8 triggers — is what
  makes the rewrite-cost argument concrete instead of hand-wavy. Promoted
  into the playbook.
- **The single replay failure was a stub gap, not an app bug**, and was
  itself informative: that migration calls `storage.extension(name)`, a
  Supabase Storage built-in, inside an upload policy. Migrations depending
  on Supabase-specific helpers are exactly the ones needing hand-rewriting
  if an app leaves Supabase — worth grepping for deliberately.
- **`docker pull` worked fine this session** (both `caddy:2.8` and
  `node:20-alpine`, plus `postgres:15-alpine`), unlike the pilot's 403s on
  registry blob CDNs. `dockerd` again wasn't running at session start and
  again came up with `nohup dockerd &` — that part is now three-for-three.
- **The `NODE_EXTRA_CA_CERTS`-pointing-straight-at-the-.crt trick from
  ren-afstand worked verbatim** for `npm ci` inside the build container. No
  `apt-get`, no `update-ca-certificates`. Confirmed as the right default.
- **`add_repo` on `hopsakee-server` was worth doing rather than
  reconstructing its patterns from the playbook's prose.** It gave the real
  `base/node-static.Dockerfile`, the real Caddyfile (including that
  `authelia_forward_auth` already copies `Remote-User`/`Remote-Email`/
  `Remote-Groups` — directly relevant to the single-sign-in question), the
  real Authelia groups (`me`/`admins`/`friends`/`work`, no family group),
  and `utils.sh`'s `wait_healthy`/`reload_caddy`. Cheap, and it removed
  guesswork from every artifact written.

## What should become a template

- **The `Dockerfile`'s two-part VITE guard** (`rm -f .env` in the build
  stage + fail-the-build-if-unset) should be copied into every remaining
  Tier-B port. It is the only thing standing between a mistyped compose file
  and a production app quietly reading and writing the *old* cloud database.
- The migration-replay-against-throwaway-Postgres recipe, as a standard
  Tier-B scoping step before any rewrite estimate is offered.
- `Caddyfile`/`compose.yaml`/`deploy.sh`/`caddy-snippet.txt` are the same
  shapes as the two Tier-A ports; the only real difference is `build.args`
  in compose and the gated (rather than public) Caddy block.

## Open questions for Jelle

All in `jonkies-tody/PORT-NOTES.md` rather than duplicated here — the
backend decision (Path A self-hosted Supabase vs Path B SQLite rewrite, with
a recommendation and both paths costed), the one-prompt-or-two login
question, Authelia family accounts, the subdomain name, and the expected
GitHub-App-install round-trip before a push to this repo works.

## What only showed up on the real server

(Jelle fills this in after deploying. Leave empty.)
