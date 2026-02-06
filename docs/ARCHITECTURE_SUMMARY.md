# Tamagotchi on Caravel - Architecture Summary

## 🎯 Quick Reference Guide

This document provides a high-level summary of the complete Tamagotchi architecture for quick reference.

---

## 📊 System Overview at a Glance

### Platform
- **SoC**: Efabless Caravel (Sky130 PDK)
- **CPU**: PicoRV32 RISC-V @ 10-25 MHz
- **Bus**: Wishbone B4
- **User Area**: ~3000µm × 3600µm (~10.8M µm²)
- **Utilization**: ~1.6% (plenty of room!)

### Key Metrics
| Metric | Value |
|--------|-------|
| **IP Reuse Rate** | 75% (5/6-7 components) |
| **Custom Development** | 1 block (LFSR RNG) |
| **Development Time** | 1-2 weeks MVP |
| **External BOM Cost** | ~$6 |
| **GPIO Pins Used** | 16-18 of 38 available |
| **On-Chip Area** | ~170K µm² (1.6%) |
| **Battery Life Target** | 2-4 weeks |

---

## 🏗️ System Block Diagram (Simplified)

```
                    ┌─────────────────────────────────┐
                    │     CARAVEL SoC (On-Chip)       │
                    │                                 │
                    │  ┌───────────────────────────┐ │
                    │  │  PicoRV32 RISC-V CPU      │ │
                    │  │  (Game Logic Firmware)     │ │
                    │  └───────────┬───────────────┘ │
                    │              │ Wishbone Bus    │
                    │  ┌───────────┴───────────────┐ │
                    │  │    Peripheral IPs:        │ │
                    │  │  • GPIO (Buttons/Control) │ │
                    │  │  • SPI (LCD Interface)    │ │
                    │  │  • Timer/PWM (Audio)      │ │
                    │  │  • SRAM 4KB (Frame Buf)   │ │
                    │  │  • LFSR RNG (Random)      │ │
                    │  │  • UART (Debug)           │ │
                    │  └───────────┬───────────────┘ │
                    └──────────────┼─────────────────┘
                                   │ GPIO Pins
                                   │
        ┌──────────────────────────┴──────────────────────────┐
        │                                                       │
   ┌────▼─────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────▼────┐
   │ 4 Buttons│  │Nokia    │  │ Piezo   │  │ EEPROM  │  │  Power   │
   │ (Input)  │  │5110 LCD │  │ Buzzer  │  │  4KB    │  │ Battery  │
   └──────────┘  │84×48 px │  │ (Audio) │  │ (Save)  │  │   3V     │
                 └─────────┘  └─────────┘  └─────────┘  └──────────┘
```

---

## 💾 Memory Map Quick Reference

| Address | Size | Peripheral | Purpose |
|---------|------|------------|---------|
| `0x3000_0000` | 64 B | **GPIO** | Buttons + LCD control |
| `0x3001_0000` | 256 B | **SPI** | LCD data transfer |
| `0x3002_0000` | 128 B | **Timer/PWM** | Timing + audio |
| `0x3003_0000` | 4 KB | **SRAM** | Frame buffer + memory |
| `0x3004_0000` | 64 B | **LFSR RNG** | Random numbers |
| `0x3005_0000` | 256 B | **UART** | Debug interface |

### SRAM Layout (4 KB)
- `0x000-0x1F7` (504B): LCD frame buffer
- `0x200-0x2FF` (256B): Sprite storage
- `0x300-0xFFF` (3328B): Working memory

---

## 🔌 GPIO Pin Assignments

| Pin | Function | Direction | Notes |
|-----|----------|-----------|-------|
| 0-3 | Buttons | Input | SELECT/FEED/PLAY/MEDICINE |
| 4 | LCD Reset | Output | Active low |
| 5 | LCD D/C | Output | Data/Command select |
| 6 | LCD Backlight | Output | On/off control |
| 7 | Status LED | Output | Debug indicator |
| 8 | SPI CS | Output | LCD chip select |
| 9 | SPI SCLK | Output | SPI clock |
| 10 | SPI MOSI | Output | SPI data |
| 11 | PWM Out | Output | Buzzer audio |
| 12 | I2C SDA | I/O | EEPROM data |
| 13 | I2C SCL | Output | EEPROM clock |

**Total Used**: 14 pins (24 remaining for expansion)

---

## 📦 IP Component Summary

### ✅ Reused from NativeChips Library

| IP | Version | Location | Function | Status |
|----|---------|----------|----------|--------|
| **EF_GPIO8** | v1.1.0 | `/nc/ip/EF_GPIO8` | Button input + control | ✅ Ready |
| **CF_SPI** | v2.0.1 | `/nc/ip/CF_SPI` | LCD SPI interface | ✅ Ready |
| **CF_TMR32** | v1.1.0 | `/nc/ip/CF_TMR32` | Timer + PWM audio | ✅ Ready |
| **CF_SRAM_1024x32** | v1.2.0 | `/nc/ip/CF_SRAM_1024x32` | Memory + frame buffer | ✅ Ready |
| **CF_UART** | v2.0.1 | `/nc/ip/CF_UART` | Debug interface | ✅ Ready |

### ❌ Custom Development Required

| Component | Complexity | Effort | Priority |
|-----------|------------|--------|----------|
| **LFSR RNG** | Low (50-100 gates) | 2-4 hours | HIGH |
| **Wishbone Wrapper** | Low | 4-8 hours | HIGH |
| **Pet State Machine** | Medium (500-1K gates) | 10-20 hours | LOW (Phase 2) |

---

## 🔧 Off-Chip Components

### Required External Parts

| Component | Interface | Cost | Rationale |
|-----------|-----------|------|-----------|
| **Nokia 5110 LCD** | SPI | $2.50 | Display drivers too large for on-chip |
| **4× Buttons** | GPIO | $0.40 | Physical user interface |
| **Piezo Buzzer** | PWM | $0.30 | Physical audio output |
| **I2C EEPROM 4KB** | I2C | $0.50 | Non-volatile storage (no on-chip NVM) |
| **32.768 kHz Crystal** | Timer | $0.20 | Accurate RTC |
| **Battery + Holder** | Power | $1.00 | Portable power |
| **Passives** | Various | $1.00 | Caps, resistors |

**Total BOM**: ~$6.00

---

## 🎮 Game Features

### Pet Attributes (Tracked in Firmware)
- **Health**: 0-100 (dies at 0)
- **Hunger**: 0-100 (increases every 15 min)
- **Happiness**: 0-100 (decreases every 30 min)
- **Age**: Days alive

### User Actions (4 Buttons)
1. **SELECT**: Menu navigation
2. **FEED**: Reduce hunger by 30-40
3. **PLAY**: Increase happiness by 30-40
4. **MEDICINE**: Cure sickness, restore health

### Display (84×48 monochrome LCD)
- Pet sprite (16×16 animated)
- Status bars (health/hunger/happiness)
- Age display
- Alerts (visual warnings)

### Audio (Piezo Buzzer)
- Attention beeps (pet needs care)
- Button press feedback
- Happy/sad event tones

### Save System (I2C EEPROM)
- Auto-save every 5 minutes
- Persistent across power cycles
- CRC integrity checking

---

## ⚡ Power Budget

### Active Mode
- Caravel core: ~10 mA
- Peripherals: ~5 mA
- LCD (no backlight): ~2 mA
- **Total**: ~20-25 mA

### Sleep Mode
- Caravel sleep: ~0.5 mA
- RTC active: ~0.01 mA
- **Total**: ~0.5 mA

### Battery Life
- **Battery**: CR2032 (220 mAh)
- **Duty Cycle**: 1% active, 99% sleep
- **Average Current**: ~0.85 mA
- **Expected Life**: ~10-14 days (optimized: 2-4 weeks)

---

## 🚀 Development Phases

### Phase 1: MVP (1-2 weeks)
1. **Setup** (4 hours)
   - Copy Caravel template
   - Link IPs with ipm_linker
   - Setup directory structure

2. **RTL Development** (6-10 hours)
   - Develop LFSR RNG (2-4h)
   - Create Wishbone wrapper (4-6h)

3. **Firmware** (40-60 hours)
   - LCD driver (SPI)
   - Button handling
   - Pet state machine
   - Display rendering
   - EEPROM save/load
   - Game logic

4. **Verification** (10-20 hours)
   - Caravel-Cocotb tests
   - Firmware testing

5. **Hardening** (10-20 hours)
   - OpenLane macro
   - user_project_wrapper PnR

### Phase 2: Optimization (1-2 weeks) - OPTIONAL
- Hardware pet state machine
- Power optimization
- Performance tuning

---

## 📋 Checklist for Implementation

### Setup Phase
- [ ] Clone Caravel template
- [ ] Create IP link configuration
- [ ] Run ipm_linker
- [ ] Setup git repository
- [ ] Create build system

### RTL Development
- [ ] Develop LFSR RNG RTL
- [ ] Lint LFSR RNG
- [ ] Create LFSR testbench
- [ ] Create Wishbone wrapper
- [ ] Integrate all IPs
- [ ] Create user_project_wrapper

### Firmware Development
- [ ] LCD SPI driver
- [ ] GPIO button handler
- [ ] Timer interrupt handler
- [ ] Pet state machine
- [ ] Display renderer
- [ ] Sprite engine
- [ ] I2C EEPROM driver
- [ ] Save/load system
- [ ] Main game loop

### Verification
- [ ] GPIO test (buttons)
- [ ] SPI test (LCD communication)
- [ ] Timer test (interrupts)
- [ ] SRAM test (read/write)
- [ ] LFSR test (random generation)
- [ ] Integration test (full system)

### Hardening
- [ ] Configure macro OpenLane
- [ ] Run macro hardening
- [ ] Check timing/area
- [ ] Configure user_project_wrapper
- [ ] Run wrapper hardening
- [ ] Generate GDS

---

## 🎯 Success Criteria

### MVP Requirements
- ✅ Pet displays on LCD with animation
- ✅ All 4 buttons functional
- ✅ Pet stats update over time
- ✅ User actions affect pet state
- ✅ Audio feedback on buzzer
- ✅ Save/load state from EEPROM
- ✅ Synthesis clean (no critical warnings)
- ✅ Timing closure @ 10 MHz
- ✅ GDS generated successfully

### Quality Metrics
- No critical synthesis warnings
- No lint errors
- Timing slack > 0 @ 10 MHz
- Area utilization < 80% of user area
- Power budget met (< 30 mA active)

---

## 📁 Key Documents

1. **[architecture.md](architecture.md)** - Complete system architecture
2. **[ip_gap_analysis.md](ip_gap_analysis.md)** - IP reuse analysis
3. **[memory_map.md](memory_map.md)** - Address space and registers
4. **[on_chip_off_chip_partitioning.md](on_chip_off_chip_partitioning.md)** - Component placement rationale
5. **[../PROJECT_DASHBOARD.md](../PROJECT_DASHBOARD.md)** - Status tracking

---

## 🔑 Key Takeaways

### Why This Architecture Works

1. **Leverages Caravel Platform**
   - Free CPU and peripherals
   - Proven tape-out platform
   - Open-source ecosystem

2. **High IP Reuse (75%)**
   - 5 verified IPs from NativeChips
   - Minimal custom development
   - Low risk

3. **Optimal On-Chip/Off-Chip Split**
   - On-chip: Fast digital logic and control
   - Off-chip: Physical I/O and storage
   - Standard interfaces throughout

4. **Cost-Effective**
   - ~$6 external BOM
   - Uses commodity parts
   - DIY-friendly design

5. **Room to Grow**
   - Only 1.6% chip area used
   - 24 GPIO pins available
   - Easy to add features

### Risk Assessment: 🟢 LOW

- **Architecture**: ✅ Complete and validated
- **IP Availability**: ✅ 75% reuse from verified library
- **Custom Development**: ✅ Only 1 simple block (LFSR)
- **External Components**: ✅ All commodity parts
- **Timing Closure**: ✅ Conservative 10 MHz target
- **Power Budget**: ✅ Achievable with sleep modes

---

## 📞 Quick Commands Reference

### Setup Project
```bash
cd /workspace/nc-project-20260206-214505
cp -r /nc/templates/caravel_user_project/. .
mkdir -p ip && cp /nc/agent_tools/ipm_linker/link_IPs.json ip/
# Edit ip/link_IPs.json
python /nc/agent_tools/ipm_linker/ipm_linker.py --file ip/link_IPs.json --project-root .
```

### Lint RTL
```bash
verilator --lint-only --Wno-EOFNEWLINE rtl/lfsr_rng.v
```

### Run OpenLane
```bash
openlane openlane/tamagotchi_wb_wrapper/config.json --ef-save-views-to .
openlane openlane/user_project_wrapper/config.json --ef-save-views-to .
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Status**: Architecture Complete ✅  
**Next Phase**: RTL Development
