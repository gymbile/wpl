# WPL Compile-Conformance Corpus

Shared fixtures that every WPL-AI compiler implementation must pass.
Each fixture defines a DSL source file and its expected compiled JSON output.
When `wpl-ai` (TypeScript) and `wpl-ai-ex` (Elixir) produce different
results for the same fixture after normalization, **one of them is wrong** —
the fixture and this document answer which.

## Directory Layout

```
conformance/compile/
  README.md
  fixtures/
    <category>/<name>/
      source.wpl        # DSL input (raw text)
      expected.json     # expected compiled JSON (pretty-printed, sorted keys)
      meta.json         # { "since", "description", "category", ["known_divergence"] }
```

`category` is a directory name such as `basic`, `exercise`, `cardio`,
`nutrition`, `meditation`, `recovery`, `habit`, `sub_plan`, `periodization`,
`personalization`, `progress`, `athlete_thresholds`, `regression`.

`name` is a kebab-case slug.

## Normalization Rules

Both runners apply the following transformations **before** comparing output
to `expected.json`.  The `expected.json` files are stored in already-
normalized form so that diffing is straightforward.

### 1. Deep-sort JSON keys alphabetically

All object keys at every nesting level are sorted alphabetically.
This eliminates key-ordering differences between compilers.

### 2. Normalize auto-generated IDs

Any string value at a key named `id` that matches the pattern
`^[a-z0-9_]+_\d+$` (e.g. `phase_1`, `week_2`, `day_1`, `exercise_1`,
`cardio_1`, `nutrition_1`, `main_block`, `warmup_block`, `cooldown_block`,
`nutrition_block`) is replaced with the literal `"<AUTO_ID>"`.

Rationale: the two compilers may auto-number IDs differently.  Fixtures that
require stable IDs should use explicit `id` syntax in the DSL (if supported
by the parser); if not, they should be limited to cases that do not expose
ID-numbering divergence.

### 3. Normalize UUID plan IDs

Any string value at a key named `id` that matches the UUID v4 pattern
`^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`
is replaced with the literal `"<UUID>"`.

Rationale: the plan-level `id` is a randomly generated UUID and will always
differ between runs and compilers.

### 4. Strip runtime-generated metadata timestamps

The fields `created_at` and `updated_at` inside any `metadata` object are
removed before comparison.  They are always set to the current wall-clock
time and will never match across compiler runs.

### 5. Coerce whole-number floats to integers

Any numeric value that is a float with no fractional part (e.g. `1.0`, `6.0`)
is coerced to its integer equivalent before comparison.  This handles the case
where one compiler emits an integer and the other emits a float for the same
logical value (e.g. Elixir may emit `1.0` for a value that TS emits as `1`).

### 6. `version` field asserted as-is

The top-level `"version"` field is compared verbatim.  A compiler emitting
an older schema version will fail fixtures that set `version` to a newer
version.  This is intentional: version-locked fixtures document when a
feature was introduced.

## Rules No Longer Needed (resolved divergences)

The following normalization rules existed to paper over compiler divergences
and were removed in `wpl-ai-ex v1.6.1` once both compilers aligned:

- **`metadata.language` strip** — TS compiler always emits `"language": "en"`;
  Elixir compiler previously omitted it.  Fixed: Elixir now defaults `language`
  to `"en"` (TS parity).  `expected.json` files include `"language": "en"` and
  runners no longer strip it.

- **Activity display `name` strip** — TS compiler auto-derives `name` from the
  exercise_ref / modality / category token for Exercise, Cardio, Nutrition,
  Meditation, Recovery, and Habit activities; Elixir previously omitted `name`.
  Fixed: Elixir now derives `name` using the same `humanise()` logic (split on
  `[_\s-]+`, title-case each word, uppercase known acronyms HIIT/AMRAP/EMOM/
  RPE/RIR/1RM).  `expected.json` files include the derived `name` and runners
  no longer strip it.

## `meta.json` Schema

```json
{
  "since": "1.0.0",
  "description": "Human-readable description of what this fixture tests.",
  "category": "basic",
  "known_divergence": [
    {
      "field": "some.field",
      "ts_value": "...",
      "ex_value": "...",
      "note": "Explanation and which normalization rule handles it."
    }
  ]
}
```

`known_divergence` is optional.  When present, each entry describes a field
where the two compilers disagree, what each emits, and whether normalization
handles it.  If normalization fully handles the divergence, both runners
should still pass.  If normalization does **not** handle it, the fixture is
marked as pending investigation and both runners should `skip` it.

## Compile Helpers

Each compiler repo ships a helper script for compiling a single source file:

- **TypeScript** (`wpl-ai`): `node scripts/compile.mjs <path/to/source.wpl>`
  Prints pretty-printed JSON to stdout. Requires a built `dist/`.

- **Elixir** (`wpl-ai-ex`): `mix run scripts/compile.exs <path/to/source.wpl>`
  Prints compact JSON to stdout. Pipe through `jq .` for pretty output.

## Adding a Fixture

1. Create `fixtures/<category>/<name>/source.wpl`.
2. Compile with both helpers and capture output.
3. Apply normalization (sort keys, replace IDs, strip timestamps).
4. Verify both outputs match after normalization.  If not, document in
   `meta.json` under `known_divergence` and simplify the fixture.
5. Save the normalized output as `expected.json`.
6. Write `meta.json`.

## Running the Suite

- **TypeScript runner**: `cd wpl-ai && npm test -- conformance`
- **Elixir runner**: `cd wpl-ai-ex && mix test test/conformance_test.exs`

The corpus directory is resolved relative to each runner's source tree
(`../../wpl/conformance/compile/fixtures/` from `__tests__/` or `test/`).
Set the `WPL_CORPUS_DIR` environment variable to override the path.

## Divergence Policy

When a fixture's `expected.json` exposes a genuine behavioral difference
between the two compilers (after normalization), that difference is a
**compiler bug**.  File an issue in the offending compiler's repo.  Do not
paper over compiler bugs by relaxing normalization rules.
