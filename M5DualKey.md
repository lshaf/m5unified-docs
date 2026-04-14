# M5DualKey — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5DualKey` (147)
**SoC:** ESP32-S3

Two-button macro-pad / quick-switch unit. Essentially two GPIO switches exposed as USB-HID or BLE keyboard.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| BtnA | Yes, GPIO17 | Arduino digitalRead |
| BtnB | Yes, GPIO0 | Arduino digitalRead |
| Grove Port A | Yes | Arduino Wire |

No display, no LED, no audio, no IMU/RTC.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| BtnA | 17 |
| BtnB | 0 (⚠️ strapping — hold during reset → download mode) |
| Grove SCL / SDA | per schematic (check M5DualKey Unit page) |

---

## Minimal sketch (HID keyboard via TinyUSB)

```cpp
#include <USB.h>
#include <USBHIDKeyboard.h>
USBHIDKeyboard Keyboard;

void setup() {
  USB.begin();
  Keyboard.begin();
  pinMode(17, INPUT_PULLUP);
  pinMode(0,  INPUT_PULLUP);
}

void loop() {
  static bool lastA = HIGH, lastB = HIGH;
  bool a = digitalRead(17), b = digitalRead(0);
  if (lastA == HIGH && a == LOW) Keyboard.print("A");
  if (lastB == HIGH && b == LOW) Keyboard.print("B");
  lastA = a; lastB = b;
  delay(10);
}
```

---

## Power management

USB-powered only — no battery.

---

## Source references

| Topic | File : line |
|-------|-------------|
| DualKey button read | [src/M5Unified.cpp:2527](src/M5Unified.cpp#L2527) |
