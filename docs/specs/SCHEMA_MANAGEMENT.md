# DBN Schema Management Guide

This document describes how schemas are managed in the DBN project using Protocol Buffers and the Buf CLI.

## Overview

The DBN project uses a multi-format schema approach:

1. **Rust structs** (`rust/dbn/src/record.rs`) - Primary source of truth for binary format
2. **Protocol Buffers** (`docs/specs/proto/`) - Language-agnostic schema definitions
3. **JSON Schemas** (`docs/specs/jsonschema/`) - Validation schemas for JSON format

## Directory Structure

```
dbn/
├── buf.yaml                    # Buf workspace configuration
├── buf.gen.yaml                # Code generation configuration
├── docs/
│   └── specs/
│       ├── proto/              # Protobuf schema definitions
│       │   ├── dbn.proto      # Main DBN v3 schema
│       │   └── README.md
│       ├── jsonschema/         # JSON schema definitions
│       │   ├── *.schema.json
│       │   └── README.md
│       └── examples/           # Example messages
└── rust/
    └── dbn/
        └── src/
            └── record.rs       # Rust struct definitions
```

## Schema Formats

### Binary Format (Rust)

The canonical DBN binary format is defined in Rust with `#[repr(C)]` structs:

```rust
#[repr(C)]
pub struct MboMsg {
    pub hd: RecordHeader,
    pub order_id: u64,
    pub price: i64,
    // ...
}
```

**Key characteristics:**
- 8-byte alignment
- Fixed-width structs
- Little-endian byte order
- No padding between fields

### Protocol Buffers

Protobuf schemas provide language-agnostic definitions:

```protobuf
message MboMsg {
  RecordHeader hd = 1;
  uint64 order_id = 2;
  int64 price = 3;
  // ...
}
```

**Use cases:**
- Code generation for multiple languages
- Schema documentation
- Type-safe APIs
- JSON schema generation

### JSON Schema

JSON schemas enable validation of JSON-formatted messages:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "order_id": { "type": "string" }
  }
}
```

**Use cases:**
- Runtime validation
- API documentation
- IDE autocomplete
- Form generation

## Buf CLI Workflow

### Installation

```bash
# Install buf CLI
brew install bufbuild/buf/buf

# Or download from https://github.com/bufbuild/buf/releases
```

### Linting

Check schema files for style and best practices:

```bash
buf lint
```

Configuration in `buf.yaml`:

```yaml
lint:
  use:
    - BASIC
  except:
    - PACKAGE_DIRECTORY_MATCH
    - COMMENT_FIELD
```

### Formatting

Auto-format protobuf files:

```bash
buf format -w
```

### Breaking Change Detection

Check for breaking changes against a baseline:

```bash
# Against main branch
buf breaking --against '.git#branch=main'

# Against a tag
buf breaking --against '.git#tag=v0.42.0'
```

### Code Generation

Generate code from protobuf schemas:

```bash
buf generate
```

This will generate:
- **Go code** → `gen/go/`
- **Python code** → `gen/python/`
- **TypeScript** → `gen/ts/`
- **JSON schemas** → `docs/specs/jsonschema/`

Configuration in `buf.gen.yaml`:

```yaml
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
  - remote: buf.build/protocolbuffers/python
    out: gen/python
```

## Schema Versioning

### Version History

DBN uses semantic versioning for its schema:

- **v3** (current) - Multi-leg strategies, 64-bit StatMsg quantity
- **v2** - Expanded symbols (71 bytes), updated InstrumentDefMsg
- **v1** - Initial release

### Compatibility Rules

1. **Never break existing parsers**
   - Keep existing field numbers
   - Don't change field types
   - Don't remove required fields

2. **Additive changes only**
   - Add new optional fields
   - Add new message types
   - Add new enum values (append only)

3. **Version bumps**
   - **Major version** - Breaking changes to binary format
   - **Minor version** - New messages or optional fields
   - **Patch version** - Documentation or non-schema changes

### Migration Path

When schema changes are needed:

1. Create new versioned namespace (e.g., `databento.dbn.v4`)
2. Implement conversion functions in Rust
3. Update protobuf schemas
4. Regenerate all derived schemas
5. Document changes in CHANGELOG.md

## Working with Schemas

### Adding a New Message Type

1. **Define in Rust** (`rust/dbn/src/record.rs`):

```rust
#[repr(C)]
#[derive(Clone, CsvSerialize, JsonSerialize)]
pub struct MyNewMsg {
    pub hd: RecordHeader,
    pub my_field: u64,
}
```

2. **Add to protobuf** (`docs/specs/proto/dbn.proto`):

```protobuf
message MyNewMsg {
  RecordHeader hd = 1;
  uint64 my_field = 2;
}
```

3. **Update RType enum**:

```protobuf
enum RType {
  // ...
  RTYPE_MY_NEW = 250;  // Choose unused value
}
```

4. **Lint and format**:

```bash
buf lint
buf format -w
```

5. **Generate schemas**:

```bash
buf generate
```

6. **Test**:
   - Unit tests in Rust
   - Integration tests with protobuf
   - Validation tests with JSON schema

### Modifying an Existing Message

1. **Check if change is breaking**:
   - Changing field types → **BREAKING**
   - Removing fields → **BREAKING**
   - Adding optional fields → **SAFE**

2. **Update all schema formats**:
   - Rust struct
   - Protobuf message
   - JSON schema (if manual)

3. **Run breaking change detection**:

```bash
buf breaking --against '.git#branch=main'
```

4. **Update documentation**:
   - Update field comments
   - Update README files
   - Update DBN_SCHEMA_SPECIFICATION.md

### Validating Consistency

Ensure schemas stay in sync:

```bash
# Rust build checks
cd rust/dbn
cargo test

# Protobuf lint
buf lint

# Generated code builds
buf generate
cd gen/python && python -m pytest
cd gen/go && go test ./...
```

## Best Practices

### Schema Design

1. **Use fixed-size integers** for known ranges
   - `uint32` for IDs, not `uint64` unless needed
   - `int64` for prices (fixed-point with 1e-9 scale)

2. **Group related fields**
   - Keep common header (`RecordHeader`) first
   - Group timestamps together
   - Group price fields together

3. **Document everything**
   - Field-level comments
   - Enum value meanings
   - Unit specifications (e.g., "UNIX nanoseconds")

4. **Preserve alignment**
   - 8-byte alignment for all structs
   - Use reserved fields if needed for padding

### Code Generation

1. **Version control generated code** (optional)
   - Pros: Easier to review changes
   - Cons: Larger repository

2. **Use consistent naming**
   - Snake_case for Rust
   - camelCase for TypeScript
   - PascalCase for Go/C#

3. **Test generated code**
   - Round-trip serialization
   - Cross-language compatibility
   - Edge cases (max values, nulls, etc.)

### Documentation

1. **Keep specs up-to-date**
   - DBN_SCHEMA_SPECIFICATION.md
   - README files in each directory
   - Inline protobuf comments

2. **Provide examples**
   - Sample messages in `docs/specs/examples/`
   - Usage examples in README
   - Test fixtures

## Troubleshooting

### Common Issues

**"package directory mismatch"**
```bash
# Fix: Disable the check in buf.yaml
lint:
  except:
    - PACKAGE_DIRECTORY_MATCH
```

**"enum value alias not allowed"**
```bash
# Fix: Remove conflicting enum values
# Protobuf doesn't allow duplicate values without allow_alias
```

**"field not documented"**
```bash
# Fix: Add comments or disable the check
lint:
  except:
    - COMMENT_FIELD
```

### Debugging

```bash
# Verbose lint output
buf lint --error-format=json

# Check specific file
buf lint docs/specs/proto/dbn.proto

# Diff against baseline
buf breaking --against '.git#branch=main' --error-format=json
```

## References

- [Buf CLI Documentation](https://buf.build/docs)
- [Protocol Buffers Guide](https://protobuf.dev/)
- [JSON Schema Specification](https://json-schema.org/)
- [DBN Binary Format](DBN_SCHEMA_SPECIFICATION.md)
- [Databento Docs](https://databento.com/docs)

## Contributing

When contributing schema changes:

1. Fork and create a feature branch
2. Make changes to all relevant schema formats
3. Run `buf lint` and `buf format -w`
4. Run `buf breaking` to check compatibility
5. Update tests and documentation
6. Submit PR with clear description of changes

For questions, open an issue on GitHub or contact the Databento team.
