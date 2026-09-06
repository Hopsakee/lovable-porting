# Porting playbook

Status: v0, nothing proven yet. Every claim below is either VERIFIED (a real port
confirmed it) or ASSUMED (it came from planning). Never promote an ASSUMED line to
VERIFIED without a run that exercised it.

## Target pattern
- ASSUMED: static apps build with Node, then serve from Caddy in the final stage.
  One HTTP server implementation across the whole box, so not nginx.
- ASSUMED: non-root, fixed UID 1001, real home dir, chowned, nologin shell.
- ASSUMED: each app gets `<repo-name>.hopsakee.top`.

## Known traps
- ASSUMED: Vite reads env at BUILD time. `VITE_*` must be compose `build.args`,
  never `environment:`. Get this wrong and the app silently ships pointing at the
  old cloud project.
- ASSUMED: PWA service workers cause stale-index.html after redeploy.
- ASSUMED: `.wasm` needs `Content-Type: application/wasm`.

## Not yet answered
- SQLite backup routine on the box.
- Whether one SQLite file can safely serve more than one container.
