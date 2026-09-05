# Crypto-DBN Quick Reference Guide

**Version**: 1.0 | **Date**: 2025-11-13 | **Status**: 🔵 Proposal
**Base**: DBN v3 → Crypto-DBN v4

---

## 📊 Executive Summary

This quick reference provides a condensed overview of **proposed** Crypto-DBN extensions for adapting Databento's binary format to cryptocurrency markets. See [Status & Implementation](#status--implementation) section for details on implementation status of each feature.

### Key Metrics

| Metric | Value |
|--------|-------|
| **New Schemas** | 18 crypto-specific record types |
| **RType Range** | 0xD0-0xFF (48 slots reserved) |
| **Storage Efficiency** | 5:1 compression ratio |
| **Throughput Target** | 5M messages/second |
| **Daily Storage** | ~1 TB/day (compressed) |
| **Query Performance** | <100ms for 1-hour ranges |

---

## 🎯 Critical Gaps Identified

### ✅ What Works (No Changes Needed)

- Fixed-width binary format
- Nanosecond timestamps
- MBO/MBP/Trade schemas for CEX orderbooks
- OHLCV bars
- Fixed-point price encoding
- Compression (Zstandard)

### ❌ What's Missing (New Schemas Required)

| Gap | Impact | Priority |
|-----|--------|----------|
| Perpetual funding rates | Cannot price perpetuals | **P0** |
| Liquidation events | Risk blind spot | **P0** |
| DEX swaps/pools | Missing 30% of volume | **P0** |
| Mark/Index prices | Derivatives pricing | **P0** |
| Oracle prices | DeFi dependency | **P1** |
| Cross-exchange spreads | Arbitrage tracking | **P1** |
| Stablecoin peg monitoring | Systemic risk | **P1** |
| On-chain events | Incomplete picture | **P2** |

---

## 🆕 New Record Types (18 Total)

### Core Derivatives (P0)

| RType | Name | Size | Use Case |
|-------|------|------|----------|
| **0xD0** | `FundingRateMsg` | 80B | Perpetual funding costs |
| **0xD1** | `LiquidationMsg` | 96B | Liquidation cascades |
| **0xD2** | `MarkPriceMsg` | 64B | Fair price for perps |
| **0xD3** | `IndexPriceMsg` | 96B | Spot price reference |

### DEX/DeFi (P0-P1)

| RType | Name | Size | Use Case |
|-------|------|------|----------|
| **0xD4** | `DexSwapMsg` | 128B | AMM swap events |
| **0xD5** | `DexPoolStateMsg` | 144B | AMM pool snapshots |
| **0xD6** | `OraclePriceMsg` | 80B | Chainlink/Pyth feeds |

### Cross-Market (P1)

| RType | Name | Size | Use Case |
|-------|------|------|----------|
| **0xD8** | `CrossRateMsg` | 80B | Cross-exchange arbitrage |
| **0xD9** | `BasisMsg` | 72B | Spot-futures spread |
| **0xDA** | `StablecoinPegMsg` | 72B | Depeg monitoring |

### Infrastructure (P2)

| RType | Name | Size | Use Case |
|-------|------|------|----------|
| **0xDB** | `GasPriceMsg` | 64B | Network fees |
| **0xDC** | `BlockInfoMsg` | 96B | Block metadata |
| **0xDD** | `SmartContractEventMsg` | 256B | Generic events |
| **0xDE** | `InsuranceFundMsg` | 64B | Protocol insurance |
| **0xDF** | `ClawbackMsg` | 64B | Socialized losses |
| **0xE0** | `DelistingMsg` | 64B | Token removal |
| **0xE1** | `TokenMigrationMsg` | 80B | Contract swaps |

---

## 📐 Schema Design Principles

### 1. **Fixed-Width Records**
All records remain 8-byte aligned with no padding:
```
Size ≡ 0 (mod 8)  ✅ 72, 80, 96, 128, 144, 256 bytes
```

### 2. **Timestamp Strategy**
```rust
pub ts_event: u64,      // Exchange/venue timestamp
pub ts_recv: u64,       // Databento capture timestamp
pub block_number: u64,  // For on-chain events
pub block_timestamp: u64 // Block time
```

### 3. **Price Encoding**
```rust
// Keep existing: i64 fixed-point (1e-9 scale)
pub price: i64,

// Add for extreme precision:
pub price_decimals: u8,  // In InstrumentDefMsg metadata

// Calculation:
real_price = price / 10^price_decimals
```

### 4. **On-Chain Fields**
```rust
pub block_number: u64,      // Block height
pub tx_hash: [u8; 32],      // Transaction hash
pub log_index: u32,         // Event index
pub contract_address: [u8; 20] // Ethereum address (20 bytes)
```

---

## 🔤 Symbology Standards

### Normalized Format
```
{BASE}/{QUOTE}.{PRODUCT_TYPE}[-{DETAILS}]

Examples:
BTC/USDT.SPOT
BTC/USD.PERP
ETH/USDT.FUT-20250328
BTC/USD.OPT-C-50000-20250625
```

### Exchange Mapping
```
Binance:  BTCUSDT       → BTC/USDT.SPOT
Coinbase: BTC-USD       → BTC/USD.SPOT
Kraken:   XBT/USD       → BTC/USD.SPOT (XBT=BTC)
Deribit:  BTC-PERPETUAL → BTC/USD.PERP
```

### Key Fields
```rust
pub raw_symbol: [c_char; 71],    // Original exchange symbol
pub base_currency: String,       // BTC, ETH, SOL
pub quote_currency: String,      // USD, USDT, USDC, BTC
pub venue: String,               // binance, uniswap_v3
```

---

## 🏗️ Lakehouse Architecture

### Storage Tiers

```
┌─────────────────────────────────────────────────────────┐
│ HOT (0-7 days)       │ 7 TB  │ S3 Standard  │ <100ms   │
│ WARM (8-90 days)     │ 90 TB │ S3 IT        │ ~500ms   │
│ COLD (91-365 days)   │ 365TB │ S3 Glacier   │ ~5s      │
│ ARCHIVE (>365 days)  │ ∞     │ Deep Archive │ ~hours   │
└─────────────────────────────────────────────────────────┘
```

### Partitioning Hierarchy

```
/crypto-lakehouse/
  /schema={trades,mbo,funding_rates,...}/
    /year={YYYY}/
      /month={MM}/
        /day={DD}/
          /hour={HH}/
            /venue={binance,coinbase,...}/
              /symbol={btc_usdt,eth_usd,...}/
                data_*.dbn.zst
```

**Benefits**:
- ✅ Fast time-range queries (year/month/day pruning)
- ✅ Fast symbol queries (direct partition access)
- ✅ Fast venue filtering (partition key)
- ✅ Parallel processing (multiple partitions)
- ✅ Lifecycle management (delete old partitions)

### File Size Targets

| Tier | Target Size | Max Size | Records/File |
|------|-------------|----------|--------------|
| Hot | 250 MB | 500 MB | 1-5M |
| Warm | 1 GB | 5 GB | 5-25M |
| Cold | 10 GB | 50 GB | 50-250M |

---

## ⚡ Performance Targets

### Ingestion Pipeline

```
CEX WebSocket → Kafka → DBN Encoder → S3
   <20ms         <50ms      <100ms      <200ms

Total Latency: <500ms (p99)
Throughput: 5M msg/s
```

### Query Performance

| Query Type | Latency (p99) | Example |
|------------|---------------|---------|
| Point query | <1ms | Single trade by ID |
| Range (1 min) | <10ms | OHLCV 1-minute |
| Range (1 hour) | <100ms | Trades for 1 hour |
| Range (1 day) | <1s | Daily aggregation |
| Cross-venue | <2s | All exchanges, 1 hour |

### Compression Ratios

| Data Type | Uncompressed | Compressed | Ratio |
|-----------|--------------|------------|-------|
| MBO | 1 TB | 200 GB | 5:1 |
| Trades | 100 GB | 18 GB | 5.5:1 |
| OHLCV | 10 GB | 2 GB | 5:1 |
| Funding Rates | 1 MB | 200 KB | 5:1 |
| DEX Swaps | 50 GB | 9 GB | 5.5:1 |

---

## 🎨 Sample Record Definitions

### FundingRateMsg (80 bytes)

```rust
struct FundingRateMsg {
    hd: RecordHeader,              // 16B
    ts_recv: u64,                  // 8B
    funding_rate: i64,             // 8B (1e-9 scale)
    predicted_funding_rate: i64,   // 8B
    mark_price: i64,               // 8B
    index_price: i64,              // 8B
    premium: i64,                  // 8B
    funding_interval_start: u64,   // 8B
    funding_interval_end: u64,     // 8B
    interval_duration_ms: u32,     // 4B
    sequence: u32,                 // 4B
    _reserved: [u8; 16],           // 16B
}                                  // = 80 bytes ✅
```

### DexSwapMsg (128 bytes)

```rust
struct DexSwapMsg {
    hd: RecordHeader,              // 16B
    ts_recv: u64,                  // 8B
    block_number: u64,             // 8B
    block_timestamp: u64,          // 8B
    tx_hash: [u8; 32],             // 32B
    log_index: u32,                // 4B
    amount_in: i64,                // 8B
    amount_out: i64,               // 8B
    fee: i64,                      // 8B
    price_before: i64,             // 8B
    price_after: i64,              // 8B
    pool_address: [u8; 20],        // 20B
    router_address: [u8; 20],      // 20B
    sender_hash: u64,              // 8B
    recipient_hash: u64,           // 8B
    gas_used: u32,                 // 4B
    gas_price: u32,                // 4B
    _reserved: [u8; 8],            // 8B
}                                  // = 128 bytes ✅
```

---

## 🔧 Implementation Checklist

### Phase 1: Core (2 months)

- [ ] Schema definitions (Rust structs)
- [ ] Encoder/decoder implementation
- [ ] Unit tests + benchmarks
- [ ] S3/MinIO storage setup
- [ ] Iceberg catalog integration
- [ ] CEX connectors (Binance, Coinbase)
- [ ] Kafka pipeline
- [ ] Symbol normalization service

### Phase 2: Advanced (2 months)

- [ ] DEX event indexers (Ethereum, BSC)
- [ ] Oracle integrations (Chainlink)
- [ ] Cross-exchange aggregation
- [ ] Query engine with predicate pushdown
- [ ] Vectorized operations (SIMD)
- [ ] Monitoring & alerting

### Phase 3: Production (1 month)

- [ ] Load testing (5M msg/s)
- [ ] Disaster recovery procedures
- [ ] Data quality validation
- [ ] Documentation
- [ ] API layer
- [ ] Analytics tools

---

## 📊 Cost Estimation

### Storage (1 Year)

| Tier | Volume | Cost/TB/Mo | Monthly Cost |
|------|--------|------------|--------------|
| Hot (7d) | 7 TB | $23 | $161 |
| Warm (90d) | 90 TB | $10 | $900 |
| Cold (365d) | 365 TB | $4 | $1,460 |
| **Total** | **462 TB** | - | **$2,521** |

### Compute (Monthly)

| Service | Spec | Cost |
|---------|------|------|
| Kafka cluster | 3 × r5.2xlarge | $1,200 |
| Ingestion workers | 10 × c6i.2xlarge | $1,600 |
| Query engine | 5 × r6i.4xlarge | $2,000 |
| Metadata DB | RDS r6g.2xlarge | $600 |
| **Total Compute** | | **$5,400** |

### **Grand Total**: ~$8,000/month for full production deployment

---

## 🎯 Key Success Metrics

| Metric | Target | Critical? |
|--------|--------|-----------|
| Ingestion latency | <500ms p99 | ✅ |
| Storage cost | <$10K/mo | ✅ |
| Query speed (1h range) | <100ms | ✅ |
| Data completeness | >99.9% | ✅ |
| Uptime | 99.99% | ✅ |
| Compression ratio | >4:1 | ⚠️ |
| False positive rate | <0.01% | ⚠️ |

---

## 📋 Status & Implementation

**Current Status:** 🔵 **Proposal / Design Phase**

This document presents a comprehensive proposal for adapting DBN to cryptocurrency markets. Implementation is currently in the **planning and design phase**.

**By Component:**
- **Core Infrastructure** (buf tooling, hierarchical schemas): ✅ Implemented
- **Crypto-specific Schemas** (FundingRateMsg, LiquidationMsg, etc.): 🔵 Proposed
- **Migration Guides**: ✅ Complete
- **Reference Implementation**: ⏳ In Progress
- **Production Deployment**: 📅 Planned

**Next Steps:**
1. Design review and stakeholder feedback
2. Create reference implementation with selected schemas
3. Validate against real cryptocurrency market data
4. Iterate on schema design based on testing
5. Production rollout (phased approach)

For implementation tracking and detailed timeline, see [IMPLEMENTATION_SUMMARY.md](IMPLEMENTATION_SUMMARY.md).

---

## 🚀 Getting Started

### 1. Review Base DBN Spec
Read `DBN_SCHEMA_SPECIFICATION.md` to understand the foundation.

### 2. Understand Crypto Requirements
Review `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md` for detailed analysis.

### 3. Prototype
Start with 3 core schemas:
- FundingRateMsg
- LiquidationMsg
- DexSwapMsg

### 4. Test with Real Data
- Connect to Binance WebSocket
- Index Uniswap V3 events
- Validate encoding/decoding

### 5. Deploy Pilot
- 7 days of data
- Single exchange
- Query benchmarks

---

## 📚 Reference Documents

1. **DBN_SCHEMA_SPECIFICATION.md** - Base DBN v3 spec
2. **CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md** - Full crypto adaptation (this doc's parent)
3. **CRYPTO_DBN_QUICK_REFERENCE.md** - This document

---

## 🤝 Contributing

For questions, issues, or contributions:
1. Open an issue on GitHub
2. Submit a pull request
3. Join the discussion in Discord/Slack

---

**Last Updated**: 2025-10-15
**Maintainer**: Crypto-DBN Working Group
**License**: Apache 2.0
