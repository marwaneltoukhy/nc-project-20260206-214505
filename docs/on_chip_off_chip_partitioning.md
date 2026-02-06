# On-Chip vs Off-Chip Partitioning Analysis

## Executive Summary

This document provides a detailed analysis of component placement decisions for the Tamagotchi system, explaining what should be integrated on the Caravel SoC chip and what should remain as external off-chip components.

**Summary**:
- **On-Chip**: CPU, bus, peripherals (GPIO/SPI/Timer/SRAM), custom logic (~75% of functionality)
- **Off-Chip**: Display, buttons, audio, storage, power (~25% of functionality, 100% of physical I/O)

---

## Decision Criteria

### Factors for On-Chip Integration
1. **Digital Logic Complexity**: Complex state machines, arithmetic
2. **High-Speed Requirements**: Fast data processing, low latency
3. **Area Efficiency**: Small gates/memory vs. large analog/display
4. **Power Efficiency**: Low power digital logic
5. **Integration Benefits**: Reduced interconnect, higher reliability
6. **Cost**: Proven IP blocks available

### Factors for Off-Chip Components
1. **Physical Interface**: Human interaction (buttons, screen, speaker)
2. **Analog Requirements**: Audio, power management, crystals
3. **Large Area**: Displays, large memories
4. **Flexibility**: Easy to swap/upgrade external parts
5. **Non-Volatile Storage**: EEPROM/Flash not available on-chip
6. **Cost**: Commodity parts cheaper externally

---

## On-Chip Components (Caravel SoC)

### 1. Management SoC Core (Caravel Built-in)
**Decision**: ✅ **ON-CHIP** (Already integrated)

**Component**: PicoRV32 RISC-V CPU @ 10-25 MHz

**Rationale**:
- Provided by Caravel platform (free)
- Runs game logic firmware
- Controls all peripherals via Wishbone
- ~10K gates already integrated

**Benefits**:
- Zero development effort (reuse)
- Proven, tape-out ready
- Full software control
- Debug capabilities

**Alternatives Considered**:
- ❌ External MCU: Defeats purpose of Caravel, adds cost/complexity
- ❌ Pure FSM logic: Too inflexible for game logic

---

### 2. Wishbone Bus Interconnect (Caravel Built-in)
**Decision**: ✅ **ON-CHIP** (Already integrated)

**Component**: Wishbone B4 compliant bus

**Rationale**:
- Standard bus protocol
- Connects CPU to peripherals
- Provided by Caravel

**Benefits**:
- Standard interface for all IPs
- No development needed
- Well-documented protocol

---

### 3. GPIO Controller (EF_GPIO8)
**Decision**: ✅ **ON-CHIP** (Use verified IP)

**Component**: 8-bit GPIO controller with interrupt support

**Rationale**:
- Simple digital I/O control
- Interrupt-driven button handling
- Small area (~100-200 gates per pin)
- Available as verified IP (`/nc/ip/EF_GPIO8`)

**Use Cases**:
- Button inputs (4 pins): Debouncing in firmware
- LCD control (3 pins): Reset, D/C, Backlight
- I2C control (2 pins): Bit-banged SDA/SCL
- Status LED (1 pin): Debug indicator

**Benefits**:
- Low latency input detection
- Hardware interrupt generation
- Small footprint
- Verified IP (zero development)

**Alternatives Considered**:
- ❌ External I/O expander: Adds latency, cost, complexity
- ❌ Direct pin control: Less flexible, no debounce support

---

### 4. SPI Master Controller (CF_SPI)
**Decision**: ✅ **ON-CHIP** (Use verified IP)

**Component**: SPI master with FIFO

**Rationale**:
- LCD (Nokia 5110) requires SPI interface
- Hardware SPI is faster and more reliable than bit-banging
- Small area (~500-1000 gates)
- Available as verified IP (`/nc/ip/CF_SPI`)

**Specifications**:
- Mode 0 (CPOL=0, CPHA=0)
- Up to 4 MHz clock
- 8-bit transfers
- TX/RX FIFOs

**Benefits**:
- Offload CPU from bit-banging
- Reliable timing
- FIFO reduces interrupt overhead
- Verified IP (zero development)

**Alternatives Considered**:
- ❌ Bit-bang SPI via GPIO: Slower, CPU intensive, timing issues
- ❌ External SPI bridge: Unnecessary complexity

---

### 5. Timer/PWM Controller (CF_TMR32)
**Decision**: ✅ **ON-CHIP** (Use verified IP)

**Component**: 32-bit timer with PWM output

**Rationale**:
- Dual purpose: Timing + Audio
- Small area (~500-1000 gates)
- Available as verified IP (`/nc/ip/CF_TMR32`)

**Use Cases**:
1. **Timer Mode**: Periodic interrupts for pet state updates (every 15 min)
2. **PWM Mode**: Audio tone generation (1-4 kHz) for piezo buzzer

**Benefits**:
- Hardware timing (accurate, low jitter)
- CPU offload for periodic tasks
- Direct PWM output for audio
- Single IP for two functions
- Verified IP (zero development)

**Alternatives Considered**:
- ❌ Software timing: Inaccurate, CPU intensive
- ❌ External timer chip: Unnecessary cost/complexity
- ❌ External audio DAC: Overkill for simple beeps

---

### 6. SRAM (CF_SRAM_1024x32)
**Decision**: ✅ **ON-CHIP** (Use verified IP)

**Component**: 4 KB SRAM (1024 × 32-bit)

**Rationale**:
- Frame buffer storage (504 bytes)
- Sprite data (256 bytes)
- Working memory (~3 KB)
- Fast access (single-cycle reads)
- Area-efficient on SKY130 (~10-20 µm² per bit)
- Available as verified IP (`/nc/ip/CF_SRAM_1024x32`)

**Memory Layout**:
- `0x000-0x1F7`: LCD frame buffer (84×48/8 = 504 bytes)
- `0x200-0x2FF`: Sprite storage (16×16 × 4 frames)
- `0x300-0xFFF`: Working memory / stack

**Benefits**:
- Zero-wait-state access
- No external bus latency
- Sufficient for display + working memory
- Verified IP (zero development)

**Alternatives Considered**:
- ❌ External SRAM: Requires many pins, adds latency, increases cost
- ❌ Larger SRAM: 4 KB is adequate, more wastes area

---

### 7. UART Debug Port (CF_UART) - Optional
**Decision**: ✅ **ON-CHIP** (Use verified IP, debug only)

**Component**: UART controller (115200 baud, 8N1)

**Rationale**:
- Useful for firmware debugging
- Minimal area (~500-1000 gates)
- Available as verified IP (`/nc/ip/CF_UART`)
- Can be removed in production builds

**Benefits**:
- Debug prints during development
- Firmware testing
- Verified IP (zero development)

**Production Decision**: 🟡 **OPTIONAL** - Remove in final version if pins/area needed

---

### 8. LFSR Random Number Generator (Custom)
**Decision**: ✅ **ON-CHIP** (Develop custom RTL)

**Component**: 16-bit Linear Feedback Shift Register

**Rationale**:
- Required for gameplay variety (random events)
- Extremely simple (~50-100 gates)
- Hardware implementation is trivial
- No suitable verified IP available

**Design**:
- 16-bit shift register
- Polynomial: x^16 + x^14 + x^13 + x^11 + 1
- Wishbone interface (3 registers: CTRL, SEED, VALUE)
- Maximal length sequence (period = 65535)

**Benefits**:
- Fast random number generation (1 cycle)
- Minimal area footprint
- Deterministic for testing
- Low development effort (~2-4 hours)

**Alternatives Considered**:
- ❌ Software PRNG: Slower, CPU overhead
- ❌ External entropy source: Overkill, adds complexity
- ❌ No RNG: Game would be too predictable

---

### 9. Pet State Machine Controller (Custom - Phase 2)
**Decision**: 🟡 **ON-CHIP - OPTIONAL** (Phase 2 optimization)

**Component**: Hardware accelerator for pet state calculations

**Rationale**:
- **Phase 1 (MVP)**: Implement in firmware (flexibility, faster development)
- **Phase 2 (Optimization)**: Hardware accelerator if needed

**Why Optional**:
- Firmware implementation adequate for MVP
- Calculations are not performance-critical
- Complexity: ~500-1000 gates
- Development effort: ~10-20 hours

**Decision**: ⏳ **Defer to Phase 2** - Start with firmware, hardware accelerate if bottleneck

---

### 10. Wishbone Integration Wrapper (Custom)
**Decision**: ✅ **ON-CHIP** (Develop custom RTL)

**Component**: Top-level wrapper connecting all IPs to Wishbone bus

**Rationale**:
- Required for Caravel integration
- Instantiates all IPs
- Address decoding
- Power domain connections

**Development**:
- Wrapper module: `tamagotchi_wb_wrapper.v`
- Instantiate: GPIO, SPI, Timer, SRAM, LFSR, UART
- Connect to Wishbone bus
- Map to memory addresses

**Effort**: ~4-8 hours (straightforward integration)

---

## On-Chip Area Estimate

| Component | Estimated Gates | Estimated Area (µm²) |
|-----------|----------------|----------------------|
| PicoRV32 CPU | ~10,000 | ~50,000 |
| Wishbone Bus | ~1,000 | ~5,000 |
| GPIO (EF_GPIO8) | ~200 | ~1,000 |
| SPI (CF_SPI) | ~1,000 | ~5,000 |
| Timer/PWM (CF_TMR32) | ~1,000 | ~5,000 |
| SRAM 4KB | ~32,768 bits | ~100,000 |
| UART (CF_UART) | ~500 | ~2,500 |
| LFSR RNG | ~100 | ~500 |
| Wishbone Wrapper | ~500 | ~2,500 |
| **Total** | **~15,300 gates + 4KB RAM** | **~170,000 µm²** |

**Caravel User Project Area**: ~10,800,000 µm² (3000µm × 3600µm)

**Utilization**: ~1.6% of available area

**Conclusion**: ✅ **Fits comfortably** with plenty of room for routing and future expansion

---

## Off-Chip Components

### 1. Nokia 5110 LCD Display (PCD8544)
**Decision**: ❌ **OFF-CHIP** (External component)

**Component**: 84×48 pixel monochrome LCD

**Rationale**:
- Display drivers require huge area (84×48 = 4032 pixel drivers)
- LCD panel itself is physical component
- Analog voltage generation (LCD bias)
- Commodity part ($2-3), widely available
- Standard SPI interface

**On-Chip Support**: CF_SPI controller + GPIO (D/C, Reset)

**Area Saved**: ~100,000+ gates (if implemented on-chip)

**Alternatives Considered**:
- ❌ On-chip display driver: Massive area, still needs external LCD panel
- ❌ Segment display: Less flexible, fewer pixels
- ✅ SPI LCD: Perfect balance (simple interface, cheap, low power)

**Why This Choice Wins**:
- Minimizes on-chip complexity
- Leverages commodity part
- Easy to source and replace
- Only 5 pins needed (SPI + control)

---

### 2. Push Buttons (4×)
**Decision**: ❌ **OFF-CHIP** (External components)

**Components**: 4 tactile momentary switches

**Rationale**:
- Physical human interface
- Trivial passive components (~$0.10 each)
- Pull-up resistors handle debouncing
- No benefit to on-chip integration

**On-Chip Support**: GPIO inputs with interrupt capability

**Alternatives Considered**:
- ❌ Capacitive touch: Requires analog sensing, more complex
- ❌ Rotary encoder: Overkill for simple menu navigation
- ✅ Tactile buttons: Simplest, most reliable, cheapest

---

### 3. Piezo Buzzer
**Decision**: ❌ **OFF-CHIP** (External component)

**Component**: Passive piezo element

**Rationale**:
- Physical acoustic transducer
- Requires high voltage drive (~10-20V for loud output)
- Passive component (~$0.30)
- PWM drive is sufficient

**On-Chip Support**: CF_TMR32 PWM output

**Area Saved**: No DAC needed (~1000-5000 gates)

**Alternatives Considered**:
- ❌ On-chip DAC + amplifier: Overkill, large area, needs external speaker anyway
- ❌ Active buzzer: Less flexible (fixed frequency)
- ✅ Passive piezo + PWM: Perfect for simple beeps

---

### 4. I2C EEPROM (AT24C32 or similar)
**Decision**: ❌ **OFF-CHIP** (External component)

**Component**: 4 KB I2C EEPROM

**Rationale**:
- **Non-volatile storage not available on Caravel/SKY130**
- Persistent save state required
- Cheap commodity part (~$0.50)
- Standard I2C interface (bit-bangable via GPIO)

**On-Chip Support**: Bit-banged I2C via GPIO (SDA, SCL)

**Data Stored**:
- Pet state (health, hunger, happiness, age): 8 bytes
- Timestamps: 8 bytes
- Statistics: 32 bytes
- CRC: 2 bytes
- Total: ~50 bytes (4 KB provides ample space)

**Alternatives Considered**:
- ❌ On-chip non-volatile: Not available in SKY130 digital flow
- ❌ External SPI flash: Larger than needed, less power efficient
- ❌ No save state: Unacceptable (pet dies on power loss)
- ✅ I2C EEPROM: Small, cheap, low power, perfect fit

**Why Off-Chip**:
- **Non-volatile memory requires special process** (eFuse, EEPROM cells)
- Not available in standard SKY130 digital libraries
- External EEPROM is proven, reliable solution

---

### 5. 32.768 kHz Crystal Oscillator
**Decision**: ❌ **OFF-CHIP** (External component)

**Component**: Watch crystal + load capacitors

**Rationale**:
- Accurate real-time clock source
- Ultra-low power (~1 µW)
- Requires external passive component
- Standard frequency for RTC applications
- Cheap (~$0.20)

**On-Chip Support**: Timer input pin

**Accuracy**: ±20 ppm (±1.7 sec/day)

**Alternatives Considered**:
- ❌ On-chip RC oscillator: Less accurate (±5% or worse), temperature sensitive
- ❌ Higher frequency crystal: More power, unnecessary
- ✅ 32.768 kHz crystal: Industry standard for RTC

**Why Off-Chip**:
- Crystals require external passive resonator
- On-chip oscillator circuits exist but crystal itself must be external

---

### 6. Power Supply (Battery + Regulator)
**Decision**: ❌ **OFF-CHIP** (External components)

**Components**:
- CR2032 coin cell (3V, 220mAh) or 2×AAA batteries
- 3.3V LDO regulator (if needed)
- Decoupling capacitors
- Power switch

**Rationale**:
- Physical power source
- Battery management required
- Voltage regulation
- Power domain isolation

**On-Chip Support**: Caravel power management

**Power Budget**:
- Active: ~20-30 mA
- Sleep: ~0.5-1 mA
- Battery life: ~2-4 weeks (with sleep optimization)

**Alternatives Considered**:
- ❌ USB power: Not portable
- ❌ Solar: Too complex for toy
- ✅ Battery: Standard, portable, proven

---

### 7. Passive Components
**Decision**: ❌ **OFF-CHIP** (External components)

**Components**:
- Decoupling capacitors (0.1µF, 10µF)
- Pull-up resistors (10kΩ for I2C, buttons)
- Crystal load capacitors (22pF)
- Current limiting resistors (for LEDs, backlight)

**Rationale**:
- Required for proper circuit operation
- Cannot be integrated on digital ASIC
- Standard, cheap components

---

## Off-Chip BOM Cost Estimate

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| Nokia 5110 LCD | 1 | $2.50 | $2.50 |
| Tactile buttons | 4 | $0.10 | $0.40 |
| Piezo buzzer | 1 | $0.30 | $0.30 |
| I2C EEPROM 4KB | 1 | $0.50 | $0.50 |
| 32.768 kHz crystal | 1 | $0.20 | $0.20 |
| CR2032 battery + holder | 1 | $1.00 | $1.00 |
| Passives (caps, resistors) | ~20 | $0.05 | $1.00 |
| **Total BOM** | | | **~$6.00** |

**Conclusion**: ✅ **Very affordable** for hobby/educational project

---

## Interface Pin Count

### Caravel GPIO Usage

| Interface | Pins | Direction | Signals |
|-----------|------|-----------|---------|
| **Buttons** | 4 | Input | BTN0, BTN1, BTN2, BTN3 |
| **LCD SPI** | 3 | Output | CS, SCLK, MOSI |
| **LCD Control** | 3 | Output | RST, D/C, Backlight |
| **Buzzer PWM** | 1 | Output | PWM_OUT |
| **I2C EEPROM** | 2 | I/O | SDA, SCL |
| **Status LED** | 1 | Output | LED |
| **UART Debug** | 2 | I/O | TX, RX (optional) |
| **Total Used** | **16-18** | | |
| **Caravel Available** | **38** | | |
| **Remaining** | **20-22** | | For future expansion |

**Conclusion**: ✅ **Plenty of GPIO pins available**

---

## Power Domain Analysis

### On-Chip Power Domains

| Domain | Voltage | Current (Active) | Current (Sleep) |
|--------|---------|------------------|-----------------|
| **Caravel Core** | 1.8V | ~5-10 mA | ~100-500 µA |
| **I/O Pads** | 3.3V | ~1-2 mA | ~10 µA |
| **User Project** | 1.8V | ~2-5 mA | ~50 µA |
| **Total On-Chip** | | **~10-20 mA** | **~200 µA** |

### Off-Chip Power Domains

| Component | Voltage | Current (Active) | Current (Sleep) |
|-----------|---------|------------------|-----------------|
| **LCD Display** | 3.3V | ~1-2 mA (no BL) | <1 µA |
| **LCD Backlight** | 3.3V | ~15 mA | 0 (off) |
| **EEPROM** | 3.3V | ~1 mA (active) | <5 µA |
| **Buttons/Buzzer** | 3.3V | <1 mA | 0 |
| **Total Off-Chip** | | **~20 mA** | **~10 µA** |

### Total System Power

| Mode | On-Chip | Off-Chip | Total | Battery Life (220mAh) |
|------|---------|----------|-------|----------------------|
| **Active** | ~15 mA | ~20 mA | **~35 mA** | ~6 hours |
| **Display Only** | ~15 mA | ~2 mA | **~17 mA** | ~13 hours |
| **Sleep (RTC on)** | ~0.2 mA | ~0.01 mA | **~0.5 mA** | ~18 days |

**Duty Cycle**: 1% active (user interaction), 99% sleep

**Estimated Battery Life**: 
- Average current: (0.01 × 35 mA) + (0.99 × 0.5 mA) ≈ **0.85 mA**
- Battery life: 220 mAh / 0.85 mA ≈ **260 hours** ≈ **10-11 days**

**With Optimization** (backlight off, deeper sleep):
- Target: **2-4 weeks** battery life

---

## Design Tradeoffs Summary

### On-Chip Integration Benefits
✅ High-speed digital logic  
✅ Low latency peripheral access  
✅ Compact area for control logic  
✅ Verified IP reuse (75%)  
✅ Integrated power management  
✅ Single-chip simplicity  

### Off-Chip Component Benefits
✅ Physical user interface (buttons, screen, audio)  
✅ Non-volatile storage (EEPROM)  
✅ Flexibility (easy to swap parts)  
✅ Cost-effective commodity components  
✅ Reduced chip area (no display drivers, analog)  
✅ Standard interfaces (SPI, I2C, GPIO)  

---

## Alternative Architectures Considered

### Alternative 1: External MCU + Custom ASIC
**Architecture**: External MCU runs firmware, Caravel only has custom accelerators

❌ **Rejected Reasons**:
- Defeats purpose of Caravel (has built-in CPU)
- Adds external MCU cost ($3-5)
- More complex interconnect
- Wastes Caravel management SoC

### Alternative 2: Fully Integrated System (Display On-Chip)
**Architecture**: Integrate display drivers on Caravel

❌ **Rejected Reasons**:
- Display drivers require massive area (~100K+ gates)
- LCD panel still needs to be external
- Analog bias generation complex
- Not worth the chip area

### Alternative 3: FPGA Implementation
**Architecture**: Use FPGA instead of Caravel ASIC

❌ **Rejected Reasons**:
- More expensive ($10-50 for FPGA)
- Higher power consumption
- Not the project goal (Caravel specified)
- No fabrication learning opportunity

### ✅ Selected Architecture: Caravel SoC + Minimal External Components
**Why This Wins**:
- Leverages free Caravel CPU and peripherals
- Reuses 75% from verified IP library
- Minimal custom development (1-2 blocks)
- Low-cost external BOM (~$6)
- Standard interfaces throughout
- Fits comfortably in user project area
- Production-ready design

---

## Scaling and Future Expansion

### Additional Features Possible with Current Architecture
- ✅ Add more GPIOs (use multiple EF_GPIO8 instances)
- ✅ Add color LCD (replace Nokia 5110 with color SPI display)
- ✅ Add more buttons/controls
- ✅ Add wireless (external BLE/WiFi module via SPI/UART)
- ✅ Add sensors (temperature, light via I2C/SPI)

### Chip Area Available for Expansion
- **Current utilization**: ~1.6% of user project area
- **Available area**: ~98.4% remaining
- **Possible additions**:
  - Audio codec (I2S interface)
  - More SRAM (up to 128 KB feasible)
  - Graphics accelerator (sprite engine)
  - More timers/PWMs
  - I2C/SPI controllers
  - USB controller

---

## Conclusion

### On-Chip Components (What Goes on Caravel)
✅ **CPU, Bus, Peripherals**: Provided by Caravel, zero development  
✅ **GPIO, SPI, Timer, SRAM, UART**: Reuse verified NativeChips IPs  
✅ **LFSR RNG**: Simple custom development (~2-4 hours)  
✅ **Wishbone Wrapper**: Integration glue logic (~4-8 hours)  
🟡 **Pet State Machine**: Optional Phase 2 optimization  

**Total On-Chip**: ~15K gates + 4KB SRAM, **1.6% area utilization**

### Off-Chip Components (What Stays External)
❌ **Display (LCD)**: Physical component, huge driver complexity  
❌ **Buttons**: Physical human interface  
❌ **Buzzer**: Physical acoustic transducer  
❌ **EEPROM**: Non-volatile storage not available on-chip  
❌ **Crystal**: Physical resonator for accurate RTC  
❌ **Power**: Battery and regulation  
❌ **Passives**: Caps, resistors, essential support components  

**Total Off-Chip BOM**: ~$6, **16-18 GPIO pins**

### Key Insights
1. **75% IP reuse** from NativeChips library dramatically reduces development
2. **On-chip area is plentiful** - only 1.6% used, room for expansion
3. **Off-chip components are minimal** - only essential physical I/O and storage
4. **Standard interfaces** (SPI, I2C, GPIO) keep external circuitry simple
5. **Cost-effective design** - ~$6 BOM is very affordable
6. **Production-ready** - all components are proven, available, tape-out ready

### Final Recommendation
✅ **Proceed with proposed architecture**  
✅ **On-chip**: Caravel core + 5 verified IPs + 1 simple custom block (LFSR)  
✅ **Off-chip**: LCD, buttons, buzzer, EEPROM, power  
✅ **Risk**: LOW - mostly reusing verified components  
✅ **Development time**: 1-2 weeks MVP, 3-4 weeks full system  

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Author**: NativeChips Agent
