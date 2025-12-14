# WPL Examples

These are reference WPL plans demonstrating the language's capabilities. They are also used as fixtures in [`conformance/valid/`](../conformance/valid/) — every validator implementation must accept them.

| File | Description |
|---|---|
| [`simple-workout.json`](simple-workout.json) | Minimal workout plan: one phase, one week, two days. Good starting point. |
| [`hiit-circuit.json`](hiit-circuit.json) | HIIT circuit demonstrating block types and time-based prescriptions. |
| [`holistic-plan.json`](holistic-plan.json) | Full holistic plan: workout + nutrition + meditation activities, personalization rules, progress tracking. |

All examples reference `https://wpl.dev/schemas/wpl/v1.schema.json`. To validate locally:

```bash
npx -y ajv-cli validate -s ../schema/v1.schema.json -d "*.json" --spec=draft2020 --strict=false
```
