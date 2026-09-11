# Port learnings — ren-afstand

Date: 2026-09-11
Ported by: Claude Code on the web, session 1 (same session as the pilot)
Tier: static

## What I assumed that was wrong

- **`Hopsakee/ren-afstand` "doesn't exist"** — it was **private**, not
  missing. This session's GitHub access is scoped to a fixed repo list at
  session start; a private repo not on that list looks identical to a
  genuinely absent one from inside the session (no `list_repos`/`add_repo`
  visibility either, since `add_repo` also can't see private repos it isn't
  already scoped to). I asked Jelle to create a new empty repo and, in
  parallel, had a background agent reconstruct the app's source from the
  live Lovable project (`ren-afstand.lovable.app`) as if starting from
  nothing. Jelle corrected this: the repo already existed with real commit
  history, just private — he flipped it public. **Consequence avoided by
  checking before pushing**: I diffed the reconstruction against the real
  `origin/main` before doing anything else. The file list matched exactly
  (86/86 files) and only `bun.lockb` (binary lockfile churn) and some
  trailing-whitespace-only lines in `index.html` differed — so the
  reconstruction was faithful, but I still rebuilt the actual working branch
  on top of the **real** git history (`origin/main`) rather than push my
  from-scratch reconstruction over it, so the genuine Lovable commit history
  (30+ real commits) survives instead of being replaced by a synthetic root
  commit. Only the new deploy-artifact files were carried over from the
  reconstruction; the app source itself came from the real repo.
- **Docker daemon availability is not constant across turns in this
  sandbox.** Unlike the pilot session where `dockerd` was already running,
  this turn's sandbox had no daemon (`docker info` → "Cannot connect").
  Starting `dockerd` manually in the background worked
  (`nohup dockerd &`, then a few seconds' wait) — worth trying before
  concluding Docker verification is unavailable.

## Where the iterations went

1. **Realizing the repo was private, not absent** — the actual time cost
   was small once Jelle answered directly, but this is the same failure
   shape as the pilot's "repo looks empty" confusion, just one layer
   different (private vs. genuinely new). Worth a standing checklist item:
   before assuming a repo doesn't exist or is a blank placeholder, ask
   "could this be private?" — a session's GitHub scope silently drops
   private repos it wasn't told about, it doesn't report them as
   access-denied.
2. **Verifying the reconstruction was safe to discard in favor of real
   history** — one `git ls-tree` file-list diff + one `git diff --stat`
   between the reconstructed root commit and real `origin/main`, both fast.
   Cheap insurance against silently clobbering real commit history; do this
   every time a "the repo is empty, let me reconstruct from Lovable" branch
   turns out to have been wrong.
3. **Getting Docker working again** — `dockerd` wasn't running this turn;
   starting it manually (`nohup dockerd > /tmp/.../dockerd.log 2>&1 &`) and
   waiting ~5s fixed it. Same `SELF_SIGNED_CERT_IN_CHAIN` sandbox-proxy issue
   as the pilot inside `npm ci`, same throwaway `NODE_EXTRA_CA_CERTS`
   workaround — but this time `apt-get install ca-certificates` (used to
   register the CA system-wide in the pilot's first draft) also got
   403'd by the sandbox's HTTP proxy. Simplified to just pointing
   `NODE_EXTRA_CA_CERTS` directly at the copied `.crt` file — no
   `update-ca-certificates`/apt needed at all, since Node reads that env
   var itself. Simpler and more reliable than the pilot's approach; worth
   using this version next time instead.

## Gotchas discovered

- **Same `package-lock.json` drift as the pilot, on a completely different
  app** (`npm ci` → `Invalid: lock file's @jridgewell/sourcemap-codec@1.5.0
  does not satisfy @jridgewell/sourcemap-codec@1.6.0`, plus several missing
  packages). Two-for-two now — this isn't pilot-specific bad luck, it's a
  real, recurring trap for every Lovable-exported app under this sandbox's
  npm version. `rm package-lock.json && npm install` fixes it every time;
  stop treating this as a one-off and just expect to regenerate the
  lockfile on every port.
- **This app confirms three of the playbook's still-ASSUMED traps as
  real-but-inapplicable-here, the same way the pilot did**: no `VITE_*` env
  vars (`grep -rn "import.meta.env"` / `grep -rl "VITE_"` both empty), no
  `.wasm` file, no PWA/service-worker (`grep -rli
  "serviceworker\|vite-plugin-pwa\|registerSW"` empty). Two apps in and the
  `VITE_*` build-arg trap in particular is still completely unexercised —
  don't let two static apps in a row create false confidence; the first
  Supabase-backed app is still the real test for it.
- `NODE_EXTRA_CA_CERTS` pointing straight at a copied `.crt` file (no
  `apt-get`/`update-ca-certificates` step) is enough to make `npm ci` trust
  this sandbox's intercepting proxy inside a Docker build — simpler than
  what the pilot did and worth using directly next time instead of
  reaching for `apt-get install ca-certificates` first.
- `dockerd` isn't guaranteed to already be running in a fresh turn of this
  sandbox, even mid-session — `docker info` tells you immediately
  ("Cannot connect to the Docker daemon"), and `nohup dockerd &` + a short
  wait brings it up.

## What should become a template

- Nothing new to add here beyond what the pilot already established
  (`node-static.Dockerfile`, the Caddyfile no-cache/immutable pattern,
  `deploy-<app>.sh`/`config/<app>/compose.yaml` shape) — this port copied
  all of it unchanged, which is itself a good sign the shared pattern holds
  up on a second, independently-built app.
- Worth adding as an explicit pre-flight step for every future port: verify
  a repo is genuinely absent (not just outside this session's private-repo
  scope) before reconstructing its source from Lovable — a quick "could
  this be private?" question to Jelle costs less than a from-scratch
  reconstruction that then has to be verified against and discarded in
  favor of real history.

## Proposed playbook edits

See the diff applied directly to `PORTING-PLAYBOOK.md` in this same PR:
added a new item under "Known traps" about `package-lock.json` regeneration
now being expected/routine rather than a one-off; added a note about
checking "private, not absent" before reconstructing a repo from Lovable;
simplified the sandbox-CA-trust verification tip. The `VITE_*` build-arg
trap stays ASSUMED — still unexercised by any port so far.

## Open questions for Jelle

- None outstanding — the subdomain-naming and Alpine/Debian questions from
  the pilot were already resolved before this port started, and this app
  needed no new judgment calls.

## What only showed up on the real server

(Jelle fills this in after deploying. Leave empty.)
