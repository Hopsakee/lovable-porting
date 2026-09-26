# Starting prompt for porting one Lovable app

Copy the block below into a fresh session and fill the four placeholders:
`<APP>`, `<LOVABLE_ID>`, `<SUBDOMAIN>`, `<GATED|PUBLIC>`. Everything else is
the same for every port and is deliberately not a placeholder — the
constraints, the gates and the "do not invent" rule are what make the result
trustworthy, and they are the first things to erode if each session rewrites
them.

Keep this template current: when a port teaches something that changes how the
*next* one should start, change it here, not only in the learnings file.

---

Port the Lovable app **`<APP>`** (Lovable project `<LOVABLE_ID>`) off Lovable
hosting and onto my Hetzner box at `hopsakee.top`.

## Read before proposing anything

In `Hopsakee/lovable-porting`:

- `PORT-SCOPE.md` — which apps, in what order, and what was decided. It
  overrides `MIGRATION-PLAN.md` on scope.
- `PORTING-PLAYBOOK.md` — how to do a port. Every line is marked VERIFIED (a
  real port proved it) or ASSUMED (it came from planning). Do not promote an
  ASSUMED line without a run that exercised it. The Tier-B cutover section is
  where the previous port's avoidable rounds are already written down.
- `learnings/jonkies-tody.md` — what actually went wrong last time.

In `Hopsakee/jonkies-tody`, the worked example of a finished Tier-B port:
`PORT-NOTES.md` (why Path C), `docs/CUTOVER-RUNBOOK.md` (the steps, in order),
`backend/migrate/0{1,2,3}-*.sh` (export, import, verify), `backend/compose.yaml`
and `backend/auth-shim/`.

In `Hopsakee/hopsakee-server`: `config/jonkies-tody/`, `config/jonkies-tody-api/`,
`server_setup/deploy-jonkies-tody.sh`, and the shared `Caddyfile` and Authelia
`configuration.yml`.

## The target pattern

**Path C**, proven once on `jonkies-tody`: plain Postgres + PostgREST + a
custom Authelia→JWT shim. Three containers. No GoTrue, no Kong, no
Supabase-Storage, no Studio. RLS policies, triggers and functions move into the
new database unchanged; `auth.uid()` is three lines of SQL over
`current_setting('request.jwt.claims', true)`.

The API must be **same-origin** with the app. A cross-origin API cannot work
behind `forward_auth`: the CORS preflight `OPTIONS` carries no cookies, so
Authelia cannot authorize it. `curl` sends no `Origin`, so curl-based
verification will never catch this — it only fails in a browser.

If this app does not fit Path C, say so before building anything, and say what
it needs instead.

## What I want from you, in this order

1. **Survey the app first, and report before writing code.** Schema (tables,
   RLS policies, triggers, functions, enums), how it authenticates, what it
   stores outside the database (images, files), which edge functions exist and
   what they do, which external APIs and secrets it needs, and how much real
   data is in it. Count things by querying, not by reading migrations —
   migrations lie about the live state.
2. **Name the parts that do not fit Path C**, with a proposal for each. Edge
   functions are the usual one: Postgres and PostgREST cannot make outbound
   HTTP calls, so anything server-side needs its own small container or has to
   move into the frontend. Decide this before starting, not during.
3. **Write the port notes and the cutover runbook** in the app's own repo,
   following `jonkies-tody`'s shape. Every SQL block that a human will run must
   be a runnable `docker exec … psql <<SQL` heredoc — a bare ```sql block gets
   pasted into a shell and answers `GRANT: command not found`.
4. **Then build**: Dockerfile, compose, the migrate scripts, the
   `hopsakee-server` config and deploy script, the Caddy block, the Authelia
   rule if gated.

Deploy target: `<SUBDOMAIN>.hopsakee.top`, `<GATED|PUBLIC>`.

## Constraints — these are not negotiable

- **I have no Hetzner credentials in this session, and I should not ask for
  them.** Never deploy, never SSH to the real box, never run the export or
  import against real data. I run those; you write them and tell me exactly
  what to run and what output to expect.
- **Never invent a URL, container name, config key, path or version.** Read it
  from the repo or ask me. If you did not run a command and read its real
  output in this session, do not call anything "verified" — say what you
  checked and what you are assuming.
- **The hard gate**: no app with real data goes live until a snapshot routine
  with a *tested restore* exists for it. Test the restore with test data,
  before real data exists — an empty restore proves nothing, and waiting for
  real data inverts the point. `RESTORE OK` into a clean Postgres under
  `ON_ERROR_STOP=1` is the gate; comparing counts against the live database is
  a separate freshness check and is only meaningful against a snapshot taken
  moments earlier.
- **A port is not finished until its backups leave the box.** Snapshots that
  only exist on Hetzner leave the box a single point of failure.
- **Backups never live in a two-way sync folder.** `~/Drive` propagates a
  local deletion to the NAS, so a copy there is a replica. The destination is
  `~/Drive-bup`, the source of an upload-only Synology Backup task.
- **Load the app in a real browser before calling the port done.** Both
  production bugs in the last port were browser-only, and one was hidden for an
  hour by an admin screen that rendered a failed query identically to "nothing
  here". Check that the UI distinguishes *error* from *empty*.
- **If a decision is genuinely open — which is not obvious from the repo —
  stop and ask.** Do not pick one and build silently.
- **Fetch `origin/main` in every repo you touch before writing anything.** A
  multi-repo port means more than one clone drifts, and a stale one reads
  exactly like a repo where the work has not been done yet.

## When it is done

Update `PORT-SCOPE.md` (status), add `learnings/<APP>.md` including what went
wrong, and fold anything reusable into `PORTING-PLAYBOOK.md` marked VERIFIED.
The point of doing these one at a time is that each port improves the playbook;
a port that teaches nothing new means the playbook is working, and a port that
teaches something and does not update it wastes the lesson.
