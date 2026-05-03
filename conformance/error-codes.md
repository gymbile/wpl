# WPL Validator Error Codes

This document defines every error code a conformant WPL validator must emit. Both reference implementations (`@gymbile/wpl-validator`, `wpl_validator`) produce these codes verbatim.

## Format

Each error has:

- **`code`** — stable identifier (UPPER_SNAKE_CASE in JSON, `:lower_snake_case` atom in Elixir).
- **`path`** — RFC 6901 JSON Pointer to the offending node.
- **`severity`** — `error` or `warning`. `warning` does not make a plan invalid.
- **`message`** — human-readable; not part of the conformance contract (validators may localize).
- **`meta`** — code-specific fields. Listed below per code; consumers may rely on the keys documented here.

## Pass 1 — Schema violations

### `SCHEMA_VIOLATION`

Raised by JSON Schema validation (Pass 1). When a `SCHEMA_VIOLATION` occurs, semantic checks (Pass 2) are skipped entirely.

- **severity**: `error`
- **meta**:
  - `keyword` (string) — the JSON Schema keyword that failed (`required`, `enum`, `type`, `const`, `additionalProperties`, etc.)
  - `params` (object) — keyword-specific params (e.g., `{ missing_property: "id" }` for `required`, `{ allowed_values: ["..."] }` for `enum`)

#### Path conventions for schema violations

The `path` is the RFC 6901 JSON Pointer to the **failing instance node**, following ajv's `instancePath` semantics:

| Keyword                  | `path` points at                          | `meta.params` carries the offender |
|--------------------------|-------------------------------------------|------------------------------------|
| `required`               | the parent object missing the property    | `{ missing_property: "<name>" }`   |
| `additionalProperties`   | the parent object that disallows extras   | `{ additional_property: "<name>" }`|
| `type`, `const`, `enum`  | the offending value itself                | keyword-specific                   |
| `pattern`, `minLength`, etc. | the offending value itself            | keyword-specific                   |

Validators built on libraries that report a different convention (e.g. `ex_json_schema` reports `additionalProperties` at the offending property, not the parent) **must normalize** to the table above before emitting the error. The conformance suite is the contract; library idiosyncrasies are not.

#### `oneOf` failures: drill into the best-matching branch

When validation fails inside a `oneOf` (e.g. an inner enum violation on an `ExerciseActivity`, which is one branch of `Activity`), validators **must surface the deepest-specific failure in the best-matching branch**, not a bare `oneOf` failure at the parent.

"Best-matching" is defined as the branch with the fewest inner errors (i.e. closest match to the supplied data, typically determined by const discriminators like `type: "exercise"`).

This matches ajv's native behavior. ex_json_schema reports a single `OneOf` error at the parent by default; the reference Elixir validator post-processes these into branch-specific errors so paths and keywords agree across validators.

## Pass 2 — Semantic invariants

### `DUPLICATE_ID`

Two siblings within the same scope share an `id`.

- **severity**: `error`
- **path**: pointer to the **second** occurrence
- **meta**:
  - `duplicate_id` (string) — the offending id value
  - `scope` (string) — `plan` | `phase:<phase_id>` | `week:<week_id>` | `day:<day_id>`
  - `first_occurrence` (string) — JSON Pointer to the first occurrence

### `UNRESOLVED_REF`

A `*_ref` value (e.g. `exercise_ref`, `meal_ref`) does not exist in the provided catalog. **Skipped entirely if no catalog is provided.**

- **severity**: `error`
- **path**: pointer to the activity's `*_ref` field
- **meta**:
  - `ref_kind` (string) — `exercise` | `meal` | `meditation` | etc.
  - `ref_value` (string) — the unresolvable id

### `CYCLIC_SUBPLAN`

Sub-plan references form a cycle.

- **severity**: `error`
- **path**: pointer to the activity that closes the cycle
- **meta**:
  - `cycle` (array of strings) — the chain of plan ids forming the cycle

### `EMPTY_PHASES_FOR_TYPE`

Plan of type `workout` or `hybrid` has zero phases.

- **severity**: `error`
- **path**: `/plan/phases`
- **meta**:
  - `plan_type` (string)

### `MISSING_EXERCISE_REF`

Activity of type `exercise` lacks `exercise_ref`. (May also be raised by JSON Schema — implementations may classify this either way; conformance treats both as equivalent.)

- **severity**: `error`
- **path**: pointer to the activity
- **meta**: none

### `INVALID_PRESCRIPTION`

Exercise prescription is malformed.

- **severity**: `error`
- **path**: pointer to the prescription object
- **meta**:
  - `reason` (string) — one of `missing_type`, `unknown_type`, `sets_reps_requires_sets_or_reps`, `time_requires_duration`
  - `prescription_type` (string, optional) — present when `reason` is `unknown_type`

### `INVALID_PERSONALIZATION_RULE`

Personalization rule has an unknown action type, an unknown scope, or a malformed condition.

- **severity**: `error`
- **path**: pointer to the offending rule (or action within it)
- **meta**:
  - `reason` (string) — `missing_required_field` | `invalid_condition` | `invalid_action_type` | `invalid_action_scope` | `actions_must_be_non_empty_list`
  - `field` (string, optional) — name of the offending field
  - `value` (any, optional) — the rejected value

### `INVALID_POINTS_RULE`

Points-system rule has missing/invalid points value.

- **severity**: `error`
- **path**: pointer to the points rule
- **meta**:
  - `reason` (string) — `missing_action` | `missing_points` | `points_must_be_non_negative_integer`

### `PHASE_DURATION_MISMATCH`

Phase's `duration` and the length of its `weeks` array disagree.

- **severity**: `warning`
- **path**: pointer to the phase
- **meta**:
  - `declared_value` (number)
  - `declared_unit` (string) — `days` | `weeks`
  - `weeks_count` (number)

## Stability

Error codes are part of the conformance contract. A code, once published, may not be renamed or have its meaning changed within a major version. New codes may be added in minor versions; consumers must tolerate unknown codes gracefully.
