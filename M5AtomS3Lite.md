# M5AtomS3Lite — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5AtomS3Lite` (137)
**SoC:** ESP32-S3

AtomS3 without LCD — single WS2812 LED on the face instead.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 LED (GPIO35) | Yes | Adafruit_NeoPixel |
| BtnA (face) | Yes, GPIO41 | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |
| Internal I²C (bottom pads) | Yes (unused) | Arduino Wire1 |
| Display / IMU / RTC / Speaker / Mic | **No** | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 data | 35 |
| BtnA | 41 |
| Grove (Port A) SCL / SDA | 1 / 2 |
| Internal I²C SCL / SDA | 39 / 38 |

---

## 1. WS2812 (GPIO35)

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 35, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  led.setPixelColor(0, 0x00FF00);
  led.show();
}
```

Copy the M5Unified RMT driver from [src/utility/led/LED_Strip_Class.cpp](src/utility/led/LED_Strip_Class.cpp) if you want the exact timing M5 uses (optimized for batched updates).

---

## 2. Button (GPIO41)

```cpp
pinMode(41, INPUT_PULLUP);
if (digitalRead(41) == LOW) { /* pressed */ }
```

---

## 3. Grove port (I²C or GPIO)

```cpp
#include <Wire.h>
Wire.begin(2 /* SDA */, 1 /* SCL */, 400000);
```

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 35, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(32);
  pinMode(41, INPUT_PULLUP);
}

void loop() {
  led.setPixelColor(0, digitalRead(41) == LOW ? led.Color(255, 0, 0) : led.Color(0, 0, 255));
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
| AtomS3Lite pin assignment | [src/M5Unified.cpp:1656](src/M5Unified.cpp#L1656) |
