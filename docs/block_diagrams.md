# Tamagotchi System Block Diagrams

## Table of Contents
1. [Top-Level System Diagram](#top-level-system-diagram)
2. [On-Chip Architecture](#on-chip-architecture)
3. [Wishbone Bus Architecture](#wishbone-bus-architecture)
4. [GPIO Pin Connections](#gpio-pin-connections)
5. [Data Flow Diagrams](#data-flow-diagrams)
6. [State Machine Diagrams](#state-machine-diagrams)

---

## Top-Level System Diagram

### Complete Tamagotchi System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                         CARAVEL SoC (SKY130)                            │
│                    User Project Area: 3000µm × 3600µm                   │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │                   Management SoC (Built-in)                       │ │
│  │                                                                   │ │
│  │    ┌─────────────────────────────────────────────────┐          │ │
│  │    │       PicoRV32 RISC-V CPU @ 10-25 MHz           │          │ │
│  │    │          Tamagotchi Game Firmware                │          │ │
│  │    │                                                  │          │ │
│  │    │  • Pet State Logic (Health/Hunger/Happiness)    │          │ │
│  │    │  • Display Rendering (Sprites/Status Bars)      │          │ │
│  │    │  • Button Input Handling                        │          │ │
│  │    │  • Timer Management (15-min updates)            │          │ │
│  │    │  • Save/Load to EEPROM                          │          │ │
│  │    └──────────────────┬──────────────────────────────┘          │ │
│  │                       │                                          │ │
│  │              Wishbone Bus (32-bit)                               │ │
│  │       ┌───────────────┴──────────────────┬───────────────────┐  │ │
│  └───────┼──────────────────────────────────┼───────────────────┼──┘ │
│          │                                  │                   │    │
│  ┌───────▼────────┐  ┌──────────────────────▼──────┐  ┌────────▼──┐ │
│  │ User Project   │  │  User Project Peripherals   │  │  Debug    │ │
│  │   Wrapper      │  │                             │  │  UART     │ │
│  │                │  │  ┌─────────┐ ┌──────────┐  │  └───────────┘ │
│  │ ┌────────────┐ │  │  │ GPIO8   │ │ SPI      │  │                │
│  │ │ Tamagotchi │ │  │  │ Buttons │ │ LCD Ctrl │  │                │
│  │ │ WB Wrapper │ │  │  └────┬────┘ └─────┬────┘  │                │
│  │ │            │ │  │       │            │       │                │
│  │ │ • GPIO     │ │  │  ┌────▼────┐  ┌────▼────┐  │                │
│  │ │ • SPI      │ │  │  │ TMR32   │  │ SRAM    │  │                │
│  │ │ • TMR32    │ │  │  │ PWM     │  │ 4KB     │  │                │
│  │ │ • SRAM     │ │  │  └────┬────┘  └────┬────┘  │                │
│  │ │ • LFSR     │ │  │       │            │       │                │
│  │ └────┬───────┘ │  │  ┌────▼────┐  ┌────▼────┐  │                │
│  │      │         │  │  │ LFSR    │  │  (more) │  │                │
│  │      │         │  │  │ RNG     │  │         │  │                │
│  └──────┼─────────┘  │  └─────────┘  └─────────┘  │                │
│         │            └─────────────────────────────┘                │
│         │                                                            │
│    GPIO Pins (38 available, 16-18 used)                             │
└─────────┼────────────────────────────────────────────────────────────┘
          │
          │ Physical Connections
          │
┌─────────┴──────────────────────────────────────────────────────────────┐
│                         External Components                            │
└────────────────────────────────────────────────────────────────────────┘

┌──────────────┐  ┌─────────────────┐  ┌──────────────┐  ┌─────────────┐
│ 4× Buttons   │  │  Nokia 5110 LCD │  │ Piezo Buzzer │  │  I2C EEPROM │
│              │  │    (PCD8544)    │  │              │  │    4KB      │
│ [SELECT]     │  │                 │  │  ♪♪♪         │  │  (AT24C32)  │
│ [FEED]       │  │  ┌───────────┐  │  │              │  │             │
│ [PLAY]       │  │  │ /\_/\     │  │  │  Passive     │  │  Save Pet   │
│ [MEDICINE]   │  │  │ >^.^<  ❤❤ │  │  │  Element     │  │  State      │
│              │  │  │  / \      │  │  │              │  │             │
│ GPIO 0-3     │  │  └───────────┘  │  │  PWM Drive   │  │  I2C (SDA/  │
│ (Pull-up)    │  │  84 × 48 px     │  │  GPIO 11     │  │       SCL)  │
│              │  │                 │  │              │  │  GPIO 12-13 │
└──────────────┘  │  SPI Interface: │  └──────────────┘  └─────────────┘
                  │  • CS (GPIO 8)  │
                  │  • SCLK (GPIO 9)│  ┌─────────────┐  ┌─────────────┐
                  │  • MOSI (GPIO10)│  │ 32.768 kHz  │  │   Battery   │
                  │  • D/C (GPIO 5) │  │   Crystal   │  │             │
                  │  • RST (GPIO 4) │  │             │  │  CR2032     │
                  │  • BL (GPIO 6)  │  │  RTC Source │  │  3V 220mAh  │
                  └─────────────────┘  │  Timer Input│  │             │
                                       └─────────────┘  └─────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│                         Power Supply                                  │
│                                                                       │
│   Battery → LDO Regulator (3.3V) → Decoupling → Caravel + External   │
│                                                                       │
│   Expected Life: 2-4 weeks with sleep optimization                   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## On-Chip Architecture

### Detailed Caravel User Project Internal Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Caravel User Project Wrapper                      │
│                                                                     │
│  Instance: "mprj"                                                   │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │            tamagotchi_wb_wrapper (Macro)                      │ │
│  │                                                               │ │
│  │  Power: VPWR (vccd2), VGND (vssd2)                           │ │
│  │                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────┐ │ │
│  │  │                 Wishbone Slave Interface                 │ │
│  │  │                                                          │ │ │
│  │  │  Inputs:  wb_clk_i, wb_rst_i                            │ │ │
│  │  │           wbs_stb_i, wbs_cyc_i, wbs_we_i                │ │ │
│  │  │           wbs_sel_i[3:0], wbs_adr_i[31:0]               │ │ │
│  │  │           wbs_dat_i[31:0]                               │ │ │
│  │  │  Outputs: wbs_ack_o, wbs_dat_o[31:0]                    │ │ │
│  │  └────────────────────┬────────────────────────────────────┘ │ │
│  │                       │                                       │ │
│  │              Address Decoder Logic                            │ │
│  │       (Routes access to correct peripheral based on address) │ │
│  │                       │                                       │ │
│  │       ┌───────────────┴───────────────────────┐              │ │
│  │       │                                       │              │ │
│  │  ┌────▼─────┐  ┌────────┐  ┌──────────┐  ┌──▼────────┐    │ │
│  │  │ EF_GPIO8 │  │ CF_SPI │  │ CF_TMR32 │  │CF_SRAM    │    │ │
│  │  │  (IP)    │  │  (IP)  │  │   (IP)   │  │1024x32(IP)│    │ │
│  │  │          │  │        │  │          │  │           │    │ │
│  │  │ 0x3000_  │  │0x3001_ │  │ 0x3002_  │  │ 0x3003_   │    │ │
│  │  │  0000    │  │  0000  │  │   0000   │  │   0000    │    │ │
│  │  │          │  │        │  │          │  │           │    │ │
│  │  │ • DIR    │  │• CTRL  │  │• CTRL    │  │ 4KB       │    │ │
│  │  │ • DATA_IN│  │• STATUS│  │• PRESCALE│  │ Memory    │    │ │
│  │  │ • DATA_  │  │• TX_    │  │• PERIOD  │  │           │    │ │
│  │  │   OUT    │  │  DATA  │  │• PWM_DUTY│  │ Frame Buf │    │ │
│  │  │ • PULLUP │  │• RX_    │  │• INT_STAT│  │ + Sprites │    │ │
│  │  │ • INT_EN │  │  DATA  │  │          │  │ + Work    │    │ │
│  │  │ • INT_   │  │• CLK_  │  │ Timer:   │  │   Memory  │    │ │
│  │  │   STATUS │  │  DIV   │  │  15 min  │  │           │    │ │
│  │  │          │  │        │  │ PWM:     │  │           │    │ │
│  │  │ Buttons: │  │ SPI    │  │  Audio   │  │           │    │ │
│  │  │  BTN[3:0]│  │ Mode 0 │  │  1-4 kHz │  │           │    │ │
│  │  │ Control: │  │ 4 MHz  │  │          │  │           │    │ │
│  │  │  LCD_RST │  │        │  └─────┬────┘  └───────────┘    │ │
│  │  │  LCD_DC  │  │        │        │                         │ │
│  │  │  LCD_BL  │  │        │    PWM_OUT                       │ │
│  │  │  LED     │  │        │   (GPIO 11)                      │ │
│  │  └──────────┘  └────┬───┘                                  │ │
│  │                     │                                       │ │
│  │                 SPI Signals                                 │ │
│  │                (GPIO 8-10)                                  │ │
│  │                     │                                       │ │
│  │  ┌──────────┐  ┌───▼──────┐                               │ │
│  │  │ LFSR_RNG │  │ CF_UART  │                               │ │
│  │  │ (Custom) │  │   (IP)   │                               │ │
│  │  │          │  │          │                               │ │
│  │  │ 0x3004_  │  │ 0x3005_  │                               │ │
│  │  │  0000    │  │  0000    │                               │ │
│  │  │          │  │          │                               │ │
│  │  │ • CTRL   │  │ • CTRL   │                               │ │
│  │  │ • SEED   │  │ • STATUS │                               │ │
│  │  │ • VALUE  │  │ • TX_DATA│                               │ │
│  │  │          │  │ • RX_DATA│                               │ │
│  │  │ 16-bit   │  │ • BAUD_  │                               │ │
│  │  │ LFSR     │  │   DIV    │                               │ │
│  │  │          │  │          │                               │ │
│  │  │ x^16+    │  │ 115200   │                               │ │
│  │  │ x^14+    │  │ baud     │                               │ │
│  │  │ x^13+    │  │          │                               │ │
│  │  │ x^11+1   │  │ Debug    │                               │ │
│  │  │          │  │ Only     │                               │ │
│  │  └──────────┘  └──────────┘                               │ │
│  │                                                            │ │
│  │  All IPs connected via internal Wishbone bus segments     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Connection to Caravel Wishbone Bus via user_project_wrapper   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Wishbone Bus Architecture

### Address Decoding and Bus Topology

```
                           PicoRV32 CPU
                                │
                                │ (Master)
                                │
                        ┌───────▼────────┐
                        │  Wishbone Bus  │
                        │   Crossbar     │
                        │                │
                        │  Address       │
                        │  Decoder       │
                        └───────┬────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼─────┐  ┌──────▼──────┐  ┌────▼──────┐
        │ 0x3000_0000 │  │ 0x3001_0000 │  │0x3002_0000│
        │             │  │             │  │           │
        │   GPIO8     │  │   CF_SPI    │  │ CF_TMR32  │
        │             │  │             │  │           │
        │ • REG[15:0] │  │ • REG[63:0] │  │• REG[31:0]│
        │             │  │             │  │           │
        └─────────────┘  └─────────────┘  └───────────┘

                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼─────┐  ┌──────▼──────┐  ┌────▼──────┐
        │ 0x3003_0000 │  │ 0x3004_0000 │  │0x3005_0000│
        │             │  │             │  │           │
        │ CF_SRAM     │  │  LFSR_RNG   │  │ CF_UART   │
        │  1024x32    │  │             │  │           │
        │             │  │ • REG[7:0]  │  │• REG[15:0]│
        │ 4096 bytes  │  │             │  │           │
        └─────────────┘  └─────────────┘  └───────────┘

Wishbone Transaction Example (Button Read):
────────────────────────────────────────────
1. CPU writes address 0x3000_0004 to wbs_adr_i (GPIO DATA_IN)
2. CPU asserts wbs_stb_i, wbs_cyc_i (transaction start)
3. Address decoder routes to GPIO8 module
4. GPIO8 drives wbs_dat_o with button state
5. GPIO8 asserts wbs_ack_o (transaction complete)
6. CPU reads wbs_dat_o, gets button values [BTN3:BTN0]

Address Map Summary:
────────────────────
Base         | Size  | Peripheral    | Slave Select
─────────────┼───────┼───────────────┼─────────────
0x3000_0000  | 64 B  | GPIO          | SS0
0x3001_0000  | 256 B | SPI           | SS1
0x3002_0000  | 128 B | Timer/PWM     | SS2
0x3003_0000  | 4 KB  | SRAM          | SS3
0x3004_0000  | 64 B  | LFSR RNG      | SS4
0x3005_0000  | 256 B | UART          | SS5
```

---

## GPIO Pin Connections

### Physical Pin Mapping

```
Caravel SoC                                    External Components
(GPIO Pads)                                    
                                               
GPIO[37:14]  ◄──────  Reserved / Unused
                      
GPIO[13]     ◄─────►  I2C SCL ───────────────► EEPROM SCL (Clock)
                      (Bit-bang)               AT24C32
GPIO[12]     ◄─────►  I2C SDA ───────────────► EEPROM SDA (Data)
                      (Bit-bang)
                      
GPIO[11]     ──────►  PWM_OUT ────────────────► Piezo Buzzer (+)
                      (TMR32 PWM)              Passive Element
                                               GND (-)
                      
GPIO[10]     ──────►  SPI_MOSI ───────────────► LCD SDIN (Data In)
                                               Nokia 5110
GPIO[9]      ──────►  SPI_SCLK ───────────────► LCD SCLK (Clock)
                                               
GPIO[8]      ──────►  SPI_CS ─────────────────► LCD SCE (Chip Select)
                      (Active Low)
                      
GPIO[7]      ──────►  STATUS_LED ──────────────► LED + Resistor → GND
                      (Debug)
                      
GPIO[6]      ──────►  LCD_BL ──────────────────► LCD Backlight Enable
                      (Backlight Ctrl)
                      
GPIO[5]      ──────►  LCD_DC ──────────────────► LCD D/C (Data/Cmd)
                      (Data=1, Cmd=0)
                      
GPIO[4]      ──────►  LCD_RST ─────────────────► LCD Reset
                      (Active Low)              (Active Low)
                      
GPIO[3]      ◄──────  BTN3 ◄──┐                 
                               │  Pull-up        [MEDICINE] Button
                               ├─ 10kΩ ── VCC   (Momentary Switch)
                               └─ Switch ── GND
                      
GPIO[2]      ◄──────  BTN2 ◄──┐                 
                               │  Pull-up        [PLAY] Button
                               ├─ 10kΩ ── VCC   (Momentary Switch)
                               └─ Switch ── GND
                      
GPIO[1]      ◄──────  BTN1 ◄──┐                 
                               │  Pull-up        [FEED] Button
                               ├─ 10kΩ ── VCC   (Momentary Switch)
                               └─ Switch ── GND
                      
GPIO[0]      ◄──────  BTN0 ◄──┐                 
                               │  Pull-up        [SELECT] Button
                               ├─ 10kΩ ── VCC   (Momentary Switch)
                               └─ Switch ── GND

Pin Summary:
────────────
Inputs:   4  (Buttons)
Outputs: 10  (LCD SPI + Control, PWM, LED)
I/O:      2  (I2C SDA/SCL)
────────────
Total:   16  (22 GPIOs remaining for expansion)
```

---

## Data Flow Diagrams

### Main Game Loop Data Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                        Power On / Reset                          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Initialize     │
                    │  Peripherals    │
                    │                 │
                    │ • GPIO (buttons)│
                    │ • SPI (LCD)     │
                    │ • Timer (RTC)   │
                    │ • SRAM (frame)  │
                    │ • LFSR (seed)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Load Pet State  │
                    │  from EEPROM    │
                    │                 │
                    │ • Health        │
                    │ • Hunger        │
                    │ • Happiness     │
                    │ • Age           │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Initialize LCD  │
                    │                 │
                    │ • Send init cmds│
                    │ • Clear buffer  │
                    │ • Display splash│
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Start Timer    │
                    │                 │
                    │ • 15 min period │
                    │ • Enable IRQ    │
                    └────────┬────────┘
                             │
                             │
        ┌────────────────────▼────────────────────┐
        │                                         │
        │            Main Event Loop              │
        │                                         │
        │  ┌───────────────────────────────────┐ │
        │  │ 1. Check for button press (IRQ)   │ │
        │  │    ├─ BTN0: SELECT menu item      │ │
        │  │    ├─ BTN1: FEED pet             │ │
        │  │    ├─ BTN2: PLAY with pet        │ │
        │  │    └─ BTN3: Give MEDICINE        │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 2. Process action if button press  │ │
        │  │    ├─ Update pet state            │ │
        │  │    ├─ Play sound (PWM tone)       │ │
        │  │    └─ Trigger display update      │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 3. Check timer interrupt (15 min)  │ │
        │  │    ├─ Increase hunger              │ │
        │  │    ├─ Decrease happiness           │ │
        │  │    ├─ Update health                │ │
        │  │    ├─ Check age increment          │ │
        │  │    └─ Random event (LFSR)          │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 4. Update display                  │ │
        │  │    ├─ Clear frame buffer           │ │
        │  │    ├─ Draw pet sprite (SRAM)       │ │
        │  │    ├─ Draw status bars             │ │
        │  │    ├─ Draw age/time                │ │
        │  │    └─ Send to LCD via SPI          │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 5. Check attention needs           │ │
        │  │    ├─ If hungry: play alert        │ │
        │  │    ├─ If unhappy: play alert       │ │
        │  │    └─ If sick: play alert          │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 6. Auto-save check (5 min)         │ │
        │  │    └─ Save state to EEPROM         │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        │  ┌──────────────▼────────────────────┐ │
        │  │ 7. Enter sleep mode                │ │
        │  │    └─ WFI (Wait For Interrupt)     │ │
        │  └──────────────┬────────────────────┘ │
        │                 │                       │
        └─────────────────┼───────────────────────┘
                          │
                          └──► Loop back to 1
```

### Display Update Flow

```
Firmware                  SRAM (0x3003_0000)           SPI (CF_SPI)            LCD
────────                  ──────────────────           ─────────────           ───────

Step 1: Clear Frame Buffer
─────────────────────────
Write 0x00 ──────────► [0x000 - 0x1F7]
to all 504                (Frame Buffer)
bytes                     All pixels off
                          
Step 2: Draw Pet Sprite (16×16)
───────────────────────────────
Read sprite ◄────────── [0x200 - 0x21F]
data from                 (Sprite #0)
SRAM                      
                          
Calculate                 
position                  
(x, y)                    
                          
Blit sprite ─────────► [Frame Buffer]
to frame                  Update pixels
buffer                    at (x, y)
                          
Step 3: Draw Status Bars
────────────────────────
Calculate                 
bar lengths               
(health %, etc)           
                          
Draw bars ───────────► [Frame Buffer]
(filled                   Update bar
rectangles)               pixels
                          
Step 4: Draw Text (Age, etc)
────────────────────────────
Lookup font               
data (5×7)                
                          
Render text ──────────► [Frame Buffer]
characters                Update text
                          pixels
                          
Step 5: Transfer to LCD
───────────────────────
Assert LCD_DC ────────────────────────────────────────► D/C = 0
(Command mode)                                         (Command)
                          
Send init    ────────────────────────► SPI TX
command                                │
                                       └───────────────► Set addr
                          
Assert LCD_DC ────────────────────────────────────────► D/C = 1
(Data mode)                                            (Data)
                          
Read frame   ◄────────── [0x000 - 0x1F7]
buffer                    
(504 bytes)               
                          
For each byte:            
  Write to ──────────────────────────► SPI TX
  SPI                                  │
                                       └───────────────► Transfer
  Wait for                             │                 byte to
  SPI done ◄───────────────────────────┘                 LCD
                          
Repeat for                             
all 504 bytes             
                          
                                                        LCD displays
                                                        new image
```

### Button Press Handling Flow

```
Hardware (GPIO)          Firmware (ISR)               Firmware (Main)
───────────────          ──────────────               ───────────────

Button pressed           
(GPIO goes LOW)          
      │                  
      │ Interrupt         
      │ triggered         
      ├──────────────────► GPIO ISR Entry
      │                   │
      │                   │ 1. Read GPIO_DATA_IN
      │                   │    register (0x3000_0004)
      │                   │    
      │                   │ 2. Determine which button:
      │                   │    • Bit 0: SELECT
      │                   │    • Bit 1: FEED
      │                   │    • Bit 2: PLAY
      │                   │    • Bit 3: MEDICINE
      │                   │    
      │                   │ 3. Set global flag:
      │                   │    button_pressed = BTN_ID
      │                   │    
      │                   │ 4. Clear interrupt:
      │                   │    Write to GPIO_INT_STATUS
      │                   │    (0x3000_001C)
      │                   │    
      │                   │ 5. Debounce timer start
      │                   │    
      │                   └──► Return from ISR
      │                   
      │                                                 Main loop detects
      │                                                 button_pressed flag
      │                                                       │
      │                                                       │
      │                                                       ▼
      │                                                 Switch(button_pressed):
      │                                                       │
      │                                                       ├─ Case BTN_FEED:
      │                                                       │  • hunger -= 35
      │                                                       │  • Play tone (2kHz)
      │                                                       │  • Update display
      │                                                       │  
      │                                                       ├─ Case BTN_PLAY:
      │                                                       │  • happiness += 40
      │                                                       │  • Play tone (3kHz)
      │                                                       │  • Update display
      │                                                       │  
      │                                                       ├─ Case BTN_MEDICINE:
      │                                                       │  • is_sick = false
      │                                                       │  • health += 50
      │                                                       │  • Play tone (2.5kHz)
      │                                                       │  • Update display
      │                                                       │  
      │                                                       └─ Case BTN_SELECT:
      │                                                          • Menu navigation
      │                                                          • Play tone (1.5kHz)
      │                                                       
      │                                                 Clear button_pressed flag
```

---

## State Machine Diagrams

### Pet State Machine (Firmware)

```
                          ┌─────────────┐
                          │    INIT     │
                          │             │
                          │ Load state  │
                          │ from EEPROM │
                          └──────┬──────┘
                                 │
                                 │ State loaded
                                 │
                          ┌──────▼──────┐
             ┌────────────│   ALIVE     │◄────────────┐
             │            │             │             │
             │            │ Normal pet  │             │
             │            │ behavior    │             │
             │            └──────┬──────┘             │
             │                   │                    │
             │                   │ Timer interrupt    │
             │                   │ (15 min)           │
             │                   │                    │
             │            ┌──────▼──────┐             │
             │            │   UPDATE    │             │
             │            │   STATS     │             │
             │            │             │             │
             │            │ • hunger++  │             │
             │            │ • happiness-│             │
             │            │ • age check │             │
             │            └──────┬──────┘             │
             │                   │                    │
             │     ┌─────────────┼─────────────┐      │
             │     │             │             │      │
             │     │             │             │      │
        Hunger     │      Health check        │   Stats OK
        very high  │             │         Sick       │
        OR         │             │         check      │
        happiness  │             │             │      │
        very low   │             │             │      │
             │     │             │             │      │
             │     │      Health reaches 0    │      │
             │     │             │       Random       │
             │     │             │       event        │
             │     │             │      (LFSR)        │
             │     │             │             │      │
      ┌──────▼─────▼──┐   ┌──────▼──────┐   ┌─▼──────▼─────┐
      │   HUNGRY/      │   │    DEAD     │   │    SICK      │
      │   UNHAPPY      │   │             │   │              │
      │                │   │ Pet died    │   │ Reduced      │
      │ Needs care     │   │ Game over   │   │ happiness    │
      │ Alert beeping  │   │ Show sad    │   │ Needs meds   │
      └────────┬───────┘   │ animation   │   └──────┬───────┘
               │           └─────────────┘          │
               │                                    │
               │ User feeds/plays                   │ User gives
               │                                    │ medicine
               │                                    │
               └────────────────────────────────────┘
                          (Return to ALIVE)

User Actions:
─────────────
FEED:     hunger -= 35, clamped to [0, 100]
PLAY:     happiness += 40, clamped to [0, 100]
MEDICINE: is_sick = false, health += 50

State Update Rules (Every 15 min):
──────────────────────────────────
• hunger += 5 (range: 0-100)
• happiness -= 5 (range: 0-100)
• if (hunger > 80 OR happiness < 20): health -= 10
• if (health > 50 AND hunger < 50): health += 5 (slowly recover)
• if (is_sick): health -= 15
• if (health <= 0): transition to DEAD
• Random event (10% chance): 
    - Get sick (5%)
    - Sudden happiness boost (5%)
• Age increments every 24 hours (real-time)
```

### Save/Load State Machine

```
                    ┌──────────────┐
                    │   BOOT UP    │
                    └──────┬───────┘
                           │
                           │
                    ┌──────▼───────┐
                    │ Read EEPROM  │
                    │ Magic Byte   │
                    │ (0xAA)       │
                    └──────┬───────┘
                           │
                ┌──────────┴──────────┐
                │                     │
          Magic found           Magic not found
                │                     │
         ┌──────▼─────┐         ┌─────▼──────┐
         │ Read State │         │  Init New  │
         │ from EEPROM│         │   Pet      │
         │            │         │            │
         │ • Health   │         │ • Health=  │
         │ • Hunger   │         │   100      │
         │ • Happiness│         │ • Hunger=0 │
         │ • Age      │         │ • Happiness│
         │ • Timestamp│         │   =100     │
         └──────┬─────┘         │ • Age=0    │
                │               └─────┬──────┘
                │                     │
         ┌──────▼─────┐               │
         │ Verify CRC │               │
         └──────┬─────┘               │
                │                     │
     ┌──────────┴──────────┐          │
     │                     │          │
  CRC OK              CRC Failed      │
     │                     │          │
     │              ┌──────▼──────┐   │
     │              │ Try Backup  │   │
     │              │ State       │   │
     │              └──────┬──────┘   │
     │                     │          │
     │              ┌──────┴──────┐   │
     │              │             │   │
     │         Backup OK    Backup bad│
     │              │             │   │
     └──────────────┴─────────────┴───┘
                    │
                    │ State loaded
                    │
             ┌──────▼───────┐
             │  Game Loop   │
             │   Running    │
             └──────┬───────┘
                    │
                    │ Every 5 minutes
                    │ OR on sleep
                    │
             ┌──────▼───────┐
             │ Save State   │
             │              │
             │ 1. Prepare   │
             │    data      │
             │ 2. Calculate │
             │    CRC       │
             │ 3. Write to  │
             │    EEPROM    │
             │    (primary) │
             │ 4. Write to  │
             │    EEPROM    │
             │    (backup)  │
             └──────┬───────┘
                    │
                    └──► Continue game

EEPROM Layout:
──────────────
0x00: Magic (0xAA)
0x01: Version
0x02: Health
0x03: Hunger
0x04: Happiness
0x05-06: Age
0x07-0A: Timestamp
...
0x30-31: CRC16
0x32-51: Backup copy
```

---

## LFSR Random Number Generator

### LFSR Block Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   LFSR_RNG Module                           │
│                                                             │
│  Wishbone Interface:                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ wbs_adr_i[31:0] ──► Address Decoder                  │  │
│  │ wbs_dat_i[31:0] ──► Data Input (Seed)                │  │
│  │ wbs_dat_o[31:0] ◄── Data Output (Random Value)       │  │
│  │ wbs_we_i ───────────► Write Enable                    │  │
│  │ wbs_cyc_i, wbs_stb_i ──► Transaction Control         │  │
│  │ wbs_ack_o ◄──────────── Acknowledge                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                              │                              │
│                              │                              │
│  ┌───────────────────────────┴──────────────────────────┐  │
│  │              Register Bank                           │  │
│  │                                                      │  │
│  │  0x00: CTRL    [1:0] = {LOAD_SEED, ENABLE}         │  │
│  │  0x04: SEED    [15:0] = Seed value                  │  │
│  │  0x08: VALUE   [15:0] = Current random output       │  │
│  └───────────────────────────┬──────────────────────────┘  │
│                              │                              │
│                              │                              │
│  ┌───────────────────────────▼──────────────────────────┐  │
│  │              Control Logic                           │  │
│  │                                                      │  │
│  │  if (LOAD_SEED):                                    │  │
│  │     lfsr_reg <= SEED                                │  │
│  │  else if (ENABLE):                                  │  │
│  │     lfsr_reg <= {lfsr[14:0], feedback}              │  │
│  └───────────────────────────┬──────────────────────────┘  │
│                              │                              │
│                              │                              │
│  ┌───────────────────────────▼──────────────────────────┐  │
│  │        16-bit LFSR Shift Register                    │  │
│  │                                                      │  │
│  │   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬──┐ │  │
│  │   │15 │14 │13 │12 │11 │10 │9  │8  │7  │6  │5  │...│ │  │
│  │   └─┬─┴─┬─┴───┴─┬─┴─┬─┴───┴───┴───┴───┴───┴───┴──┘ │  │
│  │     │   │       │   │                               │  │
│  │     │   │       │   │   Tap positions:              │  │
│  │     │   │       │   │   • Bit 15 (x^16)             │  │
│  │     │   │       │   └─► • Bit 13 (x^14)             │  │
│  │     │   │       └─────► • Bit 12 (x^13)             │  │
│  │     │   └─────────────► • Bit 10 (x^11)             │  │
│  │     │                                                │  │
│  │     └──┐    ┌───┐    ┌───┐    ┌───┐                │  │
│  │        └───►│XOR├───►│XOR├───►│XOR├───► feedback    │  │
│  │             └───┘    └───┘    └───┘                 │  │
│  │               │        │        │                    │  │
│  │             bit15    bit13    bit12                  │  │
│  │                                bit10                 │  │
│  │                                                      │  │
│  │   feedback = lfsr[15] ^ lfsr[13] ^ lfsr[12] ^      │  │
│  │              lfsr[10]                               │  │
│  │                                                      │  │
│  │   Polynomial: x^16 + x^14 + x^13 + x^11 + 1        │  │
│  │   Period: 65535 (2^16 - 1) - Maximal Length        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Output: VALUE register = lfsr_reg[15:0]                   │
└─────────────────────────────────────────────────────────────┘

Usage Example (Firmware):
─────────────────────────
// Seed with current time or button press timing
LFSR_SEED = timer_value ^ button_timing;
LFSR_CTRL = 0x03; // Load seed + enable

// Generate random number 0-9
uint16_t random = LFSR_VALUE % 10;

// Check for random event (10% probability)
if ((LFSR_VALUE % 100) < 10) {
    trigger_random_event();
}
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Author**: NativeChips Agent
