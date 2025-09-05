# Deep Dive: I2C Support in Hubris

Based on comprehensive analysis of the Hubris codebase, here's a detailed examination of I2C support and hardware abstraction layers.

## Architecture Overview

Hubris implements I2C as a **client-server architecture** with three main layers:

### 1. **I2C Server Drivers** (`drv/stm32xx-i2c-server`, `drv/lpc55-i2c`)
- **STM32 I2C Server**: Primary implementation for STM32H7/G0 microcontrollers  
- **LPC55 I2C Driver**: Support for LPC55 family chips
- Handles low-level hardware interactions, interrupt management, and protocol implementation
- Manages I2C bus multiplexing and segment switching

### 2. **I2C API Layer** (`drv/i2c-api`, `drv/i2c-types`)
- **Client API** (`drv/i2c-api/src/lib.rs`): Provides `I2cDevice` struct with methods like:
  - `read_reg<R,V>()` - Read typed register values  
  - `write()` - Write data to devices
  - `read_block()` - SMBus block operations
  - `write_read_reg()` - Combined write-then-read operations
- **Common Types** (`drv/i2c-types/src/lib.rs`): Defines controllers, ports, mux/segment addressing, and error codes

### 3. **Device Drivers** (`drv/i2c-devices`)
Over **40+ device drivers** including:
- **Temperature sensors**: ADT7420, MCP9808, TMP117, TMP451, PCT2075
- **Power controllers**: ISL68224, RAA229618, TPS546B24A, LM5066
- **GPIO expanders**: PCA9538, PCA9956B  
- **Fan controllers**: EMC2305, MAX31790
- **Hot swap controllers**: ADM1272, LTC4282, MAX5970
- **PMBus devices** with comprehensive macros for rail/phase management

## Key Technical Features

### **5-Tuple Device Addressing**
I2C devices are uniquely identified by:
```rust
pub struct I2cDevice {
    pub controller: Controller,    // I2C0-I2C7
    pub port: PortIndex,          // Port on controller  
    pub segment: Option<(Mux, Segment)>, // Optional mux/segment
    pub address: u8,              // Device I2C address
}
```

### **Advanced Multiplexer Support**
- Support for hierarchical I2C multiplexing (mux + segments)
- Multiple mux drivers: `ltc4306`, `max7358`, `pca9545`, `pca9548`, `oximux16`
- Automatic mux state management and error recovery
- Hardware reset lines for mux chips

### **Robust Error Handling**
Comprehensive error taxonomy in `drv/i2c-types/src/lib.rs`:
- Bus-level errors: `BusLocked`, `BusReset`, `BusError`
- Mux-specific errors: `MuxNotFound`, `SegmentDisconnected`  
- Device errors: `NoDevice`, `NoRegister`
- Protocol errors: `TooMuchData`, `ControllerBusy`

### **Interrupt-Driven Operation**
- Full interrupt support with sophisticated state machines
- Timeout handling and automatic recovery
- Support for both I2C master and slave (target) modes
- "Konami Code" sequences for unusual device unlock procedures

## Build System Integration

### **Code Generation Pipeline** (`build/i2c/src/lib.rs`)
The build system generates compile-time I2C configurations:

1. **Device Configuration**: Parses `app.toml` I2C device definitions
2. **Pin Mapping**: Generates GPIO/alternate function assignments
3. **Mux Configuration**: Creates multiplexer routing tables  
4. **Sensor Integration**: Maps devices to sensor IDs for monitoring
5. **Validation Support**: Creates device validation functions

Example generated code:
```rust
pub fn max31790(task: TaskId) -> I2cDevice {
    I2cDevice::new(task, 
        Controller::I2C4, 
        PortIndex(0), 
        Some((Mux::M1, Segment::S3)), 
        0x20)
}
```

### **Multi-Architecture Support**
- STM32H7 (H743/H753): 100MHz APB1, advanced timing configuration
- STM32G0 (G030/G031): 16MHz APB, optimized for lower power
- LPC55: FlexComm-based I2C with 12MHz clocking
- AMD Milan erratum 1394 workarounds (extended setup times)

## Application Integration

### **Task Configuration**
Applications declare I2C usage in `app.toml`:
```toml
[tasks.i2c_driver]
name = "drv-stm32xx-i2c-server"
features = ["h753"]
uses = ["i2c2", "i2c3", "i2c4"]
task-slots = ["sys"]
interrupts = {"i2c2.event" = "i2c2-irq"}
```

### **Real-World Usage**
- **Gimlet**: Server management with thermal/power monitoring via I2C
- **Sensor tasks**: Temperature, power, and fan control
- **Thermal management**: Multi-rail power monitoring with PMBus
- **VPD (Vital Product Data)**: EEPROM access for hardware inventory

## Advanced PMBus Integration

Hubris provides sophisticated PMBus support via macros:
```rust
// Read from specific power rail
pmbus_rail_read!(device, rail_id, READ_VIN)

// Read with rail + phase selection  
pmbus_rail_phase_read!(device, rail_id, phase_id, READ_IOUT)
```

## Reliability & Debugging

### **Extensive Tracing** 
Ring buffer tracing (`drv/stm32xx-i2c/src/lib.rs:159-199`) captures:
- Register reads/writes with timestamps
- Bus state transitions  
- Error conditions and recovery
- Interrupt handling flow

### **Hardware Validation**
- Device-specific validation routines for hardware presence detection
- Automatic device discovery and health checking
- Integration with Humility debugger for runtime inspection

## Hardware Abstraction Layer Analysis

### **No Standard HAL Integration**

Hubris does **not use standard hardware abstraction layers** like embedded-hal. Instead, it implements **custom, domain-specific abstractions** tailored for its microkernel architecture.

### **What Hubris Uses Instead**

#### **Custom Driver APIs**
Each peripheral type has its own specialized API:

```rust
// I2C API
pub struct I2cDevice {
    pub controller: Controller,
    pub port: PortIndex, 
    pub segment: Option<(Mux, Segment)>,
    pub address: u8,
}

// Task-specific APIs for GPIO, SPI, etc.
```

#### **System Call Interface** (`sys/userlib`)
Hardware access goes through the microkernel's system call layer:
```rust
// All hardware access is mediated by the kernel
sys_send(task_id, operation, data, leases)
```

#### **Task-Based Hardware Ownership**
- Hardware peripherals are **exclusively owned** by specific tasks
- Access is via **message passing** between tasks
- No shared hardware access patterns

### **Why No Standard HAL?**

#### **Microkernel Architecture Constraints**
1. **Memory Protection**: Tasks can't directly access hardware registers
2. **Exclusive Ownership**: Each peripheral belongs to exactly one task
3. **Message Passing**: All hardware access is via IPC, not function calls

#### **Real-Time Requirements** 
Standard HALs often introduce:
- **Allocation**: Hubris is `#![no_std]` with no heap
- **Runtime Overhead**: Direct system calls are more predictable
- **Generic Abstractions**: Hubris optimizes for specific use cases

#### **Compile-Time Configuration**
Hubris generates hardware configurations at build time from TOML:
```toml
[tasks.i2c_driver]
uses = ["i2c2", "i2c3", "i2c4"]  # Compile-time peripheral assignment
```

This is incompatible with runtime HAL initialization patterns.

### **Development Implications**

#### **Advantages**
- ✅ **Zero Runtime Cost**: No HAL overhead
- ✅ **Memory Safety**: Kernel enforces peripheral access
- ✅ **Predictable Timing**: No hidden allocations or abstractions
- ✅ **Type Safety**: Compile-time verification of hardware usage

#### **Trade-offs**
- ❌ **No Portability**: Drivers are Hubris-specific
- ❌ **Learning Curve**: Custom APIs instead of familiar patterns
- ❌ **Ecosystem Isolation**: Can't use standard embedded-rust crates
- ❌ **Development Overhead**: Must write custom drivers

### **Abstraction Philosophy**

Hubris follows a **"hardware-specific, software-abstract"** approach:

1. **Low Level**: Direct register manipulation in server tasks
2. **API Layer**: Clean, typed interfaces (like `I2cDevice`)  
3. **Application Layer**: High-level operations via message passing

This is fundamentally different from embedded-hal's **"hardware-abstract, software-generic"** approach.

### **Comparison with Standard Approaches**

| Aspect | embedded-hal | Hubris |
|--------|-------------|--------|
| **Hardware Access** | Direct register/trait calls | System calls + IPC |
| **Memory Model** | Shared ownership | Exclusive task ownership |
| **Abstraction** | Generic traits | Domain-specific APIs |
| **Portability** | Cross-platform | Hubris-specific |
| **Safety** | Rust type system | Microkernel isolation |
| **Performance** | Variable (trait objects) | Predictable (direct calls) |

## Summary

Hubris provides **enterprise-grade I2C support** with:
- ✅ **40+ device drivers** for real-world hardware
- ✅ **Multi-level multiplexing** with automatic management  
- ✅ **Interrupt-driven performance** with robust error recovery
- ✅ **Compile-time configuration** generation from TOML
- ✅ **PMBus integration** for power management  
- ✅ **Multi-architecture support** (STM32H7/G0, LPC55)
- ✅ **Production deployment** in Oxide server hardware

The I2C subsystem demonstrates Hubris's approach to embedded systems: type-safe, compile-time verified, with comprehensive error handling suitable for mission-critical server hardware.

### **Bottom Line on HAL**

Hubris **deliberately avoids** standard HAL layers in favor of:
- **Microkernel message passing** for hardware access
- **Custom APIs** optimized for server hardware use cases  
- **Compile-time configuration** instead of runtime abstraction
- **Task isolation** over shared hardware access patterns

This makes Hubris drivers non-portable but highly optimized for its specific use case of **reliable server firmware** where predictability and safety outweigh ecosystem compatibility.