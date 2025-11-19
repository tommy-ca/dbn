# DBN Schema Management Guide

**Date**: 2025-11-13
**Status**: ✅ Complete

This document describes how schemas are managed in the DBN project using Protocol Buffers and the Buf CLI.

## Prerequisites

Before working with DBN schemas, ensure you have the following installed:

### Required
- **Buf CLI** (v1.20.0 or later) - [Install Buf](https://buf.build/docs/installation)
  ```bash
  # macOS
  brew install bufbuild/buf/buf

  # Linux
  curl -sSL "https://github.com/bufbuild/buf/releases/download/v1.20.0/buf-Linux-x86_64" -o /usr/local/bin/buf
  chmod +x /usr/local/bin/buf

  # Verify installation
  buf --version
  ```

- **Go** (v1.19 or later) - Required for Buf's Go codegen plugin
  ```bash
  go version  # verify installation
  ```

### Optional (for language-specific code generation)
- **Python 3.8+** - For Python protobuf bindings
- **Node.js 14+** - For TypeScript protobuf bindings
- **Rust 1.56+** - If modifying Rust struct definitions

### Recommended Tools
- **VS Code** with Protobuf extension for editing .proto files
- **Git** (v2.0+) for version control

## Overview

The DBN project uses a multi-format schema approach:

1. **Rust structs** (`rust/dbn/src/record.rs`) - Primary source of truth for binary format
2. **Protocol Buffers** (`proto/`) - Language-agnostic schema definitions
3. **JSON Schemas** (`gen/jsonschema/`) - Validation schemas for JSON format

## Specs-Driven Development Flow

| Phase | Goal | Primary Artifacts | Key References |
| --- | --- | --- | --- |
| 0. Requirements | Capture latency, regulatory, and product constraints | `00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md`, `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md` | Business & product owners |
| 1. Canonical Spec | Maintain the binary DBN contract | `01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md` | DBN spec working group |
| 2. Schema Implementation | Translate spec changes into protobuf + Buf configs | `proto/`, this guide, `proto/README.md` | Schema engineers |
| 3. Generated Artifacts & Validation | Produce JSON Schema + SDKs, verify with fixtures | `gen/`, `docs/specs/03_generated_artifacts/generated/`, `docs/specs/03_generated_artifacts/examples/` | Platform + client teams |

Each pull request should link work back to a phase to keep provenance clear. For example, new crypto fields start in Phase 0/1 and then flow through Phase 2 and 3 when protobuf definitions and JSON Schemas are regenerated.

### Crypto Schema Backlog (Phase 2 entry criteria)

- **New protobuf packages** – Create `proto/databento/dbn/v3/messages/crypto/` for the RTypes listed in the canonical spec (0xD0‑0xE1). Each message should mirror the Rust structs in `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`.
- **Enum extensions** – Append the crypto instrument classes and RTypes to `enums/rtype.proto` and `enums/instrument.proto` once the canonical spec updates land.
- **Shared crypto primitives** – Introduce reusable proto messages for hashes (32 bytes), addresses (20 bytes), and block metadata so DexSwap/Liquidation/GasPrice messages do not duplicate field definitions.
- **Parity verification** – For every new schema, document which cryptofeed/tardis payload it replaces in `docs/specs/02_schema_implementation/IMPLEMENTATION_SUMMARY.md`.

For a cross-reference between the canonical DBN spec and the hierarchical
protobuf layout (including the current status of the crypto RTypes 0xD0‑0xE1),
see `DBN_PROTO_SCHEMA_MAPPING.md`.

For a proposed layout and draft message shapes for the `messages/crypto/`
package (including FundingRateMsg, LiquidationMsg, DexSwapMsg, and related
records), see `CRYPTO_PROTO_LAYOUT_PROPOSAL.md`. Treat that document as the
design reference when promoting the crypto structs from the requirements docs
into concrete `.proto` definitions.

#### Suggested Implementation Order (Crypto Schemas)

To keep the crypto workstream manageable and aligned with the requirements
priority tables, implement schemas roughly in this order:

1. **P0 Derivatives (market data critical)**
   - `FundingRateMsg` (0xD0) – Perpetual funding rates
   - `LiquidationMsg` (0xD1) – Liquidation events
   - `MarkPriceMsg` (0xD2) – Perp mark price
   - `IndexPriceMsg` (0xD3) – Spot index price

2. **P0–P1 DEX / DeFi**
   - `DexSwapMsg` (0xD4) – AMM swaps
   - `DexPoolStateMsg` (0xD5) – Pool state snapshots
   - `OraclePriceMsg` (0xD6) – Oracle price feeds

3. **P1 Cross-Market Analytics**
   - `CrossRateMsg` (0xD8) – Cross-venue spreads
   - `BasisMsg` (0xD9) – Spot–futures basis
   - `StablecoinPegMsg` (0xDA) – Stablecoin peg monitoring

4. **P2 Infrastructure / On-Chain Telemetry**
   - `GasPriceMsg` (0xDB)
   - `BlockInfoMsg` (0xDC)
   - `SmartContractEventMsg` (0xDD)
   - `InsuranceFundMsg` (0xDE)
   - `ClawbackMsg` (0xDF)
   - `DelistingMsg` (0xE0)
   - `TokenMigrationMsg` (0xE1)

Each group should go through the full specs-driven flow:

1. Validate binary struct definitions and sizes in the requirements doc.
2. Update the canonical spec (`01_canonical_spec/`) once layouts are stable.
3. Implement matching Rust `#[repr(C)]` structs.
4. Add protobuf messages under `messages/crypto/` per
   `CRYPTO_PROTO_LAYOUT_PROPOSAL.md`.
5. Extend enums (`RType`, `Schema`, crypto-specific enums) append-only.
6. Regenerate artifacts and add validation fixtures.

## Directory Structure

```
dbn/
├── buf.yaml                    # Buf workspace configuration
├── buf.gen.yaml                # Code generation configuration
├── proto/                      # Protobuf schema definitions
│   └── databento/dbn/v3/
│       ├── common/
│       ├── enums/
│       ├── messages/{aggregated,market_data,reference,system}
│       └── metadata/
├── docs/
│   └── specs/
│       ├── 00_requirements/    # Market requirements + briefs
│       ├── 01_canonical_spec/  # Binary DBN spec
│       ├── 02_schema_implementation/
│       ├── 03_generated_artifacts/
│       └── 04_validation/
├── gen/
│   ├── jsonschema/             # JSON schema definitions
│   ├── go/
│   ├── python/
│   └── ts/
└── rust/
    └── dbn/
        └── src/
            └── record.rs       # Rust struct definitions
```

## Schema Formats

### Binary Format (Rust)

The canonical DBN binary format is defined in Rust with `#[repr(C)]` structs:

```rust
#[repr(C)]
pub struct MboMsg {
    pub hd: RecordHeader,
    pub order_id: u64,
    pub price: i64,
    // ...
}
```

**Key characteristics:**
- 8-byte alignment
- Fixed-width structs
- Little-endian byte order
- No padding between fields

### Protocol Buffers

Protobuf schemas provide language-agnostic definitions:

```protobuf
message MboMsg {
  RecordHeader hd = 1;
  uint64 order_id = 2;
  int64 price = 3;
  // ...
}
```

**Use cases:**
- Code generation for multiple languages
- Schema documentation
- Type-safe APIs
- JSON schema generation

### JSON Schema

JSON schemas enable validation of JSON-formatted messages:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "order_id": { "type": "string" }
  }
}
```

**Use cases:**
- Runtime validation
- API documentation
- IDE autocomplete
- Form generation

## Buf CLI Workflow

### Installation

```bash
# Install buf CLI
brew install bufbuild/buf/buf

# Or download from https://github.com/bufbuild/buf/releases
```

### Linting

Check schema files for style and best practices:

```bash
buf lint
```

Configuration in `buf.yaml`:

```yaml
lint:
  use:
    - BASIC
  except:
    - PACKAGE_DIRECTORY_MATCH
    - COMMENT_FIELD
```

### Formatting

Auto-format protobuf files:

```bash
buf format -w
```

### Breaking Change Detection

Check for breaking changes against a baseline:

```bash
# Against main branch
buf breaking --against '.git#branch=main'

# Against a tag
buf breaking --against '.git#tag=v0.42.0'
```

### Code Generation

Generate code from protobuf schemas:

```bash
buf generate
```

This will generate:
- **Go code** → `gen/go/`
- **Python code** → `gen/python/`
- **TypeScript** → `gen/ts/`
- **JSON schemas** → `gen/jsonschema/`

Configuration in `buf.gen.yaml`:

```yaml
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
  - remote: buf.build/protocolbuffers/python
    out: gen/python
```

## Schema Versioning

### Version History

DBN uses semantic versioning for its schema:

- **v3** (current) - Multi-leg strategies, 64-bit StatMsg quantity
- **v2** - Expanded symbols (71 bytes), updated InstrumentDefMsg
- **v1** - Initial release

### Compatibility Rules

1. **Never break existing parsers**
   - Keep existing field numbers
   - Don't change field types
   - Don't remove required fields

2. **Additive changes only**
   - Add new optional fields
   - Add new message types
   - Add new enum values (append only)

3. **Version bumps**
   - **Major version** - Breaking changes to binary format
   - **Minor version** - New messages or optional fields
   - **Patch version** - Documentation or non-schema changes

### Migration Path

When schema changes are needed:

1. Create new versioned namespace (e.g., `databento.dbn.v4`)
2. Implement conversion functions in Rust
3. Update protobuf schemas
4. Regenerate all derived schemas
5. Document changes in CHANGELOG.md

## Working with Schemas

### Adding a New Message Type

1. **Define in Rust** (`rust/dbn/src/record.rs`):

```rust
#[repr(C)]
#[derive(Clone, CsvSerialize, JsonSerialize)]
pub struct MyNewMsg {
    pub hd: RecordHeader,
    pub my_field: u64,
}
```

2. **Add to protobuf** (`proto/databento/dbn/v3/...`):

```protobuf
message MyNewMsg {
  RecordHeader hd = 1;
  uint64 my_field = 2;
}
```

3. **Update RType enum**:

```protobuf
enum RType {
  // ...
  RTYPE_MY_NEW = 250;  // Choose unused value
}
```

4. **Lint and format**:

```bash
buf lint
buf format -w
```

5. **Generate schemas**:

```bash
buf generate
```

6. **Test**:
   - Unit tests in Rust
   - Integration tests with protobuf
   - Validation tests with JSON schema

### Modifying an Existing Message

1. **Check if change is breaking**:
   - Changing field types → **BREAKING**
   - Removing fields → **BREAKING**
   - Adding optional fields → **SAFE**

2. **Update all schema formats**:
   - Rust struct
   - Protobuf message
   - JSON schema (if manual)

3. **Run breaking change detection**:

```bash
buf breaking --against '.git#branch=main'
```

4. **Update documentation**:
   - Update field comments
   - Update README files
   - Update ../01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md

### Validating Consistency

Ensure schemas stay in sync:

```bash
# Rust build checks
cd rust/dbn
cargo test

# Protobuf lint
buf lint

# Generated code builds
buf generate
cd gen/python && python -m pytest
cd gen/go && go test ./...
```

## Best Practices

### Schema Design

1. **Use fixed-size integers** for known ranges
   - `uint32` for IDs, not `uint64` unless needed
   - `int64` for prices (fixed-point with 1e-9 scale)

2. **Group related fields**
   - Keep common header (`RecordHeader`) first
   - Group timestamps together
   - Group price fields together

3. **Document everything**
   - Field-level comments
   - Enum value meanings
   - Unit specifications (e.g., "UNIX nanoseconds")

4. **Preserve alignment**
   - 8-byte alignment for all structs
   - Use reserved fields if needed for padding

### Code Generation

1. **Version control generated code** (optional)
   - Pros: Easier to review changes
   - Cons: Larger repository

2. **Use consistent naming**
   - Snake_case for Rust
   - camelCase for TypeScript
   - PascalCase for Go/C#

3. **Test generated code**
   - Round-trip serialization
   - Cross-language compatibility
   - Edge cases (max values, nulls, etc.)

### Documentation

1. **Keep specs up-to-date**
   - ../01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md
   - README files in each directory
   - Inline protobuf comments

2. **Provide examples**
   - Sample messages in `docs/specs/03_generated_artifacts/examples/`
   - Usage examples in README
   - Test fixtures

## Troubleshooting

### Common Issues

**"package directory mismatch"**
```bash
# Fix: Disable the check in buf.yaml
lint:
  except:
    - PACKAGE_DIRECTORY_MATCH
```

**"enum value alias not allowed"**
```bash
# Fix: Remove conflicting enum values
# Protobuf doesn't allow duplicate values without allow_alias
```

**"field not documented"**
```bash
# Fix: Add comments or disable the check
lint:
  except:
    - COMMENT_FIELD
```

### Debugging

```bash
# Verbose lint output
buf lint --error-format=json

# Check specific file
buf lint proto

# Diff against baseline
buf breaking --against '.git#branch=main' --error-format=json
```

## References

- [Buf CLI Documentation](https://buf.build/docs)
- [Protocol Buffers Guide](https://protobuf.dev/)
- [JSON Schema Specification](https://json-schema.org/)
- [DBN Binary Format](../01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md)
- [Databento Docs](https://databento.com/docs)

## Contributing

When contributing schema changes:

1. Fork and create a feature branch
2. Make changes to all relevant schema formats
3. Run `buf lint` and `buf format -w`
4. Run `buf breaking` to check compatibility
5. Update tests and documentation
6. Submit PR with clear description of changes

For questions, open an issue on GitHub or contact the Databento team.
