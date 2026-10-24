# 06 — Messa in esercizio: Vercel, Turso remoto, cron e webhook

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 30 rilievi, 4 decisioni
> (D-38, D-41 e le scelte T-24, T-27; condivise D-37, D-39, T-22, T-23). Pronta per l'implementazione
> dopo la 05b.
> **Branch**: `feature/06-deploy` da `main`. **Dipende da**: 01–05b (l'app intera), con
> `syncWorkouts`, `syncOneWorkout`, `shouldRunScheduledReview`, `runReview`, `sendReviewNotification`,
> `isPublicPath`, `isPasswordCorrect`, `getSyncValue`, `setSyncValue`.
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-2, D-13, D-16, D-23, D-32, D-34, D-36,
> D-37, D-38, D-41, D-56; scelte T-1, T-5, T-11, T-12, T-16, T-19, T-21, T-22, T-23, T-24, T-27; contratti
> C-7 (`cron_runs` già creata dalla 04; qui nasce il repository `cron-runs.ts`), C-8, C-11, C-14,
> C-15, C-16, C-18 (li usa).

## 1. Il problema oggi (prove)

1. L'app gira solo in locale su `file:local.db`. Non esistono il progetto Vercel né il database
   Turso: la loro creazione è un passo di questa spec, con i comandi esatti (tema, BOOTSTRAP 7).
2. Vercel Hobby (doc `https://vercel.com/docs/cron-jobs/usage-and-pricing`, letta il 2026-10-24):
   cron **una volta al giorno al massimo**, UTC, **solo produzione**, best effort (**duplicati
   possibili, nessun retry**), e **il minuto non è rispettato**: `17 6 * * 6` parte in un istante
   qualsiasi fra le 06:00 e le 06:59 UTC (07:00–07:59 a Roma d'inverno, 08:00–08:59 d'estate, D-16);
   GET con `Authorization: Bearer $CRON_SECRET` e UA `vercel-cron/1.0`; function fino a 300 s;
   regione singola; log runtime 1 ora (→ `cron_runs` in tabella); Deployment Protection
   «All Deployments» gratuita ma non documentata come compatibile con il cron → produzione aperta
   con auth applicativa (D-13); i deploy di preview sono spenti (T-19).
3. Turso: nessuna regione Francoforte → gruppo `aws-eu-west-1` (Dublino) e Vercel `dub1` (T-1).
   `vercel env pull` non rilegge i Secret: la verità delle env sta nel dashboard. Il database di
   produzione **nasce da `local.db`** (D-41): stati, storia, revisioni e proposte della 04 e della
   05b restano coerenti con Hevy, che è già stato modificato dagli «Applica» reali.
4. Il webhook di Hevy (D-32): POST `{ "workoutId" }` all'URL con l'header di autorizzazione
   impostato nella pagina Hevy «Webhooks»; risposta **200** entro **5 s**. Il nome dell'header non è
   documentato: la route confronta il valore di `Authorization` (grezzo o `Bearer <segreto>`) e, su
   401, logga i **nomi** degli header ricevuti (mai i valori) per diagnosticare dove Hevy mette il
   segreto.
5. Hevy chiede di non chiamare a xx:00 → cron `17 6 * * 6` (D-16); su Hobby il minuto non è
   garantito, e lo si accetta: la richiesta è una cortesia, non un contratto.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `GET /api/cron/review`: `isCronAuthorized(header, secret)` falso (header assente o diverso, o segreto non configurato) → 401; altrimenti `runCron(deps)`: inserisce `cron_runs` (`started`), controlla `plan_uploaded_at` (assente → `error` «setup: run pnpm hevy:sync:remote first», 500), `syncWorkouts` incrementale, `shouldRunScheduledReview` → se vero `runReview({ trigger: 'cron' })` + `sendReviewNotification`, altrimenti `review_not_due`; chiude `cron_runs` con `outcome`, `details_json` e `finished_at`; un'eccezione → `outcome 'error'`, dettagli con il messaggio, risposta 500. `maxDuration = 300`. | `run-cron.test.ts` con DB in memoria, fake Hevy e fake model (§ 6). Mutazione: non scrivere `cron_runs` in caso di errore → rosso; togliere il controllo del segreto vuoto → rosso. |
| A2 | `POST /api/hevy/webhook`: `isWebhookAuthorized` falso → 401 (con `console.warn` dei nomi degli header); body senza `workoutId` stringa → 400; altrimenti risponde **subito** 200 `{ received: true }` (T-22, D-32) e in `after()` esegue `syncOneWorkout`; esito in `sync_state.last_webhook` (`{ workoutId, status, at }`). | `handle-webhook.test.ts` (§ 6): 401, 400, 200 con valore grezzo e con `Bearer`, sync eseguito (`runAfter` iniettato raccoglie la promessa e il test la attende). Mutazione: confrontare solo la forma `Bearer` → il test con il valore grezzo va rosso. |
| A3 | `vercel.json` con `regions: ["dub1"]`, il cron `{ "path": "/api/cron/review", "schedule": "17 6 * * 6" }`, `git.deploymentEnabled = { "**": false, "main": true }` (T-19); un test lo legge e verifica i valori. | `vercel-config.test.ts`. Mutazione: `"17 6 * * *"` (ogni giorno) → rosso; togliere la chiave `"**"` → rosso. |
| A4 | Esiste il database Turso `hevy-coach` nel gruppo in `aws-eu-west-1`, **creato dall'import di `local.db`** (D-41), con `__drizzle_migrations` a 2 righe (`0000_*`, `0001_reviews`; la `0002_push` arriva con la 07) e gli stessi conteggi di `local.db` in `workouts`, `slot_state` (30), `slot_state_history`, `reviews`, `proposals`, `hevy_routines` (5), `hevy_templates` (29). | Collaudo § 7 D: `pnpm db:migrate:remote` (no-op) e le query nella SQL console del dashboard. |
| A5 | Esiste il progetto Vercel `hevy-coach` collegato al repo GitHub, regione `dub1`, env di Production impostate (§ 4.3), deploy di `main` verde, `https://<prod>/login` raggiungibile (`<prod>` = il dominio che Vercel mostra in Settings → Domains), le pagine dietro login. | Collaudo. |
| A6 | Il cron risulta nel dashboard; un'esecuzione manuale (`pnpm dlx vercel crons run /api/cron/review` o il bottone del dashboard) crea una riga `cron_runs` con l'esito atteso dichiarato prima (§ 4.4 passo 6); il sabato successivo ne crea un'altra da sola. | Collaudo F(3) + misura di chiusura (§ 8) per l'esecuzione manuale; la riga del sabato è storia, annotata nella riga Stato quando arriva, non un criterio da aspettare (D-56). |
| A7 | Il webhook è iscritto su hevy.com con l'URL di produzione; salvando un allenamento su Hevy, entro un minuto `workouts` ha la riga e `/` mostra «Ultimo allenamento» aggiornato. | Collaudo. |
| A8 | `SETUP.md` completo: da clone a produzione, per un altro utente (D-34), in inglese. | Rilettura dell'owner. |

## 3. Decisioni

- **D-1/D-34** (Vercel Hobby, Turso, repo pubblico), **D-2** (chiave Anthropic della Console
  personale), **D-13** (produzione aperta + auth applicativa; preview spente), **D-16** (cron),
  **D-23** (main = produzione), **D-32** (webhook, 200), **D-36** (le route leggono `process.env`
  per i segreti del guard, come il proxy), **D-37** (questa spec rifà in produzione le misure di
  chiusura di 03–05b, § 7 D e F), **D-38** (chiusura: passi di produzione senza codice nuovo →
  commit → merge = deploy → passi post-deploy riferiti in chat, scritti nel primo commit della 07),
  **D-41** (DB di produzione da `local.db`). **T-1** (dub1 / eu-west-1), **T-5** (`run-cron.ts` sta
  in `src/lib/review/`, `handle-webhook.ts` in `src/lib/sync/`: nessuna cartella nuova), **T-11**
  (lock), **T-12** (route), **T-16**, **T-19** (`vercel.json`, preview spente), **T-21**
  (container), **T-22** (200 subito, sync in `after()`; il cron sincronizza sempre), **T-23** (il
  client Hevy ha il timeout: una GET appesa non mangia i 300 s), **T-24** (`APP_URL` non esiste),
  **T-27** (`db:migrate:remote` e `hevy:sync:remote` con `--env-file`, niente `dotenv-cli`).
- Adottate qui, ribaltabili: (a) le migrazioni in produzione le lancia l'owner dal PC con `pnpm
  db:migrate:remote` e le env remote in `.env.turso-migrate` (gitignored), **prima** del deploy che le
  usa, mai nel build di Vercel; (b) il primo sync in produzione si fa dal PC (`pnpm
  hevy:sync:remote`, incrementale: il DB è nato pieno), non dal cron; (c) il valore dell'header del
  webhook è lo stesso segreto `HEVY_WEBHOOK_SECRET` (`openssl rand -hex 32`), inserito a mano nella
  pagina Hevy; (d) tutte le env di Production sono di tipo **Secret** tranne
  `ENABLE_EXPERIMENTAL_COREPACK` (Config); (e) un `cron_runs` ogni invocazione, mai una sola per
  giorno; (f) il cron risponde 500 su `error` (Vercel non ritenta: è solo un segnale nel dashboard);
  (g) il webhook ha `maxDuration = 60`; (h) i due guard confrontano i segreti con `isPasswordCorrect`
  (C-14, tempo costante), mai con `===`; (i) il cron senza `plan_uploaded_at` è un `error` esplicito
  («setup: …») e non un `review_not_due` silenzioso ogni sabato; (j) una `reviews` `skipped/
  already_running` dal cron chiude con `review_run` e non notifica (T-17); (k) `ENABLE_EXPERIMENTAL_
  COREPACK=1` su Vercel, così il build usa la versione di pnpm di `packageManager` (pnpm 11); il
  comando di build è lo script `build` di package.json (`next build --webpack`, T-4).

## 4. La soluzione

### 4.1 Cron — `src/app/api/cron/review/route.ts` e `src/lib/review/run-cron.ts`

```ts
// route.ts
import { getDb, getHevyClient, getReviewModel } from '@/lib/container';
import { isCronAuthorized, runCron } from '@/lib/review/run-cron';

export const maxDuration = 300;

export async function GET(request: Request) {
  if (!isCronAuthorized(request.headers.get('authorization'), process.env.CRON_SECRET)) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }
  return runCron({ db: getDb(), hevy: getHevyClient(), model: getReviewModel(), now: Date.now(), clock: Date.now });
}
```

```ts
// run-cron.ts
export interface CronDeps { db: Db; hevy: HevyClient; model: ReviewModel; now: number; clock: () => number }

/** False when the secret is not configured: an unset env must never open the route. */
export function isCronAuthorized(header: string | null, secret: string | undefined): boolean {
  if (!secret || !header) return false;
  return isPasswordCorrect(header, `Bearer ${secret}`);
}

function errorMessage(error: unknown): string { return error instanceof Error ? error.message : String(error); }

export async function runCron({ db, hevy, model, now, clock }: CronDeps): Promise<Response> {
  const id = nanoid();
  await insertCronRun(db, { id, startedAt: now });
  try {
    if ((await getSyncValue(db, 'plan_uploaded_at')) === null) throw new Error('setup: run pnpm hevy:sync:remote first');
    const sync = await syncWorkouts({ db, hevy, now });
    const due = await shouldRunScheduledReview(db, now);
    if (!due) {
      await finishCronRun(db, id, { outcome: 'review_not_due', details: { sync }, finishedAt: clock() });
      return NextResponse.json({ outcome: 'review_not_due', sync });
    }
    const review = await runReview({ db, hevy, model, now, clock, trigger: 'cron' }); // now = the function's start: the deadline counts the sync above too
    const push = await sendReviewNotification({ db, review });
    await finishCronRun(db, id, { outcome: 'review_run', details: { sync, review, push }, finishedAt: clock() });
    return NextResponse.json({ outcome: 'review_run', review, push });
  } catch (error) {
    await finishCronRun(db, id, { outcome: 'error', details: { message: errorMessage(error) }, finishedAt: clock() });
    return NextResponse.json({ outcome: 'error', message: errorMessage(error) }, { status: 500 });
  }
}
```

`runReview` gestisce da sé lock, il sync già fatto (il secondo sync incrementale costa una GET senza
eventi) ed esiti `skipped`/`failed`; `sendReviewNotification` con una revisione `skipped/no_workouts`
o `failed` manda il testo corrispondente, con `skipped/already_running` non manda nulla (T-17); una
`failed` è comunque `outcome 'review_run'` (la revisione è stata tentata) e la notifica dice che è
fallita. Una function uccisa a 300 s lascia la riga `started` senza `finished_at`: è così che si
riconosce (§ 8).

Repository nuovo `src/lib/db/repositories/cron-runs.ts` (AGENTS.md § 2.1): `insertCronRun(db, { id,
startedAt }: { id: string; startedAt: number }): Promise<void>` (scrive `outcome = 'started'`,
`details_json = '{}'`), `finishCronRun(db, id, { outcome, details, finishedAt }: { outcome:
'review_run' | 'review_not_due' | 'error'; details: unknown; finishedAt: number }): Promise<void>`
(`details_json = JSON.stringify(details)`), `getLatestCronRun(db)` (per il collaudo e per
`doc/guide/deploy.md`). Il commento di `cron_runs.outcome` nello schema (04) dice già `'started' |
'review_run' | 'review_not_due' | 'error'`.

### 4.2 Webhook — `src/app/api/hevy/webhook/route.ts` e `src/lib/sync/handle-webhook.ts`

```ts
// route.ts
import { after } from 'next/server';
export const maxDuration = 60;
export function POST(request: Request) {
  return handleWebhookRequest(request, {
    secret: process.env.HEVY_WEBHOOK_SECRET,
    deps: () => ({ db: getDb(), hevy: getHevyClient(), now: Date.now() }),
    runAfter: after,
  });
}
```

```ts
// handle-webhook.ts
export interface WebhookDeps { db: Db; hevy: HevyClient; now: number }
export function isWebhookAuthorized(header: string | null, secret: string | undefined): boolean {
  if (!secret || !header) return false;
  return isPasswordCorrect(header, secret) || isPasswordCorrect(header, `Bearer ${secret}`);
}
export async function handleWebhookRequest(request: Request, { secret, deps, runAfter }: {
  secret: string | undefined; deps: () => WebhookDeps; runAfter: (task: () => Promise<void>) => void;
}): Promise<Response> {
  if (!isWebhookAuthorized(request.headers.get('authorization'), secret)) {
    console.warn('hevy webhook unauthorized', { headers: [...request.headers.keys()] }); // names only, never values
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }
  const body: unknown = await request.json().catch(() => null);
  const workoutId = typeof body === 'object' && body !== null && typeof (body as { workoutId?: unknown }).workoutId === 'string'
    ? (body as { workoutId: string }).workoutId : null;
  if (!workoutId) return NextResponse.json({ error: 'workoutId required' }, { status: 400 });
  runAfter(() => handleWebhook(deps(), workoutId));
  return NextResponse.json({ received: true }, { status: 200 });
}
export async function handleWebhook({ db, hevy, now }: WebhookDeps, workoutId: string): Promise<void>
```

`handleWebhook`: `const { status } = await syncOneWorkout({ db, hevy, now }, workoutId)` →
`setSyncValue(db, 'last_webhook', JSON.stringify({ workoutId, status, at: now }), now)`; un errore
viene catturato e scritto nello stesso valore con `status: 'error'` e `message`. Un invio doppio di
Hevy è innocuo: l'upsert è idempotente e `last_webhook` viene riscritto con lo stesso contenuto. Un
workout cancellato su Hevy arriva come `not_found` e la copia locale viene cancellata (D-48).

`isPublicPath` (05a) contiene già `/api/cron/` e `/api/hevy/webhook`.

### 4.3 Configurazione e ambienti

`vercel.json`:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "regions": ["dub1"],
  "crons": [{ "path": "/api/cron/review", "schedule": "17 6 * * 6" }],
  "git": { "deploymentEnabled": { "**": false, "main": true } }
}
```

(Doc Vercel `https://vercel.com/docs/project-configuration/git-configuration`, letta il 2026-10-24:
le chiavi sono pattern minimatch; `*` non attraversa `/`, quindi serve `**` per `feature/06-deploy`;
un branch che combacia con più regole deploya se almeno una è `true`: `main` combacia con entrambe
e deploya, gli altri no.)

Env. `EnvSchema` (`src/lib/env-schema.ts`, D-36) aggiunge `CRON_SECRET: z.string().min(16)` e
`HEVY_WEBHOOK_SECRET: z.string().min(16)`; `.env.example` li elenca con i generatori;
`vitest.config.ts` aggiunge i valori finti `CRON_SECRET: 'test-cron-secret-0000'`,
`HEVY_WEBHOOK_SECRET: 'test-hook-secret-0000'`; `package.json` → `e2e:server` (05a) aggiunge
`CRON_SECRET=e2e-cron-secret-00000 HEVY_WEBHOOK_SECRET=e2e-webhook-secret-0000`. `APP_URL` non
esiste (T-24).

| Variabile | Development (`.env.local`) | Production (Vercel, Secret) |
|---|---|---|
| `HEVY_API_KEY` | chiave reale | chiave reale |
| `ANTHROPIC_API_KEY` | chiave della Console personale | idem |
| `TURSO_DATABASE_URL` | `file:local.db` | `libsql://hevy-coach-<org>.aws-eu-west-1.turso.io` |
| `TURSO_AUTH_TOKEN` | vuoto | token senza scadenza |
| `APP_PASSWORD` | una password locale (≥ 12) | la password vera (≥ 12) |
| `SESSION_SECRET` | `openssl rand -base64 32` | un altro valore |
| `CRON_SECRET` | `openssl rand -base64 24` (serve al curl del collaudo C) | un altro valore |
| `HEVY_WEBHOOK_SECRET` | `openssl rand -hex 32` (serve al curl del collaudo C) | un altro valore |
| `ENABLE_EXPERIMENTAL_COREPACK` | — | `1` (Config: fa rispettare `packageManager`) |

Script di package.json (T-27): `"db:migrate:remote": "node --env-file=.env.turso-migrate
./node_modules/drizzle-kit/bin.cjs migrate"` e `"hevy:sync:remote": "tsx --env-file=.env.turso-migrate
--env-file-if-exists=.env.local scripts/hevy-sync.ts"` (`--env-file` di Node non sovrascrive le
variabili già nell'ambiente, e il primo file vince sul secondo: `TURSO_*` remoti, il resto da
`.env.local`). Ogni altro script CLI si lancia contro la produzione con lo stesso prefisso
(`pnpm exec tsx --env-file=.env.turso-migrate --env-file-if-exists=.env.local scripts/review-show.ts`).
`drizzle.config.ts` (03) carica `.env.local` con `dotenv` senza `override`: le variabili remote già
presenti vincono.

`.gitignore` aggiunge `.env.turso-migrate` e `.vercel/`. Il collaudo A verifica `git check-ignore
.env.turso-migrate .vercel` (stampa entrambi).

### 4.4 Passi di messa in esercizio (nell'ordine di D-38; ogni passo con effetti fuori dal disco dopo l'OK)

Sul PC dell'owner (Windows, senza WSL) si usa il dashboard Turso `https://app.turso.tech`; i comandi
CLI equivalenti stanno solo in SETUP.md (dopo `turso db locations` per il codice della regione).

1. **`local.db` pronto all'import**: `pnpm hevy:sync` (GET reali di lettura, gratis); chiudere ogni
   processo che usa `local.db` (dev server, test); se esiste `local.db-wal` con dimensione > 0,
   `pnpm exec tsx -e "import { createClient } from '@libsql/client'; await createClient({ url: 'file:local.db' }).execute('PRAGMA wal_checkpoint(TRUNCATE)')"`
   e verificare che il file `-wal` sia vuoto o assente. Esito atteso: un solo file `local.db`.
2. **Turso** (dashboard): Create group `hevy-coach` in Dublin (aws-eu-west-1); Create database
   `hevy-coach` → **Import from SQLite file** → `local.db` (D-41; scrittura permanente sull'account
   Turso, gratuita nel piano Free); Generate token senza scadenza; copiare URL e token. Esito atteso:
   nella SQL console `SELECT COUNT(*) FROM __drizzle_migrations` = 2 e i conteggi di A4 uguali a
   quelli locali (letti prima con lo stesso SQL su `local.db` via `pnpm exec tsx -e …`).
3. **Migrazioni e primo sync**: `.env.turso-migrate` con `TURSO_DATABASE_URL` e `TURSO_AUTH_TOKEN`
   remoti; `pnpm db:migrate:remote` (esito atteso: nessuna migrazione applicata); `pnpm
   hevy:sync:remote` (esito atteso: **sole GET** reali, gratis: `workouts/events` dal cursore
   importato, cartelle, routine, 5 pagine di template in sola lettura, `routines=5 templates=29`;
   se la stampa annuncia un template mancante, ci si ferma: i template esistono dalla 02). Verifica:
   `SELECT value FROM sync_state WHERE key = 'last_sync_at'` nella SQL console.
4. **Vercel**: `pnpm dlx vercel@latest login` (account personale) · `pnpm dlx vercel link` (nuovo
   progetto `hevy-coach`, framework Next.js; crea `.vercel/`, ignorato) · nel dashboard: Settings →
   Git → repository `GiuseppeDM98/hevy-coach`, production branch `main`; Settings → Deployment
   Protection → Vercel Authentication **Standard Protection** (rete, le preview sono spente);
   Settings → Environment Variables (Production): le otto Secret con `pnpm dlx vercel env add <NAME>
   production` una per una (il valore si incolla dal prompt, mai in chat né in un file), poi `pnpm
   dlx vercel env add ENABLE_EXPERIMENTAL_COREPACK production --type config` con valore `1`. La
   regione la fissa `vercel.json`. Esito atteso: nove variabili in Production, nessuna in Preview.
5. **Commit, PR, merge = deploy** (D-38): il commit unico della sessione con Stato «implementata» e
   gli esiti dei passi 1–4; `gh pr create`; dopo l'OK, merge su `main` → deploy automatico. Esito
   atteso: deploy verde; `<prod>` letto in Settings → Domains; `https://<prod>/login` → `/` con la
   password; `https://<prod>/progressi` senza cookie → redirect a `/login`.
6. **Cron** (post-merge, riferito in chat): Settings → Cron Jobs mostra `/api/cron/review`. Prima
   dell'esecuzione manuale si legge `pnpm exec tsx --env-file=.env.turso-migrate
   --env-file-if-exists=.env.local scripts/review-show.ts`: se l'ultima `completed` ha meno di 13
   giorni (la revisione manuale della 05b), esito atteso `review_not_due` con sole GET Hevy; se ha
   13 giorni o più, l'esecuzione farebbe una revisione reale (**fino a due chiamate ad Anthropic, <
   0,80 $**, proposte aperte in produzione, `superseded` sulle precedenti) e si chiede l'OK con quel
   costo. Poi `pnpm dlx vercel crons run /api/cron/review` (o il bottone del dashboard) → una riga in
   `cron_runs` con l'esito atteso e `finished_at` non null.
7. **Webhook** (post-merge): su hevy.com → Settings → Developer → Webhooks: URL
   `https://<prod>/api/hevy/webhook`, header = il valore di `HEVY_WEBHOOK_SECRET` → Subscribe
   (scrittura sulle impostazioni dell'account Hevy, a mano). Salvare un allenamento nell'app Hevy
   con un solo set (scrittura sull'account Hevy, poi cancellata a mano) → entro un minuto
   `sync_state.last_webhook` con `status 'upserted'`. Se entro un minuto non c'è: si leggono subito i
   log runtime della function su Vercel (conservati 1 ora): la riga `hevy webhook unauthorized` dice
   i nomi degli header ricevuti; ci si ferma e si chiede all'owner prima di cambiare la route.
   Cancellare l'allenamento di prova su Hevy: l'evento `deleted` arriva al prossimo sync (il webhook
   per la cancellazione, se Hevy lo manda, dà `not_found` e cancella subito).

### 4.5 SETUP.md (D-34, inglese)

Sezioni: Requirements (Node ≥ 22.9, pnpm 11, Hevy Pro, Anthropic Console key, Vercel, Turso) ·
Clone and install · Environment variables (la tabella di § 4.3 con il significato e i generatori) ·
Your plan («edit `src/lib/plan/plan.ts` and `src/lib/plan/scales.ts`, then `pnpm hevy:upload`;
after the first «Applica», `pnpm hevy:upload -- --from-db`», D-40) · Local run (`pnpm db:migrate`,
`pnpm hevy:sync`, `pnpm dev`) · Deploy (i passi 1–7, con `<your-project>.vercel.app`; CLI Turso:
`turso db locations`, `turso group create <name> --location <code>`, `turso db create <name> --group
<name> --from-file local.db`, `turso db tokens create <name> -e never`) · Operations (il cron parte
il sabato fra le 06:00 e le 06:59 UTC: 07:00–07:59 a Roma d'inverno, 08:00–08:59 d'estate; webhook;
`cron_runs` è il log; come rifare un full sync remoto con `pnpm hevy:sync:remote -- --full`) · Costs
(Opus 5.5: il costo misurato in `doc/guide/revisione.md`) · Troubleshooting (401 dal cron: il segreto
nel comando manuale non è quello salvato su Vercel; webhook senza righe: leggere i nomi degli header
nei log; token Turso revocato o ruotato: 401 dal database → nuovo token e `pnpm dlx
vercel env rm TURSO_AUTH_TOKEN production` + `env add`; riga `cron_runs` ferma a `started`: function
uccisa a 300 s, si rilancia a mano).

## 5. Vincoli e trappole

- **AGENTS.md § 8 Deploy and PWA** (stub della pianificazione; qui la parte Deploy diventa regole)
  — migrazioni dal PC prima del deploy; env verità nel dashboard; `cron_runs` è il log (Vercel ne
  tiene 1 ora); mai `vercel --prod` a mano (deploya `main`); mai `APP_URL`/`VERCEL_URL`: l'app non
  conosce il proprio dominio.
- Il cron può chiamare due volte: `runReview` ha il lock (T-11) e il sync è idempotente; `cron_runs`
  avrà due righe: giusto così.
- Il cron non segue redirect e vuole 2xx: la route risponde JSON, mai redirect; `isPublicPath` deve
  restare vero per `/api/cron/` (test 05a).
- `after()` richiede Node runtime (default) e termina entro `maxDuration`.
- `TURSO_DATABASE_URL` `libsql://` con l'entry Node del client va in HTTPS (verificato nel sorgente
  del client, 2026-10-23): niente websocket da configurare. Le pragma di connessione della 03 si
  applicano solo a `file:`; nessun percorso dell'app dipende da `ON DELETE CASCADE` (le cancellazioni
  sono esplicite, 03): il collaudo D legge comunque `PRAGMA foreign_keys` nella SQL console remota e
  annota il valore in `doc/guide/deploy.md`.
- Hobby è «personal, non-commercial»: l'app lo è.
- Segreti: `vercel env add` legge da stdin; mai scriverli in `vercel.json`, in chat, nelle PR, in
  `SESSION_NOTES.md`.
- Il cron del sabato con il cursore presente fa un sync incrementale: pochi secondi. Un full sync
  (cursore assente) dura decine di secondi e sta nei 300 s insieme alla revisione (la scadenza di
  `runReview` parte da `now` della function e li conta).

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit + DB | `src/lib/review/run-cron.test.ts`: `isCronAuthorized` (header assente, sbagliato, segreto vuoto, `'Bearer '` con segreto vuoto → false; giusto → true); `runCron` con `clock: () => now`: `plan_uploaded_at` assente → `error` «setup: …» e 500; `review_not_due` (riga con `finished_at`, `details.sync`); `review_run` con fake model → `details_json` contiene `review.id` e `push`; fake Hevy che lancia → `error` + 500, riga chiusa. | Non scrivere `cron_runs` in caso di errore → rosso; togliere `!secret` → rosso; `clock: Date.now` fisso nel codice → il caso `review_run` esce `failed/model: timeout` → rosso. |
| Unit + DB | `src/lib/sync/handle-webhook.test.ts` su `handleWebhookRequest` con `runAfter` che raccoglie la promessa: 401 (assente, sbagliato, segreto vuoto con `isWebhookAuthorized(null, '')` e `('', '')`), 400, 200 con valore grezzo e con `Bearer`; `last_webhook` scritto con `status: 'upserted'`; id ignoto → `status: 'not_found'`; fake Hevy che lancia → `status: 'error'`. | Confrontare solo la forma `Bearer` → rosso; `status: 202` → rosso; togliere il `try` → il caso `error` va rosso. |
| Unit | `vercel-config.test.ts` (in `tests/`): `regions[0] === 'dub1'`, schedule esattamente `17 6 * * 6`, path, `deploymentEnabled['**'] === false`, `deploymentEnabled.main === true`. | `"17 6 * * *"` → rosso; togliere `"**"` → rosso. |
| Unit | `env-schema.test.ts` esteso: `CRON_SECRET` di 15 caratteri → errore; `HEVY_WEBHOOK_SECRET` di 15 → errore. | Togliere `min(16)` → rosso. |
| Unit | `guards.test.ts` (05a): le due route nuove sono escluse dal guard per percorso ed hanno `maxDuration`. | Già automatico. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E verdi in locale (nessuna chiamata reale); `git check-ignore
  .env.turso-migrate .vercel` stampa entrambi.
- **C — Locale**: da Git Bash, `set -a; . ./.env.local; set +a` (le variabili restano nella sola
  shell). Cron: prima `pnpm review:show`; esito atteso `review_not_due` (l'ultima `completed` in
  `local.db` è la manuale della 05b, meno di 13 giorni fa); se ha 13 giorni o più, il curl farebbe
  una revisione reale (fino a due chiamate ad Anthropic, < 0,80 $) e si chiede l'OK con quel costo;
  verso Hevy sole GET (`workouts/events`). `curl -i -H "Authorization: Bearer $CRON_SECRET"
  localhost:3000/api/cron/review` → 200 e JSON con `outcome`, riga in `cron_runs` con `finished_at`.
  Webhook (1 GET reale `workouts/{id}`, lettura, gratis): `curl -i -X POST -H "Authorization:
  $HEVY_WEBHOOK_SECRET" -H "content-type: application/json" --data-binary @- localhost:3000/api/hevy/webhook
  <<< '{"workoutId":"<id dell'ultimo workout in local.db>"}'` → 200 e `last_webhook` con `upserted`.
- **E — Casi negativi**: gli stessi due curl con un segreto sbagliato → 401 e, per il webhook, la
  riga `hevy webhook unauthorized` nel log del dev server con i nomi degli header.
- **D — Produzione (passi 1–5 di § 4.4, ciascuno dopo l'OK, esito atteso detto prima)**: `local.db`
  checkpointato; Turso creato dall'import (conteggi uguali, `PRAGMA foreign_keys` annotato);
  `db:migrate:remote` no-op e `hevy:sync:remote` sole GET; Vercel con nove env; merge → deploy
  verde; `/login` → `/` con la password e `/progressi` senza cookie → redirect. Verifica nella SQL
  console del dashboard, mai «a occhio». Qui si rifanno in produzione le misure di chiusura (D-37)
  di 03 (`hevy:sync:remote` incrementale verde) e 04 (`review-show` remoto mostra la revisione
  `completed` importata).
- **F — Giro guidato (owner, telefono, browser) e passi 6–7**: (1) `https://<prod>` → login → Oggi
  con i dati veri (misura della 05a in produzione); (2) `/progressi` con le sessioni reali; (3) cron
  manuale (passo 6, esito atteso dichiarato) → riga `cron_runs`; (4) webhook (passo 7): salvare un
  allenamento su Hevy e vedere «Ultimo allenamento» cambiare entro un minuto; (5) misura della 05b
  in produzione, **dopo l'OK e dopo che l'owner ha applicato o ignorato le proposte aperte che vuole
  tenere**: «Esegui revisione ora» dall'app in produzione (GET Hevy + fino a due chiamate ad
  Anthropic, < 0,80 $; `superseded` sulle aperte, D-30) → `/revisioni/<id>` con il summary. Il bug
  sarebbe: una pagina senza login, un 401 del cron con il segreto giusto, un webhook che non scrive.
  Che cosa NON copre: l'E2E in produzione (gira solo in locale) e la push (07).
- **G — Ripristino**: l'allenamento di prova cancellato su Hevy; `.env.turso-migrate` resta locale e
  ignorato; `local.db` resta la copia di sviluppo (da qui in poi la produzione vive per sé: i due
  database divergono ed è voluto).

Che cosa NON copre: push (07), il primo cron reale del sabato (storia, non segnale: D-56).

## 8. Messa in esercizio

È la spec stessa (§ 4.4). Chiusura secondo D-38: il commit unico porta Stato «implementata» con gli
esiti dei passi 1–4; i passi 6–7 e la misura di chiusura si riferiscono in chat e si scrivono (Stato
«in esercizio il <data>», CLAUDE.md Latest, § 5 dell'indice) nel primo commit della sessione 07.

Misura di chiusura (D-56, 2026-10-24): **il cron lanciato a mano in produzione** (passo 6: `pnpm dlx
vercel crons run /api/cron/review` o il bottone del dashboard, esito atteso dichiarato prima) lascia
in `cron_runs` una riga con `finished_at` non null e `outcome` in (`review_not_due`, `review_run`);
se `review_run`, `details_json` ha `review.status` diverso da `failed`. Si legge nella SQL console di
Turso, mai «a occhio». Il **primo sabato reale NON è un segnale** e non si aspetta: quando arriva, la
sua riga (`started_at` fra le 06:00 e le 06:59 UTC, nata da sola) si annota nella riga Stato di questa
spec come storia; se quel sabato la riga non c'è (Hobby è best effort) si legge il log dei cron nel
dashboard e si rilancia a mano. La 07 parte appena la 06 è in esercizio, sabato o no.

## 9. Documentazione

- `SETUP.md` (nuovo, completo, § 4.5).
- `README.md § Documentation`: la riga di SETUP.md diventa il link «From clone to your own
  deployment».
- `CLAUDE.md`: Latest; § Current Status → Stack: Vercel `dub1`, Turso `aws-eu-west-1` già scritti;
  § Data & Integrations (Turso remoto, Vercel, cron, webhook); § Not yet present: togliere la riga
  «SETUP.md: born in spec 06» (dominio personalizzato, preview e alert sui sync ci sono già).
- `AGENTS.md § 8 Deploy and PWA`: la parte Deploy da stub a regole (≈ 10 righe: § 5); § 10 Commands
  (+ `db:migrate:remote`, `hevy:sync:remote`).
- `WORKFLOW.md § 3`: produzione: come si legge il DB remoto (SQL console, `--env-file=.env.turso-migrate`),
  cosa non si fa mai in produzione (`vercel --prod`, scritture a mano sul DB).
- `doc/guide/deploy.md` (≤ 200 righe): i comandi, le env, come ruotare un segreto, come rifare il
  full sync remoto, il valore di `PRAGMA foreign_keys` letto, gli orari del cron a Roma, cosa fare se
  Hevy cambia l'API.
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/deploy.md` (≤ 200).

## 10. Fuori ambito, dichiarato

- Push, icona e manifest → **07**. Dominio personalizzato, preview deploy, alert su un errore di
  sync senza revisione → già in CLAUDE.md § Not yet present. Monitoraggio su `cron_runs` `error`:
  la notifica push di `failed` (07) copre la revisione.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **M** di codice, **L** di operazioni.

```
Ciao, in questa sessione implemento la spec
doc/v1/06-messa-in-esercizio-vercel-turso-cron-webhook.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/sync.md, doc/guide/revisione.md, doc/guide/app.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 06 INTERA
- Verifica con un grep i contratti che usa (syncWorkouts, syncOneWorkout, runReview, ReviewModel,
  shouldRunScheduledReview, sendReviewNotification, PushReport, getDb, getHevyClient,
  getReviewModel, isPublicPath, isPasswordCorrect, getSyncValue, setSyncValue, EnvSchema, nanoid,
  createMemoryDb, createFakeHevy, lo schema cron_runs della 04)

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/06-deploy. Crea SESSION_NOTES.md.

Ordine: (1) route e run-cron, handle-webhook, src/lib/db/repositories/cron-runs.ts, vercel.json,
src/lib/env-schema.ts, .env.example, .gitignore, package.json (e2e:server, db:migrate:remote,
hevy:sync:remote) e i test; (2) SETUP.md e la documentazione; (3) § 7 A, C, E in locale; poi i
passi 1–4 di § 4.4 UNO PER MESSAGGIO, ciascuno dopo il mio OK, con l'esito atteso detto prima e la
verifica sul database remoto, mai a occhio; (4) commit unico e PR; dopo il mio OK al merge (che è
il deploy), i passi 6–7 e il giro F, riferiti in chat. Il cron manuale (locale e in produzione)
può fare fino a due chiamate reali ad Anthropic (< 0,80 $): dimmi prima l'esito atteso letto da
review:show e aspetta l'OK. I segreti li incollo io nei prompt dei comandi: non me li chiedere in
chat e non scriverli in nessun file tracciato. Se un punto non è coperto, fermati e chiedimelo con
AskUserQuestion. Le etichette della serie non entrano nel codice.

Prima del commit: riassumi il diff, aggiorna la riga «Stato» della spec («implementata», con gli
esiti dei passi 1–4), § 5 dell'indice e CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno
solo) e, dopo, per il merge su main.
```
