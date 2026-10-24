# 02 — Piano, scale di varianti, mappa esercizi e caricamento delle routine da CLI

> **Stato**: scritta 2026-10-23 · riletta in modo adversariale il 2026-10-24: 37 rilievi, 6 decisioni
> (D-39, D-40, D-44, D-45, D-46, D-47). Pronta per l'implementazione dopo la 01.
> **Branch**: `feature/02-plan-upload` da `main`. **Dipende da**: 01 (`createHevyClient`, schemi e
> tipi C-2, `EnvSchema`, fixture, `fakeFetch`).
> **Indice**: [00-INDICE.md](00-INDICE.md) — decisioni D-4, D-5, D-6, D-7, D-18, D-19, D-20, D-24,
> D-25, D-26, D-27, D-36, D-39, D-40, D-44, D-45, D-46, D-47; scelte T-7, T-8, T-18; contratti C-3,
> C-4, C-5, C-6 (li crea); usa C-1, C-2, C-15.

## 1. Il problema oggi (prove)

1. Su Hevy l'account non ha **nessuna routine e nessuna cartella** (`GET /v1/routines` e
   `GET /v1/routine_folders` reali del 2026-10-23: `page_count: 1`, liste vuote — fixture
   `routines_page_empty.json`, `routine_folders_page_empty.json`). L'owner si allena già a corpo
   libero senza routine (workout «Push» del 2026-10-22: Push Up, Pike Pushup, Bench Dip, Plank —
   fixture `workout.json`) e vuole le 5 routine del piano su Hevy **il prima possibile** (P1).
2. Il catalogo Hevy copre molti esercizi del piano con un id fisso (letti il 2026-10-23, fixture
   `catalog_bodyweight.json`, 121 template con attrezzatura `none`/`other`): Push Up `392887AA`,
   Kneeling Push Up `B74A95BB`, Incline Push Ups `39C99849`, Decline Push Up `C43825EA`, Diamond
   Push Up `6575F52D`, Push Up - Close Grip `947DAC23`, Pike Pushup `0EFE8162`, Plank `C6C9B8A0`,
   Side Plank `E3EDA509`, Squat (Bodyweight) `9694DA61`, Assisted Pistol Squats `5BFF35BA`, Reverse
   Lunge `C284D923`, Lateral Lunge `F2A37357`, Glute Bridge `CDA23948`, Single Leg Glute Bridge
   `CBC0D834`, Standing Calf Raise `06745E58`, Single Leg Standing Calf Raise `238BA231`, Wall Sit
   `C8706C80`, Dead Bug `D8911FC4`, Bird Dog `BD0AD077`, Superman `218DA87C`, Hip Thrust `92B8C7E1`,
   Single Leg Hip Thrust `487B3755`, Lateral Leg Raises `DC59D143`. **Inverted Row `425805F4`** è di
   catalogo ma non sta in quella fixture (ha un'altra attrezzatura): **il primo passo
   dell'implementazione** (§ 7 fase 0, D-44) è una `GET /v1/exercise_templates/425805F4` reale di
   sola lettura, la cui risposta nuda va nella nuova fixture reale `tests/fixtures/hevy/catalog_extra.json`
   (array di template; i test leggono le due fixture insieme). **Mancano** (solo versioni con
   bilanciere o manubri, o assenti): bulgarian split squat a corpo libero, stacco rumeno a una gamba
   a corpo libero, Prone Y-T-W, reverse snow angel, le varianti «con pausa», rotazioni toraciche,
   90/90. L'account ha già due custom utili: **Hollowbody Hold** (duration) e **Couch Stretch**
   (duration).
3. Un custom creato su Hevy **non si può modificare né cancellare** (OpenAPI: nessun PUT/DELETE su
   `exercise_templates`); `POST` risponde 403 `exceeds-custom-exercise-limit` oltre un limite non
   documentato (13 custom esistenti oggi). Quindi: cercare prima, creare solo ciò che manca, una
   volta, per titolo esatto (D-26, T-7). Per lo stesso motivo il client della 01 **non ritenta mai
   una POST** su 5xx (D-39): un POST fallito si segnala e il rilancio dell'upload trova per titolo
   ciò che Hevy ha creato davvero.
4. I set delle routine non hanno un modo per dire «max-2», «per gamba», «45–60"», «tempo 3-1-1»:
   vanno nelle **note dell'esercizio** (D-18, D-19, D-20), generate da `targetNote` dallo stato dello
   slot; la 04 le **rigenera** a ogni «Applica» e non le legge mai.
5. `PUT /v1/routines/{id}` sostituisce tutto: il caricamento deve essere idempotente per titolo
   (la stessa routine, rilanciata, aggiorna invece di duplicare) e `POST` risponde 403 «Routine
   limit exceeded» oltre il limite di Hevy.

## 2. Obiettivo e criteri di accettazione

| # | Criterio | Prova |
|---|---|---|
| A1 | `PLAN` (C-3) contiene le 5 routine del tema con tutti gli slot di § 4.1; ogni slot ha scala, variante di partenza, serie, target, recupero, flag per lato e flag «in revisione»; la mobilità ha `reviewed: false`. | `plan.test.ts`: conteggi (5 routine; 6 + 5 + 8 + 6 + 5 = 30 slot; 27 in revisione), uno snapshot di `PLAN` intero e uno delle note generate. Mutazione: `sets: 5` su `mon_push_up` → lo snapshot di `PLAN` va rosso. |
| A2 | `SCALES` (C-3) rispetta D-4 per costruzione: ogni variante ha `quiet: true` e `simple: true` dichiarati dal pianificatore, titolo italiano se custom, mappa Hevy (catalogo con id presente in `catalog_bodyweight.json` o `catalog_extra.json`, o custom con tipo e gruppo muscolare), target per ogni genere di slot che può raggiungerla, template del tipo giusto (tenute su `duration`, ripetizioni su `reps_only`). | `scales.test.ts`: per ogni variante di catalogo, l'id esiste nelle due fixture con lo stesso titolo; per ogni slot, ogni variante della sua scala ha `targets[slot.initialTarget.kind]`; titoli custom unici; nessun titolo custom uguale a un titolo di catalogo; tipo del template coerente con i target. Mutazione: cambiare `392887AA` in `392887AB` → rosso; mappare `wall_sit` su `9694DA61` → rosso. |
| A3 | `buildRoutineBody` (C-4) produce un `RoutineWriteBody` valido (schema C-2) per ogni routine del piano dallo stato iniziale, con set, note e recuperi di D-18…D-27. | `routine-builder.test.ts`: `RoutineWriteBodySchema.parse(body)` passa per le 5 routine; snapshot per routine; casi: max2 → `reps` e `rep_range` null + nota con la soglia; range → `rep_range {start,end}`; tenuta → `duration_seconds` = minimo + nota con il range; per lato → nota; mobilità → `rest_seconds: 0`. Mutazione: mettere `duration_seconds` al massimo → rosso. |
| A4 | `ensureTemplates` (C-6) risolve il catalogo per id e i custom per titolo esatto **e tipo** fra i template esistenti, crea solo i custom mancanti e restituisce la mappa variante → id; una seconda chiamata con la mappa già completa non crea nulla. | `ensure-templates.test.ts` con `createFakeHevy` (§ 6): 11 varianti custom in uso, 2 esistenti → 9 `POST`; rilancio con i 9 appena creati nella lista → 0 `POST`; custom omonimo di tipo diverso → errore. Mutazione: confronto case-insensitive → il test «titolo diverso solo per maiuscola NON si riusa» va rosso; togliere il controllo del tipo → rosso. |
| A5 | `uploadPlan` (§ 4.6) è idempotente: cartella trovata o creata, template assicurati, routine trovate per (cartella, titolo) e aggiornate con PUT, altrimenti create con POST; `dryRun: true` non chiama nessuna scrittura, stampa i body e il report dice che cosa creerebbe e che cosa riuserebbe. | `upload-plan.test.ts` con `createFakeHevy`: prima corsa → 1 cartella, 9 custom, 5 POST; seconda corsa (liste popolate) → 0 cartelle, 0 custom, 5 PUT, 0 POST; dry-run → 0 scritture, `templatesToCreate` con 9 titoli, `templatesReused` con 2. Mutazione: cercare la routine solo per titolo (senza cartella) → il test «routine omonima in un'altra cartella» va rosso. |
| A6 | Stima di durata: ogni routine ≤ 40 min, mercoledì ≤ 30 min (D-46), con il modello di § 4.5. | `plan.test.ts`: `estimateRoutineMinutes` per le 5 routine (il venerdì vale 40,0; il mercoledì 28,5). Mutazione: recupero 120 s sui multiarticolari → Ven sale a 42,5 e supera 40. |
| A7 | `pnpm hevy:upload` reale porta su Hevy la cartella «Coach · Corpo libero» con le 5 routine e i 9 custom; rilanciato subito dopo, fa 5 PUT e 0 POST. | Collaudo § 7 (scritture reali, dopo l'OK dell'owner). |

## 3. Decisioni

- **D-4, D-5, D-6, D-7** (vincoli e piano; D-5 precisata da D-46: mercoledì sotto i 30 min),
  **D-18** (max-2: la soglia sta in `Variant.targets.max2.threshold`, quella che lo slot adotta
  arrivando sulla variante, e poi vive in `SlotState.target.threshold`), **D-19** (tenute), **D-20**
  (per lato), **D-24** (riscaldamento e mobilità; riscritta da D-45: rotazioni toraciche a
  ripetizioni), **D-25** (nomi), **D-26** (custom in italiano, riuso per titolo esatto), **D-27**
  (recuperi; riscritta da D-47 con l'elenco di § 4.1).
- **D-36** (lo script legge le env con `EnvSchema.pick(...)`), **D-39** (il client non ritenta le
  POST), **D-40** (`hevy:upload` usa sempre gli stati iniziali: si lancia solo prima del primo
  «Applica»; la forma `--from-db` nasce nella 04), **D-44** (`catalog_extra.json` da una GET reale),
  **D-45** (rotazioni toraciche a ripetizioni), **D-46** (tetto del mercoledì 30 min, nota «~28
  min»), **D-47** (recuperi per famiglia).
- **T-7** (custom creati pigramente), **T-8** (identità dello slot: `slotId`; la risoluzione set →
  slot usa prima il `routine_id` del workout e poi il template, 04 § 4.2).
- **T-18 (adottata qui, ribaltabile)**: le ripetizioni **fisse** del piano (affondo 10, glute
  bridge 12, calf 15, Y-T-W 8, push-up con pausa 8, superman 12, snow angel 12, abduzioni 15)
  diventano **range di 4** (`n`–`n+4`; per il calf e le abduzioni `15`–`20`): la doppia progressione
  (D-6) ha bisogno di un massimo da chiudere; il minimo è il numero del piano.
- Adottate qui, ribaltabili: (a) la stima di durata di § 4.5 e i tetti 40/30 minuti; (b) la scala
  si ferma dove finisce la semplicità (nessuna variante «archer», «pseudo planche», «pistol» libero,
  «nordic»); all'ultima variante Claude può proporre solo `keep` o `increase` (04); (c) ogni custom
  nasce con `equipment_category: 'none'` e `other_muscles: []`; (d) la nota dell'esercizio è
  generata da `targetNote` e non è mai scritta a mano, e sta in **140 caratteri** (limite scelto qui:
  Hevy non ne documenta uno; il test lo fissa); (e) `superset_id: null` ovunque; (f) le due varianti
  di mobilità a tempo valgono «60" per lato» e si registrano come un set da 120 s; le rotazioni
  toraciche come un set da 10 ripetizioni per lato (D-45); (g) per una tenuta `perSide`
  (side plank) `duration_seconds` del set è il tempo di **un** lato, e la 04 legge il valore
  registrato come tempo di un lato; (h) `Slot.note` contiene solo ciò che vale per **ogni** variante
  della scala (es. «ripetizioni per lettera (Y, T, W)»); ciò che dipende dalla variante sta in
  `Variant.cue`, perché la nota dello slot resta anche dopo un «Applica»; (i) due cartelle con il
  titolo del piano, o due routine con lo stesso titolo nella cartella, sono un errore dell'account:
  l'upload si ferma con un messaggio e si sistema a mano nell'app; (j) le scale oltre la variante di
  partenza (circa 50 varianti, con titoli custom permanenti) sono quelle di § 4.2, approvate con
  questa spec: una variante si aggiunge o si toglie solo con una nuova decisione dell'owner.

## 4. La soluzione

### 4.1 Il piano — `src/lib/plan/plan.ts` (C-3)

Tipi in `src/lib/plan/types.ts`:

```ts
export type RoutineKey = 'mon_upper_a' | 'tue_lower_a' | 'wed_core' | 'thu_upper_b' | 'fri_lower_b';
export type ScaleId = string;
export type VariantId = string;
export type SlotId = string;

export type Target =
  | { kind: 'reps'; min: number; max: number }
  | { kind: 'max2'; threshold: number }
  | { kind: 'hold'; seconds: number; maxSeconds: number };
export type TargetKind = Target['kind'];

export interface HevyCatalogRef { kind: 'catalog'; templateId: string; title: string }
export interface HevyCustomRef {
  kind: 'custom';
  title: string;
  exerciseType: 'reps_only' | 'duration';
  muscleGroup: string;
}
export interface Variant {
  id: VariantId;
  /** Italian name shown in the dashboard. */
  title: string;
  hevy: HevyCatalogRef | HevyCustomRef;
  /** Short execution cue, written into the Hevy exercise note. */
  cue: string;
  /** The target a slot adopts when it moves onto this variant; one entry per slot kind that can reach it. */
  targets: Partial<Record<TargetKind, Target>>;
  quiet: true;
  simple: true;
}
export interface Scale { id: ScaleId; title: string; variants: Variant[] }

export interface Slot {
  id: SlotId;
  scaleId: ScaleId;
  startVariantId: VariantId;
  sets: number;
  /** The plan's own numbers for this slot; later states come from Variant.targets or from "increase". */
  initialTarget: Target;
  perSide: boolean;
  restSeconds: number;
  /** Extra note from the plan (e.g. "tempo 3-1-1"); empty string when none. */
  note: string;
  /** false for mobility: logged on Hevy, never reviewed by Claude. */
  reviewed: boolean;
}
export interface PlanRoutine { key: RoutineKey; title: string; notes: string; slots: Slot[] }

export interface SlotState {
  slotId: SlotId;
  variantId: VariantId;
  target: Target;
  updatedAt: number; // epoch ms
  source: 'plan' | 'apply';
}
export type TemplateIdMap = Record<VariantId, string>;
```

`src/lib/plan/plan.ts`:

```ts
import type { PlanRoutine } from './types';

export const FOLDER_TITLE = 'Coach · Corpo libero';
const WARMUP = 'Riscaldamento 5\': cerchi con le braccia, cat-camel, squat a vuoto, affondi lenti.';

export const PLAN: PlanRoutine[] = [
  {
    key: 'mon_upper_a',
    title: 'Lun · Upper A',
    notes: `${WARMUP} Recuperi 60–90". «max-2» = fermati a 2 ripetizioni dal cedimento.`,
    slots: [
      { id: 'mon_push_up', scaleId: 'push_up', startVariantId: 'push_up', sets: 4, initialTarget: { kind: 'max2', threshold: 15 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'mon_row', scaleId: 'table_row', startVariantId: 'table_row', sets: 4, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'mon_pike', scaleId: 'pike', startVariantId: 'pike_push_up', sets: 3, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'mon_push_up_close', scaleId: 'push_up_close', startVariantId: 'push_up_close', sets: 3, initialTarget: { kind: 'max2', threshold: 12 }, perSide: false, restSeconds: 60, note: '', reviewed: true },
      { id: 'mon_ytw', scaleId: 'ytw', startVariantId: 'prone_ytw', sets: 2, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: false, restSeconds: 60, note: 'ripetizioni per lettera (Y, T, W)', reviewed: true },
      { id: 'mon_plank', scaleId: 'plank', startVariantId: 'plank', sets: 2, initialTarget: { kind: 'hold', seconds: 45, maxSeconds: 60 }, perSide: false, restSeconds: 45, note: '', reviewed: true },
    ],
  },
  {
    key: 'tue_lower_a',
    title: 'Mar · Lower A',
    notes: `${WARMUP} Recuperi 60–90".`,
    slots: [
      { id: 'tue_squat', scaleId: 'squat', startVariantId: 'squat_tempo', sets: 4, initialTarget: { kind: 'reps', min: 15, max: 20 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'tue_reverse_lunge', scaleId: 'reverse_lunge', startVariantId: 'reverse_lunge', sets: 3, initialTarget: { kind: 'reps', min: 10, max: 14 }, perSide: true, restSeconds: 90, note: '', reviewed: true },
      { id: 'tue_sl_glute_bridge', scaleId: 'glute_bridge', startVariantId: 'sl_glute_bridge', sets: 3, initialTarget: { kind: 'reps', min: 12, max: 16 }, perSide: true, restSeconds: 60, note: '', reviewed: true },
      { id: 'tue_sl_calf', scaleId: 'calf', startVariantId: 'sl_calf_raise', sets: 3, initialTarget: { kind: 'reps', min: 15, max: 20 }, perSide: true, restSeconds: 60, note: '', reviewed: true },
      { id: 'tue_wall_sit', scaleId: 'wall_sit', startVariantId: 'wall_sit', sets: 2, initialTarget: { kind: 'hold', seconds: 45, maxSeconds: 60 }, perSide: false, restSeconds: 45, note: '', reviewed: true },
    ],
  },
  {
    key: 'wed_core',
    title: 'Mer · Core + mobilità',
    notes: `${WARMUP} Sessione leggera (~28 min). Chiudi con la mobilità (~6').`,
    slots: [
      { id: 'wed_dead_bug', scaleId: 'dead_bug', startVariantId: 'dead_bug', sets: 3, initialTarget: { kind: 'reps', min: 10, max: 14 }, perSide: true, restSeconds: 45, note: '', reviewed: true },
      { id: 'wed_hollow', scaleId: 'hollow', startVariantId: 'hollow_hold', sets: 3, initialTarget: { kind: 'hold', seconds: 20, maxSeconds: 30 }, perSide: false, restSeconds: 45, note: '', reviewed: true },
      { id: 'wed_side_plank', scaleId: 'side_plank', startVariantId: 'side_plank', sets: 2, initialTarget: { kind: 'hold', seconds: 30, maxSeconds: 45 }, perSide: true, restSeconds: 45, note: '', reviewed: true },
      { id: 'wed_bird_dog', scaleId: 'bird_dog', startVariantId: 'bird_dog', sets: 2, initialTarget: { kind: 'reps', min: 10, max: 14 }, perSide: true, restSeconds: 45, note: '', reviewed: true },
      { id: 'wed_superman', scaleId: 'superman', startVariantId: 'superman', sets: 2, initialTarget: { kind: 'reps', min: 12, max: 16 }, perSide: false, restSeconds: 45, note: '', reviewed: true },
      { id: 'wed_hip_9090', scaleId: 'mobility_hip_9090', startVariantId: 'hip_9090', sets: 1, initialTarget: { kind: 'hold', seconds: 120, maxSeconds: 120 }, perSide: false, restSeconds: 0, note: '', reviewed: false },
      { id: 'wed_couch_stretch', scaleId: 'mobility_couch', startVariantId: 'couch_stretch', sets: 1, initialTarget: { kind: 'hold', seconds: 120, maxSeconds: 120 }, perSide: false, restSeconds: 0, note: '', reviewed: false },
      { id: 'wed_thoracic', scaleId: 'mobility_thoracic', startVariantId: 'thoracic_rotation', sets: 1, initialTarget: { kind: 'reps', min: 10, max: 10 }, perSide: true, restSeconds: 0, note: 'lento', reviewed: false },
    ],
  },
  {
    key: 'thu_upper_b',
    title: 'Gio · Upper B',
    notes: `${WARMUP} Recuperi 60–90". «max-2» = fermati a 2 ripetizioni dal cedimento.`,
    slots: [
      { id: 'thu_decline_push_up', scaleId: 'push_up', startVariantId: 'decline_push_up', sets: 4, initialTarget: { kind: 'max2', threshold: 12 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'thu_row_pause', scaleId: 'table_row', startVariantId: 'table_row_pause', sets: 4, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'thu_pike_pause', scaleId: 'pike', startVariantId: 'pike_push_up_pause', sets: 3, initialTarget: { kind: 'reps', min: 6, max: 10 }, perSide: false, restSeconds: 90, note: '', reviewed: true },
      { id: 'thu_push_up_pause', scaleId: 'push_up_pause', startVariantId: 'push_up_pause', sets: 3, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: false, restSeconds: 60, note: '', reviewed: true },
      { id: 'thu_snow_angel', scaleId: 'snow_angel', startVariantId: 'reverse_snow_angel', sets: 2, initialTarget: { kind: 'reps', min: 12, max: 16 }, perSide: false, restSeconds: 60, note: '', reviewed: true },
      { id: 'thu_hollow', scaleId: 'hollow', startVariantId: 'hollow_hold', sets: 2, initialTarget: { kind: 'hold', seconds: 30, maxSeconds: 40 }, perSide: false, restSeconds: 45, note: '', reviewed: true },
    ],
  },
  {
    key: 'fri_lower_b',
    title: 'Ven · Lower B',
    notes: `${WARMUP} Recuperi 60–90".`,
    slots: [
      { id: 'fri_bss', scaleId: 'bulgarian', startVariantId: 'bss_sofa', sets: 4, initialTarget: { kind: 'reps', min: 10, max: 12 }, perSide: true, restSeconds: 90, note: '', reviewed: true },
      { id: 'fri_sl_rdl', scaleId: 'sl_rdl', startVariantId: 'sl_rdl', sets: 3, initialTarget: { kind: 'reps', min: 10, max: 14 }, perSide: true, restSeconds: 90, note: '', reviewed: true },
      { id: 'fri_sl_hip_thrust', scaleId: 'hip_thrust', startVariantId: 'sl_hip_thrust', sets: 3, initialTarget: { kind: 'reps', min: 12, max: 16 }, perSide: true, restSeconds: 60, note: '', reviewed: true },
      { id: 'fri_lateral_lunge', scaleId: 'lateral_lunge', startVariantId: 'lateral_lunge', sets: 3, initialTarget: { kind: 'reps', min: 8, max: 12 }, perSide: true, restSeconds: 60, note: '', reviewed: true },
      { id: 'fri_abduction', scaleId: 'abduction', startVariantId: 'side_lying_abduction', sets: 2, initialTarget: { kind: 'reps', min: 15, max: 20 }, perSide: true, restSeconds: 60, note: '', reviewed: true },
    ],
  },
];
```

I recuperi sono quelli di D-47: 90 s su push-up, push-up declinati, rematore, pike, squat, affondo
indietro, bulgarian, stacco rumeno; 60 s su push-up stretti e con pausa, affondo laterale, glute
bridge, calf, hip thrust, Y-T-W, snow angel, abduzioni; 45 s sul core e sul wall sit; 0 sulla
mobilità. Le rotazioni toraciche sono un set da 10 ripetizioni per lato (D-45). `Slot.note` è quasi sempre vuota
(§ 3 h): «tempo 3-1-1», «su gradino», «piede dietro sul divano», «schiena sul divano» e «60" per
lato» stanno nelle cue delle varianti che li riguardano.

### 4.2 Le scale di varianti — `src/lib/plan/scales.ts` (C-3)

Ogni scala va dalla variante più facile alla più difficile. `quiet: true` e `simple: true` sono
dichiarazioni del pianificatore (2026-10-23): niente salti, niente impatti, niente tecnica
(archer, pseudo-planche, pistol libero, nordic curl sono esclusi apposta). Le `targets` danno il
target che uno slot adotta quando arriva su quella variante; `reps` è il range, `max2` la soglia di
passaggio (ripetizioni su tutte le serie), `hold` secondi minimo e massimo.

Helper per scrivere meno (nello stesso file):

```ts
const cat = (templateId: string, title: string): HevyCatalogRef => ({ kind: 'catalog', templateId, title });
const cus = (title: string, exerciseType: 'reps_only' | 'duration', muscleGroup: string): HevyCustomRef => ({ kind: 'custom', title, exerciseType, muscleGroup });
const reps = (min: number, max: number): Target => ({ kind: 'reps', min, max });
const max2 = (threshold: number): Target => ({ kind: 'max2', threshold });
const hold = (seconds: number, maxSeconds: number): Target => ({ kind: 'hold', seconds, maxSeconds });
const v = (id: string, title: string, hevy: Variant['hevy'], cue: string, targets: Variant['targets']): Variant => ({ id, title, hevy, cue, targets, quiet: true, simple: true });
```

Le scale (`export const SCALES: Record<ScaleId, Scale>`), variante per variante — **id · titolo ·
mappa Hevy · cue · targets**:

| Scala | Varianti (dalla più facile) |
|---|---|
| `push_up` «Push-up» | `incline_push_up` · Push-up inclinati (mani sul tavolo) · cat `39C99849` «Incline Push Ups» · «mani sul bordo del tavolo, corpo in linea» · max2 20, reps 12–16 · · `kneeling_push_up` · Push-up sulle ginocchia · cat `B74A95BB` «Kneeling Push Up» · «ginocchia a terra, bacino avanti» · max2 20, reps 12–16 · · `push_up` · Push-up · cat `392887AA` «Push Up» · «corpo in linea, petto a un pugno da terra» · max2 15, reps 10–14 · · `decline_push_up` · Push-up declinati (piedi sul divano) · cat `C43825EA` «Decline Push Up» · «piedi sul divano, mani sotto le spalle» · max2 12, reps 8–12 · · `decline_push_up_pause` · Push-up declinati con pausa 2" · cus «Push-up declinati con pausa 2"», reps_only, chest · «2 secondi fermo in basso» · max2 10, reps 6–10 · · `decline_push_up_chair` · Push-up declinati piedi sulla sedia · cus «Push-up declinati piedi sulla sedia», reps_only, chest · «piedi più alti del divano, scapole libere» · max2 12, reps 8–12 |
| `push_up_close` «Push-up stretti» | `kneeling_close_push_up` · Push-up stretti sulle ginocchia · cus «Push-up stretti sulle ginocchia», reps_only, triceps · «mani sotto il petto, gomiti lungo i fianchi» · max2 15, reps 10–14 · · `push_up_close` · Push-up stretti · cat `947DAC23` «Push Up - Close Grip» · «mani sotto il petto, gomiti lungo i fianchi» · max2 12, reps 8–12 · · `diamond_push_up` · Push-up diamante · cat `6575F52D` «Diamond Push Up» · «pollici e indici a rombo» · max2 12, reps 8–12 · · `decline_close_push_up` · Push-up stretti declinati · cus «Push-up stretti declinati», reps_only, triceps · «piedi sul divano, mani sotto il petto» · max2 10, reps 6–10 |
| `push_up_pause` «Push-up con pausa» | `kneeling_push_up_pause` · Push-up sulle ginocchia con pausa 2" · cus «Push-up sulle ginocchia con pausa 2"», reps_only, chest · «2 secondi fermo in basso» · reps 10–14 · · `push_up_pause` · Push-up con pausa in basso · cus «Push-up con pausa in basso», reps_only, chest · «2 secondi fermo a un pugno da terra» · reps 8–12 · · `decline_push_up_pause` (la stessa variante della scala `push_up`, per riferimento) |
| `pike` «Pike push-up» | `pike_push_up_partial` · Pike push-up parziale · cus «Pike push-up parziale», reps_only, shoulders · «mezzo movimento, testa verso terra» · reps 8–12 · · `pike_push_up` · Pike push-up · cat `0EFE8162` «Pike Pushup» · «bacino alto, testa tra le mani» · reps 8–12 · · `pike_push_up_pause` · Pike push-up con pausa in basso · cus «Pike push-up con pausa in basso», reps_only, shoulders · «2 secondi fermo in basso» · reps 6–10 · · `pike_push_up_feet_sofa` · Pike push-up piedi sul divano · cus «Pike push-up piedi sul divano», reps_only, shoulders · «piedi sul divano, bacino sopra le spalle» · reps 6–10 · · `pike_push_up_feet_sofa_pause` · Pike push-up piedi sul divano con pausa · cus «Pike push-up piedi sul divano con pausa», reps_only, shoulders · «2 secondi fermo in basso» · reps 5–8 |
| `table_row` «Rematore sotto il tavolo» | `table_row_knees_bent` · Rematore sotto il tavolo ginocchia piegate · cus «Rematore sotto il tavolo ginocchia piegate», reps_only, upper_back · «piedi a terra, ginocchia piegate» · reps 8–12 · · `table_row` · Rematore sotto il tavolo · cat `425805F4` «Inverted Row» · «gambe tese, corpo in linea, petto al bordo» · reps 8–12 · · `table_row_pause` · Rematore sotto il tavolo con pausa 2" · cus «Rematore sotto il tavolo con pausa 2"», reps_only, upper_back · «2 secondi fermo in alto» · reps 8–12 · · `table_row_feet_sofa` · Rematore sotto il tavolo piedi sul divano · cus «Rematore sotto il tavolo piedi sul divano», reps_only, upper_back · «piedi rialzati, corpo in linea» · reps 8–12 · · `table_row_feet_sofa_pause` · Rematore piedi sul divano con pausa 2" · cus «Rematore piedi sul divano con pausa 2"», reps_only, upper_back · «2 secondi fermo in alto» · reps 6–10 |
| `ytw` «Prone Y-T-W» | `prone_ytw` · Prone Y-T-W · cus «Prone Y-T-W», reps_only, upper_back · «prono, pollici in alto, scapole giù» · reps 8–12 · · `prone_ytw_pause` · Prone Y-T-W con pausa 2" · cus «Prone Y-T-W con pausa 2"», reps_only, upper_back · «2 secondi in alto per lettera» · reps 8–12 |
| `snow_angel` «Reverse snow angel» | `reverse_snow_angel` · Reverse snow angel · cus «Reverse snow angel», reps_only, shoulders · «prono, braccia da fianchi a sopra la testa» · reps 12–16 · · `reverse_snow_angel_pause` · Reverse snow angel con pausa 2" · cus «Reverse snow angel con pausa 2"», reps_only, shoulders · «2 secondi sopra la testa» · reps 10–14 |
| `plank` «Plank» | `plank_knees` · Plank sulle ginocchia · cus «Plank sulle ginocchia», duration, abdominals · «gomiti sotto le spalle» · hold 45–60 · · `plank` · Plank · cat `C6C9B8A0` «Plank» · «glutei contratti, bacino neutro» · hold 45–60 · · `plank_feet_sofa` · Plank piedi sul divano · cus «Plank piedi sul divano», duration, abdominals · «piedi rialzati, corpo in linea» · hold 45–60 · · `plank_one_leg` · Plank a una gamba · cus «Plank a una gamba», duration, abdominals · «una gamba sollevata, cambia a metà tempo» · hold 30–45 · · `plank_long_lever` · Plank con mani avanti · cus «Plank con mani avanti», duration, abdominals · «gomiti davanti alla testa» · hold 30–45 |
| `hollow` «Hollow hold» | `hollow_tuck` · Hollow hold ginocchia piegate · cus «Hollow hold ginocchia piegate», duration, abdominals · «lombare a terra» · hold 20–30 · · `hollow_hold` · Hollow hold · cus **«Hollowbody Hold»** (esiste già), duration, abdominals · «lombare a terra, gambe tese, braccia ai fianchi» · hold 20–30 · · `hollow_hold_arms` · Hollow hold braccia sopra la testa · cus «Hollow hold braccia sopra la testa», duration, abdominals · «braccia tese dietro le orecchie» · hold 20–30 |
| `squat` «Squat» | `squat_tempo` · Squat corpo libero tempo 3-1-1 · cat `9694DA61` «Squat (Bodyweight)» · «3 secondi in discesa, 1 fermo, 1 in salita» · reps 15–20 · · `squat_one_and_half` · Squat 1½ · cus «Squat 1½», reps_only, quadriceps · «scendi, risali a metà, riscendi, risali» · reps 12–16 · · `squat_pause` · Squat con pausa 3" in basso · cus «Squat con pausa 3" in basso», reps_only, quadriceps · «3 secondi fermo in basso» · reps 12–16 · · `assisted_pistol` · Pistol squat assistito · cat `5BFF35BA` «Assisted Pistol Squats» · «mano al tavolo, discesa lenta» · reps 6–10 |
| `reverse_lunge` «Affondo indietro» | `reverse_lunge` · Affondo indietro · cat `C284D923` «Reverse Lunge» · «ginocchio dietro sfiora terra, busto dritto» · reps 10–14 · · `reverse_lunge_pause` · Affondo indietro con pausa 2" · cus «Affondo indietro con pausa 2"», reps_only, quadriceps · «2 secondi fermo in basso» · reps 10–14 · · `reverse_lunge_deficit` · Affondo indietro dal gradino · cus «Affondo indietro dal gradino», reps_only, quadriceps · «piede avanti sul gradino» · reps 8–12 |
| `glute_bridge` «Glute bridge» | `glute_bridge` · Glute bridge · cat `CDA23948` «Glute Bridge» · «spingi con i talloni, stringi in alto» · reps 15–20 · · `sl_glute_bridge` · Glute bridge a una gamba · cat `CBC0D834` «Single Leg Glute Bridge» · «bacino in linea, gamba libera tesa» · reps 12–16 · · `sl_glute_bridge_pause` · Glute bridge a una gamba con pausa 2" · cus «Glute bridge a una gamba con pausa 2"», reps_only, glutes · «2 secondi in alto» · reps 10–14 · · `sl_glute_bridge_foot_sofa` · Glute bridge a una gamba piede sul divano · cus «Glute bridge a una gamba piede sul divano», reps_only, glutes · «tallone sul divano» · reps 10–14 |
| `calf` «Calf raise» | `calf_raise` · Calf raise su gradino · cat `06745E58` «Standing Calf Raise» · «sul gradino, tutta l'escursione» · reps 20–25 · · `sl_calf_raise` · Calf raise a una gamba su gradino · cat `238BA231` «Single Leg Standing Calf Raise» · «sul gradino, mano al muro» · reps 15–20 · · `sl_calf_raise_pause` · Calf raise a una gamba con pausa 2" in alto · cus «Calf raise a una gamba con pausa 2" in alto», reps_only, calves · «2 secondi in punta» · reps 12–16 |
| `wall_sit` «Wall sit» | `wall_sit` · Wall sit · cat `C8706C80` «Wall Sit» · «cosce parallele a terra» · hold 45–60 · · `wall_sit_heels` · Wall sit talloni sollevati · cus «Wall sit talloni sollevati», duration, quadriceps · «talloni staccati da terra» · hold 40–60 · · `wall_sit_one_leg` · Wall sit a una gamba · cus «Wall sit a una gamba», duration, quadriceps · «una gamba tesa avanti, cambia a metà tempo» · hold 20–30 |
| `dead_bug` «Dead bug» | `dead_bug` · Dead bug · cat `D8911FC4` «Dead Bug» · «lombare a terra, braccio e gamba opposti» · reps 10–14 · · `dead_bug_pause` · Dead bug con pausa 2" · cus «Dead bug con pausa 2"», reps_only, abdominals · «2 secondi in estensione» · reps 10–14 · · `dead_bug_long` · Dead bug braccia e gambe tese · cus «Dead bug braccia e gambe tese», reps_only, abdominals · «gamba tesa a un palmo da terra» · reps 8–12 |
| `side_plank` «Side plank» | `side_plank_knees` · Side plank sulle ginocchia · cus «Side plank sulle ginocchia», duration, abdominals · «gomito sotto la spalla» · hold 30–45 · · `side_plank` · Side plank · cat `E3EDA509` «Side Plank» · «corpo in linea, bacino alto» · hold 30–45 · · `side_plank_feet_sofa` · Side plank piedi sul divano · cus «Side plank piedi sul divano», duration, abdominals · «piedi rialzati» · hold 30–45 · · `side_plank_leg_raised` · Side plank con gamba sollevata · cus «Side plank con gamba sollevata», duration, abdominals · «gamba sopra sollevata» · hold 20–30 |
| `bird_dog` «Bird dog» | `bird_dog` · Bird dog · cat `BD0AD077` «Bird Dog» · «braccio e gamba opposti, bacino fermo» · reps 10–14 · · `bird_dog_pause` · Bird dog con pausa 3" · cus «Bird dog con pausa 3"», reps_only, lower_back · «3 secondi in estensione» · reps 8–12 |
| `superman` «Superman» | `superman` · Superman · cat `218DA87C` «Superman» · «braccia e gambe sollevate, sguardo a terra» · reps 12–16 · · `superman_pause` · Superman con pausa 2" · cus «Superman con pausa 2"», reps_only, lower_back · «2 secondi in alto» · reps 10–14 |
| `bulgarian` «Bulgarian split squat» | `split_squat` · Split squat statico · cus «Split squat statico», reps_only, quadriceps · «piedi a terra, passo lungo» · reps 10–14 · · `bss_sofa` · Bulgarian split squat (piede sul divano) · cus «Bulgarian split squat (piede sul divano)», reps_only, quadriceps · «piede dietro sul divano, busto dritto» · reps 10–12 · · `bss_sofa_pause` · Bulgarian split squat con pausa 2" · cus «Bulgarian split squat con pausa 2"», reps_only, quadriceps · «2 secondi fermo in basso» · reps 8–12 · · `bss_sofa_tempo` · Bulgarian split squat tempo 3-1-1 · cus «Bulgarian split squat tempo 3-1-1», reps_only, quadriceps · «3 secondi in discesa» · reps 8–12 |
| `sl_rdl` «Stacco rumeno a una gamba» | `sl_rdl_assisted` · Stacco rumeno a una gamba con appoggio · cus «Stacco rumeno a una gamba con appoggio», reps_only, hamstrings · «mano al tavolo, schiena piatta» · reps 10–14 · · `sl_rdl` · Stacco rumeno a una gamba · cus «Stacco rumeno a una gamba», reps_only, hamstrings · «schiena piatta, gamba libera dietro» · reps 10–14 · · `sl_rdl_pause` · Stacco rumeno a una gamba con pausa 2" · cus «Stacco rumeno a una gamba con pausa 2"», reps_only, hamstrings · «2 secondi in basso» · reps 8–12 |
| `hip_thrust` «Hip thrust» | `hip_thrust` · Hip thrust (schiena sul divano) · cat `92B8C7E1` «Hip Thrust» · «schiena sul divano, mento al petto» · reps 15–20 · · `sl_hip_thrust` · Hip thrust a una gamba · cat `487B3755` «Single Leg Hip Thrust» · «schiena sul divano, bacino in linea» · reps 12–16 · · `sl_hip_thrust_pause` · Hip thrust a una gamba con pausa 2" · cus «Hip thrust a una gamba con pausa 2"», reps_only, glutes · «2 secondi in alto» · reps 10–14 |
| `lateral_lunge` «Affondo laterale» | `lateral_lunge` · Affondo laterale · cat `F2A37357` «Lateral Lunge» · «peso sul tallone, gamba tesa opposta» · reps 8–12 · · `lateral_lunge_pause` · Affondo laterale con pausa 2" · cus «Affondo laterale con pausa 2"», reps_only, adductors · «2 secondi in basso» · reps 8–12 · · `lateral_lunge_deficit` · Affondo laterale dal gradino · cus «Affondo laterale dal gradino», reps_only, adductors · «piede che lavora sul gradino» · reps 8–12 |
| `abduction` «Abduzioni» | `side_lying_abduction` · Abduzioni da sdraiato su un fianco · cat `DC59D143` «Lateral Leg Raises» · «gamba tesa, punta verso il basso» · reps 15–20 · · `side_lying_abduction_pause` · Abduzioni da sdraiato con pausa 2" · cus «Abduzioni da sdraiato con pausa 2"», reps_only, abductors · «2 secondi in alto» · reps 12–16 · · `fire_hydrant` · Abduzioni in quadrupedia · cus «Abduzioni in quadrupedia», reps_only, abductors · «ginocchio di lato, bacino fermo» · reps 12–16 |
| `mobility_hip_9090` «90/90 anche» | `hip_9090` · 90/90 anche · cus «90/90 anche», duration, other · «60" per lato, busto verso il ginocchio avanti» · hold 120–120 |
| `mobility_couch` «Couch stretch» | `couch_stretch` · Couch stretch · cus **«Couch Stretch»** (esiste già), duration, other · «60" per lato, glutei contratti» · hold 120–120 |
| `mobility_thoracic` «Rotazioni toraciche» | `thoracic_rotation` · Rotazioni toraciche · cus «Rotazioni toraciche», reps_only, other · «in quadrupedia, mano dietro la nuca» · reps 10–10 |

Regole di coerenza (test A2): `targets` di ogni variante contiene **ogni** genere richiesto dagli
slot che usano la sua scala (`push_up`: max2 e reps; `push_up_pause`: reps; le tenute: hold); i
titoli custom sono unici fra loro e diversi da ogni titolo del catalogo nelle fixture; `hollow_hold`
e `couch_stretch` hanno il titolo esatto dei custom esistenti (riuso, D-26); nessun id di catalogo
fuori dalle fixture (`catalog_bodyweight.json` + `catalog_extra.json`); il tipo del template
(`type` nella fixture per il catalogo, `exerciseType` per il custom) è `duration` se la variante ha
`targets.hold` e `reps_only` se ha `reps` o `max2`; `decline_push_up_pause` è **lo stesso oggetto**
nelle due scale; per ogni slot con `note` non vuota, nessuna cue della sua scala contiene la nota.

`src/lib/plan/lookup.ts`, tutte pure: `findSlot(slotId): Slot`, `findVariant(variantId): Variant`,
`scaleOf(slotId): Scale`, `routineOf(slotId): PlanRoutine` lanciano `Error('Unknown …: <id>')` su un
id ignoto; `nextVariant(scaleId, variantId): Variant | null` (null all'ultima variante);
`variantsOf(slotId): Variant[]` (la scala dello slot); `slotsUsingVariant(variantId): Slot[]` = gli
slot la cui **scala contiene** la variante (`decline_push_up_pause` → `mon_push_up`,
`thu_decline_push_up`, `thu_push_up_pause`); `allVariantsInUse(states): Variant[]` = le varianti
degli stati, **senza doppioni** (per `id`), nell'ordine di `PLAN` (routine, poi slot): con gli stati
iniziali sono 29 (30 slot, `hollow_hold` in due), di cui 11 custom.

`src/lib/plan/state.ts` (C-5): `initialSlotStates(now: number): SlotState[]` → uno per slot, con
`variantId = startVariantId`, `target = initialTarget`, `updatedAt = now`, `source: 'plan'`.

### 4.3 Note degli esercizi e builder — `src/lib/plan/routine-builder.ts` (C-4)

```ts
import type { RoutineWriteBody, RoutineWriteSet } from '@/lib/hevy/types';

/** The Hevy exercise note: plan note · variant cue · target text · side text. Never hand-written. */
export function targetNote(slot: Slot, state: SlotState, variant: Variant): string {
  const parts = [slot.note, variant.cue, targetText(state.target), slot.perSide ? perSideText(slot) : ''];
  return parts.filter((p) => p.length > 0).join(' · ');
}

/** Also shown by the dashboard (05a, 05b) and quoted in the review prompt (04). */
export function targetText(target: Target): string {
  switch (target.kind) {
    case 'max2': return `max-2 · soglia ${target.threshold}`;
    case 'reps': return target.min === target.max ? `${target.min} ripetizioni` : `${target.min}–${target.max} ripetizioni`;
    case 'hold': return target.seconds === target.maxSeconds ? `${target.seconds}"` : `${target.seconds}–${target.maxSeconds}"`;
  }
}

function perSideText(slot: Slot): string {
  return /lunge|rdl|calf|bridge|thrust|bss/.test(slot.id) ? 'per gamba' : 'per lato';
}

export function buildSet(target: Target): RoutineWriteSet {
  const empty = { type: 'normal' as const, weight_kg: null, reps: null, distance_meters: null, duration_seconds: null, custom_metric: null, rep_range: null };
  switch (target.kind) {
    case 'max2': return empty;
    case 'reps': return target.min === target.max ? { ...empty, reps: target.min } : { ...empty, rep_range: { start: target.min, end: target.max } };
    case 'hold': return { ...empty, duration_seconds: target.seconds };
  }
}

export function buildRoutineBody(routine: PlanRoutine, states: SlotState[], templateIds: TemplateIdMap, folderId: number | null): RoutineWriteBody {
  const exercises = routine.slots.map((slot) => {
    const state = states.find((s) => s.slotId === slot.id);
    if (!state) throw new Error(`Missing slot state for ${slot.id}`);
    const variant = findVariant(state.variantId);
    const templateId = templateIds[variant.id];
    if (!templateId) throw new Error(`Missing Hevy template for variant ${variant.id}`);
    return {
      exercise_template_id: templateId,
      superset_id: null,
      rest_seconds: slot.restSeconds,
      notes: targetNote(slot, state, variant),
      sets: Array.from({ length: slot.sets }, () => buildSet(state.target)),
    };
  });
  return { routine: { title: routine.title, folder_id: folderId, notes: routine.notes, exercises } };
}
```

Il set `max2` non ha valori: lo spec OpenAPI dichiara ogni campo del set nullable. Se nonostante ciò
Hevy rispondesse 400 al body alla fase D1, `buildSet` per `max2` scrive
`rep_range: { start: target.threshold - 4, end: target.threshold }`, lo snapshot si rigenera e il
caso si annota in AGENTS.md § 3: la scelta è presa qui, non in sessione.

### 4.4 Template assicurati — `src/lib/hevy/ensure-templates.ts` (C-6)

```ts
/**
 * Pure half: resolve what can be resolved without writing. Catalog variants are looked up by id only
 * (a renamed catalog exercise keeps its id; the title is checked by the scales test against the
 * fixtures); custom variants are matched by EXACT title among the account's templates (a custom
 * template can never be edited or deleted, so a near-miss would create a duplicate for ever) and
 * must have the expected type. Used read-only by the sync of spec 03.
 */
export function resolveTemplates(variants: Variant[], existing: HevyExerciseTemplate[]): { map: TemplateIdMap; missing: Variant[] } {
  const map: TemplateIdMap = {};
  const missing: Variant[] = [];
  for (const variant of variants) {
    const ref = variant.hevy; // local binding: narrowing survives inside the callbacks below
    if (ref.kind === 'catalog') {
      const found = existing.find((t) => t.id === ref.templateId);
      if (!found) throw new Error(`Catalog template ${ref.templateId} (${ref.title}) not found on the account`);
      map[variant.id] = found.id;
      continue;
    }
    const byTitle = existing.find((t) => t.title === ref.title);
    if (byTitle) {
      if (byTitle.type !== ref.exerciseType) throw new Error(`Custom template "${ref.title}" exists with type ${byTitle.type}, expected ${ref.exerciseType}`);
      map[variant.id] = byTitle.id;
      continue;
    }
    missing.push(variant);
  }
  return { map, missing };
}

/** Resolve, then create only the missing custom templates, in the order given. The returned map is complete. */
export async function ensureTemplates(hevy: HevyClient, variants: Variant[], existing: HevyExerciseTemplate[]): Promise<TemplateIdMap> {
  const { map, missing } = resolveTemplates(variants, existing);
  for (const variant of missing) {
    const ref = variant.hevy;
    if (ref.kind !== 'custom') continue; // resolveTemplates throws on an unresolved catalog variant
    const created = await hevy.createCustomExercise({
      title: ref.title,
      exercise_type: ref.exerciseType,
      equipment_category: 'none',
      muscle_group: ref.muscleGroup,
      other_muscles: [],
    });
    existing.push({ id: created.id, title: ref.title, type: ref.exerciseType, primary_muscle_group: ref.muscleGroup, secondary_muscle_groups: [], equipment: 'none', is_custom: true, thumbnail_url: null });
    map[variant.id] = created.id;
  }
  return map;
}
```

Un template di catalogo assente dall'account (catalogo cambiato) ferma la corsa con quel messaggio:
si corregge l'id in `scales.ts` con una GET reale (e la fixture) e si rilancia. Il test lo prova
(§ 6).

Le 9 custom create al primo upload (le varianti di partenza che non esistono): «Prone Y-T-W»,
«Rematore sotto il tavolo con pausa 2"», «Pike push-up con pausa in basso», «Push-up con pausa in
basso», «Reverse snow angel», «Bulgarian split squat (piede sul divano)», «Stacco rumeno a una
gamba», «90/90 anche», «Rotazioni toraciche». Riusate: «Hollowbody Hold», «Couch Stretch».

### 4.5 Stima della durata — `src/lib/plan/duration.ts`

`estimateRoutineMinutes(routine: PlanRoutine, states: SlotState[]): number` restituisce minuti non
arrotondati: una somma in **secondi interi**, divisa per 60 una volta sola. Riscaldamento 300 s; per
ogni slot `sets × (lavoro + restSeconds) − restSeconds` (l'ultima serie non recupera), dove lavoro
per set = 45 s per `reps` e `max2` (90 s se `perSide`: i due lati); `target.seconds + 5` per `hold`
(`× 2` se `perSide`); 60 s per lo slot con `scaleId === 'ytw'` (tre lettere). Il target è quello
dello stato dello slot. Con gli stati iniziali: Lun 34,9 · Mar 35,4 · Mer 28,5 · Gio 33,9 · Ven
40,0. Tetti (test A6, `toBeLessThanOrEqual`): 40 min; `wed_core` 30 min (D-46).

### 4.6 Caricamento — `src/lib/hevy/upload-plan.ts` e `scripts/hevy-upload.ts`

```ts
export interface UploadReport {
  folderId: number | null; // null only in a dry run with no folder yet
  folderCreated: boolean;
  templatesToCreate: string[]; // custom titles missing before the run (dry run: what would be created)
  templatesReused: string[]; // custom titles found by exact title
  templatesCreated: string[]; // equal to templatesToCreate after a write run, [] in a dry run
  routines: Array<{ key: RoutineKey; title: string; action: 'created' | 'updated' | 'dry-run'; hevyRoutineId: string | null }>;
}

function theOnly<T extends { id: unknown }>(items: T[], what: string): T | null {
  if (items.length > 1) throw new Error(`${items.length} ${what} found on the account: fix it by hand in the Hevy app and rerun`);
  return items[0] ?? null;
}

export async function uploadPlan({ hevy, states, dryRun, log }: { hevy: HevyClient; states: SlotState[]; dryRun: boolean; log: (line: string) => void }): Promise<UploadReport> {
  log(`mode: ${dryRun ? 'dry-run' : 'WRITE'}`);
  // 1. Folder by exact title (created once).
  const folders = await hevy.listRoutineFolders();
  let folder = theOnly(folders.filter((f) => f.title === FOLDER_TITLE), `folders titled "${FOLDER_TITLE}"`);
  // 2. Templates for every variant currently in use (lazy: later variants are created by "apply").
  const existing = await hevy.listExerciseTemplates();
  const variants = allVariantsInUse(states);
  const resolved = resolveTemplates(variants, existing); // throws on a missing catalog template
  const customInUse = variants.filter((v) => v.hevy.kind === 'custom');
  const templatesToCreate = resolved.missing.map((v) => v.hevy.title);
  const templatesReused = customInUse.filter((v) => !resolved.missing.includes(v)).map((v) => v.hevy.title);
  // 3. Routines by (folder, title): PUT when found, POST otherwise.
  const routines = await hevy.listRoutines();
  const findRoutine = (folderId: number, title: string) =>
    theOnly(routines.filter((r) => r.folder_id === folderId && r.title === title), `routines titled "${title}" in the plan folder`);
  const report: UploadReport = { folderId: folder?.id ?? null, folderCreated: false, templatesToCreate, templatesReused, templatesCreated: [], routines: [] };
  if (dryRun) {
    const provisional: TemplateIdMap = { ...resolved.map };
    for (const v of resolved.missing) provisional[v.id] = `<new:${v.id}>`;
    for (const routine of PLAN) {
      const found = folder ? findRoutine(folder.id, routine.title) : null;
      log(JSON.stringify(buildRoutineBody(routine, states, provisional, folder?.id ?? null), null, 2));
      report.routines.push({ key: routine.key, title: routine.title, action: 'dry-run', hevyRoutineId: found?.id ?? null });
    }
    log(`would create: folder ${folder ? 'no' : 'yes'}, ${templatesToCreate.length} custom templates (${templatesToCreate.join(', ')}), reused ${templatesReused.length} (${templatesReused.join(', ')})`);
    return report;
  }
  if (!folder) {
    folder = await hevy.createRoutineFolder(FOLDER_TITLE);
    report.folderCreated = true;
    log(`created folder → ${folder.id}`);
  }
  report.folderId = folder.id;
  const templateIds = await ensureTemplates(hevy, variants, existing); // logs nothing itself: the report lists the titles
  report.templatesCreated = templatesToCreate;
  for (const routine of PLAN) {
    const body = buildRoutineBody(routine, states, templateIds, folder.id);
    const found = findRoutine(folder.id, routine.title);
    const saved = found ? await hevy.updateRoutine(found.id, body) : await hevy.createRoutine(body);
    report.routines.push({ key: routine.key, title: routine.title, action: found ? 'updated' : 'created', hevyRoutineId: saved.id });
    log(`${found ? 'updated' : 'created'} ${routine.title} → ${saved.id}`);
  }
  return report;
}
```

`uploadPlan` non cattura errori: ogni scrittura riuscita è già nel log prima della successiva.
**Fallimento a metà** (403 `exceeds-custom-exercise-limit`, 403 «Routine limit exceeded», 400 sul
body, 5xx su una POST, che il client non ritenta): lo script cattura `HevyApiError`, stampa
`failed: <status> <code>` e il report parziale, con 403 sui custom anche i titoli ancora mancanti,
ed esce con codice 1. Il rilancio **riprende** da dove si era fermato: cartella e custom si trovano
per titolo, le routine per cartella e titolo.

`scripts/hevy-upload.ts` (forma `main()` della 01): legge
`EnvSchema.pick({ HEVY_API_KEY: true }).parse(process.env)` (D-36), crea il client,
`states = initialSlotStates(Date.now())`, `const dryRun = process.argv.includes('--dry-run')`
(niente parser: pnpm può passare un `--` letterale, e un parser che si ferma a `--` perderebbe il
flag), stampa il report come tabella e, in chiusura, la riga `plan uploaded at <ISO>` (la 03 non la
usa: ricava la data dal `created_at` delle routine su Hevy). Se è impostata `HEVY_CAPTURE_DIR`,
passa a `createHevyClient` una `fetch` che salva il body grezzo di ogni risposta POST/PUT in
`<dir>/<n>-<metodo>-<percorso>.json` (cartella **fuori dal repo**): serve al collaudo per leggere la
busta reale e ricavare le fixture (§ 7 G). Script di package.json:
`"hevy:upload": "tsx --env-file-if-exists=.env.local scripts/hevy-upload.ts"`.

**D-40**: lo script usa **sempre** gli stati iniziali del piano ed è lo script di nascita: si lancia
solo prima del primo «Applica». La 04 aggiunge `pnpm hevy:upload -- --from-db` (stati da
`slot_state`) e il rifiuto senza flag quando `slot_state` ha righe `source = 'apply'`; questa spec
non ha un DB e lo dice in `doc/guide/piano.md`.

Quando «Applica» (04) cambia una variante, la 04 riusa `ensureTemplates` con la sola variante nuova
e `buildRoutineBody` con gli stati aggiornati: **questa spec non prevede un secondo percorso**. La
03 usa `resolveTemplates` in sola lettura (T-7).

## 5. Vincoli e trappole

- **AGENTS.md § 3 Hevy** — qui vuol dire che: il PUT rimanda **tutti** gli esercizi della routine
  (mai una patch) e **sovrascrive ogni modifica fatta a mano** nell'app Hevy (un esercizio aggiunto,
  un recupero cambiato): il piano si cambia nel codice, mai nell'app; `listRoutines` percorre tutte
  le pagine da 10; un custom si crea una volta sola, per titolo esatto, e il titolo non si cambia
  mai dopo (cambiarlo nel codice creerebbe un doppione su Hevy: il test A2 fissa i titoli con uno
  snapshot); una routine spostata fuori dalla cartella o rinominata nell'app non si trova più e il
  rilancio ne crea una nuova (permanente): si rimette a posto nell'app prima di rilanciare. Le
  chiamate reali non si lanciano a xx:00.
- **AGENTS.md § 2.1** — qui vuol dire che `src/lib/hevy/ensure-templates.ts` e `upload-plan.ts` sono
  casi d'uso che conoscono il piano (importano `src/lib/plan`); `client.ts` e `schemas.ts` restano
  senza import dal piano.
- **AGENTS.md § 4 Plan data** (stub scritto al bootstrap; § 9 lo verifica) — ogni modifica a
  `plan.ts` o `scales.ts` rigenera gli snapshot e va riletta: i titoli custom e gli id di catalogo
  sono **contratti con Hevy**.
- **D-4** — una variante nuova deve dichiarare `quiet: true, simple: true`: il tipo lo impone; la
  revisione non può uscire dalle scale (04 valida gli id).
- `rest_seconds: 0` sulla mobilità è legittimo (nullable integer).
- Hevy mostra le note dell'esercizio nella routine: ogni nota generata sta in **140 caratteri**
  (test, per ogni variante di ogni scala).
- Le fixture `catalog_bodyweight.json` e `catalog_extra.json` sono del 2026-10-23/24: se Hevy
  rinomina un esercizio del catalogo, `resolveTemplates` non fallisce (cerca per id) ma il test A2
  (titolo) sì → si aggiorna la fixture con una GET reale e si annota la data nel README delle fixture.
- Fra la fase D1 e la D2 si aspetta **almeno un minuto** e si lancia prima un `--dry-run`: deve dire
  «0 custom da creare». Se Hevy non elencasse ancora i custom appena creati, il rilancio li
  ricreerebbe, e sarebbe permanente.

## 6. Test

Il client finto della serie: `tests/helpers/fake-hevy.ts`, `createFakeHevy({ templates, folders,
routines })` restituisce `{ client: HevyClient, writes: Array<{ method: string; args: unknown[] }> }`.
Implementa con liste mutabili `listExerciseTemplates`, `createCustomExercise` (id `custom-<n>`,
spinge nella lista), `listRoutineFolders`, `createRoutineFolder` (id progressivo), `listRoutines`,
`getRoutine`, `createRoutine`, `updateRoutine` (salvano `folder_id`, titolo ed esercizi dal body);
gli altri metodi lanciano `Error('not used by this fake')`. Ogni scrittura finisce in `writes`.
`templates` di partenza nei test = `catalog_bodyweight.json` + `catalog_extra.json` + i due custom
di `exercise_templates_page.json`. La 03 e la 04 lo riusano così com'è.

| Livello | Test | Mutazione che lo prova portante |
|---|---|---|
| Unit | `plan.test.ts`: 5 routine, 30 slot, 27 `reviewed`, ogni `startVariantId` nella sua scala, `expect(PLAN).toMatchSnapshot()`, snapshot di `targetNote` per ogni slot nello stato iniziale, `estimateRoutineMinutes` ≤ 40 (wed ≤ 30) con i cinque valori di § 4.5 a un decimale. | `sets: 5` su `mon_push_up` → lo snapshot di `PLAN` va rosso; `restSeconds: 120` sugli slot a 90 s → Ven 42,5 → rosso. |
| Unit | `scales.test.ts`: id di catalogo presenti nelle due fixture con lo stesso titolo; `targets` completi per genere di slot; titoli custom unici e non in catalogo; tipo del template coerente con i target; `hollow_hold` = «Hollowbody Hold», `couch_stretch` = «Couch Stretch»; `decline_push_up_pause` condiviso per riferimento; nessuna cue contiene la `note` dello slot della sua scala. | Cambiare un id → rosso; togliere `max2` da `push_up` → rosso; `wall_sit` su `9694DA61` → rosso; `note: 'tempo 3-1-1'` su `tue_squat` → rosso. |
| Unit | `state.test.ts`: `initialSlotStates(1000)` → 30 stati, `variantId = startVariantId`, `target = initialTarget`, `updatedAt = 1000`, `source = 'plan'`. | Usare `Date.now()` → rosso. |
| Unit | `routine-builder.test.ts`: `buildSet` per i tre generi e il caso `min === max`; `buildRoutineBody` snapshot delle 5 routine con `templateIds` finti (`T_<variantId>`) e `RoutineWriteBodySchema.parse(body)` verde; `rest_seconds: 0` sulla mobilità; `targetText` dei tre generi (`max-2 · soglia 15`); nota ≤ 140 per ogni slot × ogni variante della sua scala con il target `targets[kind]`; errore su stato mancante e su template mancante. | `duration_seconds: target.maxSeconds` → snapshot rosso; togliere `superset_id` → `parse` rosso. |
| Unit | `ensure-templates.test.ts` con `createFakeHevy`: `resolveTemplates` sugli stati iniziali → 9 `missing`, mappa con 20 voci; 2 custom esistenti → 9 creazioni e mappa completa (29 voci); rilancio → 0; titolo «prone y-t-w» minuscolo NON riusa «Prone Y-T-W»; custom omonimo con `type: 'duration'` al posto di `reps_only` → errore; id di catalogo assente dalla lista → errore con l'id nel messaggio. | Confronto `toLowerCase()` → rosso; togliere il controllo del tipo → rosso. |
| Unit | `upload-plan.test.ts` con `createFakeHevy`: prima corsa (1 cartella, 9 custom, 5 POST, `templatesCreated` 9 titoli), seconda corsa (0, 0, 5 PUT, `templatesReused` 11), dry-run (0 `writes`, `templatesToCreate` 9, `folderId: null`, prima riga del log `mode: dry-run`), routine omonima in un'altra cartella → POST e non PUT; due cartelle con il titolo del piano → errore. | Match senza `folder_id` → rosso; `writes.length` nel dry-run > 0 → rosso. |
| Unit | `lookup.test.ts`: `nextVariant` all'ultima variante → `null`; `slotsUsingVariant('decline_push_up_pause')` → `mon_push_up`, `thu_decline_push_up`, `thu_push_up_pause`; `allVariantsInUse(initialSlotStates(0))` → 29 elementi, 11 custom, `hollow_hold` una volta sola; `findVariant('nope')` lancia. | `nextVariant` che restituisce la prima variante al posto di `null` → rosso; togliere la deduplica → 30 → rosso. |

## 7. Collaudo guidato

Fasi, una per messaggio, esito atteso dichiarato prima; **le scritture reali partono solo dopo l'OK
dell'owner**, dichiarate con il loro effetto.

- **0 — Fixture mancante** (prima di scrivere codice, dopo l'OK): **una GET reale di sola lettura**
  `GET /v1/exercise_templates/425805F4` (gratis, nessun effetto), via `curl` con l'header `api-key`
  o con un one-liner `tsx` sul client della 01. La risposta nuda va in
  `tests/fixtures/hevy/catalog_extra.json` come array di un elemento, e il README delle fixture dice
  «real, read 2026-<data>». Esito atteso: `title` «Inverted Row», `type` `reps_only`, `is_custom`
  false (D-44). Se il tipo fosse diverso, ci si ferma: `table_row` cambierebbe mappa e serve una
  decisione dell'owner.
- **A — Invarianza**: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` verdi; `pnpm
  hevy:whoami` ancora una riga (2 GET reali: lettura, gratis).
- **C — Comportamento nuovo (senza effetti)**: `pnpm hevy:upload -- --dry-run` (circa 7 GET reali:
  cartelle, 5 pagine di template, routine; lettura, gratis; non a xx:00). Esito atteso: **prima riga
  `mode: dry-run`** (se dicesse `mode: WRITE` si interrompe con Ctrl+C prima di qualunque
  scrittura), i 5 body stampati, 0 scritture, la riga «would create» con i 9 custom da creare e i 2
  riusati.
- **D — Scrittura reale 1** (dopo l'OK): `HEVY_CAPTURE_DIR=<cartella fuori dal repo> pnpm hevy:upload`.
  Effetto: le stesse ~7 GET + 1 POST cartella (permanente: niente DELETE), 9 POST template custom
  (**permanenti**, consumano il limite custom non documentato), 5 POST routine (permanenti); gratis.
  Esito atteso: report con 5 `created`; su Hevy (app) la cartella «Coach · Corpo libero» con le 5
  routine, note leggibili, set con range/tempo. **Prima** della corsa si rilegge il dry-run: i
  titoli custom sono definitivi. Se la corsa si ferma a metà (§ 4.6), si legge il log, si risolve
  (es. custom da liberare a mano nell'app) e si rilancia: riprende da dove si era fermata.
- **D — Scrittura reale 2** (almeno un minuto dopo; prima un `--dry-run` che dica «0 custom da
  creare»): rilancio di `pnpm hevy:upload` con `HEVY_CAPTURE_DIR`. Effetto: ~7 GET + 5 PUT
  (sostituzione completa delle 5 routine); gratis. Esito atteso: 5 `updated`, 0 custom creati, 0
  cartelle; su Hevy nessun doppione. Dai body catturati si annota in AGENTS.md § 3 quale busta
  restituisce `POST/PUT /v1/routines` (`{ routine }` o nuda) e il formato dell'`id` di un custom
  creato (atteso: UUID).
- **F — Giro guidato (owner, sull'app Hevy)**: aprire ogni routine e guardare 5 cose: (1) l'ordine
  degli esercizi è quello del piano; (2) le note dicono max-2 / range / per gamba dove previsto;
  (3) i recuperi (90/60/45/0); (4) i set a tempo mostrano i secondi; (5) i custom hanno nome
  italiano e tipo giusto (ripetizioni vs tempo). Il bug sarebbe: un esercizio con il template
  sbagliato, una nota mancante, un set con ripetizioni dove serviva tempo.
- **G — Ripristino**: nulla da togliere (le routine restano: sono il prodotto). Le fixture
  sintetiche `routine_created.json`, `routine_folder_created.json`, `custom_exercise_created.json`
  si sostituiscono con le risposte reali catturate e **ripulite**: id sostituiti con quelli finti
  attuali, `created_at`/`updated_at` sostituiti con `2026-10-25T07:00:00Z`; nel README della
  cartella la riga diventa «real shape, ids and dates replaced (spec 02 verification)». La cartella
  `HEVY_CAPTURE_DIR` si cancella.

Che cosa NON copre: la sincronizzazione (03), «Applica» (04).

## 8. Messa in esercizio

- Env: `HEVY_API_KEY` (Development). DDL: nessuno.
- Misura di chiusura: su Hevy esistono cartella e 5 routine; `pnpm hevy:upload` rilanciato fa 5 PUT.
- In esercizio = PR su `main` + collaudo annotato in CLAUDE.md (D-37). **Da qui l'owner si allena.**

## 9. Documentazione

- `CLAUDE.md`: Latest; la riga «Plan and variant scales (02)» di Key Features resta e perde
  «planned» nel titolo della sezione quando tutte le righe saranno vere (per ora: «(planned; 02 is
  real)»).
- `doc/guide/piano.md` (nasce qui, ≤ 200 righe): come si legge `plan.ts` e `scales.ts`, le regole
  delle note (§ 3 h), la stima di durata, come si aggiunge una variante (titolo definitivo, `quiet`,
  `simple`, `targets` per genere, test, decisione dell'owner), quando si rilancia l'upload (solo
  prima del primo «Applica», D-40; dopo, la forma `--from-db` della 04).
- `AGENTS.md`: § 4 Plan data esiste già (6 righe): si verifica che dica il vero (tetti 40/30,
  snapshot, titoli come contratti) e si corregge; § 3 Hevy: busta delle routine e formato dell'id
  custom misurati al collaudo, e la riga sulle modifiche a mano sovrascritte dal PUT; § 10 Commands:
  `hevy:upload` c'è già.
- `README.md` (inglese, D-35): `## The plan` (le 5 routine in una tabella: giorno, titolo italiano,
  esercizi) e `## Uploading the routines` (`pnpm hevy:upload -- --dry-run`, poi `pnpm hevy:upload`;
  only before the first «Applica»).
- `WORKFLOW.md` § 3: la riga «Hevy CLI» conferma `pnpm hevy:upload [-- --dry-run]` come reale dalla
  02 e aggiunge «writes to Hevy: dry run first».
- `tests/fixtures/hevy/README.md`: la riga di `catalog_extra.json` (reale) e la riga delle fixture
  sintetiche sostituite.

## 10. Fuori ambito, dichiarato

- Registrare in Turso gli id delle routine e dei template → **03** (`hevy_routines`, `hevy_templates`).
- Cambiare variante e target in esercizio → **04** (`computeNextSlotState`, `applyProposal`).
- Varianti oltre le scale (archer, pseudo planche, pistol libero, nordic) → mai in v1 (D-4);
  CLAUDE.md § Not yet present («harder variants deliberately excluded»).
- Cancellare custom o routine su Hevy: l'API non lo permette; si fa a mano nell'app → CLAUDE.md § Not
  yet present.
- Il piano della garage gym → D-8.

## 11. Prompt per implementarla

Modello: Claude Opus 5.5, effort high. Taglia: **L** (dati del piano + 9 moduli + harness
`fake-hevy` + script + collaudo con scritture reali).

```
Ciao, in questa sessione implemento la spec doc/v1/02-piano-scale-e-caricamento-routine.md.

Contesto, prima di toccare qualsiasi file:
- Leggi WORKFLOW.md (regole di sessione e collaudo; § 3 dice come si applicano qui)
- Leggi AGENTS.md (CLAUDE.md è già nel contesto), in particolare § Hevy
- Leggi COMMENTS.md e DEVELOPMENT_GUIDELINES.md: il codice che scrivi li segue
- Leggi doc/v1/00-INDICE.md per intero (decisioni D-4…D-7, D-18…D-20, D-24…D-27, D-36, D-39,
  D-40, D-44…D-47; scelte T-7, T-8, T-18; contratti C-1…C-6, C-15; regole della serie) e poi la
  spec 02 INTERA
- Verifica con un grep che i contratti della 01 esistano con quel nome e quella firma
  (createHevyClient, gli schemi e i tipi RoutineWriteBody/RoutineWriteSet, EnvSchema in
  env-schema.ts, fakeFetch e fixture in tests/helpers, le fixture)

Prima della prima modifica: `git branch --show-current` deve dire main (la 01 è mergiata);
crea feature/02-plan-upload. Crea SESSION_NOTES.md. Il primo passo è la fase 0 del collaudo
(una GET reale di sola lettura, dopo il mio OK): senza catalog_extra.json i test di § 6 non
possono essere verdi.

Implementa la spec 02 esattamente come scritta: § 4.1 e § 4.2 sono DATI e si trascrivono
tali e quali (ogni id di catalogo, ogni titolo custom, ogni target, ogni recupero); § 4.3–4.6 il
codice; § 6 i test con le mutazioni (compreso tests/helpers/fake-hevy.ts); § 7 il collaudo fase
per fase, esito atteso prima — le scritture reali su Hevy (fase D) partono SOLO dopo il mio OK
esplicito, prima mi mostri il dry-run e la sua prima riga deve dire `mode: dry-run`; § 9 la
documentazione. Non decidere nulla che la spec ha già deciso; se trovi un punto che la spec non
copre, fermati e chiedimelo con AskUserQuestion. Le etichette della serie non entrano nel
codice.

Alla fine: riassumi il diff, scrivi lo stato nella riga «Stato» della spec e in § 5
dell'indice, aggiorna CLAUDE.md → Latest, poi chiedimi l'OK per il commit (uno solo).
```
