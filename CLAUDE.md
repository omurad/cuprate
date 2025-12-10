# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About Cuprate

Cuprate is an alternative Monero node implementation written in Rust. It validates Monero consensus rules independently to provide security and redundancy for the Monero network.

## Development Commands

### Building
```bash
# Build all workspace crates
cargo build --all-features --all-targets --workspace

# Build the main binary (cuprated)
cargo build -p cuprated

# Build a specific crate
cargo build -p cuprate-blockchain

# Build with release optimizations
cargo build --release

# Build documentation
cargo doc --all-features --no-deps

# Build documentation with warnings as errors (used in CI)
RUSTDOCFLAGS='-D warnings' cargo doc --workspace --all-features

# Build and open docs for a specific crate
cargo doc --open --package cuprate-blockchain
```

### Testing
```bash
# Run all tests
cargo test --all-features --workspace

# Test both database backends (heed and redb)
cargo test --all-features
cargo test --package cuprate-blockchain --no-default-features --features redb

# Run tests for a specific crate
cargo test -p cuprate-database

# Run a specific test
cargo test test_name

# Run tests with specific features
cargo test --package cuprate-blockchain --features redb
```

**Important:** Some tests require a `monerod` binary (the official Monero daemon) at the repository root. Download from https://www.getmonero.org/downloads/. Tests will be skipped if the binary is not found.

### Linting and Formatting
```bash
# Format code (required before committing)
cargo fmt --all

# Check formatting without modifying files
cargo fmt --all --check

# Run clippy (required before committing)
cargo clippy --all-features --all-targets --workspace -- -D warnings

# Fix typos automatically
typos -w

# Check for typos
typos
```

Note: `typos` can be installed with: `cargo install typos-cli`

### Feature Checking
```bash
# Install cargo-hack
cargo install cargo-hack --locked

# Check that crates build with different feature combinations
cargo hack --workspace check --feature-powerset --no-dev-deps
```

## Architecture Overview

### High-Level Structure

Cuprate follows a modular architecture with workspace crates organized by functional domain:

- **`binaries/cuprated/`** - The main Cuprate node binary
- **`consensus/`** - Consensus rules and verification logic
  - `consensus/rules/` - Raw consensus rules (flexible, low-level)
  - `consensus/` - High-level tower::Service interface for block/tx verification
  - `consensus/context/` - Blockchain context service
  - `consensus/fast-sync/` - Fast sync mechanism
- **`storage/`** - Database layer and persistence
  - `storage/database/` - Database abstraction over heed (LMDB) and redb
  - `storage/blockchain/` - Blockchain database (blocks, transactions, outputs)
  - `storage/txpool/` - Transaction pool database
  - `storage/service/` - Shared service infrastructure for database operations
- **`p2p/`** - Peer-to-peer networking
  - `p2p/p2p-core/` - Core P2P protocol
  - `p2p/p2p/` - High-level P2P service
  - `p2p/address-book/` - Peer address management
  - `p2p/dandelion-tower/` - Dandelion++ transaction broadcasting
- **`net/`** - Low-level network protocols
  - `net/epee-encoding/` - Monero's EPEE serialization format
  - `net/levin/` - Levin protocol (Monero's P2P protocol)
  - `net/wire/` - Monero wire format types
- **`rpc/`** - RPC interface
  - `rpc/types/` - RPC request/response types
  - `rpc/interface/` - RPC service interface
  - `rpc/json-rpc/` - JSON-RPC protocol handling
- **`types/`** - Common types used across Cuprate
- **`helper/`** - Utility functions and helpers
- **`constants/`** - Network and protocol constants
- **`test-utils/`** - Testing utilities and monerod integration

### Key Architectural Patterns

#### tower::Service Pattern
Cuprate extensively uses `tower::Service` for asynchronous component interfaces. Major services include:

- **Database Services** - All database operations (blockchain, txpool) expose a `tower::Service` backed by a thread pool
- **Consensus Services** - Block and transaction verification are implemented as services
- **Context Service** - Tracks blockchain state (height, versions, difficulty)

Services communicate via typed request/response enums, providing clean async boundaries between components.

#### Database Abstraction
The database layer has three levels:

1. **`cuprate-database`** - Backend-agnostic traits (`Env`, `TxRo`, `TxRw`, `DatabaseRo`, `DatabaseRw`)
2. **Domain databases** - `cuprate-blockchain` and `cuprate-txpool` define tables and operations
3. **Service layer** - Thread-pooled `tower::Service` API for async access

Backend selection (LMDB via `heed` or `redb`) is done via Cargo features. Default is `heed`.

#### Consensus Organization
Consensus is split into two layers:

- **`cuprate-consensus-rules`** - Pure functions for validating individual rules (flexible, requires correct data)
- **`cuprate-consensus`** - Higher-level services that orchestrate verification with context

Use `cuprate-consensus` for integration; fall back to `cuprate-consensus-rules` if you need fine-grained control.

### Network Zones
Cuprate supports multiple network zones (clearnet, Tor, i2p). The P2P layer is generic over network zones, allowing simultaneous operation across different networks. Each zone has its own address book and connection pool.

## Crate Naming Convention

All Cuprate crates are prefixed with `cuprate-`, but their directories are not:

| Directory | Crate Name |
|-----------|------------|
| `storage/database` | `cuprate-database` |
| `net/levin` | `cuprate-levin` |
| `consensus/rules` | `cuprate-consensus-rules` |

## Code Style Guidelines

- **Import organization:** Separate and sort imports as `core`, `std`, `third-party`, Cuprate crates, current crate
- **Comments:** Use `// Comment like this.` not `//like this`
- **TODOs:** Use `TODO` instead of `FIXME`
- **Avoid unsafe:** Minimize unsafe code usage
- Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines)

## Database Backend Switching

To switch between database backends:

```bash
# Use redb instead of default heed
cargo build --package cuprate-blockchain --no-default-features --features redb

# Test with redb
cargo test --package cuprate-blockchain --no-default-features --features redb
```

The backend is selected at compile time via Cargo features in `storage/database/Cargo.toml`:
- `heed` - LMDB backend (default)
- `redb` - Pure Rust backend
- `redb-memory` - In-memory redb for testing

## Documentation

- **Architecture book:** https://architecture.cuprate.org - Cuprate's internal design
- **Protocol book:** https://monero-book.cuprate.org - Monero protocol documentation
- **User book:** https://user.cuprate.org - User guide for cuprated
- **API docs:** https://doc.cuprate.org - Generated from code (all crates start with `cuprate_`)

Build docs locally: `cargo doc` then open `target/doc/cuprate_<crate>/index.html`

## CI Requirements

The CI pipeline enforces:
- Code formatting (`cargo fmt --all --check`)
- Clippy lints with warnings as errors
- Documentation builds without warnings
- All tests pass
- Typo checking
- Feature powerset compatibility checking

Run these locally before pushing to avoid CI failures.

## Common Development Patterns

### Reading from Database
Use the service API for async access:
```rust
let response = blockchain_read_handle
    .ready()
    .await?
    .call(BlockchainReadRequest::BlockByHeight(height))
    .await?;
```

### Writing to Database
Writes also go through the service:
```rust
blockchain_write_handle
    .ready()
    .await?
    .call(BlockchainWriteRequest::WriteBlock(block))
    .await?;
```

### Adding New Tables
Use the `define_tables!` macro in the relevant database crate (blockchain or txpool) to generate table definitions and associated traits.

## Testing with monerod

Some integration tests spawn `monerod` (the reference Monero node) to test compatibility. Place a `monerod` binary at the repository root. Tests using `cuprate-test-utils::monerod()` will automatically:
- Start monerod in regtest mode
- Allocate free ports for P2P/RPC
- Clean up on test completion

## Platform-Specific Notes

### Windows
- The `fuzz` crate is excluded from `default-members` due to build issues
- Use `--workspace` flag to explicitly build fuzz targets: `cargo build --workspace`

### Building cuprated
The main binary requires all features and dependencies. On first build, dependencies compile with optimization level 3 even in dev mode, so initial compilation is slower but provides better runtime performance for development.
