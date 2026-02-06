# IP Gap Analysis for Tamagotchi on Caravel

## Executive Summary

This document analyzes the availability of IP blocks from the NativeChips verified IP library for the Tamagotchi project. It identifies which IPs can be reused directly, which require minor modifications, and which need to be developed from scratch.

**Summary Status**:
- ✅ **Available & Ready**: 5 IPs
- ⚠️ **Available with Modifications**: 1 IP
- ❌ **Gap - Need Development**: 3 custom blocks
- 🔧 **Optional**: 1 IP

---

## Available NativeChips Verified IPs

### ✅ 1. CF_UART (UART Controller)
- **Location**: `/nc/ip/CF_UART`
- **Version**: v2.0.1
- **Status**: ✅ **VERIFIED & READY**
- **Use Case**: Debug/development UART interface
- **Configuration**:
  - 115200 baud
  - 8N1 format
  - TX/RX only
- **Integration**: Direct Wishbone connection
- **Notes**: Optional for production, useful for debugging
- **Recommendation**: **USE AS-IS** for debug builds, remove in production

---

### ✅ 2. CF_SPI (SPI Master Controller)
- **Location**: `/nc/ip/CF_SPI`
- **Version**: v2.0.1
- **Status**: ✅ **VERIFIED & READY**
- **Use Case**: Interface to Nokia 5110 LCD (PCD8544)
- **Configuration**:
  - SPI Mode 0 (CPOL=0, CPHA=0)
  - Clock up to 4 MHz
  - 8-bit transfers
  - FIFO support
- **Integration**: Direct Wishbone connection
- **Memory Map**: `0x3001_0000` - `0x3001_00FF`
- **Registers Required**:
  - Control Register
  - Data Register (TX/RX FIFO)
  - Status Register
- **Notes**: Perfect match for LCD SPI interface
- **Recommendation**: **USE AS-IS**

---

### ✅ 3. CF_TMR32 (32-bit Timer with PWM)
- **Location**: `/nc/ip/CF_TMR32`
- **Version**: v1.1.0
- **Status**: ✅ **VERIFIED & READY**
- **Use Cases**:
  1. **Timer**: Periodic interrupts for pet state updates (every 15 min)
  2. **PWM**: Audio tone generation for piezo buzzer (1-4 kHz)
- **Features**:
  - 32-bit counter
  - Programmable prescaler
  - Compare match interrupts
  - PWM output with configurable duty cycle
- **Integration**: Direct Wishbone connection
- **Memory Map**: `0x3002_0000` - `0x3002_007F`
- **Configuration**:
  - Timer mode: Count up, interrupt on overflow
  - PWM mode: Generate audio tones (frequency 1-4 kHz)
- **Notes**: Dual-purpose IP covers both timing and audio needs
- **Recommendation**: **USE AS-IS**

---

### ✅ 4. EF_GPIO8 (8-bit GPIO Controller)
- **Location**: `/nc/ip/EF_GPIO8`
- **Version**: v1.1.0
- **Status**: ✅ **VERIFIED & READY**
- **Use Case**: Button inputs and control signals
- **Configuration**:
  - 4 inputs: Buttons (SELECT, FEED, PLAY, MEDICINE)
  - 4 outputs: LCD control (RST, D/C, CS, Backlight)
- **Features**:
  - Configurable direction per pin
  - Pull-up/pull-down support
  - Interrupt on edge detection (button press)
- **Integration**: Direct Wishbone connection
- **Memory Map**: `0x3000_0000` - `0x3000_003F`
- **Notes**: May need multiple instances or expansion to 16 GPIOs
- **Recommendation**: **USE AS-IS** (use 1-2 instances for 8-16 pins)

---

### ✅ 5. CF_SRAM_1024x32 (1K × 32-bit SRAM)
- **Location**: `/nc/ip/CF_SRAM_1024x32`
- **Version**: v1.2.0
- **Status**: ✅ **VERIFIED & READY**
- **Capacity**: 4 KB (1024 words × 32 bits)
- **Use Cases**:
  1. **Display Frame Buffer**: 504 bytes (84×48 pixels / 8)
  2. **Sprite Storage**: 256 bytes (16×16 sprites × 4 frames)
  3. **Working Memory**: ~3 KB for firmware variables
- **Features**:
  - Single-port SRAM
  - Synchronous read/write
  - Wishbone interface
- **Integration**: Direct Wishbone connection
- **Memory Map**: `0x3003_0000` - `0x3003_0FFF`
- **Notes**: Sufficient for frame buffer and working memory
- **Recommendation**: **USE AS-IS**

---

## Available IPs Requiring Modifications

### ⚠️ 6. EF_I2C (I2C Master) - Optional Alternative
- **Location**: `/nc/ip/EF_I2C`
- **Version**: v1.1.0
- **Status**: ⚠️ **AVAILABLE BUT NOT PREFERRED**
- **Use Case**: Interface to I2C EEPROM (AT24C32)
- **Issue**: 
  - I2C IP exists but may be overkill for simple EEPROM
  - Bit-banged I2C via GPIO is simpler and lighter
  - EEPROM can also be SPI-based (use CF_SPI)
- **Recommendation**: 
  - **Option A**: Use bit-banged I2C via GPIO (firmware implementation)
  - **Option B**: Use SPI EEPROM instead and reuse CF_SPI
  - **Option C**: If using EF_I2C, USE AS-IS with Wishbone integration

**Decision**: **Use bit-banged I2C via GPIO** or switch to SPI EEPROM to minimize IP count.

---

## IP Gaps - Custom Development Required

### ❌ 1. Pet State Machine Controller
**Status**: ❌ **MUST DEVELOP**

**Purpose**: Hardware accelerator for pet state calculations

**Requirements**:
- **Inputs**:
  - Current health (8-bit)
  - Current hunger (8-bit)
  - Current happiness (8-bit)
  - Current age (16-bit)
  - Time delta since last update (16-bit)
  - User action (4-bit: NONE, FEED, PLAY, MEDICINE)
- **Outputs**:
  - Updated health (8-bit)
  - Updated hunger (8-bit)
  - Updated happiness (8-bit)
  - Updated age (16-bit)
  - Status flags (sick, sleeping, dead)
- **Logic**:
  - Hunger increases over time (configurable rate)
  - Happiness decreases over time (configurable rate)
  - Health decreases if hunger high or happiness low
  - Actions modify states (e.g., FEED reduces hunger)
  - Random events can trigger sickness

**Wishbone Registers**:
| Address Offset | Register Name      | Description                    |
|----------------|--------------------|--------------------------------|
| 0x00           | CTRL               | Control register               |
| 0x04           | HEALTH             | Current health (R/W)           |
| 0x08           | HUNGER             | Current hunger (R/W)           |
| 0x0C           | HAPPINESS          | Current happiness (R/W)        |
| 0x10           | AGE                | Current age in days (R/W)      |
| 0x14           | TIME_DELTA         | Time since last update (W)     |
| 0x18           | ACTION             | User action to apply (W)       |
| 0x1C           | STATUS             | Status flags (R)               |
| 0x20           | UPDATE_TRIGGER     | Trigger state update (W)       |

**Design Complexity**: Medium
- Estimated gates: 500-1000
- Combinational logic for state calculations
- Registered outputs

**Alternative**: Implement entirely in firmware (recommended for v1)

**Recommendation**: 
- **Phase 1**: Implement in **firmware only** (no custom RTL)
- **Phase 2**: Hardware accelerate if performance bottleneck

---

### ❌ 2. LFSR Random Number Generator
**Status**: ❌ **MUST DEVELOP**

**Purpose**: Pseudo-random number generation for gameplay variety

**Requirements**:
- **Type**: 16-bit Linear Feedback Shift Register
- **Polynomial**: x^16 + x^14 + x^13 + x^11 + 1 (maximal length)
- **Period**: 65535 (2^16 - 1)
- **Seed Source**: Power-on time, button press timing
- **Output**: 16-bit random value

**Wishbone Registers**:
| Address Offset | Register Name | Description                 |
|----------------|---------------|-----------------------------|
| 0x00           | CTRL          | Control (enable, reset)     |
| 0x04           | SEED          | Seed value (W)              |
| 0x08           | VALUE         | Current random value (R)    |

**Verilog Pseudocode**:
```verilog
always @(posedge clk or negedge rst_n) begin
  if (!rst_n)
    lfsr <= 16'hACE1; // Non-zero seed
  else if (seed_load)
    lfsr <= seed_value;
  else if (enable)
    lfsr <= {lfsr[14:0], lfsr[15] ^ lfsr[13] ^ lfsr[12] ^ lfsr[10]};
end

assign random_out = lfsr;
```

**Design Complexity**: Low
- Estimated gates: 50-100
- Simple shift register with XOR feedback
- 1 cycle per new random number

**Recommendation**: **DEVELOP SIMPLE RTL** (< 1 hour effort)

---

### ❌ 3. Display Frame Buffer Manager (Optional)
**Status**: ❌ **OPTIONAL - LOW PRIORITY**

**Purpose**: Hardware accelerated sprite blitting and frame buffer compositing

**Requirements**:
- Fast copy of sprite data to frame buffer
- Sprite positioning (X, Y coordinates)
- Transparency support (background preservation)
- DMA-like access to SRAM

**Why Optional**:
- Frame buffer is only 504 bytes (small enough for firmware)
- SPI transfer to LCD is the bottleneck, not frame buffer update
- Firmware implementation is adequate for 84×48 display

**Recommendation**: **DO NOT DEVELOP** (implement in firmware)

**If needed later**:
- Estimated gates: 1000-2000
- DMA engine + blitter logic
- Complexity: High

---

## IP Integration Summary Table

| IP Block                     | Source                | Status        | Development Effort | Integration Complexity |
|------------------------------|-----------------------|---------------|--------------------|-----------------------|
| **CF_UART**                  | `/nc/ip/CF_UART`      | ✅ Ready      | None (reuse)       | Low                   |
| **CF_SPI**                   | `/nc/ip/CF_SPI`       | ✅ Ready      | None (reuse)       | Low                   |
| **CF_TMR32**                 | `/nc/ip/CF_TMR32`     | ✅ Ready      | None (reuse)       | Low                   |
| **EF_GPIO8**                 | `/nc/ip/EF_GPIO8`     | ✅ Ready      | None (reuse)       | Low                   |
| **CF_SRAM_1024x32**          | `/nc/ip/CF_SRAM_1024x32` | ✅ Ready   | None (reuse)       | Low                   |
| **I2C/EEPROM Interface**     | Bit-bang via GPIO     | ⚠️ Firmware   | Low (bit-bang)     | Low                   |
| **Pet State Machine**        | Custom RTL (Phase 2)  | ❌ Gap        | Medium (firmware)  | Medium                |
| **LFSR RNG**                 | Custom RTL            | ❌ Gap        | Low (~1-2 hours)   | Low                   |
| **Display Manager**          | Firmware              | ❌ Not needed | None (skip)        | N/A                   |

---

## Development Roadmap

### Phase 1: Minimal Viable Product (MVP)
**Goal**: Working Tamagotchi with firmware-only game logic

**On-Chip Components**:
1. ✅ CF_SPI (LCD interface)
2. ✅ EF_GPIO8 (buttons + control)
3. ✅ CF_TMR32 (timer + PWM audio)
4. ✅ CF_SRAM_1024x32 (frame buffer + memory)
5. ❌ **LFSR RNG** (develop simple RTL)
6. 🔧 CF_UART (debug only)

**Game Logic**: Entirely in firmware (no pet state machine RTL)

**Estimated Development Time**:
- LFSR RTL: 2 hours
- Firmware: 20-40 hours
- Integration: 10 hours
- **Total**: ~1-2 weeks

---

### Phase 2: Hardware Acceleration (Optional)
**Goal**: Offload pet state calculations to hardware

**Additional Components**:
1. ❌ **Pet State Machine Controller** (develop RTL)

**Benefits**:
- Reduced firmware complexity
- Deterministic state update timing
- Lower CPU load (better power efficiency)

**Estimated Development Time**:
- Pet State Machine RTL: 10-20 hours
- Verification: 10 hours
- Integration: 5 hours
- **Total**: ~1 week

---

## IP Reuse Rate

**Total IP Blocks Needed**: 6-8

**Reuse from NativeChips Library**: 5 IPs (62-83%)

**Custom Development Required**: 1-3 blocks (17-38%)

**Reuse Rate**: **~75% IP reuse, 25% custom development**

This is an **excellent reuse rate** and demonstrates the value of the NativeChips IP library.

---

## Recommendations

### Immediate Actions
1. ✅ **Use existing IPs as-is**: CF_SPI, CF_TMR32, EF_GPIO8, CF_SRAM_1024x32, CF_UART
2. ❌ **Develop LFSR RNG**: Simple 16-bit LFSR with Wishbone interface (~2 hours)
3. ⚠️ **Implement I2C in firmware**: Bit-bang I2C via GPIO for EEPROM (simpler than IP integration)
4. ✅ **Skip hardware pet state machine**: Implement in firmware for MVP

### Long-Term Optimizations
1. Consider hardware pet state machine if firmware performance is insufficient
2. Evaluate power consumption and optimize sleep modes
3. Profile firmware to identify bottlenecks for hardware acceleration

### IP Library Gaps Identified
**For Future NativeChips IP Development**:
1. **Simple LFSR/PRNG IP**: Useful for many projects (gaming, crypto, simulation)
2. **I2C Bit-Bang Helper**: Firmware library or lightweight RTL
3. **Low-power RTC IP**: Dedicated real-time clock for battery applications

---

## Conclusion

The NativeChips IP library provides **excellent coverage** for the Tamagotchi project. With **5 verified IPs** available for direct reuse and only **1 simple custom block** (LFSR RNG) required for MVP, the project can proceed rapidly.

**Key Findings**:
- ✅ **75% IP reuse rate** - strong library coverage
- ✅ **All critical peripherals available** (SPI, GPIO, Timer, SRAM)
- ✅ **Low integration risk** - proven Wishbone interfaces
- ❌ **Minimal custom development** - only LFSR RNG needed
- ✅ **Fast time-to-market** - 1-2 weeks for MVP

**Next Steps**:
1. Finalize memory map and Wishbone addressing
2. Develop LFSR RNG RTL
3. Begin firmware development
4. Create Caravel integration plan

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Author**: NativeChips Agent
