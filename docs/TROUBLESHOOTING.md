# Ratdeck — Troubleshooting

Collected hardware and software gotchas for the LilyGo T-Deck Plus with SX1262 LoRa.

---

## Radio Issues

### TCXO voltage must be 1.8V

The T-Deck Plus SX1262 uses a TCXO that requires 1.8V. This is configured as `MODE_TCXO_1_8V_6X` (0x02) in `BoardConfig.h`.

**Symptom**: Radio reports online but can't synthesize frequencies. PLL lock fails, no TX or RX.

**Diagnosis**: `Ctrl+D` diagnostics — if `DevErrors` shows `0x0040`, that's PLL lock failure.

**Fix**: Verify `LORA_TCXO_VOLTAGE` is `0x02` in `BoardConfig.h`.

### SetModulationParams must be called from STDBY mode

The SX1262 silently rejects `SetModulationParams` (0x8B) when issued from RX or TX mode. The command appears to succeed but the hardware ignores the new SF/BW/CR values.

**Symptom**: Software logs show correct SF/BW/CR but actual TX airtime is wrong. Two devices see each other's RF (RSSI visible) but every packet fails CRC.

**Fix**: `setModulationParams()` now calls `standby()` internally before issuing the SPI command.

### Calibration must run after TCXO is enabled

Per SX1262 datasheet Section 13.1.12, if a TCXO is used, it must be enabled before calling `calibrate()`. Calibration locks to whichever oscillator is active.

**Symptom**: TX completes successfully on both devices, RSSI shows real signals, but neither device ever decodes the other's packets.

**Fix**: Init order must be: `enableTCXO()` -> `delay(5ms)` -> `setRegulatorMode(DC-DC)` -> `loraMode()` -> `standby(XOSC)` -> `calibrate()` -> `calibrate_image()`

### IRQ stale latch

The SX1262's IRQ flags can become latched from previous operations. If stale flags persist, DCD gets stuck in "channel busy" and TX never fires.

**Symptom**: First packet sends fine, then all subsequent TX attempts hang.

**Fix**: Clear all IRQ flags at the top of `receive()` before entering RX mode. In `dcd()`, clear stale header error flags when preamble is not detected.

---

## Build Issues

### PlatformIO not on PATH

**Fix**: Use `python3 -m platformio` instead of `pio`:

```bash
python3 -m platformio run
python3 -m platformio run --target upload
python3 -m platformio device monitor -b 115200
```

### Flash fails or disconnects mid-upload

The T-Deck Plus USB-Serial/JTAG can be sensitive to baud rates.

**Fix**: Enter download mode (hold trackball/BOOT button while pressing reset), then flash:

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* --baud 460800 \
    write-flash -z 0x10000 .pio/build/ratdeck_915/firmware.bin
```

### USBMode must be `default`

The build flag `ARDUINO_USB_MODE=1` selects USB-Serial/JTAG mode (not native CDC).

**Symptom**: With `hwcdc` (USB_MODE=0), the port never appears on macOS.

**Fix**: Keep `ARDUINO_USB_MODE=1` in `platformio.ini`.

---

## Boot Issues

### Boot loop detection and recovery

Ratdeck tracks consecutive boot failures in NVS. If 3 consecutive boots fail to reach the end of `setup()`, WiFi is forced OFF on the next boot.

**How it works**:
1. On each boot, NVS counter `bootc` increments
2. If `bootc >= 3`, WiFi forced to OFF
3. At end of successful `setup()`, counter resets to 0

**Manual recovery**: Connect serial at 115200 baud and watch for `[BOOT] Boot loop detected`. The device should stabilize with WiFi off, then change WiFi settings in Setup.

### Transport reference crash

`RNS::Transport::_interfaces` stores `Interface&` (references, not copies). If a `RNS::Interface` goes out of scope, it creates a dangling reference — `LoadProhibited` crash.

**Fix**: Store TCP Interface wrappers in `std::list<RNS::Interface>` at global scope. Must use `std::list`, not `std::vector` — vector reallocation invalidates references.

### GPIO 10 must be HIGH

The T-Deck Plus requires GPIO 10 to be set HIGH at boot to enable all peripherals (display, radio, keyboard, etc.).

**Symptom**: Blank screen, no serial output, device appears dead.

**Fix**: `setup()` sets `pinMode(10, OUTPUT); digitalWrite(10, HIGH);` as the very first step.

---

## Storage Issues

### LittleFS not mounting

**Symptom**: `[E][vfs_api.cpp:24] open(): File system is not mounted`

**Causes**: First boot after flash erase, or partition table mismatch.

**Fix**: FlashStore calls `LittleFS.begin(true)` which auto-formats on first use. If it persists, erase flash and reflash:

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* erase-flash
```

### SD card not detected

**Check**: SD card must be FAT32. The SD card shares the SPI bus with the radio and display (CS=GPIO 39).

**Fix**: Ensure SD card is inserted before power-on. Check serial output for `[SD]` messages. The device works without an SD card — flash is used as fallback.

### Messages not persisting

**Check**: Verify SD card directories exist. Boot should auto-create `/ratputer/messages/`, `/ratputer/contacts/`, etc.

**Fix**: If directories are missing, reboot with SD card inserted. `SDStore::formatForRatputer()` creates the full directory tree on boot.

---

## WiFi Issues

### AP and STA are separate modes

Ratdeck uses three WiFi modes: OFF, AP, STA. These are NOT concurrent — `WIFI_AP_STA` was removed because it consumed too much heap and caused instability.

- **AP mode**: Creates `ratdeck-XXXX` hotspot, TCP server on port 4242
- **STA mode**: Connects to an existing network, TCP client connections
- **OFF**: No WiFi

Switch modes in Setup -> Network. WiFi mode changes require reboot.

### TCP client won't connect

**Check**: WiFi must be in STA mode and connected to a network first. TCP clients are created after WiFi connection succeeds.

**Fix**: Verify the TCP endpoint address and port. Check serial for `[TCP]` messages. Auto-reconnect runs every 15 seconds.

### WiFi causes crashes

WiFi initialization is the most common crash source. The boot loop recovery system (3 consecutive failures -> WiFi OFF) handles this automatically.

**Manual fix**: If stuck, erase flash to reset NVS, reflash, and reconfigure WiFi settings.

---

## RF Debugging

### RSSI Monitor

Press **Ctrl+R** to sample RSSI continuously for 5 seconds. Transmit from another device during sampling. If RSSI stays at the noise floor (~-110 to -120 dBm), the RX front-end isn't hearing RF.

### Test Packet

Press **Ctrl+T** to send a test packet with FIFO readback verification. Confirms the TX path without involving Reticulum.

### Full Diagnostics

Press **Ctrl+D** for a complete diagnostic dump to serial:

| Field | Meaning |
|-------|---------|
| Identity hash | 16-byte Reticulum identity hash |
| Destination hash | LXMF destination address |
| Transport | ACTIVE or OFFLINE, endpoint or transport node |
| Paths / Links | Known Reticulum paths and active links |
| Freq / SF / BW / CR / TXP | Current radio parameters |
| SyncWord regs | Raw 0x0740/0x0741 values (should be 0x14/0x24) |
| DevErrors | SX1262 error register (0x0040 = PLL lock failure) |
| Current RSSI | Instantaneous RSSI in dBm |
| Free heap / PSRAM | Available memory |
| Flash | LittleFS used/total |
| Uptime | Seconds since boot |

---

## Factory Reset

### Settings-only reset

In Setup -> System -> Factory Reset: clears user config from flash and SD, then reboots. Radio, WiFi, display, and audio revert to defaults. Identity and messages are preserved.

### SD card wipe

Connect serial at 115200 baud. Power cycle the device and send `WIPE` within 500ms of boot. Deletes `/ratputer/*` on the SD card and recreates clean directories.

### Full flash erase

Erases everything — LittleFS, NVS, firmware:

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* erase-flash
```

Reflash firmware after erasing. A new identity will be generated. SD card data is preserved.

---

## Serial Log Tags

| Tag | Subsystem |
|-----|-----------|
| `[BOOT]` | Setup sequence |
| `[RADIO]` | SX1262 driver |
| `[LORA_IF]` | LoRa <-> Reticulum interface |
| `[WIFI]` | WiFi AP/STA |
| `[TCP]` | TCP client connections |
| `[LXMF]` | LXMF message protocol |
| `[SD]` | SD card storage |
| `[BLE]` | BLE Sideband transport |
| `[HOTKEY]` | Keyboard hotkey dispatch |
| `[ID]` | Identity management |

---

## Known Limitations

| Feature | Status | Notes |
|---------|--------|-------|
| Touchscreen | Disabled | GT911 needs calibration |
| GPS | Pins defined | Not yet implemented |
| Split packets | Header flag defined | Single-frame LoRa only (max ~254 bytes) |
| TCP client leak | Minor | Stopped clients not freed (Transport holds refs) |
