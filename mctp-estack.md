# mctp-estack Deep Dive

This document provides a comprehensive analysis of the `mctp-estack` crate's abstractions, features, and I2C support requirements.

## Core Abstractions

### Stack Architecture
**`Stack`**: The core stateless MCTP message processor that handles:
- Fragment reassembly using fixed-size buffers (`NUM_RECEIVE` concurrent messages)
- Tag ownership and flow management (`FLOWS` pending responses) 
- Message routing logic independent of transport

**`Router`**: High-level orchestrator that combines a `Stack` with transport ports:
- Manages multiple transport bindings via `PortLookup` trait
- Implements async MCTP traits (`AsyncListener`, `AsyncReqChannel`, `AsyncRespChannel`)
- Handles message bridging between ports
- Uses Embassy-sync for async coordination

### Transport Abstraction Layer
Each transport (I2C, Serial, USB) provides:
- Encapsulation/decapsulation of MCTP packets with transport headers
- MTU handling and validation
- Transport-specific error handling

## Fixed-Size Configuration System

Uses compile-time environment variables with defaults:

```rust
// Configurable via MCTP_ESTACK_* env vars
MAX_PAYLOAD: 1032 bytes          // Message size limit
NUM_RECEIVE: 4                   // Concurrent reassemblies  
FLOWS: 64                        // Pending response slots
MAX_MTU: 255                     // Transport MTU limit
PORT_TXQUEUE: 4                  // Per-port queue depth
MAX_PORTS: 2                     // Router port count
```

Memory usage: ~`MAX_PAYLOAD * NUM_RECEIVE` for reassembly buffers plus port queues.

## I2C Transport Support

### Protocol Implementation
- **SMBus-based**: Uses command code `0x0f` per DMTF DSP0237
- **PEC Support**: Optional Packet Error Code using `smbus-pec` crate with lookup tables
- **Address Format**: 7-bit I2C addresses with read/write bit handling
- **MTU**: 254 bytes maximum (limited by SMBus byte count field)

### Header Structure (`MctpI2cHeader`)
```
[dest_addr<<1][0x0f][byte_count][source_addr<<1|1]
```

### Key Requirements for I2C Usage:

1. **Address Management**: 7-bit I2C slave addresses (0x00-0x7F)
2. **PEC Handling**: Optional but recommended for error detection
3. **Byte Count Validation**: Header byte count must match actual packet length
4. **Buffer Sizing**: Output buffers need `MCTP_I2C_HEADER (4) + MCTP_MTU` bytes

### I2C Integration Pattern:
```rust
let encap = MctpI2cEncap::new(own_i2c_addr);
let (mctp_packet, i2c_source) = encap.decode(raw_i2c_data, use_pec)?;
let msg = stack.receive(mctp_packet)?;
```

## Feature Flags & Capabilities

**Logging**: Mutually exclusive `log` (default) vs `defmt` for embedded
**Transport**: I2C, Serial (DSP0253), USB (DSP0283) bindings included
**Protocol Support**: Built-in minimal MCTP Control Protocol implementation
**Async Runtime**: Uses Embassy-sync primitives, works with any async executor
**Memory Model**: `no_std` + `heapless::Vec` for zero-allocation operation

## Transport-Specific Features

### I2C
- SMBus PEC validation
- 7-bit addressing
- 254-byte MTU limit

### Serial  
- DSP0253 framing with CRC16
- Byte stuffing (0x7E flags, 0x7D escape)
- 251-byte MTU limit

### USB
- DSP0283 1.0.1 implementation  
- 512-byte transfer size
- 251-byte MTU limit

The architecture prioritizes deterministic memory usage and real-time performance for embedded applications while maintaining transport abstraction and standards compliance.