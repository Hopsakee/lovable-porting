# Migrate Jelle's Lovable apps to self-managed Hetzner (hopsakee.top)

> **Scope moved on 2026-09-26.** This document was written for 8 apps. The
> actual list is **14 to port, 2 postponed and 1 dropped**, and it lives in
> **[`PORT-SCOPE.md`](PORT-SCOPE.md)** — that file wins wherever this one
> disagrees about which apps or in what order. Everything below is kept as the
> original design and its dated status sections; the *pattern* is still current,
> the *inventory* is not.

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
---

## Stand 2026-09-11 — de beslissing is nu aan de beurt, ongemeten

De route naar het antwoord op "SQLite of self-hosted Supabase" was: eerst
`hopsakee-prompts` (1 migratie, 1 edge function) herschrijven en daar de echte
herschrijfkosten meten. Die stap is overgeslagen. Daarmee komt `jonkies-tody`
als eerste échte app bij de beslissing aan — ongemeten, en met de hoogste
inzet van de hele migratie.

Wat direct uit de broncode geverifieerd is (niet aangenomen): 26 migraties,
waarvan er 25 schoon terugspelen op een lege Postgres 15; netto **8 tabellen,
4 enums, 11 functies (10 `SECURITY DEFINER`), 8 triggers, 31 RLS-policies**;
PostgREST + GoTrue + Storage + 1 RPC in gebruik; **geen edge functions en geen
Realtime**. Dat laatste scheelt: de Supabase-variant is hier vijf containers
(`db`/`auth`/`rest`/`storage`/`kong`), niet de zeven waartegen het Besluit
afwoog.

Beide paden zijn concreet uitgewerkt en afgewogen in
`jonkies-tody/PORT-NOTES.md`, met een aanbeveling (**Path A voor déze app**,
en de SQLite-meting alsnog op `hopsakee-prompts` doen) en het eerlijkste
tegenargument erbij. Er is geen app-code geschreven vooruitlopend op de keuze.

Eén observatie die de aard van de harde grens verandert, en die expliciet
géén sluiproute is: de drie openstaande vragen zijn alle drie *SQLite*-vragen.
Valt de keuze op Postgres, dan worden ze niet beantwoord maar niet-van-
toepassing, en vervangen door één gewone vraag (een `pg_dump`-snapshotroutine
met een getest restore). De grens zelf — geen app met echte data live zolang
er geen bewezen back-uproutine draait — blijft onverkort staan.

Ook nog open, en niet onafhankelijk hiervan: één inlogprompt of twee (zie
`PORT-NOTES.md`). De variant "één prompt via Authelia-headers" bestaat alleen
als de app een eigen backend krijgt die die headers kan lezen — dus alleen
onder Path B.

## Stand 2026-09-11 (2) — de keuze is Path C, en er is een bug gevonden

Jelle duwde terug op beide opties, terecht: SQLite past slecht bij meerdere
gebruikers met rij-niveau toegangsbeperking, en een vijf-container
Supabase-stack is buiten proportie voor ~20 gebruikers als Authelia al
iedereen authenticeert.

Dat maakte een derde weg zichtbaar. Wat beschermd moest worden was nooit
Supabase, maar dát de 31 RLS-policies en 8 triggers **in de database
afgedwongen** blijven in plaats van applicatiecode te worden. Dat pleit voor
Postgres, niet voor Supabase. En `auth.uid()` — waar al die policies op
draaien — is geen Supabase-feature maar drie regels SQL over een
request-scoped setting die elke JWT-validerende gateway vult.

**Path C: kale Postgres + PostgREST + een kleine Authelia→JWT-shim.** Drie
backend-containers in plaats van vijf. GoTrue en Kong vervallen — precies de
onderdelen die dubbelop waren. In de sandbox end-to-end geverifieerd tegen
synthetische data: 26/26 migraties draaien, `auth.uid()` resolvet uit een
zelf-gemunte JWT, anonieme requests krijgen `[]`, en elke
escalatiepoging als deelnemer wordt geblokkeerd precies zoals onder echt
Supabase. `@supabase/supabase-js` blijft werken tegen kale PostgREST, dus alle
`.from()`/`.rpc()`-aanroepen blijven ongewijzigd; alleen auth en storage
moeten vervangen worden (~250 regels). Uitwerking in
`jonkies-tody/PORT-NOTES.md`, reproductie in
`jonkies-tody/docs/pathc-prototype/`.

Daarmee vervalt ook de tweede open vraag: Authelia wordt de
identity-provider, Google-login verdwijnt, dus één inlogprompt en geen
Google-OAuth-client meer nodig.

**En passant een echte bug gevonden die nú speelt**, los van de port:
prijzen die een deelnemer inwisselt worden wél geregistreerd maar **niet
afgeschreven**. `update_user_points()` doet een UPDATE op `profiles`, en die
UPDATE triggert `protect_profile_columns()`, die `total_points` terugdraait
zodra `is_admin(auth.uid())` onwaar is — bij een deelnemer dus altijd. Dat
verklaart de vier "fix balances"-migraties in de historie: die herberekenden
telkens het saldo zonder de oorzaak weg te nemen. Een geteste fix plus een
read-only controlequery staan in
`jonkies-tody/docs/proposed-fix-balance-drift.sql`, bewust níét in
`supabase/migrations/`, zodat mergen van de PR niets aan de echte database
verandert.

De harde grens blijft staan. Alleen de vorm verandert: geen drie
SQLite-vragen meer, maar één gewone `pg_dump`-snapshotroutine met een getest
restore. Of dát de grens sluit, bepaalt Jelle.

## Stand 2026-09-12 — Path C bevestigd en gebouwd

Jelle heeft alle zes de open punten beantwoord: Path C, één inlogprompt
(Authelia als identity provider, Google eruit), een nieuwe `family`-groep,
`jonkies-tody.hopsakee.top`, en `pg_dump` met een getest restore als
invulling van de harde grens.

De stack is gebouwd en end-to-end geverifieerd tegen synthetische data — niets
uitgerold, geen echte data aangeraakt. Drie containers: Postgres, PostgREST en
een eigen Authelia→JWT-shim. De shim is de enige nieuwe security-kritische code
en is ook zo getest, tegen een echte `caddy:2.8`: spoofen van `Remote-User`
lukt niet (`copy_headers` overschrijft wat de client stuurt). Twee bevindingen
veranderden het ontwerp: het "hardenen" door client-headers te strippen vóór
`forward_auth` **breekt de gate** (verkeerde directive-volgorde, alles 401't),
en `copy_headers` zet een header óók als de authorizer hem niet teruggaf — als
letterlijke placeholder-tekst, die in een JWT-claim belandde voordat de shim
erop controleerde.

De echte `@supabase/supabase-js` draait ongewijzigd tegen kale PostgREST via de
echte routing, dus alle `.from()`/`.rpc()`-aanroepen bleven staan. Een volledig
gezinsscenario klopt: goedkeuren, punten toekennen, prijs inwisselen — 100 naar
60, nul drift, met de balансfix erin.

Back-up: script geschreven, écht gedraaid, en de dump teruggezet in een lege
Postgres met gelijke saldi en gelijke rijaantallen over alle acht tabellen.
`jonkies-tody/docs/BACKUP.md` legt uit waarom telefoons van gezinsleden niets
bevatten om te back-uppen (de database staat op Hetzner, niet op het apparaat)
en waarom de NAS trekt in plaats van dat de box duwt.

Wat resteert vóór cutover staat in `jonkies-tody/PORT-NOTES.md`: de drift-query
op het live project draaien, de back-up op de échte box installeren en één keer
terugzetten (dát sluit de grens), Authelia-groep en -regels, de
`hopsakee-server`-kant, en de datamigratie met behoud van UUID's plus de
`authelia_user`-koppeling per gezinslid.

---

## Stand 2026-09-19 — `jonkies-tody` is live met echte data

De eerste Tier-B-app draait op de Hetzner-box en de gezinsdata staat er op.
Niet Path A (de aanbeveling hierboven) en niet SQLite, maar **Path C**: kale
Postgres + PostgREST + een eigen Authelia→JWT-shim. Drie containers in plaats
van vijf, met alle RLS-policies, triggers en de RPC ongewijzigd in de database.
De redenering staat in `jonkies-tody/PORT-NOTES.md`, de uitvoering in
`jonkies-tody/docs/CUTOVER-RUNBOOK.md`.

De migratie zelf is regel-voor-regel geverifieerd: 5 identiteiten, alle negen
teltabellen gelijk aan de bron, elk saldo gelijk, geen drift in het grootboek,
geen weeskinderen.

### De harde grens is gesloten — voor Postgres-apps, en niet helemaal

De `pg_dump`-snapshotroutine bestaat, draait via cron op de box, en is **écht
teruggezet**: een snapshot in een lege Postgres-container gezet en de rijtellingen
en saldi vergeleken. Dat gebeurde met testdata, vóórdat er echte data op de box
stond — precies de volgorde die de grens bedoelt.

Twee dingen die pas bij het echte uitvoeren bleken, en die in
`PORTING-PLAYBOOK.md` staan:

- `RESTORE OK` is de grens. De vergelijking met de *live* database is een
  aparte versheidscontrole, en die is alleen zinnig tegen een net gemaakte
  snapshot. Tegen een oude snapshot midden in een cutover meldt hij
  `MISMATCH` op elke tabel en verklaart hij een perfecte back-up kapot.
- Afbeeldingen en andere bestandsvolumes zitten in géén enkele database-dump.
  Apart back-uppen, en bij de cutover apart verplaatsen.

**Wat nog wél openstaat:** de snapshots verlaten de box niet. Ze bestaan, ze
zijn ingepland en ze zijn herstelbaar, maar zolang er niets ze ophaalt blijft
de box een single point of failure voor de gezinsdata. Het ontwerp ligt vast
(de Mac Mini haalt ze op en zet ze in `~/Drive-bup`, de bron van een
**Backup**-taak die alleen omhoog gaat — nadrukkelijk *geen* Drive-syncmap, want
een lokale verwijdering daarin loopt door naar de NAS; de versiegeschiedenis op
de NAS is de retentie) en het script staat in `hoggle-macmini`;
het firewall-, sleutel- en planningswerk op de Mac moet nog. Beschouw een
Tier-B-port niet als af voordat dit er is.

De drie SQLite-vragen hierboven zijn hiermee niet beantwoord maar
niet-van-toepassing voor déze app. Ze blijven onverkort staan voor elke app
die alsnog op SQLite uitkomt.

### Correctie op de cijfers hierboven

Netto **28 RLS-policies in `public`** (plus 4 op de `storage.objects`-stub),
niet de 31 die hierboven staan. Die 31 kwam uit een sandbox-telling; de
draaiende database is leidend:
`select count(*) from pg_policies where schemaname='public'`.
De overige cijfers — 8 tabellen, 4 enums, 11 functies, 8 triggers — kloppen.

---

## Stand 2026-09-26 — de scope is groter en de volgorde is anders

Jelle heeft de lijst zelf vastgesteld. Hij staat, met de projectlinks, in
**[`PORT-SCOPE.md`](PORT-SCOPE.md)**; dat bestand is voortaan leidend voor
*welke* apps en *in welke volgorde*. Hier alleen wat er verandert ten opzichte
van het ontwerp hierboven.

**Veertien apps, geen acht.** Drie daarvan draaien al (Johnny Finder,
Run Distance Planner, Point Pals). Elf moeten nog. De app-inventaris hierboven
dekte maar een deel van wat hij wil verplaatsen.

**`hopsakee-prompts` gaat nooit.** Obsoleet: Prompt Keeper SQLite is de nieuwere
versie van hetzelfde idee. Daarmee vervalt ook de keuze hierboven om juist die
app als Tier-B-pilot te nemen — die pilot is inmiddels sowieso ingehaald door
`jonkies-tody`, die als eerste Tier-B-app echt is overgezet.

**Prompt Keeper SQLite en Skill keep zijn uitgesteld**, niet om technische
redenen maar omdat Jelle eerst wil uitzoeken hoe hij prompts en skills
überhaupt wil opslaan. Niet inplannen als goedkope tussendoor-port: de kans
bestaat dat we iets verplaatsen dat hij gaat vervangen.

**My Project Hub gaat als eerste.** Dat draait de volgorde van "oplopend risico"
hierboven om, want die zette hem juist achteraan als de architecturale
buitenbeentje — SSR op Cloudflare Workers in plaats van een statische SPA, met
Supabase Storage en drie externe API's. Jelles keuze, in het volle besef
daarvan. Twee gevolgen die je maar beter kunt verwachten dan ontdekken: hij is
de eerste app die een *runtime*-container nodig heeft in plaats van het
statische base-image, en Path C heeft geen Supabase Storage, dus zijn
cover-afbeeldingen moeten ergens anders heen.

**Tweedelezer hulpje gaat naar Google.** Jelle heeft daar een account. Dus niet
OpenAI, en niet de Azure-AI-Foundry-variant. Let op: er bestaan twee Lovable-
projecten met die naam; `c6598aeb` hoort bij de lijst, `3b081154`
(`tweedelezer-hulpje-azurefoundry`) niet. Het blijft een echte herschrijving van
de modelaanroep, geen env-wissel, en de Nederlandse uitvoerkwaliteit moet
gecontroleerd worden vóór de omschakeling.

**De repositories.** Elke app staat in Jelles GitHub. De repositorynaam is de
Lovable-projectnaam in kleine letters met streepjes (`Minecraft Mob Maker` →
`minecraft-mob-maker`); `hopsakee-dashboard` is de uitzondering (op Lovable
heet die `My Project Hub`), en de titel van elke `README.md` draagt de
Lovable-naam, wat de betrouwbare kruiscontrole is. De drie al geporte apps
dragen nog oudere namen: `findjd`, `ren-afstand`, `jonkies-tody`.

Er stond hier eerst dat die repositories niet bestonden onder `Hopsakee`, op
gezag van een lijst van 41 repositories waarin ze ontbraken. **Dat was fout, en
de fout is het onthouden waard**: zo'n lijst toont alleen wat de Claude GitHub
App *toegekend* heeft gekregen, niet wat er is. Een repository zonder toegang
ziet er precies zo uit als een repository die niet bestaat, en het tweede is de
makkelijkere conclusie. Het instrument beantwoordde een smallere vraag dan de
gestelde. Ontbreekt er een repository in een sessie, dan is dat een
toegangskwestie — toevoegen aan de App op <https://claude.ai/connect-github> en
de sessie ermee starten — nooit een bewijs dat hij niet bestaat.

En bijbehorend: port vanuit de GitHub-repository, niet vanuit de
Lovable-connector. De migraties, `supabase/functions/` en clientcode in die
repository zijn de bron; de Lovable-link zegt alleen wélke app het is.
