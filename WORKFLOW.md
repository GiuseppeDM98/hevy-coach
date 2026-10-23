# Workflow — standing instructions

> **Standing instructions for any AI agent working on this repo.** They are not suggestions and not
> per-session: they hold until this file says otherwise.
>
> **Why they live in the repo and not in agent memory**: agent memory is per-machine and per
> install, so the same rules drift into different versions on each machine and in a cloud
> session. A tracked file travels with the clone and can be reviewed in a diff. If you are an agent
> with persistent memory, do **not** re-save these rules there — save one pointer to this file. If
> the owner states a new rule, it is added *here*, in that session's commit: to section 3 if it holds
> for this repo only; if it holds for every repo, ask the owner, because it enters the standard
> first (`claude-config/prompts/WORKFLOW_STANDARD.md`) and reaches the repos from there.
>
> Sections 1-2 are the portable standard, identical across every repo that adopts it. Section 3 is
> the only project-specific part: it says what "automate it yourself" and "show me the app"
> concretely mean *here*.

---

## 1. Session and collaboration rules

1. **Never commit without explicit approval.** Do not run `git commit` (nor `--amend`) until the
   owner gives the OK for that specific commit. Finish the work, summarise the diff, then ask.
   Creating the branch and editing files needs no approval — only the commit does.

2. **One branch per session.** Before starting implementation work, create a new branch from the
   branch that is active at the start of the session. Always check which one that is; never assume
   `master`/`main`.

3. **One commit per session.** Everything from a session is squashed into a single commit, never
   scattered across several.

4. **Always answer in Italian** when working on this repo. This applies to the conversational
   channel; code, identifiers and comments stay in English.

5. **Questions and proposals are asked interactively** (2026-09-11). When a decision is the owner's —
   which layers to build, on which surfaces, a wording — put it through the agent's interactive
   question tool, one batch per topic, multi-select where the options are not exclusive and the
   recommended option first; never a numbered list of questions in prose. The owner refines the
   wording through the free-text answer («hai pagato», not «pagherai», for a tax the broker
   withholds at the sale).

6. **Figures in tests are invented and round, never the owner's** (2026-10-12). A unit-test
   fixture, an E2E decoy or a seed carries amounts like 2500, 1120, 50: the owner's real data never
   enters the repo, not even as an expected value. If § 3 provides a copy of real data for the
   guided tour, it serves the tour only and phase G removes it.
   **Nothing of the owner's real data in a tracked file** (2026-09-20, 2026-10-15): a draft, a
   critique, a screenshot, a fixture or an example sentence takes invented figures and generic
   names, never an amount, an account or a family detail copied from the real data. In a public
   repo this has no exception. A private repo may waive it, or narrow it, only by saying so in § 3
   with the owner's decision and its date; public data that is not the owner's (a listed company's
   published accounts) is not the owner's data.

7. **After an audit or an analysis, stop and wait for the owner's go-ahead** (2026-10-19). An audit,
   a review or an analysis ends with the findings; no file changes until the owner has chosen which
   fixes to apply and in what order (through rule 5).

---

## 2. Guided verification (*collaudo guidato*)

When a freshly implemented feature has to be verified by hand, do **not** hand over a checklist and
disappear. The verification is done together, in chat, one phase at a time.

### Five obligations

1. **You prepare the test data.** A throwaway script (untracked by git, deleted when the
   verification ends) that plants **decoy words** — invented terms such as *fenicottero*,
   *ornitorinco*, which appear nowhere else in the data. Not entered by hand by the owner.

2. **One phase per message.** Give the phase, wait for the report, then the next one. Never deliver
   all the phases at once: it breaks their prerequisites.

3. **State the expected outcome before running, not after** — otherwise the reading always bends to
   fit whatever happened.

4. **Do every check you can automate yourself, and leave the owner only what you cannot do.**
   "Together, in chat" does not mean "one dictated click at a time". If the sessions are JWT-based
   or otherwise scriptable, write a throwaway script that opens a **real browser** (e.g. Playwright)
   with an authenticated session — your own if the role allows it, otherwise a throwaway test
   identity created for the occasion — and verify every outcome **against the database or the HTTP
   response, never against the look of the page alone**. Report the results phase by phase, with the
   expected outcome stated first.
   **Any automated end-to-end test that you are able to run, you run.** Never declare a feature
   verified while an automated check that could have covered it was left unrun. What is left to the
   owner is only what is genuinely not automatable: visual and aesthetic judgement, physical
   hardware (a real barcode scanner), or an interactive login that cannot be driven by a script (a
   real OAuth flow with MFA).

5. **Before dismantling, let the owner look.** When the session touched something visible, ask for
   the OK and then walk the owner onto the dev server **with the fixtures still alive**: exact URLs,
   under which identity, and at most **five** things to look at — for each one, what must happen and
   what would be the bug. **The tour starts at the owner's go, not when the work is ready**
   (2026-10-05): finish everything that needs no eyes, say it is ready, and wait. Where obligation 4
   allows it, **open the page for the owner, already authenticated** — but one item per message and
   one page at a time, waiting for the outcome before opening the next; two pages together only
   when the item is a comparison (the same URL under two identities), saying which window is which.
   The windows of a driven browser have no name: five at once turn the tour into working out which
   belongs to which item. Only what a probe cannot say: layout, whether the screen says what it must
   say, the words, whether an action gives feedback that it happened. State also **what that tour
   does not cover**, and never ask the owner to redo by hand something already verified. Whatever
   the tour finds becomes an assertion before the session ends, or it will come back: the tour
   exists to discover what nobody thought to assert, not to replace the tests.

### Standard phases, when they make sense

| | Phase | What it establishes |
| --- | --- | --- |
| **A** | Invarianza | What worked before still works |
| **B** | Cambio di contesto | The new role/state is genuinely active |
| **C** | Comportamento nuovo | It does what it must, and not what it must not — obligation 4 matters most here: automate |
| **D** | Sotto la UI | The same rules hold when the route is called directly |
| **E** | Casi negativi | Someone without rights is refused, with the right error |
| **F** | Giro guidato | The only phase the owner executes: they look with their own eyes, fixtures still alive |
| **G** | Ripristino | Configuration restored, fixtures removed, script deleted |

### A negative test alone never proves a security guard

It always takes the pair: **own resource** (positive control — must succeed) and **someone else's
resource** (the test — must fail), with the same identical file or record.

### Closing a verification

Restore any configuration that was changed, remove fixtures and test attachments, delete the script,
and **record the outcome somewhere that survives the session** (`CLAUDE.md` or equivalent).
*A verification that was not recorded counts as not done.*

**The outcome is one line, the signals are not** (2026-10-19). In `CLAUDE.md` the session's status
entry says what was done, where it is documented, what is still open and what comes next; it
replaces the previous one. The session's signals — type-check, test counts, E2E runs, flakes,
skipped phases, the guided tour's walk and its findings (2026-10-23) — go in the commit message
and, where they teach something, in the area's guide, never in `CLAUDE.md`: that file is injected
into every turn, and a status entry that collects them grows back into a diary after every trim.
Of the tour, the status entry keeps only its outcome: done and closed, or what it left open.

---

## 3. What this means in THIS repo

The rules above are the standard. This section is the local translation of obligations 4 and 5 — it
changes from repo to repo and is the only part to rewrite when the tooling changes.

> **State on 2026-10-23**: the repo holds documentation, the v1 spec series (`doc/v1/`) and the
> sanitised Hevy fixtures. There is no code yet: every command below is the one the specs install,
> and each spec's § 9 updates this section the day the command becomes real. Until spec 01 is in
> service, "the gates" means the four commands of the table once they exist, and nothing else.

### The commands that exist (installed by spec 01; E2E by 05a; the PWA project by 07)

| Purpose | Command | Since |
| --- | --- | --- |
| Types | `pnpm typecheck` (`tsc --noEmit`) | 01 |
| Lint + format | `pnpm lint` (`biome check .`; `pnpm lint:fix` to write) | 01 |
| Unit tests | `pnpm test` (Vitest, `TZ=Europe/Rome` set by `vitest.config.ts`) | 01 |
| Build | `pnpm build` (`next build --webpack`, kept for the service worker) | 01 |
| Browser tests | `pnpm test:e2e` (Playwright on :3100 with fake Hevy/Anthropic, seeded `e2e.db`); `pnpm test:e2e:pwa` on a production build on :3200 | 05a / 07 |
| Hevy CLI | `pnpm hevy:whoami`, `pnpm hevy:upload [-- --dry-run]`, `pnpm hevy:sync [-- --full]` | 01 / 02 / 03 |
| Review CLI | `pnpm review:run`, `pnpm review:show`, `pnpm review:apply -- <id>\|--all` | 04 |
| Database | `pnpm db:generate`, `pnpm db:migrate`, `pnpm db:studio` | 03 |

Gates before a commit, in this order: `pnpm typecheck && pnpm lint && pnpm test && pnpm build`,
then `pnpm test:e2e` once it exists. A red one stops the session. pnpm is the package manager
(`packageManager: pnpm@11`, `corepack enable`); Node ≥ 22.

### The isolated environment

- **Database**: `TURSO_DATABASE_URL="file:local.db"` in `.env.local` is the development database;
  tests use `:memory:` with the real migrations (`createMemoryDb()`); Playwright uses `file:e2e.db`
  rebuilt by `pnpm e2e:seed`. Production Turso is reached only with the env of `.env.turso-migrate`
  (gitignored) and only for migrations and the initial sync (spec 06).
- **External services**: unit tests never call Hevy or Anthropic (fixtures in `tests/fixtures/hevy/`,
  fake fetch, fake model). Playwright runs with `E2E_FAKE_SERVICES=1` (in-memory fakes behind
  `src/lib/container.ts`). A real call happens only in a guided verification, after the owner's OK,
  always declared with its effect (read / write on Hevy) and its cost (Anthropic).
- **Hevy writes are permanent**: a custom exercise cannot be edited or deleted and a routine
  cannot be deleted through the API. A write to Hevy is announced before it runs, with a dry run
  shown first where the command has one.

### Obligation 5 — how to hand the owner an app already authenticated

- From spec 05a: `pnpm dev` on :3000 with `.env.local`; the session is a signed cookie, so the
  throwaway Playwright script logs in with `APP_PASSWORD` and reuses `playwright/.auth/owner.json`
  (gitignored) to open the page already authenticated. There is one identity (the owner): no
  roles, no second account.
- Before 05a there is nothing to look at in a browser: the tour is the Hevy app on the owner's
  phone (02) or `pnpm db:studio` (03). Each spec's § 7 says which.
- Phones (Android install, iPhone install, Web Push) are the only part the owner must do by hand
  (specs 06 and 07): a probe cannot press "Add to Home Screen".

### Inspecting real data state

`pnpm db:studio` on `local.db`; `sqlite3`-style queries through `drizzle-kit studio` or a throwaway
`tsx` script against `src/lib/db/client.ts`; in production `turso db shell hevy-coach` or the Turso
dashboard (spec 06). Never judge an outcome from the page alone when a row can be read.

### Branches

`main` is the default and the integration branch **and production** (the merge of a PR deploys,
spec 06). One branch per spec (`feature/NN-slug`, named in the spec header), one squashed commit,
one PR to `main`. No `develop`, no preview deployments (single user: verification runs locally).

### Where things are recorded

- **SESSION_NOTES.md**: untracked scratch file in the root, created at the start of every session
  and deleted at its end. Before the closing prompt it ends with one block per thing learned or
  decided: *Cosa* (what was done) · *Perché* (the decision behind it) · *Nota* (traps and
  measures, each with its date) · *Dove va a fine sessione* (AGENTS.md if it holds for the whole
  repo · `doc/guide/<tema>.md` if it is a domain lesson · WORKFLOW.md if it is a session rule ·
  CLAUDE.md if it is project status · a comment on the right line if it is the why of a line).
  Every item is written to its destination before the file is deleted.
- **The outcome of a verification** goes in `CLAUDE.md § Current Status → Latest`, one line, and
  replaces the previous entry; the session's signals (test counts, E2E runs, skipped phases) go in
  the commit message. During the v1 series the same status is written in the spec's `Stato` line
  and in `doc/v1/00-INDICE.md § 5`.
- **`Draft Release Temp.md`** (this is a public repo: the session-end prompt 03 maintains it): a
  GitHub Release draft that accumulates until the owner tags. Model: title `# Draft release`, a
  quote line with the date range, then groups per area of the app in navigation order (Oggi,
  Progressi, Revisioni, Impostazioni, Hevy routines, CLI, Deploy), each a bullet list of
  `- Added/Fixed/Improved …`, two sentences at most, user-facing English, no file names, no owner
  data, never "spec retired". Ceiling 60,000 characters.
- **Nothing of the owner's data in a tracked file, commit message or PR** (rule 1.6, no exception:
  public repo). Fixtures are cleaned as `tests/fixtures/hevy/README.md` says; routine and workout
  ids of the owner stay out even in examples.
