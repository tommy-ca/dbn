# DBN Canonical vs Proto Schema Mapping

**Date:** 2025-11-19  
**Status:** 🟢 Core v3 in sync, 🔵 Crypto extensions pending

This document cross-checks the canonical DBN binary specification against the
hierarchical protobuf schemas under `proto/databento/dbn/v3/`. It has two goals:

- Confirm coverage and parity for all **DBNv3 core schemas**.
- Highlight **known gaps** for the proposed **crypto extensions** and where they
  are intentionally not yet represented in protobuf.

Canonical reference:
- `docs/specs/01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md`

Protobuf reference:
- `proto/databento/dbn/v3/README.md`
- `proto/databento/dbn/v3/**.proto`

Crypto requirements:
- `docs/specs/00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`
- `docs/specs/00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md`

---

## 1. Core v3 Schema Coverage

The table below maps the canonical DBN v3 message structs and enums to their
hierarchical protobuf counterparts. All listed core v3 messages are covered by
the hierarchical proto package and are kept binary-compatible.

### 1.1 Messages and Shared Types

| Canonical type (DBN spec)        | RType / Schema enum      | Proto message                          | Proto file path                                            | Status  |
|----------------------------------|--------------------------|----------------------------------------|------------------------------------------------------------|---------|
| `RecordHeader`                   | `RType` (all record types) | `RecordHeader`                        | `databento/dbn/v3/common/header.proto`                     | In sync |
| `BidAskPair`                     | used by MBP/BBO/CBBO     | `BidAskPair`                          | `databento/dbn/v3/common/price_levels.proto`               | In sync |
| `ConsolidatedBidAskPair`        | used by CMBP1/CBBO       | `ConsolidatedBidAskPair`              | `databento/dbn/v3/common/price_levels.proto`               | In sync |
| `MboMsg`                         | `RType::MBO` (0xA0)      | `MboMsg`                              | `databento/dbn/v3/messages/market_data/mbo.proto`          | In sync |
| `TradeMsg`                       | `RType::MBP_0` (0x00)    | `TradeMsg`                            | `databento/dbn/v3/messages/market_data/trade.proto`        | In sync |
| `Mbp1Msg`                        | `RType::MBP_1` (0x01)    | `Mbp1Msg`                             | `databento/dbn/v3/messages/market_data/mbp.proto`          | In sync |
| `Mbp10Msg`                       | `RType::MBP_10` (0x0A)   | `Mbp10Msg`                            | `databento/dbn/v3/messages/market_data/mbp.proto`          | In sync |
| `OhlcvMsg`                       | `RType::OHLCV_*` (0x20–0x24) | `OhlcvMsg`                        | `databento/dbn/v3/messages/aggregated/ohlcv.proto`         | In sync |
| `BboMsg`                         | `RType::BBO_1S/1M`       | `BboMsg`                              | `databento/dbn/v3/messages/aggregated/bbo.proto`           | In sync |
| `CbboMsg`                        | `RType::CBBO_1S/1M`      | `CbboMsg`                             | `databento/dbn/v3/messages/aggregated/cbbo.proto`          | In sync |
| `Cmbp1Msg`                       | `RType::CMBP_1/TCBBO`    | `Cmbp1Msg`                            | `databento/dbn/v3/messages/aggregated/cbbo.proto`          | In sync |
| `InstrumentDefMsg`               | `RType::INSTRUMENT_DEF`  | `InstrumentDefMsg`                    | `databento/dbn/v3/messages/reference/instrument_def.proto` | In sync |
| `StatMsg`                        | `RType::STATISTICS`      | `StatMsg`                             | `databento/dbn/v3/messages/reference/statistics.proto`     | In sync |
| `StatusMsg`                      | `RType::STATUS`          | `StatusMsg`                           | `databento/dbn/v3/messages/reference/status.proto`         | In sync |
| `ImbalanceMsg`                   | `RType::IMBALANCE`       | `ImbalanceMsg`                        | `databento/dbn/v3/messages/reference/imbalance.proto`      | In sync |
| `ErrorMsg`                       | `RType::ERROR`           | `ErrorMsg`                            | `databento/dbn/v3/messages/system/error.proto`             | In sync |
| `SystemMsg`                      | `RType::SYSTEM`          | `SystemMsg`                           | `databento/dbn/v3/messages/system/system.proto`            | In sync |
| `SymbolMappingMsg` (system)      | `RType::SYMBOL_MAPPING`  | `SymbolMappingMsg`                    | `databento/dbn/v3/messages/system/symbol_mapping.proto`    | In sync |
| `Metadata`                       | metadata header          | `Metadata`                            | `databento/dbn/v3/metadata/metadata.proto`                 | In sync (representation differs) |
| `SymbolMapping` (metadata)       | metadata mapping entries | `SymbolMapping`                       | `databento/dbn/v3/metadata/metadata.proto`                 | In sync |
| `MappingInterval`                | metadata mapping entries | `MappingInterval`                     | `databento/dbn/v3/metadata/metadata.proto`                 | In sync |

### 1.2 Enumerations

The following enums from the canonical spec are represented in the hierarchical
protos under `databento/dbn/v3/enums/`:

| Canonical enum (DBN spec) | Proto enum              | Proto file                                   | Notes                        |
|---------------------------|-------------------------|----------------------------------------------|------------------------------|
| `RType`                   | `RType`                 | `enums/rtype.proto`                          | Core values 0x00–0xC4 only   |
| `Schema`                  | `Schema`                | `enums/schema.proto`                         | 20 schemas (Mbo, Mbp1, …)    |
| `SType`                   | `SType`                 | `enums/symbology.proto`                      | Symbology types 0–12         |
| `Side`                    | `Side`                  | `enums/side.proto`                           | Includes INVALID/NONE values |
| `Action`                  | `Action`                | `enums/action.proto`                         | Includes INVALID/NONE values |
| `InstrumentClass`         | `InstrumentClass`       | `enums/instrument.proto`                     | Mirrors binary `c_char`      |
| `MatchAlgorithm`          | `MatchAlgorithm`        | `enums/instrument.proto`                     | Mirrors binary `c_char`      |

For these core v3 enums, the numeric values and semantics match the canonical
specification. Protobuf uses decimal literals but comments preserve the original
hex values where applicable.

---

## 2. Crypto Extensions (RTypes 0xD0–0xE1)

The canonical spec reserves a range of `RType` values for crypto-focused
extensions, with proposed record names and purposes documented in:

- `01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md` (Reserved Crypto Extensions)
- `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`
- `00_requirements/CRYPTO_DBN_QUICK_REFERENCE.md`

These include, for example:

| RType (hex) | Planned record          | Purpose (summary)                         |
|-------------|-------------------------|-------------------------------------------|
| 0xD0        | `FundingRateMsg`        | Perpetual funding rates and predictions   |
| 0xD1        | `LiquidationMsg`        | Forced liquidations / ADL events          |
| 0xD2        | `MarkPriceMsg`          | Fair price feed for perpetuals            |
| 0xD3        | `IndexPriceMsg`         | Multi-venue spot index                    |
| 0xD4        | `DexSwapMsg`            | AMM swap executions                       |
| 0xD5        | `DexPoolStateMsg`       | AMM pool snapshots                        |
| 0xD6        | `OraclePriceMsg`        | Oracle price updates                      |
| 0xD8–0xE1   | Cross-rate / peg / infra| Spreads, pegs, gas, blocks, migrations    |

**Current implementation status (as of 2025-11-19):**

- Canonical binary layouts for the main crypto records (e.g., `FundingRateMsg`,
  `LiquidationMsg`, `DexSwapMsg`) are now documented in
  `01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md` under **Crypto Extensions**.
- No Rust `#[repr(C)]` structs for these records exist under `rust/dbn/src/`
  yet.
- No protobuf messages exist under `proto/databento/dbn/v3/messages/**` for
  these RTypes yet.
- `enums/rtype.proto` intentionally omits the 0xD0–0xE1 values until the Rust
  implementation and Buf APIs are ready.
- `SCHEMA_MANAGEMENT.md` tracks this work under the *Crypto Schema Backlog*
  (e.g., creation of `messages/crypto/` and shared crypto primitives).

These crypto records are therefore **design-only** at this stage. They are
intentionally out of scope for the current hierarchical protobuf package, which
covers the **existing DBNv3 core schemas** only.

---

## 3. Representation Differences (Binary vs Proto)

Some differences between the binary DBN representation and the protobuf schemas
are intentional and not considered gaps:

- **Fixed-length strings vs `string`:**  
  Binary uses fixed-length `c_char` arrays (e.g., `raw_symbol`, `asset`); proto
  uses variable-length `string` fields. Size constraints remain documented in
  comments and the canonical spec.

- **Fixed-size arrays vs `repeated`:**  
  Binary structs use fixed-size arrays (e.g., `[BidAskPair; 10]`); proto uses
  `repeated` fields (e.g., `repeated BidAskPair levels`). The logical content is
  equivalent; protobuf does not encode the fixed array length in the type.

- **Metadata enums vs strings:**  
  Binary `Metadata` uses typed enums (`Schema`, `SType`) for some fields,
  whereas `metadata.proto` currently models `schema`, `stype_in`, and
  `stype_out` as strings. These should contain the string forms corresponding to
  the canonical enums.

- **Padding / reserved bytes:**  
  Binary structs include explicit padding and reserved fields for alignment and
  future expansion; protobuf omits these except where alignment is documented in
  comments.

These differences are already described in `proto/README.md` and do not imply
schema drift between DBN and the hierarchical protos.

---

## 4. Where to Add New Crypto Schemas

When promoting any of the crypto RTypes (0xD0–0xE1) from design to
implementation, the expected workflow is:

1. Finalize the binary `#[repr(C)]` struct in the canonical spec and Rust
   implementation (`rust/dbn/src/record.rs`).
2. Extend `enums/rtype.proto` (and `enums/schema.proto` / `enums/instrument.proto`
   if needed) in an append-only fashion.
3. Add new protobuf messages under a dedicated package, e.g.:  
   `proto/databento/dbn/v3/messages/crypto/`.
4. Update:
   - `proto/databento/dbn/v3/README.md`
   - `docs/specs/02_schema_implementation/SCHEMA_MANAGEMENT.md`
   - This mapping document
5. Regenerate code and JSON schemas with `buf generate` and validate with the
   examples/fixtures suite.

For detailed backlog items and acceptance criteria, see the *Crypto Schema
Backlog* section in `SCHEMA_MANAGEMENT.md` and the crypto requirements docs.
