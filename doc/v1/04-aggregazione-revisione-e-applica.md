# 04 — Aggregazione, revisione con Claude e «Applica» da CLI

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 41 rilievi, 7 decisioni
> (D-40, D-43, D-50, D-51, D-52, D-53 e la scelta T-26). Pronta per l'implementazione dopo la 03.
> **Branch**: `feature/04-review` da `main`. **Dipende da**: 01, 02, 03 (schema, sync, stati, storia,
> `ensureTemplates`, `buildRoutineBody`, `targetText`, `routineOf`, `time.ts`, `createFakeHevy`).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-2, D-3, D-4, D-6, D-7, D-14, D-15, D-18,
> D-19, D-22, D-24, D-28, D-29, D-30, D-31, D-33, D-36, D-39, D-40, D-42, D-43, D-50, D-51, D-52,
> D-53; scelte T-7, T-8, T-9, T-10, T-11, T-15, T-16, T-23, T-26; contratti usati C-1, C-3, C-4,
> C-5, C-6, C-8, C-15, C-17; crea C-7 (`reviews`, `proposals`, `cron_runs`, repository
> `reviews.ts`), C-9, C-10, C-11, C-12, C-13.

## 1. Il problema oggi (prove)

1. In Turso ci sono workout e set (03) e lo stato degli slot, ma nessuno li legge per esercizio né
   decide niente. La doppia progressione (D-6) è scritta nel tema, non nel codice.
2. Un set di un workout Hevy porta `type`, `reps`, `duration_seconds` (più peso, distanza, `rpe`), e
   il template sta sull'esercizio che lo contiene (fixture `workout.json`); in `workout_sets` (03)
   diventano `set_type`, `reps`, `duration_seconds`, `exercise_template_id`. Per dire «quante
   ripetizioni ha chiuso il push-up di lunedì» servono la storia degli slot (03) e la mappa template
   → variante. Due slot possono usare la stessa variante (`hollow_hold` mercoledì e giovedì): il
   `routine_id` del workout distingue; due slot della **stessa** routine possono arrivare alla stessa
   variante (`decline_push_up_pause` è in due scale del giovedì): la posizione nel workout distingue
   (D-50); un workout senza `routine_id` (come il «Push» del 2026-10-22) è ambiguo solo se la
   variante è condivisa.
3. L'API Anthropic (skill `claude-api`, 2026-10-06; `https://platform.claude.com/docs/en/`) con
   `claude-opus-5-5`: niente prefill, niente `tool_choice` forzato, output strutturato con
   `client.messages.parse` + `zodOutputFormat` in `output_config.format`, effort in
   `output_config.effort` (default `medium`), `stop_reason: 'refusal'` possibile, `max_tokens`
   obbligatorio. Prezzo 4 $/M input e 20 $/M output; l'output fatturato comprende il thinking, che
   su Opus 5.5 è sempre attivo: ~15k input, ~3k di JSON per 27 slot più il thinking a effort `high`
   → stima 0,15–0,35 $ a revisione, misurata nel collaudo (§ 7) e annotata in `doc/guide/revisione.md`.
4. Le proposte vanno applicate a Hevy con `GET` + `PUT` completo della routine (D-29) e i custom
   delle varianti successive non esistono ancora (T-7).
5. Il cron di Vercel può invocare due volte (06): la revisione ha bisogno di un lock (T-11).

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `aggregateWindow` (C-9) attribuisce ogni set `normal`/`failure` della finestra a uno slot (per variante attiva alla data del workout, filtrata per `routine_id` quando c'è, k-esima occorrenza → k-esimo slot, D-50), raggruppa per sessione (workout) e calcola per sessione i valori (ripetizioni o secondi per set) e `allAtTop` (D-52); per ogni slot `reviewed` emette un summary, anche vuoto, con le sole sessioni dalla riga di storia corrente (D-51), `sessionCount`, `hasEnoughData` (≥ 2) e `ruleAction`; i set non attribuibili finiscono in `unresolved` con il conteggio. | `aggregate.test.ts` (§ 6). Mutazione: includere i warmup → i valori cambiano → rosso; `[].every` su una sessione senza valori → rosso. |
| A2 | `ruleAction` (C-12): `next_variant` se le ultime **due** sessioni hanno `allAtTop` e la scala ha una variante successiva; `increase` se `allAtTop` ×2 ma la scala è finita; `keep` altrimenti. `computeNextSlotState` applica `increase` (reps +2/+2, hold +10/+10, max2 +2) o `next_variant` (variante successiva con `targets[kind]`). | `progression.test.ts` con casi tabellari. Mutazione: usare una sola sessione → rosso. |
| A3 | `runReview` (C-11): lock atomico (T-11), sync incrementale, finestra di 28 giorni (D-15), regola dei dati scarsi (D-31), prompt § 4.4, **una** chiamata a Claude con lo schema C-10 e `maxRetries: 0` dentro la scadenza unica (D-53), validazione (ogni slot eleggibile esattamente una volta, azione ammessa, `next_variant` solo se esiste), un retry applicativo su errore di validazione se resta margine, salvataggio di `reviews` + `proposals`, `superseded` sulle aperte precedenti. Esiti `completed` / `skipped` (`no_workouts`, `already_running`) / `failed` (con `error`: `sync: …`, `aggregate: …`, `model: timeout`, `model: refusal (<categoria>)`, `model: max_tokens`, `validation: …`, `save: …`, `interrupted`). Mai una riga lasciata `running`. | `run-review.test.ts` con DB in memoria, client Hevy finto e modello finto. Mutazioni in § 6. |
| A4 | `shouldRunScheduledReview(db, now)`: `true` se `romeDaysBetween(riferimento, now) ≥ 13`, dove il riferimento (`scheduledReviewReference(db)`) è `finished_at` dell'ultima revisione `completed`, altrimenti `Number(plan_uploaded_at)`; `false` se nessuno dei due esiste. | Test ai bordi (12 → false, 13 → true) e con stati misti (§ 6). |
| A5 | `applyProposal` (C-13): prenota la proposta `open` in modo atomico (`applying`, T-26) → controlla che lo stato dello slot sia ancora quello di `from_*` → nuovo stato da `to_*` → `ensureTemplates` per la variante nuova se custom e mancante → `GET` routine → `buildRoutineBody` con gli stati aggiornati → `PUT` → **un** batch: stato e storia (`replaceSlotStateStatements`), `proposals.status = 'applied'`, `hevy_routines.snapshot`; `keep` e `insufficient_data` → `applied` senza Hevy; proposta non `open` → `{ status: 'not_open' }`; errore Hevy o stato cambiato → proposta torna `open`, `apply_error` salvato, `failed`. `applyAllOpen` applica in sequenza nell'ordine di `PLAN` e restituisce un risultato per proposta. | `apply-proposal.test.ts`. Mutazione: aggiornare lo stato prima del PUT → il test «PUT fallisce → stato invariato» va rosso; togliere la prenotazione → il test dei due «Applica» concorrenti va rosso. |
| A6 | Token e costo stimato (`estimated_cost_micros`) di ogni revisione sono salvati come **somma** di tutte le chiamate fatte, anche sulle `failed`; una revisione fa al più **due** chiamate (la seconda solo su errore di validazione). | Test: modello finto conta le chiamate (1; 2 con un errore di validazione; 2 e `failed` con due errori, token = somma). |
| A7 | `pnpm review:run` reale produce una revisione completa sui dati veri e `pnpm review:apply -- <id>` applica una proposta su Hevy (dopo l'OK). | Collaudo § 7. |

## 3. Decisioni

- **D-3** (mai automatico), **D-4/D-7** (le scale sono dati: Claude non può uscirne), **D-6**
  (doppia progressione), **D-14** (Opus 5.5, high), **D-15** (28 giorni + revisione precedente),
  **D-18** (la soglia max-2 è `SlotState.target.threshold`, e `targetText` la scrive: «max-2 ·
  soglia 15»), **D-19** (tenute: valore = secondi del set), **D-22** (tre esiti), **D-24** (la
  mobilità è `reviewed: false`: non entra nell'aggregato), **D-28** (cosa cambia «aumenta»), **D-29**
  (una routine alla volta), **D-30** (manuale sempre), **D-31** (dati scarsi in codice), **D-33**
  (gli allenamenti pre-piano contano: D-42 lo rende vero in `variantAt`).
- **D-40** (`pnpm hevy:upload -- --from-db` nasce qui, § 4.6), **D-43** (`insufficient_data` fuori
  da N; «Applica tutte» le chiude), **D-50** (k-esima occorrenza → k-esimo slot), **D-51** (contano
  solo le sessioni dalla riga di storia corrente), **D-52** (`allAtTop` con almeno `slot.sets`
  valori validi, `warmup` e `dropset` scartati), **D-53** (`maxRetries: 0`, `timeout` = tempo che
  resta).
- **T-7** (il custom della variante nuova nasce in «Applica», con `ensureTemplates`), **T-8**,
  **T-9** (`ruleAction` accanto a Claude), **T-10** (parametri della chiamata; niente `fallbacks` né
  beta `server-side-fallback`: un `stop_reason: 'refusal'` o `max_tokens` rende la revisione
  `failed` con quel codice, e il rifiuto porta la sua categoria, `model: refusal (bio)`, così la
  notifica dice perché; owner, 2026-10-23), **T-11** (lock, in una sola istruzione), **T-15**, **T-16**
  (prima revisione), **T-23** (il client Hevy ha il suo timeout: una GET appesa non consuma la
  scadenza oltre 30 s), **T-26** (prenotazione atomica dell'apply, stato `applying`).
- Adottate qui, ribaltabili: (a) sessione = workout (un workout con lo stesso esercizio due volte
  conta una sessione con tutti i set); (b) `allAtTop` (D-52): si contano i set `normal` e `failure`
  con valore non nullo; la sessione è al top se ha **almeno `slot.sets` valori** e **ognuno**
  raggiunge il massimo: `reps ≥ max` per `reps`, `reps ≥ threshold` per `max2`, `duration_seconds ≥
  maxSeconds` per `hold`; per gli slot `perSide` ogni set registrato copre i due lati e il valore è
  quello di un lato (02 § 3 g); (c) le sessioni considerate dalla regola sono le ultime due in
  ordine di data, anche se non consecutive nel calendario; (d) un set con `reps` e
  `duration_seconds` entrambi null è ignorato, e una sessione senza valori non esiste; (e) la
  motivazione di Claude è limitata a 240 caratteri e (f) il `summary` a 600: lo chiede il prompt e
  lo controlla `validateReviewResponse` (errore → retry con l'istruzione), non lo zod dell'output,
  perché l'output strutturato non applica `minLength`/`maxLength` (l'SDK li toglie dallo schema
  inviato e li verifica lato client, dove un fallimento non arriva al retry); (g) il prompt è in
  italiano e chiede motivazioni in italiano; (h) la finestra è `[romeDayStartMinusDays(now, 28),
  now)`: 28 giorni pieni più il giorno corrente; (i) `reviews.prompt_json` = `{ system, user }` della
  **prima** chiamata (stringhe esatte; `null` se il modello non è stato chiamato),
  `raw_response_json` = array delle risposte grezze nell'ordine; (j) il modello e i prezzi stanno in
  `src/lib/review/pricing.ts` (`MODEL = 'claude-opus-5-5'`, `USD_PER_MTOK = { input: 4, output: 20 }`);
  (k) «la revisione precedente» del prompt è l'ultima `completed` prima di questa, con tutte le sue
  proposte non `insufficient_data` e il loro esito; (l) `superseded` si scrive solo quando la nuova
  revisione è `completed`: una `skipped` o `failed` lascia le proposte aperte com'erano; (m)
  `no_workouts` vuol dire «`listSetsBetween` non restituisce righe»: un workout fatto solo di
  esercizi fuori dal piano produce una `completed` con 27 «dati insufficienti»; (n) `unresolved`
  serve alla CLI (`review:run` e `review:show` lo stampano) e ai log: non entra nel prompt né nella
  UI; (o) `finished_at = clock()` (`clock` restituisce epoch ms, come `Date.now`); (p) la CLI in
  locale usa la stessa scadenza di 280 s del cron: stesso codice, stesso comportamento; (q) gli id
  delle righe sono `nanoid(21)`; (r) la GET della routine in «Applica» serve a leggere `folder_id`:
  il PUT riscrive l'intera routine dagli stati del DB e cancella ogni modifica fatta a mano su Hevy
  (02 § 5).

## 4. La soluzione

### 4.1 Schema aggiunto — `src/lib/db/schema.ts` (migrazione `0001_reviews`)

```ts
export const reviews = sqliteTable('reviews', {
  id: text('id').primaryKey(), // nanoid(21)
  trigger: text('trigger').notNull(), // 'cron' | 'manual' | 'cli'
  status: text('status').notNull(), // 'running' | 'completed' | 'skipped' | 'failed'
  skipReason: text('skip_reason'), // 'no_workouts' | 'already_running'
  error: text('error'),
  windowFrom: integer('window_from').notNull(),
  windowTo: integer('window_to').notNull(),
  startedAt: integer('started_at').notNull(),
  finishedAt: integer('finished_at'),
  model: text('model'),
  inputTokens: integer('input_tokens'),
  outputTokens: integer('output_tokens'),
  estimatedCostMicros: integer('estimated_cost_micros'),
  summary: text('summary'),
  promptJson: text('prompt_json'),
  rawResponseJson: text('raw_response_json'),
}, (t) => [index('reviews_started_idx').on(t.startedAt)]);

export const proposals = sqliteTable('proposals', {
  id: text('id').primaryKey(),
  reviewId: text('review_id').notNull().references(() => reviews.id, { onDelete: 'cascade' }),
  slotId: text('slot_id').notNull(),
  action: text('action').notNull(), // ProposalAction
  ruleAction: text('rule_action').notNull(),
  rationale: text('rationale').notNull(),
  fromVariantId: text('from_variant_id').notNull(),
  fromTargetJson: text('from_target_json').notNull(),
  toVariantId: text('to_variant_id').notNull(),
  toTargetJson: text('to_target_json').notNull(),
  status: text('status').notNull(), // 'open' | 'applying' (reserved by applyProposal, T-26) | 'applied' | 'dismissed' | 'superseded'
  appliedAt: integer('applied_at'),
  applyError: text('apply_error'),
  insufficientData: integer('insufficient_data', { mode: 'boolean' }).notNull(),
}, (t) => [index('proposals_review_idx').on(t.reviewId), index('proposals_status_idx').on(t.status)]);

export const cronRuns = sqliteTable('cron_runs', {
  id: text('id').primaryKey(),
  startedAt: integer('started_at').notNull(),
  finishedAt: integer('finished_at'),
  outcome: text('outcome').notNull(), // 'started' | 'review_run' | 'review_not_due' | 'error' (written by spec 06)
  detailsJson: text('details_json').notNull(),
});
```

La migrazione si genera con `pnpm db:generate --name reviews` → `drizzle/0001_reviews.sql`.
`nanoid@6.0.2` e l'SDK si aggiungono qui (`pnpm add nanoid@6.0.2 @anthropic-ai/sdk@1.1.0`).

Tutto l'SQL di questa spec sta in `src/lib/db/repositories/reviews.ts` (AGENTS.md § 2.1):
`insertRunningReviewIfFree(db, row, now): Promise<{ inserted: boolean; runningId: string }>` (il
lock, § 4.4 passo 1), `failStaleRunningReviews(db, now)`, `finishReviewStatements(db, id, patch)`
(stato, errore, token, costo, summary, prompt, raw, `finished_at`; restituisce le istruzioni),
`insertProposalsStatements(db, rows)`, `supersedeOpenProposalsStatement(db, exceptReviewId)`,
`getLatestCompletedReview(db)`, `getReview(db, id)`, `listReviews(db)`, `getProposal(db, id)`,
`listProposalsOfReview(db, reviewId)`, `listOpenProposals(db)`, `reserveProposal(db, id):
Promise<boolean>` (`UPDATE … SET status = 'applying' WHERE id = ? AND status = 'open'`),
`releaseProposal(db, id, applyError)` (torna `open`), `markProposalAppliedStatement(db, id, now)`,
`dismissProposal(db, id)`, `countActionableProposals(db): Promise<number>` (`status = 'open' AND
insufficient_data = 0`, D-43; la 05a, la 05b e la 07 lo usano per N). Le funzioni che finiscono in
`Statements`/`Statement` restituiscono `BatchItem<'sqlite'>` senza eseguire, come
`replaceSlotStateStatements` della 03.

### 4.2 Aggregazione — `src/lib/review/aggregate.ts` (C-9)

`SetRow` (riga piatta di `listSetsBetween`, con `startTime` e `routineId` del workout) e
`SlotStateHistoryRow` (riga di storia con `target` già deserializzato) sono della 03; `listSlotStateHistory(db)` idem.

```ts
export interface SetValue { setIndex: number; value: number }               // reps or seconds
export interface SessionSummary {
  workoutId: string; date: string; startTime: number;
  variantId: VariantId; target: Target;  // what the slot was on that day (variantAt / targetAt)
  values: SetValue[]; allAtTop: boolean;
}
export interface ExerciseWindowSummary {
  slotId: SlotId; routineKey: RoutineKey; variantId: VariantId; target: Target; sets: number; perSide: boolean;
  sinceMs: number;                        // validFrom of the current history row: sessions before it are not in `sessions` (D-51)
  sessions: SessionSummary[]; sessionCount: number; hasEnoughData: boolean; ruleAction: ProposalAction;
  allSessions: SessionSummary[];          // every session of the window, older variants included: Progressi (05a, D-54)
}
export interface AggregateInput {
  sets: SetRow[];                       // listSetsBetween(): flat rows with the workout's startTime and routineId
  states: SlotState[];                  // current
  history: SlotStateHistoryRow[];       // all rows (listSlotStateHistory)
  templateMap: TemplateIdMap;           // variantId → hevy template id
  routineKeys: Map<string, RoutineKey>; // hevy routine id → key
  window: { fromMs: number; toMs: number };
}
export interface AggregateResult { summaries: ExerciseWindowSummary[]; unresolved: { count: number; byTemplate: Record<string, number> } }

export function aggregateWindow(input: AggregateInput): AggregateResult
```

`summaries` ha **esattamente una voce per ogni slot di `PLAN` con `reviewed: true`** (27),
nell'ordine di `PLAN`, anche con `sessions: []` e `sessionCount: 0`; `variantId`, `target`
vengono da `states`, `sets` e `perSide` dallo slot, `sinceMs` dalla riga di storia corrente.

Attribuzione dei set (pura, `resolveSlots`, un workout alla volta): `warmup` e `dropset` → scartati
(D-52). Candidati di un set = slot `reviewed` la cui variante **attiva alla data del workout**
(`variantAt(history, slotId, startTime)`; per un istante anteriore alla prima riga vale la prima,
D-42) ha `templateMap[variantId] === set.exerciseTemplateId`. Se `routineKeys.get(workout.routineId)`
esiste → candidati filtrati su quella routine, e dentro il workout la **k-esima occorrenza** del
template (per `exerciseIndex` crescente) va al **k-esimo slot** candidato nell'ordine della routine
(D-50); le occorrenze oltre il numero di slot → `unresolved`. Senza `routine_id`: un candidato →
attribuito; zero o più di uno → `unresolved`. Un esercizio del piano fatto nel workout di un'altra
routine del piano (hollow il lunedì) ha zero candidati dopo il filtro → `unresolved`: la revisione
guarda le routine, non gli esercizi sparsi. Valore del set: `reps` per target `reps`/`max2`,
`duration_seconds` per `hold`; `null` → scartato. Una sessione che dopo gli scarti non ha nessun
valore **non viene creata**: non entra in `sessions` né in `sessionCount`. Il target di una
sessione è quello in vigore quel giorno (`targetAt`); `allAtTop` come in § 3 (b), contro quel
target e `slot.sets`. In `sessions` entrano **solo le sessioni con `startTime ≥ sinceMs`** (D-51);
`allSessions` le tiene tutte, in ordine di data: le usa Progressi (05a, che chiama `aggregateWindow`
con una `window` di 56 giorni, D-54). `sessionCount`, `hasEnoughData` e `ruleAction` =
`ruleAction(summary, state, scale)` guardano solo `sessions`.

### 4.3 Progressione — `src/lib/plan/progression.ts` (C-12)

```ts
/** Sessions are already filtered to the current history row (D-51) and allAtTop already honours slot.sets (D-52). */
export function ruleAction(summary: Pick<ExerciseWindowSummary, 'sessions'>, state: SlotState, scale: Scale): ProposalAction {
  const last = summary.sessions.slice(-2);
  if (last.length < 2 || !last.every((s) => s.allAtTop)) return 'keep';
  return nextVariant(scale.id, state.variantId) ? 'next_variant' : 'increase';
}

export function increaseTarget(target: Target): Target {
  switch (target.kind) {
    case 'reps': return { kind: 'reps', min: target.min + 2, max: target.max + 2 };
    case 'max2': return { kind: 'max2', threshold: target.threshold + 2 };
    case 'hold': return { kind: 'hold', seconds: target.seconds + 10, maxSeconds: target.maxSeconds + 10 };
  }
}

export function computeNextSlotState(state: SlotState, action: ProposalAction, scale: Scale, now: number): SlotState {
  if (action === 'keep') return state;
  if (action === 'increase') return { ...state, target: increaseTarget(state.target), updatedAt: now, source: 'apply' };
  const next = nextVariant(scale.id, state.variantId);
  if (!next) throw new Error(`No next variant after ${state.variantId} in scale ${scale.id}`);
  const target = next.targets[state.target.kind];
  if (!target) throw new Error(`Variant ${next.id} has no target of kind ${state.target.kind}`);
  return { slotId: state.slotId, variantId: next.id, target, updatedAt: now, source: 'apply' };
}
```

### 4.4 La chiamata a Claude — `src/lib/review/`

`schema.ts` (C-10):

```ts
import { z } from 'zod';
export const ProposalActionSchema = z.enum(['keep', 'increase', 'next_variant']);
export type ProposalAction = z.infer<typeof ProposalActionSchema>;
export const ReviewResponseSchema = z.object({
  // Lengths (summary <= 600, rationale 1-240) are checked by validateReviewResponse: structured
  // outputs do not enforce minLength/maxLength, and a client-side parse failure would skip the retry.
  summary: z.string(),
  proposals: z.array(z.object({ slotId: z.string(), action: ProposalActionSchema, rationale: z.string() })),
});
export type ReviewResponse = z.infer<typeof ReviewResponseSchema>;
export type ReviewStatus = 'running' | 'completed' | 'skipped' | 'failed';
export type ReviewTrigger = 'cron' | 'manual' | 'cli';
export interface ReviewRun { id: string; status: ReviewStatus; trigger: ReviewTrigger; skipReason?: 'no_workouts' | 'already_running'; error?: string; proposalCount: number }
```

`model.ts` — l'adattatore, l'unico file che importa `@anthropic-ai/sdk`:

```ts
import Anthropic from '@anthropic-ai/sdk';
import { zodOutputFormat } from '@anthropic-ai/sdk/helpers/zod';

export interface ReviewModelResult { output: ReviewResponse | null; stopReason: string; refusalCategory: string | null; inputTokens: number; outputTokens: number; raw: unknown }
export interface ReviewCallOptions { timeoutMs: number }
export interface ReviewModel { complete(system: string, user: string, options: ReviewCallOptions): Promise<ReviewModelResult> }
export type ReviewEffort = 'medium' | 'high';

export function createAnthropicReviewModel(apiKey: string, { effort = 'high' }: { effort?: ReviewEffort } = {}): ReviewModel {
  // One attempt per call (D-53): runReview gives it all the time that is left of its deadline; a
  // network failure is reported, not retried, so the cost of a review is predictable.
  const client = new Anthropic({ apiKey, maxRetries: 0 });
  return {
    async complete(system, user, { timeoutMs }) {
      try {
        const response = await client.messages.parse({
          model: MODEL,
          max_tokens: 16000, // thinking is always on and counts toward it: ~3k of JSON plus the thinking at effort high
          system,
          messages: [{ role: 'user', content: user }],
          output_config: { effort, format: zodOutputFormat(ReviewResponseSchema) },
        }, { timeout: timeoutMs });
        return {
          output: response.parsed_output ?? null,
          stopReason: response.stop_reason ?? 'unknown',
          refusalCategory: response.stop_details?.category ?? null, // e.g. 'bio': the notification says why
          inputTokens: response.usage.input_tokens,
          outputTokens: response.usage.output_tokens,
          raw: response,
        };
      } catch (error) {
        if (error instanceof Anthropic.APIConnectionTimeoutError) {
          return { output: null, stopReason: 'timeout', refusalCategory: null, inputTokens: 0, outputTokens: 0, raw: null };
        }
        throw error;
      }
    },
  };
}
```

`Anthropic.APIConnectionTimeoutError` e `stop_details.category` esistono nell'SDK 1.1.0 (skill
`claude-api`, letta il 2026-10-24): si usano così, senza cast.

`prompt.ts` — `buildSystemPrompt(): string` e `buildUserPrompt(input: ReviewPromptInput): string`.
Il **system prompt** (italiano, fisso, testo intero nel file):

> Sei il coach di un allenamento a corpo libero in casa. Ricevi, per ogni esercizio del piano, le
> sessioni delle ultime 4 settimane, la variante corrente, il target e la posizione nella scala di
> varianti, più l'esito della revisione precedente. Per ogni esercizio elencato proponi una sola
> azione: `keep` (mantieni), `increase` (alza il target: +2 ripetizioni, +10 secondi, o +2 sulla
> soglia max-2) o `next_variant` (passa alla variante successiva della scala, se esiste). Regola di
> base: doppia progressione — tutte le serie chiuse al massimo del target per due sessioni → variante
> successiva; se la scala è finita, `increase`. Puoi scostarti dalla regola (per esempio `increase`
> invece di `next_variant` quando il salto è grande, o `keep` dopo un calo) ma devi dirlo nella
> motivazione. Vincoli: proponi solo azioni fra le tre; `next_variant` solo dove
> `nextVariant` non è null; non inventare esercizi; ogni `slotId` di `exercises` compare una volta
> sola in `proposals`. Motivazioni in italiano, massimo 240 caratteri, concrete (cita i numeri).
> `summary`: 2–4 frasi sull'andamento generale. Nell'input, `ruleAction` è la regola calcolata dal
> codice: un suggerimento, non un vincolo; `sets` sono le serie prescritte e `values` i valori
> registrati di ogni serie (ripetizioni, o secondi per le tenute; per gli esercizi `perSide` il
> valore di una serie è quello di un lato); per i target «max-2» la soglia è il numero di ripetizioni
> su tutte le serie che fa passare alla variante successiva; le `sessions` sono solo quelle dall'ultimo
> cambio di variante o di target (`since`); `previousReview` dice che cosa è stato proposto l'ultima
> volta e se è stato applicato; gli slot di `skippedForInsufficientData` sono solo contesto e non
> vanno in `proposals`. Se c'è `validationErrors`, la tua risposta precedente aveva quegli errori:
> correggili.

Lo **user prompt** è `JSON.stringify(input)` di un `ReviewPromptInput` (esportato da `prompt.ts`):

```ts
export interface PromptSession { date: string; values: number[]; allAtTop: boolean }       // date = romeDate
export interface PromptExercise {
  slotId: SlotId; routine: string /* title, e.g. «Lun · Upper A» */; exercise: string /* variant title */;
  scalePosition: string /* "3/6" */; nextVariant: string | null; target: string /* targetText */;
  sets: number; perSide: boolean; since: string /* romeDate(sinceMs) */; sessions: PromptSession[]; ruleAction: ProposalAction;
}
export interface PreviousReview { date: string; proposals: Array<{ slotId: SlotId; action: ProposalAction; applied: boolean }> }
export interface ReviewPromptInput {
  window: { from: string; to: string };          // romeDate of fromMs and of toMs
  exercises: PromptExercise[];                    // eligible slots only
  previousReview: PreviousReview | null;          // § 3 (k)
  skippedForInsufficientData: SlotId[];           // 0–1 sessions: context only
  validationErrors?: string[];                    // retry only
}
```

Niente dati personali: nessun nome, nessun id Hevy.

`run-review.ts` (C-11):

```ts
export const REVIEW_DEADLINE_MS = 280_000; // 300 s of Vercel's maxDuration, minus the save and the notification
export const MIN_MODEL_CALL_MS = 60_000;   // below this a call is not started: it would be killed mid-way
export const STALE_RUNNING_MS = 600_000;   // a `running` row older than this was killed: it is closed as failed
export async function runReview({ db, hevy, model, now, clock, trigger }: { db: Db; hevy: HevyClient; model: ReviewModel; now: number; clock: () => number; trigger: ReviewTrigger }): Promise<ReviewRun>
export async function shouldRunScheduledReview(db: Db, now: number): Promise<boolean>
export async function scheduledReviewReference(db: Db): Promise<number | null>
```

`now` è l'istante d'inizio della richiesta (il cron e la route lo prendono prima del loro sync), e
la **scadenza unica** della revisione è `deadline = now + REVIEW_DEADLINE_MS`. `clock` restituisce
epoch ms e dice che ora è: le route e la CLI passano `Date.now`, i test un orologio finto. Non è una
lettura dell'orologio in un modulo puro: `run-review.ts` è un caso d'uso.

Passi, nell'ordine (ognuno con il suo test):
0. **Righe orfane**: `failStaleRunningReviews(db, now)`: le `running` con `started_at ≤ now −
   STALE_RUNNING_MS` diventano `failed` con `error = 'interrupted'` e `finished_at = now` (una
   function uccisa a 300 s non chiude la sua riga).
1. **Lock**, in una sola istruzione (`insertRunningReviewIfFree`): `INSERT INTO reviews (…) SELECT …
   WHERE NOT EXISTS (SELECT 1 FROM reviews WHERE status = 'running' AND started_at > :now − 600000)`;
   `rowsAffected = 0` → `skipped/already_running` con `id` = l'id della `running` trovata e
   `proposalCount: 0` (nessuna riga nuova). SQLite serializza le scritture: due cron simultanei non
   passano entrambi. La riga nasce con `window_from = romeDayStartMinusDays(now, 28)`, `window_to =
   now`, `started_at = now`, `model = MODEL`.
2. **Sync**: `syncWorkouts({ db, hevy, now })` (incrementale). Da qui al passo 6 tutto sta in un
   `try`: qualunque eccezione → `failed` con `error = '<fase>: ' + message` (`sync`, `aggregate`,
   `model`, `save`), `finished_at = clock()`, mai una riga lasciata `running`.
3. **Finestra e aggregazione**: `listSetsBetween(db, window)`; **zero righe** → `skipped/no_workouts`
   (§ 3 m), `finished_at`. Altrimenti `aggregateWindow({ sets, states: listSlotStates, history:
   listSlotStateHistory, templateMap: getTemplateMap, routineKeys: routineKeyByHevyId, window })`.
4. **Dati scarsi (D-31)**: `eligible = summaries.filter(hasEnoughData)`; gli altri generano
   proposte `keep` con `rationale = 'Dati insufficienti: meno di 2 sessioni nelle ultime 4
   settimane.'`, `insufficient_data = true`, senza passare da Claude. Se `eligible` è vuoto → tutte
   le proposte sono «dati insufficienti», `completed` senza chiamata al modello (`prompt_json` null).
5. **Chiamata**, dentro la scadenza (D-53): `remaining = deadline − clock()`; se `remaining <
   MIN_MODEL_CALL_MS` → `failed` (`error = 'model: timeout'`), senza chiamata. Altrimenti
   `model.complete(system, user, { timeoutMs: remaining })`: un solo tentativo con tutto il tempo
   che resta. `stopReason: 'timeout'` → `failed` (`error = 'model: timeout'`); `refusal` → `failed`
   (`error = 'model: refusal (' + refusalCategory + ')'`, oppure `'model: refusal'` senza
   categoria); `stopReason` ≠ `end_turn` → `failed` (`error = 'model: ' + stopReason`, es. `model:
   max_tokens`); un'eccezione del modello → `failed` (`error = 'model: ' + message`). `output` null
   con `end_turn` è un **errore di validazione** (`'schema: risposta non conforme'`) e segue il
   percorso qui sotto. Altrimenti **validazione**: `validateReviewResponse(output, eligible):
   string[]` (`src/lib/review/validate.ts`; vuoto = valido; ogni errore comincia con lo `slotId` o
   con `summary:`): insieme degli `slotId` uguale a quello di `eligible`, azioni ammesse,
   `next_variant` solo con successore, `rationale` di 1–240 caratteri, `summary` ≤ 600. Su errore di
   validazione **un** secondo tentativo con lo stesso `ReviewPromptInput` più la chiave
   `validationErrors`, `timeoutMs = deadline − clock()`, **solo** se `deadline − clock() ≥
   MIN_MODEL_CALL_MS`; senza margine, o al secondo errore → `failed` (`error = 'validation: ' +
   errors.join('; ')`). Token e costo si **sommano** su tutte le chiamate fatte (anche `failed`);
   `raw_response_json` conserva l'array delle risposte grezze.
6. **Salvataggio** in **un** batch: `insertProposalsStatements` (una per slot eleggibile + quelle di
   dati scarsi) con `from_*` = stato corrente e `to_*` da `computeNextSlotState(state, action,
   scale, now)`, `status='open'`; `supersedeOpenProposalsStatement(db, thisId)` (`UPDATE proposals
   SET status='superseded' WHERE status='open' AND review_id != this`; solo qui, § 3 l);
   `finishReviewStatements` → `completed`, token, costo (`estimated_cost_micros = inputTokens *
   USD_PER_MTOK.input + outputTokens * USD_PER_MTOK.output`: token × $/MTok = micro-USD), `summary`,
   `prompt_json`, `raw_response_json`, `finished_at = clock()`.
7. Ritorna `ReviewRun`.

`scheduledReviewReference(db)`: `finished_at` dell'ultima `completed`, altrimenti
`Number(plan_uploaded_at)` da `sync_state`, altrimenti `null`. `shouldRunScheduledReview(db, now)` =
`ref !== null && romeDaysBetween(ref, now) ≥ 13`. Una `skipped/no_workouts` ogni sabato (owner in
vacanza) non sposta il riferimento: la notifica «saltata» arriva ogni sabato, ed è voluto (T-17).

### 4.5 «Applica» — `src/lib/apply/apply-proposal.ts` (C-13)

```ts
export type ApplyResult = { proposalId: string; status: 'applied' | 'not_open' | 'failed'; error?: string; hevyRoutineId?: string }
export async function applyProposal({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, proposalId: string): Promise<ApplyResult>
/** Without reviewId: every `open` proposal, in PLAN order (routine, then slot). */
export async function applyAllOpen({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, reviewId?: string): Promise<ApplyResult[]>
```

Ordine in `applyProposal`:
1. **Prenotazione** (T-26): `reserveProposal(db, id)` (`UPDATE proposals SET status = 'applying'
   WHERE id = ? AND status = 'open'`); `false` → `{ status: 'not_open' }` (proposta inesistente,
   già applicata, ignorata, superata o in corso in un'altra richiesta). Da qui ogni uscita diversa
   da `applied` passa per `releaseProposal(db, id, applyError)`, che riporta `open`.
2. `keep` o `insufficient_data` → batch `[markProposalAppliedStatement]` → `applied`, senza Hevy.
3. **Stato ancora valido**: `state = listSlotStates(db)` dello slot; se `state.variantId !==
   from_variant_id` o `JSON.stringify(state.target) !== from_target_json` → `failed`, `apply_error =
   'stale: lo stato dello slot è cambiato dopo la revisione'`, nessuna chiamata Hevy (l'owner ha
   confermato il «da → a» che vede, D-3). Il nuovo stato è `nextState = { slotId, variantId:
   to_variant_id, target: JSON.parse(to_target_json), updatedAt: now, source: 'apply' }`.
4. **Template**: `templateMap = getTemplateMap(db)`; se manca `templateMap[nextState.variantId]` →
   `variant = findVariant(nextState.variantId)`; `existing = await hevy.listExerciseTemplates()`
   (5 GET); `created = await ensureTemplates(hevy, [variant], existing)` (1 POST **permanente** se
   la variante è custom e non esiste per titolo, T-7); `upsertTemplateMap(db, [{ variantId,
   hevyTemplateId: created[variantId], title: variant.hevy.title, isCustom: variant.hevy.kind ===
   'custom' }], now)` (fuori dal batch finale: è idempotente e il template esiste comunque).
5. **Routine**: `routine = routineOf(slotId)` (C-3); `routineRow = listHevyRoutines(db).find((r) =>
   r.routineKey === routine.key)` (assente → `failed`, `apply_error = 'routine not synced: run pnpm
   hevy:sync'`); `current = await hevy.getRoutine(routineRow.hevyRoutineId)` (404 → `failed`,
   `apply_error`); `states = listSlotStates(db).map((s) => s.slotId === slotId ? nextState : s)`;
   `body = buildRoutineBody(routine, states, templateMap, current.folder_id)`; `saved = await
   hevy.updateRoutine(current.id, body)` (sostituzione completa, § 3 r).
6. **Solo dopo il PUT riuscito**, in **un** batch: `...replaceSlotStateStatements(db, nextState, {
   validFrom: now, proposalId })`, `markProposalAppliedStatement(db, id, now)`, update di
   `hevy_routines` (`snapshot_json = JSON.stringify(saved)`, `hevy_updated_at =
   parseHevyInstant(saved.updated_at)`, `synced_at = now`).

Un errore in qualunque punto fra 3 e 5 lascia `slot_state`, storia e `hevy_routines` invariati,
salva `apply_error` con `releaseProposal` e ritorna `failed`. Un errore nel batch del passo 6 (raro:
il PUT è già passato) → `failed` con `apply_error = 'save: ' + message`, e il prossimo «Applica»
trova lo stato `stale`: si risincronizza con `pnpm hevy:sync` e si rifà la revisione.

`dismissProposal(db, id)` → `status='dismissed'` solo da `open` (la 05b la usa; qui repository +
test).

### 4.6 CLI — `scripts/`

Tutti nella forma `main()` della 01, env con `EnvSchema.pick(...)`, script di package.json
`"review:run": "tsx --env-file-if-exists=.env.local scripts/review-run.ts"` e così via.

- `review-run.ts` (`pnpm review:run`): `runReview` con `trigger: 'cli'`, `model =
  createAnthropicReviewModel(env.ANTHROPIC_API_KEY)`, `now = Date.now()`, `clock = Date.now`;
  stampa stato, errore, token, costo (`estimated_cost=<USD a 4 decimali>`), `unresolved.count` e le
  proposte in tabella (slot, variante, azione, regola, motivazione, `insufficient_data`).
- `review-apply.ts` (`pnpm review:apply -- <proposalId> | --all`): **non** chiama `applyAllOpen`:
  legge le proposte `open` (una, o tutte nell'ordine di `PLAN`), per ognuna stampa slot, azione e
  «da → a» (da `from_*`/`to_*` con `targetText`) e, **prima** di ogni chiamata a `applyProposal`,
  chiede `y/N` con `readline` (anche per le `keep`), perché D-3 vale anche da CLI; su `n` passa alla
  successiva. Prima del `y` stampa anche l'effetto su Hevy: «GET + PUT» oppure «GET del catalogo +
  POST permanente del custom «<titolo>» + GET + PUT» (§ 7 D).
- `review-show.ts` (`pnpm review:show`): l'ultima revisione (qualunque stato) con token, costo,
  errore, `sessionCount` per slot e le proposte con il loro stato.
- `review-compare.ts` (`pnpm review:compare -- --effort medium`): rimanda `prompt_json` (system e
  user) dell'ultima revisione `completed` **con `prompt_json` non null** a
  `createAnthropicReviewModel(key, { effort })` con `timeoutMs: REVIEW_DEADLINE_MS` e stampa,
  accanto a quelli salvati, validazione, proposte che cambiano azione, token, costo e durata. Se
  non c'è nessuna revisione con chiamata al modello esce con «nessuna revisione con chiamata al
  modello». **Non scrive nulla** (nessuna riga, nessuna proposta). Serve al collaudo C2 per decidere
  l'effort con un dato.
- `hevy-upload.ts` (02) riceve qui il flag **`--from-db`** (D-40): con il flag, `states =
  listSlotStates(db)` (il DB deve avere 30 righe, altrimenti esce con «run pnpm hevy:sync first»);
  senza il flag, se `slot_state` contiene almeno una riga `source = 'apply'`, lo script si **rifiuta**
  di scrivere con «slot_state has applied changes: use --from-db» (il dry-run resta permesso e lo
  dice). `doc/guide/piano.md` e SETUP.md (06) citano solo la forma `--from-db` dopo il primo
  «Applica».

`EnvSchema` aggiunge `ANTHROPIC_API_KEY: z.string().min(1)`; `.env.example` idem (chiave della
Console personale, D-2); `vitest.config.ts` aggiunge il valore finto `ANTHROPIC_API_KEY: 'test'`.

## 5. Vincoli e trappole

- **AGENTS.md § 6 Claude** (stub scritto al bootstrap; questa spec lo completa) — qui vuol dire che:
  `@anthropic-ai/sdk` si importa in un solo file (`model.ts`); il modello e i prezzi stanno in
  `pricing.ts`; `max_tokens` esplicito; `effort` esplicito (il default di Opus 5.5 è `medium`);
  `maxRetries: 0` (D-53); niente prefill, niente `tool_choice`, niente `fallbacks`; la risposta si
  legge da `parsed_output` e si rivalida comunque (`validateReviewResponse`): lo schema garantisce
  la forma, non i vincoli di dominio.
- **AGENTS.md § 5 Database** — tutte le scritture di una revisione o di un'applicazione sono in un
  `batch`; mai due batch dove uno basta; le funzioni `…Statements` restituiscono istruzioni, non le
  eseguono.
- **D-3** — nessun percorso applica una proposta senza una chiamata esplicita a `applyProposal`
  (`boundaries.test.ts`: il sorgente di `run-review.ts` non contiene `apply-proposal`).
- Il retry della validazione **rimanda tutto il prompt** più gli errori: non è una conversazione
  (niente storia, niente cache).
- `Date.now()` non si chiama nei moduli puri: `now` è sempre un parametro.
- Dati personali: il prompt non contiene nomi, id Hevy o date di nascita; i titoli delle varianti
  sono quelli del piano.
- Un «Applica» mentre una revisione è in corso non crea danni: la revisione salva `from_*` dagli
  stati letti prima e, se l'owner applica quella proposta dopo, il controllo `stale` del passo 3 la
  rifiuta.
- Un set registrato con un'altra variante della stessa scala al posto di quella della routine (l'owner
  sostituisce l'esercizio nell'app Hevy) finisce in `unresolved`: la revisione giudica la variante
  prescritta.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
Il client Hevy finto dei test è `createFakeHevy` (02) esteso qui con `getRoutine`/`updateRoutine`
già presenti e, dove serve una risposta d'errore, `createHevyClient` + `fakeFetch` (01). Il modello
finto è un `ReviewModel` con una coda di `ReviewModelResult` e la lista delle chiamate ricevute
(`{ system, user, options }`).

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `aggregate.test.ts`: fixture inline (righe `SetRow` inventate: 3 workout del `mon_upper_a` con push-up 4×[14,15,15,15] / [15,15,15,15] / [16,16,15,15], rematore 8–12 con [12,12,12,11]…): attribuzione per `routine_id`; `hollow_hold` condiviso → attribuito per routine; workout senza `routine_id` e variante univoca → attribuito; variante condivisa senza routine → `unresolved`; **due slot del giovedì sulla stessa variante** → prima occorrenza a `thu_decline_push_up`, seconda a `thu_push_up_pause`, terza `unresolved` (D-50); warmup e dropset esclusi; set con valori null esclusi; workout con tutti i set `reps: null` → nessuna sessione; sessione con 2 valori su `sets: 4` tutti al massimo → `allAtTop false`, con 4 → `true`, con 5 → `true` (D-52); storia: un set datato prima di un cambio variante va alla variante vecchia (`SessionSummary.variantId`) e **non entra** nel summary (D-51: `sinceMs`); set datato prima di `plan_uploaded_at` con la variante iniziale → attribuito (D-42); finestra senza set → 27 summary tutti `hasEnoughData: false`; `hasEnoughData` false con 1 sessione. | Togliere il filtro sui warmup → rosso; usare lo stato corrente invece della storia → rosso; `[].every` senza lo scarto → `sessionCount` 1 → rosso; ignorare `slot.sets` → il caso 2/4 va rosso; attribuire per posizione senza il filtro di routine → rosso. |
| Unit | `progression.test.ts`: `ruleAction` (0/1/2 sessioni, allAtTop misto, scala finita → `increase`); `increaseTarget` per i tre generi; `computeNextSlotState` `next_variant` su `push_up` → `decline_push_up` con `max2 12`; su `wall_sit_one_leg` (ultima) → throw. | Soglia a una sessione → rosso. |
| Unit | `validate.test.ts`: slot mancante, slot in più, slot doppio, azione sconosciuta, `next_variant` senza successore, motivazione vuota o di 241 caratteri, `summary` di 601 → errori che cominciano con lo `slotId` (o `summary:`); risposta buona → `[]`. | Togliere il controllo delle lunghezze → il caso dei 241 caratteri va rosso. |
| Unit | `prompt.test.ts`: lo user prompt di uno slot `max2` contiene «soglia 15», `sets`, `perSide` e `since`; il system prompt contiene «validationErrors»; `JSON.parse(buildUserPrompt(input))` ha le chiavi di `ReviewPromptInput`; nessun id Hevy né nome dentro. | Togliere `target` dal JSON → rosso. |
| Unit + DB | `run-review.test.ts`: (1) lock: riga `running` fresca → `already_running` con il suo `id`, nessuna chiamata; (1b) due `runReview` con `Promise.all` sullo stesso DB → uno `completed`, uno `already_running`, una chiamata al modello; (2) `no_workouts`; (3) percorso completo con modello finto che risponde bene → `completed`, 27 proposte, costo = `in×4 + out×20`, precedenti `superseded`, `prompt_json` con `system` e `user`, `finished_at` = orologio finto; (4) risposta con slot mancante poi buona → 2 chiamate, la seconda con `validationErrors` nello user prompt, `completed`; (5) due risposte cattive → `failed` con `validation:`, 2 chiamate, token = somma, `raw_response_json` con 2 elementi; (6) `stop_reason: 'refusal'` con categoria `bio` → `failed`, `error = 'model: refusal (bio)'`; (7) slot con 1 sessione → proposta `keep` con `insufficient_data`, non inviato al modello (il prompt catturato non lo contiene in `exercises` ma in `skippedForInsufficientData`); (8) orologio finto già oltre `deadline − MIN_MODEL_CALL_MS` → `failed`, `model: timeout`, nessuna chiamata; (9) prima risposta non valida e orologio che, dopo la chiamata, lascia meno di `MIN_MODEL_CALL_MS` → `failed` con `validation:`, 1 chiamata sola; (10) `timeoutMs` passato al modello = `deadline − clock()` (con 250 s restanti → 250 000); (11) il modello lancia → `failed` con `model:`, la riga non resta `running`; (12) client Hevy che lancia nel sync → `failed`, `error` che inizia con `sync: `; (13) `stopReason: 'max_tokens'` → `failed`, `model: max_tokens`; (14) `output: null` con `end_turn` poi una buona → 2 chiamate, `completed`; (15) riga `running` di 11 minuti → diventa `failed/interrupted` e la nuova revisione parte; (16) `skipped/no_workouts` e `failed` lasciano `open` le proposte precedenti; (17) tutti gli slot con < 2 sessioni → `completed` senza chiamata, `prompt_json` null, 27 `insufficient_data`. | Non marcare `superseded` → (3) rosso; retry infinito → (5) rosso; ignorare la scadenza prima del retry → (9) rosso; lock «controlla poi inserisci» → (1b) rosso; non chiudere le orfane → (15) rosso. |
| Unit + DB | `should-run.test.ts`: nessun riferimento → false; `plan_uploaded_at` 12 giorni fa → false; 13 → true; ultima `completed` 13 giorni fa → true; ultima `completed` 20 giorni fa e una `failed` 5 giorni fa → true; `skipped` 2 giorni fa e `completed` 14 giorni fa → true; `completed` 5 giorni fa → false. | Usare `started_at` di qualunque stato → il caso «completed 20, failed 5» va rosso. |
| Unit + DB | `reviews.repository.test.ts`: `dismissProposal` su `open` → `dismissed`, su `applied` → invariata; `reserveProposal` due volte → `true` poi `false`; `releaseProposal` → `open` con `apply_error`; `countActionableProposals` con 3 `open` di cui 1 `insufficient_data` → 2 (D-43); `insertRunningReviewIfFree` con una `running` fresca → `inserted false` e il suo id. | Contare anche le `insufficient_data` → rosso. |
| Unit + DB | `apply-proposal.test.ts`: `keep` → `applied` senza chiamate Hevy; `insufficient_data` → `applied` senza Hevy; `next_variant` su custom mancante → 1 `createCustomExercise` + GET + PUT, stato e storia aggiornati in un batch, `hevy_routines.snapshot` nuovo, proposta `applied`; PUT che fallisce (500×2 via `fakeFetch`) → `failed`, stato invariato, proposta tornata `open` con `apply_error`; stato dello slot cambiato dopo la revisione → `failed/stale`, 0 chiamate Hevy; proposta `applied` → `not_open`; due `applyProposal` con `Promise.all` sulla stessa proposta → uno `applied`, uno `not_open`, un solo PUT; `applyAllOpen` con 3 aperte (keep, increase, next) → 3 risultati nell'ordine di `PLAN`. | Scrivere lo stato prima del PUT → rosso; togliere la prenotazione → due PUT → rosso; togliere il controllo `stale` → rosso. |
| Unit | `boundaries.test.ts` (in `src/lib/review/`): il sorgente di `run-review.ts` non contiene la stringa `apply-proposal`. | Importarlo → rosso. |
| Unit + DB | `hevy-upload` con `--from-db` (`scripts/hevy-upload.test.ts` sulla funzione estratta `resolveStates(argv, db)`): con il flag legge `slot_state`; senza il flag e una riga `source = 'apply'` → lancia con «use --from-db». | Togliere il rifiuto → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli verdi; `pnpm hevy:sync` ancora incrementale (GET reali di lettura,
  gratis).
- **C — Comportamento nuovo**: `pnpm review:run`. **Fino a due chiamate reali ad Anthropic** (la
  seconda solo su errore di validazione): costo atteso < 0,40 $, massimo < 0,80 $, letto dalla riga
  `estimated_cost` stampata (somma delle chiamate); **sole GET** verso Hevy (sync incrementale). Si
  annota la durata della chiamata. Esito atteso: `completed`; 27 proposte; quelle con
  `insufficient_data` sono esattamente gli slot con meno di 2 sessioni dalla riga di storia corrente
  (`pnpm review:show` le elenca con `sessionCount`); ogni proposta non `insufficient_data` ha una
  motivazione in italiano ≤ 240 caratteri; ogni proposta con `action ≠ ruleAction` ha una
  motivazione che dice perché si scosta; `unresolved.count` stampato. Se è `failed`, l'errore è
  leggibile e la riga `reviews` lo conserva.
- **C2 — Effort `medium` a confronto (dopo l'OK, ≈ 0,20 $)**: `pnpm review:compare -- --effort
  medium` sullo stesso input di C. **Una chiamata reale ad Anthropic**, nessuna scrittura. Esito
  atteso: risposta valida; si leggono, accanto a quelli di C, le proposte che cambiano azione, i
  token di output, il costo e la durata. Il confronto si annota in `doc/guide/revisione.md` con la
  data. L'effort resta `high` (D-14) finché l'owner non decide altrimenti su quel dato.
- **D — Scrittura reale su Hevy (dopo l'OK)**: `pnpm review:apply -- <id di una proposta
  'increase' o 'next_variant'>`. Lo script stampa l'effetto previsto **prima** del `y`: se la
  variante di arrivo è custom e non è in `hevy_templates`, **1 POST permanente**
  `exercise_templates` (titolo esatto, non si cancella via API) più 5 GET del catalogo, 1 GET e 1
  PUT della routine; altrimenti 1 GET + 1 PUT (reversibile con un altro PUT). L'owner dà l'OK
  sapendo quale dei due casi è. Esito atteso: su Hevy la routine mostra il nuovo range/variante;
  `slot_state` e `slot_state_history` aggiornati (la vecchia riga con `valid_to`);
  `proposals.status = 'applied'`. Se non c'è nessuna proposta diversa da `keep` (piano giovane), si
  applica una proposta `keep` (nessuna chiamata Hevy) e si **simula** un `increase`: uno script
  usa-e-getta (fuori dal repo) salva lo stato di partenza di `mon_row` da `slot_state`, inserisce
  una `reviews` finta (`trigger 'cli'`, `status 'completed'`, `summary 'collaudo'`) e una proposta
  `open` su `mon_row` (`action 'increase'`, `rule_action 'increase'`, `from` 8–12, `to` 10–14,
  `insufficient_data false`); si applica con `pnpm review:apply -- <id finto>` (1 GET + 1 PUT), si
  verifica su Hevy; poi il secondo script fa il PUT di `mon_upper_a` con `buildRoutineBody` sugli
  stati di partenza (1 GET + 1 PUT), `DELETE FROM slot_state_history WHERE proposal_id = <finto>`,
  `UPDATE slot_state_history SET valid_to = NULL` sulla riga chiusa, `slot_state` di `mon_row`
  riportato ai valori salvati, `DELETE FROM reviews WHERE id = <finto>` (cascade sulle proposte).
- **F — Giro guidato (owner, app Hevy)**: la routine toccata mostra il cambiamento (1 cosa).
- **G — Ripristino**: script usa-e-getta cancellati; se è stato simulato, Hevy e DB riportati allo
  stato di partenza con i passi di D; la `reviews` vera di C resta (è storia vera).

Che cosa NON copre: cron, UI, notifiche.

## 8. Messa in esercizio

- Env: `ANTHROPIC_API_KEY` (Console personale). DDL: `drizzle/0001_reviews.sql` via `pnpm db:migrate`.
- Misura di chiusura: `pnpm review:show` mostra una revisione `completed` sui dati veri. In
  esercizio = PR mergiata su `main` + collaudo annotato in CLAUDE.md (D-37); la 06 rifà la misura in
  produzione.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Key Features «Review» e «Apply» con i puntatori; § Data & Integrations
  (Anthropic: modello, costo misurato); § Not yet present: + «Prompt caching and the Batch API for
  the review: not used (one call every 14 days)».
- `doc/guide/revisione.md` (≤ 300 righe): aggregazione e attribuzione (D-50, D-51, D-52), la
  regola, il prompt, lo schema, la validazione, il lock, la scadenza (D-53), i costi e la durata
  misurati (data), il confronto di effort (C2), come si rilancia, come si applica da CLI,
  `hevy:upload -- --from-db`.
- `AGENTS.md`: § 6 Claude (da stub a regole, rimando a `doc/guide/revisione.md`); § 10 Commands
  (+ `review:show`, `review:compare`, `hevy:upload -- --from-db`).
- `README.md` (inglese): `## The review`, `## Applying a proposal`.
- `WORKFLOW.md` § 3, riga Review CLI: `pnpm review:run`, `pnpm review:show`, `pnpm review:compare
  -- --effort medium`, `pnpm review:apply -- <id> | --all` (04); riga Hevy CLI: `--from-db`.
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/revisione.md` (≤ 300).

## 10. Fuori ambito, dichiarato

- Route HTTP del cron e del run manuale → **06** e **05b**; UI delle proposte → **05b**.
- Notifica di fine revisione → **07** (`runReview` non notifica; chi la chiama sì).
- `dismiss` dalla UI → **05b** (qui solo repository).
- Tornare indietro di variante → D-22 → CLAUDE.md § Not yet present (riga già presente).
- Prompt caching, batch API: inutili a una chiamata ogni 14 giorni → CLAUDE.md § Not yet present
  (riga scritta in § 9).

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/04-aggregazione-revisione-e-applica.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/piano.md, doc/guide/sync.md
- Carica la skill claude-api PRIMA di scrivere il file che chiama Anthropic e usa SOLO le forme
  che la skill documenta per TypeScript (messages.parse + zodOutputFormat, output_config) — con
  due eccezioni decise dall'owner che prevalgono sulla skill: niente `fallbacks` né beta
  `server-side-fallback` (un rifiuto è `failed`), e chiamata non in streaming con `max_tokens:
  16000` e `maxRetries: 0`
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 04 INTERA
- Verifica con un grep che i contratti di 01–03 esistano con quel nome e quella firma
  (createHevyClient, HevyApiError, EnvSchema, PLAN, SCALES, findVariant, scaleOf, routineOf,
  nextVariant, variantAt, targetAt, SlotStateHistoryRow, targetText, initialSlotStates,
  ensureTemplates, buildRoutineBody, syncWorkouts, SetRow, listSetsBetween, listSlotStates,
  listSlotStateHistory, replaceSlotStateStatements, getTemplateMap, upsertTemplateMap,
  routineKeyByHevyId, listHevyRoutines, getSyncValue, createMemoryDb, createFakeHevy, romeDate,
  romeDayStartMinusDays, romeDaysBetween, parseHevyInstant)

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/04-review. Crea SESSION_NOTES.md.

Implementa la spec 04 esattamente come scritta: § 4.1 schema, migrazione e repository, § 4.2–4.6
il codice, § 6 i test (modello e Hevy finti: mai i servizi veri nei test), § 7 il collaudo fase
per fase con l'esito atteso prima — le chiamate reali a Claude (fino a due per `review:run`, una
per `review:compare`) e il PUT reale su Hevy, più l'eventuale POST permanente di un esercizio
custom, partono SOLO dopo il mio OK, con costo ed effetto detti prima; § 9 la documentazione. Se
un punto non è coperto dalla spec, fermati e chiedimelo con AskUserQuestion. Le etichette della
serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
