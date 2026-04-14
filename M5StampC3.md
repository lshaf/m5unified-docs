# M5StampC3 — standalone code

**Board enum:** `board_M5StampC3` (134)
**SoC:** ESP32-C3 (single-core RISC-V, 4 MB flash)

Flat SMD stamp module.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 LED | Yes, GPIO2 | Adafruit_NeoPixel |
| BtnA | Yes, **GPIO3** | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 | 2 |
| BtnA | 3 |
| Grove SCL / SDA | 0 / 1 |

GPIO0/1 are regular pins on C3 (no strapping conflict here).

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 2, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  pinMode(3, INPUT_PULLUP);
}

void loop() {
  led.setPixelColor(0, digitalRead(3) == LOW ? 0x00FF00 : 0x0000FF);
  led.show();
  delay(20);
}
```

---

## Power management

External-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StampC3 pin assignment | [src/M5Unified.cpp:1640](src/M5Unified.cpp#L1640) |
