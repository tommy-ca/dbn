# Crypto DBN – Hierarchical Proto Layout Proposal

**Date:** 2025-11-19  
**Status:** 🔵 Draft / Design  
**Scope:** Proto package layout and mapping for crypto-focused DBN records (RTypes 0xD0–0xE1)

This document proposes how to represent the crypto-focused DBN extensions in the
hierarchical protobuf tree, building on the existing v3 layout.

It is a **design document only** – no `.proto` files or enums are assumed to be
final until:

1. The canonical binary spec (`01_canonical_spec/DBN_SCHEMA_SPECIFICATION.md`)
   is updated with the crypto structs, and
2. The Rust `#[repr(C)]` definitions are finalized (currently in
   `00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md`).

For current implementation status and mapping of the core v3 types, see:
`DBN_PROTO_SCHEMA_MAPPING.md`.

---

## 1. Design Goals

- **Stay aligned with existing v3 layout**  
  Reuse the `databento.dbn.v3` hierarchy and extend under `messages/crypto/`,
  rather than introducing a new major version directory prematurely.

- **Group messages by domain, not by RType**  
  Keep files focused on related concerns (derivatives, DEX, infra) so consumers
  can import only what they need.

- **Mirror binary structs 1:1 where possible**  
  Field names and types should follow the Rust structs in
  `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md` with the usual proto differences
  (fixed-length arrays → `repeated`, `c_char` arrays → `string` or `bytes`).

- **Keep crypto extensions clearly optional**  
  Treat crypto messages and enums as additive; they should not affect existing
  DBNv3 consumers unless explicitly imported.

---

## 2. Proposed Package and File Layout

Extend the existing v3 directory:

```text
proto/databento/dbn/v3/
  ├── common/
  ├── enums/
  ├── messages/
  │   ├── market_data/
  │   ├── aggregated/
  │   ├── reference/
  │   ├── system/
  │   └── crypto/           # ← NEW (proposed)
  └── metadata/
```

Within `messages/crypto/`:

```text
messages/crypto/
  ├── derivatives.proto   # Funding/mark/index, basis, liquidation
  ├── dex.proto           # DexSwap, DexPoolState
  ├── oracle.proto        # OraclePrice, possibly oracle-specific enums
  ├── spreads.proto       # CrossRate, StablecoinPeg
  └── infra.proto         # GasPrice, BlockInfo, SmartContractEvent, InsuranceFund, Delisting, TokenMigration
```

### 2.1 Package Names

Each file shares the same base package, consistent with other message groups:

```protobuf
package databento.dbn.v3.messages.crypto;
```

`option go_package` / `java_package` / etc. follow the conventions used by the
existing `messages/*` packages.

---

## 3. Mapping for P0 Derivatives Messages

This section sketches how the P0 derivatives structs from
`CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md` would map into proto messages. These are
**illustrative** and should be treated as draft definitions.

### 3.1 FundingRateMsg (RType 0xD0)

Canonical struct:
- `docs/specs/00_requirements/CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:424`

Key traits:
- Fixed size: 80 bytes, 8-byte aligned
- Uses `RecordHeader` and a set of `i64` price fields
- Contains interval start/end and an `interval_duration_ms`

Proposed proto (shape only, not final field numbers):

```protobuf
message FundingRateMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 funding_rate = 3;
  sfixed64 predicted_funding_rate = 4;
  sfixed64 mark_price = 5;
  sfixed64 index_price = 6;
  sfixed64 premium = 7;

  fixed64 funding_interval_start = 8;
  fixed64 funding_interval_end = 9;

  fixed32 interval_duration_ms = 10;
  fixed32 sequence = 11;
}
```

Notes:
- `_reserved` bytes from the binary struct are intentionally omitted.
- All price-like fields use `sfixed64` with the same 1e-9 scale as core DBN.

### 3.2 LiquidationMsg (RType 0xD1)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:462`

Key traits:
- 96 bytes, 8-byte aligned
- Mix of prices, account hashes, a `side` char, and small enums

Proto shape:

```protobuf
message LiquidationMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 liquidation_price = 3;
  sfixed64 quantity = 4;
  sfixed64 bankrupt_price = 5;
  sfixed64 mark_price = 6;
  sfixed64 unrealized_pnl = 7;

  fixed64 account_id = 8;
  fixed64 liquidator_id = 9;

  // Side could reuse enums.Side if semantics match; otherwise a new enum.
  databento.dbn.v3.enums.Side side = 10;

  uint32 liquidation_type = 11; // 0=partial, 1=full, 2=ADL
  fixed32 sequence = 12;

  sfixed64 position_value = 13;
}
```

Open questions:
- Whether `liquidation_type` should become a dedicated enum.
- Whether `side` should reuse `enums.Side` or use a separate PerpPositionSide.

### 3.3 MarkPriceMsg (RType 0xD2)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:502`

Proto shape:

```protobuf
message MarkPriceMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 mark_price = 3;
  sfixed64 index_price = 4;
  sfixed64 last_price = 5;
  sfixed64 premium = 6;

  uint32 update_reason = 7; // 1=periodic, 2=volatility, 3=funding
  fixed32 sequence = 8;
}
```

### 3.4 IndexPriceMsg (RType 0xD3)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:528`

Key traits:
- Component prices and weights as fixed-size arrays (up to 5 components).

Proto shape:

```protobuf
message IndexPriceMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 index_price = 3;

  repeated sfixed64 component_prices = 4;  // length ≤ 5
  repeated sfixed32 component_weights = 5; // 1e-6 scale, sum to 1e6

  uint32 num_components = 6;
  fixed32 sequence = 7;
}
```

Notes:
- Fixed-size arrays in the binary struct (`[i64; 5]`, `[i32; 5]`) become
  bounded `repeated` fields in proto; constraints belong in documentation.

---

## 4. Mapping for P0/P1 DEX and Oracle Messages

### 4.1 DexSwapMsg (RType 0xD4)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:560`

Key traits:
- On-chain metadata (block, tx hash, log index)
- Swap amounts, price before/after
- Pool/router addresses and gas metrics

Proto shape (illustrative):

```protobuf
message DexSwapMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  fixed64 block_number = 3;
  fixed64 block_timestamp = 4;
  bytes tx_hash = 5;       // 32 bytes
  fixed32 log_index = 6;

  sfixed64 amount_in = 7;
  sfixed64 amount_out = 8;
  sfixed64 fee = 9;
  sfixed64 price_before = 10;
  sfixed64 price_after = 11;

  bytes pool_address = 12;   // 20 bytes
  bytes router_address = 13; // 20 bytes

  fixed64 sender_hash = 14;
  fixed64 recipient_hash = 15;

  fixed32 gas_used = 16;
  fixed32 gas_price = 17; // Gwei or 1e-9 scale, to be decided
}
```

Design choice:
- Use `bytes` for on-chain hashes/addresses instead of `string` to preserve
  fixed length and avoid encoding issues.

### 4.2 DexPoolStateMsg (RType 0xD5)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:608`

Proto shape (sketch):

```protobuf
message DexPoolStateMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  fixed64 block_number = 3;
  fixed64 block_timestamp = 4;

  sfixed64 reserve0 = 5;
  sfixed64 reserve1 = 6;
  sfixed64 k_value = 7;
  sfixed64 sqrt_price_x96 = 8;
  sfixed64 liquidity = 9;
  sfixed32 tick = 10;

  bytes pool_address = 11;
  uint32 pool_type = 12;  // 1=UniV2, 2=UniV3, 3=Curve, etc.
  fixed32 fee_tier = 13; // basis points

  sfixed64 volume_24h_token0 = 14;
  sfixed64 volume_24h_token1 = 15;
  sfixed64 fee_24h = 16;
  sfixed64 tvl_usd = 17;

  fixed32 sequence = 18;
}
```

### 4.3 OraclePriceMsg (RType 0xD6)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:658`

Proto shape:

```protobuf
message OraclePriceMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  fixed64 block_number = 3;
  fixed64 block_timestamp = 4;

  sfixed64 price = 5;
  sfixed64 confidence_interval = 6;
  uint32 num_sources = 7;
  uint32 oracle_type = 8;      // 1=Chainlink, 2=Pyth, ...

  bytes oracle_address = 9;    // 20 bytes
  fixed64 round_id = 10;
  fixed32 sequence = 11;
}
```

---

## 5. Spreads and Peg Monitoring

### 5.1 CrossRateMsg (RType 0xD8)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:691`

Proto sketch:

```protobuf
message CrossRateMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 exchange1_price = 3;
  sfixed64 exchange2_price = 4;
  sfixed32 spread_bps = 5;
  sfixed64 spread_pct = 6;

  uint32 exchange1_id = 7;
  uint32 exchange2_id = 8;

  sfixed64 arb_opportunity = 9;
  sfixed64 transfer_cost = 10;
  sfixed64 net_profit = 11;

  fixed32 sequence = 12;
}
```

### 5.2 BasisMsg (RType 0xD9)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:725`

Proto sketch:

```protobuf
message BasisMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 spot_price = 3;
  sfixed64 futures_price = 4;
  sfixed64 basis = 5;
  sfixed64 basis_pct = 6;
  sfixed64 annualized_basis = 7;

  fixed32 spot_instrument_id = 8;
  fixed32 futures_instrument_id = 9;

  fixed32 sequence = 10;
}
```

### 5.3 StablecoinPegMsg (RType 0xDA)

Canonical struct:
- `CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md:755`

Proto sketch:

```protobuf
message StablecoinPegMsg {
  databento.dbn.v3.common.RecordHeader hd = 1;
  fixed64 ts_recv = 2;

  sfixed64 price_usd = 3;
  sfixed32 deviation_bps = 4;
  sfixed64 deviation_pct = 5;

  sfixed64 total_supply = 6;
  sfixed64 market_cap = 7;

  uint32 exchange_id = 8;
  sfixed64 volume_24h = 9;

  uint32 peg_confidence = 10;
  uint32 alert_level = 11;

  fixed32 sequence = 12;
}
```

---

## 6. Infra and Remaining Records

The remaining RTypes (0xDB–0xE1) are summarized in
`CRYPTO_DBN_REQUIREMENTS_AND_SPECS.md` with varying levels of detail:

- `GasPriceMsg` (0xDB)
- `BlockInfoMsg` (0xDC)
- `SmartContractEventMsg` (0xDD)
- `InsuranceFundMsg` (0xDE)
- `ClawbackMsg` (0xDF)
- `DelistingMsg` (0xE0)
- `TokenMigrationMsg` (0xE1)

For these, the proposal is:

- Start with **one infra file** (`infra.proto`) containing simple, direct
  translations of the existing Rust-like struct sketches.
- Introduce shared helper messages in `common/` **only when multiple records
  need them** (e.g., a common `BlockHeader` or `OnChainRef`).
- Keep enums local unless clearly cross-cutting; promote to `enums/` only when
  reused across multiple messages.

---

## 7. Open Questions and Next Steps

Open questions to resolve before turning this proposal into `.proto` files:

- **Versioning:**  
  Confirm whether these records should remain under `dbn.v3` (as the backlog
  currently suggests) or whether a dedicated `v4` directory should be created
  once crypto schemas stabilize.

- **Enum placement:**  
  Decide which crypto-specific enums belong in `enums/` vs. being message-local
  (e.g., `VenueType`, `ProductType`, `LiquidationType`, `OracleType`).

- **Address/hash representation:**  
  Finalize whether all on-chain identifiers are `bytes` (fixed length) or
  hex-encoded `string`s, and how to document their exact lengths.

- **Metadata extensions:**  
  Align the `CryptoMetadata` struct in the requirements doc with the existing
  `Metadata` proto and decide whether it becomes:
  - a new message in `metadata/`, or
  - a higher-level composition outside the DBN wire format.

Recommended next steps:

1. Review this layout proposal with the schema owners.
2. Decide on versioning (`v3` vs `v4`) and package paths.
3. Promote the agreed subset of messages (likely P0 + selected P1) into actual
   `.proto` files under `messages/crypto/`.
4. Update:
   - `DBN_SCHEMA_SPECIFICATION.md` with the new binary structs,
   - `enums/rtype.proto` and related enums,
   - `DBN_PROTO_SCHEMA_MAPPING.md` with crypto coverage.
5. Regenerate code and JSON schemas via `buf generate` and validate against
   real crypto data samples.

