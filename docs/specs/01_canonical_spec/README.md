# Phase 1 – Canonical Specification

This phase houses the binary contract for DBN. Everything downstream (protobuf, JSON Schema, SDKs) must map back to these documents.

## Files

- [DBN_SCHEMA_SPECIFICATION.md](DBN_SCHEMA_SPECIFICATION.md) – Canonical binary layout, enums, sentinel values, and version history for the DBN format.

## Workflow notes

- Use this spec to settle debates about field sizes, encoding, or compatibility guarantees.
- When proposing a schema change, update this file before touching protobuf so that review begins with the binary truth.
