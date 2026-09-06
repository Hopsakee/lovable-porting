# Migrate 8 Lovable apps to self-managed Hetzner (hopsakee.top)

## Context

Jelle built 8 apps on Lovable (lovable.dev) — all live on GitHub under `Hopsakee` — and wants to run them on his own Hetzner box instead of depending on Lovable's hosting/cloud-linked Supabase projects. This fits his stated preference for owning infra over Big Tech dependency. The existing `hopsakee-server` repo already has a proven, repeatable deploy pattern (2 live apps: `hopswiki-web` gated, `pkw-web` public) built on Docker Compose + a shared Caddyfile + Authelia forward-auth + file-based secrets on a Hetzner Volume. Nothing resembling a database currently runs on the box, and 5 of the 8 apps depend on Supabase (Postgres+Auth+REST+Storage+Edge-Functions).

Decisions Jelle has already made (do not re-litigate):
1. **Self-host one shared Supabase-equivalent stack** on Hetzner rather than keep the Supabase-backed apps pointed at Lovable-linked cloud projects.
2. **Pilot first**: 1 static app + 1 Supabase-backed app, prove the whole pattern end-to-end, before touching the rest.
3. **`jonkies-tody` goes behind Authelia** like every other gated app (family members get Authelia accounts) — not a separate in-app-only auth scheme.
4. **Real live data must survive** for `jonkies-tody` and `tweedelezer-hulpje` — actual Postgres data export/import, not just schema recreation.

Research already done (2 Explore agents against live repos + 1 Plan agent that directly cloned all 8 apps and the `hopsakee-server` repo) established the app inventory, corrected several assumptions by direct inspection, and produced a concrete design — reproduced/adapted below with direct spot-verification of the two highest-stakes claims (Caddy/Authelia config shape; `hopsakee-dashboard`'s real Supabase Storage usage).

## App inventory

**Tier A — static, zero backend, browser-local storage only:** `ren-afstand`, `findjd`, `skillkeep`.

**Tier B — each has its own live cloud Supabase project:** `hopsakee-prompts` (1 migration, 1 edge fn, simplest), `prompt-keeper-sqlite` (Supabase client wired but **no code path uses it** — sql.js + GitHub-PAT sync is the real storage; likely dead/orphaned, confirm live row counts before deciding whether it's actually Tier A), `jonkies-tody` (26 migrations, family points app, real live data, Google OAuth via a thin `@lovable.dev/cloud-auth-js` wrapper over Supabase Auth), `tweedelezer-hulpje` (11 migrations, 2 edge fns, OpenAI-backed document analysis, real live data, hardcoded Supabase creds in `client.ts` — needs fixing regardless), `hopsakee-dashboard` (architectural outlier: TanStack Start **SSR app** on Cloudflare Workers, not a static SPA; uses Supabase Storage for AI-generated cover images; 3 external APIs — Lovable AI Gateway, GitHub, GitLab; secrets committed to git).

## Self-hosted Supabase stack

Stand up the **official `supabase/docker` self-hosting stack, trimmed**, as `hopsakee-server/config/supabase/compose.yaml` + `server_setup/deploy-supabase.sh`, added to `server-deploy.sh`'s "must succeed" block (every Supabase-backed app now hard-depends on it).

- **Include**: `db` (Postgres), `auth` (GoTrue — hard requirement, every app's RLS keys on `auth.uid()`), `rest` (PostgREST — every app talks to Postgres only through `supabase-js.from()`), `storage` (local-filesystem backend — `hopsakee-dashboard` genuinely uses it for cover images, verified directly), `functions` (Deno edge-runtime — both existing edge functions are idiomatic Deno/npm-specifier style, should run largely unmodified), `kong` (API gateway, path-routes `/auth/v1` `/rest/v1` `/storage/v1` `/functions/v1`), `studio`.
- **Omit**: Realtime (confirmed unused everywhere), Analytics/Logflare/Vector (heaviest, most fragile part of the stock compose, no app needs it), imgproxy (no image-transform usage found).
- **Two subdomains**: `supabase-api.hopsakee.top` → Kong, **public, no Authelia gate** (annotated like `pkw-web`'s F13 precedent — access control is RLS+JWT, not network gating; this matches how Supabase's own cloud SaaS already works and is not a security downgrade). `supabase-studio.hopsakee.top` → gated hard (`policy: one_factor`, `subject: [['group:admins']]`) — or start with an SSH tunnel instead of a public subdomain at all, since Studio access is mostly Jelle debugging during migration; add the gated subdomain later only if tunneling proves annoying.
- **Frontend env vars** (`VITE_SUPABASE_URL`) must point at the **public HTTPS subdomain** (baked in at Vite build time, called from the browser) — internal Docker network names only work for genuinely server-side calls (edge functions calling back into Supabase).
- **Open item to verify before committing resources**: server is Hetzner `cx23` (`config/settings.yaml`, confirmed) — check actual vCPU/RAM against current Hetzner specs before assuming headroom for ~7 more containers (db/auth/rest/storage/functions/kong/studio) alongside the 8 app containers + Caddy + Authelia. May need a resize around/after the pilot.

## Pilot phase

**Tier A pilot: `findjd`** (not `ren-afstand` — too simple to exercise the PWA/service-worker serving quirk that `skillkeep` and `prompt-keeper-sqlite` will also need; not `skillkeep` — smaller surface, same PWA pattern, lower risk for a first attempt). `ren-afstand` becomes the easy first app of the Tier-A rollout instead, precisely because it has no PWA quirks.

**Tier B pilot: `hopsakee-prompts`** (simplest real Supabase app — 1 migration, 1 edge fn, 1 external secret, real RLS+Auth usage but no OAuth-provider complexity to untangle).

### New files per app repo (this shape repeats for every app — described once)
- `Dockerfile` — multi-stage: Node build stage (`npm ci && npm run build`), then Caddy static-serve final stage (not nginx — one HTTP server implementation to operate across the whole box). Non-root pattern from `hopsakee-server/IMPROVEMENTS.md` §1 applied explicitly (fixed UID 1001, real home dir, `chown`, nologin shell).
- `compose.yaml` — dev/reference copy, healthcheck per `IMPROVEMENTS.md` §4.
- `caddy-snippet.txt` — exact block to paste into the shared Caddyfile. Public apps omit `import authelia_forward_auth`/`import logout` (matches `pkw-web`'s `datalab-knowledge.hopsakee.top` F13 block, verified live). Gated apps include both (matches `hopswiki.hopsakee.top`/`infoflow.hopsakee.top`/`netflix.hopsakee.top`, verified live).
- `deploy.sh`, `CLAUDE.md` documenting the deploy contract.

### New files in `hopsakee-server`
- `config/findjd/compose.yaml`, `config/hopsakee-prompts/compose.yaml` — same shape as the existing `config/pkw-web/compose.yaml`.
- `server_setup/deploy-findjd.sh`, `server_setup/deploy-hopsakee-prompts.sh` — clone/`git reset --hard` → `cd` config dir → `docker compose up --build -d` → wait-for-healthy → `reload_caddy`. One new line each in `server-deploy.sh`'s "can fail independently" block.
- Extract the healthcheck wait-loop (`IMPROVEMENTS.md` §4b) out of copy-paste into a shared `wait_for_healthy()` function in `server_setup/utils.sh` — do this once now, not 8 times.
- Caddyfile: append `findjd.hopsakee.top` (public) and `hopsakee-prompts.hopsakee.top` (gated) blocks.
- Authelia: add one `access_control.rules` entry for `hopsakee-prompts.hopsakee.top` (`subject: [['group:me','group:admins']]`, matching the existing `infoflow`/`hopswiki` entries) — this forces a deliberate, isolated Authelia `--force-recreate` (no live config reload exists; don't bundle with an unrelated app deploy).
- `hopsakee-prompts`'s `github-sync` edge function: its CORS allowlist currently only allows `lovableproject.com`/`lovable.app`/`lovable.dev`/`localhost` — needs a code change to add `https://hopsakee-prompts.hopsakee.top`, or its own frontend's calls to it will be CORS-blocked.
- Secrets to move off Lovable/cloud dashboards into the Hetzner secrets-volume (`*_FILE` pattern where supported): new self-hosted Supabase project's `ANON_KEY`/`SERVICE_ROLE_KEY`/`JWT_SECRET`/`POSTGRES_PASSWORD` (freshly generated, not migrated from the old cloud project — meaningless once Postgres is a different instance), plus a new minimally-scoped `GITHUB_TOKEN` for the edge function.
- **Vite build-time env vars matter**: `VITE_SUPABASE_URL`/`VITE_SUPABASE_PUBLISHABLE_KEY` must be passed as compose `build.args`, not `environment:` — a static Vite app has no runtime env to read, get this wrong and the pilot silently ships pointing at the old cloud project.

### Data migration procedure (design now, needed for real later)
```bash
# extract — public schema only, auth/storage/realtime schemas are regenerated fresh by the new stack
pg_dump "postgresql://postgres:<db-password>@db.<project-ref>.supabase.co:5432/postgres" \
  --no-owner --no-privileges --schema=public -f <app>-export.sql

# replay app migrations against the empty new DB first (surfaces schema bugs separately from data bugs)
# each supabase/migrations/*.sql in filename order, then:
psql "postgresql://postgres:<new-local-password>@localhost:5432/postgres" -f <app>-export.sql
```
`auth.users` needs special handling for apps with FKs into it (`jonkies-tody`'s `profiles`): pre-provision matching UUIDs directly into the new GoTrue's `auth.users` rather than letting new sign-ins issue fresh UUIDs (preserves every FK/points-ledger history).

**Cutover model for `jonkies-tody`/`tweedelezer-hulpje`**: single export+import+DNS-cutover window (announce downtime, dump, restore, deploy pointed at new stack + Authelia gate + family accounts already provisioned, smoke-test, switch, keep old version up read-only a few days as fallback) — not a maintenance-mode/dual-write build, since usage is low-traffic/family-only and the engineering cost of dual-write isn't justified here.

## Roadmap after the pilot

**Phase 2 — remaining static apps**: `ren-afstand` (simplest, no PWA quirks), then `skillkeep` (same PWA pattern as `findjd`, plus verify `.wasm` gets `Content-Type: application/wasm` — the one asset type `findjd` didn't exercise).

**Phase 3 — remaining Supabase apps, ascending risk:**
1. `prompt-keeper-sqlite` — **decision, not migration**: check live row counts on its cloud project first. If empty (likely, since no code path reads it): delete the Supabase layer entirely, ship as Tier A. If real data exists, restore it into the shared Postgres for archival only (app still runs on sql.js at runtime).
2. `tweedelezer-hulpje` — data cutover per above, plus: fix hardcoded `client.ts` creds to read from `VITE_SUPABASE_URL`, fix edge-function CORS for the new domain, verify `npm:pdf-parse`/`npm:mammoth` actually run under self-hosted edge-runtime (the one real technical risk here — test early, it gates the rest of this app's migration), and decide OpenAI vs Claude for the `analyze-document` call (real Messages-API rewrite either way, not an env swap — validate Dutch-language output quality before fully cutting over).
3. `jonkies-tody` — data cutover per above (highest stakes, real family data), plus: remove `@lovable.dev/cloud-auth-js`, call `supabase.auth.signInWithOAuth('google')` directly against self-hosted GoTrue, register a real Google Cloud OAuth client with redirect URI at `supabase-api.hopsakee.top/auth/v1/callback`, provision Authelia accounts for each family member. **Confirm with Jelle before building**: Authelia-gate-then-still-see-the-app's-own-Google-login is two stacked logins, not one unified sign-in — if he wants a single prompt, that's Authelia-as-OIDC-provider or a custom shim, meaningfully more work than the default reading of "gate it like the others."
4. `hopsakee-dashboard` — its own mini-project: replace Cloudflare Workers with a plain Node container running the SSR server directly (needs its own Dockerfile, not the static-Caddy pattern); unwrap `@lovable.dev/vite-tanstack-config` into vanilla TanStack/Vite/Tailwind plugins **before** attempting a container build (unverified whether it even resolves outside Lovable's platform — check first); replace the classification half of the Lovable AI Gateway with direct Claude tool-use calls, and make an explicit separate decision on the image-generation half (Claude doesn't generate images — dropping it or calling Gemini/another API directly are the real options, don't let it silently break); scrub the currently-git-committed `.env`/secrets per Jelle's standing secrets rule, move `GITHUB_TOKEN`/`GITLAB_TOKEN` into the Hetzner secrets-volume pattern.

## Cross-cutting infra (build once)

- **Shared Node static-serve base image**: `hopsakee-server/base/node-static.Dockerfile` (non-root pattern + pinned Caddy + correct wasm/webmanifest/no-cache-sw.js defaults, proven in the pilot), built locally on the server (`docker build -t node-static-base:20 base/`) — **no GHCR registry**, matching the existing zero-registry precedent of `pkw-web`/`hopswiki-web` rather than adding new credential-management surface. Collapses each Tier-A-shaped app's own Dockerfile to `FROM node-static-base:20` + build + copy dist. Covers `findjd`, `skillkeep`, `ren-afstand`, `hopsakee-prompts`, and `prompt-keeper-sqlite` if it ends up Tier A. `hopsakee-dashboard` doesn't fit (needs a runtime, not static serving) — gets its own Dockerfile.
- **`wait_for_healthy()` in `utils.sh`** — one shared function instead of the healthcheck wait-loop pasted into 8 separate `deploy-<app>.sh` scripts.
- **Subdomain convention**: default `<repo-name>.hopsakee.top` for all 8. Quick confirm worth getting from Jelle before Phase 3: does he want a friendlier public-facing alias for `jonkies-tody` (family URL) — everything else is fine as the raw repo name.

## Verification

- **Pilot success criteria** (both must hold before starting Phase 2/3): `findjd.hopsakee.top` serves the built app correctly including PWA install/offline behavior (no stale-`index.html`/service-worker caching bug); `hopsakee-prompts.hopsakee.top` loads behind Authelia, sign-in/RLS-scoped reads+writes work against the new self-hosted Postgres, and the `github-sync` edge function completes a real sync end-to-end (proves Kong routing + GoTrue + PostgREST + Edge Functions + secrets-volume all work together, not just in isolation).
- Verify each with **Interceptor** (mandatory per standing rule) — open the live gated and public URLs, confirm no console errors/network 404s, confirm Authelia redirect-then-back-to-app flow actually works for the gated one.
- Before Phase 3 item 3 (`jonkies-tody`) and item 2 (`tweedelezer-hulpje`) cutovers: run the data-migration procedure once against a throwaway copy first (dry run) — diff row counts between old cloud project and new self-hosted instance before pointing the real family app at it.
- Confirm `prompt-keeper-sqlite`'s live Supabase row counts (dashboard check) before deciding Tier A vs Tier B for it — currently a hypothesis (dead code), not yet verified.
- Resolve the `cx23` resource-headroom question before or shortly after the pilot, once the Supabase stack's real memory footprint is observed running.

---

## Besluit 2026-09-04 — SQLite in plaats van self-hosted Supabase

Jelle duwde terug op de Supabase-keuze: ~10 gebruikers, persoonlijke apps, en een stack van zeven containers om te onderhouden. Terecht. De oorspronkelijke reden voor Supabase was **niet** schaal, maar het sparen van app-code: Lovable schreef alles tegen `supabase-js`, met RLS op `auth.uid()` en twee edge functions. Zelf-hosten was de weg om vijf apps te verplaatsen zonder ze te herschrijven.

De weging valt nu anders uit, om drie redenen:

1. Authelia authenticeert al iedereen, dus GoTrue en `auth.uid()` zijn grotendeels dubbelop.
2. `prompt-keeper-sqlite` draait al op sql.js en gebruikt Supabase mogelijk helemaal niet.
3. Alleen `jonkies-tody` (26 migraties, echte gezinsdata) en `tweedelezer-hulpje` (11) dragen serieus schema.

De Storage-behoefte van `hopsakee-dashboard` is coverafbeeldingen: een filesystem-volume, geen database.

**Nieuwe volgorde:** eerst de drie statische apps. Daarna precies één Supabase-app herschrijven naar SQLite + auth uit Authelia-headers, en dat wordt `hopsakee-prompts` (1 migratie, 1 edge function). Dat meet de echte herschrijfkosten op de goedkoopste app voordat er vijf op het spel staan. Gaat het snel, dan wordt Supabase nooit geïnstalleerd. Wordt het lelijk, dan ligt het Supabase-ontwerp hierboven nog op de plank.

### Open vragen — beantwoorden vóór de eerste app MET database

Jelle heeft deze zelf benoemd als de enige echte hobbel voor SQLite. Ze hoeven niet nu opgelost, wél voordat `hopsakee-prompts` of iets zwaarders live gaat:

- [ ] **Back-up.** Wat is de stabiele back-uproutine voor SQLite op de Hetzner-box? Kandidaat: de online-backup-API of `VACUUM INTO` naar een snapshot, `PRAGMA integrity_check` erop, atomisch hernoemen, retentie in dagen. Nooit een `cp` van een live bestand, en nooit een live `.db` in een sync-map. In `~/.claude/LIFEOS/USER/CONFIG/OPERATIONAL_RULES.md` staat het regime dat hier al voor deze box geldt; die vorm is herbruikbaar. Waar landen de snapshots, en wie bewaakt dat ze niet stilzwijgend stoppen?
- [ ] **Toegang vanuit meerdere applicaties.** Kan één SQLite-bestand door meer dan één app-container gelezen en geschreven worden, en willen we dat überhaupt? Te beantwoorden: één database per app (simpel, geen gedeelde state) versus één gedeelde database achter één schrijvende service. WAL plus `busy_timeout` maakt meerdere lezers en één schrijver werkbaar, maar over een Docker-volume met meerdere containers is dat een aanname die getest moet worden, geen gegeven.
- [ ] Volgt uit de twee hierboven: heeft de pilot een aparte data-service nodig, of praat elke app direct met zijn eigen bestand?

Zolang deze drie openstaan, geldt: geen app met echte data live zetten.