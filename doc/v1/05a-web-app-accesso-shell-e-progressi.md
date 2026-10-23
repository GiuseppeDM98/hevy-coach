# 05a — Web app: accesso, shell, design e pagina Progressi

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 04.
> **Branch**: `feature/05a-app-progressi` da `main`. **Dipende da**: 01–04 (DB, sync, aggregazione,
> stati, scale).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-11, D-13, D-17, D-21, D-35; scelte T-12,
> T-13, T-14, T-20; contratti C-14 (lo crea), C-18 (lo crea).

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
| A1 | `/login` accetta `APP_PASSWORD` (confronto a tempo costante), imposta il cookie `hc_session` (JWT HS256 firmato con `SESSION_SECRET`, 365 giorni, `httpOnly`, `secure` in produzione, `sameSite: 'lax'`, `path: '/'`) e reindirizza a `next` (solo percorsi relativi) o `/`; una password sbagliata aspetta 1 s e mostra «Password errata» senza dire altro. | `session.test.ts` (token valido/scaduto/firma sbagliata); E2E `login.spec.ts`: password sbagliata → messaggio, nessun cookie; giusta → `/` e cookie presente. Mutazione: confronto `===` sulla stringa → il test unit sulla lunghezza diversa resta verde: la prova portante è il grep `timingSafeEqual` nel test. |
| A2 | `proxy.ts` reindirizza a `/login?next=<path>` ogni pagina senza sessione valida e risponde 401 JSON a ogni `/api/*` senza sessione, tranne i percorsi pubblici di `isPublicPath` (C-14). | E2E `auth.spec.ts`: `/progressi` senza cookie → `/login?next=%2Fprogressi`; `POST /api/sync` senza cookie → 401. Mutazione: togliere `/api` dalla logica → il test 401 va rosso. |
| A3 | Ogni route handler e server function chiama `requireSession()` (C-14) prima di fare altro. | `grep -L requireSession src/app/api/**/route.ts` restituisce solo `cron/review` e `hevy/webhook` (test in `tests/guards.test.ts` che legge i file). |
| A4 | Shell: layout con barra di navigazione in basso (Oggi · Progressi · Revisioni · Impostazioni), titolo di pagina, stati vuoti con un testo e un'azione; funziona a 390 px senza scroll orizzontale; tema scuro di default. | E2E `shell.spec.ts` a 390 e 1024: `scrollWidth <= clientWidth` su ogni pagina. |
| A5 | `/` (Oggi) mostra: ultimo sync (data, modalità), prossima revisione automatica (sabato + regola dei 13 giorni: «prevista il <data>» o «al prossimo cron»), numero di proposte aperte (link a `/`’s sezione proposte che la 05b riempie: qui un testo «Nessuna proposta aperta» / «N proposte aperte»), ultimo allenamento (titolo, data). | E2E con seed: i quattro numeri attesi. |
| A6 | `/progressi`: per ogni routine (ordine Lun→Ven) una scheda per slot `reviewed` con: titolo variante, posizione nella scala («3 di 6»), target (`targetText`), ultime sessioni (data, valori per set) e un grafico Recharts (serie = valore medio per sessione; linea di riferimento = massimo del target; punti colorati se `allAtTop`), finestra 56 giorni (T-20); slot senza sessioni → stato vuoto «Nessuna sessione nelle ultime 8 settimane». | E2E `progressi.spec.ts` con seed: 27 schede, una con 3 sessioni ha 3 punti (`circle` nel SVG) e la scheda vuota ha il testo. Unit: `progress-view-model.test.ts` (§ 4.4). |
| A7 | `/impostazioni`: «Sincronizza ora» (POST `/api/sync` → `syncWorkouts`), esito inline («+3 allenamenti»); «Esci» cancella il cookie. Le sezioni push e revisione manuale arrivano con 07 e 05b. | E2E: click → testo con il conteggio del fake Hevy; logout → `/login`. |
| A8 | DESIGN.md e PRODUCT.md esistono, scritti con impeccable sul codice reale; `DEVELOPMENT_GUIDELINES.md § Design System` punta a DESIGN.md; nessun colore letterale nei componenti (solo token). | Grep `#[0-9a-f]{6}` in `src/components` e `src/app` vuoto tranne `globals.css`. |
| A9 | Cancelli: `typecheck`, `lint`, `test`, `build`, `test:e2e` verdi. | Collaudo. |

## 3. Decisioni

- **D-11** (DESIGN/PRODUCT qui), **D-13** (auth), **D-17** (Recharts), **D-21** (UI italiana),
  **D-35**. **T-12** (route e pagine), **T-13** (cookie), **T-14** (nome, tema scuro).
- **T-20 (adottata qui, ribaltabile)**: i grafici dei progressi usano una finestra di **56 giorni**
  (8 settimane); l'aggregazione è la stessa `aggregateWindow` con `window` diversa.
- **T-21 (adottata qui, ribaltabile)**: i servizi esterni passano da un container
  (`src/lib/container.ts`, C-18): `getDb()`, `getHevyClient()`, `getReviewModel()`; con
  `E2E_FAKE_SERVICES=1` restituiscono fake in memoria (`tests/e2e/fakes/`), così Playwright non
  tocca Hevy né Anthropic. In produzione la variabile non esiste.
- **La direzione visiva** (palette, tipografia, «mondo») la sceglie l'owner nella sessione, con
  il flusso della skill impeccable: è l'unico punto delegato alla sessione di implementazione, ed è
  dichiarato qui. Vincoli non negoziabili per impeccable: mobile-first (390 px), tema scuro di
  default, contrasto AA, testi italiani, nessuna animazione che blocchi l'interazione,
  `prefers-reduced-motion` rispettato, i grafici seguono la skill `dataviz` (token, `role="img"`,
  tabella `sr-only`, `isAnimationActive={false}`).
- Adottate qui, ribaltabili: (a) `jose@6` per il JWT; (b) il login è un form con Server Action
  (`src/app/login/actions.ts`); (c) `/api/sync` è un route handler `POST` che risponde
  `SyncResult`; (d) Playwright con un server dedicato su `:3100`, DB `file:e2e.db` ricreato dal
  seed a ogni corsa, `E2E_FAKE_SERVICES=1`, `APP_PASSWORD=fenicottero-2500`; (e) i Server Component
  leggono dal DB direttamente (repository), i Client Component solo per grafici e bottoni;
  (f) nessuna i18n: stringhe italiane nei componenti (`src/lib/ui/strings.ts` per quelle riusate).

## 4. La soluzione

### 4.1 Dipendenze

```bash
pnpm add jose@6 recharts@3 lucide-react@1.61.0 server-only
pnpm add -D @playwright/test@1.64.1 @axe-core/playwright@4
```

(`recharts@3`: la versione esatta la fissa `pnpm add` il giorno dell'implementazione; si annota in
CLAUDE.md § Stack.)

### 4.2 Sessione — `src/lib/auth/session.ts` (C-14) e `src/proxy.ts`

```ts
import { SignJWT, jwtVerify } from 'jose';

export const SESSION_COOKIE = 'hc_session';
export const SESSION_DAYS = 365;
const PUBLIC_PREFIXES = ['/login', '/api/cron/', '/api/hevy/webhook', '/manifest.webmanifest', '/sw.js', '/icons/', '/offline', '/favicon.ico', '/apple-icon', '/icon'];

export function isPublicPath(pathname: string): boolean {
  return PUBLIC_PREFIXES.some((p) => pathname === p || pathname.startsWith(p));
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

`src/lib/auth/require-session.ts`: `requireSession(): Promise<void>` legge `cookies()`, verifica,
e lancia `new Response('Unauthorized', { status: 401 })`… **no**: in un route handler si fa
`const denied = await sessionDenied(); if (denied) return denied;` dove `sessionDenied()` ritorna
`NextResponse | null`; nelle Server Action si chiama `await assertSession()` che lancia un
`Error('Unauthorized')`. Due helper, un solo file.

`src/proxy.ts`:

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { SESSION_COOKIE, isPublicPath, verifySessionToken } from '@/lib/auth/session';

export async function proxy(request: NextRequest) {
  const { pathname, search } = request.nextUrl;
  if (isPublicPath(pathname)) return NextResponse.next();
  const token = request.cookies.get(SESSION_COOKIE)?.value;
  const ok = token ? await verifySessionToken(token, process.env.SESSION_SECRET ?? '', Date.now()) : false;
  if (ok) return NextResponse.next();
  if (pathname.startsWith('/api/')) return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  const login = new URL('/login', request.url);
  login.searchParams.set('next', pathname + search);
  return NextResponse.redirect(login);
}

export const config = { matcher: ['/((?!_next/static|_next/image).*)'] };
```

`process.env.SESSION_SECRET` nel proxy, non `env` (il proxy non importa `server-only`); `EnvSchema`
aggiunge `APP_PASSWORD: z.string().min(12)`, `SESSION_SECRET: z.string().min(32)`; `.env.example`
li elenca con il comando `openssl rand -base64 32`.

### 4.3 Pagine e componenti

- `src/app/layout.tsx`: `lang="it"`, font (scelto con impeccable), `<meta name="color-scheme">`,
  `viewport` con `themeColor`.
- `src/app/login/page.tsx` + `actions.ts` (`login(prev, formData)`: password → `isPasswordCorrect`;
  fallimento → `await sleep(1000)` e `{ error: 'Password errata' }`; successo → cookie e
  `redirect(safeNext(formData.get('next')))`, dove `safeNext` accetta solo stringhe che iniziano con
  `/` e non con `//`).
- `src/app/(app)/layout.tsx`: shell con `<BottomNav />` (`src/components/layout/bottom-nav.tsx`,
  Client Component per l'attivo), `<PageHeader title />`.
- `src/app/(app)/page.tsx` (Oggi): Server Component; legge `sync_state`, l'ultimo workout,
  `shouldRunScheduledReview` + `nextSaturday(now)` (helper in `time.ts`: prossimo sabato Europe/Rome;
  se la regola è già vera → «al prossimo cron (sabato <data>)», altrimenti «prevista il <primo
  sabato ≥ riferimento + 13 giorni>»), conteggio `proposals.status='open'`. Sezione `<OpenProposals
  />` segnaposto che la 05b sostituisce.
- `src/app/(app)/progressi/page.tsx`: Server Component → `buildProgressViewModel` (§ 4.4) →
  `<RoutineSection>` × 5 → `<SlotCard>` con `<SessionChart>` (Client, Recharts) e `<SessionTable>`.
- `src/app/(app)/impostazioni/page.tsx`: `<SyncNowButton />` (Client: `fetch('/api/sync', { method:
  'POST' })` → testo esito), `<LogoutButton />` (Server Action `logout()` che cancella il cookie).
- `src/app/api/sync/route.ts`: `POST` → `sessionDenied()` → `syncWorkouts({ db: getDb(), hevy:
  getHevyClient(), now: Date.now() })` → JSON `SyncResult`; `export const maxDuration = 120`.
- Segnaposto `/revisioni` (lista vuota con testo «La prima revisione arriva sabato») così la nav non
  ha link morti; la 05b la riempie.

### 4.4 View model — `src/lib/progress/view-model.ts` (puro, testato)

```ts
export interface SlotProgress { slotId; routineKey; routineTitle; variantTitle; scalePosition: string; targetText: string; sessions: { date: string; values: number[]; mean: number; allAtTop: boolean }[]; referenceValue: number; unit: 'rip' | 's' }
export function buildProgressViewModel(result: AggregateResult, states: SlotState[]): SlotProgress[]
```

`referenceValue` = `max` per `reps`, `threshold` per `max2`, `maxSeconds` per `hold`; `scalePosition`
= `${indice+1} di ${lunghezza}`; ordine = `PLAN` (routine, poi slot). La pagina chiama
`aggregateWindow` con `window = { fromMs: romeDayStartMinusDays(now, 56), toMs: now }`.

### 4.5 Container — `src/lib/container.ts` (C-18)

```ts
export function getDb(): Db
export function getHevyClient(): HevyClient          // fake when process.env.E2E_FAKE_SERVICES === '1'
export function getReviewModel(): ReviewModel        // fake when E2E_FAKE_SERVICES === '1'
```

`tests/e2e/fakes/fake-hevy.ts`: client in memoria costruito dalle fixture + `addWorkout()`;
`tests/e2e/fakes/fake-model.ts`: risponde `keep` per ogni slot con una motivazione fissa
(«Fixture E2E»). Le route e le pagine usano SOLO il container (test `guards.test.ts`: nessun
`createHevyClient(` fuori da `container.ts` e `scripts/`).

### 4.6 Design — impeccable

Nella sessione: caricare la skill `impeccable`, far scegliere la direzione all'owner (tre
proposte a confronto, come nelle altre app), costruire la shell e le tre pagine in quella
direzione, poi far scrivere `DESIGN.md` e `PRODUCT.md` dalla skill (mai a mano), e aggiornare
`DEVELOPMENT_GUIDELINES.md § Design System` con il rimando. PRODUCT.md dice: utente singolo,
self-hosted, «un coach che propone, non decide», lingua italiana, uso dal telefono la sera.

### 4.7 E2E — Playwright

`playwright.config.ts`: `baseURL: http://localhost:3100`, `webServer: { command: 'pnpm e2e:server',
port: 3100, reuseExistingServer: false }`, progetti `mobile` (390×844, Pixel 7) e `desktop`
(1280×800). Script: `"e2e:server": "cross-env E2E_FAKE_SERVICES=1 TURSO_DATABASE_URL=file:e2e.db APP_PASSWORD=fenicottero-2500 SESSION_SECRET=<32+ chars fissi> HEVY_API_KEY=e2e ANTHROPIC_API_KEY=e2e PORT=3100 next dev -p 3100"`
(`cross-env` in devDependencies), `"e2e:seed": "tsx tests/e2e/seed.ts"` (cancella `e2e.db`, migra,
inserisce: 5 `hevy_routines` finte, 30 `slot_state` iniziali, `plan_uploaded_at` = 20 giorni fa, 6
workout inventati negli ultimi 56 giorni con `routine_id` delle routine finte: push-up 4×[12,12,11,10],
[13,13,12,12], [15,15,15,15]; rematore; plank; mobilità), `"test:e2e": "pnpm e2e:seed && playwright test"`.
`tests/e2e/auth.setup.ts` fa il login una volta e salva `playwright/.auth/owner.json` (gitignored).
Spec: `login.spec.ts`, `auth.spec.ts`, `shell.spec.ts` (overflow + axe su `/`, `/progressi`,
`/impostazioni`), `oggi.spec.ts`, `progressi.spec.ts`, `impostazioni.spec.ts`.

## 5. Vincoli e trappole

- **AGENTS.md § Auth** (nasce qui) — qui vuol dire che: il proxy è la prima barriera, non l'unica:
  ogni route e action chiama il guard (test A3); `isPublicPath` è l'**unico** elenco dei percorsi
  pubblici (07 aggiunge i suoi lì, non nel matcher).
- **AGENTS.md § UI** (nasce qui) — token di DESIGN.md, nessun colore letterale; grafici solo via
  `src/components/charts/recharts.ts` (l'unica porta a Recharts, come net-worth-tracker); stringhe
  italiane; stati vuoti sempre con un'azione.
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
| Unit | `session.test.ts`: token creato → verificato; scaduto (now + 366 giorni) → false; segreto diverso → false; `isPasswordCorrect` vero/falso e lunghezze diverse; `isPublicPath` per ogni voce + `/api/cron/review?x=1` + `/progressi` (false); `safeNext('//evil.com')` → `/`. | Togliere `currentDate` → il caso scaduto resta verde → rosso. |
| Unit | `view-model.test.ts`: ordine PLAN, `scalePosition`, `referenceValue` per genere, `mean` arrotondata a 1 decimale, slot senza sessioni presente con `sessions: []`. | — |
| Unit | `guards.test.ts`: legge `src/app/api/**/route.ts` e `src/app/**/actions.ts` e verifica la presenza di `sessionDenied(` o `assertSession(` salvo i due percorsi pubblici; nessun `createHevyClient(` fuori dal container. | Togliere il guard da `sync/route.ts` → rosso. |
| E2E | i sei spec di § 4.7 su `mobile` e `desktop`; axe senza violazioni `serious`/`critical`. | — |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli verdi; `pnpm review:run` da CLI ancora funzionante (CLI e app
  condividono il container? **No**: gli script usano direttamente `createHevyClient`; il container è
  dell'app).
- **C — Comportamento nuovo**: `pnpm test:e2e` verde (mobile + desktop).
- **D — Sotto la UI**: `curl -i localhost:3000/api/sync -X POST` → 401; con il cookie (preso dal
  browser) → JSON `SyncResult`.
- **F — Giro guidato (owner, dev server `pnpm dev` con `.env.local` reale, DB `local.db`)**: aperto
  già autenticato da Playwright (storageState) su (1) `/login` con password sbagliata: messaggio e
  attesa; (2) `/`: i quattro dati di Oggi corrispondono a `pnpm review:show` e a `sync_state`;
  (3) `/progressi` a 390 px: le schede del lunedì, il grafico del push-up con i punti delle sessioni
  reali; (4) `/impostazioni`: «Sincronizza ora» → «+0 allenamenti» (o +N); (5) l'estetica: è la
  direzione scelta? Il bug sarebbe: un overflow orizzontale, un grafico vuoto con sessioni presenti,
  un testo inglese.
- **G — Ripristino**: `e2e.db` cancellato; `playwright/.auth` ignorato da git.

Che cosa NON copre: proposte e revisioni in UI (05b), telefono reale (06/07).

## 8. Messa in esercizio

- Env (Development): `APP_PASSWORD`, `SESSION_SECRET`. DDL: nessuno.
- Misura di chiusura: `pnpm test:e2e` verde e giro guidato annotato.

## 9. Documentazione

- `DESIGN.md`, `PRODUCT.md` (impeccable); `DEVELOPMENT_GUIDELINES.md § Design System` → DESIGN.md.
- `CLAUDE.md`: Latest; § Key Features (Accesso, Oggi, Progressi, Impostazioni); § Testing (E2E);
  § Not yet present: togliere DESIGN/PRODUCT.
- `AGENTS.md`: § Auth, § UI, § E2E (fake services, seed, storageState).
- `doc/guide/app.md` (≤ 300 righe): shell, pagine, container, fake E2E, seed.
- `WORKFLOW.md § 3`: E2E e «come si apre l'app già autenticati» diventano veri.
- `README.md`: screenshot (inventati dal seed E2E, non dai dati reali) e § Running locally.

## 10. Fuori ambito, dichiarato

- Proposte, revisioni, «Applica», «Esegui revisione ora» → **05b**.
- PWA, manifest, push, `/offline` → **07** (i percorsi sono già in `isPublicPath`).
- Deploy → **06**.
- Rate limit del login oltre l'attesa di 1 s → CLAUDE.md § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **XL** (auth + shell + 3 pagine + design + E2E).

```
Ciao, in questa sessione implemento la spec doc/v1/05a-web-app-accesso-shell-e-progressi.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/piano.md, doc/guide/sync.md, doc/guide/revisione.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 05a INTERA
- Verifica con un grep i contratti di 01–04 che la spec usa (aggregateWindow, listSlotStates,
  shouldRunScheduledReview, syncWorkouts, targetText, romeDayStartMinusDays, PLAN, SCALES)
- Carica le skill impeccable e dataviz PRIMA di scrivere un componente

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/05a-app-progressi. Crea SESSION_NOTES.md.

Ordine di lavoro: (1) sessione, proxy, login, container, guard e i loro test; (2) il seed E2E e
Playwright; (3) con impeccable: tre direzioni visive a confronto, me le mostri e scelgo; (4)
shell e le tre pagine nella direzione scelta, grafici con dataviz; (5) DESIGN.md e PRODUCT.md
scritti dalla skill; (6) § 7 collaudo fase per fase con l'esito atteso prima, il giro guidato
sul dev server già autenticato; (7) § 9 documentazione. Se un punto non è coperto dalla spec,
fermati e chiedimelo con AskUserQuestion. Le etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
