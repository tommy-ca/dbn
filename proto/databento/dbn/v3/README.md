# DBN v3 Protocol Buffer Schemas

This directory contains the hierarchical Protocol Buffer schema definitions for Databento Binary Encoding (DBN) version 3.

## Directory Structure

The schemas are organized into a clear hierarchical structure:

```
databento/dbn/v3/
├── common/                 # Common types shared across messages
│   ├── header.proto       # RecordHeader (16 bytes)
│   └── price_levels.proto # BidAskPair, ConsolidatedBidAskPair
│
├── enums/                  # Enumeration definitions
│   ├── rtype.proto        # Record type identifiers (RType)
│   ├── schema.proto       # Data schema types (Schema)
│   ├── market.proto       # Side, Action enums
│   ├── symbology.proto    # SType symbology types
│   └── instrument.proto   # InstrumentClass enum
│
├── messages/               # Message type definitions
│   ├── market_data/       # Real-time market data
│   │   ├── mbo.proto      # MboMsg (80 bytes)
│   │   ├── trade.proto    # TradeMsg (80 bytes)
│   │   └── mbp.proto      # Mbp1Msg (104 bytes), Mbp10Msg (424 bytes)
│   │
│   ├── aggregated/        # Time-aggregated data
│   │   ├── ohlcv.proto    # OhlcvMsg (56 bytes)
│   │   ├── bbo.proto      # BboMsg (72 bytes)
│   │   └── cbbo.proto     # CbboMsg, Cmbp1Msg (72 bytes)
│   │
│   ├── reference/         # Reference and metadata
│   │   ├── instrument_def.proto  # InstrumentDefMsg (520 bytes)
│   │   ├── status.proto          # StatusMsg (48 bytes)
│   │   ├── statistics.proto      # StatMsg (80 bytes)
│   │   └── imbalance.proto       # ImbalanceMsg (120 bytes)
│   │
│   └── system/            # System and control messages
│       ├── error.proto           # ErrorMsg (312 bytes)
│       ├── system.proto          # SystemMsg (312 bytes)
│       └── symbol_mapping.proto  # SymbolMappingMsg (160 bytes)
│
└── metadata/               # Metadata structures
    └── metadata.proto     # Metadata, SymbolMapping, MappingInterval
```

## Schema Categories

### Common Types (`common/`)

Shared foundational types used across all message schemas.

**header.proto**
- `RecordHeader` - 16-byte common header for all DBN records

**price_levels.proto**
- `BidAskPair` - Single price level (32 bytes)
- `ConsolidatedBidAskPair` - Multi-venue price level (40 bytes)

### Enumerations (`enums/`)

All enumeration types organized by domain.

**rtype.proto** - Record Type Identifiers
- `RType` - 22 record type values (0x00 to 0xC4)

**schema.proto** - Data Schema Types
- `Schema` - 20 schema type values for different data streams

**market.proto** - Market-Related Enums
- `Side` - Ask, Bid, None
- `Action` - Add, Cancel, Modify, Trade, Fill, Clear, None

**symbology.proto** - Symbol Types
- `SType` - 13 symbology type values (InstrumentId, RawSymbol, ISIN, etc.)

**instrument.proto** - Instrument Classification
- `InstrumentClass` - Bond, Call, Future, Stock, Put, Spreads, Spots, etc.

### Market Data Messages (`messages/market_data/`)

Real-time order book and trade data.

**mbo.proto** - Market-by-Order
- `MboMsg` (80 bytes) - Individual order events with order IDs

**trade.proto** - Trade Executions
- `TradeMsg` (80 bytes) - Trade execution messages

**mbp.proto** - Market-by-Price
- `Mbp1Msg` (104 bytes) - Top of book aggregated by price
- `Mbp10Msg` (424 bytes) - Top 10 price levels

### Aggregated Data Messages (`messages/aggregated/`)

Time-aggregated and consolidated data.

**ohlcv.proto** - Candlestick Data
- `OhlcvMsg` (56 bytes) - OHLCV bars at various intervals (1S, 1M, 1H, 1D, EOD)

**bbo.proto** - Best Bid/Offer
- `BboMsg` (72 bytes) - Best bid/offer snapshots (1S, 1M intervals)

**cbbo.proto** - Consolidated Best Bid/Offer
- `CbboMsg` (72 bytes) - Multi-venue consolidated BBO (1S, 1M intervals)
- `Cmbp1Msg` (72 bytes) - Consolidated MBP depth 1, Trade with CBBO

### Reference Data Messages (`messages/reference/`)

Instrument definitions and market metadata.

**instrument_def.proto** - Instrument Definitions
- `InstrumentDefMsg` (520 bytes) - Complete instrument metadata including:
  - Price parameters (min tick, limits, ratios)
  - Contract specifications (multiplier, lot sizes)
  - Multi-leg strategy fields
  - Symbology and classification

**status.proto** - Trading Status
- `StatusMsg` (48 bytes) - Trading status changes (halts, pauses, etc.)

**statistics.proto** - Exchange Statistics
- `StatMsg` (80 bytes) - Exchange-published statistics (settlement, VWAP, etc.)

**imbalance.proto** - Order Imbalance
- `ImbalanceMsg` (120 bytes) - Auction imbalance information

### System Messages (`messages/system/`)

Gateway control and administrative messages.

**error.proto** - Error Messages
- `ErrorMsg` (312 bytes) - Error messages from gateway

**system.proto** - System Messages
- `SystemMsg` (312 bytes) - System messages and heartbeats

**symbol_mapping.proto** - Symbol Mappings
- `SymbolMappingMsg` (160 bytes) - Symbol resolution mappings

### Metadata (`metadata/`)

File and stream metadata structures.

**metadata.proto**
- `Metadata` - DBN file/stream metadata header
- `SymbolMapping` - Symbol resolution with time intervals
- `MappingInterval` - Time-bound symbol mapping

## Package Naming Convention

All schemas follow a consistent package naming pattern:

```
databento.dbn.v3.<category>
```

Examples:
- `databento.dbn.v3.common`
- `databento.dbn.v3.enums`
- `databento.dbn.v3.messages.market_data`
- `databento.dbn.v3.messages.aggregated`
- `databento.dbn.v3.messages.reference`
- `databento.dbn.v3.messages.system`
- `databento.dbn.v3.metadata`

## Import Paths

When importing schemas, use the full path relative to the proto root:

```protobuf
import "databento/dbn/v3/common/header.proto";
import "databento/dbn/v3/common/price_levels.proto";
import "databento/dbn/v3/enums/rtype.proto";
```

## Usage Examples

### Generating Code

```bash
# Generate Go/Python/TS/JSON Schema artifacts
buf generate --template buf.gen.yaml
```

### Importing in Your Schemas

```protobuf
syntax = "proto3";

package myapp.market;

import "databento/dbn/v3/common/header.proto";
import "databento/dbn/v3/messages/market_data/trade.proto";

message MyMarketData {
  databento.dbn.v3.messages.market_data.TradeMsg trade = 1;
}
```

### Using in Code (Go Example)

```go
import (
    common "github.com/databento/dbn/gen/go/databento/dbn/v3/common"
    marketdata "github.com/databento/dbn/gen/go/databento/dbn/v3/messages/market_data"
)

func processTrade(msg *marketdata.TradeMsg) {
    fmt.Printf("Trade: %d @ %d\n", msg.Size, msg.Price)
    fmt.Printf("Instrument: %d\n", msg.Hd.InstrumentId)
}
```

## Design Principles

### 1. Separation of Concerns

Each category has a clear responsibility:
- **common/** - Reusable building blocks
- **enums/** - Type-safe constants
- **messages/** - Business logic messages
- **metadata/** - Administrative data

### 2. Logical Grouping

Messages are grouped by use case:
- **market_data/** - Real-time trading
- **aggregated/** - Time-series analysis
- **reference/** - Static/semi-static data
- **system/** - Infrastructure

### 3. Minimize Dependencies

- Common types have no dependencies
- Enums are standalone
- Messages only import what they need
- No circular dependencies

### 4. Binary Compatibility

All schemas maintain compatibility with the DBN binary format:
- Field numbers match struct layout
- Sizes documented in comments
- Fixed-point encoding noted (1e-9 scale)
- Timestamp format documented (UNIX nanos)

## Schema Evolution

### Adding New Messages

1. Choose the appropriate category directory
2. Create a new `.proto` file
3. Define the message with proper imports
4. Update this README with the new message
5. Run `buf lint` and `buf format -w`

### Modifying Existing Messages

1. Never change existing field numbers
2. Never change field types
3. Add new fields with new numbers only
4. Mark deprecated fields with `[deprecated = true]`
5. Run `buf breaking` to check compatibility

### Version Management

- Major version changes go in new directory (e.g., `v4/`)
- Minor changes are backward-compatible additions
- All schemas in `v3/` share the same major version

## Validation

```bash
# Lint all schemas
buf lint

# Format all schemas
buf format -w

# Check for breaking changes
buf breaking --against '.git#branch=main'

# Build all schemas
buf build
```

## References

- [DBN Binary Specification](../../../docs/specs/DBN_SCHEMA_SPECIFICATION.md)
- [Schema Management Guide](../../../docs/specs/SCHEMA_MANAGEMENT.md)
- [Buf Documentation](https://buf.build/docs)
- [Protocol Buffers Guide](https://protobuf.dev/)

## Contributing

When adding or modifying schemas:

1. Follow the existing directory structure
2. Use consistent naming conventions
3. Document all fields with comments
4. Include size annotations for binary compatibility
5. Update this README
6. Run validation tools before committing

---

**Version:** DBN v3
**Last Updated:** 2025-10-18
**Total Schemas:** 21 files across 6 categories
