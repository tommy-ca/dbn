# DBN Schema Specifications

This directory contains comprehensive schema specifications and documentation for the Databento Binary Encoding (DBN) format.

## 🚀 Quick Start

**New to DBN?** Follow this path:
1. Start with [DBN_SCHEMA_SPECIFICATION.md](01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md) - Understand the base binary format
2. For crypto data, read [CRYPTO_DBN_QUICK_REFERENCE.md](00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md) - Get an overview
3. Work with schemas: See [SCHEMA_MANAGEMENT.md](02_schema_implementation/SCHEMA_MANAGEMENT.md) for setup and workflows

**Setting up development?** Jump to [SCHEMA_MANAGEMENT.md](02_schema_implementation/SCHEMA_MANAGEMENT.md#prerequisites) for prerequisites and installation.

**Looking for examples?** See [examples/README.md](03_generated_artifacts/examples/README.md) for sample data and code.

---

## Specs-Driven Development Flow

1. **Capture requirements** – Document regulatory, market, and latency needs in [CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md](00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md) and summarize the business brief in [CRYPTO_DBN_QUICK_REFERENCE.md](00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md).
2. **Author the canonical DBN specification** – Maintain the binary contract in [DBN_SCHEMA_SPECIFICATION.md](01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md).
3. **Implement portable schemas** – Encode the canonical spec in protobuf (`proto/`) using the workflow in [SCHEMA_MANAGEMENT.md](02_schema_implementation/SCHEMA_MANAGEMENT.md).
4. **Generate derived artifacts** – Produce JSON Schema, SDKs, and docs with `buf generate` (outputs in `gen/` and `03_generated_artifacts/generated/`).
5. **Validate and distribute** – Use the examples suite plus downstream tests to ensure parity before publishing clients and documentation.

This flow keeps every artifact traceable to the source requirements while making it clear where changes should originate.

## Contents by Phase

### Phase 0 – Product Requirements & Strategy (`00_requirements/`)

- **[CRYPTO_DBN_QUICK_REFERENCE.md](00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md)** – Executive summary for stakeholders: high-level record taxonomy, storage footprint expectations, and cross-market constraints.
- **[CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md](00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md)** – Detailed requirements, gap analyses, and acceptance criteria for new crypto datasets.

### Phase 1 – Canonical Specification (`01_canonical_spec/`)

- **[DBN_SCHEMA_SPECIFICATION.md](01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md)** – Binary layout, enums, and version history. Every downstream artifact references the identifiers and constraints defined here.

### Phase 2 – Schema Implementation & Tooling (`02_schema_implementation/`)

- **[SCHEMA_MANAGEMENT.md](02_schema_implementation/SCHEMA_MANAGEMENT.md)** – Environment setup, Buf workflows, and contributor expectations for evolving the schemas.
- **[IMPLEMENTATION_SUMMARY.md](02_schema_implementation/IMPLEMENTATION_SUMMARY.md)** – Traceability matrix describing how the spec-driven flow is implemented in this repository.
- **[HIERARCHICAL_SCHEMA_SUMMARY.md](02_schema_implementation/HIERARCHICAL_SCHEMA_SUMMARY.md)** – Documentation of the multi-file protobuf layout and rationale for each module.
- **Protocol Buffers (`proto/`)** – Language-agnostic schema definitions representing the canonical spec in `.proto` form:

  ```
  proto/
  └── databento/dbn/v3/
      ├── common/           # RecordHeader, BidAskPair, etc.
      ├── enums/            # RType, Schema, Side, Action ...
      ├── messages/
      │   ├── market_data/
      │   ├── aggregated/
      │   ├── reference/
      │   └── system/
      └── metadata/
  ```

  **Features:**
  - Modular files for the 15+ message types and enums
  - Comprehensive enum coverage (RType, Schema, Side, Action, etc.)
  - Fully managed with Buf (lint, breaking-change detection, formatting)

  **Usage:**
  ```bash
  buf lint && buf format -w
  buf breaking --against '.git#branch=main'
  buf generate
  ```

### Phase 3 – Generated Schemas & SDKs (`03_generated_artifacts/`)

- **JSON Schemas (`gen/jsonschema/`)** – Validation contracts derived from the protobuf definitions.

  ```
  gen/
  └── jsonschema/
      ├── *.schema.json         # Individual message schemas
      └── MboMsg.schema.json   # Example: MBO message schema
  ```

  **Features:** Draft 2020-12 compliance, snake_case + camelCase variants, and large integer safety. Use standard validators such as Ajv or `jsonschema`:

  ```javascript
  const Ajv = require('ajv');
  const schema = require('./MboMsg.schema.json');
  const validate = new Ajv().compile(schema);
  validate(data);
  ```

  ```python
  import jsonschema
  jsonschema.validate(instance=data, schema=schema)
  ```

- **Go/Python/TypeScript SDKs (`gen/go`, `gen/python`, `gen/ts`)** – Generated clients refreshed via `buf generate`.
- **HTML and narrative docs (`03_generated_artifacts/generated/`)** – Optional Buf doc outputs for publishing schema references.
- **Examples (`03_generated_artifacts/examples/`)** – Sample payloads, code walkthroughs, and future conformance fixtures.

  ```
  03_generated_artifacts/examples/
  ├── mbo_sample.json       # Example MBO message
  ├── trade_sample.json     # Example trade message
  └── metadata_sample.json  # Example metadata
  ```

### Phase 4 – Validation & Release Readiness (`04_validation/`)

- **[CLEAN_ENGINEERING_UPDATE.md](04_validation/CLEAN_ENGINEERING_UPDATE.md)** – Narrative QA + release notes tying protobuf updates back to binary guarantees.
- **Checklists & future audits** – Add structured validation docs here as new release gates emerge.

## Message Types

DBN defines 15 core message types across several categories:

### Market Data Messages

| Message | Size | RType | Description |
|---------|------|-------|-------------|
| MboMsg | 80 bytes | 0xA0 | Market-by-order tick data |
| TradeMsg | 80 bytes | 0x00 | Trade execution messages |
| Mbp1Msg | 104 bytes | 0x01 | Market-by-price depth 1 |
| Mbp10Msg | 424 bytes | 0x0A | Market-by-price depth 10 |
| BboMsg | 72 bytes | 0xC3/0xC4 | Best Bid/Offer (1S/1M) |
| Cmbp1Msg | 72 bytes | 0xB1 | Consolidated MBP depth 1 |
| CbboMsg | 72 bytes | 0xC0/0xC1 | Consolidated BBO (1S/1M) |

### Aggregated Data

| Message | Size | RType | Description |
|---------|------|-------|-------------|
| OhlcvMsg | 56 bytes | 0x20-0x24 | OHLCV candlestick data |

### Instrument & Reference Data

| Message | Size | RType | Description |
|---------|------|-------|-------------|
| InstrumentDefMsg | 520 bytes | 0x13 | Instrument definitions |
| StatMsg | 80 bytes | 0x18 | Exchange statistics |
| StatusMsg | 48 bytes | 0x12 | Trading status updates |
| ImbalanceMsg | 120 bytes | 0x14 | Order imbalance info |

### System Messages

| Message | Size | RType | Description |
|---------|------|-------|-------------|
| ErrorMsg | 312 bytes | 0x15 | Error messages |
| SystemMsg | 312 bytes | 0x17 | System/heartbeat messages |
| SymbolMappingMsg | 160 bytes | 0x16 | Symbol mappings |

## Key Concepts

### Fixed-Point Prices

All price fields use fixed-point encoding with scale factor `1e-9`:

```
Real Price = DBN Price / 1,000,000,000

Example:
  DBN: 1500000000 → Decimal: $1.50
  DBN: 100500000000 → Decimal: $100.50
```

### Timestamps

All timestamps are UNIX nanoseconds since epoch (`uint64`):

```
2024-01-01 00:00:00.123456789 UTC
= 1704067200123456789 nanoseconds
```

### Record Header

Every message starts with a 16-byte `RecordHeader`:

```rust
RecordHeader {
  length: u8,           // Record length in 32-bit words
  rtype: u8,            // Record type identifier
  publisher_id: u16,    // Databento publisher ID
  instrument_id: u32,   // Numeric instrument ID
  ts_event: u64,        // Event timestamp (UNIX nanos)
}
```

### Alignment

All records are 8-byte aligned with sizes divisible by 8.

## Schema Formats Comparison

| Format | Purpose | Source | Generated |
|--------|---------|--------|-----------|
| Rust structs | Binary encoding/decoding | ✓ Primary | - |
| Protocol Buffers | Cross-language schemas | ✓ Manual | - |
| JSON Schema | JSON validation | - | ✓ From protobuf |
| Go structs | Go language support | - | ✓ From protobuf |
| Python classes | Python language support | ✓ PyO3 bindings | ✓ From protobuf |
| TypeScript types | TypeScript support | - | ✓ From protobuf |

## Version History

### DBN Version 3 (Current)

**Released:** 2025-05-28

**Changes:**
- Multi-leg strategy fields in InstrumentDefMsg
- Expanded `asset` field (7 → 11 bytes)
- 64-bit `quantity` in StatMsg
- Larger `raw_instrument_id` (32 → 64 bits)

**Compatibility:** v1 and v2 records can be upgraded to v3

### DBN Version 2

**Released:** 2023-11-15

**Changes:**
- Expanded symbols (22 → 71 bytes)
- Added `raw_instrument_id` to InstrumentDefMsg
- Larger ErrorMsg and SystemMsg (80 → 312 bytes)

### DBN Version 1

**Released:** 2023-02-22

**Initial release** with core message types and binary encoding

## Workflow

### Using Schemas

1. **Read the specifications** - Start with DBN_SCHEMA_SPECIFICATION.md
2. **Choose your format** - Protobuf for code gen, JSON Schema for validation
3. **Generate code** - Use `buf generate` for protobuf-based code
4. **Validate data** - Use JSON schemas for runtime validation

### Contributing Schema Changes

1. **Update all formats** - Rust, protobuf, and documentation
2. **Run checks**:
   ```bash
   buf lint
   buf format -w
   buf breaking --against '.git#branch=main'
   cargo test
   ```
3. **Generate artifacts**:
   ```bash
   buf generate
   ```
4. **Update docs** - Specifications and README files
5. **Submit PR** - With clear description of changes

## Tools

### Buf CLI

Protocol buffer management and code generation:

```bash
# Install
brew install bufbuild/buf/buf

# Commands
buf lint                  # Check schemas
buf format -w            # Format schemas
buf generate             # Generate code
buf breaking --against . # Check compatibility
```

### Protobuf Compiler (protoc)

Alternative to Buf CLI for code generation:

```bash
protoc --proto_path=proto \
       --go_out=. \
       --python_out=. \
       --js_out=. \
       proto/databento/dbn/v3/**/*.proto
```

### JSON Schema Validators

- **JavaScript:** [AJV](https://ajv.js.org/)
- **Python:** [jsonschema](https://python-jsonschema.readthedocs.io/)
- **Go:** [gojsonschema](https://github.com/xeipuuv/gojsonschema)

## References

### Official Documentation

- [Databento Docs](https://databento.com/docs/standards-and-conventions/databento-binary-encoding)
- [DBN GitHub Repository](https://github.com/databento/dbn)

### Specifications

- [Protocol Buffers](https://protobuf.dev/)
- [JSON Schema](https://json-schema.org/)
- [Buf Documentation](https://buf.build/docs)

### Implementation

- [Rust Crate](https://crates.io/crates/dbn)
- [Python Package](https://pypi.org/project/databento-dbn)
- [API Documentation](https://docs.rs/dbn/latest/dbn/)

## Support

For questions or issues:

- **GitHub Issues:** https://github.com/databento/dbn/issues
- **Slack Community:** https://to.dbn.to/slack
- **Email:** support@databento.com

## License

DBN is distributed under the Apache 2.0 License.
