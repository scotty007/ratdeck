# Ratdeck — Developer Guide

## Overview

Ratdeck is a standalone Reticulum mesh node with LXMF encrypted messaging for the LilyGo T-Deck Plus (ESP32-S3). It is NOT an RNode — it does not speak KISS protocol. It runs the full Reticulum stack (microReticulum) directly on the device.

Key characteristics:
- Standalone operation — no host computer required
- LoRa transport with 1-byte header framing
- WiFi transport — AP mode (TCP server) or STA mode (TCP client)
- BLE transport — NimBLE Sideband interface
- LXMF encrypted messaging with Ed25519 signatures
- LVGL v8.4 UI with 5-tab navigation
- JSON-based runtime configuration with SD card + flash dual-backend

## Configuration System

### Compile-Time (`Config.h`)

Feature flags (`HAS_LORA`, `HAS_WIFI`, `HAS_BLE`, etc.), storage paths, protocol limits, power defaults. Changed only by editing source and recompiling.

### Runtime (`UserConfig`)

JSON-based settings persisted to storage. Schema defined by `UserSettings` struct in `UserConfig.h`. Dual-backend persistence: `UserConfig::load(SDStore&, FlashStore&)` reads from SD first (`/ratputer/config/user.json`), falls back to flash (`/config/user.json`). `save()` writes to both.

## Transport Architecture

### InterfaceImpl Pattern

All transport interfaces inherit from `RNS::InterfaceImpl`:
- `start()` / `stop()` — lifecycle
- `send_outgoing(const RNS::Bytes& data)` — transmit
- `loop()` — poll for incoming data, call `receive_incoming()` to push up to Reticulum

### HDLC Framing (WiFi + TCP)

TCP connections use HDLC-like byte framing:
- `0x7E` — frame delimiter (start/end)
- `0x7D` — escape byte
- `0x20` — XOR mask for escaped bytes

Any `0x7E` or `0x7D` in payload is escaped as `0x7D (byte ^ 0x20)`.

### LoRa 1-Byte Header

Every LoRa packet has a 1-byte header prepended:
- Upper nibble: random sequence number
- Lower nibble: flags (`0x01` = split, not currently implemented)

## Screen System (LVGL v8.4)

All screens extend `LvScreen` base class:
- `createUI(lv_obj_t* parent)` — build LVGL widgets
- `onEnter()` — called when screen becomes active
- `handleKey(const KeyEvent&)` — keyboard/trackball input
- `handleLongPress()` — trackball long-press (1200ms)

### Active Screens

| Screen | Class | Tab | Purpose |
|--------|-------|-----|---------|
| Boot | LvBootScreen | — | Boot animation with progress bar |
| Home | LvHomeScreen | 1 | Name, LXMF address, status, online nodes |
| Friends | LvContactsScreen | 2 | Saved contacts (display name only) |
| Msgs | LvMessagesScreen | 3 | Conversation list, sorted by recent, previews |
| Peers | LvNodesScreen | 4 | All discovered nodes (contacts + online) |
| Setup | LvSettingsScreen | 5 | 7-category settings editor |
| Help | LvHelpOverlay | — | Hotkey reference modal (Ctrl+H) |
| Name | LvNameInputScreen | — | First-boot name entry |

### Theme

Matrix green (#00FF41) on black. Layout: 320x240, status bar 20px top, tab bar 20px bottom, content 200px.

Key colors:
- `PRIMARY` (0x00FF41) — main text, active elements
- `ACCENT` (0x00FFFF) — cyan highlights, incoming messages
- `MUTED` (0x559955) — secondary text
- `SELECTION_BG` (0x004400) — selected row
- `ERROR_CLR` (0xFF3333) — errors, disconnected indicators

## How To: Add a New Screen

1. Create `src/ui/screens/LvMyScreen.h` and `LvMyScreen.cpp`
2. Inherit from `LvScreen` — implement `createUI()`, `handleKey()`, optionally `onEnter()`
3. In `createUI()`, build LVGL widgets on the provided parent container
4. Add a global instance in `main.cpp`
5. Wire it up: add to `tabScreens[]` array for tab navigation, or navigate to it from a hotkey/callback via `ui.showScreen()`

## How To: Add a New Hotkey

1. In `main.cpp`, create a callback function: `void onHotkeyX() { ... }`
2. Register in `setup()`: `hotkeys.registerHotkey('x', "Description", onHotkeyX);`
3. Update `docs/HOTKEYS.md` and the help overlay text in `LvHelpOverlay.cpp`

## How To: Add a Settings Category

Settings uses a category/item system in `LvSettingsScreen`:

1. Add your category to the categories array
2. Define items with types: `READONLY`, `TOGGLE`, `INTEGER`, `ENUM_CHOICE`, `TEXT_INPUT`, `ACTION`
3. Add value getter/setter logic in the category handler
4. Changes that need reboot: set flag and show toast
5. Live changes: apply immediately in the setter callback
6. Persist with `userConfig->save(sdStore, flash)`

## How To: Add a New Transport Interface

1. Create a class inheriting from `RNS::InterfaceImpl`
2. Implement `start()`, `stop()`, `loop()`, `send_outgoing()`
3. In `loop()`, call `receive_incoming(data)` when data arrives
4. In `main.cpp`, construct the impl, wrap in `RNS::Interface`, register with `RNS::Transport`
5. Store the `RNS::Interface` wrapper in a `std::list` (not vector) — Transport holds references, and vector reallocation invalidates them

## Initialization Sequence

`setup()` runs these steps in order:

1. GPIO 10 HIGH — enables all T-Deck Plus peripherals
2. Serial begin (115200 baud)
3. Shared SPI bus init (SCK=40, MISO=38, MOSI=41)
4. I2C bus init (SDA=18, SCL=8)
5. Display init (LovyanGFX + LVGL)
6. Boot screen shown
7. Keyboard init (I2C 0x55)
8. Trackball init (GPIO ISRs on pins 0/1/2/3/15)
9. Register hotkeys (Ctrl+H/M/N/S/A/D/T/R)
10. Mount LittleFS (FlashStore)
11. Boot loop detection (NVS counter)
12. Radio init — SX1262 TCXO, calibrate, configure, enter RX
13. SD card init (shares SPI, must be after radio)
14. Reticulum init — identity load/generate (triple-redundant), transport start
15. MessageStore init (dual backend)
16. LXMF init + message callback
17. AnnounceManager init + contact load
18. Register announce handler with Transport
19. Load UserConfig (SD -> flash fallback)
20. Boot loop recovery check (force WiFi OFF if triggered)
21. Apply saved radio settings
22. WiFi start (AP, STA, or OFF based on config)
23. BLE init (NimBLE Sideband)
24. Power manager init + apply brightness/timeouts
25. Audio init
26. Name input screen (first boot only)
27. Switch to Home screen
28. Initial announce broadcast
29. Clear boot loop counter (NVS reset to 0)

## Main Loop

Single-threaded on core 1:

```
loop() {
  1. inputManager.update()          -- keyboard, trackball, touch polling
  2. Long-press dispatch             -- ui.handleLongPress() at 1200ms
  3. Key event dispatch              -- hotkeys -> LvInput::feedKey() -> screen handleKey()
  4. lv_timer_handler()              -- LVGL rendering (skipped when screen off)
  5. rns.loop()                      -- Reticulum + LoRa RX (throttled to 5ms)
  6. Auto-announce (5 min interval)
  7. lxmf.loop() + announce loop     -- message queue + deferred saves
  8. WiFi STA handler                -- connect/disconnect, TCP client creation
  9. WiFi/TCP/BLE loops              -- transport processing
  10. powerMgr.loop()                -- ACTIVE -> DIMMED -> SCREEN_OFF
  11. Status bar update (1Hz)         -- battery, signal indicators
  12. Heartbeat (5s serial)
}
```

## Memory Budget

ESP32-S3 with 8MB PSRAM. PSRAM is used for large allocations; SRAM (512 KB) for stack and DMA:

| State | Free Heap | Notes |
|-------|-----------|-------|
| Boot complete (WiFi OFF) | ~170 KB | Baseline |
| Boot complete (WiFi AP) | ~150 KB | WiFi stack + TCP server |
| Boot complete (WiFi STA) | ~140 KB | WiFi stack + TCP clients |
| With BLE enabled | ~120 KB | NimBLE stack |

Monitor with `Ctrl+D` -> `Free heap` in serial output.

## Compile-Time Limits

Defined in `Config.h`:

| Constant | Default | Purpose |
|----------|---------|---------|
| `RATDECK_MAX_NODES` | 200 | Max discovered nodes (PSRAM allows more) |
| `RATDECK_MAX_MESSAGES_PER_CONV` | 100 | Max messages stored per conversation |
| `FLASH_MSG_CACHE_LIMIT` | 20 | Recent messages per conv in flash (SD has full history) |
| `RATDECK_MAX_OUTQUEUE` | 20 | Max pending outgoing LXMF messages |
| `MAX_TCP_CONNECTIONS` | 4 | Max simultaneous TCP client connections |
| `TCP_RECONNECT_INTERVAL_MS` | 15000 | Retry interval for dropped TCP connections |
| `TCP_CONNECT_TIMEOUT_MS` | 500 | Timeout for TCP connect() |
| `PATH_PERSIST_INTERVAL_MS` | 60000 | Transport path save interval |
| `SCREEN_DIM_TIMEOUT_MS` | 30000 | Default screen dim timeout |
| `SCREEN_OFF_TIMEOUT_MS` | 60000 | Default screen off timeout |

## Debugging

### Serial Output

All subsystems log with `[TAG]` prefixes at 115200 baud. Key tags: `[BOOT]`, `[RADIO]`, `[LORA_IF]`, `[WIFI]`, `[TCP]`, `[LXMF]`, `[SD]`, `[BLE]`.

### Radio Debugging

- `Ctrl+D` — dumps SX1262 registers, identity, transport status, heap, PSRAM, uptime
- `Ctrl+T` — sends test packet with FIFO readback verification
- `Ctrl+R` — 5-second continuous RSSI sampling

### Core Dump

ESP-IDF stores a core dump in the `coredump` partition (64 KB at 0xFF0000):

```bash
python3 -m esptool --chip esp32s3 --port /dev/cu.usbmodem* \
    read-flash 0xFF0000 0x10000 coredump.bin
python3 -m esp_coredump info_corefile -t raw -c coredump.bin \
    .pio/build/ratdeck_915/firmware.elf
```

### Common Crashes

| Crash | Cause | Fix |
|-------|-------|-----|
| `LoadProhibited` at transport loop | Dangling `Interface&` reference | Store in `std::list` (not vector, not local scope) |
| `Stack overflow` | Deep call chain or recursive render | Increase stack size or reduce nesting |
| `Guru Meditation` on WiFi init | Heap exhaustion | Reduce TCP connections, check for leaks |
| Boot loop (3+ failures) | WiFi or TCP init crash | Boot loop recovery auto-disables WiFi |
