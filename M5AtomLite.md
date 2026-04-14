# M5AtomLite — standalone code

**Board enum:** `board_M5AtomLite` (128)
**SoC:** ESP32-PICO-D4

Plain Atom cube with a single WS2812 face LED.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| WS2812 RGB LED (face, 1 pixel) | Yes, GPIO27 | Adafruit_NeoPixel or FastLED or ESP-IDF RMT |
| Onboard button (face "M5") | Yes, GPIO39 | Arduino `digitalRead` |
| Internal I²C bus | Yes (unused on stock) | Arduino `Wire1` |
| External I²C (Port A, Grove) | Yes | Arduino `Wire` |
| Display / Speaker / Mic / IMU / RTC / SD | **No** (HATs add them) | — |

---

## Pin map (full)

| Purpose | GPIO | Notes |
|---------|------|-------|
| WS2812 data | 27 | 1 pixel, ~20 mA at full white |
| BtnA (face) | 39 | active LOW, needs `INPUT_PULLUP` |
| Internal I²C SCL | 21 | |
| Internal I²C SDA | 25 | |
| Grove (Port A) SCL | 32 | |
| Grove (Port A) SDA | 26 | |
| IR TX | 12 | |

### Boot-time quirk: GPIO0 forced HIGH

M5Unified does `gpio_hi(GPIO_NUM_0)` at the top of every ESP32-original Atom `begin()` — it's a fix for RF interference from the CH552 USB-UART that couples into WiFi when GPIO0 floats low. If you run without M5Unified and see bad WiFi performance, add:
```cpp
pinMode(0, OUTPUT);
digitalWrite(0, HIGH);
```
See [src/M5Unified.cpp:1617](../src/M5Unified.cpp#L1617) for the original code.

---

## 1. WS2812 LED (GPIO27)

### With Adafruit_NeoPixel

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel pixel(1, 27, NEO_GRB + NEO_KHZ800);

void setup() {
  pixel.begin();
  pixel.setBrightness(32);            // 0..255 — keep low for USB
  pixel.setPixelColor(0, pixel.Color(255, 0, 0));
  pixel.show();
}
```

### With M5Unified's LED class (standalone, no M5Unified.begin)

The LED class in [src/utility/LED_Class.cpp](src/utility/LED_Class.cpp) is usable without `M5.begin()`:

```cpp
#include <M5Unified.h>       // only for the LED_Strip_Class header
m5::LED_Strip_Class led;

void setup() {
  led.begin(27 /* pin */, 1 /* count */);
  led.setBrightness(32);
  led.setPixel(0, 0xFF0000);
  led.show();
}
```

### Raw ESP-IDF RMT (smallest dep)

See [src/utility/led/LED_Strip_Class.cpp](src/utility/led/LED_Strip_Class.cpp) — it uses the RMT peripheral directly; you can copy the `_encode_pixel()` function verbatim.

---

## 2. Button (GPIO39)

```cpp
#define BTN_A 39
void setup() { pinMode(BTN_A, INPUT_PULLUP); }
void loop()  { if (digitalRead(BTN_A) == LOW) { /* pressed */ } }
```

For debounce/click/hold, copy [src/utility/Button_Class.cpp](src/utility/Button_Class.cpp) (framework-free, ~300 lines).

---

## 3. Grove Port (I²C or GPIO)

```cpp
#include <Wire.h>
Wire.begin(26 /* SDA */, 32 /* SCL */, 400000);
```

As plain GPIO: GPIO26 and GPIO32 are regular ADC1 pins.

---

## 4. Internal I²C (bottom castellations)

```cpp
Wire1.begin(25 /* SDA */, 21 /* SCL */, 400000);
```

---

## Full minimal standalone sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 27, NEO_GRB + NEO_KHZ800);

void setup() {
  pinMode(0, OUTPUT); digitalWrite(0, HIGH);   // WiFi-noise fix (optional)
  pinMode(39, INPUT_PULLUP);
  led.begin(); led.setBrightness(32);
}

void loop() {
  static int hue = 0;
  led.setPixelColor(0, led.ColorHSV((hue += 256) & 0xFFFF));
  led.show();
  if (digitalRead(39) == LOW) { hue = 0; }
  delay(20);
}
```

---

## Power management

**USB-powered only** — no battery, no charger IC, no PMIC. 5 V → AMS1117 → 3.3 V. `M5.Power.getBatteryLevel()` returns `-2` (unsupported).

If you're running off a Grove-connected external battery, monitor voltage via an ADC on a Grove pin yourself.

---

## Source references

| Topic | File : line |
|-------|-------------|
| WS2812 RMT driver | [src/utility/led/LED_Strip_Class.cpp](src/utility/led/LED_Strip_Class.cpp) |
| Button debounce | [src/utility/Button_Class.cpp](src/utility/Button_Class.cpp) |
| I²C wrapper | [src/utility/I2C_Class.cpp](src/utility/I2C_Class.cpp) |
| Board init code | [src/M5Unified.cpp:1617](src/M5Unified.cpp#L1617) |
