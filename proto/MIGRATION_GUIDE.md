# Migration Guide: Monolithic to Hierarchical Schema Structure

This guide helps you migrate from the old monolithic `dbn.proto` file to the new hierarchical structure.

## Overview

The DBN protobuf schemas have been reorganized from a single file into a modular, hierarchical structure for better maintainability and clarity.

### Before (Monolithic)

```
docs/specs/proto/
└── dbn.proto  (511 lines, all schemas)
```

### After (Hierarchical)

```
docs/specs/proto/databento/dbn/v3/
├── common/           # 2 files - shared types
├── enums/            # 5 files - enumerations
├── messages/
│   ├── market_data/  # 3 files - real-time data
│   ├── aggregated/   # 3 files - time-series
│   ├── reference/    # 4 files - metadata
│   └── system/       # 3 files - control
└── metadata/         # 1 file - file metadata

Total: 21 modular proto files
```

## What Changed

### File Organization

| Old Location | New Location | Contents |
|--------------|--------------|----------|
| `dbn.proto` (lines 10-21) | `common/header.proto` | RecordHeader |
| `dbn.proto` (lines 61-72) | `common/price_levels.proto` | BidAskPair |
| `dbn.proto` (lines 288-297) | `common/price_levels.proto` | ConsolidatedBidAskPair |
| `dbn.proto` (lines 411-436) | `enums/rtype.proto` | RType enum |
| `dbn.proto` (lines 438-460) | `enums/schema.proto` | Schema enum |
| `dbn.proto` (lines 462-468) | `enums/market.proto` | Side enum |
| `dbn.proto` (lines 470-480) | `enums/market.proto` | Action enum |
| `dbn.proto` (lines 482-496) | `enums/symbology.proto` | SType enum |
| `dbn.proto` (lines 498-511) | `enums/instrument.proto` | InstrumentClass enum |
| `dbn.proto` (lines 23-41) | `messages/market_data/mbo.proto` | MboMsg |
| `dbn.proto` (lines 43-59) | `messages/market_data/trade.proto` | TradeMsg |
| `dbn.proto` (lines 74-92) | `messages/market_data/mbp.proto` | Mbp1Msg |
| `dbn.proto` (lines 94-111) | `messages/market_data/mbp.proto` | Mbp10Msg |
| `dbn.proto` (lines 113-124) | `messages/aggregated/ohlcv.proto` | OhlcvMsg |
| `dbn.proto` (lines 299-308) | `messages/aggregated/bbo.proto` | BboMsg |
| `dbn.proto` (lines 310-319) | `messages/aggregated/cbbo.proto` | CbboMsg |
| `dbn.proto` (lines 321-332) | `messages/aggregated/cbbo.proto` | Cmbp1Msg |
| `dbn.proto` (lines 141-234) | `messages/reference/instrument_def.proto` | InstrumentDefMsg |
| `dbn.proto` (lines 126-138) | `messages/reference/status.proto` | StatusMsg |
| `dbn.proto` (lines 236-252) | `messages/reference/statistics.proto` | StatMsg |
| `dbn.proto` (lines 254-286) | `messages/reference/imbalance.proto` | ImbalanceMsg |
| `dbn.proto` (lines 334-340) | `messages/system/error.proto` | ErrorMsg |
| `dbn.proto` (lines 342-348) | `messages/system/system.proto` | SystemMsg |
| `dbn.proto` (lines 350-361) | `messages/system/symbol_mapping.proto` | SymbolMappingMsg |
| `dbn.proto` (lines 363-377) | `metadata/metadata.proto` | Metadata |
| `dbn.proto` (lines 379-382) | `metadata/metadata.proto` | SymbolMapping |
| `dbn.proto` (lines 384-388) | `metadata/metadata.proto` | MappingInterval |

### Package Names

**Old:**
```protobuf
package databento.dbn.v3;
```

**New (varies by category):**
```protobuf
// Common types
package databento.dbn.v3.common;

// Enums
package databento.dbn.v3.enums;

// Market data messages
package databento.dbn.v3.messages.market_data;

// Aggregated messages
package databento.dbn.v3.messages.aggregated;

// Reference messages
package databento.dbn.v3.messages.reference;

// System messages
package databento.dbn.v3.messages.system;

// Metadata
package databento.dbn.v3.metadata;
```

### Import Statements

**Old (no imports needed):**
```protobuf
syntax = "proto3";
package databento.dbn.v3;

message MboMsg {
  RecordHeader hd = 1;
  // ...
}
```

**New (with imports):**
```protobuf
syntax = "proto3";
package databento.dbn.v3.messages.market_data;

import "databento/dbn/v3/common/header.proto";

message MboMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  // ...
}
```

## Migration Steps

### 1. Update Import Paths

If you were importing the monolithic file:

**Before:**
```protobuf
import "dbn.proto";
```

**After:**
```protobuf
// Import only what you need
import "databento/dbn/v3/common/header.proto";
import "databento/dbn/v3/messages/market_data/trade.proto";
```

### 2. Update Fully Qualified Names

**Before:**
```protobuf
databento.dbn.v3.MboMsg
databento.dbn.v3.RecordHeader
databento.dbn.v3.Side
```

**After:**
```protobuf
databento.dbn.v3.messages.market_data.MboMsg
databento.dbn.v3.common.RecordHeader
databento.dbn.v3.enums.Side
```

### 3. Update Code Generation

**Before:**
```bash
protoc --go_out=. dbn.proto
```

**After (using buf):**
```bash
buf generate
```

**After (using protoc):**
```bash
# Generate all at once
protoc --go_out=. \
  databento/dbn/v3/**/*.proto

# Or generate selectively
protoc --go_out=. \
  databento/dbn/v3/common/*.proto \
  databento/dbn/v3/messages/market_data/*.proto
```

### 4. Update Go Import Paths

**Before:**
```go
import dbn "github.com/databento/dbn/gen/go/databento/dbn/v3"

msg := &dbn.MboMsg{}
hd := &dbn.RecordHeader{}
```

**After:**
```go
import (
    common "github.com/databento/dbn/gen/go/databento/dbn/v3/common"
    marketdata "github.com/databento/dbn/gen/go/databento/dbn/v3/messages/market_data"
    enums "github.com/databento/dbn/gen/go/databento/dbn/v3/enums"
)

msg := &marketdata.MboMsg{}
hd := &common.RecordHeader{}
side := enums.Side_SIDE_BID
```

### 5. Update Python Import Paths

**Before:**
```python
from databento.dbn.v3 import dbn_pb2

msg = dbn_pb2.MboMsg()
```

**After:**
```python
from databento.dbn.v3.common import header_pb2
from databento.dbn.v3.messages.market_data import mbo_pb2
from databento.dbn.v3.enums import market_pb2

msg = mbo_pb2.MboMsg()
hdr = header_pb2.RecordHeader()
side = market_pb2.SIDE_BID
```

### 6. Update TypeScript Import Paths

**Before:**
```typescript
import { MboMsg, RecordHeader } from './gen/databento/dbn/v3/dbn';
```

**After:**
```typescript
import { RecordHeader } from './gen/databento/dbn/v3/common/header';
import { MboMsg } from './gen/databento/dbn/v3/messages/market_data/mbo';
import { Side } from './gen/databento/dbn/v3/enums/market';
```

## Quick Reference: Type Locations

### Common Types
| Type | Package | File |
|------|---------|------|
| RecordHeader | `databento.dbn.v3.common` | `common/header.proto` |
| BidAskPair | `databento.dbn.v3.common` | `common/price_levels.proto` |
| ConsolidatedBidAskPair | `databento.dbn.v3.common` | `common/price_levels.proto` |

### Enumerations
| Type | Package | File |
|------|---------|------|
| RType | `databento.dbn.v3.enums` | `enums/rtype.proto` |
| Schema | `databento.dbn.v3.enums` | `enums/schema.proto` |
| Side | `databento.dbn.v3.enums` | `enums/market.proto` |
| Action | `databento.dbn.v3.enums` | `enums/market.proto` |
| SType | `databento.dbn.v3.enums` | `enums/symbology.proto` |
| InstrumentClass | `databento.dbn.v3.enums` | `enums/instrument.proto` |

### Market Data Messages
| Type | Package | File |
|------|---------|------|
| MboMsg | `databento.dbn.v3.messages.market_data` | `messages/market_data/mbo.proto` |
| TradeMsg | `databento.dbn.v3.messages.market_data` | `messages/market_data/trade.proto` |
| Mbp1Msg | `databento.dbn.v3.messages.market_data` | `messages/market_data/mbp.proto` |
| Mbp10Msg | `databento.dbn.v3.messages.market_data` | `messages/market_data/mbp.proto` |

### Aggregated Messages
| Type | Package | File |
|------|---------|------|
| OhlcvMsg | `databento.dbn.v3.messages.aggregated` | `messages/aggregated/ohlcv.proto` |
| BboMsg | `databento.dbn.v3.messages.aggregated` | `messages/aggregated/bbo.proto` |
| CbboMsg | `databento.dbn.v3.messages.aggregated` | `messages/aggregated/cbbo.proto` |
| Cmbp1Msg | `databento.dbn.v3.messages.aggregated` | `messages/aggregated/cbbo.proto` |

### Reference Messages
| Type | Package | File |
|------|---------|------|
| InstrumentDefMsg | `databento.dbn.v3.messages.reference` | `messages/reference/instrument_def.proto` |
| StatusMsg | `databento.dbn.v3.messages.reference` | `messages/reference/status.proto` |
| StatMsg | `databento.dbn.v3.messages.reference` | `messages/reference/statistics.proto` |
| ImbalanceMsg | `databento.dbn.v3.messages.reference` | `messages/reference/imbalance.proto` |

### System Messages
| Type | Package | File |
|------|---------|------|
| ErrorMsg | `databento.dbn.v3.messages.system` | `messages/system/error.proto` |
| SystemMsg | `databento.dbn.v3.messages.system` | `messages/system/system.proto` |
| SymbolMappingMsg | `databento.dbn.v3.messages.system` | `messages/system/symbol_mapping.proto` |

### Metadata
| Type | Package | File |
|------|---------|------|
| Metadata | `databento.dbn.v3.metadata` | `metadata/metadata.proto` |
| SymbolMapping | `databento.dbn.v3.metadata` | `metadata/metadata.proto` |
| MappingInterval | `databento.dbn.v3.metadata` | `metadata/metadata.proto` |

## Benefits of New Structure

### 1. Better Organization
- Clear separation of concerns
- Easier to navigate and understand
- Logical grouping by use case

### 2. Selective Imports
- Import only what you need
- Smaller generated code footprint
- Faster compilation

### 3. Maintainability
- Changes isolated to specific files
- Easier code reviews
- Simpler testing

### 4. Scalability
- Easy to add new message types
- Clear location for new schemas
- No merge conflicts in monolithic file

### 5. Documentation
- README per directory
- Self-documenting structure
- Clear ownership boundaries

## Backward Compatibility

The new structure maintains:
- ✅ Same field numbers
- ✅ Same field types
- ✅ Same message structure
- ✅ Same binary encoding
- ✅ Same package prefix (`databento.dbn.v3`)

What changed:
- ❌ Import paths
- ❌ Package names (added subcategories)
- ❌ Fully qualified type names

## Troubleshooting

### Issue: Import not found

**Error:**
```
import "databento/dbn/v3/common/header.proto": file does not exist
```

**Solution:**
Ensure your protoc or buf config points to `docs/specs/proto` as the import root:

```bash
# protoc
protoc -I docs/specs/proto ...

# buf.yaml
modules:
  - path: docs/specs/proto
```

### Issue: Type not found in code

**Error (Go):**
```
undefined: dbn.MboMsg
```

**Solution:**
Update imports to use new package structure:

```go
import marketdata "github.com/databento/dbn/gen/go/databento/dbn/v3/messages/market_data"

msg := &marketdata.MboMsg{}
```

### Issue: Breaking changes detected

**Error:**
```
buf breaking: Previously present field "1" with name "hd" on message "MboMsg" changed type
```

**Solution:**
This shouldn't happen if migrating correctly. Verify:
1. Field numbers unchanged
2. Field types unchanged
3. Message names unchanged

## Support

For questions or issues:
- Check the [Schema Management Guide](../../SCHEMA_MANAGEMENT.md)
- Review the [v3 README](databento/dbn/v3/README.md)
- Open an issue on GitHub

---

**Migration Date:** 2025-10-18
**Affected Version:** DBN v3
**Backward Compatible:** Binary format yes, API no
