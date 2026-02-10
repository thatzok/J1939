# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Rust crate implementing the SAE J1939 automotive protocol, used for CAN bus communication in heavy-duty vehicles. The crate is designed for `no_std` environments and targets embedded systems.

## Key Commands

### Build and Test
```bash
# Build the library
cargo build

# Run all tests
cargo test

# Run a specific test
cargo test test_name

# Run the j1939decode example
cargo run --example j1939decode 0x0CB34A29

# Check code with clippy
cargo clippy

# Format code
cargo fmt
```

### Development Requirements
- Rust 1.85.0 or later
- The crate uses edition 2024
- Default feature: `chrono` (for date/time conversions)
- Optional features: `std`, `alloc`

## Architecture

### Core Types

The crate is organized around the J1939 frame structure:

**Frame ID (`Id`)**: 29-bit CAN identifier containing:
- Priority (bits 26-28): 0-7, where 0 is highest
- Data Page (bit 24)
- PDU Format (bits 16-23): determines PDU1 vs PDU2 format
- PDU Specific (bits 8-15): DA for PDU1, GE for PDU2
- Source Address (bits 0-7)

**Frame (`Frame`)**: Complete J1939 message containing:
- Frame ID
- PDU (Protocol Data Unit): up to 8 bytes of data
- PDU length

### Builder Pattern

The crate uses builder patterns for constructing frames:

```rust
// Build ID first, then frame
let id = IdBuilder::from_pgn(PGN::Request)
    .priority(6)
    .sa(source_address)
    .da(destination_address)
    .build();

let frame = FrameBuilder::new(id)
    .copy_from_slice(&data)
    .build();
```

### PDU Format Types

J1939 has two PDU formats determined by the PF (PDU Format) byte:
- **PDU1** (PF < 240): Peer-to-peer messages with destination address
- **PDU2** (PF ≥ 240): Broadcast messages with group extension

The `Id` type automatically handles the distinction when extracting fields.

### Module Organization

- **`lib.rs`**: Core types (`Id`, `Frame`, builders), PDU format logic
- **`pgn.rs`**: Parameter Group Number (PGN) enum with all standard PGNs
- **`protocol.rs`**: Helper functions for common J1939 messages (request, address_claimed, acknowledgement, commanded_address)
- **`transport.rs`**: Transport protocol for multi-frame messages (`BroadcastTransport`)
- **`spn.rs`**: Suspect Parameter Numbers - data structures for specific PGN payloads (TimeDate, ElectronicEngineController1, etc.)
- **`diagnostic.rs`**: Diagnostic message structures (DM1, DM2, etc.)
- **`name.rs`**: NAME field structure for address claiming
- **`sa.rs`**: Source address definitions
- **`slots.rs`**: Bit slot extraction utilities

### Transport Protocol

For messages larger than 8 bytes, use `BroadcastTransport`:
- Automatically splits data into multiple frames
- First frame is TP.CM (Connection Management) with total size and packet count
- Subsequent frames are TP.DT (Data Transfer) with sequence numbers
- Supports up to 1785 bytes (`DATA_MAX_LENGTH`)

### Code Quality Constraints

The project enforces strict linting:
- `unsafe_code = "deny"` - no unsafe code allowed
- `clippy::pedantic = "warn"` - pedantic clippy warnings
- `clippy::unwrap_used = "warn"` - avoid `.unwrap()`
- All warnings are denied (`#![deny(warnings)]`)

When making changes, ensure code passes clippy and doesn't introduce warnings.

### Testing

Tests are inline with `#[cfg(test)]` blocks in each module. Test coverage includes:
- ID encoding/decoding for both PDU1 and PDU2 formats
- Frame building with various PGNs
- Transport protocol message fragmentation and reassembly
- SPN data structure serialization/deserialization

## Common Patterns

### Creating a Standard J1939 Message
Use the `protocol` module helpers for common message types rather than manually constructing frames:
- `protocol::request(da, sa, pgn)` - Request a PGN
- `protocol::address_claimed(sa, name)` - Claim an address
- `protocol::acknowledgement(sa, pgn)` - Acknowledge a message

### Working with SPNs
Use predefined structures in `spn.rs` when available:
- These provide `from_pdu()` and `to_pdu()` methods
- Example: `TimeDate`, `ElectronicEngineController1`

### Handling Unknown PGNs
The `PGN` enum has variants for undefined PGNs:
- `PGN::ProprietaryB(u32)` - Proprietary B range (65280-65535)
- `PGN::Other(u32)` - Any other PGN
