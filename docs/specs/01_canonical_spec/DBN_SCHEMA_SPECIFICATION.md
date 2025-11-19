# Databento Binary Encoding (DBN) Schema Specification

## Table of Contents
1. [Overview](#overview)
2. [Binary Format Structure](#binary-format-structure)
3. [Metadata Header](#metadata-header)
4. [Record Structure](#record-structure)
5. [Schema Definitions](#schema-definitions)
6. [Enumerations](#enumerations)
7. [Constants & Sentinel Values](#constants--sentinel-values)
8. [Version History](#version-history)
9. [Requirements & Specifications](#requirements--specifications)

---

## Overview

**Databento Binary Encoding (DBN)** is an extremely fast message encoding and storage format for normalized market data. It provides:

- **Performant binary encoding/decoding**: Fixed-width records optimized for speed
- **Self-describing format**: Metadata header describes the data that follows
- **Highly compressible**: Works well with Zstandard compression
- **Extendable schemas**: Fixed-width with versioning support
- **Current Version**: DBN Version 3 (DBNv3)

### Key Features
- All records are 8-byte aligned
- Fixed-width structs with no padding
- Prices stored as fixed-point decimals (scale: 1e-9)
- Timestamps stored as nanoseconds since UNIX epoch
- Little-endian byte order

---

## Binary Format Structure

### Complete DBN File/Stream Layout

```
┌─────────────────────────────────────┐
│  Magic String (8 bytes)             │ "DBN\x00" (4 bytes) + reserved (4 bytes)
├─────────────────────────────────────┤
│  Version (1 byte)                   │ DBN version number (current: 3)
├─────────────────────────────────────┤
│  Metadata Length (4 bytes)          │ Total metadata size including header
├─────────────────────────────────────┤
│  Metadata (variable)                │ JSON metadata + padding to 8-byte alignment
├─────────────────────────────────────┤
│  Record 1 (fixed width)             │ RecordHeader + schema-specific data
├─────────────────────────────────────┤
│  Record 2 (fixed width)             │
├─────────────────────────────────────┤
│  ...                                │
├─────────────────────────────────────┤
│  Record N (fixed width)             │
└─────────────────────────────────────┘
```

### Optional: DBN Fragment (Records only, no metadata)
For streaming scenarios, records can be transmitted without the metadata header.

---

## Metadata Header

### Metadata Structure (Version 3)

```rust
pub struct Metadata {
    pub version: u8,                    // DBN schema version (1-3)
    pub dataset: String,                // Dataset code (max 16 chars)
    pub schema: Option<Schema>,         // Data schema or None for mixed
    pub start: u64,                     // UNIX nano timestamp of query start
    pub end: Option<NonZeroU64>,        // UNIX nano timestamp of query end
    pub limit: Option<NonZeroU64>,      // Max number of records
    pub stype_in: Option<SType>,        // Input symbology type
    pub stype_out: SType,               // Output symbology type
    pub ts_out: bool,                   // Has live gateway send timestamps
    pub symbol_cstr_len: usize,         // Fixed symbol string length
    pub symbols: Vec<String>,           // Query input symbols
    pub partial: Vec<String>,           // Partially resolved symbols
    pub not_found: Vec<String>,         // Unresolved symbols
    pub mappings: Vec<SymbolMapping>,   // Symbol mappings with intervals
}
```

### SymbolMapping Structure

```rust
pub struct SymbolMapping {
    pub raw_symbol: String,             // stype_in symbol
    pub intervals: Vec<MappingInterval>,
}

pub struct MappingInterval {
    pub start_date: time::Date,         // UTC start date (inclusive)
    pub end_date: time::Date,           // UTC end date (exclusive)
    pub symbol: String,                 // Resolved symbol (stype_out)
}
```

### Constants
- `METADATA_DATASET_CSTR_LEN`: 16
- `METADATA_RESERVED_LEN`: 53
- `METADATA_FIXED_LEN`: 100 (excludes magic, version, length)

---

## Record Structure

### RecordHeader (Common to all records)

All records start with a common header:

```rust
#[repr(C)]
pub struct RecordHeader {
    pub length: u8,           // Record length in 32-bit words (internal)
    pub rtype: u8,            // Record type identifier (see RType enum)
    pub publisher_id: u16,    // Publisher ID assigned by Databento
    pub instrument_id: u32,   // Numeric instrument ID
    pub ts_event: u64,        // Matching-engine timestamp (UNIX nanos)
}
```

**Size**: 16 bytes
**Alignment**: 8 bytes

### Record Types (RType)

| RType | Value | Description | Schema |
|-------|-------|-------------|--------|
| MBP_0 | 0x00 | Market-by-price depth 0 | Trades |
| MBP_1 | 0x01 | Market-by-price depth 1 | Mbp1, Tbbo |
| MBP_10 | 0x0A | Market-by-price depth 10 | Mbp10 |
| OHLCV_1S | 0x20 | OHLCV 1-second | Ohlcv1S |
| OHLCV_1M | 0x21 | OHLCV 1-minute | Ohlcv1M |
| OHLCV_1H | 0x22 | OHLCV 1-hour | Ohlcv1H |
| OHLCV_1D | 0x23 | OHLCV 1-day UTC | Ohlcv1D |
| OHLCV_EOD | 0x24 | OHLCV end-of-day | OhlcvEod |
| STATUS | 0x12 | Exchange status | Status |
| INSTRUMENT_DEF | 0x13 | Instrument definition | Definition |
| IMBALANCE | 0x14 | Order imbalance | Imbalance |
| ERROR | 0x15 | Error from gateway | - |
| SYMBOL_MAPPING | 0x16 | Symbol mapping | - |
| SYSTEM | 0x17 | System message/heartbeat | - |
| STATISTICS | 0x18 | Statistics | Statistics |
| MBO | 0xA0 | Market-by-order | Mbo |
| CMBP_1 | 0xB1 | Consolidated MBP depth 1 | Cmbp1 |
| CBBO_1S | 0xC0 | Consolidated BBO 1-second | Cbbo1S |
| CBBO_1M | 0xC1 | Consolidated BBO 1-minute | Cbbo1M |
| TCBBO | 0xC2 | Trade with CBBO | Tcbbo |
| BBO_1S | 0xC3 | BBO 1-second | Bbo1S |
| BBO_1M | 0xC4 | BBO 1-minute | Bbo1M |

#### Reserved Crypto Extensions (In Progress)

The following identifiers are reserved for the crypto-focused schemas defined in `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`. They will be promoted to full schema sections once the binary/Rust/protobuf definitions are finalized.

| RType | Value | Planned Record | Purpose |
|-------|-------|----------------|---------|
| FUNDING_RATE | 0xD0 | `FundingRateMsg` | Perpetual funding rates and predictions |
| LIQUIDATION | 0xD1 | `LiquidationMsg` | Forced liquidations / ADL events |
| MARK_PRICE | 0xD2 | `MarkPriceMsg` | Fair price feed for perps |
| INDEX_PRICE | 0xD3 | `IndexPriceMsg` | Multi-venue spot index |
| DEX_SWAP | 0xD4 | `DexSwapMsg` | AMM swap executions |
| DEX_POOL_STATE | 0xD5 | `DexPoolStateMsg` | Liquidity snapshots / concentrates |
| ORACLE_PRICE | 0xD6 | `OraclePriceMsg` | Chainlink/Pyth oracle updates |
| CROSS_RATE | 0xD8 | `CrossRateMsg` | Cross-exchange spreads |
| BASIS | 0xD9 | `BasisMsg` | Spot-vs-perp spreads |
| STABLECOIN_PEG | 0xDA | `StablecoinPegMsg` | Peg deviations |
| GAS_PRICE | 0xDB | `GasPriceMsg` | Network fee telemetry |
| BLOCK_INFO | 0xDC | `BlockInfoMsg` | On-chain block metadata |
| SMART_CONTRACT_EVENT | 0xDD | `SmartContractEventMsg` | Generic contract logs |
| INSURANCE_FUND | 0xDE | `InsuranceFundMsg` | Fund balance tracking |
| CLAWBACK | 0xDF | `ClawbackMsg` | Socialized losses |
| DELISTING | 0xE0 | `DelistingMsg` | Venue delisting notices |
| TOKEN_MIGRATION | 0xE1 | `TokenMigrationMsg` | Contract migrations |

**Range 0xE2‑0xFF** remains reserved for future crypto and on-chain extensions.

---

## Schema Definitions

### 1. MBO (Market-by-Order) - MboMsg

**Size**: 80 bytes
**Alignment**: 8 bytes
**RType**: 0xA0

```rust
#[repr(C)]
pub struct MboMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub order_id: u64,              // Order ID from venue
    pub price: i64,                 // Fixed-point price (scale: 1e-9)
    pub size: u32,                  // Order quantity
    pub flags: FlagSet,             // Event flags (u8)
    pub channel_id: u8,             // Channel ID (0-indexed)
    pub action: c_char,             // Action: A/C/M/R/T/F/N
    pub side: c_char,               // Side: A/B/N
    pub ts_recv: u64,               // Capture server timestamp
    pub ts_in_delta: i32,           // Delta from ts_recv
    pub sequence: u32,              // Message sequence number
}
```

### 2. Trade - TradeMsg

**Size**: 80 bytes
**Alignment**: 8 bytes
**RType**: 0x00 (MBP_0)

```rust
#[repr(C)]
pub struct TradeMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub price: i64,                 // Trade price (fixed-point)
    pub size: u32,                  // Trade quantity
    pub action: c_char,             // Always 'T' for trades
    pub side: c_char,               // Aggressor side: A/B/N
    pub flags: FlagSet,             // Event flags
    pub depth: u8,                  // Book level
    pub ts_recv: u64,               // Capture timestamp
    pub ts_in_delta: i32,           // Delta from ts_recv
    pub sequence: u32,              // Sequence number
}
```

### 3. MBP-1 (Market-by-Price Level 1) - Mbp1Msg

**Size**: 104 bytes
**Alignment**: 8 bytes
**RType**: 0x01 (MBP_1)

```rust
#[repr(C)]
pub struct Mbp1Msg {
    pub hd: RecordHeader,           // 16 bytes
    pub price: i64,                 // Price (fixed-point)
    pub size: u32,                  // Quantity
    pub action: c_char,             // Action: A/C/M/R/T
    pub side: c_char,               // Side: A/B/N
    pub flags: FlagSet,             // Event flags
    pub depth: u8,                  // Book level
    pub ts_recv: u64,               // Capture timestamp
    pub ts_in_delta: i32,           // Delta from ts_recv
    pub sequence: u32,              // Sequence number
    pub levels: [BidAskPair; 1],    // Top of book (32 bytes)
}
```

### 4. MBP-10 (Market-by-Price Level 10) - Mbp10Msg

**Size**: 424 bytes
**Alignment**: 8 bytes
**RType**: 0x0A (MBP_10)

```rust
#[repr(C)]
pub struct Mbp10Msg {
    pub hd: RecordHeader,           // 16 bytes
    pub price: i64,                 // Price (fixed-point)
    pub size: u32,                  // Quantity
    pub action: c_char,             // Action: A/C/M/R/T
    pub side: c_char,               // Side: A/B/N
    pub flags: FlagSet,             // Event flags
    pub depth: u8,                  // Book level
    pub ts_recv: u64,               // Capture timestamp
    pub ts_in_delta: i32,           // Delta from ts_recv
    pub sequence: u32,              // Sequence number
    pub levels: [BidAskPair; 10],   // Top 10 levels (320 bytes)
}
```

### 5. BidAskPair (Level struct for MBP)

**Size**: 32 bytes
**Alignment**: 8 bytes

```rust
#[repr(C)]
pub struct BidAskPair {
    pub bid_px: i64,                // Bid price (fixed-point)
    pub ask_px: i64,                // Ask price (fixed-point)
    pub bid_sz: u32,                // Bid size
    pub ask_sz: u32,                // Ask size
    pub bid_ct: u32,                // Bid order count
    pub ask_ct: u32,                // Ask order count
}
```

### 6. OHLCV - OhlcvMsg

**Size**: 56 bytes
**Alignment**: 8 bytes
**RTypes**: 0x20-0x24 (OHLCV_1S, OHLCV_1M, OHLCV_1H, OHLCV_1D, OHLCV_EOD)

```rust
#[repr(C)]
pub struct OhlcvMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub open: i64,                  // Open price (fixed-point)
    pub high: i64,                  // High price (fixed-point)
    pub low: i64,                   // Low price (fixed-point)
    pub close: i64,                 // Close price (fixed-point)
    pub volume: u64,                // Total volume
}
```

### 7. InstrumentDefMsg (Version 3)

**Size**: 520 bytes
**Alignment**: 8 bytes
**RType**: 0x13 (INSTRUMENT_DEF)

> **Crypto delta roadmap:** DBNv4 work will extend this struct with explicit base/quote currency fields, venue slugs, contract type enumerations (spot, perpetual, dated future, DEX pool), and settlement asset metadata. Those additions are required for parity with cryptofeed/tardis symbol normalization and should be introduced here before updating protobuf/Rust bindings. Until then, continue using the DBNv3 string lengths (`SYMBOL_CSTR_LEN = 71`, `ASSET_CSTR_LEN = 11`) for `raw_symbol`, `leg_raw_symbol`, and `asset`.

```rust
#[repr(C)]
pub struct InstrumentDefMsg {
    pub hd: RecordHeader,                      // 16 bytes
    pub ts_recv: u64,

    // Price-related fields (all fixed-point, scale: 1e-9)
    pub min_price_increment: i64,
    pub display_factor: i64,
    pub high_limit_price: i64,
    pub low_limit_price: i64,
    pub max_price_variation: i64,
    pub unit_of_measure_qty: i64,
    pub min_price_increment_amount: i64,
    pub price_ratio: i64,
    pub strike_price: i64,
    pub leg_price: i64,
    pub leg_delta: i64,

    // Timestamp fields (UNIX nanos)
    pub expiration: u64,
    pub activation: u64,

    // Integer fields
    pub raw_instrument_id: u64,
    pub inst_attrib_value: i32,
    pub underlying_id: u32,
    pub market_depth_implied: i32,
    pub market_depth: i32,
    pub market_segment_id: u32,
    pub max_trade_vol: u32,
    pub min_lot_size: i32,
    pub min_lot_size_block: i32,
    pub min_lot_size_round_lot: i32,
    pub min_trade_vol: u32,
    pub contract_multiplier: i32,
    pub decay_quantity: i32,
    pub original_contract_size: i32,

    // Leg fields (for multi-leg strategies)
    pub leg_instrument_id: u32,
    pub leg_ratio_price_numerator: i32,
    pub leg_ratio_price_denominator: i32,
    pub leg_ratio_qty_numerator: i32,
    pub leg_ratio_qty_denominator: i32,
    pub leg_underlying_id: u32,

    // Small integer fields
    pub appl_id: i16,
    pub maturity_year: u16,
    pub decay_start_date: u16,
    pub channel_id: u16,
    pub leg_count: u16,
    pub leg_index: u16,

    // Fixed-length string fields (c_char arrays)
    pub currency: [c_char; 4],
    pub settl_currency: [c_char; 4],
    pub secsubtype: [c_char; 6],
    pub raw_symbol: [c_char; 71],              // SYMBOL_CSTR_LEN
    pub group: [c_char; 21],
    pub exchange: [c_char; 5],
    pub asset: [c_char; 11],                   // ASSET_CSTR_LEN_V3
    pub cfi: [c_char; 7],
    pub security_type: [c_char; 7],
    pub unit_of_measure: [c_char; 31],
    pub underlying: [c_char; 21],
    pub strike_price_currency: [c_char; 4],
    pub leg_raw_symbol: [c_char; 71],

    // Character flags
    pub instrument_class: c_char,              // B/C/F/K/M/P/S/T/X/Y
    pub match_algorithm: c_char,               // F/K/C/T/O/S/Q/Y/P/V
    pub security_update_action: c_char,        // A/M/D
    pub user_defined_instrument: c_char,       // Y/N
    pub leg_instrument_class: c_char,
    pub leg_side: c_char,

    // Byte fields
    pub main_fraction: u8,
    pub price_display_format: u8,
    pub sub_fraction: u8,
    pub underlying_product: u8,
    pub maturity_month: u8,
    pub maturity_day: u8,
    pub maturity_week: u8,
    pub contract_multiplier_unit: i8,
    pub flow_schedule_type: i8,
    pub tick_rule: u8,

    pub _reserved: [u8; 17],                   // Reserved for alignment
}
```

### 8. StatMsg (Version 3)

**Size**: 80 bytes
**Alignment**: 8 bytes
**RType**: 0x18 (STATISTICS)

```rust
#[repr(C)]
pub struct StatMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp
    pub ts_ref: u64,                // Reference timestamp
    pub price: i64,                 // Price statistic (fixed-point)
    pub quantity: i64,              // Quantity statistic (64-bit in v3)
    pub sequence: u32,              // Sequence number
    pub ts_in_delta: i32,           // Delta from ts_recv
    pub stat_type: u16,             // Type of statistic
    pub channel_id: u16,            // Channel ID
    pub update_action: u8,          // New/Delete
    pub stat_flags: u8,             // Additional flags
    pub _reserved: [u8; 18],        // Reserved
}
```

### 9. ImbalanceMsg

**Size**: 120 bytes
**Alignment**: 8 bytes
**RType**: 0x14 (IMBALANCE)

```rust
#[repr(C)]
pub struct ImbalanceMsg {
    pub hd: RecordHeader,
    pub ts_recv: u64,

    // Price fields (all fixed-point)
    pub ref_price: i64,
    pub auction_time: u64,
    pub cont_book_clr_price: i64,
    pub auct_interest_clr_price: i64,
    pub ssr_filling_price: i64,
    pub ind_match_price: i64,
    pub upper_collar: i64,
    pub lower_collar: i64,

    // Quantity fields
    pub paired_qty: u32,
    pub total_imbalance_qty: u32,
    pub market_imbalance_qty: u32,
    pub unpaired_qty: u32,

    // Character fields
    pub auction_type: c_char,
    pub side: c_char,
    pub unpaired_side: c_char,
    pub significant_imbalance: c_char,

    // Status fields
    pub auction_status: u8,
    pub freeze_status: u8,
    pub num_extensions: u8,

    pub _reserved: [u8; 1],
}
```

### 10. StatusMsg

**Size**: 48 bytes
**Alignment**: 8 bytes
**RType**: 0x12 (STATUS)

```rust
#[repr(C)]
pub struct StatusMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp
    pub action: u16,                // Status action type
    pub reason: u16,                // Reason for status change
    pub trading_event: u16,         // Trading event info
    pub is_trading: c_char,         // Y/N/~
    pub is_quoting: c_char,         // Y/N/~
    pub is_short_sell_restricted: c_char,  // Y/N/~
    pub _reserved: [u8; 7],
}
```

### 11. BBO (Best Bid/Offer) - BboMsg

**Size**: 72 bytes
**Alignment**: 8 bytes
**RTypes**: 0xC3 (BBO_1S), 0xC4 (BBO_1M)

```rust
#[repr(C)]
pub struct BboMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub price: i64,                 // Last trade price
    pub size: u32,                  // Last trade quantity
    pub _reserved1: u8,
    pub side: c_char,               // Last trade side
    pub flags: FlagSet,
    pub _reserved2: u8,
    pub ts_recv: u64,               // End of interval timestamp
    pub _reserved3: [u8; 4],
    pub sequence: u32,
    pub levels: [BidAskPair; 1],    // Best bid/ask (32 bytes)
}
```

### 12. CMBP-1 (Consolidated MBP) - Cmbp1Msg

**Size**: 72 bytes
**Alignment**: 8 bytes
**RTypes**: 0xB1 (CMBP_1), 0xC2 (TCBBO)

```rust
#[repr(C)]
pub struct Cmbp1Msg {
    pub hd: RecordHeader,
    pub price: i64,
    pub size: u32,
    pub action: c_char,
    pub side: c_char,
    pub flags: FlagSet,
    pub _reserved1: [u8; 1],
    pub ts_recv: u64,
    pub ts_in_delta: i32,
    pub _reserved2: [u8; 4],
    pub levels: [ConsolidatedBidAskPair; 1],  // 40 bytes
}
```

### 13. ConsolidatedBidAskPair

**Size**: 40 bytes
**Alignment**: 8 bytes

```rust
#[repr(C)]
pub struct ConsolidatedBidAskPair {
    pub bid_px: i64,                // Bid price
    pub ask_px: i64,                // Ask price
    pub bid_sz: u32,                // Bid size
    pub ask_sz: u32,                // Ask size
    pub bid_pb: u16,                // Best bid publisher ID
    pub _reserved1: [u8; 2],
    pub ask_pb: u16,                // Best ask publisher ID
    pub _reserved2: [u8; 2],
}
```

### 14. CBBO (Consolidated BBO) - CbboMsg

**Size**: 72 bytes
**Alignment**: 8 bytes
**RTypes**: 0xC0 (CBBO_1S), 0xC1 (CBBO_1M)

```rust
#[repr(C)]
pub struct CbboMsg {
    pub hd: RecordHeader,
    pub price: i64,                 // Last trade price
    pub size: u32,                  // Last trade quantity
    pub _reserved1: u8,
    pub side: c_char,               // Last trade side
    pub flags: FlagSet,
    pub _reserved2: u8,
    pub ts_recv: u64,               // End of interval timestamp
    pub _reserved3: [u8; 8],
    pub levels: [ConsolidatedBidAskPair; 1],  // 40 bytes
}
```

### 15. ErrorMsg

**Size**: 312 bytes
**Alignment**: 8 bytes
**RType**: 0x15 (ERROR)

```rust
#[repr(C)]
pub struct ErrorMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub err: [c_char; 302],         // Error message text
    pub code: u8,                   // Error code
    pub is_last: u8,                // Last in chain flag
}
```

### 16. SystemMsg

**Size**: 312 bytes
**Alignment**: 8 bytes
**RType**: 0x17 (SYSTEM)

```rust
#[repr(C)]
pub struct SystemMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub msg: [c_char; 303],         // System message text
    pub code: u8,                   // System code (heartbeat, etc.)
}
```

### 17. SymbolMappingMsg

**Size**: 160 bytes
**Alignment**: 8 bytes
**RType**: 0x16 (SYMBOL_MAPPING)

```rust
#[repr(C)]
pub struct SymbolMappingMsg {
    pub hd: RecordHeader,                       // 16 bytes
    pub stype_in: u8,                           // Input symbology type
    pub stype_in_symbol: [c_char; 71],          // Input symbol
    pub stype_out: u8,                          // Output symbology type
    pub stype_out_symbol: [c_char; 71],         // Output symbol
    pub start_ts: u64,                          // Mapping start timestamp
    pub end_ts: u64,                            // Mapping end timestamp
}
```

### 18. WithTsOut<T> (Live Gateway Wrapper)

**Size**: sizeof(T) + 8
**Alignment**: 8 bytes

When `ts_out` is enabled in metadata, records are wrapped with a send timestamp:

```rust
#[repr(C)]
pub struct WithTsOut<T: HasRType> {
    pub rec: T,                     // Inner record
    pub ts_out: u64,                // Gateway send timestamp (UNIX nanos)
}
```

---

## Crypto Extensions (Reserved RTypes 0xD0‑0xE1)

> **Status:** 🔵 Proposal / Design Phase  
> The following record layouts are proposed extensions for crypto markets,
> based on `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`. They are
> intended for DBN v4 / Crypto‑DBN and are not yet wired into the production
> Rust implementation or protobuf schemas.

### 19. FundingRateMsg

**Size**: 80 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD0 (FUNDING_RATE)

```rust
#[repr(C)]
pub struct FundingRateMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    // Funding rate data (all fixed-point i64)
    pub funding_rate: i64,          // Current funding rate (1e-9 scale = 0.0001% = 1e-7)
    pub predicted_funding_rate: i64,// Next funding rate prediction
    pub mark_price: i64,            // Fair price used for funding
    pub index_price: i64,           // Spot index price
    pub premium: i64,               // mark - index

    // Timestamps
    pub funding_interval_start: u64,// Start of current interval
    pub funding_interval_end: u64,  // End of current interval (next payment)

    // Metadata
    pub interval_duration_ms: u32,  // Interval in milliseconds (typically 28800000 = 8h)
    pub sequence: u32,              // Sequence number

    pub _reserved: [u8; 16],        // Reserved for future use
}
```

### 20. LiquidationMsg

**Size**: 96 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD1 (LIQUIDATION)

```rust
#[repr(C)]
pub struct LiquidationMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    // Liquidation details
    pub liquidation_price: i64,     // Price at liquidation (fixed-point)
    pub quantity: i64,              // Quantity liquidated (can be negative)
    pub bankrupt_price: i64,        // Bankruptcy price
    pub mark_price: i64,            // Mark price at liquidation
    pub unrealized_pnl: i64,        // Unrealized P&L at liquidation

    // Account info
    pub account_id: u64,            // Liquidated account (hashed)
    pub liquidator_id: u64,         // Liquidator account (hashed)

    // Liquidation metadata
    pub side: c_char,               // Position side: B/S (Long/Short)
    pub liquidation_type: u8,       // 0=partial, 1=full, 2=ADL
    pub sequence: u32,              // Sequence number
    pub position_value: i64,        // Total position value

    pub _reserved: [u8; 10],        // Reserved
}
```

### 21. MarkPriceMsg

**Size**: 64 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD2 (MARK_PRICE)

```rust
#[repr(C)]
pub struct MarkPriceMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    pub mark_price: i64,            // Fair price (fixed-point)
    pub index_price: i64,           // Underlying index
    pub last_price: i64,            // Last trade price
    pub premium: i64,               // mark - index

    pub update_reason: u8,          // 1=periodic, 2=volatility, 3=funding
    pub sequence: u32,              // Sequence number

    pub _reserved: [u8; 11],        // Reserved
}
```

### 22. IndexPriceMsg

**Size**: 96 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD3 (INDEX_PRICE)

```rust
#[repr(C)]
pub struct IndexPriceMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    pub index_price: i64,           // Weighted spot price (fixed-point)

    // Component prices (up to 5 exchanges)
    pub component_prices: [i64; 5], // Individual exchange prices
    pub component_weights: [i32; 5],// Weights (1e-6 scale, sum to 1e6)
    pub num_components: u8,         // Number of active components

    pub sequence: u32,              // Sequence number

    pub _reserved: [u8; 11],        // Reserved
}
```

### 23. DexSwapMsg

**Size**: 128 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD4 (DEX_SWAP)

```rust
#[repr(C)]
pub struct DexSwapMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp (off-chain)

    // On-chain metadata
    pub block_number: u64,          // Block number
    pub block_timestamp: u64,       // Block timestamp
    pub tx_hash: [u8; 32],          // Transaction hash
    pub log_index: u32,             // Event index in transaction

    // Swap details
    pub amount_in: i64,             // Input amount (token0)
    pub amount_out: i64,            // Output amount (token1)
    pub fee: i64,                   // Fee paid
    pub price_before: i64,          // Pool price before swap
    pub price_after: i64,           // Pool price after swap

    // Pool info
    pub pool_address: [u8; 20],     // Pool contract address (Ethereum)
    pub router_address: [u8; 20],   // Router contract used

    // Sender/recipient (hashed for privacy)
    pub sender_hash: u64,           // Hashed sender address
    pub recipient_hash: u64,        // Hashed recipient address

    pub gas_used: u32,              // Gas consumed
    pub gas_price: u32,             // Gas price in Gwei

    pub _reserved: [u8; 8],         // Reserved
}
```

### 24. DexPoolStateMsg

**Size**: 144 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD5 (DEX_POOL_STATE)

```rust
#[repr(C)]
pub struct DexPoolStateMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    // On-chain metadata
    pub block_number: u64,          // Block number
    pub block_timestamp: u64,       // Block timestamp

    // Pool state (AMM)
    pub reserve0: i64,              // Token0 reserves
    pub reserve1: i64,              // Token1 reserves
    pub k_value: i64,               // Constant product (reserve0 * reserve1)
    pub sqrt_price_x96: i64,        // UniV3 price (Q64.96 format)
    pub liquidity: i64,             // Total liquidity
    pub tick: i32,                  // Current tick (UniV3)

    // Pool metadata
    pub pool_address: [u8; 20],     // Pool contract address
    pub pool_type: u8,              // 1=UniV2, 2=UniV3, 3=Curve, etc.
    pub fee_tier: u32,              // Fee in basis points (30 = 0.30%)

    // Volume stats (optional, if available)
    pub volume_24h_token0: i64,     // 24h volume in token0
    pub volume_24h_token1: i64,     // 24h volume in token1
    pub fee_24h: i64,               // 24h fees collected

    // TVL
    pub tvl_usd: i64,               // Total Value Locked in USD

    pub sequence: u32,              // Sequence number
    pub _reserved: [u8; 12],        // Reserved
}
```

### 25. OraclePriceMsg

**Size**: 80 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD6 (ORACLE_PRICE)

```rust
#[repr(C)]
pub struct OraclePriceMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    // On-chain metadata
    pub block_number: u64,          // Block number
    pub block_timestamp: u64,       // Block timestamp

    // Oracle data
    pub price: i64,                 // Oracle price (fixed-point)
    pub confidence_interval: i64,   // CI width (±)
    pub num_sources: u8,            // Number of data sources
    pub oracle_type: u8,            // 1=Chainlink, 2=Pyth, 3=Band, etc.

    // Oracle metadata
    pub oracle_address: [u8; 20],   // Oracle contract address
    pub round_id: u64,              // Round/update ID

    pub sequence: u32,              // Sequence number
    pub _reserved: [u8; 10],        // Reserved
}
```

### 26. CrossRateMsg

**Size**: 80 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD8 (CROSS_RATE)

```rust
#[repr(C)]
pub struct CrossRateMsg {
    pub hd: RecordHeader,           // 16 bytes (instrument_id = normalized pair ID)
    pub ts_recv: u64,               // Capture timestamp

    // Exchange rates
    pub exchange1_price: i64,       // Price on exchange 1
    pub exchange2_price: i64,       // Price on exchange 2
    pub spread_bps: i32,            // Spread in basis points
    pub spread_pct: i64,            // Spread percentage (fixed-point)

    // Exchange identifiers
    pub exchange1_id: u16,          // Publisher ID of exchange 1
    pub exchange2_id: u16,          // Publisher ID of exchange 2

    // Arbitrage metrics
    pub arb_opportunity: i64,       // Profit potential (fixed-point)
    pub transfer_cost: i64,         // Est. transfer/fees cost
    pub net_profit: i64,            // Net arbitrage profit

    pub sequence: u32,              // Sequence number
    pub _reserved: [u8; 12],        // Reserved
}
```

### 27. BasisMsg

**Size**: 72 bytes  
**Alignment**: 8 bytes  
**RType**: 0xD9 (BASIS)

```rust
#[repr(C)]
pub struct BasisMsg {
    pub hd: RecordHeader,           // 16 bytes
    pub ts_recv: u64,               // Capture timestamp

    // Basis spread data
    pub spot_price: i64,            // Spot market price
    pub futures_price: i64,         // Futures/perpetual price
    pub basis: i64,                 // futures - spot (fixed-point)
    pub basis_pct: i64,             // Basis as percentage
    pub annualized_basis: i64,      // Annualized basis (for dated futures)

    // Instrument IDs
    pub spot_instrument_id: u32,    // Spot instrument ID
    pub futures_instrument_id: u32, // Futures instrument ID

    pub sequence: u32,              // Sequence number
    pub _reserved: [u8; 4],         // Reserved
}
```

### 28. StablecoinPegMsg

**Size**: 72 bytes  
**Alignment**: 8 bytes  
**RType**: 0xDA (STABLECOIN_PEG)

```rust
#[repr(C)]
pub struct StablecoinPegMsg {
    pub hd: RecordHeader,           // 16 bytes (instrument_id = stablecoin ID)
    pub ts_recv: u64,               // Capture timestamp

    // Peg status
    pub price_usd: i64,             // Current USD price (fixed-point)
    pub deviation_bps: i32,         // Deviation from $1.00 in bps
    pub deviation_pct: i64,         // Deviation percentage

    // Supply metrics
    pub total_supply: i64,          // Total supply
    pub market_cap: i64,            // Market cap in USD

    // Exchange data
    pub exchange_id: u16,           // Source exchange
    pub volume_24h: i64,            // 24h volume

    // Peg confidence
    pub peg_confidence: u8,         // 0-100 confidence score
    pub alert_level: u8,            // 0=normal, 1=warning, 2=critical

    pub sequence: u32,              // Sequence number
    pub _reserved: [u8; 10],        // Reserved
}
```

The remaining reserved RTypes (0xDB‑0xE1) have high-level summaries in the
requirements document but do not yet have finalized binary layouts here.

---

## Enumerations

### Side (Market Side)

```rust
#[repr(u8)]
pub enum Side {
    Ask = b'A',     // Sell order/aggressor
    Bid = b'B',     // Buy order/aggressor
    None = b'N',    // No side specified
}
```

### Action (Order Event)

```rust
#[repr(u8)]
pub enum Action {
    Modify = b'M',  // Order modified
    Trade = b'T',   // Trade executed
    Fill = b'F',    // Order filled
    Cancel = b'C',  // Order cancelled
    Add = b'A',     // New order added
    Clear = b'R',   // Clear book
    None = b'N',    // No action
}
```

### InstrumentClass

```rust
#[repr(u8)]
pub enum InstrumentClass {
    Bond = b'B',
    Call = b'C',
    Future = b'F',
    Stock = b'K',
    MixedSpread = b'M',
    Put = b'P',
    FutureSpread = b'S',
    OptionSpread = b'T',
    FxSpot = b'X',
    CommoditySpot = b'Y',
}
```

### MatchAlgorithm

```rust
#[repr(u8)]
pub enum MatchAlgorithm {
    Undefined = b' ',
    Fifo = b'F',
    Configurable = b'K',
    ProRata = b'C',
    FifoLmm = b'T',
    ThresholdProRata = b'O',
    FifoTopLmm = b'S',
    ThresholdProRataLmm = b'Q',
    EurodollarFutures = b'Y',
    TimeProRata = b'P',
    InstitutionalPrioritization = b'V',
}
```

### UserDefinedInstrument

```rust
#[repr(u8)]
pub enum UserDefinedInstrument {
    No = b'N',
    Yes = b'Y',
}
```

### SecurityUpdateAction

```rust
#[repr(u8)]
pub enum SecurityUpdateAction {
    Add = b'A',
    Modify = b'M',
    Delete = b'D',
}
```

### SType (Symbology Type)

```rust
#[repr(u8)]
pub enum SType {
    InstrumentId = 0,       // Numeric ID
    RawSymbol = 1,          // Original publisher symbols
    Continuous = 3,         // Continuous contracts
    Parent = 4,             // Parent symbols (e.g., ES.FUT)
    NasdaqSymbol = 5,       // NASDAQ suffixes
    CmsSymbol = 6,          // CMS suffixes
    Isin = 7,               // ISO 6166
    UsCode = 8,             // CUSIP codes
    BbgCompId = 9,          // Bloomberg composite ID
    BbgCompTicker = 10,     // Bloomberg composite ticker
    Figi = 11,              // Bloomberg FIGI
    FigiTicker = 12,        // Bloomberg FIGI ticker
}
```

### Schema (Data Schema)

```rust
#[repr(u16)]
pub enum Schema {
    Mbo = 0,            // Market-by-order
    Mbp1 = 1,           // MBP depth 1
    Mbp10 = 2,          // MBP depth 10
    Tbbo = 3,           // Trade with BBO
    Trades = 4,         // All trades
    Ohlcv1S = 5,        // OHLCV 1-second
    Ohlcv1M = 6,        // OHLCV 1-minute
    Ohlcv1H = 7,        // OHLCV 1-hour
    Ohlcv1D = 8,        // OHLCV 1-day UTC
    Definition = 9,     // Instrument definitions
    Statistics = 10,    // Statistics
    Status = 11,        // Trading status
    Imbalance = 12,     // Auction imbalance
    OhlcvEod = 13,      // OHLCV end-of-day
    Cmbp1 = 14,         // Consolidated MBP depth 1
    Cbbo1S = 15,        // Consolidated BBO 1-second
    Cbbo1M = 16,        // Consolidated BBO 1-minute
    Tcbbo = 17,         // Trade with consolidated BBO
    Bbo1S = 18,         // BBO 1-second
    Bbo1M = 19,         // BBO 1-minute
}
```

### Encoding

```rust
#[repr(u8)]
pub enum Encoding {
    Dbn = 0,
    Csv = 1,
    Json = 2,
}
```

### Compression

```rust
#[repr(u8)]
pub enum Compression {
    None = 0,
    Zstd = 1,
}
```

### StatType (Statistics Type)

```rust
#[repr(u16)]
pub enum StatType {
    OpeningPrice = 1,
    IndicativeOpeningPrice = 2,
    SettlementPrice = 3,
    TradingSessionLowPrice = 4,
    TradingSessionHighPrice = 5,
    ClearedVolume = 6,
    LowestOffer = 7,
    HighestBid = 8,
    OpenInterest = 9,
    FixingPrice = 10,
    ClosePrice = 11,
    NetChange = 12,
    Vwap = 13,
    Volatility = 14,
    Delta = 15,
    UncrossingPrice = 16,
}
```

### StatUpdateAction

```rust
#[repr(u8)]
pub enum StatUpdateAction {
    New = 1,
    Delete = 2,
}
```

### StatusAction

```rust
#[repr(u16)]
pub enum StatusAction {
    None = 0,
    PreOpen = 1,
    PreCross = 2,
    Quoting = 3,
    Cross = 4,
    Rotation = 5,
    NewPriceIndication = 6,
    Trading = 7,
    Halt = 8,
    Pause = 9,
    Suspend = 10,
    PreClose = 11,
    Close = 12,
    PostClose = 13,
    SsrChange = 14,
    NotAvailableForTrading = 15,
}
```

### StatusReason

```rust
#[repr(u16)]
pub enum StatusReason {
    None = 0,
    Scheduled = 1,
    SurveillanceIntervention = 2,
    MarketEvent = 3,
    InstrumentActivation = 4,
    InstrumentExpiration = 5,
    RecoveryInProcess = 6,
    Regulatory = 10,
    Administrative = 11,
    NonCompliance = 12,
    FilingsNotCurrent = 13,
    SecTradingSuspension = 14,
    NewIssue = 15,
    IssueAvailable = 16,
    IssuesReviewed = 17,
    FilingReqsSatisfied = 18,
    NewsPending = 30,
    NewsReleased = 31,
    NewsAndResumptionTimes = 32,
    NewsNotForthcoming = 33,
    OrderImbalance = 40,
    LuldPause = 50,
    Operational = 60,
    AdditionalInformationRequested = 70,
    MergerEffective = 80,
    Etf = 90,
    CorporateAction = 100,
    NewSecurityOffering = 110,
    MarketWideHaltLevel1 = 120,
    MarketWideHaltLevel2 = 121,
    MarketWideHaltLevel3 = 122,
    MarketWideHaltCarryover = 123,
    MarketWideHaltResumption = 124,
    QuotationNotAvailable = 130,
}
```

### TradingEvent

```rust
#[repr(u16)]
pub enum TradingEvent {
    None = 0,
    NoCancel = 1,
    ChangeTradingSession = 2,
    ImpliedMatchingOn = 3,
    ImpliedMatchingOff = 4,
}
```

### TriState

```rust
#[repr(u8)]
pub enum TriState {
    NotAvailable = b'~',
    No = b'N',
    Yes = b'Y',
}
```

### ErrorCode

```rust
#[repr(u8)]
pub enum ErrorCode {
    AuthFailed = 1,
    ApiKeyDeactivated = 2,
    ConnectionLimitExceeded = 3,
    SymbolResolutionFailed = 4,
    InvalidSubscription = 5,
    InternalError = 6,
}
```

### SystemCode

```rust
#[repr(u8)]
pub enum SystemCode {
    Heartbeat = 0,
    SubscriptionAck = 1,
    SlowReaderWarning = 2,
    ReplayCompleted = 3,
    EndOfInterval = 4,
}
```

### VersionUpgradePolicy

```rust
pub enum VersionUpgradePolicy {
    AsIs,           // Decode as-is
    UpgradeToV2,    // Upgrade v1 to v2
    UpgradeToV3,    // Upgrade v1/v2 to v3 (default)
}
```

---

## Constants & Sentinel Values

### Price Constants

```rust
pub const FIXED_PRICE_SCALE: i64 = 1_000_000_000;  // 1e9 scale factor
pub const UNDEF_PRICE: i64 = i64::MAX;              // Null price sentinel
```

**Price Conversion**:
- DBN stores prices as `i64` fixed-point
- Real price = `dbn_price / FIXED_PRICE_SCALE`
- Example: `1500000000` → `$1.50`

### Quantity Constants

```rust
pub const UNDEF_ORDER_SIZE: u32 = u32::MAX;         // Null order size
pub const UNDEF_STAT_QUANTITY: i64 = i64::MAX;      // Null stat quantity (v3)
```

### Timestamp Constants

```rust
pub const UNDEF_TIMESTAMP: u64 = u64::MAX;          // Null timestamp
```

**Timestamp Format**:
- All timestamps are UNIX nanoseconds (since 1970-01-01 00:00:00 UTC)
- Type: `u64`

### Symbol Length Constants

```rust
// Version 3
pub const SYMBOL_CSTR_LEN: usize = 71;              // Symbol string length
pub const ASSET_CSTR_LEN: usize = 11;               // Asset string length

// Version 2
pub const SYMBOL_CSTR_LEN_V2: usize = 71;
pub const ASSET_CSTR_LEN_V2: usize = 7;

// Version 1
pub const SYMBOL_CSTR_LEN_V1: usize = 22;
pub const ASSET_CSTR_LEN_V1: usize = 7;
```

### Null Constants

```rust
pub const NULL_LIMIT: u64 = 0;
pub const NULL_RECORD_COUNT: u64 = u64::MAX;
pub const NULL_SCHEMA: u16 = u16::MAX;
pub const NULL_STYPE: u8 = u8::MAX;
```

### Record Size Constants

```rust
pub const MAX_RECORD_LEN: usize = 528;  // sizeof(WithTsOut<InstrumentDefMsg>)
```

### Flag Constants

Bit flags for the `flags` field in records:

```rust
pub const F_LAST: u8 = 1 << 7;              // Last message in event
pub const F_TOB: u8 = 1 << 6;               // Top-of-book message
pub const F_SNAPSHOT: u8 = 1 << 5;          // Snapshot message
pub const F_MBP: u8 = 1 << 4;               // MBP message
pub const F_BAD_TS_RECV: u8 = 1 << 3;       // Bad ts_recv
pub const F_MAYBE_BAD_BOOK: u8 = 1 << 2;    // Possibly bad book
pub const F_PUBLISHER_SPECIFIC: u8 = 1 << 1; // Publisher-specific flag
```

---

## Version History

### Version 3 (Current - Released 2025-05-28)

**Major Changes**:
1. **InstrumentDefMsg**:
   - Added multi-leg strategy fields: `leg_price`, `leg_delta`, `leg_instrument_id`, `leg_ratio_price_numerator`, `leg_ratio_price_denominator`, `leg_ratio_qty_numerator`, `leg_ratio_qty_denominator`, `leg_underlying_id`, `leg_count`, `leg_index`, `leg_raw_symbol`, `leg_instrument_class`, `leg_side`
   - Expanded `asset` from 7 to 11 bytes
   - Expanded `raw_instrument_id` from 32 to 64 bits
   - Removed statistics-related fields: `trading_reference_date`, `trading_reference_price`, `settl_price_type`
   - Removed status-related field: `md_security_trading_status`
   - Size: **520 bytes** (up from 400 bytes in v2)

2. **StatMsg**:
   - Expanded `quantity` from 32-bit to 64-bit signed integer
   - Size: **80 bytes** (up from 64 bytes in v1/v2)

3. **Metadata**:
   - Always encoded with length divisible by 8 bytes for alignment

**Compatibility**:
- v1 and v2 records can be converted to v3 with `From` trait
- Default upgrade policy: `UpgradeToV3`

### Version 2 (Released 2023-11-15)

**Major Changes**:
1. **InstrumentDefMsg**:
   - Expanded `SYMBOL_CSTR_LEN` from 22 to 71 bytes
   - Added `raw_instrument_id` field
   - Changed `security_update_action` from enum to `c_char`
   - Repositioned `strike_price` field (serialization order unchanged)
   - Size: **400 bytes** (up from 360 bytes in v1)

2. **SymbolMappingMsg**:
   - Added `stype_in` and `stype_out` fields
   - Expanded symbol fields to 71 bytes
   - Size: **160 bytes** (up from 80 bytes in v1)

3. **ErrorMsg** and **SystemMsg**:
   - Increased message field lengths
   - Added `is_last` field to ErrorMsg
   - Added `code` field to both
   - Size: **312 bytes** (up from 80 bytes in v1)

4. **Metadata**:
   - Added `symbol_cstr_len` field

**Compatibility**:
- v1 records can be converted to v2 with `From` trait

### Version 1 (Original - Released 2023-02-22)

**Characteristics**:
- Initial DBN release (renamed from DBZ)
- Fixed-width records
- 8-byte alignment
- Zstandard compression support

**Record Sizes (v1)**:
- InstrumentDefMsg: 360 bytes
- StatMsg: 64 bytes
- ErrorMsg: 80 bytes
- SystemMsg: 80 bytes
- SymbolMappingMsg: 80 bytes
- Symbol length: 22 bytes

---

## Requirements & Specifications

### 1. Alignment Requirements

**All records MUST**:
- Be 8-byte aligned
- Have sizes divisible by 8
- Use `#[repr(C)]` layout
- Have no padding between fields

**Verification**:
```rust
assert_eq!(std::mem::align_of::<RecordType>(), 8);
assert_eq!(std::mem::size_of::<RecordType>() % 8, 0);
```

### 2. Byte Order

- **Little-endian** for all multi-byte integers
- Platform-native for simplicity (assumes x86/ARM)

### 3. String Encoding

**Fixed-length C strings** (`c_char` arrays):
- UTF-8 encoded
- NUL-terminated (last byte = 0)
- Padded with NULs if shorter than array length
- Maximum length = array size - 1 (for NUL terminator)

**Example**:
```
"AAPL" in [c_char; 71] = ['A','A','P','L','\0', 0, 0, ..., 0]
```

### 4. Null/Undefined Values

| Field Type | Sentinel Value | Constant |
|------------|----------------|----------|
| Price (i64) | `i64::MAX` | `UNDEF_PRICE` |
| Order size (u32) | `u32::MAX` | `UNDEF_ORDER_SIZE` |
| Stat quantity (i64) | `i64::MAX` | `UNDEF_STAT_QUANTITY` |
| Timestamp (u64) | `u64::MAX` | `UNDEF_TIMESTAMP` |
| Channel ID (u8) | `u8::MAX` | - |
| Channel ID (u16) | `u16::MAX` | - |

### 5. Fixed-Point Price Encoding

**Scale Factor**: 1,000,000,000 (1e9)

**Formula**:
```
decimal_price = fixed_point_value / FIXED_PRICE_SCALE
fixed_point_value = decimal_price * FIXED_PRICE_SCALE
```

**Examples**:
- $100.50 → `100_500_000_000`
- $0.0001 → `100_000`
- $123456.789 → `123_456_789_000_000`

**Precision**: Up to 9 decimal places

### 6. Timestamp Encoding

**Format**: UNIX nanoseconds (i64/u64)

**Formula**:
```
nanos = seconds_since_epoch * 1_000_000_000 + nanosecond_fraction
```

**Example**:
```
2024-01-01 00:00:00.123456789 UTC
= 1704067200 seconds * 1e9 + 123456789
= 1704067200123456789 nanoseconds
```

### 7. Record Length Calculation

The `length` field in `RecordHeader` stores record size in **32-bit words**:

```rust
length = (size_of::<RecordType>() / 4) as u8
```

**Examples**:
- MboMsg: 80 bytes → length = 20
- Mbp10Msg: 424 bytes → length = 106
- InstrumentDefMsg: 520 bytes → length = 130

### 8. File Format Validation

**DBN File Header**:
```
Bytes 0-3:  "DBN\x00"  (magic string)
Bytes 4-7:  Reserved (0x00)
Byte 8:     Version (1, 2, or 3)
Bytes 9-12: Metadata length (u32 little-endian)
```

**Validation Checks**:
1. Magic string matches "DBN\x00"
2. Version is 1, 2, or 3
3. Metadata length is reasonable (< 1GB)
4. Metadata is valid UTF-8 JSON
5. All records are properly aligned
6. Record lengths match expected sizes

### 9. Compression

**Zstandard (zstd)**:
- Default compression level: 3
- Includes checksum for integrity
- Multi-frame support
- Streaming decompression supported

**File Extensions**:
- `.dbn` - Uncompressed DBN
- `.dbn.zst` - Zstandard compressed DBN
- `.dbn.frag` - DBN fragment (records only, no metadata)
- `.dbn.frag.zst` - Compressed fragment

### 10. Metadata Encoding Rules

1. Metadata is JSON encoded
2. Must be valid UTF-8
3. Length includes full metadata section
4. Padded to 8-byte alignment
5. Symbol arrays must fit in `symbol_cstr_len`

**Required Fields**:
- `dataset`: Dataset code
- `schema`: Schema or null
- `start`: Start timestamp
- `stype_in`: Input symbology or null
- `stype_out`: Output symbology

### 11. Symbol Constraints

**Version 3**:
- Max symbol length: 70 chars (+ NUL terminator = 71 bytes)
- Max asset length: 10 chars (+ NUL terminator = 11 bytes)

**Version 2**:
- Max symbol length: 70 chars (+ NUL terminator = 71 bytes)
- Max asset length: 6 chars (+ NUL terminator = 7 bytes)

**Version 1**:
- Max symbol length: 21 chars (+ NUL terminator = 22 bytes)
- Max asset length: 6 chars (+ NUL terminator = 7 bytes)

### 12. Stream Processing Rules

**Sequential Processing**:
- Records MUST be processed in order
- Each record is self-contained
- No implicit state between records (except book building)

**Book Building** (for MBO/MBP):
- Maintain order book state per `instrument_id`
- Apply actions in sequence: Add, Modify, Cancel, Clear
- Clear action resets entire book for that instrument

**Timestamps**:
- `ts_event`: Venue matching engine time
- `ts_recv`: Databento capture server time
- `ts_in_delta`: Offset from ts_recv (venue send time)
- `ts_out`: Live gateway send time (when present)

### 13. Error Handling

**Invalid Records**:
- Unknown rtype → Skip or error
- Length mismatch → Error
- Invalid enum value → Error or default

**Partial Reads**:
- Incomplete record → Wait for more data (streaming)
- EOF mid-record → Error

### 14. Python Compatibility

All records are exposed via PyO3 bindings:
- Field access via properties
- Enum conversion methods
- Pretty-printing for timestamps/prices
- Zero-copy where possible

### 15. C Compatibility

C FFI bindings provided via cbindgen:
- All structs use `#[repr(C)]`
- No Rust-specific types in public API
- Conversion functions for versioned records

---

## Complete Record Size Reference

| Record Type | Version | Size (bytes) | Alignment |
|-------------|---------|--------------|-----------|
| RecordHeader | All | 16 | 8 |
| MboMsg | All | 80 | 8 |
| TradeMsg | All | 80 | 8 |
| Mbp1Msg | All | 104 | 8 |
| Mbp10Msg | All | 424 | 8 |
| BidAskPair | All | 32 | 8 |
| OhlcvMsg | All | 56 | 8 |
| InstrumentDefMsg | v1 | 360 | 8 |
| InstrumentDefMsg | v2 | 400 | 8 |
| InstrumentDefMsg | v3 | 520 | 8 |
| StatMsg | v1/v2 | 64 | 8 |
| StatMsg | v3 | 80 | 8 |
| ImbalanceMsg | All | 120 | 8 |
| StatusMsg | All | 48 | 8 |
| BboMsg | All | 72 | 8 |
| Cmbp1Msg | All | 72 | 8 |
| ConsolidatedBidAskPair | All | 40 | 8 |
| CbboMsg | All | 72 | 8 |
| ErrorMsg | v1 | 80 | 8 |
| ErrorMsg | v2/v3 | 312 | 8 |
| SystemMsg | v1 | 80 | 8 |
| SystemMsg | v2/v3 | 312 | 8 |
| SymbolMappingMsg | v1 | 80 | 8 |
| SymbolMappingMsg | v2/v3 | 160 | 8 |
| WithTsOut\<T\> | All | sizeof(T) + 8 | 8 |

---

## Schema Usage Examples

### Market-by-Order (MBO)
**Purpose**: Full order book depth with individual orders
**Record**: `MboMsg`
**Use Cases**: Order flow analysis, liquidity analysis, full book reconstruction

### Market-by-Price (MBP-1, MBP-10)
**Purpose**: Aggregated price levels
**Records**: `Mbp1Msg`, `Mbp10Msg`
**Use Cases**: Trading signals, price level analysis, reduced data volume

### Trades
**Purpose**: All trade executions
**Record**: `TradeMsg`
**Use Cases**: Execution analysis, trade reporting, VWAP calculations

### OHLCV (1S/1M/1H/1D/EOD)
**Purpose**: Candlestick/bar data at various intervals
**Record**: `OhlcvMsg`
**Use Cases**: Charting, technical analysis, backtesting

### Definition
**Purpose**: Instrument metadata and definitions
**Record**: `InstrumentDefMsg`
**Use Cases**: Symbol mapping, contract specs, options chains

### Statistics
**Purpose**: Exchange-published statistics
**Record**: `StatMsg`
**Use Cases**: Settlement prices, open interest, VWAP

### Status
**Purpose**: Trading status updates
**Record**: `StatusMsg`
**Use Cases**: Trading hours, halts, circuit breakers

### Imbalance
**Purpose**: Auction imbalance information
**Record**: `ImbalanceMsg`
**Use Cases**: Opening/closing auction analysis

---

## Implementation Notes

### 1. Memory Layout
All structs use `#[repr(C)]` to ensure consistent memory layout across platforms and languages.

### 2. Zero-Copy Parsing
Records can be parsed via transmutation from byte slices:
```rust
let record_ref = RecordRef::new(byte_slice)?;
let mbo_msg: &MboMsg = record_ref.get()?;
```

### 3. Streaming
Supports both synchronous and asynchronous streaming:
- Sync: `streaming_iterator` pattern
- Async: `async/await` with tokio

### 4. Encoding Flexibility
Can encode to:
- DBN (binary, fastest)
- CSV (human-readable)
- JSON (structured, interoperable)

### 5. Publisher & Dataset System
Each `publisher_id` maps to a specific dataset and venue combination, maintained in the `publishers.rs` module.

---

## References

- **Official Documentation**: https://databento.com/docs/standards-and-conventions/databento-binary-encoding
- **GitHub Repository**: https://github.com/databento/dbn
- **Rust Crate**: https://crates.io/crates/dbn
- **Python Package**: https://pypi.org/project/databento-dbn

---

## Revision History

| Date | Version | Changes |
|------|---------|---------|
| 2023-02-22 | v1 | Initial DBN release |
| 2023-11-15 | v2 | Expanded symbols, updated InstrumentDefMsg |
| 2025-05-28 | v3 | Multi-leg strategies, 64-bit StatMsg quantity |
| 2025-09-23 | Latest | Current version: DBNv3 |

---

**Document Version**: 1.0
**Last Updated**: 2025-10-15
**DBN Version**: 3
**Extracted From**: databento/dbn repository (commit: 781ee43)
