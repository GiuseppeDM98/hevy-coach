# Serie «v1» — indice

> Prima serie di spec di questo repo (cartella nata il 2026-10-23, decisione D-9). Convenzione:
> una spec = una sessione di implementazione = un branch = un commit; le spec sono SOLO
> istruzioni di sviluppo (chi implementa non decide niente); lo stato vive qui (§ 5),
> nell'intestazione di ogni spec e in `CLAUDE.md § Current Status`, e si chiude con un grep
> (§ 7). Non c'è codice al 2026-10-23: le prove sono i payload reali di Hevy (fixture in
> `tests/fixtures/hevy/`, letti il 2026-10-23) e la documentazione dei servizi, citata con la
> data di lettura. **Lettura adversariale: non ancora fatta** — va fatta con il prompt
> `claude-config/prompts/07_LETTURA_ADVERSARIALE_SPEC.md` PRIMA di implementare la 01
> (tetto di agenti della sessione di pianificazione: 3, di cui 1 speso in ricerca).

## 1. Il tema, con le parole dell'owner (2026-10-23)

> «Un coach personale per il mio allenamento a corpo libero, costruito sopra Hevy (l'app con cui
> registro gli allenamenti) e sull'API di Anthropic. Uso singolo: solo io. Le 5 routine settimanali
> vivono su Hevy, in una cartella dedicata; l'app le crea e le aggiorna via API. Gli allenamenti li
> registro su Hevy, NON nell'app. Ogni due settimane l'app scarica da Hevy gli allenamenti, li aggrega
> per esercizio, chiede a Claude una revisione (per esercizio: mantieni / aumenta ripetizioni / passa
> alla variante successiva, con una motivazione breve) e mi avvisa. Nella dashboard vedo i progressi
> per esercizio, lo storico delle revisioni e le proposte aperte; con «Applica» (sempre conferma
> manuale, mai applicazione automatica) l'app aggiorna le routine su Hevy. È una PWA installabile sul
> telefono come un'app normale (Android e iPhone). Esercizi solo a corpo libero, in casa, senza
> attrezzi, SILENZIOSI (la casa deve restare silenziosa la sera: niente salti, niente esercizi che fanno
> tonfi) e semplici, niente tecnica complessa. Allenamento Lun–Ven la sera, 30–40 min a sessione
> (mercoledì più leggero, ~25 min). Progressione: doppia progressione (chiuse tutte le serie al
> massimo del range → variante più difficile della stessa scala).»

Il piano di partenza (5 routine; riscaldamento 5' in ogni sessione; «max-2» = fino a 2 ripetizioni
dal cedimento) è trascritto come dato dell'app nella spec 02 § 4.1, esercizio per esercizio.

| # | Punto | Spec |
|---|---|---|
| P1 | Le 5 routine su Hevy, in una cartella dedicata, il prima possibile e anche da riga di comando | 01, 02 |
| P2 | Scaricare gli allenamenti da Hevy e tenerli in Turso | 03 |
| P3 | Aggregare per esercizio e chiedere a Claude la revisione bisettimanale | 04 |
| P4 | «Applica» con conferma manuale: la routine su Hevy cambia | 04 (CLI), 05b (UI) |
| P5 | Dashboard: progressi per esercizio, storico delle revisioni, proposte aperte | 05a, 05b |
| P6 | Cron bisettimanale, «Esegui revisione ora», webhook di Hevy | 04, 05b, 06 |
| P7 | PWA installabile con notifiche Web Push | 07 |
| P8 | Tutto dietro autenticazione; segreti mai nel repo | 05a, 06 |

## 2. Analisi del 2026-10-23 — che cosa si sa (il dettaglio sta in ogni spec § 1)

### 2.1 Hevy (OpenAPI 0.0.1, estratto da `https://api.hevyapp.com/docs/swagger-ui-init.js` il 2026-10-23; chiamate GET reali lo stesso giorno)

- Autenticazione: header `api-key` (uuid), solo utenti Pro (`GET /v1/user/info` → `is_pro: true`
  sull'account dell'owner). La descrizione dell'API chiede di **non chiamare esattamente a xx:00**.
- Paginazione: `page` (≥ 1) e `pageSize`, con massimo **10** per `routines`, `routine_folders`,
  `workouts`, `workouts/events` e **100** per `exercise_templates`. La risposta porta `page` e
  `page_count`. L'account ha 464 template (5 pagine), 13 custom, **174 workout** (18 pagine),
  **nessuna routine e nessuna cartella**.
- Set di una routine: `type` (`warmup|normal|failure|dropset`), `weight_kg`, `reps`, `distance_meters`,
  `duration_seconds`, `custom_metric`, **`rep_range {start,end}`** (tutti nullable). Non esiste un
  marcatore AMRAP/«max-2». Nei workout i set hanno anche `rpe`.
- `POST /v1/routines` → 201 `{routine}`; 403 «Routine limit exceeded». `PUT /v1/routines/{id}` →
  **sostituzione completa** (i campi non rimandati si perdono), 404 se la routine non è dell'utente.
  **Non esiste DELETE** per routine, cartelle e template.
- `POST /v1/exercise_templates` → `{id}` (lo schema dice integer, i custom reali hanno id UUID: la
  01 lo tratta come stringa e lo verifica con una GET); 403 `exceeds-custom-exercise-limit`. Nessun
  PUT/DELETE: un custom creato resta per sempre → creazione idempotente per titolo esatto (02).
- `POST /v1/routine_folders` → 201 `RoutineFolder` (creata a index 0).
- `GET /v1/exercise_history/{templateId}?start_date&end_date` → set piatti con `workout_id`,
  `workout_start_time`, `set_type`, `reps`, `duration_seconds`; **senza paginazione** (13 voci reali
  per Push Up). `GET /v1/workouts/events?since` → eventi `updated`/`deleted` per il sync incrementale.
- Id dei template di catalogo: 8 caratteri esadecimali (es. `392887AA` Push Up); custom: UUID.
- Rate limit: non documentato; nessun header di limite nelle risposte reali.
- Webhook (pagina Hevy «Webhooks», screenshot dell'owner 2026-10-23): POST `{ "workoutId": "<uuid>" }`
  all'URL scelto, con un header di autorizzazione scelto dall'utente; risposta 200 entro 5 s.

### 2.2 Vercel (doc ufficiale letta il 2026-10-23 — URL nelle spec 06 e 05a)

- Hobby: cron **una volta al giorno al massimo**, precisione **±59 min**, UTC, 100 job, **solo in
  produzione**, best effort (**duplicati possibili, nessun retry**), GET con UA `vercel-cron/1.0`;
  `CRON_SECRET` arriva come `Authorization: Bearer`. Functions: **300 s** (default e massimo), 2 GB,
  regione singola, log runtime conservati **1 ora**.
- Deployment Protection: Vercel Authentication gratis anche in produzione dal 2026-09-09, ma non è
  documentato che il cron la superi e blocca le automazioni → **Standard Protection** (solo preview)
  + autenticazione applicativa (D-13).
- Env: tipi Config/Secret; `vercel env pull` non rilegge i Secret; `VERCEL_PROJECT_PRODUCTION_URL`.
- Next 16: `middleware.ts` → `proxy.ts` (solo Node); `next build --webpack` necessario con
  `@serwist/next`; `cacheComponents` opt-in (non si adotta, T-4).

### 2.3 Turso (doc letta il 2026-10-23)

- **Nessuna regione Francoforte**: le regioni EU sono `aws-eu-west-1` (Dublino) e `aws-eu-north-1`
  → Vercel in **`dub1`** (T-1). Free: 100 DB, 5 GB, 500 M letture, 10 M scritture al mese.
- `@libsql/client` (entry Node) per Drizzle, `file:local.db` in locale; `batch()` preferito alle
  transazioni interattive (timeout 5 s). drizzle-kit `dialect: 'turso'`, tabella
  `__drizzle_migrations`; si pinna **0.45.x / 0.31.x** (latest), non `@rc`.

### 2.4 PWA e Web Push (MDN, WebKit, Apple, web.dev letti il 2026-10-23)

- iOS: push solo dalla PWA aggiunta alla Home (iOS ≥ 16.4), manifest `display: standalone`, permesso
  da gesto utente, **niente push silenziose** (Safari revoca il permesso). iOS 26: nessun requisito
  di installabilità. Chrome: il prompt di installazione richiede ancora un `fetch` handler nel SW.
- `web-push` 3.6.7 (ultima release 2024-01, repo attivo): 404/410 → cancellare la sottoscrizione;
  payload ≤ 4 KB. Next 16 `appleWebApp` emette `mobile-web-app-capable`.

### 2.5 Anthropic (skill `claude-api`, tabella modelli cache del 2026-10-06; prezzi `https://platform.claude.com/docs/en/about-claude/pricing.md`)

- `claude-opus-5-5`: 4 $/M input, 20 $/M output, 1 M contesto; thinking adattivo sempre attivo,
  `output_config.effort` (default `medium` → si imposta `high`); **niente prefill, niente
  `tool_choice` forzato**; output strutturato con `client.messages.parse` + `zodOutputFormat`
  (`output_config.format`); `stop_reason: "refusal"` da gestire; `fallbacks: "default"` con beta
  `server-side-fallback-2026-07-01`.

## 3. Decisioni dell'owner (D-N, con la data)

### 3.1 Vincoli dati nel tema (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-1 | Hosting Vercel (piano Hobby), database Turso. **Riscritta da D-34** (2026-10-23): il repo nasce **pubblico**, non privato. | 03, 06 |
| D-2 | La chiave Anthropic sta su un'organizzazione Console **personale** dell'owner, mai sull'account aziendale. | 04, 06 |
| D-3 | «Applica» è sempre una conferma manuale: nessuna modifica automatica alle routine. | 04, 05b |
| D-4 | Esercizi solo a corpo libero, in casa, senza attrezzi, **silenziosi** (niente salti, niente tonfi) e **semplici** (niente tecnica complessa). Le scale di varianti (02 § 4.2) rispettano il vincolo per costruzione; la revisione di Claude non può proporre nulla fuori dalle scale (04). | 02, 04 |
| D-5 | Allenamento Lun–Ven la sera, 30–40 min a sessione, mercoledì ~25 min. | 02 |
| D-6 | Il piano di partenza è quello del tema (02 § 4.1); progressione doppia: tutte le serie chiuse al massimo del range → variante più difficile della stessa scala. | 02, 04 |
| D-7 | Le scale di varianti sono **dato dell'app** (`src/lib/plan/scales.ts`), non testo nel prompt a Claude. | 02, 04 |
| D-8 | Fuori ambito della v1: registrare gli allenamenti nell'app (resta Hevy) · più utenti · il piano della garage gym · modifiche automatiche alle routine senza conferma. | tutte |

### 3.2 Bootstrap e stack (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-9 | Repo `hevy-coach`; spec in `doc/v1/` con `00-INDICE.md` (convenzione Hub: NN-slug.md, D-N/T-N/C-N, `PROMPT-RITIRO.md` generato con il prompt 08 dopo la lettura adversariale). | tutte |
| D-10 | Stack di riferimento: **segnaprezzi** — Next 16 App Router, React 19, TypeScript 7, pnpm, Turso + Drizzle, Biome, Vitest, Playwright, Serwist, `@anthropic-ai/sdk`, zod 4. | 01 |
| D-11 | Al bootstrap nascono CLAUDE.md, AGENTS.md, WORKFLOW.md, SESSION_NOTES.md; **DESIGN.md e PRODUCT.md nascono nella 05a** con la skill impeccable; fino ad allora CLAUDE.md li dichiara in § Not yet present. | 05a |
| D-12 | Notifiche: **Web Push della PWA** (le proposte aperte restano comunque visibili in dashboard). | 07 |
| D-13 | Autenticazione: **password unica** in env + **cookie firmato** di lunga durata; `proxy.ts` protegge tutto tranne `/login`, le route del cron e del webhook, il manifest, il service worker e le icone. Produzione senza Vercel Authentication; preview con Standard Protection. | 05a, 06 |
| D-14 | Modello per la revisione: **Claude Opus 5.5** (`claude-opus-5-5`), effort `high`. | 04 |
| D-15 | Finestra inviata a Claude: **28 giorni** più l'esito della revisione precedente. | 04 |
| D-16 | Cron settimanale **sabato ~08:17 ora di Roma** → `"17 6 * * 6"` (UTC); il codice esegue la revisione solo se sono passati **≥ 13 giorni** dall'ultima completata. | 06 |
| D-17 | Dashboard con **grafici completi (Recharts)**. | 05a |
| D-21 | Lingua di CLAUDE.md, AGENTS.md, WORKFLOW.md: **inglese**; spec, chat e UI in italiano; codice e commenti in inglese. | tutte |
| D-23 | **main = produzione**; una PR per spec da `feature/*` a `main`; niente `develop`. Le preview dei branch hanno Vercel Authentication (Standard Protection). | tutte, 06 |
| D-34 | Il repo è **open source, AGPL-3.0** (ribalta la parte «privato» di D-1): un coach a utente singolo, self-hosted; LICENSE al bootstrap, SETUP.md completo nella 06, README con il disclaimer «non affiliato a Hevy»; WORKFLOW.md § 1.6 senza eccezioni (nessun dato dell'owner nei file tracciati, nei commit e nelle PR). | tutte, 06 |
| D-35 | README.md in **inglese**; i nomi del piano e degli esercizi restano italiani (etichette del prodotto), con una riga che lo dice. | tutte |

### 3.3 Rappresentazione su Hevy (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-18 | «max-2»: set con `reps` e `rep_range` **null** e nota Hevy sull'esercizio «max-2: fermati a 2 ripetizioni dal cedimento»; la **soglia numerica di passaggio** (ripetizioni su tutte le serie) sta nella scala (`Variant.threshold`). | 02, 04 |
| D-19 | Tenute con range (Wall sit 45–60", Hollow hold 20–30"): `duration_seconds` = **limite inferiore**, nota con il range; progressione: tutte le serie al limite superiore → variante successiva. | 02, 04 |
| D-20 | Esercizi per lato: `reps` = ripetizioni **per lato** (es. 10) + nota «per gamba» / «per lato». | 02 |
| D-24 | Riscaldamento (5') nelle **note della routine**; mobilità del mercoledì = **3 esercizi a tempo** (90/90 anche, couch stretch, rotazioni toraciche), registrati ma **esclusi dalla revisione**. | 02, 04 |
| D-25 | Cartella Hevy **«Coach · Corpo libero»**; titoli **«Lun · Upper A», «Mar · Lower A», «Mer · Core + mobilità», «Gio · Upper B», «Ven · Lower B»**. | 02 |
| D-26 | Gli esercizi custom hanno titolo **in italiano**, come nel piano; quelli del catalogo tengono il titolo Hevy. Un custom che esiste già con lo stesso titolo esatto si riusa («Hollowbody Hold», «Couch Stretch»). | 02 |
| D-27 | Recuperi: **90 s** sui multiarticolari (push-up, rematore, pike, squat, affondi, bulgarian, stacco rumeno), **60 s** sugli altri, **45 s** sul core, **0** sulla mobilità. | 02 |

### 3.4 Revisione e «Applica» (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-28 | «Aumenta ripetizioni» su Hevy = **range +2/+2** (es. 8–12 → 10–14), ripetizioni fisse +2, **tenute +10 s** (nota aggiornata), max-2: **soglia +2**. Il numero di serie non cambia mai. | 04 |
| D-29 | «Applica» per **singola proposta**, con **«Applica tutte»** per le aperte; ogni applicazione = GET routine + PUT completo, una routine alla volta. | 04, 05b |
| D-30 | «Esegui revisione ora» **parte sempre** (la regola dei 13 giorni vale solo per il cron); le proposte aperte della revisione precedente diventano `superseded`. | 04, 05b |
| D-31 | Dati scarsi, **regola in codice**: un esercizio con < 2 sessioni nella finestra riceve «mantieni (dati insufficienti)» senza passare da Claude; con 0 allenamenti nella finestra la revisione è `skipped` e la notifica dice «nessun allenamento nelle ultime 4 settimane». | 04 |
| D-22 | Esiti possibili per esercizio: **solo** `keep` / `increase` / `next_variant`. Tornare indietro di variante si fa a mano su Hevy (CLAUDE.md § Not yet present). | 04 |
| D-32 | Il **webhook di Hevy** entra nella v1: `POST /api/hevy/webhook` verifica `HEVY_WEBHOOK_SECRET`, risponde 200 entro 5 s e sincronizza quel workout; il sync via `workouts/events` resta nel cron come rete di sicurezza. L'iscrizione si fa dopo il deploy (06). | 03, 06 |
| D-33 | L'app sincronizza **tutti** i workout dell'account (prima volta: 18 pagine; poi solo gli eventi dal cursore); l'aggregazione usa solo gli esercizi del piano, per `exercise_template_id`, quindi anche gli allenamenti a corpo libero fatti prima del piano contano. | 03, 04 |

## 4. Scelte adottate dal pianificatore («adottata, ribaltabile» — rovesciabili prima dell'implementazione)

| # | Scelta | Perché | Spec |
|---|---|---|---|
| T-1 | Regione Vercel **`dub1`** e gruppo Turso `aws-eu-west-1` (Dublino). | Turso non ha Francoforte (2.3); stessa regione = latenza minima. | 06 |
| T-2 | Versioni pinnate (2026-10-23): next 16.4.2, react 19.3.1, typescript 7.0.2, drizzle-orm 0.45.4, drizzle-kit 0.31.11, @libsql/client 0.18.0, @serwist/next + serwist 9.5.13, @anthropic-ai/sdk 1.1.0, zod 4.6.7, web-push 3.6.7, vitest 5.0.5, @playwright/test 1.64.1, @biomejs/biome 2.6.2, tsx 4.23.15, tailwindcss 4.3.3, recharts (ultima 3.x al momento della 05a), nanoid 6.0.2, jose (ultima 6.x). `packageManager: pnpm@11`, `engines.node >= 22`. | Le versioni di segnaprezzi erano già a un minor indietro; `npm view` del 2026-10-23. | 01 |
| T-3 | `@libsql/client` con l'entry **Node** (non `/web`): su Vercel le function sono Node (Fluid), e solo l'entry Node apre `file:local.db`. Client singleton su `globalThis` come segnaprezzi. | segnaprezzi lo fa già in produzione. | 03 |
| T-4 | Next 16 **senza** `cacheComponents`, dev con Turbopack, **`next build --webpack`** (Serwist). React Compiler spento. | Serwist richiede webpack su Next 16 (2.2); nulla da prerenderizzare. | 01, 07 |
| T-5 | Layout `src/`: `src/app` (route), `src/components`, `src/lib/{hevy,plan,db,sync,review,apply,auth,push,env.ts}`, `scripts/` (CLI con tsx), `tests/fixtures/`, test accanto ai moduli (`*.test.ts`). | Come segnaprezzi. | 01 |
| T-6 | Qualità: Biome (lint + format, config di segnaprezzi), `tsc --noEmit`, Vitest; Playwright dalla 05a. Script: `dev`, `build`, `start`, `lint`, `lint:fix`, `typecheck`, `test`, `test:watch`, `test:e2e`, `db:generate`, `db:migrate`, `db:studio`, `hevy:upload`, `hevy:sync`, `review:run`, `review:apply`. | Un solo set di comandi per tutta la serie. | 01+ |
| T-7 | I template custom si creano **pigramente**: all'upload solo le varianti presenti nelle routine di partenza (9 custom, 02 § 4.4), e le varianti successive quando «Applica» le raggiunge (`ensureTemplates`, C-16). | I custom non si cancellano; il limite di Hevy (403) non è noto. | 02, 04 |
| T-8 | L'identità di uno slot del piano è lo `slotId` (es. `mon_push_up`); lo stato corrente dello slot (variante, target) vive in Turso (`slot_state`); la storia in `slot_state_history`. L'aggregazione attribuisce un set a uno slot **prima per il `routine_id` del workout** (→ `RoutineKey`), poi per la variante attiva alla data del workout (storia) il cui template è quello del set; una variante può stare in due slot (`hollow_hold` mercoledì e giovedì): senza `routine_id` e con più candidati il set finisce in `unresolved` (04 § 4.2). | Senza storia non si sa quale variante era attiva in una data; il `routine_id` c'è sempre quando si parte dalla routine. | 03, 04 |
| T-9 | Oltre a Claude, il codice calcola `ruleAction` (la doppia progressione in forma deterministica, C-12) per ogni esercizio; la proposta salva entrambi e la dashboard mostra quando divergono. Claude riceve `ruleAction` nel prompt come suggerimento, non come vincolo. | Verificabilità: un test prova la regola senza chiamare Claude. | 04, 05b |
| T-10 | Chiamata a Claude: `client.messages.parse` con `zodOutputFormat(ReviewResponseSchema)`, `max_tokens: 8000`, `output_config: { effort: 'high', format }`, timeout 240 s, `maxRetries: 2` dell'SDK sugli errori di rete, **un** retry applicativo sull'errore di validazione; **niente `fallbacks`**: un `stop_reason` `refusal` o `max_tokens` rende la revisione `failed`. Costo stimato salvato in `reviews` (token × prezzo di listino in `src/lib/review/pricing.ts`). Una revisione = una chiamata riuscita (+1 retry). | Costo e durata prevedibili (300 s di Vercel); i fallback aggiungono una beta per un caso che qui vale un `failed` leggibile. | 04 |
| T-11 | Lock della revisione: riga `reviews` con `status='running'` più giovane di 10 min → la nuova esecuzione esce con `skipped: 'already_running'`. Serve contro i duplicati del cron Hobby. | 2.2 | 04, 06 |
| T-12 | Route: `GET /api/cron/review` (cron, Bearer `CRON_SECRET`), `POST /api/hevy/webhook`, `POST /api/reviews/run` (manuale), `POST /api/proposals/[id]/apply`, `POST /api/proposals/apply-all`, `POST /api/sync`, `POST /api/push/subscribe`, `DELETE /api/push/subscribe`. Pagine: `/login`, `/` (Oggi: proposte aperte, stato), `/progressi`, `/revisioni`, `/revisioni/[id]`, `/impostazioni`. | Superfici minime per P4–P7. | 05a, 05b, 06, 07 |
| T-13 | Cookie `hc_session`, JWT HS256 (`jose`) firmato con `SESSION_SECRET`, `httpOnly`, `secure`, `sameSite: 'lax'`, durata **365 giorni**; login con confronto a tempo costante contro `APP_PASSWORD` e attesa di 1 s sul fallimento. | Una PWA che chiede la password ogni settimana non si usa. | 05a |
| T-14 | UI in italiano, nome dell'app **«Hevy Coach»**, `short_name` «Coach», tema scuro di default; colori e tipografia li decide la 05a con impeccable. | D-21, D-9. | 05a, 07 |
| T-15 | Timezone di calendario **Europe/Rome** per sessioni, finestre e «13 giorni»; in DB gli istanti sono epoch ms UTC. | Come segnaprezzi. | 03, 04 |
| T-16 | Prima revisione: il cron la esegue il primo sabato in cui sono passati ≥ 13 giorni dalla data di caricamento delle routine (`sync_state.plan_uploaded_at`, che la 03 ricava dal `created_at` minimo delle 5 routine su Hevy e scrive una volta sola). | Senza una revisione precedente la regola dei 13 giorni non avrebbe riferimento; la 02 non ha un DB. | 03, 04, 06 |
| T-17 | La notifica push ha tre testi: «Revisione pronta: N proposte» (completed), «Revisione saltata: nessun allenamento nelle ultime 4 settimane» (skipped), «Revisione fallita: <errore breve>» (failed). Click → `/revisioni/[id]` (o `/` se manca). | D-12, D-31. | 07 |
| T-18 | Le ripetizioni **fisse** del piano (affondo 10, glute bridge 12, calf 15, Y-T-W 8, push-up con pausa 8, superman 12, snow angel 12, abduzioni 15) diventano range di 4 (`n`–`n+4`; calf e abduzioni `15`–`20`). | La doppia progressione (D-6) ha bisogno di un massimo da chiudere; il minimo resta il numero del piano. | 02 |
| T-19 | `git.deploymentEnabled` solo `main`: nessun deploy di preview per i branch `feature/*` (collaudo in locale); Standard Protection resta impostata. | Preview inutili per un'app a utente singolo; meno superfici esposte. | 06 |
| T-20 | I grafici dei progressi usano una finestra di 56 giorni (8 settimane), con la stessa `aggregateWindow`. | Due cicli di revisione e mezzo leggibili in un grafico. | 05a |
| T-21 | Container dei servizi (`src/lib/container.ts`, C-18): `getDb()`, `getHevyClient()`, `getReviewModel()`; con `E2E_FAKE_SERVICES=1` restituisce fake in memoria. L'app usa solo il container; gli script CLI usano i client direttamente. | Playwright non deve toccare Hevy né Anthropic. | 05a, 05b, 06, 07 |
| T-22 | Il webhook risponde 202 subito e sincronizza in `after()`; il cron sincronizza sempre, anche quando la revisione non è dovuta. | Limite di 5 s di Hevy; dashboard aggiornata al sabato anche senza webhook. | 06 |

## 5. Ordine, dipendenze e stato

| Spec | Titolo | Branch | Dipende da | Stato |
|---|---|---|---|---|
| 01 | Scaffold e client Hevy | `feature/01-scaffold-hevy-client` | — | scritta 2026-10-23 |
| 02 | Piano, scale di varianti, mappa esercizi e caricamento delle routine da CLI | `feature/02-plan-upload` | 01 | scritta 2026-10-23 |
| 03 | Turso: schema, migrazioni e sincronizzazione degli allenamenti | `feature/03-turso-sync` | 02 | scritta 2026-10-23 |
| 04 | Aggregazione, revisione con Claude e «Applica» da CLI | `feature/04-review` | 03 | scritta 2026-10-23 |
| 05a | Web app: accesso, shell, design e pagina Progressi | `feature/05a-app-progressi` | 04 | scritta 2026-10-23 |
| 05b | Web app: revisioni, proposte, «Applica» ed «Esegui revisione ora» | `feature/05b-app-revisioni` | 05a | scritta 2026-10-23 |
| 06 | Messa in esercizio: Vercel, Turso remoto, cron e webhook | `feature/06-deploy` | 05b | scritta 2026-10-23 |
| 07 | PWA e Web Push | `feature/07-pwa-push` | 06 | scritta 2026-10-23 |

Vincolo di priorità (P1): dopo la 02 le routine sono su Hevy e l'owner si allena; dalla 04 la
revisione gira da CLI; dalla 06 gira da sola. Una spec parte solo quando la precedente è **in
esercizio** (§ 7). La prima da incollare è il prompt in fondo alla 01.

## 6. Contratti condivisi fra spec (da usare IDENTICI: nome, firma, file)

| # | Contratto | File | Nasce in | Usato da |
|---|---|---|---|---|
| C-1 | `createHevyClient({ apiKey, fetch?, baseUrl? }): HevyClient` con `getUserInfo()`, `listExerciseTemplates()` (tutte le pagine), `getExerciseTemplate(id)`, `createCustomExercise(input)`, `listRoutineFolders()`, `createRoutineFolder(title)`, `listRoutines()` (tutte le pagine), `getRoutine(id)`, `createRoutine(body)`, `updateRoutine(id, body)`, `countWorkouts()`, `listWorkouts(page)`, `getWorkout(id)`, `listWorkoutEvents({ since, page })`, `getExerciseHistory(templateId, { startDate, endDate })`; errori `HevyApiError { status, code, body }` | `src/lib/hevy/client.ts` | 01 | 02, 03, 04, 06 |
| C-2 | Tipi e schemi zod: `HevyUserInfo`, `HevyExerciseTemplate`, `HevyRoutineFolder`, `HevyRoutine`, `HevyRoutineExercise`, `HevyRoutineSet`, `HevyWorkout`, `HevyWorkoutExercise`, `HevyWorkoutSet`, `HevyWorkoutEvent`, `HevyExerciseHistoryEntry`, `RoutineWriteBody`, `CustomExerciseInput` | `src/lib/hevy/schemas.ts` (zod) e `src/lib/hevy/types.ts` (tipi inferiti) | 01 | tutte |
| C-3 | Dati del piano: `PLAN: PlanRoutine[]` (`src/lib/plan/plan.ts`), `SCALES: Record<ScaleId, Scale>` (`src/lib/plan/scales.ts`), tipi `RoutineKey`, `SlotId`, `ScaleId`, `VariantId`, `PlanRoutine`, `Slot`, `Scale`, `Variant`, `Target` (`src/lib/plan/types.ts`), helper `findSlot(slotId)`, `findVariant(variantId)`, `scaleOf(slotId)`, `nextVariant(scaleId, variantId)`, `variantsOf(slotId)`, `slotsUsingVariant(variantId)`, `allVariantsInUse(states)` (`src/lib/plan/lookup.ts`), `variantAt(history, slotId, atMs)` (`src/lib/plan/history.ts`, 03) | `src/lib/plan/*` | 02 | 03, 04, 05a, 05b |
| C-4 | `buildRoutineBody(routine: PlanRoutine, states: SlotState[], templateIds: TemplateIdMap): RoutineWriteBody` (pura) e `targetNote(slot, state): string` | `src/lib/plan/routine-builder.ts` | 02 | 04 |
| C-5 | Stato di uno slot: `SlotState { slotId, variantId, target: Target, updatedAt, source: 'plan' \| 'apply' }` con `Target = { kind: 'reps'; min; max } \| { kind: 'max2'; threshold } \| { kind: 'hold'; seconds; maxSeconds }`; `initialSlotStates(): SlotState[]` | `src/lib/plan/types.ts`, `src/lib/plan/state.ts` | 02 | 03, 04, 05b |
| C-6 | `ensureTemplates(client, variants: Variant[], existing: HevyExerciseTemplate[]): Promise<TemplateIdMap>` con `TemplateIdMap = Record<VariantId, string>` — trova per titolo esatto, crea solo ciò che manca | `src/lib/hevy/ensure-templates.ts` | 02 | 04 |
| C-7 | `db` (Drizzle su `@libsql/client`), `Db`, schema Drizzle con le tabelle `workouts`, `workout_sets`, `sync_state`, `hevy_routines`, `hevy_templates`, `slot_state`, `slot_state_history`, `reviews`, `proposals`, `cron_runs`, `push_subscriptions` | `src/lib/db/client.ts`, `src/lib/db/schema.ts` | 03 (prime 7), 04 (`reviews`, `proposals`, `cron_runs`), 07 (`push_subscriptions`) | tutte le successive |
| C-8 | `syncWorkouts({ db, hevy, now }): Promise<SyncResult>` (`{ mode: 'full' \| 'incremental'; fetched; upserted; deleted; cursor }`) e `syncOneWorkout({ db, hevy }, workoutId)` | `src/lib/sync/sync-workouts.ts` | 03 | 04, 05b, 06 |
| C-9 | `aggregateWindow({ sets, states, window }): ExerciseWindowSummary[]` (pura) con `ExerciseWindowSummary { slotId, variantId, sessions: SessionSummary[], sessionCount, hasEnoughData, ruleAction }` | `src/lib/review/aggregate.ts` | 04 | 05a, 05b |
| C-10 | `ProposalAction = 'keep' \| 'increase' \| 'next_variant'`; `ReviewResponseSchema` (zod) → `ReviewResponse { summary: string; proposals: { slotId; action; rationale }[] }`; `ReviewRun { id, status: 'completed' \| 'skipped' \| 'failed', trigger: 'cron' \| 'manual' \| 'cli', ... }` | `src/lib/review/schema.ts` | 04 | 05b, 06, 07 |
| C-11 | `runReview({ db, hevy, anthropic, now, trigger }): Promise<ReviewRun>` — sync, aggregazione, chiamata, validazione, salvataggio; `shouldRunScheduledReview(db, now): Promise<boolean>` | `src/lib/review/run-review.ts` | 04 | 05b, 06 |
| C-12 | `ruleAction(summary, state, scale): ProposalAction` e `computeNextSlotState(state, action, scale): SlotState` (pure) | `src/lib/plan/progression.ts` | 04 | 05b |
| C-13 | `applyProposal({ db, hevy }, proposalId): Promise<ApplyResult>` e `applyAllOpen({ db, hevy }): Promise<ApplyResult[]>` | `src/lib/apply/apply-proposal.ts` | 04 | 05b |
| C-14 | `createSessionToken(secret, now)`, `verifySessionToken(token, secret, now)`, `isPasswordCorrect(given, expected)`, `SESSION_COOKIE = 'hc_session'`, `isPublicPath(pathname)` (`src/lib/auth/session.ts`); `sessionDenied(): Promise<NextResponse | null>` e `assertSession(): Promise<void>` (`src/lib/auth/require-session.ts`) | `src/lib/auth/` | 05a | 05b, 06, 07 |
| C-15 | `env` validato con zod (server-only): `HEVY_API_KEY`, `ANTHROPIC_API_KEY`, `TURSO_DATABASE_URL`, `TURSO_AUTH_TOKEN?`, `APP_PASSWORD`, `SESSION_SECRET`, `CRON_SECRET`, `HEVY_WEBHOOK_SECRET`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`, `APP_URL` — ogni spec aggiunge le sue e la 01 crea il file con le prime tre | `src/lib/env.ts` | 01 | tutte |
| C-16 | `sendReviewNotification({ db, review }): Promise<PushReport>` e `sendPushToAll({ db, sender, payload, now })` | `src/lib/push/send.ts` | 05b (no-op con la firma definitiva) → 07 (implementazione) | 05b, 06 |
| C-17 | Tempo: `parseHevyInstant(iso)`, `romeDate(ms)`, `romeDaysBetween(aMs, bMs)`, `romeDayStartMinusDays(ms, days)`, `nextSaturday(ms)` (05a) | `src/lib/time.ts` | 03 (05a aggiunge `nextSaturday`) | 04, 05a, 06 |
| C-18 | Container: `getDb()`, `getHevyClient()`, `getReviewModel()` (fake con `E2E_FAKE_SERVICES=1`) | `src/lib/container.ts` | 05a | 05b, 06, 07 |

Regola: un contratto si cambia solo nella spec che lo crea; se cambia nome o firma, si aggiornano
questa tabella e OGNI spec viva che lo cita (testo e prompt in fondo).

## 7. Regole della serie

1. **Le etichette della serie non entrano nel codice**: niente «D-12», «T-3», «spec 04» in
   identificatori, commenti o messaggi di commit; nel codice si scrive il perché per esteso.
2. **Numeri di riga**: non esistono ancora; le spec citano file, funzioni e firme. Un nome o una
   firma sbagliati sono un rilievo, un numero di riga no.
3. **Limiti dei documenti**: CLAUDE.md sotto 40.000 caratteri (obiettivo 20.000; misura:
   `node -e "console.log([...require('fs').readFileSync('CLAUDE.md','utf8').replace(/\r/g,'')].length)"`),
   AGENTS.md ≤ 1000 righe (obiettivo 600), `doc/guide/<tema>.md` ≤ 500 righe. Chi sfora non taglia
   in chiusura: lo dice e propone lo scorporo (prompt 05/06).
4. **Stato**: ogni spec apre con la riga `> **Stato**: …` (scritta · in implementazione il <data>,
   branch · implementata il <data>, PR #N · in esercizio il <data> · ritirata il <data>); lo stesso
   stato sta in § 5 qui e, per la spec corrente, in `CLAUDE.md § Current Status` → Latest. Si
   chiude con `grep -n "Stato" doc/v1/*.md` che dice la stessa cosa ovunque.
5. **In esercizio** vuol dire: PR mergiata su `main` E deploy di produzione su Vercel verde E la
   misura di chiusura della spec rifatta in produzione (ogni spec la dichiara in § 8). Per 01 e 02,
   che vivono prima del deploy: PR mergiata su `main` E il collaudo di § 7 annotato in CLAUDE.md.
6. **Ritiro**: una spec in esercizio si ritira con `doc/v1/PROMPT-RITIRO.md` (da generare con
   `claude-config/prompts/08_CREA_RITIRO_SPEC.md` nella sessione della lettura adversariale):
   verifica criterio per criterio, travaso in AGENTS.md / `doc/guide/` / CLAUDE.md, `git rm` della
   spec; l'indice resta finché vive una spec; l'ultimo ritiro porta via la cartella.
7. **Dati dell'owner**: nessun id reale di routine o workout, nessun nome, nessun segreto nei file
   tracciati (WORKFLOW.md § 1.6). Le fixture di `tests/fixtures/hevy/` sono già ripulite; una
   fixture nuova si ripulisce allo stesso modo (README della cartella).
8. **Prove sui servizi veri**: il client Hevy e Anthropic si testano con le fixture, mai contro i
   servizi; una chiamata reale si fa solo nel collaudo guidato (§ 7 di ogni spec) con l'OK
   dell'owner, ed è sempre dichiarata con il suo effetto (lettura / scrittura) e il suo costo.
9. **Ogni rinvio ha un erede**: una spec successiva per nome o `CLAUDE.md § Not yet present`.
10. **Le regole di AGENTS.md** valgono dalla 01 in poi (il file nasce con la pianificazione e dice
    già come si lavora); ogni spec ne cita in § 5 quelle che contano «qui», con «qui vuol dire che».
