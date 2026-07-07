# WPL canonical data

These files are the **normative source of truth** for the exercise-name
vocabulary and the fuzzy-matcher vocabulary used across the WPL ecosystem.

- `exercises.json` — canonical exercise names grouped by category. The flat
  catalog is the concatenation of all category arrays (consumers derive it).
- `matcher-vocab.json` — qualifier tokens + short-plural overrides used by the
  enforcement matchers.
- `goal-categories.json` — recommended vocabulary for `Goal.category` (incl.
  `general_fitness` and a `custom` escape hatch). **Soft-validated:**
  `Goal.category` stays free-form; validators SHOULD warn (not reject) on
  off-list values.
- `dietary-tags.json` — recommended vocabulary for the optional
  `NutritionActivity.dietary_tags` field. Soft-validated the same way.
- `*.schema.json` — JSON Schemas validating the above; CI enforces them.

**Edit here only.** Every consumer repo (`wpl-ai`, `wpl-ai-ex`,
`gymbile_backend`, `wpl-validator-ts`, `wpl-validator-ex`, `wpl-eval`) vendors
a copy of these files and runs a codegen step to produce its native module.
Consumers are *generated* — never hand-edit a consumer's catalog/vocab.
Bumping these files is a deliberate vendor-update (like a schema sync), pinned
per consumer by a `*-version.txt`.
