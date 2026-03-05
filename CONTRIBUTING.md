# Contributing to WPL

Thanks for your interest in improving WPL.

## Repository Layout

| Path | Purpose | License |
|---|---|---|
| `spec/SPECIFICATION.md` | Canonical language spec | CC-BY-4.0 |
| `schema/v1.schema.json` | JSON Schema (Draft 2020-12) | Apache-2.0 |
| `examples/` | Reference plans | CC-BY-4.0 |
| `conformance/` | Test fixtures all validators must satisfy | Apache-2.0 |

## Proposing Changes

1. **Spec changes** — open an issue first to discuss. Spec changes are versioned: additive changes ship in a v1.x.y release; breaking changes require a v2 line.
2. **Schema changes** — must accompany a spec change. Run `npx -y ajv-cli@5 compile -s schema/v1.schema.json --spec=draft2020 --strict=false` locally before opening a PR.
3. **Conformance fixtures** — every new error rule needs a fixture. Add the input under `conformance/invalid/<rule>.json` and the expected output under `conformance/invalid/<rule>.expected.json`. See [`conformance/README.md`](conformance/README.md).
4. **Validator implementations** — live in separate repos (`wpl-validator-ts`, `wpl-validator-ex`). File issues against those repos for implementation bugs.

## Release Process

Maintainers tag releases as `vX.Y.Z`. The release workflow opens a PR to `gymbile/wpl.dev` mirroring the schema and spec.

## Trademark

See the [README](README.md#trademark) for trademark and naming policy. In short: implementations are encouraged ("WPL-compatible"), but the name "WPL" itself is reserved.
