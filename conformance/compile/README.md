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

### 5. Strip `metadata.language` (known compiler divergence)

The `language` field inside `metadata` is present in TS output (`"en"` by
default) but absent in Elixir output.  It is stripped before comparison.
This is a **known compiler divergence** tracked in the issue tracker.

### 6. Strip auto-derived activity display names (known compiler divergence)

The `name` field on activity objects whose `type` is `"exercise"`,
`"cardio"`, or `"nutrition"` is stripped before comparison when the name
was not explicitly authored in the DSL.  The TS compiler auto-derives
display names from the exercise reference (e.g. `push_up` → `"Push Up"`,
`running` → `"Running"`); the Elixir compiler does not.  This is a
**known compiler divergence** tracked in the issue tracker.

Fixtures with explicit DSL-authored activity names should **not** apply
this rule.  Such fixtures should set `strip_activity_display_names: false`
in `meta.json` (default is `true` to enable the strip).

### 7. Exact floating-point match

Floating-point values are compared exactly.  Fixtures must not produce
float drift.  Use integer-representable floats (e.g. `1.6`, `2.2`) where
possible.

### 8. `version` field asserted as-is

The top-level `"version"` field is compared verbatim.  A compiler emitting
an older schema version will fail fixtures that set `version` to a newer
version.  This is intentional: version-locked fixtures document when a
feature was introduced.

## `meta.json` Schema

```json
{
  "since": "1.0.0",
  "description": "Human-readable description of what this fixture tests.",
  "category": "basic",
  "known_divergence": [
    {
      "field": "metadata.language",
      "ts_value": "en",
      "ex_value": null,
      "note": "TS emits language; Elixir does not. Normalized away."
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
