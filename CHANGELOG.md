# Changelog

All notable changes to the WPL specification, schema, and conformance suite.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added (conformance only — no schema change)
- Pass-2 semantic rule `ACTIVITY_BLOCK_MISMATCH` documented in `conformance/error-codes.md`: rejects activities whose `type` is not permitted in the parent block's `type`. See error-codes.md for the full allowed-activity table.
- `SPECIFICATION.md` notes that `Block.type` constrains allowed `Activity.type` values (references error-codes.md).
- New conformance fixtures: `valid/exercise-in-main-block.json`, `invalid/activity-block-mismatch-exercise-in-cooldown.json`, `invalid/activity-block-mismatch-nutrition-in-warmup.json`, `invalid/activity-block-mismatch-exercise-in-nutrition-block.json`.
- Updated `valid/simple-workout.json`: warmup activity changed from `exercise` (arm circles) to `recovery` (mobility) to comply with the new constraint.

## [1.6.0] — 2026-05-04

### Added
- **Contraindication tightening.** Optional `severity` (`low | moderate | high`, ACSM-style risk tiering) and a new `action: "require_clearance"` for plans that should gate execution behind documented medical clearance.
- **Cardio interval consistency.** `CardioPrescription.intervals.work.duration` and `.rest.duration` now accept a full `Duration` object alongside the existing bare number (seconds, retained for back-compat).
- **Cardio intensity slots.** `intensity.target` documents and pre-defines named slots: `zone`, `min_bpm`/`max_bpm`, `min_watts`/`max_watts`, and `value`+`unit` for pace (`min_per_km | min_per_mi | m_per_s | sec_per_100m`). Stays open (`additionalProperties: true`).
- **Resistance prescription extras.** `Reps.amrap: bool`, `ExercisePrescription.to_failure: bool`, and `Weight.metric` enum (`1RM | e1RM | training_max | daily_max`) for percentage-of-1RM prescriptions.
- **Typed progress measurements.** `Checkpoint.measurements[]` items now accept either a free string (back-compat) or a `MeasurementSpec` referencing the new `MeasurementMetric` enum (24 standardized metrics: body composition, hemodynamic, cardiorespiratory, strength, flexibility, sleep, RPE) and a `Questionnaire` enum (PHQ-9, GAD-7, IPAQ, PSQI, PSS-10, Borg CR-10, session RPE).
- **Recovery typing.** `RecoveryExercise` gains optional `modality` (static/dynamic/PNF/SMR/breathwork/mobility), `intensity_rpe`, a structured `pnf` block, and `body_part`.
- New conformance fixtures: `valid/contraindication-clearance.json`, `valid/cardio-intervals-duration.json`, `valid/amrap-to-failure.json`, `valid/checkpoint-typed-measurements.json`, `valid/recovery-pnf-smr.json`, `invalid/contraindication-bad-severity.json`, `invalid/contraindication-bad-action.json`, `invalid/checkpoint-bad-metric.json`, `invalid/recovery-bad-modality.json`, `invalid/weight-bad-metric.json`.

### Notes
All changes are additive; every plan that validated under 1.5.0 continues to validate under 1.6.0. Bare-number `intervals.work.duration` is now documented as deprecated but still accepted.

## [1.5.0] — 2026-05-03

### Added
- New activity variant `SubPlanActivity` (`type: "sub_plan"`, `sub_plan_ref: <plan-id>`). Lets a workout reuse a "warmup plan" or compose larger sessions from smaller plans.
- Pass-2 rule `CYCLIC_SUBPLAN` is now active (was previously deferred). Single-plan validation catches self-references (`A → A`). Cross-plan cycles (`A → B → A`) are documented as requiring a future `sub_plans` resolution map at validate time.
- `error-codes.md` updates `CYCLIC_SUBPLAN` from "deferred" to active, with a detection-scope table.
- New conformance fixtures: `valid/sub-plan-composition.json`, `invalid/cyclic-subplan-self.json`.

### Notes
The schema addition is additive (new branch in `Activity` `oneOf`). Validator-side: both reference validators (TS + Elixir) gain a single new rule. The cross-plan resolution API is a future minor release.

## [1.4.0] — 2026-05-03

### Added
- **Per-bodyweight scaling for macros and load.** `MacroRange.unit` accepts `g_per_kg` alongside `g`. `Calories.unit` accepts `kcal`, `kcal_per_kg`, or `multiplier_of_tdee` (optional; defaults to `kcal`). `Weight.type` accepts `percentage_bodyweight` alongside `absolute` and `percentage_1rm`. Lets plans express e.g. "1.6–2.2 g/kg/day protein" (Morton 2018) without baking in body weight.
- **Wellness telemetry source vocabulary.** `PersonalizationInput.source` documents recognized prefixes: `user.*`, `wellness.*` (HRV, sleep, session RPE, readiness, menstrual phase), `device.*`, `plan.*`. Closes the loop so adaptive plans can react to athlete state.
- **Clinical-condition codes for contraindications.** `Contraindication.condition` documents recognized prefixes: `icd10:`, `snomed:`, `acsm:`, `acog:`. Enables deterministic matching for clinical use cases (cardiac rehab, pregnancy, ICD-coded comorbidities) without forcing the prefix.
- New conformance fixtures: `valid/per-bodyweight-scaling.json`, `valid/wellness-telemetry-driven.json`, `valid/clinical-contraindications.json`.

### Notes
All changes additive. Existing fixtures and plans validate unchanged. The vocabulary additions for `source` and `condition` are documentation/`examples`-only — the field types remain open strings, so unrecognized values are accepted.

## [1.3.0] — 2026-05-03

### Added
- `ExerciseActivity` gains optional `primary_muscles[]`, `secondary_muscles[]`, `movement_pattern` with controlled vocabularies (`MuscleGroup` and `MovementPattern` enums). Lets analytics tools compute weekly sets per muscle group (Schoenfeld-style volume tracking) without an out-of-band exercise→muscle map.
- `CardioPrescription.intensity` gains optional `zone_model` enum (`hr_3_zone_seiler`, `hr_5_zone`, `hr_7_zone`, `power_coggan_7_zone`, `pace_critical_speed`, `rpe_borg_10`, `rpe_borg_20`) so a "Zone 2" target is unambiguous. `intensity.type` enum widened with `power` and `bpm`.
- Plan-level `athlete_thresholds` (HR max/LTHR/resting, FTP, VO2max, critical pace, body weight, 1RM list) — lets consumers convert relative-intensity targets into absolute numbers downstream.
- New conformance fixtures: `valid/muscle-tagged-workout.json`, `valid/cardio-zone-model.json`, `invalid/schema-violation-bad-muscle-group.json`.
- `error-codes.md` codifies the `oneOf` path-normalization rule: validators must drill into the best-matching branch when reporting failures inside a `oneOf`-typed schema (matches ajv's native behavior; ex_json_schema requires post-processing).

### Notes
All changes are additive. Every v1.x plan still validates unchanged.

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

## [1.0.0] — 2025-11-24

### Added
- Initial public release of the WPL specification.
- JSON Schema (`schema/v1.schema.json`) covering all required fields, enums, and structural rules.
- Three reference example plans (simple workout, HIIT circuit, holistic plan).
- Split licensing: CC-BY-4.0 for spec/examples, Apache-2.0 for schema/tooling.
- Trademark clause in README.
