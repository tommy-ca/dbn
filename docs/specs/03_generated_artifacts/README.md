# Phase 3 – Generated Artifacts & Examples

Artifacts in this phase are produced from the protobuf definitions (`proto/`) via `buf generate` plus supporting tooling.

## Directories

- [`generated/`](generated/README.md) – Placeholder for Buf doc outputs or any other derived reference material you want to ship alongside the specs.
- [`examples/`](examples/README.md) – Sample payloads, fixtures, and planned language walkthroughs for conformance testing.

## Other generated assets

- `gen/jsonschema/` – JSON Schemas
- `gen/go`, `gen/python`, `gen/ts` – Language SDKs

Keep this area synchronized with the protobuf source: regenerate SDKs + docs whenever `.proto` changes merge to main.
