# 07 — PWA e Web Push

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 34 rilievi, 3 decisioni
> (le scelte T-24, T-25 e, condivisa, D-38). Pronta per l'implementazione dopo la 06.
> **Branch**: `feature/07-pwa-push` da `main`. **Dipende da**: 05a (shell, `isPublicPath`,
> container, E2E), 05b (`sendReviewNotification` no-op, `PushReport`), 06 (produzione HTTPS, cron,
> `db:migrate:remote`).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-12, D-31, D-36, D-37, D-38, D-43, D-56; scelte
> T-2, T-4, T-5, T-12 (`POST /api/push/test`), T-14, T-17, T-24, T-25; contratti C-7
> (`push_subscriptions` e il repository `push-subscriptions.ts`), C-16 (lo riempie); usa C-10, C-14,
> C-15, C-18, `createMemoryDb` (03), `listProposalsOfReview` (04).

## 1. Il problema oggi (prove)

1. L'app è un sito: non si installa, non ha icona, non avvisa. `sendReviewNotification` è un no-op.
2. Requisiti letti il 2026-10-23:
   - Chromium (`https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable`,
     `https://developer.chrome.com/blog/update-install-criteria`): manifest con `name`, `icons` 192 e
     512, `start_url`, `display` standalone, HTTPS, e **un service worker con un `fetch` handler**
     per il prompt automatico.
   - iOS (`https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/`,
     `https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers`,
     `https://webkit.org/blog/17333/webkit-features-in-safari-26-0/`): push **solo** dalla web app
     aggiunta alla Home (iOS ≥ 16.4), `display: standalone`, permesso chiesto **da un gesto
     dell'utente** nel gestore dell'evento, **niente push silenziose** (Safari revoca), payload ≤ 4 KB,
     410 = sottoscrizione scaduta; iOS 26: qualunque sito aggiunto alla Home apre come web app.
   - Web Push (`https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe`, README di
     `web-push` 3.6.7 su GitHub): `subscribe({ userVisibleOnly: true, applicationServerKey })`, chiavi
     VAPID, `sendNotification` **rifiuta** con `WebPushError { statusCode, body }` su ogni risposta
     non 2xx (404/410 = cancellare).
   - Next 16 (`https://nextjs.org/docs/app/api-reference/functions/generate-metadata`,
     `.../file-conventions/metadata/manifest`): `app/manifest.ts` emette da sé `<link rel="manifest">`,
     `appleWebApp` emette `mobile-web-app-capable`, `app/apple-icon.png` → `apple-touch-icon`.
   - Serwist (`https://serwist.pages.dev/docs/next/getting-started`, esempio `next-basic`, letta il
     2026-10-24): `swSrc` `app/sw.ts`, `swDest` `public/sw.js`, `next build --webpack` (già in uso,
     T-4); `tsconfig` con `lib` `webworker`, `types` `@serwist/next/typings`, `exclude` `public/sw.js`;
     `.gitignore` con `public/sw*` e `public/swe-worker*`.
3. iOS: il permesso va chiesto da un tap; la sottoscrizione si perde se l'app viene rimossa dalla
   Home; non c'è modo di rilevarlo dal server se non il 410.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `/manifest.webmanifest` (da `src/app/manifest.ts`): `name` «Hevy Coach», `short_name` «Coach», `id: '/'`, `start_url: '/'`, `display: 'standalone'`, `lang: 'it'`, `background_color`/`theme_color` = l'esadecimale del token di sfondo di DESIGN.md (§ 4.2), icone 192, 512 e 512 `maskable` in `public/icons/`; `apple-icon.png` 180; `appleWebApp` nel layout; è pubblico (`isPublicPath`). | E2E `pwa.spec.ts` (build di produzione): GET manifest → JSON con i campi; **esattamente un** `link[rel=manifest]` e un `apple-touch-icon` nell'HTML di `/login`. Mutazione: impostare anche `metadata.manifest` → due link → rosso. |
| A2 | `public/sw.js` generato da `src/app/sw.ts` con Serwist: precache degli asset, `NetworkFirst` sulle pagine, `NetworkOnly` su `/api/*`, fallback `/offline` precaricato, **listener `push`** (mostra sempre una notifica: titolo/corpo/url dal payload, default se manca), **`notificationclick`** (§ 4.3: focus + `navigate` su una finestra controllata, altrimenti `openWindow`), messaggi `SKIP_WAITING` e `CLEAR_PAGES`; registrato da `<SwProvider>` in produzione; `disable` in dev. | E2E: `/sw.js` servito con `content-type` JS e contiene `push` e `notificationclick`; `/offline` risponde 200. Unit: `src/lib/push/payload.test.ts`. |
| A3 | Tabella `push_subscriptions` (migrazione `0002_push`): `endpoint` unico, `p256dh`, `auth`, `user_agent`, `created_at`, `last_success_at`, `last_error`; repository `src/lib/db/repositories/push-subscriptions.ts`. `POST /api/push/subscribe` (guard) con body `{ subscription: PushSubscriptionJSON }` validato con zod → upsert per `endpoint`, 200 `{ ok: true }`; `DELETE` con `{ endpoint }` → delete, 204; body malformato → 400. | `src/lib/push/subscriptions.test.ts` su `handleSubscribe`/`handleUnsubscribe` con `createMemoryDb()`: 2 POST stesso endpoint → 1 riga; DELETE → 0; body senza `keys` → 400. Mutazione: insert al posto dell'upsert → rosso. |
| A4 | `/impostazioni` › «Notifiche»: stato (`unsupported` / `ios_not_installed` → istruzioni «Condividi → Aggiungi alla schermata Home» / `denied` / `on` / `off`, più «Verifica…» finché la registrazione non è nota), interruttore che su tap chiede il permesso e sottoscrive con `NEXT_PUBLIC_VAPID_PUBLIC_KEY`, poi `POST`; bottone «Invia notifica di prova» → `POST /api/push/test` → una push «Notifica di prova» a tutte le sottoscrizioni, esito «Inviata a N dispositivi» (0 → «Nessun dispositivo iscritto»). | Unit `src/lib/push/ui-state.test.ts` (tabella di verità e ordine); E2E `pwa.spec.ts`: stato `unsupported` con `PushManager` tolto via `addInitScript` (testo e interruttore disabilitato) e stato `off` senza init script (interruttore «Attiva» abilitato). Mutazione: `supported` sempre vero → il primo caso va rosso. |
| A5 | `sendPushToAll` (C-16): `web-push` con VAPID, payload JSON `{ title, body, url, tag }` ≤ 4 KB, `TTL: 86400`, `urgency: 'normal'`, `timeout: 10_000`, invii in parallelo; 404/410 → riga cancellata; altri errori → `last_error`; successo → `last_success_at`. `sendReviewNotification` compone i testi di T-17 con `url` **relativo** (`/revisioni/<id>` o `/`, T-24) e N = proposte non `insufficient_data` della revisione (D-43); sender **pigro** (T-25). | `send.test.ts` con sender finto: 2 sottoscrizioni, una risponde 410 → cancellata, l'altra ok; i sei casi di `sendReviewNotification`; ogni payload composto passa `PushPayloadSchema.parse`; zero sottoscrizioni e nessuna env VAPID → `{ 0, 0, 0 }` senza toccare `web-push`. Mutazione: non cancellare sul 410 → rosso; usare `proposalCount` → rosso; url assoluto → rosso. |
| A6 | Il cron (06) e il run manuale (05b) chiamano la vera `sendReviewNotification`: nessun codice cambia lì (stessa firma, stesso `PushReport`), il no-op sparisce; i loro test restano verdi senza modifiche (T-25). | Grep: una sola definizione di `sendReviewNotification`; `pnpm test` verde senza toccare `run-cron.test.ts`. |
| A7 | Installazione e push funzionano su Android (Chrome) e su iPhone (Safari, dalla Home) con l'app in produzione; la notifica di prova arriva su entrambi i telefoni. | Collaudo § 7 F e misura di chiusura (§ 8); la prima push «Revisione pronta» del sabato è storia, annotata nella riga Stato quando arriva, non un criterio da aspettare (D-56). |

## 3. Decisioni

- **D-12** (Web Push), **D-31/T-17** (testi: titolo e corpo, caso 0, singolare, `already_running`
  senza notifica), **D-36** (le VAPID si leggono con `EnvSchema.pick` dentro il sender), **D-37/D-38**
  (chiusura: migrazione ed env prima del merge, esiti post-merge nel commit di ritiro), **D-43** (N),
  **T-2** (versioni esatte), **T-4** (`--webpack`), **T-5** (tutto in `src/lib/push/`), **T-12**
  (`POST /api/push/test`), **T-14** (nome, tema), **T-24** (url relativo, niente `APP_URL`), **T-25**
  (sender pigro).
- Adottate qui, ribaltabili: (a) icona: un monogramma «HC» disegnato con impeccable in
  `public/icon.svg` (è l'unica scelta grafica della sessione, dichiarata) e rasterizzato da
  `scripts/generate-icons.ts` (sharp) nelle misure 192, 512, 512 maskable, 180 (apple) e 32
  (`src/app/icon.png`); i PNG si committano; `src/app/favicon.ico` di create-next-app si cancella;
  (b) `web-push@3.6.7` (ultima release 2024-01, repo attivo: è ciò che la doc Next usa); (c) una
  sola «utenza»: tutte le sottoscrizioni ricevono tutto; (d) `tag` = `review-<id>` per non
  duplicare la stessa notifica su un dispositivo, `test-<now>` per la prova (così due prove di
  seguito si vedono entrambe); (e) nessun badge (`setAppBadge`) in v1; (f) il prompt di
  installazione Android (`beforeinstallprompt`) **non** si intercetta: Chrome mostra il suo; su iOS
  l'istruzione testuale; (g) `SwProvider` registra `/sw.js` con `updateViaCache: 'none'` e, su
  `waiting`, mostra una barra «Aggiornamento disponibile» con «Ricarica» (`SKIP_WAITING`, poi
  `location.reload()` al `controllerchange`); (h) in dev il SW è disattivato: la PWA si prova solo su
  `pnpm build && pnpm start` e in produzione; (i) la notifica di `failed` non include l'errore
  completo, solo i primi 80 caratteri; (j) la notifica parte anche per le revisioni manuali
  (`trigger 'manual'`: l'owner ha chiuso l'app o lo schermo nei minuti d'attesa) e **mai** da
  `pnpm review:run` (la CLI non chiama `sendReviewNotification`); (k) al caricamento di `/login`
  `<SwProvider>` manda `CLEAR_PAGES` al worker, che svuota la cache `pages`: dopo il logout le pagine
  autenticate non restano leggibili offline; (l) `pushsubscriptionchange` non si gestisce: se il
  browser ruota l'endpoint, l'owner riattiva da Impostazioni (§ 10); (m) al mount, se il browser ha
  una sottoscrizione, `<PushSettings>` la ri-POSTa (idempotente): così un DB rifatto o una riga
  cancellata per 410 si risincronizzano al primo giro; (n) `statusBarStyle: 'black-translucent'` con
  `viewportFit: 'cover'` e i `safe-area-inset` in header e `BottomNav`; (o) il payload ha `url`
  limitato a 200 caratteri, così per costruzione resta sotto i 4 KB: `encodePushPayload` lo
  controlla comunque; (p) il progetto E2E `pwa` gira su Chromium (`devices['Pixel 7']`), su una
  build di produzione, con lo stesso seed e lo stesso `storageState` della 05a.

## 4. La soluzione

### 4.1 Dipendenze e configurazione

```bash
pnpm add @serwist/next@9.5.13 web-push@3.6.7
pnpm add -D serwist@9.5.13 @types/web-push@<versione esatta di oggi> sharp@<versione esatta di oggi>
```

(`<versione esatta di oggi>`: regola di T-2, letta con `npm view <pkg> version` e scritta in T-2 nel
commit della sessione. Niente `png-to-ico`: nessun `.ico` viene generato.)

`next.config.ts`:

```ts
import { randomUUID } from 'node:crypto';
import withSerwistInit from '@serwist/next';

const offlineRevision = randomUUID();
const withSerwist = withSerwistInit({
  swSrc: 'src/app/sw.ts',
  swDest: 'public/sw.js',
  additionalPrecacheEntries: [{ url: '/offline', revision: offlineRevision }],
  disable: process.env.NODE_ENV === 'development',
  register: false,
});
export default withSerwist({ agentRules: false });
```

`tsconfig.json`: `compilerOptions.lib` aggiunge `"webworker"`, `compilerOptions.types` aggiunge
`"@serwist/next/typings"`, `exclude` aggiunge `"public/sw.js"`. `.gitignore` aggiunge `public/sw*`
e `public/swe-worker*`. `PUBLIC_PATHS` (05a) riceve `'/swe-worker-'` (prefisso: la 05a tratta come
prefissi le voci che finiscono con `/` o `-`), perché `@serwist/next` può servire quel file senza
cookie.

Env (`src/lib/env-schema.ts`, D-36): `VAPID_SUBJECT: z.string().startsWith('mailto:')` (mai
`https://localhost`: Safari risponde `BadJwtToken`; l'indirizzo è un dato dell'owner e sta solo in
`.env.local` e su Vercel), `VAPID_PUBLIC_KEY: z.string().min(80)`, `VAPID_PRIVATE_KEY:
z.string().min(40)`, `NEXT_PUBLIC_VAPID_PUBLIC_KEY: z.string().min(80)`, con
`.refine((e) => e.NEXT_PUBLIC_VAPID_PUBLIC_KEY === e.VAPID_PUBLIC_KEY, 'NEXT_PUBLIC_VAPID_PUBLIC_KEY
must equal VAPID_PUBLIC_KEY')`. Chiavi generate con `pnpm exec web-push generate-vapid-keys --json`.
`.env.example` elenca le quattro vuote. `tests/e2e/vapid.ts` esporta una coppia generata una volta
per gli E2E (mai usata in produzione) e `vitest.config.ts` aggiunge quei valori finti; `e2e:server`
(05a) ed `e2e:prod` (§ 4.6) ricevono le quattro variabili.

### 4.2 Manifest, icone, layout

`src/app/manifest.ts`:

```ts
import type { MetadataRoute } from 'next';
import { BACKGROUND_HEX } from '@/lib/ui/theme-literals';
export default function manifest(): MetadataRoute.Manifest {
  return {
    id: '/', name: 'Hevy Coach', short_name: 'Coach', description: 'Il coach del tuo allenamento a corpo libero, sopra Hevy.',
    start_url: '/', display: 'standalone', lang: 'it', dir: 'ltr',
    background_color: BACKGROUND_HEX, theme_color: BACKGROUND_HEX,
    icons: [
      { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/icons/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ],
  };
}
```

`src/lib/ui/theme-literals.ts`: `export const BACKGROUND_HEX = '#rrggbb'` = il valore esadecimale
del token di sfondo di `body` nel tema scuro di DESIGN.md, con un commento che nomina il token: è
l'**unica** eccezione alla regola «nessun colore letterale» (AGENTS.md § 7), perché il manifest e
`themeColor` non risolvono variabili CSS; il file sta in `src/lib/ui/`, fuori dal grep di
`guards.test.ts` (05a), che copre `src/app` e `src/components`. `src/app/layout.tsx`: `metadata.appleWebApp = { capable: true, title: 'Coach',
statusBarStyle: 'black-translucent' }`; `export const viewport = { themeColor: BACKGROUND_HEX,
viewportFit: 'cover' }`; `metadata.manifest` **non** si imposta (Next emette il link da
`manifest.ts`). `src/app/(app)/layout.tsx` aggiunge `padding-top: env(safe-area-inset-top)` all'header
e `bottom-nav.tsx` `padding-bottom: env(safe-area-inset-bottom)`. `src/app/apple-icon.png` (180×180)
e `src/app/icon.png` (32×32) generati da `pnpm icons:generate` (`"icons:generate": "tsx
scripts/generate-icons.ts"`). `src/app/offline/page.tsx`: «Sei offline. L'ultima pagina aperta è in
cache; riprova quando torni in rete.» (pubblica).

### 4.3 Service worker — `src/app/sw.ts`

```ts
import { defaultCache } from '@serwist/next/worker';
import type { PrecacheEntry, SerwistGlobalConfig } from 'serwist';
import { NetworkFirst, NetworkOnly, Serwist } from 'serwist';
import { parsePushPayload } from '@/lib/push/payload';

declare global { interface WorkerGlobalScope extends SerwistGlobalConfig { __SW_MANIFEST: (PrecacheEntry | string)[] | undefined } }
declare const self: ServiceWorkerGlobalScope;

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: false,
  clientsClaim: true,
  navigationPreload: true,
  runtimeCaching: [
    { matcher: ({ url }) => url.pathname.startsWith('/api/'), handler: new NetworkOnly() },
    { matcher: ({ request }) => request.mode === 'navigate', handler: new NetworkFirst({ cacheName: 'pages', networkTimeoutSeconds: 5 }) },
    ...defaultCache,
  ],
  fallbacks: { entries: [{ url: '/offline', matcher: ({ request }) => request.destination === 'document' }] },
});

self.addEventListener('push', (event) => {
  const payload = parsePushPayload(event.data?.text() ?? null);
  event.waitUntil(self.registration.showNotification(payload.title, { body: payload.body, tag: payload.tag, data: { url: payload.url }, icon: '/icons/icon-192.png' }));
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  const url = (event.notification.data as { url?: string } | undefined)?.url ?? '/';
  event.waitUntil((async () => {
    // Only a controlled window can be navigated: an uncontrolled one makes navigate() reject.
    const [client] = await self.clients.matchAll({ type: 'window', includeUncontrolled: false });
    if (!client) { await self.clients.openWindow(url); return; }
    await client.focus();
    try { await client.navigate(url); } catch { await self.clients.openWindow(url); }
  })());
});

self.addEventListener('message', (event) => {
  if (event.data?.type === 'SKIP_WAITING') self.skipWaiting();
  if (event.data?.type === 'CLEAR_PAGES') event.waitUntil(caches.delete('pages'));
});
serwist.addEventListeners();
```

`src/lib/push/payload.ts` (puro): `PushPayloadSchema = z.object({ title: z.string().max(80), body:
z.string().max(200), url: z.string().startsWith('/').max(200), tag: z.string().max(40) })`,
`PushPayload = z.infer<…>`, `parsePushPayload(text: string | null): PushPayload` → valido o
`{ title: 'Hevy Coach', body: 'Apri l\'app', url: '/', tag: 'default' }`, `encodePushPayload(payload:
PushPayload): string` → `JSON.stringify`, lancia se > 4096 byte (`Buffer.byteLength`).

### 4.4 Client — `src/components/pwa/`

- `sw-provider.tsx` (Client, montato in `src/app/layout.tsx`: registra anche da `/login`): in
  produzione `navigator.serviceWorker.register('/sw.js', { updateViaCache: 'none' })`; su
  `registration.waiting` mostra la barra fissa sopra la `BottomNav` «Aggiornamento disponibile» con
  «Ricarica» → `waiting.postMessage({ type: 'SKIP_WAITING' })` e `location.reload()` al
  `controllerchange`; quando `usePathname() === '/login'` manda `{ type: 'CLEAR_PAGES' }` al
  controller (§ 3 k).
- `push-settings.tsx` (Client, sezione «Notifiche» in `src/app/(app)/impostazioni/page.tsx`, sopra
  «Sincronizza ora»). Al mount: `registration = await navigator.serviceWorker.getRegistration('/')`
  (mai `ready`: in dev non si risolve); finché non è nota, interruttore disabilitato con «Verifica…»;
  `supported = registration !== undefined && 'PushManager' in window && 'Notification' in window`;
  `isIos = /iPhone|iPad|iPod/.test(navigator.userAgent) || (navigator.maxTouchPoints > 1 &&
  /Macintosh/.test(navigator.userAgent))` (iPad con UA da Mac); `standalone = matchMedia('(display-mode:
  standalone)').matches || (navigator as { standalone?: boolean }).standalone === true`; `subscribed`
  = `await registration?.pushManager.getSubscription()` non null, e in quel caso un `POST
  /api/push/subscribe` idempotente (§ 3 m). `pushUiState({ supported, standalone, isIos, permission,
  subscribed })` (puro, `src/lib/push/ui-state.ts`), **primo che vale**: (1) `isIos && !standalone` →
  `ios_not_installed`; (2) `!supported` → `unsupported` («Notifiche disponibili solo nell'app
  installata dalla build di produzione»); (3) `permission === 'denied'` → `denied`; (4) `subscribed`
  → `on`; (5) altrimenti `off`. Tap su «Attiva» (disabilitato in `working`): `Notification.
  requestPermission()` come **prima** chiamata del gestore, poi `registration.pushManager.subscribe({
  userVisibleOnly: true, applicationServerKey: urlBase64ToUint8Array(key) })`; su `InvalidStateError`
  (chiave VAPID ruotata) `unsubscribe()` della vecchia e nuovo `subscribe`; poi `fetch('/api/push/subscribe',
  { method: 'POST', body: JSON.stringify({ subscription: subscription.toJSON() }) })`; errore →
  «Attivazione non riuscita: riprova» e stato `off`. «Disattiva» → `unsubscribe()` + `DELETE` con
  `{ endpoint }`. «Invia notifica di prova» → `POST /api/push/test` → «Inviata a N dispositivi» (N =
  `sent`; 0 → «Nessun dispositivo iscritto»; `failed > 0` → «, M non raggiunti»).
- `src/lib/push/url-base64.ts`: `urlBase64ToUint8Array(key: string): Uint8Array` (test).

### 4.5 Server — `src/lib/push/`

Repository `src/lib/db/repositories/push-subscriptions.ts` (AGENTS.md § 2.1):
`upsertPushSubscription(db, { endpoint, p256dh, auth, userAgent }, now)` (`INSERT … ON
CONFLICT(endpoint) DO UPDATE SET p256dh, auth, user_agent`; `id = nanoid()` e `created_at = now` solo
all'insert), `deletePushSubscription(db, endpoint): Promise<number>`, `listPushSubscriptions(db)`,
`markPushSuccess(db, id, now)` (`last_success_at = now`, `last_error = null`), `markPushError(db, id,
message)`. Test `push-subscriptions.test.ts` con `createMemoryDb()`.

`src/lib/push/subscriptions.ts`: `PushSubscriptionJsonSchema = z.object({ endpoint:
z.string().url(), expirationTime: z.number().nullable().optional(), keys: z.object({ p256dh:
z.string(), auth: z.string() }) })`, `PushSubscribeBodySchema = z.object({ subscription:
PushSubscriptionJsonSchema })`, `handleSubscribe({ db }, body: unknown, userAgent: string | null,
now): Promise<Response>` (400 su body non valido, 200 `{ ok: true }`), `handleUnsubscribe({ db },
body: unknown): Promise<Response>` (400, 204). Le route sono sottili: `src/app/api/push/subscribe/route.ts`
→ `POST`: `sessionDenied()` → `handleSubscribe({ db: getDb() }, await request.json().catch(() =>
null), request.headers.get('user-agent'), Date.now())`; `DELETE` idem con `handleUnsubscribe`;
`src/app/api/push/test/route.ts` → `POST`, guard → `sendPushToAll({ db: getDb(), payload: { title:
'Notifica di prova', body: 'Se la leggi, funziona.', url: '/impostazioni', tag: \`test-${Date.now()}\` },
now: Date.now() })` → JSON `PushReport`; tutte con `maxDuration = 60`.

`src/lib/push/send.ts` (C-16; **l'unico file che importa `web-push`**):

```ts
export interface PushReport { sent: number; removed: number; failed: number }          // already exported by the 05b no-op: same shape
export interface PushSender { send(subscription: WebPushSubscription, payload: string, options: { TTL: number; urgency: 'normal'; timeout: number }): Promise<{ statusCode: number; body: string }> }
export function createWebPushSender(vapid: { VAPID_SUBJECT: string; VAPID_PUBLIC_KEY: string; VAPID_PRIVATE_KEY: string }): PushSender
export async function sendPushToAll({ db, payload, now, sender }: { db: Db; payload: PushPayload; now: number; sender?: PushSender }): Promise<PushReport>
export async function sendReviewNotification({ db, review, sender, now = Date.now() }: { db: Db; review: ReviewRun; sender?: PushSender; now?: number }): Promise<PushReport>
```

`createWebPushSender` cattura `WebPushError` e ritorna `{ statusCode: error.statusCode, body:
error.body }`; un errore senza `statusCode` (rete, timeout) ritorna `{ statusCode: 0, body:
error.message }`: `send` **non lancia mai**. `sendPushToAll`: `subs = listPushSubscriptions(db)`; con
zero righe ritorna `{ 0, 0, 0 }` **senza** costruire il sender né leggere le env (T-25); altrimenti
`sender ??= createWebPushSender(EnvSchema.pick({ VAPID_SUBJECT: true, VAPID_PUBLIC_KEY: true,
VAPID_PRIVATE_KEY: true }).parse(process.env))` (D-36); `text = encodePushPayload(payload)`; invii in
parallelo con `Promise.all` e `{ TTL: 86400, urgency: 'normal', timeout: 10_000 }`; 2xx →
`markPushSuccess`, `sent++`; 404/410 → `deletePushSubscription`, `removed++`; altro →
`markPushError(db, id, \`${statusCode} ${body}\`.slice(0, 200))`, `failed++`.

`sendReviewNotification` (T-17): `completed` → titolo «Revisione pronta», corpo da N =
`(await listProposalsOfReview(db, review.id)).filter((p) => !p.insufficientData).length`: 0 →
«Nessuna proposta: tutto resta com'è», 1 → «1 proposta da guardare», N → «N proposte da guardare»;
url `/revisioni/<id>`; `skipped/no_workouts` → «Revisione saltata» / «Nessun allenamento nelle
ultime 4 settimane», url `/`; `skipped/already_running` → nessuna notifica, ritorna `{ sent: 0,
removed: 0, failed: 0 }`; `failed` → «Revisione fallita» / `(review.error ?? 'errore sconosciuto').slice(0,
80)`, url `/revisioni/<id>`; `tag` = `review-<id>`. Ogni payload passa `PushPayloadSchema.parse`
prima dell'invio.

Migrazione `0002_push` (`pnpm db:generate --name push`):

```ts
export const pushSubscriptions = sqliteTable('push_subscriptions', {
  id: text('id').primaryKey(),
  endpoint: text('endpoint').notNull(),
  p256dh: text('p256dh').notNull(),
  auth: text('auth').notNull(),
  userAgent: text('user_agent'),
  createdAt: integer('created_at').notNull(),
  lastSuccessAt: integer('last_success_at'),
  lastError: text('last_error'),
}, (t) => [uniqueIndex('push_subscriptions_endpoint_idx').on(t.endpoint)]);
```

### 4.6 E2E su build di produzione

`playwright.config.ts`: `webServer` diventa un array: `[{ command: 'pnpm e2e:server', port: 3100,
reuseExistingServer: false }, { command: 'pnpm e2e:prod', port: 3200, reuseExistingServer: false,
timeout: 300_000 }]`, il secondo solo quando `process.env.PW_PWA === '1'`. Progetto `pwa`: `use: {
...devices['Pixel 7'], baseURL: 'http://localhost:3200', storageState: 'playwright/.auth/owner.json'
}`, `dependencies: ['setup']`, `testMatch: /pwa\.spec\.ts/`; `mobile` e `desktop` hanno `testIgnore:
/pwa\.spec\.ts/`. Script: `"test:e2e": "pnpm e2e:seed && playwright test --project=setup
--project=mobile --project=desktop"`, `"test:e2e:pwa": "pnpm e2e:seed && cross-env PW_PWA=1 playwright
test --project=setup --project=pwa"`, `"e2e:prod": "node scripts/e2e-prod.mjs"` (su ogni piattaforma:
`spawnSync('pnpm', ['exec', 'next', 'build', '--webpack'], { shell: false })` e poi `spawn` di `next
start -p 3200`, con le env di `e2e:server` più `PORT=3200` e le quattro VAPID di `tests/e2e/vapid.ts`).
Il `setup` fa il login sulla porta del progetto che lo richiede. `tests/e2e/pwa.spec.ts`: A1, A2 e i
due stati di A4. `pnpm test:e2e:pwa` entra nel cancello di chiusura (WORKFLOW § 3).

## 5. Vincoli e trappole

- **AGENTS.md § 8 Deploy and PWA** (stub della pianificazione; qui la parte PWA diventa regole) —
  il SW si prova solo su build; `/sw.js`, `/swe-worker-*` e il manifest sono pubblici in
  `isPublicPath`; mai rinominare `sw.js`: una registrazione esistente continua a cercare lo script
  al vecchio URL e, se riceve 404, il browser la rimuove e la sottoscrizione push di quel dispositivo
  si perde; `web-push` si importa in un solo file; il payload è ≤ 4 KB per costruzione
  (`encodePushPayload` lo garantisce con un test); un solo colore letterale, in `theme-literals.ts`.
- iOS: il tap deve chiamare `requestPermission` **nello stesso gestore** dell'evento (niente
  `await` prima: la registrazione si prende al mount); una notifica deve **sempre** essere mostrata
  nel `push` (altrimenti Safari revoca).
- `next build --webpack` è già il comando (T-4); l'avviso di Next sul plugin webpack è atteso.
- `public/sw.js` e `public/swe-worker*` sono generati e ignorati da git.
- Il fake E2E non ha push: `unsupported` e `off` sono gli stati testati in automatico; `on`,
  `denied` e `ios_not_installed` si provano sui telefoni (dichiarato).
- Il cron (06) chiama `sendReviewNotification` dopo `runReview`: con `timeout: 10_000` e invii in
  parallelo l'invio dura al massimo 10 s, dentro i 300 s.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `src/lib/push/payload.test.ts`: parse valido, testo non JSON → default, url non relativa → default, url di 201 caratteri → default; `encodePushPayload` lancia con un `body` di 5 000 caratteri passato con un cast. | Togliere il limite → rosso. |
| Unit | `src/lib/push/ui-state.test.ts`: tabella di verità dei 5 stati; iOS non standalone con `supported: false` e `permission: 'granted'` → `ios_not_installed`. | Controllare `!supported` prima di `ios_not_installed` → rosso. |
| Unit | `src/lib/push/url-base64.test.ts`: chiave d'esempio con `-` e `_` → 65 byte. | Non sostituire `-`/`_` → rosso. |
| Unit + DB | `src/lib/push/send.test.ts`: sender finto; 410 → riga cancellata; 500 → `last_error` «500 …»; `statusCode: 0` → `last_error`, `failed`; 201 → `last_success_at`, `last_error` null; i sei casi di `sendReviewNotification` (completed con 0, 1 e N proposte di cui alcune `insufficient_data`; skipped/no_workouts; skipped/already_running → nessun invio; failed con errore troncato a 80 e con `error` assente); ogni payload passa `PushPayloadSchema.parse`; zero sottoscrizioni senza env VAPID → `{ 0, 0, 0 }`. | Non cancellare sul 410 → rosso; contare anche `insufficient_data` → rosso; costruire il sender prima di leggere le righe → l'ultimo caso va rosso. |
| Unit | `src/lib/push/create-web-push-sender.test.ts` con `vi.mock('web-push')` che lancia `new WebPushError('gone', 410, {}, 'Gone', 'https://…')` → `{ statusCode: 410, body: 'Gone' }`; che lancia `new Error('ETIMEDOUT')` → `{ statusCode: 0 }`. | Rilanciare → rosso. |
| Unit + DB | `src/lib/push/subscriptions.test.ts`: 2 POST stesso endpoint → 1 riga (`created_at` del primo), DELETE → 0 righe e 204, body senza `keys` → 400; `push-subscriptions.test.ts`: `markPushSuccess` azzera `last_error`. | Insert al posto dell'upsert → rosso. |
| E2E (`pwa`) | `tests/e2e/pwa.spec.ts`: manifest (un solo `link[rel=manifest]`), `sw.js`, `/offline`, `/impostazioni` stato `unsupported` (init script) e `off`. | Togliere `/sw.js` da `isPublicPath` → la GET riceve il redirect a `/login` → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E dei progetti `mobile`/`desktop` verdi; `pnpm test:e2e:pwa`
  verde; i test della 05b e della 06 verdi senza modifiche (A6). Nessuna chiamata reale.
- **C — Locale su build**: `pnpm build && pnpm start` con `.env.local` (VAPID reali): Chrome desktop:
  Application → Manifest senza errori, Service Worker attivo, «Install» disponibile. Nessuna chiamata
  a Hevy né ad Anthropic.
- **C2 — Push locale**: sullo stesso `pnpm start`, Chrome desktop `/impostazioni` → «Attiva» → «Invia
  notifica di prova» → arriva entro 10 s, `push_subscriptions.last_success_at` valorizzato in
  `local.db` (invio reale al push service di Chrome, gratis). Poi «Disattiva» → 0 righe.
- **D — Sotto la UI**: con lo script usa e getta della 05a (cookie), `POST /api/push/subscribe` con
  un body d'esempio → 200 e una riga in `push_subscriptions`; stesso POST → ancora 1 riga; `DELETE` →
  204 e 0 righe.
- **E — Casi negativi**: stesso POST senza cookie → 401; body senza `keys` → 400.
- **H — Produzione (prima del merge, poi merge; D-38, ciascuno dopo l'OK)**: (1) `pnpm
  db:migrate:remote` (applica `0002_push` su Turso di produzione: scrittura DDL permanente, gratuita);
  (2) le quattro env VAPID su Vercel Production: tre Secret con `pnpm dlx vercel env add <NAME>
  production`, `NEXT_PUBLIC_VAPID_PUBLIC_KEY` con `--type config`; (3) commit unico, PR, merge →
  deploy automatico; (4) `curl -s https://<prod>/manifest.webmanifest` → JSON con `name` «Hevy
  Coach».
- **F — Giro guidato (owner, due telefoni)**: Android/Chrome: (1) banner o menu «Installa»; (2) app
  dalla Home, `/impostazioni` → «Attiva» → permesso → «Attive»; (3) «Invia notifica di prova» →
  arriva, il tap apre `/impostazioni`. iPhone/Safari: (4) Condividi → Aggiungi alla schermata Home,
  apri dalla Home, `/impostazioni` → lo stato NON dice più «installa», «Attiva» → permesso → «Attive»
  → prova → arriva; l'header non è coperto dalla status bar e la barra in basso non tocca
  l'indicatore Home. (5) Da Safari (non installata) lo stato dice «aggiungi alla Home». Ogni prova è
  un invio reale ai push service (FCM, APNs), gratis; le righe in `push_subscriptions` di produzione
  restano (sono quelle vere). Il bug sarebbe: nessuna notifica entro 10 s, un tap che non apre
  l'app, Safari che revoca il permesso (notifica non mostrata).
- **G — Ripristino**: le sottoscrizioni di prova restano (sono quelle vere); la riga di Chrome
  desktop in `local.db` cancellata da «Disattiva». Nessuna fixture.

Che cosa NON copre: la prima notifica del cron vero (storia, non segnale: D-56).

## 8. Messa in esercizio

- Env Production: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (Secret),
  `NEXT_PUBLIC_VAPID_PUBLIC_KEY` (Config). DDL: `0002_push` remoto **prima** del merge (§ 7 H).
- Chiusura secondo D-38: il commit unico porta Stato «implementata» con gli esiti di § 7 H(1)–(2);
  il giro F e la misura di chiusura si scrivono (Stato «in esercizio il <data>», CLAUDE.md Latest,
  § 5 dell'indice) nel commit di ritiro della serie (`PROMPT-RITIRO.md`).
- Misura di chiusura (D-56, 2026-10-24): in produzione, dall'app installata, «Invia notifica di
  prova» arriva su **entrambi i telefoni** entro 10 s (giro F, punti 3 e 4) e
  `push_subscriptions.last_success_at` è valorizzato per entrambe le righe (SQL console di Turso).
  La prima push «Revisione pronta» del sabato con revisione dovuta **NON è un segnale** e non si
  aspetta: quando arriva (`cron_runs.details_json` con `push.sent = 2`) si annota nella riga Stato di
  questa spec come storia. Quel sabato il cron fa il suo lavoro normale: GET Hevy e fino a due
  chiamate ad Anthropic (< 0,80 $, 04).

## 9. Documentazione

- `CLAUDE.md`: Latest; § Current Status → Stack: le versioni di `@types/web-push` e `sharp`; § Key
  Features (PWA, Notifiche); § Not yet present: badge, Declarative Web Push e offline writes ci sono
  già; + «`pushsubscriptionchange` not handled: re-enable notifications from Impostazioni».
- `AGENTS.md § 8 Deploy and PWA`: la parte PWA diventa 3–5 righe con il rimando a
  `doc/guide/pwa.md`; § 7: l'eccezione `theme-literals.ts`; § 10 Commands: + `icons:generate`,
  `test:e2e:pwa`.
- `doc/guide/pwa.md` (≤ 250 righe): manifest, SW, push, iOS, come ruotare le chiavi VAPID (invalida
  tutte le sottoscrizioni: l'owner riattiva dai telefoni), come testare su telefono, il canale
  `CLEAR_PAGES`.
- `SETUP.md`: § Notifications (generazione delle VAPID, le quattro env, iOS dalla Home, rotazione).
- `README.md` (inglese): `## Install on your phone`.
- `WORKFLOW.md § 3`: i cancelli diventano `pnpm typecheck && pnpm lint && pnpm test && pnpm build`,
  poi `pnpm test:e2e`, poi `pnpm test:e2e:pwa` (una riga); riga Commands: `icons:generate`.
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/pwa.md` (≤ 250).

## 10. Fuori ambito, dichiarato

- Badge dell'icona, notifiche silenziose, Declarative Web Push → CLAUDE.md § Not yet present (righe
  presenti).
- Offline reale (code di scrittura): l'app non scrive offline (solo lettura in cache) → CLAUDE.md
  § Not yet present (riga presente).
- `pushsubscriptionchange` → CLAUDE.md § Not yet present (riga in § 9).
- Intercettare `beforeinstallprompt` su Android: Chrome mostra il suo prompt (§ 3 f).

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/07-pwa-e-web-push.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, DESIGN.md, doc/guide/app.md, doc/guide/deploy.md, SETUP.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 07 INTERA
- Verifica con un grep i contratti che usa (isPublicPath e PUBLIC_PATHS, sendReviewNotification e
  PushReport con i due chiamanti reviews/run/route.ts e run-cron.ts, ReviewRun,
  listProposalsOfReview, sessionDenied, getDb, EnvSchema in env-schema.ts, createMemoryDb,
  nanoid, le migrazioni 0000 e 0001, playwright.config.ts con i progetti setup/mobile/desktop,
  gli script e2e:server e test:e2e, tsconfig.json)
- Carica la skill impeccable per l'icona e la sezione Notifiche: DESIGN.md è la legge

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/07-pwa-push. Crea SESSION_NOTES.md.

Implementa la spec 07 esattamente come scritta: dipendenze (versioni esatte lette con npm view e
scritte in T-2), tsconfig e .gitignore, manifest, icone e layout, sw.ts, payload e ui-state (puri,
testati), repository e route, migrazione 0002_push, send.ts che sostituisce il no-op con la stessa
firma, E2E su build (progetto pwa, webServer ad array), § 7 collaudo fase per fase con l'esito
atteso prima: A, C, C2, D, E in locale; poi i passi di produzione H(1) migrazione remota e H(2) env
VAPID uno per messaggio e dopo il mio OK, PRIMA del merge; il merge fa il deploy; i due telefoni li
guardo io dopo. Se un punto non è coperto, fermati e chiedimelo con AskUserQuestion. Le
etichette della serie non entrano nel codice.

Prima del commit: riassumi il diff, aggiorna la riga «Stato» della spec («implementata»), § 5
dell'indice e CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo) e, dopo, per il
merge.
```
