# 03 — Turso: schema, migrazioni e sincronizzazione degli allenamenti

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 33 rilievi, 3 decisioni
> (D-42, D-48, D-49). Pronta per l'implementazione dopo la 02.
> **Branch**: `feature/03-turso-sync` da `main`. **Dipende da**: 01 (client, schemi, `fakeFetch`),
> 02 (`PLAN`, `FOLDER_TITLE`, `initialSlotStates(now)`, `allVariantsInUse`, `resolveTemplates`,
> `createFakeHevy`, routine già su Hevy).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-32, D-33, D-36, D-37, D-42, D-48,
> D-49; scelte T-3, T-7, T-8, T-15, T-16; contratti C-7 (prime sette tabelle e repository), C-8,
> C-17 e la parte `history.ts` di C-3 (li crea); usa C-1, C-2, C-3, C-5, C-6, C-15.

## 1. Il problema oggi (prove)

1. Non c'è persistenza: la 02 porta le routine su Hevy e stampa gli id, ma nessuno li conserva; lo
   stato degli slot (variante e target correnti) esiste solo come `initialSlotStates(now)` in
   memoria.
2. Gli allenamenti vivono su Hevy: 174 workout su 18 pagine da 10 (`GET /v1/workouts/count` reale,
   2026-10-23). Scaricarli tutti a ogni revisione costa 18 chiamate; `GET /v1/workouts/events?since`
   restituisce solo `updated`/`deleted` dopo un istante (fixture `workout_events.json`), ordinati dal
   più recente, con `page_count`.
3. Le date di Hevy sono stringhe ISO con offset (`start_time: "2026-10-22T17:31:47+00:00"`) o con
   `Z` e millisecondi (`updated_at: "2026-10-22T17:52:49.853Z"`); l'aggregazione (04) ragiona per
   giorno di calendario in Europe/Rome (T-15).
4. Per sapere **quale variante era attiva** in una data serve la storia degli stati degli slot (T-8):
   dopo un «Applica» il template dell'esercizio cambia, e i set vecchi devono restare attribuiti
   alla variante vecchia.
5. Turso non ha Francoforte (T-1) e i log di Vercel durano 1 ora: l'ultima corsa di sync riuscita
   lascia `last_sync_at` e `last_sync_mode` in `sync_state`; il registro di ogni corsa del cron,
   anche fallita, è `cron_runs` (04 crea la tabella, 06 la scrive).

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | Lo schema Drizzle (C-7, § 4.2) genera la migrazione `drizzle/0000_*.sql` con le sette tabelle e gli indici di § 4.2; `pnpm db:migrate` la applica a `file:local.db`; i test la applicano a `:memory:`; schema e migrazioni non derivano. | `src/lib/db/migrate.test.ts`: `createMemoryDb()` poi `SELECT name FROM sqlite_master` contiene le 7 tabelle e i 6 indici; `PRAGMA foreign_keys` = 1. Mutazione: togliere `hevy_templates_hevy_id_idx` dallo schema **e lanciare `pnpm db:generate`** (nasce una migrazione con `DROP INDEX`) → rosso. Il collaudo A prova che `pnpm db:generate` a schema invariato non crea file. |
| A2 | `syncWorkouts` (C-8) in modalità **full** scarica tutte le pagine, inserisce workout e set e pota i workout locali assenti su Hevy (D-49); in modalità **incremental** legge tutti gli eventi dal cursore, tiene per workout il più recente, li applica dal più vecchio (`updated` → upsert, `deleted` → delete) e avanza il cursore solo alla fine; è idempotente: rilanciata senza eventi nuovi, `upserted = 0` e `deleted = 0` (si aggiornano solo `last_sync_at` e `last_sync_mode`). | `sync-workouts.test.ts` con `createHevyClient` + `fakeFetch`: 3 pagine → 25 workout; eventi con 1 updated + 1 deleted (su un workout prima inserito) → `upserted 1`, `deleted 1`, cursore `'2026-10-22T17:52:49.853Z'`; rilancio → 0 e 0; `[deleted X 10:00, updated X 09:00]` → X assente. Mutazione: non cancellare sui `deleted` → rosso; applicare gli eventi nell'ordine ricevuto → rosso. |
| A3 | `syncOneWorkout` (C-8) fa `getWorkout(id)` e l'upsert; un 404 (`HevyApiError`) → `{ status: 'not_found' }` senza eccezione e, se il workout esiste in locale, lo cancella (D-48). | Test con `fakeFetch`: 404 su id ignoto → `not_found`, nessuna riga; 404 su id presente → `not_found`, workout e set cancellati. Mutazione: propagare il 404 → rosso; non cancellare → rosso. |
| A4 | `syncRoutines` è di **sola lettura** su Hevy (T-7): registra in `hevy_routines` le 5 routine della cartella `FOLDER_TITLE` (per titolo → `RoutineKey`), in `hevy_templates` la mappa variante → id per tutte le varianti in uso (risolte con `resolveTemplates`; una variante senza template → errore), inserisce in `slot_state` e `slot_state_history` gli slot che mancano con `validFrom = plan_uploaded_at`, e scrive `sync_state.plan_uploaded_at` = min(`created_at`) delle 5 routine, una volta sola. | `sync-routines.test.ts` con `createFakeHevy`: routine finte nella cartella → 5 righe, 29 righe in `hevy_templates`, 30 `slot_state`, 30 righe di storia; rilancio → invariato, `plan_uploaded_at` uguale; cartella mancante → errore che contiene `pnpm hevy:upload`; custom in uso assente → errore che contiene l'id della variante e `writes.length === 0`. Mutazione: sovrascrivere `plan_uploaded_at` → rosso; chiamare `ensureTemplates` → `writes` non vuoto → rosso. |
| A5 | `workout_sets` è denormalizzata e completa: un set per riga con `exercise_template_id`, `set_type`, `reps`, `duration_seconds`; i `warmup` restano in tabella (li esclude la 04); un workout senza set si salva con 0 righe. | Test: il workout della fixture → 14 righe di set (5+3+3+3); `exercises: []` → 1 workout, 0 set, nessun errore. Mutazione: saltare i warmup → 12 righe → rosso; insert dei set sempre presente → errore sul workout vuoto → rosso. |
| A6 | `pnpm hevy:sync` reale popola `local.db` con tutti i workout e le 5 routine; rilanciato, fa le GET di `syncRoutines` (cartelle, routine, 5 pagine di template) e la GET degli eventi, nessuna GET di `/v1/workouts`. | Collaudo § 7 (sole GET; `syncRoutines` non scrive mai su Hevy). |
| A7 | Gli helper di tempo (C-17) convertono le date Hevy e contano i giorni di calendario in Europe/Rome anche a cavallo del cambio d'ora. | `time.test.ts` con casi fissi (§ 6). |

## 3. Decisioni

- **D-1** (Turso), **D-32** (webhook: qui nasce `syncOneWorkout`; la route è della 06), **D-33**
  (tutti i workout, sync incrementale via eventi; gli allenamenti pre-piano contano grazie a D-42),
  **D-36** (`db/client.ts` e lo script leggono le env con `EnvSchema.pick(...)`), **D-42** (prima
  riga di storia valida «da sempre»), **D-48** (404 su workout locale → cancellato), **D-49** (il
  full sync pota).
- **T-3** (`@libsql/client` Node, singleton, WAL su file), **T-7** (`syncRoutines` è di sola
  lettura: usa `resolveTemplates`, mai `ensureTemplates`), **T-8** (storia degli stati), **T-15**
  (Europe/Rome; epoch ms in DB), **T-16** (`plan_uploaded_at` ricavata da Hevy).
- Adottate qui, ribaltabili: (a) `raw_json` del workout conservato in `workouts` (serve a
  rigenerare i set se la 04 cambia l'aggregazione, costa pochi KB); (b) il cursore degli eventi è
  il massimo, calcolato su `parseHevyInstant(...)` (compreso `since`), fra gli `updated_at` e i
  `deleted_at` visti, salvato come `new Date(maxMs).toISOString()` e rimandato tale e quale in
  `since`; Hevy non documenta se `since` è inclusivo, quindi gli eventi con istante **≤ `since`** si
  scartano prima di applicarli (così un rilancio senza novità dà 0 e 0); (c) **nessuna transazione
  interattiva**, né nel codice né nei test: ogni scrittura composta è un `db.batch([...])`; su
  `:memory:` @libsql/client, dopo una `db.transaction()` fallita, riapre una connessione vuota
  (verificato in `../segnaprezzi/src/lib/db/testing/create-test-db.ts`), e `batch` non ha il
  problema; (d) `slot_state_history.valid_to` nullo = corrente; (e) le migrazioni si generano con
  drizzle-kit e si committano in `drizzle/`; in produzione le applica `pnpm db:migrate:remote` dal
  PC dell'owner (06), mai un hook di build; (f) `sync_state` è una tabella chiave/valore testuale
  con i formati di § 4.2; (g) le cancellazioni sono sempre **esplicite** (set poi workout, nello
  stesso batch): il `PRAGMA foreign_keys` vale per connessione e sul Turso remoto via HTTP non
  sopravvive fra le richieste, quindi la cascade dello schema è solo una rete in locale; (h) i set
  `failure` e `dropset` si salvano con il loro `set_type` (li filtra la 04); `distance_meters` e
  `custom_metric` restano solo in `raw_json`; (i) `exerciseIndex` e `setIndex` sono i campi `index`
  di Hevy; (j) `hevy_routines.snapshot_json` è la `HevyRoutine` intera di `listRoutines()`.

## 4. La soluzione

### 4.1 Dipendenze e configurazione

```bash
pnpm add @libsql/client@0.18.0 drizzle-orm@0.45.4
pnpm add -D drizzle-kit@0.31.11 dotenv@<versione esatta di oggi, 17.x o 18.x>
```

(La versione di `dotenv` si legge con `npm view dotenv version` e si annota in T-2.) `.gitignore`
aggiunge `local.db`, `local.db-wal`, `local.db-shm`.

`drizzle.config.ts` (ricalca `../segnaprezzi/drizzle.config.ts`, repository di riferimento in sola
lettura accanto a questo, con uno schema solo):

```ts
import { config } from 'dotenv';
import { defineConfig } from 'drizzle-kit';

// drizzle-kit runs outside Next.js, which is what normally loads .env.local.
config({ path: '.env.local' });

export default defineConfig({
  dialect: 'turso',
  schema: './src/lib/db/schema.ts',
  out: './drizzle',
  dbCredentials: {
    url: process.env.TURSO_DATABASE_URL ?? 'file:local.db',
    authToken: process.env.TURSO_AUTH_TOKEN || undefined,
  },
});
```

Script: `"db:generate": "drizzle-kit generate"`, `"db:migrate": "drizzle-kit migrate"`,
`"db:studio": "drizzle-kit studio"`, `"hevy:sync": "tsx --env-file-if-exists=.env.local scripts/hevy-sync.ts"`.
`.env.example` aggiunge `TURSO_DATABASE_URL="file:local.db"` e `TURSO_AUTH_TOKEN=""`; `EnvSchema`
aggiunge `TURSO_DATABASE_URL: z.string().min(1)`, `TURSO_AUTH_TOKEN: z.string().optional()`.

`src/lib/db/client.ts` ricalca `../segnaprezzi/src/lib/db/client.ts` (singleton su `globalThis`,
`import * as schema from './schema'`, `export type Db = typeof db`) con due differenze. La prima:
legge le env con `EnvSchema.pick({ TURSO_DATABASE_URL: true, TURSO_AUTH_TOKEN: true }).parse(process.env)`
da `@/lib/env-schema`, **mai** da `@/lib/env` (D-36): lo importano anche gli script `tsx`, e
`server-only` fuori da Next lancia. La seconda: i pragma stanno in una funzione esportata, così il
DB dei test prova il codice di produzione:

```ts
/** WAL and busy_timeout only make sense on a local file; foreign keys are per connection (see § 3 g). */
export async function applyConnectionPragmas(client: Client, url: string): Promise<void> {
  if (url.startsWith('file:')) {
    await client.execute('PRAGMA journal_mode = WAL');
    await client.execute('PRAGMA busy_timeout = 5000');
  }
  await client.execute('PRAGMA foreign_keys = ON');
}
```

Il singleton la chiama con `void applyConnectionPragmas(client, url)`. Per i test:
`src/lib/db/testing/memory-db.ts` → `createMemoryDb(): Promise<Db>` che fa
`createClient({ url: ':memory:' })`, `await applyConnectionPragmas(client, ':memory:')`,
`drizzle(client, { schema })` e `migrate(db, { migrationsFolder: 'drizzle' })`
(`drizzle-orm/libsql/migrator`); un client nuovo per test, mai cacheato su `globalThis`.

### 4.2 Schema — `src/lib/db/schema.ts` (C-7)

```ts
import { index, integer, real, sqliteTable, text, uniqueIndex } from 'drizzle-orm/sqlite-core';

/** One row per Hevy workout; raw_json keeps the full payload so sets can be rebuilt. Times are epoch ms UTC. */
export const workouts = sqliteTable('workouts', {
  id: text('id').primaryKey(),
  title: text('title').notNull(),
  routineId: text('routine_id'),
  startTime: integer('start_time').notNull(),
  endTime: integer('end_time').notNull(),
  hevyUpdatedAt: integer('hevy_updated_at').notNull(),
  rawJson: text('raw_json').notNull(),
  syncedAt: integer('synced_at').notNull(),
}, (t) => [index('workouts_start_time_idx').on(t.startTime)]);

/** Denormalised sets: the aggregation reads this table only. Warm-up sets are stored and filtered later. */
export const workoutSets = sqliteTable('workout_sets', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  workoutId: text('workout_id').notNull().references(() => workouts.id, { onDelete: 'cascade' }),
  exerciseIndex: integer('exercise_index').notNull(),
  setIndex: integer('set_index').notNull(),
  exerciseTemplateId: text('exercise_template_id').notNull(),
  exerciseTitle: text('exercise_title').notNull(),
  setType: text('set_type').notNull(),
  reps: integer('reps'),
  durationSeconds: integer('duration_seconds'),
  weightKg: real('weight_kg'),
  rpe: real('rpe'),
}, (t) => [
  index('workout_sets_workout_idx').on(t.workoutId),
  index('workout_sets_template_idx').on(t.exerciseTemplateId),
]);

/**
 * Key/value bookkeeping. Values: workouts_cursor = ISO UTC (toISOString()); plan_uploaded_at,
 * last_sync_at, last_full_sync_at = epoch ms in decimal (String(ms), read with Number(value));
 * last_sync_mode = 'full' | 'incremental', written together with last_sync_at.
 */
export const syncState = sqliteTable('sync_state', {
  key: text('key').primaryKey(),
  value: text('value').notNull(),
  updatedAt: integer('updated_at').notNull(),
});

export const hevyRoutines = sqliteTable('hevy_routines', {
  routineKey: text('routine_key').primaryKey(),
  hevyRoutineId: text('hevy_routine_id').notNull(),
  folderId: integer('folder_id'),
  title: text('title').notNull(),
  hevyCreatedAt: integer('hevy_created_at').notNull(),
  hevyUpdatedAt: integer('hevy_updated_at').notNull(),
  snapshotJson: text('snapshot_json').notNull(),
  syncedAt: integer('synced_at').notNull(),
}, (t) => [uniqueIndex('hevy_routines_hevy_id_idx').on(t.hevyRoutineId)]);

export const hevyTemplates = sqliteTable('hevy_templates', {
  variantId: text('variant_id').primaryKey(),
  hevyTemplateId: text('hevy_template_id').notNull(),
  title: text('title').notNull(),
  isCustom: integer('is_custom', { mode: 'boolean' }).notNull(),
  syncedAt: integer('synced_at').notNull(),
}, (t) => [index('hevy_templates_hevy_id_idx').on(t.hevyTemplateId)]);

export const slotState = sqliteTable('slot_state', {
  slotId: text('slot_id').primaryKey(),
  variantId: text('variant_id').notNull(),
  targetJson: text('target_json').notNull(),
  updatedAt: integer('updated_at').notNull(),
  source: text('source').notNull(), // 'plan' | 'apply'
});

/** Which variant/target a slot had between valid_from and valid_to (null = current). */
export const slotStateHistory = sqliteTable('slot_state_history', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  slotId: text('slot_id').notNull(),
  variantId: text('variant_id').notNull(),
  targetJson: text('target_json').notNull(),
  validFrom: integer('valid_from').notNull(),
  validTo: integer('valid_to'),
  source: text('source').notNull(),
  proposalId: text('proposal_id'),
}, (t) => [index('slot_state_history_slot_idx').on(t.slotId, t.validFrom)]);
```

DDL atteso (quello che `pnpm db:generate` deve produrre, a meno dell'ordine): sette `CREATE TABLE`
con le colonne sopra, `FOREIGN KEY (workout_id) REFERENCES workouts(id) ON DELETE CASCADE`, gli
indici `workouts_start_time_idx`, `workout_sets_workout_idx`, `workout_sets_template_idx`,
`hevy_routines_hevy_id_idx` (UNIQUE), `hevy_templates_hevy_id_idx`, `slot_state_history_slot_idx`.
`applyConnectionPragmas` accende le FK a ogni connessione; sul Turso remoto non ci si conta (§ 3 g) e
le cancellazioni sono esplicite.

### 4.3 Repository — `src/lib/db/repositories/`

Tutte le funzioni ricevono `db: Db` come primo parametro (come segnaprezzi) e non conoscono Hevy.

- `workouts.ts`: `upsertWorkout(db, workout: HevyWorkout, now)` — un `db.batch([delete sets where
  workoutId, insert workouts … onConflictDoUpdate (tutte le colonne, `routine_id` compreso)])` più
  l'insert dei set **solo se il workout ha almeno un set** (Drizzle rifiuta `values([])`; un
  workout vuoto si salva con 0 righe di set); `deleteWorkout(db, id)` — `db.batch([delete sets where
  workoutId, delete workouts where id])`, esplicito (§ 3 g), restituisce il numero di workout
  cancellati (0 o 1); `listSetsBetween(db, { fromMs, toMs }): Promise<SetRow[]>` → righe di
  `workout_sets` join `workouts` con `startTime >= fromMs AND startTime < toMs`, ordinate per
  `startTime`, `exerciseIndex`, `setIndex`, con la riga **piatta**
  `interface SetRow { workoutId: string; startTime: number; routineId: string | null; exerciseIndex: number; setIndex: number; exerciseTemplateId: string; exerciseTitle: string; setType: string; reps: number | null; durationSeconds: number | null }`;
  `getLatestWorkout(db)` (05a); `countWorkouts(db)`; `listWorkoutIds(db): Promise<string[]>` (per
  la potatura, D-49).
- `sync-state.ts`: `getSyncValue(db, key): Promise<string | null>`, `setSyncValue(db, key, value,
  now)`; i formati per chiave sono nel commento dello schema.
- `hevy-routines.ts`: `upsertHevyRoutine(db, row)` (per `routineKey`: una routine ricreata su Hevy
  con un id nuovo sovrascrive l'id; i workout vecchi con il `routine_id` precedente non si mappano
  più a una `RoutineKey` e la 04 li risolve per template), `listHevyRoutines(db)`,
  `routineKeyByHevyId(db)` → `Map<string, RoutineKey>`.
- `hevy-templates.ts`: `upsertTemplateMap(db, rows: Array<{ variantId: VariantId; hevyTemplateId:
  string; title: string; isCustom: boolean }>, now)` (`title` e `isCustom` = `title` e `is_custom`
  del template trovato in `listExerciseTemplates()`; per un template appena creato dalla 04:
  `true`), `getTemplateMap(db)` → `TemplateIdMap`, `variantsByTemplateId(db)` →
  `Map<string, VariantId[]>`.
- `slot-state.ts`: `listSlotStates(db): Promise<SlotState[]>`; `ensureInitialSlotStates(db, states,
  validFrom)` inserisce stato e riga di storia (`source 'plan'`, `proposalId null`) per ogni stato
  il cui `slotId` **manca** in `slot_state` (tabella vuota: tutti e 30; uno slot aggiunto al piano in
  futuro: solo quello); `listSlotStateHistory(db): Promise<SlotStateHistoryRow[]>` (tutte le righe,
  ordinate per `slotId`, `validFrom`); `replaceSlotStateStatements(db, next: SlotState, { validFrom,
  proposalId }): BatchItem<'sqlite'>[]` restituisce **senza eseguirle** le tre istruzioni (upsert di
  `slot_state`; `UPDATE slot_state_history SET valid_to = validFrom WHERE slot_id = ? AND valid_to IS
  NULL`; insert della nuova riga di storia), così la 04 le mette nel proprio `db.batch` insieme
  all'update della proposta; `replaceSlotState(db, next, opts)` è
  `db.batch(replaceSlotStateStatements(db, next, opts))`.
- `src/lib/plan/history.ts` (puro, parte di C-3):
  `interface SlotStateHistoryRow { slotId: SlotId; variantId: VariantId; target: Target; validFrom: number; validTo: number | null; source: 'plan' | 'apply'; proposalId: string | null }`
  (`target` già deserializzato da `target_json`) e `variantAt(history: SlotStateHistoryRow[],
  slotId, atMs): VariantId | null`: la variante della riga dello slot con `validFrom ≤ atMs <
  (validTo ?? +∞)`; se `atMs` precede la prima riga dello slot, **la variante della prima riga**
  (D-42: gli allenamenti fatti prima del piano contano, D-33); `null` solo se lo slot non ha righe.
  `targetAt(history, slotId, atMs): Target | null` con la stessa regola (la 04 e la 05a lo usano).

### 4.4 Tempo — `src/lib/time.ts` (C-17)

```ts
export const ROME = 'Europe/Rome';
/** Hevy sends ISO strings with an offset or Z; Date.parse handles both. Throws on NaN. */
export function parseHevyInstant(iso: string): number
/** Calendar date YYYY-MM-DD of an instant in Europe/Rome (Intl.DateTimeFormat, en-CA). */
export function romeDate(ms: number): string
/** Whole calendar days between two instants in Europe/Rome (b − a), DST-safe (compares romeDate values). */
export function romeDaysBetween(aMs: number, bMs: number): number
/** Start (epoch ms) of the Europe/Rome calendar day that contains `ms`, minus `days` calendar days (DST-safe: built from the date, not from ms arithmetic). */
export function romeDayStartMinusDays(ms: number, days: number): number
```

Tutte e quattro usano `Intl.DateTimeFormat` con `timeZone: ROME`, mai i getter locali di `Date`: i
test girano in UTC come Vercel (01 § 4.2) e `time.test.ts` cambia anche fuso a metà (§ 6). La
finestra della 04, `[romeDayStartMinusDays(now, 28), now)`, copre 28 giorni pieni più il giorno
corrente: è ciò che D-15 intende con «28 giorni».

### 4.5 Sincronizzazione — `src/lib/sync/`

`sync-workouts.ts` (C-8):

```ts
export interface SyncResult { mode: 'full' | 'incremental'; fetched: number; upserted: number; deleted: number; cursor: string | null }

export async function syncWorkouts({ db, hevy, now, forceFull = false }: { db: Db; hevy: HevyClient; now: number; forceFull?: boolean }): Promise<SyncResult> {
  const cursor = forceFull ? null : await getSyncValue(db, 'workouts_cursor');
  if (!cursor) return fullSync({ db, hevy, now });
  return incrementalSync({ db, hevy, now, since: cursor });
}
// fullSync: pages 1..page_count of hevy.listWorkouts(page) (page_count 0 → no pages); upsert each;
//   seen = the ids received; after the last page delete, in one batch, the local workouts whose id is
//   not in seen (D-49: counted in `deleted`); cursor = max parseHevyInstant(updated_at) as ISO, or
//   new Date(now).toISOString() when the account has no workouts (so the next run is incremental);
//   then set workouts_cursor, last_full_sync_at, last_sync_at, last_sync_mode = 'full'. Nothing in
//   sync_state is written before the last page: a failure half-way leaves the cursor untouched.
// incrementalSync: read ALL pages 1..page_count of hevy.listWorkoutEvents({ since, page }) before
//   writing; instant of an event = parseHevyInstant(workout.updated_at) or parseHevyInstant(deleted_at)
//   (a `deleted` without deleted_at: instant = since, it deletes but does not move the cursor);
//   drop events with instant <= since; keep only the most recent event per workout id; apply from the
//   oldest to the newest ('updated' → upsertWorkout, 'deleted' → deleteWorkout); `fetched` = events
//   read, `upserted` = upserts done, `deleted` = workouts really deleted; cursor =
//   new Date(max(instants, since)).toISOString(); then set workouts_cursor, last_sync_at,
//   last_sync_mode = 'incremental', after the last event applied.

/** One workout, for the webhook (06). Does not touch the cursor nor last_sync_at. */
export async function syncOneWorkout({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, workoutId: string): Promise<{ status: 'upserted' | 'not_found' }>
// getWorkout(id) → upsertWorkout → 'upserted'. HevyApiError 404 → deleteWorkout(db, id) (0 or 1 rows:
// Hevy is the source of truth, D-48) → 'not_found'. Any other error propagates.
```

Webhook (06) e cron possono sincronizzare lo stesso workout nello stesso momento: ogni upsert è un
batch atomico e i due scrivono gli stessi dati, quindi l'ultimo vince senza danni.

`sync-routines.ts` (sola lettura su Hevy, T-7):

```ts
export async function syncRoutines({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }): Promise<{ routines: number; templates: number; planUploadedAt: number }>
// 1. folders = listRoutineFolders().filter(title === FOLDER_TITLE): 0 → throw new Error('Hevy folder not found: run `pnpm hevy:upload` first'); 2+ → throw new Error('2 folders titled …: fix it by hand in the Hevy app')
// 2. routines = listRoutines().filter(folder_id === folder.id); for each PLAN routine: the routines with that title: 0 → throw (same message as the folder), 2+ → throw (duplicate); upsertHevyRoutine({ routineKey, hevyRoutineId, folderId, title, hevyCreatedAt, hevyUpdatedAt, snapshotJson: JSON.stringify(routine) }, now)
// 3. plan_uploaded_at: if missing in sync_state, String(min(parseHevyInstant(created_at)) of the 5); never overwritten
// 4. ensureInitialSlotStates(db, initialSlotStates(planUploadedAt), planUploadedAt); states = listSlotStates(db)  ← read AFTER the insert
// 5. existing = await hevy.listExerciseTemplates(); { map, missing } = resolveTemplates(allVariantsInUse(states), existing);
//    missing.length > 0 → throw new Error(`Hevy template missing for ${missing.map((v) => v.id).join(', ')}: run \`pnpm hevy:upload\` first`);
//    upsertTemplateMap(db, rows built from map + existing (title, is_custom), now). No write to Hevy, ever.
```

`scripts/hevy-sync.ts` (forma `main()` della 01; env con `EnvSchema.pick({ HEVY_API_KEY: true })`,
il DB le legge da sé): prima `syncRoutines({ db, hevy, now })`, poi `syncWorkouts({ db, hevy, now,
forceFull: process.argv.includes('--full') })`; stampa due righe, `routines=5 templates=29
planUploadedAt=<ISO>` e `workouts: mode=… fetched=… upserted=… deleted=… cursor=…`.

## 5. Vincoli e trappole

- **AGENTS.md § 5 Database** (stub scritto al bootstrap; questa spec lo completa, § 9) — qui vuol
  dire che: lo schema si cambia SOLO con `pnpm db:generate` e una migrazione committata; mai
  `drizzle-kit push`; FK accese dal client ma cancellazioni esplicite; `batch()` al posto delle
  transazioni interattive (§ 3 c); epoch ms in tutte le colonne di tempo.
- **AGENTS.md § 3 Hevy** — `listWorkouts` è a pagine da 10: il full sync fa 18+ chiamate: si lancia
  solo la prima volta o con `--full`; mai a ogni cron. `syncRoutines` non scrive mai su Hevy.
- **D-36** — `src/lib/db/client.ts` non importa `@/lib/env`; `vitest.config.ts` aggiunge i valori
  finti `TURSO_DATABASE_URL: ':memory:'` (i test non lo usano: `createMemoryDb` crea il suo client).
- **T-15** — `romeDaysBetween` confronta date di calendario, non divide i millisecondi per 86400000:
  il 2026-10-25 (fine dell'ora legale) un giorno dura 25 ore.
- L'upsert riscrive **tutte** le colonne di `workouts` (compreso `routine_id`) e cancella e riscrive
  i set: qualunque modifica fatta su Hevy (evento `updated`) arriva intera.
- `:memory:` con `@libsql/client` è una connessione sola: il client di test non va cacheato su
  `globalThis` (il `createMemoryDb` crea un client nuovo per test) e nessun test apre un secondo
  client sullo stesso DB.
- `drizzle/meta/_journal.json` e gli `.sql` sono tracciati; `local.db*` no (`.gitignore`).
- Da qui esiste `slot_state`, ma «Applica» arriva con la 04: in questa spec `pnpm hevy:upload`
  (sempre stati iniziali, D-40) resta innocuo. Una routine modificata a mano nell'app Hevy non è
  rilevata dalla sync (deriva rispetto a `slot_state`): CLAUDE.md § Not yet present lo dice (§ 9).

## 6. Test

Il client Hevy finto dei test di sync è quello vero con la rete finta della 01:
`createHevyClient({ apiKey: 'test', fetch: fakeFetch(routes).fn, retryDelayMs: 0 })`; la rotta di
`/v1/workouts/events` legge `since` dalla query e restituisce solo gli eventi con istante > `since`.
`sync-routines.test.ts` usa `createFakeHevy` della 02.

| Livello | Test | Mutazione |
|---|---|---|
| Integrazione (DB in memoria) | `src/lib/db/migrate.test.ts`: 7 tabelle e 6 indici presenti; `PRAGMA foreign_keys` = 1. | Togliere `hevy_templates_hevy_id_idx` dallo schema e lanciare `pnpm db:generate` → rosso; togliere `foreign_keys` da `applyConnectionPragmas` → rosso. |
| Integrazione | `src/lib/db/repositories/workouts.test.ts`: upsert della fixture `workout.json` → 1 workout, 14 set, `exerciseIndex`/`setIndex` = `index` di Hevy; upsert di nuovo con un set in meno → 13; `exercises: []` → 1 workout, 0 set; `deleteWorkout` → 1, poi 0 set e 0 workout; `deleteWorkout` di un id ignoto → 0; `listSetsBetween` esclude i workout fuori finestra (bordo: `toMs` escluso) e le righe hanno `startTime` e `routineId`. | Togliere il delete esplicito dei set (lasciando solo la cascade) → i set restano → rosso; insert dei set sempre presente → il caso vuoto lancia → rosso. |
| Unit | `src/lib/time.test.ts` (`beforeAll`: `process.env.TZ = 'America/New_York'`, `afterAll` lo ripristina: Node rilegge `TZ` a ogni assegnazione): `parseHevyInstant('2026-10-22T17:31:47+00:00')` = `Date.UTC(2026,9,22,17,31,47)`; `parseHevyInstant('x')` lancia; `romeDate` del 2026-10-25T00:30Z = `2026-10-25`, del 2026-10-24T22:30Z = `2026-10-25`; `romeDaysBetween(2026-10-24T20:00Z, 2026-11-07T07:17Z)` = 14 attraverso il cambio d'ora; `romeDayStartMinusDays(Date.UTC(2026,10,7,7,17), 28)` = `Date.UTC(2026,9,9,22,0)` (00:00 del 10 ottobre, ora legale). | Dividere per 86400000 → il caso del cambio d'ora va rosso; `romeDate` con i getter locali di `Date` → il caso 22:30Z va rosso; inizio del giorno − 28 × 86400000 → `Date.UTC(2026,9,9,23,0)` → rosso. |
| Unit + DB | `src/lib/sync/sync-workouts.test.ts`: full con 3 pagine finte (10+10+5) → 25 righe, cursore = ISO del massimo `updated_at`, `last_sync_mode = 'full'`; full con un workout locale in più assente dalle pagine → `deleted 1` (D-49); account vuoto (`page_count: 0`) → cursore = ISO di `now`; incremental con `workout_events.json` dopo aver inserito con `upsertWorkout` un workout con id `22222222-2222-4222-8222-222222222222` → `fetched 2`, `upserted 1`, `deleted 1`, `22222222…` e i suoi set assenti, cursore `'2026-10-22T17:52:49.853Z'`; rilancio → `upserted 0`, `deleted 0`, cursore uguale; `deleted` di un id mai visto → `deleted 0`, nessun errore; `deleted` senza `deleted_at` → cancella, cursore invariato; eventi `[deleted X 10:00, updated X 09:00]` → X assente; errore sulla seconda pagina (500 due volte) → `HevyApiError`, cursore e DB invariati; `forceFull` ignora il cursore. | Non avanzare il cursore → il rilancio rifà l'upsert → rosso; applicare nell'ordine ricevuto → X presente → rosso; scrivere il cursore pagina per pagina → il caso dell'errore va rosso. |
| Unit + DB | `src/lib/sync/sync-workouts.test.ts` (`syncOneWorkout`): 200 → `upserted`, 1 workout; 404 su id ignoto → `not_found`, 0 righe; 404 su id presente → `not_found`, workout e set cancellati; 500 due volte → lancia. | Propagare il 404 → rosso; non cancellare → rosso. |
| Unit + DB | `src/lib/sync/sync-routines.test.ts` con `createFakeHevy`: 5 routine finte nella cartella (titoli D-25) → 5 righe con `snapshot_json`, `plan_uploaded_at` = `String(min created_at)`, 30 `slot_state`, 30 righe di storia con `validFrom` = plan_uploaded_at, 29 righe in `hevy_templates` con `is_custom` giusto; rilancio → invariato, `writes.length === 0`; cartella assente → errore che contiene `pnpm hevy:upload`; due cartelle omonime → errore; un custom in uso tolto dalla lista → errore che contiene l'id della variante e `writes.length === 0`; `slot_state` con 29 righe → la trentesima nasce. | Sovrascrivere `plan_uploaded_at` → rosso; leggere `states` prima dell'insert → 0 template → rosso; chiamare `ensureTemplates` → `writes` non vuoto → rosso. |
| Unit | `src/lib/plan/history.test.ts`: `variantAt` con due righe (validTo chiuso) sceglie quella giusta ai bordi (`validFrom` incluso, `validTo` escluso); un istante **prima** della prima riga → la variante della prima riga (D-42); slot senza righe → `null`; `targetAt` idem. | Invertire i bordi → rosso; restituire `null` prima della prima riga → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` verdi; `pnpm
  db:generate` a schema invariato non crea file in `drizzle/` (niente deriva); `pnpm hevy:upload --
  --dry-run` ancora 0 scritture (circa 7 GET reali: lettura, gratis).
- **C — Comportamento nuovo**: `pnpm db:migrate` su `file:local.db` → `drizzle/0000_*.sql` applicata;
  `pnpm db:studio` mostra le 7 tabelle vuote.
- **D — Sotto la UI (sole GET reali, gratis; non a xx:00)**: `pnpm hevy:sync`. Effetto: GET
  cartelle (1), routine (1), template (5 pagine), `/v1/workouts` (18+ pagine); nessuna scrittura su
  Hevy. Esito atteso: `routines=5 templates=29` (30 slot, `hollow_hold` in due: 29 varianti
  distinte, il numero che `sync-routines.test.ts` asserisce), `mode=full fetched=<count reale ≥
  174> deleted=0`; `SELECT COUNT(*) FROM workouts` = count; `sync_state` con `workouts_cursor`,
  `plan_uploaded_at`, `last_sync_mode=full`. Poi `pnpm hevy:sync` di nuovo (le 7 GET di
  `syncRoutines` + 1 GET degli eventi) → `mode=incremental fetched=0..k upserted=0..k`. Poi, se
  l'owner ha registrato un allenamento nel frattempo, il terzo sync lo porta (controllo con
  `SELECT title, start_time FROM workouts ORDER BY start_time DESC LIMIT 1`); altrimenti la fase si
  dichiara saltata.
- **F — Giro guidato**: `pnpm db:studio`: l'owner guarda (1) `hevy_routines` con i 5 titoli, (2)
  `slot_state` con 30 righe `source='plan'`, (3) l'ultimo workout in `workouts` e i suoi set.
- **G — Ripristino**: `local.db` resta (è lo sviluppo); nessuna fixture piantata.

Che cosa NON copre: l'aggregazione (04), il webhook (06).

## 8. Messa in esercizio

- Env (Development): `TURSO_DATABASE_URL="file:local.db"`, `TURSO_AUTH_TOKEN=""`.
- DDL: `drizzle/0000_*.sql` via `pnpm db:migrate` (locale ora; in produzione nella 06, prima del
  primo deploy che la usa).
- Misura di chiusura: `pnpm hevy:sync` incrementale senza errori. In esercizio = PR mergiata su
  `main` + collaudo annotato in CLAUDE.md (D-37); la 06 rifà la misura in produzione.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Architecture Snapshot (DB, sync: cosa esiste); § Data & Integrations
  (Turso locale); § Not yet present: via la parte «migrations» della prima riga; nuove righe «Sync
  of body measurements: not done (no feature reads them)» e «Detecting routines edited by hand in
  the Hevy app: the sync does not compare them with `slot_state`».
- `AGENTS.md`: § 5 Database da stub a regole, ≤ 15 righe (Drizzle, migrazioni, batch senza
  transazioni, cancellazioni esplicite, epoch ms, formati di `sync_state`, cursore, full contro
  incrementale, ordine degli eventi); § 10 Commands invariato (elenca già `db:*` e `hevy:sync`).
- `doc/guide/sync.md` (≤ 200 righe): il modello dei dati, il cursore, la storia degli slot e D-42,
  come si rifà un full sync (`--full`, che pota), come si legge `local.db`.
- `README.md` (inglese): nuova sezione `## Syncing` (`pnpm db:migrate`, `pnpm hevy:sync`,
  `--full`), ≤ 12 righe.
- `WORKFLOW.md` § 3: `db:generate`, `db:migrate`, `db:studio`, `hevy:sync [-- --full]` reali dalla 03.
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000), `wc -l
  AGENTS.md` (≤ 600), `wc -l doc/guide/sync.md` (≤ 200).

## 10. Fuori ambito, dichiarato

- Aggregazione e finestra di revisione → **04**.
- Route del webhook e cron → **06** (usano `syncOneWorkout` e `syncWorkouts`).
- `body_measurements`: non sincronizzati (scelta di questa spec: nessuna funzione li legge) →
  CLAUDE.md § Not yet present.
- Rilevare le routine modificate a mano nell'app Hevy → CLAUDE.md § Not yet present.
- Turso remoto, token, regione → **06**.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **M**.

```
Ciao, in questa sessione implemento la spec doc/v1/03-turso-schema-e-sincronizzazione.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md, AGENTS.md (CLAUDE.md è già nel contesto), COMMENTS.md,
  DEVELOPMENT_GUIDELINES.md, doc/guide/piano.md
- Leggi doc/v1/00-INDICE.md per intero e poi la spec 03 INTERA
- Verifica con un grep che i contratti della 01 e della 02 esistano con quel nome e quella
  firma (createHevyClient, HevyClient, HevyApiError, HevyWorkout, HevyWorkoutEvent, EnvSchema in
  src/lib/env-schema.ts, fakeFetch e fixture in tests/helpers, PLAN, FOLDER_TITLE, RoutineKey,
  SlotState, Target, TemplateIdMap, initialSlotStates(now), allVariantsInUse, resolveTemplates,
  createFakeHevy in tests/helpers/fake-hevy.ts)
- Il repository di riferimento è ../segnaprezzi (sola lettura): src/lib/db/client.ts e
  drizzle.config.ts si ricalcano con le differenze che la spec elenca

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/03-turso-sync. Crea SESSION_NOTES.md.

Implementa la spec 03 esattamente come scritta: § 4.2 è lo schema (nomi di tabelle e colonne
sono contratti), § 4.3–4.5 il codice, § 6 i test (DB in memoria con le migrazioni vere), § 7 il
collaudo fase per fase con l'esito atteso prima (le chiamate reali sono sole GET, syncRoutines
non scrive mai su Hevy; le lanci dopo il mio OK), § 9 la documentazione. Se un punto non è
coperto dalla spec, fermati e chiedimelo con AskUserQuestion. Le etichette della serie non
entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
