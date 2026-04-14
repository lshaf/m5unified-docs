# M5StampS3 — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5StampS3` (136)
**SoC:** ESP32-S3

Flat stamp-style S3 module. All pads exposed.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 LED | Yes, GPIO21 | Adafruit_NeoPixel |
| BtnA (pad) | Yes, GPIO0 | Arduino digitalRead |
| External I²C (Grove) | Yes | Arduino Wire |
| Display / Speaker / Mic / IMU / RTC | **No** | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 | 21 |
| BtnA | 0 (⚠️ strapping pin — must be HIGH during reset for normal boot) |
| Grove SCL / SDA | 15 / 13 |

---

## GPIO0 caveat

BtnA uses GPIO0, which is the boot-mode strapping pin. If you hold BtnA during power-up, the chip enters USB download mode. After boot it's a regular input you can poll.

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 21, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  pinMode(0, INPUT_PULLUP);
}

void loop() {
  led.setPixelColor(0, digitalRead(0) == LOW ? 0xFF0000 : 0x0000FF);
  led.show();
  delay(20);
}
```

---

## Power management

USB/external-powered only — no battery, no charger.

---

## Source references

| Topic | File : line |
|-------|-------------|
| StampS3 pin assignment | [src/M5Unified.cpp:1673](src/M5Unified.cpp#L1673) |
