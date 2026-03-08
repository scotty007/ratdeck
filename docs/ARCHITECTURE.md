# Ratdeck — Architecture

## Overview

Ratdeck is a standalone Reticulum mesh node with LXMF encrypted messaging, built for the LilyGo T-Deck Plus (ESP32-S3). It runs the full Reticulum protocol stack on-device — no host computer, no gateway, no KISS protocol. LoRa, WiFi, and BLE transports operate independently or bridged.

## Layer Diagram

```
+---------------------------------------------+
|           UI Layer (LVGL v8.4)              |
|  Screens: Home, Friends, Msgs, Peers, Setup |
|  LvStatusBar, LvTabBar, LvInput            |
|  Theme: Matrix green (#00FF41) on black     |
+---------------------------------------------+
|         Application Layer                   |
|  LXMFManager  AnnounceManager              |
|  IdentityManager  UserConfig               |
|  MessageStore  AudioNotify                  |
+---------------------------------------------+
|         Reticulum Layer                     |
|  ReticulumManager (microReticulum C++)      |
|  Identity, Destination, Transport           |
+---------------------------------------------+
|         Transport Layer                     |
|  LoRaInterface   WiFiInterface              |
|  TCPClientInterface   BLESideband           |
+---------------------------------------------+
|         Storage Layer                       |
|  FlashStore (LittleFS, 7.875 MB)            |
|  SDStore (FAT32 microSD)                    |
|  NVS (ESP32 Preferences)                    |
+---------------------------------------------+
|         Hardware Layer                      |
|  SX1262 Radio    ST7789V Display            |
|  I2C Keyboard    GPIO Trackball             |
|  ESP32-S3 (8MB PSRAM, 16MB Flash)           |
+---------------------------------------------+
```

## Directory Structure

```
src/
+-- main.cpp                    Setup + main loop (single-threaded, core 1)
+-- config/
|   +-- BoardConfig.h           GPIO pins, SPI config, hardware constants
|   +-- Config.h                Version, feature flags, storage paths, limits
|   +-- UserConfig.*            Runtime settings (JSON, dual SD+flash backend)
+-- radio/
|   +-- SX1262.*                SX1262 LoRa driver (register-level, async TX)
|   +-- RadioConstants.h        SX1262 register addresses and command bytes
+-- hal/
|   +-- Display.*               LovyanGFX ST7789V driver + LVGL flush callback
|   +-- Power.*                 Screen dim/off/wake, battery ADC, brightness
|   +-- Keyboard.*              I2C keyboard (ESP32-C3 at 0x55)
|   +-- Trackball.*             GPIO trackball (ISR-based, 5 pins)
|   +-- TouchInput.*            GT911 capacitive touch (disabled)
+-- input/
|   +-- InputManager.*          Unified keyboard + trackball + touch polling
|   +-- HotkeyManager.*         Ctrl+key dispatch table
+-- ui/
|   +-- Theme.h                 Color palette, layout constants
|   +-- LvTheme.*               LVGL theme application
|   +-- LvStatusBar.*           Signal bars + brand + battery %
|   +-- LvTabBar.*              5-tab bar with unread badges
|   +-- LvInput.*               Key event routing to active screen
|   +-- UIManager.*             Screen lifecycle, boot/normal modes
|   +-- screens/
|       +-- LvBootScreen.*      Boot animation with progress bar
|       +-- LvHomeScreen.*      Name, LXMF address, status, online nodes
|       +-- LvContactsScreen.*  Saved friends (display name only)
|       +-- LvMessagesScreen.*  Conversation list (sorted, previews, unread dots)
|       +-- LvMessageView.*     Chat bubbles + status colors + text input
|       +-- LvNodesScreen.*     Node browser (contacts + online sections)
|       +-- LvSettingsScreen.*  7-category settings editor
|       +-- LvHelpOverlay.*     Hotkey reference modal
|       +-- LvNameInputScreen.* First-boot name entry
+-- reticulum/
|   +-- ReticulumManager.*      microReticulum lifecycle, transport loop
|   +-- AnnounceManager.*       Node discovery, friend persistence
|   +-- LXMFManager.*           LXMF send/receive, queue, delivery tracking
|   +-- LXMFMessage.*           Wire format + JSON storage format
|   +-- IdentityManager.*       Multi-slot identity management
+-- transport/
|   +-- LoRaInterface.*         SX1262 <-> Reticulum (1-byte header)
|   +-- WiFiInterface.*         WiFi AP, TCP server on port 4242
|   +-- TCPClientInterface.*    WiFi STA, TCP client to remote nodes
|   +-- BLEInterface.*          BLE transport
|   +-- BLESideband.*           NimBLE Sideband interface
+-- storage/
|   +-- FlashStore.*            LittleFS with atomic writes
|   +-- SDStore.*               SD card (FAT32) with atomic writes
|   +-- MessageStore.*          Per-conversation dual-backend storage
+-- audio/
    +-- AudioNotify.*           Notification sounds
```

## Data Flow

### Incoming LoRa Packet

```
SX1262 IRQ (DIO1, GPIO 45) -> SX1262::receive() reads FIFO
    -> LoRaInterface::loop() strips 1-byte header
        -> RNS::InterfaceImpl::receive_incoming()
            -> RNS::Transport processes packet
                +-- Announce -> AnnounceManager callback -> UI update
                +-- LXMF data -> LXMFManager -> MessageStore (flash + SD) -> UI notification
                +-- Path/link -> Transport table update -> persist to flash + SD backup
```

### Outgoing LXMF Message

```
User types message -> LvMessageView -> LXMFManager::send()
    -> Save to MessageStore (QUEUED status)
    -> Pack: dest_hash(16) + src_hash(16) + signature(64) + msgpack([ts, title, content, fields])
        -> RNS::Packet -> RNS::Transport selects interface
            +-- LoRaInterface -> prepend 1-byte header -> SX1262::beginPacket/endPacket (async)
            +-- TCPClient -> HDLC frame (0x7E delimit, 0x7D escape) -> TCP socket
    -> StatusCallback fires -> re-save with SENT/DELIVERED/FAILED status
```

### Config Save

```
LvSettingsScreen -> UserConfig::save(sd, flash)
    +-- serialize to JSON string
    +-- SDStore::writeAtomic("/ratputer/config/user.json")   -> .tmp -> verify -> .bak -> rename
    +-- FlashStore::writeAtomic("/config/user.json")         -> .tmp -> verify -> .bak -> rename
```

## Key Design Decisions

### Radio Driver

Register-level SX1262 driver with no library abstraction. Async TX mode (`endPacket(true)`) returns immediately — `isTxBusy()` polls completion in `LoRaInterface::loop()`. `waitOnBusy()` calls a yield callback so LVGL stays responsive during SPI waits. TCXO 1.8V, DIO2 as RF switch, 8 MHz SPI clock.

### Display

LovyanGFX drives the ST7789V over SPI at 15 MHz. LVGL v8.4 renders to a flush buffer with DMA transfer. 320x240 landscape, 20px status bar top, 20px tab bar bottom, 200px content area.

### Reticulum Integration

microReticulum C++ library with LittleFS-backed filesystem. Device runs as either an Endpoint (default) or Transport Node (configurable in Settings). All interfaces (LoRa, WiFi, TCP, BLE) register with `RNS::Transport`.

### LXMF Wire Format

```
[dest_hash:16][src_hash:16][signature:64][msgpack_payload]
```

MsgPack payload: `fixarray(4): [timestamp(float64), title(str), content(str), fields(map)]`

Signature covers: `dest_hash || src_hash || msgpack_payload`

Messages under MDU (~254 bytes for LoRa) are sent as single direct packets. Stored as JSON per-conversation in flash and SD.

### Shared SPI Bus

Display, SX1262 radio, and SD card all share SPI2_HOST (FSPI on ESP32-S3). Single-threaded access from `loop()` — no mutex needed. Arduino FSPI=0 maps to SPI2 hardware; do NOT use `SPI2_HOST` IDF constant directly.

### WiFi Transport

Three mutually exclusive modes:
- **OFF**: No WiFi — saves power and ~20 KB heap
- **AP**: Creates `ratdeck-XXXX` hotspot, TCP server on port 4242, HDLC framing
- **STA**: Connects to existing network, TCP client connections to configured endpoints

AP+STA concurrent mode was removed — consumed too much heap and caused instability.

### TCP Client Transport

Outbound TCP connections to remote Reticulum nodes. Created on WiFi STA connection, auto-reconnect on disconnect (15s interval). Settings changes apply live via `_tcpChangeCb` callback — `reloadTCPClients()` stops old clients and creates new ones. Switching servers clears transient nodes.

### Dual-Backend Storage

FlashStore (LittleFS, 7.875 MB partition) is the primary store. SDStore (FAT32 microSD) provides backup and extended capacity. Both use atomic writes (`.tmp` -> verify -> `.bak` -> rename) to prevent corruption on power loss. UserConfig, MessageStore, and AnnounceManager write to both backends.

### Identity System

Triple-redundant storage: Flash (LittleFS), SD card, and NVS (ESP32 Preferences). Load order: Flash -> SD -> NVS -> generate new. On any successful load or creation, saves to all three tiers. Multi-slot identity management with per-slot display names.

### Transport Reference Stability

`RNS::Transport::_interfaces` stores `Interface&` references (not copies). All `RNS::Interface` wrappers must outlive the transport — stored in `std::list` (not vector, which would invalidate references on reallocation).

### Power Management

Three states: ACTIVE -> DIMMED -> SCREEN_OFF. Strong activity (keyboard, trackball nav/click) wakes from any state. Weak activity (trackball raw movement) wakes from DIMMED only. Configurable timeouts via Settings.

### Boot Loop Recovery

NVS counter tracks consecutive boot failures. After 3 failures, WiFi is forced OFF on next boot. Counter resets to 0 at end of successful `setup()`.

### Input System

Trackball click uses deferred release with 80ms GPIO debounce. Long press fires at 1200ms hold, suppresses click on release. Nav events are rate-limited to 200ms with threshold of 3 accumulated deltas. I2C keyboard at address 0x55 with interrupt on GPIO 46.
