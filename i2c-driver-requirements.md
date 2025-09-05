# MCTP I2C Driver Requirements Deep Dive

This document details the I2C driver interface requirements for integrating with `mctp-estack`'s I2C transport implementation.

## Driver Interface Requirements

### Key Observation: No Direct HAL Dependency
**The `mctp-estack` I2C implementation does NOT directly depend on any I2C driver traits** like `embedded-hal`. Instead, it provides packet processing abstractions that work with raw byte buffers, allowing integration with any I2C driver implementation.

## Integration Architecture

### Layer Separation
```
┌─────────────────────────────┐
│ Application Layer           │
├─────────────────────────────┤
│ MctpI2cHandler              │  ← High-level message handling
├─────────────────────────────┤
│ MctpI2cEncap                │  ← Packet encode/decode
├─────────────────────────────┤
│ Your I2C Driver             │  ← Platform-specific I2C HAL
├─────────────────────────────┤
│ Hardware I2C Controller     │
└─────────────────────────────┘
```

### I2C Driver Interface Needs

Your I2C driver must provide these capabilities:

#### 1. **Target Mode (Slave) Support**
```rust
// Required capability: Receive data when addressed as target
fn target_receive(&mut self, buffer: &mut [u8]) -> Result<usize, Error>;
```
- Must handle being addressed with 7-bit slave address
- Should support up to 255-byte transfers (SMBus block transfers)
- Buffer must capture complete transaction including headers

#### 2. **Controller Mode (Master) Support** 
```rust
// Required capability: Send data to remote target
fn controller_send(&mut self, target_addr: u8, data: &[u8]) -> Result<(), Error>;
```
- Must support 7-bit addressing (0x00-0x7F range)
- Should support variable length transfers up to 255 bytes
- Must handle SMBus block write semantics

#### 3. **Optional: SMBus PEC Support**
```rust
// Optional but recommended: Hardware PEC validation
fn set_pec_enabled(&mut self, enabled: bool);
fn is_pec_error(&self) -> bool;
```

## SMBus Protocol Requirements

### Command Code Handling
- **Fixed Command Code**: `0x0F` (per DSP0237)
- Driver must support SMBus "block write" semantics
- Transaction format: `[SLAVE_ADDR] [0x0F] [BYTE_COUNT] [DATA...] [PEC?]`

### Address Format Validation
The stack enforces strict I2C address format:

```rust
// Destination: 7-bit address left-shifted, write bit (0)
dest_byte = (i2c_addr << 1) | 0;    // Must be even

// Source: 7-bit address left-shifted, read bit (1)  
source_byte = (i2c_addr << 1) | 1;  // Must be odd
```

**Driver Requirements:**
- Support 7-bit addressing (0x00-0x7F, not 0x80-0xFF)
- Handle read/write bit manipulation correctly
- Validate addresses are within 7-bit range

## Packet Buffer Management

### Receive Path Buffer Requirements
```rust
// Your driver needs to provide buffers of this size:
const RX_BUFFER_SIZE: usize = 4 + 254 + 1;  // Header + MTU + PEC
let mut rx_buf = [0u8; RX_BUFFER_SIZE];
```

### Send Path Buffer Requirements
```rust  
// Your driver must accept buffers up to:
const TX_BUFFER_SIZE: usize = 4 + 254 + 1;  // Header + MTU + PEC
```

## Integration Pattern

### Minimal Integration Example
```rust
use mctp_estack::i2c::{MctpI2cHandler, MctpI2cEncap};
use mctp_estack::{Stack, Vec};

struct MyI2cDriver { /* your implementation */ }

impl MyI2cDriver {
    fn target_receive(&mut self, buf: &mut [u8]) -> Result<usize, MyError> {
        // Your I2C target receive implementation
        todo!()
    }
    
    fn controller_send(&mut self, addr: u8, data: &[u8]) -> Result<(), MyError> {
        // Your I2C controller send implementation  
        todo!()
    }
}

// Integration setup
let mut stack = Stack::new(own_eid);
let mut send_buf: Vec<u8, 1032> = Vec::new(); // MAX_PAYLOAD sized
let mut handler = MctpI2cHandler::new(own_i2c_addr, &mut send_buf);
let mut driver = MyI2cDriver::new();

// Receive handling
let mut rx_buf = [0u8; 259]; // 4 + 254 + 1
if let Ok(len) = driver.target_receive(&mut rx_buf) {
    if let Ok(Some((msg, src_addr))) = handler.receive(&rx_buf[..len], &mut stack) {
        // Handle received MCTP message
        process_message(msg, src_addr);
    }
}

// Send handling  
if handler.is_send_ready() {
    let mut tx_buf = [0u8; 259];
    if let SendOutput::Packet(packet) = handler.send_fill(&mut tx_buf) {
        // Extract I2C destination from packet header
        let dest_addr = packet[0] >> 1; // Remove write bit
        driver.controller_send(dest_addr, packet)?;
    }
}
```

## Hardware Requirements Summary

### Minimum I2C Controller Features
1. **7-bit addressing** (not 10-bit)  
2. **Target mode** (slave receive capability)
3. **Controller mode** (master transmit capability)
4. **Variable length transfers** (1-255 bytes)
5. **SMBus compatibility** (block transfers)

### Recommended Features  
1. **Hardware PEC** calculation and validation
2. **DMA support** for large transfers
3. **Interrupt-driven** operation
4. **Clock stretching** tolerance
5. **Bus recovery** mechanisms

## PEC (Packet Error Code) Handling

### Software PEC (Always Available)
```rust
// The stack provides software PEC calculation
use mctp_estack::i2c::MctpI2cEncap;

let encap = MctpI2cEncap::new(own_addr);
let with_pec = encap.encode(dest_addr, payload, buffer, true)?; // Adds PEC
let (mctp_data, src) = encap.decode(rx_buffer, true)?; // Validates PEC
```

### Hardware PEC Integration
```rust  
// If your I2C driver has hardware PEC support:
driver.enable_pec(true);
if driver.pec_error_detected() {
    return Err(PecError);
}
// Skip software PEC validation
let (mctp_data, src) = encap.decode(rx_buffer, false)?; // No software PEC
```

## Performance Considerations

### Buffer Reuse
- The handler maintains internal message buffers
- Reuse receive/transmit buffers across calls
- Consider DMA-capable buffer alignment

### Interrupt Integration  
```rust
// Example interrupt handler integration
fn i2c_interrupt_handler() {
    static mut RX_BUF: [u8; 259] = [0; 259];
    
    if i2c.target_addressed() {
        let len = i2c.read_target_data(unsafe { &mut RX_BUF });
        // Queue for main loop processing
        enqueue_rx_data(&unsafe { RX_BUF }[..len]);
    }
}
```

## Error Handling Requirements

Your I2C driver should report these error conditions:
- **Address NACK**: Target not responding
- **Data NACK**: Target rejected data  
- **Bus errors**: SDA/SCL line issues
- **Timeout errors**: Clock stretching limits
- **PEC errors**: If hardware PEC enabled

The MCTP stack will handle protocol-level errors like invalid headers, bad message types, etc.

## Standards Compliance

This implementation follows:
- **DSP0237**: MCTP SMBus/I2C Transport Binding Specification  
- **SMBus 2.0**: System Management Bus specification
- **I2C**: Philips I2C-bus specification

The key insight is that `mctp-estack` provides protocol abstraction above your I2C driver, not a replacement for it. Your driver handles the hardware interface while the stack handles MCTP message semantics.