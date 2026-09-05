# Phase 2 – Schema Implementation & Tooling

This phase translates the canonical spec into portable protobuf schemas plus tooling guidance.

## Files

- [SCHEMA_MANAGEMENT.md](SCHEMA_MANAGEMENT.md) – Working guide for linting, generation, and review expectations.
- [IMPLEMENTATION_SUMMARY.md](IMPLEMENTATION_SUMMARY.md) – Snapshot of what was delivered during the multi-format schema refresh.
- [HIERARCHICAL_SCHEMA_SUMMARY.md](HIERARCHICAL_SCHEMA_SUMMARY.md) – Rationale and inventory for the multi-file protobuf layout.

## Protobuf source

- Directory: [`proto/`](../../../proto)
- Structure: `databento/dbn/v3/` subdivided into `common`, `enums`, `messages/*`, and `metadata`.
- Tooling: Managed by `buf.yaml` + `buf.gen.yaml` at the repo root.

Propose or review protobuf edits only after Phase 1 has been updated and approved.
