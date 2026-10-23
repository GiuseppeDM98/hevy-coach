# CLAUDE.md — hevy-coach

> **Read [WORKFLOW.md](WORKFLOW.md) before starting**: standing session rules (one branch and one
> commit per session, never commit without approval, answer in Italian) and the guided-verification
> protocol. A new rule stated in a session is added there, in that session's commit.

## Project Overview
A personal coach for home bodyweight training, built on top of Hevy (where the workouts are logged)
and the Anthropic API. Single user, self-hosted. The five weekly routines live on Hevy in a dedicated
folder and the app creates and updates them through the Hevy API; every two weeks the app pulls the
workouts, aggregates them per exercise, asks Claude for a review (keep / increase / next variant,
with a short rationale) and notifies the owner; the dashboard shows progress per exercise, the review
history and the open proposals, and «Applica» (always a manual confirmation) writes the change back
to Hevy. Installable PWA (Android and iPhone). Open source, AGPL-3.0. Not affiliated with Hevy.

**This file is the INDEX**: "what it is + where it lives", under 20,000 characters (ceiling 40,000;
it is injected into every turn). Repo-wide conventions live in **AGENTS.md**; per-area rules in
**`doc/guide/<tema>.md`** (born with the specs); the session rules in **WORKFLOW.md**; what the user
sees in **README.md**; the v1 plan in **`doc/v1/`** (index `00-INDICE.md`).

> **Language**: this file, AGENTS.md, WORKFLOW.md and README.md are in English; the specs, the chat
> and the UI are in Italian; code and comments in English. The plan's routine and exercise names stay
> Italian because they are the product's labels («Lun · Upper A», «Rematore sotto il tavolo»).

## Current Status
- **Stack (decided 2026-10-23, nothing installed yet)**: Next.js 16 (App Router, `proxy.ts`), React 19,
  TypeScript 7, pnpm 11, Turso + Drizzle (`@libsql/client`), zod 4, Biome, Vitest, Playwright,
  Serwist (PWA), `web-push`, `@anthropic-ai/sdk` with `claude-opus-5-5`, Recharts, Tailwind 4.
  Vercel Hobby in `dub1`, Turso in `aws-eu-west-1`. Versions pinned in `doc/v1/00-INDICE.md` T-2.
- **Latest (2026-10-23)**: planning session. The v1 series is written: eight specs in `doc/v1/`
  (01 scaffold + Hevy client · 02 plan, variant scales and routine upload · 03 Turso + sync · 04
  aggregation, Claude review, apply from CLI · 05a app: auth, shell, design, Progressi · 05b app:
  reviews and proposals · 06 deploy: Vercel, Turso, cron, webhook · 07 PWA + Web Push), 35 owner
  decisions and 22 planner choices in the index, sanitised Hevy fixtures in `tests/fixtures/hevy/`.
  The owner's Hevy account was read (GET only) on 2026-10-23: 464 templates, 174 workouts, no routine.
  **Next**: the adversarial reading of the whole series (`claude-config/prompts/07_LETTURA_ADVERSARIALE_SPEC.md`),
  then `PROMPT-RITIRO.md` (prompt 08), then spec 01. Still open: none.

## Architecture Snapshot (as specified; nothing exists yet)
- `src/app` (routes, route handlers, Server Actions: thin) → `src/lib/{review,sync,apply,plan}` (use
  cases and pure domain) → `src/lib/db/repositories` (Drizzle only) and `src/lib/hevy` (API client),
  `src/lib/review/model.ts` (the only file importing the Anthropic SDK). Pure modules take `now` as a
  parameter and never read the clock. Services reach the app through `src/lib/container.ts`
  (fakes under `E2E_FAKE_SERVICES=1`); CLI scripts in `scripts/` use the clients directly.
- Data: Hevy is the source of truth for workouts and routines; Turso keeps a synced copy
  (`workouts`, `workout_sets`), the plan state (`slot_state` + history), reviews and proposals, cron
  runs and push subscriptions. Calendar logic in Europe/Rome, instants as epoch ms UTC.

## Key Features (planned; each line becomes real with its spec)
- **Plan and variant scales** (02): the five routines and every exercise's scale of quiet, simple
  variants as data in `src/lib/plan/`; `pnpm hevy:upload` creates folder, custom exercises and
  routines on Hevy, idempotently. → `doc/guide/piano.md`.
- **Sync** (03): all workouts, incremental through `workouts/events`; routine ids and the template
  map recorded. → `doc/guide/sync.md`.
- **Review** (04): 28-day window, per-exercise aggregation, deterministic rule next to Claude's
  proposal, one call to `claude-opus-5-5`, validated JSON, cost recorded. → `doc/guide/revisione.md`.
- **Apply** (04 CLI, 05b UI): GET + full PUT of one routine per proposal; never automatic.
- **App** (05a, 05b): password login, Oggi, Progressi (Recharts), Revisioni, Impostazioni. →
  `doc/guide/app.md`, DESIGN.md, PRODUCT.md.
- **Operations** (06): Saturday cron (`17 6 * * 6` UTC, review when ≥ 13 days since the last),
  Hevy webhook, `cron_runs` as the log. → SETUP.md, `doc/guide/deploy.md`.
- **PWA + push** (07): manifest, Serwist service worker, Web Push (iOS from the Home Screen only).
  → `doc/guide/pwa.md`.

## Testing (as specified)
Vitest for pure modules and repositories (in-memory libSQL with the real migrations), fixtures from
real Hevy payloads, never the real services; Playwright with in-memory fakes (05a); a production-build
Playwright project for the service worker (07). Commands: WORKFLOW.md § 3.

## Data & Integrations
Hevy public API (`api-key` header, Pro account, OpenAPI 0.0.1 read 2026-10-23; limits in AGENTS.md §
Hevy) · Anthropic Messages API (`claude-opus-5-5`, ~0.10 USD per review) · Turso · Vercel Hobby ·
Web Push (VAPID).

## Known Issues (Active)
None: no code yet.

## Not yet present (do not assume it exists)
- Any code, `package.json`, migrations: the repo is specs and fixtures until spec 01.
- DESIGN.md and PRODUCT.md: born in spec 05a with the impeccable skill.
- SETUP.md: born in spec 06. `Draft Release Temp.md`: born at the first session end.
- `doc/guide/*`: each guide is born with its spec (piano 02, sync 03, revisione 04, app 05a/05b,
  deploy 06, pwa 07).
- Hevy write endpoints for workouts and body measurements: not wrapped (the app never logs workouts).
- Rate limit of the Hevy API: undocumented; the client retries once on 429/5xx.
- Going back a variant from a review (only keep / increase / next): by hand on Hevy.
- Harder variants (archer, pseudo planche, free pistol, nordic): deliberately excluded (quiet and
  simple only).
- Deleting a custom exercise or a routine on Hevy: the API has no DELETE; by hand in the app.
- Charts beyond the per-exercise session chart; app badge; declarative Web Push; offline writes.
- Preview deployments (disabled by `vercel.json`), custom domain, login rate limiting beyond the 1 s
  delay, alerting on sync errors without a review.
- Multi-user, logging workouts in the app, the garage-gym plan: out of scope for v1 (D-8).

## Key Files
- `doc/v1/00-INDICE.md` — the series index: theme, decisions D-N, choices T-N, order, shared
  contracts C-N, series rules. `doc/v1/NN-*.md` — one spec per implementation session.
- `tests/fixtures/hevy/` — sanitised real payloads (README there says what is real and what is
  synthetic).
- `COMMENTS.md`, `DEVELOPMENT_GUIDELINES.md` — binding for every line of code.

## Design Context
Not yet: DESIGN.md and PRODUCT.md are written by the impeccable skill in spec 05a and never by hand
after that. Until then: dark theme by default, mobile-first (390 px), Italian UI, "wow" bar.
