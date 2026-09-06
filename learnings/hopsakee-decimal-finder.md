# Port learnings — hopsakee-decimal-finder

Date: 2026-09-06
Ported by: Claude Code on the web, session 1
Tier: static

## What I assumed that was wrong

- **The app repo and the Lovable project were empty.** `Hopsakee/hopsakee-decimal-finder`
  had only `README.md`/`LICENSE` (single "Initial commit"), and no Lovable
  project matching that name existed in the workspace. Jelle clarified: this
  app **is** `findjd` ("Johnny Finder" in Lovable, `Hopsakee/findjd` on
  GitHub) under a different repo name — the real source was copied over from
  `findjd`, not written from scratch.
- **`caddy:2.8` is Alpine, not Debian.** I wrote the shared base image's
  non-root user creation using `groupadd`/`useradd` (the syntax every Python
  app on the box uses, via `ghcr.io/astral-sh/uv:...-trixie-slim`). Build
  failed immediately: `groupadd: not found`, exit 127. `docker run --rm
  --entrypoint sh caddy:2.8 -c 'cat /etc/os-release'` shows Alpine 3.20.
  Fixed with BusyBox's `addgroup -g 1001 -S appuser` / `adduser -S -D -u 1001
  -G appuser -s /sbin/nologin -h /home/appuser appuser`.
- **Caddy's `path` matcher doesn't follow `try_files` rewrites.** First
  Caddyfile draft matched `@nocache path /index.html /sw.js ...` and applied
  `Cache-Control: no-cache`. `curl -I http://.../sw.js` showed the header
  correctly, but `curl -I http://.../` (plain root) showed **no**
  `Cache-Control` header at all, even though `try_files {path} /index.html`
  was correctly serving `index.html`'s bytes. The matcher checks the original
  request URI, not the rewritten target — had to add `/` to the matcher's
  path list explicitly.
- **`package-lock.json` in the `findjd` source was stale relative to this
  sandbox's npm** (`npm ci` failed: `Invalid: lock file's
  @rollup/rollup-linux-x64-gnu@4.24.0 does not satisfy
  @rollup/rollup-linux-x64-gnu@4.63.1`, plus several missing-platform rollup
  optionals). Not a Lovable-specific problem — just lockfile drift. Fixed by
  `rm package-lock.json && npm install` and committing the regenerated file.
  Worth checking whether this reproduces on the actual deploy Node version.

## Where the iterations went

1. **Finding out the app/docs didn't exist yet** (~30 min of reading empty
   repos, checking Lovable's project list by name and by description search,
   confirming via commit history that both `hopsakee-decimal-finder` and
   `lovable-porting` were single-commit, same-day repos) before asking Jelle
   directly. Biggest time cost of the whole port — everything after
   clarification went fast.
2. **Docker registry access in the sandbox.** `docker pull` of both
   `docker.io/library/caddy:2.8` and `ghcr.io/astral-sh/uv:...` came back 403
   from the session's egress proxy. The blocked hosts turned out to be the
   registries' CDN *blob* backends (`production.cloudfront.docker.com`,
   `pkg-containers.githubusercontent.com`), not the registry hostnames
   themselves — worth remembering for the next port in a similarly-sandboxed
   session: allowlisting `docker.io`/`ghcr.io` by name may not be enough if
   the policy UI doesn't treat them as resolving categories.
3. **`npm ci` inside the Docker build container failed with
   `SELF_SIGNED_CERT_IN_CHAIN`** even after registry pulls started working —
   this session's proxy TLS-intercepts *all* outbound HTTPS, including from
   inside containers, and containers don't inherit the host's CA-trust env
   vars. Verified this is sandbox-only (not a real app/Dockerfile bug) with a
   throwaway `Dockerfile.sandboxtest` that added the proxy's CA bundle via
   `NODE_EXTRA_CA_CERTS` — built and ran cleanly. That file and the CA copy
   were deleted before committing; the real `Dockerfile` has no CA-trust
   workaround in it and doesn't need one on the actual Hetzner box (ordinary
   internet, no interception).

## Gotchas discovered

- `docker run --rm --entrypoint sh <image> -c 'cat /etc/os-release'` before
  writing any user-creation RUN line — don't assume Debian just because every
  other app on the box happens to be.
- `curl -I` every path a Cache-Control rule is supposed to cover, including
  `/` itself, not just the literal filenames in the matcher — Caddy's `path`
  matcher is a real gap between "looks right" and "is right" here.
- A real content change to a component only produces a new hashed asset
  filename if it survives minification — a comment-only edit round-tripped
  to the *same* `index-DXdwfVzT.js` hash across two builds; only editing an
  actually-rendered string (`SearchInput.tsx`'s placeholder) produced a new
  hash (`index-WaJ12R3C.js`). Don't use a comment as your "prove the cache
  invalidates" test case.
- `docker build --network host` did **not** fix the in-container TLS
  interception (same `SELF_SIGNED_CERT_IN_CHAIN` either way) — the fix is CA
  trust, not network mode. The proxy's own README already says this
  (`/root/.ccr/README.md`, "docker build / docker run" section); I confirmed
  it empirically before believing it.

## What should become a template

- `hopsakee-server/base/node-static.Dockerfile` — the shared static-serve
  base image (BusyBox non-root user, healthcheck, Caddy). Generic as-is for
  any Tier-A static app; nothing app-specific needs to change per use.
- This app's `Caddyfile` (the no-cache/immutable/wasm-content-type pattern) is
  reusable near-verbatim for the next static Vite/PWA app — only the
  `root */srv/app` path and any app-specific matchers would need review.
- `deploy-hopsakee-decimal-finder.sh` / `config/hopsakee-decimal-finder/compose.yaml`
  — copy directly for the next Tier-A app, renaming the service/container
  throughout.

## Proposed playbook edits

See the diff applied directly to `PORTING-PLAYBOOK.md` in this same PR:
promoted the Node-static-serve-via-Caddy line, the non-root/fixed-UID line,
and the `.wasm` content-type line from ASSUMED to VERIFIED (with the Alpine
correction folded in); added a new VERIFIED line about the `try_files`/`path`
matcher interaction; left the PWA-stale-`index.html` line ASSUMED→VERIFIED
since this port explicitly exercised and fixed it; left the `VITE_*`
build-arg line ASSUMED since this app doesn't use any `VITE_*` vars and so
never actually exercised it.

## Open questions for Jelle

- **Subdomain naming precedent.** You picked `hd.hopsakee.top` for this app
  while the repo/container/deploy-script names stay
  `hopsakee-decimal-finder` throughout — mirroring the existing
  `pkw-web`→`datalab-knowledge.hopsakee.top` precedent where the container
  name and public subdomain already differ. Worth confirming this is the
  pattern you want going forward for the rest of the Tier-A rollout
  (`ren-afstand`, `skillkeep`), or whether it was specific to this app.
- **`package-lock.json` drift.** I regenerated it with a clean `npm install`
  in this sandbox (Node 22.22.2, npm 10.9.7/10.9.8 across the two
  environments touched). Worth a `npm ci` smoke test on whatever Node version
  actually runs the build on the Hetzner box, in case there's a different
  drift there.
- **This app never exercises the `VITE_*` build-arg trap or `.wasm` serving**
  (it uses neither). Both remain real risks for the next Tier-A/Tier-B app
  that does — don't treat this port as having proven them.

## What only showed up on the real server

(Jelle fills this in after deploying. Leave empty.)
