# WPL — Wellness Plan Language

**An open, structured format for wellness plans.**

WPL is a JSON-based language for describing comprehensive wellness programs that include workouts, nutrition, meditation, recovery, and habits. It enables platforms, trainers, and AI systems to create, validate, personalize, and track wellness plans consistently.

This repository is the **source of truth** for the WPL specification, JSON Schema, and conformance test suite.

- 📖 [Specification](spec/SPECIFICATION.md)
- 📐 [JSON Schema](schema/v1.schema.json) (canonical URL: `https://wpl.dev/schemas/wpl/v1.schema.json`)
- 📦 [Examples](examples/)
- 🧪 [Conformance suite](conformance/)
- 🌐 [wpl.dev](https://wpl.dev) — language website + interactive playground

## Implementations

Reference validators (consume the schema and conformance suite from this repo):

| Language | Repo | Package |
|---|---|---|
| TypeScript | `gymbile/wpl-validator-ts` | `@gymbile/wpl-validator` (npm) |
| Elixir | `gymbile/wpl-validator-ex` | `wpl_validator` (hex) |

Both implementations must produce identical results for the [conformance suite](conformance/).

## Versioning

The WPL schema follows semver. The canonical schema URL stays at `/v1.schema.json` for the entire v1 line — additive changes only. A breaking change becomes `/v2.schema.json` at a new URL; existing v1 plans keep working.

## Licensing

This repository uses split licensing:

| Path | License |
|---|---|
| `spec/`, `examples/` | [CC-BY-4.0](LICENSE-SPEC) — documentation, free to use with attribution |
| `schema/`, `conformance/`, `.github/` | [Apache-2.0](LICENSE-CODE) — code, with patent grant |

## Trademark

"WPL" and "Wellness Plan Language" are trademarks of Gymbile. The spec and reference implementations are open under the licenses above. Implementations may declare conformance ("WPL-compatible" or "WPL v1 compliant") but **may not be named "WPL"** or imply endorsement by Gymbile. Forks must rename.

## Conformance

An implementation that passes every fixture in [`conformance/`](conformance/) may declare itself **"WPL v1 compliant."** See [`conformance/README.md`](conformance/README.md) for details.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
