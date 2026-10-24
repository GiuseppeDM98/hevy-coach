# 05b — Web app: revisioni, proposte, «Applica» ed «Esegui revisione ora»

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 38 rilievi, 4 decisioni
> (D-43, D-55 e, condivise, D-37, T-26). Pronta per l'implementazione dopo la 05a.
> **Branch**: `feature/05b-app-revisioni` da `main`. **Dipende da**: 04 (`runReview`, `applyProposal`,
> `applyAllOpen`, repository `reviews.ts`), 05a (shell, guard, container, `load*`, E2E, seed,
> DESIGN.md).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-3, D-22, D-29, D-30, D-37, D-43, D-53,
> D-55; scelte T-4, T-9, T-10, T-11, T-12, T-26; contratti C-16 (lo crea come no-op); usa C-3, C-7,
> C-10, C-11, C-13, C-14, C-17, C-18 e `targetText` (02).

## 1. Il problema oggi (prove)

1. Le proposte si vedono e si applicano solo da CLI (`pnpm review:show`, `pnpm review:apply`);
   l'owner vuole farlo dal telefono con un bottone (P4, P5).
2. `runReview` dura quanto la chiamata a Claude (fino a 280 s, D-53): la UI deve mostrare l'attesa e
   la route deve dichiarare `maxDuration` (Vercel Hobby: 300 s).
3. La pagina Oggi (05a) ha un segnaposto `<OpenProposals />` con il solo conteggio.
4. D-3: ogni applicazione è un click esplicito; «Applica tutte» applica solo le proposte **aperte**
   e chiede conferma in un dialogo con l'elenco.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `/` mostra **tutte** le proposte dell'ultima revisione `completed` (D-55), raggruppate per routine nell'ordine di `PLAN`: le `open` in alto come schede con esercizio, azione (badge: Mantieni / Aumenta / Variante successiva), «da → a» (`fromText` → `toText`), motivazione, una riga «La regola avrebbe detto: <azione>» quando `action ≠ ruleAction` (testo visibile, non tooltip: su touch non c'è hover), bottoni «Applica» («Conferma» su una `keep`, che non scrive su Hevy) e «Ignora»; le `applying`, `applied`, `dismissed` come righe compatte con lo stato (niente sparisce, D-55); in testa «Applica tutte (N)» con N = `countActionableProposals` (D-43), nascosto se N = 0, con dialogo di conferma che elenca le N voci e, in fondo, «e M con dati insufficienti: mantieni»; le proposte `insufficient_data` stanno in una sezione chiusa «Dati insufficienti (M)» senza bottoni per scheda: le chiude «Applica tutte» (D-43). | E2E `proposte.spec.ts` con il seed di una revisione con 4 proposte (keep, increase, next_variant, insufficient). Mutazione: elencare solo le `open` → dopo «Applica» la scheda sparisce → rosso. |
| A2 | «Applica» → `POST /api/proposals/[id]/apply` → `applyProposal` (fake Hevy in E2E) → la scheda passa a «Applicata» e, dopo `router.refresh()`, resta come riga compatta; il conteggio di Oggi scende di 1 senza ricaricare; `failed` → la scheda mostra `apply_error` e resta aperta con i bottoni; un secondo «Applica» concorrente ottiene `not_open` (T-26) e la scheda mostra «Già in corso». | E2E: dopo il click la scheda ha lo stato; `page.reload()` e lo stato letto dal server è lo stesso; `tests/e2e/db.ts` legge `e2e.db`: proposta `applied`, `slot_state` di `mon_row` con target 10–14, `hevy_routines.snapshot_json` cambiato (il batch della 04 li scrive solo dopo un PUT riuscito); `e2e-fake-hevy.jsonl` contiene 1 riga `updateRoutine`. Mutazione: togliere `router.refresh()` → il conteggio di Oggi non cambia → rosso. |
| A3 | «Ignora» → `POST /api/proposals/[id]/dismiss` → `{ status: 'dismissed' }` solo da `open`, altrimenti `not_open`. «Applica tutte» → `POST /api/proposals/apply-all` → `applyAllOpen` → risultati per proposta, mostrati uno per uno nel dialogo. | E2E: «Ignora» su keep → «Ignorata»; «Applica tutte» → dialogo con 1 voce (next_variant) + «e 1 con dati insufficienti» → conferma → 2 risultati `applied`. Mutazione: `dismissProposal` senza la condizione `open` → il test unit su una `applied` va rosso. |
| A4 | «Esegui revisione ora» (su `/`, in `/impostazioni` e nello stato vuoto di `/revisioni`) → `POST /api/reviews/run` → `runReview({ trigger: 'manual' })` (ignora i 13 giorni, D-30); bottone disabilitato con spinner, `aria-busy`, testo «Revisione in corso… (fino a 5 minuti)»; `completed`, `failed` o `skipped/no_workouts` → naviga a `/revisioni/[id]`; `skipped/already_running` → avviso «Una revisione è già in corso» con il link a `/revisioni/<id della running>`; `fetch` fallito per rete → «Connessione persa: la revisione può essere ancora in corso» con il link a `/revisioni`. La route chiama `sendReviewNotification` (C-16, no-op qui). | E2E con fake model (1,5 s di attesa): `aria-busy="true"` e il testo, poi `/revisioni/<id>` con «Fixture E2E» nelle motivazioni. Unit: `run/route.ts` contiene `sendReviewNotification(`. Mutazione: togliere la chiamata → rosso. |
| A5 | `/revisioni`: lista (data e ora di `started_at` in Europe/Rome, trigger, stato, N proposte, costo stimato in USD con 2 decimali da `estimated_cost_micros / 1_000_000`, «—» se null), la più recente in alto; `/revisioni/[id]`: summary (o il motivo di una `skipped`: «Nessun allenamento nelle ultime 4 settimane» / «Revisione già in corso»; o «Nessuna chiamata al modello: tutti gli esercizi con dati insufficienti» per una `completed` con `prompt_json` null), finestra, modello, token e costo, tabella delle proposte con azione, regola, stato e motivazione, bottoni «Applica»/«Ignora» su quelle ancora `open`; per una `failed` l'errore. | E2E `revisioni.spec.ts` con 2 revisioni nel seed. Mutazione: ordinare per `started_at` crescente → la prima riga è la `failed` → rosso. |
| A6 | Ogni route di questa spec chiama `sessionDenied()` come prima istruzione; `maxDuration = 300` su `reviews/run` e `apply-all`, `120` su `apply` e `dismiss`; `params` dei route handler e delle pagine dinamiche è una Promise (`await params`, Next 16). | `guards.test.ts` (05a) resta verde; `guards.test.ts` verifica anche `export const maxDuration` in ogni route di `src/app/api/**` tranne `sync` (05a) e che `(app)/layout.tsx` contenga `force-dynamic`. |

## 3. Decisioni

- **D-3**, **D-22**, **D-29** (singola + tutte), **D-30**, **D-37** (in esercizio = PR mergiata +
  collaudo annotato), **D-43** (N esclude le `insufficient_data`; «Applica tutte» le chiude),
  **D-53** (la revisione può durare 280 s: il testo dice «fino a 5 minuti»), **D-55** (`/` elenca
  tutte le proposte dell'ultima revisione `completed`). **T-4** (le pagine sotto `(app)` sono
  `force-dynamic` dalla 05a: `router.refresh()` rilegge il DB), **T-9** (regola accanto a Claude in
  UI), **T-10**, **T-11**, **T-12** (comprende `POST /api/proposals/[id]/dismiss`), **T-26** (la
  prenotazione `applying` della 04 rende innocuo il doppio click).
- Adottate qui, ribaltabili: (a) le quattro azioni sono route handler POST chiamati da Client
  Component con `fetch` e `router.refresh()` dopo; (b) `apply-all` è sequenziale e risponde alla
  fine con l'array (nessuno streaming); (c) la conferma di «Applica tutte» è un `<dialog>` nativo
  stilizzato (DESIGN.md), non un componente di terze parti; (d) il costo si mostra in USD perché i
  prezzi sono in USD; (e) una proposta `superseded` non compare su `/` ma sì nella sua revisione;
  (f) `/revisioni/[id]` ha anche i bottoni per le proposte ancora `open` di quella revisione
  (stesso componente); (g) le tre route delle proposte rispondono sempre **200** con il risultato
  JSON, anche `failed` e `not_open` (un id ignoto è `not_open`, come in `applyProposal`); 500 solo
  per un'eccezione non gestita, che la scheda mostra come «Errore: riprova»; (h) `reviews/run`: un
  errore di `sendReviewNotification` si logga e non cambia la risposta (la revisione è già salvata);
  un'eccezione di `runReview` (impossibile per contratto: ritorna `failed`) → 500 e il bottone
  mostra «Errore: riprova»; (i) l'ordine Lun→Ven delle proposte lo dà `sortByPlan` in
  `src/lib/proposals/view.ts`, non il repository (il DB non conosce la routine); (j) una proposta
  `failed` riprovata con successo perde `apply_error` (lo azzera il batch di `applyProposal`).

## 4. La soluzione

### 4.1 Route — `src/app/api/`

- `reviews/run/route.ts`: `POST`; `const denied = await sessionDenied(); if (denied) return denied;`
  `const db = getDb(); const run = await runReview({ db, hevy: getHevyClient(), model:
  getReviewModel(), now: Date.now(), clock: Date.now, trigger: 'manual' }); try { await
  sendReviewNotification({ db, review: run }); } catch (error) { console.error('push', error); }
  return NextResponse.json(run);` `export const maxDuration = 300`.
- `proposals/[id]/apply/route.ts`: `POST`, guard, `const { id } = await params;`
  `applyProposal({ db: getDb(), hevy: getHevyClient(), now: Date.now() }, id)` → JSON `ApplyResult`;
  `maxDuration = 120`.
- `proposals/[id]/dismiss/route.ts`: `POST`, guard, `await params`; `dismissProposal(db, id):
  Promise<boolean>` (04: `UPDATE … SET status = 'dismissed' WHERE id = ? AND status = 'open'`,
  `true` se ha toccato una riga) → `{ proposalId: id, status: 'dismissed' | 'not_open' }`
  (`DismissResult`, in `src/lib/proposals/view.ts`); `maxDuration = 120`.
- `proposals/apply-all/route.ts`: `POST`, guard, `applyAllOpen({ db: getDb(), hevy:
  getHevyClient(), now: Date.now() })` (senza `reviewId`: tutte le aperte, nell'ordine di `PLAN`,
  comprese le `insufficient_data`, D-43) → JSON `ApplyResult[]`; `maxDuration = 300` (27 proposte
  nel caso peggiore, ognuna con GET + PUT ed eventuale `ensureTemplates`).
- `src/lib/push/send.ts` (C-16, versione no-op): `export interface PushReport { sent: number;
  removed: number; failed: number }` e `export async function sendReviewNotification(_: { db: Db;
  review: ReviewRun }): Promise<PushReport> { return { sent: 0, removed: 0, failed: 0 }; }` con il
  commento `// No-op until Web Push is wired: the real sender keeps this exact signature.`

### 4.2 Repository — `src/lib/db/repositories/reviews.ts` (file della 04, qui si estende)

Esistono già (04): `getLatestCompletedReview`, `getReview`, `listReviews`, `getProposal`,
`listProposalsOfReview`, `listOpenProposals`, `dismissProposal`, `countActionableProposals`. Qui si
aggiunge solo `getReviewWithProposals(db, id): Promise<{ review: ReviewRow; proposals: ProposalRow[]
} | null>` (null per un id ignoto). `ProposalRow` è la riga di `proposals` (tipo inferito da
Drizzle); `listProposalsOfReview` ordina per `slot_id` (stabile), l'ordine di `PLAN` lo dà
`sortByPlan`.

### 4.3 Componenti — `src/components/proposals/`

- `src/lib/proposals/view.ts` (puro, testato): `ProposalView { id, slotId, routineKey, routineTitle,
  exerciseTitle, action, ruleAction, rationale, fromText, toText, status, applyError,
  insufficientData }`; `toProposalView(row: ProposalRow): ProposalView` costruisce `fromText` =
  `${findVariant(from_variant_id).title} · ${targetText(JSON.parse(from_target_json))}` e `toText`
  idem con `to_*`: **dalle colonne della proposta, mai dallo stato corrente** (una proposta
  applicata o vecchia non ha più lo stato di partenza); `targetText` di un `max2` porta la soglia
  (D-18: «max-2 · soglia 15»); `routineTitle` da `routineOf(slotId).title`; `sortByPlan(views):
  ProposalView[]` (ordine di `PLAN`: routine, poi slot); `actionLabel(action)` → «Mantieni» /
  «Aumenta» / «Variante successiva»; `DismissResult`.
- `proposal-card.tsx` (Client): props `ProposalView`; bottoni con stato `idle | working | done |
  error`, `aria-busy` in `working`; chiama le route; `router.refresh()` a fine; mostra «Applicata»,
  «Ignorata», «Già in corso» (`not_open`), o `apply_error`.
- `proposal-row.tsx` (Server): la riga compatta per `applying`, `applied`, `dismissed` (D-55).
- `apply-all-button.tsx` (Client): `<dialog>` con l'elenco (esercizio → azione) delle N e la riga
  «e M con dati insufficienti: mantieni», «Conferma»; risultati mostrati in una lista nel dialogo,
  poi `router.refresh()` alla chiusura. Nascosto se N = 0.
- `run-review-button.tsx` (Client): spinner, testo, `router.push('/revisioni/' + run.id)`; i casi
  di A4.
- `open-proposals.tsx` (Server): sostituisce il segnaposto della 05a; `loadLatestProposals({ db })`
  (`src/lib/proposals/load.ts`: `getLatestCompletedReview` → `listProposalsOfReview` →
  `sortByPlan(rows.map(toProposalView))`); raggruppa per routine; sezione chiusa per le
  `insufficient_data`; nessuna revisione `completed` → «Nessuna revisione ancora» con
  `<RunReviewButton />`.

### 4.4 Pagine

- `src/app/(app)/page.tsx`: `<OpenProposals />` + `<RunReviewButton />` (sotto lo stato).
- `src/app/(app)/revisioni/page.tsx`: lista; stato vuoto «Nessuna revisione ancora: la prima arriva
  sabato» con `<RunReviewButton />` (sostituisce il link «Vai a Oggi» della 05a).
- `src/app/(app)/revisioni/[id]/page.tsx`: `const { id } = await params;`
  `getReviewWithProposals` → dettaglio; `notFound()` se null.
- `src/app/(app)/impostazioni/page.tsx`: aggiunge `<RunReviewButton />`.

### 4.5 Seed ed E2E

- `tests/e2e/seed.ts` esporta `seedE2eDb(client)` (quel che la 05a fa) e la estende con: una
  `reviews` `completed` con `started_at` = 3 giorni fa alle 08:00 di Roma (`finished_at` + 90 s,
  `trigger 'cron'`, `model MODEL`, token 20 000/3 000, `estimated_cost_micros` 140 000, `summary`
  «Seed E2E», `prompt_json` `{"system":"seed","user":"seed"}`) e 4 proposte `open` con
  `from_*`/`to_*` completi: `mon_push_up` `next_variant` con `rule_action 'increase'` (da `push_up`
  max2 15 a `decline_push_up` max2 12: mostra la riga della regola), `mon_row` `increase` (da
  `table_row` 8–12 a 10–14), `wed_plank` `keep` (45–60 s → 45–60 s), `wed_dead_bug` `keep` con
  `insufficient_data = 1`; una `reviews` `failed` con `started_at` = 17 giorni fa, `error 'model:
  refusal (bio)'`. Le sessioni di push-up del seed (−21, −14, −7) rendono `mon_push_up` eleggibile:
  la revisione manuale chiama il modello finto.
- `tests/e2e/seed.ts` esporta anche `resetE2eDb()`: svuota le tabelle e richiama `seedE2eDb` su un
  client `@libsql/client` aperto su `file:e2e.db` (senza cancellare il file aperto dal server) e
  svuota `e2e-fake-hevy.jsonl`; `proposte.spec.ts`, `revisione-manuale.spec.ts` e
  `revisioni.spec.ts` la chiamano in `test.beforeEach` (i progetti `mobile` e `desktop` girano in
  sequenza, `workers: 1`, 05a).
- `tests/e2e/db.ts`: `openE2eDb()` (`createClient({ url: 'file:e2e.db' })`), usato dai test per
  leggere `proposals`, `slot_state`, `hevy_routines` e per inserire la `reviews` `running` finta
  (`started_at = Date.now()`, `window_from = window_to = Date.now()`), cancellata in `finally`.
- `tests/e2e/fakes/ids.ts`: gli id condivisi fra seed e fake (`E2E_ROUTINE_IDS`
  `{ mon_upper_a: 'e2e-routine-mon', … }`, prefisso dei custom).
- `tests/e2e/fakes/fake-hevy.ts` (05a) si completa: `listRoutines`/`getRoutine` servono le 5
  routine del seed (titoli D-25, `folder_id 'e2e-folder'`, esercizi da `buildRoutineBody` sugli
  stati iniziali); `updateRoutine(id, body)` aggiorna la copia in memoria, appende `{ kind:
  'updateRoutine', id, body }` a `e2e-fake-hevy.jsonl` e ritorna la routine; `listExerciseTemplates`
  serve `catalog_bodyweight.json` più i custom del seed, così `decline_push_up` (`C43825EA`) si
  risolve senza `createCustomExercise`; `createCustomExercise` appende e ritorna un id finto.
- `tests/e2e/fakes/fake-model.ts` attende 1500 ms prima di rispondere (così `aria-busy` è
  osservabile).
- `oggi.spec.ts` (05a) si aggiorna ai nuovi valori del seed: «3 proposte da guardare» (D-43) e
  «Prevista sabato <max(nextCronSaturday(now), firstSaturdayOnOrAfter(romeDate(3 giorni fa + 13
  giorni)))>»; è l'unico test della 05a che cambia.

## 5. Vincoli e trappole

- **AGENTS.md § 7 Auth and app** — guard su tutte le route nuove; una riga nuova: ogni bottone che
  chiama una route ha gli stati `idle | working | done | error` visibili, `aria-busy` durante
  l'attesa, e non sparisce senza un messaggio.
- **D-3** — nessun `useEffect` che applica da solo; nessuna applicazione al caricamento.
- `router.refresh()` dopo una mutazione rilegge il DB perché `(app)/layout.tsx` è `force-dynamic`
  (05a, T-4); `guards.test.ts` lo verifica.
- `maxDuration` vale su Vercel: in locale non c'è limite; il collaudo F(3) misura la durata reale
  (annotata in `doc/guide/revisione.md` con la data).
- `apply-all` con un PUT fallito a metà: i risultati precedenti restano applicati (ogni proposta è
  atomica per sé); la UI lo mostra per riga.
- Doppio «Applica» (due schede, o durante «Applica tutte»): la prenotazione `applying` della 04
  (T-26) fa vincere il primo; il secondo vede `not_open`. Nessuna logica qui.
- Un `fetch` di minuti su telefono può cadere: il testo di A4 dice che la revisione può essere
  ancora in corso, e `/revisioni` la mostra.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `src/lib/proposals/view.test.ts`: `fromText`/`toText` per keep (uguali), increase (`Rematore sotto il tavolo · 8–12` → `… · 10–14`), next_variant (`Push-up · max-2 · soglia 15` → `Push-up declinati (piedi sul divano) · max-2 · soglia 12`), increase max-2 (soglia 15 → 17), hold (45–60 s → 55–70 s); una riga `applied` usa ancora le colonne `from_*`; `sortByPlan` (fri, mon, wed → mon, wed, fri); `actionLabel`. | Leggere lo stato corrente al posto di `from_*` → rosso; ordinare per `slotId` → rosso. |
| Unit + DB | `reviews.repository.test.ts` (04, esteso): `getReviewWithProposals` di un id ignoto → null; con 4 proposte → 4 righe; `listReviews` la più recente in alto. | Ordinare per `started_at` crescente → rosso. |
| Unit | `guards.test.ts` (05a) esteso alle nuove route (già automatico); `run/route.ts` contiene `sendReviewNotification(`; ogni route ha `export const maxDuration`; `(app)/layout.tsx` contiene `force-dynamic`. | Togliere la chiamata → rosso; togliere `maxDuration` da `apply-all` → rosso. |
| E2E | `proposte.spec.ts`: 4 proposte visibili (3 schede + 1 in sezione chiusa); Oggi dice «3 proposte da guardare»; «Applica» su increase → «Applicata», conteggio «2 proposte da guardare» senza ricaricare, `page.reload()` → riga compatta «Applicata», `e2e.db` con proposta `applied`, `mon_row` 10–14, snapshot cambiato, 1 riga `updateRoutine` nel jsonl; «Ignora» su keep → «Ignorata»; «Applica tutte» → dialogo con 1 voce + «e 1 con dati insufficienti» → conferma → 2 `applied`, Oggi «Nessuna proposta da guardare»; a 390 e 1280. | Togliere `router.refresh()` → il conteggio resta → rosso; elencare solo le `open` → la riga compatta manca → rosso. |
| E2E | `revisione-manuale.spec.ts`: click «Esegui revisione ora» → `aria-busy="true"` e «Revisione in corso…» → `/revisioni/<id>` con «Fixture E2E» nelle motivazioni; tornando su `/` le 4 proposte del seed sono sparite (superseded, D-30) e ci sono quelle nuove; secondo click durante una `running` finta → avviso «Una revisione è già in corso» con il link, nessuna navigazione. | Ignorare `skipReason` → nessun avviso → rosso. |
| E2E | `revisioni.spec.ts`: lista con 2 righe (completed sopra, failed sotto), costo «0,14 $» e «—»; dettaglio della completed con tabella e i bottoni sulle `open`; dettaglio della failed con «model: refusal (bio)». | Ordinare per `started_at` crescente → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E della 05a verdi (con `oggi.spec.ts` aggiornato come in § 4.5).
  Nessuna chiamata reale.
- **C — Comportamento nuovo**: `pnpm test:e2e` con i tre spec nuovi, verde. Nessuna chiamata reale
  (fake).
- **D — Sotto la UI**: senza cookie `curl -i -X POST localhost:3000/api/proposals/inesistente/apply`
  → 401; con il cookie preso dal browser la stessa chiamata → 200 `{ "proposalId": "inesistente",
  "status": "not_open" }` (nessuna chiamata a Hevy né a Claude); `reviews/run` senza cookie → 401.
- **F — Giro guidato (owner, dev server con `.env.local` reale, DB `local.db`)**: (1) `/`: le
  proposte dell'ultima revisione reale (`pnpm review:show` concorda, lo script usa e getta della 05a
  confronta il conteggio); (2) «Applica» su **una** proposta `keep` (nessuna chiamata Hevy, scrive
  solo `local.db`) → «Applicata» e riga compatta; (3) **prima** l'owner applica o ignora le proposte
  aperte che vuole tenere, perché il passo seguente le rende `superseded` (D-30); poi «Esegui
  revisione ora» (dopo l'OK) — chiamate reali: GET Hevy del sync incrementale (lettura, gratis) e
  **1 chiamata a Claude, 2 se la prima risposta non è valida** (< 0,40 $ ciascuna) → `/revisioni/<id>`
  con il summary; si annota la durata (dal click alla navigazione, e `finished_at − started_at`
  della riga); (4) `/revisioni`: due righe almeno, costo con due decimali; (5) a 390 px le schede
  delle proposte sono leggibili e i bottoni raggiungibili con il pollice. Il bug sarebbe: un bottone
  senza feedback, una proposta applicata due volte, un costo a zero con token > 0. Che cosa NON
  copre: un `increase`/`next_variant` reale su Hevy dall'app (già coperto dalla CLI nella 04 e dagli
  E2E qui).
- **G — Ripristino**: nessuna fixture piantata nel DB reale (la revisione manuale resta: è vera);
  `e2e.db*` e `e2e-fake-hevy.jsonl` cancellati.

## 8. Messa in esercizio

- Env: nessuna nuova. DDL: nessuno.
- Misura di chiusura: una revisione manuale dall'app con proposte visibili su `/`. In esercizio =
  PR mergiata su `main` + collaudo annotato in CLAUDE.md (D-37); la 06 rifà la misura in produzione
  dichiarandone il costo.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Key Features (Oggi › proposte, Revisioni); § Not yet present: + «Undoing
  an applied proposal from the app: by hand on Hevy».
- `AGENTS.md § 7`: la riga sugli stati dei bottoni (§ 5).
- `doc/guide/app.md` (resta ≤ 300 righe): le route, i componenti delle proposte, gli stati dei
  bottoni, il seed esteso, `resetE2eDb`, il canale `e2e-fake-hevy.jsonl`.
- `doc/guide/revisione.md`: la durata misurata della revisione manuale (data).
- `README.md` (inglese): `## Reviewing and applying` con due screenshot del seed E2E a 390 px,
  `doc/screenshots/proposals.png` e `doc/screenshots/review-detail.png`, salvati da
  `screenshots.spec.ts` (05a, tag `@screenshots`).
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/app.md` (≤ 300).

## 10. Fuori ambito, dichiarato

- Notifica vera a fine revisione → **07** (qui C-16 è no-op).
- Cron → **06**.
- Annullare un'applicazione (tornare indietro) → D-22 → CLAUDE.md § Not yet present (riga in § 9).

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/05b-web-app-revisioni-proposte-e-applica.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, DESIGN.md, PRODUCT.md, doc/guide/app.md, doc/guide/revisione.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 05b INTERA
- Verifica con un grep i contratti che usa (runReview, ReviewRun, applyProposal, applyAllOpen,
  ApplyResult, dismissProposal, getLatestCompletedReview, listProposalsOfReview, listReviews,
  countActionableProposals, sessionDenied, getDb, getHevyClient, getReviewModel, targetText,
  findVariant, routineOf, PLAN, romeDate, seedE2eDb, E2E_ROUTINE_IDS)
- Carica la skill impeccable prima di scrivere componenti: DESIGN.md è la legge

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/05b-app-revisioni. Crea SESSION_NOTES.md.

Implementa la spec 05b esattamente come scritta (route, repository, componenti, pagine, seed e
fake, test, collaudo con esito atteso prima — la chiamata reale a Claude del giro guidato, fino a
due, e le GET Hevy del sync partono solo dopo il mio OK, e prima mi ricordi di applicare o
ignorare le proposte aperte che voglio tenere —, documentazione). Se un punto non è coperto,
fermati e chiedimelo con AskUserQuestion. Le etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
