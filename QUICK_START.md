# Tamagotchi on Caravel - Quick Start Guide

## 🚀 TL;DR - What You Need to Know

This project implements a **complete Tamagotchi digital pet** on the **Efabless Caravel SoC** using **Sky130 PDK**.

**Key Facts**:
- ✅ **75% IP reuse** from NativeChips library
- ✅ Only **1 simple custom block** needed (LFSR RNG)
- ✅ **1-2 weeks** to MVP
- ✅ **~$6 external BOM** cost
- ✅ **Low risk** - proven architecture

---

## 📋 Architecture at a Glance

### What's On-Chip (Caravel SoC)
```
┌──────────────────────────────────┐
│  PicoRV32 CPU (runs game logic) │
├──────────────────────────────────┤
│  GPIO  │ Buttons + LCD control   │
│  SPI   │ LCD interface           │
│  TMR32 │ Timer + PWM audio       │
│  SRAM  │ 4KB frame buffer        │
│  LFSR  │ Random number gen       │
│  UART  │ Debug (optional)        │
└──────────────────────────────────┘
```

### What's Off-Chip (External)
- **Nokia 5110 LCD** (84×48 pixels) - Display
- **4 Buttons** - User input (SELECT/FEED/PLAY/MEDICINE)
- **Piezo Buzzer** - Audio alerts
- **I2C EEPROM 4KB** - Save game state
- **Battery** - Portable power

---

## 📚 Documentation Index

### Core Architecture Documents
1. **[README.md](README.md)** - Project overview and status
2. **[PROJECT_DASHBOARD.md](PROJECT_DASHBOARD.md)** - Comprehensive status dashboard

### Detailed Technical Docs
3. **[docs/ARCHITECTURE_SUMMARY.md](docs/ARCHITECTURE_SUMMARY.md)** - Quick reference guide
4. **[docs/architecture.md](docs/architecture.md)** - Complete system architecture (504 lines)
5. **[docs/ip_gap_analysis.md](docs/ip_gap_analysis.md)** - IP reuse analysis (369 lines)
6. **[docs/memory_map.md](docs/memory_map.md)** - Address space & registers (510 lines)
7. **[docs/on_chip_off_chip_partitioning.md](docs/on_chip_off_chip_partitioning.md)** - Component placement rationale (684 lines)
8. **[docs/block_diagrams.md](docs/block_diagrams.md)** - Visual diagrams (826 lines)

**Total Documentation**: 3,854 lines across 8 files

---

## 🎯 Quick Reference Tables

### IP Components Summary

| Component | Source | Status | Development |
|-----------|--------|--------|-------------|
| **GPIO** | EF_GPIO8 v1.1.0 | ✅ Ready | None (reuse) |
| **SPI** | CF_SPI v2.0.1 | ✅ Ready | None (reuse) |
| **Timer/PWM** | CF_TMR32 v1.1.0 | ✅ Ready | None (reuse) |
| **SRAM 4KB** | CF_SRAM_1024x32 v1.2.0 | ✅ Ready | None (reuse) |
| **UART** | CF_UART v2.0.1 | ✅ Ready | None (reuse) |
| **LFSR RNG** | Custom | ❌ Todo | 2-4 hours |
| **WB Wrapper** | Custom | ❌ Todo | 4-8 hours |

### Memory Map Quick Reference

| Address | Peripheral | Key Registers |
|---------|------------|---------------|
| `0x3000_0000` | GPIO | DIR, DATA_IN, DATA_OUT, INT_EN |
| `0x3001_0000` | SPI | CTRL, STATUS, TX_DATA, RX_DATA |
| `0x3002_0000` | Timer/PWM | CTRL, PERIOD, PWM_DUTY |
| `0x3003_0000` | SRAM | 4KB memory (frame buffer) |
| `0x3004_0000` | LFSR RNG | CTRL, SEED, VALUE |
| `0x3005_0000` | UART | CTRL, TX_DATA, RX_DATA |

### GPIO Pin Usage

| Pin | Function | Direction |
|-----|----------|-----------|
| 0-3 | Buttons (SELECT/FEED/PLAY/MEDICINE) | Input |
| 4-6 | LCD Control (RST/D/C/Backlight) | Output |
| 7 | Status LED | Output |
| 8-10 | SPI (CS/SCLK/MOSI) | Output |
| 11 | PWM (Buzzer) | Output |
| 12-13 | I2C (SDA/SCL) | I/O |

**Total Used**: 14 pins (24 remaining)

---

## 🛠️ Implementation Roadmap

### Phase 1: MVP (1-2 weeks)

#### Week 1: Setup + RTL Development
**Day 1-2: Project Setup**
```bash
# Copy Caravel template
cp -r /nc/templates/caravel_user_project/. .

# Setup IP linking
mkdir -p ip
cp /nc/agent_tools/ipm_linker/link_IPs.json ip/
# Edit link_IPs.json to include:
# - CF_UART, CF_SPI, CF_TMR32, EF_GPIO8, CF_SRAM_1024x32
python /nc/agent_tools/ipm_linker/ipm_linker.py --file ip/link_IPs.json --project-root .
```

**Day 3: Develop LFSR RNG** (2-4 hours)
```bash
# Create LFSR RTL
vim rtl/lfsr_rng.v
# Lint
verilator --lint-only --Wno-EOFNEWLINE rtl/lfsr_rng.v
# Create testbench
# Verify functionality
```

**Day 4-5: Create Wishbone Wrapper** (4-8 hours)
```bash
# Create main wrapper
vim verilog/rtl/tamagotchi_wb_wrapper.v
# Instantiate all IPs: GPIO, SPI, TMR32, SRAM, LFSR, UART
# Connect to Wishbone bus with address decoding

# Create user_project_wrapper integration
vim verilog/rtl/user_project_wrapper.v
# Instantiate tamagotchi_wb_wrapper as "mprj"
```

#### Week 2: Firmware + Verification
**Day 6-8: Core Firmware** (20-30 hours)
```bash
mkdir -p firmware/tamagotchi
cd firmware/tamagotchi

# Create source files:
# - main.c          (game loop, initialization)
# - lcd.c / lcd.h   (Nokia 5110 SPI driver)
# - pet.c / pet.h   (pet state machine)
# - gpio.c / gpio.h (button handling)
# - eeprom.c / eeprom.h (save/load)
# - timer.c / timer.h (interrupt handling)
# - Makefile        (build system)
```

**Day 9-10: Verification** (10-20 hours)
```bash
# Create Caravel-Cocotb tests
mkdir -p verification/cocotb
# Test GPIO, SPI, Timer, SRAM, LFSR individually
# Integration test: full game loop
```

**Day 11-12: Hardening** (10-20 hours)
```bash
# Configure OpenLane for wrapper macro
vim openlane/tamagotchi_wb_wrapper/config.json
openlane openlane/tamagotchi_wb_wrapper/config.json --ef-save-views-to .

# Configure OpenLane for user_project_wrapper
vim openlane/user_project_wrapper/config.json
openlane openlane/user_project_wrapper/config.json --ef-save-views-to .

# Check results
python -c "from rtl_agent_skills import view_openlane_metrics; view_openlane_metrics('openlane/user_project_wrapper')"
```

### Phase 2: Optimization (Optional, 1-2 weeks)
- Hardware pet state machine accelerator
- Power optimization (sleep modes)
- Performance tuning

---

## 🎮 Game Features

### Pet Attributes
- **Health**: 0-100 (pet dies at 0)
- **Hunger**: 0-100 (increases every 15 min)
- **Happiness**: 0-100 (decreases every 30 min)
- **Age**: Days alive

### User Actions
1. **SELECT**: Navigate menus
2. **FEED**: Reduce hunger by 30-40
3. **PLAY**: Increase happiness by 30-40
4. **MEDICINE**: Cure sickness, restore health

### Display (84×48 LCD)
- Pet sprite animation (16×16 pixels)
- Status bars (health/hunger/happiness)
- Age display
- Visual alerts

### Audio (Piezo Buzzer)
- Attention beeps (pet needs care)
- Button press feedback
- Event tones (happy/sad)

### Persistence (EEPROM)
- Auto-save every 5 minutes
- Save on sleep/power-down
- CRC integrity check

---

## 📊 Key Metrics

### Resource Utilization
- **On-Chip Area**: ~170K µm² (~1.6% of user area)
- **GPIO Pins**: 16-18 used, 20-22 available
- **SRAM**: 4 KB (504B frame buffer + 3.5KB work memory)

### Performance
- **CPU Clock**: 10-25 MHz
- **Display Refresh**: ~2-5 Hz (adequate for pet animation)
- **SPI Speed**: 4 MHz (LCD transfer ~100ms)
- **State Update**: Every 15 minutes (timer interrupt)

### Power Budget
- **Active**: ~20-30 mA (display + CPU)
- **Sleep**: ~0.5 mA (RTC active)
- **Battery Life**: 2-4 weeks with optimization

### Cost
- **External BOM**: ~$6
  - Nokia 5110 LCD: $2.50
  - Buttons: $0.40
  - Buzzer: $0.30
  - EEPROM: $0.50
  - Crystal: $0.20
  - Battery: $1.00
  - Passives: $1.00

---

## 🔧 Development Commands Cheat Sheet

### Setup
```bash
# Clone Caravel template
cp -r /nc/templates/caravel_user_project/. /workspace/nc-project-20260206-214505/

# Link IPs
python /nc/agent_tools/ipm_linker/ipm_linker.py --file ip/link_IPs.json --project-root .
```

### RTL Development
```bash
# Lint Verilog
verilator --lint-only --Wno-EOFNEWLINE rtl/*.v

# Parse Verilog (check syntax)
python -c "from rtl_agent_skills import parse_verilog; print(parse_verilog('rtl/lfsr_rng.v'))"
```

### Synthesis/Hardening
```bash
# Run OpenLane for macro
openlane openlane/tamagotchi_wb_wrapper/config.json --ef-save-views-to .

# Run OpenLane for wrapper
openlane openlane/user_project_wrapper/config.json --ef-save-views-to .

# View metrics
python -c "from rtl_agent_skills import view_openlane_metrics, get_latest_run_dir; print(view_openlane_metrics('openlane/user_project_wrapper'))"
```

### Verification
```bash
# Run Caravel-Cocotb tests
cd verification/cocotb
make
```

### Query Documentation
```python
from rtl_agent_skills import query_docs_db
print(query_docs_db("Caravel Wishbone integration"))
print(query_docs_db("OpenLane timing closure"))
```

---

## ✅ Success Criteria Checklist

### MVP Milestones
- [ ] LFSR RNG RTL developed and linted
- [ ] Wishbone wrapper created with all IPs integrated
- [ ] user_project_wrapper instantiates wrapper
- [ ] LCD driver functional (SPI communication verified)
- [ ] Button inputs working (GPIO interrupts)
- [ ] Timer interrupts trigger state updates
- [ ] Pet displays on LCD with animation
- [ ] User actions affect pet state correctly
- [ ] Audio feedback works (PWM tones)
- [ ] Save/load state from EEPROM functional
- [ ] Clean synthesis (no critical warnings)
- [ ] Timing closure @ 10 MHz
- [ ] GDS files generated successfully

### Quality Gates
- [ ] No lint errors
- [ ] No synthesis critical warnings
- [ ] Positive timing slack @ 10 MHz
- [ ] Area utilization < 80%
- [ ] All verification tests pass
- [ ] Documentation complete

---

## 🆘 Troubleshooting Resources

### Documentation to Query First
Use `query_docs_db()` for:
- Caravel integration issues
- OpenLane synthesis/PnR problems
- Wishbone bus errors
- Timing violations
- Verification failures

### Common Issues
1. **Wishbone Ack Timing**: Ensure `wbs_ack_o` responds within 1-2 cycles
2. **GPIO Interrupts**: Check interrupt enable and status registers
3. **SPI Timing**: Verify clock divider matches LCD specs
4. **SRAM Access**: Ensure proper address alignment (word-aligned)
5. **Timing Closure**: Reduce clock frequency or add pipeline stages

### Key Contact Points
- **Architecture Questions**: Refer to `docs/architecture.md`
- **Memory Map Issues**: Refer to `docs/memory_map.md`
- **IP Integration**: Refer to `docs/ip_gap_analysis.md`
- **Component Placement**: Refer to `docs/on_chip_off_chip_partitioning.md`

---

## 🎯 Next Steps

### Immediate Actions (Today)
1. Review all documentation (start with PROJECT_DASHBOARD.md)
2. Understand architecture (read ARCHITECTURE_SUMMARY.md)
3. Familiarize with memory map (skim memory_map.md)

### This Week
1. Setup project (copy Caravel template, link IPs)
2. Develop LFSR RNG RTL
3. Create Wishbone wrapper

### Next Week
1. Firmware development (LCD driver, game logic)
2. Verification
3. Hardening

### Week 3-4
1. Testing and refinement
2. Power optimization
3. Final GDS generation

---

## 📞 Quick Links

| Document | Purpose | Lines |
|----------|---------|-------|
| [README.md](README.md) | Project overview | 117 |
| [PROJECT_DASHBOARD.md](PROJECT_DASHBOARD.md) | Status tracking | 468 |
| [docs/ARCHITECTURE_SUMMARY.md](docs/ARCHITECTURE_SUMMARY.md) | Quick reference | 376 |
| [docs/architecture.md](docs/architecture.md) | Full architecture | 504 |
| [docs/ip_gap_analysis.md](docs/ip_gap_analysis.md) | IP reuse | 369 |
| [docs/memory_map.md](docs/memory_map.md) | Address space | 510 |
| [docs/on_chip_off_chip_partitioning.md](docs/on_chip_off_chip_partitioning.md) | Placement rationale | 684 |
| [docs/block_diagrams.md](docs/block_diagrams.md) | Visual diagrams | 826 |

---

## 💡 Pro Tips

1. **Start Simple**: Focus on MVP first, optimize later
2. **Test Early**: Verify each IP individually before integration
3. **Document As You Go**: Update docs with any architecture changes
4. **Use Verified IPs**: Don't reinvent wheels - reuse NativeChips IPs
5. **Conservative Timing**: Start with 10 MHz, increase if needed
6. **Power First**: Design with sleep modes from day 1
7. **Modular Firmware**: Keep LCD, pet logic, EEPROM separate
8. **Version Control**: Commit frequently with clear messages

---

## 🏆 Why This Architecture Wins

1. ✅ **High IP Reuse (75%)** - Fast development, low risk
2. ✅ **Proven Platform (Caravel)** - Tape-out ready
3. ✅ **Minimal Custom Logic** - Only LFSR RNG needed
4. ✅ **Standard Interfaces** - SPI, I2C, GPIO well-supported
5. ✅ **Low Cost BOM (~$6)** - Affordable prototyping
6. ✅ **Room to Grow** - 1.6% area usage, 24 GPIO pins free
7. ✅ **Well Documented** - 3,854 lines of comprehensive docs
8. ✅ **Clear Roadmap** - 1-2 week MVP path

---

**Good luck with your Tamagotchi! 🐱🎮**

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Status**: Architecture Complete ✅  
**Next Phase**: Implementation
