# Changelog

All notable changes to the WPL specification, schema, and conformance suite.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] — 2026-05-03

### Added
- `Phase.type` enum (`accumulation | intensification | realization | deload | base | build | peak | recovery | transition`) — optional, lets consuming tools surface periodization role.
- `Week.is_deload` boolean — independent of `Phase.type` so individual deload weeks can be tagged inside any phase.
- `Tempo` definition supporting both the conventional 4-digit string ("3-1-1-0", "30X1") and a structured object (`eccentric`, `pause_bottom`, `concentric`, `pause_top`, `explosive_concentric`). `ExercisePrescription.tempo` is now `oneOf: [string, StructuredTempo]` instead of plain string — backwards compatible.
- Two new conformance fixtures: `valid/periodized-with-deload.json`, `valid/structured-tempo.json`.

### Notes
All changes are additive. No existing v1.x plan becomes invalid.

## [1.1.1] — 2026-05-02

### Changed
- Loosened `Action.type` and `Action.scope` schema enums (dropped strict enums; both now accept any string). Pass-2 semantic validation in the reference validators owns these checks, surfacing a descriptive `INVALID_PERSONALIZATION_RULE` error instead of a generic schema violation.

### Fixed
- Conformance fixtures in `conformance/invalid/` updated to satisfy schema requirements (added `Week.name`/`Week.order`, `Day.day_of_week`, `PointsConfig.enabled`, etc.), so Pass-2-only fixtures actually exercise their target rule rather than failing at Pass 1.
- `invalid-personalization-action.expected.json` updated to expect `INVALID_PERSONALIZATION_RULE` (now reachable after Action enum loosening).

## [1.1.0] — 2026-05-02

### Added
- Conformance suite (`conformance/`): valid + invalid fixtures, `error-codes.md`, `README.md` explaining matching rules.
- `.catalog.json` fixture convention for catalog-dependent error rules.
- 9 fixtures covering: `DUPLICATE_ID` (day, week), `EMPTY_PHASES_FOR_TYPE`, `INVALID_PRESCRIPTION` (sets_reps, time), `INVALID_PERSONALIZATION_RULE`, `INVALID_POINTS_RULE`, `PHASE_DURATION_MISMATCH` (warning), `UNRESOLVED_REF`, `SCHEMA_VIOLATION`.

### Changed
- Schema reconciled with `WPL_SPECIFICATION.md` and the production gymbile validator: `Plan` no longer requires `requirements`/`personalization`; `description` allowed; permissive id pattern (`^[a-z0-9][a-z0-9_-]*$`) replaces strict per-type patterns; metadata timestamps optional; `ExercisePrescription.type` required with full enum (`sets_reps|time|distance|amrap|continuous|intervals`); non-exercise activities (`cardio`, `nutrition`, `meditation`, `recovery`, `habit`) now carry `name` and the prescription/intensity/timing shapes documented in the spec; `progress.points` renamed to `points_system`; rule field `event` renamed to `action`; `Action.type` enum extended (`modify_exercise`, `use_schedule`); `Action.scope` enum added.

### Deferred
- `CYCLIC_SUBPLAN` conformance fixture — blocked on sub-plan reference shape being formalized in the spec ([#1](https://github.com/gymbile/wpl/issues/1)).

## [1.0.0] — 2026-05-02

### Added
- Initial public release of the WPL specification.
- JSON Schema (`schema/v1.schema.json`) covering all required fields, enums, and structural rules.
- Three reference example plans (simple workout, HIIT circuit, holistic plan).
- Split licensing: CC-BY-4.0 for spec/examples, Apache-2.0 for schema/tooling.
- Trademark clause in README.
