# Hevy API fixtures

Captured on 2026-10-23 with read-only GET calls against the real Hevy API (OpenAPI 0.0.1, read from
`https://api.hevyapp.com/docs/` the same day) and cleaned of personal data: user id, username, name and
URL are invented; workout ids are fixed fake UUIDs; custom exercise ids are fake. Catalog exercise
template ids (8-character, e.g. `392887AA`) are Hevy's public catalog and are real.

| File | Origin |
| --- | --- |
| `user_info.json` | real shape, invented values |
| `exercise_templates_page.json` | real catalog rows (8) + 2 custom rows with fake ids |
| `catalog_bodyweight.json` | every catalog template with `equipment` `none` or `other` (real, public) |
| `workout.json`, `workouts_page.json` | one real bodyweight workout, id replaced |
| `workout_events.json` | the same workout as an `updated` event + a `deleted` event built from the schema |
| `exercise_history.json` | real entries for Push Up, workout ids replaced |
| `routine_folders_page_empty.json`, `routines_page_empty.json` | real: the account had no folders and no routines on 2026-10-23 |
| `routine_folder_created.json`, `routine_created.json`, `custom_exercise_created.json` | **synthetic**, built from the OpenAPI response schemas: no routine had been created yet. The retirement of spec 02 replaces them with real responses (ids replaced). |
| `error_403_custom_limit.json`, `error_404_routine.json` | error bodies from the OpenAPI examples |

Never add a real id, name or date of the owner here: tests use these files, and the repo may be
read by agents.
