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

**Stage**: Architecture Design  
**Progress**: 0%

## Next Steps

1. Define complete system architecture
2. Perform IP gap analysis
3. Determine on-chip/off-chip partitioning
4. Create detailed documentation

---

**Project Start Date**: 2026-02-06  
**Target PDK**: Sky130 (130nm)  
**Platform**: Efabless Caravel SoC