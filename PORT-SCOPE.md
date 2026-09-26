# Port scope — what we are porting, in what order, and what was decided

**This file is the standing answer.** `MIGRATION-PLAN.md` is the original design
(2026-09, written for a different, smaller scope) and keeps its dated status
sections as history. When the two disagree about *which apps* or *in what
order*, this file wins. Update it here rather than re-deciding in chat.

Last updated **2026-09-26**, from Jelle's own list.

## The list — 14 to port

Three are live on the box. Eleven remain. The Lovable project id is the stable
identifier; see the open item at the bottom about GitHub repositories.

| # | App | Lovable project | Status |
|---|---|---|---|
| 1 | [My Project Hub](https://lovable.dev/projects/b05e5e82-827d-45e6-b814-5d7282c8465f) | `b05e5e82` | **next — Jelle's call, goes first** |
| 2 | [Minecraft Mob Maker](https://lovable.dev/projects/f56c2960-fe04-493a-9dcc-ec14f26ae474) | `f56c2960` | to port |
| 3 | [Game Key Hub](https://lovable.dev/projects/3aa4635e-2afb-4464-82e3-f453cbd29965) | `3aa4635e` | to port |
| 4 | [Learn systems and architecture](https://lovable.dev/projects/dbd40690-40b2-452c-bc9c-5a21b3b4feb8) | `dbd40690` | to port |
| 5 | [Carlijn's stappenmaker](https://lovable.dev/projects/90c4b9b6-b8f2-4ede-8ceb-fda9fe5e43cc) | `90c4b9b6` | to port |
| 6 | [Style Shopper Pro](https://lovable.dev/projects/a2d1abf2-204b-4c9a-b025-9b2109462977) | `a2d1abf2` | to port |
| 7 | [Alinea Advies](https://lovable.dev/projects/d0b25f6f-4722-4b6a-aa96-e4789ee9fdc5) | `d0b25f6f` | to port |
| 8 | [Lan Party Planner](https://lovable.dev/projects/4bcead12-5083-49a2-b81d-ed6cd693ef89) | `4bcead12` | to port |
| 9 | [Idee-blaffer](https://lovable.dev/projects/404c386e-0971-46c2-90d1-1de5a1ca909a) | `404c386e` | to port |
| 10 | [Tweedelezer hulpje](https://lovable.dev/projects/c6598aeb-b483-4b4d-aeae-ee414b230f1b) | `c6598aeb` | to port — **on Google**, see below |
| 11 | [Scene Weaver](https://lovable.dev/projects/9db3b898-946a-4e5c-8c8d-6f41bfc3ce30) | `9db3b898` | to port |
| 12 | [Johnny Finder](https://lovable.dev/projects/8d891421-3d6b-46c2-ac8f-01999988060b) | `8d891421` | **done** → `hopsakee-decimal-finder`, live |
| 13 | [Run Distance Planner](https://lovable.dev/projects/839ee74c-f86c-4289-a3d4-220c80533090) | `839ee74c` | **done** → `ren-afstand`, live |
| 14 | [Point Pals](https://lovable.dev/projects/b6321a21-2507-4893-9056-29fbe707d3a0) | `b6321a21` | **done** → `jonkies-tody`, live with real data |

Rows 1–11 are in no particular order beyond row 1. Only My Project Hub's
position is a decision; the rest is still open and should be sequenced by
what each app actually needs, not by this table's order.

## Postponed — do not schedule these

| App | Lovable project | Why |
|---|---|---|
| [Prompt Keeper SQLite](https://lovable.dev/projects/37a9cd7f-ec43-4cda-ac65-9ca8fee42539) | `37a9cd7f` | Jelle is researching better ways to store prompts |
| [Skill keep](https://lovable.dev/projects/93fce4e7-f591-4527-b23e-7c8618051014) | `93fce4e7` | same, for skills |

Both are blocked on a **research decision Jelle wants to make first**, not on
technical work. Neither is a cheap Tier-A win to slot in when the queue looks
thin — porting either now risks porting a thing he is about to replace. Ask
before proposing them again.

## Never — out of scope permanently

| App | Why |
|---|---|
| `hopsakee-prompts` / "Prompt Keeper Supabase" (`2bfe2c76`) | **Obsolete.** Prompt Keeper SQLite is the later version of the same idea. It is not a port candidate at any priority, and `MIGRATION-PLAN.md` naming it the Tier-B pilot is superseded. |

## Per-app decisions

**My Project Hub goes first (2026-09-26).** This reverses the original
ascending-risk order, which put it last as the architectural outlier: a
TanStack Start SSR app on Cloudflare Workers, not a static SPA, using Supabase
Storage for cover images and talking to three external APIs. Jelle's call, made
knowing that. Consequences worth expecting rather than rediscovering: it needs
a runtime container rather than the static-serve base image, so it is the first
app that does not fit the pattern the three finished ports share; and Path C
has no Supabase Storage, so its cover images need somewhere to live.

**Tweedelezer hulpje runs on Google (2026-09-26).** Jelle has a Google account,
so the document-analysis call goes there. Not OpenAI, and not the Azure AI
Foundry variant. Note there are **two** Lovable projects with this name — the
one in scope is `c6598aeb`; `tweedelezer-hulpje-azurefoundry` (`3b081154`) is
a different project and is not in the list. This is a real rewrite of the model
call, not an env swap, and the Dutch-language output quality needs checking
before cutting over.

**Prompt storage and skill storage are Jelle's research, not ours.** See
Postponed above.

## Starting a port

`prompts/START-A-PORT.md` is the starting prompt: fill four placeholders and
paste it into a fresh session. It carries the constraints and the gates, which
are the first things to erode when each session writes its own opening.

## What is known about the remaining apps

Checked 2026-09-26 through the Lovable connector — which was the wrong first
instrument: the source of truth for a port is the app's **GitHub repository**
(its migrations, its `supabase/functions/`, its client code), and a live count
confirms what the migrations claim. Use the connector only to settle a question
the repository cannot answer. The numbers below are kept because they are real,
not because that was the right way to get them.

**Every remaining app has a database enabled**, so there is no static Tier-A
step left in the queue — the two apps that might have been (Prompt Keeper SQLite, Skill keep) are the two
that are postponed. The next port is a second Path C app whether or not it is
meant to be a gentle one.

| App | Live schema | Notes |
|---|---|---|
| Lan Party Planner | 6 tables, 21 policies, `profiles` + `user_roles`, ~370 rows | same shape as `jonkies-tody`, smaller, and the data is not precious |
| Idee-blaffer | 10 tables, ~20 policies, its own access-code auth | `access_codes`, `organizations`, `departments`, `user_sessions` |
| Carlijn's stappenmaker | database enabled, contents not checked | |
| Style Shopper Pro | database enabled, contents not checked | |
| Alinea Advies | database enabled, contents not checked | |
| Scene Weaver | database enabled, contents not checked | |
| Minecraft Mob Maker | not checked | AI generation, expect an edge function and a model key |
| Game Key Hub | not checked | custom images — expect file storage |
| Learn systems and architecture | not checked | markdown in/out, drawings |
| Tweedelezer hulpje | per `MIGRATION-PLAN.md`: 11 migrations, 2 edge fns | plus the Google rewrite |
| My Project Hub | per `MIGRATION-PLAN.md`: SSR, Storage, 3 external APIs | the outlier |

"Database enabled" is not "database used" — Lovable provisions one per project
whether or not a code path touches it. Confirm with a live count at the start
of each port rather than trusting this table.

## Carry-over constraints that still apply

- **The hard gate**: no app with real data goes live before a snapshot routine
  with a *tested restore* exists for it. Closed for Postgres-backed apps, on
  the strength of a real restore — see `MIGRATION-PLAN.md` § "Stand 2026-09-19".
- **A Tier-B port is not finished until its backups leave the box.** Still open
  for `jonkies-tody`: the Mac-side pull is written and pointed at
  `~/Drive-bup/jonkies-tody/`, but the firewall/key/scheduling work on the Mac
  remains, so the box is still a single point of failure for the family's data.
- **Everything in `PORTING-PLAYBOOK.md` marked VERIFIED** was earned on a real
  port. Read it before starting the next one; it is where the next app's
  avoidable rounds are already written down.

## The GitHub repositories — this is what you port from

**Port from the GitHub repository, not from Lovable.** Every app is synced to
Jelle's GitHub. The Lovable project link identifies *which* app; the repository
is the source you actually work with, so resolving it is step zero of a port,
before anything else.

**The naming convention** (Jelle, 2026-09-26): the repository name is the
Lovable project name, lower-cased and hyphenated — `Minecraft Mob Maker` →
`minecraft-mob-maker`, `Game Key Hub` → `game-key-hub`. Two things make this
reliable rather than a guess:

- **`hopsakee-dashboard` is the exception** — on Lovable it is `My Project Hub`.
- **Every repository's `README.md` title carries the Lovable name.** That is the
  cross-check that works regardless of what the repository is called, and the
  way to identify one whose name does not follow the rule.

The three already-ported apps are the other exceptions, because they were named
before the convention settled: `Johnny Finder` → `findjd`, `Run Distance
Planner` → `ren-afstand`, `Point Pals` → `jonkies-tody`. So derive the name,
then confirm it against the README title.

### Access, and a mistake worth not repeating

An earlier version of this file said these repositories did not exist under
`Hopsakee`, on the strength of a repository listing that returned 41 and
included none of them. **That was wrong, and the error is the interesting
part**: the listing and the search both show only what the Claude GitHub App
was *granted*, not what exists. Selected-repository access makes a repository
that is merely ungranted look exactly like one that is absent, and an absence
is the easier thing to report. The instrument answered a narrower question than
the one being asked.

So: if a session cannot see one of these repositories, the fix is to add it to
the Claude GitHub App's repository access (or grant all repositories) at
<https://claude.ai/connect-github>, and to start the session with it selected —
a session's repositories are chosen when it starts. Never conclude from a
listing that a repository does not exist.
