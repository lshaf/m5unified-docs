# M5StampC3U — standalone code

**Board enum:** `board_M5StampC3U` (135)
**SoC:** ESP32-C3

StampC3 in a USB-A plug form factor. Button pin moved from GPIO3 → **GPIO9** because GPIO3 is used for USB D+.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 | 2 |
| **BtnA** | **9** (⚠️ strapping pin; same caveat as S3 GPIO0) |
| Grove SCL / SDA | 0 / 1 |

Everything else matches [M5StampC3.md](M5StampC3.md).

---

## Minimal sketch

Same code as StampC3 but change `pinMode(3, ...)` → `pinMode(9, ...)`:

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 2, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  pinMode(9, INPUT_PULLUP);
}

void loop() {
  led.setPixelColor(0, digitalRead(9) == LOW ? 0xFF0000 : 0x00FF00);
  led.show();
  delay(20);
}
```

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StampC3U pin assignment | [src/M5Unified.cpp:1644](src/M5Unified.cpp#L1644) |
