# Port learnings — jonkies-tody

Date: 2026-09-11
Ported by: Claude Code on the web
Tier: B (first one) — **scoping session only, not a completed port**

Status: the container half is built and verified; the backend architecture
decision is **answered** — Path C (plain Postgres + PostgREST + an
Authelia→JWT shim), prototyped and verified against synthetic data. See `jonkies-tody/PORT-NOTES.md`
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

- **The predicted GitHub-App-install round-trip did not happen.** The
  playbook calls this three-for-three ("expect a 403 on every app repo the
  first time a port touches it"); here the first push to `jonkies-tody` and
  the PR both went through cleanly. The plausible difference is that this
  repo was in the session's scope from the start rather than added
  mid-session via `add_repo` — `hopsakee-server`, which *was* added
  mid-session, was only ever read from, so this session doesn't settle it.
  Treat the round-trip as likely but not certain, and don't pre-emptively
  ask Jelle for it before a push has actually failed.

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

## The most useful thing I got wrong

**I recommended self-hosted Supabase, and Jelle's pushback was right.** My
reasoning was sound about *what mattered* — the 31 RLS policies and 8 triggers
must stay enforced in the database, not become app code — but I let that
conclusion pick the wrong implementation, because I framed the question as the
plan had framed it: Supabase or SQLite. I optimised for "change the least app
code" and treated the five-container stack as the price of keeping the security
model.

The thing that unlocked the third option was already sitting in my own session
output: I had **written `auth.uid()` as a three-line stub myself**, hours
earlier, to replay the migrations — and watched all 31 policies work against
it. That was direct evidence that the policies depend on a JWT claim, not on
Supabase, and I didn't draw the inference until Jelle objected.

Lesson for the next port, worth more than any of the technical notes below:
when a decision is presented as two options, check whether the constraint that
makes it binary is real. Here it wasn't — "keep RLS" and "don't run Supabase"
were never actually in conflict. And when you've already built something that
demonstrates a mechanism, ask what else it proves.

## Open questions for Jelle

All in `jonkies-tody/PORT-NOTES.md` rather than duplicated here. Both
architecture questions are now answered (Path C; one login prompt); what is
left is confirmation, the live balance-drift bug, whether the family accepts
signing in with Authelia instead of Google, Authelia family accounts, the
subdomain name, and whether a `pg_dump` routine closes the hard gate.

## What only showed up on the real server

(Jelle fills this in after deploying. Leave empty.)
