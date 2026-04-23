# WPL Conformance Suite

Shared test fixtures every WPL validator implementation must pass. The suite is the contract: when `wpl-validator-ts` and `wpl-validator-ex` produce different results for the same fixture, **one of them is wrong** — the fixture answers which.

## Layout

```
conformance/
├── error-codes.md        # canonical error code reference
├── valid/                # plans every validator must accept
│   └── *.json
└── invalid/              # plans every validator must reject
    ├── <name>.json
    └── <name>.expected.json
```

## Running the Suite

Each validator vendors a copy of this directory and runs its own test runner:

- TypeScript: `tests/conformance.test.ts` (uses Vitest)
- Elixir: `test/conformance_test.exs` (uses ExUnit)

A drift-check CI job in each validator repo fails if the vendored copy is out of sync with the latest tagged release of `gymbile/wpl`.

## `.expected.json` Format

For every `invalid/<name>.json`, a sibling `invalid/<name>.expected.json` declares the errors a conformant validator must produce.

```json
[
  {
    "code": "DUPLICATE_ID",
    "path": "/plan/phases/0/weeks/0/days/1",
    "severity": "error",
    "meta": {
      "duplicate_id": "day_1",
      "scope": "week:week_1"
    }
  }
]
```

### Matching rules

- **Order-independent**: validators may emit errors in any order; matching is on `(code, path)` tuples.
- **Subset on `meta`**: an expected error's `meta` keys must all be present with matching values, but the validator's `meta` may carry additional keys. This lets us add fields without breaking fixtures.
- **`message` is not part of the contract**: validators may localize or reword. Tests must not assert on `message`.
- **Severity must match exactly**.
- **Extra errors fail the test**: the validator must produce *exactly* the expected error set, no more.

## Catalog-dependent fixtures

If a sibling `<name>.catalog.json` exists, the test runner passes its contents as the `catalog` option to `validate()`. Without a catalog file, the runner calls `validate(plan)` with no catalog (and `UNRESOLVED_REF` checks are skipped).

Catalog file shape:

```json
{
  "exercises": ["push_up", "squat"],
  "meals": [],
  "meditations": []
}
```

Each list becomes a Set/MapSet of known IDs in the validator's catalog argument.

## Adding a Fixture

1. Add `<name>.json` (the input plan) to `valid/` or `invalid/`.
2. For invalid fixtures, add `<name>.expected.json` listing the errors.
3. If introducing a new error code, add an entry to [`error-codes.md`](error-codes.md).
4. Bump the schema repo version (minor for new rules, patch for fixture clarifications).
