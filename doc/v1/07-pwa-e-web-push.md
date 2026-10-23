# 07 — PWA e Web Push

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 06.
> **Branch**: `feature/07-pwa-push` da `main`. **Dipende da**: 05a (shell, `isPublicPath`,
> container), 05b (`sendReviewNotification` no-op), 06 (produzione HTTPS, cron, `APP_URL`).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-12, D-31; scelte T-4, T-14, T-17;
> contratti C-7 (`push_subscriptions`), C-16 (lo riempie).

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
     VAPID, `sendNotification` → 404/410 = cancellare.
   - Next 16 (`https://nextjs.org/docs/app/api-reference/functions/generate-metadata`,
     `.../file-conventions/metadata/manifest`): `app/manifest.ts`, `appleWebApp` emette
     `mobile-web-app-capable`, `app/apple-icon.png` → `apple-touch-icon`.
   - Serwist (`https://serwist.pages.dev/docs/next/getting-started`, esempio `next-basic`): `swSrc`
     `app/sw.ts`, `swDest` `public/sw.js`, `next build --webpack` (già in uso, T-4).
3. iOS: il permesso va chiesto da un tap; la sottoscrizione si perde se l'app viene rimossa dalla
   Home; non c'è modo di rilevarlo dal server se non il 410.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `/manifest.webmanifest` (da `src/app/manifest.ts`): `name` «Hevy Coach», `short_name` «Coach», `id: '/'`, `start_url: '/'`, `display: 'standalone'`, `lang: 'it'`, `background_color`/`theme_color` dai token di DESIGN.md, icone 192, 512 e 512 `maskable` in `public/icons/`; `apple-icon.png` 180; `appleWebApp` nel layout; è pubblico (`isPublicPath`). | E2E `pwa.spec.ts` (build di produzione): GET manifest → JSON con i campi; `<link rel="manifest">` e `apple-touch-icon` nell'HTML di `/login`. |
| A2 | `public/sw.js` generato da `src/app/sw.ts` con Serwist: precache degli asset, `NetworkFirst` sulle pagine, `NetworkOnly` su `/api/*`, fallback `/offline` precaricato, **listener `push`** (mostra sempre una notifica: titolo/corpo/url dal payload, default se manca) e **`notificationclick`** (focus di una finestra aperta o `openWindow(url)`); registrato da `<SwProvider>` in produzione; `disable` in dev. | E2E: `/sw.js` servito con `content-type` JS e contiene `push` e `notificationclick`; `/offline` risponde 200. Unit: `sw-payload.test.ts` sul parser del payload. |
| A3 | Tabella `push_subscriptions` (migrazione `0002_push`): `endpoint` unico, `p256dh`, `auth`, `user_agent`, `created_at`, `last_success_at`, `last_error`. `POST /api/push/subscribe` (guard) con body `{ subscription: PushSubscriptionJSON }` validato con zod → upsert per `endpoint`; `DELETE` con `{ endpoint }` → delete. | Unit + DB sulle route; body malformato → 400. |
| A4 | `/impostazioni` › «Notifiche»: stato (non supportato / iOS non installata → istruzioni «Condividi → Aggiungi alla schermata Home» / permesso negato / attive / disattive), interruttore che su tap chiede il permesso e sottoscrive con `NEXT_PUBLIC_VAPID_PUBLIC_KEY`, poi `POST`; bottone «Invia notifica di prova» → `POST /api/push/test` → una push «Notifica di prova» a tutte le sottoscrizioni. | E2E: lo stato «non supportato» in Playwright headless senza push (il test verifica il testo e che l'interruttore sia disabilitato); test unit della macchina a stati `pushUiState(...)`. |
| A5 | `sendPushToAll` (C-16): `web-push` con VAPID (`VAPID_SUBJECT` `mailto:` o `https://`), payload JSON `{ title, body, url, tag }` ≤ 4 KB, `TTL: 86400`, `urgency: 'normal'`; 404/410 → riga cancellata; altri errori → `last_error`; successo → `last_success_at`. `sendReviewNotification` compone i tre testi di T-17 e `url = APP_URL + '/revisioni/' + id` (o `/`). | `send.test.ts` con `web-push` finto (iniettato): 2 sottoscrizioni, una risponde 410 → cancellata, l'altra ok; i tre testi per `completed` (N proposte), `skipped`, `failed`. Mutazione: non cancellare sul 410 → rosso. |
| A6 | Il cron (06) e il run manuale (05b) chiamano la vera `sendReviewNotification`: nessun codice cambia lì (stessa firma), il no-op sparisce. | Grep: una sola definizione di `sendReviewNotification`. |
| A7 | Installazione e push funzionano su Android (Chrome) e su iPhone (Safari, dalla Home) con l'app in produzione; la notifica di prova arriva; al sabato arriva quella vera. | Collaudo § 7 e misura di chiusura. |

## 3. Decisioni

- **D-12** (Web Push), **D-31/T-17** (testi), **T-4** (`--webpack`), **T-14** (nome, tema).
- Adottate qui, ribaltabili: (a) icona: un monogramma «HC» generato da `public/icon.svg` con
  `scripts/generate-icons.ts` (sharp) nelle misure 192, 512, 512 maskable, 180 (apple); il disegno
  lo decide la direzione di DESIGN.md; (b) `web-push@3.6.7` (ultima release 2024-01, repo attivo:
  è ciò che la doc Next usa); (c) una sola «utenza»: tutte le sottoscrizioni ricevono tutto;
  (d) `tag` = `review-<id>` per non duplicare la stessa notifica su un dispositivo; (e) nessun
  badge (`setAppBadge`) in v1; (f) il prompt di installazione Android (`beforeinstallprompt`) **non**
  si intercetta: Chrome mostra il suo; su iOS l'istruzione testuale; (g) `SwProvider` registra
  `/sw.js` con `updateViaCache: 'none'` e, su `waiting`, mostra «Aggiornamento disponibile» con
  «Ricarica» (`SKIP_WAITING`), come segnaprezzi; (h) in dev il SW è disattivato: la PWA si prova
  solo su `pnpm build && pnpm start` e in produzione; (i) la notifica di `failed` non include
  l'errore completo, solo i primi 80 caratteri.

## 4. La soluzione

### 4.1 Dipendenze e configurazione

```bash
pnpm add @serwist/next@9.5.13 web-push@3.6.7
pnpm add -D serwist@9.5.13 @types/web-push@3 sharp@0.35 png-to-ico@3
```

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

Env: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` (`pnpm exec web-push generate-vapid-keys --json`),
`VAPID_SUBJECT` (`mailto:<indirizzo dell'owner>`, Secret; **mai `https://localhost`**: Safari
risponde `BadJwtToken`), `NEXT_PUBLIC_VAPID_PUBLIC_KEY` (= la pubblica, Config). `EnvSchema` li
aggiunge; `.env.example` idem.

### 4.2 Manifest e icone

`src/app/manifest.ts`:

```ts
import type { MetadataRoute } from 'next';
export default function manifest(): MetadataRoute.Manifest {
  return {
    id: '/', name: 'Hevy Coach', short_name: 'Coach', description: 'Il coach del tuo allenamento a corpo libero, sopra Hevy.',
    start_url: '/', display: 'standalone', lang: 'it', dir: 'ltr',
    background_color: '<token DESIGN.md>', theme_color: '<token DESIGN.md>',
    icons: [
      { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/icons/icon-512-maskable.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ],
  };
}
```

`src/app/layout.tsx`: `metadata.appleWebApp = { capable: true, title: 'Coach', statusBarStyle:
'black-translucent' }`, `metadata.manifest = '/manifest.webmanifest'`, `viewport.themeColor`.
`src/app/apple-icon.png` (180×180) e `src/app/icon.png` (32×32) generati dallo script.
`src/app/offline/page.tsx`: «Sei offline. L'ultima pagina aperta è in cache; riprova quando torni
in rete.» (pubblica).

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
  event.waitUntil(self.clients.matchAll({ type: 'window', includeUncontrolled: true }).then((list) => {
    const open = list.find((c) => 'focus' in c);
    return open ? open.navigate(url).then((c) => c?.focus()) : self.clients.openWindow(url);
  }));
});

self.addEventListener('message', (event) => { if (event.data?.type === 'SKIP_WAITING') self.skipWaiting(); });
serwist.addEventListeners();
```

`src/lib/push/payload.ts` (puro): `PushPayloadSchema = z.object({ title: z.string().max(80),
body: z.string().max(200), url: z.string().startsWith('/'), tag: z.string().max(40) })`,
`parsePushPayload(text)` → valido o `{ title: 'Hevy Coach', body: 'Apri l\'app', url: '/', tag: 'default' }`,
`encodePushPayload(payload)` → stringa ≤ 4096 byte (test).

### 4.4 Client — `src/components/pwa/`

- `sw-provider.tsx`: registra `/sw.js` in produzione; toast «Aggiornamento disponibile → Ricarica».
- `push-settings.tsx`: `pushUiState({ supported, standalone, isIos, permission, subscribed })` →
  `'unsupported' | 'ios_not_installed' | 'denied' | 'off' | 'on'` (puro, in
  `src/lib/push/ui-state.ts`); `isIos = /iPhone|iPad|iPod/.test(navigator.userAgent)`;
  `standalone = matchMedia('(display-mode: standalone)').matches || navigator.standalone === true`;
  tap su «Attiva» → `Notification.requestPermission()` → `registration.pushManager.subscribe({
  userVisibleOnly: true, applicationServerKey: urlBase64ToUint8Array(key) })` → `POST /api/push/subscribe`
  con `subscription.toJSON()`; «Disattiva» → `unsubscribe()` + `DELETE`; «Invia notifica di prova».
- `src/lib/push/url-base64.ts`: `urlBase64ToUint8Array` (test).

### 4.5 Server — `src/lib/push/send.ts` (C-16) e route

```ts
export interface PushSender { send(subscription: WebPushSubscription, payload: string, options: { TTL: number; urgency: 'normal' }): Promise<{ statusCode: number }> }
export function createWebPushSender(env: { VAPID_SUBJECT; VAPID_PUBLIC_KEY; VAPID_PRIVATE_KEY }): PushSender   // the only file importing 'web-push'
export async function sendPushToAll({ db, sender, payload, now }): Promise<PushReport>   // PushReport = { sent; removed; failed }
export async function sendReviewNotification({ db, review, sender = createWebPushSender(env), now = Date.now() }): Promise<PushReport>
```

Testi (T-17): `completed` → «Revisione pronta» / «N proposte da guardare» (`N` = proposte non
`insufficient_data`; se 0: «Nessuna proposta: tutto resta com'è»), url `/revisioni/<id>`; `skipped`
`no_workouts` → «Revisione saltata» / «Nessun allenamento nelle ultime 4 settimane», url `/`;
`skipped` `already_running` → nessuna notifica (ritorna `{ sent: 0 }`); `failed` → «Revisione
fallita» / `error.slice(0, 80)`, url `/revisioni/<id>`.

Route: `src/app/api/push/subscribe/route.ts` (`POST`/`DELETE`, guard, zod
`PushSubscriptionJsonSchema = z.object({ endpoint: z.string().url(), expirationTime: z.number().nullable().optional(), keys: z.object({ p256dh: z.string(), auth: z.string() }) })`),
`src/app/api/push/test/route.ts` (`POST`, guard → `sendPushToAll` con «Notifica di prova» / «Se la
leggi, funziona.» / `/impostazioni`).

Migrazione `0002_push`:

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

`playwright.config.ts` aggiunge il progetto `pwa` con `webServer: { command: 'pnpm e2e:prod', port:
3200 }` (`"e2e:prod": "cross-env <stesse env E2E> PORT=3200 sh -c 'next build --webpack && next start -p 3200'"`,
su Windows via `node scripts/e2e-prod.mjs` che fa le due cose) e lo spec `pwa.spec.ts` (A1, A2). Il
progetto `pwa` gira solo con `pnpm test:e2e:pwa` (lento), e nel cancello di chiusura.

## 5. Vincoli e trappole

- **AGENTS.md § PWA** (nasce qui) — qui vuol dire che: il SW si prova solo su build; `/sw.js` e il
  manifest sono pubblici in `isPublicPath`; mai rinominare `sw.js` (Safari 27 cancella le
  registrazioni con script mancante); `web-push` si importa in un solo file; il payload è ≤ 4 KB
  per costruzione (`encodePushPayload` lo garantisce con un test).
- iOS: il tap deve chiamare `requestPermission` **nello stesso gestore** dell'evento (niente
  `await` prima); una notifica deve **sempre** essere mostrata nel `push` (altrimenti Safari revoca).
- `next build --webpack` è già il comando (T-4); se Next stampa l'avviso sul plugin webpack, è
  atteso.
- `public/sw.js` è generato e ignorato da git.
- Il fake E2E non ha push: lo stato `unsupported` è l'unico testato in automatico; il resto è
  collaudo sul telefono (dichiarato).

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `payload.test.ts`: parse valido, testo non JSON → default, url non relativa → default; `encodePushPayload` lancia se > 4096 byte. | Togliere il limite → rosso. |
| Unit | `ui-state.test.ts`: tabella di verità dei 5 stati (iOS non standalone → `ios_not_installed` anche con permesso `granted`). | — |
| Unit | `url-base64.test.ts`: chiave VAPID d'esempio → 65 byte. | — |
| Unit + DB | `send.test.ts`: sender finto; 410 → riga cancellata; 500 → `last_error`; 201 → `last_success_at`; i quattro casi di `sendReviewNotification`. | — |
| Unit + DB | `subscribe-route.test.ts`: upsert per endpoint (2 POST stesso endpoint → 1 riga), DELETE, 400. | — |
| E2E (`pwa`) | manifest, `sw.js`, `/offline`, `/impostazioni` stato `unsupported`. | — |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E dei progetti `mobile`/`desktop` verdi; `pnpm test:e2e:pwa` verde.
- **C — Locale su build**: `pnpm build && pnpm start`; Chrome desktop: Application → Manifest
  senza errori, Service Worker attivo, «Install» disponibile.
- **D — Produzione (dopo il merge)**: `pnpm db:migrate` remoto (`0002`); env VAPID aggiunte su
  Vercel; deploy; `curl https://<prod>/manifest.webmanifest`.
- **F — Giro guidato (owner, due telefoni)**: Android/Chrome: (1) banner o menu «Installa»; (2)
  app dalla Home, `/impostazioni` → «Attiva» → permesso → «Attive»; (3) «Invia notifica di prova»
  → arriva, il tap apre `/impostazioni`. iPhone/Safari: (4) Condividi → Aggiungi alla schermata
  Home, apri dalla Home, `/impostazioni` → lo stato NON dice più «installa», «Attiva» → permesso →
  «Attive» → prova → arriva. (5) Da Safari (non installata) lo stato dice «aggiungi alla Home».
  Il bug sarebbe: nessuna notifica entro 10 s, un tap che non apre l'app, Safari che revoca il
  permesso (notifica non mostrata).
- **G — Ripristino**: le sottoscrizioni di prova restano (sono quelle vere). Nessuna fixture.

Che cosa NON copre: la prima notifica del cron vero (misura di chiusura).

## 8. Messa in esercizio

- Env Production: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` (Secret),
  `NEXT_PUBLIC_VAPID_PUBLIC_KEY` (Config). DDL: `0002_push` remoto prima del deploy.
- Misura di chiusura: al primo sabato con revisione dovuta arriva la push «Revisione pronta» su
  entrambi i telefoni; `push_subscriptions.last_success_at` aggiornato.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Key Features (PWA, Notifiche); § Not yet present (badge, preview).
- `AGENTS.md`: § PWA (stub → guida). `doc/guide/pwa.md` (≤ 250 righe): manifest, SW, push, iOS,
  come ruotare le chiavi VAPID (invalida tutte le sottoscrizioni), come testare su telefono.
- `SETUP.md`: § Notifications (VAPID, iOS).
- `README.md`: § Install on your phone.

## 10. Fuori ambito, dichiarato

- Badge dell'icona, notifiche silenziose, Declarative Web Push → CLAUDE.md § Not yet present.
- Offline reale (code di scrittura): l'app non scrive offline (solo lettura in cache) → CLAUDE.md
  § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/07-pwa-e-web-push.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, DESIGN.md, doc/guide/app.md, doc/guide/deploy.md, SETUP.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 07 INTERA
- Verifica con un grep i contratti che usa (isPublicPath, sendReviewNotification e i suoi due
  chiamanti, sessionDenied, getDb, env/EnvSchema, le migrazioni 0000 e 0001)
- Carica la skill impeccable per l'icona e la sezione Notifiche: DESIGN.md è la legge

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/07-pwa-push. Crea SESSION_NOTES.md.

Implementa la spec 07 esattamente come scritta: dipendenze e config, manifest e icone, sw.ts,
payload e ui-state (puri, testati), route e migrazione, send.ts che sostituisce il no-op,
E2E su build, § 7 collaudo fase per fase con l'esito atteso prima — i passi in produzione
(migrazione, env VAPID, deploy) uno per messaggio e dopo il mio OK; i due telefoni li
guardo io. Se un punto non è coperto, fermati e chiedimelo con AskUserQuestion. Le
etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo) e per il merge.
```
