# 01 — Scaffold e client Hevy

> **Stato**: scritta 2026-10-23. Pronta per l'implementazione dopo la lettura adversariale.
> **Branch**: `feature/01-scaffold-hevy-client` da `main`. **Dipende da**: niente (è la prima).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-9, D-10, D-21, D-23; scelte T-2, T-3,
> T-4, T-5, T-6; contratti C-1, C-2, C-15 (li crea).

## 1. Il problema oggi (prove)

1. Il repo contiene solo la documentazione (CLAUDE.md, AGENTS.md, WORKFLOW.md, COMMENTS.md,
   DEVELOPMENT_GUIDELINES.md, README.md), le spec e le fixture `tests/fixtures/hevy/*.json`. Non c'è
   `package.json`, non c'è codice.
2. L'API Hevy ha una forma precisa e alcune trappole, verificate il 2026-10-23 sullo spec OpenAPI
   (`https://api.hevyapp.com/docs/`, estratto da `swagger-ui-init.js`) e con chiamate GET reali:
   - header `api-key`; `pageSize` massimo **10** su `routines`, `routine_folders`, `workouts`,
     `workouts/events` e **100** su `exercise_templates` (una richiesta con `pageSize` oltre il massimo
     risponde 400 «Invalid page size»);
   - `PUT /v1/routines/{id}` sostituisce tutta la routine; 404 se non è dell'utente; nessun DELETE;
   - `POST /v1/exercise_templates` risponde `{ id }`: lo schema dichiara `integer`, ma gli id dei
     custom reali sono UUID (fixture `exercise_templates_page.json`, righe `is_custom: true`);
   - gli id del catalogo sono stringhe di 8 caratteri (`392887AA` Push Up);
   - `GET /v1/exercise_history/{id}` non è paginata; `workouts/events` porta `updated` e `deleted`;
   - le date sono ISO 8601 con offset (`2026-10-22T17:31:47+00:00`) o `Z` (`updated_at`).
   - La descrizione dell'API chiede di non chiamare esattamente a xx:00: il cron (06) lo rispetta;
     il client non deve fare nulla.
3. Nessun rate limit documentato; le risposte reali non portano header di limite. Il client tratta
   429 e 5xx come errori ritentabili (un solo retry dopo 2 s) e tutto il resto come definitivo.

## 2. Obiettivo e criteri di accettazione

Un progetto Next 16 che compila, con il client Hevy tipizzato, validato con zod e testato sulle
fixture, e con gli script di qualità che valgono per tutta la serie.

| # | Criterio | Prova |
|---|---|---|
| A1 | `pnpm install` da clone pulito, poi `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` sono verdi. | CI locale eseguita nel collaudo, esito in CLAUDE.md → Latest. |
| A2 | `createHevyClient` (C-1) espone tutti i metodi della tabella C-1 con le firme scritte in § 4.3; ogni metodo valida la risposta con lo schema zod di C-2 e lancia `HevyApiError` su status ≠ 2xx. | `src/lib/hevy/client.test.ts`: un test per metodo con `fetch` finto che serve la fixture; mutazione: togliere `rep_range` dallo schema `HevyRoutineSet` fa fallire il test di `getRoutine`. |
| A3 | `listExerciseTemplates()` e `listRoutines()` percorrono tutte le pagine (`page_count`) con il `pageSize` massimo dell'endpoint (100 e 10). | Test con `fetch` finto che conta le chiamate: 3 pagine di template → 3 chiamate con `pageSize=100`; mutazione: `pageSize=10` sui template fa fallire il test. |
| A4 | `createCustomExercise` restituisce `{ id: string }` qualunque sia il tipo JSON di `id` (stringa o numero). | Test con fixture `custom_exercise_created.json` e con `{ "id": 123 }`. |
| A5 | 429 e 5xx: un retry dopo 2 s, poi `HevyApiError`; 4xx diversi: nessun retry. | Test con `fetch` finto che risponde 500 poi 200 (successo, 2 chiamate) e 404 (1 chiamata). |
| A6 | `env` (C-15) rifiuta l'avvio con un messaggio che nomina la variabile mancante. | `src/lib/env.test.ts`: `HEVY_API_KEY` assente → `throws /HEVY_API_KEY/`. |
| A7 | `scripts/hevy-whoami.ts` stampa `username`, `is_pro` e il numero di workout dell'account usando `HEVY_API_KEY` da `.env.local`. | Collaudo § 7 (una GET reale). |

## 3. Decisioni

- **D-10** (stack segnaprezzi), **D-21** (documenti in inglese), **D-23** (`main`, PR per spec).
- **T-2** (versioni pinnate il 2026-10-23), **T-3** (`@libsql/client` entry Node: non si usa qui ma
  si installa nella 03), **T-4** (Turbopack in dev, `next build --webpack`; `cacheComponents` off),
  **T-5** (layout `src/`), **T-6** (script di package.json).
- Adottate qui, ribaltabili: (a) il client riceve `fetch` per iniezione (default `globalThis.fetch`)
  così i test non toccano la rete; (b) `baseUrl` default `https://api.hevyapp.com`; (c) le date
  restano **stringhe ISO** nei tipi Hevy: la conversione in epoch ms è della 03; (d) il retry è
  un'attesa fissa di 2 s, senza jitter; (e) nessun client HTTP di terze parti.

## 4. La soluzione

### 4.1 Scaffold

```bash
corepack enable && corepack prepare pnpm@11.22.0 --activate
pnpm dlx create-next-app@16.4.2 . --ts --tailwind --app --src-dir --import-alias "@/*" --use-pnpm --no-eslint --skip-install
```

Poi si sostituiscono i file generati con quelli di § 4.2 e si installa:

```bash
pnpm add next@16.4.2 react@19.3.1 react-dom@19.3.1 zod@4.6.7
pnpm add -D typescript@7.0.2 @types/node@22 @types/react@19 @types/react-dom@19 @biomejs/biome@2.6.2 vitest@5.0.5 tsx@4.23.15 dotenv@18.0.9 tailwindcss@4.3.3 @tailwindcss/postcss@4.3.3
```

`package.json` (le parti che contano; il resto è quello generato):

```json
{
  "name": "hevy-coach",
  "private": true,
  "packageManager": "pnpm@11.22.0",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "next dev",
    "build": "next build --webpack",
    "start": "next start",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "hevy:whoami": "tsx --env-file-if-exists=.env.local scripts/hevy-whoami.ts"
  }
}
```

Nota: `next build --webpack` fin da ora (T-4), così la 07 non cambia il comando. Fino alla 07
l'avviso di Next «webpack config found» non compare perché non c'è plugin.

### 4.2 File di configurazione

- `biome.json`: copia di `segnaprezzi/biome.json` (schema 2.6, `lineWidth: 100`, `quoteStyle:
  'single'`, `semicolons: 'always'`, domini `next`, `react`, `test`, `vcs.useIgnoreFile: true`).
- `tsconfig.json`: quello generato, più `"strict": true`, `"noUncheckedIndexedAccess": true`,
  `"paths": { "@/*": ["./src/*"] }`, `"types": ["vitest/globals"]` NON si aggiunge (si importa da
  vitest nei test).
- `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';
import path from 'node:path';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.test.ts', 'scripts/**/*.test.ts'],
    env: { TZ: 'Europe/Rome' },
  },
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
});
```

- `next.config.ts`:

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Why: `next dev` otherwise appends a generated block to AGENTS.md on every run; this repo's
  // AGENTS.md is a hand-maintained contract.
  agentRules: false,
};

export default nextConfig;
```

- `.env.example` (committato; `.env.local` mai):

```bash
# Copy to .env.local and fill in. Never commit .env.local (.gitignore).
# Hevy — Pro account; create the key at https://hevy.com/settings?developer
HEVY_API_KEY=""
```

- `src/app/layout.tsx` e `src/app/page.tsx`: pagina minima «Hevy Coach — in costruzione» (lang
  `it`, nessun componente); la 05a la sostituisce.

### 4.3 Il client Hevy — `src/lib/hevy/`

`src/lib/hevy/schemas.ts` (C-2), zod 4, tutti gli oggetti con `.passthrough()` NON usato: gli
schemi sono `z.object({...})` normali (zod 4 ignora le chiavi sconosciute di default), così un campo
nuovo di Hevy non rompe il parse:

```ts
import { z } from 'zod';

const nullableNumber = z.number().nullable();
const nullableString = z.string().nullable();

export const HevyUserInfoSchema = z.object({
  id: z.string(),
  username: z.string(),
  name: z.string(),
  url: z.string(),
  weight_unit: z.enum(['kg', 'lbs']),
  distance_unit: z.enum(['kilometers', 'miles']),
  is_pro: z.boolean(),
});

export const HevyExerciseTemplateSchema = z.object({
  id: z.string(),
  title: z.string(),
  type: z.string(), // weight_reps | reps_only | duration | bodyweight_weighted | ... (open set)
  primary_muscle_group: z.string(),
  secondary_muscle_groups: z.array(z.string()),
  equipment: z.string(),
  is_custom: z.boolean(),
  thumbnail_url: nullableString.optional(),
});

export const HevyRoutineFolderSchema = z.object({
  id: z.number(),
  index: z.number(),
  title: z.string(),
  updated_at: z.string(),
  created_at: z.string(),
});

export const RepRangeSchema = z.object({ start: nullableNumber, end: nullableNumber });

export const HevyRoutineSetSchema = z.object({
  index: z.number(),
  type: z.enum(['warmup', 'normal', 'failure', 'dropset']),
  weight_kg: nullableNumber,
  reps: nullableNumber,
  rep_range: RepRangeSchema.nullable().optional(),
  distance_meters: nullableNumber,
  duration_seconds: nullableNumber,
  custom_metric: nullableNumber,
});

export const HevyRoutineExerciseSchema = z.object({
  index: z.number(),
  title: z.string(),
  rest_seconds: nullableNumber,
  notes: nullableString,
  exercise_template_id: z.string(),
  superset_id: nullableNumber,
  sets: z.array(HevyRoutineSetSchema),
});

export const HevyRoutineSchema = z.object({
  id: z.string(),
  title: z.string(),
  folder_id: nullableNumber,
  updated_at: z.string(),
  created_at: z.string(),
  exercises: z.array(HevyRoutineExerciseSchema),
});

export const HevyWorkoutSetSchema = z.object({
  index: z.number(),
  type: z.enum(['warmup', 'normal', 'failure', 'dropset']),
  weight_kg: nullableNumber,
  reps: nullableNumber,
  distance_meters: nullableNumber,
  duration_seconds: nullableNumber,
  rpe: nullableNumber,
  custom_metric: nullableNumber,
});

export const HevyWorkoutExerciseSchema = z.object({
  index: z.number(),
  title: z.string(),
  notes: nullableString,
  exercise_template_id: z.string(),
  superset_id: nullableNumber,
  sets: z.array(HevyWorkoutSetSchema),
});

export const HevyWorkoutSchema = z.object({
  id: z.string(),
  title: z.string(),
  routine_id: nullableString,
  description: nullableString,
  start_time: z.string(),
  end_time: z.string(),
  updated_at: z.string(),
  created_at: z.string(),
  exercises: z.array(HevyWorkoutExerciseSchema),
});

export const HevyWorkoutEventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('updated'), workout: HevyWorkoutSchema }),
  z.object({ type: z.literal('deleted'), id: z.string(), deleted_at: z.string() }),
]);

export const HevyExerciseHistoryEntrySchema = z.object({
  workout_id: z.string(),
  workout_title: z.string(),
  workout_start_time: z.string(),
  workout_end_time: z.string(),
  exercise_template_id: z.string(),
  weight_kg: nullableNumber,
  reps: nullableNumber,
  distance_meters: nullableNumber,
  duration_seconds: nullableNumber,
  rpe: nullableNumber,
  custom_metric: nullableNumber,
  set_type: z.string(),
});

// Paginated envelopes
export const PagedSchema = <T extends z.ZodTypeAny>(key: string, item: T) =>
  z.object({ page: z.number(), page_count: z.number(), [key]: z.array(item) });

// Single-routine responses: the OpenAPI spec says { routine }, unverified on 2026-10-23 (no routine
// existed); accept both the envelope and the bare object and always return the bare routine.
export const RoutineEnvelopeSchema = z.union([
  z.object({ routine: HevyRoutineSchema }).transform((r) => r.routine),
  HevyRoutineSchema,
]);

// Write bodies (what we send)
export const RoutineWriteSetSchema = z.object({
  type: z.enum(['warmup', 'normal', 'failure', 'dropset']),
  weight_kg: nullableNumber,
  reps: nullableNumber,
  distance_meters: nullableNumber,
  duration_seconds: nullableNumber,
  custom_metric: nullableNumber,
  rep_range: RepRangeSchema.nullable(),
});
export const RoutineWriteExerciseSchema = z.object({
  exercise_template_id: z.string(),
  superset_id: nullableNumber,
  rest_seconds: nullableNumber,
  notes: nullableString,
  sets: z.array(RoutineWriteSetSchema),
});
export const RoutineWriteBodySchema = z.object({
  routine: z.object({
    title: z.string(),
    folder_id: nullableNumber,
    notes: z.string(),
    exercises: z.array(RoutineWriteExerciseSchema),
  }),
});

export const CustomExerciseInputSchema = z.object({
  title: z.string().min(1),
  exercise_type: z.enum(['reps_only', 'duration', 'bodyweight_reps', 'weight_reps']),
  equipment_category: z.enum(['none', 'other']),
  muscle_group: z.string(),
  other_muscles: z.array(z.string()),
});
```

`src/lib/hevy/types.ts`: `export type HevyUserInfo = z.infer<typeof HevyUserInfoSchema>` e così
via per ogni schema (`HevyExerciseTemplate`, `HevyRoutineFolder`, `HevyRoutine`,
`HevyRoutineExercise`, `HevyRoutineSet`, `HevyWorkout`, `HevyWorkoutExercise`, `HevyWorkoutSet`,
`HevyWorkoutEvent`, `HevyExerciseHistoryEntry`, `RoutineWriteBody`, `CustomExerciseInput`).

`src/lib/hevy/client.ts` (C-1):

```ts
/**
 * Hevy public API client (https://api.hevyapp.com/docs/, OpenAPI 0.0.1 read 2026-10-23).
 *
 * Design: a plain function over `fetch` with every response validated by zod, because Hevy
 * promises no stability ("we make no guarantees that we won't completely change the structure").
 * Pagination is hidden behind list* methods: the page size maximum differs per endpoint (10 for
 * routines/folders/workouts/events, 100 for exercise templates) and a larger value is a 400.
 */
import { z } from 'zod';
import * as S from './schemas';
import type * as T from './types';

export class HevyApiError extends Error {
  constructor(
    readonly status: number,
    readonly code: string,
    readonly body: unknown,
    message?: string,
  ) {
    super(message ?? `Hevy API ${status} ${code}`);
    this.name = 'HevyApiError';
  }
}

export interface HevyClientOptions {
  apiKey: string;
  fetch?: typeof globalThis.fetch;
  baseUrl?: string;
  /** Milliseconds to wait before the single retry on 429/5xx. Tests pass 0. */
  retryDelayMs?: number;
}

const PAGE_SIZE = { routines: 10, routineFolders: 10, workouts: 10, workoutEvents: 10, exerciseTemplates: 100 } as const;

export function createHevyClient(options: HevyClientOptions) {
  const fetchFn = options.fetch ?? globalThis.fetch;
  const baseUrl = options.baseUrl ?? 'https://api.hevyapp.com';
  const retryDelayMs = options.retryDelayMs ?? 2000;

  async function request<T>(method: 'GET' | 'POST' | 'PUT', path: string, schema: z.ZodType<T>, init?: { query?: Record<string, string | number>; body?: unknown }): Promise<T> {
    const url = new URL(path, baseUrl);
    for (const [key, value] of Object.entries(init?.query ?? {})) url.searchParams.set(key, String(value));
    const doFetch = () =>
      fetchFn(url, {
        method,
        headers: { 'api-key': options.apiKey, accept: 'application/json', ...(init?.body ? { 'content-type': 'application/json' } : {}) },
        body: init?.body ? JSON.stringify(init.body) : undefined,
      });
    let response = await doFetch();
    // One retry on 429/5xx: Hevy documents no rate limit; a second failure is reported as is.
    if (response.status === 429 || response.status >= 500) {
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs));
      response = await doFetch();
    }
    const text = await response.text();
    const json: unknown = text ? safeJson(text) : null;
    if (!response.ok) {
      const code = typeof json === 'object' && json !== null && 'error' in json ? String((json as { error: unknown }).error) : response.statusText;
      throw new HevyApiError(response.status, code, json);
    }
    return schema.parse(json);
  }

  async function listAll<T>(path: string, key: string, item: z.ZodType<T>, pageSize: number, query: Record<string, string> = {}): Promise<T[]> {
    const items: T[] = [];
    let page = 1;
    let pageCount = 1;
    do {
      const envelope = await request('GET', path, S.PagedSchema(key, item), { query: { ...query, page, pageSize } });
      items.push(...(envelope[key] as T[]));
      pageCount = envelope.page_count;
      page += 1;
    } while (page <= pageCount);
    return items;
  }

  return {
    getUserInfo: () => request('GET', '/v1/user/info', z.object({ data: S.HevyUserInfoSchema })).then((r) => r.data),
    listExerciseTemplates: () => listAll('/v1/exercise_templates', 'exercise_templates', S.HevyExerciseTemplateSchema, PAGE_SIZE.exerciseTemplates),
    getExerciseTemplate: (id: string) => request('GET', `/v1/exercise_templates/${id}`, S.HevyExerciseTemplateSchema),
    createCustomExercise: (input: T.CustomExerciseInput) =>
      request('POST', '/v1/exercise_templates', z.object({ id: z.union([z.string(), z.number()]) }), { body: { exercise: input } }).then((r) => ({ id: String(r.id) })),
    listRoutineFolders: () => listAll('/v1/routine_folders', 'routine_folders', S.HevyRoutineFolderSchema, PAGE_SIZE.routineFolders),
    createRoutineFolder: (title: string) => request('POST', '/v1/routine_folders', S.HevyRoutineFolderSchema, { body: { routine_folder: { title } } }),
    listRoutines: () => listAll('/v1/routines', 'routines', S.HevyRoutineSchema, PAGE_SIZE.routines),
    getRoutine: (id: string) => request('GET', `/v1/routines/${id}`, S.RoutineEnvelopeSchema),
    createRoutine: (body: T.RoutineWriteBody) => request('POST', '/v1/routines', S.RoutineEnvelopeSchema, { body }),
    updateRoutine: (id: string, body: T.RoutineWriteBody) => request('PUT', `/v1/routines/${id}`, S.RoutineEnvelopeSchema, { body }),
    countWorkouts: () => request('GET', '/v1/workouts/count', z.object({ workout_count: z.number() })).then((r) => r.workout_count),
    listWorkouts: (page: number) => request('GET', '/v1/workouts', S.PagedSchema('workouts', S.HevyWorkoutSchema), { query: { page, pageSize: PAGE_SIZE.workouts } }),
    // Verified 2026-10-23: the single-workout endpoint returns the bare workout, not { workout }.
    getWorkout: (id: string) => request('GET', `/v1/workouts/${id}`, S.HevyWorkoutSchema),
    listWorkoutEvents: (params: { since: string; page: number }) =>
      request('GET', '/v1/workouts/events', S.PagedSchema('events', S.HevyWorkoutEventSchema), { query: { since: params.since, page: params.page, pageSize: PAGE_SIZE.workoutEvents } }),
    getExerciseHistory: (templateId: string, range: { startDate: string; endDate: string }) =>
      request('GET', `/v1/exercise_history/${templateId}`, z.object({ exercise_history: z.array(S.HevyExerciseHistoryEntrySchema) }), { query: { start_date: range.startDate, end_date: range.endDate } }).then((r) => r.exercise_history),
  };
}

export type HevyClient = ReturnType<typeof createHevyClient>;

function safeJson(text: string): unknown {
  try {
    return JSON.parse(text);
  } catch {
    return { _raw: text };
  }
}
```

**Buste delle risposte singole** (verificate con GET reali il 2026-10-23): `GET /v1/workouts/{id}`
e `GET /v1/exercise_templates/{id}` restituiscono l'oggetto **nudo** (lo spec OpenAPI non lo
mostra); `GET /v1/routines/{id}`, `POST /v1/routines` e `PUT /v1/routines/{id}` sono dichiarati
`{ routine }` nello spec ma non erano verificabili (l'account non aveva routine). Quindi:
`getWorkout` usa `HevyWorkoutSchema` nudo; i tre metodi delle routine usano
`RoutineEnvelopeSchema = z.union([z.object({ routine: HevyRoutineSchema }).transform((r) => r.routine), HevyRoutineSchema])`,
che accetta entrambe le forme. Il collaudo della 02 annota in AGENTS.md § Hevy quale delle due
arriva davvero.

`src/lib/env.ts` (C-15), server-only:

```ts
import 'server-only';
import { z } from 'zod';

const EnvSchema = z.object({
  HEVY_API_KEY: z.string().min(1, 'HEVY_API_KEY is required'),
});

export const env = EnvSchema.parse(process.env);
```

(`server-only` è un pacchetto di Next: `pnpm add server-only`.) Gli script CLI non passano da
`env.ts` (importa `server-only`): leggono `process.env` con lo stesso schema esportato come
`EnvSchema` da `src/lib/env-schema.ts`, che `env.ts` ri-esporta. Regola fissa per tutta la serie:
**`src/lib/env-schema.ts` è lo schema, `src/lib/env.ts` è l'istanza server-only.**

### 4.4 Script — `scripts/hevy-whoami.ts`

```ts
import { createHevyClient } from '@/lib/hevy/client';
import { EnvSchema } from '@/lib/env-schema';

const env = EnvSchema.parse(process.env);
const hevy = createHevyClient({ apiKey: env.HEVY_API_KEY });
const [user, count] = await Promise.all([hevy.getUserInfo(), hevy.countWorkouts()]);
console.log(`${user.username} · pro=${user.is_pro} · workouts=${count}`);
```

(`tsx` risolve l'alias `@/` leggendo `tsconfig.json` `paths`.)

### 4.5 Fixture e harness dei test

Le fixture sono già in `tests/fixtures/hevy/` (README nella cartella). `tests/helpers/fake-fetch.ts`:

```ts
import { readFileSync } from 'node:fs';
import path from 'node:path';

export function fixture(name: string): unknown {
  return JSON.parse(readFileSync(path.join(process.cwd(), 'tests/fixtures/hevy', name), 'utf8'));
}

/** A fetch stub answering from a route table; records every call for assertions. */
export function fakeFetch(routes: Array<{ match: (url: URL, init?: RequestInit) => boolean; status?: number; body: unknown }>) {
  const calls: Array<{ url: URL; init?: RequestInit }> = [];
  const fn = (async (input: RequestInfo | URL, init?: RequestInit) => {
    const url = new URL(String(input));
    calls.push({ url, init });
    const route = routes.find((r) => r.match(url, init));
    if (!route) return new Response('not found', { status: 404 });
    return new Response(JSON.stringify(route.body), { status: route.status ?? 200, headers: { 'content-type': 'application/json' } });
  }) as typeof fetch;
  return { fn, calls };
}
```

## 5. Vincoli e trappole

- **AGENTS.md § Layers** — qui vuol dire che il client non sa niente del piano né di Turso: riceve
  stringhe e restituisce tipi Hevy; la traduzione sta nella 02 e nella 03.
- **AGENTS.md § Secrets** — qui vuol dire che `HEVY_API_KEY` esiste solo in `.env.local` e
  nell'ambiente; `.env.example` ha la chiave vuota; nessun test legge `.env.local`.
- **AGENTS.md § Hevy** (nasce con questa spec, § 9) — pageSize massimo per endpoint; PUT = tutto;
  niente DELETE; id custom UUID; non chiamare a xx:00.
- `next build --webpack` senza plugin webpack funziona; se Next 16.4 stampasse un avviso, si lascia
  (la 07 lo rende necessario).
- `tsx --env-file-if-exists=.env.local` carica le variabili per gli script; Vitest non carica
  `.env.local` (i test non devono dipenderne).
- Windows: Biome legge `.gitignore` (`vcs.useIgnoreFile`); i file si salvano con LF (`.gitattributes`
  con `* text=auto eol=lf` fa parte di questa spec: segnaprezzi ha avuto `pnpm lint` rosso per CRLF).

## 6. Test

| Livello | Test | Mutazione che lo prova portante |
|---|---|---|
| Unit | `schemas.test.ts`: ogni fixture di `tests/fixtures/hevy/` passa il suo schema (`user_info`, `exercise_templates_page`, `workout`, `workouts_page`, `workout_events`, `exercise_history`, `routine_created`, `routine_folder_created`, `routines_page_empty`, `routine_folders_page_empty`). | Rendere `rpe` non-nullable in `HevyWorkoutSetSchema` → `workout.json` fallisce. |
| Unit | `client.test.ts`: `listExerciseTemplates` su 3 pagine finte → 3 chiamate, `pageSize=100`, 2·100+8 elementi; `listRoutines` con `routines_page_empty.json` → `[]` e 1 chiamata. | `pageSize` di `exerciseTemplates` a 10 → asserzione sulla query fallisce. |
| Unit | `client.test.ts`: `createCustomExercise` con `{ id: 123 }` e con la fixture → `{ id: '123' }` / `{ id: 'aaaaaaaa-…' }`. | Togliere `String(r.id)` → il test con 123 fallisce. |
| Unit | `client.test.ts`: 500 poi 200 → risultato e 2 chiamate (`retryDelayMs: 0`); 404 → `HevyApiError` con `status 404`, `code` = `error` del body (`error_404_routine.json`), 1 chiamata. | Ritentare anche su 404 → il conteggio delle chiamate fallisce. |
| Unit | `client.test.ts`: `updateRoutine` manda `PUT` con `content-type: application/json` e il body intero (`RoutineWriteBody`). | Omettere il body → `JSON.parse(init.body)` fallisce. |
| Unit | `client.test.ts`: `getRoutine` accetta sia `routine_created.json` (busta `{ routine }`) sia il suo contenuto nudo e restituisce in entrambi i casi la routine con `id` `44444444-…`; `getWorkout` con `workout.json` nudo. | Togliere il ramo nudo da `RoutineEnvelopeSchema` → il secondo caso fallisce. |
| Unit | `env.test.ts`: `EnvSchema.parse({})` lancia con messaggio che contiene `HEVY_API_KEY`. | Rendere la chiave `optional()` → nessun errore. |

Nessun test di integrazione né E2E in questa spec (non c'è UI).

## 7. Collaudo guidato (WORKFLOW.md § 2; fasi che hanno senso qui)

- **A — Invarianza**: niente da preservare (repo nuovo). Si dichiara.
- **C — Comportamento nuovo**: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` verdi.
  Esito atteso: 4 comandi verdi, N test (il numero lo scrive chi implementa nel commit).
- **D — Sotto la UI**: `pnpm hevy:whoami` con la chiave in `.env.local`. **Una GET reale** (lettura,
  nessun effetto). Esito atteso: una riga `<username> · pro=true · workouts=<n ≥ 174>`.
- **F — Giro guidato**: non c'è nulla da guardare. Si dichiara.
- **G — Ripristino**: niente da ripristinare. Nessuna fixture piantata.

Che cosa NON copre: le scritture su Hevy (02), la rete reale nei test (mai).

## 8. Messa in esercizio

- Variabili d'ambiente: solo `HEVY_API_KEY` in `.env.local` (Development). Nessun Vercel, nessun
  Turso (06).
- DDL: nessuno.
- Misura di chiusura: `pnpm hevy:whoami` stampa la riga attesa. In esercizio = PR mergiata su `main`
  + collaudo annotato in CLAUDE.md (regola 5 dell'indice).

## 9. Documentazione

- `CLAUDE.md`: § Current Status → Latest (una riga: scaffold e client, dove sono, cosa viene dopo);
  § Current Status → Stack: via «nothing installed yet», e le versioni pinnate si leggono in
  `package.json`; § Architecture: la riga del client Hevy; § Not yet present: la riga «Any code,
  `package.json`, migrations» diventa «Migrations: born in spec 03», il resto resta com'è.
- `AGENTS.md`: la sezione **§ Hevy** (già scritta al bootstrap) si completa con il retry e le buste
  delle risposte singole; **§ Commands** con gli script reali; **§ Testing** (fake-fetch, fixture,
  `TZ`); nell'introduzione via «Written on 2026-10-23, before any code» (da qui il codice c'è).
- `README.md`: § Setup (pnpm, `.env.local`, `pnpm hevy:whoami`).
- `WORKFLOW.md` § 3: la tabella dei comandi diventa vera (`typecheck`, `lint`, `test`, `build`) e la
  nota «State on 2026-10-23 … There is no code yet» si toglie (le colonne «Since» dicono già il resto).
- Limiti: CLAUDE.md resta sotto 20.000 caratteri; AGENTS.md sotto 300 righe dopo questa spec.

## 10. Fuori ambito, dichiarato

- Scritture su Hevy, cartella, routine, template custom → **02**.
- Persistenza, Turso, conversione delle date → **03**.
- `POST /v1/workouts`, `PUT /v1/workouts/{id}`, body measurements: non servono (l'app non registra
  allenamenti, D-8) → CLAUDE.md § Not yet present («Hevy: write endpoints for workouts and body
  measurements are not wrapped»).
- Rate limit reale di Hevy: sconosciuto; se un 429 compare in esercizio, la regola di retry si
  rivede → CLAUDE.md § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **M** (scaffold + 1 modulo + test).

```
Ciao, in questa sessione implemento la spec doc/v1/01-scaffold-e-client-hevy.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md (regole di sessione e collaudo; § 3 dice come si applicano qui)
- Leggi AGENTS.md (CLAUDE.md è già nel contesto)
- Leggi COMMENTS.md e DEVELOPMENT_GUIDELINES.md: il codice che scrivi li segue
- Leggi doc/v1/00-INDICE.md per intero (decisioni, scelte T-N, contratti C-1, C-2, C-15, regole
  della serie) e poi la spec 01 INTERA
- Leggi tests/fixtures/hevy/README.md e le fixture che la spec cita

Prima della prima modifica: `git branch --show-current` deve dire main; crea
feature/01-scaffold-hevy-client. Crea SESSION_NOTES.md.

Implementa la spec 01 esattamente come scritta: § 4 è il codice, § 6 i test con le mutazioni
che li provano portanti, § 7 il collaudo (fase per fase, esito atteso prima; la GET reale la
lanci dopo il mio OK), § 9 la documentazione. Non decidere nulla che la spec ha già deciso;
se trovi un punto che la spec non copre, fermati e chiedimelo con AskUserQuestion. Le
etichette della serie (D-N, T-N, C-N, «spec 01») non entrano nel codice.

Alla fine: riassumi il diff, scrivi lo stato nella riga «Stato» della spec e in § 5
dell'indice, aggiorna CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
