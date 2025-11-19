# DBN Examples & Sample Data

This directory contains example messages and sample data files demonstrating DBN format usage.

## Planned Examples

### Sample Messages (Binary)

- `mbo_example.dbn` - Market-by-order sample data (80 bytes per record)
- `trade_example.dbn` - Trade execution sample (80 bytes per record)
- `mbp_example.dbn` - Market-by-price sample (104-424 bytes per record)
- `ohlcv_example.dbn` - OHLCV candlestick sample (56 bytes per record)

### Sample Messages (JSON)

- `mbo_sample.json` - MBO message in JSON format
- `trade_sample.json` - Trade message in JSON format
- `metadata_sample.json` - Metadata header example

### Code Examples

- `decode_example.rs` - Rust decoding example
- `decode_example.py` - Python decoding example
- `decode_example.go` - Go decoding example

### Crypto-Specific Examples

- `funding_rate_example.dbn` - Funding rate message sample
- `liquidation_example.dbn` - Liquidation event sample
- `dex_swap_example.dbn` - DEX swap event sample

## Structure

```
examples/
├── README.md                    # This file
├── binary/                      # Binary .dbn files
│   ├── mbo_example.dbn
│   ├── trade_example.dbn
│   └── ...
├── json/                        # JSON message samples
│   ├── mbo_sample.json
│   ├── trade_sample.json
│   └── ...
├── code/                        # Working code examples
│   ├── decode_example.rs
│   ├── decode_example.py
│   ├── decode_example.go
│   └── ...
└── crypto/                      # Crypto-specific examples
    ├── funding_rate_example.dbn
    ├── liquidation_example.dbn
    └── ...
```

## Usage

Each example file includes:
1. **Format specification** - Message type and structure
2. **Hex dump** - Raw binary representation
3. **Decoded values** - Field-by-field breakdown
4. **Code sample** - How to read/parse in each language

## Coming Soon

These examples are placeholder stubs and will be populated with actual sample data and working code examples in future releases.

## Notes

- All .dbn files are gzip-compressed and can be read with standard DBN libraries
- JSON files conform to JSON Schema definitions in `gen/jsonschema/`
- Code examples assume dbn library is available in your language of choice
- See respective language documentation for compilation/execution

---

**Last Updated**: 2025-11-13
**Status**: 🔵 Placeholder structure, awaiting sample data
