# WPL - Wellness Plan Language Specification

**Version:** 1.7.0  
**Status:** Living document (schema/v1.schema.json is normative)  
**Last Updated:** 2026-06-12

> **Normative source:** `schema/v1.schema.json` plus `conformance/` fixtures are
> the contract. Where this prose document and the schema disagree, the schema
> wins. Known prose sections pending rewrite are marked **[STALE]**.

## Overview

WPL (Wellness Plan Language) is a JSON-based markup language designed to describe comprehensive wellness plans that can include workouts, nutrition, meditation, recovery, and other wellness activities. It enables:

- **Trainers**: Create structured, reusable plans via a visual constructor
- **Backend**: Store, validate, and calculate personalized plans
- **Frontend/Mobile**: Render consistent visual representations and track progress

## Design Principles

1. **Platform Agnostic**: JSON format parseable by any language
2. **Extensible**: New activity types can be added without breaking existing plans
3. **Personalization**: Plans adapt based on client input (age, injuries, goals)
4. **Progress Tracking**: Built-in checkpoints and progress measurement
5. **Composable**: Plans can include sub-plans and templates

---

## ID Conventions

WPL uses two types of identifiers:

### Local IDs (Scoped)

Local IDs must be unique within their parent scope:

| Element | Scope | Example |
|---------|-------|---------|
| `phase.id` | Unique within plan | `"phase_1"` |
| `week.id` | Unique within phase | `"week_1"` |
| `day.id` | Unique within week | `"day_1"` |
| `block.id` | Unique within day | `"warmup_block"` |
| `activity.id` | Unique within day | `"exercise_1"` |

### Library References (Global)

Activities reference global library items via `*_ref` fields:

```json
{
  "id": "exercise_1",
  "type": "exercise",
  "exercise_ref": "push_up",
  "name": "Push-ups"
}
```

The `exercise_ref` maps to the exercise library's global ID. This separation allows:

- Multiple activities to reference the same exercise
- Clear distinction between plan-local and library-global identifiers

---

## Standard Units

WPL uses standardized units across all measurements:

### Time Units

| Unit | Usage |
|------|-------|
| `seconds` | Rest periods, tempo, short durations |
| `minutes` | Workout durations, meditation |
| `hours` | Long activities |
| `days` | Phase/plan durations |
| `weeks` | Phase/plan durations |

### Weight Units

| Unit | Usage |
|------|-------|
| `kg` | Metric weight (default) |
| `lbs` | Imperial weight |
| `percentage_1rm` | Percentage of 1 rep max |

### Distance Units

| Unit | Usage |
|------|-------|
| `meters` | Short distances |
| `km` | Long distances (metric) |
| `miles` | Long distances (imperial) |

### Intensity Units

| Unit | Usage |
|------|-------|
| `rpe` | Rate of Perceived Exertion (1-10) |
| `rir` | Reps in Reserve (0-5) |
| `heart_rate_zone` | HR zones (1-5) |
| `bpm` | Beats per minute |
| `pace` | min/km or min/mile |

### Tempo Format

Exercise tempo uses the format: `eccentric-pause_bottom-concentric-pause_top`

Example: `"3-1-2-0"` means:

- 3 seconds eccentric (lowering)
- 1 second pause at bottom
- 2 seconds concentric (lifting)
- 0 seconds pause at top

---

## Core Schema Structure

```json
{
  "$schema": "https://wpl.dev/schemas/wpl/v1.schema.json",
  "version": "1.0.0",
  "plan": {
    "id": "uuid",
    "name": "string",
    "description": "string",
    "type": "workout|nutrition|meditation|recovery|hybrid",
    "visibility": "private|public|template",
    
    "metadata": {
      "created_by": "trainer_id",
      "created_at": "iso8601",
      "updated_at": "iso8601",
      "tags": ["weight_loss", "beginner", "home_workout"],
      "difficulty": "beginner|intermediate|advanced|adaptive",
      "estimated_duration_days": 30,
      "language": "en"
    },
    
    "goals": [...],
    "requirements": {...},
    "personalization": {...},
    "phases": [...],
    "progress": {...},
    "notifications": {...}
  }
}
```

---

## 1. Goals Definition

Goals define what the plan aims to achieve and how success is measured.

```json
{
  "goals": [
    {
      "id": "goal_1",
      "type": "primary|secondary",
      "category": "weight_loss|muscle_gain|endurance|flexibility|mental_wellness|nutrition|habit",
      "name": "Lose 5kg",
      "description": "Achieve healthy weight loss over 8 weeks",
      "target": {
        "metric": "weight",
        "unit": "kg",
        "start_value": null,
        "target_value": -5,
        "measurement_type": "absolute|relative|percentage"
      },
      "deadline": "2024-03-01",
      "milestones": [
        {
          "id": "m1",
          "name": "First kg lost",
          "target_value": -1,
          "reward_points": 100,
          "badge_id": "first_kg_badge"
        }
      ]
    }
  ]
}
```

### Goal Categories

| Category | Description |
|----------|-------------|
| `weight_loss` | Body weight reduction |
| `muscle_gain` | Muscle mass increase |
| `endurance` | Cardiovascular fitness |
| `flexibility` | Range of motion |
| `strength` | Maximum force output |
| `mental_wellness` | Stress, sleep, mindfulness |
| `nutrition` | Dietary habits |
| `habit` | Behavioral changes |
| `custom` | Trainer-defined goals |

#### Recommended vocabularies (soft-validated)

As of WPL v1.9.0, the recommended goal categories are published as a data
vocabulary in [`data/goal-categories.json`](../data/goal-categories.json)
(union of the categories above plus `general_fitness`). `Goal.category` remains
a **free-form string** — unknown values are allowed. Validators SHOULD **warn**
(not reject) when a `category` is not in the recommended vocabulary; use
`custom` as the explicit off-list escape hatch.

Nutrition activities may likewise carry an optional `dietary_tags` array
(see [§5.3](#53-nutrition-activity)). Its recommended values are published in
[`data/dietary-tags.json`](../data/dietary-tags.json) (`vegetarian`, `vegan`,
`gluten_free`, `dairy_free`). The field is optional and free-form; validators
SHOULD warn (not reject) on off-list tags.

---

## 2. Client Requirements & Personalization

### Requirements (Plan Prerequisites)

```json
{
  "requirements": {
    "min_age": 16,
    "max_age": 65,
    "fitness_level": ["beginner", "intermediate"],
    "equipment": [
      {
        "id": "dumbbells",
        "name": "Dumbbells",
        "required": true,
        "alternatives": ["resistance_bands", "water_bottles"]
      },
      {
        "id": "yoga_mat",
        "name": "Yoga Mat",
        "required": false
      }
    ],
    "contraindications": [
      {
        "condition": "pregnancy",
        "action": "exclude",
        "message": "This plan is not suitable during pregnancy"
      },
      {
        "condition": "back_injury",
        "action": "modify",
        "affected_activities": ["deadlift", "squat"]
      }
    ],
    "time_commitment": {
      "min_days_per_week": 3,
      "max_days_per_week": 5,
      "min_minutes_per_day": 30,
      "max_minutes_per_day": 60
    }
  }
}
```

#### Contraindication Fields (v1.6.0)

Each contraindication entry supports optional `severity` (`low | moderate | high`, ACSM-style risk tiering) and the new `action` value `require_clearance`, which gates plan execution behind documented medical clearance. Existing `action` values (`exclude`, `modify`) remain valid.

```json
{"condition": "acsm:cardiac_event_recent", "severity": "high", "action": "require_clearance", "message": "Physician clearance required before starting."}
```

### Personalization Rules

Client input drives plan customization through conditional rules.

```json
{
  "personalization": {
    "inputs": [
      {
        "id": "client_age",
        "type": "number",
        "source": "client_profile.age",
        "label": "Age"
      },
      {
        "id": "client_injuries",
        "type": "array",
        "source": "client_profile.injuries",
        "label": "Current Injuries"
      },
      {
        "id": "available_equipment",
        "type": "array",
        "source": "questionnaire",
        "label": "Available Equipment"
      },
      {
        "id": "fitness_level",
        "type": "enum",
        "source": "assessment",
        "options": ["beginner", "intermediate", "advanced"]
      },
      {
        "id": "weekly_availability",
        "type": "number",
        "source": "questionnaire",
        "label": "Days per week available"
      }
    ],
    
    "rules": [
      {
        "id": "age_intensity_adjustment",
        "condition": {
          "operator": "and",
          "conditions": [
            {"field": "client_age", "op": ">=", "value": 50}
          ]
        },
        "actions": [
          {"type": "modify_intensity", "factor": 0.8, "scope": "plan"},
          {"type": "add_warmup_time", "minutes": 5, "scope": "day"},
          {"type": "add_activity", "activity_id": "joint_mobility", "when": "before_workout"}
        ]
      },
      {
        "id": "knee_injury_modification",
        "condition": {
          "field": "client_injuries",
          "op": "contains",
          "value": "knee"
        },
        "actions": [
          {"type": "replace_exercise", "from": "squat", "to": "wall_sit", "scope": "plan"},
          {"type": "replace_exercise", "from": "lunges", "to": "step_ups_low", "scope": "plan"},
          {"type": "exclude_exercise", "exercise_id": "jump_squat", "scope": "plan"}
        ]
      },
      {
        "id": "equipment_substitution",
        "condition": {
          "field": "available_equipment",
          "op": "not_contains",
          "value": "barbell"
        },
        "actions": [
          {"type": "replace_exercise", "from": "barbell_squat", "to": "goblet_squat", "scope": "plan"}
        ]
      }
    ]
  }
}
```

### Action Scope

The `scope` field defines where personalization actions apply:

| Scope | Description |
|-------|-------------|
| `activity` | Only the specific activity |
| `block` | All activities in the current block |
| `day` | All activities in the current day |
| `week` | All activities in the current week |
| `phase` | All activities in the current phase |
| `plan` | All activities in the entire plan (default) |

---

## 3. Plan Phases

Plans are organized into phases, each containing weeks, days, and activities.

```json
{
  "phases": [
    {
      "id": "phase_1",
      "name": "Foundation",
      "description": "Build base fitness and learn proper form",
      "order": 1,
      "duration": {
        "value": 2,
        "unit": "weeks"
      },
      "goals": ["goal_1"],
      "unlock_condition": null,
      
      "weeks": [
        {
          "id": "week_1",
          "name": "Week 1",
          "order": 1,
          "theme": "Introduction",
          "days": [...]
        }
      ]
    },
    {
      "id": "phase_2",
      "name": "Progression",
      "order": 2,
      "duration": {"value": 4, "unit": "weeks"},
      "unlock_condition": {
        "type": "phase_complete",
        "phase_id": "phase_1"
      }
    }
  ]
}
```

---

## 4. Day Structure

Each day contains multiple activity blocks.

```json
{
  "days": [
    {
      "id": "day_1",
      "day_of_week": 1,
      "name": "Upper Body Focus",
      "type": "training|rest|active_recovery|assessment",
      "estimated_duration_minutes": 45,
      
      "schedule": {
        "preferred_time": "morning|afternoon|evening|any",
        "flexibility": "strict|flexible"
      },
      
      "blocks": [
        {
          "id": "warmup_block",
          "type": "warmup",
          "order": 1,
          "activities": [...]
        },
        {
          "id": "main_block",
          "type": "main",
          "order": 2,
          "structure": "circuit|straight_sets|superset|emom|amrap|tabata",
          "rounds": 3,
          "rest_between_rounds": {"value": 90, "unit": "seconds"},
          "activities": [...]
        },
        {
          "id": "cooldown_block",
          "type": "cooldown",
          "order": 3,
          "activities": [...]
        }
      ],
      
      "nutrition_guidance": {...},
      "notes": "Focus on controlled movements"
    }
  ]
}
```

### Block Types

| Type | Description |
|------|-------------|
| `warmup` | Preparation activities |
| `main` | Primary workout content |
| `cooldown` | Recovery activities |
| `nutrition` | Meal/supplement timing |
| `meditation` | Mindfulness activities |
| `education` | Learning content |
| `assessment` | Progress checks |

`Block.type` constrains which `Activity.type` values are permitted inside `Block.activities`. Placing an activity whose type is not allowed for the parent block's type is a semantic error (`ACTIVITY_BLOCK_MISMATCH`). See [conformance/error-codes.md](../conformance/error-codes.md) for the full allowed-activity table.

---

## 5. Activity Types

### 5.1 Exercise Activity

**[STALE — see schema $defs.ExerciseActivity]** Fields `category`, `muscle_groups`, `progression`, `alternatives`, `media`, and `tracking` shown in the example below are not present in the schema's `ExerciseActivity` definition. Do not rely on this prose for the authoritative field list.

```json
{
  "id": "exercise_1",
  "type": "exercise",
  "exercise_ref": "push_up",
  "name": "Push-ups",
  "category": "strength|cardio|flexibility|balance|plyometric",
  "muscle_groups": ["chest", "triceps", "shoulders"],
  
  "prescription": {
    "type": "sets_reps|time|distance|amrap",
    "sets": 3,
    "reps": {"min": 8, "max": 12, "target": 10},
    "weight": {
      "type": "bodyweight|absolute|percentage_1rm|rpe",
      "value": null,
      "unit": "kg"
    },
    "tempo": "3-1-2-0",
    "rest": {"value": 60, "unit": "seconds"}
  },
  
  "progression": {
    "type": "linear|wave|autoregulated",
    "increment": {"value": 2, "unit": "reps", "per": "week"}
  },
  
  "alternatives": [
    {
      "exercise_ref": "knee_push_up",
      "condition": "easier"
    },
    {
      "exercise_ref": "diamond_push_up",
      "condition": "harder"
    }
  ],
  
  "media": {
    "video_url": "https://...",
    "thumbnail_url": "https://...",
    "instructions": ["Start in plank position", "Lower chest to ground", "Push back up"]
  },
  
  "tracking": {
    "log_weight": true,
    "log_reps": true,
    "log_rpe": true,
    "log_notes": true
  }
}
```

#### ExercisePrescription / Reps / Weight Fields (v1.6.0)

`ExercisePrescription` accepts optional `to_failure: true` to signal that the final set should be taken to muscular failure. `Reps` accepts optional `amrap: true` for "as many reps as possible" sets. For `percentage_1rm` weight prescriptions, `Weight.metric` clarifies the reference: `1RM | e1RM | training_max | daily_max`.

```json
{"type": "sets_reps", "sets": 3, "reps": {"min": 5, "max": 8}, "to_failure": true, "weight": {"type": "percentage_1rm", "value": 80, "metric": "e1RM"}}
```

### 5.2 Cardio Activity

```json
{
  "id": "cardio_1",
  "type": "cardio",
  "name": "Interval Running",
  "modality": "running|cycling|swimming|rowing|elliptical|jump_rope",
  
  "prescription": {
    "type": "continuous|intervals|fartlek",
    "duration": {"value": 20, "unit": "minutes"},
    "intensity": {
      "type": "heart_rate_zone|rpe|pace",
      "target": {"zone": 3, "min": 130, "max": 150}
    },
    "intervals": [
      {"work": {"duration": 30, "intensity": "high"}, "rest": {"duration": 30, "intensity": "low"}},
      {"repeat": 10}
    ]
  },
  
  "tracking": {
    "log_distance": true,
    "log_duration": true,
    "log_heart_rate": true,
    "log_calories": true
  }
}
```

#### CardioPrescription Fields (v1.6.0)

`intervals.work.duration` and `intervals.rest.duration` accept either a bare number (seconds, back-compat) or a full `Duration` object (`{"value": 40, "unit": "seconds"}`). Bare-number form is deprecated in favour of the object form.

`intensity.target` is an open object (`additionalProperties: true`) with pre-defined named slots: `zone` (integer), `min_bpm`/`max_bpm` (integers), `min_watts`/`max_watts` (numbers), and `value`+`unit` for pace where `unit` is one of `min_per_km | min_per_mi | m_per_s | sec_per_100m`.

```json
{"type": "bpm", "target": {"min_bpm": 130, "max_bpm": 150}, "zone_model": "hr_5_zone"}
```

### 5.3 Nutrition Activity

```json
{
  "id": "nutrition_1",
  "type": "nutrition",
  "category": "meal|snack|supplement|hydration",
  "name": "Post-Workout Protein",
  "dietary_tags": ["vegan"],
  "timing": {
    "type": "relative|absolute",
    "reference": "workout_end",
    "offset": {"value": 30, "unit": "minutes"}
  },
  
  "prescription": {
    "macros": {
      "protein": {"min": 25, "max": 35, "unit": "g"},
      "carbs": {"min": 30, "max": 50, "unit": "g"},
      "fat": {"max": 10, "unit": "g"}
    },
    "calories": {"min": 300, "max": 400},
    "suggestions": [
      "Protein shake with banana",
      "Greek yogurt with berries",
      "Chicken breast with rice"
    ]
  },
  
  "tracking": {
    "log_consumed": true,
    "log_actual_macros": false,
    "photo_required": false
  }
}
```

### 5.4 Meditation Activity

```json
{
  "id": "meditation_1",
  "type": "meditation",
  "category": "breathing|mindfulness|visualization|body_scan|sleep",
  "name": "Morning Mindfulness",
  
  "prescription": {
    "duration": {"value": 10, "unit": "minutes"},
    "guided": true,
    "audio_id": "meditation_audio_123"
  },
  
  "content": {
    "introduction": "Find a comfortable seated position...",
    "steps": [
      {"duration": 60, "instruction": "Focus on your breath"},
      {"duration": 120, "instruction": "Body scan from head to toe"},
      {"duration": 60, "instruction": "Return awareness to the room"}
    ]
  },
  
  "tracking": {
    "log_completed": true,
    "log_mood_before": true,
    "log_mood_after": true
  }
}
```

### 5.5 Recovery Activity

```json
{
  "id": "recovery_1",
  "type": "recovery",
  "category": "stretching|foam_rolling|massage|cold_therapy|heat_therapy|sleep",
  "name": "Evening Stretch Routine",
  
  "prescription": {
    "duration": {"value": 15, "unit": "minutes"},
    "exercises": [
      {
        "name": "Hamstring Stretch",
        "hold_time": {"value": 30, "unit": "seconds"},
        "sides": "both",
        "reps": 2
      }
    ]
  },
  
  "tracking": {
    "log_completed": true,
    "log_soreness_level": true
  }
}
```

#### RecoveryExercise Fields (v1.6.0)

Each entry in `prescription.exercises` (a `RecoveryExercise`) accepts optional: `modality` (`static | dynamic | PNF | SMR | breathwork | mobility`), `intensity_rpe` (1–10 number), `body_part` (free string), and a structured `pnf` block (`{"contract_seconds": 6, "relax_seconds": 10, "reps": 3}`).

```json
{"name": "Hip Flexor PNF", "modality": "PNF", "body_part": "hip_flexor", "pnf": {"contract_seconds": 6, "relax_seconds": 10, "reps": 3}, "intensity_rpe": 4}
```

### 5.6 Habit Activity

```json
{
  "id": "habit_1",
  "type": "habit",
  "category": "hydration|sleep|steps|screen_time|custom",
  "name": "Daily Water Intake",
  
  "prescription": {
    "target": {"value": 8, "unit": "glasses"},
    "frequency": "daily",
    "reminders": [
      {"time": "09:00", "message": "Morning hydration check"},
      {"time": "14:00", "message": "Afternoon water reminder"}
    ]
  },
  
  "tracking": {
    "log_count": true,
    "streak_enabled": true
  }
}
```

### `simple` and `recovery_exercise`

These activity types are produced by the WPL-AI compiler as intermediate forms when the source DSL doesn't carry full structural information. Authored plans typically use the typed forms documented above (`exercise`, `cardio`, `nutrition`, `meditation`, `recovery`, `habit`); tools that consume compiler output should accept both.

---

## 6. Progress Tracking

**[STALE — see schema $defs.Checkpoint]** The checkpoint example below uses `trigger` (object with `type`/`value`) and `questionnaire` (array). The schema uses `at` (integer week number) and `questions` (array of strings). The `trigger`/`questionnaire` field names are not accepted by the schema.

**[STALE — see schema $defs.Progress]** The `progress.achievements` array shown below is not present in the schema's `Progress` definition, which only has `checkpoints`, `points_system`, and `streaks`.

```json
{
  "progress": {
    "checkpoints": [
      {
        "id": "checkpoint_1",
        "name": "Week 2 Check-in",
        "trigger": {
          "type": "time|completion|manual",
          "value": {"week": 2, "day": 7}
        },
        "measurements": [
          {"metric": "weight", "unit": "kg"},
          {"metric": "body_fat", "unit": "percentage"},
          {"metric": "photos", "views": ["front", "side", "back"]},
          {"metric": "measurements", "areas": ["chest", "waist", "hips"]}
        ],
        "questionnaire": [
          {"question": "How is your energy level?", "type": "scale_1_10"},
          {"question": "Any pain or discomfort?", "type": "text"}
        ]
      }
    ],
    
    "points_system": {
      "enabled": true,
      "rules": [
        {"action": "complete_workout", "points": 10},
        {"action": "complete_day", "points": 25},
        {"action": "complete_week", "points": 100},
        {"action": "log_meal", "points": 5},
        {"action": "complete_meditation", "points": 15},
        {"action": "streak_7_days", "points": 200}
      ]
    },
    
    "achievements": [
      {
        "id": "first_week",
        "name": "First Week Champion",
        "description": "Complete your first week",
        "condition": {"type": "weeks_completed", "value": 1},
        "badge_image": "badge_first_week.png",
        "points": 500
      }
    ],
    
    "streaks": {
      "enabled": true,
      "types": ["daily_workout", "daily_nutrition", "daily_meditation"]
    }
  }
}
```

#### Checkpoint Measurements (v1.6.0)

`Checkpoint.measurements[]` items now accept either a free string (back-compat) or a `MeasurementSpec` object. `MeasurementSpec` references `MeasurementMetric` (24-value enum covering body composition, hemodynamic, cardiorespiratory, strength, flexibility, sleep, and RPE metrics) and an optional `Questionnaire` enum (`PHQ-9 | GAD-7 | IPAQ | PSQI | PSS-10 | Borg_CR-10 | session_RPE`).

```json
{"measurements": ["body_weight", {"metric": "resting_hr", "unit": "bpm"}, {"questionnaire": "PHQ-9"}]}
```

---

## 7. Notifications & Reminders

**[STALE — see schema $defs.Notification]** The example below shows `notifications` as a keyed object. The schema defines `notifications` as an array of `Notification` items (each with `id`, `enabled`, `message`, `timing_offset`, `timing_reference`). The object shape shown here is not accepted by the schema.

```json
{
  "notifications": {
    "workout_reminder": {
      "enabled": true,
      "timing": {"value": 30, "unit": "minutes", "before": "scheduled_time"},
      "message_template": "Time for {{workout_name}}! 💪"
    },
    "rest_day_motivation": {
      "enabled": true,
      "message_template": "Rest day! Your muscles are growing stronger 🌱"
    },
    "streak_at_risk": {
      "enabled": true,
      "trigger": {"hours_remaining": 4},
      "message_template": "Don't break your {{streak_days}} day streak!"
    },
    "milestone_achieved": {
      "enabled": true,
      "message_template": "🎉 Milestone reached: {{milestone_name}}"
    }
  }
}
```

---

## 8. Exercise Library Reference

Plans reference exercises from a central library.

```json
{
  "exercise_library": {
    "push_up": {
      "id": "push_up",
      "name": "Push-up",
      "aliases": ["press-up"],
      "category": "strength",
      "equipment": ["none"],
      "muscle_groups": {
        "primary": ["chest", "triceps"],
        "secondary": ["shoulders", "core"]
      },
      "difficulty": "beginner",
      "instructions": [...],
      "common_mistakes": [...],
      "video_url": "...",
      "thumbnail_url": "...",
      "contraindications": ["wrist_injury", "shoulder_injury"]
    }
  }
}
```

---

## 9. Template System

Trainers can create reusable templates.

```json
{
  "template": {
    "id": "template_1",
    "name": "HIIT Circuit Template",
    "type": "block",
    "parameters": [
      {"name": "exercises", "type": "exercise_list", "min": 4, "max": 8},
      {"name": "work_time", "type": "duration", "default": 40},
      {"name": "rest_time", "type": "duration", "default": 20},
      {"name": "rounds", "type": "number", "default": 3}
    ],
    "structure": {
      "type": "circuit",
      "rounds": "{{rounds}}",
      "activities": "{{exercises}}",
      "work_duration": "{{work_time}}",
      "rest_duration": "{{rest_time}}"
    }
  }
}
```

---

## 10. Validation Rules

### Required Fields by Plan Type

| Plan Type | Required Sections |
|-----------|-------------------|
| `workout` | goals, phases, at least one exercise activity |
| `nutrition` | goals, nutrition activities |
| `meditation` | goals, meditation activities |
| `hybrid` | goals, phases, at least 2 activity types |

### Constraints

- Phase duration must be > 0
- Day activities must have unique IDs within the day
- Exercise references must exist in library or be inline-defined
- Personalization rules must reference valid input fields
- Progress checkpoints must fall within plan duration

### Duration Consistency

When a phase specifies both `duration` and `weeks` array, they must be consistent:

| Duration | Weeks Array | Valid? |
|----------|-------------|--------|
| 2 weeks | 2 items | ✅ Yes |
| 2 weeks | 3 items | ⚠️ Warning (weeks array takes precedence) |
| 14 days | 2 items | ✅ Yes |
| Not specified | Any | ✅ Yes (duration inferred from weeks) |

**Validation behavior:**

- If `duration` is specified but `weeks` array has different count, emit a warning
- The `weeks` array is the source of truth for actual content
- `duration` is used for display and progress calculations when weeks array is empty

---

## 11. Rendering Hints

For consistent UI rendering across platforms:

```json
{
  "rendering": {
    "color_scheme": {
      "primary": "#4F46E5",
      "secondary": "#10B981",
      "accent": "#F59E0B"
    },
    "activity_icons": {
      "exercise": "dumbbell",
      "cardio": "heart",
      "nutrition": "utensils",
      "meditation": "brain",
      "recovery": "bed",
      "habit": "check-circle"
    },
    "difficulty_colors": {
      "beginner": "#10B981",
      "intermediate": "#F59E0B",
      "advanced": "#EF4444"
    }
  }
}
```

---

## 12. API Integration

### Endpoints (Future)

```text
POST   /api/v1/plans                    # Create plan
GET    /api/v1/plans/:id                # Get plan
PUT    /api/v1/plans/:id                # Update plan
DELETE /api/v1/plans/:id                # Delete plan
POST   /api/v1/plans/:id/personalize    # Generate personalized version
POST   /api/v1/plans/:id/assign         # Assign to client
GET    /api/v1/plans/:id/progress       # Get progress data
POST   /api/v1/plans/:id/log            # Log activity completion
```

---

## 13. Plan Assembly Pipeline

When assigning a plan to a client, the following pipeline is executed:

```text
┌─────────────────────────────────────────────────────────────────┐
│                    PLAN ASSEMBLY PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LOAD PLAN                                                   │
│     └─ Parse WPL JSON → Validate schema                         │
│                                                                 │
│  2. COLLECT CLIENT DATA                                         │
│     └─ Gather inputs: age, injuries, equipment, fitness level   │
│                                                                 │
│  3. EVALUATE PERSONALIZATION RULES                              │
│     └─ For each rule:                                           │
│        ├─ Evaluate condition against client data                │
│        └─ If true, queue actions for application                │
│                                                                 │
│  4. APPLY ACTIONS (in order)                                    │
│     └─ modify_intensity → replace_exercise → exclude_exercise   │
│        → reduce_sets → reduce_reps → increase_rest              │
│        → add_warmup_time → add_activity                         │
│                                                                 │
│  5. GENERATE PERSONALIZED INSTANCE                              │
│     └─ Create immutable copy with applied modifications         │
│                                                                 │
│  6. VALIDATE RESULT                                             │
│     └─ Ensure personalized plan is still valid                  │
│                                                                 │
│  7. ASSIGN TO CLIENT                                            │
│     └─ Store assignment with start_date, personalized_plan      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key principles:**

- Original plan is never modified (immutable)
- Personalized instance is stored separately per client
- Client can have multiple assignments of the same plan
- Progress is tracked against the personalized instance

---

## 14. Versioning

Plans include version information for backward compatibility:

```json
{
  "$schema": "https://wpl.dev/schemas/wpl/v1.schema.json",
  "version": "1.0.0",
  "min_app_version": "2.0.0"
}
```

---

## Example: Complete 4-Week Weight Loss Plan

See `examples/weight_loss_4_week.wpl.json` for a full implementation.

---

## Changelog

### v1.7.0 (2026-06-12)

- in/not_in, nested compounds, typed actions, version range; prose sections marked STALE pending rewrite.

### v1.1.0 (2024-11-24)

- Added ID conventions section (local vs global IDs)
- Added standard units registry (time, weight, distance, intensity)
- Added tempo format specification
- Added `scope` field to personalization actions
- Added duration consistency validation rules
- Added Plan Assembly Pipeline documentation
- Added RIR (Reps in Reserve) as intensity unit

### v1.0.0 (2024-11-24)

- Initial specification
- Core activity types: exercise, cardio, nutrition, meditation, recovery, habit
- Personalization engine with conditional rules
- Progress tracking with points and achievements
- Template system for reusable components
