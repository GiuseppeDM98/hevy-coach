# 05b — Web app: revisioni, proposte, «Applica» ed «Esegui revisione ora»

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 05a.
> **Branch**: `feature/05b-app-revisioni` da `main`. **Dipende da**: 04 (`runReview`,
> `applyProposal`, `applyAllOpen`, `dismissProposal`, `shouldRunScheduledReview`), 05a (shell, guard,
> container, E2E, DESIGN.md).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-3, D-22, D-29, D-30; scelte T-9, T-11,
> T-12; contratti C-16 (lo crea come no-op), C-13 (lo usa).

## 1. Il problema oggi (prove)

1. Le proposte si vedono e si applicano solo da CLI (`pnpm review:show`, `pnpm review:apply`);
   l'owner vuole farlo dal telefono con un bottone (P4, P5).
2. `runReview` dura quanto la chiamata a Claude (decine di secondi): la UI deve mostrare l'attesa e
   la route deve dichiarare `maxDuration` (Vercel Hobby: 300 s).
3. La pagina Oggi (05a) ha un segnaposto `<OpenProposals />`.
4. D-3: ogni applicazione è un click esplicito; «Applica tutte» applica solo le proposte **aperte**
   e chiede conferma in un dialogo con l'elenco.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `/` mostra le proposte aperte raggruppate per routine: esercizio, azione (badge: Mantieni / Aumenta / Variante successiva), «da → a» (target o variante), motivazione, un'icona quando `action ≠ ruleAction` con tooltip «La regola avrebbe detto: …», bottoni «Applica» e «Ignora»; in testa «Applica tutte (N)» con dialogo di conferma; le proposte con dati insufficienti stanno in una sezione chiusa «Dati insufficienti (N)». | E2E `proposte.spec.ts` con seed di una revisione con 4 proposte (keep, increase, next_variant, insufficient). |
| A2 | «Applica» → `POST /api/proposals/[id]/apply` → `applyProposal` (fake Hevy in E2E) → la scheda passa a «Applicata» senza ricaricare la pagina; errore → la scheda mostra `apply_error` e resta aperta. | E2E: dopo il click la scheda ha lo stato; il fake Hevy registra 1 PUT. Mutazione: non rileggere lo stato → rosso. |
| A3 | «Ignora» → `POST /api/proposals/[id]/dismiss` → `dismissed`. «Applica tutte» → `POST /api/proposals/apply-all` → risultati per proposta, mostrati uno per uno. | E2E. |
| A4 | «Esegui revisione ora» (su `/` e in `/impostazioni`) → `POST /api/reviews/run` → `runReview({ trigger: 'manual' })` (ignora i 13 giorni, D-30); bottone disabilitato con spinner e testo «Revisione in corso… (fino a 2 minuti)»; alla fine naviga a `/revisioni/[id]`; `skipped/already_running` → avviso «Una revisione è già in corso». La route chiama `sendReviewNotification` (C-16, no-op qui). | E2E con fake model: click → `/revisioni/<id>` con il summary del fake. Unit: la route importa `sendReviewNotification`. |
| A5 | `/revisioni`: lista (data, trigger, stato, N proposte, costo stimato in €¢… in **USD** con 2 decimali), la più recente in alto; `/revisioni/[id]`: summary, finestra, modello, token e costo, tabella delle proposte con azione, regola, stato e motivazione; per una `failed` l'errore. | E2E `revisioni.spec.ts` con 2 revisioni nel seed. |
| A6 | Ogni route di questa spec chiama `sessionDenied()`; `maxDuration = 300` su `reviews/run`, `120` sulle altre. | `guards.test.ts` (05a) resta verde; grep `maxDuration`. |

## 3. Decisioni

- **D-3**, **D-22**, **D-29** (singola + tutte), **D-30**. **T-9** (regola accanto a Claude in UI),
  **T-11**, **T-12**.
- Adottate qui, ribaltabili: (a) le quattro azioni sono route handler POST chiamati da Client
  Component con `fetch` e `router.refresh()` dopo; (b) `apply-all` è sequenziale e risponde alla
  fine con l'array (nessuno streaming); (c) la conferma di «Applica tutte» è un `<dialog>` nativo
  stilizzato (DESIGN.md), non un componente di terze parti; (d) il costo si mostra in USD perché i
  prezzi sono in USD; (e) una proposta `superseded` non compare su `/` ma sì nella sua revisione;
  (f) `/revisioni/[id]` ha anche il bottone «Applica» per le proposte ancora `open` di quella
  revisione (stesso componente).

## 4. La soluzione

### 4.1 Route — `src/app/api/`

- `reviews/run/route.ts`: `POST`; `sessionDenied()`; `const run = await runReview({ db: getDb(),
  hevy: getHevyClient(), model: getReviewModel(), now: Date.now(), trigger: 'manual' })`; `await
  sendReviewNotification({ db, review: run })`; JSON `run`; `export const maxDuration = 300`.
- `proposals/[id]/apply/route.ts`, `proposals/[id]/dismiss/route.ts`, `proposals/apply-all/route.ts`:
  `POST`, guard, chiamano C-13 / `dismissProposal`, rispondono `ApplyResult` o `ApplyResult[]`;
  `maxDuration = 120`.
- `src/lib/push/send.ts` (C-16, versione no-op): `export async function sendReviewNotification(_:
  { db: Db; review: ReviewRun }): Promise<{ sent: number; removed: number }> { return { sent: 0,
  removed: 0 }; }` con il commento che la 07 la riempie.

### 4.2 Repository — `src/lib/db/repositories/reviews.ts`

`listReviews(db)`, `getReviewWithProposals(db, id)`, `listOpenProposals(db)` (join con `reviews`
per la data; esclude `insufficient_data`? **No**: le include con il flag, la UI le separa),
`countOpenProposals(db)` (già usato dalla 05a: qui si sposta nel repository se la 05a l'aveva
inline).

### 4.3 Componenti — `src/components/proposals/`

- `proposal-card.tsx` (Client): props `ProposalView` (id, slotId, routineTitle, exerciseTitle,
  action, ruleAction, rationale, fromText, toText, status, applyError, insufficientData); bottoni
  con stato `idle | working | done | error`; chiama le route; `router.refresh()` a fine.
- `apply-all-button.tsx` (Client): `<dialog>` con l'elenco (esercizio → azione) e «Conferma»;
  risultati mostrati in una lista.
- `run-review-button.tsx` (Client): spinner, testo, `router.push('/revisioni/' + id)`.
- `open-proposals.tsx` (Server): sostituisce il segnaposto della 05a; raggruppa per routine
  nell'ordine `PLAN`; sezione chiusa per i dati insufficienti.
- `src/lib/proposals/view.ts` (puro): `toProposalView(row, states)` costruisce `fromText`/`toText`
  con `targetText` e i titoli delle varianti (`findVariant`); `actionLabel(action)` → «Mantieni» /
  «Aumenta» / «Variante successiva».

### 4.4 Pagine

- `src/app/(app)/page.tsx`: `<OpenProposals />` + `<RunReviewButton />` (sotto lo stato).
- `src/app/(app)/revisioni/page.tsx`: lista; stato vuoto «Nessuna revisione ancora: la prima arriva
  sabato» con il bottone «Esegui revisione ora».
- `src/app/(app)/revisioni/[id]/page.tsx`: dettaglio; `notFound()` se manca.
- `src/app/(app)/impostazioni/page.tsx`: aggiunge `<RunReviewButton />`.

### 4.5 Seed E2E

`tests/e2e/seed.ts` aggiunge: una revisione `completed` di 3 giorni fa con 4 proposte (push-up
`next_variant` con `ruleAction: 'increase'` per vedere l'icona di divergenza; rematore `increase`;
plank `keep`; dead bug `keep` con `insufficient_data`), e una `failed` di 17 giorni fa con
`error: 'model: refusal'`. Il fake Hevy espone `routines` in memoria con gli id delle `hevy_routines`
del seed, così `applyProposal` trova la routine e il PUT riesce.

## 5. Vincoli e trappole

- **AGENTS.md § Auth** — guard su tutte le route nuove.
- **AGENTS.md § UI** — stati: `idle | working | done | error` sempre visibili; nessun bottone che
  sparisce senza un messaggio; `aria-busy` durante l'attesa.
- **D-3** — nessun `useEffect` che applica da solo; nessuna applicazione al caricamento.
- `router.refresh()` dopo una mutazione: le pagine sono dinamiche (leggono il DB), nessuna cache.
- `maxDuration` su `reviews/run` vale su Vercel: in locale non c'è limite; il collaudo misura la
  durata reale (annotata in `doc/guide/revisione.md` con la data).
- `apply-all` con un PUT fallito a metà: i risultati precedenti restano applicati (ogni proposta è
  atomica per sé); la UI lo mostra per riga.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `view.test.ts`: `fromText`/`toText` per keep (uguali), increase (8–12 → 10–14), next_variant (Push-up → Push-up declinati, max-2 15 → 12), hold (45–60" → 55–70"); `actionLabel`. | — |
| Unit | `reviews.repository.test.ts`: `listOpenProposals` esclude `applied/dismissed/superseded`; ordina per routine e slot; `getReviewWithProposals` di un id ignoto → null. | — |
| Unit | `guards.test.ts` esteso alle nuove route (già automatico); `run/route.ts` contiene `sendReviewNotification(`. | Togliere la chiamata → rosso. |
| E2E | `proposte.spec.ts`: 4 proposte visibili (3 + 1 in sezione chiusa); «Applica» su increase → «Applicata», il fake ha 1 PUT; «Ignora» su keep → «Ignorata»; «Applica tutte» → dialogo con 1 voce rimasta (next_variant) → conferma → «Applicata»; a 390 e 1280. | — |
| E2E | `revisione-manuale.spec.ts`: click «Esegui revisione ora» → spinner → `/revisioni/<id>` con «Fixture E2E» nelle motivazioni; secondo click durante una `running` finta (riga inserita dal test) → avviso. | — |
| E2E | `revisioni.spec.ts`: lista con 2 righe (completed, failed), dettaglio con tabella e l'errore della failed. | — |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli + E2E della 05a verdi.
- **C — Comportamento nuovo**: `pnpm test:e2e` con i tre spec nuovi, verde.
- **D — Sotto la UI**: `curl -X POST localhost:3000/api/reviews/run` senza cookie → 401.
- **F — Giro guidato (owner, dev server con `.env.local` reale)**: (1) `/`: le proposte aperte
  dell'ultima revisione reale (`pnpm review:show` concorda); (2) «Applica» su UNA proposta `keep`
  (nessuna scrittura Hevy) → «Applicata»; (3) «Esegui revisione ora» (**una chiamata reale a
  Claude**, < 0,20 $, dopo l'OK) → `/revisioni/<id>` con il summary; (4) `/revisioni`: due righe
  almeno; (5) a 390 px le schede delle proposte sono leggibili e i bottoni raggiungibili con il
  pollice. Il bug sarebbe: un bottone senza feedback, una proposta applicata due volte, un costo a
  zero con token > 0.
- **G — Ripristino**: nessuna fixture piantata nel DB reale (la revisione manuale resta: è vera).

## 8. Messa in esercizio

- Env: nessuna nuova. DDL: nessuno.
- Misura di chiusura: una revisione manuale dall'app con proposte visibili su `/`.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Key Features (Oggi › proposte, Revisioni).
- `doc/guide/app.md`: le route, i componenti delle proposte, gli stati dei bottoni, la durata
  misurata della revisione manuale (data).
- `README.md`: § Reviewing and applying (con screenshot dal seed).

## 10. Fuori ambito, dichiarato

- Notifica vera a fine revisione → **07** (qui C-16 è no-op).
- Cron → **06**.
- Annullare un'applicazione (tornare indietro) → D-22 → CLAUDE.md § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/05b-web-app-revisioni-proposte-e-applica.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, DESIGN.md, PRODUCT.md, doc/guide/app.md, doc/guide/revisione.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 05b INTERA
- Verifica con un grep i contratti che usa (runReview, applyProposal, applyAllOpen,
  dismissProposal, sessionDenied, getDb, getHevyClient, getReviewModel, targetText, findVariant)
- Carica la skill impeccable prima di scrivere componenti: DESIGN.md è la legge

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/05b-app-revisioni. Crea SESSION_NOTES.md.

Implementa la spec 05b esattamente come scritta (route, repository, componenti, pagine, seed,
test, collaudo con esito atteso prima — la chiamata reale a Claude del giro guidato parte
solo dopo il mio OK —, documentazione). Se un punto non è coperto, fermati e chiedimelo con
AskUserQuestion. Le etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
