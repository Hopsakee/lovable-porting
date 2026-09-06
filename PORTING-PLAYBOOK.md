# Porting playbook

Status: v1, one pilot port done (hopsakee-decimal-finder / findjd, Tier-A static).
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
- ASSUMED, explicitly overridable: each app defaults to
  `<repo-name>.hopsakee.top`, but this is a default, not a rule — the pilot's
  own subdomain is `hd.hopsakee.top`, Jelle's deliberate choice, while the
  repo/container/deploy-script names all stayed `hopsakee-decimal-finder`
  (mirroring the pre-existing `pkw-web` container /
  `datalab-knowledge.hopsakee.top` subdomain precedent, where those two names
  already differ).

## Known traps
- ASSUMED, still untested: Vite reads env at BUILD time. `VITE_*` must be
  compose `build.args`, never `environment:`. Get this wrong and the app
  silently ships pointing at the old cloud project. **The pilot app uses zero
  `VITE_*` vars**, so this trap has still never actually been hit by a real
  port — treat it as live risk for the next Vite app that has any build-time
  env vars (almost certainly the first Tier-B/Supabase one).
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
