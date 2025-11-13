# DBN Schema Specifications

This directory contains comprehensive schema specifications and documentation for the Databento Binary Encoding (DBN) format.

## Contents

### Documentation

- **[DBN_SCHEMA_SPECIFICATION.md](DBN_SCHEMA_SPECIFICATION.md)** - Complete binary format specification
  - Record structure and sizes
  - Field definitions and data types
  - Enumerations and constants
  - Version history and compatibility

- **[SCHEMA_MANAGEMENT.md](SCHEMA_MANAGEMENT.md)** - Schema management guide
  - Working with protobuf and buf CLI
  - Schema versioning strategy
  - Code generation workflow
  - Best practices

- **[CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md](CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md)** - Crypto market data requirements
- **[CRYPTO_DBN_QUICK_REFERENCE.md](CRYPTO_DBN_QUICK_REFERENCE.md)** - Quick reference guide

### Schema Definitions

#### Protocol Buffers (`proto/`)

Language-agnostic schema definitions for all DBN message types.

```
proto/
├── dbn.proto              # Main DBN v3 schema
└── README.md              # Protobuf documentation
```

**Features:**
- Type-safe definitions for 15+ message types
- Comprehensive enum definitions (RType, Schema, Side, Action, etc.)
- Code generation for Go, Python, TypeScript, and more
- Managed with Buf CLI for linting and compatibility

**Usage:**
```bash
# Lint schemas
buf lint

# Format schemas
buf format -w

# Generate code
buf generate

# Check for breaking changes
buf breaking --against '.git#branch=main'
```

#### JSON Schemas (`jsonschema/`)

JSON Schema definitions for runtime validation and documentation.

```
jsonschema/
├── *.schema.json         # Individual message schemas
├── MboMsg.schema.json   # Example: MBO message schema
└── README.md            # JSON Schema documentation
```

**Features:**
- JSON Schema Draft 2020-12 compliant
- Runtime validation for JSON-formatted messages
- IDE autocomplete and documentation
- Large integer handling (uint64/int64 as strings)

**Usage:**
```javascript
// JavaScript validation
const Ajv = require('ajv');
const schema = require('./MboMsg.schema.json');
const validate = ajv.compile(schema);
validate(data);
```

```python
# Python validation
import jsonschema
jsonschema.validate(instance=data, schema=schema)
```

#### Examples (`examples/`)

Sample messages and usage examples.

```
examples/
├── mbo_sample.json       # Example MBO message
├── trade_sample.json     # Example trade message
└── metadata_sample.json  # Example metadata
```

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
protoc --go_out=. \
       --python_out=. \
       --js_out=. \
       docs/specs/proto/dbn.proto
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
