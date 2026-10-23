# 03 — Turso: schema, migrazioni e sincronizzazione degli allenamenti

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la 02.
> **Branch**: `feature/03-turso-sync` da `main`. **Dipende da**: 01 (client, schemi), 02 (`PLAN`,
> `SCALES`, `initialSlotStates`, `ensureTemplates`, `FOLDER_TITLE`, routine già su Hevy).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-32, D-33; scelte T-3, T-8, T-15,
> T-16; contratti C-7 (prime sette tabelle), C-8, C-17 (li crea).

## 1. Il problema oggi (prove)

1. Non c'è persistenza: la 02 porta le routine su Hevy e stampa gli id, ma nessuno li conserva; lo
   stato degli slot (variante e target correnti) esiste solo come `initialSlotStates()` in memoria.
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
5. Turso non ha Francoforte (T-1) e i log di Vercel durano 1 ora: ogni corsa di sync deve lasciare
   traccia in tabella, non solo nei log.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | Lo schema Drizzle (C-7, § 4.2) genera la migrazione `drizzle/0000_*.sql` con le sette tabelle e gli indici di § 4.2; `pnpm db:migrate` la applica a `file:local.db`; i test la applicano a `:memory:`. | `db/migrate.test.ts`: `migrate()` su `:memory:` poi `SELECT name FROM sqlite_master` contiene le 7 tabelle. Mutazione: togliere un indice dallo schema → il test degli indici va rosso. |
| A2 | `syncWorkouts` (C-8) in modalità **full** scarica tutte le pagine e inserisce workout e set; in modalità **incremental** usa gli eventi dal cursore, applica `updated` (upsert) e `deleted` (delete) e avanza il cursore; è idempotente (rilanciata senza eventi: 0 scritture). | `sync-workouts.test.ts` con client finto: 3 pagine → 25 workout; eventi con 1 updated + 1 deleted → 1 upsert, 1 delete, cursore = max(`updated_at`, `deleted_at`); rilancio → 0. Mutazione: non cancellare sui `deleted` → rosso. |
| A3 | `syncOneWorkout` (C-8) fa `getWorkout(id)` e l'upsert; un id inesistente (`HevyApiError` 404) → `{ status: 'not_found' }` senza eccezione. | Test con client finto. Mutazione: propagare il 404 → rosso. |
| A4 | `syncRoutines` registra in `hevy_routines` le 5 routine della cartella `FOLDER_TITLE` (per titolo → `RoutineKey`), in `hevy_templates` la mappa variante → id per tutte le varianti in uso, inizializza `slot_state` e `slot_state_history` se vuote con `validFrom = plan_uploaded_at`, e scrive `sync_state.plan_uploaded_at` = min(`created_at`) delle 5 routine, una volta sola. | `sync-routines.test.ts`: routine finte nella cartella → 5 righe; rilancio → stesse 5, `plan_uploaded_at` invariato; cartella mancante → errore «run hevy:upload first». Mutazione: sovrascrivere `plan_uploaded_at` → rosso. |
| A5 | `workout_sets` è denormalizzata e completa: un set per riga con `exercise_template_id`, `set_type`, `reps`, `duration_seconds`; i `warmup` restano in tabella (li esclude la 04). | Test: il workout della fixture → 14 righe di set (5+3+3+3). Mutazione: saltare i warmup → 12 righe → rosso. |
| A6 | `pnpm hevy:sync` reale popola `local.db` con tutti i workout e le 5 routine; rilanciato, fa solo la chiamata agli eventi. | Collaudo § 7 (sole GET). |
| A7 | Gli helper di tempo (C-17) convertono le date Hevy e contano i giorni di calendario in Europe/Rome anche a cavallo del cambio d'ora. | `time.test.ts` con casi fissi (§ 6). |

## 3. Decisioni

- **D-1** (Turso), **D-32** (webhook: qui nasce `syncOneWorkout`; la route è della 06), **D-33**
  (tutti i workout, sync incrementale via eventi).
- **T-3** (`@libsql/client` Node, singleton, WAL su file), **T-8** (storia degli stati),
  **T-15** (Europe/Rome; epoch ms in DB), **T-16** (`plan_uploaded_at` ricavata da Hevy).
- Adottate qui, ribaltabili: (a) `raw_json` del workout conservato in `workouts` (serve a
  rigenerare i set se la 04 cambia l'aggregazione, costa pochi KB); (b) il cursore degli eventi è
  l'ISO del massimo fra `updated_at` e `deleted_at` visti, e la chiamata `since` lo rimanda tale e
  quale (rivedere lo stesso evento è innocuo: l'upsert è idempotente); (c) **nessuna transazione
  interattiva**: ogni upsert è un `db.batch([...])` (delete set, upsert workout, insert set);
  (d) `slot_state_history.valid_to` nullo = corrente; (e) le migrazioni si generano con drizzle-kit e
  si committano in `drizzle/`; in produzione le applica `pnpm db:migrate` dal PC dell'owner con le
  env di produzione (06), mai un hook di build; (f) `sync_state` è una tabella chiave/valore testuale.

## 4. La soluzione

### 4.1 Dipendenze e configurazione

```bash
pnpm add @libsql/client@0.18.0 drizzle-orm@0.45.4
pnpm add -D drizzle-kit@0.31.11
```

`drizzle.config.ts` (copia di segnaprezzi, con uno schema solo):

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

`src/lib/db/client.ts`: identico a `segnaprezzi/src/lib/db/client.ts` (singleton su `globalThis`,
`PRAGMA journal_mode = WAL` e `busy_timeout = 5000` solo su `file:`), con `import * as schema from
'./schema'` e `export type Db = typeof db`. Per i test: `src/lib/db/testing/memory-db.ts` →
`createMemoryDb(): Promise<Db>` che fa `createClient({ url: ':memory:' })`, `drizzle(client, { schema })`
e `migrate(db, { migrationsFolder: 'drizzle' })` (`drizzle-orm/libsql/migrator`).

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

/** Key/value bookkeeping: workouts_cursor, last_sync_at, last_full_sync_at, plan_uploaded_at (all ISO or epoch-ms strings). */
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
Il client esegue `PRAGMA foreign_keys = ON` all'avvio (subito dopo i pragma di WAL, anche su Turso
remoto) perché SQLite non applica le FK senza.

### 4.3 Repository — `src/lib/db/repositories/`

Tutte le funzioni ricevono `db: Db` come primo parametro (come segnaprezzi) e non conoscono Hevy.

- `workouts.ts`: `upsertWorkout(db, workout: HevyWorkout, now)` — un `db.batch([delete sets where
  workoutId, insert workouts … onConflictDoUpdate, insert sets…])`; `deleteWorkout(db, id)`;
  `listSetsBetween(db, { fromMs, toMs })` → righe di `workout_sets` join `workouts` (`startTime`,
  `routineId`) con `startTime >= fromMs AND startTime < toMs`, ordinate per `startTime`,
  `exerciseIndex`, `setIndex`; `countWorkouts(db)`.
- `sync-state.ts`: `getSyncValue(db, key)`, `setSyncValue(db, key, value, now)`.
- `hevy-routines.ts`: `upsertHevyRoutine(db, row)`, `listHevyRoutines(db)`, `routineKeyByHevyId(db)`
  → `Map<string, RoutineKey>`.
- `hevy-templates.ts`: `upsertTemplateMap(db, map: TemplateIdMap, titles, now)`, `getTemplateMap(db)`
  → `TemplateIdMap`, `variantsByTemplateId(db)` → `Map<string, VariantId[]>`.
- `slot-state.ts`: `listSlotStates(db)`, `ensureInitialSlotStates(db, states, validFrom)` (solo se
  la tabella è vuota: inserisce stato e storia), `replaceSlotState(db, next: SlotState, { validFrom,
  proposalId })` — chiude la riga di storia corrente (`validTo = validFrom`) e inserisce la nuova, in
  un batch; `variantAt(history, slotId, atMs)` (pura, in `src/lib/plan/history.ts`).

### 4.4 Tempo — `src/lib/time.ts` (C-17)

```ts
export const ROME = 'Europe/Rome';
/** Hevy sends ISO strings with an offset or Z; Date.parse handles both. Throws on NaN. */
export function parseHevyInstant(iso: string): number
/** Calendar date YYYY-MM-DD of an instant in Europe/Rome (Intl.DateTimeFormat, en-CA). */
export function romeDate(ms: number): string
/** Whole calendar days between two instants in Europe/Rome (b − a), DST-safe (compares romeDate values). */
export function romeDaysBetween(aMs: number, bMs: number): number
/** Start (epoch ms) of the Europe/Rome calendar day that contains `ms`, minus `days` days. */
export function romeDayStartMinusDays(ms: number, days: number): number
```

### 4.5 Sincronizzazione — `src/lib/sync/`

`sync-workouts.ts` (C-8):

```ts
export interface SyncResult { mode: 'full' | 'incremental'; fetched: number; upserted: number; deleted: number; cursor: string | null }

export async function syncWorkouts({ db, hevy, now, forceFull = false }: { db: Db; hevy: HevyClient; now: number; forceFull?: boolean }): Promise<SyncResult> {
  const cursor = forceFull ? null : await getSyncValue(db, 'workouts_cursor');
  if (!cursor) return fullSync({ db, hevy, now });
  return incrementalSync({ db, hevy, now, since: cursor });
}
// fullSync: page 1..page_count of hevy.listWorkouts(page); upsert each; cursor = max hevy updated_at (ISO); set last_full_sync_at, last_sync_at.
// incrementalSync: page 1..page_count of hevy.listWorkoutEvents({ since, page }); 'updated' → upsert, 'deleted' → delete; cursor = max(updated_at, deleted_at, since); set last_sync_at.

export async function syncOneWorkout({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }, workoutId: string): Promise<{ status: 'upserted' | 'not_found' }>
```

`sync-routines.ts`:

```ts
export async function syncRoutines({ db, hevy, now }: { db: Db; hevy: HevyClient; now: number }): Promise<{ routines: number; templates: number; planUploadedAt: number }>
// 1. folder = listRoutineFolders().find(title === FOLDER_TITLE) ?? throw new Error('Hevy folder not found: run `pnpm hevy:upload` first')
// 2. routines = listRoutines().filter(folder_id === folder.id); for each PLAN routine: find by title ?? throw (same message); upsertHevyRoutine
// 3. plan_uploaded_at: if missing in sync_state, min(parseHevyInstant(created_at)) of the 5; never overwritten
// 4. states = listSlotStates(db); if empty → ensureInitialSlotStates(db, initialSlotStates(planUploadedAt), planUploadedAt)
// 5. templateIds = ensureTemplates(hevy, allVariantsInUse(states), await hevy.listExerciseTemplates()); upsertTemplateMap
```

`scripts/hevy-sync.ts`: `--full` forza il full sync; stampa `routines=5 templates=N workouts: mode=… fetched=… upserted=… deleted=… cursor=…`.

## 5. Vincoli e trappole

- **AGENTS.md § Database** (nasce qui, § 9) — qui vuol dire che: lo schema si cambia SOLO con
  `pnpm db:generate` e una migrazione committata; mai `drizzle-kit push`; FK accese dal client;
  `batch()` al posto delle transazioni interattive; epoch ms in tutte le colonne di tempo.
- **AGENTS.md § Hevy** — `listWorkouts` è a pagine da 10: il full sync fa 18+ chiamate: si lancia
  solo la prima volta o con `--full`; mai a ogni cron.
- **T-15** — `romeDaysBetween` confronta date di calendario, non divide i millisecondi per 86400000:
  il 2026-10-25 (fine dell'ora legale) un giorno dura 25 ore.
- Un workout Hevy può cambiare `routine_id`? No: il campo è fissato alla creazione; ma può essere
  modificato nei set (evento `updated`) → l'upsert cancella e riscrive i set.
- `:memory:` con `@libsql/client` è una connessione sola: il client di test non va cacheato su
  `globalThis` (il `createMemoryDb` crea un client nuovo per test).
- `drizzle/meta/_journal.json` e gli `.sql` sono tracciati; `local.db*` no (`.gitignore`).

## 6. Test

| Livello | Test | Mutazione |
|---|---|---|
| Integrazione (DB in memoria) | `migrate.test.ts`: tabelle e indici presenti; `PRAGMA foreign_keys` = 1. | Togliere l'indice sui template → rosso. |
| Integrazione | `workouts.repository.test.ts`: upsert della fixture → 1 workout, 14 set; upsert di nuovo con un set in meno → 13; delete → 0 set (cascade); `listSetsBetween` esclude i workout fuori finestra (bordo: `toMs` escluso). | Togliere `onDelete: 'cascade'` e la cancellazione esplicita → i set restano → rosso. |
| Unit | `time.test.ts`: `parseHevyInstant('2026-10-22T17:31:47+00:00')` = `Date.UTC(2026,9,22,17,31,47)`; `romeDate` del 2026-10-25T00:30Z = `2026-10-25`, del 2026-10-24T22:30Z = `2026-10-25`; `romeDaysBetween(2026-10-24T20:00Z, 2026-11-07T07:17Z)` = 14 attraverso il cambio d'ora; `romeDayStartMinusDays(now, 28)` cade a mezzanotte di Roma. | Dividere per 86400000 → il caso del cambio d'ora va rosso. |
| Unit + DB | `sync-workouts.test.ts`: full con 3 pagine finte (10+10+5) → 25 righe, cursore = max; incremental con `workout_events.json` → 1 upsert + 1 delete (il deleted non esisteva: 0 righe toccate, nessun errore); rilancio senza eventi (`page_count: 1`, `events: []`) → 0 scritture; `forceFull` ignora il cursore. | Non avanzare il cursore → il rilancio rifà l'upsert → rosso. |
| Unit + DB | `sync-routines.test.ts`: 5 routine finte nella cartella (titoli D-25) → 5 righe, `plan_uploaded_at` = min `created_at`, 30 `slot_state`, 30 righe di storia con `validFrom` = plan_uploaded_at; rilancio → invariato; cartella assente → errore con «hevy:upload». | Sovrascrivere `plan_uploaded_at` → rosso. |
| Unit | `history.test.ts`: `variantAt` con due righe (validTo chiuso) sceglie quella giusta ai bordi (`validFrom` incluso, `validTo` escluso). | Invertire i bordi → rosso. |

## 7. Collaudo guidato

- **A — Invarianza**: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` verdi; `pnpm
  hevy:upload -- --dry-run` ancora 0 scritture.
- **C — Comportamento nuovo**: `pnpm db:migrate` su `file:local.db` → `drizzle/0000_*.sql` applicata;
  `pnpm db:studio` mostra le 7 tabelle vuote.
- **D — Sotto la UI (sole GET reali)**: `pnpm hevy:sync`. Esito atteso: `routines=5`, `templates=30`
  (le varianti in uso: 30 slot → 29 varianti distinte più… **il numero esatto lo calcola il test
  `lookup.test.ts` della 02: `allVariantsInUse(initialSlotStates()).length`**), `mode=full
  fetched=<count reale ≥ 174>`; `SELECT COUNT(*) FROM workouts` = count; `sync_state` con
  `workouts_cursor`, `plan_uploaded_at`. Poi `pnpm hevy:sync` di nuovo → `mode=incremental
  fetched=0..k upserted=0..k`. Poi, se l'owner ha registrato un allenamento nel frattempo, il terzo
  sync lo porta (controllo con `SELECT title, start_time FROM workouts ORDER BY start_time DESC
  LIMIT 1`).
- **F — Giro guidato**: `pnpm db:studio`: l'owner guarda (1) `hevy_routines` con i 5 titoli, (2)
  `slot_state` con 30 righe `source='plan'`, (3) l'ultimo workout in `workouts` e i suoi set.
- **G — Ripristino**: `local.db` resta (è lo sviluppo); nessuna fixture piantata.

Che cosa NON copre: l'aggregazione (04), il webhook (06).

## 8. Messa in esercizio

- Env (Development): `TURSO_DATABASE_URL="file:local.db"`, `TURSO_AUTH_TOKEN=""`.
- DDL: `drizzle/0000_*.sql` via `pnpm db:migrate` (locale ora; in produzione nella 06, prima del
  primo deploy che la usa).
- Misura di chiusura: `pnpm hevy:sync` incrementale senza errori.

## 9. Documentazione

- `CLAUDE.md`: Latest; § Architecture (DB, sync); § Data & Integrations (Turso locale).
- `AGENTS.md`: § Database (Drizzle, migrazioni, batch, FK, tempo), § Sync (cursore, full vs
  incremental), § Commands (`db:*`, `hevy:sync`).
- `doc/guide/sync.md` (≤ 200 righe): il modello dei dati, il cursore, la storia degli slot, come si
  rifà un full sync, come si legge `local.db`.
- `README.md`: § Sincronizzare.

## 10. Fuori ambito, dichiarato

- Aggregazione e finestra di revisione → **04**.
- Route del webhook e cron → **06** (usano `syncOneWorkout` e `syncWorkouts`).
- `body_measurements`: non sincronizzati (D-8) → CLAUDE.md § Not yet present.
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
  firma (createHevyClient, HevyWorkout, PLAN, SCALES, initialSlotStates, allVariantsInUse,
  ensureTemplates, FOLDER_TITLE)

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/03-turso-sync. Crea SESSION_NOTES.md.

Implementa la spec 03 esattamente come scritta: § 4.2 è lo schema (nomi di tabelle e colonne
sono contratti), § 4.3–4.5 il codice, § 6 i test (DB in memoria con le migrazioni vere),
§ 7 il collaudo fase per fase con l'esito atteso prima (le chiamate reali sono sole GET, le
lanci dopo il mio OK), § 9 la documentazione. Se un punto non è coperto dalla spec, fermati e
chiedimelo con AskUserQuestion. Le etichette della serie non entrano nel codice.

Alla fine: riassumi il diff, aggiorna la riga «Stato» della spec, § 5 dell'indice e
CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
