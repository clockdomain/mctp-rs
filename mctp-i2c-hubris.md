# MCTP Integration Assessment for Hubris

After analyzing both Hubris's I2C architecture and MCTP's requirements, there are **significant architectural challenges** that reveal fundamental gaps in what Hubris provides for MCTP integration.

## ⚠️ Critical Architectural Mismatches

### 1. **Control Flow Paradigm Conflict**

**MCTP Requirements:**
- Direct hardware control for both controller (master) and target (slave) operations
- Immediate response to unsolicited I2C target transactions
- Raw buffer access for packet processing

**Hubris Reality:**
- Message-passing architecture with I2C server as intermediary
- Client-initiated request/response patterns
- Abstracted APIs that hide raw I2C buffers

**Impact:** This is a fundamental architectural incompatibility.

### 2. **Target Mode Operation Gap**

**MCTP Need:**
```rust
// MCTP needs to handle incoming I2C slave transactions like:
// [Other Controller] -> [This Device as Target] with command 0x0F
```

**Hubris Gap:** 
The I2C API (`drv/i2c-api`) provides methods like:
- `read_reg<R,V>()` - Client reading from devices  
- `write()` - Client writing to devices
- `read_block()` - Client SMBus operations

**Missing:** No API for **acting as an I2C target** and handling incoming transactions from other controllers.

## 🔍 Deep Dive: Specific Integration Gaps

### **Gap 1: I2C Target Mode Support**

**Current Hubris I2C Server (`drv/stm32xx-i2c-server`):**
```rust
// Existing: Only controller (master) operations
pub fn write(&mut self, device: &I2cDevice, data: &[u8]) -> Result<()>
pub fn read(&mut self, device: &I2cDevice, data: &mut [u8]) -> Result<()>
```

**Missing for MCTP:**
```rust
// Needed: Target (slave) mode operations  
pub fn register_target_handler(&mut self, addr: u8, handler: TargetCallback) -> Result<()>
pub fn target_respond(&mut self, data: &[u8]) -> Result<()>
```

### **Gap 2: Raw Buffer Access**

**MCTP Requirement:**
```rust
// Direct access to I2C packet bytes including headers
let (mctp_payload, i2c_source) = encap.decode(&raw_i2c_packet, pec_enabled)?;
```

**Hubris Limitation:**
```rust
// Current API abstracts away raw I2C protocol details  
device.read_reg::<Register, Value>()?;  // Typed, not raw bytes
```

### **Gap 3: Asynchronous Target Callbacks** 

**MCTP Need:** Handle incoming I2C transactions asynchronously
**Hubris Challenge:** Message-passing makes asynchronous callbacks complex

## 💡 Proposed Integration Architecture

### **Three-Layer Architecture Solution**

```
┌─────────────────────────────────────┐
│ MCTP Applications                   │  ← PLDM, Control Protocol
├─────────────────────────────────────┤
│ MCTP Server Task                    │  ← Protocol handling, fragmentation  
│ - mctp-estack Stack                 │
│ - Message reassembly                │
│ - Tag management                    │
├─────────────────────────────────────┤
│ Enhanced I2C Server                 │  ← Hardware abstraction
│ - Target mode support              │
│ - Command code demuxing (0x0F)     │
│ - Raw buffer access                 │
├─────────────────────────────────────┤
│ STM32 I2C Hardware                  │
└─────────────────────────────────────┘
```

### **Required Extensions**

#### **1. Enhanced I2C Server** (`drv/stm32xx-i2c-server`)
```rust
// New target mode operations
#[derive(Copy, Clone, Debug, PartialEq)]
pub enum I2cServerOperation {
    // Existing operations...
    Write(I2cDevice, LenLimit),
    Read(I2cDevice, LenLimit),
    
    // New MCTP target operations
    RegisterTargetHandler(u8),           // Register as I2C target 
    TargetTransactionReady(LenLimit),    // Incoming target transaction
    SendTargetResponse(LenLimit),        // Send response as target
}

// Target callback mechanism
pub struct TargetTransaction {
    pub command_code: u8,
    pub source_addr: u8, 
    pub data: [u8; 255],  // Raw I2C data
    pub len: usize,
    pub pec_valid: bool,
}
```

#### **2. MCTP Server Task** (`drv/mctp-server`)
```rust
pub struct MctpServer {
    stack: mctp_estack::Stack,
    router: mctp_estack::Router<'static>,
    i2c_client: I2cDevice,  // Connection to I2C server
    handlers: [MctpI2cHandler; MAX_I2C_BUSES],
}

// Task operations  
#[derive(Copy, Clone, Debug, PartialEq)]  
pub enum MctpOperation {
    // Application interface
    SendMessage(Eid, MsgType, LenLimit),
    RegisterListener(MsgType),
    
    // I2C integration
    I2cTargetRx(LenLimit),      // From I2C server
    I2cSendReady,               // To I2C server
}
```

## ⚡ Performance & Real-Time Analysis

### **Latency Impact Assessment**

**Direct Hardware Access (mctp-estack standalone):**
```
I2C IRQ → Handler → MCTP Stack → Application
~5-10μs
```

**Hubris Message-Passing Architecture:**  
```
I2C IRQ → I2C Server → IPC → MCTP Server → IPC → Application  
~50-100μs (10x overhead)
```

### **Critical Timing Requirements**

**MCTP Protocol Timing:**
- **Fragment Reassembly Timeout:** 6000ms (manageable)
- **Tag Response Window:** Variable per protocol (PLDM ~30s)
- **I2C Transaction Timeout:** ~10ms (hardware level)

**Hubris Impact:**
- ✅ **Protocol timeouts:** Message-passing overhead negligible vs. protocol timing  
- ⚠️ **I2C transaction response:** May need priority handling for target responses
- ✅ **Overall throughput:** Hubris scheduling sufficient for MCTP bandwidth

### **Memory Overhead**

**Additional Buffer Requirements:**
```rust
// Per I2C bus MCTP support
struct MctpI2cResources {
    rx_buffers: [[u8; 259]; 4],        // 4 * 259 = 1036 bytes
    tx_buffers: [[u8; 259]; 4],        // 4 * 259 = 1036 bytes  
    message_queues: [MctpMessage; 16], // ~4KB for message queue
}
// Total: ~6KB per I2C bus with MCTP
```

**Hubris Memory Budget:** Server platforms typically have 512KB+ RAM, so manageable.

## 📋 Implementation Strategy  

### **Phase 1: Core I2C Extensions**
1. **Extend STM32 I2C Server** with target mode support
2. **Add command code demultiplexing** (route 0x0F to MCTP)
3. **Implement raw buffer access** APIs
4. **Test target mode** with simple echo functionality

### **Phase 2: MCTP Server Task**
1. **Create `drv-mctp-server`** task with `mctp-estack` integration
2. **Implement IPC protocol** between I2C and MCTP servers
3. **Add build system integration** for MCTP device configuration
4. **Basic MCTP Control Protocol** support

### **Phase 3: Application Integration**
1. **PLDM support** via MCTP server
2. **Multi-bus MCTP routing** 
3. **Performance optimization** and buffer management
4. **Production validation** with real server hardware

### **App.toml Configuration Example**
```toml
[tasks.i2c_driver]
name = "drv-stm32xx-i2c-server"  
features = ["h753", "mctp-target"]  # New MCTP feature
uses = ["i2c2", "i2c3", "i2c4"]

[tasks.mctp_server] 
name = "drv-mctp-server"
priority = 3                        # High priority for real-time response  
uses = ["timer2"]                   # For reassembly timeouts
task-slots = ["i2c_driver"]         # IPC to I2C server

[mctp.devices]
# MCTP device on I2C4 at address 0x10
bmc_mctp = { controller = "i2c4", address = 0x10, eid = 8 }
```

## 🎯 Bottom Line Assessment

### **Feasibility: ✅ Possible but Significant Work Required**

**Advantages of Hubris for MCTP:**
- ✅ **Real-time guarantees** excellent for MCTP timing requirements
- ✅ **Memory safety** critical for server firmware reliability  
- ✅ **Existing I2C infrastructure** provides solid foundation
- ✅ **Multi-architecture support** (STM32H7/G0, LPC55)
- ✅ **Production proven** in Oxide server hardware

**Major Gaps to Address:**
- ❌ **No I2C target mode** in current API
- ❌ **No raw buffer access** - everything is typed/abstracted
- ❌ **No asynchronous target callbacks** - client/server model only
- ❌ **Message-passing overhead** adds 10x latency vs. direct access

### **Development Effort Estimate: ~3-6 months**
- **I2C Server Extensions:** 4-6 weeks
- **MCTP Server Task:** 6-8 weeks  
- **Build System Integration:** 2-3 weeks
- **Testing & Validation:** 4-6 weeks
- **Documentation & Cleanup:** 2-3 weeks

### **Risk Assessment:**

**High Risk:**
- **Architectural complexity** - message passing may not be optimal for protocol handling
- **Performance overhead** - IPC latency could impact I2C timing windows
- **Integration complexity** - significant changes to core I2C infrastructure

**Medium Risk:**  
- **Memory overhead** - additional buffers and task memory
- **Multi-bus scaling** - coordination between multiple I2C buses

**Low Risk:**
- **Protocol compatibility** - mctp-estack handles MCTP protocol correctly
- **Hardware support** - STM32 I2C supports target mode in hardware

## 🎯 Alternative: Direct I2C Ownership by MCTP Task

### **Superior Architecture Solution**

**Direct I2C ownership by the MCTP task would solve the fundamental architectural problems** identified above. This approach treats MCTP as a first-class Hubris citizen that owns its hardware resources.

### **Direct Ownership Architecture**

#### **Hubris Resource Allocation Model**
```toml
[tasks.mctp_server]
name = "drv-mctp-i2c-server"
features = ["h753"]
uses = ["i2c2", "i2c3"]              # Direct I2C ownership
interrupts = {
    "i2c2.event" = "i2c2-event-irq",
    "i2c2.error" = "i2c2-error-irq",
    "i2c3.event" = "i2c3-event-irq", 
    "i2c3.error" = "i2c3-error-irq"
}
task-slots = ["sys"]

[tasks.sensor_manager] 
name = "drv-gimlet-sensors"
uses = ["i2c4", "i2c7"]              # Other I2C buses for sensors
# No overlap with MCTP buses
```

#### **Task Architecture**
```
┌─────────────────────────────────────┐
│ MCTP Applications                   │
├─────────────────────────────────────┤
│ MCTP Task (owns I2C2, I2C3)        │  
│ - mctp-estack Stack & Router        │
│ - Direct I2C hardware control       │
│ - Target + Controller modes         │
│ - Raw buffer access                 │  
│ - Interrupt handlers                │
├─────────────────────────────────────┤
│ STM32 I2C2/I2C3 Hardware           │
└─────────────────────────────────────┘
```

## ✅ Major Advantages of Direct Ownership

### **1. Eliminates IPC Overhead**
```rust
// Direct ownership: ~5-10μs latency  
I2C IRQ → MCTP Task → mctp-estack → Application

// vs Message-passing: ~50-100μs latency
I2C IRQ → I2C Server → IPC → MCTP Server → IPC → Application  
```

### **2. Native Target Mode Support**
```rust
// MCTP task can directly handle I2C target interrupts
#[interrupt]
fn i2c2_event_irq() {
    let mctp_task = MCTP_TASK.get();
    // Direct hardware register access for target transactions
    if i2c.target_addressed_with_command(0x0F) {
        mctp_task.handle_target_transaction();
    }
}
```

### **3. Raw Buffer Access**
```rust
// Direct access to I2C hardware buffers
struct MctpI2cTask {
    i2c2: stm32h7xx_hal::i2c::I2c<stm32h7xx_hal::stm32::I2C2>,
    stack: mctp_estack::Stack,
    handlers: [MctpI2cHandler; 2], // For I2C2, I2C3
}

impl MctpI2cTask {
    fn handle_target_rx(&mut self) {
        let mut rx_buf = [0u8; 259];
        let len = self.i2c2.target_read_raw(&mut rx_buf);
        
        // Direct MCTP processing
        if let Ok(Some((msg, src))) = self.handlers[0].receive(&rx_buf[..len], &mut self.stack) {
            self.dispatch_message(msg, src);
        }
    }
}
```

## 🔧 Implementation Strategy

### **Reuse Existing I2C Driver Code**
Hubris already has excellent STM32 I2C drivers in `drv/stm32xx-i2c-server`. The MCTP task can:

1. **Embed the I2C driver logic** directly instead of running it as a server
2. **Add target mode support** to the existing STM32 I2C implementation  
3. **Extend with raw buffer APIs** for MCTP packet processing

### **Minimal Changes Required**
```rust
// drv/mctp-i2c-server/src/lib.rs
use drv_stm32xx_i2c::I2cController;  // Reuse existing driver

struct MctpI2cServer {
    // Direct hardware ownership
    i2c_controllers: [I2cController; MAX_I2C_BUSES],
    
    // MCTP protocol handling  
    stack: mctp_estack::Stack,
    router: mctp_estack::Router<'static>,
    handlers: [mctp_estack::i2c::MctpI2cHandler; MAX_I2C_BUSES],
}
```

## 📊 Architecture Comparison

| Aspect | Message-Passing | Direct Ownership |
|--------|-----------------|------------------|
| **Latency** | ~50-100μs | ~5-10μs |  
| **Complexity** | High (IPC protocol) | Low (direct calls) |
| **Target Mode** | Requires new APIs | Native hardware access |
| **Raw Buffers** | Abstraction layer needed | Direct hardware buffers |
| **Development Effort** | 3-6 months | 6-12 weeks |
| **Hubris Alignment** | Some architectural tension | Perfect fit |
| **Code Reuse** | Limited | High (existing I2C drivers) |

## ⚠️ Trade-offs to Consider

### **Potential Concerns**

#### **1. Bus Sharing Limitations**
```toml
# With direct ownership, entire I2C buses are dedicated to MCTP
[tasks.mctp_server]
uses = ["i2c2", "i2c3"]  # MCTP-only buses

[tasks.sensor_manager]  
uses = ["i2c4", "i2c7"]  # Separate buses for other devices
```

**Impact:** Need sufficient I2C buses for both MCTP and traditional devices. Most server platforms have 4+ I2C controllers, so typically not a problem.

#### **2. Mixed Protocol Support**
```rust
// Challenge: What if you want MCTP + PMBus on the same bus?
// Solution: MCTP task can provide traditional I2C APIs to other tasks
pub fn i2c_controller_request(&mut self, device: I2cDevice, operation: I2cOp) -> Result<()>
```

### **Advantages Outweigh Concerns**

#### **✅ Perfect Architectural Fit**
- **Exclusive ownership** aligns with Hubris's microkernel model
- **Direct hardware access** eliminates abstraction overhead
- **Interrupt-driven** operation matches real-time requirements
- **Memory safety** maintained through Rust ownership

#### **✅ Simpler Implementation**
- **Reuse existing I2C drivers** with minimal modifications
- **No new IPC protocols** needed
- **Standard Hubris patterns** for interrupt handling and task communication

#### **✅ Better Performance**
- **10x latency reduction** vs. message-passing
- **Predictable timing** for MCTP protocol requirements
- **Lower memory overhead** (no IPC buffers)

## 🚀 Revised Recommendation

**Direct I2C ownership by the MCTP task is the optimal solution.** This approach:

### **Solves All Major Problems:**
- ✅ **Native target mode** through direct interrupt handling
- ✅ **Raw buffer access** via hardware register manipulation  
- ✅ **No IPC overhead** - direct function calls
- ✅ **Architectural alignment** with Hubris ownership model

### **Reduces Development Effort:**
- **6-12 weeks** instead of 3-6 months
- **Reuse existing I2C driver code** with minimal changes
- **Standard Hubris patterns** for task implementation
- **No complex IPC protocols** to design and debug

### **Implementation Path:**
1. **Week 1-2:** Extract I2C controller code from `drv-stm32xx-i2c-server`
2. **Week 3-4:** Add target mode support and raw buffer APIs  
3. **Week 5-6:** Integrate `mctp-estack` with direct I2C control
4. **Week 7-8:** Add build system integration and testing
5. **Week 9-12:** PLDM applications and production validation

## 💡 Key Insight

The direct ownership model **fundamentally changes the problem** from "How do we make MCTP work with Hubris's I2C abstraction?" to "How do we make MCTP a first-class Hubris citizen?" 

The direct ownership model treats MCTP like any other hardware protocol driver in Hubris - it owns its resources and provides services to other tasks through well-defined APIs. This is exactly how Hubris is designed to work.

**This is the right architectural approach.** It leverages Hubris's strengths while eliminating the impedance mismatch between MCTP's requirements and the current I2C server model.