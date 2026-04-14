# M5NanoC6 — standalone code

**Board enum:** `board_M5NanoC6` (140)
**SoC:** ESP32-C6 (WiFi 6 + BLE + 802.15.4 Thread/Zigbee)

Small module — a WS2812 + blue LED + a single button.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 | Yes, GPIO20 | Adafruit_NeoPixel (ESP32 Arduino ≥ 3.0) |
| Blue user LED | Yes, GPIO7 | Arduino digitalWrite |
| BtnA | Yes, GPIO9 | Arduino digitalRead |
| Grove Port A I²C | Yes | Arduino Wire |

No display, speaker, mic, IMU, RTC, battery.

---

## Pin map

| Purpose | GPIO |
|---------|------|
| WS2812 | 20 |
| Blue LED | 7 |
| BtnA | 9 (⚠️ strapping, same caveat as GPIO0 on C3/S3) |
| Grove SCL / SDA | 1 / 2 |

---

## Thread / Zigbee

ESP32-C6 has an IEEE 802.15.4 radio. To use Thread/Zigbee:
- **Arduino:** install `esp-thread-br` or the `OpenThread` library via PlatformIO.
- **ESP-IDF:** enable `CONFIG_OPENTHREAD_ENABLED=y` and follow [ESP-IDF OpenThread docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c6/api-reference/network/esp_openthread.html).

M5Unified does **not** wrap the 15.4 radio — you use Espressif's API directly.

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 20, NEO_GRB + NEO_KHZ800);

void setup() {
  led.begin(); led.setBrightness(64);
  pinMode(9, INPUT_PULLUP);
  pinMode(7, OUTPUT);
}

void loop() {
  bool pressed = digitalRead(9) == LOW;
  digitalWrite(7, pressed);              // blue LED echoes button
  led.setPixelColor(0, pressed ? 0xFF0000 : 0x00FF00);
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
| NanoC6 pin assignment | [src/M5Unified.cpp:1650](src/M5Unified.cpp#L1650) |
