# 05a — Web app: accesso, shell, design e pagina Progressi

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 36 rilievi, 2 decisioni
> (D-54 e, condivisa, D-37). Pronta per l'implementazione dopo la 04.
> **Branch**: `feature/05a-app-progressi` da `main`. **Dipende da**: 01–04 (DB, sync, aggregazione,
> stati, storia, scale, `targetText`, repository).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-11, D-13, D-17, D-21, D-35, D-36, D-37,
> D-42, D-43, D-54; scelte T-4, T-12, T-13, T-14, T-20, T-21; contratti C-14 (lo crea), C-18 (lo
> crea), C-17 (aggiunge le due funzioni del sabato); usa C-1, C-3, C-5, C-7, C-8, C-9, C-11, C-15.

## 1. Il problema oggi (prove)

1. Tutto gira da CLI (02–04); non esiste una pagina. `src/app/page.tsx` è il segnaposto della 01.
2. L'app conterrà due chiavi, scriverà su Hevy e spenderà credito API (D-13): ogni pagina e ogni
   route devono stare dietro una sessione, tranne `/login`, il cron, il webhook e gli asset della
   PWA. Next 16: il file si chiama `proxy.ts`, gira solo su Node, e la doc
   (`https://nextjs.org/docs/app/api-reference/file-conventions/proxy`, letta il 2026-10-23) chiede di
   **verificare l'autorizzazione anche dentro ogni route handler e server function**, perché un
   matcher può saltare un percorso.
3. I dati per i progressi esistono (`aggregateWindow`, C-9) ma solo per la finestra della revisione;
   la pagina vuole una finestra più lunga per i grafici (T-20).
4. Non esistono DESIGN.md e PRODUCT.md (D-11): nascono qui con la skill impeccable; l'owner vuole una
   UI «wow», non un cruscotto grigio.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `/login` accetta `APP_PASSWORD` (confronto a tempo costante), imposta il cookie `hc_session` (JWT HS256 firmato con `SESSION_SECRET`, 365 giorni, `httpOnly`, `secure` in produzione (T-13), `sameSite: 'lax'`, `path: '/'`) e reindirizza a `next` (solo percorsi relativi, `safeNext`) o `/`; una password sbagliata aspetta 1 s e mostra «Password errata» senza dire altro; `/login` visitata con una sessione valida reindirizza a `/`. | `session.test.ts` (token valido/scaduto/firma sbagliata; il sorgente contiene `timingSafeEqual(` e non `===` in `isPasswordCorrect`); E2E `login.spec.ts`: password sbagliata → messaggio, nessun cookie, risposta ≥ 900 ms; giusta → `/` e cookie presente. Mutazione: `given === expected` → il test sul sorgente va rosso; togliere `sleep` → la misura va rossa. |
| A2 | `proxy.ts` reindirizza a `/login?next=<path>` ogni pagina senza sessione valida e risponde 401 JSON a ogni `/api/*` senza sessione, tranne i percorsi pubblici di `isPublicPath` (C-14). | E2E `auth.spec.ts`: `/progressi` senza cookie → `/login?next=%2Fprogressi`; `POST /api/sync` senza cookie → 401. Mutazione: togliere `/api` dalla logica → il test 401 va rosso. |
| A3 | Ogni route handler chiama `sessionDenied()` e ogni Server Action `assertSession()` (C-14) come prima istruzione, tranne `src/app/api/cron/**`, `src/app/api/hevy/webhook/**` (06) e l'action `login`. | `src/app/guards.test.ts` (§ 6), che legge i sorgenti. Mutazione: togliere il guard da `sync/route.ts` → rosso; spostarlo dopo `syncWorkouts` → rosso. |
| A4 | Shell: layout con barra di navigazione in basso (Oggi · Progressi · Revisioni · Impostazioni), titolo di pagina, stati vuoti con un testo e un'azione; funziona a 390 px senza scroll orizzontale; tema scuro di default; `<title>` «Hevy Coach» (T-14). | E2E `shell.spec.ts` a 390 e 1280: `scrollWidth <= clientWidth` e axe senza violazioni `serious`/`critical` su `/login`, `/`, `/progressi`, `/revisioni`, `/impostazioni`. Mutazione: una tabella con `min-width: 600px` → rosso a 390. |
| A5 | `/` (Oggi) mostra quattro dati: ultimo sync (data e modalità, da `sync_state`), prossima revisione automatica (§ 4.3), proposte da guardare (N = `countActionableProposals`, D-43: «Nessuna proposta da guardare» / «N proposte da guardare»; la 05b riempie la sezione), ultimo allenamento (titolo, data; `getLatestWorkout`). | E2E `oggi.spec.ts` con il seed: «Mai sincronizzato», «Al prossimo cron (sabato <nextCronSaturday(now)>)», «Nessuna proposta da guardare», titolo e data del workout del seed di −6 giorni. Mutazione: contare anche le `insufficient_data` → il test della 05b va rosso; qui: mostrare `last_full_sync_at` al posto di `last_sync_at` → rosso. |
| A6 | `/progressi`: per ogni routine (ordine Lun→Ven) una scheda per slot `reviewed` con: titolo variante, posizione nella scala («3 di 6»), target (`targetText`), ultime 5 sessioni (data, valori per set) e un grafico Recharts (serie = valore medio per sessione; linea di riferimento = massimo del target corrente; punti colorati se `allAtTop` rispetto al target **di quel giorno**; una linea verticale «→ <nuova variante>» a ogni cambio di variante, D-54), finestra 56 giorni (T-20), tutte le sessioni dello slot comprese quelle di varianti precedenti; slot senza sessioni → stato vuoto «Nessuna sessione nelle ultime 8 settimane» con il link «Sincronizza ora» → `/impostazioni`. | E2E `progressi.spec.ts` con seed: 27 schede, la scheda del push-up ha 3 punti (`circle` nel SVG) e la scheda vuota ha il testo e il link. Unit: `src/lib/progress/view-model.test.ts` (§ 4.4). Mutazione: `dot={false}` → rosso. |
| A7 | `/impostazioni`: «Sincronizza ora» (POST `/api/sync` → `syncWorkouts`), esito inline (`+${upserted} allenamenti`, più ` · −${deleted}` se > 0; 401 → `/login?next=/impostazioni`; altro errore → «Sincronizzazione non riuscita: riprova»), bottone disabilitato mentre aspetta; «Esci» cancella il cookie. Le sezioni push e revisione manuale arrivano con 07 e 05b. | E2E `impostazioni.spec.ts`: click → «+1 allenamenti» (il fake Hevy, § 4.5); logout → `/login`. Mutazione: mostrare `fetched` → rosso. |
| A8 | DESIGN.md e PRODUCT.md esistono, scritti con impeccable sul codice reale; nessun colore letterale nei componenti (solo token). | `src/app/guards.test.ts` verifica che `src/components/**` e `src/app/**` (tranne `globals.css`) non contengano `/#[0-9a-fA-F]{3,8}\b|rgba?\(|hsla?\(|oklch\(/`. Mutazione: `stroke="#fff"` in un grafico → rosso. DEVELOPMENT_GUIDELINES § Design System punta già a DESIGN.md: nessuna modifica. |
| A9 | Cancelli: `typecheck`, `lint`, `test`, `build`, `test:e2e` verdi. | Collaudo. |

## 3. Decisioni

- **D-11** (DESIGN/PRODUCT qui), **D-13** (auth), **D-17** (Recharts), **D-21** (UI italiana),
  **D-35**, **D-36** (le pagine importano `@/lib/env` solo dove servono le env; il proxy legge
  `process.env`), **D-37** (in esercizio = PR mergiata + collaudo annotato), **D-42** (le sessioni
  pre-piano compaiono in Progressi), **D-43** (N esclude le `insufficient_data`), **D-54** (cambi di
  variante nel grafico). **T-4** (precisata qui: `(app)/layout.tsx` ha `export const dynamic =
  'force-dynamic'`, perché senza `cacheComponents` Next 16 prerenderizza alla build una pagina che
  non chiama API dinamiche, e Oggi e Progressi mostrerebbero per sempre i dati del deploy), **T-12**
  (route e pagine), **T-13** (cookie; precisata qui: `secure: process.env.NODE_ENV === 'production'`,
  perché il dev e l'E2E girano su `http://localhost`), **T-14** (nome, tema scuro).
- **T-20 (adottata qui, ribaltabile)**: i grafici dei progressi usano una finestra di **56 giorni**
  (8 settimane); l'aggregazione è la stessa `aggregateWindow` con `window` diversa, e la pagina
  legge `allSessions` (04 C-9), non `sessions`.
- **T-21 (adottata qui, ribaltabile)**: i servizi esterni passano da un container
  (`src/lib/container.ts`, C-18): `getDb()`, `getHevyClient()`, `getReviewModel()`; con
  `E2E_FAKE_SERVICES=1` restituiscono fake in memoria (`tests/e2e/fakes/`), così Playwright non
  tocca Hevy né Anthropic. Il container ignora la variabile quando `process.env.VERCEL === '1'`: in
  produzione i fake non si accendono per errore.
- **La direzione visiva** (palette, tipografia, «mondo») la sceglie l'owner nella sessione, con
  il flusso della skill impeccable: è l'unico punto delegato alla sessione di implementazione, ed è
  dichiarato qui. Vincoli non negoziabili per impeccable: mobile-first (390 px), tema scuro di
  default, contrasto AA, testi italiani, nessuna animazione che blocchi l'interazione,
  `prefers-reduced-motion` rispettato, i grafici seguono la skill `dataviz` (token, `role="img"`,
  tabella `sr-only`, `isAnimationActive={false}`).
- Adottate qui, ribaltabili: (a) `jose@6` per il JWT; (b) il login è un form con Server Action
  (`src/app/login/actions.ts`); (c) `/api/sync` è un route handler `POST` che risponde
  `SyncResult`; (d) Playwright con un server dedicato su `:3100`, DB `file:e2e.db` ricreato dal
  seed a ogni corsa, `E2E_FAKE_SERVICES=1`, `APP_PASSWORD=fenicottero-2500`; (e) le pagine (Server
  Component) chiamano **una** funzione di caricamento in `src/lib/` (`loadProgress`, `loadToday`),
  che legge i repository e costruisce il view model: `src/app` resta sottile (AGENTS.md § 2.1); i
  Client Component solo per grafici e bottoni; (f) nessuna i18n: stringhe italiane nei componenti
  (`src/lib/ui/strings.ts` per quelle riusate); (g) i percorsi pubblici oltre D-13 sono gli asset
  della PWA (`/manifest.webmanifest`, `/sw.js`, `/swe-worker-`, `/icons/`, `/offline`,
  `/favicon.ico`, `/apple-icon`, `/icon`), già elencati qui per la 07; (h) il proxy risponde ai `/api/*` senza
  sessione con `401 { error: 'unauthorized' }`; (i) `/api/sync` ha `maxDuration = 120`; (j) il
  progetto E2E `mobile` usa Chromium con il viewport di un telefono, non WebKit: il telefono vero si
  prova nel collaudo della 06 e della 07; (k) `scalePosition` nella UI è «3 di 6» (nel prompt della
  04 è «3/6»).

## 4. La soluzione

### 4.1 Dipendenze

```bash
pnpm add jose@6.2.13 recharts@3.10.1 lucide-react@1.61.0
pnpm add -D @playwright/test@1.64.1 @axe-core/playwright@4.14.0 cross-env@10.1.0
pnpm exec playwright install chromium
```

(Versioni da `npm view` del 2026-10-24, già in T-2; `server-only` c'è dalla 01.) `.gitignore`
aggiunge `e2e.db*`, `e2e-fake-hevy.jsonl`, `playwright/.auth/`, `test-results/`, `playwright-report/`.

### 4.2 Sessione — `src/lib/auth/session.ts` (C-14) e `src/proxy.ts`

```ts
import { createHash, timingSafeEqual } from 'node:crypto';
import { SignJWT, jwtVerify } from 'jose';

export const SESSION_COOKIE = 'hc_session';
export const SESSION_DAYS = 365;
/** An entry ending with '/' or '-' is a prefix; any other entry matches exactly or followed by '/' or '.'. */
const PUBLIC_PATHS = ['/login', '/api/cron/', '/api/hevy/webhook', '/manifest.webmanifest', '/sw.js', '/swe-worker-', '/icons/', '/offline', '/favicon.ico', '/apple-icon', '/icon'];

export function isPublicPath(pathname: string): boolean {
  return PUBLIC_PATHS.some((p) =>
    p.endsWith('/') || p.endsWith('-') ? pathname.startsWith(p) : pathname === p || pathname.startsWith(`${p}/`) || pathname.startsWith(`${p}.`),
  );
}

/** Only a same-origin path is a valid `next`: rejects '//host', '/\host' and control characters. */
export function safeNext(value: FormDataEntryValue | null): string {
  if (typeof value !== 'string' || !value.startsWith('/')) return '/';
  if (value[1] === '/' || value[1] === '\\') return '/';
  // biome-ignore lint/suspicious/noControlCharactersInRegex: that is the point
  if (/[\u0000-\u001f]/.test(value)) return '/';
  return value;
}

export async function createSessionToken(secret: string, now: number): Promise<string> {
  return new SignJWT({ sub: 'owner' })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt(Math.floor(now / 1000))
    .setExpirationTime(Math.floor(now / 1000) + SESSION_DAYS * 86400)
    .sign(new TextEncoder().encode(secret));
}

export async function verifySessionToken(token: string, secret: string, now: number): Promise<boolean> {
  try {
    const { payload } = await jwtVerify(token, new TextEncoder().encode(secret), { currentDate: new Date(now) });
    return payload.sub === 'owner';
  } catch {
    return false;
  }
}

/** Constant-time password check over fixed-length digests so the comparison never leaks length. */
export function isPasswordCorrect(given: string, expected: string): boolean {
  const a = createHash('sha256').update(given).digest();
  const b = createHash('sha256').update(expected).digest();
  return timingSafeEqual(a, b);
}
```

`src/lib/auth/require-session.ts` esporta due helper, un solo file, entrambi leggono `await
cookies()` e verificano con `verifySessionToken(token, env.SESSION_SECRET, Date.now())`:
`sessionDenied(): Promise<NextResponse | null>` per i route handler (`const denied = await
sessionDenied(); if (denied) return denied;`, risposta `NextResponse.json({ error: 'unauthorized' },
{ status: 401 })`) e `assertSession(): Promise<void>` per le Server Action (lancia `new
Error('Unauthorized')`; `src/app/(app)/error.tsx` lo mostra come «Sessione scaduta» con il link
«Accedi» → `/login`).

`src/proxy.ts`:

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { SESSION_COOKIE, isPublicPath, verifySessionToken } from '@/lib/auth/session';

export async function proxy(request: NextRequest) {
  const { pathname, search } = request.nextUrl;
  if (isPublicPath(pathname)) return NextResponse.next();
  const secret = process.env.SESSION_SECRET;
  // A missing secret must be loud, not an endless redirect to /login.
  if (!secret) return NextResponse.json({ error: 'SESSION_SECRET is not set' }, { status: 500 });
  const token = request.cookies.get(SESSION_COOKIE)?.value;
  const ok = token ? await verifySessionToken(token, secret, Date.now()) : false;
  if (ok) return NextResponse.next();
  if (pathname.startsWith('/api/')) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  const login = new URL('/login', request.url);
  login.searchParams.set('next', pathname + search);
  return NextResponse.redirect(login);
}

export const config = { matcher: ['/((?!_next/static|_next/image).*)'] };
```

`process.env.SESSION_SECRET` nel proxy, non `env` (il proxy gira fuori dal runtime di `server-only`);
`EnvSchema` aggiunge `APP_PASSWORD: z.string().min(12)`, `SESSION_SECRET: z.string().min(32)`;
`.env.example` li elenca con il comando `openssl rand -base64 32`; `vitest.config.ts` aggiunge i
valori finti (`APP_PASSWORD: 'test-password-12'`, `SESSION_SECRET: 'x'.repeat(32)`).

### 4.3 Pagine e componenti

- Prima voce: `git rm src/app/page.tsx` (il segnaposto della 01): la pagina Oggi è
  `src/app/(app)/page.tsx` e due pagine per `/` rompono la build.
- `src/app/layout.tsx`: `lang="it"`, `metadata.title` «Hevy Coach» (T-14), font (scelto con
  impeccable, caricato da file locale in `src/app/fonts/`, mai `next/font/google`: il build non
  tocca la rete), `<meta name="color-scheme">`, `viewport` con `themeColor`.
- `src/app/login/page.tsx` + `actions.ts`: la pagina (Server Component) legge il cookie e, se la
  sessione è valida, fa `redirect('/')`; altrimenti rende il form con `<input type="hidden"
  name="next" value={safeNext(searchParams.next)}>`. `login(prev, formData)`: password →
  `isPasswordCorrect`; fallimento → `await sleep(1000)` e `{ error: 'Password errata' }`; successo →
  cookie e `redirect(safeNext(formData.get('next')))`.
- `src/app/(app)/layout.tsx`: `export const dynamic = 'force-dynamic'` (T-4); shell con
  `<BottomNav />` (`src/components/layout/bottom-nav.tsx`, Client Component per l'attivo) e il
  contenitore; ogni pagina rende il proprio `<PageHeader title="Oggi" />` (il layout non conosce il
  percorso). `src/app/(app)/error.tsx`: «Qualcosa è andato storto» con «Riprova»; se
  `error.message === 'Unauthorized'` → «Sessione scaduta» con «Accedi» → `/login`.
- `src/app/(app)/page.tsx` (Oggi): Server Component che chiama `loadToday({ db: getDb(), now:
  Date.now() })` (`src/lib/progress/load-today.ts`), che legge: `getSyncValue(db, 'last_sync_at')` e
  `getSyncValue(db, 'last_sync_mode')` (03: «Ultimo sync: <data e ora di Roma> · completo |
  incrementale»; assente → «Mai sincronizzato» con il link «Sincronizza ora» → `/impostazioni`);
  `scheduledReviewReference(db)` (04) e `shouldRunScheduledReview(db, now)` (testo in § 4.4);
  `countActionableProposals(db)` (04, D-43: 0 → «Nessuna proposta da guardare», 1 → «1 proposta da
  guardare», N → «N proposte da guardare», link alla sezione `<OpenProposals />` che qui è un
  segnaposto con lo stesso testo e la 05b riempie); `getLatestWorkout(db)` (03: null → «Nessun
  allenamento registrato»; altrimenti titolo e `romeDate`).
- `src/app/(app)/progressi/page.tsx`: Server Component → `loadProgress({ db: getDb(), now:
  Date.now() })` (§ 4.4) → `<RoutineSection>` × 5 → `<SlotCard>` con `<SessionChart>` (Client,
  Recharts via `src/components/charts/recharts.ts`, l'unica porta a Recharts) e `<SessionTable>`
  (ultime 5 sessioni).
- `src/app/(app)/impostazioni/page.tsx`: `<SyncNowButton />` (Client: `fetch('/api/sync', { method:
  'POST' })`, disabilitato mentre aspetta; 2xx → `+${upserted} allenamenti` e ` · −${deleted}` se >
  0; 401 → `router.push('/login?next=/impostazioni')`; altro → «Sincronizzazione non riuscita:
  riprova»), `<LogoutButton />` (Server Action `logout()` in `src/app/(app)/impostazioni/actions.ts`,
  dopo `assertSession()` cancella il cookie e `redirect('/login')`).
- `src/app/api/sync/route.ts`: `POST` → `sessionDenied()` → `syncWorkouts({ db: getDb(), hevy:
  getHevyClient(), now: Date.now() })` → JSON `SyncResult`; `HevyApiError` → `502 { error: 'hevy:
  <status>' }`; `export const maxDuration = 120`.
- Segnaposto `/revisioni` (lista vuota con testo «La prima revisione arriva sabato» e il link «Vai a
  Oggi» → `/`; la 05b lo sostituisce con «Esegui revisione ora») così la nav non ha link morti.

### 4.4 View model e caricamento — `src/lib/progress/`

```ts
export interface ProgressSession { date: string; values: number[]; mean: number; allAtTop: boolean }
export interface VariantChange { date: string; variantTitle: string }     // vertical ReferenceLine (D-54)
export interface SlotProgress {
  slotId: SlotId; routineKey: RoutineKey; routineTitle: string; variantTitle: string; scalePosition: string;
  targetText: string; sessions: ProgressSession[]; changes: VariantChange[]; referenceValue: number; unit: 'rip' | 's';
}
export function buildProgressViewModel(result: AggregateResult, states: SlotState[], history: SlotStateHistoryRow[]): SlotProgress[]
export async function loadProgress({ db, now }: { db: Db; now: number }): Promise<SlotProgress[]>
```

`buildProgressViewModel` (puro, testato): una voce per ogni slot `reviewed` di `PLAN`, nell'ordine
di `PLAN` (routine, poi slot); `sessions` = `summary.allSessions` (04: tutte le sessioni della
finestra, varianti precedenti comprese, D-54), con `mean` = media dei `values` arrotondata a 1
decimale (`Math.round(x * 10) / 10`) e `allAtTop` così come la 04 lo calcola sul target del giorno;
`changes` = una voce per ogni riga di storia dello slot con `validFrom` dentro la finestra e
`source = 'apply'` con cambio di `variantId` (data `romeDate(validFrom)`, titolo della nuova
variante); `referenceValue` = `max` per `reps`, `threshold` per `max2`, `maxSeconds` per `hold` del
target **corrente**; `unit` = `'s'` per `hold`, altrimenti `'rip'`; `scalePosition` = `${indice+1}
di ${lunghezza}`. Uno slot senza riga in `states` (DB mai sincronizzato) usa `startVariantId` e
`initialTarget` del piano: prima del primo `pnpm hevy:sync` Progressi mostra il piano con 27 schede
vuote. Gli `unresolved` dell'aggregazione non compaiono in Progressi.

`loadProgress`: `window = { fromMs: romeDayStartMinusDays(now, 56), toMs: now }`; `sets =
listSetsBetween(db, window)`, `states = listSlotStates(db)`, `history = listSlotStateHistory(db)`,
`templateMap = getTemplateMap(db)`, `routineKeys = routineKeyByHevyId(db)`;
`buildProgressViewModel(aggregateWindow({ sets, states, history, templateMap, routineKeys, window }),
states, history)`. La pagina chiama solo `loadProgress`.

**Prossima revisione** (Oggi): `time.ts` aggiunge `nextCronSaturday(ms: number): string` (la data di
Roma `YYYY-MM-DD` del primo sabato il cui istante di cron, 06:17 UTC, è > `ms`) e
`firstSaturdayOnOrAfter(dateRome: string): string` (C-17). Testo: riferimento null → «Nessuna
revisione prevista: sincronizza prima» con il link a `/impostazioni`; `shouldRunScheduledReview`
vero → «Al prossimo cron (sabato <d>)» con `d = nextCronSaturday(now)`; altrimenti «Prevista sabato
<d>» con `d = max(nextCronSaturday(now), firstSaturdayOnOrAfter(romeDate(ref + 13 giorni)))`.

### 4.5 Container — `src/lib/container.ts` (C-18)

```ts
export function getDb(): Db
export function getHevyClient(): HevyClient          // fake when E2E_FAKE_SERVICES === '1' and VERCEL !== '1'
export function getReviewModel(): ReviewModel        // idem
```

`tests/e2e/fakes/fake-hevy.ts` (stato su `globalThis`, così sopravvive ai ricaricamenti del dev
server): `listWorkouts(page)` → `page_count: 1` e la sola fixture `workout.json` con
`start_time`/`end_time` spostati a `2020-01-06T18:00:00Z`; `listWorkoutEvents` → un solo evento
`updated` di quel workout (`updated_at` `2020-01-06T19:00:00Z`); `listRoutines`/`listRoutineFolders`
→ vuoti (la 05b li riempie); `getRoutine`/`updateRoutine` → lanciano `not used` (la 05b li
implementa). Ogni chiamata di scrittura si appende come riga JSON a `e2e-fake-hevy.jsonl` nella
root (gitignored): Playwright gira in un altro processo e legge quel file (05b). Nessun
`addWorkout()`. `tests/e2e/fakes/fake-model.ts`: legge gli `slotId` di `exercises` dallo user
prompt (`JSON.parse`) e risponde `keep` per ognuno con la motivazione fissa «Fixture E2E» e
`summary` «Fixture E2E». Le route e le pagine usano SOLO il container (`guards.test.ts`).

### 4.6 Design — impeccable

Nella sessione: caricare la skill `impeccable`, far scegliere la direzione all'owner (tre
proposte a confronto, come nelle altre app), costruire la shell e le tre pagine in quella
direzione, poi far scrivere `DESIGN.md` e `PRODUCT.md` dalla skill (mai a mano), e aggiornare
`DEVELOPMENT_GUIDELINES.md § Design System` con il rimando. PRODUCT.md dice: utente singolo,
self-hosted, «un coach che propone, non decide», lingua italiana, uso dal telefono la sera.

### 4.7 E2E — Playwright

`playwright.config.ts`: `baseURL: http://localhost:3100`, `webServer: { command: 'pnpm e2e:server',
port: 3100, reuseExistingServer: false }`, `workers: 1`, `fullyParallel: false` (i progetti
condividono `e2e.db`); progetti: `setup` (`testMatch: /auth\.setup\.ts/`), `mobile` = `{
...devices['Pixel 7'], viewport: { width: 390, height: 844 } }` e `desktop` = `{
...devices['Desktop Chrome'], viewport: { width: 1280, height: 800 } }`, entrambi con `dependencies:
['setup']` e `storageState: 'playwright/.auth/owner.json'`; `login.spec.ts` e `auth.spec.ts` fanno
`test.use({ storageState: { cookies: [], origins: [] } })`. Script:
`"e2e:server": "cross-env E2E_FAKE_SERVICES=1 TURSO_DATABASE_URL=file:e2e.db APP_PASSWORD=fenicottero-2500 SESSION_SECRET=<32+ chars fissi> HEVY_API_KEY=e2e ANTHROPIC_API_KEY=e2e PORT=3100 next dev -p 3100"`
(le variabili passate così vincono su `.env.local`, che `next dev` non sovrascrive mai),
`"e2e:seed": "cross-env TURSO_DATABASE_URL=file:e2e.db tsx tests/e2e/seed.ts"`,
`"test:e2e": "pnpm e2e:seed && playwright test"`.

`tests/e2e/seed.ts`: all'avvio fallisce se `TURSO_DATABASE_URL` non finisce con `e2e.db`; cancella
`e2e.db*` (se il file è bloccato, su Windows, esce con «close the server on :3100 and retry»); apre
un client **proprio** con `createClient({ url: 'file:e2e.db' })` e `drizzle(client, { schema })`
(mai `src/lib/db/client.ts`: un errore di env scriverebbe in `local.db`), applica le migrazioni e
inserisce, con i repository: `upsertHevyRoutine` × 5 (`hevyRoutineId` `e2e-routine-mon` …
`e2e-routine-fri`, titoli D-25); `upsertTemplateMap` per `allVariantsInUse(initialSlotStates(t0))`
con gli id di catalogo di 02 § 4.2 e id custom `e2e-custom-<variantId>`;
`ensureInitialSlotStates(db, initialSlotStates(t0), t0)` con `t0 = romeDayStartMinusDays(now, 60)`
(stato e storia); `setSyncValue(db, 'plan_uploaded_at', String(t0), now)`;
`setSyncValue(db, 'workouts_cursor', '2020-01-01T00:00:00Z', now)` (così ogni sync è incrementale e
A7 dà sempre «+1 allenamenti»); 6 workout con `upsertWorkout`, tutti alle 19:00 di Roma:
`routine_id = 'e2e-routine-mon'` a −21, −14, −7 giorni con Push Up (`392887AA`) 4×[12,12,11,10] /
[13,13,12,12] / [15,15,15,15] e Inverted Row (`425805F4`) 4×[10,10,10,10]; `routine_id =
'e2e-routine-wed'` a −20, −13, −6 giorni con Plank (`C6C9B8A0`) 2×[40,40] s e le tre mobilità.
Numeri inventati e tondi, nessun id reale (WORKFLOW § 1.6).

`tests/e2e/auth.setup.ts` fa il login una volta e salva `playwright/.auth/owner.json` (gitignored).
Spec: `login.spec.ts`, `auth.spec.ts`, `shell.spec.ts` (overflow + axe su `/login`, `/`,
`/progressi`, `/revisioni`, `/impostazioni`), `oggi.spec.ts`, `progressi.spec.ts`,
`impostazioni.spec.ts`, `screenshots.spec.ts` (tag `@screenshots`, § 9).

## 5. Vincoli e trappole

- **AGENTS.md § 7 Auth and app** (stub già presente: si completa, senza aprire sezioni nuove) — qui
  vuol dire che: il proxy è la prima barriera, non l'unica: ogni route e action chiama il guard
  (test A3); `isPublicPath` è l'**unico** elenco dei percorsi pubblici e contiene già quelli della
  PWA (07): un percorso pubblico nuovo si aggiunge lì, mai nel matcher; token di DESIGN.md, nessun
  colore letterale; grafici solo via `src/components/charts/recharts.ts` (l'unica porta a Recharts,
  come net-worth-tracker); stringhe italiane; stati vuoti sempre con un'azione; le pagine chiamano
  una funzione `load*` e non i repository.
- **AGENTS.md § 9 Testing** — E2E: fake dietro il container, seed con client proprio,
  `storageState`, `workers: 1`, `e2e-fake-hevy.jsonl` come canale fra server e test.
- `cookies()` è asincrona in Next 16 (`await cookies()`).
- `next dev` con Turbopack e `next build --webpack`: il dev server E2E usa `next dev` (veloce); la 07
  aggiunge un progetto E2E su build di produzione per il service worker.
- Recharts in jsdom non monta `ResponsiveContainer`: i test unit coprono il view model, non il
  grafico; il grafico lo copre l'E2E (conta i `circle`).
- Il seed E2E usa **numeri inventati e tondi** (WORKFLOW § 1.6) e nessun id reale.
- `safeNext`: mai `redirect(next)` grezzo (open redirect).

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `src/lib/auth/session.test.ts`: token creato con `now = Date.UTC(2000, 0, 1)` e verificato con lo stesso `now` → true (scaduto per l'orologio reale: prova che conta `currentDate`); verificato a `now + 366 giorni` → false; segreto diverso → false; `isPasswordCorrect` vero/falso e lunghezze diverse; il sorgente di `session.ts` contiene `timingSafeEqual(` e `isPasswordCorrect` non contiene `===`; `isPublicPath` per ogni voce + `/api/cron/review?x=1` → true, `/icon.png` → true, `/swe-worker-abc123.js` → true, `/login-x` → false, `/iconografia` → false, `/progressi` → false; `safeNext`: `'//evil.com'`, `'/\\evil.com'`, `'https://evil.com'`, `null` → `'/'`; `'/progressi?x=1'` → invariato. | Togliere `currentDate` da `jwtVerify` → il primo caso va rosso; `given === expected` → rosso; `startsWith` nudo → `/login-x` va rosso. |
| Unit | `src/lib/time.test.ts` (aggiunta): `nextCronSaturday`: sabato 2026-10-24T05:00Z → `2026-10-24`; sabato 2026-10-24T07:00Z → `2026-10-31`; venerdì 2026-10-23T22:30Z (già sabato a Roma) → `2026-10-24`; `firstSaturdayOnOrAfter('2026-10-24')` → `2026-10-24`, `('2026-10-25')` → `2026-10-31`. | Confronto `>=` sul giorno senza l'ora → il secondo caso va rosso. |
| Unit | `src/lib/progress/view-model.test.ts`: ordine PLAN, `scalePosition`, `referenceValue` per genere, `mean` arrotondata a 1 decimale, slot senza sessioni presente con `sessions: []`, slot senza stato → variante e target del piano, `changes` da una riga di storia `apply` nella finestra, sessioni di una variante precedente presenti con il loro `allAtTop`. | Ordinare per `slotId` invece che per PLAN → rosso; `referenceValue = min` per `reps` → rosso; usare `sessions` al posto di `allSessions` → il caso della variante precedente va rosso. |
| Unit | `src/app/guards.test.ts`: per ogni `src/app/api/**/route.ts`, ogni handler esportato (`GET|POST|PUT|DELETE`) ha come prima istruzione `const denied = await sessionDenied();`, tranne `api/cron/**` e `api/hevy/webhook/**`; per ogni `src/app/**/actions.ts` ogni `export async function` ha come prima istruzione `await assertSession();`, tranne `login` in `src/app/login/actions.ts`; nessun `createHevyClient(` né `createAnthropicReviewModel(` in `src/app/**` e `src/components/**`; nessun colore letterale (A8). | Togliere il guard da `sync/route.ts` → rosso; spostarlo dopo `syncWorkouts` → rosso; `stroke="#fff"` → rosso. |
| E2E | i sette spec di § 4.7 su `mobile` e `desktop`; axe senza violazioni `serious`/`critical`; `login.spec.ts` misura la risposta alla password sbagliata ≥ 900 ms. | Togliere `'/login'` da `isPublicPath` → `login.spec` rosso; `dot={false}` nel grafico → `progressi.spec` rosso; togliere `sleep` → `login.spec` rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli verdi; `pnpm review:show` (nessuna chiamata esterna) e, dopo l'OK,
  `pnpm hevy:sync` (GET reali su Hevy `workouts/events`, sola lettura, gratis, scrive `local.db`)
  funzionano ancora con le due variabili nuove in `.env.local`. `pnpm review:run` **non** si lancia
  (costa e chiude le proposte aperte). CLI e app non condividono il container: gli script usano
  direttamente `createHevyClient`; il container è dell'app.
- **C — Comportamento nuovo**: `pnpm test:e2e` verde (setup + mobile + desktop), nessuna chiamata
  reale (fake).
- **D — Sotto la UI (dopo l'OK)**: `curl -i localhost:3000/api/sync -X POST` → 401 (nessuna
  chiamata esterna); con il cookie (preso dal browser) → JSON `SyncResult` (GET reale
  `workouts/events` su Hevy, sola lettura, gratis, aggiorna `local.db`).
- **F — Giro guidato (owner, dev server `pnpm dev` con `.env.local` reale, DB `local.db`)**: uno
  script usa e getta fa il login su :3000 con `APP_PASSWORD` di `.env.local` e salva
  `playwright/.auth/dev-owner.json` (gitignored; il cookie dell'E2E è firmato con un altro segreto e
  qui non vale); prima di aprire la pagina all'owner lo stesso script legge da `local.db`
  `sync_state`, `SELECT title, start_time FROM workouts ORDER BY start_time DESC LIMIT 1` e
  `SELECT COUNT(*) FROM proposals WHERE status='open' AND insufficient_data = 0` e confronta i
  quattro valori con Oggi. Passi: (1) `/`: i quattro dati; (2) `/progressi` a 390 px: le schede del
  lunedì, il grafico del push-up con i punti delle sessioni reali; (3) `/impostazioni`: «Sincronizza
  ora» → «+0 allenamenti» (o +N; GET reale su Hevy, sola lettura, gratis, dopo l'OK); (4)
  l'estetica a 390 px: è la direzione scelta. Il messaggio di password errata è coperto da
  `login.spec.ts` e non si rifà a mano. Il bug sarebbe: un overflow orizzontale, un grafico vuoto
  con sessioni presenti, un testo inglese.
- **G — Ripristino**: `e2e.db*` e `e2e-fake-hevy.jsonl` cancellati; `playwright/.auth` ignorato da
  git; lo script usa e getta cancellato.

Che cosa NON copre: proposte e revisioni in UI (05b), telefono reale (06/07).

## 8. Messa in esercizio

- Env (Development): `APP_PASSWORD`, `SESSION_SECRET`. DDL: nessuno.
- Misura di chiusura: `pnpm test:e2e` verde e giro guidato annotato. In esercizio = PR mergiata su
  `main` + collaudo annotato in CLAUDE.md (D-37); la 06 rifà il giro in produzione.

## 9. Documentazione

- `DESIGN.md`, `PRODUCT.md` (impeccable); `DEVELOPMENT_GUIDELINES.md § Design System` punta già a
  DESIGN.md (nessuna modifica).
- `CLAUDE.md`: Latest; § Current Status → Stack: la versione di recharts; § Key Features (Accesso,
  Oggi, Progressi, Impostazioni); § Testing (E2E); § Not yet present: togliere la riga DESIGN/PRODUCT
  e aggiungere «Login rate limiting beyond the 1 s delay» (se non c'è già); § Design Context →
  «DESIGN.md and PRODUCT.md (impeccable, 05a) are the source; never edit by hand»; § Key Files
  aggiunge DESIGN.md e PRODUCT.md.
- `AGENTS.md`: § 7 Auth and app (da stub a regole), § 9 Testing (E2E); nessuna sezione nuova.
- `doc/guide/app.md` (≤ 300 righe, la 05b la estende: restare intorno a 200): shell, pagine,
  `load*`, container, fake E2E, seed, prossima revisione.
- `WORKFLOW.md § 3`: `e2e:seed`, `e2e:server`, `test:e2e` e «come si apre l'app già autenticati»
  diventano veri.
- `README.md` (inglese): `## Running locally` (`pnpm install`, `.env.local` da `.env.example`, `pnpm
  db:migrate`, `pnpm hevy:sync`, `pnpm dev`) e due screenshot `doc/screenshots/oggi.png` e
  `progressi.png` a 390×844 salvati da `tests/e2e/screenshots.spec.ts` (solo con `pnpm test:e2e --
  --grep @screenshots`) dal seed, mai dai dati reali.
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/app.md` (≤ 300).

## 10. Fuori ambito, dichiarato

- Proposte, revisioni, «Applica», «Esegui revisione ora» → **05b**.
- PWA, manifest, push, `/offline` → **07** (i percorsi sono già in `isPublicPath`).
- Deploy → **06**.
- Rate limit del login oltre l'attesa di 1 s → CLAUDE.md § Not yet present (riga in § 9).
- Telefono vero (iPhone/Android): collaudo della 06 e della 07.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **XL** (auth + shell + 3 pagine + design + E2E).

```
Ciao, in questa sessione implemento la spec doc/v1/05a-web-app-accesso-shell-e-progressi.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/piano.md, doc/guide/sync.md, doc/guide/revisione.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 05a INTERA
- Verifica con un grep i contratti di 01–04 che la spec usa (aggregateWindow e AggregateInput,
  allSessions, listSetsBetween, listSlotStates, listSlotStateHistory, getTemplateMap,
  routineKeyByHevyId, getSyncValue, setSyncValue, getLatestWorkout, upsertWorkout,
  upsertHevyRoutine, upsertTemplateMap, ensureInitialSlotStates, countActionableProposals,
  shouldRunScheduledReview, scheduledReviewReference, syncWorkouts, targetText, findVariant,
  romeDate, romeDayStartMinusDays, createHevyClient, createAnthropicReviewModel, ReviewModel,
  HevyApiError, PLAN, SCALES, initialSlotStates, allVariantsInUse)
- Carica le skill impeccable e dataviz PRIMA di scrivere un componente

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/05a-app-progressi. Crea SESSION_NOTES.md.

Ordine di lavoro: (1) `git rm src/app/page.tsx`, poi sessione, proxy, login, container, guard,
`load*`, `time.ts` e i loro test; (2) `pnpm exec playwright install chromium`, il seed E2E e
Playwright; (3) con impeccable: tre direzioni visive a confronto, me le mostri e scelgo (è l'unico
punto delegato alla sessione); (4) shell e le tre pagine nella direzione scelta, grafici con
dataviz; (5) DESIGN.md e PRODUCT.md scritti dalla skill; (6) § 7 collaudo fase per fase con
l'esito atteso prima (le GET reali su Hevy dopo il mio OK; mai `review:run`), il giro guidato sul
dev server già autenticato; (7) § 9 documentazione. Se un punto non è coperto dalla spec, fermati
e chiedimelo con AskUserQuestion. Le etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
