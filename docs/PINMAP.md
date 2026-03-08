# Ratdeck — Hardware Pin Map

LilyGo T-Deck Plus (ESP32-S3) — all pin definitions in `src/config/BoardConfig.h`.

## Bus Overview

```
ESP32-S3
   ├── SPI2_HOST (shared bus) ──┬── ST7789V Display (CS=12, DC=11)
   │    SCK=40                   ├── SX1262 LoRa (CS=9)
   │    MISO=38                  └── SD Card (CS=39)
   │    MOSI=41
   │
   ├── I2C (SDA=18, SCL=8) ────┬── Keyboard ESP32-C3 (0x55)
   │                            └── GT911 Touch (0x5D)
   │
   ├── GPIO ── Trackball (UP=3, DOWN=2, LEFT=1, RIGHT=15, CLICK=0)
   │
   ├── UART ── UBlox MIA-M10Q GPS (TX=43, RX=44)
   │
   ├── I2S ── ES7210 Audio Codec
   │
   └── USB-C ── USB-Serial/JTAG
```

## Power Control

| Signal | GPIO | Notes |
|--------|------|-------|
| BOARD_POWER_PIN | 10 | **Must be set HIGH at boot** to enable all peripherals |

## SX1262 LoRa Radio

Shares SPI2_HOST bus with display and SD card:

| Signal | GPIO | Notes |
|--------|------|-------|
| CS | 9 | Chip select (active low) |
| IRQ | 45 | DIO1 interrupt |
| RST | 17 | Reset (active low) |
| BUSY | 13 | Poll before SPI transactions |
| RXEN | -1 | Not connected (DIO2 used as RF switch) |
| TXEN | -1 | Not connected |

**Radio configuration:**
- TCXO voltage: 1.8V (`MODE_TCXO_1_8V_6X` = 0x02)
- DIO2 as RF switch: enabled
- SPI clock: 8 MHz

## Display (ST7789V)

| Signal | GPIO | Notes |
|--------|------|-------|
| CS | 12 | Chip select |
| DC | 11 | Data/command |
| BL | 42 | Backlight PWM |

320x240 pixels, landscape (rotation=1), SPI clock up to 15 MHz (30 MHz overclockable).

## Shared SPI Bus

| Signal | GPIO |
|--------|------|
| SCK | 40 |
| MISO | 38 |
| MOSI | 41 |

All three SPI devices (display, radio, SD) share SPI2_HOST. Single-threaded access from `loop()`, no mutex needed.

## Keyboard (ESP32-C3)

| Signal | GPIO | Notes |
|--------|------|-------|
| I2C addr | 0x55 | Fixed address |
| INT | 46 | Interrupt pin |
| SDA | 18 | Shared I2C bus |
| SCL | 8 | Shared I2C bus |

## Trackball

| Signal | GPIO | Notes |
|--------|------|-------|
| UP | 3 | ISR-based, GPIO interrupt |
| DOWN | 2 | ISR-based, GPIO interrupt |
| LEFT | 1 | ISR-based, GPIO interrupt |
| RIGHT | 15 | ISR-based, GPIO interrupt |
| CLICK | 0 | Shared with BOOT button |

Click uses deferred release with 80ms debounce. Long press: 1200ms hold threshold.

## Touchscreen (GT911)

| Signal | GPIO | Notes |
|--------|------|-------|
| INT | 16 | Interrupt |
| I2C addr | 0x5D | Depends on INT state at boot |
| SDA | 18 | Shared I2C bus |
| SCL | 8 | Shared I2C bus |

Currently disabled — coordinates uncalibrated.

## SD Card

| Signal | GPIO | Notes |
|--------|------|-------|
| CS | 39 | Shared SPI bus |

FAT32, shares SPI2_HOST with display and radio.

## GPS (UBlox MIA-M10Q)

| Signal | GPIO | Notes |
|--------|------|-------|
| TX | 43 | ESP TX → GPS RX |
| RX | 44 | GPS TX → ESP RX |

UART at 115200 baud. Pins defined, not yet implemented.

## Battery

| Signal | GPIO | Notes |
|--------|------|-------|
| ADC | 4 | Battery voltage measurement |

## Audio (ES7210 I2S)

| Signal | GPIO |
|--------|------|
| WS (LRCK) | 5 |
| DOUT | 6 |
| BCK | 7 |
| DIN | 14 |
| SCK | 47 |
| MCLK | 48 |

## Hardware Constants

| Constant | Value | Notes |
|----------|-------|-------|
| `MAX_PACKET_SIZE` | 255 | SX1262 maximum single packet |
| `SPI_FREQUENCY` | 8 MHz | SPI clock for SX1262 |
| `TFT_WIDTH` | 320 | Display pixels (landscape) |
| `TFT_HEIGHT` | 240 | Display pixels (landscape) |
| `TFT_SPI_FREQ` | 15 MHz | Display SPI clock |
