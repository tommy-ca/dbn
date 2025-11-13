# Crypto Markets DBN Adaptation: Requirements & Specifications

## Executive Summary

This document proposes a comprehensive adaptation of the Databento Binary Encoding (DBN) format for cryptocurrency markets, addressing the unique requirements of 24/7 global trading across centralized exchanges (CEXs), decentralized exchanges (DEXs), and on-chain protocols.

**Document Version**: 1.0
**Date**: 2025-10-15
**Base DBN Version**: 3
**Target**: Crypto Lakehouse Architecture

---

## Table of Contents

1. [Gap Analysis: Traditional vs Crypto Markets](#gap-analysis)
2. [Crypto Market Requirements](#crypto-market-requirements)
3. [Proposed Schema Extensions](#proposed-schema-extensions)
4. [Enhanced Metadata for Crypto](#enhanced-metadata)
5. [New Record Types](#new-record-types)
6. [Crypto Lakehouse Architecture](#crypto-lakehouse-architecture)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Performance Specifications](#performance-specifications)

---

## Gap Analysis: Traditional vs Crypto Markets

### What Works Well (Existing DBN Strengths)

✅ **Fixed-width binary format**: Perfect for high-throughput crypto data
✅ **8-byte alignment**: Optimal for modern CPUs
✅ **Nanosecond timestamps**: Sufficient precision for crypto (even for DEX block times)
✅ **Fixed-point prices**: Works well, though crypto needs higher precision in some cases
✅ **MBO/MBP schemas**: Directly applicable to CEX orderbooks
✅ **Trade schema**: Works for spot and derivatives
✅ **OHLCV schemas**: Perfect for crypto charting
✅ **Compression-friendly**: Critical given crypto's high message rates
✅ **Multi-version support**: Essential for evolving crypto markets

### Critical Gaps for Crypto Markets

❌ **No perpetual futures support**: Missing funding rates, mark price, index price
❌ **No liquidation events**: Critical for derivatives risk management
❌ **Limited multi-venue handling**: Crypto has 100+ venues, need better consolidation
❌ **No quote currency differentiation**: BTC/USDT vs BTC/USD vs BTC/EUR critical
❌ **No DEX support**: Missing AMM pools, swap events, liquidity provision
❌ **No on-chain events**: Transfers, minting, burning essential for DeFi
❌ **No funding rate schema**: Core to perpetuals pricing
❌ **No basis/spread tracking**: Spot-futures, cross-exchange spreads
❌ **No oracle price feeds**: Chainlink, Pyth, etc. critical for DeFi
❌ **No stablecoin peg tracking**: USDT/USD premium vital
❌ **No cross-chain data**: Bridge events, cross-chain swaps
❌ **Limited instrument class**: Need perpetuals, prediction markets, synthetic assets

### Compatibility Issues

⚠️ **24/7 trading**: No session concept, timestamps never reset
⚠️ **Symbol conventions**: Multiple formats across exchanges (BTC-USD, BTCUSD, BTC/USD, BTC_USD)
⚠️ **Decimal precision**: Some altcoins need >9 decimal places (e.g., SHIB at $0.000008)
⚠️ **Venue fragmentation**: Same instrument on 50+ exchanges with different symbols
⚠️ **Settlement**: Instant settlement vs traditional T+2
⚠️ **Trading halts**: Rare in crypto, exchanges may delist instead
⚠️ **Corporate actions**: Rare, but token swaps, airdrops, hard forks exist

---

## Crypto Market Requirements

### 1. Product Type Requirements

#### 1.1 Spot Markets
**Status**: ✅ Supported by existing DBN schemas
- Uses: MBO, MBP-1, MBP-10, Trades, OHLCV
- Gap: Need quote currency differentiation

#### 1.2 Perpetual Futures (Inverse & Linear)
**Status**: ❌ **NEW SCHEMA REQUIRED**

**Requirements**:
- Funding rate (8-hour, continuous)
- Mark price (fair price)
- Index price (spot reference)
- Open interest
- Funding interval timestamps
- Premium/discount to spot
- Liquidation price levels
- Insurance fund size
- Clawback events

**Data Volume**: ~100 events/second during volatility per contract

#### 1.3 Dated Futures
**Status**: ⚠️ Partially supported
- Can use existing schemas
- Gap: Need settlement price, basis tracking

#### 1.4 Options
**Status**: ⚠️ Partially supported
- Gap: IV surface, Greeks need enhancement
- Gap: Settlement mechanism (cash vs physical)

#### 1.5 Prediction Markets (Polymarket, etc.)
**Status**: ❌ **NEW SCHEMA REQUIRED**
- Binary outcomes
- Multi-outcome markets
- Probability updates
- Settlement mechanisms

#### 1.6 Synthetic Assets & Indices
**Status**: ❌ **NEW SCHEMA REQUIRED**
- Composition tracking
- Rebalancing events
- Component weights

### 2. Venue Type Requirements

#### 2.1 Centralized Exchanges (CEXs)
**Status**: ✅ Mostly supported

**Top CEXs to support**:
- Binance, Coinbase, OKX, Bybit, Kraken, Bitfinex, Huobi, KuCoin, Gate.io, Bitget
- Deribit (options specialist)
- CME (institutional)

**CEX-specific needs**:
- ✅ Orderbook depth (MBO/MBP) - supported
- ✅ Trades - supported
- ❌ Delisting events - need new schema
- ❌ Maintenance windows - need tracking
- ❌ API rate limits - metadata

#### 2.2 Decentralized Exchanges (DEXs)
**Status**: ❌ **NEW SCHEMAS REQUIRED**

**Major DEX Types**:
1. **AMM (Constant Product)**: Uniswap V2, PancakeSwap, SushiSwap
2. **Concentrated Liquidity**: Uniswap V3, Curve V2
3. **Orderbook DEX**: dYdX, Serum/OpenBook
4. **Aggregators**: 1inch, Matcha, Paraswap

**DEX-specific data requirements**:
- Pool state (reserves, K-value)
- Swap events (in/out amounts, fees)
- Liquidity provision/removal events
- Price impact calculations
- Slippage tracking
- Gas costs per swap
- MEV detection (sandwich attacks, frontrunning)
- Pool creation/destruction
- Fee tier changes
- Liquidity concentration (V3 positions)

**Data Volume**: ~5-50 events/block on Ethereum (12s), ~200-1000/block on BSC (3s)

#### 2.3 Lending Protocols
**Status**: ❌ **NEW SCHEMA REQUIRED**
- Aave, Compound, MakerDAO, Venus
- Borrow/lend rates
- Collateralization ratios
- Liquidation events
- Total value locked (TVL)

### 3. Data Type Requirements

#### 3.1 Market Data (Tier 1 - Critical)

| Data Type | Current Support | Priority | New Schema |
|-----------|----------------|----------|------------|
| Orderbook (L1/L2/L3) | ✅ MBO/MBP | P0 | No |
| Trades | ✅ TradeMsg | P0 | No |
| OHLCV (1s/1m/1h/1d) | ✅ OhlcvMsg | P0 | No |
| Funding Rates | ❌ None | P0 | **YES** |
| Liquidations | ❌ None | P0 | **YES** |
| Open Interest | ⚠️ StatMsg | P0 | Enhance |
| Mark/Index Prices | ❌ None | P0 | **YES** |

#### 3.2 Market Data (Tier 2 - Important)

| Data Type | Current Support | Priority | New Schema |
|-----------|----------------|----------|------------|
| DEX Swaps | ❌ None | P1 | **YES** |
| DEX Pool States | ❌ None | P1 | **YES** |
| Oracle Prices | ❌ None | P1 | **YES** |
| Cross-Exchange Spreads | ❌ None | P1 | **YES** |
| Basis (Spot-Futures) | ❌ None | P1 | **YES** |
| Options IV Surface | ⚠️ StatMsg | P2 | Enhance |
| Stablecoin Pegs | ❌ None | P1 | **YES** |

#### 3.3 On-Chain Data (Tier 3 - Ecosystem)

| Data Type | Current Support | Priority | New Schema |
|-----------|----------------|----------|------------|
| Token Transfers | ❌ None | P2 | **YES** |
| Smart Contract Events | ❌ None | P2 | **YES** |
| Gas Prices | ❌ None | P2 | **YES** |
| Block Metadata | ❌ None | P2 | **YES** |
| Bridge Events | ❌ None | P3 | **YES** |

### 4. Symbology Requirements

**Challenge**: Crypto symbols are non-standardized across venues.

**Examples of same instrument**:
- Binance: `BTCUSDT`
- Coinbase: `BTC-USD`
- Kraken: `XBTUSD` or `BTC/USD`
- Deribit: `BTC-PERPETUAL`
- CME: `BTC1!`

**Requirements**:
1. **Normalized symbol format**: Use internal standard (e.g., `{BASE}/{QUOTE}`)
2. **Venue-specific raw symbols**: Store original exchange symbols
3. **Quote currency tracking**: Distinguish USDT, USD, USDC, BUSD
4. **Product type suffix**: `.PERP`, `.FUT-20250328`, `.OPT-C-50000-20250328`
5. **Multi-asset mapping**: Handle token migrations (e.g., LUNA → LUNC)

**Proposed Format**:
```
{BASE}/{QUOTE}.{PRODUCT_TYPE}[-{DETAILS}]

Examples:
BTC/USDT.SPOT
BTC/USD.PERP
ETH/USDT.FUT-20250328
BTC/USD.OPT-C-50000-20250625
BTC/USDT.INDEX
```

### 5. Timestamp Requirements

**Current DBN**: ✅ Nanosecond precision, UNIX epoch

**Crypto-specific needs**:
- ✅ Works for CEX timestamps
- ⚠️ **Block timestamps** for on-chain data (need block number + block timestamp)
- ⚠️ **Funding timestamp** (specific times: 00:00, 08:00, 16:00 UTC typically)
- ✅ **Exchange latency tracking** (ts_recv - ts_event works)
- ⚠️ **MEV timestamp** (need transaction position in block)

**New timestamp fields needed**:
- `block_number: u64` (for on-chain events)
- `tx_index: u32` (position in block)
- `log_index: u32` (for smart contract events)

### 6. Precision Requirements

**Current**: Fixed-point i64 with 1e-9 scale (9 decimals)

**Crypto needs**:
- ✅ BTC: 8 decimals (satoshis) - **SUPPORTED**
- ✅ ETH: 18 decimals (wei) - **PROBLEM**: Need 1e-18 scale
- ✅ Most tokens: 6-18 decimals
- ❌ **Meme coins**: Some need >9 decimals (SHIB: $0.00000842)
- ❌ **DeFi rates**: Need high precision for APY (0.01234567% = 1.234567e-7)

**Solutions**:
1. **Option A**: Use i128 for ultra-high precision (breaking change)
2. **Option B**: Use dynamic scale in metadata (complex)
3. **Option C**: Store raw values + decimals field (recommended)
4. **Option D**: Use f64 for prices (loses precision, not recommended)

**Recommendation**: Add `price_decimals: u8` to InstrumentDefMsg, keep i64

### 7. Performance Requirements

**CEX Market Data Rates** (peak):
- Binance BTC/USDT: ~10,000 messages/second (MBO)
- Top 10 pairs: ~100,000 msg/s aggregate
- All markets: ~1,000,000 msg/s

**DEX Event Rates**:
- Uniswap V3: ~50-200 events/block on Ethereum (12s blocks)
- PancakeSwap BSC: ~500-2000 events/block (3s blocks)

**Storage Requirements** (per day):
- MBO (Binance top 100): ~500 GB/day uncompressed
- Trades (all CEXs): ~50 GB/day
- OHLCV (1s bars): ~10 GB/day
- DEX events: ~100 GB/day (all chains)
- Funding rates: ~1 MB/day

**Compression Target**: 5:1 ratio with Zstandard

**Query Performance**:
- Point queries (single instrument): <1ms
- Range queries (1 hour): <100ms
- Aggregations (1 day OHLCV): <500ms
- Cross-venue queries: <1s

---

## Proposed Schema Extensions

### Schema Version Strategy

**Crypto-DBN Version**: 4 (extends DBN v3)

**Approach**:
- Maintain backward compatibility with DBN v3
- Add crypto-specific schemas with new RType values (0xD0-0xFF range)
- Extend InstrumentClass enum
- Add crypto-specific metadata fields

### New RType Allocations

| RType | Value | Name | Description |
|-------|-------|------|-------------|
| FUNDING_RATE | 0xD0 | Perpetual funding rate | Funding rate updates |
| LIQUIDATION | 0xD1 | Liquidation event | Perpetual/margin liquidation |
| MARK_PRICE | 0xD2 | Mark price update | Fair price for perps |
| INDEX_PRICE | 0xD3 | Index price | Spot reference for perps |
| DEX_SWAP | 0xD4 | DEX swap event | AMM swap executed |
| DEX_POOL_STATE | 0xD5 | DEX pool snapshot | AMM pool state |
| ORACLE_PRICE | 0xD6 | Oracle price feed | Chainlink, Pyth, etc. |
| TRANSFER | 0xD7 | Token transfer | On-chain transfer event |
| CROSS_RATE | 0xD8 | Cross-exchange rate | Arbitrage opportunity |
| BASIS | 0xD9 | Basis spread | Spot-futures spread |
| STABLECOIN_PEG | 0xDA | Stablecoin peg status | USDT/USD premium |
| GAS_PRICE | 0xDB | Gas price update | Network fee data |
| BLOCK_INFO | 0xDC | Block metadata | On-chain block data |
| SMART_CONTRACT_EVENT | 0xDD | Contract event | Generic event log |
| INSURANCE_FUND | 0xDE | Insurance fund update | Protocol insurance |
| CLAWBACK | 0xDF | Clawback event | Socialized losses |
| DELISTING | 0xE0 | Delisting notice | Instrument removal |
| TOKEN_MIGRATION | 0xE1 | Token swap event | Contract migration |

Reserved range: 0xE2-0xFF for future crypto extensions

### Enhanced InstrumentClass

```rust
#[repr(u8)]
pub enum InstrumentClass {
    // Existing (from DBN v3)
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

    // NEW: Crypto-specific
    CryptoSpot = b'1',          // BTC/USDT spot
    PerpetualInverse = b'2',    // BTC/USD inverse perp
    PerpetualLinear = b'3',     // BTC/USDT linear perp
    CryptoFuture = b'4',        // Dated crypto future
    PredictionMarket = b'5',    // Binary/multi outcome
    SyntheticAsset = b'6',      // Synthetic tracking
    LpToken = b'7',             // AMM LP token
    WrappedAsset = b'8',        // WBTC, WETH, etc.
    Stablecoin = b'9',          // USDT, USDC, DAI
}
```

### New Schemas (20 total additions)

#### Schema: FundingRate (NEW)
**RType**: 0xD0 | **Record**: `FundingRateMsg`

#### Schema: Liquidations (NEW)
**RType**: 0xD1 | **Record**: `LiquidationMsg`

#### Schema: MarkPrice (NEW)
**RType**: 0xD2 | **Record**: `MarkPriceMsg`

#### Schema: IndexPrice (NEW)
**RType**: 0xD3 | **Record**: `IndexPriceMsg`

#### Schema: DexSwap (NEW)
**RType**: 0xD4 | **Record**: `DexSwapMsg`

#### Schema: DexPoolState (NEW)
**RType**: 0xD5 | **Record**: `DexPoolStateMsg`

#### Schema: OraclePrice (NEW)
**RType**: 0xD6 | **Record**: `OraclePriceMsg`

#### Schema: CrossRate (NEW)
**RType**: 0xD8 | **Record**: `CrossRateMsg`

#### Schema: Basis (NEW)
**RType**: 0xD9 | **Record**: `BasisMsg`

#### Schema: StablecoinPeg (NEW)
**RType**: 0xDA | **Record**: `StablecoinPegMsg`

---

## New Record Types

### 1. FundingRateMsg

**Size**: 80 bytes | **Alignment**: 8 bytes | **RType**: 0xD0

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

**Key Fields**:
- `funding_rate`: Annualized funding rate (e.g., 0.01% = 100000000)
- `interval_duration_ms`: Usually 8 hours (28,800,000 ms) but can vary
- `premium`: Mark price - Index price

**Usage**: Track perpetual futures funding costs

---

### 2. LiquidationMsg

**Size**: 96 bytes | **Alignment**: 8 bytes | **RType**: 0xD1

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

**Key Fields**:
- `liquidation_type`: 0=partial, 1=full, 2=auto-deleveraging (ADL)
- `account_id`: Privacy-preserving hash of account
- `unrealized_pnl`: P&L at time of liquidation

**Usage**: Risk monitoring, liquidation cascades analysis, market impact studies

---

### 3. MarkPriceMsg

**Size**: 64 bytes | **Alignment**: 8 bytes | **RType**: 0xD2

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

**Usage**: Perpetual fair pricing, liquidation calculations, PnL calculations

---

### 4. IndexPriceMsg

**Size**: 96 bytes | **Alignment**: 8 bytes | **RType**: 0xD3

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

**Key Fields**:
- `component_prices`: Prices from each constituent exchange
- `component_weights`: Weights (e.g., 250000 = 25%)
- `num_components`: Typically 3-5 major exchanges

**Usage**: Index calculation transparency, outlier detection

---

### 5. DexSwapMsg

**Size**: 128 bytes | **Alignment**: 8 bytes | **RType**: 0xD4

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

**Key Fields**:
- `tx_hash`: Full transaction hash for lookup
- `pool_address`: Identifies the AMM pool
- `price_before/after`: For price impact calculation
- `gas_used`: Transaction cost

**Usage**: DEX trading analysis, MEV detection, slippage tracking

---

### 6. DexPoolStateMsg

**Size**: 144 bytes | **Alignment**: 8 bytes | **RType**: 0xD5

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

**Key Fields**:
- `k_value`: For constant product AMMs (x * y = k)
- `sqrt_price_x96`: Uniswap V3 price format
- `tick`: Uniswap V3 tick (price level)
- `pool_type`: AMM type identifier

**Usage**: Liquidity tracking, arbitrage opportunities, pool analysis

---

### 7. OraclePriceMsg

**Size**: 80 bytes | **Alignment**: 8 bytes | **RType**: 0xD6

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

**Usage**: DeFi protocol price feeds, oracle reliability monitoring

---

### 8. CrossRateMsg

**Size**: 80 bytes | **Alignment**: 8 bytes | **RType**: 0xD8

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

**Usage**: Cross-exchange arbitrage, market inefficiency detection

---

### 9. BasisMsg

**Size**: 72 bytes | **Alignment**: 8 bytes | **RType**: 0xD9

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

**Usage**: Carry trade analysis, contango/backwardation tracking

---

### 10. StablecoinPegMsg

**Size**: 72 bytes | **Alignment**: 8 bytes | **RType**: 0xDA

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

**Usage**: Stablecoin risk monitoring, depeg detection, arbitrage

---

### 11. Additional Record Types (Summary)

#### GasPriceMsg (0xDB)
**Size**: 64 bytes
- Network gas prices
- EIP-1559 base fee + priority fee
- Gas limit data
- Congestion metrics

#### BlockInfoMsg (0xDC)
**Size**: 96 bytes
- Block metadata (number, hash, timestamp)
- Block size, gas used
- Transaction count
- Validator/miner info

#### SmartContractEventMsg (0xDD)
**Size**: 256 bytes
- Generic event log
- Topic hashes
- Event data
- Contract address

#### InsuranceFundMsg (0xDE)
**Size**: 64 bytes
- Insurance fund balance
- Fund changes
- Clawback amounts

#### DelistingMsg (0xE0)
**Size**: 64 bytes
- Delisting announcement
- Effective date
- Reason code

---

## Enhanced Metadata for Crypto

### Extended Metadata Structure

```rust
pub struct CryptoMetadata {
    // Base DBN metadata (inherited)
    pub base: Metadata,

    // Crypto-specific additions
    pub quote_currency: String,         // USDT, USD, USDC, BTC, ETH
    pub base_currency: String,          // BTC, ETH, SOL, etc.
    pub venue_type: VenueType,          // CEX, DEX, OnChain
    pub chain_id: Option<u64>,          // Ethereum=1, BSC=56, etc.
    pub product_type: ProductType,      // Spot, Perp, Future, Option
    pub settlement_currency: String,    // For derivatives
    pub contract_size: Option<f64>,     // Contract multiplier
    pub tick_size: Option<f64>,         // Minimum price increment
    pub lot_size: Option<f64>,          // Minimum order size
    pub maker_fee: Option<f64>,         // Maker fee rate
    pub taker_fee: Option<f64>,         // Taker fee rate
    pub funding_interval_hours: Option<u8>, // For perpetuals (8h typical)
    pub price_decimals: u8,             // Decimal precision
    pub quantity_decimals: u8,          // Quantity precision
    pub multi_venue: bool,              // Cross-exchange data
    pub aggregated: bool,               // Pre-aggregated data
    pub oracle_sources: Vec<String>,    // Oracle providers
    pub dex_protocol: Option<String>,   // Uniswap, SushiSwap, etc.
    pub amm_version: Option<String>,    // V2, V3, etc.
}
```

### VenueType Enum

```rust
#[repr(u8)]
pub enum VenueType {
    CentralizedExchange = 1,    // Binance, Coinbase, etc.
    DecentralizedExchange = 2,  // Uniswap, PancakeSwap
    OnChainProtocol = 3,        // Aave, Compound
    OracleNetwork = 4,          // Chainlink, Pyth
    Aggregator = 5,             // 1inch, Paraswap
    Crosschain = 6,             // Multi-chain data
}
```

### ProductType Enum

```rust
#[repr(u8)]
pub enum ProductType {
    Spot = 1,
    PerpetualInverse = 2,       // BTC/USD (inverse)
    PerpetualLinear = 3,        // BTC/USDT (linear)
    DatedFuture = 4,
    CallOption = 5,
    PutOption = 6,
    Index = 7,
    OraclePrice = 8,
    LiquidityPool = 9,          // AMM pool
    LendingMarket = 10,
    PredictionMarket = 11,
}
```

---

## Crypto Lakehouse Architecture

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Ingestion Layer                              │
├─────────────────────────────────────────────────────────────────────┤
│  CEX Connectors  │  DEX Indexers   │  On-Chain Nodes  │  Oracles   │
│  (WebSocket)     │  (Event Logs)   │  (RPC/gRPC)      │  (APIs)    │
└──────────┬───────────────────┬──────────────┬───────────────────────┘
           │                   │              │
           ▼                   ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Normalization Layer                             │
├─────────────────────────────────────────────────────────────────────┤
│  • Symbol Normalization (BTC/USDT standardization)                  │
│  • Timestamp Normalization (block time → UNIX nanos)                │
│  • Price Precision Handling (decimals → fixed-point)                │
│  • Deduplication (cross-venue)                                      │
│  • DBN Encoding (struct packing)                                    │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Streaming Buffer                              │
├─────────────────────────────────────────────────────────────────────┤
│  Kafka / Redpanda Topics:                                           │
│  • crypto.trades.spot                                               │
│  • crypto.trades.perps                                              │
│  • crypto.orderbook.l2                                              │
│  • crypto.funding_rates                                             │
│  • crypto.liquidations                                              │
│  • crypto.dex.swaps                                                 │
│  • crypto.oracle.prices                                             │
│  Format: DBN binary (Zstd compressed)                               │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Storage Layer (Lakehouse)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Hot Storage (Last 7 days)                       │   │
│  │  Object Storage: MinIO / S3                                  │   │
│  │  Format: DBN files (Zstd compressed)                         │   │
│  │  Partitioning: /year/month/day/hour/venue/symbol/           │   │
│  │  File size: 100MB-500MB per partition                        │   │
│  │  Latency: <100ms                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Warm Storage (8-90 days)                        │   │
│  │  Object Storage: S3 Intelligent-Tiering                      │   │
│  │  Format: DBN files (Zstd level 9)                            │   │
│  │  Partitioning: /year/month/day/venue/                        │   │
│  │  File size: 1GB-5GB per partition                            │   │
│  │  Latency: ~500ms                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Cold Storage (>90 days)                         │   │
│  │  Object Storage: S3 Glacier / Deep Archive                   │   │
│  │  Format: DBN archives (Zstd level 19)                        │   │
│  │  Partitioning: /year/month/                                  │   │
│  │  File size: 10GB-50GB per partition                          │   │
│  │  Latency: Hours to restore                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Metadata Catalog                                │
├─────────────────────────────────────────────────────────────────────┤
│  Apache Iceberg / Delta Lake:                                       │
│  • Schema evolution tracking                                        │
│  • Partition pruning                                                │
│  • Time travel                                                      │
│  • ACID transactions                                                │
│  • Column statistics (min/max price, timestamp ranges)              │
│  Database: PostgreSQL / DynamoDB                                    │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Query Layer                                   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   DuckDB         │  │   Apache Arrow   │  │   Polars         │  │
│  │   (Analytics)    │  │   (In-Memory)    │  │   (DataFrame)    │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Custom DBN Query Engine                          │  │
│  │  • Zero-copy DBN parsing                                      │  │
│  │  • Predicate pushdown (timestamp, instrument_id filters)      │  │
│  │  • Vectorized operations (SIMD)                               │  │
│  │  • Parallel file scanning                                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Application Layer                               │
├─────────────────────────────────────────────────────────────────────┤
│  • Trading Strategies (backtesting)                                 │
│  • Risk Management (liquidation monitoring)                         │
│  • Market Making (orderbook analysis)                               │
│  • Research & Analytics (Jupyter notebooks)                         │
│  • Dashboards (Grafana, Streamlit)                                 │
│  • APIs (REST, GraphQL, WebSocket)                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Partitioning Strategy

#### Hierarchical Partitioning

```
/crypto-lakehouse/
  /schema={schema_name}/          # mbo, trades, funding_rates, etc.
    /year={YYYY}/
      /month={MM}/
        /day={DD}/
          /hour={HH}/             # For high-volume schemas
            /venue={exchange}/    # binance, coinbase, uniswap_v3
              /symbol={pair}/     # btc_usdt, eth_usd
                data_*.dbn.zst    # Files: 100MB-500MB each
```

**Example**:
```
/crypto-lakehouse/
  /schema=trades/
    /year=2025/
      /month=10/
        /day=15/
          /hour=14/
            /venue=binance/
              /symbol=btc_usdt/
                data_001.dbn.zst (250MB)
                data_002.dbn.zst (230MB)
```

#### Partition Keys Rationale

1. **schema**: Top-level separation by data type (fast schema filtering)
2. **year/month/day**: Standard time partitioning (lifecycle management, time-travel)
3. **hour**: High-frequency data needs hourly partitioning (parallel queries)
4. **venue**: Exchange-specific queries (cross-venue comparison)
5. **symbol**: Instrument-specific queries (single-pair analysis)

#### File Naming Convention

```
data_{sequence}_{start_ts}_{end_ts}_{record_count}.dbn.zst

Example:
data_00001_1697376000000000000_1697379599999999999_1234567.dbn.zst
      ↑           ↑                       ↑                ↑
   Sequence   Start Timestamp       End Timestamp    Record Count
```

### Table Schema (Iceberg/Delta)

```sql
CREATE TABLE crypto_lakehouse.trades (
    -- Partition columns
    schema STRING,
    year INT,
    month INT,
    day INT,
    hour INT,
    venue STRING,
    symbol STRING,

    -- File metadata
    file_path STRING,
    file_size_bytes BIGINT,
    record_count BIGINT,
    compression STRING,
    dbn_version TINYINT,

    -- Statistics (for predicate pushdown)
    min_timestamp BIGINT,
    max_timestamp BIGINT,
    min_price BIGINT,
    max_price BIGINT,
    min_instrument_id INT,
    max_instrument_id INT,
    total_volume DOUBLE,

    -- Metadata
    created_at TIMESTAMP,
    updated_at TIMESTAMP
)
PARTITIONED BY (schema, year, month, day, hour, venue, symbol)
STORED AS DBN;
```

### Storage Tiering Rules

| Tier | Age | Storage Class | Compression | Query Frequency | Cost/TB/Month |
|------|-----|--------------|-------------|-----------------|---------------|
| **Hot** | 0-7 days | S3 Standard / MinIO | Zstd level 3 | High (100+ QPS) | $23 |
| **Warm** | 8-90 days | S3 Intelligent-Tiering | Zstd level 9 | Medium (10-100 QPS) | $10 |
| **Cold** | 91-365 days | S3 Glacier Instant | Zstd level 15 | Low (1-10 QPS) | $4 |
| **Archive** | >365 days | S3 Deep Archive | Zstd level 19 | Rare (<1 QPS) | $1 |

### Indexing Strategy

#### Primary Indexes

1. **Timestamp Index** (Iceberg built-in)
   - Partition-level: min/max timestamps
   - File-level: Parquet metadata
   - Record-level: Sequential (implicit)

2. **Instrument ID Index**
   - Hash-based sharding
   - Bloom filters per file
   - Dictionary encoding

3. **Venue Index**
   - Partition key (built-in)
   - Cardinality: ~100 venues

#### Secondary Indexes (External)

Use ClickHouse or PostgreSQL for secondary indexing:

```sql
-- Instrument lookup table
CREATE TABLE instruments (
    instrument_id INT PRIMARY KEY,
    symbol STRING,
    base_currency STRING,
    quote_currency STRING,
    venue STRING,
    product_type STRING,
    INDEX idx_symbol (symbol),
    INDEX idx_venue_symbol (venue, symbol)
);

-- Fast symbol → instrument_id lookup
SELECT instrument_id FROM instruments
WHERE venue = 'binance' AND symbol = 'BTC/USDT';
```

### Query Optimization

#### 1. Predicate Pushdown

```rust
// Filter at file level before reading
let query = QueryBuilder::new()
    .schema("trades")
    .time_range(start_ts, end_ts)      // Partition pruning
    .instruments(vec![instrument_id])  // File-level filter
    .min_price(50000 * FIXED_PRICE_SCALE)  // Statistics filter
    .build();
```

#### 2. Vectorized Processing

```rust
// Process records in batches of 1024
const BATCH_SIZE: usize = 1024;
let mut batch: [TradeMsg; BATCH_SIZE] = unsafe { MaybeUninit::uninit().assume_init() };

while let Some(count) = decoder.decode_batch(&mut batch)? {
    // SIMD-optimized processing
    process_batch_vectorized(&batch[..count]);
}
```

#### 3. Parallel File Scanning

```rust
// Scan multiple files in parallel
use rayon::prelude::*;

let results: Vec<_> = file_paths
    .par_iter()
    .map(|path| scan_dbn_file(path))
    .collect();
```

### Data Lifecycle Management

#### Retention Policy

```yaml
schemas:
  trades:
    hot: 7 days
    warm: 90 days
    cold: 365 days
    archive: 7 years

  orderbook:
    hot: 7 days
    warm: 30 days
    cold: 90 days
    archive: 1 year

  funding_rates:
    hot: 30 days
    warm: 365 days
    archive: 10 years

  liquidations:
    hot: 30 days
    warm: 365 days
    archive: 10 years

  dex_swaps:
    hot: 7 days
    warm: 90 days
    archive: 5 years
```

#### Compaction Strategy

**Small File Problem**: High-frequency data creates many small files

**Solution**: Scheduled compaction

```python
# Daily compaction job
def compact_partition(schema, year, month, day, hour, venue, symbol):
    """
    Merge small files into larger ones
    Target: 500MB per file
    """
    files = list_files(schema, year, month, day, hour, venue, symbol)

    if sum(f.size for f in files) < 100_000_000:  # Skip if <100MB total
        return

    # Merge files
    merged_records = []
    for file in files:
        merged_records.extend(decode_dbn(file))

    # Write compacted file
    write_dbn(merged_records,
              compression_level=9,
              target_size=500_000_000)

    # Delete originals
    delete_files(files)
```

### Disaster Recovery

#### Backup Strategy

1. **Hot Tier**: Real-time replication to secondary region (S3 CRR)
2. **Warm/Cold**: Daily snapshots + versioning
3. **Archive**: Immutable with legal hold

#### Recovery Objectives

- **RPO** (Recovery Point Objective): 1 minute (hot), 1 hour (warm/cold)
- **RTO** (Recovery Time Objective): 5 minutes (hot), 1 hour (warm/cold)

---

## Implementation Roadmap

### Phase 1: Core Infrastructure (Months 1-2)

**Week 1-2: DBN Schema Extensions**
- [ ] Implement new RTypes (0xD0-0xFF)
- [ ] Add CryptoMetadata struct
- [ ] Extend InstrumentClass enum
- [ ] Unit tests for all new schemas

**Week 3-4: Encoders/Decoders**
- [ ] Implement FundingRateMsg encoder/decoder
- [ ] Implement LiquidationMsg encoder/decoder
- [ ] Implement DexSwapMsg encoder/decoder
- [ ] Benchmark encoding/decoding performance

**Week 5-6: Storage Layer**
- [ ] Setup MinIO/S3 buckets with partitioning
- [ ] Implement Iceberg catalog integration
- [ ] Setup partition lifecycle management
- [ ] Implement file compaction service

**Week 7-8: Ingestion Pipelines**
- [ ] CEX WebSocket connectors (Binance, Coinbase)
- [ ] Normalization service (symbol mapping)
- [ ] Kafka topic setup
- [ ] Data quality validation

### Phase 2: Advanced Features (Months 3-4)

**Week 9-10: DEX Integration**
- [ ] Ethereum node setup (Geth/Erigon)
- [ ] Event log indexer
- [ ] DexSwapMsg generation
- [ ] DexPoolStateMsg snapshots

**Week 11-12: Oracle & Cross-Exchange**
- [ ] Chainlink oracle integration
- [ ] CrossRateMsg computation
- [ ] BasisMsg calculation
- [ ] Multi-venue orderbook aggregation

**Week 13-14: Query Engine**
- [ ] Custom DBN reader with predicate pushdown
- [ ] Vectorized aggregations (SIMD)
- [ ] Parallel file scanning
- [ ] Query result caching

**Week 15-16: Monitoring & Alerting**
- [ ] Prometheus metrics
- [ ] Grafana dashboards
- [ ] Liquidation alerts
- [ ] Data quality monitoring

### Phase 3: Production Hardening (Month 5)

**Week 17-18: Performance Optimization**
- [ ] Load testing (10M msg/sec)
- [ ] Compression tuning
- [ ] Query optimization
- [ ] Resource scaling

**Week 19-20: Reliability**
- [ ] Disaster recovery testing
- [ ] Failover procedures
- [ ] Data validation pipelines
- [ ] Backfill procedures

### Phase 4: Analytics & Applications (Month 6)

**Week 21-22: Analytics Tools**
- [ ] Jupyter notebook templates
- [ ] DuckDB integration
- [ ] Polars DataFrames
- [ ] Backtesting framework

**Week 23-24: API Layer**
- [ ] REST API (FastAPI/Actix)
- [ ] GraphQL API
- [ ] WebSocket streaming
- [ ] API documentation

---

## Performance Specifications

### Throughput Requirements

| Component | Requirement | Target | Stretch Goal |
|-----------|-------------|--------|--------------|
| **Ingestion** | 1M msg/s | 5M msg/s | 10M msg/s |
| **Encoding** | 100K rec/s | 500K rec/s | 1M rec/s |
| **Storage Write** | 1 GB/s | 5 GB/s | 10 GB/s |
| **Query (point)** | <10ms p99 | <5ms p99 | <1ms p99 |
| **Query (range 1h)** | <500ms | <200ms | <100ms |
| **Query (agg 1d)** | <2s | <1s | <500ms |

### Latency Requirements

| Operation | Requirement | Target |
|-----------|-------------|--------|
| **CEX → Kafka** | <50ms p99 | <20ms p99 |
| **Kafka → Storage** | <500ms p99 | <200ms p99 |
| **Storage → Query** | <100ms p99 | <50ms p99 |
| **End-to-End** | <1s p99 | <500ms p99 |

### Storage Requirements

**Daily Data Volume** (estimated):

| Data Type | Records/Day | Size/Record | Daily Volume | Annual Volume |
|-----------|-------------|-------------|--------------|---------------|
| MBO (top 100) | 50B | 80 bytes | 4 TB | 1.5 PB |
| Trades (all) | 1B | 80 bytes | 80 GB | 30 TB |
| OHLCV 1s | 8.6M | 56 bytes | 480 MB | 175 GB |
| Funding Rates | 100K | 80 bytes | 8 MB | 3 GB |
| Liquidations | 1M | 96 bytes | 96 MB | 35 GB |
| DEX Swaps | 100M | 128 bytes | 12.8 GB | 4.7 TB |
| **Total (uncompressed)** | | | **~5 TB/day** | **~1.8 PB/year** |
| **Total (5:1 compression)** | | | **~1 TB/day** | **~365 TB/year** |

**Storage Cost Estimate** (AWS S3):
- Hot (7 days): 7 TB × $23/TB = $161/month
- Warm (90 days): 90 TB × $10/TB = $900/month
- Cold (365 days): 365 TB × $4/TB = $1,460/month
- **Total**: ~$2,500/month for 1 year of data

### Compression Benchmarks

| Compression | Level | Ratio | Encode Speed | Decode Speed |
|-------------|-------|-------|--------------|--------------|
| None | - | 1:1 | - | - |
| Zstd | 3 | 4.5:1 | 500 MB/s | 2 GB/s |
| Zstd | 9 | 5.2:1 | 150 MB/s | 2 GB/s |
| Zstd | 15 | 5.8:1 | 50 MB/s | 2 GB/s |
| Zstd | 19 | 6.1:1 | 10 MB/s | 2 GB/s |

**Recommendation**:
- Hot tier: Level 3 (balanced)
- Warm tier: Level 9 (better compression)
- Cold tier: Level 15 (max compression, rare access)

---

## Summary & Next Steps

### What We've Achieved

✅ **Comprehensive gap analysis** between traditional markets (DBN) and crypto markets
✅ **18 new schemas** designed for crypto-specific data types
✅ **Complete record definitions** with fixed-width, 8-byte aligned structures
✅ **Enhanced metadata** for crypto instruments and venues
✅ **Lakehouse architecture** with tiered storage and query optimization
✅ **Performance specifications** for production deployment
✅ **Implementation roadmap** with 6-month timeline

### Critical Success Factors

1. **Performance**: Must handle 1M+ msg/sec with <50ms latency
2. **Reliability**: 99.99% uptime, RPO <1 minute
3. **Cost**: Keep storage costs <$5K/month for 1 year of data
4. **Query Speed**: Sub-second queries for 1-hour ranges
5. **Scalability**: Linear scaling to 10M msg/sec

### Recommended Next Steps

1. **Prototype** (2 weeks):
   - Implement 3 core schemas (FundingRate, Liquidation, DexSwap)
   - Build simple encoder/decoder
   - Test with Binance + Uniswap data

2. **Pilot** (4 weeks):
   - Deploy on staging infrastructure
   - Ingest 7 days of data
   - Run query benchmarks
   - Validate compression ratios

3. **Production** (12 weeks):
   - Follow Phase 1-3 roadmap
   - Gradual rollout (10% → 50% → 100%)
   - Monitor and optimize

4. **Expansion**:
   - Add more exchanges (OKX, Bybit, etc.)
   - Add more chains (Solana, Avalanche, etc.)
   - Build advanced analytics features

---

**Document End**

*For questions or contributions, please refer to the main DBN repository or crypto-specific extensions.*
