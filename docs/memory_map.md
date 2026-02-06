# Tamagotchi Memory Map

## Overview

This document defines the complete memory map for the Tamagotchi system on Caravel SoC, including Wishbone peripheral addresses, register definitions, and memory allocation.

---

## Wishbone Address Space

### Peripheral Base Addresses

| Base Address  | End Address   | Size   | Peripheral              | IP Source           |
|---------------|---------------|--------|-------------------------|---------------------|
| `0x3000_0000` | `0x3000_003F` | 64 B   | GPIO Controller         | EF_GPIO8            |
| `0x3001_0000` | `0x3001_00FF` | 256 B  | SPI Master              | CF_SPI              |
| `0x3002_0000` | `0x3002_007F` | 128 B  | Timer/PWM               | CF_TMR32            |
| `0x3003_0000` | `0x3003_0FFF` | 4 KB   | SRAM                    | CF_SRAM_1024x32     |
| `0x3004_0000` | `0x3004_003F` | 64 B   | LFSR RNG                | Custom              |
| `0x3005_0000` | `0x3005_00FF` | 256 B  | UART (Debug)            | CF_UART             |
| `0x3006_0000` | `0x3006_00FF` | 256 B  | Pet State Controller    | Custom (Phase 2)    |

---

## GPIO Controller (EF_GPIO8)

**Base Address**: `0x3000_0000`

### Register Map

| Offset | Name        | Access | Description                          |
|--------|-------------|--------|--------------------------------------|
| 0x00   | `DIR`       | R/W    | Direction control (0=in, 1=out)      |
| 0x04   | `DATA_IN`   | R      | Input data register                  |
| 0x08   | `DATA_OUT`  | R/W    | Output data register                 |
| 0x0C   | `PULLUP`    | R/W    | Pull-up enable (1=enable)            |
| 0x10   | `PULLDOWN`  | R/W    | Pull-down enable (1=enable)          |
| 0x14   | `INT_EN`    | R/W    | Interrupt enable (1=enable)          |
| 0x18   | `INT_EDGE`  | R/W    | Edge select (0=falling, 1=rising)    |
| 0x1C   | `INT_STATUS`| R/W1C  | Interrupt status (write 1 to clear)  |

### GPIO Pin Assignments

| Bit | Pin Name       | Direction | Function            | Notes                    |
|-----|----------------|-----------|---------------------|--------------------------|
| 0   | `BTN_SELECT`   | Input     | Button 0 (SELECT)   | Pull-up, interrupt       |
| 1   | `BTN_FEED`     | Input     | Button 1 (FEED)     | Pull-up, interrupt       |
| 2   | `BTN_PLAY`     | Input     | Button 2 (PLAY)     | Pull-up, interrupt       |
| 3   | `BTN_MEDICINE` | Input     | Button 3 (MEDICINE) | Pull-up, interrupt       |
| 4   | `LCD_RST`      | Output    | LCD Reset           | Active low               |
| 5   | `LCD_DC`       | Output    | LCD D/C             | Data/Command select      |
| 6   | `LCD_BL`       | Output    | LCD Backlight       | On/off control           |
| 7   | `STATUS_LED`   | Output    | Status LED          | Debug indicator          |

### Example Usage (C)

```c
// Configure GPIOs
#define GPIO_BASE 0x30000000
#define GPIO_DIR       (*(volatile uint32_t*)(GPIO_BASE + 0x00))
#define GPIO_DATA_IN   (*(volatile uint32_t*)(GPIO_BASE + 0x04))
#define GPIO_DATA_OUT  (*(volatile uint32_t*)(GPIO_BASE + 0x08))
#define GPIO_PULLUP    (*(volatile uint32_t*)(GPIO_BASE + 0x0C))
#define GPIO_INT_EN    (*(volatile uint32_t*)(GPIO_BASE + 0x14))
#define GPIO_INT_EDGE  (*(volatile uint32_t*)(GPIO_BASE + 0x18))
#define GPIO_INT_STATUS (*(volatile uint32_t*)(GPIO_BASE + 0x1C))

// Setup buttons (inputs with pull-ups and interrupts)
GPIO_DIR = 0xF0;           // Bits 0-3 input, 4-7 output
GPIO_PULLUP = 0x0F;        // Pull-up on buttons
GPIO_INT_EN = 0x0F;        // Enable interrupts on buttons
GPIO_INT_EDGE = 0x00;      // Falling edge (button press)

// Read button state
uint8_t buttons = GPIO_DATA_IN & 0x0F;

// Control LCD signals
GPIO_DATA_OUT |= (1 << 4);  // Set LCD_RST high
GPIO_DATA_OUT &= ~(1 << 5); // Set LCD_DC low (command mode)
```

---

## SPI Master (CF_SPI)

**Base Address**: `0x3001_0000`

### Register Map

| Offset | Name          | Access | Description                        |
|--------|---------------|--------|------------------------------------|
| 0x00   | `CTRL`        | R/W    | Control register                   |
| 0x04   | `STATUS`      | R      | Status register                    |
| 0x08   | `TX_DATA`     | W      | Transmit data (FIFO)               |
| 0x0C   | `RX_DATA`     | R      | Receive data (FIFO)                |
| 0x10   | `CLK_DIV`     | R/W    | Clock divider                      |
| 0x14   | `CS_CTRL`     | R/W    | Chip select control                |

### CTRL Register (0x00)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `ENABLE`      | SPI enable (1=enable)                |
| 1      | `CPOL`        | Clock polarity (0=idle low)          |
| 2      | `CPHA`        | Clock phase (0=sample on first edge) |
| 3      | `LSB_FIRST`   | Bit order (0=MSB first)              |
| 7:4    | Reserved      | -                                    |

### STATUS Register (0x04)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `BUSY`        | Transfer in progress (1=busy)        |
| 1      | `TX_EMPTY`    | TX FIFO empty                        |
| 2      | `TX_FULL`     | TX FIFO full                         |
| 3      | `RX_EMPTY`    | RX FIFO empty                        |
| 4      | `RX_FULL`     | RX FIFO full                         |
| 7:5    | Reserved      | -                                    |

### Example Usage (C)

```c
#define SPI_BASE 0x30010000
#define SPI_CTRL      (*(volatile uint32_t*)(SPI_BASE + 0x00))
#define SPI_STATUS    (*(volatile uint32_t*)(SPI_BASE + 0x04))
#define SPI_TX_DATA   (*(volatile uint32_t*)(SPI_BASE + 0x08))
#define SPI_RX_DATA   (*(volatile uint32_t*)(SPI_BASE + 0x0C))
#define SPI_CLK_DIV   (*(volatile uint32_t*)(SPI_BASE + 0x10))

// Configure SPI (Mode 0, 4 MHz clock)
SPI_CTRL = 0x01;           // Enable, CPOL=0, CPHA=0, MSB first
SPI_CLK_DIV = 6;           // 25 MHz / (2 * 6) = ~4 MHz

// Send byte to LCD
void spi_write(uint8_t data) {
    while (SPI_STATUS & 0x04);  // Wait if TX FIFO full
    SPI_TX_DATA = data;
    while (SPI_STATUS & 0x01);  // Wait until not busy
}
```

---

## Timer/PWM (CF_TMR32)

**Base Address**: `0x3002_0000`

### Register Map

| Offset | Name          | Access | Description                        |
|--------|---------------|--------|------------------------------------|
| 0x00   | `CTRL`        | R/W    | Control register                   |
| 0x04   | `PRESCALER`   | R/W    | Prescaler value                    |
| 0x08   | `PERIOD`      | R/W    | Timer period                       |
| 0x0C   | `COUNTER`     | R      | Current counter value              |
| 0x10   | `COMPARE`     | R/W    | Compare value (for PWM)            |
| 0x14   | `PWM_DUTY`    | R/W    | PWM duty cycle                     |
| 0x18   | `INT_STATUS`  | R/W1C  | Interrupt status                   |

### CTRL Register (0x00)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `ENABLE`      | Timer enable (1=enable)              |
| 1      | `INT_EN`      | Interrupt enable                     |
| 2      | `PWM_EN`      | PWM output enable                    |
| 3      | `ONE_SHOT`    | One-shot mode (0=continuous)         |
| 7:4    | Reserved      | -                                    |

### Example Usage (C)

```c
#define TMR_BASE 0x30020000
#define TMR_CTRL       (*(volatile uint32_t*)(TMR_BASE + 0x00))
#define TMR_PRESCALER  (*(volatile uint32_t*)(TMR_BASE + 0x04))
#define TMR_PERIOD     (*(volatile uint32_t*)(TMR_BASE + 0x08))
#define TMR_COUNTER    (*(volatile uint32_t*)(TMR_BASE + 0x0C))
#define TMR_PWM_DUTY   (*(volatile uint32_t*)(TMR_BASE + 0x14))
#define TMR_INT_STATUS (*(volatile uint32_t*)(TMR_BASE + 0x18))

// Configure timer for 15-minute intervals (assuming 25 MHz clock)
TMR_PRESCALER = 25000;     // Divide by 25000 → 1 kHz
TMR_PERIOD = 900000;       // 900,000 ms = 15 minutes
TMR_CTRL = 0x03;           // Enable timer + interrupt

// Configure PWM for 2 kHz buzzer tone (50% duty cycle)
TMR_PRESCALER = 25;        // Divide by 25 → 1 MHz
TMR_PERIOD = 500;          // 1 MHz / 500 = 2 kHz
TMR_PWM_DUTY = 250;        // 50% duty cycle
TMR_CTRL = 0x05;           // Enable timer + PWM
```

---

## SRAM (CF_SRAM_1024x32)

**Base Address**: `0x3003_0000`  
**Size**: 4 KB (1024 words × 32 bits)  
**Access**: 32-bit word-aligned

### Memory Layout

| Address Range        | Size    | Purpose                          |
|----------------------|---------|----------------------------------|
| `0x3003_0000 - 0x3003_01F7` | 504 B   | LCD Frame Buffer (84×48/8)       |
| `0x3003_0200 - 0x3003_02FF` | 256 B   | Sprite Storage (16×16 × 4 frames)|
| `0x3003_0300 - 0x3003_0FFF` | 3328 B  | Working Memory / Stack           |

### Frame Buffer Layout

**Total Size**: 504 bytes (84 columns × 48 rows / 8 bits)

```
Memory Layout (Column-major, 6 rows of 84 bytes each):
Row 0: [0x00-0x53] - Top 8 pixels (bits 0-7) of all 84 columns
Row 1: [0x54-0xA7] - Next 8 pixels (bits 8-15)
Row 2: [0xA8-0xFB] - Next 8 pixels (bits 16-23)
Row 3: [0xFC-0x14F] - Next 8 pixels (bits 24-31)
Row 4: [0x150-0x1A3] - Next 8 pixels (bits 32-39)
Row 5: [0x1A4-0x1F7] - Bottom 8 pixels (bits 40-47)
```

### Example Usage (C)

```c
#define SRAM_BASE 0x30030000
#define FRAME_BUFFER ((volatile uint8_t*)SRAM_BASE)
#define SPRITE_DATA  ((volatile uint8_t*)(SRAM_BASE + 0x200))
#define WORK_MEM     ((volatile uint32_t*)(SRAM_BASE + 0x300))

// Clear frame buffer
void clear_framebuffer() {
    for (int i = 0; i < 504; i++) {
        FRAME_BUFFER[i] = 0x00;
    }
}

// Set pixel at (x, y)
void set_pixel(uint8_t x, uint8_t y) {
    if (x < 84 && y < 48) {
        uint16_t offset = (y / 8) * 84 + x;
        uint8_t bit = y % 8;
        FRAME_BUFFER[offset] |= (1 << bit);
    }
}

// Load sprite (16×16 pixels, 32 bytes)
void load_sprite(uint8_t sprite_id, const uint8_t* data) {
    for (int i = 0; i < 32; i++) {
        SPRITE_DATA[sprite_id * 32 + i] = data[i];
    }
}
```

---

## LFSR Random Number Generator

**Base Address**: `0x3004_0000`

### Register Map

| Offset | Name          | Access | Description                        |
|--------|---------------|--------|------------------------------------|
| 0x00   | `CTRL`        | R/W    | Control register                   |
| 0x04   | `SEED`        | W      | Seed value (16-bit)                |
| 0x08   | `VALUE`       | R      | Current random value (16-bit)      |

### CTRL Register (0x00)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `ENABLE`      | LFSR enable (1=running)              |
| 1      | `LOAD_SEED`   | Load seed (write 1, auto-clears)     |
| 7:2    | Reserved      | -                                    |

### Example Usage (C)

```c
#define LFSR_BASE 0x30040000
#define LFSR_CTRL  (*(volatile uint32_t*)(LFSR_BASE + 0x00))
#define LFSR_SEED  (*(volatile uint32_t*)(LFSR_BASE + 0x04))
#define LFSR_VALUE (*(volatile uint32_t*)(LFSR_BASE + 0x08))

// Initialize RNG with seed
void rng_init(uint16_t seed) {
    LFSR_SEED = seed;
    LFSR_CTRL = 0x03;  // Load seed + enable
}

// Get random number (0-65535)
uint16_t get_random() {
    return LFSR_VALUE & 0xFFFF;
}

// Get random number in range [min, max]
uint16_t get_random_range(uint16_t min, uint16_t max) {
    uint16_t range = max - min + 1;
    return min + (get_random() % range);
}
```

---

## UART Debug Port (CF_UART)

**Base Address**: `0x3005_0000`

### Register Map

| Offset | Name          | Access | Description                        |
|--------|---------------|--------|------------------------------------|
| 0x00   | `CTRL`        | R/W    | Control register                   |
| 0x04   | `STATUS`      | R      | Status register                    |
| 0x08   | `TX_DATA`     | W      | Transmit data                      |
| 0x0C   | `RX_DATA`     | R      | Receive data                       |
| 0x10   | `BAUD_DIV`    | R/W    | Baud rate divider                  |

### CTRL Register (0x00)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `TX_EN`       | Transmitter enable                   |
| 1      | `RX_EN`       | Receiver enable                      |
| 2      | `TX_INT_EN`   | TX interrupt enable                  |
| 3      | `RX_INT_EN`   | RX interrupt enable                  |
| 7:4    | Reserved      | -                                    |

### STATUS Register (0x04)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `TX_BUSY`     | Transmitter busy                     |
| 1      | `TX_EMPTY`    | TX buffer empty                      |
| 2      | `RX_READY`    | RX data available                    |
| 3      | `RX_OVERRUN`  | RX overrun error                     |
| 7:4    | Reserved      | -                                    |

### Example Usage (C)

```c
#define UART_BASE 0x30050000
#define UART_CTRL     (*(volatile uint32_t*)(UART_BASE + 0x00))
#define UART_STATUS   (*(volatile uint32_t*)(UART_BASE + 0x04))
#define UART_TX_DATA  (*(volatile uint32_t*)(UART_BASE + 0x08))
#define UART_RX_DATA  (*(volatile uint32_t*)(UART_BASE + 0x0C))
#define UART_BAUD_DIV (*(volatile uint32_t*)(UART_BASE + 0x10))

// Configure UART (115200 baud, 25 MHz clock)
void uart_init() {
    UART_BAUD_DIV = 217;  // 25MHz / (16 * 115200) ≈ 217
    UART_CTRL = 0x03;     // Enable TX + RX
}

// Print character
void uart_putc(char c) {
    while (UART_STATUS & 0x01);  // Wait if busy
    UART_TX_DATA = c;
}

// Print string
void uart_puts(const char* str) {
    while (*str) {
        uart_putc(*str++);
    }
}
```

---

## Pet State Controller (Phase 2 - Optional)

**Base Address**: `0x3006_0000`

### Register Map

| Offset | Name          | Access | Description                        |
|--------|---------------|--------|------------------------------------|
| 0x00   | `CTRL`        | R/W    | Control register                   |
| 0x04   | `HEALTH`      | R/W    | Health value (0-100)               |
| 0x08   | `HUNGER`      | R/W    | Hunger value (0-100)               |
| 0x0C   | `HAPPINESS`   | R/W    | Happiness value (0-100)            |
| 0x10   | `AGE`         | R/W    | Age in days                        |
| 0x14   | `TIME_DELTA`  | W      | Time since last update (seconds)   |
| 0x18   | `ACTION`      | W      | User action to apply               |
| 0x1C   | `STATUS`      | R      | Status flags                       |
| 0x20   | `UPDATE`      | W      | Trigger state update (write 1)     |

### ACTION Register Values (0x18)

| Value | Action        | Effect                               |
|-------|---------------|--------------------------------------|
| 0x00  | NONE          | No action                            |
| 0x01  | FEED          | Decrease hunger by 30-40             |
| 0x02  | PLAY          | Increase happiness by 30-40          |
| 0x03  | MEDICINE      | Cure sickness, increase health       |
| 0x04  | CLEAN         | Increase happiness slightly          |

### STATUS Register (0x1C)

| Bit    | Name          | Description                          |
|--------|---------------|--------------------------------------|
| 0      | `IS_SICK`     | Pet is sick (1=sick)                 |
| 1      | `IS_SLEEPING` | Pet is sleeping (1=sleeping)         |
| 2      | `IS_DEAD`     | Pet is dead (1=dead)                 |
| 3      | `NEEDS_ATTENTION` | Needs care (hunger/happiness low) |
| 7:4    | Reserved      | -                                    |

---

## Memory Map Summary Diagram

```
┌────────────────────────────────────────────────────────┐
│  Caravel SoC Address Space                             │
├────────────────────────────────────────────────────────┤
│  0x0000_0000 - 0x0FFF_FFFF  │ Flash XIP (Firmware)     │
│  0x1000_0000 - 0x1FFF_FFFF  │ Reserved                 │
│  0x2000_0000 - 0x2FFF_FFFF  │ Caravel System Regs      │
│  0x3000_0000 - 0x3FFF_FFFF  │ User Project Peripherals │
│     ├─ 0x3000_0000          │   GPIO (64 B)            │
│     ├─ 0x3001_0000          │   SPI (256 B)            │
│     ├─ 0x3002_0000          │   Timer/PWM (128 B)      │
│     ├─ 0x3003_0000          │   SRAM (4 KB)            │
│     ├─ 0x3004_0000          │   LFSR RNG (64 B)        │
│     ├─ 0x3005_0000          │   UART (256 B)           │
│     └─ 0x3006_0000          │   Pet State (256 B)      │
└────────────────────────────────────────────────────────┘
```

---

## Interrupt Vector Table

| IRQ # | Source         | Priority | Handler                      |
|-------|----------------|----------|------------------------------|
| 0     | Timer Overflow | High     | `timer_isr()` - State update |
| 1     | GPIO (Buttons) | High     | `button_isr()` - User input  |
| 2     | UART RX        | Low      | `uart_isr()` - Debug input   |
| 3     | SPI Done       | Medium   | `spi_isr()` - Transfer done  |

---

## EEPROM Memory Layout (Off-Chip)

**Device**: AT24C32 (4 KB I2C EEPROM)  
**I2C Address**: 0x50

### Storage Layout

| Address | Size   | Data                          | Notes                    |
|---------|--------|-------------------------------|--------------------------|
| 0x000   | 1 B    | Magic byte (0xAA)             | Validity check           |
| 0x001   | 1 B    | Version                       | Data format version      |
| 0x002   | 1 B    | Health                        | 0-100                    |
| 0x003   | 1 B    | Hunger                        | 0-100                    |
| 0x004   | 1 B    | Happiness                     | 0-100                    |
| 0x005   | 2 B    | Age (days)                    | 16-bit                   |
| 0x007   | 4 B    | Birth timestamp               | Unix time                |
| 0x00B   | 4 B    | Last save timestamp           | Unix time                |
| 0x00F   | 1 B    | Status flags                  | Sick, sleeping, etc.     |
| 0x010   | 32 B   | Statistics (lifetime)         | Total feeds, plays, etc. |
| 0x030   | 2 B    | CRC16                         | Data integrity           |
| 0x032   | 32 B   | Backup copy of core state     | Redundancy               |
| 0x052   | -      | Reserved for future expansion | -                        |

---

## Constants and Configuration

### System Configuration

```c
// Clock frequencies
#define CLK_FREQ_HZ         25000000    // 25 MHz system clock
#define RTC_FREQ_HZ         32768       // 32.768 kHz RTC

// Timing constants
#define STATE_UPDATE_MS     900000      // 15 minutes
#define AUTOSAVE_MS         300000      // 5 minutes
#define BUTTON_DEBOUNCE_MS  50          // 50 ms debounce

// Game constants
#define HEALTH_MAX          100
#define HUNGER_MAX          100
#define HAPPINESS_MAX       100
#define HUNGER_RATE         5           // Increase per update
#define HAPPINESS_RATE      5           // Decrease per update
#define FEED_AMOUNT         35
#define PLAY_AMOUNT         40
#define MEDICINE_HEAL       50

// Display constants
#define LCD_WIDTH           84
#define LCD_HEIGHT          48
#define LCD_BUFFER_SIZE     504
#define SPRITE_SIZE         32          // 16×16 pixels = 32 bytes

// Audio frequencies (Hz)
#define TONE_ATTENTION      2000
#define TONE_BUTTON         1500
#define TONE_HAPPY          3000
#define TONE_SAD            500
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-06  
**Author**: NativeChips Agent
