# M5StampPico — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StampPico` (133)
**SoC:** ESP32-PICO-D4

Flat stamp-style module.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 LED | Yes, GPIO27 | Adafruit_NeoPixel |
| BtnA (side pad) | Yes, GPIO39 | Arduino digitalRead |
| Internal I²C (unused) | pads | Wire1 |
| External I²C (Grove) | Yes | Wire |

No display, speaker, mic, IMU, RTC, battery, or SD onboard.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 | 27 |
| BtnA | 39 |
| Internal I²C SCL / SDA | 22 / 21 |
| Grove (Port A) SCL / SDA | 33 / 32 |

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 27, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  pinMode(39, INPUT_PULLUP);
}

void loop() {
  led.setPixelColor(0, digitalRead(39) == LOW ? led.Color(255, 0, 0) : led.Color(0, 255, 0));
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
| StampPico pin assignment | [src/M5Unified.cpp:1622](src/M5Unified.cpp#L1622) |
