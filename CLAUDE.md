# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Rust workspace implementing Management Component Transport Protocol (MCTP) support, primarily for embedded applications. The codebase provides both `std` and `no_std` implementations with support for various transport bindings.

## Key Architecture

### Core Crates Hierarchy
- **`mctp`**: Base types and traits (`ReqChannel`, `Listener`, `Eid`)
- **`pldm`**: Platform Level Data Model base definitions built on MCTP
- **`mctp-estack`**: Embedded MCTP stack for `no_std` with fixed allocations
- **`mctp-linux`**: Linux kernel socket transport implementation

### Transport Implementations
- **`mctp-linux`**: Uses Linux kernel MCTP sockets
- **`mctp-usb-embassy`**: MCTP over USB for `embassy-usb`
- **`standalone`**: Standalone MCTP-over-serial for TTY devices

### PLDM Protocol Support
- **`pldm-fw`**: Firmware update protocol (Firmware Device responder + Update Agent)
- **`pldm-platform`**: Platform monitoring and control
- **`pldm-file`**: File transfer operations

### Applications
- **`pldm-fw-cli`**: Command-line PLDM firmware update utility

## Development Commands

### Build and Test
```bash
# Run full CI test suite (recommended)
./ci/runtests.sh

# Basic build and test
cargo build --release
cargo test

# Build for embedded targets (no_std)
cargo build --target thumbv7em-none-eabihf --no-default-features

# Documentation
cargo doc
```

### Code Quality
```bash
# Format code (required for CI)
cargo fmt

# Lint with Clippy
cargo clippy --all-targets

# Check all targets
cargo check --all-targets --locked
```

### Embedded Development
```bash
# Add embedded target
rustup target add thumbv7em-none-eabihf

# Build no_std crates for embedded
for crate in mctp pldm pldm-fw pldm-platform pldm-file; do
    cd $crate && cargo build --target thumbv7em-none-eabihf --no-default-features
done

# Build with alloc feature (for PLDM crates)
cd pldm && cargo build --target thumbv7em-none-eabihf --no-default-features --features alloc
```

## Code Organization

### Feature Flags
- Most crates support both `std` and `no_std` via feature flags
- `alloc` feature available for heap allocation without full `std`
- `defmt` vs `log` features for different logging backends (mutually exclusive with `std`)

### Testing Strategy
- Unit tests run with `cargo test`
- Cross-compilation testing for embedded targets
- Both `defmt` and `log` feature combinations tested
- CI runs on stable, 1.85 (MSRV), and nightly Rust

### Examples
- `mctp-linux/examples/`: Basic MCTP communication examples
- `standalone/examples/`: MCTP over serial examples
- `pldm-file/examples/`: File transfer examples

## Configuration

### Code Style
- Maximum line width: 80 characters (`.rustfmt.toml`)
- Rust edition 2021
- Strict warnings enabled in CI (`RUSTFLAGS="-D warnings"`)

### Fixed Sizes
`mctp-estack` uses compile-time configuration for embedded environments - check `config` module for buffer sizes and limits.

## Commit Conventions

- Include `Signed-off-by` trailer (`git commit -s`)
- Prefix commits affecting single crates with crate name (e.g., `mctp-estack: Fix buffer overflow`)
- Follow DCO requirements per CONTRIBUTING.md