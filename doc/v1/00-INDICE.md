# Serie «v1» — indice

> Prima serie di spec di questo repo (cartella nata il 2026-10-23, decisione D-9). Convenzione:
> una spec = una sessione di implementazione = un branch = un commit; le spec sono SOLO
> istruzioni di sviluppo (chi implementa non decide niente); lo stato vive qui (§ 5),
> nell'intestazione di ogni spec e in `CLAUDE.md § Current Status`, e si chiude con un grep
> (§ 7). Non c'è codice al 2026-10-23: le prove sono i payload reali di Hevy (fixture in
> `tests/fixtures/hevy/`, letti il 2026-10-23) e la documentazione dei servizi, citata con la
> data di lettura. **Lettura adversariale fatta il 2026-10-24** (prompt
> `claude-config/prompts/07_LETTURA_ADVERSARIALE_SPEC.md`; 9 lettori, 314 rilievi): ne sono nate le
> decisioni D-36…D-55 e le scelte T-23…T-27, la tabella § 6 è stata riscritta con le firme reali e
> ogni spec porta la riga «riletta in modo adversariale il 2026-10-24». Prossimo passo:
> `PROMPT-RITIRO.md` (prompt 08, sessione a sé), poi la 01.

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
  01 lo converte in stringa con `String(id)`; il formato reale si osserva nel collaudo D della 02 e
  si annota in AGENTS.md § 3); 403 `exceeds-custom-exercise-limit`. Nessun
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
| D-5 | Allenamento Lun–Ven la sera, 30–40 min a sessione, mercoledì più leggero. **Precisata da D-46** (2026-10-24): il mercoledì sta sotto i 30 min (stima del modello di 02 § 4.5: ~28 min). | 02 |
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
| D-13 | Autenticazione: **password unica** in env + **cookie firmato** di lunga durata; `proxy.ts` protegge tutto tranne `/login`, le route del cron e del webhook, il manifest, il service worker e le icone. Produzione senza Vercel Authentication; i deploy di preview sono spenti (T-19) e Standard Protection resta impostata come rete. | 05a, 06 |
| D-14 | Modello per la revisione: **Claude Opus 5.5** (`claude-opus-5-5`), effort `high`. | 04 |
| D-15 | Finestra inviata a Claude: **28 giorni** più l'esito della revisione precedente. | 04 |
| D-16 | Cron settimanale `"17 6 * * 6"` (UTC): su Hobby il minuto non è rispettato, quindi parte il sabato **fra le 06:00 e le 06:59 UTC**, cioè fra le 07:00 e le 07:59 a Roma d'inverno e fra le 08:00 e le 08:59 d'estate (precisato il 2026-10-24); il codice esegue la revisione solo se sono passati **≥ 13 giorni** dall'ultima completata. | 06 |
| D-17 | Dashboard con **grafici completi (Recharts)**. | 05a |
| D-21 | Lingua di CLAUDE.md, AGENTS.md, WORKFLOW.md: **inglese**; spec, chat e UI in italiano; codice e commenti in inglese. | tutte |
| D-23 | **main = produzione**; una PR per spec da `feature/*` a `main`; niente `develop`. I deploy di preview sono spenti da `vercel.json` (T-19); Standard Protection resta impostata come rete se mai si riaccendessero. | tutte, 06 |
| D-34 | Il repo è **open source, AGPL-3.0** (ribalta la parte «privato» di D-1): un coach a utente singolo, self-hosted; LICENSE al bootstrap, SETUP.md completo nella 06, README con il disclaimer «non affiliato a Hevy»; WORKFLOW.md § 1.6 senza eccezioni (nessun dato dell'owner nei file tracciati, nei commit e nelle PR). | tutte, 06 |
| D-35 | README.md in **inglese**; i nomi del piano e degli esercizi restano italiani (etichette del prodotto), con una riga che lo dice. | tutte |
| D-36 | (2026-10-24) `src/lib/env.ts` resta `server-only`. Ogni modulo che può girare anche fuori da Next (gli script `tsx`, `src/lib/db/client.ts`, il sender web-push della 07) legge **solo le chiavi che usa** con `EnvSchema.pick({ … }).parse(process.env)` da `src/lib/env-schema.ts`, mai da `@/lib/env`. `vitest.config.ts` ha l'alias `'server-only'` → `tests/helpers/empty.ts` (modulo vuoto) e valori finti per tutte le chiavi obbligatorie dello schema. La regola sta in AGENTS.md § 2.3. | 01, 03, 05a, 06, 07 |
| D-37 | (2026-10-24) **In esercizio** per 01, 02, 03, 04, 05a e 05b (vivono prima del deploy) vuol dire: PR mergiata su `main` E collaudo di § 7 annotato in CLAUDE.md. La 06 rifà in produzione le misure di chiusura di 03–05b nel proprio collaudo, dichiarandone effetto e costo. (§ 7.5 riscritta.) | tutte |
| D-38 | (2026-10-24) Chiusura di una spec il cui merge è il deploy (06, 07): codice → collaudo locale → passi di produzione che non richiedono il codice nuovo (migrazione remota, env su Vercel), ognuno dopo l'OK → commit unico con Stato «implementata» → PR e merge su `main` (= deploy) → passi post-deploy riferiti in chat. Gli esiti post-merge e «in esercizio il <data>» si scrivono come **primo passo della sessione successiva** (per la 07, in un commit di sola documentazione). | 06, 07 |
| D-39 | (2026-10-24) Retry del client Hevy: **429 su tutti i metodi** (la richiesta non è stata eseguita); **5xx solo su GET e PUT** (idempotenti); **mai su POST**: una POST con 5xx diventa subito `HevyApiError`, e il rilancio di `hevy:upload` trova per titolo ciò che è stato creato davvero. Timeout ed errori di rete non si ritentano (T-23). | 01, 02, 04 |
| D-41 | (2026-10-24) Il database di produzione nasce **dall'import di `local.db`** (`turso db create hevy-coach --from-file local.db`, o l'import dal dashboard): stato degli slot, storia, revisioni e proposte passano in produzione intatti; poi `pnpm db:migrate:remote` (no-op) e `pnpm hevy:sync:remote` (incrementale). | 06 |

### 3.3 Rappresentazione su Hevy (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-18 | «max-2»: set con `reps` e `rep_range` **null** e nota Hevy sull'esercizio «max-2: fermati a 2 ripetizioni dal cedimento»; la **soglia numerica di passaggio** (ripetizioni su tutte le serie) sta nella scala (`Variant.targets.max2.threshold`, che lo slot adotta arrivando sulla variante e poi vive in `SlotState.target.threshold`). | 02, 04 |
| D-19 | Tenute con range (Wall sit 45–60", Hollow hold 20–30"): `duration_seconds` = **limite inferiore**, nota con il range; progressione: tutte le serie al limite superiore → variante successiva. | 02, 04 |
| D-20 | Esercizi per lato: `reps` = ripetizioni **per lato** (es. 10) + nota «per gamba» / «per lato». | 02 |
| D-24 | Riscaldamento (5') nelle **note della routine**; mobilità del mercoledì = 3 esercizi registrati ma **esclusi dalla revisione**. **Riscritta da D-45** (2026-10-24): 90/90 anche e couch stretch a tempo (60" per lato, un set da 120 s), rotazioni toraciche **a ripetizioni** (10 per lato; custom `reps_only`). | 02, 04 |
| D-25 | Cartella Hevy **«Coach · Corpo libero»**; titoli **«Lun · Upper A», «Mar · Lower A», «Mer · Core + mobilità», «Gio · Upper B», «Ven · Lower B»**. | 02 |
| D-26 | Gli esercizi custom hanno titolo **in italiano**, come nel piano; quelli del catalogo tengono il titolo Hevy. Un custom che esiste già con lo stesso titolo esatto si riusa («Hollowbody Hold», «Couch Stretch»). | 02 |
| D-27 | Recuperi. **Riscritta da D-47** (2026-10-24) con l'elenco di 02 § 4.1: **90 s** su push-up, push-up declinati, rematore, pike, squat, affondo indietro, bulgarian, stacco rumeno; **60 s** su push-up stretti e con pausa, affondo laterale, glute bridge, calf, hip thrust, Y-T-W, snow angel, abduzioni; **45 s** sul core e sul wall sit; **0** sulla mobilità. Così il venerdì sta in 40,0 min. | 02 |
| D-40 | (2026-10-24) `pnpm hevy:upload` usa **sempre** gli stati iniziali del piano ed è lo script di nascita: si lancia solo prima del primo «Applica». Dalla 04 esiste `pnpm hevy:upload -- --from-db`, che legge `slot_state`; senza il flag, se `slot_state` contiene righe `source = 'apply'`, lo script si rifiuta di scrivere. SETUP.md (06) cita solo la forma `--from-db` dopo il primo «Applica». | 02, 03, 04, 06 |
| D-44 | (2026-10-24) «Inverted Row» `425805F4` manca da `catalog_bodyweight.json`: prima dell'implementazione della 02 si fa **una GET reale di sola lettura** `GET /v1/exercise_templates/425805F4` e la risposta va in una nuova fixture reale `tests/fixtures/hevy/catalog_extra.json`, letta dai test insieme a `catalog_bodyweight.json`. | 02 |
| D-45 | (2026-10-24) Le rotazioni toraciche si registrano **a ripetizioni** (custom «Rotazioni toraciche», `reps_only`, un set da 10 per lato), come la spec 02; D-24 riscritta di conseguenza. | 02 |
| D-46 | (2026-10-24) Il **tetto del mercoledì è 30 min** (D-5 precisata); la nota della routine dice «~28 min», che è la stima del modello di 02 § 4.5 (28,5 min). | 02 |
| D-47 | (2026-10-24) Valgono i **recuperi della spec 02** § 4.1 (elenco per famiglia); D-27 riscritta copiandoli. | 02 |

### 3.4 Revisione e «Applica» (2026-10-23)

| # | Decisione | Spec |
|---|---|---|
| D-28 | «Aumenta ripetizioni» su Hevy = **range +2/+2** (es. 8–12 → 10–14), ripetizioni fisse +2, **tenute +10 s** (nota aggiornata), max-2: **soglia +2**. Il numero di serie non cambia mai. | 04 |
| D-29 | «Applica» per **singola proposta**, con **«Applica tutte»** per le aperte; ogni applicazione = GET routine + PUT completo, una routine alla volta. | 04, 05b |
| D-30 | «Esegui revisione ora» **parte sempre** (la regola dei 13 giorni vale solo per il cron); le proposte aperte della revisione precedente diventano `superseded`. | 04, 05b |
| D-31 | Dati scarsi, **regola in codice**: un esercizio con < 2 sessioni nella finestra riceve «mantieni (dati insufficienti)» senza passare da Claude; con 0 allenamenti nella finestra la revisione è `skipped` e la notifica dice «nessun allenamento nelle ultime 4 settimane». | 04 |
| D-22 | Esiti possibili per esercizio: **solo** `keep` / `increase` / `next_variant`. Tornare indietro di variante si fa a mano su Hevy (CLAUDE.md § Not yet present). | 04 |
| D-32 | Il **webhook di Hevy** entra nella v1: `POST /api/hevy/webhook` verifica `HEVY_WEBHOOK_SECRET`, risponde 200 entro 5 s e sincronizza quel workout; il sync via `workouts/events` resta nel cron come rete di sicurezza. L'iscrizione si fa dopo il deploy (06). | 03, 06 |
| D-33 | L'app sincronizza **tutti** i workout dell'account (prima volta: 18 pagine; poi solo gli eventi dal cursore); l'aggregazione usa solo gli esercizi del piano, per `exercise_template_id`, quindi anche gli allenamenti a corpo libero fatti prima del piano contano (come: D-42). | 03, 04 |
| D-42 | (2026-10-24) La **prima riga di storia** di ogni slot vale «da sempre»: `variantAt(history, slotId, atMs)` restituisce la variante della prima riga anche per un istante anteriore al suo `validFrom`, e `null` solo se lo slot non ha righe. Così le sessioni pre-piano con la stessa variante contano nella revisione e in Progressi (D-33). | 03, 04, 05a |
| D-43 | (2026-10-24) Proposte `insufficient_data`: **N** (badge di Oggi, bottone «Applica tutte (N)», notifica push) conta **solo** le proposte aperte non `insufficient_data`; «Applica tutte» chiude anche quelle (`applied` senza chiamate Hevy) e il dialogo le elenca in fondo come «Dati insufficienti: mantieni»; su `/` compaiono come righe compatte in fondo. | 04, 05a, 05b, 07 |
| D-48 | (2026-10-24) `syncOneWorkout` che riceve **404** per un workout presente in locale **cancella la copia locale** (workout e set, un batch) e restituisce `not_found`: Hevy è la fonte di verità. | 03, 06 |
| D-49 | (2026-10-24) Il **sync completo** (primo sync o `--full`) **pota**: dopo l'ultima pagina cancella in un batch i workout locali il cui id non è comparso, contati in `SyncResult.deleted`. | 03 |
| D-50 | (2026-10-24) Due slot della stessa routine sulla stessa variante: dentro un workout con `routine_id` del piano, la **k-esima occorrenza** del template (ordine `exerciseIndex`) va al **k-esimo slot** candidato nell'ordine della routine; le occorrenze oltre il numero di slot vanno in `unresolved`. | 04 |
| D-51 | (2026-10-24) Dopo un «Applica» a metà finestra, nell'aggregato di uno slot entrano **solo le sessioni con `startTime ≥ validFrom` della riga di storia corrente**; se sono meno di 2 → `insufficient_data`. Il prompt porta per ogni sessione `variant` e `target` del suo momento. Le sessioni precedenti restano nel DB e in Progressi (D-54). | 04 |
| D-52 | (2026-10-24) `allAtTop` di una sessione: si contano i set `normal` e `failure` (`warmup` e `dropset` scartati); la sessione è «al top» se ha **almeno `slot.sets` valori validi** e **ognuno** raggiunge il massimo del target; una sessione senza valori validi non esiste (non entra in `sessions`). Il prompt riceve le serie prescritte di ogni slot. | 04 |
| D-53 | (2026-10-24) Spesa della scadenza (T-10 riscritta): il client Anthropic nasce con `maxRetries: 0` e la chiamata passa `timeout` pari ai millisecondi che restano alla scadenza; sotto `MIN_MODEL_CALL_MS` la revisione è `failed/model: timeout` senza chiamare. Un solo tentativo, costo prevedibile. | 04 |
| D-54 | (2026-10-24) Progressi mostra **tutte** le sessioni dello slot nei 56 giorni; una linea verticale (Recharts `ReferenceLine`) marca ogni cambio di variante con il titolo della nuova; il flag «al top» di ogni punto usa il target della variante in vigore quel giorno (`variantAt`). | 05a |
| D-55 | (2026-10-24) `/` (Oggi) elenca **tutte le proposte non `superseded` dell'ultima revisione `completed`**: le aperte in alto con i bottoni, le applicate e ignorate come righe compatte con esito e data, le `insufficient_data` in fondo (D-43). Nessuna scheda sparisce dopo «Applica» o «Ignora». | 05b |

## 4. Scelte adottate dal pianificatore («adottata, ribaltabile» — rovesciabili prima dell'implementazione)

| # | Scelta | Perché | Spec |
|---|---|---|---|
| T-1 | Regione Vercel **`dub1`** e gruppo Turso `aws-eu-west-1` (Dublino). | Turso non ha Francoforte (2.3); stessa regione = latenza minima. | 06 |
| T-2 | Versioni pinnate (2026-10-23): next 16.4.2, react 19.3.1, typescript 7.0.2, drizzle-orm 0.45.4, drizzle-kit 0.31.11, @libsql/client 0.18.0, @serwist/next + serwist 9.5.13, @anthropic-ai/sdk 1.1.0, zod 4.6.7, web-push 3.6.7, vitest 5.0.5, @playwright/test 1.64.1, @biomejs/biome 2.6.2, tsx 4.23.15, tailwindcss 4.3.3, nanoid 6.0.2; (2026-10-24, `npm view`): server-only 0.0.1, jose 6.2.13, recharts 3.10.1, lucide-react 1.61.0, @axe-core/playwright 4.14.0, cross-env 10.1.0. `packageManager: pnpm@11.22.0`, `engines.node >= 22.9` (`--env-file-if-exists`). **Regola** (2026-10-24): ogni pacchetto che una spec installa ha la versione esatta nel comando `pnpm add`; dove una spec scrive `<versione esatta di oggi>` (dotenv, @types/*, @types/web-push, sharp), chi implementa la legge con `npm view` quel giorno e la scrive qui nel proprio commit. Niente `dotenv-cli` (T-27), niente `png-to-ico`. | Le versioni di segnaprezzi erano già a un minor indietro. | 01 |
| T-3 | `@libsql/client` con l'entry **Node** (non `/web`): su Vercel le function sono Node (Fluid), e solo l'entry Node apre `file:local.db`. Client singleton su `globalThis` come segnaprezzi. | segnaprezzi lo fa già in produzione. | 03 |
| T-4 | Next 16 **senza** `cacheComponents`, dev con Turbopack, **`next build --webpack`** (Serwist). React Compiler spento. Le pagine sotto `src/app/(app)/` leggono il DB a ogni richiesta: `(app)/layout.tsx` dichiara `export const dynamic = 'force-dynamic'` (precisato il 2026-10-24: senza, Next 16 le prerenderizza alla build con i dati di quel momento). | Serwist richiede webpack su Next 16 (2.2). | 01, 05a, 05b, 07 |
| T-5 | Layout `src/`: `src/app` (route), `src/components`, `src/lib/{hevy,plan,db,sync,review,apply,auth,push,progress,proposals,time.ts,env.ts,env-schema.ts,container.ts}`, `scripts/` (CLI con tsx), `tests/fixtures/`, `tests/helpers/` (harness condiviso), `tests/e2e/` (Playwright), test accanto ai moduli (`*.test.ts`); i test che leggono i sorgenti (`guards.test.ts`) stanno in `src/app/`. | Come segnaprezzi. | 01 |
| T-6 | Qualità: Biome (lint + format, file intero in 01 § 4.2), `tsc --noEmit`, Vitest; Playwright dalla 05a. Script: `dev`, `build`, `start`, `lint`, `lint:fix`, `typecheck`, `test`, `test:watch`, `hevy:whoami` (01), `hevy:upload` (02), `db:generate`, `db:migrate`, `db:studio`, `hevy:sync` (03), `review:run`, `review:show`, `review:compare`, `review:apply` (04), `e2e:seed`, `e2e:server`, `test:e2e` (05a), `db:migrate:remote`, `hevy:sync:remote` (06), `icons:generate`, `e2e:prod`, `test:e2e:pwa` (07). | Un solo set di comandi per tutta la serie; WORKFLOW.md § 3 lo ricopia il giorno in cui ogni comando diventa vero. | 01+ |
| T-7 | I template custom si creano **pigramente**: all'upload solo le varianti presenti nelle routine di partenza (9 custom, 02 § 4.4), e le varianti successive quando «Applica» le raggiunge (`ensureTemplates`, C-6). Nessun altro punto crea template: `syncRoutines` (03) risolve la mappa in sola lettura e, se una variante in uso non ha template, fallisce con «run `pnpm hevy:upload` first». | I custom non si cancellano; il limite di Hevy (403) non è noto. | 02, 03, 04 |
| T-8 | L'identità di uno slot del piano è lo `slotId` (es. `mon_push_up`); lo stato corrente dello slot (variante, target) vive in Turso (`slot_state`); la storia in `slot_state_history`. L'aggregazione attribuisce un set a uno slot **prima per il `routine_id` del workout** (→ `RoutineKey`), poi per la variante attiva alla data del workout (storia) il cui template è quello del set; una variante può stare in due slot (`hollow_hold` mercoledì e giovedì): senza `routine_id` e con più candidati il set finisce in `unresolved` (04 § 4.2). | Senza storia non si sa quale variante era attiva in una data; il `routine_id` c'è sempre quando si parte dalla routine. | 03, 04 |
| T-9 | Oltre a Claude, il codice calcola `ruleAction` (la doppia progressione in forma deterministica, C-12) per ogni esercizio; la proposta salva entrambi e la dashboard mostra quando divergono. Claude riceve `ruleAction` nel prompt come suggerimento, non come vincolo. | Verificabilità: un test prova la regola senza chiamare Claude. | 04, 05b |
| T-10 | Chiamata a Claude: `client.messages.parse` con `zodOutputFormat(ReviewResponseSchema)`, `max_tokens: 16000` (il thinking, sempre attivo su Opus 5.5, conta nel limite: ~3k di JSON per 27 slot più il ragionamento a effort `high`), `output_config: { effort: 'high', format }`; client creato con `maxRetries: 0` e ogni chiamata con `timeout` = millisecondi che restano della scadenza unica di `runReview` (280 s dentro i 300 s di Vercel; sotto `MIN_MODEL_CALL_MS` non si chiama e la revisione è `failed`, `model: timeout`; D-53), **un** retry applicativo sull'errore di validazione solo se resta margine (owner, 2026-10-23); **niente `fallbacks`**: un `stop_reason` `refusal` o `max_tokens` rende la revisione `failed`; il rifiuto porta la sua categoria (`model: refusal (bio)`, da `stop_details.category`), così la notifica dice perché (owner, 2026-10-23). Costo stimato salvato in `reviews` (token × prezzo di listino in `src/lib/review/pricing.ts`). Una revisione = una chiamata riuscita (+1 retry). | Costo e durata prevedibili (300 s di Vercel); i fallback aggiungono una beta per un caso che qui vale un `failed` leggibile. | 04 |
| T-11 | Lock della revisione: riga `reviews` con `status='running'` più giovane di 10 min → la nuova esecuzione esce con `skipped: 'already_running'`. Serve contro i duplicati del cron Hobby. | 2.2 | 04, 06 |
| T-12 | Route: `GET /api/cron/review` (cron, Bearer `CRON_SECRET`), `POST /api/hevy/webhook`, `POST /api/reviews/run` (manuale), `POST /api/proposals/[id]/apply`, `POST /api/proposals/[id]/dismiss`, `POST /api/proposals/apply-all`, `POST /api/sync`, `POST /api/push/subscribe`, `DELETE /api/push/subscribe`, `POST /api/push/test`. Pagine: `/login`, `/` (Oggi: proposte dell'ultima revisione, stato), `/progressi`, `/revisioni`, `/revisioni/[id]`, `/impostazioni`. | Superfici minime per P4–P7. | 05a, 05b, 06, 07 |
| T-13 | Cookie `hc_session`, JWT HS256 (`jose`) firmato con `SESSION_SECRET`, `httpOnly`, `secure: process.env.NODE_ENV === 'production'` (dev ed E2E girano su `http://localhost`), `sameSite: 'lax'`, durata **365 giorni**; login con confronto a tempo costante contro `APP_PASSWORD` e attesa di 1 s sul fallimento. | Una PWA che chiede la password ogni settimana non si usa. | 05a |
| T-14 | UI in italiano, nome dell'app **«Hevy Coach»**, `short_name` «Coach», tema scuro di default; colori e tipografia li decide la 05a con impeccable. | D-21, D-9. | 05a, 07 |
| T-15 | Timezone di calendario **Europe/Rome** per sessioni, finestre e «13 giorni»; in DB gli istanti sono epoch ms UTC. | Come segnaprezzi. | 03, 04 |
| T-16 | Prima revisione: il cron la esegue il primo sabato in cui sono passati ≥ 13 giorni dalla data di caricamento delle routine (`sync_state.plan_uploaded_at`, che la 03 ricava dal `created_at` minimo delle 5 routine su Hevy e scrive una volta sola). | Senza una revisione precedente la regola dei 13 giorni non avrebbe riferimento; la 02 non ha un DB. | 03, 04, 06 |
| T-17 | La notifica push ha titolo e corpo: `completed` → «Revisione pronta» / «N proposte da guardare» (N = proposte non `insufficient_data`, D-43; N = 1 → «1 proposta da guardare»; N = 0 → «Nessuna proposta: tutto resta com'è»); `skipped/no_workouts` → «Revisione saltata» / «Nessun allenamento nelle ultime 4 settimane»; `skipped/already_running` → **nessuna notifica**; `failed` → «Revisione fallita» / i primi 80 caratteri dell'errore. Click → `/revisioni/[id]` (o `/`), url **relativo** (T-24). | D-12, D-31. | 06, 07 |
| T-18 | Le ripetizioni **fisse** del piano (affondo 10, glute bridge 12, calf 15, Y-T-W 8, push-up con pausa 8, superman 12, snow angel 12, abduzioni 15) diventano range di 4 (`n`–`n+4`; calf e abduzioni `15`–`20`). | La doppia progressione (D-6) ha bisogno di un massimo da chiudere; il minimo resta il numero del piano. | 02 |
| T-19 | `vercel.json`: `"git": { "deploymentEnabled": { "**": false, "main": true } }`: nessun deploy di preview per gli altri branch (collaudo in locale); Standard Protection resta impostata come rete. Doc Vercel letta il 2026-10-24: le chiavi sono pattern minimatch, un branch non elencato è `true`, e se un branch combacia con più regole basta una `true` (`main` combacia con entrambe e deploya; `feature/01-…` combacia solo con `**`, perché `*` si ferma al `/`). | Preview inutili per un'app a utente singolo; meno superfici esposte. | 06 |
| T-20 | I grafici dei progressi usano una finestra di 56 giorni (8 settimane), con la stessa `aggregateWindow`. | Due cicli di revisione e mezzo leggibili in un grafico. | 05a |
| T-21 | Container dei servizi (`src/lib/container.ts`, C-18): `getDb()`, `getHevyClient()`, `getReviewModel()`; con `E2E_FAKE_SERVICES=1` restituisce fake in memoria. L'app usa solo il container; gli script CLI usano i client direttamente. | Playwright non deve toccare Hevy né Anthropic. | 05a, 05b, 06, 07 |
| T-22 | Il webhook risponde **200** `{ received: true }` subito (D-32: Hevy documenta «200 entro 5 s») e sincronizza in `after()`; il cron sincronizza sempre, anche quando la revisione non è dovuta. | Limite di 5 s di Hevy; dashboard aggiornata al sabato anche senza webhook. | 06 |
| T-23 | (2026-10-24) Il client Hevy ha `timeoutMs?: number` (default 30 000), applicato con `AbortSignal.timeout(timeoutMs)` a ogni tentativo; un timeout o un errore di rete diventano `HevyApiError { status: 0, code: 'network' }` e non si ritentano. | Una richiesta appesa consumerebbe la scadenza unica della revisione (04) e i 300 s di Vercel (06). | 01, 04, 06 |
| T-24 | (2026-10-24) L'`url` della notifica push è **relativo** (`/revisioni/<id>` o `/`) e il service worker lo risolve sulla propria origine; `APP_URL` non esiste (fuori da `EnvSchema`, da C-15, dalla 06 e da SETUP.md). | Un url assoluto non passerebbe lo schema del payload della 07; l'env non avrebbe altri lettori. | 06, 07 |
| T-25 | (2026-10-24) Sender web-push **pigro**: `sendReviewNotification({ db, review, sender?, now? })` costruisce il sender dentro la funzione, solo se esiste almeno una sottoscrizione, leggendo le VAPID con `EnvSchema.pick(...)` (D-36). Con zero sottoscrizioni non tocca né le env né `web-push`: i test di 05b e 06 e l'E2E restano verdi. | C-16 non cambia firma fra 05b e 07. | 05b, 06, 07 |
| T-26 | (2026-10-24) `applyProposal` prenota la proposta con un `UPDATE proposals SET status = 'applying' WHERE id = ? AND status = 'open'`: `rowsAffected = 0` → `not_open`; su errore torna `open` con `apply_error`. Quinto stato `applying` nel commento dello schema e nella UI (05b: scheda in `working`). | Due «Applica» concorrenti (doppio tocco, due schede, «Applica tutte») applicherebbero due volte. | 04, 05b |
| T-27 | (2026-10-24) Comandi contro Turso remoto senza dipendenze nuove: `"db:migrate:remote": "node --env-file=.env.turso-migrate ./node_modules/drizzle-kit/bin.cjs migrate"` e `"hevy:sync:remote": "tsx --env-file=.env.turso-migrate --env-file-if-exists=.env.local scripts/hevy-sync.ts"` (`--env-file` di Node non sovrascrive le variabili già nell'ambiente). | Niente `dotenv-cli`; `.env.turso-migrate` è gitignored. | 06, 07 |

## 5. Ordine, dipendenze e stato

| Spec | Titolo | Branch | Dipende da | Stato |
|---|---|---|---|---|
| 01 | Scaffold e client Hevy | `feature/01-scaffold-hevy-client` | — | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 02 | Piano, scale di varianti, mappa esercizi e caricamento delle routine da CLI | `feature/02-plan-upload` | 01 | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 03 | Turso: schema, migrazioni e sincronizzazione degli allenamenti | `feature/03-turso-sync` | 02 | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 04 | Aggregazione, revisione con Claude e «Applica» da CLI | `feature/04-review` | 03 | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 05a | Web app: accesso, shell, design e pagina Progressi | `feature/05a-app-progressi` | 04 | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 05b | Web app: revisioni, proposte, «Applica» ed «Esegui revisione ora» | `feature/05b-app-revisioni` | 05a | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 06 | Messa in esercizio: Vercel, Turso remoto, cron e webhook | `feature/06-deploy` | 05b | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |
| 07 | PWA e Web Push | `feature/07-pwa-push` | 06 | scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24 |

Vincolo di priorità (P1): dopo la 02 le routine sono su Hevy e l'owner si allena; dalla 04 la
revisione gira da CLI; dalla 06 gira da sola. Una spec parte solo quando la precedente è **in
esercizio** (§ 7.5, con la definizione di D-37 per 01–05b). La prima da incollare è il prompt in
fondo alla 01, dopo `PROMPT-RITIRO.md`.

## 6. Contratti condivisi fra spec (da usare IDENTICI: nome, firma, file)

| # | Contratto | File | Nasce in | Usato da |
|---|---|---|---|---|
| C-1 | `createHevyClient({ apiKey, fetch?, baseUrl?, retryDelayMs?, timeoutMs? }: HevyClientOptions)` e `type HevyClient = ReturnType<typeof createHevyClient>` con `getUserInfo()`, `listExerciseTemplates()` (tutte le pagine), `getExerciseTemplate(id)`, `createCustomExercise(input): Promise<{ id: string }>`, `listRoutineFolders()`, `createRoutineFolder(title)`, `listRoutines()` (tutte le pagine), `getRoutine(id)`, `createRoutine(body)`, `updateRoutine(id, body)`, `countWorkouts(): Promise<number>`, `listWorkouts(page): Promise<{ page; page_count; workouts: HevyWorkout[] }>`, `getWorkout(id)`, `listWorkoutEvents({ since, page }): Promise<{ page; page_count; events: HevyWorkoutEvent[] }>`, `getExerciseHistory(templateId, { startDate, endDate })`; errori `HevyApiError { status, code, body }` (rete o timeout: `status 0`, `code 'network'`); retry: D-39, timeout: T-23. Harness condiviso dei test: `fixture(name): unknown` e `fakeFetch(routes): { fn, calls }` (le rotte accettano `times?`) in `tests/helpers/fake-fetch.ts` | `src/lib/hevy/client.ts`, `tests/helpers/fake-fetch.ts` | 01 | 02, 03, 04, 05a, 06 |
| C-2 | Tipi e schemi zod: `HevyUserInfo`, `HevyExerciseTemplate`, `HevyRoutineFolder`, `HevyRoutine`, `HevyRoutineExercise`, `HevyRoutineSet`, `HevyWorkout`, `HevyWorkoutExercise`, `HevyWorkoutSet`, `HevyWorkoutEvent` (`deleted_at` facoltativo), `HevyExerciseHistoryEntry`, `RoutineWriteBody`, `RoutineWriteExercise`, `RoutineWriteSet`, `CustomExerciseInput`; `PagedSchema<K extends string, T>(key: K, item: T)` tipizzato per chiave | `src/lib/hevy/schemas.ts` (zod) e `src/lib/hevy/types.ts` (tipi inferiti) | 01 | tutte |
| C-3 | Dati del piano: `PLAN: PlanRoutine[]` e `FOLDER_TITLE` (`src/lib/plan/plan.ts`), `SCALES: Record<ScaleId, Scale>` (`src/lib/plan/scales.ts`), tipi `RoutineKey`, `SlotId`, `ScaleId`, `VariantId`, `PlanRoutine`, `Slot`, `Scale`, `Variant`, `Target`, `TargetKind`, `SlotState`, `TemplateIdMap` (`src/lib/plan/types.ts`), helper `findSlot(slotId): Slot`, `findVariant(variantId): Variant`, `scaleOf(slotId): Scale`, `routineOf(slotId): PlanRoutine` (tutti lanciano su id ignoto), `nextVariant(scaleId, variantId): Variant \| null`, `variantsOf(slotId): Variant[]`, `slotsUsingVariant(variantId): Slot[]`, `allVariantsInUse(states): Variant[]` (senza doppioni, ordine di `PLAN`) (`src/lib/plan/lookup.ts`); `SlotStateHistoryRow { slotId, variantId, target: Target, validFrom, validTo: number \| null, source, proposalId }` e `variantAt(history: SlotStateHistoryRow[], slotId, atMs): VariantId \| null` e `targetAt(history, slotId, atMs): Target \| null` (prima riga valida «da sempre», D-42) (`src/lib/plan/history.ts`, 03) | `src/lib/plan/*` | 02 (03: `history.ts`) | 03, 04, 05a, 05b |
| C-4 | `buildRoutineBody(routine: PlanRoutine, states: SlotState[], templateIds: TemplateIdMap, folderId: number \| null): RoutineWriteBody` (pura), `targetNote(slot: Slot, state: SlotState, variant: Variant): string`, `targetText(target: Target): string` (`max2` → «max-2 · soglia N»), `buildSet(target: Target): RoutineWriteSet` | `src/lib/plan/routine-builder.ts` | 02 | 04, 05a, 05b |
| C-5 | Stato di uno slot: `SlotState { slotId, variantId, target: Target, updatedAt, source: 'plan' \| 'apply' }` con `Target = { kind: 'reps'; min; max } \| { kind: 'max2'; threshold } \| { kind: 'hold'; seconds; maxSeconds }`; `initialSlotStates(now: number): SlotState[]` (`updatedAt = now`, `source: 'plan'`) | `src/lib/plan/types.ts`, `src/lib/plan/state.ts` | 02 | 03, 04, 05a, 05b |
| C-6 | `resolveTemplates(variants: Variant[], existing: HevyExerciseTemplate[]): { map: TemplateIdMap; missing: Variant[] }` (pura: catalogo per id, custom per titolo esatto e tipo) e `ensureTemplates(hevy: HevyClient, variants: Variant[], existing: HevyExerciseTemplate[]): Promise<TemplateIdMap>` (risolve e crea solo i custom mancanti) con `TemplateIdMap = Record<VariantId, string>` | `src/lib/hevy/ensure-templates.ts` | 02 | 03 (solo `resolveTemplates`), 04 |
| C-7 | `db` (Drizzle su `@libsql/client`), `Db`, `applyConnectionPragmas(client, url)`, schema Drizzle con le tabelle `workouts`, `workout_sets`, `sync_state`, `hevy_routines`, `hevy_templates`, `slot_state`, `slot_state_history`, `reviews`, `proposals`, `cron_runs`, `push_subscriptions`; `createMemoryDb(): Promise<Db>` (`src/lib/db/testing/memory-db.ts`). Un repository per tabella in `src/lib/db/repositories/`: `workouts.ts` (`upsertWorkout`, `deleteWorkout`, `listSetsBetween(db, { fromMs, toMs }): Promise<SetRow[]>` con `SetRow` piatta, `getLatestWorkout`, `listWorkoutIds`, `countWorkouts`), `sync-state.ts` (`getSyncValue(db, key)`, `setSyncValue(db, key, value, now)`), `hevy-routines.ts`, `hevy-templates.ts` (`upsertTemplateMap(db, rows, now)`, `getTemplateMap`, `variantsByTemplateId`), `slot-state.ts` (`listSlotStates`, `ensureInitialSlotStates`, `listSlotStateHistory`, `replaceSlotStateStatements`, `replaceSlotState`) — 03; `reviews.ts` — 04 (compresi `countActionableProposals`, `listProposalsOfReview`, `dismissProposal`, `reserveProposal`), esteso da 05b (`getReviewWithProposals`); `cron-runs.ts` — 06; `push-subscriptions.ts` — 07 | `src/lib/db/client.ts`, `src/lib/db/schema.ts`, `src/lib/db/repositories/*` | 03 (prime 7 tabelle), 04 (`reviews`, `proposals`, `cron_runs`), 07 (`push_subscriptions`) | tutte le successive |
| C-8 | `syncWorkouts({ db, hevy, now, forceFull? }): Promise<SyncResult>` (`{ mode: 'full' \| 'incremental'; fetched; upserted; deleted; cursor: string \| null }`; scrive anche `last_sync_mode`) e `syncOneWorkout({ db, hevy, now }, workoutId): Promise<{ status: 'upserted' \| 'not_found' }>` (404 su workout locale: D-48); `syncRoutines({ db, hevy, now }): Promise<{ routines; templates; planUploadedAt }>` (sola lettura, T-7) in `src/lib/sync/sync-routines.ts` | `src/lib/sync/sync-workouts.ts`, `src/lib/sync/sync-routines.ts` | 03 | 04, 05a, 06 |
| C-9 | `aggregateWindow(input: AggregateInput): AggregateResult` (pura) con `AggregateInput { sets: SetRow[]; states: SlotState[]; history: SlotStateHistoryRow[]; templateMap: TemplateIdMap; routineKeys: Map<string, RoutineKey>; window: { fromMs; toMs } }`, `AggregateResult { summaries: ExerciseWindowSummary[]; unresolved: { count; byTemplate } }`, `ExerciseWindowSummary { slotId, routineKey, variantId, target, sets, perSide, sinceMs, sessions: SessionSummary[], sessionCount, hasEnoughData, ruleAction, allSessions: SessionSummary[] }` (una voce per ogni slot `reviewed`, anche senza sessioni; `sessions` solo dalla riga di storia corrente, D-51; `allSessions` tutta la finestra, per Progressi, D-54), `SessionSummary { workoutId, date, startTime, variantId, target, values, allAtTop }` | `src/lib/review/aggregate.ts` | 04 | 05a |
| C-10 | `ProposalAction = 'keep' \| 'increase' \| 'next_variant'`; `ReviewResponseSchema` (zod) → `ReviewResponse { summary: string; proposals: { slotId; action; rationale }[] }`; `ReviewStatus = 'running' \| 'completed' \| 'skipped' \| 'failed'`; `ReviewTrigger = 'cron' \| 'manual' \| 'cli'`; `ReviewRun { id; status: ReviewStatus; trigger; skipReason?: 'no_workouts' \| 'already_running'; error?: string; proposalCount: number }` (`already_running` porta l'`id` della revisione in corso) | `src/lib/review/schema.ts` | 04 | 05b, 06, 07 |
| C-11 | `runReview({ db, hevy, model, now, clock, trigger }: { db: Db; hevy: HevyClient; model: ReviewModel; now: number; clock: () => number; trigger: ReviewTrigger }): Promise<ReviewRun>` — sync, aggregazione, chiamata, validazione, salvataggio, dentro una scadenza unica `now + REVIEW_DEADLINE_MS` (`clock` misura il tempo passato); `shouldRunScheduledReview(db, now): Promise<boolean>`; `scheduledReviewReference(db): Promise<number \| null>` (`finished_at` dell'ultima `completed`, altrimenti `plan_uploaded_at`, altrimenti `null`). `ReviewModel { complete(system, user, { timeoutMs }): Promise<ReviewModelResult> }` e `createAnthropicReviewModel(apiKey, { effort? })` in `src/lib/review/model.ts` | `src/lib/review/run-review.ts`, `src/lib/review/model.ts` | 04 | 05a, 05b, 06 |
| C-12 | `ruleAction(summary: Pick<ExerciseWindowSummary, 'sessions' \| 'sets'>, state: SlotState, scale: Scale): ProposalAction` e `computeNextSlotState(state, action, scale, now: number): SlotState` (pure) | `src/lib/plan/progression.ts` | 04 | — |
| C-13 | `ApplyResult = { proposalId; status: 'applied' \| 'not_open' \| 'failed'; error?; hevyRoutineId? }`; `applyProposal({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, proposalId): Promise<ApplyResult>` (prenotazione atomica, T-26) e `applyAllOpen({ db, hevy, now }, reviewId?: string): Promise<ApplyResult[]>` (senza `reviewId`: tutte le `open`, nell'ordine di `PLAN`) | `src/lib/apply/apply-proposal.ts` | 04 | 05b |
| C-14 | `createSessionToken(secret, now)`, `verifySessionToken(token, secret, now)`, `isPasswordCorrect(given, expected)` (tempo costante), `safeNext(value: FormDataEntryValue \| null): string`, `SESSION_COOKIE = 'hc_session'`, `SESSION_DAYS = 365`, `isPublicPath(pathname)` (`src/lib/auth/session.ts`); `sessionDenied(): Promise<NextResponse \| null>` e `assertSession(): Promise<void>` (`src/lib/auth/require-session.ts`) | `src/lib/auth/` | 05a | 05b, 06, 07 |
| C-15 | `EnvSchema` (zod) in `src/lib/env-schema.ts`; `env = EnvSchema.parse(process.env)` server-only in `src/lib/env.ts`, che ri-esporta `EnvSchema`. Chiavi aggiunte da chi le usa: 01 `HEVY_API_KEY` · 03 `TURSO_DATABASE_URL`, `TURSO_AUTH_TOKEN?` · 04 `ANTHROPIC_API_KEY` · 05a `APP_PASSWORD`, `SESSION_SECRET` · 06 `CRON_SECRET`, `HEVY_WEBHOOK_SECRET` · 07 `VAPID_SUBJECT`, `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `NEXT_PUBLIC_VAPID_PUBLIC_KEY`. Fuori schema: `E2E_FAKE_SERVICES` (solo il container). Fuori da Next si legge con `EnvSchema.pick(...)` (D-36) | `src/lib/env-schema.ts`, `src/lib/env.ts` | 01 | tutte |
| C-16 | `PushReport { sent: number; removed: number; failed: number }`; `sendReviewNotification({ db, review, sender?, now? }: { db: Db; review: ReviewRun; sender?: PushSender; now?: number }): Promise<PushReport>` (05b: no-op che ritorna `{ 0, 0, 0 }`; 07: implementazione con sender pigro, T-25) e `sendPushToAll({ db, sender, payload, now }): Promise<PushReport>` (07) | `src/lib/push/send.ts` | 05b (no-op con la firma definitiva) → 07 (implementazione) | 05b, 06, 07 |
| C-17 | Tempo: `ROME`, `parseHevyInstant(iso): number`, `romeDate(ms): string`, `romeDaysBetween(aMs, bMs): number`, `romeDayStartMinusDays(ms, days): number` (03); `nextCronSaturday(ms): string` (data di Roma del primo sabato il cui istante di cron, 06:17 UTC, è > `ms`) e `firstSaturdayOnOrAfter(dateRome: string): string` (05a) | `src/lib/time.ts` | 03 (05a aggiunge le due funzioni del sabato) | 04, 05a, 06 |
| C-18 | Container: `getDb(): Db`, `getHevyClient(): HevyClient`, `getReviewModel(): ReviewModel` (fake con `E2E_FAKE_SERVICES=1`) | `src/lib/container.ts` | 05a | 05b, 06, 07 |

Regola: un contratto si cambia solo nella spec che lo crea; se cambia nome o firma, si aggiornano
questa tabella e OGNI spec viva che lo cita (testo e prompt in fondo). Le firme qui sopra sono
quelle delle spec al 2026-10-24: in caso di scarto fra tabella e spec, vince la spec che crea il
contratto e la tabella è il rilievo da correggere.

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
5. **In esercizio** vuol dire, per 06 e 07: PR mergiata su `main` E deploy di produzione su Vercel
   verde E la misura di chiusura della spec rifatta in produzione (ogni spec la dichiara in § 8);
   gli esiti post-merge si scrivono come primo passo della sessione successiva (D-38). Per 01, 02,
   03, 04, 05a e 05b, che vivono prima del deploy: PR mergiata su `main` E il collaudo di § 7
   annotato in CLAUDE.md (D-37); la 06 rifà in produzione le misure di chiusura di 03–05b.
6. **Ritiro**: una spec in esercizio si ritira con `doc/v1/PROMPT-RITIRO.md` (da generare con
   `claude-config/prompts/08_CREA_RITIRO_SPEC.md` in una sessione a sé, dopo la lettura
   adversariale e prima della spec 01):
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
