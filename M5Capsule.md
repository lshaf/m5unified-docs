# M5Capsule — standalone code

> **Cross-device references:** [Display brightness](REF_DISPLAY_BRIGHTNESS.md) · [Speaker volume](REF_VOLUME_CONTROL.md) · [Battery & PMIC](REF_POWER_MANAGEMENT.md) · [Sleep / Power-off](REF_POWER_CONTROL.md)

**Board enum:** `board_M5Capsule` (139)
**SoC:** ESP32-S3

Battery-powered S3 capsule — PDM mic, buzzer, SD, RTC, no LCD.

---

## What's onboard

| Component | Presence | Standalone library |
|-----------|----------|--------------------|
| PDM microphone | Yes | ESP-IDF `i2s_pdm_rx` |
| Piezo buzzer | Yes, GPIO2 | ledcWrite |
| PCF8563 RTC | Yes, I²C `0x51` on internal bus | Wire / PCF8563 driver |
| MicroSD card | Yes (SPI) | Arduino `SD.h` |
| WS2812 LED | Yes, GPIO21 | Adafruit_NeoPixel |
| Battery + charger | Yes | simple ADC |
| Display / IMU / Touch | **No** | — |

---

## Pin map

| Purpose | GPIO |
|---------|------|
| **Power hold** | **46** (latch HIGH on boot) |
| PDM DATA-IN / CLK | 41 / 40 |
| Buzzer | 2 |
| Internal I²C SCL / SDA | 10 / 8 |
| Grove (Port A) SCL / SDA | 15 / 13 |
| SD SPI SCLK / MOSI / MISO / CS | 14 / 12 / 39 / 11 |
| WS2812 | 21 |
| BtnA | 42 |
| BtnB | 0 |

---

## 1. Power hold

```cpp
void setup() {
  pinMode(46, OUTPUT);
  digitalWrite(46, HIGH);   // LATCH — do this first on battery
}
```

---

## 2. PDM microphone

```cpp
// ESP-IDF i2s_pdm_rx
gpio_cfg.clk = GPIO_NUM_40;
gpio_cfg.din = GPIO_NUM_41;
```
Full init: [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp).

---

## 3. SD card (SPI host VSPI)

```cpp
#include <SD.h>
#include <SPI.h>
SPIClass sd_spi(HSPI);
void setup() {
  sd_spi.begin(14 /* SCK */, 39 /* MISO */, 12 /* MOSI */, 11 /* CS */);
  if (!SD.begin(11, sd_spi, 20000000)) Serial.println("SD fail");
}
```

---

## 4. Buzzer, RTC, LED, Buttons

Same patterns as [M5Dial.md](M5Dial.md) — see there for exact snippets.

---

## Minimal sketch

```cpp
#include <Adafruit_NeoPixel.h>
Adafruit_NeoPixel led(1, 21, NEO_GRB + NEO_KHZ800);

void setup() {
  pinMode(46, OUTPUT); digitalWrite(46, HIGH);
  led.begin(); led.setBrightness(32);
  pinMode(42, INPUT_PULLUP);
}
void loop() {
  led.setPixelColor(0, digitalRead(42) == LOW ? 0xFF0000 : 0x0000FF);
  led.show(); delay(20);
}
```

---

## Brightness / Volume

No LCD. Buzzer volume = software DSP with `magnification = 48`:
```cpp
M5.Speaker.setVolume(128);
M5.Speaker.tone(2000, 200);
```

See [M5StickCPlus.md § Volume](M5StickCPlus.md#volume-buzzer) for buzzer duty-cycle details.

---

## Power management

Simple charger IC + ADC. ADC on **GPIO6** with 2:1 divider.

```cpp
// standalone:
float vbat_mv() {
  int raw = analogRead(6);
  return (raw * 3300.0f / 4095.0f) * 2.0f;
}
int pct_from_mv(float mv) {
  int pct = (mv - 3300) * 100 / (4150 - 3350);
  return (pct < 0) ? 0 : (pct > 100) ? 100 : pct;
}
```

With M5Unified: `M5.Power.getBatteryLevel()` / `getBatteryVoltage()`. `isCharging()` returns `charge_unknown` — no status GPIO.

Shut down by pulling GPIO46 LOW.

---

## Source references

| Topic | File : line |
|-------|-------------|
| Capsule pinmap | [src/M5Unified.cpp:1679](src/M5Unified.cpp#L1679) |
| PDM mic | [src/utility/Mic_Class.cpp](src/utility/Mic_Class.cpp) |
