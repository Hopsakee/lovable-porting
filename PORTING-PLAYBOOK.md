# Porting playbook

Status: v1, two Tier-A static ports done (hopsakee-decimal-finder / findjd —
the pilot; ren-afstand — second app, same pattern reused unchanged).
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

## Known traps
- ASSUMED, still untested: Vite reads env at BUILD time. `VITE_*` must be
  compose `build.args`, never `environment:`. Get this wrong and the app
  silently ships pointing at the old cloud project. **Two static ports in
  (pilot + ren-afstand), zero apps have used any `VITE_*` vars** — this trap
  has still never actually been hit by a real port. Don't let two clean
  static ports create false confidence; treat it as live, unproven risk
  until the first Tier-B/Supabase app actually exercises it.
- VERIFIED (both ports): the committed `package-lock.json` from a
  Lovable-exported repo reliably fails `npm ci` under this sandbox's npm
  version (rollup/vitest optional-dependency drift — not the same missing
  packages both times, but the same failure shape). Two-for-two now, not
  pilot-specific bad luck — **expect to `rm package-lock.json && npm
  install` on every port** and budget for it up front rather than
  discovering it each time.
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

## Not yet answered
- SQLite backup routine on the box.
- Whether one SQLite file can safely serve more than one container.
