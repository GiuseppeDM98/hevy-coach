# 06 — Messa in esercizio: Vercel, Turso remoto, cron e webhook

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 05b.
> **Branch**: `feature/06-deploy` da `main`. **Dipende da**: 01–05b (l'app intera), con
> `syncWorkouts`, `syncOneWorkout`, `shouldRunScheduledReview`, `runReview`, `sendReviewNotification`,
> `sessionDenied`, `isPublicPath`.
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-2, D-13, D-16, D-23, D-32, D-34;
> scelte T-1, T-11, T-12, T-16, T-19; contratti C-7 (`cron_runs` già creata dalla 04).

## 1. Il problema oggi (prove)

1. L'app gira solo in locale su `file:local.db`. Non esistono il progetto Vercel né il database
   Turso: la loro creazione è un passo di questa spec, con i comandi esatti (tema, BOOTSTRAP 7).
2. Vercel Hobby (doc letta il 2026-10-23; URL in § 4): cron **una volta al giorno al massimo**, UTC,
   precisione ±59 min, **solo produzione**, best effort (**duplicati possibili, nessun retry**),
   GET con `Authorization: Bearer $CRON_SECRET` e UA `vercel-cron/1.0`; function fino a 300 s;
   regione singola; log runtime 1 ora (→ `cron_runs` in tabella); Deployment Protection
   «All Deployments» gratuita ma non documentata come compatibile con il cron → produzione aperta
   con auth applicativa (D-13), Standard Protection sulle preview.
3. Turso: nessuna regione Francoforte → gruppo `aws-eu-west-1` (Dublino) e Vercel `dub1` (T-1).
   `vercel env pull` non rilegge i Secret: la verità delle env sta nel dashboard.
4. Il webhook di Hevy (D-32): POST `{ "workoutId" }` all'URL con l'header di autorizzazione
   impostato nella pagina Hevy «Webhooks»; risposta 200 entro **5 s**. Il nome dell'header non è
   documentato: la route confronta il valore di `Authorization` (grezzo o `Bearer <segreto>`).
5. Hevy chiede di non chiamare a xx:00 → cron `17 6 * * 6` (D-16).

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `GET /api/cron/review`: senza `Authorization: Bearer <CRON_SECRET>` → 401; con: inserisce `cron_runs` (started), `syncWorkouts` incrementale, `shouldRunScheduledReview` → se vero `runReview({ trigger: 'cron' })` + `sendReviewNotification`, altrimenti `review_not_due`; chiude `cron_runs` con outcome e dettagli (sync result, review id); un errore → outcome `error`, dettagli con il messaggio, risposta 500. `maxDuration = 300`. | `cron-route.test.ts` con DB in memoria, fake Hevy e fake model: 401; `review_not_due`; `review_run`; errore Hevy → `error` + 500. Mutazione: non scrivere `cron_runs` in caso di errore → rosso. |
| A2 | `POST /api/hevy/webhook`: header assente o diverso da `HEVY_WEBHOOK_SECRET` (o `Bearer ` + segreto) → 401; body senza `workoutId` stringa → 400; altrimenti risponde **subito** 202 `{ received: true }` e in `after()` esegue `syncOneWorkout`; esito in `sync_state.last_webhook` (`{ workoutId, status, at }`). | `webhook-route.test.ts`: 401, 400, 202 con sync eseguito (l'`after` è atteso nel test tramite un'iniezione `runAfter` che in test è sincrona). Mutazione: confrontare solo la forma `Bearer` → il test con il valore grezzo va rosso. |
| A3 | `vercel.json` con `regions: ["dub1"]`, il cron `{ "path": "/api/cron/review", "schedule": "17 6 * * 6" }`, `git.deploymentEnabled.main = true`; un test lo legge e verifica i tre valori. | `vercel-config.test.ts`. Mutazione: `"17 6 * * *"` (ogni giorno) → rosso. |
| A4 | Esiste il database Turso `hevy-coach` nel gruppo in `aws-eu-west-1`, migrato (`0000`, `0001`, `0002` se la 07 è già scritta: no, `0000` e `0001`), popolato con il sync iniziale e le 5 `hevy_routines`. | Collaudo § 7: `pnpm db:migrate` con le env remote; `SELECT COUNT(*)` via `turso db shell`/dashboard. |
| A5 | Esiste il progetto Vercel `hevy-coach` collegato al repo GitHub, regione `dub1`, env di Production impostate (§ 4.3), deploy di `main` verde, `https://<prod>/login` raggiungibile, le pagine dietro login. | Collaudo. |
| A6 | Il cron risulta nel dashboard; un'esecuzione manuale (`vercel crons run /api/cron/review` o il bottone del dashboard) crea una riga `cron_runs`; il sabato successivo ne crea un'altra da sola. | Collaudo + misura di chiusura. |
| A7 | Il webhook è iscritto su hevy.com con l'URL di produzione; salvando un allenamento su Hevy, entro un minuto `workouts` ha la riga e `/` mostra «Ultimo allenamento» aggiornato. | Collaudo. |
| A8 | `SETUP.md` completo: da clone a produzione, per un altro utente (D-34). | Rilettura dell'owner. |

## 3. Decisioni

- **D-1/D-34** (Vercel Hobby, Turso, repo pubblico), **D-2** (chiave Anthropic della Console
  personale), **D-13** (produzione aperta + auth applicativa; preview protette), **D-16** (cron),
  **D-23** (main = produzione), **D-32** (webhook). **T-1** (dub1 / eu-west-1), **T-11** (lock),
  **T-12** (route), **T-16**.
- **T-19 (adottata qui, ribaltabile)**: `git.deploymentEnabled` solo `main`: **nessun deploy di
  preview** per i branch `feature/*` (il collaudo si fa in locale, WORKFLOW § 3); Standard Protection
  resta attiva come impostazione del progetto per sicurezza. Se un giorno servono le preview, si
  toglie la chiave e la protezione è già giusta.
- Adottate qui, ribaltabili: (a) le migrazioni in produzione le lancia l'owner dal PC con
  `pnpm db:migrate` e le env remote in `.env.turso-migrate` (gitignored), **prima** del deploy che
  le usa, mai nel build di Vercel; (b) il sync iniziale in produzione si fa dal PC
  (`pnpm hevy:sync` con le stesse env), non dal cron; (c) `APP_URL` è un'env esplicita
  (`https://hevy-coach.vercel.app` o il dominio) usata per i link delle notifiche (07): niente
  `VERCEL_URL`; (d) il webhook risponde 202 prima di sincronizzare (`after()` di `next/server`),
  perché il limite di 5 s di Hevy non deve dipendere dal tempo di una GET + batch su Turso;
  (e) il cron sincronizza **sempre** (anche quando la revisione non è dovuta), così la dashboard è
  aggiornata al sabato anche senza webhook; (f) un `cron_runs` ogni invocazione, mai una sola per
  giorno; (g) il valore dell'header del webhook è lo stesso segreto `HEVY_WEBHOOK_SECRET` (32+
  caratteri casuali), inserito a mano nella pagina Hevy; (h) tutte le env di Production sono di tipo
  **Secret** tranne `APP_URL` e `NEXT_PUBLIC_*`.

## 4. La soluzione

### 4.1 Route — `src/app/api/cron/review/route.ts`

```ts
export const maxDuration = 300;

export async function GET(request: Request) {
  const expected = `Bearer ${process.env.CRON_SECRET ?? ''}`;
  if (!process.env.CRON_SECRET || request.headers.get('authorization') !== expected) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }
  return runCron({ db: getDb(), hevy: getHevyClient(), model: getReviewModel(), now: Date.now() });
}
```

`src/lib/cron/run-cron.ts` (testabile, riceve le dipendenze):

```ts
export async function runCron(deps): Promise<Response> {
  const id = nanoid(); const startedAt = deps.now;
  await insertCronRun(db, { id, startedAt, outcome: 'started', detailsJson: '{}' });
  try {
    const sync = await syncWorkouts({ db, hevy, now });
    const due = await shouldRunScheduledReview(db, now);
    if (!due) { await finishCronRun(db, id, 'review_not_due', { sync }); return NextResponse.json({ outcome: 'review_not_due', sync }); }
    const review = await runReview({ db, hevy, model, now, trigger: 'cron' });
    const push = await sendReviewNotification({ db, review });
    await finishCronRun(db, id, 'review_run', { sync, review, push });
    return NextResponse.json({ outcome: 'review_run', review, push });
  } catch (error) {
    await finishCronRun(db, id, 'error', { message: errorMessage(error) });
    return NextResponse.json({ outcome: 'error', message: errorMessage(error) }, { status: 500 });
  }
}
```

(`runReview` gestisce da sé lock, sync già fatto — il secondo sync incrementale costa una GET
senza eventi — ed esiti `skipped`/`failed`; `sendReviewNotification` con una revisione `skipped` o
`failed` manda il testo corrispondente, T-17.)

### 4.2 Route — `src/app/api/hevy/webhook/route.ts`

```ts
import { after } from 'next/server';
export const maxDuration = 60;

export async function POST(request: Request) {
  const secret = process.env.HEVY_WEBHOOK_SECRET ?? '';
  const header = request.headers.get('authorization') ?? '';
  if (!secret || (header !== secret && header !== `Bearer ${secret}`)) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  const body = await request.json().catch(() => null);
  const workoutId = typeof body?.workoutId === 'string' ? body.workoutId : null;
  if (!workoutId) return NextResponse.json({ error: 'workoutId required' }, { status: 400 });
  after(() => handleWebhook({ db: getDb(), hevy: getHevyClient(), now: Date.now() }, workoutId));
  return NextResponse.json({ received: true }, { status: 202 });
}
```

`src/lib/sync/handle-webhook.ts`: `handleWebhook(deps, workoutId)` → `syncOneWorkout` → `setSyncValue(db,
'last_webhook', JSON.stringify({ workoutId, status, at: now }))`; un errore viene catturato e scritto
nello stesso valore con `status: 'error'`. Nel test la route riceve `runAfter` iniettato (default
`after`), così l'handler si può attendere.

`isPublicPath` (05a) contiene già `/api/cron/` e `/api/hevy/webhook`.

### 4.3 Configurazione e ambienti

`vercel.json`:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "regions": ["dub1"],
  "crons": [{ "path": "/api/cron/review", "schedule": "17 6 * * 6" }],
  "git": { "deploymentEnabled": { "main": true } }
}
```

Env (`EnvSchema` aggiunge `CRON_SECRET: z.string().min(16)`, `HEVY_WEBHOOK_SECRET: z.string().min(16)`,
`APP_URL: z.string().url()`; `.env.example` li elenca):

| Variabile | Development (`.env.local`) | Production (Vercel, Secret) |
|---|---|---|
| `HEVY_API_KEY` | chiave reale | chiave reale |
| `ANTHROPIC_API_KEY` | chiave della Console personale | idem |
| `TURSO_DATABASE_URL` | `file:local.db` | `libsql://hevy-coach-<org>.aws-eu-west-1.turso.io` |
| `TURSO_AUTH_TOKEN` | vuoto | token senza scadenza |
| `APP_PASSWORD` | una password locale | la password vera (≥ 12) |
| `SESSION_SECRET` | `openssl rand -base64 32` | un altro valore |
| `CRON_SECRET` | qualunque (non usato) | `openssl rand -base64 24` |
| `HEVY_WEBHOOK_SECRET` | qualunque | `openssl rand -hex 32` |
| `APP_URL` | `http://localhost:3000` | `https://hevy-coach.vercel.app` (Config) |

### 4.4 Passi di messa in esercizio (nell'ordine; ogni passo con effetti fuori dal disco dopo l'OK)

1. **Turso** (dashboard `https://app.turso.tech` su Windows; la CLI Turso richiede WSL — doc
   `https://docs.turso.tech/cli/installation`, letta il 2026-10-23 — quindi i comandi sono
   l'equivalente per chi la ha):
   `turso group create hevy-coach --location aws-eu-west-1` · `turso db create hevy-coach --group hevy-coach`
   · `turso db show hevy-coach --url` · `turso db tokens create hevy-coach -e never`.
   Dal dashboard: Create group (Dublin), Create database `hevy-coach`, copia URL, Generate token.
2. **Migrazioni e dati**: `.env.turso-migrate` con `TURSO_DATABASE_URL` e `TURSO_AUTH_TOKEN` remoti,
   poi `pnpm exec dotenv -e .env.turso-migrate -- pnpm db:migrate` (`dotenv-cli` in devDependencies)
   e `pnpm exec dotenv -e .env.turso-migrate -- pnpm hevy:sync` (full sync: 18+ GET). Verifica:
   `turso db shell hevy-coach "SELECT COUNT(*) FROM workouts"` o il dashboard.
3. **Vercel**: `pnpm dlx vercel@latest login` (account personale) · `pnpm dlx vercel link`
   (nuovo progetto `hevy-coach`, framework Next.js) · nel dashboard: Settings → Git → repository
   `GiuseppeDM98/hevy-coach`, production branch `main`; Settings → Functions → region `dub1` (o
   `vercel.json`); Settings → Deployment Protection → Vercel Authentication **Standard Protection**;
   Settings → Environment Variables: le nove righe di § 4.3 (Production), con
   `pnpm dlx vercel env add <NAME> production` una per una (il valore si incolla dal prompt, mai in
   chat né in un file).
4. **Deploy**: merge della PR su `main` → deploy automatico; `https://hevy-coach.vercel.app/login`.
5. **Cron**: Settings → Cron Jobs mostra `/api/cron/review`; esecuzione manuale (`pnpm dlx vercel
   crons run /api/cron/review` o il bottone nel riepilogo del deploy) → `cron_runs` ha una riga
   `review_not_due` o `review_run`.
6. **Webhook**: su hevy.com → Settings → Developer → Webhooks: URL
   `https://hevy-coach.vercel.app/api/hevy/webhook`, header = `HEVY_WEBHOOK_SECRET` → Subscribe.
   Salvare un allenamento (anche vuoto, poi cancellarlo) → `sync_state.last_webhook`.
7. **Icona e dominio**: fuori ambito (07 e CLAUDE.md § Not yet present).

### 4.5 SETUP.md (D-34)

Sezioni: Requirements (Node 22, pnpm 11, Hevy Pro, Anthropic Console key, Vercel, Turso) ·
Clone and install · Environment variables (la tabella di § 4.3 con il significato) · Your plan
(«edit `src/lib/plan/plan.ts` and `src/lib/plan/scales.ts`, then `pnpm hevy:upload`») · Local run
(`pnpm db:migrate`, `pnpm hevy:sync`, `pnpm dev`) · Deploy (i passi 1–6, in inglese) · Operations
(cron, webhook, `cron_runs`, come rifare un full sync) · Costs (Opus 5.5: ~0.10 USD per review) ·
Troubleshooting (401 dal cron, webhook senza righe, Turso token scaduto).

## 5. Vincoli e trappole

- **AGENTS.md § Deploy** (nasce qui) — qui vuol dire che: migrazioni dal PC prima del deploy;
  env verità nel dashboard; `cron_runs` è il log (Vercel ne tiene 1 ora); mai `vercel --prod` a
  mano (deploya `main`).
- Il cron può chiamare due volte: `runReview` ha il lock (T-11) e il sync è idempotente; `cron_runs`
  avrà due righe: giusto così.
- Il cron non segue redirect e vuole 2xx: la route risponde JSON, mai redirect; `isPublicPath` deve
  restare vero per `/api/cron/` (test 05a).
- `after()` richiede Node runtime (default) e termina entro `maxDuration`.
- `TURSO_DATABASE_URL` `libsql://` con l'entry Node del client va in HTTPS (verificato nel sorgente
  del client, 2026-10-23): niente websocket da configurare.
- Hobby è «personal, non-commercial»: l'app lo è.
- Segreti: `vercel env add` legge da stdin; mai scriverli in `vercel.json`, in chat, nelle PR.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit + DB | `cron-route.test.ts` (su `runCron` con deps finte): i quattro casi di A1; `review_run` con fake model → `cron_runs.details_json` contiene `review.id`. | — |
| Unit + DB | `webhook-route.test.ts`: 401 (assente, sbagliato), 400, 202 con valore grezzo e con `Bearer`; `last_webhook` scritto con `status: 'upserted'`; id ignoto → `status: 'not_found'`. | — |
| Unit | `vercel-config.test.ts`: regione, schedule (esattamente `17 6 * * 6`), path, `deploymentEnabled.main`. | — |
| Unit | `env.test.ts` esteso: `CRON_SECRET` corto → errore. | — |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E verdi in locale.
- **C — Locale**: `curl -H "Authorization: Bearer $CRON_SECRET" localhost:3000/api/cron/review` →
  JSON con `outcome`; `curl -X POST -H "Authorization: $HEVY_WEBHOOK_SECRET" -d '{"workoutId":"<id
  reale dell'ultimo workout>"}' localhost:3000/api/hevy/webhook` → 202 e `last_webhook`.
- **D — Produzione (passi 1–6 di § 4.4, ciascuno dopo l'OK, esito atteso detto prima)**: Turso
  creato e migrato (`__drizzle_migrations` con 2 righe); sync iniziale (`workouts` = count);
  Vercel verde; `/login` → `/` con la password; cron manuale → riga `cron_runs`; webhook →
  `last_webhook`. Verifica via `turso db shell` / dashboard, mai «a occhio».
- **F — Giro guidato (owner, telefono, browser)**: (1) `https://hevy-coach.vercel.app` → login →
  Oggi con i dati veri; (2) `/progressi` con le sessioni reali; (3) salvare un allenamento su Hevy
  e vedere «Ultimo allenamento» cambiare entro un minuto (webhook). Il bug sarebbe: una pagina
  senza login, un 401 del cron con il segreto giusto, un webhook che non scrive.
- **G — Ripristino**: l'allenamento di prova cancellato su Hevy (l'evento `deleted` arriva al
  prossimo sync); `.env.turso-migrate` resta locale e ignorato.

Che cosa NON copre: push (07), il primo cron reale del sabato (misura di chiusura).

## 8. Messa in esercizio

È la spec stessa (§ 4.4). Misura di chiusura: **il primo sabato dopo il deploy** `cron_runs` ha una
riga creata fra le 06:17 e le 07:17 UTC con `outcome` ≠ `error`.

## 9. Documentazione

- `SETUP.md` (nuovo, completo); `README.md` § Deploy → SETUP.md; `CLAUDE.md` Latest + § Data &
  Integrations (Turso remoto, Vercel, cron, webhook) + § Not yet present (dominio personalizzato);
  `AGENTS.md` § Deploy; `WORKFLOW.md § 3` (produzione: come si legge il DB remoto, cosa non si fa
  mai in produzione); `doc/guide/deploy.md` (≤ 200 righe: i comandi, le env, come ruotare un
  segreto, come rifare il full sync remoto, cosa fare se Hevy cambia l'API).

## 10. Fuori ambito, dichiarato

- Push → **07**. Dominio personalizzato → CLAUDE.md § Not yet present. Preview deploy → T-19 →
  CLAUDE.md § Not yet present. Monitoraggio/alert su `cron_runs` `error` → la notifica push di
  `failed` (07) copre il caso della revisione; un errore di sync senza revisione → CLAUDE.md § Not
  yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **M** di codice, **L** di operazioni.

```
Ciao, in questa sessione implemento la spec
doc/v1/06-messa-in-esercizio-vercel-turso-cron-webhook.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/sync.md, doc/guide/revisione.md, doc/guide/app.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 06 INTERA
- Verifica con un grep i contratti che usa (syncWorkouts, syncOneWorkout, runReview,
  shouldRunScheduledReview, sendReviewNotification, getDb, getHevyClient, getReviewModel,
  isPublicPath, setSyncValue)

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/06-deploy. Crea SESSION_NOTES.md.

Ordine: (1) route, run-cron, handle-webhook, vercel.json, env e i test; (2) SETUP.md e la
documentazione; (3) § 7: locale; poi i passi 1–6 di § 4.4 UNO PER MESSAGGIO, ciascuno dopo
il mio OK, con l'esito atteso detto prima e la verifica sul database remoto, mai a occhio. I
segreti li incollo io nei prompt dei comandi: non me li chiedere in chat e non scriverli in
nessun file tracciato. Se un punto non è coperto, fermati e chiedimelo con AskUserQuestion.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo) e per il merge su main, che
è il deploy.
```
