# Port learnings — jonkies-tody

Date: 2026-09-11 (scoping) — 2026-09-19 (cutover completed)
Ported by: Claude Code on the web
Tier: B (first one) — **completed: live on Hetzner with the family's real data**

Status: done. Path C (plain Postgres + PostgREST + an Authelia→JWT shim) was
prototyped against synthetic data in the scoping session, then built, deployed
and cut over on the real box. The family's data moved off Lovable on
2026-09-19 and verified row-for-row: 5 identities, all nine table counts
identical to the source, every balance identical, no ledger drift, no orphans.
See `jonkies-tody/PORT-NOTES.md` for the decision memo and
`jonkies-tody/docs/CUTOVER-RUNBOOK.md` for the procedure as actually executed.

Everything above the "real server" heading below is from the scoping session
and is left as it was written. The cutover section is the new material.

**One correction to those scoping notes, flagged here rather than rewritten
into them:** the net policy count is **28 in `public`** (plus 4 on the
`storage.objects` stub), not the 31 stated twice below. The sandbox tally was
wrong; the live database is authoritative —
`select count(*) from pg_policies where schemaname='public'`.

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

Written after the cutover. The generalisable half is promoted into
`PORTING-PLAYBOOK.md` under "Tier-B cutover"; this is the narrative, including
the parts that were my mistakes.

### The two bugs that passed every sandbox check

Both were mine, both were design errors rather than typos, and both were
invisible to `curl`.

**The API cannot live on its own subdomain.** I put it on
`jonkies-api.hopsakee.top` behind the same Authelia gate and verified it
thoroughly — with `curl`. The browser then failed with a bare `TypeError:
Failed to fetch`. A CORS preflight `OPTIONS` carries no cookies by spec, so
`forward_auth` cannot authorize it. `curl` sends no `Origin`, so **none of my
verification ever engaged CORS at all.** The fix was to move the API
same-origin under the app's own hostname. The lesson is not about CORS: it is
that a verification method which cannot exercise the failure mode is not
verification, and I had no browser in the loop until Jelle was the browser.

**PostgREST has no `_FILE` convention.** The Path C prototype used
`-e PGRST_JWT_SECRET="$SECRET"` literally and worked. When the prototype became
compose files I wrote `PGRST_JWT_SECRET_FILE=`, matching every other service on
the box — and never re-tested that shape. PostgREST ignored it silently,
answered HTTP 500 `PGRST300` to every authenticated request, and **passed the
container healthcheck**, so the deploy was green. The correct syntax is
`PGRST_JWT_SECRET=@/run/secrets/jwt_secret`.
`deploy-jonkies-tody.sh` now carries a guard that proves the secret actually
loaded — a garbage bearer token must come back 401, not 500. It needs no
secret of its own and never touches the database.

The pattern joining the two: **I changed a thing after verifying it, and
treated the earlier verification as still valid.** Both times the change was
"make it consistent with the rest of the box".

### The UI hid one of them

The admin page rendered a *failed* query identically to an empty list — the
same green check and "Geen wachtende gebruikers". `usePendingProfiles()` throws
on a PostgREST error, react-query leaves `data` undefined, and the
`!data || length === 0` branch swallowed it. So while PGRST300 was breaking
every request, the screen calmly reported that nobody was waiting, when
somebody was. Fixed; worth checking every list screen in a ported app for the
same shape.

### The migration source was not what the plan assumed

The plan said `pg_dump` against `db.<ref>.supabase.co`. On a Lovable-built app
that is close to unusable: Supabase shows the database password once at
project creation and Lovable abstracts it away, so nobody has ever seen it;
direct connections are IPv6-only without the IPv4 add-on; and the pooler
alternative needs session mode for `pg_dump`.

Lovable's dashboard export — a `.backup` (pg_dump CUSTOM, zstd, from
PostgreSQL 17.6) plus a zip of the storage bucket — sidesteps all three. The
approach that worked was to **restore it into a throwaway container and point
the existing `01-export.sh` at that**, so the tested import and its guards
stayed untouched and the only new code was standing up a scratch server.

Three traps in that route, all now in the playbook: `pg_restore --schema=X`
does not create schema X (247 objects failed on one line's absence);
the source/target version gap means the dump must be sanitised for the older
target; and the box has no `postgresql-client` at all.

### Real data is messier than any test set

The export held **7 identities and 6 profiles**, not the 5 and 5 everyone
expected. Two children had signed up twice in the app's first week — and for
one of them, the account displaying as *a different name entirely* was the
real one, carrying 114 points and a full ledger, while the account with his
own name was empty.

Getting that backwards would have cost a child 114 points and their whole
history, because every table cascades from `profiles(id)`. What prevented it
was refusing to delete anything before a query proved which row was empty.

No script can check this. `02-import.sh` verifies that everyone has *a*
mapping, never that it is the *right* one. **Budget a human round-trip for
identity reconciliation on every Tier-B port**, and do the cleanup in the
scratch container so the source is never touched.

### What I would do differently

1. **Load the app in a browser before calling any Tier-B port verified.** Both
   production bugs were browser-only.
2. **Re-verify anything I changed after verifying it**, especially changes
   whose justification is consistency rather than correctness.
3. **Ask for the identity list early.** It is the one input no amount of
   tooling can validate, and it was available from day one.
4. **Expect the export to disagree with the family's own headcount**, and ask
   about duplicates before writing the mapping rather than after.
5. **Fetch every repo the change touches before writing a line of it.** This
   port spans four repos, and the one I reached for last was two days stale.
   See "The backup destination moved" below.

### The backup destination moved, and both halves of that were a lesson

Jelle moved the Mac-side backup folder from `~/Drive/Backup` to `~/Drive-bup`
and asked what that meant for the code and the deploy.

**The rule behind the move** is the part worth carrying forward: no backup
lives in a two-way sync folder. `~/Drive` is a Synology Drive *sync* share, so
a local deletion propagates to the NAS and the copy there is a replica, not a
backup. `~/Drive-bup` is the source of an upload-only *Backup* task
(`sync_direction=1`, `ignore_local_remove=1`). That is load-bearing here
precisely because these snapshots are one undated file per artifact,
overwritten in place — the NAS version store *is* the retention, so a sync
share would have left none.

**How I got there was the other lesson.** I asked "is `~/Drive-bup` a Drive
sync folder?", was told yes, and wrote a full set of edits on that basis —
from a clone two days stale. `origin/main` already carried the whole change,
dated 2026-09-20, with the rationale the right way round: `~/Drive-bup` is
deliberately *not* a sync folder. I discarded my version rather than push a
duplicate built on an inverted premise. Two things to keep: fetch before
building, in every repo the change touches; and when a one-word answer is the
hinge of the design, confirm it against the repo instead of the conversation.

### Still open

Off-box backup transport. Snapshots exist on the Hetzner box, are scheduled,
and restore correctly — but the Mac-side pull is not yet proven running
against the real box, so the box is still a single point of failure for the
family's data. Script written in `hoggle-macmini` and now pointed at
`~/Drive-bup/jonkies-tody/`; the firewall/key/scheduling work on the Mac
remains.
