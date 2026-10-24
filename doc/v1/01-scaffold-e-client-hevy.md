# 01 — Scaffold e client Hevy

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 28 rilievi, 2 decisioni
> (D-36, D-39) e la scelta T-23. Pronta per l'implementazione.
> **Branch**: `feature/01-scaffold-hevy-client` da `main`. **Dipende da**: niente (è la prima).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-1, D-9, D-10, D-21, D-23, D-36, D-39;
> scelte T-2, T-3, T-4, T-5, T-6, T-23; contratti C-1, C-2, C-15 (li crea).

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
3. Nessun rate limit documentato; le risposte reali non portano header di limite. Il client ritenta
   una volta dopo 2 s su 429 (qualunque metodo) e su 5xx solo per GET e PUT (D-39): una POST con
   5xx potrebbe essere già stata eseguita da Hevy, e un custom o una routine doppione non si
   cancellano via API. Tutto il resto è definitivo. Ogni tentativo ha un timeout di 30 s (T-23).

## 2. Obiettivo e criteri di accettazione

Un progetto Next 16 che compila, con il client Hevy tipizzato, validato con zod e testato sulle
fixture, e con gli script di qualità che valgono per tutta la serie.

| # | Criterio | Prova |
|---|---|---|
| A1 | `pnpm install` da clone pulito, poi `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` sono verdi. | CI locale eseguita nel collaudo, esito in CLAUDE.md → Latest. |
| A2 | `createHevyClient` (C-1) espone tutti i metodi della tabella C-1 con le firme scritte in § 4.3; ogni metodo valida la risposta con lo schema zod di C-2, manda header, query e body esatti e lancia `HevyApiError` su status ≠ 2xx. | `src/lib/hevy/client.test.ts`: un test per ognuno dei 15 metodi con `fakeFetch` che serve la fixture e registra la chiamata (§ 6). Mutazione: togliere `rep_range` da `HevyRoutineSetSchema` → il `toEqual` profondo di `getRoutine` va rosso; togliere l'header `api-key` → rosso. |
| A3 | `listExerciseTemplates()` e `listRoutines()` percorrono tutte le pagine (`page_count`) con il `pageSize` massimo dell'endpoint (100 e 10); anche `listRoutineFolders`, `listWorkouts` e `listWorkoutEvents` mandano `pageSize=10`. | Test con `fakeFetch` che conta le chiamate e legge la query: 3 pagine di template → 3 chiamate con `pageSize=100`; 2 pagine di routine → 2 chiamate con `pageSize=10`. Mutazione: `PAGE_SIZE.exerciseTemplates = 10` → rosso; `PAGE_SIZE.routines = 100` → rosso. |
| A4 | `createCustomExercise` restituisce `{ id: string }` qualunque sia il tipo JSON di `id` (stringa o numero). | Test con fixture `custom_exercise_created.json` e con `{ "id": 123 }`. |
| A5 | Retry (D-39): 429 su qualunque metodo e 5xx su GET e PUT → un retry dopo 2 s, poi `HevyApiError`; 5xx su POST e 4xx diversi da 429 → nessun retry. Timeout (T-23): ogni tentativo ha `AbortSignal.timeout(timeoutMs)` (default 30 000); timeout o errore di rete → `HevyApiError { status: 0, code: 'network' }` senza retry. | Test con `fakeFetch`: GET 500 poi 200 → risultato, 2 chiamate; GET 429 poi 200 → 2 chiamate; GET 500 due volte → `HevyApiError` 500, 2 chiamate; POST 500 → `HevyApiError` 500, 1 chiamata; 404 → 1 chiamata; `fetch` che lancia `TypeError` → `HevyApiError` con `status 0`, `code 'network'`, 1 chiamata. Mutazione: ritentare anche le POST → il caso «POST 500» va rosso. |
| A6 | `EnvSchema` (C-15) rifiuta un ambiente senza `HEVY_API_KEY` con un messaggio che nomina la variabile. | `src/lib/env-schema.test.ts` (importa solo `env-schema.ts`, mai `env.ts`): `EnvSchema.parse({})` e `EnvSchema.parse({ HEVY_API_KEY: '' })` lanciano con un messaggio che contiene `HEVY_API_KEY`. |
| A7 | `scripts/hevy-whoami.ts` stampa `username`, `is_pro` e il numero di workout dell'account usando `HEVY_API_KEY` da `.env.local`. | Collaudo § 7 (due GET reali: `/v1/user/info` e `/v1/workouts/count`, lettura, nessun costo). |

## 3. Decisioni

- **D-10** (stack segnaprezzi), **D-21** (documenti in inglese), **D-23** (`main`, PR per spec),
  **D-36** (`env.ts` server-only; fuori da Next si legge con `EnvSchema.pick(...)`; Vitest ha l'alias
  `server-only` → modulo vuoto e valori finti delle env), **D-39** (retry: 429 su tutto, 5xx solo su
  GET e PUT, mai su POST).
- **T-2** (versioni pinnate), **T-3** (`@libsql/client` entry Node: non si usa qui ma si installa
  nella 03), **T-4** (Turbopack in dev, `next build --webpack`; `cacheComponents` off), **T-5**
  (layout `src/`), **T-6** (script di package.json), **T-23** (timeout di 30 s per tentativo,
  errori di rete → `HevyApiError` con `status 0`).
- Adottate qui, ribaltabili: (a) il client riceve `fetch` per iniezione (default `globalThis.fetch`)
  così i test non toccano la rete; (b) `baseUrl` default `https://api.hevyapp.com`; (c) le date
  restano **stringhe ISO** nei tipi Hevy: la conversione in epoch ms è della 03; (d) il retry è
  un'attesa fissa di 2 s, senza jitter e senza leggere `Retry-After`; (e) nessun client HTTP di
  terze parti; (f) gli schemi zod scartano le chiavi sconosciute (nessun `.strict()`), e il `type`
  dei set è `z.enum` a quattro valori: un valore nuovo di Hevy fa fallire il parse ed è un
  `ZodError`, che la 03 tratta come errore della corsa di sync.

## 4. La soluzione

### 4.1 Scaffold

`create-next-app` si rifiuta di scrivere in una cartella che non è vuota (la root del repo ha già
AGENTS.md, CLAUDE.md, README.md, `doc/`, `tests/`) e di default genera un suo AGENTS.md: lo scaffold
si genera **fuori dal repo**, nella cartella temporanea della sessione, e se ne copiano solo i file
che servono.

```bash
corepack enable && corepack prepare pnpm@11.22.0 --activate
pnpm dlx create-next-app@16.4.2 <scratchpad>/hc-scaffold --ts --tailwind --app --src-dir --import-alias "@/*" --use-pnpm --no-eslint --no-react-compiler --no-agents-md --skip-install --disable-git
```

Dalla cartella generata si copiano nel repo **solo** `package.json`, `tsconfig.json`,
`postcss.config.mjs`, `next-env.d.ts`, `src/app/globals.css` e `src/app/favicon.ico`. **Non** si
copiano `README.md`, `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `public/*.svg`, `next.config.ts`,
`src/app/layout.tsx`, `src/app/page.tsx` (restano quelli del repo o nascono da § 4.2). La cartella
temporanea si cancella. Poi si installa:

```bash
pnpm add next@16.4.2 react@19.3.1 react-dom@19.3.1 zod@4.6.7 server-only@0.0.1
pnpm add -D typescript@7.0.2 @types/node@<versione esatta di oggi, 22.x> @types/react@<esatta, 19.x> @types/react-dom@<esatta, 19.x> @biomejs/biome@2.6.2 vitest@5.0.5 tsx@4.23.15 tailwindcss@4.3.3 @tailwindcss/postcss@4.3.3
```

Le versioni scritte `<esatta>` si leggono con `npm view <pkg> version` il giorno
dell'implementazione e si annotano in T-2 nel commit (T-2). `dotenv` non si installa qui: lo
installa la 03, che è la prima a usarlo (`drizzle.config.ts`).

`.gitignore` (scritto a mano, non quello generato): `node_modules/`, `.next/`, `out/`, `*.tsbuildinfo`,
`next-env.d.ts`, `.env*.local`, `.env.local`, `.vercel/`, `coverage/`, `.DS_Store`. Le spec
successive aggiungono le loro righe (`local.db*` 03, `e2e.db*` 05a, `.env.turso-migrate` 06,
`public/sw*` 07).

`pnpm-workspace.yaml` (committato): pnpm 11 non esegue gli script di build delle dipendenze non
approvati e tratta l'omissione come errore all'install.

```yaml
allowBuilds:
  esbuild: true
  sharp: false
  unrs-resolver: false
```

(Le voci che `pnpm install` elenca come «ignored build scripts» il giorno dell'implementazione si
aggiungono qui, con `true` solo per `esbuild`, che tsx e Vitest richiedono.)

`package.json` (le parti che contano; il resto è quello generato):

```json
{
  "name": "hevy-coach",
  "version": "0.1.0",
  "private": true,
  "license": "AGPL-3.0-only",
  "packageManager": "pnpm@11.22.0",
  "engines": { "node": ">=22.9" },
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

- `biome.json` (il file intero; le fixture sono payload catturati e non si riformattano, perché il
  formatter JSON di Biome compatta gli array e farebbe rosso `pnpm lint`):

```json
{
  "$schema": "https://biomejs.dev/schemas/2.6.2/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": true, "includes": ["**", "!tests/fixtures/hevy/**"] },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2, "lineWidth": 100 },
  "linter": {
    "enabled": true,
    "rules": { "recommended": true },
    "domains": { "next": "recommended", "react": "recommended", "test": "recommended" }
  },
  "assist": { "actions": { "source": { "organizeImports": "on" } } },
  "javascript": { "formatter": { "quoteStyle": "single", "semicolons": "always" } },
  "css": { "parser": { "tailwindDirectives": true } }
}
```

- `tsconfig.json`: quello generato, più `"strict": true`, `"noUncheckedIndexedAccess": true`,
  `"paths": { "@/*": ["./src/*"] }`, `"types": ["vitest/globals"]` NON si aggiunge (si importa da
  vitest nei test).
- `vitest.config.ts`. I test girano in **UTC**, come Vercel: il codice di calendario deve usare
  `Intl` con `Europe/Rome` esplicito, e un'implementazione con l'ora locale va rossa. L'alias
  `server-only` → modulo vuoto e i valori finti delle env servono ai moduli che importano `env.ts`
  (D-36); ogni spec che aggiunge una chiave obbligatoria aggiunge qui il suo valore finto.

```ts
import { defineConfig } from 'vitest/config';
import path from 'node:path';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.test.ts', 'scripts/**/*.test.ts', 'tests/**/*.test.ts'],
    exclude: ['tests/e2e/**', 'node_modules/**'],
    // Tests run in UTC like production on Vercel: calendar code must name Europe/Rome explicitly.
    env: { TZ: 'UTC', HEVY_API_KEY: 'test-hevy-key' },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      // `server-only` throws outside the React Server runtime; tests import env.ts through modules
      // that are legitimately server-only in the app.
      'server-only': path.resolve(__dirname, 'tests/helpers/empty.ts'),
    },
  },
});
```

`tests/helpers/empty.ts` contiene solo `export {};`.

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
  `it`, nessun componente, nessun `next/font/google`: il build non deve toccare la rete); la 05a
  cancella `src/app/page.tsx` con `git rm` e riscrive il layout.

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

// `deleted_at` is not in the OpenAPI `required` list: optional, so one bare event never blocks a sync.
export const HevyWorkoutEventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('updated'), workout: HevyWorkoutSchema }),
  z.object({ type: z.literal('deleted'), id: z.string(), deleted_at: z.string().optional() }),
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

// Paginated envelopes. The key is a literal type so `envelope.workouts` is typed `HevyWorkout[]`,
// not `number | HevyWorkout[]` (a `string` key would widen the object to an index signature).
export const PagedSchema = <K extends string, T extends z.ZodTypeAny>(key: K, item: T) =>
  z
    .object({ page: z.number(), page_count: z.number() })
    .extend({ [key]: z.array(item) } as { [P in K]: z.ZodArray<T> });

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
`HevyWorkoutEvent`, `HevyExerciseHistoryEntry`, `RoutineWriteBody`, `RoutineWriteExercise`,
`RoutineWriteSet`, `CustomExerciseInput`): la 02 costruisce i set con `RoutineWriteSet`.

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
  /** Milliseconds to wait before the single retry. Tests pass 0. */
  retryDelayMs?: number;
  /** Per-attempt timeout (AbortSignal.timeout). Default 30 s: a hung request must not eat the review deadline. */
  timeoutMs?: number;
}

const PAGE_SIZE = { routines: 10, routineFolders: 10, workouts: 10, workoutEvents: 10, exerciseTemplates: 100 } as const;

export function createHevyClient(options: HevyClientOptions) {
  const fetchFn = options.fetch ?? globalThis.fetch;
  const baseUrl = options.baseUrl ?? 'https://api.hevyapp.com';
  const retryDelayMs = options.retryDelayMs ?? 2000;
  const timeoutMs = options.timeoutMs ?? 30_000;

  async function request<T>(method: 'GET' | 'POST' | 'PUT', path: string, schema: z.ZodType<T>, init?: { query?: Record<string, string | number>; body?: unknown }): Promise<T> {
    const url = new URL(path, baseUrl);
    for (const [key, value] of Object.entries(init?.query ?? {})) url.searchParams.set(key, String(value));
    const doFetch = async () => {
      try {
        return await fetchFn(url, {
          method,
          headers: { 'api-key': options.apiKey, accept: 'application/json', ...(init?.body ? { 'content-type': 'application/json' } : {}) },
          body: init?.body ? JSON.stringify(init.body) : undefined,
          signal: AbortSignal.timeout(timeoutMs),
        });
      } catch (error) {
        // Network failure or timeout: never retried (the request may be in flight on Hevy's side).
        throw new HevyApiError(0, 'network', null, error instanceof Error ? error.message : String(error));
      }
    };
    let response = await doFetch();
    // One retry: on 429 for any method (the request was not executed), on 5xx only for GET and PUT
    // (idempotent). A POST that got a 5xx may already have created a routine or a custom exercise,
    // and Hevy has no DELETE: retrying could duplicate something permanent.
    const retryable = response.status === 429 || (response.status >= 500 && method !== 'POST');
    if (retryable) {
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

  async function listAll<K extends string, T extends z.ZodTypeAny>(path: string, key: K, item: T, pageSize: number, query: Record<string, string> = {}): Promise<z.infer<T>[]> {
    const items: z.infer<T>[] = [];
    let page = 1;
    let pageCount = 1;
    do {
      const envelope = await request('GET', path, S.PagedSchema(key, item), { query: { ...query, page, pageSize } });
      items.push(...envelope[key]);
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

I tipi di ritorno che le spec successive usano: `listWorkouts(page): Promise<{ page: number;
page_count: number; workouts: HevyWorkout[] }>` e `listWorkoutEvents({ since, page }): Promise<{
page: number; page_count: number; events: HevyWorkoutEvent[] }>` (grazie al `PagedSchema` tipizzato
per chiave: `r.workouts[0]?.id` compila senza cast, e `pnpm typecheck` lo prova in `client.test.ts`).

**Buste delle risposte singole** (verificate con GET reali il 2026-10-23): `GET /v1/workouts/{id}`
e `GET /v1/exercise_templates/{id}` restituiscono l'oggetto **nudo**, come dichiara lo spec OpenAPI
(`$ref Workout`, `ExerciseTemplate`); `GET /v1/routines/{id}`, `POST /v1/routines` e `PUT /v1/routines/{id}` sono dichiarati
`{ routine }` nello spec ma non erano verificabili (l'account non aveva routine). Quindi:
`getWorkout` usa `HevyWorkoutSchema` nudo; i tre metodi delle routine usano
`RoutineEnvelopeSchema = z.union([z.object({ routine: HevyRoutineSchema }).transform((r) => r.routine), HevyRoutineSchema])`,
che accetta entrambe le forme. Il collaudo della 02 annota in AGENTS.md § Hevy quale delle due
arriva davvero.

Le variabili d'ambiente (C-15) vivono in due file. `src/lib/env-schema.ts` è lo schema, senza
`server-only`:

```ts
import { z } from 'zod';

export const EnvSchema = z.object({
  HEVY_API_KEY: z.string().min(1, 'HEVY_API_KEY is required'),
});
```

`src/lib/env.ts` è l'istanza, server-only (pacchetto `server-only`, già installato in § 4.1):

```ts
import 'server-only';
import { EnvSchema } from './env-schema';

export { EnvSchema };
export const env = EnvSchema.parse(process.env);
```

Regola fissa per tutta la serie (D-36; sta anche in AGENTS.md § 2.3): **`src/lib/env-schema.ts` è
lo schema, `src/lib/env.ts` è l'istanza server-only; ogni modulo che può girare fuori da Next (gli
script in `scripts/`, `src/lib/db/client.ts`, il sender web-push) legge SOLO le chiavi che usa con
`EnvSchema.pick({ HEVY_API_KEY: true }).parse(process.env)` e non importa mai `@/lib/env`.** Così
una chiave obbligatoria aggiunta da una spec successiva (Turso, Anthropic, segreti del cron) non
rende inutilizzabile uno script che non la usa.

### 4.4 Script — `scripts/hevy-whoami.ts`

```ts
import { createHevyClient } from '@/lib/hevy/client';
import { EnvSchema } from '@/lib/env-schema';

async function main(): Promise<void> {
  const env = EnvSchema.pick({ HEVY_API_KEY: true }).parse(process.env);
  const hevy = createHevyClient({ apiKey: env.HEVY_API_KEY });
  const [user, count] = await Promise.all([hevy.getUserInfo(), hevy.countWorkouts()]);
  console.log(`${user.username} · pro=${user.is_pro} · workouts=${count}`);
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

(`tsx` risolve l'alias `@/` leggendo `tsconfig.json` `paths`.) Gli script in `scripts/` non usano
`await` al livello del modulo: il pacchetto è CommonJS (nessun `"type": "module"`) e tsx lo
trasforma in formato `cjs`, dove il top-level await non è ammesso. Ogni script della serie segue
questa forma `main()`.

### 4.5 Fixture e harness dei test

Le fixture sono già in `tests/fixtures/hevy/` (README nella cartella). `tests/helpers/fake-fetch.ts`:

```ts
import { readFileSync } from 'node:fs';
import path from 'node:path';

export function fixture(name: string): unknown {
  return JSON.parse(readFileSync(path.join(process.cwd(), 'tests/fixtures/hevy', name), 'utf8'));
}

export interface FakeRoute {
  match: (url: URL, init?: RequestInit) => boolean;
  status?: number;
  body: unknown;
  /** How many times this route answers before `find` moves on to the next one (default: forever). */
  times?: number;
  /** Throw instead of answering (network failure). */
  throws?: Error;
}

/** A fetch stub answering from a route table; records every call for assertions. */
export function fakeFetch(routes: FakeRoute[]) {
  const calls: Array<{ url: URL; init?: RequestInit }> = [];
  const used = new Map<FakeRoute, number>();
  const fn = (async (input: RequestInfo | URL, init?: RequestInit) => {
    const url = new URL(String(input));
    calls.push({ url, init });
    const route = routes.find((r) => r.match(url, init) && (r.times === undefined || (used.get(r) ?? 0) < r.times));
    if (!route) return new Response('not found', { status: 404 });
    used.set(route, (used.get(route) ?? 0) + 1);
    if (route.throws) throw route.throws;
    return new Response(JSON.stringify(route.body), { status: route.status ?? 200, headers: { 'content-type': 'application/json' } });
  }) as typeof fetch;
  return { fn, calls };
}
```

`fixture` e `fakeFetch` sono l'harness condiviso della serie (C-1): la 02 e la 03 li usano così
come sono.

## 5. Vincoli e trappole

- **AGENTS.md § Layers** — qui vuol dire che il client non sa niente del piano né di Turso: riceve
  stringhe e restituisce tipi Hevy; la traduzione sta nella 02 e nella 03.
- **AGENTS.md § Secrets** — qui vuol dire che `HEVY_API_KEY` esiste solo in `.env.local` e
  nell'ambiente; `.env.example` ha la chiave vuota; nessun test legge `.env.local`.
- **AGENTS.md § 3 Hevy** (scritta al bootstrap) — pageSize massimo per endpoint; PUT = tutto;
  niente DELETE; id custom UUID; non chiamare a xx:00; retry per metodo (D-39).
- **AGENTS.md § 2.3** — qui vuol dire che `env-schema.ts` è lo schema ed `env.ts` l'istanza
  server-only; fuori da Next si usa `EnvSchema.pick(...)` (D-36).
- `next build --webpack` senza plugin webpack funziona senza avvisi (verificato in segnaprezzi con
  next 16.4.2); la 07 aggiunge il plugin.
- `tsx --env-file-if-exists=.env.local` carica le variabili per gli script (Node ≥ 22.9); Vitest non
  carica `.env.local` (i test non devono dipenderne: i valori finti stanno in `vitest.config.ts`).
- Windows: Biome legge `.gitignore` (`vcs.useIgnoreFile`); i file si salvano con LF
  (`.gitattributes` con `* text=auto eol=lf` e le righe `binary` esiste dal bootstrap e non si
  tocca: segnaprezzi ha avuto `pnpm lint` rosso per CRLF).
- Il collaudo D stampa lo username reale dell'owner: in CLAUDE.md e nel commit si annota solo
  «riga attesa stampata, `workouts=<n>`», mai lo username (indice § 7.7).

## 6. Test

| Livello | Test | Mutazione che lo prova portante |
|---|---|---|
| Unit | `schemas.test.ts`: ogni fixture di `tests/fixtures/hevy/` passa il suo schema (`user_info`, `exercise_templates_page`, `workout`, `workouts_page`, `workout_events`, `exercise_history`, `routine_created`, `routine_folder_created`, `routines_page_empty`, `routine_folders_page_empty`). | Rendere `rpe` non-nullable in `HevyWorkoutSetSchema` → `workout.json` fallisce. |
| Unit | `client.test.ts`: per ognuno dei 15 metodi, `fakeFetch` registra la chiamata e il test verifica metodo, percorso, query e body. In particolare: ogni chiamata porta `api-key` uguale alla chiave passata; `createCustomExercise` manda `{ exercise: input }`; `createRoutineFolder('X')` manda `{ routine_folder: { title: 'X' } }`; `listWorkoutEvents({ since: '2026-10-01T00:00:00Z', page: 2 })` manda `since`, `page=2`, `pageSize=10`; `getExerciseHistory('392887AA', …)` manda `start_date` ed `end_date`; `getUserInfo` con `user_info.json` restituisce `data`; `countWorkouts` con `{ "workout_count": 42 }` → 42; `const r = await listWorkouts(1)` usa `r.workouts[0]?.id` senza cast (lo prova `pnpm typecheck`). | Togliere l'header `api-key` → rosso; mandare `input` senza la busta `exercise` → rosso. |
| Unit | `client.test.ts`: `listExerciseTemplates` su 3 pagine finte → 3 chiamate, `pageSize=100`, 2·100+8 elementi; `listRoutines` su 2 pagine finte → 2 chiamate con `pageSize=10`; `listRoutines` con `routines_page_empty.json` → `[]` e 1 chiamata; `listRoutineFolders`, `listWorkouts(1)` e `listWorkoutEvents` → `pageSize=10` nella query. | `PAGE_SIZE.exerciseTemplates` a 10 → rosso; `PAGE_SIZE.routines` a 100 → rosso. |
| Unit | `client.test.ts`: `createCustomExercise` con `{ id: 123 }` e con la fixture → `{ id: '123' }` / `{ id: 'aaaaaaaa-…' }`. | Togliere `String(r.id)` → il test con 123 fallisce. |
| Unit | `client.test.ts` (retry, `retryDelayMs: 0`): rotta GET 500 con `times: 1` seguita da rotta 200 → risultato e 2 chiamate; lo stesso con 429; GET 500 con `times: 2` → `HevyApiError` 500 e 2 chiamate; POST 500 → `HevyApiError` 500 e **1** chiamata; POST 429 poi 201 → 2 chiamate; 404 → `HevyApiError` con `status 404`, `code` = `error` del body (`error_404_routine.json`), 1 chiamata; rotta con `throws: new TypeError('fetch failed')` → `HevyApiError` con `status 0`, `code 'network'`, 1 chiamata. | Ritentare anche le POST su 5xx → il caso «POST 500» va rosso; togliere `status === 429` → il caso 429 va rosso; ritentare anche su 404 → rosso. |
| Unit | `client.test.ts`: `updateRoutine` manda `PUT` con `content-type: application/json` e il body intero (`RoutineWriteBody`). | Omettere il body → `JSON.parse(init.body)` fallisce. |
| Unit | `client.test.ts`: `getRoutine` con `routine_created.json` (busta `{ routine }`) e con il suo contenuto nudo restituisce in entrambi i casi un oggetto `toEqual(fixture('routine_created.json').routine)` (uguaglianza profonda, `rep_range` compreso); `getWorkout` con `workout.json` nudo → `toEqual(fixture('workout.json'))`. | Togliere il ramo nudo da `RoutineEnvelopeSchema` → il secondo caso fallisce; togliere `rep_range` da `HevyRoutineSetSchema` → il `toEqual` fallisce. |
| Unit | `src/lib/env-schema.test.ts`: `EnvSchema.parse({})` e `EnvSchema.parse({ HEVY_API_KEY: '' })` lanciano con un messaggio che contiene `HEVY_API_KEY`; `EnvSchema.pick({ HEVY_API_KEY: true }).parse({ HEVY_API_KEY: 'x' })` passa. | Rendere la chiave `optional()` → nessun errore → rosso. |

Nessun test di integrazione né E2E in questa spec (non c'è UI).

## 7. Collaudo guidato (WORKFLOW.md § 2; fasi che hanno senso qui)

- **A — Invarianza**: niente da preservare (repo nuovo). Si dichiara.
- **C — Comportamento nuovo**: `git clean -xdf -e .env.local && pnpm install && pnpm typecheck &&
  pnpm lint && pnpm test && pnpm build`. Esito atteso, scritto prima di lanciare: install senza
  domande (A1), 4 comandi verdi, tanti test quanti i casi elencati in § 6 (il numero si conta dalla
  tabella e si scrive nel messaggio prima del comando).
- **D — Sotto la UI**: `pnpm hevy:whoami` con la chiave in `.env.local`. **Due GET reali**
  (`/v1/user/info` e `/v1/workouts/count`: lettura, nessun effetto, nessun costo; non a xx:00).
  Esito atteso: una riga `<username> · pro=true · workouts=<n ≥ 174>`; in CLAUDE.md si annota solo
  `workouts=<n>`, mai lo username.
- **E — Casi negativi**: `HEVY_API_KEY=sbagliata pnpm hevy:whoami` → esce con codice 1 e stampa un
  `HevyApiError` con `status 401` (una GET reale rifiutata, nessun effetto).
- **F — Giro guidato**: non c'è nulla da guardare. Si dichiara.
- **G — Ripristino**: niente da ripristinare. Nessuna fixture piantata.

Che cosa NON copre: le scritture su Hevy (02), la rete reale nei test (mai).

## 8. Messa in esercizio

- Variabili d'ambiente: solo `HEVY_API_KEY` in `.env.local` (Development). Nessun Vercel, nessun
  Turso (06).
- DDL: nessuno.
- Misura di chiusura: `pnpm hevy:whoami` stampa la riga attesa. In esercizio = PR mergiata su `main`
  + collaudo annotato in CLAUDE.md (indice § 7.5, D-37). La PR la apre chi implementa dopo l'OK al
  commit, con `gh pr create`; il merge lo fa l'owner.

## 9. Documentazione

- `CLAUDE.md`: § Current Status → Latest (una riga: scaffold e client, dove sono, cosa viene dopo);
  § Current Status → Stack: via «nothing installed yet», e le versioni pinnate si leggono in
  `package.json`; § Architecture Snapshot: il titolo «(as specified; nothing exists yet)» diventa
  «(as specified; `src/lib/hevy` exists from spec 01)» e la riga del client Hevy; § Testing:
  «Vitest, `fakeFetch` and the fixtures are in place (01); the rest as specified»; § Known Issues:
  «None.»; § Not yet present: la riga «Any code, `package.json`, migrations» diventa «Migrations:
  born in spec 03», il resto resta com'è.
- `AGENTS.md`: § 3 Hevy (già scritta al bootstrap): nessuna riga nuova sulle buste (ci sono già),
  il retry diventa quello di D-39 e il messaggio del 400 diventa «Invalid page size» (quello dello
  spec OpenAPI); § 2.3: la regola `env-schema.ts` / `env.ts` / `EnvSchema.pick(...)` (D-36) e la forma
  `main()` degli script; § 2.5 e § 9 Testing: `TZ=UTC` nei test (come Vercel), `fakeFetch` e
  `fixture`, alias `server-only`, `include` di Vitest con `tests/**`; § 10 Commands: gli script reali;
  nell'introduzione via «Written on 2026-10-23, before any code» (da qui il codice c'è).
- `README.md` (inglese, D-35): nuova sezione `## Setup` dopo `## Stack` (Node ≥ 22.9, `corepack
  enable`, `pnpm install`, copia di `.env.example` in `.env.local` con la chiave Hevy Pro, `pnpm
  hevy:whoami`); la riga «Status (2026-10-23): planning done, no code yet» diventa «Status
  (<data>): scaffold and Hevy client in place; the v1 is being implemented one spec at a time».
- `WORKFLOW.md` § 3: la tabella dei comandi diventa vera (`typecheck`, `lint`, `test`, `build`,
  `hevy:whoami`), «Tests run with `TZ=UTC` (as on Vercel)», e la nota «State on 2026-10-23 … There is
  no code yet» si toglie (le colonne «Since» dicono già il resto).
- Misura in chiusura: CLAUDE.md con il comando dell'indice § 7.3 (atteso < 20.000 caratteri);
  `wc -l AGENTS.md` (atteso < 300 dopo questa spec).

## 10. Fuori ambito, dichiarato

- Scritture su Hevy, cartella, routine, template custom → **02**.
- Persistenza, Turso, conversione delle date → **03**.
- `POST /v1/workouts`, `PUT /v1/workouts/{id}`, body measurements: non servono (l'app non registra
  allenamenti, D-8) → CLAUDE.md § Not yet present («Hevy: write endpoints for workouts and body
  measurements are not wrapped»).
- Rate limit reale di Hevy: sconosciuto; se un 429 compare in esercizio, la regola di retry si
  rivede → CLAUDE.md § Not yet present.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **M** (scaffold fuori dal repo + 1 modulo + harness
+ test).

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

Implementa la spec 01 esattamente come scritta: § 4 è il codice (lo scaffold si genera FUORI dal
repo e si copiano solo i file elencati in § 4.1; le versioni `<esatta>` le leggi con `npm view` e
le scrivi in T-2), § 6 i test con le mutazioni che li provano portanti, § 7 il collaudo (fase per
fase, esito atteso prima; le due GET reali di `hevy:whoami` le lanci dopo il mio OK e in CLAUDE.md
non scrivi mai lo username), § 9 la documentazione. Non decidere nulla che la spec ha già deciso;
se trovi un punto che la spec non copre, fermati e chiedimelo con AskUserQuestion. Le
etichette della serie (D-N, T-N, C-N, «spec 01») non entrano nel codice.

Alla fine: riassumi il diff, scrivi lo stato nella riga «Stato» della spec e in § 5
dell'indice, aggiorna CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo) e apri la
PR con `gh pr create`.
```
