# Tamagotchi on Caravel SoC

## Project Overview

This project implements a complete Tamagotchi digital pet product using the Efabless Caravel SoC platform with Google/Skywater 130nm Open PDK.

## Initial User Requirements

**User Request:** "I want to create a tamagoshi full product. I want you to give me the full architecture, IP gaps, and what needs to be on chip and what needs to be off chip. I'm going to be using Caravel SoC"

## Project Objectives

1. **Complete System Architecture**: Design a full Tamagotchi system with all required functional blocks
2. **IP Gap Analysis**: Identify which IPs from NativeChips library can be reused vs. what needs development
3. **On-chip vs Off-chip Partitioning**: Determine optimal resource allocation for Caravel SoC constraints
4. **Production-Ready Design**: Deliver synthesis-ready RTL, verification environment, and hardening configuration

## Tamagotchi Product Requirements

A Tamagotchi digital pet requires:

### Core Features
- **Virtual Pet State Machine**: Pet health, happiness, hunger levels
- **User Interaction**: Button inputs for feeding, playing, medicine
- **Display Output**: Graphics/LCD interface for pet animation and status
- **Real-Time Clock**: Time tracking for pet aging and care intervals
- **Audio Output**: Beeps and alerts for attention needs
- **Persistent Storage**: Save pet state across power cycles
- **Random Events**: Pseudo-random number generation for gameplay variety

### Peripheral Requirements
- Multi-button input interface (3-4 buttons minimum)
- Display controller (LCD/OLED interface)
- Speaker/buzzer control
- Non-volatile memory interface
- Clock generation and timekeeping
- Power management for battery operation

## Project Structure

```
nc-project-20260206-214505/
├── README.md                    # This file
├── docs/                        # Architecture and design documentation
│   ├── architecture.md          # System architecture
│   ├── ip_gap_analysis.md       # IP reuse analysis
│   ├── memory_map.md            # Address space allocation
│   └── retrospective.md         # Post-project review
├── rtl/                         # RTL source files
├── verilog/                     # Caravel integration files
│   ├── rtl/                     # RTL for Caravel
│   └── gl/                      # Gate-level netlists
├── verification/                # Testbenches and verification
├── openlane/                    # OpenLane synthesis configs
└── firmware/                    # Embedded firmware for management SoC
```

## Current Status

**Stage**: Architecture Design - ✅ **COMPLETE**  
**Progress**: 20% (Architecture phase complete)

## Completed Deliverables

✅ **System Architecture** - Complete block diagram, interfaces, and data flows  
✅ **IP Gap Analysis** - 75% IP reuse identified, only 1 simple custom block needed  
✅ **On-Chip/Off-Chip Partitioning** - Detailed component placement decisions  
✅ **Memory Map** - Complete Wishbone address space and register definitions  
✅ **Project Dashboard** - Comprehensive status tracking and metrics  

## Key Findings

### Architecture Highlights
- **CPU-centric design** leveraging Caravel's PicoRV32 RISC-V for game logic
- **75% IP reuse** from NativeChips library (5 verified IPs)
- **Minimal custom development**: Only LFSR RNG required for MVP
- **Low chip utilization**: ~1.6% of Caravel user area
- **Affordable BOM**: ~$6 for external components

### On-Chip Components
1. ✅ PicoRV32 CPU (Caravel built-in)
2. ✅ Wishbone Bus (Caravel built-in)
3. ✅ GPIO Controller (EF_GPIO8 IP)
4. ✅ SPI Master (CF_SPI IP)
5. ✅ Timer/PWM (CF_TMR32 IP)
6. ✅ SRAM 4KB (CF_SRAM_1024x32 IP)
7. ✅ UART Debug (CF_UART IP)
8. ❌ LFSR RNG (Custom - 2-4 hours development)
9. 🟡 Pet State Machine (Custom - Optional Phase 2)

### Off-Chip Components
- Nokia 5110 LCD (84×48 px) - SPI interface
- 4× Push buttons - GPIO inputs
- Piezo buzzer - PWM output
- I2C EEPROM 4KB - Non-volatile storage
- 32.768 kHz crystal - RTC source
- Battery + power supply

## Next Steps

### Phase 1: MVP Implementation (1-2 weeks)
1. Setup Caravel project structure
2. Develop LFSR RNG RTL (~2-4 hours)
3. Create Wishbone wrapper integrating all IPs
4. Develop firmware (LCD driver, game logic, save/load)
5. Verification with Caravel-Cocotb
6. OpenLane hardening

### Phase 2: Optimization (Optional, 1-2 weeks)
1. Hardware pet state machine accelerator
2. Power optimization and sleep modes
3. Performance tuning

---

**Project Start Date**: 2026-02-06  
**Target PDK**: Sky130 (130nm)  
**Platform**: Efabless Caravel SoC