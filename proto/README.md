# DBN Protocol Buffer Schemas

This directory contains Protocol Buffer (protobuf) schema definitions for Databento Binary Encoding (DBN) version 3.

## Overview

The protobuf schemas in this directory provide:

- **Language-agnostic schema definitions** for all DBN message types
- **Type-safe code generation** for multiple programming languages
- **JSON schema generation** for validation and documentation
- **Schema versioning and compatibility checking** via buf CLI

## Structure

Schemas are organized in a **hierarchical structure** for better maintainability:

```
databento/dbn/v3/
├── common/           # Shared types (RecordHeader, BidAskPair)
├── enums/            # Enumerations (RType, Schema, Side, Action, etc.)
├── messages/
│   ├── market_data/  # Real-time data (MBO, Trade, MBP)
│   ├── aggregated/   # Time-series (OHLCV, BBO, CBBO)
│   ├── reference/    # Metadata (InstrumentDef, Status, Stats)
│   └── system/       # Control messages (Error, System)
└── metadata/         # File metadata structures
```

See [databento/dbn/v3/README.md](databento/dbn/v3/README.md) for detailed structure documentation.

## Files

**Total:** 21 proto files organized into 6 categories
- **Common types:** 2 files
- **Enumerations:** 5 files
- **Market data:** 3 files
- **Aggregated data:** 3 files
- **Reference data:** 4 files
- **System messages:** 3 files
- **Metadata:** 1 file

## Message Types

The schema defines the following message types:

### Core Messages
- **RecordHeader** - Common 16-byte header for all records
- **MboMsg** - Market-by-order tick messages (80 bytes)
- **TradeMsg** - Trade execution messages (80 bytes)
- **Mbp1Msg** - Market-by-price depth 1 (104 bytes)
- **Mbp10Msg** - Market-by-price depth 10 (424 bytes)

### Price Levels
- **BidAskPair** - Single price level (32 bytes)
- **ConsolidatedBidAskPair** - Multi-venue price level (40 bytes)

### Aggregated Data
- **OhlcvMsg** - OHLCV candlestick data (56 bytes)
- **BboMsg** - Best Bid/Offer (72 bytes)
- **CbboMsg** - Consolidated Best Bid/Offer (72 bytes)
- **Cmbp1Msg** - Consolidated MBP depth 1 (72 bytes)

### Instrument & Market Data
- **InstrumentDefMsg** - Instrument definitions and metadata (520 bytes)
- **StatMsg** - Exchange statistics (80 bytes)
- **StatusMsg** - Trading status updates (48 bytes)
- **ImbalanceMsg** - Order imbalance information (120 bytes)

### System Messages
- **ErrorMsg** - Error messages from gateway (312 bytes)
- **SystemMsg** - System messages and heartbeats (312 bytes)
- **SymbolMappingMsg** - Symbol mapping information (160 bytes)

### Metadata
- **Metadata** - File/stream metadata structure
- **SymbolMapping** - Symbol resolution mappings
- **MappingInterval** - Time-bound symbol mappings

## Enumerations

The schema includes comprehensive enum definitions:

- **RType** - Record type identifiers (0x00-0xC4)
- **Schema** - Data schema types
- **Side** - Market side (Ask/Bid/None)
- **Action** - Order event actions (Add/Cancel/Modify/Trade/etc.)
- **SType** - Symbology types (InstrumentId/RawSymbol/ISIN/etc.)
- **InstrumentClass** - Instrument classifications

## Key Features

### Fixed-Point Price Encoding
All price fields use fixed-point encoding with scale factor `1e-9`:
- Real price = `proto_price / 1_000_000_000`
- Example: `1500000000` → $1.50

### Timestamp Encoding
All timestamps are UNIX nanoseconds since epoch (`uint64`).

### Binary Compatibility
The protobuf definitions are designed to match the C struct layout used in the Rust implementation:
- All structs use `#[repr(C)]` in Rust
- 8-byte alignment for all records
- Sizes specified in comments match binary format

## Usage with Buf CLI

### Linting
```bash
buf lint
```

### Breaking Change Detection
```bash
buf breaking --against '.git#branch=main'
```

### Generate Code
```bash
buf generate
```

This will generate:
- JSON schemas → `gen/jsonschema/`
- Go code → `gen/go/`
- Python code → `gen/python/`
- TypeScript code → `gen/ts/`
- HTML documentation → `docs/specs/`

### Format
```bash
buf format -w
```

## Differences from Binary Format

While these protobuf schemas closely mirror the binary DBN format, there are some key differences:

1. **Strings vs c_char arrays**: Binary format uses fixed-length null-terminated C strings, protobuf uses variable-length strings
2. **Repeated fields**: Binary format uses fixed-size arrays (e.g., `[BidAskPair; 10]`), protobuf uses `repeated`
3. **Reserved fields**: Binary format includes padding bytes for alignment, protobuf handles this internally
4. **Field numbers**: Protobuf field numbers don't correspond to byte offsets

## Schema Version History

- **Version 3** (2025-05-28): Multi-leg strategies, 64-bit StatMsg quantity
- **Version 2** (2023-11-15): Expanded symbols (71 bytes), updated InstrumentDefMsg
- **Version 1** (2023-02-22): Initial DBN release

## References

- [DBN Specification](../docs/specs/DBN_SCHEMA_SPECIFICATION.md)
- [Databento Documentation](https://databento.com/docs/standards-and-conventions/databento-binary-encoding)
- [Buf Documentation](https://buf.build/docs)
- [Protocol Buffers](https://protobuf.dev/)

## Contributing

When modifying these schemas:

1. Ensure compatibility with the binary format in `rust/dbn/src/record.rs`
2. Run `buf lint` to check for style issues
3. Run `buf breaking` to check for breaking changes
4. Update size comments if message structure changes
5. Regenerate code with `buf generate`
6. Update documentation as needed
