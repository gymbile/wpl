# Conformance Corpus Batch 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add ~70 conformance fixtures across 10 categories (meditation, habit, sub_plan, athlete_thresholds, personalization, progress, contraindications, requires, goals, regression) so both the TS and Elixir compile-conformance runners pass with the expanded corpus.

**Architecture:** Each fixture is a directory `conformance/compile/fixtures/<category>/<name>/` containing `source.wpl`, `expected.json`, and `meta.json`. Fixtures are authored in the WPL-AI DSL, compiled by both the TS helper (`node scripts/compile.mjs`) and the Elixir helper (`mix run scripts/compile.exs`), then the outputs are normalized (sort keys, replace auto-IDs, strip timestamps, coerce floats) and compared. When both outputs agree, the normalized JSON becomes `expected.json`. Both runners auto-discover fixtures, so no runner changes are required.

**Tech Stack:** WPL-AI DSL, TypeScript compiler (`wpl-ai`), Elixir compiler (`wpl-ai-ex`), Node.js scripts, Elixir Mix scripts, JSON/jq.

---

## Key Reference: DSL Syntax Cheat-Sheet

### Compile helpers
```bash
# TS (pretty JSON):
node /Users/alex/Projects/my/gymbile.com/wpl-ai/scripts/compile.mjs <path/to/source.wpl>

# Elixir (compact JSON, pipe through jq for pretty):
cd /Users/alex/Projects/my/gymbile.com/wpl-ai-ex && mix run scripts/compile.exs <path/to/source.wpl> | jq .
```

### Normalization applied before comparing (done by runners, also applied manually when writing expected.json):
1. Sort all JSON keys alphabetically at every nesting level
2. Replace auto-generated IDs (`^[a-z0-9_]+_\d+$` or `_block` suffix) with `"<AUTO_ID>"`
3. Replace UUID-format IDs with `"<UUID>"`
4. Remove `metadata.created_at` and `metadata.updated_at`
5. Coerce whole-number floats to integers (1.0 → 1)

### DSL syntax for each new category (verified from parser.ts):

**Meditation block:**
```
meditation:
  meditation <category>:
    duration N minutes
    guided true|false
    audio <audio_id>
```
Categories: `mindfulness`, `breathing`, `visualization`, `body_scan`, `sleep`

**Habit block (inside any block type, e.g. education:):**
```
education:
  habit <category>:
    target N <unit>
    frequency daily|weekly
    reminders HH:MM, HH:MM
```
Categories: `hydration`, `sleep`, `steps`, `screen_time`, `custom`, `water_intake`, `mindful_moments`, `step_count`

**Sub-plan activity (inside any block):**
```
warmup:
  subplan <ref_id>
  subplan <ref_id> "Optional Name"
```

**ATHLETE_THRESHOLDS section:**
```
ATHLETE_THRESHOLDS
  hr_max 185 bpm
  lthr 165 bpm
  resting_hr 52 bpm
  ftp 280 watts
  vo2max 55.2
  critical_pace_seconds_per_km 240
  body_weight 75 kg
  one_rm squat 140 kg
  one_rm bench_press 100 kg
```

**PERSONALIZATION section:**
```
PERSONALIZATION
  INPUTS
    client_age = user as number label "Age"
    fitness_level = assessment as enum options(beginner, intermediate, advanced)
    client_injuries = wellness as array label "Current Injuries"
    equipment_available = device as array

  RULES
    WHEN client_age >= 50:
      modify intensity factor 0.85
    WHEN fitness_level = beginner:
      reduce intensity by 20%
    WHEN client_injuries contains knee:
      replace squat -> wall_sit
      exclude jump_squat
    WHEN client_injuries contains knee OR client_injuries contains ankle:
      reduce sets by 1
    WHEN fitness_level = advanced AND client_age < 40:
      modify intensity factor 1.1
```

**REQUIRES section:**
```
REQUIRES
  age 16..65
  fitness beginner, intermediate
  equipment:
    dumbbells (required)
    yoga_mat (optional, alternatives: resistance_bands)
  contraindication pregnancy action exclude
  contraindication back_injury severity moderate action modify
  contraindication cardiac_event severity high action require_clearance
  time:
    days_per_week 3..5
    minutes_per_day 30..60
```

**GOALS section:**
```
GOALS
  GOAL primary weight_loss:
    name "Lose 5kg"
    target weight -5 kg relative
    deadline 2026-08-01
    milestone "First kg":
      at -1 kg
      reward 100 points
      badge first_kg_badge
  GOAL secondary muscle_gain:
    name "Build Strength"
```

**PROGRESS section:**
```
PROGRESS
  checkpoint "Week 2 Check-in":
    at 2 weeks
    measure:
      body_weight_kg
      questionnaire_score questionnaire phq9 note "mood tracking"
      "photos"
    ask:
      - "How is your energy level?"
      - "Any pain or discomfort?"
  points enabled:
    rules:
      - complete_workout 10
      - complete_day 25
      - complete_week 100
  streaks enabled
```

---

## Fixture Directory Map

```
conformance/compile/fixtures/
  meditation/
    guided-true/
    guided-false/
    meditation-with-duration-minutes/
    meditation-mindfulness-category/
    meditation-breathing-category/
    meditation-with-audio-id/
    meditation-multiple-in-one-block/
  habit/
    habit-water-intake/
    habit-with-frequency-daily/
    habit-with-frequency-weekly/
    habit-with-target-value-and-unit/
    habit-with-reminder-times/
    habit-with-multiple-reminders/
    habit-sleep-hours/
    habit-step-count/
    habit-mindful-moments/
  sub_plan/
    sub-plan-warmup-reference/
    sub-plan-with-explicit-name/
    sub-plan-multiple-in-one-day/
    sub-plan-in-cooldown-block/
  athlete_thresholds/
    thresholds-hr-max-only/
    thresholds-full-set/
    thresholds-with-one-rm/
    thresholds-with-multiple-one-rm/
    thresholds-with-units-kg/
    thresholds-with-units-lb/
    thresholds-body-weight/
  personalization/
    inputs-only/
    rules-with-when-replace/
    rules-with-when-exclude/
    rules-with-when-modify/
    rules-with-multiple-conditions-and/
    rules-with-multiple-conditions-or/
    rules-with-personalization-source-user/
    rules-with-personalization-source-wellness/
    rules-with-personalization-source-device/
    rules-with-personalization-source-plan/
  progress/
    checkpoint-with-typed-measurements/
    checkpoint-with-questionnaire-score/
    checkpoint-with-mixed-string-and-typed-items/
    checkpoint-with-questions-list/
    checkpoint-with-at-n-weeks/
    checkpoint-with-at-n-days/
    progress-points-system-enabled/
    progress-points-system-with-rules/
    progress-streaks/
    multiple-checkpoints-in-one-plan/
  contraindications/
    contraindication-simple/
    contraindication-with-severity-low/
    contraindication-with-severity-high-action-require-clearance/
    contraindication-action-modify/
    contraindication-with-icd10-prefix/
    contraindication-with-acsm-prefix/
  requires/
    requires-age-range/
    requires-fitness-level/
    requires-equipment-required/
    requires-equipment-with-alternatives/
    requires-time-commitment/
  goals/
    single-primary-goal/
    primary-and-secondary-goals/
    goal-with-target-value-and-unit/
    goal-with-deadline/
    goal-with-milestones/
  regression/
    legacy-1-0-style-plan/
    multi-week-multi-day-plan/
    nested-blocks-with-rest-between-rounds/
    empty-personalization-rules-allowed/
```

---

## Task 1: `meditation/` — 7 fixtures

**Files:**
- Create: `conformance/compile/fixtures/meditation/guided-true/source.wpl`
- Create: `conformance/compile/fixtures/meditation/guided-true/expected.json`
- Create: `conformance/compile/fixtures/meditation/guided-true/meta.json`
- (repeat for each of 7 meditation fixtures)

- [ ] **Step 1: Create `meditation/guided-true` fixture**

Create `/Users/alex/Projects/my/gymbile.com/wpl/conformance/compile/fixtures/meditation/guided-true/source.wpl`:
```
PLAN "Guided Meditation"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Meditation Day":
        meditation:
          meditation mindfulness:
            duration 10 minutes
            guided true
```

- [ ] **Step 2: Compile and verify both outputs match**

```bash
node /Users/alex/Projects/my/gymbile.com/wpl-ai/scripts/compile.mjs \
  /Users/alex/Projects/my/gymbile.com/wpl/conformance/compile/fixtures/meditation/guided-true/source.wpl

cd /Users/alex/Projects/my/gymbile.com/wpl-ai-ex && mix run scripts/compile.exs \
  /Users/alex/Projects/my/gymbile.com/wpl/conformance/compile/fixtures/meditation/guided-true/source.wpl | jq .
```

Compare outputs (after normalization: sort keys, replace auto-IDs, strip timestamps, coerce floats). Expected normalized JSON shape:
```json
{
  "$schema": "https://wpl.dev/schemas/wpl/v1.schema.json",
  "plan": {
    "goals": [],
    "id": "<UUID>",
    "metadata": { "language": "en" },
    "name": "Guided Meditation",
    "personalization": { "inputs": [], "rules": [] },
    "phases": [{
      "duration": { "unit": "weeks", "value": 1 },
      "id": "<AUTO_ID>",
      "name": "Week One",
      "order": 1,
      "weeks": [{
        "days": [{
          "blocks": [{
            "activities": [{
              "category": "mindfulness",
              "id": "<AUTO_ID>",
              "name": "Mindfulness Meditation",
              "prescription": {
                "duration": { "unit": "minutes", "value": 10 },
                "guided": true
              },
              "type": "meditation"
            }],
            "id": "<AUTO_ID>",
            "order": 1,
            "type": "meditation"
          }],
          "day_of_week": 1,
          "estimated_duration_minutes": 0,
          "id": "<AUTO_ID>",
          "name": "Meditation Day",
          "type": "training"
        }],
        "id": "<AUTO_ID>",
        "name": "Week 1",
        "order": 1
      }]
    }],
    "requirements": {},
    "type": "workout",
    "visibility": "public"
  },
  "version": "1.6.0"
}
```

- [ ] **Step 3: Save `expected.json` and `meta.json`**

Save the normalized agreed-upon output as `expected.json`.

Create `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation activity with guided: true flag. Tests that guided boolean is emitted in prescription.",
  "category": "meditation"
}
```

- [ ] **Step 4: Create `meditation/guided-false` fixture**

Create `source.wpl`:
```
PLAN "Unguided Meditation"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Quiet Practice":
        meditation:
          meditation breathing:
            duration 5 minutes
            guided false
```

Compile with both, verify match, save `expected.json` (guided: false) and `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation activity with guided: false. Tests that false boolean is emitted (not omitted) in prescription.",
  "category": "meditation"
}
```

- [ ] **Step 5: Create `meditation/meditation-with-duration-minutes` fixture**

Create `source.wpl`:
```
PLAN "Duration Meditation"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Wednesday training "Evening Wind Down":
        meditation:
          meditation body_scan:
            duration 20 minutes
```

No `guided` field → prescription should only contain `duration`. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation activity with only duration specified; no guided flag. Tests that prescription contains only duration.",
  "category": "meditation"
}
```

- [ ] **Step 6: Create `meditation/meditation-mindfulness-category` fixture**

Create `source.wpl`:
```
PLAN "Mindfulness Plan"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Tuesday training "Mindfulness Session":
        meditation:
          meditation mindfulness:
            duration 15 minutes
            guided true
```

Expected: `"category": "mindfulness"`, `"name": "Mindfulness Meditation"`. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation with mindfulness category. Verifies category field and auto-derived name 'Mindfulness Meditation'.",
  "category": "meditation"
}
```

- [ ] **Step 7: Create `meditation/meditation-breathing-category` fixture**

Create `source.wpl`:
```
PLAN "Breathing Plan"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Friday training "Breathing Session":
        meditation:
          meditation breathing:
            duration 8 minutes
            guided true
```

Expected: `"category": "breathing"`, `"name": "Breathing Meditation"`. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation with breathing category. Verifies category field and auto-derived name 'Breathing Meditation'.",
  "category": "meditation"
}
```

- [ ] **Step 8: Create `meditation/meditation-with-audio-id` fixture**

Create `source.wpl`:
```
PLAN "Audio Meditation"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Thursday training "Guided Audio Session":
        meditation:
          meditation visualization:
            duration 12 minutes
            guided true
            audio relaxing_ocean_001
```

Expected: prescription includes `"audio_id": "relaxing_ocean_001"`. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Meditation activity with explicit audio_id. Tests audio field in prescription.",
  "category": "meditation"
}
```

- [ ] **Step 9: Create `meditation/meditation-multiple-in-one-block` fixture**

Create `source.wpl`:
```
PLAN "Multiple Meditations"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Morning Practice":
        meditation:
          meditation breathing:
            duration 5 minutes
            guided false
          meditation mindfulness:
            duration 10 minutes
            guided true
```

Expected: two meditation activities in the block. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Two meditation activities inside a single meditation block. Tests multiple activities parsing.",
  "category": "meditation"
}
```

- [ ] **Step 10: Commit the meditation fixtures**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl
git add conformance/compile/fixtures/meditation/
git status
```

Do NOT commit yet — wait for the full batch commit in the final task.

---

## Task 2: `habit/` — 9 fixtures

**Files:**
- Create: `conformance/compile/fixtures/habit/<name>/source.wpl`, `expected.json`, `meta.json`

- [ ] **Step 1: Create `habit/habit-water-intake` fixture**

Create `source.wpl`:
```
PLAN "Water Intake Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Hydration Day":
        education:
          habit water_intake:
            target 8 glasses
            frequency daily
```

Expected: `"category": "water_intake"`, `"name": "Water Intake"`, `prescription.target = {value: 8, unit: "glasses"}`, `prescription.frequency = "daily"`.

Compile both, verify match, save `expected.json` and `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit activity for water intake with daily frequency. Tests target value/unit and frequency fields.",
  "category": "habit"
}
```

- [ ] **Step 2: Create `habit/habit-with-frequency-daily` fixture**

Create `source.wpl`:
```
PLAN "Daily Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Tuesday training "Daily Check":
        education:
          habit hydration:
            target 2 liters
            frequency daily
```

Save `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit with frequency: daily. Verifies frequency field emission.",
  "category": "habit"
}
```

- [ ] **Step 3: Create `habit/habit-with-frequency-weekly` fixture**

Create `source.wpl`:
```
PLAN "Weekly Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Wednesday training "Weekly Check":
        education:
          habit custom:
            target 3 sessions
            frequency weekly
```

Save `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit with frequency: weekly. Verifies weekly frequency is emitted correctly.",
  "category": "habit"
}
```

- [ ] **Step 4: Create `habit/habit-with-target-value-and-unit` fixture**

Create `source.wpl`:
```
PLAN "Target Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Thursday training "Target Day":
        education:
          habit steps:
            target 10000 steps
```

Save `meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit with target value and unit (10000 steps). Tests numeric target and unit string.",
  "category": "habit"
}
```

- [ ] **Step 5: Create `habit/habit-with-reminder-times` fixture**

Create `source.wpl`:
```
PLAN "Reminder Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Friday training "Reminder Day":
        education:
          habit hydration:
            target 8 glasses
            frequency daily
            reminders 09:00, 14:00
```

Expected: `prescription.reminder_times = ["09:00", "14:00"]`. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit with two reminder times. Tests reminder_times array emission.",
  "category": "habit"
}
```

- [ ] **Step 6: Create `habit/habit-with-multiple-reminders` fixture**

Create `source.wpl`:
```
PLAN "Multi Reminder Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Saturday training "Reminder Day":
        education:
          habit hydration:
            target 10 glasses
            frequency daily
            reminders 07:00, 10:00, 13:00, 16:00, 19:00
```

Expected: 5 reminder times. Compile both, verify, save.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit with five reminder times. Tests that reminder_times accepts multiple HH:MM values.",
  "category": "habit"
}
```

- [ ] **Step 7: Create `habit/habit-sleep-hours` fixture**

Create `source.wpl`:
```
PLAN "Sleep Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Sunday active_recovery "Sleep Day":
        education:
          habit sleep:
            target 8 hours
            frequency daily
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit for sleep tracking with hours as unit. Tests sleep category and hours unit.",
  "category": "habit"
}
```

- [ ] **Step 8: Create `habit/habit-step-count` fixture**

Create `source.wpl`:
```
PLAN "Step Count Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Steps Day":
        education:
          habit steps:
            target 8000 steps
            frequency daily
            reminders 12:00
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit for step count with single reminder. Tests steps category with target and reminder.",
  "category": "habit"
}
```

- [ ] **Step 9: Create `habit/habit-mindful-moments` fixture**

Create `source.wpl`:
```
PLAN "Mindful Moments Habit"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Tuesday training "Mindful Day":
        education:
          habit custom:
            target 3 pauses
            frequency daily
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Habit for mindful moments using custom category. Tests custom category token.",
  "category": "habit"
}
```

- [ ] **Step 10: Run both runners to verify no regressions (~25 fixtures added so far)**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl-ai && npm test 2>&1 | tail -10
cd /Users/alex/Projects/my/gymbile.com/wpl-ai-ex && mix test test/conformance_test.exs 2>&1 | tail -5
```

Expected: all tests pass (80 batch-1 + 7 meditation + 9 habit = 96 tests).

---

## Task 3: `sub_plan/` — 4 fixtures

**Files:**
- Create: `conformance/compile/fixtures/sub_plan/<name>/source.wpl`, `expected.json`, `meta.json`

Note: The DSL keyword is `subplan` (no underscore). The compiled JSON field is `sub_plan_ref`. The activity `type` is `"sub_plan"`.

- [ ] **Step 1: Create `sub_plan/sub-plan-warmup-reference` fixture**

Create `source.wpl`:
```
PLAN "Sub Plan Warmup"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Warmup Day":
        warmup:
          subplan plan_warmup_standard
        main straight_sets:
          push_up 3x10
```

Expected: warmup block has one activity with `type: "sub_plan"`, `sub_plan_ref: "plan_warmup_standard"`, `name: null` (no name specified). The ID uses the auto-id pattern `sub_plan_N`.

Compile both, verify match, save `expected.json`:
The sub_plan activity should look like:
```json
{
  "id": "<AUTO_ID>",
  "name": null,
  "sub_plan_ref": "plan_warmup_standard",
  "type": "sub_plan"
}
```
(Note: check if `name: null` is emitted or omitted — use compact() which drops nulls. Verify both compilers agree.)

`meta.json`:
```json
{
  "since": "1.5.0",
  "description": "Sub-plan reference in warmup block without an explicit name. Tests sub_plan activity type and sub_plan_ref field.",
  "category": "sub_plan"
}
```

- [ ] **Step 2: Create `sub_plan/sub-plan-with-explicit-name` fixture**

Create `source.wpl`:
```
PLAN "Named Sub Plan"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training "Named Day":
        warmup:
          subplan plan_warmup_full_body "Full Body Warmup"
        main straight_sets:
          squat 3x8
```

Expected: sub_plan activity has `name: "Full Body Warmup"`.

`meta.json`:
```json
{
  "since": "1.5.0",
  "description": "Sub-plan reference with explicit name string. Tests that name field is emitted when provided in DSL.",
  "category": "sub_plan"
}
```

- [ ] **Step 3: Create `sub_plan/sub-plan-in-cooldown-block` fixture**

Create `source.wpl`:
```
PLAN "Cooldown Sub Plan"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Tuesday training "Cooldown Day":
        main straight_sets:
          push_up 3x10
        cooldown:
          subplan plan_cooldown_mobility
```

`meta.json`:
```json
{
  "since": "1.5.0",
  "description": "Sub-plan reference in cooldown block. Tests sub_plan type in cooldown context.",
  "category": "sub_plan"
}
```

- [ ] **Step 4: Create `sub_plan/sub-plan-multiple-in-one-day` fixture**

Create `source.wpl`:
```
PLAN "Multi Sub Plan Day"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Wednesday training "Template Day":
        warmup:
          subplan plan_warmup_upper "Upper Warmup"
        main straight_sets:
          bench_press 4x6
        cooldown:
          subplan plan_cooldown_chest "Chest Cooldown"
```

Expected: warmup has one sub_plan, cooldown has one sub_plan.

`meta.json`:
```json
{
  "since": "1.5.0",
  "description": "Multiple sub-plan references in one day (warmup and cooldown). Tests that sub_plan activities work in multiple blocks.",
  "category": "sub_plan"
}
```

---

## Task 4: `athlete_thresholds/` — 7 fixtures

**Files:**
- Create: `conformance/compile/fixtures/athlete_thresholds/<name>/source.wpl`, `expected.json`, `meta.json`

Note: `ATHLETE_THRESHOLDS` is a top-level DSL section (peer to GOALS, PERSONALIZATION, etc.). It appears in the compiled JSON as `plan.athlete_thresholds`. Fields: `hr_max_bpm`, `lthr_bpm`, `resting_hr_bpm`, `ftp_watts`, `vo2max_ml_kg_min`, `critical_pace_seconds_per_km`, `body_weight_kg`, `one_rm` (array of `{exercise_ref, value, unit}`).

- [ ] **Step 1: Create `athlete_thresholds/thresholds-hr-max-only` fixture**

Create `source.wpl`:
```
PLAN "HR Max Only"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  hr_max 185 bpm

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `plan.athlete_thresholds = { "hr_max_bpm": 185 }` (only hr_max_bpm key, rest omitted by compact()).

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with only hr_max. Tests that only non-null fields are emitted.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 2: Create `athlete_thresholds/thresholds-full-set` fixture**

Create `source.wpl`:
```
PLAN "Full Thresholds"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  hr_max 190 bpm
  lthr 170 bpm
  resting_hr 48 bpm
  ftp 300 watts
  vo2max 58.5
  body_weight 80 kg

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: all 6 scalar threshold fields present in `plan.athlete_thresholds`.

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with all 6 scalar fields (hr_max_bpm, lthr_bpm, resting_hr_bpm, ftp_watts, vo2max_ml_kg_min, body_weight_kg). Tests full threshold set compilation.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 3: Create `athlete_thresholds/thresholds-with-one-rm` fixture**

Create `source.wpl`:
```
PLAN "One RM Thresholds"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  hr_max 182 bpm
  one_rm squat 120 kg

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `one_rm: [{ exercise_ref: "squat", value: 120, unit: "kg" }]`.

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with one_rm entry for squat. Tests one_rm array compilation.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 4: Create `athlete_thresholds/thresholds-with-multiple-one-rm` fixture**

Create `source.wpl`:
```
PLAN "Multiple One RMs"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  one_rm squat 140 kg
  one_rm bench_press 100 kg
  one_rm deadlift 180 kg

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `one_rm` array with 3 entries.

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with three one_rm entries. Tests multiple one_rm entries in array.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 5: Create `athlete_thresholds/thresholds-with-units-kg` fixture**

Create `source.wpl`:
```
PLAN "KG Units"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  body_weight 75 kg
  one_rm overhead_press 60 kg

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with kg units explicitly stated. Tests kg unit parsing.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 6: Create `athlete_thresholds/thresholds-with-units-lb` fixture**

Create `source.wpl`:
```
PLAN "LB Units"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  one_rm squat 315 lb
  one_rm bench_press 225 lb

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `unit: "lb"` for both one_rm entries.

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with lb units. Tests lb unit parsing for one_rm.",
  "category": "athlete_thresholds"
}
```

- [ ] **Step 7: Create `athlete_thresholds/thresholds-body-weight` fixture**

Create `source.wpl`:
```
PLAN "Body Weight Threshold"
TYPE workout
VISIBILITY public

GOALS

ATHLETE_THRESHOLDS
  body_weight 68 kg

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.3.0",
  "description": "ATHLETE_THRESHOLDS with only body_weight_kg. Tests body_weight field compilation.",
  "category": "athlete_thresholds"
}
```

---

## Task 5: `personalization/` — 10 fixtures

**Files:**
- Create: `conformance/compile/fixtures/personalization/<name>/source.wpl`, `expected.json`, `meta.json`

Note: Input DSL syntax: `<name> = <source> as <type> [label "text"] [options(a, b, c)]`
Sources: `user`, `wellness`, `device`, `plan`, `assessment`, `questionnaire`
Types: `number`, `string`, `array`, `enum`, `boolean`

Rule DSL syntax: `WHEN <condition>:\n  <actions>`
Actions: `replace A -> B [scope plan]`, `exclude <exercise> [scope plan]`, `modify intensity factor 0.8 [scope plan]`, `reduce intensity by 20% [scope plan]`, `reduce sets by 1 [scope plan]`, `reduce reps by 2 [scope plan]`, `add warmup 5 minutes [scope day]`, `add activity <name> [before|after|in <target>] [scope plan]`, `increase rest by 30 seconds [scope plan]`

- [ ] **Step 1: Create `personalization/inputs-only` fixture**

Create `source.wpl`:
```
PLAN "Inputs Only"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    client_age = user as number label "Age"
    fitness_level = assessment as enum options(beginner, intermediate, advanced)

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `personalization.inputs` has 2 entries, `personalization.rules = []`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION with only INPUTS and no RULES. Tests input parsing with source, type, label, and options.",
  "category": "personalization"
}
```

- [ ] **Step 2: Create `personalization/rules-with-when-replace` fixture**

Create `source.wpl`:
```
PLAN "Replace Rule"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  RULES
    WHEN injury contains knee:
      replace squat -> wall_sit
      replace lunge -> step_up

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: one rule with `condition: {field: "injury", op: "contains", value: "knee"}`, `actions` with two `replace_exercise` entries.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Personalization rule with WHEN ... contains and replace actions. Tests replace_exercise action compilation.",
  "category": "personalization"
}
```

- [ ] **Step 3: Create `personalization/rules-with-when-exclude` fixture**

Create `source.wpl`:
```
PLAN "Exclude Rule"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  RULES
    WHEN injury contains shoulder:
      exclude overhead_press
      exclude lateral_raise

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Personalization rule with WHEN ... contains and exclude actions. Tests exclude_exercise action compilation.",
  "category": "personalization"
}
```

- [ ] **Step 4: Create `personalization/rules-with-when-modify` fixture**

Create `source.wpl`:
```
PLAN "Modify Rule"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  RULES
    WHEN client_age >= 60:
      modify intensity factor 0.8
      add warmup 5 minutes

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Personalization rule with WHEN >= operator and modify intensity + add warmup actions.",
  "category": "personalization"
}
```

- [ ] **Step 5: Create `personalization/rules-with-multiple-conditions-and` fixture**

Create `source.wpl`:
```
PLAN "AND Conditions"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  RULES
    WHEN fitness_level = advanced AND client_age < 40:
      modify intensity factor 1.1

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected condition: compound type with `operator: "and"`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Personalization rule with compound AND condition. Tests compound condition compilation with and operator.",
  "category": "personalization"
}
```

- [ ] **Step 6: Create `personalization/rules-with-multiple-conditions-or` fixture**

Create `source.wpl`:
```
PLAN "OR Conditions"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  RULES
    WHEN injury contains knee OR injury contains ankle:
      reduce sets by 1

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected condition: compound type with `operator: "or"`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Personalization rule with compound OR condition. Tests compound condition compilation with or operator.",
  "category": "personalization"
}
```

- [ ] **Step 7: Create `personalization/rules-with-personalization-source-user` fixture**

Create `source.wpl`:
```
PLAN "User Source"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    client_age = user as number label "Client Age"
    client_weight = user as number label "Body Weight kg"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: both inputs have `source: "user"`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION INPUTS using user source. Tests that source field is emitted as 'user'.",
  "category": "personalization"
}
```

- [ ] **Step 8: Create `personalization/rules-with-personalization-source-wellness` fixture**

Create `source.wpl`:
```
PLAN "Wellness Source"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    sleep_quality = wellness as number label "Sleep Quality Score"
    stress_level = wellness as enum options(low, moderate, high) label "Stress Level"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION INPUTS using wellness source. Tests wellness source emission.",
  "category": "personalization"
}
```

- [ ] **Step 9: Create `personalization/rules-with-personalization-source-device` fixture**

Create `source.wpl`:
```
PLAN "Device Source"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    resting_hr = device as number label "Resting Heart Rate"
    hrv_score = device as number label "HRV Score"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION INPUTS using device source. Tests device source emission.",
  "category": "personalization"
}
```

- [ ] **Step 10: Create `personalization/rules-with-personalization-source-plan` fixture**

Create `source.wpl`:
```
PLAN "Plan Source"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    week_number = plan as number label "Current Week"
    phase_name = plan as string label "Current Phase"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION INPUTS using plan source. Tests plan source emission.",
  "category": "personalization"
}
```

- [ ] **Step 11: Run both runners mid-batch (after ~45 fixtures)**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl-ai && npm test 2>&1 | tail -10
cd /Users/alex/Projects/my/gymbile.com/wpl-ai-ex && mix test test/conformance_test.exs 2>&1 | tail -5
```

Expected: all tests pass.

---

## Task 6: `progress/` — 10 fixtures

**Files:**
- Create: `conformance/compile/fixtures/progress/<name>/source.wpl`, `expected.json`, `meta.json`

Note: PROGRESS is a top-level section. Checkpoints compile to: `{id: "<AUTO_ID>" (short hash), name, at: {value, unit}, measurements: [...], questions: [...]}`. Points compile to `{enabled, rules: [{event, points}]}`. Streaks compile to `{enabled, types}`. Note: checkpoint id is generated by `generateShortId("cp")` — this is NOT an auto-numbered id pattern, so it won't be replaced by `<AUTO_ID>`. It will be a random short string. **This means checkpoint IDs will diverge between runs and compilers.** Check whether the conformance runner's normalization handles this. If not, we need to investigate.

- [ ] **Step 0: Verify checkpoint ID normalization**

Before creating progress fixtures, compile a test plan with a checkpoint and check what ID format is generated:

```bash
# Create a temp source to test checkpoint ID format
cat > /tmp/test-checkpoint.wpl << 'EOF'
PLAN "Test Progress"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Week 2 Check-in":
    at 2 weeks
    measure:
      body_weight_kg
EOF

node /Users/alex/Projects/my/gymbile.com/wpl-ai/scripts/compile.mjs /tmp/test-checkpoint.wpl | jq '.plan.progress'
```

Check the `id` field in the checkpoint output. If it matches `^[a-z0-9_]+_\d+$` pattern → it's normalized by existing rules. If it's a random hash → it's NOT normalized and will diverge between runs. In that case, add the checkpoint `id` path to the normalization discussion or keep fixtures without checking the id field by using a wrapper around expected.json that strips checkpoint ids.

**If checkpoint IDs are random hashes:** The conformance runner currently only normalizes `id` fields matching the auto-id pattern. Random hash IDs will differ between compiler runs. **Resolution:** Use `generateShortId` output — trace what it generates. Check the compiler source — it uses `generateShortId("cp")`. This creates a short non-deterministic ID. The conformance runner does NOT normalize these. **Therefore:** Either (a) do not include checkpoint `id` in expected.json and the runner will fail, or (b) add a normalization rule for checkpoint ids. Option (b) requires runner changes (forbidden unless strictly necessary). Option (c): replace the checkpoint id in expected.json with `"<CP_ID>"` or similar, and add a normalization rule.

**Correct approach:** Check actual output. If IDs diverge between runs, these fixtures cannot be added without a runner change. Document as a known limitation and skip progress fixtures that include checkpoints, OR add a normalization rule for `^cp_[a-z0-9]+$`-pattern IDs.

Actually — re-read the normalization rules. The AUTO_ID_RE is `/^[a-z0-9_]+_\d+$|^[a-z0-9_]+_block$/`. A checkpoint id like `cp_a1b2c3` does NOT match this (has hex chars after `_`, not just digits). So checkpoint ids will NOT be normalized. **This means progress fixtures with checkpoints will fail conformance if checkpoint ids are random.**

**Plan:** Test what `generateShortId("cp")` actually produces. If it produces `cp_NNNN` with a hex/random suffix, we need to either skip progress/checkpoint fixtures OR add a normalization rule. Check the source:

```bash
grep -n "generateShortId\|shortId\|nanoid\|crypto" /Users/alex/Projects/my/gymbile.com/wpl-ai/src/compiler.ts | head -20
```

If the function uses crypto random (always different), the only way to make checkpoint fixtures work is to add a normalization rule for `^cp_[a-zA-Z0-9]+$` pattern IDs, or strip the `id` field from checkpoints entirely in expected.json.

**Decision:** Add a normalization rule for checkpoint IDs pattern `^cp_[a-z0-9]+$` to both runners. This is a **runner change** — document it. If this is too invasive, skip checkpoint fixtures and note as divergence.

After resolving this, proceed to create progress fixtures.

- [ ] **Step 1: Create `progress/checkpoint-with-at-n-weeks` fixture**

After resolving checkpoint ID normalization, create `source.wpl`:
```
PLAN "Checkpoint At Weeks"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Week 2 Check-in":
    at 2 weeks
```

Expected: `progress.checkpoints[0]` has `name: "Week 2 Check-in"`, `at: {value: 2, unit: "1"}` (note: the compiler maps `at N weeks` to `{value: N, unit: String(unit_count)}` where `unit_count` is 1). Verify exact output.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "PROGRESS checkpoint with `at N weeks` trigger syntax. Tests time-triggered checkpoint compilation.",
  "category": "progress"
}
```

- [ ] **Step 2: Create `progress/checkpoint-with-at-n-days` fixture**

Create `source.wpl`:
```
PLAN "Checkpoint At Days"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Day 14 Check-in":
    at 14 days
```

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "PROGRESS checkpoint with `at N days` trigger syntax. Tests day-based trigger.",
  "category": "progress"
}
```

- [ ] **Step 3: Create `progress/checkpoint-with-typed-measurements` fixture**

Create `source.wpl`:
```
PLAN "Typed Measurements"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Baseline":
    at 0 weeks
    measure:
      body_weight_kg
      resting_hr_bpm
      hrv_rmssd_ms
```

Expected: `measurements` array has 3 objects with `metric` field (typed MeasurementSpec).

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Checkpoint with typed MeasurementSpec items (metric field). Tests v1.6.0 typed measurement compilation.",
  "category": "progress"
}
```

- [ ] **Step 4: Create `progress/checkpoint-with-questionnaire-score` fixture**

Create `source.wpl`:
```
PLAN "Questionnaire Checkpoint"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Mental Health Check":
    at 4 weeks
    measure:
      questionnaire_score questionnaire phq9 note "depression screening"
      questionnaire_score questionnaire gad7
```

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Checkpoint with questionnaire_score measurements (PHQ-9 and GAD-7). Tests v1.6.0 questionnaire field in MeasurementSpec.",
  "category": "progress"
}
```

- [ ] **Step 5: Create `progress/checkpoint-with-mixed-string-and-typed-items` fixture**

Create `source.wpl`:
```
PLAN "Mixed Measurements"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Mixed Check":
    at 4 weeks
    measure:
      body_weight_kg
      "photos"
      hrv_rmssd_ms
```

Expected: measurements = `[{metric: "body_weight_kg"}, "photos", {metric: "hrv_rmssd_ms"}]`.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Checkpoint mixing typed MeasurementSpec and plain string items. Tests back-compat string items alongside typed specs.",
  "category": "progress"
}
```

- [ ] **Step 6: Create `progress/checkpoint-with-questions-list` fixture**

Create `source.wpl`:
```
PLAN "Questions Checkpoint"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  checkpoint "Weekly Survey":
    at 1 weeks
    ask:
      - "How is your energy level?"
      - "Any pain or discomfort?"
      - "How motivated do you feel?"
```

Expected: `questions: ["How is your energy level?", "Any pain or discomfort?", "How motivated do you feel?"]`.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Checkpoint with ask: questions list. Tests questions array compilation.",
  "category": "progress"
}
```

- [ ] **Step 7: Create `progress/progress-points-system-enabled` fixture**

Create `source.wpl`:
```
PLAN "Points Enabled"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  points enabled
```

Expected: `progress.points = {enabled: true}` (no rules key).

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PROGRESS with points system enabled and no rules. Tests points enabled flag without rules.",
  "category": "progress"
}
```

- [ ] **Step 8: Create `progress/progress-points-system-with-rules` fixture**

Create `source.wpl`:
```
PLAN "Points With Rules"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  points enabled:
    rules:
      - complete_workout 10
      - complete_day 25
      - complete_week 100
```

Expected: `progress.points = {enabled: true, rules: [{event: "complete_workout", points: 10}, ...]}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PROGRESS points system with rules. Tests rules array with event and points fields.",
  "category": "progress"
}
```

- [ ] **Step 9: Create `progress/progress-streaks` fixture**

Create `source.wpl`:
```
PLAN "Streaks"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (1 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10

PROGRESS
  streaks enabled
```

Compile both, check what `streaks enabled` compiles to. Expected structure: `{enabled: true, types: []}` or similar. Verify both outputs match.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PROGRESS with streaks enabled. Tests streaks block compilation.",
  "category": "progress"
}
```

- [ ] **Step 10: Create `progress/multiple-checkpoints-in-one-plan` fixture**

Create `source.wpl`:
```
PLAN "Multiple Checkpoints"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "P1" (2 weeks):
    WEEK 1:
      DAY Monday training:
        main straight_sets:
          push_up 3x10
    WEEK 2:

PROGRESS
  checkpoint "Week 1 Check":
    at 1 weeks
    measure:
      body_weight_kg
  checkpoint "Week 2 Check":
    at 2 weeks
    measure:
      body_weight_kg
      resting_hr_bpm
```

Expected: `progress.checkpoints` array with 2 entries.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "PROGRESS with two checkpoints. Tests multiple checkpoint compilation in one plan.",
  "category": "progress"
}
```

---

## Task 7: `contraindications/` — 6 fixtures

**Files:**
- Create: `conformance/compile/fixtures/contraindications/<name>/source.wpl`, `expected.json`, `meta.json`

Note: These go inside `REQUIRES` block. The compiler emits `plan.requirements.contraindications`.

- [ ] **Step 1: Create `contraindications/contraindication-simple` fixture**

Create `source.wpl`:
```
PLAN "Simple Contraindication"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication pregnancy action exclude

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `requirements.contraindications = [{condition: "pregnancy", action: "exclude"}]` (no severity).

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Simple contraindication with action only (no severity). Tests basic contraindication compilation.",
  "category": "contraindications"
}
```

- [ ] **Step 2: Create `contraindications/contraindication-with-severity-low` fixture**

Create `source.wpl`:
```
PLAN "Low Severity Contraindication"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication mild_arthritis severity low action exclude

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `{condition: "mild_arthritis", action: "exclude", severity: "low"}`.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Contraindication with severity: low (v1.6.0 field). Tests severity field emission.",
  "category": "contraindications"
}
```

- [ ] **Step 3: Create `contraindications/contraindication-with-severity-high-action-require-clearance` fixture**

Create `source.wpl`:
```
PLAN "High Severity Clearance"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication cardiac_event severity high action require_clearance

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `{condition: "cardiac_event", action: "require_clearance", severity: "high"}`.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Contraindication with severity: high and action: require_clearance (v1.6.0). Tests require_clearance action and high severity.",
  "category": "contraindications"
}
```

- [ ] **Step 4: Create `contraindications/contraindication-action-modify` fixture**

Create `source.wpl`:
```
PLAN "Modify Contraindication"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication back_injury severity moderate action modify

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `{condition: "back_injury", action: "modify", severity: "moderate"}`.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Contraindication with severity: moderate and action: modify. Tests modify action with severity.",
  "category": "contraindications"
}
```

- [ ] **Step 5: Create `contraindications/contraindication-with-icd10-prefix` fixture**

Create `source.wpl`:
```
PLAN "ICD10 Contraindication"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication icd10_M54_5 severity moderate action modify

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Note: ICD-10 codes like `M54.5` (low back pain) use underscore as separator since dots aren't valid bare words. Verify the parser accepts this.

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Contraindication using icd10_ prefix convention. Tests that ICD-10 style condition names compile correctly.",
  "category": "contraindications"
}
```

- [ ] **Step 6: Create `contraindications/contraindication-with-acsm-prefix` fixture**

Create `source.wpl`:
```
PLAN "ACSM Contraindication"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  contraindication acsm_cardiac_event_recent severity high action require_clearance

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

`meta.json`:
```json
{
  "since": "1.6.0",
  "description": "Contraindication using acsm_ prefix convention. Tests ACSM-style condition names.",
  "category": "contraindications"
}
```

---

## Task 8: `requires/` — 5 fixtures

**Files:**
- Create: `conformance/compile/fixtures/requires/<name>/source.wpl`, `expected.json`, `meta.json`

- [ ] **Step 1: Create `requires/requires-age-range` fixture**

Create `source.wpl`:
```
PLAN "Age Range"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  age 18..65

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `requirements = {min_age: 18, max_age: 65}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "REQUIRES with age range only. Tests min_age and max_age field emission.",
  "category": "requires"
}
```

- [ ] **Step 2: Create `requires/requires-fitness-level` fixture**

Create `source.wpl`:
```
PLAN "Fitness Level"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  fitness beginner, intermediate

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `requirements = {fitness_level: ["beginner", "intermediate"]}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "REQUIRES with fitness level list. Tests fitness_level array emission.",
  "category": "requires"
}
```

- [ ] **Step 3: Create `requires/requires-equipment-required` fixture**

Create `source.wpl`:
```
PLAN "Required Equipment"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  equipment:
    dumbbells (required)
    yoga_mat (optional)

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `requirements.equipment = [{id: "dumbbells", name: "dumbbells", required: true}, {id: "yoga_mat", name: "yoga_mat", required: false}]`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "REQUIRES with equipment list (required and optional). Tests equipment array with required flag.",
  "category": "requires"
}
```

- [ ] **Step 4: Create `requires/requires-equipment-with-alternatives` fixture**

Create `source.wpl`:
```
PLAN "Equipment Alternatives"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  equipment:
    barbell (required, alternatives: dumbbells, resistance_bands)
    bench (optional, alternatives: floor)

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: equipment entries have `alternatives` arrays.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "REQUIRES equipment with alternatives list. Tests alternatives field emission.",
  "category": "requires"
}
```

- [ ] **Step 5: Create `requires/requires-time-commitment` fixture**

Create `source.wpl`:
```
PLAN "Time Commitment"
TYPE workout
VISIBILITY public

GOALS

REQUIRES
  time:
    days_per_week 3..5
    minutes_per_day 30..60

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `requirements.time_commitment = {min_days_per_week: 3, max_days_per_week: 5, min_minutes_per_day: 30, max_minutes_per_day: 60}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "REQUIRES with time_commitment block. Tests time commitment fields compilation.",
  "category": "requires"
}
```

---

## Task 9: `goals/` — 5 fixtures

**Files:**
- Create: `conformance/compile/fixtures/goals/<name>/source.wpl`, `expected.json`, `meta.json`

Note: Goals compile to `plan.goals` array. Each goal: `{id: "goal_N", type: "primary"|"secondary", category, name?, description?, target?, deadline?, milestones?}`.

- [ ] **Step 1: Create `goals/single-primary-goal` fixture**

Create `source.wpl`:
```
PLAN "Single Goal"
TYPE workout
VISIBILITY public

GOALS
  GOAL primary weight_loss:
    name "Lose Weight"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `goals = [{id: "goal_1", type: "primary", category: "weight_loss", name: "Lose Weight"}]`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Single primary goal with name. Tests goal array with type, category, and name fields.",
  "category": "goals"
}
```

- [ ] **Step 2: Create `goals/primary-and-secondary-goals` fixture**

Create `source.wpl`:
```
PLAN "Two Goals"
TYPE workout
VISIBILITY public

GOALS
  GOAL primary muscle_gain:
    name "Build Muscle"
  GOAL secondary endurance:
    name "Improve Cardio"

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `goals` array with 2 entries.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Primary and secondary goals. Tests multiple goals and type field values.",
  "category": "goals"
}
```

- [ ] **Step 3: Create `goals/goal-with-target-value-and-unit` fixture**

Create `source.wpl`:
```
PLAN "Goal With Target"
TYPE workout
VISIBILITY public

GOALS
  GOAL primary weight_loss:
    name "Lose 5kg"
    target weight -5 kg relative

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: goal has `target: {metric: "weight", value: -5, unit: "kg", measurement_type: "relative"}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Goal with target (metric, value, unit, measurement_type). Tests target field compilation.",
  "category": "goals"
}
```

- [ ] **Step 4: Create `goals/goal-with-deadline` fixture**

Create `source.wpl`:
```
PLAN "Goal With Deadline"
TYPE workout
VISIBILITY public

GOALS
  GOAL primary endurance:
    name "Run 5K"
    deadline 2026-08-01

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: goal has `deadline: "2026-08-01"`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Goal with deadline date. Tests deadline field compilation.",
  "category": "goals"
}
```

- [ ] **Step 5: Create `goals/goal-with-milestones` fixture**

Create `source.wpl`:
```
PLAN "Goal With Milestones"
TYPE workout
VISIBILITY public

GOALS
  GOAL primary weight_loss:
    name "Lose 5kg"
    milestone "First kg":
      at -1 kg
      reward 100 points
      badge first_kg_badge

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: goal has `milestones: [{name: "First kg", at_value: -1, at_unit: "kg", reward_points: 100, badge: "first_kg_badge"}]`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Goal with a milestone (name, at, reward, badge). Tests milestone compilation.",
  "category": "goals"
}
```

---

## Task 10: `regression/` — 4 fixtures

**Files:**
- Create: `conformance/compile/fixtures/regression/<name>/source.wpl`, `expected.json`, `meta.json`

- [ ] **Step 1: Create `regression/legacy-1-0-style-plan` fixture**

Create `source.wpl` (a realistic 1.0-era plan with no 1.2+ features):
```
PLAN "Legacy Plan"
TYPE workout
VISIBILITY public
DIFFICULTY beginner

GOALS
  GOAL primary muscle_gain:
    name "Build Strength"

PHASES
  PHASE "Foundation" (4 weeks):
    WEEK 1:
      DAY Monday training 45m "Upper Body":
        warmup:
          jumping_jack 3m
          arm_circles 2m
        main straight_sets:
          push_up 3x10 rest 60 seconds
          dumbbell_row 3x10 rest 60 seconds
        cooldown:
          chest_stretch 30s x2 sides both
    WEEK 2:
    WEEK 3:
    WEEK 4:
```

This should compile with version `1.6.0` (current schema version) even though it uses only 1.0 features.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "A realistic 1.0-style plan (no v1.2+ features) that must still compile to version 1.6.0 JSON. Tests backward compatibility of the DSL parser and compiler.",
  "category": "regression"
}
```

- [ ] **Step 2: Create `regression/multi-week-multi-day-plan` fixture**

Create `source.wpl`:
```
PLAN "Multi Week Multi Day"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Block 1" (2 weeks):
    WEEK 1:
      DAY Monday training 45m "Push Day":
        main straight_sets:
          push_up 3x10
          dumbbell_press 3x8
      DAY Wednesday training 45m "Pull Day":
        main straight_sets:
          dumbbell_row 3x10
          pull_up 3x5
      DAY Friday training 30m "Legs":
        main straight_sets:
          squat 3x10
          lunge 3x10
    WEEK 2:
      DAY Monday training 45m "Push Day":
        main straight_sets:
          push_up 4x10
      DAY Wednesday training 45m "Pull Day":
        main straight_sets:
          dumbbell_row 4x10
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Multi-week plan with multiple training days per week. Tests full phase/week/day nesting with days in multiple weeks.",
  "category": "regression"
}
```

- [ ] **Step 3: Create `regression/nested-blocks-with-rest-between-rounds` fixture**

Create `source.wpl`:
```
PLAN "Circuit With Rest"
TYPE workout
VISIBILITY public

GOALS

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
      DAY Monday training 30m "Circuit":
        main circuit:
          rounds 3
          rest_between_rounds 90 seconds
          push_up 3x10
          squat 3x10
          mountain_climber 3x20
```

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "Circuit block with rounds and rest_between_rounds. Tests block-level rounds and rest_between_rounds fields.",
  "category": "regression"
}
```

- [ ] **Step 4: Create `regression/empty-personalization-rules-allowed` fixture**

Create `source.wpl`:
```
PLAN "Empty Personalization"
TYPE workout
VISIBILITY public

GOALS

PERSONALIZATION
  INPUTS
    client_age = user as number

PHASES
  PHASE "Week One" (1 weeks):
    WEEK 1:
```

Expected: `personalization = {inputs: [{id: "client_age", type: "number", source: "user"}], rules: []}`.

`meta.json`:
```json
{
  "since": "1.0.0",
  "description": "PERSONALIZATION with inputs and no RULES. Tests that empty rules array is accepted and not emitted as null.",
  "category": "regression"
}
```

---

## Task 11: Final verification, commit, and push

**Files:**
- No new files — just verification and commit.

- [ ] **Step 1: Count all new fixtures**

```bash
find /Users/alex/Projects/my/gymbile.com/wpl/conformance/compile/fixtures -name "source.wpl" | sort | wc -l
find /Users/alex/Projects/my/gymbile.com/wpl/conformance/compile/fixtures -mindepth 2 -maxdepth 2 -type d | sort
```

Verify all expected categories exist.

- [ ] **Step 2: Run full TS conformance suite**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl-ai && npm test 2>&1 | tail -5
```

Expected: all tests pass with the new fixture count.

- [ ] **Step 3: Run full Elixir conformance suite**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl-ai-ex && mix test test/conformance_test.exs 2>&1 | tail -3
```

Expected: all tests pass.

- [ ] **Step 4: Fix any failures**

If any fixture fails:
- Compile both compilers and compare outputs manually
- Check if divergence is a genuine compiler bug → document it, skip the fixture
- If normalization doesn't cover it → document as new divergence, simplify or skip fixture

- [ ] **Step 5: Commit batch 2 fixtures to wpl repo**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl
git status
git add conformance/compile/fixtures/meditation/ \
        conformance/compile/fixtures/habit/ \
        conformance/compile/fixtures/sub_plan/ \
        conformance/compile/fixtures/athlete_thresholds/ \
        conformance/compile/fixtures/personalization/ \
        conformance/compile/fixtures/progress/ \
        conformance/compile/fixtures/contraindications/ \
        conformance/compile/fixtures/requires/ \
        conformance/compile/fixtures/goals/ \
        conformance/compile/fixtures/regression/
git commit -m "feat(conformance): batch 2 fixtures (meditation/habit/sub_plan/athlete_thresholds/personalization/progress/contraindications/requires/goals/regression)"
git push origin master
```

- [ ] **Step 6: Verify push succeeded**

```bash
cd /Users/alex/Projects/my/gymbile.com/wpl && git log --oneline -3
```

Report: DONE or DONE_WITH_CONCERNS with final fixture count and any skipped fixtures.

---

## Known Risk Areas to Investigate During Execution

1. **Checkpoint IDs**: `generateShortId("cp")` may produce non-deterministic IDs that diverge between compiler runs. Check what it produces and whether it needs a new normalization rule.

2. **Progress streaks**: The `streaks enabled` DSL emits a streaks object, but the compiler may or may not include it in the JSON output (the `compileProgress` function registers a pointer but doesn't currently emit `streaks` into the compiled JSON). Test before creating the fixture.

3. **Sub-plan name null vs omitted**: When no name is given, the parser sets `name: null`. The compiler's `compact()` removes null fields. Check that both compilers agree on omitting `name` when not provided.

4. **ATHLETE_THRESHOLDS `vo2max`**: The DSL uses `vo2max N` (bare number, no unit) and emits `vo2max_ml_kg_min`. Check if the Elixir compiler handles the float field correctly after normalization rule 5 (coerce whole-number floats).

5. **Personalization `source` back-compat**: Sources like `user`, `wellness`, `device`, `plan`, `assessment` — verify all accepted by both parsers.

6. **`requires-time-commitment`**: The `time:` block requires `days_per_week N..N` and `minutes_per_day N..N`. Check exact DSL keyword acceptance in the parser.
