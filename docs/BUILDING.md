# Ratdeck — Build & Flash Reference

## Prerequisites

- **Python 3.x** with PlatformIO: `pip install platformio`
- **Git**
- **USB-C cable** (data-capable)

No USB drivers needed on macOS or Linux — the ESP32-S3's USB-Serial/JTAG interface is natively supported.

> If `pio` is not on your PATH after install, use `python3 -m platformio` for all commands in this document.

## Build

```bash
git clone https://github.com/defidude/Ratdeck.git
cd Ratdeck

python3 -m platformio run
```

Output: `.pio/build/ratdeck_915/firmware.bin`

First build downloads all dependencies automatically (microReticulum, Crypto, ArduinoJson, LovyanGFX, NimBLE-Arduino, LVGL).

## Flash

### Via PlatformIO

```bash
python3 -m platformio run --target upload
```

### Via esptool (more reliable)

PlatformIO defaults to the upload speed in `platformio.ini` (460800 baud). If flashing fails, use esptool directly:

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* --baud 460800 \
    --before default-reset --after hard-reset \
    write-flash -z 0x10000 .pio/build/ratdeck_915/firmware.bin
```

### Download Mode

If the device doesn't respond to flash commands, enter download mode:
1. Hold the trackball (BOOT/GPIO 0) button
2. While holding, press the reset button
3. Release both — device enters bootloader mode
4. Flash as normal

### Web Flash

Visit [ratspeak.org/flash](https://ratspeak.org/flash) to flash directly from your browser using WebSerial — no build tools required.

### Creating a Merged Binary

A merged binary includes bootloader + partition table + app in one file:

```bash
python3 -m esptool --chip esp32s3 merge-bin \
    --output ratdeck_merged.bin \
    --flash-mode dio --flash-size 16MB \
    0x0    ~/.platformio/packages/framework-arduinoespressif32/tools/sdk/esp32s3/bin/bootloader_dio_80m.bin \
    0x8000 .pio/build/ratdeck_915/partitions.bin \
    0xe000 ~/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin \
    0x10000 .pio/build/ratdeck_915/firmware.bin

python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* --baud 460800 \
    --before default-reset --after hard-reset \
    write-flash 0x0 ratdeck_merged.bin
```

## Serial Monitor

```bash
python3 -m platformio device monitor -b 115200
```

Or any serial terminal at 115200 baud.

## USB Port Identification

The T-Deck Plus ESP32-S3 uses USB-Serial/JTAG (not a separate UART chip):

| OS | Port | Notes |
|----|------|-------|
| macOS | `/dev/cu.usbmodem*` | Use glob to match |
| Linux | `/dev/ttyACM0` | May need `dialout` group |

On Linux: `sudo usermod -aG dialout $USER` (log out and back in).

## Build Flags

From `platformio.ini`:

| Flag | Purpose |
|------|---------|
| `-std=gnu++17` | C++17 standard |
| `-fexceptions` | C++ exceptions (required by microReticulum) |
| `-DRATDECK=1` | Main feature flag |
| `-DARDUINO_USB_CDC_ON_BOOT=1` | USB CDC serial on boot |
| `-DARDUINO_USB_MODE=1` | USB-Serial/JTAG mode (not native CDC) |
| `-DRNS_USE_FS` | microReticulum: use filesystem for persistence |
| `-DRNS_PERSIST_PATHS` | microReticulum: persist transport paths |
| `-DMSGPACK_USE_BOOST=OFF` | Disable Boost dependency in MsgPack |
| `-DBOARD_HAS_PSRAM` | Enable 8MB PSRAM |
| `-DDISPLAY_WIDTH=320` | Display width |
| `-DDISPLAY_HEIGHT=240` | Display height |
| `-DLV_CONF_INCLUDE_SIMPLE` | LVGL config via root `lv_conf.h` |
| `"-I${PROJECT_DIR}"` | Required for LVGL to find root `lv_conf.h` |

`build_unflags = -fno-exceptions -std=gnu++11` removes Arduino defaults.

## Partition Table

`partitions_16MB.csv` — 16MB flash layout with OTA support:

| Name | Type | Offset | Size | Purpose |
|------|------|--------|------|---------|
| nvs | data/nvs | 0x9000 | 20 KB | Non-volatile storage (boot counter, WiFi creds) |
| otadata | data/ota | 0xE000 | 8 KB | OTA boot selection |
| app0 | app/ota_0 | 0x10000 | 4 MB | Primary firmware |
| app1 | app/ota_1 | 0x410000 | 4 MB | OTA update slot |
| littlefs | data/spiffs | 0x810000 | 7.875 MB | LittleFS — identity, config, messages, paths |
| coredump | data/coredump | 0xFF0000 | 64 KB | ESP-IDF core dump on crash |

## CI/CD

GitHub Actions workflow (`.github/workflows/build.yml`):

- **Build**: Triggers on push to `main` and PRs. Runs `pio run`, uploads `firmware.bin` as artifact.
- **Release**: Triggers on `v*` tags. Builds firmware, creates a GitHub Release with the binary ZIP attached. Powers the web flasher at [ratspeak.org/flash](https://ratspeak.org/flash).

### Release Process

```bash
# 1. Bump version in src/config/Config.h (all 4 defines)
# 2. Commit and push
git add -A && git commit -m "v1.4.2: description"
git push origin main

# 3. Tag and push tag
git tag v1.4.2
git push origin v1.4.2
# CI creates GitHub release automatically
```

## Dependencies

All managed by PlatformIO's `lib_deps`:

| Library | Source | Purpose |
|---------|--------|---------|
| microReticulum | `attermann/microReticulum#392363c` | Reticulum protocol stack (C++) |
| Crypto | `attermann/Crypto` | Ed25519, X25519, AES, SHA-256 |
| ArduinoJson | `bblanchon/ArduinoJson ^7.4.2` | Config and message serialization |
| LovyanGFX | `lovyan03/LovyanGFX ^1.1.16` | SPI display driver |
| NimBLE-Arduino | `h2zero/NimBLE-Arduino ^2.1` | BLE Sideband transport |
| LVGL | `lvgl/lvgl ^8.3.4` | UI framework (v8.4) |

## Erasing Flash

To completely erase the ESP32-S3 flash (useful if LittleFS is corrupted):

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* erase-flash
```

After erasing, reflash the firmware. LittleFS will auto-format on first boot and a new identity will be generated.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Could not open port` | Device not connected or wrong port | Check USB cable, try `/dev/cu.usbmodem*` glob |
| `Timed out waiting for packet header` | Baud rate too high | Use `--baud 460800` with esptool |
| Serial disconnects mid-flash | Unstable USB connection | Enter download mode (hold trackball + reset) |
| `No module named platformio` | PlatformIO not installed | `pip install platformio` |
| `pio: command not found` | Not on PATH | Use `python3 -m platformio` instead |
