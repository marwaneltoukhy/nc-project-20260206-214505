# Tamagotchi System Architecture

## Table of Contents
1. [System Overview](#system-overview)
2. [Functional Requirements](#functional-requirements)
3. [Block Diagram](#block-diagram)
4. [On-Chip Components](#on-chip-components)
5. [Off-Chip Components](#off-chip-components)
6. [Interface Specifications](#interface-specifications)
7. [Data Flow](#data-flow)
8. [State Management](#state-management)

---

## System Overview

The Tamagotchi system is built around the **Efabless Caravel SoC**, which provides:
- **PicoRV32 RISC-V CPU** (management SoC core) for firmware execution
- **Wishbone bus** for peripheral interconnect
- **38 GPIO pins** for external interfaces
- **~3000μm × 3600μm user project area** for custom logic
- **Power management** and configuration logic

### Design Philosophy
- **CPU-centric architecture**: Leverage Caravel's RISC-V CPU for game logic
- **Hardware acceleration**: Use custom RTL for time-critical functions (display, timers)
- **Minimize on-chip resources**: Offload storage, display, and audio to external components
- **Low power operation**: Battery-powered design with sleep modes

---

## Functional Requirements

### FR1: Pet State Management
- Track pet attributes: Health (0-100), Hunger (0-100), Happiness (0-100), Age (days)
- Update states based on time and user actions
- Death condition if health reaches 0

### FR2: User Input
- 4 buttons: SELECT, FEED, PLAY, MEDICINE
- Debounced button detection
- Interrupt-driven input handling

### FR3: Display Output
- 84×48 pixel monochrome LCD (Nokia 5110 / PCD8544)
- Display pet sprite (16×16 pixels with animations)
- Show status bars for health/hunger/happiness
- Show age and clock

### FR4: Audio Feedback
- Piezo buzzer for alerts
- Attention beeps (pet needs care)
- Action feedback sounds
- PWM-based tone generation

### FR5: Timekeeping
- Real-time clock for pet aging
- Wake-up intervals for state updates (e.g., every 15 minutes)
- 32.768 kHz crystal oscillator for accurate timing

### FR6: Non-Volatile Storage
- Save pet state to EEPROM/Flash
- Auto-save every 5 minutes or on sleep
- Restore state on power-up

### FR7: Random Events
- LFSR-based pseudo-random number generator
- Random pet behaviors (sickness, happiness changes)
- Gameplay variety

---

## Block Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CARAVEL SoC                                 │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │            Management SoC (PicoRV32 RISC-V)                   │ │
│  │                  Tamagotchi Firmware                          │ │
│  └────────────────────────────┬──────────────────────────────────┘ │
│                               │                                     │
│                       Wishbone Bus                                  │
│         ┌─────────────┬───────┴────────┬──────────┬──────────┐    │
│         │             │                │          │          │    │
│    ┌────▼────┐  ┌─────▼─────┐  ┌──────▼─────┐ ┌─▼────┐  ┌──▼───┐ │
│    │  GPIO8  │  │ SPI Master│  │   TMR32    │ │ SRAM │  │ UART │ │
│    │   (IP)  │  │    (IP)   │  │  Timer/PWM │ │ (IP) │  │ (IP) │ │
│    └────┬────┘  └─────┬─────┘  │    (IP)    │ └──────┘  └──────┘ │
│         │             │         └──────┬─────┘                     │
│  ┏━━━━━━┷━━━━━━━━━━━━━┷━━━━━━━━━━━━━━━┷━━━━━━━━━━━━━┓            │
│  ┃        User Project Area (Custom Logic)           ┃            │
│  ┃  • Pet State Machine Controller                   ┃            │
│  ┃  • Display Frame Buffer Manager                   ┃            │
│  ┃  • LFSR Random Number Generator                   ┃            │
│  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛            │
│         │             │                │                           │
└─────────┼─────────────┼────────────────┼───────────────────────────┘
          │             │                │
    GPIO  │       SPI   │          PWM   │
     Pins │        Bus  │         Output │
          │             │                │
┌─────────▼──┐   ┌──────▼───────┐   ┌───▼────┐
│  4 Buttons │   │ Nokia 5110   │   │ Piezo  │
│  (Input)   │   │  LCD Display │   │ Buzzer │
└────────────┘   │  (PCD8544)   │   └────────┘
                 │   84×48 px   │
                 └──────────────┘
                        │
                 ┌──────▼──────┐
                 │   I2C/SPI   │
                 │   EEPROM    │
                 │   (AT24C32) │
                 │    4KB      │
                 └─────────────┘
```

---

## On-Chip Components

### 1. Management SoC (Caravel Built-in)
- **PicoRV32 RISC-V CPU** @ 10-25 MHz
- **Flash XIP Controller**: Execute firmware from external SPI flash
- **Wishbone Interconnect**: Connect peripherals
- **Function**: Run main Tamagotchi game logic firmware

### 2. GPIO Controller (EF_GPIO8 IP)
- **Source**: `/nc/ip/EF_GPIO8` (NativeChips verified IP)
- **Configuration**: 8 GPIO pins
  - 4 inputs: Button 0-3 (SELECT, FEED, PLAY, MEDICINE)
  - 4 outputs: Status LEDs or additional control
- **Features**: Interrupt on button press, debouncing in firmware
- **Wishbone Interface**: Register-mapped control

### 3. SPI Master Controller (CF_SPI IP)
- **Source**: `/nc/ip/CF_SPI` (NativeChips verified IP)
- **Function**: Interface to Nokia 5110 LCD (PCD8544 SPI)
- **Configuration**: 
  - Mode 0 (CPOL=0, CPHA=0)
  - Clock up to 4 MHz
  - 8-bit transfers
- **Wishbone Interface**: FIFO-based SPI transactions

### 4. Timer/PWM Controller (CF_TMR32 IP)
- **Source**: `/nc/ip/CF_TMR32` (NativeChips verified IP)
- **Functions**:
  - **Timer**: 32-bit timer for pet state update intervals
  - **PWM**: Generate audio tones for buzzer (1-4 kHz)
- **Configuration**: Programmable period and duty cycle
- **Interrupt**: Timer overflow for periodic state updates

### 5. SRAM Storage (CF_SRAM_1024x32 IP)
- **Source**: `/nc/ip/CF_SRAM_1024x32` (NativeChips verified IP)
- **Capacity**: 1024 × 32-bit = 4 KB
- **Function**: 
  - Display frame buffer (504 bytes for 84×48 LCD)
  - Pet sprite storage (256 bytes)
  - Working memory for firmware
- **Wishbone Interface**: Memory-mapped access

### 6. UART Debug Port (CF_UART IP - Optional)
- **Source**: `/nc/ip/CF_UART` (NativeChips verified IP)
- **Function**: Debug/development interface
- **Baud Rate**: 115200
- **Can be removed in production**

### 7. Custom Logic Blocks (To be Developed)

#### a. Pet State Machine Controller
- **Function**: Hardware accelerator for pet state calculations
- **Inputs**: Current states (health, hunger, happiness, age)
- **Outputs**: Updated states based on time delta and actions
- **Registers**:
  - `STATE_CTRL`: Control register
  - `HEALTH_REG`: Health value (8-bit)
  - `HUNGER_REG`: Hunger value (8-bit)
  - `HAPPINESS_REG`: Happiness value (8-bit)
  - `AGE_REG`: Age in days (16-bit)
  - `ACTION_REG`: User action to apply
  - `UPDATE_REG`: Trigger state update
- **Size**: ~500-1000 gates
- **Wishbone Interface**: Yes

#### b. LFSR Random Number Generator
- **Function**: 16-bit Linear Feedback Shift Register
- **Polynomial**: x^16 + x^14 + x^13 + x^11 + 1
- **Seeded from**: Power-on time or button timing
- **Registers**:
  - `LFSR_SEED`: Seed value
  - `LFSR_OUT`: Random output
  - `LFSR_CTRL`: Enable/reset
- **Size**: ~50-100 gates
- **Wishbone Interface**: Yes

#### c. Display Frame Buffer Manager (Optional HW Accelerator)
- **Function**: Fast sprite blitting and compositing
- **Can be implemented in firmware using SRAM**
- **If implemented**:
  - DMA-like access to SRAM frame buffer
  - Sprite position registers
  - Blit trigger
- **Complexity**: Medium (1000-2000 gates)
- **Decision**: Implement in firmware first, hardware accelerate if needed

---

## Off-Chip Components

### 1. Nokia 5110 LCD Display (PCD8544)
- **Resolution**: 84 × 48 pixels (monochrome)
- **Interface**: SPI (on-chip CF_SPI controller)
- **Connections**:
  - VCC: 3.3V
  - GND: Ground
  - SCE (CS): SPI Chip Select
  - RST: Reset (GPIO)
  - D/C: Data/Command (GPIO)
  - SDIN (MOSI): SPI Data In
  - SCLK: SPI Clock
  - LED: Backlight (PWM or GPIO)
- **Frame Buffer**: 504 bytes (84×48/8)
- **Cost**: ~$2-3
- **Rationale**: External display minimizes on-chip area and provides standard interface

### 2. Push Buttons (4x)
- **Type**: Tactile momentary switches
- **Connections**: GPIOs with pull-up resistors
- **Debouncing**: Firmware-based
- **Labels**: SELECT, FEED, PLAY, MEDICINE
- **Cost**: ~$0.50

### 3. Piezo Buzzer
- **Type**: Passive piezo element
- **Drive**: PWM from CF_TMR32
- **Frequency Range**: 1-4 kHz
- **Connections**: PWM output + GND
- **Cost**: ~$0.30

### 4. I2C/SPI EEPROM (AT24C32 or similar)
- **Type**: I2C EEPROM
- **Capacity**: 4 KB (32 kbit)
- **Interface**: I2C or SPI
- **Function**: Non-volatile pet state storage
- **Data Stored**:
  - Pet stats (health, hunger, happiness, age): 8 bytes
  - Birth timestamp: 4 bytes
  - Lifetime statistics: 32 bytes
  - CRC/checksum: 2 bytes
- **Connections**: I2C (SCL, SDA) via bit-banged GPIO or I2C controller
- **Cost**: ~$0.50
- **Alternative**: Use external SPI flash if already present for firmware

### 5. 32.768 kHz Crystal Oscillator
- **Type**: Watch crystal
- **Function**: Real-time clock source
- **Connections**: Dedicated RTC pins or timer input
- **Accuracy**: ±20 ppm
- **Power**: Ultra-low power for always-on timekeeping
- **Cost**: ~$0.20
- **Note**: Caravel may have internal oscillator, verify if external needed

### 6. Power Supply
- **Battery**: CR2032 coin cell (3V, 220mAh) or 2× AAA batteries
- **Regulator**: 3.3V LDO if using higher voltage
- **Power Management**: Sleep modes to extend battery life
- **Expected Life**: Several weeks to months with sleep optimization

### 7. Passive Components
- **Decoupling Capacitors**: 0.1μF, 10μF for power rails
- **Pull-up Resistors**: 10kΩ for I2C, buttons
- **Crystal Load Capacitors**: 22pF for 32.768 kHz crystal
- **Current Limiting Resistors**: For LCD backlight, LEDs

---

## Interface Specifications

### Wishbone Bus Memory Map

| Base Address | Size    | Peripheral              | IP Source        |
|--------------|---------|-------------------------|------------------|
| `0x3000_0000`| 64 B    | GPIO Controller         | EF_GPIO8         |
| `0x3001_0000`| 256 B   | SPI Master              | CF_SPI           |
| `0x3002_0000`| 128 B   | Timer/PWM               | CF_TMR32         |
| `0x3003_0000`| 4 KB    | SRAM                    | CF_SRAM_1024x32  |
| `0x3004_0000`| 64 B    | Pet State Controller    | Custom           |
| `0x3005_0000`| 16 B    | LFSR RNG                | Custom           |
| `0x3006_0000`| 256 B   | UART (Debug)            | CF_UART          |

### GPIO Pin Allocation

| GPIO Pin | Direction | Function            | Notes                    |
|----------|-----------|---------------------|--------------------------|
| GPIO 0   | Input     | Button 0 (SELECT)   | Pull-up, interrupt       |
| GPIO 1   | Input     | Button 1 (FEED)     | Pull-up, interrupt       |
| GPIO 2   | Input     | Button 2 (PLAY)     | Pull-up, interrupt       |
| GPIO 3   | Input     | Button 3 (MEDICINE) | Pull-up, interrupt       |
| GPIO 4   | Output    | LCD Reset           | Active low               |
| GPIO 5   | Output    | LCD D/C             | Data/Command select      |
| GPIO 6   | Output    | LCD Backlight       | PWM or on/off            |
| GPIO 7   | Output    | Status LED          | Debug/status indicator   |
| GPIO 8   | Output    | SPI CS (LCD)        | Chip select              |
| GPIO 9   | Output    | SPI SCLK            | SPI clock                |
| GPIO 10  | Output    | SPI MOSI            | SPI data out             |
| GPIO 11  | Output    | PWM (Buzzer)        | Audio output             |
| GPIO 12  | I/O       | I2C SDA             | EEPROM data              |
| GPIO 13  | Output    | I2C SCL             | EEPROM clock             |

### SPI Interface to Nokia 5110 LCD

| Signal | Direction | Description              |
|--------|-----------|--------------------------|
| CS     | Output    | Chip Select (active low) |
| SCLK   | Output    | SPI Clock (up to 4 MHz)  |
| MOSI   | Output    | Master Out Slave In      |
| DC     | Output    | Data(1) / Command(0)     |
| RST    | Output    | Reset (active low)       |

**SPI Configuration**:
- Mode 0: CPOL=0, CPHA=0
- 8-bit transfers
- MSB first
- Max clock: 4 MHz

### I2C Interface to EEPROM

| Signal | Direction | Description      |
|--------|-----------|------------------|
| SDA    | I/O       | Serial Data      |
| SCL    | Output    | Serial Clock     |

**I2C Configuration**:
- Standard mode: 100 kHz
- 7-bit address: 0x50 (typical for AT24C32)
- Bit-banged via GPIO or dedicated I2C controller

---

## Data Flow

### 1. Initialization Sequence
1. **Power-On Reset**: Caravel initializes, loads firmware from SPI flash
2. **Peripheral Setup**: Firmware configures GPIO, SPI, Timer, SRAM
3. **Load Pet State**: Read from EEPROM via I2C
4. **Display Init**: Initialize Nokia 5110 LCD via SPI
5. **Start Timer**: Begin RTC/timer for state updates

### 2. Main Game Loop (Firmware)
```
loop:
  1. Check button inputs (GPIO interrupts)
  2. Process user action if button pressed
  3. Update pet state (call hardware accelerator or firmware function)
  4. Update display (write frame buffer, send to LCD via SPI)
  5. Check if audio needed (play tone via PWM)
  6. Auto-save state to EEPROM every 5 minutes
  7. Enter sleep mode until next timer interrupt
  goto loop
```

### 3. Button Press Flow
```
Button Interrupt → GPIO ISR → Read button ID → 
Perform Action (FEED/PLAY/MEDICINE) → 
Update Pet State → Update Display → Play Sound
```

### 4. Timer Interrupt Flow (Every 15 Minutes)
```
Timer Interrupt → Update hunger/happiness (decrement) → 
Check if pet needs attention → Display alert → 
Play attention sound → Save state
```

### 5. Display Update Flow
```
Firmware updates frame buffer in SRAM → 
Compose pet sprite + status bars + text → 
Send frame buffer to LCD via SPI (504 bytes)
```

---

## State Management

### Pet State Variables

```c
typedef struct {
    uint8_t  health;        // 0-100
    uint8_t  hunger;        // 0-100 (100 = starving)
    uint8_t  happiness;     // 0-100
    uint16_t age_days;      // Age in days
    uint32_t birth_time;    // Unix timestamp
    uint32_t last_feed;     // Timestamp of last feeding
    uint32_t last_play;     // Timestamp of last play
    uint8_t  is_sick;       // Boolean flag
    uint8_t  is_sleeping;   // Boolean flag
    uint8_t  sprite_frame;  // Animation frame (0-3)
    uint16_t crc;           // Data integrity check
} pet_state_t;
```

### State Transitions

#### Health Calculation
- Decreases if hunger > 80 or happiness < 20
- Decreases if sick and no medicine given
- Increases slowly over time if well cared for
- Death if health reaches 0

#### Hunger Calculation
- Increases by 5-10 every 15 minutes
- Decreases by 30-40 when fed
- Caps at 100

#### Happiness Calculation
- Decreases by 5-10 every 30 minutes
- Increases by 30-40 when played with
- Decreases if hungry or sick

#### Age Calculation
- Increments every 24 hours (real-time)
- Displayed on screen

### State Persistence
- **Auto-save**: Every 5 minutes to EEPROM
- **Manual save**: On sleep/power-down
- **Restore**: On power-up, check CRC
- **Backup**: Keep previous state in case of corruption

---

## Power Considerations

### Active Mode Power Budget
- Caravel SoC Core: ~5-10 mA @ 10 MHz
- LCD Display: ~1-2 mA (backlight off), ~15 mA (backlight on)
- EEPROM: <1 mA (active), <5 μA (standby)
- Total Active: ~20-30 mA

### Sleep Mode Power Budget
- Caravel Sleep: ~100-500 μA
- LCD Sleep: <1 μA
- Timer Active: ~10 μA
- Total Sleep: ~500 μA - 1 mA

### Battery Life Estimate
- CR2032 Battery: 220 mAh
- Duty Cycle: 1% active (1 min/hour), 99% sleep
- Average Current: (0.01 × 25 mA) + (0.99 × 0.5 mA) ≈ 0.75 mA
- Battery Life: 220 mAh / 0.75 mA ≈ 293 hours ≈ **12 days continuous**
- With optimizations: **Weeks to months**

---

## Design Decisions Summary

### Why Caravel SoC?
- Provides free CPU and peripherals
- Proven open-source platform
- SKY130 PDK for fabrication
- Reduces custom RTL complexity

### Why External Display?
- On-chip display drivers would consume massive area
- SPI interface is simple and well-supported
- Nokia 5110 is cheap, available, low power
- 84×48 resolution is adequate for pixel art

### Why External EEPROM?
- Caravel doesn't have non-volatile memory
- 4 KB is sufficient for pet state
- I2C/SPI interface is trivial
- Cost-effective ($0.50)

### Why External Buzzer?
- On-chip audio would require DAC or complex PWM
- Piezo buzzer is simple passive component
- PWM from timer is sufficient

### Why Hardware Accelerators?
- Pet state machine: Offload firmware, deterministic timing
- LFSR RNG: Simple, effective, hardware-efficient
- Display manager: Optional, firmware may be sufficient

---

## Next Steps

1. **IP Gap Analysis**: Identify which IPs need development
2. **Memory Map Finalization**: Lock down addresses
3. **RTL Development**: Create custom blocks
4. **Firmware Development**: Write Tamagotchi game logic
5. **Verification**: Test with Caravel-Cocotb
6. **Integration**: Hardening with OpenLane
7. **PCB Design**: Create prototype board

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Author**: NativeChips Agent
