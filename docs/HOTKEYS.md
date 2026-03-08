# Ratdeck — Hotkey Reference

## Hotkeys (Ctrl+key)

| Shortcut | Action |
|----------|--------|
| Ctrl+H | Toggle help overlay (shows all hotkeys on screen) |
| Ctrl+M | Jump to Messages tab |
| Ctrl+N | Compose new message |
| Ctrl+S | Jump to Settings tab |
| Ctrl+A | Force announce to all interfaces (LoRa + TCP) |
| Ctrl+D | Dump full diagnostics to serial |
| Ctrl+T | Send radio test packet |
| Ctrl+R | RSSI monitor (5-second continuous sampling) |

## Navigation

| Input | Action |
|-------|--------|
| Trackball up/down | Scroll / navigate lists |
| Trackball left/right | Cycle tabs |
| Trackball click | Select / confirm |
| Trackball long-press (1.2s) | Context menu |
| `,` (comma) | Previous tab |
| `/` (slash) | Next tab |
| Enter | Select / confirm / send message |
| Esc | Back / cancel |
| Backspace | Delete character / go back (if input empty) |

## Text Input

When a text field is active (message compose, WiFi password, etc.):
- Type normally to enter characters
- **Backspace** to delete
- **Enter** to submit / send
- **Esc** to cancel and go back

## Tabs

| Tab | Name | Contents |
|-----|------|----------|
| 1 | Home | Name, LXMF address, connection status, online node count |
| 2 | Friends | Saved contacts (display name only) |
| 3 | Msgs | Conversation list — sorted by most recent, with preview and unread dots |
| 4 | Peers | All discovered nodes — contacts section + online section |
| 5 | Setup | 7-category settings (Device, Display, Radio, Network, Audio, Info, System) |

## Home Screen

- **Enter / trackball click**: Announce to all connected interfaces (LoRa + TCP)

## Messages Screen

- **Up/Down**: Navigate conversations
- **Enter**: Open conversation
- **Long-press**: Context menu (Add Friend / Delete Chat / Cancel)
  - Navigate menu with Up/Down, confirm with Enter

## Message View (Chat)

- **Type**: Compose message
- **Enter**: Send message
- **Up/Down**: Scroll message history
- **Esc / Backspace** (empty input): Go back to conversation list

## Peers Screen

- **Up/Down**: Navigate node list
- **Enter**: Open chat with selected node
- **'s' key**: Toggle saved/friend status
- **Long-press**: Unsaved node → add as friend; saved friend → delete prompt

## Settings Screen

- **Up/Down**: Navigate settings items
- **Enter**: Toggle / edit selected setting
- **Left/Right**: Adjust numeric values, cycle enum choices
- Categories: Device, Display & Input, Radio, Network, Audio, Info, System

## Serial Diagnostics

**Ctrl+D** prints to serial (115200 baud):
- Identity hash, destination hash, transport status
- Path/link counts
- Radio parameters (freq, SF, BW, CR, TX power, preamble)
- SX1262 register dump (sync word, IQ, LNA, OCP, TX clamp)
- Device errors, current RSSI
- Free heap, PSRAM, flash usage, uptime

**Ctrl+T** sends a test packet and verifies FIFO readback.

**Ctrl+R** samples RSSI continuously for 5 seconds.
