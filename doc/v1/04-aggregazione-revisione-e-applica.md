# 04 — Aggregazione, revisione con Claude e «Applica» da CLI

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 03.
> **Branch**: `feature/04-review` da `main`. **Dipende da**: 01, 02, 03 (schema, sync, stati, storia,
> `ensureTemplates`, `buildRoutineBody`, `time.ts`).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-2, D-3, D-4, D-6, D-7, D-14, D-15, D-18,
> D-19, D-22, D-28, D-29, D-30, D-31; scelte T-8, T-9, T-10, T-11, T-16; contratti C-7 (`reviews`,
> `proposals`, `cron_runs`), C-9, C-10, C-11, C-12, C-13 (li crea).

## 1. Il problema oggi (prove)

1. In Turso ci sono workout e set (03) e lo stato degli slot, ma nessuno li legge per esercizio né
   decide niente. La doppia progressione (D-6) è scritta nel tema, non nel codice.
2. Un set di Hevy porta solo `exercise_template_id`, `reps`, `duration_seconds`, `set_type`
   (fixture `workout.json`): per dire «quante ripetizioni ha chiuso il push-up di lunedì» servono la
   storia degli slot (03) e la mappa template → variante. Due slot possono usare la stessa variante
   (`hollow_hold` mercoledì e giovedì): il `routine_id` del workout distingue; un workout senza
   `routine_id` (come il «Push» del 2026-10-22) è ambiguo solo se la variante è condivisa.
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
| A1 | `aggregateWindow` (C-9) attribuisce ogni set non-warmup della finestra a uno slot (prima per `routine_id`, poi per variante attiva alla data del workout), raggruppa per sessione (workout) e calcola per sessione i valori (ripetizioni o secondi per set), `allAtTop` secondo il target, e per slot `sessionCount`, `hasEnoughData` (≥ 2) e `ruleAction`; i set non attribuibili finiscono in `unresolved` con il conteggio. | `aggregate.test.ts` (§ 6). Mutazione: includere i warmup → i valori cambiano → rosso. |
| A2 | `ruleAction` (C-12): `next_variant` se le ultime **due** sessioni hanno `allAtTop` e la scala ha una variante successiva; `increase` se `allAtTop` ×2 ma la scala è finita; `keep` altrimenti. `computeNextSlotState` applica `increase` (reps +2/+2, hold +10/+10, max2 +2) o `next_variant` (variante successiva con `targets[kind]`). | `progression.test.ts` con casi tabellari. Mutazione: usare una sola sessione → rosso. |
| A3 | `runReview` (C-11): sync incrementale, finestra di 28 giorni (D-15), regola dei dati scarsi (D-31), prompt § 4.4, chiamata a Claude con lo schema C-10, validazione (ogni slot eleggibile esattamente una volta, azione ammessa, `next_variant` solo se esiste), un retry su errore di validazione, salvataggio di `reviews` + `proposals`, `superseded` sulle aperte precedenti, lock (T-11), una scadenza unica (`REVIEW_DEADLINE_MS`) da cui escono timeout e retry di ogni chiamata. Esiti `completed` / `skipped` (`no_workouts`, `already_running`) / `failed` (con `error`, anche `model: timeout` e `model: refusal (<categoria>)`). | `run-review.test.ts` con DB in memoria, client Hevy finto e modello finto. Mutazioni in § 6. |
| A4 | `shouldRunScheduledReview(db, now)`: `true` se `romeDaysBetween(riferimento, now) ≥ 13`, dove il riferimento è `finished_at` dell'ultima revisione `completed`, altrimenti `plan_uploaded_at`; `false` se nessuno dei due esiste. | Test ai bordi (12 → false, 13 → true). |
| A5 | `applyProposal` (C-13): proposta `open` → nuovo stato (C-12) → `ensureTemplates` per la variante nuova se custom e mancante → `GET` routine → `buildRoutineBody` con gli stati aggiornati → `PUT` → `replaceSlotState` + `proposals.status = 'applied'` + `hevy_routines.snapshot`; `keep` → solo `applied`; proposta non `open` → `{ status: 'not_open' }`; errore Hevy → proposta resta `open`, `apply_error` salvato. `applyAllOpen` applica in sequenza e restituisce un risultato per proposta. | `apply-proposal.test.ts`. Mutazione: aggiornare lo stato prima del PUT e non rollback → il test «PUT fallisce → stato invariato» va rosso. |
| A6 | Il costo stimato (`estimated_cost_micros`) e i token di ogni revisione sono salvati; una revisione non supera **una** chiamata riuscita (+1 retry). | Test: modello finto conta le chiamate (1; 2 con un errore di validazione; 2 e `failed` con due errori). |
| A7 | `pnpm review:run` reale produce una revisione completa sui dati veri e `pnpm review:apply -- <id>` applica una proposta su Hevy (dopo l'OK). | Collaudo § 7. |

## 3. Decisioni

- **D-3** (mai automatico), **D-4/D-7** (le scale sono dati: Claude non può uscirne), **D-6**
  (doppia progressione), **D-14** (Opus 5.5, high), **D-15** (28 giorni + revisione precedente),
  **D-22** (tre esiti), **D-28** (cosa cambia «aumenta»), **D-29** (una routine alla volta), **D-30**
  (manuale sempre), **D-31** (dati scarsi in codice).
- **T-8**, **T-9** (`ruleAction` accanto a Claude), **T-10** (parametri della chiamata; **qui si
  precisa**: niente `fallbacks`: un `stop_reason: 'refusal'` o `max_tokens` rende la revisione
  `failed` con quel codice, e il rifiuto porta la sua categoria, `model: refusal (bio)`, così la
  notifica dice perché; owner, 2026-10-23), e il tempo: **una scadenza unica** per `runReview`,
  dentro i 300 s di Vercel, da cui si calcolano timeout e retry di ogni chiamata (§ 4.4 passo 5;
  owner, 2026-10-23), **T-11** (lock), **T-16** (prima revisione).
- Adottate qui, ribaltabili: (a) sessione = workout (un workout con lo stesso esercizio due volte
  conta una sessione con tutti i set); (b) `allAtTop` per `reps`: ogni set `reps ≥ max`; per
  `max2`: ogni set `reps ≥ threshold`; per `hold`: ogni set `duration_seconds ≥ maxSeconds`;
  (c) le sessioni considerate dalla regola sono le ultime due in ordine di data, anche se non
  consecutive nel calendario; (d) un set con `reps` e `duration_seconds` entrambi null è ignorato;
  (e) la motivazione di Claude è limitata a 240 caratteri e (f) il `summary` a 600: lo chiede il
  prompt e lo controlla `validateReviewResponse` (errore → retry con l'istruzione), non lo zod
  dell'output, perché l'output strutturato non applica `minLength`/`maxLength` (l'SDK li toglie
  dallo schema inviato e li verifica lato client, dove un fallimento non arriva al retry); (g) il
  prompt è in italiano e chiede motivazioni in italiano; (h) la finestra è `[romeDayStartMinusDays(now,
  28), now)`; (i) `reviews.prompt_json` conserva l'input inviato (ripetibilità), `raw_response_json`
  la risposta; (j) il modello e i prezzi stanno in `src/lib/review/pricing.ts` (`MODEL = 'claude-opus-5-5'`,
  `USD_PER_MTOK = { input: 4, output: 20 }`).

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
  status: text('status').notNull(), // 'open' | 'applied' | 'dismissed' | 'superseded'
  appliedAt: integer('applied_at'),
  applyError: text('apply_error'),
  insufficientData: integer('insufficient_data', { mode: 'boolean' }).notNull(),
}, (t) => [index('proposals_review_idx').on(t.reviewId), index('proposals_status_idx').on(t.status)]);

export const cronRuns = sqliteTable('cron_runs', {
  id: text('id').primaryKey(),
  startedAt: integer('started_at').notNull(),
  finishedAt: integer('finished_at'),
  outcome: text('outcome').notNull(), // 'review_run' | 'review_not_due' | 'sync_only' | 'error'
  detailsJson: text('details_json').notNull(),
});
```

`nanoid@6.0.2` si aggiunge qui (`pnpm add nanoid@6.0.2 @anthropic-ai/sdk@1.1.0`).

### 4.2 Aggregazione — `src/lib/review/aggregate.ts` (C-9)

```ts
export interface SetValue { setIndex: number; value: number }               // reps or seconds
export interface SessionSummary { workoutId: string; date: string; startTime: number; values: SetValue[]; allAtTop: boolean }
export interface ExerciseWindowSummary {
  slotId: SlotId; routineKey: RoutineKey; variantId: VariantId; target: Target;
  sessions: SessionSummary[]; sessionCount: number; hasEnoughData: boolean; ruleAction: ProposalAction;
}
export interface AggregateInput {
  sets: SetRow[];                       // listSetsBetween(): includes workout.startTime and workout.routineId
  states: SlotState[];                  // current
  history: SlotStateHistoryRow[];       // all rows
  templateMap: TemplateIdMap;           // variantId → hevy template id
  routineKeys: Map<string, RoutineKey>; // hevy routine id → key
  window: { fromMs: number; toMs: number };
}
export interface AggregateResult { summaries: ExerciseWindowSummary[]; unresolved: { count: number; byTemplate: Record<string, number> } }

export function aggregateWindow(input: AggregateInput): AggregateResult
```

Attribuzione di un set (pura, `resolveSlot`): `warmup` → scartato. Candidati = slot `reviewed` la
cui variante **attiva alla data del workout** (`variantAt(history, slotId, startTime)`) ha
`templateMap[variantId] === set.exerciseTemplateId`. Se `routineKeys.get(workout.routineId)` esiste
→ candidati filtrati su quella routine. Un candidato → attribuito; zero o più di uno → `unresolved`.
Valore del set: `reps` per target `reps`/`max2`, `duration_seconds` per `hold`; `null` → scartato.
`allAtTop` come in § 3 (b). `ruleAction` = `ruleAction(summary, state, scale)`.

### 4.3 Progressione — `src/lib/plan/progression.ts` (C-12)

```ts
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
export interface ReviewModelResult { output: ReviewResponse | null; stopReason: string; refusalCategory: string | null; inputTokens: number; outputTokens: number; raw: unknown }
export interface ReviewCallOptions { timeoutMs: number; maxRetries: number }
export interface ReviewModel { complete(system: string, user: string, options: ReviewCallOptions): Promise<ReviewModelResult> }
export type ReviewEffort = 'medium' | 'high';

export function createAnthropicReviewModel(apiKey: string, { effort = 'high' }: { effort?: ReviewEffort } = {}): ReviewModel {
  const client = new Anthropic({ apiKey });
  return {
    async complete(system, user, { timeoutMs, maxRetries }) {
      try {
        // timeout and retries per request: runReview computes them from what is left of its deadline.
        const response = await client.messages.parse({
          model: MODEL,
          max_tokens: 16000, // thinking is always on and counts toward it: ~3k of JSON plus the thinking at effort high
          system,
          messages: [{ role: 'user', content: user }],
          output_config: { effort, format: zodOutputFormat(ReviewResponseSchema) },
        }, { timeout: timeoutMs, maxRetries });
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

Due verifiche all'inizio della sessione, contro l'SDK installato: il nome della classe d'errore del timeout (`Anthropic.APIConnectionTimeoutError` nell'SDK TypeScript) e il campo `stop_details.category` sul tipo della risposta. Se l'SDK non lo tipizza, lo si legge dalla risposta grezza con un cast stretto, commentato.

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
> codice: un suggerimento, non un vincolo; `values` sono le ripetizioni di ogni serie (i secondi per
> le tenute); `previousReview` dice che cosa è stato proposto l'ultima volta e se è stato applicato;
> gli slot di `skippedForInsufficientData` sono solo contesto e non vanno in `proposals`.

Lo **user prompt** è JSON (stringificato con indentazione 0) con: `window { from, to }`,
`exercises[]` per ogni slot eleggibile (`slotId`, `routine`, `exercise` (titolo variante),
`scalePosition` `"3/6"`, `nextVariant` (titolo o null), `target` (testo di `targetText`),
`sessions[] { date, values }`, `ruleAction`), `previousReview` (null o `{ date, proposals[] {
slotId, action, applied: boolean } }`), `skippedForInsufficientData[]` (slot id con 0–1 sessioni,
per contesto: **non** da proporre). Niente dati personali: nessun nome, nessun id Hevy.

`run-review.ts` (C-11):

```ts
export const REVIEW_DEADLINE_MS = 280_000; // 300 s of Vercel's maxDuration, minus the save and the notification
export const MIN_MODEL_CALL_MS = 60_000;   // below this a call is not started: it would be killed mid-way
export async function runReview({ db, hevy, model, now, clock, trigger }: { db: Db; hevy: HevyClient; model: ReviewModel; now: number; clock: () => number; trigger: ReviewTrigger }): Promise<ReviewRun>
```

`now` è l'istante d'inizio della richiesta (il cron e la route lo prendono prima del loro sync), e
la **scadenza unica** della revisione è `deadline = now + REVIEW_DEADLINE_MS`. `clock` dice quanto
tempo è passato: le route e la CLI passano `Date.now`, i test un orologio finto. Non è una
lettura dell'orologio in un modulo puro: `run-review.ts` è un caso d'uso.

Passi, nell'ordine (ognuno con il suo test):
1. **Lock**: se esiste una `reviews` con `status='running'` e `started_at > now − 10 min` →
   `skipped/already_running` (nessuna riga nuova). Altrimenti inserisce la riga `running`.
2. **Sync**: `syncWorkouts({ db, hevy, now })` (incrementale; un errore Hevy qui → `failed` con
   `error = 'sync: ' + message`).
3. **Finestra e aggregazione**: `window = { fromMs: romeDayStartMinusDays(now, 28), toMs: now }`;
   `listSetsBetween`; se **nessun workout** nella finestra → `skipped/no_workouts`, `finished_at`.
4. **Dati scarsi (D-31)**: `eligible = summaries.filter(hasEnoughData)`; gli altri generano
   proposte `keep` con `rationale = 'Dati insufficienti: meno di 2 sessioni nelle ultime 4
   settimane.'`, `insufficient_data = true`, senza passare da Claude. Se `eligible` è vuoto → tutte
   le proposte sono «dati insufficienti», `completed` senza chiamata al modello.
5. **Chiamata**, dentro la scadenza: `remaining = deadline − clock()`; se `remaining <
   MIN_MODEL_CALL_MS` → `failed` (`error = 'model: timeout'`), senza chiamata. Altrimenti
   `maxRetries = remaining ≥ 2 × MIN_MODEL_CALL_MS ? 1 : 0` e `timeoutMs = floor(remaining / (1 +
   maxRetries))`, così nemmeno i retry di rete dell'SDK superano la scadenza; poi
   `model.complete(system, user, { timeoutMs, maxRetries })`. `stopReason: 'timeout'` → `failed`
   (`error = 'model: timeout'`); `refusal` → `failed` (`error = 'model: refusal (' +
   refusalCategory + ')'`, oppure `'model: refusal'` senza categoria); un altro `stopReason` ≠
   `end_turn` o `output` null → `failed` (`error = 'model: ' + stopReason`); un'eccezione del
   modello → `failed` (`error = 'model: ' + message`), mai una riga lasciata `running`. Altrimenti
   **validazione** (`validateReviewResponse(output,
   eligible)`: insieme degli `slotId` uguale, azioni ammesse, `next_variant` solo con successore,
   `rationale` di 1–240 caratteri, `summary` ≤ 600);
   su errore di validazione **un** secondo tentativo con lo user prompt più la riga
   `"validationErrors": [...]`, con le stesse regole di tempo, **solo** se `deadline − clock() ≥
   MIN_MODEL_CALL_MS`; senza margine, o al secondo errore → `failed` (`error = 'validation: ...'`).
6. **Salvataggio** in un batch: `proposals` (una per slot eleggibile + quelle di dati scarsi) con
   `from/to` calcolati da `computeNextSlotState`, `status='open'`; `UPDATE proposals SET
   status='superseded' WHERE status='open' AND review_id != this`; `reviews` → `completed`, token,
   costo (`estimated_cost_micros = inputTokens * 4 + outputTokens * 20`: token × $/MTok =
   micro-USD), `summary`, `prompt_json`, `raw_response_json`, `finished_at`.
7. Ritorna `ReviewRun`.

`shouldRunScheduledReview(db, now)` come in A4.

### 4.5 «Applica» — `src/lib/apply/apply-proposal.ts` (C-13)

```ts
export type ApplyResult = { proposalId: string; status: 'applied' | 'not_open' | 'failed'; error?: string; hevyRoutineId?: string }
export async function applyProposal({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, proposalId: string): Promise<ApplyResult>
export async function applyAllOpen(deps, reviewId?: string): Promise<ApplyResult[]>
```

Ordine in `applyProposal`: leggi proposta (`open` o `not_open`); `keep` → `applied` subito, senza
Hevy; altrimenti: `states = listSlotStates`; `nextState = computeNextSlotState(state, action, scale,
now)`; se `templateMap[nextState.variantId]` manca → `ensureTemplates(hevy, [variant],
listExerciseTemplates())` e `upsertTemplateMap`; `routineRow = hevy_routines[slot.routineKey]`;
`current = hevy.getRoutine(routineRow.hevyRoutineId)` (404 → `failed`, `apply_error`); `body =
buildRoutineBody(planRoutine, statesWith(nextState), templateMap, current.folder_id)`; `saved =
hevy.updateRoutine(id, body)`; **solo dopo il PUT riuscito**, in un batch: `replaceSlotState`,
`proposals.status='applied', applied_at`, `hevy_routines.snapshot_json = saved`. Un errore in
qualunque punto prima del batch lascia DB e proposta invariati, salva `apply_error` e ritorna
`failed`.

`dismissProposal(db, id)` → `status='dismissed'` (la 05b la usa; qui solo repository + test).

### 4.6 CLI — `scripts/`

- `review-run.ts` (`pnpm review:run`): `runReview` con `trigger: 'cli'`, stampa stato, costo, le
  proposte in tabella (slot, variante, azione, regola, motivazione).
- `review-apply.ts` (`pnpm review:apply -- <proposalId> | --all`): applica e stampa i risultati.
  Prima di ogni PUT stampa il diff degli esercizi che cambiano (titolo, da → a) e chiede conferma
  con `readline` (`y/N`), perché D-3 vale anche da CLI.
- `review-show.ts` (`pnpm review:show`): l'ultima revisione con le proposte e il loro stato.
- `review-compare.ts` (`pnpm review:compare -- --effort medium`): rimanda il `prompt_json`
  dell'ultima revisione `completed` a `createAnthropicReviewModel(key, { effort })` e stampa,
  accanto a quelli salvati, validazione, proposte che cambiano azione, token, costo e durata.
  **Non scrive nulla** (nessuna riga, nessuna proposta). Serve al collaudo C2 per decidere l'effort
  con un dato.

`EnvSchema` aggiunge `ANTHROPIC_API_KEY: z.string().min(1)`; `.env.example` idem (chiave della
Console personale, D-2).

## 5. Vincoli e trappole

- **AGENTS.md § Claude** (nasce qui) — qui vuol dire che: `@anthropic-ai/sdk` si importa in un solo
  file (`model.ts`); il modello e i prezzi stanno in `pricing.ts`; `max_tokens` esplicito; `effort`
  esplicito (il default di Opus 5.5 è `medium`); niente prefill, niente `tool_choice`; la risposta
  si legge da `parsed_output` e si rivalida comunque (`validateReviewResponse`): lo schema
  garantisce la forma, non i vincoli di dominio.
- **AGENTS.md § Database** — tutte le scritture di una revisione o di un'applicazione sono in un
  `batch`; mai due batch dove uno basta.
- **D-3** — nessun percorso applica una proposta senza una chiamata esplicita a `applyProposal`
  (grep nel test: `runReview` non importa `apply-proposal`).
- Il retry della validazione **rimanda tutto il prompt** più gli errori: non è una conversazione
  (niente storia, niente cache).
- `Date.now()` non si chiama nei moduli puri: `now` è sempre un parametro.
- Dati personali: il prompt non contiene nomi, id Hevy o date di nascita; i titoli delle varianti
  sono quelli del piano.

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Unit | `aggregate.test.ts`: fixture inline (righe `SetRow` inventate: 3 workout del `mon_upper_a` con push-up 4×[14,15,15,15] / [15,15,15,15] / [16,16,15,15], rematore 8–12 con [12,12,12,11]…): attribuzione per `routine_id`; `hollow_hold` condiviso → attribuito per routine; workout senza `routine_id` e variante univoca → attribuito; variante condivisa senza routine → `unresolved`; warmup esclusi; set con valori null esclusi; storia: un set datato prima di un cambio variante va alla variante vecchia; `hasEnoughData` false con 1 sessione. | Togliere il filtro sui warmup; usare lo stato corrente invece della storia. |
| Unit | `progression.test.ts`: `ruleAction` (0/1/2 sessioni, allAtTop misto, scala finita → `increase`); `increaseTarget` per i tre generi; `computeNextSlotState` `next_variant` su `push_up` → `decline_push_up` con `max2 12`; su `wall_sit_one_leg` (ultima) → throw. | Soglia a una sessione → rosso. |
| Unit | `validate.test.ts`: slot mancante, slot in più, slot doppio, azione sconosciuta, `next_variant` senza successore, motivazione vuota o di 241 caratteri, `summary` di 601 → errori con il `slotId`. | Togliere il controllo delle lunghezze → il caso dei 241 caratteri va rosso. |
| Unit + DB | `run-review.test.ts`: (1) lock: riga `running` fresca → `already_running`, nessuna chiamata; (2) `no_workouts`; (3) percorso completo con modello finto che risponde bene → `completed`, N proposte, costo = `in×4 + out×20`, precedenti `superseded`; (4) risposta con slot mancante poi buona → 2 chiamate, `completed`; (5) due risposte cattive → `failed`, 2 chiamate; (6) `stop_reason: 'refusal'` con categoria `bio` → `failed`, `error = 'model: refusal (bio)'`; (7) slot con 1 sessione → proposta `keep` con `insufficient_data`, non inviato al modello (il prompt catturato non lo contiene in `exercises`); (8) orologio finto già oltre `deadline − MIN_MODEL_CALL_MS` → `failed`, `model: timeout`, nessuna chiamata; (9) prima risposta non valida e orologio che, dopo la chiamata, lascia meno di `MIN_MODEL_CALL_MS` → `failed` con `validation:`, 1 chiamata sola; (10) `timeoutMs` e `maxRetries` passati al modello calcolati dal tempo che resta (con 250 s → 1 retry e 125 s; con 100 s → 0 retry e 100 s); (11) il modello lancia → `failed`, la riga non resta `running`. | Non marcare `superseded` → (3) rosso; retry infinito → (5) rosso; ignorare la scadenza prima del retry → (9) rosso. |
| Unit + DB | `should-run.test.ts`: nessun riferimento → false; `plan_uploaded_at` 12 giorni fa → false; 13 → true; ultima `completed` 13 giorni fa → true; ultima `failed` 13 giorni fa ma `completed` 5 giorni fa → false. | Usare `started_at` di qualunque stato → l'ultimo caso va rosso. |
| Unit + DB | `apply-proposal.test.ts`: `keep` → `applied` senza chiamate Hevy; `next_variant` su custom mancante → 1 `createCustomExercise` + GET + PUT, stato e storia aggiornati, `hevy_routines.snapshot` nuovo; PUT che fallisce (500×2) → `failed`, stato invariato, `apply_error`; proposta `applied` → `not_open`; `applyAllOpen` con 3 aperte (keep, increase, next) → 3 risultati nell'ordine. | Scrivere lo stato prima del PUT → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: cancelli verdi; `pnpm hevy:sync` ancora incrementale.
- **C — Comportamento nuovo (senza effetti esterni oltre a Claude)**: `pnpm review:run`. **Una
  chiamata reale ad Anthropic** (costo atteso < 0,40 $, letto dalla riga `estimated_cost` stampata);
  **sole GET** verso Hevy. Esito atteso: `completed`; una proposta per slot in revisione (27), molte
  «dati insufficienti» se il piano è iniziato da meno di due settimane; le altre con motivazione in
  italiano ≤ 240 caratteri; `ruleAction` e `action` concordi dove i dati sono chiari. Se è
  `failed`, l'errore è leggibile e la riga `reviews` lo conserva.
- **C2 — Effort `medium` a confronto (dopo l'OK, ≈ 0,20 $)**: `pnpm review:compare -- --effort
  medium` sullo stesso input di C. **Una chiamata reale ad Anthropic**, nessuna scrittura. Esito
  atteso: risposta valida; si leggono, accanto a quelli di C, le proposte che cambiano azione, i
  token di output, il costo e la durata. Il confronto si annota in `doc/guide/revisione.md` con la
  data. L'effort resta `high` (D-14) finché l'owner non decide altrimenti su quel dato.
- **D — Scrittura reale su Hevy (dopo l'OK)**: `pnpm review:apply -- <id di una proposta
  'increase' o 'next_variant'>`; conferma `y`. Esito atteso: 1 GET + 1 PUT; su Hevy la routine
  mostra il nuovo range/variante; `slot_state` e `slot_state_history` aggiornati (la vecchia riga
  con `valid_to`); `proposals.status = 'applied'`. Se non c'è nessuna proposta diversa da `keep`
  (piano giovane), si applica una proposta `keep` (nessuna chiamata Hevy) e si **simula** un
  `increase` con uno script usa-e-getta che inserisce una proposta `open` inventata su `mon_row`
  (8–12 → 10–14), si applica, si verifica su Hevy, poi si **riapplica** lo stato di partenza con
  un secondo script (PUT con `initialSlotStates`) e si cancellano le righe inventate.
- **F — Giro guidato (owner, app Hevy)**: la routine toccata mostra il cambiamento (1 cosa).
- **G — Ripristino**: script usa-e-getta cancellati; se è stato simulato, Hevy e DB riportati allo
  stato di partenza; `reviews` della prova resta (è storia vera).

Che cosa NON copre: cron, UI, notifiche.

## 8. Messa in esercizio

- Env: `ANTHROPIC_API_KEY` (Console personale). DDL: `drizzle/0001_reviews.sql` via `pnpm db:migrate`.
- Misura di chiusura: `pnpm review:show` mostra una revisione `completed` sui dati veri.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Key Features «Revisione» e «Applica» con puntatori; § Data & Integrations
  (Anthropic: modello e costo).
- `doc/guide/revisione.md` (≤ 300 righe): aggregazione e attribuzione, la regola, il prompt, lo
  schema, la validazione, il lock, i costi misurati (data), come si rilancia, come si applica da CLI.
- `AGENTS.md`: § Claude (stub + regole), § Review (stub → guida), § Commands.
- `README.md`: § La revisione, § Applicare.

## 10. Fuori ambito, dichiarato

- Route HTTP del cron e del run manuale → **06** e **05b**; UI delle proposte → **05b**.
- Notifica di fine revisione → **07** (`runReview` non notifica; chi la chiama sì).
- `dismiss` dalla UI → **05b** (qui solo repository).
- Tornare indietro di variante → D-22 → CLAUDE.md § Not yet present.
- Prompt caching, batch API: inutili a una chiamata ogni 14 giorni → CLAUDE.md § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L**.

```
Ciao, in questa sessione implemento la spec doc/v1/04-aggregazione-revisione-e-applica.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/piano.md, doc/guide/sync.md
- Carica la skill claude-api PRIMA di scrivere il file che chiama Anthropic e usa SOLO le forme
  che la skill documenta per TypeScript (messages.parse + zodOutputFormat, output_config)
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 04 INTERA
- Verifica con un grep che i contratti di 01–03 esistano con quel nome e quella firma
  (createHevyClient, PLAN, SCALES, nextVariant, variantAt, ensureTemplates, buildRoutineBody,
  syncWorkouts, listSetsBetween, listSlotStates, replaceSlotState, romeDayStartMinusDays,
  romeDaysBetween)

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/04-review. Crea SESSION_NOTES.md.

Implementa la spec 04 esattamente come scritta: § 4.1 schema e migrazione, § 4.2–4.6 il
codice, § 6 i test (modello e Hevy finti: mai i servizi veri nei test), § 7 il collaudo fase
per fase con l'esito atteso prima — la chiamata reale a Claude e il PUT reale su Hevy partono
SOLO dopo il mio OK, con il costo stimato detto prima; § 9 la documentazione. Se un punto non
è coperto dalla spec, fermati e chiedimelo con AskUserQuestion. Le etichette della serie non
entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
