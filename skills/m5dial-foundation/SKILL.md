---
name: m5dial-foundation
description: Foundational rules and patterns for writing Arduino firmware for the M5Stack Dial (M5Dial). Use whenever the user asks to program, extend, or debug code for the M5Dial device. Covers board selection, required libraries, display/encoder/touch API, WiFi manager with NVS captive portal, WebSocket protocol, raw-string safety rules, and memory constraints.
---

## Overview

The M5Stack Dial is a round ESP32-S3 device with a 240x240 GC9A01 circular display,
a rotary encoder with centre button, a capacitive touch screen, and built-in WiFi/BT.

All firmware details are in the reference files below.
Load `references/hardware.md` for device, board, and API facts.
Load `references/wifi-websocket.md` for the WiFi manager and WebSocket patterns.
Load `references/arduino-pitfalls.md` for compiler pitfalls and raw-string rules.

## Reference Files

- `references/hardware.md`       — device specs, board setup, library install, display/encoder/touch API
- `references/wifi-websocket.md` — WiFi state machine, NVS credentials, captive portal, WebSocket protocol
- `references/arduino-pitfalls.md` — raw-string rules, broadcastTXT fix, deferred-connect pattern, memory tips

## Quick Rules (always apply)

1. Use `#include <M5Dial.h>` — never `M5Unified.h` for this device.
2. Call `M5Dial.begin(cfg, true, false)` in `setup()`.
3. All display/touch/encoder calls use `M5Dial.Display`, `M5Dial.Touch`, `M5Dial.Encoder`, `M5Dial.BtnA`.
4. Use `R"HTML(...)HTML"` raw string delimiter — never plain `R"()"`.
5. Never use `&#xNNNN;` HTML entities or JS backtick template literals inside raw strings.
6. Never call `startConnecting()` from inside an HTTP handler — use a `pendingConnect` flag instead.
7. WebSocket messages use short prefixed strings: `V:50`, `T:x,y`, `B` — keep them under 20 bytes.
8. Never store the HTML page string in a global variable — build it on demand inside the handler to avoid eating heap at all times.
