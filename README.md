# hevy-coach

A personal coach for home bodyweight training, built on top of [Hevy](https://hevy.com) and the
Anthropic API. Single user, self-hosted, installable on your phone as a PWA.

> **Status (2026-10-23)**: planning done, no code yet. The v1 is specified in [`doc/v1/`](doc/v1/)
> and is being implemented one spec at a time. Not affiliated with, endorsed by or connected to
> Hevy; it uses Hevy's public API, which requires a Hevy Pro account.

## What it does

- **Your routines live on Hevy.** The five weekly routines (Mon–Fri, 30–40 minutes, quiet and
  simple bodyweight exercises only) are data in the app; one command creates the folder, the custom
  exercises and the routines on Hevy, idempotently. You log your workouts in Hevy as usual.
- **Every two weeks, a review.** A Saturday cron pulls your workouts, aggregates them per exercise
  over the last 28 days and asks Claude (`claude-opus-5-5`) for one proposal per exercise: keep,
  increase the target, or move to the next variant of its scale, with a short rationale. A
  deterministic double-progression rule is shown next to Claude's proposal.
- **You decide.** The dashboard shows progress per exercise, the review history and the open
  proposals. «Applica» writes the change back to the routine on Hevy; nothing is ever applied
  automatically.
- **It tells you.** Web Push notifications when a review is ready (on iPhone from the installed
  app), and a Hevy webhook keeps the dashboard fresh as soon as you save a workout.

The plan and the exercise names are in Italian — they are the labels the owner sees on Hevy and
in the app. The documentation for contributors and agents is in English; the specs in `doc/v1/`
are in Italian.

## Stack

Next.js 16 · React 19 · TypeScript · pnpm · Turso (libSQL) + Drizzle · zod · Biome · Vitest ·
Playwright · Serwist · web-push · `@anthropic-ai/sdk` · Recharts · Tailwind 4 · Vercel (Hobby).

## Documentation

| File | Scope |
| --- | --- |
| [`CLAUDE.md`](CLAUDE.md) | Index — what each piece is and where it lives; what does not exist yet |
| [`AGENTS.md`](AGENTS.md) | Repo-wide conventions, the Hevy API facts, testing, gotchas |
| [`WORKFLOW.md`](WORKFLOW.md) | Session rules and the guided-verification protocol |
| [`doc/v1/00-INDICE.md`](doc/v1/00-INDICE.md) | The v1 series: decisions, order, shared contracts, one spec per session |
| [`tests/fixtures/hevy/`](tests/fixtures/hevy/) | Sanitised real Hevy payloads used by the tests |
| [`COMMENTS.md`](COMMENTS.md), [`DEVELOPMENT_GUIDELINES.md`](DEVELOPMENT_GUIDELINES.md) | Binding code and comment guidelines |

`SETUP.md` (from clone to your own deployment on Vercel + Turso) arrives with spec 06.

## License

[AGPL-3.0](LICENSE). If you run a modified version as a service, you must publish its source.
