# M5Dial Hardware & API Reference

## Contents
- Device specifications
- Arduino IDE board setup
- Required libraries
- Initialisation
- Display API
- Rotary encoder API
- Touch API
- Centre button API
- Display constants

---

## Device specifications

| Property | Value |
|---|---|
| SoC | ESP32-S3 |
| Display | 1.28" round GC9A01, 240x240 pixels |
| Encoder | Rotary encoder with centre push button |
| Touch | Capacitive touch (same surface as display) |
| Connectivity | WiFi 802.11 b/g/n, Bluetooth 5 |
| USB | USB-C (flashing and power) |
| Flash | 8 MB |
| PSRAM | 8 MB |

---

## Arduino IDE board setup

1. `File -> Preferences -> Additional Boards Manager URLs` — add:
   ```
   https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/arduino/package_m5stack_index.json
   ```
2. `Tools -> Board -> Boards Manager` — search **M5Stack** — Install (v2.1.0 or newer).
3. `Tools -> Board -> M5Stack -> M5Dial`

---

## Required libraries

Install all via Arduino IDE Library Manager:

| Library | Search term | Notes |
|---|---|---|
| M5Dial | `M5Dial` by M5Stack | Board-specific header; provides encoder, display, touch |
| WebSockets | `WebSockets by Markus Sattler` | WebSocketsServer for push to browser |
| WiFi | built-in | Part of ESP32 Arduino core |
| WebServer | built-in | HTTP server |
| DNSServer | built-in | Captive portal DNS redirect |
| Preferences | built-in | NVS key-value store |

Do NOT use `M5Unified.h` as the primary include — it does not expose `M5Dial.Encoder`.

---

## Initialisation

```cpp
#include <M5Dial.h>

void setup() {
  auto cfg = M5.config();
  M5Dial.begin(cfg, true, false);  // (config, enableEncoder, enableRFID)
  M5Dial.Display.setRotation(0);
  M5Dial.Display.setBrightness(200);  // 0-255
}

void loop() {
  M5Dial.update();  // must be called every loop — updates touch, encoder, button
}
```

---

## Display API

The display is `M5Dial.Display` (a LovyanGFX / M5GFX instance).
Screen centre is (120, 120). The screen is circular — corners are outside the visible area.

```cpp
M5Dial.Display.fillScreen(TFT_BLACK);
M5Dial.Display.setTextDatum(MC_DATUM);       // centre-align text
M5Dial.Display.setTextColor(TFT_WHITE, TFT_BLACK);
M5Dial.Display.setTextSize(2);               // 1-4
M5Dial.Display.drawString("Hello", 120, 120);
M5Dial.Display.fillCircle(x, y, r, color);
M5Dial.Display.drawCircle(x, y, r, color);
M5Dial.Display.drawLine(x1, y1, x2, y2, color);
M5Dial.Display.drawPixel(x, y, color);
M5Dial.Display.color565(r, g, b);            // convert RGB to 16-bit colour
```

Common colours: `TFT_BLACK`, `TFT_WHITE`, `TFT_RED`, `TFT_GREEN`, `TFT_BLUE`,
`TFT_CYAN`, `TFT_YELLOW`, `TFT_DARKGREY`, `0xFD20` (orange).

Arc drawing — no native arc primitive; iterate degrees manually:
```cpp
for (float deg = startDeg; deg <= endDeg; deg += 0.5f) {
  float rad = deg * DEG_TO_RAD;
  float cr = cosf(rad), sr = sinf(rad);
  for (int r = innerR; r <= outerR; r++)
    M5Dial.Display.drawPixel(CX + (int)(r*cr), CY + (int)(r*sr), color);
}
```

Add this if `DEG_TO_RAD` is not defined by the libraries included:
```cpp
#ifndef DEG_TO_RAD
#define DEG_TO_RAD 0.017453292519943295f
#endif
```

---

## Rotary encoder API

The raw encoder counts 4 units per physical click. Use integer division to get clean clicks.

```cpp
long lastEncRaw = 0;

// In setup():
lastEncRaw = M5Dial.Encoder.read();

// In loop() (after M5Dial.update()):
long raw   = M5Dial.Encoder.read();
long delta = raw - lastEncRaw;
int clicks = (int)(delta / 4);          // +1 per clockwise click, -1 counter-clockwise
if (clicks != 0) {
  lastEncRaw += clicks * 4;             // consume only whole clicks
  value = constrain(value + clicks, MIN, MAX);
}
```

---

## Touch API

```cpp
// In loop() (after M5Dial.update()):
auto touch = M5Dial.Touch.getDetail();
if (touch.wasPressed()) {
  int x = touch.x;   // 0-239
  int y = touch.y;   // 0-239
  // handle tap
}
```

Coordinates are in display pixels (0–239 on each axis), centre (120,120).

---

## Centre button API

The centre button is the encoder knob pressed down.

```cpp
// In loop() (after M5Dial.update()):
if (M5Dial.BtnA.wasPressed()) {
  // handle press
}
```
